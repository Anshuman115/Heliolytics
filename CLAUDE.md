# CLAUDE.md — Heliolytics (Go server)

The Go backend for Heliolytics. Public repo. Current line: `v6`.

## Project Identity

- **Service:** Heliolytics API · Go 1.22 · pgx/v5 · PostgreSQL + TimescaleDB
- **Deployment:** VPS + Cloudflare Tunnel (prod), Docker Compose (dev)
- **Single-owner system.** No user accounts, no `user_id`, no multi-tenancy. One
  shared signing secret gates the API

Part of a 3-repo system, cloned as **siblings** under one parent (compose builds the
web image from `../../Heliolytics_Web`):

| Repo | Role |
|------|------|
| `Heliolytics` (this) | Go API — parses, stores, serves. **The hub** |
| `Heliolytics_App` | Flutter — BLE sync, uploads raw session bytes |
| `Heliolytics_Web` | Next.js dashboard — reads metrics |

**The core split: all health parsing happens here.** The phone uploads raw strap
bytes and never parses or persists health data. That's why raw blobs are stored
forever and can be re-parsed.

## Docs — read before changing a feature

| Doc | When |
|-----|------|
| [docs/features/ingest-and-parsing.md](docs/features/ingest-and-parsing.md) | Ingest, parse pipeline, coverage watermarks |
| [docs/features/metrics-api.md](docs/features/metrics-api.md) | Routes, middleware chain, store layer |
| [docs/features/auth-and-security.md](docs/features/auth-and-security.md) | HMAC token, replay defense, rate limiting, env |
| [docs/features/readiness-score.md](docs/features/readiness-score.md) | Recovery score model and weights |
| [docs/features/storage-and-schema.md](docs/features/storage-and-schema.md) | Tables, hypertables, migrations |
| [docs/features/deploy.md](docs/features/deploy.md) | Compose stack, tunnel, secrets |

`docs/features/` is published. `docs/reference/` — the cross-repo wire contract,
per-repo architecture manuals, glossary, and debug cookbook — is **local-only and
gitignored**. It's the deepest material in the system; consult it locally, but never
cite it from a published file.

## Actual Folder Structure

```
Heliolytics/
├── cmd/server/main.go        # Entry point — wire deps, start
├── internal/
│   ├── api/                  # HTTP handlers (flat) + routes.go
│   ├── auth/                 # Token sign/verify, nonce store
│   ├── config/               # Env config
│   ├── middleware/           # hmac, logging, ratelimit
│   ├── parse/                # Blob → records → day rollups
│   ├── readiness/            # Recovery score (pure, no I/O)
│   └── store/                # pgx data layer
│       └── db/               # sqlc GENERATED — never hand-edit
├── migrations/               # Numbered incremental changes
├── sql/queries/              # Hand-written SQL, sqlc input
├── schema.sql                # Full from-scratch schema
├── scripts/                  # Local helpers (gitignored)
└── deploy/                   # compose, Dockerfile, deploy scripts
```

## The Single Most Important Rule

**One responsibility per file. No file should exceed ~150 lines.**

Handlers handle HTTP. Parsers parse bytes. `readiness` computes a score and touches
no I/O. If a file does more than one, split it.

## Naming

- Packages: short, lowercase, no underscores — `api`, `store`, `parse`, `auth`
- Files: `snake_case.go`
- Types/interfaces: `PascalCase`
- Functions/methods: `camelCase` (private), `PascalCase` (exported)
- Constants: `PascalCase` exported, `camelCase` package-private
- Error variables: `Err` prefix — `ErrNotFound`, `ErrUnauthorized`

## Code Style

- **Standard library first.** Add a dependency only if the stdlib can't do it cleanly
- **Errors are values.** Return `(T, error)`. Never ignore errors
- **No global state.** Inject dependencies via constructors
- **Short functions.** If it doesn't fit on a screen, it's too long
- **Table-driven tests.** `t.Run` subtests
- Run `gofmt`, `go vet`, `go test ./...` before every commit

## API Design

- REST + JSON, versioned `/api/v1/...`
- **Responses are plain JSON — there is no `{data, error}` envelope.** Errors are
  HTTP status + plain text
- Auth: `X-Heliolytics-Token` only. **No Firebase, no JWT, no user identity**
- `GET /health` is deliberately **outside** the auth chain — healthchecks need it

## Security (non-negotiable)

