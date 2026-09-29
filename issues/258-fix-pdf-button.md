# Issue #258: Fix PDF button from report

Source issue: https://github.com/benitogonzalezh/five-panel/issues/258

## Problem

The report PDF download can cut off report content. The download must contain the current report at a fixed A4 portrait width, continuing downward across as many pages as needed, including all loaded rows and report-displayed columns hidden by scrolling. The PDF is primarily for viewing on a computer. The agreed scope uses the same PDF layout on all devices; device-specific mobile and tablet layouts may follow later.

## Current Behavior

- `frontend/src/views/ReportView.vue` mounts a separate export copy and downloads it with `html2pdf.js`. It uses the saved report definition, current filters, and loaded dataset state.
- Export is available to superusers and disabled during editing, dataset loading, or an existing export. It provides progress, success, and error messages.
- `frontend/src/style.css` fixes the export shell and document at 1120px wide. `ReportCanvas.vue` chooses its mobile layout from the browser width, with the mobile breakpoint at 768px. These independent sizing decisions can produce an export unlike the visible report.
- Grid item content and table panels constrain content with fixed layout heights and hidden overflow. Export styles make table scrolling regions overflow visibly, but do not remove all ancestor constraints. They also request that entire tables avoid page breaks.
- Export waits for the grid's ready event. This alone does not establish that fonts and charts have finished rendering.
- Paper settings support A4 or Letter and portrait or landscape, defaulting to A4 portrait, with 10mm margins. The export uses a light appearance and a filename based on the report slug or name.

These are findings from code inspection, not a confirmed browser reproduction of the reported PDF. Existing export tests largely inspect source code and cannot establish that the downloaded pages contain the full report.

## Desired Behavior

Clicking PDF downloads a complete, readable copy of the current saved report and its applied filters. Include the report title, filter summary, report blocks, and every loaded table row and report-displayed column, regardless of the page's or a table's scroll position. Column coverage follows the saved report: preserve configured column selection, order, headings, and formatting. Fields intentionally omitted by report configuration remain omitted. For grouped tables, retain grouping labels and summaries without adding grouping fields as extra table columns.

Always generate standard A4 portrait pages, each 210mm wide and 297mm high, with 10mm margins and a 190mm usable content width. Content continues downward across as many pages as needed, suitable for continuous scrolling in a PDF viewer. Do not generate one unbounded-height page or impose a one-page limit. Use the same export layout regardless of the device or browser width used to start the download. Base report block arrangement on the saved desktop layout, fitted to the usable A4 width, preserving block order and grouping while allowing heights and page placement to change for complete content. Mobile layout overrides do not control this export.

This PDF action always uses A4 portrait, even when the saved print settings specify Letter or landscape. Preserve those saved settings without using them for this action; do not rewrite report definitions. Long tables continue onto additional pages with column headings repeated. First wrap cell text and reduce unnecessary spacing to fit table columns within the available width. Do not keep shrinking text to force every column onto one page. Only when a table still cannot fit readably, divide it into successive groups of columns on additional A4 portrait pages. When columns need additional pages, repeat row labels and any group labels needed to associate values with their rows. For tables divided into column groups, assign each row a stable export-only row number in report order and repeat it in every column group and any continuation of that row. Numbers must be unique within the report block, including across its groups, so blank or duplicate row labels and different page breaks cannot make row matching ambiguous. Preserve column and row order, and include totals without changing their meaning.

Keep rows and charts together when they fit on a page. A row taller than a page may continue across pages with all its text retained. Fit charts proportionally within the printable area without losing labels or data. Export must not omit content or shrink the entire report to one page.

Use one consistent set of report data, applied filters, and layout choices throughout export. Wait for the export copy's layout, fonts, and charts to be ready. Keep the live report's layout, filters, scroll position, and theme unchanged. On failure, show the existing error feedback, remove temporary export content, and allow another attempt.

## Scope

- Correct browser-side PDF sizing, complete content capture, and page breaks.
- Export all already loaded rows and report-displayed columns, including ordinary and grouped tables; preserve configured omissions.
- Use one fixed A4 portrait export layout on desktop, phone, and tablet browsers.
- Preserve existing permissions, edit/loading restrictions, filename, light export appearance, and status feedback. Override paper size and orientation for this PDF action without changing saved settings.
- Add focused regression coverage and inspect real downloaded PDFs.

## Out Of Scope

- Fetching additional records beyond the report's loaded results or changing dataset limits.
- Exporting unsaved report edits or changing report editing and saved layout behavior.
- New export controls, paper settings, or report-definition fields.
- Device-specific mobile/tablet PDF layouts, automatic viewport-based PDF layout selection, and an export-profile framework. These remain possible future work.
- Letter, landscape, custom-width, or single unbounded-height output from this PDF action.
- A server-side PDF service, database migrations, API changes, or unrelated report refactoring.
- New export formats or a requirement for selectable/searchable PDF text.

## Acceptance Criteria

