---
name: to-prd
description: Turn the current conversation context into a PRD and write it as a local Markdown file under `prd/` at the repo root. Use when user wants to create a PRD from the current context.
---

This skill takes the current conversation context and codebase understanding and produces a PRD. Do NOT interview the user — just synthesize what you already know.

## Process

1. Before drafting the PRD, ask the user via a multiple-choice question whether they want tests implemented for this feature. Present the options:

   - A) Yes, write tests for all modules
   - B) Yes, but only for selected modules (ask which ones)
   - C) No tests for now

   Wait for the user's answer before proceeding. The answer determines whether the `Testing Decisions` section is filled in, scoped to specific modules, or omitted from the PRD.

2. Explore the repo to understand the current state of the codebase, if you haven't already. Use the project's domain glossary vocabulary throughout the PRD, and respect any ADRs in the area you're touching.

3. Sketch out the major modules you will need to build or modify to complete the implementation. Actively look for opportunities to extract deep modules that can be tested in isolation.

A deep module (as opposed to a shallow module) is one which encapsulates a lot of functionality in a simple, testable interface which rarely changes.

Check with the user that these modules match their expectations.

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
