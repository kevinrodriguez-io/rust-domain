# Agent instructions

This repository contains **rust-domain**, an agent skill for putting an app's domain entirely in Rust.

The skill lives in `.agents/skills/rust-domain/`. Its `SKILL.md` is the entry point; detailed per-surface material is in `references/`.

## Working in this repo

- The canonical skill content is `.agents/skills/` — edit it there and nowhere else.
- `.claude/skills` is a pointer to `.agents/skills` so Claude Code and Grok can discover the skill. Never edit content through the pointer.
- `CLAUDE.md` is a one-line import of this file. Keep it that way.
- `SKILL.md` frontmatter must use only the six Agent Skills spec fields (`name`, `description`, `license`, `compatibility`, `metadata`, `allowed-tools`). Vendor extensions such as `paths`, `when_to_use`, or `argument-hint` cause hard errors on Anthropic's upload path.
- Keep `SKILL.md` under 500 lines and keep every file reference one level deep from the skill root.
- `name` in `SKILL.md` must exactly match its parent directory name.

## Verifying a change

Validate the skill before committing, if `skills-ref` is available:

```bash
skills-ref validate .agents/skills/rust-domain
```

Otherwise check by hand that frontmatter parses, `name` matches the directory, and no reference link points more than one level deep.
