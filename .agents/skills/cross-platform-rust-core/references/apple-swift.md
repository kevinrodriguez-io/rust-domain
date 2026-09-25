# Apple: Swift, XCFramework, Xcode

Verified 2026-09-19. Pins in versions.md.

**Ownership:** the human creates the Xcode project and target and adds the package through Xcode; the agent owns `Package.swift`, the XCFramework build, and generated Swift, and never edits `project.pbxproj` — see [bootstrap.md](bootstrap.md).

## Host tooling

**Use the user's Xcode.** Do not install a newer one, and do not refuse to build because it is not the version this skill was verified on (Xcode 27, 2026-09-19). Read `xcodebuild -version` and stay inside what that Xcode accepts.

If that Xcode is 27, it is Apple silicon only, requires macOS Tahoe 26.6+, rejects a macOS deployment target of 11 and watchOS 8, and will not debug a device or simulator below iOS 17. Those are limits of that Xcode. An older Xcode does not have them, and shipping to an older iOS still works on 27. Linux still cannot link Apple targets.

One useful split: *generating* the Swift bindings works on any host, because `uniffi-bindgen-swift` only emits `.swift`, `.h`, and `.modulemap` files. Only compiling the Rust staticlibs for Apple targets and assembling the XCFramework require macOS with Xcode. So the Swift API surface can be developed and reviewed on Linux; the link step cannot.

Set the Swift package's platform versions to the deployment target of the app the user created. The sample below uses iOS 15 and macOS 12 only as a placeholder.

Swift 6 language mode remains opt-in — modes 6, 5, 4.2, and 4 are available where the installed Xcode has them, so strict concurrency is a choice.

## Use `uniffi-bindgen-swift`, not `-l swift`

UniFFI ships a dedicated Swift binary that gives the control XCFramework packaging needs: separate header / modulemap / source generation, a single modulemap for a whole library, **XCFramework-compatible modulemaps**, and custom module names.

Add it as a second `[[bin]]` in your bindgen crate:

```rust
fn main() {
    uniffi::uniffi_bindgen_swift()
}
```

It always runs in library mode, so proc-macro-based generation always works.

```bash
# Swift sources
cargo run -p uniffi-bindgen-swift -- \
    target/release/libyourapp_core.a build/swift --swift-sources

# Headers
cargo run -p uniffi-bindgen-swift -- \
    target/release/libyourapp_core.a build/swift/Headers --headers

# XCFramework-compatible modulemap
cargo run -p uniffi-bindgen-swift -- \
    target/release/libyourapp_core.a build/swift/Modules \
    --xcframework --modulemap --modulemap-filename yourapp.modulemap
```

Note: UniFFI's `swift/xcode.md` doc page describes a `.udl`-based Xcode Build Rule flow. That page is stale guidance — UDL is deprecated. Prefer the `uniffi-bindgen-swift` page.

## Building the XCFramework

Targets you probably need:

```bash
rustup target add aarch64-apple-ios          # device
rustup target add aarch64-apple-ios-sim      # simulator, Apple silicon
rustup target add x86_64-apple-ios           # simulator, Intel
rustup target add aarch64-apple-darwin x86_64-apple-darwin   # macOS, if targeting it
```

Build each, `lipo` the same-platform slices together, then assemble:

```bash
lipo -create \
  target/aarch64-apple-ios-sim/release/libyourapp_core.a \
  target/x86_64-apple-ios/release/libyourapp_core.a \
  -output build/sim/libyourapp_core.a

xcodebuild -create-xcframework \
  -library target/aarch64-apple-ios/release/libyourapp_core.a -headers build/swift/Headers \
  -library build/sim/libyourapp_core.a -headers build/swift/Headers \
  -output build/YourAppCore.xcframework
```

**Do not `lipo` device and simulator slices together** — they are separate XCFramework entries. That is the most common error here.

### Tooling choices

| Option | Verdict |
|---|---|
| `uniffi-bindgen-swift` + a checked-in `lipo`/`xcodebuild` script | **Recommended.** First-party bindings, explicit and debuggable build, no third-party dependency |
| `cargo-swift` 0.11.1 | Maintained and reasonable if you want a turnkey Swift Package with no build scripts |
| The `xcframework` crate (`cargo xcframework`) | Alive but thin — 18 stars, single maintainer. Not recommended as a pipeline foundation |

**There is no crate named `cargo-xcframework`.** `cargo install cargo-xcframework` fails. The `cargo xcframework` subcommand comes from the crate named `xcframework`.

## Swift Package layout

```swift
// Package.swift
let package = Package(
    name: "YourAppCore",
    platforms: [.iOS(.v15), .macOS(.v12)],
    products: [.library(name: "YourAppCore", targets: ["YourAppCore"])],
    targets: [
        .binaryTarget(name: "YourAppCoreFFI", path: "build/YourAppCore.xcframework"),
        .target(name: "YourAppCore", dependencies: ["YourAppCoreFFI"], path: "Sources/YourAppCore"),
    ]
)
```

Generated `.swift` files go in `Sources/YourAppCore/`. Keep them out of version control and generate during the build.

## SwiftUI integration

Generated `async` functions are ordinary Swift `async` — call them from a `Task`. Keep the core behind an `@Observable` (or `ObservableObject`) view model rather than calling it from a `View` body.

Remember that **cancellation does not propagate**. A Swift `Task` that is cancelled will not stop the Rust future; the core keeps running until it finishes. Use the explicit `cancel()` pattern from rust-core-uniffi.md and treat `Task.isCancelled` as advisory on the Swift side only.

## Apple-specific UniFFI fixes worth knowing

Reasons not to go below the pinned version:

- `0.31.1` fixed an **iOS crash when address sanitizer is enabled**, and a memory leak in async code.
- `0.31.2` fixed `FfiConverterString` silently stripping a leading U+FEFF byte order mark from Rust strings, and a strict-concurrency warning on callback interface tables.

If you enable ASan and see crashes inside UniFFI glue, check the version first.

## Object lifetime

Generated Swift classes hold Rust handles and free them in `deinit`, so ARC normally handles cleanup. The failure mode is a **reference cycle between a Swift closure and a Rust object** — particularly with foreign trait implementations, where UniFFI explicitly does not help. Use `[weak self]` in closures captured by Rust-held objects.
