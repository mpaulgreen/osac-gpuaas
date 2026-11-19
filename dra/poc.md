# DRA for GPUs on OpenShift — Research Notes

> **TL;DR:** DRA provides attribute-based GPU scheduling; MIG provides hardware isolation. Together they enable fine-grained multi-tenant GPU allocation without node-level label management. DRA is GA in OCP 4.21. The full DRA stack requires GPU Operator v26.7 with TWO CRs: `NvidiaDriver` (kernel driver, no NGC credentials — OCP DTK builds in-cluster) + `GPUCluster` (DRA driver + DCGM). Switching from `ClusterPolicy` to this stack is a one-way migration.

**GPUaaS connection**: DRA is the discovery and selection layer beneath the entitlement controller. ResourceSlices replace manual GPU counting. DeviceClasses map to SKU tiers. ResourceClaims are what workloads submit. MIG + DRA together replace the global node-label management that makes MIG operationally painful at scale.

---

## Prerequisites

- **OCP 4.21.42** (K8s 1.34) — DRA GA, `resource.k8s.io/v1`, `DynamicResourceAllocation` on by default. Whole-GPU DRA path validated here.
- **OCP 4.22.22** (K8s 1.35) — MIG+DRA path validated here. Requires `TechPreviewNoUpgrade` FeatureSet to unlock `DRAPartitionableDevices`.
- **GPU Operator v26.7** — required. Provides two CRDs: `GPUCluster` (DRA driver) and `NvidiaDriver` (kernel driver). Both are needed for the full DRA stack.
- **No NGC credentials needed** — `NvidiaDriver` CR uses OCP Driver Toolkit (DTK) to build the driver in-cluster from `quay.io/openshift-release-dev` (included in OCP pull secret). No `nvcr.io` access required.
- **Red Hat operator coverage**: DRA framework in OCP core (Red Hat). NVIDIA DRA driver in `gpu-operator-certified` v26.7 (certified-operators catalog — OLM-managed, no Helm).
- `ClusterPolicy` and `GPUCluster` are **mutually exclusive** — both are permanently irreversible once created; `ClusterPolicy` must be deleted before `GPUCluster` can be applied.
- **Known K8s 1.34 DRA bug**: 1.34.0 and 1.34.1 have a scheduler bug. Verify OCP patch level is 4.21.3+ before enabling.
- **MIG + DRA**: Static MIG geometry discovery and DynamicMIG both require `DRAPartitionableDevices` feature gate — NOT available in OCP 4.21 `TechPreviewNoUpgrade` (added in OCP 4.22). See OCP 4.22 Validation section.

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

> **Validated on:** OCP 4.21.42 / K8s 1.34 (whole-GPU path) and OCP 4.22.22 / K8s 1.35 (MIG+DRA path), GPU Operator v26.7.2, 2026-11-18

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

### Full DRA Stack: NvidiaDriver + GPUCluster (Validated — No NGC Credentials)

> **Official approach** per `docs.nvidia.com/datacenter/cloud-native/openshift/latest/dra-gpu-ocp.html`.
> Always extract YAML from the CSV `alm-examples` annotation — this gives the correct image digests.
> Do NOT hand-write the GPUCluster or NvidiaDriver spec.

```bash
CSV=$(oc get csv -n nvidia-gpu-operator | grep gpu-operator | awk '{print $1}')

# Step 1: Delete ClusterPolicy (required — they cannot co-exist)
oc delete clusterpolicy gpu-cluster-policy

# Step 2: Create NvidiaDriver CR (kernel driver — uses OCP DTK, no NGC credentials)
# The pinned sha256 digest in alm-examples triggers the OCP Driver Toolkit build path.
oc get csv -n nvidia-gpu-operator $CSV \
  -o jsonpath='{.metadata.annotations.alm-examples}' \
  | jq 'map(select(.kind == "NVIDIADriver")) | .[0]' \
  | oc apply -f -
# NvidiaDriver pod uses: quay.io/openshift-release-dev/ocp-v4.0-art-dev (OCP DTK)
# NOT nvcr.io/nvidia/driver — no NGC pull secret needed.

# Step 3: Create GPUCluster CR (DRA driver + DCGM — extracted from CSV for correct schema)
oc get csv -n nvidia-gpu-operator $CSV \
  -o jsonpath='{.metadata.annotations.alm-examples}' \
  | jq 'map(select(.kind == "GPUCluster")) | .[0]' \
  | oc apply -f -

# Step 4: Wait for both CRs to be ready
until oc get nvidiadriver gpu-driver -o jsonpath='{.status.state}' 2>/dev/null | grep -q "ready"; do
  echo "  waiting for NvidiaDriver..."; sleep 15
done
until oc get gpucluster gpu-cluster -o jsonpath='{.status.state}' 2>/dev/null | grep -q "ready"; do
  echo "  waiting for GPUCluster..."; sleep 15
done

# Step 5: Verify ResourceSlices
oc get resourceslices
# Expected: 1 ResourceSlice (whole-GPU mode) with 8 devices — one entry per H100 chip
```

