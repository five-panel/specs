# Issue #294: External ID updates implementation guide

Read the [spec](294-dataservice-update-silently.md) for behavior and the [technical implementation plan](294-dataservice-update-silently-implementation-plan.md) for tasks and test commands. This guide describes the proposed fix; the application does not yet save external ID changes through this update path.

## What changes and what we reuse

Change the existing `useDataService` update operation in `backend/src/services/DataService.ts`. It will validate a supplied external ID, compare it with the current value, and pass it to the existing repository save. Add focused service, import, and PostgreSQL tests; no new production module is needed.

Reuse the current company checks, current-model and record locks, transaction, and complete record result. `CustomDataRepository` already stores the external ID in `custom_data.external_id`. Its shared history code already saves record changes, history, and automation dispatch together.

An omitted or `undefined` external ID stays unchanged. Null requests clearing, and strings are saved as supplied. Existing model rules and uniqueness still apply; other types are rejected. The internal record `id` remains unchanged.

## Concrete flow example

These are sample values for the proposed behavior. JSON uses ISO strings for dates; the service's returned `DataRecord` uses JavaScript `Date` values before serialization.

**1. Start with this stored repository record.** Its only model field is `name`:

```json
{
  "id": "22222222-2222-4222-8222-222222222222",
  "tenantId": "33333333-3333-4333-8333-333333333333",
  "modelId": "11111111-1111-4111-8111-111111111111",
  "data": { "name": "Example" },
  "externalId": "source-old",
  "importSourceKey": null,
  "createdBy": "44444444-4444-4444-8444-444444444444",
  "updatedBy": "44444444-4444-4444-8444-444444444444",
  "createdAt": "2026-11-12T10:00:00.000Z",
  "updatedAt": "2026-11-12T10:00:00.000Z"
}
```

**2. Call `dataService.update`.** The complete business input is:

```json
{
  "modelId": "11111111-1111-4111-8111-111111111111",
  "id": "22222222-2222-4222-8222-222222222222",
  "data": { "externalId": "source-new" }
}
```

The server supplies its authenticated `user` for the company above and passes this trusted `eventContext` alongside that business input:

```json
{
  "operationId": "55555555-5555-4555-8555-555555555555",
  "tenantId": "33333333-3333-4333-8333-333333333333",
  "source": { "type": "api" },
  "actor": { "type": "user", "id": "44444444-4444-4444-8444-444444444444" }
}
```

The service loads the current model and record within the existing transaction. It validates `source-new`, sees that it differs from `source-old`, and calls the existing scoped repository update. The repository writes `external_id = source-new`, keeps `data = {"name":"Example"}`, and updates the normal actor and time fields. In this example the saved update time is `2026-11-12T10:01:00.000Z`.

**3. Save the existing logical history event.** The shared history path emits one `custom_data.updated` event, version 3, with this `data`:

```json
{
  "before": { "name": "Example", "externalId": "source-old", "importSourceKey": null },
  "after": { "externalId": "source-new" }
}
```

The event keeps the same operation ID and actor. If its generated event ID is `66666666-6666-4666-8666-666666666666`, the existing `dispatch-automation-event` job receives:

```json
{
  "id": "66666666-6666-4666-8666-666666666666",
  "tenantId": "33333333-3333-4333-8333-333333333333"
}
```

The existing dispatcher reconstructs the final values before checking automation conditions or references: `after.externalId` is `source-new`, and `changedFields` contains `externalId`. The record, event, and queued job commit together. A failure rolls back the action.

**4. Return the saved logical record.** The serialized service result is:

```json
{
  "record": {
    "id": "22222222-2222-4222-8222-222222222222",
    "name": "Example",
    "externalId": "source-new",
    "importSourceKey": null,
    "createdAt": "2026-11-12T10:00:00.000Z",
    "updatedAt": "2026-11-12T10:01:00.000Z"
  },
  "changed": true
}
```

Repeating the call returns this same record with `changed: false`; it changes no timestamp and adds no event or dispatch job. Sending `externalId: null` instead clears the external ID and records an explicit null in the changed-value patch.

## Imports use the same fix

If an import matches this record through `name`, its existing mapped payload already includes `externalId`. Configure `name` as the match field and the external ID as a data-only mapping. The importer calls the same service, keeps the internal record ID, and counts the external ID correction as one update. A repeated identical row counts as unchanged. Existing row failure reporting and source-snapshot rules remain in place; this fix does not make an entire import one transaction or change how snapshots treat blank external IDs.
