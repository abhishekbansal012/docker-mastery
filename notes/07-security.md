# Domain 5: Security (15% of Exam)

[← Back to Index](./README.md) · [Previous: Networking](./06-networking.md)

---

## Docker Security — Defense in Depth

```mermaid
graph TB
    subgraph "Security Layers"
        L1["Layer 1: Linux Kernel Security<br/>Namespaces, cgroups, Capabilities,<br/>Seccomp, AppArmor / SELinux"]
        L2["Layer 2: Docker Daemon Security<br/>TLS, rootless mode,<br/>user namespace remapping"]
        L3["Layer 3: Image Security<br/>Content Trust (DCT), Scanning,<br/>Minimal base images, No secrets"]
        L4["Layer 4: Container Runtime Security<br/>Read-only fs, drop capabilities,<br/>no-new-privileges, resource limits"]
        L5["Layer 5: Swarm Security<br/>Mutual TLS, encrypted overlay,<br/>secrets management, autolock"]
    end

    L1 --> L2 --> L3 --> L4 --> L5

    style L1 fill:#c0392b,color:#fff
    style L2 fill:#e74c3c,color:#fff
    style L3 fill:#e67e22,color:#fff
    style L4 fill:#f1c40f,color:#000
    style L5 fill:#27ae60,color:#fff
```

---

## Linux Security Features

```mermaid
graph LR
    subgraph "Namespaces (Isolation)"
        PID[PID: Process isolation]
        NET[NET: Network isolation]
        MNT[MNT: Filesystem isolation]
        UTS[UTS: Hostname isolation]
        IPC[IPC: IPC isolation]
        USR[USER: UID/GID isolation]
    end

    subgraph "cgroups (Resource Limits)"
        CPU[CPU: shares / quota]
        MEM[Memory: limits]
        IO[Block I/O: throttle]
        PIDS[PIDs: process limit]
    end

    subgraph "Capabilities"
        CAP["Fine-grained root privileges<br/>NET_BIND_SERVICE<br/>SYS_ADMIN<br/>CHOWN<br/>NET_RAW<br/>... 40+ capabilities"]
    end
```

### Linux Capabilities

Docker drops many capabilities by default. Key ones:

| Capability | What It Allows |
|-----------|---------------|
| `NET_BIND_SERVICE` | Bind to ports < 1024 |
| `SYS_ADMIN` | Broad admin (mount, sethostname, etc.) — **dangerous** |
| `CHOWN` | Change file ownership |
| `NET_RAW` | Raw sockets (ping, etc.) |
| `SETUID` / `SETGID` | Change UID/GID |
| `MKNOD` | Create device files |

---

## Container Security Best Practices

```bash
# 1. Run as non-root user
docker run --user 1000:1000 nginx

# 2. Read-only filesystem
docker run --read-only nginx

# 3. Drop ALL capabilities, add only what's needed
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE nginx

# 4. Prevent privilege escalation
docker run --security-opt no-new-privileges nginx

# 5. Limit resources (prevent DoS)
docker run --memory=512m --cpus=1.0 nginx

# 6. Use seccomp profile
docker run --security-opt seccomp=/path/to/profile.json nginx

# 7. Limit PIDs (prevent fork bombs)
docker run --pids-limit 100 nginx

# 8. Read-only with writable tmpfs for needed dirs
docker run --read-only --tmpfs /tmp --tmpfs /var/run nginx
```

---

## Docker Content Trust (DCT)

```mermaid
graph LR
    subgraph "Docker Content Trust Flow"
        PUSH["docker push<br/>Signs image with<br/>private key"]
        REG["Registry<br/>Stores image +<br/>signature (Notary)"]
        PULL["docker pull<br/>Verifies signature<br/>with public key"]
    end

    PUSH --> REG --> PULL
```

```bash
# Enable DCT
export DOCKER_CONTENT_TRUST=1

# Push signed image
docker push myregistry/myimage:1.0

# Pull (verifies signature when DCT enabled)
docker pull myregistry/myimage:1.0

# Disable for a single operation
DOCKER_CONTENT_TRUST=0 docker pull unsigned-image
```

- Uses **Notary** for signature management
- Root key + repository key model
- Prevents pulling tampered or unsigned images

---

## Docker Secrets (Swarm Only)

