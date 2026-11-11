---
name: five-panel-write-issue-spec
description: Use when drafting, discussing, or publishing an implementation-ready product or engineering spec for a selected Five Panel GitHub issue in the five-panel/specs repository.
---

# Five Panel Issue Spec

## Scope

Use only for the selected issue passed by the launcher. Do not select a different issue.

The launcher supplies the issue number and title, source and specs repositories, branch/worktree, and spec file path. It selects an issue labeled `ready to spec` and claims it with `spec WIP` before starting the agent. Verify that claim through the checks below.

This document owns the research, discussion, format, approvals, review, and publishing instructions. The launcher prompt supplies task details and points here. This task writes the spec only; do not implement application code.

Source repo: `benitogonzalezh/five-panel`
Specs repo: `five-panel/specs`

## Required Checks

Before research, discussion, or writing, verify the source issue:
- is open
- has `ready to spec`
- has `spec WIP`
- does not have `spec review`, `ready to dev`, or `WIP`
- is not assigned to someone else
- is not already handled by an open PR

If any check fails, stop and report the reason in the conversation. Do not write a spec, open a PR, change labels, or comment on GitHub unless the user explicitly asks for that follow-up.

## Research

Read the issue body and comments. Before proposing or writing any spec, read the complete application source in `/root/git/five-panel`. This review is required for every issue, including small ones. Files named in the issue are a starting point, not the limit of the review. Do not implement code.

Complete the code review before presenting a proposed solution:
- Read `AGENTS.md`, `RULES.md`, and the applicable project docs. Review the frontend, backend, database, integrations, storage, ops, and tests to understand the whole application and how its parts connect.
- Read complete source files and trace the affected behavior from its entry points through services, repositories, stored data, external systems, and the return or recovery path. Search results, filenames, and selected snippets do not replace reading the code.
- Build an inventory of all data and behavior the issue must preserve or cover. Derive it from the current schema, migrations, services, repositories, and storage providers. Include configuration, permissions, history, background work, and files where they apply. Compare that inventory with what the current flow actually handles; an existing hardcoded list is not proof of completeness.
- Follow dependencies and callers beyond the files named in the issue. Check existing services we can reuse, normal behavior, failures, recovery, company isolation, older-data compatibility, and the relevant tests.
- Account for every item in the inventory: already handled, missing, deliberately excluded, or owned by a related issue. Work owned by another issue must still be considered when checking the complete user flow; link the dependency and explain what remains to be verified.
- Keep evidence in working notes: real source paths, current behavior, gaps, and checks actually run. Clearly separate findings from reading code, behavior reproduced in a test, and assumptions that remain unverified. Never describe a partial review as a complete one.

If code or context needed to complete the review is unavailable, explain the gap and resolve it before presenting the spec for writing approval. Keep the user-approved spec format short; thorough research does not require extra document sections.

If the issue is complex, unfamiliar, architectural, or likely to affect future product direction, research how the problem is generally solved in industry before proposing a solution. Use primary/authoritative sources where possible, summarize the relevant patterns, and explain how they do or do not fit Five Panel.

## Discussion Gate

Do not write the spec file yet.

After the required checks and complete code review, start the user conversation in this order:

1. Explain the issue in simple words.
2. Explain the best solution you currently recommend in simple words.
3. If there are meaningful alternatives, present 2-3 options with trade-offs and your recommendation.
4. Ask clarifying questions one at a time when requirements, scope, or trade-offs need user input.

Keep the discussion focused on decisions that change the spec. Avoid posting to GitHub, committing, pushing, opening a PR, or changing labels during this stage.

When the direction is clear, present the proposed spec using the same short five-section format defined below. Follow the user's preferred review pace. Put essential decisions and constraints into those sections and keep detailed research in working notes.

Ask for explicit approval before writing the spec file. Accept clear approvals such as "yes", "approved", "update the spec", or "write it".

If requirements remain ambiguous or contradictory after discussion, report the blocking questions in the conversation and stop. Resolve them before asking for writing approval; do not add an Open Questions section to the spec. Keep `spec WIP`; do not open a spec PR or change labels.

## Spec File

Only after the user approves the plan, create exactly one spec file in this repo:

`issues/<issue-number>-<slug>.md`

Keep the issue title and source issue link at the top. Use exactly these five sections, in this order. For an integration, replace the third heading with `How the integration would work`; otherwise use `How it would work`.

```md
# Issue #<number>: <title>

Source issue: https://github.com/benitogonzalezh/five-panel/issues/<number>

## TL;DR
Explain what we will do in simple words. Keep code impact in the next section.

## What will change
Explain what needs new code, what existing code needs focused changes, and which existing services we can reuse. Include a Small, Medium, or Large code-impact rating with a short reason.

## How it would work
Explain the flow in a short numbered list, referring to real files or services where useful. Separate existing code we reuse or extend from proposed new code. Keep shared capabilities generic and provider-specific rules in the connector where applicable. Add only the common capabilities this issue needs. Leave exact classes, routes, tables, and other detailed implementation choices to implementation planning unless they are necessary to explain the behavior.

## Done when
Use 3-6 short, observable checks that show the feature works, including important recovery or existing-feature checks where relevant.

## Out of scope
Use a short list of clear boundaries. Include important first-version limits and work deliberately left for later.
```

Use simple, common words and avoid repetition. Keep the whole spec under 1,000 words unless the user asks for more detail; small issues should be much shorter. Do the required research and self-review, but do not add separate long sections for research, risks, API/database designs, test plans, or repeated background. Put essential decisions and constraints into the five sections above.

## Review Gate

Run a self-review after writing and before publishing:

- The complete application code review was done before proposing the spec; the data and behavior inventory is supported by source evidence
- Every inventory item is covered by observable completion checks or a clear boundary or linked dependency
- The title and source issue link are at the top, followed by exactly the five required sections in order, with the correct third heading
- `What will change` includes the code-impact rating and a short reason
- `Done when` has 3-6 observable checks, including important recovery or existing-feature checks where relevant
- The spec stays under 1,000 words unless the user asked for more detail, and has no unresolved requirements
- No placeholders such as TBD or TODO
- Every acceptance criterion is observable
- Scope and out-of-scope do not contradict each other
- Technical guidance leaves detailed implementation choices to planning unless they are required to explain the behavior

Then summarize the local draft for the user and ask for explicit approval to publish. Do not commit, push, open a PR, comment on GitHub, or change labels until the user approves publishing.

If the user requests changes, revise the spec, rerun the self-review, and ask again.

## Publishing

Only after explicit publishing approval:

- commit the spec on the provided branch
- push it
- open a PR in `five-panel/specs`
- comment on the source issue
- update source issue labels

The source issue comment should include:
- the spec PR link
- the spec file path
- a short summary
- any non-blocking notes

The source issue label update should:
- remove `ready to spec`
- remove `spec WIP`
- add `spec review`

Do not add `ready to dev`; that label is the human review gate after the spec PR is accepted.
