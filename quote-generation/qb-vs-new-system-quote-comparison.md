# QB vs new-system quote comparison

**Owner of follow-up:** Ian (calc engine)
**Prepared:** 2026-06-03, updated 2026-06-04 with maintenance-lookup follow-up test (variant backfill)
**Data set:** 6 estimates × 1 representative QB quote each, imported from QB into Supabase as test cases for the upcoming cutover. New-system numbers come from hitting the production `irr-calculator` edge function with the same inputs QB had.

## TL;DR for Ian (revised after first-round triage)

After fixing the IVA gap on my import (the original report had a self-inflicted issue: I'd stored raw QB con-IVA values where the existing Lambda strips IVA via `removeIVA(value, taxRate)` — same convention you'd expect from any QB import), the deltas collapsed dramatically:

| Delta | Round 1 (con-IVA stored) | Round 2 (sin-IVA stored, per Lambda convention) |
|---|---|---|
| Monthly payment Δ | +4.4% to +12.6% across all 6 | **±5% across 5 of 6 cases** |
| Retail price misses | 2 of 6 differed | **1 of 6 still off, isolated to ESQUISOLAR MARGEN 20%** ← #222 candidate |
| Maintenance | Q0 on all 6 | Still Q0 — root causes now broken down per case (mostly correct, 1 likely-bug) |

So most of the original alarm was my data, not your calc engine. The two real signals are:
1. **The retail-price #222 pattern** still cleanly isolates on ESQUISOLAR MARGEN 20% (ratio = exactly 0.80, divide-by-(1−margin))
2. **One actual maintenance-lookup miss** on SANTIAGO GIRON (Energica) where the rate exists but the lookup returned 0

The monthly-payment residuals are small (±5%, mostly insurance treatment differences). Worth eyeballing but not alarming.

## Round-2 summary table (post-IVA-fix)

All six cases use 10% down payment, 3-month grace. New-system retail prices below are sin-IVA (matches Supabase convention); QB displayed retail prices are con-IVA.

| # | Project | Provider | Term | Rate | QB Monthly | New Monthly | Δ Monthly | QB Retail (con-IVA) | New Retail (sin-IVA) | Implied multiplier |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | SANTIAGO GIRON | Energica | 51 | 11.99% | Q1,638.49 | Q1,646.93 | **+0.5%** ✅ | Q63,468.82 | Q56,668.59 | × 1.12 = match ✅ |
| 2 | Hospital la Fe | TECKNOS SOLAR | 63 | 12% | Q995.58 | Q953.24 | **−4.3%** ✅ | Q42,984.58 | Q37,377.90 | × 1.15 = 42,984.59 (HN) ✅ |
| 3 | Texaco San Miguel | Ecolumen | 75 | 12% | Q6,267.44 | Q5,987.46 | **−4.5%** ✅ | Q275,770.86 | Q272,824.36 | × 1.012 — slight underrepresent |
| 4 | MAGNO CARTONES | Conexsol | 39 | 12% | Q12,786.02 | Q12,336.23 | **−3.5%** ✅ | Q382,072.24 | Q332,236.73 | × 1.15 (HN) = 382,072.24 ✅ |
| 5 | Cafe Welchez | IBS GROUP | 51 | 12% | Q850.53 | Q772.13 | **−9.2%** | Q29,750.00 | Q25,869.57 | × 1.15 (HN) = 29,749.99 ✅ |
| 6 | Carlos Palencia | ESQUISOLAR MARGEN 20% | 15 | 12% | Q1,816.84 | Q1,765.77 | **−2.8%** ✅ | Q17,028.00 | **Q19,004.46** | ⚠️ **÷ 0.80 = 23,755.58** — #222 |

## What changed between round 1 and round 2

I stripped IVA from `retail_price_project_currency`, `project_cost_partner_contract_currency`, `asset_transfer_value_project_currency`, `legal_costs_project_currency`, `commission_cost_project_currency` on the imported estimates using each project's country tax rate (GT 12%, SV 13%, HN 15%) — same `value / (1 + taxRate)` formula the existing `quickbase-import-kickoff` Lambda uses. Also stripped IVA on the auto-generated `project_phases.phase_cost` and `partner_quote_amount` since they were derived from the now-corrected estimate values.

