# Heliolytics Flutter App Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Flutter Android research tool that connects to the Helio strap, extracts raw BLE data, and produces a catalog of every data-type code the strap returns.

**Architecture:** Single Flutter app, feature-folder layout, `flutter_riverpod` state management, `flutter_blue_plus` for BLE, raw bytes stored as opaque `.bin` files per type with JSON metadata. No server. No Zepp login. Three screens: AuthKey, Home, Sessions.

**Tech Stack:** Flutter (Dart 3.x), `flutter_riverpod`, `flutter_blue_plus`, `flutter_secure_storage`, `path_provider`, `archive`, `share_plus`, `pointycastle`, `crypto`, `flutter_lints`.

**Spec:** `docs/superpowers/specs/2026-06-07-heliolytics-flutter-app-design.md`
**Protocol Reference:** `docs/ble-protocol.md`

---

## Phase 0: Project Foundation

### Task 1: Initialize Flutter project

**Files:**
- Create: `heliolytics/` (Flutter project root)

- [ ] **Step 1: Verify Flutter is installed**

Run: `flutter --version`
Expected: prints Dart and Flutter versions.

- [ ] **Step 2: Create the Flutter project**

Run from the parent directory (the one containing `README.md`, etc.):

```bash
cd "/Users/anshumantripathy/Documents/Hobby Projects"
flutter create --org com.heliolytics --project-name heliolytics --platforms android heliolytics_app
mv heliolytics_app/* heliolytics_app/.[!.]* "/Users/anshumantripathy/Documents/Hobby Projects/Heliolytics/" 2>/dev/null || true
rmdir heliolytics_app 2>/dev/null || true
```

Expected: Flutter scaffold moves into the project root alongside `README.md` and `docs/`. The existing `docs/` is preserved.

- [ ] **Step 3: Verify the project runs**

Run: `cd "/Users/anshumantripathy/Documents/Hobby Projects/Heliolytics" && flutter pub get && flutter analyze`
Expected: `No issues found!`

- [ ] **Step 4: Initialize git and commit the scaffold**

```bash
cd "/Users/anshumantripathy/Documents/Hobby Projects/Heliolytics"
git init
git add .
git commit -m "chore: initial flutter project scaffold"
```

---

### Task 2: Configure Android manifest for BLE permissions

**Files:**
- Modify: `android/app/src/main/AndroidManifest.xml`

- [ ] **Step 1: Add BLE permissions**

In `android/app/src/main/AndroidManifest.xml`, inside `<manifest>` before `<application>`, add:

```xml
    <uses-permission android:name="android.permission.BLUETOOTH" android:maxSdkVersion="30" />
    <uses-permission android:name="android.permission.BLUETOOTH_ADMIN" android:maxSdkVersion="30" />
    <uses-permission android:name="android.permission.BLUETOOTH_SCAN" android:usesPermissionFlags="neverForLocation" />
    <uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" android:maxSdkVersion="30" />
    <uses-feature android:name="android.hardware.bluetooth_le" android:required="true" />
```

- [ ] **Step 2: Verify and commit**

```bash
flutter analyze
git add android/app/src/main/AndroidManifest.xml
git commit -m "chore: add BLE permissions and feature declaration to AndroidManifest"
```

---

### Task 3: Add dependencies

**Files:**
- Modify: `pubspec.yaml`

- [ ] **Step 1: Edit pubspec.yaml**

Add to `dependencies:`:

```yaml
  flutter_riverpod: ^2.5.0
  flutter_blue_plus: ^1.31.0
  flutter_secure_storage: ^9.0.0
  path_provider: ^2.1.0
  archive: ^3.6.0
  share_plus: ^9.0.0
  pointycastle: ^3.9.0
  crypto: ^3.0.0
  path: ^1.9.0
```

Ensure `dev_dependencies` has `flutter_test` (sdk) and add `flutter_lints: ^4.0.0`.

- [ ] **Step 2: Install and commit**

```bash
flutter pub get
git add pubspec.yaml pubspec.lock
git commit -m "chore: add app dependencies"
```

---

### Task 4: Configure lint rules

**Files:**
- Modify: `analysis_options.yaml`

- [ ] **Step 1: Replace analysis_options.yaml**

```yaml
include: package:flutter_lints/flutter.yaml

linter:
  rules:
    prefer_const_constructors: true
    prefer_const_literals_to_create_immutables: true
    require_trailing_commas: true

analyzer:
  exclude:
    - "**/*.g.dart"
    - "**/*.freezed.dart"
```

- [ ] **Step 2: Verify and commit**

```bash
flutter analyze
git add analysis_options.yaml
git commit -m "chore: configure flutter_lints rules"
```

---

## Phase 1: Pure Utilities

### Task 5: Constants file (TDD)

**Files:**
- Create: `lib/config/constants.dart`
- Create: `test/config/constants_test.dart`

- [ ] **Step 1: Write the failing test**

`test/config/constants_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:heliolytics/config/constants.dart';

void main() {
  test('all known type codes are present', () {
    expect(knownTypeCodes, containsAll([
      '0x01', '0x05', '0x13', '0x25', '0x2E',
      '0x38', '0x3A', '0x3D', '0x48', '0x49',
    ]));
  });
  test('chunked-write and chunked-read UUIDs are distinct', () {
    expect(chunkedWriteUUID, isNot(equals(chunkedReadUUID)));
  });
}
```

- [ ] **Step 2: Run test (should fail)**

```bash
flutter test test/config/constants_test.dart
```

- [ ] **Step 3: Implement constants.dart**

```dart
const String huamiServiceUUID = '0000fee0-0000-1000-8000-00805f9b34fb';
const String chunkedWriteUUID = '00000016-0000-3512-2118-0009af100700';
const String chunkedReadUUID  = '00000017-0000-3512-2118-0009af100700';
const String activityControlUUID = '00000004-0000-3512-2118-0009af100700';
const String activityDataUUID    = '00000005-0000-3512-2118-0009af100700';
const String liveHeartRateUUID   = '00002a37-0000-1000-8000-00805f9b34fb';

const List<String> knownTypeCodes = [
  '0x01', '0x05', '0x13', '0x25', '0x2E',
  '0x38', '0x3A', '0x3D', '0x48', '0x49',
];

const String liveHeartRateTypeCode = '0x2a37';

const String appDocsSubdir = 'heliolytics';
const String sessionsSubdir = 'sessions';
const String authKeyStorageKey = 'heliolytics.auth_key';

const int defaultFetchWindowHours = 48;
const int defaultListenDurationSec = 300;
const int scanTimeoutSec = 10;
const int chunkReceiveTimeoutSec = 5;
const int chunkRetryCount = 1;
```

- [ ] **Step 4: Verify and commit**

```bash
flutter test test/config/constants_test.dart
git add lib/config/constants.dart test/config/constants_test.dart
git commit -m "feat(config): add constants for UUIDs, type codes, and storage"
```

---

### Task 6: Auth key validator (TDD)

**Files:**
- Create: `lib/auth/auth_key_validator.dart`
- Create: `test/auth/auth_key_validator_test.dart`

- [ ] **Step 1: Write the failing test**

`test/auth/auth_key_validator_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:heliolytics/auth/auth_key_validator.dart';

void main() {
  group('AuthKeyValidator.validate', () {
    test('accepts valid 32-char lowercase hex', () {
      expect(AuthKeyValidator.validate('a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6'), isNull);
    });
    test('accepts valid 32-char uppercase hex', () {
      expect(AuthKeyValidator.validate('A1B2C3D4E5F6A7B8C9D0E1F2A3B4C5D6'), isNull);
    });
    test('rejects empty string', () {
      expect(AuthKeyValidator.validate(''), isNotNull);
    });
    test('rejects wrong length', () {
      expect(AuthKeyValidator.validate('a1b2c3d4'), contains('32'));
    });
    test('rejects non-hex chars', () {
      expect(AuthKeyValidator.validate('g1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6'), contains('hex'));
    });
  });
  group('AuthKeyValidator.normalize', () {
    test('lowercases uppercase hex', () {
      expect(AuthKeyValidator.normalize('AABBCC'), 'aabbcc');
    });
    test('trims whitespace', () {
      expect(AuthKeyValidator.normalize('  aabbcc  '), 'aabbcc');
    });
  });
  group('AuthKeyValidator.toBytes', () {
    test('converts 32-char hex to 16 bytes', () {
      final bytes = AuthKeyValidator.toBytes('a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6');
      expect(bytes.length, 16);
      expect(bytes[0], 0xa1);
      expect(bytes[15], 0xd6);
    });
    test('throws on invalid', () {
      expect(() => AuthKeyValidator.toBytes('zzzz'), throwsFormatException);
    });
  });
}
```

- [ ] **Step 2: Run test (should fail)**

```bash
flutter test test/auth/auth_key_validator_test.dart
```

- [ ] **Step 3: Implement auth_key_validator.dart**

```dart
import 'dart:typed_data';

class AuthKeyValidator {
  static String? validate(String key) {
    final n = normalize(key);
    if (n.length != 32) return 'Auth key must be exactly 32 hex characters (got ${n.length}).';
    if (!RegExp(r'^[0-9a-f]{32}$').hasMatch(n)) {
      return 'Auth key must contain only hex characters (0-9, a-f).';
    }
    return null;
  }

  static String normalize(String key) => key.trim().toLowerCase();

  static Uint8List toBytes(String key) {
    final reason = validate(key);
    if (reason != null) throw FormatException('Invalid auth key: $reason');
    final n = normalize(key);
    final bytes = Uint8List(16);
    for (var i = 0; i < 16; i++) {
      bytes[i] = int.parse(n.substring(i * 2, i * 2 + 2), radix: 16);
    }
    return bytes;
  }
}
```

- [ ] **Step 4: Verify and commit**

```bash
flutter test test/auth/auth_key_validator_test.dart
git add lib/auth/auth_key_validator.dart test/auth/auth_key_validator_test.dart
git commit -m "feat(auth): add auth key validator with hex check and byte conversion"
```

---

### Task 7: Auth key storage (TDD with in-memory fake)

**Files:**
- Create: `lib/auth/auth_key_store.dart`
- Create: `lib/auth/auth_key_storage.dart`
- Create: `test/auth/auth_key_storage_test.dart`

- [ ] **Step 1: Define interface `lib/auth/auth_key_store.dart`**

```dart
abstract class AuthKeyStore {
  Future<void> write(String key, String value);
  Future<String?> read(String key);
  Future<void> delete(String key);
}
```

- [ ] **Step 2: Write test `test/auth/auth_key_storage_test.dart`**

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:heliolytics/auth/auth_key_storage.dart';
import 'package:heliolytics/auth/auth_key_store.dart';

class InMemoryStore implements AuthKeyStore {
  final Map<String, String> _m = {};
  @override Future<String?> read(String k) async => _m[k];
  @override Future<void> write(String k, String v) async => _m[k] = v;
  @override Future<void> delete(String k) async => _m.remove(k);
}

void main() {
  late InMemoryStore backend;
  late AuthKeyStorage storage;

  setUp(() {
    backend = InMemoryStore();
    storage = AuthKeyStorage(store: backend);
  });

  test('save normalizes and persists the key', () async {
    await storage.save('AABBCC11223344556677889900AABBCC');
    expect(await storage.read(), 'aabbcc11223344556677889900aabbcc');
  });

  test('save rejects invalid key', () async {
    expect(() => storage.save('not-hex'), throwsA(isA<FormatException>()));
  });

  test('read returns null when nothing stored', () async {
    expect(await storage.read(), isNull);
  });

  test('clear removes stored key', () async {
    await storage.save('aabbcc11223344556677889900aabbcc');
    await storage.clear();
    expect(await storage.read(), isNull);
  });

  test('hasKey returns true after save', () async {
    expect(await storage.hasKey(), isFalse);
    await storage.save('aabbcc11223344556677889900aabbcc');
    expect(await storage.hasKey(), isTrue);
  });
}
```

- [ ] **Step 3: Implement `lib/auth/auth_key_storage.dart`**

```dart
import 'dart:typed_data';

import 'package:heliolytics/auth/auth_key_store.dart';
import 'package:heliolytics/auth/auth_key_validator.dart';
import 'package:heliolytics/config/constants.dart';

class AuthKeyStorage {
  final AuthKeyStore _store;
  AuthKeyStorage({required AuthKeyStore store}) : _store = store;

  Future<void> save(String key) async {
    final reason = AuthKeyValidator.validate(key);
    if (reason != null) throw FormatException('Invalid auth key: $reason');
    await _store.write(authKeyStorageKey, AuthKeyValidator.normalize(key));
  }

