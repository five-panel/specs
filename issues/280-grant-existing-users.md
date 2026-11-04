# Issue #280: Grant existing users access to another tenant through the Create user flow

Source issue: https://github.com/benitogonzalezh/five-panel/issues/280

## Problem

An administrator cannot add someone to their company when that person's email already belongs to an account in another company. The current Create user action rejects the email, even though one account can already belong to several companies.

Reuse the existing Users page to create an account for a new email or grant an existing active account access to the current company. A tenant means a company. An account holds the shared name, email, and password; a membership holds access and administrator status for one company.

## Current Behavior

- `frontend/src/views/UserManagementView.vue` asks for email, name, password, and password confirmation together. Its service always sends `{ email, name, password }` to `POST /users`.
- `backend/src/index.ts` requires email, name, and password for that route. It uses `authenticate` and `requireSuperuser`, where superuser means an administrator in the authenticated company.
- `UserService.create` creates a new account and membership, but rejects an existing account from another company with `Email already in use`. `UserServiceMembership.test.ts` explicitly tests that rejection.
- `user_tenants` already stores company-specific `isActive` and `isSuperuser` values and has a unique `(user_id, tenant_id)` constraint. The account's original `users.tenant_id` remains separate from its memberships.
- `SessionService` verifies the account password and resolves active memberships. `LoginView.vue` already offers company selection when several memberships are available.
- Deactivate turns off only the current membership. Activate and Reactivate reuse that membership and enforce existing restrictions on globally inactive accounts.
- Account edits are already restricted by `canModifyGlobalUserForTenant`: the current company must be the account's original company, and there must be no active memberships outside it. The Edit dialog still presents those protected account fields as editable. Reactivation text can also promise a password update that the shared-account protection prevents.
- The creation route returns the service's account object directly. Public responses for this work must explicitly exclude `passwordHash` and other internal account fields.
- The rejection and these constraints were confirmed by reading the service, routes, and tests. No new browser reproduction or implementation verification was performed during spec research.

## Desired Behavior

### Create or grant access

1. An administrator opens the existing New user dialog at `/:tenantSlug/settings/users`. Initially, show an email field and Continue.
2. Continue validates the email and performs a read-only, administrator-only email check. This action does not create or reactivate anything.
3. For a new email, show the current name, password, and password-confirmation fields and a Create user action. Preserve the existing server password rules and form confirmation check.
4. For exactly one eligible, globally active account without a membership in the current company, show: "This account already exists. It will be added to this company and will keep its password." Show Add to company, without editable name or password fields.
5. Saving the existing-account path creates only an active membership in the authenticated current company, with ordinary-user defaults. Preserve the complete shared account, its original company association, and all other companies' access and permission records.
6. For active access already present in the current company, show a clear already-has-access message and do not offer another grant.
7. Changing the email clears the checked result and any password values. Ignore late responses for a previous email or a closed dialog. Show loading and failure states, allow retry, and prevent duplicate submissions while a request is pending.
8. After a successful save, close the dialog, refresh the company user list, and report whether an account was created or access was granted. The server's actual result determines the message.

### Inactive access, removal, and editing

- An inactive membership in the current company follows the existing explicit reactivation confirmation. Reuse its membership ID, role, and company permissions; do not insert a replacement membership or silently reset its role.
- For shared accounts whose details the current administrator cannot edit, reactivation asks only to restore company access and explains that the account details and password stay unchanged. For accounts editable under the existing policy, retain the current ability to explicitly provide a name or password during reactivation. Keep all existing global-account reactivation restrictions.
- An inactive account outside the current company remains unavailable. Do not create access, reactivate the global account, or expose a reactivation user ID. Multiple account rows owning the email remain unsafe to link; do not select an arbitrary account or bypass the existing duplicate-owner protection.
- Keep the current Deactivate action: it disables only this company's membership. The user loses access to that company, including through an existing session on the next protected request. Other active memberships and their sessions continue working. A new login omits the disabled company from the existing company-selection flow.
- In Edit, show name and email as read-only and hide password inputs when the existing server policy disallows account changes. Explain that these details belong to the shared account. Company-specific administrator status and assigned views remain manageable through their existing flows.
- Determine editability from the existing owner-company and active-membership policy, rather than inventing a permanent shared-account flag. For an account active in both A and B, neither company's administrator can change its shared details through this screen. Preserve the existing policy as memberships later become inactive.

