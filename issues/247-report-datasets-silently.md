# Issue #247: Report datasets silently truncate after 1,000 rows

Source issue: https://github.com/benitogonzalezh/five-panel/issues/247

## TL;DR

Remove the silent 1,000-row cutoff. Load each complete query result within clear safety limits, then show tables in pages of 100 rows. Totals, charts, summaries, and PDF use the complete loaded data.

Companion documents: [Technical implementation plan](247-report-datasets-silently-implementation-plan.md) · [Short implementation guide](247-report-datasets-silently-implementation.md).

## What will change

**Medium code impact:** extend dataset loading, add shared table page controls, and strengthen error handling and PDF checks. Reuse existing report services, filters, permissions, protected SQL execution, and the PDF snapshot flow.

The main summary TOTAL is already calculated in SQL across matching records. Daily group subtotals are calculated by the table from its received detail rows. Returning complete detail fixes those subtotals without changing the report's SQL formulas.

## How it would work

1. Opening a report, refreshing it, or changing relevant filters loads each affected dataset once. Accept complete results up to **25,000 rows and 10 MiB of uncompressed response JSON per dataset**, retaining the existing five-second SQL statement limit. Exceeding either size limit returns a clear error asking users to narrow filters; no partial result is presented as success. Preserve deliberate limits written into a report's own SQL.
2. Each table shows **100 result rows per page**, its current range, and the complete result count. Hide page controls for 100 or fewer rows. Page buttons use already loaded data and make no further requests. Each widget keeps its own page position; changed results, relevant filters, views, or definitions reset it to page one.
3. Grouped tables build complete groups and their full subtotals before dividing rows into pages. A continuing group repeats its heading and clearly identifies its totals as covering the whole group. Group headings do not consume the 100-row allowance. An SQL total row counts as one result row and receives total styling only at its actual final position.
4. PDF includes all loaded rows regardless of the visible table page. Keep existing export permissions and saved-report restrictions. Export requires current, successful, complete results for every dataset used by the exported widgets, including when filters or report settings are changing.
5. Failed loads show the actual error and a retry action. Retained older rows are visibly marked out of date and cannot authorize PDF export. Narrowing filters and retrying can restore a complete report.

## Done when

- The reported 11,476 detail rows are reachable across 115 pages without missing or repeated rows; page changes make no data requests.
- SQL summary totals remain correct, full-group subtotals stay unchanged across pages, and ordinary page-ending rows never receive SQL-total styling.
- A real PDF exported from the first, middle, or last live page successfully includes all 11,476 detail rows, final details, and complete totals, with readable content and no table page controls.
- Filters, refreshes, and report changes reset relevant pages. Pending, failed, or superseded loads cannot mix older rows with new filter labels in PDF; a successful retry restores export.
- Exactly 25,000 rows and exactly 10,485,760 response bytes are accepted; exceeding either cap fails clearly without partial rows. Query failures remain clear and recoverable.
- Existing small reports, company separation, access checks, admin-only draft previews, and SQL filter options keep working. Page controls and errors work in English and Spanish, on narrow screens and in both themes.

## Out of scope

- Unlimited results, database requests on each page click, server-held paging sessions, and saved page settings. The caps apply separately to each dataset; they do not guarantee peak memory use, total HTTP duration, or one database snapshot across the whole report.
- New export formats, file storage, background jobs, migrations, and changes to saved report definitions or historical tenant scripts.
- Unrelated SQL formula or layout changes. Focused PDF fixes required to pass the known 11,476-row case remain in scope.
