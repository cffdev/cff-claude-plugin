---
name: cashflowfrog
description: Track, analyze, plan, and report on business cash flow using Cash Flow Frog, connected to Xero or QuickBooks Online. Use this skill for current cash position and runway, cash flow forecasts and projections, accounts receivable (invoices, estimates, sales orders), accounts payable (bills, purchase orders), what-if scenario modeling, budget vs actual variance analysis, category and account mapping, PDF report generation, and custom data exports. Requires an active Cash Flow Frog account with a connected accounting integration.
---

# Cash Flow Frog

Cash Flow Frog connects Claude to live accounting data for end-to-end cash flow intelligence: tracking actual balances, forecasting future positions, managing AR/AP timing, planning scenarios, and generating reports.

## Session Setup — Always Do This First

Every session, before any other tool call:

1. **`listCompanies`** — get the `id` (companyId) and note `accountingSoftware` (XERO or QUICKBOOKS).
2. **`listScenarios`** — get the available scenarios. Use `active: true` as the default unless the user specifies otherwise.

If the user has multiple companies, ask which one before proceeding. Never assume or reuse a companyId from a prior turn without confirming.

> **Connection check:** Handle these states in order:
> - **MCP connection fails entirely** (new user, no token): No tools are reachable. Tell the user: "It looks like you don't have a Cash Flow Frog account connected yet. Sign up at [accounts.cashflowfrog.com/signup](https://accounts.cashflowfrog.com/signup) to connect your Xero or QuickBooks account, then come back."
> - **`getIntegrationStatus` shows PENDING**: The integration is being set up — tell the user it typically takes a few minutes and suggest they try again shortly. Do **not** call `getOnboardingUrl`.
> - **`listCompanies` returns empty or `getIntegrationStatus` shows DISCONNECTED/ERROR**: Call `getOnboardingUrl()` — no parameters needed. Present the returned `url` as a clickable link with the `message` as context. After the user confirms they've completed the flow, call `listCompanies` again to verify.

---

## Workflow Patterns

### Cash Position & Forecasting

**Trigger phrases:** "cash position", "how much cash do I have", "runway", "will I go negative", "projected balance", "bank balance"

- **Quick snapshot** → `getCashPosition` — returns current balance, 30/60/90-day projections, lowest projected balance, and first date balance goes negative. Start here for any cash health question.
- **Risk scan** → `detectNegativeBalancePeriods` — scans the full forecast horizon and returns every contiguous at-risk period with minimum balance.
- **Full forecast timeline** → `getCashFlowForecast` — combines bank balances with all AR/AP/projections into a per-period breakdown. Choose grouping based on urgency: DAILY/WEEKLY for near-term cash concerns, MONTHLY for planning, QUARTERLY for high-level overview.
- **Bank account breakdown** → `listBankAccounts` — individual account balances.
- **Balance on a specific future date** → `getBankBalance(date)`.

**Data staleness:** If the user asks why the forecast looks off or data seems old:
1. `getIntegrationStatus` — check connection status and last sync timestamps.
2. If stale and connected, offer to call `syncCompanyData` (respects daily sync limit; inform user if rate-limited).

---

### Cash Flow Analysis & Breakdown

**Trigger phrases:** "where is my money going", "cash in vs cash out", "breakdown by category", "by customer", "by project", "budget vs actual", "variance"

- **By category** → `getCashFlowBreakdown(groupBy: CATEGORY)` — the primary analysis tool. Returns inflows and outflows by cash flow category with period-by-period totals.
- **By customer or vendor** → `getCashFlowBreakdown(drillDownBy: CUSTOMER_OR_VENDOR)`.
- **By project / department / class / location** → `getCashFlowBreakdown(groupBy: PROJECT | DEPARTMENT | CLASS | LOCATION)`.
- **Xero tracking categories** → `getCashFlowBreakdown(groupBy: XERO_TRACKING)` — requires `xeroTrackingCategoryId` from `listDimensions`.
- **Drill into a specific category** → pass `groupId` from `listCashFlowCategories` as the filter in `getCashFlowBreakdown`.
- **Budget vs actual** → `getCategoryVariance` — compares actual cash flows against planned amounts by category. Positive variance = favorable. Use historical date ranges for meaningful results.
- **Individual transactions** → `listTransactions` — **only for targeted lookups** (specific counterparty, account, keyword, transaction type, or minimum amount). Not for aggregation or bulk analytics — use `getCashFlowBreakdown` for those.

**Tip:** The `categoryId` from `listCashFlowCategories` is interchangeable with `groupId` in `getCashFlowBreakdown` and `getCategoryVariance` — use it to drill from summary to detail.

---

### Accounts Receivable (AR)

