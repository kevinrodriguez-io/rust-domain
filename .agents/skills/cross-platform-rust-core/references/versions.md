# Pinned versions

All versions verified against primary sources on **2026-09-19**. Pin exactly. Several components in this stack ship patch releases weekly.

## Rust core and UniFFI

| Component | Pin | Notes |
|---|---|---|
| Rust toolchain | `1.98.1` | Stable, released 2026-09-03 |
| Rust MSRV floor | `1.90.0` | Set by UniFFI 0.32.x. This is the binding floor for the whole stack |
| `uniffi` | `=0.32.1` | Released 2026-09-09. Use `=` — not `^` |
| `uniffi_bindgen` | `=0.32.1` | Must equal `uniffi` exactly |
| bindgen binary | built in-workspace | `fn main() { uniffi::uniffi_bindgen_main() }` |

`0.32.1` is the minimum for Android: `0.32.0` had a checksum failure on aarch64, and `0.31.2` fixed JNA signedness bugs on ARM32. Do not go below `0.32.1`.

## Android

| Component | Pin | Notes |
|---|---|---|
| Kotlin | `2.4.20` | Released 2026-09-07 |
| AGP | `9.4.1` | Requires Gradle ≥ 9.6.0, JDK 17, Build Tools 36.0.0 |
| Gradle | `9.7.1` | Released 2026-08-19 |
| JDK | `17` minimum | 21 works |
| Compose BOM | `2026.09.00` | → Compose UI/runtime/foundation 1.12.1, material3 1.4.0 |
| Compose compiler | Kotlin version (`2.4.20`) | Plugin `org.jetbrains.kotlin.plugin.compose` |
| `compileSdk` / `targetSdk` | `36` minimum | Play has required 36 since 2026-08-31. 37 is available |
| `minSdk` | `24` | Engineering judgment, not a Google requirement. Avoids the D8/R8 interface-method limitation below 24 |
| SDK Build Tools | `36.0.0` | AGP 9.4 minimum and default |
| NDK | `28.2.13676358` (r28c) | Pin explicitly. This is the AGP 9.4 default — do **not** use newest (r30) |
| `cargo-ndk` | `4.1.2` | MSRV 1.86 |
| JNA | `5.19.1` | Use the `@aar` classifier on Android. UniFFI requires ≥ 5.12.0 |
| `kotlinx-coroutines-core` | `1.11.0` | UniFFI requires ≥ 1.6 |

**Do not reference `androidx.compose.compiler:compiler`.** That artifact is frozen at 1.5.15 (last published 2024-08-07). Since Kotlin 2.0 the Compose compiler ships with Kotlin.

## Apple

| Component | Pin | Notes |
|---|---|---|
| Xcode | `27` | **Requires macOS Tahoe 26.6+.** Ships Swift 6.4 and the iOS 27 SDK |
| Swift | `6.4.0` | Released 2026-09-15 |
| iOS deployment target | `15` minimum supported by Xcode 27 | Choose based on product needs |
| macOS deployment target | `12` minimum | Floor rose from 11 in Xcode 27 |
| watchOS deployment target | `9` minimum | Floor rose from 8 |
| `cargo-swift` (optional) | `0.11.1` | Only if you want a turnkey Swift Package |

**On-device debugging requires iOS 17+** with Xcode 27, up from iOS 15. You can still ship to iOS 15, but older test devices cannot be debugged.

## Node service

| Component | Pin | Notes |
|---|---|---|
| Node | `24.x` (Active LTS) | Latest 24.21.0. **Node 20 is EOL; 22 is Maintenance LTS only** |
| `napi` (Rust crate) | `3.12.6` | MSRV 1.88 |
| `napi-derive` | `3.6.7` | |
| `napi-build` | `2.4.3` | |
| `@napi-rs/cli` | `3.10.4` | Install from `latest`. The `alpha`/`beta`/`canary` tags are years stale |
| `bullmq` | `6.3.8` | v6 line. See queue-bullmq.md for v5→v6 breaking changes |
| `ioredis` | `6.0.0` | **Install explicitly** — optional peer dependency in BullMQ v6 |
| Redis server | `6.2.0` minimum | Must set `maxmemory-policy=noeviction` |
| `@parse/node-apn` | `8.1.0` | APNs client. Supports Node 20/22/24 |
| `firebase-admin` | `14.4.0` | FCM client. **Requires Node ≥ 22** — this is what sets the Node floor |

**Never use the bare `node-apn` package.** It is abandoned at 3.0.0 from November 2020 and still declares Node 4.6 support. The maintained fork is `@parse/node-apn`.

## Deliberately excluded

Do not add these without an explicit request. Each one drags the stack backwards.

| Excluded | Why |
|---|---|
| WebAssembly / browser target | UniFFI ships no first-party WASM generator. Requires third-party tooling pinned to UniFFI 0.31.0, or a hand-written `wasm-bindgen` facade |
| Kotlin Multiplatform / Gobley | Gobley pins `uniffi =0.29.5` and sits on Kotlin 2.1.10 / AGP 8.7.3. Incompatible with AGP 9 |
| Third-party UniFFI generators | Go, C#, Dart, Java, C++, React Native all track UniFFI 0.31.x or older |
| `@taskforcesh/bullmq-pro` | Commercial licence. See queue-bullmq.md for the OSS design |
| `cross` (for cross-compilation) | Last release 0.2.5 from 2023-02-04 despite active commits. Prefer `cargo-zigbuild` 0.23.4, or install cross from git |

## Verification sources

When a pin needs rechecking:

- Rust: `https://static.rust-lang.org/dist/channel-rust-stable.toml`
- Crates: `https://crates.io/api/v1/crates/<name>`
- npm: `https://registry.npmjs.org/<name>`
- Android artifacts: `https://dl.google.com/android/maven2/<group-path>/maven-metadata.xml`
- Kotlin / JVM artifacts: `https://repo1.maven.org/maven2/<group-path>/maven-metadata.xml`
- Xcode and Apple SDKs: `https://developer.apple.com/xcode/system-requirements/`
- NDK: `https://github.com/android/ndk/releases`
- Node release status: `https://nodejs.org/dist/index.json` and `https://github.com/nodejs/Release/blob/main/schedule.json`
