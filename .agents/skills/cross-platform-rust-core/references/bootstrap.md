# Bootstrapping a new project

Verified 2026-09-19. Read versions.md first and pin everything from it.

## Order of work

Do these in order. Each step is verifiable before moving on, which keeps failures local.

### 1. Toolchain

```bash
rustup update stable          # need >= 1.90.0; UniFFI 0.32.x floor
rustc --version               # confirm before continuing
```

Add targets for the hosts you are actually building:

```bash
# Android
rustup target add aarch64-linux-android armv7-linux-androideabi \
                  x86_64-linux-android i686-linux-android
# Apple (macOS host only)
rustup target add aarch64-apple-ios aarch64-apple-ios-sim x86_64-apple-ios
```

`cargo binstall cargo-ndk` for Android. Apple needs Xcode 27 on macOS Tahoe 26.6+ — there is no Linux path.

### 2. Workspace skeleton

```
Cargo.toml              # [workspace] members = ["core", "uniffi-bindgen", "node/addon"]
core/                   # domain logic, cdylib + staticlib
uniffi-bindgen/          # [[bin]] uniffi-bindgen + [[bin]] uniffi-bindgen-swift
android/                # Gradle project
apple/                  # Swift Package
node/
  addon/                # NAPI-RS crate
  src/                  # service: queue, push, db
```

One lockfile at the root is the point — it is what makes the bindgen and the core share a UniFFI version structurally rather than by convention.

### 3. Core crate

Per rust-core-uniffi.md: `crate-type = ["cdylib", "staticlib"]`, `uniffi = "=0.32.1"`, `uniffi::setup_scaffolding!()`.

Export one trivial function first and get it all the way to both hosts before writing real logic. A round-trip like this proves every seam:

```rust
#[derive(uniffi::Record)]
pub struct Health { pub ok: bool, pub version: String }

#[uniffi::export]
pub fn health() -> Health {
    Health { ok: true, version: env!("CARGO_PKG_VERSION").to_string() }
}
```

**Keep IO out from the very first commit.** It is far easier to hold the line than to remove a database dependency later. See core-purity-and-io.md.

### 4. Bindgen binary

Two bin targets in `uniffi-bindgen/`, both pinned to `=0.32.1`: `uniffi::uniffi_bindgen_main()` and `uniffi::uniffi_bindgen_swift()`.

Verify:

```bash
cargo build --release -p yourapp-core
cargo run -p uniffi-bindgen -- generate \
    target/release/libyourapp_core.so --language kotlin --out-dir /tmp/kt
ls /tmp/kt
```

If that produces Kotlin, the hardest part of the plumbing is done.

### 5. Android

Per android-kotlin.md. Order: `cargo ndk` into `jniLibs` → generate Kotlin → wire the Gradle tasks → add JNA `@aar` and coroutines → call `health()` from a ViewModel and render it in Compose.

Pin the NDK explicitly and make `ndkVersion` in Gradle match. Do not apply `org.jetbrains.kotlin.android` — AGP 9 has built-in Kotlin.

### 6. Apple

Per apple-swift.md. Order: build the iOS targets → generate Swift sources, headers, and an XCFramework-compatible modulemap → `lipo` the simulator slices → `xcodebuild -create-xcframework` → Swift Package with a `binaryTarget` → call `health()` from an observable view model.

Remember device and simulator slices are separate XCFramework entries; do not `lipo` them together.

### 7. Node service

Per node-napi.md and queue-bullmq.md. Order:

1. `npx @napi-rs/cli new node/addon`, wrap `health()`, confirm it loads from Node.
2. Start Redis (6.2+) and set `maxmemory-policy=noeviction`.
3. `npm i bullmq ioredis` — **ioredis explicitly**, it is an optional peer dependency in v6.
4. Stand up one queue and one worker; confirm a job round-trips.
5. `npm i @parse/node-apn firebase-admin` and add the senders per push-apns-fcm.md.

Build the APNs client **once per process**, not per job.

### 8. Credentials

Needed only for real delivery, not for building:

- **Apple:** `.p8` key file, Key ID, Team ID, bundle ID.
- **Firebase:** service account JSON, project ID, and the same `.p8` uploaded to Firebase (at least one of dev/prod).

Keep them out of the repo. Provide a local mock sender so the service runs end-to-end without credentials — worth doing on day one so tests never need secrets.

## Verification gates

Do not call the bootstrap done until all of these pass:

- [ ] `rustc --version` ≥ 1.90.0
- [ ] `cargo tree -p yourapp-core` shows no database, HTTP, or filesystem crate
- [ ] `uniffi` version identical in core and bindgen, from one lockfile
- [ ] `crate-type` includes `staticlib`
- [ ] Android app displays the value from `health()` on a real device, not just an emulator
- [ ] Apple app displays the value from `health()`
- [ ] A BullMQ job round-trips against Redis with `noeviction` set
- [ ] A mock push send completes through the queue path
- [ ] No `with_foreign`, `--library`, `--lib-file`, or `UniffiCustomTypeConverter` anywhere

## Sequencing advice

Get `health()` to all three hosts **before** designing the real interface. The plumbing is where the version and toolchain problems live, and they are much cheaper to diagnose against a one-field record than against a real domain model.

Once plumbing is proven, design the interface per core-purity-and-io.md: pure functions taking owned data and returning owned decisions, with the hosts performing IO.
