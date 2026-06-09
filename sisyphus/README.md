# sisyphus

Talos Linux cluster configuration for the `sisyphus` homelab cluster.

- **API endpoint:** `https://kube.sisyphus.calebbrown.dev:6443`
- **Kubernetes version:** v1.36.1
- **Talos version:** v1.13.3
- **Pod CIDR:** `10.244.0.0/16`
- **Service CIDR:** `10.96.0.0/12`

## Files

| File | Description |
|------|-------------|
| `secrets.yaml` | Talos-generated secrets bundle (cluster ID, bootstrap tokens, all CA private keys) |
| `talosconfig` | Client config for `talosctl` — contains your admin credentials |
| `controlplane.yaml` | Machine config applied to controlplane nodes |
| `worker.yaml` | Machine config applied to worker nodes |

All four files are encrypted with [SOPS](https://github.com/getsops/sops) using an age key.
`secrets.yaml` and `talosconfig` are fully encrypted. `controlplane.yaml` and `worker.yaml`
encrypt only sensitive fields (`key`, `secret`, `token`, `secretboxEncryptionSecret`);
public certificate data (`crt`) is left in plaintext.

## Prerequisites

```
brew install sops age
```

Your age private key must be present at `~/.config/sops/age/keys.txt`. The corresponding
public key is recorded in `../.sops.yaml`.

## Usage

### Decrypt a file to stdout

```bash
sops --decrypt sisyphus/secrets.yaml
sops --decrypt sisyphus/talosconfig
```

### Edit a file in-place (decrypts, opens $EDITOR, re-encrypts on save)

```bash
sops sisyphus/controlplane.yaml
```

### Apply config to a node

Decrypt on the fly and pipe directly to `talosctl`:

```bash
# Apply controlplane config
sops --decrypt sisyphus/controlplane.yaml | talosctl apply-config --insecure --nodes <IP> --file /dev/stdin

# Apply worker config
sops --decrypt sisyphus/worker.yaml | talosctl apply-config --insecure --nodes <IP> --file /dev/stdin
```

### Use talosctl with the decrypted talosconfig

```bash
sops --decrypt sisyphus/talosconfig > /tmp/talosconfig
TALOSCONFIG=/tmp/talosconfig talosctl --nodes <IP> get members
rm /tmp/talosconfig
```

Or merge it into your default talosconfig:

```bash
sops --decrypt sisyphus/talosconfig | talosctl config merge /dev/stdin
```

### Bootstrap the cluster (first time only)

```bash
sops --decrypt sisyphus/talosconfig > /tmp/talosconfig
TALOSCONFIG=/tmp/talosconfig talosctl bootstrap --nodes <controlplane-IP>
rm /tmp/talosconfig
```

## Regenerating secrets

If you need to rotate secrets, use `talosctl gen secrets` and `talosctl gen config` to
produce new files, then re-encrypt them:

```bash
sops --encrypt --in-place sisyphus/secrets.yaml
sops --encrypt --in-place sisyphus/talosconfig
sops --encrypt --in-place sisyphus/controlplane.yaml
sops --encrypt --in-place sisyphus/worker.yaml
```
