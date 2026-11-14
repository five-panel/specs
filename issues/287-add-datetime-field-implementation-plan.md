# Issue #287: Technical implementation plan

[Spec](287-add-datetime-field.md) · [Short implementation guide](287-add-datetime-field-implementation.md)

Source issue: https://github.com/benitogonzalezh/five-panel/issues/287

## Goal and source basis

Implement the approved `datetime` field across the existing model, record, import, and report paths. This is a plan; the feature and the new checks below are not implemented by this PR.

Source review covers the complete tracked application source and relevant tests, updated through application main `2e8e85acd9fc8fe92a0e3db66aa5c58d3ef4f6a9`. Use the current `useModelService`, `useDataService`, and repository factories. Keep routes and operation handlers thin, validation and access decisions in services, and persistence in repositories. Follow `RULES.md` and `docs/agent/backend-standards.md`; do not refactor unrelated code or add a dependency.

The existing JSON record format already fits. Store one original string in `custom_data.data`; keep its generated SQL view column as `text`. PostgreSQL converts `timestamptz` to UTC and discards the supplied offset, so use that type only in comparison and ordering expressions. This follows the [PostgreSQL datetime documentation](https://www.postgresql.org/docs/17/datatype-datetime.html). The distinction between an instant and its written offset follows [RFC 3339](https://www.rfc-editor.org/rfc/rfc3339.html).

## Contracts to implement

### Values and validation

Add `datetime` to both field type and UI type definitions. The catalog entry returned through existing `GET /models/field-types` uses `valueType: 'string'`, `graphqlType: 'OffsetDateTime'`, `filterType: 'OffsetDateTimeFilter'`, `sqlType: 'text'`, and `uiTypes: ['datetime']`.

Use this explicit first-version profile:

- `YYYY-MM-DDTHH:mm:ss`, optional decimal seconds with one to six digits, then uppercase `Z` or a signed `HH:mm` offset.
- Real Gregorian dates in years `0001`–`9999`; hours `00`–`23`, minutes and seconds `00`–`59`. No leap seconds, rollover dates, whitespace, or offset-free values.
- Known offsets from `-14:00` to `+14:00`; the minutes must be zero at either 14-hour limit. Reject RFC 3339’s unknown-offset marker `-00:00`. Accept `Z` and `+00:00` and preserve whichever was supplied.
- Preserve all accepted text, including trailing fractional zeroes. Do not trim, round, normalize to UTC, or pass values through a JavaScript `Date` at a storage/API boundary. Numbers and `Date` objects are not datetime field values.
- Missing and `null` follow the existing optional/required field rules. Clearing a form control and reading an empty import cell produce `null`; API/service datetime strings must be valid and cannot be empty. Defaults apply on create when absent; an omitted update does not reset a value. Explicit `null` does not select a default.

Proposed new pure helpers in `backend/src/fields/offsetDateTime.ts` and `frontend/src/utils/offsetDateTime.ts` share this contract and the same acceptance fixtures:

```ts
type OffsetDateTimeParts = {
  raw: string;
  date: string;
  time: string;
  offset: string;
  instantMicros: bigint;
};
function parseOffsetDateTime(value: unknown): OffsetDateTimeParts | null;
function compareOffsetDateTimes(left: string, right: string): number;
```

The parser returns `null` for invalid input. Compute the instant without losing the fourth through sixth fractional digits; pad the fraction to microseconds and apply the numeric offset. Account for JavaScript’s special handling of years below 100 if using its date arithmetic. Keep `bigint` inside helpers, never in API JSON. A formatter in the frontend helper shows original date/time parts and a visible `UTC±HH:mm` suffix (`Z` displays as `UTC+00:00`), without changing the saved string.

### GraphQL and queries

Extend the generated schema in `backend/src/schema.ts` with an `OffsetDateTime` scalar whose literal, variable, and serialization paths validate and return the original string. Keep existing `DateTime` and `DateTimeFilter` behavior for system timestamps and older date fields.

Add `OffsetDateTimeFilter` to `backend/src/types/filters.ts` and its generated definitions. Supported operations are `eq`, `ne`, `gt`, `gte`, `lt`, `lte`, `between`, and `isNull`. All non-null operands use `OffsetDateTime`; `between` requires exactly two valid bounds in chronological order and is inclusive. `isNull` remains boolean. Do not offer text operators. Validate filters in the service as well as GraphQL, because internal callers bypass GraphQL.

For the example model, slug `order`, plural slug `orders`, these existing generated operation names remain:

```graphql
scalar OffsetDateTime
input OffsetDateTimeFilter {
  eq: OffsetDateTime
  ne: OffsetDateTime
  gt: OffsetDateTime
  gte: OffsetDateTime
  lt: OffsetDateTime
  lte: OffsetDateTime
  between: [OffsetDateTime!]
  isNull: Boolean
}
```

The model’s `orderedAt` input/output uses that scalar. `createOrder(input: createOrderInput!): ID` and `updateOrder(id: ID!, input: updateOrderInput!): ID` still return an ID. Read it with `order(where: {id: {eq: id}})`, or use the existing `orders` connection. The short guide shows complete requests and responses.

`useFilterQueryBuilder` must resolve the actual field definition at every root or relation path. For datetime fields, cast both the text column and bound value to `timestamptz`; select the uncast text for output. Apply the same rule within `AND`, `OR`, and `NOT`, range filters, and nested ordering. Bind operands and use trusted model metadata for identifiers. For `SELECT DISTINCT`, include the required ordering expressions without overwriting the raw output column. Keep existing null behavior, tenant/model scoping, and the ascending record-ID tie breaker for stable paging when two values name the same instant.

Extend repository `findOne` with trusted model `fields: Field[]`, supplied by `useDataService.findOne` after resolving and checking the model. Validate datetime match values there; compare matching datetime JSON properties by instant inside the repository. Its public service input remains `{modelId, tenantId, filter}`. This covers import matching and lookup defaults without bypassing company checks. Non-datetime matching stays as it is. Reject `unique: true` for datetime fields in model validation and do not expose that option in the editor; temporal uniqueness is outside this version.

### Shared sample data

Both implementation documents use tenant `11111111-1111-4111-8111-111111111111`, model `22222222-2222-4222-8222-222222222222`, and created record `33333333-3333-4333-8333-333333333333`. These are safe example IDs. An existing `order` model has this complete field definition after configuration:

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

Its entire saved `custom_data.data` value in the example is:

```json
{
  "orderedAt": "2026-09-11T10:00:00-03:00"
}
```

The repository owns the separate row ID, model ID, tenant ID, authors, and system timestamps. No second UTC/offset property, new column, index, or data migration is required. Model fields and defaults stay in `models.fields`; template presets stay in existing configuration JSON.

## Implementation order and checks

### 1. Define the type and enforce valid writes

**Modify:** `backend/src/fields/fieldTypes.ts`, `backend/src/fields/modelFields.ts`, `backend/src/types/schema.ts`, `backend/src/services/DataService.ts`, `backend/src/utils/formTemplateValidation.ts`, `backend/src/schema.ts`, and `backend/src/types/filters.ts`. **Create:** the backend helper above. Reuse existing `useModelService` transactions, generated view recreation, schema invalidation, record locks, and automatic history.

- [ ] Add failing tests in new `backend/test/fields/offsetDateTime.test.ts`, existing `backend/test/schema.test.ts`, `backend/test/services/DataServiceInputs.test.ts`, and `backend/test/formTemplates/validation.test.ts`. Cover valid defaults/presets, scalar variables and literals, leap years, invalid dates, offset limits, missing offsets, unknown offset, too much precision, required/optional clearing, and `unique: true` rejection. Preserve `.123400`, `Z`, and `+00:00` exactly.
- [ ] Implement the helper, type definitions, catalog, scalar, and semantic checks. Apply semantic validation by `field.type`, regardless of UI choice, to model defaults, template presets, and fully merged record values. Check datetime strings before existing empty-value shortcuts so `''` cannot be stored and then fail scalar serialization. Reuse `RecordValidationError` and `INVALID_RECORD_FIELDS`/`fieldIds` for service record errors; scalar errors must also name the invalid input clearly. Add English/Spanish backend validation and import messages in `backend/src/locales/{en,es}`.
- [ ] Extend `backend/test/services/DataServiceUpdateNoOp.test.ts` and `DataServiceEventLog.test.ts`: unchanged values and unrelated edits preserve the text; a wholly unchanged save preserves timestamps and adds no history. Changing only the offset text is a real edit even for an equal instant and records one normal history change. Exercise create/update through service callers as well as GraphQL.

Expected: invalid writes fail before persistence; successful writes and their history remain in the existing transaction. Existing external-ID updates from #294 keep working.

### 2. Compare moments while returning original text

**Modify:** `backend/src/services/FilterQueryBuilder.ts`, `backend/src/services/DataService.ts`, and `backend/src/repositories/CustomDataRepository.ts`. Reuse the `text` view expression from `backend/src/repositories/ModelRepository.ts`; verify it rather than replacing it with a timestamp column.

- [ ] Add `backend/test/integration/datetimeFields.test.ts` using the existing database test setup. Write failing assertions for ID/DataLoader reads, lists, direct service reads, and SQL view reads returning identical strings; instant-based filters and `findOne`; relation filters/order; nulls; equal instants; reversed ranges; and pagination ties.
- [ ] Implement field-aware comparisons and the repository match contract above. Make invalid internal filter operands fail in the service before PostgreSQL casting.
- [ ] Verify ordering with `2026-09-11T10:00:00-03:00`, `2026-09-11T14:00:00+02:00`, and `2026-09-11T13:00:00Z`: the `+02:00` value comes first; the other two compare equally and are ordered by record ID. Also distinguish `.123456` from `.123457`. Repeat under two PostgreSQL session time zones and with a second tenant having the same model slug; no cross-company rows may appear.

Expected: comparisons use instants at microsecond precision; every output path still returns the original representation. No database-driver timestamp conversion touches the selected text.

### 3. Add model setup and one shared form control

**Modify:** `frontend/src/types/metadata.ts`, `frontend/src/types/ModelManagement.ts`, `frontend/src/components/FieldEditor.vue`, `frontend/src/components/ModelForm.vue`, `frontend/src/components/ModelFormDialog.vue`, `frontend/src/utils/recordFields.ts`, and `frontend/src/utils/formTemplates.ts`. **Create:** `frontend/src/components/DateTimeInput.vue`, `frontend/src/composables/useDateTimeInput.ts`, and the frontend pure helper above. Keep the existing model-management page/service and `frontend/src/stores/dataStore.ts` request path.

The new component accepts `modelValue: string | null`, `inputId`, `labelledBy`, `disabled`, and `readonly`. Emit `update:modelValue` with a complete valid string or cleared `null`, and `update:valid` with the current draft’s validity. The composable owns date/time/offset drafts, the original raw value, and whether the offset came from the user. Use a time input that retains seconds and up to six fractional digits, without native-control rounding. Parent forms, default editors, and filters must block submission while a draft is incomplete or invalid; a previous valid emitted value must never be silently submitted instead. Initialize validity when values/defaults change. Readonly display uses the same formatter.

- [ ] Add `frontend/test/offsetDateTime.test.ts` and `frontend/test/dateTimeInput.test.ts`; extend `recordFields.test.ts`, `formTemplates.test.ts`, and `modelFields.test.ts` before implementation. Assert the backend profile, exact unchanged payloads, clearing, invalid draft blocking, and create-only defaults/presets. No conversion to `Date` in record or template values.
- [ ] Add the catalog/editor choice and reuse the new control for field defaults and record/template forms. For a genuinely new value, derive the browser offset for the selected local date/time. Until manually overridden, recompute for date/time changes. For saved values, defaults, and presets, retain their offset; subsequent editing does not silently replace it with a browser offset. Untouched values retain their exact original seconds, fraction, and offset spelling.
- [ ] If the browser cannot represent a chosen local time without normalizing it, require an explicit offset rather than changing the entered wall time. For a repeated local hour, keep the proposed offset visible and editable. This is an offset editor, not a named-zone scheduling system.
- [ ] Add English/Spanish labels, hints, and errors in the relevant `frontend/src/locales/{en-US,es-ES}/{modelManagement,common,table,report}.json` files. Extend browser coverage in `frontend/test/browser/fieldDefaultControls.mjs` and `e2e/test/record-form-accessibility.spec.ts`: labels identify each part, keyboard input works, failed saves retain drafts, and layout works on phone/desktop in light/dark themes.

Expected: a value entered as `10:00:00` with `-03:00` reopens unchanged in a browser set to Europe/Berlin. Changing another field never normalizes it. Template visibility and readonly settings remain presentation rules; service permissions remain authoritative.

### 4. Connect table, board, and report use

**Modify:** `frontend/src/views/ModelView.vue` and `frontend/src/views/KanbanView.vue`. Resolve datetime metadata for direct and related table fields, use the shared formatter, and offer only the defined datetime filter operators with the shared input (two inputs for `between`). Preserve offset strings in saved filters. Board titles/descriptions can display datetime values; open the existing full record editor when editing one. Do not introduce datetime grouping. Keep table CSV output as the original string.

**Modify reports:** `backend/src/services/report/ReportTypes.ts`, `ReportDefinitionValidator.ts`, `ReportDatasetService.ts`; `frontend/src/types/report.ts`, `frontend/src/components/report/blocks/ReportFilterBlock.vue`, `frontend/src/components/report/reportRuntimeUtils.ts`, `reportValueFormatter.ts`, and `blocks/chartOptions.ts`.

- [ ] Add column `type: 'datetime'` and scalar filter `inputType: 'datetime'`, value `string | null`, with strict defaults/runtime validation and the shared control. A report range uses two scalar filters, such as `orderedFrom` and `orderedUntil`; do not add a second array/range contract. Preserve explicit offsets through URL encoding, including a literal `+`.
- [ ] Keep dataset SQL authored as today. Select the raw `ordered_at` text; write temporal predicates/order with explicit `::timestamptz`, including bound parameters, for example `ordered_at::timestamptz >= :orderedFrom::timestamptz`. Use the existing optional-filter SQL convention when a bound is unset. Configuration validation cannot infer time semantics from arbitrary SQL. Test a complete report definition with both bounds and chronological ordering; do not rewrite older report scripts.
- [ ] Route table, grouped table, text, chart labels/tooltips, and shared PDF renderers through datetime formatting. When chart categories come from a datetime column, compare instants rather than text, preserving the original label. Grouping still uses the existing source values; no new calendar buckets or time-zone analytics.
- [ ] Extend `frontend/test/tableFilters.test.ts`, `reportRuntime.test.ts`, `reportChartOptions.test.ts`, `reportPdfExport.test.ts`, and existing backend report validator/dataset tests. Cover exact text display/CSV, PDF-visible offsets, microseconds, bounds in different offsets, unset bounds, reload from URL, and table paging. Reuse report SQL read-only role, allowed views, parameter binding, tenant guards, timeouts, and row limits.

Expected: forms, tables, boards, and reports agree on the saved wall time and offset; report ranges/order compare instants. [Application PR #318](https://github.com/five-panel/five-panel/pull/318) separately implements report completeness/paging for [#247](https://github.com/five-panel/five-panel/issues/247); recheck compatibility with it at implementation time, without adding its dataset-limit work here.

### 5. Preserve values through imports and existing background work

**Modify:** `backend/src/services/import/types.ts`, `backend/src/services/import/operations/transforms/type.ts`, its `index.ts` registry, and `backend/src/services/import/ColumnProcessor.ts`. Reuse `RowProcessor`, `ImportOrchestrator`, existing providers, file storage, and queued import execution.

- [ ] Add column type `datetime`, transform `toDateTime`, and `castType` support for that target. `toDateTime` returns accepted strings unchanged, retains ordinary empty handling, and returns a column error for offset-free text or numeric spreadsheet dates. It does not infer the original offset.
- [ ] For datetime model targets, reject existing `toDate`/`toISODate` and `castType` to date transforms that would discard the offset. Keep those transforms unchanged for date targets. Validate the final logical value through `useDataService`, including values supplied by static/copy/previous-row/lookup defaults. Use the instant-based `findOne` matching from step 2.
- [ ] Add `backend/test/services/import/DateTimeImport.test.ts`: valid CSV and string spreadsheet cells survive SOURCE → TRANSFORM → DEFAULT → VALIDATE → record write; invalid input uses the existing per-row error/count flow and the following valid row can succeed. Repeat an unchanged import to check existing no-op/history behavior; equivalent-offset matching must find the existing record.
- [ ] Verify a provider import through downloaded file, normalized CSV, saved import configuration/file metadata, queue, and record read. Dropbox and Google Drive share the retained string behavior; no provider settings or credential changes are required.

Automation record create/update/read already calls the shared data service. Test those calls with valid and invalid datetime strings and inspect the existing run JSON and record history for retained text. Keep untyped automation condition operators and scheduling unchanged. Restore a tenant backup containing a datetime model, default, template, and record, rebuild views through maintained model services, and verify the same value. Existing backup work for automation definitions/history belongs to [#290](https://github.com/five-panel/five-panel/issues/290); no backup inventory repair is part of this field feature. Do not edit historical migrations or tenant scripts.

## Final verification and evidence boundary

Run each new/changed focused test after its failing test and implementation. Then run these existing package checks from the application repository:

```sh
npm --prefix backend test
npm --prefix backend run build
npm --prefix frontend test
npm --prefix frontend run build
```

Database and browser checks require the project’s documented test environment; use the existing test setup and browser harnesses, including the new datetime integration file. Record the commands and results. A unit-only run cannot prove PostgreSQL casting, real provider processing, PDF appearance, browser offsets, or company isolation.

Before completing implementation, confirm all six spec acceptance checks, existing date-only tests (`frontend/test/calendarDates.test.ts`), default/template behavior, system timestamps, access checks, history, and latest external-ID behavior. No runtime feature tests are claimed by this documentation PR. Research baseline checks were 53 focused existing tests on `c3142a2`, followed by 33 existing DataService tests on `2e8e85a`; these runs overlap and are not a combined total. No browser or database integration suite was run during spec research.
