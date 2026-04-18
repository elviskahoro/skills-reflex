# Services and environments

## Services

A **service** is a deployed container. Each service stores its source config (GitHub repo / Docker image / local), variables, build + start commands, and a log of deployments.

### Types

- **Persistent** — always running (web app, API, queue, DB).
- **Scheduled (cron)** — runs to completion on a schedule. See [cron-and-functions.md](cron-and-functions.md).

### Sources

| Source | How |
|---|---|
| GitHub repo | "Connect Repo" in service settings; pushes to the linked branch auto-deploy. Requires the Railway GitHub App: https://github.com/apps/railway-app/installations/new |
| Public Docker image | Paste path: `bitnami/redis`, `ghcr.io/owner/img:tag`, `quay.io/owner/repo:tag`, `registry.gitlab.com/owner/img`, `mcr.microsoft.com/dotnet/aspire-dashboard`. |
| Private Docker image | Pro plan only. Provide credentials at service creation. GHCR needs a classic PAT. |
| Local directory | Create an empty service, then `railway link` + `railway up` from the directory. |

### Docker image auto-updates

Railway monitors pulled images. When a new version is available, a button appears in settings. Behavior depends on the tag:
- Versioned tag (e.g. `nginx:1.25.3`): updating stages the new version.
- Floating tag (`nginx:latest`): updating redeploys to pull the newest digest.

Auto-update schedule + maintenance window configurable. Services with attached volumes incur brief downtime during the redeploy.

### Dockerfile auto-detection

If `Dockerfile` (capital D) exists in the source root, Railway **always** uses it — no opt-in needed. This overrides `builder = "RAILPACK"` in config. To choose a different Dockerfile, set `RAILWAY_DOCKERFILE_PATH` or `build.dockerfilePath` in `railway.toml`.

Railway prints this line in the build log when it's using your Dockerfile:
```
==========================
Using detected Dockerfile!
==========================
```

### Ephemeral storage

- Free plan: 1 GB
- Paid: 100 GB

Deployments exceeding the limit can be forcefully stopped and redeployed. Use a **Volume** for persistence or size beyond that.

### Monorepo

See https://docs.railway.com/deployments/monorepo. Two patterns:
- **Root Directory** — point the service at a subdirectory (`apps/backend`). Railway treats that as the source root.
- **Watch Paths** — deploy only when changed files match a glob. Works with Focused PR Environments to skip unaffected services on PRs.

### Deploy approval

If someone pushes to a linked branch but doesn't have a Railway account linked to their GitHub, Railway creates a Deployment Approval. You Approve or Reject from the service canvas.

### Constraints

- Service name: max 32 chars.
- Staged changes: adding/editing services queues changes that must be reviewed and deployed.

## Environments

An **environment** is an isolated copy of all services in a project. `production` exists by default.

### Types

- **Persistent** — `production`, `staging`, etc. Stick around. Typically auto-deploy from different git branches.
- **PR (ephemeral)** — auto-created when a PR is opened, deleted on merge/close.

### Create an environment

1. Top-nav env dropdown → "+ New Environment", or Settings → Environments.
2. Pick:
   - **Duplicate** — copies services, variables, config. Staged as changes until reviewed.
   - **Empty** — no services.

### Sync between environments

Import services from one env into another:
1. Switch to the **receiving** env.
2. Click "Sync" at top of canvas.
3. Pick the source env.
4. Each service is tagged `New`, `Edited`, or `Removed`. Review staged changes, then Deploy.

### PR environments

Enable in Project Settings → Environments. When enabled, every PR spins up a temporary env. Deleted on merge/close.

**Gotchas:**
- Railway won't deploy PR branches from users outside your workspace unless their GitHub account is linked to a project membership.
- For auto-provisioned domains in PR envs, the **base env** services must use Railway-provided domains.

**Focused PR environments** (beta): only deploy services affected by changed files (via watch paths or root directory) plus their variable-referenced dependencies. Skipped services can be deployed manually per-PR. Enable: Project Settings → Environments → Enable Focused PR Environments.

**Bot PR environments**: toggle on Environments tab to also spin up PR envs for bots (Dependabot, Renovate, Copilot, Claude Code, Devin, Jules, etc.).

### Environment RBAC (Enterprise only)

Restrict non-admins from seeing sensitive envs:

| Role | Can access | Can toggle |
|---|---|---|
| Admin | ✔️ | ✔️ |
| Member | ❌ | ❌ |
| Deployer | ❌ | ❌ |

Deployer role can still trigger deploys via git push even without env access.

### Forked environments (deprecated Jan 2024)

Use **Sync Environments** instead. Any pre-existing forked envs remain but merges require the sync flow.

### The `pr` magic environment

When writing config-as-code, `environments.pr` is a reserved name — settings under it apply to every ephemeral PR deploy. Priority: named-ephemeral-env → `pr` → base env → base config → dashboard. Useful for PR-specific start commands, feature flags, or trimmed pre-deploy commands.
