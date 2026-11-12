# Issue #247: Complete report datasets and table pagination — Implementation Plan

> **For agentic workers:** Use `superpowers:executing-plans` to implement these tasks in order. This document plans future application changes; the spec PR contains documents only.

**Goal:** return complete bounded report results, browse them in local table pages, and export their full contents safely to PDF.

**Architecture:** keep the existing dataset route and service validation. Add a separate bounded repository read for datasets, preserving the SQL filter-option operation. Share local pagination between table widgets; keep complete rows in the runtime and PDF snapshot.

**Tech stack:** TypeScript, Fastify, Kysely/PostgreSQL, Vue composables, existing PrimeVue paginator, and the current native PDF exporter.

**Spec:** [Agreed behavior](247-report-datasets-silently.md). [Short implementation guide and complete example](247-report-datasets-silently-implementation.md).

## Global constraints

- Fixed production page size: 100 actual result rows per widget; page clicks perform zero HTTP/SQL work.
- Accept at most 25,000 rows and 10,485,760 bytes of full uncompressed UTF-8 response JSON per dataset. Read a 25,001st row only to detect overflow. Keep the 5,000 ms SQL statement timeout.
- Success always means the complete authored query result for the supplied filters. Preserve authored SQL ordering and deliberate `LIMIT` clauses. No success containing trimmed rows.
- Keep service validation/access checks and repository-owned read-only transactions, local role, parameter binding, and tenant-specific view allowlist. No migration, dependency, saved definition, storage, history, or worker changes.
- PDF requires current, complete results and must actually succeed on the known 11,476-row fixture. Clear failure alone does not satisfy that case.

## Review focus

1. Exact 1,000 versus 1,001 rows, and exact 25,000 versus 25,001: Task 1 must prove complete success or rejection.
2. Groups crossing a page boundary or interleaved in SQL order: Task 3 must preserve the existing grouped display order and full-group totals.
3. Filter edits before route synchronization, delayed text input, and another report reusing dataset IDs: Task 2 must invalidate export readiness before asynchronous work.
4. PDF started from a later live page: Task 4 must bypass slicing and verify the downloaded large artifact.
5. A larger reload failing while older rows remain: Task 2 must expose the actual failure/retry and block PDF until recovery.

## Existing behavior and affected files

Reviewed report code at application main `c3142a20994974b979a37f7fbbcb27e3b3c9ef7a`; report files match the research snapshot `ebccffa359d21ccd7687fc5b2cc3d691ede3919a`. The repository currently wraps SQL in an outer limit of at most 1,000 and returns an array. The service returns rows without a completeness guarantee. Both tables render every received row; grouped summaries sum those rows. PDF snapshots current widget states but does not currently prove that all required results are current and successful.

| Files | Responsibility |
| --- | --- |
| `backend/src/repositories/ReportSqlRepository.ts` | Add the dataset-only bounded read; preserve `execute()` for filter options. |
| `backend/src/services/report/ReportDatasetService.ts`, `ReportTypes.ts` | Complete-result contract, row/byte rejection, typed limit error. |
| `backend/src/index.ts`, `backend/src/locales/{en,es}/rest.json` | Thin mapping of the new error to a localized HTTP response. |
| `frontend/src/types/report.ts`, `frontend/src/components/report/reportRuntimeUtils.ts`, `frontend/src/composables/useReportDatasets.ts` | Accept complete results; preserve request ownership and result revisions. |
| `frontend/src/views/ReportView.vue`, report `ReportCanvas.vue`, `ReportBlockRenderer.vue`, `blocks/ReportFilterBlock.vue` | Invalidate before filter/view changes, pass pending-input state and page-reset revisions, strengthen PDF readiness. |
| New `frontend/src/composables/useReportTablePagination.ts`, new `frontend/src/components/report/ReportTablePagination.vue` | Local page ranges, reset/clamp behavior, shared controls. |
| Report `blocks/ReportTableBlock.vue`, `blocks/ReportGroupedTableBlock.vue`, `ReportBlockStatePanel.vue` | Slice live rows correctly and show useful reload failures. |
| Report `ReportExportDocument.vue`, `reportPdfExport.ts`, `reportPdfTable.ts` | Keep full export snapshots and apply focused rendering fixes if the large artifact requires them. |
| `frontend/src/locales/{en-US,es-ES}/report.json` | Range, group-continuation/full-group labels, readiness and recovery messages. |

