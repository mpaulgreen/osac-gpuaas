# Provision OpenShift + 1× H100 (or H200) on IBM Cloud (UPI)

A step-by-step, novice-friendly guide to stand up a **self-managed OpenShift cluster** on **IBM Cloud
VPC** with **1 GPU worker node** (8 GPUs — H100 or H200), ending with GPUs schedulable and
`nvidia-smi` passing. Read [`README.md`](./README.md) first for the **cost/quota warning** and *why*
this uses UPI.

**Target:** `ca-tor` region / `ca-tor-3` zone (H100/H200 available; adjust region/zone in Part A5 if needed).

**Parts:** 0 orientation · A prerequisites (once) · B base cluster (3 masters, UPI) · C add 1× GPU
node · D enable GPUs · E validate · F teardown.

> **Conventions.** Everything is driven by env vars in `~/.ibmcloud-h200-env` (copy from
> [`ibmcloud-h200-env.example`](./ibmcloud-h200-env.example)). Re-`source` it in every new terminal.
> Commands capture IDs straight into variables and persist the important ones with `sed`. **This guide
> uses `ca-tor` / `ca-tor-3` as the target region/zone.** If you prefer a different region (e.g.,
> `eu-de` / `eu-de-2`), update `IBMCLOUD_REGION` and `IBMCLOUD_ZONE` in the env file before starting Part B.

---

## Part 0 — What you're building

```
your laptop ── ibmcloud CLI + openshift-install + oc
      │
      ▼  (UPI: you create every instance directly, so ignition works)
IBM Cloud VPC (region ca-tor / zone ca-tor-3)
  ├── 3× master  (bx2-8x32)      ← OpenShift control plane
  └── 1× worker  (gx3d-160x1792x8h100 / gx3d-160x1792x8h200)  ← 8 GPUs (H100 preferred, H200 if available)
```

- **UPI** = User-Provisioned Infrastructure: you run `openshift-install` only to generate configs +
  wait; *you* create the VPC, load balancers, DNS, and instances with `ibmcloud`. (IPI is broken on
  IBM Cloud VPC — see README.)
- **No RoCE / RDMA** in this guide (2 standalone nodes). **No model-serving stack** (provision-only).
- Everything the base cluster needs (masters, operators) also runs on the master nodes (they carry the
  `worker` role), so you don't need non-GPU workers.

---

## Part A — Prerequisites (once per workstation/account)

### A1. Install tools

```bash
# IBM Cloud CLI: https://cloud.ibm.com/docs/cli?topic=cli-install-ibmcloud-cli
ibmcloud version
ibmcloud plugin install vpc-infrastructure -f     # the 'ibmcloud is' commands
ibmcloud plugin install cis -f                     # DNS (Internet Services)
ibmcloud plugin install cloud-object-storage -f    # COS (for the bootstrap ignition + RHCOS image)

# The rest (macOS/Homebrew shown; use your package manager on Linux)
brew install jq helm awscli            # aws CLI is only used for COS presigned URLs
```

Install a **matching** `openshift-install` and `oc` (keep them the same minor version). Example with 4.21 on macOS Apple Silicon (swap `arm64`→`amd64` on Intel/Linux):

```bash
BASEURL=https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable-4.21
curl -L "$BASEURL/openshift-install-mac-arm64.tar.gz" | tar -xz -C /tmp openshift-install
curl -L "$BASEURL/openshift-client-mac-arm64.tar.gz"  | tar -xz -C /tmp oc kubectl
sudo mv /tmp/openshift-install /tmp/oc /tmp/kubectl /usr/local/bin/ && sudo chmod +x /usr/local/bin/{openshift-install,oc,kubectl}
openshift-install version && oc version --client
```

Verify the downloaded versions are 4.21.x:
```bash
openshift-install version | grep openshift-install
oc version --client | grep Client
```

### A2. Log in to IBM Cloud (authenticate first)

You must be logged in **before** you can discover resources — so log in interactively first.

```bash
# If a PREVIOUS setup left a stale key in your shell (e.g. ~/.zshrc sources an old env file),
# `ibmcloud login` will try it and fail with "Provided API key could not be found". Clear it first:
unset IBMCLOUD_API_KEY IC_API_KEY IBMCLOUD_REGION

# Two options for login:
#   a) Federated / SSO account (most Red Hat employees):
ibmcloud login --sso          # opens a one-time-passcode URL
#   b) Classic IBMid (username/password):
#      ibmcloud login
```

**Multi-account disambiguation** — if your SSO identity has access to more than one IBM Cloud account
(e.g. "Markets-Segments" vs "Ecosystem AI Workloads"), `ibmcloud login --sso` may land on the wrong
one. Confirm and switch:

```bash
ibmcloud account list                   # find the GUID for the account you want
ibmcloud target -c <ACCOUNT_GUID> -r ca-tor -g Default
ibmcloud target                         # confirm: Account, Region, Resource group are all correct
```

Copy the GUID into `IBMCLOUD_ACCOUNT_ID` in the env file (A5). After that, `ibmcloud_login` always
targets the right account explicitly.

> **If region is wrong despite the `ca-tor` default:** your `~/.zshrc` (or `~/.bashrc`) likely
> `source`s an old env file that sets `IBMCLOUD_REGION=eu-de`. The `${IBMCLOUD_REGION:-ca-tor}` syntax
> only falls back to `ca-tor` when the variable is **unset** — a stale export overrides it. Fix with
> `grep -n "ibmcloud.*env\|IBMCLOUD_REGION" ~/.zshrc` and remove/update the stale line, or
> `unset IBMCLOUD_REGION` before running the target command.

> **If new terminals keep pre-loading a bad API key:** same root cause — an old env file is being
> sourced. Update it to `source ~/.ibmcloud-h200-env` (created in A5) or remove it.

### A3. Discover account resources (or create missing ones)

Now that you're logged in, gather or create the three account artifacts. The env file template
already has most of these pre-filled from discovery; check/update as needed.

