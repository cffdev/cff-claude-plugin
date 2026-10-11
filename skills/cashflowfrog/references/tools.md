# Cash Flow Frog — Tool Reference

Complete tool catalog. Grouped by domain. Each entry shows: tool name, what it does, and key parameters.

---

## Company & Setup

### `listCompanies()`
REQUIRED FIRST STEP in every session. Lists all companies the authenticated user has access to. Returns: `id` (use as `companyId` everywhere), `name`, `currencyCode`, `timezone`, `accountingSoftware` (XERO | QUICKBOOKS).

### `getOnboardingUrl()`
Returns the appropriate URL for the authenticated user to set up or reconnect their accounting integration. No parameters — the server determines the correct URL from auth context and integration state. Returns:
- `url`: the link to present to the user
- `action`: `"setup"` (no companies yet — redirects to add company) | `"reconnect"` (integration disconnected — redirects to reconnect flow with `?companyId=`) | `"connected"` (edge case; all integrations connected)
- `message`: human-readable explanation to show alongside the link

Call when `listCompanies` returns empty, or when `getIntegrationStatus` shows DISCONNECTED or ERROR. Do **not** call for PENDING status (user should wait, not reconnect).

### `getIntegrationStatus(companyId)`
Returns accounting software connection status: platform (XERO | QUICKBOOKS), status (CONNECTED | DISCONNECTED | ERROR | PENDING), last sync attempt, last successful sync, and any error message. Call when the user asks about data freshness or sync problems.

### `syncCompanyData(companyId)`
Triggers a full sync from the connected accounting software. Runs synchronously. If the daily manual sync limit is reached, `rateLimitExceeded: true` — inform the user and do not retry.

### `getCompanySettings(companyId)`
Returns all forecast settings grouped by area: projections, invoices, bills, estimates, salesOrders, purchaseOrders, cashBalance, advanced.

### `updateCompanySettings(companyId, settings)`
Patch semantics — only provided fields are changed. Read current state with `getCompanySettings` first. Returns full updated settings.

---

## Cash Position & Forecasting

### `getCashPosition(companyId, scenarioId, horizonDays?)`
Cash health snapshot. Returns: current bank balance, projected balances at 30/60/90 days (up to `horizonDays`, default 90), lowest projected balance within horizon, first date balance goes negative (if any). Starting point for any cash health question.
- `horizonDays`: 30 | 60 | 90 | 180

### `detectNegativeBalancePeriods(companyId, scenarioId, startDate?, endDate?, threshold?)`
Scans forecast and returns all contiguous date ranges where projected balance drops below `threshold` (default: 0). Each period includes: start date, end date, minimum balance. Also returns overall minimum and cash runway in days.
- `startDate`: default today
- `endDate`: default 365 days out
- `threshold`: default 0

