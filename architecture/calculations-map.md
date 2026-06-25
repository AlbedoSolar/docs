# Calculations Map

**Purpose:** A single inventory of *where every business calculation lives today*, so we can consolidate toward one definition per calculation. Working map for the calc-consolidation effort (shared language: SQL / live views).

**Last updated:** 2026-06-25, after a six-surface verification audit (edge functions, lambdas, both frontends, dbt, Postgres). File:line refs are starting points — verify before editing; the codebase moves fast.

## Surfaces where calculations live

| Surface | What runs there | Status |
|---|---|---|
| **TS edge functions** (`supabase/supabase/functions/`) | The live quote/finance engine | ✅ canonical |
| **TS lambdas** (`packages/lambdas/`) | `contract-generator`, `supabase-client-handler`, `quickbase-import-kickoff` carry real finance calcs | ⚠️ duplicated/divergent |
| **Frontend — quotes-app** | Quote/offer display derivations | ✅ |
| **Frontend — solar_base_frontend** | Projects/finances; mostly *reads* views, some inline calc | ✅ |
| **dbt** (`dbt/main/`) | Analytics + the `app.*` live views the app reads | ✅ |
| **Postgres views/functions** | `v_active_projects`, RLS helpers, status views | ✅ |
| **Python** (`python-calculations`, `quote-generator`, `calculations-service`) | Source present in repo but **NOT deployed** (CDK migrated to edge fns) — dead/unused | ⚠️ present but dead |

> ⚠️ **Python is NOT deleted.** A 2026-06-17 deletion in this workstream did not persist (uncommitted; repo has merged forward since). The dirs are back. They're dead/unused (not deployed) but they exist — treat as dead, don't extend, remove only via a deliberate committed change.

Two live frontends both consume these. Only true language boundary is **TS↔SQL** — TS↔TS can share a package; TS↔SQL bridges via persisted columns or a shared view.

---

## A. Quote financial engine (cashflow + KPIs) — bigger than first mapped

Live home: `supabase/supabase/functions/_shared/` (+ `calculate-interest/`). Results persisted to `monthly_cash_flows` + `quotes`. The engine is a multi-file pipeline; the audit found these pieces (most were missing from the first draft):

| Piece | Location | Notes |
|---|---|---|
| Forward calc pipeline | `_shared/quote-forward-calc.ts` (`runForwardCalc`) | One full cashflow+KPI pass; called per iteration by solvers |
| Goal-seek solvers | `_shared/quote-goal-seek.ts` (`goalSeekAprByIrr`, `goalSeekAprByMonthlyPayment`, `bisect`) | Reverse-solve APR; used by quote-generator, quote-solver, irr-calculator |
| Lease / annuity | `_shared/quote-integration-flow.ts` (`calculateLease`, `calculatePayment`) | Grace-aware amortization |
| Services schedule | `_shared/quote-integration-flow.ts` (`serviciosTable`) | Insurance (deflation) + maintenance (inflation) + legal; supports per-partner fixed annual cost |
| Combine + integration KPIs | `_shared/quote-integration-flow.ts` (`combineLeaseServe`, `getIntegrationKpis`, `getAmortizationTable`) | ⚠️ **parallel KPI path** — may duplicate `getKpis()` in quote-calculator |
| Impact-score / APR discount | `_shared/quote-impact.ts` (`calculateImpactScore`, `lookupImpactAdjustments`) | Impact-aware tier pricing (irr-calculator) |
| Rate lookups (waterfalls) | `_shared/quote-helpers.ts` (`getMaintenanceRate`, `getInsuranceRate`, `getProviderPricing`) | dept > region > country > config |
| Installation schedule | `_shared/quote-helpers.ts` (`calculateInstallationSchedule`) | phases → [p1,p2,p3], default [0.5,0.4,0.1] |
| Installed kW | `_shared/quote-helpers.ts` (`getEstimateInstalledKw`) | sums panel capacity for maintenance pricing |

**Confirmed drift inside the engine (TS-internal, real):**
- **IRR:** `irrBisection` (KPIs, quote-calculator) + `calculateIRRForProjectLegacy` + `calculateXIRRForProject` (calculate-interest) all coexist.
- **Amortization:** `calculateAmortization` (daily-prorated) vs `calculateAmortizationLegacy` (simple monthly) in one file, PLUS `amortForm`/`amortTimeForm` in quote-calculator. 3-4 implementations.
- **Tax/IVA per row:** quote-calculator + serviciosTable both apply it.

---

## B. Retail price / margin / cartera — drift was OVERSTATED

