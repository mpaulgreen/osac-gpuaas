# osac-gpuaas

Knowledgebase for the **GPU-as-a-Service (GPUaaS) prototype** on IBM Cloud VPC. Documents the provisioning, GPU partitioning, and AI platform stack needed to validate the hardware isolation model for multi-tenant LLM serving on OpenShift.

---

## Repository layout

| Path | Purpose |
|---|---|
| `ibmc/openshift-h200-provisioning.md` | Full UPI provisioning guide — OCP 4.22 on IBM Cloud VPC with H100 GPU worker (Parts A–F) |
| `ibmc/ibmcloud-h200-env.example` | Environment template → copy to `~/.ibmcloud-h200-env` |
| `ibmc/install-certificates.md` | Let's Encrypt TLS via cert-manager + IBM CIS DNS-01 webhook |
| `mig/poc.md` | MIG partitioning research notes — isolation, telemetry, reconfiguration, RHOAI integration (all validated on H100) |
| `mig/rhoai-3.5-deploy.md` | RHOAI 3.5 deployment guide (prerequisite for MIG exercises 3.4+) |
| `dra/poc.md` | Dynamic Resource Allocation (DRA) research notes (WIP) |

---

## Current cluster

| | |
|---|---|
| **Account** | Ecosystem AI workloads (IBM Cloud 3179756) |
| **Region / Zone** | `ca-tor` / `ca-tor-3` |
| **OCP version** | 4.22.22 |
| **Control plane** | 3 × `bx2-8x32` masters |
| **GPU worker** | 1 × `gx3d-160x1792x8h100` — 8 × H100 80GB HBM3 |
| **GPU Operator** | v26.7.2, GPUCluster + NvidiaDriver (DRA mode) |
| **RHOAI** | 3.5.3, DSC `Ready` |
| **TLS** | Let's Encrypt via cert-manager + IBM CIS webhook |
| **FeatureSet** | `TechPreviewNoUpgrade` (DRAPartitionableDevices active) |

---

## Quick start

**Provision from scratch** → `ibmc/openshift-h200-provisioning.md`  
**Install RHOAI 3.5** → `mig/rhoai-3.5-deploy.md`  
**MIG research** → `mig/poc.md`

**Login to existing cluster:**
```bash
source ~/.ibmcloud-h200-env && ibmcloud_login
export KUBECONFIG=~/ocp-h200-install/auth/kubeconfig
oc get nodes
```

**Console:** `https://console-openshift-console.apps.ocp-h200.ibmc.kni.syseng.devcluster.openshift.com`

---

## GPUaaS connection

This work validates the hardware layer for the GPUaaS I0 isolation grade (MIG slices as the unit of sale). Key findings are captured in `mig/poc.md`:

- MIG provides hardware-enforced memory and compute isolation between tenants
- `nvidia.com/mig-1g.10gb` (~9.5 GB) is the smallest H100 slice — the Sandbox/Serve-Small SKU unit
- DCGM reports per-slice telemetry via `GPU_I_PROFILE` + `GPU_I_ID` labels — the billing telemetry foundation
- RHOAI 3.5 `HardwareProfile` exposes MIG slices as selectable accelerators in the dashboard
