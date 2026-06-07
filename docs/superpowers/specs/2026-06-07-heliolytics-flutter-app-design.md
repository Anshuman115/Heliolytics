# Heliolytics — Flutter Android App (Research Tool) — Design

**Status:** Approved (brainstorming complete)
**Date:** 2026-06-07
**Author:** Brainstorming session with the project owner
**Scope:** A Flutter Android app whose job is to extract data from the Helio strap and catalog every data-type code the strap returns. This is a research instrument, not a product. The full system (Go server, Postgres+TimescaleDB, Next.js webapp, analytics) is out of scope here and will get its own specs once the strap's data is understood.

---

## 1. Context & Motivation

The Amazfit Helio Smart Ring collects health data, but Zepp's app is opaque — no raw data, no cross-device sync, no research-backed insights. The long-term goal of Heliolytics is a full-stack health analytics platform that replaces Zepp with a transparent, evidence-based system.

The immediate problem: we don't know exactly what data the Helio strap exposes, in what format, on which characteristics, with what hex codes. Zepp's app shows computed scores (BioCharge, Exertion, VO2 Max, PAI, etc.) but those are derived from raw signals in Zepp's cloud. To design a real system, we need the **raw** inputs.

This spec describes a **research tool** that connects to the strap, pulls the raw data over BLE, and produces a catalog of every type code that comes back. That catalog is the input to the next spec (the real app + Go backend).

## 2. Goals

- Connect to the Helio strap over BLE and complete the full auth handshake (Zepp-style ECDH + AES challenge, using a user-pasted auth key — no Zepp login).
- Request data for every known type code (`0x01, 0x05, 0x13, 0x25, 0x2E, 0x38, 0x3A, 0x3D, 0x48, 0x49`) and the live HR notification (`0x2a37`).
- Save raw bytes for every type code received, including unrecognized ones.
- Surface a discovery summary: which type codes came back, with what status (ok / empty / rejected / unknown), and how many samples / bytes each produced.
- Run in listen-mode for a configurable window after the fetch loop, capturing any unsolicited data the strap broadcasts.
- Be modular, simple, and built with small focused files — easy to extend when we discover new type codes.

## 3. Non-Goals

- Go server, Postgres+TimescaleDB, Next.js webapp — separate future specs.
- Cloud auth, multi-user, account management — auth key is device-local.
- iOS, macOS, web — Android only for v1.
- Cross-device sync — data is per-device for now.
- Research-backed analytics algorithms (HRV trends, recovery scores, etc.) — server-side, later.
- LLM integration — later.
- Polished UI, charts, trends — this is a research tool, not a product. Three screens.
- A typed / structured database — raw blobs are the source of truth.

## 4. Architecture Overview

A single Flutter project, feature-folder layout, one concern per folder, small focused files.

```
lib/
├── main.dart                          # App entry, ProviderScope
├── app.dart                           # MaterialApp + router
│
├── config/
│   └── constants.dart                 # UUIDs, Zepp endpoints (none used), type codes
│
├── auth/                              # Auth key only — no Zepp login
│   ├── auth_key_storage.dart          # flutter_secure_storage wrapper
│   └── auth_key_validator.dart        # 32-hex-char validation
│
├── ble/                               # Everything BLE
│   ├── scanner.dart                   # Scan → emits DiscoveredDevice
│   ├── connector.dart                 # GATT connect → emits GattConnection
│   ├── ecdh_auth.dart                 # sect163k1 ECDH + AES-128 challenge
│   ├── chunked_io.dart                # Huami chunked read/write
│   ├── data_requester.dart            # Loop over types, drive chunked protocol
│   ├── listen_mode.dart               # Subscribe to all characteristics, log unsolicited
│   ├── models.dart                    # DiscoveredDevice, GattConnection, DataChunk
│   └── parsers/                       # One file per type code
│       ├── activity.dart              # 0x01
│       ├── workout.dart               # 0x05 (TBD — unverified by reference doc)
│       ├── stress.dart                # 0x13
│       ├── spo2.dart                  # 0x25
│       ├── temperature.dart           # 0x2E
│       ├── sleep_resp_rate.dart       # 0x38
│       ├── resting_hr.dart            # 0x3A
│       ├── max_hr.dart                # 0x3D
│       ├── sleep_session.dart         # 0x48
│       ├── hrv.dart                   # 0x49
│       ├── live_hr.dart               # 0x2a37 (BLE notify, not chunked)
│       └── unknown.dart               # Catch-all: stores raw bytes for any unrecognized code
│
├── data/                              # Local persistence (raw blobs + JSON)
│   ├── session_store.dart             # Create/write/read session folders
│   ├── dump_writer.dart               # Append raw bytes per type
│   ├── catalog_writer.dart            # Write session.json + types.json
│   ├── session_exporter.dart          # Zip session folder, share via Android intent
│   └── models.dart                    # Session, DumpEntry, SessionCatalog
│
├── ui/                                # Three screens
│   ├── auth_key_screen.dart           # First-boot: paste 32-char hex key
│   ├── home_screen.dart               # Connect, fetch, listen, show summary
│   └── sessions_screen.dart           # List of past sessions + detail view
│
└── utils/
    ├── huami_time.dart                # Packed-bytes timestamp encode/decode
    ├── crypto.dart                    # AES-128 ECB, hex helpers
    └── log.dart                       # In-app + on-disk logger
```