**What GPUCluster manages** (CRD fields: `draDriver`, `dcgm`, `dcgmExporter`, `daemonsets`, `hostPaths`):
- DRA kubelet plugin (publishes `gpu.nvidia.com` and `mig.nvidia.com` DeviceClasses)
- DCGM and DCGM Exporter

**What GPUCluster does NOT manage** (still need NvidiaDriver or ClusterPolicy for these):
- Kernel driver daemonset → managed by `NvidiaDriver` CR
- Container toolkit → NOT deployed in DRA mode
- GPU Feature Discovery (GFD) → NOT deployed (NFD handles node labeling)
- MIG Manager → NOT deployed (DynamicMIG via DRA replaces it in OCP 4.22+)
- `nvidia.com/gpu` extended resource → absent (replaced by ResourceSlices)
- `nvidia` RuntimeClass → absent (toolkit creates it; toolkit not deployed)

**DeviceClasses auto-created by GPU Operator** (no need to create them manually):
```bash
oc get deviceclass
# gpu.nvidia.com                        — whole GPU chips
# mig.nvidia.com                        — MIG slice instances
# vfio.gpu.nvidia.com                   — VFIO passthrough
# compute-domain-daemon.nvidia.com      — multi-node NVLink (GB200+)
# compute-domain-default-channel.nvidia.com
```

**Library injection for pods needing nvidia-smi / NVML** (NvidiaDriver exposes libs at host path):

> `nvidia` RuntimeClass does not exist in DRA mode. Pods must mount libraries manually.
> Mount the full lib directory carefully — it can cause glibc symbol conflicts in some configs.

```yaml
# Add to any pod that needs nvidia-smi or NVML (e.g. monitoring, management tools)
volumeMounts:
  - name: nvidia-libs
    mountPath: /usr/local/nvidia/lib
env:
  - name: LD_LIBRARY_PATH
    value: /usr/local/nvidia/lib
volumes:
  - name: nvidia-libs
    hostPath:
      path: /run/nvidia/driver/usr/lib   # validated path: usr/lib (not usr/lib/x86_64-linux-gnu)
```

### MIG + DRA — OCP Version Requirements

> **Critical finding (validated 2026-11-18):** Both static MIG and DynamicMIG via DRA require
> the `DRAPartitionableDevices` Kubernetes feature gate. This gate is:
> - **OCP 4.21**: NOT in `TechPreviewNoUpgrade` (added in OCP 4.22)
> - **OCP 4.22+**: In `TechPreviewNoUpgrade` — enable with the FeatureGate below

**Consequence on OCP 4.21:** Even if MIG is configured in hardware before switching to GPUCluster,
the DRA driver does NOT publish `mig.nvidia.com` ResourceSlices. The DRA driver logs:
`"some fields were dropped by the apiserver, probably because these features are disabled: DRAPartitionableDevices"`

**On OCP 4.22:** Enable with:
```bash
oc patch featuregate cluster --type=merge -p '{"spec":{"featureSet":"TechPreviewNoUpgrade"}}'
# PERMANENT WARNING: TechPreviewNoUpgrade is irreversible once set (CEL validation rule).
# It cannot be changed back to "" or CustomNoUpgrade. It does block minor version upgrades.
```

