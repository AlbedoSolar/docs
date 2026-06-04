# QB vs new-system quote comparison

**Owner of follow-up:** Ian (calc engine)
**Prepared:** 2026-06-03
**Data set:** 6 estimates × 1 representative QB quote each, imported from QB into Supabase as test cases for the upcoming cutover. New-system numbers come from hitting the production `irr-calculator` edge function with the same inputs QB had.

## TL;DR for Ian

Across 6 diverse cases:
- **Retail price is exact in 4 of 6** ✅, but **differs by +10.8% and +25.0% in the other 2** ⚠️
- **Monthly payment is HIGHER in the new system in all 6 cases**, by +4.4% to +12.6%
- **Monthly maintenance comes back as Q0 from the new system in all 6 cases**, but QB had real maintenance amounts. Likely a rates-table lookup gap on the imported estimates.
- The 25% retail-price miss looks like **issue #222** (divide-by-(1−margin) vs multiply-by-(1+margin))

This report is meant to give you exact reproducible inputs so you can re-run them yourself and locate the calc deltas.

## Methodology

1. Picked 6 representative estimates spanning 5 different providers, project sizes from Q17k to Q382k, and term lengths from 15 to 75 months
2. For each, pulled the actual QB quote variant from `qb_raw.quotes` with its inputs (rate, term, dp%, grace) and stored outputs (monthly payment, retail price)
3. Called the production `irr-calculator` edge function with the **exact same inputs**
4. Compared outputs

