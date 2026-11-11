---
name: five-panel-write-issue-spec
description: Use when drafting, discussing, or publishing an implementation-ready product or engineering spec for a selected Five Panel GitHub issue in the five-panel/specs repository.
---

# Five Panel Issue Spec

## Scope

Use only for the selected issue passed by the launcher. Do not select a different issue.

The launcher supplies the issue number and title, source and specs repositories, branch/worktree, and spec file path. It selects an issue labeled `ready for spec` and claims it with `spec WIP` before starting the agent. Verify that claim through the checks below.

This document owns the research, discussion, format, approvals, review, and publishing instructions. The launcher prompt supplies task details and points here. Produce three planning documents in English: the spec, a technical implementation plan, and a short implementation guide. Do not implement application code.

Source repo: `benitogonzalezh/five-panel`
Specs repo: `five-panel/specs`

## Required Checks

Before research, discussion, or writing, verify the source issue:
- is open
- has `ready for spec`
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

When the direction is clear, present the proposed spec using the same short five-section format defined below. Explain that it will be accompanied by a technical implementation plan and a short implementation guide. Follow the user's preferred review pace. Put essential decisions and constraints into the spec and keep detailed research in working notes.

Ask for explicit approval before writing the documents. Accept clear approvals such as "yes", "approved", "update the spec", or "write it". Apply that approval to the three-document set; do not request a separate approval for each file.

If requirements remain ambiguous or contradictory after discussion, report the blocking questions in the conversation and stop. Resolve them before asking for writing approval; do not add an Open Questions section to the spec. Keep `spec WIP`; do not open a spec PR or change labels.

## Required Documents

After writing approval, create these three Markdown files in this repository, using the issue number and slug from the launcher-provided spec path:

| Document | Default path | Purpose |
| --- | --- | --- |
| Spec | `issues/<issue-number>-<slug>.md` | The agreed behavior, scope, impact, and completion checks. |
| Technical implementation plan | `issues/<issue-number>-<slug>-implementation-plan.md` | Concrete instructions for implementing and verifying the change. |
| Short implementation guide | `issues/<issue-number>-<slug>-implementation.md` | A simple explanation of how the pieces fit, with a complete example flow and concrete data. |

For an existing issue, update its established document paths rather than creating duplicate plans or renaming files. Write all three documents in English. Link them to each other with relative Markdown links; put links to both implementation documents near the top of the spec inside its existing sections.

## Spec File

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

## Technical Implementation Plan

Write a separate plan for the person or agent implementing the approved spec. Scale its detail to the change. Include:

- Existing files and services to reuse, focused modifications, and new files, with their responsibilities and implementation order.
- Concrete interfaces and data contracts: actual routes, operation names, inputs, outputs, saved values, and required storage changes.
- Tasks with meaningful verification steps and expected outcomes, including relevant access checks, compatibility, failures, and recovery.
- A clear distinction between existing behavior and proposed code. Label example data and planned operations accurately; examples do not prove that a feature is already implemented.

Keep the technical plan consistent with the short guide. Detailed signatures, task checklists, and test commands belong here.

## Short Implementation Guide

Write a separate guide for the user in simple English, comparable in length and level of detail to the spec. Explain what existing code changes, what new code is added, what is reused, where data is stored, and how those pieces work together. Use a small, complete worked example to make the main flow concrete.

The guide must include a **Concrete flow example** section. Show the start, every meaningful handoff, and the final observable result using consistent sample values:

- **API or backend work:** show the trigger or method/route, full request or event data, the relevant service/processing path, transformations and written record values, and the response/result. Follow the same example data from input to stored values and output.
- **Data-only work:** show actual before/after records, the job or operation input, how values are transformed and saved, and the final result. Include only the interfaces that this flow uses.
- **Frontend work:** start at the actual page, button, or form. Show concrete entered values or local state, the action taken, requests and backend/data changes when present, the returned data, and how the UI reaches its visible final state. For a frontend-only change, show the local state and UI transitions without inventing API or database work.
- **Automations, queues, or multi-step processing:** include the complete relevant automation/job definition, the initial input, each step's input/output handoff, continuation parameters, and how processing finishes. Show the planned JSON shape, not only operation names or a diagram.

Use complete, valid JSON for applicable payloads, definitions, and record values. Choose small examples so no payload needs ellipses or unnamed placeholders. Keep IDs, field names, types, counts, cursors, and values consistent across all examples and both implementation documents. Use safe sample data, never credentials. Explain which values are examples and which structures are the contract to implement.

Keep the narration brief and focused on understanding the implementation. The technical plan carries the full file list, signatures, task breakdown, and verification commands; the short guide carries the complete example flow. Resolve contract gaps before requesting writing approval rather than presenting incompatible examples as implementation-ready.

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
- All three English documents exist, have distinct purposes, and link to the correct companion files
- The technical plan specifies the implementation tasks, concrete contracts, and relevant checks; it agrees with the spec and guide
- The short guide explains the implementation simply and includes a complete concrete flow for its API/data/frontend work, including applicable automation or queue handoffs
- Example JSON parses and matches existing contracts or the new contracts specified in the plan; example inputs, written values, outputs, and continuation parameters agree. Use available parsers/validators where relevant. Report only checks actually run

Then summarize the three local drafts with clearly labeled links and ask for explicit approval to publish. Do not commit, push, open a PR, comment on GitHub, or change labels until the user approves publishing.

If the user requests changes, revise the spec, rerun the self-review, and ask again.

## Publishing

Only after explicit publishing approval:

- commit all three documents on the provided branch
- push it
- open one PR in `five-panel/specs` containing all three documents, with clearly labeled links to each
- comment on the source issue
- update source issue labels

The source issue comment should include:
- the spec PR link
- the spec file path and links to the technical implementation plan and short implementation guide
- a short summary
- any non-blocking notes

The source issue label update should:
- remove `ready for spec`
- remove `spec WIP`
- add `spec review`

Do not add `ready to dev`; that label is the human review gate after the spec PR is accepted.

The final response must link all three documents separately as **Spec**, **Technical implementation plan**, and **Short implementation guide**. Use working repository links or absolute local file links so the user can open each file directly.
