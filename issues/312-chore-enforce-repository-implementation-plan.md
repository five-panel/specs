# Issue #312: repository formatting implementation plan

Source issue: https://github.com/benitogonzalezh/five-panel/issues/312

[Spec](312-chore-enforce-repository.md) · [Short implementation guide](312-chore-enforce-repository-implementation.md)

**Goal:** make all supported repository files follow the current rules, preserve application behavior, and enforce formatting through native Git hooks and CI.

**Approach:** first establish a repository-wide formatting baseline. A small installer enables `.githooks/pre-commit`; lint-staged handles staged-file selection and recovery, and existing Prettier rules decide formatting. CI checks the whole supported repository. This document describes proposed work in the application repository; these planning documents do not format the application or install a hook.

## Existing behavior and boundaries

The root has Prettier locked at 3.7.4 and no hook setup. Backend and frontend formatting scripts cover only their `src` files. Existing CI installs each package separately, runs tests and builds, and checks test-file placement. It does not install root tools or check formatting.

Keep `.prettierrc.json` unchanged: TypeScript uses single quotes; Vue and JSON retain their double-quote overrides. Add lint-staged 17.6.0 as a root development dependency and update only the required lock entries. Use Node 24 for host development tools and CI; this does not change the application's Node 22 containers.

The initial formatting pass includes historical tenant scripts and published helpers. Their logic, string values, stored-data contracts, and behavior must stay the same. Keep commit and CI checks read-only with respect to file contents; a failed commit must preserve the exact index and worktree contents. Setup and formatting tools must not import application modules or run migrations, tenant scripts, jobs, backups, or external services. Application tests remain separate validation.

## Files and order

| Order | File                                                                 | Responsibility                                                                                                        |
| ----- | -------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| 1     | `.prettierignore`                                                    | Reuse existing dependency and generated-output exclusions; add no tenant or published-file exceptions.                |
| 1     | `backend/package.json`, `frontend/package.json`                      | Make both existing formatting commands read `../.prettierignore` and `../.gitignore` explicitly.                      |
| 1     | `.lintstagedrc.json` (new)                                           | One shared staged-file policy.                                                                                        |
| 1     | Root `package.json`, `package-lock.json`                             | Add lint-staged and the commands below; preserve existing CLI commands and Prettier version.                          |
| 2     | `.githooks/install.mjs` (new)                                        | Enable the native hook through local Git configuration.                                                               |
| 2     | `.githooks/pre-commit` (new)                                         | Executable shell entry point that calls the root staged check and explains failures.                                  |
| 3     | All supported repository files                                       | Initial formatting pass, including old tenant scripts and helpers, kept in a separate commit.                         |
| 3     | Source-text assertions in existing tests, including `frontend/test/` | Adjust expected source spelling/spacing when formatting changes it; retain the same assertions about behavior.        |
| 4     | `.github/workflows/ci.yml`                                           | Add a whole-repository formatting job and install root dependencies in the ops job for the new regressions.           |
| 1–4   | `ops/test/formatting.test.ts` (new)                                  | Real disposable Git repositories testing setup, selected content, normal exclusions, and repository-wide enforcement. |
| 4     | `README.md`                                                          | Host setup, explicit file formatting, partial staging, editor use, and recovery.                                      |

Use the existing ops `tsx`/`node:test` runner. Keep fixture helpers as direct functions in the new test file unless their size warrants a focused helper under `ops/test/helpers/`. No new application services, routes, database records, or storage changes are needed.

## Commands and configuration

Add these root script entries alongside the existing ones:

```json
{
  "prepare": "node .githooks/install.mjs",
  "hooks:install": "node .githooks/install.mjs",
  "format": "prettier --write --ignore-unknown",
  "format:all": "prettier --write . --ignore-unknown",
  "format:check": "prettier --check . --ignore-unknown",
  "format:staged": "lint-staged --config .lintstagedrc.json"
}
```

The root `format` command takes explicit file arguments for normal fixes. `format:all` is the explicit repository-wide write command; `format:check` checks that entire set without writing. Neither setup nor the commit hook runs `format:all`. `.lintstagedrc.json` contains:

```json
{
  "*": "prettier --check --ignore-unknown"
}
```

Run from the repository root. Let lint-staged pass file arguments safely and retain its backup, partial-staging, and restoration defaults. Do not construct shell file lists, add `git add`, use write mode, or disable recovery. Supported source, tests, ops TypeScript, JSON, YAML, Markdown, CSS, and other built-in Prettier formats are eligible. Unknown formats are skipped. Existing root ignores still apply; package commands need explicit root ignore paths because their working directory differs.

Apply the same policy to all supported files. In particular, `backend/src/db/scripts/` is fully eligible: existing dated scripts, undated helpers such as `zipp/commerce/`, new files, and top-level maintained runners. Do not add directory, date, or exact-path exceptions for these files. Supported files in hidden repository folders remain covered too. Check package locks; do not add a blanket lock-file exclusion. Retain the existing dependency/generated-artifact ignores and skip unsupported formats rather than adding plugins.

## Task 1: shared policy and file coverage

- [ ] Add real-Git regressions that invoke the planned root command: TypeScript, Vue double quotes, JSON, ops/tests/docs/config, hidden-folder files, unknown formats, and ignored generated files. Bad formatting in an existing published script, an existing undated helper, a new tenant script, and a maintained runner must fail; correctly formatted versions must pass.
- [ ] Run `node ops/node_modules/tsx/dist/cli.mjs --test ops/test/formatting.test.ts` from the repository root; new command-dependent cases should fail before implementation.
- [ ] Add the dependency, scripts, static config, and focused ignore changes above. Preserve existing package source globs; append `--ignore-path ../.prettierignore --ignore-path ../.gitignore` to both package commands.
- [ ] Rerun that focused command. Tests may copy the real formatting files into fixtures but must not import or execute application modules. Prove package commands read shared generated-file ignores while the backend command checks both existing tenant scripts and maintained runners.

## Task 2: native setup and normal commits

The installer uses Node built-ins and Git subprocesses with argument arrays. Resolve the application root from the installer's location. Use `git rev-parse` so a linked worktree's `.git` file is supported, then set `git config --local core.hooksPath .githooks`. Store this relative path in the repository's local Git configuration; linked worktrees share that configuration and resolve the hook against their own working root.

Make setup repeatable. Skip with a clear message in `CI=true` or an archive without Git metadata. For a different configured hooks path, report the conflict and fail without replacing it; document manual resolution. Surface other Git failures. Commit the hook with LF endings and executable mode `100755`; installation must not rewrite source files.

The shell hook checks that npm and the root local lint-staged/Prettier tools are present, then calls `npm run format:staged`. Missing tools fail with instructions to run root `npm ci` using Node 24. On other failures, keep the original tool output and explain: fix the reported files, review changes, stage intended lines, retry. No automatic downloads or global-tool fallback.

- [ ] Add setup tests for fresh root `npm ci`, repeated setup, linked worktrees, `CI=true`, no Git metadata, a different hooks path, and missing tools. In disposable setup tests, explicitly set `CI=false`; the outer CI environment must not suppress the fixture hook.
- [ ] Test direct staged checks on good and bad partly staged files: compare file bytes, index blobs, and staged/unstaged diffs before and after each call. Then test normal commits for a compliant file, bad formatting, parser failure, and the spaced file name in the guide. Failed commits preserve those snapshots exactly. Successful commits save the original selected blobs, clear the staged diff, and retain the unselected edits on disk. Leave an untouched badly formatted file in the baseline to prove there is no sweep.
- [ ] Implement the installer and hook, then rerun the focused tests. Use a fresh disposable checkout with real root dependencies for the install test. Other fixtures may link installed tools to avoid repeated downloads.

The shared fixture values for the guide are `ops/test/format sample.ts`, committed `export const label = 'Draft';\n`, selected `export const label = "Ready";\n`, and an unselected trailing `// Finish the next change later.\n`. After an explicitly selected formatting fix, the commit contains only `export const label = 'Ready';\n`; the comment remains uncommitted.

