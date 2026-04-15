# Templates Reference

Copy-paste-ready templates for all configuration files needed in a Reflex GCP deployment. Replace placeholder values marked with `<PLACEHOLDER>`.

## Dockerfile (Backend)

```dockerfile
# Reflex Backend Dockerfile for Cloud Run
# Place in your app directory (e.g., reflex_app/Dockerfile)

FROM python:3.12

# --- Configuration ---
# REDIS_URL: Private IP of your Memorystore Redis instance
# Get with: gcloud redis instances describe reflex-redis --region=<REGION> --format="value(host)"
ENV REDIS_URL=redis://<REDIS_IP>:6379
# Prevent Python from buffering stdout/stderr (important for Cloud Run logs)
ENV PYTHONUNBUFFERED=1

WORKDIR /app

# Copy all application files
COPY . .

# Install Python dependencies
# If using uv: RUN pip install uv && uv sync
RUN pip install -r requirements.txt

# Run Reflex backend in production mode
# --backend-only: Frontend is served by Firebase Hosting
# --loglevel debug: Use for initial deployment, switch to info later
ENTRYPOINT ["reflex", "run", "--env", "prod", "--backend-only", "--loglevel", "debug"]
```

### Dockerfile Variations

**With uv package manager:**

```dockerfile
FROM python:3.12

ENV REDIS_URL=redis://<REDIS_IP>:6379 PYTHONUNBUFFERED=1

WORKDIR /app
COPY . .

RUN pip install uv && uv sync --no-dev

ENTRYPOINT ["reflex", "run", "--env", "prod", "--backend-only", "--loglevel", "debug"]
```

**With AUTH-enabled Redis:**

```dockerfile
FROM python:3.12

# Include AUTH string in Redis URL
ENV REDIS_URL=redis://:<AUTH_STRING>@<REDIS_IP>:6379
ENV PYTHONUNBUFFERED=1

WORKDIR /app
COPY . .
RUN pip install -r requirements.txt

ENTRYPOINT ["reflex", "run", "--env", "prod", "--backend-only", "--loglevel", "debug"]
```

**With database migrations:**

```dockerfile
FROM python:3.12

ENV REDIS_URL=redis://<REDIS_IP>:6379
ENV DB_URL=postgresql://<USER>:<PASS>@<DB_HOST>:5432/<DB_NAME>
ENV PYTHONUNBUFFERED=1

WORKDIR /app
COPY . .
RUN pip install -r requirements.txt

# Run migrations before starting
# Note: For Cloud Run, consider running migrations as a separate job
RUN reflex db migrate

ENTRYPOINT ["reflex", "run", "--env", "prod", "--backend-only", "--loglevel", "debug"]
```

### .dockerignore

Place in the same directory as your Dockerfile:

```
.web/
__pycache__/
*.pyc
.git/
.gitignore
.env
.venv/
node_modules/
*.egg-info/
dist/
build/
.pytest_cache/
.mypy_cache/
```

---

## cloudbuild.yaml (CI/CD)

```yaml
# Cloud Build configuration for Reflex backend deployment
# Place at: <APP_DIR>/cloudbuild.yaml
# Or: <APP_DIR>/cloudbuild/cloudbuild-staging.yaml

steps:
  # Build Docker image
  - name: 'gcr.io/cloud-builders/docker'
    id: Build
    args:
      - 'build'
      - '-t'
      - '$_AR_HOSTNAME/$PROJECT_ID/cloud-run-source-deploy/$REPO_NAME/$_SERVICE_NAME:$COMMIT_SHA'
      - '-f'
      - 'Dockerfile'
      - '.'
    dir: <APP_DIR>

  # Push to Artifact Registry
  - name: 'gcr.io/cloud-builders/docker'
    id: Push
    args:
      - push
      - >-
        $_AR_HOSTNAME/$PROJECT_ID/cloud-run-source-deploy/$REPO_NAME/$_SERVICE_NAME:$COMMIT_SHA
    dir: <APP_DIR>

  # Deploy to Cloud Run
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk:slim'
    args:
      - run
      - services
      - update
      - $_SERVICE_NAME
      - '--platform=managed'
      - >-
        --image=$_AR_HOSTNAME/$PROJECT_ID/cloud-run-source-deploy/$REPO_NAME/$_SERVICE_NAME:$COMMIT_SHA
      - >-
        --labels=managed-by=gcp-cloud-build-deploy-cloud-run,commit-sha=$COMMIT_SHA,gcb-build-id=$BUILD_ID,gcb-trigger-id=$_TRIGGER_ID
      - '--region=$_DEPLOY_REGION'
      - '--quiet'
    id: Deploy
    entrypoint: gcloud

images:
  - >-
    $_AR_HOSTNAME/$PROJECT_ID/cloud-run-source-deploy/$REPO_NAME/$_SERVICE_NAME:$COMMIT_SHA

options:
  automapSubstitutions: true
  logging: CLOUD_LOGGING_ONLY

substitutions:
  _DEPLOY_REGION: <REGION>
  _AR_HOSTNAME: <REGION>-docker.pkg.dev
  _TRIGGER_ID: <TRIGGER_ID>
  _PLATFORM: managed
  _SERVICE_NAME: <SERVICE_NAME>

tags:
  - gcp-cloud-build-deploy-cloud-run
  - gcp-cloud-build-deploy-cloud-run-managed
  - <SERVICE_NAME>
```

