# Plane v1 Project Hub — Railway template spec

## Listing

| Field | Value |
|---|---|
| Name | **Plane v1 Project Hub** (version label `v1` = Plane major version; Plane tags releases `v1.x.y`) |
| Description | Issues, sprints, docs and roadmaps. Full Plane stack, no per-seat fees. (81 chars, unchanged from LISTINGS.md) |
| Category | Other |
| Tags | plane, project-management, issues, jira-alternative, linear-alternative, kanban |
| User input at deploy | **None.** All secrets are generated. SMTP, OAuth and other instance settings are configured in Plane's admin UI (`/god-mode`). |

## Base market template

- **`plane-jira-alternative` "Plane"** (split services, v1.4.0, 10 recent projects) — chosen as base for the service layout: official split images, Caddy router on one domain, migrations in the API pre-deploy, RabbitMQ with a fixed node name. Replaced: MinIO (excluded by the standard), the Caddyfile stored in an environment variable, v1.4.0 → v1.4.2, IPv4-only listeners, uncapped Celery concurrency, missing healthchecks.
- `plane` "Plane | Jira and Linear Alternative on Railway" (AIO image `makeplane/plane-aio-community:stable`, 98 recent) — rejected as base: one container runs every process, unpinned `stable` tag, MinIO exposed on a second public domain.
- `MrCqPF` "Plane" (2024, unpinned images, MinIO + console, bitnami Redis on a TCP proxy) — outdated; nothing taken.
- `_upstream/railwayapp-templates__plane-caddy-proxy` (MIT) — read for the Railway trusted-proxy range; no file copied. `_upstream/railwayapp-templates__minio-console` — not used (MinIO).

**Split vs. all-in-one.** The split layout runs every Plane process as its own Railway service from the official images, which is how Plane's own compose and Kubernetes charts deploy it. It costs a little more idle memory than the AIO image but lets workers, API and live server scale and restart independently, and every service has its own healthcheck. All inter-service traffic stays on the private network; only the router has a domain.

## Pinned versions

| Service | Image | Tag | Release date | Checked |
|---|---|---|---|---|
| api, worker, beat-worker | `makeplane/plane-backend` | `v1.4.2` | 2026-08-23 | 2026-10-10 |
| web | `makeplane/plane-frontend` | `v1.4.2` | 2026-08-23 | 2026-10-10 |
| admin | `makeplane/plane-admin` | `v1.4.2` | 2026-08-23 | 2026-10-10 |
| space | `makeplane/plane-space` | `v1.4.2` | 2026-08-23 | 2026-10-10 |
| live | `makeplane/plane-live` | `v1.4.2` | 2026-08-23 | 2026-10-10 |
| proxy | `caddy` (built from `services/proxy`) | `2.11.7-alpine` | same Caddy minor as Plane's proxy image | 2026-10-10 |
| RabbitMQ | `rabbitmq` | `3.13.7-management-alpine` | upstream pins the 3.13 line; latest patch | 2026-10-10 |
| bucket-cors | `amazon/aws-cli` | `2.37.12` | latest | 2026-10-10 |
| Postgres | `ghcr.io/railwayapp-templates/postgres-ssl` | `18` | Railway base | 2026-10-10 |
| Redis | `redis` | `8.2` | Railway base (upstream uses Valkey 7.2, protocol-compatible) | 2026-10-10 |

`v1.4.3-rc1` exists (2026-10-06) but is a release candidate. Upgrade path: change all seven `v1.4.2` tags together and redeploy; `api` runs `migrate` in its pre-deploy step and the workers wait for migrations.

## Architecture

```
Internet ──HTTPS──▶ proxy (Caddy, only public domain, :8080)
                     ├─ /spaces/*            ─▶ space  :3000 (React Router SSR)
                     ├─ /god-mode/*          ─▶ admin  :3000 (nginx, static)
                     ├─ /live/*  (WebSocket) ─▶ live   :3000 (collaborative editing)
                     ├─ /api/* /auth/* /static/* ─▶ api :8000 (Django, gunicorn/uvicorn)
                     └─ everything else      ─▶ web    :3000 (nginx, static SPA)
api / worker / beat-worker ─▶ Postgres, Redis, RabbitMQ (Celery), Bucket (S3 API)
live ─▶ api (private), Redis
Browser ──presigned POST/GET──▶ Bucket (CORS rule set by the one-shot bucket-cors job)
```

