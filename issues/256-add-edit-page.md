# Issue #256: Add and edit page customizable

Source issue: https://github.com/benitogonzalezh/five-panel/issues/256

## Problem

Five Panel builds record forms directly from a model's fields. This gives every user the same form even when their work only requires a small part of a large model. For example, a model may contain 20 fields while a particular user should enter only five; the other values may be preset, shown without editing, or omitted from that user's form.

Tenant administrators need to create named form templates, control each template's field behavior, and assign templates to users. Users should then get the right create or edit experience without requiring custom code for each tenant.

## Current Behavior

- The New action opens one generic `ModelForm` dialog generated from the current model.
- The form uses a fixed three-column grid and is not designed for phone and tablet widths.
- The form supports record creation only. Existing records are edited by double-clicking individual table cells.
- All model fields that pass the model's existing conditional visibility rules are included in the create form.
- There is no stored form-template metadata, no template management page, and no user assignment or template-selection flow.
- Table views often query only their selected columns, so a displayed row is not guaranteed to contain all values needed by a full edit form.
- Record creation and updates already pass through the normal dynamic GraphQL mutations and backend data service, which applies model defaults and validation.

## Desired Behavior

Tenant administrators can manage named, tenant-scoped form templates. Each template belongs to one model, applies to Create, Edit, or Both, and contains an ordered list of field rules. A field rule marks a field as editable, read-only, or hidden and may provide a create-time preset value.

Template authoring stays deliberately simple in this version. Five Panel's existing `SmartJsonEditor` edits one versioned JSON object containing the template name, model, Create/Edit/Both applicability, and ordered field rules. The JSON editor provides syntax feedback, formatting, and fullscreen editing, while server validation reports invalid settings with their JSON paths. Assigned users stay outside the JSON and use a normal active-user picker so administrators do not need to find or type user IDs. There is no visual or field-by-field form builder.

Administrators can assign each template to one or more active users in the same tenant. For the requested model and operation:

- A user with no assigned applicable template receives an automatic all-fields form.
- A user with one assigned applicable template opens that form directly.
- A user with more than one assigned applicable template chooses a template before the form opens.

The New action uses the create-template flow. Each data row has an Edit action that uses the edit-template flow. The new full-record edit form replaces the current double-click cell editor. Delete behavior is unchanged.

The first version treats templates as frontend behavior backed by stored metadata. The backend stores, validates, assigns, and returns the templates, but record mutations do not enforce a template's hidden, read-only, or preset rules. Direct API callers can therefore bypass those presentation rules. This limitation must be documented and must not be presented as field-level security.

Forms are responsive without separate layouts per device. They use one field per row on phones, two per row on tablets when space permits, and up to three per row on desktop. On phones, the form container uses the available viewport width, stays within the viewport height, and gives the form body its own vertical scroll area. Field reading order remains the configured order at every size, controls do not require horizontal form scrolling, and form actions remain easy to reach while scrolling.

Template rendering remains separate from record submission. Create and edit forms continue to use Five Panel's standard mutation path so existing or future workflows can observe the same record operations regardless of whether they came from a form, import, API call, or future inline editor. The stored configuration is versioned so later work can add compatible form features. Pre-hooks, post-hooks, and workflow authoring are not part of this issue.

## Scope

- Add tenant-scoped persistence for form templates and their user assignments.
- Add authenticated API operations for administrators to create, read, update, delete, and assign templates.
- Add an authenticated API operation that returns only the current user's assigned templates for a model and operation.
- Restrict template management to tenant administrators and all template data to the current tenant.
- Add an administrator Form Templates settings page.
- Reuse the existing `SmartJsonEditor` for administrators to enter the complete versioned template configuration: name, model, Create/Edit/Both use, ordered field rules, field modes, and create-time preset values.
- Use a normal active-user picker for template assignments rather than placing user IDs in the JSON configuration.
- Validate the complete JSON configuration, including its name, model, applicability, field references, and preset value types.
- Refactor the existing create-only model form into a reusable metadata-driven create and edit form.
- Add the zero, one, and multiple-template resolution flows.
- Add a row Edit action and remove double-click cell editing.
- Fetch the complete set of record values required by the chosen edit template before displaying the form.
- Preserve existing model field types, model defaults, required-field validation, relation options, select options, and conditional visibility behavior.
- Add responsive phone, tablet, and desktop behavior to the user form, template chooser, and administrator template editor.
- Add English and Spanish copy for all new labels, instructions, states, and errors.
- Add automated and browser-level coverage for management, assignment, selection, form rendering, saving, and responsive behavior.

## Out Of Scope

- Enforcing template field rules in GraphQL mutations or the backend data service.
- Treating hidden or read-only form fields as security controls.
- Pre-hooks, post-hooks, workflow definitions, or workflow execution.
- A new or more powerful inline table editor.
- A graphical field picker, field-by-field setup wizard, live form preview, or other visual form builder.
- Drag-and-drop rows, columns, sections, tabs, or separate device-specific layouts.
- Assigning templates through roles, teams, or groups.
- Formulas, scripts, or dynamic preset values based on the current user or record context.
- Remembering or configuring a preferred template when a user has multiple applicable templates.
- Public or unauthenticated data-entry forms.

