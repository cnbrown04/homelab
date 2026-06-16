# Bootstrap ArgoCD on Sisyphus

Run these commands once on a new cluster after Talos is provisioned and kubeconfig is configured.

## Prerequisites

- `helm`, `kubectl`, `sops` available locally
- Age private key at `~/.config/sops/age/keys.txt`
- kubeconfig pointing at the sisyphus cluster

## Steps

```bash
# 1. Add Helm repos
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# 2. Create argocd namespace and store the age private key
kubectl create namespace argocd
kubectl create secret generic sops-age-key \
  --from-file=keys.txt=$HOME/.config/sops/age/keys.txt \
  -n argocd

# 3. Install ArgoCD
helm install argocd argo/argo-cd \
  --namespace argocd \
  --version 7.x \
  -f sisyphus/bootstrap/argocd-values.yaml

# 4. Bootstrap the root Application (ArgoCD manages everything else from here)
kubectl apply -f sisyphus/bootstrap/root-app.yaml
```

## Sync order (automatic via sync-waves)

| Wave | App | What it deploys |
|------|-----|-----------------|
| -2 | newt-infra | newt namespace + PVC |
| -1 | newt-secrets | newt-main-tunnel-auth secret (SOPS decrypted) |
| 0 | newt | Newt Helm chart (newt 1.5.0) |
| 0 | jellyfin | Jellyfin deployment + storage |
| 0 | csi-driver-nfs | NFS CSI driver Helm chart |

## Accessing ArgoCD UI

After Pangolin is configured to route `argocd.calebbrown.dev` → `argocd-server.argocd:80`:

```bash
# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

Then log in at https://argocd.calebbrown.dev with username `admin`.

## Pangolin config

Add a new site/route in Pangolin pointing to:
- Target: `http://<node-ip>:<nodePort>` where nodePort is from `kubectl get svc argocd-server -n argocd`
- Or if using ClusterIP, expose via `kubectl port-forward` temporarily to set up Pangolin

ArgoCD server runs in insecure (HTTP) mode since Pangolin terminates TLS.
