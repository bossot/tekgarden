# Plantilles Ansible

Rols i playbooks anonimitzats per gestió de configuracions d'homelab. Basats en Debian per als nodes del cluster.

## Índex

- [Playbook per LXC Debian](#playbook-per-lxc-debian)
- [Variables anonimitzades](#variables-anonimitzades)
- [Handler de serveis](#handler-de-serveis)
- [Estàndards](#estàndards)

## Playbook per LXC Debian

Playbook bàsic per aprovisionar un contenidor LXC Debian amb configuració de xarxa, usuaris i serveis.

```yaml
# playbook-lxc.yml
# Aprovisionament bàsic d'un contenidor LXC Debian
---
- name: Bastir contenidor LXC
  hosts: [HOST_GROUP]
  gather_facts: yes
  become: yes

  vars_files:
    - vars/main.yml

  pre_tasks:
    - name: Esperar que el sistema estigui disponible
      ansible.builtin.wait_for_connection:
        delay: 5
        timeout: 120

    - name: Actualitzar cache de paquets
      ansible.builtin.apt:
        update_cache: yes
        cache_valid_time: 3600

  tasks:
    # --- Configuració de xarxa ---
    - name: Configurar DNS
      ansible.builtin.copy:
        dest: /etc/resolv.conf
        content: |
          nameserver [DNS_PRIMARY]
          nameserver [DNS_SECONDARY]
        mode: "0644"

    - name: Configurar hosts
      ansible.builtin.lineinfile:
        path: /etc/hosts
        regexp: "^{{ ansible_default_ipv4.address }}"
        line: "{{ ansible_default_ipv4.address }} {{ inventory_hostname }}.{{ domain_name }} {{ inventory_hostname }}"
        state: present

    # --- Usuari administrador ---
    - name: Crear grup admin
      ansible.builtin.group:
        name: "{{ admin_group }}"
        state: present

    - name: Crear usuari administrador
      ansible.builtin.user:
        name: "{{ admin_user }}"
        group: "{{ admin_group }}"
        groups: sudo
        shell: /bin/bash
        create_home: yes
        state: present

    - name: Afegir clau SSH pública
      ansible.posix.authorized_key:
        user: "{{ admin_user }}"
        key: "{{ ssh_public_key }}"
        state: present

    - name: Configurar sudo sense contrasenya
      ansible.builtin.lineinfile:
        path: /etc/sudoers.d/{{ admin_user }}
        line: "{{ admin_user }} ALL=(ALL) NOPASSWD:ALL"
        create: yes
        mode: "0440"
        validate: "visudo -cf %s"

    # --- Seguretat bàsica ---
    - name: Deshabilitar login root per SSH
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "^PermitRootLogin"
        line: "PermitRootLogin no"
      notify: restart ssh

    - name: Deshabilitar autenticació per contrasenya
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "^PasswordAuthentication"
        line: "PasswordAuthentication no"
      notify: restart ssh

    - name: Configurar firewall (ufw)
      community.general.ufw:
        rule: allow
        port: "{{ item.port }}"
        proto: "{{ item.proto }}"
      loop:
        - { port: "22", proto: "tcp" }
        - { port: "6443", proto: "tcp" }
        - { port: "8472", proto: "udp" }
        - { port: "51820", proto: "udp" }

    - name: Habilitar UFW
      community.general.ufw:
        state: enabled
        policy: deny

    # --- Serveis essencials ---
    - name: Instal·lar paquets base
      ansible.builtin.apt:
        name: "{{ base_packages }}"
        state: present

    - name: Habilitar i iniciar servei [SERVICE_NAME]
      ansible.builtin.systemd:
        name: "[SERVICE_NAME]"
        enabled: yes
        state: started
        daemon_reload: yes

    # --- Configuració específica del servei ---
    - name: Desplegar configuració de [SERVICE_NAME]
      ansible.builtin.template:
        src: templates/[SERVICE_NAME].conf.j2
        dest: /etc/[SERVICE_NAME]/[SERVICE_NAME].conf
        mode: "0644"
      notify: restart [SERVICE_NAME]

  handlers:
    - name: restart ssh
      ansible.builtin.systemd:
        name: ssh
        state: restarted

    - name: restart [SERVICE_NAME]
      ansible.builtin.systemd:
        name: "[SERVICE_NAME]"
        state: restarted
```

## Variables anonimitzades

```yaml
# vars/main.yml
---
# --- Xarxa ---
domain_name: "[DOMAIN]"
dns_primary: "[DNS_PRIMARY]"
dns_secondary: "[DNS_SECONDARY]"
ip_range: "[IP_RANGE]"

# --- Usuaris ---
admin_user: "[ADMIN_USER]"
admin_group: "[ADMIN_GROUP]"
ssh_public_key: "[SSH_PUBLIC_KEY]"

# --- Repositori de paquets ---
apt_mirror: "[APT_MIRROR_URL]"

# --- Paquets base ---
base_packages:
  - curl
  - wget
  - htop
  - vim
  - git
  - ufw
  - fail2ban
  - logrotate
  - chrony
  - tmux

# --- Servei específic ---
service_name: "[SERVICE_NAME]"
service_port: [SERVICE_PORT]
service_version: "[SERVICE_VERSION]"
service_config_path: "/etc/{{ service_name }}/{{ service_name }}.conf"

# --- Logs i monitoratge ---
log_retention_days: 30
monitoring_enabled: yes
metrics_port: [METRICS_PORT]
```

## Handler de serveis

```yaml
# handlers/main.yml
---
- name: restart ssh
  ansible.builtin.systemd:
    name: ssh
    state: restarted
    daemon_reload: yes

- name: restart [SERVICE_NAME]
  ansible.builtin.systemd:
    name: "[SERVICE_NAME]"
    state: restarted
    daemon_reload: yes

- name: reload [SERVICE_NAME]
  ansible.builtin.systemd:
    name: "[SERVICE_NAME]"
    state: reloaded

- name: restart ufw
  community.general.ufw:
    state: reloaded

- name: restart fail2ban
  ansible.builtin.systemd:
    name: fail2ban
    state: restarted

- name: restart chrony
  ansible.builtin.systemd:
    name: chrony
    state: restarted

- name: restart rsyslog
  ansible.builtin.systemd:
    name: rsyslog
    state: restarted
```

## Estàndards

- **Variables tipades**: Sempre amb tipus explícit (bool, int, str) i valors per defecte
- **Tags per fases**: `network`, `users`, `security`, `packages`, `service` per executar només parts específiques
- **Handlers**: Totes les accions que requereixin reinici van amb notify a handler
- **Idempotència**: Cada tasca ha de ser segura d'executar múltiples vegades
- **Secrets**: Mai al repositori — usar Ansible Vault o lookup amb 1Password CLI
- **Estructura de rols**: Cada rol amb tasks/, handlers/, templates/, vars/ i defaults/
- **Comentaris**: En català per la documentació, noms de variables en anglès
