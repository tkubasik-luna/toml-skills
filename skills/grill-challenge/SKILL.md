---
name: grill-challenge
description: Grilling session like grill-me, but every recommendation gets independently challenged by an expert architect sub-agent specialised in the relevant tech. The architect reviews each proposed answer through a maintainability and long-term evolvability lens. Use when user wants their plan grilled AND each recommendation peer-reviewed.
---

Interview the user relentlessly about every aspect of the plan until shared understanding is reached. Walk each branch of the decision tree, resolving dependencies between decisions one by one. For each question, provide a recommended answer.

Ask questions one at a time. Wait for feedback before continuing.

If a question can be answered by exploring the codebase, explore instead of asking.

## The challenge loop

After formulating each question + recommended answer — BEFORE showing it to the user — spawn a sub-agent via the `Agent` tool (`subagent_type: general-purpose`, **`model: opus`** — mandatory, max reasoning quality) acting as an expert architect in the technology under discussion. Pick the specialty per question:

- DB schema / transactions → senior PostgreSQL / data architect
- Backend module boundaries → Spring Boot / DDD architect
- Frontend state / data fetching → React / TanStack architect
- Mobile / KMP → Kotlin Multiplatform architect
- Auth / security → application security architect
- Async / events / realtime → distributed-systems architect
- Etc. Match the tech precisely; do not default to "generic senior dev".

### Sub-agent prompt template

Self-contained brief (the agent has zero conversation context):

```
You are a senior <SPECIALTY> architect. Review this design recommendation through one lens only: long-term maintainability and evolvability of the codebase.

Context (project): <2–4 line summary of relevant project context>
Question being asked: <the question>
Recommended answer: <the recommendation, with the reasoning>

Produce:
1. Verdict — agree / agree-with-caveats / disagree
2. Maintainability risks — what hurts in 6–24 months (coupling, lock-in, migration cost, test friction, onboarding cost)
3. Evolvability risks — what blocks likely future changes (scale, new requirements, team growth, tech swap)
4. Counter-proposal if disagree, or refinement if caveats
5. One-line bottom line

Be blunt. No filler. Under 250 words.
```

Run sub-agents in foreground — the verdict shapes what is shown next. Always pass `model: "opus"` in the `Agent` call so the architect reasons with max capability.

### Presenting to the user

For each turn, output in this order:

1. **Question** — one sentence
2. **My recommendation** — the answer + brief reasoning
3. **Architect review** (`<SPECIALTY>`) — the sub-agent's verdict + key risks + counter-proposal if any
4. **Synthesis** — if architect agrees, restate confidence; if architect pushes back, present revised recommendation OR explicitly surface the trade-off and ask the user to arbitrate

Then wait for user feedback before moving to the next question.

## Rules

- One question per turn. Never batch.
- Always run the architect sub-agent — even on questions that feel obvious. Cheap insurance against blind spots.
- Maintainability and evolvability are the ONLY review axes. Don't let the architect drift into perf micro-optimisation, style nits, or feature scope.
- If two consecutive questions hit the same specialty, reuse the same architect framing but spawn a fresh agent each time (no shared memory between turns is fine — keeps reviews independent).
- If the architect's counter-proposal opens a new decision branch, queue it and address it as the next question.
