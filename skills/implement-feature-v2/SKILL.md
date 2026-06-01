---
name: implement-feature-v2
description: Workflow-accelerated feature implementation. Same as implement-feature, but runs the AFK issues as a per-issue DAG in stacked parallel git worktrees via the Workflow tool — every issue starts the moment its blockers finish (no level barrier), with adversarial verify per issue — while HITL issues + commits stay in the main thread. Implements from a local PRD and its issues under `prd/` and `issues/`. Use when user wants the Workflow-accelerated implementation, mentions "implement-feature v2", or wants max-parallel worktree fan-out.
---

# Implement Feature

End-to-end implementation driver for the local file-based tracker (PRDs in `prd/`, issues in `issues/` at the repo root). No GitHub, no Linear.

## Preconditions

Before doing anything else, verify the local tracker is set up:

1. **`prd/` directory exists and contains at least one `*.md` file.** If missing or empty, abort and tell the user:

   ```
   Aucun PRD trouvé. Il faut au moins un fichier `prd/<NNNN>-<slug>.md`.
   Lance `/to-prd` pour en créer un.
   ```

2. **`issues/` directory exists and contains at least one `*.md` file.** If missing or empty, abort and tell the user:

   ```
   Aucune issue trouvée. Il faut au moins un fichier `issues/<NNNN>-<slug>.md`.
   Lance `/to-issues` pour découper le PRD en issues.
   ```

3. **At least one issue must reference an existing PRD via its `Parent` field.** If no issue maps to any PRD, abort and tell the user which PRDs have zero linked issues, and suggest running `/to-issues` on those.

Only proceed to the picker once all three checks pass. Be explicit about which file/folder is missing — never silently skip.

## Inputs

**Always start with a multiple-choice picker, regardless of arguments and regardless of how many PRDs exist.** Even when only one PRD is available, still present it as a QCM with that single option plus an "Abort" option — never auto-select.

Build the picker by listing every file matching `prd/*.md` at the repo root, sorted by `<NNNN>`. For each entry, show:

- The id `<NNNN>`
- The PRD title (first `# Heading` or the slug)
- Issue progress: `X / Y issues implemented` based on how many issues whose `Parent` references this PRD already have a corresponding commit on the current branch (best-effort — fall back to `0 / Y` if uncertain)

Append two trailing options:

- **S) Implement an explicit set of issues instead** — then prompt for issue refs (file paths or `<NNNN>` ids)
- **X) Abort**

If the user passed a PRD or issue argument when invoking the skill, still show the QCM but pre-highlight that argument as the suggested choice (e.g. `1) [suggested] 0003 — ...`). The user must still confirm by typing the letter/number.

After the user picks, resolve the selection to absolute file paths under `prd/` or `issues/`. Abort with a clear message if a reference can't be resolved.

## Process

### 1. Load the work set

