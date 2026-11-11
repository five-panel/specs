# Issue #247: Short Implementation Guide

Read the [spec](247-report-datasets-silently.md) and [technical implementation plan](247-report-datasets-silently-implementation-plan.md). These describe proposed code, not an already implemented fix.

## How the pieces fit

The report service already checks access, validates filters, and prepares safe SQL. Keep those checks. Add a separate repository read that can return up to 25,000 complete rows. Leave the existing 1,000-row SQL filter-option operation alone.

The service rejects oversized results instead of silently trimming them. Successful results carry `complete: true`. The full response must fit within 10,485,760 UTF-8 bytes; the SQL statement keeps its five-second limit.

A new shared paging composable and control show 100 rows at a time in tables. The complete result stays in browser memory; clicking another page makes no request. Refreshes and relevant filter changes load again and reset pages.

The main summary TOTAL already comes from SQL over all matching records. Grouped tables add their own daily subtotals from the complete loaded detail. Build full groups before paging, so a group spanning pages keeps the same full-group subtotal.

PDF uses the existing separate snapshot and renders every row without table paging. Pending, failed, or older results block export. Failed reloads show their actual error and retry action even when older rows remain visible.

No database records, report definitions, files, or jobs are written by loading or paging. Page position is temporary and belongs to each widget.

## Concrete flow example

This is a small test fixture, not the full ALCI report. It uses page size **2 only inside the paging test** so every value fits here; production always uses 100. Sample view ID is `11111111-1111-4111-8111-111111111111` and dataset ID is `dailyBreakdown`.

The saved fixture definition is complete:

```json
{
  "version": 1,
  "filters": [
    { "id": "dia", "label": "Day", "inputType": "date" }
  ],
  "datasets": [
    {
      "id": "dailyBreakdown",
      "type": "sql",
      "sql": "SELECT entry, day, amount FROM sample_work_entries WHERE (:dia::date IS NULL OR day = :dia::date) ORDER BY entry",
      "useFilters": ["dia"],
      "columns": [
        { "id": "entry", "label": "Entry", "type": "number" },
        { "id": "day", "label": "Day", "type": "string" },
        { "id": "amount", "label": "Amount", "type": "number", "format": "integer" }
      ]
    }
  ],
  "widgets": [
    {
      "id": "daily-table",
      "type": "groupedTable",
      "config": {
        "dataset": "dailyBreakdown",
        "title": "Daily detail",
        "groupBy": {
          "field": "day",
          "summary": [{ "label": "Amount", "field": "amount", "format": "integer" }]
        }
      }
    }
  ],
  "layout": {
    "desktop": [{ "id": "daily-table", "x": 0, "y": 0, "w": 12, "h": 8 }]
  }
}
```

The fixture's accessible tenant model view `sample_work_entries` contains exactly these logical row values. Preparing a fixture is separate from loading the report:

```json
[
  { "entry": 1, "day": "2026-09-01", "amount": 100 },
  { "entry": 2, "day": "2026-09-01", "amount": 200 },
  { "entry": 3, "day": "2026-09-02", "amount": 50 }
]
```

1. Open the fixture report and clear Day. `reportService.runDataset` sends `POST /views/11111111-1111-4111-8111-111111111111/report/datasets/dailyBreakdown/run` with this entire body:

```json
{ "filters": { "dia": null } }
```

2. The service checks the saved view and company, allows only that company's model views, and binds both occurrences of `:dia` to null. The new repository operation runs the read-only query with a 25,001-row probe. Three rows fit; the transaction closes. The service validates the full response size and returns:

```json
{
  "datasetId": "dailyBreakdown",
  "columns": [
    { "id": "entry", "label": "Entry", "type": "number" },
    { "id": "day", "label": "Day", "type": "string" },
    { "id": "amount", "label": "Amount", "type": "number", "format": "integer" }
  ],
  "rows": [
    { "entry": 1, "day": "2026-09-01", "amount": 100 },
    { "entry": 2, "day": "2026-09-01", "amount": 200 },
    { "entry": 3, "day": "2026-09-02", "amount": 50 }
  ],
  "generatedAt": "2026-11-12T10:00:00.000Z",
  "complete": true
}
```

3. The runtime accepts the result for the current request and gives it a new revision. Full groups are September 1 with entries 1–2 and subtotal 300, and September 2 with entry 3 and subtotal 50. The two-row test page shows entries 1–2 and “1–2 of 3.” Clicking Next changes only local page state: entry 3 appears with “3–3 of 3.” No request occurs and the stored result is unchanged. In production this tiny fixture shows all three rows without page controls.
4. An administrator exports the saved report while viewing test page two. The handler checks current complete data and captures it synchronously. The export tree receives all three rows and renders both groups, with totals 300 and 50 and no controls. It creates no new database records.

The real 11,476-row case uses the same flow with 100 rows per page: 115 pages, with 76 rows on the last page. Its downloaded PDF must actually contain the full report; a safe error does not count as passing that case.