**Routing.** Railway gives each service one domain for one port, but Plane serves five apps from one origin and its cookies and CSRF checks assume a single origin. The `proxy` service owns the public domain and forwards each path prefix to the matching private service, mirroring Plane's own `apps/proxy/Caddyfile.ce` routing table. Caddy resolves `*.railway.internal` per connection, so it boots before its upstreams. It trusts Railway's edge for `X-Forwarded-Proto`, so Django sees HTTPS (secure cookies, CSRF).

**Files.** With `USE_MINIO=0` Plane signs URLs against the S3 endpoint: browsers upload with a presigned POST and download via a presigned GET, directly to and from the Railway Bucket (no service egress). Browser uploads need a bucket CORS rule; the `bucket-cors` job sets it once per deploy for the proxy's domain and exits.

### Deviations from upstream compose (and why)

- No `plane-minio`: Railway Bucket via `USE_MINIO=0` + presigned URLs (standard: never MinIO).
- No `migrator` container: `api` pre-deploy runs `wait_for_db` + `migrate`.
- `plane-proxy` image replaced by stock Caddy with a Railway-specific Caddyfile (upstream file uses compose hostnames like `api:8000` and handles TLS itself; Railway terminates TLS).
- API: same upstream entrypoint script, with gunicorn's bind rewritten from `0.0.0.0` to `[::]` at start (dual-stack, works on IPv6-only legacy environments). web/admin nginx get an extra `listen [::]:3000` and 2 worker processes instead of 4.
- worker: Celery `--concurrency` set from `CELERY_WORKER_CONCURRENCY` (default 2). Upstream relies on Celery's default of one process per CPU, which on Railway means one per host CPU.
- RabbitMQ: fixed `RABBITMQ_NODENAME` (data on the volume survives redeploys) and Erlang schedulers capped to 2.

## Services

All services are created together from the template; references resolve at deploy time. Values are entered exactly as shown.

### proxy

- **Source:** GitHub repo `baranberkay96/plane-railway`, root directory `/services/proxy` (Dockerfile build)
- **Start command:** image default
- **Healthcheck:** `/proxy-health` (timeout 60 s)
- **Public domain:** yes → target port `8080`
- **TCP proxy:** none
- **Volume:** none
- **Restart policy:** ON_FAILURE (max 10 retries)
- **Replicas:** 1

| Variable | Value | Description |
|---|---|---|
| `PORT` | `8080` | Port Caddy listens on (public domain target). |
| `WEB_UPSTREAM` | `${{web.RAILWAY_PRIVATE_DOMAIN}}:3000` | Main web app. |
| `SPACE_UPSTREAM` | `${{space.RAILWAY_PRIVATE_DOMAIN}}:3000` | Public spaces (/spaces). |
| `ADMIN_UPSTREAM` | `${{admin.RAILWAY_PRIVATE_DOMAIN}}:3000` | Instance admin (/god-mode). |
| `LIVE_UPSTREAM` | `${{live.RAILWAY_PRIVATE_DOMAIN}}:3000` | Live collaboration server (/live). |
| `API_UPSTREAM` | `${{api.RAILWAY_PRIVATE_DOMAIN}}:8000` | API (/api, /auth, /static). |
| `FILE_SIZE_LIMIT` | `${{api.FILE_SIZE_LIMIT}}` | Max request body size in bytes (same as api). |

### web

- **Source:** Docker image `makeplane/plane-frontend:v1.4.2`
- **Start command:** `/bin/sh -c "sed -i -e 's/^worker_processes .*/worker_processes 2;/' -e 's/listen 3000;/listen 3000; listen [::]:3000;/' /etc/nginx/nginx.conf && nginx -t && exec nginx -g 'daemon off;'"`
- **Healthcheck:** `/` (timeout 120 s)
- **Public domain:** no (private network only)
- **TCP proxy:** none
- **Volume:** none
- **Restart policy:** ON_FAILURE (max 10 retries)
- **Replicas:** 1

| Variable | Value | Description |
|---|---|---|
| `PORT` | `3000` | Port the app listens on (private network; proxy target). |

### admin