1. **Pull secret** — download from <https://console.redhat.com/openshift/install/pull-secret>, then:
   ```bash
   cp ~/Downloads/pull-secret*.txt ~/.pull-secret.json && chmod 600 ~/.pull-secret.json
   jq empty ~/.pull-secret.json && echo "pull secret OK"
   ```

2. **CIS (DNS)** — you already have `ocp-cis` in your account (discovered), which manages
   `ibmc.kni.syseng.devcluster.openshift.com`. Verify it's set up and has an active zone:
   ```bash
   ibmcloud cis instance-set ocp-cis
   ibmcloud cis domains                    # expect: ibmc.kni.syseng.devcluster.openshift.com (active)
   ```
   If this is wrong (different domain or CIS instance), update the env file in A5.

3. **COS (Cloud Object Storage)** — check first, create if absent:
   ```bash
   ibmcloud resource service-instances --service-name cloud-object-storage
   # If none listed, create one:
   ibmcloud resource service-instance-create ocp-cos cloud-object-storage standard global
   # When prompted "Multiple deployments found...", select: 2 (premium-global-deployment-iam)
   ```
   Update `COS_INSTANCE` in the env file to match the name you used.

4. **SSH key** — SSH keys are **region-scoped** in IBM Cloud VPC. A key uploaded in `eu-de` does not
   exist in `ca-tor`. Check if a key is already present in your target region, and upload if not:
   ```bash
   [ -f ~/.ssh/id_rsa.pub ] || ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
   ibmcloud is keys                       # lists keys in the currently targeted region
   # If none listed, upload your pub key:
   ibmcloud is key-create my-ocp-key-ca-tor @~/.ssh/id_rsa.pub
   ```
   Copy the returned key ID (starts with `r038-` for ca-tor) and update `SSH_KEY_ID` in the env file:
   ```bash
   sed -i '' "s|^export SSH_KEY_ID=.*|export SSH_KEY_ID=<new-id>|" ~/.ibmcloud-h200-env
   ```

### A4. ⚠️ Request H200 quota FIRST, then confirm availability

8×H200 hosts are constrained. Open an IBM Cloud **support case** (or ask your account rep) to raise
your VPC GPU quota for `gx3d-160x1792x8h200` in your target region **before** building anything.
This guide targets `ca-tor-3` (H200 has been provisioned here, but H100 fallback is common if H200 quota is not pre-raised). If you prefer a different region, mention it in the case.

Confirm the profile is actually offered to you **after** targeting your region:

```bash
# First, set your target region in the env file (or via ibmcloud target -r <region>)
source ~/.ibmcloud-h200-env
ibmcloud target -r $IBMCLOUD_REGION
ibmcloud is instance-profiles | grep gx3d-160x1792x8h200
```

- **Output shown** → good, continue.
- **No output** → no H200 quota/capacity in this region yet. Try another zone/region (H200 also in
  `us-east`, `eu-de`, `eu-gb`, `au-syd`, `chennai`). Update `IBMCLOUD_REGION` and `IBMCLOUD_ZONE` in
  the env file, re-source, and check again. **Do not start Part B** until the profile is listed.

### A5. Populate the env file and authenticate with your API key

Copy the template and fill in the values from A2–A4:

> **Important:** Always start from `ibmcloud-h200-env.example`, never create the file from scratch.
> Parts B3 and B5 persist bucket names with `sed` by matching existing `export COS_IGN_BUCKET=` and
> `export COS_IMAGE_BUCKET=` lines — if those lines are absent, `sed` silently no-ops and the bucket
> names won't be saved to the file.

```bash
cp ibmc/ibmcloud-h200-env.example ~/.ibmcloud-h200-env    # run from the repo root
chmod 600 ~/.ibmcloud-h200-env                             # keep it private — contains your API key

# Edit the file and replace the placeholders:
#   - IBMCLOUD_API_KEY: Your API key created on IBM Cloud (Manage → Access (IAM) → API keys)
#     IMPORTANT: create this key while logged into the TARGET account in the browser.
#     The API key is account-scoped — it determines which account ibmcloud_login lands on.
#   - COS_INSTANCE:  "ocp-cos" (or whatever you named it in A3.3)
#   - BASE_DOMAIN:   "ibmc.kni.syseng.devcluster.openshift.com" (already filled in; update if different)
#   - SSH_KEY_ID:    Your uploaded key's ID from A3.4
nano ~/.ibmcloud-h200-env   # or: vi, or your preferred editor
```

Then source the env and test login with the key:

```bash
source ~/.ibmcloud-h200-env
ibmcloud_login                 # authenticates using IBMCLOUD_API_KEY from the env file
ibmcloud target                # confirm region + resource group + account
```

> **On macOS:** if you have stale env-file sourcing in `~/.zshrc`, grep it and remove/update it first
> (see A2 note). The first `source ~/.ibmcloud-h200-env` will fail if `IC_API_KEY` is still set to an
> old value.

---

## Part B — Base cluster: 3 masters via UPI

> Long pole: ~90–120 min. Run everything from one terminal with `~/.ibmcloud-h200-env` sourced.

### B1. Create the VPC network

```bash
source ~/.ibmcloud-h200-env; ibmcloud_login

# VPC
export VPC_ID=$(ibmcloud is vpc-create $VPC_NAME --address-prefix-management manual --output json | jq -r '.id')
sed -i '' "s|^export VPC_ID=.*|export VPC_ID=$VPC_ID|" ~/.ibmcloud-h200-env
echo "VPC_ID=$VPC_ID"

# Address prefix + subnet
ibmcloud is vpc-address-prefix-create mgmt-prefix $VPC_ID $IBMCLOUD_ZONE 10.240.0.0/16
export MGMT_SUBNET_ID=$(ibmcloud is subnet-create $SUBNET_NAME $VPC_ID --zone $IBMCLOUD_ZONE --ipv4-cidr-block $SUBNET_CIDR --output json | jq -r '.id')
sed -i '' "s|^export MGMT_SUBNET_ID=.*|export MGMT_SUBNET_ID=$MGMT_SUBNET_ID|" ~/.ibmcloud-h200-env

# Public gateway (lets nodes egress + the cloud controller reach the VPC API)
export PGW_ID=$(ibmcloud is public-gateway-create ocp-h200-pgw $VPC_ID $IBMCLOUD_ZONE --output json | jq -r '.id')
ibmcloud is subnet-update $MGMT_SUBNET_ID --pgw $PGW_ID
echo "Subnet=$MGMT_SUBNET_ID  PGW=$PGW_ID"
```

