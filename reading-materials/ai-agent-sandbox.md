# AI Agent Sandbox: Secure Execution Environment

## Overview

An AI Agent Sandbox provides an isolated Docker container where AI agents can safely execute commands without exposing your host system to risk.

---

## Why You Need a Sandbox

### Risks of Direct Execution
```
❌ Unsafe: AI agents execute directly on host
docker exec host-system "ai-generated-command"
                ↓
        Host system compromise
        File deletion
        Data theft
        Credential theft
```

### Safe: Isolated Container
```
✅ Safe: AI agents execute in container
docker exec ai-sandbox "ai-generated-command"
                ↓
        Only affects container
        Host system protected
        Data isolated per user
        Easy cleanup/restart
```

---

## Architecture

### Single-User Sandbox
```
┌─ Host System ─────────────────────────┐
│                                       │
│  ┌─ Docker Container ────────────┐   │
│  │  Ubuntu 22.04                 │   │
│  │  ├─ /workspace (volume)       │   │
│  │  ├─ Isolated processes        │   │
│  │  ├─ Resource limits           │   │
│  │  └─ Limited filesystem access │   │
│  └───────────────────────────────┘   │
│           ↑                           │
│    docker exec (commands)             │
│                                       │
└───────────────────────────────────────┘
```

### Multi-User Sandbox (Scale)
```
┌─ Kubernetes Cluster ─────────────────────┐
│                                          │
│  ┌─ Namespace: user-1 ─────────────┐    │
│  │ Container: ai-sandbox-1         │    │
│  │ Volume: workspace-1             │    │
│  │ Resources: CPU, Memory limits   │    │
│  └─────────────────────────────────┘    │
│                                          │
│  ┌─ Namespace: user-2 ─────────────┐    │
│  │ Container: ai-sandbox-2         │    │
│  │ Volume: workspace-2             │    │
│  │ Resources: CPU, Memory limits   │    │
│  └─────────────────────────────────┘    │
│                                          │
└──────────────────────────────────────────┘
```

---

## Basic Setup

### Docker Compose Configuration
```yaml
version: '3.8'

services:
  ai-sandbox:
    image: ubuntu:22.04
    container_name: ai-sandbox-${USER_ID}
    stdin_open: true
    tty: true
    
    # Persistent storage
    volumes:
      - ai-workspace-${USER_ID}:/workspace
    
    # Resource limits
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G
        reservations:
          cpus: '1'
          memory: 1G
    
    # Security
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    
    command: /bin/bash

volumes:
  ai-workspace-${USER_ID}:
    driver: local
```

---

## Python Implementation

### Sandbox Manager
```python
import docker
import subprocess
from typing import Dict, Optional

class AIAgentSandbox:
    def __init__(self, user_id: str, max_timeout: int = 300):
        self.user_id = user_id
        self.max_timeout = max_timeout
        self.container_name = f"ai-sandbox-{user_id}"
        self.client = docker.from_env()
    
    def create_container(self) -> bool:
        """Create isolated container for user."""
        try:
            self.container = self.client.containers.run(
                "ubuntu:22.04",
                command="/bin/bash",
                name=self.container_name,
                stdin_open=True,
                tty=True,
                volumes={f"ai-workspace-{self.user_id}": 
                        {"bind": "/workspace", "mode": "rw"}},
                mem_limit="2g",
                cpus=2,
                cap_drop=["ALL"],
                detach=True,
            )
            return True
        except Exception as e:
            print(f"Error: {e}")
            return False
    
    def execute_command(self, command: str, 
                       timeout: Optional[int] = None) -> Dict:
        """Execute command safely in sandbox."""
        timeout = timeout or self.max_timeout
        
        # Validate command
        if not self._validate_command(command):
            return {"success": False, "stderr": "Command not allowed"}
        
        try:
            exit_code, output = self.container.exec_run(
                cmd=["bash", "-c", command],
                stdout=True,
                stderr=True,
                timeout=timeout
            )
            
            return {
                "success": exit_code == 0,
                "stdout": output.decode('utf-8', errors='replace'),
                "exit_code": exit_code
            }
        except subprocess.TimeoutExpired:
            return {
                "success": False,
                "stderr": f"Command timeout after {timeout}s",
                "exit_code": -1
            }
    
    def _validate_command(self, command: str) -> bool:
        """Whitelist safe commands."""
        allowed = {"mkdir", "touch", "echo", "cat", "ls", 
                  "cd", "cp", "mv", "rm", "git", "npm"}
        try:
            first_cmd = command.split()[0].split("/")[-1]
            return first_cmd in allowed
        except:
            return False
    
    def cleanup(self) -> bool:
        """Stop and remove container."""
        try:
            self.container.stop()
            self.container.remove()
            return True
        except:
            return False
```

---

## Security Features

### 1. Filesystem Isolation
```
Container sees:
├─ /workspace (bound volume)
└─ /tmp (isolated tmpfs)

Cannot access:
├─ Host root /
├─ Other containers' data
└─ Host private keys
```

### 2. Process Isolation
```
Container processes are isolated by namespace
Container PID 1 ≠ Host PID 1
Signals don't affect host
Reboot only kills container
```

### 3. Network Isolation
```
Container has own network interface
No exposed ports by default
Only localhost accessible
Inter-container communication blocked (unless specified)
```