1. A report with tables that scroll both vertically and horizontally exports every loaded row and report-displayed column, including the last row, rightmost displayed column, group labels, and totals. A grouped table with configured column selection retains that selection, order, and headings; omitted dataset fields do not appear as extra columns. Exporting after scrolling produces the same report content as exporting from the top-left position.
2. Exports of the same report state at 390px, 768px, 769px, and 1280px have the same A4 page dimensions, block arrangement, column grouping, and page breaks. Configured mobile layout overrides do not change the PDF. No block is missing, overlapped, or cropped at a page edge.
3. Long tables continue across pages with column headings on every continuation page. Rows that fit within a page are not cut in two; oversized rows retain all text across their continuation pages.
4. Wide tables retain all report-displayed columns. Cell text wraps, and tables that still do not fit continue in ordered column groups on additional pages with row and group labels repeated. Stable export-only row numbers repeat across column groups and row continuations, so each value can be matched to its row even when labels are blank or repeated and cell wrapping causes different page breaks.
5. The title, filter summary, formatted values, and chart data match the report state when export starts. Later filter or viewport changes cannot mix different report states within the PDF.
6. Charts, text, and other report blocks finish rendering before capture. Chart labels and plotted data appear completely; charts that fit a page are not split across pages.
7. Every exported page is A4 portrait (210mm by 297mm), with 10mm margins, including when the saved report specifies Letter or landscape. A long report produces additional pages rather than a single extremely tall page or a truncated document. Saved paper settings remain unchanged; the existing filename convention and light appearance are retained.
8. Export does not alter the live report's filters, layout, theme, or scroll position and does not include navigation, editor controls, or export progress messages in the PDF.
9. Existing visibility and disabled-button rules remain effective. Repeated clicks do not start duplicate exports. A capture or download failure shows error feedback, clears the busy state and temporary content, and permits retry; failed capture is not reported as success.

## Implementation Notes

No data, API, or schema changes are expected. Reuse the saved definition, applied filter values, and loaded dataset results. Resolve export layout explicitly rather than inheriting the live browser breakpoint. Keep A4 sizing and export layout decisions local to the export path so a future mobile/tablet layout can be added without changing data loading or the live report. Do not build additional profiles or settings now.

Start with `frontend/src/views/ReportView.vue`, `frontend/src/components/report/ReportCanvas.vue`, `frontend/src/components/report/reportResponsiveLayout.ts`, the table and chart blocks under `frontend/src/components/report/blocks/`, and export styles in `frontend/src/style.css`.

Reproduce the defect with a real download before selecting the exact fix. Keep export sizing explicitly fixed to A4 portrait and independent of mobile/desktop browser selection. Expanding a table must also move later content so it cannot overlap; changing overflow alone is insufficient. Apply export-specific sizing to the temporary document rather than the live report.

Prefer the existing browser export tools. Keep pagination and capture small enough to avoid depending on one oversized image for an entire long report. Account for both the grid layout and block rendering before capture, with bounded failure handling and cleanup. The precise component boundaries and capture method are implementation choices, provided the acceptance criteria hold.

Relevant library guidance:

- [html2pdf.js documentation](https://ekoopmans.github.io/html2pdf.js/) describes capture options and page-break rules. Those controls are useful, but keeping every table unbroken does not meet the long-table requirement.
- [html2canvas FAQ](https://html2canvas.hertzen.com/faq.html) documents browser image-size limits that can cause blank or cut-off output. This supports checking actual long-report output and avoiding one unbounded capture.

## Test Expectations

- Add focused behavior checks for fixed A4 dimensions and viewport-independent layout, complete table pagination, column continuation, stable report state during export, readiness, and cleanup/retry. Retain coverage of existing export permissions and restrictions.
- Download and inspect actual PDFs from browser runs at 390px, 768px, 769px, and 1280px. Confirm matching page layout and content across phone, tablet, and desktop widths, including reports with and without configured mobile overrides.
- Use reports with short and multipage tables, wide columns, long cell text, grouped rows and totals, charts, text, and filters. Include a grouped table that intentionally omits dataset fields, and a table that is both wide and long with duplicate/blank row labels and uneven cell heights. Check configured column selection and repeated row numbers across column groups. Check first/last rows and report-displayed columns against known loaded values and inspect each page for clipping, overlap, missing headings, and readable labels.
- Verify A4 portrait output for reports with absent print settings, A4 settings, and Letter/landscape settings; confirm the saved settings are unchanged. Verify export from scrolled positions, applied filter changes, and exports from light and dark application themes.
- Exercise filter/viewport changes during export, repeated clicks, render/capture failure, and a successful retry. Confirm the live report is unchanged after success and failure.
- Run the relevant frontend tests and `npm --prefix frontend run build`. Source-pattern checks alone are not sufficient evidence for this visual fix.

## Risks

- Very large browser captures can exceed memory or image-size limits, especially on phones. Test representative long reports and avoid silently saving incomplete output.
- Grid positions, expanded table heights, and PDF page breaks can disagree, causing overlap or gaps. Validate the final downloaded pages rather than only the temporary HTML.
- A4 width cannot fit arbitrarily many columns at readable sizes. Wide-table continuation is a fallback and must preserve row/group context and totals. Excessive shrinking can hide the defect while making the document unusable.
- Reports previously configured for Letter or landscape will now download as A4 portrait. This is an intentional behavior change for this action; saved settings must remain intact.
- Fonts and chart rendering can finish after grid layout. Capture readiness must cover the rendered content without leaving an export stuck indefinitely.

## Open Questions

None.
