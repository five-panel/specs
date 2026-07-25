# Issue #229: spec: Audit events table for mutation provenance

Source issue: https://github.com/benitogonzalezh/five-panel/issues/229

## Problem

Five Panel imports can create and update rows in `custom_data`, but the resulting rows do not keep a durable link back to the import run, source file, file checksum, row, or job execution that produced the mutation.

This made the ALCI Cosecha SV 2026 duplicate import cleanup imprecise. The current `files` table preserves file/import history, but it is not linked to the `custom_data` rows created from those files. Cleanup scripts had to approximate with model-specific business filters, timestamps, file UUID extraction, and retained file metadata instead of selecting the precise rows created by one bad import operation.

## Current Behavior

Dynamic business records are stored in `custom_data` with `tenant_id`, `model_id`, JSONB `data`, optional `external_id`, `created_by`, `updated_by`, `created_at`, and `updated_at`.

`DataService.create`, `DataService.update`, and `DataService.delete` are the main application paths for `custom_data` mutations. Generated GraphQL mutations call those methods directly. The import pipeline also calls them through `ImportOrchestrator.findOrCreateRecord`.

The import system reads a file, processes rows, maps row fields into base and reference model payloads, and persists each payload with find-or-create semantics:

- a new payload creates a `custom_data` row
- a matched payload updates the row when the normalized data changes
- a matched payload returns `unchanged` when there is no data change
- skipped rows and failed rows do not mutate `custom_data`

File/import history is stored in `files.metadata`. Integration file metadata can include `toImport`, `statusImport`, `source`, `importConfig`, `importResult`, and `reprocessedFrom`. File records also store `checksum`, `original_name`, and `storage_path`.

Scheduled integrations create rows in `job_executions`, but the job context currently passes only `db` and `tenantId` into job scripts. The import path does not receive `job_execution_id`, and `job_executions` has retention cleanup that deletes old execution rows.

There is no append-only event log, no first-class `operation_id`, and no durable link from a `custom_data` mutation to a file record, source path, checksum, import config, request, script, or job execution.

## Desired Behavior

Five Panel should add one append-only `event_log` table. For this issue, the table records `custom_data` mutation provenance for imports. The table is intentionally named generically because future work may record other durable events, but this issue does not build an automation queue, event bus, subscriber system, or delivery workflow.

Each event records one actual mutation. A create event is emitted only when a row is inserted. An update event is emitted only when a row changes. A delete event is emitted only when a row is deleted through a path that supplies event context, such as a cleanup script. Unchanged import matches, skipped rows, failed rows, reads, downloads, view-only activity, and process-completed facts do not create events in this issue.

The initial table schema should be:

```sql
CREATE TABLE event_log (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id uuid NOT NULL REFERENCES tenants(id),

  event_type text NOT NULL,
  event_version integer NOT NULL DEFAULT 1 CHECK (event_version >= 1),

  operation_id uuid NOT NULL,
  entity_type text NOT NULL,
  entity_id text NOT NULL,
  model_id uuid NULL,

  data jsonb NOT NULL DEFAULT '{}'::jsonb,
  metadata jsonb NOT NULL DEFAULT '{}'::jsonb,

  occurred_at timestamptz NOT NULL DEFAULT now(),
  created_at timestamptz NOT NULL DEFAULT now()
);
```

The table should not add foreign keys to `custom_data`, `files`, `job_executions`, or `models` in the first version. Event history must remain readable after operational rows are deleted, pruned, or recreated. `tenant_id` should follow the repository's existing convention and reference `tenants.id`.

Initial event types are:

- `custom_data.created`
- `custom_data.updated`
- `custom_data.deleted`

Do not add database enum types or check constraints for `event_type`, source values, actor values, or provider values in this issue. Keep those names as application-level conventions so new event/source names do not require schema migrations.

Initial indexes should be:

```sql
CREATE INDEX idx_event_log_tenant_time
  ON event_log (tenant_id, occurred_at DESC, id DESC);

CREATE INDEX idx_event_log_operation
  ON event_log (tenant_id, operation_id, occurred_at ASC, id ASC);

CREATE INDEX idx_event_log_entity
  ON event_log (tenant_id, entity_type, entity_id, occurred_at ASC, id ASC);

CREATE INDEX idx_event_log_model
  ON event_log (tenant_id, model_id, occurred_at DESC, id DESC)
  WHERE model_id IS NOT NULL;

CREATE INDEX idx_event_log_source_file
  ON event_log (tenant_id, ((metadata->'source'->>'fileId')), occurred_at DESC)
  WHERE metadata->'source'->>'fileId' IS NOT NULL;

CREATE INDEX idx_event_log_source_fingerprint
  ON event_log (tenant_id, ((metadata->'source'->>'fingerprint')), occurred_at DESC)
  WHERE metadata->'source'->>'fingerprint' IS NOT NULL;
```

