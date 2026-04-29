# Arquitectura del TekGarden

Visió general del sistema a alt nivell. Aquest document descriu els components,
les relacions entre ells i els principis de disseny del TekGarden.

## Diagrama d'arquitectura

```
┌─────────────────────────────────────────────────────────────┐
│                    INTERNET (WAN)                           │
└──────────────────────┬──────────────────────────────────────┘
                       │
               ┌───────┴───────┐
               │   Router *    │  ← ISP, firewall NAT
               └───────┬───────┘
                       │
          ┌────────────┼───────────────────────┐
          │            │                       │
   ┌──────┴──────┐ ┌──┴──┐          ┌─────────┴──────────┐
   │   DNS 1 *   │ │ DMZ │          │   Reverse Proxy *  │
   │ (Pi-hole)   │ │VLAN │          │   (Traefik/HAProxy)│
   └─────────────┘ └─────┘          └────────────────────┘
                                              │
          ┌───────────────────────────────────┼───────────────────────┐
          │                 VLAN 10 (management)                      │
   ┌──────┴──────┐  ┌──────────┴─────────┐  ┌───────────────────┐   │
   │  Proxmox    │  │   QNAP NAS *       │  │   PBS *           │   │
   │  (Altair)   │  │   (Maggie)         │  │   (Backups)       │   │
   │  Hypervisor │  │   RAID10, 8 TB     │  │                   │   │
   └──────┬──────┘  └────────────────────┘  └───────────────────┘   │
          │                                                          │
          │  ┌────────────────── VLAN 20 (kubernetes) ──────────┐   │
          │  │                                                    │  │
          │  │    ┌──────────┐  ┌──────────┐  ┌──────────┐      │  │
          │  │    │  CP-01   │  │  CP-02   │  │  CP-03   │      │  │
          │  │    │  k3s     │  │  k3s     │  │  k3s     │      │  │
          │  │    │  control  │  │  control  │  │  control  │      │  │
          │  │    └────┬─────┘  └────┬─────┘  └────┬─────┘      │  │
          │  │         └──────┬──────┘               │           │  │
          │  │                │ etcd                 │           │  │
          │  │         ┌──────┴──────────────────────┴───┐       │  │
          │  │         │         Cluster k3s             │       │  │
          │  │         │    (FluxCD, Traefik, CoreDNS)   │       │  │
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
          │    │ (SSO)    │   │  (Fotos)    │  │ (RSS)   │
          │    └─────────┘   └─────────────┘  └─────────┘
          │
          │  Servidors addicionals:
          │  ┌─────────────────────────────────────────────┐
          │  │   VPS Hetzner * (Edgeway) / Pangolin        │
          │  │   Reverse proxy extern + Ansible cada 15min │
          │  └─────────────────────────────────────────────┘
          │
          │  * Nota: Aquests serveis corren fora del cluster k3s
```

## Components principals

### Hypervisor: Proxmox VE
- **Nom**: Altair
- **CPU**: Intel Ultra 7 265T
- **RAM**: 64 GB
- **Emmagatzematge**: 2 TB NVMe
- **Sistema**: Debian amb Proxmox VE (KVM + LXC)

Gestiona totes les màquines virtuals i contenidors LXC del TekGarden.
Proporciona snapshots, clonat ràpid i migració en viu entre nodes.

### Kubernetes: Cluster k3s
- **Distribució**: k3s (Rancher/SUSE)
- **Nodes**: 6 (3 control plane + 3 workers)
- **Versió**: 1.34.x
- **Storage**: Longhorn per emmagatzematge distribuït

Gestiona tots els serveis del TekGarden via manifests declaratius.
FluxCD sincronitza automàticament l'estat del cluster amb el repositori Git.

### NAS: QNAP TS-473A
- **Nom**: Maggie
- **RAM**: 64 GB
- **Discos**: 4 × 4 TB en RAID10
- **Conectivitat**: 2 × 2.5 GbE

Emmagatzematge centralitzat per dades persistents (fotos, vídeos, backups).
Comparteix volums via NFS amb el cluster k3s i via SMB amb estacions de treball.

