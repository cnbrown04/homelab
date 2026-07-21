# Repository context

Living map of this repo. Keep this current — see `CLAUDE.md` for the update rule.

## What this is

GitOps config for `sisyphus`, a Talos Linux Kubernetes cluster, managed by ArgoCD.
Everything under `sisyphus/apps/` is an ArgoCD `Application`; pushing to `main` is
the deploy mechanism. There is no CI — ArgoCD polls the repo and self-heals.

External access goes through **Pangolin**, reached via an in-cluster **Basic
WireGuard site**: a kernel-WireGuard client (`wireguard` namespace) dials the
Pangolin VPS, and an HAProxy sidecar L4-proxies each service's tunnel IP to its
cluster service. There is no Kubernetes `Ingress` anywhere in this repo —
routing/TLS termination happens in Pangolin, not in-cluster. (This replaced the
old Newt tunnel client in July 2026 for ~12× the throughput; see
`docs/migrate-newt-to-kernel-wireguard.md`.)

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
      tools.yaml                    -> sisyphus/workloads/tools (termix)
      website.yaml                  -> sisyphus/workloads/website (cronarch.com marketing site)
      wireguard.yaml                -> sisyphus/workloads/wireguard + wireguard-secrets (SOPS,
                                       sops-decrypt plugin). Standalone WireGuard client
                                       (linuxserver/wireguard) dialing out with a Pangolin-issued
                                       wg0.conf that comes from the secret. Secret is a SIBLING
                                       dir, not nested — see the sops-decrypt gotcha below.

  workloads/                        Raw manifests, one dir per app, each with its own kustomization.yaml
    storage/                        Cluster-wide `nfs-config` StorageClass (server 10.0.1.111,
                                     share /mnt/styx/data/config, subDir per namespace)
    media/                          Shared namespaces.yaml + kustomization.yaml that pulls in all
                                     the *arr-stack + media-server workload dirs (see table below)
    jellyfin/, sonarr/, radarr/, prowlarr/, flaresolverr/, qbittorrent/, qbittorrent-secrets/,
    seerr/, calibre/, audiobookshelf/    Individual app manifests, referenced by media/kustomization.yaml
    tools/                          Termix deployment (namespace: tools, includes guacd sidecar)
    website/                        Cronarch marketing site manifests
    wireguard/                      Pangolin Basic WireGuard *site* gateway (namespace: wireguard,
                                     privileged PSA). Two containers: linuxserver/wireguard runs a
                                     Pangolin-issued wg0.conf (kernel WG, dials out, no inbound) and
                                     in postStart adds a `local` route for the site subnet ($SITE_CIDR);
                                     an HAProxy sidecar (haproxy.cfg via configMapGenerator) binds each
                                     service's WG-subnet IP with `transparent` and TCP-proxies it to the
                                     service's cluster DNS name (resolved live via CoreDNS 10.96.0.10).
                                     This is the high-throughput replacement for Newt: kernel WG does the
                                     crypto, HAProxy does an L4 splice, NO iptables. Add a service = one
                                     frontend/backend block in haproxy.cfg (next free WG IP) + a Pangolin
                                     Resource targeting that IP:port. Site subnet is 100.89.128.64/26 (VPS
                                     gerbil.site_block_size: 26); pod is .64, services bind .65+. SITE_CIDR
                                     in wireguard.yaml must match the wg0.conf Address.
    wireguard-secrets/              SOPS-encrypted wg0.conf (client key + Pangolin peer) for the
                                     WireGuard client. Deliberately a SIBLING of wireguard/, not
                                     nested inside it — see the sops-decrypt gotcha below.
