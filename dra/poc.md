# DRA for GPUs on OpenShift — Research Notes

> **TL;DR:** DRA provides attribute-based GPU scheduling; MIG provides hardware isolation. Together they enable fine-grained multi-tenant GPU allocation without node-level label management. DRA is GA in OCP 4.21. Enabling it on our cluster requires upgrading the GPU Operator to v26.3 and migrating from `ClusterPolicy` to the new `GPUCluster` CRD — a one-way migration.

**GPUaaS connection**: DRA is the discovery and selection layer beneath the entitlement controller. ResourceSlices replace manual GPU counting. DeviceClasses map to SKU tiers. ResourceClaims are what workloads submit. MIG + DRA together replace the global node-label management that makes MIG operationally painful at scale.

---

## Prerequisites

- **OCP 4.21.42** (K8s 1.34) — DRA GA, `resource.k8s.io/v1`, `DynamicResourceAllocation` on by default
- **GPU Operator v26.3** — required for DRA driver via `GPUCluster` CRD (currently v25.10.1 on cluster, upgrade needed)
- **Red Hat operator coverage**: DRA framework in OCP core (Red Hat). NVIDIA DRA driver embedded in `gpu-operator-certified` v26.3 (certified-operators catalog)
- `ClusterPolicy` and `GPUCluster` are **mutually exclusive** — switching to DRA removes the device plugin (`nvidia.com/gpu` resource disappears)
- **Known K8s 1.34 DRA bug**: 1.34.0 and 1.34.1 have a scheduler bug affecting DRA. Verify OCP patch level is 4.21.3+ (uses 1.34.2+) before enabling

---

## Architecture

```
  ┌─────────────────────────────────────────────────────────────┐
  │                 DRA Data Flow (OCP 4.21+)                    │
  │                                                             │
  │  GPU Operator v26.3                                          │
  │  ┌──────────────────┐    publishes    ┌───────────────────┐  │
  │  │  NVIDIA DRA       │───────────────▶│   ResourceSlice    │  │
  │  │  kubelet plugin   │                │   (per node,       │  │
  │  │  (gpu.nvidia.com) │                │    per GPU chip)   │  │
  │  └──────────────────┘                │                   │  │
  │                                       │  devices:         │  │
  │  DeviceClasses (admin-defined)         │  - uuid: ...      │  │
  │  ┌──────────────────┐                │    productName:    │  │
  │  │ gpu.nvidia.com   │                │      "H100 80GB"   │  │
  │  │ mig.nvidia.com   │◀───selector────│    memory: 80Gi   │  │
  │  │ (CEL expressions)│                │    architecture:   │  │
  │  └──────────────────┘                │      "Hopper"      │  │
  │                                       └────────┬──────────┘  │
  │  Scheduler (CEL evaluation)                    │              │
  │                                               ▼              │
  │  ┌──────────────────┐    matches     ┌───────────────────┐  │
  │  │  ResourceClaim    │◀──────────────│    Scheduler       │  │
  │  │  (workload        │               │    evaluates CEL   │  │
  │  │   request)        │               │    against slices  │  │
  │  └──────────┬────────┘               └───────────────────┘  │
  │             │                                                 │
  │             ▼ allocated GPU/MIG-slice                         │
  │  ┌──────────────────┐                                         │
  │  │  Pod gets device  │                                         │
  │  └──────────────────┘                                         │
  └─────────────────────────────────────────────────────────────┘

  DeviceClass objects created by GPU Operator v26.3:
    gpu.nvidia.com          — whole H100 chips
    mig.nvidia.com          — MIG slice instances (profile + parentUUID)
    compute-domain-daemon.nvidia.com    — multi-node NVLink (GB200+)

  Key concept shift:
    Device Plugin: "I need 1 nvidia.com/gpu" — count-based, no attribute awareness
    DRA:           "I need 1 GPU where memory >= 40Gi and architecture == Hopper"
                   — attribute-based, evaluated by scheduler against ResourceSlice data
```

---

## Configuration

### Verify DRA Framework (already in OCP 4.21)

The DRA API objects are part of OCP core — no separate install needed.

