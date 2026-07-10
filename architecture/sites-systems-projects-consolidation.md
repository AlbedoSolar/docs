# Sites · Systems · Projects — Consolidation Diagnosis

Diagnosis only. The Jun 18, 2026 working session with Fernanda surfaced
"duplication / confusion" across the `sites`, `systems`, and `projects`
layers. This document records the current state, the overlaps that need to
collapse, and the migration order for a future cleanup PR. **No code changes
yet** — the cleanup belongs to its own plan; this exists so that plan starts
from shared facts.

---

## 1. Intended 3-layer model

| Layer        | Owns                                                | Identifies                                         |
| ------------ | --------------------------------------------------- | -------------------------------------------------- |
| `sites`      | Physical location: address, country/dept/municipio, GPS coordinates, SMOD code | A site at a client (one client → many sites)        |
| `systems`    | Installed solar equipment: capacity, panels, inverters, brand, monitoring, Popular Power linkage (`is_in_pp`, `pp_id`) | A solar system at a site (one site → potentially many systems over time) |
| `projects`   | The financed deal: contract dates, financing, insurance, workflow status, client delivery, maintenance schedule, electricity provider | The commercial transaction backing a system at a site |

`project_systems` is the many-to-many junction (made M:N on
2026-02-26 — see `2026-02-26-create-project-systems-junction.sql`).
Production data is mostly 1:1 today; the M:N flexibility was added for future
multi-phase or staged installations.

Per memory `project_location_lives_on_site`, the convention going forward is
**location lives on `sites`**. Frontend reads should be `project.site.country`
etc. and writes should target `sites`.

---

## 2. Documented overlaps

### 2.1 Legacy location columns on `projects`

`projects.country_id`, `projects.department_id`, `projects.municipality_id`
predate the sites layer (sites was added per `project_qb_full_import_complete`
during the QB→Supabase import). They are still populated for backward
compatibility but the source of truth is `sites.country_id` /
`.department_id` / `.municipality_id` reached via `projects.site_id`.

**Surfaced callers** to audit before drop:
- Any DBT model selecting `projects.country_id`/`department_id`/`municipality_id`
- Any frontend service reading these columns directly (vs. through a site join)
- The `v_active_projects` view chain
- QB import path (likely still writes both for safety)

Open question: a few QB imports may still rely on the legacy columns when
the site is missing. Inventory before dropping.

### 2.2 `project_maintenances` anchored on `projects`, not on `systems`

`project_maintenances.project_id` references the project, not the system.
This was fine when projects had exactly one system; with M:N
`project_systems` it's now ambiguous which system a given maintenance event
belongs to.

In practice this hasn't caused a production bug yet because almost all
projects still have exactly one system. But the moment two systems share a
project, "this project's year-3 maintenance" no longer answers the question
"which system needs servicing." Either:

- Make `project_maintenances.system_id` the primary FK (and derive
  project_id at read time), or
- Add an optional `system_id` and treat the project-only case as the legacy
  default while letting newer rows specify the system

The first is cleaner; the second is cheaper. Defer the call to the cleanup
plan.

### 2.3 Capacity lives only on `systems`, but UI sometimes treats it as project-level

`systems.installed_capacity_kw` is the canonical capacity. The frontend
(quote calc, project list, DBT marts) currently reaches it via the join.
A few legacy spots compute or display "project capacity" without explicitly
joining; those should be normalized to read `sum(project_systems.system.installed_capacity_kw)`
through the M:N junction. Inventory during the cleanup pass.

### 2.4 Popular Power linkage is hidden inside `systems`

`systems.is_in_pp` + `systems.pp_id` (text) link a system to the Popular
Power monitoring platform. SystemForm exposes `pp_id` as a free-text input;
nothing labels it "Popular Power" in the UI. That isn't an overlap per se,
but two readers in two months have asked "what is pp_id?" — worth a label
fix during the cleanup.

---

## 3. Suggested migration order

Once the cleanup plan is written, execute in this order so each step is
independently safe:

1. **Audit surfaced callers** of `projects.country_id`/`department_id`/`municipality_id`. Anything reading these gets rewritten to `project.site.*` first. Write the script to grep the codebase + DBT models; commit the call-site changes.
2. **Backfill any project still missing a `site_id`**. Continuity script in `scripts/qb-continuity-import.ts` style. Verify via `SELECT count(*) FROM projects WHERE site_id IS NULL`.
3. **Drop the legacy location columns** on `projects` in one migration; refresh `schema-snapshot-YYYY-MM-DD.sql`; entry to `database/changelog/CHANGELOG.md` per CLAUDE.md.
4. **Decide and migrate `project_maintenances` system anchoring** (Section 2.2). This is the biggest call — depends on whether multi-system projects are a real near-term concern. If not, defer indefinitely and just document the assumption.
5. **Frontend label fix**: rename `pp_id` input to "ID Popular Power" (or similar) — purely cosmetic, no schema change.

---

## 4. Open questions for the cleanup plan

1. **Enforce `project_systems.is_primary` exactly-one-true?** — Today nothing
   stops two primaries per project. A partial unique index
   (`CREATE UNIQUE INDEX ... WHERE is_primary`) would enforce it cheaply.
   But: is there a real product reason `is_primary` exists, or is it a
   leftover from when it was 1:1? If it's only there for back-compat, the
   cleanest fix is to drop it entirely and let callers join by some other
   property (e.g. most-recent installation date).
2. **Should `sites` get a soft-delete column** (`deleted_at`)? It currently
   doesn't; the canonical pattern across the codebase is mixed. Decide as
   part of the cleanup, not now.
3. **DBT marts that join `projects` to the legacy columns** — full list to
   come from step 1 of the migration order. Some impact reports may need
   updating in lockstep.

---

## 5. Out of scope for this diagnosis

- Doing any of the above. This is a diagnosis; implementation is its own
  plan with its own review.
- Any change to `client_addresses` (a separate question — clients can have
  multiple sites, but billing/legal addresses are different from
  install-site addresses).
- Renaming or restructuring `qb_record_id` linkage; QB import remains the
  source of truth for historical projects per `project_qb_full_import_complete`.