The top-level columns have stable meanings:

- `event_type`: what happened, using dot-separated names such as `custom_data.created`.
- `event_version`: payload version for the specific `event_type`.
- `operation_id`: one logical operation that may produce many event rows.
- `entity_type`: the mutated entity kind, initially `custom_data`.
- `entity_id`: the mutated entity id stored as text so the table can later support non-UUID ids if needed.
- `model_id`: the `custom_data.model_id` for model-level cleanup and debugging.
- `data`: the business mutation payload, including before/after values.
- `metadata`: flexible provenance and execution context.
- `occurred_at`: when the domain event happened.
- `created_at`: when the `event_log` row was inserted.

For imports:

- `operation_id` is generated once at the start of each logical file import run.
- All reference-model and base-model `custom_data` mutations produced while importing that file share the same `operation_id`.
- Reprocessing the same file creates a new file history row today and should also create a new `operation_id`.
- A scheduled job execution can import multiple files; each file import gets its own `operation_id`, and all events produced by that job carry the same `metadata.import.jobExecutionId`.
- Manual single-file imports and reprocesses have no `metadata.import.jobExecutionId`.

The `data` payload stores before/after mutation values. Values should use the same custom-data field identifiers used by `custom_data.data`.

For create events, include the created values:

```json
{
  "before": null,
  "after": {
    "fieldId": "created value"
  },
  "externalId": "optional external id",
  "businessKey": {
    "fieldOrExternalId": "value used for import matching"
  }
}
```

For update events, include only changed fields:

```json
{
  "before": {
    "fieldId": "old value"
  },
  "after": {
    "fieldId": "new value"
  },
  "changedFields": ["fieldId"],
  "externalId": "optional external id",
  "businessKey": {
    "fieldOrExternalId": "value used for import matching"
  }
}
```

For delete events, include the deleted row snapshot:

```json
{
  "before": {
    "fieldId": "deleted value"
  },
  "after": null,
  "externalId": "optional external id",
  "businessKey": {
    "fieldOrExternalId": "value used for cleanup or import matching"
  }
}
```

The `metadata` payload stores provenance/context. The initial import shape should be:

```json
{
  "source": {
    "type": "import",
    "provider": "dropbox",
    "fileId": "files.id",
    "path": "human-safe source identifier",
    "fingerprint": "files.checksum"
  },
  "import": {
    "rowNumber": 42,
    "sheetName": "Sheet1",
    "jobExecutionId": "optional job_executions.id",
    "integrationConfigId": "optional integration config id",
    "reprocessedFrom": "optional original file id",
    "matchColumns": ["external_id"],
    "matchedBy": {
      "field": "external_id",
      "value": "business key value"
    },
    "result": "created"
  },
  "actor": {
    "type": "user",
    "id": "user id",
    "userId": "user id"
  }
}
```

`metadata.source.provider` should initially use `dropbox`, `google-drive`, or `local`. `metadata.source.path` should be the most useful non-secret human identifier available: Dropbox `importConfig.sourcePath` for path sources, `shared_link:<sha256(sharedLink)>` for Dropbox shared links, `folder:<folderId>/file:<fileName>` for Google Drive, or the uploaded file `original_name` for local manual uploads.

Neither `data` nor `metadata` should store integration credentials, raw file contents, access tokens, or full shared links.

Event writes must be transactional with the data mutation. If the `custom_data` mutation rolls back, its event rolls back. If the event insert fails, the `custom_data` mutation rolls back.

`event_log` should be append-only. The application should expose only insert/read repository methods. The migration should also add a small trigger that rejects `UPDATE` and `DELETE` against `event_log`.

## Scope

- Add a backend migration that creates `event_log`, its indexes, and the append-only trigger.
- Add Kysely `EventLogTable` types to `backend/src/db/types.ts`, with `eventLog: EventLogTable` in `Database` and DB aliases following the existing `EventLogDB`, `NewEventLogDB`, and `EventLogUpdateDB` naming pattern.
- Add an `EventLogRepository` with insert/read methods only.
- Add an `EventLogService` or small service-level helper if needed to keep event payload construction out of repositories.
- Wire event logging into `createServiceContext`.
- Extend `DataService.create`, `DataService.update`, and `DataService.delete` to accept optional event context and write `event_log` rows in the same database transaction as the `custom_data` mutation when that context is supplied.
- Extend import types so `importSingleFile`, `importFiles`, and `ImportOrchestrator.import` can pass file, source, operation, job execution, actor, and row metadata into `DataService`.
- Update scheduled job execution context so provider job scripts receive the current `job_execution_id` and pass it through import calls.
- Document and test cleanup queries that select rows by `operation_id`, by source file id, by source fingerprint, and by entity id.

