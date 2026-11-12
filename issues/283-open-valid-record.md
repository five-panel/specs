# Issue #283: Open valid record-table slug links without a missing-page redirect

Source issue: https://github.com/benitogonzalezh/five-panel/issues/283

## TL;DR

Open valid table links by their readable name, called a slug, without a false “page unavailable” message or a redirect. For example, `/zipp/views/commerce-productVariants` should open the table and keep that address.

Companion documents: [Technical implementation plan](283-open-valid-record-implementation-plan.md) · [Short implementation guide](283-open-valid-record-implementation.md).

## What will change

**Code impact: Small.** Make one focused change to the page lookup in `frontend/src/views/ModelView.vue`. Reuse the existing ID-or-slug lookup in `userViewsStore`, which the router and page dispatcher already use. Add tests for the table setup and real browser flow, and update existing test substitutes for the lookup.

The table will continue using the resolved page ID for saved settings and the existing model information for records. This needs no new service, API, database change, or dependency.

## How it would work

1. The existing router loads the current user’s page list and accepts a matching page ID or exact saved slug, within the active company.
2. The existing dispatcher chooses the table page. Its loader will use the same shared lookup to find the page.
3. The table keeps the resolved page ID for its saved settings and sidebar selection. Its saved model name and query continue to control record loading.
4. A valid slug stays in the address bar. Pages that cannot be found, and real loading failures, keep their existing recovery and error paths.

## Done when

- A valid table slug opens directly, through app navigation, and after reload without an unavailable-page message, fallback redirect, or address change.
- Slug and ID links open the same table and share its existing columns, widths, filters, sorting, and saved page ID.
- Populated and empty tables retain normal loading, search, filtering, sorting, scrolling, record actions, and export. Preserve the empty-record message from [#274](https://github.com/five-panel/five-panel/issues/274).
- Missing pages, pages absent from the user’s list, wrong-company links, and real load failures retain their existing access, error, and recovery behavior, including the no-pages case.
- Existing ID links and report slug routes still work; report filters remain in the URL and report requests still use the page ID.
- Browser checks cover Spanish and English at desktop and phone sizes, and the new slug regression fails before the change and passes afterward.

## Out of scope

- Empty-state wording, page layout, or table redesign.
- Backend queries, migrations, seed data, stored-ID changes, files, integrations, jobs, automations, or change-history work.
- New slug rules, redirects for renamed slugs, permission changes, or unrelated routing and component refactors.
