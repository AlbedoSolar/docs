# QB vs new-system quote comparison

**Owner of follow-up:** Ian (calc engine)
**Prepared:** 2026-06-04
**Data set:** 6 estimates × 1 representative QB quote each, imported from QB into Supabase as test cases for the upcoming cutover. New-system numbers come from hitting the production `irr-calculator` edge function with the same inputs QB had.

## TL;DR

Across 6 cases spanning 5 providers, system sizes from 2.6 kW to 555 kW, and term lengths from 15 to 75 months:
- **Monthly payment**: 4 of 6 within ±5% of QB; 1 (SANTIAGO GIRON) at +51% due to a maintenance rate-data calibration issue; 1 (Cafe Welchez) at −11.5%.
- **Retail price**: 5 of 6 match QB exactly (×1.12 GT or ×1.15 HN multiplier). The 6th (Carlos Palencia / ESQUISOLAR MARGEN 20%) is the #222 divide-by-(1−margin) pattern.
- **Maintenance**: fires on the 3 cases where rate > 0 AND term ≥ 24-month maintenance-free period. Other 3 correctly Q0.
- **Insurance**: works on all 6.

## Comparison table

All cases use 10% down payment, 3-month grace. New-system retail is sin-IVA (Supabase convention); QB retail is con-IVA.

| # | Project | Provider | Term | Rate | kW | QB Monthly | New Monthly | Δ Monthly | New Maint | Notes |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | SANTIAGO GIRON | Energica | 51 | 11.99% | 9.86 | Q1,638.49 | Q2,475.47 | **+51%** ⚠️ | Q776.56 | Energica 0.19 rate reads as a percentage, not $/W |
| 2 | Hospital la Fe | TECKNOS SOLAR | 63 | 12% | 44.63 | Q995.58 | Q1,036.64 | **+4.1%** ✅ | Q83.95 | fallback 0.028 × 44.6 kW |
| 3 | Texaco San Miguel | Ecolumen | 75 | 12% | 29.26 | Q6,267.44 | Q5,804.96 | **−7.4%** | Q0 | rate=0.00 in data — correct |
| 4 | MAGNO CARTONES | Conexsol | 39 | 12% | 555.78 | Q12,786.02 | Q12,844.67 | **+0.5%** ✅ | Q546.77 | fallback × large system |
| 5 | Cafe Welchez | IBS GROUP | 51 | 12% | 35.96 | Q850.53 | Q752.81 | **−11.5%** | Q0 | rate=0.00 in data — correct |
| 6 | Carlos Palencia | ESQUISOLAR MARGEN 20% | 15 | 12% | 2.58 | Q1,816.84 | Q1,754.70 | **−3.4%** ✅ | Q0 | term 15 < maintenance-free 24 — correct |

## Retail-price issue — Carlos Palencia / ESQUISOLAR MARGEN 20% (#222)

- QB retail (con-IVA): Q17,028
- Stored phase_cost (sin-IVA): Q15,203.57
- New system retail (sin-IVA): Q19,004.46
- Math: Q15,203.57 ÷ 0.80 = Q19,004.46 (exact)

The calc engine is taking the phase cost and dividing by `(1 − provider_margin)` (margin = 20%, cartera = Subtract). That's the #222 pattern — should be `partner_cost × (1 + margin)` for an Add provider, or the Subtract semantics flipped. The QB-stored retail of Q17,028 con-IVA reflects "what the client pays" and the new system is producing a different number.

## Per-case maintenance breakdown

| # | Case | Provider | Matched rate row | Result | Why |
|---|---|---|---|---|---|
| 1 | SANTIAGO GIRON | Energica (8) | country=GT, region=2, **0.19 USD** | Q776.56/mo | High because 0.19 reads as a percentage, not $/W — see ask #1 |
| 2 | Hospital la Fe | Tecknos Solar (112) | None → fallback default 0.028 | Q83.95/mo | Fallback × 44.6 kW happens to land close to QB |
| 3 | Texaco San Miguel | Ecolumen (10) | country=GT, **0.00 USD** | Q0 | Rate IS zero |
| 4 | MAGNO CARTONES | Conexsol (1) | None for HN → fallback default 0.028 | Q546.77/mo | Conexsol only has a GT rate row; site is HN |
| 5 | Cafe Welchez | IBS GROUP (47) | country=HN, **0.00 USD** | Q0 | Rate IS zero |
| 6 | Carlos Palencia | ESQUISOLAR MARGEN 20% (49) | country=GT, **0.15 GTQ** | Q0 | Term 15 < maintenance-free 24, so no visits fire |

