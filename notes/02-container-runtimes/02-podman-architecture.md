# Podman Architecture & Engine

[← Back to Section Index](./README.md) · [← Main Index](../README.md) · [Related: Docker Architecture](./01-docker-architecture.md)

---

## Podman Platform Overview

Podman (Pod Manager) is a **daemonless, rootless** container engine developed by Red Hat. Unlike Docker, there is no central daemon — each `podman` command forks its own runtime process directly. This eliminates the single point of failure that `dockerd` represents.

```mermaid
graph TB
    subgraph "Podman Client"
        CLI["podman CLI"]
        COMPOSE["podman compose"]
        API["podman system service<br/>REST API (optional)"]
    end

    subgraph "Podman Host (No Daemon)"
        CONMON["conmon<br/>Container Monitor"]
        RUNC["crun / runc<br/>OCI Runtime"]
        subgraph "Managed Objects"
            IMG[Images]
            CONT[Containers]
            POD[Pods]
            NET[Networks]
            VOL[Volumes]
        end
    end

    subgraph "Registry"
        QUAY["Quay.io"]
        DHR["Docker Hub"]
        PRIV["Private Registry<br/>Harbor / ECR / GHCR"]
    end

    CLI -->|"fork + exec"| CONMON
    COMPOSE -->|"fork + exec"| CONMON
    API -->|"fork + exec"| CONMON
    CONMON --> RUNC
    RUNC --> CONT
    CONT -.- POD
    CLI --> IMG & NET & VOL
    CLI <-->|"push / pull"| QUAY & DHR & PRIV

    style CONMON fill:#892ca0,color:#fff
    style RUNC fill:#263238,color:#fff
    style CLI fill:#892ca0,color:#fff
    style POD fill:#e74c3c,color:#fff
```

### Docker vs Podman — Architecture at a Glance

```mermaid
graph TB
    subgraph "Docker Architecture"
        direction TB
        DC["docker CLI"] -->|REST API| DD["dockerd ⬛<br/>Central Daemon"]
        DD --> DCD["containerd"]
        DCD --> DRC["runc"]
        DRC --> DCT1["Container"]
        DRC --> DCT2["Container"]
    end

    subgraph "Podman Architecture"
        direction TB
        PC["podman CLI"] -->|"fork"| PM1["conmon 🟣"]
        PC -->|"fork"| PM2["conmon 🟣"]
        PM1 --> PRC1["crun / runc"]
        PM2 --> PRC2["crun / runc"]
        PRC1 --> PCT1["Container"]
        PRC2 --> PCT2["Container"]
    end

    style DD fill:#2496ED,color:#fff
    style PM1 fill:#892ca0,color:#fff
    style PM2 fill:#892ca0,color:#fff
```

> **Key difference**: Docker routes everything through a single daemon (`dockerd`). Podman forks a dedicated `conmon` process per container — no central daemon, no single point of failure.

---

## Engine Components

| Component | Role |
|-----------|------|
| **`podman`** | CLI tool. No daemon — directly interacts with the runtime. Drop-in replacement for `docker` CLI. |
| **`conmon`** | Container monitor. One per container. Holds STDIN/STDOUT, monitors exit status, handles logging. |
| **`crun`** | Default OCI runtime (written in C, faster than runc). `runc` is also supported. |
| **`containers/image`** | Library for pulling, pushing, and inspecting images across registries. |
| **`containers/storage`** | Library for managing image layers and container filesystems (OverlayFS, VFS, etc.). |
| **`CNI` / `Netavark`** | Network stack. Netavark (Rust-based) is the new default, replacing CNI plugins. |
| **`Aardvark-dns`** | DNS server for container name resolution, paired with Netavark. |

---

## Process Hierarchy — No Daemon, No Problem

The biggest architectural difference from Docker: **there is no long-running daemon**. The `podman` CLI is a regular process that does its work and exits. Here's what the process tree looks like on a host running Podman containers:

```
systemd (or user session)
 │
 ├── conmon                     ← one per container (container monitor)
 │    └── nginx                 ← actual container process
 │
 ├── conmon                     ← one per container
 │    └── redis-server
 │
 └── (no podman process)        ← CLI already exited!
```

