# Domain 4: Networking (15% of Exam)

[← Back to Index](./README.md) · [Previous: Installation & Configuration](./05-installation-configuration.md)

---

## Container Network Model (CNM)

```mermaid
graph TB
    subgraph "Container Network Model"
        SB["Sandbox<br/>Network namespace<br/>for a container"]
        EP["Endpoint<br/>Connects sandbox<br/>to network"]
        NW["Network<br/>Group of endpoints<br/>that communicate"]
    end

    SB --- EP --- NW

    style SB fill:#e74c3c,color:#fff
    style EP fill:#f39c12,color:#fff
    style NW fill:#3498db,color:#fff
```

| CNM Component | Description |
|---------------|-------------|
| **Sandbox** | Isolated network namespace for a container (interfaces, routes, DNS) |
| **Endpoint** | Virtual network interface connecting a sandbox to a network |
| **Network** | A group of endpoints that can communicate directly |

---

## Network Drivers

```mermaid
graph TB
    subgraph "Docker Network Drivers"
        BR["bridge (default)<br/>━━━━━━━━━━━━━━<br/>Isolated network on single host<br/>Containers communicate via bridge<br/>Default for standalone containers"]

        HOST["host<br/>━━━━━━━━━━━━━━<br/>No network isolation<br/>Container uses host network<br/>Best performance, no port mapping"]

        NONE["none<br/>━━━━━━━━━━━━━━<br/>No networking at all<br/>Only loopback interface<br/>Complete isolation"]

        OV["overlay<br/>━━━━━━━━━━━━━━<br/>Multi-host networking<br/>Swarm service communication<br/>Uses VXLAN encapsulation"]

        MAC["macvlan<br/>━━━━━━━━━━━━━━<br/>MAC address per container<br/>Appears as physical device<br/>Direct physical network access"]
    end

    style BR fill:#3498db,color:#fff
    style HOST fill:#e74c3c,color:#fff
    style NONE fill:#7f8c8d,color:#fff
    style OV fill:#9b59b6,color:#fff
    style MAC fill:#27ae60,color:#fff
```

| Driver | Scope | Use Case |
|--------|-------|----------|
| **bridge** | Single host | Default; standalone containers on same host |
| **host** | Single host | Maximum performance; no NAT overhead |
| **none** | Single host | Complete network isolation |
| **overlay** | Multi-host | Swarm services across nodes |
| **macvlan** | Single host | Containers need to appear on physical network |

---

## Bridge Network

```mermaid
graph TB
    subgraph "Docker Host"
        subgraph "docker0 (default bridge)"
            C1["Container 1<br/>172.17.0.2"]
            C2["Container 2<br/>172.17.0.3"]
        end
        subgraph "my-bridge (user-defined)"
            C3["Container 3<br/>172.18.0.2<br/>name: web"]
            C4["Container 4<br/>172.18.0.3<br/>name: api"]
        end
        ETH["eth0 (Host NIC)<br/>192.168.1.100"]
    end

    C1 <-->|"IP only"| C2
    C3 <-->|"IP + DNS name ✅"| C4
    C1 -.->|"❌ Isolated"| C3

    style ETH fill:#e74c3c,color:#fff
```

### Default Bridge vs User-Defined Bridge

| Feature | Default Bridge (`docker0`) | User-Defined Bridge |
|---------|---------------------------|---------------------|
| **DNS resolution** | ❌ No (use `--link`, deprecated) | ✅ Automatic by container name |
| **Isolation** | All containers on same bridge | Only containers on that network |
| **Connect/disconnect live** | ❌ Must recreate container | ✅ Can connect/disconnect anytime |
| **Environment sharing** | Via `--link` (deprecated) | Not shared |

> **Always use user-defined bridge networks.** The default bridge is legacy.

---

## Overlay Network (Swarm / Multi-Host)