**FeatureGate permanence gotcha (validated):**
Both `TechPreviewNoUpgrade` and `CustomNoUpgrade` are permanently irreversible once applied.
If `TechPreviewNoUpgrade` is already set (e.g. from a previous operation), you CANNOT switch
to `CustomNoUpgrade` to add `DRAPartitionableDevices` individually:
```
Error: spec.featureSet: Invalid value: "string": TechPreviewNoUpgrade may not be changed
```

**Version table:**

| OCP | K8s | `DRAPartitionableDevices` available | Enable via |
|---|---|---|---|
| 4.21 | 1.34 | No | `CustomNoUpgrade` only (but blocked if TechPreviewNoUpgrade already set) |
| **4.22** | **1.35** | **Yes** | `TechPreviewNoUpgrade` |
| 5.0+ | 1.36+ | Default | No gate needed |

**Static MIG + DRA (OCP 4.22+):**
```bash
# Pre-configure MIG while ClusterPolicy is still active (mig/poc.md)
# Then switch to NvidiaDriver + GPUCluster
# mig.nvidia.com ResourceSlices auto-appear from pre-configured MIG geometry
oc get resourceslices | grep mig
```

**DynamicMIG + DRA (OCP 4.22+):**
```bash
# Enable DynamicMIG in GPUCluster (requires DRAPartitionableDevices gate)
oc patch gpucluster gpu-cluster --type=merge -p '{
  "spec": {"draDriver": {"featureGates": {"DynamicMIG": true}, "gpus": {}}}
}'
# DRA driver then creates/destroys MIG partitions on-demand per ResourceClaim
# No pre-configured MIG geometry needed — DRAPartitionableDevices enables partitioning API
```

See the **OCP 4.22 Validation — Completed** section for full DynamicMIG validation steps.

---

## Findings

### Finding 1: ResourceSlice — GPU Inventory Layer

> **Validated on:** OCP 4.21.42, GPU Operator v26.7.2, 2026-11-18

The NVIDIA DRA driver publishes ResourceSlices per node. The count depends on MIG state:
- **MIG disabled (whole-GPU mode):** 1 ResourceSlice with 8 device entries (one per H100 chip)
- **MIG enabled in hardware (with `DRAPartitionableDevices`):** multiple ResourceSlices, one per GPU chip, each listing the chip's MIG instances as individual devices

Each device entry contains structured attributes:

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

**CEL attribute access** (K8s 1.34 / OCP 4.21, whole-GPU devices):
```
# WRONG:   device.attributes["gpu.nvidia.com"].architecture   ← domain lookup on flat map fails
# CORRECT: device.attributes["architecture"].string           ← flat map, explicit type accessor
#          device.capacity["memory"]                          ← memory is in capacity, not attributes
```

> **K8s 1.35 / OCP 4.22 — MIG devices use domain-keyed access (different structure):**
> `device.attributes['gpu.nvidia.com'].profile == '1g.10gb'`
> No `.string` accessor — K8s 1.35 auto-unwraps to native CEL types.
> See Finding 6 and the OCP 4.22 Validation section for details.

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
# Note: device.attributes['gpu.nvidia.com'].profile is the K8s 1.35 / OCP 4.22 syntax (domain-keyed)
# The has() guard is required — whole-GPU devices in the same ResourceSlice have no .profile attribute
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
          has(device.attributes['gpu.nvidia.com'].profile) &&
          device.attributes['gpu.nvidia.com'].profile == '1g.10gb'
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
                expression: "has(device.attributes['gpu.nvidia.com'].profile) && device.attributes['gpu.nvidia.com'].profile == '3g.40gb'"
          count: 1
EOF
```

Different workloads can claim different MIG profile sizes from the same node simultaneously — no node label changes, no drain.

### Finding 6: Dynamic MIG — On-Demand Partitioning (Tech Preview)

> **Validated on:** OCP 4.22.22 / K8s 1.35, GPU Operator v26.7.2, 2026-11-18

With `DRAPartitionableDevices` enabled (OCP 4.22 `TechPreviewNoUpgrade`), the DRA driver creates MIG partitions on-demand per ResourceClaim.

**CEL syntax (K8s 1.35 / OCP 4.22):**
- `device.attributes` is domain-keyed: `device.attributes['gpu.nvidia.com']` returns the NVIDIA attribute namespace
- `.profile` accesses the profile attribute within that namespace — no `.string` accessor (K8s 1.35 auto-unwraps)
- `has()` guard is required: whole-GPU devices in the same ResourceSlice (`gpu-7`, `gpu-6`, etc.) do NOT have a `.profile` attribute. Without `has()`, the CEL fails when evaluated against whole-GPU devices.

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
                  has(device.attributes['gpu.nvidia.com'].profile) &&
                  device.attributes['gpu.nvidia.com'].profile == '1g.10gb'
          count: 1
EOF
```

