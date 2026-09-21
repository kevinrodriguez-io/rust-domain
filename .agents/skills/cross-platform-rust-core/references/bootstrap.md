# Bootstrapping a new project

Verified 2026-09-19. Read versions.md first and pin everything from it.

## Division of labour — read before doing anything

**The human creates the Apple and Android targets in Xcode and Android Studio. The agent never does.**

Why this is a hard rule and not a preference: `project.pbxproj` and Android Studio's module wiring are **IDE-generated artifacts**. They are large, order-sensitive, contain generated UUIDs, and have no stable public schema. An agent editing them produces files that *look* plausible and parse, then fail much later as confusing build errors — a missing build phase surfacing as a link error three steps downstream, or a corrupted file reference that only breaks on a clean checkout. A human does the same thing correctly in the IDE in about two minutes using the wizard that exists for it.

| Owner | Responsibility |
|---|---|
| **Human (in the IDE)** | Creating the Xcode project and app target; creating the Android Studio project and app module; adding the package/framework dependency through the IDE UI; signing and capabilities; running on device |
| **Agent (everything textual)** | Cargo workspace and `core` crate; the in-workspace `uniffi-bindgen` binary; build scripts; `Package.swift`; `build.gradle.kts` / `settings.gradle.kts` edits it can verify with `./gradlew`; the Node service; all generated-bindings plumbing |

### Prohibitions

- **Never open, generate, or hand-edit `project.pbxproj`.** Not to add a file, not to add a build phase, not to "fix" a path. If something needs to change inside the Xcode project, the human does it in Xcode and the agent verifies the result.
- Never create or edit `.xcodeproj` / `.xcworkspace` contents, `.xcscheme` files, or Android Studio's `.idea/` directory.
- Never run a codegen tool whose job is to emit an Xcode project.
- `Package.swift`, `build.gradle.kts`, and `settings.gradle.kts` **are** fair game — they are plain text with stable syntax and the agent can prove its edits with `swift build` or `./gradlew`.

## How to run the interactive steps

Steps below are marked **▸ HUMAN** or **✓ GATE**. The protocol is not optional.

1. When you reach a **▸ HUMAN** step, print *only that step* and stop.
2. **Wait for the human to confirm they have done it.** Do not print the next step. Do not print several human steps at once. Do not assume completion and continue.
3. When they confirm, run the **✓ GATE** for that step yourself and compare against the stated expected result.
4. Only if the gate passes, move on.
5. **If a gate fails, stop.** Report the command, the actual output, and the expected output, then ask. Do not improvise a fix inside the IDE project, and do not proceed hoping it resolves later.

A gate is a command the agent runs, not an instruction it prints. "Step by step" without gates degenerates into a wall of text where neither side knows which step broke.

---

## 1. Toolchain — agent

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

Installing Xcode and Android Studio is the human's job; the agent only checks they are present.

## 2. Workspace skeleton — agent

```
Cargo.toml              # [workspace] members = ["core", "uniffi-bindgen", "node/addon"]
core/                   # agent  — domain logic, cdylib + staticlib
uniffi-bindgen/         # agent  — [[bin]] uniffi-bindgen + [[bin]] uniffi-bindgen-swift
android/                # HUMAN  — Android Studio project; agent edits only *.gradle.kts
apple/                  # HUMAN  — Xcode project; agent edits only Package.swift
node/
  addon/                # agent  — NAPI-RS crate
  src/                  # agent  — service: queue, push, db
```

Create only the agent-owned directories now. Leave `android/` and `apple/` empty — the human fills them in steps 5 and 6.

One lockfile at the root is the point — it is what makes the bindgen and the core share a UniFFI version structurally rather than by convention.

## 3. Core crate — agent

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

## 4. Bindgen binary — agent

Two bin targets in `uniffi-bindgen/`, both pinned to `=0.32.1`: `uniffi::uniffi_bindgen_main()` and `uniffi::uniffi_bindgen_swift()`.

**✓ GATE** — bindings generate at all:

```bash
cargo build --release -p yourapp-core
cargo run -p uniffi-bindgen -- generate \
    target/release/libyourapp_core.so --language kotlin --out-dir /tmp/kt
ls /tmp/kt
```

