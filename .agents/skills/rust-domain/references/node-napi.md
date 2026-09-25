# Node service: NAPI-RS

Expansion guide. The starting scope is Android and iOS. Apply this only when adding a server.

Verified 2026-09-19. Pins in versions.md.

## Why NAPI-RS and not UniFFI here

UniFFI's Node support is third-party and immature. NAPI-RS is the mature path for Node, gives native performance and real threading, and its per-platform prebuilt binary distribution is a solved problem. The Node service consumes the same core crate through a thin `#[napi]` wrapper.

## Scaffolding

```bash
npx @napi-rs/cli new node/addon
```

It prompts for the package name, the minimum Node-API level, target triples, license, TypeScript declarations, and whether to keep the GitHub Actions workflow.

Requirements: Node `^20.17.0 || ^22.13.0 || >=23.5.0` for the CLI itself, and Rust 1.88+. Note the separate constraint that `firebase-admin` 14 requires Node ≥ 22, so the service floor is higher than the CLI's — pin Node 24 (Active LTS).

Generated project shape:

```
src/lib.rs        # #[napi] exports
Cargo.toml
build.rs          # required napi-rs build setup
package.json      # includes the `napi` config block
.github/workflows/CI.yml
```

## The binding surface

### Core types cannot cross the boundary directly

This is the single thing to get right, and the naive version does not compile.