**Validated result (OCP 4.22.22, 2026-11-18):** DRA driver allocated `gpu-7-mig-1g10gb-19-0`. Pod ran with `nvidia-smi` showing MIG mode and ~9.5 GB memory. No pre-configured MIG geometry needed — DRA driver partitioned on-demand.

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

### Step 3: Migrate to NvidiaDriver + GPUCluster

**Always extract YAML from the CSV — do not hand-write the spec.**

```bash
CSV=$(oc get csv -n nvidia-gpu-operator | grep gpu-operator | awk '{print $1}')
echo "Using CSV: $CSV"

# Delete ClusterPolicy (required — mutually exclusive with GPUCluster)
oc delete clusterpolicy gpu-cluster-policy

# Apply NvidiaDriver CR (kernel driver — OCP DTK build, no NGC credentials)
oc get csv -n nvidia-gpu-operator $CSV \
  -o jsonpath='{.metadata.annotations.alm-examples}' \
  | jq 'map(select(.kind == "NVIDIADriver")) | .[0]' \
  | oc apply -f -

# Apply GPUCluster CR (DRA driver + DCGM)
oc get csv -n nvidia-gpu-operator $CSV \
  -o jsonpath='{.metadata.annotations.alm-examples}' \
  | jq 'map(select(.kind == "GPUCluster")) | .[0]' \
  | oc apply -f -

# Wait for both ready
until oc get nvidiadriver gpu-driver -o jsonpath='{.status.state}' 2>/dev/null | grep -q "ready"; do
  echo "  NvidiaDriver: waiting..."; sleep 15
done
echo "NvidiaDriver ready ✅"

until oc get gpucluster gpu-cluster -o jsonpath='{.status.state}' 2>/dev/null | grep -q "ready"; do
  echo "  GPUCluster: waiting..."; sleep 15
done
echo "GPUCluster ready ✅"
```

### Step 4: Verify ResourceSlices and DeviceClasses

```bash
# ResourceSlices (whole-GPU mode: 1 slice with 8 devices)
oc get resourceslices
# Expected: NAME starting with "00000-gpu.nvidia.com-ocp-h200-0-..."

# DeviceClasses (auto-created by GPU Operator — no manual creation needed)
oc get deviceclass
# Expected: gpu.nvidia.com, mig.nvidia.com, vfio.gpu.nvidia.com, compute-domain-*.nvidia.com

# Inspect device attributes (correct path: .attributes not .basic.attributes)
RS=$(oc get resourceslices -o jsonpath='{.items[0].metadata.name}')
oc get resourceslice ${RS} -o json | jq '{
  node: .spec.nodeName,
  driver: .spec.driver,
  deviceCount: (.spec.devices | length),
  firstDevice: .spec.devices[0] | {name, attributes, capacity}
}'
# Expected: deviceCount=8, architecture="Hopper", productName="NVIDIA H100 80GB HBM3"
```

### Step 5: Verify Auto-Created DeviceClasses (and Create Custom Ones if Needed)

GPU Operator auto-creates `gpu.nvidia.com`, `mig.nvidia.com`, and other DeviceClasses. You can
create additional ones with custom CEL selectors for GPUaaS tiers:

