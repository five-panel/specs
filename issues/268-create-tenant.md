# Issue #268: create a tenant

Source issue: https://github.com/benitogonzalezh/five-panel/issues/268

## Problem

Create Zipp, a business that sells products through Shopify, with the records and one executive report needed to understand its sales, purchases, expenses, and money movements. The owner should be able to answer ordinary questions such as “How is my business doing?”, “Who still owes me?”, and “Where did my money go?” without learning accounting or the internal data structure.

Include the commerce models, individual receipts and payments, and one ready-to-use executive report that brings the main business figures together. Use Zipp as the first installation of a small reusable commerce setup. Another business should be able to start with the same definitions and its own identity, users, settings, records, and saved report.

## Current Behavior

- Five Panel already supports tenant creation, users and tenant memberships, custom models, relations, generated GraphQL operations, model SQL views, table views, and sidebar sections.
- Business records use the existing `custom_data` storage. Model fields and relation values are validated by the generic data service; relations must resolve inside the current tenant.
- The report runtime already supports SQL datasets, select/date filters, summary values, text, charts, tables, grouped tables, and desktop/mobile layouts. Definitions and dataset execution are checked against the current tenant's allowed model views.
- Reports can display supporting tables within the same report. The current summary and table blocks do not provide arbitrary cell links or automatic navigation from every number to a record.
- Report SQL execution is read-only, has a timeout, and returns at most 1,000 rows per dataset. A summary must aggregate all matching records before that result limit is applied.
- Existing tenant scripts show provisioning patterns, but there is no tracked Zipp setup or shared commerce setup matching this scope. This is a repository finding; it does not establish whether a Zipp record exists in a deployed database.
- Current record forms operate on one model at a time. Defining a sale and its items does not itself create a single form that saves both.
- The inspected source contains no Shopify, Meta Ads, or SII connector. This issue does not depend on one.

## Desired Behavior

### Zipp and the reusable setup

Provision the following installation:

| Setting | Value |
| --- | --- |
| Name | `Zipp` |
| Slug | `zipp` |
| Domain | `zipp.cl` |
| Default currency | `CLP` |
| Business time zone | `America/Santiago` |
| System user | `system.fivepanel@zipp.cl` |
| Administrator | `admin@zipp.cl` |

Currency and time zone are implementation defaults chosen for this Chilean business. Store the installation defaults in the existing tenant settings JSON, for example under `commerce: { defaultCurrency, timeZone, definitionRevision }`, without changing unrelated settings. Generate the model defaults and saved report definition from that configuration; changing a setting later is not an automatic rewrite of the installed models or report.

Passwords are supplied when provisioning; no password belongs in the script, spec, repository, or logs. Use the existing master user to create the tenant and first user. Both initial users are active superusers in this tenant; only the system account has `isDefaultSystemUser: true`. Both must have the memberships required by the current login flow.

Keep the ten model definitions and the single report definition reusable. Their installation accepts the target tenant, business name/domain, default currency, business time zone, and initial-user inputs. The report definition obtains SQL view names from the installed models; it must not contain a fixed Zipp tenant ID, view prefix, domain, or business name.

This reuse is a small set of definitions and ordinary functions using the current services. Each installation owns its records and saved report. Editing Zipp does not edit another tenant. Installing the setup for another business must not copy Zipp's customers, transactions, credentials, or user IDs. Later changes use an explicit dated update for each target tenant; an old provisioning script must not silently pick up a different set of model/report definitions through mutable shared imports.

### Simple words and navigation

Create a `Mi negocio` sidebar section with exactly one report: `Mi negocio en un vistazo`, with stable slug `commerce-executive-summary`.

Create a separate `Registros` section with a usable table view for each model. The administrator must receive access and sidebar placements for both sections and their children. Creating views only for the system user is insufficient. Keep the views private to the tenant with explicit access for the initial administrator; do not make them anonymously accessible.

The administrator can enter records using the existing forms and tables. The report uses common Spanish words, with a short explanation of what each figure includes. The owner must not write SQL, build a report, or inspect model definitions to read it. Single-step sale/purchase forms and `Me pagaron`/`Pagué` actions remain outside this issue.

### Ten models

Use the model slugs, field IDs, and relations below as the initial data contract. Display labels are in Spanish. All records retain the existing tenant and audit fields; do not duplicate those as custom fields.

Field notation:

- `text`: `FieldType.STRING`, `FieldUIType.TEXT`.
- `number`: `FieldType.NUMBER`, `FieldUIType.NUMBER`.
- `date`: `FieldType.STRING`, `FieldUIType.DATE`, storing a date-only `YYYY-MM-DD` value.
- `select`: string/select with explicit Spanish labels and the stable values stated below.
- `relation(model)`: object/relation to that tenant's model, using `id` as `valueKey` and its name, number, or description field as `labelKey`.
- Required fields are listed separately. Everything else is optional and has no default unless one is stated. Missing unit cost is `null`, never an implicit zero.

