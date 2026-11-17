# Installing Let's Encrypt Certificates on OpenShift on IBM Cloud VPC

This guide documents the complete process for installing Let's Encrypt certificates on an
**OpenShift cluster running on IBM Cloud VPC**, where DNS is managed by **IBM Cloud Internet
Services (CIS)** rather than AWS Route53. It covers the unique challenges — NS delegation, CNAME
shadowing, and CIS API integration — that are specific to this environment.

> **Tested on:** OpenShift 4.21.30, cert-manager v1.20.0, IBM Cloud VPC (ca-tor).
> The same approach also works on OpenShift 4.19+ with cert-manager v1.18+.

---

## Overview

This procedure installs Let's Encrypt certificates using **cert-manager** with **DNS-01**
challenge validation. Because the cluster's DNS is managed by IBM Cloud CIS (not Route53), we
deploy an **IBM CIS webhook** that writes TXT challenge records directly to the CIS API. The
certificates cover:

- `api.<cluster>.<domain>`
- `*.apps.<cluster>.<domain>`

After installation, the router serves a CA-trusted certificate and auto-renews 30 days before
expiry.

---

## Prerequisites

- OpenShift 4.x cluster with `cluster-admin` access
- DNS managed by **IBM Cloud Internet Services (CIS)** — typically via the `ocp-cis` instance
  created during provisioning
- IBM Cloud API key with **Manager** access to the CIS instance
- CIS instance CRN and domain ID (populated in `~/.ibmcloud-h200-env`)

**Session setup** — `API_LB_ID` and `CIS_DOMAIN_ID` are session variables not persisted in the
env file. Run this block in every new terminal before starting:

```bash
source ~/.ibmcloud-h200-env && ibmcloud_login
export KUBECONFIG="$INSTALL_DIR/auth/kubeconfig"
export API_LB_ID=$(ibmcloud is load-balancers --output json | jq -r '.[] | select(.name=="ocp-h200-api-lb") | .id')
ibmcloud cis instance-set $CIS_INSTANCE
export CIS_DOMAIN_ID=$(ibmcloud cis domains --output json | jq -r --arg d "$BASE_DOMAIN" '.[] | select(.name==$d) | .id')
export API_LB_HOST=$(ibmcloud is load-balancer $API_LB_ID --output json | jq -r '.hostname')
export SERVER="https://$API_LB_HOST:6443"
echo "API_LB_HOST=$API_LB_HOST  CIS_DOMAIN_ID=$CIS_DOMAIN_ID"
```

---

## Architecture & Why It's Different Here

### DNS Delegation Chain

On IBM Cloud VPC, the DNS hierarchy works like this:

```
Let's Encrypt   →   queries   →   kni.syseng.devcluster.openshift.com
                                      │
                                      │ NS delegation in Route53
                                      ▼
                                 ns071.name.cloud.ibm.com  (IBM Cloud CIS)
                                      │
                                      ▼
                              ocp-h200.ibmc.kni.syseng... zone
```

The parent zone `kni.syseng.devcluster.openshift.com` lives in **AWS Route53** but has an **NS
delegation** to IBM Cloud CIS for the subdomain. This means:

| What you might try | Why it fails |
|---|---|
| **cert-manager Route53 solver** | Writes TXT to Route53 zone — but Let's Encrypt follows the NS delegation to CIS, so it never sees those records |
| **Manual TXT mirroring** | Works once but must be repeated for every renewal — not sustainable |
| **HTTP-01 challenge** | Let's Encrypt doesn't allow HTTP-01 for wildcard `*.apps` domains — DNS-01 is mandatory |

### The Fix: IBM CIS Webhook

