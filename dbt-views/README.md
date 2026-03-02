# DBT Views Reference

This document describes every dbt model across the two dbt projects in this repository. It is intended as a quick-reference wiki for engineers and analysts who need to understand what data is available, where it comes from, and how it is calculated.

---

## Projects Overview

There are two dbt projects:

| Project | Location | Data Source | Purpose |
|---------|----------|-------------|---------|
| **main** | `dbt/main/` | Static Supabase tables (`public.*`) | Application views, analytics, investor reports, AI/RAG |
| **albedo** | `dbt/albedo/` | Quickbase JSON (`public.quickbase_raw_data`) | Legacy financial calculations and accounting |

Both projects output to separate schemas to avoid collisions.

---

## Main Project (`dbt/main/`)

### Output Schemas

| Schema | Purpose | Consumers |
|--------|---------|-----------|
| `app` | Application-facing views | Quotes App frontend, Supabase real-time |
| `analytics` | Core analytics models | Internal dashboards, investor reports |
| `data_chat` | RAG/AI-optimized views | AI chatbot, natural-language analytics |

### Architecture

```
Raw tables (public.*)
  |
  +-- stg_*  (staging: filter soft-deletes, pick signed records)
  |
  +-- int_*  (intermediate: joins, currency conversion, aggregation)
  |
  +-- mart_* (marts: business-facing reports)
  |
  +-- mv_* / v_* (app + data_chat: thin wrappers for consumers)
```

---

### Application Views (`app` schema)

These are the views the Quotes App frontend queries directly.

#### `mv_projects_v1`

Thin wrapper over `int_projects_mega_view`. Returns one row per signed project with all metadata and financial aggregates.

**Key columns**: project_reference, client_name, country, department, municipality, provider, project_date, project_status, duration, grace_period, monthly_interest_rate_percent, total_value, gross_investment, total_finance_income, total_insurance, total_maintenance, total_legal_costs, retail_price, project_cost.

#### `mv_active_projects_v1`

Thin wrapper over `int_active_projects_mega_view`. Same shape as `mv_projects_v1` but filtered to projects that do **not** have a signed contract yet (pipeline projects). Adds `quotes_count` and `estimates_count`.

#### `mv_monthly_cash_flows_v1`

One row per payment period per project. Includes the raw payment components plus computed tax and nullification flags.

**Source**: `public.monthly_cash_flows` joined with projects, countries, quotes, and estimates.

**Key columns**:
- `project_reference`, `project_currency`, `payment_date`, `payment_number`
- Payment components (all in project currency): `down_payment`, `asset_transfer_value`, `legal_costs`, `monthly_payment`, `insurance_payment`, `maintenance_payment`, `interest_payment`, `principal_payment`, `remaining_principal`
- `quote_id`

**Computed columns**:
| Column | Formula |
|--------|---------|
| `total_to_be_paid_without_iva` | down_payment + asset_transfer + legal_costs + monthly_payment + insurance + maintenance |
| `iva_to_be_paid` | total_to_be_paid_without_iva × tax_rate |
| `total_to_be_paid_with_iva` | total_to_be_paid_without_iva × (1 + tax_rate) |
| `is_nullified` | TRUE when the estimate was addended and this payment_date falls after the addendum date |

---

### Data Chat Views (`data_chat` schema)

Currency-specific views designed for AI/RAG consumption. Each includes full project context, location metadata, and impact indicators so the AI can answer questions without extra joins.

#### `v_projects_gtq` / `v_projects_usd`

One row per signed project with all financial totals converted to GTQ (or USD).

**Key financial columns**: total_contract_value, gross_investment, total_finance_income, total_insurance_income, total_legal_costs, retail_price, project_cost, monthly_solar_savings.

**Key context columns**: project_reference, project_name, country_name, department_name, municipality_name, client_legal_name, project_status, sale_type, first_payment_date, payment_duration_months.

**Impact indicators**: women_led, non_profit, rural_area, youth_led, educational_institution, impoverished_area, poverty percentile flags.

#### `v_monthly_cash_flows_gtq` / `v_monthly_cash_flows_usd`

One row per payment period per project, with all amounts converted to GTQ (or USD). Same context enrichment as the project views — location, client, impact indicators attached to every row.

---

### Staging Models (`stg_*`)

