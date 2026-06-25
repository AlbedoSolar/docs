# Calculations Map

**Purpose:** A single inventory of *where every business calculation lives today*, so we can consolidate toward one definition per calculation. This is the working map for the calc-consolidation effort (shared language: SQL / live views).

**Last updated:** 2026-06-25. Built from the 2026-06-16 four-surface survey + live DB inspection. File:line refs are starting points — verify before editing.

## Surfaces where calculations live

| Surface | What runs there | Live? |
|---|---|---|
| **TS edge functions** (`supabase/functions/`) | The live quote/finance engine | ✅ canonical |
| **TS lambdas** (`packages/lambdas/`) | A few copies of edge logic (e.g. `supabase-client-handler`) | ⚠️ duplicated |
| **Frontend — quotes-app** | Quote/offer display derivations | ✅ |
| **Frontend — solar_base_frontend** | Projects/finances; mostly *reads* dbt views | ✅ |
| **dbt** (`dbt/main/`) | Analytics + the `app.*` live views the app reads | ✅ |
| **Postgres views/functions** | `v_active_projects`, RLS helpers, etc. | ✅ |
| **Python** (`python-calculations`, calc lambdas) | — | ❌ DELETED 2026-06-17 |

Two live frontends both consume these. The only true language boundary is **TS/SQL** — TS↔TS can share a package; TS↔SQL bridges via persisted columns or a shared view.

---

## A. Quote financial engine (cashflow table + KPIs)

The core: given an estimate + parameters, produce the month-by-month cashflow and KPIs. **Results are persisted** to `monthly_cash_flows` + `quotes`, so dbt/app read stored numbers (good — compute once).

| Calculation | Live home | Also computed in | Drift risk |
|---|---|---|---|
| IRR / XIRR | `calculate-interest/utils/financial-calculations.ts` | duplicated in `supabase-client-handler` lambda; legacy bisection still in `_shared/quote-calculator.ts` | ⚠️ legacy-vs-current; dup copy |
| Amortization (principal/interest/balance) | `calculate-interest/.../financial-calculations.ts` | `calculateAmortizationLegacy` still present in same file | ⚠️ two formulas, one file |
| APR / recommended rate | `_shared/quote-calculator.ts` (step-table) | — (Python linear version deleted) | ✅ now single |
| Insurance & maintenance schedules | `_shared/quote-calculator.ts` | — | ✅ |
| Commission schedule | `_shared/quote-helpers.ts` | — | ✅ |
| Tax / IVA per row | `_shared/quote-calculator.ts` | dbt `mv_monthly_cash_flows_v1` | ⚠️ two surfaces, same rule |
| NPV / payback / cumulative NII | `_shared/quote-calculator.ts` | — | ✅ |
| Gross margin (quote) | `_shared/quote-calculator.ts` | frontend `retail-preview.ts`, dbt | ⚠️ see §B |

**Consolidation target:** keep ONE TS implementation; collapse the legacy/dup copies into a shared module imported by edge + lambda. Persisted results stay the source for dbt/app.

---

## B. Retail price / margin / cartera

| Calculation | Live home(s) | Drift risk |
|---|---|---|
| Retail price (provider margin Add/Subtract) | `_shared/quote-calculator.ts` | (Python copy deleted) — verify no other |
| Retail preview from phases | frontend `utils/retail-preview.ts` (+3 consumers) | ⚠️ frontend-only, re-derived |
| Gross margin / Margen Bruto | `retail-preview.ts`, `FinancesDiligencePage.tsx` (inline), `ActiveProjectDetails.tsx`, dbt | 🔴 3 frontend copies + dbt; FinancesDiligence has its own IVA gross-up |
| Cartera classification (Albedo/Socio) | baked into `deriveCostAndRetail()` | ⚠️ |
| IVA con/sin gross-up | `FinancesDiligencePage` inline + helper elsewhere | ⚠️ |

