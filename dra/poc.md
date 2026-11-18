# DRA for GPUs on OpenShift — Research Notes

> **TL;DR:** DRA provides attribute-based GPU scheduling; MIG provides hardware isolation. Together they enable fine-grained multi-tenant GPU allocation without node-level label management. DRA is GA in OCP 4.21. Enabling it on our cluster requires upgrading the GPU Operator to **v26.7** and migrating from `ClusterPolicy` to the new `GPUCluster` CRD — a one-way migration.

**GPUaaS connection**: DRA is the discovery and selection layer beneath the entitlement controller. ResourceSlices replace manual GPU counting. DeviceClasses map to SKU tiers. ResourceClaims are what workloads submit. MIG + DRA together replace the global node-label management that makes MIG operationally painful at scale.

---

## Prerequisites

- **OCP 4.21.42** (K8s 1.34) — DRA GA, `resource.k8s.io/v1`, `DynamicResourceAllocation` on by default
- **GPU Operator v26.7** — required for DRA driver via `GPUCluster` CRD (validated on v26.7.2)
- **Red Hat operator coverage**: DRA framework in OCP core (Red Hat). NVIDIA DRA driver embedded in `gpu-operator-certified` v26.7 (certified-operators catalog)
- `ClusterPolicy` and `GPUCluster` are **mutually exclusive** — switching to DRA removes the device plugin (`nvidia.com/gpu` resource disappears)
- **Known K8s 1.34 DRA bug**: 1.34.0 and 1.34.1 have a scheduler bug affecting DRA. Verify OCP patch level is 4.21.3+ (uses 1.34.2+) before enabling

---

## Architecture

```
  ┌─────────────────────────────────────────────────────────────┐
  │                 DRA Data Flow (OCP 4.21+)                    │
  │                                                             │
  │  GPU Operator v26.7                                          │
  │  ┌──────────────────┐    publishes    ┌───────────────────┐  │
  │  │  NVIDIA DRA       │───────────────▶│   ResourceSlice    │  │
  │  │  kubelet plugin   │                │   (per node —      │  │
  │  │  (gpu.nvidia.com) │                │  all GPUs in one)  │  │
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

  DeviceClass objects created by GPU Operator v26.7:
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

> **Validated on:** OCP 4.21.42 / K8s 1.34, GPU Operator v26.7.2, 2026-11-18

### Verify DRA Framework (already in OCP 4.21)

The DRA API objects are built into OCP core — not CRDs, so `oc get crd | grep resource.k8s.io` returns nothing. Use `oc api-resources` instead.

```bash
source ~/.ibmcloud-h200-env
export KUBECONFIG="$INSTALL_DIR/auth/kubeconfig"

# DRA API group (should exist on OCP 4.21 without any GPU Operator changes)
oc api-resources | grep resource.k8s.io
# Expected output:
#   deviceclasses        resource.k8s.io/v1   false   DeviceClass
#   resourceclaims       resource.k8s.io/v1   true    ResourceClaim
#   resourceclaimtemplates  resource.k8s.io/v1  true  ResourceClaimTemplate
#   resourceslices       resource.k8s.io/v1   false   ResourceSlice
#
# "No resources found" on oc get resourceslices = API present, driver not running yet

# DRA feature gate is on by default in K8s 1.34 — no manual configuration needed
```

### Upgrade GPU Operator to v26.7

> **Important corrections from initial research:**
> - `GPUCluster` CRD is in **v26.7** (not v26.3 as originally documented)
> - `ClusterPolicy` and `GPUCluster` are **mutually exclusive** — not hybrid mode
> - `GPUCluster` replaces `ClusterPolicy` and manages: DRA driver, DCGM, dcgmExporter only
> - MIG Manager is NOT part of `GPUCluster` — requires separate management
> - `nvidia.com/gpu` extended resource disappears when switching to GPUCluster

```bash
# Update subscription channel to v26.7
oc patch subscription gpu-operator-certified -n nvidia-gpu-operator \
  --type=merge -p '{"spec":{"channel":"v26.7"}}'

until oc get csv -n nvidia-gpu-operator 2>/dev/null | grep gpu-operator | grep "v26\.7" | grep -q Succeeded; do
  echo "  waiting..."; sleep 15
done
oc get csv -n nvidia-gpu-operator | grep gpu-operator
# Expected: gpu-operator-certified.v26.7.x   Succeeded
```

### Migrate ClusterPolicy → GPUCluster (DRA enablement)

```bash
# Step 1: Delete the existing ClusterPolicy (required — they cannot co-exist)
oc delete clusterpolicy gpu-cluster-policy

