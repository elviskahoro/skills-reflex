# Cron jobs and Functions

Two ways to run work without a full service: **cron jobs** (scheduled services) and **Functions** (one-file TypeScript on Bun).

## Cron jobs

Configure a service to run on a schedule. Railway starts the container on schedule, runs the start command, and expects the process to **exit**. When it exits, the container stops — you pay only for the run time.

### Configure

Two ways (equivalent):

**Dashboard:** Service Settings → Cron Schedule → enter crontab expression.

**Config as code:**
```toml
[deploy]
cronSchedule = "*/15 * * * *"
```

### Cron expression format

```
* * * * *
│ │ │ │ │
│ │ │ │ └─ Day of week (0-7, 0 and 7 = Sunday)
│ │ │ └─── Month (1-12)
│ │ └───── Day of month (1-31)
│ └─────── Hour (0-23)
└───────── Minute (0-59)
```

Syntax:
- `*` = any
- `n,m` = list
- `n-m` = range
- `*/n` = step
- integer = literal

All times UTC.

### Examples

| Schedule | Meaning |
|---|---|
| `30 * * * *` | Every hour at :30 |
| `15 15 * * *` | Daily at 15:15 UTC |
| `0 8 * * 1` | Mondays 08:00 UTC |
| `0 0 1 * *` | 1st of each month, 00:00 UTC |
| `30 14 * 1 0` | Sundays in January, 14:30 UTC |
| `30 9 * * 1-5` | Weekdays 09:30 UTC |
| `*/15 * * * *` | Every 15 minutes |
| `0 */2 * * *` | Every 2 hours |

### Service execution requirements (critical)

The process **must exit cleanly** when work is done. If it leaves open resources (DB connections, HTTP servers, etc.) and doesn't exit, Railway will **skip the next run** instead of killing and restarting.

Symptom: "Previous execution still active" in logs; new runs silently skipped.

Fix:
- Close DB pools (e.g. `await pool.end()` in Node, `conn.close()` in Python).
- Don't `listen()` on an HTTP server — this is a job, not a service.
- Don't use in-code schedulers (`node-cron`, `APScheduler`) inside a cron service — Railway's scheduler + a long-running process conflict.

### Frequency

- **Minimum interval: 5 minutes.** Railway won't run more frequently.
- No sub-minute precision; actual start time can drift a few minutes.
- Don't use for hard-real-time scheduling.

### When to use / not use

**Use for:** short idempotent tasks (nightly backup, hourly ETL, weekly report). Save resources vs. an always-on scheduler.

**Don't use for:** web servers, Discord bots, any long-running process; workloads that must fire every <5 min; workloads needing strict timing.

### Rolling your own timezone

Railway runs in UTC. Offset manually:
- 09:00 US/Pacific (UTC-8) ⇒ `0 17 * * *` in winter, `0 16 * * *` in summer (handle DST yourself).

## Functions

Write and deploy a **single TypeScript file** that runs on the [Bun](https://bun.sh/) runtime — no repo, no build step, no Dockerfile.

A Function is a Service. It has variables, logs, metrics, deployments, and can attach volumes, like any other service.

### Key features

- **Instant deploys** — changes go live in seconds; no build phase.
- **Import any NPM package** — pin versions with `package@version` syntax:
  ```ts
  import { Hono } from "hono@4";
  import { z } from "zod@3.22";
  ```
- **Native Bun APIs** — `Bun.file()`, `Bun.serve()`, etc.
- **Env access** — service variables in `import.meta.env`, `process.env`, or `Bun.env`.
- **Volumes** — attachable for persistent state.
- **Vim bindings** in editor — `Ctrl+Option+V` (Mac) / `Ctrl+Alt+V` (Windows).

### Limitations

- **1 file** per function.
- **Max 96 KB** file size.

For anything larger, use a regular GitHub-sourced service.

### Editor workflow

- Edit: Service → Source Code tab.
- Stage change: `Cmd/Ctrl + S`.
- Deploy staged: `Shift + Enter`.

### Versioning

Every deploy creates a new version automatically. Rollback via the Deployments tab → "Redeploy" on any prior version. Source of a prior version is viewable from its deployment details.

### Example: HTTP webhook receiver

```ts
import { Hono } from "hono@4";

const app = new Hono();

app.post("/webhook", async (c) => {
  const body = await c.req.json();
  console.log("received:", body);
  // forward, store, or process here
  return c.json({ ok: true });
});

export default {
  port: Number(process.env.PORT ?? 3000),
  fetch: app.fetch,
};
```

### Example: Cron function (Function + cronSchedule)

You can combine: set a `cronSchedule` on a Function service to run a one-file TS task on a schedule.

```ts
// Nightly: delete old rows from a remote DB.
import postgres from "postgres@3";

const sql = postgres(process.env.DATABASE_URL!);
await sql`DELETE FROM events WHERE created_at < now() - interval '30 days'`;
await sql.end();
```

### When to pick Function vs. regular service

**Function:** webhook receivers, thin APIs, simple cron tasks, throwaway experiments, internal tools. No git history needed, you value instant iteration.

**Regular service:** production code you want in git, multi-file projects, anything needing a build step, apps >96 KB, anything you need to self-host locally.
