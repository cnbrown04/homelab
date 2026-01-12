# Homelab: The Pantheon

Welcome to the documentation for my home infrastructure. This lab is a mix of production services, container orchestration, and high-performance computing experiments.

## Network Topology

All external traffic enters through a central gateway and is routed via Reverse Proxy to internal services.

- **Domain:** `*.calebbrown.dev`
- **Gateway:** Nginx Proxy Manager (Atlas)
- **Internal Network:** 10.0.1.0/24

---

## Hosts

### Atlas (Docker Host)
*The bearer of the heavens; provides the entry point for all traffic.*
- **Role:** Edge Gateway & Core Services
- **OS:** Ubuntu Server 22.04
- **Key Services:**
  - **Nginx Proxy Manager:** Handles SSL termination and routing.
  - **Portainer:** For container management.
  - **Uptime Kuma:** Monitoring and heartbeats.

### Sisyphus (Kubernetes Cluster)
*Endless workloads, perpetually managed.*
- **Role:** Application Orchestration
- **Distribution:** Talos
- **Services:**
  - [No Services Yet]
- **Integration:** Ingress resources are routed via **Atlas** to the cluster load balancer.

---

## Future Expansion 

### Tartarus (Storage Server)
*The deep abyss; high-capacity data retention.*
- **Status:** 🛠️ *Under Construction*
- **Role:** NAS / Cold Storage
- **Planned Stack:** TrueNAS
- **Goal:** Provide persistent volumes (NFS/iSCSI) for the Sisyphus cluster. As well as acting as the NAS for other servers.

### Prometheus (Compute Server)
*The bringer of fire; high-performance processing.*
- **Status:** *Planned*
- **Role:** HPC / AI Training / Video Transcoding
- **Planned Specs:** High-core count CPU, [e.g., NVIDIA RTX GPU]
- **Goal:** Handle intensive workloads that exceed the capacity of Sisyphus.

---

## Management & Deployment

### Kubernetes Deployment
For **Sisyphus**, deployments are managed via:
- [x] Helm
- [ ] ArgoCD (Planned)
- [x] Manual `kubectl` apply

---

## Maintenance Notes
- **Updates:** Updates are performed Monthly.
- **Security:** SSH keys only github.com/cnbrown04.keys; root login disabled;
