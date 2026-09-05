# Day 10 — Docker Command Reference

Complete command cheat-sheet from Classes 39–49 (Docker Fundamentals → Secure Docker Compose).
All labs run on AWS EC2 Ubuntu 22.04. Replace `<dockerhub-username>`, `<region>`, `<aws_account_id>` with your own values.

---

## 1. Install Docker (Ubuntu)

```bash
sudo apt update
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER   # run docker without sudo
newgrp docker                    # apply group change (or re-login)
docker version
```

Install Docker Compose:

```bash
sudo apt install -y docker-compose
```

---

## 2. Images

```bash
docker pull nginx                          # download image from Docker Hub
docker images                              # list local images
docker rmi -f <image_id>                   # force-remove an image
docker build -t flask-audit-app:latest .   # build image from Dockerfile in cwd
docker tag flask-audit-app:latest <dockerhub-username>/flask-audit-app:v1
docker history <image>                     # inspect layer sizes
```

---

## 3. Containers — Full Lifecycle

```bash
docker run -d -p 8080:80 --name webserver nginx   # detached, port map, name
docker ps                                         # running containers
docker ps -a                                      # all containers (incl. stopped)
docker stop webserver
docker rm webserver
docker logs webserver                             # view container logs
docker logs <container_id>
docker exec -it flaskapp /bin/bash                # shell inside a running container
docker inspect <container_name_or_id>             # full JSON details (IP, mounts, network)
hostname -I                                       # (inside container) get its IP
```

| Flag | Meaning |
|------|---------|
| `-d` | detached (background) |
| `-p host:container` | port mapping |
| `--name` | container name |
| `-e KEY=value` | environment variable |
| `--env-file .env` | load env vars from file |
| `-v vol:/path` | mount volume |
| `--network <net>` | attach to a network |
| `-it` | interactive + TTY (with exec) |

---

## 4. Registry Workflows

### Docker Hub

```bash
docker login
docker tag django-notes-app <dockerhub-username>/django-notes-app:latest
docker push <dockerhub-username>/django-notes-app:latest
```

### GitHub Container Registry (GHCR)

```bash
# 1. Create a PAT at https://github.com/settings/tokens
#    scopes: write:packages, read:packages, delete:packages (optional)
echo <YOUR_GITHUB_PAT> | docker login ghcr.io -u <YOUR_GITHUB_USERNAME> --password-stdin
docker tag my-image ghcr.io/<your-username>/my-image:latest
docker push ghcr.io/<your-username>/my-image:latest
```

### AWS ECR

```bash
# Install AWS CLI v2
sudo apt update && sudo apt install unzip curl -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip awscliv2.zip
sudo ./aws/install
aws --version

# Configure credentials (IAM user with ECR + EC2 access)
aws configure   # Access Key ID / Secret Key / region / json

aws ecr create-repository --repository-name django-notes-app
aws ecr get-login-password --region <region> | docker login --username AWS \
  --password-stdin <aws_account_id>.dkr.ecr.<region>.amazonaws.com
docker tag django-notes-app:latest \
  <aws_account_id>.dkr.ecr.<region>.amazonaws.com/django-notes-app:latest
docker push <aws_account_id>.dkr.ecr.<region>.amazonaws.com/django-notes-app:latest
docker pull <aws_account_id>.dkr.ecr.<region>.amazonaws.com/django-notes-app:latest
```

---

## 5. Lab 1 — NGINX Lifecycle (Class 39)

```bash
docker pull nginx
docker images
docker run -d -p 8080:80 --name webserver nginx
docker ps
# test http://localhost:8080 (or EC2 public IP) → NGINX welcome page
docker stop webserver
docker rm webserver
docker run -d -p 8080:80 --name webserver nginx
docker logs webserver
```

EC2 security group: open **TCP 8080**.

---

## 6. Lab — Flask App on Docker (Class 40)

```bash
git clone https://github.com/Umair1012/Flask-Based-Python-Application-on-AWS-EC2-with-Docker.git
cd Flask-Based-Python-Application-on-AWS-EC2-with-Docker

# Manual run first (venv)
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
python app.py 0.0.0.0:8000

# Dockerized
docker build -t flask-audit-app:latest .
docker login
docker tag flask-audit-app:latest <dockerhub-username>/flask-audit-app:v1
docker push <dockerhub-username>/flask-audit-app:v1
docker run -d -p 8000:8000 <dockerhub-username>/flask-audit-app:v1
# test http://<EC2-PUBLIC-IP>:8000
```

EC2 security group: **TCP 8000**.

---

## 7. Lab — Django Notes App on Docker (Class 41)

