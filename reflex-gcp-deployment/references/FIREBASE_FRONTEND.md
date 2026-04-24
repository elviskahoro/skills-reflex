# Firebase Frontend Reference

Deploying the Reflex static frontend to Firebase Hosting.

## Architecture

Reflex compiles the frontend to static HTML/JS/CSS files. These are exported with `reflex export --frontend-only` and hosted on Firebase Hosting, a free CDN-backed static hosting service.

The frontend connects to the Cloud Run backend via the `API_URL` environment variable, which is baked in at export time.

```
User → Firebase Hosting (CDN) → Static HTML/JS
                                    ↓ WebSocket + HTTP
                              Cloud Run Backend (:8000)
```

## Initial Firebase Setup

### Install Firebase CLI

```bash
npm install -g firebase-tools
```

### Login and Initialize

```bash
firebase login
firebase init hosting
```

During init:
- **Project:** Select your GCP project
- **Public directory:** `reflex_app/.web/_static` (where Reflex exports static files)
- **Single-page app:** No (Reflex handles routing client-side but exports individual pages)
- **GitHub CI/CD:** Optional (you can set this up later)

### Create a Firebase Hosting Site

If you need a named site (for multi-site projects):

```bash
firebase hosting:sites:create perfwriter-reflex
```

## firebase.json Configuration

```json
{
  "hosting": [
    {
      "site": "perfwriter-reflex",
      "public": "reflex_app/.web/_static",
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

**Field guide:**

| Field | Value | Notes |
|-------|-------|-------|
| `site` | Your Firebase site name | Must match exactly |
| `public` | Path to exported static files | Relative to `firebase.json` location |
| `ignore` | Files to exclude | Standard ignores |
| `rewrites` | URL rewriting rules | 404 redirect for missing pages |

### Multi-Site Configuration

If you have multiple Firebase Hosting sites (e.g., docs + app):

```json
{
  "hosting": [
    {
      "site": "perfwriter-reflex",
      "public": "reflex_app/.web/_static",
      "ignore": ["firebase.json", "**/.*", "**/node_modules/**"],
      "rewrites": [{"source": "404", "destination": "/404.html", "type": 301}]
    },
    {
      "site": "perfwriter-docs",
      "public": "docs/build",
      "ignore": ["firebase.json", "**/.*"]
    }
  ]
}
```

Deploy specific site: `firebase deploy --only hosting:perfwriter-reflex`

## Frontend Export and Deploy

### Manual Deploy Script

```bash
#!/bin/bash
set -euo pipefail

# Configuration — edit these
CLOUD_RUN_URL="https://perfwriter-reflex-881125452023.us-central1.run.app"
FIREBASE_SITE="perfwriter-reflex"
APP_DIR="reflex_app"

echo "=== Exporting Reflex frontend ==="
cd "${APP_DIR}"

# API_URL tells the frontend where to find the backend
# This is baked into the static build at export time
API_URL="${CLOUD_RUN_URL}" reflex export --frontend-only --no-zip

if [ $? -eq 0 ]; then
    echo "=== Deploying to Firebase Hosting ==="
    cd ..
    firebase deploy --only "hosting:${FIREBASE_SITE}"
    echo "=== Deploy complete ==="
else
    echo "ERROR: Reflex export failed. Firebase deploy will not be executed."
    exit 1
