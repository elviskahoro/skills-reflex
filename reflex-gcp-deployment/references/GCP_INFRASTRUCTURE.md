# GCP Infrastructure Reference

Detailed configuration for VPC networking, Memorystore Redis, and Cloud Run services in a Reflex deployment.

## VPC Network

### Default VPC

Most GCP projects already have a `default` VPC with auto-created subnets in every region. Check:

```bash
gcloud compute networks list
gcloud compute networks describe default
```

If no default VPC exists:

```bash
gcloud compute networks create default --subnet-mode=auto
```

### Creating a Custom VPC (Optional)

Only needed if you want network isolation from other resources:

```bash
gcloud compute networks create reflex-vpc \
  --subnet-mode=custom \
  --description="VPC for Reflex app deployment"

gcloud compute networks subnets create reflex-subnet \
  --network=reflex-vpc \
  --region=us-central1 \
  --range=10.0.0.0/24
```

If using a custom VPC, substitute `default` with your VPC name in all commands.

### Serverless VPC Access Connector

The connector bridges Cloud Run (serverless) to the VPC (private network). Without it, Cloud Run cannot reach Memorystore Redis.

```bash
gcloud compute networks vpc-access connectors create staging-vpc-connector \
  --region=us-central1 \
  --network=default \
  --range=10.8.0.0/28 \
  --min-instances=2 \
  --max-instances=10 \
  --machine-type=e2-micro
```

**Parameter guide:**

| Parameter | Value | Notes |
|-----------|-------|-------|
| `--range` | `10.8.0.0/28` | /28 = 16 IPs. Must not overlap with existing subnets. |
| `--min-instances` | `2` | Minimum connector instances (cost floor) |
| `--max-instances` | `10` | Scales with traffic |
| `--machine-type` | `e2-micro` | Cheapest option, sufficient for most Reflex apps |

**Check connector status:**

```bash
gcloud compute networks vpc-access connectors describe staging-vpc-connector \
  --region=us-central1
```

States: `CREATING` → `READY` (takes 1-2 minutes)

**Troubleshooting connector creation failures:**

- **Range overlap:** Pick a different `/28` range (e.g., `10.8.1.0/28`)
- **API not enabled:** `gcloud services enable vpcaccess.googleapis.com`
- **Quota exceeded:** Check VPC connector quota in IAM & Admin → Quotas

### Firewall Rules

The default VPC has default firewall rules that allow internal traffic. If you created a custom VPC or modified default rules, ensure:

```bash
# Allow internal traffic within VPC (needed for Redis)
gcloud compute firewall-rules create allow-internal \
  --network=default \
  --allow=tcp,udp,icmp \
  --source-ranges=10.0.0.0/8
```

---

## Memorystore Redis

### Instance Creation

```bash
gcloud redis instances create reflex-redis \
  --size=1 \
  --region=us-central1 \
  --network=default \
  --redis-version=redis_7_2 \
  --tier=basic
```

**Parameter guide:**

| Parameter | Options | Notes |
|-----------|---------|-------|
| `--size` | `1` to `300` (GB) | 1GB is sufficient for most Reflex apps |
| `--tier` | `basic`, `standard` | Basic = no replication. Standard = automatic failover. |
| `--redis-version` | `redis_7_2`, `redis_7_0`, `redis_6_x` | Use latest available |
| `--network` | VPC name | **Must match** the VPC your connector is on |

### Tier Comparison

| Feature | Basic | Standard |
|---------|-------|----------|
| Replication | None | Automatic failover replica |
| Availability SLA | None | 99.9% |
| Cost (1GB) | ~$35/mo | ~$70/mo |
| Use case | Dev/staging, low-traffic prod | Production with uptime requirements |

### Getting Redis Connection Info

```bash
# Get host IP
gcloud redis instances describe reflex-redis \
  --region=us-central1 --format="value(host)"

# Get port (default 6379)
gcloud redis instances describe reflex-redis \
  --region=us-central1 --format="value(port)"

# Full connection string for Reflex
REDIS_IP=$(gcloud redis instances describe reflex-redis \
  --region=us-central1 --format="value(host)")
echo "REDIS_URL=redis://${REDIS_IP}:6379"
```

### AUTH and TLS

**AUTH (disabled by default):**

AUTH adds a password requirement. Since the instance is VPC-only, it's optional but adds defense-in-depth:

```bash
# Create with AUTH enabled
gcloud redis instances create reflex-redis \
  --size=1 \
  --region=us-central1 \
  --network=default \
  --redis-version=redis_7_2 \
  --tier=basic \
  --enable-auth

# Get the auth string
AUTH_STRING=$(gcloud redis instances get-auth-string reflex-redis \
  --region=us-central1 --format="value(authString)")

# Connection string with AUTH
echo "REDIS_URL=redis://:${AUTH_STRING}@${REDIS_IP}:6379"
```

If AUTH is enabled, update the Dockerfile `ENV REDIS_URL` accordingly.