These filter raw tables down to usable records.

| Model | Logic |
|-------|-------|
| `stg_projects_not_deleted` | `deleted_at IS NULL` |
| `stg_estimates_not_deleted` | `deleted_at IS NULL` |
| `stg_quotes_not_deleted` | `deleted_at IS NULL` |
| `stg_signed_quotes` | `status = 'signed'` |
| `stg_signed_estimates` | Estimates linked to a signed quote |
| `stg_signed_projects` | Projects that have at least one signed quote |
| `stg_active_projects` | Projects with **no** signed quote |
| `stg_signed_monthly_cash_flows` | Cash flows from signed, non-addended quotes |

---

### Intermediate Models (`int_*`)

#### Core Hub: `int_projects_mega_view`

The central model that most downstream views reference. One row per signed project.

**Joins**: projects + signed monthly cash flows (aggregated) + original estimates + original quotes + clients + countries + departments + municipalities + providers.

**Aggregated columns from cash flows**:
| Column | Formula |
|--------|---------|
| `total_value` | SUM(monthly_payment + down_payment + asset_transfer + insurance + legal + maintenance) |
| `gross_investment` | SUM(monthly_payment + down_payment + asset_transfer) |
| `total_finance_income` | SUM(interest_payment) |
| `total_insurance` | SUM(insurance_payment) |
| `total_maintenance` | SUM(maintenance_payment) |
| `total_legal_costs` | SUM(legal_costs) |
| `duration` | COUNT(DISTINCT payment_date) |
| `grace_period` | COUNT(DISTINCT payment_date WHERE monthly_payment = 0) |
| `project_status` | 'Finalizado' if current_date > last payment date, else 'Activo' |
| `monthly_interest_rate_percent` | monthly_interest_rate × 100 |

#### Currency Variants

- `int_projects_mega_view_gtq` — adds GTQ-converted totals (total_contract_value_gtq, remaining_payment_gtq, avg_monthly_payment_gtq)
- `int_projects_mega_view_usd` — same in USD

#### Original Record Models

- `int_original_estimates` — first estimate per project (addendum_number = 0 or NULL)
- `int_original_quotes` — original signed quote per project, with `project_date` derived from contract_signing_date

#### Currency Conversion Models

- `int_estimates_convert_currencies` — converts estimate amounts (retail_price, project_cost, solar_savings, etc.) to GTQ and USD using exchange rates at project_start_date
- `int_monthly_cash_flows_convert_currencies` — **materialized as table** (heavy, reused everywhere). Converts all 12 cash flow component columns to project_currency, USD, and GTQ versions using exchange rates at each payment_date

#### Cash Flow Adjustments

- `int_monthly_cash_flows_month_0_adjustment` — corrects first-payment-date anomalies
- `int_monthly_cash_flows_month_0_adjustment_converted_currencies` — adjusted cash flows with currency conversion

#### Financial Analysis

- `int_first_payment_dates` — first payment date and month number per project
- `int_remaining_principal_by_project_month_gtq` / `_usd` — outstanding principal per month
- `int_long_term_insurance_by_project_month_gtq` — monthly insurance breakdown
- `int_long_term_interest_by_project_month_gtq` — monthly interest breakdown
- `int_official_accounts_receivable_by_project_gtq` / `_usd` — accounts receivable by project
- `int_static_cash_flows_for_irr_calculation` — structured for IRR computation
- `int_month_spine` — calendar month spine for time-series joins
- `int_active_projects_mega_view` — projects without signed contracts + quote/estimate counts

---

### Mart Models (`mart_*`)

#### Finance

**`mart_finance_historic_accounting_report`** — Yearly accounting breakdown per project (2022, 2023, 2024). Shows interest payments, total payments, legal costs, and insurance by year. Includes a TOTAL summary row. All amounts in GTQ.

**`mart_finance_projections_guatemala`** — Guatemala-specific financial projections.

**`mart_finance_financial_projections_december_2025`** — Snapshot of financial projections.

#### Impact

**`mart_impact_overall_summary`** — Organization-wide impact KPIs in a single row:
- number_of_projects, mw_installed, mwh_production_lifetime, kwh_production_to_date
- co2e_tons_avoided_30_year, equivalent_cars_taken_off_road (CO2e / 4.6), equivalent_sf_sydney_flights (CO2e / 2.6)
- accumulated_savings_to_date_usd, net_savings_30_year_usd
- Impact percentages: women_led, youth_led, rural, educational, non_profit, poverty_impact, financed

