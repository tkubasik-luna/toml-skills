---
name: grill-me-v2
description: Workflow-accelerated grill. Same relentless feature interview as grill-me, but first fans out parallel read-only explorers (Workflow tool) to map the codebase, prior art, constraints, data surface, and surface decision branches before grilling. Ends with a final recap. Use when user wants the Workflow-accelerated grill, mentions "grill me v2", or wants a pre-grill recon fan-out.
---

Goal: grill the user about the **feature** only. No implementation talk, no dev planning, no "next steps". Output is a shared understanding of the feature + a final recap.

Forced flow (do not skip, do not add steps):

1. **UPFRONT INPUTS (QCM).** Ask via `AskUserQuestion` whether the user has data / files / text to provide upfront (specs, mockups, logs, schemas, PRD, transcripts, screenshots, dumps). Options: "Yes, files" / "Yes, pasted text" / "Yes, both" / "No, nothing". If ≠ "No": wait for inputs, Read each file fully, parse pasted text, extract key facts (constraints, locked decisions, actors, deadlines, exact names), and KEEP all info from those inputs in context for later questions. Short recap (3–6 bullets) of what was retained, then proceed to step 2. If "No": proceed directly.

2. **CLAUDE DESIGN / FIGMA (QCM).** Ask via `AskUserQuestion` if user has a Claude Design / Figma link for the feature. Options: "Yes, Figma selection link" / "Yes, full Figma file" / "Yes, multiple nodes" / "No design". If ≠ "No": wait for link(s)/node-id(s), then use Figma MCP tools (`get_design_context`, `get_screenshot`, `get_variable_defs`, `get_code_connect_map`, `get_metadata`) to extract everything needed to understand the **feature**: screen structure, components, variants, design tokens (colors, typo, spacing, radii), variables, assets, interactive states, layout constraints, screenshots. Store in MD files under `grill-me/design/` at repo root:
   - `grill-me/design/overview.md` — screen/node list, summary capture, source link
   - `grill-me/design/tokens.md` — colors, typo, spacing, radii, elevations (exact values)
   - `grill-me/design/components.md` — components used, variants, props
   - `grill-me/design/screens/{screen-name}.md` — one file per screen: hierarchy, states, interactions, required assets, referenced screenshot
   - `grill-me/design/assets.md` — icons/images referenced (node-id + format)

   Short recap (5–10 bullets) of stored design. Keep MDs referenced in context for later questions. If "No": proceed directly.

