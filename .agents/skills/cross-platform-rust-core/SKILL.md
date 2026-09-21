---
name: cross-platform-rust-core
description: Bootstrap or extend a cross-platform app built on a shared Rust core exposed through UniFFI to Kotlin/Jetpack Compose on Android and Swift/SwiftUI on Apple, plus a Node service using NAPI-RS with BullMQ on Redis for APNs and FCM push delivery. Use when working with UniFFI, uniffi-bindgen, cargo-ndk, XCFramework builds, NAPI-RS native addons, BullMQ queues, APNs token authentication, or FCM HTTP v1.
license: Apache-2.0
compatibility: Requires Rust 1.90+ and cargo. Android builds need the Android SDK and NDK; Apple builds require macOS Tahoe 26.6+ with Xcode 27. Node work requires Node 22+ and a Redis 6.2+ instance.
metadata:
  version: "1.0"
  verified-on: "2026-09-19"
---

# Cross-platform Rust core: UniFFI + NAPI-RS + push service

Build one Rust core and consume it from three hosts: Android (Kotlin/Compose), Apple (Swift/SwiftUI), and a Node push service. All version pins were verified against primary sources on **2026-09-19**.

## What this architecture is for

Four reasons to pay the FFI cost: **reusability** (one implementation of the domain, not three that drift), **testability** (the logic is plain Rust, tested without a device or an emulator), **standardization** (one set of rules and versions across Apple, Android, and the server), and **native UI freedom** (each platform keeps its own idiomatic UI).

**Non-goals — these are design choices, not gaps.** This is *not* write-once-run-anywhere, and it is *not* a shared UI layer: the UI is deliberately native per platform and is never shared. The core is shared **logic, not data** — it holds no persistence and no IO (rule 3). If the request is really "one UI everywhere," this skill is the wrong tool and React Native or Flutter is the right one.

## Architecture in one paragraph

A single Rust crate holds domain logic only. UniFFI generates first-party Kotlin and Swift bindings from it. A NAPI-RS addon exposes the same crate to a Node service, which runs BullMQ on Redis and delivers notifications through APNs and FCM. **The core performs no IO** — no database, no filesystem, no network. Hosts own all IO and pass owned data in and out.

## Non-negotiable rules

These are the ones that cause silent failure or expensive rework. Each links to detail.

1. **Pin every version exactly.** Use [references/versions.md](references/versions.md) as the source of truth. Do not resolve "latest".
2. **Pin `uniffi`, `uniffi_bindgen`, and the bindgen binary to the same exact version** and build the bindgen as a `[[bin]]` in your own workspace. UniFFI's runtime version guard does **not** catch a bindgen/scaffolding mismatch. See [references/rust-core-uniffi.md](references/rust-core-uniffi.md).
3. **The core owns no persistence.** Prefer pure functions over callback traits. See [references/core-purity-and-io.md](references/core-purity-and-io.md).
4. **UniFFI has no cancellation.** Expose an explicit `cancel()` that sets a flag the core checks; map it to an error variant. Never promise Swift `CancellationError` or coroutine cancellation semantics.
5. **Every push can be delivered twice.** BullMQ is at-least-once and a stalled job is retried by another worker. Make sends idempotent. See [references/queue-bullmq.md](references/queue-bullmq.md).
6. **Hold one long-lived APNs HTTP/2 connection pool per process.** Apple may treat repeated connect/disconnect as a denial-of-service attack. Never build a client inside a job processor.
7. **Use OSS BullMQ only.** No `@taskforcesh/bullmq-pro`, no `group` job option, no `job.getBatch()`.
8. **Never create or hand-edit IDE project files.** `project.pbxproj`, `.xcodeproj`/`.xcworkspace` contents, `.xcscheme`, and Android Studio's `.idea/` are IDE-generated and agent edits corrupt them in ways that surface later as confusing build errors. **The human creates the Apple and Android targets in Xcode and Android Studio; you walk them through it and verify each step.** You own everything textual — the Cargo workspace, the bindgen, build scripts, `Package.swift`, `*.gradle.kts`, and the Node service. See [references/bootstrap.md](references/bootstrap.md).

## Scope

**In scope:** Rust core, UniFFI Kotlin + Swift bindings, Android and Apple host integration, NAPI-RS Node addon, BullMQ queue architecture, APNs and FCM delivery.

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
| [references/node-napi.md](references/node-napi.md) | NAPI-RS addon, threadsafe functions, per-platform binary distribution |
| [references/queue-bullmq.md](references/queue-bullmq.md) | BullMQ v6 architecture, ordering, concurrency, stalled jobs, fairness |
| [references/push-apns-fcm.md](references/push-apns-fcm.md) | APNs JWT and headers, FCM HTTP v1, token lifecycle, error handling |

## Minimum viable shape

A correct project has these pieces. Details in the references.

```
core/                     # Rust crate: cdylib + staticlib, domain logic only
  src/lib.rs              #   uniffi::setup_scaffolding!()
uniffi-bindgen/           # [[bin]] pinned to the same uniffi version
android/                  # HUMAN-created in Android Studio; consumes generated Kotlin + jniLibs
apple/                    # HUMAN-created in Xcode; consumes XCFramework + generated Swift
node/                     # NAPI-RS addon + BullMQ service + APNs/FCM senders
```

## Fast sanity checks

Before declaring a build correct:

- `cargo tree | grep -i -E 'sqlx|diesel|rusqlite|sqlite|reqwest|tokio-postgres'` returns nothing in the core. IO crates in the core mean the architecture drifted.
- The `uniffi` version in the core and in the bindgen binary are byte-identical, and both come from one lockfile.
- `crate-type` includes both `cdylib` and `staticlib`. Missing `staticlib` breaks iOS only, and late.
- No occurrence of `with_foreign`, `--library`, `--lib-file`, or `UniffiCustomTypeConverter` anywhere.
- Android: the NDK version is pinned explicitly and matches the AGP default, not the newest installed NDK.
- Node: `ioredis` is a direct dependency, not assumed present. BullMQ v6 made it an optional peer dependency.
- Redis has `maxmemory-policy=noeviction`. Any other policy silently corrupts queue state.

## When you cannot verify something

This skill pins versions verified on 2026-09-19. If a pin looks stale, check the registry rather than guessing: crates.io for Rust, the npm registry for Node, Google Maven for Android artifacts, and `developer.apple.com/xcode/system-requirements` for Xcode. State the verification date when you change a pin, and prefer an exact version over a range.
