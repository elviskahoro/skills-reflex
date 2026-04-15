---
name: reflex-docker-deploy
description: Use when self-hosting a Reflex app with Docker and Docker Compose, creating Dockerfiles for Reflex backend/frontend, configuring nginx as a reverse proxy, deploying to EC2 or any VPS, or pushing Reflex Docker images to a container registry (ECR, Docker Hub).
license: Proprietary
compatibility: Requires Docker and Docker Compose installed, Python 3.11+, a working Reflex app that runs locally with `reflex run`, and a requirements.txt with all Python dependencies.
metadata:
  author: elviskahoro
  version: "1.0"
  tags: [reflex, docker, self-hosting, nginx, deployment, devops, ec2, compose]
---

# Self-Hosting Reflex with Docker

Containerized deployment of a Reflex app using Docker Compose with a split architecture: nginx-served static frontend, Python backend running in `--backend-only` mode, and Redis for state management.

**Architecture overview:**

```
Internet → nginx (port 80/443)
              ├── Static files (frontend export)
              ├── /_event → proxy to backend:8000 (WebSocket)
              ├── /ping → proxy to backend:8000
              └── /_upload → proxy to backend:8000
           Reflex backend (--backend-only, port 8000)
              ↕
           Redis (state manager)
```

The frontend is a static export (`reflex export --frontend-only --no-zip`) served by nginx inside a container. The backend runs `reflex run --env prod --backend-only`. Redis handles state persistence across requests. All three run as separate Docker Compose services.

## When to Use

- Self-hosting a Reflex app on your own infrastructure with Docker
- Deploying a Reflex app to EC2, a VPS, or any Docker-capable host
- Creating Docker images for a Reflex app (backend + frontend)
- Setting up nginx as a reverse proxy for Reflex WebSocket and API routes
- Pushing Reflex Docker images to ECR, Docker Hub, or any container registry
- Running a Reflex app locally via Docker Compose for testing

## Prerequisites

Before starting, verify:

1. **Working Reflex app locally** — `reflex run` succeeds with no errors
2. **Docker Compose installed** — `docker compose version` works
3. **`requirements.txt` exists** — in the top-level app directory with all Python dependencies

## Required Files

All 4 files go at the **same level as `rxconfig.py`**:

```
{app_name}/
├── .web/
├── assets/
├── {app_name}/
│   ├── __init__.py
│   └── {app_name}.py
├── compose.yml          ← NEW
├── Dockerfile           ← NEW
├── nginx.conf           ← NEW
├── web.Dockerfile       ← NEW
├── requirements.txt
└── rxconfig.py
```

## Execution Steps for Agents

### Step 1: Create the Dockerfile (Backend)

This builds the Reflex backend container. It runs in `--backend-only` mode since nginx serves the frontend.

```dockerfile
FROM python:3.12

ENV REDIS_URL=redis://redis PYTHONUNBUFFERED=1

WORKDIR /app
COPY . .

RUN pip install -r requirements.txt

ENTRYPOINT ["reflex", "run", "--env", "prod", "--backend-only", "--loglevel", "debug" ]
```

**Key details:**
- `REDIS_URL=redis://redis` — Docker Compose makes services accessible by service name, so `redis` resolves to the Redis container
- `--backend-only` — essential, the frontend is served separately by nginx
- `--loglevel debug` — recommended for initial deployment, switch to `info` later
- Container listens on port **8000** (Reflex backend default)

### Step 2: Create web.Dockerfile (Frontend)

This is a multi-stage build: first exports the frontend static files, then serves them with nginx.

```dockerfile
FROM python:3.12 AS builder

WORKDIR /app

COPY . .
RUN pip install -r requirements.txt
RUN reflex export --frontend-only --no-zip

FROM nginx

COPY --from=builder /app/.web/_static /usr/share/nginx/html
COPY ./nginx.conf /etc/nginx/conf.d/default.conf
```

**Key details:**
- Stage 1 (`builder`): installs deps and runs `reflex export --frontend-only --no-zip` to generate static files in `.web/_static`
- Stage 2: copies the static files into the nginx html directory and applies custom nginx config
- No explicit ENTRYPOINT needed — the nginx image has its own