  Future<String?> read() => _store.read(authKeyStorageKey);
  Future<Uint8List?> readBytes() async {
    final s = await read();
    return s == null ? null : AuthKeyValidator.toBytes(s);
  }
  Future<bool> hasKey() async => (await read()) != null;
  Future<void> clear() => _store.delete(authKeyStorageKey);
}
```

- [ ] **Step 4: Verify and commit**

```bash
flutter test test/auth/auth_key_storage_test.dart
git add lib/auth/auth_key_store.dart lib/auth/auth_key_storage.dart test/auth/auth_key_storage_test.dart
git commit -m "feat(auth): add auth key storage wrapper"
```

---

### Task 8: Huami time utilities (TDD)

**Files:**
- Create: `lib/utils/huami_time.dart`
- Create: `test/utils/huami_time_test.dart`

- [ ] **Step 1: Write the failing test**

`test/utils/huami_time_test.dart`:

```dart
import 'dart:typed_data';
import 'package:flutter_test/flutter_test.dart';
import 'package:heliolytics/utils/huami_time.dart';

void main() {
  test('encodes unix epoch to 4 zero bytes', () {
    final b = HuamiTime.fromDateTime(DateTime.utc(1970, 1, 1));
    expect(b, Uint8List.fromList([0, 0, 0, 0]));
  });
  test('encodes 2026-06-07 to expected bytes', () {
    final b = HuamiTime.fromDateTime(DateTime.utc(2026, 6, 7));
    expect(b, Uint8List.fromList([0x6A, 0x19, 0xE4, 0x00]));
  });
  test('decodes 4 zero bytes to unix epoch', () {
    expect(HuamiTime.toDateTime(Uint8List.fromList([0, 0, 0, 0]))
        .isAtSameMomentAs(DateTime.utc(1970, 1, 1)), isTrue);
  });
  test('roundtrips a known timestamp', () {
    final orig = DateTime.utc(2026, 6, 7, 14, 32);
    expect(HuamiTime.toDateTime(HuamiTime.fromDateTime(orig)).isAtSameMomentAs(orig), isTrue);
  });
}
```

- [ ] **Step 2: Implement `lib/utils/huami_time.dart`**

```dart
import 'dart:typed_data';

class HuamiTime {
  static Uint8List fromDateTime(DateTime dt) {
    final s = dt.toUtc().millisecondsSinceEpoch ~/ 1000;
    final b = ByteData(4)..setUint32(0, s, Endian.big);
    return b.buffer.asUint8List();
  }

  static DateTime toDateTime(Uint8List bytes) {
    final b = ByteData.sublistView(bytes);
    return DateTime.fromMillisecondsSinceEpoch(b.getUint32(0, Endian.big) * 1000, isUtc: true);
  }

  static Uint8List nowMinusHours(int hours) =>
      fromDateTime(DateTime.now().toUtc().subtract(Duration(hours: hours)));
}
```

- [ ] **Step 3: Verify and commit**

```bash
flutter test test/utils/huami_time_test.dart
git add lib/utils/huami_time.dart test/utils/huami_time_test.dart
git commit -m "feat(utils): add Huami packed-bytes timestamp utilities"
```

---

### Task 9: Crypto helpers — AES-128 ECB and hex (TDD)

**Files:**
- Create: `lib/utils/crypto.dart`
- Create: `test/utils/crypto_test.dart`

- [ ] **Step 1: Write the failing test**

`test/utils/crypto_test.dart`:

```dart
import 'dart:typed_data';
import 'package:flutter_test/flutter_test.dart';
import 'package:heliolytics/utils/crypto.dart';

void main() {
  group('hex helpers', () {
    test('bytesToHex lowercases and pads', () {
      expect(CryptoUtils.bytesToHex(Uint8List.fromList([0, 0xab, 0x0f])), '00ab0f');
    });
    test('hexToBytes accepts mixed case', () {
      expect(CryptoUtils.hexToBytes('00AB0F'), [0, 0xab, 0x0f]);
    });
    test('hexToBytes throws on odd length', () {
      expect(() => CryptoUtils.hexToBytes('abc'), throwsFormatException);
    });
    test('hexToBytes throws on non-hex', () {
      expect(() => CryptoUtils.hexToBytes('zzzz'), throwsFormatException);
    });
  });
  group('AES-128 ECB', () {
    test('encrypts deterministically', () {
      final key = Uint8List.fromList(List.generate(16, (i) => i));
      final plain = Uint8List.fromList(List.generate(16, (i) => 0x10 + i));
      final e1 = CryptoUtils.aes128EcbEncrypt(plain, key);
      final e2 = CryptoUtils.aes128EcbEncrypt(plain, key);
      expect(e1, e2);
      expect(e1.length, 16);
      expect(e1, isNot(equals(plain)));
    });
    test('decrypts ciphertext back to plaintext', () {
      final key = Uint8List.fromList(List.generate(16, (i) => i));
      final plain = Uint8List.fromList(List.generate(16, (i) => 0x10 + i));
      final enc = CryptoUtils.aes128EcbEncrypt(plain, key);
      expect(CryptoUtils.aes128EcbDecrypt(enc, key), plain);
    });
  });
}
```

- [ ] **Step 2: Implement `lib/utils/crypto.dart`**

```dart
import 'dart:typed_data';
import 'package:pointycastle/export.dart';

class CryptoUtils {
  static String bytesToHex(Uint8List bytes) {
    final sb = StringBuffer();
    for (final b in bytes) sb.write(b.toRadixString(16).padLeft(2, '0'));
    return sb.toString();
  }

  static Uint8List hexToBytes(String hex) {
    if (hex.length % 2 != 0) {
      throw FormatException('Hex string must have even length (got ${hex.length}).');
    }
    final out = Uint8List(hex.length ~/ 2);
    for (var i = 0; i < out.length; i++) {
      final v = int.tryParse(hex.substring(i * 2, i * 2 + 2), radix: 16);
      if (v == null) throw FormatException('Invalid hex at ${i * 2}');
      out[i] = v;
    }
    return out;
  }

  static Uint8List aes128EcbEncrypt(Uint8List input, Uint8List key) {
    _checkAes(input, key);
    final c = ECBBlockCipher(AESEngine())..init(true, KeyParameter(key));
    return c.process(input);
  }

  static Uint8List aes128EcbDecrypt(Uint8List input, Uint8List key) {
    _checkAes(input, key);
    final c = ECBBlockCipher(AESEngine())..init(false, KeyParameter(key));
    return c.process(input);
  }

  static void _checkAes(Uint8List input, Uint8List key) {
    if (input.length != 16) throw ArgumentError('AES input must be 16 bytes (got ${input.length}).');
    if (key.length != 16) throw ArgumentError('AES key must be 16 bytes (got ${key.length}).');
  }
}
```

- [ ] **Step 3: Verify and commit**

```bash
flutter test test/utils/crypto_test.dart
git add lib/utils/crypto.dart test/utils/crypto_test.dart
git commit -m "feat(utils): add AES-128 ECB and hex helpers"
```

---

## Phase 2: Chunked I/O Protocol

### Task 10: Chunk assembler (TDD)

**Files:**
- Create: `lib/ble/chunked_protocol.dart`
- Create: `test/ble/chunked_protocol_test.dart`

- [ ] **Step 1: Write the failing test**

`test/ble/chunked_protocol_test.dart`:

```dart
import 'dart:typed_data';
import 'package:flutter_test/flutter_test.dart';
import 'package:heliolytics/ble/chunked_protocol.dart';

void main() {
  group('ChunkAssembler', () {
    test('appends a single chunk (counter stripped)', () {
      final a = ChunkAssembler();
      a.append(Uint8List.fromList([0x00, 0xAA, 0xBB]));
      expect(a.payload, [0xAA, 0xBB]);
    });
    test('appends multiple chunks in order', () {
      final a = ChunkAssembler();
      a.append(Uint8List.fromList([0x00, 0xAA]));
      a.append(Uint8List.fromList([0x01, 0xBB]));
      a.append(Uint8List.fromList([0x02, 0xCC]));
      expect(a.payload, [0xAA, 0xBB, 0xCC]);
    });
    test('handles counter wrap 0xFF → 0x00', () {
      final a = ChunkAssembler();
      a.append(Uint8List.fromList([0xFE, 0xAA]));
      a.append(Uint8List.fromList([0xFF, 0xBB]));
      a.append(Uint8List.fromList([0x00, 0xCC]));
      expect(a.payload, [0xAA, 0xBB, 0xCC]);
    });
    test('isComplete true after expected chunks', () {
      final a = ChunkAssembler()..expectChunks(3);
      a.append(Uint8List.fromList([0x00, 0xAA]));
      a.append(Uint8List.fromList([0x01, 0xBB]));
      a.append(Uint8List.fromList([0x02, 0xCC]));
      expect(a.isComplete, isTrue);
    });
    test('isComplete false when chunks missing', () {
      final a = ChunkAssembler()..expectChunks(5);
      a.append(Uint8List.fromList([0x00, 0xAA]));
      expect(a.isComplete, isFalse);
    });
    test('detects counter gap', () {
      final a = ChunkAssembler();
      a.append(Uint8List.fromList([0x00, 0xAA]));
      a.append(Uint8List.fromList([0x02, 0xCC])); // gap
      expect(() => a.append(Uint8List.fromList([0x03, 0xDD])),
          throwsA(isA<ChunkGapException>()));
    });
    test('reset clears state', () {
      final a = ChunkAssembler();
      a.append(Uint8List.fromList([0x00, 0xAA]));
      a.reset();
      a.append(Uint8List.fromList([0x00, 0xBB]));
      expect(a.payload, [0xBB]);
    });
  });
  group('buildChunk', () {
    test('prepends counter byte', () {
      expect(buildChunk(0x05, Uint8List.fromList([0xAA, 0xBB])), [0x05, 0xAA, 0xBB]);
    });
  });
}
```

- [ ] **Step 2: Implement `lib/ble/chunked_protocol.dart`**

```dart
import 'dart:typed_data';

class ChunkGapException implements Exception {
  final int expected, got;
  ChunkGapException(this.expected, this.got);
  @override String toString() => 'ChunkGapException: expected $expected, got $got';
}

class ChunkAssembler {
  final List<int> _payload = [];
  int _next = 0;
  int _expected = -1;

  void expectChunks(int n) {
    _expected = n;
  }

  void append(Uint8List chunk) {
    if (chunk.isEmpty) throw ArgumentError('Empty chunk');
    if (chunk[0] != _next) throw ChunkGapException(_next, chunk[0]);
    for (var i = 1; i < chunk.length; i++) _payload.add(chunk[i]);
    _next = (_next + 1) & 0xFF;
  }

  bool get isComplete => _expected >= 0 && _payloadChunksReceived >= _expected;

  int get _payloadChunksReceived {
    if (_expected < 0) return 0;
    return _next;
  }

  Uint8List get payload => Uint8List.fromList(_payload);

  void reset() {
    _payload.clear();
    _next = 0;
    _expected = -1;
  }
}

Uint8List buildChunk(int counter, Uint8List payload) {
  final out = Uint8List(payload.length + 1);
  out[0] = counter & 0xFF;
  for (var i = 0; i < payload.length; i++) out[i + 1] = payload[i];
  return out;
}
```

- [ ] **Step 3: Verify and commit**

```bash
flutter test test/ble/chunked_protocol_test.dart
git add lib/ble/chunked_protocol.dart test/ble/chunked_protocol_test.dart
git commit -m "feat(ble): add chunked protocol assembler with counter + wrap handling"
```

---

## Phase 3: Parsers (pure functions, TDD)

### Task 11: Activity parser (0x01) — TDD

**Files:**
- Create: `lib/ble/parsers/activity.dart`
- Create: `test/ble/parsers/activity_test.dart`

- [ ] **Step 1: Write the failing test**

`test/ble/parsers/activity_test.dart`:

```dart
import 'dart:typed_data';
import 'package:flutter_test/flutter_test.dart';
import 'package:heliolytics/ble/parsers/activity.dart';

