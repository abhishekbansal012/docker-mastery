# Containers — Fundamentals & Evolution

[← Back to Index](./README.md)

---

## What Are Containers?

Containers are **OS-level virtualization** units that package an application with its dependencies into a standardized, isolated unit. They share the host OS kernel but are isolated using Linux kernel features (namespaces + cgroups).

---

## Containers vs Virtual Machines

```mermaid
graph TB
    subgraph VM["🖥️ VIRTUAL MACHINE"]
        direction TB
        subgraph VM1["Linux"]
            A1["Application"]
            LD1["Libs &nbsp; | &nbsp; Deps"]
            OS1["OS"]
        end
        subgraph VM2["Windows"]
            A2["Application"]
            LD2["Libs &nbsp; | &nbsp; Deps"]
            OS2["OS"]
        end
        HV["⬛ Hypervisor"]
        HW1["🟦 Hardware Infrastructure"]
    end

    subgraph CT["📦 CONTAINER"]
        direction TB
        subgraph C1["Container 1"]
            A3["Application"]
            LD3["Libs &nbsp; | &nbsp; Deps"]
        end
        subgraph C2["Container 2"]
            A4["Application"]
            LD4["Libs &nbsp; | &nbsp; Deps"]
        end
        DK["🐳 Docker"]
        OS3["🟠 OS"]
        HW2["🟦 Hardware Infrastructure"]
    end

    A1 --- LD1 --- OS1
    A2 --- LD2 --- OS2
    OS1 --- HV
    OS2 --- HV
    HV --- HW1

    A3 --- LD3
    A4 --- LD4
    LD3 --- DK
    LD4 --- DK
    DK --- OS3 --- HW2

    style VM fill:#fff5f5,stroke:#e74c3c,stroke-width:2px
    style CT fill:#f0fff0,stroke:#27ae60,stroke-width:2px
    style VM1 fill:#fff,stroke:#e67e22,stroke-width:1px
    style VM2 fill:#fff,stroke:#e67e22,stroke-width:1px
    style C1 fill:#e8f8f5,stroke:#1abc9c,stroke-width:1px
    style C2 fill:#e8f8f5,stroke:#1abc9c,stroke-width:1px
    style HV fill:#e74c3c,color:#fff,stroke:#c0392b
    style HW1 fill:#3498db,color:#fff,stroke:#2980b9
    style DK fill:#3498db,color:#fff,stroke:#2980b9
    style OS3 fill:#e8a87c,color:#fff,stroke:#e67e22
    style HW2 fill:#6c3483,color:#fff,stroke:#5b2c6f

    linkStyle default stroke:#999,stroke-width:1px
```

> **Key Differences at a Glance**
>
> | | Virtual Machines | Containers |
> |---|---|---|
> | 📊 **Utilization** | Heavy (full OS per VM) | Lightweight (shared OS) |
> | 💾 **Size** | **GBs** | **MBs** |
> | ⏱️ **Boot up** | **Minutes** | **Seconds** |

| Feature | Containers | Virtual Machines |
|---------|-----------|-----------------|
| Boot Time | Seconds | Minutes |
| Size | MBs | GBs |
| Performance | Near native | Overhead from hypervisor |
| OS | Shares host kernel | Full guest OS |
| Isolation | Process-level (namespaces) | Hardware-level |
| Density | 1000s per host | Tens per host |
| Portability | Highly portable | Less portable |

---

## Evolution of Container Technology

```mermaid
timeline
    title Evolution of Containers
    1979 : chroot
         : Unix — Changed root directory for a process
         : First filesystem isolation concept
    2000 : FreeBSD Jails
         : Process + network + filesystem isolation
         : First practical container-like tech
    2004 : Solaris Zones
         : Full OS-level virtualization on Solaris
    2006 : cgroups (Google)
         : Resource limiting / metering for process groups
         : Contributed to Linux kernel
    2008 : LXC (Linux Containers)
         : First container manager using upstream kernel
         : Built on cgroups + namespaces
    2013 : Docker
         : Developer-friendly container platform
         : Dockerfile, images, registry, simple CLI
    2014 : Kubernetes
         : Container orchestration at scale (Google)
    2015 : OCI Standards
         : Open Container Initiative
         : Standardized image + runtime specs
    2017 : containerd / CRI-O
         : Lightweight container runtimes
         : Docker donated containerd to CNCF
```

---

## Linux Kernel Features Behind Containers

Containers are not a single kernel feature — they're built from **five core Linux kernel mechanisms** working together. Without any one of these, containers as we know them wouldn't work.

- **Namespaces** provide **isolation** — they make a container *think* it has its own OS. Each namespace type hides a specific system resource (processes, network, filesystems, etc.) so containers can't see or interfere with each other.
- **cgroups** (Control Groups) provide **resource control** — they set hard limits on how much CPU, memory, and I/O a container can consume, preventing a single container from starving others.
- **UnionFS** (OverlayFS, AUFS) provides the **layered filesystem** — this is what makes Docker images work. Multiple read-only layers are stacked with a thin writable layer on top, enabling efficient storage and fast image builds.
- **Seccomp** (Secure Computing Mode) provides **system call filtering** — it restricts which Linux syscalls a container can make. Docker's default seccomp profile blocks ~44 of the 300+ syscalls (e.g., `reboot`, `mount`, `kexec_load`), reducing the attack surface.
- **AppArmor / SELinux** provide **Mandatory Access Control (MAC)** — they enforce policies on what files, capabilities, and network resources a process can access, adding a security layer *beyond* what namespaces alone provide.

