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

## ~2026-04 · Cartera convention

`projects.cartera` distinguishes whether a deal was sourced by Albedo or by a partner. "Albedo" = deal sourced internally; "Socio" = sourced by the partner installer. Affects retail-price computation (Subtract margin → Albedo cartera; Add margin → Socio cartera). Memory: `project_cartera_field.md`.

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