```bash
oc get deviceclass
# Already present: gpu.nvidia.com, mig.nvidia.com, vfio.gpu.nvidia.com, compute-domain-*.nvidia.com

# Optional: custom DeviceClasses for specific tiers
# I1 — whole H100 (Serve) with Hopper architecture filter
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

# I0 — MIG 1g.10gb slice (Sandbox) — requires DRAPartitionableDevices (OCP 4.22+)
# Uses domain-keyed CEL syntax (K8s 1.35). has() guard needed — whole-GPU devices have no .profile.
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
          has(device.attributes['gpu.nvidia.com'].profile) &&
          device.attributes['gpu.nvidia.com'].profile == '1g.10gb'
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

# Schedule a pod against the claim — with library mount (NvidiaDriver exposes libs at host path)
# Without the mount, nvidia-smi fails: "NVIDIA-SMI couldn't find libnvidia-ml.so"
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
      command: ["sh", "-c", "nvidia-smi -L && echo 'DRA allocation SUCCESS'"]
      resources:
        claims:
          - name: gpu
      volumeMounts:
        - name: nvidia-libs
          mountPath: /usr/local/nvidia/lib
      env:
        - name: LD_LIBRARY_PATH
          value: /usr/local/nvidia/lib
  resourceClaims:
    - name: gpu
      resourceClaimName: h100-claim
  volumes:
    - name: nvidia-libs
      hostPath:
        path: /run/nvidia/driver/usr/lib   # NvidiaDriver daemonset exposes this
  restartPolicy: Never
EOF

sleep 30
oc get pod dra-test -n poc4-dra
oc logs -n poc4-dra dra-test
# Expected: GPU 0: NVIDIA H100 80GB HBM3 (UUID: ...)
# Expected: DRA allocation SUCCESS

# Verify claim has allocation
oc get resourceclaim h100-claim -n poc4-dra \
  -o jsonpath='{.status.allocation}' | jq .
```

### Step 7: MIG + DRA — Requires OCP 4.22 (DRAPartitionableDevices)

> **OCP 4.21 limitation:** `mig.nvidia.com` ResourceSlices do NOT appear on OCP 4.21 even when
> MIG is configured in hardware. The DRA driver requires `DRAPartitionableDevices` gate to publish
> MIG partition information. This gate is only in `TechPreviewNoUpgrade` on OCP 4.22+.
> See **OCP 4.22 Validation — Completed** section for full results.

