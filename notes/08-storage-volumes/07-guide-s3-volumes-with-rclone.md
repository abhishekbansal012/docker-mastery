# Guide: S3-Backed Docker Volumes with rclone

[← Back to Section Index](./README.md) · [← Main Index](../README.md) · [Volumes](./01-volumes.md)

---

## Overview

This guide walks through setting up Amazon S3 as a backend for Docker volumes using the [rclone Docker volume plugin](https://rclone.org/docker/). This lets containers read and write to S3 as if it were a local directory — no application code changes needed.

```mermaid
graph LR
    subgraph "Container"
        APP["App writes to /data"]
    end

    subgraph "Docker"
        PLUGIN["rclone volume plugin<br/>(manages FUSE mount)"]
    end

    subgraph "AWS"
        S3["S3 Bucket<br/>(private, encrypted)"]
    end

    APP -->|"local filesystem ops"| PLUGIN
    PLUGIN -->|"S3 API calls<br/>(authenticated)"| S3

    style PLUGIN fill:#575757,color:#fff
    style S3 fill:#FF9900,color:#fff
```

> The S3 bucket does **NOT** need to be public. rclone authenticates using IAM credentials, instance roles, or environment variables — just like the AWS CLI.

---

## Prerequisites

- Docker Engine 19.03.15 or later
- Linux host (the managed plugin doesn't work on macOS/Windows natively — Docker Desktop runs in a VM)
- FUSE installed on the host
- An AWS account with an S3 bucket
- IAM credentials with access to the bucket

---

## Step 1: Install FUSE on the Host

rclone mounts remote storage using FUSE (Filesystem in Userspace). It must be installed on the Docker host:

```bash
# Ubuntu / Debian
sudo apt-get update && sudo apt-get -y install fuse3

# CentOS / RHEL / Amazon Linux
sudo yum install -y fuse fuse3

# Verify
fusermount3 --version
```

---

## Step 2: Create Plugin Directories

The rclone plugin requires two directories to exist before installation. The plugin **will not** create them automatically:

```bash
# Config directory — holds rclone.conf
sudo mkdir -p /var/lib/docker-plugins/rclone/config

# Cache directory — holds plugin state and VFS caches
sudo mkdir -p /var/lib/docker-plugins/rclone/cache
```

---

## Step 3: Create an IAM Policy and Credentials

Create an IAM policy scoped to your specific bucket. Do **not** use a wildcard `*` for the resource.

### IAM Policy (least privilege)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RcloneDockerVolumeAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::my-docker-volumes",
        "arn:aws:s3:::my-docker-volumes/*"
      ]
    }
  ]
}
```

### Access Methods

```mermaid
graph TB
    subgraph "Option A: IAM User Credentials"
        A1["Create IAM user"] --> A2["Attach S3 policy"]
        A2 --> A3["Generate access key + secret"]
        A3 --> A4["Put in rclone.conf"]
    end

    subgraph "Option B: EC2 Instance Role (Recommended)"
        B1["Create IAM role"] --> B2["Attach S3 policy"]
        B2 --> B3["Attach role to EC2 instance"]
        B3 --> B4["rclone auto-detects via<br/>instance metadata service"]
    end

    subgraph "Option C: Environment Variables"
        C1["Set AWS_ACCESS_KEY_ID"] --> C2["Set AWS_SECRET_ACCESS_KEY"]
        C2 --> C3["rclone reads from env"]
    end

    style A4 fill:#f39c12,color:#fff
    style B4 fill:#27ae60,color:#fff
    style C3 fill:#2980b9,color:#fff
```

| Method | Security | Best For |
|--------|----------|----------|
| **IAM User + access keys** | Keys on disk — rotate regularly | Dev machines, quick testing |
| **EC2 Instance Role** (recommended) | No keys on disk — auto-rotated | Production EC2 / ECS hosts |
| **Environment variables** | Keys in memory only | CI/CD pipelines |

---

## Step 4: Create the rclone Configuration

### Option A: Using a config file (IAM User credentials)

Create the rclone configuration file with your S3 remote:

```bash
sudo tee /var/lib/docker-plugins/rclone/config/rclone.conf > /dev/null << 'EOF'
[my-s3]
type = s3
provider = AWS
access_key_id = AKIAIOSFODNN7EXAMPLE
secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
region = us-east-1
EOF

# Lock down permissions — this file contains secrets
sudo chmod 600 /var/lib/docker-plugins/rclone/config/rclone.conf
```

### Option B: Using EC2 Instance Role (no config file needed)

If your Docker host is an EC2 instance with an IAM role attached, create a minimal config — rclone will auto-discover credentials from the instance metadata service:

```bash
sudo tee /var/lib/docker-plugins/rclone/config/rclone.conf > /dev/null << 'EOF'
[my-s3]
type = s3
provider = AWS
env_auth = true
region = us-east-1
EOF
```

> `env_auth = true` tells rclone to use the EC2 instance role, ECS task role, or environment variables for authentication. No access keys needed.

### Option C: Config-less (inline options at volume creation)

You can skip the config file entirely and pass all options when creating the volume. See Step 6 for examples.

---

## Step 5: Install the rclone Docker Volume Plugin

```bash
# Install for amd64 (most common)
docker plugin install rclone/docker-volume-rclone:amd64 \
  args="-v" \
  --alias rclone \
  --grant-all-permissions