*Expected:* at least one `.kt` file under `/tmp/kt`. If this fails, stop here — nothing downstream can work, and it is far cheaper to debug now.

## 5. Android — interactive

Reference: android-kotlin.md. Do **not** apply `org.jetbrains.kotlin.android` (AGP 9 has built-in Kotlin), and pin `ndkVersion` to the AGP default.

### 5a ▸ HUMAN — create the project

> In Android Studio: **New Project → Empty Activity**. Set the package name and save it into `android/` in this repo. Accept the defaults for everything else. Tell me when the project is created.

**Then stop and wait.**

**✓ GATE:**

```bash
cd android && ./gradlew projects
```

*Expected:* the output lists `--- Project ':app'`. Also confirm `android/app/build.gradle.kts` exists.

### 5b — agent: build the Rust library into jniLibs

```bash
cargo ndk -t arm64-v8a -t armeabi-v7a -t x86_64 -t x86 \
  -o android/app/src/main/jniLibs build --release
```

**✓ GATE:**

```bash
ls android/app/src/main/jniLibs/arm64-v8a/
```

*Expected:* `libyourapp_core.so` present.

### 5c — agent: generate Kotlin bindings and wire Gradle

Generate into a source directory, then add the JNA and coroutines dependencies and register the generated directory as a Kotlin source set. Agent-owned, because it is all `*.gradle.kts`.

**✓ GATE:**

```bash
cd android && ./gradlew :app:dependencies --configuration debugRuntimeClasspath | grep -E 'jna|coroutines-core'
```

*Expected:* `net.java.dev.jna:jna:5.19.1` and `kotlinx-coroutines-core:1.11.0` both resolve. A missing JNA `@aar` classifier shows up here rather than as a runtime crash later.

### 5d ▸ HUMAN — call the core from the app

> Add a call to `health()` in your Activity or ViewModel and render the result. Import it from the generated package (`uniffi.yourapp_core`). Tell me when you have added it.

**Then stop and wait.**

**✓ GATE** — bindings import and the native library is packaged:

```bash
cd android && ./gradlew :app:assembleDebug
unzip -l app/build/outputs/apk/debug/app-debug.apk | grep 'lib/arm64-v8a/libyourapp_core.so'
```

*Expected:* `BUILD SUCCESSFUL`, and the `.so` appears in the APK listing. A successful build proves the generated Kotlin imported; the `unzip` line proves the native library is actually shipped, which a build alone does not.

### 5e ▸ HUMAN — run on a real device

> Run the app on a physical device, not an emulator, and tell me what value `health()` returned.

**Then stop and wait.** Device-only checksum failures are a known UniFFI/JNA class of bug (see android-kotlin.md), so an emulator pass is not sufficient evidence.

## 6. Apple — interactive

Reference: apple-swift.md. macOS with Xcode 27 only.

### 6a — agent: build the XCFramework

Build the iOS targets, generate Swift sources, headers, and an XCFramework-compatible modulemap, `lipo` the simulator slices together, then `xcodebuild -create-xcframework`. Device and simulator slices are **separate** XCFramework entries — do not `lipo` them together.

**✓ GATE:**

```bash
xcodebuild -create-xcframework -help >/dev/null && \
  ls build/YourAppCore.xcframework/Info.plist && \
  ls build/YourAppCore.xcframework
```

*Expected:* `Info.plist` exists and there are **two** platform directories (device and simulator). One directory means the slices were merged incorrectly.

### 6b — agent: write Package.swift

`Package.swift` is plain text and agent-owned. Declare a `binaryTarget` pointing at the XCFramework plus a `target` for the generated Swift.

**✓ GATE:**

```bash
cd apple && swift build 2>&1 | tail -5
```

*Expected:* the package resolves and builds. (On a package containing only a binary target plus generated Swift, this proves the modulemap and headers are coherent before Xcode is involved at all.)

### 6c ▸ HUMAN — create the app target

> In Xcode: **File → New → Project → App**, save it into `apple/` in this repo. Set the bundle identifier to match what you will use for APNs. Tell me when the project is created.

**Then stop and wait.**

**✓ GATE:**

```bash
xcodebuild -list -project apple/YourApp.xcodeproj
```

*Expected:* the target and a scheme are listed.

