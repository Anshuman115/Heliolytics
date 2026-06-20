# Heliolytics Schema v2 — Design

> **Status:** Design only — not applied to `schema.sql` yet.  
> **Goal:** Production-grade dedup. Same sleep/workout/sample must not multiply when the phone uploads a **new** `session_id` with overlapping data.

---

## 1. What we have today (v1)

Eight tables in `schema.sql`:

| Layer | Table | Primary identity today |
|-------|--------|----------------------|
| Ingest audit | `sync_sessions` | `session_id` (phone: `DateTime.now().microsecondsSinceEpoch`) |
| Ingest audit | `raw_type_blobs` | `(session_id, type_code)` |
| Canonical rollup | `daily_metrics` | `day_key` ✅ merges correctly |
| Event | `sleep_sessions` | `(sync_session_id, started_at)` ❌ |
| Event | `workouts` | `(sync_session_id, started_at)` ❌ |
| Event | `activity_sessions` | `(sync_session_id, started_at)` ❌ |
| Series | `health_samples` | `(sampled_at, id)` — **no business unique** ❌ |
| Series | `temperature_samples` | `(sampled_at, id)` — **no business unique** ❌ |

### Why duplicates happen

```
Sync 1 → session_id = 100 → sleep @ 2025-06-09T23:00 → row (100, 23:00)
Sync 2 → session_id = 200 → same sleep in blob     → row (200, 23:00)  ← duplicate
```

- **Re-upload same session:** `Replace*` deletes rows for that `session_id` first → idempotent ✅  
- **New session, same events:** different `session_id` → new rows ❌  
- **`daily_metrics`:** `ON CONFLICT (day_key)` + COALESCE/GREATEST → merges ✅  
- **`health_samples`:** insert-only after session delete → duplicates across sessions ❌  

Coverage API uses `MAX(...)` over canonical tables — duplicates do not break coverage math much, but **UI lists inflate** (double workouts, double chart points).

---

## 2. Design principles (v2)

1. **Two layers, two jobs**
   - **Ingest layer** — prove what arrived, enable reparse: `sync_sessions`, `raw_type_blobs` (unchanged role).
   - **Canonical layer** — what the app/web reads: everything else.

2. **Natural keys on real-world identity**
   - Keys come from **event time + type**, not from upload batch id.
   - `sync_session_id` becomes **provenance** (`source_session_id`), not part of uniqueness.

3. **Upsert, not session-scoped delete**
   - Stop `DELETE FROM sleep WHERE sync_session_id = $1` before insert.
   - Use `INSERT … ON CONFLICT (natural_key) DO UPDATE` with explicit merge rules.

4. **Single-user scope (for now)**
   - One strap / one API key → no `user_id` column yet.
   - Optional `device_mac` on canonical rows for future multi-device; not in unique keys until needed.

5. **Reparse-safe**
   - `/api/v1/reparse` reads `raw_type_blobs`, re-runs parse → aggregate → upsert canonical. Natural keys make reparse idempotent.

6. **Timescale kept for series**
   - `health_samples` and `temperature_samples` stay hypertables on `sampled_at`.

---

## 3. Natural keys (canonical layer)

| Table | Natural key | Rationale |
|-------|-------------|-----------|
| `daily_metrics` | `day_key` | Calendar day in IST (unchanged) |
| `sleep_sessions` | `(started_at, is_nap)` | One main sleep + naps per start instant |
| `workouts` | `started_at` | Strap workout start is unique |
| `activity_sessions` | `started_at` | Auto-detected activity start |
| `health_samples` | `(metric, sampled_at)` | One value per metric per timestamp |
| `temperature_samples` | `sampled_at` | One skin temp reading per instant |

**Tie-break:** if parser fixes change metrics for same key, `DO UPDATE` overwrites with latest ingest (`source_session_id`, `updated_at`).

**Edge case — two workouts same second:** rare; defer. If seen in dumps, extend to `(started_at, sport_type, duration_sec)`.

---

## 4. Proposed schema (v2 SQL sketch)

