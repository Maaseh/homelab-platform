# IP Address Plan

Management Subnet: x.x.10.0/24

## Range Allocation

### .1-.9 - Network Infrastructure
- .1 = Gateway (UDM)

### .10-.19 - Proxmox Hypervisors
- .11 = pve1
- .12 = pve2

### .20-.29 - Bastion & Security
- .20 = Tuxedo (bastion host)

### .30-.49 - Core Infrastructure (Orchestration, NTP, Internal DNS)
- .30 = hl-orch-01 (orchestrator)
- .31 = hl-dns-01 (future Pi-hole)
- .32-.49 = reserved for infrastructure services

### .50-.79 - Monitoring & Observability
- .50 = hl-mon-01 (future Prometheus/Grafana)
- .51 = hl-log-01 (future Loki)

### .100-.149 - Kubernetes/Containers
- .100 = k3s-master
- .101 = k3s-worker-01
- .102-.149 = K3s scaling capacity

### .150-.199 - Standalone Docker Hosts
- .150 = hl-docker-01

### .200-.249 - Application VMs
- .200 = hl-web-01 (Nextcloud, etc.)
