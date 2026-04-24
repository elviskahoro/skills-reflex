# Cloud Build Reference

CI/CD configuration for automatically building and deploying the Reflex backend to Cloud Run.

## How Cloud Build Works with Cloud Run

1. A Git event (push, PR) triggers Cloud Build
2. Cloud Build runs your `cloudbuild.yaml` steps:
   - Build Docker image from your Dockerfile
   - Push image to Artifact Registry
   - Update Cloud Run service with new image
3. Cloud Run creates a new revision and routes traffic to it

## Annotated cloudbuild.yaml

```yaml
steps:
  # Step 1: Build the Docker image
  # Uses the official Docker builder to create the image
  # The image tag includes $COMMIT_SHA for traceability
  - name: 'gcr.io/cloud-builders/docker'
    id: Build
    args:
      - 'build'
      - '-t'
      # Image path: REGION-docker.pkg.dev/PROJECT/REPO/SERVICE:TAG
      - '$_AR_HOSTNAME/$PROJECT_ID/cloud-run-source-deploy/$REPO_NAME/$_SERVICE_NAME:$COMMIT_SHA'
      - '-f'
      - 'Dockerfile'
      - '.'
    # dir: sets the working directory for this step
    # Use this when your Dockerfile is in a subdirectory
    dir: reflex_app

  # Step 2: Push the built image to Artifact Registry
  - name: 'gcr.io/cloud-builders/docker'
    id: Push
    args:
      - push
      - >-
        $_AR_HOSTNAME/$PROJECT_ID/cloud-run-source-deploy/$REPO_NAME/$_SERVICE_NAME:$COMMIT_SHA
    dir: reflex_app

  # Step 3: Update the Cloud Run service with the new image
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

# Record the image in Artifact Registry for cleanup/auditing
images:
  - >-
    $_AR_HOSTNAME/$PROJECT_ID/cloud-run-source-deploy/$REPO_NAME/$_SERVICE_NAME:$COMMIT_SHA

options:
  # Allow $_VAR references in args
  automapSubstitutions: true
  # Only log to Cloud Logging (cheaper, no GCS bucket needed)
  logging: CLOUD_LOGGING_ONLY

# User-defined substitution variables
# These can be overridden per-trigger in the Cloud Console
substitutions:
  _DEPLOY_REGION: us-central1
  _AR_HOSTNAME: us-central1-docker.pkg.dev
  _TRIGGER_ID: <your-trigger-id-here>
  _PLATFORM: managed
  _SERVICE_NAME: perfwriter-reflex

tags:
  - gcp-cloud-build-deploy-cloud-run
  - gcp-cloud-build-deploy-cloud-run-managed
  - perfwriter-reflex
```

## Substitution Variables

### Built-in Variables (Available Automatically)

| Variable | Value |
|----------|-------|
| `$PROJECT_ID` | GCP project ID |
| `$BUILD_ID` | Unique build ID |
| `$COMMIT_SHA` | Full commit SHA |
| `$SHORT_SHA` | First 7 characters of commit SHA |
| `$REPO_NAME` | GitHub repository name |
| `$BRANCH_NAME` | Git branch name |
| `$TAG_NAME` | Git tag name (if triggered by tag) |
| `$REVISION_ID` | Same as `$COMMIT_SHA` |

### User-Defined Variables (Must Start with `_`)

| Variable | Purpose | Example |
|----------|---------|---------|
| `_SERVICE_NAME` | Cloud Run service name | `perfwriter-reflex` |
| `_DEPLOY_REGION` | Deployment region | `us-central1` |
| `_AR_HOSTNAME` | Artifact Registry hostname | `us-central1-docker.pkg.dev` |
| `_TRIGGER_ID` | The trigger's own ID | `0edafb45-...` |
| `_PLATFORM` | Cloud Run platform | `managed` |
| `_FLASK_ENV` | Custom env identifier | `staging`, `production` |

## Creating Triggers

### Via gcloud CLI

```bash
# Push-to-branch trigger (production)
gcloud builds triggers create github \
  --name="perfwriter-reflex-prod" \
  --repo-name=perfwriter \
  --repo-owner=adellhk \
  --branch-pattern="^main$" \
  --build-config="reflex_app/cloudbuild.yaml" \
  --region=global \
  --description="Build and deploy to Cloud Run on push to main"

# Pull request trigger (staging)
gcloud builds triggers create github \
  --name="perfwriter-reflex-staging" \
  --repo-name=perfwriter \
  --repo-owner=adellhk \
  --pull-request-pattern="^main$" \
  --build-config="reflex_app/cloudbuild-staging.yaml" \
  --region=global \
  --description="Build and deploy staging on PR to main"
```

