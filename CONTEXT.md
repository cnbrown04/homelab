# Repository context

Living map of this repo. Keep this current — see `CLAUDE.md` for the update rule.

## What this is

GitOps config for `sisyphus`, a Talos Linux Kubernetes cluster, managed by ArgoCD.
Everything under `sisyphus/apps/` is an ArgoCD `Application`; pushing to `main` is
the deploy mechanism. There is no CI — ArgoCD polls the repo and self-heals.

External access goes through **Pangolin**, reached via a **Newt** tunnel client
running in-cluster. There is no Kubernetes `Ingress` anywhere in this repo —
routing/TLS termination happens in Pangolin, not in-cluster.

## Directory map

```
sisyphus/
  controlplane.yaml, worker.yaml   Talos machine configs (SOPS-encrypted, sensitive fields only)
  secrets.yaml, talosconfig        Talos secrets bundle + admin client config (fully SOPS-encrypted)
  wireguard-patch.yaml             Talos machine-config patch that loads the wireguard kernel module
  README.md                        Talos/SOPS usage: decrypt, edit, apply-config, bootstrap, rotate secrets

  bootstrap/
    README.md                     One-time steps to stand up ArgoCD on a fresh cluster
    argocd-values.yaml             Helm values for the ArgoCD install itself
    root-app.yaml                  The single "root" Application ArgoCD is bootstrapped with;
                                    it points at sisyphus/apps and app-of-apps's everything else

  apps/                            One file per ArgoCD Application (the "app of apps" layer)
    kustomization.yaml              Lists every Application file below — must add new ones here
    infrastructure/
      csi-driver-nfs.yaml           NFS CSI driver Helm chart (kube-system)
    workloads/
      storage.yaml                  -> sisyphus/workloads/storage (nfs-config StorageClass)
      media.yaml                    -> sisyphus/workloads/media (namespaces + jellyfin/sonarr/radarr/
                                       prowlarr/flaresolverr/qbittorrent/calibre/audiobookshelf) +
                                       qbittorrent-secrets (SOPS) + seerr (own dir) + seerr Helm chart
      newt.yaml                     -> sisyphus/workloads/newt + newt/secrets (SOPS) + the `newt`
                                       Helm chart from charts.fossorial.io. THE PANGOLIN BLUEPRINT
                                       LIVES HERE inline as `blueprintData` — every public
                                       hostname/route/healthcheck is defined in this one file.
      tools.yaml                    -> sisyphus/workloads/tools (termix, iperf)
      website.yaml                  -> sisyphus/workloads/website (cronarch.com marketing site)

  workloads/                        Raw manifests, one dir per app, each with its own kustomization.yaml
    storage/                        Cluster-wide `nfs-config` StorageClass (server 10.0.1.111,
                                     share /mnt/styx/data/config, subDir per namespace)
    media/                          Shared namespaces.yaml + kustomization.yaml that pulls in all
                                     the *arr-stack + media-server workload dirs (see table below)
    jellyfin/, sonarr/, radarr/, prowlarr/, flaresolverr/, qbittorrent/, qbittorrent-secrets/,
    seerr/, calibre/, audiobookshelf/    Individual app manifests, referenced by media/kustomization.yaml
    newt/                           Namespace, PVC, kustomization for the Newt tunnel client
                                     (Helm values/blueprint itself live in apps/workloads/newt.yaml)
    tools/                          Termix deployment (namespace: tools, includes guacd sidecar)
                                     plus `iperf` — an sshd+iperf3 pod for network testing,
                                     ClusterIP-only, reached by SSH from Termix
    website/                        Cronarch marketing site manifests
```

## Workloads (namespace, domain, cluster address, storage)

