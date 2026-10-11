# NOTICE — Plane v1 Project Hub

## Software deployed (not redistributed)

| Component | Source | License |
|---|---|---|
| Plane (backend, frontend, admin, space, live) | Docker images `makeplane/plane-*:v1.4.2` from https://github.com/makeplane/plane | AGPL-3.0 |
| Caddy | `caddy:2.11.7-alpine` (base of `services/proxy`) | Apache-2.0 |
| RabbitMQ | `rabbitmq:3.13.7-management-alpine` | MPL-2.0 |
| AWS CLI | `amazon/aws-cli:2.37.12` | Apache-2.0 |
| PostgreSQL | `ghcr.io/railwayapp-templates/postgres-ssl:18` (Railway base) | PostgreSQL License |
| Redis | `redis:8.2` (Railway base) | Redis 8 tri-license (RSALv2 / SSPLv1 / AGPLv3) |

## Upstream files checked (tag `v1.4.2`, 2026-10-10)

- `deployments/cli/community/docker-compose.yml` and `variables.env` — services, commands, environment.
- `apps/proxy/Caddyfile.ce` and `Dockerfile.ce` — routing table and Caddy version.
- `apps/api/bin/docker-entrypoint-{api,worker,beat,migrator}.sh` — start sequences (the api script is used unchanged except for the bind address, rewritten at start).
- `apps/api/plane/settings/common.py`, `storage.py`, `db/management/commands/create_bucket.py`, `bgtasks/export_task.py` — S3 behaviour with `USE_MINIO=0`.
- `apps/{web,admin,space}/nginx/nginx.conf`, `apps/*/Dockerfile.*`, `apps/live/src/{env,server}.ts`, `controllers/health.controller.ts` — ports, listen addresses, health endpoints.

## Original files in this package

- `services/proxy/Caddyfile` and `Dockerfile` — written for this template. The path-to-service mapping follows Plane's `apps/proxy/Caddyfile.ce` (AGPL-3.0) so that Plane behaves as in its own deployment; the file itself is new (Railway upstream variables, trusted proxies, health endpoint, no TLS).

## Reused ideas

- **`_upstream/railwayapp-templates__plane-caddy-proxy`** — MIT License, Copyright (c) 2024 Brody. Idea reused: trusting private ranges and `100.0.0.0/8` as Railway's proxy addresses. No file copied.
- **Market template `plane-jira-alternative`** (no license): structural ideas only — migrations in the api pre-deploy command, fixed RabbitMQ node name, capped Erlang schedulers. No text copied; all descriptions are original.
- **Railway base definitions** `_railway-base/postgres.json`, `redis.json`: copied as instructed by STANDARD.md.
- `_upstream/railwayapp-templates__minio-console`: not used (MinIO is excluded).