```mermaid
graph TB
    subgraph "Host 1"
        S1[Service Task 1]
        VX1[VXLAN Tunnel Endpoint]
    end
    subgraph "Host 2"
        S2[Service Task 2]
        VX2[VXLAN Tunnel Endpoint]
    end
    subgraph "Host 3"
        S3[Service Task 3]
        VX3[VXLAN Tunnel Endpoint]
    end

    S1 --- VX1
    S2 --- VX2
    S3 --- VX3
    VX1 <-->|"Overlay Network<br/>(VXLAN)"| VX2
    VX2 <-->|"Overlay Network<br/>(VXLAN)"| VX3
    VX1 <-->|"Overlay Network<br/>(VXLAN)"| VX3

    style S1 fill:#3498db,color:#fff
    style S2 fill:#3498db,color:#fff
    style S3 fill:#3498db,color:#fff
```

- Uses **VXLAN** encapsulation to tunnel L2 frames over L3
- Created automatically for swarm services
- Can be made `--attachable` to allow standalone containers

```bash
# Create overlay for swarm
docker network create --driver overlay my-overlay

# Attachable overlay (standalone containers can join)
docker network create --driver overlay --attachable my-overlay

# Encrypted overlay (data plane encryption)
docker network create --driver overlay --opt encrypted my-secure-overlay
```

---

## Host Network

```bash
# Container shares host's network namespace
docker run --network host nginx

# No port mapping needed — container binds directly to host ports
# nginx is directly on host:80
```

- No network isolation between container and host
- No NAT, no port mapping — maximum performance
- Container can see all host interfaces
- **Linux only** (on macOS/Windows, host networking works differently)

---

## Network Commands

```bash
# List networks
docker network ls

# Create a bridge network
docker network create my-network
docker network create --driver bridge --subnet 172.20.0.0/16 my-network

# Connect container to network
docker network connect my-network my-container

# Disconnect from network
docker network disconnect my-network my-container

# Inspect a network
docker network inspect my-network

# Run container on specific network
docker run -d --name web --network my-network nginx

# Remove / prune
docker network rm my-network
docker network prune
```

---

## Port Publishing

```bash
# Publish port — host:container
docker run -d -p 8080:80 nginx

# Bind to specific host interface
docker run -d -p 127.0.0.1:8080:80 nginx

# Random host port for all EXPOSE ports
docker run -d -P nginx

# UDP port
docker run -d -p 8080:80/udp myapp

# Check published ports
docker port <container>
```

---

## DNS in Docker

```mermaid
graph TB
    subgraph "User-Defined Bridge Network"
        C1["Container: web"]
        C2["Container: api"]
        DNS["Embedded DNS Server<br/>127.0.0.11"]
    end

    C1 -->|"ping api"| DNS
    DNS -->|"Resolves to 172.18.0.3"| C2
```

- **User-defined networks** → Automatic DNS resolution by container name
- **Default bridge** → No DNS, must use IP or deprecated `--link`
- Embedded DNS server at `127.0.0.11`
- Containers can also use `--dns` flag to specify external DNS

```bash
# Custom DNS
docker run --dns 8.8.8.8 nginx

# Custom DNS search domain
docker run --dns-search example.com nginx
```

---

## Swarm Routing Mesh

```mermaid
graph TB
    subgraph "Swarm Cluster"
        subgraph "Node 1 (has task)"
            INGRESS1["Published Port: 8080"]
            T1["Service Task ✅"]
        end
        subgraph "Node 2 (NO task)"
            INGRESS2["Published Port: 8080"]
        end
        subgraph "Node 3 (has task)"
            INGRESS3["Published Port: 8080"]
            T2["Service Task ✅"]
        end
    end

    CLIENT["External Client"] -->|"Request to :8080<br/>on ANY node"| INGRESS2
    INGRESS2 -->|"Routing Mesh<br/>Load Balances"| T1
    INGRESS2 -.->|"or routes to"| T2

    style CLIENT fill:#e74c3c,color:#fff
```

- Published ports are available on **every node** in the swarm
- The **ingress network** handles routing to a node that has the task
- Built-in **internal load balancing** distributes across tasks

### Bypass Routing Mesh

```bash
# Direct host mode — only accessible on nodes running the task
docker service create --name web \
  --publish mode=host,target=80,published=8080 \
  nginx
```

---

## Swarm Network Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| **2377** | TCP | Cluster management (Raft) |
| **7946** | TCP + UDP | Container network discovery (gossip) |
| **4789** | UDP | Overlay network data (VXLAN) |

---

[← Back to Index](./README.md) · [Next: Security →](./07-security.md)
