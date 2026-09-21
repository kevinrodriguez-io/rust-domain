# Extending an existing project

Verified 2026-09-19. **Detect before you change.** Several widely-copied patterns in this stack are now deprecated or removed, and most fail quietly — at runtime, on one architecture, or only on a real device.

## Detection checklist

Run through this before editing anything. Each row names the predecessor you are likely to find.

### UniFFI

| Check | Finding | Action |
|---|---|---|
| `uniffi` version in `Cargo.toml` | ≤ 0.29 | `UniffiCustomTypeConverter` and `typedef extern` still in use. Two migrations needed |
| | 0.30.x | UDL trait interfaces need `#[uniffi::trait_interface]`; Kotlin `NoPointer` → `NoHandle` |
| | 0.31.x | `--config` semantics and `[ByRef] bytes` → `&[u8]` changes pending; Kotlin call sites need direct `ByteBuffer` |
| | 0.32.0 | **Upgrade to 0.32.1** — 0.32.0 has a checksum failure on aarch64 |
| `grep -rn 'with_foreign'` | any hit | Deprecated in 0.32.0 → `#[uniffi::export(rust, foreign)]` |
| `grep -rn 'UniffiCustomTypeConverter'` | any hit | Removed in 0.29 → `uniffi::custom_type!` |
| `grep -rn -- '--library\|--lib-file'` | any hit in build scripts | `--library` inert since 0.31.0, `--lib-file` removed. Pass the library path directly |
| `uniffi` vs `uniffi_bindgen` versions | not identical | Pin both to the same exact version; the runtime guard will not catch this |
| bindgen source | a globally installed `uniffi-bindgen` binary | Move it into the workspace as a `[[bin]]` |
| `crate-type` | missing `staticlib` | Add it, or iOS linking fails later |

### Android

| Check | Finding | Action |
|---|---|---|
| AGP major version | 8.x | AGP 9 migration: remove `org.jetbrains.kotlin.android`, `kotlinOptions` → `compilerOptions`, kapt → KSP or `com.android.legacy-kapt`, set `targetSdk` explicitly |
| `grep -rn 'androidx.compose.compiler:compiler'` | any hit | Frozen at 1.5.15 since 2024-08-07 → `org.jetbrains.kotlin.plugin.compose` |
| `grep -rn 'android.libraryVariants'` | any hit | Removed by AGP 9's Variant API → `androidComponents` |
| `ndkVersion` in Gradle | unset | Pin explicitly to the AGP default (`28.2.13676358`) and match `ANDROID_NDK_HOME` |
| `targetSdk` | < 36 | Play has required 36 since 2026-08-31 |
| JNA dependency | missing `@aar`, or < 5.12.0 | Use `net.java.dev.jna:jna:5.19.1@aar` |
| Rust→Kotlin calls exist | no `attach_current_thread_permanently` | Add the thread-attach pattern; JNA otherwise attaches per call |

If the project copied UniFFI's own Gradle snippet, expect **both** problems at once: deprecated UDL flow and the removed `libraryVariants` API.

### Apple

| Check | Finding | Action |
|---|---|---|
| macOS version on build machines and CI | < Tahoe 26.6 | Cannot run Xcode 27. Upgrade runners before anything else |
| macOS deployment target | 11 | Floor is 12 in Xcode 27 |
| watchOS deployment target | 8 | Floor is 9 |
| CI test devices | iOS < 17 | Xcode 27 cannot debug on-device below iOS 17 |
| Bindings generation | `uniffi-bindgen -l swift` | Switch to `uniffi-bindgen-swift` for XCFramework modulemap control |
| `cargo install cargo-xcframework` in scripts | any hit | **No such crate.** The subcommand comes from the crate named `xcframework` |

### Node service

| Check | Finding | Action |
|---|---|---|
| Node version | 20.x | **EOL.** `firebase-admin` 14 requires ≥ 22. Move to 24 (Active LTS) |
| `bullmq` major | 5.x | Run the v6 tripwire list below |
| `ioredis` in dependencies | absent | v6 made it an optional peer dependency — add it explicitly |
| `grep -rn 'node-apn'` | bare `node-apn` | Dead since 2020 → `@parse/node-apn` |
| `grep -rn 'sendAll\|sendMulticast'` | any hit | Removed in firebase-admin v13.0.0 → `sendEach` / `sendEachForMulticast` |
| `grep -rn 'fcm/send\|googleapis.com/batch'` | any hit | Both return 404 → HTTP v1 |
| `grep -rn 'limiter.*groupKey'` | any hit | Removed in BullMQ 3.0. Not available in OSS v6 |
| `grep -rn '@taskforcesh/bullmq-pro'` | any hit | Commercial licence — remove per the OSS-only decision |
| Redis `maxmemory-policy` | anything but `noeviction` | Fix it; other policies silently corrupt queue state |
| APNs client construction | inside a job processor | Hoist to process scope |