## Out Of Scope

- Building automation workers, event subscribers, queues, retries, replay, or delivery state.
- Treating `event_log` as an outbox table.
- Building a UI for browsing events.
- Adding GraphQL or REST read APIs for events unless needed for tests.
- Logging process facts such as `import.completed`.
- Logging normal reads, downloads, exports, report views, or support/admin access events.
- Auditing every direct GraphQL/user mutation that does not pass event context.
- Reworking import duplicate detection, match key generation, or Cosecha-specific import configs.
- Backfilling full provenance for existing `custom_data` rows where no reliable source can be proven.
- Changing `files` history retention, `job_executions` retention, or physical upload storage behavior.
- Adding a generic JSONB GIN index or retention/archive system for `event_log`.

## Acceptance Criteria

- A migration creates `event_log` with the columns listed in Desired Behavior.
- The migration creates the six indexes listed in Desired Behavior.
- The migration adds a database trigger that rejects `UPDATE` and `DELETE` against `event_log`.
- Backend database types include `eventLog: EventLogTable` and exported selectable, insertable, and updateable type aliases consistent with existing `DB` suffix conventions.
- `EventLogRepository` exposes insert/read methods only and does not expose update or delete methods.
- Import-created `custom_data` rows produce one `custom_data.created` event in the same transaction as the row insert.
- Import-updated `custom_data` rows produce one `custom_data.updated` event in the same transaction as the row update.
- Import-updated events include sparse `data.before`, `data.after`, and `data.changedFields` values for only the fields changed by that update.
- Import create events include `data.before = null` and `data.after` with the created `custom_data.data` values.
- Delete calls that supply event context produce one `custom_data.deleted` event with `data.before` containing the deleted row data and `data.after = null`.
- Unchanged import matches, skipped import rows, failed import rows, and read-only actions do not create events.
- A single file import generates one `operation_id`, and all events for that file import share it.
- One scheduled job execution importing multiple files creates separate `operation_id` values per file while retaining the same `metadata.import.jobExecutionId`.
- Manual file imports and reprocesses create import events without `metadata.import.jobExecutionId`.
- Import events include `metadata.source.type = "import"`, `metadata.source.fileId = files.id`, `metadata.source.fingerprint = files.checksum`, and the mapped provider value.
- Import events include row-level metadata sufficient to identify the source row, including row number when available.
- Cleanup code can select rows affected by a bad import using `operation_id` without relying on broad campo/date filters.
- Cleanup code can select rows affected by a bad source file using `metadata.source.fileId`.
- Cleanup code can select rows affected by repeated imports of the same file content using `metadata.source.fingerprint`.
- Existing direct GraphQL create, update, and delete mutations keep their current behavior when no event context is supplied.
- Existing `custom_data` rows are not given synthetic create events unless a separate tenant-specific backfill can prove provenance.
- Event payloads never store raw file contents, integration credentials, access tokens, or full shared links.

## Implementation Notes

Add the migration after the current latest migration in `backend/src/db/migrations`. Keep the database table name `event_log` and the Kysely table key `eventLog`.

The append-only trigger can be implemented as a small PostgreSQL function:

```sql
CREATE FUNCTION prevent_event_log_mutation()
RETURNS trigger AS $$
BEGIN
  RAISE EXCEPTION 'event_log is append-only';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER event_log_no_update_delete
BEFORE UPDATE OR DELETE ON event_log
FOR EACH ROW EXECUTE FUNCTION prevent_event_log_mutation();
```

The `down` migration should drop `event_log_no_update_delete`, then `prevent_event_log_mutation`, then the indexes/table.

Create explicit TypeScript types for event context rather than passing arbitrary metadata through every layer. A practical shape is:

```ts
type EventLogMutationContext = {
  operationId: string;
  occurredAt?: Date;
  source: {
    type: 'import' | 'manual' | 'script' | 'api' | 'system';
    provider?: string | null;
    fileId?: string | null;
    path?: string | null;
    fingerprint?: string | null;
  };
  import?: {
    rowNumber?: number | null;
    sheetName?: string | null;
    jobExecutionId?: string | null;
    integrationConfigId?: string | null;
    reprocessedFrom?: string | null;
    matchColumns?: string[];
    matchedBy?: { field: string; value: unknown } | null;
  };
  actor?: {
    type: 'user' | 'worker' | 'system' | 'script' | 'api_key';
    id?: string | null;
    userId?: string | null;
  };
  metadata?: Record<string, unknown>;
};
```

For imports, build an operation context in `importSingleFile` after loading the `FileRecord` and before calling `ImportOrchestrator.import`. Pass the generated `operationId`, full `FileRecord`, optional `jobExecutionId`, source mapping, and actor context into the orchestrator. The orchestrator should add row-specific metadata before each `DataService` mutation.

