# Heliolytics Schema v3 — Design

> **Status:** Implemented on v3 branch — apply via `schema_v3.sql` + `deploy/reset-db.sh`.  
> **Do not edit v2:** `schema.sql` + `SCHEMA_DESIGN.md` stay frozen on v2.  
> **Apply v3 via:** `schema_v3.sql` + `deploy/reset-db.sh` pointing at v3 file.

---

## Decisions (locked)

| Topic | Choice |
|-------|--------|
| `day_key` type | **`DATE` everywhere** (IST calendar day from parser) |
| DB migration | **Full reset OK** — wipe volume, apply `schema_v3.sql` |
| Event dedup | Natural keys — no `session_id` in UNIQUE |
| `sleep_sessions` | **`UNIQUE (started_at, day_key)`** |
| `workouts` / `activity_sessions` | **`UNIQUE (day_key, started_at)`** |
| Continuous HR | **New `heart_rate_samples` hypertable — `0x46` only** |
| Minute HR (`0x01`) | **Out of scope v3** — no backfill from activity records |
| Workout per-sec HR (`0x06`) | **Out of scope v3** — stays in workout summary cols only |
| Temperature dedup | **`PRIMARY KEY (sampled_at)`** — strap minute time, upsert |
| Vitals series dedup | **`PRIMARY KEY (metric, sampled_at)`** on `health_samples` |
| Provenance | **`source_session_id`** on canonical rows (audit only) |

---

## v3 vs v2 delta

| Change | v2 | v3 |
|--------|----|----|
| `day_key` | `TEXT` `'YYYY-MM-DD'` | `DATE` |
| HR time series | none | **`heart_rate_samples`** from **`0x46`** |
| `temperature_samples` PK | `(sampled_at, id)` | `(sampled_at)` — drop surrogate `id` |
| `health_samples` PK | `(sampled_at, id)` | `(metric, sampled_at)` — drop `id` |
| Canonical session col | `sync_session_id` | **`source_session_id`** (not in UNIQUE) |
| Flutter sync types | no `0x46` | **must add `0x46`** |

---

## Table model

### Ingest layer (unchanged role)

- **`sync_sessions`** — one row per phone upload batch
- **`raw_type_blobs`** — `(session_id, type_code)` raw bytes for reparse

### Canonical layer

| Table | Primary key | Hypertable | Notes |
|-------|-------------|------------|-------|
| `daily_metrics` | `day_key DATE` | no | One row per IST day; upsert merge rules unchanged |
| `sleep_sessions` | `id BIGSERIAL` + **`UNIQUE (started_at, day_key)`** | no | |
| `workouts` | `id BIGSERIAL` + **`UNIQUE (day_key, started_at)`** | no | |
| `activity_sessions` | `id BIGSERIAL` + **`UNIQUE (day_key, started_at)`** | no | |
| `heart_rate_samples` | **`sampled_at`** | **yes** | ~1/sec from **`0x46`** |
| `health_samples` | **`(metric, sampled_at)`** | yes | stress, hrv, spo2, rhr, resp_rate, max_hr |
| `temperature_samples` | **`sampled_at`** | yes | ~1/min skin temp from **`0x2E`** |

---

## `heart_rate_samples`

Continuous PPG session data from BLE type **`0x46`**. Separate table because cardinality (~86k rows/day) dwarfs other vitals.

```sql
CREATE TABLE heart_rate_samples (
  sampled_at        TIMESTAMPTZ NOT NULL,
  day_key           DATE NOT NULL,
  bpm               SMALLINT NOT NULL CHECK (bpm BETWEEN 30 AND 220),
  source_session_id TEXT NOT NULL,
  PRIMARY KEY (sampled_at)
);
```

- **`sampled_at`** — strap sample time (not ingest time)
- **`day_key`** — denormalized IST date for `WHERE day_key BETWEEN …` without TZ math in SQL
- **No `source` column** — v3 only ingests `0x46`
- **Upsert:** `ON CONFLICT (sampled_at) DO UPDATE SET bpm = EXCLUDED.bpm, …`

Helio Strap live HR on watch face = real-time UI. Historical continuous HR in our DB = whatever the band wrote into **`0x46`** blobs during PPG sessions.

---

## `temperature_samples`

Strap minute grid from **`0x2E`** (catalog `roundStart` + 60s steps). Safe business key = `sampled_at`.

```sql
PRIMARY KEY (sampled_at)
-- upsert on conflict; drop BIGSERIAL id
```

---

## `health_samples`

Keep one hypertable for lower-rate vitals. **Do not** store HR here in v3.

| `metric` | BLE | Resolution |
|----------|-----|------------|
| `stress` | `0x13` | ~1/min |
| `hrv` | `0x49` | sparse |
| `spo2` | `0x25` | spot |
| `spo2_sleep` | `0x26` | sleep |
| `rhr` | `0x3A` | daily-ish |
| `resp_rate` | `0x38` | sleep |
| `max_hr` | `0x3D` | daily |

```sql
PRIMARY KEY (metric, sampled_at)
```

---

## Write path

```
POST /ingest
  → sync_sessions + raw_type_blobs
  → parse → aggregate → upsert canonical (natural keys)
```

| Table | ON CONFLICT | Update policy |
|-------|-------------|---------------|
| `daily_metrics` | `day_key` | COALESCE / GREATEST (same as v2) |
| `sleep_sessions` | `(started_at, day_key)` | replace fields from EXCLUDED |
| `workouts` / `activity_sessions` | `(day_key, started_at)` | replace fields from EXCLUDED |
| `heart_rate_samples` | `sampled_at` | `bpm = EXCLUDED.bpm` |
| `temperature_samples` | `sampled_at` | `celsius = EXCLUDED.celsius` |
| `health_samples` | `(metric, sampled_at)` | `value = EXCLUDED.value` |

Stop session-scoped `DELETE … WHERE sync_session_id = $1` before insert on canonical tables.

---

## Read API (new / extended for v3)

| Endpoint | Purpose |
|----------|---------|
| `GET /api/v1/metrics/hr?from&to` | **new** — `heart_rate_samples` for charts |
| `GET /api/v1/metrics/series?from&to` | unchanged — `health_samples` only |
| `GET /api/v1/metrics/temperature?from&to` | unchanged table, fewer dupes |
| `GET /api/v1/metrics/days?from&to` | `dayKey` JSON still ISO date string from `DATE` |

Coverage: add `0x46` end timestamp from `MAX(heart_rate_samples.sampled_at)`.

---

## Flutter / ingest contract

**Required for v3:**

1. Add **`0x46`** to `fetchTypeCodes` in Heliolytics_App
2. Upload part **`0x46_raw.bin`** on ingest (same multipart shape as other types)
3. No payload shape change for existing types

---

## Implementation order

1. Land `schema_v3.sql`; wire `docker-compose` / `reset-db.sh` on v3 branch
2. sqlc: new `heart_rate_samples.sql`; fix temp + health upserts; DATE params for `day_key`
3. Go: `ParseContinuousHr` for `0x46`; store upsert; coverage + metrics HR endpoint
4. Flutter: fetch + upload `0x46`
5. Reset DB + smoke ingest + chart spot-check

---

## Deferred (post-v3)

- Minute HR from **`0x01`** into `heart_rate_samples` or separate table
- Per-second workout HR from **`0x06`** as time series
- Timescale compression / retention policies
- `device_mac` in natural keys (multi-strap)
- `user_id` column