All new text and changed confirmations must work in English and Spanish. Reuse the existing dialog components, keyboard behavior, and translation patterns, including the email-placeholder repair covered by issue #276.

## Scope

- Add the email-first branch to the current Create user dialog and frontend service/types.
- Add a restricted, read-only email-check route and extend the existing creation action to grant company access.
- Expose the existing account-edit policy to the Users screen, and align Edit and reactivation fields and text with it.
- Use explicit public user responses for the create, edit, deactivate, activate, and reactivate actions used by this screen.
- Preserve company-specific removal, permissions, reactivation rules, and account protections, with focused regression coverage.
- Reuse the existing account, membership, and login structures. No database migration is expected.

## Out Of Scope

- A global superadmin dashboard or a new way to edit shared credentials or globally reactivate protected accounts.
- Invitations, email verification, password resets, account merging, or tenant creation.
- Changes to `createFirstUserForTenant`, tenant bootstrap scripts, or Zipp-specific setup.
- Copying or editing another company's roles, model/view permissions, or sidebar assignments.
- Changing email matching or normalization, merging legacy duplicate email owners, or adding a new global email-uniqueness policy.
- Redesigning login, company selection, the wider settings area, or unrelated API responses.
- Changing the existing rules that decide whether account details or a globally inactive account may be updated. This work exposes those rules accurately in the form.

## Acceptance Criteria

1. Continuing with a valid new email shows name and password fields without writing data. Saving valid details creates exactly one account and one active ordinary-user membership in the authenticated company. The supplied initial password works at login.
2. Continuing with an eligible existing active email shows the account-exists explanation and Add to company without name or password inputs. Saving adds exactly one membership for the existing account ID, with `isActive: true` and `isSuperuser: false`.
3. After a grant, the existing account row, including name, email, password hash, original `tenantId`, flags, and audit fields, is unchanged. Other memberships, roles, permissions, and sidebar assignments are unchanged.
4. A direct administrator request containing an existing email and valid replacement name/password grants access without changing either account field. An email-only request for a new account fails validation without creating an account or membership.
5. The existing password allows login to both companies through the existing selection flow. The session for each company uses that company's role. Administrator status in A does not make the new membership an administrator in B.
6. The added user appears once in B's user list. The response reports B's membership state and tenant ID. Normal permission defaults apply in B, and granting or later changing B permissions does not change A's permissions.
7. Checking an email with active access in the current company shows an already-has-access message. Repeated or simultaneous grants for the same account/company leave one membership, preserve its role, and return a clear result rather than an unhandled database error.
8. An inactive membership in the current company offers explicit reactivation and reuses the existing membership. Cancelling changes nothing. Shared-account reactivation leaves name/password unchanged and does not show inputs or text promising an ignored password change. Editable accounts retain their existing explicit update path.
9. An inactive external account or ambiguous email ownership cannot be linked or globally reactivated by this flow. Responses do not disclose external account IDs, password hashes, or other company details. Preserve the duplicate-owner check before exposing a local inactive-user reactivation hint.
10. Unauthenticated email checks and mutations return 401; authenticated non-administrators receive 403. A client-supplied tenant ID, account ID, or role cannot redirect a grant or bypass the current company's permissions. The server uses the authenticated current tenant.
11. Deactivating the added membership in B blocks B access on the next protected request and removes B from a subsequent login's company choices. A access, credentials, and an A session still work. Reactivating B restores its membership with the existing role and permissions. The existing self-deactivation protection still applies.
12. For an account active in A and B, Edit in either company shows protected account details as read-only, omits password controls, and explains the restriction. An administrator can still change company-local administrator status and assigned views. A forged request changing protected name, email, or password is rejected without changing either company's account or membership data.
13. The account-edit capability agrees with the existing owner-company and active-membership checks, including after memberships are deactivated. A normal editable account in its original company retains its current edit behavior. A role-only edit still observes the existing last-administrator protection.
14. Create/grant and the related user-management mutation responses contain only the documented public user fields and operation metadata. They do not include `passwordHash` or internal account/audit fields.
15. An email change invalidates the earlier check; late lookup responses cannot select the wrong path. Lookup/save failures leave the dialog usable for retry. Saving rechecks current account/membership state, and the displayed result reflects the actual server outcome if it changed after the lookup.
16. Browser checks in English and Spanish cover new-account creation, granting access, duplicate access, reactivation, shared-account editing, company-local deactivation, and login selection. Dialogs can be cancelled and reopened without stale passwords, a stuck backdrop, or translation errors.

