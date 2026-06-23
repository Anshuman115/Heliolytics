# Heliolytics — Go API Architecture

## Platform (3 repos)

| Repo | Role |
|------|------|
| **Heliolytics** (this repo) | Go ingest API, PostgreSQL, Docker deploy |
| **Heliolytics_App** | Flutter mobile — BLE sync, upload raw sessions |
| **Heliolytics_Web** | Next.js dashboard — read metrics from API |

Clone all three as sibling folders under the same parent directory so `deploy/docker-compose.yml` can build the web service from `../../Heliolytics_Web`.

## Data flow

```
Helio Strap ──BLE──► Flutter (raw .bin + JSON on device)
                         │ POST /api/v1/ingest (multipart, HMAC token)
                         ▼
                    Go API (parse raw → metrics)
                         │
                         ▼
                    PostgreSQL / TimescaleDB
                    ╱         ╲
              Next.js web    Flutter (GET metrics only)
```

## Auth

- **Strap:** 32-char hex key, ECDH + AES (Flutter secure storage, BLE only)
- **API (app + server):** `HELIOLYTICS_SIGNING_SECRET` signs short-lived HMAC tokens (`X-Heliolytics-Token`, 5-minute window, nonce replay rejected)
- **Web dashboard:** password sign-in (`HELIOLYTICS_WEB_PASSWORD`); signing secret stays server-side in Docker
- **Reparse:** disabled by default; enable with `REPARSE_ENABLED=true` + optional `REPARSE_SECRET` header

## TLS

Cloudflare Tunnel (`cloudflared` in `deploy/docker-compose.yml`) terminates HTTPS at the edge. API and web stay HTTP on the Docker network; `TRUST_PROXY=true` on the API trusts Cloudflare proxy headers.

## Layout

```
cmd/server/          Entry point
internal/api/        HTTP handlers
internal/parse/      BLE blob parsers
internal/store/      PostgreSQL (sqlc queries)
schema.sql           Full Postgres schema (fresh DB init)
deploy/              docker-compose + cloudflared + install.sh
```

## Deploy

```bash
cd deploy
cp .env.example .env
./install.sh
```