Use `currency` as required text on monetary parent records and variants, with the installation currency as its default. Monetary item fields inherit their parent's currency. Use uppercase ISO currency codes as the data convention. The report selects one currency at a time; no conversion is performed.

| Model / plural slug | Required fields | Optional fields and defaults |
| --- | --- | --- |
| `product` / `products` — Producto | `name` text; `status` select, default `active` | `description`, `brand`, `category` text. Status: `active` / `inactive`. |
| `productVariant` / `productVariants` — Variante de producto | `product` relation(product); `name` text; `currency` text; `status` select, default `active` | `sku`, `barcode`, `optionDescription` text; `currentPrice`, `currentUnitCost` number. Status: `active` / `inactive`. `optionDescription` can describe size, color, or another sellable option without a new options engine. |
| `customer` / `customers` — Cliente | `name` text | `email`, `phone` text. |
| `salesOrder` / `salesOrders` — Venta | `number` text; `saleDate` date; `currency` text; `totalAmount` number; `status` select, default `draft` | `customer` relation(customer); `customerName`, `notes` text; `dueDate` date; `shippingNetAmount` number, default `0`; `taxAmount` number, default `0`. Status: `draft` / `confirmed` / `cancelled`. |
| `salesOrderItem` / `salesOrderItems` — Producto vendido | `salesOrder` relation(salesOrder); `nameAtSale` text; `quantity`, `unitPriceNet` number | `productVariant` relation(productVariant); `skuAtSale` text; `unitCostAtSale` number; `discountNetAmount`, `refundNetAmount`, `recoveredCostAmount` number, each default `0`. |
| `purchaseOrder` / `purchaseOrders` — Pedido a proveedor | `number` text; `supplierName` text; `orderDate` date; `currency` text; `status` select, default `draft` | `supplierTaxId`, `notes` text; `expectedDeliveryDate` date. Status: `draft` / `sent` / `confirmed` / `cancelled`. |
| `purchaseOrderItem` / `purchaseOrderItems` — Producto o servicio encargado | `purchaseOrder` relation(purchaseOrder); `description` text; `quantity`, `unitPriceNet` number | `productVariant` relation(productVariant); `taxAmount` number, default `0`. |
| `purchaseExpense` / `purchaseExpenses` — Compra o gasto | `description` text; `incurredDate` date; `currency` text; `totalAmount` number; `status` select, default `draft` | `number`, `supplierName`, `supplierTaxId`, `documentType`, `documentReference`, `notes` text; `dueDate` date; `purchaseOrder` relation(purchaseOrder). Status: `draft` / `confirmed` / `cancelled`. |
| `purchaseExpenseItem` / `purchaseExpenseItems` — Detalle de compra o gasto | `purchaseExpense` relation(purchaseExpense); `description` text; `kind` select; `quantity`, `unitAmountNet` number | `productVariant` relation(productVariant); `category` text; `discountNetAmount`, `taxAmount` number, default `0`. Kind: `inventory` (“Productos para vender”) / `operating` (“Gasto del negocio”). |
| `moneyMovement` / `moneyMovements` — Dinero recibido o pagado | `description` text; `movementDate` date; `direction` select (`in` / `out`); `amount` number; `currency` text; `status` select, default `confirmed` | `reference`, `counterpartyName`, `method`, `notes` text; `salesOrder` relation(salesOrder); `purchaseExpense` relation(purchaseExpense); `purchaseOrder` relation(purchaseOrder). Status: `confirmed` / `cancelled`. |

A product without size/color choices is represented by one ordinary variant record. This issue's provisioning does not seed products or build an automatic product-creation workflow.

Labels for states must be plain: `Borrador`, `Confirmada`/`Confirmado`, `Anulada`/`Anulado`, `Enviado`, `Activo`, and `Inactivo`, as appropriate. Movement directions display as `Entrada` and `Salida`. States are manually entered values, not an automated transition system. No field in this initial setup requires a uniqueness constraint; use `unique: false` rather than implying that codes or references are validated as unique.

The field contracts use the validation that Five Panel already has. They do not imply that its `unique` flag enforces SKU uniqueness or that the schema supports numeric bounds, formulas, repeatable embedded items, or cross-record validation. Do not add a general validation platform to this issue. The report checks below handle incomplete or inconsistent manual financial data without inventing a confident result.

### Meaning of the records