- **Source:** Docker image `makeplane/plane-admin:v1.4.2`
- **Start command:** `/bin/sh -c "sed -i -e 's/^worker_processes .*/worker_processes 2;/' -e 's/listen 3000;/listen 3000; listen [::]:3000;/' /etc/nginx/nginx.conf && nginx -t && exec nginx -g 'daemon off;'"`
- **Healthcheck:** `/god-mode/` (timeout 120 s)
- **Public domain:** no (private network only)
- **TCP proxy:** none
- **Volume:** none
- **Restart policy:** ON_FAILURE (max 10 retries)
- **Replicas:** 1

| Variable | Value | Description |
|---|---|---|
| `PORT` | `3000` | Port the app listens on (private network; proxy target). |

### space

- **Source:** Docker image `makeplane/plane-space:v1.4.2`
- **Start command:** image default
- **Healthcheck:** `/spaces/` (timeout 180 s)
- **Public domain:** no (private network only)
- **TCP proxy:** none
- **Volume:** none
- **Restart policy:** ON_FAILURE (max 10 retries)
- **Replicas:** 1

| Variable | Value | Description |
|---|---|---|
| `PORT` | `3000` | Port the app listens on (private network; proxy target). |

### live

- **Source:** Docker image `makeplane/plane-live:v1.4.2`
- **Start command:** image default
- **Healthcheck:** `/live/health` (timeout 180 s)
- **Public domain:** no (private network only)
- **TCP proxy:** none
- **Volume:** none
- **Restart policy:** ON_FAILURE (max 10 retries)
- **Replicas:** 1

| Variable | Value | Description |
|---|---|---|
| `PORT` | `3000` | Port the live server listens on (private network; proxy target). |
| `API_BASE_URL` | `http://${{api.RAILWAY_PRIVATE_DOMAIN}}:8000` | Private API URL used by the live server. |
| `REDIS_URL` | `${{Redis.REDIS_URL}}` | Redis pub/sub for collaborative editing. |
| `LIVE_SERVER_SECRET_KEY` | `${{api.LIVE_SERVER_SECRET_KEY}}` | Shared secret with api. |
| `LIVE_BASE_PATH` | `/live` | Path prefix routed by the proxy. |

### api

- **Source:** Docker image `makeplane/plane-backend:v1.4.2`
- **Start command:** `/bin/bash -c "sed 's/--bind 0.0.0.0:/--bind [::]:/' ./bin/docker-entrypoint-api.sh > /tmp/docker-entrypoint-api.sh && exec bash /tmp/docker-entrypoint-api.sh"`
- **Pre-deploy command:** `/bin/bash -c "python manage.py wait_for_db && python manage.py migrate --noinput"`
- **Healthcheck:** `/api/instances/` (timeout 600 s)
- **Public domain:** no (private network only)
- **TCP proxy:** none
- **Volume:** none
- **Restart policy:** ON_FAILURE (max 10 retries)
- **Replicas:** 1

| Variable | Value | Description |
|---|---|---|
| `PORT` | `8000` | Gunicorn port (private network; proxy target). |
| `GUNICORN_WORKERS` | `2` | Gunicorn/uvicorn worker processes for the API. |
| `DEBUG` | `0` | Django debug mode off. |
| `WEB_URL` | `https://${{proxy.RAILWAY_PUBLIC_DOMAIN}}` | Public URL of Plane (e-mail links, redirects). Update with a custom domain. |
| `CORS_ALLOWED_ORIGINS` | `https://${{proxy.RAILWAY_PUBLIC_DOMAIN}}` | Allowed browser origins; also used as CSRF trusted origins. Comma-separate extra domains. |
| `SECRET_KEY` | `${{secret(64)}}` | Django signing key (sessions, CSRF, password reset). Do not change after first boot. |
| `LIVE_SERVER_SECRET_KEY` | `${{secret(64)}}` | Shared secret between api and the live (collaboration) server. |
| `DATABASE_URL` | `${{Postgres.DATABASE_URL}}` | Postgres connection string (private network). |
| `REDIS_URL` | `${{Redis.REDIS_URL}}` | Redis for cache and sessions. |
| `AMQP_URL` | `${{RabbitMQ.AMQP_URL}}` | RabbitMQ broker for Celery jobs. |
| `USE_MINIO` | `0` | 0 = external S3 (Railway Bucket); browsers upload and download with presigned URLs. |
| `AWS_S3_ENDPOINT_URL` | `${{Bucket.ENDPOINT}}` | S3 endpoint (Railway Bucket). |
| `AWS_REGION` | `${{Bucket.REGION}}` | Bucket region (Railway Bucket). |
| `AWS_DEFAULT_REGION` | `${{Bucket.REGION}}` | Region for boto3 clients created without an explicit region. |
| `AWS_S3_BUCKET_NAME` | `${{Bucket.BUCKET}}` | Bucket name (Railway Bucket). |
| `AWS_ACCESS_KEY_ID` | `${{Bucket.ACCESS_KEY_ID}}` | Bucket access key. |
| `AWS_SECRET_ACCESS_KEY` | `${{Bucket.SECRET_ACCESS_KEY}}` | Bucket secret key. |
| `FILE_SIZE_LIMIT` | `5242880` | Max upload size in bytes (5 MB, Plane default). Also enforced by the proxy. |
| `API_KEY_RATE_LIMIT` | `60/minute` | Throttle for API-key requests. |
| `AUTHENTICATION_RATE_LIMIT` | `10/minute` | Per-IP throttle for anonymous sign-in / magic-link endpoints. |

