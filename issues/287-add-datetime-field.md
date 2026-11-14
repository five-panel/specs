# Issue #287: Add a datetime field type that preserves the original UTC offset

Source issue: https://github.com/benitogonzalezh/five-panel/issues/287

## TL;DR

Add a `datetime` field that keeps the original date, time, and UTC offset. Saving `2026-09-11T10:00:00-03:00` must preserve that value when reopened anywhere.

[Technical implementation plan](287-add-datetime-field-implementation-plan.md) · [Short implementation guide](287-add-datetime-field-implementation.md)

## What will change

**Code impact: Medium.** Extend model setup, shared forms, validation, queries, and reports. Reuse existing record storage, access checks, and change history.

Add a shared date, time, and offset control, with English and Spanish labels.

## How it would work

1. An administrator adds a **Date and time** field through the model editor.
2. For a new value, the form fills the browser’s offset for the chosen date and time. The offset stays visible and editable. Saved values, defaults, and template presets keep their supplied offset.
3. APIs and imports accept values with an explicit offset. Validation checks the actual date, time, and offset. Seconds can include up to six decimal places.
4. Storage and API responses retain the original text. Reopening a value or changing another field leaves it intact.
5. Filters and sorting compare the actual moment. For example, `10:00-03:00` equals `13:00Z` on the same date.
6. Tables, forms, reports, and exports show the original time and offset. Report ranges and ordering use the actual moment.

## Done when

- Administrators can add datetime fields and valid defaults.
- Standard forms and templates preserve values through create, reopen, edit, and changes to the browser’s time zone.
- Equal moments compare equally, and ranges, sorting, and paging work across different offsets.
- Reports display the original offset and return correctly ordered results.
- Invalid input shows a clear error and preserves the draft; unchanged edits create no extra history.
- Existing date fields, system timestamps, and company access rules still work.

## Out of scope

- Time-only and duration fields.
- Named time-zone settings, scheduling, and recurrence.
- Converting existing fields or rewriting old records.
- Unique datetime constraints and new automation condition operators.
