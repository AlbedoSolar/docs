# Quote Generation: Business Decisions Log

Chronological record of the business and convention decisions behind quote
generation, cash flows, dates, impact reporting and the data model. Each entry
says **what was decided, why, where it lives in the code, and its status**, so
decisions made in meetings, Slack threads or pairing sessions are not lost in
chat history.

**How to read it.** The **register** below lists every rule in effect, one line
each, grouped by topic, with a link to its entry. The **open questions** table
follows. The full entries come after, in date order. An entry marked ⚠️ is in
conflict with another entry; the conflicts are being resolved on
[the conflicts issue](https://github.com/AlbedoSolar/albedo-automations-infra/issues/596) and each entry says which number.

## What belongs here

An entry is a **business rule**: a statement of how Albedo prices, schedules,
classifies or reports something, that applies to every case meeting its
condition, from a date, decided by a named person. The test: *if a new project
showed up tomorrow, would this entry change what the system does with it?*

- **Yes → here.** "Commission is a cost, not part of retail." "Insurance is
  calculated on retail con IVA." "Two free years of maintenance by default."
  Conventions the system relies on count too (seam semantics for adendas, the
  signing-date source of truth, what "signed" means).
- **No, it describes one contract, project or row → not here.** A negotiated
  adenda for 1499-01, a payment holiday for one client, a margin corrected on
  one provider: those are **records**, kept in the private infra repo at
  `docs/contract-adjustments.md`. If the one-off produced a rule ("negotiated
  totals win over the sheet's component splits"), the rule comes here and the
  record stays there.
- **No, it is wrong data** (a missing date, a dangling link, a count that does
  not add up) → the Loose Ends issue (infra #587).
- **No, it is how to do something** (run an import, recover a view) → the
  owning repo's docs.

Status and dates matter more than prose: an entry that no longer holds is
marked *Superseded* with a pointer, never deleted.

**Status legend.** **Open**: under discussion. **Decided**: agreed, not yet live
in production. **In effect**: live in production. **Superseded**: replaced by a
later entry, kept for history.

**How to add an entry.** In the same session the decision is made: (1) add a
line to the register under its topic; (2) append an entry at the end of the
chronology with the template at the bottom of this file; (3) if it replaces an
earlier rule, mark that entry *Superseded* with a pointer and remove its
register line. Project references only, no client legal names or personal
data: this repository is public.

## Register: rules in effect

### Pricing and margins

| Date | Rule | Status | Entry |
|---|---|---|---|
| ~2026-03 | IVA comes from `countries.tax_rate` (GT 12 %, SV 13 %, HN 15 %), never from the estimate. | In effect | [entry](#e-iva-source) |
| ~2026-03 | Public offer page shows base term ±12 months × {0 %, 10 %} down payment. | In effect | [entry](#e-offer-matrix) |
| ~2026-04 | Down payment: explicit amount → request percentage → legacy estimate percentage; stored per quote. | In effect | [entry](#e-down-payment-chain) |
| 2026-05-08 | Retail = phase cost ÷ (1 − margin) for every phase, whatever the direction. | In effect | [entry](#e-retail-gross-up) |
| 2026-05-13 | Each impact category = 1 point; points lower the target IRR (0.25 pp at 1 point, 2.00 pp at 6); computed in the engine. | In effect | [entry](#e-impact-discount) |
| 2026-06-23 | Displayed monthly payment = median over paying months; the engine KPI and rep matrix use month 10. | In effect | [entry](#e-median-payment) |
| 2026-07-10 | Default WACC 9 % everywhere (payload → config → constant). | In effect | [entry](#e-wacc-9) |
| 2026-07-13 | The tier term-solver is gone; `offer-generator` has one job; tiers are an IRR label only. | In effect | [entry](#e-solve-tiers-deleted) |
| 2026-07-13 | Calculator manual mode runs the same engine as estimate mode. | In effect | [entry](#e-manual-mode) |
| 2026-08-21 | Cartera Albedo: the installer quotes cost and we add margin. Cartera Socio: the installer quotes retail and we take a discount. | In effect | [entry](#e-cartera-add) |
| 2026-09-04 | Stamp base facts and client commitments; derive aggregates live from one definition. | In effect | [entry](#e-stamping) |
| 2026-09-15 | The calculadora may show an estimated sales margin: display-only, labelled estimado. | In effect | [entry](#e-margen-estimado) |
| 2026-09-15 | Per-phase margin override lives in `project_phases.margin_override_percent` (Ian's build). | In effect | [entry](#e-margin-override) |
| 2026-09-15 | Commission is an Albedo cost, not part of the client's retail price, whatever the cartera. | In effect | [entry](#e-commission-is-cost) |
| 2026-09-19 | Electricity price inflation per country: GT 5 %, SV 5 %, HN 6 % (`countries` column). | In effect | [entry](#e-electricity-inflation) |
| 2026-09-24 | Cartera is on the estimate; provider terms are one versioned row; unsigned quotes float, signed ones freeze. | In effect | [entry](#e-cartera-on-estimate) |
| 2026-10-01 | Albedo cartera: 12 % default margin (config) with per-phase override; Socio: the provider's discount. | In effect | [entry](#e-albedo-margin-12) |
| 2026-10-09 | One terms row per provider (payout split + socio discount); no discount = 0 %; sales floor 5 % on the Albedo margin. | In effect | [entry](#e-provider-terms) |
| 2026-10-20 | One quoter configuration for everyone: the single active `quote_generator_configs` row holds the defaults; no per-user or per-version configs. | In effect | [entry](#e-one-config) |

### Services: insurance, maintenance, legal

| Date | Rule | Status | Entry |
|---|---|---|---|
| 2026-05-13 | Insurance starts after the grace period plus the estimate's offset (default 1 month); maintenance start is per estimate. | In effect | [entry](#e-maintenance-grace) |
| 2026-05-14 | Insurance rate applies to retail con IVA; maintenance rate to installation cost sin IVA. | In effect | [entry](#e-service-bases) |
| 2026-05-14 | Albedo marks up insurance and maintenance cost by 30 % by default (amended 2026-10-20; was 10 %). | In effect | [entry](#e-premium-10) |
| 2026-05-14 | The cash-flow legal fee column is sin IVA. | In effect | [entry](#e-legal-fee-sin-iva) |
| 2026-07-27 | Insurance coverage ends 30 days after the last financing payment; Contado = No aplica. | In effect | [entry](#e-insurance-coverage) |
| 2026-09-15 | The legal fee is an Albedo cost (in `predebt_cost_sin_iva` and `expense_services`) as well as client income. | In effect | [entry](#e-legal-fee-is-cost) |
| 2026-10-21 | Two free years of maintenance by default (charges start year 3); a rep may shorten it per deal on the estimate form. | In effect | [entry](#e-free-maintenance-years) |

### Cash flows, schedules and adendas

| Date | Rule | Status | Entry |
|---|---|---|---|
| ~2026-03 | Quotes are append-only snapshots; a signed variant is a child row. | In effect | [entry](#e-append-only-quotes) |
| 2026-07-10 | Total contract value has con- and sin-IVA flavors and always includes maintenance. | In effect | [entry](#e-tcv-flavors) |
| 2026-07-20 | An adenda freezes the old contract at the seam: rows up to the seam stay history, later rows are nullified. | In effect | [entry](#e-seam) |
| 2026-07-24 | Current contract = latest non-deleted quote with status `signed` or `signed_and_addended`; status describes lifecycle only. | In effect | [entry](#e-status-semantics) |
| 2026-08-20 | Annulling a signed contract is its own state (`annulled`), with seam semantics, admin-only RPC. | In effect | [entry](#e-annulment) |
| 2026-08-27 | Native financed originals carry a payment-0 closing row (down payment, legal costs, opening balance); new imports write it. | In effect | [entry](#e-month-0-row) |
| 2026-09-23 | The interest column is interest accrued that month, including grace months. | In effect | [entry](#e-interest-accrued) |
| 2026-09-25 | A payment-0 row never carries interest; the enganche is all principal. | In effect | [entry](#e-no-interest-month-0) |
| 2026-09-30 | `v_project_cash_flows` is the one cash-flows view (chain + month 0 derived at read time where missing); the raw table stays frozen; #505 shelved. | In effect | [entry](#e-canonical-cash-flows) |

### Dates and milestones

| Date | Rule | Status | Entry |
|---|---|---|---|
| ~2026-02 | Honduras invoices are due on the 10th of the cash-flow month (for DPD). | In effect | [entry](#e-hn-due-date) |
| 2026-04-13 | Production start = installation end, else install start + 1 month (never before signing), else signing + 2 months. | In effect | [entry](#e-production-start) |
| 2026-07-27 | Pre-2026 projects are closed via milestone #14 at the meter-change date, else 2025-12-31. | In effect | [entry](#e-cierre-operativo) |
| 2026-07-29 | Project date: the signing date from 2025-11-01 on, the first payment date before. | In effect | [entry](#e-project-date-cutover) |
| 2026-08-25 | Project signing date = the original client contract's `signed_on`, never an adenda's. | In effect | [entry](#e-signing-date-original) |
| 2026-08-26 | One `project_date` rule on every surface: override → signing date (from 2025-11-01) → first payment date. | In effect | [entry](#e-project-date-one-rule) |
| 2026-10-02 | Signing date = the client contract on the root quote only; the quote column is a synced mirror. | In effect | [entry](#e-signing-date-contract-only) |
| 2026-10-12 | Maintenance calendar = Cambio de Medidor date + 12 months per year (projects seeded since 2026-10-12). | In effect | [entry](#e-maintenance-anchor) |

### Impact and reporting

| Date | Rule | Status | Entry |
|---|---|---|---|
| ~2026-04 | Impact flags: NULL means 'Not Sure'; rural and impoverished are derived, never edited by hand. | In effect | [entry](#e-impact-not-sure) |
| ~2026-04 | Impoverished area = the site's municipality is in the poorest 20 % (`municipalities.in_poorest_20_percent`), in all three countries. | In effect | [entry](#e-impoverished) |
| 2026-04-08 | Grid emission factors: GT 0.630, SV 0.410, HN 0.520 tCO2/MWh (`countries.grid_factor`). | In effect | [entry](#e-co2-factors) |
| 2026-04-14 | Honduras poverty flag: UNDP IDH 2009, bottom 20 % by HDI (59 of 298 municipalities). | In effect | [entry](#e-hn-poverty) |
| 2026-04-14 | El Salvador poverty flag: FISDL 2004, the 52 poorest municipalities, on the old 262-municipality structure. | In effect | [entry](#e-sv-poverty) |
| 2026-05-23 | Impact reporting uses a fixed 30-year useful life. | In effect | [entry](#e-useful-life-30) |
| 2026-07-09 | Rural = site SMOD code 11 or 12, GPS only; no coordinates = Data Pending. | In effect | [entry](#e-rural-gps-only) |
| 2026-10-28 | Impact indicators have one home (`v_project_impact_flags`): women-led, youth-led, educational and non-profit are the client's answers; rural from site GPS; poverty from the site municipality. Pricing, the app and reports all read it; a project without coordinates gets no rural point. | In effect | [entry](#e-impact-flags-one-home) |
| 2026-07-24 | Funder marts: origination columns from the original quote, everything else from the current contract. | Decided | [entry](#e-origination-vs-current) |

### Data conventions and integrations

| Date | Rule | Status | Entry |
|---|---|---|---|
| ~2026-02 | Albedo operates in Guatemala, Honduras and El Salvador only. | In effect | [entry](#e-countries) |
| ~2026-02 | HN/SV accounting names are DBAs; Solar Base holds the legal entity; reconcile both ways. | In effect | [entry](#e-hn-sv-dba) |
| ~2026-03 | QuickBase provider join key = Record ID#2 (field 9). The QB API is revoked; `qb_raw` is the archive. | In effect | [entry](#e-qb-provider-key) |
| ~2026-03 | Several providers across phases is normal; warnings are informational. | In effect | [entry](#e-multi-provider) |
| 2026-07-10 | Installed kW = Σ panels × watts from the estimate's equipment; no other source. | In effect | [entry](#e-capacity-equipment) |
| 2026-07-22 | Any water pump in the equipment makes the system type Bombeo Solar - Offgrid. | In effect | [entry](#e-bombeo-any-pump) |
| 2026-07-28 | Canonical public views define every business concept; dbt reports, keeps history and tests; the flow is one-way. | In effect | [entry](#e-layering) |
| 2026-08-11 | QB 'Pagaré' deals (promissory notes) are not imported. | In effect | [entry](#e-pagare) |
| 2026-09-04 | No GT tax fallback (loud NULL); representative estimate = signed > approved > newest; the watts formula has one home. | In effect | [entry](#e-loud-null) |
| 2026-09-25 | Each country has one NA department and municipality for "no real location", sorted last. | In effect | [entry](#e-na-placeholders) |
| 2026-10-16 | Multi-unit residential clients are sector 3 (commercial), not 10. | In effect | [entry](#e-sector-3) |
| 2026-10-21 | Quote status transitions have one home: service_role code (`approve-quote-variant`, `disapprove-quote-variant`, `set-quote-status`, the contract-generator, `annul_project()`), enforced by column grants on `quotes.status`. | Decided | [entry](#e-quote-status-one-home) |
| 2026-10-21 | Database code computes nothing; it may refuse a write that breaks an invariant (RLS, grants, raising triggers) and may cascade only in an audit-only or ruled-and-documented trigger. | In effect | [entry](#e-db-compute-refuse-cascade) |

## Open questions

| Since | Question | Status | Entry |
|---|---|---|---|
| 2026-05-13 | Adjustable maintenance guarantee period per quote. | Closed 2026-10-21 (exists: the estimate dropdown; rule logged 2026-10-21) | [entry](#e-maintenance-guarantee) |
| 2026-05-14 | Is commission part of the retail the client pays? | Closed 2026-10-20 (no, since 2026-09-15) | [entry](#e-commission-in-retail-open) |
| ≤2026-05 | Commission on projects mixing Add and Subtract phases. | Closed 2026-10-19 (moot) | [entry](#e-open-mixed-commission) |
| ≤2026-05 | Service payment spreading divides by `term − grace + 1` but applies over `term − grace` (~1.6 % under-collection): bug or intended? | Open | [entry](#e-open-off-by-one) |
| ≤2026-05 | Insurance base: installation cost or financed retail? | Closed 2026-10-20 (retail con IVA) | [entry](#e-open-insurance-base) |
| ≤2026-05 | FX is a single snapshot at quote time while payouts and payments span years: accepted risk? | Open | [entry](#e-open-fx-timing) |
| ≤2026-05 | Validate that provider payouts sum to 1.0. | Closed 2026-10-20 (constraint exists) | [entry](#e-open-payout-sum) |
| ≤2026-05 | Maintenance inflation baseline differed between the NII table and the integration flow. | Closed 2026-10-20 (NII engine deleted) | [entry](#e-open-maint-inflation) |
| ≤2026-05 | Canonical home of the impact indicators (clients vs projects vs sites) and the reporting cut-over. | Open | [entry](#e-open-impact-home) |
| ≤2026-05 | An "Otros" client gender category for organisations. | Open | [entry](#e-open-otros) |
| ≤2026-05 | How ADA reports handle retroactive CO2 changes (version vs freeze). | Open | [entry](#e-open-co2-retro) |

The twelve ⚠️ conflicts (two entries that disagree, or an open item later decided) are listed with a recommendation on [the conflicts issue](https://github.com/AlbedoSolar/albedo-automations-infra/issues/596); rulings get applied here.

---

## Entries, in date order

Dates marked ~ are approximate (backfilled 2026-05-14 from memory, CLAUDE.md, CALCULATIONS.md, meeting docs and migration headers).

<a id="e-countries"></a>
### ~2026-02 · Operating countries

Albedo operates in **Guatemala, Honduras, and El Salvador only**. Not Colombia (despite historical references in some code). Memory: `project_operating_countries.md`.

**Status.** In effect.

<a id="e-hn-due-date"></a>
### ~2026-02 · HN invoice due date rule

For Honduras invoices, the due date is the 10th of the cash-flow month. Used for DPD (days past due) calculations in HN loan-tape reporting. Memory: `project_hn_invoice_due_date_rule.md`.

**Status.** In effect.

<a id="e-hn-sv-dba"></a>
### ~2026-02 · HN/SV accounting customer = DBA, not legal entity

HN and SV accounting customers are stored as DBA names (e.g. "Plaza Nexus"), whereas Supabase holds the legal entity. Cross-check gap lists from both sides when reconciling. Memory: `project_hn_dba_vs_legal_name.md`.

**Status.** In effect.

<a id="e-iva-source"></a>
### ~2026-03 · IVA tax rate source

Tax rate comes from `countries.tax_rate` (GT 12%, SV 13%, HN 15%), **not** from `estimates.solar_tax_rate`. The estimate column is legacy. Memory: `feedback_iva_tax_rate.md`.

**Status.** In effect.

<a id="e-qb-provider-key"></a>
### ~2026-03 · QB provider key uses Record ID#2 (field 9)

The QuickBase providers table uses field 9 (Record ID#2) as the Supabase foreign key, not the default Record ID# (field 3). Affects QB-import joins. Memory: `project_qb_provider_key.md`.

**Status.** In effect (archive only).

<a id="e-append-only-quotes"></a>
### ~2026-03 · Append-only quote model

Each `quotes` row is a snapshot — never updated post-create. Signed variants get appended as a child row (`is_base=false`, `base_quote_id=<base>`). `quote_offers.quote_id` is UNIQUE per offer. This convention is what enables the client-offer flow to keep a stable token URL even if the variant matrix later changes.

**Status.** In effect.

<a id="e-offer-matrix"></a>
### ~2026-03 · Client offer matrix format

Public offer page (`/offer/<token>`) shows: base term ±12 months × {0%, 10%} down payment matrix. Sent to clients via WhatsApp/email. Memory: `project_client_offer_page.md`.

**Status.** In effect.

<a id="e-tier-solving"></a>
### ~2026-03 · Quote tier solving (Gold/Silver/Bronze)


For each tier, goal-seek the loan term that achieves the target IRR when monthly payment is fixed at the client's monthly solar savings. Target IRRs (Gold 16%, Silver 14%, Bronze 12%) shift down per the impact-discount entry below. Memory: `project_quote_tier_solving.md`.

**Status.** Superseded by 2026-07-13 (`solve_tiers` deleted). The tier names survive only as the colour label on the IRR in the calculadora; the tier bonus was retired 2026-09-15; the impact discount lives in the engine.

<a id="e-signed-only"></a>
### ~2026-03 · Signed-only data for analytics

> **Superseded by 2026-07-24 (quote status semantics).**

Treat `quotes.status = 'signed'` rows as the canonical/real data. Unsigned/pending rows are test/exploration data. DBT mart models filter to signed. Memory: `project_test_vs_real_data.md`.

**Status.** Superseded by 2026-07-24: the current contract is the latest quote with status `signed` OR `signed_and_addended`.

<a id="e-multi-provider"></a>
### ~2026-03 · Multiple providers across phases is normal

Phases on a single project can use different providers. Treat warnings about "multiple providers" as informational, not errors. Memory: `project_multi_provider.md`.

**Status.** In effect.

<a id="e-impact-not-sure"></a>
### 2026-04 · Impact category cleanup (NULL → "Not Sure")

Bulk cleaned the 6 impact category manual fields on `projects`:
- NULLs converted to "Not Sure" so the impact reports treat them as known-unknown rather than missing.
- Fixed one project with the value `'Si '` (trailing space, Spanish) — normalized to `'Yes'`.
- Blocked editing of `rural_area` and `impoverished_area` in the operations form (now derived).
- Fixed a bug where the other 4 categories couldn't be edited.

**Status.** In effect.

<a id="e-impoverished"></a>
### ~2026-04 · Impoverished area derived from municipality (GT only)

> **Amended 2026-10-20 (Jake): applies to all three countries.** Honduras and El Salvador flags were seeded on 2026-04-14 and the dbt rule has no country filter; "GT only" was true only before that seed.

ADA's impoverished_area definition uses `municipalities.in_poorest_20_percent` for Guatemala. The 20% threshold was chosen by Alex in coordination with ADA. The five other impact categories were still on `projects.*` until 2026-05 (see entry below). Canonical doc: `dbt/main/CALCULATIONS.md` § 6.

**Status.** In effect.

<a id="e-down-payment-chain"></a>
### ~2026-04 · Down payment priority chain (quote-generator)

When resolving a per-quote down payment, the edge function uses this priority order:
1. `down_payment_amount_sin_iva` — explicit amount in the request.
2. `down_payment_percentage` — request payload × retail.
3. `estimates.down_payment_1_percentage` — legacy QB-import fallback.

Per-quote down payment is stored in `quotes.generation_parameters` (JSONB). The `estimates.down_payment_1_percentage` column is **legacy only** — do not use as source of truth for a specific quote. Canonical doc: `CLAUDE.md` § "Quote Generator — Down Payment Priority".

**Status.** In effect.

<a id="e-cartera-2026-04"></a>
### ~2026-04 · Cartera convention — ⚠️ PAIRING SUPERSEDED 2026-08-21

> **Superseded. The pairing written here was inverted; see 2026-08-21, then 2026-09-24 and 2026-10-09 for where cartera lives now.**

`projects.cartera` distinguishes whether a deal was sourced by Albedo or by a partner. "Albedo" = deal sourced internally; "Socio" = sourced by the partner installer. Affects retail-price computation.

**The margin-type pairing originally recorded here (Subtract → Albedo, Add → Socio) was WRONG — it is the inverse.** See the 2026-08-21 entry "Cartera Albedo = margin type Add" for the corrected convention and the QuickBase evidence. Memory: `project_cartera_field.md`.

**Status.** Superseded by 2026-08-21 (the pairing) and 2026-09-24 (cartera lives on the estimate).

<a id="e-ada-gps-rural"></a>
### 2026-04-08 · ADA endorsed GPS-based rural with two-layer fallback

ADA's Smiti formally endorsed Alex's proposal to switch rural classification from the administrative proxy (cabecera = urbano, resto = rural) to GPS-based using GHS-SMOD. ADA recommended a two-layer approach (GPS primary + administrative fallback) and asked for a portfolio assessment before full migration.

**Operational blocker** (also from this meeting): need to define WHEN GPS is captured for new projects (at quote? at install? require from socios?) and backfill the 269 existing signed projects. As of 2026-05-14, 36% of sites have `smod_code` populated.

**Why.** Current admin proxy misclassifies villages like Santa Cruz La Laguna (~3,000 inhabitants) as non-rural because they are municipal seats. GHS-SMOD is the UN standard, aligned with IRIS+, IFC, SDGs. Albedo can frame this as a USP per Smiti.

**Where.** Meeting summary: `docs/reunión-ada-8-abril-2026-resumen.md`. Full meeting doc: Google Doc `1XuqfcmLhwy7-5XjrnmPQ2B0eymwn2jmf_fMPiPzaAcU`. Implementation rule documented under the 2026-05-14 rural entry.

**Status.** Decided (history). The live rule is the 2026-07-09 entry; the two-layer fallback ADA suggested was removed there.

<a id="e-co2-factors"></a>
### 2026-04-08 · CO2 emission factors

Per-country emission factors used for impact reporting:

| Country | Factor (tCO2/MWh) |
|---|---|
| GT | 0.630 |
| SV | 0.410 |
| HN | 0.520 |

**Open follow-up.** Validate these with ADA + check vida útil / degradation assumptions.

**Status.** In effect (values live in `countries.grid_factor`). Open: validate with ADA; the Valuing Impact factors may replace them.

<a id="e-production-start"></a>
### 2026-04-13 — Production starts at installation, not at signing (logged 2026-07-29)

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

Open question worth revisiting: branch 2's one-month assumption was tuned when
it covered 90% of projects and now covers 18%, so its weight in the reported
totals is much smaller than when it was chosen.

**Status.** In effect.

<a id="e-hn-poverty"></a>
### 2026-04-14 · Honduras poverty data seeded (UNDP IDH 2009, bottom 20%)


`municipalities.in_poorest_20_percent` for Honduras populated from UNDP Honduras IDH (Índice de Desarrollo Humano) at municipal level — bottom 20% by HDI = poorest 20%. 59 of 298 municipalities flagged. Source year 2009 (most recent publicly downloadable structured dataset; 2022 update exists but is only behind an interactive web magazine).

**Why this source.** UNDP 2022 update is inaccessible programmatically. 2009 HDI is the only public structured municipal-level dataset for Honduras. HDI is an inverse poverty proxy — low HDI = high poverty.

**Where.** Migration `2026-04-14-seed-hn-sv-poverty-data.sql`. Full provenance in `2026-04-14-seed-hn-sv-poverty-data-SOURCES.md`.

**Open follow-up.** Refresh when UNDP publishes a structured 2022 dataset.

**Status.** In effect. Refresh when UNDP publishes a structured 2022 dataset.

<a id="e-sv-poverty"></a>
### 2026-04-14 · El Salvador poverty data seeded (FISDL 2004, top 20% by rate)


`municipalities.in_poorest_20_percent` for El Salvador populated from FISDL/FLACSO Mapa de Pobreza 2004. Top 52 municipalities by poverty rate (≥64.7%) flagged. Coverage 260/263 (3 unmatched left NULL).

**Why this source.** FISDL is the canonical Salvadoran government poverty classification (used for Comunidades Solidarias targeting). Newer World Bank 2022 study exists as paper but not as structured download. El Salvador restructured to 44 municipalities in May 2024; our DB and the source data both use the old 262 structure — if projects start using the new structure, a remapping is needed.

**Where.** Same migration as above. SV provenance in the `_SOURCES.md`.

**Status.** In effect. Remap if projects start using the 2024 44-municipality structure.

<a id="e-retail-gross-up"></a>
### 2026-05-08 · Subtract retail gross-up

**Decision.** Retail = `phase_cost / (1 − margin)` for **both** "Subtract" and
"Add" margin types. Previously only "Add" was grossed up, under-pricing
Subtract deals by one margin's worth.

**Why.** Historical signed quotes (e.g. estimate `894-01-02`: phase_cost
51,834, retail 54,562, margin 0.05) confirm the convention. The pre-fix
code returned `phase_cost` directly for Subtract — wrong.

**Where.** `_shared/quote-helpers.ts::calculateRetailPriceComponents`.
Commit `3cb0bef`. Deployed to prod 2026-05-14.


**Status.** In effect.

<a id="e-tier-bonuses"></a>
### 2026-05-13 · Commission tier bonuses

> **Superseded 2026-09-15 (Ian, issue #454).** The tier bonus is retired: the config columns are 0 and the calculadora no longer multiplies by them.

**Decision.** Default commission bonus by IRR tier: **Silver +25%, Gold
+50%, Bronze 0%** (baseline). Stored in `quote_generator_configs` so they
can be tuned without code changes. Effective commission multiplier =
`1 + bonus`.

**Why.** Ian/Jake May 13 meeting. Previous frontend constant was 50% / 100% /
0% — those were placeholder values Jake had put in.

**Where.** Migration `add_commission_tier_bonuses_to_quote_generator_configs`
adds three columns. Frontend `DefaultQuotesPageV2.tsx` loads them on mount
and uses them in the Commission Breakdown panel.


**Status.** Superseded by 2026-09-15.

<a id="e-maintenance-grace"></a>
### 2026-05-13 · Maintenance grace period feature

> **Amended 2026-10-20 (Ian):** insurance starts after the grace period plus the estimate's "Meses después del período de gracia para iniciar seguro" (`insurance_start_month_after_grace_period`, default 1). The engine's constant month 4 applies only to a caller that sends nothing.

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

irr-calculator inherit it through the shared serviciosTable. Storage of
the per-estimate value is **open** (could go on the estimate row or be
derived from contract data).

**Status.** In effect; the per-estimate offset is `estimates.insurance_start_month_after_grace_period`.

<a id="e-maintenance-guarantee"></a>
### 2026-05-13 · Maintenance guarantee period adjustability

> **Closed 2026-10-21 (Jake): done.** The per-quote adjustment exists as the estimate form's "Mantenimiento empieza a cobrarse en" dropdown; the standing rule is the 2026-10-21 entry.

**Decision.** Allow the maintenance "guarantee period" (window during which
Albedo covers maintenance) to be adjusted per quote. Tracked as a separate
concept from the grace period.

**Where.** Not yet implemented. Tracked under Task #6.


**Status.** Closed 2026-10-21: implemented as the estimate dropdown; see 2026-10-21.

<a id="e-impact-discount"></a>
### 2026-05-13 · Impact discount formula


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


**Status.** In effect. Since 2026-07 the discount is computed in the engine (`_shared/quote-impact.ts`: impact score → IRR adjustment and APR-floor discount) and returned as `impact_irr_adjustment`; the calculadora only displays it.

<a id="e-commission-in-retail-open"></a>
### 2026-05-14 · Commission added to retail (open)


> **Closed 2026-10-20: superseded by 2026-09-15.** Commission is a cost, not part of the price the client pays.

**Open question.** SolarBase computes
`final_retail = retail_grossed_up + commission_total`. The Golden Copy
spreadsheet treats commission as an expense only — it doesn't inflate the
price the client pays.

**Default position** (per Jake): SolarBase is right; the client pays
commission as part of retail. Ian wants to confirm with Jake/Alex before
locking this in.

**Where.** `_shared/quote-helpers.ts::calculateProjectRetailWithCommission`
line ~482.


**Status.** Superseded by 2026-09-15.

<a id="e-service-bases"></a>
### 2026-05-14 · Service bases (insurance vs maintenance)


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


**Status.** In effect. Confirmed 2026-10-20 (Ian, agreed with operations / Fernanda): insurance is calculated on retail con IVA; note that `first_insurance_cost_sin_iva = retail_con_iva × insurance_rate`.

<a id="e-premium-10"></a>
### 2026-05-14 · Insurance/maintenance premium markup default 10%

> **Amended 2026-10-20 (Jake, confirmed with Ian): the markup is 30 %.** `DEFAULT_INSURANCE_PREMIUM` and `DEFAULT_MAINTENANCE_PREMIUM` are 0.30 in `_shared/quote-constants.ts` (since at least 2026-05-28); every native quote stores `maintenance_premium = 0.3` and the client pays 1.30 × the service cost. The 10 % below was the Golden Copy figure and is history.

**Decision.** Albedo applies a 10% markup on top of the underlying
insurance and maintenance cost (both, same default).

**Why.** Golden Copy uses 10% on both (Services tab B8 and B21). SolarBase
previously defaulted to 1.8% for insurance — a stale value.

**Where.** `quote-generator/index.ts`: `insurance_premium` default
0.018 → 0.10. `maint_premium` default already 0.10. Eventually these
should move into `quote_generator_configs` (alongside the commission tier
bonuses) so they're adjustable per country/provider.


**Status.** In effect at 30 % (amended 2026-10-20). The defaults live in `quote-constants.ts`; moving them into the config table is still open.

<a id="e-legal-fee-sin-iva"></a>
### 2026-05-14 · Legal fee column is sin-IVA in the cashflow

**Decision.** The `legal_costs` column in the per-month cashflow table
reports **sin-IVA** legal fee. `calculateLegalFee` returns the con-IVA
bracket value; we divide by `(1 + tax_rate)` before placing it in the
cashflow column.

**Why.** Bug fix. Every other monthly column is sin-IVA — legal was the
odd one out, showing Q 1,175 (con-IVA) when GC expects Q 1,049 (sin-IVA).

**Where.** `quote-generator/index.ts` line ~199: `legalFee =
legalFeeConIva / (1 + taxRate)` before passing to serviciosTable.


**Status.** In effect.

<a id="e-impact-indicators-home"></a>
### 2026-05-14 · Impact indicators move from projects to canonical homes — **PROVISIONAL, needs team alignment**

**Provisional decision** (Jake May 14, NOT yet stakeholder-confirmed). Long-term, the six impact indicators move from `projects.*` to their semantic owners:

| Indicator | Proposed home | How resolved |
|---|---|---|
| women_led | clients | manual field |
| youth_led | clients | manual field |
| educational_institution | clients (open: could be sites) | manual field |
| non_profit | clients (open: could be sites) | manual field |
| rural_area | derived from `sites.smod_code` | see rural decision below |
| impoverished_area | derived from `sites.municipality_id` → `municipalities.in_poorest_20_percent` (all three countries since the 2026-04-14 seeds) | already in effect |

**Why.** Each indicator is structurally an attribute of its owning entity. women_led/youth_led describe who the customer **is** (the client), not the specific project. Rural/impoverished describe **where** the install is (the site's location). The current state on `projects` produces real bugs — see Apr 8 ADA meeting prep: "3 clients have projects with **distinct** values of `women_led`, which is impossible if it's a property of leadership. The '9-client gap' in the women's count comes from this inconsistency."

#### What's implemented today (live, May 14-15)

- Migration `add_impact_indicators_to_clients` added `women_led`, `youth_led`, `educational_institution`, `non_profit` columns to `clients` (nullable, no backfill).
- Quote wizard step 1 (`QuoteWizardStep1Client.tsx`) captures values into those columns.
- Impact-discount helper (`src/utils/impactDiscount.ts`) resolves via the canonical source first, falling back to `projects.*` when the new column is null.
- **Impact reports continue reading `projects.*` exclusively** — no behavior change in reporting yet.

#### What's NOT yet decided (needs broader team / stakeholder coordination)

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

#### Where (code, today)

- Migration: `add_impact_indicators_to_clients` (applied to prod 2026-05-14).
- Frontend: `QuoteWizardStep1Client.tsx`, `src/schemas/clientSchema.ts`, `src/utils/impactDiscount.ts`, `src/pages/DefaultQuotesPageV2.tsx`.


**Status.** Superseded 2026-10-28 by [Impact indicators have one home](#e-impact-flags-one-home): educational and non-profit stay on the client, the cut-over and backfill are done, deprecation of the project columns is infra #616.

<a id="e-rural-smod-fallback"></a>
### 2026-05-14 · Rural definition: GHS-SMOD 11/12 with manual fallback

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


**Status.** Superseded by 2026-07-09 (manual fallback removed). The SMOD 11/12 boundary is unchanged.

<a id="e-useful-life-30"></a>
### 2026-05-23 · Project useful life fixed at 30 years for impact reporting

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

<a id="e-median-payment"></a>
### 2026-06-23 · Displayed monthly payment = median of the paying months; the engine KPI stays month 10

**Decision.** Every surface that shows one monthly payment for a quote (offer
sheet headline and matrix, the per-year plan, the project page's DD panel)
shows the **median** of `total_payment_con_iva` over the variant's cash-flow
rows with `income_total > 0`. Fallback chain, mirrored everywhere: manual
override on the approved cell → median → the engine's `Monthly.Payment` KPI
(skeleton QB imports only). The engine KPI and `goalSeekAprByMonthlyPayment`
still use **month 10**, and the rep matrix shows that number.

**Why.** Insurance and maintenance change the cuota over the term, so one
month is a poor headline; the median is what the client will most often pay.
Switching the engine KPI too would mean rewriting the goal-seek from its
analytic shortcut to a true bisection; we fixed what clients and reviewers
see and left the rep-facing math alone until that rewrite is worth it.

**Where.** `ClientOfferPage.getMonthlyPayment` / `buildYearlyPlan`,
`ActiveProjectDetails` (aligned 2026-06-23); engine
`quote-integration-flow.ts` (`monthVal(10, …)`), `QuoteOfferMatrix.extractMonthlyPayment`.

**Open.** If the engine KPI ever changes, the goal-seek and the rep matrix
change with it in the same PR.


**Status.** In effect.

<a id="e-rural-gps-only"></a>
### 2026-07-09 · Rural is GPS-only — manual fallback removed; SMOD backfilled and automated

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

<a id="e-capacity-equipment"></a>
### 2026-07-10 — Installed capacity: equipment sum is the ONLY source

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


**Status.** In effect. Ops review of the capacity discrepancies is on Loose Ends.

<a id="e-wacc-9"></a>
### 2026-07-10 — Default WACC is 9% everywhere

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

irr-calculator `forward_with_rate` and quote-generator at the same
estimate/term/rate return identical IRR (8.39) and payment (2,205.47).

**Status.** In effect.

<a id="e-commission-total"></a>
### 2026-07-10 — Commission priced into retail = the commission type's TOTAL

> **Superseded by 2026-09-15 (Ian, issue #454; confirmed 2026-10-20).** Commission is an Albedo cost and is not part of the client's retail price, whatever the cartera.

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
director row (surfaced by the 2599-01 discrepancy, issue #339).
A separate unversioned engine build also briefly applied the total ÷ 1.12;
that treatment is explicitly rejected — the percentage applies to sin-IVA
retail. Quotes generated before 2026-07-10 keep their as-shared prices (no
regeneration); the ~91 pending quotes priced under drifted definitions are
deliberately left as the client saw them.

**Where.** `supabase/functions/_shared/quote-helpers.ts` →
`deriveCommissionPercentages` (comment links back here). Deploy tags
`deploy/*/20260709T16*` mark when it went live.


**Status.** Superseded by 2026-09-15.

<a id="e-tcv-flavors"></a>
### 2026-07-10 — Total contract value: con & sin IVA flavors, maintenance always included

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


**Status.** In effect.

<a id="e-solve-tiers-deleted"></a>
### 2026-07-13 — solve_tiers deleted; quote-solver renamed offer-generator


**Decision.** The `solve_tiers` mode (original Gold/Silver/Bronze flow: fix
payment at the client's monthly savings, goal-seek the TERM per target IRR
16/14/12) is deleted — 23 runs ever, all March 2026, zero callers in code
since the offer-flow redesign replaced it with `generate_offer`. With one
job left, the function is renamed **offer-generator**.

**Transition.** Old `quote-solver` deployment stays live until browser
sessions cycle out (todo: delete ~2026-07-20 after confirming zero traffic).
Frontend hits the new name as of staging deploy 2026-07-13.


**Status.** In effect.

<a id="e-manual-mode"></a>
### 2026-07-13 — Manual mode stays, but runs THE engine

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


**Status.** In effect.

<a id="e-seam"></a>
### 2026-07-20 — Addendum supersession uses SEAM semantics

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


**Status.** In effect.

<a id="e-bombeo-sole-load"></a>
### 2026-07-22 — Bombeo Solar - Offgrid: cuarto tipo de sistema, bomba como carga única — SUPERSEDED (same day, see entry above)

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

in effect on real data (no pump estimates existed).

**Status.** Superseded the same day by Gerson's any-pump rule (next entry).

<a id="e-bombeo-any-pump"></a>
### 2026-07-22 (bis) — Bombeo Solar - Offgrid: cualquier bomba basta

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


**Status.** In effect.

<a id="e-status-semantics"></a>
### 2026-07-24 — Quote status semantics: QuickBase overwrite convention retired

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


**Status.** In effect (dbt repointed; Phase 3 scoping of `int_original_quotes` still open).

<a id="e-origination-vs-current"></a>
### 2026-07-24 — Funder reports: origination columns read the ORIGINAL quote, everything else the CURRENT contract

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


**Status.** Decided; dbt Phase 3 (scoping `int_original_quotes`) still pending.

<a id="e-insurance-coverage"></a>
### 2026-07-27 — Insurance coverage is DERIVED: ends 30 days after the last financing payment

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

201 Vigente / 14 Vencido).

**Status.** In effect.

<a id="e-cierre-operativo"></a>
### 2026-07-27 — Cierre operativo for pre-2026 projects = milestone #14; 'Instalado' override retired


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

**Data note.** 1080-02 took the fallback because its meter-change date was a
future-dated typo; the fix is tracked on Loose Ends (infra #587), not here.

Activo / 33 Medidor Bidireccional Instalado / 18 sin actividad / 22 others).

**Status.** In effect.

<a id="e-layering"></a>
### 2026-07-28 — Layering: canonical views are the semantic layer; dbt is reporting, history and tests

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

live (canonical repoint verified no-op on prod). Outstanding: two app reads of
`app.mv_*` still to retire (`mv_monthly_cash_flows_v1`,
`mv_impact_project_summary_v1`), Phase 3 (original-contract scoping), Phase 4
(housekeeping + source tests).

**Status.** In effect. Since 2026-10-13 view migrations live in the `supabase` repo; view sources stay in infra `database/views/` until the docs move.

<a id="e-project-date-cutover"></a>
### 2026-07-29 — project_date cutover is 2025-11-01, not 2025-12-01

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

question on `int_original_quotes`; scoping that model to origination-only
columns is still outstanding.

**Status.** In effect.

<a id="e-pagare"></a>
### 2026-08-11 · "Pagaré" deals stay out of Solar Base

**Decision.** (Jake.) QuickBase quotes whose status is **Pagaré** (a
promissory note, not a lease: 768-01 and 888-01, both November 2025) are
deliberately not imported. Sweeps of `qb_raw` for signed-but-missing deals
filter on `Firmado` / `Firmado y adendado` only, and the continuity import
cohort omits them.

**Why.** A pagaré is a different instrument with no installment schedule to
materialize; importing it as a lease would put wrong cash flows in every
report.

**Where.** `scripts/qb-continuity-import.ts` (infra) and the signed-deals
gap queries from 2026-08-11.


**Status.** In effect.

<a id="e-annulment"></a>
### 2026-08-20 — Annulment: voiding a SIGNED contract is its own state

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


**Status.** In effect (live in prod).

<a id="e-cartera-add"></a>
### 2026-08-21 — Cartera Albedo = margin type "Add" (labels were inverted)


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

**Status.** In effect for the mapping. Storage superseded: cartera lives on the estimate (2026-09-24) and the provider column was dropped (2026-10-09), so the pending label fix and provider flips are moot.

<a id="e-signing-date-original"></a>
### 2026-08-25 · Project signing date = ORIGINAL contract date, even for addendums (all surfaces)

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

quotes, 0 copy disagreements, 0 blanks after). Follow-up: repoint direct
readers of `quotes.contract_signing_date` (dbt `mart_debita_snapshots`,
`int_original_quotes`; generator fallback), then rename the column
`*_legacy`.

**Status.** In effect. The legacy quote-column fallback was removed by 2026-10-02.

<a id="e-project-date-one-rule"></a>
### 2026-08-26 · project_date: one rule everywhere (cutover applies to all surfaces)

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

**Status.** In effect.

<a id="e-month-0-row"></a>
### 2026-08-27 · Month-0 row ratified as the schedule convention


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

**Status.** In effect. Consequence 1 means new continuity imports write the row; frozen legacy schedules get it derived at read time (2026-09-30). Wording settled 2026-10-20.

<a id="e-loud-null"></a>
### 2026-09-04 · Equipment-derived views: loud NULL over GT default; representative estimate = signed > approved > recent; watts formula gets one home

**Decision.** Three rulings from the governance walkthrough (Jake):
(1) `v_estimate_calcs` no longer falls back to 0.12 when an estimate's
site/country link is broken — tax_rate and retail_con_iva go NULL loudly
("prefer that the system breaks rather than assume Guatemalan tax rate").
(2) The estimate that represents a project (capacity, calc reads) is chosen
signed > **approved** > most recent — an approved-but-unsigned estimate
outranks a newer draft (14 projects changed representative at apply time).
(3) The Σ panels×watts formula has exactly one home:
`v_estimate_calcs.installed_capacity_kwp` (matched by `equipment_types.slug`,
never hardcoded type ids); `v_project_installed_kw` reads it rather than
re-summing. Third copy in dbt solar-savings remains a Phase-4 audit lead.

**Why.** The 0.12 fallback silently priced HN/SV estimates at GT IVA on any
broken link (0 such rows at apply time — the rule is prevention); the old
chooser let a newer draft displace an approved estimate; the formula lived
twice with different matching rules (id vs slug), identical only by luck.

**Where.** Infra migration
`2026-09-04-estimate-calcs-slug-match-and-installed-kw-one-home.sql`;
CHANGELOG same date; AUDIT-2026-08.md (dbt repo) Phase-2 section.

0 NULL tax rates and installed_kw ≡ calc layer).

**Status.** In effect.

<a id="e-stamping"></a>
### 2026-09-04 · Stamping doctrine: freeze base facts and client-facing commitments; derive aggregates live

**Decision.** (Jake) Write-time stamping applies to *base-level, client-facing
values* — the rate, the retail price, the per-month cash-flow rows the client
agreed to. *Aggregates* (contract totals, finance income, margins) are always
derived live from those static base facts, with exactly one definition: if
the definition is corrected, history restates — deliberately. Externally
shipped figures (Debita tapes) restate with a note, never silently.

**Why.** Proven the same day: the purchase-option epsilon bug in
v_active_projects' totals was healed retroactively on 14 projects *because*
totals are derived live. Had they been stamped at approval, the bug would be
frozen into history behind a backfill. Conversely, rates were being
re-derived and drifted (the 28-quote label backfill) — commitments need
freezing, summaries need one live definition.

**Where.** Precedents: approve-quote-variant rate stamp (supabase 8b2d045);
v_active_projects rollup rework
(`2026-09-04-v-active-projects-rollup-rework.sql`). Consequence for later:
the offer-JSON parsing in that view should eventually move to a calc-layer
view (one format-parsing home, e.g. v_offer_cell_flows) — NOT to stamped
totals, per this ruling.


**Status.** In effect.

<a id="e-custom-margin-09-14"></a>
### 2026-09-14 · Per-phase custom margin supersedes the provider margin (and rescues NULL-margin providers)

> **Superseded by 2026-09-15 (column) and 2026-10-01 (margin resolution).**

**Decision.** (Jake + team) The wholesale-margin override lives on
`project_phases.custom_wholesale_margin` (phase grain, fraction storage,
entered as percent in the estimate form). The engine resolves
`phase.custom_wholesale_margin ?? provider.standard_wholesale_margin`; the
missing-margin error now fires only when BOTH are null — an override
deliberately rescues a provider with no configured margin. Margin TYPE
(Add/Subtract) still comes from the provider/cartera. Endgame: retire the
margin-variant provider clones ("ESQUISOLAR MARGEN 20%" et al).

**Why.** The engine consumes margin per phase; multi-provider estimates have
per-phase margins; the clone was always a per-phase choice. Estimate grain
was considered and rejected (team ruling 2026-09-14).

**Where.** Column: infra `2026-09-14-custom-wholesale-margin-to-project-phases.sql`.
Engine: supabase d886f2d (deployed + tagged all 5 calc functions). Offer
page: infra `2026-09-14-offer-sheet-phases-custom-margin.sql`. Form:
ProjectPhasesManager per-row input. Verified: byte-identical prod A/B with
all overrides NULL; exact-formula positive cycle on 1050-13-01.


**Status.** Superseded: the column was renamed on 2026-09-15 and the provider-margin fallback removed on 2026-10-01.

<a id="e-margen-estimado"></a>
### 2026-09-15 · "Margen de venta (estimado)" — a ruled exception to no-frontend-calculation

**Decision.** (Jake) The calculadora header may show an *estimated* sales
margin computed in the browser — retail con IVA ÷ costo socio con IVA − 1 —
from Ian's PR #482. The no-business-calculation-in-the-frontend rule stands;
this is allowed because it is a display-only ratio of two engine outputs,
never stored or sent, labelled "estimado" with a tooltip that says what it
approximates and why it differs from the engine's per-phase, sin-IVA,
pre-discount margin. Any future exception needs the same: a comment naming the
rule and the ruling, and "estimado" in the label.

**Where.** `DefaultQuotesPageV2.tsx` header block (comment names this entry).


**Status.** In effect.

<a id="e-margin-override"></a>
### 2026-09-15 · Per-phase margin override — Ian's implementation is the one (column `margin_override_percent`)

**Decision.** (Jake) The 2026-09-14 entry below duplicated a feature Ian had
already built (infra PR #482 / supabase PR #26, 2026-09-11). Nobody had used the
new column (holidays), so the column is renamed to Ian's
`project_phases.margin_override_percent` and his form, plumbing and engine read
are the ones in the codebase. Everything else in the 09-14 entry still holds:
phase grain, fraction storage, percent entry, engine resolves
`phase.margin_override_percent ?? provider.standard_wholesale_margin`, direction
from the provider, NULL-margin providers rescued by an override.

**One behavioural difference.** Ian's input is shown only for Albedo-cartera
(Add) projects; the 09-14 input showed for every phase.

**Where.** Rename: infra `2026-09-15-rename-custom-wholesale-margin-to-margin-override-percent.sql`.
Engine: supabase main (quote-helpers). Form: `ProjectPhasesManager` `MarginOverrideInput`.


**Status.** In effect.

<a id="e-electricity-inflation"></a>
<a id="e-commission-is-cost"></a>
### 2026-09-15 · Commission is an Albedo cost, not part of the client's retail price

**Decision.** (Ian, issue #454; confirmed to the log 2026-10-20.) The sales
commission is no longer added to the retail price the client pays, for Albedo
and Socio cartera alike. It is still computed (`commission_cost_schedule`) and
counts as an Albedo cost in the cost ladder and the IRR. The tier bonus on
commission is retired at the same time (config columns set to 0).

**Why.** Ian's review against the Golden Copy: the commission is an expense of
the deal, not something the client finances; pricing it in made identical
deals price differently with data-entry completeness and inflated retail.
Supersedes the 2026-07-10 ruling that priced the commission type's total into
retail, and closes the 2026-05-14 open question.

**Where.** `supabase/functions/_shared/quote-helpers.ts`
`calculateProjectRetailWithCommission` (final retail = discounted base retail;
the field name `final_retail_with_commission` is kept for stored payloads);
migration `20260915130000_retail_price_changes_454.sql` (applied 2026-09-15).

**Status.** In effect since 2026-09-15.

<a id="e-legal-fee-is-cost"></a>
### 2026-09-15 · The legal fee is an Albedo cost as well as client income

**Decision.** (Ian, issue #333; confirmed to the log 2026-10-20.) The legal /
admin fee charged at signing counts as an Albedo cost in the amortization
table (`predebt_cost_sin_iva`, `expense_services`), in addition to being client
income. The 2026-05-14 rule that the cash-flow column is sin IVA is unchanged.

**Why.** The fee had been pulled out of the amortization earlier in 2026
because it broke the IRR; with the fee fixed to the con-IVA tier table the
cost side was put back so the deal's margin reflects what Albedo actually
spends on legal work.

**Where.** `supabase/functions/_shared/quote-integration-flow.ts`
`getAmortizationTable` (same commit as the commission change).

**Status.** In effect since 2026-09-15.

### 2026-09-19 · Electricity price inflation is a country constant: 5 % GT, 5 % SV, 6 % HN since 2026-09-22

**Decision.** The yearly inflation of the client's electricity price, used in
every savings projection (offer page, impact reports), lives as a column on `countries`
(`electricity_price_escalation_rate`, a fraction, next to `tax_rate` and
`grid_factor`), with change tracking. One value per country, no per-project
override. Loaded 2026-09-22 at Alex's request: Guatemala 0.05, El Salvador
0.05, Honduras 0.06 (previously 0.03 everywhere). The Valuing Impact
model's savings file is a 3 % snapshot: regenerate it for VI at 3 % only.

**Why.** Jake rejected a separate key-value constants table as overkill for
one value; a real constants table comes when the Valuing Impact parameters
(about 40) move into Solar Base. Changing the rate restates every savings
figure from the next rebuild (30-year gross savings rose about 45 % on the
Jun-30 cut), so the value is tracked and never defaulted.

**Where.** Infra migrations 2026-09-19 (column + audit trigger) and the
`get_public_offer_data` RPC; dbt `int_estimate_electricity_escalation` with
`POWER(1 + rate, year)` in both savings models and a test that every signed
estimate resolves a rate; frontend `utils/taxRate.ts` (baked fallback map,
kept equal to the table).

**Status.** In effect; the three values are live in production.

<a id="e-interest-accrued"></a>
### 2026-09-23 · Cash-flow interest column = interest *accrued*, not interest paid

**Decision.** (Jake) In materialized cash flows (`monthly_cash_flows.interest_payment_project_currency`),
the interest column states the interest that **accrued** that month — including
grace-period months, where the client pays nothing and the accrual capitalizes
into the balance. "The client makes one payment we break down on our side";
the breakdown should tell the truth about accrual, not report 0 because no
cash moved.

**Why.** The 2026-06-08 sign-time materialization mapped the engine's
`interest` field (interest *paid* — 0 during grace) and dropped
`interest_accrued`, silently changing the convention for every native signing
from July 2026 on (~14 projects): grace months showed 0 interest while the
saldo visibly grew. Erwin (accounting) flagged it on project 2373-01 —
accounting recognizes interest income monthly during grace, as all pre-July
schedules did. On payment months `interest_accrued == interest` (the payment
covers the accrual in full), so the mapping change only affects months with
no/partial payment — exactly the months where "paid" under-reports.

**Where.** Mapper: infra `packages/shared/supabase-sdk/src/client.ts`
(`interest_payment_project_currency` now prefers `cf.interest_accrued`, falls
back to `cf.interest` for older offer JSONs). Backfill of the ~14 post-July
signings: pending Jake's Zoho damage check (issue #446 thread has the related
month-0 questions for Ian).


**Status.** In effect; the 36-row backfill ran 2026-09-23.

<a id="e-cartera-on-estimate"></a>
### 2026-09-24 · Cartera is a property of the estimate; provider commercial terms are one versioned row; unsigned quotes float, signed quotes freeze

**Decision.** (Jake, with Ian's per-phase override as the given)

1. **Cartera lives on the estimate** (`estimates.cartera` ∈ `albedo` | `socio`),
   because it describes who sourced the deal, not who installs it. The engine
   takes the pricing *direction* from it. The provider row stops carrying a
   direction (`providers.quote_margin_type` becomes read-only legacy and is
   dropped after cutover).
2. **A provider's commercial terms are one versioned row** in
   `provider_payment_schedules` (to be renamed `provider_terms` once both
   frontends are off the old embed name): payout split **and** two margins,
   `albedo_margin` / `socio_margin`. The effective margin for a phase is
   `phase.margin_override_percent ?? terms.<cartera>_margin`. This is what lets
   the 43 margin-variant provider rows ("ESQUISOLAR MARGEN 20%", "INESO +",
   "JK ENERGY +" …) be deleted: variant = base provider + cartera + (rarely) a
   per-phase override.
3. **Unsigned quotes float, signed quotes freeze.** A phase is re-pinned to the
   provider's terms row *in force today* on every save (existing behaviour of
   `resolvePhasesWithPaymentSchedules`, kept deliberately): a quote is not a
   commitment until signed, so it is regenerated on whatever terms are in force
   when the rep regenerates it. Signed quotes are snapshots and never move.
   Versioning still buys two things: signed phases keep the row they were
   signed under (auditable by join), and future-dated changes ("90/10 from
   Oct 1") can be entered ahead of time — which requires the lookup to select
   by date (`valid_from <= today AND (valid_until IS NULL OR valid_until >=
   today)`), not by `valid_until IS NULL` as today.
4. **Correction vs renegotiation.** A typo on the current row is an in-place
   UPDATE (the audit trigger records it); a real change of terms is a new
   version with a `valid_from`. The provider form offers both.
5. **Sales owns schedule corrections.** 14 providers created 2026-06-17 →
   2026-09-17 sit on the trigger's `0 / 1 / 0` placeholder (the create form
   dropped the schedule fields). Nothing in the system can recover the intended
   splits (single trigger-written row each, never edited; no partner contract
   documents on their phases; post-QB). Three have a same-entity sibling with
   real terms (2032 Diseños Equipos y Sistema 10 % → 0.80/0.15/0.05; 2037 JK
   ENERGY + → 0.50/0.40/0.10; 2039 INESO + 5 → 0.60/0.30/0.10) — proposed to
   sales as defaults, not applied unilaterally.

**Why.** The provider-row-as-cartera model forced one provider row per
(entity × cartera × margin) and has bitten three times: the 2026-03-13 dedupe
that had to be reverted (consumers read the row), the 2026-08-21 inversion
(labels vs QB), and the 2026-09-24 IRR failure on 2804-01-01 (enganche above
the month-0 payout — 40 % on a 50/40/10 split, *any* enganche on 0/1/0).
QuickBase itself stored cartera per estimate (`Carteras` table, Estimates
f593/f594) independent of the variant row: even the "clean" ESQUISOLAR family
has 18 Albedo-cartera deals on the Subtract base and 24 Socio-cartera deals on
MARGEN 20 %. Alternatives considered: margins on `providers` (non-versioned) —
rejected because it leaves the payout split and the margin with different
histories for the same negotiation; pin-once terms for in-flight quotes —
rejected because an unsigned quote should reflect the terms in force at
signing.

**Multienergias is not a broken row.** The 2026-08-20 cartera audit flagged
provider 9 "ROTO" (base name ⇒ expected Socio) at *confianza baja*. Evidence
says Add 0.2 is right: QB `proveedores` #19 = Add/0.2 since 2025-03-19; QB's
own per-deal cartera on its 1,072 estimates = 1,027 Albedo vs 42 Socio; the
124 signed contracts price at retail ÷ cost ≈ 1.25. No wrong retail prices
went out. Techsol (76) *is* inverted vs QB but its margin is 0, so direction
never changed a price; it is flipped in the cleanup with no exposure.

**Where.** Migration
`albedo-automations-infra/database/migrations/2026-09-24-provider-terms-margins-and-estimate-cartera.sql`
(additive: columns, stamp, trigger default). Engine read:
`supabase/functions/_shared/quote-helpers.ts` (cartera → direction, terms →
margin; NULL cartera falls back to the provider direction until the frontend
writes it). Frontend: `solar_base_frontend` ProviderForm / ProjectPhasesManager /
QuoteWizardStep3Estimate / `resolveSocioSolarName`. Dedupe + variant deletion:
a later DATA migration that proves recomputed retail is unchanged for every
unsigned estimate before deleting a row.

dedupe) applied 2026-10-05 — see the 2026-10-01 entry.
Supersedes the pairing notes in the 2026-08-21 entry only in *where* cartera
is stored — the `Add` = Albedo mapping is unchanged and is what the stamp uses.

**Status.** In effect. The two-margins-per-provider part (item 2) is superseded by 2026-10-01.

<a id="e-no-interest-month-0"></a>
### 2026-09-25 · No interest at month 0 — the enganche is entirely principal

**Decision.** (Jake) A payment-0 row never shows interest: no time has passed
at signing. The enganche counts entirely as principal. Where the QB migration
transform charged one month of interest inside the enganche (59 legacy
quotes), that charge is presented as ordinary accrued interest in payment 1,
capitalizing into the saldo — the same convention as modern grace months.

**Why.** The transform split each enganche into interest + principal (the
enganche settled the first month's accrual at the first payment date). Three
presentations were tried: split in payment 1 (phantom principal on a
zero-payment row), split in row 0 (interest income at the signing instant),
and this one — which reconciles with the frozen tape to the cent by
construction, because the split identity (interest + principal = enganche)
is exactly the capitalization identity (saldo₁ = saldo₀ + interest₁).

**Where.** View: infra `2026-09-25-mv-cash-flows-month0-no-interest-at-signing.sql`
(v5 of `app.mv_monthly_cash_flows_v1`; presentation only, table untouched).
The month-0 table backfill (#505) must materialize this same shape.


**Status.** In effect.

<a id="e-na-placeholders"></a>
### 2026-09-25 · "NA" department + municipality placeholders, always last in dropdowns

**Decision.** Every country has exactly one department named `NA`, and that
department has exactly one municipality named `NA`. They are the only sanctioned
way to record "no real location" on a site/project — never leave a wrong
department picked, never invent per-country variants. In every dropdown the
`NA` rows sort last, after the alphabetical list.

**Why.** Sites with no usable location (placeholder projects, off-book
equipment, early drafts) were being parked under whatever department came
first. A shared placeholder keeps the FK chain intact without polluting the
real geography. Alphabetical order would bury `NA` mid-list, so the ordering
rule lives in one helper rather than per-dropdown hacks. The placeholders carry
NULL poverty/rural flags, so impact metrics treat them as unknown, not "no".
`departments.region_id` is NOT NULL and unread anywhere; GT's `NA` uses region 3
(Remote), SV 5, HN 4.

**Where.** Rows: infra migration
`2026-09-25-add-na-department-and-municipality-per-country.sql` (departments
57/58/59, municipalities 7/8/9). Ordering: `frontend/quotes-app/src/utils/locations.ts`
(`sortLocationsNaLast`), applied in `supabase-data-service.getDepartments` /
`getMunicipalities` and the KPI calculator's direct reads; mirrored in
`solar_base_frontend`. CHANGELOG entry 2026-09-25.

commit 74f29573).

**Status.** In effect.

<a id="e-canonical-cash-flows"></a>
### 2026-09-30 · One canonical cash-flows view; the month-0 table backfill is shelved


**Decision.** (Jake) Day-to-day consumers (the app, ad-hoc SQL, future calc
views) read ONE canonical view, `public.v_project_cash_flows`: chain-consolidated
live rows (`chain_payment_number`, month 0 at position 0) with the month-0
derivation baked in at read time, plus currency/IVA columns. The raw
`monthly_cash_flows` table stays frozen — historical reports (dbt, yearly
accounting, AR/EEFFs) keep their current inputs and never move. The #505
physical backfill is shelved: it remains a documented option, not a
prerequisite.

**Why.** The month-0 treatment was the only substantive disagreement between
read paths; unifying reports onto it means restating history, which we are
deliberately not doing. A read-time canonical view gives the "one view"
benefits without touching a row. Non-consumers by design: the contract
generator (frozen Anexos), v_project_signing / v_financing_date_ranges (date
anchors — the derived row-0 date comes FROM v_project_signing), dbt analytics.

**Where.** Infra `2026-09-30-v-project-cash-flows-canonical.sql` (applied;
`app.mv_monthly_cash_flows_v1` is now a byte-identical shim over it), dbt#9
(model mirrors the shim), infra #528 (repoint app + drop shim), #505 (shelved).


**Status.** In effect.

<a id="e-albedo-margin-12"></a>
### 2026-10-01 · Albedo margin is one going-forward default (12 %), not a provider attribute; socio discount stays per provider

**Decision.** (Javi, in a meeting with Jake 2026-10-01; recorded by Jake.)

- **Cartera Albedo:** the installer quotes us the *cost*. Every phase gets the
  same margin regardless of installer — default **12 %**, stored once in
  `quote_generator_configs.default_albedo_margin`, shown on the estimate form
  and adjustable per phase (`project_phases.margin_override_percent`). Retail =
  cost ÷ (1 − margin) — margin is a share of retail, the engine's existing
  convention (cost 100 → retail 113.64), not a markup on cost.
- **Cartera Socio:** the installer quotes the client's *retail*. We finance that
  retail and pay the installer retail × (1 − discount). The discount is the
  provider's (`provider_payment_schedules.socio_margin`) and is never shown on
  the form.
- **12 % is going-forward only.** Nothing already quoted moves: every Albedo
  phase that had no margin of its own got the margin it had been pricing with
  stamped into `margin_override_percent` (859 phases), so the engine switch is
  a proven price no-op.

**Why.** Under the previous model the Albedo margin was a provider attribute,
which is what forced one provider row per (installer × margin) — "ESQUISOLAR
MARGEN 20 %", "INESO + 5" — and left reps overriding downward by hand (live
overrides at 12 %, 10 %, 8 % existed before this ruling). Making it a single
default removes the only information the Add-variant rows carried, so the
dedupe becomes a pure re-pointing. The 2026-09-24 entry's "two margins per
provider" (`albedo_margin` / `socio_margin`) is superseded on the Albedo side;
cartera-on-the-estimate and the versioned terms row stand.

**Where.** Engine: `supabase/functions/_shared/quote-helpers.ts`
`resolvePhasePricingTerms()` (albedo → override ?? config default; socio →
override ?? terms.socio_margin ?? provider std); constant
`DEFAULT_ALBEDO_MARGIN` in `quote-constants.ts`. DB: supabase migration
`20261001120000_albedo_margin_default_and_stamp.sql` (config column, stamp,
`v_offer_sheet_phases`, `get_public_offer_data`). Frontend (step 3): the
per-phase margin input is prefilled with the config default on Albedo
estimates and hidden on Socio ones. `provider_payment_schedules.albedo_margin`
is deprecated, not yet dropped (`v_partner_contract_context` depends on it).

**Open.** Whether Tix Solar's socio discount is 3 % (in-app since 2026-07-06,
what has priced) or 5 % (QuickBase) — the migration keeps 3 %. Javi's margin
sheet shrinks to one column: the socio discount per company.

applied 04:14 UTC; supabase PR #27 and infra PR #533 merged and deployed). The
27 margin-variant provider rows are gone; full-table backups in
`backups.*_2026_10_04_cartera`.

**Status.** In effect since 2026-10-05.

<a id="e-signing-date-contract-only"></a>
### 2026-10-02 · Signing date comes from the client contract only — no quote fallback

**Decision.** A project's signing date is the `signed_on` of the client
contract on its ORIGINAL (root) quote, and nothing else.
`quotes.contract_signing_date` is no longer a fallback. It stays as a mirror
kept in sync by `trg_sync_quote_signing_date`. Adenda keep their own
contracts, but those never set the project's signing date. Tightens
"Project signing date = ORIGINAL contract date" (2026-08-25).

**Why.** Two sources for one date is how they drift. The fallback only ever
fired for 7 projects, all adenda chains where the 2026-06-01 QuickBase import
created a contract for the newest quote only.

**Where.** `v_project_signing` (infra migration
`2026-10-02-signing-date-contract-only.sql`, which first backfilled the 8
missing client contracts with their quotes' existing dates, marked
`data.backfill`). No displayed date changed.

**Open.** Adenda dates to check against the signed PDFs: 981-01/02/03
(adenda dated the same day as the original), 679-01 (original 2025-09-01,
seam 2025-08-31, both adendas 2025-08-28), 577-03 (adenda exactly one year
after the original).


**Status.** In effect 2026-10-02.

<a id="e-provider-terms"></a>
### 2026-10-09 · One provider terms row per provider; a Socio deal cannot be priced without the provider's discount

**Decision.** (Jake, 2026-10-02 and 2026-10-09; in effect since 2026-10-09 02:18 UTC.)

- A provider's commercial terms are **one versioned row** in `provider_terms`
  (formerly `provider_payment_schedules`): the installer payout split
  (`period_1/2/3_payout`, month 0 / 1 / 2, summing to 1) and the Socio
  discount (`socio_margin`). The payout split does not depend on the cartera,
  so the parallel `add_*` columns are gone; so is `albedo_margin` (the Albedo
  margin is the config default, 2026-10-01 entry).
- The provider row carries **no pricing information** any more: cartera lives
  on the estimate, the Albedo margin on the config / phase override, the
  Socio discount on the terms row. The estimate's old Add/Subtract select is
  gone too; `estimates.cartera` is the only cartera.
- **No discount means 0 %.** (Jake, 2026-10-09, same day.) A provider with no
  negotiated Socio discount carries `socio_margin = 0` on its terms row (NOT
  NULL, default 0): the deal is financed at the installer's retail and the
  installer is paid all of it — that is the deal, not an error. The migration
  applied at 02:18 had briefly made such phases refuse to price; the 13:00
  follow-up set the 19 empty rows to 0 and the engine prices them again
  (1511-02 Premium Value Services, 2801-01 AT ENERGY, 1511-01 GRUPO SEP).

**Why.** Jake 2026-10-02: "we only need one margin per provider … payouts
don't vary depending on add or subtract so we only need three payout
columns." The legacy columns were the last place a pricing input could hide
outside the three homes above; removing them makes "where did this price
come from" answerable from the estimate, the phase and the terms row alone.

**Where.** Supabase migrations `20261009120000_provider_terms_schema_cleanup.sql`
(renames, drops, proofs) and `20261009120100_estimates_drop_quote_margin_type.sql`;
engine `resolvePhasePricingTerms()` (cartera required, no provider fallback);
frontend provider form writes the terms row directly; contract lambda and
HubSpot read the terms row / the estimate's cartera. Infra CHANGELOG 2026-10-09.

- **Sales floor: 5 %.** (Jake, 2026-10-09.) On Albedo cartera a rep may lower
  a phase's margin below the 12 % default but not below 5 %; the estimate form
  refuses to save below that. UI guard only — the engine prices what is
  stored (27 phases carried overrides below 5 % that day, 1 signed; they keep
  pricing as quoted until edited). Socio phases carry the provider's
  discount, not a typed margin, so no floor applies.

**Open.** Nothing on pricing. "Add provider is inactive" (#387) still needs
Ian's one sentence.

**Status.** In effect since 2026-10-09.

<a id="e-maintenance-anchor"></a>
### 2026-10-12 · Maintenance visits are anchored on the Cambio de Medidor date

**Decision.** (Jake, 2026-10-12, after talking to Javi.) When ops registers
the **Cambio Medidor** milestone on the ops page, the app seeds the project's
maintenance calendar: `scheduled_date = fecha de cambio de medidor + maintenance_year × 12 months`,
one row per year up to the end of the lease, with the ordinal cost mapping
and flags of the 2026-10-09 seed. Ops confirms a dialog first ("esto creará
el calendario de mantenimientos… primer mantenimiento el <fecha + 12 m>").
Only signed Financiado projects with no maintenance rows; Contado and
already-seeded projects save the milestone normally. Correcting the cambio
medidor date later moves the maintenance dates with it while no visit is
Completado, and the UI says so. Trade-off accepted: a project signed from now
on does not appear on Mantenimientos until its cambio de medidor is
registered.

**History.** The 230 projects seeded before this ruling keep their dates: an
earlier anchor (Fernanda, 2026-06-04) used the client delivery date, else
the first paid cuota, and those rows are not re-anchored. An
installation-end anchor validated in May was never applied.

**Why.** The meter change is the moment the system starts producing for the
client, and ops records it in the app already; delivery dates were rarely
filled. One trigger, one dialog, no silent rescheduling.

**Where.** `governance-milestones` edge function (`_shared/maintenance/seed.ts`,
replaying the 2026-10-09 seed exactly); ops page milestone dialog
(`GovernanceTimelineWidget`); design doc `architecture/maintenance-sync-at-cambio-medidor.md`.
Failures land in `signing_failures` (stage `seed_project_maintenances`).

**Status.** In effect since 2026-10-12 (no project has reached Cambio
Medidor since, so no rows seeded by this path yet as of 2026-10-16).

<a id="e-sector-3"></a>
### 2026-10-16 · Multi-unit residential clients are sector 3 (Comercio minorista, the commercial bucket), not 10 (Casa habitacional)

**Decision.** (Convention in use since mid-2026; logged now.) When the client
is an asociación de vecinos, condominio or asociación civil that owns or
runs a residential building, `project_sector_id` is **3**, the commercial
catch-all. Sector **10** is reserved for a private individual's home. Judge by
the client's legal name, not the project name: "Residencial X" as a project
name does not make it residential. Precedents: 492, 1814, 2358.

**Why.** `project_sectors` has no multifamily or real-estate bucket, and a
building that collects fees from residents is run as a business. Sector
curation stopped being manual in January 2026, so the rule has to be
written down for the form.

**Where.** `projects.project_sector_id`; QB free-text sectors flow into
`colloquial_name` only.


**Status.** In effect.

<a id="e-one-config"></a>
### 2026-10-20 · One quoter configuration for everyone

**Decision.** (Jake.) The quoter has one set of defaults, the single active
row of `quote_generator_configs`, read by every calc edge function. There are
no named, versioned or per-user configurations: the three rows created in
November 2025 collapse to one (`default`), the per-user override table is
dropped, and the ten columns no code read are dropped with it.

**Why.** The multi-config design was never used: every quote priced off the
active row, the per-user lookup never matched (keyed by email while the app
sends a numeric id), and the unread columns made the admin page a placebo
("24-month free period" edited nothing). One engine, one quoter, same numbers
for everyone; the editable defaults stay in the table so admins can change
them without a deploy (the 2026-10-01 Albedo margin lives there).

**Where.** Supabase migration `20261020200000_one_quote_generator_config.sql`;
engine `getActiveConfig` (`_shared/quote-helpers.ts`); admin page
`/quote-config-admin` (edits the one row). Infra CHANGELOG 2026-10-20.

**Status.** In effect since 2026-10-20.

<a id="e-free-maintenance-years"></a>
### 2026-10-21 · Two free years of maintenance by default; a rep may shorten it per deal

**Decision.** (Jake, closing the May open item.) A financed client gets the
first two annual maintenance visits at no charge: maintenance enters the
schedule from month 36 (engine default, the estimate form's "Año 3
(predeterminado)"). A rep may shorten the free window for a specific deal
by choosing Año 1 or Año 2 on the estimate form; that is the only per-quote
adjustment and it is deliberate. The client's maintenance payment is the
visit costs with the 30 % markup spread flat over every cuota after grace,
so the free years lower the monthly amount rather than removing it (by
design). The visit calendar is dated from Cambio de Medidor regardless.

**Why.** The May 2026 request for an adjustable guarantee period was met by
the dropdown; nobody closed the item. The config column
`maintenance_free_period_months` (24) was never read by the engine and was
dropped on 2026-10-20, so the form is the one place the window is set.

**Where.** Engine `resolveMaintenanceStartMonth` (`_shared/quote-helpers.ts`:
request override → `estimates.maintenance_start_year × 12` → 36);
`MaintenanceInsuranceSection` dropdown; visit costs mapped per contract year
in `_shared/maintenance/seed.ts`. Measured 2026-10-20 on 230 projects: 128
two free, 14 one free, 61 none (mostly legacy schedules), 24 never charged
(Loose Ends).

**Status.** In effect.

<a id="e-quote-status-one-home"></a>
### 2026-10-21 · Quote status transitions have one home: service_role code, enforced by column grants on `quotes.status`

**Decision.** (Jake, 2026-10-21.) `quotes.status` is written only by
service_role code: `approve-quote-variant` (into approved),
`disapprove-quote-variant` (approval withdrawn by offer), `set-quote-status`
(the remaining transitions: pending → rejected | cancelled; approved →
rejected | cancelled | pending; rejected → pending | cancelled), the
contract-generator Lambda (into signed) and `annul_project()` (into
annulled). No browser session writes the column: the `authenticated` role
loses INSERT and UPDATE on `quotes.status` through Postgres column-level
grants (every other column stays writable), so the database refuses the
write rather than trusting every screen to remember the rule. A new column
on `quotes` must be granted to `authenticated` in the migration that adds
it. Supersedes the project page's direct status write (the "Acciones"
dropdown, which also offered "Aprobar"); no earlier entry covered it.
Complements the 2026-07-24 entry on what `status` means.

**Why.** Two paths wrote the same column. The function path enforces one
approved quote per project, the signed-block and the addendum linkage,
stamps `approved_at` / `approved_by`, copies the offer's solved rate onto
the quote (contract-rate ruling 2026-08-26) and marks the offer. The
dropdown path skipped all of it: a contado quote on project 1835-01 was
"approved" there on 2026-10-15 and never reached the Finanzas Diligencia
page. Since June 2026, 111 transitions into approved: 69 through the
function, 42 direct (37 June/July imports and scripts, 5 from the
dropdown). RLS cannot express "this column, not this row", so the grant is
the enforcement and the functions are the only writers left.

**Where.** supabase repo: edge function `set-quote-status` (rule and tests
in `_shared/quote-status-transitions.ts`), migration
`20261021180000_quotes_status_column_grant.sql` (AlbedoSolar/supabase#59);
project page `ActiveProjectDetails.handleStatusChange`
(AlbedoSolar/solar_base_frontend#50, mirror
AlbedoSolar/albedo-automations-infra#606); infra
`database/policies/signed-protection.md` → "Column grants" and the
CHANGELOG 2026-10-21 entry (AlbedoSolar/albedo-automations-infra#607).

**Status.** Decided 2026-10-21; in effect once the migration is applied
(after the function and the frontend ship).

<a id="e-db-compute-refuse-cascade"></a>
### 2026-10-21 · Database code computes nothing; it may refuse, and may cascade only when audit-only or ruled and documented

**Decision.** (Jake, 2026-10-21.) Three verbs decide what belongs in
Postgres code. **Compute** — never: derived values live in views, edge
functions or dbt (the four homes in `architecture/`), not in functions or
triggers. **Refuse** — yes: the database may reject a write that would break
an invariant, through RLS, column grants, constraints or a trigger that
raises. **Cascade** — only when the cascade is audit-only (it records, it
changes no business data) or when the rule it applies has been ruled and is
documented in infra `database/triggers/`. Every new trigger justifies itself
against this before it is written; "it is convenient" is not a
justification. The cascade triggers that exist today are grandfathered and
listed in infra `database/triggers/`: workflow-status promotion
(`trg_quotes_promote_project_workflow`, `trg_estimates_promote_project_workflow`),
maintenance seeding at Cambio de Medidor, and change tracking
(`trg_track_changes_*`, the audit log).

**Why.** Each trigger is a place where a write does more than it says; the
cost shows up as a report that disagrees with a screen, or a backfill that
fires side effects nobody intended. The repo rule "logic lives in code, not
in Postgres" already names the allowed exceptions; this entry states the
test behind them so the next trigger is argued the same way. The
`quotes.status` ruling above is the refuse case: the grant says no, the
code does the work.

**Where.** Repo rules: supabase and infra `CLAUDE.md` ("Logic lives in code,
not in Postgres"); infra `database/triggers/README.md` (trigger policy and
the current list); this log for the grandfathered cascades
(workflow-status promotion 2026-07-13, maintenance anchor 2026-10-12).

**Status.** In effect.

<a id="e-open-mixed-commission"></a>
### Open · Commission on mixed Add/Subtract phase projects

> **Closed 2026-10-19 (Jake): moot.** Since 2026-09-24 cartera lives on the estimate, so every phase of a quote prices in the same direction; and since 2026-09-15 commission is not part of the retail price at all.

For projects mixing "Add" and "Subtract" margin-type providers, commission is added to the aggregate retail unconditionally — double-counting commission on the "Subtract" phases (which have it baked in already). Per `4-known-inconsistencies.md` § 1.3. Only impacts projects with both margin types in the same quote. Need a business decision on the right treatment.

**Status.** Closed 2026-10-19: moot (see banner).

<a id="e-open-off-by-one"></a>
### Open · Off-by-one in service payment spreading denominator

Insurance/maintenance total cost is divided by `term - gracePeriod + 1` but applied across `term - gracePeriod` months — a ~1.6% systematic under-collection. Code comment says "Match R code... to allow for discounts". Need confirmation: intentional client discount or legacy bug? Per `4-known-inconsistencies.md` § 1.5.

**Status.** Open.

<a id="e-open-insurance-base"></a>
### Open · Insurance base: installation cost or financed retail?

> **Closed 2026-10-20: retail con IVA**, as the 2026-05-14 "Service bases" entry says; confirmed by Ian with operations.

Insurance cost is computed as `insuranceRate × installation_cost_sin_iva`. If the insurance is meant to cover physical asset replacement, the installation cost is correct. If it's meant to protect Albedo's receivable (the financed retail price, ~15-25% higher), there's structural under-insurance. Need clarification from the business on what the insurance covers. Per `4-known-inconsistencies.md` § 3.2.

**Status.** Closed 2026-10-20: retail con IVA.

<a id="e-open-fx-timing"></a>
### Open · Exchange rate timing risk

Currency conversion uses a single FX snapshot at quote time. Provider payouts span 3 periods, client payments span 5+ years — the snapshot doesn't reflect realized exchange movements. Is this an accepted business risk, or should we build periodic re-quoting / sensitivity analysis? Per `4-known-inconsistencies.md` § 3.3.

**Status.** Open.

<a id="e-open-payout-sum"></a>
### Open · Provider payout schedule validation (sum ≠ 1.0)

> **Closed 2026-10-20 (Jake): done.** `provider_terms_payout_sum_chk` enforces that the three payouts sum to 1.

`provider_payment_schedules.period_{1,2,3}_payout` aren't validated to sum to 1.0. A data-entry error would silently distort installation totals (and every downstream calc). No business decision needed — just add the constraint. Per `4-known-inconsistencies.md` § 3.5.

**Status.** Closed 2026-10-20: the constraint exists.

<a id="e-open-maint-inflation"></a>
### Open · Maintenance inflation baseline (NII vs integration flow)

> **Closed 2026-10-20 (Jake): moot.** The NII engine was deleted on 2026-07-13; manual mode runs the shared engine, so there is one baseline.

The NII table (manual mode) and the integration flow count maintenance inflation years from different baselines. Same parameters, different results. Per `4-known-inconsistencies.md` § 2.3. Either pick one or document why they differ.

**Status.** Closed 2026-10-20: moot since 2026-07-13.

<a id="e-open-impact-home"></a>
### Open · Impact-indicator canonical home (clients vs projects vs sites)

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

**Status.** Open. Tracked as infra issue #436.

<a id="e-open-otros"></a>
### Open · "Otros" category for non-gendered clients

Stephanie proposed adding an "Otros" client gender category for sociedades / organizations without a leadership gender. Per Apr 8 ADA meeting. Confirm with Stephanie where the manual classification comes from (Excel? QB?) before implementing.

**Status.** Open.

<a id="e-open-co2-retro"></a>
### Open · CO2 retroactive change handling in ADA reports

When QB → Supabase migration changes CO2 cleanups, ADA's Q3/Q4 reports show different historical numbers. Need an alignment on whether to: (a) color-code/version flagged retroactive changes, or (b) freeze historical numbers and surface only forward changes. Per Apr 8 ADA meeting.

**Status.** Open.

---

<a id="e-impact-flags-one-home"></a>
## 2026-10-28 · Impact indicators have one home: client answers, site location

**Decision.** The six impact indicators are resolved in one place, the
database view `v_project_impact_flags`, and everything reads it: the quote
engine's impact discount, the project and operations pages, and every
report (impact summaries, ADA, ATTA, investor marts, Valuing Impact table).
- Women-led, youth-led, educational institution and non-profit are
  attributes of the **client**, answered Sí / No / Sin confirmar on the
  client form. Educational and non-profit stay on the client rather than the
  site: a school chain or an NGO is the same organisation at every site.
- Rural comes from the site's GPS density code (11 or 12 = rural). A project
  **without coordinates gets no rural point** in pricing and reads "not
  sure" in reports until ops adds them.
- Low-income area comes from the site's municipality (poorest 20 %).
- The project's own flag columns are legacy. During the transition a client
  with no answer falls back to the project value; the fallback and the
  columns go when every signed client is answered.
Supersedes [2026-05-14 · Impact indicators move from projects to canonical
homes](#e-impact-indicators-home). Applies [2026-07-09 · Rural = GPS
only](#e-rural-gps-only) to pricing and to the ADA/ATTA reports, which still
fell back to the manual flag.

**Why.** Three sources answered the same six questions. Pricing and the
project forms used the manual project flags, reports used the manual flags
for four indicators and GPS for rural, and the client form (since May) wrote
to the client where nothing read it. 110 signed projects disagreed between
the manual rural flag and GPS, and an export to Valuing Impact showed 37
rural projects where the rule gives 105. Alternatives: educational and
non-profit on the site (rejected: only four clients disagree across
projects, all data-entry conflicts, not multi-site organisations); keep
scoring rural from the manual flag when GPS is missing (rejected: rewards
missing data and contradicts the July ruling).

**Effect.** 189 clients received their answers from the agreement of their
signed projects. Reports moved by about one point (youth-led 13 → 14 %,
"any impact" 63 → 64 %; ADA unchanged). 35 of 359 open deals gain one or two
impact points on their next quote, none lose any. Signed quotes are frozen.

**Where.** supabase migration `20261028150000_impact_flags_canonical_home.sql`
(supabase#68); quote engine `quote-helpers.ts` (supabase#71); dbt
`int_project_impact_flags` and twelve report models (dbt#30, CALCULATIONS.md
§ "Impact indicators"); frontend client forms, project page, Impact page
(solar_base_frontend#60, infra mirror #628). Tests
`test_impact_flags_match_canonical_view` and
`test_signed_clients_have_impact_flags` (the work queue of unanswered
clients). Deprecation: albedo-automations-infra#616.

**Open.** Who edits the answers on the Impact page: clients are writable by
sales and admin only, so operations can no longer edit them there. Widening
that is a permission-matrix decision.

**Status.** In effect 2026-10-28 for reports and the app's views; pricing
once supabase#71 merges; forms once solar_base_frontend#60 merges.

## Entry template

```
<a id="e-short-slug"></a>
### YYYY-MM-DD · One-line title summarising the decision

**Decision.** The rule, in at most three sentences. (Who decided, when.)

**Why.** Context and the alternatives considered.

**Where.** File, view, migration or PR. Link back here from the code comment.

**Status.** Open / Decided / In effect / Superseded by <date>.
```

Then add the one-line register row. If the entry changes an earlier one, mark
that entry Superseded, point both ways, and remove its register line.