```mermaid
graph TB
    subgraph "Secrets Lifecycle"
        CREATE["1. Create Secret<br/>docker secret create"]
        RAFT["2. Encrypted in Raft Log<br/>AES-256 at rest"]
        ASSIGN["3. Assign to Service<br/>--secret flag"]
        MOUNT["4. Mounted in Container<br/>/run/secrets/&lt;name&gt;<br/>tmpfs (in-memory only)"]
    end

    CREATE --> RAFT --> ASSIGN --> MOUNT

    style CREATE fill:#3498db,color:#fff
    style RAFT fill:#e74c3c,color:#fff
    style ASSIGN fill:#f39c12,color:#fff
    style MOUNT fill:#27ae60,color:#fff
```

```bash
# Create secret from stdin
echo "my_password" | docker secret create db_password -

# Create from file
docker secret create ssl_cert ./server.crt

# List secrets
docker secret ls

# Use in a service
docker service create --name web --secret db_password nginx
# Inside container: cat /run/secrets/db_password

# Inspect metadata (value NOT shown)
docker secret inspect db_password

# Remove
docker secret rm db_password
```

### Secret Properties

- Encrypted at rest in Raft log (AES-256-GCM)
- Transmitted over mTLS connections
- Mounted as tmpfs (RAM-only) at `/run/secrets/<name>`
- Only accessible to services explicitly granted access
- Max size: 500 KB

---

## Docker Configs (Swarm Only)

```bash
# Create a config
docker config create nginx_conf ./nginx.conf

# Use in service
docker service create --name web \
  --config source=nginx_conf,target=/etc/nginx/nginx.conf \
  nginx

# List configs
docker config ls

# Remove
docker config rm nginx_conf
```

### Secrets vs Configs

| | Secrets | Configs |
|---|---------|---------|
| **Encrypted at rest** | ✅ Yes | ❌ No |
| **Mounted as** | tmpfs (in-memory) | Regular file |
| **Max size** | 500 KB | 500 KB |
| **Use for** | Passwords, keys, certificates | Config files, scripts |

---

## Swarm Mutual TLS (mTLS)

```mermaid
sequenceDiagram
    participant New as New Node
    participant CA as Swarm CA (Manager)

    New->>CA: Join request + join token
    CA->>CA: Verify token
    CA->>New: Issue TLS certificate
    Note over New,CA: All communication now<br/>encrypted with mTLS

    Note over CA: Certificate rotation<br/>Default: 90 days<br/>Min: 1 hour
```

```bash
# View current certificate expiry
docker system info | grep "Expiry Duration"

# Change certificate rotation period
docker swarm update --cert-expiry 48h

# Use external CA
docker swarm init --external-ca protocol=cfssl,url=https://ca.example.com
```

### mTLS in Swarm

- **Automatic** — no manual certificate management
- Each node gets a TLS certificate signed by the Swarm CA
- Certificates identify the node's **role** (manager/worker) and **ID**
- All inter-node communication is encrypted
- Certificates rotate automatically (default: 90 days)

---

## Securing the Docker Daemon

```bash
# Protect the socket
# Default: unix:///var/run/docker.sock (local only)

# Remote access with TLS (daemon.json)
# {
#   "tls": true,
#   "tlscacert": "/etc/docker/ca.pem",
#   "tlscert": "/etc/docker/server-cert.pem",
#   "tlskey": "/etc/docker/server-key.pem",
#   "hosts": ["unix:///var/run/docker.sock", "tcp://0.0.0.0:2376"]
# }

# Client connects with:
docker --tlsverify \
  --tlscacert=ca.pem \
  --tlscert=cert.pem \
  --tlskey=key.pem \
  -H=tcp://host:2376 ps
```

> **Warning:** Never expose the Docker socket over TCP without TLS. Socket access = **root access** to the host.

---

## Image Security Scanning

```bash
# Docker Scout (built-in scanning)
docker scout cves myimage:1.0
docker scout quickview myimage:1.0
```

### Image Security Best Practices

1. Use **minimal base images** (alpine, distroless, scratch)
2. **Don't store secrets** in images (use build args or secrets)
3. Use **specific tags**, not `:latest`
4. Scan images for **vulnerabilities** before deploying
5. Use **multi-stage builds** to exclude build tools
6. Set `USER` to run as **non-root**
7. Enable **Docker Content Trust**

---

[← Back to Index](./README.md) · [Next: Storage & Volumes →](./08-storage-volumes.md)