### Module boundaries (one-liner for each)

| Module | Owns | Knows nothing about |
|---|---|---|
| `BleSessionController` | State machine, current session, lifecycle | UI, parsers, storage |
| `Scanner` | BLE scan → `DiscoveredDevice` | Anything else |
| `Connector` | GATT connect → `GattConnection` | Auth, data, parsers |
| `EcdhAuth` | sect163k1 ECDH + AES challenge over the open GATT | Higher-level chunked protocol |
| `ChunkedIo` | Read/write of the Huami chunked protocol | Specific data types |
| `DataRequester` | Loop over types: send start sync → ack → receive chunks → ack | How bytes are parsed, where samples go |
| `parsers/*` | Bytes → typed samples for one type code | BLE, storage, UI |
| `ListenMode` | Subscribe to all characteristics, log unsolicited data | Requested-data flow |
| `SessionStore` / `DumpWriter` / `CatalogWriter` | Write/read raw bytes + JSON to disk | Parsers, UI |

State management: **`flutter_riverpod`**. Each module exposes a provider for its service. The `BleSessionController` is a `Notifier<SessionState>`.

## 5. Auth Flow

**No Zepp login.** The user already has the auth key (obtained from their reference `Heliolytics` Kotlin app or directly from Zepp once). The app never talks to Zepp's cloud.

### First boot
1. App launches with no auth key in `flutter_secure_storage` → routes to `AuthKeyScreen`.
2. User pastes a 32-character hex string.
3. `AuthKeyValidator` checks: exactly 32 chars, all hex (`[0-9a-fA-F]`), normalized to lowercase.
4. On save: bytes are stored in `flutter_secure_storage` (Android Keystore-backed).
5. App routes to `HomeScreen`.

### Subsequent boots
- App reads the key from secure storage on launch and goes straight to `HomeScreen`.
- If the key is missing or corrupted, falls back to `AuthKeyScreen`.

### The two keys (for reference, see `ble-protocol.md`)
- **Auth Key** — 16 bytes (32 hex), permanent, comes from Zepp's API once. Stored in secure storage.
- **Session Key** — 16 bytes, temporary, derived every connection as `ECDH(our_priv, ring_pub)[8..24] XOR auth_key`. Used only for the current BLE session.

The app never persists the session key. It lives in memory and is discarded on disconnect.

## 6. BLE Flow

### State machine

