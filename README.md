# cross-platform-rust-core skill

An AI agent skill for putting an app's **domain entirely in Rust** — one source of truth, tested with `cargo test`, and shared with native hosts:

- **Android** — Kotlin / Jetpack Compose via UniFFI
- **Apple** — Swift / SwiftUI via UniFFI

A Node service (NAPI-RS, BullMQ on Redis, APNs + FCM) is an optional expansion. The references for it stay in the skill and are used only when a server is added.

All version pins were verified against primary sources on **2026-09-19**.

## Why this architecture

The domain lives 100% in Rust. That is the goal. The hosts render and do IO.

- **Unified testability** — `cargo test` is the suite for the domain. No simulator, no emulator, no device farm, and no database. A fix in Rust is a fix on both platforms.
- **Proper sharing** — Android and iOS call the same functions. A later Node service calls that same core rather than growing another copy.
- **Single source of truth** — a rule exists in one place. A Kotlin or Swift copy of it has already drifted.
- **Native UI stays native** — SwiftUI stays SwiftUI and Compose stays Compose, which is what keeps the domain free of UI and IO so the tests stay plain Rust.

### Why not React Native or Expo

React Native and Expo share the **UI**. This shares the **logic** and deliberately does not share the UI. If your hard problem is shipping the same screens to both platforms quickly, React Native or Expo is the better tool and you should use it. If your hard problem is that the same non-trivial rules — sync, scheduling, pricing, crypto, offline reconciliation — have to behave *identically* on iOS and Android, then sharing the UI solves the wrong half. The same core can later serve a Node process, so those rules do not have to be rewritten in JavaScript either.

### What it costs

This is not free, and the costs are the reason not to adopt it casually:

- **It needs fluency in three stacks to start.** Rust for the core, Kotlin for Android, and Swift for Apple. A Node service is a fourth stack, and only if you expand. A team that cannot staff the three will struggle, and the FFI seam is precisely where thin expertise hurts most.
- **The FFI seam is a real maintenance tax.** Every type crossing it costs generated bindings, and on the Node side a mirrored type plus conversions. Version mismatches between the bindings generator and the scaffolding surface as runtime checksum errors rather than compile errors.
- **Native builds get more complicated.** An XCFramework and four Android ABIs have to be produced, pinned, and kept in step with the IDE projects.

That tax is why the seam is **coarse and narrow by design** — few, chunky functions that take owned data and return owned decisions, rather than a fine-grained API. The skill enforces that shape, because a chatty boundary multiplies every one of these costs.

### Non-goals

Stated explicitly, because the wrong expectation here is the expensive one:

- **Not write-once-run-anywhere.** Each platform still has a real app that a platform engineer builds.
- **Not a shared UI layer.** The UI is native per platform and is never shared. That is the point of the design, not a limitation of it.
- **Shared logic, not shared data.** The core holds no persistence, no filesystem, and no network — hosts own all IO. A database is IO, so it belongs to the host.

## Layout

The canonical content lives in one directory. Everything else is a pointer or a thin adapter.

```
AGENTS.md                                  # always-on instructions (canonical)
CLAUDE.md                                  # one-line "@AGENTS.md" import
.agents/skills/cross-platform-rust-core/
  SKILL.md                                 # entry point, spec-only frontmatter
  references/                              # per-surface detail, loaded on demand
    versions.md                            #   the pin table — read first
    rust-core-uniffi.md
    core-purity-and-io.md
    android-kotlin.md
    apple-swift.md
    node-napi.md                           # expansion
    queue-bullmq.md                        # expansion
    push-apns-fcm.md                       # expansion
    bootstrap.md
    extend-existing.md
.claude/skills -> ../.agents/skills        # pointer for Claude Code and Grok
```

**Note on `.claude/skills`:** this is a symlink in the source repository, but the GitHub API cannot create symlinks. If you cloned this repo and the directory is missing, recreate it:

```bash
ln -s ../.agents/skills .claude/skills
```

## Harness compatibility

| Harness | Instructions | Skill discovery |
|---|---|---|
| OpenAI Codex | `AGENTS.md` (native) | `.agents/skills/` (native) |
| Cursor | `AGENTS.md` (native) | `.agents/skills/` (native) |
| GitHub Copilot | `AGENTS.md` (native) | `.agents/skills/` (native) |
| Gemini CLI | needs `.gemini/settings.json` | `.agents/skills/` (native alias) |
| Claude Code | `CLAUDE.md` → `@AGENTS.md` | `.claude/skills` pointer |
| Grok Build | `AGENTS.md` (native) | `.claude/skills` pointer (Claude compat) |
| Zed, Cline, Amp, Factory, Warp | `AGENTS.md` (native) | — |

The `.claude/skills` symlink serves both Claude Code and Grok, since Grok reads Claude Code's skills directory with zero configuration.

**If your environment does not preserve symlinks** (Windows without `core.symlinks`, or some CI clones), replace the symlink with a generated copy of `.agents/skills/` plus a drift check in CI.

## Editing

Edit `.agents/skills/` only — never through the `.claude/skills` pointer.

`SKILL.md` frontmatter must use only the six Agent Skills spec fields (`name`, `description`, `license`, `compatibility`, `metadata`, `allowed-tools`). Vendor extensions such as `paths`, `when_to_use`, or `argument-hint` work in Claude Code on disk but cause **hard errors** on Anthropic's upload path. Scope by nesting instead.

Validate before committing:

```bash
skills-ref validate .agents/skills/cross-platform-rust-core
```

## Scope

**Starting scope:** Rust core, UniFFI Kotlin + Swift bindings, Android and Apple integration.

**Expansion, used as a guide when asked:** NAPI-RS Node addon, BullMQ queue architecture, APNs and FCM delivery.

**Deliberately out of scope:** WebAssembly and browser targets, Kotlin Multiplatform (and Gobley), third-party UniFFI generators, and BullMQ Pro. Each would pin the stack to older UniFFI versions or add a commercial dependency. Excluding them is what allows pinning UniFFI 0.32.1.
