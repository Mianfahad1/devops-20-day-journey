# Day 10 Docker Command Reference

The commands below assume Docker Engine and the Compose plugin are installed and
that each application is being run from its project directory.

## Docker fundamentals

```bash
docker version
docker info
docker pull nginx:latest
docker run -d --name web -p 8080:80 nginx:latest
docker ps
docker logs web
docker exec -it web sh
docker stop web
docker rm web
docker image ls
```

Open `http://localhost:8080` to verify the NGINX container, then clean it up:

```bash
docker rm -f web
```

## Build and run an application image

```bash
docker build -t my-app:1.0 .
docker run --rm -p 8000:8000 my-app:1.0
docker tag my-app:1.0 <dockerhub-user>/my-app:1.0
docker login
docker push <dockerhub-user>/my-app:1.0
```

Do not put passwords, tokens, `.env` files, `.git`, virtual environments, or
dependency caches in an image. Add them to `.dockerignore`.

## Volumes and networks

```bash
docker volume create app-data
docker run -d --name db -v app-data:/var/lib/mysql mysql:8
docker volume inspect app-data

docker network create app-net
docker run -d --name mysql --network app-net mysql:8
docker run --rm -it --network app-net busybox ping -c 3 mysql
```

Containers on a user-defined bridge network resolve one another by container
name. Use a named volume for data that must survive container replacement.

## Compose

```bash
docker compose config
docker compose build
docker compose up -d
docker compose ps
docker compose logs -f backend
docker compose exec backend sh
docker compose down
docker compose down -v
```

Use `docker compose down -v` only when deleting the named database volume is
intentional.

## ECR and GHCR

```bash
# AWS ECR
aws ecr get-login-password --region <aws-region> |
  docker login --username AWS --password-stdin <account>.dkr.ecr.<aws-region>.amazonaws.com
docker tag my-app:1.0 <account>.dkr.ecr.<aws-region>.amazonaws.com/my-app:1.0
docker push <account>.dkr.ecr.<aws-region>.amazonaws.com/my-app:1.0

# GitHub Container Registry
echo "$CR_PAT" | docker login ghcr.io -u <github-user> --password-stdin
docker tag my-app:1.0 ghcr.io/<github-user>/my-app:1.0
docker push ghcr.io/<github-user>/my-app:1.0
```

Use a short-lived token or secret manager for registry credentials; never
commit them to the repository.

## Inspect and clean up

```bash
docker inspect <container>
docker stats
docker system df
docker image prune
docker container prune
```

Review the output before running prune commands because they remove unused
Docker resources.
