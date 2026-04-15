# Troubleshooting Reference

Debugging guide for common issues when deploying Reflex to GCP.

## Diagnostic Checklist

Run these checks in order when something isn't working:

```bash
# 1. Is Cloud Run service healthy?
CLOUD_RUN_URL=$(gcloud run services describe <SERVICE_NAME> \
  --region=<REGION> --format="value(status.url)")
curl -s "${CLOUD_RUN_URL}/ping"
# Expected: "pong"

# 2. Is VPC connector ready?
gcloud compute networks vpc-access connectors describe <VPC_CONNECTOR> \
  --region=<REGION> --format="value(state)"
# Expected: READY

# 3. Is Redis instance running?
gcloud redis instances describe reflex-redis \
  --region=<REGION> --format="value(state)"
# Expected: READY

# 4. Can Cloud Run reach Redis? (check logs)
gcloud run services logs read <SERVICE_NAME> \
  --region=<REGION> --limit=50 2>&1 | grep -i redis

# 5. Is frontend pointing to correct backend?
# Open browser DevTools → Network → check WebSocket URL
# Should be: wss://<CLOUD_RUN_URL>/_event
```

---

## Backend Issues

### `/ping` Returns Nothing or 502/503

**Symptoms:** `curl <URL>/ping` times out, returns 502, or returns 503.

**Causes and fixes:**

| Cause | Diagnosis | Fix |
|-------|-----------|-----|
| Container failing to start | Check Cloud Run logs | Fix Dockerfile or dependencies |
| Wrong container port | Logs show "listening on 8000" but Cloud Run expects different | Set container port to `8000` in Cloud Run config |
| Dependencies missing | `ModuleNotFoundError` in logs | Add missing packages to `requirements.txt` |
| Reflex init not run | `.web` directory missing errors | Add `reflex init` to Dockerfile before `run` |
| Out of memory | Container killed (OOMKilled) | Increase memory limit in Cloud Run |

**Check logs:**

```bash
# Recent logs
gcloud run services logs read <SERVICE_NAME> --region=<REGION> --limit=100

# Live tail
gcloud run services logs tail <SERVICE_NAME> --region=<REGION>

# Filter for startup errors
gcloud logging read \
  "resource.type=cloud_run_revision AND resource.labels.service_name=<SERVICE_NAME> AND severity>=ERROR" \
  --limit=20 --format="value(textPayload)"
```

### Container Starts But Crashes Immediately

**Symptoms:** Logs show Reflex starting, then container exits.

**Common causes:**

1. **Missing REDIS_URL or wrong IP:**
   ```
   ConnectionError: Error connecting to redis://wrong-ip:6379
   ```
   Fix: Verify Redis IP with `gcloud redis instances describe reflex-redis --region=<REGION> --format="value(host)"`

2. **Database connection failure (if using DB):**
   ```
   OperationalError: could not connect to server
   ```
   Fix: Check `DB_URL` env var, ensure database is accessible from VPC

3. **Reflex version mismatch:**
   ```
   AttributeError: module 'reflex' has no attribute '...'
   ```
   Fix: Pin reflex version in requirements.txt, rebuild

### Cold Start Timeouts

**Symptoms:** First request after idle period takes 10-30+ seconds or times out.

**Fixes:**

```bash
# Set minimum instances to 1 (eliminates cold starts, costs more)
gcloud run services update <SERVICE_NAME> \
  --region=<REGION> \
  --min-instances=1

# Enable CPU boost (faster cold starts)
gcloud run services update <SERVICE_NAME> \
  --region=<REGION> \
  --cpu-boost
```

---

## Redis Connectivity Issues

### Cloud Run Can't Reach Redis

**Symptoms:** Logs show `ConnectionRefusedError`, `ConnectionError`, or `TimeoutError` for Redis.

**Diagnostic steps:**

```bash
# 1. Verify Redis is on the correct network
gcloud redis instances describe reflex-redis \
  --region=<REGION> --format="value(authorizedNetwork)"
# Should show: projects/<PROJECT_ID>/global/networks/default

# 2. Verify VPC connector is on the same network
gcloud compute networks vpc-access connectors describe <VPC_CONNECTOR> \
  --region=<REGION> --format="value(network)"
# Should show: default (or match Redis network)

# 3. Verify Cloud Run service has VPC connector attached
gcloud run services describe <SERVICE_NAME> \
  --region=<REGION> --format="value(spec.template.metadata.annotations)"
# Should include: run.googleapis.com/vpc-access-connector
```

**Common fixes:**

| Problem | Fix |
|---------|-----|
| VPC connector not attached to Cloud Run | Redeploy with `--vpc-connector=<CONNECTOR>` |
| VPC connector on wrong network | Recreate connector on the same network as Redis |
| VPC egress not set | Add `--vpc-egress=all-traffic` to Cloud Run deploy |
| Redis on different region than Cloud Run | Recreate Redis in the same region |
| IP range conflict | Recreate VPC connector with different `--range` |

### Redis AUTH Failures

**Symptoms:** `AuthenticationError` or `NOAUTH Authentication required` in logs.

```bash
# Check if AUTH is enabled
gcloud redis instances describe reflex-redis \
  --region=<REGION> --format="value(authEnabled)"

# If true, get the auth string
gcloud redis instances get-auth-string reflex-redis --region=<REGION>

# Update REDIS_URL to include auth
# redis://:AUTH_STRING@IP:6379
```

---

## Frontend Issues

### Frontend Loads But Can't Connect to Backend

**Symptoms:** App renders but state doesn't update, buttons don't work. Browser console shows WebSocket errors.

**Diagnosis:**

1. Open browser DevTools → Network tab → WS (WebSocket) filter
2. Look for `_event` connection
3. Check the WebSocket URL — should be `wss://<CLOUD_RUN_URL>/_event`

