# Deploy RHOAI 3.5 on IBM Cloud VPC OpenShift

A step-by-step guide for deploying Red Hat OpenShift AI 3.5 on the `ocp-h200` cluster (OCP 4.21.42,
IBM Cloud VPC `ca-tor-3`). Required for MIG exercises 3.4 (DCGM metrics) and 3.6 (HardwareProfile).

**Upstream BOM reference**: [`bom_rhoai_3.5.md`](https://github.com/mpaulgreen/osac-maas-design-drafts/blob/main/RHOAI%203.5/bom/bom_rhoai_3.5.md)

---

## Already Installed (from ibmc provisioning)

| Component | Version | Channel | Action |
|---|---|---|---|
| Node Feature Discovery | v4.21.0 | `stable` | ✅ Skip |
| NVIDIA GPU Operator | v25.10.1 | `v25.10` | ✅ Skip (BOM says v26.3; v25.10 works for exercises 3.4/3.6) |
| cert-manager | v1.20.1 | `stable-v1.20` | ✅ Skip (newer than BOM's v1.18 — compatible) |

## Service Mesh

OCP **4.21.42 ≥ 4.21.22** → Service Mesh is **ingress-managed via Sail Library**. The ingress
operator runs `istiod-openshift-gateway` automatically. **Do not install the OSSM OLM operator.**

---

## Installation

Set up kubeconfig before starting:

```bash
source ~/.ibmcloud-h200-env
export KUBECONFIG="$INSTALL_DIR/auth/kubeconfig"
```

### Step 1 — RHCL (Red Hat Connectivity Link)

RHCL 1.4.x auto-installs Authorino, Limitador, and DNS as OLM dependencies. No sub-operator
pre-creation needed.

```bash
oc apply -f - <<'EOF'
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: rhcl-operator
  namespace: openshift-operators
spec:
  channel: stable
  installPlanApproval: Automatic
  name: rhcl-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

until oc get csv -n openshift-operators 2>/dev/null | grep rhcl-operator | grep -q Succeeded; do
  echo "  waiting for RHCL..."; sleep 10
done
echo "RHCL installed: $(oc get csv -n openshift-operators | grep rhcl-operator | awk '{print $1}')"
```

### Step 2 — Leader Worker Set

```bash
oc apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-lws-operator
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-lws-operator
  namespace: openshift-lws-operator
spec:
  targetNamespaces:
    - openshift-lws-operator
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: leader-worker-set
  namespace: openshift-lws-operator
spec:
  channel: stable-v1.0
  installPlanApproval: Automatic
  name: leader-worker-set
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

### Step 3 — RHOAI 3.5

```bash
oc apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: redhat-ods-operator
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: rhods-operator
  namespace: redhat-ods-operator
spec: {}
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: rhods-operator
  namespace: redhat-ods-operator
spec:
  channel: stable-3.5
  installPlanApproval: Automatic
  name: rhods-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

oc wait --for=condition=Available deployment/rhods-operator -n redhat-ods-operator --timeout=300s
```

### Step 4 — Cluster Observability Operator

Provides `MonitoringStack` CRD used by RHOAI for its Prometheus instance.

```bash
oc apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-cluster-observability-operator
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: cluster-observability-operator
  namespace: openshift-cluster-observability-operator
spec: {}
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cluster-observability-operator
  namespace: openshift-cluster-observability-operator
spec:
  channel: stable
  installPlanApproval: Automatic
  name: cluster-observability-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

### Step 5 — OpenTelemetry Operator

Required for the OTel Collector that scrapes DCGM metrics and attaches pod/namespace labels,
enabling per-tenant GPU usage attribution (Exercise 3.4 billing path).

```bash
oc apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-opentelemetry-operator
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-opentelemetry-operator
  namespace: openshift-opentelemetry-operator
spec: {}
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: opentelemetry-product
  namespace: openshift-opentelemetry-operator
spec:
  channel: stable
  installPlanApproval: Automatic
  name: opentelemetry-product
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

### Step 6 — Wait for DSCI, then Apply DataScienceCluster

RHOAI auto-creates `default-dsci` after the operator is ready. Wait for it, then apply the DSC.

> **Critical:** Use `apiVersion: .../v2`. Applying with v1 silently prunes the `aigateway` block
> and MaaS never activates.

```bash
until oc get dsci default-dsci 2>/dev/null | grep -q default-dsci; do echo "  waiting for DSCI..."; sleep 10; done
oc wait --for=condition=ReconcileComplete dsci/default-dsci --timeout=300s

oc apply -f - <<'EOF'
apiVersion: datasciencecluster.opendatahub.io/v2
kind: DataScienceCluster
metadata:
  name: default-dsc
spec:
  components:
    dashboard:
      managementState: Managed
    kserve:
      managementState: Managed
    aigateway:
      managementState: Managed
      modelsAsAService:
        managementState: Managed
    aipipelines:
      managementState: Removed
    kueue:
      managementState: Removed
    ray:
      managementState: Removed
    trainingoperator:
      managementState: Removed
    trustyai:
      managementState: Removed
    workbenches:
      managementState: Removed
EOF

oc wait --for=jsonpath='{.status.phase}'=Ready dsc/default-dsc --timeout=900s
```

### Step 7 — Enable DCGM ServiceMonitor

Needed for Prometheus to scrape DCGM metrics (Exercise 3.4 Part B).

```bash
oc patch clusterpolicy gpu-cluster-policy --type=merge -p \
  '{"spec":{"dcgmExporter":{"serviceMonitor":{"enabled":true}}}}'
```

### Step 8 — Enable User Workload Monitoring

Required for Thanos Querier to aggregate DCGM metrics scraped by the user-workload Prometheus.

```bash
oc apply -f - <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
EOF

oc wait pod -l app.kubernetes.io/name=prometheus \
  -n openshift-user-workload-monitoring --for=condition=Ready --timeout=180s
```

---

## Verification Checklist

```bash
source ~/.ibmcloud-h200-env
export KUBECONFIG="$INSTALL_DIR/auth/kubeconfig"

# 1. DSC Ready
oc get dsc default-dsc -o jsonpath='{.status.phase}'
# Expected: Ready

# 2. istiod (Sail Library — no OSSM OLM on ≥4.21.22)
oc get deploy istiod-openshift-gateway -n openshift-ingress
# Expected: 1/1

# 3. GatewayClass
oc get gatewayclass data-science-gateway-class -o jsonpath='{.status.conditions[?(@.type=="Accepted")].status}'
# Expected: True

# 4. Dashboard pods
oc get pods -l app=rhods-dashboard -n redhat-ods-applications | head -4
# Expected: Running

# 5. KServe APIs
oc api-resources --api-group=serving.kserve.io | grep -E 'inferenceservice|llminference'
# Expected: inferenceservices, llminferenceservices

# 6. User workload monitoring
oc get pods -n openshift-user-workload-monitoring | grep prometheus-user-workload
# Expected: 2 pods Running

# 7. DCGM ServiceMonitor
oc get servicemonitor nvidia-dcgm-exporter -n nvidia-gpu-operator
# Expected: exists

# 8. All CSVs Succeeded
oc get csv --all-namespaces | grep -E "rhods|rhcl|lws|observability|telemetry" | awk '!seen[$2]++' | awk '{print $2, $NF}'
```

---

## Installed Stack (validated on 2026-11-17)

| Operator | Version | Channel |
|---|---|---|
| Red Hat OpenShift AI | 3.5.3 | `stable-3.5` |
| Red Hat Connectivity Link | 1.4.3 | `stable` |
| Leader Worker Set | 1.0.1 | `stable-v1.0` |
| Cluster Observability Operator | 1.5.5 | `stable` |
| OpenTelemetry Operator | 0.158.0 | `stable` |
| Authorino (auto) | 1.4.3 | — |
| Limitador (auto) | 1.4.2 | — |
| DNS Operator (auto) | 1.4.2 | — |
