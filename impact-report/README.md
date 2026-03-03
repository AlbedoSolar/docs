# Impact Report — Technical Reference

This document describes the three dbt models that power Albedo's impact reporting: per-project calculations, the organization-wide summary, and the quarterly rollup. It covers the data flow, every calculation formula, key assumptions, and known edge cases.

**Location**: `dbt/main/models/models_on_static_tables/marts/impact/`

---

## Data Flow

```
Raw tables
  |
  +-- stg_signed_quotes           (status = 'signed')
  +-- stg_signed_projects         (projects with at least one signed quote)
  +-- stg_signed_estimates        (estimates linked to a signed quote)
  |
  +-- int_original_quotes         (original signed quote per project, derives project_date)
  +-- int_original_estimates      (addendum_number = 0 or NULL)
  +-- int_estimates_convert_currencies  (all estimate amounts in GTQ + USD)
  |
  +-- estimate_equipment + equipments + equipment_types  (panel specs, degradation)
  +-- countries.grid_factor       (CO2 emissions factor by country)
  |
  +-- mart_impact_projects_summary    <-- one row per project
  |       |
  |       +-- mart_impact_overall_summary   <-- one row, org-wide KPIs
  |
  +-- mart_impact_quarterly           <-- quarterly client/project rollup
```

---

## Model 1: `mart_impact_projects_summary`

**Grain**: One row per signed project (keyed by `estimate_id`).

**Purpose**: Calculate installed capacity, lifetime electricity production, CO2 avoided, and solar savings for each project. Also compares Albedo's calculated savings against the partner's estimate.

### Input Data

| Source | What it provides |
|--------|-----------------|
| `int_estimates_convert_currencies` | Estimate fields with USD-converted prices: `price_of_electricity_per_kwh_usd`, `monthly_solar_savings_partner_estimate_usd` |
| `int_original_quotes` | `project_date`, `contract_signing_date`, `estimate_id` |
| `estimate_equipment` | Which equipment is on each estimate (amount, equipment_id) |
| `equipments` | Per-equipment specs: `potential_watts`, `first_year_production_loss_percentage`, `annual_depreciation_percentage`, `production_guarantee_years` |
| `equipment_types` | Filters to `name = 'Panel Solar'` (only solar panels count for production) |
| `countries` | `grid_factor` — CO2 emissions per MWh for the country's electric grid |
| `stg_signed_projects` | Social impact flags: `women_led`, `non_profit`, `rural_area`, `youth_led`, `educational_institution`, `impoverished_area` |

### Calculation Pipeline (CTEs)

#### 1. Calendar Spine (`months`)

Generates every month from January 2018 to December 2100.

#### 2. Lifetime Months (`lifetime_months`)

Cross-joins the calendar spine with `int_original_quotes` to enumerate every month from each project's `project_date` onward. Derives:

| Column | Formula |
|--------|---------|
| `payment_number` | Months between `project_date` and `payment_date`, plus 1 |
| `year_number` | `CEIL(payment_number / 12)` — year 1, 2, 3, ... |
| `year_index` | `MAX(0, year_number - 1)` — 0-indexed year for escalation |

#### 3. Equipment Calculations (`equipment_calculations`)

Sums installed watts per estimate, counting only `Panel Solar` equipment:

```
watts_installed = SUM(estimate_equipment.amount * equipments.potential_watts)
WHERE equipment_types.name = 'Panel Solar'
```

#### 4. Estimate Calculations (`estimate_calculations`)

Derives the production multiplier and baseline monthly production/savings:

| Column | Formula |
|--------|---------|
| `production_multiplier` | If partner estimate exists: `partner_monthly_kwh / watts_installed`. Otherwise: `0.13` (Guatemala default based on sunlight hours) |
| `monthly_solar_production_kwh_estimate` | `watts_installed * production_multiplier` |
| `solar_savings_estimate` | `monthly_solar_production_kwh_estimate * price_of_electricity_per_kwh_usd` |

> **Note on solar tax rate (VAD)**: The ~25% VAD tax on net metering credits is currently **commented out**. It only applies to excess production sent to the grid, not direct consumption. Without consumption-vs-export split data, applying it would be inaccurate.

#### 5. Monthly Equipment Production (`monthly_equipment_production`)

