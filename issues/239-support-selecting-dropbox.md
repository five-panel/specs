# Issue #239: Support selecting Dropbox integration configurations per scheduled job

Source issue: https://github.com/benitogonzalezh/five-panel/issues/239

## Problem

Dropbox scheduled jobs cannot choose which Dropbox integration configurations they run.
`scheduled_jobs.params` can store JSON, but the Dropbox job ignores it. Every Dropbox
job syncs every enabled Dropbox configuration for the tenant, then imports every
pending Dropbox file for the tenant.

This prevents tenants from running separate Dropbox jobs with different schedules and
different selected configurations. A fast Cosecha job and a slower Listadura job still
process the same enabled configurations today.

## Current Behavior

- `backend/src/jobs/dropbox-integration.ts` accepts `params`, but does not read them.
- `backend/src/integrations/dropbox/syncFilesFromFolder.ts` loads all enabled Dropbox
  configs through `integrationConfigService.find({ filter: { type: 'dropbox', enabled: true } })`.
- `backend/src/integrations/dropbox/filesImporter.ts` imports all pending files for
  the tenant and provider through `fileService.findFilesToImport({ tenantId, source })`.
- `backend/src/services/JobExecutor.ts` only prevents two executions of the same
  scheduled job from running at the same time. It does not treat Dropbox sources or
  integration configs as globally exclusive resources.
- `frontend/src/views/DropboxIntegrationView.vue` only allows one Dropbox schedule
  from the Dropbox settings view.
- `frontend/src/views/ScheduledJobsView.vue` can edit basic schedule fields, but has
  no Dropbox configuration selector.

## Desired Behavior

Dropbox scheduled jobs can run all enabled Dropbox configurations or can be
associated with one or many explicit Dropbox integration configurations.

The accepted params shape is:

```ts
{
  integrationConfigIds?: string[] | null
}
```

`integrationConfigIds` may contain any enabled Dropbox config IDs that belong to the
job tenant. There are no grouping restrictions: configs that point to the same
Dropbox file may be selected in one job, split across different jobs, selected by
more than one job, or left out of a job.

If `integrationConfigIds` is missing or `null`, the job keeps the legacy behavior
and runs all enabled Dropbox configurations for the tenant. This preserves existing
Dropbox jobs that have `{}` params. An empty `integrationConfigIds` array is not a
valid saved selection and must not be treated as "all configs".

When `integrationConfigIds` is present and non-empty, a Dropbox job only syncs the
selected configs and only imports files queued or requeued by that same job
execution. It must not import old pending Dropbox files or pending files produced by
another scheduled job execution.

Each scheduled Dropbox execution should use this flow:

1. Read the job params and resolve the config selection.
2. Load the selected tenant-owned Dropbox integration configs, or all enabled
   Dropbox configs for legacy all-config jobs.
3. Download the Dropbox files needed by those configs.
4. Convert each selected config's sheet into one CSV import file.
5. Create a new pending `files` row for each CSV import file, tagged with the
   current `scheduledJobId`, `jobExecutionId`, and `integrationConfigId`.
6. Import only the pending `files` rows tagged with the current `jobExecutionId`.

Legacy singular params from the earlier scheduling migration should be accepted at
execution time when the new array is absent:

```ts
{
  integration_config_id: string
}
```

This value should be treated as `integrationConfigIds: [integration_config_id]`.
New writes should use only `integrationConfigIds`.

## Scope

- Add typed Dropbox scheduled-job params validation for create and update.
- Resolve selected Dropbox configs using the authenticated tenant, not a tenant ID
  supplied by the client.
- Require selected configs to exist, belong to the tenant, have `type === 'dropbox'`,
  and be enabled.
- Reject empty explicit `integrationConfigIds` arrays for create and update. The UI
  must use missing or `null` params for the all-config mode, and require at least one
  config for the selected-config mode.
- Update the Dropbox job to pass its selected configs into Dropbox sync.
- Update Dropbox sync so it can run either all enabled configs or an explicit selected
  config list.
- Preserve the current optimization that downloads a Dropbox source at most once
  inside a single execution when multiple selected configs reference the same source.
- Add job/execution metadata to new per-execution pending `files` rows created or
  requeued by scheduled Dropbox runs so import can select exactly that execution's
  files.
