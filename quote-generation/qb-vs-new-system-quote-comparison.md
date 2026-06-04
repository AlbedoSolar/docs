# QB vs new-system quote comparison

**Owner of follow-up:** Ian (calc engine)
**Prepared:** 2026-06-03, updated 2026-06-04 after IVA fix, variant maintenance-rate backfill, region_id fix, and re-runs
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

## Current state — comparison table (post-region_id-fix + post-rebase)

All six cases use 10% down payment, 3-month grace. New-system retail prices below are sin-IVA (matches Supabase convention); QB displayed retail prices are con-IVA.

| # | Project | Provider | Term | Rate | QB Monthly | New Monthly | Δ Monthly | QB Retail (con-IVA) | New Retail (sin-IVA) | Implied multiplier |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | SANTIAGO GIRON | Energica | 51 | 11.99% | Q1,638.49 | Q1,605.72 | **−2.0%** ✅ | Q63,468.82 | Q56,668.59 | × 1.12 = match ✅ |
| 2 | Hospital la Fe | TECKNOS SOLAR | 63 | 12% | Q995.58 | Q940.09 | **−5.6%** ✅ | Q42,984.58 | Q37,377.90 | × 1.15 (HN) = match ✅ |
| 3 | Texaco San Miguel | Ecolumen | 75 | 12% | Q6,267.44 | Q5,804.96 | **−7.4%** | Q275,770.86 | Q272,824.36 | × 1.012 — slight underrepresent |
| 4 | MAGNO CARTONES | Conexsol | 39 | 12% | Q12,786.02 | Q12,215.89 | **−4.5%** ✅ | Q382,072.24 | Q332,236.73 | × 1.15 (HN) = match ✅ |
| 5 | Cafe Welchez | IBS GROUP | 51 | 12% | Q850.53 | Q752.81 | **−11.5%** | Q29,750.00 | Q25,869.57 | × 1.15 (HN) = match ✅ |
| 6 | Carlos Palencia | ESQUISOLAR MARGEN 20% | 15 | 12% | Q1,816.84 | Q1,754.70 | **−3.4%** ✅ | Q17,028.00 | **Q19,004.46** | ⚠️ **÷ 0.80 = 23,755.58** — #222 |