void main() {
  test('parses a single 4-byte sample', () {
    final bytes = Uint8List.fromList([0x01, 0x40, 0x00, 0xAB, 0x48]);
    final s = ActivityParser.parse(bytes);
    expect(s.length, 1);
    expect(s[0].kind, 0x01);
    expect(s[0].intensity, 0x40);
    expect(s[0].steps, 0xAB);
    expect(s[0].heartRate, 72);
  });
  test('parses multiple samples', () {
    final bytes = Uint8List.fromList([
      0x01, 0x40, 0x00, 0x0A, 0x48,
      0x01, 0x80, 0x00, 0x14, 0x50,
    ]);
    final s = ActivityParser.parse(bytes);
    expect(s.length, 2);
    expect(s[1].heartRate, 80);
  });
  test('empty input returns empty list', () {
    expect(ActivityParser.parse(Uint8List(0)), isEmpty);
  });
  test('throws on non-multiple-of-4 length', () {
    expect(() => ActivityParser.parse(Uint8List.fromList([0x01, 0x40, 0x00])),
        throwsArgumentError);
  });
}
```

- [ ] **Step 2: Implement `lib/ble/parsers/activity.dart`**

```dart
import 'dart:typed_data';

class ActivitySample {
  final int kind, intensity, steps, heartRate;
  const ActivitySample({required this.kind, required this.intensity, required this.steps, required this.heartRate});
}

class ActivityParser {
  static List<ActivitySample> parse(Uint8List bytes) {
    if (bytes.isEmpty) return [];
    if (bytes.length % 4 != 0) {
      throw ArgumentError('Activity payload length ${bytes.length} is not a multiple of 4');
    }
    final out = <ActivitySample>[];
    for (var i = 0; i < bytes.length; i += 4) {
      out.add(ActivitySample(
        kind: bytes[i],
        intensity: bytes[i + 1],
        steps: bytes[i + 2],
        heartRate: bytes[i + 3],
      ));
    }
    return out;
  }
}
```

- [ ] **Step 3: Verify and commit**

```bash
flutter test test/ble/parsers/activity_test.dart
git add lib/ble/parsers/activity.dart test/ble/parsers/activity_test.dart
git commit -m "feat(ble): add Activity (0x01) parser"
```

---

### Task 12: HRV parser (0x49) — TDD

**Files:**
- Create: `lib/ble/parsers/hrv.dart`
- Create: `test/ble/parsers/hrv_test.dart`

- [ ] **Step 1: Write the failing test**

`test/ble/parsers/hrv_test.dart`:

```dart
import 'dart:typed_data';
import 'package:flutter_test/flutter_test.dart';
import 'package:heliolytics/ble/parsers/hrv.dart';

void main() {
  test('parses a single 6-byte sample', () {
    final b = Uint8List.fromList([0x6A, 0x19, 0xE4, 0x00, 0x42, 0xFF]);
    final s = HrvParser.parse(b);
    expect(s.length, 1);
    expect(s[0].rmssd, 0x42);
    expect(s[0].unknown, 0xFF);
    expect(s[0].timestamp.isAtSameMomentAs(DateTime.utc(2026, 6, 7)), isTrue);
  });
  test('parses multiple samples', () {
    final b = Uint8List.fromList([
      0x00, 0x00, 0x00, 0x00, 0x40, 0x00,
      0x00, 0x00, 0x00, 0x01, 0x50, 0x00,
    ]);
    expect(HrvParser.parse(b).length, 2);
  });
  test('empty input returns empty list', () {
    expect(HrvParser.parse(Uint8List(0)), isEmpty);
  });
  test('throws on non-multiple-of-6 length', () {
    expect(() => HrvParser.parse(Uint8List.fromList([0, 0, 0, 0, 0])), throwsArgumentError);
  });
}
```

- [ ] **Step 2: Implement `lib/ble/parsers/hrv.dart`**

```dart
import 'dart:typed_data';
import 'package:heliolytics/utils/huami_time.dart';

class HrvSample {
  final Uint8List timestampBytes;
  final DateTime timestamp;
  final int rmssd, unknown;
  const HrvSample({
    required this.timestampBytes,
    required this.timestamp,
    required this.rmssd,
    required this.unknown,
  });
}

class HrvParser {
  static List<HrvSample> parse(Uint8List bytes) {
    if (bytes.isEmpty) return [];
    if (bytes.length % 6 != 0) {
      throw ArgumentError('HRV payload length ${bytes.length} is not a multiple of 6');
    }
    final out = <HrvSample>[];
    for (var i = 0; i < bytes.length; i += 6) {
      final ts = Uint8List.fromList(bytes.sublist(i, i + 4));
      out.add(HrvSample(
        timestampBytes: ts,
        timestamp: HuamiTime.toDateTime(ts),
        rmssd: bytes[i + 4],
        unknown: bytes[i + 5],
      ));
    }
    return out;
  }
}
```

- [ ] **Step 3: Verify and commit**

```bash
flutter test test/ble/parsers/hrv_test.dart
git add lib/ble/parsers/hrv.dart test/ble/parsers/hrv_test.dart
git commit -m "feat(ble): add HRV (0x49) parser"
```

---

### Task 13: Unknown parser (catch-all)

**Files:**
- Create: `lib/ble/parsers/unknown.dart`
- Create: `test/ble/parsers/unknown_test.dart`

- [ ] **Step 1: Write the failing test**

`test/ble/parsers/unknown_test.dart`:

```dart
import 'dart:typed_data';
import 'package:flutter_test/flutter_test.dart';
import 'package:heliolytics/ble/parsers/unknown.dart';

void main() {
  test('returns a single sample with raw bytes preserved', () {
    final b = Uint8List.fromList([0xDE, 0xAD, 0xBE, 0xEF]);
    final s = UnknownParser.parse('0x1A', b);
    expect(s.length, 1);
    expect(s[0].typeCode, '0x1A');
    expect(s[0].rawBytes, b);
  });
  test('captures a short hex preview of first bytes', () {
    final b = Uint8List.fromList(List.generate(20, (i) => i));
    final s = UnknownParser.parse('0x2A', b);
    expect(s[0].firstBytesHex.length, 32);
  });
  test('handles empty input', () {
    final s = UnknownParser.parse('0x00', Uint8List(0));
    expect(s.length, 1);
    expect(s[0].rawBytes.isEmpty, isTrue);
  });
}
```

- [ ] **Step 2: Implement `lib/ble/parsers/unknown.dart`**

```dart
import 'dart:typed_data';
import 'package:heliolytics/utils/crypto.dart';

class UnknownSample {
  final String typeCode;
  final Uint8List rawBytes;
  final String firstBytesHex;
  const UnknownSample({required this.typeCode, required this.rawBytes, required this.firstBytesHex});
}

class UnknownParser {
  static List<UnknownSample> parse(String typeCode, Uint8List bytes) {
    final previewLen = bytes.length < 16 ? bytes.length : 16;
    final preview = bytes.sublist(0, previewLen);
    return [
      UnknownSample(
        typeCode: typeCode,
        rawBytes: bytes,
        firstBytesHex: CryptoUtils.bytesToHex(preview),
      ),
    ];
  }
}
```

- [ ] **Step 3: Verify and commit**

```bash
flutter test test/ble/parsers/unknown_test.dart
git add lib/ble/parsers/unknown.dart test/ble/parsers/unknown_test.dart
git commit -m "feat(ble): add unknown-parser catch-all"
```

---

### Task 14: Stub parsers for the other 8 known types

**Files:** (one file per stub)

- [ ] **Step 1: Create the 8 stub files**

`lib/ble/parsers/workout.dart`:

```dart
import 'dart:typed_data';
import 'package:heliolytics/ble/parsers/unknown.dart';

/// Workout (0x05) — TBD. Defers to UnknownParser until real format is known.
class WorkoutParser {
  static List<UnknownSample> parse(Uint8List bytes) =>
      UnknownParser.parse('0x05', bytes);
}
```

`lib/ble/parsers/stress.dart`:

```dart
import 'dart:typed_data';
import 'package:heliolytics/ble/parsers/unknown.dart';

/// Stress (0x13) — format not yet confirmed.
class StressParser {
  static List<UnknownSample> parse(Uint8List bytes) =>
      UnknownParser.parse('0x13', bytes);
}
```

`lib/ble/parsers/spo2.dart`:

```dart
import 'dart:typed_data';
import 'package:heliolytics/ble/parsers/unknown.dart';

/// SpO2 (0x25) — format not yet confirmed.
class Spo2Parser {
  static List<UnknownSample> parse(Uint8List bytes) =>
      UnknownParser.parse('0x25', bytes);
}
```

`lib/ble/parsers/temperature.dart`:

```dart
import 'dart:typed_data';
import 'package:heliolytics/ble/parsers/unknown.dart';

/// Temperature (0x2E) — format not yet confirmed.
class TemperatureParser {
  static List<UnknownSample> parse(Uint8List bytes) =>
      UnknownParser.parse('0x2E', bytes);
}
```

`lib/ble/parsers/sleep_resp_rate.dart`:

```dart
import 'dart:typed_data';
import 'package:heliolytics/ble/parsers/unknown.dart';

/// Sleep respiratory rate (0x38) — format not yet confirmed.
class SleepRespRateParser {
  static List<UnknownSample> parse(Uint8List bytes) =>
      UnknownParser.parse('0x38', bytes);
}
```

`lib/ble/parsers/resting_hr.dart`:

```dart
import 'dart:typed_data';
import 'package:heliolytics/ble/parsers/unknown.dart';

/// Resting HR (0x3A) — format not yet confirmed.
class RestingHrParser {
  static List<UnknownSample> parse(Uint8List bytes) =>
      UnknownParser.parse('0x3A', bytes);
}
```

`lib/ble/parsers/max_hr.dart`:

```dart
import 'dart:typed_data';
import 'package:heliolytics/ble/parsers/unknown.dart';

/// Max HR (0x3D) — format not yet confirmed.
class MaxHrParser {
  static List<UnknownSample> parse(Uint8List bytes) =>
      UnknownParser.parse('0x3D', bytes);
}
```

`lib/ble/parsers/sleep_session.dart`:

```dart
import 'dart:typed_data';
import 'package:heliolytics/ble/parsers/unknown.dart';

/// Sleep session (0x48) — 594-byte blob per docs, exact structure TBD.
class SleepSessionParser {
  static List<UnknownSample> parse(Uint8List bytes) =>
      UnknownParser.parse('0x48', bytes);
}
```

`lib/ble/parsers/live_hr.dart`:

```dart
import 'dart:typed_data';
import 'package:heliolytics/ble/parsers/unknown.dart';

/// Live heart rate (0x2a37) — standard BLE HR profile.
class LiveHrParser {
  static List<UnknownSample> parse(Uint8List bytes) =>
      UnknownParser.parse('0x2a37', bytes);
}
```

- [ ] **Step 2: Verify and commit**

```bash
flutter analyze
git add lib/ble/parsers/
git commit -m "feat(ble): add stub parsers for remaining type codes (defer to unknown)"
```

---

## Phase 4: Data Models and Storage

### Task 15: Session and DumpEntry models (TDD)

**Files:**
- Create: `lib/data/models.dart`
- Create: `test/data/models_test.dart`

- [ ] **Step 1: Write the failing test**

`test/data/models_test.dart`:

```dart
import 'dart:convert';
import 'package:flutter_test/flutter_test.dart';
import 'package:heliolytics/data/models.dart';

void main() {
  test('DumpEntry.toJson includes schemaVersion and all fields', () {
    final e = DumpEntry(
      code: '0x01', status: DumpStatus.ok,
      samples: 142, bytes: 568, file: '0x01_activity.bin',
    );
    final j = e.toJson();
    expect(j['schemaVersion'], 1);
    expect(j['code'], '0x01');
    expect(j['status'], 'ok');
    expect(j['samples'], 142);
    expect(j['bytes'], 568);
    expect(j['file'], '0x01_activity.bin');
  });
  test('DumpEntry round-trips through JSON', () {
    final orig = DumpEntry(
      code: '0x05', status: DumpStatus.rejected,
      samples: 0, bytes: 0, errorByte: '0x04',
    );
    final round = DumpEntry.fromJson(jsonDecode(jsonEncode(orig.toJson())));
    expect(round.code, '0x05');
    expect(round.status, DumpStatus.rejected);
    expect(round.errorByte, '0x04');
  });
  test('Session.toJson includes schemaVersion and metadata', () {
    final s = Session(
      sessionId: 'abc-123',
      startedAt: DateTime.utc(2026, 6, 7, 14, 32),
      endedAt: DateTime.utc(2026, 6, 7, 14, 38, 12),
      deviceMac: 'AA:BB:CC:DD:EE:FF',
      fetchWindowHours: 48,
      listenDurationSec: 300,
      mode: SessionMode.fetchAndListen,
      entries: const [],
      unsolicited: const [],
    );
    final j = s.toJson();
    expect(j['schemaVersion'], 1);
    expect(j['sessionId'], 'abc-123');
    expect(j['mode'], 'fetch+listen');
  });
}
```

- [ ] **Step 2: Implement `lib/data/models.dart`**

```dart
enum DumpStatus { ok, empty, rejected, unknown }
extension DumpStatusX on DumpStatus {
  String get label => switch (this) {
    DumpStatus.ok => 'ok',
    DumpStatus.empty => 'empty',
    DumpStatus.rejected => 'rejected',
    DumpStatus.unknown => 'unknown',
  };
  static DumpStatus parse(String s) => switch (s) {
    'ok' => DumpStatus.ok, 'empty' => DumpStatus.empty,
    'rejected' => DumpStatus.rejected, 'unknown' => DumpStatus.unknown,
    _ => throw FormatException('Unknown DumpStatus: $s'),
  };
}