## Implementation Notes

### Data and API shape

The following is the proposed small extension to the existing user API. Keep the rules below if naming or internal factoring is adjusted to match repository conventions.

| Surface | Request | Public result |
| --- | --- | --- |
| `POST /users/check-email` | `{ email: string }` | `{ status: 'new' \| 'existing' \| 'already_member' \| 'inactive_member' \| 'unavailable' }`; only `inactive_member` may also return the current-company `userId` and `canEditAccountDetails`. |
| `POST /users` | `{ email: string, name?: string, password?: string }` | The existing public user fields plus `outcome: 'created' \| 'access_granted'`. |
| `GET /users` | Existing request | Existing current-company list fields plus `canEditAccountDetails: boolean` for each row. |
| Existing edit/deactivate/activate/reactivate routes | Existing requests | Explicit public user fields; preserve the current inactive-user reactivation contract. |

- `new` means no matching account. `existing` means one globally active account without a current-company membership. `already_member` refers to active current-company access. `inactive_member` refers only to an existing local inactive membership that passes the current duplicate-owner checks. `unavailable` covers external inactive accounts and ambiguous ownership without exposing their records. Preserve the existing rejection precedence where a local active membership already blocks creation.
- Public user fields are `id`, `email`, `name`, `tenantId`, `createdAt`, `isActive`, and `isSuperuser`. Membership flags and returned `tenantId` describe the authenticated company. Capability and outcome fields are explicit additions; do not serialize the full repository account object.
- Use the existing email syntax and length constraints for the check. Creation still validates name/password for genuinely new accounts. Keep password confirmation in the frontend. Existing-account requests never validate a replacement password as a prerequisite for an email-only grant or call the account update/password-hashing path.
- Both the check and creation routes require `authenticate` and `requireSuperuser`, plus a valid authenticated tenant. The check is read-only. Restrict creation input to supported fields; reject or ignore unexpected tenant/role/account selectors so they cannot affect the target.
- Keep successful creation/grant responses at the existing success status. Use a clear conflict response for active duplicate access, including insert races. Preserve `409 { error: 'INACTIVE_USER_EXISTS', message, userId }` for the existing local reactivation path. Missing new-account details and blocked ownership cases remain validation/business errors. Use stable error codes where the form needs a branch and existing translated messages for display.
- `canEditAccountDetails` is computed by the server for the current actor and target membership using the same policy as account updates. It is false for callers without administrator rights. The client omits protected account fields from membership-only edits, and server checks remain authoritative if the capability becomes stale.

### Service and repository guidance

- Change ordinary `UserService.create` behavior for the one eligible existing-account case. Keep `createFirstUserForTenant` behavior and tests separate.
- For a new account, retain atomic account-and-membership creation and existing password-strength validation/hashing. For an existing account, insert only its local membership, with the acting administrator as membership creator/updater.
- Reuse the unique membership constraint to handle repeated/concurrent inserts safely. Do not use a conflict handler that overwrites the role or reactivates an existing inactive membership.
- Keep the original `users.tenant_id` and all global account fields intact when granting access. Return the account combined with the new membership for the current company, rather than reporting the original company's role or tenant ID.
- Keep lookup and save decisions aligned, but always resolve the current state again at save time. Treat missing or stale lookup information as a form state to refresh, never as permission to bypass service checks.
- Reuse the existing account-edit policy rather than duplicating different rules in the frontend. Its current rule is original-company ownership and zero active memberships outside the current tenant, accounting for whether the current membership is active.
- Keep protected reactivation membership-only. A globally inactive account that fails the existing reactivation policy still cannot be restored by this tenant administrator. Do not add a new platform-admin bypass.

