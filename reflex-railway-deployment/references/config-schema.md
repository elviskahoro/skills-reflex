# railway.toml / railway.json — full reference

Source: https://docs.railway.com/config-as-code/reference

Railway reads `railway.toml` or `railway.json` at the source root. Both accept the same keys; TOML uses `[build]`/`[deploy]`/`[env]` tables, JSON uses `{"build": {}, "deploy": {}, "environments": {}}`. Schema URL for editor validation: `https://railway.com/railway.schema.json`.

## Resolution priority

For each deploy, settings are resolved in this order (highest wins):

**Non-PR environment:**
1. `environments.<name>` config in code
2. Base config in code
3. Dashboard service settings

**PR (ephemeral) environment:**
1. `environments.<ephemeral-env-name>` config in code
2. `environments.pr` config in code (magic name)
3. `environments.<base-env-name>` config in code
4. Base config in code
5. Dashboard service settings

Code-as-config applies **only to that deployment** — the dashboard is never updated from code.

## `[build]`

| Key | Type | Valid values | Notes |
|---|---|---|---|
| `builder` | string | `RAILPACK` (default), `DOCKERFILE` | If a file literally named `Dockerfile` exists, Railway uses it regardless of this value. |
| `watchPatterns` | array of globs | e.g. `["src/**", "Dockerfile"]` | Paths whose changes trigger a new build. Files outside the patterns skip builds. |
| `buildCommand` | string \| null | e.g. `"yarn build"` | For Railpack builds. Ignored for Dockerfile. |
| `dockerfilePath` | string \| null | e.g. `"Dockerfile.backend"` | Non-standard Dockerfile path. Equivalent env var: `RAILWAY_DOCKERFILE_PATH`. |
| `railpackVersion` | string \| null | e.g. `"0.7.0"` | Pin Railpack version. Equivalent env var: `RAILPACK_VERSION`. |

## `[deploy]`

| Key | Type | Valid values | Notes |
|---|---|---|---|
| `startCommand` | string \| null | e.g. `"node index.js"` | Dockerfile/image: overrides `ENTRYPOINT` in exec form — wrap in `/bin/sh -c "..."` if you need `$VAR` expansion. Railpack: runs via shell automatically. |
| `preDeployCommand` | array of strings | e.g. `["npm run db:migrate"]` | Runs once in the same container as the app, before the start command. See deployments/pre-deploy-command for details. |
| `healthcheckPath` | string \| null | e.g. `"/health"` | Hit from `healthcheck.railway.app`. Must return HTTP 200 within timeout for deploy to go live. Not used for continuous monitoring. |
| `healthcheckTimeout` | integer \| null | seconds, default 300 | Max wait for 200. Equivalent env var: `RAILWAY_HEALTHCHECK_TIMEOUT_SEC`. |
| `restartPolicyType` | string | `ON_FAILURE` (default), `ALWAYS`, `NEVER` | Free plan: `ALWAYS` unavailable. |
| `restartPolicyMaxRetries` | integer \| null | e.g. `5` | Free plan caps `ON_FAILURE` at 10. |
| `cronSchedule` | string \| null | crontab, e.g. `"*/15 * * * *"` | Minimum every 5 min; UTC. Service must exit on completion. |
| `overlapSeconds` | string \| null | e.g. `"60"` | How long old deploy overlaps with new during rollout. Default 0. Env var: `RAILWAY_DEPLOYMENT_OVERLAP_SECONDS`. |
| `drainingSeconds` | string \| null | e.g. `"10"` | Seconds between SIGTERM and SIGKILL during teardown. Default 0. Env var: `RAILWAY_DEPLOYMENT_DRAINING_SECONDS`. |
| `multiRegionConfig` | object \| null | see below | Horizontal scaling across regions. |
| `numReplicas` | integer | default 1 | Replicas in the default region. Paid plans only for >1. |

### `multiRegionConfig`

Map of region slug → `{ numReplicas: N }`:

```json
{
  "deploy": {
    "multiRegionConfig": {
      "us-west2": { "numReplicas": 2 },
      "us-east4-eqdc4a": { "numReplicas": 2 },
      "europe-west4-drams3a": { "numReplicas": 2 },
      "asia-southeast1-eqsg3a": { "numReplicas": 2 }
    }
  }
}
```

Available region slugs: `us-west2`, `us-east4-eqdc4a`, `europe-west4-drams3a`, `asia-southeast1-eqsg3a` (check https://docs.railway.com/deployments/regions for current list).

## `[env]` (TOML only — sets env vars for all envs via code)

```toml
[env]
NODE_ENV = "production"
PYTHONUNBUFFERED = "1"
```

These are **less flexible** than dashboard/Shared Variables — they can't reference other services (`${{Postgres.DATABASE_URL}}`). Use them for static, non-secret flags only. For secrets and reference variables, prefer dashboard Service/Shared Variables.

## Environment overrides

TOML:
```toml
[environments.staging.deploy]
startCommand = "npm run staging"

[environments.pr.build]
buildCommand = "npm run build:pr"
```

JSON:
```json
{
  "environments": {
    "staging": { "deploy": { "startCommand": "npm run staging" } },
    "pr": { "deploy": { "startCommand": "npm run pr" } }
  }
}
```

`pr` is a reserved environment name applied to every ephemeral PR deploy. A named ephemeral env (if you use persistent PR envs) takes priority over `pr`.

## Complete example (Dockerfile-built app with Postgres migration)

```toml
[build]
builder = "DOCKERFILE"
watchPatterns = ["src/**", "Dockerfile", "package.json"]

[deploy]
startCommand = "/bin/sh -c \"exec node dist/server.js --port $PORT\""
preDeployCommand = ["node dist/migrate.js"]
healthcheckPath = "/healthz"
healthcheckTimeout = 60
restartPolicyType = "ON_FAILURE"
restartPolicyMaxRetries = 5
drainingSeconds = 30
overlapSeconds = 10

[deploy.multiRegionConfig]
us-west2 = { numReplicas = 2 }
europe-west4-drams3a = { numReplicas = 1 }

[environments.staging.deploy]
healthcheckPath = "/_ping"
preDeployCommand = []                   # skip migrations on staging

[environments.pr.deploy]
startCommand = "/bin/sh -c \"exec node dist/server.js --pr --port $PORT\""
preDeployCommand = []
```

## Validation tips

- Add the `$schema` field in JSON to enable editor autocomplete and validation.
- `null` vs omission: most fields accept explicit `null` to clear a dashboard value; omission means "inherit from priority chain."
- Setting a dashboard value AND code value for the same key ⇒ code wins (silently).
- A typo in an environment name under `environments.<name>` becomes a silent no-op — there's no config error for unknown environments.
