# Core purity: where IO lives

Verified 2026-09-19. The architectural rule: **data is not part of the core, only the logic that manages it.** A database is IO, and IO belongs to the host.

## What this removes

- No SQLite or other C database in the Rust build — so no C dependency to get through `cargo-ndk` for four Android ABIs and then into an XCFramework.
- No migrations in Rust. Schema evolution stays with the host, which already has tooling (Room, SwiftData/Core Data versioning, whatever the Node service uses).
- No cross-platform file path, sandbox, or encryption-at-rest abstraction in the core.
- Core tests need no device, no database, and no FFI — plain `cargo test`.

## Prefer command/result over a callback port

There are two ways to express "the host owns IO". They are not equally good.

### Preferred: command/result (pure core)

Core functions take owned data and return owned decisions. The host reads, calls the core, and performs the writes.

```rust
#[uniffi::export]
pub fn plan_notifications(
    devices: Vec<DeviceRecord>,
    events: Vec<DomainEvent>,
) -> Result<Vec<SendCommand>, CoreError> { ... }
```

- **One FFI crossing per operation instead of N.**
- No foreign trait, so no reference-cycle risk and no callback error-mapping surface.
- Deterministic and testable without FFI.
- Matches what the rule actually says: the core *manages* data, it does not fetch it.

Recursive enums (0.32.0) and methods on records/enums (0.31.0) make rich command and result types ergonomic, so this style does not force anaemic structs.

### Fallback: foreign trait port

Only when access is genuinely pull-shaped and the core cannot know up front what it needs — lazy paging, cache-miss lookups.

```rust
#[uniffi::export(rust, foreign)]      // `rust, foreign` permits an in-memory Rust fake for tests
pub trait Repository: Send + Sync + Debug {
    fn load_batch(&self, keys: Vec<String>) -> Result<Vec<Record>, CoreError>;
}
```

Use `rust, foreign` rather than `foreign` so you can write an in-memory Rust implementation for core unit tests.

## Why the callback port is costly

### Foreign trait methods cannot take references

UniFFI's docs: "references in foreign trait methods aren't supported, so all parameters must be passed by value." The tracking issue (mozilla/uniffi-rs#2263) has been **open since 2024-10-10 with no movement since 2025-05-02** — do not plan around a fix.

So every parameter is an owned `String` / `Vec<u8>` / owned record, and every call allocates and copies in both directions. The zero-copy `&[u8]` support added in 0.32.0 does not help here: it is "not yet supported in async functions on any language," and host database IO is asynchronous.

### Rust→Kotlin calls are individually expensive

From UniFFI's Kotlin docs: without an attached thread, "JNA has to attach and detach a Java thread into the native thread in **every function call, which is a heavy operation**."

Two mitigations, both mandatory rather than optional:

1. Cache the `JavaVM` and attach permanently on your runtime's worker threads — see android-kotlin.md for the exact pattern.
2. **Make the port coarse-grained.** Batch methods, never row-at-a-time. One call returning 500 records beats 500 calls, because per-call cost dominates per-byte cost.

### Errors panic by default

All trait methods must return `Result<>` with a compatible error type, "otherwise these errors will panic." And you **must** implement `From<uniffi::UnexpectedUniFFICallbackError>` for that error type, or "the generated code will panic."

For a persistence port this is critical rather than pedantic: database errors are routine, and the default behaviour on an unmapped one is a panic across the FFI.

```rust
impl From<uniffi::UnexpectedUniFFICallbackError> for CoreError {
    fn from(e: uniffi::UnexpectedUniFFICallbackError) -> Self {
        CoreError::Host { reason: e.reason }
    }
}
```

### Cycles leak

A host repository holding a Rust object that holds the repository trait is a reference cycle. UniFFI "doesn't try to help here and there's no universal advice." Keep the host implementation from retaining the Rust object.

### No cancellation

A slow query cannot be aborted from the core. Enforce timeouts inside the host's trait implementation.

## The Node service: invert, don't call back

NAPI-RS makes it easy to hand a JS function to Rust as a `ThreadsafeFunction`. Avoid it for persistence. Let Node do the database work and pass owned data into Rust.

Concrete reasons:

- **TSFN argument and return types must be `'static`.** Scoped JS values (`Unknown<'env>`, `Object<'env>`, `Function<'env, …>`) cannot be used as `CallJsBackArgs` — you get `E0521`. So you copy anyway.
- **`CalleeHandled: false` is dangerous.** Under it, "a synchronous throw in the JavaScript callback is routed through `napi_fatal_exception`, and a returned `Promise` is not awaited automatically." A database callback can throw, and `napi_fatal_exception` tears down the process — in a BullMQ worker that turns one bad query into lost in-flight jobs and, given at-least-once semantics, duplicate pushes on redelivery. Keep the default `CalleeHandled: true`.

The natural layering for the push service:

```
BullMQ processor
  → Node reads what it needs from the database
  → Rust core computes the send decisions (pure)
  → Node performs the APNs/FCM sends and the database writes
```

Both genuinely IO-shaped concerns — persistence and push delivery — stay in the host, where the connection-pooling requirements already have to be satisfied.

## Interface rules to enforce

1. No database, ORM, migration, HTTP client, or filesystem crate in the core's dependency graph. This is lintable.
2. Default to pure exported functions over owned domain types.
3. Foreign traits only for unavoidable pull-shaped access, declared `#[uniffi::export(rust, foreign)]`, every method returning `Result<>`, with a mandatory `From<uniffi::UnexpectedUniFFICallbackError>` impl and coarse batch granularity.
4. All foreign-trait parameters by value.
5. The Android thread-attach pattern is mandatory wherever Rust threads call into Kotlin.
6. Cancellation and query timeouts are host-side.
7. On Node, pass owned data in rather than installing a callback.
8. Watch for Rust↔host reference cycles anywhere a foreign trait is used.
