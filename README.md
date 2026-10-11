# Cash Flow Frog for Claude

Connect Claude to [Cash Flow Frog](https://cashflowfrog.com) for intelligent cash flow analysis, AR/AP management, scenario planning, and financial reporting — grounded in live data from your connected accounting platform.

## Prerequisites

- An active Cash Flow Frog account ([sign up at accounts.cashflowfrog.com/signup](https://accounts.cashflowfrog.com/signup))
- At least one company configured with a connected accounting integration (Xero or QuickBooks Online)

## What You Can Do

**Visibility & Forecasting**
- Get your current cash position with 30/60/90-day projections and runway
- Run cash flow forecasts over any date range with daily, weekly, monthly, or quarterly groupings
- Detect periods where your projected balance goes negative before they happen
- View all bank account balances

**AR & AP Management**
- Review outstanding invoices, bills, estimates, sales orders, and purchase orders
- Set or reschedule expected collection and payment dates to fine-tune your forecast
- Split large transactions across multiple installment dates
- Bulk-reschedule payments for cash conservation strategies

**Scenario Planning**
- Create "what if" scenarios (new hire, new lease, expansion plan)
- Add one-off or recurring income and expense projections
- Compare up to 5 scenarios side by side over any period

**Analysis & Reporting**
- Break down cash flow by category, customer/vendor, project, department, or class
- Compare actuals vs. planned amounts by category (budget vs. actual)
- Generate PDF cash flow reports with charts and bar graphs
- Export custom AI-built reports as Excel or CSV files

## Getting Started

Once installed, ask Claude anything about your cash flow:

> "What's my current cash position and runway?"
> "Show me my cash flow forecast for the next quarter"
> "Which invoices are overdue and by how much?"
> "Create a scenario where I add two salespeople starting next month"
> "Compare my base scenario vs the expansion plan for Q1"
> "Generate a PDF cash flow report with category breakdown"

Claude handles multi-step workflows automatically and will ask for clarification when needed.

## Notes

- **Data freshness:** Data is synced from your accounting software (Xero or QuickBooks Online). If your forecast looks stale, ask Claude to check your integration status or trigger a sync.
- **Account categories:** Cash flow categories ("Account Types") control how accounts are grouped in reports. Ask Claude to review or update your account mappings if categories look incorrect.
- **Scenarios:** Every forecast runs against a scenario. Your main scenario is the one marked as active. New scenarios let you model changes without affecting your base forecast.

## Keeping This Plugin Up to Date

The tool reference at `skills/cashflowfrog/references/tools.md` should be updated whenever new tools are added to the MCP server. The skill strategy in `skills/cashflowfrog/SKILL.md` should be updated when new workflow patterns emerge or user journeys change.