### B2. Generate install configs

```bash
export IC_API_KEY="$IBMCLOUD_API_KEY"
rm -rf "$INSTALL_DIR" && mkdir -p "$INSTALL_DIR"

cat <<EOF | tee "$INSTALL_DIR/install-config.yaml"
apiVersion: v1
baseDomain: ${BASE_DOMAIN}
metadata:
  name: ${CLUSTER_NAME}
compute:
- architecture: amd64
  name: worker
  replicas: 0                 # GPU nodes are added by hand in Part C
  platform:
    ibmcloud:
      zones: [${IBMCLOUD_ZONE}]
controlPlane:
  architecture: amd64
  name: master
  replicas: 3
  platform:
    ibmcloud:
      type: bx2-8x32
      zones: [${IBMCLOUD_ZONE}]
platform:
  ibmcloud:
    region: ${IBMCLOUD_REGION}
    resourceGroupName: ${IBMCLOUD_RESOURCE_GROUP}
    networkResourceGroupName: ${IBMCLOUD_RESOURCE_GROUP}
    vpcName: ${VPC_NAME}
    controlPlaneSubnets: [${SUBNET_NAME}]
    computeSubnets: [${SUBNET_NAME}]
credentialsMode: Manual
pullSecret: '$(jq -c . $PULL_SECRET_PATH)'
sshKey: '$(cat $SSH_KEY_PATH)'
EOF
cp "$INSTALL_DIR/install-config.yaml" "$INSTALL_DIR/install-config.yaml.backup"

# credentialsMode: Manual → generate manifests, inject 5 credential secrets, then ignition
openshift-install create manifests --dir "$INSTALL_DIR"
for ns_secret in \
  "openshift-cloud-controller-manager:ibm-cloud-credentials" \
  "openshift-machine-api:ibmcloud-credentials" \
  "openshift-image-registry:installer-cloud-credentials" \
  "openshift-ingress-operator:cloud-credentials" \
  "openshift-cluster-csi-drivers:ibm-cloud-credentials"; do
  NS="${ns_secret%%:*}"; SECRET="${ns_secret##*:}"
  cat <<EOF | tee "$INSTALL_DIR/manifests/${NS}-${SECRET}-credentials.yaml"
apiVersion: v1
kind: Secret
metadata:
  name: ${SECRET}
  namespace: ${NS}
type: Opaque
stringData:
  ibmcloud_api_key: ${IBMCLOUD_API_KEY}
EOF
done
openshift-install create ignition-configs --dir "$INSTALL_DIR"
wc -c "$INSTALL_DIR"/bootstrap.ign "$INSTALL_DIR"/master.ign   # bootstrap > 64KB, master < 64KB
```

### B3. Put bootstrap.ign in COS + make a small "shim" ignition

`bootstrap.ign` is larger than the 64 KB VPC user-data limit, so it goes to COS and the bootstrap
node fetches it via a presigned URL.

```bash
export COS_CRN=$(ibmcloud resource service-instance $COS_INSTANCE --output json | jq -r '.[0].crn')
ibmcloud cos config crn --crn $COS_CRN
ibmcloud cos config region --region $IBMCLOUD_REGION

# COS bucket names must be globally unique — append a random suffix
export COS_IGN_BUCKET="ocp-h200-ign-${IBMCLOUD_REGION}-$(uuidgen | tr '[:upper:]' '[:lower:]' | cut -c1-8)"
ibmcloud cos bucket-create --bucket "$COS_IGN_BUCKET" --region "$IBMCLOUD_REGION"
sed -i '' "s|^export COS_IGN_BUCKET=.*|export COS_IGN_BUCKET=$COS_IGN_BUCKET|" ~/.ibmcloud-h200-env
echo "Ignition bucket: $COS_IGN_BUCKET"

# temporary HMAC keys → upload bootstrap.ign via AWS CLI
# (ibmcloud cos object-put has a known CRN-resolution bug — AWS CLI + HMAC avoids it)
KEYS_JSON=$(ibmcloud resource service-key-create ocp-h200-hmac --instance-name $COS_INSTANCE --parameters '{"HMAC":true}' --output json)
export COS_ACCESS_KEY=$(echo "$KEYS_JSON" | jq -r '.credentials.cos_hmac_keys.access_key_id')
export COS_SECRET_KEY=$(echo "$KEYS_JSON" | jq -r '.credentials.cos_hmac_keys.secret_access_key')
AWS_ACCESS_KEY_ID=$COS_ACCESS_KEY AWS_SECRET_ACCESS_KEY=$COS_SECRET_KEY \
  aws s3 cp "$INSTALL_DIR/bootstrap.ign" "s3://$COS_IGN_BUCKET/bootstrap.ign" \
  --endpoint-url "https://s3.$IBMCLOUD_REGION.cloud-object-storage.appdomain.cloud" \
  --region "$IBMCLOUD_REGION"

# presigned GET URL (valid 4h)
export BOOTSTRAP_IGN_URL=$(AWS_ACCESS_KEY_ID=$COS_ACCESS_KEY AWS_SECRET_ACCESS_KEY=$COS_SECRET_KEY \
  aws s3 presign "s3://$COS_IGN_BUCKET/bootstrap.ign" \
  --endpoint-url "https://s3.$IBMCLOUD_REGION.cloud-object-storage.appdomain.cloud" \
  --region "$IBMCLOUD_REGION" --expires-in 14400)
curl -so /dev/null -w "presigned URL http=%{http_code} (want 200)\n" "$BOOTSTRAP_IGN_URL"

cat <<EOF | tee "$INSTALL_DIR/bootstrap-shim.ign"
{"ignition":{"version":"3.2.0","config":{"merge":[{"source":"${BOOTSTRAP_IGN_URL}"}]}}}
EOF
```