### `getCashFlowForecast(companyId, scenarioId, startDate, endDate, grouping, dimensionFilters?)`
Primary forecast tool. Combines actual bank balances with all AR/AP/projections. Returns: opening/closing balance for full period, total inflows/outflows, per-period breakdown with balance and cash movements.
- `grouping`: DAILY | WEEKLY | MONTHLY | QUARTERLY (Required — always provide; default to MONTHLY if user doesn't specify)
- `dimensionFilters`: list of `{dimensionType, value}` to scope to a specific customer/vendor/project

### `getBankBalance(companyId, scenarioId, date?)`
Projected (not live) total bank balance on a specific future date. Date defaults to today.

### `listBankAccounts(companyId)`
Lists all bank and financial accounts with current balances. Returns: `id`, `name`, `type` (Bank | CreditCard), `subType` (Checking | Savings | etc.), `balance`, `currencyCode`, `active`.

---

## Cash Flow Analysis

### `getCashFlowBreakdown(companyId, scenarioId, startDate, endDate, grouping, groupBy?, drillDownBy?, groupId?, xeroTrackingCategoryId?, minGroupAmount?)`
Categorized breakdown of inflows and outflows. More granular than `getCashFlowForecast`.
- `grouping`: DAILY | WEEKLY | MONTHLY | QUARTERLY (Required)
- `groupBy`: CATEGORY (default) | CLASS | DEPARTMENT | LOCATION | PROJECT | XERO_TRACKING
- `drillDownBy`: NONE (default) | CUSTOMER_OR_VENDOR | ACCOUNT | SUBCLASS
- `groupId`: filter to a specific category (id from `listCashFlowCategories`)
- `xeroTrackingCategoryId`: required when `groupBy: XERO_TRACKING`. Get available Xero tracking categories by calling `listDimensions(companyId)` — find entries with type `XERO_TRACKING` and use their `id` as this value.
- Output: `periods[]`, `cashIn/cashOut` with `total`, `totalsByPeriod`, `groups[]` (each group has `label`, `groupId`, `total`, `totalsByPeriod`, `drillDown[]`)

### `getCategoryVariance(companyId, scenarioId, startDate, endDate, grouping, minGroupAmount?)`
Actual vs planned variance per cash flow category. Returns: `periods[]`, `cashIn/cashOut` with aggregate and per-category `actual`, `planned`, `variance` (actual − planned), `variancePct`. Positive variance = favorable. Use historical date ranges. Pass `categoryId` as `groupId` to `getCashFlowBreakdown` for drill-down.

### `listTransactions(companyId, scenarioId, startDate, endDate, ...filters)`
Individual transaction rows (historical + forecast). **Targeted lookups only — always apply at least one specific filter.** Not for aggregation.
- `kind`: HISTORY | FORECAST (omit for both)
- `type`: INVOICE | BILL | ESTIMATE | MANUAL | RECURRING_TXN | SALES_ORDER | PURCHASE_ORDER | DEPOSIT | VAT
- `search`, `direction` (CASH_IN | CASH_OUT), `counterpartyId`, `counterpartyName`, `categoryId`, `accountId`, `minAmount`, `maxAmount`
- Paginated: `pageNumber`, `pageSize`

---

## Accounts Receivable

### `getARSummary(companyId, scenarioId)`
High-level AR snapshot: total open balance, total overdue balance, open and overdue counts, age of oldest overdue invoice (days), largest open invoice.

### `listInvoices(companyId, scenarioId, ...filters)`
Open invoices (unpaid only). `expectedCollectionDate` drives the forecast; falls back to `dueDate` if null. Sorted by expected collection date.
- Optional filters: `search`, `overdueOnly`, `dueDateFrom/To`, `expectedDateFrom/To`, `minAmount/maxAmount`

### `getInvoice(companyId, scenarioId, invoiceId)`
Single invoice with expected collection date and any split schedule.

### `setInvoiceExpectedCollectionDate(companyId, scenarioId, invoiceId, expectedCollectionDate)`
Set when this invoice will be collected. Changes when the inflow appears in the forecast.

### `splitInvoiceCollection(companyId, scenarioId, invoiceId, scheduleLines)`
Split collection across multiple dates. `scheduleLines`: `[{date, amount}]`. Fully replaces any existing split.

### `clearInvoiceCollectionSchedule(companyId, scenarioId, invoiceId)`
Remove split schedule, revert to single expected date. Idempotent.

### `bulkSetInvoiceExpectedCollectionDate(companyId, scenarioId, updates)`
Update expected collection date for multiple invoices. `updates`: `[{invoiceId, expectedCollectionDate}]`. Partial success is normal — returns success count + per-invoice failures.

### `getEstimateSummary(companyId, scenarioId)`
High-level summary of open estimates: total amount, count, expiring-soon count (within 30 days), largest estimate.

### `listEstimates(companyId, scenarioId, ...filters)`
Open sales estimates (uninvoiced balance only). Optional: `search`, `startDate/endDate` (issue date), `included`, `minAmount/maxAmount`.

### `getEstimate(companyId, scenarioId, estimateId)`
Single estimate with expected collection date and any split schedule.

### `setEstimateExpectedCollectionDate(companyId, scenarioId, estimateId, expectedDate)`
### `splitEstimateCollection(companyId, scenarioId, estimateId, scheduleLines)`
### `clearEstimateCollectionSchedule(companyId, scenarioId, estimateId)`
Same pattern as invoice equivalents.

### `getSalesOrderSummary(companyId, scenarioId)`
High-level summary of open sales orders: total amount, count, largest order.

### `listSalesOrders(companyId, scenarioId, ...filters)`
Open customer orders (uninvoiced balance only). Optional: `search`, `startDate/endDate`, `included`, `minAmount/maxAmount`.

### `getSalesOrder(companyId, scenarioId, salesOrderId)`
Single sales order with expected collection date and any split schedule.

### `setSalesOrderExpectedCollectionDate(companyId, scenarioId, salesOrderId, expectedDate)`
### `splitSalesOrderCollection(companyId, scenarioId, salesOrderId, scheduleLines)`
### `clearSalesOrderCollectionSchedule(companyId, scenarioId, salesOrderId)`
Same pattern as invoice equivalents.

---

## Accounts Payable

### `getAPSummary(companyId, scenarioId)`
High-level AP snapshot: total open balance, total overdue balance, open and overdue counts, age of oldest overdue bill (days), largest open bill.

### `listBills(companyId, scenarioId, ...filters)`
Open bills (unpaid only). `expectedPaymentDate` drives the forecast; falls back to `dueDate` if null. Sorted by expected payment date.
- Optional filters: `search`, `overdueOnly`, `dueDateFrom/To`, `expectedDateFrom/To`, `minAmount/maxAmount`

### `getBill(companyId, scenarioId, billId)`
Single bill with expected payment date and any split schedule.

### `setBillExpectedPaymentDate(companyId, scenarioId, billId, expectedPaymentDate)`
Set when this bill will be paid. Changes when the outflow appears in the forecast.

### `splitBillPayment(companyId, scenarioId, billId, scheduleLines)`
Split payment across multiple dates. `scheduleLines`: `[{date, amount}]`. Fully replaces any existing split.

### `clearBillPaymentSchedule(companyId, scenarioId, billId)`
Remove split schedule, revert to single expected date. Idempotent.

### `bulkSetBillExpectedPaymentDate(companyId, scenarioId, updates)`
Update expected payment date for multiple bills. `updates`: `[{billId, expectedPaymentDate}]`. Partial success is normal.

### `getPurchaseOrderSummary(companyId, scenarioId)`
High-level summary of open purchase orders: total amount, count, largest order.

### `listPurchaseOrders(companyId, scenarioId, ...filters)`
Open vendor orders (unbilled amount only — represents expected cash outflow). Optional: `search`, `startDate/endDate`, `included`, `minAmount/maxAmount`.

### `getPurchaseOrder(companyId, scenarioId, purchaseOrderId)`
Single purchase order with expected payment date and any split schedule.

### `setPurchaseOrderExpectedPaymentDate(companyId, scenarioId, purchaseOrderId, expectedDate)`
### `splitPurchaseOrderPayment(companyId, scenarioId, purchaseOrderId, scheduleLines)`
### `clearPurchaseOrderPaymentSchedule(companyId, scenarioId, purchaseOrderId)`
Same pattern as bill equivalents.

---

## Scenarios

### `listScenarios(companyId)`
Lists all scenarios. `active: true` = the scenario shown on the main cash flow view. Use this as the default.

### `createScenario(companyId, name, description?)`
Create a new scenario. Give it a clear, descriptive name (e.g., "New Office Lease — 2025").

### `updateScenario(companyId, scenarioId, name, description)`
Rename or update description. Does not affect projections.

### `deleteScenario(companyId, scenarioId)`
**Irreversible.** Deletes scenario and all its projections. Cannot delete the base/active scenario. Confirm with user first.

### `compareScenariosForPeriod(companyId, scenarioIds, startDate, endDate, grouping?)`
Side-by-side forecast for 2–5 scenarios. Returns one forecast per scenario in the same order, each with opening/closing balance, total inflows/outflows, and per-period breakdown.
- `grouping`: default MONTHLY

---

## Projections

### `listProjections(companyId, scenarioId, ...filters)`
Returns up to 500 projections. Note: `recurringTemplate: true` = from accounting software (do not delete); `patternDetected: true` = auto-generated by CFF.
- Filters: `startDate`, `endDate`, `direction` (CASH_IN | CASH_OUT), `search`, `categoryId`, `relatedEntityId/Name`, `dimensionFilters`, `active`, `minAbsAmount`, `maxAbsAmount`, `recurringTemplate`, `patternDetected`

### `getProjection(companyId, scenarioId, projectionId)`
Single projection by id.

### `addRecurringExpense(companyId, scenarioId, description, amount, frequency, startDate, endDate?, vendorId?, accountApiId)`
Simplified recurring cash-out. Handles direction and VENDOR dimension automatically.
- `frequency`: ONCE | DAILY | WEEKLY | BIWEEKLY | MONTHLY | QUARTERLY | YEARLY
- Get `accountApiId` from `listDimensionOptions(ACCOUNT)`; optionally get `vendorId` from `listDimensionOptions(VENDOR)`
- Use for: rent, salaries, subscriptions, loan repayments

### `addRecurringIncome(companyId, scenarioId, description, amount, frequency, startDate, endDate?, customerId?, accountApiId)`
Simplified recurring cash-in. Handles direction and CUSTOMER dimension automatically.
- Same `frequency` options as above
- Get `accountApiId` from `listDimensionOptions(ACCOUNT)`; optionally get `customerId` from `listDimensionOptions(CUSTOMER)`
- Use for: retainer payments, subscription revenue, rental income

### `createProjection(projection)`
Full control. Required fields: `account`, `date`, `recurrenceType`, `amount`, and CUSTOMER (income) or VENDOR (expense) dimension. Category is auto-derived from account.
- `recurrenceType`: NONE | DAILY | WEEKLY | BIWEEKLY | SEMIMONTHLY | MONTHLY | QUARTERLY | SEMIANNUAL | ANNUAL
- Always resolve account and counterparty ids via `listDimensionOptions` first.

### `updateProjection(projectionId, projection)`
Replaces all fields. Call `getProjection` first, modify, then send full object. Must preserve CUSTOMER or VENDOR dimension. Changes apply to all future occurrences.

### `bulkCreateProjections(companyId, scenarioId, projections)`
Create up to 500 projections. Follows a 6-step workflow — see `workflows.md`. Invalid items returned in `errors[]`. `companyId`/`scenarioId` from individual items are ignored; use top-level params.

### `deleteProjection(companyId, scenarioId, projectionId)`
**Irreversible.** Deletes projection and all its occurrences. Do NOT use on `recurringTemplate: true` projections. To stop from a future date without deleting history, set `endDate` to today via `updateProjection`.

---

## Categories & Accounts

### `listCashFlowCategories(companyId, type?, includeDisabled?)`
Lists cash flow categories ("Account Types" in UI). `type`: CASH_IN | CASH_OUT | ANY. Disabled categories hidden by default.

### `getCashFlowCategory(companyId, categoryId)`
Single category by id.

### `createCashFlowCategory(companyId, name, type)`
Create a new category. Name must be unique. `type`: CASH_IN | CASH_OUT | ANY (use ANY only when genuinely applies to both directions).

### `updateCashFlowCategory(companyId, categoryId, name?, type?, disabled?)`
Null fields are left unchanged.

### `deleteCashFlowCategory(companyId, categoryId)`
Fails if accounts or projections still reference it. Categories synced from accounting system are soft-disabled instead of hard-deleted.

### `listAccounts(companyId, search?, categoryId?, includeInactive?)`
Chart of accounts with current category mapping. `categoryId: null` = unmapped (falls into "OTHER" bucket). `search` matches account name or category name.

### `getAccount(companyId, accountId)`
Single account with current category mapping.

### `setAccountCategory(companyId, accountId, categoryId)`
Map an account to a category. Idempotent. `categoryId` must come from `listCashFlowCategories`.

### `clearAccountCategory(companyId, accountId)`
Remove account's category mapping. Idempotent.

---

## Dimensions

### `listDimensions(companyId)`
Lists available dimension types: CUSTOMER, VENDOR, CLASS (QuickBooks), DEPARTMENT, LOCATION, PROJECT. Returns: `id`, `name`, `type`, whether custom.

### `listDimensionOptions(companyId, dimensionType, search?)`
Lists selectable values for a dimension type. Returns up to 200; `truncated: true` if more exist. The `value` field is what goes in the `account` field (for ACCOUNT) or `dimensions` entry (for other types) when creating projections.
- `dimensionType`: ACCOUNT | CLASS | CUSTOMER | DEPARTMENT | LOCATION | PROJECT | VENDOR | CUSTOM
- Call once per unique counterparty name, not all upfront.
- Required before `createProjection` and `bulkCreateProjections`.

---

## Reporting

### `generateCashFlowReportPdf(companyId, scenarioId, startDate, endDate, grouping, ...options)`
Generates a PDF report with bar and line charts. **The only tool that produces charts/graphs.** Call immediately when the user asks for PDF, chart, graph, or visual summary — do not pre-fetch data. Returns a 15-minute download link.
- `grouping`: DAY | WEEK | MONTH | QUARTER
- `cashFlowType`: ANY (default) | CASH_IN | CASH_OUT
- `topLevelGroupBy`: ACCOUNT_TYPE | QB_CLASS | XERO_TRACKING_CATEGORY | DEPARTMENT | LOCATION | PROJECT
- `drillDownGroupBy`: RELATED_ENTITY | ACCOUNT | SUBCLASS
- `pages`: optional per-page config. Standard pages: COVER, OVERVIEW, CATEGORY_BREAKDOWN, CASH_BALANCE, CASH_IN_BREAKDOWN, CASH_OUT_BREAKDOWN, CUSTOMER_INSIGHTS, NOTES
- `dimensionFilters`: optional scoping
- `xeroTrackingCategoryId`: required when `topLevelGroupBy: XERO_TRACKING_CATEGORY`

### `listCustomReports()`
List all saved custom AI report definitions. Use to discover available reports before exporting.

### `exportCustomReport(reportId, startDate, endDate)`
Export a saved custom report with live data. Returns 15-minute presigned download link. If result is too large, suggest a shorter date range or the two-stage flow.

### `prepareCustomReportExport(reportId, startDate, endDate)`
Stage 1 of two-stage export. Fetches cash flow data and returns it with `processingPrompt` and a signed `exportToken`. Claude transforms the data per `processingPrompt`, then passes to `finalizeCustomReportExport`. Token expires in 15 minutes.

### `finalizeCustomReportExport(exportToken, headers, rows)`
Stage 2 of two-stage export. Takes Claude's transformed data, builds XLSX/CSV, uploads, returns 15-minute presigned download link.
- `headers`: list of column name strings
- `rows`: list of lists; use JSON numbers for numeric cells

### `saveCustomReport(name, description, processingPrompt, groupBy, scenarioMode, outputFormat, ...options)`
Save a new custom AI report definition.
- `groupBy`: MONTH | QUARTER | WEEK
- `scenarioMode`: FIXED | USER_SELECTED | COMPARISON
- `outputFormat`: CSV | XLSX
- `aggregationDimensions`: PERIOD | PROJECT | DEPARTMENT | LOCATION | CLASS | CATEGORY | COUNTERPARTY | TYPE
- Schema A: raw transactions (`aggregationDimensions` empty); Schema B: pre-aggregated (`aggregationDimensions` non-empty)

### `updateCustomReport(reportId, ...same fields as saveCustomReport)`
Full replacement — include all fields.

### `uploadReportTemplate(reportId, attachmentId)`
Links an Excel template file to a saved report. Call immediately after `saveCustomReport` when `pageContext` contains `pendingTemplateAttachmentId`.

### `getReportBuilderStep(step)`
Returns instructions for the current step of the custom report intake flow.
- `step: 1` — start (template upload or requirements gathering)
- `step: 2` — draft and save

---

## Utility

### `reportMissingCapability(description, userRequest)`
Report a capability gap to the Cash Flow Frog team. Call only when the user requests something genuinely impossible with available tools — not for temporary errors, auth issues, or rephrasing. Always explain the limitation to the user first, then call this tool.
- `description`: concise capability description
- `userRequest`: verbatim or close paraphrase of what the user asked
