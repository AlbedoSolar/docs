# Quote Generation: Known Inconsistencies & Items to Investigate

Issues discovered during documentation of the quote generation system. Organized by severity: logical inconsistencies that affect output correctness, silent divergences between the two calculation engines, and structural gaps in the data model.

---

## Table of Contents

1. [Logical Inconsistencies (Affect Output)](#1-logical-inconsistencies)
2. [Silent Divergences Between Engines](#2-silent-divergences-between-engines)
3. [Structural Gaps](#3-structural-gaps)
4. [Python Lambda vs TypeScript Edge Function Divergences](#4-python-lambda-vs-typescript-edge-function-divergences)
5. [R-to-TypeScript Port Status](#5-r-to-typescript-port-status)

---

## 1. Logical Inconsistencies

These produce incorrect or misleading results in production.

---

### 1.1 Maintenance Rate Provider Selection for Multi-Provider Projects

**Status:** Resolved

The frontend (QuoteWizardStep4Quote.tsx:93-123) and backend (`loadQuoteIntegrationContext()` in quote-helpers.ts, consumed by quote-generator/index.ts) both now select the provider from the phase with the highest cost (in project currency). This is typically the primary installer who handles the bulk of the work and maintenance.

- `loadQuoteIntegrationContext()` returns a `primary_provider_id` field derived from the highest-cost phase
- `quote-generator` uses `context.primary_provider_id` for maintenance rate and provider pricing lookups
- `quote-rates` receives the provider ID from the frontend request, which also uses the highest-cost phase logic
- The frontend shows a yellow warning banner when multiple providers are detected

**Remaining limitation:** A single provider's maintenance rate is used for the entire project. If different phases have meaningfully different maintenance characteristics, this is still an approximation. See [3.1 No Per-Phase Insurance or Maintenance](#31-no-per-phase-insurance-or-maintenance).

---

### 1.2 Provider Pricing (Recommended APR) Uses Highest-Cost Provider

**Status:** Resolved (same fix as 1.1)

The recommended APR depends on provider-specific fields (`default_premium_with_down_payment`, `default_premium_without_down_payment`, `time_premium_per_year`, `max_time_premium`). Both frontend and backend now use the highest-cost phase's provider for this lookup, consistent with maintenance rate selection.

**Remaining limitation:** If providers have different risk profiles, the recommended APR only reflects one of them. Low impact because the user can override the annual rate manually.

---

### 1.3 Commission Added to Aggregate Retail Regardless of Per-Phase Margin Type

**Location:** `quote-helpers.ts:477`

```ts
const finalRetailWithCommission = discounted.retail_after_discount_pre_commission + commissionTotal;
```

**Problem:** Each phase's base retail is correctly calculated per its own margin type ("Add" reverses the margin, "Subtract" passes through). But after aggregating all phases, commission is unconditionally added on top of the full total. For "Subtract" type providers, the phase cost already includes commission — adding it again double-counts.

**Example:** Phase A is "Add" with $10,000 cost → base retail $11,765. Phase B is "Subtract" with $5,000 cost → base retail $5,000 (commission embedded). Aggregate = $16,765. Commission at 5% = $838 added on top. But $838 includes 5% of the $5,000 "Subtract" phase, which already has commission baked in.

**Impact:** Retail price is inflated for projects mixing "Add" and "Subtract" providers. The client overpays and the financial metrics (margin, markup) are skewed.

**Scope:** Only affects projects with both "Add" and "Subtract" providers. If all phases use the same margin type, this is not an issue.

**Possible fix:** Split the commission calculation: only compute commission on the sum of "Add" type phase retail prices, not on "Subtract" type phases.

---

### 1.4 Legal Fee Preview Differs from Actual Legal Fee

**Location:**
- Preview: `quote-rates/index.ts:56-61` — uses `asset_cost` as `retailPrice`
- Actual: `quote-generator/index.ts:160-165` — uses `projectRetail.final_retail_with_commission`

**Problem:** The `quote-rates` endpoint calculates the legal fee using the raw `asset_cost` input as the price for bracket determination. The `quote-generator` uses the fully calculated retail price (after margin reversal + commission), which is typically 15-25% higher. Since legal fees are bracket-based with hard thresholds ($8K/$13K for USD, Q60K/Q100K for GTQ), a project near a threshold could land in a different bracket.

**Example (USD):** Asset cost = $7,800 → preview shows $150 (bracket: ≤$8,000). Retail with 15% margin + 5% commission = ~$9,500 → actual fee = $200 (bracket: $8,001-$12,999). User sees $150 in the form, calculation uses $200.

**Impact:** The legal fee shown in the Step 4 form may not match what's actually used in the calculation. Low financial impact (max $150 difference for USD, Q1,175 for GTQ) but creates confusion.

**Scope:** Only affects projects near bracket thresholds.

**Possible fix:** Have the `quote-rates` endpoint calculate the full retail price before determining the legal fee, or accept that this is a preview and document it.

---

### 1.5 Off-by-One in Service Payment Spreading Denominator

**Location:** `quote-integration-flow.ts:153`

```ts
const denominator = Math.max(term - gracePeriod + 1, 1);
```

**Problem:** Payments are applied from `serviciosStartMonth` (default: `gracePeriod + 1`) through `term` inclusive — that's `term - gracePeriod` months of actual payments. But the denominator dividing the total cost is `term - gracePeriod + 1`, which is one more. The total cost is spread thinner than the number of collection months.

**Example:** term = 63, gracePeriod = 3.
- Denominator = 63 - 3 + 1 = 61
- Payment months = month 4 through 63 = 60 months
- Total collected = (totalCost / 61) * 60 = totalCost * 60/61 ≈ 98.4% of cost

**Impact:** ~1.6% systematic under-collection on both insurance and maintenance costs. Over a full portfolio, this adds up.

**Mitigation:** The code comment says this is intentional — "Match R code: use gracePeriod in denominator (not serviciosStartMonth) to allow for discounts." But it's unclear whether this was a deliberate pricing decision or a legacy bug that was preserved for consistency.

**Action needed:** Confirm with the business whether this ~1.6% gap is an intended client discount or an error inherited from the R codebase.

---

## 2. Silent Divergences Between Engines

The quote-generator has two calculation paths: the **integration flow** (used by the QuoteWizard via estimate reference) and the **NII table** (used in manual/admin mode). These should produce equivalent results for equivalent inputs but diverge in several ways.

---

### 2.1 Insurance Premium Applied in Integration Flow but Not NII Table

**Location:**
- Integration flow: `quote-integration-flow.ts:148` — `totalInsuranceCost = SUM(insuranceCost) * (1 + insurancePremium)`
- NII table: `quote-calculator.ts:273` — `totalInsuranceCost = SUM(insuranceCost)` (no premium)

**Problem:** The integration flow applies an insurance premium markup (default 1.8%) to the total insurance cost before spreading into payments. The NII table does not. For the same underlying insurance costs, the integration flow charges the client more.

**Impact:** If both engines are ever compared (e.g., validating manual mode against estimate mode), they will disagree on insurance payment amounts.

---

### 2.2 Down Payment and Purchase Option Base Differs

**Location:**
- Integration flow: `quote-generator/index.ts:167-175`
  ```ts
  downPaymentAmount = finalRetailWithCommission * down_payment_1_percentage
  opDeCompSinIva = finalRetailWithCommission * opDeCompPct
  ```
- NII table: `quote-calculator.ts:204-207`
  ```ts
  downPaymentAmount = retailPrice * down_payment_percentage
  purchaseOption = retailPrice * purchase_option_percentage
  ```

**Problem:** In the integration flow, down payment and purchase option are percentages of the full retail price including commission. In the NII table, `retailPrice` is derived from `asset_cost + margin` but commission is not added to the `retailPrice` variable — it's handled separately in the schedule. So the base for these percentage calculations differs.

**Impact:** For the same deal parameters, down payment and purchase option amounts will be different between the two engines.

---

### 2.3 Maintenance Inflation Baseline Differs

**Location:**
- NII table: `quote-calculator.ts:256-257`
  ```ts
  inflationYears = MAX(floor(month / 12) - maintenanceInflationStartYear, 0)
  ```
- Integration flow: `quote-integration-flow.ts:133`
  ```ts
  inflationYears = MAX(floor((month - maintStartMonth) / 12), 0)
  ```

**Problem:** The NII table counts inflation years from the start of the loan (relative to `maintenanceInflationStartYear`, default year 3). The integration flow counts inflation years from the first maintenance visit (`maintStartMonth`, default month 24).

**Example at month 48:**
- NII table: `floor(48/12) - 3 = 1` year of inflation
- Integration flow: `floor((48-24)/12) = 2` years of inflation

**Impact:** Maintenance costs inflate at different rates between the two engines. The integration flow inflates faster.

---

### 2.4 Mismatched Internal Defaults in serviciosTable()

**Location:** `quote-integration-flow.ts:106-114`

| Parameter | serviciosTable() default | Config/system default | Passed by quote-generator |
|-----------|-------------------------|----------------------|--------------------------|
| `insuranceRate` | 0.012093 | 0.017 | Location-resolved rate |
| `maintRate` | 0.03 | 0.028 | Location-resolved rate |
| `maintPremium` | 0.5 (50%) | 0.1 (10%) | 0.1 from event data |
| `insurancePremium` | 0 (0%) | 0.018 (1.8%) | 0.018 from event data |
| `maintStartMonth` | 36 | 24 | 24 from event data |

**Problem:** The quote-generator always passes explicit values, so these internal defaults are never reached in production. But they're wrong compared to the system config defaults. If anyone calls `serviciosTable()` directly (new endpoint, test, refactor), they'll silently get different behavior.

**Impact:** No production impact currently. Risk of subtle bugs if the code is reused.

**Possible fix:** Remove internal defaults and make all parameters required, or align them with the config defaults.

---

## 3. Structural Gaps

Design-level issues where the data model doesn't fully support the business reality.

---

### 3.1 No Per-Phase Insurance or Maintenance

**Problem:** Insurance and maintenance are both calculated on a single `installationTotal` with a single rate. But phases can involve:
- Different providers (who may have different maintenance rates)
- Different equipment types (which may have different insurance risk profiles)
- Different physical locations (if a project spans sites)

The system flattens everything into one aggregate number.

**Impact:** Inaccurate cost modeling for complex multi-phase projects. Simple single-provider projects are unaffected.

**Possible fix:** Calculate insurance and maintenance per-phase, using each phase's provider and location, then sum the results.

---

### 3.2 Insurance Protects at Installation Cost, Not Financed Value

**Location:** `serviciosTable()` and `generateNiiTable()` — both use `installationCostSinIva` (or `assetCost`) as the insurance base.

**Problem:** Insurance cost is calculated as `installationCost * insuranceRate`. But the client is financing the retail price (which includes the provider margin and commission markup — typically 15-25% higher). If the insured asset is damaged, the replacement cost might be the installation cost, but Albedo's financial exposure is the remaining balance of the retail price.

**Impact:** Potential under-insurance relative to Albedo's financial risk. Whether this matters depends on what the insurance is actually covering — if it's physical asset replacement, installation cost is the right base. If it's protecting Albedo's receivable, the retail price might be more appropriate.

**Action needed:** Clarify with the business what the insurance is intended to cover and whether the base amount is correct.

---

### 3.3 Exchange Rate Timing Risk

**Problem:** Currency conversion uses the most recent exchange rate on or before the `referenceDate` (defaults to today). But:
- The installation may not start for weeks or months
- Provider payments are spread across 3 periods (50%/40%/10% by default)
- Client payments span 5+ years

All of these are locked to a single exchange rate snapshot. Currency fluctuations between quote generation and actual payment flows create unhedged exposure.

**Impact:** For projects where the partner contract currency differs from the project currency, exchange rate movement between quote time and payment time is unaccounted for. The bigger the time gap and the more volatile the currency pair, the larger the risk.

**Action needed:** Determine whether this is an accepted business risk or whether some form of exchange rate sensitivity analysis or periodic re-quoting should be built in.

---

### 3.4 Commission Type Intersection May Produce No Results

**Location:** Frontend commission type selection (QuoteWizardStep3Estimate.tsx)

**Problem:** The frontend selects a commission type by intersecting the commission types shared by all selected vendors and affiliates, then filtering for exact role-column matches. If the intersection is empty (no commission type satisfies all selected roles), the user gets no options and can't proceed.

**Impact:** Certain vendor/affiliate combinations may be impossible to quote even if they're valid business arrangements, simply because no commission type record covers all the roles simultaneously.

**Possible fix:** Allow per-role commission types instead of requiring a single type that covers all roles. Or surface a clearer error message explaining which role combination has no matching commission structure.

---

### 3.5 No Validation That Provider Payout Percentages Sum to 1.0

**Location:** `calculateInstallationSchedule()` (quote-helpers.ts:902-904)

```ts
const p1 = schedule?.period_1_payout ?? fallbackSchedule[0];
const p2 = schedule?.period_2_payout ?? fallbackSchedule[1];
const p3 = schedule?.period_3_payout ?? fallbackSchedule[2];
```

**Problem:** There's no check that `p1 + p2 + p3 == 1.0`. If a provider payment schedule has payouts summing to 0.8 or 1.2, the installation total will be 80% or 120% of the actual phase cost. This would silently distort the installation schedule and every downstream calculation that depends on it (insurance base, maintenance base, debt tracking, sale markup).

**Impact:** Data entry error in `provider_payment_schedules` could cause significant financial miscalculation without any error or warning.

**Possible fix:** Add a validation check that payout percentages sum to approximately 1.0 (within a small tolerance), and throw or warn if they don't.

---

### 3.6 The /1.12 IVA Removal in Per-Phase Commission Is Hardcoded

**Location:** `calculateRetailPriceComponents()` (quote-helpers.ts:397)

```ts
const commissionAmount = officialVendorCommissionPercent * retailAfterDiscount / 1.12;
```

**Problem:** The `1.12` is the Guatemala IVA rate, hardcoded. This function doesn't receive `taxRate` as a parameter. If this code path were ever used for El Salvador (13%) or Honduras (15%) projects, the IVA removal would be wrong.

**Impact:** Currently no production impact because `calculateProjectRetailWithCommission()` calls this function with `officialVendorCommissionPercent: 0`, bypassing the commission calculation entirely. But the hardcoded value is a latent bug if the per-phase commission path is ever activated.

**Possible fix:** Pass `taxRate` as a parameter, or remove this dead code path if it's not intended to be used.

---

## 4. Python Lambda vs TypeScript Edge Function Divergences

The quote generation system was originally implemented as a Python Lambda (in `albedo-automations-infra/packages/lambdas/quote-generator/`), then re-implemented as a TypeScript Supabase Edge Function (in `supabase/supabase/functions/quote-generator/`). Both are documented in the codebase. The following divergences were identified during cross-referencing. These matter if the Python Lambda is still active, or if its documentation is used as a reference for the TypeScript implementation.

---

### 4.1 Commission Distribution Timing

**Python Lambda:** Three configurable tranches using `commission_expense_month_1`, `commission_expense_month_2`, `commission_expense_month_3`. Defaults: 80% at month 0, 0% at month 1, 20% at month 2.

**TypeScript Edge Function:** Two tranches using `commission_first_payout_share` and `commission_final_payout_share`. First tranche (80%) at month 1, final tranche (20%) at `MIN(MAX(gracePeriod, 1), loanTerm)` (month 3 by default).

**Differences:**
- Python starts commission at month 0 (project start); TypeScript at month 1
- Python has 3 configurable slots (with the middle one defaulting to 0%); TypeScript has 2
- Python's final tranche is at month 2; TypeScript's final tranche is at the end of the grace period (month 3)

**Impact:** Commission cost timing affects the debt balance (since debt accrues from month 0). Earlier commission payout = higher debt cost in the early months. The Python flow front-loads commission more aggressively.

---

### 4.2 Retail Price Formula May Differ

**Python docs** (calculate_retail_price.md) describe the retail price as "margin-adjusted" but the underlying formula may use `cost * (1 + markup)` (additive markup). **TypeScript** uses margin reversal: `(cost / (margin - 1)) * -1`.

**Example with 15% margin/markup, $10,000 cost:**
- Additive markup: $10,000 * 1.15 = $11,500
- Margin reversal: $10,000 / 0.85 = $11,764.71

**Status:** Needs verification against actual Python source code. The Python documentation may be using simplified language. If both systems use margin reversal, this is a documentation issue only, not a code issue.

**Impact:** If the formulas truly differ, every downstream calculation (commission, down payment, purchase option, legal fee, amortization) would produce different results between the two systems.

---

### 4.3 Capital Cost Distribution Timing

**Python Lambda:** Three configurable periods using `asset_expense_month_1`, `asset_expense_month_2`, `asset_expense_month_3`. Defaults: 50% at month 0, 40% at month 1, 10% at month 2.

**TypeScript Edge Function (Estimate Reference Mode):** Uses `provider_payment_schedules` table with `period_1_payout`, `period_2_payout`, `period_3_payout`. Same defaults (50%/40%/10%) but pulled from actual provider payment schedule records rather than global config.

**TypeScript Edge Function (Manual Mode):** Uses `asset_expense_month_1/2/3` from config — identical to the Python Lambda approach.

**Impact:** In estimate reference mode, the TypeScript version uses provider-specific payout schedules, which is more granular than the Python Lambda's global config approach. Results will differ for providers with non-default payment schedules.

---

### 4.4 NII Table Column Structure

**Python Lambda:** Produces a single NII table with four scenarios: "Leasing", "Leasing w Debt", "w Servicios", "Total w IVA". Each scenario has its own income/expense/NII columns.

**TypeScript Integration Flow:** Produces three separate tables (lease, services, combined) and three KPI scenarios: "Leasing", "Servicios", "Total". The frontend maps these to the four-scenario display by duplicating Leasing for "Leasing w Debt".

**TypeScript NII Table (Manual Mode):** Matches the Python Lambda's four-scenario structure.

**Impact:** The integration flow lacks a true "Leasing w Debt" scenario — it duplicates the base Leasing KPIs. The NII table has this scenario (where expense includes debt cost but not services costs). This means the integration flow's "Leasing w Debt" column is identical to "Leasing", which may mislead users comparing the two columns.

---

### 4.5 IVA Handling: Income Tax vs Cost Tax

**Python Lambda docs** describe two distinct tax-related fields:
- `tax` (income side): `income_with_services * taxRate` — IVA on client payments
- `cost_tax` (expense side): `(SUM(income_with_services) - (SUM(capital_costs) - SUM(commission_costs))) * taxRate / (1 + taxRate) / months` — a separate, unusual formula that spreads a total across payment months

**TypeScript Edge Function:** The integration flow computes `iva_payment = total_payment_sin_iva * taxRate` and `iva_cost = iva_payment` — income-side IVA equals cost-side IVA (they're the same number). There is no separate `cost_tax` formula.

**TypeScript NII Table (Manual Mode):** Does implement `cost_tax` matching the Python formula: `(SUM(income_with_services) - (SUM(capital_costs) - SUM(commission_costs))) * taxRate / (1 + taxRate) / months`.

**Impact:** The integration flow and the NII table handle tax expense differently. The NII table's `cost_tax` formula produces a different (usually lower) per-month expense than the integration flow's `iva_cost = iva_payment` approach. This is an additional silent divergence between the two TypeScript engines.

---

### 4.6 Existing Documentation Coverage Outside Quote Generation

Several scattered docs cover topics that are adjacent to, but not part of, the quote generation calculation itself:

| Document | Topic | In centralized docs? |
|----------|-------|---------------------|
| `DATA_MAPPING.md` | Quickbase → Supabase field mappings, QB field IDs, IVA removal at ingestion | No — data pipeline, not quote generation |
| `DATABASE_DOCUMENTATION.md` | Full Supabase schema, column types, constraints | No — schema reference, not quote generation |
| `calculate_solar_tax_rate.md` | 4-level priority chain for tax rate resolution | Partially — section 5 covers the TypeScript equivalent |
| `calculate_asset_transfer_value.md` | `retail_price * asset_transfer_percentage` | Mentioned in section 20 (post-quote interest calc) |
| `calculate_retail_price.md` | Python retail price function with override path | Yes — section 2 covers the TypeScript equivalent |
| `calculate_commission_cost.md` | Python commission formula with `/1.12` | Yes — section 4 and inconsistency 3.6 |
| `calculate_legal_costs.md` | Legal fee tiers and override path | Yes — section 3 |
| `CALCULATION_BREAKDOWN.md` | Post-quote interest/amortization calc, Legacy IRR vs XIRR | Yes — section 20 |
| `QUOTE_GENERATOR_EXPLANATION.md` | Python Lambda architecture, run tracking, admin calculator | Partially — architecture is covered; `quote_generator_runs` table and admin UI paths are not |
| `BACKEND_DETAILED_EXPLANATION.md` | Full worked example through Python Lambda | No worked example in centralized docs (formulas are there, not a step-by-step trace) |
| `quote_logic_for_integration.md` | Python integration flow with `QuoteFlowInputs` dataclass | Yes — TypeScript equivalent is fully documented |

**Recommendations before deleting scattered docs:**
1. **Safe to delete** (fully covered): `calculate_retail_price.md`, `calculate_commission_cost.md`, `calculate_legal_costs.md`, `calculate_asset_transfer_value.md`, `calculate_solar_tax_rate.md`, `CALCULATION_BREAKDOWN.md`, `quote_logic_for_integration.md`
2. **Keep or relocate** (outside scope of quote generation docs): `DATA_MAPPING.md`, `DATABASE_DOCUMENTATION.md`
3. **Extract useful content first**: `BACKEND_DETAILED_EXPLANATION.md` has a worked step-by-step example that would be valuable to add to the centralized docs; `QUOTE_GENERATOR_EXPLANATION.md` documents the `quote_generator_runs` tracking table and admin calculator behavior

---

### 4.7 Notable Data Model Details from Scattered Docs

Items discovered in the scattered documentation that are not bugs or divergences but are worth noting:

- **`exchange_rates.usd_rate` and `gtq_rate` are stored as TEXT** (not NUMERIC) in the database schema. The TypeScript code parses them with `parseFloat()`, which works but could fail silently on malformed data.
- **`maintenance_payment_project_currency` is always null** in `monthly_cash_flows` — the field exists but is never populated by the data pipeline.
- **Payback period uses sentinel value `9999`** for months where cumulative NII hasn't yet broken even. Months 0-2 are automatically set to 9999 regardless of actual NII.
- **`countries.grid_factor`** exists in the schema (NUMERIC(6,3)) and is used for solar production calculations, not quote generation.
- **`municipalities` has granular poverty flags** (`in_poorest_20/25/33/50_percent` as BOOLEAN) — relevant for impact tracking but not financial calculations.
- **12 Quickbase field mappings are still marked as TODO** in DATA_MAPPING.md, including `legal_name`, `tax_number`, `commission_type_id`, and several currency/energy fields.

---

## 5. R-to-TypeScript Port Status

Line-by-line comparison of the R source (`quoting_tool/quoting_tool/AlbedoQuoteR/R/new_amort_funs.R` and related files) against the TypeScript Edge Functions. The R "new pipeline" is the canonical reference.

---

### 5.1 Integration Flow — Matches R New Pipeline

Every formula in the TypeScript integration flow (`quote-integration-flow.ts`) is a faithful port of the R new pipeline (`new_amort_funs.R`):

| R function | TypeScript function | Status |
|---|---|---|
| `calculate_payment_fun` | `calculatePayment` | Exact match |
| `calculate_lease_fun` | `calculateLease` | Exact match |
| `servicios_table_fun` | `serviciosTable` | Exact match (all function defaults match R) |
| `combine_lease_serve_fun` | `combineLeaseServe` | Exact match |
| `get_amortization_table` | `getAmortizationTable` | Exact match |
| `get_kpis` (new, `calculate_kpis.R`) | `getIntegrationKpis` | Exact match |

Verified details:
- Payment vector construction (grace zeros + payments + final with opDeComp)
- Grace period interest compounding on starting balance
- Insurance deflation, maintenance inflation formulas
- Spreading denominator `term - gracePeriod + 1`
- Insurance and maintenance premium multipliers
- Debt tracking loop (month 0 initial, months 1+ iterative, floored at 0)
- `iva_cost = iva_payment` (R has "wrong wrong wrong" comment; both use the same formula)
- NII = income - expense for 3 scenarios (Leasing, Servicios, Total)
- Cumulative NII, payback period (9999 sentinel, months 0-2 excluded)
- NPV discounting at `riskFreeRate / 12`
- IRR annualized as `monthly * 12 * 100` (R uses `jrvFinance::irr`, TS uses bisection — different solver, same annualization)
- KPI extraction: same rows, same column structure, Services IRR = null

---

### 5.2 Orchestrator Defaults Differ from R Function Defaults

The `serviciosTable()` function defaults match R exactly. But `quote-generator/index.ts` overrides three of them with different production values:

| Parameter | R function default | TS function default | TS orchestrator passes |
|---|---|---|---|
| `maintPremium` | 0.5 (50%) | 0.5 (50%) | **0.1** (10%) |
| `maintStartMonth` | 36 | 36 | **24** |
| `insurancePremium` | 0 (0%) | 0 (0%) | **0.018** (1.8%) |

These come from event data / frontend input, not from the function itself. The question is whether 0.1 / 24 / 0.018 are the intended current business values or whether they should revert to the R defaults.

---

### 5.3 NII Table (Manual Mode) — Matches R Old Pipeline, Not R New

The TypeScript NII table (`quote-calculator.ts:generateNiiTable`) was ported from the R old pipeline (`calculate_cashflows.R`) and diverges from the R new pipeline in three ways:

**5.3a: No insurance premium in NII table**

- R new (`servicios_table_fun`): `total_insurance_cost = SUM(insurance_cost) * (1 + insurance_premium)`
- R old (`calculate_cashflows_fun_old`): `total_insurance_cost = SUM(insurance_cost)` (no premium)
- TypeScript NII table: `totalInsuranceCost = SUM(insuranceCost)` (no premium) — **matches R old**

**5.3b: Spreading denominator off by 1**

- R new: `term - grace_period + 1`
- R old: `loan_term - seguro_start + 1` where `seguro_start = grace_period + 1`, i.e., `loan_term - grace_period`
- TypeScript NII table: `loanTerm - insuranceStart + 1` where `insuranceStart = gracePeriod + 1`, i.e., `loanTerm - gracePeriod` — **matches R old**

Difference: 61 vs 60 for a 63-month term with 3-month grace.

**5.3c: Missing Sale.Markup KPI**

- R `get_kpis`: returns Sale.Markup = `(retail_price / sum(installation_cost) - 1) * 100`
- TypeScript `getKpis`: does not include Sale.Markup — **missing**

---

### 5.4 NII Table Bug: Monthly Payment KPI Returns 0

**Location:** `quote-calculator.ts:482-483`

The NII table `getKpis()` reads `total_payment_con_iva` for the "w Servicios" and "Total w IVA" Monthly Payment values. But `generateNiiTable()` never computes a `total_payment_con_iva` field on its rows. The field lookup returns 0.

**Fix:** Compute `total_payment_con_iva` in the NII table, or use the existing fields: `income_with_services * (1 + taxRate)` or `income_total`.