**Common causes:**

| Cause | Fix |
|-------|-----|
| `API_URL` not set during export | Re-export with `API_URL=<CLOUD_RUN_URL> reflex export --frontend-only --no-zip` |
| `API_URL` points to wrong URL | Verify Cloud Run URL, re-export |
| Mixed content (HTTP frontend → HTTPS backend) | Both must be HTTPS |
| CORS issues | Reflex handles CORS by default; check if custom middleware blocks it |
| Cloud Run not allowing unauthenticated | Redeploy with `--allow-unauthenticated` |

### 404 on Page Refresh

**Symptoms:** Direct URL navigation works on initial load but returns 404 on page refresh.

Reflex uses client-side routing. Firebase needs to serve `index.html` for all routes:

```json
{
  "hosting": [{
    "site": "<SITE>",
    "public": "<APP_DIR>/.web/_static",
    "rewrites": [
      {"source": "**", "destination": "/index.html"}
    ]
  }]
}
```

**Note:** This is a catch-all rewrite. If your app has API routes on the same domain, be more specific.

### Stale Frontend After Deploy

**Symptoms:** Old version of the app shows after Firebase deploy.

**Fixes:**

1. Clear browser cache (hard refresh: Ctrl+Shift+R / Cmd+Shift+R)
2. Check Firebase deploy actually completed:
   ```bash
   firebase hosting:releases:list --site <SITE> --limit=3
   ```
3. CDN cache may take a few minutes to propagate globally

---

## Cloud Build Issues

### Trigger Doesn't Fire

**Symptoms:** You push to the repo but no build starts.

**Check:**

```bash
# List triggers
gcloud builds triggers list --region=global

# Check trigger configuration
gcloud builds triggers describe <TRIGGER_NAME> --region=global
```

**Common causes:**

| Cause | Fix |
|-------|-----|
| Wrong branch pattern | Check `--branch-pattern` matches your branch |
| GitHub App not connected | Reconnect repo in Cloud Build → Repositories |
| Trigger disabled | Enable in Cloud Build → Triggers |
| Config file path wrong | Verify `cloudbuild.yaml` path matches repo |

### Build Succeeds But Deploy Fails

**Symptoms:** Docker image builds and pushes, but the Deploy step fails.

**Check Cloud Build logs:**

```bash
gcloud builds list --limit=5 --region=global
gcloud builds log <BUILD_ID> --region=global
```

**Common causes:**

| Error | Fix |
|-------|-----|
| `Permission denied on Cloud Run` | Grant `roles/run.admin` to build service account |
| `Service not found` | Cloud Run service name in `_SERVICE_NAME` must match exactly |
| `Region mismatch` | `_DEPLOY_REGION` must match where Cloud Run service exists |

---

## WebSocket Issues

### WebSocket Disconnects Frequently

**Symptoms:** App works briefly then loses state. Console shows WebSocket reconnection attempts.

**Causes and fixes:**

| Cause | Fix |
|-------|-----|
| No session affinity | Enable: `--session-affinity` on Cloud Run |
| Request timeout too short | Increase: `--timeout=300` or higher |
| CPU throttled between requests | Switch to "CPU always allocated" mode |
| Cloud Run scaling down | Set `--min-instances=1` |
| Proxy/CDN timeout | Check if any proxy between client and Cloud Run has a short timeout |

### WebSocket Upgrade Fails (101 → 400)

**Symptoms:** WebSocket connection attempt returns 400 instead of 101.

Cloud Run supports WebSocket natively. If you see this:

1. Ensure Cloud Run service allows HTTP/1.1 (WebSocket upgrade requires it)
2. Do NOT enable "HTTP/2 end-to-end" — it breaks WebSocket upgrade on Cloud Run
3. Check for middleware or reverse proxies that strip `Upgrade` headers

---

## Cost Debugging

### Unexpected High Costs

```bash
# Check Cloud Run usage
gcloud run services describe <SERVICE_NAME> \
  --region=<REGION> --format="yaml(status.traffic)"

# Check active revisions (old revisions may still be serving)
gcloud run revisions list --service=<SERVICE_NAME> --region=<REGION>

# Check VPC connector instances
gcloud compute networks vpc-access connectors describe <VPC_CONNECTOR> \
  --region=<REGION>
```

**Cost reduction tips:**

| Action | Savings |
|--------|---------|
| Set `--min-instances=0` | No cost when idle |
| Use request-based CPU | Lower cost for low-traffic apps |
| Reduce VPC connector min-instances | From 2 to 2 (minimum) |
| Use Basic tier Redis | ~$35/mo vs ~$70/mo for Standard |
| Clean up old Artifact Registry images | Storage costs |
| Delete unused Cloud Run revisions | No cost, but reduces clutter |

---

## Useful Commands Quick Reference

```bash
# === Cloud Run ===
gcloud run services list --region=<REGION>
gcloud run services describe <SERVICE> --region=<REGION>
gcloud run services logs read <SERVICE> --region=<REGION> --limit=50
gcloud run services logs tail <SERVICE> --region=<REGION>
gcloud run revisions list --service=<SERVICE> --region=<REGION>

# === Redis ===
gcloud redis instances list --region=<REGION>
gcloud redis instances describe reflex-redis --region=<REGION>

# === VPC ===
gcloud compute networks vpc-access connectors list --region=<REGION>
gcloud compute networks vpc-access connectors describe <CONNECTOR> --region=<REGION>

# === Cloud Build ===
gcloud builds list --limit=10 --region=global
gcloud builds log <BUILD_ID> --region=global
gcloud builds triggers list --region=global

# === Firebase ===
firebase hosting:releases:list --site <SITE> --limit=5
firebase hosting:sites:list
```
