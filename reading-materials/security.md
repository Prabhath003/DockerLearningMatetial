# Docker Security Best Practices

## Image Privacy & Registries

### Docker Hub (Public by Default)

**Free Account Behavior**:
```
┌─ Docker Hub ──────────────────┐
│ myusername/my-app             │
│ ├─ Visibility: PUBLIC          │
│ ├─ Anyone can pull             │
│ ├─ Cannot make private         │
│ └─ 1 private repo limit        │
└───────────────────────────────┘
```

**What this means**:
- ❌ Code is visible to everyone
- ❌ Anyone can pull your image
- ❌ No authentication required
- ⚠️ Secrets in image are exposed

**Upgrade to Pro to make private**:
- Docker Hub → Billing → Upgrade to Pro
- Then set repository to private

---

### Other Registries (Private by Default)

| Registry | Default | Free | Private |
|----------|---------|------|---------|
| **GitHub (GHCR)** | Private | ✅ Yes | ✅ Yes |
| **AWS ECR** | Private | ❌ Paid | ✅ Yes |
| **Google Artifact** | Private | ❌ Paid | ✅ Yes |
| **Azure ACR** | Private | ❌ Paid | ✅ Yes |
| **GitLab** | Private | ✅ Yes | ✅ Yes |

---

## Using GitHub Container Registry (GHCR)

### Free and Private Alternative to Docker Hub

**Setup**:
```bash
# 1. Create Personal Access Token
# GitHub Settings → Developer Settings → Personal access tokens

# 2. Login
docker login ghcr.io -u USERNAME -p TOKEN

# 3. Tag image
docker tag myapp:latest ghcr.io/myusername/myapp:latest

# 4. Push
docker push ghcr.io/myusername/myapp:latest

# 5. Pull (private, requires auth)
docker pull ghcr.io/myusername/myapp:latest
```

**In compose.yaml**:
```yaml
services:
  app:
    image: ghcr.io/myusername/myapp:latest
    # Requires authentication
```

---

## Using AWS ECR (Private Registry)

**Setup**:
```bash
# 1. Create repository
aws ecr create-repository --repository-name my-app

# 2. Login
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789.dkr.ecr.us-east-1.amazonaws.com

# 3. Tag
docker tag myapp:latest \
  123456789.dkr.ecr.us-east-1.amazonaws.com/my-app:latest

# 4. Push
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/my-app:latest
```

---

## Secrets Management

### ❌ DON'T: Store Secrets in Images

**Bad Example**:
```dockerfile
FROM python:3.12
ENV DATABASE_PASSWORD=mypassword123      # ❌ Exposed!
ENV API_KEY=secret-api-key               # ❌ Exposed!
COPY . .
CMD python app.py
```

**Why it's bad**:
- Anyone with image can see secrets
- Secrets in layer history forever
- Can't change secrets without rebuilding

---

### ✅ DO: Load Secrets at Runtime

**Good Example (Dockerfile)**:
```dockerfile
FROM python:3.12
# No secrets hardcoded
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD python app.py
```

**.env file** (NOT in image):
```bash
DATABASE_PASSWORD=mypassword123
API_KEY=secret-api-key
DATABASE_URL=postgresql://user:pass@db:5432/app
```

**docker-compose.yaml**:
```yaml
services:
  app:
    image: myapp
    env_file: .env              # Load from external file
    environment:
      - CUSTOM_VAR=value
```

**.gitignore** (prevent accidental commit):
```bash
.env
.env.local
*.key
*.pem
secrets/
```

---

### Docker Secrets (Production)

**For Docker Swarm**:
```yaml
version: '3.1'

services:
  app:
    image: myapp
    secrets:
      - db_password
      - api_key

secrets:
  db_password:
    file: ./secrets/db_password.txt
  api_key:
    file: ./secrets/api_key.txt
```

**In application**:
```python
# Read from /run/secrets/
with open('/run/secrets/db_password', 'r') as f:
    password = f.read().strip()
```

---

### Kubernetes Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  DATABASE_PASSWORD: mypassword
  API_KEY: secret-key

---
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp
    env:
    - name: DATABASE_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: DATABASE_PASSWORD
```

---

## Image Security Best Practices

### 1. Don't Run as Root

**Bad**:
```dockerfile
FROM ubuntu:22.04
COPY . /app
CMD python app.py           # Runs as root!
```

**Good**:
```dockerfile
FROM ubuntu:22.04

# Create non-root user
RUN useradd -m -u 1000 appuser

WORKDIR /app
COPY --chown=appuser:appuser . .

USER appuser                # Switch to non-root
CMD python app.py
```

---

### 2. Use Minimal Base Images

**Large (1.5GB)**:
```dockerfile
FROM ubuntu:22.04
```

**Medium (300MB)**:
```dockerfile
FROM python:3.12-slim
```

**Small (5MB)**:
```dockerfile
FROM alpine:3.18
RUN apk add python3
```

---

### 3. Scan for Vulnerabilities

```bash
# Trivy scanner (free)
trivy image myapp:latest

