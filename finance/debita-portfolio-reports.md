# Reportes de Cartera — Debita SPV

Documentation for the three portfolio reports used for Debita SPV due diligence. These reports combine contract data from Solarbase with billing data from Zoho Books (Guatemala) and manual accounting exports (Honduras).

---

## Data Sources

| Source | Country | What it provides | Table |
|---|---|---|---|
| **Solarbase** | All | Contract terms, amortization schedules, project/client details | `projects`, `estimates`, `quotes`, `monthly_cash_flows` |
| **Zoho Books** | Guatemala | Invoice status, payment dates, outstanding balances | `zoho_invoices` |
| **Zoho Books** | Guatemala | Individual payment records | `lease_payments` |
| **HN accounting CSV** | Honduras | Invoices and payments (pre-converted to USD at 26.55 HNL/USD) | `zoho_invoices`, `lease_payments` |
| **Exchange rates** | All | Daily GTQ/USD and HNL/USD rates | `exchange_rates` |

### Important: Invoice data is the source of truth for delinquency

All delinquency metrics (DPD, cuotas en mora, estado del credito, bucket de mora) are driven by invoice status in the `zoho_invoices` table, not by comparing Solarbase cash flows to payment records. Both Zoho-sourced (GT) and HN accounting-sourced invoices use the same table and status conventions: "Overdue" = unpaid, "Closed" = paid.

### Honduras-specific notes

- Honduras does not use Zoho Books. Invoice and payment data comes from a manual accounting export (`ALBEDO SOLAR S.A. DE C.V._Facturas y pagos recibidos.csv`), processed by `external-resources/honduras_fifo_loader.py`.
- HN amounts are converted to USD using a flat rate of 26.55 HNL/USD (the Lempira is a managed peg with minimal fluctuation). All HN rows in `zoho_invoices` and `lease_payments` have `currency_code = 'USD'`.
- HN invoice due date = **10th of the month** in which the payment is registered in the cash flow schedule.
- Some HN clients have **combined billing** (one invoice covers multiple projects). These are split proportionally in `zoho_invoices` using median monthly payment weights from `monthly_cash_flows`. Split invoices have suffixed numbers (e.g., `000-001-01-00000038-01`, `-02`, `-03`). Affected clients: Hospital Clínicas San Jorge (3 projects), Productos Lácteos Nelly (2 projects), Hilda Cabrera (2 projects).
- HN accounting customer names are often **DBA / commercial names** (e.g., "Plaza Nexus") while Supabase has the legal entity (e.g., "EMPRESA COMERCIAL E INVERSIONES SOCIEDAD DE RESPONSABILIDAD LIMITADA DE CAPITAL VARIABLE"). The mapping is maintained in `external-resources/honduras_facturas_filled.csv`.

---

## Report 1: Loan Tape (`mart_debita_loan_tape`)

**Purpose:** Current state of every signed lease. One row per project.

**Scope:** Financed leases in Guatemala and Honduras (`sale_type = 'Financiado'`, `country_id IN (1, 3)`). Contado projects and El Salvador are excluded.

### Columns

