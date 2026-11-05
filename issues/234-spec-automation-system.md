# Issue #234: Automation system boundaries and execution model

Source issue: https://github.com/benitogonzalezh/five-panel/issues/234

## TL;DR

Build a shared sequential automation runner. A trigger starts named steps, each performing one operation and saving outputs for later steps.

Keep complete event data permanently in the existing `event_log`. Use Graphile for queued work, and separate automation tables for definitions and execution history. This provides a foundation for Shopify without putting Shopify rules into the runner.

## What will change

- **New code:** saved automations for each company, trigger matching, step execution, saved outputs, optional transaction groups, and admin APIs for management, history, and retries. Use `automations`, `automation_runs`, and `automation_step_runs`.
- **Focused changes:** extend `backend/src/services/EventLogService.ts` to record complete update snapshots; extend `DataService.ts` and the queue helper so related changes share a database transaction. Register automation jobs in `backend/src/worker/taskList.ts` and connect import completion.
- **Reuse:** `GraphileWorkerService.ts`, `JobScheduleService.ts`, existing record services, authentication, and import history.

**Code impact: Large.** Most work is the new runner and recovery history. Existing record saving and queue handling need focused changes. Full snapshots increase audit storage; queued jobs carry references.

## How it would work

1. **Define an automation.** Save its name, enabled state, trigger, conditions, and named steps. Start with validated definitions managed through company-admin APIs or setup scripts. Steps use registered operations, initially reading, creating, updating, and deleting records, with the same contract for checks and connector operations. Save the definition and version with each run so later edits cannot alter work already started.

2. **Save complete record events.** Keep the existing `custom_data.created`, `custom_data.updated`, and `custom_data.deleted` names. Create has `before: null` and the complete created data in `after`; update has complete `before` and `after` data plus `changedFields`; delete has the complete deleted data in `before` and `after: null`. Capture these during the mutation transaction. New update events use a new format version; existing events remain unchanged and are not automatically replayed. Conditions can check changed fields and unchanged values from that moment.

3. **Accept other triggers.** Support schedules, manual starts, verified webhooks, and import completion using explicit event types and their own saved business payloads in `event_log`. Resolve company identity through trusted authentication or connection settings. Repeated external deliveries reuse the original event identity. Import completion includes the file, models, import operation ID, and result counts. It means processing finished; counts can include failed rows. Save that operation ID with the import result. Commit the result, completion status, event, and dispatch job together; earlier row commits remain independent. Record-change events identify changed records, not unchanged or skipped input rows.

4. **Dispatch through Graphile.** Save each record change, its event, and a `dispatch-automation-event` job together. Other triggers save their event and job together. Jobs carry company and event IDs. The handler selects enabled automations by company, event type, and model, then uses [`json-rules-engine`](https://github.com/CacheControl/json-rules-engine) to evaluate nested AND, OR, and NOT conditions against saved event data. Custom comparisons are registered in code and only inspect supplied values. For each match, save one run linked to the event and queue its first step together. A unique company/automation/event rule prevents duplicate runs. No general subscriber registry is needed. Add jobs within the transaction through [Graphile's SQL API](https://worker.graphile.org/docs/sql-add-job).

5. **Pass saved outputs between steps.** Each step receives resolved inputs, performs one operation, and saves its output, status, attempts, and errors. Later steps can use references such as `{{ steps.find_product.output.id }}`. Whole-value references preserve their data type; invalid references fail clearly. For database steps, save changes, audit rows, successful output, and the next job together. The runner advances steps; handlers do not call one another.

6. **Optionally group database steps.** Independent steps are the default. A transaction group runs inside one job and shares one database transaction. Pass temporary outputs within that job. Save successful outputs and the next job in that transaction; they become visible only when it commits. Failure rolls back changes, audit rows, outputs, and queued jobs; record attempt details separately as rolled back. Retry the group with fresh outputs. External calls stay outside groups.

7. **Recover without repeating completed work.** Resume from saved progress after a restart. Repeated job delivery must not repeat committed steps. External handlers use stable operation identifiers; an uncertain result pauses for checking before another attempt. Preserve the originating event and parent run, and prevent an automation from repeatedly triggering itself through its own changes.

8. **Keep origin and history separate.** Every run has its own `operation_id` for changes it causes. Keep the triggering operation separately. Automation mutations use normal create/update/delete events with `source.type: automation` and run/step references in metadata. `automation_runs.id` identifies the run; `job_execution_id` is optional existing scheduled-job context, not required for every queued step. Show progress, outputs, and errors through admin history APIs. Import history links to related runs. Post-import checks belong in registered read-only steps; suspicious findings are warnings and do not undo imports or block reports. Keep event and run history without automatic pruning in this version.

## Done when

- Create, update, and delete events preserve the agreed data, and nested conditions correctly use unchanged field values despite later record changes.
- Failed record saves leave no mutation, audit event, or queued dispatch job; no-op updates produce no change event.
- Matching stays within the company, repeated delivery starts one run, and saved step outputs survive restarts without repeating committed effects.
- Failed transaction groups leave no business changes or audit events; retry uses fresh outputs and preserves rolled-back attempt details.
- Uncertain external outcomes wait for checking, and automation loops stop.
- Every supported trigger starts its configured steps; history explains failures and warnings, while existing imports and scheduled jobs still work.

## Out of scope

- A visual automation builder, parallel branches, and a general subscriber system.
- DBOS, Redis, a custom queue, and replacing existing scheduling.
- Shopify-specific handlers and tenant-specific duplicate policies.
- Automatic duplicate cleanup, report blocking, historical backfills, and retention cleanup.
- Changing or closing PR #236; that remains separate work.