### DNS: Pi-hole
- **Nodes**: 2 (alta disponibilitat)
- **Funció**: DNS sinkhole (bloqueig de publicitat i trackers)
- **Xarxa**: VLAN 40 (serveis)

Filtra consultes DNS a nivell de xarxa. Bloqueja dominis de publicitat,
trackers i malware coneguts. Migració prevista a AdGuard Home.

### Reverse Proxy: Traefik + Pangolin
- **Traefik**: Ingress controller del cluster k3s
- **Pangolin**: Reverse proxy extern al VPS Hetzner
  - TLS termination, rate limiting, WAF
  - Proxy invers per serveis no-K8s

### GitOps: FluxCD
- Sincronització automàtica des del repositori Git
- HelmReleases per charts externs
- Kustomizations per desplegaments locals
- ExternalSecrets amb 1Password per secrets

### Infraestructura com a Codi
- **OpenTofu**: Provisionament de LXC a Proxmox
- **Ansible**: Configuració dels nodes (xarxa, usuaris, paquets)
- **FluxCD**: Desplegament i manteniment al cluster

### Monitoratge: Prometheus + Grafana
- **Prometheus**: Recollida de mètriques (nodes, pods, serveis)
- **Grafana**: Dashboards i alertes
- **Loki**: Logs centralitzats
- **AlertManager**: Notificacions per Telegram

## Principis de disseny

1. **Declaratiu**: Tot el que es pugui definir en codi, es defineix en codi.
   Configuració manual és el diable — la màquina ha de ser reproduïble.

2. **Reproduïble**: El cluster es pot recrear des de zero seguint la documentació.
   Si un node mor, es substitueix amb OpenTofu + Ansible.

3. **Segur**: Xarxes segmentades per VLANs, accés mínim necessari,
   secrets a 1Password amb accés per aplicació.

4. **Documentat**: Decisions registrades com a ADRs, runbooks per recuperació
   i operacions quotidianes.

5. **Minimalista**: Preferir serveis lleugers (Alpine, distroless). k3s en comptes
   de kubeadm. Prometheus amb regles senzilles.

6. **Autònom**: El sistema ha de funcionar sense intervenció humana.
   - FluxCD autocorregeix desviacions
   - Failed pods es reinicien automàticament
   - Alertes només per situacions que requereixin acció humana

7. **Observable**: Mètriques, logs i traces de tots els serveis.
   Dashboards a Grafana per visibilitat immediata.

8. **Econòmic**: Recursos limitats → decisions intel·ligents.
   No sobre-dimensionar. Contenidors, no VMs pesades.

## Flux de tràfic

### Petició web externa
```
Usuari → Internet → Router → Pangolin (VPS Hetzner) → Traefik (k3s)
                                                         ↓
                                                    Servei (pod)
                                                         ↓
                                          Base de dades / Cache (pod)
```

### Accés intern (LAN)
```
Usuari → Router → VLAN 20 (k3s) → Servei (pod, ClusterIP/NodePort)
                                         ↓
                            VLAN 30 (NFS) → QNAP NAS
```

## Seguretat

- **Perímetre**: Un sol port obert (443) al router. Traefik al cluster gestiona
  TLS i routing intern.
- **Xarxes**: VLANs aïllen tràfic de gestió, kubernetes, emmagatzematge i serveis.
  Només tràfic autoritzat entre VLANs.
- **Autenticació**: Authentik (SSO) per tots els serveis accessibles via web.
- **Secrets**: 1Password Connect com a servei extern de secrets. Mai al repositori.
- **Actualitzacions**: FluxCD actualitza automàticament les imatges.
  Ansible per actualitzacions de sistema als nodes LXC.

## Notes tècniques

- **k3s Traefik**: El Traefik integrat de k3s es substitueix per una instància
  configurada via HelmRelease (FluxCD).
- **etcd**: Clúster etcd embedded de k3s amb 3 control planes per HA.
- **Backups PBS**: Proxmox Backup Server per snapshots incrementals dels LXC.
- **Longhorn**: Emmagatzematge distribuït amb replicació per PVs del cluster.
