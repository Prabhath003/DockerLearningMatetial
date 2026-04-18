# Docker Compose: Networking and Configuration

## What is Docker Compose?

Docker Compose is a tool for defining and running multi-container Docker applications. It uses YAML files to configure services and their relationships.

---

## Basic Structure

```yaml
version: '3.8'

services:
  app:
    image: node:18
  
  database:
    image: mongo:6

volumes:
  data:

networks:
  default:
```

---

## Networking in Docker Compose

### Automatic Network Creation
When you run `docker-compose up`:
1. Docker creates a bridge network
2. All services connect to it automatically
3. Service names become hostnames

```
┌─ Network: myapp_default ──────┐
│  ├─ app (172.17.0.2)           │
│  ├─ database (172.17.0.3)      │
│  └─ cache (172.17.0.4)         │
└───────────────────────────────┘

Access: app → database by hostname "database"
```

### Service Discovery
```yaml
services:
  app:
    environment:
      - DATABASE_HOST=database      # Auto-resolved to 172.17.0.3
  
  database:
    image: mongo:6
```

---

## `links` vs `depends_on`

### `links` (Deprecated)

```yaml
version: '2'  # Only in v2 and below

services:
  app:
    links:
      - database          # Creates network alias
    environment:
      - DB_HOST=database  # Auto-set by links
  
  database:
    image: mongo:6
```

**What it does**:
- Creates network alias (`database` → container IP)
- Sets environment variables automatically
- Implies dependency ordering
- **Legacy approach** (Docker v1.x era)

**Problems**:
- ❌ Deprecated since v3.0
- ❌ Doesn't work with user-defined networks
- ❌ Implicit dependencies (unclear code)

---

### `depends_on` (Modern)

```yaml
version: '3.8'  # v3.0+

services:
  app:
    depends_on:
      database:
        condition: service_started
    environment:
      - DATABASE_HOST=database      # You set this
  
  database:
    image: mongo:6
```

**What it does**:
- Controls startup order
- Database starts before app
- Doesn't automatically set env vars
- **Modern approach** (recommended)

**Advantages**:
- ✅ Explicit dependencies
- ✅ Works with modern networks
- ✅ Supports health checks
- ✅ Better control

---

### Startup Order Comparison

**With `links`** (implicit):
```
User runs: docker-compose up
           ↓
    Start: database, app (order unclear)
```

**With `depends_on`** (explicit):
```
User runs: docker-compose up
           ↓
    Start: database (waits)
           ↓
    Check: if condition met
           ↓
    Start: app
```

---

## `depends_on` Conditions

### `service_started` (Just started)
```yaml
depends_on:
  database:
    condition: service_started    # Minimal wait
```

**Use when**: Service starts instantly (stateless)

**Problem**: Doesn't check if service is ready
```
┌─ Database Container ────┐
│ Starting...             │
│ (but not accepting      │
│  connections yet)       │
└─────────────────────────┘
        ↓
┌─ App Starts ────────────┐
│ Tries to connect        │
│ Connection refused ✗    │
└─────────────────────────┘
```

---

### `service_healthy` (Passed health check)

```yaml
services:
  app:
    depends_on:
      database:
        condition: service_healthy
  
  database:
    image: mongo:6
    healthcheck:
      test: ["CMD", "mongosh", "localhost", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s
```

**Use when**: Service needs startup time

**Advantage**: Waits for actual readiness
```
┌─ Database Container ──────┐
│ Starting                  │
│ Running...                │
│ Ready ✓ (healthcheck OK)  │
└───────────────────────────┘
        ↓
┌─ App Starts ──────────────┐
│ Connects successfully ✓   │
└───────────────────────────┘
```

---

## Networking Configuration

### Default Network
```yaml
version: '3.8'

services:
  app:
    image: node:18
  
  db:
    image: mongo:6

# Network created automatically: myproject_default
```

