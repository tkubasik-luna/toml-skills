---
name: to-spec-v2
description: Workflow-accelerated spec — fuses to-prd-v2 and to-issues-v2 into a SINGLE human gate. After a grill, one workflow runs the module-design judge panel (PRD) and the multi-angle slice sweep (issues) in parallel over the shared codebase context, then you present both together for one approval and write the PRD + issue files. Use when user wants PRD and issues in one shot, mentions "to-spec" / "spec it", or wants to collapse the PRD→issues round-trips.
---

# To Spec (v2)

Collapses `to-prd-v2` + `to-issues-v2` into one fan-out and one approval. Same local file-based tracker: PRDs under `prd/`, issues under `issues/`. No GitHub, no Linear.

## Process

### 1. Up-front decisions (QCM)

Two decisions must be locked **before** the workflow runs. Passing them in as hard constraints is what keeps the PRD modules and the issue slices from diverging — otherwise the two halves invent different implementations and you pay a costly arbitration round-trip (the most expensive latency there is: a human turn).

Ask via `AskUserQuestion` (batch both in one call; if the grill already settled one, pre-fill it as the recommended option and just confirm):

1. **Tests** — A) all modules / B) selected modules (ask which) / C) none. Scopes the PRD's `Testing Decisions` section.
2. **Implementation strategy** — A) clean refactor (new modules / interfaces) / B) incremental (surgical edits to existing code). This decides what BOTH the module panel and the slice sweep encode, so they agree by construction.

Wait for the answers. Carry them — plus every decision locked during the grill — into the workflow as a `LOCKED DECISIONS (honor verbatim, do not re-litigate)` block.

### 2. Reuse the shared context

If `.workflow/context.md` exists at the repo root, read it — it holds the codebase area, conventions, prior art, and data/API surface from the grill-me-v2 recon. Use it as your codebase understanding; only explore directly for gaps it does not cover. Use the project's domain glossary vocabulary throughout.

### 3. Fan out PRD modules + issue slices in one workflow

Call the `Workflow` tool with the inline script below, passing `args` = `{ feature: "<feature summary>", locked: "<every locked grill decision + the strategy/tests choices from step 1, verbatim>", strategy: "<clean | incremental>", context: "<contents of .workflow/context.md>" }`. It runs both sub-pipelines concurrently and returns `{ modules, slices }`.

Launch it with `run_in_background: true` and, while it runs, **prefetch** so writing is instant on approval (see step 5): compute the next `prd/` and `issues/` sequence numbers and read the PRD + issue templates. This hides the prefetch behind the workflow's wall-clock.

