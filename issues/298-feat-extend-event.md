# Issue #298: feat: Extend event_log to all application data and configuration changes

Source issue: https://github.com/five-panel/five-panel/issues/298

## TL;DR

Make change history automatic whenever the application saves a real data or configuration change. Record what changed, who caused it, and where it came from. Failed changes leave no successful-change event.

## What will change

**Large code impact:** most backend write paths need history and trusted action context.

Add shared history helpers and extend repository writes. Services keep validation, access checks, and coordination. Reuse the current `use…` composables, `useTransaction`, event format, append-only storage, and automation dispatch. Move document history out of `DataService` to avoid duplicate events.

Cover these saved areas:

| Data | Event entity types |
|---|---|
| Documents; models and fields | `custom_data`, `model` |
| Automation definitions | `automation` |
| Views, sections, reports; templates and assignments | `view`, `template` |
| Accounts; company memberships, roles and active state | `user`, `user_tenant` |
| Shares; saved sidebar placements | `entity_share`, `user_view` |
| Companies and settings; scheduled definitions | `tenant`, `scheduled_job` |
| Integration configuration; credentials | `integration_config`, `integration_credential` |
| Saved file records and lasting metadata | `file` |

Close existing bypasses, including tenant writes in `backend/src/index.ts` and the account update in `PasswordResetTokenRepository`. Include configuration changes made by sync workers.

## How it would work

1. **Extend existing services:** API calls, imports, workers and maintained script entry points supply trusted company, actor, source and operation details. Pass these per call; shared composables must not retain a caller’s context. Missing or mismatched context rejects a covered write.

2. **Add shared repository history:** save the change, its event and queued automation dispatch in the same database transaction. Reuse the caller’s transaction; only its owner completes it. Related writes within one backend action share an operation ID. That ID groups events; it does not remove distinct events or make an entire import one transaction.

3. **Extend each covered write path:** use `<entityType>.created/updated/deleted` for currently supported operations, with the exact affected record ID. Cover bulk writes and covered child records removed together, including shares, placements and templates. Reprocessing groups the new file row and resulting document changes under the current action, preserving the separate saved download operation ID.

4. **Reuse the logical value format:** updates save complete safe `before` values and only changed values in `after`; creates use null/full values and deletes full/null values. Preserve field paths and casing across JSON and dedicated columns. Reuse current locked snapshots and complete service results. Unchanged or timestamp-only writes produce no change event.

5. **Add safe value filtering:** detect real changes before removing secrets. Never store passwords, hashes, tokens, credential values or URLs containing access keys in snapshots or metadata. Secret-only changes record a safe changed marker. Keep provider-specific filtering in connectors. Global account changes, including password recovery, produce safe history in affected member companies using trusted account/membership data, without exposing other companies’ access details. Membership changes remain local to their company.

6. **Keep existing automations and scripts working:** keep document event versions, reconstruction of final values, existing automation starts and recovery of individual steps. New history types do not become automation triggers. Keep legacy service/repository calls working through maintained compatibility code without editing published scripts or helpers.

## Done when

- Every covered supported write produces exactly one event per affected record through its repository, including bulk and related deletions.
- PostgreSQL checks show changes, events and dispatch jobs commit or roll back together, including shared transactions and concurrent changes.
- Unchanged/timestamp-only writes add no events; secret-only changes remain visible without revealing secrets.
- Company isolation, trusted actors, operation grouping and account-versus-membership privacy hold across API, recovery, import, worker and maintained script paths.
- Existing validation, logical field names, complete record results, GraphQL ID responses, saved view settings and access rules still work.
- Document history, automation reconstruction/retries, partial import success, file reprocessing and maintained script compatibility keep working.

## Out of scope

- Derived views, schema/search caches, queue state, execution/run/step progress, import status/results, scheduler run times, sessions, reset tokens, rate limits and history’s own writes.
- Browser-only preferences and file bytes. Database/history rollback does not include filesystem changes.
- Rewriting published scripts/helpers or migrations; capturing their existing direct writes, bootstrap or restore operations; inventing past events.
- New history UI, undo/replay, retention changes, unsupported CRUD operations or new automation triggers. Import completion remains [#289](https://github.com/five-panel/five-panel/issues/289); external-ID changes remain [#294](https://github.com/five-panel/five-panel/issues/294).
- Tenant history/automation backup and restore remain [#290](https://github.com/five-panel/five-panel/issues/290). Current tenant backups omit these records; restoring a tenant with existing history still needs verification there.
