# Issue #276: For some reason, creation of a user is not working

Source issue: https://github.com/benitogonzalezh/five-panel/issues/276

## Problem

An administrator cannot open the new-user form reliably. The reported production screenshot at `/alci/settings/users` shows the page dimmed behind a dialog backdrop, no usable form, and `SyntaxError: 10` in the browser console.

Restore the existing user-creation flow by correcting the translated email examples that cause the form to fail while opening.

## Current Behavior

- `frontend/src/views/UserManagementView.vue` translates `userManagement.dialog.create.emailPlaceholder` and `userManagement.dialog.edit.emailPlaceholder` when rendering the corresponding dialogs.
- Both keys contain a raw `@` in each language: `usuario@ejemplo.com` in `es-ES` and `user@example.com` in `en-US`.
- Vue I18n treats `@` as message syntax. These four strings raise `INVALID_LINKED_FORMAT`, whose production error is `SyntaxError: 10`.
- Research confirmed that the exact production bundle shown in the screenshot, `index-w6yQDeGx.js`, contains these strings and that the reported stack enters the translation parser's invalid-linked-format path.
- Running the project's message compiler in production mode reproduces the same error for all four strings. Correcting the `@` in memory lets all four compile and render the intended email examples. The corrected application has not yet been tested through a complete browser creation flow.
- The login and password-recovery translations already use Vue I18n's supported literal format for `@`.

## Desired Behavior

Administrators can open the Create and Edit dialogs in Spanish and English. Each dialog displays an ordinary example email address, remains usable, and can be closed and reopened without a translation error or a stuck backdrop.

An administrator can complete the existing Create flow with valid details and see the new user in the current tenant's user list.

## Scope

- Correct the create and edit email-placeholder messages in `frontend/src/locales/es-ES/userManagement.json` and `frontend/src/locales/en-US/userManagement.json`.
- Add focused regression coverage for compilation and displayed output of the four affected messages.
- Verify the dialogs and normal user creation in both languages using a production build.

## Out Of Scope

- Changing password requirements or aligning the separate password-check gap between the form and server.
- Changing server routes, request fields, response fields, database tables, tenant membership, permissions, or account-linking rules.
- Redesigning the dialogs or changing unrelated translated text.
- Adding a translation framework, dependency, or application-wide translation cleanup.

## Acceptance Criteria

1. In both `es-ES` and `en-US`, an administrator can open the Create dialog from Settings → Users and see the complete form without a translation error in the browser console.
2. In both languages, an administrator can open the Edit dialog for an existing user without a translation error, and its existing editable fields remain usable.
3. The Create and Edit email inputs display exactly `usuario@ejemplo.com` in Spanish and `user@example.com` in English. They do not display literal escape syntax or translation keys.
4. Closing either dialog with Cancel removes the backdrop and leaves the user list usable. Reopening it works without a translation error. Merely opening or cancelling a dialog does not submit a user-creation request.
5. In each language, submitting a new, unused email address, a name, a password that meets the existing server requirements, and matching password confirmation sends one creation request. On success, the dialog closes, the success message appears, and the new user appears once in the current tenant's refreshed list.
6. The existing required-field, email, and password-confirmation checks still prevent invalid submissions. Existing server-error and inactive-user reactivation handling remains available without introducing translation errors.

## Implementation Notes

- Prefer the existing literal-interpolation pattern already used by the login and recovery pages. The intended message values are:

  | Language | Message keys within `userManagement` | Message value | Displayed value |
  | --- | --- | --- | --- |
  | `es-ES` | `dialog.create.emailPlaceholder`, `dialog.edit.emailPlaceholder` | `usuario{'@'}ejemplo.com` | `usuario@ejemplo.com` |
  | `en-US` | `dialog.create.emailPlaceholder`, `dialog.edit.emailPlaceholder` | `user{'@'}example.com` | `user@example.com` |

- This follows [Vue I18n's documented literal interpolation](https://vue-i18n.intlify.dev/guide/essentials/syntax#literal-interpolation) and the repository's earlier login repair in [PR #263](https://github.com/five-panel/five-panel/pull/263).
- `frontend/test/loginView.test.ts` provides an existing example of checking messages with `baseCompile` and rendering them with `createI18n`. Reuse the installed tools and the package-level `frontend/test/` layout.
- Keep the existing `POST /users` request shape: `{ email: string, name: string, password: string }`. Password confirmation remains a form check. No server or database change is needed for this repair.

## Test Expectations

- Exercise all four affected messages with the real message compiler, treating compiler errors as failures. Render them with the real translation runtime and assert the exact displayed email examples. This regression check must fail against the original raw-`@` messages and pass after the correction.
- In a production build, open Create and Edit in each language, check the displayed email example, cancel, and reopen. Observe the browser console for translation errors and confirm that the backdrop does not block the page after closing.
- Complete a valid creation in each language using a unique test email in a test tenant. Check the single creation request, success message, closed dialog, and refreshed tenant user list.
- Check invalid required fields, invalid email, and mismatched password confirmation. Verify the existing error and inactive-user handling through focused tests or browser checks using controlled responses.
- Run the frontend test suite and production build through the existing Docker Compose setup: `docker compose exec -T frontend npm test` and `docker compose exec -T frontend npm run build`.

## Risks

- Incorrect escaping could continue to throw an error or display the formatting syntax to the user. Both compilation and exact-output assertions are required.
- Fixing only Create or only Spanish would leave the same failure in Edit or English. Cover all four messages explicitly.
- A successful message-compilation check does not prove that a user can be saved. The production-build browser checks must include normal creation; the research reproduction alone is insufficient.

## Open Questions

None.