### BullMQ v5 → v6 tripwires

- A bare `Connection` constructor parameter → `BackendFactory`
- `repeat` option on `add()`/`addBulk()`, `getRepeatableJobs()`, `removeRepeatable()` → Job Schedulers
- Reads of the `paused` state in `getJobCounts()` → now reported as `waiting`
- Un-awaited `Worker#resume()` → now async
- `debounce` / `debounceId` / `debounced` → `deduplication` / `deduplicationId` / `deduplicated`
- `Job#discard()` → `UnrecoverableError`
- Imports of `Scripts`, `createScripts`, `JobJsonRaw`, `RedisJobOptions` → backend APIs
- `Queue#client`, `Worker#blockingClient`, `FlowProducer#client` → `getBackend()`
- `RepeatOptions` with `utc: true` → `tz: 'UTC'`

### Architecture drift

| Check | Finding | Action |
|---|---|---|
| `cargo tree -p <core>` | any of `sqlx`, `diesel`, `rusqlite`, `sqlite`, `reqwest`, `tokio-postgres` | The core is doing IO. See core-purity-and-io.md |
| Foreign traits with `&str` / `&[u8]` params | any | Not supported (issue #2263, open) — must be by value |
| Foreign trait error types | no `From<uniffi::UnexpectedUniFFICallbackError>` | Generated code **panics** without it |
| Foreign trait methods | not returning `Result<>` | Errors panic |
| Foreign trait granularity | row-at-a-time | Batch it; per-call cost dominates |
| Cancellation | promised as Swift/coroutine cancellation | Does not propagate. Use an explicit `cancel()` flag |

## Upgrade ordering

Do not bump everything at once. This order keeps each failure attributable:

1. **Rust toolchain** to ≥ 1.90.0. Nothing else works below it.
2. **UniFFI** to `=0.32.1`, one minor at a time if coming from ≤ 0.30, applying each release's migrations. Regenerate bindings and rebuild both hosts after each step — checksum errors surface at runtime, not compile time.
3. **Android**: AGP → 9.4.1 with Gradle 9.7.1 and JDK 17+, then Kotlin 2.4.20, then Compose BOM. The built-in-Kotlin change is the disruptive one.
4. **Apple**: upgrade runners to macOS Tahoe 26.6+, then Xcode 27, then raise deployment target floors.
5. **Node**: Node → 24, then `firebase-admin` → 14, then BullMQ → 6 with `ioredis` added explicitly.

After each step, re-run the verification gates in bootstrap.md.

## Symptom → cause

Quick lookup for the failures that are hard to diagnose from the error alone:

| Symptom | Likely cause |
|---|---|
| Checksum error at runtime on device, fine on emulator | UniFFI < 0.32.1 on aarch64, or ARM32 signedness bug < 0.31.2 |
| Checksum error after upgrading UniFFI | Bindings and scaffolding from different versions — the contract-version guard does not catch this |
| iOS link failure, everything else fine | `staticlib` missing from `crate-type` |
| Android build works, Rust library not found at runtime | `jniLibs` layout or ABI filter mismatch |
| Crash inside UniFFI glue only with ASan on | UniFFI < 0.31.1 |
| Rust strings losing a leading character | BOM stripping in `FfiConverterString`, fixed in 0.31.2 |
| Android Rust→Kotlin calls unexpectedly slow | Missing `attach_current_thread_permanently` |
| `Cannot find module` for the native addon | Platform package missing from `optionalDependencies`, or a partial napi-rs release |
| Queue "backlog" that never drains but jobs complete fine | Rate-limited jobs stay in `waiting`, not `delayed` |
| Jobs processed twice | Normal. At-least-once plus stalled-job retry. Make sends idempotent |
| Push 404 on multi-send, single send fine | Legacy `/batch` endpoint via `sendAll`/`sendMulticast` |
| APNs `403 ExpiredProviderToken` | JWT `iat` older than one hour |
| APNs `429 TooManyProviderTokenUpdates` | Minting a JWT per request |
| APNs connection dropped under load | Too many 4xx — usually a dirty token list. Prune on 410 |
| FCM background push rejected on iOS | FCM defaulted `apns-priority` to 10; set it to 5 explicitly |
| FCM payload rejected only on topic sends | Topic cap is 2048 bytes, half the direct cap |
