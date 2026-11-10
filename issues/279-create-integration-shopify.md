# Issue #279: Create integration shopify

Source issue: https://github.com/benitogonzalezh/five-panel/issues/279

## TL;DR

Connect Shopify to Five Panel so a business can manage products, prices, stock, customers, orders, payments, shipping, returns, and refunds from Five Panel.

Build a shared model for integration definitions, customer connections, and actions. Installing and approving the Five Panel Shopify app creates the customer's connection.

First, import the store’s existing Shopify data. Then keep both systems in sync, including new sales from Shopify. Users choose which system controls each field; its value is automatically restored when the two differ.

Provide five tabs: **Connection, Field mapping, Schedule, History, and Actions**. Reuse existing automation steps and queues for progress and recovery.

The first version supports one Shopify store per business, with multiple stock locations.

## What will change

- **New code:** shared connection and request handling, Shopify definitions and app installation, import, sync, record matching, store actions, and five setup tabs.
- **Focused changes:** extend existing integration registration, credential/configuration handling, automation operations, and history. Current settings mainly support file imports; preserve those flows.
- **Reuse:** existing automation, events, retries, scheduling, credentials, and record services. Shared work uses composables; direct handlers call services for validation and access checks. Repositories own persistence.

**Code impact: Large overall**, mainly because of the amount of new functionality. Changes to existing code should stay focused.

## How the integration would work

Use the existing [automation backend](https://github.com/five-panel/five-panel/blob/main/docs/automations.md) with three shared concepts:

- **Integration definition:** connection fields, authentication and setup rules, credential renewal, disconnection, and available actions. Developers provide definitions initially.
- **Connection:** a business's saved address, protected credentials, approved permissions, and status.
- **Action:** named inputs, whether it reads or changes data, request details, and output/error rules. HTTP actions describe method, path, and body; GraphQL actions include a query or mutation and variables.

Exact storage and new module names remain implementation decisions.

1. **Connect Shopify.** Add the five-tab view through `frontend/src/views/IntegrationsDashboardView.vue`. The user selects Connect Shopify, [approves the app](https://shopify.dev/docs/apps/build/authentication-authorization/authenticate-standalone-apps), and links their business. Verify installation callbacks and store authorization; save the connection using extended `backend/src/services/IntegrationCredentialService.ts` and `backend/src/services/IntegrationConfigService.ts`. Support manual credentials for definitions that need them. Renew expiring credentials using provider rules. Reconnect updates the connection; disconnect or uninstall stops further calls. Keep credentials out of step history.

2. **Import first.** Enable models and map fields, including the local field storing each external record ID. Read Shopify through its actions in resumable batches; save records through `backend/src/services/DataService.ts`. Fill external IDs on import or external creation; keep them unique within connection/resource context, protect accidental edits, and review relinking. Shopify supplies starting values even for fields Five Panel will later control. Save progress, collect incoming changes, and catch up before review and activation. Historical orders must not repeat stock deductions or business actions.

3. **Run small automation steps.** Reuse `backend/src/services/AutomationService.ts`, `backend/src/services/AutomationRunner.ts`, and `backend/src/worker/taskList.ts`. Record changes, verified Shopify webhooks, schedules, and manual actions start flows. Each step performs one operation; the runner saves output and queues the next step. Later inputs can use `{{ steps.find_product.output.id }}`. Reuse `backend/src/services/JobScheduleService.ts` and `backend/src/services/GraphileWorkerService.ts`; save database changes, events, progress, and queued work together. An import service queues successive batches without adding general automation loops.

4. **Follow each field's main system.** Read current mappings and latest owning-system values. A Five Panel-owned price changed from 20 to 25 updates Shopify; a later Shopify edit to 15 is corrected to 25. Keep mapping, ownership, and stock rules separate from request execution. When Five Panel controls stock, apply confirmed Shopify sales, stock-releasing cancellations, and restocking returns/refunds once before syncing quantities back. Include scheduled checks; repeated updates and historical orders already reflected in starting stock must not change stock again.

5. **Execute connection actions.** A shared request service checks connection access and action compatibility, supplies authentication, executes the selected action, and reads its response. Shopify actions such as listing products can be request definitions; small provider functions handle installation, webhooks, or special processing. Register through `backend/src/types/automation.ts`. Check inputs, defaults, eligibility, and provider errors, including GraphQL errors within successful HTTP responses. External calls stay outside database transactions and reuse stable operation identities. Retry according to action behavior, not HTTP method; pause uncertain results for checking. Field corrections do not submit business actions.

6. **Show history and recover.** Reuse automation run/step history with connection, resource, record, and action references. Show inputs, outputs, corrections, attempts, and errors; resume without repeating completed steps. Keep `backend/src/services/JobExecutionService.ts` for scheduled-job summaries. Shared history remains independent of Shopify rules.

## Done when

- Installing creates the correct business's connection; another business cannot use it, and disconnected connections cannot execute actions.
- Shared request handling runs Shopify read/write actions, including GraphQL, and reports provider failures correctly.
- Selected Shopify data imports before outgoing activation; afterward mapped fields follow and restore their selected main system's latest values.
- With Five Panel controlling stock and both systems at 10 units, a Shopify sale of one unit leaves both at 9, even if its update arrives twice.
- Store actions work; retries/restarts preserve completed work without duplicate orders, payments, shipments, refunds, or stock movements. Uncertain results remain paused.
- History shows inputs, outputs, and errors; existing integrations and unrelated automations still work.

## Out of scope

- More than one Shopify store per business.
- Other conflict policies or temporary overrides of the selected main system.
- Automatic permanent deletion or recreation of deleted linked records.
- Building a storefront, checkout, payment processor, or accounting system.
- Subscription management, issuing gift cards, advanced wholesale or market-specific pricing, or automatic exchanges.
- A user-facing integration/action builder, non-HTTP connection types, or unrelated changes to existing integrations.
- Public Shopify App Store publication and review are separate release work; app installation and authorization are included here.
- A visual automation builder, general subscriber system, Redis, replacing the queue, or multi-step transaction groups; each database step keeps its existing transaction boundary.
