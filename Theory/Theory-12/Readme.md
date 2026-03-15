# Docker Compose Multi-Container Experiment

## Objective

To understand and implement Docker Compose for managing multi-container applications using WordPress and MySQL.  
This experiment demonstrates container orchestration, networking, volumes, scaling, restart policies, and resource limiting.

---

## What is Docker Compose?

Docker Compose is a tool used to define and manage multi-container Docker applications using a YAML configuration file (`docker-compose.yml`).

Instead of running multiple `docker run` commands, we define everything declaratively in one file.

---

## Architecture Used

This experiment uses:

- WordPress (Frontend Application)
- MySQL (Database Server)

WordPress communicates with MySQL using Docker's internal network.


---

## docker-compose.yml Configuration

```yaml
services:
  mysql:
    image: mysql:8.0
    container_name: mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: wppass
    volumes:
      - mysql_data:/var/lib/mysql

  wordpress:
    image: wordpress:latest
    restart: always
    mem_limit: 512m
    cpus: 0.5
    environment:
      WORDPRESS_DB_HOST: mysql
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: wppass
      WORDPRESS_DB_NAME: wordpress
    depends_on:
      - mysql
    volumes:
      - wp_content:/var/www/html

volumes:
  mysql_data:
  wp_content:
```

# Docker Compose vs Docker Run – Complete DevOps Lab Experiment

---

# 📌 Experiment Title

Implementation and Comparison of Docker Run and Docker Compose for Multi-Container Application Deployment

---

# 🎯 Objective

1. To understand Docker Run and Docker Compose.
2. To compare imperative vs declarative container execution.
3. To deploy a multi-container application (WordPress + MySQL).
4. To demonstrate networking, volumes, scaling, restart policy, and resource limiting.
5. To understand lifecycle management of containers.

---

# 🐳 PART 1 – Docker Run (Imperative Approach)

Docker Run follows an imperative approach.

We manually specify all options in a long command.

## Example: Running Nginx Using Docker Run

```bash
docker run \
  --name my-nginx \
  -p 8080:80 \
  -v ./html:/usr/share/nginx/html \
  -e NGINX_HOST=localhost \
  --restart unless-stopped \
  -d \
  nginx:alpine
```

### Explanation of Flags

| Flag | Meaning |
|------|---------|
| --name | Assign container name |
| -p | Port mapping |
| -v | Volume mount |
| -e | Environment variable |
| --restart | Restart policy |
| -d | Detached mode |

Problem:
- Long command
- Hard to maintain
- Difficult for multi-container apps

---

# 🐳 PART 2 – Docker Compose (Declarative Approach)

Docker Compose uses a YAML file.

Instead of telling Docker HOW to run,
we declare WHAT we want.

---

# 📂 Step 1 – Create Project Folder

```bash
mkdir docker-compose-demo
cd docker-compose-demo
```

---

# 📄 Step 2 – Create docker-compose.yml

```bash
touch docker-compose.yml
```

---

# 🚀 Step 3 – Run Single Container (Nginx)

```yaml
services:
  nginx:
    image: nginx:alpine
    container_name: my-nginx
    ports:
      - "8080:80"
```

Run:

```bash
docker compose up -d
```

Visit:

http://localhost:8080

---

# ⚠️ Error Faced – Docker Daemon Not Running

Error:
Cannot connect to the Docker daemon.

Solution:
Start Docker Desktop.

---

# ⚠️ Error Faced – ARM64 MySQL Issue (M1/M2 Mac)

Error:
no matching manifest for linux/arm64/v8

Reason:
mysql:5.7 does not support ARM architecture.

Solution:
Use mysql:8.0

---

# 🏗 PART 3 – Multi-Container Application

We deployed WordPress with MySQL.

---

# 📝 Final docker-compose.yml Used

```yaml
services:
  mysql:
    image: mysql:8.0
    container_name: mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: wppass
    volumes:
      - mysql_data:/var/lib/mysql

  wordpress:
    image: wordpress:latest
    restart: always
    mem_limit: 512m
    cpus: 0.5
    environment:
      WORDPRESS_DB_HOST: mysql
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: wppass
      WORDPRESS_DB_NAME: wordpress
    depends_on:
      - mysql
    volumes:
      - wp_content:/var/www/html

volumes:
  mysql_data:
  wp_content:
```

---

# 🚀 Step 4 – Run Multi-Container Setup

```bash
docker compose up -d
```

WordPress connects to MySQL automatically using internal Docker network.

---

# 🌐 Networking Concept

Docker Compose automatically creates a bridge network.

Containers communicate using service names.

Example:

WORDPRESS_DB_HOST: mysql

No need for manual IP configuration.

---

# 💾 Volume Persistence

Volumes created:

- mysql_data
- wp_content

Test:

```bash
docker compose down
docker compose up -d
```

Data remains intact.

---

# 📊 Step 5 – Scaling Containers

Removed port mapping.

Scaled WordPress:

```bash
docker compose up -d --scale wordpress=2
```

Result:
- wordpress-1
- wordpress-2
- mysql

Scaling improves availability and load distribution.

---

# 🔁 Step 6 – Restart Policy

```yaml
restart: always
```

Test:

```bash
docker stop <container-name>
```

Container restarts automatically.

Restart Policies:

| Policy | Meaning |
|--------|----------|
| no | Default |
| always | Always restart |
| on-failure | Restart on error |
| unless-stopped | Restart unless manually stopped |

---

# ⚙️ Step 7 – Resource Limiting

```yaml
mem_limit: 512m
cpus: 0.5
```

Meaning:
- Maximum 512MB RAM
- Half CPU core

Verify:

```bash
docker inspect <container-name>
docker stats
```

Prevents one container from consuming full system resources.

---

# 🖥 Step 8 – Access Containers

Enter WordPress:

```bash
docker exec -it wordpress bash
```

Enter MySQL:

```bash
docker exec -it mysql bash
mysql -u root -p
```

---

# 📜 Step 9 – View Logs

```bash
docker compose logs
docker compose logs mysql
docker compose logs wordpress
```

Used for debugging.

---

# 🧹 Step 10 – Cleanup

Remove containers and volumes:

```bash
docker compose down -v
```

Remove unused images:

```bash
docker system prune -a
```

---

# Docker Run vs Docker Compose Comparison

| Docker Run | Docker Compose |
|------------|----------------|
| Imperative | Declarative |
| Single container | Multi-container |
| Long commands | YAML configuration |
| Hard to maintain | Easy lifecycle |
| No dependency handling | Supports depends_on |

---

# Key Concepts Learned

- Imperative vs Declarative approach
- Multi-container orchestration
- Service-to-service networking
- Volume persistence
- Scaling
- Restart policy
- Resource limiting
- Lifecycle management
- ARM compatibility issues
- Debugging using logs and exec

---

# Conclusion

Docker Compose simplifies the management of multi-container applications by allowing configuration through a single YAML file.

Compared to Docker Run, it:
- Improves maintainability
- Enhances scalability
- Enables automatic networking
- Provides easier lifecycle control
- Supports production-ready configurations

Docker Compose is essential for modern DevOps workflows and microservices architecture.

---
