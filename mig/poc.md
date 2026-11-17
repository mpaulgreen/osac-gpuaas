# MIG Partitioning on H100 — Research Notes

> **TL;DR:** MIG lets you rent out fractions of a GPU to multiple tenants safely. This research validates that MIG partitioning works correctly on the provisioned H100 node and that per-slice telemetry, reconfiguration behavior, and RHOAI integration all behave as the GPUaaS design requires.

**GPUaaS connection**: The I0 isolation grade (Sandbox and Serve-Small SKUs) relies on MIG slices as the unit of sale. The entitlement controller creates ClusterQueues with `nvidia.com/mig-1g.10gb` as the resource (smallest H100 MIG slice — ~9.5 GB), and HardwareProfiles that expose these slices in the RHOAI dashboard.

---

## Prerequisites

- **Cluster**: OCP 4.21.42 on IBM Cloud VPC (`ca-tor-3`), GPU Operator v25.10.1
- **GPU nodes**: NVIDIA H100 or A100 (Ampere architecture or later). MIG is not supported on A10G, T4, L40S, or other non-Ampere GPUs.
- **GPU Operator**: NVIDIA GPU Operator installed with ClusterPolicy `ready` (`oc get clusterpolicy gpu-cluster-policy`)
- **NFD**: Node Feature Discovery labeling the GPU node (`nvidia.com/mig.capable=true`)
- **RHOAI 3.5** (required for DCGM Prometheus path and HardwareProfile integration):
  Deploy first → see [`rhoai-3.5-deploy.md`](./rhoai-3.5-deploy.md)

---

## Environment

**Node**: `ocp-h200-0` — 8 × NVIDIA H100 80GB HBM3 (`gx3d-160x1792x8h100`, IBM Cloud VPC ca-tor-3)

```bash
source ~/.ibmcloud-h200-env
export KUBECONFIG="$INSTALL_DIR/auth/kubeconfig"
GPU_NODE="$H200_0_NODE"   # ocp-h200-0
```

```
                    ┌─────────────────────────────────┐
                    │     NVIDIA H100 (80GB HBM3)     │
                    │  8 chips per node                │
                    │  ┌──────────────────────────┐   │
                    │  │   MIG Mode: Enabled      │   │
                    │  │  ┌─────┐ ┌─────┐ ┌─────┐│   │
                    │  │  │1g.  │ │1g.  │ │1g.  ││   │
                    │  │  │10gb │ │10gb │ │10gb ││   │
                    │  │  │Pod A│ │Pod B│ │Pod C││   │
                    │  │  └─────┘ └─────┘ └─────┘│   │
                    │  │  ┌─────┐ ┌──────────────┐│   │
                    │  │  │1g.  │ │   3g.40gb    ││   │
                    │  │  │10gb │ │   Pod D      ││   │
                    │  │  └─────┘ └──────────────┘│   │
                    │  └──────────────────────────┘   │
                    └─────────────────────────────────┘

  GPU Operator MIG Manager:
    - Reads MIG config from a ConfigMap (default-mig-parted-config or custom)
    - Applies partitioning via nvidia-smi MIG commands
    - Exposes slices as Kubernetes extended resources

  Two advertisement strategies:
    mixed:  nvidia.com/mig-1g.10gb = 16, nvidia.com/mig-3g.40gb = 8  (per-profile resources)
    single: nvidia.com/gpu = 56  (uniform layout, all slices same size)
```

---

## MIG Profile Reference (H100 80GB)

Each H100 chip has 7 compute slices (GPCs) and 8 memory slices. Both budgets must be satisfied simultaneously — a valid layout exhausts all 7 compute and all 8 memory slices.

| Profile | Compute Slices | Memory Slices | Memory | Max per H100 |
|---------|---------------|--------------|--------|--------------|
| `1g.10gb` | 1 GPC (~14 SMs) | 1 | ~9.5 GB | 7 |
| `1g.20gb` | 1 GPC | 2 | ~19.5 GB | 4 |
| `2g.20gb` | 2 GPCs (~28 SMs) | 2 | ~19.5 GB | 3 |
| `3g.40gb` | 3 GPCs (~42 SMs) | 4 | ~39.25 GB | 2 |
| `4g.40gb` | 4 GPCs (~56 SMs) | 4 | ~39.25 GB | 1 |
| `7g.80gb` | 7 GPCs (full GPU) | 8 | ~78.75 GB | 1 |

