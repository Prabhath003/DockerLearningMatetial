# Environment Variables in Docker

## Overview

Environment variables are key-value pairs that configure application behavior at runtime. Docker provides multiple ways to set them.

---

## Three Types of Variables

### 1. ARG (Build-time)

**Location**: Dockerfile

**Purpose**: Configure during image build

```dockerfile
ARG PYTHON_VERSION=3.12.11
ARG NODE_ENV=production
ARG BUILD_DATE

FROM python:${PYTHON_VERSION}-slim
```

**Characteristics**:
- ✓ Available during build only
- ✓ Can be overridden at build time
- ✓ Not available in running container
- ✓ Used for build configuration

**Usage**:
```bash
# Use default
docker build -t myapp .

# Override
docker build --build-arg PYTHON_VERSION=3.11 -t myapp .

# Multiple args
docker build \
  --build-arg PYTHON_VERSION=3.12 \
  --build-arg NODE_ENV=production \
  -t myapp .
```

---

### 2. ENV (Runtime)

**Location**: Dockerfile

**Purpose**: Set variables available in running container

```dockerfile
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
ENV APP_HOME=/app
```

**Characteristics**:
- ✓ Available in running container
- ✓ Persist across layers
- ✓ Can be overridden at runtime
- ✓ Used for application config

**Usage**:
```bash
docker run -e DEBUG=true myapp
docker run --env-file .env myapp
```

---

### 3. .env File (External)

**Location**: `.env` file in project root

**Purpose**: Store configuration outside code

**.env file**:
```bash
DATABASE_URL=postgresql://user:pass@localhost:5432/db
API_KEY=secret123
DEBUG=true
NODE_ENV=production
LOG_LEVEL=info
```

**Characteristics**:
- ✓ External to code
- ✓ Easy to change per environment
- ✓ Version control friendly (add to .gitignore)
- ✓ Loaded by Docker Compose

---

## Hierarchy and Precedence

```
Highest Priority (overrides everything):
1. docker run -e VAR=value          (command line)
2. environment: in compose.yaml     (explicit override)
3. env_file: in compose.yaml        (from file)
4. ENV in Dockerfile                (hardcoded in image)

Lowest Priority
```

**Example**:
```yaml
services:
  app:
    image: myapp
    env_file: .env                      # DATABASE_URL=mongo
    environment:
      DATABASE_URL=postgres             # Overrides .env
      DEBUG=true                        # Additional
```

Result: `DATABASE_URL=postgres` (not from .env)

---

## Using Variables in Dockerfile

### ARG in FROM
```dockerfile
ARG PYTHON_VERSION=3.12
FROM python:${PYTHON_VERSION}-slim
```

### ARG in RUN
```dockerfile
ARG BUILD_DATE
RUN echo "Built on ${BUILD_DATE}" > /build-info.txt
```

### ENV for runtime
```dockerfile
ENV DATABASE_HOST=localhost
ENV DATABASE_PORT=5432
```

### Multi-stage with ARG
```dockerfile
ARG BUILD_STAGE=dev

FROM python:3.12 as base
ENV PYTHONDONTWRITEBYTECODE=1

FROM base as dev
RUN pip install pytest

FROM base as prod
# Smaller image without dev dependencies
```

---

## Using Variables in Compose

### Environment section
```yaml
services:
  app:
    environment:
      - DATABASE_HOST=db
      - DATABASE_PORT=5432
      - DEBUG=false
      - PYTHONUNBUFFERED=1
```

### Env file
```yaml
services:
  app:
    env_file: .env
```

### Reference .env in compose
```yaml
services:
  app:
    image: myapp:${APP_VERSION}
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - API_KEY=${API_KEY}
```

### Multiple env files (priority)
```yaml
services:
  app:
    env_file:
      - .env                           # Base
      - .env.${ENVIRONMENT}            # Override
```

---

## Loading .env in Docker Compose

### Automatic loading
```bash
# Docker Compose automatically loads .env in current directory
docker-compose up
```

### Specific env file
```bash
docker-compose --env-file .env.production up
```

