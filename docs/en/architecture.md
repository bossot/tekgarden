# TekGarden Architecture

High-level system overview. This document describes the components,
their relationships, and the design principles of TekGarden.

## Architecture diagram

```
┌──────────────────────────────────────────────────────────────┐
│                      INTERNET (WAN)                          │
└──────────────────────┬───────────────────────────────────────┘
                       │
               ┌───────┴───────┐
               │   Router *    │  ← ISP, NAT firewall
               └───────┬───────┘
                       │
          ┌────────────┼───────────────────────┐
          │            │                       │
   ┌──────┴──────┐ ┌──┴──┐          ┌─────────┴──────────┐
   │   DNS 1 *   │ │ DMZ │          │   Reverse Proxy *  │
   │ (Pi-hole)   │ │VLAN │          │  (Traefik/HAProxy) │
   └─────────────┘ └─────┘          └────────────────────┘
                                              │
          ┌───────────────────────────────────┼───────────────────────┐
          │                VLAN 10 (management)                      │
   ┌──────┴──────┐  ┌──────────┴─────────┐  ┌───────────────────┐   │
   │  Proxmox    │  │   QNAP NAS *       │  │   PBS *           │   │
   │  (Altair)   │  │   (Maggie)         │  │   (Backups)       │   │
   │  Hypervisor  │  │   RAID10, 8 TB     │  │                   │   │
   └──────┬──────┘  └────────────────────┘  └───────────────────┘   │
          │                                                          │
          │  ┌───────────────── VLAN 20 (kubernetes) ───────────┐   │
          │  │                                                    │  │
          │  │    ┌──────────┐  ┌──────────┐  ┌──────────┐      │  │
          │  │    │  CP-01   │  │  CP-02   │  │  CP-03   │      │  │
          │  │    │  k3s     │  │  k3s     │  │  k3s     │      │  │
          │  │    │  control  │  │  control  │  │  control  │      │  │
          │  │    └────┬─────┘  └────┬─────┘  └────┬─────┘      │  │
          │  │         └──────┬──────┘               │           │  │
          │  │                │ etcd                 │           │  │
          │  │         ┌──────┴──────────────────────┴───┐       │  │
          │  │         │         k3s Cluster             │       │  │
          │  │         │   (FluxCD, Traefik, CoreDNS)    │       │  │
          │  │         └──────┬──────────────────────────┘       │  │
          │  │                │                                  │  │
          │  │    ┌───────────┼───────────┬──────────────┐      │  │
          │  │    │           │           │              │      │  │
          │  │  ┌─┴───┐  ┌───┴───┐  ┌───┴───┐  ┌───────┴───┐  │  │
          │  │  │WKR-01│  │WKR-02│  │WKR-03│  │  OpenClaw  │  │  │
          │  │  │worker│  │worker│  │worker│  │   agent    │  │  │
          │  │  └─────┘  └─────┘  └─────┘  └───────────┘  │  │
          │  └──────────────────────────────────────────────┘  │
          │                                                     │
          │  VLAN 30 (storage) ─────────────────────────────────┘
          │        │
          │        └── NFS/SMB mounts from QNAP
          │
          │  VLAN 40 (services) ──────┐
          │         ┌─────────────────┼──────────────┐
          │         │                 │              │
          │    ┌────┴────┐    ┌──────┴──────┐  ┌────┴────┐
          │    │ Authentik│   │  Immich     │  │Miniflux │
          │    │ (SSO)    │   │  (Photos)   │  │ (RSS)   │
          │    └─────────┘   └─────────────┘  └─────────┘
          │
          │  Additional servers:
          │  ┌──────────────────────────────────────────────┐
          │  │   VPS Hetzner * (Edgeway) / Pangolin         │
          │  │   External reverse proxy + Ansible every 15m │
          │  └──────────────────────────────────────────────┘
          │
          │  * Note: These services run outside the k3s cluster
```

## Core components

### Hypervisor: Proxmox VE
- **Name**: Altair
- **CPU**: Intel Ultra 7 265T
- **RAM**: 64 GB
- **Storage**: 2 TB NVMe
- **OS**: Debian with Proxmox VE (KVM + LXC)

Manages all virtual machines and LXC containers in TekGarden.
Provides snapshots, fast cloning, and live migration between nodes.

