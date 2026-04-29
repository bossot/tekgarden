# Decisions d'Arquitectura (ADRs)

Registre de decisions d'arquitectura del TekGarden, anonimitzades per compartir.
Cada decisió segueix el format ADR (Architecture Decision Record).

## ADRs registrats

- **[ADR-001](./decisions.md#adr-001-elecció-de-k3s-com-a-distribució-kubernetes):** Elecció de k3s com a distribució Kubernetes
- **[ADR-002](./decisions.md#adr-002-fluxcd-per-gitops):** FluxCD per GitOps
- **[ADR-003](./decisions.md#adr-003-estructura-de-vlans-i-xarxes):** Estructura de VLANs i xarxes
- **[ADR-004](./decisions.md#adr-004-estratègia-de-backups):** Estratègia de backups

---

## ADR-001: Elecció de k3s com a distribució Kubernetes

**Data:** 2025-10-15
**Estat:** Acceptada
**Decisors:** Santi (Platform Engineer), Perple (Arquitecta)

### Context

El TekGarden necessita un orquestrador de contenidors per gestionar serveis com:
- Monitoratge (Prometheus, Grafana, Loki)
- Autenticació (Authentik)
- Gestió d'imatges (Immich)
- Lectors RSS (Miniflux)
- Reverse proxy (Traefik)

Requisits principals:
- Recursos limitats (6 nodes LXC, 2-4 GB RAM cadascun)
- Manteniment senzill (operat per una persona)
- Compatible amb FluxCD per GitOps
- Alta disponibilitat bàsica

### Decisió

Utilitzar **k3s** (de Rancher/SUSE) com a distribució Kubernetes.

### Raons

1. **Lleuger**: k3s funciona en 512 MB de RAM per node. Alternatives com kubeadm requereixen més recursos base.
2. **Instal·lació senzilla**: Un sol binari, un sol comando per node. Ideal per un homelab gestionat per una persona.
3. **Components integrats**: Traefik com a ingress controller per defecte, CoreDNS, i containerd com a runtime.
4. **High Availability**: Suport natiu per etcd (external o embedded) amb 3+ nodes de control.
5. **Manteniment**: Les actualitzacions són tan simples com substituir el binari i reiniciar el servei.
6. **Comunitat activa**: Àmpliament usat en homelabs i producció a petita escala.

### Conseqüències

**Positives:**
- Consum de recursos mínim (deixa espai per aplicacions)
- Desplegament ràpid (minuts, no hores)
- Compatible amb Helm i FluxCD (stack GitOps complet)
- Fàcil provar i destruir

**Negatives:**
- No té algunes features de K8s vanilla (ex: certs alpha APIs)
- Dependència de Rancher per suport comercial
- Base de dades SQLite per defecte (cal migrar a etcd per HA real)
- No recomanat per clusters massius (>100 nodes)

### Alternatives considerades

| Opció | Raó per no escollir-la |
|-------|------------------------|
| **kubeadm** | Massa pesat per als recursos disponibles |
| **MicroK8s** | Snap obligatori (no disponible a Debian minimal) |
| **RKE2** | Similar a k3s però més pesat |
| **Només Docker Compose** | Sense orquestració ni GitOps |

### Rollback

Si k3s resulta insuficient, migrar a RKE2 (mateix stack) o kubeadm amb etcd extern.

---

## ADR-002: FluxCD per GitOps

**Data:** 2025-10-20
**Estat:** Acceptada

### Context

Cal un sistema GitOps per gestionar els desplegaments al cluster k3s. El sistema ha de:
- Sincronitzar l'estat del cluster amb un repositori Git
- Detectar desviacions i corregir-les automàticament
- Gestionar secrets de forma segura
- Suportar Helm i manifests YAML

### Decisió

Utilitzar **FluxCD** (v2) com a eina GitOps.

### Raons

1. **Declaratiu pur**: Aplicar el que hi ha al repo, res més — sense dependències de servidor central
2. **Multi-tenant**: Suport natiu per separar per namespace o cluster
3. **Integració Helm**: HelmRelease com a recurs natiu de Flux
4. **OCIRepository**: Suport per charts en registres OCI
5. **Secret management**: Integració amb External Secrets Operator per 1Password
6. **Comunitat Flux**: Molt activa, millor documentació que alternatives

### Conseqüències

- Stack complet de GitOps establert
- Necessari formar-se en Kustomize i Helm
- FluxCD requereix un repositori dedicat (fluxcd)

---

## ADR-003: Estructura de VLANs i xarxes

**Data:** 2025-10-25
**Estat:** Acceptada

### Context

El TekGarden té múltiples subsistemes: Proxmox, k3s, QNAP NAS, DNS, i serveis diversos.
Cal segmentar la xarxa per seguretat i aïllament.

### Decisió

Crear les següents VLANs:

| VLAN ID | Nom | Subnet | Propòsit |
|---------|-----|--------|----------|
| 10 | management | 192.168.10.0/24 | Gestió de Proxmox, QNAP, IPMI |
| 20 | kubernetes | 192.168.20.0/24 | Cluster k3s (nodes i pods) |
| 30 | storage | 192.168.30.0/24 | Tràfic NFS/SMB entre QNAP i k3s |
| 40 | services | 192.168.40.0/24 | Serveis crítics (DNS, DHCP, VPN) |
| 50 | iot | 192.168.50.0/24 | Dispositius IoT (sense accés a internet) |
| 60 | guest | 192.168.60.0/24 | Xarxa convidats (WiFi aïllada) |
| 70 | dmz | 192.168.70.0/24 | Serveis exposats públicament |

### Conseqüències

- Aïllament complet entre subsistemes
- Regles de firewall permeses entre VLANs específiques (ex: k3s → storage)
- Complexitat de configuració, però guany en seguretat

---

## ADR-004: Estratègia de backups

**Data:** 2025-10-30
**Estat:** Acceptada

### Context

Cal una estratègia de backups que cobreixi:
- Configuracions dels nodes (LXC, Proxmox)
- Dades persistents (bases de dades, imatges)
- Secrets (1Password)
- Estat del cluster Kubernetes

### Decisió

Implementar la següent estratègia de 3-2-1:

1. **3 còpies dels serveis crítics**:
   - Còpia local al NAS (QNAP, RAID10)
   - Còpia externa a Hetzner Storage Box (via rsync/restic)
   - Còpia refrigerada a disc extern (manual, mensual)

2. **Eines**:
   - **Proxmox Backup Server** per snapshots dels LXC
   - **Restic** per backups xifrats de dades persistents
   - **Velero** per backups del cluster Kubernetes (PV, etcd)
   - **1Password** com a font única de secrets

3. **Programació**:
   - Snapshots LXC: diaris (retenció: 7 dies)
   - Backups de dades: cada 6 hores (retenció: 30 dies)
   - Backups externs: cada 24 hores
   - Backups refrigerats: mensuals

### Conseqüències

- Cost d'emmagatzematge extern (Hetzner Storage Box ~5€/mes)
- Complexitat de configuració dels diferents sistemes de backup
- Temps de restauració: ~30 min per node, ~2 h per cluster complet
- Documentació de runbooks essencial per no perdre'ls

---

*Fet amb 💚 des de Mallorca*
