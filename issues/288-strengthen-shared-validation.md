# Issue #288: Strengthen shared validation for dynamic model field values

Source issue: https://github.com/benitogonzalezh/five-panel/issues/288

## TL;DR

Strengthen the shared checks we already have so forms, API requests, imports, model defaults, and form-template presets agree on valid field values. Reject invalid new values before saving, while keeping older records editable.

Companion documents: [Technical implementation plan](288-strengthen-shared-validation-implementation-plan.md) and [Short implementation guide](288-strengthen-shared-validation-implementation.md).

## What will change

**Code impact: Medium.** Extend `backend/src/fields/fieldTypes.ts` with small shared value checks, then reuse them from `DataService`, model field validation, and template validation. Make focused changes to form serialization, import date handling, JSON reads, and date filters. Keep the existing services, repositories, field catalog, storage, and transactions.

The backend owns the checks. Forms prepare the expected values and show field errors. No new validation framework or database tables are needed.

## How it would work

1. Define one accepted-value contract for every current field type:

   | Field | Non-empty value |
   | --- | --- |
   | Text | A string; ordinary text has no new business restrictions |
   | Number | A finite number, including zero, negative numbers, and fractions |
   | Boolean | `true` or `false` |
   | Date | A real Gregorian day in `YYYY-MM-DD`, with correct month lengths and leap years |
   | Object/JSON | A JSON object or array, supplied directly or as valid JSON text |
   | Money | A finite number with its configured currency partner |
   | Measurement | A finite number with its configured unit partner |

   Keep existing selection membership and related-record checks. Supported relation option objects are reduced to their configured ID before record validation and lookup.

2. Use the agreed meaning of empty values: **`null` means no value**. On update, an omitted field keeps its saved value. On create, omission can use an eligible default; explicit `null` prevents that fallback. Ordinary text `""` stays empty text. Zero and `false` remain values. Cleared non-text form controls send `null`; direct non-text strings are not silently turned into numbers, booleans, or dates. Preserve the existing paired-field clear rule.

3. Validate each final record before persistence, including direct API requests, imports, and automation record operations. Keep required fields, conditional visibility, selection choices, relations, uniqueness, and paired-field checks. Defaults and presets use the same value rules; presentation settings do not bypass backend checks.

4. Apply the date rule to both `type: 'date'` and older `type: 'string', uiType: 'date'` fields. Reject new timestamps and impossible days. An unchanged older value must not block an unrelated edit solely because of a newly added check; existing record rules still apply. Never rewrite the old value as a side effect of editing another field. Show an unrecognizable saved date as its original value with a repair message.

   For existing saved timestamp defaults and presets, use their valid written calendar day when creating a new record, without changing the saved setting or converting timezones. New or changed configuration must use the strict contract. An impossible inherited day needs correction or an explicit optional-field clear.

5. Make imports produce valid calendar-day values after their configured conversions and defaults. Preserve existing matching keys, source ownership, file configurations, and original files. Reimporting must find the same records. Row errors prevent source reconciliation; earlier successful record writes retain the existing partial-success behavior.

6. Make date equality and inclusive ranges work for new day strings and recognized older timestamps, using the written day without timezone conversion. Use the same day comparison for existing unique date fields. Keep older text-date string-filter operations. Preserve JSON objects and arrays through save and reopen using the existing GraphQL text contract. Return translated field errors through the existing paths, and retain the form draft after rejection.

## Done when

- Valid and invalid examples cover all seven types, defaults, presets, missing values, `null`, empty text, zero, and `false`.
- Forms, API writes, imports, and automation record steps enforce the same field rules; rejected records produce no record change, change event, or dispatch job.
- Leap days work; impossible days and newly supplied timestamps fail for both forms of date field; date filters include their expected equality and range boundaries.
- Unrelated edits preserve unchanged older values exactly; existing timestamp defaults and presets work without rewriting their saved configuration.
- Import repeats retain record IDs and matching keys, report field and row errors, and do not delete stale source records after a row fails.
- Existing access, required, visibility, choice, relation, uniqueness, and paired-field behavior passes regression checks; JSON reopens correctly and failed saves retain entered values for retry.

## Out of scope

- New business restrictions, a new validation framework, or new field types. Business `datetime` remains [#287](https://github.com/benitogonzalezh/five-panel/issues/287).
- Changes to system timestamps, report filter contracts, authentication, or general HTTP validation.
- Automatic historical cleanup, history rewriting, published tenant-script changes, or repair of data already unreadable before this change.
- Whole-file transactions, changes to import identity or match priority, a separate stored state for “missing versus null,” or a new general-purpose field-delete operation.
