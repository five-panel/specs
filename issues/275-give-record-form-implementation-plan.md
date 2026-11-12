# Issue #275: Technical implementation plan

[Spec](275-give-record-form.md) · [Short implementation guide](275-give-record-form-implementation.md)

**Goal:** make the real record-form combobox use its visible field label throughout its existing states.

**Approach:** add a label ID and the supported PrimeVue label-reference binding in the shared form, with a real browser regression. Keep the existing Vue 3, PrimeVue 4.5.1, Pinia, and Playwright setup.

Source paths below belong to [five-panel/five-panel at 9c777bd](https://github.com/five-panel/five-panel/tree/9c777bd8f7f7cd492c3d4e5f105a631b00c736fb). These are implementation instructions; the application fix has not been implemented.

## Files and responsibilities

| File | Work |
| --- | --- |
| `frontend/src/components/ModelForm.vue` | Modify the existing label and both Dropdown branches. |
| `e2e/test/record-form-accessibility.spec.ts` | Add the browser regression. This is the only planned new file. |
| `frontend/src/components/FieldEditor.vue` | Reuse its existing `aria-labelledby` binding pattern as a reference; leave this separate surface unchanged. |
| `frontend/src/utils/formTemplates.ts`, `frontend/src/utils/recordFields.ts` | Reuse field selection, defaults, payloads, pair validation, and save-error handling. |
| `frontend/src/views/ModelView.vue`, `frontend/src/stores/dataStore.ts` | Reuse create/edit entry points and saving. |
| `e2e/test/fixtures.ts`, `e2e/test/support.ts` | Reuse isolated company/account fixtures, API helpers, login, and cleanup. |

## Naming and data contracts

- Existing visible text comes from `field.displayName`, with the existing required marker when applicable. It is model metadata, not a translation key.
- Preserve `for="record-${field.id}"` and `inputId="record-${field.id}"`. Add `id="record-${field.id}-label"` to the visible label and use that ID for both Dropdown references.
- Bind through the component's supported `aria-labelledby` prop, which PrimeVue maps to `ariaLabelledby` and forwards to its focusable `span[role="combobox"]`. A reference on the outer wrapper alone does not fix the issue.
- Keep `v-model`, options, placeholders, relation filtering, clearing, invalid/disabled state, and event handlers intact. No new props, events, wrappers, or composables are needed.
- ModelForm's existing `save` event still emits `Record<string, any>` containing business values. Currency options save strings such as `CLP` and `USD`; relations save their `valueKey` IDs; zero remains a number. Clearing a dropdown retains the existing null behavior. Naming attributes never enter `formData` or the payload.
- Existing metadata/template requests remain `GET /models` and `GET /form-templates/assigned?modelId=<modelId>&operation=create|edit`. Record queries/mutations continue through `/graphql`.
- For the test model slug `qaLabelItem`, existing save operations are `createQaLabelItem(input)` and `updateQaLabelItem(id, input)`, returning a record ID. Create sends the completed draft; edit sends changed editable fields. The guide's final example gives concrete input values.
- Labels remain in `models.fields`; business values remain in `custom_data.data`. The existing schema, DataService, and repositories retain tenant checks, validation, relation resolution, transactions, and history. No storage change is required.

## Implementation and verification order

### 1. Add a failing browser case

- [ ] Create the new test file using the existing `test`, `scenario`, and `api` fixtures. Prepare all data through the existing public APIs and use returned IDs.
- [ ] Create a related model, `qaLabelProduct`, with a `name` text field and a sample record named `Café de origen`. Create `qaLabelItem` with these fields:

| Field | Definition |
| --- | --- |
| `name` | Required string/text. |
| `product` | Required object/relation to `qaLabelProduct`, `valueKey: id`, `labelKey: name`. |
| `status` | Required string/select; values `active`, `inactive`; default `active`. |
| `currentPrice`, `currentUnitCost` | Optional Money/number fields, each linked reciprocally to its own currency field. |
| `currentPriceCurrency`, `currentUnitCostCurrency` | Optional string/select fields; options `CLP`, `USD`; no default needed for the fixture. |
| `recordDate` | Optional date/date field for the existing native-label check. |
| `kind` | Optional string/select, values `retail`, `wholesale`; visible when `status` equals `inactive`. |

- [ ] Use the guide's Spanish money labels and their English equivalents: `Current price`, `Currency: current price`, `Current unit cost`, `Currency: current unit cost`. Name the other fields `Nombre`/`Name`, `Producto`/`Product`, `Estado`/`Status`, `Fecha`/`Date`, and `Tipo`/`Kind`.
- [ ] Create a table view through `post('/views')` for `qaLabelItem`, with a `qaLabelItems` connection query and a name column. Do not use `createTable` unchanged: its query is fixed to the inventory model.
- [ ] Open the real table and choose Nuevo/New. Assert that the two empty currencies are individually found by their field labels and combobox names. On the original code, these assertions must fail for the reported naming problem.

For bilingual tests, create a browser context using the base `browser` fixture, set its locale and saved `locale` before navigation, and close it in teardown. The shared `page` fixture always installs Spanish initialization; avoid competing initialization scripts. Reuse `signIn`, then locate Nuevo/New in the selected language.

### 2. Make the focused form change

- [ ] Add the label ID and the matching reference to both existing Dropdown branches. The following shows only the proposed bindings; retain all other markup and behavior:

```vue
<label :id="`record-${field.id}-label`" :for="`record-${field.id}`">
  <!-- Keep the current visible label and required marker. -->
</label>

<!-- Apply these bindings to the existing Select and Relation Dropdowns. -->
<Dropdown
  :inputId="`record-${field.id}`"
  :aria-labelledby="`record-${field.id}-label`"
/>
```

- [ ] Rerun the failing case. Check the actual focusable combobox and its computed name, not just the presence of an attribute. Keep selected text/value assertions separate from name assertions; existing required-marker text may appear in the name.

### 3. Cover the existing user flow

Run the naming and template checks with both Spanish and English sample labels.

- [ ] Check ordinary Select and Relation controls when empty, defaulted, selected, and cleared. Use keyboard opening, arrow selection, Enter, and Escape; verify relation filtering still works.
- [ ] Enter the guide's draft, save through the real backend, and reopen it for editing. Read back the values, checking independent `CLP`/`USD` currencies, `0`, and the selected relation ID.
- [ ] Clear one currency while its amount remains; confirm existing pair errors and retained values. Restore it and save. Interrupt one otherwise valid save request in the browser, confirm the draft and field names remain, then restore requests and retry successfully.
- [ ] Check an assigned template with the product preset/read-only, date hidden, and status/kind editable. Change status to reveal, hide, and reveal Kind; each rendered reference must resolve to exactly one current label. Close/reopen the form and check again.
- [ ] Verify numeric/date labels and a 390 × 844 viewport, including scrolling to both currencies and reaching Save. Use the existing unit/component tests for the remaining loading/retry, hidden-field, and pending-save behavior. The Doop annotations are not product UI or a substitute for these tests.

## Commands and expected results

Run from an isolated application workspace configured using `e2e/README.md`, with its normal Docker application and E2E dependencies available:

```sh
docker compose exec -T frontend node --import tsx --test test/recordFields.test.ts test/formTemplates.test.ts test/formTemplateComponents.test.ts
npm --prefix e2e run typecheck
npm --prefix e2e test -- test/record-form-accessibility.spec.ts
npm --prefix e2e test -- test/records.spec.ts test/forms.spec.ts
docker compose exec -T frontend npm run build
git diff --check
```

Expect the new browser cases and existing form journeys to pass, type checking/build to succeed, and no whitespace errors. Spot-check one empty and selected control with a screen reader; verify both its field name and selected value remain available. Date conversion, other form surfaces, and backend refactors are outside this change.

Research used the focused review explicitly approved by the user. The three existing frontend test files passed 36 tests, and the unchanged real form reproduced the incorrect names in both languages. The separate Doop prototype passed nine preview checks. These are baseline/design findings, not verification of an application fix or the planned new test file.
