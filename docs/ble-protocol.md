# Heliolytics — BLE Protocol Reference

> Source: project owner's reference notes (Kotlin, Heliolytics) shared during brainstorming on 2026-06-07.
> This is the protocol guide the Flutter app implements. Keep in sync if the protocol changes.

---

## The Big Picture

```
┌─────────────┐                    ┌─────────────┐
│  Zepp Cloud │                    │  Helio Ring │
│  (Internet) │                    │  (BLE)      │
└──────┬──────┘                    └──────┬──────┘
       │                                  │
       │ 1. Get auth key                 │
       │◄─────────────────────────────────┤ (login with email/password)
       │                                  │
       │                                  │
┌──────▼──────────────────────────────────▼──────┐
│              Your App                            │
│                                                   │
│  2. Scan for ring (BLE)                          │
│  3. Connect (BLE)                                │
│  4. Authenticate (ECDH + AES)                    │
│  5. Request data (chunked)                       │
│  6. Receive data (chunked)                       │
│  7. Parse bytes → samples                        │
│  8. Store in database                            │
└───────────────────────────────────────────────────┘
```

In this Flutter app, **step 1 is replaced by a user-pasted auth key** (no Zepp login).
Steps 2–7 are implemented in the `ble/` folder of the app. Step 8 is local-file storage
in v1 (raw blobs + JSON), with a Go server coming in a future spec.

---

## Step 1: Get the Auth Key (From Zepp Cloud)

**What it is:** A 16-byte (32 hex characters) symmetric key that proves to the ring that you own it.

**Where it comes from:** Zepp's cloud servers (NOT from the ring itself).

**How to get it (reference only — not implemented in this app):**

```kotlin
// 1. Login to Zepp with your email/password
POST https://api-user-us2.zepp.com/v2/registrations/tokens
   (encrypted with AES-128-CBC, key="xeNtBVqzDc6tuNTh", iv="MAAAYAAAAAAAAABg")

// 2. Get access token from redirect (HTTP 303)

// 3. Login to get app_token
POST /v2/client/login

// 4. Get your devices (including the ring's auth_key)
GET /users/{userId}/devices
   Header: apptoken: {appToken}

Response:
{
  "items": [
    {
      "macAddress": "AA:BB:CC:DD:EE:FF",
      "activeStatus": 1,
      "additionalInfo": "{\"auth_key\":\"a1b2c3d4e5f6...\"}"  // ← THIS IS THE KEY
    }
  ]
}
```

**The auth key looks like:** `a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6` (32 hex chars = 16 bytes)

**Why you need it:** The ring won't talk to you without it. It's like a password that proves you're the owner.

**Important:** Fetch this key ONCE and save it. Don't login to Zepp every time (it might log you out of the official app).

**In this app:** the user pastes this key on first boot. We never call Zepp.

---

## Step 2: Scan for the Ring (BLE)

```kotlin
// BLE Service UUID (Huami protocol)
private val chunkedWriteUUID = UUID.fromString("00000016-0000-3512-2118-0009af100700")
private val chunkedReadUUID  = UUID.fromString("00000017-0000-3512-2118-0009af100700")

// Scan
bluetoothLeScanner.startScan(scanCallback)
// Ring advertises itself, your app sees it
```

**What happens:** Your phone scans for nearby Bluetooth devices. The ring advertises itself. Your app sees it in the scan results.

---

## Step 3: Connect to the Ring (BLE)

```kotlin
fun connect(device: BluetoothDevice) {
    gatt = device.connectGatt(context, false, gattCallback, BluetoothDevice.TRANSPORT_LE)
}
```

**What happens:**
- Phone establishes a BLE connection
- Discovers services (the ring exposes several)
- Finds the characteristics you need:
  - `0x16` (Chunked Write) — you write commands here
  - `0x17` (Chunked Read) — ring sends responses here
  - `0x04` (Activity Control) — you send "start sync" commands here
  - `0x05` (Activity Data) — ring sends data here
  - `0x2a37` (Heart Rate) — live HR notifications

