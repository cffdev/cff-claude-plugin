# CashFlow Frog — Multi-Step Workflow Recipes

Step-by-step guides for the workflows that require a specific sequence of tool calls.

---

## New Scenario with Projections

Use when the user wants to model a "what if" situation.

1. `listCompanies` → get `companyId`
2. `listScenarios` → confirm active scenario; decide if user wants to modify it or create a new one
3. `createScenario(name, description)` — give it a clear, descriptive name
4. For each projection to add:
   - If simple (salary, rent, subscription): use `addRecurringExpense` or `addRecurringIncome`
     - `listDimensionOptions(ACCOUNT)` → get `accountApiId`
     - Optionally `listDimensionOptions(CUSTOMER | VENDOR)` → get counterparty id
     - Call `addRecurringExpense` or `addRecurringIncome`
   - If complex (custom dimensions, unusual recurrence): use `createProjection`
     - See the createProjection workflow below
5. `getCashFlowForecast` or `getCashPosition` on the new scenario to show impact
6. Optionally `compareScenariosForPeriod` to show new scenario vs base

---

## `createProjection` — Full Flow

Use when `addRecurringExpense/Income` isn't expressive enough.

1. `listDimensionOptions(ACCOUNT, search?)` → get the `value` field for the account (use as `account` in the projection)
2. `listDimensionOptions(CUSTOMER | VENDOR, search?)` → get the `value` for the counterparty (required for income/expense respectively)
3. Optionally `listDimensionOptions(CLASS | DEPARTMENT | LOCATION | PROJECT)` for additional classification
4. Build the projection object:
   ```
   {
     account: <accountApiId from step 1>,
     date: <start date>,
     recurrenceType: NONE | DAILY | WEEKLY | BIWEEKLY | SEMIMONTHLY | MONTHLY | QUARTERLY | SEMIANNUAL | ANNUAL,
     amount: <positive for income, negative for expense>,
     dimensions: [
       { dimensionType: "CUSTOMER", value: <customerId> }   // for income
       // OR
       { dimensionType: "VENDOR", value: <vendorId> }       // for expense
     ],
     endDate: <optional, for recurring with end>
   }
   ```
5. `createProjection(projection)`

---

## `bulkCreateProjections` — 6-Step Workflow

Use when creating many projections at once (up to 500). Follow these steps in order.

**Step 1 — PARSE**
Parse the user's input into a list of raw projection intents. Each should have: description, amount, frequency, start date, account name (human-readable), counterparty name (human-readable).

**Step 2 — RESOLVE ACCOUNT**
Call `listDimensionOptions(ACCOUNT)` once. Map each projection's account name to the `value` field from the results.

**Step 3 — RESOLVE COUNTERPARTIES**
For each unique counterparty name, call `listDimensionOptions(CUSTOMER | VENDOR, search: <name>)`. Call once per unique name — not per projection. Map names to `value` fields.

**Step 4 — MAP RECURRENCE**
Convert human frequency descriptions to API values:
- "monthly" → MONTHLY, "weekly" → WEEKLY, "bi-weekly" → BIWEEKLY, "once" / "one-time" → NONE
- "twice a month" / "semi-monthly" → SEMIMONTHLY, "quarterly" → QUARTERLY, "yearly" / "annual" → ANNUAL

**Step 5 — CONFIRM**
Present a summary to the user:
- Total projections to create
- List with: description, amount, frequency, account, counterparty, start date
- Ask the user to confirm before proceeding

**Step 6 — CALL**
`bulkCreateProjections(companyId, scenarioId, projections)` with the resolved list. Report: how many created, any errors from `errors[]`.

---

## Custom Report Export — Two-Stage Flow

Use when `exportCustomReport` returns a size error or when the data needs AI transformation before export.

1. `listCustomReports()` → get `reportId` and the report's `processingPrompt`
2. `prepareCustomReportExport(reportId, startDate, endDate)` → returns:
   - `data`: raw cash flow data
   - `processingPrompt`: instructions for how to transform the data
   - `exportToken`: must be used within 15 minutes
3. Transform `data` according to `processingPrompt` — build:
   - `headers`: list of column name strings
   - `rows`: list of lists (use JSON numbers, not strings, for numeric cells)
4. `finalizeCustomReportExport(exportToken, headers, rows)` → returns 15-minute presigned download link
5. Present the download link to the user

---

## Building a New Custom Report

1. `getReportBuilderStep(step: 1)` → follow the instructions to gather requirements
   - If user has an Excel template, capture the `pendingTemplateAttachmentId` from the upload
2. `getReportBuilderStep(step: 2)` → follow the instructions to draft the report definition
3. `saveCustomReport(name, description, processingPrompt, groupBy, scenarioMode, outputFormat, ...)` → save the definition
4. If `pendingTemplateAttachmentId` was present: immediately call `uploadReportTemplate(reportId, attachmentId)`
5. Confirm the report is saved; offer to export it now

---

## Cash Conservation Strategy (Bulk AP Reschedule)

Use when a user wants to delay payments to protect cash.

1. `getAPSummary` → get total AP picture
2. `listBills(overdueOnly: false)` with a date range → identify upcoming bills
3. `getCashPosition` or `detectNegativeBalancePeriods` → identify the at-risk window
4. Present the user with bills that could be deferred and ask for their strategy (delay all, delay by vendor, delay by amount threshold)
5. Build the update list: `[{billId, expectedPaymentDate}]`
6. `bulkSetBillExpectedPaymentDate(companyId, scenarioId, updates)` → apply the changes
7. `getCashPosition` or `detectNegativeBalancePeriods` again → show the improvement

---

## Scenario Comparison

1. `listScenarios` → list all available scenarios, identify the ones to compare
2. `compareScenariosForPeriod(companyId, scenarioIds: [2–5], startDate, endDate, grouping?)` → get side-by-side data
3. Present: for each scenario — name, opening balance, total inflows, total outflows, closing balance, per-period breakdown
4. Highlight key differences: which scenario has better closing balance, lower negative exposure, higher inflow

---

## Onboarding Check (First Session)

Two distinct failure modes — handle them differently.

### MCP connection fails entirely (brand new user, no CFF account)

No tools are reachable. Claude cannot call any tools.

Tell the user: "It looks like you don't have a Cash Flow Frog account connected yet. Sign up at accounts.cashflowfrog.com/signup to connect your Xero or QuickBooks account, then come back."

### MCP connects but companies are missing or integration is broken

1. `listCompanies`:
   - Returns empty → call `getOnboardingUrl()` → present the returned setup URL.
   - Returns companies → continue to step 2.
2. `getIntegrationStatus(companyId)`:
   - CONNECTED but stale → offer to call `syncCompanyData(companyId)`.
   - DISCONNECTED or ERROR → call `getOnboardingUrl()` → present the returned reconnect URL. Report the error message from `getIntegrationStatus` to help the user understand what went wrong.

Note: `getOnboardingUrl` takes no parameters — the server determines the correct URL (setup or reconnect) from the authenticated user's context. Response fields: `url` (present as a link), `action` ("setup" | "reconnect" | "connected"), `message` (show alongside the link).
