# Domain 6: Storage & Volumes (10% of Exam)

[← Back to Main Index](../README.md) · [Previous: Security](../07-security.md)

---

Docker supports multiple storage types for persisting data outside a container's writable layer. The official docs at [docs.docker.com/engine/storage](https://docs.docker.com/engine/storage/) list five types.

## Storage Types Overview

```mermaid
graph TB
    subgraph "Docker Storage Mount Types"
        VOL["Volumes (Recommended)<br/>Managed by Docker daemon<br/>/var/lib/docker/volumes/<br/>Best for persistent data"]
        BM["Bind Mounts<br/>Map host path to container<br/>Full path on host<br/>Good for dev, config files"]
        TMP["tmpfs Mounts<br/>In host memory<br/>Ephemeral, fast<br/>Good for sensitive data"]
        IM["Image Mounts<br/>Mount another image FS<br/>Read-only<br/>Requires containerd image store"]
        NP["Named Pipes<br/>Host-container IPC<br/>Common on Windows<br/>Docker Engine API access"]
    end

    subgraph "Container"
        FS["Container Filesystem<br/>(Writable Layer)"]
    end

    VOL -->|mount| FS
    BM -->|mount| FS
    TMP -->|mount| FS
    IM -->|mount read-only| FS
    NP -->|IPC| FS

    style VOL fill:#27ae60,color:#fff
    style BM fill:#f39c12,color:#fff
    style TMP fill:#9b59b6,color:#fff
    style IM fill:#2980b9,color:#fff
    style NP fill:#7f8c8d,color:#fff
```

## Comparison Table

| Feature | Volume | Bind Mount | tmpfs | Image Mount |
|---------|--------|-----------|-------|-------------|
| **Location** | `/var/lib/docker/volumes/` | Anywhere on host | Host memory (RAM) | Another image's filesystem |
| **Managed by Docker** | ✅ Yes | ❌ No | N/A | ✅ Yes (read-only) |
| **Pre-populate from image** | ✅ Yes (if volume is empty) | ❌ No | ❌ No | N/A |
| **Volume drivers** | ✅ (NFS, cloud, etc.) | ❌ No | N/A | N/A |
| **Cross-platform** | ✅ Linux + Mac + Win | ✅ Yes | ❌ Linux only | ✅ Yes |
| **Share between containers** | ✅ Yes | ✅ Yes | ❌ No | ✅ Yes (read-only) |
| **Persists after container removed** | ✅ Yes | ✅ Yes (host files remain) | ❌ No | N/A (image stays) |
| **Writable** | ✅ Yes | ✅ Yes | ✅ Yes | ❌ Read-only only |
| **Performance** | Host-native filesystem speed | Host-native | Fastest (in memory) | Host-native (read-only) |
| **`-v` syntax** | ✅ Yes | ✅ Yes | ❌ No (`--tmpfs` instead) | ❌ No (`--mount` only) |
| **Requires containerd image store** | ❌ No | ❌ No | ❌ No | ✅ Yes |

> **Best practice:** Use **volumes** for persistent data (databases, uploads). Use **bind mounts** for development and config files. Use **tmpfs** for sensitive temporary data. Use **image mounts** for injecting tools or read-only assets from another image.

## Pages in This Section

