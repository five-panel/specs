# Shopify Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Connect Shopify, import its existing data, and run configurable sync and commerce actions through shared connections and the existing automation system.

**Architecture:** Extend the existing configuration and encrypted-credential models; keep actions and mappings in JSON. Reuse sequential automation steps, PostgreSQL transactions, Graphile, and business-record services. Keep request execution and data mapping generic, with Shopify authorization and business rules in its connector.

**Tech Stack:** TypeScript, Fastify, Kysely/PostgreSQL, Graphile Worker, Vue 3, PrimeVue, existing Node tests and Playwright.

**Spec:** [Issue #279](../../../issues/279-create-integration-shopify.md). **Designs:** [13 saved screens and prototype](../../../issues/279-shopify-designs/README.md).

## Global Constraints

- The first version supports one Shopify store per business, with multiple stock locations.
- Store external record IDs in the selected business-record fields.
- Users add actions without code registration.
- Keep generic data mapping separate from ownership and Shopify rules.
- General recursive automation remains blocked.
- Keep existing per-step transactions.
- Never expose credentials in history.
- No application code is implemented by this document. All paths below refer to the application repository unless marked otherwise.
- Follow `AGENTS.md`, `RULES.md`, and `docs/agent/backend-standards.md`: composables, direct handlers, service-owned validation/access, repository-owned storage, and trusted company context.

## Review Focus

1. Duplicate and out-of-order sales/inventory notifications must leave stock correct, including after polling or a crash; Task 7.
2. A crash between saving a batch and scheduling its successor must preserve saved data and produce one continuation; Tasks 5–6.
3. Disconnection or an action edit while work is queued must not leak credentials or silently change an existing request; Tasks 2–4.
4. A timeout after a successful charge or refund must require verification rather than submit it again; Task 8.
5. Missing mappings, partial order-history permission, and tab/reload navigation must not silently complete setup; Tasks 3, 6, and 10.

---

## Starting point and boundaries

Source references were checked against `origin/main` at `ebccffa359d21ccd7687fc5b2cc3d691ede3919a` on 2026-11-11. Recheck the named files and migration numbers before implementation. This document records source findings and proposed changes; it does not claim application tests have run.

| Existing code | Reuse or focused extension |
| --- | --- |
| `backend/src/services/IntegrationConfigService.ts`, `backend/src/repositories/IntegrationConfigRepository.ts` | Currently file-import configuration. Add HTTP/Shopify shapes; keep Dropbox/Drive validation and file-only `reprocess` behavior. |
| `backend/src/services/IntegrationCredentialService.ts`, `backend/src/repositories/IntegrationCredentialRepository.ts` | Reuse encryption; add lookup by credential ID and company. Current provider-wide uniqueness cannot represent independent HTTP connections. |
| `backend/src/services/AutomationService.ts`, `AutomationRunner.ts`, `repositories/AutomationRepository.ts` | Keep definition snapshots, saved step outputs, retry/resolve, run locking, and transactional next-step jobs. Add action snapshots and controlled continuation. |
| `backend/src/services/GraphileWorkerService.ts`, `JobScheduleService.ts`, `backend/src/worker/taskList.ts` | Reuse the queue and scheduled automation jobs. No new broker or subscriber service. |
| `backend/src/services/DataService.ts`, `backend/src/services/automation/recordHandlers.ts` | Reuse record validation, access, changes, and events within the caller's transaction. |
| `backend/src/routes/automationRoutes.ts` | Reuse run details, retry, and resolution. Its generic webhook signature is not Shopify's signature. |
| `frontend/src/views/IntegrationsDashboardView.vue`, `frontend/src/router/index.ts` | Add connections without rewriting file integration screens. Build reusable activity and execution views. |

`backend/src/integrations/integrationRequestProvider.ts` only reads a provider name from a body; it is not an HTTP client. Existing run steps live in `automation_runs.step_runs` JSON, not a separate step table.

Deliver the tasks in order. Each is a reviewable unit with its own tests; the feature remains unavailable until its required flow is complete. This is one integration plan with small shared additions, not a separate automation-platform redesign.

For each task, write its named tests first, run them to observe the missing behavior fail, implement the change, then rerun and commit. Commands below are future implementation checks. Database files need separate disposable databases when their harness requires an empty database; a skipped check is not a passing check.

## Data and operation contracts

Define shared types in proposed `backend/src/types/integration.ts`:

- `ConnectionConfiguration`: `version`, `baseUrl`, `authentication` (method and secret references), `status` (`connected | disconnected`), `phase` (`mapping | importing | review | active | paused`), `actions`, `models`, `schedule`, and `currentImport`. Keep only the current checkpoint here; old history comes from saved runs.
- `RequestAction`: named input rules, `method`, `path`, non-secret `headers`, JSON `body`, output/error rules, version, and retry behavior. GraphQL stores its document and variables in the body. Store no executable user code.
- `ModelMapping`: enabled resource, destination model, external-ID field, field mappings with `owner: five_panel | remote`, and required location mappings. Mark unsupported write fields explicitly.
- `DataMapping`: `{ fields: { source: string; target: string; required?: boolean; convert?: 'string' | 'number' | 'boolean' | 'date' }[] }`. Paths are safe dotted names; array input applies the same mapping to each item. Preserve null/false/zero; omit missing optional values and fail missing required values.
- `IntegrationRequest`: `{ connectionId: string; action: string; arguments: Record<string, unknown> }`.
- `RequestResult`: `{ data: unknown; remoteId?: string; warnings?: string[] }`; errors use `AutomationStepError` codes and uncertainty.
- `ImportBatch`: `{ items: Record<string, unknown>[]; nextCursor: string | null; hasMore: boolean; total: number | null }`.
- `NextExecution`: `{ automationId: string; data: Record<string, unknown> } | null`.
- `SaveBatchResult`: `{ saved: number; processed: number; nextExecution: NextExecution }`.
- `ImportReview`: import ID/phase, per-resource counts and sample-record links, ownership/schedule/action summaries, permission/baseline warnings, and `canActivate: boolean` with blocking reasons.
- `ActivityPage`: `{ items: ActivitySummary[]; nextCursor: string | null }`; `ActivitySummary` contains operation ID, business kind/status/summary, progress, record/execution links, and allowed follow-up actions. Define these DTOs with the shared activity service in Task 9.

IDs are real UUIDs in storage and API calls. `IMP-42`, `page-2`, 250, and 1,248 in the conversation are explanatory examples; provider cursors are opaque, and totals can be unknown.

## Task 1: Store connections and operation identities

**Files:** modify `backend/src/db/types.ts`, both integration config/credential services and repositories above; create `backend/src/types/integration.ts`, `backend/src/services/IntegrationConnectionService.ts`, `OperationEffectService.ts`, `backend/src/repositories/OperationEffectRepository.ts`, and `backend/src/db/migrations/038_add_http_connections.ts`; test `backend/test/services/IntegrationConfigService.test.ts` and new `backend/test/integration/connections.test.ts`.

**Interfaces:** `useIntegrationConnectionService({ db })` produces `findById({ user, id })`, `create({ user, data })`, and `update({ user, id, data })`, returning a secret-free connection. Credential service adds `findById({ tenantId, id })` for trusted callers and `create({ tenantId, provider, scope, credentials })`; retain legacy methods. `useOperationEffectService({ db }).claim({ tenantId, connectionId, effectKey }): Promise<boolean>` claims a trusted receipt/effect in the caller's transaction; false means already claimed.

- [ ] Write tests for legacy configs, two HTTP connections using the same provider, foreign credential IDs, and secret-free API responses. Assert `legacy.configuration.reprocess === true` only for legacy file updates and `http.configuration.reprocess === undefined`.
- [ ] Run the focused tests and observe the new behavior fail before implementation.
- [ ] Extend `integration_configs` with nullable `credential_id` and a company-scoped foreign key, adding the required `(tenant_id, id)` unique keys. Add credential `scope` defaulting to `legacy`; replace provider-wide uniqueness with `(tenant_id, provider, scope)`. Legacy lookups/upserts explicitly use `scope: legacy`; new connections use their ID as scope. Preserve encrypted values.
- [ ] Enforce one Shopify connection per company and one company binding per canonical Shopify shop, including concurrent installs. Index the verified shop address for webhook lookup; generic HTTP connections remain independent. Test duplicate installs and a store already bound to another company.
- [ ] Add `operation_effects` with company, connection, stable effect key, applied quantity/outcome JSON, timestamps, and unique `(tenant_id, connection_id, effect_key)`. This is frequently checked operational data, not configuration or record links. Test concurrent claims and transaction rollback; stock rules are added in Task 7.
- [ ] Use discriminated configuration validation. Keep endpoints/actions/mappings together in JSON; write only the changed configuration part under a row lock so progress does not overwrite user edits. Add redacted configuration change events atomically; internal checkpoint-only updates do not create business activity.
- [ ] Run migration tests against an isolated database containing old credentials/configurations, then the service tests. Expected: all pass with data preserved. Commit this unit.

## Task 2: Execute saved HTTP and GraphQL actions

**Files:** create `backend/src/services/IntegrationRequestService.ts`, `backend/src/integrations/http/request.ts`, `backend/src/integrations/http/actionDefinition.ts`; test new `backend/test/services/IntegrationRequestService.test.ts`.

**Interfaces:** `useIntegrationRequestService({ db }).execute({ user, request, executionKey, actionSnapshot }): Promise<RequestResult>`. `actionSnapshot` is a server-validated immutable `RequestAction`. Direct calls obtain a snapshot through the same connection service; queued calls use Task 4's saved snapshot.

- [ ] Add tests named `graphql_user_errors_fail_successful_http`, `disconnected_connection_makes_no_request`, `foreign_connection_is_not_found`, `redirect_cannot_forward_credentials`, and `mutation_timeout_is_uncertain`. Assert zero HTTP calls for unauthorized/disconnected requests, provider error codes for HTTP 200 GraphQL failures, and `uncertain === true` after an ambiguous write.
- [ ] Run the new file to confirm failure, then implement supported authentication: none, bearer, basic, named secret header, and Shopify-managed token. Inject secrets only at send time. Check live connection status and permissions on every attempt, including direct calls.
- [ ] Validate inputs, URL, method, output rules, timeouts, and response size. Resolve endpoints against the saved address; reject internal/private destinations and unchecked redirects. Treat GraphQL errors and mutation `userErrors` as failures; preserve known partial results rather than retrying an entire partially applied mutation. Handle throttling using provider guidance; HTTP POST alone does not determine retry safety.
- [ ] Keep authentication headers, tokens, and secret input values out of saved inputs, outputs, errors, and logs. Custom actions default to verification after ambiguous writes unless their actual API contract provides safe duplicate prevention.
- [ ] In `backend/`, run `npx tsx --test test/services/IntegrationRequestService.test.ts`. Expected: pass against controlled HTTP fixtures. Commit.

## Task 3: Install, reconnect, and disconnect Shopify

**Files:** create `backend/src/integrations/shopify/auth.ts`, `webhooks.ts`, `subscriptions.ts`, `actions.ts`, `capabilities.ts`, and `backend/src/routes/shopifyRoutes.ts`; modify `backend/src/index.ts` and service composition in `backend/src/services/context.ts`; test new `backend/test/routes/shopifyRoutes.test.ts`.

**Interfaces:** `useShopifyConnectionService({ db })` in new `backend/src/services/ShopifyConnectionService.ts` produces `connect({ user, shop }): Promise<{ authorizationUrl: string }>`, `complete({ query }): Promise<{ connectionId: string; returnPath: string }>`, `disconnect({ user, connectionId }): Promise<void>`, and `receiveWebhook({ shop, headers, rawBody }): Promise<void>`.

- [ ] Test signed callback/state validation, state replay, changed company/session, wrong store on reconnect, expired-token renewal, uninstall, and restricted historical orders. Observe failing tests first.
- [ ] Add authenticated `POST /integrations/shopify/connect` and signed `GET /integrations/shopify/callback`. Bind one-time, expiring state to the current authorized company/user, shop, and return path. Callback must recheck the user's active access; do not accept a tenant ID from query/body or present a company chooser.
- [ ] Store offline credentials encrypted, with returned expiry/refresh information. Serialize refreshes per credential. Keep app secrets on the server. Request scopes for enabled capabilities and show granted limits, including unavailable older orders; do not silently claim the whole store was imported.
- [ ] Register required Shopify webhook subscriptions and add signed raw-body `POST /integrations/shopify/webhooks`; resolve the connection from the verified shop. Verify Shopify's base64 HMAC before accepting events. Persist receipt/event and a deduplicated dispatch job before acknowledging; reuse Task 1's effect storage. Disconnect/uninstall marks access invalid and stops future calls; reconnecting the same store keeps settings.
- [ ] Run `npx tsx --test test/routes/shopifyRoutes.test.ts` in `backend/`; expected pass. Record a development-store install/reconnect/uninstall check in Task 11. Commit.

The Shopify preset uses a pinned Admin GraphQL API version, initially `2026-10`. Define product, variant, inventory/location, customer, order, payment, fulfillment, return, and refund read/write capabilities; lack of an API write capability must disable that field/action with an explanation. Do not promise that arbitrary order fields can be edited by a product-style update.

## Task 4: Add shared operations and freeze request definitions

**Files:** create `backend/src/services/automation/handlers.ts`, `integrationRequest.ts`, `dataMap.ts`, and `backend/src/services/DataMappingService.ts`; modify `backend/src/types/automation.ts`, `backend/src/services/AutomationService.ts`, `AutomationRunner.ts`, and `backend/src/repositories/AutomationRepository.ts`; test new `backend/test/services/DataMappingService.test.ts` and extend `backend/test/integration/automations.test.ts`.

**Interfaces:** `data.map` consumes `{ value: unknown; mapping: DataMapping }` and produces `{ value: unknown }`; `mapData({ value, mapping }): unknown` is the pure shared function. `integration.request` consumes `IntegrationRequest` and returns `RequestResult`. Both handlers are direct functions using `AutomationHandlerContext`.

- [ ] Test nested mapping, arrays, null/false/zero, missing required values, and forbidden prototype paths with explicit assertions such as `assert.deepEqual(mapped, { price: 0, active: false, note: null })`. No arbitrary expressions or JavaScript evaluation.
- [ ] For this version, validate literal connection IDs and action names in request steps; arguments may use existing whole-value merge tags. At dispatch, resolve those saved actions through the connection service and store a secret-free `operationSnapshot` on their step runs. Customer action edits affect future runs. Load current credentials and connection status at execution time.
- [ ] Register shared operations once in the new handler map, consumed by both service validation and runner execution. `context.ts` only composes services; it is not a per-endpoint registration list. Adding a customer action changes JSON configuration, not this map.
- [ ] Test an action edited between attempts, a removed/disabled connection, and a later step using an earlier output. Assert the saved action version remains the same, credentials are absent, and existing record-only runs behave unchanged. Direct field references need no extra mapping step.
- [ ] Run the mapper test and the automation database suite with `AUTOMATION_TEST_DATABASE_URL` set to a fresh disposable database. Expected: pass without skipped database checks. Commit.

## Task 5: Queue a requested execution as a visible step

**Files:** create `backend/src/services/automation/automationEnqueue.ts`; modify the handler map, automation types/service, and repository; extend `backend/test/integration/automations.test.ts`.

**Interfaces:** `automation.enqueue` consumes `{ request: NextExecution }` and returns `{ queued: boolean; eventId: string | null }`. A new trusted `AutomationService.enqueue({ tenantId, parentRunId, stepName, request }): Promise<{ queued: boolean; eventId: string | null }>` uses the caller's transaction. It never calls Five Panel over HTTP.

- [ ] Add failing tests for `null`, a disabled/foreign target, duplicate job delivery, rollback, non-advancing continuation, and an import exceeding 16 batches. Assert null creates no job; duplicate delivery creates one child execution; rollback leaves neither child event nor job.
- [ ] Implement it as an **internal** operation. Save the target event, dispatch job, and completion of the enqueue step together. Derive company, actor, parent, operation identity, and continuation authorization from saved server state, never from `request.data`.
- [ ] Preserve normal cycle/depth checks. Add a narrow continuation permit for a saved import batch that advanced its checkpoint: one successor per parent step/checkpoint. The enqueue handler passes opaque data; the import service validates the checkpoint and target before issuing this permit. Keep it in trusted event metadata, bound to the parent run and target. Ordinary self/cyclic triggers remain blocked; import continuation does not accumulate an unbounded recursive ancestry.
- [ ] Test the normal cycle guard still blocks a hand-written self-trigger even if its payload forges import IDs or metadata. Run the database suite; expected pass with all rollback and continuation assertions exercised. Commit.

## Task 6: Import selected models through batches

**Files:** create `backend/src/services/ConnectionImportService.ts`, `backend/src/services/RecordLinkService.ts`, `backend/src/services/automation/importSaveBatch.ts`, `backend/src/integrations/shopify/importDefinitions.ts`, and `backend/src/routes/connectionRoutes.ts`; modify configuration repository, automation service/handler map, `backend/src/services/DataService.ts`, `ModelService.ts`, `backend/src/types/schema.ts`, and `backend/src/index.ts`; test new `backend/test/integration/connectionImports.test.ts`.

**Interfaces:** import service produces `start({ user, connectionId }): Promise<{ importId: string }>`, `saveBatch({ context, importId, resource, cursor, batch }): Promise<SaveBatchResult>`, and `review({ user, connectionId }): Promise<ImportReview>`. Link service produces `findOrSave({ user, mapping, remoteId, values, eventContext }): Promise<{ recordId: string; created: boolean }>` using `DataService`.

- [ ] Test incomplete mappings, duplicate IDs, interrupted save/enqueue, replayed cursor, unknown total, and a Shopify-owned starting value in a future Five Panel-owned field. Assert failed batches roll back records, progress, and events; completed batches remain after the next step fails.
- [ ] Add authenticated `POST /connections/:id/imports` for **Save and import** and `GET /connections/:id/import-review`. Validate selected mappings, dependencies, locations, and permissions. Under the connection lock, repeated starts return the same active import. Freeze its mapping version; edits require new review, not checkpoint rewriting.
- [ ] Seed data without outgoing effects. Capture incoming changes, import dependencies, then catch up before review and activation. Historical seed writes retain record history but carry trusted bootstrap metadata that prevents business-automation dispatch; test that historical orders trigger no stock or money actions. Preserve other imports' existing dispatch behavior.
- [ ] Establish each location's starting stock with a final inventory read after catch-up; save its observation/cutover and mark already represented movements without subtracting them again. New movement identity uses the movement's time/state, not the parent order's creation time. If a consistent baseline cannot be confirmed, retain pending stock and block outgoing quantity writes.
- [ ] Save selected external IDs on business records for import/external creation, using compatible unique fields. Add generic `systemManaged?: boolean` field metadata and an internal `managedFieldWrites?: string[]` DataService option. Ordinary API/automation writes cannot change those IDs; only the validated link service supplies that option. Model edits preserve the managed flag. Test ordinary edit denial and authorized link writes. Do not assume the reserved `externalId` column supports arbitrary updates; conflicting links stop for explicit review, never fuzzy matching.
- [ ] Lock the current import checkpoint, validate expected cursor, save records/events/progress and the runner's next **step** job in one transaction. `save_batch` returns `nextExecution`; it does not start the next batch. The checkpoint rejects stale replays and unchanged cursors. After the last resource, catch up and return `nextExecution: null`.
- [ ] Run `npx tsx --test test/integration/connectionImports.test.ts` in `backend/` against a fresh disposable database. Assert five batch executions for the 1,248/250 fixture, cumulative counts `250, 500, 750, 1000, 1248`, and one stable `operationId = importId`. Commit.

Each connection/resource receives a preset batch automation with a fixed action, preserving Task 4's snapshot rule. The initial event carries `importId`, `cursor: null`, and `limit`; each successor includes the opaque next cursor. The same import ID groups resources and catch-up work.

```json
{
  "trigger": { "eventType": "automation.manual" },
  "steps": [
    {
      "name": "read_batch",
      "operation": "integration.request",
      "input": {
        "connectionId": "11111111-1111-4111-8111-111111111111",
        "action": "list_products",
        "arguments": {
          "cursor": "{{ event.data.cursor }}",
          "limit": "{{ event.data.limit }}"
        }
      }
    },
    {
      "name": "save_batch",
      "operation": "import.save_batch",
      "input": {
        "importId": "{{ event.data.importId }}",
        "resource": "products",
        "cursor": "{{ event.data.cursor }}",
        "batch": "{{ steps.read_batch.output.data }}"
      }
    },
    {
      "name": "queue_next_execution",
      "operation": "automation.enqueue",
      "input": { "request": "{{ steps.save_batch.output.nextExecution }}" }
    }
  ]
}
```

`list_products` output rules normalize Shopify's response into `ImportBatch`. Add a `data.map` step only when a separately logged transformation is useful. No hidden cursor lookup replaces the explicit event parameter.

## Task 7: Enforce ownership and apply stock effects once

**Files:** create `backend/src/services/ConnectionSyncService.ts`, `backend/src/integrations/shopify/stock.ts`, `sync.ts`, `backend/src/services/automation/syncPush.ts`, and `syncSaveResult.ts`; extend Task 1's effect service/repository, automation types/service/handler map, and webhook adapter; test new `backend/test/integration/shopifySync.test.ts`.

**Interfaces:** sync service produces `reconcile({ user, connectionId, resource, remoteId }): Promise<void>`, `push({ context }): Promise<RequestResult>`, and `saveResult({ context, result }): Promise<void>`. `sync.push` consumes fixed `connectionId`, `action`, `resource`, and the remote/local record references; it is external. `sync.save_result` consumes connection/resource, `direction: push | pull`, and `result`; it is internal. Effect service adds `apply({ user, connectionId, effectKey, desiredQuantity, recordId, eventContext }): Promise<{ appliedDelta: number }>` in the caller's transaction.

- [ ] Write fixtures for a price changed `20 → 25` locally and `15` remotely; assert the remote ends at `25`. Test a queued stale intent: reload current values before sending; complete a superseded intent without an external write and schedule a fresh one. Freeze each actual request's payload/identity for its retries.
- [ ] Webhooks and overlapping scheduled scans start `integration.request → sync.save_result` flows; saved watermarks advance only after applying the results. Pull saves remote-owned fields and queues corrections for local-owned fields. Outgoing `sync.push → sync.save_result` uses the current local value and common request service. Extend Task 4's action snapshots to `sync.push`; it saves links/outcomes or schedules a replacement intent atomically. Mapping itself stays in `mapData`, not a Shopify preparation step.
- [ ] Extend Task 1's effect service with locked quantity/outcome updates. Stock and its applied marker commit together; outgoing actions can retain verified remote outcomes under their stable key. A claimed delivery is distinct from a completed business effect.
- [ ] For Five Panel-owned quantities, ignore inventory notifications as local adjustments. Normalize confirmed sale/cancellation/restock facts into a signed cumulative effect per order line/location, then apply `desiredQuantity - previouslyAppliedQuantity`. Polling and webhooks use the same stable key. Return and refund views of the same restock contribute once to that projection. Baseline-covered effects are marked without moving stock. Bind outbound local reservations to returned Shopify line IDs so their echo cannot reserve again.
- [ ] When Shopify owns quantity, import its observed quantity and do not additionally subtract the order. When Five Panel owns it, catch up potentially related orders before restoring a quantity; if causality/baseline is unresolved, defer correction and retain pending work. Use Shopify's conditional quantity update; a changed comparison quantity triggers reconciliation, not a blind overwrite.
- [ ] Test both start at `10`, sale of `1` gives `9`, inventory-before-order, duplicate delivery, later poll, crash/rollback, partial cancellation, restocking/non-restocking refund, and a new return against a historical order. Assert one effect per actual movement and correct final stock in both ownership modes. Run the new database suite and commit.

## Task 8: Perform commerce actions and verify uncertain results

**Files:** create `backend/src/services/ConnectionActionService.ts`, `backend/src/integrations/shopify/commerce.ts`, `backend/src/routes/connectionActionRoutes.ts`; extend action presets, connection routes/composition, and Task 7's effect service; test new `backend/test/integration/shopifyActions.test.ts`.

**Interfaces:** action service produces `start({ user, connectionId, action, recordId, commandId, options }): Promise<{ operationId: string }>`, `check({ user, runId }): Promise<{ outcome: 'succeeded' | 'not_executed' | 'unknown' }>`, and `retry({ user, runId }): Promise<void>`. The confirmation supplies a stable UUID `commandId`; the service validates state/options and atomically creates one event per command. Reusing that ID with different inputs fails.

- [ ] Add failing tests per capability below, including invalid state, explicit defaults, duplicate submission, and timeout after remote success. Action options come from defaults plus reviewed overrides, not field ownership rules.

| Capability | Shopify action and required behavior |
| --- | --- |
| Products, variants, customers | Separate product and variant mutations; prices belong to variants. Read/create/update supported fields and save returned IDs. |
| Create order | `orderCreate`; explicitly choose inventory behavior according to which system already reserved stock. Do not rely on Shopify's default `BYPASS`. |
| Cancel order | `orderCancel`; reason, notification, eligible unshipped-unit restock. The designed cancellation flow does not automatically issue a separate refund. |
| Payments | `orderCapture` for an existing authorization; external-payment recording is a distinct action, not a gateway charge. Check amount, currency, and remaining balance. |
| Shipping | `fulfillmentCreate` using fulfillment-order line IDs; support partial quantities, tracking, and notification. |
| Returns and refunds | `returnCreate`/`returnProcess` and `refundCreate` as applicable; separate money and actual restock effects. Never refund twice if the selected provider operation already includes a refund. |

- [ ] Add authenticated `POST /connections/:id/actions/:action` and `POST /connections/:id/runs/:runId/check`. Reuse saved action definitions and the existing runner for small steps. A typical creation flow is `record.read → data.map → integration.request → record.update`; build provider inputs from current linked records, then save returned IDs. Automatic field correction cannot invoke a commerce command.
- [ ] Use stable command identity and provider idempotency where supported; retain it across HTTP resubmission and worker retries. Save returned asynchronous job IDs and poll them in read steps before claiming completion. An ambiguous write pauses. `check` requires identifiable Shopify evidence: success saves recovered output and advances; confirmed non-execution permits safe retry; unknown stays paused. Never infer non-execution solely from a missing webhook or approximate record match.
- [ ] Test `timeout_after_capture_creates_one_charge`: assert `chargeCount === 1`, paused before checking, completed after confirmed success, and no second send. Add corresponding refund/order/fulfillment checks and no double local/remote restock. Run the new database suite and commit.

## Task 9: Reuse execution data for business activity and recovery

**Files:** create `backend/src/services/AutomationActivityService.ts`, `backend/src/repositories/AutomationActivityRepository.ts`, `backend/src/routes/automationActivityRoutes.ts`; extend trusted event metadata, automation service/repository, and run filtering; test new `backend/test/integration/automationActivity.test.ts`.

**Interfaces:** `find({ user, connectionId?, operationId?, cursor? }): Promise<ActivityPage>` returns grouped activity and contextual record/action links. Extend trusted emit metadata with `{ connectionId, resource?, recordId?, action?, activityKind }`; the client cannot supply company or trusted source metadata. Activity IDs are operation IDs.

- [ ] Test one import activity across five runs, restarted batches, two companies, and more than 100 runs. Assert cumulative progress is not summed twice and all execution pages remain reachable; do not inherit the current unpaginated 100-run default.
- [ ] Query existing events/runs by trusted source and operation ID. Derive import totals from committed checkpoints/step outputs and preserve final summaries in those runs. No duplicate activity table or per-product import entries. Keep old activities after `currentImport` changes.
- [ ] Add authenticated `GET /automation-activities` and paginated operation-run details. Project business labels/statuses and links through shared activity code; Shopify supplies action/result meanings. Keep scheduled-job summaries in existing `JobExecutionService.ts`.
- [ ] Return repair/check affordances according to real saved state. Price repair opens the variant; corrected source data starts a fresh safe sync intent and links back to the prior activity. Payment check uses Task 8; resolve/retry still enforce permissions and uncertainty.
- [ ] Run `npx tsx --test test/integration/automationActivity.test.ts` in `backend/` with its disposable database. Expected: pass with stable grouping, pagination, and company isolation. Commit.

## Task 10: Implement the approved UI and generic action editor

**Files:** create `frontend/src/views/ConnectionView.vue`, `frontend/src/composables/useConnectionSetup.ts`, `frontend/src/services/IntegrationService.ts`, `frontend/src/components/integrations/ConnectionSettings.vue`, `FieldMapping.vue`, `SyncSchedule.vue`, `ConnectionActions.vue`, `ActionEditor.vue`, `frontend/src/components/automations/ActivityHistory.vue`, and `ExecutionDetails.vue`; modify dashboard/router, `frontend/src/locales/{en-US,es-ES}/integrations.json`, connection routes/service, and `backend/src/services/ScheduledJobService.ts`, `JobScheduleService.ts`; test new `frontend/test/connectionSetup.test.ts`, `e2e/test/shopify.spec.ts`, and extend `e2e/test/fixtures.ts` and `support.ts` with isolated provider fixtures.

**Interfaces:** `useConnectionSetup({ connectionId })` loads the server's connection/checkpoint and exposes `saveAndImport()`, `activate()`, `pause()`, and `disconnect()`. Add `activate({ user, connectionId })`, `pause({ user, connectionId })`, and `disconnect({ user, connectionId })` to the connection service, returning its secret-free state. `IntegrationService` wraps the defined routes and existing configuration CRUD. Shared history/viewer components accept connection or operation filters; they do not import Shopify rules.

- [ ] Test the setup state function: tab changes preserve phase; incomplete selected mappings block import; only explicit activation reaches `active`. Keep connection status separate from sync phase. Pausing stops automatic outgoing work while retaining connection access and incoming catch-up; disconnect stops requests.
- [ ] Implement the five tabs and the design index's complete flow. Import stays in Field mapping, with **Save and import**. Save/restore progress across navigation/reload. Review shows actual counts and sample-record links, pending work, ownership, schedule, enabled actions, and any order-history scope limit. Explain managed ID protection when a record edit is rejected; server enforcement from Task 6 remains authoritative.
- [ ] Add authenticated `POST /connections/:id/activate`, `POST /connections/:id/pause`, `POST /connections/:id/disconnect`, and `PUT /connections/:id/credentials` in `connectionRoutes.ts`; handlers call services. Activation rechecks catch-up, baseline, mapping, and permissions. Require current company-admin checks for setup/automation administration and service permissions for business records. Schedule changes reuse scheduled automation events; validate company/target, save settings and queued schedule changes together, and honor pause/disconnect when jobs fire.
- [ ] Extend generic connection creation with base address and supported authentication. The Actions tab lists presets and lets an admin add/edit named actions, endpoint/method/headers/body, GraphQL document/variables, and output/error rules. Reuse the existing JSON editor components for structured fields. Secret fields are write-only credential inputs. Preserve existing file integration cards and screens.
- [ ] Implement open order → cancel → review options → confirm → result, price repair → related variant → return, and payment check → verified result. Show business activity first and **View execution** for actual runs/batches/steps. Match existing theme tokens; translate visible copy, keep JSON names English, support keyboard/focus and 390 px layouts.
- [ ] Run `npm --prefix frontend test`, `npm --prefix frontend run build`, and the Shopify browser journey through Docker. Assert all thirteen saved screen states plus custom-action editing, reload persistence, and foreign-company denial. Commit.

## Task 11: Verify the complete flow and existing behavior

**Files:** extend `e2e/test/shopify.spec.ts`, `e2e/test/fixtures.ts` and `support.ts` for isolated deterministic provider fixtures; update `docs/automations.md`, new `docs/integrations/shopify.md`, and `e2e/README.md`.

**Interfaces:** fixtures simulate Shopify only in test/development configuration; production callback/signature/access rules remain enabled. Reuse existing tenant/admin fixtures and Docker application. No live credentials in test data or artifacts.

- [ ] Run the complete journey: install → map selected models → five-batch import → catch-up → sample review → activate → Shopify sale → field correction → confirmed cancellation → price repair → uncertain-payment verification → pause → disconnect/reconnect. Exercise record links and execution detail from the same activity.
- [ ] Run `npm --prefix backend test`, `npm --prefix frontend test`, backend/frontend builds, then `npm --prefix e2e test -- shopify.spec.ts imports.spec.ts jobs.spec.ts`. Use separate fresh disposable databases for database integration files; confirm checks actually ran, with no required database test skipped. Verify old Dropbox, Google Drive, manual imports, and unrelated automation cycle/retry behavior.
- [ ] In a development Shopify store, verify install/refresh/uninstall, granted scopes, signed webhook delivery, partial fulfillment, cancellation/restock/refund, and ambiguous-outcome checking. Record API version and actual evidence. Mock or prototype checks cannot substitute for these provider checks.
- [ ] Document setup, supported field/action capabilities, history retention, pause/disconnect behavior, recovery, and permission limits. Record outcomes with the implementation PR. Public app distribution/review stays separate; do not choose or publish a distribution mode implicitly. Commit the verified unit.

## Coverage and implementation references

| Spec flow | Tasks |
| --- | --- |
| Connect and protect credentials | 1–3, 10 |
| Map, import, review, activate | 4–6, 10 |
| Small steps and durable continuation | 4–6 |
| Ownership and stock | 7 |
| Commerce actions and uncertain outcomes | 2, 8 |
| Activity, execution detail, and recovery | 9–11 |

The [design index](../../../issues/279-shopify-designs/README.md) lists every frame and its required behavior. The design agent reported 29 mock interaction checks at each of 1440 and 390 px; this plan requires new application checks rather than treating those as backend evidence.

Primary references for implementation:

- [Shopify authorization](https://shopify.dev/docs/apps/build/authentication-authorization/authenticate-standalone-apps) and [access-token lifecycle](https://shopify.dev/docs/apps/build/authentication-authorization/access-tokens).
- [Webhook delivery verification](https://shopify.dev/docs/apps/build/webhooks/verify-deliveries) and [reconciliation guidance](https://shopify.dev/docs/apps/build/webhooks): receipt identity and business-effect identity are different.
- [Order access](https://shopify.dev/docs/api/admin-graphql/latest/objects/Order), [order inventory options](https://shopify.dev/docs/api/admin-graphql/latest/input-objects/OrderCreateOptionsInput), [conditional inventory writes](https://shopify.dev/docs/api/admin-graphql/latest/mutations/inventorySetQuantities), [fulfillment](https://shopify.dev/docs/api/admin-graphql/latest/mutations/fulfillmentCreate), and [refunds](https://shopify.dev/docs/api/admin-graphql/latest/mutations/refundCreate). Check the pinned API's exact capabilities when implementing.
- [Graphile transactional SQL jobs](https://worker.graphile.org/docs/sql-add-job): database commits and queued work share the same transaction; duplicate delivery still requires safe handlers.
- [HTTP destination protection](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html) for user-defined endpoints with server-held credentials.
