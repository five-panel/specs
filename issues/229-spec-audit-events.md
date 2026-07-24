# Issue #229: spec: Audit events table for mutation provenance

Source issue: https://github.com/benitogonzalezh/five-panel/issues/229

## Problem

Five Panel imports can create and update rows in `custom_data`, but the resulting rows do not retain enough provenance to identify the exact logical operation, file, integration source, or job run that produced each mutation.

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

There is no append-only audit event table, no first-class `operation_id`, and no durable link from a `custom_data` mutation to a file record, source path, checksum, import config, request, script, or job execution.

## Desired Behavior

Five Panel should add an append-only `audit_events` table for mutation provenance. The initial producer is `custom_data` mutations, starting with imports, but the table name and schema should remain generic enough for future audited mutations without becoming a domain event bus.

Each audit event records one actual mutation. A create event is emitted only when a row is inserted. An update event is emitted only when a row changes. A delete event is emitted only when a row is deleted. Unchanged import matches, skipped rows, failed rows, reads, downloads, and view-only activity do not create audit events.

The initial table schema should be:

```sql
CREATE TABLE audit_events (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id uuid NOT NULL REFERENCES tenants(id),
  operation_id uuid NOT NULL,
  job_execution_id uuid NULL,

  event_type varchar(160) NOT NULL,
  event_version integer NOT NULL DEFAULT 1 CHECK (event_version >= 1),
  operation varchar(16) NOT NULL CHECK (operation IN ('create', 'update', 'delete')),
  entity_type varchar(80) NOT NULL,
  entity_id uuid NOT NULL,
  model_id uuid NULL REFERENCES models(id),

  actor_type varchar(32) NOT NULL CHECK (
    actor_type IN ('user', 'system', 'script', 'api_key', 'integration', 'worker')
  ),
  actor_id text NULL,
  user_id uuid NULL REFERENCES users(id),

  occurred_at timestamptz NOT NULL,
  recorded_at timestamptz NOT NULL DEFAULT now(),
  request_id varchar(128) NULL,
  trace_id varchar(128) NULL,

  source_type varchar(32) NOT NULL CHECK (
    source_type IN ('manual', 'import', 'script', 'api', 'system')
  ),
  source_provider varchar(64) NULL,
  source_id text NULL,
  source_path text NULL,
  source_fingerprint text NULL,

  data jsonb NOT NULL DEFAULT '{}'::jsonb,
  metadata jsonb NOT NULL DEFAULT '{}'::jsonb,

  CONSTRAINT audit_events_custom_data_requires_model
    CHECK (entity_type <> 'custom_data' OR model_id IS NOT NULL)
);
```

`job_execution_id` is intentionally nullable and should not have a foreign key in the initial migration. Current job execution retention deletes old execution rows; the audit table must preserve the execution id value even after operational job history is pruned. If the product later retains `job_executions` permanently, a separate migration can add a foreign key.

Initial indexes should be:

```sql
CREATE INDEX idx_audit_events_entity_history
  ON audit_events (tenant_id, entity_type, entity_id, occurred_at DESC, id DESC);

CREATE INDEX idx_audit_events_operation
  ON audit_events (tenant_id, operation_id, occurred_at ASC, id ASC);

CREATE INDEX idx_audit_events_source
  ON audit_events (tenant_id, source_type, source_provider, source_id, occurred_at DESC);

CREATE INDEX idx_audit_events_source_fingerprint
  ON audit_events (tenant_id, source_fingerprint, occurred_at DESC)
  WHERE source_fingerprint IS NOT NULL;

CREATE INDEX idx_audit_events_job_execution
  ON audit_events (tenant_id, job_execution_id, occurred_at DESC)
  WHERE job_execution_id IS NOT NULL;

CREATE INDEX idx_audit_events_custom_data_model_operation
  ON audit_events (tenant_id, model_id, operation, occurred_at DESC)
  WHERE entity_type = 'custom_data';
```

The initial `custom_data` event types are:

- `fivepanel.custom_data.created` with `operation = 'create'`
- `fivepanel.custom_data.updated` with `operation = 'update'`
- `fivepanel.custom_data.deleted` with `operation = 'delete'`

Allowed value sets are:

- `operation`: `create`, `update`, `delete`
- `source_type`: `manual`, `import`, `script`, `api`, `system`
- `actor_type`: `user`, `system`, `script`, `api_key`, `integration`, `worker`

Initial recognized `source_provider` values are `dropbox`, `google-drive`, and `local`, but `source_provider` should remain a nullable string rather than a checked value. New integrations should be able to add provider names without a schema migration.

For imports:

