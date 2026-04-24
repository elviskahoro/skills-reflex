# Builds

Two builders: **Railpack** (auto-detection, default for new services) and **Dockerfile** (wins automatically if `Dockerfile` exists).

## Dockerfile builds

### Auto-detection

Railway looks for a file literally named `Dockerfile` (capital D) at the source root. If found, it's used regardless of `builder = "RAILPACK"`. Build log confirms with:

```
==========================
Using detected Dockerfile!
==========================
```

### Custom Dockerfile path

Two ways (equivalent):

**Variable:**
```
RAILWAY_DOCKERFILE_PATH=Dockerfile.origin
RAILWAY_DOCKERFILE_PATH=/build/Dockerfile
```

**Config as code:**
```toml
[build]
dockerfilePath = "Dockerfile.origin"
```

### Using variables at build time

Railway injects Railway-provided + service variables at build time, but Dockerfile only sees them if you declare `ARG`:

```dockerfile
ARG RAILWAY_SERVICE_NAME
RUN echo $RAILWAY_SERVICE_NAME
```

Declare `ARG` **in the stage where it's needed** — multi-stage builds don't propagate.

```dockerfile
FROM node AS build
ARG NODE_ENV
RUN echo $NODE_ENV

FROM node AS runtime
ARG NODE_ENV       # must be re-declared if used here
```

### Cache mounts

Speed up repeat builds for package managers. Syntax:

```dockerfile
RUN --mount=type=cache,id=s/<service-id>-<target-path>,target=<target-path> \
    <command>
```

Env vars **cannot** be used in the `id=` — it's parsed literally.

Python/pip:
```dockerfile
RUN --mount=type=cache,id=s/abc123-/root/.cache/pip,target=/root/.cache/pip \
    pip install -r requirements.txt
```

Node/npm:
```dockerfile
RUN --mount=type=cache,id=s/abc123-/root/.npm,target=/root/.npm \
    npm ci
```

uv:
```dockerfile
RUN --mount=type=cache,id=s/abc123-/root/.cache/uv,target=/root/.cache/uv \
    uv sync --frozen
```

Look up your tool's default cache dir (e.g. `PIP_CACHE_DIR=/root/.cache/pip`).

### Docker Compose import

Drag a `compose.yaml` onto the project canvas → Railway imports services + mounted volumes as staged changes. Not every compose field is supported — expect to clean up after import.

## Railpack builds

Railpack auto-detects language/framework (Node, Python, Ruby, Go, etc.) and builds without a Dockerfile. Start commands are auto-inferred; see the troubleshooting file for overrides.

### Pin Railpack version

```toml
[build]
railpackVersion = "0.7.0"
```

Or:
```
RAILPACK_VERSION=0.7.0
```

Valid versions: https://github.com/railwayapp/railpack/releases.

### When Railpack fails

If Railpack can't detect a start command: "No start command could be found" error. See [troubleshooting-reflex.md](troubleshooting-reflex.md) for fixes, including framework-specific start commands.

## Build commands

Only meaningful for Railpack. Override in config-as-code:

```toml
[build]
buildCommand = "yarn build"
```

For Dockerfile, put the equivalent in `RUN` instructions.

## Watch patterns

Only re-build when matching files change. Patterns are globs relative to repo root.

```toml
[build]
watchPatterns = ["src/**", "Dockerfile", "package.json"]
```

Files outside the patterns trigger a **skipped build** (the deploy is skipped entirely — Railway reuses the prior image).

Common patterns:
- Monorepo: `watchPatterns = ["apps/backend/**", "packages/shared/**"]`
- Python: `watchPatterns = ["**/*.py", "pyproject.toml", "uv.lock", "Dockerfile"]`
- Node: `watchPatterns = ["src/**", "package*.json", "Dockerfile"]`

## Skipped builds

A build is skipped when:
1. No changes match `watchPatterns`, OR
2. Same git SHA as a prior successful build, OR
3. Explicitly requested (Service Settings → Deploy Triggers → Skip).

A skipped build still deploys (reusing the image). To skip the deploy too, disable deploy triggers.

## Private registries

Pro plan required. For GHCR, use a classic personal access token with `read:packages` scope, not a fine-grained one. See https://docs.railway.com/builds/private-registries.

## Build-time memory

Railway's free plan builds with ~512 MB. Alpine images, `--no-cache-dir`, virtual build deps (`apk add --virtual .build-deps` then `apk del .build-deps`), multi-stage builds, and cache mounts all reduce peak memory. Slim Python/Debian images often don't fit — see [troubleshooting-reflex.md](troubleshooting-reflex.md) for the battle-tested pattern.