### worker

- **Source:** Docker image `makeplane/plane-backend:v1.4.2`
- **Start command:** `/bin/bash -c "python manage.py wait_for_db && python manage.py wait_for_migrations && exec celery -A plane worker -l info --concurrency ${CELERY_WORKER_CONCURRENCY:-2}"`
- **Healthcheck:** none (no HTTP endpoint)
- **Public domain:** no (private network only)
- **TCP proxy:** none
- **Volume:** none
- **Restart policy:** ON_FAILURE (max 10 retries)
- **Replicas:** 1

| Variable | Value | Description |
|---|---|---|
| `CELERY_WORKER_CONCURRENCY` | `2` | Celery prefork processes (default would be one per host CPU, far too many on Railway). |
| `DEBUG` | `${{api.DEBUG}}` | Shared backend setting: references api.DEBUG (edit it there). |
| `WEB_URL` | `${{api.WEB_URL}}` | Shared backend setting: references api.WEB_URL (edit it there). |
| `CORS_ALLOWED_ORIGINS` | `${{api.CORS_ALLOWED_ORIGINS}}` | Shared backend setting: references api.CORS_ALLOWED_ORIGINS (edit it there). |
| `SECRET_KEY` | `${{api.SECRET_KEY}}` | Shared backend setting: references api.SECRET_KEY (edit it there). |
| `LIVE_SERVER_SECRET_KEY` | `${{api.LIVE_SERVER_SECRET_KEY}}` | Shared backend setting: references api.LIVE_SERVER_SECRET_KEY (edit it there). |
| `DATABASE_URL` | `${{api.DATABASE_URL}}` | Shared backend setting: references api.DATABASE_URL (edit it there). |
| `REDIS_URL` | `${{api.REDIS_URL}}` | Shared backend setting: references api.REDIS_URL (edit it there). |
| `AMQP_URL` | `${{api.AMQP_URL}}` | Shared backend setting: references api.AMQP_URL (edit it there). |
| `USE_MINIO` | `${{api.USE_MINIO}}` | Shared backend setting: references api.USE_MINIO (edit it there). |
| `AWS_S3_ENDPOINT_URL` | `${{api.AWS_S3_ENDPOINT_URL}}` | Shared backend setting: references api.AWS_S3_ENDPOINT_URL (edit it there). |
| `AWS_REGION` | `${{api.AWS_REGION}}` | Shared backend setting: references api.AWS_REGION (edit it there). |
| `AWS_DEFAULT_REGION` | `${{api.AWS_DEFAULT_REGION}}` | Shared backend setting: references api.AWS_DEFAULT_REGION (edit it there). |
| `AWS_S3_BUCKET_NAME` | `${{api.AWS_S3_BUCKET_NAME}}` | Shared backend setting: references api.AWS_S3_BUCKET_NAME (edit it there). |
| `AWS_ACCESS_KEY_ID` | `${{api.AWS_ACCESS_KEY_ID}}` | Shared backend setting: references api.AWS_ACCESS_KEY_ID (edit it there). |
| `AWS_SECRET_ACCESS_KEY` | `${{api.AWS_SECRET_ACCESS_KEY}}` | Shared backend setting: references api.AWS_SECRET_ACCESS_KEY (edit it there). |
| `FILE_SIZE_LIMIT` | `${{api.FILE_SIZE_LIMIT}}` | Shared backend setting: references api.FILE_SIZE_LIMIT (edit it there). |
| `API_KEY_RATE_LIMIT` | `${{api.API_KEY_RATE_LIMIT}}` | Shared backend setting: references api.API_KEY_RATE_LIMIT (edit it there). |
| `AUTHENTICATION_RATE_LIMIT` | `${{api.AUTHENTICATION_RATE_LIMIT}}` | Shared backend setting: references api.AUTHENTICATION_RATE_LIMIT (edit it there). |

