# Estructura OpenTofu — Organització multi-mòdul i exemples avançats

## Índex

- [Organització del repositori](#organització-del-repositori)
- [Root module vs child module](#root-module-vs-child-module)
- [Exemples](#exemples)
  - [Mòdul de xarxa (VLANs, subnets)](#1-mòdul-de-xarxa-vlans-subnets)
  - [Mòdul de DNS (registres)](#2-mòdul-de-dns-registres)
  - [Backend S3 per estat compartit](#3-backend-s3-per-estat-compartit)
  - [Com cridar mòduls des d'un root module](#4-com-cridar-mòduls-des-dun-root-module)
  - [Remote state data source](#5-remote-state-data-source)
- [Estàndards del repositori](#estàndards-del-repositori)

---

## Organització del repositori

```
opentofu/
├── modules/                          ← Child modules reutilitzables
│   ├── proxmox-lxc/                  ← Mòdul per LXC a Proxmox
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   ├── network/                      ← Mòdul de xarxa (VLANs, subnets)
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   └── dns/                          ← Mòdul de DNS (registres)
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── README.md
│
├── stacks/                           ← Root modules per entorn/stack
│   ├── infrastructure/               ← Stack d'infraestructura base
│   │   ├── main.tf                   ← Crida als mòduls
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── terraform.tfvars          ← Valors per defecte (gitignored si sensitiu)
│   │   └── backend.tf                ← Configuració del backend
│   ├── network/                      ← Stack de xarxa (VLANs)
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── backend.tf
│   └── dns/                          ← Stack de DNS
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── backend.tf
│
├── global/                           ← Config global compartida
│   ├── providers.tf                  ← Providers globals
│   └── variables-global.tf           ← Variables comunes
│
├── scripts/                          ← Scripts d'automatització
│   ├── init.sh                       ← Inicialització de workspaces
│   └── plan-all.sh                   ← Plan de tots els stacks
│
├── .gitignore
├── .terraform-version                ← Versió d'OpenTofu
└── README.md
```

**Cada stack** té el seu propi directori amb:
- `main.tf` — Crides als mòduls
- `variables.tf` — Variables d'entrada del stack
- `outputs.tf` — Outputs que exposa
- `backend.tf` — On es desa l'estat (S3, local per dev)

---

## Root module vs child module

### Root module

És el directori des d'on s'executa `tofu plan/apply`. Té un backend associat i crida child modules.

```hcl
# stacks/infrastructure/main.tf — Exemple de root module
terraform {
  required_version = ">= 1.6"

  required_providers {
    proxmox = {
      source  = "bpg/proxmox"
      version = "~> 0.60"
    }
  }
}

provider "proxmox" {
  endpoint = "https://[PROXMOX_HOST]:8006/api2/json"
  username = "[PROXMOX_USER]@pam"
  password = var.proxmox_password
  insecure = true
}

module "web_lxc" {
  source       = "../../modules/proxmox-lxc"
  proxmox_node = "[PROXMOX_NODE]"
  hostname     = "web-[NUM]-[DOMAIN]"
  vm_id        = [VM_ID]
  cpu_cores    = 2
  memory_dedicated = 2048
  disk_size    = 20
  network_ip   = "192.168.[VLAN].[IP]/24"
  network_gateway = "192.168.[VLAN].1"
  network_dns  = ["192.168.1.1", "1.1.1.1"]
  ssh_public_keys = var.ssh_public_keys
  tags         = ["web", "[ENV]", "lxc"]
}
```

**Característiques:**
- Té backend configurat (S3, local, etc.)
- Defineix els providers
- Crida mòduls amb `module`
- Pot tenir `data` sources per llegir informació externa

### Child module

És un directori dins `modules/` que encapsula un recurs o grup de recursos. Es reutilitza des de múltiples root modules.

```hcl
# modules/proxmox-lxc/main.tf — Exemple de child module (veure README.md per versió completa)
```

**Característiques:**
- Té fitxers `variables.tf`, `outputs.tf`, `main.tf`
- NO té backend ni providers (els hereda del root module)
- És reutilitzable: un mateix mòdul pot crear 10 LXC des de diferents root modules
- Publica `outputs` informatius per a qui el crida

| Aspecte | Root module | Child module |
|---------|-------------|--------------|
| Executa `tofu plan/apply` | ✅ | ❌ |
| Té backend | ✅ | ❌ |
| Defineix providers | ✅ | ❌ |
| Es reutilitza | ❌ (un sol ús per stack) | ✅ |
| Té `variables.tf` | ✅ | ✅ |
| Té `outputs.tf` | ✅ | ✅ |

---

## Exemples

### 1. Mòdul de xarxa (VLANs, subnets)

Mòdul per gestionar VLANs i subnets a un switch gestionable o a Proxmox.

```hcl
# modules/network/main.tf
terraform {
  required_version = ">= 1.6"

  required_providers {
    proxmox = {
      source  = "bpg/proxmox"
      version = "~> 0.60"
    }
  }
}

# Crear VLANs a Proxmox (com a tags de bridges existents)
resource "proxmox_virtual_environment_network" "vlan" {
  for_each = var.vlans

  node_name = var.proxmox_node

  bridge {
    name   = each.value.bridge
    vlan   = each.value.vlan_id
    cidr   = each.value.cidr
    gateway = each.value.gateway
    comment = each.value.comment
  }
}

# Crear subnets lògiques (per documentació/inventari)
# Nota: Això és un exemple conceptual — Proxmox no té un recurs "subnet"
# Substituir pel recurs real del vostre provider
```

```hcl
# modules/network/variables.tf
---
variable "proxmox_node" {
  description = "Node de Proxmox on configurar les VLANs"
  type        = string
}

variable "vlans" {
  description = "Mapa de VLANs per crear"
  type = map(object({
    vlan_id  = number
    bridge   = string
    cidr     = string
    gateway  = string
    comment  = string
  }))
  default = {}
}
```

```hcl
# modules/network/outputs.tf
---
output "vlan_ids" {
  description = "IDs de les VLANs creades"
  value       = { for k, v in var.vlans : k => v.vlan_id }
}

output "vlan_cidrs" {
  description = "CIDRs de les VLANs"
  value       = { for k, v in var.vlans : k => v.cidr }
}
```

**Exemple de crida al mòdul:**

```hcl
module "homelab_network" {
  source       = "../../modules/network"
  proxmox_node = "[PROXMOX_NODE]"

  vlans = {
    management = {
      vlan_id  = 10
      bridge   = "vmbr0"
      cidr     = "192.168.10.0/24"
      gateway  = "192.168.10.1"
      comment  = "Xarxa de gestió"
    }
    infrastructure = {
      vlan_id  = 20
      bridge   = "vmbr0"
      cidr     = "192.168.20.0/24"
      gateway  = "192.168.20.1"
      comment  = "Xarxa d'infraestructura"
    }
    k8s = {
      vlan_id  = 30
      bridge   = "vmbr0"
      cidr     = "192.168.30.0/24"
      gateway  = "192.168.30.1"
      comment  = "Xarxa de Kubernetes"
    }
  }
}
```

---

### 2. Mòdul de DNS (registres)

Mòdul per gestionar registres DNS a un provider (ex: Hetzner, Cloudflare, bind).

```hcl
# modules/dns/main.tf
terraform {
  required_version = ">= 1.6"

  required_providers {
    dns = {
      source  = "hashicorp/dns"
      version = "~> 3.4"
    }
  }
}

# Provedor DNS (per exemple, amb provider genèric DNS)
# Nota: Substituir pel vostre provider real (cloudflare, hetznerdns, etc.)

resource "dns_a_record_set" "host" {
  for_each = var.a_records

  zone = var.zone
  name = each.value.name
  addresses = each.value.addresses
  ttl = each.value.ttl
}

resource "dns_cname_record" "cname" {
  for_each = var.cname_records

  zone = var.zone
  name = each.value.name
  cname = each.value.cname
  ttl = each.value.ttl
}

resource "dns_srv_record" "srv" {
  for_each = var.srv_records

  zone = var.zone
  name = each.value.name
  service = each.value.service
  protocol = each.value.protocol
  priority = each.value.priority
  weight = each.value.weight
  port = each.value.port
  target = each.value.target
  ttl = each.value.ttl
}
```

```hcl
# modules/dns/variables.tf
---
variable "zone" {
  description = "Zona DNS (ex: example.org)"
  type        = string
}

variable "a_records" {
  description = "Registres A"
  type = map(object({
    name      = string
    addresses = list(string)
    ttl       = optional(number, 300)
  }))
  default = {}
}

variable "cname_records" {
  description = "Registres CNAME"
  type = map(object({
    name  = string
    cname = string
    ttl   = optional(number, 300)
  }))
  default = {}
}

variable "srv_records" {
  description = "Registres SRV"
  type = map(object({
    name     = string
    service  = string
    protocol = string
    priority = number
    weight   = number
    port     = number
    target   = string
    ttl      = optional(number, 300)
  }))
  default = {}
}
```

```hcl
# modules/dns/outputs.tf
---
output "a_record_names" {
  description = "Noms dels registres A creats"
  value       = keys(var.a_records)
}

output "cname_record_names" {
  description = "Noms dels registres CNAME creats"
  value       = keys(var.cname_records)
}
```

**Exemple de crida al mòdul:**

```hcl
module "homelab_dns" {
  source = "../../modules/dns"
  zone   = "[DOMAIN]"

  a_records = {
    proxmox = {
      name      = "proxmox"
      addresses = ["192.168.10.10"]
      ttl       = 3600
    }
    k8s_api = {
      name      = "k8s"
      addresses = ["192.168.30.10", "192.168.30.11", "192.168.30.12"]
      ttl       = 300
    }
    grafana = {
      name      = "grafana"
      addresses = ["192.168.20.50"]
      ttl       = 300
    }
  }

  cname_records = {
    www = {
      name  = "www"
      cname = "[DOMAIN]"
      ttl   = 3600
    }
  }
}
```

---

### 3. Backend S3 per estat compartit

Configuració per desar l'estat d'OpenTofu en un bucket S3 (o compatible: MinIO, Garage, Ceph RGW).

```hcl
# stacks/[STACK_NAME]/backend.tf
terraform {
  backend "s3" {
    bucket = "[BUCKET_NAME]"
    key    = "[STACK_NAME]/terraform.tfstate"
    region = "[REGION]"

    # Per S3 compatible (MinIO, Garage)
    endpoints = {
      s3 = "https://s3.[DOMAIN]"
    }

    # Autenticació via variables d'entorn preferiblement
    # AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY

    # Deshabilitar checks de regió per S3 compatible
    skip_region_validation      = true
    skip_credentials_validation = true
    skip_requesting_account_id  = true
    skip_s3_checksum            = true

    # Xifrat en repòs (opcional)
    encrypt = true
  }
}
```

**Variables d'entorn necessàries:**

```bash
# .envrc (direnv) o fitxer d'entorn
export AWS_ACCESS_KEY_ID="[S3_ACCESS_KEY]"
export AWS_SECRET_ACCESS_KEY="[S3_SECRET_KEY]"
export AWS_S3_ENDPOINT="https://s3.[DOMAIN]"
```

**Estructura d'estats recomanada al bucket:**

```
[BUCKET_NAME]/
├── infrastructure/terraform.tfstate    ← LXC, VMs, xarxa
├── network/terraform.tfstate           ← VLANs, subnets
├── dns/terraform.tfstate              ← Registres DNS
├── k8s/terraform.tfstate              ← Recursos Kubernetes
└── global/terraform.tfstate          ← Config global (cross-stack)
```

**Bloqueig d'estat (dynamodb o equivalent):**

```hcl
terraform {
  backend "s3" {
    bucket         = "[BUCKET_NAME]"
    key            = "[STACK_NAME]/terraform.tfstate"
    region         = "[REGION]"
    dynamodb_table = "[LOCK_TABLE_NAME]"  # Per bloqueig d'estat
    encrypt        = true
  }
}
```

> **Nota:** Si uses S3 compatible sense DynamoDB, el bloqueig d'estat no funciona. Per evitar corrupció d'estat, assegurar-se que només una persona executa `apply` alhora.

---

### 4. Com cridar mòduls des d'un root module

Exemple complet d'un root module que crida múltiples child modules i combina els seus outputs.

```hcl
# stacks/infrastructure/main.tf — Root module complet
terraform {
  required_version = ">= 1.6"

  required_providers {
    proxmox = {
      source  = "bpg/proxmox"
      version = "~> 0.60"
    }
    dns = {
      source  = "hashicorp/dns"
      version = "~> 3.4"
    }
  }
}

# Provider Proxmox
provider "proxmox" {
  endpoint = "https://[PROXMOX_HOST]:8006/api2/json"
  username = "[PROXMOX_USER]@pam"
  password = var.proxmox_password
  insecure = true
}

# ──────────── Xarxa ────────────

module "network" {
  source       = "../../modules/network"
  proxmox_node = "[PROXMOX_NODE]"

  vlans = var.vlans
}

# ──────────── LXC: Servidors web ────────────

module "web_servers" {
  source = "../../modules/proxmox-lxc"
  count  = length(var.web_servers)

  proxmox_node      = var.web_servers[count.index].proxmox_node
  hostname          = var.web_servers[count.index].hostname
  vm_id             = var.web_servers[count.index].vm_id
  cpu_cores         = var.web_servers[count.index].cpu_cores
  memory_dedicated  = var.web_servers[count.index].memory_dedicated
  disk_size         = var.web_servers[count.index].disk_size
  network_ip        = var.web_servers[count.index].network_ip
  network_gateway   = var.web_servers[count.index].network_gateway
  network_dns       = var.web_servers[count.index].network_dns
  network_vlan      = var.web_servers[count.index].network_vlan
  ssh_public_keys   = var.ssh_public_keys
  tags              = concat(["web"], var.web_servers[count.index].extra_tags)
  start_on_boot     = true
}

# ──────────── DNS ────────────

module "dns_records" {
  source = "../../modules/dns"
  zone   = "[DOMAIN]"

  a_records = {
    for i, server in var.web_servers :
    server.hostname => {
      name      = server.hostname
      addresses = [split("/", server.network_ip)[0]]
      ttl       = 300
    }
  }
}

# ──────────── Outputs combinats ────────────

output "summary" {
  description = "Resum de tota la infra desplegada"
  value = {
    vlans     = module.network.vlan_ids
    web_hosts = {
      for i, server in var.web_servers :
      server.hostname => {
        ip  = split("/", server.network_ip)[0]
        id  = module.web_servers[i].container_vm_id
        dns = "https://${server.hostname}.[DOMAIN]"
      }
    }
  }
}
```

```hcl
# stacks/infrastructure/variables.tf
---
variable "proxmox_password" {
  description = "Contrasenya de l'usuari Proxmox"
  type        = string
  sensitive   = true
}

variable "ssh_public_keys" {
  description = "Claus SSH públiques per accés als contenidors"
  type        = list(string)
}

variable "vlans" {
  description = "Mapa de VLANs"
  type = map(object({
    vlan_id  = number
    bridge   = string
    cidr     = string
    gateway  = string
    comment  = string
  }))
}

variable "web_servers" {
  description = "Llista de servidors web per crear"
  type = list(object({
    hostname         = string
    proxmox_node     = string
    vm_id            = number
    cpu_cores        = number
    memory_dedicated = number
    disk_size        = number
    network_ip       = string
    network_gateway  = string
    network_dns      = list(string)
    network_vlan     = number
    extra_tags       = optional(list(string), [])
  }))
}
```

---

### 5. Remote state data source

Permet llegir outputs d'un altre stack per utilitzar-los sense duplicar dades.

```hcl
# stacks/dns/main.tf — Stack de DNS que llegeix IPs del stack d'infra
terraform {
  required_version = ">= 1.6"

  required_providers {
    dns = {
      source  = "hashicorp/dns"
      version = "~> 3.4"
    }
  }
}

# Llegir l'estat remot del stack d'infraestructura
data "terraform_remote_state" "infrastructure" {
  backend = "s3"

  config = {
    bucket = "[BUCKET_NAME]"
    key    = "infrastructure/terraform.tfstate"
    region = "[REGION]"

    endpoints = {
      s3 = "https://s3.[DOMAIN]"
    }
    skip_region_validation      = true
    skip_credentials_validation = true
    skip_requesting_account_id  = true
    skip_s3_checksum            = true
  }
}

# Utilitzar els outputs de l'altre stack
locals {
  web_hosts = data.terraform_remote_state.infrastructure.outputs.summary.web_hosts
}

resource "dns_a_record_set" "auto" {
  for_each = local.web_hosts

  zone      = "[DOMAIN]"
  name      = each.key
  addresses = [each.value.ip]
  ttl       = 300
}

output "dns_records_created" {
  description = "Registres DNS creats automàticament"
  value       = keys(local.web_hosts)
}
```

**Beneficis del remote state:**

- No dupliques dades entre stacks (les IPs es defineixen a infra, es llegeixen a DNS)
- Un canvi d'IP a infra es reflecteix automàticament al DNS
- Pots dividir el repositori en stacks independents amb responsabilitats clares
- Cada stack es pot planificar i aplicar per separat

**Consideracions:**

- L'estat remot ha d'existir abans de poder-lo llegir
- Si canvies l'estructura d'outputs d'un stack, has d'actualitzar tots els que en depenen
- Evita dependències circulars (stack A llegeix B, B llegeix A → error)

---

## Estàndards del repositori

- **Mòduls reutilitzables**: Cada recurs al seu propi mòdul dins `modules/`
- **Stacks separats**: No barrejar recursos no relacionats al mateix `main.tf`
- **Variables documentades**: Totes les variables tenen `description`, `type` i `default` (si escau)
- **Outputs informatius**: Cada mòdul i stack exposa outputs útils
- **Secrets**: Mai al repositori — usar variables d'entorn o `terraform.tfvars` al `.gitignore`
- **Backend S3**: Tots els stacks usen backend remot, mai local
- **Versió d'OpenTofu**: Fixada a `.terraform-version` per consistència entre desenvolupadors
- **Format**: `tofu fmt` abans de cada commit
- **Documentació**: `terraform-docs` per generar README dels mòduls
- **Tags**: Tags descriptives per filtrar recursos per funció, entorn i tipus

---

## Referències

- [OpenTofu Documentation](https://opentofu.org/docs/)
- [OpenTofu Modules](https://opentofu.org/docs/language/modules/)
- [Remote State Data Source](https://opentofu.org/docs/language/state/remote-state-data/)
- [S3 Backend](https://opentofu.org/docs/language/settings/backends/s3/)
- [terraform-docs](https://terraform-docs.io/)