| Column | Description | Source |
|---|---|---|
| `id_arrendamiento` | Project reference (e.g., "402-01") | Solarbase |
| `id_cliente` | Numeric client ID | Solarbase |
| `nombre_cliente` | Legal name of the client | Solarbase |
| `pais` | Country (Guatemala, Honduras, El Salvador) | Solarbase |
| `fecha_de_proyecto` | Project date — adjusted per internal logic (post-Nov 2025: signing date; pre-Nov 2025: first payment date; with manual overrides) | Solarbase |
| `fecha_vencimiento` | Last scheduled payment date | Solarbase cash flows |
| `moneda_arrendamiento` | Original currency (GTQ, USD, HNL) | Solarbase |
| `monto_prestado_sin_iva_usd` | Retail price excluding IVA, converted to USD | Solarbase |
| `monto_prestado_con_iva_usd` | Retail price including IVA, converted to USD | Solarbase |
| `total_pagado_con_iva_usd` | Total amount paid to date (sum of invoice amounts minus remaining balances) | Zoho invoices |
| `monto_principal_vigente_con_iva_usd` | Outstanding principal from the amortization schedule, adjusted for actual payments made (see below) | Solarbase + Zoho |
| `saldo_total_vigente_con_iva_usd` | Total remaining obligation = total contract value con IVA minus total paid | Solarbase + Zoho |
| `tasa_interes_anual_pct` | Annual interest rate (%). Uses `annual_interest_rate` if available, otherwise `monthly_interest_rate × 12`. | Solarbase |
| `plazo_original_meses` | Total number of payment months in the contract (excludes month 0) | Solarbase cash flows |
| `seasoning_meses` | Number of payment periods that have come due as of today (excludes month 0) | Solarbase cash flows |
| `plazo_remanente_meses` | Number of payment periods with future dates | Solarbase cash flows |
| `cuota_mensual_con_iva_usd` | Median monthly payment including IVA | Solarbase cash flows |
| `dpd` | Days Past Due — days since the oldest overdue invoice's due date, after a 10-day grace period. NULL if no invoice data exists. | Zoho invoices |
| `cuotas_en_mora` | Count of overdue invoices with outstanding balance > $5 (sub-$5 residuals from rounding are excluded) | Zoho invoices |
| `estado_credito` | Credit status (see definitions below) | Derived |
| `suspension_por_incidencia_operativa` | Manual flag for operational suspensions — not yet populated | Manual |
| `bucket_mora` | Delinquency bucket (see definitions below) | Derived |
| `kw_instalados` | Installed solar capacity in kilowatts | Solarbase |

### How Monto Principal Vigente works

The amortization schedule in Solarbase defines the remaining principal after each payment. Instead of looking up the principal at today's date (which assumes the client is current), we look it up at the number of payments actually made.

Example: A project with 8 years of payments (96 months). If the client has paid 60 invoices, we use the remaining principal after payment #60, even if 65 payments should have been made by now. This gives a more accurate balance for delinquent clients.

If a project has no invoice data, we fall back to the first payment row as a conservative estimate.

### How Saldo Total Vigente works

Total remaining obligation from the client's perspective:

    Saldo = (Total contract value × (1 + IVA rate)) − Total paid to date

- **Total contract value** is the sum of all scheduled cash flows (monthly payments + down payment + insurance + maintenance + asset transfer + legal costs), before IVA.
- **IVA rate** comes from the country (Guatemala 12%, El Salvador 13%, Honduras 15%).
- **Total paid** is the sum of `(invoice total − invoice balance)` across all Zoho invoices for the project. This is already con IVA since Zoho invoices include tax.

### Estado del Crédito definitions

Per the *Definiciones Operativas de Cartera* document:

| Status | Condition |
|---|---|
| **Sin datos de facturación** | No Zoho invoices exist for this project |
| **Finalizado** | Past the last scheduled payment date and all invoices are paid |
| **Vigente** | Active lease with zero overdue invoices (DPD = 0) |
| **Mora Temprana** | DPD 1–29 |
| **En Mora** | DPD 30–89 |
| **Default** | DPD 90+ (classification only — does not imply automatic execution) |

### DPD calculation

DPD counts days since the invoice due date, but only after a **10-day grace period** per the *Definiciones Operativas*. An invoice is not considered overdue until `current_date > due_date + 10 days`. This means a payment made within 10 days of the due date does not trigger any delinquency.

DPD is calculated at the contract level based on the **oldest** unpaid invoice (i.e., worst-case across all overdue invoices for the project).

Invoices with a remaining balance of **$5 or less** are excluded from the overdue calculation, as these represent rounding residuals rather than real delinquency.

### Bucket de Mora definitions

