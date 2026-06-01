---
name: to-issues-v2
description: Workflow-accelerated issue breakdown. Same as to-issues, but generates tracer-bullet vertical slices via a parallel angle-sweep (Workflow tool — slicers from user-story / demoability / risk / data-flow lenses, then dedup + dependency-order merge) instead of a single-shot draft. Writes local Markdown issues under `issues/`. Use when user wants the Workflow-accelerated breakdown, mentions "to-issues v2", or wants a multi-angle slice sweep.
---

# To Issues

Break a plan into independently-grabbable issues using vertical slices (tracer bullets).

## Process

### 1. Gather context

Work from whatever is already in the conversation context. The project uses a local file-based tracker — issues live as Markdown files under `issues/` at the repo root, and PRDs under `prd/`. If the user passes a reference (file path, issue number like `0042`, or slug), resolve it to the matching file under `issues/` or `prd/` and read its full content.

### 2. Reuse the shared context, do not re-explore

If `.workflow/context.md` exists at the repo root, read it — it already holds the codebase area, conventions, prior art, and data/API surface from the grill-me-v2 recon. Use it instead of re-exploring; only explore directly for gaps it does not cover (or if absent). Issue titles and descriptions should use the project's domain glossary vocabulary, and respect ADRs in the area you're touching.

### 3. Draft vertical slices via an angle-sweep Workflow

Break the plan into **tracer bullet** issues. Each issue is a thin vertical slice that cuts through ALL integration layers end-to-end, NOT a horizontal slice of one layer.

Slices may be 'HITL' or 'AFK'. HITL slices require human interaction, such as an architectural decision or a design review. AFK slices can be implemented and merged without human interaction. Prefer AFK over HITL where possible.

<vertical-slice-rules>
- Each slice delivers a narrow but COMPLETE path through every layer (schema, API, UI, tests)
- A completed slice is demoable or verifiable on its own
- Prefer many thin slices over few thick ones
</vertical-slice-rules>

Rather than slicing single-shot, fan out candidate slicings from several angles and merge them — different lenses catch slices a single pass misses. Call the `Workflow` tool with the inline script below, passing `args` = `{ plan: "<full plan / PRD text>", context: "<contents of .workflow/context.md>" }`. It returns `proposal` — one deduplicated, dependency-ordered slice set — which you then take into step 4 to quiz the user.

```javascript
export const meta = {
  name: 'issue-slice-sweep',
  description: 'Generate vertical-slice candidates from multiple angles, dedup, propose a dependency-ordered set',
  phases: [{ title: 'Slice' }, { title: 'Merge' }],
}

const plan = args.plan || ''
const ctx = args.context || ''
const ANGLES = [
  'by user story — one thin slice per user story, each end-to-end',
  'by demoability — slices that each produce something visibly demoable',
  'by integration risk — slice so the riskiest end-to-end path is proven first (tracer bullet)',
  'by data flow — slices following data from its source through every layer to the UI',
]
const SLICES = {
  type: 'object',
  properties: {
    slices: {
      type: 'array',
      items: {
        type: 'object',
        properties: {
          title: { type: 'string' },
          type: { type: 'string', enum: ['HITL', 'AFK'] },
          whatToBuild: { type: 'string' },
          acceptance: { type: 'array', items: { type: 'string' } },
          dependsOn: { type: 'array', items: { type: 'string' }, description: 'titles of slices that must finish first' },
        },
        required: ['title', 'type', 'whatToBuild', 'acceptance'],
      },
    },
  },
  required: ['slices'],
}
// slicers are mechanical fan-out (run on sonnet); the merge is judgment (inherits opus)
const candidates = await parallel(ANGLES.map((a, i) => () =>
  agent(`Break this plan into TRACER-BULLET vertical slices using the lens: ${a}.\nEach slice is a thin but COMPLETE path through every layer (schema, API, UI, tests), demoable on its own. Prefer many thin slices over few thick ones. Mark each HITL (needs a human decision/design) or AFK.\n\nPlan:\n${plan}\n\nCodebase context:\n${ctx}`, { label: `slice:${i}`, phase: 'Slice', model: 'sonnet', schema: SLICES })
    .then(r => ({ angle: a, slices: r.slices }))))
const valid = candidates.filter(Boolean)
const all = valid.flatMap(c => c.slices)
const proposal = await agent(`Merge these vertical-slice candidates (from ${valid.length} angles) into ONE deduplicated, dependency-ordered set. Drop duplicates, merge near-identical slices, keep granularity thin but not trivial. For each final slice give: title, type (HITL/AFK), what to build, acceptance criteria, and "blocked by" referencing earlier slice titles. Flag any slice too coarse to be a single tracer bullet.\n\nCandidates:\n${JSON.stringify(all, null, 2)}`, { label: 'merge', phase: 'Merge' })
return { proposal, rawCount: all.length, angles: valid.length }
```

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
