# Variables

Variables are set in dashboard / CLI / API / `railway.toml [env]` and injected as env vars at:
- build time
- service runtime
- `railway run <cmd>` (local)
- `railway shell`

Changes queue as staged changes — must be reviewed and deployed.

## Template syntax

```
${{NAMESPACE.VAR}}
```

- `NAMESPACE` = `shared` (shared vars) or a service name (e.g. `Postgres`, `backend-api`).
- `VAR` = the variable key.

Composable:

```
DOMAIN=${{shared.DOMAIN}}
GRAPHQL_PATH=/v1/gql
GRAPHQL_ENDPOINT=https://${{DOMAIN}}/${{GRAPHQL_PATH}}
```

## Variable kinds

### Service variables

Scoped to one service, one environment. Defined in the service's Variables tab. Can paste `.env`-style or JSON via the Raw Editor.

**Suggested variables** — Railway scans the connected repo for `.env`, `.env.example`, `.env.local`, `.env.production`, `.env.<anything>` and suggests keys to import.

### Shared variables

Project-level, reusable across services. Defined at Project Settings → Shared Variables → per-environment.

Use: Click "Share" from the Shared Variables page and pick services, OR add to a service's Variables tab via "Shared Variable". Adding creates a reference variable `${{ shared.KEY }}` on the service.

### Reference variables

Reference another service's variable, a shared variable, or the same service's variable using template syntax.

```
# Same service
${{ SOME_VAR }}

# Other service
${{ Postgres.DATABASE_URL }}
${{ backend-api.PORT }}

# Shared
${{ shared.API_TOKEN }}
```

The dashboard has autocomplete in both name and value fields for refs.

### Sealed variables

Security-first variables: values injected to build/deploy but **never** visible in UI or API.

**Caveats (one-way, asymmetric behavior):**
- Cannot be un-sealed.
- Not shown in `railway variables` or `railway run` locally.
- Not copied to PR environments.
- Not copied when duplicating a service or environment.
- Not shown in environment-sync diffs.
- Not synced to external integrations (e.g. Doppler).

Seal via the 3-dot menu on a variable. Updates go through the edit dialog only (not the Raw Editor).

### Multiline variables

Press `Cmd/Ctrl + Enter` in the value field, or paste a newline in the Raw Editor. Useful for PEM certs, multi-line tokens.

## Railway-provided variables

Injected automatically into every deploy. Reference them from your app with `process.env.NAME` / `os.environ["NAME"]`.

| Name | Description |
|---|---|
| `RAILWAY_PUBLIC_DOMAIN` | Public service/customer domain, e.g. `example.up.railway.app` |
| `RAILWAY_PRIVATE_DOMAIN` | Private DNS inside Railway's network (use for service-to-service) |
| `RAILWAY_TCP_PROXY_DOMAIN` | TCP proxy domain, e.g. `roundhouse.proxy.rlwy.net` |
| `RAILWAY_TCP_PROXY_PORT` | External TCP proxy port |
| `RAILWAY_TCP_APPLICATION_PORT` | Internal port for TCP proxy |
| `RAILWAY_PROJECT_NAME` / `RAILWAY_PROJECT_ID` | Project identity |
| `RAILWAY_ENVIRONMENT_NAME` / `RAILWAY_ENVIRONMENT_ID` | Environment identity |
| `RAILWAY_SERVICE_NAME` / `RAILWAY_SERVICE_ID` | Service identity |
| `RAILWAY_REPLICA_ID` | Replica ID for this deployment |
| `RAILWAY_REPLICA_REGION` | e.g. `us-west2` |
| `RAILWAY_DEPLOYMENT_ID` | This deployment's ID |
| `RAILWAY_SNAPSHOT_ID` | Snapshot ID for this deployment |
| `RAILWAY_VOLUME_NAME` / `RAILWAY_VOLUME_MOUNT_PATH` | Attached volume info |
| `PORT` | Port the app should listen on (auto-set per deploy) |

### Git variables (GitHub-triggered deploys only)

| Name | Example |
|---|---|
| `RAILWAY_GIT_COMMIT_SHA` | `d0beb8f5c55b36df7d674d55965a23b8d54ad69b` |
| `RAILWAY_GIT_AUTHOR` | `gschier` |
| `RAILWAY_GIT_BRANCH` | `main` |
| `RAILWAY_GIT_REPO_NAME` | `myproject` |
| `RAILWAY_GIT_REPO_OWNER` | `mycompany` |
| `RAILWAY_GIT_COMMIT_MESSAGE` | `Fixed a few bugs` |

### User-provided configuration variables

Set these yourself to control Railway behavior:

| Name | Effect | Default |
|---|---|---|
| `RAILWAY_DEPLOYMENT_OVERLAP_SECONDS` | Old/new deploy overlap duration | 0 |
| `RAILWAY_DOCKERFILE_PATH` | Path to Dockerfile | `Dockerfile` |
| `RAILWAY_HEALTHCHECK_TIMEOUT_SEC` | Healthcheck timeout | 300 |
| `RAILWAY_DEPLOYMENT_DRAINING_SECONDS` | SIGTERM→SIGKILL window | 0 |
| `RAILWAY_RUN_UID` | UID of main container process; `0` = root | — |
| `RAILWAY_SHM_SIZE_BYTES` | `/dev/shm` size in bytes | 67108864 (64 MB) |
| `RAILPACK_VERSION` | Pin Railpack build version | latest |

## Using variables in Dockerfiles

Build-time: declare `ARG` in the stage you need it:

```dockerfile
ARG RAILWAY_ENVIRONMENT
ARG RAILWAY_SERVICE_NAME
RUN echo "Building for $RAILWAY_ENVIRONMENT"
```

Runtime: use `ENV` or read from `process.env` / `os.environ` normally — Railway injects all service variables.

## Local development

Run local code with Railway-project env vars:

```bash
railway link
railway run npm run dev
# or
railway shell
```

Sealed vars are not available locally.

## Import from Heroku

Service Variables page → Command Palette → import from Heroku app. Requires OAuth to Heroku.

## Doppler integration

Sync secrets from Doppler via Doppler's Railway integration: https://docs.doppler.com/docs/railway. Syncs **create** Railway variables; sealed ones are not back-synced.
