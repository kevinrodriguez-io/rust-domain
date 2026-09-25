---
name: rust-domain
description: Put an Android and iOS app's domain entirely in one Rust core — a single source of truth, tested with cargo test, and shared through UniFFI to Kotlin/Jetpack Compose and Swift/SwiftUI. Use when the goal is unified Rust testability and one implementation of the rules, or when working with UniFFI, uniffi-bindgen, cargo-ndk, or XCFramework builds. A Node service (NAPI-RS, BullMQ, APNs, FCM) is an optional later expansion.
license: Apache-2.0
compatibility: Requires Rust 1.90+ and cargo. Android and Apple builds use the Android Studio and Xcode the user already has; do not upgrade them. Apple linking needs macOS. The optional Node expansion needs Node 22+ and a Redis 6.2+ instance.
metadata:
  version: "1.2"
  verified-on: "2026-09-19"
---

# Rust domain

Build one Rust core and consume it from two hosts: Android (Kotlin/Compose) and Apple (Swift/SwiftUI). A Node service is an optional expansion, outside the starting shape. All version pins were verified against primary sources on **2026-09-19**.

## The goal

The domain lives **100% in Rust**. That is the point of paying the FFI cost, and it is what the hosts are not allowed to erode.

- **Unified testability.** `cargo test` is the test suite for the domain. No simulator, no emulator, no device, no database. A bug fixed in Rust is fixed on both platforms.
- **Proper sharing.** Android and iOS call the same functions through generated bindings. Sharing means that crate, not a Kotlin port and a Swift port of the same rules.
- **Single source of truth.** A rule exists in one place. If a business rule appears in Kotlin or Swift, it has already drifted; move it into the core and cover it with a Rust test.
- **The whole domain fits.** Calculations, state transitions, validation, and decisions belong in the core. Hosts render UI and perform IO. They pass owned data in and receive owned decisions back.

Screens, navigation, persistence, and network stay on each host, so the shared slice of a whole app is however much of it is rules. Every rule that lives in the core exists once.

**Non-goals — these are design choices, not gaps.** This is *not* write-once-run-anywhere, and it is *not* a shared UI layer: the UI is deliberately native per platform and is never shared. The core is shared **logic, not data** — it holds no persistence and no IO (rule 3). If the request is really "one UI everywhere," this skill is the wrong tool and React Native or Flutter is the right one.

## Architecture in one paragraph

A single Rust crate holds the domain. UniFFI generates first-party Kotlin and Swift bindings from it. **The core performs no IO** — no database, no filesystem, no network — so the domain stays plain Rust and `cargo test` stays sufficient. Each host owns its IO and passes owned data in and out. The same crate can later be wrapped for a Node service; that path is documented and stays out of bootstrap.

## Non-negotiable rules

These are the ones that cause silent failure or expensive rework. Each links to detail.

1. **Pin library versions exactly.** Use [references/versions.md](references/versions.md) for UniFFI and the other libraries a mismatch breaks at runtime. Do not resolve "latest" for those. The user's Xcode and Android Studio are theirs — use what is installed, and do not upgrade it.
2. **Pin `uniffi`, `uniffi_bindgen`, and the bindgen binary to the same exact version** and build the bindgen as a `[[bin]]` in your own workspace. UniFFI's runtime version guard does **not** catch a bindgen/scaffolding mismatch. See [references/rust-core-uniffi.md](references/rust-core-uniffi.md).
3. **The core owns no persistence.** Prefer pure functions over callback traits. See [references/core-purity-and-io.md](references/core-purity-and-io.md).
4. **UniFFI has no cancellation.** Expose an explicit `cancel()` that sets a flag the core checks; map it to an error variant. Never promise Swift `CancellationError` or coroutine cancellation semantics.
5. **Never create or hand-edit IDE project files.** `project.pbxproj`, `.xcodeproj`/`.xcworkspace` contents, `.xcscheme`, and Android Studio's `.idea/` are IDE-generated and agent edits corrupt them in ways that surface later as confusing build errors. **The human creates the Apple and Android targets in Xcode and Android Studio; you walk them through it and verify each step.** You own everything textual — the Cargo workspace, the bindgen, build scripts, `Package.swift`, and `*.gradle.kts`. See [references/bootstrap.md](references/bootstrap.md).

**Expansion rules.** These apply only once a Node service is in scope. Do not introduce that service in order to satisfy them.

