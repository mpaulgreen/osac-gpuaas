# DRA Phase 2 — Exhaustive Proof-Out and Deep Dive

> **TL;DR:** Phase 1 ([`poc.md`](./poc.md)) proved the DRA *fundamentals* work on this cluster —
> ResourceSlices, DeviceClasses + CEL, ResourceClaims, `firstAvailable`, and DynamicMIG. Phase 2
> builds the *mental model* and pushes on everything Phase 1 did not touch: **GPU sharing**
> (time-slicing / MPS), **multi-tenant quota & isolation** (ResourceQuota, admin access,
> no-preemption), **multi-GPU allocation with constraints**, and the **extended-resource / Kueue
> bridge**. It separates, explicitly, what is **GA/beta and runnable on this locked cluster** from
> what is **alpha and blocked here** — with the exact manifests and the Kubernetes 1.36/1.37
> graduation path for the blocked pieces.

**GPUaaS connection**: Phase 1 answered "can the scheduler pick the right device?". Phase 2 answers
the operator questions: "can two tenants safely *share* one GPU, can I *cap* a tenant's consumption,
can I allocate *multiple* GPUs with topology constraints, and how does DRA coexist with the classic
`nvidia.com/gpu` world the rest of the stack (Kueue, RHOAI) still speaks?"

---

## Cluster under test (Phase 2)

| | |
|---|---|
| OCP / K8s | **4.22.22 / v1.35.9** |
| FeatureSet | **`TechPreviewNoUpgrade`** (locked, irreversible) |
| GPU Operator | **v26.7.3** (certified-operators) |
| GPU node | 1 × `gx3d-160x1792x8h100` — 8 × H100 80GB |
| DRA stack | `NVIDIADriver` (DTK) + `GPUCluster` (`draDriver.featureGates.DynamicMIG=true`) |

> **Login:** `source ~/.ibmcloud-h200-env && export KUBECONFIG=~/ocp-h200-install/auth/kubeconfig`

---

## 1. The DRA mental model

Five ideas carry the whole system. If these click, every manifest in this doc reads naturally.

**1. Drivers publish inventory; the scheduler shops from it.** A DRA driver (here the NVIDIA DRA
driver, `gpu.nvidia.com` + `mig.nvidia.com`) enumerates the hardware it manages on each node and
writes it into **`ResourceSlice`** objects — one logical *device* per entry, each carrying structured
**attributes** (architecture, product, UUID, MIG `profile`, …) and **capacity** (memory, engines).
This is the *only* place the scheduler learns what exists. No node labels, no device-plugin counts.

**2. Admins define categories; CEL is the query language.** A **`DeviceClass`** is a named category
with CEL *selectors* over device attributes — "any `Hopper` GPU", "a `mig.nvidia.com` device whose
`profile == '1g.10gb'`". DeviceClasses are cluster-scoped and authored by the admin/driver.

**3. Workloads request; they never name hardware.** A **`ResourceClaim`** asks for devices by
`DeviceClass` (+ optional per-request CEL), with a `count` or `allocationMode: All`. A
**`ResourceClaimTemplate`** stamps out one claim *per pod* (garbage-collected with the pod) — the
right choice for Deployments/Jobs; a standalone `ResourceClaim` is shared and lives until deleted.

**4. Allocation happens at pod-scheduling time, atomically, and is not preemptible.** There is no
"immediate" allocation in structured-parameters DRA — the scheduler picks a node that satisfies *all*
of a pod's claims at once (delayed / `WaitForFirstConsumer`), writes `status.allocation` on the
claim, and records the pod in `status.reservedFor` (max **256** consumers per claim). **The scheduler
does not preempt DRA devices** — a higher-priority pod needing a busy GPU stays `Pending` until it
frees. This is the single most important fairness fact for multi-tenant design (see §B4).

**5. Vendor knobs ride in opaque config.** Anything Kubernetes doesn't understand — sharing strategy,
MPS thread %, MIG hints — travels in a `config[].opaque` block keyed to the driver
(`driver: gpu.nvidia.com`, `kind: GpuConfig`). Kube passes it through untouched; the driver
interprets it and wires the device into the container via CDI.

```
  Admin ──defines──▶ DeviceClass (CEL)           Driver ──publishes──▶ ResourceSlice (devices+attrs)
                          │                                                   │
  Tenant ──submits──▶ ResourceClaim ──┐                                       │
       (DeviceClass + CEL + count)    ▼                                       ▼
                               ┌──────────────── kube-scheduler ─────────────────┐
                               │  evaluate CEL over slice devices, check counters │
                               │  pick node satisfying ALL pod claims (atomic)    │
                               │  write claim.status.allocation + reservedFor     │
                               └───────────────────────┬──────────────────────────┘
                                                       ▼
                                   kubelet + DRA driver prepare device (CDI) ─▶ Pod runs
```

**Lifecycle corollaries.** Template-generated claims are deleted when the pod ends; a standalone
claim persists until the last reservation clears. When draining a node, **drain the DRA driver
last** — drained before the claim-holding pods, it cannot *un-prepare* devices.

---

## 2. Feature-gate maturity — and what is actually on *this* cluster

