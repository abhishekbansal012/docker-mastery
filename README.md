# 🐳 Docker Mastery

A personal knowledge base for mastering Docker — from container fundamentals to orchestration, networking, security, and beyond. Built as a structured study companion for the **Docker Certified Associate (DCA)** exam and as a general Docker reference.

---

## What's Inside

### 📖 Study Notes

In-depth notes covering all DCA exam domains, plus foundational topics and supplementary material.

| # | Topic | Description |
|---|-------|-------------|
| 01 | [Containers Fundamentals](./notes/01-containers-fundamentals.md) | OS-level virtualization, containers vs VMs, Linux kernel features (namespaces, cgroups), LXC/LXD |
| 02 | [Container Runtimes](./notes/02-container-runtimes/README.md) | Docker architecture, Podman architecture, Docker vs Podman comparison |
| 03 | [Orchestration](./notes/03-orchestration/README.md) | Docker Swarm — architecture, services, updates, networking, secrets, stacks, security (25% of exam) |
| 04 | [Image Creation & Management](./notes/04-image-creation-management.md) | Dockerfile, image layers, registries, tagging, multi-stage builds (20% of exam) |
| 05 | [Installation & Configuration](./notes/05-installation-configuration.md) | Docker Engine setup, daemon configuration, storage drivers, logging (15% of exam) |
| 06 | [Networking](./notes/06-networking/README.md) | CNM, bridge, overlay, host, macvlan, ipvlan, DNS, port publishing (15% of exam) |
| 07 | [Security](./notes/07-security.md) | Namespaces, cgroups, seccomp, AppArmor, Docker Content Trust, secrets (15% of exam) |
| 08 | [Storage & Volumes](./notes/08-storage-volumes/README.md) | Volumes, bind mounts, tmpfs, image mounts, storage drivers, CoW (10% of exam) |
| 09 | [Docker Compose](./notes/09-docker-compose.md) | Multi-container applications, YAML configuration, services, networks, volumes |
| 10 | [Commands Cheat Sheet](./notes/10-commands-cheatsheet.md) | Quick reference for commonly used Docker commands |
| 11 | [Exam Tips & Strategy](./notes/11-exam-tips.md) | DCA exam format, study strategy, tips for exam day |

### ⌨️ Commands Reference

Quick-reference command files for everyday Docker usage.

| File | Description |
|------|-------------|
| [Docker Commands](./commands/COMMANDS.md) | Core Docker CLI commands — containers, images, networks |
| [Swarm Commands](./commands/SWARM_COMMANDS.md) | Docker Swarm commands — init, services, scaling |

---

## Repo Structure

```
docker-mastery/
├── notes/                          # Detailed study notes
│   ├── 01-containers-fundamentals.md
│   ├── 02-container-runtimes/      # Docker & Podman deep dives
│   │   ├── 01-docker-architecture.md
│   │   ├── 02-podman-architecture.md
│   │   └── 03-docker-vs-podman.md
│   ├── 03-orchestration/             # Swarm architecture, services, updates, networking, security
│   │   ├── 01-swarm-architecture.md
│   │   ├── 02-services-tasks-scheduling.md
│   │   ├── 03-rolling-updates-rollbacks.md
│   │   ├── 04-swarm-networking.md
│   │   ├── 05-secrets-and-configs.md
│   │   ├── 06-stacks-and-compose.md
│   │   └── 07-swarm-security-and-locking.md
│   ├── 04-image-creation-management.md
│   ├── 05-installation-configuration.md
│   ├── 06-networking/                # CNM, bridge, overlay, DNS, port publishing
│   │   ├── 01-container-network-model.md
│   │   ├── 02-bridge-networks.md
│   │   ├── 03-overlay-networks.md
│   │   ├── 04-host-macvlan-ipvlan-none.md
│   │   ├── 05-dns-and-service-discovery.md
│   │   └── 06-port-publishing-and-traffic-flow.md
│   ├── 07-security.md
│   ├── 08-storage-volumes/          # Volumes, bind mounts, tmpfs, image mounts, CoW
│   │   ├── 01-volumes.md
│   │   ├── 02-bind-mounts.md
│   │   ├── 03-tmpfs-mounts.md
│   │   ├── 04-image-mounts-and-named-pipes.md
│   │   └── 05-storage-drivers-and-cow.md
│   ├── 09-docker-compose.md
│   ├── 10-commands-cheatsheet.md
│   └── 11-exam-tips.md
├── commands/                       # Command quick-references
│   ├── COMMANDS.md
│   └── SWARM_COMMANDS.md
└── README.md
```

---

## DCA Exam at a Glance

| Detail | Info |
|--------|------|
| **Questions** | 55 (13 multiple choice + 42 DOMC) |
| **Duration** | 90 minutes |
| **Cost** | $199 USD |
| **Prerequisite** | 6–12 months hands-on Docker experience |
| **Proctoring** | Remote via Examity |

**Domain Weights:**
Orchestration (25%) · Images & Registry (20%) · Installation & Config (15%) · Networking (15%) · Security (15%) · Storage & Volumes (10%)

---

## Useful Links

- [Docker Official Docs](https://docs.docker.com)
- [Play with Docker](https://labs.play-with-docker.com)
- [DCA Prep Guide — GitHub](https://github.com/Evalle/DCA)
- [Official Study Guide v1.5 (PDF)](https://a.storyblok.com/f/146871/x/2001ce939c/docker-study-guide_v1-5-jan-2025.pdf)