- `operation_id` is generated once at the start of each logical file import run.
- All reference-model and base-model `custom_data` mutations produced while importing that file share the same `operation_id`.
- Reprocessing the same file creates a new file history row today and should also create a new `operation_id`.
- A scheduled job execution can import multiple files; each file import gets its own `operation_id`, and all of those operation ids carry the same `job_execution_id`.
- Manual single-file imports and reprocesses have `job_execution_id = NULL`.
- Import mutations use `source_type = 'import'`.
- `source_provider` maps file metadata source values as follows: `dropbox` to `dropbox`, `google-drive` to `google-drive`, and `manual` to `local`.
- `source_id` is the `files.id` of the file history row being imported.
- `source_fingerprint` is `files.checksum`.
- `source_path` is the most useful non-secret human identifier available: Dropbox `importConfig.sourcePath` for path sources, `shared_link:<sha256(sharedLink)>` for Dropbox shared links, `folder:<folderId>/file:<fileName>` for Google Drive, or the uploaded file `original_name` for local manual uploads.

The import audit metadata should include the row and configuration context needed to debug a mutation without reparsing logs:

- `import.file_id`
- `import.original_name`
- `import.storage_path`
- `import.sheet_name`
- `import.row_number` using the user-visible sheet row number
- `import.base_model`
- `import.model_slug`
- `import.integration_config_id` when available from `files.metadata.integrationTaskId` or the import config source
- `import.reprocessed_from` when `files.metadata.reprocessedFrom` exists
- `import.match_columns` from the payload lookup strategy
- `import.matched_by` for updates, including the matched field and value
- `import.result` with `created` or `updated`

`data` and `metadata` have separate purposes:

- `data` stores business-level mutation payloads and diffs.
- `metadata` stores technical execution, source, row, lookup, and debugging context.
- Neither column should store integration credentials, raw file contents, access tokens, or full shared links.

For `custom_data` create events, `data` should include:

```json
{
  "externalId": "optional external id",
  "after": {
    "fieldId": "created value"
  },
  "businessKey": {
    "fieldOrExternalId": "value used for import matching"
  }
}
```

For `custom_data` update events, `data` should include only changed fields, plus the record identity:

```json
{
  "externalId": "optional external id",
  "changedFields": ["fieldId"],
  "before": {
    "fieldId": "old value"
  },
  "after": {
    "fieldId": "new value"
  },
  "businessKey": {
    "fieldOrExternalId": "value used for import matching"
  }
}
```

For `custom_data` delete events, `data` should include the deleted row snapshot:

```json
{
  "externalId": "optional external id",
  "before": {
    "fieldId": "deleted value"
  },
  "businessKey": {
    "fieldOrExternalId": "value used for cleanup or import matching"
  }
}
```

`operation_id` is the primary grouping key for cleanup and debugging. `job_execution_id` is only scheduler infrastructure context.

Append-only behavior should be enforced in the database with a trigger that rejects `UPDATE` and `DELETE` against `audit_events`. The application should also expose only insert/read repository methods for audit events. The migration `down` path can drop the trigger before dropping the table.

## Scope

- Add a backend migration that creates `audit_events`, indexes, check constraints, and the append-only trigger.
- Add Kysely `AuditEventsTable` types to `backend/src/db/types.ts`.
- Add an `AuditEventRepository` and `AuditEventService` following existing repository/service naming and object-parameter conventions.
- Wire the audit service into `createServiceContext`.
- Extend `DataService.create`, `DataService.update`, and `DataService.delete` to accept optional audit context and write audit events in the same database transaction as the `custom_data` mutation.
- Extend import types so `importSingleFile`, `importFiles`, and `ImportOrchestrator.import` can pass file, source, operation, job execution, actor, and row metadata into `DataService`.
- Update scheduled job execution context so provider job scripts receive the current `job_execution_id` and pass it through import calls.
- Preserve direct GraphQL mutation behavior by generating a new `operation_id` per mutation when no caller-supplied operation context exists.
- Document and test cleanup queries that select rows by `operation_id`, `source_id`, and `source_fingerprint`.

## Out Of Scope

- Building a UI for browsing audit events.
- Adding GraphQL or REST read APIs for audit events unless needed for tests.
- Logging normal reads, downloads, exports, report views, or support/admin access events.
- Reworking import duplicate detection, match key generation, or Cosecha-specific import configs.
- Backfilling full provenance for existing `custom_data` rows where no reliable source can be proven.
- Changing `files` history retention or physical upload storage behavior.
- Turning `audit_events` into a generic domain event, notification event, webhook, or queue table.

## Acceptance Criteria

