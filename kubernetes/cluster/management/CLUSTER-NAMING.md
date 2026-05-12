# Cluster DNS naming (calebbrown.dev root)

**Cluster name (kube / Talos):** `scylla` — this is only the context name in configs (e.g. `talosconfig`); it does **not** appear in public DNS.

**Convention:** Services exposed from this cluster use **one label** under the apex:

`<service>.calebbrown.dev`

That shape matches **Cloudflare Universal SSL** for `*.calebbrown.dev` when proxied, and keeps NPM / cert management simple.

**Examples (Omni):**

| Role | Hostname |
|------|----------|
| Omni UI / API | `omni.calebbrown.dev` |
| Kubernetes API proxy | `omni-k8s.calebbrown.dev` |
| Machine API (SideroLink) | `omni-siderolink.calebbrown.dev` |

Point DNS (or `*.calebbrown.dev` → NPM) and NPM proxy hosts at these names, then forward to the Traefik LoadBalancer IP for this cluster (`EXTERNAL-IP` on `Service/traefik` in `flux-system`).

## Pocket ID (OIDC for Omni)

| Role | Hostname |
|------|----------|
| Pocket ID (IdP UI + OIDC issuer) | `pocketid.calebbrown.dev` |

Ingress uses the Traefik **`web`** entrypoint (plain HTTP to the cluster) so NPM can forward **HTTP** to Traefik after terminating TLS.

**One-time after Pocket ID is running**

1. Open `https://pocketid.calebbrown.dev/setup` and finish admin / passkey setup.
2. In Pocket ID, create an **OIDC client** for Omni:
   - **Client ID:** `omni` (must match `omni-auth.sops.yaml`).
   - **Client secret:** decrypt the repo secret and use the same value:  
     `sops -d kubernetes/cluster/management/omni/omni-auth.sops.yaml` → copy `clientSecret` from `omni-auth.yaml`.  
     (If Pocket ID only issues random secrets, run `sops edit kubernetes/cluster/management/omni/omni-auth.sops.yaml` and set `clientSecret` to match what Pocket ID shows.)
   - **Redirect URI:** `https://omni.calebbrown.dev/oidc/consume`
3. NPM: proxy `pocketid.calebbrown.dev` and `omni.calebbrown.dev` (and the other Omni hosts) to Traefik as you already do for Omni.

Omni loads OIDC settings from a **second** config file (`omni-auth.sops.yaml` → `Secret/omni-auth`), merged with `omni-config`.

