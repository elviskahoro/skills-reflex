# skills-reflex

Claude Code skills for building and deploying [Reflex](https://reflex.dev) web applications.

## Skills

### reflex-web-framework

Build full-stack web apps in pure Python using the Reflex framework. This skill covers:

- Project setup and structure (`reflex init`, `reflex run`)
- UI components in Python (Radix UI, charts, forms, tables) with Buridan UI component library
- Reactive state management with `rx.State`
- Compile-time vs. runtime patterns (`rx.cond`, `rx.foreach`, `rx.match`)
- Pages, routing, and dynamic routes
- Styling (CSS props, theming, responsive design)
- Database integration via SQLModel/SQLAlchemy with Alembic migrations
- Deployment (self-hosted, Docker, Reflex Cloud)

**Requirements:** Python 3.11+, `reflex` pip package, `buridan-ui` CLI, Node.js

### reflex-gcp-deployment

Deploy a Reflex app to Google Cloud Platform using a split architecture:

```
Internet -> Firebase Hosting (static frontend)
                |  HTTPS
             Cloud Run (Reflex backend, --backend-only)
                |  VPC private network
             Memorystore Redis (state manager)
```

Covers the full deployment pipeline:

- VPC networking and Serverless VPC connector setup
- Memorystore Redis for production state management
- Dockerfile and Cloud Run service configuration
- Cloud Build CI/CD triggers (GitHub integration)
- Firebase Hosting for the static frontend export
- Troubleshooting connectivity, WebSocket, and Redis issues

**Requirements:** `gcloud` CLI, `firebase-tools` CLI, Docker, Python 3.11+, a working Reflex app

**Estimated cost:** ~$42/mo minimum (Redis + VPC connector)

## Installation

Copy the `.agents/skills/` directory into your project's `.agents/skills/` folder. The skills will be available to Claude Code automatically.

## Author

[@elviskahoro](https://github.com/elviskahoro)