```bash
source ~/.ibmcloud-h200-env
export KUBECONFIG="$INSTALL_DIR/auth/kubeconfig"

# DRA CRDs (should exist on OCP 4.21 without any GPU Operator changes)
oc get crd | grep resource.k8s.io
# Expected:
#   deviceclasses.resource.k8s.io
#   resourceclaims.resource.k8s.io
#   resourceclaimtemplates.resource.k8s.io
#   resourceslices.resource.k8s.io

# DRA feature gate is on by default in K8s 1.34
oc get apiservers cluster -o jsonpath='{.spec.unsupportedConfigOverrides}' 2>/dev/null
# Should be empty — no manual feature gate needed
```

### Upgrade GPU Operator to v26.3

> **Warning:** This is a one-way migration. The existing `ClusterPolicy` will be replaced by `GPUCluster`. The `nvidia.com/gpu` extended resource disappears — workloads using the device plugin model must be migrated to `ResourceClaim` before or after.

```bash
# Step 1: Update the subscription channel
oc patch subscription gpu-operator-certified -n nvidia-gpu-operator \
  --type=merge -p '{"spec":{"channel":"v26.3","installPlanApproval":"Automatic"}}'

# Step 2: Wait for the new CSV to reach Succeeded
until oc get csv -n nvidia-gpu-operator 2>/dev/null | grep gpu-operator | grep -q Succeeded; do
  echo "  waiting for v26.3 CSV..."; sleep 15
done
oc get csv -n nvidia-gpu-operator | grep gpu-operator
# Expected: gpu-operator-certified.v26.3.x   Succeeded
```

### Migrate ClusterPolicy → GPUCluster (DRA enablement)

```bash
# Step 1: Delete the existing ClusterPolicy (device plugin path)
oc delete clusterpolicy gpu-cluster-policy

# Step 2: Create GPUCluster (DRA path)
# Note: GPU Operator v26.3 provides the GPUCluster CRD schema.
# The minimal spec below enables the DRA driver for whole GPUs.
# For MIG + DRA, see the MIG section below.
cat <<'EOF' | oc apply -f -
apiVersion: nvidia.com/v1alpha1
kind: GPUCluster
metadata:
  name: gpu-cluster
  namespace: nvidia-gpu-operator
spec:
  operator:
    defaultRuntime: crio
  dra:
    enabled: true
  driver:
    enabled: true
    useOpenShiftDriverToolkit: true
  toolkit:
    enabled: true
  devicePlugin:
    enabled: false   # DRA replaces the device plugin
  gfd:
    enabled: true    # still deployed for node metadata labels
  dcgm:
    enabled: true
  dcgmExporter:
    enabled: true
    serviceMonitor:
      enabled: true
EOF

# Step 3: Wait for GPUCluster to be ready
until oc get gpucluster gpu-cluster -n nvidia-gpu-operator \
  -o jsonpath='{.status.state}' 2>/dev/null | grep -q "ready"; do
  echo "  waiting for GPUCluster ready..."; sleep 15
done

# Step 4: Verify ResourceSlices appear (the DRA driver published GPU inventory)
oc get resourceslices
# Expected: one ResourceSlice per GPU chip (8 for our H100 node)
```

### Static MIG + DRA

Pre-configure MIG geometry using the node label approach from `mig/poc.md`. Once the MIG slices exist in hardware, the DRA driver discovers them and publishes `mig.nvidia.com` ResourceSlice entries — no additional configuration needed.

```bash
# 1. Enable MIG (custom-mixed layout) as documented in mig/poc.md
#    After state=success, the DRA driver auto-discovers the slices

# 2. Verify MIG slices appear as DRA devices
oc get resourceslices -o json | jq '
  .items[] | select(.spec.driver=="mig.nvidia.com") | {
    node: .spec.nodeName,
    devices: [.spec.devices[] | {name: .name, profile: .basic.attributes["gpu.nvidia.com"].profile}]
  }'
# Expected: entries with profile="1g.10gb", "2g.20gb", "3g.40gb"
```

### Dynamic MIG + DRA (Tech Preview — OCP 4.21)

Dynamic MIG allows the scheduler to request MIG partitions on-demand per `ResourceClaim`, eliminating the drain-reconfigure-drain cycle. Requires enabling the `DRAPartitionableDevices` feature gate.

```bash
# Enable feature gate on kube-apiserver and kube-scheduler (Tech Preview — not for production)
oc patch apiservers cluster --type=merge -p '{
  "spec": {
    "unsupportedConfigOverrides": {
      "apiServerArguments": {
        "feature-gates": ["DRAPartitionableDevices=true"]
      }
    }
  }
}'
oc patch scheduler cluster --type=merge -p '{
  "spec": {
    "unsupportedConfigOverrides": {
      "schedulerArguments": {
        "feature-gates": ["DRAPartitionableDevices=true"]
      }
    }
  }
}'
# Note: "unsupportedConfigOverrides" = Tech Preview / breaks supportability boundary
```