| App | Namespace | Public domain (Pangolin) | ClusterIP:Port | Persistent storage |
|-----|-----------|---------------------------|-----------------|---------------------|
| Jellyfin | jellyfin | jellyfin.calebbrown.dev | jellyfin.jellyfin.svc.cluster.local:8096 | config: nfs-config; media: NFS PV `/mnt/styx/data/media` |
| Calibre | calibre | books.calebbrown.dev | calibre.calibre.svc.cluster.local:8080 | config: nfs-config; books: NFS PV `/mnt/styx/data/media/books` |
| Audiobookshelf | audiobookshelf | audiobooks.calebbrown.dev | audiobookshelf.audiobookshelf.svc.cluster.local:80 | config+metadata: nfs-config; audiobooks: NFS PV `/mnt/styx/data/media/audiobooks` |
| Sonarr | sonarr | sonarr.calebbrown.dev | sonarr.sonarr.svc.cluster.local:8989 | config: nfs-config; media: NFS PV `/mnt/styx/data/media` |
| Radarr | radarr | radarr.calebbrown.dev | radarr.radarr.svc.cluster.local:7878 | config: nfs-config; media: NFS PV `/mnt/styx/data/media` |
| Prowlarr | prowlarr | prowlarr.calebbrown.dev | prowlarr.prowlarr.svc.cluster.local:9696 | config: nfs-config |
| Flaresolverr | flaresolverr | — (internal only, used by prowlarr) | flaresolverr.flaresolverr.svc.cluster.local | none |
| qBittorrent | qbittorrent | torrent.calebbrown.dev | qbittorrent.qbittorrent.svc.cluster.local:8080 | config: nfs-config; downloads: NFS PV `/mnt/styx/data/media/downloads`; gluetun VPN sidecar, secrets from qbittorrent-secrets |
| Seerr | seerr | seerr.calebbrown.dev | seerr-seerr-chart.seerr.svc.cluster.local:80 | config: nfs-config (via Helm chart values); media: NFS PV `/mnt/styx/data/media` |
| Termix | tools | termix.calebbrown.dev | termix.tools.svc.cluster.local:8080 | nfs-config PVC |
| iperf | tools | — (internal only, SSH in via Termix) | iperf.tools.svc.cluster.local:2222 | none — `/config` is an emptyDir on purpose (sshd won't start if it can't chown its host keys, which nfs-config disallows) |
| ArgoCD | argocd | argocd.calebbrown.dev | argocd-server.argocd.svc.cluster.local:80 | n/a |
| Cronarch website | website | cronarch.com, www.cronarch.com | website.website.svc.cluster.local:3000 | none |
| Newt | newt | — (tunnel client itself) | n/a | own config PVC |

NFS server for everything above: `10.0.1.111`. Shares used: `/mnt/styx/data/config`
(subdivided per-namespace by the `nfs-config` StorageClass) and `/mnt/styx/data/media`
(and subdirectories `downloads`, `books`, `audiobooks`, mounted as dedicated PVs).


## Pangolin / Newt (public routing)

There is no in-cluster Ingress. Public hostnames are declared as a Newt "blueprint"
— a block of YAML embedded as `blueprintData` inside the Helm `valuesObject` in
`sisyphus/apps/workloads/newt.yaml`. Each entry under `public-resources` maps a
`full-domain` to a `targets[].hostname:port` (a ClusterIP service DNS name) plus an
HTTP healthcheck. **To expose a new service publicly, add an entry there** — this
is the actual source of truth, not a separate Pangolin UI config (there isn't one
tracked in-repo; the UI is only used for one-off things not yet migrated to the
blueprint, if any).

Resources can also be raw TCP (`mode: tcp` + `proxy-port`, no `full-domain`, and
no `method` on targets) — see the `iperf` entry. **A blueprint entry alone is not
enough**: the resource registers in Pangolin and shows up in the UI, but nothing
ever listens unless the VPS also has

1. a traefik entrypoint named `tcp-<proxy-port>` (address `:<port>/tcp`) in
   `traefik_config.yml`, and
2. a matching port published on the **gerbil** container in `docker-compose.yml`,
   plus the VPS firewall opened — both needing a stack restart.

Existing entrypoints are `tcp-25`, `tcp-587`, `tcp-993` (plus `web`/`websecure`);
gerbil publishes 25, 80, 443, 587, 993, 21820/udp, 51820/udp. A raw resource whose
`proxy-port` reuses one of those works with no VPS change — which is why `iperf`
borrows 993. **Those two VPS edits are the missing piece for the k8s API TODO.**
Note raw resources are served by traefik TCP routers, so proxy protocol settings
in `dynamic_config.yml` apply; enabling it breaks backends that don't speak it.

VPS tunnel topology (useful and non-obvious): WireGuard does not exist on the VPS
host — there is no `wg` binary and no `wg0`. Pangolin runs as Docker containers
(`pangolin`, `gerbil`, `traefik`), and `wg0` lives inside **gerbil's** network
namespace, addressed `100.89.128.1/24` (MTU 1280); traefik shares that namespace
via `network_mode: service:gerbil`. gerbil routes only `100.89.128.0/24` over the
tunnel — it has **no route to the pod/service CIDRs** — so Newt is reachable but
is a proxy for declared resources, not a subnet router. To get a shell on the
tunnel: `docker run --rm --network container:gerbil nicolaka/netshoot ...`.

**Two Newt settings gate tunnel throughput. Both defaulted badly.**

