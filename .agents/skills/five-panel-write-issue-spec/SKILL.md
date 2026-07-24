---
name: five-panel-write-issue-spec
description: Use when writing an implementation-ready product or engineering spec for a selected Five Panel GitHub issue in the five-panel/specs repository.
---

# Five Panel Issue Spec

## Scope

Use only for the selected issue passed by the launcher. Do not select a different issue.

Source repo: `benitogonzalezh/five-panel`
Specs repo: `five-panel/specs`

## Required Checks

Before writing, verify the source issue:
- is open
- has `ready for spec`
- has `spec WIP`
- does not have `spec review`, `ready to dev`, or `WIP`
- is not assigned to someone else
- is not already handled by an open PR

If any check fails, comment on the source issue with the reason and stop.

## Research

Read the issue body and comments. Inspect `/root/git/five-panel` only enough to understand current behavior, relevant files, constraints, and test surface. Do not implement code.

If requirements are ambiguous or contradictory, comment on the issue with the blocking questions and stop. Keep `spec WIP`; do not open a spec PR.

## Spec File

Create exactly one spec file in this repo:

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

## Completion

Run a self-review before publishing:
- No placeholders such as TBD or TODO
- Every acceptance criterion is observable
- Scope and out-of-scope do not contradict each other
- Implementation notes are guidance, not a mandated hidden implementation unless required by the issue

Commit the spec on the provided branch, push it, and open a PR in `five-panel/specs`.

Comment on the source issue with:
- the spec PR link
- the spec file path
- a short summary
- any non-blocking notes

Update source issue labels:
- remove `ready for spec`
- remove `spec WIP`
- add `spec review`

Do not add `ready to dev`; that label is the human review gate after the spec PR is accepted.
