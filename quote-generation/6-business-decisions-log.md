# Quote Generation: Business Decisions Log

Chronological record of business + convention decisions that affect the
quote generation pipeline. Each entry: **what was decided, why, where it
lives in the code, and status**.

This file exists so decisions made in meetings, Slack threads, or pairing
sessions don't get lost in code comments and chat history. Add an entry
**before** or **at the same time as** the implementation lands.

Status legend:
- **Decided** — agreed and implemented.
- **In effect** — agreed and live in prod.
- **Open** — under discussion; placeholder for future decision.

---

# Earlier decisions (pre-2026-05)

Backfilled May 14 from memory, CLAUDE.md, CALCULATIONS.md, meeting docs, migration headers, and the existing inconsistencies catalogue. Dates are approximate; cross-references point at the canonical documentation.

## 2026-04-14 · Honduras poverty data seeded (UNDP IDH 2009, bottom 20%)

`municipalities.in_poorest_20_percent` for Honduras populated from UNDP Honduras IDH (Índice de Desarrollo Humano) at municipal level — bottom 20% by HDI = poorest 20%. 59 of 298 municipalities flagged. Source year 2009 (most recent publicly downloadable structured dataset; 2022 update exists but is only behind an interactive web magazine).

**Why this source.** UNDP 2022 update is inaccessible programmatically. 2009 HDI is the only public structured municipal-level dataset for Honduras. HDI is an inverse poverty proxy — low HDI = high poverty.

**Where.** Migration `2026-04-14-seed-hn-sv-poverty-data.sql`. Full provenance in `2026-04-14-seed-hn-sv-poverty-data-SOURCES.md`.

**Open follow-up.** Refresh when UNDP publishes a structured 2022 dataset.

## 2026-04-14 · El Salvador poverty data seeded (FISDL 2004, top 20% by rate)

`municipalities.in_poorest_20_percent` for El Salvador populated from FISDL/FLACSO Mapa de Pobreza 2004. Top 52 municipalities by poverty rate (≥64.7%) flagged. Coverage 260/263 (3 unmatched left NULL).

**Why this source.** FISDL is the canonical Salvadoran government poverty classification (used for Comunidades Solidarias targeting). Newer World Bank 2022 study exists as paper but not as structured download. El Salvador restructured to 44 municipalities in May 2024; our DB and the source data both use the old 262 structure — if projects start using the new structure, a remapping is needed.

**Where.** Same migration as above. SV provenance in the `_SOURCES.md`.

## 2026-04-08 · ADA endorsed GPS-based rural with two-layer fallback

ADA's Smiti formally endorsed Alex's proposal to switch rural classification from the administrative proxy (cabecera = urbano, resto = rural) to GPS-based using GHS-SMOD. ADA recommended a two-layer approach (GPS primary + administrative fallback) and asked for a portfolio assessment before full migration.

**Operational blocker** (also from this meeting): need to define WHEN GPS is captured for new projects (at quote? at install? require from socios?) and backfill the 269 existing signed projects. As of 2026-05-14, 36% of sites have `smod_code` populated.

**Why.** Current admin proxy misclassifies villages like Santa Cruz La Laguna (~3,000 inhabitants) as non-rural because they are municipal seats. GHS-SMOD is the UN standard, aligned with IRIS+, IFC, SDGs. Albedo can frame this as a USP per Smiti.

**Where.** Meeting summary: `docs/reunión-ada-8-abril-2026-resumen.md`. Full meeting doc: Google Doc `1XuqfcmLhwy7-5XjrnmPQ2B0eymwn2jmf_fMPiPzaAcU`. Implementation rule documented under the 2026-05-14 rural entry.

## 2026-04-08 · CO2 emission factors

Per-country emission factors used for impact reporting:

| Country | Factor (tCO2/MWh) |
|---|---|
| GT | 0.630 |
| SV | 0.410 |
| HN | 0.520 |

**Open follow-up.** Validate these with ADA + check vida útil / degradation assumptions.

## 2026-04 · Impact category cleanup (NULL → "Not Sure")

Bulk cleaned the 6 impact category manual fields on `projects`:
- NULLs converted to "Not Sure" so the impact reports treat them as known-unknown rather than missing.
- Fixed one project with the value `'Si '` (trailing space, Spanish) — normalized to `'Yes'`.
- Blocked editing of `rural_area` and `impoverished_area` in the operations form (now derived).
- Fixed a bug where the other 4 categories couldn't be edited.

## ~2026-04 · Impoverished area derived from municipality (GT only)

ADA's impoverished_area definition uses `municipalities.in_poorest_20_percent` for Guatemala. The 20% threshold was chosen by Alex in coordination with ADA. The five other impact categories were still on `projects.*` until 2026-05 (see entry below). Canonical doc: `dbt/main/CALCULATIONS.md` § 6.

## ~2026-04 · Down payment priority chain (quote-generator)

When resolving a per-quote down payment, the edge function uses this priority order:
1. `down_payment_amount_sin_iva` — explicit amount in the request.
2. `down_payment_percentage` — request payload × retail.
3. `estimates.down_payment_1_percentage` — legacy QB-import fallback.

Per-quote down payment is stored in `quotes.generation_parameters` (JSONB). The `estimates.down_payment_1_percentage` column is **legacy only** — do not use as source of truth for a specific quote. Canonical doc: `CLAUDE.md` § "Quote Generator — Down Payment Priority".

## ~2026-04 · Cartera convention — ⚠️ PAIRING SUPERSEDED 2026-08-21

`projects.cartera` distinguishes whether a deal was sourced by Albedo or by a partner. "Albedo" = deal sourced internally; "Socio" = sourced by the partner installer. Affects retail-price computation.

**The margin-type pairing originally recorded here (Subtract → Albedo, Add → Socio) was WRONG — it is the inverse.** See the 2026-08-21 entry "Cartera Albedo = margin type Add" for the corrected convention and the QuickBase evidence. Memory: `project_cartera_field.md`.

## ~2026-03 · IVA tax rate source

Tax rate comes from `countries.tax_rate` (GT 12%, SV 13%, HN 15%), **not** from `estimates.solar_tax_rate`. The estimate column is legacy. Memory: `feedback_iva_tax_rate.md`.

## ~2026-03 · QB provider key uses Record ID#2 (field 9)

