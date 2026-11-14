# Issue #287: Short implementation guide

[Spec](287-add-datetime-field.md) · [Technical implementation plan](287-add-datetime-field-implementation-plan.md)

Source issue: https://github.com/benitogonzalezh/five-panel/issues/287

## How the pieces fit

A datetime value will stay one string, such as `2026-09-11T10:00:00-03:00`. The last part says that the local clock was three hours behind UTC. It names the same moment as `2026-09-11T13:00:00Z`.

The existing model editor gains **Date and time**. New shared code checks real dates, times, and known offsets. One shared control captures the date, time, and visible offset in forms, field defaults, and filters. For a new value, it suggests the browser’s offset for the chosen date/time; users can change it. Existing values, defaults, and template presets keep their own offset.

The record service keeps its access checks, required-field checks, and automatic history. The repository saves the original string in existing JSON storage. Reads return that text. Queries compare its moment; they do not sort these strings as ordinary text. Reports use the same display rules and explicit time comparisons. Imports and automation record operations pass through the same service validation.

## Concrete flow example

This shows the planned create-and-reopen flow, not an already working feature. The sample tenant is `11111111-1111-4111-8111-111111111111`; its existing model is `22222222-2222-4222-8222-222222222222`, with slug `order` and plural slug `orders`. An administrator has added this field through Settings → Models:

```json
{
  "id": "orderedAt",
  "displayName": "Ordered at",
  "type": "datetime",
  "uiType": "datetime",
  "required": true,
  "unique": false
}
```

**1. Enter the event.** Open the orders table and its create-record form, using the standard form or a template. Enter date `2026-09-11`, time `10:00:00`, and offset `-03:00` in **Ordered at**. Choosing the offset explicitly replaces the browser suggestion. The shared control produces `2026-09-11T10:00:00-03:00`. An incomplete date/time/offset blocks Save and keeps the draft.

**2. Save.** `ModelForm.vue` builds the record payload, and the existing data store sends this logical request to `POST /graphql` using the current signed-in company context:

```json
{
  "query": "mutation CreateOrder($input: createOrderInput!) { createOrder(input: $input) }",
  "variables": {
    "input": {
      "orderedAt": "2026-09-11T10:00:00-03:00"
    }
  }
}
```

The new `OffsetDateTime` scalar validates the string and leaves it unchanged. The existing mutation calls `useDataService.create`, which checks access, resolves the model, validates the final values, and calls the repository. Its existing transaction writes the record and its change history together.

For created record `33333333-3333-4333-8333-333333333333`, this is the complete `custom_data.data` JSON value:

```json
{
  "orderedAt": "2026-09-11T10:00:00-03:00"
}
```

The row’s model and tenant IDs are the IDs above. Its ID, authors, and system timestamps remain separate existing columns. No extra UTC or offset field is saved. The mutation returns an ID, as it does today:

```json
{
  "data": {
    "createOrder": "33333333-3333-4333-8333-333333333333"
  }
}
```

**3. Reopen.** The usual record read uses this logical `/graphql` request:

```json
{
  "query": "query ReadOrder($id: ID!) { order(where: { id: { eq: $id } }) { id orderedAt } }",
  "variables": {
    "id": "33333333-3333-4333-8333-333333333333"
  }
}
```

The existing company check and record loader return the stored value. The selected response is complete:

```json
{
  "data": {
    "order": {
      "id": "33333333-3333-4333-8333-333333333333",
      "orderedAt": "2026-09-11T10:00:00-03:00"
    }
  }
}
```

The form restores date `2026-09-11`, time `10:00:00`, and offset `-03:00`, even in a browser set to Europe/Berlin. A table or report displays `2026-09-11 10:00:00 UTC-03:00`. Saving without changes preserves the exact string and creates no extra history. Changing another field also leaves it intact.

**4. Compare the moment.** With only this sample record present, this query also finds it:

```json
{
  "query": "query { orders(first: 10, where: { orderedAt: { eq: \"2026-09-11T13:00:00Z\" } }, orderBy: { orderedAt: ASC }) { nodes { id orderedAt } totalCount } }",
  "variables": {}
}
```

The query builder compares timestamps while selecting the original text from the SQL view. The response is:

```json
{
  "data": {
    "orders": {
      "nodes": [
        {
          "id": "33333333-3333-4333-8333-333333333333",
          "orderedAt": "2026-09-11T10:00:00-03:00"
        }
      ],
      "totalCount": 1
    }
  }
}
```

## Other paths

Template presets use this same string. Invalid input shows a clear error and retains the draft. Imports must supply an explicit offset; they cannot recover one from a date-only value. Table/report exports retain the original time and offset. Report ranges use separate start and end inputs and compare moments in SQL. Existing date-only fields, system timestamps, company access, queues, and file storage keep their current roles.
