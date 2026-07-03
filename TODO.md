# TODO

- [ ] Expose the Kubernetes API server (port 6443) externally through Pangolin/Newt as a raw TCP resource, so `kubectl` works outside the home network without opening new firewall ports or standing up a separate VPN. Reuses the existing tunnel/auth infra in `sisyphus/workloads/newt/`.