enum SessionMode { fetchAndListen, listenOnly }
extension SessionModeX on SessionMode {
  String get label => switch (this) {
    SessionMode.fetchAndListen => 'fetch+listen',
    SessionMode.listenOnly => 'listen-only',
  };
  static SessionMode parse(String s) => switch (s) {
    'fetch+listen' => SessionMode.fetchAndListen,
    'listen-only' => SessionMode.listenOnly,
    _ => throw FormatException('Unknown SessionMode: $s'),
  };
}

class DumpEntry {
  final String code;
  final DumpStatus status;
  final int samples, bytes;
  final String? file;
  final String? errorByte;
  const DumpEntry({
    required this.code, required this.status,
    required this.samples, required this.bytes,
    this.file, this.errorByte,
  });
  Map<String, dynamic> toJson() => {
    'schemaVersion': 1,
    'code': code, 'status': status.label,
    'samples': samples, 'bytes': bytes,
    if (file != null) 'file': file,
    if (errorByte != null) 'errorByte': errorByte,
  };
  factory DumpEntry.fromJson(Map<String, dynamic> j) => DumpEntry(
    code: j['code'] as String,
    status: DumpStatusX.parse(j['status'] as String),
    samples: (j['samples'] as num).toInt(),
    bytes: (j['bytes'] as num).toInt(),
    file: j['file'] as String?,
    errorByte: j['errorByte'] as String?,
  );
}

class UnsolicitedEntry {
  final String code, kind, file;
  final int count;
  final String? firstBytesHex;
  const UnsolicitedEntry({
    required this.code, required this.kind,
    required this.count, required this.file,
    this.firstBytesHex,
  });
  Map<String, dynamic> toJson() => {
    'code': code, 'kind': kind, 'count': count, 'file': file,
    if (firstBytesHex != null) 'firstBytesHex': firstBytesHex,
  };
  factory UnsolicitedEntry.fromJson(Map<String, dynamic> j) => UnsolicitedEntry(
    code: j['code'] as String, kind: j['kind'] as String,
    count: (j['count'] as num).toInt(), file: j['file'] as String,
    firstBytesHex: j['firstBytesHex'] as String?,
  );
}

class Session {
  final String sessionId;
  final DateTime startedAt;
  final DateTime? endedAt;
  final String? deviceMac;
  final int fetchWindowHours, listenDurationSec;
  final SessionMode mode;
  final List<DumpEntry> entries;
  final List<UnsolicitedEntry> unsolicited;
  const Session({
    required this.sessionId, required this.startedAt,
    this.endedAt, this.deviceMac,
    required this.fetchWindowHours, required this.listenDurationSec,
    required this.mode, required this.entries, required this.unsolicited,
  });
  Map<String, dynamic> toJson() => {
    'schemaVersion': 1,
    'sessionId': sessionId,
    'startedAt': startedAt.toIso8601String(),
    if (endedAt != null) 'endedAt': endedAt!.toIso8601String(),
    if (deviceMac != null) 'deviceMac': deviceMac,
    'fetchWindowHours': fetchWindowHours,
    'listenDurationSec': listenDurationSec,
    'mode': mode.label,
  };
  factory Session.fromJson(Map<String, dynamic> j) => Session(
    sessionId: j['sessionId'] as String,
    startedAt: DateTime.parse(j['startedAt'] as String),
    endedAt: j['endedAt'] != null ? DateTime.parse(j['endedAt'] as String) : null,
    deviceMac: j['deviceMac'] as String?,
    fetchWindowHours: (j['fetchWindowHours'] as num).toInt(),
    listenDurationSec: (j['listenDurationSec'] as num).toInt(),
    mode: SessionModeX.parse(j['mode'] as String),
    entries: (j['entries'] as List<dynamic>? ?? [])
      .map((e) => DumpEntry.fromJson(e as Map<String, dynamic>)).toList(),
    unsolicited: (j['unsolicited'] as List<dynamic>? ?? [])
      .map((e) => UnsolicitedEntry.fromJson(e as Map<String, dynamic>)).toList(),
  );
}

class SessionCatalog {
  static const int schemaVersion = 1;
  final String sessionId;
  final List<DumpEntry> chunked;
  final List<UnsolicitedEntry> unsolicited;
  const SessionCatalog({
    required this.sessionId, required this.chunked, required this.unsolicited,
  });
  Map<String, dynamic> toJson() => {
    'schemaVersion': schemaVersion,
    'sessionId': sessionId,
    'chunked': chunked.map((e) => e.toJson()).toList(),
    'unsolicited': unsolicited.map((e) => e.toJson()).toList(),
  };
  factory SessionCatalog.fromJson(Map<String, dynamic> j) => SessionCatalog(
    sessionId: j['sessionId'] as String,
    chunked: (j['chunked'] as List<dynamic>)
      .map((e) => DumpEntry.fromJson(e as Map<String, dynamic>)).toList(),
    unsolicited: (j['unsolicited'] as List<dynamic>)
      .map((e) => UnsolicitedEntry.fromJson(e as Map<String, dynamic>)).toList(),
  );
}
```

- [ ] **Step 3: Verify and commit**

```bash
flutter test test/data/models_test.dart
git add lib/data/models.dart test/data/models_test.dart
git commit -m "feat(data): add Session, DumpEntry, UnsolicitedEntry, SessionCatalog models"
```

---

### Task 16: Session store (TDD)

**Files:**
- Create: `lib/data/session_store.dart`
- Create: `test/data/session_store_test.dart`

- [ ] **Step 1: Write the failing test**

`test/data/session_store_test.dart`:

```dart
import 'dart:io';
import 'package:flutter_test/flutter_test.dart';
import 'package:heliolytics/data/models.dart';
import 'package:heliolytics/data/session_store.dart';
import 'package:path/path.dart' as p;

void main() {
  late Directory tmpRoot;
  setUp(() async {
    tmpRoot = await Directory.systemTemp.createTemp('heliolytics_test_');
  });
  tearDown(() async {
    if (tmpRoot.existsSync()) await tmpRoot.delete(recursive: true);
  });

  test('createSession returns id and creates folder', () async {
    final s = SessionStore(rootDir: tmpRoot);
    final id = await s.createSession(
      deviceMac: 'AA:BB:CC:DD:EE:FF',
      fetchWindowHours: 48, listenDurationSec: 300,
      mode: SessionMode.fetchAndListen,
    );
    expect(id, isNotEmpty);
    expect(Directory(p.join(tmpRoot.path, 'sessions', id)).existsSync(), isTrue);
  });

  test('appendBytes writes raw bytes to per-type file', () async {
    final s = SessionStore(rootDir: tmpRoot);
    final id = await s.createSession(
      deviceMac: null, fetchWindowHours: 48, listenDurationSec: 0,
      mode: SessionMode.fetchAndListen,
    );
    await s.appendBytes(id, '0x01', [0x01, 0x02, 0x03]);
    await s.appendBytes(id, '0x01', [0x04, 0x05]);
    final f = File(p.join(tmpRoot.path, 'sessions', id, '0x01_activity.bin'));
    expect(f.existsSync(), isTrue);
    expect(f.readAsBytesSync(), [0x01, 0x02, 0x03, 0x04, 0x05]);
  });

  test('writeSessionJson and readSessionJson round-trip', () async {
    final s = SessionStore(rootDir: tmpRoot);
    final id = await s.createSession(
      deviceMac: 'AA:BB:CC:DD:EE:FF',
      fetchWindowHours: 48, listenDurationSec: 300,
      mode: SessionMode.fetchAndListen,
    );
    final session = Session(
      sessionId: id, startedAt: DateTime.utc(2026, 6, 7, 14, 32),
      deviceMac: 'AA:BB:CC:DD:EE:FF',
      fetchWindowHours: 48, listenDurationSec: 300,
      mode: SessionMode.fetchAndListen,
      entries: const [], unsolicited: const [],
    );
    await s.writeSessionJson(session);
    final read = await s.readSessionJson(id);
    expect(read.sessionId, id);
    expect(read.deviceMac, 'AA:BB:CC:DD:EE:FF');
  });

  test('writeCatalogJson and readCatalogJson round-trip', () async {
    final s = SessionStore(rootDir: tmpRoot);
    final id = await s.createSession(
      deviceMac: null, fetchWindowHours: 48, listenDurationSec: 0,
      mode: SessionMode.fetchAndListen,
    );
    final c = SessionCatalog(
      sessionId: id,
      chunked: const [
        DumpEntry(code: '0x01', status: DumpStatus.ok, samples: 142, bytes: 568, file: '0x01_activity.bin'),
      ],
      unsolicited: const [],
    );
    await s.writeCatalogJson(c);
    final read = await s.readCatalogJson(id);
    expect(read.chunked.first.code, '0x01');
    expect(read.chunked.first.samples, 142);
  });

  test('listSessions returns reverse chronological', () async {
    final s = SessionStore(rootDir: tmpRoot);
    final id1 = await s.createSession(
      deviceMac: null, fetchWindowHours: 48, listenDurationSec: 0,
      mode: SessionMode.fetchAndListen,
    );
    await Future.delayed(const Duration(milliseconds: 5));
    final id2 = await s.createSession(
      deviceMac: null, fetchWindowHours: 48, listenDurationSec: 0,
      mode: SessionMode.fetchAndListen,
    );
    expect(await s.listSessions(), [id2, id1]);
  });
}
```

- [ ] **Step 2: Implement `lib/data/session_store.dart`**

```dart
import 'dart:convert';
import 'dart:io';
import 'package:heliolytics/data/models.dart';
import 'package:path/path.dart' as p;

class SessionStore {
  final Directory rootDir;
  SessionStore({required this.rootDir});

  Directory _sessionDir(String id) =>
      Directory(p.join(rootDir.path, 'sessions', id));

  String _binFileName(String code) {
    const names = {
      '0x01': '0x01_activity', '0x05': '0x05_workout',
      '0x13': '0x13_stress', '0x25': '0x25_spo2',
      '0x2E': '0x2E_temperature', '0x38': '0x38_sleep_resp_rate',
      '0x3A': '0x3A_resting_hr', '0x3D': '0x3D_max_hr',
      '0x48': '0x48_sleep_session', '0x49': '0x49_hrv',
      '0x2a37': '0x2a37_live_hr',
    };
    return '${names[code] ?? 'unknown_$code'}.bin';
  }

  Future<String> createSession({
    required String? deviceMac,
    required int fetchWindowHours,
    required int listenDurationSec,
    required SessionMode mode,
  }) async {
    final id = DateTime.now().toUtc().microsecondsSinceEpoch.toString();
    final dir = _sessionDir(id);
    await dir.create(recursive: true);
    final s = Session(
      sessionId: id, startedAt: DateTime.now().toUtc(),
      deviceMac: deviceMac,
      fetchWindowHours: fetchWindowHours,
      listenDurationSec: listenDurationSec, mode: mode,
      entries: const [], unsolicited: const [],
    );
    await writeSessionJson(s);
    return id;
  }

  Future<void> appendBytes(String sessionId, String typeCode, List<int> bytes) async {
    final f = File(p.join(_sessionDir(sessionId).path, _binFileName(typeCode)));
    await f.parent.create(recursive: true);
    await f.writeAsBytes(bytes, mode: FileMode.append, flush: true);
  }

  Future<void> writeSessionJson(Session s) async {
    final f = File(p.join(_sessionDir(s.sessionId).path, 'session.json'));
    await f.writeAsString(jsonEncode(s.toJson()), flush: true);
  }

  Future<Session> readSessionJson(String sessionId) async {
    final f = File(p.join(_sessionDir(sessionId).path, 'session.json'));
    final m = jsonDecode(await f.readAsString()) as Map<String, dynamic>;
    final c = await _readCatalogIfPresent(sessionId);
    return Session(
      sessionId: m['sessionId'] as String,
      startedAt: DateTime.parse(m['startedAt'] as String),
      endedAt: m['endedAt'] != null ? DateTime.parse(m['endedAt'] as String) : null,
      deviceMac: m['deviceMac'] as String?,
      fetchWindowHours: (m['fetchWindowHours'] as num).toInt(),
      listenDurationSec: (m['listenDurationSec'] as num).toInt(),
      mode: SessionModeX.parse(m['mode'] as String),
      entries: c?.chunked ?? const [],
      unsolicited: c?.unsolicited ?? const [],
    );
  }

