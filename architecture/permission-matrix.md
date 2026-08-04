# Permission Matrix

Source of truth for RLS write policies across all public tables. SELECT access is granted to all authenticated users unless noted otherwise.

## Roles

| Role | Description |
|---|---|
| admin | Full access (role_id = 1) |
| sales | Sales team — can create and manage quotes, estimates, clients |
| operations | Operations team — can update/delete projects and sites |
| finances | Finance team — can update projects (for the diligence workflow; UI funnels edits to estimated_closing_date, diligence_notes, diligence_status) and non-signed quotes (added 2026-07-27 so DD can mark the winning quote as firmada). |

## Helper Functions

| Function | Purpose |
|---|---|
| `is_admin()` | Returns true if user has role_id = 1 |
| `user_has_role(role_name text)` | Returns true if user has the specified role |
| `is_sales_or_admin()` | Shorthand for `user_has_role('sales') OR is_admin()` |
| `estimate_is_signed(est_id bigint)` | Returns true if estimate has any quote with status `signed` or `signed_and_addended` |

## Business Tables

### Quote/Estimate Tables (signed-record protection)

| Table | INSERT | UPDATE | DELETE | Signed Protection |
|---|---|---|---|---|
| `quotes` | sales, admin | sales, finances, admin | sales, admin | Yes — quote with status `signed` or `signed_and_addended` is immutable |
| `estimates` | sales, admin | sales, admin | sales, admin | Yes — estimate with any signed quote is immutable |
| `estimate_equipment` | sales, admin | sales, admin | sales, admin | Yes — inherits from parent estimate |
| `estimate_affiliates` | sales, admin | sales, admin | sales, admin | Yes — inherits from parent estimate |
| `monthly_cash_flows` | sales, admin | sales, admin | sales, admin | Yes — inherits from parent quote |

### Project/Site/Client Tables (no signed protection)

| Table | INSERT | UPDATE | DELETE | Notes |
|---|---|---|---|---|
| `projects` | sales, admin | sales, ops, finances, admin | sales, ops, admin | Finance can UPDATE (intended for diligence columns via /finances/diligence). Operations can update/delete but not create. |
| `sites` | sales, admin | sales, ops, admin | sales, ops, admin | Operations can update/delete but not create |
| `clients` | sales, admin | sales, admin | sales, admin | — |

### Gobernanza Operativa (Solarbase Release 1A)

Source spec: `external-resources/Solarbase Release 1 Gobernanza Operativa.docx` Sec. 6. The DB tables are write-gated by RLS + a locking trigger; writes are funnelled through the `governance-milestones` edge function which adds role + motivo checks. Authenticated users SELECT freely (read side of the new GovernanceProjectsPage).

| Table | INSERT | UPDATE | DELETE | Notes |
|---|---|---|---|---|
| `project_milestones` | edge function only | edge function only | — | RLS read for all authenticated. INSERT/UPDATE only via service_role (no policy for authenticated). The `enforce_project_milestone_lock` trigger auto-sets `bloqueado=true` on first registration and rejects subsequent UPDATE unless `is_admin()` AND `app.motivo` ≥ 10 chars. DELETE not granted. |
| `governance_audit_log` | edge function only | — | — | Append-only. SELECT for admin / operations / operations-admin / finances. UPDATE and DELETE REVOKEd from every role at the grant layer (including service_role). |
| `milestone_catalog` | migrations only | migrations only | — | RLS read-only for authenticated. No write policy. |

| Edge function operation | Registrar (primera vez) | Modificar | Anular |
|---|---|---|---|
| operations, operations-admin, sales, finances | ✓ (subject to V1–V13 sequence rules) | — | — |
| admin | ✓ | ✓ (motivo ≥ 10 chars) | ✓ (motivo ≥ 10 chars) |

The `fecha_fee_administrativo` milestone is sourced from Finanzas (`projects.fee_administrativo_date`) — Operaciones see it read-only in the Timeline widget per spec Sec. 9.

## Other Tables

> Add rows here as RLS policies are created for additional tables.

| Table | INSERT | UPDATE | DELETE | Notes |
|---|---|---|---|---|
| `equipments` | sales, admin | sales, admin | sales, admin | SELECT public; writes gated by `is_sales_or_admin()` |
| `equipment_brands` | sales, admin | sales, admin | sales, admin | SELECT public; writes gated by `is_sales_or_admin()` |
| `providers` | sales, admin | sales, admin | — | SELECT public; writes gated by `is_sales_or_admin()`. DELETE intentionally not granted (downstream FKs from estimates / project_phases). **Nav (2026-08-04):** the Proveedores item is shown to sales, operations **and finances**; finances needs payment schedules and margins for payables. Read-only for finances — `ProvidersTable` gates create/edit on `isAdmin \|\| hasSalesRole`, so no write action is offered and RLS is unchanged. |
| `v_investor_portfolio_map` (view) | — | — | — | Read-only de-identified view (signed projects, ~2.7 km grid-snapped coords, no client fields). SELECT for all authenticated; REVOKEd from anon. Frontend `/investor-map` page and nav item are open to every authenticated user — the privacy boundary is the view itself, not a role. |
