---
name: reflex-cloud-deploy
description: Use when deploying a Reflex app to Reflex Cloud using `reflex deploy`, configuring cloud.yml for regions, VM types, scaling, environment variables, custom domains, or deployment strategies. Also use when managing multi-environment deployments (dev/staging/prod) on Reflex Cloud.
license: Proprietary
compatibility: Requires Python 3.11+, reflex>=0.6.6, a working Reflex app (`reflex init` + `reflex run`), and a requirements.txt with all Python dependencies.
metadata:
  author: elviskahoro
  version: "1.0"
  tags: [reflex, deployment, cloud, hosting, reflex-cloud]
---

# Deploying to Reflex Cloud

Reflex Cloud is the managed hosting service for Reflex apps. Deploy with a single command — no infrastructure configuration required. Reflex handles the backend, frontend, networking, and scaling.

## When to Use

- Deploying a Reflex app with `reflex deploy`
- Configuring `cloud.yml` for regions, VM types, or scaling
- Setting up environment variables and secrets for a deployed app
- Configuring custom domains or subdomains
- Choosing a deployment strategy (immediate, rolling, blue-green, canary)
- Managing multi-environment deployments (dev, staging, prod)
- Troubleshooting Reflex Cloud deployments

## Prerequisites

Before deploying, verify:

1. **Reflex version** — `reflex>=0.6.6` installed
2. **App works locally** — `reflex init` completed and `reflex run` succeeds with no errors
3. **`requirements.txt` exists** — in the top-level app directory with all Python dependencies

```bash
# Generate requirements.txt from current environment
pip freeze > requirements.txt
```

## Execution Steps for Agents

### Step 1: Authenticate with Reflex Cloud

```bash
reflex login
```

This opens a browser for authentication via GitHub or Gmail. Once authenticated, the Reflex Cloud dashboard is accessible showing workspace projects.

### Step 2: Deploy the App

```bash
reflex deploy
```

The command is interactive and will prompt for:

1. **Requirements validation** — compares `requirements.txt` against the current Python environment. If differences exist, it asks whether to update the file.
2. **App existence check** — searches for a previously deployed app matching the current app name. If none found, prompts to confirm creating a new deployment.
3. **Optional description** — requests an optional app description.

**Important:** Deployment continues in the cloud after the command exits. The CLI indicates when it is safe to close without affecting the deployment process. The Cloud UI shows deployment status.

### Step 3: Configure with `cloud.yml`

Create a `cloud.yml` file in the project root to customize deployment behavior. **All fields are optional** — sensible defaults are applied.

```yaml
# cloud.yml — all fields optional
name: my-app                  # Deployment identifier (default: folder name)
description: My Reflex app    # Deployment description

# Regions and scaling
# Format: region_code: instance_count
regions:
  sjc: 1                      # San Jose, 1 instance (default)

# VM sizing
vmtype: c1m1                  # CPU/memory tier (default: c1m1)

# Custom subdomain
hostname: my-app              # Sets my-app.reflex.run (default: null, auto-generated)

# Environment variables
envfile: .env                 # Path to env file (default: .env)

# Project organization
project: <uuid>               # Project UUID (from dashboard URL)
projectname: my-project       # Or use project name instead of UUID

# System packages (installed via apt)
packages:
  - ffmpeg
  - libmagic1

# SQLite inclusion (NOT recommended for production)
include_db: false             # Include local SQLite DB (default: false)

# Deployment strategy
strategy: immediate           # How to roll out changes (default: immediate)
```

### Step 4: Configure Regions and Scaling

The `regions` field maps region codes to instance counts:

```yaml
# Single region, one instance
regions:
  sjc: 1

# Multi-region deployment
regions:
  sjc: 2
  iad: 1
```

Increase the instance count per region to handle more traffic.

### Step 5: Choose a Deployment Strategy