1. **Native (kernel) WireGuard** — `global.nativeMode.enabled` plus
   `newtInstances[].useNativeInterface`. **`newtInstances[].nativeMode.enabled` is
   not a real key**; it was set here for months, Helm silently ignored it, and Newt
   ran on its default securityContext (uid 65534, `capabilities.drop: [ALL]`),
   falling back to wireguard-go's *userspace netstack* — every byte of every hosted
   service through a Go TCP/IP stack. Native mode requires a privileged container
   (root, `NET_ADMIN`, `SYS_MODULE`), which is why the `newt` namespace carries
   `pod-security.kubernetes.io/enforce: privileged`; the cluster otherwise enforces
   baseline, which forbids added capabilities. The tell in the logs is
   `auth-daemon must be run as root` — if you see it, native mode is NOT engaged.
2. **CPU limit** — the chart defaults to `200m` (`global.resources`). Newt proxies
   all traffic in userspace, so that quota throttles the whole tunnel. CFS
   throttling is easy to misread as a network fault: throughput stalls to zero for
   whole seconds with *no* packet loss and a healthy congestion window.
   `newtInstances[].resources` is now set explicitly.

Measured on this link (Auburn ↔ RackNerd LA, 53ms RTT, symmetric gigabit at home):
**445–513 Mbit/s direct, pod→VPS, bypassing the tunnel entirely** — versus 16 up /
30 down through the tunnel on the bad config. The direct number is the benchmark to
compare against; don't trust a public speedtest server to establish the VPS ceiling
(serverius reported 185 Mbit/s and badly understated it). To re-measure the raw
path: `iperf3 -s -1` on the VPS, `iperf3 -c <vps-ip> -t 15` from a pod.

**Load-testing the tunnel takes services down.** Parallel unthrottled streams drive
packet loss until WireGuard's own keepalives fail and the tunnel drops — Pangolin's
UI stays up (it's local to the VPS) while everything proxied 502s. Cap with
`-b`/`-P 1` and prefer testing the direct path, which can't affect the tunnel.

The Newt Helm chart itself comes from `https://charts.fossorial.io` (chart `newt`);
the client image tag is pinned separately via `global.image.tag` in the same file.
Check both the chart's index (`charts.fossorial.io/index.yaml`) and the client's
GitHub releases (`fosrl/newt`) for the current versions before bumping either —
they version independently.

## Secrets (SOPS)

Encryption rules live in `.sops.yaml`, keyed to a single age recipient. Convention:
files named `*.enc.yaml` in a directory *without* a `kustomization.yaml` are decrypted
via a separate ArgoCD Application using `plugin: name: sops-decrypt` pointed at that
directory (see `qbittorrent-secrets`, `newt/secrets`, `tools/secrets`). Talos configs
(`controlplane.yaml`, `worker.yaml`) only encrypt sensitive fields (`key`, `secret`,
`token`, `secretboxEncryptionSecret`); `secrets.yaml` and `talosconfig` are encrypted
in full. See `sisyphus/README.md` for the decrypt/edit/apply workflow.

`.gitleaks.toml` allowlists the Talos placeholder key blob and `.claude/` paths.

## Adding a new workload (checklist)

1. `sisyphus/workloads/<app>/` — manifests + `kustomization.yaml`. Reuse the
   `nfs-config` StorageClass for config, and a hand-written PV (driver
   `nfs.csi.k8s.io`, server `10.0.1.111`) for any media/bulk storage, following the
   existing jellyfin/sonarr/calibre pattern.
2. Either add it to an existing app-of-apps bundle (e.g. `media/kustomization.yaml`
   + `media/namespaces.yaml` if it's media-related) or create a new
   `sisyphus/apps/workloads/<app>.yaml` Application and list it in
   `sisyphus/apps/kustomization.yaml`.
3. If it needs to be reachable from the internet, add an entry to the Newt
   `blueprintData` block in `sisyphus/apps/workloads/newt.yaml`.
4. Update the workloads table in `README.md` (and in this file).
5. Push to `main` — ArgoCD syncs automatically (`automated: prune, selfHeal`).

## Cluster facts

- API endpoint: `https://kube.sisyphus.calebbrown.dev:6443`
- Kubernetes v1.36.1, Talos v1.13.3
- Pod CIDR `10.244.0.0/16`, Service CIDR `10.96.0.0/12`
- Outstanding TODO (see `TODO.md`): expose the Talos/k8s API (6443) externally
  through Pangolin/Newt as a raw TCP resource, reusing the existing tunnel.