# Verify installation
docker plugin ls
```

> The `args="-v"` flag sets verbosity. Use `args="-vv"` for debug output. The args value cannot be empty due to a Docker bug — always pass at least `-v`.

### Other architectures

```bash
# ARM64 (e.g., AWS Graviton, Raspberry Pi 4)
docker plugin install rclone/docker-volume-rclone:arm64 \
  args="-v" --alias rclone --grant-all-permissions

# ARM v7
docker plugin install rclone/docker-volume-rclone:arm-v7 \
  args="-v" --alias rclone --grant-all-permissions
```

---

## Step 6: Create an S3-Backed Volume

### Method 1: Using the config file remote

```bash
# Create volume pointing to a bucket path via the config remote
docker volume create my-s3-vol \
  -d rclone \
  -o remote=my-s3:my-docker-volumes/app-data \
  -o allow-other=true \
  -o vfs-cache-mode=full

# Verify
docker volume inspect my-s3-vol
```

### Method 2: Config-less (inline options)

No rclone.conf needed — pass everything via `-o` flags:

```bash
docker volume create my-s3-vol \
  -d rclone \
  -o type=s3 \
  -o s3-provider=AWS \
  -o s3-access-key-id=AKIAIOSFODNN7EXAMPLE \
  -o s3-secret-access-key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY \
  -o s3-region=us-east-1 \
  -o path=my-docker-volumes/app-data \
  -o allow-other=true \
  -o vfs-cache-mode=full
```

### Method 3: Config-less with EC2 Instance Role

```bash
docker volume create my-s3-vol \
  -d rclone \
  -o type=s3 \
  -o s3-provider=AWS \
  -o s3-env-auth=true \
  -o s3-region=us-east-1 \
  -o path=my-docker-volumes/app-data \
  -o allow-other=true \
  -o vfs-cache-mode=full
```

### Key Volume Options Explained

| Option | Value | Purpose |
|--------|-------|---------|
| `remote` | `my-s3:bucket/path` | References a remote from rclone.conf with a bucket path |
| `type` | `s3` | Backend type (for config-less volumes) |
| `s3-provider` | `AWS` | S3 provider (AWS, Minio, Wasabi, GCS, etc.) |
| `s3-env-auth` | `true` | Use EC2 instance role / environment variables |
| `s3-region` | `us-east-1` | AWS region of the bucket |
| `path` | `bucket/prefix` | Bucket name and optional key prefix |
| `allow-other` | `true` | Allow non-root users (containers) to access the mount |
| `vfs-cache-mode` | `full` | Cache files locally for read/write. Required for most apps. |

---

## Step 7: Use the Volume in a Container

```bash
# Mount into a container
docker run -d \
  --name myapp \
  -v my-s3-vol:/data \
  myapp-image

# Or with --mount (required if passing volume-driver inline)
docker run -d \
  --name myapp \
  --mount type=volume,source=my-s3-vol,target=/data \
  myapp-image
```

### Test it works

```bash
# Start a test container
docker run --rm -it -v my-s3-vol:/mnt alpine sh

# Inside the container:
echo "Hello from Docker" > /mnt/test.txt
ls -la /mnt/
cat /mnt/test.txt
exit

# Verify in AWS — the file should appear in your S3 bucket
aws s3 ls s3://my-docker-volumes/app-data/
```

---

## Step 8: Use with Docker Compose

```yaml
services:
  myapp:
    image: myapp-image
    volumes:
      - s3-data:/data

volumes:
  s3-data:
    driver: rclone
    driver_opts:
      remote: 'my-s3:my-docker-volumes/app-data'
      allow_other: 'true'
      vfs_cache_mode: full
      poll_interval: 0
```

> **YAML note:** Boolean values like `true` and `false` must be quoted in Compose files because YAML treats them as reserved words.

### Config-less Compose (no rclone.conf)

```yaml
volumes:
  s3-data:
    driver: rclone
    driver_opts:
      type: s3
      s3_provider: AWS
      s3_env_auth: 'true'
      s3_region: us-east-1
      path: my-docker-volumes/app-data
      allow_other: 'true'
      vfs_cache_mode: full
```

> **Compose uses underscores** (`_`) instead of dashes (`-`) in option names.

---

## Step 9: Use with Docker Swarm

On Swarm, install the rclone plugin and create the config on **every node** in the cluster:

```bash
# On EVERY swarm node:
sudo mkdir -p /var/lib/docker-plugins/rclone/config
sudo mkdir -p /var/lib/docker-plugins/rclone/cache
# Copy the same rclone.conf to each node
docker plugin install rclone/docker-volume-rclone:amd64 \
  args="-v" --alias rclone --grant-all-permissions
```

Then create the service:

```yaml
# swarm-stack.yml
version: '3.8'
services:
  myapp:
    image: myapp-image
    deploy:
      replicas: 3
    volumes:
      - s3-shared:/data

