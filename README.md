# Homelab Platform

A personal infrastructure platform built for fun and learning. 

## Philosophy

**Learn by doing, have fun building.** This homelab exists because playing with technology is enjoyable. Every tool deployed solves a real problem while teaching production-grade skills. A happy IT guy is an effective IT guy.

## Project Roadmap

### Step 1: Foundation layer
*Goal: Build a solid, reproducible base for everything else*

**Challenge 1.1: Manual VM provisioning is tedious**
- Solution: Ansible playbooks for automated VM setup
- Status: done
- Progression: 
    - Ansible control node deployed (hl-orch-01). 
    - Inventory done. 
    - SSH key-based authentication established. 

**Challenge 1.2: No consistent base configuration**
- Solution: Ansible roles (common, docker-host, monitoring-agent)
- Status: Planned

**Challenge 1.3: Time drift across VMs**
- Solution: Centralized NTP server (chrony)
- Status: Done
- Details: hl-orch-01 serving time to infrastructure (CH upstream pools)

**Challenge 1.4: VPN access to the bastion**
- Solution: Install and configure Netbird
- Status: Planned
- Details: None

### Step 2: Observability 
*Goal: See what's happening in the infrastructure*

**Challenge 2.1: No visibility into system health**
- Solution: Prometheus + Grafana monitoring stack
- Status: Planned
- Repo: Inc.

**Challenge 2.2: Logs scattered everywhere**
- Solution: Loki for centralized logging
- Status: Planned

**Challenge 2.3: No alerting when things break**
- Solution: Alertmanager with smart alerts
- Status: Planned

### Step 3: Service Platform 
*Goal: Deploy useful services easily*

**Challenge 3.1: Hard to remember IPs and ports**
- Solution: Pi-hole DNS + Traefik reverse proxy
- Status: Planned

**Challenge 3.2: Manual SSL certificate management**
- Solution: Traefik with Let's Encrypt automation
- Status: Planned

**Challenge 3.3: Family needs cloud storage**
- Solution: Nextcloud deployment
- Status: Planned
- Repo: Inc.

### Step 4: Automation & GitOps
*Goal: Git as the source of truth*

**Challenge 4.1: Configuration drift**
- Solution: GitOps workflow with Ansible
- Status: Planned

**Challenge 4.2: Manual deployments**
- Solution: CI/CD pipeline with GitLab CE
- Status: Planned

**Challenge 4.3: No rollback strategy**
- Solution: Versioned configs, automated rollback
- Status: Planned

### Step 5: Advanced Topics 🎓
*Goal: Learn complex concepts hands-on*

**TBD** - Will evolve based on learning interests (K8s, service mesh, advanced networking)

### Future Projects ideas (TBD)
Ideas collected during tech watch sessions. No priority, no commitment — just inspiration for when the foundation is solid.

**Security**
- CrowdSec — Community-driven Fail2ban alternative with shared threat intelligence. Integrates natively with Traefik as a bouncer. Relevant once services are exposed externally.
- Wazuh — Security monitoring and SIEM (already in initial roadmap)
- Caddy — Reverse proxy with automatic HTTPS (built-in ACME/DNS-01). Simpler config than Traefik, native CrowdSec integration. Evaluate as potential Certbot + reverse proxy replacement at Step 3.

**Communication & Collaboration**
- Matrix (Synapse/Dendrite) + Element — Self-hosted, federated, E2E encrypted messaging. Potential Discord/WhatsApp replacement for family use. Protocol = Matrix, client = Element, server = Synapse/Dendrite.

**Tech Watch & Information**
- FreshRSS / Miniflux — Self-hosted RSS aggregator. Use cases: automated CVE feeds filtered by stack (Rocky, Proxmox, GitLab, Docker), tech news, homelab discoveries.
- RSS → Notification pipeline — CVE feed → filter by deployed tech → push notification (mail, Matrix, etc.)

**Media & Storage**
- Jellyfin — Self-hosted open-source media server (Netflix-like). Runs in Docker, points to network storage.
- Centralized NAS — Confirmed need for media storage (capacity, reliability, throughput). Two paths to evaluate:
  1. Commercial NAS / Synology (KISS, plug & play)
  2. TrueNAS DIY (learning opportunity, more control, lower cost)

**External Access**
- WireGuard VPN — Secure remote access without exposing services
- Cloudflare Proxy / DDoS Protection — For when services go public

---

This repository contains the foundation layer automation:
```
homelab-platform/
├── ansible/
│   ├── playbooks/          # VM provisioning and configuration
│   ├── roles/              # Reusable roles (common, docker-host, etc.)
│   └── inventory/          # Host definitions
├── docs/
│   ├── infra.md            # infrastructure design
│   └── procedures/         # Runbooks and how-tos
└── scripts/                # Utility scripts
```

## Contributing

This is a personal learning project, but **tips, advice, and suggestions are always welcome!** 

Feel free to open an issue if you spot something that could be improved or have ideas for new challenges to tackle.

## License
MIT