- A migration creates `audit_events` with all fields listed in Desired Behavior, including `operation_id`, nullable `job_execution_id`, source fields, actor fields, `data`, `metadata`, and the custom-data model constraint.
- The migration creates the six indexes listed in Desired Behavior.
- The database rejects `UPDATE` and `DELETE` statements against `audit_events` through an append-only trigger.
- `operation`, `source_type`, and `actor_type` are constrained to the allowed values defined in Desired Behavior.
- Backend database types include `auditEvents: AuditEventsTable` and exported selectable, insertable, and updateable type aliases consistent with existing `DB` suffix conventions.
- `AuditEventRepository` exposes insert and read methods only; it does not expose update or delete methods.
- `DataService.create` records one `fivepanel.custom_data.created` event in the same transaction as each successful `custom_data` insert.
- `DataService.update` records one `fivepanel.custom_data.updated` event in the same transaction as each successful changed update and records no event when it returns `changed: false`.
- `DataService.delete` records one `fivepanel.custom_data.deleted` event in the same transaction as each successful delete and includes the deleted row's prior data in `data.before`.
- Generated GraphQL create, update, and delete mutations still work without callers passing audit context, and each such direct mutation receives a generated `operation_id`.
- A single file import generates one `operation_id` and all audit events for that file import share it.
- Import-created `custom_data` rows have audit events with `source_type = 'import'`, `source_id = files.id`, `source_fingerprint = files.checksum`, and the mapped `source_provider`.
- Import-updated `custom_data` rows have audit events with changed-field `before` and `after` payloads and import row metadata.
- Skipped import rows, failed import rows, and unchanged matched rows do not create audit events.
- Scheduled integration jobs pass the active `job_execution_id` through to audit events produced by file imports in that job.
- One scheduled job execution importing multiple files creates separate `operation_id` values per file while retaining the same `job_execution_id`.
- Manual file imports and reprocesses create import audit events with `job_execution_id = NULL`.
- Cleanup code can select rows created by a bad import using either `operation_id`, `source_id`, or `source_fingerprint` without relying on broad campo/date filters.
- Cleanup deletions create new `delete` audit events with their own cleanup `operation_id`; the original `create` audit events remain present.
- Existing `custom_data` rows are not given synthetic create audit events unless a separate tenant-specific backfill can prove provenance. The default migration only enables full provenance for future mutations.
- Audit metadata never stores raw file contents, integration credentials, access tokens, or full shared links.

## Implementation Notes

Add the migration after the current latest migration in `backend/src/db/migrations`. Keep the table name `audit_events` and the Kysely table key `auditEvents`, matching the existing camel-case plugin conventions.

The append-only trigger can be implemented as a small PostgreSQL function:

```sql
CREATE FUNCTION prevent_audit_events_mutation()
RETURNS trigger AS $$
BEGIN
  RAISE EXCEPTION 'audit_events is append-only';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER audit_events_no_update_delete
BEFORE UPDATE OR DELETE ON audit_events
FOR EACH ROW EXECUTE FUNCTION prevent_audit_events_mutation();
```

The `down` migration should drop `audit_events_no_update_delete`, then `prevent_audit_events_mutation`, then the indexes/table.

Create explicit TypeScript types for audit context rather than passing unstructured metadata through every layer. A practical shape is:

```ts
type AuditMutationContext = {
  operationId?: string;
  jobExecutionId?: string | null;
  sourceType?: 'manual' | 'import' | 'script' | 'api' | 'system';
  sourceProvider?: string | null;
  sourceId?: string | null;
  sourcePath?: string | null;
  sourceFingerprint?: string | null;
  actorType?: 'user' | 'system' | 'script' | 'api_key' | 'integration' | 'worker';
  actorId?: string | null;
  userId?: string | null;
  occurredAt?: Date;
  requestId?: string | null;
  traceId?: string | null;
  data?: Record<string, unknown>;
  metadata?: Record<string, unknown>;
};
```

`DataService` should normalize missing audit context for direct mutations:

- `operationId`: generate a UUID for that one mutation
- `sourceType`: `manual`
- `actorType`: `user`
- `actorId`: current user id
- `userId`: current user id
- `occurredAt`: the mutation timestamp

For imports, build an import operation context in `importSingleFile` after loading the `FileRecord` and before calling `ImportOrchestrator.import`. Pass the full `FileRecord`, generated `operationId`, optional `jobExecutionId`, source mapping, and actor context into the orchestrator. The orchestrator should add row-specific metadata before each `DataService` mutation.

For update events, compute the diff before writing and insert the audit event after the repository update returns. The `custom_data` update and audit insert must share a Kysely transaction. If audit insertion fails, the data mutation should roll back. The same transaction rule applies to create and delete.