**`all-balanced` layout** (2×`1g.10gb` + 1×`2g.20gb` + 1×`3g.40gb` per chip = 7 compute, 8 memory):
- `nvidia.com/mig-1g.10gb: 16` (2 × 8 chips)
- `nvidia.com/mig-2g.20gb: 8` (1 × 8 chips)
- `nvidia.com/mig-3g.40gb: 8` (1 × 8 chips)

---

## Configuration

### Enabling MIG

The MIG strategy must be set in the ClusterPolicy **before** changing node labels — without it, the MIG Manager partitions the hardware but Kubernetes sees no `nvidia.com/mig-*` resources.

```bash
# 1. Set strategy in ClusterPolicy
oc patch clusterpolicy gpu-cluster-policy --type=merge -p '{"spec":{"mig":{"strategy":"mixed"}}}'

# 2. Delete the state label to avoid stale success signals, then apply the config
oc label node ${GPU_NODE} nvidia.com/mig.config.state-
oc label node ${GPU_NODE} nvidia.com/mig.config=all-1g.10gb --overwrite

# 3. Wait for state=success (guaranteed fresh since we deleted it first)
until oc get node ${GPU_NODE} -o jsonpath='{.metadata.labels.nvidia\.com/mig\.config\.state}' \
  2>/dev/null | grep -q "^success$"; do echo "  waiting..."; sleep 15; done

# 4. Wait for device plugin to re-advertise resources
until [ "$(oc get node ${GPU_NODE} -o jsonpath='{.status.allocatable.nvidia\.com/mig-1g\.10gb}' \
  2>/dev/null)" = "56" ]; do sleep 10; done

oc get node ${GPU_NODE} -o jsonpath='{.status.allocatable}' | jq . | grep -i mig
# "nvidia.com/gpu": "0"          ← whole GPUs hidden when MIG is active (expected)
# "nvidia.com/mig-1g.10gb": "56" ← 7 slices × 8 chips
```

Confirm at hardware level:

```bash
MIG_POD=$(oc get pods -n nvidia-gpu-operator -l app.kubernetes.io/component=nvidia-driver \
  --field-selector spec.nodeName=${GPU_NODE} -o jsonpath='{.items[0].metadata.name}')
oc exec -n nvidia-gpu-operator ${MIG_POD} -c nvidia-driver-ctr -- nvidia-smi mig -lgi
# Lists all 56 MIG instances across 8 GPUs
```

### Heterogeneous (mixed) Layout via Custom ConfigMap

The `all-balanced` layout is available as a built-in profile (`nvidia.com/mig.config=all-balanced`).
For layouts not in the built-ins, use a custom ConfigMap.

**Key rules:**
1. Every profile used as a node label must exist in the active ConfigMap — otherwise MIG Manager fails with `selected mig-config not present`.
2. After updating a ConfigMap, **restart the MIG Manager pod** before changing the node label. Kubernetes propagates ConfigMap changes to pods with a 30-60s delay; restarting the pod guarantees it reads the updated content immediately.
3. Delete the `mig.config.state` label before applying a new label to avoid stale `success` signals.

