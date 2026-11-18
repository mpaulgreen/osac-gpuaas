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
  poc.md                          # DRA (Dynamic Resource Allocation) research notes (WIP)
```

## Current cluster state (as of 2026-11-18)

- **Account**: Ecosystem AI workloads (IBM Cloud 3179756), `ca-tor` / `ca-tor-3`
- **Cluster**: OCP 4.21.42, 3 masters (bx2-8x32) + 1 H100 worker (gx3d-160x1792x8h100)
- **GPU**: 8 × H100 80GB HBM3, `nvidia.com/gpu: 8` schedulable, GPU Operator v25.10.1
- **Stack**: RHOAI 3.5.3, RHCL 1.4.3, cert-manager 1.20.1, COO 1.5.5, OTel, NFD
- **MIG PoC**: All exercises (3.1–3.6) validated and documented in `mig/poc.md`
- **TLS**: Let's Encrypt certs active on API + router

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