```

## Workloads (namespace, domain, cluster address, storage)

> **Note:** the "Public domain" column below is the Pangolin mapping. Each domain
> is a Pangolin Resource whose target is a tunnel IP served by the HAProxy gateway
> (see the Pangolin / WireGuard section). The `haproxy.cfg` IP↔service map is the
> in-repo source of truth; the Resources/domains themselves live in the Pangolin UI.

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
| ArgoCD | argocd | argocd.calebbrown.dev | argocd-server.argocd.svc.cluster.local:80 | n/a |
| Cronarch website | website | cronarch.com, www.cronarch.com | website.website.svc.cluster.local:3000 | none |
| WireGuard gateway | wireguard | — (client; dials out to Pangolin VPS) | n/a — no Service; binds tunnel IPs `100.89.128.65+` | none — `/config` is an emptyDir; wg0.conf comes from the `wireguard-config` SOPS secret |

NFS server for everything above: `10.0.1.111`. Shares used: `/mnt/styx/data/config`
(subdivided per-namespace by the `nfs-config` StorageClass) and `/mnt/styx/data/media`
(and subdirectories `downloads`, `books`, `audiobooks`, mounted as dedicated PVs).


## Pangolin / WireGuard (public routing)

There is no in-cluster Ingress. Public routing is a **Pangolin Basic WireGuard
site** terminated in-cluster by the `wireguard` workload, with an HAProxy sidecar
acting as the service map. The pieces:

- **Kernel-WireGuard client** (`linuxserver/wireguard`, privileged) runs a
  Pangolin-issued `wg0.conf` from the `wireguard-config` SOPS secret and dials out
  to gerbil. Talos ships the wireguard module in-kernel, so this is real kernel WG
  (full throughput), not wireguard-go userspace. It has no inbound port.
- **Site subnet** is `100.89.128.64/26` (64 addresses; the VPS sets
  `gerbil.site_block_size: 26`). The pod itself is `.64`; services bind `.65+`.
  `SITE_CIDR` in `wireguard.yaml` must equal the `Address` in `wg0.conf`.
- **`ip route replace local $SITE_CIDR dev wg0`** (wg container postStart) makes
  every address in the subnet locally deliverable, so packets for a service's
  tunnel IP land on the HAProxy socket. **That one route is the whole gateway
  plumbing — no iptables, no per-IP `ip addr`.**
- **HAProxy sidecar** (`mode tcp`, config in `haproxy.cfg` via configMapGenerator)
  `bind <tunnel-IP>:<port> transparent` (IP_TRANSPARENT, needs NET_ADMIN) for each
  service and TCP-proxies to the service's `*.svc.cluster.local` name, re-resolved
  live via CoreDNS (`10.96.0.10`). Because it shares the pod netns, it sees `wg0`
  and the local route.

**To expose a new service:** add a `frontend`/`backend` pair in `haproxy.cfg` on
the next free tunnel IP, then create a **Pangolin Resource** (UI) on the WireGuard
site whose target is `http://<tunnel-IP>:<port>`. Pangolin routes a tunnel IP only
after a Resource exists for it — cryptokey routing adds a `/32` per registered
target (`getAllowedIps()`), so an HAProxy block with no matching Resource never
receives traffic. `haproxy.cfg` is the in-repo IP↔service source of truth; the
Resources/domains live in the Pangolin UI (Basic WG has no in-repo blueprint).

**Basic WG targets must be IPs, not hostnames** — enforced in Pangolin's
`createTarget.ts` (`isIpInCidr(targetData.ip, site.subnet)`). That's *why* HAProxy
exists: Pangolin holds the stable tunnel IP, HAProxy maps it to the DNS name. Only
Newt could target `*.svc.cluster.local` directly (it ran inside the cluster); the
WG site can't, hence the proxy.