### beat-worker

- **Source:** Docker image `makeplane/plane-backend:v1.4.2`
- **Start command:** `./bin/docker-entrypoint-beat.sh`
- **Healthcheck:** none (no HTTP endpoint)
- **Public domain:** no (private network only)
- **TCP proxy:** none
- **Volume:** none
- **Restart policy:** ON_FAILURE (max 10 retries)
- **Replicas:** 1

| Variable | Value | Description |
|---|---|---|
| `DEBUG` | `${{api.DEBUG}}` | Shared backend setting: references api.DEBUG (edit it there). |
| `WEB_URL` | `${{api.WEB_URL}}` | Shared backend setting: references api.WEB_URL (edit it there). |
| `CORS_ALLOWED_ORIGINS` | `${{api.CORS_ALLOWED_ORIGINS}}` | Shared backend setting: references api.CORS_ALLOWED_ORIGINS (edit it there). |
| `SECRET_KEY` | `${{api.SECRET_KEY}}` | Shared backend setting: references api.SECRET_KEY (edit it there). |
| `LIVE_SERVER_SECRET_KEY` | `${{api.LIVE_SERVER_SECRET_KEY}}` | Shared backend setting: references api.LIVE_SERVER_SECRET_KEY (edit it there). |
| `DATABASE_URL` | `${{api.DATABASE_URL}}` | Shared backend setting: references api.DATABASE_URL (edit it there). |
| `REDIS_URL` | `${{api.REDIS_URL}}` | Shared backend setting: references api.REDIS_URL (edit it there). |
| `AMQP_URL` | `${{api.AMQP_URL}}` | Shared backend setting: references api.AMQP_URL (edit it there). |
| `USE_MINIO` | `${{api.USE_MINIO}}` | Shared backend setting: references api.USE_MINIO (edit it there). |
| `AWS_S3_ENDPOINT_URL` | `${{api.AWS_S3_ENDPOINT_URL}}` | Shared backend setting: references api.AWS_S3_ENDPOINT_URL (edit it there). |
| `AWS_REGION` | `${{api.AWS_REGION}}` | Shared backend setting: references api.AWS_REGION (edit it there). |
| `AWS_DEFAULT_REGION` | `${{api.AWS_DEFAULT_REGION}}` | Shared backend setting: references api.AWS_DEFAULT_REGION (edit it there). |
| `AWS_S3_BUCKET_NAME` | `${{api.AWS_S3_BUCKET_NAME}}` | Shared backend setting: references api.AWS_S3_BUCKET_NAME (edit it there). |
| `AWS_ACCESS_KEY_ID` | `${{api.AWS_ACCESS_KEY_ID}}` | Shared backend setting: references api.AWS_ACCESS_KEY_ID (edit it there). |
| `AWS_SECRET_ACCESS_KEY` | `${{api.AWS_SECRET_ACCESS_KEY}}` | Shared backend setting: references api.AWS_SECRET_ACCESS_KEY (edit it there). |
| `FILE_SIZE_LIMIT` | `${{api.FILE_SIZE_LIMIT}}` | Shared backend setting: references api.FILE_SIZE_LIMIT (edit it there). |
| `API_KEY_RATE_LIMIT` | `${{api.API_KEY_RATE_LIMIT}}` | Shared backend setting: references api.API_KEY_RATE_LIMIT (edit it there). |
| `AUTHENTICATION_RATE_LIMIT` | `${{api.AUTHENTICATION_RATE_LIMIT}}` | Shared backend setting: references api.AUTHENTICATION_RATE_LIMIT (edit it there). |

### Postgres

