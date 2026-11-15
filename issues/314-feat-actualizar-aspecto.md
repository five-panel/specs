# Issue #314: feat: Actualizar el aspecto de las tablas según el diseño de Doop

Source issue: https://github.com/five-panel/five-panel/issues/314

## TL;DR

Match ordinary record tables to the Doop references: compact desktop tables, reduced tablet tables, and readable phone lists. Preserve existing data, saved columns, and record actions.

References: [desktop](https://five-pane-dev-server.tail7d89bf.ts.net/c/J88FjcHO5C?frame=10YN7Fxs_p), [materials](https://five-pane-dev-server.tail7d89bf.ts.net/c/J88FjcHO5C?frame=GCRnLvRaOq), [tablet](https://five-pane-dev-server.tail7d89bf.ts.net/c/J88FjcHO5C?frame=3DjrHQBomz), [phone](https://five-pane-dev-server.tail7d89bf.ts.net/c/J88FjcHO5C?frame=vk7BX1UtYg).

Companion documents: [Technical implementation plan](314-feat-actualizar-aspecto-implementation-plan.md) and [Short implementation guide](314-feat-actualizar-aspecto-implementation.md).

## What will change

**Code impact: Medium.** The change stays in the frontend but covers three layouts, shared value formatting, and existing table behavior.

Update `ModelView.vue` and add small shared presentation components. Reuse the current data store, model metadata, saved views, forms, and record handlers. Keep fonts and table styles local to this screen.

Add narrow-screen paging and a read-only record detail dialog. No new saved view settings are needed.

## How it would work

1. Load the existing accessible view, configured columns, and records. Keep search, filters, server sorting, saved preferences, and protections against outdated responses.
2. At widths of 1024 px and above, follow the desktop reference for the view name, search, controls, New button, counts, and Export. Use compact rows, quiet headers, and thin horizontal separators. Keep headers visible during scrolling and all configured columns available horizontally.
3. From 768 to 1023 px, show a reduced table with eight records per page. Below 768 px, show a list with six records per page. Previous and next actions reuse existing record loading.
4. Derive the summary automatically: the first configured data column is the title; reserve the first remaining money field as the amount, falling back to a number or measurement field; the first remaining select field supplies a badge. Keep currency or unit attached to its value. Show up to three remaining columns in saved order, using each field once. Respect field visibility, omit unavailable roles, and use the translated word “Record” when the title is missing. This is a generic implementation rule for the Doop layout, without new settings or business-specific field names.
5. Format values from their real types and labels. Preserve zero, false, calendar dates, nested relations, and older values. Use neutral select badges; do not guess business meanings or currencies.
6. A record arrow opens all configured visible values in a read-only dialog. Keep Edit separate and reuse existing create, edit, and delete flows. Keep Export's current behavior. Provide loading, empty, and error states with retry, keyboard focus, and touch controls.

## Done when

- Screens at 1440, 768, and 390 px match the reference layout, spacing, and density with at least two real models.
- Search, filters, sorting, record loading, and saved columns work in all layouts, including views opened by UUID or slug.
- Summaries and details preserve configured values, visibility, currencies, units, zero, false, and long text.
- Create, edit, cancel, failed-save recovery, delete confirmation or refusal, and CSV export remain available.
- Light and dark themes, Spanish and English, keyboard use, short screens, loading, empty results, and retry remain usable.
- Resizing saves no configuration or records; other tables remain unchanged; focused tests, browser checks, and the frontend build pass.

## Out of scope

- Sidebar, report, administration, and form redesigns.
- New APIs, database fields, permissions, or mobile settings.
- Bulk selection, configurable pinning, totals, or new related-record navigation.
- Expanding CSV scope. Any remaining initial-load timing correction belongs to [#284](https://github.com/five-panel/five-panel/issues/284); verify the combined behavior before shipping.
