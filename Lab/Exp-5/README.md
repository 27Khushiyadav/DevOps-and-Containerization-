# Experiment 5: Docker — Volumes, Environment Variables, Monitoring & Networks

> **Platform:** MacBook Air M1 | **Date:** March 14–15, 2026  
> **Purpose:** Learn essential Docker features for building, configuring, and managing production-ready containerized applications.

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Docker Volumes](#part-1-docker-volumes)
- [Part 2: Environment Variables](#part-2-environment-variables)
- [Part 3: Docker Monitoring](#part-3-docker-monitoring)
- [Part 4: Docker Networks](#part-4-docker-networks)
- [Part 5: Real-World Example](#part-5-real-world-example)
- [Cleanup](#cleanup)
- [Errors Encountered & Fixes](#errors-encountered--fixes)
- [Key Takeaways](#key-takeaways)

---

## Overview

| Part   | Topic                  | What Was Done                                          |
|--------|------------------------|--------------------------------------------------------|
| Part 1 | Docker Volumes         | Anonymous, Named & Bind Mounts with persistence demo   |
| Part 2 | Environment Variables  | -e flag, --env-file, Flask app configuration           |
| Part 3 | Docker Monitoring      | stats, logs, top, inspect, system df                   |
| Part 4 | Docker Networks        | Bridge, Host, None & Custom networks                   |
| Part 5 | Real-World App         | Flask + PostgreSQL + Redis multi-container setup       |

---

## Part 1: Docker Volumes

Docker containers are ephemeral — all data written inside a container is lost when it is removed. Docker Volumes solve this by persisting data outside the container lifecycle.

---

### Lab 1: Understanding Data Persistence

#### Step 1 — Verify Docker Installation

```bash
docker --version
docker ps
```

> 📌 Docker Desktop must be running before executing any docker commands.

---

#### Step 2 — Create Container and Write Data

```bash
docker run -it --name test-container ubuntu /bin/bash

# Inside container:
mkdir /data && echo "Hello World" > /data/message.txt
cat /data/message.txt
# Output: Hello World
exit
```

---

#### Step 3 — Prove Data is Ephemeral

```bash
# Restart and check — data still exists (container just stopped)
docker start test-container
docker exec test-container cat /data/message.txt

# Now remove and recreate
docker stop test-container
docker rm test-container

docker run -it --name test-container ubuntu /bin/bash
cat /data/message.txt
# ERROR: No such file or directory ← data is gone!
exit
```

> 📌 This proves container data does NOT survive removal.

---

#### ❌ Error: Cannot remove running container

```
Error response from daemon: cannot remove container "test-container": 
container is running: stop the container before removing or force remove
```

#### ✅ Fix:

```bash
docker stop test-container
docker rm test-container
```

---

### Lab 2: Volume Types

#### Step 4 — Anonymous Volumes

```bash
docker run -d -v /app/data --name web1 nginx
docker volume ls
docker inspect web1 | grep -A 5 Mounts
```

**Output:** A random hash name is auto-generated for the anonymous volume, mounted at `/app/data`.

---

#### Step 5 — Named Volumes

```bash
docker volume create mydata
docker run -d -v mydata:/app/data --name web2 nginx
docker volume ls
docker volume inspect mydata
```

**Output:**
```json
{
  "Name": "mydata",
  "Driver": "local",
  "Mountpoint": "/var/lib/docker/volumes/mydata/_data",
  "Scope": "local"
}
```

---

#### Step 6 — Bind Mounts (Host Directory)

```bash
mkdir ~/myapp-data
docker run -d -v ~/myapp-data:/app/data --name web3 nginx

# Add file on host
echo "From Host" > ~/myapp-data/host-file.txt

# Verify inside container
docker exec web3 cat /app/data/host-file.txt
# Output: From Host
```

> 📌 Bind mounts enable live sync between Mac host and container — any file added on the host is immediately visible inside the container.

---

### Lab 3: Practical Volume Examples

#### Step 7 — MySQL with Persistent Volume

```bash
docker run -d \
  --name mysql-db \
  -v mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  mysql:8.0

# Remove the container
docker stop mysql-db && docker rm mysql-db

# Recreate with same volume — data is preserved!
docker run -d \
  --name new-mysql \
  -v mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  mysql:8.0

docker ps
```

---

#### Step 8 — Nginx with Config Bind Mount

```bash
mkdir ~/nginx-config

echo 'server {
  listen 80;
  server_name localhost;
  location / {
    return 200 "Hello from mounted config!";
  }
}' > ~/nginx-config/nginx.conf

docker run -d \
  --name nginx-custom \
  -p 8080:80 \
  -v ~/nginx-config/nginx.conf:/etc/nginx/conf.d/default.conf \
  nginx

curl http://localhost:8080
# Output: Hello from mounted config!
```

---

### Lab 4: Volume Management Commands

#### Step 9 — Volume Management

```bash
docker volume ls                    # list all volumes
docker volume create app-volume     # create a volume
docker volume inspect app-volume    # inspect details
docker volume prune                 # remove unused anonymous volumes
docker volume rm volume-name        # remove specific volume
```

---

## Part 2: Environment Variables

Environment variables allow dynamic configuration of containers without hardcoding values into the image.

---

### Lab 1: Setting Environment Variables

#### Step 10 — Using the -e Flag

```bash
docker run -d \
  --name app1 \
  -e VAR1=value1 \
  -e VAR2=value2 \
  -e VAR3=value3 \
  nginx

docker exec app1 env
# Output includes: VAR1=value1, VAR2=value2, VAR3=value3
```

---

#### Step 11 — Using --env-file

```bash
echo "DATABASE_HOST=localhost" > ~/.env
echo "DATABASE_PORT=5432" >> ~/.env
echo "API_KEY=secret123" >> ~/.env

docker run -d \
  --name app2 \
  --env-file ~/.env \
  nginx

docker exec app2 env
# Output includes: DATABASE_HOST, DATABASE_PORT, API_KEY
```

---

### Lab 2: Flask App with Environment Variables

#### Step 12 — Create Flask Application

**app.py:**
```python
import os
from flask import Flask
app = Flask(__name__)

db_host = os.environ.get('DATABASE_HOST', 'localhost')
debug_mode = os.environ.get('DEBUG', 'false').lower() == 'true'
api_key = os.environ.get('API_KEY')

@app.route('/config')
def config():
    return {
        'db_host': db_host,
        'debug': debug_mode,
        'has_api_key': bool(api_key)
    }

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 5000))
    app.run(host='0.0.0.0', port=port, debug=debug_mode)
```

**requirements.txt:**
```
flask
```

**Dockerfile:**
```dockerfile
FROM python:3.9-slim
ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY app.py .
ENV PORT=5000
ENV DEBUG=false
EXPOSE 5000
CMD ["python", "-u", "app.py"]
```

```bash
docker build -t flask-app .
```

---

#### ❌ Error: Container exited immediately in detached mode

```
docker logs flask-app → (empty output)
docker ps | grep flask-app → (no output, container not running)
```

**Cause:** Python output buffering — Flask was not emitting logs, so the container appeared to crash silently.

#### ✅ Fix: Use -u flag for unbuffered output

Changed `CMD` in Dockerfile:
```dockerfile
CMD ["python", "-u", "app.py"]
```
Then rebuilt the image:
```bash
docker build -t flask-app .
```

---

#### ❌ Error: Port 5000 already in use

```
Error response from daemon: ports are not available: exposing port TCP 0.0.0.0:5000 
→ 127.0.0.1:0: listen tcp 0.0.0.0:5000: bind: address already in use
```

**Cause:** macOS uses port 5000 for AirPlay Receiver by default.

#### ✅ Fix: Use port 5001 instead

```bash
docker run -d \
  --name flask-app \
  -p 5001:5000 \
  -e DATABASE_HOST="prod-db.example.com" \
  -e DEBUG="true" \
  -e API_KEY="myapikey123" \
  flask-app
```

---

#### Step 13 — Test Flask App

```bash
curl http://localhost:5001/config
```

**Response:**
```json
{
  "db_host": "prod-db.example.com",
  "debug": true,
  "has_api_key": true
}
```

---

## Part 3: Docker Monitoring

Docker provides several commands to monitor container health, resource usage, processes, and logs.

---

### Lab 1: Resource Metrics

#### Step 14 — docker stats

```bash
# Snapshot of all containers
docker stats --no-stream

# Formatted output
docker stats --no-stream --format \
  "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}"
```

---

#### Step 14b — docker top

```bash
docker top flask-app
```

**Output:**
```
UID    PID   PPID  CMD
root   2448  2425  python -u app.py
root   2469  2448  /usr/local/bin/python /app/app.py
```

---

### Lab 2: Log Monitoring

#### Step 15 — docker logs

```bash
docker logs flask-app                   # view logs
docker logs -t flask-app                # with timestamps
docker logs --tail 5 flask-app          # last 5 lines
docker logs -f --tail 10 -t flask-app   # follow live logs
```

**Live log output showed:**
- Flask startup on port 5000
- 3 GET /config requests with status 200

---

### Lab 3: Container Inspection

#### Step 16 — docker inspect

```bash
docker inspect --format='{{.State.Status}}' flask-app
docker inspect --format='{{.NetworkSettings.IPAddress}}' flask-app
docker inspect --format='{{.Config.Env}}' flask-app
docker inspect --format='{{.HostConfig.Memory}}' flask-app
docker system df
```

**docker system df output:**
```
TYPE            TOTAL   SIZE      RECLAIMABLE
Images          5       2.638GB   2.638GB (100%)
Containers      12      610.3kB   12.29kB (2%)
Local Volumes   6       496.8MB   0B (0%)
Build Cache     11      217.5MB   24.58kB
```

---

## Part 4: Docker Networks

Docker networks enable secure, isolated communication between containers. Containers on the same custom network can reach each other using container names as hostnames — Docker's built-in DNS.

---

### Lab 1: Network Types

#### Step 17 — List and Create Networks

```bash
docker network ls
# Output: bridge, host, none (defaults)

docker network create my-network
docker network inspect my-network
```

---

#### Step 17b — Container Communication via Custom Network

```bash
docker run -d --name net-web1 --network my-network nginx
docker run -d --name net-web2 --network my-network nginx

# Test communication using container name as hostname
docker exec net-web1 curl -s http://net-web2
# Output: nginx welcome HTML page ✅
```

> 📌 Docker's built-in DNS resolved `net-web2` to its container IP automatically.

---

### Lab 2: Network Isolation

#### Step 18 — None Network (Full Isolation)

```bash
docker run -d --name isolated-app --network none alpine sleep 3600
docker exec isolated-app ifconfig
# Only loopback (lo) interface visible — no network access
```

---

#### ❌ Error: nslookup not found in nginx image

```
OCI runtime exec failed: exec: "nslookup": executable file not found in $PATH
```

#### ✅ Fix: Use docker network inspect instead

```bash
docker network inspect my-network
```

---

#### ❌ Error: ping not found in nginx image

```
OCI runtime exec failed: exec: "ping": executable file not found in $PATH
```

#### ✅ Fix: Connectivity already proven via curl

Container communication was already confirmed using:
```bash
docker exec net-web1 curl -s http://net-web2
```

---

#### Step 18b — Network Inspection

```bash
docker network inspect my-network
```

**Output confirmed:**
```
net-web1 → 172.19.0.2/16
net-web2 → 172.19.0.3/16
Gateway  → 172.19.0.1
Subnet   → 172.19.0.0/16
```

---

#### Step 18c — Connect Existing Container to Network

```bash
docker network connect my-network flask-app
docker network inspect my-network
# flask-app now appears in Containers section
```

---

## Part 5: Real-World Example

A full 3-tier application stack: **Flask** web app + **PostgreSQL** database + **Redis** cache — all connected via a custom Docker network with persistent volumes.

### Architecture

```
┌─────────────────────────────────────────┐
│           myapp-network                 │
│                                         │
│  ┌──────────┐  ┌──────────┐  ┌───────┐  │
│  │flask-app │  │ postgres │  │ redis │  │
│  │:5001→5000│  │  :5432   │  │ :6379 │  │
│  └──────────┘  └──────────┘  └───────┘  │
│       │              │            │      │
│       └──── volume ──┘            │      │
│            postgres-data    redis-data   │
└─────────────────────────────────────────┘
```

---

### Step 19 — Deploy the Stack

#### Create Network

```bash
docker network create myapp-network
```

#### Start PostgreSQL

```bash
docker run -d \
  --name postgres \
  --network myapp-network \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -e POSTGRES_DB=mydatabase \
  -v postgres-data:/var/lib/postgresql/data \
  postgres:15
```

#### Start Redis

```bash
docker run -d \
  --name redis \
  --network myapp-network \
  -v redis-data:/data \
  redis:7-alpine
```

#### Verify Services

```bash
docker ps | grep -E "postgres|redis"
```

**Output:**
```
redis:7-alpine   → Up  (port 6379)
postgres:15      → Up  (port 5432)
```

---

### Step 20 — Connect Flask and Verify Network

```bash
docker network connect myapp-network flask-app
docker network inspect myapp-network | grep "Name"
```

---

#### ❌ Error: curl not found in flask image

```
OCI runtime exec failed: exec: "curl": executable file not found in $PATH
```

**Cause:** `python:3.9-slim` is a minimal image — no curl installed.

#### ✅ Fix: Use docker network inspect

```bash
docker network inspect myapp-network
# Confirmed flask-app, postgres, and redis all listed under Containers
```

---

### Step 20b — Verify Logs

```bash
docker logs postgres
# PostgreSQL 15 initialized, mydatabase created
# "database system is ready to accept connections"

docker logs redis
# Redis 7.4.8 started on port 6379
# "Ready to accept connections tcp"
```

---

### Step 21 — Monitor All Services

```bash
docker stats --no-stream --format \
  "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

---

## Cleanup

```bash
# Stop and remove all containers
docker stop $(docker ps -aq)
docker rm $(docker ps -aq)

# Remove unused volumes (anonymous only — named volumes are kept)
docker volume prune -f

# Remove unused networks (except defaults)
docker network prune -f

# Remove unused images
docker image prune -f
```

**After cleanup:**
- Containers → 0 remaining
- Networks → only `bridge`, `host`, `none`
- Named volumes retained (prune never removes named volumes)

---

## Errors Encountered & Fixes

| # | Error | Cause | Fix |
|---|-------|-------|-----|
| 1 | Cannot remove running container | `docker rm` called while container was still running | Run `docker stop` first, then `docker rm` |
| 2 | Container exited immediately in detached mode | Python output buffering — Flask not emitting logs | Added `-u` flag: `CMD ["python", "-u", "app.py"]` and rebuilt |
| 3 | Port 5000 already in use | macOS AirPlay Receiver occupies port 5000 by default | Used `-p 5001:5000` to map host port 5001 to container 5000 |
| 4 | `nslookup` not found in nginx image | Official nginx image is minimal — no DNS tools installed | Used `docker network inspect` to verify container IPs |
| 5 | `ping` not found in nginx image | Same reason — minimal nginx image | Connectivity already proven via `curl http://net-web2` |
| 6 | `curl` not found in flask image | `python:3.9-slim` has no curl | Used `docker network inspect` to confirm network membership |

---

## Key Takeaways

1. **Volumes persist data** beyond container lifecycle — always use named volumes for production data
2. **Environment variables** configure containers dynamically — use `.env` files for sensitive config
3. **Custom networks** provide built-in DNS resolution — containers communicate using names
4. **Monitoring commands** (`stats`, `logs`, `top`, `inspect`) help debug and optimize containers
5. **Minimal Docker images** (nginx, python-slim) may lack common tools like curl, ping, nslookup
6. **macOS port 5000** is reserved by AirPlay — use an alternative port when running Flask locally
7. **`docker volume prune`** only removes anonymous volumes — named volumes are always preserved
8. **Multi-container apps** should always use custom networks for isolation and service discovery

---

*End of Experiment 5*
