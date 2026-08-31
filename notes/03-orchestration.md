# Domain 1: Orchestration (25% of Exam)

[← Back to Index](./README.md) · [Previous: Docker Architecture](./02-container-runtimes/01-docker-architecture.md)

> 🔴 **Highest weighted domain — study this thoroughly!**

---

## Docker Swarm Architecture

```mermaid
graph TB
    subgraph "Docker Swarm Cluster"
        subgraph "Manager Nodes"
            M1["Manager 1<br/>LEADER ⭐"]
            M2["Manager 2<br/>REACHABLE"]
            M3["Manager 3<br/>REACHABLE"]
        end
        subgraph "Worker Nodes"
            W1[Worker 1]
            W2[Worker 2]
            W3[Worker 3]
            W4[Worker 4]
        end
    end

    M1 <-->|Raft Consensus| M2
    M2 <-->|Raft Consensus| M3
    M3 <-->|Raft Consensus| M1

    M1 -->|Task Assignment| W1 & W2
    M2 -->|Task Assignment| W3
    M3 -->|Task Assignment| W4

    style M1 fill:#e74c3c,color:#fff
    style M2 fill:#f39c12,color:#fff
    style M3 fill:#f39c12,color:#fff
    style W1 fill:#3498db,color:#fff
    style W2 fill:#3498db,color:#fff
    style W3 fill:#3498db,color:#fff
    style W4 fill:#3498db,color:#fff
```

### Manager vs Worker Nodes

| | Manager Nodes | Worker Nodes |
|---|--------------|--------------|
| **Role** | Maintain cluster state, schedule tasks | Execute container tasks |
| **Raft** | Participate in Raft consensus | Do not participate |
| **Run tasks?** | Yes (by default) | Yes |
| **API access** | Accept management commands | Cannot manage cluster |

> Managers can be drained to prevent running tasks: `docker node update --availability drain <manager>`

---

## Swarm Setup Commands

```bash
# Initialize swarm on first manager
docker swarm init --advertise-addr <MANAGER-IP>

# Get join tokens
docker swarm join-token manager
docker swarm join-token worker

# Join as worker
docker swarm join --token <WORKER-TOKEN> <MANAGER-IP>:2377

# Join as manager
docker swarm join --token <MANAGER-TOKEN> <MANAGER-IP>:2377

# Promote / demote nodes
docker node promote <node-name>
docker node demote <node-name>

# List nodes
docker node ls

# Inspect a node
docker node inspect --pretty <node-name>

# Drain a node (maintenance)
docker node update --availability drain <node-name>

# Re-activate a drained node
docker node update --availability active <node-name>

# Leave swarm
docker swarm leave           # Worker leaves
docker swarm leave --force   # Manager leaves (⚠️ may break quorum)
```

---

## Raft Consensus & Quorum

```mermaid
graph TB
    subgraph "Quorum = (N/2) + 1"
        Q1["1 manager → Quorum: 1<br/>Fault Tolerance: 0 ❌"]
        Q2["2 managers → Quorum: 2<br/>Fault Tolerance: 0 ❌"]
        Q3["3 managers → Quorum: 2<br/>Fault Tolerance: 1 ✅"]
        Q5["5 managers → Quorum: 3<br/>Fault Tolerance: 2 ✅"]
        Q7["7 managers → Quorum: 4<br/>Fault Tolerance: 3 ✅"]
    end
```

| Managers | Quorum | Fault Tolerance |
|----------|--------|-----------------|
| 1 | 1 | 0 |
| 2 | 2 | 0 |
| **3** | **2** | **1** |
| **5** | **3** | **2** |
| **7** | **4** | **3** |

> **Rules:**
> - Always use an **odd number** of managers (3, 5, or 7)
> - Docker recommends a **maximum of 7** managers (more adds Raft overhead)
> - If quorum is lost, the swarm cannot accept new management commands

---

## Services, Tasks & Containers

```mermaid
graph TB
    SVC["Service<br/>(desired state definition)"]
    SVC --> T1[Task 1]
    SVC --> T2[Task 2]
    SVC --> T3[Task 3]

    T1 --> C1["Container<br/>Node 1"]
    T2 --> C2["Container<br/>Node 2"]
    T3 --> C3["Container<br/>Node 3"]

    style SVC fill:#9b59b6,color:#fff
    style T1 fill:#e67e22,color:#fff
    style T2 fill:#e67e22,color:#fff
    style T3 fill:#e67e22,color:#fff
    style C1 fill:#2ecc71,color:#fff
    style C2 fill:#2ecc71,color:#fff
    style C3 fill:#2ecc71,color:#fff
```

