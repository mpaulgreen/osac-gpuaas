# osac-gpuaas — Claude Context

## What this project is

Knowledgebase for the **GPUaaS prototype** on IBM Cloud VPC: provisioning guides, GPU research notes,
and runbooks for a self-managed OpenShift cluster with H100 GPU workers. Uses UPI (IPI is broken on
IBM Cloud VPC). Currently validates the hardware isolation layer (MIG) and AI platform stack (RHOAI 3.5)
needed for the GPUaaS I0–I4 isolation grades.

## Repository layout

```
ibmc/
  openshift-h200-provisioning.md  # UPI provisioning guide (Parts A–F) — primary reference
  ibmcloud-h200-env.example       # Env template → copy to ~/.ibmcloud-h200-env
  install-certificates.md         # Let's Encrypt TLS via cert-manager + IBM CIS webhook
  README.md                       # ibmc section index

mig/
  poc.md                          # MIG partitioning research notes (H100, all exercises validated)
  rhoai-3.5-deploy.md             # RHOAI 3.5 deployment guide (prerequisite for mig/poc.md)

dra/
  poc.md                          # DRA research notes — all exercises validated (OCP 4.21 whole-GPU, OCP 4.22 DynamicMIG)
```

## Current cluster state (as of 2026-11-19)

- **Account**: Ecosystem AI workloads (IBM Cloud 3179756), `ca-tor` / `ca-tor-3`
- **Cluster**: OCP 4.22.22 (K8s 1.35), 3 masters (bx2-8x32) + 1 H100 worker (gx3d-160x1792x8h100)
- **GPU**: 8 × H100 80GB HBM3, `nvidia.com/gpu: 8` schedulable, GPU Operator v26.7.3 (ClusterPolicy mode)
- **Stack**: RHOAI 3.5.3, RHCL 1.4.3, cert-manager 1.20.1, COO 1.5.5, OTel, NFD
- **MIG PoC**: All exercises (3.1–3.6) validated and documented in `mig/poc.md`
- **DRA PoC**: All exercises validated and documented in `dra/poc.md` (DynamicMIG proven on OCP 4.22)
- **TLS**: Let's Encrypt certs active on API + router
- **FeatureSet**: `TechPreviewNoUpgrade` permanently enabled (unlocks `DRAPartitionableDevices`)

## Key facts

- **API key**: Account-scoped — create from within Ecosystem AI workloads account in browser
- **Env file**: `~/.ibmcloud-h200-env` (never committed). Always start from `ibmcloud-h200-env.example`
- **Login**: `source ~/.ibmcloud-h200-env && ibmcloud_login` in every new terminal
- **GPU profile**: `gx3d-160x1792x8h100` (H100; H200 available in ca-tor-3 but H100 chosen)
- **Base domain**: `ibmc.kni.syseng.devcluster.openshift.com` (CIS instance `ocp-cis`)

## Gotchas (provisioning)

- **Wrong account after SSO login**: `ibmcloud login --sso` defaults to Markets-Segments. Fix: create API key from within the target account in browser — the key is account-scoped.
- **Stale IBMCLOUD_REGION in shell**: `${IBMCLOUD_REGION:-ca-tor}` only falls back when unset. `unset IBMCLOUD_REGION` before login if `~/.zshrc` sources an old env file.
- **SSH keys are region-scoped**: Upload a new key per region (`ibmcloud is key-create`).
- **Presigned URLs expire in 4h**: Regenerate `bootstrap-shim.ign` before B8 if B3 was done earlier.
- **H100 hardware init takes 60-90 min**: Serial console is blank during this period — normal.
- **`COS_IGN_BUCKET`/`COS_IMAGE_BUCKET` missing**: Env file must be started from the example or sed silently fails in B3/B5.

## Gotchas (MIG)

- **ConfigMap propagation delay (30-60s)**: After updating a MIG ConfigMap, restart the MIG Manager pod before changing the node label — otherwise MIG Manager reads stale content and fails with `mig-config not present`.
- **State label trick**: Delete `nvidia.com/mig.config.state` before changing the config label to avoid stale `success` signals in wait loops.
- **Custom ConfigMap must include all profiles**: Every profile used as a node label must exist in the active ConfigMap — including `all-disabled` for toggle cycles.
- **ClusterPolicy switch first**: Always switch `migManager.config.name` in ClusterPolicy before changing the node label.
- **RHOAI HardwareProfile**: Use `infrastructure.opendatahub.io/v1` (not v1alpha1 — rejected). `spec.scheduling.type: Node` is required. `AcceleratorProfile` CRD removed in RHOAI 3.5.

## Gotchas (DRA)

- **TechPreviewNoUpgrade is permanent**: Once set, it cannot be changed back and blocks minor version upgrades (e.g. OCP 4.22 → 4.23). Apply only on a cluster intended for tech preview work.
- **ClusterPolicy and GPUCluster are mutually exclusive**: Cannot coexist. Delete ClusterPolicy before applying GPUCluster. The switch is effectively one-way (both CRs are irreversible once created).
- **NvidiaDriver uses OCP DTK — no NGC credentials needed**: The `NvidiaDriver` CR builds the kernel driver in-cluster from `quay.io/openshift-release-dev` (included in OCP pull secret). Do NOT use `nvcr.io/nvidia/driver-rhel9` — that requires NGC credentials.
- **DynamicMIG required for all MIG+DRA**: `mig.nvidia.com` ResourceSlices only appear when `featureGates.DynamicMIG: true` is set in GPUCluster. Without it, only whole-GPU `gpu.nvidia.com` devices appear — even if hardware has MIG instances configured.
- **CEL syntax in K8s 1.35 (OCP 4.22) is domain-keyed**: MIG profile selectors use `device.attributes['gpu.nvidia.com'].profile == '1g.10gb'` (NOT `device.attributes["profile"].string`). The old flat-key syntax fails with "no such key" at runtime.
- **`has()` guard required for profile**: Whole-GPU devices in the same ResourceSlice have no `.profile` attribute. Without `has(device.attributes['gpu.nvidia.com'].profile)`, CEL panics when evaluated against whole-GPU devices.
- **Restart GPU Operator controller after ClusterPolicy → GPUCluster switch**: If the operator errors "cannot find RESTMapping for GPUCluster", delete the gpu-operator pod to force a REST mapping reload.
- **Always extract GPUCluster/NvidiaDriver CR from CSV `alm-examples`**: Do not hand-write the spec — image digests will be wrong. Use `oc get csv ... -o jsonpath='{.metadata.annotations.alm-examples}' | jq 'map(select(.kind=="GPUCluster"))[0]'`.
- **MIG Manager `ERROR_IN_USE`**: If MIG Manager fails reconfiguring with `ERROR_IN_USE`, a previous DynamicMIG session left compute instances attached. Clear the node label to stop MIG Manager, then use GPUCluster + DynamicMIG to let the DRA driver manage geometry instead.