```javascript
export const meta = {
  name: 'spec-prd-and-issues',
  description: 'One pass: module-design judge panel (PRD) + multi-angle slice sweep (issues) over one shared context',
  phases: [{ title: 'Propose' }, { title: 'Synthesize' }],
}

const feature = args.feature || 'the feature'
const ctx = args.context || ''
const locked = args.locked || '(none)'
const strategy = args.strategy || 'follow the codebase conventions'
const CONSTRAINTS = `LOCKED DECISIONS (honor verbatim — do NOT re-litigate, weaken, or "improve" them):\n${locked}\n\nIMPLEMENTATION STRATEGY (mandatory): ${strategy}`

const MODULE_ANGLES = [
  'MVP-first — smallest viable surface that still delivers the feature',
  'risk-first — isolate the volatile / uncertain parts behind their own modules',
  'deep-module — maximize functionality hidden behind simple, stable interfaces',
  'testability-first — modules whose external behavior is testable in isolation',
]
const SLICE_ANGLES = [
  'by user story — one thin slice per user story, each end-to-end',
  'by demoability — slices that each produce something visibly demoable',
  'by integration risk — slice so the riskiest end-to-end path is proven first (tracer bullet)',
  'by data flow — slices following data from its source through every layer to the UI',
]

const PROP = {
  type: 'object',
  properties: {
    modules: {
      type: 'array',
      items: {
        type: 'object',
        properties: {
          name: { type: 'string' },
          responsibility: { type: 'string' },
          interface: { type: 'string' },
          depth: { type: 'string' },
        },
        required: ['name', 'responsibility', 'interface'],
      },
    },
    rationale: { type: 'string' },
  },
  required: ['modules', 'rationale'],
}
const MODULES_OUT = {
  type: 'object',
  properties: {
    modules: {
      type: 'array',
      items: {
        type: 'object',
        properties: {
          name: { type: 'string' },
          responsibility: { type: 'string' },
          interface: { type: 'string' },
          whyDeep: { type: 'string' },
        },
        required: ['name', 'responsibility', 'interface'],
      },
    },
  },
  required: ['modules'],
}
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
          dependsOn: { type: 'array', items: { type: 'string' } },
        },
        required: ['title', 'type', 'whatToBuild', 'acceptance'],
      },
    },
    criticalPathDepth: { type: 'number', description: 'longest blocker chain length (the merge fills this; slicers may omit)' },
    maxParallelWidth: { type: 'number', description: 'largest set of slices with no dependency between them (the merge fills this)' },
  },
  required: ['slices'],
}

// PRD module panel — architects (opus, judgment) propose, judge synthesizes the deepest set
const modulesPromise = (async () => {
  const proposals = await parallel(MODULE_ANGLES.map((a, i) => () =>
    agent(`You are a software architect. Propose a module decomposition for this feature using the lens: ${a}.\nFavor DEEP modules (lots of functionality behind a simple, stable, testable interface).\n\nFeature:\n${feature}\n\n${CONSTRAINTS}\n\nCodebase context:\n${ctx}`, { label: `module:${i}`, phase: 'Propose', schema: PROP })
      .then(p => ({ angle: a, ...p }))))
  const valid = proposals.filter(Boolean)
  return agent(`Judge these ${valid.length} module decompositions. Score each module on depth (functionality vs interface width), testability in isolation, stability, and low coupling. Pick the strongest as base, graft superior modules from the others. Output the final module list — name, responsibility, public interface, why each is deep.\n\n${CONSTRAINTS}\nReject or rework any module that violates a locked decision or the mandatory strategy.\n\nProposals:\n${JSON.stringify(valid, null, 2)}`, { label: 'judge-modules', phase: 'Synthesize', schema: MODULES_OUT })
})()

// Issue slice sweep — slicers (sonnet, mechanical) fan out, merge (opus) dedups + orders
const slicesPromise = (async () => {
  const cands = await parallel(SLICE_ANGLES.map((a, i) => () =>
    agent(`Break this feature into TRACER-BULLET vertical slices using the lens: ${a}.\nEach slice is a thin but COMPLETE path through every layer (schema, API, UI, tests), demoable on its own. Prefer many thin slices. Mark each HITL or AFK.\nSlice the slices so they build the chosen implementation strategy. Slice ONLY net-new work: treat anything the context flags as already-shipped / neighbouring (e.g. its "In scope vs neighbouring" section) as DO-NOT-REBUILD — never emit a slice that re-implements a shipped feature.\n\nFeature:\n${feature}\n\n${CONSTRAINTS}\n\nCodebase context:\n${ctx}`, { label: `slice:${i}`, phase: 'Propose', model: 'sonnet', schema: SLICES })
      .then(r => ({ angle: a, slices: r.slices }))))
  const all = cands.filter(Boolean).flatMap(c => c.slices)
  return agent(`Merge these vertical-slice candidates into ONE deduplicated, dependency-ordered set. Drop duplicates, merge near-identical slices, keep granularity thin but not trivial. For each final slice give: title, type (HITL/AFK), what to build, acceptance criteria, and dependsOn referencing earlier slice titles. Flag any slice too coarse for a single tracer bullet.\n\n${CONSTRAINTS}\nDROP any candidate that re-implements an already-shipped / neighbouring feature, or that contradicts a locked decision; align every surviving slice with the mandatory strategy.\n\nOPTIMIZE FOR PARALLELISM (this is what lets the implementer fan out — a serial spine cannot be parallelized by any implementer): keep a dependsOn ONLY when the later slice genuinely needs the earlier slice's code, never for mere thematic ordering. Minimize the critical path (longest blocker chain) and maximize the count of mutually-independent slices — prefer a THIN shared foundation that then branches WIDE over a deep serial spine. Return criticalPathDepth (longest chain length) and maxParallelWidth (largest set of slices with no dependency between them). If the mandatory strategy inherently forces a deep spine, call that out.\nDISJOINT-FILES RULE: two slices truly run in parallel only if they touch DISJOINT files. If two otherwise-independent slices would edit the SAME file (e.g. a shared enum or registry), add a dependsOn to serialize them — nominal parallelism that conflicts at integration is worse than honest serialization. Where a shared file is the bottleneck, prefer a small upfront slice that lands the shared file once, then branch wide off it.\n\nCandidates:\n${JSON.stringify(all, null, 2)}`, { label: 'merge-slices', phase: 'Synthesize', schema: SLICES })
})()