| Calculation | Live home(s) | Revised verdict |
|---|---|---|
| Retail price (Add/Subtract margin) | edge `_shared/quote-helpers.ts` (`calculateRetailPriceComponents`, `calculateProjectRetailWithCommission`) | ✅ single canonical (Python copy is dead) |
| Gross margin / Margen Bruto | each frontend: `utils/retail-preview.ts` `computeGrossMargin()`, used by FinancesDiligencePage + ActiveProjectDetails | ✅ **within each app it's ONE shared helper** (not 3 drifting copies) — map was wrong |
| Retail preview from phases | `retail-preview.ts` `computeRetailPreviewFromPhases()` + consumers | ✅ shared helper |
| IVA con/sin gross-up | `utils/retailDisplay.ts` `applyIva()`, `utils/totalContractValue.ts` | ✅ shared helper |
| Gross margin (quote, backend) | edge `_shared/quote-calculator.ts` | ⚠️ same concept as frontend helper + dbt |

⚠️ **Real remaining risk: cross-APP duplication.** `retail-preview.ts` / `retailDisplay.ts` / `totalContractValue.ts` exist **separately in both quotes-app and solar_base_frontend**. Within an app they're shared; across the two apps they're copies that can drift. → candidate for a shared TS package.

---

## C. Estimate-level derivations