Compare this to Docker's tree:

```
Docker:                              Podman:
─────────────────────────            ─────────────────────────
systemd                              systemd
 ├── dockerd          ← always       │
 │    └── containerd  ← always       ├── conmon  ← per container
 │                                   │    └── nginx
 ├── containerd-shim  ← per ctr     │
 │    └── nginx                      ├── conmon  ← per container
 │                                   │    └── redis
 ├── containerd-shim  ← per ctr     │
 │    └── redis                      └── (nothing else)
```

| Process | Lifetime | What It Does |
|---------|----------|-------------|
| **`podman`** | Ephemeral — exits after command completes | CLI tool. Prepares the container bundle, forks conmon, then exits |
| **`conmon`** | Per container — lives as long as the container | Direct parent of container process. Holds stdio, captures logs, reports exit codes |
| **`crun` / `runc`** | Ephemeral — exits immediately | Sets up namespaces + cgroups, exec's the process, then exits |

> Notice: only **one** long-lived process per container (`conmon`). Docker needs **three** always-running processes (`dockerd` + `containerd` + `containerd-shim`) plus one shim per container.

### Who Does What — Without a Daemon

In Docker, `dockerd` handles images/networks/volumes and `containerd` handles container lifecycle. In Podman, those responsibilities are split across **libraries** linked into the `podman` binary itself — not separate daemons.

```mermaid
graph LR
    subgraph "podman CLI (runs, then exits)"
        P1["containers/image<br/>Pull, push, inspect images"]
        P2["containers/storage<br/>Manage layers, OverlayFS"]
        P3["Netavark / CNI<br/>Set up container networking"]
        P4["Buildah<br/>Build images from Containerfile"]
    end

    subgraph "conmon (stays alive per container)"
        C1["Hold STDIN / STDOUT / STDERR"]
        C2["Capture container logs"]
        C3["Monitor exit status"]
        C4["Reap zombie processes"]
    end

    subgraph "crun (exits immediately)"
        R1["Create namespaces + cgroups"]
        R2["Set up rootfs"]
        R3["Exec container process"]
    end

    P1 --> C1
    C1 --> R1

    style P1 fill:#892ca0,color:#fff
    style P2 fill:#892ca0,color:#fff
    style P3 fill:#892ca0,color:#fff
    style P4 fill:#892ca0,color:#fff
    style C1 fill:#e67e22,color:#fff
    style C2 fill:#e67e22,color:#fff
    style C3 fill:#e67e22,color:#fff
    style C4 fill:#e67e22,color:#fff
    style R1 fill:#263238,color:#fff
    style R2 fill:#263238,color:#fff
    style R3 fill:#263238,color:#fff
```

> In plain terms:
> - **`podman` CLI** = *"Prepare everything, fork conmon, then I'm done."*
> - **`conmon`** = *"Babysit the container process for its entire life."*
> - **`crun`** = *"Set up the sandbox, launch the process, then exit."*

### Surviving Without a Daemon

Since there's no daemon to restart, the question becomes: **what keeps containers alive when the user logs out?**

The answer is **`conmon`** + **systemd lingering**:

1. `conmon` is reparented to PID 1 (systemd) after the CLI exits — it doesn't depend on your shell session
2. By default, systemd kills user processes on logout. **Lingering** prevents this:

```bash
# Enable lingering — your user's processes survive logout
loginctl enable-linger $USER

# Verify
loginctl show-user $USER | grep Linger
# → Linger=yes
```

3. For production, use `podman generate systemd` to create proper service units (covered in the Systemd section below)

### Docker vs Podman — Failure Impact

| Scenario | Docker | Podman |
|----------|--------|--------|
| **Daemon crashes** | `dockerd` crash = can't manage ANY container (API gone). Containers keep running via shims, but no new commands work until daemon restarts. | No daemon to crash. Each container is independent. |
| **Single container monitor crashes** | `containerd-shim` crash = that one container is orphaned | `conmon` crash = that one container is orphaned |
| **CLI crashes mid-command** | Daemon keeps the operation state | Operation may be partially complete. Container state stored in `/var/lib/containers` (or `~/.local/share/containers`) allows recovery. |
| **Restart runtime** | `live-restore: true` in daemon.json | Not needed — there's nothing to restart |