The QuickBase providers table uses field 9 (Record ID#2) as the Supabase foreign key, not the default Record ID# (field 3). Affects QB-import joins. Memory: `project_qb_provider_key.md`.

## ~2026-03 · Append-only quote model

Each `quotes` row is a snapshot — never updated post-create. Signed variants get appended as a child row (`is_base=false`, `base_quote_id=<base>`). `quote_offers.quote_id` is UNIQUE per offer. This convention is what enables the client-offer flow to keep a stable token URL even if the variant matrix later changes.

## ~2026-03 · Client offer matrix format

Public offer page (`/offer/<token>`) shows: base term ±12 months × {0%, 10%} down payment matrix. Sent to clients via WhatsApp/email. Memory: `project_client_offer_page.md`.

## ~2026-03 · Quote tier solving (Gold/Silver/Bronze)

For each tier, goal-seek the loan term that achieves the target IRR when monthly payment is fixed at the client's monthly solar savings. Target IRRs (Gold 16%, Silver 14%, Bronze 12%) shift down per the impact-discount entry below. Memory: `project_quote_tier_solving.md`.

## ~2026-03 · Signed-only data for analytics

Treat `quotes.status = 'signed'` rows as the canonical/real data. Unsigned/pending rows are test/exploration data. DBT mart models filter to signed. Memory: `project_test_vs_real_data.md`.

## ~2026-03 · Multiple providers across phases is normal

Phases on a single project can use different providers. Treat warnings about "multiple providers" as informational, not errors. Memory: `project_multi_provider.md`.

## ~2026-02 · Operating countries

Albedo operates in **Guatemala, Honduras, and El Salvador only**. Not Colombia (despite historical references in some code). Memory: `project_operating_countries.md`.

## ~2026-02 · HN invoice due date rule

For Honduras invoices, the due date is the 10th of the cash-flow month. Used for DPD (days past due) calculations in HN loan-tape reporting. Memory: `project_hn_invoice_due_date_rule.md`.

## ~2026-02 · HN/SV accounting customer = DBA, not legal entity

HN and SV accounting customers are stored as DBA names (e.g. "Plaza Nexus"), whereas Supabase holds the legal entity. Cross-check gap lists from both sides when reconciling. Memory: `project_hn_dba_vs_legal_name.md`.

---

# Open business decisions (no resolution yet)

These are calc/data conventions that need a business decision but haven't gotten one. Each links to the technical analysis in `4-known-inconsistencies.md`.

## Open · Commission on mixed Add/Subtract phase projects

For projects mixing "Add" and "Subtract" margin-type providers, commission is added to the aggregate retail unconditionally — double-counting commission on the "Subtract" phases (which have it baked in already). Per `4-known-inconsistencies.md` § 1.3. Only impacts projects with both margin types in the same quote. Need a business decision on the right treatment.

## Open · Off-by-one in service payment spreading denominator

Insurance/maintenance total cost is divided by `term - gracePeriod + 1` but applied across `term - gracePeriod` months — a ~1.6% systematic under-collection. Code comment says "Match R code... to allow for discounts". Need confirmation: intentional client discount or legacy bug? Per `4-known-inconsistencies.md` § 1.5.

## Open · Insurance base: installation cost or financed retail?

Insurance cost is computed as `insuranceRate × installation_cost_sin_iva`. If the insurance is meant to cover physical asset replacement, the installation cost is correct. If it's meant to protect Albedo's receivable (the financed retail price, ~15-25% higher), there's structural under-insurance. Need clarification from the business on what the insurance covers. Per `4-known-inconsistencies.md` § 3.2.

## Open · Exchange rate timing risk

Currency conversion uses a single FX snapshot at quote time. Provider payouts span 3 periods, client payments span 5+ years — the snapshot doesn't reflect realized exchange movements. Is this an accepted business risk, or should we build periodic re-quoting / sensitivity analysis? Per `4-known-inconsistencies.md` § 3.3.

## Open · Provider payout schedule validation (sum ≠ 1.0)

`provider_payment_schedules.period_{1,2,3}_payout` aren't validated to sum to 1.0. A data-entry error would silently distort installation totals (and every downstream calc). No business decision needed — just add the constraint. Per `4-known-inconsistencies.md` § 3.5.

## Open · Maintenance inflation baseline (NII vs integration flow)

The NII table (manual mode) and the integration flow count maintenance inflation years from different baselines. Same parameters, different results. Per `4-known-inconsistencies.md` § 2.3. Either pick one or document why they differ.

## Open · Impact-indicator canonical home (clients vs projects vs sites)

The May 14 work moved schema and form capture to `clients.*` but the broader move is **provisional, not yet team-aligned**. Detailed list of sub-decisions in the May 14 entry above:
1. educational_institution / non_profit: clients or sites?
2. When do impact reports cut over from `projects.*` to `clients.*`?
3. Backfill strategy + handling of 3+ clients with conflicting per-project values.
4. Form authority — is rep-entered data the source of truth or does ops review?
5. When do we deprecate `projects.*` impact columns?
6. Reconciling existing inconsistencies before backfill.
7. Sales-rep communication about the new form fields.

**Stakeholders**: Sales lead, ops, Stephanie (impact reporting), Ian, ADA contacts.
**Until resolved**: new fields capture data but impact reports unchanged.

## Open · "Otros" category for non-gendered clients

Stephanie proposed adding an "Otros" client gender category for sociedades / organizations without a leadership gender. Per Apr 8 ADA meeting. Confirm with Stephanie where the manual classification comes from (Excel? QB?) before implementing.

## Open · CO2 retroactive change handling in ADA reports

When QB → Supabase migration changes CO2 cleanups, ADA's Q3/Q4 reports show different historical numbers. Need an alignment on whether to: (a) color-code/version flagged retroactive changes, or (b) freeze historical numbers and surface only forward changes. Per Apr 8 ADA meeting.

---

# 2026-05 decisions

## 2026-05-08 · Subtract retail gross-up

**Decision.** Retail = `phase_cost / (1 − margin)` for **both** "Subtract" and
"Add" margin types. Previously only "Add" was grossed up, under-pricing
Subtract deals by one margin's worth.

**Why.** Historical signed quotes (e.g. estimate `894-01-02`: phase_cost
51,834, retail 54,562, margin 0.05) confirm the convention. The pre-fix
code returned `phase_cost` directly for Subtract — wrong.

**Where.** `_shared/quote-helpers.ts::calculateRetailPriceComponents`.
Commit `3cb0bef`. Deployed to prod 2026-05-14.

**Status.** In effect.

---

## 2026-05-13 · Commission tier bonuses

**Decision.** Default commission bonus by IRR tier: **Silver +25%, Gold
+50%, Bronze 0%** (baseline). Stored in `quote_generator_configs` so they
can be tuned without code changes. Effective commission multiplier =
`1 + bonus`.

**Why.** Ian/Jake May 13 meeting. Previous frontend constant was 50% / 100% /
0% — those were placeholder values Jake had put in.

**Where.** Migration `add_commission_tier_bonuses_to_quote_generator_configs`
adds three columns. Frontend `DefaultQuotesPageV2.tsx` loads them on mount
and uses them in the Commission Breakdown panel.

**Status.** In effect (2026-05-14).

---

## 2026-05-13 · Maintenance grace period feature

**Decision.** Insurance/maintenance payments can start later than the loan
grace period ends. Controlled by a new param `maintenance_grace_period`
(additional months added on top of the loan grace).

**Why.** Sales wants the flexibility to offer "X months of free maintenance"
as an incentive without distorting the loan IRR. Per Ian (May 14), the
overall convention is:
  - **Insurance**: starts the month after the loan grace ends (matches the
    asset becoming productive). Currently `gracePeriod + 1`.
  - **Maintenance**: contract-dependent. Default to `gracePeriod + 1` but
    overridable per-estimate.

**Where.** `_shared/quote-integration-flow.ts::serviciosTable` accepts
`maintenanceGracePeriod` param. `quote-generator` passes from request body.

**Status.** In effect; param is on quote-generator only — quote-solver and
irr-calculator inherit it through the shared serviciosTable. Storage of
the per-estimate value is **open** (could go on the estimate row or be
derived from contract data).

---

## 2026-05-13 · Maintenance guarantee period adjustability

**Decision.** Allow the maintenance "guarantee period" (window during which
Albedo covers maintenance) to be adjusted per quote. Tracked as a separate
concept from the grace period.

**Where.** Not yet implemented. Tracked under Task #6.

**Status.** Open (implementation pending).

---

## 2026-05-13 · Impact discount formula

**Decision.** Each impact category a project qualifies for contributes one
"impact point". Total points reduce the target IRR uniformly across all
three tiers:

| Points | IRR reduction (percentage points) |
|---|---|
| 0 | 0.00 |
| 1 | 0.25 |
| 2 | 1.00 |
| 3 | 1.25 |
| 4 | 1.50 |
| 5 | 1.75 |
| 6 | 2.00 |

Example: a project with 3 impact points sees Bronze 12% → 10.75%, Silver
14% → 12.75%, Gold 16% → 14.75%.

**Why.** Ian May 14 confirmation, in line with the May 13 meeting decision
to "implement impact discount based on number of impact categories".

**Where.** `src/utils/impactDiscount.ts` exports `getImpactPoints` and
`getImpactDiscount`. Applied to cascade `tierTargets` and the post-cascade
locked `tierIrr` in `DefaultQuotesPageV2.tsx`. `irrTier` / `irrColor` /
`irrBgClass` / `irrLabel` accept an `impactDiscount` argument that shifts
the classification thresholds.

**Status.** In effect.

---

## 2026-05-14 · Commission added to retail (open)

**Open question.** SolarBase computes
`final_retail = retail_grossed_up + commission_total`. The Golden Copy
spreadsheet treats commission as an expense only — it doesn't inflate the
price the client pays.

**Default position** (per Jake): SolarBase is right; the client pays
commission as part of retail. Ian wants to confirm with Jake/Alex before
locking this in.

**Where.** `_shared/quote-helpers.ts::calculateProjectRetailWithCommission`
line ~482.

**Status.** Open. Pending Jake/Alex sign-off.

---

## 2026-05-14 · Service bases (insurance vs maintenance)

**Decision.** Insurance and maintenance rates apply to different bases:

  - **Insurance**: `retail × (1 + tax_rate)` (i.e. retail con-IVA).
  - **Maintenance**: `installation_cost` (sin-IVA).

**Why.** Matches the Golden Copy spreadsheet formulas (Services tab: B12
for insurance, B27 for maintenance). Verified via formula extraction
on 2026-05-14. Ian wants to confirm this convention is what we want
long-term — flagged as an item for the Friday Golden Copy review.

**Where.** `_shared/quote-integration-flow.ts::serviciosTable` accepts two
optional params: `insuranceBaseAmount` and `maintBaseAmount`.
`quote-generator` passes `retail × (1 + taxRate)` for insurance and
defaults maintenance to `installation_cost`.

**Status.** In effect; convention to be confirmed Friday May 15.

---

## 2026-05-14 · Insurance/maintenance premium markup default 10%

**Decision.** Albedo applies a 10% markup on top of the underlying
insurance and maintenance cost (both, same default).

**Why.** Golden Copy uses 10% on both (Services tab B8 and B21). SolarBase
previously defaulted to 1.8% for insurance — a stale value.

**Where.** `quote-generator/index.ts`: `insurance_premium` default
0.018 → 0.10. `maint_premium` default already 0.10. Eventually these
should move into `quote_generator_configs` (alongside the commission tier
bonuses) so they're adjustable per country/provider.

**Status.** In effect. Migration to config table is **open**.

---

## 2026-05-14 · Legal fee column is sin-IVA in the cashflow

**Decision.** The `legal_costs` column in the per-month cashflow table
reports **sin-IVA** legal fee. `calculateLegalFee` returns the con-IVA
bracket value; we divide by `(1 + tax_rate)` before placing it in the
cashflow column.

**Why.** Bug fix. Every other monthly column is sin-IVA — legal was the
odd one out, showing Q 1,175 (con-IVA) when GC expects Q 1,049 (sin-IVA).

**Where.** `quote-generator/index.ts` line ~199: `legalFee =
legalFeeConIva / (1 + taxRate)` before passing to serviciosTable.

**Status.** In effect.

---

## 2026-05-14 · Impact indicators move from projects to canonical homes — **PROVISIONAL, needs team alignment**

**Provisional decision** (Jake May 14, NOT yet stakeholder-confirmed). Long-term, the six impact indicators move from `projects.*` to their semantic owners:

| Indicator | Proposed home | How resolved |
|---|---|---|
| women_led | clients | manual field |
| youth_led | clients | manual field |
| educational_institution | clients (open: could be sites) | manual field |
| non_profit | clients (open: could be sites) | manual field |
| rural_area | derived from `sites.smod_code` | see rural decision below |
| impoverished_area | derived from `sites.municipality_id` → `municipalities.in_poorest_20_percent` (GT only) | already in effect |

**Why.** Each indicator is structurally an attribute of its owning entity. women_led/youth_led describe who the customer **is** (the client), not the specific project. Rural/impoverished describe **where** the install is (the site's location). The current state on `projects` produces real bugs — see Apr 8 ADA meeting prep: "3 clients have projects with **distinct** values of `women_led`, which is impossible if it's a property of leadership. The '9-client gap' in the women's count comes from this inconsistency."

### What's implemented today (live, May 14-15)

- Migration `add_impact_indicators_to_clients` added `women_led`, `youth_led`, `educational_institution`, `non_profit` columns to `clients` (nullable, no backfill).
- Quote wizard step 1 (`QuoteWizardStep1Client.tsx`) captures values into those columns.
- Impact-discount helper (`src/utils/impactDiscount.ts`) resolves via the canonical source first, falling back to `projects.*` when the new column is null.
- **Impact reports continue reading `projects.*` exclusively** — no behavior change in reporting yet.

### What's NOT yet decided (needs broader team / stakeholder coordination)

This is a real data-architecture change with cross-functional consequences. Each of these needs explicit alignment before the migration is "done":

1. **Educational institution / non-profit: clients or sites?**
   - Open between clients (default chosen for now) and sites (multi-site clients with mixed natures).
   - Need to confirm with sales: do we ever have a non-profit client whose specific site is not non-profit (or vice versa)?
   - **Stakeholder**: Sales lead, ops, Ian.

2. **When do impact reports cut over from `projects.*` to `clients.*`?**
   - Current state: form captures into `clients.*`, reports still read `projects.*`. Until cut-over happens, the new client columns have NO downstream effect on impact reporting.
   - Risks of cutting over too early: incomplete client-level data → undercount.
   - Risks of waiting too long: form data captured but ignored → diverging sources of truth.
   - **Stakeholder**: ADA reporting (Stephanie, Alex, Nausica), Ian.

3. **Backfill strategy.**
   - How do we populate `clients.women_led` etc. for existing 184 clients? Sources:
     - From `projects.*` of one of their projects? (Which one if values disagree?)
     - From a manual review by Stephanie / Sales?
     - Both, with manual override winning?
   - For clients with multiple projects that have conflicting values (the "9-client gap"), how do we reconcile?
   - **Stakeholder**: Sales (data ownership), Stephanie (impact reporting source of truth).

4. **Form authority / data ownership.**
   - The wizard form lets reps capture/edit these on the client at quote-creation time. Should reps be the source of truth, or should ops review/approve after capture?
   - What happens when a new project is signed for an existing client whose impact data was filled in earlier — do we revisit?
   - **Stakeholder**: Sales lead, ops.

5. **Legacy `projects.*` deprecation.**
   - Once reports cut over to client-level data, the `projects.*` columns become legacy fallback. When do we rename them to `_legacy` (per the Apr 8 follow-up) or drop them?
   - Memory note `project_post_ada_meeting_followups.md` already had this on the slate; it's blocked on the cut-over above.
   - **Stakeholder**: Ian, ADA.

6. **Inconsistency reconciliation.**
   - 3 known clients (as of Apr 8) have projects with conflicting `women_led` values. Before we backfill, those need a human decision. Same audit needed for the other 5 categories.
   - **Stakeholder**: Sales (knows the actual leadership), Stephanie (does manual classification).

7. **Communication to sales reps using the form.**
   - The wizard now asks reps to fill in impact indicators on the client. Reps need to understand: (a) this is the canonical source going forward, (b) "Not Sure" is acceptable but reduces impact discount, (c) values they enter may be reviewed.
   - **Stakeholder**: Sales training, sales lead.

### Where (code, today)

- Migration: `add_impact_indicators_to_clients` (applied to prod 2026-05-14).
- Frontend: `QuoteWizardStep1Client.tsx`, `src/schemas/clientSchema.ts`, `src/utils/impactDiscount.ts`, `src/pages/DefaultQuotesPageV2.tsx`.

**Status.** Schema + form + quote-flow lookup live in code as of 2026-05-15. The broader decision (canonical home, reporting cut-over, backfill) is **OPEN — pending team coordination**. Until that lands, the new fields capture data but don't change downstream reporting.

---

## 2026-05-14 · Rural definition: GHS-SMOD 11/12 with manual fallback

> **Amended 2026-07-09**: the manual fallback is retired and the backfill gap below is closed — rural is GPS-only, fully classified, and auto-maintained. See the [2026-07-09 entry](#2026-07-09--rural-is-gps-only--manual-fallback-removed-smod-backfilled-and-automated). The SMOD 11/12 rule itself is unchanged.

**Decision.** Rural classification follows the
[Global Human Settlement SMOD](https://human-settlement.emergency.copernicus.eu/ghs_smod2023.php)
classification, based on the site's GPS coordinates:

  - `sites.smod_code IN (11, 12)` → **rural**
    - 11 = Very Low Density Rural
    - 12 = Low Density Rural
  - SMOD codes 10 (uninhabited/water), 13 (rural cluster), and ≥ 21
    (peri-urban / urban) → **not rural**.
  - When `sites.smod_code IS NULL` (~64% of sites currently): fall back
    to the manual `projects.rural_area = 'Yes'`.

**Why.** Same convention already used by ATTA annual impact reports and
the public ImpactPage. See
`dbt/main/models/.../mart_atta_annual_impact_results.sql` lines 33-72.
Excludes SMOD 13 because rural clusters are denser (small-town pattern)
and historically Albedo's "rural" connotation refers to the lower
densities. Excludes SMOD 10 because uninhabited/water cells don't make
sense as solar installation sites.

The earlier 12% (manual) vs 63% (municipal `territorial_distinction`) vs
SMOD-derived debate was resolved in favour of SMOD because it's the most
objective, GIS-precision source. The municipal `territorial_distinction`
classification (cabecera departamental = urbano) was too coarse.

**Where.**
  - dbt: `mart_atta_annual_impact_results.sql` already uses this rule.
  - Frontend: `ImpactPage.tsx` already uses this rule.
  - Quote pipeline: `src/utils/impactDiscount.ts` updated to use the
    same rule for impact-discount calculation (with fallback to
    `projects.rural_area`).

**Backfill gap.** As of 2026-05-14, only 102 / 280 sites (36%) have a
populated `smod_code`. The remaining 178 sites fall back to the manual
field. Backfill is needed; tracked separately.

**Status.** In effect.

---

## 2026-05-14 · ESQUISOLAR partner margin (data correction)

**Decision.** ESQUISOLAR's margin is **3%** (per the `providers` table),
not 5% (used in some Golden Copy explorations).

**Why.** The 5% value was Ian using a margin override in the Golden Copy
spreadsheet to explore alternative pricing. Production data has 3% and
that's the value to use.

**Status.** Data correct; no code change.

---

## 2026-05-23 · Project useful life fixed at 30 years for impact reporting

**Decision.** All impact calculations in `mart_impact_projects_summary` use a
fixed **30-year project useful life**. Previously the horizon was derived from
each panel's `equipments.production_guarantee_years` (the listed warranty,
typically 25 years for the brands in our catalog).

**Why.** Three reasons (transcribed from the code header comment):
1. Different panel brands have different warranty periods (10-30 years). Using
   the warranty creates artificial variation in reported impact driven by
   equipment choice rather than physical reality.
2. Physical solar panel useful life is typically 25-30 years regardless of the
   warranty terms.
3. Inversores have separate warranties (~10 years) and previously caused data
   entry confusion when both warranties lived on the same `equipments` table.

30 years aligns with the GIIN IRIS+ and IFC OPIM conventions for solar impact
reporting.

**Where.** `dbt/main/models/models_on_static_tables/marts/impact/mart_impact_projects_summary.sql`
line ~40: `{% set PROJECT_USEFUL_LIFE_YEARS = 30 %}`. The rationale lives in
the comment block above that line.

**Effect on previously delivered reports.** Investor and impact reports built
under the prior (warranty-based) horizon report lower lifetime savings, CO₂
avoided, and electricity production than the current marts. The Dec 31 2025
investor sheet shows aggregate `total_solar_savings` ~10-13% lower than the
current marts for the same cohort, driven by this methodology change. This is
a deliberate change in reporting policy, not a calculation bug — past
deliveries reflect the prior policy and aren't directly comparable to current
output without restating.

**Status.** In effect.

---

## 2026-05-14 · Maintenance rate (data correction)

**Decision.** Maintenance rate for this provider/region pulls from the DB
lookup (`getMaintenanceRate`). Current value 0.15% for the ESQUISOLAR
case; Golden Copy was using 0.20% (likely rounding).

**Why.** The DB rate is the source of truth (it's per-provider/region).
Golden Copy was using a generic 0.20% for exploration.

**Status.** Data correct; no code change.

---

## 2026-07-13 — solve_tiers deleted; quote-solver renamed offer-generator

**Decision.** The `solve_tiers` mode (original Gold/Silver/Bronze flow: fix
payment at the client's monthly savings, goal-seek the TERM per target IRR
16/14/12) is deleted — 23 runs ever, all March 2026, zero callers in code
since the offer-flow redesign replaced it with `generate_offer`. With one
job left, the function is renamed **offer-generator**.

**Transition.** Old `quote-solver` deployment stays live until browser
sessions cycle out (todo: delete ~2026-07-20 after confirming zero traffic).
Frontend hits the new name as of staging deploy 2026-07-13.

**Status.** In effect. Deploy tag `deploy/offer-generator/20260713T182125Z-262d278`.

## 2026-07-13 — Manual mode stays, but runs THE engine

**Decision.** The quote-generator's manual mode (calculator inputs with no
estimate in the system) is kept — it may become useful — but it must produce
the same numbers as every other surface. It now maps its inputs onto the
shared forward-calc pipeline; the parallel engine from the original R port
(`generateNiiTable`/`getKpis` + private amortization forms) is deleted.

**Why.** The old manual engine had drifted: Subtract-margin costs weren't
grossed up, different amortization form, insurance started at grace+1.
Forensics showed it was never used in anger — 56 runs ever, all dev tests
(Jake/Ian/Danniel), last on 2026-02-21 — so no client number ever came from
it, but keeping a divergent engine invites the next drift bug.

**Behavior changes for future manual runs**: Subtract margins gross up like
everywhere else; insurance start month defaults to 4 (configurable) instead
of grace+1; amortization matches the pipeline. Explicit `annual_rate` still
bypasses the APR floor (old behavior preserved).

**Proof.** Equivalence test on 2645-01-01's real inputs: manual mode vs
estimate mode → all 10 KPIs and every numeric cell of all 61 cashflow months
identical. Deploy tag `deploy/quote-generator/20260713T173900Z-d02b743`.

**Status.** In effect. Closes duplication inventory v2 item 6.

## 2026-07-10 — Installed capacity: equipment sum is the ONLY source

**Decision.** Installed capacity (kW) = Σ (panel amount ×
`equipments.potential_watts`) from the estimate's equipment list. No other
source: `estimates.installed_potential_w` (QB import; nothing writes it) and
`systems.installed_capacity_kw` (one-off ops CSV, 2026-02-26) are legacy —
renamed `*_legacy`, unread by any live surface.

**Why.** The two static fields were unmaintained snapshots that had already
diverged: the operations table was missing capacity on 31 signed projects
(24 CSV gaps + 7 signed after the import, with no automation creating
system rows) and disagreed with equipment on ~36 more. Verified on all 284
signed estimates: zero cases of stored-without-equipment, zero
disagreements > 0.5 kW — equipment is a strict superset (272 vs 224
covered). "Actual installed ≠ sold" corrections, if ever needed, belong in
the auditable equipment list, not a free-typed kW field.

**Where.** SQL: `v_project_installed_kw` (new, project grain),
`v_estimate_calcs.installed_capacity_kwp` (equipment-only),
`v_investor_portfolio_map`. dbt: `int_estimate_installed_kw` (single home)
→ mega view / `mart_sales_by_partner` / `mart_debita_loan_tape`;
`mart_impact_projects_summary` already complied. Code: ops table service,
offer pages, APD maintenance panel, contract-generator
`equipmentInstalledKw` (which also fixed leasing contracts printing an
empty kW for new-engine quotes). Migrations
`2026-07-10-installed-capacity-equipment-canonical.sql` +
`...-rename-legacy-columns.sql`.

**Open follow-up.** Ops to review the 35-project discrepancy list (several
exactly 2× or ½× — smells like per-phase CSV rows or duplicated equipment
lines); any confirmed real corrections get fixed in the equipment list.

**Status.** In effect (deployed 2026-07-10).

---

## 2026-07-10 — Default WACC is 9% everywhere

**Decision.** When no WACC is supplied, every pricing surface uses **9%**
(0.09). One resolution chain, shared by quote-solver, quote-generator, and
irr-calculator: payload `albedo_wacc`/`wacc` → config row `default_wacc` →
`DEFAULT_WACC` (0.09).

**Why.** The 2026-07-10 duplication re-audit (inventory v2 item 1) found the
three engines disagreed: solver/generator fell back to 0.10, irr-calculator
to 0.09 — so the interactive dial could price the same estimate differently
than the persisted offer. The frontend already assumed 0.09, making 0.10 the
outlier. WACC feeds calcRecommendedApr's min-APR floor, so the default
matters whenever a caller omits the field.

**Where.** `_shared/quote-params.ts::resolveWacc` (new single home for
request-param resolution, same commit consolidates the down-payment
three-tier chain and the `maint_*` aliases across all three engines);
`DEFAULT_WACC` in `_shared/quote-constants.ts`; `quote_generator_configs`
rows updated 0.10 → 0.09 (migration
`2026-07-10-quote-configs-default-wacc-9pct.sql`); QuoteConfigForm fallback.
Deploy tags `deploy/*/20260710T2126*-2276f44`.

**Exposure note.** The app's offer flow always sends `wacc: 0.09`
explicitly, and no estimate row carries a stored `down_payment_1_percentage`
> 0, so the historic divergence had little-to-no live blast radius — the fix
is prophylactic alignment, not a data repair.

**Status.** In effect (deployed 2026-07-10). Post-deploy smoke test:
irr-calculator `forward_with_rate` and quote-generator at the same
estimate/term/rate return identical IRR (8.39) and payment (2,205.47).

---

## 2026-07-10 — Commission priced into retail = the commission type's TOTAL

**Decision.** The commission included in the client's retail price is
`commission_types.commission_percentage` (the row's total), applied to the
post-discount sin-IVA retail and added in full. It does **not** depend on
which vendor/affiliate rows are attached to the estimate: the sales
director's share applies even when no director is assigned, and affiliate
shares are governed by the chosen commission *type*, not by affiliate rows.
Role and affiliate rows determine **payout attribution** (who receives what,
including the tier multiplier on the earnings side) — never the price.

**Why.** The engine previously summed per-role percentages for roles present
on the estimate, which made the price depend on data-entry completeness —
identical deals priced at 3% or 4% depending on whether someone attached the
director row (surfaced by the 2599-01 Juan Sánchez discrepancy, issue #339).
A separate unversioned engine build also briefly applied the total ÷ 1.12;
that treatment is explicitly rejected — the percentage applies to sin-IVA
retail. Quotes generated before 2026-07-10 keep their as-shared prices (no
regeneration); the ~91 pending quotes priced under drifted definitions are
deliberately left as the client saw them.

**Where.** `supabase/functions/_shared/quote-helpers.ts` →
`deriveCommissionPercentages` (comment links back here). Deploy tags
`deploy/*/20260709T16*` mark when it went live.

**Status.** Decided (Jake) / In effect.

---

## 2026-07-09 · Rural is GPS-only — manual fallback removed; SMOD backfilled and automated

**Decision** (Jake, 2026-07-03: "we should always be using the GPS data"; implemented 2026-07-03 → 2026-07-07). Rural classification is **exclusively** GHS-SMOD-derived: `sites.smod_code IN (11, 12)` → rural, any other code → not rural, no coordinates → **Data Pending**. The manual-fallback clause of the [2026-05-14 rural definition](#2026-05-14--rural-definition-ghs-smod-1112-with-manual-fallback) is retired — `projects.rural_area` is no longer consulted anywhere. The SMOD 11/12 boundary itself is unchanged.

**Why.** The fallback existed only because 64% of sites lacked coordinates. That gap is closed: the 2026-06-26 field export supplied coordinates for the whole signed portfolio, GHS-SMOD **E2020 R2023A** codes were backfilled (epoch validated 92% against the original sample; E2025/E2030 scored worse), and classification is now **automatic** — `trg_sites_set_smod_code` + the `public.ghs_smod_grid` lookup table stamp `smod_code` whenever site coordinates are saved, so the gap cannot silently reopen. Keeping the fallback after that would have meant two sources disagreeing (manual said 12% rural; GPS says 38%) with the mix depending on data coverage — exactly the ambiguity funders flagged.

**Effect on published numbers.** Portfolio rural went from "14% (soft, 143 of 277 unclassified)" to **38% fully classified (104 Yes / 173 No, 0 pending)**; high-social-impact moved 53% → 69%. Same methodology as before, applied to 100% of the portfolio instead of 48% — flagged proactively in the June 30 investor redelivery.

**Where.**
  - dbt: `mart_impact_projects_summary.sql` (rural_area CASE; manual field "intentionally ignored"), consumed by all `mart_investors_*`, ADA, ATTA.
  - Frontend: `ImpactPage.tsx` (fallback removed 2026-07-07).
  - DB: `trg_sites_set_smod_code`, `compute_smod_code()`, `ghs_smod_grid` — docs at `albedo-automations-infra/database/triggers/sites-smod-code.md`; loader `database/scripts/load-ghs-smod-grid.py`.
  - Epoch upgrades are a deliberate, CHANGELOG'd decision (numbers move) — see the trigger doc.

**Remaining process gap.** Automation fires only when coordinates exist; WHEN GPS is captured for new projects (at quote? at install? required from socios?) is still the open item from the 2026-05-14 meeting. `projects.rural_area` is now a dead field pending the `_legacy` rename (see the provisional "canonical homes" entry above).

**Status.** In effect.

---

## 2026-07-10 — Total contract value: con & sin IVA flavors, maintenance always included

**Decision.** `total_contract_value` exists in two flavors — sin IVA and con
IVA — and BOTH always include maintenance payments when the schedule has
them, alongside monthly lease, down payment, asset transfer, insurance, and
legal costs.

**Why.** The 2026-07-10 duplication audit found four disagreeing definitions
(app SQL views: con-IVA without maintenance; dbt: sin-IVA with; the contract
generator carrying two internal variants). This ruling makes maintenance
inclusion non-negotiable and blesses both IVA flavors as first-class. The
app views were brought into compliance the same day (zero value change —
no signed project carries maintenance rows yet).

**Where.** `database/views/v_project_calcs.sql` (canonical),
`v_active_projects` (migration 2026-07-10-views-tcv-include-maintenance.sql),
dbt `int_projects_mega_view` (already conforming, sin-IVA flavor),
contract-generator snapshot (already conforming). See
`architecture/business-logic-map.md` item D.

**Status.** Decided (Jake) / In effect.

---

## 2026-07-20 — Addendum supersession uses SEAM semantics

**Decision.** When an addendum is signed, the old contract is frozen **at the
seam** (= the addendum's signing date). The old quote's cash-flow rows are
stamped `addended_at = seam`, so payments up to the seam remain **active
history** and payments after the seam are nullified; the addendum's schedule
governs from the seam on. The old quote and its estimate (when the addendum
lives on a different estimate) also get `addended_at = seam`. An addendum is
declared at **approval** time (`approve-quote-variant` with
`addendum_of_quote_id`, which stamps `replaces_quote_id` /
`original_quote_id` / `addendum_number`) and supersession executes at **sign**
time (`supersedeAddendumChain` in the contract-generator sign path). Full
restatement (old flows entirely nullified because the new schedule restates
history — the 541-02 early-payoff shape) is the exception and stays a manual
operation.

**Why.** The seven pre-migration chains were hand-crafted under two
conflicting conventions: seam (577-02/03 — by accident: rows never stamped,
old schedules happened to end at the seam) and full-nullify-via-backdating
(981-x, 679-01, 541-02). Seam is the loan-servicing-standard "modification"
shape: it preserves real payment history for the loan tape / Zoho sync /
Debita reporting, and it is what the `is_nullified` formula
(`addended_at IS NOT NULL AND payment_date > addended_at`) was designed to
express. It also matches what the calc engine naturally produces (schedules
run forward from the signing date).

**Where.** `supabase/functions/approve-quote-variant` +
`disapprove-quote-variant` (addendum mode; supabase PR #22),
`packages/shared/supabase-sdk` `supersedeAddendumChain` + contract-generator
sign hook + `QuoteOfferMatrix.addendumTarget` (infra PR #369). The invariant
to test: per project, the active (non-nullified) flows across the chain form
one timeline — no month covered by two quotes, no gap at the seam.

**Status.** Decided (Jake) / In effect.

---

## 2026-07-21 — 810-07 addendum: QB rows verbatim, grace 5→12 (pure deferral)

**Decision.** The 810-07 addendum restates the original QB contract's payment
breakdown **exactly** — 48 paying months of Q3,470.14 lease (first row
3,470.13, QB's own rounding) + Q109.89 insurance, IVA 12% → Q4,009.63/month
all-in, asset transfer Q1,067.83 in the final row — with the grace period
extended from 5 to 12 months (paying window moves Mar 2026–Feb 2030 →
Oct 2026–Sep 2030). The 7 extra grace months are a **pure deferral**: the
balance freezes at Q115,691.68 (the original post-grace balance) with zero
interest accrual — Albedo absorbs the year of financing, the client's total
obligation is unchanged. The legal fee is excluded (paid Sep 2025 under the
original contract).

**Why.** The calc engine cannot reproduce another engine's rows (its own rate
solve, insurance and rounding differ), so a re-generated offer at the same
Q4,009.63 total produced a different internal breakdown and, at "plazo 53",
only 41 paying months instead of 48. The client agreed to the *original
numbers, later* — so the offer is hand-crafted from the imported original
schedule (monthly_cash_flows of quote 33884), not engine-generated. The
capitalizing alternative (engine offer at 12-month grace, higher payments)
was considered and rejected in favor of matching the QB contract exactly.

**Where.** Quote 20 (`810-07-04-03`) + quote_offer 1261 (single manual cell,
`calc_engine_version = 'manual-addendum-restatement'`), created by
`albedo-automations-infra/database/migrations/2026-07-21-810-07-addendum-restated-offer.sql`.
Cash flows materialize from this variant at sign time; supersession of quote
33884 then follows the seam ruling above with the seam re-stamped to
2025-10-01 (keeps the paid legal fee active, nullifies the forgiven
Mar–Jul 2026 rows).

**Status.** Decided (Jake, 2026-07-21) / In effect — pending signature.

---

## 2026-07-22 (bis) — Bombeo Solar - Offgrid: cualquier bomba basta

**Decision.** Supersedes the same-day entry below. An estimate classifies as
**`Bombeo Solar - Offgrid`** whenever it contains one or more `Bomba de agua`
(equipment type 7), **unconditionally** — panels, baterías, medidores,
calentadores, anything else present does not matter. The pump branch remains
first in the CASE, ahead of the battery test.

**Why.** Gerson's ruling, July 22nd 2026: having a bomba is what makes it a
bombeo project — the rest of the equipment mix is irrelevant to the
classification. This replaces Jake's earlier same-day "pump must be sole
load" rule (below), under which a pump system with batteries stayed plain
Offgrid.

**Where.** `v_estimate_calcs.system_type` (single canonical definition).
Migration
`albedo-automations-infra/database/migrations/2026-07-22-v-estimate-calcs-bombeo-any-pump.sql`;
CHANGELOG entry same date. Still forward-looking: zero pump equipment in
catalog or estimates at ruling time.

**Status.** Decided (Gerson, 2026-07-22) / In effect.

---

## 2026-07-22 — Bombeo Solar - Offgrid: cuarto tipo de sistema, bomba como carga única — SUPERSEDED (same day, see entry above)

**Decision.** A fourth `system_type` exists alongside Ongrid/Híbrida/Offgrid:
**`Bombeo Solar - Offgrid`**, for solar water-pumping projects. An estimate
classifies as Bombeo when it contains `Bomba de agua` equipment (type 7) AND
has no batteries AND no calentador solar térmico (the pump is the sole load;
panels/inverters/reguladores are allowed as supporting equipment) AND
`number_of_meters_to_install = 0`. The branch is evaluated **before** the
battery test. A pump system **with** batteries stays plain Offgrid.

**Why.** New solar-pumping product line needs its own type for offers and
reporting. Under the 3-way rule a battery-less pump system fell through to
Ongrid (the "no batteries → Ongrid" branch fired first), which is wrong —
pumping systems are inherently isolated. Alternatives considered: (a) any
pump present → Bombeo regardless of batteries (rejected: a pump+battery
system is a hybrid load, not a pumping project); (b) relabel only within the
existing Offgrid branch (rejected: battery-less pump systems would stay
Ongrid). Jake ruled for "pump must be sole load."

**Where.** `v_estimate_calcs.system_type` — the single canonical definition
(no TS/dbt recomputation). Migration
`albedo-automations-infra/database/migrations/2026-07-22-v-estimate-calcs-bombeo-solar-offgrid.sql`;
CHANGELOG entry same date. Forward-looking: zero pump equipment existed in
catalog or estimates at decision time; verified no existing estimate
reclassified.

**Status.** Superseded same day by Gerson's any-pump rule (entry above); never
in effect on real data (no pump estimates existed).

---

## 2026-07-24 — Quote status semantics: QuickBase overwrite convention retired

**Decision** (Jake). "Current quote" is a **derived** property, never a status
convention: the current contract for a project is the latest non-deleted quote
with `status IN ('signed','signed_and_addended')`, ordered by `created_at`.
A quote's status describes its own lifecycle only — `signed_and_addended`
means "was signed, later addended," NOT "no longer counts" (that was the
QuickBase semantics, where an addendum overwrote the record and the old quote
stopped counting as signed). Any reader that filters `status = 'signed'` alone,
or relies on the write-side invariant "the newest addendum quote is literally
'signed'," is on the retired convention.

**Why.** dbt's whole staging layer (`stg_signed_quotes` and everything above
it) still encodes the QB convention; it agrees with the canonical app views
today only because the write side happens to maintain the old invariant. The
2026-07-24 drift audit confirmed data parity is currently exact (279 = 279
signed set, 0 kW/cash-flow mismatches) — which makes now the moment to migrate
dbt to the derived rule while the swap is provably a no-op. Deriving currency
from data rather than encoding it in sibling statuses is also the standard
modeling shape and what the next reader will expect.

**Where.** Canonical implementation: the `signed` CTE of
`database/views/v_operations_projects.sql` and the equivalents in
`v_active_projects` / `v_project_installed_kw`. dbt migration tracked as the
"repoint staging at canonical views" cleanup (this entry is its Phase 0).

**Status.** Decided (Jake, 2026-07-24) / dbt migration in progress.

---

## 2026-07-24 — Funder reports: origination columns read the ORIGINAL quote, everything else the CURRENT contract

**Decision** (Jake). In funder-facing marts, columns that describe
**origination** (monto prestado / original principal, contract signing date,
original term) are read from the original quote (`addendum_number = 0`) —
explicitly and labeled as such. Every other measure (outstanding, payments,
prices, capacity, TCV, status) tracks the **current contract** (see the status
entry above) with seam-semantics cash-flow history (2026-07-20 entry).

**Why.** dbt's `int_original_quotes` currently feeds original-quote values
into *all* marts indiscriminately — a QB relic, since under QB the original
was the only record. Post-addendum that silently freezes prices/equipment at
origination everywhere. Loan-tape convention (industry standard) is:
origination fields frozen, servicing fields current.

**Where.** dbt `int_original_quotes` (to be renamed/scoped to origination use
only during the cleanup's Phase 3). Consumers: `mart_debita_*`,
`mart_finance_*`, `mart_investors_*`.

**Status.** Decided (Jake, 2026-07-24) / implementation pending (Phase 3).

---

## 2026-07-27 — Insurance coverage is DERIVED: ends 30 days after the last financing payment

**Decision** (Jake, from Fernanda's Release 1 Proyectos request 2026-07-22).
Insurance vigencia in the projects layer is derived from financing state, not
maintained by hand: coverage ends **30 days after the last financing
payment** (`financing_end_date + 30`). Financed projects are `Vigente` until
that date and `Vencido` after it; Contado projects are `No aplica` (no
financed coverage). The manual `projects.insurance_status` column (QB-era
free text) is untouched and shown separately — it is not the source of truth
for vigencia.

**Why.** Ops needs to exclude finished policies from insurance management
without hand-updating a status field per project. The end date was already
being computed as `financing_end_date + 30 días` in the frontend table — the
rule is now in one place, in the view.

**Where.** `public.v_operations_projects` (`insurance_end_date`,
`insurance_coverage_status`), migration
`2026-07-27-v-operations-projects-insurance-coverage.sql`. Both frontends
read the columns; the local `computeInsuranceEndDate()` duplicates were
removed. Filterable as "Vigencia Seguro" on the ops projects page.

**Status.** Implemented and live 2026-07-27 (279 projects: 64 No aplica /
201 Vigente / 14 Vencido).

---

## 2026-07-27 — Cierre operativo for pre-2026 projects = milestone #14; 'Instalado' override retired

**Decision (Jake, with Fernanda's rule from the 2026-07-22 Release 1 meeting).**
Pre-2026 projects get a real fecha de cierre operativo instead of a display
override. The date is the fecha de cambio de medidor, or 2025-12-31 when there
is none — exactly as agreed in the meeting. The cierre operativo is not a new
concept: it is milestone catalog #14 (`fecha_cierre_operativo_intensivo`), so
closed projects rank as **'O&M Activo'** through the normal estado ladder. No
'Cerrado' label was introduced; if ops wants different wording we relabel later.

**Why.** Fernanda's ask was that pre-2026 projects stop appearing as pending
in management lists and become countable as closed in the upcoming summary
panel. Once they carry milestone #14, the 2026-07-08 'Instalado' override
(created because their milestone history was sparse) loses its reason to
exist — one condition, one word.

**Where.** Backfill + view change in migration
`2026-07-27-cierre-operativo-pre-2026-backfill.sql` (206 projects: 194 meter
dates, 12 fallback). Override removed from `v_project_milestone_stage` and the
three `computeEstado` TS mirrors (quotes-app, solar_base_frontend, supabase
`_shared/governance`); `governance-milestones` edge function redeployed.

**Data note.** 1080-02 has meter_change_date 2027-07-25 (future — typo); it
took the fallback. Correct the meter date, then update its milestone.

**Status.** Implemented and live 2026-07-27 (estado distribution: 206 O&M
Activo / 33 Medidor Bidireccional Instalado / 18 sin actividad / 22 others).

---

## 2026-07-28 — Layering: canonical views are the semantic layer; dbt is reporting, history and tests

**Decision (Jake).** Reaffirms and sharpens the 2026-06-30 architecture. There
is **one** definition of every business concept, it lives in a canonical
`public` view in the infra repo, and everything else reads it:

    tables → public canonical views → dbt marts → funder deliverables
                     ↘ app reads canonical views directly

- **Canonical `public` views = what is true.** Signed set, cash-flow validity,
  installed capacity, milestone estado, insurance vigencia, retail money. The
  app reads these directly; dbt reads them as the `canonical` source.
- **dbt = what a given audience is shown.** Four jobs, no more: (1) report
  shaping — Debita tapes, ADA/ATTA factsheets, investor splits are opinionated
  *documents*, not truths; (2) history — snapshots/SCD-2 answer "what did the
  portfolio look like on 31 March", which a live view cannot; (3) the test
  harness — schema tests plus source tests pointed at the canonical views, so
  the nightly audits the semantic layer; (4) `data_chat` flattenings.
- **The flow is one-way. dbt never publishes to the app.** Any app read of an
  `app.mv_*` model inverts ownership (the product depending on nightly-flavoured
  analytics output) and is to be retired.

**Rule of thumb for new logic.** App shows it, or two or more consumers need
it → canonical view. Exactly one report needs it → dbt int/mart. Needs
point-in-time history → dbt snapshot. Quote generation → edge functions
(unchanged, see the calculations rule).

**What this means for the dbt layers.** The `stg_`/`int_` layers split in two.
Models that re-derive shared business concepts (`stg_signed_quotes`,
`stg_signed_projects`, cash-flow validity, capacity, the mega views,
original-quote selection) dissolve into canonical views — dbt sources the view
instead. Models that are genuinely report math (currency conversion at report
FX, month-0 adjustments, AR ladders, remaining-principal schedules) stay: no
app page needs them and their shape is dictated by what the funder asked for.
The staging layer shrinks to a few `source()` pass-throughs kept only so
lineage stays legible in dbt docs.

**Why not move the app-facing views into dbt** (considered 2026-07-24 and
again 2026-07-28, rejected): (a) it splits every feature across two repos —
today a change like the derived seguros columns is one infra commit carrying
the migration and the frontend together; (b) dbt-authored views need
`security_invoker = true` and grants applied via post-hooks, so one forgotten
hook silently bypasses RLS, whereas the infra-repo convention keeps that in
the file being edited; (c) the hoped-for reuse of the `stg`/`int` layers does
not materialise in a *separate* dbt project anyway. The real prize — tests and
lineage over app views — is obtained instead by pointing dbt **source tests**
at the existing `canonical` source. Revisit only if authoring plain SQL
becomes the measured bottleneck; so far the canonical views have each been
written faster than their verification.

**Where.** Canonical views: `albedo-automations-infra/database/views/` (+
migration + CHANGELOG per change). dbt `canonical` source: `dbt/main/models/
sources.yml`. Prior entries: 2026-06-30 (architecture agreed), 2026-07-24
(status semantics + origination-vs-current rulings).

**Status.** Direction agreed 2026-07-28. Phases 0–2 of the dbt cleanup are
live (canonical repoint verified no-op on prod). Outstanding: two app reads of
`app.mv_*` still to retire (`mv_monthly_cash_flows_v1`,
`mv_impact_project_summary_v1`), Phase 3 (original-contract scoping), Phase 4
(housekeeping + source tests).

---

## 2026-07-29 — project_date cutover is 2025-11-01, not 2025-12-01

**Decision** (Jake, 2026-07-29). The signing-date cutover in
`int_original_quotes` is **November 1, 2025**. Quotes with
`contract_signing_date >= 2025-11-01` take `contract_signing_date` as their
`project_date` (falling back to `adjusted_project_date` when set); anything
earlier uses `first_payment_date`, because older signing dates are not
trustworthy. Originally approved by Jacob Stern.

**Why this needed a ruling.** The code has always said `'2025-11-01'` while the
comment above it said "December 1, 2025". One of the two was wrong and nobody
had established which. It was not academic — the disagreement covers every
quote signed in November 2025:

- 11 signed original quotes fall in the window
- **0** carry an `adjusted_project_date` (which would have made both readings agree)
- **8** resolve to a different `project_date` under the two readings

Five of those eight cross a fiscal-year boundary — signed 2025-11-19 with
`first_payment_date` 2026-01-01 — so the reading decides whether they are FY2025
or FY2026 contracts. `mart_investors_portfolio_overview` buckets by
`EXTRACT(YEAR FROM project_date)` into literal "2025"/"2026" columns, so this
moves funder-facing figures, and it feeds the ADA/ATTA quarterlies and the EEFFs
month spine as well.

Affected projects: `1665-01`, `1847-01`, `1934-02`, `1934-03`, `1934-05`,
`1959-01`, `2040-01`, `820-01`. The five that would have shifted year are
`1847-01`, `1934-02`, `1934-03`, `1934-05`, `2040-01`.

**Outcome.** The code was right; the comment was wrong. **No output changed** —
this ruling confirms current behaviour and corrects the documentation, so no
report needs restating and the five projects stay in FY2025.

**Where.** `dbt/main/models/models_on_static_tables/intermediate/int_original_quotes.sql`
(both `CASE` expressions, unchanged). The misleading comment was replaced
2026-07-29 and now links back to this entry.

**Status.** Decided and in effect. Closes the date half of the open Phase 3
question on `int_original_quotes`; scoping that model to origination-only
columns is still outstanding.

---

## 2026-04-13 — Production starts at installation, not at signing (logged 2026-07-29)

**Decision** (Jake, from the ADA 8-abril-2026 prep). Impact "to date" metrics
count production from when the system actually started generating, not from
the contract signing date. Resolved by a three-branch waterfall, in order:

1. `actual_installation_end_date` — the real commissioning date.
2. `GREATEST(actual_installation_start_date + 1 month, project_date)` — assume
   one month from start of installation to commissioning, but **never claim
   production started before signing**.
3. `project_date + 2 months` — last resort.

Both constants are empirical, from the April analysis: the observed average
gap between signing and installation was **~62 days (2 months)**, and one
month is the assumed start→commissioning lag.

**Why.** The previous behaviour assumed production began on the signing date,
which overstates accumulated production and accumulated savings for every
project that took weeks or months to install — directly inflating the ADA and
ATTA impact numbers. The `GREATEST` clamp in branch 2 exists because the
install dates are dirty: **15 of the 50 projects currently on that branch**
have an install date more than a month before signing, including a handful
decades off. Without the clamp those would claim production years before the
contract existed.

The `CASE` wrapper around branch 2 is load-bearing, not style: Postgres
`GREATEST()` ignores NULLs, so without it a NULL install start would return
`project_date` and swallow branch 3 rather than falling through to it.

**Why it was logged late.** The rule shipped in dbt commit `7b1ee9c`
(2026-04-13) documented only as a code comment plus talking point #9 in
`docs/reunión-ada-8-abril-2026.md`. It surfaced during the 2026-07-29
extraction and is recorded here because it moves funder-facing numbers.

**Current branch distribution** (281 signed original quotes, measured
2026-07-29): branch 1 **205** · branch 2 **50** (15 of them clamped) ·
branch 3 **26**. Note this has shifted a lot since the rule was designed —
the April doc had `actual_installation_end_date` on only 21/269 (8%), so
branch 2 was expected to carry ~90% of projects. Today branch 1 carries 73%,
because the Feb 2026 CSV data-entry pass filled in the completion dates.

**Where.** `dbt/main/models/models_on_static_tables/intermediate/int_production_start_dates.sql`
— the single definition since 2026-07-29 (`23788c6`). It was previously
copy-pasted character-for-character into both `int_project_monthly_solar_savings`
and `mart_impact_projects_summary`; both now `ref()` it. Install dates come
from `source('canonical', 'v_project_governed_dates')`, which reads
`project_milestones` first — see the 2026-07-29 entry on governed dates.

**Status.** In effect since 2026-04-13; consolidated to one model 2026-07-29.
Open question worth revisiting: branch 2's one-month assumption was tuned when
it covered 90% of projects and now covers 18%, so its weight in the reported
totals is much smaller than when it was chosen.

---

## 2026-08-20 — Annulment: voiding a SIGNED contract is its own state

**Decision.** Four rulings, taken together:

1. **A new `annulled` state**, on both `projects.workflow_status` and
   `quotes.status` — *not* the existing `cancelled`. `cancelled` and
   `rejected` mean the deal died **before** signature; annulment means a
   signed contract was voided **after** the fact. One value for both would
   make them indistinguishable in every current-state read.
2. **Seam semantics**, reusing the 2026-07-20 addendum ruling. Payments on or
   before `projects.annulled_on` stay live history; payments after it are
   void. An annulment is a supersession with no successor.
3. **The signed quote moves to `annulled`**, which is what actually removes
   the project from the post-signature surfaces.
4. **The forward book disappears from the loan tape; the cash received does
   not.** dbt drops the project's *schedule* entirely, which is correct —
   nothing is owed. What was actually collected survives in
   `mart_debita_historical_payment_tape` (driven by `lease_payments`, no
   signed filter) and in `mart_debita_snapshots` (incremental, append-only),
   so tapes already delivered to Debita still reconcile.

Annulment is **admin-only** and runs through one RPC, `annul_project()`.

**Why.** Almost every post-signature surface gates on quote status, and every
one of them filters *for* signed rather than against cancelled — so moving the
quote off `signed` drops the project from the 32 dbt models under
`stg_signed_quotes`, `v_operations_projects`, `v_financial_risk_portfolio`,
`financing_date_ranges` and `v_project_calcs` with no changes to any of them.
The strongest evidence this is the right seam is that two health checks
*self-resolve*: `v_addendum_chain_health` and dbt's
`test_quotes_accounted_for` both assert "signed quote ⇒ has flows", and
annulment keeps that invariant true instead of requiring a carve-out.

A separate `annulled` quote status (rather than reusing `cancelled`) also
closed a real hole: the APD status dropdown offered `signed → cancelled`
directly. RLS blocks that for sales and finance but **not for admins** — the
people who annul. Taken, it dropped the project out of dbt while leaving no
seam, no voided contract, no reason and no audit trail. With one shared value
that option could not be removed without also removing the legitimate one.

The alternative on ruling 4 — widening `stg_signed_quotes` to keep the
pre-seam schedule — was rejected: it edits the chokepoint 32 models depend on,
and would have forced carve-outs into both health checks above.

**Where.** `database/annulment.md` (full DB/code split),
`database/migrations/2026-08-20-*.sql` (four migrations; the enum one must
land first), `v_cash_flows_all_states.is_active`, and
`ActiveProjectDetails` / `AnnulProjectModal` on the frontend.
Supersedes nothing; extends the 2026-07-20 seam ruling to a second seam.

**Status.** Decided (Jake) / Implemented, verified locally, not yet deployed.

---

## 2026-08-21 — Cartera Albedo = margin type "Add" (labels were inverted)

**Status: In effect** (data migration ran 2026-08-23; UI label fix pending).

**Decision.** The canonical pairing is **cartera Albedo = `quote_margin_type
'Add'`**, cartera Socio = `'Subtract'`. Every internal surface that encoded the
inverse (wizard/ProviderForm dropdowns, `ProvidersTable.carteraLabel`, the
Spanish tooltip, the earlier ~2026-04 entry in this log, `v_estimate_calcs*`
views, `scripts/qb-continuity-import.ts`) is wrong and is being corrected.
`resolveSocioSolarName` (issue #325, `Add` → "Albedo Solar") was correct all
along.

**Evidence.** QuickBase is the source of truth:
- `qb_raw.proveedores` f8: Albedo Solar (id 5) = `Add`, margin 0.2 (queried 2026-08-21).
- QB `Estimates` f585: cartera = "Albedo" iff the user code's provider is "Albedo Solar".
- QB `Estimates` f451: quote shows "Albedo Solar" when cartera = Albedo — matches
  `resolveSocioSolarName` only under Add = Albedo.
- QB `Estimates` f681: vendor commission priced into retail only for `Add` — Albedo house
  reps (cartera Albedo) must carry commission, so Albedo Solar is `Add`.
- Phase-ratio forensics: provider-5 phases stamped before the 2026-07-17 flip have
  `phase_cost = partner_quote_amount` (Add semantics); phases stamped after have the
  ×0.8 Subtract signature.

**Impact.** Provider 5 (`Albedo Solar`) held `Subtract` from 2026-07-17 to the fix,
under-pricing every Albedo Solar quote written in that window by 20% (no gross-up) plus
the missing vendedor commission. 5 estimates affected, all unsigned; 3 phases restamped.

**Where.** Migration
`albedo-automations-infra/database/migrations/2026-08-21-fix-albedo-solar-cartera-margin-type-inversion.sql`
(run scoped `AND p.id = 5`; margin-0 providers 76/87/95/100 pending a name-match check —
one has signed exposure and needs its retail pinned first). Import-script fix in the same
commit. Related: known-inconsistency §1.3 (mixed-phase commission) is unchanged by this;
issue #222 (imported Subtract phase_cost) is a separate defect.

## 2026-08-25 · Project signing date = ORIGINAL contract date, even for addendums (all surfaces)

**Decision.** The project-level signing date — on EVERY surface (operations
page, proyectos firmados, dbt project-level marts, reports), not just ops
(scope widened by Jake 2026-08-26 from the original ops-only ruling) — is the
**original** client contract's `signed_on` — the addendum-0 quote's contract.
Addendums do not move it. All surfaces must resolve it through the
`v_project_signing` intermediate view (or, for frozen snapshot surfaces,
stamp from the truth at sign time) — never re-derive it locally.
Truth for signing dates is `contracts.signed_on` (client contract row);
`quotes.contract_signing_date` is legacy/denormalized and slated for
demotion to a synced mirror, then `_legacy` rename.

**Why.** Signing dates today are split across `quotes.contract_signing_date`
(QB-era imports only — the in-app sign flow never writes it, which is why
ops rows show blanks) and `contracts.signed_on` (what the sign endpoint
actually stamps). `v_operations_projects` reads the quote column via
newest-signed-quote (`ORDER BY created_at DESC`), so an addendum silently
swaps the date shown to the addendum's date — ops wants the anchor date of
the deal, not the latest paperwork event. Addendum-level dates still exist
per contract row for invoicing/seam logic (see the adenda procedure:
invoicing changes at addendum signature); this decision is about the
project-level display/reporting anchor. Contract snapshots stay faithful to
their OWN contract (an addendum contract's snapshot carries the addendum's
dates) — the project-level anchor is a display/reporting concern resolved via
`v_project_signing`, not by rewriting snapshots.

**Where.** Implemented 2026-08-25: resolver view
`database/views/v_project_signing.sql` (chain-root quote → client-contract
`signed_on`, frozen legacy quote column as fallback), `v_operations_projects`
repointed to it — migration
`2026-08-25-signing-date-source-of-truth-ops-view.sql`. One refinement vs the
original plan: the 8 QB-era addended originals with no contract row do NOT get
fabricated contract rows (readers expect real `data` snapshots) — the resolver
falls back to their frozen legacy date instead, same shape as
`v_project_governed_dates`. Legacy-column mirror (backfill + sync trigger):
`2026-08-25-sync-quote-contract-signing-date.sql` +
`database/triggers/quote-signing-date-sync.md`. Sign flow that stamps the
truth: `contract-generator/src/controllers.ts` (`updateContract`, final-sign
path).

**Status.** In effect (views live in prod 2026-08-25; audited 300 signed
quotes, 0 copy disagreements, 0 blanks after). Follow-up: repoint direct
readers of `quotes.contract_signing_date` (dbt `mart_debita_snapshots`,
`int_original_quotes`; generator fallback), then rename the column
`*_legacy`.

## 2026-08-27 · Month-0 row ratified as the schedule convention

**Decision (Jake, 2026-08-27).** The payment_number = 0 closing row — down
payment, legal costs, opening balance, dated at closing — is the standing
convention for native financed originals. The engine already writes it
(offer cashflow month 0 -> sign-time materialization); contado and addendum
schedules correctly have none. Ian owns the accounting treatment of the
closing row.

**Consequences.**
1. QB-continuity imports should synthesize the month-0 row AT IMPORT TIME
   (normalize at the source), shrinking the analytics normalizer's pool to
   the frozen legacy set.
2. The analytics normalizer (dbt int_monthly_cash_flows_month_0_adjustment,
   rewritten assumption-free 2026-08-27) covers the 25 legacy projects that
   will never be re-imported.

**Where.** Engine: quote-forward-calc.ts month-0 cell + sign-time
materialization (already live). Census + rewrite: dbt repo
AUDIT-2026-08.md, Phase 2.

## 2026-08-26 · project_date: one rule everywhere (cutover applies to all surfaces)

**Decision (Jake).** There is ONE `project_date` (Fecha de Proyecto) rule for
every surface — display pages, ops, dbt marts, investor/Debita reporting:

1. Manual `adjusted_project_date` override always wins.
2. Signing date on/after **2025-11-01**: use the signing date, contract-first
   (`contracts.signed_on` on the client contract, frozen legacy quote column
   as fallback).
3. Signing date before the cutover: use the project's **first payment date**
   (earliest live cash-flow row) — pre-cutover signing dates are not
   trustworthy.
4. Addendums never move it: everything anchors on the chain-root quote's
   contract.

This confirms the 2026-07-29 cutover ruling and widens it from the marts to
all surfaces. It supersedes the interim display-only rule (override → signing
date, no cutover) that `v_project_signing.project_date` carried earlier the
same day.

**Why.** Phase 0 of the post-QB audit measured the two coexisting rules
disagreeing on 188 of 292 projects (all pre-cutover) — one name, two values,
with the marts feeding external reporting on one and the app displaying the
other.

**Where.** Canonical home: `public.v_project_signing.project_date`
(migration `2026-08-26-v-project-signing-project-date-cutover.sql`, applied;
in-transaction verify 0/292 disagreements vs `int_original_quotes`). dbt keeps
its re-implementation and pins it with a canonical parity test (dbt repo,
`tests_on_static_tables/canonical_parity/`).

## 2026-08-25 · 1499-01 payoff restructure: manual addendum, two August rows, totals win

**Decision.** 1499-01 (Gasolinera / Distribuidora Mapher, 5+ invoices in
arrears) gets a negotiated payoff addendum ending Dec 2026: arrears stay on
the still-active old rows (the "Vencido" Q88,087.10 is a collection target
against existing invoices, not a new cash-flow row), an extra August
principal payment of 99,922.24 sin IVA (modeled as enganche:
`down_payment` column, `monthly_payment = 0`, so the old Aug-1 row and the
extra row coexist without tripping `double_billed_month`), then Sep–Dec at
exactly **Q152,500.00 con IVA** each. Where the negotiation sheet's
component splits disagree with its own totals (its IVA math mixes 12%-of-
gross with 12/112), the **negotiated totals win** and components are rebased
(principal 132,372.71 + flat interest 3,053.00 + seguro 735.00). Book
restates 694,493.02 → 629,413.08 at the seam: write-down 65,079.94 plus all
future scheduled interest — accepted as part of the deal. (Amended same
day: the extra payment is a NORMAL cuota, not an enganche —
monthly_payment = principal = **99,922.23**, invoicing Q111,912.90 so
August collects a flat Q200,000.00; the sheet's 99,922.24 principal cell
was mis-rounded against its own top-down total, and the book restates to
629,413.07. `double_billed_month` refined to permit deliberate seam-month
extras — see infra `database/addendum-supersession.md`.)

**Why.** The schedule is not engine-producible (flat fixed interest,
zero-interest enganche month, 4-month tail), so it follows the legacy
hand-crafted-chain pattern (541-02) — but with full FK linkage and the
standard `supersede_addendum_chain` RPC so the health view, APD tabs, and
contract dedupe all keep working. Keeping the arrears on the old rows avoids
voiding/re-issuing the Zoho invoices that already exist (1900, 2996).
Alternative considered and rejected: writing the Vencido row into the new
schedule with a backdated seam — cleaner sheet-match, but double-invoices
Mar/Apr and detaches the Feb partial payment from its row.

**Known residual.** Old active rows carry Q41,234.84 more scheduled-unpaid
than the negotiated arrears figure (sheet credits the Feb 15,000 against
Mar–Jul and skips one month vs. the DB's ledger; the Dec-2025 row has a
paid amount with NULL date). After the client pays Q88,087.10, that residual
still shows overdue until Finanzas reconciles or writes it off.

**Where.** Migration
`database/migrations/2026-08-25-1499-01-addendum-restructure.sql`
(self-verifying; seam/date constants edited at run time). CHANGELOG entry
same date. Source sheet: Drive `1WqM6qzd9ww1VJmRDIYK6NwC_LetsdFBi` cols E–J.

**Status.** Decided (migration drafted; runs after the adenda is physically
signed).

## How to add a new entry

1. Date the entry (`YYYY-MM-DD`).
2. One-line title summarising the decision.
3. **Decision** — the rule, in one short paragraph.
4. **Why** — context + the alternatives considered.
5. **Where** — file/line/migration name. Cross-reference both directions
   (link to this entry from the code comment).
6. **Status** — Decided / In effect / Open.
7. If the decision changes an existing one, link to the previous entry.

Prefer adding **before** the implementation lands — that way the PR
description can just link here.