```
                ┌──────────────────┐
                │  NoAuthKey       │  (first boot — AuthKeyScreen)
                └────────┬─────────┘
                         │ auth key saved
                         ▼
                ┌──────────────────┐
                │  Idle            │  (HomeScreen, have key, not connected)
                └────────┬─────────┘
                         │ user taps "Connect"
                         ▼
                ┌──────────────────┐
                │  Scanning        │  (10s timeout, BLE scan for ring MAC)
                └────────┬─────────┘
                         │ ring found (or timeout → Idle)
                         ▼
                ┌──────────────────┐
                │  Connecting      │  (GATT connect, service discovery)
                └────────┬─────────┘
                         │ GATT ready (or failure → ErrorState)
                         ▼
                ┌──────────────────┐
                │  Authenticating  │  (ECDH + AES challenge)
                └────────┬─────────┘
                         │ auth OK (or failure → ErrorState)
                         ▼
                ┌──────────────────┐
                │  Connected       │  (session-key established, ready to fetch)
                └────────┬─────────┘
                         │ user taps "Fetch" with type selection
                         ▼
                ┌──────────────────┐
                │  Fetching        │  (sequential loop over selected types, 2-day window)
                └────────┬─────────┘
                         │ all types done
                         ▼
                ┌──────────────────┐
                │  Listening       │  (5 min, all characteristics, log unsolicited)
                └────────┬─────────┘
                         │ listen window elapsed
                         ▼
                ┌──────────────────┐
                │  Idle            │  (back to Home, discovery summary populated)
                └──────────────────┘

        (any state) ─── error ───► ErrorState (dismissable → Idle)
```

States are an enum:
```dart
enum SessionState {
  noAuthKey,
  idle,
  scanning,
  connecting,
  authenticating,
  connected,
  fetching,
  listening,
  error,
}
```

Errors are typed and carried on the state:
```dart
enum SessionError {
  zeppKeyMissing,       // shouldn't happen in this design
  scanTimeout,
  scanFailed,
  gattFailed,
  authRejected,
  authTimeout,
  chunkTimeout,
  parseFailed,          // logged, doesn't break the loop
  bleDisconnected,
  unknown,
}
```

### Fetch round-trip per type

For each type code `T` the user has selected:

```
1. ChunkedIo.write(command: [0x01, T, ...timestamp(2-day window)])
   ↓
2. ChunkedIo.read() ← response: [0x01, T, status, count, ...]
   • status=0x01 + count=N → expected N samples
   • status=0x02 / 0x04 → rejected, log it
   • other → unknown response, log it
   ↓
3. ChunkedIo.write(command: [0x02])   (transfer)
   ↓
4. ChunkedIo.read() × N ← chunks: [counter, data...]
   • reassemble counter-stripped bytes into one buffer
   • if a counter resets or chunks stall → 1 retry, then mark failed
   ↓
5. ChunkedIo.write(command: [0x03, 0x09])   (ack)
   ↓
6. parsers/<T>.parse(buffer) → List<Sample>
   • if no parser or parse fails → parsers/unknown.dart writes raw bytes
   ↓
7. DumpWriter.append(sessionId, T, raw_bytes)
   ↓
8. CatalogWriter.addEntry(sessionId, {code: T, count, bytes, status})
```

Sequential — chunked protocol is one-outstanding-at-a-time, no parallelism.

### Per-type response outcomes (all four are recorded, nothing is silently dropped)

| Outcome | Meaning | Catalog entry |
|---|---|---|
| `ok` | Strap returned data, parser succeeded | `{code, status: ok, samples, bytes}` |
| `empty` | Strap returned `count=0` | `{code, status: empty, samples: 0, bytes: 0}` |
| `rejected` | Strap returned an error code | `{code, status: rejected, errorByte}` |
| `unknown` | Parser failed or no parser exists | `{code, status: unknown, bytes}` |

## 7. Discovery Mode

The primary output of this app is the **type-code catalog** built up across sessions.

### Default fetch behavior

| Setting | Default | Override |
|---|---|---|
| Fetch window | **2 days** (`since = now - 48h`) | settings dialog (1h / 6h / 1d / 2d / 7d) |
| Types to request | All known: `0x01, 0x05, 0x13, 0x25, 0x2E, 0x38, 0x3A, 0x3D, 0x48, 0x49` | checkboxes on HomeScreen |
| Request order | Sequential, in code order | not configurable |
| Listen-mode after fetch | 5 min | settings dialog (off / 1m / 5m / 15m) |
| Listen-only mode | Off (toggle) | checkbox on HomeScreen (skips fetch loop) |

### Discovery summary card (HomeScreen, after each session)

The main artifact of the app. Updates after every fetch + listen cycle.

