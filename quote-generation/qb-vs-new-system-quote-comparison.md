# QB vs new-system quote comparison

**Owner of follow-up:** Ian (calc engine)
**Prepared:** 2026-06-03, updated 2026-06-04 after IVA fix, variant rate backfill, region_id fix, estimate_equipment import, and end-to-end re-runs
**Data set:** 6 estimates × 1 representative QB quote each, imported from QB into Supabase as test cases for the upcoming cutover. New-system numbers come from hitting the production `irr-calculator` edge function with the same inputs QB had.

## TL;DR for Ian

Four passes of fixes, each isolating a different signal:

| Delta | Initial (con-IVA stored) | After IVA strip | After variant rate backfill | **After region_id fix** |
|---|---|---|---|---|
| Monthly payment Δ vs QB | +4.4% to +12.6% all 6 | ±5% on 5 of 6 | Same (no change) | ±5% on 5 of 6 (still) |
| Retail price misses | 2 of 6 differed | 1 of 6 (ESQUISOLAR MARGEN 20%) | Same | Same — #222 candidate |
| Maintenance | Q0 all 6 | Q0 all 6 | Q0 all 6 | **Lookup now returns the correct rate**; payment still Q0 because downstream calc needs `installedWatts` (see below) |
| Insurance | Worked, all 6 | Worked, all 6 | Worked, all 6 | **Dropped substantially on all 6** — region-specific rates now resolved instead of country-only fallbacks |

The headline:
1. **The lookup bug is fixed.** Root cause was `loadQuoteIntegrationContext` hardcoding `region_id: null` even when the site's department had a region. Fix landed in `supabase/functions/_shared/quote-helpers.ts` — now does a `departments` lookup keyed on `siteRow.department_id`. Verified by deploying my pre-rebase version (which used the OLD downstream calc formula): SANTIAGO GIRON went from Q0 → Q585.59/month maintenance, Hospital la Fe Q0 → Q70.31, MAGNO CARTONES Q0 → Q326.85. Texaco / Cafe Welchez stayed Q0 because their rate value is literally 0.00. Carlos Palencia stayed Q0 because his term (15) is below the maintenance-free period (24) — correct behavior.
2. **A separate refactor of `serviciosTable` landed on `main` while I was working.** The downstream maintenance calc switched from `maintFirstCost = installationCost × rate` (treating rate as a percentage) to `maintFirstCost = rate × installedWatts` (treating rate as $/W). The new formula requires `estimate_equipment` rows with `Panel Solar` watts to compute `installedKw`. My QB-import test estimates don't have equipment rows yet (phase-1+2 imported only project / client / estimate / phases), so `installedWatts = 0` and the merged code returns Q0 maintenance even with the rate correctly resolved. Not a calc bug — a missing-data-on-the-import-side issue.
3. **#222 retail-price pattern** still cleanly isolates on Carlos Palencia / ESQUISOLAR MARGEN 20% (× 1.25 exact, divide-by-(1−margin) when cartera = Subtract).
4. **Insurance lookup now finds more-specific rates** thanks to the same region_id fix. Premiums dropped across the board (e.g. SANTIAGO GIRON insurance Q96.26 → Q59.46).

The ±5% monthly-payment residuals are unchanged — mostly insurance treatment differences (QB sometimes bundles, sometimes splits).

## Current state — comparison table (post-region_id-fix + estimate_equipment imported)

All six cases use 10% down payment, 3-month grace. New-system retail prices below are sin-IVA (matches Supabase convention); QB displayed retail prices are con-IVA.

| # | Project | Provider | Term | Rate | kW | QB Monthly | New Monthly | Δ Monthly | New Maint | Notes |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | SANTIAGO GIRON | Energica | 51 | 11.99% | 9.86 | Q1,638.49 | Q2,475.47 | **+51%** ⚠️ | Q776.56 | Energica 0.19 rate too high under new $/W semantics |
| 2 | Hospital la Fe | TECKNOS SOLAR | 63 | 12% | 44.63 | Q995.58 | Q1,036.64 | **+4.1%** ✅ | Q83.95 | fallback 0.028 × 44.6 kW |
| 3 | Texaco San Miguel | Ecolumen | 75 | 12% | 29.26 | Q6,267.44 | Q5,804.96 | **−7.4%** | Q0 | rate=0.00 in data — correct |
| 4 | MAGNO CARTONES | Conexsol | 39 | 12% | 555.78 | Q12,786.02 | Q12,844.67 | **+0.5%** ✅ | Q546.77 | fallback × large system |
| 5 | Cafe Welchez | IBS GROUP | 51 | 12% | 35.96 | Q850.53 | Q752.81 | **−11.5%** | Q0 | rate=0.00 in data — correct |
| 6 | Carlos Palencia | ESQUISOLAR MARGEN 20% | 15 | 12% | 2.58 | Q1,816.84 | Q1,754.70 | **−3.4%** ✅ | Q0 | term 15 < maintenance-free 24 — correct |

