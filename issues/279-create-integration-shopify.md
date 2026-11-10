# Issue #279: Create integration shopify

Source issue: https://github.com/benitogonzalezh/five-panel/issues/279

## TL;DR

Connect Shopify to Five Panel so a business can manage products, prices, stock, customers, orders, payments, shipping, returns, and refunds from Five Panel.

First, import the store’s existing Shopify data. Then keep both systems in sync, including new sales from Shopify. Users choose which system controls each field; its value is automatically restored when the two differ.

Provide five setup tabs: **Connection, Field mapping, Schedule, History, and Actions**. Run sync through the existing automation steps and background queue, with clear progress, errors, and recovery.

The first version supports one Shopify store per business, with multiple stock locations.

## What will change

- **New code:** the Shopify connection, initial import, ongoing sync, record matching, stock and order actions, and the five setup tabs.
- **Changes to existing code:** make Shopify appear in the app, register integration operations with the automation backend, and extend settings and history where needed.
- **Reuse existing services:** reuse the automation runner, saved events, step outputs, retries, scheduling, credentials, and record services. New shared work uses composables; direct handlers call services for validation and access checks, while repositories own persistence. Keep shared changes focused.

**Code impact: Large overall**, mainly because of the amount of new functionality. Changes to existing code should stay focused.

## How the integration would work

The shared design uses connections, external record IDs, field mappings, and the existing [automation backend](https://github.com/five-panel/five-panel/blob/main/docs/automations.md). The Shopify connector owns Shopify's API calls and rules. File references guide reuse; new module names and storage details remain implementation decisions.

1. **Connect and configure the store.** Add Shopify to `frontend/src/views/IntegrationsDashboardView.vue`, opening a new screen with the five tabs. Reuse `backend/src/services/IntegrationCredentialService.ts` to protect credentials and extend `backend/src/services/IntegrationConfigService.ts` to support the connection and its settings. Shared settings handling stays generic; the Shopify connector checks Shopify-specific requirements.

2. **Import the existing store.** Users enable the models to sync. The connector reads Shopify data, and `backend/src/services/DataService.ts` validates and saves Five Panel records. In Field mapping, users choose which field on each mapped model stores the external record ID. Fill it when importing or creating the external counterpart. Keep links unique within their connection and resource context, protect them from accidental edits, and review relinking. Save progress against the connection. Outgoing changes start only after review and activation.

3. **Run through existing automation steps.** Reuse `backend/src/services/AutomationService.ts`, `backend/src/services/AutomationRunner.ts`, and the jobs in `backend/src/worker/taskList.ts`. Record changes, verified Shopify webhooks, scheduled checks, and manual actions start configured flows. The connector verifies Shopify webhooks before handing them to the automation service. Reuse `JobScheduleService.ts` for schedules and `GraphileWorkerService.ts` for queued delivery. Each step performs one operation; the runner saves its output and queues the next step. Later inputs can reference `{{ steps.find_product.output.id }}`. Save database changes, events, progress, and queued work together. No general subscriber system is needed.

4. **Keep each field aligned with its main system.** The shared sync flow reads the current mappings and latest main-system values; the connector performs the permitted external changes. For example, changing a Five Panel-owned price from 20 to 25 updates Shopify to 25. If someone then changes Shopify to 15, the integration restores 25. Normal local record saving continues through the shared data services, with Shopify-specific rules kept in the connector.

   When Five Panel controls stock, apply confirmed movements from Shopify sales, cancellations that release stock, and returns or refunds that restock units once in Five Panel before syncing quantities back. Include movements found by scheduled checks; repeated updates and historical orders already reflected in starting stock must not change stock again.

5. **Handle store actions.** Register integration operations through the existing handler contract in `backend/src/types/automation.ts`; select the connector through its connection. Keep each handler thin, with Shopify rules in connector services. Check inputs, defaults, and eligibility. External calls run outside database transactions and reuse a stable operation identity. Reuse the runner's retry and uncertain-result handling: confirm success or non-execution before resuming. Ordinary field corrections do not submit business actions.

6. **Show results and recover failures.** Build the Shopify History tab on existing automation run and step history, adding connection, resource, record, and action references where needed. Show inputs, outputs, corrections, attempts, and readable errors. Resume saved progress without repeating completed steps. Keep `backend/src/services/JobExecutionService.ts` for scheduled-job summaries. Shared history stays independent of Shopify rules.

## Done when

- Existing Shopify data imports before outgoing sync is activated.
- Each mapped field follows its selected main system's latest value and corrects changes made in the other system.
- With Five Panel controlling stock and both systems at 10 units, a Shopify sale of one unit leaves both at 9, even if its update arrives twice.
- Configured store actions work; retries do not duplicate orders, payments, shipments, refunds, or stock changes.
- Existing automation steps run sync flows; History shows step inputs, outputs, and errors, and interrupted work resumes without repeating completed steps.
- Existing integrations and unrelated automations still work.

## Out of scope

- More than one Shopify store per business.
- Other conflict policies or temporary overrides of the selected main system.
- Automatic permanent deletion or recreation of deleted linked records.
- Building a storefront, checkout, payment processor, or accounting system.
- Subscription management, issuing gift cards, advanced wholesale or market-specific pricing, or automatic exchanges.
- A general integration builder or unrelated changes to existing integrations.
- A visual automation builder, general subscriber system, Redis, replacing the queue, or multi-step transaction groups; each database step keeps its existing transaction boundary.
