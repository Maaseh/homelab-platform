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
        - Note: Even if this configuration was initially incorrect, this was not the root cause of the initial issue.

## Decision


## Conclusion