fi
```

Save as `deploy-frontend.sh` and `chmod +x deploy-frontend.sh`.

### What `reflex export --frontend-only --no-zip` Does

1. Compiles all Reflex components to React/Next.js
2. Builds optimized static HTML, JS, CSS bundles
3. Writes output to `.web/_static/` directory
4. `--no-zip` skips creating a zip file (Firebase needs the directory)
5. `API_URL` env var is embedded into the JavaScript bundles

**Critical:** If you change the Cloud Run URL, you must re-export and redeploy the frontend.

## Custom Domains

### Firebase Default Domains

Every Firebase Hosting site gets two domains for free:
- `${SITE_NAME}.web.app`
- `${SITE_NAME}.firebaseapp.com`

### Adding a Custom Domain

```bash
firebase hosting:sites:list  # Verify your site exists
```

Then in Firebase Console:
1. Go to **Hosting** → your site
2. Click **Add custom domain**
3. Enter your domain (e.g., `reflex.app.perfwriter.com`)
4. Add the DNS records Firebase provides (A records or CNAME)
5. Wait for SSL provisioning (can take up to 24 hours)

Or via CLI:

```bash
firebase hosting:channel:deploy live --site perfwriter-reflex
```

### DNS Configuration

For apex domains (e.g., `perfwriter.com`):
- Add A records pointing to Firebase's IPs (provided during setup)

For subdomains (e.g., `app.perfwriter.com`):
- Add CNAME record pointing to `perfwriter-reflex.web.app`

## Firebase + Cloud Run Integration (Shortcut)

Cloud Run has a built-in Firebase Hosting integration that can automate the frontend-backend connection:

1. Go to Cloud Run → your service → **Integrations** tab
2. Click **Add Integration**
3. Select **Firebase Hosting**
4. This creates a Firebase site linked to your Cloud Run service

**When to use this shortcut:**
- New project with no existing Firebase setup
- You want Firebase to automatically proxy API requests to Cloud Run

**When NOT to use:**
- You already have Firebase configured
- You need custom `firebase.json` rewrites
- You want separate control over frontend deploys

## Preview Channels

Firebase Hosting supports preview channels for testing before going live:

```bash
# Create a preview channel
firebase hosting:channel:deploy preview-branch-name --site perfwriter-reflex

# This gives you a URL like:
# https://perfwriter-reflex--preview-branch-name-abc123.web.app
```

Preview channels auto-expire after 7 days by default. Useful for PR previews.

## Automating Frontend Deploys

### GitHub Actions

```yaml
# .github/workflows/deploy-frontend.yml
name: Deploy Frontend to Firebase

on:
  push:
    branches: [main]
    paths:
      - 'reflex_app/**'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: |
          cd reflex_app
          pip install -r requirements.txt
          npm install -g firebase-tools

      - name: Export frontend
        run: |
          cd reflex_app
          API_URL=${{ secrets.CLOUD_RUN_URL }} reflex export --frontend-only --no-zip

      - name: Deploy to Firebase
        run: firebase deploy --only hosting:perfwriter-reflex
        env:
          FIREBASE_TOKEN: ${{ secrets.FIREBASE_TOKEN }}
```

Generate `FIREBASE_TOKEN` with: `firebase login:ci`

### Script for CI/CD Pipelines

```bash
#!/bin/bash
# deploy-frontend-ci.sh — for use in any CI system
set -euo pipefail

CLOUD_RUN_URL="${CLOUD_RUN_URL:?Set CLOUD_RUN_URL environment variable}"
FIREBASE_SITE="${FIREBASE_SITE:-perfwriter-reflex}"
APP_DIR="${APP_DIR:-reflex_app}"

cd "${APP_DIR}"
API_URL="${CLOUD_RUN_URL}" reflex export --frontend-only --no-zip

cd ..
firebase deploy --only "hosting:${FIREBASE_SITE}" --non-interactive
```

## Rollback

Firebase keeps previous releases. To rollback:

1. Go to Firebase Console → Hosting
2. Find the previous release in the release history
3. Click the ⋮ menu → **Rollback to this release**

Or via CLI:

```bash
# List recent deploys
firebase hosting:releases:list --site perfwriter-reflex --limit=5

# Rollback to a specific version
firebase hosting:clone perfwriter-reflex:VERSION perfwriter-reflex:live
```

## Cache and Performance

Firebase Hosting serves from a global CDN. Default cache headers:

| Content | Cache | TTL |
|---------|-------|-----|
| HTML | CDN + browser | 10 minutes |
| JS/CSS (hashed) | CDN + browser | 1 year |
| Images | CDN + browser | 1 year |

Reflex's Next.js build uses content hashing for JS/CSS bundles, so long cache TTLs are safe — new deploys generate new filenames.

### Custom Cache Headers

In `firebase.json`:

```json
{
  "hosting": [{
    "site": "perfwriter-reflex",
    "public": "reflex_app/.web/_static",
    "headers": [
      {
        "source": "**/*.@(js|css)",
        "headers": [{"key": "Cache-Control", "value": "public, max-age=31536000, immutable"}]
      },
      {
        "source": "**/*.html",
        "headers": [{"key": "Cache-Control", "value": "public, max-age=600"}]
      }
    ]
  }]
}
```
