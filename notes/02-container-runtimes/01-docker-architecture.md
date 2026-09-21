# Docker Architecture & Engine

[← Back to Section Index](./README.md) · [← Main Index](../README.md) · [Previous: Containers Fundamentals](../01-containers-fundamentals.md)

---

## Docker Platform Overview

```mermaid
graph TB
    subgraph "Docker Client"
        CLI["docker CLI"]
        COMPOSE["docker compose"]
    end

    subgraph "Docker Host / Daemon"
        DOCKERD["dockerd<br/>Docker Daemon"]
        CONTAINERD["containerd<br/>Container Runtime"]
        RUNC["runc<br/>OCI Runtime"]
        subgraph "Managed Objects"
            IMG[Images]
            CONT[Containers]
            NET[Networks]
            VOL[Volumes]
        end
    end

    subgraph "Registry"
        DHR[Docker Hub]
        PRIV["Private Registry<br/>DTR / Harbor / ECR"]
    end

    CLI -->|REST API| DOCKERD
    COMPOSE -->|REST API| DOCKERD
    DOCKERD --> CONTAINERD
    CONTAINERD --> RUNC
    RUNC --> CONT
    DOCKERD --> IMG & NET & VOL
    DOCKERD <-->|push / pull| DHR & PRIV

    style DOCKERD fill:#2496ED,color:#fff
    style CONTAINERD fill:#575757,color:#fff
    style RUNC fill:#263238,color:#fff
```

---

## Engine Components

| Component | Role |
|-----------|------|
| **`dockerd`** | Main daemon. Exposes REST API, manages images, networks, volumes |
| **`containerd`** | Container lifecycle management — start, stop, pause, delete |
| **`runc`** | OCI-compliant low-level runtime. Creates containers using Linux kernel features |
| **`containerd-shim`** | Keeps STDIN/STDOUT open, reports exit status after runc exits. Allows daemonless containers |

---

## Process Hierarchy — Who Stays Alive and Why

A common misconception is that `runc` or `containerd` are short-lived like the CLI. In reality, here's what the process tree looks like on a running Docker host:

```
systemd
 ├── dockerd                    ← always running (management layer)
 │    └── containerd            ← always running (container lifecycle layer)
 │
 ├── containerd-shim            ← one per container (lives with the container)
 │    └── nginx                 ← actual container process
 │
 ├── containerd-shim            ← one per container
 │    └── redis-server
```

| Process | Lifetime | What It Does |
|---------|----------|-------------|
| **`dockerd`** | Always running | Management plane — REST API, images, networks, volumes, Swarm |
| **`containerd`** | Always running | Container lifecycle — tracks all containers, handles start/stop/pause |
| **`containerd-shim`** | Per container — lives as long as the container | Direct parent of the container process. Holds stdio, reports exit codes |
| **`runc`** | Ephemeral — exits immediately | Sets up namespaces + cgroups, exec's the process, then exits |

### dockerd vs containerd — Responsibility Split

The Docker engine was refactored post-2016 to cleanly separate concerns between `dockerd` and `containerd`. Think of it as:

```mermaid
graph LR
    subgraph "dockerd — The Manager"
        D1["REST API<br/>/var/run/docker.sock"]
        D2["Image mgmt<br/>pull, push, build"]
        D3["Network mgmt<br/>bridge, overlay, DNS"]
        D4["Volume mgmt"]
        D5["Swarm orchestration"]
        D6["Logging drivers<br/>Restart policies"]
    end

    subgraph "containerd — The Executor"
        C1["Run containers<br/>via shims + runc"]
        C2["Container state tracking<br/>created, running, stopped"]
        C3["Filesystem snapshots<br/>layers for running containers"]
        C4["Task management<br/>the process inside the container"]
    end

    D1 --> C1

    style D1 fill:#2496ED,color:#fff
    style D2 fill:#2496ED,color:#fff
    style D3 fill:#2496ED,color:#fff
    style D4 fill:#2496ED,color:#fff
    style D5 fill:#2496ED,color:#fff
    style D6 fill:#2496ED,color:#fff
    style C1 fill:#575757,color:#fff
    style C2 fill:#575757,color:#fff
    style C3 fill:#575757,color:#fff
    style C4 fill:#575757,color:#fff
```

> In plain terms:
> - **`dockerd`** = *"What should run and how should it be configured?"*
> - **`containerd`** = *"Actually run it and keep track of it."*
> - **`runc`** = *"Set up the Linux sandbox and launch the process."*
> - **`containerd-shim`** = *"Babysit the process after runc leaves."*

### Surviving Daemon Restarts — `live-restore`

This layered design gives Docker a critical property — **you can restart `dockerd` without killing running containers**.

