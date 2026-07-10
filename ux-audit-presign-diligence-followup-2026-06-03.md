# Overnight implementation report — sales nav consolidation + wizard exit

**Prepared:** 2026-06-03 (early morning)
**Companion to:** `docs/ux-audit-presign-diligence.md`
**Scope:** consolidate Mis Cotizaciones + Ventas into a single project-centric view, add server-side search across estimate/quote references, add "Guardar y salir" exits on wizard Steps 1–3.

## TL;DR

All planned overnight changes are in. Type-check (`tsc -b`) passes clean. Changes are localized to ~7 frontend files and 2 translation files. Nothing was merged or deployed — you're reviewing local code. The Mis Cotizaciones page file is kept around (at `/my-quotes-legacy`) as a safety hatch in case you find a regression after using the new flow for a day.

## What changed (file-by-file)

### Sidebar (1 file)
**`frontend/quotes-app/src/components/layouts/MainLayout.tsx`**
- Removed the duplicate "Ventas" sub-item that linked to `/sales`
- Renamed `/sales` sub-item to **"Ventas en proceso"** (was "Ventas")
- Renamed calculator sub-item to **"Nueva cotización"** (was "Calculadora de Cotización"), and pointed it at `/quote-wizard` instead of `/loan-kpi-calculator`
- `isActive` for the renamed sub-item now also captures `/my-quotes` so the redirect lands correctly highlighted

### Route (1 file)
**`frontend/quotes-app/src/routes/AppRouter.tsx`**
- `/my-quotes` now `<Navigate to="/sales" replace />` — preserves any bookmarks
- Original `MyQuotesPage` still reachable at `/my-quotes-legacy` for one-release fallback. If nothing surfaces an issue with the consolidated view in a few days, delete the route + the page file in a cleanup PR.

### ProjectsTable — the canonical "Ventas en proceso" view (1 file)
**`frontend/quotes-app/src/components/organisms/ProjectsTable.tsx`**
- Added **workflow_status badge column** (mode=active only). Color palette mirrors the existing quote-status badge convention (gray/amber/blue/teal/emerald/green/purple/red).
- Added **"Última actividad" column** using `project.updated_at`. Caveat below — this isn't quite "true" last activity yet.
- Added **chip strip filter** above the table with six buckets:
  - Todos
  - Borrador (`draft`, `estimate_requested`)
  - Con estimado (`estimate_created`)
  - Con cotización (`quote_generated`, `quote_sent`)
  - En diligencia (`due_diligence_in_process`, `due_diligence_completed`, `due_diligence_rejected`)
  - Listo para firmar (`due_diligence_approved`)
- Selected chip is URL-synced (`?chip=...`) so reps can bookmark or share their filtered view.
- **Server-side global search** kicks in when the search box has ≥2 chars in `mode="active"`. Hits the database directly across project_reference, project name, client legal_name, estimate_reference, and quote_reference. No row cap (the 200-row Mis Cotizaciones limit is gone). 300ms debounce.

### Data service (2 files)
**`frontend/quotes-app/src/services/data-service.interface.ts`** + **`supabase-data-service.ts`**
- New method `searchActiveProjectsByAnyReference(query)` that does 4 parallel `from(...).ilike()` queries (against `projects`, `clients`, `estimates`, `quotes`) and unions the matched project IDs. Then re-hydrates the full Project rows via `mv_active_projects_v1` for shape consistency with `getActiveProjects()`.

