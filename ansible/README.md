# Ansible Homelab

## Overview

This repository manages the configuration and provisioning of the homelab infrastructure.
It follows a structured, scalable approach based on Ansible best practices.

The inventory is split across three axes: functional (what the machine does),
technical (how Ansible manages it), and environmental (production vs staging).

---

## Repository Structure

```
ansible/
|-- ansible.cfg                         # Global Ansible configuration
|-- README.md                           # This file
|
|-- inventory/
|   |-- production/                     # Production environment
|   |   |-- hosts.yml                   # Hosts and group definitions
|   |   |-- group_vars/
|   |   |   |-- all/
|   |   |   |   |-- main.yml            # Variables applied to every host
|   |   |   |   `-- vault.yml           # Encrypted secrets (ansible-vault)
|   |   |   |-- os_rhel/
|   |   |   |   `-- main.yml            # Variables for Rocky/Alma/RHEL hosts
|   |   |   |-- os_debian/
|   |   |   |   `-- main.yml            # Variables for Debian hosts
|   |   |   |-- type_lxc/
|   |   |   |   `-- main.yml            # Variables for LXC containers
|   |   |   |-- infra_networking/
|   |   |   |   `-- main.yml            # Variables for networking hosts
|   |   |   |-- infra_monitoring/
|   |   |   |   `-- main.yml            # Variables for monitoring hosts
|   |   |   `-- infra_hypervisors/
|   |   |       `-- main.yml            # Variables for Proxmox nodes
|   |   `-- host_vars/
|   |       |-- pve1/main.yml           # Variables specific to pve1
|   |       |-- pve2/main.yml           # Variables specific to pve2
|   |       |-- hl-dns-01/main.yml
|   |       `-- hl-mon-01/main.yml
|   |
|   `-- staging/                        # Staging environment (VM provisioning)
|       |-- hosts.yml
|       `-- group_vars/
|           `-- all/
|               `-- main.yml
|
|-- playbooks/
|   |-- site.yml                        # Master playbook, runs all roles on all hosts
|   |-- bootstrap-vm.yml               # Provisions a new VM from template
|   |-- prometheus-install.yml
|   `-- test-common.yml
|
|-- roles/
|   |-- common/                         # Baseline configuration for every managed host
|   |   |-- defaults/main.yml
|   |   |-- handlers/main.yml
|   |   `-- tasks/main.yml
|   `-- bootstrap/                      # VM provisioning (network + hostname)
|       |-- defaults/main.yml
|       |-- handlers/main.yml
|       `-- tasks/
|           |-- main.yml
|           |-- hostname.yml
|           `-- network.yml
|
`-- files/
    |-- ip-addresses-plan.md            # IP address planning document
    |-- machines-ip.md                  # Current machine IP reference
    |-- repo/                           # Binary assets and templates
    |-- services/                       # Systemd service unit files
    `-- ssh_keys/                       # Public keys deployed to managed hosts
```

---

## Inventory Design

The inventory uses three independent group axes. A single host can and should belong
to one group per axis simultaneously.

### Axis 1 - Technical (os_*, type_*)

Defines how Ansible interacts with the host. Group variables here control
package managers, firewall backends, SELinux state, and similar OS-level concerns.

| Group           | Description                         |
|-----------------|-------------------------------------|
| os_rhel         | Rocky Linux, AlmaLinux, RHEL        |
| os_debian       | Debian                              |
| os_ubuntu       | Ubuntu (when applicable)            |
| type_vm         | Proxmox virtual machines            |
| type_lxc        | Proxmox LXC containers              |
| type_baremetal  | Physical machines                   |

### Axis 2 - Functional (infra_*)

Defines what the machine does. Group variables here contain service-specific
configuration such as DNS zones, VPN peers, or monitoring targets.

| Group                | Description                    |
|----------------------|--------------------------------|
| infra_hypervisors    | Proxmox nodes                  |
| infra_orchestration  | Ansible controller             |
| infra_networking     | DNS, VPN, routing              |
| infra_monitoring     | Prometheus, Grafana, alerting  |

### Axis 3 - Environmental (env_*)

Used to scope playbook runs and apply environment-specific overrides.

| Group           | Description                         |
|-----------------|-------------------------------------|
| env_production  | All production hosts                |
| env_staging     | Temporary VM during provisioning    |

---

## Secrets Management

Secrets are stored in `inventory/production/group_vars/all/vault.yml`, encrypted
with ansible-vault. The file is never committed in cleartext.

Encrypt the vault:
```
ansible-vault encrypt inventory/production/group_vars/all/vault.yml
```

Edit the vault:
```
ansible-vault edit inventory/production/group_vars/all/vault.yml
```

Run a playbook with vault:
```
ansible-playbook playbooks/site.yml -i inventory/production --ask-vault-pass
```

Or use a vault password file (recommended for automation):
```
ansible-playbook playbooks/site.yml -i inventory/production --vault-password-file ~/.vault_pass
```

---

## Common Usage

### Run the full site playbook on production
```
ansible-playbook playbooks/site.yml -i inventory/production
```

### Target a specific OS group
```
ansible-playbook playbooks/site.yml -i inventory/production -l os_rhel
```

### Target a specific functional group
```
ansible-playbook playbooks/site.yml -i inventory/production -l infra_networking
```

### Provision a new VM from template
```
ansible-playbook playbooks/bootstrap-vm.yml -i inventory/staging \
  -e "target_ip=172.16.10.42 target_name=hl-dns-02"
```

### Dry run (check mode)
```
ansible-playbook playbooks/site.yml -i inventory/production --check --diff
```

### Inspect the inventory graph
```
ansible-inventory -i inventory/production --graph
ansible-inventory -i inventory/production --list
```

---

## Adding a New Host

1. Add the host under the relevant functional group in `inventory/production/hosts.yml`.
2. Add the host to the appropriate `os_*` and `type_*` groups.
3. Add the host to `env_production`.
4. If the host requires specific variables, create `inventory/production/host_vars/<hostname>/main.yml`.
5. Run a check: `ansible-playbook playbooks/site.yml -i inventory/production -l <hostname> --check`

---

## Adding a New Role

1. Create the role directory: `ansible-galaxy role init roles/<role_name>`
2. Add the role to the relevant play in `playbooks/site.yml`.
3. Document variables in `roles/<role_name>/defaults/main.yml`.

---

## Conventions

- All YAML files use the `.yml` extension.
- Group names follow the `axis_name` convention: `os_rhel`, `type_lxc`, `infra_networking`.
- Host names follow the `hl-<service>-<index>` convention: `hl-dns-01`, `hl-mon-01`.
- Secrets are prefixed with `vault_` when referenced in cleartext variable files.
- Every file includes a header block describing its purpose and version.