The chain of survival works because:
1. `containerd-shim` is the direct parent of the container process — it doesn't depend on `dockerd`
2. `containerd` tracks shims, but shims survive `containerd` restarts too
3. With `live-restore` enabled, the entire daemon can restart and reconnect to running containers

```json
// /etc/docker/daemon.json
{
  "live-restore": true
}
```

```bash
# Restart Docker daemon — containers keep running
sudo systemctl restart docker

# Verify containers are still up
docker ps   # → containers still listed and running
```

> **Why Kubernetes dropped dockerd**: This separation is exactly why Kubernetes talks directly to `containerd` via the CRI (Container Runtime Interface) since v1.24. `dockerd` was unnecessary overhead — K8s handles its own scheduling, networking, and image management. It only needed the "executor" layer.

---

## What Happens When You Run `docker run`

```mermaid
sequenceDiagram
    participant User as docker CLI
    participant Daemon as dockerd
    participant CD as containerd
    participant Shim as containerd-shim
    participant Runc as runc
    participant Container as Container Process

    User->>Daemon: docker run nginx
    Daemon->>Daemon: Check image locally
    Daemon->>Daemon: Pull image if not found
    Daemon->>CD: Create container
    CD->>Shim: Start shim process
    Shim->>Runc: Create + start container
    Runc->>Container: Exec container process
    Runc-->>Shim: Exit (runc exits after start)
    Note over Shim,Container: Shim becomes parent<br/>of container process
    Shim-->>CD: Report container status
    CD-->>Daemon: Container running
    Daemon-->>User: Container ID
```

### Key Takeaways

- **`runc` exits** after the container starts — it doesn't stay running
- **`containerd-shim`** becomes the parent process of the container
- This design means the Docker daemon can be restarted **without killing running containers** (when `live-restore` is enabled)

---

## Docker Objects

```mermaid
graph LR
    subgraph "Docker Objects"
        I["Image<br/>Read-only template<br/>Layered filesystem"]
        C["Container<br/>Running instance of image<br/>Writable layer on top"]
        N["Network<br/>Communication channel<br/>between containers"]
        V["Volume<br/>Persistent data storage<br/>Outlives containers"]
        S["Service<br/>Swarm-managed<br/>container definition"]
    end

    I -->|"docker run"| C
    C ---|connects to| N
    C ---|mounts| V
    S -->|manages| C
```

| Object | Description |
|--------|-------------|
| **Image** | Read-only template with app code, runtime, libraries, config. Built from Dockerfile. |
| **Container** | Runnable instance of an image. Has a writable layer on top of image layers. |
| **Network** | Provides communication between containers. Multiple drivers available. |
| **Volume** | Persistent storage that outlives containers. Managed by Docker. |
| **Service** | Definition of tasks to run in a Swarm. Manages desired state of replicas. |

---

## Docker Client-Server Communication

```mermaid
graph LR
    subgraph "Communication Channels"
        UNIX["Unix Socket (default)<br/>/var/run/docker.sock<br/>Local only"]
        TCP["TCP Socket<br/>tcp://host:2375 (no TLS)<br/>tcp://host:2376 (TLS)"]
    end

    CLIENT[Docker CLI] -->|default| UNIX
    CLIENT -->|remote| TCP
    UNIX --> DAEMON[dockerd]
    TCP --> DAEMON

    style UNIX fill:#27ae60,color:#fff
    style TCP fill:#e74c3c,color:#fff
```

```bash
# Default: local unix socket
docker ps

# Remote: specify host
docker -H tcp://192.168.1.100:2376 ps

# Or via environment variable
export DOCKER_HOST=tcp://192.168.1.100:2376
docker ps
```

> **Security:** Never expose the TCP socket without TLS. Access to the Docker socket = root access to the host.

---

## Container Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Created: docker create
    Created --> Running: docker start
    [*] --> Running: docker run
    Running --> Paused: docker pause
    Paused --> Running: docker unpause
    Running --> Stopped: docker stop
    Running --> Stopped: docker kill
    Stopped --> Running: docker start
    Stopped --> Removed: docker rm
    Removed --> [*]
```

| Command | Signal | Behavior |
|---------|--------|----------|
| `docker stop` | SIGTERM → (10s) → SIGKILL | Graceful shutdown with timeout |
| `docker kill` | SIGKILL | Immediate force stop |
| `docker pause` | SIGSTOP (via cgroup freezer) | Freeze all processes |
| `docker unpause` | SIGCONT | Resume frozen processes |

---

[← Back to Section Index](./README.md) · [← Main Index](../README.md) · [Next: Podman Architecture →](./02-podman-architecture.md) · [Orchestration →](../03-orchestration/README.md)