**Trigger phrases:** "outstanding invoices", "who owes me money", "overdue AR", "when am I getting paid", "receivables", "estimates", "sales orders"

**Read / summarize:**
- `getARSummary` — high-level: total open balance, overdue balance, counts, oldest overdue age, largest open invoice.
- `listInvoices` — full list with expected collection dates. Use `overdueOnly: true` to focus on overdue. Filter by `expectedDateFrom/To` to see what's forecasted in a period.
- `getEstimateSummary` / `listEstimates` — open quotes not yet invoiced.
- `getSalesOrderSummary` / `listSalesOrders` — open customer orders not yet fully invoiced.

**Reschedule / update (directly changes the forecast):**
- Single invoice date → `setInvoiceExpectedCollectionDate`
- Split invoice across installments → `splitInvoiceCollection(scheduleLines: [{date, amount}])`
- Clear a split, revert to single date → `clearInvoiceCollectionSchedule`
- Bulk update multiple invoices → `bulkSetInvoiceExpectedCollectionDate`
- Estimates follow the same pattern: `setEstimateExpectedCollectionDate`, `splitEstimateCollection`, `clearEstimateCollectionSchedule`
- Sales orders follow the same pattern: `setSalesOrderExpectedCollectionDate`, `splitSalesOrderCollection`, `clearSalesOrderCollectionSchedule`

**Key rule:** `expectedCollectionDate` is what drives the forecast. If null, `dueDate` is used as fallback. Setting it is how users tell the forecast "this will actually come in on this date."

---

### Accounts Payable (AP)

**Trigger phrases:** "outstanding bills", "what do I owe", "overdue AP", "upcoming payments", "payables", "purchase orders", "vendor payments"

**Read / summarize:**
- `getAPSummary` — high-level: total open balance, overdue balance, counts, oldest overdue age, largest open bill.
- `listBills` — full list with expected payment dates. Use `overdueOnly: true`, or filter by expected/due date ranges.
- `getPurchaseOrderSummary` / `listPurchaseOrders` — open vendor orders not yet fully billed.

**Reschedule / update (directly changes the forecast):**
- Single bill date → `setBillExpectedPaymentDate`
- Split bill across installments → `splitBillPayment(scheduleLines: [{date, amount}])`
- Clear a split → `clearBillPaymentSchedule`
- Bulk reschedule for cash conservation → `bulkSetBillExpectedPaymentDate`
- Purchase orders follow the same pattern: `setPurchaseOrderExpectedPaymentDate`, `splitPurchaseOrderPayment`, `clearPurchaseOrderPaymentSchedule`

**Key rule:** `expectedPaymentDate` drives when bill outflows appear in the forecast. Moving it out pushes the cash outflow into the future — useful for modeling payment deferral scenarios.

---

### Scenario Planning & Projections

**Trigger phrases:** "what if", "model a scenario", "new hire", "new lease", "add recurring expense", "add projection", "best case", "worst case", "compare scenarios"

**Scenarios:**
- `listScenarios` — see all scenarios; `active: true` is the main view.
- `createScenario(name, description)` — create a named scenario (e.g., "Expansion — 3 New Hires"). Always give it a clear, descriptive name.
- `updateScenario` — rename or update description.
- `deleteScenario` — **irreversible**; cannot delete the base scenario. Confirm with user first.
- `compareScenariosForPeriod(scenarioIds: [2–5])` — side-by-side comparison with per-period breakdown.

**Adding projections (prefer these in order of simplicity):**

1. **`addRecurringExpense`** — simplest for recurring cash out (rent, salaries, subscriptions, loan payments). Before calling: `listDimensionOptions(ACCOUNT)` to get `accountApiId`; optionally `listDimensionOptions(VENDOR)` for `vendorId`.

2. **`addRecurringIncome`** — simplest for recurring cash in (retainers, subscription revenue, rental income). Before calling: `listDimensionOptions(ACCOUNT)` for `accountApiId`; optionally `listDimensionOptions(CUSTOMER)` for `customerId`.

3. **`createProjection`** — for full control over dimensions and recurrence. Always resolve account and counterparty ids via `listDimensionOptions` first. See `references/workflows.md` for the full flow.

4. **`bulkCreateProjections`** — for creating many projections at once (up to 500). Follows a strict 6-step workflow — see `references/workflows.md`.

**Editing projections:**
- `listProjections` — filter by date range, direction, category, or search term.
- `updateProjection` — replaces all fields; get current state with `getProjection` first, modify, then call.
- `deleteProjection` — **irreversible** and removes all future occurrences. Do NOT use on `recurringTemplate: true` projections — those are managed in the accounting software.
- To stop a recurring projection from a future date without deleting history: `updateProjection` with `endDate` set to today.

---

### Category & Account Setup