const [modules, slices] = await Promise.all([modulesPromise, slicesPromise])
return { modules, slices }
```

### 4. One approval gate

Present BOTH results together in a single message:

- **Modules** (from `modules.modules`): name + responsibility + interface each.
- **Issue slices** (from `slices.slices`): numbered, each with title, type (HITL/AFK), depends-on, acceptance count.
- **Parallelism shape**: show `slices.criticalPathDepth` (serial depth) and `slices.maxParallelWidth` (how many issues can run at once), plus the dependency levels (level N = issues whose blockers all sit in earlier levels). If `criticalPathDepth` is high relative to the slice count — a near-serial spine — **flag it**: `implement-feature-v2` will barely parallelize it. Offer to re-slice flatter, or state plainly that the depth is inherent to the chosen strategy (e.g. a clean bottom-up refactor is naturally more serial than an incremental one).

Because strategy + locked decisions were fixed up-front (step 1), the two halves should already agree — present once. If modules and slices still describe different implementations, that is a workflow miss, not a user decision: **reconcile it yourself in the presentation** (align the slices to the chosen strategy) rather than spending a human round-trip on it. Only ask the user when a genuinely NEW fork emerged that the grill never settled.

Ask once: "Does this spec look right? (yes / adjust modules / adjust slices / abort)". Iterate only on what the user flags — do not re-run the whole workflow unless the feature itself changed. Wait for explicit approval before writing any files.

### 5. Write the files

Only after approval. **Branch guard first**: run `git rev-parse --abbrev-ref HEAD` and confirm it matches the branch recorded in `.workflow/context.md` (the `<!-- branch: … -->` line). If it differs — or you are unintentionally on `main`/`master` — STOP and confirm with the user before writing the PRD/issues onto the wrong branch. Then write immediately (you already have the next `<NNNN>` and the templates from the step-3 prefetch) — no fresh exploration.

1. **PRD** at `prd/<NNNN>-<slug>.md` — use the exact PRD template from `to-prd-v2` (Problem Statement, Solution, User Stories, Implementation Decisions with the approved modules, Testing Decisions per step 1, Out of Scope, Further Notes). `<NNNN>` = next zero-padded 4-digit after the highest existing `prd/` file.
2. **Issues** at `issues/<NNNN>-<slug>.md` — one file per approved slice, using the exact issue template from `to-issues-v2` (Parent → the PRD file, What to build, Acceptance criteria, Blocked by). Write in dependency order so `Blocked by` can reference real issue ids. Set `Parent` to the PRD written above.

Do NOT publish to any external tracker. Do NOT modify existing PRD/issue files.

### 6. Hand off

Tell the user the PRD + issue files written, and that `implement-feature-v2` can now pick up the PRD.