> **Note:** The `AWS_ACCESS_KEY_ID=$COS_ACCESS_KEY` prefix above is an *inline* variable assignment (`VAR=value command`). It sets the variable **only** for that single `aws` command — once the command exits, `AWS_ACCESS_KEY_ID` is gone. So `echo $AWS_ACCESS_KEY_ID` returning blank *after* the upload is expected. The upload worked (look for `upload: ... to s3://` in the output).

### B4. Security group

```bash
export OCP_SG_ID=$(ibmcloud is security-group-create $SG_NAME $VPC_ID --output json | jq -r '.id')
sed -i '' "s|^export OCP_SG_ID=.*|export OCP_SG_ID=$OCP_SG_ID|" ~/.ibmcloud-h200-env
CIDR=$SUBNET_CIDR
ibmcloud is security-group-rule-add $OCP_SG_ID inbound tcp --port-min 22 --port-max 22 --remote 0.0.0.0/0
ibmcloud is security-group-rule-add $OCP_SG_ID inbound tcp --port-min 6443 --port-max 6443 --remote 0.0.0.0/0
ibmcloud is security-group-rule-add $OCP_SG_ID inbound tcp --port-min 22623 --port-max 22623 --remote $CIDR
ibmcloud is security-group-rule-add $OCP_SG_ID inbound tcp --port-min 2379 --port-max 2380 --remote $CIDR
ibmcloud is security-group-rule-add $OCP_SG_ID inbound tcp --port-min 10250 --port-max 10259 --remote $CIDR
ibmcloud is security-group-rule-add $OCP_SG_ID inbound udp --port-min 4789 --port-max 4789 --remote $CIDR
ibmcloud is security-group-rule-add $OCP_SG_ID inbound udp --port-min 6081 --port-max 6081 --remote $CIDR
ibmcloud is security-group-rule-add $OCP_SG_ID inbound tcp --port-min 9000 --port-max 9999 --remote $CIDR
ibmcloud is security-group-rule-add $OCP_SG_ID inbound udp --port-min 9000 --port-max 9999 --remote $CIDR
ibmcloud is security-group-rule-add $OCP_SG_ID inbound tcp --port-min 30000 --port-max 32767 --remote $CIDR
ibmcloud is security-group-rule-add $OCP_SG_ID inbound icmp --remote $CIDR
ibmcloud is security-group-rule-add $OCP_SG_ID outbound icmp_tcp_udp --remote 0.0.0.0/0
```

### B5. Import the RHCOS image

```bash
# find + download the IBM Cloud RHCOS qcow2 for your installer version
RHCOS_URL=$(openshift-install coreos print-stream-json | jq -r '.architectures.x86_64.artifacts.ibmcloud.formats["qcow2.gz"].disk.location')
curl -L -o /tmp/rhcos.qcow2.gz "$RHCOS_URL" && gunzip -f /tmp/rhcos.qcow2.gz    # → /tmp/rhcos.qcow2

# let the VPC image service read COS, upload the image, import it
ibmcloud iam authorization-policy-create is cloud-object-storage Reader \
  --source-resource-type image \
  --target-service-instance-id $(ibmcloud resource service-instance $COS_INSTANCE --output json | jq -r '.[0].guid') 2>&1 || echo "auth policy already exists — OK"
# COS bucket names must be globally unique — append a random suffix
export COS_IMAGE_BUCKET="ocp-h200-image-${IBMCLOUD_REGION}-$(uuidgen | tr '[:upper:]' '[:lower:]' | cut -c1-8)"
ibmcloud cos bucket-create --bucket "$COS_IMAGE_BUCKET" --region "$IBMCLOUD_REGION"
sed -i '' "s|^export COS_IMAGE_BUCKET=.*|export COS_IMAGE_BUCKET=$COS_IMAGE_BUCKET|" ~/.ibmcloud-h200-env
echo "Image bucket: $COS_IMAGE_BUCKET"

# Upload RHCOS image via AWS CLI (HMAC keys from B3; regenerate if in a new terminal)
if [ -z "$COS_ACCESS_KEY" ] || [ -z "$COS_SECRET_KEY" ]; then
  KEYS_JSON=$(ibmcloud resource service-key-create ocp-h200-hmac2 --instance-name $COS_INSTANCE --parameters '{"HMAC":true}' --output json)
  export COS_ACCESS_KEY=$(echo "$KEYS_JSON" | jq -r '.credentials.cos_hmac_keys.access_key_id')
  export COS_SECRET_KEY=$(echo "$KEYS_JSON" | jq -r '.credentials.cos_hmac_keys.secret_access_key')
fi
AWS_ACCESS_KEY_ID=$COS_ACCESS_KEY AWS_SECRET_ACCESS_KEY=$COS_SECRET_KEY \
  aws s3 cp /tmp/rhcos.qcow2 "s3://$COS_IMAGE_BUCKET/rhcos.qcow2" \
  --endpoint-url "https://s3.$IBMCLOUD_REGION.cloud-object-storage.appdomain.cloud" \
  --region "$IBMCLOUD_REGION"
ibmcloud is image-create ocp-h200-rhcos \
  --file "cos://$IBMCLOUD_REGION/$COS_IMAGE_BUCKET/rhcos.qcow2" \
  --os-name rhel-coreos-stable-amd64
echo "Waiting for image to become available..."
until [ "$(ibmcloud is image ocp-h200-rhcos --output json | jq -r '.status')" = "available" ]; do sleep 15; done
export IMAGE_ID=$(ibmcloud is image ocp-h200-rhcos --output json | jq -r '.id')
sed -i '' "s|^export IMAGE_ID=.*|export IMAGE_ID=$IMAGE_ID|" ~/.ibmcloud-h200-env
echo "IMAGE_ID=$IMAGE_ID"
```

