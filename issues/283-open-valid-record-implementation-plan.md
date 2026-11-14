# Issue #283: Technical implementation plan

> **For agentic workers:** Use `superpowers:executing-plans` to implement these tasks in order, or `superpowers:subagent-driven-development` if the implementation session explicitly chooses delegation. The checkboxes describe future application work.

**Goal:** Open an existing record table by ID or slug, keep a valid slug URL, and preserve the table’s saved ID and behavior.

**Architecture:** Reuse `userViewsStore.findViewableViewByIdOrSlug` in `ModelView.loadData`. Add tests around the real component setup and routed browser page. No new production function, service, endpoint, storage, or dependency is needed.

**Tech stack:** Vue 3, Pinia, Vue Router, PrimeVue, TypeScript, and Playwright.

**Spec:** [Agreed behavior and completion checks](283-open-valid-record.md). [Short implementation guide](283-open-valid-record-implementation.md).

Source issue: https://github.com/benitogonzalezh/five-panel/issues/283

**Source baseline:** Complete application review at `9c777bd8f7f7cd492c3d4e5f105a631b00c736fb`, including changed files at current main `c3142a20994974b979a37f7fbbcb27e3b3c9ef7a`. Implement on current main, preserving its table-state restoration and stale-request protections. Paths below belong to the application repository.

## Global constraints

- Keep exact existing slugs, ID-first lookup priority, and the current user/company page list.
- Keep the resolved page ID for settings and the saved model information for records.
- Preserve recovery, table actions, empty-state copy, and report URL behavior.
- Make no backend, storage, dependency, permission, or unrelated refactor changes.

## Review focus

- A fallback can show correct rows; Task 2 checks the address and transient toasts too.
- Saved settings must survive slug entry; both tasks check their shared ID.
- Pending loads must not overwrite a new table; Task 1 checks delayed loads.
- Unavailable links and failed requests need recovery; both tasks cover these controls.
- Reports, both languages, and phone entry must work; Task 2 checks these paths.

## Files and responsibilities

| File | Planned work |
| --- | --- |
| `frontend/src/views/ModelView.vue` | Change the route lookup inside `loadData`; keep downstream loading and recovery unchanged. |
| `frontend/src/stores/userViewsStore.ts`, `frontend/src/router/index.ts`, `frontend/src/views/ViewDispatcher.vue` | Reuse current lookup, list loading, routing, and dispatch; no production changes expected. |
| `frontend/test/modelViewSlug.test.ts` | New tests mounting the real table setup through `test/helpers/componentSetup.ts`, using the real page lookup. |
| `frontend/test/modelViewRecordEdit.test.ts`, `frontend/test/modelViewRequests.test.ts`, `frontend/test/modelViewSorting.test.ts` | Update store substitutes to expose the lookup now used by the component; retain their existing assertions. The latter two files are already on current main. |
| `e2e/test/viewSlug.spec.ts` | New browser checks using `fixtures.ts` and `support.ts` for direct slug links, reload, saved settings, locales, phone size, and unavailable links. |
| `e2e/test/reports.spec.ts` | Extend the existing report-filter journey with explicit slug-path and ID-based request assertions. |

## Existing contracts to preserve

| Boundary | Input and result |
| --- | --- |
| Route and lookup | `/:tenantSlug/views/:uuid` already accepts IDs or slugs. `findViewableViewByIdOrSlug(viewIdOrSlug?: string \| null): UserView \| undefined` tries exact ID first, then exact slug and the existing viewable-type check. Preserve case, including `commerce-productVariants`. |
| Page list | `viewService.getViews()` reads authenticated `GET /views` into `UserView[]`. Resolve only from that list; never search `/views/all` or `/views/public`. |
| Table identity | `currentViewId = view.id`; browser table state uses `datatable-${view.id}`. The router already saves `lastViewId = view.id`. Do not use a slug or sidebar `placementId` for either value. |
| Model and records | Keep `loadColumnPreferences({ view, generation })` → `GET /models/fields/productVariant`, then `loadCurrentModel({ view, generation })` → `GET /models`. Select the saved model slug. `dataStore.loadRecords(query, modelSlug, options)` keeps `UserViewQuery`, initial `first: 50`, and filter/search/order options. Plural selection `productVariants` differs from page slug `commerce-productVariants`. |
| Existing writes | Column settings still use `updateView(view)` → `editViewData(view.id, view.data)` → `PUT /views/:id`, with body property `data` equal to `view.data`. Keep record actions and history unchanged. Opening a link introduces no new backend write. |
| Recovery | Missing lookup result keeps preferred-page recovery by ID or tenant home, stale `lastViewId` cleanup, and existing messages. A view-list or record-query failure remains a real failure. |
| Reports | The router still changes report ID URLs to their slug while preserving query values. Report definition, filter-option, and dataset calls continue using the resolved report ID. |