  Future<SessionCatalog?> _readCatalogIfPresent(String sessionId) async {
    final f = File(p.join(_sessionDir(sessionId).path, 'types.json'));
    if (!f.existsSync()) return null;
    return SessionCatalog.fromJson(jsonDecode(await f.readAsString()) as Map<String, dynamic>);
  }

  Future<void> writeCatalogJson(SessionCatalog c) async {
    final f = File(p.join(_sessionDir(c.sessionId).path, 'types.json'));
    final tmp = File('${f.path}.tmp');
    await tmp.writeAsString(jsonEncode(c.toJson()), flush: true);
    await tmp.rename(f.path);
  }

  Future<SessionCatalog> readCatalogJson(String sessionId) async {
    final c = await _readCatalogIfPresent(sessionId);
    if (c == null) throw StateError('No types.json for session $sessionId');
    return c;
  }

  Future<List<String>> listSessions() async {
    final dir = Directory(p.join(rootDir.path, 'sessions'));
    if (!dir.existsSync()) return [];
    final ids = dir.listSync().whereType<Directory>().map((d) => p.basename(d.path)).toList();
    ids.sort((a, b) => b.compareTo(a));
    return ids;
  }
}
```

- [ ] **Step 3: Verify and commit**

```bash
flutter test test/data/session_store_test.dart
git add lib/data/session_store.dart test/data/session_store_test.dart
git commit -m "feat(data): add session store with file I/O and JSON catalog"
```

---

## Phase 5: BLE State Machine and Riverpod

### Task 17: Session state enum and snapshot

**Files:**
- Create: `lib/ble/session_state.dart`

- [ ] **Step 1: Create session_state.dart**

```dart
import 'package:heliolytics/data/models.dart';

enum SessionState {
  noAuthKey, idle, scanning, connecting, authenticating,
  connected, fetching, listening, error,
}

enum SessionError {
  none, scanTimeout, scanFailed, gattFailed, authRejected,
  authTimeout, chunkTimeout, bleDisconnected, unknown,
}

class SessionSnapshot {
  final SessionState state;
  final SessionError error;
  final String? currentTypeCode;
  final Session? lastSession;
  final String? lastErrorMessage;
  const SessionSnapshot({
    required this.state,
    required this.error,
    this.currentTypeCode,
    this.lastSession,
    this.lastErrorMessage,
  });
  static const initial = SessionSnapshot(
    state: SessionState.noAuthKey,
    error: SessionError.none,
  );
  SessionSnapshot copyWith({
    SessionState? state,
    SessionError? error,
    String? currentTypeCode,
    Session? lastSession,
    String? lastErrorMessage,
  }) =>
      SessionSnapshot(
        state: state ?? this.state,
        error: error ?? this.error,
        currentTypeCode: currentTypeCode ?? this.currentTypeCode,
        lastSession: lastSession ?? this.lastSession,
        lastErrorMessage: lastErrorMessage ?? this.lastErrorMessage,
      );
}
```

- [ ] **Step 2: Verify and commit**

```bash
flutter analyze
git add lib/ble/session_state.dart
git commit -m "feat(ble): add SessionState / SessionError enums and SessionSnapshot"
```

---

### Task 18: BLE interfaces and stub scanner/connector

**Files:**
- Create: `lib/ble/ble_devices.dart`
- Create: `lib/ble/scanner.dart`
- Create: `lib/ble/connector.dart`

- [ ] **Step 1: Create `lib/ble/ble_devices.dart`**

```dart
import 'dart:typed_data';

class DiscoveredDevice {
  final String remoteId, name;
  final int rssi;
  const DiscoveredDevice({required this.remoteId, required this.name, required this.rssi});
}

class BleCharacteristic {
  final String uuid;
  final bool canNotify, canWriteWithoutResponse;
  const BleCharacteristic({
    required this.uuid,
    required this.canNotify,
    required this.canWriteWithoutResponse,
  });
}

abstract class GattConnection {
  Future<List<BleCharacteristic>> discoverCharacteristics();
  Future<void> writeChunked(Uint8List bytes);
  Stream<Uint8List> get incoming;
  Future<void> dispose();
}

abstract class BleScanner {
  Stream<DiscoveredDevice> scan({Duration? timeout});
  Future<void> stop();
}

abstract class BleConnector {
  Future<GattConnection> connect(String remoteId);
}
```

- [ ] **Step 2: Create `lib/ble/scanner.dart` (stub)**

```dart
import 'package:heliolytics/ble/ble_devices.dart';

class StubScanner implements BleScanner {
  @override
  Stream<DiscoveredDevice> scan({Duration? timeout}) async* {
    throw UnimplementedError('Scanner not yet wired to flutter_blue_plus');
  }
  @override
  Future<void> stop() async {}
}

BleScanner scannerProvider() => StubScanner();
```

- [ ] **Step 3: Create `lib/ble/connector.dart` (stub)**

```dart
import 'dart:typed_data';
import 'package:heliolytics/ble/ble_devices.dart';
import 'package:heliolytics/ble/scanner.dart';

class StubConnector implements BleConnector {
  @override
  Future<GattConnection> connect(String remoteId) async {
    throw UnimplementedError('Connector not yet wired to flutter_blue_plus');
  }
}

BleConnector connectorProvider() => StubConnector();
```

- [ ] **Step 4: Verify and commit**

```bash
flutter analyze
git add lib/ble/ble_devices.dart lib/ble/scanner.dart lib/ble/connector.dart
git commit -m "feat(ble): add BLE interfaces and stub scanner/connector"
```

---

### Task 19: ECDH auth (TDD with placeholder curve)

**Files:**
- Create: `lib/ble/ecdh_auth.dart`
- Create: `test/ble/ecdh_auth_test.dart`

**NOTE for engineer:** The `_sharedSecret` is a hash-based placeholder. The real protocol needs actual `sect163k1` curve math. The smoke test (Task 27) will reveal whether this needs to be replaced.

- [ ] **Step 1: Write the failing test**

`test/ble/ecdh_auth_test.dart`:

```dart
import 'dart:typed_data';
import 'package:flutter_test/flutter_test.dart';
import 'package:heliolytics/ble/ecdh_auth.dart';

void main() {
  test('generateKeypair returns 24-byte private and 48-byte public', () {
    final kp = EcdhAuth.generateKeypair();
    expect(kp.privateKey.length, 24);
    expect(kp.publicKey.length, 48);
  });
  test('deriveSessionKey is deterministic', () {
    final kp = EcdhAuth.generateKeypair();
    final remotePub = Uint8List.fromList(List.generate(48, (i) => i));
    final authKey = Uint8List.fromList(List.generate(16, (i) => 0xA0 + i));
    final s1 = EcdhAuth.deriveSessionKey(privateKey: kp.privateKey, remotePublicKey: remotePub, authKey: authKey);
    final s2 = EcdhAuth.deriveSessionKey(privateKey: kp.privateKey, remotePublicKey: remotePub, authKey: authKey);
    expect(s1, s2);
    expect(s1.length, 16);
  });
  test('different auth keys produce different session keys', () {
    final kp = EcdhAuth.generateKeypair();
    final remotePub = Uint8List.fromList(List.generate(48, (i) => i));
    final s1 = EcdhAuth.deriveSessionKey(privateKey: kp.privateKey, remotePublicKey: remotePub, authKey: Uint8List.fromList(List.generate(16, (i) => 0)));
    final s2 = EcdhAuth.deriveSessionKey(privateKey: kp.privateKey, remotePublicKey: remotePub, authKey: Uint8List.fromList(List.generate(16, (i) => 0xFF)));
    expect(s1, isNot(equals(s2)));
  });
  test('buildAuthPayload produces [header] + 48-byte public key', () {
    final kp = EcdhAuth.generateKeypair();
    final p = EcdhAuth.buildAuthPayload(kp.publicKey);
    expect(p.length, 4 + 48);
    expect(p[0], 0x04);
    expect(p[1], 0x02);
    expect(p[2], 0x00);
    expect(p[3], 0x02);
    expect(p.sublist(4), kp.publicKey);
  });
  test('buildChallengeResponse produces [0x05] + 32 bytes', () {
    final p = EcdhAuth.buildChallengeResponse(
      authKey: Uint8List.fromList(List.generate(16, (i) => 0xA0 + i)),
      sessionKey: Uint8List.fromList(List.generate(16, (i) => 0x10 + i)),
      challenge: Uint8List.fromList(List.generate(16, (i) => 0x40 + i)),
    );
    expect(p[0], 0x05);
    expect(p.length, 1 + 16 + 16);
  });
}
```

- [ ] **Step 2: Implement `lib/ble/ecdh_auth.dart`**

```dart
import 'dart:math';
import 'dart:typed_data';
import 'package:pointycastle/api.dart' show KeyParameter;
import 'package:pointycastle/digests/sha1.dart';
import 'package:pointycastle/ecc/api.dart';
import 'package:pointycastle/ecc/curves/sec.dart';
import 'package:pointycastle/key_generators/api.dart';
import 'package:pointycastle/key_generators/ec_key_generator.dart';
import 'package:pointycastle/macs/hmac.dart';
import 'package:heliolytics/utils/crypto.dart';

class EcdhKeyPair {
  final Uint8List privateKey, publicKey;
  const EcdhKeyPair({required this.privateKey, required this.publicKey});
}

/// ECDH on Huami's binary curve + AES-128 challenge proof.
/// See docs/ble-protocol.md §4.
class EcdhAuth {
  /// Generate a keypair. The real protocol uses sect163k1; pointycastle
  /// doesn't ship that binary curve, so we use secp160k1 (closest match)
  /// and emit 24-byte private + 48-byte public to match the protocol's
  /// expected sizes. The smoke test will validate this against the real
  /// ring; if it rejects, replace _sharedSecret with proper curve math.
  static EcdhKeyPair generateKeypair() {
    final curve = ECCurve_secp160k1();
    final keyGen = ECKeyGenerator()
      ..init(ParametersWithRandom(
        ECKeyGeneratorParameters(curve),
        Random.secure(),
      ));
    final keyPair = keyGen.generateKeyPair();
    final privateKey = (keyPair.privateKey as ECPrivateKey).d;
    final publicKey = (keyPair.publicKey as ECPublicKey).Q;
    return EcdhKeyPair(
      privateKey: _bigIntToBytes(privateKey!.toBigInteger(), 24),
      publicKey: _encodePoint(publicKey, 24),
    );
  }

  /// Derive the session key from a shared secret XOR auth key.
  static Uint8List deriveSessionKey({
    required Uint8List privateKey,
    required Uint8List remotePublicKey,
    required Uint8List authKey,
  }) {
    final shared = _sharedSecret(privateKey, remotePublicKey);
    final sessionKey = Uint8List(16);
    for (var i = 0; i < 16; i++) {
      sessionKey[i] = shared[i + 8] ^ authKey[i];
    }
    return sessionKey;
  }

  /// Build the [header] + public key payload sent at step 4b.
  static Uint8List buildAuthPayload(Uint8List publicKey) {
    final out = Uint8List(4 + publicKey.length);
    out[0] = 0x04; out[1] = 0x02; out[2] = 0x00; out[3] = 0x02;
    for (var i = 0; i < publicKey.length; i++) out[4 + i] = publicKey[i];
    return out;
  }

  /// Build the [0x05] + enc1 + enc2 payload sent at step 4f.
  static Uint8List buildChallengeResponse({
    required Uint8List authKey,
    required Uint8List sessionKey,
    required Uint8List challenge,
  }) {
    final enc1 = CryptoUtils.aes128EcbEncrypt(challenge, authKey);
    final enc2 = CryptoUtils.aes128EcbEncrypt(challenge, sessionKey);
    final out = Uint8List(1 + enc1.length + enc2.length);
    out[0] = 0x05;
    for (var i = 0; i < enc1.length; i++) out[1 + i] = enc1[i];
    for (var i = 0; i < enc2.length; i++) out[1 + enc1.length + i] = enc2[i];
    return out;
  }

  // ---- internal helpers ----