### B6. Load balancers (API + API-int)

```bash
export API_LB_ID=$(ibmcloud is load-balancer-create ocp-h200-api-lb public --subnet $MGMT_SUBNET_ID --family application --output json | jq -r '.id')
export API_INT_LB_ID=$(ibmcloud is load-balancer-create ocp-h200-api-int-lb private --subnet $MGMT_SUBNET_ID --family application --output json | jq -r '.id')
echo "Waiting for LBs active..."
until [ "$(ibmcloud is load-balancer $API_LB_ID --output json | jq -r '.provisioning_status')" = active ] \
   && [ "$(ibmcloud is load-balancer $API_INT_LB_ID --output json | jq -r '.provisioning_status')" = active ]; do sleep 15; done
export API_LB_HOSTNAME=$(ibmcloud is load-balancer $API_LB_ID --output json | jq -r '.hostname')
export API_INT_LB_HOSTNAME=$(ibmcloud is load-balancer $API_INT_LB_ID --output json | jq -r '.hostname')

# pools (health-monitor ports)
export API_POOL_ID=$(ibmcloud is load-balancer-pool-create ocp-api-pool $API_LB_ID round_robin tcp 15 2 5 tcp --health-monitor-port 6443 --output json | jq -r '.id')
export API_INT_POOL_ID=$(ibmcloud is load-balancer-pool-create ocp-api-int-pool $API_INT_LB_ID round_robin tcp 15 2 5 tcp --health-monitor-port 6443 --output json | jq -r '.id')
export MCS_POOL_ID=$(ibmcloud is load-balancer-pool-create ocp-mcs-pool $API_INT_LB_ID round_robin tcp 15 2 5 tcp --health-monitor-port 22623 --output json | jq -r '.id')

# listeners — IBM Cloud LBs serialize mutations; API-INT LB gets 2 listeners so wait between them
ibmcloud is load-balancer-listener-create $API_LB_ID --port 6443 --protocol tcp --default-pool $API_POOL_ID
ibmcloud is load-balancer-listener-create $API_INT_LB_ID --port 6443 --protocol tcp --default-pool $API_INT_POOL_ID
# API-INT LB is now update_pending — poll until active before adding its second listener
until [ "$(ibmcloud is load-balancer $API_INT_LB_ID --output json | jq -r '.provisioning_status')" = active ]; do sleep 5; done
ibmcloud is load-balancer-listener-create $API_INT_LB_ID --port 22623 --protocol tcp --default-pool $MCS_POOL_ID

# CRITICAL: the auto-created LB security group blocks 6443/22623 — open them or you get i/o timeout
export LB_SG_ID=$(ibmcloud is load-balancer $API_LB_ID --output json | jq -r '.security_groups[0].id')
ibmcloud is security-group-rule-add $LB_SG_ID inbound tcp --port-min 6443 --port-max 6443 --remote 0.0.0.0/0
ibmcloud is security-group-rule-add $LB_SG_ID inbound tcp --port-min 6443 --port-max 6443 --remote $SUBNET_CIDR
ibmcloud is security-group-rule-add $LB_SG_ID inbound tcp --port-min 22623 --port-max 22623 --remote $SUBNET_CIDR
```

### B7. DNS records (api + api-int)

```bash
ibmcloud cis instance-set $CIS_INSTANCE
export CIS_DOMAIN_ID=$(ibmcloud cis domains --output json | jq -r --arg d "$BASE_DOMAIN" '.[] | select(.name==$d) | .id')

# delete any stale records from a previous run (ignore errors if they don't exist)
for R in $(ibmcloud cis dns-records $CIS_DOMAIN_ID --output json 2>/dev/null | jq -r --arg n "api.${CLUSTER_NAME}" '.[] | select(.name==$n) | .id'); do
  echo y | ibmcloud cis dns-record-delete $CIS_DOMAIN_ID $R 2>/dev/null
done
for R in $(ibmcloud cis dns-records $CIS_DOMAIN_ID --output json 2>/dev/null | jq -r --arg n "api-int.${CLUSTER_NAME}" '.[] | select(.name==$n) | .id'); do
  echo y | ibmcloud cis dns-record-delete $CIS_DOMAIN_ID $R 2>/dev/null
done

ibmcloud cis dns-record-create $CIS_DOMAIN_ID --type CNAME --name "api.${CLUSTER_NAME}"     --content "$API_LB_HOSTNAME"     --ttl 120
ibmcloud cis dns-record-create $CIS_DOMAIN_ID --type CNAME --name "api-int.${CLUSTER_NAME}" --content "$API_INT_LB_HOSTNAME" --ttl 120
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder 2>/dev/null
dig +short api.${CLUSTER_NAME}.${BASE_DOMAIN}     # should resolve
```

### B8. Create bootstrap + 3 masters

```bash
# bootstrap (small, uses the shim ignition)
export BOOTSTRAP_IP=$(ibmcloud is instance-create ocp-h200-bootstrap $VPC_ID $IBMCLOUD_ZONE bx2-4x16 $MGMT_SUBNET_ID \
  --image $IMAGE_ID --user-data @"$INSTALL_DIR/bootstrap-shim.ign" --sgs $OCP_SG_ID --metadata-service true \
  --output json | jq -r '.primary_network_interface.primary_ip.address')

# 3 masters (use the full master.ign)
for n in 0 1 2; do
  eval export MASTER${n}_IP=$(ibmcloud is instance-create ocp-h200-master-$n $VPC_ID $IBMCLOUD_ZONE bx2-8x32 $MGMT_SUBNET_ID \
    --image $IMAGE_ID --user-data @"$INSTALL_DIR/master.ign" --sgs $OCP_SG_ID --metadata-service true \
    --output json | jq -r '.primary_network_interface.primary_ip.address')
done
ibmcloud is instances --output json | jq -r '.[] | select(.name|startswith("ocp-h200-")) | "\(.name) \(.status) \(.primary_network_interface.primary_ip.address)"'
```

