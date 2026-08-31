# Container Runtimes — Docker, Podman & Comparison

[← Back to Main Index](../README.md) · [Previous: Containers Fundamentals](../01-containers-fundamentals.md)

---

This section covers the architecture, internals, and trade-offs of the two major container runtimes.

## Pages in This Section

| # | Page | Description |
|---|------|-------------|
| 01 | [Docker Architecture & Engine](./01-docker-architecture.md) | dockerd, containerd, runc, process hierarchy, lifecycle, client-server model |
| 02 | [Podman Architecture & Engine](./02-podman-architecture.md) | Daemonless model, conmon, crun, pods, rootless, systemd integration |
| 03 | [Docker vs Podman — Comparison](./03-docker-vs-podman.md) | Side-by-side comparison across architecture, security, networking, orchestration, CLI, and more |

---

## Quick Navigation

```mermaid
graph LR
    A["01 — Docker<br/>Architecture"] --> C["03 — Docker vs Podman<br/>Comparison"]
    B["02 — Podman<br/>Architecture"] --> C

    style A fill:#2496ED,color:#fff
    style B fill:#892ca0,color:#fff
    style C fill:#27ae60,color:#fff
```

> Read Docker and Podman architecture pages first, then use the comparison page as a consolidated reference.

---

[← Back to Main Index](../README.md) · [Next: Orchestration →](../03-orchestration.md)