Calculates per-equipment, per-month electricity production and savings. This is the core degradation engine:

**Production (kWh)**:
| Period | Formula |
|--------|---------|
| Year 1 (months 1-12) | `amount * potential_watts * production_multiplier` |
| Years 2+ (within guarantee) | `amount * potential_watts * production_multiplier * (1 - first_year_loss) * (1 - annual_depreciation)^(year - 2)` |
| Beyond guarantee period | `0` |

**Savings (USD)**:
| Period | Formula |
|--------|---------|
| Year 1 | `production * price_per_kwh_usd` |
| Years 2+ (within guarantee) | `degraded_production * price_per_kwh_usd * 1.03^year_index` |
| Beyond guarantee period | `0` |

The `1.03^year_index` factor assumes a **3% annual escalation** in electricity prices.

#### 6. Monthly Electricity Production (`monthly_electricity_production`)

Aggregates across all equipment for a given estimate and month:
```
monthly_electricity_production = SUM(monthly_equipment_production)
monthly_solar_savings = SUM(monthly_equipment_solar_savings)
```

#### 7. Partner Estimate Comparison (`partner_estimate_monthly_savings`)

Separately calculates what savings would be if using the partner's estimate directly (with the same degradation and escalation logic), allowing comparison between Albedo's calculated value and the partner's claim.

### Output Columns

| Column | Description |
|--------|-------------|
| `estimate_id` | Primary key |
| `project_name` | Project name |
| `project_reference` | e.g. `ALB-001-01` |
| `contract_signing_date` | When the quote was signed |
| `total_kw_installed` | `watts_installed / 1000`, rounded to 2 decimals |
| `monthly_solar_production_kwh_estimate` | Month-0 expected production (before degradation) |
| `electricity_production_to_date` | Sum of monthly production for months before today |
| `total_electricity_production` | Sum of monthly production across entire guarantee window |
| `CO2_emissions_avoided_tons` | `total_electricity_production / 1000 * grid_factor` |
| `accumulated_savings_to_date` | Sum of monthly savings for months before today |
| `average_monthly_solar_savings` | Average monthly savings (excluding zero-months beyond guarantee) |
| `total_solar_savings` | Sum of monthly savings across entire guarantee window |
| `partner_monthly_savings_estimate_usd` | Partner's claimed monthly savings |
| `monthly_savings_difference_vs_partner` | `partner_estimate - our_average_monthly` |
| `partner_estimate_inflation_percent` | How much higher the partner's estimate is vs ours (positive = inflated) |
| `total_solar_savings_from_partner_estimate` | What total savings would be using partner estimate with degradation |
| Social impact flags | `women_led`, `non_profit`, `rural_area`, `youth_led`, `educational_institution`, `impoverished_area` |

### Filtering

- Only projects where `watts_installed > 0` (must have solar panels)
- Only signed quotes with `contract_signing_date` present
- Ordered by `contract_signing_date DESC` (newest first)

### Debug Mode

The model supports a debug variable to inspect intermediate CTEs:
```bash
dbt run --select mart_impact_projects_summary --vars '{"debug_cte": "equipment_calculations"}'
```

Valid values: `equipment_calculations`, `estimate_calculations`, `monthly_equipment_production`, `monthly_electricity_production`, `final` (default).

---

## Model 2: `mart_impact_overall_summary`

**Grain**: Single row — organization-wide impact KPIs.

**Purpose**: Aggregates `mart_impact_projects_summary` into the headline numbers used for the impact dashboard and investor presentations.

### Filter

Only includes projects with `contract_signing_date < '2026-01-01'`.

### Output Columns

| Column (aliased) | Formula |
|-------------------|---------|
| `# of Projects` | `COUNT(DISTINCT project_id)` |
| `MW Installed` | `SUM(total_kw_installed) / 1000` |
| `MWh to be produced - Project Useful Life` | `SUM(total_electricity_production) / 1000` |
| `CO2e to be avoided - Project Useful Life` | `SUM(CO2_emissions_avoided_tons)` |
| `Total Electricity Production To Date` | `SUM(electricity_production_to_date)` (kWh) |
| `Financed Projects (%)` | `% of projects where sale_type = 'Financiado'` |
| `Accumulated Savings to Date (USD)` | `SUM(accumulated_savings_to_date)` |
| `Net Savings - 30-Year Project Useful Life (USD)` | `SUM(total_solar_savings)` |
| `Equivalent number of cars taken off the road` | `CO2e_tons / 4.6` |
| `Equivalent number of flights from San Francisco to Sydney` | `CO2e_tons / 2.6` |