### Wizard — "Guardar y salir" on Steps 1–3 (3 files)
All three steps got the same pattern:
- New `exitAfterSubmitRef` ref + `useNavigate` import
- A "Guardar y salir" button next to the primary submit that sets the ref to `true` and calls `handleSubmit(onSubmit)()`
- The existing onSubmit handler, on success, checks the ref. If true: toast "Guardado" + navigate; else: call `onComplete` (the wizard's usual "advance to next step" path)

**Step 1 (Client)** — exits to `/clients/:client_reference`
**Step 2 (Project)** — exits to `/projects/active/:project_reference`
**Step 3 (Estimate)** — exits to `/projects/active/:project_reference` *(this is the one you explicitly called out)*

### Translations (2 files)
**`es.json` / `en.json`**
- `nav.salesInProgress`, `nav.newQuote`
- All 14 `activeProjects.workflowStatus.*` labels (Spanish + English)
- 6 `salesInProgress.chip.*` labels
- `table.columns.workflowStatus`, `table.columns.lastActivity`
- `quoteWizard.saveAndExit`, `quoteWizard.saved`

## Behavior changes a rep will see

1. Sidebar collapses from `Ventas > [Ventas, Mis Cotizaciones, Calculadora de Cotización]` to `Ventas > [Ventas en proceso, Nueva cotización]`.
2. "Ventas en proceso" page now has six chip filter buttons above the table.
3. Each row shows a colored badge for its workflow stage.
4. There's a "Última actividad" column with the last-updated date.
5. Typing 2+ chars in the search box now finds projects by quote ref (`COT-...`) or estimate ref (`EST-...`) too — and crucially, results aren't capped to "recent 200".
6. The wizard's Step 1, 2, and 3 each show a "Guardar y salir" button alongside the primary action. Clicking it saves the form, shows a toast, and navigates away (client detail / project detail / project detail).
7. Anyone with `/my-quotes` bookmarked or in muscle memory gets transparently redirected.

## Known limitations / caveats

These are intentional simplifications; flagging so you can decide what's worth iterating on.

- **"Última actividad" is just `project.updated_at`.** It doesn't yet reflect estimate/quote-level edits unless those bump the parent project's updated_at (they may or may not, depending on triggers). For "most recently touched," the right move is a computed `last_activity_at = GREATEST(project.updated_at, max(estimates.updated_at), max(quotes.updated_at))` — best done via the materialized view or a view function. Punted to keep tonight's scope honest.
- **No "matched in" hint on search results yet.** When the server-side search finds a project via a quote_reference match, the result row doesn't surface which quote ref matched. Easy follow-up: return the match metadata from `searchActiveProjectsByAnyReference` and render a small subtext under the project_reference cell. Skipped tonight to avoid wider changes to the Project type.
- **Server-side search makes 4 parallel queries on every typing pause.** It's debounced (300ms), so cheap in practice, but if usage grows you'll want to consolidate into a single Postgres function or a single query with `or()` clauses across joined tables. Tonight's approach prioritized clarity over raw efficiency.
- **The chip filter buckets are subjective.** I grouped `due_diligence_in_process | due_diligence_completed | due_diligence_rejected` under "En diligencia" together because they're all "currently in due-diligence land." If `due_diligence_rejected` should be its own bucket (or surface in red somewhere), tell me and I'll add it.
- **The new estimate-on-existing-project flow** (i.e., re-entering the wizard with `?projectRef=X` but no `estimateId`) was *not* explicitly tested tonight. The wizard's URL-param restoration logic at `QuoteWizard.tsx:126–180` looks like it'd handle this correctly — it restores selectedProject from `projectRef` and leaves selectedEstimate null, which should cause Step 3 to start fresh. But verify in browser before relying on this for real reps.
- **`MyQuotesPage.tsx` is not deleted.** It's reachable at `/my-quotes-legacy` and the sidebar doesn't link to it. Delete in a follow-up after a few days of safety margin.
- **No CHANGELOG entries** for these changes — they're frontend-only with no DB schema work, so per the repo conventions there's nothing to log there.

## Risk read

**Low to medium.** The only architecturally non-trivial change is the new data-service method, and it falls back gracefully (empty search results → table shows the empty state, no crash). The chip strip, badge column, and last-activity column add new UI without changing any existing data flow. The wizard "Guardar y salir" buttons reuse the same submit handlers as the primary buttons — the only difference is the post-success branch.

The biggest test-it-in-browser items are:
1. Server-side search returning the right projects across all 4 query paths (especially the embedded-client one).
2. "Guardar y salir" on Step 3 when *editing* an existing estimate (the edit branch, not just create).
3. The /my-quotes redirect actually landing on /sales for users with the bookmark.

## Suggested next moves (when you wake up)

1. Spin up `npm run dev`, click through:
   - Sidebar nav (3 sub-items, no Mis Cotizaciones)
   - Ventas en proceso page (chip filter, badge column, last activity)
   - Search for a quote_reference you know exists — should find the project
   - Wizard Step 3, Guardar y salir → project detail
2. If it feels right, smoke-merge to staging.
3. Open follow-up issues from the caveats above (last-activity rollup, matched-in hint, chip bucket review).
4. Triage the audit's project-detail recommendations (Diligence panel, Continuar configuración CTA) — that's the natural sequel to tonight's work and the place where the multi-estimate / multi-quote-per-estimate hierarchy needs to live.

## Files touched

```
frontend/quotes-app/src/components/layouts/MainLayout.tsx
frontend/quotes-app/src/routes/AppRouter.tsx
frontend/quotes-app/src/components/organisms/ProjectsTable.tsx
frontend/quotes-app/src/components/organisms/QuoteWizardStep1Client.tsx
frontend/quotes-app/src/components/organisms/QuoteWizardStep2Project.tsx
frontend/quotes-app/src/components/organisms/QuoteWizardStep3Estimate.tsx
frontend/quotes-app/src/services/data-service.interface.ts
frontend/quotes-app/src/services/supabase-data-service.ts
frontend/quotes-app/src/utils/language/es.json
frontend/quotes-app/src/utils/language/en.json
```

10 files. `tsc -b` clean.