# Docker Scout
docker scout cves myapp:latest

# Output:
# High: 3 vulnerabilities
# Medium: 5 vulnerabilities
```

---

### 4. Use Specific Base Image Versions

**Bad** (unpredictable):
```dockerfile
FROM python:latest          # What version?
FROM ubuntu                 # Which release?
```

**Good** (reproducible):
```dockerfile
FROM python:3.12.1-slim-bookworm
FROM ubuntu:22.04
```

---

### 5. Multi-stage Builds

Reduce image size and expose fewer secrets:

```dockerfile
# Build stage
FROM python:3.12 as builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user -r requirements.txt

# Runtime stage (smaller)
FROM python:3.12-slim
COPY --from=builder /root/.local /root/.local
COPY app.py .
ENV PATH=/root/.local/bin:$PATH
CMD python app.py

# Result: Only app and dependencies, no build tools
```

---

### 6. Pin Dependencies

**Bad** (unpredictable versions):
```dockerfile
RUN pip install flask      # Latest version (could break)
```

**Good** (reproducible):
```dockerfile
RUN pip install flask==2.3.0
```

**requirements.txt**:
```
flask==2.3.0
sqlalchemy==2.0.0
requests==2.31.0
```

---

## Network Security

### Don't Expose Unnecessary Ports

**Bad**:
```yaml
services:
  db:
    image: postgres
    ports:
      - "5432:5432"        # Exposed to all
```

**Good**:
```yaml
services:
  db:
    image: postgres
    ports:
      - "127.0.0.1:5432:5432"   # Localhost only
    # Or no ports (internal access only)
```

---

### Use Network Policies

```yaml
version: '3.8'

services:
  frontend:
    image: nginx
    networks:
      - frontend
  
  api:
    image: myapi
    networks:
      - frontend
      - backend
  
  db:
    image: postgres
    networks:
      - backend

networks:
  frontend:
    driver: bridge
  
  backend:
    driver: bridge
    # Only api can reach db
```

---

## Container Security Context

### Drop Capabilities

```yaml
services:
  app:
    image: myapp
    cap_drop:
      - ALL                 # Drop all capabilities
    cap_add:
      - NET_BIND_SERVICE    # Add only what's needed
```

### Read-only Root

```yaml
services:
  app:
    image: myapp
    read_only: true         # Immutable root filesystem
    tmpfs:
      - /tmp
      - /run
```

---

## Supply Chain Security

### Verify Image Signatures

```bash
# Sign image
docker trust sign myregistry/myapp:latest

# Verify on pull
docker pull myregistry/myapp:latest
# Only pulls if signature valid
```

### Use SBOM (Software Bill of Materials)

```bash
# Generate SBOM
docker scout sbom myapp:latest

# Check dependencies
docker scout cves myapp:latest
```

---

## Audit and Monitoring

### Log Container Actions

```bash
# Monitor real-time events
docker events --filter type=container

# View image history
docker history myapp:latest

# Inspect layer
docker inspect myapp:latest
```

### Audit File

```python
import logging
import json

logger = logging.getLogger('docker_audit')

def audit_log(action, user, result):
    log_entry = {
        "timestamp": datetime.now().isoformat(),
        "action": action,
        "user": user,
        "result": result
    }
    logger.info(json.dumps(log_entry))

audit_log("image_pulled", "user123", "success")
```

---

## Security Checklist

### Before Building Image
- ✅ Use minimal base image (python:3.12-slim)
- ✅ Use specific version tags (not latest)
- ✅ Pin all dependencies
- ✅ Remove build artifacts
- ✅ Don't store secrets

### In Dockerfile
- ✅ Run as non-root user
- ✅ Drop unnecessary capabilities
- ✅ Use read-only filesystem
- ✅ Set security context
- ✅ Add health checks

### Before Pushing
- ✅ Scan for vulnerabilities (trivy, scout)
- ✅ Review base image vulnerabilities
- ✅ Verify no secrets in image
- ✅ Test image runs correctly
- ✅ Sign image

### Registry Settings
- ✅ Use private registry (GHCR, ECR)
- ✅ Enable access controls
- ✅ Require authentication
- ✅ Enable scanning on push
- ✅ Set immutable tags (v1.0.0)

### Runtime
- ✅ Load secrets from .env
- ✅ Set resource limits
- ✅ Use network policies
- ✅ Monitor container behavior
- ✅ Regular cleanup

---

## Quick Decision Guide

| Scenario | Solution |
|----------|----------|
| Personal project | GitHub GHCR (private) |
| Company internal | AWS ECR or Azure ACR |
| Public open source | Docker Hub (public) |
| Sensitive data | AWS ECR + Secrets Manager |
| Secrets in code | Use .env file + .gitignore |
| Multi-stage needed | Use multi-stage build |
| Need vulnerability scan | Use Trivy or Docker Scout |
| Need immutable images | Pin version tags |
