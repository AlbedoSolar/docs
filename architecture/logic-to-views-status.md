# Business logic → canonical views: program status

*The returnable scoreboard for the long-running migration. Last measured: **2026-08-12**. Re-measure with the commands at the bottom — every number here is reproducible.*

## What the program is

Three rulings, one goal — each derived value has exactly one home, and that home is queryable:

1. **Calc architecture (2026-06-30):** operational/app-facing calcs live as plain Postgres views in `albedo-automations-infra/database/views/`; the app consumes them directly; dbt *consumes* them as sources (never re-implements). Quote *pricing* stays in the TS edge functions; frontend does display transforms only.
2. **Reads through section views (2026-08-04):** every app read goes through a `v_<section>` view; tables are for writes by id. Doc: `read-through-views.md`.
3. **mv_ retirement (2026-06-01):** user-facing reads never wait for a dbt rebuild — live views/tables only.

Related docs: `business-logic-map.md` (where the money math lives), `calculations-map.md` (per-domain drift inventory), `read-through-views.md` (the reads ruling).

## Scoreboard

| # | Front | State 2026-08-12 | Done when |
|---|---|---|---|
| 1 | Canonical views stood up | **24 deployed** (`v_project_calcs`, `v_estimate_calcs`, `v_cash_flows(_all_states)`, `v_active_projects`, `v_operations_projects`, `v_addendum_chain_health`, governance, FRD, …) | Each app section has its view |
| 2 | App reads through views | **CI-enforced** (`scripts/check-read-through-views.mjs`, runs in pr-checks). 204 table reads / 59 tables; **31 tables on PENDING_MIGRATION**; 67 reads already on views/exempt. 12 distinct `v_*` consumed | PENDING_MIGRATION list empty |
| 3 | mv_* retired from user paths | `mv_projects_v1` **dropped** 2026-06-10; `mv_active_projects_v1` off. **Two left:** `mv_monthly_cash_flows_v1` (2 reads, data service) + `mv_impact_project_summary_v1` (1 read, `pages/ImpactPage.tsx`) | Zero `mv_*` reads in frontend |
| 4 | dbt consumes canonical | **3 sources consumed** (`v_cash_flows`, `v_estimate_calcs`, `v_project_governed_dates`) + `v_operations_projects` parity test + `v_addendum_chain_health` invariant test. Repoint phases 0–2 done 2026-07-24 | Phase 3 done; no dbt model restates an app-facing rule |
| 5 | Guardrails | `estimates(project_id)` index **applied** ✅. `select('*')` in the data service: **29 remaining** ⚠️ | Zero `select('*')` on views |

## The four next moves, in order of value

1. **`mv_monthly_cash_flows_v1` → `v_cash_flows_all_states`** — now **unblocked**: the canonical view exists and carries `is_active` (the mv's `is_nullified`, inverted) since 2026-07-20, boundary-corrected 2026-08-11. Two frontend reads move; note `admin_income_project_currency` and the IVA columns need a home (add to the canonical view or compute in a display util). ⚠ `getActiveCashFlows` also feeds the Zoho sync — coordinate the shape change.
2. **dbt repoint Phase 3 — `int_original_quotes` split** (= addendum-robustness workstream 3): introduce `int_current_quotes` (chain head); origination attributes stay on original, live economics move to current. The Debita loan tape reads the *original* quote's interest rate today — wrong the day an adenda restates a rate. 39 model files reference the original-quote intermediates; classify each.
3. **Burn down PENDING_MIGRATION** — biggest sections first: `projects` (16 reads), `estimates` (14), `quotes` (10) all say "pending v_project_detail" — one section view retires three of the biggest entries. Delete lines from the allowlist as they land; the check fails on stale entries, so the list can't rot.
4. **Kill the 29 `select('*')`** — mechanical; each one defeats Postgres column-pruning on views (the `v_active_projects` 8.8s lesson).

Still deliberately out of scope: restrictive read RLS (deferred while the team is small), analytics-only dbt models (F-domain — stays in dbt by design).

## Re-measure (run from repo root)

```bash
# Front 2: reads through views + allowlist debt
cd albedo-automations-infra && node scripts/check-read-through-views.mjs

# Front 3: remaining mv_* reads — scope the WHOLE src/, not just services+features
# (the old command scanned only services/ and features/, so it missed the
#  ImpactPage read and reported "one left" when there were two)
grep -rn "from([\"'\`]\(app\.\)\?mv_" frontend/quotes-app/src

# Front 4: dbt canonical consumption
grep -rho "source('canonical', '[a-z_]*')" ../dbt/main/models/ | sort | uniq -c

# Front 5: select('*') count
grep -c "select('\*')" frontend/quotes-app/src/services/supabase-data-service.ts

# Front 1: deployed canonical views (needs psql creds)
# select viewname from pg_views where schemaname='public' and viewname like 'v\_%';
```

## History

- **2026-08-19** — Front 3 corrected: `mv_impact_project_summary_v1` is still read by `pages/ImpactPage.tsx` in both frontends. The 2026-08-12 measurement said "one left" because the re-measure command scanned only `src/services` and `src/features`; the command now covers all of `src/`. Fronts 4 and 5 re-measured unchanged (5 canonical sources, 29 `select('*')`).
- **2026-08-12** — doc created; measured all fronts. Since last touch: read-through CI shipped (was "not built"), `estimates(project_id)` index applied (was "pending"), `v_addendum_chain_health` + atomic supersession landed, seam boundary corrected.
- **2026-08-04** — reads-through-views ruling; deleted-estimates incident (12/30 read paths honored `deleted_at`).
- **2026-07-24** — dbt repoint phases 0–2 (signed rule, deletion/IVA fixes, canonical repoint, shadow-build CI).
- **2026-07-20** — `v_cash_flows(_all_states)` canonical liveness views.
- **2026-06-30** — calc architecture ruling.
- **2026-06-10** — `mv_projects_v1` dropped.
- **2026-06-01** — mv_ retirement direction set.