**For remote deployment** (when the backend URL differs from localhost), add `API_URL` before the export:

```dockerfile
FROM python:3.12 AS builder

WORKDIR /app

COPY . .
ENV API_URL=http://ec2-XX-XXX-XXX-XX.us-west-2.compute.amazonaws.com
RUN pip install -r requirements.txt
RUN reflex export --frontend-only --no-zip

FROM nginx

COPY --from=builder /app/.web/_static /usr/share/nginx/html
COPY ./nginx.conf /etc/nginx/conf.d/default.conf
```

### Step 3: Create nginx.conf

nginx serves the static frontend and proxies WebSocket/API requests to the backend.

```nginx
server {
 listen 80;
 listen  [::]:80;
 server_name frontend;

 error_page   404  /404.html;

 location /_event {
    proxy_set_header   Connection "upgrade";
    proxy_pass http://backend:8000;
    proxy_http_version 1.1;
    proxy_set_header   Upgrade $http_upgrade;
 }

 location /ping {
    proxy_pass http://backend:8000;
 }

 location /_upload {
    proxy_pass http://backend:8000;
 }

 location / {
   root /usr/share/nginx/html;
 }
}
```

**Key details:**
- `/_event` — WebSocket endpoint, requires `Connection "upgrade"` and `Upgrade` headers
- `/ping` — health check endpoint proxied to backend
- `/_upload` — file upload endpoint proxied to backend
- `/` — serves static frontend files from the nginx html directory
- `http://backend:8000` — uses Docker Compose service name resolution

### Step 4: Create compose.yml

Orchestrates all three services with correct startup order.

```yaml
services:
  backend:
    build:
      dockerfile: Dockerfile
    ports:
     - 8000:8000
    depends_on:
     - redis
  frontend:
    build:
      dockerfile: web.Dockerfile
    ports:
      - 3000:80
    depends_on:
      - backend
  redis:
    image: redis
```

**Key details:**
- **Startup order:** redis → backend → frontend (via `depends_on`)
- Backend exposes port 8000 externally, maps to 8000 internally
- Frontend exposes port 3000 externally, maps to port 80 internally (nginx default)
- Redis uses the official image with default settings

### Step 5: Run Locally

```bash
# Build and start all services
docker compose up

# Visit the app
# http://localhost:3000

# Rebuild after code changes
docker compose up --build
```

**Note:** Docker Desktop must be running, or you'll see: `Cannot connect to the Docker daemon at unix:///var/run/docker.sock`

## Remote Deployment (EC2 / VPS)

### Step 6: Build and Push Images to a Registry

Build images with the target platform specified:

```bash
# Build backend image
docker build . -f Dockerfile \
  -t 83XXXXXXXX86.dkr.ecr.us-west-2.amazonaws.com/example_app/backend:v0.1 \
  --platform linux/amd64

# Build frontend image (make sure API_URL is set in web.Dockerfile)
docker build . -f web.Dockerfile \
  -t 83XXXXXXXX86.dkr.ecr.us-west-2.amazonaws.com/example_app/frontend:v0.1 \
  --platform linux/amd64
```

Push to registry:

```bash
# Authenticate with ECR
docker login -u AWS -p $(aws ecr get-login-password --region us-west-2) \
  83XXXXXXXX86.dkr.ecr.us-west-2.amazonaws.com

# Push images
docker push 83XXXXXXXX86.dkr.ecr.us-west-2.amazonaws.com/example_app/backend:v0.1
docker push 83XXXXXXXX86.dkr.ecr.us-west-2.amazonaws.com/example_app/frontend:v0.1
```

**For Docker Hub instead of ECR:**

```bash
docker build . -f Dockerfile -t yourusername/myapp-backend:v0.1 --platform linux/amd64
docker build . -f web.Dockerfile -t yourusername/myapp-frontend:v0.1 --platform linux/amd64
docker login
docker push yourusername/myapp-backend:v0.1
docker push yourusername/myapp-frontend:v0.1
```

