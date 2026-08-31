# Domain 6: Storage & Volumes (10% of Exam)

[← Back to Index](./README.md) · [Previous: Security](./07-security.md)

---

## Storage Types Overview

```mermaid
graph TB
    subgraph "Docker Storage Options"
        VOL["Volumes ⭐<br/>━━━━━━━━━━━<br/>Managed by Docker<br/>/var/lib/docker/volumes/<br/>Best for persistent data"]
        BM["Bind Mounts<br/>━━━━━━━━━━━<br/>Map host path to container<br/>Full path on host<br/>Good for dev, config files"]
        TMP["tmpfs Mounts<br/>━━━━━━━━━━━<br/>In-memory only<br/>Never written to disk<br/>Good for sensitive data"]
    end

    subgraph "Container"
        FS["Container Filesystem<br/>(Writable Layer)"]
    end

    VOL -->|mount| FS
    BM -->|mount| FS
    TMP -->|mount| FS

    style VOL fill:#27ae60,color:#fff
    style BM fill:#f39c12,color:#fff
    style TMP fill:#9b59b6,color:#fff
```

---

## Volumes vs Bind Mounts vs tmpfs

| Feature | Volume | Bind Mount | tmpfs |
|---------|--------|-----------|-------|
| **Location** | `/var/lib/docker/volumes/` | Anywhere on host | Memory (RAM) |
| **Managed by Docker** | ✅ Yes | ❌ No | N/A |
| **Pre-populate from image** | ✅ Yes | ❌ No | ❌ No |
| **Volume drivers** | ✅ (NFS, cloud, etc.) | ❌ No | N/A |
| **Cross-platform** | ✅ Linux + Mac + Win | ✅ Yes | ❌ Linux only |
| **Share between containers** | ✅ Yes | ✅ Yes | ❌ No |
| **Persists after container removed** | ✅ Yes | ✅ Yes | ❌ No |
| **Performance** | Good | Host-native | Fastest |

> **Best practice:** Use **volumes** for persistent data. Use **bind mounts** for development and config files. Use **tmpfs** for sensitive temporary data.

---

## Volume Commands

```bash
# Create a named volume
docker volume create my-data

# List volumes
docker volume ls

# Inspect a volume
docker volume inspect my-data

# Remove a volume
docker volume rm my-data

# Prune all unused volumes
docker volume prune
```

---

## Using Volumes

### `-v` syntax (short form)

```bash
# Named volume
docker run -d --name db -v my-data:/var/lib/mysql mysql:8

# Anonymous volume
docker run -d -v /var/lib/mysql mysql:8
```

### `--mount` syntax (explicit, preferred)

```bash
# Named volume
docker run -d --name db \
  --mount type=volume,source=my-data,target=/var/lib/mysql \
  mysql:8

# Read-only volume
docker run -d \
  --mount type=volume,source=my-data,target=/data,readonly \
  nginx
```

### `-v` vs `--mount`

| | `-v` / `--volume` | `--mount` |
|---|-------------------|-----------|
| **Syntax** | Compact | Verbose, explicit |
| **Missing host dir** | Auto-creates | **Error** (bind mount) |
| **Recommended** | Quick use | Production, clarity |

> **Exam tip:** `--mount` is preferred for its explicit syntax and because it **errors on missing source** for bind mounts instead of silently creating a directory.

---

## Bind Mounts

```bash
# Bind mount with -v
docker run -d -v /host/path:/container/path nginx

# Bind mount with --mount (preferred)
docker run -d \
  --mount type=bind,source=/host/path,target=/container/path \
  nginx

# Read-only bind mount
docker run -d \
  --mount type=bind,source=/host/path,target=/container/path,readonly \
  nginx
```

### Bind Mount Behavior

```mermaid
graph LR
    subgraph "Host Filesystem"
        HP["/home/user/app<br/>(source)"]
    end
    subgraph "Container"
        CP["/app<br/>(target)"]
    end

    HP <-->|"Bind Mount<br/>Changes visible both ways"| CP
```

- Changes on host are **immediately visible** in container and vice versa
- If target dir in container has existing content, it gets **obscured** (not deleted) by the bind mount
- Host path must be an **absolute path**

---

## tmpfs Mounts

```bash
# tmpfs mount
docker run -d \
  --mount type=tmpfs,target=/app/temp,tmpfs-size=100m \
  myapp

# Short form
docker run -d --tmpfs /app/temp myapp
```

- Stored in **host memory only** — never written to disk
- Removed when container stops
- Good for secrets, session data, temporary files
- **Linux only**

---

## Copy-on-Write (CoW) Strategy

```mermaid
graph TB
    subgraph "Read Operation"
        R1["Image Layer<br/>config.txt (original)"]
        R2["Container reads config.txt<br/>→ Reads from image layer ✅"]
    end

    subgraph "First Write Operation"
        W1["Image Layer<br/>config.txt (original)"]
        W2["Container Layer<br/>config.txt (COPY created)"]
        W1 -->|"Copy-Up on first write"| W2
    end

    subgraph "Subsequent Operations"
        S1["Image Layer<br/>config.txt (untouched)"]
        S2["Container Layer<br/>config.txt (modified)"]
        S2 -->|"All reads/writes<br/>use this copy"| S2
    end
```

1. Container **reads** from image layers (no copy)
2. On **first write**, the file is **copied up** to the writable layer
3. All subsequent reads/writes use the **container's copy**
4. Original image layer stays **unchanged**

> CoW is why container writes are slower than volume writes. For write-heavy workloads, always use **volumes**.

---

## Volumes in Swarm

```bash
# Service with named volume
docker service create --name db \
  --mount type=volume,source=db-data,target=/var/lib/mysql \
  mysql:8
```

### Volume Drivers for Shared Storage

```bash
# NFS volume
docker volume create --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw \
  --opt device=:/path/to/dir \
  nfs-volume

# Use third-party drivers: REX-Ray, Portworx, etc.
```

> **Swarm caveat:** Volumes are **local to each node**. If a task moves to a different node, the data won't follow unless you use a shared storage driver (NFS, cloud volumes, etc.).

---

## VOLUME Instruction in Dockerfile

```dockerfile
# Creates an anonymous volume at the specified path
VOLUME /var/lib/mysql
```

- Creates a mount point and marks it as externally mounted
- If no volume is explicitly provided at `docker run`, Docker creates an **anonymous volume**
- Cannot specify the source (host path or named volume) in Dockerfile — that's done at runtime

---

## Storage Summary

```mermaid
graph TB
    Q{"What kind of data?"}
    Q -->|"Persistent app data<br/>(databases, uploads)"| VOL["Use Volume ⭐"]
    Q -->|"Source code in dev<br/>Config files"| BM["Use Bind Mount"]
    Q -->|"Sensitive temp data<br/>Secrets, sessions"| TMP["Use tmpfs"]
    Q -->|"Ephemeral, disposable"| CL["Use Container Layer"]

    style VOL fill:#27ae60,color:#fff
    style BM fill:#f39c12,color:#fff
    style TMP fill:#9b59b6,color:#fff
    style CL fill:#95a5a6,color:#fff
```

---

[← Back to Index](./README.md) · [Next: Docker Compose →](./09-docker-compose.md)