> **Bottom line**: Docker has a single point of failure (`dockerd`). Podman has none. But Docker's daemon model makes features like Swarm and event streaming simpler to implement.

---

## What Happens When You Run `podman run`

```mermaid
sequenceDiagram
    participant User as podman CLI
    participant Storage as containers/storage
    participant Conmon as conmon
    participant Runtime as crun / runc
    participant Container as Container Process

    User->>Storage: Check image locally
    User->>Storage: Pull image if not found
    User->>User: Prepare container bundle (rootfs + config.json)
    User->>Conmon: Fork conmon process
    Conmon->>Runtime: Create + start container
    Runtime->>Container: Exec container process
    Runtime-->>Conmon: Exit (runtime exits after start)
    Note over Conmon,Container: conmon becomes parent<br/>of container process
    Note over User: podman CLI exits<br/>(no daemon to stay running)
    Conmon-->>User: Report status on next podman command
```

### Key Takeaways

- **No daemon** — the `podman` CLI process itself orchestrates container creation, then exits
- **`conmon`** becomes the parent process of each container (similar to `containerd-shim` in Docker)
- **`crun`** exits after the container starts — just like `runc` in Docker
- Containers survive the CLI process exiting — `conmon` keeps them alive
- Restarting your shell or user session doesn't kill containers (with lingering enabled)

---

## Podman Objects

```mermaid
graph LR
    subgraph "Podman Objects"
        I["Image<br/>OCI / Docker format<br/>Layered filesystem"]
        C["Container<br/>Running instance of image<br/>Writable layer on top"]
        P["Pod<br/>Group of containers<br/>Shared namespaces"]
        N["Network<br/>Netavark / CNI<br/>Container networking"]
        V["Volume<br/>Persistent data<br/>Named / anonymous"]
    end

    I -->|"podman run"| C
    C ---|part of| P
    C ---|connects to| N
    C ---|mounts| V
    P -->|"podman generate kube"| K["Kubernetes YAML"]

    style P fill:#e74c3c,color:#fff
    style K fill:#326ce5,color:#fff
```

| Object | Description |
|--------|-------------|
| **Image** | OCI-compliant image. Compatible with Docker images. Built with `podman build` or `Containerfile`. |
| **Container** | Runnable instance of an image. Rootless by default. |
| **Pod** | Group of containers sharing the same network, PID, and IPC namespaces — same concept as a Kubernetes Pod. Each pod has an **infra container** that holds the namespaces. |
| **Network** | Managed by Netavark (default) or CNI plugins. Supports bridge, macvlan, and ipvlan drivers. |
| **Volume** | Persistent storage. Named volumes, bind mounts, and tmpfs supported — same semantics as Docker. |

---

## Pods — Podman's Unique Feature

Pods are first-class objects in Podman (Docker has no equivalent). A pod groups containers that need to share namespaces, just like Kubernetes pods.

```mermaid
graph TB
    subgraph POD["Pod: my-webapp"]
        INFRA["Infra Container<br/>(holds namespaces)<br/>Pause process"]
        WEB["Web Container<br/>nginx:latest<br/>Port 80"]
        APP["App Container<br/>flask-api:latest<br/>Port 5000"]
        LOG["Log Container<br/>fluentd:latest"]
    end

    INFRA ---|shared network namespace| WEB
    INFRA ---|shared network namespace| APP
    INFRA ---|shared network namespace| LOG

    NET["Shared localhost<br/>Containers talk via 127.0.0.1"]
    POD --- NET

    style INFRA fill:#95a5a6,color:#fff
    style WEB fill:#2496ED,color:#fff
    style APP fill:#27ae60,color:#fff
    style LOG fill:#f39c12,color:#fff
    style POD fill:#fff5f5,stroke:#e74c3c,stroke-width:2px
```

