# Tether Crypto Wire Format — byte-level interop spec

**Owner:** Epic #5 (E2E C1 crypto)
**Status:** v0.1 lock — any change requires `keyVersion` bump.
**Cross-language:** Go (`internal/crypto/`) is canonical; Rust (Tauri Mobile) MUST mirror byte-for-byte.

This spec freezes the cryptographic wire formats for v0.1 so the Go CLI/daemon and the Rust Mobile App can decrypt each other's envelopes without coordination beyond the public spec. Companion to spec §11.C / §11.M / §11.J. Source of truth: `tether/internal/crypto/`.

## 1. Algorithms

| Layer | Choice | Crate / package |
|---|---|---|
| ECDH | X25519 (RFC 7748, curve25519, 32-byte keys, little-endian scalar/point) | Go `golang.org/x/crypto/curve25519` ⇆ Rust `x25519-dalek` |
| KDF | HKDF-SHA256 | Go `golang.org/x/crypto/hkdf` ⇆ Rust `hkdf` + `sha2` |
| AEAD | XChaCha20-Poly1305 (24-byte random nonce, 16-byte tag) | Go `golang.org/x/crypto/chacha20poly1305` ⇆ Rust `chacha20poly1305::XChaCha20Poly1305` |
| SAS | HKDF-SHA256 to 4 bytes → big-endian u32 → mod 1_000_000 → 6-digit decimal | both sides match |
| Replay | LRU dedup (30000 entries) + 5-minute skew window | both sides MUST use same window |

## 2. HKDF info strings (frozen — DO NOT rename without `keyVersion` bump)

| Constant | Use |
|---|---|
| `tether-pair-v1` | Pairing-time wrap_key derivation |
| `tether-session-wrap-v1` | session-key wrap_key from shared secret |
| `tether-session-key-v1` | per-session encryption key derivation |
| `tether-pair-sas` | SAS (short auth string) derivation during pairing |
| `tether-session-persist-v1` | on-disk keys.bin AEAD key derivation |

## 3. Envelope wire format

Envelopes are fixed-layout big-endian, length-prefixed:

```
[1B  wireVersion=1]
[2B  sessionIdLen][sessionId UTF-8]
[2B  fromIdLen][fromId UTF-8]
[2B  toIdLen][toId UTF-8]
[4B  keyVersion BE]
[24B nonce]
[4B  ctLen BE][ciphertext...]
```

**Identifier length bounds:** each `*Len` field is 2-byte big-endian, so each identifier (sessionId / fromId / toId) is bounded at **65535 bytes** of UTF-8. Encoder MUST reject longer values with an explicit error; decoder MUST reject malformed length frames (truncated input, length pointing past end of buffer). Go reference: `internal/crypto/xchacha20poly1305.go` `Marshal` / `UnmarshalEnvelope` enforce both.

`ciphertext = XChaCha20Poly1305(session_key).Seal(nonce, plaintext, AD)` where AD is computed (NOT transmitted) as:

```
AD = [2B BE]sessionId || [2B BE]fromId || [2B BE]toId || [4B BE]keyVersion
```

**Why length-prefix the AD instead of plain concat:** without the prefix, an attacker could shift bytes between adjacent ID fields and still produce a valid AD. The 2-byte big-endian length frame closes that aliasing.

## 4. keys.bin persistence format (CLI/daemon-side)

CLI / daemon stores wrapped session keys at `~/.tether/users/<user>/keys.bin`. Mobile uses platform Keystore (iOS Keychain / Android Keystore via tauri-plugin-stronghold for v0.1). The keys.bin format is **CLI-only**, but if a future "shared keys.bin" feature lands the Rust side will need to mirror this exact layout — so it's frozen here.

```
[1B  formatVersion=1]
[16B salt]
[24B nonce]
[4B  keyVersion BE]
[4B  ctLen BE][ciphertext]
```

`ciphertext = XChaCha20Poly1305(persistKey).Seal(nonce, sessionKeyBytes, AD)` where:

```
persistKey = HKDF-SHA256(wrap_key, salt, info="tether-session-persist-v1", L=32)
AD         = "tether-keysbin-v1"  (literal UTF-8, no length prefix)
```

The salt is per-file random (16 bytes) and stored alongside; rotates on every write.

## 5. Pairing

### 5.1 X25519 keypair representation

Both sides use raw 32-byte little-endian curve points. Go: `curve25519.X25519` / Rust: `x25519_dalek::PublicKey::from(secret).to_bytes()`. Public keys are identical bit-for-bit.

### 5.2 Fingerprint (QR / "out of band" check)

```
sha = SHA256(pubkey_32bytes)
fingerprint = hex(sha[0..4]) + ":" + hex(sha[28..32])
```

Example: `1a2b3c4d:e5f6a7b8` (17 ASCII chars including colon). Rust must produce identical strings — case-sensitive lowercase hex.

### 5.3 SAS (Short Auth String) — server-fallback path

```
sas_bytes = HKDF-SHA256(shared_secret, salt=nil, info="tether-pair-sas", L=4)
sas       = format("%06d", be_u32(sas_bytes) % 1_000_000)
```

Always 6 decimal digits, leading zeros preserved. Both peers must derive the same value to confirm pairing without a QR scan.

### 5.4 Pairing transport metadata

