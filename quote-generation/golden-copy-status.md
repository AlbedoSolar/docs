# Golden Copy Status

_Generated 2026-05-14T23:00:29.545908+00:00 against `https://ghulhjlbniwczcmhrysy.supabase.co`._  
_Source: `scripts/golden_copy/run_tests.py` in the [supabase repo](https://github.com/AlbedoSolar/supabase)._

Daily drift check: each case runs a known set of inputs through the deployed `quote-generator` edge function and compares the output to values captured from the Pricing Logic Estimator spreadsheet. See [Business Decisions Log](./6-business-decisions-log.md) for the conventions that govern expected behavior.

## Summary — 1/1 cases passed within tolerance

✅ 1 fully matching

## Cases

| # | Scenario | Retail final | Monthly pmt (first active) | Insurance | Maint | Legal fee | Status |
|---|----------|--------------|----------------------------|-----------|-------|-----------|--------|
| 01_gt_base | GT base · Subtract · 0% DP · no impact | 56,471.67 / 56,471.67 (0) | 1,144.09 / 1,144.09 (0) | 38.91 / 38.91 (0) | 7.70 / 7.70 (0) | 1,049.11 / 1,049.11 (0) | ✅ |

_Format per cell: `expected / actual (Δ)`._

## Case details

### ✅ 01_gt_base — GT base · Subtract · 0% DP · no impact

Anchor case: estimate 2200-06-01 (ESQUISOLAR 3% Subtract), term 127, grace 3, 0% down payment, annual rate pinned to GC's 20.24%, maintenance grace 5 (compensates SB's services-start +1 vs GC's 0).

| Field | Expected | Actual | Δ | Tolerance | Status |
|-------|----------|--------|---|-----------|--------|
| `retail_base_pre_discount` | 54,562.00 | 54,562.00 | 0 | ±0.5 | ✅ |
| `retail_final_with_commission` | 56,471.67 | 56,471.67 | 0 | ±0.5 | ✅ |
| `commission_total` | 1,909.67 | 1,909.67 | 0 | ±0.5 | ✅ |
| `legal_fee` | 1,049.11 | 1,049.11 | 0 | ±0.5 | ✅ |
| `annual_rate_echoed` | 0.2024 | 0.2024 | 0 | ±0.0001 | ✅ |
| `monthly_payment_first_active` | 1,144.09 | 1,144.09 | 0 | ±0.5 | ✅ |
| `monthly_insurance_payment` | 38.91 | 38.91 | 0 | ±0.5 | ✅ |
| `monthly_maintenance_payment` | 7.70 | 7.70 | 0 | ±0.5 | ✅ |
| `irr_achieved` | — | 15.78 |  | ±0.05 | ✅ — no expected — informational only |

**Expected deltas vs Golden Copy** (open business decisions or known convention differences):

- `retail_final_with_commission`: GC reports 54,562 (no commission). SB adds commission per the May 14 convention. ~Q 1,910 expected delta on retail and every downstream payment column.
- `monthly_payment_first_active`: GC 1,105. SB 1,144 = +Q 39/mo from larger principal.

_Ran in 1741 ms._