## Acceptance Criteria

1. A tenant administrator can create a template by entering one JSON configuration with a non-empty name, a model in the current tenant, an applicability of Create, Edit, or Both, and at least one valid field rule.
2. A tenant administrator can edit and delete a template and can assign or unassign it to active users in the same tenant.
3. A non-administrator cannot access template-management pages or successfully call template-management API operations.
4. Template, model, and user-assignment reads and writes cannot cross tenant boundaries, including when identifiers from another tenant are submitted directly.
5. A template configuration is one versioned JSON object containing its name, model ID, applicability, and ordered list of unique model field IDs. Each field rule supports editable, read-only, or hidden mode and an optional create-time preset value of a type accepted by that model field.
6. The administrator create and edit screen uses the existing `SmartJsonEditor` for the complete template configuration and a normal active-user picker for assignments. User IDs are not part of the JSON. Save is unavailable while the JSON has a syntax error. A server validation error for a missing or invalid name, a model outside the current tenant, invalid applicability, duplicate field rules, fields outside the chosen model, invalid field modes, or invalid preset values is shown next to the JSON editor with the affected configuration path.
7. When a model field referenced by an existing template no longer exists, the user form ignores that rule without crashing and the management screen shows which JSON configuration path needs attention.
8. For Create or Edit, a user with no assigned applicable template opens an automatic fallback form containing all current model fields in model order.
9. For Create or Edit, a user with exactly one assigned applicable template opens it without an extra selection step.
10. For Create or Edit, a user with two or more assigned applicable templates can see their names, choose one, cancel, and open the chosen form.
11. A template marked Both participates in both create and edit resolution; a Create-only or Edit-only template appears only for its matching operation.
12. The rendered form follows the template's field order. Editable fields use the model's normal input control, read-only fields are visible but cannot be changed, and hidden fields are not displayed.
13. On create, an editable field with a preset starts with that value and can be changed. A read-only or hidden field with a preset submits that value without offering an editable control. When no preset exists, the form displays a model default when one exists and otherwise leaves the value for existing backend default and validation behavior.
14. On edit, editable and read-only fields start with the record's current values. Hidden fields are omitted from the update payload and remain unchanged. Create-time preset values do not overwrite an existing record.
15. Existing model `visibleWhen` rules continue to apply. A field hidden by either the model condition or the template is hidden, and changing a controlling value updates dependent field visibility as it does today.
16. Opening an edit form loads every record value needed by the chosen template even when those fields are absent from the table view's query or visible columns.
17. Successful create and edit submissions use the existing dynamic record mutations, refresh the displayed records, close the form, and show the existing translated success feedback.
18. A failed record load does not open an empty form as if the record loaded successfully and offers a useful translated error or retry state. A failed save keeps the form open, preserves the user's entered values, and shows a useful translated error.
19. Data cells no longer enter edit mode on double-click. Each non-placeholder record has an accessible Edit action, while the existing Delete action keeps its current behavior.
20. At phone widths below 640 pixels, the form uses one field per row, uses the available viewport width, stays within the viewport height, and scrolls its body vertically. At tablet widths from 640 through 1023 pixels, it uses up to two fields per row. At desktop widths of at least 1024 pixels, it uses up to three fields per row.
21. At all supported widths, configured field order is also the visual and keyboard reading order, labels and validation messages wrap without clipping, form controls do not cause horizontal form scrolling, and Save and Cancel remain reachable while the form scrolls.
22. The template chooser and administrator template editor remain usable with touch input at phone and tablet widths and do not depend on hover or precise pointer actions.
23. All new user-facing text is available in English and Spanish, and form labels, disabled/read-only states, selection dialogs, focus handling, and action controls are exposed clearly to keyboard and assistive-technology users.
24. Existing direct GraphQL create and update behavior remains compatible and does not require a form-template ID. Automated tests demonstrate that template rules are not represented as backend field permissions in this version.
25. Form rendering and template resolution remain separate from the shared create/update submission path, allowing later automation work to add pre- or post-operation behavior without creating a form-only record-write path.

## Implementation Notes

The recommended persistence shape is a `form_templates` record containing one versioned JSON configuration, plus a many-to-many user assignment record. Keep record identity, tenant ownership, audit columns, and timestamps outside the administrator-authored JSON. Follow the repository's existing UUID, tenant, foreign-key, and timestamp patterns where they apply. If the backend copies validated JSON values such as model ID or name into columns for indexing or referential integrity, those values must be derived from the JSON and updated atomically rather than becoming a second editable source. Deleting a model or template should remove dependent template or assignment rows through explicit service behavior or database cascades. Template names should be unique within a model and tenant.

Reuse `frontend/src/components/json/SmartJsonEditor.vue` for the complete configuration instead of creating another JSON input or a visual form builder. A new template should start with a valid, formatted version-one example. The management page should provide enough model and field reference information for an administrator to enter valid IDs without guessing. Pass server-side schema errors through the editor's existing validation-error surface. Keep Save disabled for JSON syntax errors, pending validation, or server validation failures, while preserving the administrator's text so it can be corrected. Keep the active-user assignment control separate from the editor.