DRA is not one switch. It is a GA *core* plus a dozen sub-features at different maturities, split
across **two gate systems**: Kubernetes feature gates (set by the OCP FeatureSet) and **NVIDIA DRA
driver** gates (set via `GPUCluster.spec.draDriver.featureGates`, rendered as the driver's
`FEATURE_GATES` env var).

### Kubernetes DRA gates (K8s 1.35 = OCP 4.22)

| Gate / capability | Stage in 1.35 | On this cluster | Why it matters here |
|---|---|---|---|
| `DynamicResourceAllocation` (core API `resource.k8s.io/v1`) | **GA, locked-on** | ✅ | the whole framework |
| `DRAAdminAccess` (privileged inspect of in-use devices) | **Beta, on** | ✅ | operator troubleshooting (§B3) |
| `DRAPrioritizedList` (`firstAvailable`) | **Beta, on** | ✅ | graceful degradation (P1 Finding 4) |
| `DRAResourceClaimDeviceStatus` (per-device status) | **Beta, on** | ✅ | observability |
| `DRAPartitionableDevices` (counters → DynamicMIG) | **Alpha** | ✅ (in TechPreview) | MIG-on-DRA (P1 Finding 6) |
| `DRADeviceTaints` / `DRADeviceTaintRules` (drain a device) | **Alpha** | ❌ **off, unreachable** | would drain one GPU/slice (§B5) |
| `DRAExtendedResource` (classic `resources.limits` ↔ DRA) | **Alpha** | ❌ **off, unreachable** | the Kueue bridge (§D) |
| `DRAConsumableCapacity` (fractional device share) | **Alpha** | ❌ **off, unreachable** | independent-pod sharing (§A4) |
| `DRADeviceBindingConditions` (wait-for-ready bind) | **Alpha** | ❌ off, unreachable | fabric-attached devices (N/A for local GPUs) |

> **Why "unreachable" and not just "off":** enabling an individual alpha gate requires the
> `CustomNoUpgrade` FeatureSet. This cluster is on `TechPreviewNoUpgrade`, which is **irreversible**
> and **cannot be switched** to `CustomNoUpgrade` (`spec.featureSet ... may not be changed`). So the
> alpha gates *not* bundled into `TechPreviewNoUpgrade` (everything except `DRAPartitionableDevices`)
> can never be turned on here. Those capabilities are **documented with manifests + roadmap**, not
> executed. Verified live:
> ```
> $ oc get featuregate cluster -o jsonpath='{.status.featureGates[0].enabled[*].name}' | tr ' ' '\n' | grep DRA
> DRAPartitionableDevices        # the ONLY DRA gate TechPreviewNoUpgrade turns on
> ```

### NVIDIA DRA driver gates (`GPUCluster.spec.draDriver.featureGates`)

| Driver gate | Stage | Set here | Notes / exclusivity |
|---|---|---|---|
| (base) whole-GPU + existing-MIG allocation | **GA** | always on | — |
| time-slicing via shared claim | **GA** | always on | whole-GPU only; **no-op on MIG** |
| `DynamicMIG` | alpha | **true** | needs K8s `DRAPartitionableDevices`; **⊥ MPSSupport** |
| `MPSSupport` | alpha | attempted | **⊥ DynamicMIG**; **⊥ time-slicing**; rejected under `ConsumableShares` |
| `TimeSlicingSettings` (custom interval) | alpha | attempted | default interval needs no gate |
| `ConsumableShares` | alpha | ❌ | requires K8s `DRAConsumableCapacity` (unreachable here) → blocked |
| `PassthroughSupport` (VFIO), `NVMLDeviceHealthCheck` | alpha | ❌ | both ⊥ DynamicMIG; out of scope |

**Mutual-exclusivity map (record as first-class results — §10):**
`DynamicMIG ⊥ {MPSSupport, PassthroughSupport, NVMLDeviceHealthCheck}` · `MPS ⊥ time-slicing`
(MPS needs `EXCLUSIVE_PROCESS`, time-slicing needs `DEFAULT`) · time-slicing does nothing on MIG
devices · MPS is rejected when `ConsumableShares` is active.

---

## 3. Pre-flight — migrate to the DRA stack

> **Validated on:** OCP 4.22.22 / v1.35.9, GPU Operator v26.7.3, 2026-11-19. The cluster had been
> returned to ClusterPolicy (device-plugin) by Phase 1's cleanup; a leftover `nvidia-mig-2g20gb`
> DeviceClass remained.

```bash
CSV=$(oc get csv -n nvidia-gpu-operator | grep gpu-operator | awk '{print $1}')   # gpu-operator-certified.v26.7.3

# 1. Remove the leftover custom DeviceClass from Phase 1
oc delete deviceclass nvidia-mig-2g20gb --ignore-not-found

# 2. Delete ClusterPolicy — mutually exclusive with GPUCluster
oc delete clusterpolicy gpu-cluster-policy

# 3. NVIDIADriver (kernel driver via OCP Driver Toolkit — no NGC credentials)
oc get csv -n nvidia-gpu-operator "$CSV" -o jsonpath='{.metadata.annotations.alm-examples}' \
  | jq 'map(select(.kind=="NVIDIADriver"))[0]' | oc apply -f -

# 4. GPUCluster with DynamicMIG on from the start
#    (draDriver.featureGates is a settable map[string]bool → rendered as FEATURE_GATES on the driver)
oc get csv -n nvidia-gpu-operator "$CSV" -o jsonpath='{.metadata.annotations.alm-examples}' \
  | jq 'map(select(.kind=="GPUCluster"))[0]
        | .spec.draDriver.featureGates = {"DynamicMIG": true}
        | .spec.draDriver.gpus = {}' \
  | oc apply -f -
```

