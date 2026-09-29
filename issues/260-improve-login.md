# Issue #260: Improve Login

Source issue: https://github.com/benitogonzalezh/five-panel/issues/260

## Problem

Replace the current login presentation with the approved responsive design and supplied assets, while preserving the existing sign-in flow.

The design source is [five-panel-login-handoff.zip](https://github.com/user-attachments/files/32784539/five-panel-login-handoff.zip), attached to the source issue. Its `HANDOFF.md`, `design-tokens.json`, `assets/rotation.json`, and four files in `screens/` provide the visual reference. The user approved the plan and the link exception below during spec discussion.

## Current Behavior

`frontend/src/views/LoginView.vue` shows a dark blue page, large heading, email and password controls, and a large logo. It uses PrimeVue controls and the existing authentication store.

`authStore.login` sends email and password to `POST /auth/login`, optionally including `tenantSlug`. Accounts belonging to several workspaces first receive workspace choices. Selecting one submits the credentials with its slug. Successful login stores the session, resets cached data and views, identifies the signed-in user for existing analytics, and follows the redirect query or opens the workspace home.

The page supports English and Spanish, password visibility, loading, and credential/server error messages. Workspace choices support focus management and are cleared when credentials change. The current recovery link points to `/forgot-password`, but the router has no dedicated recovery page. No existing destinations were found for Help, Contact, Privacy, or Terms.

## Desired Behavior

### Layout and branding

Reproduce the supplied cream-and-green design with the cap SVG beside “Five Panel,” persistent field labels, a welcome heading, sports photography, and the common two-line message “Todo en su lugar. Tú, a lo tuyo.” Do not show sport names or add motivational copy.

Use the supplied Spanish wording and equivalent English translations through the existing locale system. Keep the brand name unchanged. Use Inter for the interface and wordmark and Lora 400 for the photo message. Keep text as text and use sensible fallback fonts while fonts load.

Use these reference layouts, allowing the page to grow and scroll:

| Reference | Composition | Key dimensions |
| --- | --- | --- |
| 1440 × 960 | Photo left; brand/help, centered form, and footer right | Outer padding 24px; gap 24px; photo 672 × 912px; right inner padding 48px; photo radius 16px |
| 1024 × 768 | Two columns | Outer padding 20px; gap 24px; photo 440 × 728px; right inner padding 24px; photo radius 12px |
| 768 × 1024 | Brand/help, photo strip, centered form, footer | Outer padding 32px; main gap 28px; photo 704 × 224px; radius 12px |
| 390 × 844 | Compact stacked layout | Outer padding 24px; main gap 24px; photo 342 × 158px; radius 10px |

Start with stacked layouts below 960px and two columns from 960px; use the compact stacked treatment through 600px. Adjust intermediate sizing where needed without losing the four reference compositions. Form width is at most 384px and shrinks with available space. Inputs and the main button are 52px high with 7px corners. Use the handoff tokens, including background `#FAFAF8`, brand `#263E31`, white inputs, and input border `#D9DDD5`.

Keep the approved light login appearance in both app theme modes without changing the saved app theme or other screens. Preserve visible focus and readable error states.

### Photos

Choose randomly from photos 01–09 once per login-page mount. Repeated visits may select the same photo; there is no timer, saved choice, or rotation history. Keep that selection through edits, validation, failed requests, and workspace selection.

Retain coastal photo 00 for deterministic visual reference checks; it is outside the normal nine-photo selection pool. All ten assets must have verified crops in wide, tablet-portrait, and compact layouts. Preserve faces and the main sporting action and keep the message readable using the handoff overlays. The coastal narrow crop starting points are `50% 23%` on mobile and `50% 36%` on portrait tablet; other photos may need their own positions.

Serve an optimized size of only the selected photo, reserve its layout space, and avoid preloading the collection. A slow or failed photo must not block login, move the form, or make the message unreadable. Treat photos and the cap beside the visible wordmark as decorative for screen readers.

### Form and links

Preserve the existing login contract, session storage, workspace selection, redirect behavior, cache resets, and analytics timing. Fit workspace options and their back action into the new design, allowing long names and lists to wrap and scroll. Preserve existing focus movement and stale-response handling.

Use associated visible labels and password-manager autocomplete (`username` for email and `current-password` for password). The password visibility control must work with a keyboard and expose an accessible name and state. Keep required-field/email validation, loading feedback, and distinguish invalid credentials from server/network failure. Retain entered values after failed login. Block repeated button or Enter submissions while a request is pending.

Show password recovery, Help, Contact (“Hablemos”), Privacy, and Terms in the positions from the design, but leave them without destinations. This is an explicit user-approved exception to the handoff's requirement for working destinations. Render them as unavailable link text with appropriate accessible semantics; clicking or pressing Enter must not navigate, reload, jump the page, or submit the form. Do not use an active empty `href`, `#`, or the current broken recovery path. They must not appear as working keyboard actions.

Password recovery is tracked separately in [issue #261](https://github.com/five-panel/five-panel/issues/261), labeled `ready for spec`. It will connect the recovery link when implemented.

## Scope

- Login presentation, supplied logo and photos, optimized image variants, and page-specific styling.
- English and Spanish login text, including the shared photo message and new labels.
- Responsive layouts, accessibility, photo selection, and form state presentation.
- Preservation and verification of existing authentication and workspace-selection behavior.
- Unavailable link presentation as agreed above.

## Out Of Scope

- Password recovery implementation, email delivery, and reset tokens; these belong to #261.
- Help/contact destinations, legal pages, or their content.
- New authentication providers, sign-up, roles, session policy, or backend changes.
- Changes to the database or API shape.
- Restyling other screens, replacing the global app font, or changing tenant themes.
- Timed photo carousels, transition effects, selection persistence, or a configurable photo-management system.

## Acceptance Criteria

1. At all four reference sizes, a Spanish login with coastal photo 00 follows the supplied composition, spacing, typography, colors, and branding. Any deviations needed for accessibility are documented in the implementation PR.
2. At widths 320, 360, 601, 834, 960, and 1280px, neither locale has horizontal page overflow. At reduced heights, 200% zoom, and with a mobile/tablet keyboard open, all form controls and workspace options remain reachable by scrolling.
3. The login retains its approved light appearance when the app has either light or dark mode selected, and does not alter the saved theme.
4. Each of photos 01–09 can be selected on page entry. The chosen photo remains unchanged during typing, failed login, and workspace selection. Failed login and workspace selection do not clear entered credentials. Photo 00 can be selected in test setup for reference comparisons without adding a public photo selector.
5. All ten photos have verified crops in each of the three compositions; faces, the main action, and the common message are visible. No sport names appear on the page.
6. Network inspection shows no downloads of unselected photos. Image sizes suit the rendered area. Slow or failed image loading leaves the reserved photo area stable and the form usable.
7. Email and password have visible, associated labels and appropriate autocomplete. Keyboard users can reveal and hide the password, identify the control's state, and see focus. Decorative images do not duplicate spoken content.
8. Successful single-workspace and multiple-workspace login preserve existing session and redirect behavior. Workspace selection, credential changes, and the back action preserve existing focus and stale-response handling.
9. Empty or malformed required fields cannot submit. While a request is pending, repeated clicks and Enter presses produce no additional login requests. A failed request clears loading, shows the appropriate localized error, and allows retry without clearing credentials.
10. Password recovery, Help, Contact, Privacy, and Terms are visible but unavailable. Mouse and keyboard activation do not navigate, reload, jump, or submit. Assistive technology receives their unavailable state, and keyboard navigation does not present them as working actions.
11. English and Spanish contain all new visible and accessible text, with no missing translation keys. The visible brand remains “Five Panel.”

## Implementation Notes

- Start in `frontend/src/views/LoginView.vue`, `frontend/src/locales/en-US/login.json`, and `frontend/src/locales/es-ES/login.json`. Reuse PrimeVue controls and existing authentication functions.
- `frontend/src/stores/authStore.ts` and `frontend/src/router/index.ts` describe the behavior to preserve; this spec does not require changes to their contracts.
- Follow `docs/agent/frontend-theming.md`. It explicitly permits standalone branded styling on the login page. Keep any new font and color treatment local to login rather than changing global app typography or partially inheriting dark app colors.
- Reuse the project's font-loading mechanism with Inter and Lora and fallbacks. No new design system or general-purpose theme layer is needed.
- Preserve the supplied SVG's internal space; keep approximately 9px between its 32px canvas and the wordmark. Use the supplied image files rather than generating replacements or screenshotting the mockups.
- A small static photo list containing asset paths and composition-specific crop positions is sufficient. Optimized sources can use normal responsive image markup. No runtime API or database model is needed.
- Validate supplied archive checksums when importing assets. The canvas snapshot is reference material, not production code.
- Tests may control image selection to inspect each crop and reproduce the supplied screens; production does not need a new configuration setting.

## Test Expectations

- Add focused behavior coverage for one selection per page mount, stable photo and credentials across errors, duplicate-submit prevention, unavailable links, and workspace-selection interactions.
- Run relevant existing frontend authentication tests, including `frontend/test/authStore.test.ts` and `frontend/test/clarityAuthFlow.test.ts`, and the frontend build. Keep new automated tests under `frontend/test/`.
- Verify actual successful login, multiple-workspace selection and back navigation, redirect handling, credential errors, network/server failure, and retries in the browser.
- Compare against all four supplied screens using photo 00. Inspect every photo in wide, portrait-tablet, and compact layouts; verify both languages and both saved app theme modes.
- Check keyboard order, focus visibility, accessible names/states, password autofill, zoom, reduced heights, mobile/tablet keyboards, and long workspace names/lists.
- Use browser network controls to verify selected-image-only loading, responsive image sizing, stable layout, and image failure behavior.
- This PR is a specification only. Application tests and browser checks above are requirements for the implementation PR, not claims of completed validation.

## Risks

- The supplied rotation photos have not been validated at narrow crops; individual crop positions may be necessary.
- English text, errors, and long workspace lists need more space than the Spanish resting-state mockups. Layout must grow instead of clipping controls.
- Global PrimeVue dark styles and the existing app font can leak into the branded page unless styling is scoped carefully.
- Large original photos or font loading can slow first display. Optimize assets and keep the form usable while assets load or fail.
- Empty links are intentionally unavailable in this issue. Password recovery remains unavailable until #261 is implemented; other destinations require later product decisions.

## Open Questions

None.
