# Troubleshooting — Reflex on Railway

Real-world fixes collected from deploying Reflex apps to Railway. Reflex is a Python framework that compiles to a Next.js frontend + FastAPI-style backend; most of the pain on Railway comes from the build phase and the first-boot frontend compilation.

## The OOM problem (and the Alpine Dockerfile that fixes it)

### Symptom

Railway deploy fails with:
- Build killed ~1-3 min into `pip install` or `uv sync`.
- `exit code 137` (Linux OOMKilled).
- "Deployment failed during build" with no clear Python error.
- Succeeds locally with `docker build` but not on Railway.

Root cause: Railway's free plan builds with ~512 MB RAM. `python:3.11-slim` (Debian-based) plus Reflex's dependencies (numpy, pydantic, fastapi, uvicorn, and the JS toolchain Reflex pulls in) easily exceeds that during install. The Debian build toolchain itself is heavy.

### The pattern that works

Use **Alpine Linux**, install build deps as a **virtual package** that gets removed after install, and skip dev dependencies. This file is battle-tested in production:

```dockerfile
FROM python:3.11-alpine

WORKDIR /app

# Install build deps as a virtual package — removed after uv sync.
RUN apk add --no-cache --virtual .build-deps gcc musl-dev linux-headers

# Copy only dependency manifests first (Docker layer caching).
COPY pyproject.toml ./

# Install uv + project deps, then strip build deps in one layer.
RUN pip install --no-cache-dir uv && \
    uv sync --frozen --no-dev && \
    apk del .build-deps

# Copy application code after deps — unchanged code reuses the dep layer.
COPY . .

EXPOSE 3000

ENV PORT=3000 \
    REFLEX_ENV=prod

# Reflex compiles the frontend on first startup (not at build time) — expect
# first-boot memory spike; sizing is determined by runtime plan, not build plan.
CMD [".venv/bin/reflex", "run", "--env", "prod"]
```

Paired `railway.toml`:
```toml
[build]
builder = "DOCKERFILE"
watchPatterns = ["Dockerfile", "pyproject.toml", "web/**"]

[deploy]
startCommand = ".venv/bin/reflex run --env prod"
restartPolicyType = "ON_FAILURE"
restartPolicyMaxRetries = 5
drainingSeconds = 30
healthcheckPath = "/"
healthcheckTimeout = 30

[env]
REFLEX_ENV = "prod"
PYTHONUNBUFFERED = "1"
```

### Why each piece matters

| Choice | Why |
|---|---|
| `python:3.11-alpine` | ~45 MB base vs ~130 MB slim; fewer background processes during build. |
| `--virtual .build-deps` | Groups gcc/musl-dev/linux-headers under a name you can remove in one line. |
| `apk del .build-deps` in the same `RUN` | Same layer means the intermediate is never committed. If you put `apk del` in a separate `RUN`, the previous layer still contains gcc and your image stays bloated. |
| `pip install --no-cache-dir uv` | Skip pip's wheel cache — you only install uv once. |
| `uv sync --frozen --no-dev` | Lockfile install, no dev deps (pytest, mypy, etc.) — often shaves 50-100 MB. |
| `COPY pyproject.toml ./` before `COPY . .` | Lets Docker reuse the dep layer when only app code changes. |

### If Alpine isn't an option (C-extension incompatibility)

Some wheels don't ship for `musllinux` (numpy/scipy variants, old C-extensions). If you hit `No module named '_psycopg'` or similar, either:
1. Use a slim image + upgrade Railway plan (1 GB+ RAM). OR
2. Switch to the `python:3.11` full image and use cache mounts to reduce peak working-set memory:
   ```dockerfile
   RUN --mount=type=cache,id=s/<svc-id>-/root/.cache/uv,target=/root/.cache/uv \
       uv sync --frozen --no-dev
   ```

## Healthcheck failures on Reflex

### Symptom

Deploy builds, container starts, but healthcheck times out. Logs show Reflex launching normally, but Railway marks the deploy failed after 5 min.

### Causes & fixes

**1. First-boot frontend compilation.** Reflex compiles the JS bundle on first startup in prod mode. This can take 1-3 min on small plans. If `healthcheckTimeout` is shorter than compile time, the deploy fails.

Fix: set a long timeout or skip healthcheck on first deploy:
```toml
[deploy]
healthcheckTimeout = 180     # or unset entirely for first deploy
```

**2. `/` serves the SPA shell during compilation.** Until Reflex finishes compiling, `/` may return partial HTML or 503.

Fix: point healthcheck at a stable endpoint, or unset it. Reflex doesn't guarantee a `/ping` route by default — check your app's routes first.

**3. App not listening on `$PORT`.** Reflex reads `PORT` from env, but if you hardcode `--port 3000` or set a different `PORT`, Railway's healthcheck hits the wrong port.

Fix: let Railway set `PORT`; don't override in the start command.

**4. Host-filter middleware rejecting `healthcheck.railway.app`.** Rare in Reflex, but if you've wrapped with a middleware that filters by `Host:`, add `healthcheck.railway.app` to the allowlist.