**Observed:** `NVIDIADriver gpu-driver` reported `ready` almost immediately (the DTK driver image was
cached from Phase 1 — no 5–15 min rebuild). Effective driver spec:

```json
{"computeDomains":{"enabled":true},"featureGates":{"DynamicMIG":true},"gpus":{}}
```

**ResourceSlices published** (GPUCluster `ready`, ~4 min to converge):

```
$ oc get pods -n nvidia-gpu-operator | grep dra
nvidia-dra-driver-controller-...        1/1  Running
nvidia-dra-driver-kubelet-plugin-...    2/2  Running
nvidia-dra-validator-...                1/1  Running

$ oc get deviceclass
gpu.nvidia.com   mig.nvidia.com   vfio.gpu.nvidia.com
compute-domain-daemon.nvidia.com   compute-domain-default-channel.nvidia.com

# ResourceSlices (driver gpu.nvidia.com): 8 chips × 26 devices each  ← Phase 1's "26/chip" CONFIRMED
#   + 1 compute-domain.nvidia.com slice (2 devices)
```

The two DeviceClasses differ only by the device `type` attribute (both are published by driver
`gpu.nvidia.com`):

```
gpu.nvidia.com  selector: device.attributes['gpu.nvidia.com'].type == 'gpu'
mig.nvidia.com  selector: device.attributes['gpu.nvidia.com'].type == 'mig'
```

Each chip publishes **26 logical devices = 1 whole-GPU (`gpu-N`) + 25 MIG placements**, all drawing
from one per-chip counter set (`gpu-N-counter-set`). See §8 for the counter deep dive.

---

## A. GPU sharing

The headline Phase-2 question: can two tenants share **one physical GPU**? Three mechanisms exist and
they are *not* interchangeable. The dividing line: **time-slicing and MPS share a device only across
containers of the *same* `ResourceClaim`**; the only way *independent* claims/pods share one GPU is
**consumable capacity** (blocked here — §A3).

### A1. Time-slicing — two containers, one GPU ✅ (GA mechanism; gated on this build)

A single whole-GPU claim carries a `GpuConfig` opaque config; two containers reference the same
request and are multiplexed onto the one chip by the time-sliced CUDA context.

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata: { name: ts-shared-gpu, namespace: poc-dra-phase2 }
spec:
  devices:
    requests:
      - name: gpu
        exactly: { deviceClassName: gpu.nvidia.com, count: 1 }
    config:
      - requests: ["gpu"]
        opaque:
          driver: gpu.nvidia.com
          parameters:
            apiVersion: resource.nvidia.com/v1beta1
            kind: GpuConfig
            sharing:
              strategy: TimeSlicing
              timeSlicingConfig: { interval: Medium }   # Default | Short | Medium | Long
---
# Pod: two containers, each resources.claims:[{name: gpu, request: gpu}]  (same request → same device)
```

**Validated result:** both containers reported the **same** physical GPU —
```
C0: GPU 0: NVIDIA H100 80GB HBM3 (UUID: GPU-e985ba31-e771-5eb0-b376-a5db8e6c78b0)
C1: GPU 0: NVIDIA H100 80GB HBM3 (UUID: GPU-e985ba31-e771-5eb0-b376-a5db8e6c78b0)
```
Claim allocated device `gpu-7` (whole chip), request `gpu`, driver `gpu.nvidia.com`.

> **⚠️ Correction to the public docs (validated 2026-11-19):** NVIDIA's docs describe *default-interval*
> time-slicing as GA with **no** feature gate, and only *custom intervals* needing the alpha
> `TimeSlicingSettings` gate. On this build (DRA driver v0.5.x / GPU Operator **v26.7.3**), **any**
> `strategy: TimeSlicing` — default interval included — fails preparation without the gate:
> ```
> FailedPrepareDynamicResources: error validating GPU config: "TimeSlicing" is selected as the GPU
> sharing strategy, but the "TimeSlicingSettings" feature gate is not enabled
> ```
> Enable it via the GPUCluster (coexists with DynamicMIG — not in its exclusivity set):
> ```bash
> oc patch gpucluster gpu-cluster --type=merge \
>   -p '{"spec":{"draDriver":{"featureGates":{"DynamicMIG":true,"TimeSlicingSettings":true}}}}'
> # driver rolls; new kubelet-plugin pod shows FEATURE_GATES=DynamicMIG=true,TimeSlicingSettings=true
> ```
> With the gate on, both default and custom (`Medium`) intervals prepared and ran.

**Isolation note:** time-slicing is *temporal* sharing of the full GPU — **no memory or fault
isolation** between the two contexts. It is a density/utilization lever for cooperative or
same-tenant workloads, **not** a multi-tenant isolation boundary (that is MIG). It is also a **no-op
on MIG devices** (whole-GPU only).

### A2. MPS — two containers, one GPU, partitioned threads & memory ✅ (alpha gate; ⊥ DynamicMIG)

MPS (Multi-Process Service) gives *spatial* sharing: concurrent CUDA contexts on one GPU with a
thread-percentage cap and a pinned-memory limit per client — stronger multiplexing than time-slicing.
It is an **alpha** driver gate (`MPSSupport`) and **mutually exclusive with DynamicMIG** (and with
time-slicing). To run it we flipped the driver gates:

```bash
oc patch gpucluster gpu-cluster --type=merge \
  -p '{"spec":{"draDriver":{"featureGates":{"DynamicMIG":false,"TimeSlicingSettings":false,"MPSSupport":true}}}}'