---

## Step 4: Authenticate (ECDH + AES)

This is the cryptographic handshake. Here's what happens:

### 4a. Generate your keypair (ECDH on curve sect163k1)

```kotlin
val kp = HuamiECDH.generateKeypair()
// kp.privateKey = your private key (24 bytes)
// kp.publicKey = your public key (48 bytes)
```

**What it is:** Elliptic Curve Diffie-Hellman key exchange. You and the ring each generate a keypair. You'll use these to create a shared secret.

### 4b. Send your public key to the ring

```kotlin
val header = byteArrayOf(0x04, 0x02, 0x00, 0x02)
val payload = header + authPub  // 4 bytes header + 48 bytes public key
sendChunked(0x0082.toShort(), payload)
```

**What happens:** You send your public key to the ring via the chunked write characteristic.

### 4c. Ring responds with its public key + random challenge

```kotlin
// Ring sends back:
// - 16 random bytes (challenge)
// - 48 bytes (ring's public key)
val random = payload.sliceArray(3 until 19)        // 16 bytes
val remotePub = payload.sliceArray(19 until 67)    // 48 bytes
```

### 4d. Derive shared secret (ECDH)

```kotlin
val shared = HuamiECDH.generateShared(authPriv, remotePub)
// shared = 48 bytes (the shared secret)
```

**What it is:** Both you and the ring combine your private key with the other's public key. Magic of math: you both get the SAME result, but nobody listening can figure it out.

### 4e. Derive session key

```kotlin
val sessionKey = ByteArray(16)
val authKeyBytes = authKeyHex.chunked(2).map { it.toInt(16).toByte() }.toByteArray()

for (i in 0 until 16) {
    sessionKey[i] = (shared[i + 8].toInt() xor authKeyBytes[i].toInt()).toByte()
}
```

**What it is:** You XOR the middle 16 bytes of the shared secret with your auth key. This is the session key that proves you know the auth key.

### 4f. Encrypt the challenge (proves you know the auth key)

```kotlin
val enc1 = AES128.ecbEncryptNoPadding(authKeyBytes, random)  // Encrypt with auth key
val enc2 = AES128.ecbEncryptNoPadding(sessionKey, random)   // Encrypt with session key

val cmd = byteArrayOf(0x05) + enc1 + enc2
sendChunked(0x0082.toShort(), cmd)
```

**What it is:** You encrypt the ring's random challenge twice:
1. With the auth key (proves you have the auth key)
2. With the session key (proves you derived the session key correctly)

The ring can verify both = you must be the real owner.

### 4g. Ring accepts → authentication complete

```kotlin
if (status == 0x01.toByte()) {
    _isAuthenticated.value = true
    // Now you can request data!
}
```

---

## Step 5: Request Data (Chunked Protocol)

The ring uses a custom chunked protocol because BLE has small packet sizes (max ~247 bytes).

### 5a. Send "start sync" command

```kotlin
// For each data type, send a command:
// Format: [0x01, type_code] + timestamp_bytes
val cmd = byteArrayOf(0x01, 0x01) + HuamiTime.bytesFromDate(sinceDate)
// 0x01 = start sync command
// 0x01 = activity data type
// timestamp = when you want data from
```

**Data types you can request:**

| Code | Data Type | Source |
|------|-----------|--------|
| `0x01` | Activity (HR, kind, steps, intensity) | confirmed |
| `0x05` | Workout | TBD — see app's TODO.md |
| `0x13` | Stress | confirmed |
| `0x25` | SpO2 | confirmed |
| `0x2E` | Temperature | confirmed |
| `0x38` | Sleep respiratory rate | confirmed |
| `0x3A` | Resting HR | confirmed |
| `0x3D` | Max HR | confirmed |
| `0x48` | Sleep session | confirmed |
| `0x49` | HRV | confirmed |

Plus, separate from chunked-sync:

| Code | Stat | Transport |
|------|------|-----------|
| `0x2a37` | Heart Rate (live) | BLE notify (standard profile) |

