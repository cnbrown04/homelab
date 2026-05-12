# Cluster DNS naming (calebbrown.dev root)

**Cluster name (kube / Talos):** `scylla` — this is only the context name in configs (e.g. `talosconfig`); it does **not** appear in public DNS.

**Convention:** Services exposed from this cluster use **one label** under the apex:

`<service>.calebbrown.dev`

That shape matches **Cloudflare Universal SSL** for `*.calebbrown.dev` when proxied, and keeps NPM / cert management simple.

**Example:**

| Role | Hostname |
|------|----------|
| Pocket ID (IdP UI + OIDC issuer) | `auth.calebbrown.dev` |

Point DNS (or `*.calebbrown.dev` → NPM) and NPM proxy hosts at these names, then forward to the Traefik LoadBalancer IP for this cluster (`EXTERNAL-IP` on `Service/traefik` in `flux-system`).

## Pocket ID (`auth.calebbrown.dev`)

| Role | Hostname |
|------|----------|
| Pocket ID (IdP UI + OIDC issuer) | **`auth.calebbrown.dev`** |

Ingress uses the Traefik **`websecure`** entrypoint (HTTPS to the cluster on **:443**) so NPM can use one pattern for every app: **SSL → `https://<traefik-vip>:443`**, with multiple domain names on the same proxy host if you like. Use Cloudflare/NPM **Full** (not strict) unless Traefik serves a publicly trusted cert for those hostnames.

**One-time after Pocket ID is running:**

1. Open **`https://auth.calebbrown.dev/setup`** and finish admin / passkey setup.
2. NPM: add **`auth.calebbrown.dev`** and forward **`https://<traefik-vip>:443`**; Traefik routes by `Host`.

**Note:** `auth.calebbrown.dev` is a **single** label under the apex, so the same **`*.calebbrown.dev`** / Universal SSL patterns you use elsewhere often apply—confirm in Cloudflare / NPM that this hostname is covered by your cert.
