# Arquitectura del TekGarden

Visió general del sistema a alt nivell.

## Components principals

- **Hypervisor**: Proxmox VE per màquines virtuals i contenidors LXC
- **Kubernetes**: Cluster k3s amb 3 nodes de control i 3 workers
- **Emmagatzematge**: NAS local amb RAID10 per dades persistents
- **Xarxa**: VLANs separades per serveis, seguretat i aïllament
- **DNS**: Pi-hole (migració a AdGuard Home prevista)
- **GitOps**: FluxCD per desplegament declaratiu
- **Infraestructura com a codi**: OpenTofu per provisionament Proxmox
- **Configuració**: Ansible per gestió d'estat dels nodes
- **Monitoratge**: Prometheus + Grafana amb alertes
- **Logs**: Loki + Grafana per logs centralitzats

## Principis de disseny

1. **Declaratiu**: Tot el que es pugui definir en codi, es defineix en codi
2. **Reproduïble**: El cluster es pot recrear des de zero seguint la documentació
3. **Segur**: Xarxes segmentades, accés mínim necessari, secrets a 1Password
4. **Documentat**: Decisions registrades com a ADRs, runbooks per recuperació