```bash
# Create the ConfigMap with all needed profiles
cat <<EOF | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: custom-mig-config
  namespace: nvidia-gpu-operator
data:
  config.yaml: |
    version: v1
    mig-configs:
      all-disabled:
        - devices: all
          mig-enabled: false
      all-1g.10gb:
        - devices: all
          mig-enabled: true
          mig-devices:
            "1g.10gb": 7
      custom-mixed:
        - devices: all
          mig-enabled: true
          mig-devices:
            "1g.10gb": 2
            "2g.20gb": 1
            "3g.40gb": 1
EOF

# Switch ClusterPolicy to the custom ConfigMap FIRST
oc patch clusterpolicy gpu-cluster-policy --type=merge \
  -p '{"spec":{"migManager":{"config":{"name":"custom-mig-config"}}}}'

# Restart MIG Manager so it reads the updated ConfigMap immediately
MIG_POD=$(oc get pod -n nvidia-gpu-operator -l app=nvidia-mig-manager \
  --field-selector spec.nodeName=${GPU_NODE} -o jsonpath='{.items[0].metadata.name}')
oc delete pod -n nvidia-gpu-operator ${MIG_POD}
oc wait pod -l app=nvidia-mig-manager -n nvidia-gpu-operator --for=condition=Ready --timeout=60s

# Apply the layout
oc label node ${GPU_NODE} nvidia.com/mig.config.state-
oc label node ${GPU_NODE} nvidia.com/mig.config=custom-mixed --overwrite
sleep 20   # brief settle before checking state
until oc get node ${GPU_NODE} -o jsonpath='{.metadata.labels.nvidia\.com/mig\.config\.state}' \
  2>/dev/null | grep -q "^success$"; do echo "  waiting..."; sleep 15; done

# Confirm MIG Manager applied the right config (not a stale signal)
oc logs -n nvidia-gpu-operator -l app=nvidia-mig-manager --tail=3 | grep "Successfully updated"
# Expected: Successfully updated to MIG config: custom-mixed

# Wait for device plugin to advertise all three resource types
until [ "$(oc get node ${GPU_NODE} -o jsonpath='{.status.allocatable.nvidia\.com/mig-3g\.40gb}' \
  2>/dev/null)" = "8" ]; do sleep 10; done
oc get node ${GPU_NODE} -o jsonpath='{.status.allocatable}' | jq . | grep -i mig
# "nvidia.com/mig-1g.10gb": "16"  (2 × 8 chips)
# "nvidia.com/mig-2g.20gb": "8"   (1 × 8 chips)
# "nvidia.com/mig-3g.40gb": "8"   (1 × 8 chips)
```

---

## Findings

### Finding 1: Isolation is Hardware-Enforced

Two pods scheduled concurrently on the same physical H100 chip land on different MIG instances with fully isolated memory and compute engines. Neither pod can observe the other's data.

Verification — run two jobs requesting `mig-1g.10gb`:

```bash
oc create namespace poc3-mig

for N in 1 2; do
cat <<EOF | oc apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: mig-workload-${N}
  namespace: poc3-mig
spec:
  template:
    spec:
      containers:
        - name: cuda-test
          image: nvcr.io/nvidia/cuda:12.6.3-base-ubi9
          command: ["sh", "-c", "nvidia-smi && nvidia-smi -L && sleep 120"]
          resources:
            requests:
              nvidia.com/mig-1g.10gb: "1"
            limits:
              nvidia.com/mig-1g.10gb: "1"
      restartPolicy: Never
      nodeSelector:
        nvidia.com/gpu.present: "true"
      tolerations:
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
EOF
done

oc logs -n poc3-mig job/mig-workload-1 | grep -E 'GI|UUID|MIG'
oc logs -n poc3-mig job/mig-workload-2 | grep -E 'GI|UUID|MIG'
```

**Observed output** (both pods on chip `GPU 0`, different instances):

```
# Pod 1: GPU 0, GI 7, memory 9984 MiB / 9984 MiB, UUID: MIG-1f03ab9d-...
# Pod 2: GPU 0, GI 8, memory 9984 MiB / 9984 MiB, UUID: MIG-7e33cbc8-...
```

Both see `MIG M.` mode, `~9.5 GB` memory, and different GI IDs — confirming hardware isolation.

### Finding 2: Per-MIG-Slice Telemetry via DCGM

DCGM reports one metric row per MIG instance using labels `GPU_I_PROFILE` and `GPU_I_ID` (not `gpu_instance_id` as the NVIDIA docs sometimes show).

> **Note:** The DCGM exporter container does not include `wget` or `curl`. Use `oc port-forward`.

```bash
DCGM_POD=$(oc get pods -n nvidia-gpu-operator -l app=nvidia-dcgm-exporter \
  --field-selector spec.nodeName=${GPU_NODE} -o jsonpath='{.items[0].metadata.name}')

oc port-forward -n nvidia-gpu-operator pod/${DCGM_POD} 9400:9400 &
PF_PID=$!; sleep 3

# Per-MIG-instance rows (each line = one slice on one chip)
curl -s http://localhost:9400/metrics | grep 'GPU_I_PROFILE' | head -10
kill $PF_PID 2>/dev/null
```

