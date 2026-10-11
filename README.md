# Deploy and Host Plane with Railway

[![Deploy on Railway](https://railway.com/button.svg)](https://railway.com/deploy/plane-v1-project-hub?referralCode=20H68q)

Plane is an open-source project management tool for issues, cycles (sprints), modules, pages and roadmaps. This community template deploys the full Plane community edition on Railway as separate services from the official images, with Postgres, Redis, RabbitMQ and a Railway Bucket, behind one domain and with no per-seat fees.

## About Hosting Plane

A complete Plane install is more than one container: a Django API, Celery workers and a scheduler, a real-time server for collaborative pages, three front-end apps (main app, public spaces, instance admin) and a reverse proxy that serves them all from one origin. It also needs Postgres, Redis, RabbitMQ and S3-compatible storage. This template runs each process as its own Railway service on the private network, routes the public domain through a small Caddy router, stores uploads in a Railway Bucket using presigned URLs, runs migrations as a pre-deploy step and generates every secret. Instance settings such as SMTP and OAuth are managed in Plane's own admin UI.

## Common Use Cases

- Replacing Jira or Linear for engineering and product teams that want to own their data
- Planning sprints (cycles), epics (modules) and roadmaps with Kanban, list, Gantt and spreadsheet views
- Writing specs and meeting notes in collaborative Pages next to the work items
- Publishing a read-only roadmap or feedback board through Plane spaces

## Dependencies for Plane Hosting

- Plane community images: backend (api, worker, beat-worker), frontend, admin, space, live — pinned to v1.4.2
- PostgreSQL (Railway Postgres 18)
- Redis (cache, sessions, live-server pub/sub)
- RabbitMQ (Celery broker)
- S3-compatible object storage (Railway Bucket)

### Deployment Dependencies

- Plane self-hosting docs: https://developers.plane.so/self-hosting/overview
- Plane Docker Compose reference (v1.4.2): https://github.com/makeplane/plane/tree/v1.4.2/deployments/cli/community
- Plane releases: https://github.com/makeplane/plane/releases
- Railway Buckets (CORS for browser uploads): https://docs.railway.com/storage-buckets/uploading-serving

### Implementation Details

| Service | Image | Role |
|---|---|---|
| proxy | Caddy 2.11 (`services/proxy`) | Only public domain; routes `/api`, `/auth`, `/spaces`, `/god-mode`, `/live` and the web app |
| web | `makeplane/plane-frontend:v1.4.2` | Main web app |
| admin | `makeplane/plane-admin:v1.4.2` | Instance admin (`/god-mode`) |
| space | `makeplane/plane-space:v1.4.2` | Public spaces (`/spaces`) |
| live | `makeplane/plane-live:v1.4.2` | Real-time collaboration (`/live`, WebSocket) |
| api | `makeplane/plane-backend:v1.4.2` | Django API; migrations in pre-deploy |
| worker | `makeplane/plane-backend:v1.4.2` | Celery worker |
| beat-worker | `makeplane/plane-backend:v1.4.2` | Celery beat scheduler |
| Postgres / Redis / RabbitMQ | Railway Postgres 18, Redis 8.2, RabbitMQ 3.13 | Data, cache, broker (each with a volume) |
| bucket-cors | `amazon/aws-cli` | One-shot job that sets the Bucket CORS rule, then exits |
| Bucket | Railway Bucket | Attachments, images, exports |

**First login:** open `/god-mode/` on the proxy domain to create the instance admin and configure e-mail and sign-in methods, then open `/` to create your first workspace.

**Scaling:** add replicas to worker (or raise `CELERY_WORKER_CONCURRENCY`) for background load and to api (or raise `GUNICORN_WORKERS`) for request load. Keep beat-worker at one replica.

**Versions:** all Plane services share one tag. To upgrade, change the seven `v1.4.2` tags together and redeploy; migrations run once in the api pre-deploy step. For a custom domain, update `WEB_URL` and `CORS_ALLOWED_ORIGINS` on api and the CORS rule on bucket-cors.

### Why Deploy Plane on Railway?

Railway runs Plane's many small services, its databases and the file bucket in one project with private networking, per-service scaling and usage-based pricing. You get the same layout Plane uses in its own Docker deployments without managing hosts, proxies or certificates.