Relevant files are `backend/src/index.ts`, `backend/src/services/UserService.ts`, `backend/src/repositories/UserRepository.ts`, `backend/src/repositories/UserTenantRepository.ts`, `frontend/src/views/UserManagementView.vue`, `frontend/src/services/UserService.ts`, `frontend/src/types/User.ts`, and the existing backend/frontend user-management translations. The membership migration and login code are references, not expected redesign targets.

### Industry pattern and fit

[Auth0 documents assigning an existing account to an organization separately from account creation](https://auth0.com/docs/manage-users/organizations/configure-organizations/assign-members). [OWASP recommends binding tenant operations to a server-verified identity and current membership](https://cheatsheetseries.owasp.org/cheatsheets/Multi_Tenant_Security_Cheat_Sheet.html). These patterns fit Five Panel's existing accounts and memberships: add a membership while preserving shared credentials, and authorize the operation in the authenticated company. A new identity provider, invitation system, or separate administrative product is unnecessary for this issue.

## Test Expectations

- Extend `backend/test/services/UserServiceMembership.test.ts`: replace the ordinary-create rejection of an eligible external active account with grant coverage. Assert no account create/update/hash change, correct current-company return values, ordinary-user role despite an administrator source membership, and untouched other-company permissions.
- Keep coverage for external inactive accounts, ambiguous email owners, protected shared-account edits/reactivation, absent tenant memberships, transactional new-account creation, and the separate bootstrap flow. Add capability checks for owner/non-owner companies and active/inactive membership combinations.
- Add focused service/repository or database-backed checks for active duplicates, concurrent grants, and failed creation rollback. Verify the unique membership count and that a duplicate does not change an existing role or reactivate inactive access.
- Add executable HTTP coverage for the email check and save paths, conditional new-account validation, supplied name/password on a grant, 401/403, forged tenant/role selectors, duplicate conflicts, unchanged inactive-user responses, and safe response fields across the user-management mutations. Source-text assertions alone do not establish these API behaviors.
- Extend frontend tests under `frontend/test/` for both paths, email-check loading/errors, late responses and email changes, duplicate prevention, safe reactivation text/fields, protected Edit fields, correct outcome messages, and translated messages. Keep real message compilation/output checks for affected translations.
- Use real browser checks with two test companies and known credentials. Verify both normal form paths, one user row per company, company-local roles/views, existing-password login and company selection, deactivate/reactivate isolation, shared-account Edit, and an existing session losing only the deactivated company's access. Exercise English and Spanish.
- Run the focused tests, backend/frontend package suites, and builds through the existing Docker Compose workflow. For the eventual implementation, relevant full checks are `docker compose exec -T backend npm test`, `docker compose exec -T backend npm run build`, `docker compose exec -T frontend npm test`, and `docker compose exec -T frontend npm run build`. These are implementation expectations; a spec-only PR does not claim they have been run.

## Risks

- An email check can disclose account existence. Limit it to authenticated company administrators, accept a single exact email, and return the minimum branch information without an external profile or company list.
- Reusing a global account can accidentally expose its password hash or update shared credentials. Explicit public response fields and assertions that the account row is unchanged are required, including the related edit/reactivation responses.
- An account or membership can change between lookup and save. Authoritative save checks, the unique membership constraint, and accurate server outcomes prevent stale form state from overwriting accounts or duplicating access.
- New membership defaults and restored membership settings differ. Grants start with ordinary access; reactivation preserves that company's existing settings. Neither operation copies another company's permissions.
- A frontend-only edit restriction can be bypassed or become stale. Reuse the existing server policy and enforce it on every mutation. Keep visible reactivation claims consistent with what that policy permits.
- Legacy duplicate owners and globally inactive accounts have existing protections. This feature must not weaken those protections while enabling the normal single active-account case.

## Open Questions

None.