## "No start command could be found"

### Symptom

Deploy builds, then fails with `No start command could be found` before the container starts.

### Cause

Railway is using Railpack (no Dockerfile detected) and can't auto-detect how to start your app.

### Fixes

**1. Add a Dockerfile** at repo root. Railway uses it automatically.

**2. Set start command in config:**
```toml
[deploy]
startCommand = ".venv/bin/reflex run --env prod"
```

**3. Set in dashboard:** Service Settings → Start Command.

Common Reflex start commands:
- Docker: `.venv/bin/reflex run --env prod`
- Railpack (uv-managed): `uv run reflex run --env prod`

## Slow deployments

### Symptom

Deploys take 5-10+ minutes. Frustrating for iteration.

### Fixes (stack them for cumulative effect)

**1. Cache mount uv/pip:**
```dockerfile
RUN --mount=type=cache,id=s/<service-id>-/root/.cache/uv,target=/root/.cache/uv \
    uv sync --frozen --no-dev
```

Get your `<service-id>` from the dashboard URL: `railway.com/project/<pid>/service/<sid>`.

**2. Order Dockerfile layers dep-before-code:**
```dockerfile
COPY pyproject.toml uv.lock ./          # dep layer
RUN uv sync --frozen --no-dev           # cached unless deps change
COPY . .                                # cheap code-only rebuilds
```

**3. Tighten watch patterns** so docs/config changes don't trigger rebuilds:
```toml
[build]
watchPatterns = ["Dockerfile", "pyproject.toml", "uv.lock", "web/**", "src/**"]
```

**4. Skip dev deps:** `--no-dev` (uv) or `--omit=dev` (npm).

**5. Multi-stage for large build artifacts:**
```dockerfile
FROM python:3.11-alpine AS builder
# ... heavy build ...

FROM python:3.11-alpine AS runtime
COPY --from=builder /app/.venv /app/.venv
COPY . .
CMD [".venv/bin/reflex", "run", "--env", "prod"]
```

**6. Reuse images across commits** — if `watchPatterns` excludes the changed files, Railway skips build and redeploys the existing image.

## SIGTERM not handled (abrupt crashes on redeploy)

### Symptom

On redeploy, existing connections die abruptly; in-flight requests are cut; Reflex logs stop mid-line.

### Cause

Railway sends SIGTERM, waits `drainingSeconds`, then SIGKILL. If Python doesn't receive the signal (e.g. wrapped in a shell that swallows it), the server never cleans up.

### Fixes

**1. Use exec form in `CMD`** so Python is PID 1:
```dockerfile
CMD [".venv/bin/reflex", "run", "--env", "prod"]     # exec form — good
# NOT:
CMD ".venv/bin/reflex run --env prod"                # shell form — SIGTERM may not reach Reflex
```

**2. If using `sh -c`, prefix with `exec`:**
```toml
[deploy]
startCommand = "/bin/sh -c \"exec .venv/bin/reflex run --env prod\""
```

**3. Increase `drainingSeconds`** to give Reflex time to shut down gracefully:
```toml
[deploy]
drainingSeconds = 30
```

## Postgres wiring for Reflex

### Pattern

1. Add Postgres: project canvas → New → Database → PostgreSQL.
2. In the Reflex service's Variables tab, add:
   ```
   DATABASE_URL=${{Postgres.DATABASE_URL}}
   ```
3. Use the private URL for app-to-DB traffic (it's what `${{Postgres.DATABASE_URL}}` resolves to by default inside Railway's network — check the variable reference to confirm).

### Reflex-specific

In `rxconfig.py`:
```python
import os
import reflex as rx

config = rx.Config(
    app_name="web",
    db_url=os.environ.get("DATABASE_URL"),
)
```

Run migrations via pre-deploy command:
```toml
[deploy]
preDeployCommand = [".venv/bin/reflex db migrate"]
```

## Check logs when stuck

```bash
railway logs                  # stream current service
railway logs --deployment <id>   # specific deployment
railway status                # show linked project/env/service
```

Or dashboard → service → Logs tab.

## Nothing works, and the app used to deploy

Often a dependency resolution or upstream image change. Isolate:

1. Compare `uv.lock` / `package-lock.json` to the last-successful commit.
2. Check if Reflex released a breaking version — pin with `reflex==<last-known-good>` in `pyproject.toml`.
3. Redeploy a prior successful deployment from the Deployments tab (rollback) to confirm the regression is in recent code, not Railway itself.
4. Check Railway status: https://status.railway.app.

## Plan-level constraints to rule out first

| Constraint | Free | Paid |
|---|---|---|
| Build RAM | ~512 MB | 1+ GB (plan-dependent) |
| Runtime RAM | ~512 MB | plan-dependent |
| Restart policy `ALWAYS` | ❌ | ✔️ |
| Private Docker registries | ❌ | ✔️ (Pro+) |
| Multi-replica / multi-region | ❌ | ✔️ |
| Ephemeral storage | 1 GB | 100 GB |

A surprising number of "why won't this deploy" issues are free-plan caps.