# driver rolls → FEATURE_GATES=DynamicMIG=false,MPSSupport=true,TimeSlicingSettings=false
# ResourceSlices collapse from 9 slices / 26-per-chip back to 1 slice / 8 whole-GPU devices
```

```yaml
# claim config
config:
  - requests: ["gpu"]
    opaque:
      driver: gpu.nvidia.com
      parameters:
        apiVersion: resource.nvidia.com/v1beta1
        kind: GpuConfig
        sharing:
          strategy: MPS
          mpsConfig:
            defaultActiveThreadPercentage: 50      # each client ≤ 50% of SMs
            defaultPinnedDeviceMemoryLimit: 10Gi   # each client ≤ 10Gi pinned
```

**Validated result:** both containers shared one chip, and the driver spun up a **per-claim MPS
control daemon**:
```
C0: GPU 0: NVIDIA H100 80GB HBM3 (UUID: GPU-d4283ca6-...)
C1: GPU 0: NVIDIA H100 80GB HBM3 (UUID: GPU-d4283ca6-...)   ← same chip
$ oc get pods -A | grep mps
nvidia-gpu-operator   mps-control-daemon-defcbcac-...-3fm66gr   1/1  Running   ← daemon per claim
poc-dra-phase2        mps-two-containers                        2/2  Running
```

**Confirmed:** `MPSSupport` **is** settable through the operator-managed GPUCluster
(`draDriver.featureGates`, rendered as the driver's `FEATURE_GATES` env) — no hand-rolled Helm
install needed. The mutual exclusivity with DynamicMIG is real and enforced at the driver, so a node
is *either* a DynamicMIG node *or* an MPS node — a scheduling-pool design decision, not a per-pod one.

### A3. Consumable capacity — independent pods sharing one GPU ⛔ blocked on this cluster

The only mechanism by which **separate** pods/claims share one physical GPU. The driver publishes the
device with `allowMultipleAllocations: true`; each claim requests a slice of capacity:

```yaml
# ResourceClaimTemplate request (NVIDIA ConsumableShares, accounting mode "memory")
requests:
  - name: gpu
    exactly:
      deviceClassName: gpu.nvidia.com
      capacity:
        requests: { memory: 4Gi }     # or: shares: 2
# allocation records consumedCapacity + a shareID per co-tenant
```

**Blocked here, and why:** the NVIDIA `ConsumableShares` driver gate requires the Kubernetes
`DRAConsumableCapacity` feature gate, which is **alpha and off** in K8s 1.35 and **not** bundled into
OCP 4.22's `TechPreviewNoUpgrade` — and this cluster cannot switch to `CustomNoUpgrade` to add it
(§2). It is also **accounting-only** (the scheduler prevents over-subscription; it does *not* add
hardware isolation beyond what MIG gives). **Roadmap:** `DRAConsumableCapacity` → Beta in **K8s 1.36**;
revisit on an OCP 4.23+/CustomNoUpgrade cluster.

---

## B. Multi-tenant quota & isolation

### B1. ResourceQuota — cap a tenant's GPU consumption ✅ (GA)

The GA, production-ready lever for per-tenant fairness. A namespaced `ResourceQuota` keyed on the
DeviceClass caps how many devices (and how many claim objects) a tenant may hold.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata: { name: tenant-gpu-quota, namespace: poc-dra-phase2 }
spec:
  hard:
    gpu.nvidia.com.deviceclass.resource.k8s.io/devices: "2"   # ≤ 2 whole GPUs
    count/resourceclaims.resource.k8s.io: "3"                 # ≤ 3 claim objects
```

**Validated result:** a claim requesting `count: 2` was admitted (`used devices=2`); a claim requesting
`count: 3` was **rejected at admission**:

```
Error from server (Forbidden): resourceclaims.resource.k8s.io "q-toobig" is forbidden:
exceeded quota: tenant-gpu-quota, requested: gpu.nvidia.com.deviceclass.resource.k8s.io/devices=3,
used: ...=2, limited: ...=2
```

Notes: (1) the dotted key `<deviceclass-name>.deviceclass.resource.k8s.io/devices` works verbatim even
though the DeviceClass name itself contains dots; (2) the device count is charged **at claim
creation** from the request's `count` — before any pod binds — so a tenant cannot over-request even
speculatively; (3) `count/resourceclaims.resource.k8s.io` caps claim *objects* independently.

### B2. Admin access — privileged inspection, namespace-gated ✅ (Beta, on by default)

`adminAccess: true` lets an operator claim grant access to **in-use** devices (health/inspection)
without blocking the tenant workload. It is gated on a namespace **label** so tenants cannot
self-grant it.

```yaml
# namespace must carry: resource.kubernetes.io/admin-access: "true"
spec:
  devices:
    requests:
      - name: gpu
        exactly: { deviceClassName: gpu.nvidia.com, allocationMode: All, adminAccess: true }
```

