# Issue #250: Be able to see the executions from the individual integration

Source issue: https://github.com/benitogonzalezh/five-panel/issues/250

## Problem

The Dropbox integration page includes its own Scheduled Jobs table, but that table does not show when each job last ran and does not provide direct access to that job's execution history. Users must leave the Dropbox integration page and find the same job on another screen to understand whether it ran, whether it succeeded, and what happened during the run.

## Current Behavior

The Scheduled Jobs table in `DropboxIntegrationView.vue` lists each Dropbox job's name, schedule, enabled status, next run, and existing actions for running, enabling or disabling, editing, and deleting the job.

The scheduled-job response already includes `lastRunAt`, and the frontend service already supports fetching a single job's executions through `GET /scheduled-jobs/:id/executions`. The main Scheduled Jobs page already uses these fields and endpoints to show a Last Run column and a job-specific execution history dialog. The Dropbox table does not render or expose those existing features.

## Desired Behavior

The Dropbox integration page shows the last run for every scheduled job and provides a history action on every job row. Selecting the history action opens an in-page dialog for that job only, without taking the user away from the Dropbox integration page.

The history dialog follows the existing Scheduled Jobs behavior. It shows the selected job's execution status, start time, completion time, duration, and details. Execution details include available error and result data. When an execution has related Dropbox files, the user can open the files view filtered to that execution.

The history action is read-only and remains available when the scheduled job or the Dropbox integration is disabled.

## Scope

- Add a Last Run column after Next Run in the Dropbox Scheduled Jobs table.
- Display `lastRunAt` using the existing localized date-time formatter.
- Display the existing localized Never value when `lastRunAt` is empty.
- Add a localized, keyboard-accessible history action to every Dropbox scheduled-job row, placed after Run Now.
- Load up to 50 executions for the selected scheduled job through the existing job-specific endpoint.
- Show the job-specific execution history, execution details, and supported execution-file action in dialogs on the Dropbox integration page.
- Provide localized loading, empty, and error behavior without showing results from a previously selected job.
- Add focused frontend tests for the new Dropbox table behavior.

## Out Of Scope

- Changing the Google Drive Scheduled Jobs table.
- Redesigning the main Scheduled Jobs page or the global Job Executions page.
- Changing scheduled-job or job-execution database tables, API response shapes, endpoints, or tenant access rules.
- Changing execution retention limits.
- Adding live polling or automatic refresh behavior for the Last Run value.
- Changing how jobs are created, edited, run, enabled, disabled, or deleted.
- A broad refactor of integration or scheduled-job screens.

## Acceptance Criteria

1. The Scheduled Jobs table on the Dropbox integration page displays a localized Last Run column after Next Run and before Actions.
2. A Dropbox scheduled job with a `lastRunAt` value displays that value using the existing localized date-time format.
3. A Dropbox scheduled job without a `lastRunAt` value displays the existing localized Never text.
4. Every Dropbox scheduled-job row displays a history icon after Run Now, with a localized tooltip and accessible name.
5. The history action remains enabled when the selected job is disabled or the Dropbox integration master switch is off.
6. Selecting a row's history action opens a dialog whose title identifies that scheduled job and requests executions using that job's ID, not the tenant-wide executions endpoint.
7. The dialog displays at most 50 executions for the selected job in newest-first order. Only the request for the currently selected job may update the execution rows, loading state, or error state, even when an earlier job's request finishes later.
8. Each history row displays a text-labeled status, start time, completion time when available, duration when available, and a details action.
9. The details view displays the execution status and start time, plus the completion time, error message, and result summary when those values are available.
10. When the existing execution-files route helper supports a Dropbox execution, its history row offers a localized View Files action that opens the integration files page with `source=dropbox` and the selected `jobExecutionId`.
11. While history is loading, the dialog displays a loading state. When no executions exist, it displays the existing localized empty-history message. When loading fails, it clears any previous job's results and displays the existing localized history error message.
12. Existing Run Now, enable or disable, edit, and delete actions on the Dropbox Scheduled Jobs table continue to behave as before.
13. The new column, action, dialog labels, empty state, and error state work in both supported frontend locales, English and Spanish.

## Implementation Notes

- Reuse the existing `ScheduledJob.lastRunAt`, `JobExecution` type, `fileService.getJobExecutions`, date-time formatting, status mapping, duration formatting, translations, and `buildExecutionFilesRoute` behavior already used by `ScheduledJobsView.vue`.
- Keep the request tenant-safe by using the existing `GET /scheduled-jobs/:id/executions?limit=50` endpoint, which checks that the job belongs to the signed-in user's tenant.
- Clear or hide the previous execution list before loading a newly selected job. Ignore or cancel stale responses so an earlier request cannot replace the current job's execution rows, loading state, or error state.
- A small shared execution-history component may be extracted if that keeps the Dropbox and main Scheduled Jobs screens consistent. Do not require a broad view refactor or change the visible behavior of the main Scheduled Jobs page.
- Keep the existing action ordering and increase the actions column width only as needed to fit the added history button.
- No backend or database work is expected because the required fields, endpoint, and tenant check already exist.

## Test Expectations

- Add focused frontend coverage showing that the Dropbox Scheduled Jobs table renders `lastRunAt`, the Never fallback, and the accessible history action.
- Cover selection of two different jobs and resolve their history requests out of order, including a stale failed request. Verify that each request uses the selected job's ID and that only the latest selection can update the dialog's rows, loading state, or error state.
- Cover completed, running, and failed execution display, including optional error and result details.
- Cover history access for a disabled job and when the Dropbox master switch is off.
- Cover the supported Dropbox execution-file action and verify that its route contains `source=dropbox` and the selected `jobExecutionId`.
- Run `npm --prefix frontend test` and `npm --prefix frontend run build`.
- In a browser, verify the Dropbox page with at least two scheduled jobs, including one job with no execution history, and confirm the English and Spanish labels.

## Risks

- The Dropbox and main Scheduled Jobs tables contain similar behavior and may drift apart. Reusing the existing behavior or a small shared component can reduce that risk without expanding this issue into a broad refactor.
- Execution history is loaded on demand. A slow or failed request must not leave a previous job's history visible under the newly selected job.

## Open Questions

None.
