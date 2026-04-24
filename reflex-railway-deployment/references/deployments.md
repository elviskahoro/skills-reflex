# Deployments

A deployment is one build+run attempt for a service in an environment. Every deploy produces an immutable snapshot that you can redeploy or roll back to.

## Lifecycle

```
INITIALIZING → BUILDING → DEPLOYING → SUCCESS | FAILED | CRASHED | REMOVED
```

Phases:
1. **INITIALIZING** — source fetched, image selected, build env prepared.
2. **BUILDING** — Dockerfile or Railpack builds the image.
3. **DEPLOYING** — image starts; pre-deploy command runs; healthcheck gates traffic switch.
4. **SUCCESS** — healthy, serving traffic.
5. **FAILED** — build or start failed.
6. **CRASHED** — was SUCCESS, then process died and restart policy gave up.
7. **REMOVED** — torn down (replaced by newer deploy, or manually).

## Pre-deploy command

Runs **once** in the **same container** as the main start process, **before** the start command. Most common use: database migrations.

```toml
[deploy]
preDeployCommand = ["python manage.py migrate"]
```

Rules:
- Runs once per deploy attempt, not per replica.
- Same filesystem + env as runtime — any file generated is visible to the start command.
- Non-zero exit fails the deploy before it goes live.
- Multiple commands: pass as separate array items or chain with `&&` inside one string.

**Do not** put long-running processes here. Use a separate service for workers/queues.

## Start command

Auto-detected by Railpack for common frameworks. For Dockerfile/image services, defaults to `ENTRYPOINT`/`CMD`.

**Exec form gotcha:** Dockerfile/image start commands run in exec form, so `$VAR` is **not expanded**. If you need env var expansion:

```
/bin/sh -c "exec python main.py --port $PORT"
```

Railpack-built services run via shell automatically — no wrap needed.

Framework-specific starts (override when Railpack's guess is wrong):
- Node: `node main.js`
- Next.js: `npx next start --port $PORT`
- Nuxt: `node .output/server/index.mjs`
- FastAPI: `uvicorn main:app --host 0.0.0.0 --port $PORT`
- Flask: `gunicorn main:app`
- Django: `gunicorn myproject.wsgi`
- Rails: `bundle exec rails server -b 0.0.0.0 -p $PORT`
- Vite/CRA: `serve --single --listen $PORT dist` (or `build`) — needs `serve` package.
- Reflex: `.venv/bin/reflex run --env prod` (port auto-read from `$PORT`).

## Healthchecks

Railway queries `healthcheckPath` until it gets HTTP 200, then switches traffic from the old deploy to the new. No 200 within `healthcheckTimeout` ⇒ deploy fails.

**Key facts:**
- Only invoked **at deploy time**, not continuously. For continuous monitoring, add an external monitor (e.g. Uptime Kuma template).
- Hits from `healthcheck.railway.app` — allow-list this host if you filter by hostname.
- Uses the `PORT` env var by default; if you listen on a different port, manually set `PORT` as a service variable.
- Default timeout: 300 s. Override with `healthcheckTimeout` in config or `RAILWAY_HEALTHCHECK_TIMEOUT_SEC`.
- Services with **attached volumes** always incur brief downtime — Railway refuses to run two deployments with the same volume mounted. Healthcheck doesn't rescue zero-downtime here.

**Common failures:**
- "service unavailable" ⇒ app isn't listening on `PORT`, or host allow-list excludes `healthcheck.railway.app`.
- "failed with status 400" ⇒ hostname filter; or endpoint exists but requires auth.
- Deploy hangs 5 min then fails ⇒ endpoint is wrong path / not returning 200.

## Restart policy

Applied when the running process exits.

| Policy | Behavior |
|---|---|
| `ON_FAILURE` (default) | Restart only on non-zero exit. |
| `ALWAYS` | Restart regardless (also for exit code 0). |
| `NEVER` | No automatic restart. |

**Plan limits:**
- Free / trial: `ALWAYS` is not available. `ON_FAILURE` capped at 10 retries.
- Paid: any policy, any retry count.

Config:
```toml
[deploy]
restartPolicyType = "ON_FAILURE"
restartPolicyMaxRetries = 10
```

Multiple replicas: a restart only affects the crashed replica; others keep serving.

## Deployment teardown (zero-downtime tuning)

Two knobs control the rollout window:

| Key | Meaning | Default |
|---|---|---|
| `overlapSeconds` | Seconds the old deploy keeps serving after new deploy goes healthy. Use for in-flight connection completion. | 0 |
| `drainingSeconds` | Seconds between SIGTERM and SIGKILL when tearing down the old deploy. | 0 |

Typical web app:
```toml
[deploy]
overlapSeconds = "10"
drainingSeconds = "30"
```

The process **must** handle SIGTERM gracefully (close server, finish in-flight requests, close DB pool, then exit). If it ignores SIGTERM, Railway SIGKILLs after `drainingSeconds`.

## SIGTERM handling (language gotchas)

- **Node.js**: default handler just exits. In PM2/npm wrappers, SIGTERM may not propagate to your app. Prefer `CMD ["node", "server.js"]` (exec form) so Node gets PID 1 and the signal. See https://docs.railway.com/deployments/troubleshooting/nodejs-sigterm-handling.
- **Python**: default handler raises `SystemExit` on SIGTERM, which gunicorn/uvicorn handle gracefully. If running under `sh -c "..."`, wrap with `exec` so signals reach Python: `/bin/sh -c "exec python main.py"`.
- **Go/Rust/Java**: must implement explicit handlers.

## Regions

Services default to a single region. Override per-service, and configure multi-region via `multiRegionConfig`:

```toml
[deploy.multiRegionConfig]
us-west2 = { numReplicas = 2 }
europe-west4-drams3a = { numReplicas = 1 }
```

Common region slugs: `us-west2`, `us-east4-eqdc4a`, `europe-west4-drams3a`, `asia-southeast1-eqsg3a`. Check https://docs.railway.com/deployments/regions for the current list. Region availability depends on plan.

## Scaling

**Vertical**: plan-dependent — bigger plans get more CPU/memory per container. Can't be overridden in config.

**Horizontal**: `numReplicas` (single region) or `multiRegionConfig` (across regions). Paid plans only for >1 replica.

```toml
[deploy]
numReplicas = 3
```

Stateless services only — replicas don't share storage. A single volume cannot mount on multiple replicas.

## GitHub autodeploys

Enabled by default when a service is linked to a repo. Push to the linked branch ⇒ new deploy. Disable per-service in settings, or gate with Deployment Approvals (for external contributors).

## Serverless (experimental)

Services can scale-to-zero when idle. See https://docs.railway.com/deployments/serverless. Cold-start caveats and pricing differ from persistent services; mostly useful for low-traffic APIs.

## Image auto-updates

For Docker image services, Railway polls upstream for new digests. Updating a floating tag (`:latest`) redeploys to the latest digest; versioned tags stage an explicit update. Configurable schedule + maintenance window. Volume-attached services incur brief downtime.

## Observability

- **Logs**: `railway logs` or dashboard Logs tab. Structured logs parsed when JSON.
- **Metrics**: CPU, memory, network, HTTP in dashboard Metrics tab.
- **Webhooks**: Project Settings → Webhooks for deploy-status notifications.

## Optimize performance

See https://docs.railway.com/deployments/optimize-performance. Key ideas:
- Cache mounts for package managers (see [builds.md](builds.md)).
- Multi-stage Dockerfiles.
- Watch patterns to skip unnecessary builds.
- Pre-deploy only for tasks that truly need the deploy's new code.
