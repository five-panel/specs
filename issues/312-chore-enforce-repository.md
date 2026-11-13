# Issue #312: chore: enforce repository formatting before commits

Source issue: https://github.com/benitogonzalezh/five-panel/issues/312

## TL;DR

Bring every supported repository file into line with the current Prettier rules, including existing tenant scripts and published helpers. Then keep that standard with Git's own commit hooks and CI. Formatting may change layout and quote style; application behavior must stay the same.

Read the [technical implementation plan](312-chore-enforce-repository-implementation-plan.md) and [short implementation guide](312-chore-enforce-repository-implementation.md).

## What will change

**Medium code impact:** add development tools, a native Git hook, a CI check, and setup instructions, plus an initial formatting pass across existing files. Many files may change, but business logic and stored data must stay the same.

Reuse the root Prettier dependency and configuration. Add lint-staged to check the files selected for a commit while preserving partly staged edits. Add a small installer to enable the checked-in hook during root dependency installation; Husky is not needed.

Use the same policy across source, tests, configuration, and docs. No file is exempt because it is old, tenant-specific, or already published. Keep the existing dependency and generated-file exclusions, and make package commands read the root ignore files. Adjust tests that depend on exact source formatting without weakening what they test.

## How it would work

1. Establish a clean baseline by formatting all supported repository files with the existing configuration. Keep that pass in a separate commit for review and run the existing checks to catch behavior changes.
2. After a fresh checkout, the developer installs root dependencies on the host using Node 24. The installer points Git to `.githooks`. Document manual setup when install scripts are disabled and how editors find Node and npm.
3. Before a normal CLI or editor commit, Git calls the shared hook. It runs lint-staged and Prettier on staged supported files. Existing Vue and JSON overrides still apply.
4. Bad formatting, invalid syntax, or missing tools stop the commit with useful guidance. The author fixes the reported files, reviews the changes, stages only the intended lines, and retries. The hook checks without rewriting file contents.
5. Preserve the Git index—the selected commit content—and all unrelated edits on success and failure. File names with spaces must work. The full formatting pass is an explicit command, never part of setup or an ordinary commit check.
6. CI checks the whole supported repository against the same rules. Once the initial formatting pass is complete, existing and new files must both pass.

## Done when

- A fresh root install enables the native hook; CLI and editor commits run it, including in a linked Git worktree.
- Compliant staged files pass; bad formatting, parser errors, and missing tools stop commits with recovery instructions.
- Partly staged files, unrelated unstaged edits, and file names with spaces retain their selected and unselected contents on success and failure.
- All supported source, tests, configuration, and docs pass the repository check, including existing and new tenant scripts and published helpers.
- CI reports formatting failures anywhere in that set, including bypassed commits, while existing tests and builds pass with the same behavior checks.
- The baseline diff contains only formatting and necessary source-text test adjustments; setup and checks do not start application or database work or rewrite unrelated edits.

## Out of scope

- Changing formatting rules or doing the separate cleanup of PR #305.
- Adding formatters for unsupported files such as shell scripts and SQL.
- Changing application behavior, data, permissions, or GitHub branch-protection settings.
