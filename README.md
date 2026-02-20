# Homelab: The Pantheon

Welcome to the documentation for my home infrastructure. This lab is a mix of production services, container orchestration, and virtualization across two distinct hardware nodes.

## Network Topology

All external traffic enters through a central gateway and is routed via Reverse Proxy to internal services.

- **Domain:** `*.calebbrown.dev` `*.cronarch.com`
- **Gateway:** Nginx Proxy Manager -> Wireguard (RackNerd VPS)
- **Internal Network:** 10.0.1.0/24

---

## Hosts

### Atlas (Hypervisor #1)
*Entrance node, bearer of the heavens*
- **Hardware:** ASRock DeskMini 110
- **Specs:** i5-7600 (4c/4t) / 16gb RAM
- **Role:** Edge Gateway & Kubernetes Host
- **OS:** Proxmox VE
- **Key Workloads:**
  - **Tailscale Exit Node:** Primarily serves for internal access to the hypervisor systems.
  - **Wireguard Tunnel:** Connected to the RackNerd VPS running Nginx Proxy Manager to route to specific things.

### Prometheus (Hypervisor #2)
*The bringer of fire; the foundation of the lab's storage and compute.*
- **Hardware:** Dell PowerEdge R740
- **Specs:** Dual Xeon Gold 4110 (8c/16t) / 64GB RAM
- **Role:** Virtualization & Storage Host
- **OS:** Proxmox VE
- **Key Workloads:**
  - **Tartarus (Storage VM):** The virtualized storage backbone.
  - **Docker Host (Ubuntu VM):** Runs portainer and any docker containers that I don't want to run in the kubernetes cluster.

## Virtual Machines

### Tartarus (Storage VM)
*The deep abyss; high-capacity data retention.*
- **Status:** 🟢 *Operational / Virtualized*
- **Parent Host:** Prometheus (R740)
- **Role:** NAS / Data Management
- **Stack:** TrueNAS
- **Goal:** Provide persistent volumes (NFS/iSCSI) for the **Sisyphus** cluster and centralized storage for the network.
- **Capacity:** Currently running 7 SFF 1.6TB SAS 10k HDD's in a RAIDZ2, around 8tb of usable storage. In the future it will be migrated to an LFF server as a second TrueNAS node.

---

## Management & Deployment

### Virtualization Management
- **Proxmox Cluster:** Centralized management of Atlas and Prometheus resources.
- **TrueNAS Dashboard:** Storage pool health and share permissions.

---

## Maintenance Notes
- **Updates:** Updates are performed Monthly.
- **Security:** 
  - SSH keys only: `github.com/cnbrown04.keys`
  - Root login disabled.
  - **Hardware Passthrough:** Ensure the R740's HBA/Disks are correctly passed through to the Tartarus VM to maintain ZFS pool integrity.