### Custom Network
```yaml
version: '3.8'

services:
  app:
    image: node:18
    networks:
      - frontend
  
  db:
    image: mongo:6
    networks:
      - backend
  
  nginx:
    image: nginx
    networks:
      - frontend
      - backend

networks:
  frontend:
    driver: bridge
  
  backend:
    driver: bridge
```

**Access**:
- `app` ↔ `nginx` (same network)
- `nginx` ↔ `db` (same network)
- `app` ↔ `db` (different networks, no direct access)

---

### Port Mapping

```yaml
services:
  app:
    ports:
      - "3000:8000"           # host:container
      - "127.0.0.1:5000:5000" # localhost only
      - "9000"                # random host port
```

**Behavior**:
- `localhost:3000` → Container port 8000
- `127.0.0.1:5000` → Not accessible from outside
- `9000` → Random high port assigned

---

## Service Discovery Examples

### Multiple Microservices

```yaml
version: '3.8'

services:
  frontend:
    image: react-app
    ports:
      - "3000:3000"
    depends_on:
      - api
    environment:
      - API_URL=http://api:8000
  
  api:
    image: python-api
    depends_on:
      database:
        condition: service_healthy
    environment:
      - DATABASE_URL=postgresql://db:5432/mydb
  
  database:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD=secret
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
```

**Service names as hostnames**:
- `frontend` → `http://api:8000` ✓ (cross-service)
- `api` → `postgresql://db:5432/mydb` ✓ (to database)

---

## Environment Variables in Compose

### Method 1: Environment Section
```yaml
services:
  app:
    environment:
      - DATABASE_HOST=db
      - DATABASE_PORT=5432
      - DEBUG=true
```

### Method 2: Env File
```yaml
services:
  app:
    env_file: .env
```

### Method 3: Override
```yaml
services:
  app:
    env_file: .env              # Base values
    environment:
      - DEBUG=false             # Override
```

---

## Compose Networking Behavior

### Container Network Interface
```bash
# Inside container
app@container# ifconfig
eth0: 172.17.0.2 netmask 255.255.255.0
lo: 127.0.0.1 netmask 255.255.255.255
```

### DNS Resolution
```bash
# Container resolves by name
app@container# ping database
PING database (172.17.0.3) ...

# Hostname → IP (internal DNS)
database → 172.17.0.3
```

---

## Common Patterns

### Web + Database
```yaml
version: '3.8'

services:
  web:
    image: nginx
    ports:
      - "80:80"
    depends_on:
      app:
        condition: service_healthy
  
  app:
    image: myapp
    depends_on:
      db:
        condition: service_healthy
    environment:
      - DATABASE_URL=postgresql://db:5432/app
  
  db:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD=secret
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

### Frontend + API + Cache
```yaml
version: '3.8'

services:
  frontend:
    image: react-app
    ports:
      - "3000:3000"
    environment:
      - REACT_APP_API_URL=http://api:8000
  
  api:
    image: fastapi-app
    environment:
      - CACHE_URL=redis://cache:6379
    depends_on:
      cache:
        condition: service_started
  
  cache:
    image: redis:7
    ports:
      - "6379:6379"
```

---

## Troubleshooting

### Container can't reach another service
```bash
# Check: Service name is correct
# Check: Both in same network
# Check: Service is actually running
docker-compose ps

# Check: Network connectivity
docker-compose exec app ping database
```

### Connection refused
```bash
# Service started but not ready
# Solution: Add healthcheck to depends_on condition

# Check service logs
docker-compose logs database
```

### Port already in use
```bash
# Change port mapping
ports:
  - "8001:8000"    # Use different host port

# Or use random port
ports:
  - "8000"
```

### Can't resolve hostname
```bash
# Check service name spelling
# Check service is in same network
# Check network is created
docker network ls
```

---

## Best Practices

✅ **DO**:
- Use `depends_on` with health checks
- Define networks explicitly for multi-tier apps
- Use service names as hostnames
- Set environment variables in env_file
- Keep compose files organized

❌ **DON'T**:
- Use deprecated `links`
- Assume services are ready immediately
- Use `depends_on` without conditions
- Hardcode IP addresses
- Create circular dependencies