```
┌─────────────────────────────────────────────────────────┐
│  Last session — 2026-06-07 14:32                        │
│  Device: AA:BB:CC:DD:EE:FF                              │
│                                                         │
│  Type codes received (chunked, 2-day window):           │
│    0x01  Activity        142 samples    568 bytes       │
│    0x05  Workout           0 samples    (rejected: 04)  │
│    0x13  Stress           60 samples    360 bytes       │
│    0x25  SpO2              2 samples     24 bytes       │
│    0x2E  Temperature       0 samples    (empty)         │
│    0x3A  Resting HR        0 samples    (empty)         │
│    0x3D  Max HR            0 samples    (empty)         │
│    0x48  Sleep session     0 samples    (empty)         │
│    0x49  HRV              30 samples    180 bytes       │
│                                                         │
│  Type codes received (unsolicited / live):              │
│    0x2a37 Heart Rate (live)   87 notifications          │
│    0x1A  ???                 12 notifications   ← NEW   │
│                                                         │
│  New / unrecognized codes (since last session):         │
│    0x1A  (12 notifs, payload ≈ 1B+1B)  [view bytes]     │
│                                                         │
│  [ View raw dumps ]  [ Share session (zip) ]           │
└─────────────────────────────────────────────────────────┘
```

### Listen mode

After the fetch loop (or instead of it, in listen-only mode), subscribe to notifications on **every characteristic the strap exposes** for 5 minutes (configurable). Anything that arrives is written to the session with `source: "unsolicited"` and surfaced in the discovery summary as "unsolicited / live."

This is the safest way to discover new types — we're not asking, we're listening. No risk of the strap interpreting a malformed request as a command.

### Out of scope for v1: active probing

A "Probe unknown types" mode that tries a range of type codes (e.g., `0x02`–`0x4A` excluding known) is **not** in v1. The risk: the strap might interpret unknown codes as commands. If we want this later, it must be opt-in, behind a confirmation dialog, and clearly labeled in the catalog as `probed: true`.

## 8. Data Type Codes

### Confirmed (from the reference protocol doc — see `ble-protocol.md`)

| Code | Stat | Source | Notes |
|------|------|--------|-------|
| `0x01` | Activity (HR, kind, steps, intensity) | chunked | 4 bytes/sample |
| `0x13` | Stress | chunked | |
| `0x25` | SpO2 | chunked | |
| `0x2E` | Temperature | chunked | |
| `0x38` | Sleep respiratory rate | chunked | |
| `0x3A` | Resting HR | chunked | |
| `0x3D` | Max HR | chunked | |
| `0x48` | Sleep session | chunked | 594 bytes/session fixed blob |
| `0x49` | HRV | chunked | 6 bytes/sample (timestamp[4] + rmssd[1] + ?[1]) |
| `0x2a37` | Heart Rate (live) | BLE notify | standard BLE HR profile, separate transport |

### TBD (placeholder, verify against the real strap)

| Code | Stat | Source | Status |
|------|------|--------|--------|
| `0x05` | Workout | chunked | Mentioned in project's `TODO.md`; not in the reference doc's data-type table. Parser file exists but is marked TBD until verified. |

### Catch-all

`parsers/unknown.dart` is invoked whenever:
- The strap returns a response for a type code we don't have a parser for, OR
- A parser is invoked but fails (bad length, bad magic, bad checksum, etc.)

It writes the raw bytes to the session's dump folder and records the type code in the catalog with `status: unknown`.

### Explicitly NOT in this app (out of scope)

The following metrics are **derived**, computed by Zepp's cloud from the raw signals above. They have **no hex code** that the BLE app would ever receive. They belong in the future server-side analytics spec, not in this app's parsers folder.

