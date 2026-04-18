# Docker Volumes: Storage and Persistence

## What are Docker Volumes?

Docker volumes are a mechanism for persisting data generated and used by Docker containers. They are independent of the container lifecycle and survive container deletion.

## Volume Types

### 1. Named Volumes
**Definition**: Volumes managed by Docker with a specific name.

**Creation**:
```yaml
volumes:
  database:
    driver: local
```

**Usage**:
```yaml
services:
  db:
    image: mongo:6
    volumes:
      - database:/data/db
```

**Access**:
```bash
docker volume ls
docker volume inspect database
```

**Pros**: ✅ Easy to manage, ✅ Reusable, ✅ Can be shared between containers
**Cons**: ❌ Not directly accessible from host

---

### 2. Bind Mounts
**Definition**: Mount a host directory directly into container.

**Syntax**:
```yaml
volumes:
  - /host/path:/container/path    # Absolute path
  - ./app:/usr/src/app            # Relative path (recommended)
```

**Example**:
```yaml
services:
  app:
    image: node:18
    volumes:
      - ./app:/usr/src/app        # Sync host app to container
```

**Pros**: ✅ Direct host file access, ✅ Good for development
**Cons**: ❌ Platform-specific, ❌ Can have permission issues

---

### 3. Anonymous Volumes
**Definition**: Volumes without a name, created automatically.

```yaml
volumes:
  - /data
  - /logs
```

**Pros**: ✅ Simple for temporary data
**Cons**: ❌ Hard to manage, ❌ Difficult to reuse

---

## Where Volumes Are Stored

### On Linux
```
/var/lib/docker/volumes/
├── database/_data
├── logs/_data
└── cache/_data
```

Access:
```bash
ls -la /var/lib/docker/volumes/database/_data
```

### On macOS
```
Inside Docker Desktop VM (not directly accessible):
~/Library/Containers/com.docker.docker/Data/vms/0/data/Docker.raw
```

**Access volumes on macOS**:
```bash
# Inspect volume details
docker volume inspect database

# Mount in temporary container to explore
docker run -v database:/data alpine ls -la /data

# Copy files from volume
docker cp container-name:/data/file.txt ./
```

### On Windows
```
C:\ProgramData\Docker\volumes\
└── database\_data
```

---

## Protecting Folders from Override

When using bind mounts, specific volumes can protect directories:

```yaml
volumes:
  - ./app:/usr/src/app              # Bind mount (host syncs)
  - /usr/src/app/node_modules       # Protected (container version)
  - /usr/src/app/.next              # Protected (container version)
  - /usr/src/app/dist               # Protected (container version)
```

**How it works**:
1. Host `./app` syncs to `/usr/src/app`
2. Subdirectories `/usr/src/app/node_modules` use container's version
3. Changes in host don't override protected directories
4. Priority: specific volumes > bind mounts > container defaults

**Real example**:
```yaml
services:
  app:
    image: node:18
    working_dir: /app
    volumes:
      - ./:/app                   # Main app code
      - /app/node_modules         # Protect dependencies
      - /app/.next                # Protect build cache
      - /app/dist                 # Protect distribution files
```

---

## Volume Lifecycle

### Create
```bash
docker volume create my-volume

# Or auto-create with compose
docker-compose up
```

### Use
```bash
docker run -v my-volume:/data alpine
```

### Inspect
```bash
docker volume inspect my-volume
```

### Backup
```bash
docker run --rm -v my-volume:/data -v $(pwd):/backup \
  alpine tar czf /backup/volume-backup.tar.gz -C /data .
```

### Restore
```bash
docker run --rm -v my-volume:/data -v $(pwd):/backup \
  alpine tar xzf /backup/volume-backup.tar.gz -C /data
```

### Remove
```bash
docker volume rm my-volume

# Remove unused volumes
docker volume prune
```

---

## Multi-Container Volume Sharing

**Share volume between containers**:
```yaml
version: '3.8'

services:
  app:
    image: node:18
    volumes:
      - shared-data:/data
  
  api:
    image: python:3.12
    volumes:
      - shared-data:/data

volumes:
  shared-data:
```

**How it works**: Both containers have read/write access to `/data`

---

## Volume Driver Options

### Local Driver
```yaml
volumes:
  myvolume:
    driver: local
    driver_opts:
      type: tmpfs
      device: tmpfs
      o: size=100m,uid=1000
```

### NFS Driver (for network storage)
```yaml
volumes:
  nfs-volume:
    driver: local
    driver_opts:
      type: nfs
      o: addr=nfs-server,vers=4,soft,timeo=180,bg,tcp,rw
      device: ":/exports/data"
```

---

## Performance Considerations

### Bind Mounts Performance
- ⚠️ macOS: Slower than Linux (Docker Desktop VM overhead)
- ⚠️ Windows: Slower than Linux (WSL2 overhead)
- ✅ Linux: Native performance

### Optimize for macOS/Windows
```yaml
# Exclude slow directories
volumes:
  - ./app:/app
  - /app/node_modules     # Don't sync
  - /app/.git             # Don't sync
  - /app/.next            # Don't sync
```

### Use tmpfs for temporary data
```yaml
tmpfs:
  - /tmp
  - /run
```

---

## Best Practices

✅ **DO**:
- Use named volumes for persistent data
- Use bind mounts for development
- Protect generated/installed files (node_modules, .venv)
- Back up important volumes
- Use volume labels for organization

❌ **DON'T**:
- Store secrets in volumes
- Use volumes for temporary data (use tmpfs)
- Bind mount large directories
- Mix volume types unnecessarily

---

## Common Patterns

### Development with Hot Reload
```yaml
services:
  app:
    image: node:18
    volumes:
      - ./src:/app/src           # Source code
      - /app/node_modules        # Protected
    command: npm run dev
```

### Database with Persistent Data
```yaml
services:
  db:
    image: mongo:6
    volumes:
      - mongodb-data:/data/db
      
volumes:
  mongodb-data:
```

### Multiple Protected Directories
```yaml
services:
  app:
    volumes:
      - ./app:/app
      - /app/node_modules
      - /app/.next
      - /app/dist
      - /app/__pycache__
      - /app/.pytest_cache
```

---

## Troubleshooting

**Volume not found**:
```bash
docker volume ls                    # Check if exists
docker volume create my-vol         # Create if missing
```

**Permission denied**:
```bash
# Run with correct user
docker run --user 1000:1000 -v myvolume:/data alpine

# Or fix permissions
docker exec container-name chown -R 1000:1000 /data
```

**Volume full**:
```bash
docker volume inspect myvolume      # Check size
docker volume prune                 # Remove unused

# Check container usage
docker exec container-name du -sh /data
```

**Data not persisting**:
- Ensure volume is specified in compose
- Check volume isn't anonymous
- Verify container path correct