## Task 3: repository-wide formatting baseline

- [ ] Work in a clean application checkout. Run the existing suites and builds to record the starting result, then run `npm run format:check` to record existing formatting failures. Do not format over another person's uncommitted edits.
- [ ] Run local Prettier with `--debug-check . --ignore-unknown` before the write pass, then run `npm run format:all`. The debug check is an additional check for unintended code changes, not proof of unchanged application behavior. Review the actual diff and run tests and builds too.
- [ ] Include every supported file reached by the standard root command, including historical tenant scripts and published helpers. Keep behavior, string values, database operations, access checks, and runtime interfaces the same. Do not execute historical scripts as part of formatting. No formatter-policy changes or file exemptions may be used to hide failures.
- [ ] Where a test reads source text and expects an old quote or line layout, update only that expected representation to match formatting. Preserve the behavior assertion; do not remove, skip, or weaken tests to make the pass succeed. Reformat those adjusted tests too.
- [ ] Verify `npm run format:check` passes. A second `npm run format:all` must make no further changes. Run backend/frontend/ops tests and backend/frontend builds. Keep this formatting pass and the necessary source-text assertion adjustments in a distinct formatting commit, separate from hook setup and CI changes, within the implementation PR.

## Task 4: whole-repository CI and instructions

Use the existing `pull_request`/`push` events and read-only permissions. Add a formatting job with the normal checkout, Node 24, root lock-file caching, root `npm ci`, and `npm run format:check`. The job checks the whole supported checkout, returns failure for formatting or parser errors, and never writes fixes. Keep current layout, backend, frontend, and ops jobs; add root installation to the ops job before its normal test command so these tests have their tools. No commit-range helper or history-fetch logic is needed.

- [ ] Add fixture tests proving the whole-repository command fails for a badly formatted file even if that file was unchanged in the latest commit, including an existing published script. A commit created with the hook bypassed must still fail the full check. A clean fixture must pass; ignored generated files and unsupported formats must not cause failure.
- [ ] Prove `format:all` fixes those fixtures and a second run leaves their bytes unchanged. Keep the separate staged-only tests to show that normal commit checks preserve unselected work.
- [ ] Implement the job, run focused tests, and inspect its command and lock-file cache path. The full formatting check must pass after the baseline pass and before the implementation PR is ready.

- [ ] Document root host `npm ci` alongside existing Docker/package setup. Explain that installing backend/frontend/ops dependencies alone does not enable host Git hooks. For `--ignore-scripts`, run `npm run hooks:install` afterward. Show `git config --local --get core.hooksPath` and the expected `.githooks` value.
- [ ] Document terminal/editor use and Node/npm availability in the editor's environment. Explain `npm run format -- "path with spaces.ts"`, review, and staging only intended lines with the editor or `git add -p`; never advise blindly restaging a partly staged file. Explain the explicit full formatting and full check commands and the lack of exceptions for old or published scripts.
- [ ] Run `npm ci`, `npm ci --prefix ops`, `npm --prefix ops test`, the focused formatting tests, `npm run format:check`, `sh -n .githooks/pre-commit`, `node --check .githooks/install.mjs`, and `git diff --check`. Existing backend/frontend suites, builds, and test-layout checks must pass in their normal CI jobs. Confirm no business-logic or runtime-data changes were included in the baseline diff.
- [ ] Manually verify one CLI commit and one editor commit after a fresh install. Confirm setup and checks work with the application stopped and no database credentials. Record what actually ran; the native-tool research does not prove the future application implementation or GitHub workflow already passes.

Reference behavior: [Git hooks](https://git-scm.com/docs/githooks), [hook path configuration](https://git-scm.com/docs/git-config#Documentation/git-config.txt-corehooksPath), [npm lifecycle](https://docs.npmjs.com/cli/v11/using-npm/scripts/), [lint-staged](https://github.com/lint-staged/lint-staged), [Prettier ignores](https://prettier.io/docs/ignore), and [Prettier CLI](https://prettier.io/docs/cli).
