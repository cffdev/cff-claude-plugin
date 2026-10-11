---
description: Get a combined accounts receivable and accounts payable overview — outstanding balances, overdue amounts, and net position.
---

Get the AR/AP overview:

1. Call `listCompanies` and `listScenarios` (use active scenario).
2. Call `getARSummary` and `getAPSummary`.
3. If AR has overdue invoices (overdueTotalBalance > 0): call `listInvoices(overdueOnly: true)` and show the top overdue invoices.
4. If AP has overdue bills (overdueTotalBalance > 0): call `listBills(overdueOnly: true)` and show the top overdue bills.
5. Present:
   - **AR:** total open balance, overdue amount and count, oldest overdue invoice age
   - **AP:** total open balance, overdue amount and count, oldest overdue bill age
   - **Net working capital position:** AR open balance minus AP open balance
6. Offer next steps:
   - "Would you like to reschedule any overdue items to update your forecast?"
   - "Should I also pull in open estimates or sales/purchase orders?"
   - "Would you like to see the full invoice or bill list with expected dates?"