Paths abbreviated as “report” above are under `frontend/src/components/report/`. Verify current locations before editing; do not refactor unrelated code.

## Task 1: Complete bounded dataset reads

**Interfaces**

- Preserve `execute({ text, values, rowLimit?, timeoutMs? }): Promise<Record<string, unknown>[]>`, including its existing 1,000 maximum/default for SQL filter options.
- Add repository `findDatasetRows({ text, values }): Promise<Record<string, unknown>[]>`. It uses a fixed outer `LIMIT 25001` and a fixed 5,000 ms statement timeout; no caller limit/timeout parameter. Reuse the existing controlled read-only transaction setup through direct internal functions. Close the transaction before returning rows to the service.
- Keep `useReportDatasetService().run(...)` inputs unchanged. Add mandatory `complete: true` to both backend and frontend `ReportDatasetResult`; keep backend `generatedAt: Date` and wire/frontend ISO string. No total-count or page fields; `rows.length` is the exact accepted result count.
- Add `ReportDatasetLimitError` with `reason: 'rows' | 'bytes'`. Preserve it through the service's catch before generic runtime-error wrapping.

The existing `POST /views/:id/report/datasets/:datasetId/run` accepts `{ filters, reportDefinition? }`; optional definitions remain admin-only drafts. No new endpoint or execution controls. Client-supplied SQL, tenant, row-limit, page, or timeout values cannot alter execution. Use trusted request identity exactly as today.

- [ ] Extend `backend/test/repositories/ReportSqlRepository.test.ts` and `backend/test/services/report/ReportDatasetService.test.ts`: results 0, 999, 1,000, 1,001, 11,476, and 25,000 succeed with all rows; 25,001 rejects with no result rows. An authored `LIMIT 3` remains complete. Existing option-query tests keep array/1,000 behavior. Run them before changing production code and confirm the new assertions fail.
- [ ] Implement the repository operation and switch only dataset execution to it, including the service's injected repository type. Preserve SQL safety checks, physical-view allowlisting, filter coercion, saved-view permissions, and admin draft validation before execution.
- [ ] Reject the probe row in the service, construct the complete result once, and measure `Buffer.byteLength(JSON.stringify(result), 'utf8')`. Include columns, metadata, date serialization, and escaping. Accept exactly 10,485,760; reject one byte more. Preserve PostgreSQL numeric strings and current JSON semantics. Serialization failures remain sanitized runtime errors. Prove that Fastify's actual response payload uses the same JSON representation and byte count; do not check only raw row cells or character length.
- [ ] Map the typed error to HTTP 422 in the thin route, using localized copy. Reuse the existing `ApiError` parsing; no new API transport layer. Public English response for row overflow:

```json
{
  "error": "This report result is too large. Narrow the filters and try again.",
  "code": "REPORT_DATASET_TOO_LARGE",
  "reason": "rows",
  "limits": { "rows": 25000, "bytes": 10485760 }
}
```

For byte overflow use `reason: "bytes"` with the same limits and localized message. Do not send rows or a successful response before all checks finish.

- [ ] Verify exact byte boundaries with multibyte text, quotes, nulls, dates, and metadata. Use real PostgreSQL to check success, statement timeout, prohibited writes, foreign-company physical view rejection, rollback, restored local role/timeout, and connection reuse. The report role can select generated views across tenants: the per-request allowlist is essential isolation.

## Task 2: Current results, errors, and page-reset signals

**Interfaces:** extend runtime state with `revision: number`, allocated from a monotonic runtime counter on invalidation/loading and accepted replacement. Add runtime `invalidateDatasets(datasetIds: string[]): void`; retain old rows under loading presentation, abort requests, and invalidate their tokens synchronously. Forward each state's revision as numeric `resetKey` to its table widget.

