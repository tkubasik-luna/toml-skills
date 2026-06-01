---
name: to-prd-v2
description: Workflow-accelerated PRD. Same as to-prd, but decomposes the feature into modules via a parallel module-design judge panel (Workflow tool — N architects propose through different lenses, a judge synthesizes the deepest set) instead of a single-shot sketch. Writes a local Markdown PRD under `prd/`. Use when user wants the Workflow-accelerated PRD, mentions "to-prd v2", or wants a module-design panel.
---

This skill takes the current conversation context and codebase understanding and produces a PRD. Do NOT interview the user — just synthesize what you already know.

## Process

1. Before drafting the PRD, ask the user via a multiple-choice question whether they want tests implemented for this feature. Present the options:

   - A) Yes, write tests for all modules
   - B) Yes, but only for selected modules (ask which ones)
   - C) No tests for now

   Wait for the user's answer before proceeding. The answer determines whether the `Testing Decisions` section is filled in, scoped to specific modules, or omitted from the PRD.

2. **Reuse the shared context, do not re-explore.** If `.workflow/context.md` exists at the repo root, read it — it already holds the codebase area, conventions, prior art, and data/API surface from the grill-me-v2 recon. Use it as your codebase understanding. Only explore the repo directly for gaps the artifact does not cover (or if the file is absent). Use the project's domain glossary vocabulary throughout the PRD, and respect any ADRs in the area you're touching.

3. Decompose the feature into modules via a **module-design judge panel**, not a single-shot guess. A deep module (as opposed to a shallow module) is one which encapsulates a lot of functionality in a simple, testable interface which rarely changes — the panel optimizes for depth.

   Call the `Workflow` tool with the inline script below, passing `args` = `{ feature: "<feature summary + decisions so far>", context: "<contents of .workflow/context.md>" }`. The workflow has N architects each propose a decomposition through a different lens, then a judge picks the strongest base and grafts the best modules from the others. It returns `recommendation` (the final module list) + the raw `proposals`.

   ```javascript
   export const meta = {
     name: 'prd-module-panel',
     description: 'N architects propose module decompositions through different lenses, judge synthesizes the deepest set',
     phases: [{ title: 'Propose' }, { title: 'Judge' }],
   }

   const feature = args.feature || 'the feature'
   const ctx = args.context || ''
   const ANGLES = [
     'MVP-first — smallest viable surface that still delivers the feature',
     'risk-first — isolate the volatile / uncertain parts behind their own modules',
     'deep-module — maximize functionality hidden behind simple, stable interfaces',
     'testability-first — modules whose external behavior is testable in isolation',
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
             interface: { type: 'string', description: 'the simple public interface this module exposes' },
             depth: { type: 'string', description: 'why this is a deep (not shallow) module' },
           },
           required: ['name', 'responsibility', 'interface'],
         },
       },
       rationale: { type: 'string' },
     },
     required: ['modules', 'rationale'],
   }
   const proposals = await parallel(ANGLES.map((a, i) => () =>
     agent(`You are a software architect. Propose a module decomposition for this feature using the lens: ${a}.\nFavor DEEP modules (lots of functionality behind a simple, stable, testable interface).\n\nFeature:\n${feature}\n\nCodebase context:\n${ctx}`, { label: `propose:${i}`, phase: 'Propose', schema: PROP })
       .then(p => ({ angle: a, ...p }))))
   const valid = proposals.filter(Boolean)
   const recommendation = await agent(`Judge these ${valid.length} module decompositions for a PRD. Score each module on depth (functionality vs interface width), testability in isolation, stability (rarely changes), and low coupling. Pick the strongest decomposition as the base, then graft superior modules from the others. Output the final recommended module list — name, responsibility, public interface, and a one-line note on why each is a deep module.\n\nProposals:\n${JSON.stringify(valid, null, 2)}`, { label: 'judge', phase: 'Judge' })
   return { recommendation, proposals: valid }
   ```

   Present the workflow's `recommendation` to the user and check the modules match their expectations before drafting. Iterate if they push back.

4. Write the PRD using the template below as a local Markdown file at `prd/<NNNN>-<slug>.md` at the repo root. `<NNNN>` is a zero-padded 4-digit sequence number (next available after the highest existing file in `prd/`, starting at `0001`). `<slug>` is a kebab-case short title. Create the `prd/` directory if it does not exist. Do NOT publish to any external issue tracker (no GitHub, no Linear, etc.) — the project uses a local file-based tracker only.

<prd-template>

## Problem Statement

The problem that the user is facing, from the user's perspective.

## Solution

The solution to the problem, from the user's perspective.

## User Stories

A LONG, numbered list of user stories. Each user story should be in the format of:

1. As an <actor>, I want a <feature>, so that <benefit>

<user-story-example>
1. As a mobile bank customer, I want to see balance on my accounts, so that I can make better informed decisions about my spending
</user-story-example>

This list of user stories should be extremely extensive and cover all aspects of the feature.

## Implementation Decisions

A list of implementation decisions that were made. This can include:

- The modules that will be built/modified
- The interfaces of those modules that will be modified
- Technical clarifications from the developer
- Architectural decisions
- Schema changes
- API contracts
- Specific interactions

Do NOT include specific file paths or code snippets. They may end up being outdated very quickly.

## Testing Decisions

A list of testing decisions that were made. Include:

- A description of what makes a good test (only test external behavior, not implementation details)
- Which modules will be tested
- Prior art for the tests (i.e. similar types of tests in the codebase)

## Out of Scope

A description of the things that are out of scope for this PRD.

## Further Notes

Any further notes about the feature.

</prd-template>
