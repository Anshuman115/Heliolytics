# PLAN — Recovery Score Audit + Healthspan ("Body Age") Feature

Status: proposal / not yet implemented
Scope: `Heliolytics` (Go backend) + `Heliolytics_App` (Flutter)
Author note: this plan is separate from `HELIOLYTICS_AUDIT.html`. Nothing here is built yet.

Decisions locked (owner-approved):
- ✅ Build the `user_profile` table (Part 2.3).
- ✅ Recovery R1: add skin-temperature deviation, ~0.09 weight (Part 1.4).
- ✅ Recovery R2: add sleeping SpO₂, ~0.06 weight (Part 1.4).
- ✅ VO₂max: compute our own estimate — it is NOT in the BLE data (Part 2.4).

---

## Part 0 — TL;DR

1. **Recovery score is good and already WHOOP-shaped.** It is a baseline-deviation z-score model,
   HRV-dominant, and — importantly — it already samples HRV/SpO₂/respiratory rate **over the sleep
   window**, not the whole calendar day. That is the single most important thing WHOOP does that most
   clones miss, and we already do it. Verdict: **keep the architecture, tune three things.**
2. **Two WHOOP components we have the data for but don't use yet:** skin-temperature deviation and
   sleeping SpO₂. Adding them as minor weighted inputs is the highest-value change.
3. **Healthspan / Body Age does not exist yet and needs new foundations:** a user-profile table
   (DOB, sex, height, weight), a VO₂max estimate (we have no direct VO₂max — must derive it from
   workout HR + resting HR), and population reference curves. This is a larger build, phased below.
4. **Delivery:** the age computation is heavy and slow-moving, so it is computed in the backend as a
   stored **report snapshot** and **streamed to the app section-by-section over SSE**, exactly as
   requested.

---

## Part 1 — Recovery Score Audit

### 1.1 What we actually run today

Single scorer: `internal/readiness/readiness.go`, function `Compute([]DayVitals) (int, bool)`.

- Called from `internal/parse/write_batch.go` → `store.RecomputeReadiness` after every ingest.
- Writes to `daily_metrics.computed_readiness`; the API serves `COALESCE(readiness, computed_readiness)`,
  i.e. **the device's own 0x39 readiness wins when present**, our score fills the gaps.
- There is exactly one implementation. No dead/alternate scorers. Good.

Model summary:

| Aspect | Current implementation |
|---|---|
| Method | Per-metric z-score vs personal baseline → `clamp(50 ± 25·z, 0, 100)`, weighted mean |
| HRV | `ln(RMSSD)` (log-normal correction), **mean over the sleep window** (`vitals_rollup.go`) |
| Baseline | 7-day mean, 60-day SD, **prior days only** (target day excluded) |
| Weights | HRV 0.50, RHR 0.25, sleep 0.15, resp 0.10; missing components renormalize |
| Cold start | < 3 valid HRV nights → no score; population-prior SD until 14 days of personal history |
| Inputs used | HRV, RHR, respiratory rate, sleep score |
| Inputs captured but NOT used in score | **skin temperature**, **sleeping SpO₂**, stress |

### 1.2 How WHOOP does it (for comparison)

WHOOP Recovery (0–100 %) is, by their published descriptions:

- **HRV-dominant**, measured **during sleep** (WHOOP specifically weights the HRV reading from the
  **last slow-wave-sleep (SWS) period** of the night, considered the most stable/least-noisy window).
- Combined with **resting heart rate**, **sleep performance**, **respiratory rate**, and — added in
  later hardware/firmware — **skin temperature** and **blood oxygen (SpO₂)**.
- Scored against **your own rolling baseline**, not a population norm.
- Proprietary calibration/weighting on top.

### 1.3 Side-by-side verdict