**CO2 equivalence conversion factors** (from an internal Asana task by Alex):
- Average car emits **4.6 metric tons CO2/year**
- Average SF-to-Sydney flight emits **2.6 metric tons CO2/passenger**

### Social Impact Percentages

Each impact category uses the same pattern:

```
percentage = COUNT(projects where flag = 'Yes')
           / COUNT(projects where flag IS NOT NULL AND flag != 'Not Sure')
           * 100
```

Projects with `NULL` or `'Not Sure'` are **excluded from the denominator** (not counted as non-impact).

| Column | Flag field |
|--------|-----------|
| `Women-led Projects (%)` | `women_led` |
| `Educational Institutions (%)` | `educational_institution` |
| `Non-profits (%)` | `non_profit` |
| `Rural Projects (%)` | `rural_area` |
| `Youth-led Projects (%)` | `youth_led` |
| `Poverty Impact (%)` | `impoverished_area` |
| `Impact Projects (%)` | Any of the above = 'Yes' |

---

## Model 3: `mart_impact_quarterly`

**Location**: `dbt/main/models/models_on_static_tables/marts/internal_reports/mart_impact_quarterly.sql`

**Grain**: Multiple rows (one per metric label), with columns for Q4 2024, Q1 2025, Q2 2025, Q3 2025.

**Source**: `int_projects_mega_view` (not `mart_impact_projects_summary`).

**Purpose**: Quarterly client and project counts with social impact breakdowns. Used for internal reports tracking growth over time.

### Metrics (rows)

| Row Label | What it counts |
|-----------|---------------|
| Total Clients | `COUNT(DISTINCT client_id)` cumulative up to quarter end |
| Total Clients - Non-Woman Led | Same, filtered `women_led = 'No'` |
| Total Clients - Woman Led | Same, filtered `women_led = 'Yes'` |
| New Clients | `COUNT(DISTINCT client_id)` within the quarter window |
| Total Projects | `COUNT(*)` cumulative |
| New Projects | `COUNT(*)` within the quarter |
| Total/New Projects - Women Led | Filtered `women_led = 'Yes'` |
| Total/New Projects - Youth Led | Filtered `youth_led = 'Yes'` |
| Total/New Projects - Rural | Filtered `rural_area = 'Yes'` |
| Total/New Projects - Education | Filtered `educational_institution = 'Yes'` |
| Total/New Projects - Impoverished Area | Filtered `impoverished_area = 'Yes'` |

**Note**: "Total" rows use cumulative filters (`project_date < quarter_end`), while "New" rows use window filters (`project_date BETWEEN quarter_start AND quarter_end`).

---

## Key Assumptions & Constants

| Assumption | Value | Source |
|-----------|-------|--------|
| Default production multiplier (no partner estimate) | 0.13 kWh/W/month | Guatemala sunlight hours |
| Electricity price escalation rate | 3% per year | Hardcoded |
| CO2 per car per year | 4.6 metric tons | Internal Asana task (Alex) |
| CO2 per SF-Sydney flight per passenger | 2.6 metric tons | Internal Asana task (Alex) |
| Solar tax rate (VAD) | Currently **disabled** (commented out) | Needs consumption-vs-export split data |
| Production degradation | Year 1: `first_year_production_loss_percentage`, Year 2+: compounds `annual_depreciation_percentage` | Per-equipment in `equipments` table |
| Production guarantee window | `production_guarantee_years` per equipment | Panel-specific, stored in `equipments` table |
| Impact flag exclusions | `NULL` and `'Not Sure'` excluded from denominator | Prevents skewing percentages |

---

## Dependency Graph

```
equipments ──────────────┐
equipment_types ─────────┤
estimate_equipment ──────┤
                         ├──> mart_impact_projects_summary ──> mart_impact_overall_summary
int_estimates_convert_currencies ─┤
int_original_quotes ─────┤
stg_signed_projects ─────┤
countries (grid_factor) ──┘

int_projects_mega_view ──> mart_impact_quarterly
```