---

## firebase.json (Frontend Hosting)

```json
{
  "hosting": [
    {
      "site": "<FIREBASE_SITE>",
      "public": "<APP_DIR>/.web/_static",
      "ignore": [
        "firebase.json",
        "**/.*",
        "**/node_modules/**"
      ],
      "rewrites": [
        {
          "source": "404",
          "destination": "/404.html",
          "type": 301
        }
      ]
    }
  ]
}
```

---

## deploy-frontend.sh (Frontend Deploy Script)

```bash
#!/bin/bash
set -euo pipefail

# === Configuration ===
CLOUD_RUN_URL="<CLOUD_RUN_URL>"
FIREBASE_SITE="<FIREBASE_SITE>"
APP_DIR="<APP_DIR>"

# === Export ===
echo "Exporting Reflex frontend..."
cd "${APP_DIR}"
API_URL="${CLOUD_RUN_URL}" reflex export --frontend-only --no-zip

if [ $? -eq 0 ]; then
    echo "Deploying to Firebase Hosting..."
    cd ..
    firebase deploy --only "hosting:${FIREBASE_SITE}"
    echo "Deploy complete: https://${FIREBASE_SITE}.web.app"
else
    echo "ERROR: Reflex export failed. Aborting."
    exit 1
fi
```

---

## rxconfig.py (Reflex Configuration)

```python
import reflex as rx

config = rx.Config(
    app_name="<APP_NAME>",

    # For local development, this is overridden by API_URL env var in production
    # The frontend export uses API_URL to know where to find the backend
    api_url="http://localhost:8000",

    # Database (optional, for apps with database models)
    # db_url="postgresql://user:pass@localhost:5432/mydb",

    # Telemetry
    telemetry_enabled=False,
)
```

**Note:** In production, `API_URL` environment variable overrides `api_url` in config. The frontend export step sets this to the Cloud Run URL.

---

## requirements.txt (Minimal)

```
reflex>=0.6.0
```

Add your other dependencies below `reflex`. Common additions:

```
reflex>=0.6.0
sqlmodel>=0.0.14    # If using database models
httpx>=0.27.0       # If making HTTP requests from event handlers
python-dotenv>=1.0  # If using .env files locally
```

---

## Placeholder Reference

| Placeholder | Description | Example |
|-------------|-------------|---------|
| `<REDIS_IP>` | Memorystore Redis private IP | `10.60.48.3` |
| `<AUTH_STRING>` | Redis AUTH string (if enabled) | `abc123...` |
| `<REGION>` | GCP region | `us-central1` |
| `<SERVICE_NAME>` | Cloud Run service name | `perfwriter-reflex` |
| `<APP_DIR>` | Reflex app directory in repo | `reflex_app` |
| `<TRIGGER_ID>` | Cloud Build trigger ID | `0edafb45-0a21-4139-ad0f-8b58a7a85300` |
| `<FIREBASE_SITE>` | Firebase Hosting site name | `perfwriter-reflex` |
| `<CLOUD_RUN_URL>` | Cloud Run service URL | `https://perfwriter-reflex-881125452023.us-central1.run.app` |
| `<APP_NAME>` | Reflex app name (matches dir) | `my_app` |
| `<PROJECT_ID>` | GCP project ID | `perfwriter` |
