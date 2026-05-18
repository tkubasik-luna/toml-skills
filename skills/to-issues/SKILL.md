---
name: to-issues
description: Break a plan, spec, or PRD into independently-grabbable issues, written as local Markdown files under `issues/` at the repo root, using tracer-bullet vertical slices. Use when user wants to convert a plan into issues, create implementation tickets, or break down work into issues.
---

# To Issues

Break a plan into independently-grabbable issues using vertical slices (tracer bullets).

## Process

### 1. Gather context

Work from whatever is already in the conversation context. The project uses a local file-based tracker — issues live as Markdown files under `issues/` at the repo root, and PRDs under `prd/`. If the user passes a reference (file path, issue number like `0042`, or slug), resolve it to the matching file under `issues/` or `prd/` and read its full content.

### 2. Explore the codebase (optional)

If you have not already explored the codebase, do so to understand the current state of the code. Issue titles and descriptions should use the project's domain glossary vocabulary, and respect ADRs in the area you're touching.

### 3. Draft vertical slices

Break the plan into **tracer bullet** issues. Each issue is a thin vertical slice that cuts through ALL integration layers end-to-end, NOT a horizontal slice of one layer.

Slices may be 'HITL' or 'AFK'. HITL slices require human interaction, such as an architectural decision or a design review. AFK slices can be implemented and merged without human interaction. Prefer AFK over HITL where possible.

<vertical-slice-rules>
- Each slice delivers a narrow but COMPLETE path through every layer (schema, API, UI, tests)
- A completed slice is demoable or verifiable on its own
- Prefer many thin slices over few thick ones
</vertical-slice-rules>

### 4. Quiz the user

Present the proposed breakdown as a numbered list. For each slice, show:

- **Title**: short descriptive name
- **Type**: HITL / AFK
- **Blocked by**: which other slices (if any) must complete first
- **User stories covered**: which user stories this addresses (if the source material has them)

Ask the user:

- Does the granularity feel right? (too coarse / too fine)
- Are the dependency relationships correct?
- Should any slices be merged or split further?
- Are the correct slices marked as HITL and AFK?

Iterate until the user approves the breakdown.

### 5. Write the issues as local files

For each approved slice, create a new Markdown file at `issues/<NNNN>-<slug>.md` at the repo root. `<NNNN>` is a zero-padded 4-digit sequence number (next available after the highest existing file in `issues/`, starting at `0001`). `<slug>` is a kebab-case short title. Create the `issues/` directory if it does not exist. Use the issue body template below as the file content.

Do NOT publish to any external issue tracker (no GitHub, no Linear, etc.) — the project uses a local file-based tracker only.

Write issues in dependency order (blockers first) so you can reference real file identifiers (`<NNNN>-<slug>`) in the "Blocked by" field.

<issue-template>
## Parent

A reference to the parent issue or PRD file (e.g. `prd/0003-feature-x.md` or `issues/0007-some-slice.md`) if the source was an existing local file. Otherwise omit this section.

## What to build

A concise description of this vertical slice. Describe the end-to-end behavior, not layer-by-layer implementation.

## Acceptance criteria

- [ ] Criterion 1
- [ ] Criterion 2
- [ ] Criterion 3

## Blocked by

- A reference to the blocking issue file (e.g. `issues/0004-some-slice.md`) if any.

Or "None - can start immediately" if no blockers.

</issue-template>

Do NOT modify or delete any parent issue or PRD file.