| Dimension | WHOOP | Heliolytics today | Gap? |
|---|---|---|---|
| HRV dominant | Yes | Yes (0.50) | ✅ none |
| HRV during sleep, not whole day | Yes | Yes (sleep-window mean) | ✅ none |
| Log-normal HRV handling | Implied | Yes (`ln(RMSSD)`) | ✅ none |
| Personal baseline, not population | Yes | Yes (7d/60d, prior-only) | ✅ none |
| RHR | Yes | Yes (0.25) | ✅ none |
| Respiratory rate | Yes | Yes (0.10) | ✅ none |
| Sleep | Yes | Yes (0.15, device score) | ✅ none |
| Skin temperature deviation | Yes | **Captured, not scored** | ⚠️ add |
| Sleeping SpO₂ | Yes | **Captured, not scored** | ⚠️ add |
| HRV from **last SWS period** specifically | Yes | Whole-sleep-window mean | 🔸 minor refinement |
| Calibrated weights | Proprietary | Heuristic (evidence-informed) | 🔸 acceptable, document it |

**Bottom line: our recovery score is genuinely close to WHOOP's design.** It is not a toy. The two
"⚠️ add" rows are the only material feature gaps, and we already store the data for both.

### 1.4 Recommendations to make it more WHOOP-like

Ordered by value-for-effort.

#### R1 — Add skin-temperature deviation as a minor component (highest value)
- We store `temperature_samples` (0x2E) and a daily avg. Compute nightly mean skin temp over the
  sleep window (reuse `meanInWindow` in `vitals_rollup.go`).
- Add to `DayVitals`: `SkinTempC *float64`.
- Sub-score: deviation is **bidirectional and non-linear** — both a rise and a fall from baseline are
  bad (fever vs poor perfusion). Use `subscore` with `sign = -1` on the **absolute** deviation, or a
  V-shaped map: `100 - min(100, 40·|z|)`.
- Weight ~0.08–0.10; renormalize others down proportionally.

#### R2 — Add sleeping SpO₂ as a minor component
- Already captured to `daily_metrics.spo2_avg` via sleep-window mean (`spo2Sleep` preferred).
- Add `Spo2 *float64` to `DayVitals`. Lower is worse. Weight ~0.05–0.08.
- Guard: SpO₂ is noisy on wrist optical sensors — floor the influence and require a minimum sample
  count in the window before it contributes.

#### R3 — Refine HRV window to the last SWS period (optional, lower priority)
- Today: mean HRV over the whole main-sleep window.
- WHOOP: HRV from the last slow-wave-sleep block.
- We have sleep stages (0x48, `stages_json`). Could restrict `meanInWindow` to samples that fall
  inside the **last `deep` stage segment** of the night.
- Risk: fewer HRV samples in that narrow window → noisier. Consider "last SWS, else fall back to
  whole-window mean". Measure before committing; this is a refinement, not a fix.

#### R4 — Document the weights as evidence-informed, and add a calibration hook
- The weights are defensible heuristics, not fitted parameters. Keep them, but:
  - Move all weights + priors to named constants with citations (already partly done).
  - Add an internal `readiness_debug` field in the API (behind a flag) that returns each sub-score
    and its weight, so we can eyeball calibration against how we actually feel.

#### Proposed new weight vector (after R1+R2)
```
HRV        0.42
RHR        0.22
Sleep      0.13
Resp       0.08
SkinTemp   0.09
SpO2       0.06
```
(HRV stays dominant; new components take weight proportionally. Tune after 2–3 weeks of data.)

### 1.5 What NOT to change
- Do **not** switch to a population baseline. Personal baseline is correct and WHOOP-aligned.
- Do **not** remove the `COALESCE(device, computed)` preference — device readiness stays authoritative.
- Do **not** fold the target day into its own baseline (current code correctly excludes it).

### 1.6 Recovery-score work breakdown
- [ ] `DayVitals`: add `SkinTempC`, `Spo2` (pure package, no DB).
- [ ] `vitals_rollup.go`: compute sleep-window skin-temp mean into `DayAcc` (SpO₂ already there).
- [ ] `readiness.go`: add two sub-scores + renormalize weights; add bidirectional temp map.
- [ ] Tests: extend the table-driven suite in `readiness_test.go` — temp-high, temp-low, missing-temp,
      low-SpO₂, missing-SpO₂, renormalization correctness.
- [ ] Optional R3 behind a comparison test against real nights.
- No API shape change (still one integer 0–100). No app change required.

---

## Part 2 — Healthspan / "Body Age" Feature

Goal: a WHOOP-Age-style page. Two headline numbers plus a beautiful breakdown:
1. **Body Age** — physiological age vs calendar age ("you're 31, your body is 27").
2. **Pace of Aging** — years aged per calendar year (trend slope; 0.8 = aging slowly).