| Bucket | Days Past Due |
|---|---|
| **Al día** | DPD = 0 |
| **1-29** | 1–29 days |
| **30-59** | 30–59 days |
| **60-89** | 60–89 days |
| **90+** | 90 or more days (Default) |

### PAR (Portfolio at Risk)

PAR is a portfolio-level metric, not included in the loan tape directly. It can be derived:

    PAR30 = Sum of saldo_total_vigente where DPD >= 30 / Sum of saldo_total_vigente for all active contracts

The numerator uses the **full contract balance**, not just overdue installments.

### Currency conversion

All USD amounts are converted using the exchange rate from the **last day of the previous month**. For example, a report generated any day in March uses the February 28 (or closest prior business day) rate. This ensures the report produces stable, reproducible numbers regardless of when it's run within the month.

IVA is applied to Solarbase-sourced amounts using the country's tax rate before currency conversion. Zoho-sourced amounts (total paid, overdue balances) already include IVA.

---

## Report 2: Payment Tape (`mart_debita_historical_payment_tape`)

**Purpose:** Transaction-level record of every payment received. One row per payment.

**Scope:** Payments with amount > 0 for Guatemala and Honduras projects (excludes tax-only entries and El Salvador).

### Columns

| Column | Description | Source |
|---|---|---|
| `id_prestamo` | Project reference (mapped from invoice → project via `invoice_project_map`) | Zoho → Solarbase |
| `numero_factura` | Zoho invoice number | Zoho payments |
| `fecha_esperada` | Invoice due date from `zoho_invoices.due_date`, falling back to cash flow `payment_date` if no matching invoice exists. For GT this is the Zoho-defined due date; for HN it's the 10th of the month. | Zoho invoices / Solarbase cash flows |
| `fecha_real_pago` | Actual date the payment was received | Zoho payments |
| `monto_pagado_usd` | Payment amount in USD | Zoho payments |
| `dpd_al_momento` | Days between the scheduled due date and the actual payment date. Positive = late, negative = early. NULL if no linked cash flow. | Derived |

### Data coverage

**Guatemala:** Payment data covers January 2025 onward (from Zoho Books exports). Payments before 2025 are not in the system. Not all payments are linked to a specific cash flow row — `fecha_esperada` may be NULL for older payments imported before the linking logic was in place.

**Honduras:** Payment data covers July 2025 onward (from the HN accounting CSV export). Payments are FIFO-matched to invoices by the `honduras_fifo_loader.py` script and linked to cash flow rows where possible. HN payments are stored in USD (converted from Lempiras at 26.55). The `source` column distinguishes HN records (`hn_accounting_csv`) from GT records.

---

## Report 3: Monthly Snapshots (`mart_debita_snapshots`)

**Purpose:** Point-in-time view of every lease at each month-end over the last 24 months. Enables vintage analysis, cohort tracking, and delinquency evolution.

**Scope:** Financed leases in Guatemala and Honduras. One row per lease per month-end. A lease only appears in months after its contract signing date.

### Columns

Similar to the Loan Tape but with some differences. The snapshot adds `fecha_snapshot` and omits some fields that are only meaningful as a current-state view (total paid, saldo vigente, kW installed).