### 5b. Ring responds with data count

```kotlin
// Ring sends back:
// 0x01 (response type)
// 0x01 (start sync command echo)
// 0x01 (status: OK)
// expected_count (how many samples you'll get)
```

### 5c. Trigger data transfer

```kotlin
// You send: [0x02] (transfer command)
// Ring starts sending data in chunks
```

### 5d. Receive data chunks

```kotlin
// Each chunk:
// [counter_byte] [data_bytes...]
// counter_byte increments to detect packet loss
// data_bytes is the raw payload
```

### 5e. Acknowledge receipt

```kotlin
// You send: [0x03, 0x09] (acknowledge command)
// Ring stops sending
```

### 5f. Parse the raw bytes

```kotlin
// Now you have raw bytes, e.g.:
// Activity: 4 bytes per sample (kind, intensity, steps, hr)
// HRV: 6 bytes per sample (timestamp[4] + rmssd[1] + ???[1])
// Sleep: 594 bytes per session (fixed blob)

val samples = HelioLegacyParser.parse(code, name, rawBytes, roundStart)
```

---

## Step 6: Store & Use the Data

In the reference Kotlin app, this writes to a local Room/SQLite database. In the Heliolytics Flutter app v1, raw bytes are written to opaque `.bin` files per type, with `session.json` + `types.json` for metadata. Server sync is a future spec.

---

## Summary: The Complete Flow

```
1. Get auth key from Zepp cloud (one-time, 32 hex chars)
   (skipped in this app — user pastes the key)
   ↓
2. Scan for ring (BLE)
   ↓
3. Connect to ring (BLE GATT)
   ↓
4. Authenticate (ECDH key exchange + AES challenge)
   - Generate your keypair
   - Send your public key
   - Receive ring's public key + random challenge
   - Derive shared secret
   - Derive session key (shared_secret XOR auth_key)
   - Encrypt challenge (proves you know auth key)
   - Send encrypted challenge
   - Ring verifies → authenticated!
   ↓
5. Request data (for each type: activity, HRV, sleep, etc.)
   - Send "start sync" command with timestamp
   - Ring responds with expected count
   - Send "transfer" command
   - Receive data in chunks
   - Send "acknowledge" command
   - Parse raw bytes → samples
   ↓
6. Store raw bytes in app-private files; build the type-code catalog
   ↓
7. Display discovery summary in the app; share zipped session via Android intent
```

---

## The Two Keys Explained

| Key | Size | Where it comes from | What it does |
|-----|------|---------------------|--------------|
| **Auth Key** | 16 bytes (32 hex) | Zepp cloud (from your Zepp account) | Proves you own the ring. Fetched once. |
| **Session Key** | 16 bytes | Derived from ECDH shared secret XOR auth key | Used for this session only. Changes every time. |

**The auth key is permanent** (until you reset the ring).
**The session key is temporary** (regenerated every connection).

---

## Why This Is Complex

The Huami protocol uses:
1. **Custom ECDH** on binary curve `sect163k1` (not standard `secp256k1`)
2. **Custom chunking** because BLE packets are small
3. **Custom encryption** with the auth key as proof of ownership
4. **Custom time format** (packed bytes, not Unix timestamp)

All of this is to prevent unauthorized apps from talking to the ring. Zepp wants to control which apps can access the data.

---

## What This App Builds

1. **BLE scanner** — find the ring (uses `flutter_blue_plus`)
2. **GATT client** — connect and discover services
3. **ECDH implementation** — the `sect163k1` curve math
4. **AES-128** — for the challenge encryption
5. **Zepp API client** — **NOT** in this app. User pastes the auth key directly.
6. **Chunked encoder/decoder** — for the custom protocol
7. **Byte parsers** — to decode each data type (one file per type, small and focused)
8. **Type-code catalog** — the primary output of the app

That's a lot of work! But once you have all 8 pieces, you can extract any data the ring provides and know exactly which hex codes it speaks.
