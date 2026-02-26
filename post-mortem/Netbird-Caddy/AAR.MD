# After Action Review on the implementation of Caddy for Netbird service
Author: Thomas "Maaseh" Bonnet
---

## Context

During Challenge 1.5, I decided to implement an open source, zero-Trust network management platform solution for peer-to-peer mesh VPN. Using technologies like Wireguard, WebRTC, and Coturn; This solution seems to be a perfect fit for my project, as well as it's free for individual use. This challenge requires to implement several services, as a reverse proxy, a IAM, and we also want to use Cloudflare to add an extra security layer.

## Timeline

### Infrastructure implementation
- **Creation of a VM inside Proxmox:**
    - Name: hl-vpn-01
    - CPU: 2 cores
    - RAM: 4 Go
    - Network: 172.16.10.3 
    - 
- **Automation with Ansible:**
    - bootstrap playbook == **OK**
    - Rocky-linux playbook == **OK**
- **DNS configuration:**
    - adding entry for hl-vpn-01
- **Adding extra packages:**
    - Docker
### Applications installation
- **Netbird installation:**
    - reading the [official documentation](https://docs.netbird.io/selfhosted/selfhosted-quickstart) to implement netbird through docker
    - Creating a dedicated docker folder under /opt/docker/netbird
    - Initializing the installation under the netbird folder:
        - Domain: netbird.vansys.io
        - Proxy: Using external Caddy (option 4)
    - Using ```docker compose up -d``` to verify that netbird-server and netbird-dashbord run with no issue
- **Configuring CloudFlare to act as "public shield"**
  - configuring Cloudflare tunnel
  - adding a published application route inside netbird tunnel
      - Hostname:
          - subdomain: netbird
          - domain: vansys.io
      - Service:
          - type: HTTP
          - URL: caddy:80
  - creation .env to store the secret key and chmod 600 the file
  - Configuring cloudflared inside docker-compose
    ```
      # Cloudflared
      cloudflare-tunnel:
        image: cloudflare/cloudflared:latest
        container_name: cloudflare_tunnel
        command:  tunnel --no-autoupdate run --token ${TOKEN_CLOUDFLARE}
        restart: unless-stopped
        networks: [netbird]
        logging:
          driver: "json-file"
          options:
            max-size: "500m"
            max-file: "2"
    ```
  - Use docker compose up -d, checking if the container runs without issue and checking the tunnel health on CloudFlare website
- **Caddy installation and configuration**
    - Adding a conf folder inside netbird path
    - touch Caddyfile
    - Reading the [Caddy documentation](https://caddyserver.com/docs/caddyfile)
    - modifying the caddyfile-netbird.txt to meet the requirements and copy it to the Caddyfile
        ```
        :80 {
            # Native gRPC (needs HTTP/2 cleartext to backend)
            @grpc header Content-Type application/grpc*
            reverse_proxy @grpc h2c://netbird-server:80
        
            # Combined server paths (relay, signal, management, OAuth2)
            @backend path /relay* /ws-proxy/* /api/* /oauth2/*
            reverse_proxy @backend netbird-server:80
        
            # Dashboard (everything else)
            reverse_proxy /* dashboard:80
        }
        ```
    - creating the caddy service inside docker-compose
        ```
          # Caddy
          caddy:
            image: caddy:latest
            restart: unless-stopped
            networks: [netbird]
            volumes:
              - ./conf:/etc/caddy
              - caddy_data:/data
              - caddy_config:/config
        
        volumes:
          netbird_data:
          caddy_data:
          caddy_config:
        ```
    - testing the configuration to ensure the container runs well
### Application configuration
- **connection to netbird.vansys.io**
  - first connexion == **OK**
  - Admin account creation == **OK**
  - Endpoint configuration == **NOK**
  ```
    netbird-server  | 2026-02-24T21:54:39.282Z ERRO [context: HTTP, requestID: {ID}] management/server/store/sql_store.go:1106: error when getting account {account} from the store: record not found
  ```

## Investigation
As I am unable to connect any endpoint to Netbird:
  - Netbird endpoint configuration on Macbook == **NOK**
  - Netbird endpoint configuration on bastion == **NOK**
- checking netbird-server docker logs:
  ```
  netbird-server  | 2026-02-24T21:52:10.357Z INFO combined/cmd/root.go:221: Relay server created
  netbird-server  | 2026-02-24T21:52:10.357Z INFO combined/cmd/root.go:272: Management server created
  netbird-server  | 2026-02-24T21:52:10.357Z INFO combined/cmd/root.go:288: Signal server created
  netbird-server  | 2026-02-24T21:52:10.366Z INFO management/server/migration/migration.go:263: No plain setup keys found in table setup_keys, no migration needed
  netbird-server  | 2026-02-24T21:52:10.366Z INFO management/server/migration/migration.go:301: Migration of plain setup key to hashed setup key completed
  netbird-server  | 2026-02-24T21:52:10.366Z INFO management/server/migration/migration.go:344: No rows with empty name found in table users, no migration needed
  netbird-server  | 2026-02-24T21:52:10.367Z INFO management/server/migration/migration.go:358: Migration of empty name to default value in table users completed
  netbird-server  | 2026-02-24T21:52:10.367Z INFO management/server/migration/migration.go:344: No rows with empty email found in table users, no migration needed
  netbird-server  | 2026-02-24T21:52:10.367Z INFO management/server/migration/migration.go:358: Migration of empty email to default value in table users completed
  netbird-server  | 2026-02-24T21:52:10.385Z INFO management/server/migration/migration.go:399: index idx_account_ip already exists on table peers
  netbird-server  | 2026-02-24T21:52:10.385Z INFO management/server/migration/migration.go:399: index idx_account_dnslabel already exists on table peers
  netbird-server  | 2026-02-24T21:52:10.579Z INFO management/server/account_request_buffer.go:45: set account request buffer interval to 100ms
  netbird-server  | 2026-02-24T21:52:10.579Z INFO management/server/migration/migration.go:344: No rows with empty name found in table deleted_users, no migration needed
  netbird-server  | 2026-02-24T21:52:10.579Z INFO management/server/migration/migration.go:358: Migration of empty name to default value in table deleted_users completed
  netbird-server  | 2026-02-24T21:52:10.579Z INFO management/server/migration/migration.go:344: No rows with empty enc_algo found in table deleted_users, no migration needed
  netbird-server  | 2026-02-24T21:52:10.579Z INFO management/server/migration/migration.go:358: Migration of empty enc_algo to default value in table deleted_users completed
  netbird-server  | 2026-02-24T21:52:10.581Z INFO management/server/account_request_buffer.go:45: set account request buffer interval to 100ms
  **netbird-server  | 2026-02-24T21:52:10.581Z INFO management/server/account.go:245: single account mode enabled, accounts number 0**
  netbird-server  | 2026-02-24T21:52:35.968Z INFO [context: HTTP, requestID: d6f1r8sr5s9s73fuiod0] management/server/http/handlers/instance/instance_handler.go:49: instance setup status: true
  netbird-server  | 2026-02-24T21:54:33.791Z INFO [context: HTTP, requestID: d6f1s6cr5s9s73fuiodg] management/server/instance/manager.go:171: created owner user maaseh@tuta.com in embedded IdP
  netbird-server  | 2026-02-24T21:54:33.791Z INFO [context: HTTP, requestID: d6f1s6cr5s9s73fuiodg] management/server/http/handlers/instance/instance_handler.go:70: instance setup completed: created user maaseh@tuta.com
  netbird-server  | 2026-02-24T21:54:36.295Z INFO [context: HTTP, requestID: d6f1s74r5s9s73fuioe0] management/server/http/handlers/instance/instance_handler.go:49: instance setup status: false
  netbird-server  | 2026-02-24T21:54:38.583Z INFO [context: HTTP, requestID: d6f1s7kr5s9s73fuioeg] management/server/http/handlers/instance/instance_handler.go:49: instance setup status: false
  netbird-server  | 2026-02-24T21:54:39.125Z INFO [context: HTTP, requestID: d6f1s7sr5s9s73fuiof0] management/server/http/handlers/instance/instance_handler.go:49: instance setup status: false
  netbird-server  | 2026-02-24T21:54:39.282Z ERRO [context: HTTP, requestID: d6f1s7sr5s9s73fuiofg] management/server/store/sql_store.go:1106: error when getting account d6f1s7sr5s9s73fuiog0 from the store: record not found
  ```
- First question: Is Caddy is trusted by netbird
  - By looking on the config.yaml, the trustedHTTPProxies doesn't fit the Caddy's network configuration:
    - Fix:
      - ```docker inspect netbird-caddy-1 | grep IPAddress``` . This command helped me to find the IP address of Caddy
      - To avoid the caddy IP Address to change, I modified the docker-compose file
        ```
            #Caddy config:
              networks:
                netbird:
                  ipv4_address: 172.18.0.10
    
            # Network config:
              networks:
                netbird:
                  ipam:
                    config:
                      - subnet: 172.18.0.0/24
        ```
        - Note: Even if this configuration was initially incorrect, this was not the root cause of the initial issue.\

- **Second question: Is Caddy able to handle gRPC traffic?**
  - Caddy logs showed: `HTTP/2 skipped because it requires TLS`
  - gRPC requires HTTP/2. Since Caddy listens on `:80` (no TLS, as TLS is terminated by Cloudflare), HTTP/2 was disabled by default
  - Fix: Added global options to enable h2c (HTTP/2 cleartext)
```
    {
        servers {
            protocols h1 h2c
        }
    }
```
  - Result: Warning disappeared, but endpoint connection still failing

- **Third question: Is the embedded IdP functional?**
  - Tested OIDC endpoint from the VM:
```
    curl -s https://netbird.vansys.io/oauth2/.well-known/openid-configuration
```
  - Result: valid JSON response with correct issuer, endpoints, and supported scopes == **OK**
  - The embedded IdP serves correctly through the full chain (Cloudflare → tunnel → cloudflared → Caddy → netbird-server)

- **Fourth question: Can clients authenticate via gRPC?**
  - Client error persisted after all fixes:
```
    retrying Login to the Management service due to error rpc error: code = Unknown desc = getting device authorization flow info failed with error: failed while getting Management Service public key
```
  - After wiping volumes (`docker compose down -v`) and recreating the stack, new error appeared:
```
    unable to find appropriate key
    token could not be parsed: token is unverifiable
```
  - Dashboard login also returned: `token invalid` (HTTP 401)

- **Fifth question: Is this a known issue?**
  - Research on NetBird GitHub issues revealed multiple reports of the same error when using Cloudflare Tunnel
  - Root cause: **Cloudflare Tunnel does not correctly forward gRPC trailers**, which are required by the NetBird management service for authentication key exchange
  - This is a Cloudflare limitation, not a configuration issue
  - Confirmed by community: even with direct routing of `/management.ManagementService/` through the tunnel, trailers are stripped

- **Sixth question: Same issue with Traefik?**
  - Reconfigured the stack using Traefik (natively supported by NetBird) instead of Caddy
  - Traefik failed to obtain TLS certificate:
```
    unable to generate a certificate for the domains [netbird.vansys.io]: error: 403 :: Cannot negotiate ALPN protocol "acme-tls/1" for tls-alpn-01 challenge
```
  - Cause: Cloudflare Tunnel intercepts TLS, preventing Let's Encrypt TLS-ALPN-01 challenge from reaching Traefik
  - This confirmed that Cloudflare Tunnel is incompatible with the default NetBird self-hosted setup regardless of the reverse proxy used

## Decision

Based on the investigation, Cloudflare Tunnel is incompatible with NetBird self-hosted due to:
1. gRPC trailers being stripped by Cloudflare, breaking management service authentication
2. TLS-ALPN-01 challenge being blocked, preventing automatic certificate provisioning

**Decision: Remove Cloudflare Tunnel from the NetBird architecture.**

New architecture:
- `netbird.vansys.io` → A record (grey cloud, proxy OFF) → UDM port forward 443/TCP → Caddy (automatic TLS via Let's Encrypt) → NetBird services
- `turn.vansys.io` → A record (grey cloud, proxy OFF) → UDM port forward UDP 3478, 5349, 49152-49162 → coturn
- Cloudflare remains as DNS registrar only for NetBird-related subdomains
- Caddy chosen over Traefik for simplicity (KISS principle), as the NetBird Reverse Proxy feature (requiring Traefik) is still in beta

Trade-off accepted: opening port 443/TCP on the UDM increases the attack surface compared to the tunnel approach, but this is acceptable for a homelab context with a single self-hosted VPN service.

## Conclusion

### What worked
- VM provisioning and Ansible automation
- Cloudflare Tunnel setup and connectivity (dashboard was reachable)
- Docker Compose stack unification (all services on the same bridge network)
- Caddy path-based routing configuration for multi-service architecture
- OIDC endpoint functional through the full chain

### What didn't work
- Cloudflare Tunnel + NetBird gRPC authentication (fundamental incompatibility)
- Cloudflare Tunnel + Let's Encrypt TLS-ALPN-01 challenge (TLS interception)

### Lessons learned
1. **Always verify tool compatibility before designing the architecture.** Cloudflare Tunnel was chosen for its security benefits, but its limitations with gRPC were not identified upfront
2. **Follow the vendor's recommended setup first, then customize.** NetBird's default configuration works without Cloudflare Tunnel — adding it introduced unnecessary complexity
3. **Separate concerns when debugging.** Testing each layer independently (DNS, tunnel, reverse proxy, authentication) was essential to isolate the root cause
4. **Document failures.** This AAR exists because the failure taught more than a successful deployment would have

### Next steps
- Reconfigure NetBird with Caddy (no Cloudflare Tunnel)
- Open port 443/TCP on UDM with forwarding to hl-vpn-01
- Validate automatic TLS provisioning via Let's Encrypt
- Test endpoint connectivity from external network
- Update CLAUDE.md to reflect the new architecture
- Update README Challenge 1.5 details
