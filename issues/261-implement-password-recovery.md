# Issue #261: Implement password recovery (¿Olvidaste tu contraseña?)

Source issue: https://github.com/benitogonzalezh/five-panel/issues/261

## Problem

People who forget their password cannot recover access from the login page. Add an email recovery flow that changes the person's single account password across all their workspaces, without revealing whether an email belongs to an account.

The agreed direction is email-only recovery, a single-use link valid for 30 minutes, normal sign-in after the reset, and removal of the user's existing sessions. Use Amazon SES through the existing SMTP/Nodemailer mail service. Keep the provider configurable through the existing mail settings.

## Current Behavior

- The inspected `frontend/src/views/LoginView.vue` points recovery to `/forgot-password`, but `frontend/src/router/index.ts` has no dedicated recovery page. The approved [issue #260 spec](260-improve-login.md) instead leaves recovery visibly unavailable until this issue connects it. Support whichever presentation has landed when implementation starts.
- `SessionService.login` verifies credentials, then offers workspace selection when needed. A single `users` row holds the password; `user_tenants` connects that user to workspaces and holds workspace-specific access and roles.
- `UserService` rejects creating or changing an account to an email already owned by another user. However, the database only enforces uniqueness of `(tenant_id, email)`. Sign-in deliberately handles multiple user rows sharing an email, and the membership migration does not merge them. Their presence in a live database has not been verified.
- Passwords are hashed with Argon2. The existing normal-user password rule is 8–128 characters with at least one digit or one of the special characters accepted by `validatePassword` in `UserService.ts`.
- `SessionRepository` stores session token hashes and supports deletion by user ID. `SessionService.validate` checks for a stored, unexpired session before accepting a session token.
- `MailService` already uses Nodemailer and SMTP with `MAIL_HOST`, `MAIL_PORT`, `MAIL_USER`, `MAIL_PASSWORD`, `MAIL_FROM`, and `MAIL_SECURE`. The separate `/mail/send` admin/test route is disabled by default; recovery does not need that route enabled.
- Graphile Worker already provides database-backed background jobs. Backend and frontend translation systems support English and Spanish. The frontend also loads Clarity from `index.html`, before Vue starts, which matters when protecting recovery codes from tracking.

## Desired Behavior

### Request recovery

The login recovery action opens the public `/forgot-password` page. Ask for an email address only, with a visible label, a submit button, and a working return-to-login action. Do not ask for a workspace, list workspaces, or disclose account details.

For every syntactically valid email, return the same confirmation: “If an account exists for this email, we'll send a password recovery link.” Provide equivalent Spanish wording. This also applies to unknown addresses, disabled accounts, accounts without active workspace membership, duplicate account records, and requests suppressed by the email sending limit.

Queue the request before looking up an account; perform lookup and email sending in the worker. This keeps the public response independent of account existence and SMTP delivery time. Malformed input, an IP request limit, and an infrastructure-wide queue failure may return distinct, localized errors because those checks do not depend on account existence. The confirmation means the request was accepted, not that an email reached an inbox.

Match email using the current sign-in lookup rule after trimming surrounding whitespace. Do not merge email variants or introduce a new case-insensitive identity rule in this issue.

### Resolve the account and send the link

The worker sends recovery mail only when the email identifies exactly one user, that user is globally active, and the user has at least one active membership in an existing workspace. Multiple memberships for the same user are normal and produce one link for that user's one password.

If multiple separate user records match, send no recovery link, do not pick the first or only active record, and write a sanitized internal diagnostic for an administrator. This is safe failure handling; repairing or merging accounts is separate work. Unknown or ineligible accounts are normal no-send results and are not retried as delivery errors.

Generate a cryptographically random 32-byte recovery code in the worker. Store only its SHA-256 hash in the recovery table. Bind the record to the user ID and a fingerprint of the user's email and password hash at issuance. The link expires exactly 30 minutes after issuance, according to server time.

Construct the link using the trusted configured `FRONTEND_URL`, for example:

```text
https://<configured-frontend>/reset-password?lang=es#token=<recovery-code>
```

The code belongs in the URL fragment, so the browser does not send it in the initial HTTP request. Do not use a caller-provided host, workspace domain, return URL, or forwarded host to choose its destination. Production links use HTTPS; local development may use the configured localhost HTTP address.

Send the message in the request's supported language, including the reset action, the 30-minute lifetime, an explanation that the password applies to all of the user's workspaces, and wording that the recipient can ignore an unrequested recovery email. Use text and HTML content. Do not include passwords or list the user's workspaces. Disable provider click tracking for recovery links so the secret is not routed through a tracking URL.

### Set the new password

The public `/reset-password` page reads the code into page memory and removes it from the visible URL before tracking scripts can read it. Never place it in local storage, session storage, cookies, or application logs. Clear it when leaving the reset page or after success. A refresh after URL removal requires reopening the original email link or requesting another link; explain that path clearly.

Show visibly labeled new-password and confirm-password fields, the existing password requirements, and a submit action. Permit paste and password managers, use `autocomplete="new-password"`, and provide accessible password visibility controls. Password confirmation is checked in the browser; the server independently validates the submitted password.

Simply opening a link must not consume it, change credentials, or end sessions. A missing or malformed code shows the unusable-link state immediately. A well-formed but expired, already-used, revoked, or unrecognized code is rejected on submission with the same localized unusable-link message and a working action to request another link. This design does not require a separate token-validation endpoint.

A successful submission changes only the identified user's password, makes the user's outstanding recovery links unusable, and removes all existing sessions for that user across workspaces and devices. These database changes succeed or fail together. No workspace membership, role, active flag, or other account's password or sessions may change.

Show success and an action to sign in with the new password. Do not create a login session automatically or reuse a previously stored redirect destination. Normal sign-in continues to offer workspace selection for the same user when needed. Send a localized password-change notice to the account's email after a committed reset; it contains no password or recovery code. A notification delivery failure must not undo a successful password reset.

Recovery pages remain usable when opened directly, when signed out, and when a browser already has a session. Authorization for the reset comes from the recovery code, never from a workspace URL or the browser's logged-in user.

### Loading, errors, and abuse limits

Disable repeated button/Enter submissions while a request is pending. Show and announce progress, confirmation, validation errors, rate limits, and retryable connection/server errors. Retain the request email after a failure. Retain password fields only in page memory after a retryable failure; clear them on success or when the link is unusable.

Use these initial server-side limits, with shared PostgreSQL-backed counters rather than counters that reset separately in each API process:

| Limit | Behavior when reached |
| --- | --- |
| Recovery requests: 10 per IP per fixed 15-minute window | Return `429` with `Retry-After` and a localized retry message |
| Recovery emails: 3 requests per email per fixed hour | Return the usual `202` confirmation but enqueue no further request for that email during the window |
| Reset submissions: 30 per IP per fixed 15-minute window | Return `429` with `Retry-After` and a localized retry message |

Apply email counters to every well-formed address, including nonexistent ones. Count attempts atomically under concurrent requests. Store an email fingerprint as the counter key rather than copying raw emails into limit records. Do not lock the user's account or change their existing password or sessions when recovery is merely requested. Use the server's trusted client IP handling; do not trust arbitrary forwarded headers.

## Scope

- The login recovery destination, public request/reset pages, and English/Spanish interface and email text.
- Public recovery endpoints, strict input validation, request limits, recovery token storage, expiration, and atomic consumption.
- Worker tasks that reuse the existing mail service for the recovery link and password-change notice, with bounded delivery retries and sanitized failures.
- User-specific password updates, invalidation of recovery links, and session removal, including necessary coordination with sign-in session creation.
- Amazon SES SMTP configuration instructions and delivery verification from the actual hosting environment.
- Automated service, route, database, worker, and frontend checks plus a real browser and email-delivery verification.

## Out Of Scope

- Login visual redesign, Help/Contact destinations, legal pages, or unrelated interface changes.
- Workspace-specific passwords, new workspace membership flows, new roles, sign-up, social sign-in, MFA, or changes to the existing password-strength policy.
- Merging duplicate users, globally changing email matching or uniqueness, or migrating account ownership.
- An email-provider selection interface, tenant-specific sending providers, a general template editor, a new queue system, or an email audit dashboard.
- Provisioning an AWS account, purchasing services, or automatically changing production secrets or DNS in the application change. Document the needed setup and verify delivery to controlled test inboxes before release.

## Acceptance Criteria

1. From login, the recovery action reaches `/forgot-password`. Both recovery routes also work on direct navigation and refresh without being treated as workspace slugs or requiring authentication.
2. Known active, unknown, disabled, membership-ineligible, and duplicate-account emails produce identical `202` bodies for valid, non-IP-limited requests. The public handler queues without an account lookup or SMTP call; an email-limited request also returns that body without enqueueing another job. For any request that needs enqueueing, a queue failure returns `503` without checking whether the account exists.
3. One active user with two active workspace memberships receives one recovery link. A successful reset makes the new password work through normal sign-in for both workspaces, leaves workspace choices and permissions intact, and makes the old password fail. Other users' credentials and sessions remain unchanged.
4. An email matching multiple user IDs, including a mix of active and inactive users, produces no recovery email or credential/session changes and produces the sanitized duplicate diagnostic. A disabled user or a user without active membership receives no recovery email. Eligibility and the email/password fingerprint are checked again when a reset is submitted.
5. Recovery records contain a unique code hash, user ID, credential fingerprint, issuance/expiration times, and use state, but never the raw code. A code works strictly before its 30-minute expiration and fails at or after expiration. Changing the user's email or password after issuance invalidates the link.
6. An email scanner or browser opening the link does not consume it or change account/session state. Missing/malformed codes show an unusable-link state; valid-looking but invalid, expired, used, or revoked codes show it on submission. The request-another-link and return-to-login actions work.
7. A successful reset makes every outstanding recovery link for that user unusable. Two concurrent submissions, using either the same link or different outstanding links for that user, allow exactly one successful password change. A transaction failure leaves the old password, sessions, and link use state intact.
8. Immediately after a committed reset, previously issued session tokens fail authentication in every workspace. A login that verified the old password before the reset cannot insert a surviving session after the reset commits. The reset response contains no login token, and the person signs in through the normal flow.
9. Server-side password validation matches the existing normal-user rule, and the stored new password uses Argon2. Mismatched confirmation or invalid passwords cannot complete a reset. Password-validation failures do not consume an otherwise valid link.
10. The API limits in Desired Behavior hold across concurrent requests and API processes. IP-limited requests receive `429` and `Retry-After`; email-limited requests receive the usual confirmation. Requesting recovery never disables an account, changes its password, or signs it out.
11. Recovery codes and passwords do not appear in application/proxy request URLs, logs, job payloads, tracking requests, or browser persistent storage. The reset-page URL is cleared before third-party scripts can inspect it, and the reset page does not start session recording. Capture HTTP bodies and structured errors safely so Fastify and worker error paths do not log these secrets.
12. Both locales contain all visible text, field labels, accessible names, status announcements, password guidance, and both email types. The queued language controls email wording independently of the worker's default locale. Password-change notices are sent only after a committed reset and contain no credentials.
13. Button and Enter submissions cannot overlap. Loading ends after success or failure; recoverable errors allow retry, unusable links offer a new request, and successful reset offers sign-in. Keyboard focus reaches every control and moves to relevant feedback, with visible focus and announced errors/status changes.
14. At 320px, 390px, 768px, and 1280px widths and at 200% zoom, both locales have no horizontal page overflow and all controls remain reachable by scrolling, including with a phone keyboard open. Recovery pages follow the existing login presentation without redesigning login.
15. A real browser verification follows a delivered recovery email, sets a new password, observes old-session rejection, and signs in with the new password. A separate live Amazon SES test from the intended DigitalOcean host reaches a controlled inbox; record the environment and sanitized outcome in the implementation PR, without exposing secrets or active links.

## Implementation Notes

### Suggested code boundaries

Keep the change local to authentication and mail delivery. Suggested additions are a `PasswordRecoveryService`, a `PasswordResetTokenRepository`, recovery route registration, two Vue views, and worker tasks. Reuse the existing service/repository conventions, object parameters, Kysely transactions, PrimeVue controls, translation loading, and Graphile Worker. Class/file names are guidance; the behaviors and contracts below are requirements.

Relevant starting points:

- `backend/src/services/SessionService.ts`, `UserService.ts`, `MailService.ts`, and `context.ts`.
- `backend/src/repositories/UserRepository.ts`, `UserTenantRepository.ts`, and `SessionRepository.ts`.
- `backend/src/routes/`, `backend/src/index.ts`, `backend/src/worker/taskList.ts`, and `backend/src/services/GraphileWorkerService.ts`.
- `backend/src/db/types.ts` and a new migration using the next available migration number.
- `frontend/src/views/LoginView.vue`, new recovery views, `frontend/src/router/index.ts`, `frontend/src/stores/authStore.ts`, and the locale files.
- `frontend/index.html` for protecting the reset URL before the existing early Clarity loader. Removing the code only after a Vue component mounts is too late.

Extract or reuse the existing normal-user password validator rather than writing a different rule for recovery. Add the public recovery routes before tenant fallback handling. Use the frontend's configured API URL and send its selected language through `Accept-Language`; save the resolved `en`/`es` language with each worker request.

### API contract

Both endpoints are public JSON routes and must not accept caller-selected user IDs, workspace IDs, recipient overrides, destinations, or email content.

| Endpoint | Request body | Responses |
| --- | --- | --- |
| `POST /auth/forgot-password` | `{ "email": "person@example.com" }` | `202 { "status": "accepted" }`; `400` for invalid input; `429` for the IP limit; `503` if the request cannot be queued |
| `POST /auth/reset-password` | `{ "token": "<code>", "password": "<new-password>" }` | `200 { "status": "password_reset" }`; `400` with `INVALID_REQUEST`, `PASSWORD_INVALID`, or `RESET_LINK_INVALID`; `429` for the IP limit; `503` for retryable infrastructure failure |

Errors use a stable machine-readable code and localized message, for example `{ "code": "RESET_LINK_INVALID", "message": "This recovery link is invalid or has expired. Request a new link." }`. Do not distinguish expired from used or unrecognized codes. Send `Cache-Control: no-store` on recovery responses. Define and enforce bounded body and string sizes; email is at most 255 characters and the password remains at most 128. Encode codes as unpadded base64url and validate their exact expected length. Reject unexpected input properties.

### Data shape and lifecycle

Use a new `password_reset_tokens` table, mapped to the project's camel-case TypeScript names:

| Column | Shape and purpose |
| --- | --- |
| `id` | UUID primary key |
| `user_id` | Required foreign key to `users.id`, cascade on user deletion |
| `token_hash` | Required, unique SHA-256 hash of the random code |
| `credential_fingerprint` | Required SHA-256 fingerprint of an unambiguous encoding of the email and password hash at issuance |
| `created_at` | Required timestamp with time zone |
| `expires_at` | Required timestamp with time zone, 30 minutes after issuance |
| `used_at` | Nullable timestamp with time zone |

Index user ID for revoking outstanding links and expiration for cleanup. There is no workspace ID: this is a user credential. Multiple outstanding links may coexist until a reset succeeds or their fingerprints/expiration become invalid. Merely requesting another link must not revoke a previously delivered link.

Keep request-limit storage small: hashed key, limit kind, fixed-window start, and count are sufficient, with uniqueness on kind/key/window and atomic increments. Use a bounded periodic cleanup through the existing worker for expired limit windows and expired/used recovery records; a daily task with a maximum seven-day retention is sufficient. Do not create a general rate-limit framework.

### Background delivery

Queue only the request email, resolved language, a request ID, and request time. Never queue a raw recovery code, password, or rendered recovery email. The worker generates the secret and renders the message in memory. Discard recovery jobs more than 30 minutes old before creating a link. Use at most three delivery attempts per job; do not retry a completed request simply because the account was unknown or ineligible.

On a retry after an uncertain SMTP result, a fresh code may be issued; earlier delivered links remain usable until one reset succeeds or their normal expiry. Never log a raw provider error that may contain a message, URL, or SMTP credentials; use the existing sanitized error summary. Check the transport's accepted/rejected recipient result instead of treating every resolved send call as accepted delivery.

Arrange password-change notification enqueueing as part of the successful database commit, using the existing database-backed worker mechanism. Queue non-secret user/destination and language information, not a password or code. This prevents a process crash between the committed reset and notification scheduling from silently losing the notice. Delivery occurs after the transaction, so SMTP failure cannot roll back the password change. No new general-purpose email outbox is required.

### Atomic reset and session creation

Serialize resets for the same user using a user-row lock, or an equivalent database mechanism. In the transaction, recheck the live user, matching-email uniqueness, active membership, code hash/use/expiration, and credential fingerprint before changing any credential. Password update, use/revocation of recovery records, deletion of the user's sessions, and notification scheduling belong to that one commit. Recheck expiration at consumption, not only before expensive password hashing.

Deleting existing sessions alone is insufficient if a login is already in progress. Coordinate sign-in's final session insertion with the same user lock and recheck that the password hash used for verification is still current. After a reset wins that lock, an old-password login must fail or repeat verification against the new hash. Keep the existing successful login response, tenant-selection behavior, and session format unchanged. Repository methods used inside this transaction must use the transaction connection.

### Provider and deployment

Use Amazon SES SMTP as the initial production provider, not an SES-specific frontend/API contract or a new SDK dependency. The earlier [mail-provider comparison on PR #215](https://github.com/five-panel/five-panel/pull/215#issuecomment-5410509998) identified SES as the lower-cost option with more setup; the spec discussion selected that direction.

Document these existing settings in the maintained development/production environment examples and deployment instructions, without placing real secrets in the repository:

```dotenv
MAIL_HOST=email-smtp.<selected-aws-region>.amazonaws.com
MAIL_PORT=2465
MAIL_SECURE=true
MAIL_USER=<ses-smtp-username>
MAIL_PASSWORD=<ses-smtp-password>
MAIL_FROM=Five Panel <noreply@verified-domain.example>
FRONTEND_URL=https://<trusted-five-panel-frontend>
```

Use an SES region with a verified sender domain, region-specific SMTP credentials, and approved production access before allowing recovery for real users. Add the DNS records AWS provides for sender authentication. Credentials must have the permissions needed to send these messages; ordinary AWS login credentials are not SMTP credentials. Configure provider tracking so recovery links are not rewritten. `MAIL_SEND_ENDPOINT_ENABLED` can remain false.

DigitalOcean documents default blocks on ports 25, 465, and 587 for Droplets. SES supports implicit encrypted SMTP on port 2465, which fits `MAIL_SECURE=true` without changing the current transport. Confirm connectivity from the actual host; do not assume that an unlisted port is reachable. Missing credentials, sandbox restrictions, and blocked connectivity are rollout blockers, not reasons to switch silently to an unauthenticated or unencrypted transport.

Do not copy provider prices into product behavior or add an AWS tenant for each Five Panel workspace. This feature uses one application sender and the existing generic SMTP configuration.

### Research supporting the design

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html): consistent responses, background delivery or otherwise consistent timing, random expiring single-use codes, request limits, normal sign-in after reset, and removal of existing sessions. These fit the existing database and worker; a new signed-token authentication system is unnecessary.
- [Amazon SES production access](https://docs.aws.amazon.com/ses/latest/dg/request-production-access.html) and [verified identities](https://docs.aws.amazon.com/ses/latest/dg/verify-addresses-and-domains.html): sender verification and sandbox removal are deployment prerequisites.
- [Amazon SES SMTP connections](https://docs.aws.amazon.com/ses/latest/dg/smtp-connect.html): port 2465 supports encryption from connection start. This fits the existing Nodemailer SMTP transport.
- [DigitalOcean SMTP restrictions](https://docs.digitalocean.com/support/why-is-smtp-blocked/): standard SMTP ports are blocked on Droplets by default, so actual host connectivity must be checked.

## Test Expectations

- Keep new TypeScript/JavaScript tests in `backend/test/` and `frontend/test/`, following repository instructions. Extend existing membership/session tests where behavior changes rather than replacing the current sign-in coverage.
- Service and route tests cover both endpoint contracts, schema limits, equal public responses, queue outages, active/inactive/missing/duplicate accounts, a user shared across workspaces, fingerprint changes, and the stated IP/email limits. Use injected clock, random-code generator, mail transport, and queue boundaries where useful.
- Real PostgreSQL integration tests cover migration apply/rollback, hash uniqueness, fixed-window counter contention, user deletion, two simultaneous resets with the same and different links, transaction rollback, all-workspace session revocation, and the login/reset race. Mocked repositories alone do not establish these guarantees.
- Worker tests verify that raw codes/passwords never enter job payloads, language survives queueing, stale requests are discarded, no-send cases finish without retry, SMTP rejection/failure has bounded retries, and password-change notification scheduling survives a successful reset commit. Include accepted/rejected recipient results and safe error summaries.
- Frontend tests cover public routing before workspace fallback, fragment extraction and removal before analytics starts, refresh/missing-token handling, password confirmation and rules, duplicate-submit prevention, all result states, retry paths, translations, and keyboard/focus behavior. Keep existing login/workspace-choice coverage passing.
- Run the focused checks during development, then the backend and frontend test suites and builds. Run `git diff --check`. Apply the migration through the root ops CLI and use Docker Compose for normal local runtime.
- A real Playwright/browser flow must use the actual API, database, worker, SMTP delivery to a controlled test inbox, and received link rather than a mocked success response or a token copied from database/log output. Cover both languages, phone/desktop layouts, new-password sign-in, old-password failure, unusable-link recovery, and a session opened before reset becoming unauthorized. A local SMTP inbox can make this repeatable; it does not replace the separate SES delivery check from DigitalOcean.
- Record sanitized verification results and any screenshots in the implementation PR. Hide passwords, credentials, recovery codes, active links, and unrelated customer data. A network/worker/email configuration failure must be reported as an incomplete delivery check, not as a passed end-to-end test.

## Risks

- SES domain verification, production-access approval, DNS setup, and connectivity from DigitalOcean are external deployment prerequisites. They have not been performed while preparing this spec. Application tests cannot prove production inbox delivery.
- SMTP acceptance does not guarantee inbox placement. Delivery limits, rejections, or spam filtering can prevent a person from receiving the email. Keep the generic confirmation, allow a later request, and expose sanitized worker failures to operators without disclosing account existence to the requester.
- The database may contain separate accounts sharing an email; this has not been checked against live data. Those accounts deliberately cannot use self-service recovery until an administrator resolves the ambiguity. This issue must not merge or silently select them.
- A shared account's password reset signs out all its workspaces. State this in the recovery message and password form; workspace membership and permissions stay unchanged.
- Queue retries may produce more than one recovery email after uncertain delivery. Per-email request limits and bounded attempts constrain this; each link remains user-bound and all are invalidated by a successful reset.
- Early Clarity loading and concurrent session creation are existing integration points that can defeat otherwise correct recovery code. Verify secret protection before analytics initialization and the login/reset database race explicitly.

## Open Questions

None. Production SES setup and delivery verification are release prerequisites described above, not unresolved product decisions.