For update events, compute the diff from the existing row and the normalized new data before writing. The import flow already needs to know whether a matched row changed; reuse that comparison path where possible. Store only changed keys in `before`, `after`, and `changedFields`.

For create events, `after` can use the values being inserted. For delete events, use the row snapshot already loaded for deletion or load the row inside the same transaction before deleting.

`DataService` should write events only when a caller supplies `EventLogMutationContext`. Direct GraphQL mutations should not silently become audited by this issue because they do not provide source-file provenance.

For imports, `findOrCreateRecord` should return enough information to capture the lookup path:

- which match columns were tried
- which field/value matched an existing row
- whether the final result was created, updated, or unchanged

The existing `created_by` and `updated_by` fields should continue to use the current mutation user. Event `metadata.actor` can represent the initiating user or execution principal separately. For scheduled imports, use `actor.type = "worker"` and `actor.id = jobExecutionId`. For user-triggered imports, use `actor.type = "user"`, `actor.id = req.user.id`, and `actor.userId = req.user.id`.

`JobExecutor.executeJob` currently creates a `job_executions` row before loading the provider job script. Extend the job script context with `jobExecutionId: execution.id`. The Dropbox and Google Drive job scripts should pass that id into `importFiles`. The `/integrations/import`, `/integrations/run`, `/integrations/import/:fileId`, and `/files/:fileId/reprocess` routes should pass the authenticated request user as the event actor where available.

Cleanup examples:

Find rows created by one bad file import:

```sql
SELECT entity_id AS custom_data_id, model_id, operation_id, occurred_at
FROM event_log
WHERE tenant_id = $1
  AND event_type = 'custom_data.created'
  AND operation_id = $2;
```

Find rows created by one source file:

```sql
SELECT entity_id AS custom_data_id, model_id, operation_id
FROM event_log
WHERE tenant_id = $1
  AND event_type = 'custom_data.created'
  AND metadata->'source'->>'fileId' = $2;
```

Find rows created by repeated imports of the same file content:

```sql
SELECT entity_id AS custom_data_id, model_id, operation_id, metadata->'source'->>'fileId' AS file_id
FROM event_log
WHERE tenant_id = $1
  AND event_type = 'custom_data.created'
  AND metadata->'source'->>'fingerprint' = $2;
```

Find the history for one `custom_data` row:

```sql
SELECT event_type, operation_id, occurred_at, data, metadata
FROM event_log
WHERE tenant_id = $1
  AND entity_type = 'custom_data'
  AND entity_id = $2
ORDER BY occurred_at ASC, id ASC;
```

Cleanup scripts that delete rows selected from `event_log` should create a fresh cleanup `operation_id` and delete through `DataService.delete` with event context when relationship validation is desired. Those delete events should use `metadata.source.type = "script"` and metadata linking back to the bad import `operation_id` or source file.

## Test Expectations

- Migration tests or focused database checks verify table creation, indexes, and append-only trigger behavior.
- Repository/service tests verify that events can be inserted and read, and cannot be updated or deleted through exposed repository methods.
- `DataService` tests cover create with event context, changed update with event context, unchanged update with event context, delete with event context, and no event when context is omitted.
- Transaction tests verify that a failed event insert rolls back the `custom_data` mutation.
- Import orchestrator tests cover one operation id per file import, row metadata, source mapping from `files.metadata.source`, file checksum propagation, sparse update diffs, and no events for skipped, failed, or unchanged rows.
- Scheduled job tests cover passing `jobExecutionId` from `JobExecutor` into provider job scripts and then into import event metadata.
- Cleanup-query tests or SQL fixtures demonstrate selecting `custom_data` ids by `operation_id`, by `metadata.source.fileId`, by `metadata.source.fingerprint`, and by entity id.
- Existing backend build and focused backend tests pass with `npm --prefix backend run build` and `npm --prefix backend run test` or the narrower test commands used by the implementation branch.

## Risks

- Event rows may contain business data snapshots from `custom_data.data`; access to future event read surfaces must follow tenant scoping and avoid exposing sensitive fields casually.
- Recording events synchronously increases write volume during large imports. The first implementation should keep update payloads sparse and avoid broad JSONB indexes.
- If event inserts are not in the same transaction as `custom_data` mutations, cleanup and debugging can still have gaps. Transactional writes are required for trust in this table.
- JSON metadata can drift if builders are loose. The implementation should centralize import event construction and cover the expected shape in tests.
- Existing legacy rows remain partially untraceable unless a separate tenant-specific backfill can prove their source. The new table improves future provenance and future cleanup precision.

## Open Questions

None.