### B9. Add instances to the LB pools

IBM Cloud LBs serialise mutations — concurrent adds to the same LB fail with `update_pending`.
The `lb_wait` helper polls until the target LB is `active` before the next call.

```bash
lb_wait() { until [ "$(ibmcloud is load-balancer "$1" --output json | jq -r '.provisioning_status')" = active ]; do sleep 5; done; }

for IP in $BOOTSTRAP_IP $MASTER0_IP $MASTER1_IP $MASTER2_IP; do
  ibmcloud is load-balancer-pool-member-create $API_LB_ID     $API_POOL_ID     6443  $IP
  lb_wait $API_LB_ID
  ibmcloud is load-balancer-pool-member-create $API_INT_LB_ID $API_INT_POOL_ID 6443  $IP
  lb_wait $API_INT_LB_ID
  ibmcloud is load-balancer-pool-member-create $API_INT_LB_ID $MCS_POOL_ID     22623 $IP
  lb_wait $API_INT_LB_ID
done
```

### B10. Bootstrap → hand-off → install-complete

In a **second terminal**, keep macOS DNS from caching a stale NXDOMAIN while the installer runs:

```bash
while true; do sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder 2>/dev/null; sleep 30; done
```

Back in the first terminal:

```bash
openshift-install wait-for bootstrap-complete --dir "$INSTALL_DIR" --log-level=info    # ~30-40 min
```

When it says *"safe to remove the bootstrap resources"*, remove the bootstrap from the pools + delete
it, and clean the ignition from COS:

```bash
lb_wait() { until [ "$(ibmcloud is load-balancer "$1" --output json | jq -r '.provisioning_status')" = active ]; do sleep 5; done; }

for POOL in "$API_LB_ID:$API_POOL_ID" "$API_INT_LB_ID:$API_INT_POOL_ID" "$API_INT_LB_ID:$MCS_POOL_ID"; do
  LB="${POOL%%:*}"; P="${POOL##*:}"
  lb_wait "$LB"
  MID=$(ibmcloud is load-balancer-pool-members "$LB" "$P" --output json | jq -r --arg ip "$BOOTSTRAP_IP" '.[] | select(.target.address==$ip) | .id')
  [ -n "$MID" ] && { ibmcloud is load-balancer-pool-member-delete "$LB" "$P" "$MID" --force; lb_wait "$LB"; }
done
ibmcloud is instance-delete ocp-h200-bootstrap --force
ibmcloud resource service-key-delete ocp-h200-hmac --force 2>/dev/null || true
ibmcloud resource service-key-delete ocp-h200-hmac2 --force 2>/dev/null || true
# Delete the ignition object from COS (the bucket itself is cleaned in Part F)
AWS_ACCESS_KEY_ID=$COS_ACCESS_KEY AWS_SECRET_ACCESS_KEY=$COS_SECRET_KEY \
  aws s3 rm "s3://$COS_IGN_BUCKET/bootstrap.ign" \
  --endpoint-url "https://s3.$IBMCLOUD_REGION.cloud-object-storage.appdomain.cloud" \
  --region "$IBMCLOUD_REGION" 2>/dev/null || true
```

Point CoreDNS at a resolver that can see your CIS domain, then publish `*.apps` once the cloud
controller has created the ingress LB, then finish:

```bash
export KUBECONFIG="$INSTALL_DIR/auth/kubeconfig"
oc patch dns.operator default --type merge -p "{\"spec\":{\"servers\":[{\"name\":\"apps-fwd\",\"zones\":[\"${BASE_DOMAIN}\"],\"forwardPlugin\":{\"upstreams\":[\"8.8.8.8\",\"8.8.4.4\"],\"policy\":\"Random\"}}]}}"

echo "Waiting for the ingress load balancer (created by the cloud controller)..."
until INGRESS=$(oc get svc -n openshift-ingress router-default -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null); [ -n "$INGRESS" ]; do sleep 30; done
ibmcloud cis instance-set $CIS_INSTANCE

# delete stale *.apps record from a previous run (ignore errors if it doesn't exist)
for R in $(ibmcloud cis dns-records $CIS_DOMAIN_ID --output json 2>/dev/null | jq -r '.[] | select(.name | contains("apps")) | .id'); do
  echo y | ibmcloud cis dns-record-delete $CIS_DOMAIN_ID $R 2>/dev/null
done

ibmcloud cis dns-record-create $CIS_DOMAIN_ID --type CNAME --name "*.apps.${CLUSTER_NAME}" --content "$INGRESS" --ttl 120

openshift-install wait-for install-complete --dir "$INSTALL_DIR" --log-level=info    # ~20-30 min
```

Stop the DNS-flush loop (2nd terminal, Ctrl-C). Verify:

```bash
oc get nodes            # 3 masters Ready (control-plane,master,worker)
oc get co               # all Available=True, Degraded=False
echo "console: $(oc whoami --show-console)"
echo "kubeadmin pw: $(cat "$INSTALL_DIR/auth/kubeadmin-password")"
```

---

## Part C — Add 1× GPU node (H100 or H200)

