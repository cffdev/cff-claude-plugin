---
description: Get the current cash position with near-term projections, runway, and any at-risk periods where the balance could go negative.
---

Get the cash position and risk overview:

1. Call `listCompanies` to get the companyId.
2. Call `listScenarios` and use the scenario where `active: true`.
3. Call `getCashPosition` (default horizonDays: 90).
4. Call `detectNegativeBalancePeriods` to identify any periods where the balance drops below zero.
5. Call `listBankAccounts` to show the account-level breakdown.
6. Present:
   - Current total bank balance (with per-account breakdown)
   - Projected balances at 30, 60, 90 days
   - Lowest projected balance and when it occurs
   - Cash runway (days until balance goes negative, if applicable)
   - Any negative balance periods: date range, minimum balance
7. If there are at-risk periods, offer to investigate: "Would you like to look at what's driving the low balance — overdue invoices, upcoming bills, or specific projections?"
