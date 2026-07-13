# Data Consistency Review — July 2026

Systematic review of the Supabase production data for inconsistencies, run 2026-07-12.
Method: every rule the data should obey is a numbered invariant, checked with SQL,
violations recorded here. Fixes land via migration + CHANGELOG; each confirmed check
should graduate to a dbt test, a DB constraint/trigger, or a `mart_data_health` row.

Legend: ✅ clean · 🔴 investor-facing violation · 🟠 app-facing · 🟡 internal/latent ·
⚪ informational/benign · 🔍 needs human review

## A. State implies field — ALL CLEAN ✅

| # | Invariant | Status |
|---|---|---|
| A1 | signed quote ⇒ contract_signing_date not null | ✅ (Texaco fix holds) |
| A2 | signed original ⇒ has cash flows | ✅ |
| A3 | contract signed ⇒ quote signed | ✅ |
| A4 | signed ⇒ retail > 0 | ✅ (2240-01 fix holds) |

## B. Grain and uniqueness

| # | Invariant | Status | Findings |
|---|---|---|---|
| B1 | mega views: one row per project | ✅ | exchange-rate dedupe holds |
| B2 | exchange_rates unique (date, ccy) | ✅ | constraint in place |
| B3 | one signed original per project | ✅ | |
| B4 | investor views: unique refs | ✅ | |
| B5 | dup (estimate, equipment) pairs | ⚪ | 3 signed estimates (1136-01, 1583-01, 679-01) have split equipment lines. **Validated benign**: summed quantities match QB's independent production estimates (22.0/71.2/15.4 kW all consistent with 2,672/10,500/2,079 kWh/mo). One row (est 3925/row 7435) has NULL amount — hygiene only. |

## C. Soft-delete coherence

| # | Invariant | Status | Findings |
|---|---|---|---|
| C1–C3 | deleted project/client cascade | ✅ | |
| C4 | soft-deleted quotes still status=signed | 🟡 | 2 (quotes 1000014=2240-01, 24182=700-01). **Inert** — projects deleted too. Frontend twin: ImpactPage signed-quotes query lacks `deleted_at` filter (one-line fix queued). |

## D. Cross-field contradictions

| # | Invariant | Status | Findings |
|---|---|---|---|
| D1 | site dept = muni's dept | ✅ | 8 fixed 2026-07-09; clean |
| D2 | site country = dept's country | 🟡 | site 294 "Cafe Welchez" (1414-01, pipeline, no quotes): country HN correct, dept wrongly GT-Petén (should be Copán) — **ops confirm**. Site 271 "Another test site": orphaned test data — **delete candidate**. |
| D3 | muni→dept→country chain | ✅ | |
| D5 | smod_code = raster (excl 8 documented legacy) | ✅ | trigger keeping it true |
| D6 | project.client = site.client | ✅ | |
| D7 | install end ≥ install start | 🔍 | 776-04 (end 2024-08-19 < start 2024-09-02), 98-01 (end 2022-02-01 < start 2022-12-10). Both in portfolio; skews production_start → accumulated-savings-to-date. **Ops confirm true dates.** |
| D8 | project_date in range | ✅ | |

## E. Duplicated-attribute drift

| # | Invariant | Status | Findings |
|---|---|---|---|
| E1 | projects location = site location | ✅ | 7 divergences found (fallout of 07-09 dept realignment touching sites only) — **re-synced 2026-07-12**. Goes away permanently when legacy columns drop. |
| E2 | clients vs projects impact flags | ⚪ | clients layer still empty (0/191) — see decisions-log "canonical homes" entry; storage migration pending team alignment |
| E3 | contract snapshots vs live (date/retail/ccy) | ✅ | fully consistent after Texaco patch |

## F. Import fidelity vs QuickBase

| # | Invariant | Status | Findings |
|---|---|---|---|
| F1a | electricity price zeroed on import | ✅ | 51-01 was the only one; fixed 2026-07-03 |
| F1b | monthly production estimate dropped on import | 🟠 | **14 portfolio estimates**: 679-02, 264-06/07/08/09, 255-01, 595-01, 367-01, 556-01, 562-01, 625-01, 292-01, 615-01, 292-02. QB has values (241–8,900 kWh/mo); Supabase 0/NULL. Affects production-multiplier precision in savings calc. **Backfill candidate (bounded, values in register query).** |
| F1c | partner cost zeroed | ✅ | |
| F1d | retail vs QB | ⚪/🔍 | Systematic: QB stores retail **con IVA**, Supabase **sin IVA** (HN ratio exactly 1/1.15, SV 1/1.13, GT 1/1.12). **~17 GT estimates deviate from the IVA line** beyond 2% (worst: 204-01 0.71, 311-01 0.72, 202-01 0.83, 57-01 1.20; two worst "outliers" 776-02/264-07 were multi-revision artifacts, actually clean). Internal identities (G) all hold, so Supabase is self-consistent — likely QB staleness, but **finance should eyeball the 17**. |

## G. Financial identities — CORE CLEAN ✅

| # | Invariant | Status | Findings |
|---|---|---|---|
| G1 | contract value = Σ cash-flow components | ✅ | 0 mismatches portfolio-wide |
| G3 | duration = cash-flow rows (payment_number ≥ 1) | ✅ | first pass flagged 40 — all the month-0 down-payment row convention on app-created quotes; benign |
| G4 | Σ interest = finance income | ✅ | 0 mismatches |
| G2 | IVA = country rate | ✅ | audited 2026-07-06 (768 cells) |

## H. Report reconciliation — ALL CLEAN ✅