- A product holds shared information; a variant identifies the specific sellable option. Sale items retain their own name, code, price, and cost, so changing a variant does not rewrite a past sale. The operator supplies these sale values; no automatic copy is implied by a relation.
- A sale is the complete transaction with a customer; its items say what was sold. A customer and variant relation can be absent without losing the recorded sale description.
- A purchase order records an intended supplier purchase. It does not automatically create a purchase/expense, payment, stock movement, or profit deduction.
- A purchase/expense records something acquired or incurred, whether paid or unpaid. It can exist without an invoice, receipt, supplier name, or purchase order. A purchase order can relate to multiple purchase/expense records for partial billing. Each purchase/expense relates to at most one purchase order in this version.
- Each purchase/expense item says whether it is merchandise for resale or a running expense. One purchase can contain both kinds. Do not classify the entire document as operating expense merely because it has one operating line.
- A money movement records an actual receipt or payment. Positive `amount` plus `direction` determines its sign; do not represent an outflow by entering a negative amount. A refund is an actual movement in the opposite direction linked to the original sale or purchase/expense.
- A movement may be unlinked, so money can be recorded before its purpose is identified. It can link to a sale, a purchase/expense, or just a purchase order for an advance. A purchase/expense and its purchase order may both be linked on the same movement. A sale and purchase/expense must not both be settlement targets of one movement.
- When an advance is later assigned to a purchase/expense, update that same movement's relation. Do not create a second movement for the same money. Do not automatically spread a purchase-order advance over every related purchase/expense.
- Do not store an independently editable `paidAmount`, `collectedAmount`, or payment status on the commercial headers. The report derives them from the dated movements. A payment does not create a second expense, and receiving money does not create a second sale.
- Unlinked receipts such as an owner contribution can appear in the cash group without being counted as sales or profit. Unlinked payments do not become operating expenses merely because money left.

### Report calculation rules

All calculations use current confirmed commercial records and the selected currency. Draft and cancelled commercial records do not contribute to sales or operating-result totals. Confirmed money movements remain actual money movements even if the linked commercial record is later cancelled.

Sale header `totalAmount` is the current amount due including taxes and shipping, after all commercial discounts/refunds. `taxAmount` and `shippingNetAmount` are their current amounts after adjustments. A sale item's discount and refund are line totals, not per-unit amounts.

```text
item net sales = quantity × unitPriceNet − discountNetAmount − refundNetAmount
item cost used = quantity × unitCostAtSale − recoveredCostAmount
sale net sales = totalAmount − taxAmount
sale merchandise sales = totalAmount − taxAmount − shippingNetAmount
operating expense line = quantity × unitAmountNet − discountNetAmount
estimated operating result = net sales − cost of sold items − operating expense lines
```

The item net-sales sum must match sale merchandise sales. Shipping income is included once in total net sales; product-sales highlights exclude shipping. For a purchase/expense, the sum of its line net amounts plus their tax amounts must match its `totalAmount`. Header totals are manually supplied in the existing forms; this issue does not introduce automatic write-time total calculation.

Inventory purchase lines are shown as purchases but excluded from operating expense in the result calculation. Their sold-item cost is already represented by `unitCostAtSale`. Do not deduct both the merchandise acquisition and that sold-item cost. Costs are independent historical inputs; no FIFO, weighted-average valuation, or automatic assignment from purchase prices is included.

`recoveredCostAmount` is an explicitly recorded cost recovery for a return, defaulting to zero. A customer refund reduces sales but does not automatically recover the product's cost or restore stock. A known zero unit cost is valid; a missing unit cost makes the relevant cost/profit result incomplete.

The estimate uses the entered net sale and expense amounts, with taxes shown separately. Label it `Ganancia estimada` and explain `Según ventas, costos y gastos registrados; impuestos por separado`. It does not calculate recoverable VAT, taxable profit, depreciation, or a full accounting result. Cost snapshots must be entered on a consistent basis in the sale's currency; no tax treatment or exchange conversion is inferred.

Commercial corrections/refund amounts update the original sale or expense's date bucket using its current values. Money refunds use the actual movement date. This is a current-record operating report, not an event-date refund ledger or a frozen accounting close. State this in the report explanation so a later correction to an old sale is not mistaken for a change to its cash history.

Before presenting derived results, detect confirmed headers without items, header/item mismatches, invalid quantities or amounts, refunds/discounts above the sold line amount, a recovered cost above its known original cost, and missing sale cost snapshots. Quantities and movement amounts must be positive for report calculations; prices, known costs, commercial totals, discounts, refunds, and tax amounts may be zero but not negative. Distinguish invalid data from a legitimate zero. Keep known sales and cash figures visible where possible, list affected records, and suppress any affected total cost, profit, or settlement figure with `Faltan datos` or `Revisar registros`. Do not hide incomplete records and present the remaining subtotal as a complete business result. No new generic write-time range validator is required.

### One executive report

Save exactly one ordinary `viewType: 'report'` view named `Mi negocio en un vistazo`, with slug `commerce-executive-summary` and a validated version-1 `reportDefinition`. Use the current text, chart, table, and grouped-table blocks. The owner reads the important figures together in this one report, with the following plainly labelled groups:

| Group | What the owner sees |
| --- | --- |
| `Cómo va el negocio` | `Vendiste`, `Costo de los productos vendidos`, `Gastos del negocio`, and `Ganancia estimada`, plus sales/result comparisons with the preceding equal-length period. Show merchandise acquisitions separately as `Compras de mercadería`; do not subtract them again from profit. |
| `La plata que entró y salió` | `Recibiste`, `Pagaste`, and the difference between them during the selected dates. Label the difference as money in versus money out, not available balance or profit. |
| `Lo pendiente hoy` | `Te deben`, `Debes pagar`, `Debes devolver`, and `Tienes por recuperar`, with overdue amounts identified only where due dates are known. These figures include older outstanding records and use today's cutoff, regardless of the selected sales/cash period. |
| `Qué está pasando` | A daily net-sales trend, up to three products with the highest net merchandise sales, and up to three operating-expense categories by recorded net expense. These highlights use the selected dates and currency. Product sales are not product profit; their labels must make that clear. |
| `Qué necesita atención` | Missing-cost and inconsistent-record counts affecting the selected period, unassigned receipts/payments for that period, and current unapplied supplier advances. Show a clear message when a figure cannot yet be calculated, with identifying references for affected records. |

Use a single date selection for the period groups and a single currency selection for the whole report. Offer `Hoy`, `Esta semana`, `Este mes`, and a custom date range, defaulting to the current month through today. Evaluate relative dates when the report runs, in the configured business time zone; do not freeze “this month” on the provisioning date. Weeks start on Monday. A valid custom range overrides the preset with inclusive boundaries; a single selected date means that day. Reuse existing filter conflict behavior where suitable.

Display the actual selected dates and a separate `Pendiente hoy — <business date>` label. Changing the period changes sales, costs, purchases, cash, trends, and period highlights; it does not remove earlier unpaid amounts from `Lo pendiente hoy`. If the selected period has no records, its empty message applies to those groups, while any current pending amounts remain visible.

The currency filter defaults to the installation currency. Its options include the default and currencies present in relevant tenant records. Currency selection filters commercial parents and movements consistently. Always display the currency code; an invalid cross-currency settlement link is not a conversion. Use existing numeric formats without labelling a non-CLP value as CLP. Plain decimal formatting with an explicit currency code is sufficient; no new currency-formatting system is required.

Keep the report compact: summary values, short explanations, a simple sales trend, and small supporting tables. Show up to three product/category highlights, up to five pending records, and up to five records needing review, with each limit clearly labelled. Pending records include a reference, counterparty, remaining amount, and due date when known; prioritize overdue records, then known due dates. Review records include a source reference and a plain explanation of the missing or inconsistent data. Use a stable ID as the final ordering tie-breaker.

Product highlights use net merchandise sales, excluding shipping, and do not claim to rank product profit. Group by known product/variant identity, with an explicitly unlinked group for sale items without a variant; similar saved names do not establish product identity. Expense highlights use operating lines and the purchase/expense's incurred date, whether paid or unpaid. Missing categories display as `Sin categoría`. Explain `Compras y gastos registrados; puede haber montos pendientes de pago`; actual payments belong to the cash group.

Full source records remain available through the ten table views under `Registros`. Identify the relevant source references in the report so the owner can find them there. Do not add separate report views, hidden detail reports, arbitrary cell links, or full transaction lists to this issue.

Calculate all summaries and ranked highlights from all matching source records before applying any display limit. A top-three table or five-record attention list is not the source of the total. SQL aggregation must happen before the report runtime's 1,000-row result cap. No pagination/export platform is needed.

Compare net sales and estimated operating result against the immediately preceding date range of the same length. Display both actual ranges; do not label that comparison “last month” unless those are the dates being compared. If the previous value is zero or the result is incomplete, display an explanation rather than infinity or an invented percentage. Percentage change is `(current − previous) / abs(previous) × 100` when the previous value is nonzero and both values are complete.

Show empty, loading, incomplete, and failed states without a broken chart or a false success. With no matching period records, known totals can be zero, period highlights are empty, and that group says `No hay registros para estas fechas`. Failure to load is not a zero result. Incomplete cost/profit figures retain their warning even when sales and cash are known; identifying examples do not replace the full affected-record count.

Provide explicit desktop and mobile layouts. At 390px and 1280px, titles, filters, summary values, explanations, and supporting tables remain usable. On a phone, the groups can be read by scrolling this one report; do not squeeze every figure into one physical screen. Allow horizontal scrolling inside a table where necessary; keep summary values readable without page-wide horizontal scrolling.

For pending amounts, aggregate valid movements by their direct settlement target before joining any item or report detail:

```text
net customer receipts = linked inflows − linked outflows
net supplier payments = linked outflows − linked inflows
sale remainder = current sale total − net customer receipts
purchase/expense remainder = current purchase/expense total − net supplier payments
```

Include only confirmed movements dated through today, in the same currency as the target. A positive remainder on a confirmed document is still to collect/pay. A negative remainder is a separate amount to return/recover, never silently clamped to zero. For a cancelled document, the amount due is zero, so retained money is still visible as an unsettled refund. A movement linked only to a draft document, only to a purchase order, or to no document stays separately identifiable as an advance/unapplied amount; it does not reduce an arbitrary confirmed document.

