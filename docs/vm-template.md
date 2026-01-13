# VM template information
##### V0.1
> KISS version. More improvement and a cloud-init upgrade will come later.

## Purpose:

This document relates all useful information about the VM Template created on Proxmox.
The idea is to have a clear VM base to simply and easily create more virtual machines.
As soon as  it's created, Ansible will run to configure the VM based on the template and the purpose.

## Configuration

### Hardware

**General:**
- VM ID: `9000`
- Name: `rocky10-template`

**OS:**
- ISO: `Rocky-10.1-x86_64-minimal.iso`

**System:**
- Machine: `q35`
- BIOS: `SeaBIOS`
- SCSI Controller: `VirtIO SCSI`

**Disks:**
- Size: `32 GB`
- Storage: `local-lvm`

**CPU:**
- Type: `host`
- Cores: `2`

**Memory:**
- RAM: `2048 MB`

**Network:**
- Model: `VirtIO`


> **Note:** CPU type `host` required to avoid kernel panic on Rocky/Alma Linux.
> 
> See: [Proxmox Forum Thread](https://forum.proxmox.com/threads/kernel-panic-centos-9.104656/)

### Software

#### Installation
```
#!/bin/bash

sudo dnf update -y

sudo dnf install -y \
    cloud-init \
    qemu-guest-agent \
    vim \
    wget \
    curl \
    net-tools

sudo systemctl enable --now qemu-guest-agent
```

#### Cleaning

```
#!/bin/bash
# Remove SSH host keys
sudo rm -f /etc/ssh/ssh_host_*

# Remove machine-id
sudo truncate -s 0 /etc/machine-id
sudo rm -f /var/lib/dbus/machine-id

# Clean logs
sudo truncate -s 0 /var/log/wtmp
sudo truncate -s 0 /var/log/lastlog
sudo find /var/log -type f -name "*.log" -exec truncate -s 0 {} \;

# Clean bash history
history -c
cat /dev/null > ~/.bash_history
sudo cat /dev/null > /root/.bash_history

# Clean dnf cache
sudo dnf clean all

```
### Network
Network is configured as following:
```
IP address: x.x.10.250
Netmask: 255.255.255.0
Gateway: x.x.10.1
DNS: 8.8.8.8;1.1.1.1
```

**Staging IP Strategy:**  
Cloned VMs start with a temporary staging IP (`x.x.10.250`) outside production ranges. Ansible then reconfigures each VM to its final IP address based on its role. This ensures no IP conflicts during the provisioning process.

### User
Create a temporary user that can be use with Ansible for the first time. This will be deleted once another end-user will be created.


## Specific configuration:

### Rocky Linux

```
#!/bin/bash

#Check SELinux configuration
sudo getenforce

# if not in enforcing mode, do:
sudo setenforce 1

sudo systemctl status firewalld

# If not enabled, do:
sudo systemctl enable --now firewalld

# Check general firewalld configuration
sudo firewall-cmd --list-all

# If no SSH, do:
sudo firewall-cmd --add-service=ssh --permanent
sudo firewall-cmd --reload
```
