# osac-gpuaas — Claude Context

## What this project is

Provisioning guides and runbooks for standing up **self-managed OpenShift on IBM Cloud VPC** with
**2× H200/H100 GPU worker nodes** (16 GPUs), to serve large coding models for the agentic workspace
prototype. Uses UPI (not IPI — IPI is broken on IBM Cloud VPC).

## Folder structure

```
ibmc/
  openshift-h200-provisioning.md   # Main step-by-step guide (Parts A–F)
  ibmcloud-h200-env.example        # Env template → copy to ~/.ibmcloud-h200-env
  install-certificates.md          # Post-provisioning: Let's Encrypt via cert-manager + CIS webhook
  README.md
```

## Key facts to remember

- **Target account:** Ecosystem AI workloads (IBM Cloud account 3179756)
- **Region/zone:** `ca-tor` / `ca-tor-3`
- **GPU profile in use:** `gx3d-160x1792x8h100` (H100 — H200 quota confirmed available but H100 chosen)
- **Base domain:** `ibmc.kni.syseng.devcluster.openshift.com` (managed by CIS instance `ocp-cis`)
- **COS instance:** `ocp-cos`
- **SSH key:** `my-ocp-key-ca-tor` (uploaded to ca-tor region)
- **API key:** Account-scoped — must be created from within the Ecosystem AI workloads account in
  the browser; this is what determines which account `ibmcloud_login` lands on (no `-c` flag needed)

## Env file conventions

- Real env file lives at `~/.ibmcloud-h200-env` — never committed (contains API key)
- `ibmcloud-h200-env.example` is the safe-to-commit template with placeholders
- The guide's `sed` commands persist IBM Cloud resource IDs into `~/.ibmcloud-h200-env` as the
  deployment progresses — the file must be started from the example (not created from scratch) or
  the `sed` targets won't exist
- Always `source ~/.ibmcloud-h200-env && ibmcloud_login` at the start of every new terminal

## Common gotchas already encountered

- **Wrong account after SSO login:** `ibmcloud login --sso` defaults to Markets-Segments account.
  Fix: create the API key from within the Ecosystem AI workloads account in the browser.
- **Wrong region despite `ca-tor` default:** A stale `IBMCLOUD_REGION=eu-de` in `~/.zshrc` overrides
  the `${IBMCLOUD_REGION:-ca-tor}` fallback. Fix: `unset IBMCLOUD_REGION` or update `~/.zshrc`.
- **SSH keys are region-scoped:** The old `r010-...` key was eu-de. Upload a new key in ca-tor.
- **Presigned URLs expire in 4h:** Regenerate `bootstrap-shim.ign` before B8 if more than 4 hours
  have passed since B3.
- **LB `update_pending` errors:** Always use the `lb_wait` helper between sequential mutations on
  the same load balancer.
- **`COS_IGN_BUCKET`/`COS_IMAGE_BUCKET` missing:** If env file was created from scratch (not from
  the example), sed in B3/B5 silently fails. Add the lines manually before running B3.
- **Outbound SG rule:** Use `icmp_tcp_udp` not `all` (deprecated).