- Do not update an existing original or shared `files` row in place to claim ownership
  for a scheduled job execution.
- Update import filtering so scheduled Dropbox jobs import only files queued or
  requeued by the same execution.
- Keep provider-level manual actions, such as "sync", "import", and "sync and
  import" from the integration settings views, able to run all enabled Dropbox
  configs and all pending Dropbox files unless those actions are explicitly updated
  later.
- Add a multi-select UI for Dropbox configurations in scheduled-job create and edit
  flows.
- Remove the one-Dropbox-schedule limit from the Dropbox settings view.
- Show selected config information in execution result summaries.

## Out Of Scope

- Adding restrictions that force configs from the same Dropbox source into the same job.
- Adding locks that block two different jobs from using configs that point to the same
  Dropbox file.
- Preventing the same Dropbox config from being selected by more than one job.
- Implementing the Listadura no-UUID atomic replacement strategy.
- Changing Dropbox credentials, source-path semantics, shared-link semantics, or
  workbook parsing.
- Automatically choosing production job intervals or assigning final ALCI config
  groups.
- Changing Google Drive scheduled jobs.

## Acceptance Criteria

- A user can create at least two enabled Dropbox scheduled jobs with different cron
  schedules and different `integrationConfigIds` selections.
- A Dropbox scheduled job with a non-empty `integrationConfigIds` list only downloads
  Dropbox sources needed by the selected configs.
- Running one Dropbox scheduled job does not import pending Dropbox files that were
  created or requeued by another scheduled job execution.
- Running one Dropbox scheduled job does not import pre-existing pending Dropbox
  files that are not tagged for that execution.
- A Dropbox scheduled job with `{}` params, missing `integrationConfigIds`, or
  `integrationConfigIds: null` continues to run all enabled Dropbox configs for the
  tenant.
- A Dropbox scheduled job with legacy `{ integration_config_id: "<id>" }` params runs
  that one config when `integrationConfigIds` is absent.
- The API rejects Dropbox job params when `integrationConfigIds` is not `null` and is
  not an array of strings.
- The API rejects `integrationConfigIds: []` for Dropbox scheduled-job create and
  update requests.
- The API rejects selected config IDs that do not exist, belong to another tenant,
  are disabled, or are not Dropbox configs.
- Configs that reference the same Dropbox source can be selected in different jobs
  without validation errors.
- The same Dropbox config can be selected by more than one job without validation
  errors.
- Each scheduled Dropbox execution creates pending `files` rows tagged with that
  execution's `jobExecutionId`; it does not retag an existing original or shared file
  row to point at the current execution.
- If two scheduled Dropbox executions select the same config or Dropbox source, each
  execution can import its own pending file rows without one execution overwriting or
  hiding the other execution's queued work.
- Within one job execution, a Dropbox source is downloaded at most once even when
  multiple selected configs reference it.
- Job execution result summaries include selected config IDs, processed config IDs,
  skipped config IDs when any are skipped, source labels, file counts, import counts,
  and errors.
- The Dropbox settings UI no longer blocks creation when another Dropbox scheduled
  job already exists.
- The scheduled jobs UI lets users select multiple enabled Dropbox configs for a
  Dropbox job and saves those selections as `params.integrationConfigIds`.
- The scheduled jobs UI preserves existing custom cron values when editing a job.

## Implementation Notes

- Keep this change small and simple. Reuse scheduled-job params, job executions, and
  per-execution pending `files` rows before adding new tables, locks, grouping rules,
  or broad refactors.
- Prefer a small Dropbox params parser such as `parseDropboxJobParams(params)` that
  validates the actual params shape:

  ```ts
  type DropboxJobParams = {
    integrationConfigIds?: string[] | null;
  };
  ```

- Treat missing or `null` `integrationConfigIds` as "all enabled Dropbox configs" for
  backward compatibility.
- Do not treat `integrationConfigIds: []` as "all enabled Dropbox configs". Reject it
  so clearing the multi-select cannot accidentally make a job run every Dropbox config.
- Treat legacy `integration_config_id` as a one-item selection only when
  `integrationConfigIds` is absent.
- Put tenant-safe config lookup and validation in a service layer rather than trusting
  the raw params object in the job script.
