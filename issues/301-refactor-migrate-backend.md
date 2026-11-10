# Issue #301: refactor: Migrate backend application classes to composables with readable direct flows

Source issue: https://github.com/five-panel/five-panel/issues/301

## TL;DR

Make business operations easy to read from start to finish. Today, following a record save means jumping through setup code and callbacks. Use clear, named steps while keeping the same results and failure behavior.

## What will change

**Code impact: Large** — covers backend application layers and their maintained callers.

**New or converted code:** small TypeScript composables: `useX({ db, ... })` functions that prepare dependencies and return named operations. Cover services, repositories, runners, import processors, and related application helpers we own.

**Focused changes:** replace callback-based business flows in `services/context.ts`, `services/DataService.ts`, `repositories/ModelRepository.ts`, the automation service/runner, and `services/import/ImportOrchestrator.ts`. Include the direct transaction calls previously tracked by [#291](https://github.com/five-panel/five-panel/issues/291) in this issue. Update GraphQL, API, worker, scheduled-job, and supported command-line callers as each family moves. Paths here are under `backend/src/`.

**Reuse:** existing validation, field and relation rules, import transformations, event handling, Kysely, and Graphile. Use Kysely's [existing transaction support](https://kysely.dev/docs/examples/transactions/controlled-transaction). Reuse suitable transaction work already available in [#300](https://github.com/five-panel/five-panel/pull/300); recheck it and [#286](https://github.com/five-panel/five-panel/pull/286) before implementation so existing changes are included once.

## How it would work

1. Inventory the remaining application classes. Migrate records/repositories/events first, then automations, imports, and remaining application families. Keep every stage focused and working.
2. Create shared services once per application or worker context. Composables prepare ordinary dependencies internally. Create services tied to a transaction only for the operation using it. Keep user, company, run, import configuration, and transaction values local to each call.
3. Keep routes and handlers as direct functions calling services. Services check IDs, input shape, current models, record existence, company access, and relations; repositories own database access. Keep names such as `findById`, `create`, `update`, and `delete`, with one object argument.
4. Express operations as direct steps: load the current model and record, prepare and validate changes, compare, save, and record the event. Extract helpers when their names explain meaningful work or remove repetition. Functions inside a composable may share dependencies; remove callback wrappers that control business steps, pass dependencies, or start transactions. Include direct transaction calls for automation dispatch, execution, retry, recovery, and event creation.
5. Keep existing inputs, results, transaction rules, and service reuse unchanged. Verify this with behavior tests.

## Done when

- All inventoried application implementations use composables with readable direct operations, including automation transaction control. Required library callbacks remain at clear boundaries.
- Normal setup reuses shared services without callers assembling dependencies by hand. Concurrent calls cannot mix company, user, run, import, or transaction state.
- GraphQL saves, all four automation record operations and references, relation deletion checks, import counts, record inputs/results, saved event values, and unchanged-update timestamps/events retain their behavior.
- Real-service PostgreSQL/Graphile tests verify locks, savepoints, rollback of records/events/queued jobs, transaction ownership, cleanup after success/early returns/errors, retries, and recovery when an external result is unknown. External calls stay outside transactions.
- Focused behavior tests cover validation, conditions, references, transformations, custom operation/comparison settings, and import failures and ordering. Tests stay under package-level `test/` folders.
- Backend tests/build and lint for changed files pass, with runtime checks through Docker Compose and disposable databases. Affected frontend/ops checks pass; historical tenant scripts remain runnable through maintained compatibility support.

## Out of scope

- New features or automation operations; changes to public HTTP/GraphQL APIs, database schemas, record formats, or business/transaction rules.
- Replacing libraries, adding Vue runtime tools, or adding a dependency container or general workflow framework.
- Converting custom errors or library-required classes, editing dated tenant scripts, or unrelated frontend changes.
- Adding this refactor to the automation feature PR.
