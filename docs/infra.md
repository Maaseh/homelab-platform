```
%%{init: {'theme':'base', 'themeVariables': {'fontSize':'16px'}}}%%
graph TD
    subgraph Internet_Zone["☁️ Internet"]
        Internet[Internet]
    end
    
    subgraph Intranet["🏠 Local Infrastructure"]
        Orange[ISP]
        UDM[UDM]
        Switch[USW]
        
        subgraph Management["Management Subnet x.x.10.0/24"]
            subgraph Cluster["Proxmox Cluster - HA Enabled"]
                PVE1[⭐ pve1 - Optiplex 1<br/>Master Node<br/>x.x.10.x]
                PVE2[pve2 - Optiplex 2<br/>Replica Node<br/>x.x.10.x]
            end
            Tuxedo[🔑 Local bastion<br/>x.x.10.x]
        end
        
        subgraph WiFi["WiFi Infrastructure"]
            U7E[U7 1 - AP WiFi]
            U7S[U7 2 - AP WiFi]
            U7G[U7 3 - AP WiFi]
        end
        
        Services1[⚙️ Services HA Cluster<br/><br/>🐋 Docker hosts<br/>📊 Monitoring - planned<br/>🌐 Web services - planned<br/><br/>Auto-failover enabled]
    end
    
    Internet --> Orange
    Orange --> UDM
    UDM --> Switch
    
    Switch --> PVE1
    Switch --> PVE2
    Switch --> Tuxedo
    Switch --> U7E
    Switch --> U7S
    Switch --> U7G
    
    PVE1 <-.Corosync/HA.-> PVE2
    
    Tuxedo -.SSH/443 cert only.-> PVE1
    Tuxedo -.SSH/443 cert only.-> PVE2
    
    PVE1 --> Services1
    PVE2 --> Services1
    
    Switch -.access.-> Services1
    
    style Internet fill:#87ceeb,stroke:#4682b4,stroke-width:3px
    style Orange fill:#ff9500
    style UDM fill:#0066cc
    style Switch fill:#0066cc
    style Tuxedo fill:#28a745,stroke:#155724,stroke-width:3px
    style PVE1 fill:#dc3545,stroke:#721c24,stroke-width:3px
    style PVE2 fill:#6c757d
    style Services1 fill:#ffc107
    style U7E fill:#00a8e8
    style U7S fill:#00a8e8
    style U7G fill:#00a8e8
    style Management fill:#f0f0f0,stroke:#333,stroke-dasharray: 5 5
    style Cluster fill:#ffe0e0,stroke:#dc3545,stroke-dasharray: 5 5
```