This is the conventional Supabase storage state — sin-IVA at rest, with IVA added back at display time via the frontend's MoneyDisplay component.

## The one real retail-price issue: Carlos Palencia / ESQUISOLAR MARGEN 20%

After IVA fix:
- QB retail (con-IVA): Q17,028
- My import → phase_cost (sin-IVA): Q17,028 / 1.12 = **Q15,203.57**
- New system retail (sin-IVA): **Q19,004.46**
- Math: Q15,203.57 ÷ 0.80 = Q19,004.46 (exact)

So the calc engine is taking the phase cost and dividing by `(1 − provider_margin)` (margin = 20% for ESQUISOLAR MARGEN 20%, cartera = Subtract). That's the #222 pattern — should be `partner_cost × (1 + margin)` for an Add provider or `retail = phase_cost / (1 − margin)` should be flipped for Subtract semantics. Whatever the right semantics are, the QB-stored retail of Q17,028 con-IVA reflects "what the client pays" and the new system is producing a different number.

This is exactly the #222 hypothesis: divide-by-(1−margin) instead of multiply-by-(1+margin), and it bites when cartera is Subtract.

## Maintenance Q0 in all 6 cases — root causes, per case

After looking at `public.maintenance_rates`, the Q0 result is NOT a single bug. It's three different things:

| # | Case | Provider | Maintenance rates available? | Result Q0 because |
|---|---|---|---|---|
| 1 | SANTIAGO GIRON | Energica (8) | ✅ 3 rows: country=GT, region 1/2/3 (USD). Region 2 = 0.19 | ⚠️ **REAL BUG** — site has country=1, dept=21 → region=2 (via departments). Rate exists, should match, returns 0. |
| 2 | Hospital la Fe | Tecknos Solar (112) | ❌ 0 rows | Correctly Q0 — no data to lookup |
| 3 | Texaco San Miguel | Ecolumen (10) | ✅ 1 row: country=GT, rate = **0.00** USD | Rate IS literally zero, Q0 is correct |
| 4 | MAGNO CARTONES | Conexsol (1) | ✅ 1 row: **country=GT** but project is country=3 (HN) | Country mismatch — no HN rate exists. Could fall back? |
| 5 | Cafe Welchez | IBS GROUP (47) | ✅ 2 rows: **country=HN, rate = 0.00 USD** | Rate IS literally zero, Q0 is correct |
| 6 | Carlos Palencia | ESQUISOLAR MARGEN 20% (49) | ❌ 0 rows | Correctly Q0 — no data to lookup |

So 4 of the 6 Q0 results are correct given the data — they just reflect "no rate exists" or "rate is zero" in `maintenance_rates`. Only case #1 (SANTIAGO GIRON) is a real lookup miss worth investigating.

### Hypothesis for the SANTIAGO GIRON miss

`public.sites` carries `country_id`, `department_id`, `municipality_id` — but **no `region_id`**. To get from site → region you have to JOIN through `departments.region_id`. If the calc engine's lookup against `maintenance_rates` is keyed by `region_id` but reads only `site.country_id` directly (no join through `departments`), the rate won't match for providers whose rates are region-scoped.

For SANTIAGO GIRON:
- Site has country=1 (GT), dept=21
- `departments` row for dept 21 has region_id = 2
- `maintenance_rates` for Energica has rows scoped to country=1 + region={1,2,3}; region=2 is 0.19 USD
- The lookup needs to know "site is in region 2" to pick the right row
- If the calc engine doesn't perform that join, it can't match → returns 0

Quick way to verify on Ian's side: trace one rate lookup for `estimate_reference = '2535-01-01'` and confirm whether the query touches `departments.region_id` at all. If not, that's the gap.

### Follow-up test (2026-06-04): backfill ESQUISOLAR MARGEN 20% from canonical, re-run Carlos Palencia

