# Finance Models — Technical Reference

> **Looking for the Debita portfolio reports (loan tape, payment tape, snapshots)?** See [debita-portfolio-reports.md](debita-portfolio-reports.md).

This document describes the dbt models that power Albedo's financial reporting: monthly cash flows, interest amortization, accounts receivable, financial projections, accounting reports, and reconciliation. It covers the data flow, calculation formulas, key assumptions, and how the two sale types (financed vs. paid upfront) are handled.

**Locations**:
- Core models: `dbt/albedo/models/marts/finance/` and `dbt/albedo/models/intermediate/finance/`
- Reporting layer: `dbt/main/models/models_on_static_tables/marts/finance/` and `marts/reconciliation/`

---

## Data Flow

```
QuickBase raw tables (monthly_cash_flows, quotes, estimates, projects)
  |
  +-- stg_quickbase__* staging models (filter deleted records, normalize fields)
  |
  +-- int_financed_cash_flows / int_paid_up_front_cash_flows
  |     (split by sale_type: 'Financiado' vs 'Contado')
  |
  +-- int_financed_cash_flows_calculate_iva
  |     (reverse-engineer IVA from QB's combined payment field)
  |
  +-- int_financed_cash_flows_calculate_monthly_payments
  |     (extract monthly payment from total client charge)
  |
  +-- int_financed_cash_flows_combine_columns   [TABLE]
  |     (split QB's combined field into down_payment / asset_transfer / legal)
  |
  +-- int_financed_cash_flows_calculate_interest   [TABLE]
  |     (recursive amortization schedule)
  |
  +-- int_historic_monthly_cash_flows
  |     (final financed cash flows in project currency)
  |
  +-- int_historic_monthly_cash_flows_gtq / int_paid_up_front_cash_flows_gtq
  |     (currency conversion to GTQ)
  |
  +-- int_monthly_cash_flows_convert_currencies   [TABLE]
  |     (multi-currency: project currency, USD, GTQ)
  |
  +-- int_accounts_receivable_by_project → _gtq → _summary_gtq
  |
  +-- MARTS:
        +-- mart_monthly_cash_flows             (combined financed + upfront)
        +-- mart_financial_projections          (monthly P&L-style pivot)
        +-- mart_historic_accounting_report     (per-project year-by-year summary)
        +-- mart_cash_flows_for_irr_calculation (IRR input format)
        +-- mart_paid_up_front_projects_summary (upfront project overview)
        +-- mart_reconciliation_eeffs_*         (Zoho-integrated accounting reconciliation)
```

---

## Sale Types

Albedo handles two fundamentally different sale structures, and most of the finance model complexity comes from keeping them separate through the pipeline and merging them only at the mart layer.

| | Financed (`Financiado`) | Paid Upfront (`Contado`) |
|---|---|---|
| Payment structure | Monthly over contract term (e.g. 36–60 months) | Single or few upfront payments |
| Interest | Yes — amortized monthly | None |
| Down payment / asset transfer | Split across first 4 payments | Not applicable |
| Principal tracking | Remaining principal calculated recursively | Not applicable |
| Grace period | Supported (zero-payment months) | Not applicable |

---

## Core Calculation Pipeline (Financed Projects)

### 1. IVA Extraction

**Model**: `int_financed_cash_flows_calculate_iva`

QuickBase stores amounts with IVA baked in. Since QB's own IVA calculation isn't reliable, dbt strips IVA from each component using the country's tax rate:

```
component_without_iva = component_with_iva / (1 + tax_rate)
```

Applied to: down payment, legal costs, insurance, insurance costs, maintenance, commission.

### 2. Monthly Payment Extraction

**Model**: `int_financed_cash_flows_calculate_monthly_payments`

The only reliable QB field is the total amount charged to the client each month (`monthly_payment_with_insurance_and_iva`). The monthly payment is reverse-engineered:

```
monthly_payment_without_iva =
    (monthly_payment_with_insurance_and_iva / (1 + tax_rate))
    - down_payment_without_iva
    - insurance_payment_without_iva

iva_to_receive =
    (monthly_payment_with_insurance_and_iva / (1 + tax_rate)) * tax_rate
```

### 3. Payment Component Splitting

**Model**: `int_financed_cash_flows_combine_columns` (materialized as **table**)

QuickBase groups legal costs, down payment, and asset transfer into a single field. This model splits them:

| Payment Number | Down Payment | Asset Transfer | Legal Costs |
|---------------|-------------|----------------|-------------|
| 1 | combined_field - legal_costs | 0 | legal_costs |
| 2–4 | combined_field | 0 | 0 |
| 5+ | 0 | combined_field | 0 |

It also calculates the total amount going toward principal:

```
total_toward_principal = monthly_payment + down_payment (or asset_transfer)
```

This model is materialized as a table because the next step (recursive interest) is computationally expensive.

### 4. Interest & Amortization (Recursive CTE)

**Model**: `int_financed_cash_flows_calculate_interest` (materialized as **table**)

This is the core amortization engine. It uses a recursive CTE to calculate month-by-month interest, principal, and remaining balance.

**Month 1 (base case)**:
```
interest = retail_price * monthly_interest_rate
principal = MAX(0, total_toward_principal - interest)
remaining = retail_price - principal
```

**Month N (recursive step)**:
```
interest = previous_remaining_principal * monthly_interest_rate
principal = MAX(0, total_toward_principal - interest)
remaining = previous_remaining_principal - principal
```

The recursion joins on `project_reference` and `payment_number = previous_payment_number + 1`, walking forward one month at a time.

**Key behaviors**:
- If `monthly_interest_rate` is NULL (e.g. upfront projects that shouldn't be here), all interest/principal fields are NULL
- `GREATEST(0, ...)` prevents negative principal payments in edge cases
- Grace period months have `total_toward_principal = 0`, so the full payment goes to interest (or interest accrues on the remaining balance)

**Worked example** (retail price Q100,000, 2% monthly rate, Q5,000/month toward principal):

| Month | Remaining Start | Interest | Principal | Remaining End |
|-------|----------------|----------|-----------|---------------|
| 1 | Q100,000 | Q2,000 | Q3,000 | Q97,000 |
| 2 | Q97,000 | Q1,940 | Q3,060 | Q93,940 |
| 3 | Q93,940 | Q1,879 | Q3,121 | Q90,819 |
| ... | ... | ... | ... | ... |

Interest decreases over time as the principal is paid down (standard amortization).

### 5. Currency Conversion

**Models**: `int_historic_monthly_cash_flows_gtq`, `int_paid_up_front_cash_flows_gtq`, `int_monthly_cash_flows_convert_currencies`

All monetary fields are converted from project currency to GTQ and USD using exchange rates looked up by payment date:

```
-- Project currency to GTQ:
amount_gtq = amount * gtq_rate    (if project currency is USD)
amount_gtq = amount               (if project currency is GTQ)

-- Project currency to USD:
amount_usd = amount / gtq_rate    (if project currency is GTQ)
amount_usd = amount               (if project currency is USD)
```

Exchange rates come from `static_tables.exchange_rates`, matched by `effective_date` closest to `payment_date`.

---

## Accounts Receivable

**Models**: `int_accounts_receivable_by_project` → `_by_project_gtq` → `_summary_gtq`

Accounts receivable is calculated per-project per-month using a month spine (2020–2050). For any given reporting month, it splits outstanding amounts into short-term (due within 12 months) and long-term (due after 12 months).

### Principal AR

```
short_term_principal = SUM(principal_payment)
    WHERE payment_date > month_start
      AND payment_date <= month_start + 12 months
      AND contract_signing_date <= month_start

long_term_principal = remaining_principal_at_month_start - short_term_principal
```

### Interest AR

```
short_term_interest = SUM(interest_payment)
    WHERE payment_date > month_start
      AND payment_date <= month_start + 12 months
      AND contract_signing_date <= month_start

long_term_interest = SUM(interest_payment)
    WHERE payment_date > month_start + 12 months
      AND contract_signing_date <= month_start
```

### Insurance AR

Same pattern as interest — split by the 12-month boundary.

The `_summary_gtq` model aggregates across all projects into company-wide AR totals.

---

## Mart Models

### `mart_monthly_cash_flows`

**Grain**: One row per project per payment month.
**Materialization**: Table.

Combines financed and paid-upfront cash flows via UNION ALL. Paid-upfront rows have NULL for interest/principal/remaining_principal fields.

| Column | Description |
|--------|-------------|
| `project_reference` | Project identifier |
| `payment_number` | Sequential month number |
| `payment_date` | Payment due date |
| `sale_type` | `'Financiado'` or `'Contado'` |
| `down_payment_without_iva_gtq` | Down payment (financed only) |
| `asset_transfer_value_without_iva_gtq` | Asset transfer (financed only) |
| `monthly_payment_without_iva_gtq` | Monthly payment |
| `iva_to_receive_gtq` | IVA amount |
| `interest_payment_gtq` | Interest portion (financed only) |
| `principal_payment_gtq` | Principal portion (financed only) |
| `remaining_principal_gtq` | Outstanding principal (financed only) |
| `insurance_payment_without_iva_gtq` | Insurance income |
| `insurance_costs_without_iva_gtq` | Insurance direct cost |
| `legal_costs_without_iva_gtq` | Legal/admin costs |
| `maintenance_costs_without_iva_gtq` | Maintenance costs |
| `commission_costs_without_iva_gtq` | Sales commission costs |

---

### `mart_financial_projections`

**Grain**: One row per month (monthly P&L-style pivot).
**Materialization**: View.

Aggregates all cash flow data into a single wide table with one column per financial metric, pivoted by month. Used for financial modeling and investor reporting.

**Revenue metrics** (GTQ):
| Column | Source |
|--------|--------|
| `paid_up_front_installations_gtq` | Sum of upfront project retail prices by signing month |
| `financed_installations_gtq` | Sum of financed project retail prices by signing month |
| `finance_income_gtq` | Sum of interest payments by payment month |
| `insurance_income_gtq` | Sum of insurance payments by payment month |
| `legal_costs_gtq` | Sum of legal costs (both sale types) |
| `total_payments_gtq` | Sum of all payment components |

**Cost metrics** (GTQ):
| Column | Source |
|--------|--------|
| `paid_up_front_installation_costs_gtq` | Supplier cost for upfront projects |
| `financed_installation_costs_gtq` | Supplier cost for financed projects |
| `financed_commission_costs_gtq` | Commission on financed sales |
| `paid_up_front_commission_costs_gtq` | Commission on upfront sales |
| `insurance_costs_gtq` | Insurance direct costs |
| `maintenance_costs_gtq` | Maintenance costs |

**Balance sheet metrics** (GTQ):
| Column | Source |
|--------|--------|
| `short_term_accounts_receivable_gtq` | Principal due within 12 months |
| `long_term_accounts_receivable_gtq` | Principal due after 12 months |
| `short_term_interest_receivable_gtq` | Interest due within 12 months |
| `long_term_interest_receivable_gtq` | Interest due after 12 months |
| `short_term_insurance_receivable_gtq` | Insurance due within 12 months |
| `long_term_insurance_receivable_gtq` | Insurance due after 12 months |

**Config**: Projection window defined at top of model (currently 2025–2030). Only includes projects signed before the `last_actuals_date` to avoid double-counting pipeline.

---

### `mart_historic_accounting_report`

**Grain**: One row per project + one totals row.
**Materialization**: View.

Provides a per-project financial summary with year-by-year breakdowns for accounting. All amounts in GTQ without IVA.

| Column Group | Details |
|-------------|---------|
| Project identifiers | `project_reference`, `first_payment_month`, `contract_signing_date` |
| Cost & price | `project_cost_without_iva_gtq`, `retail_price_without_iva_gtq` |
| Contract summary | `number_of_payments`, `contract_payments_sum_gtq`, `monthly_interest_rate` |
| Interest by year | `interest_payment_gtq_2022`, `_2023`, `_2024` |
| Payments made by year | `payments_made_gtq_2022` (cumulative through EOY), `_2023`, `_2024` |
| Legal costs | Annual and cumulative breakdowns |
| Insurance payments | Annual and cumulative breakdowns |
| Combined | `total_legal_and_insurance_costs_without_iva_gtq` |

The `'TOTAL'` row sums all projects. `payments_made_gtq_YYYY` uses cumulative logic (all payments through end of year), while `interest_payment_gtq_YYYY` uses calendar-year windowing (only payments within that year).

---

### `mart_cash_flows_for_irr_calculation`

**Grain**: One row per project per payment (including month 0 for initial outlay).
**Materialization**: Table.

Structures cash flows for Internal Rate of Return calculations:

```
Month 0: net_cash_flow = -1 * retail_price_without_iva  (initial investment outflow)
Month N: net_cash_flow = total_amount_going_toward_principal  (cash inflow)
```

This produces the standard IRR input format: a negative initial cash flow followed by positive periodic returns. External tools (e.g. Excel, Python) consume this to compute IRR.

---

### `mart_paid_up_front_projects_summary`

**Grain**: One row per paid-upfront project.
**Materialization**: View.

Simple overview for cash-sale projects with Spanish-named columns (designed for direct consumption by business users):

| Column | Description |
|--------|-------------|
| `Fecha firma contrato` | Contract signing date |
| `Primer mes pago` | First payment month |
| `Ano Validado` | Validated year |
| `Costo Albedo` | Cost paid to supplier (COGS) |
| `Venta Albedo` | Commercial value / retail price |

---

### Reconciliation Models

**Models**: `mart_reconciliation_eeffs_by_financed_project`, `mart_reconciliation_eeffs_by_paid_up_front_project`

**Location**: `dbt/main/models/models_on_static_tables/marts/reconciliation/`

These produce wide pivoted tables with months as columns (2021–2030), designed for integration with Zoho accounting. Each project has multiple rows, one per account type:

**Account types (financed)**:
| Account (Spanish) | What it represents |
|---|---|
| Ingresos por Intereses | Interest income |
| Ingresos por Admin de Seguros | Insurance administration income |
| Ingresos por Admin de Arrendamientos | Lease administration income |
| AxC - Principal (CP) | Short-term principal receivable |
| AxC - Principal (LP) | Long-term principal receivable |
| AxC - Intereses (CP) | Short-term interest receivable |
| AxC - Intereses (LP) | Long-term interest receivable |
| CxC - Servicios de Contratos (CP) | Short-term service receivable |
| CxC - Servicios de Contratos (LP) | Long-term service receivable |
| Pago Principal | Principal payment received |
| Pago Total | Total payment received |
| Precio Retail | Retail price |
| Costo Proyecto | Project cost |

---

## Payment Component Breakdown

This diagram shows how a single monthly client payment is decomposed through the pipeline:

```
Client pays: monthly_payment_with_insurance_and_iva
  |
  ├── IVA (tax) → iva_to_receive
  |
  └── Amount without IVA
       |
       ├── Insurance payment → insurance_payment_without_iva
       |
       ├── Down payment / Asset transfer (months 1-4 / 5+)
       |
       └── Monthly payment → monthly_payment_without_iva
            |
            ├── Interest portion → interest_payment
            |
            └── Principal portion → principal_payment
                 |
                 └── Reduces → remaining_principal
```

---

## Currency Handling

All financial models exist in up to three currency variants:

| Suffix | Currency | Primary use |
|--------|----------|-------------|
| `_project_currency` | Whatever the contract is denominated in (GTQ or USD) | Source of truth |
| `_gtq` | Guatemalan Quetzales | Local accounting, tax reporting |
| `_usd` | US Dollars | Investor reporting, cross-country comparison |

Exchange rates are stored in `static_tables.exchange_rates` with fields `currency_code = 'GTQ'` and `effective_date`. Conversion uses the rate effective on the payment date to maintain temporal accuracy.

---

## Key Assumptions & Design Decisions

| Topic | Detail |
|-------|--------|
| **QB data quality** | QB doesn't reliably calculate individual payment components or IVA, so dbt reverse-engineers from the total client charge |
| **Interest rate source** | Monthly interest rates come from a Python-calculated staging table (`stg_calculated__interest_rates`), not from QB |
| **Materialization strategy** | Recursive operations (interest calculation) and their inputs are materialized as tables for performance; everything else is views |
| **Grace periods** | Handled naturally — months with zero `monthly_payment` still accrue interest on the remaining principal |
| **Addendum handling** | Cash flows after `addended_at` date are marked `is_nullified = TRUE` and excluded from active calculations |
| **Projection cutoff** | Financial projections only include projects signed before `last_actuals_date` to prevent double-counting pipeline deals |
| **Totals rows** | Accounting reports include a synthetic `'TOTAL'` row that sums across all projects |
| **Year boundaries** | Year-by-year metrics use two patterns: calendar-year windowing (interest) and cumulative-through-EOY (payments made) |

---

## Dependency Graph

```
stg_quickbase__monthly_cash_flows ──┐
stg_quickbase__quotes ──────────────┤
stg_calculated__interest_rates ─────┤
int_estimates_convert_currencies ───┤
rates_by_country (tax rates) ───────┤
exchange_rates (FX rates) ──────────┤
                                    │
  ┌─── int_financed_cash_flows ─────┤
  │       ↓                         │
  │    int_*_calculate_iva          │
  │       ↓                         │
  │    int_*_calculate_monthly_payments
  │       ↓                         │
  │    int_*_combine_columns [TABLE]│
  │       ↓                         │
  │    int_*_calculate_interest [TABLE]
  │       ↓                         │
  │    int_historic_monthly_cash_flows
  │       ↓                         │
  │    int_historic_monthly_cash_flows_gtq
  │                                 │
  ├─── int_paid_up_front_cash_flows │
  │       ↓                         │
  │    int_paid_up_front_*_calculate_iva
  │       ↓                         │
  │    int_paid_up_front_cash_flows_gtq
  │                                 │
  └─────────────────────────────────┤
                                    ↓
  mart_monthly_cash_flows ←── UNION ALL (financed + upfront)
  mart_financial_projections ←── metric slices + month spine
  mart_historic_accounting_report ←── year-by-year aggregation
  mart_cash_flows_for_irr_calculation ←── IRR cash flow format
  mart_paid_up_front_projects_summary ←── upfront project overview
  mart_reconciliation_eeffs_* ←── Zoho accounting pivot tables

  int_accounts_receivable_by_project ──→ _gtq ──→ _summary_gtq
    (month spine × cash flows, split by 12-month boundary)
```