3. **PRE-GRILL RECON + shared context artifact.** Before grilling, map what exists so the grill starts informed. Read-only — never edits code. Produces the **shared context artifact `.workflow/context.md`** that every later phase (to-prd-v2, to-issues-v2, implement-feature-v2) reuses instead of re-exploring.

   **Cache check first.** Run `git rev-parse HEAD` for the current SHA and `git rev-parse --abbrev-ref HEAD` for the working branch (record both). If `.workflow/context.md` exists and its first line is `<!-- sha: <current-sha> -->`, the codebase has not moved — **skip recon**, read the file, reuse it. Only recon on a cache miss (no file, or SHA differs).

   **Pick the recon mode by topic shape:**
   - **Targeted** — the topic names concrete symbols / files / a specific bug (e.g. "the JwtProcessingHandler that turns the QR into ScanInfo"). Do a FAST inline locate yourself: Glob/Grep the named symbols, Read the entry point + its callers + the result/error types. Seconds, not minutes, and it lands exactly on the subject. This is your primary grounding for the grill. A broad lens-sweep on a symbol-named topic tends to drift onto neighbouring features — **do not lead with it**.
   - **Broad** — the topic is vague / greenfield (e.g. "improve onboarding"). There is no symbol to anchor on, so run the lens-sweep Workflow below; breadth wins.

   **Overlap recon with the grill to hide its latency — quality-safe, never quality-lossy.** Recon compute runs behind the human's thinking time, not in front of it:
   - In targeted mode, the inline locate is fast: do it, OPTIONALLY launch the broad lens-sweep with `run_in_background: true` to enrich `.workflow/context.md`, and **proceed to the grill immediately**. Fold any background findings into later rounds when they land.
   - **Hard rule that protects grill quality: never fire a grill question uninformed.** Recon-independent product/behaviour forks (what the user wants to happen — trust, precedence, copy, success criteria) can be asked while recon runs. Any branch whose *framing* depends on the codebase waits until you have the relevant finding. Overlapping only **reorders** work — it never skips grounding, never drops a branch, never trims questions.

   On a broad-mode miss (or to enrich a targeted run), call `Workflow` with the inline script below, passing `args` = `{ topic: "<feature in one line>", context: "<key facts retained from upfront inputs + design>" }`. Launch it with `run_in_background: true` so the grill can start in parallel.

   ```javascript
   export const meta = {
     name: 'pre-grill-recon',
     description: 'Fan out read-only explorers before grilling: codebase map, prior art, constraints, data surface, decision branches',
     phases: [{ title: 'Recon' }, { title: 'Synthesize' }],
   }

   const topic = args.topic || 'the feature'
   const ctx = args.context || ''
   const LENSES = [
     { key: 'area', q: `Map where "${topic}" would live in this codebase: relevant modules, folders, existing files, entry points.` },
     { key: 'prior-art', q: `Find prior art for "${topic}": similar features already shipped, reusable patterns, components, conventions to follow.` },
     { key: 'constraints', q: `Extract constraints affecting "${topic}": CLAUDE.md rules, ADRs, layering boundaries, naming, locked decisions.` },
     { key: 'data', q: `Map the data/schema surface "${topic}" touches: existing tables, models, DTOs, endpoints, events.` },
     { key: 'branches', q: `Surface open FEATURE decision branches for "${topic}" a designer must resolve: scope edges, states, edge cases, copy, success criteria. Frame each as a question. Stay out of implementation.` },
   ]
   const FIND = {
     type: 'object',
     properties: {
       findings: { type: 'array', items: { type: 'string' } },
       files: { type: 'array', items: { type: 'string' } },
       openQuestions: { type: 'array', items: { type: 'string' } },
     },
     required: ['findings'],
   }
   // recon lenses run as read-only Explore agents (lightweight); synthesis on sonnet (mechanical write-up)
   const recon = await parallel(LENSES.map(l => () =>
     agent(`${l.q}\n\nIf "${topic}" names concrete symbols / files, LOCATE and READ them first, and ground every finding in a real file path. Stay strictly on THIS subject — do NOT drift onto neighbouring or already-shipped features, even if they look similar.\n\nProject context: ${ctx}`, { label: `recon:${l.key}`, phase: 'Recon', agentType: 'Explore', schema: FIND })
       .then(r => ({ lens: l.key, ...r }))))
   const valid = recon.filter(Boolean)
   const SYNTH = {
     type: 'object',
     properties: {
       context: { type: 'string', description: 'full shared codebase context (markdown), written to .workflow/context.md' },
       recap: { type: 'array', items: { type: 'string' }, description: '5-8 bullet recap for the user — readable without the full context' },
       addressesTopic: { type: 'boolean', description: 'false if the recon drifted off the topic\'s named subject onto other features' },
     },
     required: ['context', 'recap', 'addressesTopic'],
   }
   const synth = await agent(`Write the SHARED CODEBASE CONTEXT for the feature "${topic}", reusable by every downstream phase (PRD, issue breakdown, implementation). Sections: ## Codebase area (modules/folders/files), ## Conventions to respect (from CLAUDE.md/ADRs/layering), ## Prior art to reuse, ## Data & API surface, ## In scope vs neighbouring (flag already-shipped neighbours that downstream MUST NOT rebuild). Be concrete, cite real file paths. Durable reference, NOT feature decisions.\n\nThen self-check: does this context actually address "${topic}" and its named symbols? Set addressesTopic=false if the recon drifted onto other / already-shipped features. Also return a 5-8 bullet recap.\n\nRaw recon:\n${JSON.stringify(valid, null, 2)}`, { label: 'context', phase: 'Synthesize', model: 'sonnet', schema: SYNTH })
   const openQuestions = valid.flatMap(r => r.openQuestions || [])
   return { context: synth.context, recap: synth.recap, addressesTopic: synth.addressesTopic, openQuestions, recon: valid }
   ```

   When the background recon completes (you are notified), **relevance-gate it**: if `addressesTopic` is false — or the synthesis never mentions the topic's named symbols — it drifted; **discard it** and keep the targeted inline locate as your context instead (never let a drifted sweep overwrite a good targeted read). Otherwise write `context` to `.workflow/context.md` at the repo root with a first line `<!-- sha: <current-sha> -->` and a second line `<!-- branch: <working-branch> -->` (create `.workflow/` if missing). The recorded branch is the session's working branch — every downstream phase (to-spec-v2, implement-feature-v2) cross-checks it before writing files or launching a worktree workflow, so a run never lands on the wrong branch. Use the returned `recap` for the user-facing bullets (no need to re-read the full payload; the workflow result is large and truncates in the notification — Read the output file only when you need the full `context` to write the artifact). Carry `openQuestions` into the grill, folding them into the rounds that have not been asked yet.

4. **TRIAGE (QCM).** Judge feature complexity from the recon: count open decision branches and whether any are architectural/irreversible forks. Ask via `AskUserQuestion`:
   - **Grill complet** — walk every branch interactively (default for ≥ ~5 open branches or any architectural fork).
   - **Fast-path** — few/no real forks: skip the interview, go straight to step 6 recap using recommended answers for each branch; the user validates the recap in one shot.

   Recommend the option that fits the branch count. If the user picks fast-path, jump to step 6.

5. **GRILL THE FEATURE.** Interview the user relentlessly until shared understanding is reached. Walk each branch of the feature decision tree, resolving dependencies one-by-one. Seed the branches from the recon `openQuestions`. If recon is still running in the background (targeted-mode overlap), open with the recon-independent product/behaviour forks while it finishes, then fold its `openQuestions` into later rounds as they land — but never ask a code-dependent question before its grounding has arrived. For each question, provide your recommended answer. Stay focused on the feature: user value, behavior, scope, edge cases, states, copy, data needs, success criteria. **Never** drift into implementation, architecture, tickets, dev planning, estimates, or "next steps".

   **Batch independent branches to cut human round-trips.** Branches with no dependency between them can be asked together: emit up to 4 in a single `AskUserQuestion` call (one question per branch, each with its recommended answer pre-selected). Keep **dependent** branches sequential — an answer that changes which branches remain must be resolved before asking what it unblocks. If a question can be answered from `.workflow/context.md` or the codebase, answer it yourself instead of asking.

6. **FINAL RECAP.** When the grill is done (no more open branches, user confirms shared understanding), output a single recap covering:
   - **Feature summary** — 2–4 lines stating what the feature is.
   - **Decisions made** — every resolved branch as a one-line bullet (`question → chosen answer`).
   - **Open points** — any branch left unresolved with the reason it was parked.
   - **Constraints & assumptions** — locked facts gathered from upfront inputs + design.

   Do not propose tasks, tickets, dev steps, or what to do next. Recap only.
