# Issue #288: Technical implementation plan

Source issue: https://github.com/benitogonzalezh/five-panel/issues/288

[Spec](288-strengthen-shared-validation.md) · [Short implementation guide](288-strengthen-shared-validation-implementation.md)

This document describes proposed work. The application code is not implemented by this spec change. Use the current `useDataService`, `useModelService`, and other service factories; earlier class-based examples are not the implementation baseline.

## Existing behavior and evidence

The field catalog already defines seven types. Records, model defaults, and template presets currently use different combinations of primitive checks, UI checks, and `Date.parse`. `DataService.update` validates the complete merged record before its no-op shortcut. The repositories already own transactional history and dispatch. Preserve the external-ID update behavior added by #294/#320.

Research reproduced the following against real service code with test repositories, without database writes:

- `2026-02-30` and `2026-09-11T00:00:00Z` pass date validation; `not-a-date` fails.
- Optional number and boolean fields accept `""`. Object fields accept malformed JSON text; presets reject even valid JSON text while accepting native objects and arrays.
- Generated GraphQL collection reads cannot serialize a native JSON object through the existing `String` field without an explicit serializer.
- Import `toDate` emits timestamps, can normalize an impossible day, and can move the written day when a configured local date is converted to UTC.

Read-only PostgreSQL probes confirmed the date equality/range-boundary mismatch caused by binding a JavaScript `Date` against a text view column, and empty-string cast failures for numeric, boolean, and JSONB views. A local development database contained older timestamp strings in text/date fields; this was not a production-data audit.

The research baseline at `c3142a2` passed 69 focused backend tests and 61 focused frontend tests. Later relevant source changes, including #320, were checked separately. Those results are existing-behavior evidence, not tests of this proposed implementation. Full builds, optional integration suites, and live browser journeys were not run for this planning task.

## Shared value contract

Extend the existing catalog and checks. Keep `FIELD_TYPES` data-only because `GET /models/field-types` exposes it to the editor. Add small pure helpers in `backend/src/fields/fieldValues.ts`; do not add a registry, dependency, shared-package project, or configurable validation framework.

The proposed shared result allows existing callers to retain their error envelopes:

```ts
type FieldValueResult =
  | { valid: true; value: unknown }
  | { valid: false; reason: 'type' | 'date' | 'json' | 'choice' | 'relation' };

function checkFieldValue(input: {
  field: Field;
  value: unknown;
}): FieldValueResult;
```

Keep `isValidFieldValue` as a compatibility wrapper over the shared check if callers still need its boolean result. Helpers import field types only; avoid a runtime import cycle with the catalog. The check validates one supplied value. Record-level presence, visibility, pair completeness, existence, and access remain service responsibilities.

### Accepted non-empty values

| Type or existing field variant | Accept | Reject as a new value |
| --- | --- | --- |
| Ordinary `string` text/textarea/email/url | `"DEL-001"`, `""`, whitespace, literal markup | Numbers, booleans, objects, arrays |
| `number` | `0`, `-2`, `2.5` | `"0"`, `""`, `true`, `NaN`, either infinity |
| `boolean` | `true`, `false` | `"false"`, `0`, `1`, `""` |
| `date`, or `string` with `uiType: 'date'` | `"2026-09-11"`, `"2028-02-29"` | `"2026-02-30"`, `"2026-02-29"`, unpadded/partial dates, surrounding whitespace, timestamps, `Date` instances, `""` |
| Non-relation `object` | Plain JSON objects/arrays, or strings encoding those shapes, such as `"{\"fragile\":false}"` and `"[1,null]"` | Malformed JSON, scalar JSON text such as `"42"`, non-JSON instances, cycles, nested non-finite numbers |
| `money` | A finite number, including `0`, with its existing configured currency partner | Numeric strings, non-finite values, incomplete or invalid pairs |
| `measurement` | A finite number, including `0`, with its existing configured unit partner | Numeric strings, non-finite values, incomplete or invalid pairs |
| Existing string/select | Configured option values; preserve its existing optional-empty behavior | Non-empty values outside the configured options |
| Existing object/relation | UUID string, or the supported option object whose configured `valueKey` contains that UUID | Invalid UUIDs, arrays, option objects without a valid configured ID |

