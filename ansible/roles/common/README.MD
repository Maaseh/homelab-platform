# Ansible Role: common

Applies a baseline configuration to every managed Linux host.
Covers timezone, NTP (chrony), and base package installation.

Designed to be OS-agnostic through group variables — no
`ansible_facts` OS checks inside the role.

Tested on Rocky Linux 9 and Debian 12.

---

## Requirements

- Ansible 2.12 or higher
- The following collection must be installed:

```
ansible-galaxy collection install community.general
```

---

## Role Variables

### Defined in `defaults/main.yml`

| Variable | Default | Description |
|---|---|---|
| `system_timezone` | `Europe/Paris` | System timezone (tzdata format) |
| `ssh_permit_root_login` | `no` | SSH root login policy |
| `ssh_password_auth` | `no` | SSH password authentication policy |

### Expected from `group_vars`

These variables have no defaults and must be provided
via `group_vars` or `host_vars`.

| Variable | Expected in | Description |
|---|---|---|
| `ntp_local_server` | `group_vars/all/main.yml` | Internal NTP server IP or hostname |
| `pkg_manager` | `group_vars/os_*/main.yml` | Package manager: `dnf` or `apt` |
| `base_packages` | `group_vars/os_*/main.yml` | List of packages to install on every host |

### Recommended `group_vars` layout

```yaml
# group_vars/all/main.yml
ntp_local_server: 192.168.1.10

# group_vars/os_rhel/main.yml
pkg_manager: dnf
base_packages:
  - vim-enhanced
  - curl
  - git
  - bash-completion

# group_vars/os_debian/main.yml
pkg_manager: apt
base_packages:
  - vim
  - curl
  - git
  - bash-completion
```

---

## Dependencies

None.

---

## File Structure

```
roles/common/
  defaults/main.yml       Variables and their default values
  tasks/
    main.yml              Entry point, calls sub-task files in order
    timezone.yml          System timezone configuration
    ntp.yml               Chrony install and configuration
    packages.yml          Package cache update and base package install
  handlers/
    main.yml              Restart chrony on config change
```

---

## Design Notes

### Variable-driven OS behavior

The role contains no `when: ansible_facts['os_family']` conditions.
OS-specific behavior is controlled entirely through `group_vars`.

The `pkg_manager` variable drives which cache update module is used.
Adding support for a new OS requires only a new `group_vars` file —
the role itself does not change.

### Chrony config path handling

The chrony configuration file path differs across distributions:
- RHEL/Rocky: `/etc/chrony.conf`
- Debian: `/etc/chrony/chrony.conf`

The `first_found` lookup handles this transparently without
duplicating tasks or adding conditionals.

---

## Example Playbook

```yaml
- name: Apply baseline to all managed hosts
  hosts: all
  become: true

  roles:
    - common
```

With variable overrides:

```yaml
- name: Apply baseline with custom timezone
  hosts: all
  become: true

  roles:
    - role: common
      vars:
        system_timezone: America/New_York
        ntp_local_server: 10.0.0.1
```