```bash
# Create a pod
podman pod create --name my-webapp -p 8080:80

# Add containers to the pod
podman run -d --pod my-webapp nginx:latest
podman run -d --pod my-webapp flask-api:latest

# All containers in the pod share localhost
# nginx can reach flask at 127.0.0.1:5000

# Generate Kubernetes YAML from a pod
podman generate kube my-webapp > my-webapp.yaml

# Play a Kubernetes YAML file as Podman pods
podman play kube my-webapp.yaml
```

---

## Rootless Containers

Podman's flagship feature. Containers run entirely in **user space** without requiring root privileges.

```mermaid
graph TB
    subgraph "Rootful (Docker default)"
        ROOT_D["dockerd<br/>Runs as root 🔴"]
        ROOT_C["Container<br/>Root-mapped process"]
        ROOT_D --> ROOT_C
    end

    subgraph "Rootless (Podman default)"
        USER_P["podman CLI<br/>Runs as user 🟢"]
        USER_C["Container<br/>UID-mapped process"]
        USERNS["User Namespace<br/>UID 0 inside → UID 100000+ outside"]
        USER_P --> USER_C
        USER_C --- USERNS
    end

    style ROOT_D fill:#e74c3c,color:#fff
    style USER_P fill:#27ae60,color:#fff
    style USERNS fill:#f39c12,color:#fff
```

| Aspect | Rootful | Rootless |
|--------|---------|----------|
| **Daemon/Process** | Runs as root | Runs as regular user |
| **UID Mapping** | Container UID 0 = Host UID 0 | Container UID 0 = Host UID 100000+ |
| **Risk** | Container escape = root on host | Container escape = unprivileged user |
| **Port Binding** | Any port | Ports > 1024 (by default) |
| **Storage** | `/var/lib/containers` | `~/.local/share/containers` |
| **Network** | Full access (bridge, macvlan) | slirp4netns or pasta (user-mode networking) |

```bash
# Rootless is the default — just run as your normal user
podman run -d -p 8080:80 nginx

# Check who owns the process on the host
ps aux | grep nginx
# → your-username ... conmon ... nginx

# UID mapping visible via
podman unshare cat /proc/self/uid_map
```

---

## Podman vs Docker — Command Compatibility

Podman is designed as a drop-in replacement for the Docker CLI. Most commands are identical.

```bash
# These commands work identically in both
alias docker=podman   # The famous alias

podman pull nginx
podman run -d --name web -p 80:80 nginx
podman ps
podman logs web
podman exec -it web bash
podman stop web
podman rm web
podman build -t myapp .
podman push myapp quay.io/user/myapp
```

### Differences That Matter

| Feature | Docker | Podman |
|---------|--------|--------|
| **Daemon** | `dockerd` required (always running) | No daemon (fork per command) |
| **Rootless** | Supported but not default | Default and first-class |
| **Pods** | Not supported (Swarm services instead) | Native pod support |
| **Compose** | `docker compose` (built-in plugin) | `podman compose` (via podman-compose or docker-compose) |
| **Swarm** | Built-in orchestration | Not supported (use Kubernetes) |
| **Build** | BuildKit (default) | Buildah (integrated) |
| **Kubernetes** | No direct integration | `podman generate kube` / `podman play kube` |
| **Socket** | `/var/run/docker.sock` | `/run/user/UID/podman/podman.sock` (rootless) |
| **Systemd** | Restart policies only | `podman generate systemd` for full service management |
| **Default runtime** | runc | crun (faster, lower memory) |

---

## Container Lifecycle

The lifecycle is nearly identical to Docker, with the same commands.

```mermaid
stateDiagram-v2
    [*] --> Created: podman create
    Created --> Running: podman start
    [*] --> Running: podman run
    Running --> Paused: podman pause
    Paused --> Running: podman unpause
    Running --> Stopped: podman stop
    Running --> Stopped: podman kill
    Stopped --> Running: podman start
    Stopped --> Removed: podman rm
    Removed --> [*]
```

| Command | Signal | Behavior |
|---------|--------|----------|
| `podman stop` | SIGTERM → (10s) → SIGKILL | Graceful shutdown with timeout |
| `podman kill` | SIGKILL | Immediate force stop |
| `podman pause` | SIGSTOP (via cgroup freezer) | Freeze all processes |
| `podman unpause` | SIGCONT | Resume frozen processes |