```bash
git clone https://github.com/Umair1012/django-notes-app.git
cd django-notes-app

# Manual run first
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
python manage.py migrate
python manage.py runserver 0.0.0.0:8000

# Background options (production-like)
tmux        # run, then Ctrl+B, D to detach
nohup python manage.py runserver 0.0.0.0:8000 &

# Dockerized
docker build -t django-notes-app .
docker tag django-notes-app <dockerhub-username>/django-notes-app:latest
docker push <dockerhub-username>/django-notes-app:latest
docker run -d -p 8000:8000 <dockerhub-username>/django-notes-app:latest
```

EC2 security group: **TCP 8000**.

---

## 8. Lab — Two-Tier Flask + MySQL, Manual Deploy (Class 42)

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install python3-pip python3-venv mysql-server -y
sudo apt install python3-dev default-libmysqlclient-dev build-essential -y

git clone https://github.com/Umair1012/two-tier-flask-app.git
cd two-tier-flask-app
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
```

MySQL setup:

```bash
sudo systemctl start mysql
sudo mysql -u root
```

```sql
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'yourpassword';
FLUSH PRIVILEGES;
CREATE DATABASE flaskapp;
USE flaskapp;
CREATE TABLE feedback (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100),
    message TEXT
);
EXIT;
```

Environment + run:

```bash
# .env in project root
MYSQL_HOST=127.0.0.1
MYSQL_USER=root
MYSQL_PASSWORD=yourpassword
MYSQL_DB=flaskapp
```

```bash
python app.py --host=0.0.0.0 --port=5000
# verify data
sudo mysql -u root -p -e "USE flaskapp; SELECT * FROM feedback;"
```

EC2 security group: **TCP 5000** (+22 SSH).

---

## 9. Lab — Docker Volumes (Class 43)

```bash
git clone https://github.com/Umair1012/django-todo.git
cd django-todo
```

Volume types: **named** (Docker-managed, persistent), **anonymous** (temporary), **bind mount** (host dir mapped in).

```bash
docker volume create django-data
docker volume ls
docker volume inspect django-data
docker volume rm <volume>

# Build + push to GHCR
docker build -t <dockerhub-username>/django-todo .
echo <GITHUB_PAT> | docker login ghcr.io -u <github-username> --password-stdin
docker tag <dockerhub-username>/django-todo ghcr.io/<github-username>/django-todo:latest
docker push ghcr.io/<github-username>/django-todo:latest

# Run with volume
docker run -d \
  --name todo-app \
  -v django-data:/app/db.sqlite3 \
  -p 8000:8000 \
  <dockerhub-username>/django-todo
```

**Persistence proof:**

```bash
docker stop todo-app && docker rm todo-app
docker run -d --name todo-app -v django-data:/app/db.sqlite3 -p 8000:8000 \
  <dockerhub-username>/django-todo
# TODOs created before are still there — data lives outside the container
```

---

## 10. Lab — Docker Networks (Class 44)

Network types: **bridge** (default, single host), **host** (shares host stack), **none** (isolated), **overlay** (multi-host, Swarm).

```bash
docker network create flask-net
docker network ls
docker network inspect flask-net

# MySQL container on the custom network
docker run -d \
  --name mysql \
  --network flask-net \
  -v mysql-data:/var/lib/mysql \
  -e MYSQL_DATABASE=mydb \
  -e MYSQL_ROOT_PASSWORD=admin \
  -p 3306:3306 \
  mysql:8.0

# Flask app container — talks to DB by container NAME via Docker DNS
docker run -d \
  --name flaskapp \
  --network flask-net \
  -e MYSQL_HOST=mysql \
  -e MYSQL_USER=root \
  -e MYSQL_PASSWORD=admin \
  -e MYSQL_DB=mydb \
  -p 5000:5000 \
  <dockerhub-username>/two-tier-flask-app
```

Connectivity test:

```bash
docker exec -it flaskapp /bin/bash
apt update && apt install iputils-ping -y
ping mysql          # resolves by name on user-defined networks
ping 172.18.0.2     # or by IP from docker inspect
hostname -I         # container's own IP
exit
```

Verify:

```bash
docker ps
docker logs flaskapp    # expect "Connected to DB at mysql"
# test http://<EC2-PUBLIC-IP>:5000
```

EC2 security group: **TCP 5000** + **TCP 3306** (3306 only for testing — restrict in production).

---

## 11. Lab — Multi-Stage Build + ECR (Class 45)

```bash
git clone https://github.com/Umair1012/django-notes-app.git
cd django-notes-app
# write multistage Dockerfile (see README.md), then:
docker build -t <dockerhub-username>/django-notes-app .

# Push to ECR (workflow in section 4)
aws ecr create-repository --repository-name django-notes-app
aws ecr get-login-password --region <region> | docker login --username AWS \
  --password-stdin <aws_account_id>.dkr.ecr.<region>.amazonaws.com
