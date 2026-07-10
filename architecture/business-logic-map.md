# Business-Logic Map & Duplication Inventory

*Generated 2026-07-10 from a full-codebase audit (frontend + edge functions + lambdas + scripts + SQL + dbt). This is the answer to "where does the money math live?" Keep it current: when you move or add a business calculation, update this file.*

## The doctrine (four homes for derived data)

| Layer | Owns | Examples |
|---|---|---|
| **Supabase edge functions** (`supabase/functions/`) | Quote/pricing calculation — anything that prices a deal | retail stages, commission, schedules, IRR |
| **Public SQL views** (`albedo-automations-infra/database/views/` + migrations) | App-facing eager rollups over live tables | `v_active_projects`, `v_project_calcs`, `v_estimate_calcs` |
| **dbt** (`dbt/main/`) | Analytics, snapshots, funder reports — consuming (not paralleling) the views | `mart_investors_*`, `mart_debita_*` |
| **Frontend utils** (`frontend/quotes-app/src/utils/`) | Per-row display transforms ONLY | IVA gross-up for display, formatting |

The frontend must not decide persisted numbers; the model to copy is `services/estimate-calculations-service.ts` (calls the edge function, persists what it returns).

## Canonical homes — review these files to review the business

### Pricing engine (the revenue formula)
- `supabase/functions/_shared/quote-helpers.ts` — retail stages (`calculateProjectRetailWithCommission`), commission percentage rule (`deriveCommissionPercentages` — commission type TOTAL, per 2026-07-10 ruling), commission schedule (`buildCommissionCostsSchedule`), legal fee tiers (`calculateLegalFee`), maintenance/insurance rate waterfalls (`getMaintenanceRate`/`getInsuranceRate`), estimate context loading, pricing receipt (`buildCalculationPayload`)
- `supabase/functions/_shared/quote-integration-flow.ts` — insurance/maintenance schedules, NII table (the canonical schedule builder)
- `supabase/functions/_shared/quote-goal-seek.ts` + `quote-forward-calc.ts` — solving
- `supabase/functions/calculate-interest/utils/financial-calculations.ts` — IRR/XIRR/amortization
- Entry points: `quote-solver` (offers), `quote-generator` (single quotes + manual mode), `irr-calculator` (variant matrices), `quote-rates`
- Written spec: `docs/quote-generation/` + decisions in `6-business-decisions-log.md`. ⚠ The legacy spec still sits in the dead `packages/shared/python-calculations/docs/` — port pending.

### Contract production
- `albedo-automations-infra/packages/lambdas/contract-generator/src/helpers.ts` — document rendering from stored offers (reproduces the offer; does not re-price). Reads canonical cell-root cash flows via `packages/shared/supabase-sdk/src/client.ts`.

### App rollups (SQL)
- `database/views/v_project_calcs.sql`, `v_estimate_calcs.sql` (canonical files) + `v_active_projects` (defined across migrations; latest full def 2026-06-29 + wraps). All variant-matrix reads are root-first (`COALESCE(cell->'cashflow_table', cell->'response'->…)`).

### Analytics
- `dbt/main/models/` — `int_projects_mega_view` → marts. ⚠ Known divergences from the app views (see inventory D/F below).

### Data ingestion (QB era) — RETIRED
- `packages/lambdas/quickbase-import-kickoff/` (signed deals), `scripts/qb-continuity-import.ts` (unsigned backfills) — retired with the QB pipeline (API access revoked); each carries its own copy of the financial engine, kept only as historical reference (see inventory item 6).

### Frontend sanctioned utils (display + labeled parity copies)
- `utils/retailDisplay.ts` (applyIva etc.), `utils/taxRate.ts`, `utils/offer-calculations.ts` (offer-sheet derivations + 30-yr projections), `utils/retail-preview.ts` (labeled pre-commission preview), `utils/maintenance.ts` (labeled engine mirror), `utils/totalContractValue.ts` (labeled contract mirror)

