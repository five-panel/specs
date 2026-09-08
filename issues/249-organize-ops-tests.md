# Issue #249: Organize ops tests under a dedicated test folder

Source issue: https://github.com/benitogonzalezh/five-panel/issues/249

## Problem

Five Panel has no single repository-wide rule for where tests belong. The source issue began with four tests mixed with runnable files in `ops/`, but the same inconsistency exists across the backend and frontend. This makes test discovery harder to understand and allows new tests to be placed where package commands may not find them.

The repository should use one simple convention without changing application or operational behavior.

## Current Behavior

At the time of research, the repository has 116 test files spread across several layouts:

- Backend: 73 files, with 46 under `backend/test/`, 13 beside source files under `backend/src/`, and 14 under `backend/src/db/scripts/alci/__tests__/`.
- Frontend: 39 files, with 17 under `frontend/test/` and 22 under `frontend/tests/`.
- Ops: 4 files in the `ops/` root beside CLI entrypoints and runnable scripts.

The package test commands reflect these competing layouts:

- Backend searches both `test/**/*.test.ts` and `src/**/*.test.ts`.
- Frontend searches `test/*.test.js` and `tests/*.test.ts`.
- Ops searches `*.test.ts` in its package root.

CI calls each package's existing `npm test` command, but it does not reject a test placed outside the paths searched by those commands. Some backend tests also check for the current two-pattern backend command.

## Desired Behavior

Every maintained JavaScript or TypeScript test must live under the singular `test/` folder of the package it tests:

- `backend/test/**`
- `frontend/test/**`
- `ops/test/**`

Test filenames must keep the `.test.ts` or `.test.js` suffix. Package test commands must search only their own `test/` tree, and CI must reject tracked test files placed outside the three allowed trees. Existing test logic, package boundaries, and runtime behavior must remain unchanged.

## Scope

- Move every backend test currently outside `backend/test/` into that folder.
- Move every frontend test currently under `frontend/tests/` into `frontend/test/`.
- Move every ops test currently in the package root into `ops/test/`.
- Include any additional misplaced test files present on the implementation branch so the completed change leaves one convention across the whole repository.
- Preserve useful feature subfolders within each package's `test/` tree and use clear matching subfolders when moving tests out of `backend/src/`.
- Update relative imports, dynamic imports, file URLs, and file-system paths affected by the moves.
- Update the backend, frontend, and ops test scripts to search their package `test/` trees recursively.
- Update tests that assert the old backend test-script patterns.
- Add a dependency-free CI check that reports and rejects tracked `.test.ts` or `.test.js` files outside the allowed package test trees.
- Update `AGENTS.md` so contributors are told where tests belong and how to run all three package test suites.
- Keep the existing CI package commands unchanged: `npm --prefix backend test`, `npm --prefix frontend test`, and `npm --prefix ops test`.

## Out Of Scope

- Rewriting test cases or changing what they cover.
- Adding, removing, or intentionally renaming test cases.
- Changing the test runner or adding test dependencies.
- Converting JavaScript tests to TypeScript or TypeScript tests to JavaScript.
- Moving or refactoring application code, CLI entrypoints, backup and restore scripts, provisioning scripts, or other runtime files.
- Changing product, API, database, backup, restore, provisioning, or CLI behavior.
- Combining package tests into one repository-root test folder or one shared test command.
- Reorganizing already compliant tests solely to make their internal subfolders mirror source files.

## Acceptance Criteria

- Every tracked `.test.ts` and `.test.js` file is under `backend/test/`, `frontend/test/`, or `ops/test/`, according to the package it tests.
- No test files remain beside files under `backend/src/`, in `backend/src/**/__tests__/`, under `frontend/tests/`, or in the `ops/` package root.
- All 116 test files found during spec research are preserved by the move; if the implementation base contains additional tests, those tests also follow the new convention.
- A before-and-after inventory on the same implementation base shows that no test file or test case was lost.
- `backend/package.json` searches only `test/**/*.test.ts` for backend tests.
- `frontend/package.json` searches recursively for both `.test.js` and `.test.ts` files under `test/`.
- `ops/package.json` searches only `test/**/*.test.ts` for ops tests.
- The existing backend tests that validate package test scripts pass with expectations for the new single-folder pattern.
- The repository's CI check fails with a clear list of offending paths when a tracked `.test.ts` or `.test.js` file is outside the allowed package test trees.
- `npm --prefix backend test`, `npm --prefix frontend test`, and `npm --prefix ops test` each discover all tests for their package and pass.
- `npm --prefix backend run build` and `npm --prefix frontend run build` pass after the moves.
- CI continues to invoke the same three package test commands and all three jobs pass.
- `AGENTS.md` states the package-level `test/` convention and the three commands used to run the suites.
- No runtime source file or runnable ops script is moved or changed as part of this work.

## Implementation Notes

- Prefer tracked file moves so file history remains easy to follow.
- Refresh the test-file inventory from the implementation branch before moving files. The counts in this spec describe the researched `main` branch and are a baseline, not a reason to omit newer tests.
- For tests moved from beside backend source files, place them in a matching area under `backend/test/`. For example, a service test can move from `backend/src/services/Example.test.ts` to `backend/test/services/Example.test.ts`.
- Existing tests already under a package's singular `test/` folder do not need internal moves unless required to avoid a path collision.
- Pay special attention to tests that use `import.meta.url`, `__dirname`, `process.cwd()`, relative imports, or direct file reads. Their paths must still resolve from the new location and the package command's working directory.
- Keep explicit quoted recursive patterns in package scripts so discovery does not depend on shell expansion.
- The layout check may be a small script or a direct CI command. It should inspect tracked files, avoid new dependencies, print every misplaced path, and return a nonzero status when the convention is broken.
- Update the two backend tooling tests that currently require both `test/**` and `src/**` patterns; do not remove the coverage those checks provide.
- Moving backend tests out of `src/` means they will no longer be included by the backend TypeScript build. This is an expected packaging result; production code and behavior must not change.
- Package lockfiles should not change because no dependency changes are required.

## Test Expectations

- Record a test-file and test-case baseline from the implementation base before moving files, then compare it with the completed layout.
- Run `npm --prefix backend test` and confirm every backend test is discovered.
- Run `npm --prefix frontend test` and confirm every frontend test is discovered.
- Run `npm --prefix ops test` and confirm every ops test is discovered.
- Run the repository test-layout check and confirm it reports no misplaced files.
- Verify the layout check against a temporary misplaced test path, or test its path-validation logic directly, and confirm it returns a nonzero status with the offending path. Do not commit the temporary file.
- Run `npm --prefix backend run build`.
- Run `npm --prefix frontend run build`.
- Run `git diff --check`.

## Risks

- Moving many tests can break relative imports or file-system paths even when the test logic is unchanged.
- An incomplete package glob can silently skip a moved test. Comparing before-and-after inventories and test-case counts reduces this risk.
- The CI layout check can miss future tests if its supported suffix list and the package test commands drift apart. Keep those rules together and document both.
- Moving source-adjacent backend tests removes them from the backend compiler's `src/**/*` input. The backend build must confirm that no production module depended on a test file.
- Open development branches may add tests at old paths and can conflict with the move. The implementation should refresh the inventory after rebasing and place newly added tests under the package `test/` folder.

## Open Questions

None.