A movement with incompatible settlement targets or currency still counts in the cash group if its own monetary fields are valid. It does not settle a commercial document; flag it and the affected document for review, with their count and identifying examples in `Qué necesita atención`. When a movement links both a purchase/expense and its purchase order, apply it once to the purchase/expense. A purchase-order relation is context, not a second settlement.

## Scope

- Provision Zipp, its two initial users and memberships, the ten empty models, their table views, the two sidebar sections, and exactly one executive report.
- A small reusable set of commerce definitions and installation functions, exercised for Zipp and a separate test tenant. Zipp-specific scripts supply Zipp's identity and credentials without introducing commerce behavior that branches on `zipp`.
- The report's queries, filters, summary calculations, data-quality messages, compact supporting tables, access, layouts, and numerical behavior described above.
- New dated setup scripts and focused tests using existing tenant, model, view, data, and report services. Preserve existing dated scripts and unrelated tenants.
- Document script order, required initial-user inputs, default currency/time zone, safe rerun behavior, and the report calculation basis in the new scripts' usage comments and the existing relevant operations documentation if needed.

## Out Of Scope

- Shopify, Meta Ads, SII, banking, payment-gateway, or other external connections; application registration, OAuth, webhooks, historical imports, and recurring sync jobs.
- Tax-document issuance, automatic tax treatment, full accounting, tax filing, recoverable VAT calculation, bank reconciliation, or a claim that the result is a statutory profit figure.
- Automatic inventory movements, valuation, purchase-to-sale cost assignment, depreciation, or allocation of shared operating expenses to individual products.
- Event-date commercial refund accounting, historical balance snapshots, frozen period closes, and a separate return/credit-note model. Cash refund dates are retained in money movements.
- A single payment allocated across multiple sales or multiple purchases/expenses. One commercial document can have many movements; allocation across documents may later be added without replacing the recorded movements.
- Compound sale/purchase forms, automatic header totals on write, automatic price/cost snapshot copying, payment-entry buttons, automatic record-state transitions, or a new general business-validation system.
- A supplier-management model, bank-account model, reusable document/attachment system, or an options/collections engine beyond the fields stated above.
- A visual template builder, tenant-template marketplace, automatic propagation of changes to existing tenants, or a general tenant-provisioning framework.
- Additional report views, new report widget types, arbitrary clickable number/cell navigation, exports, pagination, or a report editor redesign. Use the existing report runtime and compact supporting tables.
- Bespoke branding, production execution, password resets for existing users, and seeded real or example business transactions. Example transactions below are test fixtures only.

## Acceptance Criteria

Numerical examples are test fixtures with tax and shipping amounts set to zero unless stated otherwise; they are not transactions to seed into Zipp.

