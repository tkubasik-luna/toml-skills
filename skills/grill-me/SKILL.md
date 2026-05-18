---
name: grill-me
description: Interview the user relentlessly about a plan or design until reaching shared understanding, resolving each branch of the decision tree. Use when user wants to stress-test a plan, get grilled on their design, or mentions "grill me".
---

Interview me relentlessly about every aspect of this plan until we reach a shared understanding. Walk down each branch of the design tree, resolving dependencies between decisions one-by-one. For each question, provide your recommended answer.

**FIRST STEP — UPFRONT INPUTS (QCM).** Before any design question, ask via `AskUserQuestion` whether the user has data / files / text to provide upfront (specs, mockups, logs, schemas, PRD, transcripts, screenshots, dumps). Options: "Yes, files" / "Yes, pasted text" / "Yes, both" / "No, nothing". If ≠ "No": wait for the inputs, Read each file fully, parse pasted text, extract key facts (constraints, locked decisions, actors, deadlines, exact names), and KEEP all info from those inputs in context for later questions. Short recap (3–6 bullets) of what was retained, then proceed to the first design question. If "No": proceed directly.

**SECOND STEP — CLAUDE DESIGN / FIGMA (QCM).** Right after upfront inputs, ask via `AskUserQuestion` if user has a Claude Design / Figma link for the feature. Options: "Yes, Figma selection link" / "Yes, full Figma file" / "Yes, multiple nodes" / "No design". If ≠ "No": wait for link(s)/node-id(s), then use Figma MCP tools (`get_design_context`, `get_screenshot`, `get_variable_defs`, `get_code_connect_map`, `get_metadata`) to extract everything needed for implementation: screen structure, components, variants, design tokens (colors, typo, spacing, radii), variables, assets, interactive states, layout constraints, existing Code Connect mapping, screenshots. Store in MD files under `grill-me/design/` at repo root:
- `grill-me/design/overview.md` — screen/node list, summary capture, source link
- `grill-me/design/tokens.md` — colors, typo, spacing, radii, elevations (exact values)
- `grill-me/design/components.md` — components used, variants, props, mapping to repo design system
- `grill-me/design/screens/{screen-name}.md` — one file per screen: hierarchy, states, interactions, required assets, referenced screenshot
- `grill-me/design/assets.md` — icons/images to export (node-id + target format)

Short recap (5–10 bullets) of stored design. Keep MDs referenced in context for later design questions. If "No": proceed directly.

Ask the questions one at a time.

If a question can be answered by exploring the codebase, explore the codebase instead.