The guide contains the complete example response and record operation. Its IDs are illustrative. All schemas, records, shares, placements, sessions, templates, settings, files, imports, jobs, automations, and history stay unchanged. Inaccessible controls use pages absent from `GET /views`; changing stale references with `permission: null` is outside scope.

## Task 1: Pin down the table lookup and make the focused change

- [ ] Add `modelViewSlug.test.ts` with `mountSetup`/`flushSetup` and the real store seeded with the guide’s page. Substitute model/data requests, router actions, and toasts; keep the real lookup.
- [ ] Run it on unchanged main. Assert slug `commerce-productVariants` initializes ID `11111111-1111-4111-8111-111111111111`, its table-state key, and `productVariant` record reads, without a toast or navigation. It must fail on the existing lookup.
- [ ] Replace `findViewableViewById(viewId)` with `findViewableViewByIdOrSlug(viewId)` in `loadData`. Keep the ID-only helper elsewhere, `view?.id`, generation checks, state restoration, awaits, and fallback branches.
- [ ] Update the three existing ModelView store substitutes, retaining their current assertions.
- [ ] Run focused tests covering both inputs, cold list loading, delayed route changes, UUID-keyed settings, and invalid saved state. Missing/section/divider inputs must not read records; preferred-page/no-pages recovery must remain. Failed record requests must keep the record-load error.

Expected result: both valid route forms initialize the same existing table; no other production path changes.

## Task 2: Verify real navigation and existing behavior

- [ ] Add `viewSlug.spec.ts` using `scenario` and authenticated API fixtures. Use `createModel` for `Product variants` / `productVariant` / `productVariants`, with optional string/text field `sku` labelled `SKU`. Send the guide’s `name`, `slug`, `viewType`, `parentId`, `data`, `isPublic`, and `publicPermission` to `POST /views`; use returned IDs and metadata.
- [ ] Enter the slug via `signIn({ page, account, path })`; `openTable` constructs ID paths. Capture transient unavailable toasts throughout navigation. Assert the exact URL after loaded content and reload. Repeat empty, then with `createRecord({ api, modelSlug: 'productVariant', data: { sku: 'SKU-001' } })`. A visible row alone misses the bug.
- [ ] Repeat for `es-ES`/`en-US` at `1440 × 1000`/`390 × 844`. Use fresh contexts with the web base URL and one locale init script; the default page fixture forces Spanish. Close each context.
- [ ] Save width, sort, and filter settings by ID, then navigate by slug and back. Assert restored settings, UUID-based `datatable-${view.id}` and `lastViewId`, including client navigation between tables.
- [ ] Check unknown slug → preferred page or no-pages home, private/unshared page as `scenario.member`, and wrong-company links. Use `foreignApi` for identical and foreign-only slugs. Show only the primary company’s page or existing fallback; add no placement.
- [ ] Delay, fail, then restore a record request to verify loading, existing error, and successful reload. Exercise existing record actions/export after slug entry; reuse `records.spec.ts` coverage.
- [ ] Extend the report test for `producto = 'Café QA'`: `informe` path and filter survive reload; requests use the returned report ID. Keep its setup and access rules.
- [ ] Demonstrate the main slug browser regression failing on unchanged main and passing with the lookup change. Do not mark it as an expected failure or suppress errors globally.

## Verification commands and expected outcomes

Prepare an isolated Docker application using `e2e/README.md`: unique `WORKTREE`, ports/volumes, migrations, report SQL role, E2E dependencies, and Chromium. Use generated fixtures, not real tenant restores or the shared main stack.

```sh
docker compose exec -T frontend ./node_modules/.bin/tsx --test test/modelViewSlug.test.ts test/modelViewRecordEdit.test.ts test/modelViewRequests.test.ts test/modelViewSorting.test.ts test/mainLayoutRouteKey.test.ts test/reportServices.test.ts
docker compose exec -T frontend npm test
docker compose exec -T frontend npm run build
npm --prefix e2e run typecheck
npm --prefix e2e test -- viewSlug.spec.ts access.spec.ts records.spec.ts reports.spec.ts forms.spec.ts
git diff --check
```

Expected: selected tests, build, types, and whitespace checks pass. Record the browser matrix, unchanged slug address, reload, captured toasts, ID-based settings, recovery, and report results. Broaden checks only when a failure or change warrants it.

## Evidence already gathered, and remaining work

Complete source review preceded the proposal. A scratch check of unmodified ModelView setup with the real page store reproduced the slug failure on both versions; ID entry worked. Model/data boundaries were substituted; no browser was rendered. Existing focused checks passed, including 14 frontend auth/menu/table-edit and 40 frontend report checks. These are baseline results, not verification of the fix. Current-main changes were read rather than executed as a full checkout. The implementation, new tests, browser matrix, and builds remain future work.