- Every data endpoint requires a valid signed token. `/health` is the one exception
- HMAC-SHA256 over `"ts:nonce"`; replay defense = 5-min window + nonce store
- Constant-time comparison (`crypto/subtle`) — never `==` on a signature
- No secrets in code. Environment only
- Parameterized queries only. No string concatenation into SQL
- Never log raw sensor data or tokens
- `TRUST_PROXY=true` is only safe because Cloudflare is genuinely in front. Off, and
  every request rate-limits as one client; on without a trusted proxy, anyone spoofs
  their IP via a header
- `/api/v1/reparse` is off by default and needs its own `REPARSE_SECRET` — it
  rewrites history

## Database

- **PostgreSQL + TimescaleDB.** `schema.sql` is the from-scratch definition;
  `migrations/` carries an existing DB forward. **Both must end at the same shape**
- The 4 per-sample tables are hypertables on `sampled_at`. Rollup tables are plain
- Raw strap bytes live in `raw_type_blobs` — kept forever, they make reparse possible
- Most queries via **sqlc**: edit `sql/queries/*.sql`, regenerate. **Never hand-edit
  `internal/store/db/`**. Some store code (`coverage.go`, `readiness.go`) is
  deliberately hand-written pgx where codegen doesn't fit
- Writes upsert — re-uploading a sync session must stay idempotent
- `deploy/reset-db.sh` and `docker compose down -v` **destroy all data**

## Cross-repo invariants — break these and another repo breaks

- **`X-Heliolytics-Token` format** (`ts.nonce.sig`) must stay byte-identical across
  `internal/auth/signing.go`, app `lib/services/network/heliolytics_token.dart`, and
  web `lib/api/signing.ts`
- **This server owns sync state.** `/metrics/coverage` is the sole authority on what
  is ingested — the phone keeps no bookmarks, so a reinstall must resume correctly
- Coverage must measure sessions by **end** time, not start, or the phone re-fetches
- Changing readiness weights silently rewrites the meaning of every historical score

## Testing

- Unit: parsers, readiness, domain logic — no DB, no HTTP
- Integration: `httptest` for handlers; real queries against a test schema
- Dump-replay tests (`dump_integration_test.go`, `pipeline_e2e_test.go`) run real
  captured sessions through the pipeline — the strongest signal we have

## Git

- Commits: `type(scope): message` — e.g., `feat(api): add sync endpoint`
- Types: `feat`, `fix`, `chore`, `refactor`, `test`, `docs`
- One logical change per commit. No WIP commits on main.

## Deployment

- Compose stack: `db` (timescaledb pg16) · `api` · `web` · `cloudflared`
- Compose lives in **`deploy/`**, not the repo root
- HTTPS terminates at Cloudflare; `cloudflared` dials out. **No Caddy, no open
  80/443.** `api`/`web` bind to `127.0.0.1` only
- Health-gated startup: `api` waits on `db`, `web` waits on `api`
- Config via env only. `deploy/.env.example` documents every required var; `install.sh`
  copies it to `.env` on first run
- `deploy/deploy-web.sh` rebuilds only web, no API downtime

## Published vs. local-only

This repo is public. `CLAUDE.md` and `docs/features/` **are published** — write them
for an outside reader, not just for yourself.

Also tracked and public: `README.md`, `ARCHITECTURE.md`, `SCHEMA_DESIGN.md`,
`deploy/deployment.md`.

Local-only (in `.gitignore`, on disk for personal reference only):

| Path | Purpose |
|------|---------|
| `docs/reference/` | Cross-repo contract, architecture manuals, glossary, debug cookbook |
| `docs/README.md` | Local index |
| `TODO.md`, `PLAN.md` | Status and proposals |
| `deploy/DEV.md` | Local dev loop |
| `scripts/` | Local helpers |
| `_learnspace/`, `lessons/` | Learning artifacts |

**Never cite a local-only path from a published file.** A published doc that links to
`docs/reference/` is a broken link for everyone who clones the repo — describe the
thing in prose instead.

## Attribution and legal hygiene

- No third-party project names in code comments, commit messages, or tracked markdown
- No "ported from", "direct port", or file-path attribution to other repos
- Describe behavior in terms of Huami BLE / Gadgetbridge-compatible protocol only
- Hardware/protocol names (Huami, ZeppOS) are fine where they name the actual device
  or wire format. **Competitor product names are not**