### Kubernetes: k3s cluster
- **Distribution**: k3s (Rancher/SUSE)
- **Nodes**: 6 (3 control plane + 3 workers)
- **Version**: 1.34.x
- **Storage**: Longhorn for distributed storage

Manages all TekGarden services through declarative manifests.
FluxCD automatically syncs the cluster state with the Git repository.

### NAS: QNAP TS-473A
- **Name**: Maggie
- **RAM**: 64 GB
- **Drives**: 4 × 4 TB RAID10
- **Connectivity**: 2 × 2.5 GbE

Centralized storage for persistent data (photos, videos, backups).
Shares volumes via NFS with the k3s cluster and via SMB with workstations.

### DNS: Pi-hole
- **Nodes**: 2 (high availability)
- **Function**: DNS sinkhole (ad and tracker blocking)
- **Network**: VLAN 40 (services)

Filters DNS queries at the network level. Blocks domains from advertising,
trackers, and known malware. Migration to AdGuard Home planned.

### Reverse Proxy: Traefik + Pangolin
- **Traefik**: Ingress controller for k3s cluster
- **Pangolin**: External reverse proxy at Hetzner VPS
  - TLS termination, rate limiting, WAF
  - Reverse proxy for non-K8s services

### GitOps: FluxCD
- Automatic sync from Git repository
- HelmReleases for external charts
- Kustomizations for local deployments
- ExternalSecrets with 1Password for secrets

### Infrastructure as Code
- **OpenTofu**: LXC provisioning on Proxmox
- **Ansible**: Node configuration (network, users, packages)
- **FluxCD**: Deployment and maintenance on the cluster

### Monitoring: Prometheus + Grafana
- **Prometheus**: Metrics collection (nodes, pods, services)
- **Grafana**: Dashboards and alerts
- **Loki**: Centralized logs
- **AlertManager**: Notifications via Telegram

## Design principles

1. **Declarative**: Everything that can be defined in code, is defined in code.
   Manual configuration is the devil — the machine must be reproducible.

2. **Reproducible**: The cluster can be rebuilt from scratch following the docs.
   If a node dies, it's replaced with OpenTofu + Ansible.

3. **Secure**: Network segmentation via VLANs, least privilege access,
   secrets in 1Password with per-application access.

4. **Documented**: Decisions recorded as ADRs, runbooks for recovery
   and daily operations.

5. **Minimalist**: Prefer lightweight services (Alpine, distroless). k3s instead
   of kubeadm. Prometheus with simple rules.

6. **Autonomous**: The system should work without human intervention.
   - FluxCD auto-corrects deviations
   - Failed pods restart automatically
   - Alerts only for situations requiring human action

7. **Observable**: Metrics, logs, and traces for all services.
   Grafana dashboards for immediate visibility.

8. **Cost-effective**: Limited resources → smart decisions.
   No over-provisioning. Containers, not heavy VMs.

## Traffic flow

### External web request
```
User → Internet → Router → Pangolin (Hetzner VPS) → Traefik (k3s)
                                                        ↓
                                                   Service (pod)
                                                        ↓
                                            Database / Cache (pod)
```

### Internal access (LAN)
```
User → Router → VLAN 20 (k3s) → Service (pod, ClusterIP/NodePort)
                                      ↓
                         VLAN 30 (NFS) → QNAP NAS
```

## Security

- **Perimeter**: Single open port (443) on the router. Traefik on the cluster
  handles TLS and internal routing.
- **Networks**: VLANs isolate management, kubernetes, storage, and service traffic.
  Only authorized traffic between VLANs.
- **Authentication**: Authentik (SSO) for all web-accessible services.
- **Secrets**: 1Password Connect as external secret service. Never in the repo.
- **Updates**: FluxCD auto-updates container images.
  Ansible for system updates on LXC nodes.

## Technical notes

- **k3s Traefik**: The default Traefik bundled with k3s is replaced by a
  custom instance configured via HelmRelease (FluxCD).
- **etcd**: Embedded k3s etcd cluster with 3 control planes for HA.
- **PBS Backups**: Proxmox Backup Server for incremental LXC snapshots.
- **Longhorn**: Distributed replicated storage for cluster PVs.