**Consolidation target:** these are cheap-relational + cross-language (frontend + dbt) → a **shared SQL view** (`v_estimate_financials`) both read; frontend helper becomes display-only or reads the view.

---

## C. Estimate-level derivations

| Calculation | Live home(s) | Drift risk |
|---|---|---|
| System type / has_batteries (grid type) | `ClientOfferPage.tsx` (equipment string-match) **vs** dbt investor mart (`estimates.has_batteries` column) | 🔴 two definitions can contradict |
| Savings %, annual savings, installed kW | `ClientOfferPage.tsx` | ✅ low (simple, single page) |
| Electricity price per kWh (USD) | dbt `int_estimates_convert_currencies` only | ✅ keep in dbt (legacy-currency fallback complexity) |

**Consolidation target:** grid type → seed view `v_estimate_derived` (`equipment_type_id = 3`), reconcile the stored column. (This was the original task.)

---

## D. Project-level rollups — ⚠️ THE BIG OVERLAP

The same project rollups are computed in **three** places:

| Calculation | `public.v_active_projects` (live view) | dbt `app.mv_projects_v1` (live view) | dbt `int_projects_mega_view` |
|---|---|---|---|
| total_contract_value | ✅ (added 2026-06-18) | ✅ (`total_value`) | ✅ |
| gross_investment | ✅ | — | ✅ |
| finance income / insurance / maintenance / legal totals | ✅ | partial | partial |
| approved quote shortcuts (term, dp, retail, monthly pmt) | ✅ (2026-06-22) | — | — |
| quotes_count / estimates_count | ✅ | — | — |
| project_status (Finalizado/Activo) | — | — | ✅ |
| duration / grace_period | — | — | ✅ |

- `v_active_projects`: **22 correlated subqueries**, was 8.8s on the full list → **0.2s with one index** on `estimates(project_id)` (not yet applied). See [perf note](#).
- `mv_projects_v1`: plain live view (despite `mv_` name), ~46ms (277 signed rows), stacks on `int_original_quotes`.

**Consolidation target:** this is the highest-value merge. One project-rollup definition (a properly-indexed live view computing each rollup once via LATERAL), consumed by both frontends + dbt. Resolve the v_active_projects vs mv_projects_v1 vs mega_view overlap.

---

## E. Payment / credit / receivables

| Calculation | Live home | Drift risk |
|---|---|---|
| credit_status, DPD, paid-on-time/late counts | frontend `PaymentSummary.tsx` (hardcoded 2025-01-01 cutoff) | ⚠️ frontend-only, brittle |
| Accounts receivable short/long term | dbt `int_official_accounts_receivable_by_project_gtq` | ✅ dbt |
| Remaining principal | dbt `int_remaining_principal_by_project_month_gtq` | ✅ dbt |

---

## F. Impact / analytics (dbt-only — leave in dbt)

`monthly_solar_savings`, total electricity production (30yr), CO₂ avoided, `production_start_date` (waterfall), `rural_area` (smod_code), `impoverished_area` (municipalities.in_poorest_20_percent). All analytics-only, no app-side equivalent. ✅ No action — correct as dbt.

---

## G. UI-only mappings (frontend shared const, NOT a view)

Workflow status → badge color / chip bucket / allowed transitions: duplicated across `ProjectsTable.tsx`, `ActiveProjectDetails.tsx`, `ProjectSearchBar.tsx`. → extract to one shared TS const.

---

## Consolidation priority (proposed)

1. **D — project rollups** (3-way overlap, the 8.8s view, biggest correctness+perf win)
2. **B — gross margin / IVA / retail** (3+ copies, real drift)
3. **A — collapse legacy/dup TS in the finance engine** (IRR/amortization)
4. **C — grid type seed view** (already scoped)
5. **E / G** — smaller cleanups

## Decisions to make with Ian
- For each cross-TS/SQL calc: live view vs persisted column? (Default: live view + indexes; persist only if proven too slow.)
- Project rollups: which of the 3 becomes canonical?
- Naming: rename `mv_*` → `v_*` (they're not materialized).