### 4. Resource Limits
```yaml
resources:
  limits:
    cpus: '2'         # Max 2 CPU cores
    memory: 2G        # Max 2GB RAM
  reservations:
    cpus: '1'         # Minimum resources
    memory: 1G
```

### 5. User Isolation
```dockerfile
# Run as non-root
RUN useradd -m appuser
USER appuser
```

---

## Command Validation

### Whitelist Approach
```python
ALLOWED_COMMANDS = {
    "mkdir", "touch", "echo", "cat", "ls",
    "cd", "cp", "mv", "rm", "find", "grep",
    "curl", "wget", "git", "npm", "python"
}

def validate(command: str) -> bool:
    first_word = command.split()[0]
    return first_word in ALLOWED_COMMANDS
```

### Blacklist Approach (Not recommended)
```python
DANGEROUS = {
    "rm -rf /", ":(){ :|: & };:", "dd if=/dev/zero"
}

# ❌ Can't catch all dangerous commands
# Use whitelist instead
```

---

## Scaling Strategy

### Tier 1: < 50 users
- Docker Compose on single machine
- Simple volume management
- Manual cleanup

### Tier 2: 50-500 users
- Docker Swarm or single Kubernetes node
- Shared storage (NFS)
- Load balancing

### Tier 3: 500-5k users
- Kubernetes multi-node
- Namespace per user
- Resource quotas

### Tier 4: 5k+ users
- Distributed Kubernetes
- Queue-based execution
- Multi-region failover

---

## API Integration (FastAPI)

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()
controller = AIAgentController()

class CommandRequest(BaseModel):
    user_id: str
    command: str
    timeout: int = 300

@app.post("/execute")
async def execute(request: CommandRequest):
    """Execute command for user."""
    result = controller.execute_for_user(
        request.user_id,
        request.command
    )
    return result

@app.post("/cleanup/{user_id}")
async def cleanup(user_id: str):
    """Clean up user sandbox."""
    if controller.cleanup_user(user_id):
        return {"status": "cleaned"}
    return {"error": "Not found"}
```

---

## File Management

### Create File
```python
sandbox.execute_command("echo 'content' > /workspace/file.txt")
```

### Read File
```python
result = sandbox.execute_command("cat /workspace/file.txt")
print(result["stdout"])
```

### List Workspace
```python
result = sandbox.execute_command("ls -la /workspace")
```

### Copy Out
```bash
docker cp ai-sandbox-user1:/workspace/file.txt ./
```

---

## Monitoring and Cleanup

### Health Check
```python
def is_healthy(sandbox):
    result = sandbox.execute_command("echo 'alive'")
    return result["success"]
```

### Resource Usage
```bash
docker stats ai-sandbox-user1
```

### Auto-cleanup (Inactivity)
```python
def cleanup_idle_containers(timeout_minutes=30):
    """Stop containers inactive for timeout."""
    for sandbox in get_all_sandboxes():
        if sandbox.last_activity > timeout_minutes:
            sandbox.cleanup()
```

### Log Operations
```python
def log_operation(user_id, command, result):
    import json
    import datetime
    log_entry = {
        "timestamp": datetime.datetime.now().isoformat(),
        "user_id": user_id,
        "command": command,
        "success": result["success"]
    }
    with open("audit.log", "a") as f:
        f.write(json.dumps(log_entry) + "\n")
```

---

## Best Practices

✅ **DO**:
- Validate all commands before execution
- Set resource limits
- Use per-user containers
- Monitor resource usage
- Clean up idle containers
- Log all operations
- Implement health checks
- Use timeouts for long operations

❌ **DON'T**:
- Execute unvalidated commands
- Skip resource limits
- Expose host filesystem
- Trust AI output blindly
- Forget cleanup
- Run as root
- Expose debugging ports
- Store secrets in containers

---

## Troubleshooting

### Container won't start
```bash
docker logs ai-sandbox-user1
# Check: Resource availability, port conflicts
```

### Command fails
```bash
# Check: Command in whitelist, syntax correct
docker exec -it ai-sandbox-user1 bash
# Debug interactively
```

### Disk full
```bash
docker exec ai-sandbox-user1 du -sh /workspace
# Clean up old files
docker exec ai-sandbox-user1 rm -rf /workspace/old
```

### Memory issues
```bash
docker stats ai-sandbox-user1
# Increase limit or kill hanging processes
```

---

## Complete Example

```python
# Create sandbox
sandbox = AIAgentSandbox("user123")
sandbox.create_container()

# Execute AI command
result = sandbox.execute_command("mkdir -p /workspace/project")

# Create files
sandbox.execute_command("echo 'Hello' > /workspace/project/file.txt")

# Read files
content = sandbox.execute_command("cat /workspace/project/file.txt")
print(content["stdout"])

# Cleanup
sandbox.cleanup()
```

---

## Security Checklist

- ✅ Validate all AI-generated commands
- ✅ Set CPU/memory/disk limits
- ✅ Run containers as non-root user
- ✅ Use read-only filesystems where possible
- ✅ Drop unnecessary Linux capabilities
- ✅ Enable security contexts
- ✅ Monitor resource usage
- ✅ Implement timeout mechanisms
- ✅ Audit all operations
- ✅ Regular cleanup of old containers