| Calculation | Live home(s) | Verdict |
|---|---|---|
| **System type / grid (Ongrid/Offgrid)** | `ClientOfferPage.tsx` ~line 934 | 🔴 **HARDCODED `"Ongrid"`** — no battery derivation in code. (The "derive from batteries" rule is agreed but unimplemented; an earlier edit didn't persist.) dbt investor mart separately uses `estimates.has_batteries`. |
| Installed capacity (kWp) | `ClientOfferPage.tsx` (`installed_potential_w/1000`, fallback sums Panel Solar equipment) | low risk |
| Savings %, bill-after-solar, annual savings | `ClientOfferPage.tsx` | display-only, single page |
| 30-yr escalated savings plan (3%/yr) | `ClientOfferPage.tsx` (`buildYearlyPlan`) | mirrors dbt impact escalation — watch |
| CO₂ (grid_factor × lifetime kWh) | `ClientOfferPage.tsx` + `utils/taxRate.ts` const | mirrors dbt `mart_impact_projects_summary` |
| Electricity price → USD | dbt `int_estimates_convert_currencies` only (legacy-currency fallback) | ✅ keep in dbt |

---

## D. Project-level rollups — 🔴 THE BIG ONE (4-way overlap + tax-semantics bug)

The same rollups are computed in **four** places:

1. `public.v_active_projects` (Postgres live view)
2. dbt `app.mv_projects_v1` → `int_projects_mega_view` (+ `_gtq`/`_usd` variants)
3. lambda `contract-generator` `buildSignedProjectSnapshot()` (`helpers.ts:214-258`)
4. dbt data-chat views `v_projects_gtq` / `v_projects_usd`

| Rollup | v_active_projects | int_projects_mega_view | Agreement |
|---|---|---|---|
| total_contract_value | ✅ | ✅ `total_value` | 🔴 **DRIFT: v_active_projects is ×(1+tax) [post-IVA]; mega_view is raw [pre-IVA]** |
| finance income / insurance / maintenance / legal | ✅ (post-IVA) | ✅ (pre-IVA) | 🔴 **same tax-semantics drift** |
| gross_investment | ✅ | ✅ | ✅ agree (both pre-IVA) |
| project_status / duration / grace_period | ❌ | ✅ | only mega_view |
| quotes_count / estimates_count | ✅ | ❌ (in `int_active_projects_mega_view`) | inconsistent placement |
| approved_* shortcuts (term, dp, retail sin/con IVA, monthly pmt) | ✅ (2026-06-22) | ❌ | v_active_projects only; `approved_retail_con_iva` = retail×(1+tax), undocumented in CALCULATIONS.md |

🔴 **Critical:** the tax-semantics mismatch means the two live views return *different numbers for the same project's totals*. This is the highest-priority item — a correctness bug, not just duplication. Plus contract-generator computes its own copies (with `deriveEffectiveTaxRate()` reverse-engineering IVA from `income_total`).

**Perf note:** `v_active_projects` full-list load was 8.8s; one index on `estimates(project_id)` → 0.2s (44×). Index not yet applied.

---

## E. Payment / credit / receivables

| Calculation | Live home | Verdict |
|---|---|---|
| credit_status, DPD, paid on-time/late | `solar_base_frontend/.../PaymentSummary.tsx:101-165` | ⚠️ frontend-only; brittle |
| — DPD offset `-10` | `PaymentSummary.tsx:128` | 🔴 unexplained magic number; no documented dbt equivalent |
| — pre-data cutoff `2025-01-01` | `PaymentSummary.tsx:23` | 🔴 hardcoded, will age out |
| — credit_status thresholds (Vigente/Mora/Default) | `PaymentSummary.tsx:132-138` | ⚠️ if dbt has different thresholds → silent drift |
| — invoice_status enum matching (case-insensitive) | `PaymentSummary.tsx` (6 sites) | ⚠️ breaks if Zoho changes casing |
| Accounts receivable (short/long term) | dbt `int_official_accounts_receivable_by_project_gtq` | ✅ dbt |
| Remaining principal | dbt `int_remaining_principal_by_project_month_gtq` (+usd) | ✅ dbt |

---

## F. Impact / analytics (dbt-only — leave in dbt) ✅ verified

`monthly_solar_savings`, total electricity production (30yr), CO₂ avoided, `production_start_date` (waterfall), rural_area (smod 11/12), impoverished_area (in_poorest_20_percent), electricity price USD. **Additions found:** `accumulated_savings_to_date`, `average_monthly_solar_savings`. App reads these via `mv_impact_project_summary_v1` (true materialized view) — does not recompute. ✅

---

## G. Contract-generator calcs (lambda) — NEW, was missing

`packages/lambdas/contract-generator/src/helpers.ts`:
- `deriveEffectiveTaxRate()` (502-525) — reverse-engineer IVA from `income_total` so contracts reproduce the approved offer exactly
- `buildPaymentRows()` (559-657) — per-row con-IVA, solar cuota, closing extras
- `buildSignedProjectSnapshot()` (214-258) — project rollups (total_value, finance income, gross_investment, insurance/maintenance/legal, duration, grace) → **duplicates Domain D**

---

## H. Investor / ADA marts (dbt) — NEW additions

`high_social_impact_project` (women/youth/rural/educational/non-profit/impoverished), `savings_to_contract_value_ratio`, `annual_interest_rate` (= monthly×12), `finance_years` (floor(months/12)), `remaining_term_months`, `financing_start/end_date` (min/max payment_date). Several not in CALCULATIONS.md.

---

## I. UI-only mappings (frontend shared const, not a view)

Workflow/diligence status → badge/chip/allowed-transitions, duplicated across `ProjectsTable`, `ActiveProjectDetails`, `ProjectSearchBar`, `FinancesDiligencePage`. → one shared TS const.

---

## Other DB-resident derivations (not calcs to consolidate, but mapped)

- Views: `financing_date_ranges`, `pipeline_metrics`, `pipeline_timeline`, `quote_latest_status`, `quote_approvals`, `quote_signatures`, `v_maintenance_anchor_drift`, `active_quote_generator_config`, `user_roles_view`.
- RLS-embedded derivations: `is_signed_quote_status()` → `estimate_is_signed()` / `quote_is_signed()`.
- Triggers that compute (not audit): `set_project_phase_payment_schedule`, `create_default_provider_payment_schedule`.
- **No STORED generated columns** exist (confirmed).

---

## Drift register (prioritized for consolidation)

1. 🔴 **D — project rollups tax-semantics bug** (v_active_projects post-IVA vs mega_view pre-IVA) + 4-way overlap. Correctness + highest value.
2. 🔴 **A — finance-engine internal drift** (3-4 IRR / amortization implementations; integration-KPI vs getKpis duplicate path).
3. ⚠️ **Lambda finance copies** — `supabase-client-handler/financial-calculations.ts` is DIVERGENT/stale vs edge (different IVA handling, missing legacy amort); `quickbase-import-kickoff` has its own IRR; contract-generator rollups (G).
4. ⚠️ **C — grid type** unimplemented (hardcoded) and contradicts dbt has_batteries.
5. ⚠️ **B — cross-app TS duplication** (retail-preview/retailDisplay/totalContractValue in both frontends).
6. ⚠️ **E — PaymentSummary** brittle magic numbers (DPD −10, 2025-01-01 cutoff, status thresholds).

## Decisions to make with Ian
- **D first:** pick ONE canonical project-rollup definition and ONE tax convention (post- vs pre-IVA per field). This is a correctness fix, not just cleanup.
- For each cross-TS/SQL calc: live view vs persisted column? (Default: live view + indexes; persist only if proven too slow.)
- Naming: rename `mv_*` → `v_*` (they're plain live views, not materialized — except `mv_impact_project_summary_v1` which IS materialized).
- Remove dead Python deliberately (committed).
