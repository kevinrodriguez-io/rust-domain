# Core purity: where IO lives

Verified 2026-09-19. The architectural rule: **the domain lives entirely in the core, and data does not.** Logic — rules, transitions, validation, decisions — is Rust, tested with `cargo test`, and is the single source of truth for both hosts. A database is IO, and IO belongs to the host.

## What this removes

- No SQLite or other C database in the Rust build — so no C dependency to get through `cargo-ndk` for four Android ABIs and then into an XCFramework.
- No migrations in Rust. Schema evolution stays with the host, which already has tooling (Room, SwiftData/Core Data versioning, or the Node service if you add one later).
- No cross-platform file path, sandbox, or encryption-at-rest abstraction in the core.
- Core tests need no device, no database, and no FFI — plain `cargo test`.

## Crate layout: UniFFI derives on the core, mirrors only for NAPI

```
core/                 # domain logic + UniFFI derives. No napi. No IO.
bindings/napi/        # optional expansion — NAPI-RS adapter, mirror types + From conversions
```

The `bindings/napi/` crate does not exist until a Node service is requested. **The asymmetry is deliberate. Do not "fix" it by making the core binding-free.** Three reasons:

1. **UniFFI derives are inert.** They expand to trait impls and metadata — no IO, no runtime behaviour, nothing that executes unless a binding calls in. So `uniffi` in the core's dependency graph does **not** violate rule 3, and the sanity-check grep is deliberately a list of **IO** crates (`sqlx`, `rusqlite`, `reqwest`, …), not binding crates. A binding crate is not an IO crate.
2. **Both mobile hosts consume UniFFI.** Moving its derives into an adapter would make that path pay a mirrored declaration per domain type purely to look symmetric with an optional Node adapter — which **maximizes** total duplication instead of minimizing it.
3. **The tools are genuinely asymmetric, so the layout should be too.** UniFFI is first-party for both mobile hosts *and* has remote types for the case where a crate cannot carry its derives. NAPI-RS is neither. Mirroring for the one host that requires it is the minimum, not a compromise.

`napi` stays out of the core: putting its derives on core would pull a Node-oriented binding crate into the artifacts built for four Android ABIs and an XCFramework, and would buy nothing the single adapter crate does not already give you.

| Host | Reaches core types via | Mirror cost |
|---|---|---|
| Apple (Swift) | UniFFI derives on the core directly | none |
| Android (Kotlin) | UniFFI derives on the core directly | none |
| Node service | `#[napi(object)]` mirror + `From` both ways in `bindings/napi` | one mirror per crossing type |

The underlying constraint is the same in both directions — Rust's orphan rule means an adapter crate cannot implement a foreign trait (`FromNapiValue`, or UniFFI's converters) for a foreign type. UniFFI names the orphan rule explicitly in [its remote-types docs](https://github.com/mozilla/uniffi-rs/blob/v0.32.1/docs/manual/src/types/remote_ext_types.md) and offers an escape from it; NAPI-RS just requires the derive on the defining crate. See node-napi.md for the worked mirror-type pattern and the serialized-payload alternative.

**Node mirroring is not a reason to flatten the domain.** Every type that crosses into Node costs a mirror struct and two conversions. Android and iOS pay nothing extra for another type or another function, so factor the core for reuse. If a Node adapter is added later, it mirrors the types that actually cross.

### Optional: a binding-free core via feature gates

**Not the default.** But if the transitive `uniffi` dependency in the Node build genuinely matters to you, feature-gate the derives on the core rather than mirroring them into an adapter:

```rust
#[cfg_attr(feature = "uniffi", derive(uniffi::Record))]
pub struct DeviceRecord { /* … */ }
```

The trade is worth stating plainly: that is *N* one-line attributes that live next to the type they describe and therefore **cannot drift**, versus *N* duplicated declarations in a separate crate that **can**. So if someone wants a core with no binding attributes compiled in, feature gates are the way to get it — not a UniFFI adapter full of `#[uniffi::remote(...)]` mirrors.

## Factor the domain; keep IO in the host

The core is a Rust library. Give it that shape.

- **Pure functions** for calculations and decisions, small enough to test and compose.
- **Types** — records, enums, and objects — that carry the domain and expose behavior.
- **Traits** where a protocol is the real abstraction. UniFFI exposes objects as classes, and traits as Swift protocols and Kotlin interfaces.

The host reads and writes. It calls these functions and methods with values it already has. One function that accepts the whole input and returns every decision cannot be reused and cannot be tested in pieces. Do not write it.

```rust
#[derive(uniffi::Record)]
pub struct Money { pub cents: i64, pub currency: String }

#[uniffi::export]
pub fn add(left: Money, right: Money) -> Result<Money, CoreError> { ... }

#[derive(uniffi::Object)]
pub struct Ledger { /* entries */ }

#[uniffi::export]
impl Ledger {
    pub fn balance(&self) -> Money { ... }
    pub fn apply(&self, entry: Entry) -> Result<Arc<Ledger>, CoreError> { ... }
}
```

These are core types. Apple and Android reach them through UniFFI with no mirror. A later Node adapter pays one `#[napi(object)]` mirror per type that crosses, per the table above.

A trait the host calls is part of this API. A trait the core calls back into the host, to pull data, is the IO fallback below. Use it for that, not to structure the domain.

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
2. **Batch a foreign trait that pulls data.** Row-at-a-time callbacks into Kotlin pay the attach cost on every call. That constraint is about this IO port. It is not a reason to merge domain functions together.

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

Expansion. Skip this section on a mobile-only project.

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
2. Export a factored Rust API: pure functions, objects, and traits. Do not add a single entry point that takes the whole input and returns the whole output.
3. Foreign traits only for unavoidable pull-shaped IO, declared `#[uniffi::export(rust, foreign)]`, every method returning `Result<>`, with a mandatory `From<uniffi::UnexpectedUniFFICallbackError>` impl. Batch that pull.
4. All foreign-trait parameters by value.
5. The Android thread-attach pattern is mandatory wherever Rust threads call into Kotlin.
6. Cancellation and query timeouts are host-side.
7. On Node, pass owned data in rather than installing a callback.
8. Watch for Rust↔host reference cycles anywhere a foreign trait is used.