```sql
-- ========== INGEST (audit) — unchanged purpose ==========

CREATE TABLE sync_sessions (
  session_id    TEXT PRIMARY KEY,
  device_mac    TEXT,
  started_at    TIMESTAMPTZ NOT NULL,
  ended_at      TIMESTAMPTZ,
  battery_pct   INT,
  catalog_json  JSONB,
  ingested_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE raw_type_blobs (
  session_id  TEXT NOT NULL REFERENCES sync_sessions(session_id) ON DELETE CASCADE,
  type_code   TEXT NOT NULL,
  byte_len    INT NOT NULL,
  payload     BYTEA NOT NULL,
  PRIMARY KEY (session_id, type_code)
);

-- ========== CANONICAL — natural keys ==========

CREATE TABLE daily_metrics (
  day_key                  TEXT PRIMARY KEY,
  steps                    INT NOT NULL DEFAULT 0,
  pai_score                INT,
  readiness                INT,
  spo2_avg                 INT,
  hrv_rmssd                INT,
  resting_hr               INT,
  max_hr                   INT,
  resp_rate_avg            INT,
  stress_avg               INT,
  sleep_score              INT,
  sleep_mins               INT,
  sleep_deep_mins          INT,
  sleep_rem_mins           INT,
  sleep_light_mins         INT,
  temp_avg_c               NUMERIC(4, 1),
  nap_count                INT NOT NULL DEFAULT 0,
  workout_count            INT NOT NULL DEFAULT 0,
  activity_session_count   INT NOT NULL DEFAULT 0,
  source_session_id        TEXT,          -- last ingest that touched this row
  updated_at               TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE sleep_sessions (
  id                  BIGSERIAL PRIMARY KEY,
  started_at          TIMESTAMPTZ NOT NULL,
  is_nap              BOOLEAN NOT NULL DEFAULT false,
  day_key             TEXT NOT NULL,
  score               INT NOT NULL DEFAULT 0,
  total_mins          INT NOT NULL DEFAULT 0,
  deep_mins           INT NOT NULL DEFAULT 0,
  rem_mins            INT NOT NULL DEFAULT 0,
  light_mins          INT NOT NULL DEFAULT 0,
  wake_mins           INT NOT NULL DEFAULT 0,
  stages_json         JSONB,
  source_session_id   TEXT NOT NULL,
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (started_at, is_nap)
);

CREATE TABLE workouts (
  id                  BIGSERIAL PRIMARY KEY,
  started_at          TIMESTAMPTZ NOT NULL UNIQUE,
  day_key             TEXT NOT NULL,
  sport_type          INT NOT NULL DEFAULT 0,
  sport_name          TEXT,
  duration_sec        INT NOT NULL DEFAULT 0,
  calories            INT,
  avg_hr              INT,
  max_hr              INT,
  source_session_id   TEXT NOT NULL,
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE activity_sessions (
  id                  BIGSERIAL PRIMARY KEY,
  started_at          TIMESTAMPTZ NOT NULL UNIQUE,
  day_key             TEXT NOT NULL,
  sport_type          INT NOT NULL DEFAULT 0,
  sport_name          TEXT,
  duration_sec        INT NOT NULL DEFAULT 0,
  calories            INT,
  avg_hr              INT,
  max_hr              INT,
  source_session_id   TEXT NOT NULL,
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE temperature_samples (
  sampled_at          TIMESTAMPTZ NOT NULL,
  day_key             TEXT NOT NULL,
  celsius             NUMERIC(4, 1) NOT NULL,
  source_session_id   TEXT NOT NULL,
  PRIMARY KEY (sampled_at)
);

CREATE TABLE health_samples (
  metric              TEXT NOT NULL,
  sampled_at          TIMESTAMPTZ NOT NULL,
  day_key             TEXT NOT NULL,
  value               NUMERIC(6, 2) NOT NULL,
  source_session_id   TEXT NOT NULL,
  PRIMARY KEY (metric, sampled_at)
);

-- Indexes (unchanged intent)
CREATE INDEX idx_daily_metrics_updated ON daily_metrics(updated_at DESC);
CREATE INDEX idx_sleep_day ON sleep_sessions(day_key, started_at DESC);
CREATE INDEX idx_workouts_day ON workouts(day_key, started_at DESC);
CREATE INDEX idx_activity_day ON activity_sessions(day_key, started_at DESC);
CREATE INDEX idx_temp_day ON temperature_samples(day_key, sampled_at);
CREATE INDEX idx_health_day_metric ON health_samples(day_key, metric, sampled_at);

SELECT create_hypertable('temperature_samples', 'sampled_at', if_not_exists => TRUE);
SELECT create_hypertable('health_samples', 'sampled_at', if_not_exists => TRUE);
```

### What changed vs v1

| Change | Why |
|--------|-----|
| Drop `sync_session_id` from UNIQUE constraints | Stop tying identity to upload batch |
| Add `source_session_id` + `updated_at` | Audit trail + debugging |
| `health_samples` PK = `(metric, sampled_at)` | Hard dedup for series |
| `temperature_samples` PK = `sampled_at` | Hard dedup |
| Remove `BIGSERIAL` surrogate from series PKs | Simpler upsert; hypertable still works |

