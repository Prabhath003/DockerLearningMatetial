# Docker Containers: Behavior and Isolation

## What is a Docker Container?

A Docker container is a lightweight, standalone, executable package containing everything needed to run an application. It includes code, runtime, system tools, and libraries.

## Container vs Image

| Aspect | Image | Container |
|--------|-------|-----------|
| **Type** | Blueprint/Template | Running instance |
| **State** | Static | Dynamic |
| **Creation** | Built once | Created from image |
| **Storage** | Small (MB-GB) | Larger (with data) |
| **Lifecycle** | Permanent | Temporary (by default) |

---

## Container Isolation

### Process Isolation
```
Host System                  Container 1              Container 2
├─ PID 1 (init)             ├─ PID 1 (app)           ├─ PID 1 (app)
├─ PID 100 (docker)         ├─ PID 2 (child)         ├─ PID 2 (child)
├─ PID 1000 (app1)          └─ ...                   └─ ...
└─ PID 2000 (app2)
```

**Namespace isolation**: Each container has its own PID namespace
- Container sees its own processes as PID 1, 2, 3...
- Host processes invisible to container
- Host unaffected by container processes

---

### Filesystem Isolation
```
Host Filesystem                Container Filesystem
├─ /etc/                      ├─ /etc/ (isolated)
├─ /home/                     ├─ /app/ (bind mount)
├─ /var/lib/docker/volumes/   └─ /data/ (volume)
└─ ...
```

**What containers can access**:
- Their own filesystems
- Mounted volumes
- Bind mounts (if specified)

**What containers CANNOT access**:
- Host root filesystem
- Other containers' filesystems
- Other containers' data

---

### Network Isolation
```
Container 1                   Container 2
├─ eth0 (172.17.0.2)         ├─ eth0 (172.17.0.3)
├─ localhost:8000            ├─ localhost:3000
└─ Can access Container 2     └─ Can access Container 1
  via 172.17.0.3               via 172.17.0.2
```

**Default network**: Bridge network
- Containers on same network can communicate
- External access requires port mapping
- Network isolated from host by default

---

### User Isolation
```dockerfile
# Good practice: Run as non-root
RUN useradd -m appuser
USER appuser

# Container sees different UID than host
# User "appuser" with UID 1000 in container
# Maps to UID 1000 on host (with security mapping)
```

---

## Container Lifecycle

### States
```
┌──────────┐
│ Created  │ ← docker create
└────┬─────┘
     │
     ↓
┌──────────┐
│ Running  │ ← docker run / docker start
└────┬─────┘
     │
     ├─ Pause ──→ ┌────────┐
     │            │ Paused │
     │            └────────┘
     │
     ↓
┌──────────┐
│  Stopped │ ← docker stop / SIGTERM (15s timeout)
└────┬─────┘
     │
     ↓
┌──────────┐
│  Removed │ ← docker rm
└──────────┘
```

### Signals
- **SIGTERM (15)**: "Terminate gracefully" (15 second timeout)
- **SIGKILL (9)**: "Kill immediately" (no cleanup)
- **SIGSTOP (19)**: "Pause container"

---

## What Happens with `sudo reboot` in Container?

### Scenario
```bash
docker run ubuntu:22.04
root@container# sudo reboot
```

### Result
```
┌─ Container Process Tree ─┐
│ PID 1: bash              │ ← sudo reboot targets this
│ └─ PID 2: reboot         │
└──────────────────────────┘
          ↓
     Container stops
     (reboot kills PID 1)
     
┌─ Host System ─────────────┐
│ (Continues running)        │
│ ❌ NOT rebooted           │
└────────────────────────────┘
```

### Why it doesn't reboot the host
1. **Namespace isolation**: Container has own PID namespace
2. **Kernel is shared**: Containers don't have own kernel
3. **PID 1 is isolated**: Container's PID 1 ≠ host's PID 1
4. **Reboot signal**: Stops container process, not host

### What actually happens
```bash
# Result: Container exits
docker run ubuntu:22.04
bash: reboot command not found
# Or if reboot exists:
# Container stops after reboot signal

docker ps -a
# Shows container as exited
```