6. **Every push can be delivered twice.** BullMQ is at-least-once and a stalled job is retried by another worker. Make sends idempotent. See [references/queue-bullmq.md](references/queue-bullmq.md).
7. **Hold one long-lived APNs HTTP/2 connection pool per process.** Apple may treat repeated connect/disconnect as a denial-of-service attack. Never build a client inside a job processor.
8. **Use OSS BullMQ only.** No `@taskforcesh/bullmq-pro`, no `group` job option, no `job.getBatch()`.

## Scope

**Starting scope:** Rust core, UniFFI Kotlin + Swift bindings, Android and Apple host integration.

**Expansion — do not add unless asked.** The references stay valid as a guide: NAPI-RS Node addon, BullMQ on Redis, APNs and FCM delivery. Load them when the project grows a server. Leave them unread during a mobile bootstrap.

**Out of scope — do not add without being asked:** WebAssembly or any browser target, Kotlin Multiplatform (and therefore Gobley), and all third-party UniFFI generators (Go, C#, Dart, Java, C++, React Native). These constrain the stack to older UniFFI versions. Keeping them out is what allows pinning UniFFI 0.32.1.

## Choose your path

**Bootstrapping a new project?** Follow [references/bootstrap.md](references/bootstrap.md).

**Extending an existing project?** Run the detection checklist in [references/extend-existing.md](references/extend-existing.md) *before* changing anything. Several widely-copied patterns are now deprecated or removed, and the failure modes are quiet.

## Reference map

Load only what the current task needs.

| File | Use when |
|---|---|
| [references/versions.md](references/versions.md) | Pinning anything at all. Always read this first. |
| [references/rust-core-uniffi.md](references/rust-core-uniffi.md) | Writing the core crate, UniFFI exports, error types, custom types, bindgen invocation |
| [references/core-purity-and-io.md](references/core-purity-and-io.md) | Deciding where persistence, network, or filesystem access lives; designing foreign traits |
| [references/android-kotlin.md](references/android-kotlin.md) | Android builds, cargo-ndk, Gradle/AGP, Compose, JNA performance |
| [references/apple-swift.md](references/apple-swift.md) | XCFramework builds, Swift Package layout, Xcode and deployment targets |
| [references/node-napi.md](references/node-napi.md) | Expansion. NAPI-RS addon, threadsafe functions, per-platform binary distribution |
| [references/queue-bullmq.md](references/queue-bullmq.md) | Expansion. BullMQ v6 architecture, ordering, concurrency, stalled jobs, fairness |
| [references/push-apns-fcm.md](references/push-apns-fcm.md) | Expansion. APNs JWT and headers, FCM HTTP v1, token lifecycle, error handling |

## Minimum viable shape

A correct project has these pieces. Details in the references.

```
core/                     # Rust crate: cdylib + staticlib, domain logic only
  src/lib.rs              #   uniffi::setup_scaffolding!()
uniffi-bindgen/           # [[bin]] pinned to the same uniffi version
android/                  # HUMAN-created in Android Studio; consumes generated Kotlin + jniLibs
apple/                    # HUMAN-created in Xcode; consumes XCFramework + generated Swift
```

A Node service (`node/`, NAPI-RS + BullMQ + APNs/FCM) is added later, and only when asked. Follow the expansion section in [references/bootstrap.md](references/bootstrap.md).

## Fast sanity checks

Before declaring a build correct:

- Domain rules live in the Rust core and have Rust tests. A Kotlin or Swift copy of a business rule is drift.
- `cargo tree | grep -i -E 'sqlx|diesel|rusqlite|sqlite|reqwest|tokio-postgres'` returns nothing in the core. IO crates in the core mean the domain is no longer fully testable with `cargo test`.
- The `uniffi` version in the core and in the bindgen binary are byte-identical, and both come from one lockfile.
- `crate-type` includes both `cdylib` and `staticlib`. Missing `staticlib` breaks iOS only, and late.
- No occurrence of `with_foreign`, `--library`, `--lib-file`, or `UniffiCustomTypeConverter` anywhere.
- Android: `ndkVersion` matches the Android Gradle Plugin already in the user's project, not a newer NDK the agent picked.

When a Node service exists:

- `ioredis` is a direct dependency, not assumed present. BullMQ v6 made it an optional peer dependency.
- Redis has `maxmemory-policy=noeviction`. Any other policy silently corrupts queue state.

## When you cannot verify something

This skill pins versions verified on 2026-09-19. If a pin looks stale, check the registry rather than guessing: crates.io for Rust, the npm registry for Node, Google Maven for Android artifacts, and `developer.apple.com/xcode/system-requirements` for Xcode. State the verification date when you change a pin, and prefer an exact version over a range.
