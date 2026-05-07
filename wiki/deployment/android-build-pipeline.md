# Android build pipeline (B1)

Spec D-13 mandates v0.1 ships an Android Tauri Mobile target alongside Desktop.
This doc captures the verified build pipeline so a fresh dev box can
`cargo check --target=aarch64-linux-android --lib` clean without trial-
and-error on NDK / cargo / cc-rs interactions.

## Status

Verified 2026-05-07 against:
- NDK r26d
- Rust 1.90 stable + `aarch64-linux-android`, `armv7-linux-androideabi`,
  `x86_64-linux-android` targets
- tether-app at commit on `main` past PR #22 (cross-stack pair fixes)

All three targets `cargo check --lib` clean. Full deps tree compiles —
ring, quinn, web-transport-quinn, chacha20poly1305, stronghold,
barcode-scanner, deep-link, fs, full Tauri 2.x plugin set.

Runtime smoke (emulator / physical device) is **not part of this doc** —
that needs an Android Studio AVD or USB-tethered phone, neither of which
fits the headless-CI scope.

## Required tools

| Tool | Min version | Why |
|---|---|---|
| Android NDK | r26 | provides per-target clang + llvm-ar; r26 is the LTS line. r25 also works but missing some C++23 headers ring may want post-1.x. |
| Rust toolchain | stable + 3 targets | `aarch64-linux-android` is mandatory (modern phones); `armv7-linux-androideabi` for older 32-bit phones; `x86_64-linux-android` for emulator. |
| Cargo | 1.74+ | `[env]` table inheritance, `linker = "<binary-on-PATH>"` resolution. |

## Setup (one-time per dev box)

```bash
# 1. Install NDK
curl -L -o /tmp/ndk.zip \
  https://dl.google.com/android/repository/android-ndk-r26d-linux.zip
sudo unzip -q -d /opt /tmp/ndk.zip
export ANDROID_NDK_HOME=/opt/android-ndk-r26d
# (persist via ~/.bashrc or ~/.zshrc)

# 2. Install Rust targets
rustup target add aarch64-linux-android armv7-linux-androideabi x86_64-linux-android

# 3. (per-shell) Source the env helper
cd <repo>/tether-app
source scripts/android-env.sh
```

`scripts/android-env.sh` exports:
- `ANDROID_NDK_TOOLCHAIN_BIN` — path to `$NDK/toolchains/llvm/prebuilt/<host>/bin`
- `CC_<triple>` and `AR_<triple>` for all 4 Android targets — required by
  `cc-rs` (which ring's build.rs uses) and which cargo's `linker = ...`
  config does NOT propagate
- prepends `ANDROID_NDK_TOOLCHAIN_BIN` to `PATH` so `src-tauri/.cargo/config.toml`'s
  bare `linker = "aarch64-linux-android21-clang"` resolves

## Build commands

After `source scripts/android-env.sh`:

```bash
cd <repo>/tether-app/src-tauri

# Lint / type-check (fast — ~30-60s after first run cached)
cargo check --target=aarch64-linux-android --lib
cargo check --target=x86_64-linux-android --lib       # emulator
cargo check --target=armv7-linux-androideabi --lib    # legacy phones

# Real build (slower; produces .so for tauri-cli to bundle into APK)
cargo build --target=aarch64-linux-android --release --lib

# Tauri full app (requires Android Studio for gradle + signing)
cargo install tauri-cli --version "^2.0"
cargo tauri android init        # one-time scaffold (writes gen/android/)
cargo tauri android dev         # debug build to attached device / emulator
cargo tauri android build       # release APK / AAB
```

## minSdk / targetSdk

`scripts/android-env.sh` pins `*-android21-clang` (API 21 = Android 5.0
Lollipop, 2014). This matches Tauri 2 Mobile's documented floor. To bump:

1. Edit `scripts/android-env.sh` — change all `21` suffixes
2. Edit `gen/android/app/build.gradle.kts` — `minSdk = <new>` (after `cargo tauri android init`)
3. Verify NDK ships clang for that level (`ls $NDK/toolchains/llvm/prebuilt/*/bin/aarch64-linux-android*-clang`)

Tauri 2's `targetSdk` is currently 34 (Android 14); cargo-mobile2 manages
that side.

## Known landmines

1. **`cargo check` works without `cargo tauri android init`, BUT real APK build needs the gradle scaffold.** The init step generates `gen/android/`, AndroidManifest, signing config, etc. Don't commit `gen/android/` (it's per-machine + has machine-local paths).

2. **`ring` crate's build.rs reads `CC_<triple>` env vars, NOT cargo's `[target.<triple>] linker = ...` config.** This is why `scripts/android-env.sh` exports both — cargo needs the linker, cc-rs needs CC_*.

3. **Stronghold KDF `compile_error!` gate** — the Tauri lib refuses release builds without `--features insecure-kdf` until the SipHash-2-4 placeholder gets replaced with Argon2id. See `src-tauri/src/lib.rs::compile_error!` block. **Do NOT ship a v0.1 release APK until this is fixed.** For dev / smoke builds use `cargo build --features insecure-kdf` if needed.

4. **`web-transport-quinn` deps quinn / rustls / ring.** All three cross-compile fine on Android with the env above. If a future quinn / rustls bump introduces an OS-specific socket extension or KEM that requires a feature gate change, this doc + scripts/android-env.sh may need updating. Re-run `cargo check --target=aarch64-linux-android --lib` after any quinn/rustls major bump.

5. **`tauri-plugin-stronghold` and `tauri-plugin-barcode-scanner`** are `#[cfg(mobile)]`-gated upstream — desktop builds skip them entirely; the Android build pulls them in. Both compile clean on aarch64.

6. **iOS is NOT in v0.1.** Spec D-13 explicitly defers iOS to v0.1.x. Do not attempt `cargo tauri ios *` from this doc — different toolchain, different runbook.

## CI integration (recommended)

Add to GitHub Actions / equivalent:

```yaml
- name: Android cargo check (aarch64)
  env:
    ANDROID_NDK_HOME: /opt/android-ndk-r26d
  run: |
    rustup target add aarch64-linux-android
    source scripts/android-env.sh
    cd src-tauri
    cargo check --target=aarch64-linux-android --lib
```

Repeat for the other two targets if you want to gate against them.
`cargo build` (vs `check`) is much slower (~10x) — only run on release branches.

## Runtime smoke

Out of scope for this doc. Operator runs:

```bash
cd <repo>/tether-app
cargo tauri android init          # one-time, writes gen/android/
cargo tauri android dev           # debug build to attached device / AVD
```

If smoke fails:
1. `adb logcat | grep tether` — Rust panic / Tauri JS errors
2. AVD must be API 21+ aarch64 (or x86_64 + matching target)
3. Daemon must be reachable from device — for AVD, daemon URL `https://10.0.2.2:4444` (the AVD's loopback alias for the host). For real device on local network, use the host's LAN IP.
4. The dev cert SHA256 fingerprint pin must match — see `tether daemon -v`'s startup log line.

## Files

- `tether-app/scripts/android-env.sh` — sourced before any cargo command
- `tether-app/src-tauri/.cargo/config.toml` — per-target linker config (uses bare names; resolved via PATH from android-env.sh)
- This doc — `wiki/deployment/android-build-pipeline.md`

Refs: spec D-13 (mobile target), §11.V (transport implementation), §10 PoC-2.6 (Android smoke).