# Step 2: Create GPUCluster — deploys ONLY the DRA driver + DCGM
# The minimal spec (gpus: {}) enables the gpu.nvidia.com DRA kubelet plugin
cat <<'EOF' | oc apply -f -
apiVersion: nvidia.com/v1alpha1
kind: GPUCluster
metadata:
  name: gpu-cluster
  namespace: nvidia-gpu-operator
spec:
  draDriver:
    gpus: {}
  dcgm:
    enabled: true
  dcgmExporter:
    enabled: true
    serviceMonitor:
      enabled: true
EOF

# Step 3: Wait for ResourceSlices to appear
until oc get resourceslices --no-headers 2>/dev/null | wc -l | grep -qv "^0$"; do
  echo "  waiting for ResourceSlices..."; sleep 15
done
oc get resourceslices
# Expected: 1 ResourceSlice containing all 8 H100 GPU devices
```

> **Note:** GPUCluster does NOT manage the NVIDIA driver daemonset, container toolkit, GPU Feature Discovery, or MIG Manager. These are managed by `ClusterPolicy`. Without ClusterPolicy:
> - `nvidia.com/gpu` extended resource disappears
> - MIG Manager is not running (static MIG must be pre-configured and persists)
> - Container toolkit is not running (CDI library injection requires toolkit or manual CDI setup)
>
> To restore the full stack, delete GPUCluster and recreate ClusterPolicy (see Full Cleanup).

### Static MIG + DRA

> **Important limitation (validated 2026-11-18):** `GPUCluster` does NOT deploy MIG Manager. MIG geometry must be pre-configured via node labels BEFORE switching to GPUCluster (while ClusterPolicy is still running). Once GPUCluster is active, the pre-existing MIG geometry persists in hardware, and the DRA driver publishes `mig.nvidia.com` ResourceSlice entries.

```bash
# 1. While ClusterPolicy is still active, enable MIG (from mig/poc.md)
#    After state=success, switch to GPUCluster

# 2. After GPUCluster is active, verify MIG slices appear as DRA devices
oc get resourceslices -o json | jq '
  .items[] | select(.spec.driver=="mig.nvidia.com") | {
    node: .spec.nodeName,
    devices: [.spec.devices[] | {name: .name, profile: .attributes["profile"].string}]
  }'
# Expected: entries with profile="1g.10gb", "2g.20gb", "3g.40gb"
```

### Dynamic MIG + DRA (Tech Preview — OCP 4.21)

Dynamic MIG allows the scheduler to request MIG partitions on-demand per `ResourceClaim`. Requires enabling the `DRAPartitionableDevices` feature gate (Tech Preview — breaks supportability boundary).

```bash
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
```

---

## Findings

### Finding 1: ResourceSlice — GPU Inventory Layer

> **Validated on:** OCP 4.21.42, GPU Operator v26.7.2, 2026-11-18

The NVIDIA DRA driver publishes **one ResourceSlice containing all GPU devices** per node (not one per chip as originally described). Each device entry contains structured attributes:

```bash
# Inspect a ResourceSlice
RS=$(oc get resourceslices -o jsonpath='{.items[0].metadata.name}')
oc get resourceslice ${RS} -o json | jq '{
  node: .spec.nodeName,
  driver: .spec.driver,
  devices: [.spec.devices[] | {name: .name, attributes: .attributes, capacity: .capacity}]
}'
```

**Actual H100 device attributes** (from `gpu.nvidia.com` driver on our cluster):

> **Correction from research:** Attribute keys do NOT use the `gpu.nvidia.com/` prefix in the actual ResourceSlice. They are top-level keys. Memory is in `.capacity.memory`, not in `.attributes`.

```json
{
  "name": "gpu-3",
  "attributes": {
    "addressingMode": {"string": "HMM"},
    "architecture":   {"string": "Hopper"},
    "brand":          {"string": "Nvidia"},
    "cudaComputeCapability": {"version": "9.0.0"},
    "cudaDriverVersion":     {"version": "13.0.0"},
    "driverVersion":         {"version": "580.105.8"},
    "productName":           {"string": "NVIDIA H100 80GB HBM3"},
    "resource.kubernetes.io/numaNode":  {"int": 0},
    "resource.kubernetes.io/pciBusID": {"string": "0000:c1:00.0"},
    "type": {"string": "gpu"},
    "uuid": {"string": "GPU-77459654-6751-15a3-20bd-439dcc9e1bdc"}
  },
  "capacity": {
    "memory": {"value": "81559Mi"}
  }
}
```

**CEL attribute access** (corrected from research):
```
# WRONG (research doc):  device.attributes["gpu.nvidia.com"].architecture
# CORRECT (validated):   device.attributes["architecture"].string
#                        device.capacity["memory"]
```

**GPUaaS billing implication**: ResourceSlices replace manual GPU counting. The entitlement controller reads ResourceSlice data to know what physical capacity exists — no polling `nvidia.com/gpu` node labels required.

### Finding 2: DeviceClass + CEL — Attribute-Based Scheduling

Admin-defined `DeviceClass` objects use CEL expressions evaluated by the scheduler against `ResourceSlice` device attributes. No node labels needed for scheduling decisions.

```bash
> **Validated CEL syntax (corrected from initial research):**
> - Attribute keys are **top-level** (no `gpu.nvidia.com/` prefix): `device.attributes["architecture"].string`
> - Memory is in **capacity**, not attributes: `device.capacity["memory"].value.compareTo(quantity("40000Mi")) >= 0`
> - Value types must match: `.string`, `.version`, `.int` depending on the attribute type in ResourceSlice

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
          device.attributes["architecture"].string == "Hopper"
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
          device.capacity["memory"].value.compareTo(quantity("40000Mi")) >= 0
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
          device.attributes["profile"].string == "1g.10gb"
