# Image Mounts & Named Pipes

[← Back to Section Index](./README.md) · [← Main Index](../README.md) · [Previous: tmpfs Mounts](./04-tmpfs-mounts.md)

---

## Image Mounts

Image mounts make the filesystem of **another image** available inside a container at a path you choose. The mounted image is always **read-only** and isn't part of the container's own image.

> **Requires the containerd image store** (default since Docker Engine 29.0 for fresh installations). Not available with classic storage drivers.

### Use Cases

- **Debug minimal images** — mount a tool-rich image (like `busybox`) into a stripped-down container to get shell access and utilities
- **Share read-only assets** — datasets, ML models, or static content packaged as an image
- **Keep app images small** — package optional tooling in a separate image and mount only when needed

### Usage

```bash
# Pull the source image first — Docker does NOT auto-pull for image mounts
docker pull busybox:musl

# Mount busybox into an Alpine container at /dbg
docker run -it \
  --mount type=image,source=busybox:musl,target=/dbg \
  alpine sh

# Inside the container, use busybox tools from /dbg
/dbg/bin/ls
/dbg/bin/wget

# Mount only a subdirectory of the image
docker run -it \
  --mount type=image,source=busybox:musl,target=/tools,image-subpath=bin \
  alpine sh
```

### `--mount` Options for Image Mounts

| Option | Description |
|--------|-------------|
| `source`, `src` | Image reference (e.g., `busybox`, `busybox:musl`). Must exist locally. |
| `destination`, `dst`, `target` | Path inside the container to mount at |
| `image-subpath` | Mount a specific subdirectory from the source image instead of its root |

> There is **no `-v` syntax** for image mounts — you must use `--mount type=image`.

---

## Named Pipes

Named pipes can be used for communication between the Docker host and a container. The most common use case is running a third-party tool inside a container and connecting it to the Docker Engine API.

```bash
# Mount the Docker socket (a Unix named pipe) into a container
docker run -v /var/run/docker.sock:/var/run/docker.sock docker:cli docker ps
```

> Named pipes are more commonly used on **Windows** containers but the concept applies to Unix sockets on Linux as well.

---

[← Back to Section Index](./README.md) · [← Main Index](../README.md) · [Previous: tmpfs Mounts](./04-tmpfs-mounts.md) · [Next: Storage Drivers & CoW →](./06-storage-drivers-and-cow.md)
