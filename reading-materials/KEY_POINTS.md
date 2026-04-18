# Docker Learning - Key Points

## Essential Concepts

### 1. Docker Volumes
- **Definition**: Persistent data storage independent of container lifecycle
- **Location on macOS**: Inside Docker Desktop VM at `/var/lib/docker/volumes/`
- **Access**: Use `docker volume inspect` and `docker exec` to interact
- **Types**: Named volumes, anonymous volumes, bind mounts

### 2. Container Isolation
- Containers share the host kernel but have isolated processes
- `sudo reboot` inside a container only stops that container, not the host
- Namespace isolation prevents containers from affecting each other
- Process ID 1 in container is isolated from host processes

### 3. Bind Mounts & File Protection
- Main bind mount: `./app:/usr/src/app` syncs host to container
- Protected folders: `/usr/src/app/node_modules` prevents host override
- Multiple volumes can be specified to protect multiple directories
- Order matters: specific volumes take priority over general bind mounts

### 4. Docker Compose Networking
- **`links` (deprecated)**: Legacy way to enable container communication
- **`depends_on`** (modern): Controls startup order and service readiness
- Default network: All services in same compose file auto-connect by service name
- No need for `links` in modern Docker—just use service name as hostname

### 5. Environment Variables
- **ARG**: Build-time only variables in Dockerfile
- **ENV**: Runtime variables available in container
- **.env file**: External configuration file loaded by Docker Compose
- **Priority**: CLI flags > environment section > env_file > Dockerfile defaults

### 6. AI Agent Sandboxing
- Per-user containers provide isolation and security
- Resource limits prevent runaway processes
- Volumes protect host filesystem
- Queue-based execution enables fair resource allocation at scale

### 7. Docker Image Privacy
- **Docker Hub free**: PUBLIC by default (anyone can pull)
- **Docker Hub paid**: Can be made private
- **Other registries** (GitHub, AWS, Azure): PRIVATE by default
- Never include secrets in Docker images

### 8. Scaling Strategies
- **< 50 users**: Docker Compose on single machine
- **50-500 users**: Docker Swarm or single Kubernetes cluster
- **500-5k users**: Multi-node Kubernetes with namespaces
- **5k+ users**: Distributed Kubernetes with multi-region support

---

## Quick Reference

### Create and manage volumes
```bash
docker volume ls              # List volumes
docker volume inspect name    # Inspect volume
docker volume rm name         # Remove volume
```

### Execute in container
```bash
docker exec -it container-name bash        # Interactive shell
docker exec container-name command         # Execute command
```

### Docker Compose basics
```bash
docker-compose up                          # Start services
docker-compose down                        # Stop services
docker-compose logs service-name           # View logs
docker-compose ps                          # Show status
```

### Protect multiple folders in bind mount
```yaml
volumes:
  - ./app:/usr/src/app           # Synced
  - /usr/src/app/node_modules    # Protected
  - /usr/src/app/.next           # Protected
  - /usr/src/app/dist            # Protected
```

### Use env_file in Compose
```yaml
services:
  app:
    env_file: .env               # Load all .env variables
    environment:
      - CUSTOM_VAR=value         # Override or add
```

---

## Best Practices Checklist

- ✅ Use named volumes for data that persists across container restarts
- ✅ Use bind mounts for development (hot reload)
- ✅ Protect `node_modules`, `.venv`, and build artifacts from host override
- ✅ Use `depends_on` with `condition: service_healthy` for reliable startup order
- ✅ Store secrets in `.env` files, never hardcode in images
- ✅ Use private registries (GitHub, AWS, Azure) instead of public Docker Hub
- ✅ Set resource limits for all containers (CPU, memory)
- ✅ Use per-user containers for AI agent sandboxes
- ✅ Validate and sanitize all commands before executing in containers
- ✅ Always use environment variables for configuration, not hardcoded values

---

## Common Pitfalls to Avoid

- ❌ Storing secrets in Docker images
- ❌ Using `links` instead of `depends_on`
- ❌ Assuming Docker Hub images are private (they're not by default)
- ❌ Running `reboot` expecting host to restart
- ❌ Bind mounting without protecting generated files/folders
- ❌ Not setting resource limits in production
- ❌ Executing arbitrary AI-generated commands without validation
- ❌ Missing healthchecks in `depends_on` conditions
- ❌ Using deprecated Docker features (links, implicit networks)

---

## Further Learning

See individual files in this directory for detailed information:
- `volumes.md` - Docker volumes, storage, and persistence
- `containers.md` - Container behavior, isolation, and lifecycle
- `docker-compose.md` - Compose networking, configuration, and linking
- `environment-variables.md` - .env files, ARG, ENV, and variable management
- `ai-agent-sandbox.md` - Sandboxing AI agents securely
- `security.md` - Security best practices and image privacy
- `scaling.md` - Scaling strategies by user count
