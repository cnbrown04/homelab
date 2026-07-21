# Migrate from Newt to Kernel WireGuard

This guide tells you how to change the cluster from Newt to a Pangolin Basic
WireGuard site. The Basic WireGuard site uses kernel WireGuard. Kernel WireGuard
gives more throughput than Newt.

## About this change

Newt sends all data through a userspace tunnel. This limits the throughput to
about 32 Mbit/s.

A Basic WireGuard site uses the kernel. The kernel gives about 215 to 385 Mbit/s
on the same link.

Pangolin sends public names to targets. For a WireGuard site, each target must be
an IP address in the site subnet. Pangolin does not accept host names for a
WireGuard site.

An HAProxy gateway in the cluster changes each subnet IP address to a cluster
service. Kernel WireGuard does the encryption. HAProxy does a simple TCP
connection. The gateway does not use iptables.

## Before you start

You must have these items:

- Access to the Pangolin VPS with Docker.
- Access to the Git repository for the cluster.
- The SOPS age key. You use the key to encrypt the secret.

## Part 1 — Make the site subnet larger

A new WireGuard site gets a /30 subnet by default. A /30 subnet gives only a few
addresses. Change the block size to /26. A /26 subnet gives about 61 addresses
that you can use.

**Caution: This procedure restarts the Pangolin containers. The tunnels stop for
a short time.**

1. Log in to the VPS.
2. Go to the Pangolin stack directory.
3. Make a backup of the configuration file:

   ```bash
   cp config/config.yml config/config.yml.bak
   ```

4. Open the file `config/config.yml`.
5. Add this line to the `gerbil:` section:

   ```yaml
   gerbil:
       start_port: 51820
       base_endpoint: "pangolin.example.com"
       site_block_size: 26        # add this line
   ```

6. Save the file.
7. Restart the stack:

   ```bash
   docker compose down && docker compose up -d
   ```

## Part 2 — Make the WireGuard site

1. Open the Pangolin dashboard.
2. Delete the old WireGuard site if a WireGuard site is present.
3. Make a new Basic WireGuard site.
4. Make sure that the new site gets a /26 subnet.
5. If the site gets a /30 subnet, reset the exit node. Then make the site again.
   The exit node keeps the old block size until you reset it.
6. Download the `wg0.conf` file.
7. Write down the `Address` value. This value is the site subnet, for example
   `100.89.128.64/26`. The pod uses the first address (`.64`). The services use
   the addresses from `.65`.

## Part 3 — Put the WireGuard client in the cluster

The repository has the WireGuard workload in `sisyphus/workloads/wireguard/`. The
pod has two containers:

- A WireGuard container. It runs the `wg0.conf` file and makes the kernel tunnel.
- An HAProxy container. It sends each subnet IP address to a cluster service.

Do these steps:

1. Open the secret file `sisyphus/workloads/wireguard-secrets/wireguard.enc.yaml`.
2. Put the new `wg0.conf` content into the `wg0.conf` field. Keep 4 spaces of
   indent on each line.
3. Encrypt the secret:

   ```bash
   sops --encrypt --in-place sisyphus/workloads/wireguard-secrets/wireguard.enc.yaml
   ```

   **Caution: The secret directory must be a sibling of the workload directory.
   Do not put the secret in a subdirectory of the workload. The sops-decrypt
   plugin makes the workload render as empty if a secret is below it.**

4. Open `sisyphus/workloads/wireguard/wireguard.yaml`.
5. Set the `SITE_CIDR` value to the site subnet from Part 2, for example
   `100.89.128.64/26`. The `SITE_CIDR` value must be the same as the `Address`
   value in the `wg0.conf` file.

The WireGuard container adds a `local` route for the `SITE_CIDR` value. The route
lets the subnet addresses arrive at the HAProxy container. You do not add
iptables rules.

## Part 4 — Add a service to the gateway

You add each service in two places: the HAProxy configuration and the Pangolin
dashboard.

### Add the service to HAProxy

1. Open `sisyphus/workloads/wireguard/haproxy.cfg`.
2. Copy one `frontend` block and one `backend` block.
3. Give the service the next free IP address in the subnet, for example
   `100.89.128.66`.
4. Set the service DNS name and the port. Use the cluster DNS name of the
   service. See this example:

   ```
   frontend sonarr
       bind 100.89.128.66:8989 transparent
       default_backend sonarr

   backend sonarr
       server sonarr sonarr.sonarr.svc.cluster.local:8989 resolvers coredns init-addr none check
   ```

### Add the service to Pangolin

1. Open the Pangolin dashboard.
2. Make a new resource on the WireGuard site.
3. Set the full domain, for example `sonarr.example.com`.
4. Set the target to the same IP address and port, for example
   `http://100.89.128.66:8989`. Use `http://`, not `https://`.

## Part 5 — Push the changes

1. Commit the changes.
2. Push to the `main` branch. ArgoCD applies the changes.
3. Make sure that the pod is ready:

   ```bash
   kubectl -n wireguard get pods
   ```

## Part 6 — Test the throughput (optional)

You measure the throughput through the tunnel. Pangolin routes an IP address only
after you make a resource for it. Make a resource first.

1. Add the iperf3 tool to the pod:

   ```bash
   kubectl -n wireguard exec deploy/wireguard -c wireguard -- apk add --no-cache iperf3
   ```

2. Start the iperf3 server in the pod:

   ```bash
   kubectl -n wireguard exec deploy/wireguard -c wireguard -- sh -c 'iperf3 -s -D'
   ```

3. On the VPS, run the iperf3 client in the gerbil namespace. Use a service IP
   address that has a Pangolin resource, for example `100.89.128.65`:

   ```bash
   docker run --rm --network container:gerbil networkstatic/iperf3 -c 100.89.128.65 -t 15
   ```

4. Add `-R` to the command to test the other direction (the pod sends the data).
   This direction is the direction for media.
5. Stop the iperf3 server in the pod when you are done:

   ```bash
   kubectl -n wireguard exec deploy/wireguard -c wireguard -- pkill iperf3
   ```

The iperf3 tool is not permanent. The tool goes away when the pod restarts.

## Part 7 — Remove Newt

Do this part only after all services work through the WireGuard gateway.

1. Delete the file `sisyphus/apps/workloads/newt.yaml`.
2. Delete the directory `sisyphus/workloads/newt/`.
3. Remove the line `- workloads/newt.yaml` from
   `sisyphus/apps/kustomization.yaml`.
4. Commit and push. ArgoCD removes the Newt workload.

## Result

| Path | Throughput |
|---|---|
| Newt (userspace) | about 32 Mbit/s |
| Kernel WireGuard, VPS to pod | about 385 Mbit/s |
| Kernel WireGuard, pod to VPS (media) | about 215 Mbit/s |
| Direct pod to VPS (no tunnel) | 445 to 513 Mbit/s |