We use **[IBM/cert-manager-webhook-ibmcis](https://github.com/IBM/cert-manager-webhook-ibmcis)**,
an official cert-manager webhook that talks directly to the IBM Cloud CIS API. It creates and
deletes `_acme-challenge` TXT records in the CIS zone where Let's Encrypt actually queries them.

### CNAME Shadowing (Critical!)

Two CNAME records needed for the cluster to function will **shadow** the ACME challenge TXT
records:

```
api.ocp-h200...  CNAME  →  0d8bd133-ca-tor.lb.appdomain.cloud
*.apps.ocp-h200...  CNAME  →  79a30af4-ca-tor.lb.appdomain.cloud
```

DNS doesn't serve a TXT record at the same name as a CNAME — the CNAME wins. And a wildcard
CNAME (`*.apps`) also synthesizes for `_acme-challenge.apps`. So during validation, the `api`
CNAME must be temporarily removed. The wildcard `*.apps` is less disruptive (we patch the router
after the cert is issued), but the `api` CNAME blocks cluster access while gone.

> **Workaround for `api` downtime:** Switch your `oc` / `kubectl` to use the LB hostname
> directly (e.g., `--server=https://0d8bd133-ca-tor.lb.appdomain.cloud:6443`) while the CNAME is
> removed.

---

## Step 1 — Install cert-manager Operator

```bash
source ~/.ibmcloud-h200-env
export KUBECONFIG="$INSTALL_DIR/auth/kubeconfig"

# Derive the API LB hostname (one-time)
API_LB_HOST=$(ibmcloud is load-balancer $API_LB_ID --output json | jq -r '.hostname')
SERVER="https://$API_LB_HOST:6443"

# Create namespace and OperatorGroup
oc create namespace cert-manager-operator --server="$SERVER" --insecure-skip-tls-verify=true

cat <<EOF | oc apply --server="$SERVER" --insecure-skip-tls-verify=true -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: cert-manager-operator
  namespace: cert-manager-operator
spec:
  targetNamespaces:
  - cert-manager-operator
EOF

# Subscribe to the operator (use the latest stable channel for your OCP version)
cat <<EOF | oc apply --server="$SERVER" --insecure-skip-tls-verify=true -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-cert-manager-operator
  namespace: cert-manager-operator
spec:
  channel: stable-v1.20
  installPlanApproval: Automatic
  name: openshift-cert-manager-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

# Wait
oc wait --for=jsonpath='{.status.state}'=AtLatestKnown \
  subscription/openshift-cert-manager-operator \
  -n cert-manager-operator --timeout=300s \
  --server="$SERVER" --insecure-skip-tls-verify=true

# Verify cert-manager pods are running
sleep 10
oc get pods -n cert-manager --server="$SERVER" --insecure-skip-tls-verify=true
# Expected: cert-manager, cert-manager-cainjector, cert-manager-webhook all 1/1 Running
```

> **Note:** The `--server` and `--insecure-skip-tls-verify` flags are needed because the cluster
> has a self-signed certificate at this point. Once Let's Encrypt certs are installed, `oc` will
> trust the connection natively.

---

## Step 2 — Deploy the IBM CIS Webhook

> **Order matters:** Create the `ibmcis-credentials` secret (Step 3) immediately after applying
> the manifest — the webhook pod won't start without it.

The webhook lives at `IBM/cert-manager-webhook-ibmcis` and provides a single `kubectl apply`
manifest. It creates everything: namespace, RBAC, deployment, service, APIService, and internal
PKI.

```bash
source ~/.ibmcloud-h200-env
API_LB_HOST=$(ibmcloud is load-balancer $API_LB_ID --output json | jq -r '.hostname')
SERVER="https://$API_LB_HOST:6443"

# Create namespace manually (the manifest assumes it exists)
oc create namespace cert-manager-webhook-ibmcis \
  --server="$SERVER" --insecure-skip-tls-verify=true

# Deploy the webhook
oc apply -f https://raw.githubusercontent.com/IBM/cert-manager-webhook-ibmcis/main/cert-manager-webhook-ibmcis.yaml \
  --server="$SERVER" --insecure-skip-tls-verify=true

# Wait for the pod to be ready
oc rollout status deployment/cert-manager-webhook-ibmcis \
  -n cert-manager-webhook-ibmcis \
  --server="$SERVER" --insecure-skip-tls-verify=true --timeout=120s

# Verify APIService is available
oc get apiservice v1alpha1.acme.borup.work \
  --server="$SERVER" --insecure-skip-tls-verify=true
# Expected: AVAILABLE = True
```

---

## Step 3 — Create IBM Cloud API Key Secret

The webhook authenticates to CIS using an API key. This key needs **Manager** access to the
target CIS instance. Your provisioning env file already contains the key.

```bash
source ~/.ibmcloud-h200-env
API_LB_HOST=$(ibmcloud is load-balancer $API_LB_ID --output json | jq -r '.hostname')
SERVER="https://$API_LB_HOST:6443"

oc create secret generic ibmcis-credentials \
  --from-literal=api-token="$IBMCLOUD_API_KEY" \
  -n cert-manager-webhook-ibmcis \
  --server="$SERVER" --insecure-skip-tls-verify=true
```

> **Least-privilege:** For production, create a dedicated service ID with access scoped to the
> CIS instance only:
> ```bash
> ibmcloud iam service-id-create cert-manager-webhook-sid -d "cert-manager DNS-01"
> ibmcloud iam service-policy-create cert-manager-webhook-sid \
>   --service-instance $(ibmcloud resource service-instance $CIS_INSTANCE --output json | jq -r '.[0].guid') \
>   --roles Reader,Writer,Manager
> ibmcloud iam service-api-key-create cert-manager-webhook-apikey cert-manager-webhook-sid
> ```

---

## Step 4 — Create the ClusterIssuer

The ClusterIssuer references the webhook and passes the CIS instance CRN. Get the CRN:

```bash
source ~/.ibmcloud-h200-env
ibmcloud resource service-instance $CIS_INSTANCE --output json | jq -r '.[0].crn'
# Example: crn:v1:bluemix:public:internet-svcs:global:a/...:58d7b2ee-...::
```

Then create the issuer:

```bash
API_LB_HOST=$(ibmcloud is load-balancer $API_LB_ID --output json | jq -r '.hostname')
SERVER="https://$API_LB_HOST:6443"
CIS_CRN=$(ibmcloud resource service-instance $CIS_INSTANCE --output json | jq -r '.[0].crn')

cat <<EOF | oc apply --server="$SERVER" --insecure-skip-tls-verify=true -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: mpaul@redhat.com
    privateKeySecretRef:
      name: letsencrypt-production-account-key
    solvers:
    - dns01:
        webhook:
          groupName: acme.borup.work
          solverName: ibmcis
          config:
            apiKeySecretRef:
              name: ibmcis-credentials
              key: api-token
            cisCRN:
              - "$CIS_CRN"
      selector:
        dnsZones:
          - "${CLUSTER_NAME}.ibmc.kni.syseng.devcluster.openshift.com"
EOF

# Wait for Ready
sleep 5
oc get clusterissuer letsencrypt-production \
  --server="$SERVER" --insecure-skip-tls-verify=true
# Expected: READY = True
```

---

## Step 5 — Remove cert-manager's Recursive DNS Configuration (Critical!)

During the operator installation, cert-manager may have been configured with
`--dns01-recursive-nameservers=8.8.8.8,1.1.1.1`. While this works for Route53-based setups, it
will cause perpetual `propagation check failed` errors when using the CIS webhook. Reason: public
DNS caches have propagation lag (seconds to minutes) and retain negative caches (NXDOMAIN),
whereas CoreDNS resolves directly against the CIS delegated nameservers with no caching delay.

```bash
API_LB_HOST=$(ibmcloud is load-balancer $API_LB_ID --output json | jq -r '.hostname')
SERVER="https://$API_LB_HOST:6443"

# Remove the recursive DNS override
oc patch certmanager cluster --type=merge \
  -p '{"spec":{"unsupportedConfigOverrides":null}}' \
  --server="$SERVER" --insecure-skip-tls-verify=true

# Wait for cert-manager to restart
oc rollout status deployment/cert-manager -n cert-manager \
  --server="$SERVER" --insecure-skip-tls-verify=true --timeout=120s

# Verify the args no longer contain dns01-recursive-nameservers
oc get deployment cert-manager -n cert-manager \
  -o jsonpath='{.spec.template.spec.containers[0].args}' \
  --server="$SERVER" --insecure-skip-tls-verify=true
```

> **New clusters:** Skip the recursive DNS config entirely — cert-manager on IBM Cloud VPC
> should resolve via CoreDNS → VPC DNS → CIS, not via public resolvers.

---

## Step 6 — Request the Certificate

```bash
source ~/.ibmcloud-h200-env
API_LB_HOST=$(ibmcloud is load-balancer $API_LB_ID --output json | jq -r '.hostname')
SERVER="https://$API_LB_HOST:6443"

cat <<EOF | oc apply --server="$SERVER" --insecure-skip-tls-verify=true -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: router-certs
  namespace: openshift-ingress
spec:
  secretName: router-certs
  duration: 2160h0m0s
  renewBefore: 360h0m0s
  subject:
    organizations:
    - Red Hat
  privateKey:
    algorithm: RSA
    size: 2048
  usages:
  - server auth
  - client auth
  dnsNames:
  - api.${CLUSTER_NAME}.ibmc.kni.syseng.devcluster.openshift.com
  - '*.apps.${CLUSTER_NAME}.ibmc.kni.syseng.devcluster.openshift.com'
  issuerRef:
    name: letsencrypt-production
    kind: ClusterIssuer
    group: cert-manager.io
EOF
```

---

## Step 7 — Handle CNAME Shadowing for Validation

At this point the certificate will be in `Issuing` state with pending challenges. The webhook
writes `_acme-challenge` TXT records to CIS, but the `api` CNAME shadows the TXT — cert-manager
can't verify the record exists.

### 7a. Verify the TXT is in CIS

```bash
source ~/.ibmcloud-h200-env
ibmcloud cis instance-set $CIS_INSTANCE
ibmcloud cis dns-records $CIS_DOMAIN_ID | grep _acme-challenge
# Should show one or more TXT records
```

### 7b. Temporarily Remove the `api` CNAME

> ⚠️ This breaks cluster DNS temporarily — switch `oc` to the LB hostname first.

```bash
source ~/.ibmcloud-h200-env
ibmcloud cis instance-set $CIS_INSTANCE

# Switch to LB hostname for the duration
API_LB_HOST=$(ibmcloud is load-balancer $API_LB_ID --output json | jq -r '.hostname')
SERVER="https://$API_LB_HOST:6443"

# Find and delete the api CNAME
API_CID=$(ibmcloud cis dns-records $CIS_DOMAIN_ID --output json 2>/dev/null | \
  jq -r '.[] | select(.name == "api.'"$CLUSTER_NAME"'.ibmc.kni.syseng.devcluster.openshift.com" and .type == "CNAME") | .id')
if [ -n "$API_CID" ]; then
  echo y | ibmcloud cis dns-record-delete $CIS_DOMAIN_ID "$API_CID"
fi
```

### 7c. Wait for Certificate to Become Ready

```bash
until [ "$(oc get certificate router-certs -n openshift-ingress \
  --server="$SERVER" --insecure-skip-tls-verify=true \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')" = True ]; do
  echo "$(date) - waiting for certificate..."
  sleep 15
done
echo "Certificate Ready!"
```

### 7d. Restore the `api` CNAME

```bash
source ~/.ibmcloud-h200-env
API_LB_HOST=$(ibmcloud is load-balancer $API_LB_ID --output json | jq -r '.hostname')

ibmcloud cis dns-record-create $CIS_DOMAIN_ID \
  --type CNAME \
  --name "api.${CLUSTER_NAME}" \
  --content "$API_LB_HOST" \
  --ttl 120
```

---

## Step 8 — Patch the IngressController

Once the `router-certs` secret exists, tell the router to use it:

```bash
API_LB_HOST=$(ibmcloud is load-balancer $API_LB_ID --output json | jq -r '.hostname')
SERVER="https://$API_LB_HOST:6443"

oc patch ingresscontroller default -n openshift-ingress-operator \
  --type=merge \
  -p '{"spec":{"defaultCertificate":{"name":"router-certs"}}}' \
  --server="$SERVER" --insecure-skip-tls-verify=true

# Wait for router rollout
oc rollout status deployment/router-default -n openshift-ingress \
  --server="$SERVER" --insecure-skip-tls-verify=true --timeout=120s
```

---

## Step 9 — Verify

```bash
# Check certificate in secret
oc get secret router-certs -n openshift-ingress \
  -o jsonpath='{.data.tls\.crt}' --server="$SERVER" --insecure-skip-tls-verify=true \
  | base64 -d | openssl x509 -noout -issuer -subject -dates

# Expected output:
#   issuer=C=US, O=Let's Encrypt, CN=YR2
#   subject=CN = api.ocp-h200.ibmc.kni.syseng.devcluster.openshift.com
#   notBefore=...  notAfter=...
```

```bash
# Test the console route (after DNS propagates)
echo | openssl s_client \
  -connect console-openshift-console.apps.$CLUSTER_NAME.ibmc.kni.syseng.devcluster.openshift.com:443 \
  -servername console-openshift-console.apps.$CLUSTER_NAME.ibmc.kni.syseng.devcluster.openshift.com 2>/dev/null \
  | openssl x509 -noout -issuer -subject

# Should show "issuer=...O=Let's Encrypt"
```

---

## Troubleshooting

### Cluster unreachable after api CNAME deletion

Switch `oc` to the LB hostname directly:

```bash
export KUBECONFIG="$INSTALL_DIR/auth/kubeconfig"
API_LB_HOST=$(ibmcloud is load-balancer $API_LB_ID --output json | jq -r '.hostname')
SERVER="https://$API_LB_HOST:6443"

oc get nodes --server="$SERVER" --insecure-skip-tls-verify=true
```

### Certificates stuck in pending — propagation check failed

Check cert-manager logs:

```bash
oc logs -n cert-manager deployment/cert-manager --tail=20 \
  --server="$SERVER" --insecure-skip-tls-verify=true \
  | grep propagation
```

Common causes:

| Symptom | Fix |
|---|---|
| `propagation check failed` with recursive DNS enabled | Remove `--dns01-recursive-nameservers` (Step 5) |
| `_acme-challenge` TXT not in CIS | Check webhook logs: `oc logs -n cert-manager-webhook-ibmcis deployment/cert-manager-webhook-ibmcis` |
| Challenge stuck on old `route53-aws-key` secret | Delete zombie challenge: `oc patch challenge <name> -n openshift-ingress -p '{"metadata":{"finalizers":[]}}' --type=merge`, then delete it |

### Webhook pod won't start

The webhook requires the `ibmcis-credentials` secret to exist before it can fully initialize. If
you deployed the manifest before creating the secret, the pod will fail readiness probes. Create
the secret then restart:

```bash
oc rollout restart deployment/cert-manager-webhook-ibmcis -n cert-manager-webhook-ibmcis
```

### macOS DNS cache blocking resolution

After restoring the `api` CNAME, macOS may cache the NXDOMAIN. Flush:

```bash
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

### Renewal failures

If renewal fails (e.g., CIS API key expired), cert-manager will retry with exponential backoff.
Fix the credential and the renewal will pick up automatically. Monitor with:

```bash
oc describe certificate router-certs -n openshift-ingress
```

---

## Auto-Renewal

cert-manager automatically renews certificates 30 days before expiry (`renewBefore: 360h`). The
renewal process is identical to the initial issuance — the webhook creates new TXT records, Let's
Encrypt validates them, and the secret is updated in-place. The router automatically picks up the
new certificate without a restart.

### Monitor Expiry

```bash
oc get certificate -A -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.notAfter}{"\n"}{end}'
```

### Backup Account Key

The Let's Encrypt account private key is critical for renewal. Back it up:

```bash
oc get secret letsencrypt-production-account-key -n cert-manager -o yaml \
  > letsencrypt-production-account-key-backup.yaml
```

---

## References

- [cert-manager Documentation](https://cert-manager.io/docs/)
- [IBM/cert-manager-webhook-ibmcis](https://github.com/IBM/cert-manager-webhook-ibmcis) — the CIS webhook used in this guide
- [rhpds/cert-manager-webhook-ibmcis](https://github.com/rhpds/cert-manager-webhook-ibmcis) — v2 rewrite (Go 1.26, modern SDK; no released images as of Sep 2026)
- [OpenShift cert-manager Operator](https://docs.openshift.com/container-platform/latest/security/cert_manager_operator/index.html)
- [Let's Encrypt Rate Limits](https://letsencrypt.org/docs/rate-limits/)

---

**Last Updated:** September 7, 2026
**Tested On:** OpenShift 4.21.30 on IBM Cloud VPC (ca-tor), cert-manager v1.20.0, IBM CIS webhook v0.1.0
