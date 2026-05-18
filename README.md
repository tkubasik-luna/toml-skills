# tom-feature-workflow

Claude Code plugin: end-to-end feature workflow.

## Skills

- **grill-me** — Stress-test plan/design via relentless interview until shared understanding.
- **to-prd** — Convert current conversation context into a PRD under `prd/`.
- **to-issues** — Break PRD into independently-grabbable issues under `issues/` (tracer-bullet vertical slices).
- **implement-feature** — Implement PRD end-to-end. Dep graph from `Blocked by`, parallel subagents for independent issues, sequential for chained.

## Workflow

```
/grill-me        →  align on design
/to-prd          →  write prd/NNNN-feature.md
/to-issues       →  write issues/NNNN-*.md
/implement-feature → ship it
```

## Install

```bash
# 1. Add marketplace (once)
/plugin marketplace add <git-url-or-local-path>

# 2. Install plugin
/plugin install tom-feature-workflow@tom-feature-workflow
```

## Update

```bash
/plugin update tom-feature-workflow
```

## Modify locally

Plugin installs to `~/.claude/plugins/tom-feature-workflow/`. Edit files directly to test changes live.

Upstream changes via PR on this repo.