**`mart_impact_projects_summary`** — Per-project: kW installed, lifetime electricity production, CO2 avoided, solar savings.

**`mart_impact_quarterly`** — Quarterly impact rollup.

#### Investors

**`mart_investors_portfolio_overview`** — Year-over-year portfolio comparison (2023 / 2024 / 2025) with growth rates. Sections: overall summary, country breakdown (Guatemala, Honduras, El Salvador), currency breakdown. Metrics include project counts by sector/sale-type, sales values, installed capacity, solar savings, and social impact percentages.

**`mart_investors_summary_by_project_gtq`** / **`_usd`** — Financial project summary for investor dashboards.

**`mart_investors_impact_by_project`** — Impact metrics per project for investors.

#### Reconciliation / Exports

**`mart_reconciliation_projects_overview`** — Project list with key financial metrics.

**`mart_reconciliation_eeffs_by_financed_project`** — EEFF (financial statements) export for financed projects.

**`mart_reconciliation_eeffs_by_financed_project_HN_ES_USD`** — EEFF export for Honduras/El Salvador in USD.

**`mart_reconciliation_eeffs_by_paid_up_front_project`** — EEFF export for paid-upfront (cash) projects.

#### Sales

**`mart_sales_by_partner`** — Sales revenue grouped by partner.

#### Zoho Templates

**`mart_templates_projects_template`** — Format for Zoho project imports.

**`mart_templates_cash_flows_template`** — Format for Zoho cash flow imports.

**`mart_templates_cash_flows_header_template`** — Header row for Zoho cash flow imports.

---

## Albedo Project (`dbt/albedo/`)

### Output Schemas

| Schema | Purpose |
|--------|---------|
| `quickbase_app` | Application views from Quickbase data |
| `analytics` | Quickbase-derived analytics |
| `quickbase_frozen` | Incremental snapshots |
| `quickbase_data_chat` | AI/RAG views from Quickbase data |

### Architecture

```
public.quickbase_raw_data (JSON)
  |
  +-- stg_quickbase_* (extract & decode JSON fields)
  |
  +-- stg_calculated_* (derived rates)
  |
  +-- int_* (financial transformations split by project type)
  |     |
  |     +-- Financed projects: interest calc -> payment decomposition -> IVA -> combine
  |     +-- Paid-upfront projects: simpler pipeline -> IVA
  |     +-- Impact: electricity calculations
  |
  +-- UNION -> mart_monthly_cash_flows (all project types)
  |
  +-- mart_* (reports, accounting, projections)
  +-- frozen_* (incremental snapshots)
```

### Staging — Quickbase Extraction (`stg_quickbase_*`)

Each model extracts and decodes one entity type from the JSON blob in `quickbase_raw_data`.

| Model | Entity |
|-------|--------|
| `stg_quickbase__clients` | Client master data |
| `stg_quickbase__countries` | Countries with tax rates |
| `stg_quickbase__departments` | Geographic departments |
| `stg_quickbase__equipment_providers` | Equipment vendors |
| `stg_quickbase__equipment_types` | Solar equipment categories |
| `stg_quickbase__equipments` | Individual equipment specs (watts, degradation, guarantee) |
| `stg_quickbase__estimates` | Cost estimates with currency |
| `stg_quickbase__exchange_rates` | GTQ/USD exchange rates by date |
| `stg_quickbase__municipalities` | Location details with impact indicators |
| `stg_quickbase__monthly_cash_flows` | Raw payment schedules |
| `stg_quickbase__projects` | Project metadata |
| `stg_quickbase__proyectos_en_marcha` | Active projects snapshot |
| `stg_quickbase__quotes` | Contracts/quotes |
| `stg_quickbase__required_equipments` | Equipment per estimate |

#### Calculated Staging

- `stg_calculated__interest_rates` — Derives interest rates for financing calculations.

### Intermediate — Financial Transformations

#### Financed Projects (loan-based)

These models decompose loan payments into interest, principal, and tax components:

| Model | Step |
|-------|------|
| `int_financed_cash_flows` | Base cash flows for financed projects |
| `int_financed_cash_flows_calculate_interest` | Calculate interest component per payment |
| `int_financed_cash_flows_calculate_monthly_payments` | Decompose into principal + interest |
| `int_financed_cash_flows_calculate_iva` | Apply tax to each component |
| `int_financed_cash_flows_combine_columns` | Combine all components into final row |

#### Paid-Upfront Projects (cash sales)

| Model | Step |
|-------|------|
| `int_paid_up_front_cash_flows` | Base cash flows for cash-sale projects |
| `int_paid_up_front_cash_flows_calculate_iva` | Apply tax |
| `int_paid_up_front_cash_flows_gtq` / `_usd` | Currency conversion |

#### Historical & Accounts Receivable

| Model | Purpose |
|-------|---------|
| `int_historic_monthly_cash_flows` | Historical financed cash flows |
| `int_historic_monthly_cash_flows_gtq` / `_usd` | Currency-converted versions |
| `int_accounts_receivable_by_project` | AR tracking per project |
| `int_accounts_receivable_by_project_gtq` | AR in GTQ |
| `int_accounts_receivable_summary_gtq` | AR summary |

#### Other Intermediate

| Model | Purpose |
|-------|---------|
| `int_projects_considered` | Union of financed + paid-upfront project types |
| `int_estimates_remove_iva` | Estimates without tax |
| `int_quotes_remove_iva` | Quotes without tax |
| `int_estimates_corrected_prices_and_costs` | Price/cost adjustments |
| `int_estimates_convert_currencies` | Estimates in GTQ/USD |
| `int_monthly_cash_flows_convert_currencies` | Cash flows in GTQ/USD |
| `int_prepare_and_filter_cash_flows` | Filter and prep cash flows |
| `int_project_electricity_calculations` | Lifetime electricity, CO2, savings per project |
| `int_month_spine` | Calendar month spine |

### Marts

| Model | Purpose |
|-------|---------|
| `mart_monthly_cash_flows` | **Core**: Union of financed + paid-upfront cash flows (all project types) |
| `mart_financial_projections` | Revenue, finance income, insurance, legal by time period |
| `mart_historic_accounting_report` | Annual accounting by project with year breakdowns |
| `mart_cash_flows_for_irr_calculation` | Structured for IRR analysis |
| `mart_paid_up_front_projects_summary` | Summary of cash-sale projects |

### Frozen Snapshots

| Model | Purpose |
|-------|---------|
| `frozen_cash_flows` | Incremental cash flow history |
| `frozen_accounting_report` | Incremental accounting history |

### Snapshots (SCD Type 2)

| Snapshot | Tracks changes to |
|----------|-------------------|
| `snapshot_financial_projections` | Financial projections over time |
| `snapshot_monthly_interest_rates` | Interest rate changes |
| `snapshot_accounting_report` | Accounting report changes |
| `snapshot_monthly_cash_flows` | Cash flow changes |
| `info_projects_snaphot` | Project metadata changes |

---

## Macros

Shared SQL functions used across models.

| Macro | Purpose | Example |
|-------|---------|---------|
| `convert_to_quetzales` | Convert any currency amount to GTQ using exchange rate | `convert_to_quetzales('USD', er.gtq_rate, amount)` |
| `convert_to_dollars` | Convert any currency amount to USD | `convert_to_dollars('GTQ', er.gtq_rate, amount)` |
| `convert_to_currency` | Generic currency conversion | Used internally by the above |
| `get_months_between_dates` | Month difference between two dates | Duration calculations |
| `get_production_multiplier` | kWh production multiplier for energy calcs | Impact models |
| `generate_custom_schema` | Custom schema naming for dbt | dbt_project.yml |

---

## Key Relationships

```
projects (1) ---> (N) estimates ---> (N) quotes ---> (N) monthly_cash_flows
    |                    |                |
    +-- client_id        +-- provider_id  +-- status = 'signed'
    +-- country_id       +-- addendum_number
    +-- department_id
    +-- municipality_id
```

- A **project** has multiple **estimates** (revisions/addenda)
- Each estimate can have multiple **quotes** (pricing versions)
- Only **signed** quotes generate **monthly_cash_flows**
- The **original** estimate (addendum_number = 0) and its signed quote are used for project-level reporting
- When an estimate is **addended**, its old cash flows are marked `is_nullified = TRUE`