- **Service** — the definition: which image, how many replicas, ports, networks
- **Task** — a slot in the scheduler that maps to one container
- **Container** — the actual running process on a node

---

## Replicated vs Global Services

```mermaid
graph TB
    subgraph "Replicated Service (default)"
        RS["Service: web<br/>replicas = 3"]
        RS --> N1A["Node 1: ✅"]
        RS --> N2A["Node 2: ✅"]
        RS --> N3A["Node 3: ✅"]
        RS -.->|No task| N4A["Node 4: ❌"]
    end

    subgraph "Global Service"
        GS["Service: monitoring<br/>mode = global"]
        GS --> N1B["Node 1: ✅"]
        GS --> N2B["Node 2: ✅"]
        GS --> N3B["Node 3: ✅"]
        GS --> N4B["Node 4: ✅"]
    end
```

| Mode | Behavior | Use Case |
|------|----------|----------|
| **Replicated** | Runs N replicas across the cluster | Web apps, APIs |
| **Global** | Exactly one task per node | Monitoring agents, log collectors |

---

## Service Commands

```bash
# Create a replicated service
docker service create --name web --replicas 3 -p 8080:80 nginx

# Create a global service
docker service create --name agent --mode global datadog/agent

# List services
docker service ls

# Inspect service
docker service inspect --pretty web

# View tasks
docker service ps web

# Scale a service
docker service scale web=5

# Remove a service
docker service rm web
```

---

## Rolling Updates & Rollbacks

```mermaid
sequenceDiagram
    participant Swarm as Swarm Manager
    participant T1 as Task 1 (v1)
    participant T2 as Task 2 (v1)
    participant T3 as Task 3 (v1)

    Note over Swarm: docker service update --image v2<br/>--update-parallelism 1<br/>--update-delay 10s

    Swarm->>T1: Stop v1, Start v2
    Note over T1: ✅ Task 1 now v2
    Note over Swarm: Wait 10s delay
    Swarm->>T2: Stop v1, Start v2
    Note over T2: ✅ Task 2 now v2
    Note over Swarm: Wait 10s delay
    Swarm->>T3: Stop v1, Start v2
    Note over T3: ✅ Task 3 now v2
    Note over Swarm: ✅ Update complete
```

### Update Configuration Flags

```bash
docker service update \
  --image nginx:1.25 \
  --update-parallelism 2 \       # Update 2 tasks at a time
  --update-delay 10s \           # Wait 10s between batches
  --update-failure-action rollback \  # Auto-rollback on failure
  --update-max-failure-ratio 0.25 \   # Tolerate 25% failures
  --update-order start-first \   # Start new before stopping old
  web
```

### Rollback

```bash
# Manual rollback
docker service rollback web

# Rollback config (set at service creation)
docker service create \
  --rollback-parallelism 1 \
  --rollback-delay 5s \
  --rollback-failure-action pause \
  --name web nginx
```

---

## Placement Constraints & Preferences

```bash
# Constraint: only on worker nodes
docker service create --constraint 'node.role==worker' --name web nginx

# Constraint: specific label
docker node update --label-add region=us-east node1
docker service create --constraint 'node.labels.region==us-east' --name web nginx

# Preference: spread across zones
docker service create \
  --placement-pref 'spread=node.labels.datacenter' \
  --name web nginx
```

| Type | Purpose | Behavior |
|------|---------|----------|
| **Constraint** | Hard requirement | Task will NOT be scheduled if constraint not met |
| **Preference** | Soft preference | Swarm tries to spread, but won't fail if it can't |

---

## Stacks (Compose in Swarm)

```bash
# Deploy a stack using compose file
docker stack deploy -c docker-compose.yml myapp

# List stacks
docker stack ls

# List services in a stack
docker stack services myapp

# List tasks in a stack
docker stack ps myapp

# Remove a stack
docker stack rm myapp
```

> Stacks use **version 3+** compose files. Some compose features (like `build`) are ignored in stack mode.

---

## Locking a Swarm (Autolock)

```bash
# Enable autolock on init
docker swarm init --autolock

# Enable autolock on existing swarm
docker swarm update --autolock=true

# Unlock after manager restart
docker swarm unlock

# Rotate unlock key
docker swarm unlock-key --rotate
```

> **Why autolock?** Raft logs and TLS keys are encrypted at rest. Autolock ensures a restarted manager must provide the unlock key before rejoining the cluster, protecting against disk theft scenarios.

---

## Swarm Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| **2377** | TCP | Cluster management & Raft |
| **7946** | TCP + UDP | Node-to-node communication (gossip) |
| **4789** | UDP | Overlay network traffic (VXLAN) |

---

[← Back to Index](./README.md) · [Next: Image Creation & Management →](./04-image-creation-management.md)