**Validated result:** identical claims — **accepted** in a namespace labeled
`resource.kubernetes.io/admin-access: "true"`; **rejected** in an unlabeled one:

```
The ResourceClaim "admin-claim-bad" is invalid: spec.devices.requests[0].adminAccess: Forbidden:
admin access to devices requires the `resource.kubernetes.io/admin-access: true` label on the
containing namespace
```

RBAC separation follows naturally: admins/drivers own cluster-scoped `DeviceClass` / `ResourceSlice`;
tenants are confined to namespaced `ResourceClaim` / `ResourceClaimTemplate`; only an admin-labeled
namespace can mint privileged claims.

### B3. No preemption — the fairness fact that shapes multi-tenancy ✅

DRA devices are **not preemptible**. A low-priority pod holding GPUs is **not** evicted to make room
for a higher-priority pod — the latter simply waits.

Setup: a `dra-low` (prio 100) pod grabs **all 8** GPUs via `allocationMode: All`; then a `dra-high`
(prio 100000) pod requests 1 GPU.

**Validated result:**

```
want-high (prio 100000): phase=Pending      hog-low (prio 100): phase=Running   ← NOT evicted
FailedScheduling: 0/4 nodes are available: 4 cannot allocate all claims. still not schedulable,
  preemption: 0/4 nodes are available: 4 Preemption is not helpful for scheduling.
```

**GPUaaS implication:** priority alone does **not** guarantee a high-tier tenant can seize a GPU from
a low-tier one. Fairness/guarantees need an *admission/quota* layer (ResourceQuota per tenant, §B1,
or Kueue — §D) plus, for hard guarantees, **reserved capacity** (dedicated nodes or pre-provisioned
MIG for I3). Over-subscription relies on tenants releasing devices, not on eviction.

### B4. Device taints / tolerations — drain one GPU or slice ⛔ blocked on this cluster

Cordon or evict workloads off a *single* unhealthy device without touching the rest of the node —
the DRA analogue of node taints.

```yaml
# Admin taints one device (CEL selects by a unique attribute, e.g. uuid); NoExecute also EVICTS.
apiVersion: resource.k8s.io/v1
kind: DeviceTaintRule
metadata: { name: drain-gpu-3 }
spec:
  deviceSelector:
    driver: gpu.nvidia.com
    selectors:
      - cel: { expression: "device.attributes['gpu.nvidia.com'].uuid == 'GPU-...'" }
  taint: { key: gpuaas.io/maintenance, value: "1", effect: NoExecute }
# Claims opt back in with a matching tolerations: list on the device request.
```

**Blocked here, and why:** `DRADeviceTaints` (+ `DRADeviceTaintRules`) is **alpha and off** in K8s
1.35, not in OCP 4.22 `TechPreviewNoUpgrade`, and unreachable on this locked cluster (§2). **Roadmap:**
Beta in **K8s 1.36**, GA in **1.37**. Until then, device-level maintenance means draining the whole
node (and the DRA driver last).

---

## C. Multi-GPU allocation & constraints

### C1. `count: 2` — two whole GPUs in one claim ✅ (GA)

```yaml
requests:
  - name: gpus
    exactly: { deviceClassName: gpu.nvidia.com, count: 2 }
```

**Validated result:** the pod saw two distinct chips —
```
GPU 0: NVIDIA H100 80GB HBM3 (UUID: GPU-c82b4af3-...)
GPU 1: NVIDIA H100 80GB HBM3 (UUID: GPU-d8e1503f-...)
```
`allocationMode: All` (§B3) is the other multi-device mode — it grabbed all 8 chips (`gpu-0`…`gpu-7`).

### C2. `matchAttribute` — co-locate two MIG slices on the same chip ✅ (GA)

One claim, two requests of different MIG profiles, constrained to share a physical GPU:

```yaml
spec:
  devices:
    requests:
      - name: small
        exactly:
          deviceClassName: mig.nvidia.com
          selectors: [{ cel: { expression: "device.attributes['gpu.nvidia.com'].profile == '1g.10gb'" }}]
      - name: big
        exactly:
          deviceClassName: mig.nvidia.com
          selectors: [{ cel: { expression: "device.attributes['gpu.nvidia.com'].profile == '2g.20gb'" }}]
    constraints:
      - requests: ["small","big"]
        matchAttribute: "gpu.nvidia.com/parentUUID"
```

**Validated result:** `nvidia-smi -L` from the pod showed **one** parent GPU hosting **both** slices:
```
GPU 0: NVIDIA H100 80GB HBM3 (UUID: GPU-397c9eea-...)
  MIG 2g.20gb  Device 0: (UUID: MIG-56483ba4-...)
  MIG 1g.10gb  Device 1: (UUID: MIG-11722c19-...)
```
Allocation: `small → gpu-1-mig-1g10gb-19-0`, `big → gpu-1-mig-2g20gb-14-2` (same chip `gpu-1`). The
inverse, `distinctAttribute`, forces devices onto *different* parents (e.g. spread for fault
tolerance).

