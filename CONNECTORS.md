# Connectors

This document describes the `~~category` connector placeholders used throughout the Lendtrain plugin. Each connector represents an external service that the plugin communicates with via MCP (Model Context Protocol) server connections.

Connector placeholders follow the `~~name` convention. When the plugin references `~~pricer` or `~~los`, it is invoking tools provided by the corresponding MCP server defined in `.mcp.json`.

---

## ~~pricer -- Mortgage Pricing Engine

**Service:** Mortgage-pricer MCP endpoint (`https://mortgage-pricer-production.up.railway.app/mcp`)

**Purpose:** Calculates mortgage rate quotes based on a loan scenario. Returns multiple rate/point combinations with monthly payments, APR, and adjustment breakdowns. Also provides the current rate sheet effective date.

**Connection:** Configured via `.mcp.json` as an HTTP MCP server. No `.env` file or API key needed.

**Tools:**

| Tool | Description |
|------|-------------|
| `calculate_pricing` | Submit a `LoanScenario` and receive a `PricingResult` with pricing array, adjustment breakdown, product details, and calculation timestamp. |
| `get_effective_date` | Retrieve the effective date of the currently loaded rate sheet. Returns the date the active rate sheet was published. |

### calculate_pricing — Request Fields (LoanScenario)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `loanPurpose` | `'rateTermRefi' \| 'cashOutRefi'` | Yes | Refinance type |
| `loanAmount` | number | Yes | Loan amount (> 0) |
| `propertyValue` | number | Yes | Property value (>= loanAmount for rate-term) |
| `creditScore` | number | Yes | Integer 300-850 |
| `state` | `'AL' \| 'FL' \| 'GA' \| 'KY' \| 'NC' \| 'OR' \| 'SC' \| 'TN' \| 'TX' \| 'UT'` | Yes | Licensed state |
| `loanTerm` | `10 \| 15 \| 20 \| 25 \| 30` | Yes | Term in years |
| `lockPeriod` | `15 \| 30 \| 45 \| 60` | Yes | Lock period in days |
| `occupancy` | `'primary' \| 'secondHome' \| 'investment'` | Yes | Occupancy type |
| `propertyType` | `'singleFamily' \| 'condo' \| 'townhouse' \| 'multiUnit' \| 'manufactured'` | Yes | Property type |
| `productType` | `'conventional' \| 'fha' \| 'va'` | No (default: conventional) | Loan program type |
| `vaFundingFeeExempt` | boolean | No | VA only. True if veteran has 10%+ service-connected disability |
| `condoType` | `'attached' \| 'highRise' \| 'detached'` | No | Condo only |
| `employmentType` | `'employed' \| 'selfEmployed' \| 'retired' \| 'other'` | No | Employment status |
| `dti` | number | No | Debt-to-income ratio 0-100 |
| `hasSubordinateFinancing` | boolean | No | Has HELOC or second lien |
| `escrowWaiver` | boolean | No | Waive escrow |
| `isHighBalance` | boolean | No | High-balance conforming |
| `units` | `1 \| 2 \| 3 \| 4` | No | Unit count (multiUnit only) |
| `ausPreference` | `'DU' \| 'LP'` | No | Automated Underwriting System |

### calculate_pricing — Response Fields (PricingResult)

| Field | Type | Description |
|-------|------|-------------|
| `success` | boolean | Whether calculation succeeded |
| `error` | string? | Error message if success is false |
| `scenario` | LoanScenario & { ltv } | Echo of submitted scenario with computed LTV |
| `product` | object | Selected product: `code`, `name`, `term`, `productType` |
| `adjustments` | object | Adjustment breakdown with `creditScoreLtv`, `stateSrp`, `stateEscrow`, `features`, `incentives`, `brokerComp`, `compensationCapped`, `total`, `itemized[]` |
| `financedFees` | FinancedFees? | **FHA/VA only.** Financed fee breakdown (see below) |
| `pricing` | RatePricing[] | Array of rate options (see below) |
| `effectiveDate` | string | Rate sheet date |
| `effectiveTime` | string | Rate sheet time |
| `calculatedAt` | string | Calculation timestamp |