### Step 7: Create Remote compose.yml

On the remote server, create a `compose.yml` that pulls images instead of building:

```yaml
services:
  backend:
    image: XXXXXX.dkr.ecr.us-west-2.amazonaws.com/example_app/backend:v0.1
    entrypoint: ["reflex", "run", "--env", "prod", "--backend-only", "--loglevel", "debug" ]
    depends_on:
     - redis
  frontend:
    image: XXXXX.dkr.ecr.us-west-2.amazonaws.com/example_app/frontend:v0.1
    ports:
      - 80:80
    depends_on:
      - backend
  redis:
    image: redis
```

**Key differences from local compose.yml:**
- Uses `image:` instead of `build:` — pulls pre-built images
- Frontend maps to port **80** (production) instead of 3000
- Backend `entrypoint` is explicit since there's no Dockerfile context
- No backend port mapping needed (nginx proxies internally)

### Step 8: Set Up EC2 and Run

```bash
# On the EC2 instance (Amazon Linux)
sudo su
yum install docker -y

# Install docker-compose
curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) \
  -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# Start Docker
systemctl start docker

# Authenticate with ECR
aws configure  # provide access key ID and secret
aws ecr get-login-password --region us-west-2 | \
  docker login --username AWS --password-stdin XXXXXX.dkr.ecr.us-west-2.amazonaws.com

# Create compose.yml and start
vi compose.yml  # paste the remote compose.yml
docker-compose up -d
```

## Adding SSL/TLS (Production)

For production, add TLS termination. See [PRODUCTION.md](references/PRODUCTION.md) for detailed SSL setup options including:
- Caddy as a reverse proxy (automatic HTTPS)
- Certbot with nginx (Let's Encrypt)
- AWS ALB with ACM certificates

## Common Mistakes to Avoid

| Mistake | Problem | Fix |
|---------|---------|-----|
| Missing `--backend-only` in Dockerfile | Backend tries to serve frontend too | Add `--backend-only` to ENTRYPOINT |
| No `API_URL` in web.Dockerfile for remote | Frontend can't find backend, WebSocket fails | Set `ENV API_URL=http://your-server-address` before `reflex export` |
| Wrong nginx proxy target | 502 Bad Gateway | Use `http://backend:8000` (Docker service name) |
| Missing WebSocket headers in nginx | `/_event` connection fails | Include `Connection "upgrade"` and `Upgrade $http_upgrade` headers |
| Docker Desktop not running | `Cannot connect to Docker daemon` | Start Docker Desktop before running `docker compose` |
| Not rebuilding after changes | Old code still running | Use `docker compose up --build` |
| `--platform` not specified for remote | Image won't run on target host | Add `--platform linux/amd64` when building for EC2/VPS |
| Port 3000 blocked on remote | Can't access the app | Use port 80 in remote compose.yml, ensure security group allows it |
| `requirements.txt` missing deps | `ModuleNotFoundError` in container | Run `pip freeze > requirements.txt` before building |

## Quick Reference

| Task | Command |
|------|---------|
| Start locally | `docker compose up` |
| Rebuild and start | `docker compose up --build` |
| Start detached | `docker compose up -d` |
| View logs | `docker compose logs -f` |
| Stop all services | `docker compose down` |
| Build backend image | `docker build . -f Dockerfile -t name:tag` |
| Build frontend image | `docker build . -f web.Dockerfile -t name:tag` |
| Push image | `docker push name:tag` |
| ECR login | `aws ecr get-login-password --region REGION \| docker login --username AWS --password-stdin ACCOUNT.dkr.ecr.REGION.amazonaws.com` |

## References

- [PRODUCTION.md](references/PRODUCTION.md) — SSL/TLS setup, Caddy, Certbot, database configuration
- [TEMPLATES.md](references/TEMPLATES.md) — Copy-paste ready templates for all 4 files
- [Reflex Self-Hosting Blog Post](https://reflex.dev/blog/self-hosting-reflex-with-docker/) — Original tutorial by Tom Gotsman