- The current `syncFilesFromFolder` helper already groups selected tasks by Dropbox
  source for download efficiency. Keep that internal grouping as an optimization only;
  do not use it as a validation rule.
- Add optional sync input fields for the selected configs and scheduled execution
  metadata, for example `integrationConfigs`, `scheduledJobId`, and `jobExecutionId`.
- When scheduled sync saves or requeues a file, create a new per-execution pending
  `files` row and include metadata that can be queried later:

  ```ts
  {
    toImport: true,
    source: 'dropbox',
    statusImport: 'pending',
    scheduledJobId,
    jobExecutionId,
    integrationConfigId
  }
  ```

- For content that already exists, prefer the existing `reprocessFileFromChecksum` and
  `reprocessFile` pattern, which creates a new `files` row pointing at the same stored
  file through `metadata.reprocessedFrom`.
- `reprocessFileFromChecksum` and `reprocessFile` may need to accept extra metadata so
  requeued files get the same execution tag as newly saved files.
- Avoid adding a separate import queue table for this issue unless the existing
  per-execution `files` row model cannot satisfy the acceptance criteria.
- Add repository or service methods that can find pending files by tenant, provider,
  and `jobExecutionId`. Scheduled job imports should use that query. Provider-level
  manual imports should keep using the existing provider-wide pending-file query.
- Pass `jobExecutionId` into the job script context from `JobExecutor` after the
  execution row is created. Keep the existing per-job running guard.
- Update `frontend/src/services/FileService.ts` types so `ScheduledJob.params` can
  expose `integrationConfigIds`.
- In `ScheduledJobsView.vue`, show the config multi-select only for Dropbox jobs.
  The options should come from enabled Dropbox integration configs for the current
  tenant.
- In `DropboxIntegrationView.vue`, remove the one-schedule guard and create new
  Dropbox schedules with either a non-empty selected ID list from the UI or `{}` /
  `integrationConfigIds: null` when the user chooses the legacy all-configs behavior.

## Test Expectations

- Add backend unit tests for the Dropbox params parser, including `{}`, `null`,
  empty array rejection, non-empty array, invalid non-array values, invalid non-string
  entries, and legacy `integration_config_id`.
- Add backend service or route tests showing that selected config IDs are tenant
  isolated, provider validated, and enabled-state validated.
- Add backend route tests showing that `integrationConfigIds: null` is accepted for
  all-config mode and `integrationConfigIds: []` is rejected.
- Add Dropbox sync tests showing that explicit config selections filter downloaded
  sources and still download a shared source only once within the same execution.
- Add Dropbox sync tests showing scheduled runs create per-execution pending `files`
  rows with `scheduledJobId`, `jobExecutionId`, and `integrationConfigId`.
- Add file import tests showing scheduled-job imports only read pending `files` rows
  tagged with the current `jobExecutionId`.
- Add overlap tests showing two executions for the same config or source keep separate
  pending `files` rows and do not overwrite each other's execution tags.
- Add job executor tests showing `jobExecutionId` is available to the Dropbox job
  script context.
- Add regression tests showing manual provider-wide imports still import pending
  Dropbox files across the provider.
- Add frontend tests for saving multiple selected Dropbox configs on a scheduled job.
- Add frontend tests showing Dropbox settings can create more than one Dropbox
  schedule.
- Run focused verification:
  - `npm --prefix backend run test`
  - `npm --prefix backend run build`
  - `npm --prefix frontend run test`
  - `npm --prefix frontend run build`

## Risks

- If scheduled sync mutates an existing original or shared file row instead of creating
  a per-execution pending row, overlapping jobs could overwrite each other's execution
  tags.
- Legacy pending Dropbox files without `jobExecutionId` must remain importable through
  provider-level manual import actions, but must not be drained by a selected
  scheduled job.
- Allowing the same config or Dropbox source in multiple jobs can cause duplicate work
  when schedules overlap. This is allowed by design for this issue.
- Rejecting empty selected-config lists means the UI needs a clear all-config mode so
  users do not confuse "no selected configs" with "all configs".
- Existing UI text assumes one Dropbox schedule in a few places and may need copy
  updates in both English and Spanish locale files.

## Open Questions

None.
