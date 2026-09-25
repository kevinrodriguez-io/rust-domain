# Android: Kotlin, Compose, cargo-ndk

Verified 2026-09-19. Pins in versions.md.

**Ownership:** the human creates the Android Studio project and app module; the agent owns `*.gradle.kts`, the `cargo-ndk` build, and generated Kotlin, and never edits `.idea/` — see [bootstrap.md](bootstrap.md).

## AGP 9 changed the rules — read this before copying any older guide

AGP 9.0 (January 2026) introduced breaking changes that invalidate most pre-2026 Android+Rust material.

- **Built-in Kotlin is on by default.** Do **not** apply `org.jetbrains.kotlin.android` or `kotlin-android` — that plugin "is not compatible with the new DSL." AGP has a runtime dependency on KGP 2.2.10 and auto-upgrades lower versions.
- **`kotlinOptions` → `compilerOptions`.**
- **kapt** → use KSP, or `com.android.legacy-kapt` (versioned with AGP).
- **`targetSdk` now defaults to `compileSdk`**, not `minSdk`. Set it explicitly so an implicit bump does not change runtime behaviour.
- A library's consumers must use the same or higher compile SDK by default; override with `AarMetadata.minCompileSdk` if you publish.
- Escape hatches `android.builtInKotlin=false` and `android.newDsl=false` exist but are **scheduled for removal in AGP 10**. The new Variant API becomes mandatory in AGP 10.

`minSdk` note: D8/R8 no longer supports compiling a library with `default`/`static`/`private` interface methods to DEX at `minSdk < 24` with rules that keep those methods. `minSdk 24` avoids the problem.

## Building the Rust library

Add the four Android targets once:

```bash
rustup target add aarch64-linux-android armv7-linux-androideabi \
                  x86_64-linux-android i686-linux-android
```

Install `cargo-ndk` with `cargo binstall cargo-ndk` to avoid compiling it in CI.

Build into the Gradle `jniLibs` layout:

```bash
cargo ndk -t arm64-v8a -t armeabi-v7a -t x86_64 -t x86 \
  -o android/src/main/jniLibs build --release
```

**Match the NDK to the user's project.** `cargo-ndk` "will find the most recent NDK version and use it" unless told otherwise. Left alone, it can pick a newer NDK than the Android Gradle Plugin in the project the user created. Set `ndkVersion` and `ANDROID_NDK_HOME` to the NDK that AGP already expects. Do not upgrade Android Studio or AGP to chase the version in versions.md.

## Gradle wiring

UniFFI has **no official Gradle plugin**. The snippet in UniFFI's own docs is unusable as-is on a current project: it is written against the deprecated UDL flow (`generate <PATH TO .udl FILE>`) *and* the `android.libraryVariants` API that AGP 9's new Variant API removes.

Write the task against library mode and `androidComponents` instead. Shape:

1. A task that runs `cargo ndk … build --release` to produce `jniLibs`.
2. A task that runs your in-workspace bindgen against the built library with `--language kotlin`, outputting into a generated source directory.
3. Register that directory as a Kotlin source dir, and make Kotlin compilation depend on both tasks.

Keep generated Kotlin out of version control.

## Required dependencies

```kotlin
dependencies {
    implementation("net.java.dev.jna:jna:5.19.1@aar")          // note @aar
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.11.0")
}
```

The `@aar` classifier matters — the plain JAR does not carry the Android native libraries. UniFFI requires JNA ≥ 5.12.0 and coroutines ≥ 1.6.

## The JNA performance cliff — mandatory mitigation

UniFFI uses JNA direct mapping (since 0.30.0). When Rust threads call **into** Kotlin, JNA must attach and detach a Java thread per call, which UniFFI's docs call "a heavy operation."

Cache the `JavaVM` once and attach your runtime's worker threads permanently:

```rust
static VM: once_cell::sync::OnceCell<jni::JavaVM> = once_cell::sync::OnceCell::new();

// Rust 1.85+ requires the `unsafe(...)` form. You are on 1.98.
#[unsafe(export_name = "Java_com_example_app_MainActivity_nativeInit")]
pub extern "system" fn native_init(env: jni::JNIEnv, _class: jni::objects::JClass) {
    let vm = env.get_java_vm().unwrap();
    let _ = VM.set(vm);
}
```

```rust
tokio::runtime::Builder::new_multi_thread()
    .on_thread_start(|| {
        VM.get().expect("init java vm")
            .attach_current_thread_permanently()
            .unwrap();
    })
    .build()
    .unwrap();
```

Kotlin side:

```kotlin
class MainActivity : ComponentActivity() {
    external fun nativeInit()
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        System.loadLibrary("yourapp_core")
        nativeInit()
    }
}
```

The `export_name` must match `Java_<package_with_underscores>_<Class>_<method>`.

This only matters for Rust→Kotlin calls, which is another reason to prefer the pure-core design in core-purity-and-io.md: with command/result there are far fewer of them.

## Compose

Use the BOM and the Kotlin-shipped compiler plugin:

```kotlin
plugins {
    id("org.jetbrains.kotlin.plugin.compose") version "2.4.20"
}

dependencies {
    implementation(platform("androidx.compose:compose-bom:2026.09.00"))
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.material3:material3")
}
```

Never reference `androidx.compose.compiler:compiler` — frozen at 1.5.15 since 2024-08-07.

Keep the core out of composables: the generated Kotlin API should be called from a ViewModel, with results exposed as state. UniFFI `suspend fun`s integrate with coroutines normally.

## Object lifetime

Generated Kotlin objects hold Rust handles (a `u64` since 0.30.0) and implement `AutoCloseable`. Call `.close()`, or use `.use { }`, or tie the lifetime to a ViewModel's `onCleared()`. Leaked objects leak Rust memory. `uniffiIsDestroyed` reports whether the Rust reference is gone.

## Known Android-specific UniFFI bugs

Both are reasons `0.32.1` is the floor:

- `0.32.0` had a **checksum failure on aarch64**, fixed in `0.32.1`.
- `0.31.2` fixed JNA direct-mapped `u8`/`u16` return values on ARM32, where signedness mismatches broke checksum validation.

If you see checksum errors on a device but not an emulator, check the UniFFI version before anything else.