1. Fresh provisioning creates one `Zipp` tenant with slug `zipp`, domain `zipp.cl`, currency `CLP`, and business time zone `America/Santiago`, plus active system/admin users with the stated addresses and the memberships needed for administrator sign-in. Initial credentials come from supplied inputs and are absent from tracked files and logs.
2. The tenant has exactly the ten setup-defined models, with the slugs, types, required fields, defaults, and relations stated above. It has no seeded products, sales, purchases, expenses, or movements. A purchase/expense without a fiscal document and a sale without a customer relation can be saved using the existing API.
3. The administrator can open and use all ten record table views under `Registros` and exactly one report under `Mi negocio`: `Mi negocio en un vistazo`, with slug `commerce-executive-summary`. No additional report views or hidden detail reports are created. Another tenant's user cannot read these views or their datasets by supplying their IDs.
4. Sale items, purchase-order items, and purchase/expense items can be saved against their own parent records. Existing API checks reject an invalid or foreign-tenant relation. Changing a variant's current name, price, or cost does not alter a saved sale item's name/price/cost or its historical report calculation.
5. The report definition passes the current definition validator, SQL safety checks, view allowlist checks, and filter compilation checks. Its SQL references only physical model views belonging to the installation tenant. Dataset failures return the current safe error behavior instead of raw SQL or a zero result.
6. With a confirmed merchandise acquisition of 100 units at CLP 3,000, a confirmed sale of 10 units at CLP 10,000 with CLP 3,000 unit cost, and CLP 20,000 of operating expenses, the executive report shows sales of CLP 100,000, sold-item cost of CLP 30,000, operating expenses of CLP 20,000, and estimated result of CLP 50,000. The inventory acquisition of CLP 300,000 is not deducted again. Purchase-order records alone do not change those totals.
7. For that sale, record CLP 30,000 received in October and CLP 70,000 in November. The cash group shows CLP 30,000 received for October and CLP 70,000 for November; sales are counted once on the sale date. With the clock set before the November receipt, `Lo pendiente hoy` shows CLP 70,000 still to collect. It shows zero after the second dated receipt becomes applicable.
8. A CLP 300,000 confirmed purchase/expense with one CLP 100,000 outgoing movement shows CLP 200,000 still to pay. Linking that movement to both the expense and its purchase order does not double the payment. A purchase-order-only advance appears in the cash and attention groups without automatically settling every related expense.
9. A mixed purchase/expense containing CLP 300,000 of inventory and CLP 20,000 of operating expense contributes CLP 20,000 to operating-result expenses. The report shows CLP 300,000 of merchandise acquisitions separately from CLP 20,000 of running expenses; any payment contributes to cash only on its movement date.
10. On a separate sale originally worth CLP 100,000 with CLP 30,000 recorded cost, a CLP 10,000 commercial refund and matching header adjustment leave CLP 90,000 net sales. With no recorded cost recovery or other operating expenses, the report shows CLP 30,000 of sold-item cost and CLP 60,000 of estimated result. A separate CLP 10,000 outgoing cash refund appears on its actual date. The report explains that the commercial correction affects the original sale period. A confirmed receipt retained on a cancelled sale appears as money to return in `Lo pendiente hoy`.
11. A sold line with `unitCostAtSale = null` keeps its sales visible but marks its cost and related profit incomplete. The report lists the affected sale. A known `0` unit cost is treated as complete. A confirmed header without items or with inconsistent totals produces a visible review message and no falsely complete affected profit figure.
12. Two sale items and multiple receipts do not multiply either sales or receipts through joins. The summary and supporting highlights reconcile with their source-record aggregates. Product highlights are labelled as net merchandise sales rather than product profit, and top-three tables do not determine full-business totals.
13. Selecting CLP excludes USD/EUR values from the totals; selecting another recorded currency shows only that currency and its code. A foreign-currency movement cannot reduce a same-number commercial balance in CLP. That relation is flagged for review without omitting the actual movement from its own currency's cash group.
14. Relative date filters use the business time zone and still select the current month after the month of installation. Date boundaries are inclusive; summary, chart, and supporting datasets agree. Prior-period comparisons display their actual ranges, and a zero/incomplete comparison basis produces no infinite or misleading percentage. `Lo pendiente hoy` includes older unpaid documents even when the selected sales/cash period is this month.
15. The empty report opens successfully, with zero known totals, empty highlights, and a clear no-records message for the selected period. A period containing expenses but no sales shows a negative result when complete. With more than 1,000 matching source records, full summaries and top-three rankings remain correct because aggregation happens before display limits. Supporting tables clearly state their top-three/five-record limits rather than implying that they list every transaction.
16. At 390px and 1280px, the single report exposes usable filters, readable summaries/explanations, and accessible supporting tables. Its business, cash, pending, highlights, and attention groups can be read in one report. Loading, empty, incomplete, and error states can be distinguished. Reading it requires no SQL or report-definition editing.
17. Running the setup again creates no duplicate tenant, users, memberships, models, sections, views, shares, or sidebar placements. It preserves transactions, passwords, the user-edited report definition, labels, and existing navigation order. A stopped run can resume by creating only missing compatible setup objects. Conflicting tenant identity, globally used account addresses, incompatible model fields, or occupied view slugs fail with a specific message rather than deleting or overwriting existing data.
18. Install the same commerce definitions into a separate disposable test tenant with another slug, domain, currency default, and time zone. Both tenants have independent users, model/view IDs, records, saved report definitions, and sidebar access. Each installation contains exactly one report. Queries use the second tenant's physical views and settings without changing shared definitions or branching on the tenant's name. Editing Zipp's saved report or adding a Zipp sale does not change the second tenant's report or results.
19. With the business clock fixed at October 31, 2026, put the acquisition, sale, and unpaid operating expense from criterion 6 in October, link the October CLP 100,000 supplier payment from criterion 8 to that acquisition, and use the October/November receipts from criterion 7. For October in CLP, the executive report shows net sales of CLP 100,000, sold-item cost of CLP 30,000, operating expenses of CLP 20,000, estimated result of CLP 50,000, and merchandise acquisitions of CLP 300,000. It also shows CLP 30,000 received, CLP 100,000 paid, CLP 70,000 more paid than received, CLP 70,000 still to collect today, and CLP 220,000 still to pay today, including the unpaid operating expense. Changing the selected period to an empty September preserves today's pending amounts under their own label. Removing the sale cost makes profit incomplete without erasing known sales or cash. The top-three product/category highlights agree with independently aggregated source records for the same dates/currency.

## Implementation Notes

### Existing code to reuse

