# cross-platform-rust-core skill

An AI agent skill for bootstrapping or extending applications built on a **shared Rust core** exposed to native hosts:

- **Android** — Kotlin / Jetpack Compose via UniFFI
- **Apple** — Swift / SwiftUI via UniFFI
- **Node service** — NAPI-RS addon, BullMQ on Redis, APNs + FCM delivery

All version pins were verified against primary sources on **2026-09-19**.

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
    node-napi.md
    queue-bullmq.md
    push-apns-fcm.md
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

**In scope:** Rust core, UniFFI Kotlin + Swift bindings, Android and Apple integration, NAPI-RS Node addon, BullMQ queue architecture, APNs and FCM delivery.

**Deliberately out of scope:** WebAssembly and browser targets, Kotlin Multiplatform (and Gobley), third-party UniFFI generators, and BullMQ Pro. Each would pin the stack to older UniFFI versions or add a commercial dependency. Excluding them is what allows pinning UniFFI 0.32.1.