### 2.1 How WHOOP's version works (target model)
- Takes ~9 longitudinal metrics (VO₂max is the heaviest: cardio fitness is the strongest single
  predictor of physiological age / mortality), plus RHR, HRV, sleep, activity, lean mass, strength
  frequency, zone-2 minutes.
- Maps each metric to an **age-equivalent** against **population reference curves by age & sex**
  ("your VO₂max matches the average 26-year-old").
- Weighted-combines age-equivalents → single Body Age.
- Tracks the slope over months → Pace of Aging.

### 2.2 The hard truth about our inputs
| WHOOP input | Do we have it? |
|---|---|
| VO₂max | ❌ **not measured — must estimate** |
| Resting HR | ✅ `daily_metrics.resting_hr` |
| HRV | ✅ `daily_metrics.hrv_rmssd` |
| Sleep | ✅ sleep score / duration |
| Daily activity / steps | ✅ `daily_metrics.steps` |
| Workout HR (for VO₂max est.) | ✅ `workouts.avg_hr`, `max_hr`, `duration_sec` |
| Age / sex / height / weight | ❌ **no user profile at all** |
| Lean body mass | ❌ (needs weight + body-fat; skip v1) |

**Two blockers before any age math:** (a) no user profile, (b) no VO₂max.

### 2.3 Foundation A — user profile (new table)
```sql
CREATE TABLE user_profile (
  id            SMALLINT PRIMARY KEY DEFAULT 1,          -- single-tenant: one row
  date_of_birth DATE    NOT NULL,
  sex           TEXT    NOT NULL CHECK (sex IN ('male','female')),
  height_cm     NUMERIC,
  weight_kg     NUMERIC,
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```
- Populated from a new app settings screen (name/DOB/sex/height/weight).
- New endpoints: `GET /api/v1/profile`, `PUT /api/v1/profile` (behind existing HMAC auth).
- `sex` + `age` are **required** for population reference curves; height/weight optional in v1.

### 2.4 Foundation B — VO₂max estimate

**Why we don't already have it (confirmed):** VO₂max is **not present in the raw BLE streams**. Zepp/
Huami compute it **server-side in their cloud** and only display it in the Zepp app — it is not written
to the band's on-flash data we sync. Verified two ways: (a) none of the 15 parsed type codes
(0x01…0x49) carries it; (b) prior reverse-engineering notes explicitly list VO₂max under "available via
Zepp API only, computed server-side" alongside training load, body battery, and per-second HR. Pulling
it from Zepp's cloud would require a Zepp login and defeats self-hosting, so **we compute our own estimate.**

No direct VO₂max. Estimate it, best-available method first:

1. **Resting-HR method (Uth–Sørensen), always available:**
   `VO2max ≈ 15.3 × (HRmax / HRrest)`
   - `HRrest` = `daily_metrics.resting_hr` (rolling median).
   - `HRmax` — **derive from per-second HR (`heart_rate_samples`, 0x46), not just `workouts.max_hr`.**
     Per-second data covers all moments (daily spikes + workouts), so it catches the true peak more
     often than logged sessions alone. Three corrections are mandatory:
     ```
     HRmax = max(
       percentile_99( heart_rate_samples.bpm over trailing 90d ),   // observed, artifact-robust
       208 − 0.7 · age                                              // Tanaka floor
     )
     ```
     - **Use the 99th percentile, not `MAX()`** — a single PPG artifact (even within the 30–220 parser
       filter) would poison a raw max. Alternative: require the peak be *sustained* for N consecutive
       seconds.
     - **Floor with the Tanaka age formula** so a sedentary window (never near true max) can't
       underestimate VO₂max. "Observed max" only equals "true HRmax" if the user actually pushed hard.
     - **Caveat to remember:** wrist optical sensors lose lock at high HR + high motion — i.e. the
       noisiest data is exactly at the moments we care about — which is the deeper reason to prefer a
       robust percentile over the raw peak.
   - Needs a new store query: `HRMaxPercentile(ctx, sinceDays, pct)` over the `heart_rate_samples`
     hypertable (cheap: indexed range scan; use `percentile_disc` / `percentile_cont`).