- Read the PRD (if provided) in full.
- Read every issue file in scope. If a PRD is given, scope = all issues whose `Parent` references that PRD (or the PRD's slug).
- Extract for each issue: `<NNNN>-<slug>` id, title, `Blocked by` list, `HITL`/`AFK` type, acceptance criteria.

### 2. Build the dependency graph

- Nodes = issues. Edges = `Blocked by` references.
- Detect cycles. If found, stop and report the cycle to the user.
- Compute a topological ordering grouped into **levels**: level N contains all issues whose blockers are all in levels < N. Level 0 = no blockers.
- HITL issues always pause the loop at their level: report the HITL issue to the user, request the decision/review/input it asks for, and wait for explicit user confirmation before continuing past it.

### 3. Confirm the execution plan

Show the user:

- Total issues, count per level, count of HITL vs AFK
- The full ordered plan: for each level, list `<NNNN>-<slug>` + title + type
- Which levels will run in parallel (level with > 1 AFK issue and no HITL)

Ask: "Proceed with this plan? (yes / adjust / abort)". Wait for explicit approval before any code changes.

### 4. Execute — AFK fan-out via Workflow, HITL in main thread

The heavy lifting (independent issues implemented + verified in parallel) is delegated to the **Workflow tool**. Interactive HITL steps and all git commits stay in the main thread — the Workflow tool has no human-in-loop and concurrent git commits race, so neither belongs inside a workflow.

**4.0 — Branch guard (capture the session branch + base commit).** Before anything else, capture where the session is working and make it the source of truth for the whole run:

- `BRANCH = git rev-parse --abbrev-ref HEAD` and `BASE_SHA = git rev-parse HEAD`.
- If `BRANCH` is `HEAD` (detached), or `main`/`master` when the feature clearly should land elsewhere, STOP and confirm with the user before any code change.
- Cross-check against the branch recorded in `.workflow/context.md` (the `<!-- branch: … -->` line written by grill-me-v2). If they differ, STOP and ask which is correct — never guess.

`BRANCH` + `BASE_SHA` are passed into the workflow (`args.branch`, `args.baseSha`) and `BRANCH` is re-asserted before every commit. **Why this matters (the bug this prevents):** `git worktree add` cannot check out a branch that is already checked out in the main worktree, so the worktree tool may silently base an agent's worktree on `main` instead of the session branch — the agent then implements against the wrong code. Passing `BASE_SHA` (a commit ref, not a branch — it sidesteps the already-checked-out restriction) lets each worktree realign to the intended commit before it does any work.

**4.1 — Partition.** Split the scoped issues into AFK and HITL. An AFK issue is *workflow-ready* when its entire transitive blocker-closure is AFK and already done (or contains only other workflow-ready AFK issues). An AFK issue that is transitively blocked by an unresolved HITL issue is **deferred** until that HITL resolves.

**4.2 — Pre-warm the build cache.** Before fanning out, run the project's build once in the main tree (`cd <sub-project> && ./gradlew build` or the assemble task per `CLAUDE.md`). The per-chain worktrees share the global Gradle build cache (`~/.gradle`, enabled via `org.gradle.caching=true`), so a warm cache turns their otherwise-cold KMP compiles into cache hits. Skip only if you know the cache is already warm from this session.

**4.3 — Run the AFK workflow.** Call the `Workflow` tool with the inline script below, passing `args` = `{ prd, context, claudeMdHint, issues, branch, baseSha }`:

- `prd` — the PRD's `Implementation Decisions` + `Testing Decisions` + relevant user stories (verbatim text).
- `context` — the contents of `.workflow/context.md` (the shared codebase map from grill-me-v2 recon). Passing it inline means subagents reuse it instead of each re-exploring the repo.
- `claudeMdHint` — one line pointing at `CLAUDE.md` + any ADRs in the touched area.
- `issues` — array of the workflow-ready AFK issues, each `{ id, title, blockedBy: [ids], acceptance, body }` where `body` is the full issue file content.
- `branch` — the session working branch from the 4.0 guard (used in worktree messages / assertions).
- `baseSha` — the session HEAD commit from the 4.0 guard. **Every worktree realigns to this commit before implementing**, so no agent ever silently works against `main`.

The workflow schedules the AFK issues as a **per-issue DAG**, NOT as coarse connected-component chains (a single feature is usually one connected component — grouping by component would collapse the whole graph into one sequential worktree and kill all parallelism). Instead **every issue runs in its own git worktree and starts the moment ALL its blockers have finished** — no level barrier, maximum concurrency. A dependent issue's worktree is a clean copy of the branch, so the workflow hands it the **own-diffs of its transitive blockers** (each once, in topo order) to apply as a baseline before it implements, so the upstream code is present. Each issue is then **adversarially verified** against its own acceptance criteria. The workflow `log()`s a per-issue start / build / verify line so **live progress shows in `/workflows`** (tell the user to open it — this run is otherwise opaque until it returns). It returns one entry per issue (own-diff + buildOk + verdict). It does NOT commit.

```javascript
export const meta = {
  name: 'implement-feature-parallel',
  description: 'Implement AFK issues: per-issue DAG in stacked parallel worktrees, adversarial verify per issue',
  phases: [
    { title: 'Implement', detail: 'one worktree agent per issue, started the moment its blockers finish' },
    { title: 'Verify', detail: 'adversarial review of each issue diff' },
  ],
}

const issues = args.issues || []
if (!issues.length) { log('No AFK issues passed'); return { results: [] } }

const byId = {}
for (const it of issues) byId[it.id] = it
const blockersOf = it => (it.blockedBy || []).filter(b => byId[b])

// transitive blockers of an issue, topo-ordered (blockers before dependents), each once.
// Their own-diffs are applied as the baseline before this issue is implemented.
function ancestorsTopo(id) {
  const out = []; const seen = new Set()
  const visit = x => {
    for (const b of blockersOf(byId[x])) if (!seen.has(b)) { seen.add(b); visit(b); out.push(b) }
  }
  visit(id)
  return out
}
log(`${issues.length} AFK issues scheduled as a per-issue DAG (each starts when its blockers finish)`)

const IMPL_SCHEMA = {
  type: 'object',
  properties: {
    summary: { type: 'string' },
    filesTouched: { type: 'array', items: { type: 'string' } },
    ownDiff: { type: 'string', description: "git diff of THIS issue's own changes only (after the upstream baseline commit)" },
    buildOk: { type: 'boolean' },
    notes: { type: 'string' },
  },
  required: ['summary', 'filesTouched', 'ownDiff', 'buildOk'],
}
const VERDICT_SCHEMA = {
  type: 'object',
  properties: {
    real: { type: 'boolean', description: 'true only if implementation satisfies every acceptance criterion' },
    problems: { type: 'array', items: { type: 'string' } },
    bottomLine: { type: 'string' },
  },
  required: ['real', 'problems', 'bottomLine'],
}

const implPrompt = (it, upstream) => `Implement ONE issue end-to-end in the current worktree (an isolated copy of the repo).

BRANCH GUARD — do this FIRST, before reading or writing anything else:
- Run \`git rev-parse HEAD\`.${args.baseSha ? ` It MUST be ${args.baseSha}.` : ''} If it differs from the intended base${args.baseSha ? ` (${args.baseSha}${args.branch ? `, session branch ${args.branch}` : ''})` : ''}, the worktree was based on the wrong ref (the worktree tool can fall back to main when the session branch is already checked out). Run \`git reset --hard ${args.baseSha || '<the intended base commit>'}\` to realign — safe, this is a throwaway isolated worktree. NEVER implement against main.

PRD context:
${args.prd}

${args.context ? `Shared codebase context (reuse this — do NOT re-explore what it already covers):\n${args.context}\n` : ''}
${args.claudeMdHint || 'Respect CLAUDE.md and any ADRs in the touched area.'}

${upstream.length ? `FIRST apply these upstream blocker diffs IN ORDER so their code is present, then make a THROWAWAY local baseline commit so your own work stays isolated:\n\n${upstream.map(u => `--- upstream ${u.id} (own diff) ---\n${u.ownDiff}`).join('\n\n')}\n\nApply each (e.g. \`git apply\`), then \`git add -A && git commit -m baseline\` (local throwaway, never pushed).` : 'This issue has no blockers — start from the clean worktree.'}

### Issue ${it.id} — ${it.title}
${it.body}

Rules:
- Lean on the shared context above first; only explore the codebase for what it does not cover.
- Implement code AND tests where the PRD testing decision covers these modules.
- Build with the project's task per CLAUDE.md (the Gradle build cache is shared + warm, so this is fast). Report buildOk honestly.
- Do NOT commit to the real branch, push, or open a PR — the baseline commit above is a local throwaway ONLY.
- Return ONLY YOUR OWN diff (\`git diff HEAD\` after the baseline), the files YOU touched, buildOk, and a short summary.`

const verifyPrompt = (it, impl) => `You are an adversarial reviewer. Decide if this implementation truly satisfies the acceptance criteria. Default to real=false if unsure.

Issue + acceptance:
[${it.id}] ${it.title}
${it.acceptance || it.body}

Reported summary: ${impl.summary}
Build passed (self-reported): ${impl.buildOk}
Files touched: ${(impl.filesTouched || []).join(', ')}

Diff under review (this issue's own changes):
${impl.ownDiff}

Hunt for acceptance gaps, missing tests, broken layering, stubbed logic, raw strings, manual DI bypassing Koin. Be blunt.`

// memoized per-issue scheduler: an issue starts as soon as ALL its blockers resolve.
// Ready issues run concurrently (the workflow runtime caps live agents); a dependent
// issue gets its transitive blockers' own-diffs as the worktree baseline.
const implOf = {}
const running = {}
function run(id) {
  if (running[id]) return running[id]
  running[id] = (async () => {
    const it = byId[id]
    const deps = blockersOf(it)
    await Promise.all(deps.map(run))                 // gate on blockers (resolves all transitive ancestors too)
    const upstream = ancestorsTopo(id)               // each ancestor once, topo order
      .map(aid => ({ id: aid, ownDiff: (implOf[aid] || {}).ownDiff || '' }))
      .filter(u => u.ownDiff)
    log(`▶ ${id} ${it.title} — start${deps.length ? ` (after ${deps.join(', ')})` : ''}`)
    const impl = await agent(implPrompt(it, upstream), { isolation: 'worktree', schema: IMPL_SCHEMA, phase: 'Implement', label: `impl:${id}` })
    implOf[id] = impl
    log(`${impl.buildOk ? '✔' : '✖'} ${id} impl — build ${impl.buildOk ? 'ok' : 'FAIL'}`)
    const verdict = await agent(verifyPrompt(it, impl), { schema: VERDICT_SCHEMA, phase: 'Verify', label: `verify:${id}` })
    log(`${verdict.real ? '✔' : '✖'} ${id} verify — ${verdict.real ? 'pass' : 'FAIL'}`)
    return { id, impl, verdict }
  })()
  return running[id]
}

const results = await Promise.all(issues.map(it => run(it.id)))
log(`Done — ${results.filter(r => r.verdict.real).length}/${results.length} issues verified real`)
return { results }
```

**4.4 — Integrate + commit (main thread).** When the workflow returns, **first re-assert the branch**: `git rev-parse --abbrev-ref HEAD` must still equal `BRANCH` from the 4.0 guard. If it does not, STOP — never apply diffs or commit the feature onto the wrong branch. Then walk `results` in **topological order** (blockers before dependents). For each issue:

- If `verdict.real === false`, surface `verdict.problems` to the user and stop — do NOT commit a failed issue.
- Otherwise apply that issue's `impl.ownDiff` to the working tree (the worktrees are isolated copies; the own-diff is the integration channel — applying in topo order reconstructs the feature, since each own-diff was authored against its blockers' baseline). Run the project's verification (build/typecheck/test per `CLAUDE.md`), then commit just that issue's files:

   ```
   <type>(<scope>): <short title> (#<NNNN>)

   Implements issues/<NNNN>-<slug>.md.
   ```

   `<type>` follows Conventional Commits; `<scope>` matches the touched module. Do NOT push. Do NOT open a PR. If verification is red, stop the loop and report — never continue on a red build.
- **If an own-diff fails to apply** (a hunk conflicts with a sibling issue that touched the same file), that is a missed dependency the graph treated as independent. Stop, report which issues overlap on which file, and ask the user to add a `Blocked by` to serialize them — then re-run; `resumeFromRunId` reuses the issues already done.

**4.5 — HITL issues.** When a HITL issue is reached in dependency order, never send it to the workflow. Surface it to the user, gather their decision/input, implement directly in the main thread (or via one focused `Agent` call), verify, and commit. Then **re-invoke the workflow** for the AFK issues that HITL just unblocked — pass `resumeFromRunId` from the prior run so already-implemented chains return from cache instead of re-running.

### 5. Progress display

**During the workflow run, live progress lives in `/workflows`** — the workflow `log()`s a `▶ start / ✔|✖ build / ✔|✖ verify` line per issue, so the user watches the per-issue tree there. This is the visibility channel for the otherwise-opaque background run: when you launch step 4.3, explicitly tell the user to open `/workflows`.

**After the workflow returns**, render the progress block inline as Markdown (no `TodoWrite`, no harness widget — works on Claude Desktop) from `results`, and re-post it as each commit lands during step 4.4 so the latest chat message always reflects committed state.

**Status legend:**

```
☐ pending   ▶ in-progress   ✔ done   ✖ failed   ⏸ HITL pause
```

**Progress block** — group by level, indent issues, show id + title + inline annotation:

```
Niveau 0
  ✔ 0012 Schéma users + migration Flyway          [12s, commit a3f1c2]
  ✔ 0013 DTO ApiTenant + mapping                  [9s,  commit b7e2d4]

Niveau 1
  ▶ 0014 Endpoint POST /auth/tenant-login         [subagent, 32s écoulées]
  ▶ 0015 Use case AuthenticateTenantUseCase       [subagent, 32s écoulées]

Niveau 2
  ⏸ 0016 Décision archi: stockage refresh         [HITL — input requis]

Niveau 3
  ☐ 0017 Écran SwiftUI LoginView
  ☐ 0018 Écran Compose LoginScreen
```

Re-post on these events:

- Start of a level (issues flip ☐ → ▶)
- Subagent return (▶ → ✔ or ✖, with elapsed time)
- HITL pause (☐ → ⏸)
- Commit landed (append `[commit <sha7>]` to the line)
- Verification result (red build → ✖ on the affected issue + halt notice)

**Per-level recap table** — print once a level finishes green, before moving to the next:

```
Niveau 1 terminé en 2m14s.
ID    | Statut | Build | Tests        | Commit
------|--------|-------|--------------|------------------
0014  | ✔ OK   | green | 3/3 passing  | feat(auth): tenant login endpoint (#0014)
0015  | ✔ OK   | green | 4/4 passing  | feat(auth): authenticate tenant use case (#0015)
```

### 6. Update CLAUDE.md + feature doc

Once all levels are green and committed, document what was just shipped so future sessions pick up the context without re-reading every commit.

**Keep `CLAUDE.md` short.** Do NOT dump the feature description into it. Instead:

1. **Write/update a sub-doc** under `docs/features/<NNNN>-<slug>.md` (create the folder if missing). Use the PRD `<NNNN>-<slug>` as filename. Content:

   ```markdown
   # <PRD title>

   Shipped on <YYYY-MM-DD> from PRD `prd/<NNNN>-<slug>.md`.

   ## What it does
   <2–4 sentences — user-visible behavior, not implementation>

   ## Technical surface
   - Endpoints / screens added
   - Modules / folders touched (high level)
   - Flyway migrations / tables added
   - Application events published/consumed

   ## Notable decisions
   <bullet list — non-obvious choices, gotchas, invariants to respect when evolving the feature>

   ## Issues
   - `issues/<NNNN>-<slug>.md` — <title> — commit <sha7>
   - ...
   ```

   All feature docs written in English (per repo convention).

   If the file already exists (re-run on same PRD), update in place — don't append duplicates.

2. **Add a one-line reference in `CLAUDE.md`** under a `## Shipped features` section (create the section if missing, near the bottom, before any final notes). Format:

   ```markdown
   ## Shipped features

   - [<NNNN> <PRD title>](docs/features/<NNNN>-<slug>.md) — <one-line hook, < 80 chars>
   ```

   One line per feature. Sorted by `<NNNN>`. If the line for this PRD already exists, update it in place; never duplicate.

3. **Do NOT** add architecture/style/convention rules to `CLAUDE.md` here — those belong in the relevant existing section and only if the user explicitly validated the rule. The goal of this step is a discoverable index of shipped features, not a changelog.

4. Stage and commit the doc changes as a final commit on top of the issue commits:

   ```
   docs(<scope>): index <NNNN> <slug> in CLAUDE.md
   ```

### 7. Final report

Once all levels are done (or the loop stopped on failure / HITL), print the final summary:

```
Implémentation <NNNN> <PRD title>

Implémentées : N / Total
Échouées     : N
Build final  : ✔ green | ✖ red
Tests        : <pass>/<total> passing
Commits      : <count> (sur <branche actuelle>)

Prochaines étapes suggérées :
- git log --oneline pour relire
- git push -u origin <branche>
- Ouvrir la PR
```

Include also:

- Files touched (high level, by feature folder, not exhaustive list)
- Verification status (build / tests)
- Do NOT push, do NOT open a PR automatically — these are the user's call.

## Rules

- **One commit per issue, on the current branch.** Commits happen automatically after each level passes verification. Never push, never open a PR — the user pushes and opens the PR manually after reviewing the commit log.
- **Never modify** the source PRD or issue files. They are the spec, not a checklist to mutate.
- **Respect domain conventions** declared in `CLAUDE.md` (naming, layering, typed IDs, value classes, use cases, etc.).
- **Subagent isolation**: each parallel subagent only touches the files relevant to its issue. If two subagents in the same level would clearly overlap on the same files, treat that as a missed dependency — stop and ask the user to refine `Blocked by`.
- **Tests follow the PRD's testing decision.** If the PRD opted out of tests, do not write tests. If the PRD scoped tests to specific modules, only write tests for those.
- **`CLAUDE.md` stays short.** Feature documentation lives in `docs/features/<NNNN>-<slug>.md` and is written in English (per repo convention). `CLAUDE.md` only carries a one-line link per feature under `## Shipped features`. Never inline feature descriptions into `CLAUDE.md`.
