# Issue #271: Generic companion fields and shared record validation

Source issue: https://github.com/benitogonzalezh/five-panel/issues/271

## Problem

Some field types need a second value to be meaningful. Money needs an amount and its own currency; a measurement needs a value and unit. Five Panel currently makes users create those fields separately and cannot enforce that they remain a complete pair.

The solution must describe a general field relationship. Money is one use of it. A meaningful field `type` should select code that creates its companion and validates the pair, while records continue to contain separate scalar values.

## Current Behavior

- Backend field definitions contain `id`, `displayName`, `type`, `uiType`, `required`, `unique`, optional defaults, and conditional visibility. Select and relation definitions add their own settings. There is no companion link.
- Backend types currently include string, number, date, boolean, and object. The editor exposes text, text area, number, date, boolean, select, and relation choices. Some existing representations differ between frontend and backend, including date and checkbox fields.
- Model fields already persist as JSON. Model creation/update saves the definition and rebuilds its SQL view in separate steps without one enclosing transaction.
- GraphQL value/filter types, SQL view casts, and record type checks each inspect `field.type`. Adding a semantic type without updating these consumers would fall back to text or skip a numeric check.
- `DataService.create` applies defaults and checks required values, types, choices, and relations. `update` merges a patch but does not repeat the complete set of create checks. Imports call these same service methods.
- Model editing in the inspected main checkout has an unfinished save branch. Record creation closes the form before an asynchronous save succeeds, and table edits submit one field at a time. PR [#257](https://github.com/five-panel/five-panel/pull/257), for issue #256, separately introduces full record editing and preserves failed drafts; implementation must account for its merge state.
- The product has no tenant data requiring a rollout migration for this change. Existing source definitions and tests still need to work; no tenant backfill or historical-data repair is required.

## Desired Behavior

### Field definitions and code-owned types

Keep `type` as the field's meaning and `uiType` as the input component. Keep the mapping to programming, GraphQL, filter, and SQL types in code. Do not add a persisted `primitiveType` attribute.

Add one optional property to a field:

```ts
companion?: { fieldId: string }
```

Both members store a reciprocal link. The object contains only `fieldId`: no currency-specific key, role, owner, generated flag, or copy of the rules. A field has at most one companion, in the same model. Existing field metadata remains alongside the link.

Example saved Money definition:

```json
[
  {
    "id": "price",
    "displayName": "Price",
    "type": "money",
    "uiType": "number",
    "required": false,
    "unique": false,
    "companion": { "fieldId": "priceCurrency" }
  },
  {
    "id": "priceCurrency",
    "displayName": "Price currency",
    "type": "string",
    "uiType": "select",
    "required": false,
    "unique": false,
    "options": [
      { "label": "CLP", "value": "CLP" },
      { "label": "USD", "value": "USD" }
    ],
    "companion": { "fieldId": "price" }
  }
]
```

The record is still `{ "price": 1000, "priceCurrency": "CLP" }`. A separate `cost` field has its own companion and can use USD in that same record.

A small registry in code supplies each supported type's value mapping, compatible/default UI, companion recipe where applicable, and supported validation handlers. The backend owns this definition; the editor obtains its preview from the same definitions, through a shared module or a serialized backend catalog. Do not maintain separate handwritten recipes in the two applications.

The first automatic types are:

| Type | Main field | Generated companion | Initial selector choices |
| --- | --- | --- | --- |
| `money` | Numeric amount, `uiType: number` | `<id>Currency`, `type: string`, `uiType: select` | CLP, USD |
| `measurement` | Numeric value, `uiType: number` | `<id>Unit`, `type: string`, `uiType: select` | kg, g |

The selector remains a real, editable field definition. Its saved options determine the accepted values; administrators can configure the currency or unit choices through the existing select settings. For example, a volume measurement can use L and mL. Initial choices are conveniences, not a conversion service or a global unit/currency catalog. Neither recipe supplies a record-value default. A generated companion starts optional and non-unique; the pair's required behavior is defined below.

`money` and `measurement` both map to a numeric programming value, GraphQL `Float`, `NumberFilter`, and SQL `numeric`. Their selector companions retain string values, string filters, and text columns. Keep current supported mappings for ordinary fields, including both existing date representations and relation handling.

Every reciprocal companion pair selects the generic `completeTogether` rule. The same rule also works for a complete pair of ordinary scalar fields supplied through the model API, such as two numeric coordinates. The registry adds type-specific checks without making the pair validator depend on Money. Arbitrary rules are not stored inside `companion`.

### Model creation, updates, and deletion

1. Selecting Money or Measurement in the editor immediately previews both fields. Before their first save, users can choose the primary ID and configure labels, options, required settings, and supported defaults. The preview tracks draft ID changes and shows a collision rather than silently selecting a new ID.
2. On a model create, or when adding a new field to an existing model, an unlinked Money/Measurement field causes the backend to generate exactly one companion with the suffix above and both reciprocal links. A complete explicitly supplied pair is checked and retained without another generation pass. An explicit link to a missing field is an error, not a request to invent that field.
3. Generated selector fields do not trigger more generation. An ordinary unlinked number field, such as quantity, stays independent. A caller can supply a complete new pair with custom IDs; once saved, the explicit links are authoritative, not the suffix.
4. On an update, compare with the saved definition before generating anything. Existing field IDs, types, and companion links are immutable. This also prevents attaching a new companion to an already saved unpaired field. Reordering fields and changing labels do not alter identity. Compatible existing settings, such as options, required flags, and defaults, may be edited; revalidate the resulting definition without rewriting record values.
5. Missing or altered links on a saved pair fail. Never recreate a removed saved companion automatically. Reject duplicate IDs, self-links, cross-model/dangling links, nonreciprocal links, shared companions, unsupported types, and incompatible companion definitions. Money/Measurement require a string/select companion with valid, nonempty, distinct options; their main input must remain compatible with a number.
6. Check field names against supported GraphQL names, reserved fields, and generated SQL column aliases. Two different IDs that produce the same SQL alias are also a collision. Report both conflicting IDs and do not reuse or overwrite an existing field.
7. Save the completed metadata and generated database view in one transaction. A failed view creation/replacement leaves the previous definition and view intact, or leaves neither behind on first creation. Invalidate the schema cache only after commit and return the completed definition to the caller.
8. In an unsaved draft, Remove pair removes both draft members. For saved pairs, Remove pair is allowed only when the model has no records, including records where both values happen to be empty. Removing just one member is rejected. Model definition editing never deletes or clears record values. The empty-model check must still hold when removal commits; an accepted concurrent record creation must not defeat it. Existing deletion behavior for unpaired fields and whole models is otherwise unchanged.

Definition validation errors use the existing model API error channel with an appropriate client-error status, a stable error code, and the relevant field IDs/definition paths. They must not appear only as an unexplained server error.

### Shared record validation

Use a shared backend validation path for record creation and updates. Keep definition validation separate from record-value validation. Reuse the existing required, type, choice, and relation checks, and add `completeTogether` as one small rule handler. This is an extensible set of supported handlers, not a promise to implement every conceivable validation rule.

For pair completeness, missing/undefined, `null`, and the exact empty string `''` are empty. `0` and `false` are present. Do not trim arbitrary strings automatically. Normalize explicit cleared paired values to `null` before persistence, so an empty string does not reach a numeric SQL cast. This does not make a string value valid input to a GraphQL numeric argument.

If either member has `required: true`, both values are required. Otherwise, both may be empty or both must be present. Validate each present value against its resolved type and saved choices. Numeric values must be finite numbers, including zero; a numeric string is not a number. Never manufacture a zero for an unknown amount.

For an update, remove undefined patch entries, load the stored record, merge supplied values, and validate the final result before writing. Omitted keys preserve saved values; explicit `null`/`''` clears them. A partial update is valid when the saved counterpart completes the pair. Apply supported required/type/choice checks on updates as well as creation, including unpaired fields under their existing visibility semantics. Preserve existing relation rules; unchanged valid relation IDs need not trigger redundant lookups.

A valid unchanged update still returns the existing no-change result and performs no write, timestamp change, or import updated-count increment. The no-change shortcut must not bypass supported value/pair checks. Validation failures leave the saved record unchanged and name the affected field or both members of a pair.

Pair validation is independent of field visibility and form layouts. Preserve the current visibility behavior for unpaired fields. Every supported writer, including generated GraphQL mutations, import creates/upserts, and scripts using `DataService`, goes through the same rules. Use the current server-owned model definition at the write boundary: a write begun after a completed model change cannot bypass its rules by supplying an older definition. This requires no record scan or persisted definition-version system.

### Defaults and imports

- On creation, a paired model default may complete a pair only when one incoming member is already nonempty and the other key is omitted. A configured default is checked against its field type/options before use.
- If both incoming members are empty, suppress paired model defaults. An optional pair remains empty; a required pair fails. This is true even when both members have configured defaults.
- Explicit `null` or `''` is a deliberate clear and is not replaced. No model defaults run during record updates. An explicit configured numeric default may be used under the same rule; no implicit amount default is added.
- Paired default handling is independent of field visibility. Unpaired defaults retain their existing behavior.
- Existing import-column defaults remain preprocessing. Their results are incoming values for the shared record service. A lone import-column default can therefore produce an incomplete-pair error; it must never result in a half-filled saved record. Mapped blank cells represented as `null` are explicit empties; an unmapped/omitted key can receive a model default on creation.
- Form initialization must not eagerly turn a lone model currency/unit default into a supplied value for an untouched optional pair. The browser follows the same pair default semantics. Template create presets, when supported, are incoming values like import defaults and are validated accordingly.

Examples below assume an optional amount/currency pair and an explicitly configured USD currency default where mentioned:

| Operation | Input or patch | Expected result |
| --- | --- | --- |
| Create, with currency default | Neither value supplied | Both stay empty |
| Create, with currency default | `{ "price": 0 }` | Save zero and USD |
| Create, without currency default | `{ "price": 0 }` | Reject missing currency |
| Create, with currency default | `{ "price": 12, "priceCurrency": null }` | Reject; do not replace the explicit clear |
| Create | `{ "price": 12, "priceCurrency": "EUR" }`, with only CLP/USD configured | Reject unsupported choice |
| Update stored 12/USD | `{ "price": 20 }` | Save 20/USD |
| Update stored 12/USD | `{ "price": null }` | Reject; keep 12/USD |
| Update stored 12/USD | Both values `null` | Clear both |
| Create required pair | Both empty, even when hidden | Reject |
| Import create | Only a column currency default yields USD | Reject incomplete pair |

### Editor and record forms

The model editor exposes Money and Measurement, previews both fields, preserves both links through editing, and reloads the canonical backend definition after a successful save. Saved core settings are disabled in the UI and enforced by the backend. All supported default inputs submit values of the correct type, including numeric zero.

Record forms keep both editable values in one draft and submit one mutation. A user can fill a previously empty optional pair or clear an existing pair without first saving an invalid intermediate state. A one-field update remains valid when it can use a stored counterpart. Do not require the table view to display both columns.

Load counterpart values required for validation even if they are absent from the displayed table columns or chosen form fields. Preserve hidden/read-only saved values and omit unchanged hidden values from update payloads. Loading a value does not make it editable. Continue to honor visibility and template presentation rules, while still enforcing pair validation on the backend. If the current form hides a missing value that is needed to save, identify that field and explain that its visibility condition or form configuration needs correction; do not silently clear data, skip the rule, or expose a hidden input.

Keep the form open and retain its draft while saving and after a rejected save. Display field-specific errors, including a visible form-level message for an error on a hidden field. Close and refresh only after success. Disable duplicate submissions while a request is pending. New labels and errors follow the existing English/Spanish translation patterns and remain readable by keyboard and assistive-technology users.

Use the full record form from #256/PR #257 if it is available at implementation time. If it has not landed, provide the smallest shared-form change needed for one multi-field save and failed-draft retention; do not build a second permanent pair editor or expand this issue into a form-template redesign.

## Scope

- The generic reciprocal companion link and a small code-owned type/creation/rule registry.
- Automatic Money and Measurement creation, with real scalar sibling fields and configurable selector choices.
- Central model-definition checks, immutable saved core settings, safe pair deletion on empty models, and transactional definition/view persistence.
- Shared supported value checks for creates and merged updates, including generic pair completeness, precise defaults, and API/import error reporting.
- Type resolution in validation, GraphQL values/inputs/filters, generated SQL views, and frontend controls.
- Model-editor save/preview support and the record-form changes necessary to save a pair in one operation and recover from errors.
- Focused backend, frontend, database, and browser verification of the behavior described here.

## Out Of Scope

- Persisted primitive types, nested money/measurement record objects, composite query values, or more than one companion per field.
- Type conversion, saved field-ID changes, linking/relinking existing fields, or detaching a saved pair into independent fields.
- Deleting saved pairs from a model containing records or silently deleting companion data.
- Old-record migrations, backfills, repair jobs, scans before tightening rules, and automatic tenant adoption. Existing dated tenant scripts remain unchanged.
- Date-range, location, and other new automatic types beyond Money and Measurement in this issue.
- Currency/unit conversion, exchange rates, arithmetic across different currencies/units, or a new global catalog service.
- Arbitrary expressions, workflow automation, a user-authored rule language, or a large new rule catalog. Uniqueness enforcement, range/pattern rules, date ordering, and conditional-required rules are not added merely because attributes or examples mention them.
- Changes to template permissions, custom layout authoring, or unrelated record-editor behavior.

## Acceptance Criteria

1. Creating an unlinked Money or Measurement field through the model API or editor persists exactly two compatible fields with reciprocal `companion.fieldId` links. An ordinary number creates no companion. A second save and schema reload preserve the same IDs and links without adding fields.
2. Complete explicit pairs with custom IDs round-trip unchanged. A coordinate pair of ordinary numbers can use the same completeness handler, and the Measurement preset demonstrates automatic creation without Money-specific branches.
3. Price and cost can have separate companions and different valid currencies in one record. Amount and unit/currency remain separate scalar values in the API, stored record, exports, and SQL view.
4. Money and Measurement produce numeric GraphQL input/output, numeric filter inputs, and SQL numeric columns. Queries sort 2 before 10 and numeric comparisons work. Existing ordinary field mappings, date forms, and relations remain usable.
5. Invalid/self/dangling/nonreciprocal/shared links, incompatible types/UI, empty or duplicate selector options, unsupported types, and field/SQL-alias/reserved-name collisions fail before anything is persisted. Errors identify the affected fields.
6. Changes to a saved field's ID, type, or companion link are rejected. Label changes and valid supported-setting changes preserve the pair and stored record values. Dropping a saved companion fails rather than causing automatic recreation.
7. Removing both draft fields succeeds. Removing a saved pair succeeds only on a model with no records; removing one member or removing a pair from a populated model fails without changing definitions or records. Concurrent writes cannot bypass that restriction.
8. Injecting a database-view failure into create/update leaves no partial new model or changed saved definition/view. Successful commits trigger schema reload with the complete pair.
9. Optional pairs allow both empty or both present. Required on either member requires both. Zero is accepted with a valid counterpart; wrong types, nonfinite numbers in direct service calls, and choices outside the saved list are rejected.
10. The defaults and explicit-clear examples above hold through the shared service. An untouched optional form does not submit a lone selector model default. Import preprocessing never allows a half-filled pair to be saved.
11. Updates validate merged stored/patch values. Updating only a main value can retain its counterpart; clearing only one member fails; clearing both optional members succeeds. Undefined patch keys leave values unchanged.
12. Required/type/choice validation also runs for supported unpaired fields on updates, retaining their visibility semantics. A valid no-op stays a no-op, including import counts and unchanged relation lookup behavior.
13. Direct GraphQL writes, import creates/upserts, and service/script calls reject the same invalid final values, identify the relevant field IDs, and leave prior records unchanged. Hidden fields and omitted layout fields do not bypass pair checks.
14. The editor previews and saves both automatic types, reopens their completed definitions, locks saved core settings, and provides typed default inputs. Model definition failures keep the editor draft available for correction.
15. The record form can fill an empty pair, change both values, and clear both values with one save. It loads the stored counterpart when a table/template omits it, preserves hidden/read-only saved values, and reports when a missing hidden counterpart prevents saving.
16. A delayed or failed create/update keeps the form open, retains user input, prevents duplicate submissions, and shows actionable field/form errors. Correcting the values and retrying succeeds. New UI text is available in English and Spanish.
17. After a model required/option setting change commits, a subsequent record write checks the current definition even if its caller supplies an older model snapshot. A value newly disallowed by that change is rejected without a write.

## Implementation Notes

- Follow `RULES.md`: use small, readable functions and existing service/repository patterns. The registry describes today's types and rules; it does not need a dependency graph engine, persisted registry version, or executable user configuration.
- Extend `backend/src/types/schema.ts`, `frontend/src/types/metadata.ts`, and `frontend/src/types/ModelManagement.ts`. The existing fields JSON persistence can store the new types and links without another database column.
- Put definition completion/validation at the common `ModelService.create/update` boundary, including callers outside REST. Compare saved definitions before expansion. Validate generated names through the same path as explicit fields. Use the existing tenant/model authorization boundaries.
- `backend/src/repositories/ModelRepository.ts` and `backend/src/services/ModelService.ts` need transaction-aware persistence/view work. Keep the deletion guard in the same protected operation as deletion; reuse existing record-count behavior where possible and coordinate it with record creation. This is an ordinary operation constraint, not a legacy-data scan.
- `backend/src/index.ts` already passes field arrays through model create/update. Preserve the nested metadata and translate definition errors into actionable client errors. Existing model API responses return the completed fields; no new record API envelope is needed.
- Use resolved type information in `backend/src/schema.ts`, `backend/src/types/filters.ts`, `ModelRepository`, and `DataService`. Do not resolve every date to an undifferentiated string filter: existing date wire/filter behavior must remain explicit. Preserve the relation field's current ID-based storage behavior.
- Refactor only the necessary create/update logic in `backend/src/services/DataService.ts`. Separate incoming normalization, create-only defaults, merged-value validation, no-change detection, and persistence. Deduplicate pair checks by stable field IDs so reciprocal links do not produce duplicate errors. Keep the existing special handling for record metadata such as `externalId` and import keys.
- Import integration is at `backend/src/services/import/ImportOrchestrator.ts`. Inspect its column-default and blank-cell behavior as well as the record-service path. Preserve per-row errors and valid no-op update counts; do not introduce a new import format or source-tracking mechanism.
- Frontend integration includes `FieldEditor.vue`, `ModelFormDialog.vue`, `ModelForm.vue`, `ModelManagementView.vue`, `ModelView.vue`, and their stores/services. Pair dependency loading extends the existing visibility-dependency approach without changing the table's selected columns. Recheck #257 before changing form save/loading code so the two features share one implementation.
- Prior research supports the shape without requiring a new library. [JSON Schema conditional dependencies](https://json-schema.org/understanding-json-schema/reference/conditionals) illustrate reciprocal field dependencies, but property presence alone does not handle Five Panel's empty-value/update semantics. [Django Money](https://django-money.readthedocs.io/en/latest/) illustrates separate amount/currency values and configurable choices; Five Panel keeps its own scalar API and uses the same generic pair behavior for measurements.

## Test Expectations

- Backend definition tests under `backend/test/`: both recipes, ordinary number, explicit custom-ID pairs, ordinary non-money pairs, create/update round-trips, repeated saves, immutable core settings, all invalid-link/type/name cases, supported-setting edits, and pair deletion on empty/populated models. Cover the delete/create race with a real database test.
- Backend record tests: required on either side; both empty; one missing; zero; wrong types; invalid options; defaults with omitted versus explicit empty keys; two defaults on an untouched pair; imported column defaults; partial updates; both clears; undefined keys; hidden fields; separate price/cost currencies; and unchanged valid records. Exercise the same rule with Measurement and an ordinary numeric coordinate pair.
- Integration tests: actual generated GraphQL create/update/query/filter inputs and outputs; actual numeric SQL casts/sorting; schema reload; model API error responses; injected view failures and transaction rollback; import create/upsert/error/no-change paths; tenant isolation for model operations; and a write using an older model snapshot after a completed required/options change.
- Preserve and extend the relevant existing suites: `ModelService.test.ts`, `DataService.test.ts`, `DataServiceVisibility.test.ts`, `DataServiceUpdateNoOp.test.ts`, import suites, and frontend `fieldVisibility.test.ts`.
- Frontend tests under `frontend/test/`: preview without duplicate generation, link preservation on reopen, typed defaults, locked saved settings, real model-update invocation, pair draft submission, dependency fetching, untouched optional defaults, hidden-field errors, failed-save retention, and retry after correction.
- Browser verification: create Money and Measurement through Settings, save/reopen, rename labels, observe rejected core/deletion changes, enter zero with a selected currency/unit, fill and clear an optional pair in one save, edit with the counterpart absent from displayed columns, and recover from a delayed server rejection. Exercise English/Spanish copy and a narrow screen using the existing form layout.
- Run the relevant backend/frontend test suites and builds for the implementation. This spec PR itself changes documentation only; its validation is a requirement/consistency review and Markdown diff checks, not a claim that the described feature is implemented.

## Risks

- Missing one direct `field.type` consumer could cause string sorting, incorrect GraphQL inputs, or skipped validation. Numeric schema/view integration tests are required, not just registry unit tests.
- Eager frontend or import defaults can accidentally turn an empty pair into a half-pair. Tests distinguish model defaults from incoming presets/import values and omitted keys from explicit clears.
- PR #257 overlaps the record-form changes. Check its actual merge state and reuse its multi-field save/load/error handling rather than adding a parallel editor.
- Updating options or required settings does not scan or repair saved records. Later writes must satisfy the current supported rules; an unrelated update can therefore require correcting a value affected by a setting change. This is intentional under the approved no-preflight scope.
- A form configuration can hide a required counterpart. Pair checks remain authoritative and the form must explain how the missing field blocks saving; this issue does not redesign layouts or silently override hidden/read-only settings.
- View transactions and the empty-model deletion guard must preserve existing view permissions, tenant boundaries, and successful record-write behavior. Verify actual database failure/concurrency cases.
- The single `fieldId` supports exactly one companion. A future address or other multi-part type may need a separately designed structure; do not add unused metadata now.

## Open Questions

None.