2. **Submaximal-workout method (better when data exists):** use steady-state `avg_hr` at a known
   effort during walks/runs. Requires pace/distance we may not have — **defer to v2**.

Store a rolling `vo2max_est` (e.g. `daily_metrics.vo2max_est NUMERIC`, or a small `fitness_daily`
table) computed on ingest. Smooth over 7–14 days; it should move slowly.

### 2.5 The age model (backend, pure package `internal/bodyage`)
Mirror the readiness package's design: **pure functions, no I/O, fully unit-tested.**

```
BodyAge(profile, metrics []DayFitness) (report BodyAgeReport, ok bool)
```

Pipeline:
1. For each contributing metric, compute the person's current smoothed value:
   VO₂max, RHR, HRV, sleep regularity, daily activity.
2. Map each to an **age-equivalent** via a population reference curve `f_metric(value, sex) → age`.
   - Curves come from published population data (see 2.6). Encode as piecewise-linear tables per sex.
3. Weighted mean of age-equivalents → **Body Age** (VO₂max heaviest, ~0.35–0.45).
4. **Pace of Aging** = linear-regression slope of Body Age vs calendar time over the trailing window
   (e.g. last 90–180 days), expressed as Δ(body years)/Δ(calendar year).
5. Cold-start gating: require N days of VO₂max + RHR history before showing a number; show
   "calibrating" otherwise (same pattern as readiness).

`BodyAgeReport` shape:
```go
type BodyAgeReport struct {
    ChronologicalAge float64
    BodyAge          float64
    PaceOfAging      float64            // e.g. 0.82
    Contributors     []Contributor      // per metric: value, ageEquivalent, weight, delta
    Confidence       string             // "calibrating" | "provisional" | "good"
    ComputedAt       time.Time
}
type Contributor struct {
    Metric        string   // "vo2max","rhr","hrv","sleep","activity"
    Value         float64
    AgeEquivalent float64
    Weight        float64
    VsChrono      float64  // age-equivalent minus chronological (− = younger)
}
```

### 2.6 Reference curves — data sourcing
- **VO₂max by age & sex:** ACSM / Cooper Institute normative tables, or FRIEND registry percentiles.
- **RHR, HRV by age & sex:** published population percentile tables (e.g. large wearable cohort papers).
- Encode as small static Go tables (`internal/bodyage/refcurves.go`), piecewise-linear interpolation.
- Cite each table in comments. Keep them swappable — this is where accuracy lives.
- **Legal/attribution:** cite public academic sources only; no third-party product names in code
  (per CLAUDE.md).

### 2.7 Storage — the report snapshot
The age report is heavy and slow-moving → compute periodically, store, serve fast.
```sql
CREATE TABLE bodyage_report (
  id             BIGSERIAL PRIMARY KEY,
  computed_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  chronological  NUMERIC NOT NULL,
  body_age       NUMERIC NOT NULL,
  pace_of_aging  NUMERIC NOT NULL,
  confidence     TEXT NOT NULL,
  contributors   JSONB NOT NULL          -- []Contributor
);
CREATE INDEX idx_bodyage_recent ON bodyage_report (computed_at DESC);
```
- Recompute trigger: on ingest (cheap check "is today's snapshot stale?") or a daily job.
- Keep history so the page can chart Body Age over time (this is also what Pace-of-Aging regresses on).

### 2.8 Delivery — backend computes, streams to app (SSE)
Requirement: "backend calculation done in backend report, streamed to app."

**Endpoint:** `GET /api/v1/metrics/bodyage/stream` (HMAC-authed), `Content-Type: text/event-stream`.

Design: the endpoint emits the report **progressively**, so the app's beautiful page fills in as each
section is ready, instead of one slow blocking JSON:
```
event: status      data: {"stage":"loading","confidence":"good"}
event: headline    data: {"chronological":31.2,"bodyAge":27.4,"paceOfAging":0.82}
event: contributor data: {"metric":"vo2max","value":48.1,"ageEquivalent":25,"vsChrono":-6.2}
event: contributor data: {"metric":"rhr","value":52,"ageEquivalent":28,"vsChrono":-3.1}
event: contributor data: {"metric":"hrv","value":74,"ageEquivalent":26,"vsChrono":-5.0}
event: trend       data: {"points":[{"date":"2026-05-01","bodyAge":28.9}, ...]}
event: done        data: {"computedAt":"2026-07-04T03:00:00Z"}
```
Go implementation notes:
- Use `http.Flusher`; write `data:` frames and `flush()` after each.
- Respect `r.Context()` cancellation (client disconnect stops the stream).
- Source data from the stored `bodyage_report` snapshot (fast) — the SSE is a *presentation* stream,
  not a recompute. If the snapshot is stale, kick a recompute and stream `status: computing` first.