H3 dashboard = investor counts ✓ · H4 matview = engine per-project ✓ · H1 band = per-project (verified 07-07) ✓.
**Incident during review**: `app.mv_impact_project_summary_v1` was momentarily absent
(likely a concurrent partial dbt run hitting the documented DROP..CASCADE gotcha);
rebuilt within minutes. Reinforces the case for a data-health check on matview existence.

## I. Reference data

| # | Invariant | Status | Findings |
|---|---|---|---|
| I1 | countries tax/grid factors | ✅ | |
| I3 | panel watts 100–800 | 🟡 | equipments 660 (GSHK "680W") and 661 (AE "620W") have **NULL potential_watts** with the wattage in their specs text. Unsigned quotes only today; becomes NA-capacity the day one signs. **Fix: set 680/620.** |
| I4 | smod grid covers all coords | ✅ | |

## J. Statistical outliers — for human review 🔍

| # | Scan | Findings |
|---|---|---|
| J1 | $/kW outside 500–4,000 | 15 rows. High side (574-04 $8.6k, 218-01 $6.4k, 84-01, 460-01, 1331-01, 291-02) plausibly battery/offgrid-heavy. Low side (30-01 $82!, 50-04 $316, 525-02 $323, 1558-01 $387, 343-01 $477) = add-on contracts carrying full panel lists — same class as the savings-ratio outliers below. |
| J2 | savings ratio > 30× | 8 rows: 7-01 (82×), 776-01 (67×), 1959-01 (48×), 732-01 (47×), 30-01 (43×), 525-02 (38×), 50-04 (37×), 1558-01 (33×). Pattern: small contract value + full-system savings. Boss's 12.1× average is safe, but these tails deserve a look before anyone quotes per-project ratios. |
| J3 | interest rate outside 8–30% | 24 rows. Two smell like data errors: **343-01 (49.7%) and 541-02 (49.1%)** — likely monthly/annual confusion. The sub-8% cluster is early-era projects (plausible promo rates). |
| J4/J5/J6 | signing gaps / repeated coords | ✅ done earlier (Texaco; benign clusters) |

## K. History mining

| # | Scan | Status | Findings |
|---|---|---|---|
| K1 | signed → unsigned transitions | ✅ | 5, all known (2240-01 test, 2174-01 fell through, Texaco's bounce) |
| K2 | retail edited post-signing | ✅ | 4 hits = 2026-04-28 QB-import initial population (null→value), not edits |
| K3 | snapshot divergence | ✅ | see E3 |

## L. Existing dbt test suite (full run, 2026-07-12)

19 of 23 pass — including all four of Jake's modified working-tree tests except one.
The four failures, dissected:

| Test | Fail rows | Verdict |
|---|---|---|
| `test_no_orphaned_clients` | 378 | **Stale premise** — predates the full QB import, which deliberately brought prospect clients without projects (and it doesn't filter deleted). Redefine or drop. |
| `test_interest_payments_match` | 2,684 (44 projects) | **Two populations**: (a) 577-01/02 manual cash-flow replacements (max diff Q170,919 — restructure broke interest = rate × principal); (b) whole-schedule failures on recent app-generated projects (1934-*, 2040, 2132, 2153, 2267, 2444…) with moderate diffs — the quote engine's amortization doesn't match the test's assumption (interest_t = rate × remaining_{t-1}). Since G1/G4/principal+interest=payment/zero-at-end all PASS, schedules are internally coherent — **the test's model needs review, not (necessarily) the data**. Finance to arbitrate. |
| `test_total_income_matches` | 2 | **577-02 and 577-03 only** — the manually-replaced schedules again (payments short of retail+interest by Q17k / Q10.6k). Finance: confirm the restructure intended this. |
| `test_ar_principal_totals` | 5,027 | **3,997 are ±Q1 rounding noise** (test threshold too tight — add tolerance). 1,030 real diffs across 213 projects, max Q1.07M (1963-01) — the AR view vs recompute genuinely disagree on long-term principal splits for a wide set. **Feeds the loan tape → top-priority finance follow-up**, likely a definition drift between `int_official_accounts_receivable_*` and the test's expected calc. |

Recurring theme: **project family 577** (manual cash-flow replacement, 2026-01) shows up in
every financial identity failure. Recommend a dedicated reconciliation of 577-01/02/03.

## Fix batch (awaiting approval)

1. **F1b backfill**: 14 monthly production estimates from QB values.
2. **I3**: equipments 660/661 potential_watts ← 680/620 (from their own specs).
3. **B5 hygiene**: estimate_equipment row 7435 NULL amount → 0 (or merge into row 7434).
4. **D2**: delete test site 271; site 294 department → ops confirm Copán first.
5. **ImpactPage**: add `deleted_at is null` to the signed-quotes query (frontend one-liner).

## Human-review queue

- **AR view vs recompute: 1,030 real diffs / 213 projects (L) — top finance item (loan-tape input)**
- **577-01/02/03 reconciliation** — manual cash-flow replacement broke interest + income identities
- `test_interest_payments_match` model vs quote-engine amortization — finance arbitrate, then fix test or engine
- F1d: 17 GT estimates off the con-IVA line — finance eyeball
- D7: 776-04, 98-01 installation date inversions — ops
- J2/J1 low-$/kW: add-on projects carrying full panel lists — decide whether their kW/savings should attribute to the parent project (affects per-project ratios, not totals)
- J3: 343-01, 541-02 ~49% annual rates — finance verify
- 292-02, 124-03, 595-01/02 suspect coordinates (carried from 07-09)
- 29 manual impact-flag cells (ops, Impact page)

## Graduation candidates (make permanent)

dbt tests: A1–A4, B1/B3/B4, C1–C4, D1/D2/D5/D6/D7, E1 (until drop), G1/G3/G4, H3/H4, I3 + matview-exists check.
DB constraints: none new needed beyond existing (exchange-rates unique, smod trigger).
