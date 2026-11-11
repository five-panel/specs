# Issue #294: External ID updates implementation plan

> **For agentic workers:** use `superpowers:executing-plans` to implement the tasks in order. This document plans application changes; it does not contain an implemented fix.

**Goal:** save requested external ID changes through the current record service while preserving history, company checks, and existing callers.

**Architecture:** extend the `update` operation returned by `useDataService`. Services validate input and access; repositories map values to storage and automatically save history. Reuse the current transaction and automation event handling.

**Tech stack:** TypeScript, Kysely, PostgreSQL, Graphile Worker, and Node's test runner through `tsx`.

**Spec:** [agreed behavior and completion checks](294-dataservice-update-silently.md). [Short implementation guide and worked example](294-dataservice-update-silently-implementation.md).

## Current baseline and files

Rechecked application `main` at [`c3142a2`](https://github.com/five-panel/five-panel/commit/c3142a20994974b979a37f7fbbcb27e3b3c9ef7a). `DataService.ts` still removes `externalId` into `_externalId` before comparing or saving. A service-level reproduction with test repositories confirmed that an external-ID-only request reports unchanged, a combined request saves only the ordinary field, and explicit null is ignored. That reproduction did not use PostgreSQL.

| File under `backend/` | Responsibility |
| --- | --- |
| `src/services/DataService.ts` | Focused change: validate and compare the requested external ID, then include it in the existing repository update. |
| `src/repositories/CustomDataRepository.ts` | Reuse column storage, record scope, `customDataValues`, and `updateWithHistory`; no new repository operation is planned. |
| `src/repositories/changeHistory.ts`, `src/history/values.ts` | Reuse locked before/after records, logical field mapping, safe-value filtering, and automatic history. |
| `src/services/automation/event.ts`, `src/services/automation/recordUpdate.ts` | Reuse full final values, derived changed fields, and the existing record result. |
| `src/services/import/ImportOrchestrator.ts` | Reuse existing matching and mapped payloads; add coverage rather than an import-specific persistence path. |
| `test/services/DataServiceUpdateNoOp.test.ts`, `DataServiceInputs.test.ts`, `DataServiceEventLog.test.ts` | Extend focused input, result, and no-change coverage. |
| `test/services/import/ImportOrchestratorNoOpUpdates.test.ts` | Extend mapping and import-count coverage. |
| `test/integration/eventLog.test.ts`, `applicationHistory.test.ts`, `automations.test.ts` | Extend real PostgreSQL checks using their existing fixtures and transaction patterns. |

No new production file, route, class, table, or migration is planned. All test paths above are relative to `backend/`.

## Global constraints and contracts

- Keep the current composable and direct operation flow. Do not add a callback-based transaction wrapper or a service-owned history call.
- Keep internal record `id` immutable and preserve company/model scope, trusted user and event context, actor fields, current-model checks, and existing lock order.
- Preserve the database-wide `custom_data.external_id` uniqueness constraint from `005_create_custom_data.ts`; do not replace it with company- or model-local uniqueness. Existing model-field uniqueness checks remain in force.
- Keep model-field, companion, relation, and `importSourceKey` behavior. External ID validation must also work when no model field declares `externalId`.
- Keep import matching priority, partial-success behavior, and source-snapshot rules unchanged. Snapshot reconciliation still trims accepted external IDs and excludes blank/null values; exact service string storage does not change those rules.
- Reuse `useTransaction`: only the owner completes the transaction. Records, history, and dispatch jobs succeed or roll back together.
- Preserve event versions and safe-value filtering. Do not rewrite older events or add a second external ID outside logical record snapshots.

The existing service accepts one object containing either `modelId` or `model`, plus `id`, `data`, `user`, and optional `eventContext`. Keep its return type `Promise<DataUpdateResult>`, where `DataUpdateResult` is `{ record: DataRecord; changed: boolean }`. The returned record is flat and includes `id`, model fields, `externalId`, `importSourceKey`, `createdAt`, and `updatedAt`.

| Incoming `data.externalId` | Planned result |
| --- | --- |
| Omitted or `undefined` | Keep the stored external ID. |
| Primitive string | Save exactly that string, subject to existing model rules and database uniqueness. Do not trim or turn an empty string into null. |
| `null` | Request clearing, subject to existing model rules. Clearing an already-null value is unchanged when the record is valid. |
| Number, boolean, array, or object | Reject with normal service input validation, using the existing `DataInputError` contract. |

Keep the external ID outside `custom_data.data`. Pass it as `data.externalId` in the existing scoped repository update only when supplied after undefined values are removed. Return the saved record through the existing conversion; do not change callers to the old `{ id, changed }` shape.

For the guide's example, the existing event format stores this `data` for `custom_data.updated`, version 3:

```json
{
  "before": { "name": "Example", "externalId": "source-old", "importSourceKey": null },
  "after": { "externalId": "source-new" }
}
```

`automationEvent` reconstructs `after` by applying that patch to `before` and derives `changedFields: ["externalId"]`. Changed fields belong to the reconstructed automation event, not a new stored event format. Explicit null must remain in the patch.

## Review focus

Cover these conditions in the owning tasks below:

- A changed external ID with otherwise identical data must not take the no-change return.
- Omission, `undefined`, an unchanged string, and null-to-null must preserve the previous record and time.
- Invalid types and duplicate external IDs must not partly save ordinary fields.
- The record returned to an automation must contain the new external ID at the existing field path.
- An import matched by an ordinary field must correct the same row and remain unchanged on its next identical pass.

## Task 1: Validate and save external ID changes

**Modify:** `src/services/DataService.ts` and the focused service tests listed above.

**Interface:** preserve `useDataService(...).update(input): Promise<DataUpdateResult>` and `CustomDataRepository.update`.

- [ ] Add failing cases for assigning an external ID to null, replacing one, clearing one, changing it together with a normal field, and rejecting each invalid type. Assert the unchanged internal record ID, returned external ID, and repository column payload; assert no external ID is added to custom JSON.
- [ ] Correct the existing misleading no-change test: its fixture currently has a null external ID but sends `external-1` and expects no write. Use a truly equal stored external ID for the no-change case, and cover null-to-string separately.
- [ ] Add omission, explicit `undefined`, equal-string, null-to-null, empty-string, and preserved-whitespace cases. Assert no repository call and unchanged timestamps when the entire request is unchanged. Cover rejected types in a combined request to prove there is no partial write.
- [ ] Run these cases on the existing code and confirm the new mutation cases fail for the discarded external ID.
- [ ] In the existing `update` operation, detect a supplied external ID after undefined removal, validate it, derive the next value from the locked record, include its difference in change detection, and pass it to the existing repository update. Use the merged next logical values for model validation while keeping column-backed fields out of stored custom JSON. Cover older model definitions that declare `externalId`, including their required-field rules; new model definitions already reserve this name. Keep ordinary field uniqueness/relations and existing lock order intact.
- [ ] Rerun the focused service cases. Expected: all pass; an external-ID-only mutation returns the updated flat record with `changed: true`.

## Task 2: Verify repository history, rollback, and automation

**Extend tests:** `test/integration/eventLog.test.ts`, `applicationHistory.test.ts`, and `automations.test.ts`, using the actual services and repositories.

**Interface:** reuse `updateWithHistory`, `logicalHistoryValues`, `recordChange`, and `automationEvent`. The event and job contracts stay unchanged.

- [ ] Add real database cases for external-ID-only and combined changes, explicit clearing, unchanged input, and repeated updates. Assert the column, custom JSON, returned record, exactly one logical event, and the existing queued dispatch payload.
- [ ] Assert full old logical values and the changed-only new patch, including null. Check reconstructed `after.externalId`, `before.externalId`, `changedFields`, condition matching, and references. Verify `record.update` outputs the saved external ID for a later step.
- [ ] Reject an external ID already held by another row, including a row in another company. Preserve the existing uniqueness rule while keeping record reads and updates company-scoped.
- [ ] Test company/model access failures and invalid types. For combined updates, verify ordinary fields, external ID, timestamps, history, and dispatch remain unchanged after failure.
- [ ] Inject history-write and dispatch-enqueue failures through the existing database test patterns. Check that an owned action rolls back. For a caller-owned transaction, propagate the failure and verify rollback by its owner; inner services and history helpers must not complete that transaction.
- [ ] Retain ordinary update, model uniqueness, paired-field, import-source, and existing automation compatibility checks. Do not change history or runner production code merely to repeat capabilities already present.

## Task 3: Verify the existing import flow

**Extend tests:** `test/services/import/ImportOrchestratorNoOpUpdates.test.ts` and the existing PostgreSQL import fixtures in `test/integration/eventLog.test.ts`.

**Interface:** preserve `useImportOrchestrator(...).import(...)`, mapped `payload.externalId`, and the current `ImportResult` counters.

- [ ] Configure an ordinary field such as `name` as the only match field. Map `externalId` as a data column without `matchPriority`, so an external ID collision cannot select a different record first. Import `name = Example` with `externalId = source-new` against the existing `source-old` row; also cover an initially null external ID. Use the real record service for the PostgreSQL cases.
- [ ] For this base-record-only fixture, assert one existing record, the same internal record ID, the corrected external ID, one event, `successful: 1`, `created: 0`, and `updated: 1`. Repeat the identical import directly or through the existing reprocess flow and assert `updated: 0` with no additional record event. Do not use duplicate file upload as the repeat test; upload rejects identical content before import.
- [ ] Cover a duplicate-external-ID failure with reconciliation enabled and a stale record owned by the same source. Check existing row error reporting, `failed: 1`, no partial update to the matched record, and preservation of the stale record because failed rows prevent source-snapshot reconciliation. Do not require earlier reference-record writes or the whole import to roll back.
- [ ] After a successful correction to a normal nonblank external ID, verify reconciliation retains the corrected record and removes genuinely stale records only within the same company, model, and source.

## Verification and evidence

Before the fix, the seven focused files below passed **69 existing tests** at the reviewed main commit. They do not cover successful external ID mutation yet; one explicitly expects the bug. The new regression and PostgreSQL checks described here remain implementation work.

From the application repository, run the focused tests:

```bash
cd backend
npx tsx --test test/services/DataServiceUpdateNoOp.test.ts test/services/DataServiceInputs.test.ts test/services/DataServiceUnique.test.ts test/services/DataServiceEventLog.test.ts test/services/EventLogService.test.ts test/services/ChangeHistoryValues.test.ts test/services/import/ImportOrchestratorNoOpUpdates.test.ts
```

Run each PostgreSQL suite with its own explicitly supplied empty disposable database: `AUDIT_TEST_DATABASE_URL` for `test/integration/eventLog.test.ts`, `APPLICATION_HISTORY_TEST_DATABASE_URL` for `test/integration/applicationHistory.test.ts`, and `AUTOMATION_TEST_DATABASE_URL` for `test/integration/automations.test.ts`. Use the existing Docker/ops workflow to provision them. A skipped integration test does not verify the fix.

After implementation, run `npm --prefix backend test`, `npm --prefix backend run build`, focused lint on changed TypeScript files, and `git diff --check`. Expected: no new failures, all external ID regression cases pass, and database suites actually execute. This documentation change does not implement the fix or claim these new checks have passed.
