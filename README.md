# Hytale Server (Docker Compose / Dockge)

This repository provides a `docker-compose.yml` that runs a Hytale server in a container and (optionally) includes a small “restarter” sidecar to **restart the server on a schedule** (e.g., twice per day).

> **Shoutout / Credits:** The server container image is based on the awesome project by **deinfreu**:  
> https://github.com/deinfreu/hytale-server-container  
> Thanks for maintaining and sharing it! 🙌

---

## Features

- Runs the Hytale server as a Docker container
- Persists server data via a local `./data` volume
- Mounts the host `machine-id` read-only (important for stable identity)
- Optional scheduled restarts via a sidecar container (e.g., 00:00/12:00 or every 12 hours)
- Container memory limits and JVM tuning via `JAVA_ARGS`

---

## Requirements

- Docker Engine + Docker Compose plugin
- Optional: Dockge for deploying/managing the stack
- Enough host RAM (depends on world size / player count)

---

## Quick Start

1. Clone the repository or place the `docker-compose.yml` into a directory
2. Create the data directory:
   ```bash
   mkdir -p data
   ```
3. Fix permissions (see **Important: Permissions** below)
4. Start the stack:
   ```bash
   docker compose up -d
   ```

In Dockge: add stack → paste `docker-compose.yml` → **Redeploy**.

---

## Ports

- UDP `5520` is mapped to the host:
  - `5520:5520/udp`

Adjust the mapping if needed.

---

## Persistence (Volumes)

- `./data:/home/container`  
  Stores server data persistently in the `data` directory.
- `/etc/machine-id:/etc/machine-id:ro`  
  Mounts the host machine-id file (read-only).

---

## Important: Permissions (data folder & machine-id)

### Data directory permissions

If the container cannot write to `./data`, you may run into problems (e.g., missing/inconsistent files or identity/machine-id related issues).

**Recommendation:** Ensure the `data` directory is writable for the container. Depending on your host/image setup, this can look like:

```bash
sudo chown -R 1000:1000 ./data
sudo chmod -R u+rwX,go+rX ./data
```

> Note: The exact UID/GID can differ depending on image/host. If unsure, check the user ID inside the container or consult the image documentation/issues.

### machine-id

The file `/etc/machine-id` is mounted **read-only**. Make sure it exists on the host:

```bash
ls -l /etc/machine-id
```

---

## RAM / Java Heap (avoiding OutOfMemoryError)

A common error is:

```text
java.lang.OutOfMemoryError: Java heap space
```

This means: **the JVM heap is too small**, even if the container is allowed to use more RAM overall.

### Recommended settings

- Increase container RAM (`mem_limit`)
- Set JVM heap via `JAVA_ARGS`

Example (with a 10G container limit):

```yaml
mem_limit: 10G
environment:
  JAVA_ARGS: "-Xms6G -Xmx8G"
```

**Why not set `-Xmx` equal to `mem_limit`?**  
The JVM also needs memory for metaspace, threads, native buffers, etc. Always leave headroom.

### Verify that JVM flags are applied

```bash
docker exec -it hytale-server sh -lc "ps -o args= -C java"
```

You should see `-Xms... -Xmx...` in the output.

---

## Optional: Scheduled restarts (sidecar “restarter”)

Docker Compose does include a built-in “restart every 12 hours” timer. If you don't need it, just remove the hytale-restarter service.

### Security note

The restarter mounts:

```text
/var/run/docker.sock:/var/run/docker.sock
```

This gives the sidecar **very powerful access** to Docker (effectively root-equivalent at the Docker level). Use this only if you trust the stack.

### Test the restart mechanism

Trigger a restart once manually:

```bash
docker exec -it hytale-restarter docker restart --time 30 hytale-server
```

---

## Configuration (Environment variables)

Common variables in the `hytale` service:

- `SERVER_IP` / `SERVER_PORT` – bind address / port
- `PROD` / `DEBUG` – mode/logging
- `TZ` – time zone
- `JAVA_ARGS` – additional JVM flags (heap, GC tuning, etc.)

For more options, refer to the image documentation (deinfreu project):  
https://deinfreu.github.io/hytale-server-container/technical/docker/

---

## Troubleshooting

### Changes don’t take effect

In Dockge, “Restart” is often not enough — use **Redeploy** (recreate).

CLI:

```bash
docker compose up -d --force-recreate
```

### Server crashes or is unstable

- Check logs:
  ```bash
  docker logs -f hytale-server
  ```
- Increase RAM/heap (see **RAM / Java Heap**)
- If available, reduce settings (e.g., player limit/view distance) via environment variables

---

## Credits

- Compose setup: this repository
- Hytale server container image: **deinfreu**  
  https://github.com/deinfreu/hytale-server-container

