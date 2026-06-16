# homelab

Talos Linux Kubernetes cluster (sisyphus) managed via ArgoCD GitOps.

## Cluster access

ArgoCD UI: https://argocd.calebbrown.dev
- Pangolin → `argocd-server.argocd.svc.cluster.local:80` (HTTP — TLS terminated by Pangolin)
- Initial admin password: `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`

Jellyfin: https://jellyfin.calebbrown.dev
- Pangolin → `jellyfin.jellyfin.svc.cluster.local:8096`

## Adding a new workload

1. Create manifests under `sisyphus/workloads/<app>/` with a `kustomization.yaml`
2. Create an ArgoCD Application at `sisyphus/apps/workloads/<app>.yaml`
3. Add it to `sisyphus/apps/kustomization.yaml` under `resources:`
4. Push — ArgoCD syncs automatically

For encrypted secrets, name the file `*.enc.yaml` and place it in a directory without a `kustomization.yaml`. Create a separate Application for that directory with `plugin: name: sops-decrypt`.

## Fresh cluster bootstrap

See `sisyphus/bootstrap/README.md` for full steps. Summary:

```bash
kubectl create namespace argocd
kubectl create secret generic sops-age-key \
  --from-file=keys.txt=$HOME/.config/sops/age/keys.txt -n argocd
helm install argocd argo/argo-cd --version 9.5.21 \
  -f sisyphus/bootstrap/argocd-values.yaml -n argocd
kubectl apply -f sisyphus/bootstrap/root-app.yaml
```

After that ArgoCD manages everything. Then configure Pangolin routes for each service.

## Workloads

| App | URL | ClusterIP:Port |
|-----|-----|----------------|
| Jellyfin | jellyfin.calebbrown.dev | jellyfin.jellyfin.svc.cluster.local:8096 |
| ArgoCD | argocd.calebbrown.dev | argocd-server.argocd.svc.cluster.local:80 |
| Cronarch website | cronarch.com | website.website.svc.cluster.local:3000 |
| Newt | — | tunnel only |
