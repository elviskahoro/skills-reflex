# Deployment Reference

Complete reference for deploying Reflex apps: development, production, self-hosting, Docker, and Reflex Cloud.

## Development Server

```bash
# Start dev server with hot reload
reflex run
# Frontend: http://localhost:3000
# Backend: http://localhost:8000
```

## Production Build

```bash
# Build optimized production version
reflex run --env prod
```

Production mode creates an optimized static frontend build and runs the backend with production settings.

## Configuration (rxconfig.py)

```python
import reflex as rx

config = rx.Config(
    app_name="my_app",

    # Backend API URL — MUST be set for production/deployment
    # Frontend uses this to connect to the backend via WebSocket
    api_url="https://api.example.com:8000",

    # Database
    db_url="postgresql://user:pass@db-host:5432/mydb",

    # Frontend port (default 3000)
    frontend_port=3000,

    # Backend port (default 8000)
    backend_port=8000,

    # Telemetry
    telemetry_enabled=False,
)
```

**Critical:** `api_url` must be set to the server's publicly accessible address in production. Without this, the frontend cannot reach the backend.

### Environment Variables

Override config values with environment variables:

| Variable | Purpose |
|----------|---------|
| `API_URL` | Backend URL (overrides `api_url` in config) |
| `DB_URL` | Database connection string |
| `REDIS_URL` | Redis URL for production state manager |

## Self-Hosting

### Prerequisites

- Python 3.10+
- Node.js (for frontend compilation)
- Redis (for production state management)
- PostgreSQL (recommended over SQLite for production)
- Reverse proxy (Nginx) for WebSocket handling

### Step-by-Step

1. **Clone and install:**
```bash
git clone <your-repo>
cd your-app
pip install -r requirements.txt
# or: uv sync
```

2. **Configure `rxconfig.py`:**
```python
config = rx.Config(
    app_name="my_app",
    api_url="https://your-domain.com:8000",
    db_url="postgresql://user:pass@localhost:5432/mydb",
)
```

3. **Run database migrations:**
```bash
reflex db init  # First time only
reflex db migrate
```

4. **Build and run in production:**
```bash
reflex run --env prod
```

### Redis State Manager

The default in-memory state manager does not persist across restarts and does not support horizontal scaling. For production, use Redis:

```bash
# Install Redis
sudo apt install redis-server

# Set environment variable
export REDIS_URL="redis://localhost:6379"
```

Reflex automatically uses Redis when `REDIS_URL` is set. No code changes needed.

### Nginx Reverse Proxy

Nginx handles TLS termination and WebSocket proxying:

```nginx
# /etc/nginx/sites-available/my-reflex-app
server {
    listen 80;
    server_name your-domain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name your-domain.com;

    ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;

    # Frontend (Next.js)
    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Backend API and WebSocket
    location /api/ {
        proxy_pass http://localhost:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # WebSocket endpoint
    location /_event {
        proxy_pass http://localhost:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 86400;  # Keep WebSocket alive
    }
}
```

## Docker Deployment

### Dockerfile

```dockerfile
FROM python:3.12-slim

# Install Node.js (required for frontend build)
RUN apt-get update && apt-get install -y curl && \
    curl -fsSL https://deb.nodesource.com/setup_20.x | bash - && \
    apt-get install -y nodejs && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Initialize Reflex (downloads frontend dependencies)
RUN reflex init

# Build production frontend
RUN reflex export --no-zip

# Expose ports
EXPOSE 3000 8000

# Run in production mode
CMD ["reflex", "run", "--env", "prod"]
```

### Docker Compose

```yaml
version: "3.8"

services:
  app:
    build: .
    ports:
      - "3000:3000"
      - "8000:8000"
    environment:
      - API_URL=https://your-domain.com:8000
      - DB_URL=postgresql://user:pass@db:5432/mydb
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    volumes:
      - redisdata:/data

volumes:
  pgdata:
  redisdata:
```

## Static Export

Export frontend and backend as separate deployable artifacts:

```bash
reflex export
# Creates: frontend.zip and backend.zip
```

- **frontend.zip** — Static files, deployable to any CDN/static host (Netlify, Vercel, S3)
- **backend.zip** — Python backend, deployable to any server

This is useful when you want to host frontend and backend separately.

## Reflex Cloud

One-command deployment with managed infrastructure:

```bash
# Deploy to Reflex Cloud
reflex deploy
```

Features:
- Zero-config deployment
- Automatic scaling
- Built-in observability
- Custom domains (Enterprise)
- Free tier available

## Process Management (systemd)

For self-hosted production, use systemd to manage the Reflex process:

```ini
# /etc/systemd/system/reflex-app.service
[Unit]
Description=Reflex App
After=network.target redis.service postgresql.service

[Service]
Type=simple
User=www-data
WorkingDirectory=/opt/my-app
Environment=API_URL=https://your-domain.com:8000
Environment=DB_URL=postgresql://user:pass@localhost:5432/mydb
Environment=REDIS_URL=redis://localhost:6379
ExecStart=/opt/my-app/.venv/bin/reflex run --env prod
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable reflex-app
sudo systemctl start reflex-app
```

## Production Checklist

| Item | Action |
|------|--------|
| `api_url` set | Must point to public server address |
| Redis configured | Set `REDIS_URL` for state persistence |
| PostgreSQL | Switch from SQLite for production workloads |
| Migrations applied | Run `reflex db migrate` before starting |
| Nginx/reverse proxy | Handle TLS and WebSocket upgrade headers |
| Process manager | systemd, Docker, or supervisor for auto-restart |
| Static assets | Serve `assets/` via CDN if high traffic |
| Secrets management | Use env vars, never hardcode credentials |
| Backups | Database backup strategy in place |
| Monitoring | Application logs + health check endpoint |