  /// Placeholder: hash-based "shared secret". Real implementation needs
  /// proper sect163k1 curve math (dA * QB). Replace if smoke test fails.
  static Uint8List _sharedSecret(Uint8List privateKey, Uint8List remotePublicKey) {
    final hmac = HMac(SHA1Digest(), 64)..init(privateKey);
    hmac.update(remotePublicKey);
    final digest = hmac.process(Uint8List(0));
    final out = Uint8List(48);
    for (var i = 0; i < 20; i++) out[i] = digest[i];
    return out;
  }

  static Uint8List _encodePoint(ECPoint p, int fieldSizeBytes) {
    final x = _bigIntToBytes(p.x!.toBigInteger()!, fieldSizeBytes);
    final y = _bigIntToBytes(p.y!.toBigInteger()!, fieldSizeBytes);
    final out = Uint8List(fieldSizeBytes * 2);
    for (var i = 0; i < x.length; i++) out[i] = x[i];
    for (var i = 0; i < y.length; i++) out[fieldSizeBytes + i] = y[i];
    return out;
  }

  static Uint8List _bigIntToBytes(BigInt n, int len) {
    final hex = n.toRadixString(16).padLeft(len * 2, '0');
    final bytes = Uint8List(len);
    for (var i = 0; i < len; i++) {
      bytes[i] = int.parse(hex.substring(i * 2, i * 2 + 2), radix: 16);
    }
    return bytes;
  }
}
```

- [ ] **Step 3: Verify and commit**

```bash
flutter test test/ble/ecdh_auth_test.dart
git add lib/ble/ecdh_auth.dart test/ble/ecdh_auth_test.dart
git commit -m "feat(ble): add ECDH auth (placeholder curve, AES challenge builder)"
```

---

### Task 20: SessionController (TDD with fakes)

**Files:**
- Create: `lib/ble/session_controller.dart`
- Create: `test/ble/session_controller_test.dart`

- [ ] **Step 1: Write the failing test**

`test/ble/session_controller_test.dart`:

```dart
import 'dart:async';
import 'dart:io';
import 'dart:typed_data';

import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';

import 'package:heliolytics/auth/auth_key_storage.dart';
import 'package:heliolytics/auth/auth_key_store.dart';
import 'package:heliolytics/ble/ble_devices.dart';
import 'package:heliolytics/ble/connector.dart';
import 'package:heliolytics/ble/scanner.dart';
import 'package:heliolytics/ble/session_controller.dart';
import 'package:heliolytics/ble/session_state.dart';
import 'package:heliolytics/data/session_store.dart';

class _MemStore implements AuthKeyStore {
  String? k;
  @override Future<String?> read(String k) async => this.k;
  @override Future<void> write(String k, String v) async => this.k = v;
  @override Future<void> delete(String k) async => this.k = null;
}

class _FakeScanner implements BleScanner {
  final _c = StreamController<DiscoveredDevice>.broadcast();
  @override Stream<DiscoveredDevice> scan({Duration? timeout}) => _c.stream;
  @override Future<void> stop() async => _c.close();
}

class _FakeGatt implements GattConnection {
  final _out = StreamController<Uint8List>.broadcast();
  final writes = <Uint8List>[];
  @override Stream<Uint8List> get incoming => _out.stream;
  @override Future<List<BleCharacteristic>> discoverCharacteristics() async => [];
  @override Future<void> writeChunked(Uint8List b) async => writes.add(b);
  @override Future<void> dispose() async => _out.close();
}

class _FakeConnector implements BleConnector {
  _FakeGatt? last;
  @override Future<GattConnection> connect(String id) async {
    last = _FakeGatt();
    return last!;
  }
}

void main() {
  late ProviderContainer c;
  late _MemStore mem;
  late _FakeScanner scanner;
  late _FakeConnector connector;
  late Directory tmp;

  setUp(() {
    mem = _MemStore();
    scanner = _FakeScanner();
    connector = _FakeConnector();
    tmp = Directory.systemTemp.createTempSync('sc_test_');
    c = ProviderContainer(overrides: [
      authKeyStoreProvider.overrideWithValue(mem),
      bleScannerProvider.overrideWithValue(scanner),
      bleConnectorProvider.overrideWithValue(connector),
      sessionStoreProvider.overrideWith((ref) async => SessionStore(rootDir: tmp)),
    ]);
  });
  tearDown(() => c.dispose());

  test('initial state is noAuthKey when no key saved', () {
    expect(c.read(sessionControllerProvider).state, SessionState.noAuthKey);
  });

  test('saving a valid key transitions to idle', () async {
    await c.read(sessionControllerProvider.notifier).saveAuthKey('a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6');
    expect(c.read(sessionControllerProvider).state, SessionState.idle);
  });

  test('saving invalid key throws and does not transition', () async {
    final n = c.read(sessionControllerProvider.notifier);
    expect(() => n.saveAuthKey('not-hex'), throwsA(isA<FormatException>()));
    expect(c.read(sessionControllerProvider).state, SessionState.noAuthKey);
  });
}
```

- [ ] **Step 2: Implement `lib/ble/session_controller.dart`**

```dart
import 'dart:async';
import 'dart:io';
import 'dart:typed_data';

import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:path/path.dart' as p;
import 'package:path_provider/path_provider.dart';

import 'package:heliolytics/auth/auth_key_storage.dart';
import 'package:heliolytics/auth/auth_key_store.dart';
import 'package:heliolytics/ble/ble_devices.dart';
import 'package:heliolytics/ble/connector.dart';
import 'package:heliolytics/ble/ecdh_auth.dart';
import 'package:heliolytics/ble/scanner.dart';
import 'package:heliolytics/ble/session_state.dart';
import 'package:heliolytics/config/constants.dart';
import 'package:heliolytics/data/models.dart';
import 'package:heliolytics/data/session_store.dart';

final authKeyStoreProvider = Provider<AuthKeyStore>(
  (ref) => throw UnimplementedError('Override in ProviderScope'),
);
final bleScannerProvider = Provider<BleScanner>((ref) => scannerProvider());
final bleConnectorProvider = Provider<BleConnector>((ref) => connectorProvider());
final sessionStoreProvider = FutureProvider<SessionStore>((ref) async {
  final root = await getApplicationDocumentsDirectory();
  final dir = Directory(p.join(root.path, appDocsSubdir));
  if (!dir.existsSync()) await dir.create(recursive: true);
  return SessionStore(rootDir: dir);
});

class SessionController extends Notifier<SessionSnapshot> {
  late AuthKeyStorage _authStorage;
  late BleScanner _scanner;
  late BleConnector _connector;
  SessionStore? _store;
  StreamSubscription<DiscoveredDevice>? _scanSub;
  GattConnection? _gatt;

  @override
  SessionSnapshot build() {
    _authStorage = AuthKeyStorage(store: ref.read(authKeyStoreProvider));
    _scanner = ref.read(bleScannerProvider);
    _connector = ref.read(bleConnectorProvider);
    () async {
      _store = await ref.read(sessionStoreProvider.future);
      final hasKey = await _authStorage.hasKey();
      state = state.copyWith(state: hasKey ? SessionState.idle : SessionState.noAuthKey);
    }();
    return SessionSnapshot.initial;
  }

  Future<void> saveAuthKey(String key) async {
    await _authStorage.save(key);
    state = state.copyWith(state: SessionState.idle);
  }

  Future<void> clearAuthKey() async {
    await _authStorage.clear();
    state = state.copyWith(state: SessionState.noAuthKey);
  }

  Future<void> scan() async {
    if (state.state != SessionState.idle) return;
    state = state.copyWith(state: SessionState.scanning, error: SessionError.none);
    final c = Completer<String?>();
    _scanSub = _scanner
        .scan(timeout: Duration(seconds: scanTimeoutSec))
        .listen((d) => c.complete(d.remoteId));
    final id = await c.future.timeout(
      Duration(seconds: scanTimeoutSec + 1),
      onTimeout: () {
        _scanSub?.cancel();
        state = state.copyWith(state: SessionState.error, error: SessionError.scanTimeout);
        return null;
      },
    );
    await _scanSub?.cancel();
    if (id == null) return;
    await _connectAndAuth(id);
  }

  Future<void> _connectAndAuth(String remoteId) async {
    state = state.copyWith(state: SessionState.connecting);
    try {
      _gatt = await _connector.connect(remoteId);
    } catch (_) {
      state = state.copyWith(state: SessionState.error, error: SessionError.gattFailed);
      return;
    }
    state = state.copyWith(state: SessionState.authenticating);
    try {
      await _runEcdhAuth();
      state = state.copyWith(state: SessionState.connected);
    } catch (_) {
      state = state.copyWith(state: SessionState.error, error: SessionError.authRejected);
    }
  }

  /// Stub auth flow. Real GATT-wired version lives in Task 26.
  Future<void> _runEcdhAuth() async {
    final kp = EcdhAuth.generateKeypair();
    final authKey = await _authStorage.readBytes();
    if (authKey == null) throw StateError('No auth key');
    final payload = EcdhAuth.buildAuthPayload(kp.publicKey);
    await _gatt!.writeChunked(payload);
    throw UnimplementedError('Real ECDH auth over BLE not yet wired');
  }

  /// Stub startFetch. Real version with DataRequester lives in Task 28.
  Future<void> startFetch({
    required List<String> typeCodes,
    required int fetchWindowHours,
    required int listenDurationSec,
  }) async {
    if (state.state != SessionState.connected) return;
    state = state.copyWith(state: SessionState.fetching);
    state = state.copyWith(state: SessionState.idle);
  }
}

final sessionControllerProvider =
    NotifierProvider<SessionController, SessionSnapshot>(SessionController.new);
```

- [ ] **Step 3: Verify and commit**

```bash
flutter test test/ble/session_controller_test.dart
git add lib/ble/session_controller.dart test/ble/session_controller_test.dart
git commit -m "feat(ble): add SessionController with state machine and Riverpod wiring"
```

---

## Phase 6: UI

### Task 21: AuthKeyScreen and app shell

**Files:**
- Create: `lib/ui/auth_key_screen.dart`
- Create: `lib/app.dart`
- Create: `lib/main.dart` (overwrite)
- Create: `lib/ui/home_screen.dart` (placeholder)
- Create: `lib/ui/sessions_screen.dart` (placeholder)

- [ ] **Step 1: Create `lib/ui/auth_key_screen.dart`**

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import 'package:heliolytics/auth/auth_key_validator.dart';
import 'package:heliolytics/ble/session_controller.dart';
import 'package:heliolytics/ble/session_state.dart';

class AuthKeyScreen extends ConsumerStatefulWidget {
  const AuthKeyScreen({super.key});
  @override
  ConsumerState<AuthKeyScreen> createState() => _AuthKeyScreenState();
}

class _AuthKeyScreenState extends ConsumerState<AuthKeyScreen> {
  final _ctrl = TextEditingController();
  String? _error;
  bool _saving = false;

  @override
  void dispose() {
    _ctrl.dispose();
    super.dispose();
  }

  Future<void> _save() async {
    final reason = AuthKeyValidator.validate(_ctrl.text);
    if (reason != null) {
      setState(() => _error = reason);
      return;
    }
    setState(() { _error = null; _saving = true; });
    try {
      await ref.read(sessionControllerProvider.notifier).saveAuthKey(_ctrl.text);
    } catch (e) {
      setState(() { _error = e.toString(); _saving = false; });
    }
  }

  @override
  Widget build(BuildContext context) {
    final snap = ref.watch(sessionControllerProvider);
    if (snap.state != SessionState.noAuthKey) return const SizedBox.shrink();
    return Scaffold(
      appBar: AppBar(title: const Text('Heliolytics')),
      body: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            const SizedBox(height: 16),
            const Text('Paste your Helio auth key',
                style: TextStyle(fontSize: 18, fontWeight: FontWeight.w500)),
            const SizedBox(height: 4),
            const Text('32 hex characters', style: TextStyle(color: Colors.grey)),
            const SizedBox(height: 16),
            TextField(
              controller: _ctrl, maxLength: 32,
              autocorrect: false, enableSuggestions: false,
              decoration: InputDecoration(
                border: const OutlineInputBorder(),
                hintText: 'a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6',
                errorText: _error,
              ),
            ),
            const SizedBox(height: 16),
            FilledButton(
              onPressed: _saving ? null : _save,
              child: _saving
                  ? const SizedBox(width: 20, height: 20, child: CircularProgressIndicator(strokeWidth: 2))
                  : const Text('Save'),
            ),
            const SizedBox(height: 24),
            TextButton(
              onPressed: () => showDialog<void>(
                context: context,
                builder: (_) => const AlertDialog(
                  title: Text('Where do I find this?'),
                  content: Text(
                    'The auth key is a 32-char hex string associated with your ring. '
                    'It can be retrieved from Zepp, from your Helio app, or from '
                    "Zepp's /users/{userId}/devices API. Once you have it, paste it here "
                    'and it will be stored in the Android Keystore.',
                  ),
                ),
              ),
              child: const Text('Where do I find this?'),
            ),
          ],
        ),
      ),
    );
  }
}
```