- [ ] Add meaningful cases in `frontend/test/reportRuntime.test.ts` and `frontend/test/reportServices.test.ts` before implementation: complete success, unknown/missing completeness rejected as a load error, late success/error ignored, and failed reload retaining visibly old data. Rows, columns, completeness, and revision must be accepted together. Error-state rows cannot satisfy PDF readiness.
- [ ] Extend `useReportDatasets` with invalidation of the affected IDs before route synchronization or filter-option/default initialization yields. Capture view, definition, and applicable filter values for each load. Invalidate and recreate results when the view/definition changes, even with identical dataset IDs; invalidate tokens before deleting old state. Keep selective `useFilters` reloads for unaffected datasets.
- [ ] Add a `pending-filter` event carrying `{ filterId: string, pending: boolean }` from the filter block through renderer/canvas to the view for the existing 350 ms text/number delay. Block PDF while pending, then apply through existing filter handlers. Gate initialization as well. Reset pages on applied relevant changes, not each keystroke.
- [ ] In `ReportBlockStatePanel`, show the actual error and a usable retry button even when old rows exist; preserve the out-of-date presentation. Surface a report-level failure state. Disable live pager actions while the containing result is loading or failed. Clear failure after a current complete retry succeeds.

Verification must cover a new filter before the router promise resolves, a pending text/number edit, option/default initialization, refresh failure/recovery, and navigation between views with the same dataset IDs. These checks protect the spec's explicit current-result/PDF requirement; do not add general request cancellation machinery. Browser abort still does not promise immediate database cancellation.

## Task 3: Local table pagination and full-group totals

**Interfaces:** `useReportTablePagination({ totalRows, resetKey, pageSize? })` consumes Vue refs and returns `page`, `pageCount`, `startIndex`, `endIndex`, `rangeStart`, `rangeEnd`, and direct `setPage(page: number)`/`reset()` functions. `page` is one-based; indexes form a zero-based, end-exclusive slice; empty range is 0–0. The default is 100. A private `pageSize` override is for small unit fixtures only; production always uses 100, with no saved setting or HTTP knob.

`ReportTablePagination.vue` accepts page, totalRows, and disabled state; emits `update:page`. Reuse the installed PrimeVue paginator with fixed 100 rows and localized range/navigation labels. Keep it outside the scrolling body so controls remain reachable. Hide it at 100 rows or fewer.

Both table blocks gain `resetKey: number`, `disabled?: boolean`, and `exportMode?: boolean` props. The renderer passes the runtime revision, `status !== 'ready'` as disabled, and the existing export mode; export rendering ignores live pager state.

- [ ] Create `frontend/test/reportTablePagination.test.ts`; extend `frontend/test/reportComponents.test.ts`. Prove 0/1/100/101/200/201 boundaries, last-page clamping, independent widget pages, and reset on runtime revision/result changes.
- [ ] Plain table: slice full rows only for live rendering; use original global indexes for row keys and last-row total styling. A 101-row result with a final SQL TOTAL has that row once on page two. Do not synthesize, pin, or repeat a total.
- [ ] Grouped table: build the existing complete Map groups and summaries first. Preserve first-seen group order and source order within each group. Slice a 100-detail-row window across that grouped order, rendering intersecting fragments with original group indexes and within-group offsets. Repeat the heading and full-group summaries, clearly labeled as full-group totals; mark continuing fragments. Headers/summaries do not consume page rows.
- [ ] Test interleaved input `A1, B1, A2` preserving display order `A1, A2, B1`. Test group A with 101 rows and B with 100: pages contain A1–100; A101 plus B1–99; B100. Concatenating pages must equal the prior full grouped display, without duplicates. Independently sum each full group and assert summaries stay identical on every fragment. Keep existing null-key and number-format behavior.

## Task 4: Full PDF and the real large case