- BioCharge, Exertion, Exertion Load
- Training Status, VO2 Max, PAI, Fitness Level, Fatigue Level
- Sleep stages, Sleep graph (the strap's `0x48` is the raw sleep-session blob; stages are derived from it)
- Calories (likely a rollup; could come as part of a daily-summary bundle, but we won't try to fetch it)

## 9. Storage Layout

App-private internal storage. Standard Flutter path resolver (`path_provider`'s `getApplicationDocumentsDirectory()`).

```
<app-docs>/
└── heliolytics/
    └── sessions/
        └── <session-uuid>/
            ├── session.json
            ├── types.json
            ├── 0x01_activity.bin
            ├── 0x05_workout.bin
            ├── 0x13_stress.bin
            ├── 0x25_spo2.bin
            ├── 0x2E_temperature.bin
            ├── 0x38_sleep_resp_rate.bin
            ├── 0x3A_resting_hr.bin
            ├── 0x3D_max_hr.bin
            ├── 0x48_sleep_session.bin
            ├── 0x49_hrv.bin
            ├── 0x2a37_live_hr.bin
            ├── unsolicited_<code>_<n>.bin
            └── ...
```

### `session.json`

```json
{
  "schemaVersion": 1,
  "sessionId": "uuid",
  "startedAt": "2026-06-07T14:32:00Z",
  "endedAt": "2026-06-07T14:38:12Z",
  "deviceMac": "AA:BB:CC:DD:EE:FF",
  "fetchWindowHours": 48,
  "listenDurationSec": 300,
  "mode": "fetch+listen"
}
```

### `types.json`

```json
{
  "schemaVersion": 1,
  "sessionId": "uuid",
  "chunked": [
    {"code": "0x01", "status": "ok",      "samples": 142, "bytes": 568, "file": "0x01_activity.bin"},
    {"code": "0x05", "status": "rejected", "samples": 0,   "errorByte": "0x04"},
    {"code": "0x13", "status": "ok",      "samples": 60,  "bytes": 360, "file": "0x13_stress.bin"},
    {"code": "0x25", "status": "ok",      "samples": 2,   "bytes": 24,  "file": "0x25_spo2.bin"},
    {"code": "0x2E", "status": "empty",   "samples": 0,   "bytes": 0},
    {"code": "0x3A", "status": "empty",   "samples": 0,   "bytes": 0},
    {"code": "0x3D", "status": "empty",   "samples": 0,   "bytes": 0},
    {"code": "0x48", "status": "empty",   "samples": 0,   "bytes": 0},
    {"code": "0x49", "status": "ok",      "samples": 30,  "bytes": 180, "file": "0x49_hrv.bin"}
  ],
  "unsolicited": [
    {"code": "0x2a37", "kind": "live_hr", "count": 87, "file": "0x2a37_live_hr.bin"},
    {"code": "0x1A",   "kind": "unknown", "count": 12, "file": "unsolicited_0x1A_0.bin", "firstBytesHex": "0A1F..."}
  ]
}
```

`schemaVersion` is present from day 1 so future versions can detect old sessions and migrate or skip them.

### Auth key

Stored in `flutter_secure_storage` (Android Keystore-backed), **not** in the file system. Key: `heliolytics.auth_key`. Value: 32 lowercase hex chars.

### Export

"Sessions" screen has a "Share" action per session → zips the session folder → Android share intent. User can drop the zip into Drive, send it to themselves, or `adb pull` it. This is the only way data leaves the device for now (server sync is a future spec).

## 10. UI Screens

Three screens, no navigation library beyond Flutter's `Navigator`. Material 3 theme, system dark mode.

### Screen 1: `AuthKeyScreen` (first boot only)

```
┌─────────────────────────────────────┐
│  Heliolytics                        │
│                                     │
│  Paste your Helio auth key          │
│  (32 hex characters)                │
│                                     │
│  ┌─────────────────────────────┐   │
│  │ a1b2c3d4e5f6a7b8...         │   │
│  └─────────────────────────────┘   │
│                                     │
│  Where do I find this? [link]       │
│                                     │
│  [ Save ]                           │
└─────────────────────────────────────┘
```

- Validates on submit: exactly 32 chars, all hex, normalized to lowercase.
- "Where do I find this?" → opens a small modal explaining: from Zepp, or from the reference Helio app, or via Zepp's `GET /users/{userId}/devices` endpoint (auth key is in the device's `additionalInfo.auth_key` field).
- After save → `HomeScreen`.

### Screen 2: `HomeScreen`

```
┌─────────────────────────────────────┐
│  Heliolytics              [⚙]       │
│                                     │
│  ● Idle                             │
│                                     │
│  [   Connect to ring   ]            │
│                                     │
│  Types to fetch:                    │
│   ☑ 0x01 Activity                   │
│   ☑ 0x05 Workout  (TBD)             │
│   ☑ 0x13 Stress                     │
│   ☑ 0x25 SpO2                       │
│   ☑ 0x2E Temperature                │
│   ☑ 0x38 Sleep resp rate            │
│   ☑ 0x3A Resting HR                 │
│   ☑ 0x3D Max HR                     │
│   ☑ 0x48 Sleep session              │
│   ☑ 0x49 HRV                        │
│   ☐ Listen-only (skip fetch)        │
│                                     │
│  Window: [2 days  ▼]                │
│  Listen: [5 min   ▼]                │
│                                     │
│  [ Fetch + Listen ]                 │
│  [ Listen only ]                    │
│                                     │
│  ── Last session ───────────────    │
│  2026-06-07 14:32                   │
│  ... (discovery summary card)       │
│                                     │
│  [ View sessions ]                  │
└─────────────────────────────────────┘
```

- The state pill at the top reflects `SessionState` in real time.
- The "Connect" / "Fetch" / "Listen only" buttons are enabled/disabled per state (e.g., can't Fetch without being Connected).
- The discovery summary card appears after the first completed session and updates after every subsequent one.
- The settings cog (⚙) opens a small dialog for fetch-window and listen-duration defaults.

### Screen 3: `SessionsScreen`

```
┌─────────────────────────────────────┐
│  Sessions                    [←]    │
│                                     │
│  2026-06-07 14:32                   │
│  AA:BB:CC:DD:EE:FF  ·  6 types      │
│                                     │
│  2026-06-07 10:15                   │
│  AA:BB:CC:DD:EE:FF  ·  4 types      │
│                                     │
│  2026-06-06 22:01                   │
│  AA:BB:CC:DD:EE:FF  ·  9 types      │
│                                     │
└─────────────────────────────────────┘
```

- Tapping a session → `SessionDetailScreen` (the same discovery-summary card, full-screen, with "Share zip" and "Delete" actions).

### No charts, no trends, no settings beyond the two dropdowns. This is a research tool.

## 11. Testing Approach

### Unit tests (run in CI / locally)

Targeted at the parts that don't need a real BLE radio:

- `auth_key_validator_test.dart` — rejects wrong lengths, non-hex chars, mixed case (normalizes to lowercase).
- `huami_time_test.dart` — roundtrip encode/decode for known timestamps (Unix epoch, leap years, DST boundaries).
- `chunked_io_test.dart` — chunk assembly with counter wrap, missing-chunk detection, oversized-chunk rejection.
- `parsers/activity_test.dart` — parses the 4-bytes/sample format from the reference doc.
- `parsers/hrv_test.dart` — parses the 6-bytes/sample format from the reference doc.
- `parsers/unknown_test.dart` — captures bytes correctly when given a type code with no parser.
- `catalog_writer_test.dart` — `types.json` schema is well-formed, `schemaVersion: 1` always present, atomic write (no half-written files on crash).
- `session_exporter_test.dart` — zip includes all files, no path traversal.

Use the `test` package + `flutter_test`. ~80% of the test value in the smallest amount of code.

### What we do NOT test

- **No BLE mocking.** The whole point is to test against the real ring. We don't write a fake `flutter_blue_plus` — that would be testing our mock, not our code.
- **No emulator testing for BLE.** Android emulators have no real Bluetooth radio.
- **No integration tests beyond a manual smoke test.**

### Manual smoke test (the integration test)

1. Plug in an Android phone with the ring paired and on the wrist.
2. `flutter run` on the device.
3. Paste auth key, save.
4. Tap Connect → confirm state goes Idle → Scanning → Connecting → Authenticating → Connected.
5. Tap Fetch + Listen → confirm discovery summary populates with non-zero counts for at least the high-cadence types (`0x01`, `0x13`, `0x49`).
6. Open the session's zipped export and confirm the binary files are non-empty and the JSON is well-formed.
7. Repeat with a 7-day window to catch low-cadence types (`0x48` sleep, `0x3A` RHR) — should now show non-zero.

If those three things work, the chain works end-to-end. If they don't, the bug is in BLE handling, parsing, or storage — and the discovery summary tells you which.

### Lint

`flutter_lints` package, default rules. No custom rules. CI runs `flutter analyze` and `flutter test`.

## 12. Future-Proofing Seams

Tiny, intentional, no over-engineering:

- **`DataRequester.fetch(type, since)` method.** When the Go server exists, the only change is swapping the file-write sink for an HTTP POST. One method, no other changes.
- **`parsers/` folder shape.** Adding `exertion.dart` or `biocharge.dart` later is a one-file change. If we later discover the strap does send derived metrics, we add parsers for them.
- **`schemaVersion` in `session.json` and `types.json` from day 1.** Future versions can detect old sessions and migrate or skip them.
- **Type codes are centralized in `config/constants.dart`.** The catalog, the parsers folder, and the home-screen checkboxes all read from the same source. Adding a new code = one entry in one file.
- **No analytics, no charts, no derived metrics.** This app stays focused on extraction. The next spec (the real app) will introduce a `DerivedMetric` layer on top of the raw data.

## 13. Dependencies

`pubspec.yaml` will pin:

- `flutter_riverpod: ^2.5.0` — state management
- `flutter_blue_plus: ^1.31.0` — BLE on Android (most mature cross-platform option)
- `flutter_secure_storage: ^9.0.0` — auth key in Android Keystore
- `path_provider: ^2.1.0` — find app docs dir
- `archive: ^3.6.0` — zip a session folder for export
- `share_plus: ^9.0.0` — Android share intent
- `crypto: ^3.0.0` — AES-128, hex helpers (or `pointycastle` if we need ECDH on `sect163k1`; verify when implementing)

## 14. Out-of-Scope Reminders (for the next spec)

These will need their own specs after this app produces its catalog:

- **Go server** — HTTP API, ingest endpoint, auth model.
- **Postgres + TimescaleDB** — schema, migrations, retention policy.
- **Next.js webapp** — dashboards, charts, login.
- **Mobile app v2** — proper UI for browsing historical data, charts, trends. Reads from Go server, not from local dumps.
- **Analytics layer** — research-backed algorithms for HRV trends, recovery, sleep scoring, etc.
- **LLM insights** — natural-language summaries of trends.
- **iOS, macOS, web** — Flutter already supports these, but BLE and platform integrations will need verification.

## 15. Open Questions

These are TBD and won't block v1:

- Does the strap actually support `0x05` (Workout)? Verify with the dummy app.
- Does the strap send a daily-summary bundle (steps, calories, distance as a rollup) on a separate code? The reference doc doesn't show one, but Gadgetbridge references a `dataTypesHex` flag with a `STEPS` bit. If yes, we'll see it in the discovery summary and add a parser.
- What does the strap broadcast unsolicitedly on the Huami custom characteristics? The listen-mode window is how we'll find out.
- Does the strap have a `0x2a37` live-HR notification behavior that conflicts with the chunked HR data? We may need to disable one while using the other.

## 16. Decision Log

| Decision | Choice | Rationale |
|---|---|---|
| Platform | Android only | User already has Android + reference Kotlin app; Flutter BLE on Android is mature. |
| Data source | Strap (BLE) raw only | Per user direction: "Strap for raw + we compute derived" — derived metrics are server-side later. |
| Auth | User-pasted 32-char hex key, no Zepp login | User already has the key; avoids Zepp API client + logout risk. |
| Fetch window default | 2 days | User-specified. Catches sleep + RHR + low-cadence types. |
| Storage | Opaque binary blobs per type + JSON metadata | No premature schema; raw bytes are source of truth. |
| Storage location | App-private internal storage | Standard for app data; not visible to other apps. |
| Export | Zip + Android share intent | Simplest cross-device transfer for now. |
| State management | `flutter_riverpod` | Small, no boilerplate, testable. |
| BLE library | `flutter_blue_plus` | Mature, well-documented, what most Flutter BLE projects use. |
| Tests | Unit tests for parsers/validators/utils; no BLE mocking | Don't test our mocks. Manual smoke test is the integration test. |
| UI | Three screens, Material 3, no charts | Research tool, not a product. |
| Derived metrics | Out of scope, future server-side spec | Zepp-cloud-computed; not raw strap data. |

---

**End of design.** This spec is the input to the `writing-plans` skill, which will produce the implementation plan.
