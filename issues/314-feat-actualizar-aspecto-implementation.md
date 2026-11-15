# Issue #314: Short Implementation Guide

Source issue: https://github.com/five-panel/five-panel/issues/314

Read the [Spec](314-feat-actualizar-aspecto.md) for the agreed behavior and the [Technical implementation plan](314-feat-actualizar-aspecto-implementation-plan.md) for files, interfaces, tasks, and checks. This guide describes proposed frontend work; the new presentation is not implemented yet.

## How the pieces fit

`ModelView.vue` will keep owning the current view, search, filters, sorting, and record actions. A new `useRecordTablePresentation` composable will read saved columns and full model metadata to choose summary fields and format values. Small components will show the toolbar, values, compact rows, and read-only details. Their styles and Inter font stay local to ordinary record tables.

Desktop keeps the existing PrimeVue DataTable, with the Doop spacing and compact rows. Tablet shows eight records per page; phone shows six. Both use the existing sparse record cache and range-loading methods. The desktop table stays mounted while hidden so saved table state and CSV keep one owner. Export receives the same collection and columns at every width.

The first configured column supplies the title. A select supplies the badge, money supplies the amount, and up to three remaining columns supply secondary values. Currency/unit fields stay attached to their value. These roles come from existing column order and metadata; no new mobile settings are saved. Hidden values stay hidden, and missing titles use translated “Record.”

The record arrow opens a local read-only dialog with every configured visible value. Edit remains a separate action using the existing form/template flow. Create, save, delete, and their failure messages keep using the current services. A failed read offers Retry for that read; it never repeats a successful save.

Business records remain in `custom_data.data`. Shared column settings remain in `views.data`, and personal sidebar placement remains in `userViews`. Existing filters/sort use `datatable-<canonical view UUID>` local storage. Page number, layout, and open details stay in memory. Resizing and opening details write nothing to the server, history, or automations.

## Concrete flow example

These are safe example values using the existing contracts. The summary and final local UI state describe the proposed code.

1. Open `/demo/views/11111111-1111-4111-8111-111111111111` at 1440 px, with locale `en-US`, empty search, no filters, and no sort. Existing `GET /views` and `GET /models` supply the accessible view named **Products** and the `product` model (`pluralSlug: products`). Its fields are:

| Field | Label | Type / UI type | Display metadata |
| --- | --- | --- | --- |
| `name` | Product | string / text | Plain text |
| `code` | Code | string / text | Plain text |
| `stage` | Stage | string / select | Option value `ready`, label `Ready` |
| `owner` | Owner | string / text | Plain text |
| `dueDate` | Due date | string / date | Calendar date |
| `total` | Total | money / number | Companion `totalCurrency` |
| `totalCurrency` | Currency | string / select | Options `CLP` and `USD`, same labels; companion `total` |

All seven fields are optional and non-unique. The complete saved `view.data` for this example is:

```json
{
  "modelSlug": "product",
  "displayType": "table",
  "columns": [
    { "id": "name", "label": "Product", "width": 240 },
    { "id": "code", "label": "Code", "width": 120 },
    { "id": "stage", "label": "Stage", "width": 120 },
    { "id": "owner", "label": "Owner", "width": 160 },
    { "id": "dueDate", "label": "Due date", "width": 160 },
    { "id": "total", "label": "Total", "width": 160 }
  ],
  "query": {
    "query": {
      "products": {
        "id": true,
        "name": true,
        "code": true,
        "stage": true,
        "owner": true,
        "dueDate": true,
        "total": true
      }
    }
  }
}
```

2. The existing query helper adds `totalCurrency` to a runtime copy so the amount has its real currency. Saved `view.data` stays unchanged. `ModelView` calls `loadRecords` with the copied query, model slug `product`, and first count 50. The existing builder sends `POST /graphql` with this complete body:

```json
{
  "query": "query { products (first: 50) { edges { node { id name code stage owner dueDate total totalCurrency } cursor } nodes { id name code stage owner dueDate total totalCurrency } pageInfo { hasNextPage hasPreviousPage startCursor endCursor } totalCount } }"
}
```

The normal authenticated tenant/model read returns this complete GraphQL response; no stored record changes:

```json
{
  "data": {
    "products": {
      "edges": [
        {
          "node": {
            "id": "33333333-3333-4333-8333-333333333333",
            "name": "Cable kit",
            "code": "SKU-10",
            "stage": "ready",
            "owner": "Alex Lee",
            "dueDate": "2026-11-14",
            "total": 420000,
            "totalCurrency": "CLP"
          },
          "cursor": "MA=="
        }
      ],
      "nodes": [
        {
          "id": "33333333-3333-4333-8333-333333333333",
          "name": "Cable kit",
          "code": "SKU-10",
          "stage": "ready",
          "owner": "Alex Lee",
          "dueDate": "2026-11-14",
          "total": 420000,
          "totalCurrency": "CLP"
        }
      ],
      "pageInfo": {
        "hasNextPage": false,
        "hasPreviousPage": false,
        "startCursor": "MA==",
        "endCursor": "MA=="
      },
      "totalCount": 1
    }
  }
}
```

3. The store places this record at absolute offset 0 and records total 1. The presentation composable chooses `name` as title, `stage` as badge, `total` as amount, and `code`, `owner`, `dueDate` as secondary fields. Desktop shows all six configured columns; `totalCurrency` is only a formatting dependency.

4. Resize to 390 px. Local state becomes `layout: phone`, `compactPage: 0`, `pageSize: 6`, and `detailRecordId: null`. The record is already cached, so no new read is needed. The list shows **Cable kit**, neutral badge **Ready**, **SKU-10**, **Alex Lee**, **Nov 14, 2026**, and **CLP 420,000**. The count shows one record and the footer shows 1–1 of 1. Previous and next are disabled. The desktop field count remains six.

5. Press that record's arrow. Only `detailRecordId` changes to `33333333-3333-4333-8333-333333333333`. The dialog shows Product: Cable kit; Code: SKU-10; Stage: Ready; Owner: Alex Lee; Due date: Nov 14, 2026; Total: CLP 420,000. It uses the loaded record and the same formatter. There is no new request, configuration save, record mutation, history entry, or automation event. Closing returns focus to the record's visible arrow.

For larger results, next/previous request only missing offsets through the existing range method, keeping the same search, filters, and server order. Failed ranges remain retryable; late results cannot replace a newer view. The technical plan defines those checks alongside the Doop comparisons at all three widths.