On OCP 4.22 (after enabling `TechPreviewNoUpgrade`):
```bash
# Verify MIG slices appear as DRA devices
# On K8s 1.35, MIG device attributes are domain-keyed in JSON: .attributes["gpu.nvidia.com"]["profile"]
oc get resourceslices -o json | jq '
  .items[] | select(.spec.driver=="mig.nvidia.com") | {
    node: .spec.nodeName,
    deviceCount: (.spec.devices | length),
    slices: [.spec.devices[] | {
      name: .name,
      profile: .attributes["gpu.nvidia.com"]["profile"].string
    }]
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
| NvidiaDriver CR (no NGC) | ✅ OCP DTK build | `quay.io/openshift-release-dev` — OCP pull secret sufficient |
| NvidiaDriver ready | ✅ | Driver version 595.91.7 installed via DTK |
| GPUCluster + NvidiaDriver co-exist | ✅ | Both CRs active simultaneously |
| ResourceSlice published | ✅ 1 slice with 8 devices | whole-GPU mode: 1 ResourceSlice per node |
| H100 attributes in ResourceSlice | ✅ productName, architecture=Hopper, memory=81559Mi, cudaCC=9.0 | Attribute keys: no `gpu.nvidia.com/` prefix; memory in `.capacity` |
| DeviceClasses auto-created | ✅ | gpu.nvidia.com, mig.nvidia.com, vfio.gpu.nvidia.com — no manual creation needed |
| DeviceClass with custom CEL selectors | ✅ Created and accepted | `device.attributes["architecture"].string` |
| ResourceClaim allocation (whole GPU) | ✅ Allocated, pod ran nvidia-smi -L | UUID-matched GPU confirmed |
| Library injection via NvidiaDriver | ✅ `/run/nvidia/driver/usr/lib` host path mount | nvidia-smi SUCCESS confirmed in DRA pod |
| `firstAvailable` syntax | ✅ `name` + `deviceClassName` + `count` (flat) | No `exactly` nesting |
| `firstAvailable` priority selection | ✅ `prefer-h100` chosen over fallback | Confirmed in DRA driver kubelet plugin logs |
| MIG Manager in GPUCluster | ❌ Not managed | MIG Manager is ClusterPolicy-only |
| `mig.nvidia.com` ResourceSlices (MIG+DRA) | ❌ OCP 4.21 | Requires `DRAPartitionableDevices` gate — only in OCP 4.22 TechPreviewNoUpgrade |
| DynamicMIG | ❌ OCP 4.21 | `DRAPartitionableDevices` not in OCP 4.21 TechPreviewNoUpgrade |
| FeatureGate permanence gotcha | ⚠️ | TechPreviewNoUpgrade AND CustomNoUpgrade both permanently irreversible once set |
| nvidia RuntimeClass | ❌ Not present in DRA mode | Toolkit creates it; toolkit not deployed in GPUCluster path |

**OCP 4.22.22 / K8s 1.35 results (validated 2026-11-18):**

| Check | Result | Notes |
|---|---|---|
| `TechPreviewNoUpgrade` enables `DRAPartitionableDevices` | ✅ OCP 4.22 | kube-apiserver rollout ~5 min; DRA driver logs no longer show "fields dropped" |
| `mig.nvidia.com` ResourceSlices with DynamicMIG | ✅ OCP 4.22 | 26 devices per chip: whole GPU + all MIG profile combinations + `consumesCounters` |
| DynamicMIG: on-demand partition for `1g.10gb` | ✅ OCP 4.22 | Allocated `gpu-7-mig-1g10gb-19-0`; pod output `POD-1G SUCCESS` |
| DynamicMIG: simultaneous `1g.10gb` + `2g.20gb` isolation | ✅ OCP 4.22 | GI 19-0 vs GI 14-2; different memory (~9.5 GB vs ~19.5 GB); no node label changes |
| DynamicMIG teardown | ✅ OCP 4.22 | `nvidia-smi mig -lgi` empty after claim deletion |
| Correct MIG CEL syntax on K8s 1.35 | ✅ | `device.attributes['gpu.nvidia.com'].profile == '1g.10gb'` (domain-keyed, no `.string`) |
| `has()` guard required for profile | ✅ | Whole-GPU devices (`gpu-7`, `gpu-6`...) have no `.profile` → guard prevents CEL panic |
| Static MIG + DRA (node label → ResourceSlice) | ✅ Finding confirmed | `mig.nvidia.com` ResourceSlices require `DynamicMIG: true` — no separate static mode exists. DynamicMIG allocates from existing geometry when available. |
| ClusterPolicy restore (Full Cleanup) | ✅ OCP 4.22 | `nvidia.com/gpu: 8`, MIG `all-disabled`, ClusterPolicy `ready` |

---

### Full Cleanup

> **Validated on:** OCP 4.22.22, 2026-11-19. Result: `nvidia.com/gpu: 8`, MIG `all-disabled`, ClusterPolicy `ready`.

```bash
oc delete pods --all -n poc4-dra 2>/dev/null || true
oc delete jobs --all -n poc4-dra 2>/dev/null || true
oc delete resourceclaims --all -n poc4-dra 2>/dev/null || true
oc delete resourceclaimtemplates --all -n poc4-dra 2>/dev/null || true
oc delete namespace poc4-dra --ignore-not-found
oc delete deviceclass nvidia-h100 nvidia-mig-1g10gb gpu-large-memory --ignore-not-found

# Revert to ClusterPolicy (restores nvidia.com/gpu, toolkit, MIG Manager, GFD)
oc delete gpucluster gpu-cluster 2>/dev/null || true
oc delete nvidiadriver gpu-driver 2>/dev/null || true
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

## OCP 4.22 Validation — Completed

> **Status:** All items validated on OCP 4.22.22 / K8s 1.35, GPU Operator v26.7.2, 2026-11-18.
> These validations could not be done on OCP 4.21 because `DRAPartitionableDevices` is not in
> OCP 4.21's `TechPreviewNoUpgrade` catalog. A fresh OCP 4.22 cluster was provisioned.

### Enable on OCP 4.22

On OCP 4.22, `DRAPartitionableDevices` is included in `TechPreviewNoUpgrade`:

```bash
oc patch featuregate cluster --type=merge -p '{"spec":{"featureSet":"TechPreviewNoUpgrade"}}'
# PERMANENT: TechPreviewNoUpgrade is irreversible. It cannot be changed back and blocks minor
# version upgrades. Apply only after the cluster is provisioned and stable.
# Triggers rolling restart of kube-apiserver and kube-scheduler (~5 min)
until oc get clusteroperator kube-apiserver -o jsonpath='{.status.conditions[?(@.type=="Progressing")].status}' | grep -q True; do sleep 5; done
until oc get clusteroperator kube-apiserver -o jsonpath='{.status.conditions[?(@.type=="Progressing")].status}' | grep -q False; do sleep 15; done
echo "kube-apiserver rollout complete"
```

Verify DRA driver no longer drops DRAPartitionableDevices fields:
```bash
oc logs -n nvidia-gpu-operator -l app=nvidia-dra-driver-kubelet-plugin --tail=10 | grep -i "dropped\|DRAPartition"
# Expected: no matches
```

### 1. DynamicMIG — On-Demand MIG Partitioning ✅

Enable DynamicMIG in GPUCluster and submit a `mig.nvidia.com` ResourceClaim:
```bash
oc patch gpucluster gpu-cluster --type=merge -p \
  '{"spec":{"draDriver":{"featureGates":{"DynamicMIG":true},"gpus":{}}}}'
# Wait for GPU Operator to restart DRA driver pod (~30s)

# Label namespace to allow privileged pods (DRA + hostPath volumes need this)
oc label namespace poc4-dra \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/warn=privileged \
  pod-security.kubernetes.io/audit=privileged --overwrite

cat <<'EOF' | oc apply -f -
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: mig-pod-1g
  namespace: poc4-dra
spec:
  devices:
    requests:
      - name: mig-slice
        exactly:
          deviceClassName: mig.nvidia.com
          selectors:
            - cel:
                expression: "has(device.attributes['gpu.nvidia.com'].profile) && device.attributes['gpu.nvidia.com'].profile == '1g.10gb'"
          count: 1
---
apiVersion: v1
kind: Pod
metadata:
  name: mig-pod-1g
  namespace: poc4-dra
spec:
  containers:
    - name: cuda
      image: nvcr.io/nvidia/cuda:12.6.3-base-ubi9
      command: ["sh", "-c", "nvidia-smi && echo 'POD-1G SUCCESS'"]
      resources:
        claims:
          - name: mig-slice
      volumeMounts:
        - name: nvidia-libs
          mountPath: /usr/local/nvidia/lib
      env:
        - name: LD_LIBRARY_PATH
          value: /usr/local/nvidia/lib
  resourceClaims:
    - name: mig-slice
      resourceClaimName: mig-pod-1g
  volumes:
    - name: nvidia-libs
      hostPath:
        path: /run/nvidia/driver/usr/lib
  restartPolicy: Never
EOF
```

**Validated result:** DRA driver allocated `gpu-7-mig-1g10gb-19-0`. Pod completed with `POD-1G SUCCESS`. `nvidia-smi` showed MIG mode enabled, ~9.5 GB memory. No pre-configured MIG geometry was needed.

### 2. Two-Pod Simultaneous MIG Isolation ✅

Submit `1g.10gb` and `2g.20gb` claims simultaneously:
```bash
cat <<'EOF' | oc apply -f -
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: mig-pod-2g
  namespace: poc4-dra
spec:
  devices:
    requests:
      - name: mig-slice
        exactly:
          deviceClassName: mig.nvidia.com
          selectors:
            - cel:
                expression: "has(device.attributes['gpu.nvidia.com'].profile) && device.attributes['gpu.nvidia.com'].profile == '2g.20gb'"
          count: 1
---
apiVersion: v1
kind: Pod
metadata:
  name: mig-pod-2g
  namespace: poc4-dra
spec:
  containers:
    - name: cuda
      image: nvcr.io/nvidia/cuda:12.6.3-base-ubi9
      command: ["sh", "-c", "nvidia-smi && echo 'POD-2G SUCCESS'"]
      resources:
        claims:
          - name: mig-slice
      volumeMounts:
        - name: nvidia-libs
          mountPath: /usr/local/nvidia/lib
      env:
        - name: LD_LIBRARY_PATH
          value: /usr/local/nvidia/lib
  resourceClaims:
    - name: mig-slice
      resourceClaimName: mig-pod-2g
  volumes:
    - name: nvidia-libs
      hostPath:
        path: /run/nvidia/driver/usr/lib
  restartPolicy: Never
EOF

# Monitor both pods
oc get pods -n poc4-dra -w
```

