# Cluster DNS naming (Scylla)

**Cluster name:** `scylla` (aligned with the Talos / kubeconfig context for this cluster).

**Convention:** Any hostname published for workloads on this cluster uses:

`<service>.scylla.calebbrown.dev`

where `<service>` is the app or role (e.g. `omni`, `omni-k8s`). The literal segment **`scylla`** is the cluster subdomain; do not omit it when creating DNS or reverse-proxy entries.

**Examples (Omni):**

| Role | Hostname |
|------|----------|
| Omni UI / API | `omni.scylla.calebbrown.dev` |
| Kubernetes API proxy | `omni-k8s.scylla.calebbrown.dev` |
| Machine API (SideroLink) | `omni-siderolink.scylla.calebbrown.dev` |

Update external DNS and Nginx Proxy Manager (or TLS certs) whenever you add a new service so they target these names and reach the Traefik LoadBalancer IP for this cluster.
