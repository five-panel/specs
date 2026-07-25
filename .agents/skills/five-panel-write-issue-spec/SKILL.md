---
name: five-panel-write-issue-spec
description: Use when drafting, discussing, or publishing an implementation-ready product or engineering spec for a selected Five Panel GitHub issue in the five-panel/specs repository.
---

# Five Panel Issue Spec

## Scope

Use only for the selected issue passed by the launcher. Do not select a different issue.

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

Read the issue body and comments. Inspect `/root/git/five-panel` only enough to understand current behavior, relevant files, constraints, and test surface. Do not implement code.

If the issue is complex, unfamiliar, architectural, or likely to affect future product direction, research how the problem is generally solved in industry before proposing a solution. Use primary/authoritative sources where possible, summarize the relevant patterns, and explain how they do or do not fit Five Panel.

## Discussion Gate

Do not write the spec file yet.

After the required checks and research, start the user conversation in this order:

1. Explain the issue in simple words.
2. Explain the best solution you currently recommend in simple words.
3. If there are meaningful alternatives, present 2-3 options with trade-offs and your recommendation.
4. Ask clarifying questions one at a time when requirements, scope, or trade-offs need user input.

Keep the discussion focused on decisions that change the spec. Avoid posting to GitHub, committing, pushing, opening a PR, or changing labels during this stage.

When the direction is clear, present the full plan for the spec:

- proposed data/API/schema shape, if applicable
- desired behavior
- scope
- out-of-scope items
- acceptance criteria
- implementation notes
- test expectations
- risks
- open questions

Ask for explicit approval before writing the spec file. Accept clear approvals such as "yes", "approved", "update the spec", or "write it".

If requirements remain ambiguous or contradictory after discussion, report the blocking questions in the conversation and stop. Keep `spec WIP`; do not open a spec PR or change labels.

## Spec File

Only after the user approves the plan, create exactly one spec file in this repo:

`issues/<issue-number>-<slug>.md`

Use this structure:

```md
# Issue #<number>: <title>

Source issue: https://github.com/benitogonzalezh/five-panel/issues/<number>

## Problem

## Current Behavior

## Desired Behavior

## Scope

## Out Of Scope

## Acceptance Criteria

## Implementation Notes

## Test Expectations

## Risks

## Open Questions
```

Write concrete, testable acceptance criteria. Put `None` under Open Questions only when there are no blockers.

## Review Gate

Run a self-review after writing and before publishing:
- No placeholders such as TBD or TODO
- Every acceptance criterion is observable
- Scope and out-of-scope do not contradict each other
- Implementation notes are guidance, not a mandated hidden implementation unless required by the issue

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
- remove `ready for spec`
- remove `spec WIP`
- add `spec review`

Do not add `ready to dev`; that label is the human review gate after the spec PR is accepted.
