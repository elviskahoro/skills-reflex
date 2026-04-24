---
name: reflex-gcp-deployment
description: Use when deploying a Reflex app to Google Cloud Platform, setting up Cloud Run for the backend, Firebase Hosting for the frontend, configuring VPC networking with Redis, or creating Cloud Build CI/CD triggers for a Reflex project.
license: Proprietary
compatibility: Requires gcloud CLI authenticated with a GCP project, firebase-tools CLI, Docker, Python 3.11+, reflex pip package, and a working Reflex app that runs locally with `reflex run`.
metadata:
  author: elviskahoro
  version: "1.0"
  tags: [reflex, gcp, cloud-run, firebase, redis, deployment, devops, ci-cd]
---

# Deploying Reflex to Google Cloud Platform

Production deployment of a Reflex app on GCP using a split architecture: static frontend on Firebase Hosting, backend on Cloud Run inside a VPC with managed Redis for state.

**Architecture overview:**

```
Internet → Firebase Hosting (static frontend)
              ↕ HTTPS (api_url → Cloud Run :8000)
           Cloud Run (Reflex backend, --backend-only)
              ↕ VPC private network
           Memorystore Redis (state manager)
```

The frontend is a static export (`reflex export --frontend-only`) served by Firebase Hosting. The backend runs on Cloud Run with `reflex run --env prod --backend-only`. Redis on Memorystore handles state persistence and is accessible only within the VPC.

## When to Use

- Deploying a Reflex app to GCP for the first time
- Setting up Cloud Run for a Reflex backend
- Configuring Firebase Hosting for a Reflex frontend
- Creating a VPC with Redis for Reflex state management
- Setting up Cloud Build CI/CD for automatic Reflex deployments
- Debugging a deployed Reflex app on GCP (ping/pong, WebSocket, Redis connectivity)
- Migrating a local Reflex app to production on GCP

## Prerequisites

Before starting, verify:

1. **Working Reflex app locally** — `reflex run` succeeds with no errors
2. **gcloud CLI installed and authenticated** — `gcloud auth list` shows active account
3. **GCP project selected** — `gcloud config get-value project` returns your project ID
4. **Firebase CLI installed** — `firebase --version` works
5. **Docker installed** — `docker --version` works
6. **APIs enabled** — Cloud Run, Cloud Build, Artifact Registry, Memorystore, Serverless VPC Access

```bash
# Enable all required APIs in one command
gcloud services enable \
  run.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  redis.googleapis.com \
  vpcaccess.googleapis.com \
  compute.googleapis.com
```

## Execution Steps for Agents

### Step 1: Gather Project Parameters

Collect these values before proceeding. All subsequent steps reference them.

| Parameter | Example | How to find |
|-----------|---------|-------------|
| `PROJECT_ID` | `perfwriter` | `gcloud config get-value project` |
| `REGION` | `us-central1` | Choose closest to users |
| `SERVICE_NAME` | `perfwriter-reflex` | Your Cloud Run service name |
| `APP_DIR` | `reflex_app/` | Directory containing your Reflex app |
| `REDIS_IP` | `10.60.48.3` | Created in Step 3 |
| `VPC_CONNECTOR` | `staging-vpc-connector` | Created in Step 2 |
| `FIREBASE_SITE` | `perfwriter-reflex` | Your Firebase Hosting site name |
| `CLOUD_RUN_URL` | `https://SERVICE_NAME-HASH.REGION.run.app` | Created in Step 4 |

### Step 2: Create VPC Network and Serverless VPC Connector

The VPC lets Cloud Run talk to Redis over private IPs. See [GCP_INFRASTRUCTURE.md](references/GCP_INFRASTRUCTURE.md) for detailed setup.

```bash
# Use existing default VPC or create a new one
gcloud compute networks describe default --format="value(name)" 2>/dev/null || \
  gcloud compute networks create default --subnet-mode=auto

# Create Serverless VPC Access connector
gcloud compute networks vpc-access connectors create ${VPC_CONNECTOR} \
  --region=${REGION} \
  --network=default \
  --range=10.8.0.0/28 \
  --min-instances=2 \
  --max-instances=10 \
  --machine-type=e2-micro
```

Verify connector is ready:

```bash
gcloud compute networks vpc-access connectors describe ${VPC_CONNECTOR} \
  --region=${REGION} --format="value(state)"
# Expected: READY
```

### Step 3: Create Redis Instance on Memorystore

Redis is the production state manager for Reflex. It must be on the same VPC network.

```bash
gcloud redis instances create reflex-redis \
  --size=1 \
  --region=${REGION} \
  --network=default \
  --redis-version=redis_7_2 \
  --tier=basic
```

Wait for creation (takes 3-5 minutes), then capture the IP:

```bash
REDIS_IP=$(gcloud redis instances describe reflex-redis \
  --region=${REGION} --format="value(host)")
echo "Redis endpoint: redis://${REDIS_IP}:6379"
```

