# Record Table Appearance Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Apply the approved Doop layouts to ordinary record tables while preserving their data, settings, and actions.

**Architecture:** Keep `ModelView.vue` as the single owner of query state and record actions. Add a presentation composable and small display components; use the existing stores and server contracts. Keep the desktop DataTable mounted as the single saved-state and CSV owner while showing a separate compact presentation below 1024 px.

**Tech Stack:** Vue 3, TypeScript, Pinia, installed PrimeVue 4.5.1, Vue I18n, existing GraphQL client, Node test runner with tsx, Playwright.

**Spec:** [Approved behavior](314-feat-actualizar-aspecto.md). **Guide:** [Worked example and explanation](314-feat-actualizar-aspecto-implementation.md).

## Global Constraints

- Viewport widths: desktop at least 1024 px; tablet 768–1023 px, eight records per page; phone below 768 px, six records per page.
- Use existing configured columns and real model types. Add no saved mobile settings, API, database field, or permission.
- Preserve existing record actions, CSV behavior, shared configuration, server ordering, and outdated-response guards.
- Scope Inter and CSS to ordinary record tables. Preserve the sidebar, reports, administration tables, forms, and global theme roles.
- Use shared composables and direct handlers. Backend validation, access checks, persistence, history, and automations remain in their current services and repositories.

## Review Focus

- Hidden, nullable, nested, and older values must remain safe and readable without exposing dependency fields: Task 1.
- Reloading or resizing must preserve shared columns and local filters without saving responsive state: Task 2.
- Slow, failed, and overlapping first/range requests must not replace the current view or strand paging: Task 3.
- Failed saves, refused deletes, and a page removed by deletion must preserve recovery and focus: Tasks 3 and 4.
- Non-admin users, long translations, dark theme, and short screens must have working controls and no unauthorized configuration writes: Tasks 2 and 4.

---

## Existing contracts and implementation boundaries

Research baseline: source main `2c15e2a42569bc63c6957b318d38e9b68c0b3524`, including the merged UUID-or-slug view lookup. The following are existing behavior; the tasks below describe proposed code.

| Interface or storage | Existing contract to retain |
| --- | --- |
| Page and view identity | `/:tenantSlug/views/:uuid`; the `uuid` route parameter accepts UUID or slug. Use the resolved canonical view UUID for settings and `datatable-<UUID>` local storage. |
| View and model reads | `GET /views` returns accessible `UserView[]`; `GET /models` supplies full model metadata. `/models/fields/<slug>` is the reduced column-picker tree, not the formatting source. |
| Saved columns | `UserViewColumn = { id: string; label: string; width?: number }`; IDs may be dotted paths. `view.data` holds `modelSlug`, `columns`, and `query.query[model.pluralSlug]`, with other properties preserved. |
| Configuration write | Existing `userViewsStore.updateView` → `PUT /views/<UUID>` with `{ data: nextViewData }` → existing `View` response and view refresh. The server requires authentication, tenant context, superuser access, and the view service's access checks. A view's `write` permission alone is insufficient. |
| Record reads | `dataStore.loadRecords(query, modelSlug, { first: 50, where, orderBy, search })`; range reads use `loadRecordsAtOffset(query, modelSlug, offset, { count, where, orderBy, search })`; successful saves/deletes reuse `refreshRecords`. All return `Promise<void>`. |
| Query and response | `POST /graphql`; the existing query builder supplies Relay `edges`, `nodes`, `pageInfo`, and `totalCount`. The store holds a sparse `Map` of absolute offsets, total, and request metadata. Range cursor is base64 of `offset - 1`; retain the current builder. |
| Record actions | `openRecordForm('create' \| 'edit', id?)` uses `GET /form-templates/assigned?modelId=<UUID>&operation=create` or `edit`, then the existing template/form flow. For the sample model, generated GraphQL operations are `createProduct(input)`, `updateProduct(id, input)`, and `deleteProduct(id)`, returning a record ID. Keep existing model-derived names, validated form inputs, save/delete handlers, and server checks. |
| Storage and side effects | Records remain in tenant/model-scoped `custom_data.data` and existing source columns. Shared view configuration remains in `views.data`; `userViews` holds sidebar placement. Responsive state is local memory only. Explicit saves/deletes retain transactional history and automations. |

