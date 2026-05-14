# Golden Copy Status

_Generated 2026-05-14T23:45:51.178370+00:00 against `https://ghulhjlbniwczcmhrysy.supabase.co`._  
_Source: `scripts/golden_copy/run_tests.py` in the [supabase repo](https://github.com/AlbedoSolar/supabase)._

Daily drift check: each case runs a known set of inputs through the deployed `quote-generator` edge function and compares the output to values captured from the Pricing Logic Estimator spreadsheet. See [Business Decisions Log](./6-business-decisions-log.md) for the conventions that govern expected behavior.

## Summary — 1/1 cases passed within tolerance

🟡 1 with expected deltas (convention differences)

## Cases

| # | Scenario | Retail final | Monthly pmt (first active) | Insurance | Maint | Legal fee | Status |
|---|----------|--------------|----------------------------|-----------|-------|-----------|--------|
| 01_gt_base | GT base · Subtract · 0% DP · no impact | GC 54,562.07 / SB 56,471.67 (+1,909.60) | GC 1,105.35 / SB 1,144.09 (+38.74) | GC 39.22 / SB 38.91 (-0.31) | GC 10.14 / SB 7.70 (-2.44) | GC 1,049.11 / SB 1,049.11 (0) | 🟡 |

_Cell format: `GC expected / SB actual (Δ vs GC)`._

## Case details

### 🟡 01_gt_base — GT base · Subtract · 0% DP · no impact

Anchor case: estimate 2200-06-01 (ESQUISOLAR 3% Subtract), term 127, grace 3, 0% down payment, annual rate pinned to GC's 20.24%, maintenance grace 5 (compensates SB's services-start +1 vs GC's 0).

| Field | GC | SB actual | Δ vs GC | Intentional Δ | Residual | Tol | Status |
|-------|------|-----------|---------|---------------|----------|-----|--------|
| `retail_base_pre_discount` | 54,562.07 | 54,562.00 | -0.07 | — | -0.07 | ±0.5 | ✅ |
| `retail_final_with_commission` | 54,562.07 | 56,471.67 | +1,909.60 | +1,909.67 | -0.07 | ±1.0 | 🟡 |
| `commission_total` | 0 | 1,909.67 | +1,909.67 | +1,909.67 | 0 | ±1.0 | 🟡 |
| `legal_fee` | 1,049.11 | 1,049.11 | 0 | — | 0 | ±0.5 | ✅ |
| `annual_rate_echoed` | 0.2024 | 0.2024 | 0 | — | 0 | ±0.0001 | ✅ |
| `monthly_payment_first_active` | 1,105.35 | 1,144.09 | +38.74 | +38.74 | 0 | ±1.0 | 🟡 |
| `monthly_insurance_payment` | 39.22 | 38.91 | -0.31 | -0.31 | 0 | ±0.5 | 🟡 |
| `monthly_maintenance_payment` | 10.14 | 7.70 | -2.44 | -2.44 | 0 | ±0.5 | 🟡 |
| `irr_achieved` | — | 15.78 |  | — |  | ±1.0 | ✅ — no GC reference — informational only |

_Pass logic: `|SB − (GC + intentional_delta)| ≤ tolerance`. 🟡 = passed via a documented intentional delta; ✅ = matched GC directly._

**Intentional delta justifications:**

- `retail_final_with_commission`: Commission added to retail per May 14 convention (client pays commission as part of retail). GC treats commission as an expense only — doesn't inflate retail. See decisions log § 2026-05-14 'Commission added to retail (open)'.
- `commission_total`: Same convention as above — SB exposes commission_total as a top-level field; GC has commission only in the expense column.
- `monthly_payment_first_active`: Propagation of the +Q 1,910 commission addition through the standard PMT formula on a larger principal. Expected to disappear if the commission-in-retail decision flips.
- `monthly_insurance_payment`: Off-by-one in services-payment denominator (Q 1,950 / 125 vs Q 1,950 / 124). See decisions log open item 'Off-by-one in service payment spreading denominator'.
- `monthly_maintenance_payment`: GC uses maintenance rate 0.20% and ESQUISOLAR margin 5%; SB pulls 0.15% from getMaintenanceRate and 3% from the providers table (both confirmed by Ian as the correct DB values 2026-05-14). GC will need updating when the spreadsheet is refreshed.

_Ran in 1883 ms._