No cluster network / RoCE — just create the instance on the management subnet with the worker
ignition, then approve its CSRs. (SSH key comes from the ignition; `--keys` is silently ignored on GPU
instances, which is fine — you don't need SSH here.)

```bash
source ~/.ibmcloud-h200-env; export KUBECONFIG="$INSTALL_DIR/auth/kubeconfig"

ID=$(ibmcloud is instance-create ocp-h200-0 $VPC_ID $IBMCLOUD_ZONE $H200_GPU_PROFILE $MGMT_SUBNET_ID \
  --image $IMAGE_ID --user-data @"$INSTALL_DIR/worker.ign" --sgs $OCP_SG_ID --metadata-service true \
  --output json | jq -r '.id')
sed -i '' "s|^export H200_0_ID=.*|export H200_0_ID=$ID|" ~/.ibmcloud-h200-env
echo "ocp-h200-0 = $ID"
until [ "$(ibmcloud is instance ocp-h200-0 --output json 2>/dev/null | jq -r '.status')" = running ]; do
  S=$(ibmcloud is instance ocp-h200-0 --output json 2>/dev/null | jq -r '.status')
  echo "  ocp-h200-0: $S"
  if [ "$S" = failed ]; then
    echo "  ❌ ocp-h200-0 failed — see status_reasons:"
    ibmcloud is instance ocp-h200-0 --output json | jq '.status_reasons'
    echo "  If code is cannot_start_capacity, delete and retry with the other profile."
    break
  fi
  sleep 30
done
source ~/.ibmcloud-h200-env
```

> **If it fails with `cannot_start_capacity`:** delete and switch profile:
> ```bash
> ibmcloud is instance-delete ocp-h200-0 --force
> sed -i '' 's|^export H200_GPU_PROFILE=.*|export H200_GPU_PROFILE=gx3d-160x1792x8h200|' ~/.ibmcloud-h200-env
> source ~/.ibmcloud-h200-env
> # rerun the create block above
> ```

**Approve CSRs** (each node issues two rounds: client, then serving). Run this block a few times over
~5 minutes until the GPU node is `Ready`:

```bash
oc get csr --no-headers | grep Pending | awk '{print $1}' | xargs -r oc adm certificate approve
oc get nodes
```

Expected end state: `ocp-h200-0` shows `Ready`. Label it:

```bash
export H200_0_NODE=$(oc get nodes --no-headers | awk '/ocp-h200-0/{print $1}')
sed -i '' "s|^export H200_0_NODE=.*|export H200_0_NODE=$H200_0_NODE|" ~/.ibmcloud-h200-env
oc label node $H200_0_NODE node-role.kubernetes.io/worker="" node-role.kubernetes.io/gpu=true workload.openshift.io/ai-ml=true --overwrite
```

**If the GPU node can't reach the internet from pods** (DNS/UDP sometimes doesn't initialize on
manually-joined nodes), restart its OVN pod:

```bash
oc delete pod -n openshift-ovn-kubernetes -l app=ovnkube-node --field-selector spec.nodeName=$H200_0_NODE
```

---

## Part D — Enable GPUs (NFD + NVIDIA GPU Operator)

### D1. Node Feature Discovery (labels GPU nodes)

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata: {name: openshift-nfd}
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata: {name: openshift-nfd, namespace: openshift-nfd}
spec: {targetNamespaces: [openshift-nfd]}
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata: {name: nfd, namespace: openshift-nfd}
spec:
  channel: stable
  installPlanApproval: Automatic
  name: nfd
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

echo "Waiting for NFD operator..."; until oc get csv -n openshift-nfd 2>/dev/null | grep nfd | grep -q Succeeded; do sleep 10; done

oc apply -f - <<EOF
apiVersion: nfd.openshift.io/v1
kind: NodeFeatureDiscovery
metadata: {name: nfd-instance, namespace: openshift-nfd}
spec:
  workerConfig:
    configData: |
      sources:
        pci:
          deviceClassWhitelist: ["0300","0302"]
          deviceLabelFields: [vendor]
EOF

sleep 90
oc get nodes -l feature.node.kubernetes.io/pci-10de.present=true    # the H100 node should list
```

### D2. NVIDIA GPU Operator

Use a recent channel (the operator installs a driver that supports H200 — needs ≥ 570 / CUDA 12.8;
`v25.10` is known-good). Check OperatorHub for the current channel if this one is stale.

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata: {name: nvidia-gpu-operator}
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata: {name: nvidia-gpu-operator, namespace: nvidia-gpu-operator}
spec: {targetNamespaces: [nvidia-gpu-operator]}
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata: {name: gpu-operator-certified, namespace: nvidia-gpu-operator}
spec:
  channel: v25.10
  installPlanApproval: Automatic
  name: gpu-operator-certified
  source: certified-operators
  sourceNamespace: openshift-marketplace
EOF

echo "Waiting for GPU operator..."; until oc get csv -n nvidia-gpu-operator 2>/dev/null | grep gpu-operator | grep -q Succeeded; do sleep 10; done

# minimal ClusterPolicy (no RDMA — we're not using RoCE)
# Note: v25.x+ requires daemonsets and nodeStatusExporter fields
oc apply -f - <<EOF
apiVersion: nvidia.com/v1
kind: ClusterPolicy
metadata: {name: gpu-cluster-policy}
spec:
  operator: {defaultRuntime: crio, use_ocp_driver_toolkit: true}
  daemonsets: {}
  nodeStatusExporter: {enabled: true}
  driver: {enabled: true}
  toolkit: {enabled: true}
  devicePlugin: {enabled: true}
  dcgm: {enabled: true}
  dcgmExporter: {enabled: true}
  gfd: {enabled: true}
  validator: {plugin: {env: [{name: WITH_WORKLOAD, value: "false"}]}}
EOF

echo "Waiting for ClusterPolicy ready (5-10 min: driver build + container pulls on the node)..."
until [ "$(oc get clusterpolicy gpu-cluster-policy -o jsonpath='{.status.state}' 2>/dev/null)" = ready ]; do sleep 15; done
```

---

## Part E — Validate

