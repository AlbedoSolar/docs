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

## Duplication inventory v2 (2026-07-10 refresh — LIVE code only)

*Re-audited 2026-07-10 by three parallel scans (edge+lambdas, frontend, SQL/dbt) with all dead/retired surfaces excluded (python package, QB importers, the orphaned lambda financial-calculations copy). Supersedes the v1 inventory below the "Resolved" section.*

### DRIFT-HOT — two live paths can already show different money for the same input

1. ~~**irr-calculator resolves engine params differently than quote-solver/quote-generator.**~~ *(FIXED 2026-07-10: new `_shared/quote-params.ts` is the single param-resolution home for all three engines — resolveWacc (ruled 9% everywhere; config rows + form fallback updated too), resolveDownPayment (three-tier), resolveMaint\*/resolveRiskFreeRate, getWithFallback hoisted; dead partnerQuoteMarginType arg deleted. Deployed `2276f44`; smoke test: dial and engine now return identical IRR/payment. Historic exposure was minimal — the app always sent wacc explicitly and no estimate carries a stored down-payment %.)* Original finding:
   - WACC default **0.09** (`irr-calculator/index.ts:112`) vs **0.10** (`quote-solver/index.ts:275`, `quote-generator/index.ts:325`, `DEFAULT_CONFIG.default_wacc` in `quote-helpers.ts:59`); irr-calculator also ignores the `albedo_wacc` alias the other two honor. WACC drives the min-APR floor → payment → IRR.
   - Down payment: irr-calculator defaults to **0%** with no fallback (`irr-calculator/index.ts:110`); solver/generator use the three-tier chain ending in `estimates.down_payment_1_percentage` (`quote-solver/index.ts:367`, `quote-generator/index.ts:240`).
   - `maint_premium`/`maint_inflation` short aliases honored by solver/generator, ignored by irr-calculator (`irr-calculator/index.ts:117`).
   → Fix: one shared param resolver in `_shared/` used by all three. **Verified non-issue:** the long-suspected `partnerQuoteMarginType` asymmetry is a dead argument — `calculateRetailProjectionFromEstimateContext` doesn't accept it; margin type flows from `context.phase_pricing_inputs` identically in all three. Delete the misleading arg at `irr-calculator/index.ts:296`.

2. **"Precio al Contado" computed three ways that disagree.** `ActiveProjectDetails.tsx:1286` (via `computeTotalContractValueConIva`, includes closing costs) vs `ActiveProjectDetails.tsx:1904` (plain `retail×(1+tax)`, same label, same component) vs `ClientOfferPage.tsx:586` (`Σ income_total`). For contado deals with admin/asset closing costs these are three different client-facing numbers. → One `resolveContadoCashPrice` in utils, used everywhere.

3. ~~**Grace/duration off-by-one fix NOT ported to `v_project_calcs`.**~~ *(FIXED 2026-07-10: migration `2026-07-10-v-project-calcs-grace-duration-schedule-rows-only.sql` — both branches (mcf AND variant-matrix JSON, whose month-0 row had the same bug). Deltas on 3,098 projects: duration −1 on 41, grace −1 on 196, zero unexpected; backup in `backups.v_project_calcs_duration_grace_2026_07_10`.)*

4. ~~**Repo `v_project_calcs.sql` lags prod.**~~ *(FIXED 2026-07-10: root-first hand-ported to all 9 read sites (comments preserved) + the item-3 expressions; verified byte-identical to prod via scratch-schema `pg_get_viewdef` md5 comparison.)*

5. ~~**data_chat `v_projects_gtq/usd` TCV omits maintenance**~~ *(FIXED 2026-07-10: `v_monthly_cash_flows_*` now emit `maintenance_payment_*`; `v_projects_*` add the rollup and include it in TCV; deployed via dbt run, zero values changed. Residue: the reconciliation marts' per-row EEFF sums still omit maintenance — left alone deliberately, they reconcile against accounting entries and aren't a TCV; revisit if they ever feed contract-value totals.)*

