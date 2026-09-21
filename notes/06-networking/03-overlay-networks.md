# Overlay Networks & Swarm Networking

[← Back to Section Index](./README.md) · [← Main Index](../README.md) · [Previous: Bridge Networks](./02-bridge-networks.md)

---

## What Is an Overlay Network?

Overlay networks connect containers across multiple Docker hosts. They use **VXLAN** encapsulation to tunnel Layer 2 frames over the Layer 3 network, creating a virtual network that spans the cluster.

```mermaid
graph TB
    subgraph "Host 1"
        S1["Service Task 1<br/>10.0.0.2"]
        VX1["VXLAN Tunnel Endpoint"]
    end
    subgraph "Host 2"
        S2["Service Task 2<br/>10.0.0.3"]
        VX2["VXLAN Tunnel Endpoint"]
    end
    subgraph "Host 3"
        S3["Service Task 3<br/>10.0.0.4"]
        VX3["VXLAN Tunnel Endpoint"]
    end

    S1 --- VX1
    S2 --- VX2
    S3 --- VX3
    VX1 <-->|"VXLAN over UDP:4789"| VX2
    VX2 <-->|"VXLAN over UDP:4789"| VX3
    VX1 <-->|"VXLAN over UDP:4789"| VX3

    style S1 fill:#9b59b6,color:#fff
    style S2 fill:#9b59b6,color:#fff
    style S3 fill:#9b59b6,color:#fff
```

---

## Creating Overlay Networks

> **Prerequisite:** Overlay networks require Docker Swarm. You must initialize Swarm before creating overlays — even a single-node Swarm is enough for testing. Without Swarm you'll get: `Error: This node is not a swarm manager.`
>
> ```bash
> # Initialize single-node Swarm (required before creating overlays)
> docker swarm init
>
> # Leave Swarm when done testing
> docker swarm leave --force
> ```

```bash
# Basic overlay for Swarm services
docker network create --driver overlay my-overlay

# Attachable overlay — standalone containers can also join
docker network create --driver overlay --attachable my-overlay

# Encrypted overlay — data plane encryption (IPSec)
docker network create --driver overlay --opt encrypted my-secure-overlay

# With specific subnet
docker network create --driver overlay --subnet 10.10.0.0/16 my-overlay
```

| Flag | Purpose |
|------|---------|
| `--driver overlay` | Use the overlay driver |
| `--attachable` | Allow standalone containers (not just Swarm services) to connect |
| `--opt encrypted` | Encrypt data traffic between nodes (IPSec). Adds overhead. |
| `--internal` | No external connectivity — only inter-container traffic |

> Without `--attachable`, only Swarm services can use the overlay. Standalone `docker run` containers cannot join.

---

## Swarm Networking — Built-in Networks

When you initialize a Swarm, Docker creates two special networks automatically:

| Network | Driver | Purpose |
|---------|--------|---------|
| **ingress** | overlay | Handles the routing mesh for published service ports. All nodes participate. |
| **docker_gwbridge** | bridge | Connects overlay networks to the Docker host's network. Handles outbound traffic from Swarm. |

```bash
# See them after swarm init
docker network ls
# NETWORK ID     NAME              DRIVER    SCOPE
# abc123         bridge            bridge    local
# def456         docker_gwbridge   bridge    local
# ghi789         host              host      local
# jkl012         ingress           overlay   swarm
# mno345         none              null      local
```

---

## Routing Mesh

The routing mesh ensures that a published service port is accessible on **every node** in the Swarm, even nodes that aren't running a task for that service.

```mermaid
graph TB
    subgraph "Swarm Cluster"
        subgraph "Node 1 (has task)"
            INGRESS1["Published Port :8080"]
            T1["Service Task"]
        end
        subgraph "Node 2 (NO task)"
            INGRESS2["Published Port :8080"]
        end
        subgraph "Node 3 (has task)"
            INGRESS3["Published Port :8080"]
            T2["Service Task"]
        end
    end

    CLIENT["External Client"] -->|"Request to :8080<br/>on ANY node"| INGRESS2
    INGRESS2 -->|"Routing mesh<br/>load balances"| T1
    INGRESS2 -.->|"or routes to"| T2

    style CLIENT fill:#e74c3c,color:#fff
    style T1 fill:#9b59b6,color:#fff
    style T2 fill:#9b59b6,color:#fff
```

- Published ports are available on **every node** via the ingress network
- Built-in **internal load balancing** distributes across running tasks
- The routing mesh uses IPVS (IP Virtual Server) in the Linux kernel

### Bypass the Routing Mesh

For cases where you need direct access (e.g., sticky sessions, host-level load balancer):

```bash
# Host mode — port only accessible on nodes running the task
docker service create --name web \
  --publish mode=host,target=80,published=8080 \
  nginx
```

| Mode | Behavior |
|------|----------|
| `mode=ingress` (default) | Port available on all nodes; routing mesh distributes traffic |
| `mode=host` | Port only available on nodes running a task; no load balancing |

---

## Swarm Network Ports

These ports must be open between Swarm nodes:

| Port | Protocol | Purpose |
|------|----------|---------|
| **2377** | TCP | Cluster management and Raft consensus |
| **7946** | TCP + UDP | Container network discovery (gossip protocol) |
| **4789** | UDP | Overlay network data traffic (VXLAN) |

> If you enable `--opt encrypted`, the VXLAN traffic on port 4789 is encrypted with IPSec (ESP, IP protocol 50).

---

## Overlay Network Traffic Flow

```mermaid
sequenceDiagram
    participant C1 as Container A (Host 1)
    participant BR1 as Overlay Bridge (Host 1)
    participant VTEP1 as VXLAN Endpoint (Host 1)
    participant NET as Physical Network
    participant VTEP2 as VXLAN Endpoint (Host 2)
    participant BR2 as Overlay Bridge (Host 2)
    participant C2 as Container B (Host 2)

    C1->>BR1: Ethernet frame to Container B
    BR1->>VTEP1: Frame needs cross-host delivery
    VTEP1->>NET: Encapsulate in VXLAN (UDP:4789)
    NET->>VTEP2: Deliver UDP packet
    VTEP2->>BR2: Decapsulate, extract inner frame
    BR2->>C2: Deliver to Container B
```

---

## Cleanup After Leaving Swarm

`docker swarm leave --force` removes the node from the cluster but does **not** delete any networks. Overlay networks, `ingress`, and `docker_gwbridge` become orphaned — they show in `docker network ls` but are non-functional.

```bash
# Leave Swarm
docker swarm leave --force

# Orphaned networks still visible
docker network ls
# NETWORK ID     NAME              DRIVER    SCOPE
# abc123         my-overlay        overlay   swarm   ← orphaned, unusable
# def456         ingress           overlay   swarm   ← orphaned
# ghi789         docker_gwbridge   bridge    local   ← leftover

# Clean up manually
docker network rm my-overlay
docker network rm ingress
docker network rm docker_gwbridge

# Or prune all unused networks at once
docker network prune
```

> This is consistent with Docker's general philosophy — Docker never auto-deletes resources. Just like `docker rm` doesn't delete volumes and `docker service rm` doesn't delete networks, `docker swarm leave` doesn't clean up networks created during Swarm membership.

---

[← Back to Section Index](./README.md) · [← Main Index](../README.md) · [Previous: Bridge Networks](./02-bridge-networks.md) · [Next: Host, Macvlan, IPvlan & None →](./04-host-macvlan-ipvlan-none.md)