EOF
```

> **Note**: Always verify attribute names against `oc get resourceslice <name> -o json | jq '.spec.devices[0]'` on your cluster before writing CEL expressions — the exact keys and types are driver-version-dependent.

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

> **Validated on:** OCP 4.21.42, 2026-11-18. Correct syntax confirmed.

```bash
# Correct firstAvailable syntax (validated): each item needs both 'name' and 'deviceClassName'
# WRONG: nesting under 'exactly' block — rejected by API
# CORRECT: flat structure with 'name' (RFC 1123 label) + 'deviceClassName' + 'count'
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
            deviceClassName: nvidia-h100
            count: 1
          - name: fallback-any
            deviceClassName: gpu-large-memory
            count: 1
EOF
```

**Validated result**: The DRA scheduler selected `prefer-h100` (confirmed in kubelet plugin logs: `[gpu/prefer-h100] ocp-h200-0 gpu-4`). The H100 was chosen over the fallback because it matched the `nvidia-h100` DeviceClass first.

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
                expression: 'device.attributes["profile"].string == "3g.40gb"'
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
                  device.attributes["profile"].string == "1g.10gb" &&
                  device.capacity["memory"].value.compareTo(quantity("9000Mi")) >= 0
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

**Current workaround**: none available via GPU Operator alone. `ClusterPolicy` (device plugin) and `GPUCluster` (DRA) are mutually exclusive — validated on v26.7.2. Running both simultaneously is not supported. Until DRA-in-Kueue ships, the choices are: (a) stay on ClusterPolicy + device plugin for Kueue quota support, or (b) switch to GPUCluster + DRA and lose Kueue quota accounting.

**Future state**: DRA-in-Kueue (upstream KEP in progress, no downstream ETA). When shipped, Kueue will natively manage `ResourceClaim`-based workloads and quotas.

---

## Proof Guide

### Step 1: Verify DRA Framework on Existing Cluster

```bash
source ~/.ibmcloud-h200-env
export KUBECONFIG="$INSTALL_DIR/auth/kubeconfig"

# DRA API group (built into API server — NOT CRDs, so oc get crd returns nothing)
oc api-resources | grep resource.k8s.io
# Expected: deviceclasses, resourceclaims, resourceclaimtemplates, resourceslices (all v1)

# No ResourceSlices yet (GPU Operator v25.10 uses device plugin, not DRA)
oc get resourceslices
# Expected: No resources found

# Create poc namespace
oc create namespace poc4-dra
```

### Step 2: Upgrade GPU Operator to v26.7

```bash
# Update subscription to v26.7 channel (GPUCluster CRD is in v26.7, not v26.3)
oc patch subscription gpu-operator-certified -n nvidia-gpu-operator \
  --type=merge -p '{"spec":{"channel":"v26.7"}}'

until oc get csv -n nvidia-gpu-operator 2>/dev/null | grep gpu-operator | grep "v26\.7" | grep -q Succeeded; do
  echo "  waiting..."; sleep 15
