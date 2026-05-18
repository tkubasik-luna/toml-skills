---
name: implement-feature
description: Implement a feature end-to-end from a local PRD and its issues under `prd/` and `issues/`. Builds a dependency graph from each issue's `Blocked by` field, then parallelizes independent issues via subagents and chains dependent ones sequentially. Use when the user wants to implement a PRD, ship a feature, or work through a set of issues.
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

### 4. Execute level by level

For each level, in topological order:

1. **AFK issues in the level** — dispatch in parallel via the `Agent` tool, one subagent per issue.

   **CRITICAL — true parallelism requires a single message.** When a level has ≥ 2 AFK issues, you MUST emit all their `Agent` tool calls in **one single assistant message**, as multiple tool-use blocks in the same response. Do NOT send one `Agent` call, wait for the result, then send the next — that is sequential and defeats the whole point of this skill. Only one assistant turn dispatches the entire level's fan-out; subagents then run concurrently and their results stream back together.

   If the level has only 1 AFK issue, a single `Agent` call is fine.

   Each subagent gets a self-contained prompt containing:
   - The full issue file content
   - The PRD's `Implementation Decisions`, `Testing Decisions`, and relevant user stories
   - A pointer to `CLAUDE.md` and any ADRs in the touched area
   - The instruction to implement code AND tests (if the PRD's testing decision covers this issue's modules), respecting domain conventions
   - The instruction to leave the working tree modified and report a concise summary back. Subagents must NOT commit themselves, NOT push, NOT open a PR — the main thread handles commits sequentially after each level finishes.
   - Use `isolation: "worktree"` when the level fans out ≥ 2 subagents so they don't stomp each other's working tree. The main thread integrates their diffs back after the level finishes.

   Wait for all subagents in the level to finish before moving on.

   **Commit per issue.** Once all subagents in the level report success and the verification (step 4.3) is green, create one commit per issue in dependency order. Stage only the files that subagent touched, then commit with a message of the form:

   ```
   <type>(<scope>): <short title> (#<NNNN>)

   Implements issues/<NNNN>-<slug>.md.
   ```

   Where `<type>` follows Conventional Commits (`feat`, `fix`, `refactor`, `test`, `chore`, ...) and `<scope>` matches the feature folder / module touched. Do NOT push. Do NOT open a PR.

2. **HITL issues in the level** — never parallelize. Surface the issue to the user, gather their decision/input, then either implement directly in the main thread (if small) or hand off to one focused subagent.

3. **After each level**: run the project's quick verification (build/typecheck/test as relevant — see `CLAUDE.md` for commands). If it fails, stop the loop and report. Do NOT continue to the next level on a red build.

### 5. Progress display

Render progress inline as Markdown (no `TodoWrite`, no harness widget — works on Claude Desktop). Re-post the full progress block at every state transition so the latest message in the chat always reflects current state.

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
