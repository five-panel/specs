# Shopify #279: Implementation Guide

How the pieces fit together, with concrete JSON formats to implement. Read alongside the [spec](279-create-integration-shopify.md), [designs](279-shopify-designs/README.md), and [detailed technical plan](../docs/superpowers/plans/2026-11-11-shopify-integration.md).

## Reuse and focused changes

Reuse the automation runner, Graphile queue, scheduling, protected credentials, and `DataService` for business records.

| Existing part | Change |
| --- | --- |
| Configuration and credentials | Support connections with their own credentials and saved actions. |
| Automation engine | Add request, mapping, batch-save, and enqueue operations; allow controlled batch continuation. |
| Record services | Protect the selected external-ID fields while allowing integration writes. |
| History and settings | Add connection activity and the approved setup screens. |

**Code impact: Large**, mainly from new functionality. Keep existing-service changes focused and preserve current integrations.

## New pieces and stored data

Shared code handles HTTP/GraphQL requests, field mapping, batch imports, and ownership rules. Shopify-specific code handles app authorization, signed notifications, and commerce rules. Users add saved request actions without registering new code.

Extend the existing configuration and credential models. Keep addresses, actions, mappings, ownership, and current progress together in JSON; keep credentials encrypted separately. External IDs live in the selected business-record fields. History reuses saved automation runs.

Duplicate protection needs a stored business-operation identity, saved together with its applied change. The technical plan currently proposes an `operation_effects` table for that state. **A new table is an implementation choice, not a feature requirement.** Run history alone does not identify the same sale arriving through different events.

## Runtime flow

1. Connect the current business, then select models, fields, locations, and each field's main system.
2. Import Shopify's starting data through **read → save → enqueue**. Each batch has its own run; steps keep their own inputs, outputs, errors, and attempts. Safe retries preserve completed steps.
3. Catch up changes received during import, review the result, then explicitly activate outgoing sync. Historical orders do not replay stock or money actions.
4. Local changes, Shopify notifications, and scheduled checks start flows. Restore the selected main system's latest values. Apply sales/restocks once; when importing Shopify-owned stock, do not deduct the order again.
5. Run confirmed commerce actions with saved results. Verify uncertain outcomes before another attempt. Pausing stops automatic outgoing sync; disconnecting blocks new requests.

## Concrete JSON example

This example imports three products in batches of two. The IDs and product values are sample data; the JSON structures are the exact contracts to implement. The new operations are not implemented yet.

**Saved automation**, using the existing `POST /automations` body format. Its returned ID in this example is `22222222-2222-4222-8222-222222222222`:

```json
{
  "name": "Import Shopify products batch",
  "enabled": true,
  "definition": {
    "trigger": { "eventType": "automation.manual" },
    "steps": [
      {
        "name": "read_batch",
        "operation": "integration.request",
        "input": {
          "connectionId": "11111111-1111-4111-8111-111111111111",
          "action": "list_products",
          "arguments": {
            "cursor": "{{ event.data.cursor }}",
            "limit": "{{ event.data.limit }}"
          }
        }
      },
      {
        "name": "save_batch",
        "operation": "import.save_batch",
        "input": {
          "importId": "{{ event.data.importId }}",
          "resource": "products",
          "batchNumber": "{{ event.data.batchNumber }}",
          "cursor": "{{ event.data.cursor }}",
          "batch": "{{ steps.read_batch.output.data }}"
        },
        "retry": {
          "maxAttempts": 3,
          "delaySeconds": 5,
          "retryableErrors": ["temporary_database_error"]
        }
      },
      {
        "name": "queue_next_execution",
        "operation": "automation.enqueue",
        "input": {
          "request": "{{ steps.save_batch.output.nextExecution }}"
        }
      }
    ]
  }
}
```

**First run input:** the import service creates an event with this `data`, and groups its runs under `operationId = importId`:

```json
{
  "importId": "33333333-3333-4333-8333-333333333333",
  "cursor": null,
  "limit": 2,
  "batchNumber": 1
}
```

**`read_batch` output:** the saved `list_products` action converts Shopify's GraphQL response into this shared batch format:

```json
{
  "data": {
    "items": [
      { "id": "gid://shopify/Product/1001", "title": "Trail Jacket" },
      { "id": "gid://shopify/Product/1002", "title": "Trail Bottle" }
    ],
    "nextCursor": "eyJsYXN0X2lkIjoxMDAyfQ==",
    "hasMore": true,
    "total": 3
  }
}
```

**Written record values:** the saved mapping selects the Products model and maps `id → shopify_id`, `title → name`. `save_batch` receives the full `data` object above, then writes each product through `DataService`. The first product's values are:

```json
{
  "shopify_id": "gid://shopify/Product/1001",
  "name": "Trail Jacket"
}
```

The chosen `shopify_id` field identifies an existing linked record on a later import; the integration updates that record instead of creating another one.

**`save_batch` output**, after both records and progress are committed:

```json
{
  "saved": 2,
  "processed": 2,
  "nextExecution": {
    "automationId": "22222222-2222-4222-8222-222222222222",
    "data": {
      "importId": "33333333-3333-4333-8333-333333333333",
      "cursor": "eyJsYXN0X2lkIjoxMDAyfQ==",
      "limit": 2,
      "batchNumber": 2
    }
  }
}
```

**`queue_next_execution` output:** its input `request` is exactly the `nextExecution` object above. It queues batch 2 as a new run of the same automation:

```json
{
  "queued": true,
  "eventId": "77777777-7777-4777-8777-777777777777"
}
```

The next run receives the same import ID, the returned cursor, and batch number 2. It reads the remaining product. Later work can target another selected model or catch-up. When no follow-up remains, `nextExecution` is `null`; enqueue returns `{"queued": false, "eventId": null}`.

## Delivery order

Build shared connections/actions, Shopify authorization, and batch import first. Add live sync and commerce actions, then connect the screens and history. Verify recovery, duplicate stock notifications, and uncertain payments. The detailed plan holds the exact files and test tasks.
