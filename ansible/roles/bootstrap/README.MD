# Ansible Role: bootstrap

Provisions a freshly cloned Linux VM from a Proxmox template.
Creates the Ansible system user, hardens SSH, sets the hostname,
and reconfigures the network interface to its final IP address —
all in a single play, without rebooting.

Designed to be used as Play 1 in a two-play provisioning playbook,
with `add_host` handling the reconnection on the new IP for Play 2.

Tested on Rocky Linux 9.

---

## Requirements

- Ansible 2.12 or higher
- The following collections must be installed:

```
ansible-galaxy collection install community.general ansible.posix
```

- The target host must be reachable at its template IP before running.
- SSH public key files must be present at the paths defined in
  `bootstrap_key_admin` and `bootstrap_key_ansible`.

---

## Role Variables

### Required — no defaults

These must be passed at runtime via `-e`. The role will fail
with an explicit error message if either is missing.

| Variable | Example | Description |
|---|---|---|
| `target_ip` | `192.168.1.42` | Final IP address for the VM |
| `target_name` | `web-01` | Final hostname for the VM |

### Defined in `defaults/main.yml`

| Variable | Default | Description |
|---|---|---|
| `bootstrap_interface` | `ens18` | Network interface to reconfigure |
| `bootstrap_gateway` | `YOUR_GATEWAY_IP` | Default gateway |
| `bootstrap_prefix` | `24` | Network prefix length (CIDR) |
| `bootstrap_dns` | `[YOUR_DNS_IP]` | DNS server(s) applied during provisioning |
| `bootstrap_ansible_user` | `ansible` | System user created for Ansible |
| `bootstrap_ansible_uid` | `900` | UID for the Ansible system user |
| `bootstrap_ansible_shell` | `/bin/bash` | Shell for the Ansible system user |
| `bootstrap_ansible_comment` | `Ansible automation user` | GECOS comment |
| `bootstrap_admin_user` | `admin` | Existing user to receive the admin SSH key |
| `bootstrap_key_admin` | `files/ssh_keys/homelab_admin.pub` | Path to admin public key |
| `bootstrap_key_ansible` | `files/ssh_keys/ansible-key.pub` | Path to Ansible public key |

### Expected from `group_vars`

| Variable | Expected in | Description |
|---|---|---|
| `internal_domain` | `group_vars/all/main.yml` | Domain used in `/etc/hosts` entry |

---

## Dependencies

None.

---

## File Structure

```
roles/bootstrap/
  defaults/main.yml       Variables and their default values
  tasks/
    main.yml              Entry point, validates required vars, calls sub-tasks
    user.yml              Creates ansible system user and sudoers file
    ssh.yml               Deploys SSH keys, disables password authentication
    hostname.yml          Sets hostname via hostnamectl (no reboot required)
    network.yml           Reconfigures network interface via nmcli
  handlers/
    main.yml              Reload sshd, reload nmcli connection
```

---

## Design Notes

### No reboot required

Hostname changes via `hostnamectl` take effect immediately.
Network changes via `nmcli` apply without rebooting.
The connection drop after the IP change is expected and handled
by `add_host` in the calling playbook.

### Task ordering

The sub-task files must run in this order:

1. `user.yml` — the ansible user must exist before its SSH key is deployed
2. `ssh.yml` — SSH must be hardened before the network change cuts the connection
3. `hostname.yml` — no hard dependency, logical to set before network
4. `network.yml` — always last, drops the connection immediately

### Sudoers validation

The sudoers file is written with `validate: /usr/sbin/visudo --check --file=%s`.
If the syntax is invalid, the task fails before writing — sudo access
on the host is never broken.

---

## Example Playbook

```yaml
- name: Play 1 - Provision VM from template IP
  hosts: staging_vm
  become: true
  gather_facts: false

  roles:
    - bootstrap

  post_tasks:
    - name: Register new host for Play 2
      ansible.builtin.add_host:
        name:         "{{ target_name }}"
        ansible_host: "{{ target_ip }}"
        ansible_user: "{{ bootstrap_ansible_user }}"
        groups:       newly_provisioned
      changed_when: false

- name: Play 2 - Post-provisioning on final IP
  hosts: newly_provisioned
  become: true
  gather_facts: true

  roles:
    - common
```

Run:
```
ansible-playbook playbooks/bootstrap-vm.yml \
  -i inventory/staging \
  -e "target_ip=YOUR_TARGET_IP target_name=your-hostname"
```