6. **Manual mode is still a second engine** (`generateNiiTable` + `getKpis` in `quote-calculator.ts`, used only by quote-generator's `runManualMode`): retail gross-up differs on Subtract-margin providers (`quote-calculator.ts:236` vs `quote-helpers.ts:568`), different amortization form, different insurance-start logic. The maintenance_premium 0.1/0.3 split is FIXED (both now import `DEFAULT_MAINTENANCE_PREMIUM`), but the engines remain independent. At minimum: document that manual mode ≠ estimate mode.

7. **Client-facing schedules/policy still frontend-only** (no server source): contado 50/50 schedule (`DefaultQuotesPageV2.tsx:1529`), PMT annuity seed (`:747`), payout commission tier multipliers 1.5/1.25/1.0 (`:159`, applied `:1727`), 30-yr savings/CO2 (`offer-calculations.ts`), commission-split sum-to-1 only in a form.

### DRIFT-WARM — identical or latent today, will diverge on next edit

8. **Two `convertCurrency` in utils/**: `retail-preview.ts:32` (canonical, USD-pivot, returns null) vs `currency.ts:35` (throws on cross-currency, zero live callers). Delete the currency.ts copy before someone imports it. Also two formatters (`currency.ts:85` vs `helpers.ts:1`).
9. **Inline IVA gross-ups: 19 business sites** bypass `applyIva` (list in audit; worst: `DefaultQuotesPageV2` ×9, `ActiveProjectDetails` ×4, both offer pages' admin fee). Sibling gap: no `stripIva` helper, so ÷(1+tax) is always inline. `computeDownPaymentAmountConIva` (`retailDisplay.ts:25`) is ORPHANED while its exact formula is inlined 3× in DefaultQuotesPageV2. Tax-rate lookup re-inlined at 7 sites despite `getTaxRateFromCountries`.
10. **Engine-internal clones**: frontend-shape mappers redefined verbatim in `quote-generator/index.ts:92,102` (canonical: `quote-forward-calc.ts:75,87`); `getWithFallback` copied in solver+generator; IRR bisection stack (`npv`/`irrBisection`/`annualizedIrrPercent`) exists in both `quote-integration-flow.ts:330` and `quote-calculator.ts:414`; `round2` ×3. → hoist to `_shared/quote-utils.ts`.
11. **MonthlyCashFlow vs ActiveMonthlyCashFlow**: table render duplicated verbatim; real differences are only data source + CSV shape. → extract presentational `CashFlowTableView`, keep two thin wrappers.
12. **Cross-layer naming hazards**: dbt mega-view `total_contract_value_project_currency` is sin-IVA but unlabeled (app views say `_sin_iva`/`_con_iva`); estimate retail vs approved (post-commission) retail share the name "retail"; ~~installed capacity derived two ways~~ *(FIXED 2026-07-10, ruling: equipment sum is the ONLY source — new `v_project_installed_kw` + `int_estimate_installed_kw`, every consumer rewired, static columns renamed `*_legacy`; see decisions log)*; `0.12` tax fallback literal duplicated. Doc rot: `quote-rates/index.ts:26` JSDoc claims premiums the code doesn't use; `CALCULATIONS.md` still cites deprecated `mv_monthly_cash_flows_v1`.

### Meta
13. **CLAUDE.md is not checked into any repo** — the doctrine everyone cites lives only on Jake's machine at the workspace root. Commit it (or this file) somewhere versioned.

### Resolved since v1 (verified by the re-audit)
- ~~maintenance_premium 0.1 vs 0.3~~ — both builders import `DEFAULT_MAINTENANCE_PREMIUM` (0.3); only stale comments remain.
- ~~Effective tax rate two methods~~ — single `deriveEffectiveTaxRate` (`contract-generator/src/helpers.ts:519`) feeds both contract builders.
- ~~calculation_payload duplication~~ — one `buildCalculationPayload` (`quote-helpers.ts:1153`) called by solver + generator.
- ~~Constants copied across the three index files~~ — centralized in `_shared/quote-constants.ts`.
- ~~partnerQuoteMarginType asymmetry~~ — dead argument, no behavioral difference (see item 1).
- ~~Grace/duration off-by-one in dbt~~ — fixed 2026-07-10 (but see item 3: the app view still lags).
- ~~The financial engine cloned 4×~~ — only the `calculate-interest` edge copy is live; lambda copy is a dead orphan (delete), QB import copies retired.
- ~~convertCurrency inline in DefaultQuotesPageV2~~ — now imports from `retail-preview.ts` (but see item 8: a second utils copy exists).
- ~~TCV inline in ActiveProjectDetails / QuoteOfferMatrix~~ — both use `computeTotalContractValueConIva` (but see item 2: the contado block at :1904 bypasses it).
- ~~Down-payment con-IVA on offer sheets~~ — both share `resolveDownPaymentConIva` (`offer-calculations.ts:99`).
- dbt marts recomputing the mega view: only `mart_investors_summary_by_project_*` does, intentionally (cutoff reproducibility, documented `CALCULATIONS.md:352`). Keep.