- **Source:** Docker image `ghcr.io/railwayapp-templates/postgres-ssl:18`
- **Start command:** image default
- **Healthcheck:** none (no HTTP endpoint)
- **Public domain:** no (private network only)
- **TCP proxy:** none
- **Volume:** `/var/lib/postgresql/data`
- **Restart policy:** ON_FAILURE (max 10 retries)
- **Replicas:** 1

| Variable | Value | Description |
|---|---|---|
| `PGDATA` | `/var/lib/postgresql/data/pgdata` | Location where the database will be initialized. |
| `PGHOST` | `${{RAILWAY_PRIVATE_DOMAIN}}` | Railway Private Domain Name. |
| `PGPORT` | `5432` | Port to connect to Postgres. |
| `PGUSER` | `${{ POSTGRES_USER }}` | Required variable for Data panel. |
| `PGDATABASE` | `${{POSTGRES_DB}}` | Required variable for the data panel. |
| `PGPASSWORD` | `${{POSTGRES_PASSWORD}}` | Required variable for Data panel. |
| `POSTGRES_DB` | `railway` | Default database created when image is started. |
| `DATABASE_URL` | `postgresql://${{PGUSER}}:${{POSTGRES_PASSWORD}}@${{RAILWAY_PRIVATE_DOMAIN}}:5432/${{PGDATABASE}}` | URL to connect to Postgres database. |
| `POSTGRES_USER` | `postgres` | User to connect to Postgres DB. |
| `SSL_CERT_DAYS` | `820` | SSL certificate expiry in days. |
| `POSTGRES_PASSWORD` | `${{ secret(32, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ") }}` | Password to connect to DB. |
| `RAILWAY_DEPLOYMENT_DRAINING_SECONDS` | `60` | Allow Postgres to cleanly shut down. |

### Redis

- **Source:** Docker image `redis:8.2`
- **Start command:** `/bin/sh -c "rm -rf $RAILWAY_VOLUME_MOUNT_PATH/lost+found/ && exec docker-entrypoint.sh redis-server --requirepass $REDIS_PASSWORD --save 60 1 --dir $RAILWAY_VOLUME_MOUNT_PATH"`
- **Healthcheck:** none (no HTTP endpoint)
- **Public domain:** no (private network only)
- **TCP proxy:** none
- **Volume:** `/data`
- **Restart policy:** ON_FAILURE (max 10 retries)
- **Replicas:** 1

| Variable | Value | Description |
|---|---|---|
| `REDISHOST` | `${{RAILWAY_PRIVATE_DOMAIN}}` | Private network hostname of the Redis service, only resolvable from services in the same environment. |
| `REDISPORT` | `6379` | Port that Redis listens on. |
| `REDISUSER` | `default` | Username for authenticating with Redis. |
| `REDIS_URL` | `redis://${{ REDISUSER }}:${{ REDIS_PASSWORD }}@${{ REDISHOST }}:${{ REDISPORT }}` | Connection string for connecting to Redis using the private network. |
| `REDISPASSWORD` | `${{REDIS_PASSWORD}}` | Alias of REDIS_PASSWORD for clients that expect the unseparated name. |
| `REDIS_PASSWORD` | `${{ secret(32, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ") }}` | Randomly generated password for authenticating with Redis. |

### RabbitMQ

- **Source:** Docker image `rabbitmq:3.13.7-management-alpine`
- **Start command:** image default
- **Healthcheck:** none (no HTTP endpoint)
- **Public domain:** no (private network only)
- **TCP proxy:** none
- **Volume:** `/var/lib/rabbitmq`
- **Restart policy:** ON_FAILURE (max 10 retries)
- **Replicas:** 1