- [ ] Forward existing `exportMode` to both table widgets. In that mode bypass slicing and page controls entirely; reuse full groups and summaries. Retain `ReportExportDocument`'s separate snapshot tree. Hiding controls with CSS alone does not restore sliced rows.
- [ ] Strengthen both the button gate and the `exportPdf` handler. Every dataset referenced by an exported table, chart, or dataset text widget must be ready, error-free, current, and `complete === true`. Unused definitions do not block export; static-only reports and complete empty results are valid. Keep admin-only saved-report export rules. Show the reason when unavailable.
- [ ] Preserve the synchronous deep snapshot before the first await. Extend `frontend/test/reportPdfExport.test.ts`, `reportPdfTable.test.ts`, and component tests: first/middle/last live pages produce identical full row content; pending/failed/unknown/stale results reject before artifact generation; changes after capture cannot alter the snapshot.
- [ ] Extend `e2e/test/reports.spec.ts` with a real paginated report and request counting. Page turns issue no dataset requests; filters reload affected data and reset pages. Verify first/last rows and totals in the downloaded PDF, not only mocked helpers or DOM counts.
- [ ] Run the restored ALCI case in a disposable Docker environment using the maintained ops flow and immutable `20260906_01_create_contratistas_report.ts`. The 11,476-row daily result must load, expose 115 pages, and export successfully from first, middle, and last pages. Inspect the actual artifact for final rows, group totals, legibility, clipping, and absence of controls. The existing exporter measures every cell and scales one tall page: use focused fixes or existing table continuation where necessary to pass, rather than declaring a clear error acceptable. Record observed load/export duration, response bytes, PDF size, and visible row count; no invented performance SLA.

Check keyboard navigation, narrow layouts, English/Spanish, light/dark themes, and representative repeated widgets. Live tables render no more than 100 detail rows each; charts and text still consume the full result. Existing deliberate chart category limits remain deliberate presentation choices.

## Verification and preserved areas

Run focused package tests after each task, then the affected builds and browser journey from the application repository:

```sh
npm --prefix backend test
npm --prefix frontend test
npm --prefix backend run build
npm --prefix frontend run build
npm --prefix e2e test -- reports.spec.ts
```

Use Docker Compose for the application; do not start host dev servers. Run real-SQL checks in an isolated test database, not the user's working database. Ensure tests are executing rather than silently skipping integration prerequisites.

Saved version-1 reports, filter defaults/coercion/date conflicts, SQL filter options, hidden declared fields, chart/text rendering, read/share permissions, tenant allowlists, and existing retries remain covered. Verify ALCI's filtered reference slice from [#244](https://github.com/five-panel/five-panel/issues/244) / [#248](https://github.com/five-panel/five-panel/issues/248), including its 22 rows and established totals. Zipp's intentional SQL `LIMIT 3`/`LIMIT 5` must retain their meaning. No writes means no new automatic change-history events. Imports, cloud providers, backups, jobs, and stored uploads need no change. Historical PDF work [#258](https://github.com/five-panel/five-panel/issues/258) does not override the current exporter or this spec's full-output requirement.

One bounded load keeps page browsing consistent without unique SQL keys or retained server state. Independent `LIMIT/OFFSET` requests need predictable ordering and can observe later database changes; see [PostgreSQL's LIMIT documentation](https://www.postgresql.org/docs/17/queries-limit.html). This design does not create a single snapshot across separate dataset queries. Caps bound rows, statement time, and encoded response size; they do not bound peak server/browser memory, complete HTTP duration, or aggregate report memory.

## Research evidence versus implementation checks

The complete source-batch review covered 457 distinct application/docs/ops files collectively, plus the relevant report test sources and package/configuration files. This is not a claim that every test or every one of the 673 inventory files was manually read. Earlier baseline checks passed 88 focused backend and 130 focused frontend tests; they predate the proposed fix and do not prove it works.

Read-only execution of the authored SQL with cleared filters in the available local data returned 11,476 daily rows and 723 summary rows. The main summary TOTAL is calculated inside SQL; grouped daily subtotals are calculated in the table. The saved contractor report was absent from that local database, so an actual large API/browser/PDF flow remains an implementation check. No application code was changed by the spec work.