**Trigger phrases:** "set up categories", "account mapping", "account type", "why is X in the wrong bucket", "unmapped accounts"

- `listCashFlowCategories` — list categories (called "Account Types" in the UI). Each has type: CASH_IN, CASH_OUT, or ANY.
- `listAccounts` — chart of accounts with current category mapping. `categoryId: null` means unmapped — those transactions fall into the "OTHER" bucket.
- `setAccountCategory(accountId, categoryId)` — map an account to a category. `categoryId` must come from `listCashFlowCategories`.
- `clearAccountCategory` — unmap an account (idempotent).
- `createCashFlowCategory` — create a new category. Name must be unique; pick type carefully (CASH_IN / CASH_OUT / ANY).
- `updateCashFlowCategory` — rename, change type, or disable. Null fields are left unchanged.
- `deleteCashFlowCategory` — fails if accounts or projections still reference it; clear those first.

---

### Reporting

**PDF report with charts:**
When the user asks for a PDF, chart, graph, or visual summary → call `generateCashFlowReportPdf` immediately. **Do not fetch data first** — it fetches its own data internally. Returns a 15-minute download link. This is the only tool that produces charts and graphs.

Standard pages: COVER, OVERVIEW, CATEGORY_BREAKDOWN, CASH_BALANCE, CASH_IN_BREAKDOWN, CASH_OUT_BREAKDOWN, CUSTOMER_INSIGHTS, NOTES. Pass `pages` to configure specific sections.

**Custom reports (Excel / CSV exports):**
- `listCustomReports` — discover saved report definitions.
- `exportCustomReport(reportId, startDate, endDate)` — export a saved report; returns 15-minute download link. Use for standard-sized exports.
- Two-stage flow for large exports → `prepareCustomReportExport` → transform data per `processingPrompt` → `finalizeCustomReportExport`. Export token expires in 15 minutes.
- `getReportBuilderStep(step: 1)` — begin the intake flow to build and save a new custom report definition.

**Company settings:**
- `getCompanySettings` — read all forecast settings grouped by area (projections, invoices, bills, estimates, salesOrders, purchaseOrders, cashBalance, advanced).
- `updateCompanySettings` — patch semantics: only provided fields are changed. Read current state first.

---

## Proactive Offers — Surface Depth Without Being Asked

After answering a question, always offer the most natural next step. These are the highest-value follow-ups:

**After cash position or forecast:**
- "Want me to break this down by category to see where cash is coming from and going?"
- "Should I scan for any periods where your balance could go negative?"
- "Would you like to compare this against another scenario?"
- "I can generate a PDF report with charts if you'd like to share this."

**After a category breakdown (`getCashFlowBreakdown`):**
- "I can drill into [largest category] to show you the specific customers/vendors driving it."
- "Want to see how this compares to your plan? I can run a budget vs. actual comparison."
- "Should I break this down by project or department instead?"

**After showing AR summary or invoice list:**
- "Would you like me to reschedule any of these to update your forecast?"
- "I can also pull in open estimates and sales orders for the full receivables picture."
- "Want to see how collecting these on time (vs. late) changes your 90-day cash position?"

**After showing AP summary or bill list:**
- "If cash is tight, I can show you which bills could be safely pushed out and by how much."
- "I can run a bulk reschedule to model a payment deferral strategy."
- "Want to see open purchase orders as well, for the full payables picture?"

**After creating or modifying a scenario:**
- "Want me to compare this scenario against your base forecast over the next quarter?"
- "Should I add any recurring income or expense projections to flesh it out?"

**After any analysis:**
- "Want a PDF report of this for your records or to share with your team?"

---

## Key Rules

1. **Never guess companyId or scenarioId.** Always retrieve from `listCompanies` and `listScenarios`.
2. **`listTransactions` is for targeted lookups only** — always apply at least one specific filter beyond the date range. Use `getCashFlowBreakdown` or `getCashFlowForecast` for aggregations.
3. **PDF / chart requests go to `generateCashFlowReportPdf` immediately** — call it without pre-fetching data.
4. **`deleteProjection` and `deleteScenario` are irreversible** — always confirm with the user first.
5. **Do not call `deleteProjection` on `recurringTemplate: true` projections** — those are managed in the accounting software, not Cash Flow Frog.
6. **`updateProjection` replaces all fields** — always call `getProjection` first, then modify and send the full object.
7. **Reporting gaps:** When the user requests something genuinely impossible with available tools, explain the limitation first, then call `reportMissingCapability`.

## Tool Reference

See [`references/tools.md`](references/tools.md) for the complete tool catalog with parameters.
See [`references/workflows.md`](references/workflows.md) for multi-step workflow recipes (bulkCreateProjections, custom report export, new scenario setup).