**Health checks are Newt-only and unavailable on this site.** Pangolin's target
health checks are executed by the Newt data-plane, so a Basic WG site never appears
in the health-check UI — confirmed by the maintainer on fosrl/pangolin#2105 (*"Health
checks have always only worked for newt sites... The newt itself is what is making
the health check request"*) and fosrl/pangolin#3105. Failover instead comes from
HAProxy: each backend has `check`, and in `mode tcp` an all-servers-down backend
refuses the socket, so a dead service stops accepting connections. Don't mix a WG
target with a Newt target on one Resource — the WG target (no health data) drags the
Resource into "degraded" (#3105).

**Why we left Newt (July 2026):** Newt proxies every byte through wireguard-go's
userspace netstack with sequential packet processing, capping the site tunnel at
~32 Mbit/s per TCP connection against a path that does 445–513. Even `--native-main`
(kernel WG on the main tunnel, newt 1.14.0) plus a raised CPU limit didn't clear it
to our satisfaction, so we moved to a Basic WG site + HAProxy: **~215 Mbit/s pod→VPS
(media direction), ~385 VPS→pod** — ~12× better. Full procedure and measurements in
`docs/migrate-newt-to-kernel-wireguard.md`.

### VPS tunnel topology (still current, non-obvious)

WireGuard does not exist on the VPS host — no `wg` binary, no `wg0`. Pangolin runs
as Docker containers (`pangolin`, `gerbil`, `traefik`); `wg0` lives inside
**gerbil's** network namespace at `100.89.128.1` (MTU 1280), and traefik shares that
namespace via `network_mode: service:gerbil`. gerbil routes only the site subnet
over the tunnel — it has **no route to the pod/service CIDRs** — so it is a proxy
for declared resources, not a subnet router. Shell on the tunnel:
`docker run --rm --network container:gerbil nicolaka/netshoot ...` (or
`networkstatic/iperf3` for throughput).

**Raw TCP resources** (`mode: tcp` + `proxy-port`, no `full-domain`) need VPS-side
plumbing beyond the Pangolin Resource: a traefik entrypoint `tcp-<proxy-port>`
(`:<port>/tcp`) in `traefik_config.yml` **and** a published port on the gerbil
container in `docker-compose.yml` (+ firewall), both needing a stack restart.
Existing entrypoints: `tcp-25`, `tcp-587`, `tcp-993` (plus `web`/`websecure`);
gerbil publishes 25, 80, 443, 587, 993, 21820/udp, 51820/udp. A `proxy-port` reusing
one of those works with no VPS change (e.g. borrowing 993). **These two edits are the
missing piece for the k8s API TODO.** Raw resources are served by traefik TCP
routers, so proxy-protocol settings in `dynamic_config.yml` apply — enabling it
breaks backends that don't speak it.

**MTU stays 1280.** Gerbil's default is client-wide; raising it invites
fragmentation and failed handshakes. Clamp MSS instead.

`sisyphus/wireguard-patch.yaml` (Talos `machine.kernel.modules: [wireguard]`) has
**never been applied** — `machine.kernel.modules` is still commented out in both
`controlplane.yaml` and `worker.yaml`, and nothing references the patch. Talos ships
WireGuard in-kernel for KubeSpan, so kernel WG works without it; if the tunnel ever
falls back to userspace `wireguard-go`, this patch is the first thing to try.

**Benchmarks.** On this link (Auburn ↔ RackNerd LA, 53ms RTT, symmetric gigabit at
home): **445–513 Mbit/s direct pod→VPS bypassing the tunnel** (the ceiling to compare
against — don't trust a public speedtest server; serverius reported 185 and badly
understated it); **~215 up / ~385 down through the kernel WG tunnel**. Re-measure raw:
`iperf3 -s -1` on the VPS, `iperf3 -c <vps-ip> -t 15` from a pod. **Load-testing the
tunnel takes services down** — parallel unthrottled streams drive loss until
WireGuard keepalives fail and the tunnel drops (Pangolin's UI stays up, everything
proxied 502s). Cap with `-b`/`-P 1` and prefer the direct path.

- Basic WG targets must be IPs (maintainer): https://github.com/fosrl/pangolin/issues/524
- Health checks are Newt-only (maintainer): https://github.com/fosrl/pangolin/issues/2105 · https://github.com/fosrl/pangolin/issues/3105
- Site types and their limits: https://docs.pangolin.net/manage/sites/understanding-sites
- Health checks & failover: https://docs.pangolin.net/manage/healthchecks-failover

## Secrets (SOPS)

Encryption rules live in `.sops.yaml`, keyed to a single age recipient. Convention:
files named `*.enc.yaml` in a directory *without* a `kustomization.yaml` are decrypted
via a separate ArgoCD Application source using `plugin: name: sops-decrypt` pointed at
that directory (see `qbittorrent-secrets`, `wireguard-secrets`). Talos configs
(`controlplane.yaml`, `worker.yaml`) only encrypt sensitive fields (`key`, `secret`,
`token`, `secretboxEncryptionSecret`); `secrets.yaml` and `talosconfig` are encrypted
in full. See `sisyphus/README.md` for the decrypt/edit/apply workflow.

**sops-decrypt CMP gotcha — keep secret dirs OUT of workload source trees.** The
plugin (defined in `sisyphus/bootstrap/argocd-values.yaml`) has a mismatched config:
its `discover` glob is recursive (`**/*.enc.yaml`) but its `generate` command is not
(`for f in *.enc.yaml` — top level only). So ArgoCD detects *any* source directory
that has an `.enc.yaml` **anywhere beneath it** as this Plugin instead of Kustomize,
then generates nothing (no top-level `.enc.yaml` in the parent) and silently drops
every plain manifest in that source. The `status.sourceTypes` of the app will read
`Plugin` where you expected `Kustomize`, with no error condition. Therefore a
workload's SOPS secret must live in a *sibling* directory (`wireguard-secrets`,
`qbittorrent-secrets`), never in a `secrets/` subdir of the workload. (The old
`newt/secrets` violated this and rendered its Namespace/PVC source empty; it's gone
now that Newt is decommissioned, but the trap remains for any new workload.) The
proper root-cause fix is to make the CMP `discover` glob non-recursive (`*.enc.yaml`) in the
argocd bootstrap values, but that config is applied by the `helm install` in
`bootstrap/`, not via GitOps, so editing the file alone changes nothing live — it
needs the repo-server's `argocd-cmp-cm` reloaded. Until then, the sibling-dir
convention is the workaround.

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
3. If it needs to be reachable from the internet, add a `frontend`/`backend` pair
   in `sisyphus/workloads/wireguard/haproxy.cfg` on the next free tunnel IP, then
   create a matching Pangolin Resource targeting `http://<tunnel-IP>:<port>` (see
   the Pangolin / WireGuard section).
4. Update the workloads table in `README.md` (and in this file).
5. Push to `main` — ArgoCD syncs automatically (`automated: prune, selfHeal`).

## Cluster facts

- API endpoint: `https://kube.sisyphus.calebbrown.dev:6443`
- Kubernetes v1.36.1, Talos v1.13.3
- Pod CIDR `10.244.0.0/16`, Service CIDR `10.96.0.0/12`
- Outstanding TODO (see `TODO.md`): expose the Talos/k8s API (6443) externally
  through Pangolin as a raw TCP resource, reusing the existing WireGuard tunnel.
