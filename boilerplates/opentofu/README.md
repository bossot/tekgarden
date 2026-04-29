# Mòduls OpenTofu

Infraestructura com a codi per provisionament de recursos a Proxmox VE amb OpenTofu. Tots els exemples estan anonimitzats.

## Índex

- [Mòdul per LXC a Proxmox](#mòdul-per-lxc-a-proxmox)
- [Variables del mòdul](#variables-del-mòdul)
- [Outputs del mòdul](#outputs-del-mòdul)
- [Exemple d'ús](#exemple-dús)
- [Estàndards](#estàndards)

## Mòdul per LXC a Proxmox

Mòdul reutilitzable per crear contenidors LXC a Proxmox VE. Admet configuració de xarxa, emmagatzematge i recursos.

```hcl
# main.tf — Mòdul per crear contenidors LXC a Proxmox
terraform {
  required_version = ">= 1.6"

  required_providers {
    proxmox = {
      source  = "bpg/proxmox"
      version = "~> 0.60"
    }
  }
}

# Obtenir la darrera plantilla disponible
data "proxmox_virtual_environment_download_url" "template" {
  count = var.create_vm ? 1 : 0

  content_type = "iso"
  node_name    = var.proxmox_node
  url          = var.template_url

  checksum         = var.template_checksum
  checksum_algorithm = var.template_checksum_algorithm
  filename         = var.template_filename
}

# Crear el contenidor LXC
resource "proxmox_virtual_environment_container" "lxc" {
  count = var.create_vm ? 1 : 0

  node_name = var.proxmox_node
  vm_id     = var.vm_id

  start_on_boot = var.start_on_boot
  unprotected   = var.unprotected

  # Plantilla
  template {
    file_id = data.proxmox_virtual_environment_download_url.template[0].id
  }

  # Recursos
  cpu {
    cores = var.cpu_cores
  }

  memory {
    dedicated = var.memory_dedicated
    swap      = var.memory_swap
  }

  disk {
    datastore_id = var.datastore_id
    size         = var.disk_size
  }

  # Xarxa
  network_interface {
    name   = "eth0"
    bridge = var.network_bridge
    vlan   = var.network_vlan
    ip     = var.network_ip
    gw     = var.network_gateway
    dns    = var.network_dns
  }

  # Opcions del contenidor
  operating_system {
    template_file_id = var.template_id
    type             = var.os_type
  }

  # Configuració d'inici
  initialization {
    hostname = var.hostname
    user_account {
      username = var.ssh_username
      keys     = var.ssh_public_keys
    }
    ip_config {
      ipv4 {
        address = var.network_ip
        gateway = var.network_gateway
      }
    }
    dns {
      servers = var.network_dns
    }
  }

  # Hooks i features
  features {
    nesting = var.feature_nesting
    fuse    = var.feature_fuse
  }

  tags = var.tags
}
```

## Variables del mòdul

```hcl
# variables.tf
---
variable "create_vm" {
  description = "Crear o no el contenidor (per desactivar temporalment)"
  type        = bool
  default     = true
}

variable "proxmox_node" {
  description = "Node de Proxmox on crear el contenidor"
  type        = string
}

variable "vm_id" {
  description = "ID del contenidor (deixar buit per assignació automàtica)"
  type        = number
  default     = null
}

variable "hostname" {
  description = "Hostname del contenidor"
  type        = string
}

variable "template_id" {
  description = "ID de la plantilla al datastore de Proxmox"
  type        = string
}

variable "template_url" {
  description = "URL de la imatge del sistema (ex: Debian cloud image)"
  type        = string
  default     = "https://cloud-images.debian.org/debian/current/debian-12-genericcloud-amd64.qcow2"
}

variable "template_checksum" {
  description = "Checksum SHA256 de la plantilla"
  type        = string
  default     = ""
}

variable "template_checksum_algorithm" {
  description = "Algorisme de checksum (sha256, sha512, md5)"
  type        = string
  default     = "sha256"
}

variable "template_filename" {
  description = "Nom del fitxer de la plantilla al datastore"
  type        = string
  default     = null
}

variable "cpu_cores" {
  description = "Nombre de cores de CPU"
  type        = number
  default     = 2
}

variable "memory_dedicated" {
  description = "RAM dedicada en MB"
  type        = number
  default     = 2048
}

variable "memory_swap" {
  description = "Swap en MB (0 per deshabilitar)"
  type        = number
  default     = 512
}

variable "disk_size" {
  description = "Mida del disc en GB"
  type        = number
  default     = 20
}

variable "datastore_id" {
  description = "Identificador del datastore (ex: local-zfs, local-lvm)"
  type        = string
  default     = "local-zfs"
}

variable "network_bridge" {
  description = "Bridge de xarxa (ex: vmbr0)"
  type        = string
  default     = "vmbr0"
}

variable "network_vlan" {
  description = "VLAN ID (deixar buit per sense VLAN)"
  type        = number
  default     = null
}

variable "network_ip" {
  description = "Adreça IP amb CIDR (ex: 192.168.1.10/24)"
  type        = string
}

variable "network_gateway" {
  description = "Gateway de la xarxa"
  type        = string
}

variable "network_dns" {
  description = "Servidors DNS"
  type        = list(string)
  default     = ["192.168.1.1", "1.1.1.1"]
}

variable "ssh_username" {
  description = "Usuari SSH per defecte"
  type        = string
  default     = "admin"
}

variable "ssh_public_keys" {
  description = "Claus SSH públiques per accés al contenidor"
  type        = list(string)
}

variable "start_on_boot" {
  description = "Iniciar el contenidor en arrencar Proxmox"
  type        = bool
  default     = true
}

variable "unprotected" {
  description = "Permetre destruir el contenidor des de l'UI"
  type        = bool
  default     = false
}

variable "os_type" {
  description = "Tipus de SO (debian, ubuntu, alpine, etc.)"
  type        = string
  default     = "debian"
}

variable "feature_nesting" {
  description = "Habilitar nesting (contenidors dins del contenidor)"
  type        = bool
  default     = false
}

variable "feature_fuse" {
  description = "Habilitar FUSE (per muntar sistemes de fitxers)"
  type        = bool
  default     = false
}

variable "tags" {
  description = "Tags per categoritzar el contenidor"
  type        = list(string)
  default     = []
}
```

## Outputs del mòdul

```hcl
# outputs.tf
---
output "container_id" {
  description = "ID del contenidor LXC"
  value       = try(proxmox_virtual_environment_container.lxc[0].id, null)
}

output "container_vm_id" {
  description = "VM ID del contenidor a Proxmox"
  value       = try(proxmox_virtual_environment_container.lxc[0].vm_id, null)
}

output "hostname" {
  description = "Hostname del contenidor"
  value       = try(proxmox_virtual_environment_container.lxc[0].hostname, null)
}

output "ip_address" {
  description = "Adreça IP del contenidor"
  value       = var.network_ip
}

output "network_interface" {
  description = "Informació de la interfície de xarxa"
  value = try(proxmox_virtual_environment_container.lxc[0].network_interface, null)
}

output "template_id" {
  description = "ID de la plantilla usada"
  value       = var.template_id
}

output "resource_summary" {
  description = "Resum de recursos del contenidor"
  value = try({
    cpu_cores      = var.cpu_cores
    memory_mb      = var.memory_dedicated
    disk_gb        = var.disk_size
    datastore      = var.datastore_id
  }, null)
}
```

## Exemple d'ús

```hcl
# exemple/main.tf — Exemple de crida al mòdul LXC
terraform {
  required_version = ">= 1.6"

  required_providers {
    proxmox = {
      source  = "bpg/proxmox"
      version = "~> 0.60"
    }
  }

  backend "s3" {
    bucket = "[TFSTATE_BUCKET]"
    key    = "proxmox/[NODE_NAME]/[VM_NAME]/terraform.tfstate"
    region = "[REGION]"
  }
}

provider "proxmox" {
  endpoint = "https://[PROXMOX_HOST]:8006/api2/json"
  username = "[PROXMOX_USER]@pam"
  password = var.proxmox_password

  # No verificar certificat autosignat de Proxmox
  insecure = true
}

# Crida al mòdul per crear un contenidor LXC
module "web_server" {
  source = "../"

  proxmox_node = "[PROXMOX_NODE]"
  hostname     = "web-[NUM]-[DOMAIN]"
  vm_id        = [VM_ID]

  template_id = "[TEMPLATE_NAME]"

  cpu_cores      = 2
  memory_dedicated = 2048
  disk_size      = 20

  network_ip      = "192.168.[VLAN].[IP]/24"
  network_gateway = "192.168.[VLAN].1"
  network_dns     = ["192.168.1.1", "1.1.1.1"]
  network_vlan    = [VLAN_ID]

  ssh_public_keys  = ["ssh-ed25519 [KEY_PUB] [USER]"]
  ssh_username     = "[ADMIN_USER]"

  start_on_boot    = true
  feature_nesting  = true

  tags = ["web", "[ENV]", "lxc"]
}

output "web_server_info" {
  description = "Informació del servidor web creat"
  value = {
    id       = module.web_server.container_id
    hostname = module.web_server.hostname
    ip       = module.web_server.ip_address
  }
}
```

## Estàndards

- **Outputs sempre definits**: Cada mòdul exposa outputs útils per a qui l'utilitzi
- **Variables amb descripció**: Totes les variables tenen `description` i `type`
- **Backend remot**: S3 (o compatible) per l'estat de terraform — mai local
- **Tags descriptives**: Tags per filtrar recursos per funció, entorn i tipus
- **Secrets**: Mai al repositori — usar variables d'entorn o `terraform.tfvars` al gitignore
- **Mòduls reutilitzables**: Cada recurs amb el seu propi mòdul i path `modules/`
- **Documentació automàtica**: `terraform-docs` per generar README dels mòduls
