# Issue #288: Short implementation guide

Source issue: https://github.com/benitogonzalezh/five-panel/issues/288

[Spec](288-strengthen-shared-validation.md) · [Technical implementation plan](288-strengthen-shared-validation-implementation-plan.md)

## What changes

The application already checks record values. We will bring the duplicated checks together and fill the gaps. `DataService` will use those shared checks for every record save. Model defaults and form-template presets will use them too.

A number stays a number, `false` stays `false`, and a date must be a real day written as `YYYY-MM-DD`. JSON fields accept objects or arrays, including the JSON text used by the existing API. The existing required-field, visibility, choice, relation, uniqueness, and paired-field rules remain.

**`null` means no value.** Sending it clears an optional field. Leaving a field out of an edit keeps its saved value. Ordinary empty text stays `""`; we do not turn every empty-looking value into `null`.

## How the pieces fit

The shared field checks extend `backend/src/fields/fieldTypes.ts`, with a small new helper file for calendar days, JSON values, and relation IDs. Existing services still check access and coordinate the save; repositories still save the record, its history, and queued work together.

The existing form prepares values for `/graphql`. The backend does the final check. Errors identify the field, and the form keeps the user's work so they can correct and retry.

Records stay in `custom_data.data`. Model defaults stay in `models.fields`, and presets stay in `templates.config`. Original imported files and their saved configurations stay where they are. No new tables are needed.

Imports still call `DataService`. Their date conversion must preserve the calendar day and also keep the values used by existing matching-key expressions. The technical plan shows both values through a complete import example. Date filters compare calendar days; system timestamps and the separate report filter path keep their existing behavior.

## Concrete flow example

This is planned behavior, not an already implemented feature. The IDs and values below are safe examples. The request and response structures follow the existing API.

Assume the authenticated user belongs to company `33333333-3333-4333-8333-333333333333`. Its **Deliveries** table uses model `delivery`, collection `deliveries`, and these fields: required text `reference`, optional date `day`, optional number `quantity` with default `0`, and optional checkbox `confirmed` with default `false`.

1. On the Deliveries table, the user clicks **New**, enters reference **DEL-001**, selects **11 September 2026**, and leaves quantity at **0** and the checkbox unchecked. The form prepares:

   ```json
   {
     "reference": "DEL-001",
     "day": "2026-09-11",
     "quantity": 0,
     "confirmed": false
   }
   ```

2. `ModelForm.vue` passes the prepared values to the existing data store, which sends `POST /graphql`. This is the complete equivalent request using variables; the current form may send the same input inline:

   ```json
   {
     "query": "mutation CreateDelivery($input: createDeliveryInput!) { createDelivery(input: $input) }",
     "variables": {
       "input": {
         "reference": "DEL-001",
         "day": "2026-09-11",
         "quantity": 0,
         "confirmed": false
       }
     }
   }
   ```

3. The resolver calls `DataService.create` with the model ID, input, authenticated user, and server-created history context. The service loads and locks the company's current model, applies eligible missing defaults, then checks the complete values. The date passes; `0` and `false` are present values. The repository saves record `22222222-2222-4222-8222-222222222222`, owned by that company and model. Its complete `data` value is:

   ```json
   {
     "reference": "DEL-001",
     "day": "2026-09-11",
     "quantity": 0,
     "confirmed": false
   }
   ```

   The existing transaction also saves the creation history and queues its dispatch. For an example event ID `77777777-7777-4777-8777-777777777777`, the existing `dispatch-automation-event` worker receives:

   ```json
   {
     "tenantId": "33333333-3333-4333-8333-333333333333",
     "id": "77777777-7777-4777-8777-777777777777"
   }
   ```

   Assume no automation matches this example event. Dispatch finishes without starting a run. This issue adds no new worker definition or payload.

4. The mutation returns:

   ```json
   {
     "data": {
       "createDelivery": "22222222-2222-4222-8222-222222222222"
     }
   }
   ```

   The form closes and refreshes the table through its existing collection query. Reopening **DEL-001** loads those four values and displays **11 September**, **0**, and an unchecked checkbox. The day does not move when the browser timezone changes.

5. The user clears the optional date and saves. The form sends only that changed field:

   ```json
   {
     "query": "mutation ClearDeliveryDay($id: ID!, $input: updateDeliveryInput!) { updateDelivery(id: $id, input: $input) }",
     "variables": {
       "id": "22222222-2222-4222-8222-222222222222",
       "input": {
         "day": null
       }
     }
   }
   ```

   The service locks the record, merges the change, and saves this complete `data` value:

   ```json
   {
     "reference": "DEL-001",
     "day": null,
     "quantity": 0,
     "confirmed": false
   }
   ```

   The response is:

   ```json
   {
     "data": {
       "updateDelivery": "22222222-2222-4222-8222-222222222222"
     }
   }
   ```

   After refresh and reopen, the date is empty. The reference, quantity, and checkbox are unchanged. An edit containing only `"quantity": 2` would leave `day` exactly as saved; it would not apply a date default.

## Older values and failed saves

An older saved date such as `2026-09-11T00:00:00Z` stays exactly that way when another field changes. If the user selects a different date, the new value must be a valid day string. An existing saved timestamp default or preset can supply its written day for a new record without rewriting that setting.

A new `2026-02-30` or a newly supplied date timestamp fails before the record is saved. The error identifies `day`; the form stays open with the draft. Correcting it allows a normal retry. Clearing a required field still fails, and both members of a required pair must remain complete.