The sample model metadata, complete saved `view.data`, request, response, and UI transition are in the guide. Both documents use `product` / `products`, view `11111111-1111-4111-8111-111111111111`, and record `33333333-3333-4333-8333-333333333333`. Its six configured fields map to this **proposed local projection**, which is never persisted:

```json
{
  "title": "name",
  "badge": "stage",
  "amount": "total",
  "secondary": ["code", "owner", "dueDate"]
}
```

Use the existing visibility query helper on a clone. Add only display dependencies needed by configured paths, including currency/unit and relation labels at the correct nested path; never add them as displayed columns. Use the real `pluralSlug`, never `slug + 's'`. Bound metadata traversal by the finite configured path, including models with cyclic relations.

No backend or storage change is planned. Imports, files, jobs, authorization, backups, reports, and existing history remain on their current paths. The tenant-backup history/automation gap belongs to [#290](https://github.com/five-panel/five-panel/issues/290). Initial-load timing work remains with [#284](https://github.com/five-panel/five-panel/issues/284), but held and overlapping requests must also pass the new layout checks.

## Task 1: Shared column resolution and value presentation

**Files:** Create `frontend/src/composables/useRecordTablePresentation.ts`, `frontend/src/components/records/RecordTableValue.vue`, and `frontend/test/recordTablePresentation.test.ts`. Reuse `frontend/src/types/{View,Model,metadata}.ts` and `frontend/src/utils/{fieldVisibility,dateUtils}.ts`.

**Interfaces:** Export these proposed types and functions from the composable. `Ref`, `ComputedRef`, `BaseField`, `UserViewColumn`, `UserViewQuery`, and `Model` are existing Vue/project types.

```ts
interface ResolvedRecordColumn extends UserViewColumn {
  metadata: Array<{ path: string[]; field: BaseField; siblings: BaseField[] }>;
}
interface RecordTableProjection {
  title: string | null;
  badge: string | null;
  amount: string | null;
  secondary: string[];
}
interface RecordTableCell {
  text: string;
  kind: 'text' | 'link' | 'badge';
  align: 'start' | 'end';
  href?: string;
}
function useRecordTablePresentation(input: {
  columns: Readonly<Ref<UserViewColumn[]>>;
  query: Readonly<Ref<UserViewQuery>>;
  model: Readonly<Ref<Model | undefined>>;
  models: Readonly<Ref<Model[]>>;
}): {
  columns: ComputedRef<ResolvedRecordColumn[]>;
  projection: ComputedRef<RecordTableProjection>;
  runtimeQuery: ComputedRef<UserViewQuery>;
};
function formatRecordTableCell(input: {
  column: ResolvedRecordColumn;
  record: Record<string, unknown>;
  locale: string;
  yesLabel: string;
  noLabel: string;
}): RecordTableCell | null;
```

`metadata` retains each field and sibling context along the configured path so visibility and companion lookup use the correct record object. An empty chain permits safe fallback for an unknown legacy field. `null` from the formatter means hidden; a visible missing value uses an em dash. `RecordTableValue` takes `{ column, record }`, reads the active locale, and renders this result as escaped text or a valid link. It performs no request or write.

- [ ] Write fixture-based behavior tests, including `summary_reserves_currency_and_each_role_once`, `nested_visibility_does_not_expose_controllers`, and `formatting_preserves_legacy_values`. Assert that the guide fixture's projection equals the JSON above, the cloned runtime selection includes `totalCurrency: true`, and the saved query is unchanged. Include hidden ancestors, missing metadata, cyclic relations, absent roles, `0`, `false`, unknown select values, long strings, arrays, and objects.

  For the guide fixture, retain its original query and use the projection JSON above as `expectedProjection`:

  ```ts
  assert.deepEqual(presentation.projection.value, expectedProjection);
  assert.equal(presentation.runtimeQuery.value.query.products.totalCurrency, true);
  assert.deepEqual(savedView.data.query, originalQuery);
  ```

- [ ] Run `node --import tsx --test test/recordTablePresentation.test.ts` in `frontend`; confirm the new expectations fail before implementation.
- [ ] Implement metadata resolution and the automatic projection. Reserve the first configured column as title; reserve the first remaining money field, otherwise number/measurement, and its companion; choose the first remaining non-companion select as badge; then take up to three remaining columns in saved order. Use each field once. Fix roles per view rather than reshuffling them for each record. Hidden/missing title uses translated “Record”; other unavailable summary roles are omitted. Details retain every configured visible column.
- [ ] Implement the formatter and value component. Use exact select labels with raw fallback and neutral badges. Format only fields with numeric metadata, avoiding default decimal truncation with `maximumFractionDigits: 20`; use the actual currency code with `currencyDisplay: 'code'`, or number plus literal code if invalid. Measurements use the actual companion unit; respect companion visibility too. Calendar fields use `parseCalendarDate` and localized medium dates, including older timestamp-suffixed calendar values. Keep raw text when a legacy value cannot be formatted. Booleans use translated Yes/No. Relations use declared label/value keys with raw/ID fallback; compound values use escaped compact JSON. Only valid metadata-backed HTTP(S) URLs and email addresses become links. Never parse numeric-looking text identifiers or infer meaning from labels.
- [ ] Rerun the new tests plus `test/fieldVisibility.test.ts` and `test/calendarDates.test.ts`; expect all passing. Verify both Spanish and English, missing currency, an unknown currency code, nested companions, and no date shift across time zones.

## Task 2: Desktop layout and real shared controls

**Files:** Modify `frontend/src/views/ModelView.vue` and `frontend/src/locales/{en-US,es-ES}/table.json`. Create `frontend/src/components/records/RecordTableToolbar.vue`, `frontend/src/components/records/recordTable.css`, and `frontend/test/recordTableControls.test.ts`. Reuse the existing data, models, auth, and user-view stores.

**Interfaces:** Consume Task 1's columns and runtime query. Toolbar props carry view name, resolved columns, search, existing filter state, sort field/order, loading, and `canConfigure`. Emit `search(string)`, `filters(Record<string, PrimeVueFilterMeta | PrimeVueFilterGroup>)`, `sort({ field: string | null; order: 1 | -1 | null })`, `create()`, `export()`, and `columns(event: MouseEvent)` for the existing menu anchor. Keep business handlers in `ModelView`; add direct `setRecordSort(command): Promise<void>` with the same sort payload and let both the existing DataTable sort event and toolbar call it.

- [ ] Add failing setup tests using `frontend/test/helpers/componentSetup.ts`: `controls_forward_same_query_options`, `responsive_change_never_saves_view`, and `configuration_denial_restores_confirmed_columns`. Assert search remains debounced by 400 ms; clearing search preserves filters/sort; clearing filters clears search/rules but preserves sort; zero-valued and restored legacy filters survive. Assert configuration uses the canonical UUID and real plural slug, preserves unrelated `data`, and performs no write for a non-superuser.

  With search `Cable` and descending `total` sort, record the read calls. After 400 ms and a resize:

  ```ts
  assert.equal(lastRead.options.search, 'Cable');
  assert.deepEqual(lastRead.options.orderBy, [{ field: 'total', direction: 'DESC' }]);
  assert.equal(viewWrites.length, 0);
  ```

- [ ] Run `node --import tsx --test test/recordTableControls.test.ts test/modelViewSorting.test.ts test/tableFilters.test.ts`; confirm new behavior tests fail.
- [ ] Connect the presentation composable to configured data columns, normalizing the current local `TablePref` column `header` to `label`. Preserve IDs/order/widths and render desktop cells through `RecordTableValue`; keep existing action columns outside the projection.
- [ ] Implement toolbar arrangements from Doop. Desktop has the view-name chip, search, working controls, New, Export, and record/configured-field counts in its footer. Tablet exposes Filter, Columns, Export, and sorting through a working menu. Phone exposes search, Filter, Sort, New, and Columns/Export through overflow. Narrow layouts show the record count and a range/paging footer. Reuse supported text/select/number filters, operators, normalization, and server sort conversion; add no new nested-filter framework. Column actions reuse current add/hide/reorder/resize flow, correcting plural-key construction where necessary. Gate configuration writes with the current superuser policy and retain server authority; record CRUD is a separate existing policy.
- [ ] Add local styles under `.record-table-surface`, including classes on teleported menus/dialogs. Target a 36 px desktop header and measured 40 px single-line outer rows, including borders; set virtual `itemSize` to exactly 40 and match skeletons. Use 13 px/20 px Inter body text, 12 px cell padding, thin horizontal separators, right numeric alignment, and 32 px desktop controls. Start with desktop content padding 20 px 24 px 12 px, tablet 24 px, phone 16 px; compare against references. Use explicit pixels where the app's 14 px root would distort sizing. Use existing semantic surface/text/border tokens for dark theme.
- [ ] Fit the table inside the existing `100dvh` shell with `min-height: 0` and `min-width: 0`; preserve the mobile tenant header and sidebar behavior. Keep the header visible and horizontal access to all desktop columns. Keep the DataTable mounted with `v-show`, its full total-length value and existing state key. Ignore hidden virtual-scroll load events. Do not feed compact rows to CSV or add an exportable detail field; preserve even the existing export's blank-placeholder/pseudo-column behavior.
- [ ] On explicit column-save failure, show the current error and restore/reload the last confirmed view. Distinguish a failed save from a failed refresh after a successful save; retry the refresh without repeating the write. Rerun the control tests and existing sorting, filter, and slug tests; expect passing without unauthorized or resize-triggered writes.

## Task 3: Compact paging, details, and recovery

**Files:** Modify `frontend/src/views/ModelView.vue`, the local CSS/translations, and `frontend/test/modelViewRequests.test.ts`. Create `frontend/src/components/records/RecordTableCompact.vue`, `frontend/src/components/records/RecordTableDetails.vue`, and `frontend/test/modelViewResponsive.test.ts`.

**Interfaces:** Compact display consumes layout (`tablet` or `phone`), resolved columns, projection, sparse page rows, first offset, total (`number | null`), page size, loading, and error. Emit `page(-1 | 1)`, `details(recordId)`, `edit(recordId)`, and `delete(recordId)`; placeholders emit nothing. Details takes the selected record and all resolved columns; emit `close()` and `edit(recordId)`. Add direct `loadCompactPage(): Promise<void>`, `retryRecordLoad(): Promise<void>`, `openRecordDetails(recordId: string): void`, and `closeRecordDetails(): void` in `ModelView`.

- [ ] Write failing setup tests: `page_uses_absolute_missing_range`, `stale_range_cannot_clear_new_loading`, `deleted_last_page_clamps_and_recovers`, and `details_open_does_not_fetch_or_save`. Use more than 50 records and deferred first/range requests. Assert tablet offsets 0/8/16 and phone 0/6/12, count from server total, no extra request for cached rows, same query options after refresh, and no action on unloaded placeholders. Cover initial failure, range failure/retry, view/tenant/query changes, and late errors/finalizers.

  With total 65, offsets 0–49 cached, and phone page 9, assert the missing range. Separately, opening the guide record must update local details without a write:

  ```ts
  assert.equal(lastRange.offset, 54);
  assert.equal(lastRange.options.count, 6);
  assert.equal(state.detailRecordId, '33333333-3333-4333-8333-333333333333');
  assert.equal(viewWrites.length, 0);
  ```

- [ ] Run `node --import tsx --test test/modelViewResponsive.test.ts test/modelViewRequests.test.ts`; confirm the new cases fail.
- [ ] Implement local `compactPage = 0`, layout, and detail record ID. Compute first as page × size; request only missing ranges through `loadRecordsAtOffset`, with count capped by remaining total. Skip zero-count requests. Preserve sparse absolute order; never sort/filter the loaded subset or fetch all results. Reset the compact page on layout, query, tenant, or view changes, while preserving query/preferences/cache for width changes. Clamp to the last valid page after refresh/deletion and load it if absent.
- [ ] Tie compact loading, errors, and finalizers to the canonical view, existing query/request generation, and a compact request ID. Disable paging while its request is pending. Distinguish pending/unknown totals, successful empty results, no configured columns, and failed reads. Add visible error and Retry: range retry repeats that range; first/refresh retry repeats the current read, never a successful save. Preserve existing store guards rather than replacing them.
- [ ] Implement tablet's maximum five data columns: title plus first secondary beneath it, badge, two remaining secondaries, and amount. Phone arranges the same roles as title/arrow, badge/first secondary, and remaining secondaries with amount. Start with 70 px tablet and 120 px phone minimum row heights; allow growth for long text. Use 44 px touch controls and a range/previous/next footer. No compact virtual row-height assumption is needed.
- [ ] Implement a read-only PrimeVue detail dialog listing all configured visible labels/values through `RecordTableValue`. Opening uses the loaded record, with no fetch, route, or write. Edit closes it and calls the existing form handler; assigned edit forms are not a substitute for full details. Close on view/query change, contain focus, support Escape, and return focus to the visible trigger or search after resize/removal. Reuse current create/template chooser/save/delete handlers, preserving drafts on save failure and delete refusal.
- [ ] Rerun responsive/request tests plus existing `test/modelViewRecordEdit.test.ts` and `test/dataStoreSorting.test.ts`; expect passing. Verify the hidden desktop scroller cannot issue competing loads.

## Task 4: Browser evidence and final regression checks

**Files:** Create `e2e/test/recordTables.spec.ts`; reuse `e2e/test/{fixtures,support}.ts` and the existing isolated E2E environment. Adapt affected layout selectors in `e2e/test/{records,views,viewSlug,record-form-accessibility}.spec.ts` while retaining their behavioral assertions. Keep fixture setup within existing test contracts.

**Interfaces:** Use actual controls and existing network routes. Use at least two existing real model definitions, including nested/nullable fields and numeric-looking text, without hard-coded business meanings. The guide's small model is an additional deterministic formatting fixture, not a replacement for real-model checks.

- [ ] Add browser cases `layout_matches_each_reference`, `actions_and_failures_work_at_each_width`, and `keyboard_and_configuration_access_survive_resize`. Capture 1440×900, 768×1053, and 390×1166; also check 767/768/1023/1024 boundaries and 390×844. Measure actual desktop row rectangles at 40 px over a long virtual scroll, verify sticky headers/horizontal access, and compare toolbar, spacing, footer, and density against all four Doop frames. Long text must remain available in details.

  Assert measured layout and network behavior around a resize and local details action:

  ```ts
  expect(measuredDesktopRowHeight).toBe(40);
  expect(capturedMutationRequests).toHaveLength(0);
  ```

- [ ] Exercise search/filter/sort, saved columns, UUID/slug entry, cached and uncached pages, held/overlapping requests, empty/no-column/error/Retry states, and final-page deletion. Assert width changes and local details cause no configuration/record mutation. Compare CSV before/after resizing with the same loaded collection and configured columns. Exercise create/edit/template choice/cancel, failed save with retained draft, failed post-save refresh without duplicate mutation, delete confirmation/refusal, and focus return.
- [ ] Check Spanish/English, light/dark, non-superuser access, keyboard focus, touch targets, the short viewport, sidebar resizing/overlay, and an unrelated report and administration table. Pass when controls remain reachable and the unrelated surfaces retain their existing appearance. If #287 datetime work merges first, preserve its contract in these display checks without extending this issue's scope.
- [ ] Run from `frontend`: `node --import tsx --test test/recordTablePresentation.test.ts test/recordTableControls.test.ts test/modelViewResponsive.test.ts test/modelViewSorting.test.ts test/modelViewRequests.test.ts test/dataStoreSorting.test.ts test/tableFilters.test.ts test/modelViewRecordEdit.test.ts test/modelViewSlug.test.ts test/fieldVisibility.test.ts test/calendarDates.test.ts test/semanticThemeTokens.test.js test/reportTableStyles.test.ts`, then `npm run build`. Expect zero failures and a successful type check/build.
- [ ] Run from `e2e`: `npm run typecheck` and `npm test -- test/recordTables.spec.ts test/records.spec.ts test/views.spec.ts test/viewSlug.spec.ts test/access.spec.ts test/record-form-accessibility.spec.ts test/reports.spec.ts`. Expect all selected cases passing with retained screenshots/traces for failures. Resolve failures or record a concrete blocking dependency before declaring the approved flow complete.

The existing focused baseline passed **44 tests** at the source revision above. No new presentation, browser comparison, or build is claimed as completed by this planning document. Record actual implementation results when executing the tasks.
