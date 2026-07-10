# Payment Data Schema — Loan Tape

## Overview

Two new tables to centralize payment data from Zoho Books into Solarbase.

**Key design decision:** We don't duplicate invoice financial data (due dates, balances, etc.) — that's already in Solarbase via `monthly_cash_flows`. We only import:
1. **Actual payments received** (`lease_payments`)
2. **Invoice → project mapping** (`invoice_project_map`) — because Zoho payments only have Customer Name, which is ambiguous for multi-project clients

DPD, saldo vigente, and cuotas en mora are all **computed** by comparing actual payments against the scheduled cash flows already in Solarbase.

## Tables

### Table 1: `lease_payments`

One row per actual payment received from a client. This is the core data we're importing.

```sql
CREATE TABLE lease_payments (
  id SERIAL PRIMARY KEY,

  -- Zoho identifiers
  zoho_payment_id TEXT UNIQUE NOT NULL,       -- Zoho's CustomerPayment ID
  zoho_invoice_payment_id TEXT,               -- Zoho's InvoicePayment ID (per-invoice allocation)
  zoho_customer_id TEXT,

  -- Link to Solarbase
  project_id INTEGER REFERENCES projects(id), -- Resolved via invoice_project_map

  -- Payment details
  payment_date DATE NOT NULL,
  amount NUMERIC(12, 2) NOT NULL,             -- Total payment amount
  amount_applied NUMERIC(12, 2),              -- Amount applied to the linked invoice
  currency_code TEXT DEFAULT 'GTQ',
  exchange_rate NUMERIC(12, 6),               -- Zoho's exchange rate at time of payment
  amount_usd NUMERIC(12, 2),                  -- Converted to USD
  payment_mode TEXT,                          -- Transferencia Bancaria, Efectivo
  withholding_tax_amount NUMERIC(12, 2) DEFAULT 0,

  -- Invoice link
  zoho_invoice_number TEXT,                   -- e.g., 1INV-000007
  customer_name TEXT,                         -- From Zoho, for debugging/matching

  -- Metadata
  source TEXT NOT NULL DEFAULT 'zoho_v2',     -- zoho_v1, zoho_v2, manual
  raw_data JSONB,                             -- Full Zoho row for debugging
  synced_at TIMESTAMPTZ DEFAULT NOW(),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_lease_payments_project ON lease_payments(project_id);
CREATE INDEX idx_lease_payments_date ON lease_payments(payment_date);
CREATE INDEX idx_lease_payments_invoice ON lease_payments(zoho_invoice_number);
```

### Table 2: `invoice_project_map`

Mapping table that resolves which Solarbase project each Zoho invoice belongs to. Built from the facturas export — we only need the invoice number and project name, not the financial data.

```sql
CREATE TABLE invoice_project_map (
  id SERIAL PRIMARY KEY,

  -- Zoho invoice
  zoho_invoice_number TEXT NOT NULL,          -- e.g., 1INV-000007
  zoho_project_name TEXT,                     -- e.g., "Vicente Gil", "C-204"
  zoho_customer_name TEXT,

  -- Solarbase link
  project_id INTEGER REFERENCES projects(id), -- Resolved manually or semi-automatically
  project_reference TEXT,                      -- e.g., "204-01" (denormalized for convenience)

  -- Status
  match_method TEXT,                          -- 'auto_name', 'auto_reference', 'manual'
  match_confidence TEXT DEFAULT 'unmatched',  -- 'high', 'low', 'unmatched'

  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),

  UNIQUE(zoho_invoice_number)
);

CREATE INDEX idx_invoice_project_map_project ON invoice_project_map(project_id);
```

## How the loan tape computes DPD and mora

No separate invoice table needed — we compare actual payments against Solarbase's scheduled cash flows:

```sql
-- For each project: find scheduled payments that should have been paid by now
-- but don't have a matching lease_payment

WITH scheduled AS (
  SELECT project_id, payment_date AS due_date,
    monthly_payment_project_currency AS amount_due
  FROM monthly_cash_flows mcf
  JOIN quotes q ON mcf.quote_id = q.id
  WHERE q.status = 'signed' AND mcf.payment_number > 0
    AND mcf.payment_date <= CURRENT_DATE
),
actual AS (
  SELECT project_id, SUM(amount_applied) AS total_paid
  FROM lease_payments
  GROUP BY project_id
),
expected AS (
  SELECT project_id, SUM(amount_due) AS total_due,
    MIN(due_date) AS earliest_due
  FROM scheduled
  GROUP BY project_id
)
SELECT
  e.project_id,
  e.total_due,
  COALESCE(a.total_paid, 0) AS total_paid,
  e.total_due - COALESCE(a.total_paid, 0) AS saldo_vigente,
  -- DPD: if underpaid, days since earliest unpaid period
  -- (simplified — real calc needs per-period matching)
  CASE
    WHEN COALESCE(a.total_paid, 0) >= e.total_due THEN 0
    ELSE CURRENT_DATE - e.earliest_due
  END AS dpd_estimate
FROM expected e
LEFT JOIN actual a ON e.project_id = a.project_id;
```

Note: The real DPD calculation needs per-period matching (which specific months are unpaid), not just total amounts. This is a simplified version to illustrate the approach.

## The `source` field

Tracks where each payment record came from:
- `zoho_v2` — Current Zoho Books (2025+), auto-synced
- `zoho_v1` — Historical export from old Zoho, one-time import
- `manual` — Manually entered corrections

## Sync strategy

**Phase 1 (now):** CSV import of the 4 exports we already have (facturas 2025/2026, pagos 2025/2026)

**Phase 2 (ongoing):** Daily sync via Zoho Books API → Supabase (once we have API access set up)