**Observed metric label format:**
```
DCGM_FI_DEV_GPU_TEMP{gpu="0",GPU_I_PROFILE="1g.10gb",GPU_I_ID="9",...} 29
DCGM_FI_DEV_GPU_TEMP{gpu="0",GPU_I_PROFILE="2g.20gb",GPU_I_ID="3",...} 29
DCGM_FI_DEV_GPU_TEMP{gpu="0",GPU_I_PROFILE="3g.40gb",GPU_I_ID="2",...} 29
```

With `custom-mixed` active on 8 chips, 32 total metric rows are generated (4 MIG instances per chip × 8 chips).

**Prometheus/Thanos query** (requires user workload monitoring + DCGM ServiceMonitor):

```bash
# Enable user workload monitoring (one-time)
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

# Enable DCGM ServiceMonitor in ClusterPolicy
oc patch clusterpolicy gpu-cluster-policy --type=merge -p \
  '{"spec":{"dcgmExporter":{"serviceMonitor":{"enabled":true}}}}'

# Wait for Prometheus to scrape (~60s), then query Thanos
sleep 60
THANOS_ROUTE=$(oc get route thanos-querier -n openshift-monitoring -o jsonpath='{.spec.host}')
TOKEN=$(oc create token prometheus-k8s -n openshift-monitoring --duration=1h)
curl -sk -H "Authorization: Bearer ${TOKEN}" \
  "https://${THANOS_ROUTE}/api/v1/query?query=DCGM_FI_DEV_GPU_TEMP" | \
  jq '.data.result[] | {gpu: .metric.gpu, profile: .metric.GPU_I_PROFILE, instance_id: .metric.GPU_I_ID, temp_c: .value[1]}'
```

**GPUaaS billing implication:** `GPU_I_PROFILE` identifies the billing tier; `GPU_I_ID` + `gpu` + `UUID` uniquely identify the slice. The OTel Collector joins these DCGM rows with pod/namespace labels to attribute GPU-seconds to specific tenants.

### Finding 3: Drain-Before-Reconfigure is Mandatory and Disruptive

The MIG Manager drains ALL GPU workloads before reconfiguring — even when the slice type the workload uses exists in both the old and new config. The drain window lasts ~60-90s while resources show `0` in Kubernetes.

Verification — start a `sleep 600` job on a MIG slice, then trigger reconfiguration:

```bash
# Start long-running workload
oc apply -n poc3-mig -f - <<'EOF'
...  (job spec requesting nvidia.com/mig-1g.10gb: 1) ...
EOF

# Observe: trigger reconfiguration and poll pod state + resources
oc label node ${GPU_NODE} nvidia.com/mig.config.state-
oc label node ${GPU_NODE} nvidia.com/mig.config=all-disabled --overwrite
sleep 20
oc label node ${GPU_NODE} nvidia.com/mig.config=all-1g.10gb --overwrite

until ...; do
  echo "Pod: $(oc get pods -n poc3-mig --no-headers | awk '{print $1, $3}')"
  echo "MIG: $(oc get node ${GPU_NODE} -o jsonpath='{.status.allocatable}' | jq . | grep mig)"
  sleep 10
done
```

**Observed sequence:**
1. Pod remains `Running` for ~80s
2. MIG resources drop to `0` (device plugin disabled during drain)
3. Hardware reconfigures
4. `state=success`; `nvidia.com/mig-1g.10gb: 56` re-appears

**Operational implication:** O3 (reserved) tenants need dedicated nodes with stable MIG configs; O0 (scavenged) tenants accept eviction during reconfiguration as part of their service contract. The entitlement controller should avoid MIG reconfigs on nodes with active entitlements.

### Finding 4: ConfigMap Propagation Delay

Kubernetes propagates ConfigMap changes to running pods with a 30-60s delay. If the node label changes before the MIG Manager pod receives the updated ConfigMap, it fails with:
```
Error selecting MIG config: selected mig-config not present: <profile-name>
```