A selected relation option is an input representation. Reduce new/changed relation option objects to the UUID before record checks and persistence, then resolve the related model and record in the authenticated company. An option object must not bypass the lookup. Do not add email, URL, amount-sign, rounding, currency, unit, or text-length business rules.

For JSON, preserve the supplied supported storage representation: native JSON stays native, and JSON text stays text. Parse text to validate it, not to rewrite it. Nested `null`, empty arrays, and empty objects are legitimate JSON contents. Keep the current record cleanup of undefined object properties and undefined array entries before checking final values; reject other unsupported shapes rather than silently serializing them into different data.

Fix the existing TypeScript definitions to represent both date metadata forms and the already supported non-relation object field. Do not introduce a standalone `array` field type merely because frontend branches mention it.

### Presence, clears, and defaults

| Input | Create | Update | Defaults/presets |
| --- | --- | --- | --- |
| Field omitted | Apply an eligible create default, otherwise leave absent | Preserve the saved value | An absent preset permits the normal model-default behavior |
| Explicit `null` | No value; suppress default fallback | Save `null` | Valid empty value; existing hidden/read-only required-field coverage can still reject it |
| Ordinary text `""` | Save empty text unless required | Save empty text unless required | Same text-value contract |
| `0` or `false` | Present value | Present value | Present value |
| Non-text `""` | Reject rather than broadly coerce | Reject rather than broadly coerce | Same rule |

The existing paired-field clear normalization is the narrow exception: a supplied empty paired member becomes `null`, and pair completeness still runs. Apply that same normalization when checking its defaults/presets. Preserve string/select's established empty handling. Do not use truthiness to detect an omitted or empty value.

Existing form controls send `null` when the user clears a date, number, JSON editor, or selection. A checkbox's `false` means false. Keep ordinary empty text as `""`. This issue adds neither a distinct stored “deleted” state nor a universal new clear button. Required/visibility and pair rules decide whether the final empty value is allowed.

At the import boundary, after configured transforms, defaults, and validators, prepare a remaining empty cell (`""`, missing, or null) as a record `null` for a non-text field. Keep ordinary text `""`, `0`, and `false`. Preserve the existing computation value used by matching expressions. This preparation does not allow a direct API numeric string or an invalid non-empty date.

### Calendar rules and old values

Use exact four-digit years `0001`–`9999`, two-digit months/days, and Gregorian month lengths. Check leap centuries: `1900-02-29` fails and `2000-02-29` passes. Use arithmetic or checked components, not broad JavaScript parsing. Frontend construction must handle years below 100 without JavaScript's 1900 adjustment.

Separate three operations:

1. **Strict value check:** accept a real `YYYY-MM-DD` only. New record inputs containing a time are rejected.
2. **Stored-value compatibility:** under the record lock, an unchanged saved value may bypass only a newly introduced restriction. Required, visibility, choice, and pair rules still run. Do not expose a caller-controlled bypass flag or retain a second full validator.
3. **Trusted old configuration use:** a recognized valid timestamp in an existing saved default or preset can supply its written day. Its original setting stays unchanged. New or changed defaults/presets must pass the strict contract.

Recognize the older ISO forms actually produced or accepted by the application: a checked full-date prefix followed by `T`, hours/minutes, optional seconds/fraction, and optional `Z` or numeric `±HH:MM` offset. Validate the time components enough to identify that representation; do not interpret an absent offset or convert an instant. Other old formats remain raw values for repair, not automatic conversion candidates. An impossible date prefix is never a usable legacy default.

On update, determine unchanged values before applying new representation normalization. If a valid incoming day equals the written day of a recognized saved timestamp, retain the saved timestamp. This prevents full-form resubmission and routine reimports from rewriting an unchanged day. A different valid day replaces it; explicit `null` clears it. A different newly supplied timestamp still fails. New stricter JSON/blank/relation-shape checks also must not block unrelated edits to values unchanged from storage; they do not waive a rule that already applied before this issue.

Show an invalid or unrecognizable old date as its raw value with a field message. Do not let `parseCalendarDate` silently display a corrected day and then save it. An invalid inherited create default/preset must be replaced with a valid value or explicitly cleared if optional before saving. Keep existing invalid configuration readable for repair.

## Files and implementation order