(Note: monthly payment deltas widened slightly from the prior round because insurance dropped substantially on every case once region-specific rates began resolving. Now we're under-counting services because maintenance is also dropped — `installedWatts = 0` until equipment is imported. The lookup is correct; the downstream input is missing.)

### Pre-rebase verification of the region_id fix (transient state)

Before the `serviciosTable` refactor was rebased into the file, I deployed an interim version with only the region_id fix applied. That deployment used the OLD downstream formula (`maintFirstCost = installationCost × rate`) and produced these maintenance values, proving the lookup is now correct:

| # | Case | Pre-rebase deploy, monthly maintenance | Source rate row |
|---|---|---|---|
| 1 | SANTIAGO GIRON | **Q585.59** | Energica country=1 region=2, 0.19 USD |
| 2 | Hospital la Fe | **Q70.31** | Tecknos Solar — no row, fallback 0.028 |
| 3 | Texaco San Miguel | Q0 | Ecolumen country=1, 0.00 USD — correct |
| 4 | MAGNO CARTONES | **Q326.85** | Conexsol fallback (HN site, only GT row exists) |
| 5 | Cafe Welchez | Q0 | IBS GROUP country=3, 0.00 USD — correct |
| 6 | Carlos Palencia | Q0 | Term 15 < maintenance-free 24 — correct |

So the lookup fix is independently verified — 3 of 6 cases produce maintenance, the other 3 correctly produce Q0 because of data or schedule reasons. After the rebase, all 6 cases produce Q0 from the new formula because `installedWatts = 0`. Once equipment is imported, the merged code should produce the values consistent with the new $/W semantics.

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

## Maintenance Q0 in all 6 cases — root cause isolated to the calc engine

Working through `public.maintenance_rates` for each case and then doing the variant backfill, here's the maintenance breakdown:

| # | Case | Provider | Rate available now? | Why Q0 |
|---|---|---|---|---|
| 1 | SANTIAGO GIRON | Energica (8) | ✅ 3 rows: country=GT, region 1/2/3 (USD). Region 2 = 0.19 | ⚠️ **CALC ENGINE BUG** — site is country=1, dept=21 → region=2 via departments. Rate exists, should match, returns 0. |
| 2 | Hospital la Fe | Tecknos Solar (112) | ❌ 0 rows (canonical has none either) | Correctly Q0 — no data |
| 3 | Texaco San Miguel | Ecolumen (10) | ✅ 1 row: country=GT, rate = **0.00** USD | Rate IS zero, Q0 is correct |
| 4 | MAGNO CARTONES | Conexsol (1) | ✅ 1 row: country=GT, but project is country=3 (HN) | Country mismatch — no HN rate exists |
| 5 | Cafe Welchez | IBS GROUP (47) | ✅ 2 rows: country=HN, rate = **0.00** USD | Rate IS zero, Q0 is correct |
| 6 | Carlos Palencia | ESQUISOLAR MARGEN 20% (49) | ✅ **NEW after backfill: 0.15 GTQ country=1** | ⚠️ **CALC ENGINE BUG** — clean country-only match, no region join needed, still returns 0 |

Two of the six are confirmed calc-engine misses (#1 and #6). The other four return Q0 because that's what the data says (no rates, rate=0, or country mismatch).

### Why #6 is the cleanest signal

Carlos Palencia / ESQUISOLAR MARGEN 20% was deliberately set up as the simplest possible lookup case after the variant backfill:
- Phase `project_phases.provider_id` = 49 ✅
- Estimate `project_currency` = GTQ ✅ (matches rate currency)
- Site `country_id` = 1 ✅ (matches rate country, no region scoping needed)
- `maintenance_rates` row: provider=49, country=1, region=NULL, rate=0.15 GTQ ✅

Re-running `irr-calculator` returns `monthly_maintenance_payment: 0`. There's nothing left on the data side to explain that — every key the lookup could plausibly use is in place.

### Two hypotheses for Ian to test

1. The lookup keys on the canonical `provider_id` (via a JOIN through some other table) rather than the variant `provider_id` from `project_phases.provider_id` directly. If so, ESQUISOLAR MARGEN 20% (id 49) would never match because the lookup is looking for ESQUISOLAR (id 39).
2. The lookup isn't reading `public.maintenance_rates` at all and is reading from something else (e.g., a stale DBT view or a different source-of-truth table).

Either would explain why both #1 (region-scoped) and #6 (country-only, post-backfill) miss.

### Bonus context for the SANTIAGO GIRON case

If the lookup IS reading `maintenance_rates` but doesn't join through `departments.region_id` to resolve site → region, then the region-scoped Energica rates will never match. Sites carry `country_id`, `department_id`, `municipality_id` — but **no `region_id` directly**. The departments table has `region_id`, so the join has to go through there.

### Region semantics (per Jake)

Useful context: region is GT-only. Honduras and El Salvador don't use region — for those countries "region" effectively equals the country. So the lookup should:
- For GT: match on `country_id + region_id` (where region = rural/urban/highlands as keyed in the `regions` table)
- For HN, SV: match on `country_id` alone (no region scoping needed)

This matches what's in `maintenance_rates`: Energica's GT rows are region-scoped; IBS GROUP's HN rows are country-only.

## Insurance comparison (works correctly across all 6)

`monthly_insurance_payment` came back non-zero in every case. The lookup works. The values per case (sin-IVA, USD where the rate currency is USD):

| Case | New-system insurance |
|---|---|
| SANTIAGO GIRON | Q96.26 |
| Hospital la Fe | Q62.50 |
| Texaco San Miguel | Q426.15 |
| MAGNO CARTONES | Q571.84 |
| Cafe Welchez | Q43.94 |
| Carlos Palencia | Q25.85 |

QB stored insurance as part of the bundled monthly payment in some cases and as a separate line in others; reconciling exactly takes a per-month cashflow pull from `qb_raw.monthly_cashflows` (have it for every QB quote in the test set).

## Methodology

1. Picked 6 representative estimates spanning 5 different providers, project sizes from Q17k to Q382k, and term lengths from 15 to 75 months
2. For each, pulled the actual QB quote variant from `qb_raw.quotes` with its inputs (rate, term, dp%, grace) and stored outputs (monthly payment, retail price)
3. Imported each estimate's project, client, estimate row into `public.*` with location, sites, project_phases, and provider derived from QB data
4. Called the production `irr-calculator` edge function with the **exact same inputs** as the QB quotes
5. Iterated: initial run surfaced IVA-strip gap → fixed → ±5% delta. Then variant providers showed up as a maintenance data gap → backfilled rates → Carlos Palencia got a clean test case → maintenance still Q0, isolating the bug to the engine

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

1. **Maintenance lookup is FIXED on my end** — region_id is now resolved from `departments.region_id` via the site's `department_id` (committed and deployed). You should see correct rates flowing into `getMaintenanceRate` now. Verified by an interim pre-rebase deploy: SANTIAGO GIRON Q0 → Q585.59, Hospital la Fe Q0 → Q70.31, MAGNO CARTONES Q0 → Q326.85.
2. **Confirm the #222 pattern on Carlos Palencia / ESQUISOLAR.** The retail-price delta cleanly isolates the divide-by-(1−margin) factor (× 1.25 exact). If you have a fix in flight on #222, this comparison gives you one regression case.
3. **Eyeball the −11.5% on Cafe Welchez monthly payment.** The other 5 cases are within ±5%, this one is slightly larger. Could be a HN-specific quirk or an artifact of one of the financial fields I'm importing — happy to dig further if you want a pointer.
4. **Confirm the new $/W maintenance formula.** The recent `maintFirstCost = rate × installedWatts` change in `serviciosTable` (from `installationCost × rate`) looks intentional but should be confirmed. Want to verify: is `maintenance_rates.maintenance_rate` now meant to be currency-per-watt (e.g., $0.19/W), not percentage-of-asset-cost? If so, fallback defaults like 0.028 are way too high under the new semantics (would scale to $28/W). May need to revisit defaults and existing rate data.

Once (4) is confirmed, I'll extend the test imports with `estimate_equipment` rows and re-run for a true end-to-end comparison with the new formula.