- Tenant/user setup: `backend/src/services/TenantService.ts`, `backend/src/services/UserService.ts`, and the creation patterns under `backend/src/db/scripts/sample/` and `backend/src/db/scripts/alci/`.
- Models and data: `backend/src/services/ModelService.ts`, `backend/src/services/DataService.ts`, `backend/src/types/schema.ts`, and `backend/src/repositories/ModelRepository.ts`.
- SQL model naming: `backend/src/utils/modelViewName.ts`. Logical view names come from `pluralSlug`; physical names also include tenant identity and can be shortened/hashed. Use returned physical names, not hand-built strings. Camel-case field IDs appear as snake-case SQL column aliases.
- Views/access/sidebar: `backend/src/services/ViewService.ts`, existing section/table-view scripts, and `docs/agent/hierarchical-menus.md`. Check current sidebar placement APIs rather than relying only on historical `parentId` examples.
- Reports: `backend/src/services/report/ReportTypes.ts`, `ReportDefinitionService.ts`, `ReportDatasetService.ts`, `ReportSqlEngine.ts`, `backend/src/repositories/ReportSqlRepository.ts`, and `backend/src/db/scripts/alci/20260906_01_create_contratistas_report.ts` as a definition/provisioning example.
- Frontend report blocks and layouts: `frontend/src/components/report/`, `frontend/src/views/ReportView.vue`, and `frontend/src/composables/useReportFilters.ts`.

Historical scripts are references, not maintained service contracts. Do not copy sample passwords, deprecated report filter shapes, hardcoded tenant-prefixed SQL, destructive view replacement, or an obsolete service overload. New selects use `inputType: 'select'` plus `valueType`; dates use the current date-filter shape. Persist the report definition through `validateForPersistence` before creating the view.

### Provisioning and reuse

Add new `YYYYMMDD_NN_description.ts` scripts under `backend/src/db/scripts/zipp/`. Keep preparation order explicit: validate identity/inputs, create or find tenant, create or find initial users/memberships, create or find models, create sections/table views, then validate/create the single report and grant administrator access. Follow the root ops CLI to run them; do not implement a separate execution path.

Keep reusable commerce definitions separate from the small Zipp entrypoints. A plain helper taking configuration and the normal service context is enough. Limit it to this installation's concrete needs; do not add a template registry, persistence layer, plugin API, or UI for templates. Preserve immutable dated-script behavior when organizing shared code, for example by retaining the initial definition revision and adding a new revision for later installations/updates.

Resolve the existing master user and use the tenant-scoped system-user lookup after creation. Never run the master-user recreation script as a prerequisite: it has destructive behavior. Use current user services so account creation includes membership. Initial credentials should be task-specific operator inputs such as `ZIPP_SYSTEM_PASSWORD` and `ZIPP_ADMIN_PASSWORD`; the reusable helper receives their values without owning Zipp-specific environment names. Existing complete accounts do not require new passwords on a rerun and are not reset or re-enabled.

Check for conflicts before writing each dependent group. Skip matching/compatible existing objects; preserve operator changes. If the same stable slug has an incompatible shape, report the conflict and stop. Do not drop models, replace reports, reassign an account from another tenant, delete records, or attempt a reset. Use available transactions where they fit current services, but do not promise one all-or-nothing transaction across all provisioning stages. A failed partial installation must be safely resumable.

No new business SQL tables or hand-built business API endpoints are needed. Store business records through the current custom-model APIs, retain the existing optional external/import identity support for future connectors, and use the generated GraphQL schema and model views. SQL calculations in reports do not imply automatic write-time calculations or domain workflows.

### Report query shape

Build independent sources at the necessary grain: sale header, sale item, purchase/expense header, purchase/expense item, and money movement. Aggregate item amounts and applied movements by parent ID before joining them. Join by tenant and parent IDs, not by human reference, product name, or SKU. A header's amount must never be repeated once for every item or every payment.

Use consistent source/aggregate builders within the single report definition so its summaries, trend, and highlights follow the same arithmetic and completeness rules. Keep period-filtered aggregates separate from today's pending snapshot. Reuse the existing text, chart, and table blocks; this does not require a new dashboard framework, report-composition runtime, or summary widget type.

Reuse the existing SQL safety compiler, runtime parameter binding, allowed-view validation, report database role, statement timeout, and row cap. Do not interpolate runtime filter values into SQL or relax report permissions to make a query run. Filter-option queries must use the same tenant-safe model views.

Compute dynamic date bounds and the explanatory period/currency/completeness text in datasets and display them through current dataset-backed text blocks. Do not create a new widget type for a summary card. Use compact supporting tables with references that identify source records in `Registros`, rather than creating separate detail reports. Preserve the existing loading/error behavior; use dataset-backed messages to distinguish empty data from incomplete financial data.

For pending amounts, use the same record-date and currency basis throughout, with movement totals computed independently from document items. Show commercial references/descriptions so the owner can find the source in `Registros`. Only explicit directly linked movements settle a document; an unlinked movement is never guessed into place by equal amount or similar text.

### Research and fit

