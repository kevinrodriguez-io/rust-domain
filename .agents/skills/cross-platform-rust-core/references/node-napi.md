# Node service: NAPI-RS

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

## Exporting the core

Keep the addon a thin translation layer. The core stays pure; the addon converts owned types and nothing else.

```rust
use napi_derive::napi;

#[napi]
pub fn plan_notifications(devices: Vec<DeviceRecord>, events: Vec<DomainEvent>) -> napi::Result<Vec<SendCommand>> {
    yourapp_core::plan_notifications(devices, events)
        .map_err(|e| napi::Error::from_reason(e.to_string()))
}
```

`async fn` works and maps to a JS `Promise`. Enable the `tokio_rt` / `async` features on the `napi` crate if the core needs a Tokio runtime.

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