---

## Findings

### Finding 1: ResourceSlice — GPU Inventory Layer

The NVIDIA DRA driver publishes one `ResourceSlice` per GPU chip (whole GPU path) or per MIG slice (MIG path). Each device entry contains structured attributes:

```bash
# Inspect a ResourceSlice
RS=$(oc get resourceslices -o jsonpath='{.items[0].metadata.name}')
oc get resourceslice ${RS} -o json | jq '{
  node: .spec.nodeName,
  driver: .spec.driver,
  devices: [.spec.devices[] | {name: .name, attributes: .basic.attributes}]
}'
```

**H100 device attributes** (published by `gpu.nvidia.com` driver):

| Attribute | Example value |
|---|---|
| `gpu.nvidia.com/productName` | `"NVIDIA H100 80GB HBM3"` |
| `gpu.nvidia.com/architecture` | `"Hopper"` |
| `gpu.nvidia.com/cudaComputeCapability` | `"9.0"` |
| `gpu.nvidia.com/memory` | `81920` (MiB) |
| `gpu.nvidia.com/uuid` | `"GPU-8ea59646-..."` |
| `gpu.nvidia.com/driverVersion` | `"580.105.08"` |

**MIG slice attributes** (published by `mig.nvidia.com` driver):

| Attribute | Example value |
|---|---|
| `gpu.nvidia.com/profile` | `"1g.10gb"` |
| `gpu.nvidia.com/parentUUID` | `"GPU-8ea59646-..."` |
| `gpu.nvidia.com/memory` | `9984` (MiB) |
| `gpu.nvidia.com/multiprocessors` | `16` |

**GPUaaS billing implication**: ResourceSlices replace manual GPU counting. The entitlement controller reads ResourceSlice data to know what physical capacity exists — no polling `nvidia.com/gpu` node labels required.

### Finding 2: DeviceClass + CEL — Attribute-Based Scheduling

Admin-defined `DeviceClass` objects use CEL expressions evaluated by the scheduler against `ResourceSlice` device attributes. No node labels needed for scheduling decisions.

```bash
# DeviceClass for H100 (whole GPU, any)
cat <<'EOF' | oc apply -f -
apiVersion: resource.k8s.io/v1
kind: DeviceClass
metadata:
  name: nvidia-h100
spec:
  selectors:
    - cel:
        expression: >
          device.driver == "gpu.nvidia.com" &&
          device.attributes["gpu.nvidia.com"].architecture == "Hopper"
EOF

# DeviceClass for "any GPU with >= 40GB memory"
cat <<'EOF' | oc apply -f -
apiVersion: resource.k8s.io/v1
kind: DeviceClass
metadata:
  name: gpu-large-memory
spec:
  selectors:
    - cel:
        expression: >
          device.driver == "gpu.nvidia.com" &&
          device.attributes["gpu.nvidia.com"].memory >= 40000
EOF

# DeviceClass for MIG 1g.10gb slice (GPUaaS I0 Sandbox tier)
cat <<'EOF' | oc apply -f -
apiVersion: resource.k8s.io/v1
kind: DeviceClass
metadata:
  name: nvidia-mig-1g10gb
spec:
  selectors:
    - cel:
        expression: >
          device.driver == "mig.nvidia.com" &&
          device.attributes["gpu.nvidia.com"].profile == "1g.10gb"
EOF
```

> **Note**: CEL attribute names depend on what the NVIDIA DRA driver publishes. Verify exact names and types against your cluster's actual `ResourceSlice` content before writing production CEL expressions.

### Finding 3: ResourceClaim / ResourceClaimTemplate

**ResourceClaim** (pre-created, referenced by pod):
```bash
cat <<'EOF' | oc apply -f -
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: h100-claim
  namespace: poc4-dra
spec:
  devices:
    requests:
      - name: gpu
        exactly:
          deviceClassName: nvidia-h100
          count: 1
EOF

# Reference from pod
# resources:
#   claims:
#     - name: gpu
# resourceClaims:
#   - name: gpu
#     resourceClaimName: h100-claim
```