### Via Cloud Console

1. Go to **Cloud Build → Triggers → Create trigger**
2. **Name:** `perfwriter-reflex-us-central1-...`
3. **Region:** Global
4. **Event:** Choose one:
   - **Push to a branch** — for production deployments on merge
   - **Pull request** — for staging/preview deployments
   - **Push new tag** — for release-based deployments
5. **Source:** Connect your GitHub repo (GitHub App)
6. **Configuration:**
   - Type: **Cloud Build configuration file**
   - Location: **Repository**
   - Path: `/reflex_app/cloudbuild.yaml` (must match your repo structure)
7. **Service account:** Default compute service account (or custom with limited perms)
8. Click **Save**

### Manually Running a Trigger

```bash
# List triggers
gcloud builds triggers list --region=global

# Run a specific trigger
gcloud builds triggers run TRIGGER_NAME \
  --region=global \
  --branch=main
```

Or in Cloud Console: Cloud Build → Triggers → click **Run** next to your trigger.

## Staging vs Production Patterns

### Option A: Separate Config Files

```
reflex_app/
  cloudbuild.yaml           # Production: deploys to perfwriter-reflex
  cloudbuild-staging.yaml   # Staging: deploys to perfwriter-reflex-staging
```

Staging config uses different `_SERVICE_NAME`:

```yaml
substitutions:
  _SERVICE_NAME: perfwriter-reflex-staging
  _DEPLOY_REGION: us-central1
```

### Option B: Single Config with Variable Override

Use one `cloudbuild.yaml` and override `_SERVICE_NAME` per trigger:

In Cloud Console → Trigger → Substitution variables:
- Production trigger: `_SERVICE_NAME = perfwriter-reflex`
- Staging trigger: `_SERVICE_NAME = perfwriter-reflex-staging`

### Option C: Branch-Based with Substitution

```yaml
substitutions:
  _SERVICE_NAME: perfwriter-reflex-${BRANCH_NAME}
```

This creates per-branch services (e.g., `perfwriter-reflex-feature-xyz`). Good for preview environments, but watch Cloud Run service limits.

## Build Logs and Debugging

### Viewing Build History

```bash
# List recent builds
gcloud builds list --limit=10 --region=global

# Get details of a specific build
gcloud builds describe BUILD_ID --region=global

# Stream logs of a running build
gcloud builds log BUILD_ID --region=global --stream
```

### Common Build Failures

| Error | Cause | Fix |
|-------|-------|-----|
| `Step "Build" failed` | Dockerfile error or dependency issue | Check Dockerfile, run `docker build` locally |
| `denied: Permission denied` | Service account lacks Artifact Registry access | Grant `roles/artifactregistry.writer` to build service account |
| `Step "Deploy" failed` | Cloud Run deployment error | Check service account has `roles/run.admin` |
| `could not find config file` | Wrong path in trigger config | Verify `cloudbuild.yaml` path matches repo structure |
| `quota exceeded` | Build machine quota | Check quotas or add `machineType` option |
| `TIMEOUT` | Build takes too long | Add `timeout: '1200s'` to options |

### Build Timeout

Default timeout is 10 minutes. For large Docker builds:

```yaml
options:
  timeout: '1200s'  # 20 minutes
  machineType: 'E2_HIGHCPU_8'  # Faster builds
```

## Service Account Permissions

The Cloud Build service account needs these roles:

| Role | Purpose |
|------|---------|
| `roles/run.admin` | Deploy to Cloud Run |
| `roles/iam.serviceAccountUser` | Act as the Cloud Run service account |
| `roles/artifactregistry.writer` | Push Docker images |
| `roles/logging.logWriter` | Write build logs |

```bash
PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")
SA="${PROJECT_NUMBER}-compute@developer.gserviceaccount.com"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:${SA}" \
  --role="roles/run.admin"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:${SA}" \
  --role="roles/artifactregistry.writer"
```

## Artifact Registry Cleanup

Old images accumulate. Set up a cleanup policy:

```bash
# List images
gcloud artifacts docker images list \
  us-central1-docker.pkg.dev/$PROJECT_ID/cloud-run-source-deploy

# Delete old images (keep last 10)
gcloud artifacts docker images list \
  us-central1-docker.pkg.dev/$PROJECT_ID/cloud-run-source-deploy/$REPO_NAME/$SERVICE_NAME \
  --sort-by=~CREATE_TIME --limit=100 --format="value(version)" | \
  tail -n +11 | \
  xargs -I {} gcloud artifacts docker images delete \
    us-central1-docker.pkg.dev/$PROJECT_ID/cloud-run-source-deploy/$REPO_NAME/$SERVICE_NAME@{} --quiet
```