**Validated result:**
- `mig-pod-1g` → `gpu-7-mig-1g10gb-19-0` (GI 19-0, ~9.5 GB)
- `mig-pod-2g` → `gpu-7-mig-2g20gb-14-2` (GI 14-2, ~19.5 GB)
- Both pods `Completed`. Hardware isolation confirmed: different GI IDs, different profiles.
- No node label changes were needed during the entire test.

### 3. DynamicMIG Teardown ✅

After pods complete, verify MIG instances are destroyed:
```bash
# Find the NvidiaDriver pod (not the GPU Operator pod)
DRIVER_POD=$(oc get pods -n nvidia-gpu-operator -l app=nvidia-driver-daemonset -o jsonpath='{.items[0].metadata.name}')

oc exec -n nvidia-gpu-operator $DRIVER_POD -c nvidia-driver-ctr -- nvidia-smi mig -lgi
# Expected: No MIG instances — DRA driver destroyed partitions after claim deletion
```

**Validated result:** `nvidia-smi mig -lgi` returned empty after `mig-pod-1g` and `mig-pod-2g` pods completed and their claims were released.

### 4. Static MIG + DRA — Key Finding ✅ Validated (Concept Clarified)

> **Validated on:** OCP 4.22.22 / K8s 1.35, GPU Operator v26.7.2, 2026-11-19

**Finding:** `mig.nvidia.com` ResourceSlices only appear when `DynamicMIG: true` is set in GPUCluster.
Without DynamicMIG, only whole-GPU `gpu.nvidia.com` devices appear — even if hardware has MIG instances configured.

This was confirmed by applying the default GPUCluster CR (DynamicMIG=false) while the hardware had a full mixed MIG layout (1g.10gb + 2g.20gb + 3g.40gb on each chip). Result: only 8 `gpu.nvidia.com` devices published, no `mig.nvidia.com` ResourceSlices at all.

```bash
# Test: GPUCluster without DynamicMIG + hardware with MIG instances
oc get resourceslices
# ACTUAL result: only gpu.nvidia.com ResourceSlice (8 whole-GPU devices)
# NO mig.nvidia.com ResourceSlices despite active hardware MIG partitions

oc get resourceslices -o json | jq '.items[] | select(.spec.driver=="mig.nvidia.com")'
# ACTUAL result: empty — DRAPartitionableDevices alone is not sufficient
# DynamicMIG: true in GPUCluster is required
```

**Conclusion:** There is no separate "Static MIG + DRA" mode. `DynamicMIG: true` is required for all MIG+DRA use cases. The DRA driver only publishes `mig.nvidia.com` ResourceSlices (and the partitionable device capability) when DynamicMIG is explicitly enabled.

**MIG + DRA modes (corrected):**

| Mode | DynamicMIG flag | MIG ResourceSlices | Behavior |
|---|---|---|---|
| Whole-GPU only | `false` (default) | ❌ | 8 × `gpu.nvidia.com` devices |
| MIG + DRA | `true` | ✅ 26 per chip | All profile combos + `consumesCounters`; driver creates partitions on-demand |

**Operational implication:** When DynamicMIG is enabled, the driver will allocate from existing hardware MIG partitions if they already match the requested profile. It only creates new partitions when no matching geometry exists. Node labels (via ClusterPolicy + MIG Manager) can pre-configure the hardware geometry, which DynamicMIG will then manage — but DynamicMIG must be on for ResourceSlices to publish.

### 5. OCP Version Roadmap ✅

| OCP | K8s | `DRAPartitionableDevices` | How to Enable |
|---|---|---|---|
| 4.21 | 1.34 | Not in TechPreview | `CustomNoUpgrade` only — blocked if `TechPreviewNoUpgrade` already set |
| **4.22** | **1.35** | **In TechPreviewNoUpgrade** ✅ | `oc patch featuregate cluster --type=merge -p '{"spec":{"featureSet":"TechPreviewNoUpgrade"}}'` |
| 5.0+ | 1.36+ | Default-on | No gate needed |

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