done
oc get csv -n nvidia-gpu-operator | grep gpu-operator
# Expected: gpu-operator-certified.v26.7.x   Succeeded
```

### Step 3: Migrate to GPUCluster

```bash
# Delete existing ClusterPolicy (removes nvidia.com/gpu resource)
oc delete clusterpolicy gpu-cluster-policy

# Create GPUCluster (DRA path) — spec validated against GPU Operator v26.7 schema
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

> **Correction from research:** CEL attribute access is `device.attributes["architecture"].string`, NOT `device.attributes["gpu.nvidia.com"].architecture`. Memory is in `device.capacity["memory"]`.

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
          device.attributes["architecture"].string == "Hopper"
EOF

# Any GPU with >= 40GB memory
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
          device.capacity["memory"].value.compareTo(quantity("40000Mi")) >= 0
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
          device.attributes["profile"].string == "1g.10gb"
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

## Validation Summary (2026-11-18)

| Check | Result | Notes |
|---|---|---|
| DRA framework on OCP 4.21.42 | ✅ `resource.k8s.io/v1` GA | Use `oc api-resources`, not `oc get crd` |
| GPU Operator version for GPUCluster | ✅ **v26.7** (not v26.3) | v26.3 has the CRD but GPUCluster is in v26.7 |
| ClusterPolicy + GPUCluster co-exist | ❌ Mutually exclusive | Error: "ClusterPolicy and GPUCluster cannot co-exist" |
| ResourceSlice published | ✅ 1 slice with 8 devices | One ResourceSlice per node (not per GPU chip) |
| H100 attributes in ResourceSlice | ✅ productName, architecture=Hopper, memory=81559Mi, cudaCC=9.0 | Attribute keys: no `gpu.nvidia.com/` prefix |
| DeviceClass with CEL selectors | ✅ Created and accepted | Attribute syntax: `device.attributes["architecture"].string` |
| ResourceClaim allocation | ✅ Allocated and pod ran nvidia-smi -L | UUID-matched GPU confirmed |
| `firstAvailable` syntax | ✅ `name` + `deviceClassName` + `count` (flat) | No `exactly` nesting |
| `firstAvailable` priority selection | ✅ `prefer-h100` chosen over fallback | Confirmed in DRA driver kubelet plugin logs |
| MIG Manager in GPUCluster | ❌ Not managed | MIG Manager is ClusterPolicy-only in v26.7 |
| Static MIG + DRA | 🔶 Not fully validated | Requires pre-existing MIG + GPUCluster active simultaneously |
| CDI library injection (toolkit-less) | ⚠️ Partial | Works for some claims; NVIDIA toolkit needed for reliable injection |

---

### Full Cleanup

```bash
oc delete pods --all -n poc4-dra 2>/dev/null || true
oc delete jobs --all -n poc4-dra 2>/dev/null || true
oc delete resourceclaims --all -n poc4-dra 2>/dev/null || true
oc delete resourceclaimtemplates --all -n poc4-dra 2>/dev/null || true
oc delete namespace poc4-dra --ignore-not-found
oc delete deviceclass nvidia-h100 nvidia-mig-1g10gb gpu-large-memory --ignore-not-found

# Revert to ClusterPolicy (restores nvidia.com/gpu, toolkit, MIG Manager, GFD)
oc delete gpucluster gpu-cluster 2>/dev/null || true
sleep 10
# Recreate ClusterPolicy — see ibmc/openshift-h200-provisioning.md Part D for full YAML
cat <<'EOF' | oc apply -f -
apiVersion: nvidia.com/v1
kind: ClusterPolicy
metadata:
  name: gpu-cluster-policy
spec:
  operator:
    defaultRuntime: crio
    use_ocp_driver_toolkit: true
  daemonsets: {}
  nodeStatusExporter:
    enabled: true
  driver:
    enabled: true
  toolkit:
    enabled: true
  devicePlugin:
    enabled: true
  dcgm:
    enabled: true
  dcgmExporter:
    enabled: true
    serviceMonitor:
      enabled: true
  gfd:
    enabled: true
  migManager:
    enabled: true
    config:
      name: default-mig-parted-config
  validator:
    plugin:
      env:
        - name: WITH_WORKLOAD
          value: "false"
EOF

until oc get clusterpolicy gpu-cluster-policy -o jsonpath='{.status.state}' 2>/dev/null | grep -q "ready"; do
  echo "  restoring..."; sleep 15
done
oc get node ocp-h200-0 -o jsonpath='{.status.allocatable}' | jq . | grep nvidia
# Expected: nvidia.com/gpu: 8
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
