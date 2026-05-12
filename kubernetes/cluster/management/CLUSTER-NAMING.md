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
