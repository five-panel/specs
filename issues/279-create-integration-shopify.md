# Issue #279: Create integration shopify

Source issue: https://github.com/benitogonzalezh/five-panel/issues/279

## TL;DR

Connect Shopify to Five Panel so a business can manage products, prices, stock, customers, orders, payments, shipping, returns, and refunds from Five Panel.

Use shared connections and named actions for Shopify and other HTTP APIs. Users can add request actions; Shopify installation supplies ready-to-use settings and actions.

First, import the store’s existing data. Then sync selected models, including new Shopify sales. Users choose which system controls each supported field; its latest value is automatically restored when the two differ.

Provide five tabs: **Connection, Field mapping, Schedule, History, and Actions**, following the [approved prototype](https://five-pane-dev-server.tail7d89bf.ts.net/c/J88FjcHO5C/proto?frame=VUYT7ICyJS). [Saved screens](279-shopify-designs/README.md) and the [implementation plan](../docs/superpowers/plans/2026-11-11-shopify-integration.md) accompany this spec.

The first version supports one Shopify store per business, with multiple stock locations.

## What will change

- **New code:** shared connections, editable request actions, mapping, batch import, sync, Shopify installation and rules, and setup screens.
- **Focused changes:** extend configuration/credential handling, automation operations, and history. Preserve current file integrations and unrelated automations.
- **Reuse:** existing automation, events, queue, retries, scheduling, credentials, and record services. Shared work uses composables; thin handlers call services for validation and access checks. Repositories own persistence.

**Code impact: Large overall**, mainly because of the amount of new functionality. Changes to existing code should stay focused.

## How the integration would work

Use the existing [automation backend](https://github.com/five-panel/five-panel/blob/main/docs/automations.md) and keep these concepts generic:

- **Connection:** saved address, authentication, permissions, status, settings, and named actions. Shopify supplies installation and action presets; users can also create HTTP connections and actions using supported authentication methods.
- **Action:** inputs, read/write behavior, method, endpoint, headers, body, and output/error rules. GraphQL uses a document and variables, often at the same endpoint.
- **Storage:** keep small configuration in JSON and secrets encrypted separately, extending existing storage. Use indexes or related data for frequent business checks. Store external record IDs in the selected business-record fields.

Exact interfaces and file changes are in the implementation plan.

1. **Connect.** Extend `frontend/src/views/IntegrationsDashboardView.vue`. Shopify approval links the store to the already selected, authorized business; no company chooser. Extend `IntegrationConfigService.ts` and `IntegrationCredentialService.ts` for connection-scoped access, renewal, reconnection, and disconnect/uninstall. Check connection status for every request, whether direct or queued. Never expose credentials in history.

2. **Map, import, review, activate.** In Field mapping, enable models and choose destination models, external-ID fields, field owners, and stock locations. Show readiness and block incomplete selected mappings. **Save and import** starts one resumable import through `DataService.ts`, including starting values for Five Panel-owned fields. Protect linked IDs, prevent duplicates, and review relinking. Collect incoming changes and catch up before review. Show counts, sample records, ownership, schedule, and actions; only explicit activation starts outgoing sync. Tab navigation preserves setup state. Historical data must not replay business actions or stock deductions.

3. **Run small steps.** Reuse `AutomationService.ts`, `AutomationRunner.ts`, `JobScheduleService.ts`, and Graphile. Events, verified webhooks, schedules, and manual actions start flows. Save each step's output for later inputs such as `{{ steps.read_batch.output }}`. Import batches run **read → save → enqueue**: saving returns `nextExecution`; generic `automation.enqueue` starts that request or does nothing for `null`. Pass `importId`, `cursor`, and `limit` explicitly. Keep one import identity across runs, commit database work and its next queued step together, and prevent duplicate or non-advancing continuations. General recursive automation remains blocked.

4. **Follow the main system.** Keep generic data mapping separate from ownership and Shopify rules. Restore the owning system's latest field values. When Five Panel owns stock, ignore Shopify quantity edits as local adjustments, but apply confirmed sales, stock-releasing cancellations, and actual restocks once. Catch up orders before correcting stock, including when inventory webhooks arrive first. Persist business-movement identities and apply movements atomically. When Shopify owns stock, import quantities without also deducting orders. Scheduled checks cover missed webhooks; pause outgoing stock until its starting baseline is confirmed.

5. **Execute actions.** Generic `integration.request` runs saved HTTP/GraphQL actions; `data.map` handles separate data transformations when needed. Users add actions without code registration. Keep Shopify installation, webhook verification, eligibility, and business effects in its connector. Cover order creation/cancellation, payments, shipping, returns, and refunds with explicit options and confirmation. Field corrections never repeat those actions. Keep external calls outside database transactions; freeze action definitions for retries, use stable identities, and pause uncertain outcomes for checking. Pause sync separately from disconnecting.

6. **Show activity and recover.** Reuse saved automation data for a business activity view, filtered to this connection. One initial import remains one activity across batches. Offer record links, price repair, and payment verification. **View execution** opens the shared technical viewer with batches, inputs, outputs, attempts, and errors. Resume completed work safely; keep unknown outcomes paused.

## Done when

- Installation binds the selected business securely; disconnection blocks requests, and reconnection preserves configuration.
- Saved custom and Shopify actions run through shared HTTP/GraphQL handling without exposing secrets or hiding provider errors.
- Selected models pass mapping, resumable import, review, and explicit activation; tab changes preserve state and field ownership is enforced afterward.
- With both stocks at 10, a Shopify sale leaves both at 9 even with duplicate notifications, reversed delivery order, or a later scheduled check.
- Commerce actions and recovery avoid duplicate orders, payments, shipments, refunds, and stock effects; uncertain outcomes stay paused.
- History groups import batches, links business records and recovery actions, and exposes execution details; existing integrations and unrelated automations still work.

## Out of scope

- Multiple stores per business; other conflict policies; automatic permanent deletion or recreation.
- Storefronts, checkout, payment processing, accounting, subscriptions, gift-card issuance, advanced pricing, or automatic exchanges.
- Non-HTTP connectors, a full authentication-provider builder, or unrelated integration refactors.
- Public Shopify App Store publication/review; installation and authorization are included.
- A visual automation builder, general subscribers, Redis, queue replacement, or multi-step transaction groups. Keep existing per-step transactions.
