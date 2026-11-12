# Issue #275: Give record-form dropdowns their visible field labels as accessible names

Source issue: https://github.com/benitogonzalezh/five-panel/issues/275

## TL;DR

Make record-form dropdowns use their visible field labels as the names read by screen readers. People can then tell the price currency, cost currency, status, and related-record fields apart. The form keeps its appearance and saved values.

Companion documents: [Technical implementation plan](275-give-record-form-implementation-plan.md) · [Short implementation guide](275-give-record-form-implementation.md).

[Doop preview](https://five-pane-dev-server.tail7d89bf.ts.net/c/J88FjcHO5C/proto?frame=wdTVgJirzc) shows the difference in what a screen reader identifies.

## What will change

**Small code impact:** connect the existing labels to both dropdown branches in `frontend/src/components/ModelForm.vue`, then add a focused browser regression using real PrimeVue controls.

Reuse PrimeVue's supported label reference and the existing form loading, templates, validation, and saving paths. No new application service is needed.

## How it would work

1. The existing table opens a create or edit form. Its model definitions continue to supply the visible field labels.
2. Give each rendered label a stable ID and make its dropdown refer to that label on the actual focusable control.
3. Apply this to ordinary Select and Relation fields. Currency and unit companions already use the ordinary Select branch, so they receive the same fix.
4. Keep the field name when the control is empty, defaulted, selected, cleared, or reopened. Continue using the displayed label in English or Spanish, including forms using templates and conditional fields.

## Done when

- Select and Relation controls have names containing their visible field labels; price and cost currencies have distinct names.
- Those names survive empty, defaulted, selected, cleared, and saved/reopened states in create and edit forms.
- English and Spanish labels work in default and assigned-template forms. Hidden, read-only, and conditionally shown fields keep their existing rules.
- Keyboard selection, relation filtering, numeric/date labels, required indicators, and the form at a 390 × 844 phone viewport remain usable.
- Required/pair errors can be corrected and failed saves retain the draft. Saved currencies, relation IDs, and numeric zero values remain correct.
- A browser regression checks the real rendered combobox by role and label, including selection and reopening; replacing PrimeVue with a test stub is insufficient.

## Out of scope

- Money storage, companion rules, currency conversion, backend changes, or database migrations.
- Automatic translation of model field names, date conversion changes, other form surfaces, or a form redesign.
- Adding the Doop preview's name readouts or sample-language controls to the product.