**ResourceClaimTemplate** (auto-generates one claim per pod — preferred for jobs/deployments):
```bash
cat <<'EOF' | oc apply -f -
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: h100-template
  namespace: poc4-dra
spec:
  spec:
    devices:
      requests:
        - name: gpu
          exactly:
            deviceClassName: nvidia-h100
            count: 1
EOF

# Reference from pod spec:
# resourceClaims:
#   - name: gpu
#     resourceClaimTemplateName: h100-template
```

### Finding 4: DRAPrioritizedList (`firstAvailable`) — Graceful Degradation

```bash
cat <<'EOF' | oc apply -f -
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: preferred-gpu
  namespace: poc4-dra
spec:
  devices:
    requests:
      - name: gpu
        firstAvailable:
          - name: prefer-h100
            exactly:
              deviceClassName: nvidia-h100
              count: 1
          - name: fallback-any
            exactly:
              deviceClassName: gpu-large-memory
              count: 1
EOF
```

**GPUaaS mapping**: O1 (flex/queued) tenant contract — "I want H100 but accept A100". The scheduler tries in priority order without the tenant needing hardware awareness.

### Finding 5: MIG Slice Selection Without Node Relabeling

With the device plugin, changing which MIG profile a workload can use requires changing the global node label (`nvidia.com/mig.config`), which drains all GPU workloads on the node. With DRA + static MIG, the scheduler selects by CEL expression per-workload:

```bash
cat <<'EOF' | oc apply -f -
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: mig-claim
  namespace: poc4-dra
spec:
  devices:
    requests:
      - name: mig-slice
        exactly:
          deviceClassName: nvidia-mig-1g10gb
          count: 1
---
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: mig-claim-large
  namespace: poc4-dra
spec:
  devices:
    requests:
      - name: mig-slice
        exactly:
          deviceClassName: mig.nvidia.com
          selectors:
            - cel:
                expression: 'device.attributes["gpu.nvidia.com"].profile == "3g.40gb"'
          count: 1
EOF
```

Different workloads can claim different MIG profile sizes from the same node simultaneously — no node label changes, no drain.

### Finding 6: Dynamic MIG — On-Demand Partitioning (Tech Preview)

With `DRAPartitionableDevices` enabled, the scheduler can request MIG partitions be created on-demand per claim:

```bash
cat <<'EOF' | oc apply -f -
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: dynamic-mig-claim
  namespace: poc4-dra
spec:
  devices:
    requests:
      - name: mig-slice
        exactly:
          deviceClassName: mig.nvidia.com
          selectors:
            - cel:
                expression: >
                  device.attributes["gpu.nvidia.com"].profile == "1g.10gb" &&
                  device.attributes["gpu.nvidia.com"].memory >= 9000
          count: 1
EOF
```

The DRA driver receives the claim, calls nvidia-smi to carve the requested profile from an available GPU, and the pod is scheduled to it. **No node label changes, no drain, no reconfigure downtime.**

Operational impact on GPUaaS: eliminates the "drain-and-reconfigure is disruptive" limitation documented in `mig/poc.md` Finding 3. The entitlement controller can change a tenant's MIG tier dynamically without evicting other tenants.

### Finding 7: DRA-in-Kueue Gap

DRA and Kueue today operate independently:
- Kueue manages quotas via `coveredResources` counting `nvidia.com/gpu` (device plugin units)
- DRA allocates via `ResourceClaim` (no `nvidia.com/gpu` resource counted)
- When DRA replaces the device plugin, Kueue cannot account for DRA allocations

**Current workaround**: maintain both device plugin (`nvidia.com/gpu`) and DRA in parallel — the device plugin for Kueue quota counting, DRA for attribute-based scheduling. GPU Operator v26.3 supports hybrid mode on some configurations.

**Future state**: DRA-in-Kueue (upstream KEP in progress, no downstream ETA). When shipped, Kueue will natively manage `ResourceClaim`-based workloads and quotas.

---

## Proof Guide

### Step 1: Verify DRA Framework on Existing Cluster

```bash
source ~/.ibmcloud-h200-env
export KUBECONFIG="$INSTALL_DIR/auth/kubeconfig"

# DRA CRDs (expected on OCP 4.21 without any GPU changes)
oc get crd | grep resource.k8s.io
# Expected: deviceclasses, resourceclaims, resourceclaimtemplates, resourceslices

# No ResourceSlices yet (GPU Operator v25.10 uses device plugin, not DRA)
oc get resourceslices
# Expected: No resources found

# Create poc namespace
oc create namespace poc4-dra
```