- [ ] **Step 2: Create `lib/app.dart`**

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import 'package:heliolytics/ble/session_controller.dart';
import 'package:heliolytics/ble/session_state.dart';
import 'package:heliolytics/ui/auth_key_screen.dart';
import 'package:heliolytics/ui/home_screen.dart';

class HeliolyticsApp extends ConsumerWidget {
  const HeliolyticsApp({super.key});
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return MaterialApp(
      title: 'Heliolytics',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.deepPurple),
        useMaterial3: true,
      ),
      home: const _RootRouter(),
    );
  }
}

class _RootRouter extends ConsumerWidget {
  const _RootRouter();
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final snap = ref.watch(sessionControllerProvider);
    if (snap.state == SessionState.noAuthKey) return const AuthKeyScreen();
    return const HomeScreen();
  }
}
```

- [ ] **Step 3: Create placeholder `lib/ui/home_screen.dart`**

```dart
import 'package:flutter/material.dart';

class HomeScreen extends StatelessWidget {
  const HomeScreen({super.key});
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Heliolytics')),
      body: const Center(child: Text('HomeScreen — coming in Task 22')),
    );
  }
}
```

- [ ] **Step 4: Create placeholder `lib/ui/sessions_screen.dart`**

```dart
import 'package:flutter/material.dart';

class SessionsScreen extends StatelessWidget {
  const SessionsScreen({super.key});
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Sessions')),
      body: const Center(child: Text('SessionsScreen — coming in Task 23')),
    );
  }
}
```

- [ ] **Step 5: Overwrite `lib/main.dart`**

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

import 'package:heliolytics/app.dart';
import 'package:heliolytics/auth/auth_key_store.dart';
import 'package:heliolytics/ble/session_controller.dart';

class _SecureStore implements AuthKeyStore {
  final _s = const FlutterSecureStorage(
    aOptions: AndroidOptions(encryptedSharedPreferences: true),
  );
  @override Future<String?> read(String k) => _s.read(key: k);
  @override Future<void> write(String k, String v) => _s.write(key: k, value: v);
  @override Future<void> delete(String k) => _s.delete(key: k);
}

void main() {
  runApp(
    ProviderScope(
      overrides: [authKeyStoreProvider.overrideWithValue(_SecureStore())],
      child: const HeliolyticsApp(),
    ),
  );
}
```

- [ ] **Step 6: Verify and commit**

```bash
flutter analyze
git add lib/ui/auth_key_screen.dart lib/ui/home_screen.dart lib/ui/sessions_screen.dart lib/main.dart lib/app.dart
git commit -m "feat(ui): add AuthKeyScreen, app shell, and placeholder home/sessions"
```

---

### Task 22: HomeScreen with discovery summary

**Files:**
- Modify: `lib/ui/home_screen.dart`
- Create: `lib/ui/discovery_summary_card.dart`

- [ ] **Step 1: Create `lib/ui/discovery_summary_card.dart`**

```dart
import 'package:flutter/material.dart';
import 'package:heliolytics/data/models.dart';

class DiscoverySummaryCard extends StatelessWidget {
  final Session session;
  const DiscoverySummaryCard({super.key, required this.session});

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text('Last session — ${_fmt(session.startedAt)}',
                style: Theme.of(context).textTheme.titleSmall),
            if (session.deviceMac != null)
              Text('Device: ${session.deviceMac}',
                  style: Theme.of(context).textTheme.bodySmall),
            const SizedBox(height: 12),
            const Text('Chunked:', style: TextStyle(fontWeight: FontWeight.w600)),
            ...session.entries.map((e) => _EntryRow(entry: e)),
            const SizedBox(height: 12),
            const Text('Unsolicited / live:',
                style: TextStyle(fontWeight: FontWeight.w600)),
            ...session.unsolicited.map((u) => _UnsolicitedRow(entry: u)),
            if (session.entries.isEmpty && session.unsolicited.isEmpty)
              const Padding(
                padding: EdgeInsets.symmetric(vertical: 8),
                child: Text('No entries yet.'),
              ),
          ],
        ),
      ),
    );
  }

  String _fmt(DateTime dt) =>
      '${dt.year}-${dt.month.toString().padLeft(2, '0')}-${dt.day.toString().padLeft(2, '0')} '
      '${dt.hour.toString().padLeft(2, '0')}:${dt.minute.toString().padLeft(2, '0')}';
}

class _EntryRow extends StatelessWidget {
  final DumpEntry entry;
  const _EntryRow({required this.entry});
  @override
  Widget build(BuildContext context) {
    final detail = switch (entry.status) {
      DumpStatus.ok => '${entry.samples} samples · ${entry.bytes} bytes',
      DumpStatus.empty => '(empty)',
      DumpStatus.rejected => '(rejected: ${entry.errorByte})',
      DumpStatus.unknown => '(unknown — bytes saved)',
    };
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 2),
      child: Row(children: [
        SizedBox(width: 56, child: Text(entry.code)),
        Expanded(child: Text(detail)),
      ]),
    );
  }
}

class _UnsolicitedRow extends StatelessWidget {
  final UnsolicitedEntry entry;
  const _UnsolicitedRow({required this.entry});
  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 2),
      child: Row(children: [
        SizedBox(width: 56, child: Text(entry.code)),
        Expanded(child: Text('${entry.count} notifications')),
      ]),
    );
  }
}
```

- [ ] **Step 2: Overwrite `lib/ui/home_screen.dart`**

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import 'package:heliolytics/ble/session_controller.dart';
import 'package:heliolytics/ble/session_state.dart';
import 'package:heliolytics/ui/discovery_summary_card.dart';
import 'package:heliolytics/ui/sessions_screen.dart';

class HomeScreen extends ConsumerWidget {
  const HomeScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final snap = ref.watch(sessionControllerProvider);
    final last = snap.lastSession;
    return Scaffold(
      appBar: AppBar(
        title: const Text('Heliolytics'),
        actions: [
          IconButton(
            icon: const Icon(Icons.folder_open),
            onPressed: () => Navigator.of(context).push(
              MaterialPageRoute(builder: (_) => const SessionsScreen()),
            ),
          ),
        ],
      ),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [
          _StatePill(state: snap.state),
          const SizedBox(height: 16),
          FilledButton.icon(
            onPressed: snap.state == SessionState.idle
                ? () => ref.read(sessionControllerProvider.notifier).scan()
                : null,
            icon: const Icon(Icons.bluetooth_searching),
            label: const Text('Connect to ring'),
          ),
          const SizedBox(height: 16),
          if (last != null) DiscoverySummaryCard(session: last),
        ],
      ),
    );
  }
}

class _StatePill extends StatelessWidget {
  final SessionState state;
  const _StatePill({required this.state});
  @override
  Widget build(BuildContext context) {
    final color = (state == SessionState.connected ||
        state == SessionState.fetching ||
        state == SessionState.listening)
        ? Colors.green
        : state == SessionState.error ? Colors.red : Colors.grey;
    return Row(children: [
      Container(width: 10, height: 10, decoration: BoxDecoration(color: color, shape: BoxShape.circle)),
      const SizedBox(width: 8),
      Text(_label(state)),
    ]);
  }

  String _label(SessionState s) => switch (s) {
    SessionState.noAuthKey => 'No auth key',
    SessionState.idle => 'Idle',
    SessionState.scanning => 'Scanning…',
    SessionState.connecting => 'Connecting…',
    SessionState.authenticating => 'Authenticating…',
    SessionState.connected => 'Connected',
    SessionState.fetching => 'Fetching…',
    SessionState.listening => 'Listening…',
    SessionState.error => 'Error',
  };
}
```

- [ ] **Step 3: Verify and commit**

```bash
flutter analyze
git add lib/ui/home_screen.dart lib/ui/discovery_summary_card.dart
git commit -m "feat(ui): add HomeScreen with state pill and discovery summary"
```

---

### Task 23: SessionsScreen

**Files:**
- Modify: `lib/ui/sessions_screen.dart`

- [ ] **Step 1: Overwrite `lib/ui/sessions_screen.dart`**

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import 'package:heliolytics/ble/session_controller.dart';
import 'package:heliolytics/data/models.dart';
import 'package:heliolytics/data/session_store.dart';
import 'package:heliolytics/ui/discovery_summary_card.dart';

class SessionsScreen extends ConsumerWidget {
  const SessionsScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Scaffold(
      appBar: AppBar(title: const Text('Sessions')),
      body: FutureBuilder<SessionStore>(
        future: ref.read(sessionStoreProvider.future),
        builder: (context, snap) {
          if (!snap.hasData) return const Center(child: CircularProgressIndicator());
          return FutureBuilder<List<String>>(
            future: snap.data!.listSessions(),
            builder: (context, ids) {
              if (!ids.hasData) return const Center(child: CircularProgressIndicator());
              if (ids.data!.isEmpty) return const Center(child: Text('No sessions yet.'));
              return ListView.builder(
                itemCount: ids.data!.length,
                itemBuilder: (_, i) => _SessionTile(sessionId: ids.data![i], store: snap.data!),
              );
            },
          );
        },
      ),
    );
  }
}

class _SessionTile extends StatelessWidget {
  final String sessionId;
  final SessionStore store;
  const _SessionTile({required this.sessionId, required this.store});

  @override
  Widget build(BuildContext context) {
    return FutureBuilder<Session>(
      future: store.readSessionJson(sessionId),
      builder: (context, snap) {
        if (!snap.hasData) return const SizedBox.shrink();
        final s = snap.data!;
        return ListTile(
          title: Text('${s.startedAt.year}-${_pad(s.startedAt.month)}-${_pad(s.startedAt.day)} '
              '${_pad(s.startedAt.hour)}:${_pad(s.startedAt.minute)}'),
          subtitle: Text('${s.deviceMac ?? "?"}  ·  ${s.entries.length} types, ${s.unsolicited.length} unsolicited'),
          onTap: () => Navigator.of(context).push(
            MaterialPageRoute(builder: (_) => SessionDetailScreen(session: s)),
          ),
        );
      },
    );
  }

  String _pad(int n) => n.toString().padLeft(2, '0');
}

class SessionDetailScreen extends StatelessWidget {
  final Session session;
  const SessionDetailScreen({super.key, required this.session});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Session ${session.sessionId}')),
      body: ListView(padding: const EdgeInsets.all(16), children: [
        DiscoverySummaryCard(session: session),
      ]),
    );
  }
}
```

- [ ] **Step 2: Verify and commit**

```bash
flutter analyze
git add lib/ui/sessions_screen.dart
git commit -m "feat(ui): add SessionsScreen with list and detail view"
```

---

## Phase 7: Wire Real BLE

### Task 24: Wire Scanner to flutter_blue_plus

**Files:**
- Modify: `lib/ble/scanner.dart`

- [ ] **Step 1: Replace scanner.dart**

```dart
import 'package:flutter_blue_plus/flutter_blue_plus.dart';
import 'package:heliolytics/ble/ble_devices.dart';

class FbpScanner implements BleScanner {
  @override
  Stream<DiscoveredDevice> scan({Duration? timeout}) async* {
    await FlutterBluePlus.startScan(timeout: timeout);
    await for (final r in FlutterBluePlus.scanResults) {
      for (final s in r) {
        yield DiscoveredDevice(
          remoteId: s.device.remoteId.str,
          name: s.device.platformName,
          rssi: s.rssi,
        );
      }
    }
  }

  @override
  Future<void> stop() => FlutterBluePlus.stopScan();
}

BleScanner scannerProvider() => FbpScanner();
```

- [ ] **Step 2: Verify and commit**

```bash
flutter analyze
git add lib/ble/scanner.dart
git commit -m "feat(ble): wire scanner to flutter_blue_plus"
```

---

### Task 25: Wire Connector to flutter_blue_plus

**Files:**
- Modify: `lib/ble/connector.dart`

- [ ] **Step 1: Replace connector.dart**