**Important:** AUTH is disabled by default. The instance is secured by VPC network isolation — only resources within the VPC can reach it.

See [GCP_INFRASTRUCTURE.md](references/GCP_INFRASTRUCTURE.md) for sizing, AUTH, and TLS options.

### Step 4: Create Dockerfile for Backend

Create the Dockerfile in your app directory. See [TEMPLATES.md](references/TEMPLATES.md) for the full annotated template.

```dockerfile
FROM python:3.12

# Redis URL points to Memorystore private IP
ENV REDIS_URL=redis://${REDIS_IP}:6379 PYTHONUNBUFFERED=1

WORKDIR /app
COPY . .

RUN pip install -r requirements.txt

# Backend-only mode — frontend is served separately by Firebase
ENTRYPOINT ["reflex", "run", "--env", "prod", "--backend-only", "--loglevel", "debug"]
```

**Critical details:**
- Container port must be **8000** (Reflex backend default)
- `--backend-only` flag is essential — frontend is on Firebase
- `REDIS_URL` must use the Memorystore private IP from Step 3
- `--loglevel debug` is recommended for initial deployment, switch to `info` later

### Step 5: Create Cloud Build Configuration

Create `cloudbuild.yaml` (or `cloudbuild-staging.yaml` for staging) in your repo. See [CLOUD_BUILD.md](references/CLOUD_BUILD.md) for the full annotated template.

```yaml
steps:
  - name: 'gcr.io/cloud-builders/docker'
    id: Build
    args:
      - 'build'
      - '-t'
      - '$_AR_HOSTNAME/$PROJECT_ID/cloud-run-source-deploy/$REPO_NAME/$_SERVICE_NAME:$COMMIT_SHA'
      - '-f'
      - 'Dockerfile'
      - '.'
    dir: ${APP_DIR}

  - name: 'gcr.io/cloud-builders/docker'
    id: Push
    args:
      - push
      - >-
        $_AR_HOSTNAME/$PROJECT_ID/cloud-run-source-deploy/$REPO_NAME/$_SERVICE_NAME:$COMMIT_SHA
    dir: ${APP_DIR}

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
  _DEPLOY_REGION: ${REGION}
  _AR_HOSTNAME: ${REGION}-docker.pkg.dev
  _TRIGGER_ID: <your-trigger-id>
  _PLATFORM: managed
  _SERVICE_NAME: ${SERVICE_NAME}

tags:
  - gcp-cloud-build-deploy-cloud-run
  - gcp-cloud-build-deploy-cloud-run-managed
  - ${SERVICE_NAME}
```

### Step 6: Deploy Cloud Run Service

Deploy the initial Cloud Run service with VPC connector attached:

```bash
gcloud run deploy ${SERVICE_NAME} \
  --source=${APP_DIR} \
  --region=${REGION} \
  --port=8000 \
  --allow-unauthenticated \
  --vpc-connector=${VPC_CONNECTOR} \
  --vpc-egress=all-traffic \
  --session-affinity \
  --execution-environment=gen2 \
  --cpu-boost \
  --max-instances=100 \
  --min-instances=0 \
  --concurrency=1000 \
  --timeout=300
```

Capture the Cloud Run URL:

```bash
CLOUD_RUN_URL=$(gcloud run services describe ${SERVICE_NAME} \
  --region=${REGION} --format="value(status.url)")
echo "Backend URL: ${CLOUD_RUN_URL}"
```

**Verify backend is running:**

```bash
curl ${CLOUD_RUN_URL}/ping
# Expected: "pong"
```

See [GCP_INFRASTRUCTURE.md](references/GCP_INFRASTRUCTURE.md) for Cloud Run configuration details.

### Step 7: Set Up Cloud Build Trigger

Create a trigger that auto-deploys on push or PR:

```bash
# For push-to-branch trigger
gcloud builds triggers create github \
  --name="${SERVICE_NAME}-deploy" \
  --repo-name=<your-repo-name> \
  --repo-owner=<your-github-username> \
  --branch-pattern="^main$" \
  --build-config="${APP_DIR}/cloudbuild.yaml" \
  --region=global
```

Or configure via Cloud Console: Cloud Build → Triggers → Create trigger. Set:
- **Event:** Push to branch (or Pull request for staging)
- **Source:** Your GitHub repo
- **Configuration:** Cloud Build config file path (e.g., `app/cloudbuild/cloudbuild-staging.yaml`)
- **Build logs:** Send to GitHub (checked)

See [CLOUD_BUILD.md](references/CLOUD_BUILD.md) for trigger configuration, substitution variables, and staging vs production patterns.

### Step 8: Deploy Frontend to Firebase Hosting

The frontend is a static export pointed at the Cloud Run backend URL.

```bash
# Export frontend with API_URL pointing to Cloud Run
cd ${APP_DIR}
API_URL=${CLOUD_RUN_URL} reflex export --frontend-only --no-zip
```

Configure Firebase (`firebase.json` in repo root):