### 6d ▸ HUMAN — add the package to the target

> In Xcode: select the project → your app target → **General → Frameworks, Libraries, and Embedded Content → +** → **Add Local…** → choose the `apple/` Swift package. Confirm it appears in the list. Tell me when it is added.

**Then stop and wait.** This is the step that writes to `project.pbxproj`, and it is why the agent does not.

**✓ GATE** — the framework is genuinely linked, not merely referenced:

```bash
xcodebuild -project apple/YourApp.xcodeproj -scheme YourApp \
  -destination 'generic/platform=iOS Simulator' build 2>&1 | tail -5
```

*Expected:* `BUILD SUCCEEDED`. Then confirm the UniFFI symbols are actually in the product:

```bash
nm "$(xcodebuild -project apple/YourApp.xcodeproj -scheme YourApp \
  -destination 'generic/platform=iOS Simulator' -showBuildSettings \
  | awk -F' = ' '/ BUILT_PRODUCTS_DIR/{d=$2} / EXECUTABLE_PATH/{e=$2} END{print d"/"e}')" \
  | grep -c 'uniffi_'
```

*Expected:* a count greater than `0`. A target can reference a package and still not link it; this distinguishes the two, which a green build does not.

### 6e ▸ HUMAN — call the core from SwiftUI

> Add `import YourAppCore`, call `health()`, and render the result. Keep the call behind an `@Observable` view model rather than in a `View` body. Tell me when it is in.

**Then stop and wait.**

**✓ GATE:** rebuild as in 6d. *Expected:* `BUILD SUCCEEDED` — which now also proves the generated Swift module imports.

## 7. Node service — agent

Per node-napi.md and queue-bullmq.md. Fully agent-owned; no IDE involved.

1. `npx @napi-rs/cli new node/addon`, wrap `health()`, confirm it loads from Node.
2. Start Redis (6.2+) and set `maxmemory-policy=noeviction`.
3. `npm i bullmq ioredis` — **ioredis explicitly**, it is an optional peer dependency in v6.
4. Stand up one queue and one worker; confirm a job round-trips.
5. `npm i @parse/node-apn firebase-admin` and add the senders per push-apns-fcm.md.

Build the APNs client **once per process**, not per job.

**✓ GATE:**

```bash
node -e "console.log(require('./node/addon').health())"
redis-cli config get maxmemory-policy
```

*Expected:* the health record prints, and the policy is `noeviction`.

## 8. Credentials — human

Needed only for real delivery, not for building:

- **Apple:** `.p8` key file, Key ID, Team ID, bundle ID.
- **Firebase:** service account JSON, project ID, and the same `.p8` uploaded to Firebase (at least one of dev/prod).

Keep them out of the repo. Provide a local mock sender so the service runs end-to-end without credentials — worth doing on day one so tests never need secrets.

---

## Final verification gates

Do not call the bootstrap done until all of these pass. Agent-run unless marked.

- [ ] `rustc --version` ≥ 1.90.0
- [ ] `cargo tree -p yourapp-core` shows no database, HTTP, or filesystem crate
- [ ] `uniffi` version identical in core and bindgen, from one lockfile
- [ ] `crate-type` includes `staticlib`
- [ ] `git status` shows **no agent modifications** to `project.pbxproj`, `.xcworkspace`, or `.idea/`
- [ ] Android: `:app:assembleDebug` succeeds and the `.so` is present in the APK
- [ ] Android *(human)*: the app shows `health()` on a **physical device**
- [ ] Apple: `BUILD SUCCEEDED` for the simulator destination, and `nm | grep uniffi_` is non-zero
- [ ] Apple *(human)*: the app shows `health()`
- [ ] A BullMQ job round-trips against Redis with `noeviction` set
- [ ] A mock push send completes through the queue path
- [ ] No `with_foreign`, `--library`, `--lib-file`, or `UniffiCustomTypeConverter` anywhere

## Sequencing advice

Get `health()` to all three hosts **before** designing the real interface. The plumbing is where the version and toolchain problems live, and they are much cheaper to diagnose against a one-field record than against a real domain model.

Once plumbing is proven, design the interface per core-purity-and-io.md: pure functions taking owned data and returning owned decisions, with the hosts performing IO.