**Fix:** After every ConfigMap update, restart the MIG Manager pod before changing the node label. The restarted pod reads the ConfigMap on startup, bypassing the propagation delay.

```bash
MIG_POD=$(oc get pod -n nvidia-gpu-operator -l app=nvidia-mig-manager \
  --field-selector spec.nodeName=${GPU_NODE} -o jsonpath='{.items[0].metadata.name}')
oc delete pod -n nvidia-gpu-operator ${MIG_POD}
oc wait pod -l app=nvidia-mig-manager -n nvidia-gpu-operator --for=condition=Ready --timeout=60s
# Then change the node label
```

### Finding 5: RHOAI 3.5 HardwareProfile Integration

RHOAI 3.5 uses `HardwareProfile` (`infrastructure.opendatahub.io/v1`) exclusively. The `AcceleratorProfile` CRD was removed in RHOAI 3.5.

**Corrections from docs:**
- Use `apiVersion: infrastructure.opendatahub.io/v1` (v1alpha1 is rejected)
- `spec.scheduling.type` is required; use `"Node"` for nodeSelector-based scheduling
- `AcceleratorProfile` CRD does not exist in RHOAI 3.5

```bash
# Create HardwareProfiles for each MIG slice type offered
cat <<EOF | oc apply -f -
apiVersion: infrastructure.opendatahub.io/v1
kind: HardwareProfile
metadata:
  name: nvidia-mig-1g10gb
  namespace: redhat-ods-applications
  annotations:
    opendatahub.io/disabled: 'false'
    opendatahub.io/display-name: "NVIDIA H100 MIG 1g.10gb (~9.5 GB)"
spec:
  identifiers:
    - defaultCount: 1
      displayName: "MIG 1g.10gb Slice"
      identifier: nvidia.com/mig-1g.10gb
      maxCount: 7
      minCount: 1
      resourceType: Accelerator
  scheduling:
    type: Node
    node:
      nodeSelector:
        nvidia.com/mig.capable: "true"
      tolerations:
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
---
apiVersion: infrastructure.opendatahub.io/v1
kind: HardwareProfile
metadata:
  name: nvidia-mig-2g20gb
  namespace: redhat-ods-applications
  annotations:
    opendatahub.io/disabled: 'false'
    opendatahub.io/display-name: "NVIDIA H100 MIG 2g.20gb (~19.5 GB)"
spec:
  identifiers:
    - defaultCount: 1
      displayName: "MIG 2g.20gb Slice"
      identifier: nvidia.com/mig-2g.20gb
      maxCount: 3
      minCount: 1
      resourceType: Accelerator
  scheduling:
    type: Node
    node:
      nodeSelector:
        nvidia.com/mig.capable: "true"
      tolerations:
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
---
apiVersion: infrastructure.opendatahub.io/v1
kind: HardwareProfile
metadata:
  name: nvidia-mig-3g40gb
  namespace: redhat-ods-applications
  annotations:
    opendatahub.io/disabled: 'false'
    opendatahub.io/display-name: "NVIDIA H100 MIG 3g.40gb (~39 GB)"
spec:
  identifiers:
    - defaultCount: 1
      displayName: "MIG 3g.40gb Slice"
      identifier: nvidia.com/mig-3g.40gb
      maxCount: 2
      minCount: 1
      resourceType: Accelerator
  scheduling:
    type: Node
    node:
      nodeSelector:
        nvidia.com/mig.capable: "true"
      tolerations:
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
EOF

oc get hardwareprofile -n redhat-ods-applications
```

**Validation** — test pod using the same nodeSelector and resource as the HardwareProfile:

```bash
oc apply -n poc3-mig -f - <<'TESTEOF'
apiVersion: v1
kind: Pod
metadata:
  name: mig-hw-profile-test
  namespace: poc3-mig
spec:
  containers:
    - name: cuda
      image: nvcr.io/nvidia/cuda:12.6.3-base-ubi9
      command: ["sh", "-c", "nvidia-smi && sleep 10"]
      resources:
        requests:
          nvidia.com/mig-1g.10gb: "1"
        limits:
          nvidia.com/mig-1g.10gb: "1"
  nodeSelector:
    nvidia.com/mig.capable: "true"
  tolerations:
    - key: nvidia.com/gpu
      operator: Exists
      effect: NoSchedule
  restartPolicy: Never
TESTEOF

oc wait pod/mig-hw-profile-test -n poc3-mig --for=condition=Ready --timeout=60s
oc logs -n poc3-mig mig-hw-profile-test | grep -E 'MIG|GI|Memory'
# Expected: MIG device table with 1g.10gb profile, ~9984 MiB
oc delete pod mig-hw-profile-test -n poc3-mig --grace-period=0
```

**ServingRuntime annotation** — once a runtime exists, annotate it so the RHOAI dashboard recommends the MIG profile:

```bash
oc patch clusterservingruntime <runtime-name> --type=merge -p '
{
  "metadata": {
    "annotations": {
      "opendatahub.io/recommended-accelerators": "[\"nvidia.com/mig-1g.10gb\"]"
    }
  }
}'
```

Dashboard URL: `https://rhods-dashboard-redhat-ods-applications.apps.ocp-h200.ibmc.kni.syseng.devcluster.openshift.com`

---

## Operational Procedures

### Change MIG layout (from any state)

```bash
source ~/.ibmcloud-h200-env
export KUBECONFIG="$INSTALL_DIR/auth/kubeconfig"
GPU_NODE="${H200_0_NODE:-ocp-h200-0}"

# 1. If using a custom ConfigMap, update it and restart MIG Manager first
#    (see Configuration section for the custom ConfigMap block)

# 2. Delete state label + change label (toggle through all-disabled if switching between non-disabled configs)
oc label node ${GPU_NODE} nvidia.com/mig.config.state-
oc label node ${GPU_NODE} nvidia.com/mig.config=all-disabled --overwrite
sleep 20
oc label node ${GPU_NODE} nvidia.com/mig.config=<target-profile> --overwrite

# 3. Wait for confirmed success
until oc get node ${GPU_NODE} -o jsonpath='{.metadata.labels.nvidia\.com/mig\.config\.state}' \
  2>/dev/null | grep -q "^success$"; do echo "  waiting..."; sleep 15; done
oc logs -n nvidia-gpu-operator -l app=nvidia-mig-manager --tail=3 | grep "Successfully updated"

# 4. Wait for device plugin to re-advertise
until [ "$(oc get node ${GPU_NODE} -o jsonpath='{.status.allocatable.nvidia\.com/mig-<profile>}' \
  2>/dev/null)" != "0" ]; do sleep 10; done
```

---

## Full Cleanup