| Column | Description | Source |
|---|---|---|
| `fecha_snapshot` | The month-end date this row represents (e.g., "2025-06-30") | Generated |
| `id_arrendamiento` | Project reference | Solarbase |
| `id_cliente` | Numeric client ID | Solarbase |
| `nombre_cliente` | Legal name of the client | Solarbase |
| `pais` | Country | Solarbase |
| `fecha_de_proyecto` | Project date (adjusted per internal logic) | Solarbase |
| `fecha_vencimiento` | Last scheduled payment date | Solarbase cash flows |
| `moneda_arrendamiento` | Original currency (GTQ, USD, HNL) | Solarbase |
| `monto_prestado_con_iva_usd` | Retail price including IVA, converted to USD | Solarbase |
| `monto_principal_vigente_con_iva_usd` | Outstanding principal at snapshot date, including IVA | Solarbase |
| `tasa_interes_anual_pct` | Annual interest rate (%). Uses `annual_interest_rate` if available, otherwise `monthly_interest_rate × 12`. | Solarbase |
| `plazo_original_meses` | Total payment months in the contract (excludes month 0) | Solarbase cash flows |
| `seasoning_meses` | Payment periods due as of the snapshot date (excludes month 0) | Solarbase cash flows |
| `plazo_remanente_meses` | Payment periods after the snapshot date | Solarbase cash flows |
| `cuota_mensual_con_iva_usd` | Median monthly payment including IVA | Solarbase cash flows |
| `dpd` | Days Past Due at the snapshot date | Zoho invoices |
| `cuotas_en_mora` | Overdue invoice count at the snapshot date | Zoho invoices |
| `estado_credito` | Credit status at the snapshot date | Derived |
| `suspension_por_incidencia_operativa` | Manual flag — not yet populated | Manual |
| `bucket_mora` | Delinquency bucket at the snapshot date | Derived |

All date-relative fields (seasoning, remaining term, DPD, mora, estado) are computed as of the snapshot date, not today.

### How historical delinquency is reconstructed

Since Zoho invoice status reflects the current state (not historical), we reconstruct what the status was at each snapshot date using two fields:

- **due_date**: When the invoice was due
- **last_payment_date**: When it was actually paid (NULL if still unpaid)

An invoice counts as **overdue at a given snapshot date** if:
1. Its due date is on or before the snapshot date (it was already due), AND
2. It either has no payment date, or its payment date is after the snapshot date (it hadn't been paid yet at that point)

Example: An invoice due February 13 that was paid on June 15 will show as overdue in the February, March, April, and May snapshots, and as current from June onward.

### Currency conversion

Each snapshot row is converted to USD using the exchange rate from that snapshot's month-end date (or the closest prior business day if the month-end falls on a weekend/holiday). This means earlier snapshots use the FX rate that was current at that time, giving a historically accurate USD value.

---

## Known Limitations

1. **Invoice data is static.** The `zoho_invoices` table is populated from CSV exports, not a live API sync. The last GT export was on **April 9, 2026**; the last HN export was on **April 8, 2026**. Payments received after these exports won't be reflected until the next import. The upsert script is at `external-resources/zoho_invoice_upsert.py` (GT) and `external-resources/honduras_fifo_loader.py` (HN).

2. **Pre-2025 payment gap (GT).** Guatemala payment records cover January 2025 onward. All payments before that are assumed to have been made. Honduras payment records cover July 2025 onward (the start of the HN entity's invoicing).

3. **Snapshot delinquency is an approximation.** The historical reconstruction works well for invoices that go from overdue to paid. However, if an invoice was created after a snapshot date, it won't appear in earlier snapshots even if the underlying payment was theoretically due. This is a minor edge case since invoices are typically created near their due date. The snapshot model also applies the $5 balance threshold to avoid rounding residuals affecting historical mora status.

4. **Suspensión por incidencia operativa** is not yet populated. This column exists as a placeholder for manual flags indicating operational issues (e.g., meter not installed, grid connection delayed) that explain delinquency without it being a credit risk.

5. **Projects without invoices** show as "Sin datos de facturación" with NULL delinquency metrics. As of April 2026, there are ~17 such projects — mostly recently signed leases in grace periods or old GT projects that completed before Zoho tracking began.

6. **~120 GT invoices remain unmapped** to a Supabase project (no Zoho Project ID available). These are tracked in `invoice_project_map` with `match_method = 'pending'` and need manual resolution.

7. **HN multi-project combined billing** relies on proportional allocation weights from the `monthly_cash_flows` median. If a client's projects have materially different payment schedules (e.g., one project in grace, another active), the allocation may need manual adjustment.
