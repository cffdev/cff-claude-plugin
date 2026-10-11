---
description: Run a cash flow forecast for the active scenario. Prompts for date range and grouping, then presents a period-by-period breakdown with key highlights.
---

Run a cash flow forecast:

1. Call `listCompanies` to get the companyId.
2. Call `listScenarios` and use the scenario where `active: true`, unless the user has specified a different one.
3. If the user hasn't specified a date range, ask: "What period would you like to forecast? (e.g., next 90 days, next quarter, full year)"
4. Ask for grouping preference if not specified — default to MONTHLY. Suggest WEEKLY for near-term cash concerns, DAILY for acute situations.
5. Call `getCashFlowForecast` with the resolved parameters.
6. Present:
   - Opening balance, total inflows, total outflows, closing balance for the full period
   - Per-period highlights: any periods where balance is notably low or drops
7. Call `detectNegativeBalancePeriods` if the closing balance suggests risk.
8. Offer next steps: "Would you like a breakdown by category, a look at specific invoices or bills affecting this forecast, or a PDF report?"
