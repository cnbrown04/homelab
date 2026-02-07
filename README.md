This adds the specific hardware profiles to your documentation. The **R740 (Prometheus)** is now correctly identified as the heavy-lifter for your storage and virtualization, while the **DeskMini (Atlas)** serves as the lean, dedicated head-end for your networking and Kubernetes orchestration.

# Homelab: The Pantheon

Welcome to the documentation for my home infrastructure. This lab is a mix of production services, container orchestration, and virtualization across two distinct hardware nodes.

## Network Topology

All external traffic enters through a central gateway and is routed via Reverse Proxy to internal services.

- **Domain:** `*.calebbrown.dev`
- **Gateway:** Nginx Proxy Manager (Atlas)
- **Internal Network:** 10.0.1.0/24

---

## Hosts

### Atlas (The Edge / Hypervisor #1)
*The bearer of the heavens; compact and resilient.*
- **Hardware:** ASRock DeskMini 110
- **Role:** Edge Gateway & Kubernetes Host
- **OS:** Proxmox VE
- **Key Workloads:**
  - **Sisyphus (Kubernetes Cluster):** The primary orchestration engine (Talos) runs here.
  - **Nginx Proxy Manager:** SSL termination and routing.
  - **Portainer:** Management for standalone Docker containers.

### Prometheus (The Powerhouse / Hypervisor #2)
*The bringer of fire; the foundation of the lab's storage and compute.*
- **Hardware:** Dell PowerEdge R740
- **Specs:** 16 Cores / 64GB RAM
- **Role:** Virtualization & Storage Host
- **OS:** Proxmox VE
- **Key Workloads:**
  - **Tartarus (Storage VM):** The virtualized storage backbone.
  - **General Compute:** Large-scale VMs and experimental services.

### Tartarus (Storage VM)
*The deep abyss; high-capacity data retention.*
- **Status:** 🟢 *Operational / Virtualized*
- **Parent Host:** Prometheus (R740)
- **Role:** NAS / Data Management
- **Stack:** TrueNAS
- **Goal:** Provide persistent volumes (NFS/iSCSI) for the **Sisyphus** cluster and centralized storage for the network.

### Sisyphus (Kubernetes Cluster)
*Endless workloads, perpetually managed.*
- **Status:** 🟢 *Operational*
- **Parent Host:** Atlas (DeskMini 110)
- **Distribution:** Talos
- **Integration:** Ingress resources are routed via the Nginx Proxy Manager (Atlas) to the cluster load balancer.

---

## Management & Deployment

### Kubernetes Deployment
For **Sisyphus**, deployments are managed via:
- [x] Helm
- [ ] ArgoCD (Planned)
- [x] Manual `kubectl` apply

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
