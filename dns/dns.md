# Bind9 DNS Configuration — hl-dns-01

## Overview

Internal DNS server for `your.dns.domain.lab` running Bind9 on a Rocky Linux LXC container (`hl-dns-01` — `172.16.10.2`).

Provides forward and reverse resolution for the homelab, with Cloudflare (`1.1.1.1` / `1.0.0.1`) as forwarders for external queries.

---

## Configuration Files

### `/etc/named.conf`
```conf
acl trusted-network {
    127.0.0.1; ::1;
};

options {
#       listen-on port 53 { 127.0.0.1; };
#       listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        secroots-file   "/var/named/data/named.secroots";
        recursing-file  "/var/named/data/named.recursing";
        allow-query     { trusted-network; };
        allow-recursion { trusted-network; };
        allow-transfer  { none; };
        forwarders      { 1.1.1.1; 1.0.0.1; };

        recursion yes;
        dnssec-validation yes;

        managed-keys-directory "/var/named/dynamic";
        geoip-directory "/usr/share/GeoIP";
        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";

        include "/etc/crypto-policies/back-ends/bind.config";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
        type hint;
        file "named.ca";
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";

// Forward zone
zone "your.dns.domain.lab" IN {
        type master;
        file "your.dns.domain.lab.db";
        allow-update { none; };
};

// Reverse zone
zone "10.16.172.in-addr.arpa" IN {
        type master;
        file "your.dns.domain.lab.rev";
        allow-update { none; };
};
```

### `/var/named/your.dns.domain.lab.db`
```dns
$TTL 86400
@ IN SOA hl-dns-01.your.dns.domain.lab. admin.your.dns.domain.lab. (
    2026021601 ;Serial
    3600       ;Refresh
    1800       ;Retry
    604800     ;Expire
    86400      ;Minimum TTL
)

;Name Server Information
@ IN NS hl-dns-01.your.dns.domain.lab.

;IP for Name Server
hl-dns-01 IN A 172.16.10.2

;A Records
pve1       IN A 172.16.10.10
pve2       IN A 172.16.10.11
hl-orch-01 IN A 172.16.10.30
```

### `/var/named/your.dns.domain.lab.rev`
```dns
$TTL 86400
@ IN SOA hl-dns-01.your.dns.domain.lab. admin.your.dns.domain.lab. (
    2026021601 ;Serial
    3600       ;Refresh
    1800       ;Retry
    604800     ;Expire
    86400      ;Minimum TTL
)

;Name Server Information
@ IN NS hl-dns-01.your.dns.domain.lab.

;Reverse lookup for Name Server
2  IN PTR hl-dns-01.your.dns.domain.lab.

;PTR Records
10 IN PTR pve1.your.dns.domain.lab.
11 IN PTR pve2.your.dns.domain.lab.
30 IN PTR lab-orch-01.your.dns.domain.lab.
```

---

## File Placement

| File | Path | Owner | Description |
|------|------|-------|-------------|
| `named.conf` | `/etc/named.conf` | `root:named` | Main Bind9 configuration |
| `yourdomain.lab.io` | `/var/named/your.dns.domain.lab` | `named:named` | Forward zone file |
| `yourdomain.lab.rev` | `/var/named/your.dns.domain.lab.rev` | `named:named` | Reverse zone file |

## Maintenance

When adding/removing hosts:

1. Edit both zone files (forward AND reverse)
2. Increment the serial number (format: `YYYYMMDDNN`)
3. Verify syntax: `named-checkconf -z /etc/named.conf`
4. Reload: `rndc reload`

## Client Configuration

- **DNS server itself** (`hl-dns-01`): `resolv.conf` → `nameserver 127.0.0.1`
- **All other lab machines**: `resolv.conf` → `nameserver 172.16.10.2`
