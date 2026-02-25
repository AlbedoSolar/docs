# Quote Generation: Calculation Reference

Every formula in the quote generation system, organized by component. For each calculation: the formula, every input (with its source table/field), the output, all defaults, and edge cases.

---

## Table of Contents

1. [Global Configuration Defaults](#1-global-configuration-defaults)
2. [Retail Price Calculation](#2-retail-price-calculation)
3. [Legal Fee Calculation](#3-legal-fee-calculation)
4. [Commission Calculation](#4-commission-calculation)
5. [Tax Rate Resolution](#5-tax-rate-resolution)
6. [Insurance Rate Resolution](#6-insurance-rate-resolution)
7. [Maintenance Rate Resolution](#7-maintenance-rate-resolution)
8. [Provider Pricing / Recommended APR](#8-provider-pricing--recommended-apr)
9. [Currency Conversion](#9-currency-conversion)
10. [Loan Amortization](#10-loan-amortization)
11. [Insurance Cost Schedule](#11-insurance-cost-schedule)
12. [Maintenance Cost Schedule](#12-maintenance-cost-schedule)
13. [Payment Spreading](#13-payment-spreading)
14. [Installation Schedule](#14-installation-schedule)
15. [Combined Table & Debt Tracking](#15-combined-table--debt-tracking)
16. [Net Income (NII) Calculation](#16-net-income-nii-calculation)
17. [NPV Calculation](#17-npv-calculation)
18. [IRR Calculation](#18-irr-calculation)
19. [KPI Summary Table](#19-kpi-summary-table)
20. [Interest Rate Calculation (Post-Quote)](#20-interest-rate-calculation-post-quote)
21. [Down Payment Calculation](#21-down-payment-calculation)
22. [Purchase Option](#22-purchase-option)
23. [Quote Reference Generation](#23-quote-reference-generation)
24. [Sale Type Determination](#24-sale-type-determination)

---

## 1. Global Configuration Defaults

**Source:** `quote_generator_configs` table, with fallbacks in `DEFAULT_CONFIG` (quote-helpers.ts:29-51)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `asset_expense_month_1` | 0.5 | Fraction of installation cost paid to provider in month 0 |
| `asset_expense_month_2` | 0.4 | Fraction paid in month 1 |
| `asset_expense_month_3` | 0.1 | Fraction paid in month 2 |
| `commission_first_payout_share` | 0.8 | First commission tranche (80%) |
| `commission_final_payout_share` | 0.2 | Second commission tranche (20%) |
| `default_tax_rate` | 0.12 | 12% IVA (Guatemala) |
| `default_insurance_percentage` | 0.017 | 1.7% of installation cost annually |
| `default_maintenance_percentage` | 0.028 | 2.8% of installation cost per visit |
| `default_premium_with_down_payment` | 0.02 | APR risk premium when client puts money down |
| `default_premium_without_down_payment` | 0.03 | APR risk premium when no down payment |
| `default_time_premium_per_year` | 0.01 | Additional APR premium per year of loan term |
| `default_max_time_premium` | 0.06 | Cap on time premium |
| `maintenance_free_period_months` | 24 | No maintenance visits for first 24 months |
| `maintenance_inflation_start_year` | 3 | Maintenance inflation kicks in after year 3 |
| `default_grace_period` | 3 | 3 months before payments begin |
| `default_wacc` | 0.1 | 10% weighted average cost of capital |
| `default_risk_free_rate` | 0.04 | 4% risk-free rate for NPV discounting |
| `default_target_margin` | 0 | No target margin by default |
| `default_risk_level` | "Low" | Risk classification |

**Config resolution order:**
1. User-specific config: `quote_generator_user_active_configs.config_id` → `quote_generator_configs`
2. Global active config: `quote_generator_configs WHERE is_active = true`
3. Hardcoded `DEFAULT_CONFIG`

Each level merges with defaults: `{ ...DEFAULT_CONFIG, ...loadedConfig }`.

---

## 2. Retail Price Calculation

**Source code:** `calculateRetailPriceComponents()` (quote-helpers.ts:360-408) and `calculateProjectRetailWithCommission()` (quote-helpers.ts:425-486)

### Step 2a: Phase-Level Base Retail Price

For **each project phase**, the base retail price is calculated from the phase cost and the phase's provider margin.

**Inputs:**
| Input | Source |
|-------|--------|
| `technicalProjectPrice` | `project_phases.phase_cost` (converted to project currency) |
| `officialSolarPartnerMarginPercent` | `providers.standard_wholesale_margin` (decimal, e.g., 0.15) |
| `partnerQuoteMarginType` | `providers.quote_margin_type` ("Add" or "Subtract") |

**Formula — "Add" type (or empty string):**
```
IF marginDecimal == 1.0:
    baseRetail = technicalProjectPrice
ELSE:
    baseRetail = (technicalProjectPrice / (marginDecimal - 1)) * -1
```

**Worked example:** Provider margin = 0.15 (15%), phase cost = $10,000
```
baseRetail = ($10,000 / (0.15 - 1)) * -1
           = ($10,000 / -0.85) * -1
           = $11,764.71
```
The phase cost represents what the provider keeps (85%), so the retail is the full 100%.

**Formula — "Subtract" type:**
```
baseRetail = technicalProjectPrice
```
The phase cost IS the retail price. No margin markup needed.

**Edge case:** If `marginDecimal == 1.0` with "Add" type, no division occurs (would be division by zero). The base retail equals the technical project price directly.

### Step 2b: Aggregate Across Phases

```
totalBaseRetailPreDiscount = SUM(baseRetail for each phase)
```

Each phase's cost is first converted to project currency (see [Currency Conversion](#9-currency-conversion)).

### Step 2c: Apply Discount

**Inputs:**
| Input | Source |
|-------|--------|
| `retailPriceDiscount` | User input via `eventData.retail_price_discount` |

**Discount type resolution** (`resolveDiscountFields()`, quote-helpers.ts:410-423):
```
IF retailPriceDiscount < 1:
    Type = "%"  (percentage discount)
    priceDiscount = retailPriceDiscount
ELSE:
    Type = "Monto"  (flat amount)
    descuentoAdicional = retailPriceDiscount
```

**Formula — Percentage discount:**
```
retailAfterDiscount = totalBaseRetail - (priceDiscount * totalBaseRetail)
                    = totalBaseRetail * (1 - priceDiscount)
```

**Formula — Flat amount (Monto):**
```
retailAfterDiscount = totalBaseRetail - descuentoAdicional
```

**Formula — No discount type / default:**
```
retailAfterDiscount = totalBaseRetail - (priceDiscount * totalBaseRetail)
```
(Same as percentage.)

### Step 2d: Calculate Commission Amount

See [Commission Calculation](#4-commission-calculation) for how commission percentages are derived.

```
commissionTotal = SUM(commissionCostSchedule)
```

The commission cost schedule is built from `retailAfterDiscount * SUM(commissionPercentages)`, split into two tranches.

### Step 2e: Final Retail Price

```
finalRetailWithCommission = retailAfterDiscount + commissionTotal
```

**Note:** In the per-phase `calculateRetailPriceComponents()` function, there is also a per-phase commission calculation:
```
commissionAmount = officialVendorCommissionPercent * retailAfterDiscount / 1.12
```
The division by 1.12 removes IVA from the commission base. This is only applied for "Add" type margins. For "Subtract" type, `finalRetailPrice = retailAfterDiscount` (commission is not added).

### NII Table Variant (Manual Mode)

In the NII table generator (quote-calculator.ts:186-202), the retail price calculation is simpler:

```
IF officialSolarPartnerMarginPercent is provided:
    IF partnerQuoteMarginType == "Add" or "":
        IF marginDecimal == 1.0:
            retailPrice = assetCost
        ELSE:
            retailPrice = (assetCost / (marginDecimal - 1)) * -1
    ELSE:
        retailPrice = assetCost
ELSE:
    retailPrice = assetCost  (no provider margin at all)
```

No discount or commission is applied to the retail price in this path — commission is handled separately in the NII table rows.

---

## 3. Legal Fee Calculation

**Source code:** `calculateLegalFee()` (quote-helpers.ts:302-327)

**Inputs:**
| Input | Source |
|-------|--------|
| `retailPrice` | Calculated retail price (see above) |
| `projectCurrency` | `estimates.project_currency` |
| `projectCostPartnerContractCurrency` | `estimates.project_cost_partner_contract_currency` |
| `config.default_legal_costs_percentage` | Config table (default: 0.05) |

**Formula — USD projects:**
```
IF retailPrice <= 8000:  return 150
IF retailPrice <  13000: return 200
ELSE:                     return 300
```

**Formula — GTQ projects:**
```
IF retailPrice <= 60000:  return 1175
IF retailPrice <  100000: return 1600
ELSE:                      return 2350
```

**Formula — Other currencies:**
```
legalFee = projectCostPartnerContractCurrency * default_legal_costs_percentage
```
Default percentage: 5% (0.05).

**Edge cases:**
- Currency comparison is case-insensitive (uppercased before comparison)
- If `projectCostPartnerContractCurrency` is null/undefined for non-USD/GTQ currencies, the fee is 0

**Note on quote-rates endpoint:** The legal fee in the `quote-rates` response uses `asset_cost` as the `retailPrice` input (not the calculated retail price), since the full retail calculation hasn't happened yet at that point. The quote-generator recalculates it with the actual retail price.

---

## 4. Commission Calculation

### 4a. Deriving Commission Percentages

**Source code:** `deriveCommissionPercentages()` (quote-helpers.ts:530-557)

**Inputs:**
| Input | Source |
|-------|--------|
| Vendor roles | `estimate_vendors.role` (primary_vendor, secondary_vendor, sales_director) |
| Affiliate types | `estimate_affiliates.affiliates.affiliate_type` (Aliado, Embajador) |
| Role percentages | `commission_types.*_percentage` fields |

**Process:**
```
percentages = []

FOR each vendor in estimate_vendors:
    role_field = {
        "primary_vendor":   "primary_vendor_percentage",
        "secondary_vendor": "secondary_vendor_percentage",
        "sales_director":   "sales_director_percentage",
    }[vendor.role]

    IF commission_type[role_field] is not null:
        percentages.push(commission_type[role_field])

FOR each affiliate_link in estimate_affiliates:
    IF affiliate.affiliate_type == "Aliado":
        IF commission_type.aliado_percentage is not null:
            percentages.push(commission_type.aliado_percentage)
    IF affiliate.affiliate_type == "Embajador":
        IF commission_type.embajador_percentage is not null:
            percentages.push(commission_type.embajador_percentage)

RETURN percentages  (array of decimals, e.g., [0.03, 0.02, 0.01])
```

### 4b. Building the Commission Payout Schedule

**Source code:** `buildCommissionCostsSchedule()` (quote-helpers.ts:329-358)

**Inputs:**
| Input | Source | Default |
|-------|--------|---------|
| `loanTerm` | User input | 63 |
| `gracePeriod` | User input | 3 |
| `retailPrice` | Calculated retail price (before commission is added back) | - |
| `commissionPercentages` | From 4a above | [] |
| `firstPayoutShare` | Config `commission_first_payout_share` | 0.8 |
| `finalPayoutShare` | Config `commission_final_payout_share` | 0.2 |

**Formula:**
```
schedule = array of (loanTerm + 1) zeros

totalPct = SUM(commissionPercentages)
totalCommission = retailPrice * totalPct

firstMonth = 1
finalMonth = MIN(MAX(gracePeriod, firstMonth), loanTerm)

IF firstMonth == finalMonth:
    schedule[firstMonth] = totalCommission * (firstPayoutShare + finalPayoutShare)
ELSE:
    schedule[firstMonth] = totalCommission * firstPayoutShare
    schedule[finalMonth] = totalCommission * finalPayoutShare
```

**Worked example:** retailPrice = $11,765, commissionPercentages = [0.03, 0.02], gracePeriod = 3
```
totalCommission = $11,765 * 0.05 = $588.25
schedule[1] = $588.25 * 0.8 = $470.60  (month 1)
schedule[3] = $588.25 * 0.2 = $117.65  (month 3)
All other months = $0
```

**Edge case:** If `gracePeriod <= 1`, then `firstMonth == finalMonth == 1`, and the full commission is paid in month 1.

**Edge case:** If `commissionPercentages` is empty, `totalPct = 0`, `totalCommission = 0`, all schedule entries are 0.

---

## 5. Tax Rate Resolution

**Source code:** `getTaxRate()` (quote-helpers.ts:96-118)

**Inputs:**
| Input | Source |
|-------|--------|
| `countryId` | From project site's `country_id` |

**Resolution:**
```
1. IF countryId is null/undefined → return config.default_tax_rate (0.12)
2. Fetch countries.tax_rate WHERE id = countryId
3. IF found and not null → return that rate
4. ELSE → return config.default_tax_rate (0.12)
```

**Known tax rates:**
| Country | Rate |
|---------|------|
| Guatemala | 0.12 (12%) |
| El Salvador | 0.13 (13%) |
| Honduras | 0.15 (15%) |

---

## 6. Insurance Rate Resolution

**Source code:** `getInsuranceRate()` (quote-helpers.ts:194-263)

**Inputs:**
| Input | Source |
|-------|--------|
| `departmentId` | From project site's `department_id` |
| `regionId` | From project site's `region_id` (via department) |
| `countryId` | From project site's `country_id` |

**Resolution — cascading priority, stops at first match:**
```
1. Department-specific:
   SELECT insurance_rate FROM insurance_rates
   WHERE department_id = :departmentId
   → If found, return it

2. Region-specific (no department):
   SELECT insurance_rate FROM insurance_rates
   WHERE region_id = :regionId AND department_id IS NULL
   → If found, return it

3. Country-specific (no department or region):
   SELECT insurance_rate FROM insurance_rates
   WHERE country_id = :countryId AND department_id IS NULL AND region_id IS NULL
   → If found, return it

4. Global (all location fields null):
   SELECT insurance_rate FROM insurance_rates
   WHERE department_id IS NULL AND region_id IS NULL AND country_id IS NULL
   → If found, return it

5. Fallback: config.default_insurance_percentage (0.017)
```

**Edge cases:**
- Each step only queries if the corresponding ID is not null
- If `departmentId` is null, step 1 is skipped entirely
- The query explicitly checks for NULL on other fields to avoid catching broader records

---

## 7. Maintenance Rate Resolution

**Source code:** `getMaintenanceRate()` (quote-helpers.ts:120-192)

**Inputs:**
| Input | Source |
|-------|--------|
| `providerId` | From first project phase's `provider_id` |
| `departmentId` | From project site |
| `regionId` | From project site |
| `countryId` | From project site |

**Provider selection:** When multiple phases exist with different providers, both the frontend and backend select the provider from the phase with the highest cost (in project currency). This is the `primary_provider_id` returned by `loadQuoteIntegrationContext()`. The rationale is that the highest-cost phase typically represents the primary installer who handles maintenance.

**Resolution — single query, scored by specificity:**
```
1. IF providerId is null → return config.default_maintenance_percentage (0.028)

2. Fetch ALL maintenance_rates WHERE provider_id = :providerId

3. Score each row by specificity:
   - department_id matches → specificity = 4 (highest)
   - region_id matches    → specificity = 3
   - country_id matches   → specificity = 2
   - all location fields null AND all input location fields null → specificity = 1
   - no match → skip this row

4. Return rate from highest-specificity matching row

5. If no rows match → return config.default_maintenance_percentage (0.028)
```

**Key difference from insurance:** Maintenance rates are provider-specific. Insurance rates are not — they're purely location-based. Also, maintenance uses a single query with client-side scoring, while insurance uses separate cascading queries.

**Edge case:** A row with `department_id` set will ONLY match if the input `departmentId` also matches. If the row has `department_id = 5` but input `departmentId = 3`, it's skipped — not treated as a region or country-level match.

**Edge case:** The "global" fallback (specificity 1) only matches if ALL input location fields are also null. If you have a provider with a global rate but you're querying with a specific country, the global rate won't match.

---

## 8. Provider Pricing / Recommended APR

### 8a. Provider Pricing Lookup

**Source code:** `getProviderPricing()` (quote-helpers.ts:265-300)

**Inputs:**
| Input | Source |
|-------|--------|
| `providerId` | First provider from project phases |

**Resolution:**
```
1. IF providerId is null → return config defaults
2. Fetch from providers table:
   - default_premium_with_down_payment
   - default_premium_without_down_payment
   - time_premium_per_year
   - max_time_premium
3. For each field, if the provider value is null → use config default
```

| Field | Config Default | Hardcoded Fallback |
|-------|---------------|-------------------|
| `default_premium_with_down_payment` | `config.default_premium_with_down_payment` | 0.02 |
| `default_premium_without_down_payment` | `config.default_premium_without_down_payment` | 0.03 |
| `time_premium_per_year` | `config.default_time_premium_per_year` | 0.01 |
| `max_time_premium` | `config.default_max_time_premium` | 0.06 |

### 8b. Recommended APR

**Source code:** `calcRecommendedApr()` (quote-calculator.ts:35-60)

**Formula:**
```
defaultPremium = IF downPayment > 0
                   THEN providerPricing.default_premium_with_down_payment
                   ELSE providerPricing.default_premium_without_down_payment

timePremium = MIN(
    (loanTerm / 12) * providerPricing.time_premium_per_year,
    providerPricing.max_time_premium
)

recommendedAPR = WACC + targetMargin + defaultPremium + timePremium
```

**Worked example:** WACC = 0.10, targetMargin = 0, downPayment = $500 (> 0), term = 63 months
```
defaultPremium = 0.02
timePremium = MIN((63/12) * 0.01, 0.06) = MIN(0.0525, 0.06) = 0.0525
recommendedAPR = 0.10 + 0 + 0.02 + 0.0525 = 0.1725 (17.25%)
```

**Usage:** If the user provides an `annual_rate`, it overrides the recommended APR. If not, the recommended APR is used.

---

## 9. Currency Conversion

**Source code:** `convertAmountBetweenCurrencies()` (quote-helpers.ts:499-528) and `getExchangeRatesRow()` (quote-helpers.ts:488-497)

### Exchange Rate Lookup

```
SELECT usd_rate, gtq_rate, hnl_rate, effective_date
FROM exchange_rates
WHERE effective_date <= :referenceDate
ORDER BY effective_date DESC
LIMIT 1
```

`referenceDate` defaults to today if not provided.

### Conversion Formula

```
Step 1: Convert to USD
    IF fromCurrency == "USD":
        usdAmount = amount
    ELSE:
        usdAmount = amount / exchangeRates[fromCurrency]

Step 2: Convert from USD to target
    IF toCurrency == "USD":
        return usdAmount
    ELSE:
        return usdAmount * exchangeRates[toCurrency]
```

**Supported currencies:**
- USD (rate is always 1)
- GTQ (Guatemalan Quetzal)
- HNL (Honduran Lempira)

**Edge cases:**
- If `fromCurrency == toCurrency`, returns amount unchanged
- If exchange rate is null or <= 0, returns amount unchanged (no conversion)
- If no exchange rate row exists at all, returns amount unchanged
- Currency strings are uppercased and trimmed before comparison

---

## 10. Loan Amortization

### Integration Flow (Estimate Reference Mode)

**Source code:** `calculatePayment()` and `calculateLease()` (quote-integration-flow.ts:8-86)

**Inputs:**
| Input | Source | Default |
|-------|--------|---------|
| `retailPriceSinIva` | Calculated retail price with commission | - |
| `annualRate` | User input or recommended APR | 0.18 |
| `term` | User input `loan_term` | 63 |
| `gracePeriod` | User input | 3 |
| `opDeComp` | `retailPrice * op_de_comp_pct` | retailPrice * 0.01 |
| `downpaymentAmt` | User input or calculated | 0 |

**Payment calculation (`calculatePayment()`):**
```
monR = annualRate / 12  (monthly rate)
monN = term - gracePeriod  (payment months)

num = monR * (1 + monR)^monN
den = (1 + monR)^monN - 1

opDeCompFv = opDeComp * (1 + monR)^(-monN)  (present value of purchase option)
balanceLessOdc = startingBalance - opDeCompFv
gracePeriodInterest = startingBalance * (1 + monR)^gracePeriod - startingBalance
paymentBalance = balanceLessOdc + gracePeriodInterest

IF den == 0:
    payment = paymentBalance / MAX(monN, 1)
ELSE:
    payment = paymentBalance * num / den

Output vector:
    [0, 0, 0, ...(gracePeriod zeros)..., payment, payment, ..., payment + opDeComp]
```

The starting balance is `retailPriceSinIva - downpaymentAmt`.

**Lease table (`calculateLease()`):**

Month 0:
```
starting_principal = retailPriceSinIva
down_payment = downpaymentAmt
ending_principal = retailPriceSinIva - downpaymentAmt
```

Months 1 to term:
```
interestAccrued = startingPrincipal * (annualRate / 12)
leasePayment = paymentVector[month - 1]
interestPayment = MIN(leasePayment, interestAccrued)
principalPayment = leasePayment - interestPayment
endingPrincipal = startingPrincipal + interestAccrued - leasePayment
```

**Edge case:** During grace period, `leasePayment = 0`, so `interestPayment = 0`, `principalPayment = 0`, and the ending principal grows by `interestAccrued` each month.

### NII Table (Manual Mode)

**Source code:** `amortForm()` and `amortTimeForm()` (quote-calculator.ts:62-136)

**Formula per month (`amortForm()`):**
```
monthlyRate = annualRate / 12
numerator = monthlyRate * (1 + monthlyRate)^term
denominator = (1 + monthlyRate)^term - 1

IF denominator == 0:
    payment = principalBalanceTarget / MAX(term, 1)
ELSE:
    payment = principalBalanceTarget * numerator / denominator

interestAccrued = principalBalance * monthlyRate
principalPayment = payment - interestAccrued

IF inGracePeriod:
    realPayment = 0
    realPrincipalPayment = 0
    realInterestPayment = 0
ELSE:
    realPayment = payment
    realPrincipalPayment = principalPayment
    realInterestPayment = interestAccrued

newBalance = principalBalance + interestAccrued - realPayment
```

**Grace period handling in `amortTimeForm()`:**
- Months 1 to gracePeriod: `inGrace = true`, payments are 0, balance grows with interest
- Month `gracePeriod + 1`: `principalBalanceTarget` is reset to the current balance (which has accumulated grace period interest), and `calcTerm` is set to `term - gracePeriod`
- Months after grace period: regular amortization with `calcTerm = term - gracePeriod`

---

## 11. Insurance Cost Schedule

### Integration Flow

**Source code:** `serviciosTable()` (quote-integration-flow.ts:88-169)

**Inputs:**
| Input | Source | Default |
|-------|--------|---------|
| `installationCostSinIva` | Sum of phase costs in project currency | - |
| `insuranceRate` | Location-based (see #6) | 0.012093 |
| `insuranceDeflationRate` | User input | 0.96 |
| `insurancePremium` | Config/user input | 0.018 |
| `insuranceStartMonth` | Config/user input | 4 |

**Annual cost assessment formula:**
```
insuranceFirstCost = installationCostSinIva * insuranceRate

FOR each month 0 to term:
    IF (12 - insuranceStartMonth + month) % 12 == 0 AND month <= term:
        insuranceCost = insuranceFirstCost * insuranceDeflationRate^(floor(month / 12))
    ELSE:
        insuranceCost = 0
```

This assesses insurance every 12 months, with the first assessment occurring such that the modular arithmetic aligns with `insuranceStartMonth`. Each subsequent year, the cost deflates.

**Worked example:** insuranceStartMonth = 4, so costs hit at months 4, 16, 28, 40, 52.
```
Month 4:  cost = firstCost * 0.96^0 = firstCost * 1.0
Month 16: cost = firstCost * 0.96^1 = firstCost * 0.96
Month 28: cost = firstCost * 0.96^2 = firstCost * 0.9216
Month 40: cost = firstCost * 0.96^3 = firstCost * 0.8847
Month 52: cost = firstCost * 0.96^4 = firstCost * 0.8493
```

### NII Table (Manual Mode)

**Source code:** `generateNiiTable()` (quote-calculator.ts:246-248)

The formula is the same but uses `insuranceStart = gracePeriod + 1` instead of a configurable `insuranceStartMonth`:
```
insuranceCostEstimate = assetCost * insurancePct
insuranceStart = gracePeriod + 1

FOR each month:
    IF month > loanTerm OR (12 - insuranceStart + month) % 12 != 0:
        insuranceCost = 0
    ELSE:
        insuranceCost = insuranceCostEstimate * insuranceDeflationRate^(floor(month / 12))
```

---

## 12. Maintenance Cost Schedule

### Integration Flow

**Source code:** `serviciosTable()` (quote-integration-flow.ts:128-135)

**Inputs:**
| Input | Source | Default |
|-------|--------|---------|
| `installationCostSinIva` | Sum of phase costs | - |
| `maintRate` | Location+provider based (see #7) | 0.03 |
| `maintInflation` | User input | 0.05 |
| `maintPremium` | Config/user input | 0.5 (50%) |
| `maintStartMonth` | Config/user input | 36 |
| `maintFreq` | Config/user input | 12 (annual) |

**Note:** The integration flow default for `maintPremium` is 0.5 (50%), which differs from the NII table default of 0.1 (10%). The quote-generator explicitly passes 0.1 from the event data, overriding the integration flow default.

**Assessment formula:**
```
maintFirstCost = installationCostSinIva * maintRate

FOR each month 0 to term:
    IF month >= maintStartMonth
       AND month <= term
       AND (month - maintStartMonth) % maintFreq == 0:
        inflationYears = MAX(floor((month - maintStartMonth) / 12), 0)
        maintCost = maintFirstCost * (1 + maintInflation)^inflationYears
    ELSE:
        maintCost = 0
```

**Worked example:** maintStartMonth = 24, maintFreq = 12, maintInflation = 0.05
```
Month 24: cost = firstCost * (1.05)^0 = firstCost * 1.0
Month 36: cost = firstCost * (1.05)^1 = firstCost * 1.05
Month 48: cost = firstCost * (1.05)^2 = firstCost * 1.1025
Month 60: cost = firstCost * (1.05)^3 = firstCost * 1.1576
```

### NII Table (Manual Mode)

**Source code:** `generateNiiTable()` (quote-calculator.ts:251-258)

```
maintenanceCostPerVisit = maintenancePct * assetCost
maintenanceFreeMonths = config.maintenance_free_period_months (24)
maintenanceInflationStartYear = config.maintenance_inflation_start_year (3)

FOR each month:
    IF month > maintenanceFreeMonths AND month <= loanTerm AND month % 12 == 0:
        inflationYears = MAX(floor(month / 12) - maintenanceInflationStartYear, 0)
        maintenanceCost = maintenanceCostPerVisit * (1 + maintenanceInflationRate)^inflationYears
    ELSE:
        maintenanceCost = 0
```

**Key difference:** The NII table uses `month % 12 == 0` (every 12 months from the start, starting at month 24) while the integration flow uses `(month - maintStartMonth) % maintFreq == 0` (every `maintFreq` months from `maintStartMonth`).

Also, the NII table inflation years are calculated as `floor(month/12) - inflationStartYear` while the integration flow uses `floor((month - startMonth) / 12)`.

---

## 13. Payment Spreading

Both insurance and maintenance costs are summed across all months, then spread as equal monthly payments.

### Integration Flow

**Source code:** `serviciosTable()` (quote-integration-flow.ts:147-161)

```
totalMainCost = SUM(all maintCost values) * (1 + maintPremium)
totalInsuranceCost = SUM(all insuranceCost values) * (1 + insurancePremium)

denominator = MAX(term - gracePeriod + 1, 1)

maintPayment = totalMainCost / denominator
insurancePayment = totalInsuranceCost / denominator

Applied to months: serviciosStartMonth through term (inclusive)
    (serviciosStartMonth defaults to gracePeriod + 1)
```

**Important note:** The denominator uses `term - gracePeriod + 1`, NOT `term - serviciosStartMonth + 1`. The code comment says this is to "allow for discounts" and matches the original R implementation. This means the payment divisor may include months where payments aren't actually applied, resulting in slightly lower monthly payments.

### NII Table (Manual Mode)

**Source code:** `generateNiiTable()` (quote-calculator.ts:271-281)

```
totalMaintenanceCost = SUM(all maintenanceCost) * (1 + maintenancePremium)
totalInsuranceCost = SUM(all insuranceCost)  // NO premium multiplier in this path

paymentMonths = MAX(loanTerm - insuranceStart + 1, 1)
maintenancePayment = totalMaintenanceCost / paymentMonths
insurancePayment = totalInsuranceCost / paymentMonths

Applied to months: insuranceStart through loanTerm (inclusive)
    (insuranceStart = gracePeriod + 1)
```

**Key difference:** In the NII table path, insurance does NOT have a premium multiplier applied to the total. The `insurance_percentage` input already accounts for this. Only maintenance gets the premium (default 10%).

---

## 14. Installation Schedule

**Source code:** `calculateInstallationSchedule()` (quote-helpers.ts:805-935)

This determines how Albedo pays the provider — not a single lump sum, but in installments.

**Inputs:**
| Input | Source | Default |
|-------|--------|---------|
| Phase costs | `project_phases.phase_cost` | - |
| Phase currencies | `project_phases.partner_contract_currency` | "USD" |
| Payout schedule | `provider_payment_schedules.period_1/2/3_payout` | [0.5, 0.4, 0.1] |

**Formula:**
```
FOR each phase:
    phaseCostProjectCurrency = convertCurrency(phaseCost, phaseCurrency, projectCurrency)

    p1 = period_1_payout (or fallback 0.5)
    p2 = period_2_payout (or fallback 0.4)
    p3 = period_3_payout (or fallback 0.1)

    totals[0] += phaseCostProjectCurrency * p1
    totals[1] += phaseCostProjectCurrency * p2
    totals[2] += phaseCostProjectCurrency * p3

Output: [month0_payment, month1_payment, month2_payment]
```

**In the combined table** (for Estimate Reference Mode), `binaryForIan` is set to 0, which means the installation schedule values are used directly (already in absolute terms from the calculation above). If it were 1, the schedule values would be multiplied by `installationCostSinIvaVal` (treating them as percentages).

**In the NII table** (Manual Mode), the installation schedule is simulated via `capital_costs`:
```
Month 0: assetCost * asset_expense_month_1 (default 0.5)
Month 1: assetCost * asset_expense_month_2 (default 0.4)
Month 2: assetCost * asset_expense_month_3 (default 0.1)
```

**Edge cases:**
- If `requireProviderPayoutSchedule = true` and any period payout is null, an error is thrown
- If the installation total is <= 0 after calculation, an error is thrown
- If no phases exist, returns [0, 0, 0] (or throws if `throwOnError = true`)

---

## 15. Combined Table & Debt Tracking

### Integration Flow

**Source code:** `combineLeaseServe()` (quote-integration-flow.ts:176-208) and `getAmortizationTable()` (quote-integration-flow.ts:210-292)

**Combining step:** Merges lease table + services table + commission schedule + installation schedule into one row per month.

**Debt tracking:**

Month 0:
```
total_payment_sin_iva = leasePayment + downPayment + legalFee + insurancePayment + maintPayment
iva_payment = total_payment_sin_iva * taxRate
total_payment_con_iva = total_payment_sin_iva + iva_payment

predebt_cost_sin_iva = installationCost + commissionCost + maintCost + insuranceCost
iva_cost = iva_payment
predebt_cost_con_iva = predebt_cost_sin_iva + iva_cost

debt_cost = (predebt_cost_con_iva - total_payment_con_iva) * (wacc / 12)
debt_balance = predebt_cost_con_iva + debt_cost - total_payment_con_iva
```

Months 1+:
```
debtBase = previous_debt_balance + current_predebt_cost_con_iva - current_total_payment_con_iva
debt_cost = debtBase * (wacc / 12)
debt_balance = debtBase * (1 + wacc / 12)
```

**Floor:** `debt_cost = MAX(debt_cost, 0)` and `debt_balance = MAX(debt_balance, 0)`. Debt never goes negative.

### NII Table (Manual Mode)

**Source code:** `generateNiiTable()` (quote-calculator.ts:297-313)

Month 0:
```
debt_balance = assetCost * asset_expense_month_1 + commission_costs[0]
             - (payment + down_payment + purchase_option)
```

Months 1+:
```
debt_balance = (prev_debt_balance + capital_costs - income_lease) * (1 + wacc / 12)
```

Month i:
```
debt_cost = IF i == 0 THEN 0 ELSE MAX(prev_debt_balance * (wacc / 12), 0)
```

---

## 16. Net Income (NII) Calculation

### Integration Flow

**Source code:** `getAmortizationTable()` (quote-integration-flow.ts:248-268)

```
expense_lease = installationCost + commissionCost + debtCost + legalFee + ivaCost
expense_services = maintCost + insuranceCost
expense_total = expense_lease + expense_services

income_lease = leasePayment + downPayment + legalFee + ivaPayment
income_services = maintPayment + insurancePayment
income_total = income_lease + income_services

nii_lease = income_lease - expense_lease
nii_services = income_services - expense_services
nii_total = income_total - expense_total
```

**Cumulative NII:**
```
cum_nii_lease[month] = cum_nii_lease[month-1] + nii_lease[month]
cum_nii_services[month] = cum_nii_services[month-1] + nii_services[month]
cum_nii_total[month] = cum_nii_total[month-1] + nii_total[month]
```

**Payback period:**
```
pos_nii_lease_mon = IF month <= 2 THEN 9999
                    ELSE IF cum_nii_lease < 0 THEN 9999
                    ELSE month
```
(Same pattern for services and total.)

### NII Table (Manual Mode)

**Source code:** `generateNiiTable()` (quote-calculator.ts:282-354)

The NII table has four scenarios instead of three:

```
income_lease = payment + down_payment + purchase_option
income_with_services = income_lease + insurance_payment + legal_costs + maintenance_payment
tax = income_with_services * taxRate
income_total = income_with_services + tax

expense_lease = capital_costs + commission_costs
expense_lease_with_debt = expense_lease + debt_cost
expense_with_services = expense_lease_with_debt + maintenance_cost + insurance_cost
cost_tax (see below)
expense_total = expense_with_services + cost_tax

net_income_lease = income_lease - expense_lease
net_income_lease_with_debt = income_lease - expense_lease_with_debt
net_income_with_services = income_with_services - expense_with_services
net_income_total = income_total - expense_total
```

**cost_tax calculation:**
```
cost_tax_total = (
    SUM(income_with_services) - (SUM(capital_costs) - SUM(commission_costs))
) * taxRate / (1 + taxRate)

cost_tax_per_month = cost_tax_total / MAX(loanTerm - insuranceStart, 1)

Applied to months: insuranceStart through loanTerm (inclusive)
```

---

## 17. NPV Calculation

**Source code:** Both engines — quote-calculator.ts:359-365 and quote-integration-flow.ts:294-300

**Formula (identical in both):**
```
discount_factor[month] = 1 / (1 + riskFreeRate / 12)^month

npv_nii[month] = discount_factor[month] * nii[month]

NPV = SUM(npv_nii for all months)
```

The risk-free rate is annual (default 0.04 / 4%), divided by 12 for monthly discounting.

---

## 18. IRR Calculation

**Source code:** Both engines — quote-calculator.ts:367-398 and quote-integration-flow.ts:302-330

### Bisection Method

```
cashflows = [nii[0], nii[1], nii[2], ..., nii[term]]

Guard: must have both positive and negative values, otherwise return null

low = -0.9999
high = 10.0

FOR 200 iterations:
    mid = (low + high) / 2
    npvMid = NPV(mid, cashflows)

    IF |npvMid| < 1e-9:
        RETURN mid  (converged)

    IF NPV(low, cashflows) * npvMid < 0:
        high = mid
    ELSE:
        low = mid

RETURN (low + high) / 2  (best approximation)
```

### Annualization

```
monthlyIRR = irrBisection(cashflows)
annualIRR = ROUND(monthlyIRR * 12 * 100, 2)  (as percentage)
```

**Edge case:** If cashflows don't have both positive and negative values, IRR returns null.

### Post-Quote XIRR (calculate-interest endpoint)

After a quote is saved, the `calculate-interest` edge function recalculates the interest rate using XIRR with actual dates:

**New mode (XIRR):**
```
cashflows_with_dates = [
    { amount: -retailPrice, date: first_payment_date },
    { amount: monthly_payment + down_payment + asset_transfer, date: payment_date },
    ...
]

Filter out zero/negative flows
XIRR returns ANNUAL rate

monthlyRate = (1 + annualRate)^(1/12) - 1
```

**Legacy mode (IRR):**
```
cashflows = [-retailPrice, payment1, payment2, ...]
IRR returns MONTHLY rate
annualRate = monthlyRate * 12
```

**Amortization (new mode):**
```
FOR each payment:
    daysBetween = (currentDate - previousDate) in days
    daysFraction = daysBetween / 365
    interestPayment = remainingPrincipal * ((1 + annualRate)^daysFraction - 1)
    principalPayment = totalTowardPrincipal - interestPayment
    remainingPrincipal -= principalPayment
```

**Amortization (legacy mode):**
```
interestPayment = remainingPrincipal * monthlyInterestRate
principalPayment = totalTowardPrincipal - interestPayment
remainingPrincipal -= principalPayment
```

---

## 19. KPI Summary Table

### Integration Flow KPIs

**Source code:** `getIntegrationKpis()` (quote-integration-flow.ts:332-388)

| KPI | Leasing | Servicios | Total |
|-----|---------|-----------|-------|
| Income | SUM(income_lease) | SUM(income_services) | SUM(income_total) |
| Expense | SUM(expense_lease) | SUM(expense_services) | SUM(expense_total) |
| Profit | Income - Expense | Income - Expense | Income - Expense |
| NPV | SUM(npv_nii_lease) | SUM(npv_nii_services) | SUM(npv_nii_total) |
| IRR | annualizedIRR(nii_lease) | null | annualizedIRR(nii_total) |
| Gross Margin | (Profit/Expense)*100 | (Profit/Expense)*100 | (Profit/Expense)*100 |
| Payback Period | MIN(pos_nii_lease_mon) | MIN(pos_nii_services_mon) | MIN(pos_nii_total_mon) |
| Monthly Payment | month10.lease_payment | month10.insurance_payment + month5.maint_payment | month10.total_payment_con_iva |
| APR | month5.annual_rate * 100 | same | same |
| Sale Markup | ((retailPrice / installationTotal) - 1) * 100 | same | same |

**Note:** IRR for Servicios is always null.

**Mapping to frontend shape:** The integration flow KPIs have columns (Leasing, Servicios, Total) but the frontend expects (Leasing, "Leasing w Debt", "w Servicios", "Total w IVA"). The mapping is:
```
"Leasing"        = Leasing
"Leasing w Debt" = Leasing  (duplicate)
"w Servicios"    = Servicios
"Total w IVA"    = Total
```

### NII Table KPIs

**Source code:** `getKpis()` (quote-calculator.ts:400-493)

| KPI | Leasing | Leasing w Debt | w Servicios | Total w IVA |
|-----|---------|----------------|-------------|-------------|
| Income | SUM(income_lease) | SUM(income_lease) | SUM(income_with_services) | SUM(income_total) |
| Expense | SUM(expense_lease) | SUM(expense_lease_with_debt) | SUM(expense_with_services) | SUM(expense_total) |
| Profit | Income - Expense | Income - Expense | Income - Expense | Income - Expense |
| NPV | SUM(npv_lease) | SUM(npv_lease_with_debt) | SUM(npv_with_services) | SUM(npv_total) |
| IRR | annualizedIRR(net_income_lease) | annualizedIRR(net_income_lease_with_debt) | annualizedIRR(net_income_with_services) | annualizedIRR(net_income_total) |
| Gross Margin | (Profit/Expense)*100 | (Profit/Expense)*100 | (Profit/Expense)*100 | (Profit/Expense)*100 |
| Payback Period | MIN(payback_period_lease) | MIN(payback_period_lease_with_debt) | MIN(payback_period_with_services) | MIN(payback_period_total) |
| Monthly Payment | month10.payment | month10.payment | month10.total_payment_con_iva | month10.total_payment_con_iva |
| APR | month5.annual_rate * 100 | same | same | same |

**Edge case:** If expense is 0, Gross Margin returns 0 (not infinity or NaN).

**Edge case:** Payback period defaults to 9999 if cumulative NII never becomes non-negative.

---

## 20. Interest Rate Calculation (Post-Quote)

**Source code:** `calculate-interest/index.ts` and `calculate-interest/utils/financial-calculations.ts`

This runs after quote creation to calculate the precise interest rate and amortization schedule for the stored cashflows.

**IVA removal:** All amounts from the database have IVA added. Before calculations:
```
removeIVA(amount, taxRate) = amount / (1 + taxRate)
```

**Monthly payment extraction (without IVA):**
```
totalAmountWithoutIVA = removeIVA(
    monthly_payment + down_payment + asset_transfer_value,
    taxRate
)
```

**Cash flow construction:**
```
Month 0: -retailPrice (initial investment, negative)
Months 1+: totalAmountWithoutIVA for each month
```

The result (monthly interest rate) is stored back in `quotes.monthly_interest_rate`, and the calculated amortization (interest/principal split per month) is stored in the `monthly_cash_flows` records.

---

## 21. Down Payment Calculation

**Source code:** quote-generator/index.ts:167-169

```
IF eventData.down_payment_amount_sin_iva is provided:
    downPaymentAmount = eventData.down_payment_amount_sin_iva
ELSE:
    downPaymentAmount = finalRetailWithCommission * estimates.down_payment_1_percentage
```

If neither is available, defaults to 0.

In Manual Mode (NII table):
```
downPaymentAmount = retailPrice * down_payment_percentage
(default down_payment_percentage = 0.03, i.e., 3%)
```

---

## 22. Purchase Option

**Source code:** quote-generator/index.ts:174-175 and quote-calculator.ts:207

The purchase option is the amount the client pays at the end of the lease to own the equipment.

```
opDeCompPct = eventData.op_de_comp_pct (default: 0.01, i.e., 1%)
opDeCompSinIva = finalRetailWithCommission * opDeCompPct
```

In the NII table:
```
purchaseOption = retailPrice * purchase_option_percentage (default: 0.01)
Applied to the final month only.
```

---

## 23. Quote Reference Generation

**Source:** Frontend QuoteWizardStep4Quote.tsx

```
quoteNumber = (number of existing quotes for this project) + 1
quoteReference = "{estimateReference}-{quoteNumber.padStart(2, '0')}"
```

Example: If estimate reference is "CLI-01-01-01" and this is the first quote:
```
quoteReference = "CLI-01-01-01-01"
```

---

## 24. Sale Type Determination

**Source:** Frontend QuoteWizardStep4Quote.tsx

```
IF down_payment_percentage >= 1.0:
    saleType = "Contado"     (cash/upfront payment)
ELSE:
    saleType = "Financiado"  (financed/leased)
```

A "Contado" sale means the client pays 100% upfront — no monthly payments, no interest, no amortization.
