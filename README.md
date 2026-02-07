Here is the updated documentation for **The Pantheon**, reflecting the consolidation of your hardware and the refined roles for your servers.

# Homelab: The Pantheon

Welcome to the documentation for my home infrastructure. This lab is a mix of production services, container orchestration, and virtualization.

## Network Topology

All external traffic enters through a central gateway and is routed via Reverse Proxy to internal services.

- **Domain:** `*.calebbrown.dev`
- **Gateway:** Nginx Proxy Manager (Atlas)
- **Internal Network:** 10.0.1.0/24

---

## Hosts

### Prometheus (The All-Father / Hypervisor)
*The bringer of fire; the foundation upon which the Pantheon is built.*
- **Role:** Virtualization Host (Type-1 Hypervisor)
- **OS:** Proxmox VE
- **Specs:** 16 Cores / 64GB RAM
- **Key Function:** Hosts the majority of the lab's infrastructure as Virtual Machines and Containers.

### Tartarus (Storage VM)
*The deep abyss; high-capacity data retention.*
- **Status:** 🟢 *Operational / Virtualized*
- **Parent Host:** Prometheus
- **Role:** NAS / Data Management
- **Stack:** TrueNAS
- **Goal:** Provide persistent volumes (NFS/iSCSI) for the Sisyphus cluster and centralized storage for all network users.

### Atlas (Core Services VM)
*The bearer of the heavens; provides the entry point for all traffic.*
- **Parent Host:** Prometheus (or Dedicated Hardware)
- **Role:** Edge Gateway & Core Services
- **OS:** Ubuntu Server 22.04
- **Key Services:**
  - **Nginx Proxy Manager:** SSL termination and routing.
  - **Portainer:** Container management for non-K8s workloads.
  - **Uptime Kuma:** Monitoring and heartbeats.

### Sisyphus (Kubernetes Cluster)
*Endless workloads, perpetually managed.*
- **Role:** Application Orchestration
- **Distribution:** Talos (Running as VMs on Prometheus)
- **Services:**
  - [No Services Yet]
- **Integration:** Ingress resources are routed via **Atlas** to the cluster load balancer.

---

## Management & Deployment

### Kubernetes Deployment
For **Sisyphus**, deployments are managed via:
- [x] Helm
- [ ] ArgoCD (Planned)
- [x] Manual `kubectl` apply

### Virtualization Management
- **Proxmox GUI:** Centralized management of Prometheus resources.
- **TrueNAS Dashboard:** Storage pool health and share permissions.

---

## Maintenance Notes
- **Updates:** Updates are performed Monthly.
- **Security:** 
  - SSH keys only: `github.com/cnbrown04.keys`
  - Root login disabled.
  - Hardware Passthrough: Ensure HBA/Disks are correctly passed through from Prometheus to Tartarus for ZFS integrity.