- [Shopify ProductVariant](https://shopify.dev/docs/api/admin-graphql/latest/objects/ProductVariant) and [Order LineItem](https://shopify.dev/docs/api/admin-graphql/latest/objects/LineItem) support separate shared products, sellable variants, and sale items. This fits the existing single-record relations without adding embedded repeatable fields.
- [Shopify finance reports](https://help.shopify.com/en/manual/reports-and-analytics/shopify-reports/report-types/default-reports/finances-report) distinguish sales, payments, and missing-cost coverage. Use that separation while presenting ordinary business questions and readable supporting details.
- [Shopify profit reports](https://help.shopify.com/en/manual/reports-and-analytics/shopify-reports/report-types/default-reports/profit-reports) depend on recorded cost at sale. Five Panel likewise needs its own cost snapshot and an explicit missing-cost state; a variant's current cost is insufficient for previous sales.
- [IAS 2's inventory explanation](https://www.ifrs.org/issued-standards/list-of-standards/ias-2-inventories/) distinguishes inventory acquisition from expense associated with sold inventory. Use that distinction to avoid double-counting merchandise purchases, without claiming to implement the standard's full valuation rules or Zipp's statutory accounts.

The rejected simpler alternative is a single undifferentiated income/expense list: it loses the difference between a sale, its collection, a merchandise acquisition, and an operating expense. The rejected larger alternative is a full accounting/provisioning framework. The chosen scope uses the records needed now and the report tools Five Panel already has.

## Test Expectations

- Add focused tests under `backend/test/` for model definitions, relation/default metadata, setup preflight/rerun/resume behavior, initial-user membership, report validation, tenant-specific SQL view generation, filters, and administrator access. Use disposable credentials and data.
- Execute report SQL against a disposable PostgreSQL database containing both installation tenants. Assert actual dataset values for acceptance criteria 6–15, including inventory versus operating expense, split payments across months, a purchase-order advance later assigned to an expense, mixed expense lines, refunds, a cancelled sale retaining a receipt, missing versus zero cost, multiple item/payment joins, missing items, inconsistent totals, and currency mismatch. SQL string checks alone are not evidence that the financial results are correct.
- Verify criterion 19 against independent expected values from the source fixtures, including the fixed-date pending snapshot, period changes, incomplete profit, and top-three highlights. Confirm there is exactly one report view and sidebar entry for the installation, with business, cash, pending, highlights, and attention groups present together on desktop and mobile.
- Freeze or inject the relevant business date for date-boundary tests. Cover month rollover, week start, a custom one-day/range selection, previous-period comparison with a zero basis, and an older unpaid document. Use the configured business time zone rather than the test runner's local zone.
- Cover more than 1,000 source records while checking that totals and rankings include all matching records before applying the compact tables' display limits. Keep fixtures limited to what verifies that behavior.
- Reuse existing report-runtime tests for unchanged widgets and filters. Add frontend tests only for behavior changed by this work, rather than duplicating existing generic component tests.
- Verify the actual running app through the normal Docker/ops flow: administrator login; the ten record views and one report entry; manual creation of a small linked data set; summary/highlight amounts matching source records; state distinctions; currency/date filtering; and readable desktop/mobile output. Confirm a second tenant cannot access Zipp's report or records. Do not use real customer data for evidence.
- Run the focused affected tests and required backend/frontend checks if those packages change. Record the commands and results in the implementation PR. This spec-only change does not execute provisioning, product tests, or production actions.

## Risks

- Incorrect manual totals, classifications, dates, or cost inputs can produce incomplete figures. The report must surface known inconsistencies, but cannot detect an expense the owner never recorded or an incorrectly entered yet plausible cost. `Ganancia estimada` must not imply complete company accounting.
- Net operating estimates, current-record commercial adjustments, and actual dated cash movements answer different questions. Labels and explanations are part of the feature, not optional copy. Taxes, inventory losses, capital assets, loans, and other accounting cases need later work if the business requires them.
- The executive report puts period activity and today's unpaid amounts together. Keep their date labels visible and use consistent calculations across its datasets, so it cannot silently hide old debts or present a cash difference as profit or available balance.
- This issue preserves dated receipts/payments but not historical versions of all commercial fields. A later correction can change an earlier sale-period estimate; it must not change the stored dates/amounts of unrelated actual money movements.
- The present single-record forms require separate header/item entry. Simple reading of the report is included; a guided nontechnical entry experience is a separate feature. The report must not conceal this limitation by promising actions the application does not provide.
- Reusable definitions can accidentally bake in Zipp identity or make old scripts change behavior. A second-tenant installation test and explicit definition revisions are required to keep reuse and existing tenants independent.
- The report row cap and timeout still apply. Aggregate all source records before taking the top-three/five-record highlights; do not use a shortened list as a source of business totals.
- Setup can span multiple transactions. Preflight checks, conflict errors, preservation of existing objects, and safe resumption are necessary to avoid duplicate or destructive reruns.

## Open Questions

None. The scope is the commerce models, individual receipts/payments, and one executive report. The implementation defaults and limitations above are explicit; initial passwords are execution inputs rather than unresolved product decisions.