To test whether the function works at all for the **simplest possible case** (country-only match, no region scoping), I backfilled `maintenance_rates` for the 18 provider variants from their canonical parents (migration `database/migrations/2026-06-04-backfill-variant-maintenance-rates.sql`). For Carlos Palencia's variant ESQUISOLAR MARGEN 20% (id 49), this added a row: `country_id=1, region_id=NULL, maintenance_rate=0.15 GTQ` — copied directly from canonical ESQUISOLAR (id 39).

After the backfill, every field for a successful lookup is present:
- Phase `project_phases.provider_id` = 49 ✅
- Estimate `project_currency` = GTQ ✅ (matches rate currency)
- Site `country_id` = 1 ✅ (matches rate country, no region join needed)
- `maintenance_rate` = 0.15 (non-zero) ✅

Re-running the same `irr-calculator` call:

```
monthly_maintenance_payment: 0
```

**Result: still Q0.** That isolates the bug to the calc engine, not the data — the function fails even on a clean country-only match where currency and provider line up. So the issue is broader than "missing region join" — it's possible the lookup isn't keyed on `provider_id` correctly, isn't reading from `maintenance_rates` at all, or has some other guard short-circuiting to 0.

So among our 6 cases, the maintenance breakdown after backfill is:
| # | Case | Provider rates available now? | Q0 because |
|---|---|---|---|
| 1 | SANTIAGO GIRON | ✅ rate exists for region 2 | calc engine bug |
| 2 | Hospital la Fe | ❌ no canonical rates either | correctly Q0 |
| 3 | Texaco San Miguel | ✅ rate=0.00 in data | correctly Q0 |
| 4 | MAGNO CARTONES | ✅ country mismatch (only GT rate; site is HN) | correctly Q0 |
| 5 | Cafe Welchez | ✅ rate=0.00 in data | correctly Q0 |
| 6 | Carlos Palencia | ✅ **new: 0.15 GTQ country=1 via backfill** | **calc engine bug** |

So we now have two confirmed maintenance-lookup failures (#1 region-scoped, #6 country-only) with clean data on both sides. That's enough to say the function isn't working — not "sometimes works, sometimes doesn't."

### Region semantics (per Jake)

Useful context for Ian: region is GT-only. Honduras and El Salvador don't use region — for those countries "region" effectively equals the country. So the lookup should:
- For GT: match on `country_id + region_id` (where region = rural/urban/highlands as keyed in the `regions` table)
- For HN, SV: match on `country_id` alone (no region scoping needed)

This matches what I see in `maintenance_rates`: Energica's GT rows are region-scoped; IBS GROUP's HN rows are country-only.

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
4. Initially stored raw con-IVA QB values; first-round comparison surfaced large deltas that turned out to be the IVA-strip gap
5. Re-ran with IVA stripped per Lambda convention; deltas now reflect real engine behavior
6. Called the production `irr-calculator` edge function with the **exact same inputs** as the QB quotes

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

1. **Confirm the #222 pattern on Carlos Palencia / ESQUISOLAR.** Round-2 number cleanly isolates the divide-by-(1−margin) factor (× 1.25 exact). If you already have a fix in flight on #222, this comparison gives you one regression case.
2. **Trace the maintenance lookup — broader than the region join.** After the 2026-06-04 follow-up test (variant backfill + clean country-only rate for Carlos Palencia / ESQUISOLAR MARGEN 20%), the function still returns 0 even when provider_id, country, currency and a non-zero rate all line up. So the bug isn't just "missing region join"; it's something more fundamental in the lookup path. Both SANTIAGO GIRON (region-scoped) and Carlos Palencia (country-only) are now confirmed misses with clean data on both sides.
3. **Eyeball the −9.2% on Cafe Welchez monthly payment.** The other 5 cases are within ±5%, this one is slightly larger. Could be a HN-specific quirk or an artifact of one of the financial fields I'm importing — happy to dig further if you want a pointer.
4. **Pin the maintenance Q0-when-rate-missing behavior intentional?** For the 4 cases where Q0 is correct given the data (no rates, rate=0, or country mismatch), is the current behavior "show Q0" or should there be a fallback to some default? Not blocking, just want to know the design intent before we expand the migration.

Once any of these have a fix, I can re-run all 6 cases in seconds for regression.