> **Pod-form gotcha (hit during validation):** a `ResourceClaim` with multiple requests is referenced
> by the pod **once** in `resourceClaims`; the *container* then names each request under
> `resources.claims: [{name: <podClaimName>, request: small}, {name: <podClaimName>, request: big}]`.
> Referencing the same claim twice in `pod.spec.resourceClaims` fails with
> `status.reservedFor[1]: Duplicate value`.

### C3. NVLink / clique topology constraints ⛔ N/A on a single 8×H100 node

There is **no** `nvlink` / `clique` / `board` attribute on `gpu.nvidia.com` devices. "Same NVLink
domain" is a property of **multi-node NVLink fabrics** (GB200 NVL72) handled by the ComputeDomain
driver, or of **NVSwitch/Fabric-Manager partitioning** (`partitionN` attribute, alpha
`FabricManagerPartitioning`, needs `FABRIC_MODE=1`). On this node the available topology levers are
`parentUUID` (MIG co-location, §C2) and the PCI/NUMA attributes `resource.kubernetes.io/pcieRoot`,
`pciBusID`, `numaNode` in CEL selectors. Recorded as a **negative result**, not a gap.

---

## 8. Partitionable-device counter model (DynamicMIG internals) ✅

This is *how* the scheduler guarantees a whole-GPU claim and a MIG slice never overlap on one chip.
With `DRAPartitionableDevices` + `DynamicMIG`, the `gpu.nvidia.com` pool on `ocp-h200-0` publishes
**9 ResourceSlices** (`pool.resourceSliceCount: 9`):

- **1 "header" slice** with **0 devices** whose `spec.sharedCounters` defines **8 counter sets**,
  `gpu-0-counter-set` … `gpu-7-counter-set` — one capacity pool per physical chip.
- **8 device slices**, one per chip, **26 logical devices each** (208 total).

Each per-chip counter set holds (verified live on `gpu-7-counter-set`):

```
memory 81152Mi · multiprocessors 132 · copy-engines 8 · decoders 7 · jpeg-engines 7 · ofa-engines 1
memory-slice-0 … memory-slice-7  (8 × "1")
```

Every logical device declares what it draws via `consumesCounters[].counterSet` + `counters`. The
**whole-GPU device `gpu-N` consumes the entire set** (all 8 memory-slices, 132 SMs, every engine) —
so allocating it exhausts the chip's counters and **no MIG slice on that chip can be allocated**, and
vice-versa. That mutual exclusion is pure counter arithmetic in the scheduler; the driver only carves
hardware once a claim is prepared.