The estimates were imported from QB into `public.estimates` earlier today (test migration, 20 projects total). Each estimate got one `project_phases` row with the provider derived from QB field 431 (with the Record ID#2 fallback for variant providers). Sites were also backfilled so the calc engine could resolve location. **`annual_maintenance_cost_sin_iva` and `annual_maintenance_currency` were deliberately left NULL on the imported estimates** — the calc engine is expected to fall back to the `maintenance_rates` table for these.

## Summary table

| # | Project | Provider | Term | DP% | Rate | QB Monthly | New Monthly | Δ Monthly | QB Retail | New Retail | Δ Retail |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | SANTIAGO GIRON | Energica | 51 | 10% | 11.99% | Q1,638.49 | Q1,844.57 | **+Q206 (+12.6%)** | Q63,468.82 | Q63,468.82 | ✅ exact |
| 2 | Hospital la Fe | TECKNOS SOLAR | 63 | 10% | 12% | Q995.58 | Q1,096.23 | **+Q101 (+10.1%)** | Q42,984.58 | Q42,984.58 | ✅ exact |
| 3 | Texaco San Miguel | Ecolumen | 75 | 10% | 12% | Q6,267.44 | Q6,705.95 | **+Q438 (+7.0%)** | Q275,770.86 | Q305,563.28 | ⚠️ **+Q29,792 (+10.8%)** |
| 4 | MAGNO CARTONES | Conexsol | 39 | 10% | 12% | Q12,786.02 | Q14,186.66 | **+Q1,400 (+11.0%)** | Q382,072.24 | Q382,072.24 | ✅ exact |
| 5 | Cafe Welchez | IBS GROUP | 51 | 10% | 12% | Q850.53 | Q887.95 | **+Q37 (+4.4%)** | Q29,750.00 | Q29,750.00 | ✅ exact |
| 6 | Carlos Palencia | ESQUISOLAR MARGEN 20% | 15 | 10% | 12% | Q1,816.84 | Q1,977.66 | **+Q161 (+8.9%)** | Q17,028.00 | Q21,285.00 | ⚠️ **+Q4,257 (+25.0%)** |

Maintenance from new system in all 6 cases: **Q0**. QB's stored maintenance ranged from Q952/yr to Q3K+/yr.

## Hypotheses, ranked by my confidence

### 1. Retail price miss on Carlos Palencia = issue #222 (high confidence)

Carlos Palencia uses provider **ESQUISOLAR MARGEN 20%**. The math:
- QB retail: Q17,028
- New retail: Q21,285
- Ratio: 17,028 / 21,285 = **0.80** (exactly)

That's the divide-by-(1−margin) pattern. Q17,028 × 1/0.80 = Q21,285. If QB stored the partner cost as Q17,028 and applied the 20% margin as ×1.20 = Q20,433.60, but the new system applies as ÷0.80 = Q21,285, that's an extra Q851. Reverse the formulas and the QB and new system would match — which is what your issue #222 hypothesis says.

Suggests the new calc engine is applying margin via divide-by-(1−margin) at least for `Subtract` cartera providers (ESQUISOLAR is `Subtract`).

### 2. Maintenance not loading from rates table for any case (high confidence)

All 6 cases return `monthly_maintenance_payment = 0` from the calc engine. QB had real maintenance (e.g. SANTIAGO GIRON was Q952/yr).

The imported estimates have `annual_maintenance_cost_sin_iva = NULL` because we deliberately left it null (waiting for the calc engine to fall back to `maintenance_rates`). Either:
- The fallback to `maintenance_rates` isn't kicking in
- The lookup needs more than just `provider_id` (maybe department or municipality) and the imported sites don't have it
- Or the new behavior is "rep must enter manually" and the QB import should fill in `annual_maintenance_cost_sin_iva` directly

This is independently fixable — once maintenance loads, monthly payments will go up a bit more, widening the delta from QB. So Q0 maintenance isn't currently the cause of the monthly-payment delta — it's a separate gap.

### 3. Retail price miss on Texaco San Miguel (medium confidence)

Texaco uses **Ecolumen**. The ratio:
- QB retail: Q275,770.86
- New retail: Q305,563.28
- Ratio: 275,770.86 / 305,563.28 = **0.9025**

That's not a clean (1−margin) like Carlos Palencia's was. Ecolumen has a different effective margin (need to check `providers.standard_wholesale_margin` for Ecolumen vs whatever margin was used in QB). Could still be the same divide-vs-multiply pattern but at a different margin %, or it could be something else (currency conversion?).

Worth checking: what's Ecolumen's standard_wholesale_margin in Supabase, and what cartera (`quote_margin_type`) does it have?

### 4. Monthly payment delta varies, not constant (medium confidence)

Across the 6 cases the new-system monthly payment is ALWAYS higher, but by varying amounts (4.4% to 12.6%). If it were a single formula bug we'd expect a constant ratio. Three possibilities:
- **Insurance treatment difference**: QB might have bundled insurance into the monthly payment for some cases but not others. The new system returns `monthly_insurance_payment` as a separate line.
- **Grace period interpretation**: the formula for how interest accrues during grace might differ.
- **IVA treatment**: QB stores IVA-stripped vs IVA-included differently per the existing Lambda's translation logic; we may be feeding IVA-included into the new system somewhere it expects IVA-stripped.

## How to reproduce any case

The new-system call for each row above:

```bash
curl -X POST "$SUPABASE_URL/functions/v1/irr-calculator" \
  -H "Content-Type: application/json" \
  -H "apikey: $SUPABASE_ANON_KEY" \
  -d '{
    "estimate_reference": "<ref>",
    "mode": "forward_with_rate",
    "annual_rate": <rate>,
    "term_months": <term>,
    "grace_period": 3,
    "down_payment_percentage": <dp>
  }'
```

The QB quote rows are in `qb_raw.quotes`. Use `(raw_data->'6'->>'value')::int IN (...)` with the estimate IDs to find them. Each case's exact QB quote_record_id:

| # | estimate_ref | qb_quote_id | qb_quote_ref |
|---|---|---|---|
| 1 | 2535-01-01 | 38521 | 2535-01-01-01 |
| 2 | 2359-01-01 | 35490 | 2359-01-01-02 |
| 3 | 1037-01-01 | 15196 | 1037-01-01-01 |
| 4 | 1511-01-01 | 18631 | 1511-01-01-02 |
| 5 | 1414-01-01 | 16378 | 1414-01-01-02 |
| 6 | 2526-01-01 | 38441 | 2526-01-01-01 |

## Other things worth knowing

- This is the FIRST end-to-end test of the new-system calc against real QB data. Previous testing was per-component.
- The 20 projects we imported as test data are visible in `/sales` (Ventas en proceso) on staging today. Reps can open them and try to generate quotes in the wizard — the math will be the same as what's in this report.
- The imported estimates aren't yet wired up with the cartera variants or some of the maintenance/insurance overrides. As we expand to the 55 in-diligence projects, those gaps grow.

## Asks for Ian

1. Confirm #1 (Carlos Palencia / ESQUISOLAR retail miss = #222 divide-by-(1−margin) pattern). If yes, this is your existing ticket — quote the connection back to this comparison for prioritization.
2. Look at #2 (Q0 maintenance everywhere) — is the calc engine *supposed* to fall back to `maintenance_rates` when `annual_maintenance_cost_sin_iva IS NULL`, or does the import need to set it?
3. Sanity-check #3 (Texaco / Ecolumen retail miss) — same root cause as #1, or different?
4. Pick one of the monthly-payment deltas and trace it through. The grace-period interpretation feels like the most likely shared cause.

Once we have a fix for any of these, I can re-run all 6 cases in seconds for regression. If it's useful I can also pull the per-month cashflow breakdown from `qb_raw.monthly_cashflows` (we have it for every QB quote in the test set) and compare line-by-line against the new system's `cashflow_table` to localize the disagreement to a specific month/component.