| # | Page | Description |
|---|------|-------------|
| 01 | [Volumes](./01-volumes.md) | Volume commands, `-v` vs `--mount`, pre-population, subpath, anonymous volumes, Swarm, drivers, backup/restore, Dockerfile VOLUME |
| 02 | [Bind Mounts](./02-bind-mounts.md) | Bind mount usage, behavior, propagation, SELinux labeling |
| 03 | [Volumes vs Bind Mounts](./03-volumes-vs-bind-mounts.md) | Side-by-side comparison across architecture, behavior, security, Compose, and when to choose which |
| 04 | [tmpfs Mounts](./04-tmpfs-mounts.md) | In-memory storage, cgroup limits, swap behavior, mount options |
| 05 | [Image Mounts & Named Pipes](./05-image-mounts-and-named-pipes.md) | Image mounts (containerd image store), named pipes, IPC |
| 06 | [Storage Drivers & Copy-on-Write](./06-storage-drivers-and-cow.md) | CoW strategy, copy_up triggers, overlay2, containerd image store |
| 07 | [Guide: S3 Volumes with rclone](./07-guide-s3-volumes-with-rclone.md) | Step-by-step setup of S3-backed Docker volumes, IAM access, Compose, Swarm |
| 08 | [Guide: GitHub Actions OIDC with AWS](./08-guide-github-actions-oidc-aws.md) | Keyless CI/CD authentication — OIDC federation, trust policies, ECR push, EKS deploy |
| 09 | [Storage Plugins & Drivers Across Clouds](./09-storage-plugins-across-clouds.md) | AWS/GCP/Azure CSI drivers, standalone Docker plugins, vendor-neutral solutions, decision tree |

---

## Quick Navigation

```mermaid
graph LR
    A["01 - Volumes"] --> C["03 - Volumes vs<br/>Bind Mounts"]
    B["02 - Bind Mounts"] --> C
    C --> F["06 - Storage Drivers<br/>and CoW"]
    D["04 - tmpfs Mounts"] --> F
    E["05 - Image Mounts<br/>and Named Pipes"] --> F

    style A fill:#27ae60,color:#fff
    style B fill:#f39c12,color:#fff
    style C fill:#e74c3c,color:#fff
    style D fill:#9b59b6,color:#fff
    style E fill:#2980b9,color:#fff
    style F fill:#575757,color:#fff
```

> Read Volumes and Bind Mounts first, then the comparison page (03) for a consolidated reference. Pages 04–05 cover other mount types. Page 06 explains how Docker manages image layers internally.

## Quick Reference

| Mount Type | Persistent | Writable | Shared | Best For |
|-----------|-----------|----------|--------|----------|
| **Volume** | ✅ | ✅ | ✅ | Databases, uploads, persistent state |
| **Bind Mount** | ✅ | ✅ | ✅ | Dev source code, config files, DNS |
| **tmpfs** | ❌ | ✅ | ❌ | Secrets, session data, caches |
| **Image Mount** | N/A | ❌ | ✅ | Debug tools, read-only assets |
| **Named Pipe** | N/A | N/A | N/A | Host ↔ container communication |
| **Container Layer** | ❌ | ✅ | ❌ | Ephemeral scratch data |

```mermaid
graph TB
    Q{"What kind of data?"}
    Q -->|"Persistent app data<br/>(databases, uploads)"| VOL["Use Volume"]
    Q -->|"Source code in dev<br/>Config files"| BM["Use Bind Mount"]
    Q -->|"Sensitive temp data<br/>Secrets, sessions"| TMP["Use tmpfs"]
    Q -->|"Read-only tools/assets<br/>from another image"| IM["Use Image Mount"]
    Q -->|"Host-container IPC"| NP["Use Named Pipe"]
    Q -->|"Ephemeral, disposable"| CL["Use Container Layer"]

    style VOL fill:#27ae60,color:#fff
    style BM fill:#f39c12,color:#fff
    style TMP fill:#9b59b6,color:#fff
    style IM fill:#2980b9,color:#fff
    style NP fill:#7f8c8d,color:#fff
    style CL fill:#95a5a6,color:#fff
```

---

**Sources**: [Docker Storage Overview](https://docs.docker.com/engine/storage/), [Volumes](https://docs.docker.com/engine/storage/volumes/), [Bind Mounts](https://docs.docker.com/engine/storage/bind-mounts/), [tmpfs](https://docs.docker.com/engine/storage/tmpfs/), [Image Mounts](https://docs.docker.com/engine/storage/image-mounts/), [Storage Drivers](https://docs.docker.com/engine/storage/drivers/). Content was rephrased for compliance with licensing restrictions.

---

[← Back to Main Index](../README.md) · [Next: Docker Compose →](../09-docker-compose.md)