**TLS (disabled by default):**

TLS encrypts traffic between Cloud Run and Redis. For most internal VPC deployments this is unnecessary, but for compliance:

```bash
gcloud redis instances create reflex-redis \
  --size=1 \
  --region=us-central1 \
  --network=default \
  --redis-version=redis_7_2 \
  --transit-encryption-mode=SERVER_AUTHENTICATION
```

Connection string with TLS: `rediss://${REDIS_IP}:6378` (note: double `s`, port 6378)

### Monitoring Redis

```bash
# Check instance status
gcloud redis instances describe reflex-redis --region=us-central1

# List all instances
gcloud redis instances list --region=us-central1
```

In Cloud Console: Memorystore → Redis → select instance → Monitoring tab shows memory usage, connected clients, operations/sec.

### Deleting Redis Instance

```bash
gcloud redis instances delete reflex-redis --region=us-central1
```

**Warning:** This is irreversible and data is lost immediately.

---

## Cloud Run Service

### Initial Deployment

```bash
gcloud run deploy perfwriter-reflex \
  --source=reflex_app/ \
  --region=us-central1 \
  --port=8000 \
  --allow-unauthenticated \
  --vpc-connector=staging-vpc-connector \
  --vpc-egress=all-traffic \
  --session-affinity \
  --execution-environment=gen2 \
  --cpu-boost \
  --max-instances=100 \
  --min-instances=0 \
  --concurrency=1000 \
  --timeout=300
```

### Configuration Parameter Reference

| Parameter | Value | Why |
|-----------|-------|-----|
| `--port=8000` | Reflex backend port | Must match Reflex default |
| `--allow-unauthenticated` | Public access | Frontend needs to reach backend |
| `--vpc-connector` | Your connector name | Required for Redis access |
| `--vpc-egress=all-traffic` | Route all traffic through VPC | Ensures Redis reachability. Alternative `private-ranges-only` may work but is less reliable. |
| `--session-affinity` | Sticky sessions | WebSocket connections need to stay on the same instance |
| `--execution-environment=gen2` | 2nd gen runtime | Better CPU/network performance, full Linux |
| `--cpu-boost` | Startup CPU boost | Faster cold starts |
| `--max-instances=100` | Scale ceiling | Adjust based on budget/traffic |
| `--min-instances=0` | Scale to zero | Cost saving. Set to `1` to avoid cold starts. |
| `--concurrency=1000` | Requests per instance | High because of WebSocket + Redis. Reflex can handle many concurrent WS connections per process. |
| `--timeout=300` | 5 minute request timeout | Generous for WebSocket upgrade + long-running requests |

### CPU Allocation Strategy

Two modes:

| Mode | Flag | Behavior | Cost | Use When |
|------|------|----------|------|----------|
| Request-based | `--no-cpu-throttling=false` (default) | CPU only during requests | Lower | Light traffic, cost-sensitive |
| Always-on | `--no-cpu-throttling` | CPU always allocated | Higher | Heavy WebSocket use, background tasks |

For Reflex apps with active WebSocket connections, "always-on" prevents the backend from losing CPU between HTTP requests while maintaining WebSocket state. However, request-based works fine for most apps — Cloud Run keeps CPU alive during active WebSocket frames.

### Updating an Existing Service

```bash
# Update configuration without redeploying
gcloud run services update perfwriter-reflex \
  --region=us-central1 \
  --min-instances=1 \
  --concurrency=500

# Update with new image
gcloud run services update perfwriter-reflex \
  --region=us-central1 \
  --image=us-central1-docker.pkg.dev/PROJECT_ID/cloud-run-source-deploy/REPO/SERVICE:TAG
```

### Traffic Management

```bash
# Route 100% to latest
gcloud run services update-traffic perfwriter-reflex \
  --region=us-central1 \
  --to-latest

# Split traffic between revisions (canary)
gcloud run services update-traffic perfwriter-reflex \
  --region=us-central1 \
  --to-revisions=perfwriter-reflex-00010-abc=90,perfwriter-reflex-00011-def=10

# List revisions
gcloud run revisions list --service=perfwriter-reflex --region=us-central1
```

### Environment Variables

Set env vars on Cloud Run (alternative to hardcoding in Dockerfile):

```bash
gcloud run services update perfwriter-reflex \
  --region=us-central1 \
  --set-env-vars="REDIS_URL=redis://10.60.48.3:6379,PYTHONUNBUFFERED=1"
```

This is more flexible than Dockerfile `ENV` — you can change values without rebuilding.

### Viewing Logs

```bash
# Stream live logs
gcloud run services logs read perfwriter-reflex --region=us-central1 --limit=50

# Tail logs
gcloud run services logs tail perfwriter-reflex --region=us-central1

# Filter for errors
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=perfwriter-reflex AND severity>=ERROR" --limit=20
```

### Deleting a Service

```bash
gcloud run services delete perfwriter-reflex --region=us-central1
```