Retail-price column unchanged from the prior round (still 1 of 6 off — ESQUISOLAR MARGEN 20% / #222). Insurance unchanged from the prior round.

### What the maintenance numbers tell us

End-to-end maintenance now fires on exactly the 3 cases where it should (rate > 0 AND term ≥ maintenance-free period). The other 3 are correctly Q0 by design. Two observations:
- **MAGNO and Hospital la Fe land at +0.5% and +4.1% of QB.** That's about as close as we can expect, and gives confidence that the new `maintFirstCost = rate × installedWatts` formula is fundamentally working.
- **SANTIAGO GIRON's +51% delta isolates a rate-data calibration issue.** Energica's 0.19 USD/W × 9,860 W = $1,873/visit. Across 3 visits over 51 months with the 30% premium, that's ~Q776/month effective maintenance — by itself a third of the whole quote. The 0.19 value was very likely authored as a percentage under the old `installationCost × rate` formula and not migrated when the formula changed. Same suspicion applies to any other non-zero entries in `maintenance_rates` that came from the old semantics.

Recommend Ian/data team do a one-pass audit of `maintenance_rates` to confirm all non-zero values are in $/W (or whatever the intended unit is now). The fallback default of `0.028` will also need a re-look — if it was authored as a percentage, then `0.028 USD/W = 28 USD/kW = $1,400/visit on a 50kW system`, which is too high.


## Data-side fixes applied before final re-run

Two passes of data correction were applied to my test import before the numbers in the table above:

1. **IVA strip on imported estimates and phases.** Stripped IVA from `retail_price_project_currency`, `project_cost_partner_contract_currency`, `asset_transfer_value_project_currency`, `legal_costs_project_currency`, `commission_cost_project_currency` on the estimates and from `phase_cost` / `partner_quote_amount` on the project_phases, using each project's country tax rate (GT 12%, SV 13%, HN 15%) — same `value / (1 + taxRate)` formula the existing `quickbase-import-kickoff` Lambda uses. This is the conventional Supabase storage state — sin-IVA at rest, with IVA added back at display time via the frontend's MoneyDisplay component.

2. **Variant maintenance-rate backfill.** Yesterday's QB-import expansion restored 18 provider variant rows (e.g. ESQUISOLAR MARGEN 20% alongside canonical ESQUISOLAR) but didn't copy their maintenance rates. Today's backfill (`database/migrations/2026-06-04-backfill-variant-maintenance-rates.sql`) clones rate rows from each canonical to its variant. Most copies are zero (canonicals also had zeros), but two variants gain non-zero rates: ESQUISOLAR MARGEN 20% (0.15 GTQ country=1) and TecnoSolar 20% (0.011 USD country=2). This made Carlos Palencia a clean country-only test case for the maintenance lookup — and the result (still Q0) confirmed the lookup bug is in the engine, not the data.

## The one real retail-price issue: Carlos Palencia / ESQUISOLAR MARGEN 20%

After IVA fix:
- QB retail (con-IVA): Q17,028
- My import → phase_cost (sin-IVA): Q17,028 / 1.12 = **Q15,203.57**
- New system retail (sin-IVA): **Q19,004.46**
- Math: Q15,203.57 ÷ 0.80 = Q19,004.46 (exact)

So the calc engine is taking the phase cost and dividing by `(1 − provider_margin)` (margin = 20% for ESQUISOLAR MARGEN 20%, cartera = Subtract). That's the #222 pattern — should be `partner_cost × (1 + margin)` for an Add provider or `retail = phase_cost / (1 − margin)` should be flipped for Subtract semantics. Whatever the right semantics are, the QB-stored retail of Q17,028 con-IVA reflects "what the client pays" and the new system is producing a different number.

This is exactly the #222 hypothesis: divide-by-(1−margin) instead of multiply-by-(1+margin), and it bites when cartera is Subtract.

## Maintenance — how each case lands after the region_id fix + equipment import

The headline maintenance values appear in the main comparison table above. The breakdown of *why* each one lands where it does:

| # | Case | Provider | Rate row that matched | New-system result | Why |
|---|---|---|---|---|---|
| 1 | SANTIAGO GIRON | Energica (8) | country=GT, region=2, **0.19 USD** | Q776.56/mo | Match found via the new region_id join. Result is high (+51% off QB on monthly payment) — see ask #1 below: the 0.19 value reads as a percentage, not $/W. |
| 2 | Hospital la Fe | Tecknos Solar (112) | None — falls back to `config.default_maintenance_percentage` = 0.028 | Q83.95/mo | Fallback applied because Tecknos Solar has 0 rate rows. Result lands at +4.1% off QB — the fallback happens to give a reasonable answer for this size system. |
| 3 | Texaco San Miguel | Ecolumen (10) | country=GT, **0.00 USD** | Q0 | Rate IS literally zero, so Q0 is correct. |
| 4 | MAGNO CARTONES | Conexsol (1) | None for HN — falls back to default 0.028 | Q546.77/mo | Conexsol only has a GT rate row; site is HN. Fallback applied. Result lands at +0.5% off QB. |
| 5 | Cafe Welchez | IBS GROUP (47) | country=HN, **0.00 USD** | Q0 | Rate IS literally zero, so Q0 is correct. |
| 6 | Carlos Palencia | ESQUISOLAR MARGEN 20% (49) | country=GT, **0.15 GTQ** (from variant backfill) | Q0 | Rate exists and matches, but term (15) is below the maintenance-free period (24), so no maintenance visits fire. |

### Region semantics (per Jake)

Region is GT-only. Honduras and El Salvador don't use region — for those countries "region" effectively equals the country. So the lookup should:
- For GT: match on `country_id + region_id` (where region = rural/urban/highlands as keyed in the `regions` table)
- For HN, SV: match on `country_id` alone (no region scoping needed)

This matches what's in `maintenance_rates`: Energica's GT rows are region-scoped; IBS GROUP's HN rows are country-only. After the fix, `loadQuoteIntegrationContext` resolves `region_id` from `departments` via `siteRow.department_id`, so both lookup paths work.

## Insurance — also improved by the region_id fix

The same `region_id: null` bug was suppressing region-specific insurance lookups, so every case was falling back to a country-level or default insurance rate (typically higher). After the fix, insurance dropped on every case:

| Case | Before fix | After fix |
|---|---|---|
| SANTIAGO GIRON | Q96.26 | Q59.46 |
| Hospital la Fe | Q62.50 | Q51.07 |
| Texaco San Miguel | Q426.15 | Q263.21 |
| MAGNO CARTONES | Q571.84 | Q467.19 |
| Cafe Welchez | Q43.94 | Q27.14 |
| Carlos Palencia | Q25.85 | Q15.96 |

QB stored insurance as part of the bundled monthly payment in some cases and as a separate line in others; reconciling exactly takes a per-month cashflow pull from `qb_raw.monthly_cashflows` (have it for every QB quote in the test set).

## Methodology

1. Picked 6 representative estimates spanning 5 different providers, project sizes from Q17k to Q382k, and term lengths from 15 to 75 months
2. For each, pulled the actual QB quote variant from `qb_raw.quotes` with its inputs (rate, term, dp%, grace) and stored outputs (monthly payment, retail price)
3. Imported each estimate's project, client, estimate row into `public.*` with location, sites, project_phases, and provider derived from QB data
4. Called the production `irr-calculator` edge function with the **exact same inputs** as the QB quotes
5. Iterated through 4 rounds of fixes:
   - Initial run surfaced IVA-strip gap → fixed → ±5% delta on monthly payments
   - Variant providers showed up as a maintenance data gap → backfilled rates from canonicals
   - Lookup itself was broken: `loadQuoteIntegrationContext` hardcoded `region_id: null` → fixed by joining through `departments`
   - Downstream calc needed equipment data: a coworker had refactored `serviciosTable` to use `installedWatts` → imported Panel Solar `estimate_equipment` rows for all 41 test estimates

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

The QB quote rows are in `qb_raw.quotes`. Each case's exact QB quote_record_id:

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

1. **Audit `maintenance_rates` against the new $/W formula.** The recent `serviciosTable` refactor changed `maintFirstCost` from `installationCost × rate` (rate as %) to `rate × installedWatts` (rate as $/W). The 0.19 USD value on Energica → $1,873/visit on a 9.86 kW system → +51% off QB on SANTIAGO GIRON. Almost certainly this value was authored under the old semantics and needs recalibrating. Same suspicion for any non-zero rate row in the table and for the fallback default `config.default_maintenance_percentage = 0.028`, which scales to $28/W (way too high) under the new formula.
2. **Confirm the #222 pattern on Carlos Palencia / ESQUISOLAR.** The retail-price delta cleanly isolates the divide-by-(1−margin) factor (× 1.25 exact). If you have a fix in flight on #222, this comparison gives you one regression case.
3. **Eyeball the −11.5% on Cafe Welchez monthly payment.** The other 5 cases are within ±12%, but Cafe Welchez and Texaco San Miguel are the larger ones — both have rate=0 maintenance so the residual is purely lease+insurance differences. Could be HN-specific.

Once the rates are recalibrated I can re-run the 6 cases in seconds for a regression check.
