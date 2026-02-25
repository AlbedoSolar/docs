# Quote Generation: Data Flow Map

How data moves through the system — which tables feed which calculations, in what order, and how values propagate.

---

## Table of Contents

1. [End-to-End Flow Overview](#1-end-to-end-flow-overview)
2. [Table Dependency Map](#2-table-dependency-map)
3. [Detailed Flow: Estimate Reference Mode](#3-detailed-flow-estimate-reference-mode)
4. [Detailed Flow: Manual Mode](#4-detailed-flow-manual-mode)
5. [Detailed Flow: Quote Rates Endpoint](#5-detailed-flow-quote-rates-endpoint)
6. [Detailed Flow: Calculate Interest Endpoint](#6-detailed-flow-calculate-interest-endpoint)
7. [Value Lineage: Key Outputs](#7-value-lineage-key-outputs)

---

## 1. End-to-End Flow Overview

```
CLIENT DATA ENTRY (Steps 1-3)
================================

[User Input]                    [Database Tables]
     │                               │
     ▼                               ▼
┌─────────┐    saves to →    ┌──────────────┐
│ Step 1:  │ ──────────────→ │   clients    │
│ Client   │                 └──────────────┘
└────┬─────┘
     │
     ▼
┌─────────┐    saves to →    ┌──────────────┐    reads ←──── ┌──────────────────┐
│ Step 2:  │ ──────────────→ │   projects   │ ◄──────────── │ countries         │
│ Project  │                 └──────────────┘               │ departments       │
└────┬─────┘                       │                        │ municipalities    │
     │                             │                        │ sites             │
     ▼                             │                        │ project_sectors   │
┌─────────┐    saves to →    ┌─────┴────────┐              └──────────────────┘
│ Step 3:  │ ──────────────→ │  estimates   │
│ Estimate │                 │  + related:  │
└────┬─────┘                 │  - project_  │    reads ←──── ┌──────────────────┐
     │                       │    phases    │ ◄──────────── │ providers         │
     │                       │  - estimate_ │               │ commission_types  │
     │                       │    vendors   │               │ user_codes        │
     │                       │  - estimate_ │               │ affiliates        │
     │                       │    affiliates│               │ equipment_*       │
     │                       │  - estimate_ │               │ provider_payment_ │
     │                       │    equipment │               │   schedules       │
     │                       └──────────────┘              └──────────────────┘
     │
     ▼

QUOTE CALCULATION (Step 4)
================================

     │
     ├──→ [quote-rates endpoint]     (pre-populates form)
     │         │
     │         ▼
     │    ┌──────────────────────────────────────────────────┐
     │    │ Returns: maintenance_rate, insurance_rate,       │
     │    │ legal_fee, retail_price_before/after_commission, │
     │    │ commission_costs                                 │
     │    └──────────────────────────────────────────────────┘
     │
     ├──→ [User adjusts form parameters]
     │
     ▼
[quote-generator endpoint]
     │
     ├──→ Reads estimate + all related tables
     ├──→ Reads exchange_rates
     ├──→ Reads countries.tax_rate
     ├──→ Reads insurance_rates
     ├──→ Reads maintenance_rates
     ├──→ Reads providers (margin data)
     ├──→ Reads provider_payment_schedules
     ├──→ Reads quote_generator_configs
     │
     ▼
┌──────────────────────────────────────┐
│ Returns: KPIs + cashflow_table +     │
│ calculation_payload (commission,     │
│ retail prices, rates, schedules)     │
└──────────────┬───────────────────────┘
               │
               ▼
         [User saves quote]
               │
               ├──→ saves to → quotes
               └──→ saves to → monthly_cash_flows
                        │
                        ▼
               [calculate-interest endpoint]
                        │
                        ├──→ Reads quotes + monthly_cash_flows
                        ├──→ Reads estimates + projects + countries
                        │
                        ▼
                  Updates: quotes.monthly_interest_rate
                  Updates: monthly_cash_flows (interest/principal split)
```

---

## 2. Table Dependency Map

Which tables are read during quote generation, and what data they provide.

```
CONFIGURATION
─────────────
quote_generator_configs ──────→ All default parameters
  └── quote_generator_user_     (asset expense splits, commission shares,
        active_configs             tax/insurance/maintenance defaults,
                                   grace period, WACC, risk-free rate,
                                   maintenance-free period, etc.)


LOCATION DATA (determines rates)
────────────────────────────────
countries ────────────────────→ tax_rate
  └── departments
        └── municipalities

insurance_rates ──────────────→ insurance_rate
  (keyed by department_id,       (cascading: department → region → country → global)
   region_id, country_id)

maintenance_rates ────────────→ maintenance_rate
  (keyed by provider_id +        (scored by specificity: dept > region > country > global)
   department_id, region_id,
   country_id)


PROVIDER DATA (determines margin & pricing)
────────────────────────────────────────────
providers ────────────────────→ quote_margin_type ("Add" / "Subtract")
                                standard_wholesale_margin (decimal)
                                default_premium_with_down_payment
                                default_premium_without_down_payment
                                time_premium_per_year
                                max_time_premium

provider_payment_schedules ───→ period_1_payout, period_2_payout, period_3_payout
  (linked via project_phases)    (installation payout timing)


COMMISSION DATA
───────────────
commission_types ─────────────→ primary_vendor_percentage
                                secondary_vendor_percentage
                                sales_director_percentage
                                aliado_percentage
                                embajador_percentage


ESTIMATE DATA (the core input)
──────────────────────────────
estimates ────────────────────→ project_currency
                                partner_contract_currency
                                project_cost_partner_contract_currency
                                commission_type_id
                                down_payment_1_percentage

project_phases ───────────────→ phase_cost (installation cost per phase)
  (linked to estimates)          partner_contract_currency (per phase)
                                provider_id → providers (margin data)
                                provider_payment_schedule_id → schedules

estimate_vendors ─────────────→ role + user_code_id
  (linked to estimates)          (determines which commission % to use)

estimate_affiliates ──────────→ affiliate_type (Aliado / Embajador)
  (linked to estimates)          (determines which commission % to use)


CURRENCY DATA
─────────────
exchange_rates ───────────────→ usd_rate, gtq_rate, hnl_rate
  (keyed by effective_date)      (most recent rate <= reference date)


OUTPUT TABLES
─────────────
quotes ───────────────────────→ quote_reference, sale_type, monthly_interest_rate
                                first_payment_date, contract_signing_date
                                addendum_number, original_quote_id

monthly_cash_flows ───────────→ One row per month with all payment components
  (linked to quotes)             (stored WITHOUT IVA)
```

---

## 3. Detailed Flow: Estimate Reference Mode

Step-by-step data flow when the quote-generator processes an estimate reference.

```
INPUT: estimate_reference (e.g., "CLI-01-01-01")
       + user-configurable parameters (annual_rate, loan_term, etc.)
═══════════════════════════════════════════════════════════════════

PHASE 1: LOAD CONFIG
─────────────────────
quote_generator_user_active_configs ─┐
                                     ├──→ [getActiveConfig()] ──→ config{}
quote_generator_configs ─────────────┘


PHASE 2: LOAD ESTIMATE CONTEXT
───────────────────────────────
estimates ────────────────┐
  ├── project_phases ─────┤
  │     ├── providers ────┤
  │     └── provider_     │
  │         payment_      │
  │         schedules ────┤
  ├── estimate_vendors ───┤──→ [loadQuoteIntegrationContext()] ──→ context{}
  │     └── user_codes ───┤        │
  ├── estimate_affiliates─┤        ├── commission_percentages[]
  │     └── affiliates ───┤        ├── phase_pricing_inputs[]
  └── projects ───────────┤        ├── country_id, department_id
        └── sites ────────┘        ├── project_currency
                                   ├── provider_ids[]
commission_types ─────────────────→├── partner_provided_price
                                   └── down_payment_1_percentage

exchange_rates ───────────────────→ (used for phase currency conversion within context loading)


PHASE 3: RESOLVE LOCATION-BASED RATES
──────────────────────────────────────
countries ───────────→ [getTaxRate()] ─────────────→ taxRate

maintenance_rates ───→ [getMaintenanceRate()] ─────→ maintenanceRate
  (+ primary_provider_id, (overridden by user input
   department_id,          if provided)
   region_id,
   country_id)

  primary_provider_id = provider from the highest-cost phase
  (set in loadQuoteIntegrationContext, matches frontend logic)

insurance_rates ─────→ [getInsuranceRate()] ───────→ insuranceRate
  (+ department_id,      (overridden by user input
   region_id,             if provided)
   country_id)


PHASE 4: CALCULATE RETAIL PRICE
────────────────────────────────
context.phase_pricing_inputs[] ─┐
  (phase_cost,                  │
   margin_type,                 │
   margin_percent)              │
                                ├──→ [calculateProjectRetailWithCommission()]
context.commission_percentages[]│        │
                                │        ├──→ base_retail_pre_discount
config (payout shares) ─────────┘        ├──→ retail_after_discount_pre_commission
                                         ├──→ commission_cost_schedule[]
                                         ├──→ commission_total
                                         └──→ final_retail_with_commission


PHASE 5: CALCULATE LEGAL FEE
─────────────────────────────
final_retail_with_commission ─┐
project_currency ─────────────┤──→ [calculateLegalFee()] ──→ legalFee
partner_provided_price ───────┘


PHASE 6: CALCULATE DOWN PAYMENT
────────────────────────────────
user_input.down_payment_amount ──┐
   OR                            ├──→ downPaymentAmount
final_retail * down_payment_1_pct┘


PHASE 7: CALCULATE INSTALLATION SCHEDULE
─────────────────────────────────────────
project_phases[] ─────────────┐
  ├── phase_cost ─────────────┤
  ├── partner_contract_       │
  │   currency ───────────────┤──→ [calculateInstallationSchedule()]
  └── provider_payment_       │        │
      schedules ──────────────┤        └──→ installationSchedule[3]
                              │             (month 0, 1, 2 payments)
exchange_rates ───────────────┘


PHASE 8: BUILD COMMISSION SCHEDULE
───────────────────────────────────
retail_after_discount ────────┐
commission_percentages[] ─────┤──→ [buildCommissionCostsSchedule()]
loan_term, grace_period ──────┤        │
config (payout shares) ───────┘        └──→ commissionSchedule[loanTerm+1]


PHASE 9: GENERATE LEASE TABLE
──────────────────────────────
final_retail_with_commission ─┐
annual_rate ──────────────────┤
loan_term ────────────────────┤──→ [calculateLease()] ──→ leaseTable[]
grace_period ─────────────────┤        (one row per month: principal,
purchase_option_amount ───────┤         interest, payment, balance)
down_payment_amount ──────────┘


PHASE 10: GENERATE SERVICES TABLE
──────────────────────────────────
installation_total ───────────┐
legal_fee ────────────────────┤
insurance_rate ───────────────┤
insurance_deflation_rate ─────┤──→ [serviciosTable()] ──→ servicesTable[]
insurance_premium ────────────┤        (one row per month: insurance cost,
maintenance_rate ─────────────┤         maint cost, insurance payment,
maintenance_inflation ────────┤         maint payment, legal fee)
maintenance_premium ──────────┤
maintenance_start_month ──────┤
loan_term, grace_period ──────┘


PHASE 11: COMBINE & CALCULATE NII
──────────────────────────────────
leaseTable[] ─────────────────┐
servicesTable[] ──────────────┤
commissionSchedule[] ─────────┤──→ [combineLeaseServe()] ──→ combined[]
installationSchedule[] ───────┘
                                        │
                                        ▼
combined[] ───────────────────┐
wacc ─────────────────────────┤──→ [getAmortizationTable()] ──→ niiTable[]
tax_rate ─────────────────────┤        (adds: income, expense, debt,
risk_free_rate ───────────────┘         NII, cumulative NII, NPV,
                                        payback period per month)

PHASE 12: EXTRACT KPIs
──────────────────────
niiTable[] ───────────────────→ [getIntegrationKpis()] ──→ kpis[]
                                       │
                                       └──→ Income, Expense, Profit, NPV,
                                            IRR, Gross Margin, Payback Period,
                                            Monthly Payment, APR, Sale Markup


OUTPUT: { kpis[], cashflow_table[], calculation_payload{} }
```

---

## 4. Detailed Flow: Manual Mode

```
INPUT: asset_cost + all parameters directly (no estimate_reference)
═══════════════════════════════════════════════════════════════════

[getActiveConfig()] ──→ config{}
       │
       ▼
┌─ IF user provides pricing overrides (tax_rate, insurance_percentage, etc.):
│     Use provided values directly (skip DB lookups)
│
└─ ELSE:
      countries ──→ taxRate
      insurance_rates ──→ insuranceRate
      maintenance_rates ──→ maintenanceRate
      providers ──→ providerPricing{}
       │
       ▼
[generateNiiTable()]
  Inputs:
    asset_cost ──────────────────→ Base for retail price, insurance, maintenance
    commission_percentages[] ────→ Commission schedule
    annual_rate (or recommended)─→ Amortization
    down_payment_percentage ────→ Down payment
    loan_term, grace_period ────→ Schedule structure
    insurance_%, deflation ─────→ Insurance costs
    maintenance_%, inflation ───→ Maintenance costs
    wacc, risk_free_rate ───────→ Debt tracking, NPV
    margin_type, margin_% ──────→ Retail price derivation

  Internal steps:
    1. Calculate retail price from asset_cost + margin
    2. Generate amortization schedule (amortTimeForm)
    3. Build commission schedule
    4. Calculate insurance costs per month
    5. Calculate maintenance costs per month
    6. Spread insurance/maintenance into equal payments
    7. Calculate income, expense, debt, NII per month
    8. Calculate cumulative NII, NPV, payback
       │
       ▼
[getKpis(niiTable)] ──→ kpis[]

OUTPUT: { kpis[], cashflow_table[] }
```

---

## 5. Detailed Flow: Quote Rates Endpoint

Called by the frontend when Step 4 loads, before the user sees the form.

```
INPUT: country_id (required)
       + provider_id, department_id, region_id
       + asset_cost, project_currency
       + estimate_reference (optional)
═══════════════════════════════════════════

[getActiveConfig()] ──→ config{}

maintenance_rates ──→ [getMaintenanceRate()] ──→ maintenance_rate
insurance_rates ───→ [getInsuranceRate()] ─────→ insurance_rate

IF asset_cost AND project_currency provided:
    [calculateLegalFee()] ──→ legal_fee
    (NOTE: uses asset_cost as retailPrice proxy here)

IF estimate_reference provided:
    [loadEstimateContextByReference()]
         │
         ▼
    [calculateRetailProjectionFromEstimateContext()]
         │
         ├──→ retail_price_before_commission
         ├──→ retail_price_after_commission
         └──→ commission_costs

OUTPUT: {
    maintenance_rate,
    insurance_rate,
    maintenance_premium: 0.1,     (hardcoded)
    insurance_premium: 0.018,     (hardcoded)
    legal_fee?,                   (if asset_cost provided)
    retail_price_before_commission?,  (if estimate_reference provided)
    retail_price_after_commission?,
    commission_costs?
}
```

---

## 6. Detailed Flow: Calculate Interest Endpoint

Called after a quote is saved to compute precise interest rate and amortization.

```
INPUT: project_id
       + optional: mode ("new" or "legacy"), recalculate flag
═══════════════════════════════════════════════════════════════

quotes ──────────────┐
monthly_cash_flows ──┤──→ Load quote + all cash flow rows
estimates ───────────┤
projects ────────────┤
countries ───────────┘──→ tax_rate

FOR EACH quote in project:
    │
    ├─ Remove IVA from all amounts:
    │    rawAmount / (1 + taxRate) = amountWithoutIVA
    │
    ├─ Build cash flow array:
    │    Month 0: -retailPrice (negative = outflow)
    │    Month N: monthlyPayment + downPayment + assetTransfer
    │
    ├─ IF mode == "new":
    │    │
    │    ├─ [calculateXIRRForProject()]
    │    │    Uses actual dates for each payment
    │    │    Returns annual interest rate
    │    │
    │    ├─ monthlyRate = (1 + annualRate)^(1/12) - 1
    │    │
    │    └─ [calculateAmortization()]
    │         Daily compound interest:
    │         interest = principal * ((1 + annual)^(days/365) - 1)
    │
    └─ IF mode == "legacy":
         │
         ├─ [calculateIRRForProjectLegacy()]
         │    Simple monthly IRR from cash flows
         │    annualRate = monthlyIRR * 12
         │
         └─ [calculateAmortizationLegacy()]
              interest = principal * monthlyRate

OUTPUT:
    Updates quotes.monthly_interest_rate
    Updates monthly_cash_flows:
        interest_payment_project_currency
        principal_payment_project_currency
        remaining_principal_project_currency
```

---

## 7. Value Lineage: Key Outputs

Tracing where the most important output values come from.

### Retail Price (final_retail_with_commission)

```
project_phases.phase_cost
    │ (converted to project currency via exchange_rates)
    ▼
providers.standard_wholesale_margin + providers.quote_margin_type
    │
    ▼
[calculateRetailPriceComponents()] per phase
    │ (reverses margin for "Add" type, passthrough for "Subtract")
    ▼
SUM across all phases = base_retail_pre_discount
    │
    ▼
user discount input (% or flat amount)
    │
    ▼
retail_after_discount_pre_commission
    │
    ▼
commission_types.{role}_percentage (for each vendor/affiliate)
    │ × retail_after_discount
    │ Split into two tranches → commission_total
    │
    ▼
retail_after_discount + commission_total = FINAL RETAIL PRICE
```

### Monthly Lease Payment

```
FINAL RETAIL PRICE
    │
    ├── minus down_payment
    ├── minus present_value(purchase_option)
    ├── plus grace_period_interest
    │
    ▼
paymentBalance
    │
    ▼
Standard annuity formula with:
    monthly_rate = annual_rate / 12
    payment_months = loan_term - grace_period
    │
    ▼
MONTHLY LEASE PAYMENT
```

### Monthly Insurance Payment

```
installation_total (sum of phase costs in project currency)
    │ × insurance_rate (from insurance_rates table, location-based)
    │
    ▼
annual_insurance_cost
    │ × deflation_rate^year (applied each assessment year)
    │
    ▼
SUM across all assessment years = total_insurance_cost
    │ × (1 + insurance_premium)
    │ ÷ payment_months
    │
    ▼
MONTHLY INSURANCE PAYMENT (flat, applied from serviciosStartMonth to term)
```

### Monthly Maintenance Payment

```
installation_total
    │ × maintenance_rate (from maintenance_rates, provider+location)
    │
    ▼
maintenance_cost_per_visit
    │ × (1 + inflation)^years (for visits after inflation start year)
    │
    ▼
SUM across all visit months = total_maintenance_cost
    │ × (1 + maintenance_premium)
    │ ÷ (term - grace_period + 1)
    │
    ▼
MONTHLY MAINTENANCE PAYMENT (flat, applied from serviciosStartMonth to term)
```

### Commission Total

```
commission_types.primary_vendor_percentage ──┐
commission_types.secondary_vendor_percentage ┤
commission_types.sales_director_percentage ──┤──→ SUM = totalPct
commission_types.aliado_percentage ──────────┤
commission_types.embajador_percentage ───────┘
    (only roles that exist in estimate_vendors / estimate_affiliates)

retail_after_discount × totalPct = COMMISSION TOTAL

Split:
    Month 1: 80% of total
    Month gracePeriod: 20% of total
    (or 100% in month 1 if gracePeriod <= 1)
```

### IRR

```
nii_total[month 0], nii_total[month 1], ..., nii_total[month term]
    │
    ▼
[irrBisection()] ──→ monthly_irr
    │ × 12 × 100
    │
    ▼
ANNUAL IRR (percentage)
```

### Sale Markup

```
retail_price_sin_iva ÷ installation_total - 1 × 100 = SALE MARKUP %
```

---

## Summary: All Database Tables Involved

| Table | Role in Quote Generation |
|-------|--------------------------|
| `clients` | Client identity (Step 1) |
| `projects` | Project location and metadata (Step 2) |
| `sites` | Precise location for rate lookups |
| `countries` | Tax rate |
| `departments` | Rate lookup key |
| `municipalities` | Rate lookup key |
| `estimates` | Core financial inputs (Step 3) |
| `project_phases` | Installation cost per phase |
| `providers` | Margin type, margin %, pricing defaults |
| `provider_payment_schedules` | Installation payout timing |
| `commission_types` | Commission % per role |
| `estimate_vendors` | Which vendor roles are active |
| `estimate_affiliates` | Which affiliate roles are active |
| `user_codes` | Vendor identity |
| `affiliates` | Affiliate type (Aliado/Embajador) |
| `insurance_rates` | Location-based insurance rate |
| `maintenance_rates` | Provider+location maintenance rate |
| `exchange_rates` | Currency conversion |
| `quote_generator_configs` | All configurable defaults |
| `quote_generator_user_active_configs` | Per-user config mapping |
| `quotes` | Output: saved quote record |
| `monthly_cash_flows` | Output: month-by-month payment schedule |
| `estimate_equipment` | Informational only (not used in calculations) |
| `equipment_types` / `equipment_subtypes` / `equipment_brands` | Informational only |
| `equipment_providers` | Informational only |
| `project_sectors` | Informational only |