### FinancedFees Object

Present when `productType` is `'fha'` or `'va'`:

| Field | Type | Present When | Description |
|-------|------|-------------|-------------|
| `ufmip` | number | FHA only | UFMIP amount (1.75% of base loan) |
| `vaFundingFee` | number | VA non-exempt only | Funding fee dollar amount |
| `fundingFeePercent` | number | VA non-exempt only | Fee percentage applied (0.5% IRRRL, 3.3% cash-out) |
| `totalFinanced` | number | Always | Total dollar amount financed (added to base loan) |
| `totalLoanAmount` | number | Always | Base loan amount + totalFinanced |
| `annualMip` | number | FHA only | Annual MIP amount (0.55% of base loan) |
| `monthlyMip` | number | FHA only | Monthly MIP payment (annualMip / 12) |

### RatePricing Fields

| Field | Type | Description |
|-------|------|-------------|
| `rate` | number | Note rate (e.g., 6.375) |
| `clientPrice` | number | Client price (par = 100.00; above 100 = lender credit) |
| `points` | number | Points cost (negative = lender credit) |
| `dollarAmount` | number | Dollar cost (negative = credit to borrower) |
| `monthlyPayment` | number | P&I payment. For FHA/VA, based on `totalLoanAmount` |
| `apr` | number | Annual Percentage Rate |
| `isPar` | boolean? | Whether this is the par rate option |
| `monthlyMip` | number? | **FHA only.** Monthly MIP amount |
| `totalMonthlyPayment` | number? | **FHA only.** `monthlyPayment` + `monthlyMip` |

**IMPORTANT**: `basePrice` is intentionally excluded. Never display or reference it.

### LTV Limits by Product Type

| Product | Loan Purpose | Max LTV |
|---------|-------------|---------|
| Conventional | All | 97% |
| FHA | Rate/Term (Streamline) | 96.5% |
| FHA | Cash-Out | 80% |
| VA | IRRRL (rateTermRefi) | 100% |
| VA | Cash-Out | 80% |

### Adjustment Pipeline Differences

> For FHA/VA loans, the pricer uses a completely different adjustment pipeline:
> - `adjustments.creditScoreLtv` reflects a government FICO adjuster (not the conventional LLPA matrix)
> - `adjustments.features` is always 0 (feature adjustments don't apply to government loans)
> - Individual adjustment items have `category: 'government'` instead of `'creditLtv'` or `'feature'`
> - Shared adjustments (state SRP, escrow waiver, incentives, broker comp) still apply normally
> - Maximum client price cap: 103.750 for fixed-rate government loans

---

## ~~los -- Arive Loan Origination System

**Service:** Arive API-Connect v1.0

**Purpose:** Creates and manages mortgage leads and loan applications in the Arive LOS. Handles the transition from initial lead capture through full 1003 application submission.

**Tools:**

| Tool | Description |
|------|-------------|
| `auth_login` | Authenticate with Arive using API credentials. Returns a Bearer token valid for 3600 seconds. |
| `create_lead` | Create a new lead in Arive with borrower information, loan details, and property data. Returns `ariveLeadId` and `deepLinkURL`. |
| `update_lead` | Update an existing lead with additional borrower data, employment details, or declarations (partial update via PATCH). |
| `convert_lead_to_loan` | Convert a qualified lead into a full loan application. Assigns the loan officer and returns `ariveLoanId`. |

**Configuration:**
- Base URL set via `ARIVE_API_URL` in `.env`
- Authentication credentials: `ARIVE_CLIENT_ID`, `ARIVE_SECRET_KEY`, `ARIVE_API_KEY`, `ARIVE_APP_ID`, `ARIVE_APP_SECRET_HASH`
- Lead assignment: `ARIVE_ASSIGNEE_EMAIL`