- This slots beside the existing metrics handlers; **no new auth**, same token middleware.

Fallback: also expose a plain `GET /api/v1/metrics/bodyage` returning the whole snapshot as one JSON,
for clients/tests that don't want SSE. (Cheap to keep both — same data source.)

### 2.9 App side (Flutter, new page)
- New screen `lib/screens/body_age_screen.dart` (full page) + provider
  `lib/providers/body_age_provider.dart` (AsyncNotifier — per CLAUDE.md, no logic in build).
- Service `lib/services/body_age_stream_client.dart`: consume SSE via `dio`'s
  `ResponseType.stream` (or `http` package), parse `event:`/`data:` frames, push partial state into
  the provider so the UI animates in as frames arrive.
- Models in `lib/models/body_age.dart`: `BodyAgeReport`, `Contributor`, `TrendPoint`.
- UI (reuse existing design system):
  - Hero: big radial gauge — Body Age vs Chronological (reuse `helio_score_ring` styling).
  - Pace-of-Aging pill (green < 1.0, amber ≈ 1.0, red > 1.0).
  - Contributor list: each metric, its age-equivalent, and a ± vs your real age.
  - Trend chart: Body Age over time (reuse chart stack; `fl_chart`).
- Nav: add a tab/entry point in `helio_nav_provider` / shell.
- Empty/calibrating state: mirror readiness "building baseline" copy.

### 2.10 Healthspan work breakdown (phased)
**Phase 1 — Foundations**
- [ ] `user_profile` table + migration + `GET/PUT /profile` endpoints.
- [ ] App: profile settings screen (DOB, sex, height, weight).
- [ ] VO₂max estimator (`internal/bodyage/vo2max.go`) + rolling store column/table + tests.

**Phase 2 — Model**
- [ ] `internal/bodyage` pure package: reference curves, age-equivalent mapping, Body Age, Pace of Aging.
- [ ] Table-driven tests incl. cold-start, missing-metric renormalization, known-value fixtures.
- [ ] `bodyage_report` table + recompute-on-ingest hook (staleness check).

**Phase 3 — Delivery**
- [ ] `GET /api/v1/metrics/bodyage` (snapshot JSON) + `.../bodyage/stream` (SSE).
- [ ] SSE frame protocol + `http.Flusher` + context cancellation + tests (httptest).

**Phase 4 — App page**
- [ ] SSE client + provider + models.
- [ ] Body Age screen: radial gauge, pace pill, contributors, trend chart.
- [ ] Nav entry + calibrating/empty states.

**Phase 5 — Tuning**
- [ ] Validate VO₂max estimate against any known reference (a lab test, a treadmill test, or a
      cross-check app) — ground truth beats guessing.
- [ ] Tune weights + curves after 3–4 weeks of real data.

### 2.11 Risks & honest caveats
- **VO₂max is estimated, not measured** — the biggest input is the least certain. Label the number
  "estimated" in the UI. This is the #1 accuracy risk.
- **Reference curves are population averages** — Body Age is a *relative* motivator, not a clinical
  figure. Say so on the page (system-role disclaimer, like the recovery score).
- **Single-tenant assumption** — `user_profile` uses one row (id=1). Multi-user needs `user_id`
  everywhere (same migration as the broader multi-tenancy story). Fine for now.
- **Don't over-promise precision** — WHOOP has years of cohort calibration; ours is v1 heuristic.
  "Evidence-informed estimate, honestly labeled" is the framing.

---

## Part 3 — Suggested sequencing
1. **Recovery R1 + R2** (skin temp + SpO₂) — small, high value, no new tables, ships this week.
2. **Healthspan Phase 1** (profile + VO₂max) — unblocks everything age-related.
3. **Healthspan Phases 2–4** — the model, the stream, the page.
4. **Tuning passes** for both, once real data accumulates.