| Order | Existing code to change or reuse | Responsibility |
| --- | --- | --- |
| 1 | `backend/src/fields/fieldTypes.ts`; new `backend/src/fields/fieldValues.ts`; `backend/src/types/schema.ts` | Shared checks, small calendar/JSON/relation helpers, accurate existing field types |
| 2 | `backend/src/fields/recordFields.ts`; `backend/src/services/DataService.ts` | Presence/default provenance, full-record rules, locked old-value comparison, field errors, create/update validation |
| 3 | `backend/src/fields/modelFields.ts`; `backend/src/services/ModelService.ts`; `backend/src/utils/formTemplateValidation.ts`; `backend/src/services/FormTemplateService.ts` | Same value rules for configuration, precise paths, trusted old configuration handling |
| 4 | `frontend/src/utils/dateUtils.ts`, `recordFields.ts`, `formTemplates.ts`; `frontend/src/components/ModelForm.vue`, `FieldEditor.vue` | Correct day preparation, explicit clear values, raw old-value warnings, JSON object/array input, failed-save recovery |
| 5 | `backend/src/services/import/operations/transforms/type.ts`, `types.ts`, `processors/Transformer.ts`, `processors/DefaultResolver.ts`, `ColumnProcessor.ts`, `RowProcessor.ts`, `ModelMapper.ts`, `ImportOrchestrator.ts` | Calendar record values without changing existing identity expressions or source matching |
| 6 | `backend/src/schema.ts`; `backend/src/types/filters.ts`; `backend/src/services/FilterQueryBuilder.ts`; `backend/src/repositories/CustomDataRepository.ts`; small shared SQL-expression helper beside repositories | Date-filter input/comparison, JSON output serialization, calendar-aware existing uniqueness |
| 7 | Existing backend/frontend locale files and package-level tests; `e2e/test/records.spec.ts`, `forms.spec.ts`, `imports.spec.ts` | Translated errors and observable save/read/import/recovery checks |

Reuse `CustomDataRepository`, `ModelRepository`, `changeHistory.ts`, `useTransaction`, current schema invalidation, file storage, and the automation runner/savepoints. Do not move application checks into handlers or add duplicate history writes in services. No table, column, index, or stored-data migration is planned.

### Task 1: Shared checks and record saves

Implement the contract table with focused valid/invalid examples. In `DataService.validateValues`, stop skipping every `""` before type validation; skip missing/null values after the full record's required and pair checks. Ordinary text and existing selection empties use their stated rules.

Create flow remains: authenticate/resolve company → load and lock current model → strip undefined values → apply eligible defaults → normalize existing paired clears and supported input representations → validate full record, uniqueness, and relations → repository write/history/dispatch → commit.

Extend the create-default helper to return the IDs of values it supplied, alongside the prepared values. Only those values can use the old saved-model-default date conversion. A caller submitting the same timestamp directly does not acquire that exception. Preserve the current visibility and pair-default activation rules, including explicit null suppressing fallback.

Update flow remains: load/lock current model → validate allowed keys → lock current record → prepare incoming changes and merge → run full record rules and narrow stored-value compatibility → detect actual changes → check changed uniqueness/relations → repository update/history/dispatch → commit. No-op updates retain dates, authors, timestamps, and produce no event or dispatch. Preserve column-backed `externalId`/`importSourceKey` in the logical values and the behavior added by #320.

Verification: real service create/update tests must prove rejection before repository writes, literal `null` versus omitted field behavior, zero/false defaults, both date metadata forms, unchanged old values, current-model settings, and no-op preservation. Wrong-company records and foreign relations remain inaccessible.

### Task 2: Configuration and form agreement

Have model-default and preset validation call the same shared check. Keep `ModelDefinitionError` paths such as `fields.day.defaultValue`, and template paths such as `fields[1].presetValue`. Required coverage for hidden/read-only create fields remains a separate existing check; allowing a type-level null does not make a required hidden field complete.

When a management update retains an existing timestamp default/preset unchanged, allow that recognized old setting to remain. Compare against server-loaded prior configuration by model/field identity, not array position or a request flag. Other invalid old configuration remains visible with its existing warning/repair path. Assignment-only updates retain their current behavior. When forms use an existing valid timestamp preset, send its checked written day through the ordinary mutation; records do not gain a template-ID bypass.