---

## Container Memory and Resources

### Memory Usage
```bash
# Check container memory
docker stats container-name

# Set memory limits
docker run -m 512m --memory-swap 1g ubuntu
```

### CPU Usage
```bash
# Limit CPU to 2 cores
docker run --cpus="2" ubuntu

# Limit CPU time
docker run --cpu-period=100000 --cpu-quota=50000 ubuntu
```

### Disk Usage
```bash
docker run --storage-opt size=10G ubuntu
```

---

## Viewing Container Internals

### Inspect container
```bash
# View configuration
docker inspect container-name

# View logs
docker logs container-name

# Follow logs
docker logs -f container-name

# Logs with timestamps
docker logs --timestamps container-name
```

### Execute commands
```bash
# Interactive shell
docker exec -it container-name /bin/bash

# Execute single command
docker exec container-name ls -la /app

# Execute with specific user
docker exec -u 1000 container-name whoami
```

### View running processes
```bash
docker top container-name
```

### Copy files
```bash
# From host to container
docker cp ./file.txt container-name:/app/

# From container to host
docker cp container-name:/app/file.txt ./
```

---

## Container Persistence

### Default (Non-persistent)
```bash
docker run --rm ubuntu
# --rm flag: Remove container after exit
# No data persists
```

### Persistent Container
```bash
docker run -d ubuntu sleep 1000
# Container continues running
# Can restart with: docker restart container-name
# Data persists in container layer
```

### Persistent Data (Recommended)
```bash
docker run -v myvolume:/data ubuntu
# Data persists even after container removal
docker rm container-name
# Data still exists in volume
docker volume ls
```

---

## Container Networking

### Bridge Network (Default)
```yaml
services:
  app:
    image: node:18
    networks:
      - mynetwork
  
  db:
    image: mongo:6
    networks:
      - mynetwork

networks:
  mynetwork:
    driver: bridge
```

**Access**:
- `app` → `db` via hostname `db`
- Both on bridge network
- External access requires port mapping

### Host Network
```bash
docker run --network host ubuntu
# Container uses host's network stack directly
# No isolation
# Warning: Less secure
```

### None Network
```bash
docker run --network none ubuntu
# No network access
# Only loopback interface
```

---

## Container Health Checks

### Define health check
```dockerfile
HEALTHCHECK --interval=10s --timeout=5s --retries=3 \
  CMD curl -f http://localhost:8000/ || exit 1
```

### Usage
```bash
docker inspect --format='{{.State.Health.Status}}' container-name
# Returns: healthy, unhealthy, or starting
```

### In Compose
```yaml
services:
  app:
    image: myapp
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000"]
      interval: 10s
      timeout: 5s
      retries: 3
```

---

## Container Restart Policies

```yaml
services:
  app:
    image: myapp
    restart_policy:
      condition: on-failure    # no, always, on-failure, unless-stopped
      max_retries: 5
      delay: 5s
```

### Conditions
- **no**: Don't automatically restart
- **always**: Always restart if stopped
- **on-failure**: Restart only on non-zero exit
- **unless-stopped**: Always restart unless explicitly stopped

---

## Best Practices

✅ **DO**:
- Run containers as non-root users
- Set resource limits (memory, CPU)
- Use health checks
- Enable restart policies
- Keep containers lightweight
- Use volumes for persistent data

❌ **DON'T**:
- Run containers as root
- Assume data persists without volumes
- Ignore container logs
- Run multiple services in one container
- Forget to set restart policies
- Ignore security best practices

---

## Troubleshooting

**Container exits immediately**:
```bash
docker logs container-name    # Check why it exited
docker run -it image-name bash # Debug interactively
```

**Can't access container port**:
```bash
docker run -p 8000:8000 image-name    # Map port
docker port container-name            # Check mappings
```

**Container out of memory**:
```bash
docker stats                    # Monitor usage
docker run -m 2g image-name    # Increase limit
```

**File changes not persisting**:
```bash
docker run -v myvolume:/data image-name    # Add volume
```
