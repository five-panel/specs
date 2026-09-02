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

Dropbox scheduled jobs can be associated with zero, one, or many Dropbox integration
configurations.

The canonical params shape is:

```ts
{
  integrationConfigIds?: string[]
}
```

`integrationConfigIds` may contain any enabled Dropbox config IDs that belong to the
job tenant. There are no grouping restrictions: configs that point to the same
Dropbox file may be selected in one job, split across different jobs, selected by
more than one job, or left out of a job.

If `integrationConfigIds` is missing, `null`, or an empty array, the job keeps the
legacy behavior and runs all enabled Dropbox configurations for the tenant. This
preserves existing Dropbox jobs that have `{}` params.

When `integrationConfigIds` is present and non-empty, a Dropbox job only syncs the
selected configs and only imports files queued or requeued by that same job
execution. It must not import old pending Dropbox files or pending files produced by
another scheduled job execution.

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
- Update the Dropbox job to pass its selected configs into Dropbox sync.
- Update Dropbox sync so it can run either all enabled configs or an explicit selected
  config list.
- Preserve the current optimization that downloads a Dropbox source at most once
  inside a single execution when multiple selected configs reference the same source.
- Add job/execution metadata to files created or requeued by scheduled Dropbox runs
  so import can select exactly that execution's files.
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
- A Dropbox scheduled job with `{}` params, missing `integrationConfigIds`, or an
  empty `integrationConfigIds` array continues to run all enabled Dropbox configs for
  the tenant.
- A Dropbox scheduled job with legacy `{ integration_config_id: "<id>" }` params runs
  that one config when `integrationConfigIds` is absent.
- The API rejects Dropbox job params when `integrationConfigIds` is not an array of
  strings.
- The API rejects selected config IDs that do not exist, belong to another tenant,
  are disabled, or are not Dropbox configs.
- Configs that reference the same Dropbox source can be selected in different jobs
  without validation errors.
- The same Dropbox config can be selected by more than one job without validation
  errors.
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

- Prefer a small Dropbox params parser such as `parseDropboxJobParams(params)` that
  returns one normalized shape:

  ```ts
  type DropboxJobParams = {
    integrationConfigIds?: string[];
  };
  ```

- Treat missing, `null`, or empty `integrationConfigIds` as "all enabled Dropbox
  configs" for backward compatibility.
- Treat legacy `integration_config_id` as a one-item selection only when
  `integrationConfigIds` is absent.
- Put tenant-safe config lookup and validation in a service layer rather than trusting
  the raw params object in the job script.
- The current `syncFilesFromFolder` helper already groups selected tasks by Dropbox
  source for download efficiency. Keep that internal grouping as an optimization only;
  do not use it as a validation rule.
- Add optional sync input fields for the selected configs and scheduled execution
  metadata, for example `integrationConfigs`, `scheduledJobId`, and `jobExecutionId`.
- When sync saves or requeues a file during a scheduled job execution, include metadata
  that can be queried later:

  ```ts
  {
    source: 'dropbox',
    statusImport: 'pending',
    scheduledJobId,
    jobExecutionId,
    integrationConfigId
  }
  ```

- `reprocessFileFromChecksum` and `reprocessFile` may need to accept extra metadata so
  requeued files keep the same execution tag as newly saved files.
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
  Dropbox schedules with either the selected IDs from the UI or `{}` when the user
  chooses the legacy all-configs behavior.

## Test Expectations

- Add backend unit tests for the Dropbox params parser, including `{}`, empty array,
  non-empty array, invalid non-array values, invalid non-string entries, and legacy
  `integration_config_id`.
- Add backend service or route tests showing that selected config IDs are tenant
  isolated, provider validated, and enabled-state validated.
- Add Dropbox sync tests showing that explicit config selections filter downloaded
  sources and still download a shared source only once within the same execution.
- Add file import tests showing scheduled-job imports only read pending files tagged
  with the current `jobExecutionId`.
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

- If queued files are not tagged consistently for both new saves and requeues, a
  scheduled job could still import another job's pending work.
- Legacy pending Dropbox files without `jobExecutionId` must remain importable through
  provider-level manual import actions, but must not be drained by a selected
  scheduled job.
- Allowing the same config or Dropbox source in multiple jobs can cause duplicate work
  when schedules overlap. This is allowed by design for this issue.
- Existing UI text assumes one Dropbox schedule in a few places and may need copy
  updates in both English and Spanish locale files.

## Open Questions

None.