---

## 5. Write semantics (store layer)

Replace current pattern:

```
Delete*BySession(sid) → Insert rows tagged sid
```

With:

```
Upsert*Canonical(rows, source_session_id=sid)
```

| Operation | v1 | v2 |
|-----------|----|----|
| Ingest session + blobs | insert | same |
| Sleep / workouts / activity | delete by session, insert | upsert on natural key |
| Health / temp series | delete by session, insert | upsert on natural key |
| Daily rollup | upsert by day_key | same (+ set source_session_id) |

### Merge rules (ON CONFLICT)

| Table | UPDATE policy |
|-------|----------------|
| `daily_metrics` | Keep v1 COALESCE/GREATEST logic |
| `sleep_sessions` | Replace all fields from EXCLUDED (latest parse wins) |
| `workouts` / `activity_sessions` | Replace all fields from EXCLUDED |
| `health_samples` | `value = EXCLUDED.value` |
| `temperature_samples` | `celsius = EXCLUDED.celsius` |

**Optional later:** reject update if existing row is newer (`updated_at` guard).

---

## 6. Read API impact

**Minimal.** List queries already filter by `day_key` range — no `sync_session_id` in API responses today.

| Endpoint | Change |
|----------|--------|
| `GET /metrics/days` | None |
| `GET /metrics/sleep` | None (may return fewer dupes) |
| `GET /metrics/workouts` | Drop `sync_session_id` from JSON if exposed (optional cleanup) |
| `GET /metrics/coverage` | None — MAX timestamps still valid |

---

## 7. Migration plan

Personal DB — full reset is acceptable today (`deploy/reset-db.sh`).

| Step | Action |
|------|--------|
| 1 | Replace `schema.sql` with v2 |
| 2 | Update `sql/queries/*.sql` — upsert on natural keys, remove session deletes |
| 3 | Update `internal/store/*` — rename `Replace*` → `Upsert*` |
| 4 | Update `WriteBatch` order (unchanged: sleep → workouts → activity → temp → health → days) |
| 5 | One-time dedup script if keeping old data (optional): `DELETE` duplicates keeping newest `source_session_id` |
| 6 | Run parser e2e + ingest integration tests |

**No Flutter change required** for schema v2 — upload payload unchanged.

---

## 8. Data lifecycle (diagram)

```
Phone sync
   │
   ▼
POST /ingest
   │
   ├─► sync_sessions      (audit: this upload happened)
   ├─► raw_type_blobs     (audit: raw bytes per type)
   │
   └─► parse → aggregate → upsert canonical
         │
         ├─► sleep_sessions      UNIQUE(started_at, is_nap)
         ├─► workouts            UNIQUE(started_at)
         ├─► activity_sessions   UNIQUE(started_at)
         ├─► health_samples      UNIQUE(metric, sampled_at)
         ├─► temperature_samples UNIQUE(sampled_at)
         └─► daily_metrics       UNIQUE(day_key)
```

Reparse path:

```
raw_type_blobs → parse → aggregate → same upserts (idempotent)
```

---

## 9. Decisions needed from you

Before we edit `schema.sql`:

1. **Full reset OK?** Wipe prod/personal DB and apply v2 fresh — or need dedup migration preserving history?
2. **Workout key:** `started_at` alone, or `(started_at, sport_type)`?
3. **Stale events:** If strap stops reporting an old workout, should it stay forever (yes for now) or add tombstone/reconcile job later?
4. **Multi-device later:** Add `device_mac` to natural keys now, or YAGNI until second strap?

---

## 10. Implementation order (after design sign-off)

1. `schema.sql` v2  
2. sqlc queries + regenerate  
3. `metrics_upsert.go` / `health_samples.go` / `temperature.go`  
4. Tests (`parse_batch`, store integration)  
5. Reset DB + manual ingest smoke  
6. Update `ARCHITECTURE.md` + `TODO.md` in Heliolytics repo  

---

## Appendix — v1 → v2 column map

| v1 column | v2 |
|-----------|-----|
| `sleep_sessions.sync_session_id` | `source_session_id` (not in UNIQUE) |
| `workouts.sync_session_id` | `source_session_id` |
| `activity_sessions.sync_session_id` | `source_session_id` |
| `health_samples.sync_session_id` | `source_session_id` |
| `health_samples.id` | removed (composite PK) |
| `temperature_samples.id` | removed (PK = sampled_at) |
| `temperature_samples.sync_session_id` | `source_session_id` |