For imports, `findOrCreateRecord` should return enough information to capture the lookup path:

- which match column was tried
- which field/value matched an existing row
- whether the final result was created, updated, or unchanged

The existing `created_by` and `updated_by` fields should continue to use the current mutation user. Audit actor fields can represent the initiating user or execution principal separately. For scheduled imports, use `actor_type = 'worker'`, `actor_id = job_execution_id`, and include the system user id used for `created_by`/`updated_by` in `metadata.execution_user_id`. For user-triggered imports, use `actor_type = 'user'`, `actor_id = req.user.id`, and `user_id = req.user.id`, even if the import uses a system user internally.

`JobExecutor.executeJob` currently creates a `job_executions` row before loading the provider job script. Extend the job script context with `jobExecutionId: execution.id`. The Dropbox and Google Drive job scripts should pass that id into `importFiles`. The `/integrations/import`, `/integrations/run`, `/integrations/import/:fileId`, and `/files/:fileId/reprocess` routes should pass the authenticated request user as the audit actor where available.

Do not add a blanket GIN index on `data` or `metadata` in the initial migration. The top-level provenance columns and targeted btree indexes cover the cleanup and debugging workflows in this spec. Add JSONB indexes later only when a real query path requires them.

Cleanup examples:

Find rows created by one bad file import:

```sql
SELECT ae.entity_id AS custom_data_id, ae.model_id, ae.operation_id, ae.occurred_at
FROM audit_events ae
WHERE ae.tenant_id = $1
  AND ae.entity_type = 'custom_data'
  AND ae.operation = 'create'
  AND ae.source_type = 'import'
  AND ae.source_id = $2;
```

Find rows created by any import of the same file content:

```sql
SELECT ae.entity_id AS custom_data_id, ae.model_id, ae.source_id, ae.operation_id
FROM audit_events ae
WHERE ae.tenant_id = $1
  AND ae.entity_type = 'custom_data'
  AND ae.operation = 'create'
  AND ae.source_type = 'import'
  AND ae.source_provider = 'dropbox'
  AND ae.source_fingerprint = $2;
```

Find the whole history for a row before deciding whether it is safe to delete:

```sql
SELECT ae.operation, ae.operation_id, ae.source_type, ae.source_provider,
       ae.source_id, ae.occurred_at, ae.data, ae.metadata
FROM audit_events ae
WHERE ae.tenant_id = $1
  AND ae.entity_type = 'custom_data'
  AND ae.entity_id = $2
ORDER BY ae.occurred_at ASC, ae.id ASC;
```

Cleanup scripts that delete rows selected from audit events should create a fresh cleanup `operation_id`, delete through `DataService.delete` when relationship validation is desired, or insert delete audit events in the same transaction when direct SQL deletion is required for bulk cleanup. Those delete events should use `source_type = 'script'`, `source_id = '<script file name>'`, and metadata linking back to the bad import `operation_id` or source file.

## Test Expectations

- Migration tests or focused database checks verify table creation, check constraints, indexes, and append-only trigger behavior.
- Repository/service tests verify that audit events can be inserted and read, and cannot be updated or deleted through exposed repository methods.
- `DataService` tests cover create, changed update, unchanged update, and delete event creation, including transaction rollback when audit insertion fails.
- Import orchestrator tests cover one operation id per file import, row metadata, source mapping from `files.metadata.source`, file checksum propagation, and no events for skipped, failed, or unchanged rows.
- Scheduled job tests cover passing `jobExecutionId` from `JobExecutor` into provider job scripts and then into import audit events.
- Cleanup-query tests or SQL fixtures demonstrate selecting `custom_data` ids by `operation_id`, by `source_id`, and by `source_fingerprint`.
- Existing backend build and focused backend tests pass with `npm --prefix backend run build` and `npm --prefix backend run test` or the narrower test commands used by the implementation branch.

## Risks

- Audit rows may contain business data snapshots from `custom_data.data`; access to future audit read surfaces must follow tenant scoping and avoid exposing sensitive fields casually.
- Recording audit events synchronously increases write volume during large imports. The first implementation should keep payloads compact for updates and avoid broad JSONB indexes.
- If audit inserts are not in the same transaction as `custom_data` mutations, cleanup and debugging can still have gaps. Transactional writes are required for trust in this table.
- `job_execution_id` is stored without an initial foreign key because current job execution retention deletes old rows. This preserves the id value but does not guarantee a join target forever.
- Existing legacy rows remain partially untraceable unless a separate tenant-specific backfill can prove their source. The new table improves future provenance and future cleanup precision.

## Open Questions

None.
