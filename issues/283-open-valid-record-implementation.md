# Issue #283: Short implementation guide

[Spec](283-open-valid-record.md) · [Technical implementation plan](283-open-valid-record-implementation-plan.md)

Source issue: https://github.com/benitogonzalezh/five-panel/issues/283

## What changes

A page ID and its slug are two ways to find the same page. The router and page dispatcher already accept both. `ModelView.vue`, which opens record tables, currently checks only the ID.

Change that one lookup to use the existing `findViewableViewByIdOrSlug` function. The result is the same saved page object that an ID link would return. Keep all later loading, settings, and recovery steps. The new files are tests for the table setup and browser flow; the plan lists the existing tests that also need small updates.

Page definitions and columns stay in `views.data`, and business records stay in `custom_data`. Browser table settings keep the page ID in their storage key. No database, API, file, job, or history change is needed.

## Concrete flow example

This is a small sample fixture using existing response shapes, not the deployed Zipp page or its real IDs. The example table selects only `sku` from the existing `productVariant` model. It has no records or saved filters/sort settings.

**1. Open the link.** A signed-in user enters `/zipp/views/commerce-productVariants`. The normal authenticated `GET /views` has loaded this example response into `userViewsStore`:

```json
[
  {
    "id": "11111111-1111-4111-8111-111111111111",
    "tenantId": "22222222-2222-4222-8222-222222222222",
    "name": "Product variants",
    "slug": "commerce-productVariants",
    "viewType": "view",
    "parentId": null,
    "data": {
      "modelSlug": "productVariant",
      "displayType": "table",
      "columns": [{ "id": "sku", "label": "SKU", "width": 220 }],
      "query": { "query": { "productVariants": { "id": true, "sku": true } } }
    },
    "isPublic": false,
    "publicPermission": "read",
    "createdBy": "33333333-3333-4333-8333-333333333333",
    "updatedBy": "33333333-3333-4333-8333-333333333333",
    "createdAt": "2026-11-12T00:00:00.000Z",
    "updatedAt": "2026-11-12T00:00:00.000Z",
    "permission": "write",
    "mode": "owner",
    "orderIndex": 0,
    "placementId": "44444444-4444-4444-8444-444444444444",
    "parentPlacementId": null
  }
]
```

**2. Find the page.** The router and dispatcher already find that object by slug. The changed table lookup will find it too, instead of showing an error and redirecting. The router keeps `lastViewId` as `11111111-1111-4111-8111-111111111111`. The table sets `currentViewId` to that same ID and uses browser key `datatable-11111111-1111-4111-8111-111111111111`.

**3. Load the table.** Existing model calls, `GET /models/fields/productVariant` and `GET /models`, supply the model fields and definition. The table selects `productVariant`, builds the saved `SKU` column at width 220, and passes the saved selection to `dataStore.loadRecords` with model slug `productVariant` and initial page size 50. The store expands the selection into the existing connection query. URQL sends this logical request to `/graphql`; its existing GET/POST encoding and authentication stay unchanged:

```json
{
  "query": "query { productVariants(first: 50) { edges { node { id sku } cursor } nodes { id sku } pageInfo { hasNextPage hasPreviousPage startCursor endCursor } totalCount } }",
  "variables": {}
}
```

The backend uses the existing model and company checks to read the records. No business record is changed. A successful empty response is:

```json
{
  "data": {
    "productVariants": {
      "edges": [],
      "nodes": [],
      "pageInfo": {
        "hasNextPage": false,
        "hasPreviousPage": false,
        "startCursor": null,
        "endCursor": null
      },
      "totalCount": 0
    }
  }
}
```

**4. Show the result.** The record store records a total of zero under `productVariant`. The table finishes loading with an empty row list and shows the existing “No records found.” message, or “No se encontraron registros.” in Spanish. The address remains `/zipp/views/commerce-productVariants`, with no unavailable-page toast. Reload repeats this flow. Opening `/zipp/views/11111111-1111-4111-8111-111111111111` uses the same page, settings, and record query.

## What stays protected

The shared lookup still uses the current user’s page list and existing company route checks. Missing pages and real request failures keep their existing handling. Reports keep their slug routes and ID-based requests. The implementation checks both languages, phone and desktop screens, populated and empty tables, saved settings, unavailable links, and normal table actions.

This is the planned result. The application change and its browser checks have not been implemented yet.