**The 26 logical devices per chip** (Phase 1's "26/chip" figure — now **confirmed with composition**):

| | whole GPU | 1g.10gb | 1g.10gb+me | 1g.20gb | 2g.20gb | 3g.40gb | 4g.40gb | 7g.80gb | total |
|---|---|---|---|---|---|---|---|---|---|
| count | 1 | 7 | 7 | 4 | 3 | 2 | 1 | 1 | **26** |

These are *all possible* MIG placements advertised simultaneously; the counters make overlapping ones
mutually exclusive, so only a counter-consistent subset is ever allocated at once. (`+me` = the
media-engine variant of the 1g slice.)

---

## D. Kueue / extended-resource bridge — closing the Phase 1 Finding 7 gap

Phase 1 Finding 7 flagged that Kueue counts `nvidia.com/gpu` (device-plugin units) and cannot account
for DRA `ResourceClaim`s, so switching to DRA loses Kueue quota. Phase 2 separates what *can* be done
today from the clean future answer.

### D1. `DRAExtendedResource` — the clean bridge ⛔ blocked on this cluster

A DeviceClass can advertise a **classic extended-resource name**; then legacy pods keep using
`resources.limits` and existing quota / cluster-autoscaler / Kueue accounting keep working, while DRA
satisfies the request underneath. The *same* name can be served by the device plugin on some nodes and
DRA on others — the migration lever.

```yaml
apiVersion: resource.k8s.io/v1
kind: DeviceClass
metadata: { name: gpu.nvidia.com }
spec:
  selectors: [{ cel: { expression: "device.attributes['gpu.nvidia.com'].type == 'gpu'" }}]
  extendedResourceName: nvidia.com/gpu      # ← legacy pods: resources.limits: nvidia.com/gpu: 1
```

**Blocked here, and why:** `DRAExtendedResource` is **alpha and off** in K8s 1.35, not in OCP 4.22
`TechPreviewNoUpgrade`, unreachable on this locked cluster (§2). **Roadmap:** Beta in **K8s 1.36**.
This is the piece to validate first on a future 4.23+/`CustomNoUpgrade` cluster.

### D2. What works today — ResourceQuota (GA) ✅

The per-DeviceClass `ResourceQuota` from §B1 is the GA, production-ready substitute for the *quota*
half of the Kueue gap: it enforces per-tenant GPU caps at admission **without** Kueue and **without**
the extended-resource gate. It does not provide Kueue's *queueing/borrowing/fair-sharing* — only hard
caps. The implicit form `deviceclass.resource.kubernetes.io/<class>` needs no `extendedResourceName`.

### D3. True DRA-in-Kueue — still upstream-pending

Native Kueue management of `ResourceClaim` workloads (queueing, cohorts, borrowing over DRA devices)
is an upstream work-in-progress with no downstream OpenShift ETA. Until it ships, the choice stands:
**(a)** device-plugin + Kueue for full quota/queueing, or **(b)** DRA for attribute-based
allocation + `ResourceQuota` caps. RHOAI 3.5 is in camp (a): its `HardwareProfile` maps to
`nvidia.com/gpu` device-plugin resources, **not** DRA claims — so a DRA-native GPUaaS serving path
runs through hand-authored DRA manifests today, not through RHOAI.

---

## 9. Monitoring DRA allocations

- **DCGM exporter** is deployed by GPUCluster (`nvidia-dcgm-exporter-dra`). Classic device-plugin
  attribution assumed 1 GPU = 1 pod; DRA breaks that (shared/MIG allocations), so enable
  **ResourceSlice-based pod attribution**:
  `dcgmExporter.enablePodLabels: true` / `enablePodUID: true` — DCGM metrics then carry pod
  name/UID/namespace for shared and MIG allocations (the metering foundation, as in Phase 1 Finding 2
  but now DRA-aware).
- **Kubelet-plugin health**: gRPC probes report plugin availability only (GPU plugin port **51516**,
  ComputeDomain **51515**) — **not** allocation health. NVML device health is a separate alpha gate
  (`NVMLDeviceHealthCheck`, ⊥ DynamicMIG).
- **Inspection idioms** used throughout this doc:
  ```bash
  oc get resourceslices                                   # inventory + slice count
  oc get resourceclaim <c> -o jsonpath='{.status.allocation.devices.results}'   # what got allocated
  oc describe resourceclaim -n <ns> | grep -E 'Name:|Device:|State:'
  oc get pod <p> -o jsonpath='{.status.containerStatuses[*].ready}'
  ```

---

## 10. Negative / mutual-exclusivity matrix (first-class results)

| Boundary | Status | Evidence |
|---|---|---|
| `MPSSupport` ⊥ `DynamicMIG` | **enforced** | validated — had to flip DynamicMIG off to enable MPS (§A2); the driver renders one `FEATURE_GATES` set |
| `MPS` ⊥ time-slicing | by design | MPS needs `EXCLUSIVE_PROCESS`, time-slicing needs `DEFAULT` compute mode |
| time-slicing is a **no-op on MIG** | by design | time-slicing is whole-GPU temporal sharing only |
| `TimeSlicing` strategy requires `TimeSlicingSettings` on this build | **validated** | prepare fails without the gate even at default interval (§A1) |
| `DynamicMIG` ⊥ `PassthroughSupport` (VFIO), `NVMLDeviceHealthCheck` | by design | documented driver exclusivities; out of scope here |
| MPS rejected under `ConsumableShares` | by design | — |
| ComputeDomain / IMEX | **N/A** on single node | multi-node NVLink (GB200) only; all 8 H100 already share local NVLink |
| Fabric-Manager `partitionN` constraints | **N/A** | needs NVSwitch fabric + `FABRIC_MODE=1` |

A node is therefore **one mode at a time**: *DynamicMIG* **or** *MPS* **or** plain whole-GPU
(+time-slicing). This is a **node-pool** design decision for GPUaaS, not a per-pod toggle.

---

## 11. Corrections & additions to Phase 1 (`poc.md`)

1. **GPU Operator is v26.7.3** (Phase 1 says v26.7.2); DRA driver reports `595.91.7`, CUDA driver
   `13.2.0`.
2. **"26 devices per chip" is confirmed** — and decomposed: 1 whole-GPU + 25 MIG placements
   (1g.10gb×7, 1g.10gb+me×7, 1g.20gb×4, 2g.20gb×3, 3g.40gb×2, 4g.40gb×1, 7g.80gb×1). The counter
   definitions live in a **0-device "header" ResourceSlice** (`spec.sharedCounters`), 8 counter sets.
3. **JSON vs CEL attribute access:** in raw ResourceSlice JSON, attributes are **flat**
   (`.attributes.type.string`, `.attributes.profile.string`); CEL addresses them **domain-keyed**
   (`device.attributes['gpu.nvidia.com'].type`). Both describe the same data — don't expect a nested
   `gpu.nvidia.com` object in the JSON.
4. **DeviceClass selectors** `gpu.nvidia.com` and `mig.nvidia.com` differ only by
   `...type == 'gpu'` vs `'mig'`; **both are published by driver `gpu.nvidia.com`** (there is no
   separate `mig.nvidia.com` *driver*).
5. **Time-slicing needs the `TimeSlicingSettings` gate on v26.7.3** — corrects the public-docs claim
   that only custom intervals need it (§A1).
6. **No DRA preemption** — add to the design notes; priority does not evict DRA holders (§B3).
7. **Finding 7 (Kueue) is partially closable today** via GA `ResourceQuota` (§D2); the clean
   `DRAExtendedResource` bridge is alpha/blocked here (§D1).
8. **Drain ordering:** when draining the GPU node, drain the **DRA driver last** (it must un-prepare
   devices after claim-holding pods are gone).

---

## 12. GPUaaS mapping & graduation roadmap

| Isolation grade | Mechanism | DRA capability | Runnable here |
|---|---|---|---|
| **I0 Sandbox** | MIG `1g.10gb` | `mig.nvidia.com` + CEL `profile`; DynamicMIG on-demand | ✅ |
| **I0+ density** | time-slice / MPS share | `GpuConfig` sharing (same-claim) | ✅ (time-slice, MPS) |
| **I1 Serve** | whole GPU | `gpu.nvidia.com` + product/memory CEL | ✅ |
| **I2 Multi-GPU** | 2+ chips, co-located slices | `count`/`All` + `matchAttribute: parentUUID` | ✅ (NVLink-clique N/A on 1 node) |
| **I3 Reserved** | dedicated capacity | per-tenant node pool + `ResourceQuota`; no-preemption makes reservation necessary | ✅ (quota); node-pool is design |
| **tenant caps** | per-namespace quota | `ResourceQuota` per DeviceClass | ✅ |
| **fair-share/queueing** | Kueue over DRA | `DRAExtendedResource` / DRA-in-Kueue | ⛔ alpha/pending |
| **device drain** | taint one GPU | `DRADeviceTaints` | ⛔ alpha |
| **fractional share** | sub-slice divisible | `DRAConsumableCapacity` | ⛔ alpha |

**Graduation path (why a production GPUaaS should track 1.36/1.37):** the alpha gates this locked
4.22 cluster cannot run — `DRAExtendedResource`, `DRADeviceTaints`, `DRAConsumableCapacity`,
`DRADeviceBindingConditions` — all reach **Beta in K8s 1.36**, and taints/device-status reach **GA in
1.37**. On OpenShift that means an un-upgradeable `TechPreviewNoUpgrade` cluster (this one) is a
*validation* cluster; the production grade of these multi-tenant controls arrives with OCP 4.23+
(K8s 1.36) where the gates move to Beta/default-on and no `TechPreviewNoUpgrade` lock is required.

---

## 13. Cleanup & end state

**Test workloads removed** (validated): namespaces `poc-dra-phase2` and `poc-dra-admin` deleted with
all claims/pods/quota; the MPS control daemon disappeared once its claim was gone.

**End state of this run:** the cluster is left **in the DRA stack** (not reverted to ClusterPolicy),
restored to the DynamicMIG configuration so further research can continue immediately:

```
gpucluster/gpu-cluster     ready   (draDriver.featureGates: DynamicMIG=true, TimeSlicingSettings=true)
nvidiadriver/gpu-driver    ready   (DTK build, driver 595.91.7)
ResourceSlices: 8 chips × 26 devices (208) + compute-domain (2)   ← DynamicMIG model active
```

**To fully revert to ClusterPolicy / device-plugin** (`nvidia.com/gpu: 8`), reuse Phase 1's Full
Cleanup — delete `gpucluster` + `nvidiadriver`, recreate the ClusterPolicy
([`poc.md` → Full Cleanup](./poc.md)), and confirm `nvidia.com/gpu: 8`. Do this before tearing the
node down, or if the rest of the stack (RHOAI HardwareProfiles, Kueue) needs device-plugin resources.

> **Stopped ≠ free.** The H100 node bills whether in DRA or device-plugin mode. Reverting to
> ClusterPolicy does **not** save cost — only **deleting** the instance does.

---

## References

- [DRA concepts — Kubernetes](https://kubernetes.io/docs/concepts/resource-management/dynamic-resource-allocation/) ·
  [feature gates](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) ·
  [quota for DRA](https://kubernetes.io/docs/concepts/policy/resource-quotas/) ·
  [device taints](https://kubernetes.io/docs/concepts/resource-management/dynamic-resource-allocation/device-taints/)
- [DRA GA — K8s 1.34 blog](https://kubernetes.io/blog/2025/09/01/kubernetes-v1-34-dra-updates/) ·
  [consumable capacity — K8s 1.34 blog](https://kubernetes.io/blog/2025/09/18/kubernetes-v1-34-dra-consumable-capacity/)
- NVIDIA DRA driver: [gpu-allocation](https://dra-driver-nvidia-gpu.sigs.k8s.io/docs/concepts/gpu-allocation/) ·
  [time-slicing](https://dra-driver-nvidia-gpu.sigs.k8s.io/docs/guides/gpu-allocation/time-slicing/) ·
  [MPS](https://dra-driver-nvidia-gpu.sigs.k8s.io/docs/guides/mps/) ·
  [MIG](https://dra-driver-nvidia-gpu.sigs.k8s.io/docs/guides/gpu-allocation/mig/) ·
  [feature-gates](https://dra-driver-nvidia-gpu.sigs.k8s.io/docs/reference/feature-gates/) ·
  [resourceslice-attributes](https://dra-driver-nvidia-gpu.sigs.k8s.io/docs/reference/resourceslice-attributes/)
- NVIDIA on OpenShift: [dra-gpu-ocp](https://docs.nvidia.com/datacenter/cloud-native/openshift/latest/dra-gpu-ocp.html)
- Red Hat: [DRA GA in OCP 4.21](https://developers.redhat.com/articles/2026/03/25/dynamic-resource-allocation-goes-ga-red-hat-openshift-421-smarter-gpu) ·
  [Multitenant AI inference with DRA](https://developers.redhat.com/articles/2026/08/03/multitenant-ai-inference-dynamic-resource-allocation-openshift)
- Phase 1: [`poc.md`](./poc.md) · MIG: [`../mig/poc.md`](../mig/poc.md) · Provisioning: [`../ibmc/openshift-h200-provisioning.md`](../ibmc/openshift-h200-provisioning.md)