---

## Podman with Systemd

Unlike Docker's restart policies, Podman integrates natively with systemd for managing container services.

```bash
# Generate a systemd unit file from a running container
podman generate systemd --new --name web > ~/.config/systemd/user/container-web.service

# Enable and start
systemctl --user daemon-reload
systemctl --user enable --now container-web.service

# Check status
systemctl --user status container-web.service

# Containers start on boot (with lingering)
loginctl enable-linger $USER
```

---

## Podman on macOS / Windows — Podman Machine & Podman Desktop

Podman doesn't run natively on macOS/Windows (needs a Linux kernel). Two complementary tools solve this:

1. **`podman machine`** — CLI tool that creates and manages a lightweight Linux VM behind the scenes
2. **Podman Desktop** — A full GUI application for managing containers, pods, images, and Kubernetes resources

```mermaid
graph TB
    subgraph "macOS / Windows"
        PD["Podman Desktop<br/>(GUI Application)"]
        CLI2["podman CLI"]
    end

    subgraph "Podman Machine (Linux VM)"
        PODMAN_SVC["podman system service"]
        CONMON2["conmon"]
        CRUN2["crun"]
        CT["Containers"]
    end

    PD -->|"API"| PODMAN_SVC
    CLI2 -->|"SSH / API"| PODMAN_SVC
    PODMAN_SVC --> CONMON2 --> CRUN2 --> CT

    style PD fill:#e74c3c,color:#fff
    style CLI2 fill:#892ca0,color:#fff
    style PODMAN_SVC fill:#892ca0,color:#fff
```

### Podman Machine (CLI)

```bash
# Initialize a Podman VM
podman machine init

# Start it
podman machine start

# Now use podman normally — it routes to the VM
podman run -d nginx

# SSH into the VM if needed
podman machine ssh
```

**VM providers by platform:**

| Platform | VM Provider |
|----------|------------|
| **macOS** | Apple Virtualization (`applehv`) with Rosetta for x86_64 translation (near-native speed) |
| **Windows** | WSL2 (default) or Hyper-V |
| **Linux** | Optional — runs natively, no VM needed |

### Podman Desktop (GUI)

[Podman Desktop](https://podman-desktop.io) is a standalone open-source desktop application — a graphical counterpart to the `podman` CLI. It's a **CNCF Sandbox project** (since Nov 2024) available on Linux, macOS, and Windows.

**Key capabilities:**
- **Container management** — build, run, stop, inspect, and delete containers and pods through a visual interface
- **Image management** — pull, push, build, and manage images across registries
- **Kubernetes integration** — explore and manage pods, deployments, services, and ingresses; spin up local clusters via Kind or Minikube extensions
- **Extensions** — plugin system supporting Kind, Minikube, Headlamp, and community extensions
- **Multi-engine support** — works with Podman and Docker engines
- **GPU acceleration** — supports GPU passthrough for local AI/ML container workflows
- **Multi-arch builds** — build images for ARM and x86_64 from the GUI

> Podman Desktop handles Podman Machine lifecycle (init, start, stop) automatically through its GUI — no CLI required for getting started.

---

## When to Use Podman vs Docker

| Use Case | Recommendation |
|----------|---------------|
| CI/CD pipelines needing rootless builds | **Podman** |
| Kubernetes-aligned local development | **Podman** (native pod + kube support) |
| Existing Docker Compose workflows | **Docker** (more mature compose support) |
| Security-sensitive environments | **Podman** (rootless default, no daemon) |
| Docker Swarm orchestration | **Docker** (Swarm is Docker-only) |
| macOS/Windows developer experience | **Docker Desktop** (more polished) |
| RHEL / Fedora / CentOS environments | **Podman** (ships by default, Docker not in repos) |
| Learning containers for the first time | **Docker** (larger community, more tutorials) |

---

[← Back to Section Index](./README.md) · [← Main Index](../README.md) · [Related: Docker Architecture](./01-docker-architecture.md)