Every exported argument must implement `FromNapiValue` and every returned value must implement `ToNapiValue` ([type conversions](https://napi.rs/docs/concepts/type-conversions)). `Vec<T>` maps to `Array<T>` but "Input requires each element to implement `FromNapiValue`". Those impls come from `#[napi(object)]`, which is a derive — so the crate that *defines* the struct must depend on `napi`.

That means you cannot write this:

```rust
// ✗ Does not compile. DeviceRecord is a core type with no napi impls.
#[napi]
pub fn plan_notifications(devices: Vec<DeviceRecord>, events: Vec<DomainEvent>)
    -> napi::Result<Vec<SendCommand>> { ... }
```

And you cannot fix it from the adapter either: `FromNapiValue` is a foreign trait and `DeviceRecord` is a foreign type, so Rust's orphan rule forbids the impl. The only way to make the core's own types work would be to put `#[napi(object)]` on them in the core crate — which pulls a Node-oriented binding crate into the crate that is also compiled for four Android ABIs and linked into an XCFramework, to serve an optional host. See core-purity-and-io.md for why the core deliberately carries UniFFI derives but not these.

### Mirror types in the adapter

So the adapter owns its own types and converts:

```rust
// bindings/napi/src/lib.rs
use napi_derive::napi;
use yourapp_core as core;

/// #[napi(object)] requires all fields to be public.
#[napi(object)]
pub struct DeviceRecord {
    pub id: String,
    pub platform: String,
    pub fid: Option<String>,
}

impl From<DeviceRecord> for core::DeviceRecord {
    fn from(v: DeviceRecord) -> Self { /* field-by-field */ }
}

#[napi(object)]
pub struct SendCommand { /* … */ }

impl From<core::SendCommand> for SendCommand {
    fn from(v: core::SendCommand) -> Self { /* field-by-field */ }
}

#[napi]
pub fn plan_notifications(
    devices: Vec<DeviceRecord>,
    events: Vec<DomainEvent>,
) -> napi::Result<Vec<SendCommand>, CoreErrorCode> {
    let commands = core::plan_notifications(
        devices.into_iter().map(Into::into).collect(),
        events.into_iter().map(Into::into).collect(),
    )
    .map_err(to_napi)?;
    Ok(commands.into_iter().map(Into::into).collect())
}
```

`async fn` works and maps to a JS `Promise`. Enable the `tokio_rt` / `async` features on the `napi` crate if the core needs a Tokio runtime.

### The duplication is a real cost — name it, do not hide it

Every core type on the Node boundary needs a mirror struct plus one or two `From` impls, and they drift silently: adding a field to the core type compiles fine until you notice the adapter never forwards it. Mitigations, in order:

1. **Keep the boundary narrow.** This is the strongest argument for the command/result design in core-purity-and-io.md — few, coarse functions mean few mirrored types. A chatty API multiplies this cost directly.
2. **Add a round-trip test per mirrored type** (`core → mirror → core` equality). It is the only cheap defence against a forgotten field.
3. **Pass a serialized payload instead** when the type count makes mirroring worse than the copy. One `String` (or `Buffer`) of JSON across the boundary, deserialized into core types on the Rust side, needs no mirrors at all. You pay serialization on every call and lose the generated TypeScript shape, so this is the right trade when there are many types crossing rarely, and the wrong one for a hot path with two types. napi-rs can convert `serde_json::Value` with the `serde-json` feature, but note the documented caveat that it "is not a lossless representation of arbitrary JavaScript."

**Contrast with UniFFI, which pays nothing here.** The core carries UniFFI derives directly, so Apple and Android reach core types with no mirrors and no conversions at all. That asymmetry is a deliberate decision, not an oversight — see core-purity-and-io.md for the reasoning. UniFFI *additionally* supports *remote types* for crates that cannot carry its derives — "types defined in other crates that do not use UniFFI," needed "because of Rust's orphan rule," and solved with a mirrored **declaration** under `#[uniffi::remote(Record)]` rather than a duplicated type ([remote and external types](https://github.com/mozilla/uniffi-rs/blob/v0.32.1/docs/manual/src/types/remote_ext_types.md)) — but that mechanism is not needed under the default layout. NAPI-RS has neither option, which is why the Node side is the only one that mirrors.

## Type mapping

From [type conversions](https://napi.rs/docs/concepts/type-conversions) and [values](https://napi.rs/docs/concepts/values), verified 2026-09-19. Only the rows that matter for a domain boundary:

| Rust | TypeScript | Direction | Notes |
|---|---|---|---|
| `u32`, `i32`, `f64` | `number` | both | |
| `f32` | `number` | Rust → JS only | "there is no `FromNapiValue` implementation. Use `f64` for input" |
| `i64` | `number` | both | **"Values outside JavaScript's safe-integer range can lose precision"** |
| `i64n` | `bigint` | both | The wrapper form when you want a BigInt |
| `u64`, `u128`, `i128`, `usize`, `isize` | `bigint` | **Rust → JS only** | "Output-only to avoid silently narrowing arbitrary JavaScript BigInts". Needs `napi6` |
| `BigInt` | `bigint` | both | Needs `napi6`. Getters report whether narrowing was lossless |
| `String` | `string` | both | |
| `bool` | `boolean` | both | |
| `Option<T>` | `T \| null` | both | Accepts `T`, `null`, or `undefined`; returns `null` for `None` |
| `Vec<T>` | `Array<T>` | both | "Copies/converts every element" |
| `HashMap<String, T>` | `Record<string, T>` | both | |
| `#[napi(object)] struct` | `interface` | both | Owned plain-object shape, **not** a class |
| `#[napi] struct` | `class` | both | Use for native identity and methods |
| `#[napi] enum` | numeric `const enum` | both | |
| `#[napi(string_enum)] enum` | string `const enum` | both | |
| `#[napi] enum` with data | discriminated union, `type` discriminator | both | See below |

Two behaviours worth knowing because they cause surprise:

- **`#[napi(object)]` is cloned, not referenced.** "The `#[napi(object)]` struct passed to a Rust `fn` is cloned from the *JavaScript Object*. Any mutation on it will not be reflected in the original *JavaScript* object." Mutating a parameter is a silent no-op from JS's perspective.
- **`Option<T>` in an object field is an optional property by default** and `None` is omitted on output. `#[napi(use_nullable)]` makes it a required `T | null` instead. If the JS side does `'field' in obj` checks, pick deliberately.
- Field names are camelCased in the generated declaration (`dev_dependencies` → `devDependencies`).

### Integers: the `i64` trap

`i64` maps to `number`, not `bigint` — "`i64` deliberately maps to `number`, while `i64n` maps to `bigint`." So a core type carrying an `i64` timestamp in nanoseconds, or any ID above 2^53, **loses precision silently across this boundary**. Either use `i64n`/`BigInt` in the mirror type, or carry the value as a `String`. For inputs specifically, `u64` will not work at all (output-only); accept `BigInt` and check the lossless flag:

```rust
let (negative, narrowed, lossless) = value.get_u64();
if negative || !lossless { /* reject */ }
```

### Bytes: `Buffer` vs `Uint8Array`

Both exist and the choice is about lifetime, not taste:

| Type | JS | Use when |
|---|---|---|
| `Buffer` | Node.js `Buffer` | Async. "Keeps a reference to JavaScript-owned data and is designed for async use." Best lifecycle handling with `napi4` |
| `BufferSlice<'env>` | Node.js `Buffer` | Synchronous only — "Do not hold it across `await`" |
| `Uint8Array` | `Uint8Array` | Async; owned/reference-retaining wrapper |
| `Uint8ArraySlice<'env>` | `Uint8Array` | Synchronous borrowed view |
| `&[u8]` | typed array | JS → Rust **in a synchronous callback only**; cannot outlive the callback |

Prefer `Uint8Array` for a payload that is not Node-specific, and `Buffer` when the JS side is already Node-idiomatic. Use the owned forms (`Buffer`, `Uint8Array`) for anything that touches an `async fn` or a queue processor; the `*Slice` forms will not survive the await.

**Do not promise zero-copy.** External buffers "can be zero-copy when the runtime accepts external backing stores. A runtime may reject external buffers; constructors such as `BufferSlice::from_data` then fall back to a copy. Do not promise zero-copy behavior across every Node-compatible runtime."

## Errors: keep the variant

The core returns a typed enum; the default conversion throws it away. `napi::Error::from_reason(e.to_string())` produces `Status::GenericFailure` and a message, so JS sees `error.code === 'GenericFailure'` for every failure and has to parse strings to tell them apart.

The fix is documented: **`Error<S>` accepts any status type implementing `AsRef<str>`, and that sets `error.code`** ([error handling](https://napi.rs/docs/concepts/error-handling)).

```rust
pub struct CoreErrorCode(core::CoreError);

impl AsRef<str> for CoreErrorCode {
    fn as_ref(&self) -> &str {
        match self.0 {
            core::CoreError::NotFound { .. } => "ERR_NOT_FOUND",
            core::CoreError::Invalid { .. }  => "ERR_INVALID",
            core::CoreError::Cancelled       => "ERR_CANCELLED",
        }
    }
}

fn to_napi(e: core::CoreError) -> napi::Error<CoreErrorCode> {
    let message = e.to_string();
    napi::Error::new(CoreErrorCode(e), message)
}
```

The exported function then returns `napi::Result<T, CoreErrorCode>`. "The generated wrapper accepts the custom status because it only needs `AsRef<str>`." JS gets a stable `error.code` per variant plus the `Display` text as `error.message`.

Three related facts:

- The `Error` fields map as `reason` → `error.message`, `status.as_ref()` → `error.code`, `cause` → `error.cause` (set with `set_cause`, "nested causes are converted recursively").
- **`Status` is "primarily a Node-API status, not an application error taxonomy"** — which is exactly why a custom status type is the right tool for domain errors.
- **"TypeScript does not encode thrown exceptions or rejected Promises."** The generated `.d.ts` will not mention your error codes, so document them in JSDoc and assert their shape in a test. This is the one part of the surface the type generator cannot cover for you.

If a variant carries data the JS side needs to branch on structurally rather than by code, return it as a value instead: a `#[napi]` enum with data becomes a discriminated union (`{ type: 'FileChanged'; path: string }`, discriminator configurable via `discriminant`), which is often a better fit than an exception for expected outcomes.

Avoid `anyhow` on this boundary: the conversion "uses `Status::GenericFailure` and formats the anyhow error chain into the reason," so it re-loses the variant.

## Generated TypeScript declarations

`napi build` emits `index.js` (the loader) and `index.d.ts` (the declarations) alongside the `.node` artifact. Declaration generation requires the `typedef` feature; the CLI exposes `--dts <path>`, `--dts-header`, `--no-dts-header`, and `--dts-cache` (default `true`) ([build](https://napi.rs/docs/cli/build)).

**They are committed, not gitignored.** The maintained [package template](https://github.com/napi-rs/package-template) ignores `*.node` but checks in both `index.js` and `index.d.ts` — verified 2026-09-19. That is the right default here too: the declarations are part of the published package, consumers need them without a Rust toolchain, and committing them makes a drifted mirror type visible in review as a `.d.ts` diff. Treat the file as generated output: never hand-edit it, and regenerate on every interface change so the diff shows up in the same commit as the Rust change.

## Threadsafe functions — only if you truly need a callback

Prefer passing owned data in over installing a callback (see core-purity-and-io.md). If you do need one, the behaviour is controlled by generic parameters roughly `<Args, Return, CallJsBackArgs, ErrorStatus, CalleeHandled, Weak, MaxQueueSize>`.

The four things that matter:

- **Call modes.** `Blocking` waits for queue space; `NonBlocking` returns `Status::QueueFull` immediately when full. `MaxQueueSize` sets that capacity — this is your backpressure knob.
- **Keep the default `CalleeHandled: true`.** Rust `Err` becomes the JS callback's first argument, Node-style. Under `CalleeHandled: false`, "a synchronous throw in the JavaScript callback is routed through `napi_fatal_exception`, and a returned `Promise` is not awaited automatically." `napi_fatal_exception` tears down the process — in a queue worker that means losing every in-flight job.
- **Argument and return types must be `'static`.** Do not use scoped `Unknown<'env>`, `Object<'env>`, or `Function<'env, …>` as `CallJsBackArgs` — you get `E0521`. Convert to owned `String`/`Buffer`/owned struct first.
- **`Weak: true` does not guarantee delivery.** It only stops the TSFN from keeping the event loop alive; Node may exit before queued callbacks run.

## Binary distribution

NAPI-RS publishes one small root package plus one optional package per platform:

```
@scope/addon
@scope/addon-darwin-arm64
@scope/addon-linux-x64-gnu
@scope/addon-linux-x64-musl
@scope/addon-win32-x64-msvc
```

Each platform package holds one `.node` artifact and declares npm `os`, `cpu`, and where applicable `libc`. The root lists them as exact-version `optionalDependencies`, and the generated `index.js` loads the matching one. **musl vs glibc is handled by the `libc` field**, which is why there are separate `-gnu` and `-musl` packages — relevant if you deploy on Alpine.

This deliberately avoids shipping source (no toolchain needed on consumers) and `postinstall` downloads (no install-time network failures).

Node-API's ABI stability means a binary compiled against a Node-API level stays compatible with later Node releases providing that level. That is a different guarantee from the Node versions napi-rs CI actually exercises.

## Release pipeline and its traps

| Command | Responsibility |
|---|---|
| `napi create-npm-dirs` | Create one package directory per configured target |
| `napi build` | Build **one** target per invocation — CI runs it per matrix row |
| `napi artifacts` | Collect downloaded `.node` files into root and platform packages |
| `napi pre-publish` | Sync versions and optional deps, publish platform packages |
| `npm publish` | Publish the root package |

Traps, all documented by NAPI-RS:

- **"A multi-platform publication is not atomic."** npm versions are immutable, so a failure can leave platform packages published without the root. Treat release jobs as production changes.
- **Do not rely on `npm publish --dry-run`** — lifecycle scripts may still invoke `napi prepublish`, "which can publish the real platform packages." Use `npm pack --dry-run --ignore-scripts`.
- **The CLI warns and continues when an expected target file is missing.** Add an explicit CI gate asserting every configured target directory exists and holds exactly the expected artifact before publishing the root.
- `napi.targets` defines what gets *packaged*, not what one `napi build` compiles. Each target needs a config entry, an npm directory, and a CI job. "An accepted target triple alone is not a support guarantee."
- **npm has no atomic rollback.** Never republish different bits under the same version. Recovery is `napi prepublish -t npm` with unchanged artifacts.

## Service layout

```
node/
  addon/          # NAPI-RS crate wrapping the core
  src/
    queue/        # BullMQ wiring — see queue-bullmq.md
    push/         # APNs + FCM senders — see push-apns-fcm.md
    db/           # all persistence lives here, not in Rust
```

The addon is a dependency of the service, published or linked via a workspace. In development, `napi build` writes the `.node` locally and the generated loader prefers it.