| Strategy | Behavior |
|----------|----------|
| `immediate` | Replace all instances at once (default) |
| `rolling` | Replace instances one at a time, zero downtime |
| `bluegreen` | Spin up new instances alongside old, switch traffic once healthy |
| `canary` | Boot a single new instance, verify health, then restart the rest |

```yaml
# For production apps, prefer rolling or canary
strategy: rolling
```

### Step 6: Set Up Environment Variables

Create a `.env` file in the project root (or the path specified by `envfile`):

```bash
# .env
DATABASE_URL=postgresql://user:pass@host:5432/db
API_KEY=sk-...
SECRET_KEY=your-secret-key
```

Reference a different env file in `cloud.yml`:

```yaml
envfile: .env.production
```

**Warning:** Do not commit `.env` files to version control. Add `.env*` to `.gitignore`.

### Step 7: Install System Packages

If your app depends on system-level libraries (e.g., for PDF generation, image processing), list them under `packages`:

```yaml
packages:
  - ffmpeg
  - poppler-utils
  - libmagic1
  - tesseract-ocr
```

These are installed via `apt` in the deployment environment.

### Step 8: Multi-Environment Deployments

Create separate config files for each environment:

```
cloud-dev.yml
cloud-staging.yml
cloud-prod.yml
```

Each file can specify different regions, VM types, strategies, and env files:

```yaml
# cloud-prod.yml
name: my-app-prod
regions:
  sjc: 2
  iad: 2
vmtype: c1m1
strategy: canary
envfile: .env.production
```

```yaml
# cloud-dev.yml
name: my-app-dev
regions:
  sjc: 1
vmtype: c1m1
strategy: immediate
envfile: .env.development
```

### Step 9: Verify Deployment

After deploying, check the Reflex Cloud dashboard for:

- Deployment status and health
- Logs and error output
- The live app URL

## Quick Reference

| Task | Command / Config |
|------|-----------------|
| Login | `reflex login` |
| Deploy | `reflex deploy` |
| Generate requirements | `pip freeze > requirements.txt` |
| Config file | `cloud.yml` in project root |
| Set region | `regions: { sjc: 1 }` in `cloud.yml` |
| Set VM type | `vmtype: c1m1` in `cloud.yml` |
| Custom domain | `hostname: my-app` in `cloud.yml` |
| Env vars | `envfile: .env` in `cloud.yml` |
| System packages | `packages: [ffmpeg]` in `cloud.yml` |
| Deploy strategy | `strategy: rolling` in `cloud.yml` |
| Multi-env | `cloud-dev.yml`, `cloud-staging.yml`, `cloud-prod.yml` |

## Common Mistakes to Avoid

| Mistake | Problem | Fix |
|---------|---------|-----|
| Missing `requirements.txt` | Deploy fails — dependencies unknown | Run `pip freeze > requirements.txt` before deploying |
| Stale `requirements.txt` | Deployed app missing new dependencies | Re-run `pip freeze > requirements.txt` after adding packages |
| Using `include_db: true` in production | SQLite data is lost on restart | Use a managed database service (PostgreSQL) instead |
| Committing `.env` to git | Secrets exposed in version control | Add `.env*` to `.gitignore`, use `envfile` in `cloud.yml` |
| Not testing locally first | Bugs discovered only after deploy | Always verify `reflex run` works before `reflex deploy` |
| Wrong `envfile` path | App starts without required env vars | Verify the path in `cloud.yml` matches the actual file location |
| Same `name` across environments | Deploys overwrite each other | Use distinct names: `my-app-dev`, `my-app-staging`, `my-app-prod` |
| `immediate` strategy in production | All instances replaced at once, brief downtime | Use `rolling` or `canary` for production deployments |

## References

- [Deploy Quick Start](https://reflex.dev/docs/hosting/deploy-quick-start/) — Getting started with `reflex deploy`
- [Config File Reference](https://reflex.dev/docs/hosting/config-file/) — Full `cloud.yml` configuration options