### Step 2: Upgrade GPU Operator to v26.3

```bash
# Update subscription to v26.3 channel
oc patch subscription gpu-operator-certified -n nvidia-gpu-operator \
  --type=merge -p '{"spec":{"channel":"v26.3"}}'

until oc get csv -n nvidia-gpu-operator 2>/dev/null | grep gpu-operator | grep -q Succeeded; do
  echo "  waiting..."; sleep 15
done
echo "GPU Operator v26.3 ready"
```

### Step 3: Migrate to GPUCluster

```bash
# Delete existing ClusterPolicy (removes nvidia.com/gpu resource)
oc delete clusterpolicy gpu-cluster-policy

# Create GPUCluster (DRA path) — adjust spec to match GPU Operator v26.3 schema
cat <<'EOF' | oc apply -f -
apiVersion: nvidia.com/v1alpha1
kind: GPUCluster
metadata:
  name: gpu-cluster
  namespace: nvidia-gpu-operator
spec:
  operator:
    defaultRuntime: crio
  dra:
    enabled: true
  driver:
    enabled: true
    useOpenShiftDriverToolkit: true
  toolkit:
    enabled: true
  devicePlugin:
    enabled: false
  gfd:
    enabled: true
  dcgm:
    enabled: true
  dcgmExporter:
    enabled: true
    serviceMonitor:
      enabled: true
EOF

until oc get gpucluster gpu-cluster -n nvidia-gpu-operator \
  -o jsonpath='{.status.state}' 2>/dev/null | grep -q "ready"; do
  echo "  waiting for GPUCluster..."; sleep 15
done
```

### Step 4: Verify ResourceSlices

```bash
oc get resourceslices
# Expected: 8 ResourceSlices (one per H100 chip on ocp-h200-0)

# Inspect attributes on first slice
RS=$(oc get resourceslices -o jsonpath='{.items[0].metadata.name}')
oc get resourceslice ${RS} -o json | jq '{
  node: .spec.nodeName,
  driver: .spec.driver,
  devices: [.spec.devices[] | {
    name: .name,
    attributes: .basic.attributes
  }]
}'
```

### Step 5: Create DeviceClasses for GPUaaS Tiers

```bash
# I1 — whole H100 (Serve)
cat <<'EOF' | oc apply -f -
apiVersion: resource.k8s.io/v1
kind: DeviceClass
metadata:
  name: nvidia-h100
spec:
  selectors:
    - cel:
        expression: >
          device.driver == "gpu.nvidia.com" &&
          device.attributes["gpu.nvidia.com"].architecture == "Hopper"
EOF

# I0 — MIG 1g.10gb slice (Sandbox)
cat <<'EOF' | oc apply -f -
apiVersion: resource.k8s.io/v1
kind: DeviceClass
metadata:
  name: nvidia-mig-1g10gb
spec:
  selectors:
    - cel:
        expression: >
          device.driver == "mig.nvidia.com" &&
          device.attributes["gpu.nvidia.com"].profile == "1g.10gb"
EOF

oc get deviceclass
```

### Step 6: Submit ResourceClaim and Schedule a Pod

```bash
# Create claim for a whole H100
cat <<'EOF' | oc apply -f -
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: h100-claim
  namespace: poc4-dra
spec:
  devices:
    requests:
      - name: gpu
        exactly:
          deviceClassName: nvidia-h100
          count: 1
EOF

# Schedule a pod against the claim
cat <<'EOF' | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: dra-test
  namespace: poc4-dra
spec:
  containers:
    - name: cuda
      image: nvcr.io/nvidia/cuda:12.6.3-base-ubi9
      command: ["nvidia-smi", "-L"]
      resources:
        claims:
          - name: gpu
  resourceClaims:
    - name: gpu
      resourceClaimName: h100-claim
  restartPolicy: Never
EOF

oc wait pod/dra-test -n poc4-dra --for=condition=Ready --timeout=60s
oc logs -n poc4-dra dra-test
# Expected: GPU 0: NVIDIA H100 80GB HBM3 (UUID: ...)

# Verify claim has allocation
oc get resourceclaim h100-claim -n poc4-dra \
  -o jsonpath='{.status.allocation}' | jq .
```

### Step 7: MIG + DRA (Static)

