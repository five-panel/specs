# Issue #294: bug: DataService.update silently ignores externalId changes

Source issue: https://github.com/five-panel/five-panel/issues/294

## TL;DR

Allow callers to set, replace, or clear a record's external ID, and let imports assign or correct it. Save each real change with its history so automation sees the saved values. The record's internal `id` stays the same.

## What will change

**Code impact: Small.** Add external ID validation, change detection, and saving to the existing `useDataService` update path in `backend/src/services/DataService.ts`, with focused tests.

Reuse `CustomDataRepository`, its automatic change history, the shared transaction, and existing automation dispatch. These are already on `main`; no new history system or database change is needed. Keep the current `{ record, changed }` result.

See the [technical implementation plan](294-dataservice-update-silently-implementation-plan.md) and [short implementation guide](294-dataservice-update-silently-implementation.md).

## How it would work

1. **Extend the existing service:** an omitted or `undefined` external ID leaves it unchanged; `null` requests clearing; strings are saved as supplied, including empty strings. Keep existing model rules, including required fields, and reject other types through normal input validation.
2. Read the current model and record using the existing company checks, locks, and transaction. Compare the requested external ID alongside ordinary fields and import-source tracking. An external-ID-only change is a real update; unchanged input preserves the timestamp and creates no event.
3. **Reuse the repository save:** pass the external ID to its existing column, without adding a copy to the custom JSON fields. Keep existing model validation and uniqueness rules. Save the record, its history, and queued automation dispatch together; failure saves none of them.
4. **Reuse current history and automation:** history stores complete logical `before` values and changed values in `after`. The external ID stays at `before.externalId` and `after.externalId`, with existing safe-value filtering. Automation reconstructs the final record and includes `externalId` among its changed fields.
5. **Reuse the importer:** `ImportOrchestrator.findOrCreateRecord` already sends the mapped external ID to this service. Matching through another configured field can now assign or correct it, updating the existing record and import counts.

## Done when

- Setting, replacing, or clearing only the external ID saves it, returns the same internal record ID with `changed: true`, and produces exactly one update event.
- A combined external ID and ordinary-field update saves both together; the returned record contains the saved values.
- Omitted or `undefined` external IDs preserve the stored value. An unchanged external ID, including clearing an already-null value, causes no write, event, queued dispatch, or timestamp change when nothing else changes.
- History contains the correct old and new external IDs, including explicit null. Automation conditions and references see those logical fields and the correct changed-field list.
- An import matched through another field assigns or corrects the external ID without creating a duplicate; repeating unchanged input adds no update event or updated count.
- Service and PostgreSQL tests verify invalid input, duplicate external IDs, company access failures, and history or dispatch failures leave the action unchanged. Ordinary updates, model rules, and import-source tracking still work.

## Out of scope

- Changing the internal record ID, external ID uniqueness policy, database schema, or field storage.
- New screens, routes, import matching or source-snapshot rules, whole-import rollback, or provider-specific fixes.
- Rewriting past history, adding import-completion triggers, or changing automation retries and replay.
- Transaction API cleanup or changes to published tenant scripts.