docker tag django-notes-app:latest \
  <aws_account_id>.dkr.ecr.<region>.amazonaws.com/django-notes-app:latest
docker push <aws_account_id>.dkr.ecr.<region>.amazonaws.com/django-notes-app:latest

# Run
docker run -d --name notes-app -p 8000:8000 <dockerhub-username>/django-notes-app
```

Why multi-stage: smaller images (build tools never ship), faster deploys, smaller attack surface, clean build/run separation.

---

## 12. Lab — Secure Dockerfile + Secure Runtime (Class 46)

Build the secure image (non-root `appuser`, gunicorn — Dockerfile in README.md), then run hardened:

```bash
docker build -t django-notes-app .

docker run -d \
  --name notes-app \
  --restart unless-stopped \
  --user 1000:1000 \
  --read-only \
  --tmpfs /tmp \
  --cap-drop=ALL \
  --security-opt no-new-privileges \
  --memory=512m \
  --cpus=1.0 \
  -p 8000:8000 \
  -v "$(pwd)/logs:/app/logs:rw" \
  --health-cmd="curl -f http://localhost:8000/health || exit 1" \
  --health-interval=30s \
  --health-retries=3 \
  --health-start-period=10s \
  django-notes-app:latest

docker ps      # STATUS shows "(healthy)"
```

Push to ECR (section 4). Extra hardening checklist:

- `.dockerignore`: `.env`, `__pycache__`, `.git`
- Secrets via environment variables, never baked into the image
- HTTPS with Nginx + Certbot in production
- Scan images with **Trivy** or **Dockle** in CI/CD
- Database in a separate container/host (PostgreSQL or managed)

---

## 13. Lab — Three-Tier App + Compose Intro (Class 47)

Concepts: **YAML** (human-readable config format — used by Compose, K8s, Ansible, GitHub Actions), **microservices** (small independent loosely-coupled services), **three-tier** (frontend → backend → database).

```bash
git clone https://github.com/Umair1012/three-tier-app.git
cd three-tier-app
cd frontend && npm start          # React
cd ../backend && node server.js   # Node API
# backend/.env → MONGO_URI=mongodb+srv://<user>:<pass>@cluster0.mongodb.net/three-tier-db
```

First Compose file — single NGINX service:

```yaml
version: '3.8'
services:
  nginx:
    image: nginx:latest
    container_name: nginx-container
    ports:
      - "80:80"
    restart: always
```

```bash
docker-compose up -d
docker-compose down
```

---

## 14. Lab — Docker Compose MERN Stack (Class 48)

Dockerfiles per service (`frontend/Dockerfile`, `backend/Dockerfile` — see README.md), MongoDB uses the official image. Compose file in README.md.

```bash
git clone https://github.com/Umair1012/three-tier-app.git
cd three-tier-app

docker-compose up -d          # start the whole stack
docker-compose ps             # status of services
docker-compose logs frontend  # logs of one service
docker-compose down           # stop & remove

# validate
curl http://<EC2_PUBLIC_IP>:5000      # → "Node API Working"
# browser: http://<EC2_PUBLIC_IP>:3000 → React app
```

Key Compose keywords: `build` (context), `image`, `ports`, `volumes`, `env_file`, `depends_on`, named top-level `volumes:` (e.g. `mongo-data:/data/db`).

EC2 security group: **TCP 3000, 5000, 27017** (27017 only for debugging — keep Mongo private in production).

---

## 15. Lab — Secure Compose Deployment (Class 49)

Multi-stage Alpine Dockerfiles + non-root users + healthchecks + private bridge network (files in README.md).

```bash
git clone https://github.com/Umair1012/three-tier-app.git
cd three-tier-app

docker-compose up --build -d

# reset everything and rebuild
docker-compose down --volumes --remove-orphans
docker-compose build
docker-compose up -d

docker-compose logs frontend    # debug if a service fails
```

Security best practices applied:

- Multi-stage builds (build tools never reach production)
- Alpine-based minimal base images
- Non-root users (`user: "1000:1000"`, `USER appuser`)
- `read_only: true` root filesystems
- Healthchecks on every service
- Dedicated bridge network (`app-network`), Mongo reachable internally only
- `restart: always`

---

## 16. Cleanup & Housekeeping

```bash
docker system df                 # disk usage by images/containers/volumes
docker container prune           # remove stopped containers
docker image prune -a            # remove unused images
docker volume prune              # remove unused volumes
docker network prune             # remove unused networks
docker system prune -a --volumes # full cleanup (careful!)
```

---

**Prepared by:** Fahad Ali Seemab — 20-Day DevOps Journey, Day 10 (2026-09-04)