```bash
# Pre-configure MIG (custom-mixed) using mig/poc.md procedures
# Then verify slices appear as DRA devices:
oc get resourceslices -o json | jq '
  .items[] | select(.spec.driver=="mig.nvidia.com") | {
    node: .spec.nodeName,
    slices: [.spec.devices[] | {name: .name, profile: .basic.attributes["gpu.nvidia.com"].profile}]
  }'

# Schedule a pod to a specific MIG slice via ResourceClaim
cat <<'EOF' | oc apply -f -
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: mig-claim
  namespace: poc4-dra
spec:
  devices:
    requests:
      - name: slice
        exactly:
          deviceClassName: nvidia-mig-1g10gb
          count: 1
EOF
```

### Full Cleanup

```bash
oc delete pods --all -n poc4-dra
oc delete jobs --all -n poc4-dra
oc delete resourceclaims --all -n poc4-dra
oc delete resourceclaimtemplates --all -n poc4-dra
oc delete namespace poc4-dra
oc delete deviceclass nvidia-h100 nvidia-mig-1g10gb gpu-large-memory --ignore-not-found

# To revert to device plugin path: delete GPUCluster, recreate ClusterPolicy
# (see ibmc/openshift-h200-provisioning.md Part D for ClusterPolicy YAML)
```

---

## Implications for GPUaaS

| Isolation Grade | Mechanism | DRA Role |
|---|---|---|
| **I0 Sandbox** | MIG `1g.10gb` slice (~9.5 GB, hardware-isolated) | `mig.nvidia.com` DeviceClass, CEL `profile == "1g.10gb"` |
| **I1 Serve** | Whole GPU | `gpu.nvidia.com` DeviceClass, product/memory CEL |
| **I2 Multi-GPU** | Multi-chip, NVLink | count > 1 + `matchAttribute: parentUUID` constraints |
| **I3 Reserved** | Dedicated node | `DeviceClass.suitableNodes` with node selector |
| **I4 Cluster** | Separate cluster | Out of DRA scope |

**Key operational improvements over device-plugin + MIG:**

1. **No global node label changes for MIG sizing**: Workloads select MIG profiles via CEL selectors in `ResourceClaim`. Different tenants on the same node can claim different profile sizes simultaneously without admin intervention.

2. **Dynamic MIG (Tech Preview)**: eliminates the drain-reconfigure-drain cycle entirely. The entitlement controller can change a tenant's MIG tier dynamically by updating their `DeviceClass` CEL selector — the DRA driver handles geometry on-demand.

3. **Graceful degradation via `firstAvailable`**: multi-tier workloads specify H100 → A100 → any fallback. Maps directly to O1 (flex) tenant obtainability contracts.

4. **Structured inventory**: ResourceSlices give the entitlement controller real-time GPU capacity data without polling node labels or device plugin allocatable counts.

**Gap: DRA-in-Kueue.** Quota accounting for DRA-allocated workloads requires Kueue integration not yet shipped downstream. Until then, running both device plugin (for Kueue quota) and DRA (for attribute-based scheduling) is the recommended hybrid mode in GPU Operator v26.3.

---

## References

- [DRA GA in Kubernetes 1.34](https://kubernetes.io/blog/2025/09/01/kubernetes-v1-34-dra-updates/)
- [DRA concepts — Kubernetes docs](https://kubernetes.io/docs/concepts/resource-management/dynamic-resource-allocation/)
- [NVIDIA DRA Driver docs (GPU Operator 26.3)](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/26.3/dra-intro-install.html)
- [NVIDIA DRA Driver GitHub (kubernetes-sigs)](https://github.com/kubernetes-sigs/dra-driver-nvidia-gpu)
- [NVIDIA DRA + MIG Guide](https://dra-driver-nvidia-gpu.sigs.k8s.io/docs/guides/gpu-allocation/mig/)
- [Red Hat Developer — DRA GA in OCP 4.21](https://developers.redhat.com/articles/2026/03/25/dynamic-resource-allocation-goes-ga-red-hat-openshift-421-smarter-gpu)
- [Red Hat Developer — Multitenant AI inference with DRA](https://developers.redhat.com/articles/2026/08/03/multitenant-ai-inference-dynamic-resource-allocation-openshift)
- MIG partitioning research: [`../mig/poc.md`](../mig/poc.md)
- RHOAI 3.5 deployment: [`../mig/rhoai-3.5-deploy.md`](../mig/rhoai-3.5-deploy.md)
- Cluster provisioning: [`../ibmc/openshift-h200-provisioning.md`](../ibmc/openshift-h200-provisioning.md)
