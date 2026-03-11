---
closing_cost_estimate_percent: 1.2  # Fallback only — the closing-costs skill provides itemized estimates
min_recommendation_score: 6
min_monthly_savings_threshold: 50
max_breakeven_months: 48
default_lock_period: 30
---

# Lendtrain Org-Specific Configuration

This file contains organization-specific configuration values referenced by the `mortgage-loan-officer` skill. These values control recommendation thresholds and pricing defaults.

## Configuration Reference

### `closing_cost_estimate_percent`

Estimated closing costs as a percentage of the loan amount, used for breakeven calculations. When a borrower receives a refinance quote, this percentage is applied to the new loan amount to estimate total closing costs. The breakeven period is then calculated as total closing costs divided by monthly savings.

**Current value**: 1.2 (i.e., 1.2% of the loan amount)

### `min_recommendation_score`

Minimum score on a 1-10 scale required to recommend refinancing and offer to submit an application. Quotes that score below this threshold will still be presented to the borrower with a transparent explanation, but the agent will not proactively suggest proceeding with an application.

**Current value**: 6

### `min_monthly_savings_threshold`

Minimum monthly payment savings, in dollars, to consider a refinance worthwhile. If the projected monthly savings fall below this amount, the recommendation score is reduced accordingly and the agent communicates that the savings may not justify the cost and effort of refinancing.

**Current value**: 50

### `max_breakeven_months`

Maximum acceptable breakeven period in months. The breakeven period represents how long it takes for cumulative monthly savings to offset total closing costs. Longer breakeven periods lower the recommendation score. Scenarios exceeding this threshold receive a significantly reduced score and a clear explanation to the borrower.

**Current value**: 48

### `default_lock_period`

Default rate lock period in days when calling the pricer API. This determines how long the quoted rate is guaranteed. Shorter lock periods may yield slightly better rates; longer periods provide more time to close. This value is used as the default unless the borrower specifies a different preference.

**Current value**: 30