```bash
source ~/.ibmcloud-h200-env
export KUBECONFIG="$INSTALL_DIR/auth/kubeconfig"
GPU_NODE="${H200_0_NODE:-ocp-h200-0}"   # from ~/.ibmcloud-h200-env

# Step 1: Delete workloads and namespace
oc delete jobs --all -n poc3-mig 2>/dev/null || true
oc delete pod mig-hw-profile-test -n poc3-mig --grace-period=0 2>/dev/null || true
oc delete namespace poc3-mig

# Step 2: Switch ClusterPolicy back to default ConfigMap FIRST.
# CRITICAL: do this before changing the node label, or MIG Manager can't find "all-disabled".
oc patch clusterpolicy gpu-cluster-policy --type=merge -p \
  '{"spec":{"migManager":{"config":{"name":"default-mig-parted-config"}}}}'

# Step 3: Restart MIG Manager so it reads the default ConfigMap immediately
# (avoids the 30-60s propagation delay that causes "not present: all-disabled" errors)
MIG_POD=$(oc get pod -n nvidia-gpu-operator -l app=nvidia-mig-manager \
  --field-selector spec.nodeName=${GPU_NODE} -o jsonpath='{.items[0].metadata.name}')
oc delete pod -n nvidia-gpu-operator ${MIG_POD}
oc wait pod -l app=nvidia-mig-manager -n nvidia-gpu-operator --for=condition=Ready --timeout=60s

# Step 4: Disable MIG on the node and wait for full GPUs to re-appear
oc label node ${GPU_NODE} nvidia.com/mig.config.state-
oc label node ${GPU_NODE} nvidia.com/mig.config=all-disabled --overwrite
until oc get node ${GPU_NODE} -o jsonpath='{.metadata.labels.nvidia\.com/mig\.config\.state}' \
  2>/dev/null | grep -q "^success$"; do echo "  disabling MIG..."; sleep 15; done
until [ "$(oc get node ${GPU_NODE} -o jsonpath='{.status.allocatable.nvidia\.com/gpu}' 2>/dev/null)" = "8" ]; do
  sleep 10
done
oc get node ${GPU_NODE} -o jsonpath='{.status.allocatable}' | jq . | grep -E 'nvidia.com/gpu'
# Expected: "nvidia.com/gpu": "8"  (8 full H100 chips, no MIG slices)

# Step 5: Delete the custom MIG ConfigMap
oc delete configmap custom-mig-config -n nvidia-gpu-operator --ignore-not-found

# Step 6: Remove the MIG strategy from ClusterPolicy (restores default non-MIG mode)
oc patch clusterpolicy gpu-cluster-policy --type=json \
  -p '[{"op":"remove","path":"/spec/mig"}]' 2>/dev/null || \
oc patch clusterpolicy gpu-cluster-policy --type=merge -p '{"spec":{"mig":null}}'

# Step 7: Delete MIG HardwareProfiles created during research
oc delete hardwareprofile nvidia-mig-1g10gb nvidia-mig-2g20gb nvidia-mig-3g40gb \
  -n redhat-ods-applications --ignore-not-found
oc get hardwareprofile -n redhat-ods-applications
# Expected: only default-profile remains

# Step 8: Verify clean state
oc get node ${GPU_NODE} -o jsonpath='{.status.allocatable}' | jq . | grep -E 'nvidia.com/gpu'
# Expected: "nvidia.com/gpu": "8"  (no nvidia.com/mig-* resources)
```

> **Stopped ≠ free.** Stopping an H100 instance (`ibmcloud is instance-stop <id> --force`) keeps it
> billing and counting against quota. To stop paying, **delete** it.

---

## Implications for GPUaaS

1. **MIG is the I0 isolation grade.** Hardware-partitioned memory and compute. Each MIG slice is a fully isolated CUDA device — no KV-cache or memory leaks between tenants. Strongest isolation short of cluster-per-tenant.

2. **`nvidia.com/mig-1g.10gb` is the unit of sale for Sandbox/Serve-Small SKUs on H100.** (On A100 this was `mig-1g.5gb`; H100's smallest slice is `1g.10gb` at ~9.5 GB — nearly double the memory, same price tier.)

3. **Mixed strategy is required for multi-SKU on the same node.** If a node serves both Sandbox (`1g.10gb`) and Serve-Small (`2g.20gb`), `mixed` strategy is the only option. `single` strategy restricts all slices to the same size.

4. **MIG reconfig is a maintenance operation, not dynamic.** The entitlement controller should set the MIG config at entitlement provisioning time and avoid changing it while workloads are running.

5. **DCGM per-MIG metrics enable tenant billing attribution.** `GPU_I_PROFILE` identifies the billing tier; `GPU_I_ID` + chip UUID uniquely identifies the instance. Combined with pod/namespace labels, this provides per-tenant GPU utilization for metering.

6. **HardwareProfile is the RHOAI 3.5 integration point.** `AcceleratorProfile` has been removed. Use `infrastructure.opendatahub.io/v1` with `spec.scheduling.type: Node`.

---

## References

- [NVIDIA GPU Operator MIG Support on OpenShift](https://docs.nvidia.com/datacenter/cloud-native/openshift/latest/mig-ocp.html)
- [NVIDIA MIG User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/)
- [RHOAI 3.5 — Working with Hardware Profiles](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.5/html/working_with_accelerators/working-with-hardware-profiles_accelerators)
- Cluster deployment: [`../ibmc/openshift-h200-provisioning.md`](../ibmc/openshift-h200-provisioning.md)
- RHOAI deployment: [`rhoai-3.5-deploy.md`](./rhoai-3.5-deploy.md)
