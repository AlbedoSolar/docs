# Total Solar Savings — Detailed Calculation Reference

This document provides a deep technical breakdown of how **total solar savings** (lifetime USD savings from solar energy) are calculated in the impact report. It supplements the [Impact Report README](./README.md) with worked examples, edge-case handling, and step-by-step formula derivations.

**Source model**: `dbt/main/models/models_on_static_tables/marts/impact/mart_impact_projects_summary.sql`

---

## Overview

Total solar savings represents the cumulative financial benefit a project receives over its equipment guarantee period (typically 30 years) by generating electricity from solar panels instead of purchasing it from the grid. The calculation accounts for:

- **Panel degradation** — solar panels lose efficiency over time
- **Electricity price escalation** — grid electricity prices rise ~3% annually
- **Equipment guarantee window** — production is set to zero after the guarantee expires

The final metric (`total_solar_savings`) is the sum of monthly savings across every month in the guarantee window, expressed in USD.

---

## Step-by-Step Calculation

### Step 1: Determine Installed Watts

Sum the total wattage of all solar panels on the estimate:

```
watts_installed = SUM(estimate_equipment.amount * equipments.potential_watts)
WHERE equipment_types.name = 'Panel Solar'
```

Only equipment of type **"Panel Solar"** is counted. Inverters, batteries, and other equipment are excluded.

**Example**: 10 panels × 550W each = **5,500 W installed**

---

### Step 2: Derive the Production Multiplier

The production multiplier converts installed watts into expected monthly kWh output:

```
IF partner provides a monthly production estimate AND it is > 0 AND watts_installed > 0:
    production_multiplier = monthly_solar_production_kwh_partner_estimate / watts_installed
ELSE:
    production_multiplier = 0.13
```

The default `0.13 kWh/W/month` is calibrated for Guatemala based on average solar insolation and system efficiency losses.

**Example with partner estimate**: Partner says 715 kWh/month for 5,500 W → multiplier = 715 / 5500 = **0.13 kWh/W/month**

**Example without partner estimate**: Falls back to **0.13 kWh/W/month**

---

### Step 3: Calculate Baseline Monthly Production (Year 1)

```
monthly_production_year1 = amount * potential_watts * production_multiplier
```

This is per-equipment. For the full project in month 1:

```
project_monthly_production_year1 = watts_installed * production_multiplier
```

**Example**: 5,500 W × 0.13 = **715 kWh/month**

---

### Step 4: Apply Panel Degradation (Years 2+)

Solar panels degrade over time. The model uses a two-phase degradation approach sourced from each equipment record in the `equipments` table:

| Parameter | Description | Typical Value |
|-----------|-------------|---------------|
| `first_year_production_loss_percentage` | One-time loss applied after Year 1 | ~2-3% |
| `annual_depreciation_percentage` | Ongoing annual degradation from Year 2 onward | ~0.5% |
| `production_guarantee_years` | Window beyond which production is assumed to be 0 | ~30 years |

**Monthly production formula by period**:

| Period | Formula |
|--------|---------|
| **Year 1** (months 1–12) | `amount * potential_watts * production_multiplier` |
| **Year 2** (months 13–24) | `amount * potential_watts * production_multiplier * (1 - first_year_loss)` |
| **Year N** (N ≥ 2, within guarantee) | `amount * potential_watts * production_multiplier * (1 - first_year_loss) * (1 - annual_depreciation)^(N - 2)` |
| **Beyond guarantee** (month > guarantee_years × 12) | `0` |

The `year_number` is derived as `CEIL(payment_number / 12)`, where `payment_number` counts months from the project start date.

**Worked example** (5,500 W system, multiplier 0.13, 2% first-year loss, 0.5% annual depreciation):

| Year | Monthly Production (kWh) | Calculation |
|------|-------------------------|-------------|
| 1 | 715.00 | 5500 × 0.13 |
| 2 | 700.70 | 5500 × 0.13 × (1 - 0.02) |
| 3 | 697.20 | 5500 × 0.13 × (1 - 0.02) × (1 - 0.005)^1 |
| 5 | 690.22 | 5500 × 0.13 × (1 - 0.02) × (1 - 0.005)^3 |
| 10 | 673.09 | 5500 × 0.13 × (1 - 0.02) × (1 - 0.005)^8 |
| 20 | 640.49 | 5500 × 0.13 × (1 - 0.02) × (1 - 0.005)^18 |
| 30 | 609.33 | 5500 × 0.13 × (1 - 0.02) × (1 - 0.005)^28 |
| 31+ | 0.00 | Beyond guarantee window |

---

### Step 5: Calculate Monthly Savings (USD)

Monthly savings multiply the degraded production by the electricity price, with a 3% annual price escalation:

**Monthly savings formula by period**:

| Period | Formula |
|--------|---------|
| **Year 1** | `monthly_production * price_per_kwh_usd` |
| **Year N** (N ≥ 2, within guarantee) | `degraded_monthly_production * price_per_kwh_usd * 1.03^(N - 1)` |
| **Beyond guarantee** | `0` |

The `year_index` used for escalation is `MAX(0, year_number - 1)`, so Year 1 has index 0 (no escalation), Year 2 has index 1 (3% escalation), etc.

**Worked example** (continuing above, electricity price = $0.18/kWh):