### How it works
```
.env file:
DATABASE_URL=mongodb://localhost:27017
API_KEY=secret123
DEBUG=true

              ↓ (docker-compose up)

Container environment:
DATABASE_URL=mongodb://localhost:27017
API_KEY=secret123
DEBUG=true
```

---

## Accessing Variables in Applications

### Python
```python
import os

# Get with default
database_url = os.getenv("DATABASE_URL", "localhost")

# Get or fail
api_key = os.environ["API_KEY"]  # Raises KeyError if not set

# Check existence
if "DEBUG" in os.environ:
    debug = os.environ["DEBUG"].lower() == "true"
```

### Node.js
```javascript
const databaseUrl = process.env.DATABASE_URL || 'localhost';
const apiKey = process.env.API_KEY;

if (process.env.DEBUG === 'true') {
  // Debug mode
}
```

### Go
```go
package main

import "os"

func main() {
    databaseUrl := os.Getenv("DATABASE_URL")
    apiKey := os.Getenv("API_KEY")
}
```

### Bash
```bash
#!/bin/bash
DATABASE_URL="${DATABASE_URL:-localhost}"
API_KEY="${API_KEY:-default}"
DEBUG="${DEBUG:-false}"
```

---

## Real-world Examples

### Development Setup
```yaml
# .env
DATABASE_URL=postgresql://user:pass@db:5432/mydb
API_KEY=dev-key-123
DEBUG=true
LOG_LEVEL=debug
ENVIRONMENT=development
```

### Production Setup
```yaml
# .env.production
DATABASE_URL=postgresql://user:prod-secret@prod-db:5432/mydb
API_KEY=prod-key-secret
DEBUG=false
LOG_LEVEL=warn
ENVIRONMENT=production
```

### Docker Compose
```yaml
version: '3.8'

services:
  app:
    build: .
    env_file:
      - .env                  # Base variables
      - .env.${ENVIRONMENT}   # Environment-specific
    environment:
      - DATABASE_HOST=db      # Service name
    depends_on:
      db:
        condition: service_healthy
  
  db:
    image: postgres:15
    env_file: .env
    environment:
      - POSTGRES_DB=${DATABASE_NAME}
      - POSTGRES_USER=${DATABASE_USER}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DATABASE_USER}"]
```

---

## Best Practices

✅ **DO**:
- Use .env files for configuration
- Add .env to .gitignore (keep secrets safe)
- Use environment-specific .env files
- Document required variables
- Use meaningful variable names
- Provide defaults in code

❌ **DON'T**:
- Hardcode secrets in code
- Commit .env to git
- Use ambiguous variable names
- Assume variables are set
- Mix multiple config methods
- Store sensitive data in ENV (use secrets instead)

---

## Security Best Practices

### Never commit secrets
```bash
# .gitignore
.env
.env.local
.env.*.local
```

### Use secrets for production
```yaml
# Docker Secrets (for Swarm)
services:
  app:
    secrets:
      - db_password
      - api_key

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

### Validate at startup
```python
import os
import sys

required_vars = ['DATABASE_URL', 'API_KEY']
missing = [var for var in required_vars if var not in os.environ]

if missing:
    print(f"Missing required variables: {missing}")
    sys.exit(1)
```

---

## Troubleshooting

### Variable not set
```bash
# Check if file exists and is readable
cat .env

# Check Docker Compose loads it
docker-compose config | grep DATABASE_URL

# Debug
docker-compose run app env | grep DATABASE
```

### Variable empty or wrong value
```bash
# Check variable is defined in .env
echo $DATABASE_URL

# Check for whitespace
# Don't do: DATABASE_URL = value (space before =)
# Do: DATABASE_URL=value
```

### Can't access variable in app
```python
# Verify variable is set
import os
print(os.environ.get('DATABASE_URL', 'NOT SET'))

# Check if variable name matches exactly
# Environment variables are case-sensitive
```

---

## Quick Reference

| Task | How |
|------|-----|
| Set at build | `ARG` in Dockerfile |
| Set at runtime | `ENV` in Dockerfile |
| Load from file | `env_file:` in Compose |
| Override | `environment:` in Compose |
| Build arg | `docker build --build-arg` |
| Run override | `docker run -e VAR=value` |
| Access in Python | `os.getenv("VAR")` |
| Access in Node | `process.env.VAR` |
