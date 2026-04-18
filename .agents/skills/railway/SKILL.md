---
name: railway
description: Reference for deploying, configuring, and troubleshooting applications on Railway (railway.com). Covers config-as-code (`railway.json` / `railway.toml` schema), services, environments (persistent + PR), variables + reference variables, cron jobs, Functions, builds (Dockerfile + Railpack), deployments (healthchecks, restart policy, pre-deploy, regions, scaling), and a Reflex-on-Railway troubleshooting guide including an Alpine Dockerfile pattern that survives free-tier memory limits. Invoke explicitly (e.g. via `/railway`) when working on Railway config — the skill does not auto-trigger on incidental Railway mentions.
---

# Railway

Railway is a Heroku-style PaaS. A **project** groups one or more **services**; each service is deployed into one or more **environments**. Every service has build + deploy settings that can live in a `railway.json` or `railway.toml` file alongside the code — this is the recommended way to configure Railway for an AI agent, because it's reviewable, diffable, and reproducible across PR environments.

## Mental model

```
Project ──▶ Environment  (production | staging | pr-123 | ...)
              └─▶ Service  (container from GitHub repo | Docker image | local dir)
                    └─▶ Deployment  (one attempt: build → pre-deploy → start → healthcheck → active)
```

- **Project** — top-level container. Holds Shared Variables usable across services.
- **Environment** — isolated copy of all services. `production` exists by default. Persistent envs (e.g. `staging`) stick around; PR envs are ephemeral per pull request.
- **Service** — deploy target. Sources: GitHub repo, Docker Hub / ghcr.io / quay.io / gcr image, or local via `railway up`. Service names: max 32 chars.
- **Deployment** — one build+run attempt. Lifecycle: `INITIALIZING → BUILDING → DEPLOYING → SUCCESS | FAILED | CRASHED | REMOVED`.

Any service change creates a **staged change** that must be reviewed and deployed.

## Config as code (quickstart)

Drop a `railway.toml` at the repo root. Config-in-code **always overrides** dashboard settings — but **only for that deployment**; the dashboard itself isn't updated.

Minimal `railway.toml`:

```toml
[build]
builder = "DOCKERFILE"                 # DOCKERFILE | RAILPACK (default). Railway uses Dockerfile automatically if one exists.
watchPatterns = ["src/**", "Dockerfile"]

[deploy]
startCommand = "python main.py"        # for Dockerfile/image: overrides ENTRYPOINT in exec form
healthcheckPath = "/health"            # must return 200 for deploy to go live
healthcheckTimeout = 300               # seconds; default 300
restartPolicyType = "ON_FAILURE"       # ON_FAILURE (default) | ALWAYS | NEVER
restartPolicyMaxRetries = 10           # capped to 10 on free plan; ALWAYS unavailable on free
drainingSeconds = 30                   # SIGTERM → SIGKILL buffer during rollout
preDeployCommand = ["python manage.py migrate"]   # runs once before start, in same container

[env]
NODE_ENV = "production"
```

The equivalent JSON form:

```json
{
  "$schema": "https://railway.com/railway.schema.json",
  "build": { "builder": "DOCKERFILE" },
  "deploy": {
    "startCommand": "python main.py",
    "healthcheckPath": "/health",
    "restartPolicyType": "ON_FAILURE"
  }
}
```

Per-environment overrides (the env name must match a real env; `pr` is a magic name for ephemeral PR envs):

```toml
[environments.staging.deploy]
startCommand = "python main.py --env staging"

[environments.pr.deploy]
startCommand = "python main.py --env pr"
```

Resolution priority (highest first): environment-specific config → base config → dashboard service settings. For PR deploys: named-env config → `pr` config → base-env config → base config → dashboard.

**Full schema, every field, every value:** [references/config-schema.md](references/config-schema.md).

## Deploying a Reflex app (the primary workflow this skill is tuned for)

Reflex has Railway-specific sharp edges. Internalize these before editing anything:

1. **Reflex listens on `$PORT`.** Railway injects `PORT`; `reflex run --env prod` reads it. Don't hardcode 3000 in the start command.
2. **Reflex builds the frontend on first startup** in prod mode (not during `docker build`). Peak memory is during first boot, not image build. Plan memory accordingly — this is the #1 cause of Railway OOMs for Reflex apps.
3. **Healthchecks against `/` are fragile** on Reflex because `/` serves a SPA. Options: leave `healthcheckPath` unset for the first deploy, use a short `healthcheckTimeout` only after verifying the app responds, or add an explicit `/ping` endpoint. Reflex does not guarantee `/ping` by default, so verify before trusting it.
4. **Use Alpine-based Python images.** Slim images exhaust the free-tier 512MB build budget. See [references/troubleshooting-reflex.md](references/troubleshooting-reflex.md) for the exact pattern (build deps as virtual, `uv sync --frozen --no-dev`, `apk del .build-deps`).
5. **Postgres wiring.** Add a Postgres service via "New → Database → PostgreSQL", then reference it from the Reflex service as `DATABASE_URL=${{Postgres.DATABASE_URL}}`. Always use the **private** URL for service-to-service — it's free, faster, and doesn't leave Railway's network. The public URL incurs egress.
6. **Single-service vs split.** Reflex apps run as one service by default (Python process serves both backend on :8000 and frontend on :3000 via Reflex's own bundling). Only split into two services if you have a reason — the Caddy reverse-proxy pattern is for split deployments.

Example `railway.toml` for a typical single-service Reflex app:

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

## When to read which reference

Load only the file(s) relevant to the task:

- **Editing `railway.json` / `railway.toml`** → [references/config-schema.md](references/config-schema.md) — every field, value, nesting rule.
- **Services, environments, PR envs, Focused PR envs, monorepos** → [references/services-and-environments.md](references/services-and-environments.md).
- **Variables, reference syntax, shared vars, sealed vars, Railway-provided vars** → [references/variables.md](references/variables.md).
- **Build issues: Dockerfile detection, Railpack, cache mounts, watch patterns, build args** → [references/builds.md](references/builds.md).
- **Deploy behavior: healthchecks, restart policy, pre-deploy, regions, scaling, draining, multi-region** → [references/deployments.md](references/deployments.md).
- **Cron jobs (schedule format, exit requirements) and TypeScript Bun Functions** → [references/cron-and-functions.md](references/cron-and-functions.md).
- **OOM, slow builds, "No start command could be found", SIGTERM handling, and the Reflex Dockerfile war story** → [references/troubleshooting-reflex.md](references/troubleshooting-reflex.md).

## CLI quick reference

```
railway login                         # auth
railway link                          # link cwd to a project + service
railway up                            # deploy current dir to the linked service
railway run <cmd>                     # run cmd locally with project env vars injected
railway shell                         # interactive shell with env loaded
railway logs                          # stream service logs
railway variables                     # list env vars (sealed values are hidden)
railway status                        # show current project / env / service
railway redeploy                      # redeploy the current active deployment
railway down                          # remove the most recent deployment
railway deployment list               # list recent deployments for this service
```

Run `railway <cmd> --help` for full options.

## Non-obvious behaviors to remember

- **Dockerfile auto-wins.** If a file literally named `Dockerfile` (capital D) exists, Railway uses it regardless of `builder` setting. Use `RAILWAY_DOCKERFILE_PATH=Dockerfile.backend` to pick a different one.
- **Healthcheck hostname is `healthcheck.railway.app`** — if your app filters hosts, allow this one or you'll see "service unavailable".
- **Healthchecks are one-shot** — only run at deploy, not continuous. For continuous, add an external monitor.
- **Start-command `$VAR` expansion** requires a shell wrap for Dockerfile/image services: `"/bin/sh -c \"exec python main.py --port $PORT\""`. Railpack-built services already run via shell.
- **Volumes ⇒ deploy downtime.** Railway refuses to run two deployments with the same volume mounted, so zero-downtime isn't possible there.
- **Cron semantics.** Min frequency 5 min. Schedules are UTC. If the previous run is still active when the next fires, Railway skips it — the process MUST exit cleanly.
- **Sealed variables are one-way.** Cannot be un-sealed, not in `railway variables`, not copied to PR envs, not copied when duplicating services/envs.
- **Free plan caps.** ~512MB RAM, `restartPolicyType = ALWAYS` unavailable, `ON_FAILURE` capped at 10 retries, private Docker registries disabled.
- **Ephemeral storage** limit is 1GB free / 100GB paid. Exceeding it forces a restart. Use a volume for anything that must persist or exceed that.

## Operating style for this skill

When asked to edit Railway config:
1. Read the current `railway.toml` / `railway.json` and `Dockerfile` first.
2. Before changing `healthcheckPath`, verify the endpoint returns 200 in the actual running app — a wrong path wedges deploys for 5 minutes per attempt.
3. Before suggesting `restartPolicyType = ALWAYS`, check the plan — it silently no-ops on free.
4. When adding env-specific overrides, confirm the env name exists (`railway environment` or dashboard); a typo becomes a silent no-op rather than an error.
5. Prefer editing `railway.toml`/`railway.json` over dashboard changes — code-as-config is diffable and reviewable.
