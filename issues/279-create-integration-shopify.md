# Issue #279: Create integration shopify

Source issue: https://github.com/benitogonzalezh/five-panel/issues/279

## TL;DR

Connect Shopify to Five Panel so a business can manage products, prices, stock, customers, orders, payments, shipping, returns, and refunds from Five Panel.

First, import the store’s existing Shopify data. Then keep both systems in sync, including new sales from Shopify. Users choose which system controls each field; its value is automatically restored when the two differ.

Provide five setup tabs: **Connection, Field mapping, Schedule, History, and Actions**. Run sync through the existing background queue, with clear progress, errors, and recovery.

The first version supports one Shopify store per business, with multiple stock locations.

## What will change

- **New code:** the Shopify connection, initial import, ongoing sync, record matching, stock and order actions, and the five setup tabs.
- **Changes to existing code:** make Shopify appear in the app, connect its background jobs, and extend settings and history where needed.
- **Reuse existing services:** use the current tools for reading and saving records, protecting credentials, scheduling, and queues. Any necessary change to a shared service should be small, general, and explained in the implementation plan.

**Code impact: Large overall**, mainly because of the amount of new functionality. Changes to existing code should stay focused.

## How the integration would work

The shared design uses connections, external record IDs, field mappings, progress, and results. The Shopify connector handles Shopify's API calls and specific rules. The existing files below are reference points for reuse or focused changes; new module names and the exact storage layout remain implementation decisions.

1. **Connect and configure the store.** Add Shopify to `frontend/src/views/IntegrationsDashboardView.vue`, opening a new screen with the five tabs. Reuse `backend/src/services/IntegrationCredentialService.ts` to protect credentials and extend `backend/src/services/IntegrationConfigService.ts` to support the connection and its settings. Shared settings handling stays generic; the Shopify connector checks Shopify-specific requirements.

2. **Import the existing store.** The Shopify connector reads the selected data, and `backend/src/services/DataService.ts` validates and saves Five Panel records. In Field mapping, users choose which field on each mapped model stores the external record ID. The integration fills that field when importing a record or creating its external counterpart. Keep links unique within their connection and resource context, protect them from accidental edits, and review any relinking. Save import progress against the connection so work can resume. Outgoing changes start only after review and activation.

3. **Run work in the background.** Reuse `backend/src/services/GraphileWorkerService.ts` for queued work and `backend/src/services/JobScheduleService.ts` for scheduled checks. Extend `backend/src/worker/taskList.ts` to register the required jobs. Jobs select the appropriate connector through their connection. Pending work must survive a restart, and saving a local change must also preserve the obligation to sync it.

4. **Keep each field aligned with its main system.** The shared sync flow reads the current mappings and latest main-system values; the connector performs the permitted external changes. For example, changing a Five Panel-owned price from 20 to 25 updates Shopify to 25. If someone then changes Shopify to 15, the integration restores 25. Normal local record saving continues through the shared data services, with Shopify-specific rules kept in the connector.

5. **Handle store actions.** Use a common way to submit and track actions, with Shopify handlers for operations such as creating an order, shipping items, or issuing a refund. Check the configured inputs, defaults, and eligibility before execution. Save the request and its result so retries can follow the same operation. An uncertain result requires checking before another attempt. Ordinary field corrections do not submit business actions.

6. **Show results and recover failures.** Reuse `backend/src/services/JobExecutionService.ts` for run summaries. Add reusable integration details tied to the connection, resource, record, and operation, including corrections, attempts, results, and readable errors. The Shopify History tab displays those details, including pending, failed, and uncertain work. Shared history handling stays independent of Shopify rules. Check existing integrations whenever shared registration, settings, or history behavior changes.

## Done when

- Existing Shopify data imports before outgoing sync is activated.
- Each mapped field follows its selected main system's latest value and corrects changes made in the other system.
- Configured store actions work; retries do not duplicate orders, payments, shipments, refunds, or stock changes.
- History shows problems, interrupted work can resume, and existing integrations still work.

## Out of scope

- More than one Shopify store per business.
- Other conflict policies or temporary overrides of the selected main system.
- Automatic permanent deletion or recreation of deleted linked records.
- Building a storefront, checkout, payment processor, or accounting system.
- Subscription management, issuing gift cards, advanced wholesale or market-specific pricing, or automatic exchanges.
- A general integration builder or unrelated changes to existing integrations.