## Duplication inventory (2026-07-10 audit)

### DRIFT-HOT — surfaces can already show a human different money numbers
1. **total_contract_value — four disagreeing formulas.** *(RULED 2026-07-10: con & sin IVA flavors, maintenance always included — app views brought into compliance same day; residue: marts recomputing inline instead of consuming the view, and the contract doc's per-offer effective rate, see E.)* SQL views: con-IVA, *no maintenance*. dbt mega-view: sin-IVA, *with* maintenance. Contract generator: two internal versions (snapshot sin-IVA vs document TOTAL at per-offer effective rate). Frontend util mirrors the contract. dbt marts split between consuming the view and recomputing inline. → Needs one blessed definition (likely per-purpose names, single formulas).
2. **Two schedule builders inside the engine**: `quote-integration-flow` (estimate mode, canonical) vs `generateNiiTable` in `quote-calculator.ts` (manual mode) — already drifted: maintenance_premium default 0.1 vs 0.3 (in-code comment admits it). Manual-mode quotes price maintenance differently than estimate-mode today.
3. ~~**Grace/duration off-by-one**~~ *(FIXED 2026-07-10: mega view + data-chat views now count schedule rows only, matching the contract generator; corrected duration on 40 and grace on 190 of 277 signed projects, all deltas exactly one.)*
4. **Contado schedule built in the frontend** (`DefaultQuotesPageV2` ~:1546) and persisted to `variant_matrix` — the frontend deciding a client-facing payment plan with no engine involvement.
5. **Frontend payout commission** (tier multipliers ×1.5/1.25/1.0, IRR tier thresholds 12/14/16, 80/20 split) exists only in `DefaultQuotesPageV2` — in-code comment admits the double-compute. Payout policy should be server-owned.

### DRIFT-WARM — identical today, will diverge on next edit
6. ~~**The financial engine cloned 4×**~~ *(DOWNGRADED 2026-07-10: only the `calculate-interest` edge copy is live. The `supabase-client-handler` lambda copy is an orphan from the aborted January lambda experiment — added in 3486b50d, its callers reverted in dfcf8501 ("we're moving to supabase edge functions"), the util file left behind with zero importers since — delete it. The `quickbase-import-kickoff` and `qb-continuity-import.ts` copies are retired with the QB pipeline (API access revoked). No consolidation needed; just don't resurrect them.)*
7. **Engine context/args assembly cloned 3×**: quote-solver has clean helpers (`loadSharedContext`, `computeRetailAndSchedules`, `buildGoalSeekArgs`); quote-generator and irr-calculator inline ~90% copies. Live asymmetry: irr-calculator passes `partnerQuoteMarginType` where the others don't.
8. **Effective tax rate**: flat country rate (SQL/dbt) vs per-offer derived (contract generator) — differs on legacy QB offers.
9. **Frontend inline dupes**: cash-price/admin-fee con-IVA re-implemented in BOTH offer pages (bypass `offer-calculations.ts`); down-payment con-IVA inline (bypasses `retailDisplay`); ~11 inline `×(1+tax)` sites; two `convertCurrency` implementations with different failure semantics; `MonthlyCashFlow` vs `ActiveMonthlyCashFlow` near-duplicate components; PMT fallback formula in the wizard.

### Business logic in the wrong layer (single copy, but frontend-only)
10. 30-year savings projection / lifetime savings / CO2 impact (`offer-calculations.ts`) — client-facing headline numbers with **no server source of truth**; grid emission factors hardcoded (`taxRate.ts`).
11. Commission-split sum-to-1 invariant enforced only in a form (`QuoteConfigForm`) — no DB constraint.

### Meta
12. **CLAUDE.md is not checked into any repo** — the doctrine everyone cites lives only on Jake's machine at the workspace root. Commit it (or this file) somewhere versioned.
