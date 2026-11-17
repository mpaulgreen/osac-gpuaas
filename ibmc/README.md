# IBM Cloud — OpenShift with 2× H200 GPU nodes

Provision an OpenShift cluster on **IBM Cloud VPC** with **1 H100 GPU worker node**, so we can serve
a tool-capable coding model (Qwen2.5-Coder-32B / larger Granite) and run a real agentic task — the
top item on the prototype roadmap that the AWS A10G cluster is too small for.

This folder is written for someone who has **not** touched IBM Cloud in a while: every step is
explicit and explained.

- **[openshift-h200-provisioning.md](./openshift-h200-provisioning.md)** — the full step-by-step guide (Parts A–F).
- **[ibmcloud-h200-env.example](./ibmcloud-h200-env.example)** — environment template (copy to `~/.ibmcloud-h200-env`).
- **[install-certificates.md](./install-certificates.md)** — post-provisioning: install Let's Encrypt TLS certs via cert-manager + IBM CIS webhook.

---

## Read this before you start

**1. "Bare-metal H200" on IBM Cloud = a sole-tenant VSI, not the "Bare Metal Server" product.**
IBM Cloud's H200 offering is the profile **`gx3d-160x1792x8h200`**: a **single-tenant virtual server
on a full NVIDIA HGX host** — you get the *whole* 8-GPU server to yourself (bare-metal-class
performance). IBM's separate "Bare Metal Server for VPC" GPU product is **AMD MI300X only** — there is
no NVIDIA "metal" profile. So:

> **The smallest H100/H200 unit you can buy is one 8-GPU node = 8 GPUs**
> (160 vCPU / 1792 GiB / 8× 80 GB H100 or 141 GB H200).

**2. Why this path (self-managed OpenShift via UPI).**

| Option | Verdict here |
|---|---|
| **ROKS** (managed OpenShift) | ❌ No H100/H200 worker pools — managed GPU pools top out at L4 / L40S. |
| **IPI** (installer-provisioned) | ❌ Broken on IBM Cloud VPC — the CAPI provider boots RHCOS with no ignition (`metadata_service=false`, `user_data=null`). Wasted 7 attempts in the prior PoC. |
| **UPI** (user-provisioned) | ✅ **The path this guide uses.** You create the VPC + instances directly, so ignition works. Proven on this exact profile. |

**3. We deliberately keep it simple.** You asked for 2 *standalone* GPU nodes (not a multi-node
training fabric), so this guide **drops RoCE/RDMA** entirely — no cluster network, no SR-IOV, no
NVIDIA Network Operator. (Bonus: cross-node RoCE doesn't even work on these VPC instances — the prior
PoC proved traffic falls back to TCP.) That removes the hardest, most error-prone steps.

**4. ⚠️ Cost & capacity — plan before you provision.**
- An 8×H200 HGX host is among the **most expensive** VPC resources (order **~$30–40/hour each**, so
  **~$60–80/hour for two**, plus ~$1/hour for the 3 small masters). You pay for the whole node whether
  or not all 8 GPUs are busy. **Provision → validate → tear down.** (Part F.)
- **You almost certainly need a quota / capacity increase for H200** — request it via an IBM Cloud
  **support case / your account rep first** (it has lead time). `cannot_start_capacity` means no H200s
  are free in that zone right now — retry or try another zone. **Stopped instances still bill and still
  count against quota.**
- **Right-sizing note:** for *serving a 32B model* a single H100 GPU (80 GB) is sufficient.
  An L40S (`gx3-48x240x2l40s`) would also serve it at lower cost. 1× 8-GPU H100 node gives ample
  headroom for model serving plus experimentation.

**5. Time budget:** ~3–4 hours end to end (the UPI control plane is the long pole: ~90–120 min).

**6. End state:** 3 masters + 1 H100 node `Ready`, NVIDIA GPU Operator installed, **8
`nvidia.com/gpu` allocatable**, `nvidia-smi` passing. Then tear down (or continue to model serving —
see the guide's "Next steps").

---

## What you'll need (details in Part A)

- macOS/Linux workstation; `ibmcloud` CLI (+ `vpc-infrastructure`, `cis`, `cloud-object-storage`
  plugins), `openshift-install` + `oc` (matching, recent 4.x), `jq`, `helm`, `aws` CLI (for COS
  presign only), `ssh-keygen`.
- An IBM Cloud account with an **API key**, a **CIS** instance (DNS for your base domain), a **COS**
  instance, an uploaded **SSH public key**, and a Red Hat **pull secret**.
- The requested **H200 quota** in your target region/zone.

> Reuses the proven procedure from the internal `ibmc-ipi-roce` PoC (`manual/PHASE1/2/3B/4B/5`),
> minus RoCE and the model-serving stack.