Reuse the form's existing relation ID extraction and JSON text transport. Accept JSON text presets as well as native JSON shapes. Reject newly entered scalar JSON for an object/array field with a useful message. Preserve entered text on failure, including hidden-field errors and pair messages.

Verification: new/changed defaults and presets reject the same invalid semantic values as records. Default and preset `0`, `false`, `null`, and absence remain distinct. Valid old timestamp settings create the same calendar day through forms and backend defaults in UTC−6 and UTC+10. Test invalid inherited dates without allowing the UI to normalize them silently.

### Task 3: Imports without duplicate records

Do not simply change every `toDate` output to `YYYY-MM-DD`. `SourceResolver` uses `context.processedData` in computed expressions, and maintained ALCI configurations derive external IDs from transformed dates. Evidence includes the expressions installed by `backend/src/db/scripts/alci/20260311_01_fix_2024_import_keys.ts`. Those published scripts and saved configurations must remain unchanged.

Keep the existing processing value for computed expressions and introduce one internal record-value handoff:

```ts
// Add to the existing operation/column result when conversion needs a distinct record value.
recordValue?: unknown;

// Extend the existing row result; both maps are keyed by column name.
type PreparedRowValues = {
  data: Record<string, unknown>;
  recordValues: Record<string, unknown>;
};

// Extend the existing mapper result; these maps use target field IDs.
type PreparedModelValues = {
  data: Record<string, unknown>;
  matchData: Record<string, unknown>;
};
```

These are local processing contracts, not new configuration or API fields. Determine calendar meaning from the actual target model field, including older string/date metadata, not `ColumnConfig.type` alone. For other fields, the record value is the existing processing value. Detect an explicitly present `recordValue: null` by property presence, not `??` or truthiness.

Calendar-aware date operations validate the source day before any normalization and carry its day as `recordValue`, while retaining their established processing output for existing computed keys. Reuse the configured date-fns format for supported human input such as `MM/dd/yy`, `dd-MM-yy`, and `dd-MM-yyyy`; use checked local calendar components for the record value. With no format, accept a strict calendar day rather than guessing a locale or treating an arbitrary timestamp as a new day input. Preserve existing non-calendar transform behavior.

Carry this value through chained date operations. If a later configured transform changes the processing value, accept its strict day result or recompute using that explicit conversion; otherwise reject an inconsistent result. Never attach a stale day to a changed value. For Excel serial input, use the existing XLSX library's calendar parts; reject its fictitious `1900-02-29` rather than persisting an impossible day.

Apply the same handoff rule to defaults and validator results: an explicit final null clears the record value too. A validator that changes a date must leave a checked calendar value, not an earlier carried day. Keep the configured warn/null/skip behavior while ensuring that the final saved value satisfies the field contract.

Keep SOURCE → TRANSFORM → DEFAULT → VALIDATE order. A default can supply a raw previous-row value after the first transform pass. In that case apply the field's configured date conversion to that supplied value, carrying both results. For `copyFrom`, use the already prepared record value for calendar meaning while retaining the old processing value for expressions. Do not run all unrelated transforms a second time or infer a missing source format. Invalid non-empty calendar input must fail the row instead of becoming an unnoticed optional null.

`RowProcessor` keeps computation values in `context.processedData` and record values in the parallel map. `ModelMapper` builds the record payload from record values and matching values from computation values. Computed external IDs therefore retain their current bytes. For a calendar field explicitly used as a match column, try its previous exact representation, then its strict day representation; keep match-column priority unchanged. Do not broaden a match to any record on the same day or invent a new identity policy.

Only the final record payload reaches `DataService`. Its unchanged-day compatibility preserves an existing timestamp when the source day has not changed. A meaningful changed day follows the strict contract. Keep the existing row/file counters, source ownership, accepted external IDs, continued row-error accounting, and reconciliation condition `failed === 0`. Successful earlier reference records/rows can remain; this is not a whole-row or whole-file transaction change.

#### Complete import handoff example

This is a planned test fixture, running in UTC to make the existing identity text concrete. It uses the same `delivery` fields and company as the short guide. A model with ID `11111111-1111-4111-8111-111111111111` has quantity default `0` and confirmed default `false`.

The complete saved import configuration is:

```json
{
  "fileName": "deliveries.csv",
  "sheetName": "Sheet1",
  "baseModel": "delivery",
  "columns": [
    {
      "name": "Reference",
      "targetField": "reference",
      "type": "string",
      "source": { "type": "direct", "column": "Reference" }
    },
    {
      "name": "Day",
      "targetField": "day",
      "type": "date",
      "source": { "type": "direct", "column": "Day" },
      "transforms": [{ "type": "toDate", "params": { "format": "MM/dd/yy" } }]
    },
    {
      "name": "__key",
      "targetField": "externalId",
      "type": "string",
      "virtual": true,
      "source": { "type": "computed", "expression": "${Reference}|${Day}" },
      "matchPriority": 1
    }
  ]
}
```

The input file buffer encodes this complete CSV:

```csv
Reference,Day
DEL-002,09/11/26
```

The existing `importService.import` operation receives that buffer, the saved configuration above, the authenticated import user, and this server-owned context:

```json
{
  "tenantId": "33333333-3333-4333-8333-333333333333",
  "operationId": "88888888-8888-4888-8888-888888888888",
  "source": { "type": "import", "provider": "manual" },
  "actor": { "type": "user", "id": "44444444-4444-4444-8444-444444444444" }
}
```

After row/column processing, the two complete value maps are:

```json
{
  "data": {
    "Reference": "DEL-002",
    "Day": "2026-09-11T00:00:00.000Z",
    "__key": "DEL-002|2026-09-11T00:00:00.000Z"
  },
  "recordValues": {
    "Reference": "DEL-002",
    "Day": "2026-09-11",
    "__key": "DEL-002|2026-09-11T00:00:00.000Z"
  }
}
```

The mapper's corresponding values are:

```json
{
  "data": {
    "reference": "DEL-002",
    "day": "2026-09-11",
    "externalId": "DEL-002|2026-09-11T00:00:00.000Z"
  },
  "matchData": {
    "reference": "DEL-002",
    "day": "2026-09-11T00:00:00.000Z",
    "externalId": "DEL-002|2026-09-11T00:00:00.000Z"
  }
}
```

The orchestrator looks up that unchanged external ID through `DataService.findOne`. Assume no match. `DataService.create` supplies the model defaults, validates, and passes these record values to persistence:

```json
{
  "modelId": "11111111-1111-4111-8111-111111111111",
  "tenantId": "33333333-3333-4333-8333-333333333333",
  "externalId": "DEL-002|2026-09-11T00:00:00.000Z",
  "importSourceKey": null,
  "data": {
    "reference": "DEL-002",
    "day": "2026-09-11",
    "quantity": 0,
    "confirmed": false
  },
  "createdBy": "44444444-4444-4444-8444-444444444444",
  "updatedBy": "44444444-4444-4444-8444-444444444444"
}
```

The repository supplies record ID `99999999-9999-4999-8999-999999999999` and timestamps, and writes history/dispatch within the same transaction. No references or additional models participate. The importer returns:

```json
{
  "successful": 1,
  "created": 1,
  "updated": 0,
  "failed": 0,
  "skipped": 0,
  "skippedRows": [],
  "skippedDetails": [],
  "errors": [],
  "warnings": []
}
```

The file layer saves this result with its existing import time and completed status. It adds no continuation cursor or new queued import step. Repeating the same file finds the same ID and returns successful `1`, created `0`, updated `0`, with the other result fields unchanged. Seed the corresponding older timestamp record in a separate test: the same import must retain its external ID, raw date, record ID, timestamps, and zero update count when the written day is unchanged. Repeat timezone-sensitive checks using the existing processing value produced in that timezone; this change must not redefine old key strings.

### Task 4: Reads, filters, and existing uniqueness

Keep date mutation/output fields as `String` and raw date view columns as text. Add `DateFilter` to `backend/src/types/filters.ts` and map the catalog's declared `date` type to it:

```graphql
input DateFilter {
  eq: String
  ne: String
  gt: String
  gte: String
  lt: String
  lte: String
  between: [String!]
  isNull: Boolean
}
```

