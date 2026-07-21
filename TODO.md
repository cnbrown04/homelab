# TODO

- [ ] Expose the Kubernetes API server (port 6443) externally through Pangolin as a raw TCP resource, so `kubectl` works outside the home network without opening new firewall ports or standing up a separate VPN. Reuses the existing WireGuard tunnel in `sisyphus/workloads/wireguard/` (add an HAProxy `mode tcp` frontend on a tunnel IP → `kube.sisyphus.calebbrown.dev:6443`, plus the VPS-side raw-resource plumbing noted in `CONTEXT.md`).