```bash
# 8 GPUs allocatable
echo "$H200_0_NODE: $(oc get node $H200_0_NODE -o jsonpath='{.status.allocatable.nvidia\.com/gpu}') GPUs"

# nvidia-smi via the driver container (not `oc debug`)
POD=$(oc get pods -n nvidia-gpu-operator -l app.kubernetes.io/component=nvidia-driver --field-selector spec.nodeName=$H200_0_NODE -o jsonpath='{.items[0].metadata.name}')
oc exec -n nvidia-gpu-operator $POD -c nvidia-driver-ctr -- nvidia-smi --query-gpu=name,memory.total --format=csv,noheader
# expect: 8x  "NVIDIA H100 80GB, 81920 MiB"

# smoke test: run a CUDA workload on a GPU
oc run cuda-smoke --restart=Never --image=nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda12.5.0 \
  --overrides='{"spec":{"containers":[{"name":"cuda-smoke","image":"nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda12.5.0","resources":{"limits":{"nvidia.com/gpu":"1"}}}]}}'
oc wait pod/cuda-smoke --for=condition=Ready --timeout=60s 2>/dev/null || true
sleep 10
oc logs cuda-smoke    # expect: "Test PASSED"
oc delete pod cuda-smoke
```

✅ **Done — 3 masters + 1 H100 node, 8 GPUs schedulable.**

---

## Part F — Teardown (do this when finished — the GPUs bill by the hour)

```bash
source ~/.ibmcloud-h200-env; ibmcloud_login
export KUBECONFIG="$INSTALL_DIR/auth/kubeconfig"
ibmcloud cis instance-set $CIS_INSTANCE
export CIS_DOMAIN_ID=$(ibmcloud cis domains --output json | jq -r --arg d "$BASE_DOMAIN" '.[] | select(.name==$d) | .id')

# 1) GPU node first (the expensive part)
ibmcloud is instance-delete $H200_0_ID --force 2>/dev/null

# 1b) clean up any remaining COS HMAC service keys (hmac/hmac2 are removed in B10; hmac3+ if presigned URL was regenerated)
for KEY in $(ibmcloud resource service-keys --output json 2>/dev/null | jq -r '.[] | select(.name | startswith("ocp-h200-hmac")) | .name'); do
  ibmcloud resource service-key-delete "$KEY" --force 2>/dev/null || true
done

# 2) delete ALL load balancers (the cloud controller made an extra ingress LB), then wait for their reserved IPs to clear
for LB in $(ibmcloud is load-balancers --output json | jq -r '.[].id'); do ibmcloud is load-balancer-delete $LB --force; done
until [ "$(ibmcloud is subnet-reserved-ips $MGMT_SUBNET_ID --output json | jq '[.[]|select(.target.resource_type=="load_balancer")]|length')" = 0 ]; do echo "  waiting for LB IPs to clear..."; sleep 120; done

# 3) masters + custom image + SG + DNS + COS
ibmcloud is instance-delete ocp-h200-master-0 ocp-h200-master-1 ocp-h200-master-2 --force; sleep 60
ibmcloud is image-delete ocp-h200-rhcos --force 2>/dev/null
ibmcloud is security-group-delete $SG_NAME --force 2>/dev/null
for r in $(ibmcloud cis dns-records $CIS_DOMAIN_ID --output json | jq -r --arg c "$CLUSTER_NAME" '.[]|select(.name|contains($c))|.id'); do echo y | ibmcloud cis dns-record-delete $CIS_DOMAIN_ID $r; done
export COS_CRN=$(ibmcloud resource service-instance $COS_INSTANCE --output json | jq -r '.[0].crn')
ibmcloud cos bucket-delete --bucket "$COS_IGN_BUCKET" --region "$IBMCLOUD_REGION" --force 2>/dev/null
ibmcloud cos bucket-delete --bucket "$COS_IMAGE_BUCKET" --region "$IBMCLOUD_REGION" --force 2>/dev/null
# Remove the VPC image → COS Reader IAM authorization policy created in B5
for POL in $(ibmcloud iam authorization-policies --output json 2>/dev/null | \
  jq -r '.[] | select(.subjects[].attributes[] | select(.name=="resourceType" and .value=="image")) | .id'); do
  ibmcloud iam authorization-policy-delete "$POL" --force 2>/dev/null || true
done

# 4) subnet → PGW → address prefix → VPC
echo y | ibmcloud is subnet-public-gateway-detach $MGMT_SUBNET_ID
ibmcloud is subnet-delete $MGMT_SUBNET_ID --force
ibmcloud is public-gateway-delete ocp-h200-pgw --force
PREFIX=$(ibmcloud is vpc-address-prefixes $VPC_ID --output json | jq -r '.[]|select(.name=="mgmt-prefix")|.id')
ibmcloud is vpc-address-prefix-delete $VPC_ID $PREFIX --force
ibmcloud is vpc-delete $VPC_ID --force
rm -rf "$INSTALL_DIR"
```

> **Stopped ≠ free.** Stopping an H100 instance (`ibmcloud is instance-stop <id> --force`) keeps it
> billing and counting against quota. To actually stop paying, **delete** it.

> **In-cluster operators** (cert-manager, NFD, GPU Operator) disappear automatically when the
> masters are deleted — no separate cloud teardown needed for them.

---

## Next steps (beyond this guide)

With 8 GPUs schedulable you can serve a **tool-capable model** and run a real agentic task —
roadmap #1 (and enabling #3 write-gating, #6 other agents):

1. Install RHOAI/KServe + vLLM and deploy e.g. **Qwen2.5-Coder-32B** (fits comfortably on one H200).
   The prior PoC's `manual/PHASE5`–`PHASE6` show the RHOAI + `LLMInferenceService` path.
2. Point **OpenShell + Codex** (see `../prototype/openshift/`) at the served model and run:
   upload repo → fix a failing test → run `pytest` → produce a diff.

## References

- Proven source PoC (internal): `ibmc-ipi-roce/manual/PHASE1`, `PHASE2`, `PHASE3B`, `PHASE4B`, `PHASE5`.
- IBM Cloud VPC — [instance profiles (gx3d H200)](https://cloud.ibm.com/docs/vpc?topic=vpc-profiles),
  [managing GPUs](https://cloud.ibm.com/docs/vpc?topic=vpc-managing-gpus).
- Red Hat — [Installing OpenShift on IBM Cloud VPC](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/installing_on_ibm_cloud/).
- [NVIDIA GPU Operator on OpenShift](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/openshift/introduction.html).