> **How they work together**: Namespaces isolate the view, cgroups limit the resources, UnionFS manages the filesystem, and Seccomp + AppArmor/SELinux lock down the security boundary. Together, they create a lightweight, portable, and secure sandbox — without needing a full guest OS.

```mermaid
graph LR
    subgraph "Linux Kernel Features"
        NS[Namespaces<br/>━━━━━━━━<br/>Isolation]
        CG[cgroups<br/>━━━━━━━━<br/>Resource Limits]
        UFS[UnionFS<br/>━━━━━━━━<br/>Layered Filesystem]
        SC[Seccomp<br/>━━━━━━━━<br/>System Call Filter]
        AA[AppArmor / SELinux<br/>━━━━━━━━<br/>Mandatory Access Control]
    end

    NS -->|PID| P1[Process Isolation]
    NS -->|NET| P2[Network Isolation]
    NS -->|MNT| P3[Mount Isolation]
    NS -->|UTS| P4[Hostname Isolation]
    NS -->|IPC| P5[IPC Isolation]
    NS -->|USER| P6[User Isolation]

    CG -->|cpu| R1[CPU Limits]
    CG -->|memory| R2[Memory Limits]
    CG -->|blkio| R3[Block I/O Limits]
    CG -->|pids| R4[Process Count Limits]
```

### Namespaces — Isolation

Each container gets its own set of namespaces. When a process runs inside a container, namespaces make it appear as if the container is the entire machine — it has PID 1, its own network stack, its own hostname. The host and other containers remain invisible.

| Namespace | Isolates |
|-----------|----------|
| **PID** | Process IDs — container sees its own PID 1 |
| **NET** | Network interfaces, IPs, routes, ports |
| **MNT** | Filesystem mount points |
| **UTS** | Hostname and domain name |
| **IPC** | Inter-process communication (shared memory, semaphores) |
| **USER** | User and group IDs (UID/GID remapping) |

### cgroups — Resource Control

cgroups are organized in a tree hierarchy. Docker creates a cgroup for each container and sets limits on it. If a container exceeds its memory limit, the kernel's OOM (Out-Of-Memory) killer terminates it. CPU limits use a quota/period model — e.g., 50ms of CPU time per 100ms period = 0.5 CPU.

| cgroup subsystem | Controls |
|-----------------|----------|
| **cpu** | CPU shares, quota, period |
| **memory** | Memory limit, swap, OOM behavior |
| **blkio** | Block device I/O throttling |
| **pids** | Maximum number of processes |
| **cpuset** | Pin to specific CPU cores |

---

## LXC, LXD & LXCFS

```mermaid
graph TB
    subgraph "Linux Container Ecosystem"
        LXD["LXD<br/>━━━━━━━━━━<br/>Management Daemon<br/>REST API + CLI<br/>Snapshots, Migration<br/>Clustering"]
        LXC["LXC — liblxc<br/>━━━━━━━━━━<br/>Low-level Runtime<br/>cgroups + namespaces<br/>System Containers"]
        LXCFS["LXCFS<br/>━━━━━━━━━━<br/>FUSE Filesystem<br/>Accurate /proc views<br/>Container-aware metrics"]
        KERNEL["Linux Kernel<br/>cgroups, namespaces<br/>seccomp, AppArmor"]
    end

    LXD -->|uses| LXC
    LXC -->|relies on| KERNEL
    LXCFS -->|intercepts /proc reads| LXC

    style LXD fill:#4a90d9,color:#fff
    style LXC fill:#e8a838,color:#fff
    style LXCFS fill:#50c878,color:#fff
    style KERNEL fill:#c0392b,color:#fff
```

| Technology | Role | Key Detail |
|------------|------|------------|
| **LXC** | Low-level container runtime | Creates *system containers* (full OS userspace) using cgroups + namespaces. Provides `liblxc` C library and CLI tools (`lxc-create`, `lxc-start`). |
| **LXD** | High-level management daemon | REST API on top of LXC. Handles images, snapshots, live migration, clustering. CLI: `lxc launch`, `lxc exec`. Can also manage QEMU/KVM VMs. |
| **LXCFS** | FUSE filesystem overlay | Makes `/proc/meminfo`, `/proc/cpuinfo`, `/proc/stat` reflect **container's cgroup limits** instead of host values. Fixes issues with JVM heap sizing, monitoring tools, etc. |

---

## Docker's Place in the Ecosystem

```mermaid
graph TB
    subgraph "Container Runtimes"
        DOCKER["Docker<br/>Developer-focused<br/>Build + Ship + Run"]
        LXC2["LXC/LXD<br/>System containers<br/>VM-like experience"]
        PODMAN["Podman<br/>Daemonless<br/>Rootless by default"]
        CRIO["CRI-O<br/>Kubernetes-native<br/>Minimal runtime"]
    end

    subgraph "Standards"
        OCI["OCI<br/>Image + Runtime Spec"]
    end

    DOCKER & LXC2 & PODMAN & CRIO --> OCI
```

- **Docker** containers typically run a **single application process**
- **LXC** containers run a **full OS userspace** (init system, multiple processes)
- Both use the same kernel features under the hood

---

[← Back to Index](./README.md) · [Next: Docker Architecture →](./02-container-runtimes/01-docker-architecture.md)