| Variable | Value | Description |
|---|---|---|
| `RABBITMQ_DEFAULT_USER` | `plane` | Broker user created on first boot. |
| `RABBITMQ_DEFAULT_PASS` | `${{secret(32)}}` | Broker password (generated). |
| `RABBITMQ_DEFAULT_VHOST` | `plane` | Broker virtual host. |
| `RABBITMQ_NODENAME` | `rabbit@localhost` | Fixed node name so the data on the volume survives redeploys (container hostnames change). |
| `RABBITMQ_SERVER_ADDITIONAL_ERL_ARGS` | `+S 2:2 +SDio 2` | Limit Erlang schedulers to 2 (the VM otherwise sizes itself to the host's CPU count). |
| `AMQP_URL` | `amqp://${{RABBITMQ_DEFAULT_USER}}:${{RABBITMQ_DEFAULT_PASS}}@${{RAILWAY_PRIVATE_DOMAIN}}:5672/${{RABBITMQ_DEFAULT_VHOST}}` | Connection string for Celery (private network). |

### bucket-cors

- **Source:** Docker image `amazon/aws-cli:2.37.12`
- **Start command:** `/bin/sh -c "aws s3api put-bucket-cors --bucket $BUCKET_NAME --endpoint-url $S3_ENDPOINT --region $S3_REGION --cors-configuration \"$CORS_CONFIGURATION\" && echo 'Bucket CORS configured for' $PUBLIC_ORIGIN"`
- **Healthcheck:** none (no HTTP endpoint)
- **Public domain:** no (private network only)
- **TCP proxy:** none
- **Volume:** none
- **Restart policy:** NEVER
- **Replicas:** 1

| Variable | Value | Description |
|---|---|---|
| `AWS_ACCESS_KEY_ID` | `${{Bucket.ACCESS_KEY_ID}}` | Bucket access key. |
| `AWS_SECRET_ACCESS_KEY` | `${{Bucket.SECRET_ACCESS_KEY}}` | Bucket secret key. |
| `BUCKET_NAME` | `${{Bucket.BUCKET}}` | Bucket to configure. |
| `S3_ENDPOINT` | `${{Bucket.ENDPOINT}}` | Bucket S3 endpoint. |
| `S3_REGION` | `${{Bucket.REGION}}` | Bucket region. |
| `PUBLIC_ORIGIN` | `https://${{proxy.RAILWAY_PUBLIC_DOMAIN}}` | Plane's public origin (for the log line). |
| `CORS_CONFIGURATION` | `{"CORSRules":[{"AllowedOrigins":["https://${{proxy.RAILWAY_PUBLIC_DOMAIN}}"],"AllowedMethods":["GET","HEAD","PUT","POST"],"AllowedHeaders":["*"],"ExposeHeaders":["ETag"],"MaxAgeSeconds":3600}]}` | CORS rule letting browsers on Plane's domain upload to / read from the bucket with presigned URLs. Add custom domains to AllowedOrigins and redeploy this service. |

### Bucket (Railway Bucket)

Create a Bucket named `Bucket` (any region). Services reference `BUCKET`, `ENDPOINT`, `REGION`, `ACCESS_KEY_ID` and `SECRET_ACCESS_KEY` from it.

## Improvements over market templates

| # | Change | Why |
|---|---|---|
| 1 | Railway Bucket + presigned URLs instead of MinIO; one-shot `bucket-cors` job | MinIO community images are discontinued; Bucket egress is free and files leave the proxy path. |
| 2 | All Plane images pinned to `v1.4.2` (base: `v1.4.0`; AIO: `stable`) | Current stable release, reproducible deploys. |
| 3 | Router as a reviewable `Caddyfile` in `services/proxy` with upstreams from variables (base: Caddyfile pasted into an env var) | Easier to audit and change; build-time `caddy validate`; `/proxy-health` endpoint. |
| 4 | Dual-stack listeners for api (gunicorn `[::]`) and web/admin (nginx `[::]:3000`) | Works in IPv6-only (legacy) Railway environments as well as new ones. |
| 5 | Celery concurrency capped (default 2) | Avoids one worker process per host CPU (memory and DB connections). |
| 6 | Healthchecks on every HTTP service: `/api/instances/`, `/`, `/god-mode/`, `/spaces/`, `/live/health`, `/proxy-health` | Railway only routes to healthy deployments; broken releases stop at deploy time. |
| 7 | `AMQP_URL` defined once on RabbitMQ from its private domain (base: hardcoded `rabbitmq.railway.internal`) | Survives service renames. |
| 8 | Backend settings defined once on `api`, referenced by worker and beat-worker | No drift between Django processes. |
| 9 | RabbitMQ latest 3.13 patch, fixed node name, scheduler cap (kept from base) | Data on the volume survives redeploys; bounded CPU/RAM. |

## First login

1. Open `https://<proxy domain>/god-mode/` and create the **instance admin** (first visitor only). Configure SMTP, Google/GitHub/GitLab OAuth, AI and image settings there; Plane stores them in the database.
2. Open `https://<proxy domain>/` and create the first workspace.
3. Optional: disable open sign-ups in god-mode → Authentication.

## Custom domain

Add the domain to **proxy**, then update `WEB_URL` and `CORS_ALLOWED_ORIGINS` on **api** (comma-separate several origins), add the origin to `CORS_CONFIGURATION` on **bucket-cors**, and redeploy api, worker, beat-worker and bucket-cors.

## Scaling

- **worker:** raise `CELERY_WORKER_CONCURRENCY` or add replicas (stateless).
- **api:** raise `GUNICORN_WORKERS` or add replicas (stateless; migrations run once in pre-deploy).
- **web / admin / space / live / proxy:** stateless; add replicas if needed. Keep **beat-worker at 1 replica** (scheduler).
- Postgres, Redis, RabbitMQ: vertical scaling.

## Resource estimate

Railway prices: RAM $10/GB-month, vCPU $20/vCPU-month, volume $0.15/GB-month, Bucket $0.015/GB-month.

| Service | Idle RAM | Typical RAM (10–30 users) | Typical CPU |
|---|---|---|---|
| proxy | 30 MB | 40 MB | 0.02 |
| web | 10 MB | 20 MB | <0.01 |
| admin | 10 MB | 20 MB | <0.01 |
| space | 120 MB | 160 MB | 0.02 |
| live | 110 MB | 180 MB | 0.03 |
| api | 350 MB | 600 MB | 0.20 |
| worker | 300 MB | 500 MB | 0.15 |
| beat-worker | 150 MB | 160 MB | 0.01 |
| Postgres | 120 MB | 350 MB | 0.08 |
| Redis | 20 MB | 50 MB | 0.01 |
| RabbitMQ | 130 MB | 160 MB | 0.03 |
| bucket-cors | 0 (exits) | 0 | 0 |
| **Total** | **≈1.35 GB** (≈0.08 vCPU) | **≈2.25 GB** | **≈0.55 vCPU** |

- **Idle:** ≈ $13.5 RAM + $1.6 CPU + $0.3 volumes ≈ **$15/month**.
- **Typical:** ≈ $22.5 RAM + $11 CPU + $0.5 volumes + Bucket ≈ **$34/month**.
- **Minimum plan:** Hobby (12 services; Trial credit does not cover a month).

## Verification checklist

- [ ] **bucket-cors** deployment logs `Bucket CORS configured for https://…` and exits 0.
- [ ] **api** pre-deploy log ends with migrations applied; api turns healthy on `/api/instances/`.
- [ ] `https://<proxy domain>/proxy-health` → `OK`; `/` loads the web app; `/god-mode/` loads the admin; create the instance admin.
- [ ] Create a workspace, project and work item; attach an image to the work item (browser upload to the Bucket must succeed; check the object in Railway → Bucket → Files) and reload the page (presigned GET).
- [ ] Open a Page with two browsers and type in both → edits appear live (`/live` WebSocket).
- [ ] **worker** logs show Celery tasks (e.g. issue activity, notifications); **beat-worker** logs `beat: Starting...` and scheduled sends.
- [ ] Redeploy api and RabbitMQ: data intact, worker reconnects.

## Open questions

- **Bucket CORS through the S3 API:** `put-bucket-cors` is what Railway documents for browser uploads; not executed against a live Bucket here. If it fails, uploads from the browser are blocked (CORS) while everything else works.
- **boto3 addressing style:** Plane's boto3 clients use the default addressing (virtual-hosted for a custom endpoint on recent botocore), which matches Railway's documented style; verify with the attachment test.
- **RabbitMQ 3.13:** Plane v1.4.2 pins the 3.13 line, but docker-library stopped publishing 3.13 builds (last 2025-12-02). Moving to RabbitMQ 4.x should be tested with Celery's transient queues before switching.
- **Railway edge IP range:** the proxy trusts private ranges and `100.0.0.0/8` for `X-Forwarded-*`. If Railway's edge uses other source addresses, Django would see `http`; CSRF still passes (trusted origins), but secure-cookie behaviour should be checked.
- **Local test scope:** the Caddy routing table was validated and smoke-tested locally against dummy upstreams; the Plane containers themselves were not run locally.
