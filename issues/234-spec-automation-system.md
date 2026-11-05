# Issue #234: Automation system boundaries and execution model

Source issue: https://github.com/benitogonzalezh/five-panel/issues/234

## TL;DR

Build a shared sequential automation runner. A trigger starts named steps, each performing one operation and saving outputs for later steps.

Keep complete event data permanently in the existing `event_log`. Use Graphile for queued work, and separate automation tables for definitions and execution history. This provides a foundation for Shopify without putting Shopify rules into the runner.

## What will change

- **New code:** saved automations for each company, trigger matching, independent steps, saved outputs, step-defined retries, and admin APIs for management and history. Use `automations`, `automation_runs`, and `automation_step_runs`.
- **Focused changes:** extend `backend/src/services/EventLogService.ts` for complete update snapshots; extend `DataService.ts` and the queue helper so each database step saves changes, progress, and jobs together. Register automation jobs in `backend/src/worker/taskList.ts` and connect import completion.
- **Reuse:** `GraphileWorkerService.ts`, `JobScheduleService.ts`, existing record services, authentication, and import history.

**Code impact: Large.** Most work is the new runner and recovery history. Existing record saving and queue handling need focused changes. Full snapshots increase audit storage; queued jobs carry references.

## How it would work

1. **Define an automation.** Save each company's definitions through validated company-admin APIs or setup scripts. Each has a name, enabled state, trigger, conditions, and named steps. Save its definition and version with every run. Step names describe intent; registered operations remain generic, initially record reads, creates, updates, and deletes. The runner calls handlers; record handlers call existing services. Keep checks and future connector operations behind the same contract.

2. **Save complete record events.** Keep `custom_data.created`, `custom_data.updated`, and `custom_data.deleted`. Create has `before: null` and complete created data in `after`; update has complete `before` and `after` plus `changedFields`; delete has complete deleted data in `before` and `after: null`. Capture these during the mutation transaction. New updates use a new event format version; existing events stay unchanged and are not automatically replayed. Conditions use changed fields and unchanged values from that moment.

3. **Accept other triggers.** Schedules, manual starts, verified webhooks, and import completion save explicit business event types and their own data in `event_log`. Derive the company from trusted authentication or connection settings. Repeated external deliveries reuse the original event identity. Import completion includes file, models, import operation ID, and counts, including failed rows: finished does not mean every row succeeded. Save that operation ID with the import result. Commit the result, completion status, event, and dispatch job together; earlier row commits stay independent. Record events exclude unchanged or skipped input rows.

4. **Dispatch through Graphile.** Save each record change, its event, and a `dispatch-automation-event` job together. Other triggers save their event and job together. Jobs carry company and event IDs. Select enabled automations by company, event type, and model. Use [`json-rules-engine`](https://github.com/CacheControl/json-rules-engine) for nested AND, OR, and NOT conditions against saved event data. Custom comparisons are registered in code and only inspect supplied values. Save each matching run and its first job together; a unique company/automation/event rule prevents duplicates. Add jobs within the transaction through [Graphile's SQL API](https://worker.graphile.org/docs/sql-add-job).

5. **Execute one step.** The runner resolves inputs, calls one handler, and saves output, status, attempts, and errors. Whole-value references such as `{{ steps.find_product.output.id }}` preserve types; invalid references fail clearly. Handlers may share helpers but do not advance steps or manage retries. Execution, history, and retries use the run's trusted `tenant_id`. Reject references to other tenants' records, models, runs, or outputs. For database steps, save business changes, audit events, successful output, and the next job in one transaction.

6. **Retry independent steps.** Each step defines maximum attempts, delay, and retryable failures; no retry policy means one attempt. The runner applies that policy only to safe temporary failures. Invalid inputs or configuration fail without retry. Retry only the failed step: earlier completed steps, outputs, and effects remain. A failed database step rolls back its changes, audit events, successful output, and jobs; save failure details separately. Queue redelivery checks saved state and never overrides the step's policy.

7. **Recover without repeating completed work.** Resume saved progress after restart without repeating committed steps. External actions reuse stable identifiers across attempts. Confirmed success saves the recovered result and advances without another call. Retry only after confirmed non-execution or when the service guarantees duplicate-safe replay. Unknown results remain paused, including through manual retry APIs. Preserve the originating event and parent run, and stop self-trigger loops.

8. **Keep origin and history separate.** Every run has its own `operation_id`; keep the triggering operation separately. Automation mutations use normal create/update/delete events with `source.type: automation` and run/step references in metadata. `automation_runs.id` identifies the run; `job_execution_id` remains optional scheduled-job context. History APIs show progress, outputs, and errors; import history links related runs. Post-import checks use registered read-only steps; suspicious findings are warnings, not grounds to undo imports or block reports. Keep event and run history without automatic pruning.

## Done when

- Create, update, and delete events preserve the agreed data, and nested conditions correctly use unchanged field values despite later record changes.
- Failed database steps roll back business changes, audit events, successful outputs, and jobs from that transaction; no-op updates emit no change event.
- Matching, execution, references, history, and retries reject cross-tenant access; repeated event delivery starts one run.
- Step retry settings are honored; restart and retry preserve completed steps, saved outputs, and earlier effects without repeating them.
- Unknown external outcomes stay paused; confirmed results resume safely, and self-trigger loops stop.
- Every supported trigger starts its configured steps; history explains failures and warnings, while existing imports and scheduled jobs still work.

## Out of scope

- A visual automation builder, parallel branches, and a general subscriber system.
- Multi-step transaction groups for now, automatic undo of completed steps, and service splitting.
- DBOS, Redis, a custom queue, and replacing existing scheduling.
- Shopify-specific handlers and tenant-specific duplicate policies.
- Automatic duplicate cleanup, report blocking, historical backfills, and retention cleanup.
- Changing or closing PR #236; that remains separate work.