Validate non-null `DateFilter` comparison inputs as strict day strings in the service collection-read path too, including AND/OR/NOT and nested relations. A range has exactly two valid day strings in ascending order. `eq: null` and `isNull` retain their actual-empty checks. Leave the existing `DateTime` scalar and `DateTimeFilter` alone. Clients explicitly declaring `DateTimeFilter` variables for a declared date field must use `DateFilter`; ordinary inline day-string filters retain their shape. Older string/date fields keep `StringFilter` and all its existing operators.

The existing scoped `DataService.findOne` used by import identity matching keeps exact stored-value lookups. Those internal lookups must accept the previous timestamp representation and the strict-day fallback described above; do not run them through public `DateFilter` conversion or broaden them to calendar-day matches.

Create one small SQL expression helper for a checked written-day projection. Reuse it in both branches of filter condition collection and the relevant repository read/count/uniqueness paths. A valid day string projects to itself; a recognized old timestamp projects to its checked first ten characters. An unrecognizable/impossible old date projects to SQL null for calendar comparisons without casting it as a date or silently treating it as valid. Raw values remain available for reads and repair. Actual `isNull`/`eq: null` checks use the raw column so an invalid date is not reported as an empty field.

Use the same calendar rules as the shared checker. On PostgreSQL 17, a shape-guarded [`pg_input_is_valid(prefix, 'date')`](https://www.postgresql.org/docs/17/functions-info.html#FUNCTIONS-INFO-VALIDITY) can check a prefix safely; reject year zero separately. Do not create a new database function or cast arbitrary historical text. Do not change view column types, report SQL contracts, or introduce a date table-filter control.

For older string/date fields, use day comparison for `eq`, `ne`, `in`, and `notIn` when the operands are strict day strings; preserve raw timestamp/text operands and substring operators as existing string operations. Both the normal filter walk and grouped/nested walk must use this logic. Collection nodes and total counts must agree.

A declared date filter after the short guide's creation, before its explicit clear, has this complete request and result:

```json
{
  "query": "query DeliveryDays { deliveries(where: { day: { gte: \"2026-09-11\", lte: \"2026-09-11\" } }) { nodes { id reference day quantity confirmed } totalCount } }"
}
```

```json
{
  "data": {
    "deliveries": {
      "nodes": [
        {
          "id": "22222222-2222-4222-8222-222222222222",
          "reference": "DEL-001",
          "day": "2026-09-11",
          "quantity": 0,
          "confirmed": false
        }
      ],
      "totalCount": 1
    }
  }
}
```

Keep the example in an isolated scenario; the separate import fixture is not present. An older timestamp record on that same written day also matches without changing its returned representation.

Reuse existing unique-field locks and checks. For a calendar field marked unique, pass the service-selected calendar comparison to repository conflict/duplicate queries, using that same checked day projection. New writes must not evade uniqueness because one stored value is a timestamp and another is a day string; enabling uniqueness must detect those existing duplicates. Unchanged historical duplicates are not automatically repaired or made to block an unrelated update. Keep all other unique-field comparisons unchanged.

For non-relation object fields, add an explicit GraphQL field serializer: native JSON objects/arrays become JSON text; an already stored text representation remains text; a null remains null. Cover ID-loader and collection-view reads. Do not switch the public field to a different scalar or change the existing JSON scalar's unrelated behavior.

### Task 5: Error and recovery contracts

Use the current translated messages with the field label and ID. Add missing English/Spanish messages in the existing validation/form locale files.

- Record failures: `RecordValidationError`, `extensions.code: 'INVALID_RECORD_FIELDS'`, `extensions.fieldIds` containing the affected field, or both pair members.
- Model-default failures: existing `INVALID_MODEL_FIELDS`, `fieldIds`, and `paths` in the REST error response.
- Preset failures: existing `FormTemplateError` and `validationErrors` with the affected preset path.
- Import failures: preserve `errors: [{ row, error }]`; the error string names the model field and the row accounting remains accurate.
- Automation record failure: the existing runner records `invalid_input` and rolls back that step's effects. Do not add a safe retry for invalid values or change event/step payloads.

The short guide's complete create and update requests use the existing names `createDelivery`, `createDeliveryInput`, `updateDelivery`, and `updateDeliveryInput`, all under `/graphql`. Their successful mutation result remains the record ID. Add no routes, client-supplied tenant IDs, template-ID bypass, or authorization exceptions.

Verify rejection through the public Yoga response, not only helper predicates. Inject a failed repository/history/dispatch write and verify rollback. After a form rejection, its entered values and field message remain; correcting the field and retrying saves once.

## Verification and completion order

1. Add focused cases under `backend/test/fields/` and extend `DataCompanions`, `DataServiceVisibility`, `DataServiceInputs`, `DataServiceUpdateNoOp`, `DataServiceExternalId`, `ModelCompanions`, and `formTemplates/validation` coverage. Verify real failures/successes and saved values rather than repeating helper internals.
2. Add import service coverage for formatted/ISO/Excel days, impossible days, empty cells, defaults after transforms, copied dates, validator null results, composite external IDs, calendar match columns, unchanged old dates, reprocessing, and failed-row source reconciliation. Assert the same record IDs and keys on repeats.
3. Add an isolated PostgreSQL integration file `backend/test/integration/fieldValues.test.ts`, gated by `FIELD_VALUES_TEST_DATABASE_URL`, using the existing event-log/application-history fixture approach. Cover GraphQL create/update/read/filter, actual saved JSON, both date metadata forms, null versus omission, JSON serialization, uniqueness, company isolation, borrowed transactions, and event/job rollback. Default-suite skips do not count as running these checks.
4. Extend `frontend/test/calendarDates.test.ts`, `recordFields.test.ts`, `formTemplates.test.ts`, and `formTemplateComponents.test.ts`. Verify UTC−6/UTC+10, legacy values, explicit null clears, zero/false, JSON objects/arrays and nested null, invalid inherited values, and retained drafts.
5. Extend the existing browser journeys for create/reopen/clear, unchanged legacy-date editing, rejected value then retry, assigned presets, and actual upload/reimport without duplicates. Use isolated Docker volumes and the maintained E2E fixture; no host development servers or real cloud credentials.
6. Run the package suites and builds after focused checks pass. Run the existing paired/history/automation integration checks explicitly in their isolated databases. Verify schema reload and the maintained view-rebuild operation if any view-generation code changes; do not run a historical cleanup script.

From the application repository root, run the ordinary checks:

```sh
npm --prefix backend test
npm --prefix frontend test
npm --prefix backend run build
npm --prefix frontend run build
npm --prefix e2e run typecheck
npm --prefix e2e test -- records.spec.ts forms.spec.ts imports.spec.ts
```

For the new PostgreSQL suite, set `FP_FIELD_VALUES_TEST_DB_URL` to a separate disposable PostgreSQL database URL, then run from `backend/`:

```sh
FIELD_VALUES_TEST_DATABASE_URL="$FP_FIELD_VALUES_TEST_DB_URL" ./node_modules/.bin/tsx --test test/integration/fieldValues.test.ts
```

Existing integration entry points use `PAIRED_FIELDS_DB_TEST`, `AUDIT_TEST_DATABASE_URL`, `APPLICATION_HISTORY_TEST_DATABASE_URL`, and `AUTOMATION_TEST_DATABASE_URL`; follow each file's isolation/setup requirements. Use the root ops CLI for maintained migration/rebuild flows. Report only checks actually executed, with skips separated from passes.

## Preserved areas and limits

Keep company isolation, current-model locks, relation scope, existing unique locks, default activation, pair clears, import partial success, source reconciliation, files/configuration history, and transactional history/dispatch. No new provider calls or worker payloads are required. Account recovery, credentials, menus, report filters, backups, and generic request validation are outside the change. Pre-existing unreadable casts are not a data-cleanup project; the change prevents new invalid writes and preserves the raw record repair path.

Coordinate the calendar helpers with [#287](https://github.com/benitogonzalezh/five-panel/issues/287), but do not implement its new business datetime field or its comparison-by-instant rules here. System `createdAt`/`updatedAt` remain unchanged.

The approach follows the calendar format and month/leap rules in [RFC 3339 §§5.6–5.7](https://www.rfc-editor.org/rfc/rfc3339.html#section-5.6), and the storage distinction in [PostgreSQL 17 JSON documentation](https://www.postgresql.org/docs/17/datatype-json.html). [GraphQL input coercion](https://spec.graphql.org/September2025/#sec-Input-Objects.Input-Coercion) distinguishes omission from explicit null; Five Panel keeps its existing update/default meaning for those inputs. These sources support small checked conversions within the existing services rather than a new validation framework.