| Year | Monthly Production (kWh) | Price Escalation | Effective $/kWh | Monthly Savings (USD) |
|------|-------------------------|------------------|-----------------|----------------------|
| 1 | 715.00 | 1.03^0 = 1.000 | $0.180 | $128.70 |
| 2 | 700.70 | 1.03^1 = 1.030 | $0.185 | $129.87 |
| 5 | 690.22 | 1.03^4 = 1.126 | $0.203 | $139.89 |
| 10 | 673.09 | 1.03^9 = 1.305 | $0.235 | $158.12 |
| 20 | 640.49 | 1.03^19 = 1.754 | $0.316 | $202.23 |
| 30 | 609.33 | 1.03^29 = 2.357 | $0.424 | $258.33 |

Notice that despite degradation reducing production, the 3% annual price escalation outpaces it, so monthly savings **increase** over time in nominal USD.

---

### Step 6: Aggregate to Total Solar Savings

```
total_solar_savings = SUM(monthly_solar_savings)
    FOR ALL months WHERE payment_number BETWEEN 1 AND (production_guarantee_years * 12)
```

This sums every monthly savings value across the entire guarantee window.

**Worked example** (30-year guarantee = 360 months):

Using the parameters above, the approximate total would be:

- Year 1 total: 12 × $128.70 ≈ **$1,544**
- Year 2 total: 12 × $129.87 ≈ **$1,558**
- ...accumulating with price escalation...
- Year 30 total: 12 × $258.33 ≈ **$3,100**
- **30-year total ≈ $66,000–$70,000** (varies with exact degradation/escalation compounding)

---

### Step 7: Derived Metrics

From the per-project `total_solar_savings`, the overall summary model (`mart_impact_overall_summary`) aggregates across all projects:

| Metric | Formula |
|--------|---------|
| `Net Savings - 30-Year Project Useful Life (USD)` | `SUM(total_solar_savings)` across all signed projects |
| `Accumulated Savings to Date (USD)` | `SUM(accumulated_savings_to_date)` — same logic but only months before today |

---

## Multi-Equipment Handling

When a project has multiple sets of solar panels (e.g., different wattages or brands added via addendum), each equipment line is calculated independently in `monthly_equipment_production`, then summed per month in `monthly_electricity_production`:

```
For each month:
    monthly_electricity_production = SUM(monthly_equipment_production) across all Panel Solar equipment
    monthly_solar_savings = SUM(monthly_equipment_solar_savings) across all Panel Solar equipment
```

Each equipment row can have its own `potential_watts`, `first_year_production_loss_percentage`, `annual_depreciation_percentage`, and `production_guarantee_years`. The guarantee window is evaluated per-equipment — one equipment type might stop producing at year 25 while another continues to year 30.

---

## Partner Estimate Comparison

The model separately computes what total savings would be if using the partner's (installer's) monthly savings estimate instead of Albedo's calculated value. This uses the same degradation and escalation logic:

```
partner_monthly_savings =
    CASE
        Year 1:    partner_estimate_usd
        Year N:    partner_estimate_usd * (1 - avg_first_year_loss) * (1 - avg_annual_depreciation)^(N-2) * 1.03^(N-1)
        Beyond:    0
    END
```

Output columns for comparison:

| Column | Meaning |
|--------|---------|
| `partner_monthly_savings_estimate_usd` | Raw partner claim (no degradation) |
| `average_monthly_solar_savings` | Albedo's calculated average monthly savings |
| `monthly_savings_difference_vs_partner` | `partner_estimate - our_average` (positive = partner claims more) |
| `partner_estimate_inflation_percent` | How much the partner's estimate exceeds Albedo's (as a percentage) |
| `total_solar_savings_from_partner_estimate` | Lifetime savings using partner's baseline with degradation/escalation |

---

## Note on Solar Tax Rate (VAD)

The solar tax rate (VAD, approximately 25%) is currently **commented out** in the calculation. This tax applies only to net metering credits — electricity sent back to the grid — not to electricity consumed directly by the building. Without data distinguishing self-consumption from grid export, applying the tax would be inaccurate. The savings figures therefore represent a **gross savings** estimate before any net metering taxation.

---

## CO2 Emissions Avoided

While not a financial savings metric, CO2 avoidance is derived directly from total electricity production:

```
CO2_emissions_avoided_tons = total_electricity_production / 1000 * grid_factor
```

Where `grid_factor` comes from the `countries` table and represents metric tons of CO2 emitted per MWh of grid electricity in that country. The overall summary further converts this into equivalences:

| Equivalence | Formula |
|-------------|---------|
| Cars taken off the road | `CO2e_tons / 4.6` (avg car = 4.6 t CO2/year) |
| SF-to-Sydney flights avoided | `CO2e_tons / 2.6` (avg flight = 2.6 t CO2/passenger) |

---

## Summary of Key Constants

| Constant | Value | Notes |
|----------|-------|-------|
| Default production multiplier | 0.13 kWh/W/month | Fallback for Guatemala when no partner estimate |
| Electricity price escalation | 3% per year | Hardcoded as `POWER(1.03, year_index)` |
| CO2 per car per year | 4.6 metric tons | Source: internal Asana task |
| CO2 per SF-Sydney flight | 2.6 metric tons/passenger | Source: internal Asana task |
| Solar tax rate (VAD) | Disabled (~25%) | Needs consumption-vs-export data to enable |