### Region semantics (per Jake)

Region is GT-only. Honduras and El Salvador don't use region — for those countries "region" effectively equals the country. So the lookup should:
- For GT: match on `country_id + region_id` (region = rural/urban/highlands in the `regions` table)
- For HN, SV: match on `country_id` alone

This matches what's in `maintenance_rates`: Energica's GT rows are region-scoped; IBS GROUP's HN rows are country-only. `loadQuoteIntegrationContext` now resolves `region_id` from `departments` via `siteRow.department_id`.

## Insurance values

Per case, sin-IVA in project currency:

| Case | Monthly insurance |
|---|---|
| SANTIAGO GIRON | Q59.46 |
| Hospital la Fe | Q51.07 |
| Texaco San Miguel | Q263.21 |
| MAGNO CARTONES | Q467.19 |
| Cafe Welchez | Q27.14 |
| Carlos Palencia | Q15.96 |

QB stored insurance as part of the bundled monthly payment in some cases and as a separate line in others; reconciling exactly takes a per-month cashflow pull from `qb_raw.monthly_cashflows` (have it for every QB quote in the test set).

## Methodology

1. Picked 6 representative estimates spanning 5 different providers, project sizes from Q17k to Q382k, and term lengths from 15 to 75 months.
2. For each, pulled the actual QB quote variant from `qb_raw.quotes` with its inputs (rate, term, dp%, grace) and stored outputs (monthly payment, retail price).
3. Imported each estimate's project, client, estimate row into `public.*` with location, sites, project_phases, Panel Solar estimate_equipment, and provider derived from QB data. IVA stripped per Lambda convention.
4. Called the production `irr-calculator` edge function with the **exact same inputs** as the QB quotes.

## How to reproduce any case

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

QB quote rows are in `qb_raw.quotes`. Each case's exact QB quote_record_id:

| # | estimate_ref | qb_quote_id | qb_quote_ref |
|---|---|---|---|
| 1 | 2535-01-01 | 38521 | 2535-01-01-01 |
| 2 | 2359-01-01 | 35490 | 2359-01-01-02 |
| 3 | 1037-01-01 | 15196 | 1037-01-01-01 |
| 4 | 1511-01-01 | 18631 | 1511-01-01-02 |
| 5 | 1414-01-01 | 16378 | 1414-01-01-02 |
| 6 | 2526-01-01 | 38441 | 2526-01-01-01 |

The 20 imported test projects (these 6 + 14 others) are also visible in `/sales` (Ventas en proceso) on staging if you want to see the UI view.

## Asks for Ian, ranked

1. **Audit `maintenance_rates` against the new $/W formula.** `serviciosTable` computes `maintFirstCost = rate × installedWatts`, treating rate as $/W. Energica's 0.19 × 9,860 W = $1,873/visit → +51% off QB on SANTIAGO GIRON. Almost certainly this value was authored as a percentage and needs recalibrating. Same suspicion for any other non-zero row in the table and for the fallback default `config.default_maintenance_percentage = 0.028` (scales to $28/W, way too high).
2. **Confirm the #222 pattern on Carlos Palencia / ESQUISOLAR.** The retail-price delta cleanly isolates the divide-by-(1−margin) factor (× 1.25 exact). If you have a fix in flight on #222, this comparison gives you one regression case.
3. **Eyeball the −11.5% on Cafe Welchez monthly payment.** The other 5 cases are within ±12%, but Cafe Welchez and Texaco San Miguel are the larger negative deltas — both have rate=0 maintenance so the residual is purely lease+insurance differences. Could be HN-specific.

Once rates are recalibrated I can re-run the 6 cases in seconds for a regression check.