volumes:
  s3-shared:
    driver: rclone
    driver_opts:
      remote: 'my-s3:my-docker-volumes/shared-data'
      allow_other: 'true'
      vfs_cache_mode: full
```

```bash
docker stack deploy -c swarm-stack.yml mystack
```

> All 3 replicas on different nodes will mount the same S3 path. This is the main advantage over local volumes in Swarm — **data follows the task across nodes**.

---

## VFS Cache Modes

The `vfs-cache-mode` option controls how rclone caches files locally. This is critical for performance and compatibility:

| Mode | Reads | Writes | Use Case |
|------|-------|--------|----------|
| `off` | Direct from S3 (slow) | ❌ Not supported | Read-only mounts |
| `minimal` | Direct from S3 | Cached locally, then uploaded | Small writes, mostly reads |
| `writes` | Direct from S3 | Cached locally, then uploaded | Apps that write and immediately close |
| `full` (recommended) | Cached locally | Cached locally, then uploaded | Most apps. Required for random I/O, seeks, databases |

```bash
# Set cache mode at volume creation
docker volume create my-s3-vol -d rclone \
  -o remote=my-s3:my-bucket \
  -o vfs-cache-mode=full
```

> **Warning:** `vfs-cache-mode=full` caches files on the Docker host's local disk at `/var/lib/docker-plugins/rclone/cache/`. Ensure sufficient disk space for your working set.

---

## Access Management Summary

```mermaid
graph TB
    subgraph "Authentication"
        AUTH["How rclone authenticates to S3"]
        AUTH --> KEYS["IAM Access Keys<br/>(in rclone.conf or inline)"]
        AUTH --> ROLE["EC2 Instance Role<br/>(env_auth = true)"]
        AUTH --> ENV["Environment Variables<br/>(AWS_ACCESS_KEY_ID)"]
    end

    subgraph "Authorization"
        AUTHZ["What rclone can access"]
        AUTHZ --> POLICY["IAM Policy<br/>(scoped to specific bucket)"]
        AUTHZ --> BUCKET["S3 Bucket Policy<br/>(optional, for cross-account)"]
        AUTHZ --> ENCRYPT["S3 Encryption<br/>(SSE-S3, SSE-KMS)"]
    end

    style ROLE fill:#27ae60,color:#fff
    style POLICY fill:#2980b9,color:#fff
```

| Layer | Controls | Recommendation |
|-------|----------|---------------|
| **Authentication** | Who is making the request | Use EC2 Instance Roles in production (no keys on disk) |
| **IAM Policy** | What actions are allowed on which resources | Scope to specific bucket + prefix, least privilege |
| **Bucket Policy** | Who can access the bucket (optional) | Use for cross-account access or additional restrictions |
| **Encryption** | Data at rest protection | Enable SSE-S3 (default) or SSE-KMS for compliance |
| **VPC Endpoint** | Network path to S3 | Use S3 Gateway Endpoint to keep traffic off public internet |

---

## Troubleshooting

```bash
# Check plugin is running
docker plugin ls

# View plugin logs (part of Docker daemon log)
sudo journalctl --unit docker | grep rclone

# Increase verbosity
docker plugin disable rclone
docker plugin set rclone RCLONE_VERBOSE=2
docker plugin enable rclone

# Inspect a volume
docker volume inspect my-s3-vol

# Test S3 connectivity from the host (without Docker)
rclone ls my-s3:my-docker-volumes/ --config /var/lib/docker-plugins/rclone/config/rclone.conf

# Can't remove volume? Find containers using it
docker ps -a --filter volume=my-s3-vol

# Plugin won't disable? Find and remove all volumes first
docker volume ls --filter driver=rclone
```

### Common Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| Plugin install fails with "no such file or directory" | Missing plugin directories | Create `/var/lib/docker-plugins/rclone/config` and `cache` |
| "transport endpoint is not connected" | FUSE mount crashed | Restart the plugin: `docker plugin disable rclone && docker plugin enable rclone` |
| Access Denied from S3 | IAM policy too restrictive or wrong credentials | Check IAM policy includes `s3:ListBucket` on the bucket ARN (not just `/*`) |
| Slow reads/writes | `vfs-cache-mode=off` or `minimal` | Set `vfs-cache-mode=full` for most workloads |
| Volume options not updating | Docker ignores `volume create` on existing volumes | Remove the volume first, then recreate with new options |
| Plugin works on one Swarm node but not others | Plugin/config not installed on all nodes | Install plugin and copy rclone.conf to every Swarm node |

---

## Cleanup

```bash
# Remove volumes
docker volume rm my-s3-vol

# Remove the plugin
docker plugin disable rclone
docker plugin rm rclone

# Remove directories
sudo rm -rf /var/lib/docker-plugins/rclone
```

---

**Sources**: [rclone Docker Volume Plugin docs](https://rclone.org/docker/), [Docker Volumes — official docs](https://docs.docker.com/engine/storage/volumes/), [rclone S3 backend docs](https://rclone.org/s3/). Content was rephrased for compliance with licensing restrictions.

---

[← Back to Section Index](./README.md) · [← Main Index](../README.md) · [Volumes](./01-volumes.md)
