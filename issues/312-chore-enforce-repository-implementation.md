# Issue #312: repository formatting implementation guide

Source issue: https://github.com/benitogonzalezh/five-panel/issues/312

[Spec](312-chore-enforce-repository.md) · [Technical implementation plan](312-chore-enforce-repository-implementation-plan.md)

This describes the planned change. The examples use a disposable Git checkout; the application has not been changed by these documents.

## How the pieces fit

Git already supports running a command before a commit. The new `.githooks/pre-commit` file provides that command. A small installer enables it when the developer runs root `npm ci` on the host with Node 24. Git saves the `.githooks` path in its local configuration. The hook file itself is tracked so the team shares its contents. Husky is not needed.

The hook calls the new root `format:staged` command. The added lint-staged tool finds the selected files and temporarily hides unselected edits in partly staged files. Existing Prettier checks their formatting using the existing rules. lint-staged then restores those edits, whether the check passes or fails. The hook reports errors and stops the commit until the author fixes them.

An initial formatting pass brings every supported repository file into line with the existing rules. This includes historical tenant scripts, published helpers, maintained runners, tests, configuration, and docs. Their behavior must stay the same. Keep the formatting pass in its own commit so it can be reviewed clearly, and update source-text test expectations only where formatting changed their spelling or spacing.

Existing backend and frontend formatting commands will read the root ignore files. Those still exclude dependencies and generated output; there are no special exclusions for tenant or published files. Unsupported file types are skipped.

CI checks the whole supported repository with the same formatting rules. It can catch a bad file even when that file was not selected for the latest commit.

## Concrete flow example

These JSON objects show complete example file contents for a test; they are not application API payloads. `staged` means the version selected for the next commit, while `working` is the file currently on disk.

1. The initial repository formatting pass is complete, and a developer has installed root dependencies. They change a tracked file, select the label change, and leave a comment unselected:

```json
{
  "path": "ops/test/format sample.ts",
  "head": "export const label = 'Draft';\n",
  "staged": "export const label = \"Ready\";\n",
  "working": "export const label = \"Ready\";\n// Finish the next change later.\n"
}
```

2. They run `git commit`. Git calls the native hook, which calls `format:staged`. lint-staged hides the unselected comment and passes the file name safely to Prettier. Prettier rejects the double quotes in this TypeScript file. The commit stops, and all the values above remain unchanged.
3. They run `npm run format -- "ops/test/format sample.ts"`, review the result, and use their editor to stage only the corrected label line. The comment stays unselected. They retry the commit; Prettier accepts it, lint-staged restores the comment, and Git saves:

```json
{
  "path": "ops/test/format sample.ts",
  "committed": "export const label = 'Ready';\n",
  "working": "export const label = 'Ready';\n// Finish the next change later.\n"
}
```

4. After a push, CI checks this file and the rest of the supported repository and passes. The uncommitted comment is still on the developer's disk. If the bad version had been committed by bypassing the hook, CI would report a formatting failure instead.

## Setup and recovery

When install scripts are disabled, `npm run hooks:install` enables the hook separately. Editors need Node and npm available in their environment. Missing tools or a formatter parser error stop the commit with instructions; setup never starts the application or database work.

`npm run format:all` explicitly formats the repository; `npm run format:check` checks it without writing changes. The normal commit hook checks only selected files. Each selected file is checked in full, so the author still reviews and selects formatting fixes carefully when part of a file is uncommitted.