```json
{
  "hosting": [
    {
      "site": "${FIREBASE_SITE}",
      "public": "${APP_DIR}/.web/_static",
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

Deploy:

```bash
firebase deploy --only hosting:${FIREBASE_SITE}
```

See [FIREBASE_FRONTEND.md](references/FIREBASE_FRONTEND.md) for custom domains, CI/CD for frontend, and the Firebase-Cloud Run integration shortcut.

### Step 9: Verify Full Deployment

Run these checks in order:

| Check | Command | Expected |
|-------|---------|----------|
| Backend health | `curl ${CLOUD_RUN_URL}/ping` | `"pong"` |
| Redis connectivity | Check Cloud Run logs for Redis connection | No Redis errors |
| Frontend loads | Open `https://${FIREBASE_SITE}.web.app` in browser | App renders |
| WebSocket works | Open browser DevTools → Network → WS tab | `_event` connection established |
| State persists | Interact with app, refresh page | State maintained |

See [TROUBLESHOOTING.md](references/TROUBLESHOOTING.md) for common failures and fixes.

## Architecture Decision Summary

| Decision | Choice | Why |
|----------|--------|-----|
| Frontend hosting | Firebase Hosting | Free CDN, custom domains, fast static serving |
| Backend hosting | Cloud Run | Auto-scaling, pay-per-use, container-native |
| State manager | Memorystore Redis | Managed, VPC-native, no ops overhead |
| Networking | Serverless VPC connector | Lets Cloud Run reach Redis private IP |
| CI/CD | Cloud Build triggers | Native GCP integration, GitHub webhook support |
| Traffic routing | Route all to VPC | Simplifies networking, Redis always reachable |
| Execution env | 2nd generation | Better CPU/network, full Linux compatibility |
| Session affinity | Enabled | WebSocket connections stay on same instance |

## Quick Reference: Key Files

| File | Location | Purpose |
|------|----------|---------|
| `Dockerfile` | `${APP_DIR}/Dockerfile` | Backend container definition |
| `cloudbuild.yaml` | `${APP_DIR}/cloudbuild.yaml` | CI/CD build steps |
| `firebase.json` | Repo root | Frontend hosting config |
| `rxconfig.py` | `${APP_DIR}/rxconfig.py` | Reflex app configuration |
| `requirements.txt` | `${APP_DIR}/requirements.txt` | Python dependencies |

## Common Mistakes to Avoid

| Mistake | Problem | Fix |
|---------|---------|-----|
| Forgetting `--backend-only` in Dockerfile | Cloud Run tries to serve frontend too, wastes resources | Add `--backend-only` to ENTRYPOINT |
| Wrong container port | Cloud Run can't reach Reflex backend | Set container port to `8000` in Cloud Run config |
| Redis not on same VPC | Cloud Run can't connect to Redis | Create Redis on `default` network, same as VPC connector |
| Missing VPC connector on Cloud Run | Backend can't reach Redis private IP | Attach VPC connector with `--vpc-egress=all-traffic` |
| `API_URL` not set during frontend export | Frontend can't find backend, WebSocket fails | Set `API_URL=https://your-cloud-run-url` before `reflex export` |
| Using `reflex export` without `--frontend-only` | Exports both, but you only need frontend | Use `--frontend-only --no-zip` for Firebase |
| Cloud Build config path wrong | Trigger never fires or can't find config | Verify path matches repo structure exactly |
| No session affinity | WebSocket connections drop when instance shifts | Enable session affinity in Cloud Run networking |
| CPU only during requests | Backend WebSocket drops when CPU is throttled | Consider "CPU always allocated" for heavy WebSocket use |
| Firebase site name mismatch | Deploy goes to wrong site | Match `site` in `firebase.json` to your Firebase project site |

## Cost Estimates (as of 2024)

| Service | Tier | Approximate Cost |
|---------|------|-----------------|
| Cloud Run | Per-request pricing | Free tier: 2M requests/mo, then ~$0.40/M |
| Memorystore Redis | Basic 1GB | ~$35/mo |
| Firebase Hosting | Spark (free) | Free: 10GB storage, 360MB/day transfer |
| Cloud Build | Free tier | 120 min/day free |
| VPC Connector | e2-micro × 2 min | ~$7/mo |

**Total minimum:** ~$42/mo (Redis + VPC connector are the fixed costs)

## References

- [GCP_INFRASTRUCTURE.md](references/GCP_INFRASTRUCTURE.md) — VPC, Redis, Cloud Run configuration details
- [CLOUD_BUILD.md](references/CLOUD_BUILD.md) — CI/CD triggers, substitution variables, staging/prod patterns
- [FIREBASE_FRONTEND.md](references/FIREBASE_FRONTEND.md) — Firebase Hosting setup, custom domains, frontend deploy script
- [TEMPLATES.md](references/TEMPLATES.md) — Annotated Dockerfile, cloudbuild.yaml, firebase.json, deploy script templates
- [TROUBLESHOOTING.md](references/TROUBLESHOOTING.md) — Debugging connectivity, logs, common errors and fixes
