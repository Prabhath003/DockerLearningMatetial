# Docker Learning - Reading Materials Index

A comprehensive guide to Docker concepts, best practices, and advanced topics learned through practical experience.

---

## 📚 Table of Contents

### Core Concepts
- **[KEY_POINTS.md](KEY_POINTS.md)** — Quick reference of essential concepts and best practices

### Detailed Guides
1. **[volumes.md](volumes.md)** — Docker volumes, storage persistence, and file management
2. **[containers.md](containers.md)** — Container behavior, isolation, and lifecycle
3. **[docker-compose.md](docker-compose.md)** — Networking, configuration, and service linking
4. **[environment-variables.md](environment-variables.md)** — ARG, ENV, and .env file management
5. **[security.md](security.md)** — Image privacy, secrets management, and security best practices
6. **[ai-agent-sandbox.md](ai-agent-sandbox.md)** — Building isolated execution environments for AI agents
7. **[scaling.md](scaling.md)** — Scaling strategies from 1 to 50k+ users

---

## 🎯 Quick Navigation

### By Topic

**If you want to learn about...**

| Topic | File | Section |
|-------|------|---------|
| Where volumes are stored | [volumes.md](volumes.md#where-volumes-are-stored) | Storage locations |
| Container isolation | [containers.md](containers.md#container-isolation) | Namespaces & isolation |
| Service discovery | [docker-compose.md](docker-compose.md#networking-in-docker-compose) | Auto-networking |
| links vs depends_on | [docker-compose.md](docker-compose.md#links-vs-depends_on) | Networking patterns |
| .env file loading | [environment-variables.md](environment-variables.md#using-variables-in-compose) | Configuration |
| AI agent sandboxing | [ai-agent-sandbox.md](ai-agent-sandbox.md) | Isolation & security |
| Image privacy | [security.md](security.md#image-privacy--registries) | Registries |
| Scaling strategy | [scaling.md](scaling.md) | Infrastructure planning |

---

### By User Level

**Beginner** (Just starting with Docker):
1. Start with [KEY_POINTS.md](KEY_POINTS.md)
2. Read [volumes.md](volumes.md) - Understand persistence
3. Read [containers.md](containers.md) - Understand isolation
4. Read [environment-variables.md](environment-variables.md) - Configuration basics

**Intermediate** (Using Docker Compose):
1. Read [docker-compose.md](docker-compose.md) - Multi-container setups
2. Read [security.md](security.md) - Protecting your applications
3. Learn [ai-agent-sandbox.md](ai-agent-sandbox.md) - Advanced use cases

**Advanced** (Building at scale):
1. Study [scaling.md](scaling.md) - Architecture decisions
2. Reference all guides for optimization
3. Review [security.md](security.md) - Production hardening

---

### By Use Case

**Development Setup**:
- [volumes.md](volumes.md) - Bind mounts for hot reload
- [docker-compose.md](docker-compose.md) - Multi-service setup
- [environment-variables.md](environment-variables.md) - Local configuration

**Production Deployment**:
- [security.md](security.md) - Hardening & secrets
- [containers.md](containers.md) - Resource limits
- [scaling.md](scaling.md) - Infrastructure planning

**AI/ML Workloads**:
- [ai-agent-sandbox.md](ai-agent-sandbox.md) - Sandboxing
- [containers.md](containers.md) - Process isolation
- [scaling.md](scaling.md) - Distributed execution

**Microservices**:
- [docker-compose.md](docker-compose.md) - Service networking
- [environment-variables.md](environment-variables.md) - Configuration
- [scaling.md](scaling.md) - Multi-service scaling

---

## 📋 Document Summaries

### KEY_POINTS.md
Quick reference guide covering:
- Essential Docker concepts
- Best practices checklist
- Common pitfalls
- Command reference

**Use when**: You need a quick reminder or overview

---

### volumes.md
Comprehensive guide to Docker storage:
- Named volumes vs bind mounts vs anonymous volumes
- Where volumes are stored on different OSes
- Protecting folders from bind mount override
- Volume lifecycle and backup/restore
- Performance considerations
- Multi-container volume sharing

**Use when**: Working with data persistence or file systems

---

### containers.md
Understanding container behavior:
- What happens with `sudo reboot` in container
- Process, filesystem, network, and user isolation
- Container lifecycle and states
- Resource management (CPU, memory, disk)
- Viewing container internals
- Health checks and restart policies
- Troubleshooting

**Use when**: Debugging container behavior or understanding isolation

---

### docker-compose.md
Multi-container orchestration:
- Automatic networking and service discovery
- `links` (deprecated) vs `depends_on` (modern)
- Startup order and health check conditions
- Custom networks and port mapping
- Environment variable integration
- Real-world patterns and examples

**Use when**: Building multi-service applications

---

### environment-variables.md
Configuration management:
- ARG (build-time) vs ENV (runtime)
- .env file loading and precedence
- Accessing variables in applications
- Multiple environment files
- Best practices and security
- Troubleshooting missing variables

**Use when**: Managing application configuration

---

### security.md
Security best practices:
- Docker image privacy on Docker Hub vs other registries
- Secrets management (don't put in images)
- Running as non-root user
- Minimal base images
- Vulnerability scanning
- Network security and policies
- Supply chain security
- Audit and monitoring

**Use when**: Preparing for production or handling sensitive data

---

### ai-agent-sandbox.md
Isolated execution environments:
- Why sandboxing is needed
- Single-user and multi-user architecture
- Python implementation with Docker
- Command validation and whitelisting
- Security features
- File management
- Scaling from 1 to 50k+ users
- Monitoring and cleanup

**Use when**: Building systems that execute untrusted code

---

### scaling.md
Infrastructure planning:
- Tier 1 (1-50 users): Docker Compose
- Tier 2 (50-500): Swarm/K8s
- Tier 3 (500-5k): Multi-node K8s
- Tier 4 (5k-50k): Distributed + queue
- Tier 5 (50k+): Global multi-region
- Cost analysis per tier
- Decision tree for choosing architecture

**Use when**: Planning infrastructure for growing user base

---

## 🔑 Key Concepts Summary

### Container Isolation
- Processes: Isolated via PID namespace
- Filesystem: Isolated except for bind mounts/volumes
- Network: Separate network interface (bridge network by default)
- User: Can run as non-root
- **Result**: `reboot` only stops container, not host

### Volume Types
- **Named volumes**: Managed by Docker, reusable
- **Bind mounts**: Host directory mounted directly, good for dev
- **Anonymous volumes**: Temporary, hard to manage
- **Protection**: Specific volumes override broader bind mounts

### Networking
- **Legacy**: `links` (implicit, deprecated)
- **Modern**: `depends_on` (explicit, recommended)
- **Discovery**: Service names = hostnames automatically
- **Isolation**: Can use custom networks to control access

### Configuration
- **Build-time**: `ARG` in Dockerfile
- **Runtime**: `ENV` in Dockerfile or compose
- **External**: `.env` file loaded by compose
- **Priority**: CLI > compose environment > env_file > Dockerfile

### Security
- **Images**: PUBLIC on Docker Hub free (use GHCR or ECR instead)
- **Secrets**: Load from .env, never hardcode
- **User**: Run as non-root
- **Base**: Use minimal, specific version images

### Scaling
- **1-50 users**: Single machine
- **50-500**: Load balancing
- **500+**: Kubernetes
- **5k+**: Queue + multi-region
- **50k+**: Global distributed system

---

## 💡 Common Questions & Answers

**Q: Where are Docker volumes stored?**
A: See [volumes.md#where-volumes-are-stored](volumes.md#where-volumes-are-stored)

**Q: What happens if I run `sudo reboot` in a container?**
A: See [containers.md#what-happens-with-sudo-reboot-in-container](containers.md#what-happens-with-sudo-reboot-in-container)

**Q: What's the difference between `links` and `depends_on`?**
A: See [docker-compose.md#links-vs-depends_on](docker-compose.md#links-vs-depends_on)

**Q: How do I prevent node_modules from being overridden?**
A: See [volumes.md#protecting-folders-from-override](volumes.md#protecting-folders-from-override)

**Q: Are Docker images public or private?**
A: See [security.md#docker-hub-public-by-default](security.md#docker-hub-public-by-default)

**Q: How do I load .env variables into Docker?**
A: See [environment-variables.md#using-variables-in-compose](environment-variables.md#using-variables-in-compose)

**Q: How do I sandbox AI agents safely?**
A: See [ai-agent-sandbox.md](ai-agent-sandbox.md)

**Q: How do I scale Docker for many users?**
A: See [scaling.md](scaling.md)

---

## 🚀 Quick Start Paths

### "I'm new to Docker"
```
1. KEY_POINTS.md (5 min)
2. containers.md (15 min)
3. volumes.md (15 min)
4. environment-variables.md (10 min)
Total: ~45 minutes
```

### "I need to set up multi-container services"
```
1. docker-compose.md (20 min)
2. volumes.md (15 min)
3. environment-variables.md (10 min)
Total: ~45 minutes
```

### "I'm building an AI agent system"
```
1. containers.md (15 min)
2. ai-agent-sandbox.md (20 min)
3. security.md (15 min)
4. scaling.md (15 min)
Total: ~65 minutes
```

### "I'm deploying to production"
```
1. security.md (20 min)
2. scaling.md (20 min)
3. volumes.md - persistence (10 min)
4. docker-compose.md - networking (15 min)
Total: ~65 minutes
```

---

## 📖 Reading Order by Topic Depth

**For Complete Understanding** (recommended order):
1. [KEY_POINTS.md](KEY_POINTS.md) - Overview
2. [containers.md](containers.md) - Fundamentals
3. [volumes.md](volumes.md) - Storage
4. [docker-compose.md](docker-compose.md) - Multi-service
5. [environment-variables.md](environment-variables.md) - Configuration
6. [security.md](security.md) - Hardening
7. [ai-agent-sandbox.md](ai-agent-sandbox.md) - Advanced use case
8. [scaling.md](scaling.md) - Architecture

---

## 🔗 Cross-References

**Topics that work together**:
- Volumes + Compose: How to mount volumes in services
- Networking + Compose: How services discover each other
- Environment + Compose: How to configure services
- Security + Images: How to build safe images
- Sandboxing + Scaling: How to scale isolated environments

---

## 📝 Contributing

These materials are organized for quick reference and comprehension. Each file is:
- ✅ Self-contained (can be read independently)
- ✅ Organized by topic with clear sections
- ✅ Includes practical examples
- ✅ Covers both what and why
- ✅ References related content

---

## 📚 Additional Resources

**Official Documentation**:
- [Docker Documentation](https://docs.docker.com)
- [Docker Compose Reference](https://docs.docker.com/compose/compose-file/)
- [Kubernetes Documentation](https://kubernetes.io/docs)

**Tools Referenced**:
- [Trivy - Vulnerability Scanner](https://github.com/aquasecurity/trivy)
- [Docker Scout](https://docs.docker.com/scout/)
- [Prometheus - Monitoring](https://prometheus.io/)
- [Jaeger - Distributed Tracing](https://www.jaegertracing.io/)

---

**Last Updated**: 2026-04-18
**Total Content**: 8 comprehensive guides covering Docker fundamentals to enterprise scaling