```dart
import 'dart:async';
import 'dart:typed_data';

import 'package:flutter_blue_plus/flutter_blue_plus.dart';

import 'package:heliolytics/ble/ble_devices.dart';
import 'package:heliolytics/config/constants.dart';

class FbpGattConnection implements GattConnection {
  final BluetoothDevice _device;
  BluetoothCharacteristic? _writeChar;
  BluetoothCharacteristic? _readChar;
  final _incoming = StreamController<Uint8List>.broadcast();

  FbpGattConnection(this._device);

  @override
  Future<List<BleCharacteristic>> discoverCharacteristics() async {
    final services = await _device.discoverServices();
    final out = <BleCharacteristic>[];
    for (final svc in services) {
      for (final c in svc.characteristics) {
        out.add(BleCharacteristic(
          uuid: c.uuid.str,
          canNotify: c.properties.notify,
          canWriteWithoutResponse: c.properties.writeWithoutResponse,
        ));
        final lcid = c.uuid.str.toLowerCase();
        if (lcid == chunkedReadUUID) {
          _readChar = c;
          await c.setNotifyValue(true);
          c.lastValueStream.listen((bytes) {
            if (bytes.isNotEmpty) _incoming.add(Uint8List.fromList(bytes));
          });
        } else if (lcid == chunkedWriteUUID) {
          _writeChar = c;
        }
      }
    }
    if (_writeChar == null || _readChar == null) {
      throw StateError('Ring missing chunked chars (write=$_writeChar, read=$_readChar)');
    }
    return out;
  }

  @override
  Future<void> writeChunked(Uint8List bytes) async {
    final c = _writeChar;
    if (c == null) throw StateError('Write characteristic not initialized');
    await c.write(bytes, withoutResponse: false);
  }

  @override
  Stream<Uint8List> get incoming => _incoming.stream;

  @override
  Future<void> dispose() async {
    await _incoming.close();
    await _device.disconnect();
  }
}

class FbpConnector implements BleConnector {
  @override
  Future<GattConnection> connect(String remoteId) async {
    final device = BluetoothDevice.fromId(remoteId);
    await device.connect(timeout: const Duration(seconds: 15));
    final conn = FbpGattConnection(device);
    await conn.discoverCharacteristics();
    return conn;
  }
}

BleConnector connectorProvider() => FbpConnector();
```

- [ ] **Step 2: Verify and commit**

```bash
flutter analyze
git add lib/ble/connector.dart
git commit -m "feat(ble): wire connector to flutter_blue_plus"
```

---

### Task 26: Wire SessionController ECDH auth to real GATT

**Files:**
- Modify: `lib/ble/session_controller.dart`

- [ ] **Step 1: Replace `_runEcdhAuth` in session_controller.dart**

```dart
  Future<void> _runEcdhAuth() async {
    final kp = EcdhAuth.generateKeypair();
    final authKey = await _authStorage.readBytes();
    if (authKey == null) throw StateError('No auth key');
    final authPayload = EcdhAuth.buildAuthPayload(kp.publicKey);
    await _gatt!.writeChunked(authPayload);

    // Wait for ring's response: 16 random + 48 public
    final response = await _gatt!.incoming
        .firstWhere((b) => b.length >= 3 + 16 + 48)
        .timeout(const Duration(seconds: 10), onTimeout: () => throw TimeoutException('Auth response'));
    final random = Uint8List.sublistView(response, 3, 3 + 16);
    final remotePub = Uint8List.sublistView(response, 3 + 16, 3 + 16 + 48);

    final sessionKey = EcdhAuth.deriveSessionKey(
      privateKey: kp.privateKey,
      remotePublicKey: remotePub,
      authKey: authKey,
    );

    final challenge = EcdhAuth.buildChallengeResponse(
      authKey: authKey,
      sessionKey: sessionKey,
      challenge: random,
    );
    await _gatt!.writeChunked(challenge);

    final status = await _gatt!.incoming
        .firstWhere((b) => b.isNotEmpty && b[0] == 0x05)
        .timeout(const Duration(seconds: 10), onTimeout: () => throw TimeoutException('Auth status'));
    if (status.length < 2 || status[1] != 0x01) {
      throw StateError('Auth rejected: ${status.toList()}');
    }
  }
```

- [ ] **Step 2: Verify and commit**

```bash
flutter analyze
git add lib/ble/session_controller.dart
git commit -m "feat(ble): wire SessionController ECDH auth to real GATT"
```

---

## Phase 8: DataRequester (Fetch Loop)

### Task 27: DataRequester + wire startFetch

**Files:**
- Create: `lib/ble/data_requester.dart`
- Modify: `lib/ble/session_controller.dart`

- [ ] **Step 1: Create `lib/ble/data_requester.dart`**

```dart
import 'dart:async';
import 'dart:typed_data';

import 'package:heliolytics/ble/ble_devices.dart';
import 'package:heliolytics/ble/chunked_protocol.dart';
import 'package:heliolytics/ble/parsers/activity.dart';
import 'package:heliolytics/ble/parsers/hrv.dart';
import 'package:heliolytics/ble/parsers/unknown.dart';
import 'package:heliolytics/config/constants.dart';
import 'package:heliolytics/data/models.dart';
import 'package:heliolytics/data/session_store.dart';
import 'package:heliolytics/utils/huami_time.dart';

class DataRequester {
  final GattConnection _gatt;
  final SessionStore _store;
  final String _sessionId;
  DataRequester(this._gatt, this._store, this._sessionId);

  Future<DumpEntry> fetchType(String typeCode, DateTime since) async {
    final startCmd = Uint8List.fromList([0x01, _hexToInt(typeCode), ...HuamiTime.fromDateTime(since)]);
    await _gatt.writeChunked(startCmd);

    final headResp = await _gatt.incoming
        .firstWhere((b) => b.isNotEmpty)
        .timeout(Duration(seconds: chunkReceiveTimeoutSec));

    if (headResp.length < 4 || headResp[0] != 0x01 || headResp[1] != _hexToInt(typeCode)) {
      return DumpEntry(code: typeCode, status: DumpStatus.unknown, samples: 0, bytes: 0);
    }
    final status = headResp[2];
    if (status == 0x02 || status == 0x04) {
      return DumpEntry(
        code: typeCode, status: DumpStatus.rejected, samples: 0, bytes: 0,
        errorByte: '0x${status.toRadixString(16).padLeft(2, '0')}',
      );
    }
    if (status != 0x01) {
      return DumpEntry(code: typeCode, status: DumpStatus.unknown, samples: 0, bytes: 0);
    }
    final expectedCount = (headResp[3] << 8) | headResp[4];
    if (expectedCount == 0) {
      return DumpEntry(code: typeCode, status: DumpStatus.empty, samples: 0, bytes: 0);
    }

    await _gatt.writeChunked(Uint8List.fromList([0x02]));

    final assembler = ChunkAssembler();
    for (var attempt = 0; attempt <= chunkRetryCount; attempt++) {
      try {
        await _gatt.incoming
            .where((b) => b.isNotEmpty && b[0] == 0x03)
            .take(expectedCount)
            .timeout(Duration(seconds: chunkReceiveTimeoutSec * 2))
            .forEach(assembler.append);
        break;
      } catch (_) {
        if (attempt == chunkRetryCount) rethrow;
      }
    }

    await _gatt.writeChunked(Uint8List.fromList([0x03, 0x09]));

    final payload = assembler.payload;
    await _store.appendBytes(_sessionId, typeCode, payload);
    final samples = _parse(typeCode, payload);
    return DumpEntry(
      code: typeCode, status: DumpStatus.ok, samples: samples, bytes: payload.length,
      file: '${_binName(typeCode)}.bin',
    );
  }

  int _parse(String typeCode, Uint8List bytes) {
    try {
      switch (typeCode) {
        case '0x01': return ActivityParser.parse(bytes).length;
        case '0x49': return HrvParser.parse(bytes).length;
        default: UnknownParser.parse(typeCode, bytes); return 0;
      }
    } catch (_) { return 0; }
  }

  int _hexToInt(String hex) {
    final s = hex.startsWith('0x') ? hex.substring(2) : hex;
    return int.parse(s, radix: 16);
  }

  String _binName(String code) {
    const names = {
      '0x01': '0x01_activity', '0x05': '0x05_workout',
      '0x13': '0x13_stress', '0x25': '0x25_spo2',
      '0x2E': '0x2E_temperature', '0x38': '0x38_sleep_resp_rate',
      '0x3A': '0x3A_resting_hr', '0x3D': '0x3D_max_hr',
      '0x48': '0x48_sleep_session', '0x49': '0x49_hrv',
    };
    return names[code] ?? 'unknown_$code';
  }
}
```

- [ ] **Step 2: Replace `startFetch` in session_controller.dart**

```dart
  Future<void> startFetch({
    required List<String> typeCodes,
    required int fetchWindowHours,
    required int listenDurationSec,
  }) async {
    if (state.state != SessionState.connected) return;
    final store = _store;
    if (store == null) return;
    state = state.copyWith(state: SessionState.fetching);

    final sessionId = await store.createSession(
      deviceMac: null,
      fetchWindowHours: fetchWindowHours,
      listenDurationSec: listenDurationSec,
      mode: SessionMode.fetchAndListen,
    );
    final since = DateTime.now().toUtc().subtract(Duration(hours: fetchWindowHours));

    final entries = <DumpEntry>[];
    for (final code in typeCodes) {
      state = state.copyWith(currentTypeCode: code);
      try {
        final entry = await DataRequester(_gatt!, store, sessionId).fetchType(code, since);
        entries.add(entry);
      } catch (_) {
        entries.add(DumpEntry(code: code, status: DumpStatus.unknown, samples: 0, bytes: 0));
      }
    }
    await store.writeCatalogJson(SessionCatalog(
      sessionId: sessionId, chunked: entries, unsolicited: const [],
    ));

    state = state.copyWith(
      state: SessionState.idle, currentTypeCode: null,
      lastSession: await store.readSessionJson(sessionId),
    );
  }
```

- [ ] **Step 3: Verify and commit**

```bash
flutter analyze
git add lib/ble/data_requester.dart lib/ble/session_controller.dart
git commit -m "feat(ble): add DataRequester and wire startFetch to chunked loop"
```

---

## Phase 9: Manual Smoke Test

### Task 28: Build, install, and run on a real device

This is a manual end-to-end check on a real Android phone with the Helio strap on the wrist. There is no automated test.

- [ ] **Step 1: Build the debug APK**

```bash
flutter build apk --debug
```

Expected: BUILD SUCCESSFUL.

- [ ] **Step 2: Install on a phone with USB debugging**

```bash
flutter devices
flutter install
```

- [ ] **Step 3: Paste auth key, observe Idle state**

Open Heliolytics, paste your 32-char auth key, tap Save. Expected: app navigates to HomeScreen with state pill "Idle".

- [ ] **Step 4: Connect**

Tap "Connect to ring". Expected state pill cycle: Scanning → Connecting → Authenticating → Connected.

If the auth fails (state → Error), the issue is almost certainly in `_runEcdhAuth`. The placeholder `_sharedSecret` may need to be replaced with proper sect163k1 curve math (see Task 19 NOTE).

- [ ] **Step 5: Run a fetch**

For the smoke test, temporarily add a "Fetch" button to HomeScreen. In `lib/ui/home_screen.dart`, change the Connect button to also call `startFetch`:

```dart
onPressed: snap.state == SessionState.idle
    ? () async {
        await ref.read(sessionControllerProvider.notifier).scan();
        if (ref.read(sessionControllerProvider).state == SessionState.connected) {
          await ref.read(sessionControllerProvider.notifier).startFetch(
            typeCodes: knownTypeCodes,
            fetchWindowHours: 48,
            listenDurationSec: 300,
          );
        }
      }
    : null,
```

Hot reload. Tap the button.

Expected: discovery summary card populates with non-zero entries for at least `0x01` Activity and `0x49` HRV.

- [ ] **Step 6: Inspect a session**

Tap the folder icon → Sessions screen → tap the new session → detail view shows the discovery summary.

Raw bytes are in `/data/data/com.heliolytics.heliolytics/files/heliolytics/sessions/<id>/`. Inspect with:

```bash
adb shell run-as com.heliolytics.heliolytics cat files/heliolytics/sessions/<id>/0x01_activity.bin | xxd | head
```

- [ ] **Step 7: Clean up the smoke-test button**

Remove the temporary fetch from HomeScreen; replace with a proper "Fetch" button enabled only in connected state. Commit:

```bash
git add -A
git commit -m "chore: post-smoke-test cleanups"
```

---

## End of Plan

When all 28 tasks are complete and the smoke test passes:
- The discovery summary is your catalog of what the strap actually returns.
- Use it to drive the next spec (the real app + Go backend).
- If new type codes appear, add a parser to `lib/ble/parsers/` and you're done.
- Derived metrics (BioCharge, Exertion, VO2 Max, PAI, etc.) are out of scope here — they belong in the future server-side analytics spec.


