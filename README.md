# toml-skills

Claude Code plugin: end-to-end feature workflow skills.

## Skills

- **grill-me** — Stress-test plan/design via relentless interview until shared understanding.
- **grill-challenge** — Same as grill-me, with each recommendation peer-reviewed by an Opus architect sub-agent (maintainability + evolvability lens).
- **to-prd** — Convert current conversation context into a PRD under `prd/`.
- **to-issues** — Break PRD into independently-grabbable issues under `issues/` (tracer-bullet vertical slices).
- **implement-feature** — Implement PRD end-to-end. Dep graph from `Blocked by`, parallel subagents for independent issues, sequential for chained.

## Workflow

```
/grill-me          →  align on design
/grill-challenge   →  grill + Opus architect peer-review per recommendation
/to-prd            →  write prd/NNNN-feature.md
/to-issues         →  write issues/NNNN-*.md
/implement-feature →  ship it
```

## Install

```bash
# 1. Add marketplace (once)
/plugin marketplace add tkubasik-luna/toml-skills

# 2. Install plugin
/plugin install toml-skills@toml-skills
```

## Update

```bash
/plugin update toml-skills
```

## Modify locally

Plugin installs to `~/.claude/plugins/toml-skills/`. Edit files directly to test changes live.

Upstream changes via PR on this repo.