A version-one configuration can use this conceptual shape:

```ts
type FormTemplateConfig = {
  version: 1;
  name: string;
  modelId: string;
  appliesTo: "create" | "edit" | "both";
  fields: Array<{
    fieldId: string;
    mode: "editable" | "readOnly" | "hidden";
    presetValue?: unknown;
  }>;
};

type FormTemplate = {
  id: string;
  tenantId: string;
  config: FormTemplateConfig;
};
```

Property absence must remain distinguishable from an explicit `null` preset. Fields omitted from a stored configuration are hidden for that template. User assignments are separate records and are not accepted inside the configuration. The fallback all-fields form is generated from current model metadata and does not need a stored template row.

The API should accept and return the complete configuration as one JSON object and provide operations for administrator CRUD, replacement of a template's assigned user list, and current-user lookup by `modelId` plus `create` or `edit` operation. Current-user lookup must derive the user and tenant from the authenticated request rather than accepting a target user ID. Management writes must verify that the model referenced by the configuration, the template, and the separately assigned active users belong to that tenant.

Template validation should use current model metadata for field IDs, field types, select values, and relation shape. It should warn or reject configurations that obviously cannot satisfy an applicable required field through user input, a template preset, or a model default. The existing data service remains the final validator for record writes.

Reuse the current model field renderers and field-visibility helpers rather than building a second field-type system. The shared form should have explicit create and edit modes and a payload builder that includes editable values and applicable create presets while omitting edit-time hidden fields. On create, initialize fields from a template preset first and a model default second; keep an editable initialized value changeable, and submit a read-only or hidden initialized value without exposing an editable control. Model conditional visibility is an additional restriction and cannot make a template-hidden field visible.

Do not build edit state from the displayed table row alone. Construct a focused query for the chosen template's editable and read-only fields, their visibility dependencies, and relation value/label fields, then load the record by ID before opening the edit form.

Use responsive CSS grid behavior with one, two, and three column limits at the specified widths. The DOM order must match template order instead of using visual-only reordering. On phones, prefer a near-full-screen dialog or equivalent contained page with a scrollable body and reachable actions.

Keep template selection, form rendering, payload construction, and record submission as separate units. Submission must call the existing shared mutation path. Do not add guessed `preHooks` or `postHooks` properties to version one. A later template version or mutation context can carry the selected template ID if a future automation design needs form-specific conditions; this issue has no dependency on such a system.

## Test Expectations

- Add migration tests for template and assignment tables, tenant-scoped uniqueness, foreign keys, and deletion behavior.
- Add repository and service tests for administrator CRUD, active same-tenant assignments, current-user lookup, applicability filtering, stale field references, and template validation.
- Add route tests proving administrator-only management, authenticated assigned-template reads, tenant isolation, and rejection of cross-tenant identifiers.
- Add frontend unit tests for zero/one/multiple template resolution, Create/Edit/Both filtering, field ordering, field modes, preset handling, edit payload omission, and the generated fallback form.
- Extend field-visibility tests to cover the intersection of model conditional visibility and template visibility.
- Add component tests for the administrator management form, complete-configuration `SmartJsonEditor` integration, valid starter configuration, syntax and server validation errors, preserved invalid text, separate user assignment control, template chooser, create form, complete-record edit loading, success states, and recoverable errors.
- Add regression coverage proving that double-click no longer edits a cell, the row Edit action uses the template flow, Delete is unchanged, and direct dynamic GraphQL mutations remain compatible.
- Run the backend and frontend full test suites and builds, plus formatting or static checks available in each package.
- Perform browser checks in English and Spanish at representative phone widths below 640 pixels, tablet widths from 640 through 1023 pixels, and desktop widths of at least 1024 pixels. Cover zero, one, and multiple assigned templates with keyboard and touch-style interaction.

## Risks

- Template rules are presentation rules only. Users who can call the API directly can bypass hidden, read-only, and preset behavior until a separate field-permission or backend-enforcement feature exists.
- Model changes can leave stored field references stale or introduce new required fields that an older create template cannot satisfy. Runtime handling must not crash, and administrators need clear warnings so they can repair affected templates.
- Edit forms cannot rely on partial table rows. Failing to fetch template fields and their visibility or relation dependencies could display missing values or overwrite data incorrectly.
- Preset values need consistent conversion and validation for text, number, boolean, date, select, and relation fields.
- JSON authoring is more technical than a visual builder. A valid starter configuration, model and field reference help, path-based validation messages, and the existing format and fullscreen tools must make errors easy for administrators to correct.
- Replacing quick cell editing may slow some existing desktop workflows. A future, more capable inline editor can restore fast table editing while sharing the same submission and automation path.
- Responsive grids can create confusing reading order if implemented with visual reordering. DOM order and keyboard order must remain aligned with template order.

## Open Questions

None.