The QR payload `{cli_pubkey, fingerprint, server_url, pairing_token, server_pubkey_pin}` is the QR-encoder's concern (not crypto-core). This spec only freezes the cryptographic primitives (ECDH + fingerprint + SAS). QR encode/parse + auth-token issuance + server-mediated pairing fallback live in a future `internal/pairing/` package atop these primitives.

## 6. Replay defense

The crypto layer exposes a replay guard that the transport layer feeds with caller-supplied envelope IDs (transport-layer UUIDs, NOT the AEAD nonce — decoupled so upper layers can pick their own ID-uniqueness scheme):

- LRU capacity: 30000 entries
- Time-skew window: 5 minutes either direction from local clock
- Eviction: time-based GC; entries older than the window drop on next sweep
- Concurrency: guarded by mutex; safe for transport goroutine + supervisor goroutine

Both Go and Rust sides MUST use identical (capacity, window) values to avoid desync under clock drift.

**Transport-layer fields NOT in the inner envelope.** Two related fields named in master spec §11.C live at the **outer frame** (transport layer) and are NOT part of the `Envelope` struct in `internal/crypto/`:

- **`envelopeId`** — a transport-layer UUID; replay guard consumes it via `ReplayGuard.Check(id, ts)`. Wire shape and uniqueness scheme are owned by the transport (e.g. `internal/wire/`); the crypto layer is agnostic.
- **`ts`** (timestamp, server-visible metadata in §11.C) — also transport-layer, NOT serialized inside the §3 envelope wire format. The replay guard takes `ts` as a parameter for skew-window comparison; it does not embed `ts` in any AEAD-protected bytes.

Rust implementers should treat envelope-id and ts as part of the OUTER protocol frame (alongside routing metadata), not as fields to add to §3's envelope layout.

## 7. Versioning policy

There are **three independent version fields** in this layer; conflating them is the most common Rust↔Go interop hazard. Read this whole section before changing any version number.

### 7.1 `wireVersion` (envelope §3) — currently `1`

Identifies the **envelope wire layout**. v0.1 is `wireVersion=1`. Bumps only on a structural change to §3 (e.g. adding/removing a length-prefixed field, changing the nonce size). On bump:

1. Old peers reject newer wire versions outright.
2. New peers MAY accept old wire formats via a compat reader during a deprecation window.

### 7.2 `formatVersion` (keys.bin §4) — currently `1`

Identifies the **keys.bin on-disk layout** (CLI/daemon-side persistence). Independent from `wireVersion` — keys.bin can rev without affecting the envelope wire and vice versa.

### 7.3 Envelope `keyVersion` (envelope §3 + AD) — **fixed at `1` for v0.1**

Master spec §11.C locks: **`keyVersion = 1` is constant for all v0.1 envelopes.** It bumps **only on a cryptographic algorithm or KDF context change** (e.g. swapping XChaCha20-Poly1305 for AES-GCM-SIV, changing an HKDF info string). It does **NOT** bump on `session_key` rotation.

Why: rotating a session_key already invalidates prior envelopes (the new key cannot decrypt them); there is no on-the-wire signal of rotation, and Rust implementers must NOT increment envelope `keyVersion` on rotate or interop will break against Go.

If you find yourself wanting to "tag" an envelope with which session_key version produced it, that's a routing concern at the transport layer, not a crypto-core invariant — handle it there, not by mutating envelope `keyVersion`.

### 7.4 keys.bin internal `keyVersion` (§4) — local-only per-rotation counter

The `keyVersion` field inside the keys.bin format (§4) **is** a per-rotation counter — it increments every time `SessionKey.Rotate()` writes to disk, so the daemon can validate "did this key load the latest material on restart?" against in-memory state.

**This is local bookkeeping. It is NEVER on the wire** and Rust implementations of "shared keys.bin" (if/when that lands) MUST mirror Go's local-counter semantics rather than treating it as a wire-level identifier.

### 7.5 Quick reference

| Field | Where | Role | When to bump |
|---|---|---|---|
| `wireVersion` | envelope §3 | envelope wire layout | structural change to §3 |
| `formatVersion` | keys.bin §4 | keys.bin on-disk layout | structural change to §4 |
| envelope `keyVersion` | envelope §3 + AD | **algorithm / KDF context** | swap AEAD or HKDF info string. **NOT** on session_key rotation. v0.1 = `1` constant. |
| keys.bin internal `keyVersion` | keys.bin §4 | local per-rotation counter | every `SessionKey.Rotate()`. Never on the wire. |

## 8. Out of scope (explicit)

Not part of this spec, defer to other tickets:

- **`session_key` transport at session start** — a control envelope carrying a wrap_key-encrypted session_key; the AEAD primitive in §3 covers it but the routing (channel selection, retry, in-band vs out-of-band) is upper-layer concern.
- **Dev backdoor (`TETHER_DEV_DECRYPT=1`, build-tag `tether_dev`)** — spec §11.C requires it; lives in daemon binary entrypoint with build-tag-controlled logger attach, not in crypto core. Tracked as a daemon Epic #3 follow-up.
- **CI grep guard** for `tether_dev` build-tag symbol leak in release builds — release-pipeline ticket.
- **Hardware-backed keys** (TPM, Secure Enclave) — v0.2.
