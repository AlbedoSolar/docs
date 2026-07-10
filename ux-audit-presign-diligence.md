# UX Audit — Pre-sign diligence + rep navigation

**Prepared:** 2026-06-02 (overnight)
**Scope:** the lifecycle stage between "client agrees to a quote" and "contract is signed," plus how sales reps find and edit their prior work.
**Method:** code audit (no runtime). Three parallel Explore agents + manual reconciliation. All file:line references current as of staging at time of audit.

---

## TL;DR

The codebase already has a **due diligence concept baked in** — `projects.workflow_status` has four `due_diligence_*` states and a thunk comment explicitly defines "active projects" as "pre-signature: draft through quote_sent." None of this is surfaced in the UI. Reps see no diligence checklist, no "is this project ready to sign?" indicator, no way to filter drafts, and no breadcrumb from a quote back to its project. The DB enum and the frontend type for `workflow_status` are also **out of sync** (3 values diverge), which is independently worth fixing.

The single hard gate before signing is `g_drive_folder_id` being non-null on both client and project — and it fails silently with a generic error after the user clicks Generate Contract. Everything else (legal rep, NIT, insurance, technician, grid access, installation dates) is optional, NULL by default, and never validated.

The most leveraged change is **a single "Diligence" panel on the project detail page** that surfaces the checklist, blocks contract generation pre-flight (instead of post-click), and gives reps a continue-where-you-left-off CTA. The smaller wins (workflow_status filter on the projects table, quote→project link from the calculator, sync DB/frontend enums) are independently shippable.

---

## 1. The hidden architecture

The diligence stage exists in the data model. It's just not visible.

**`projects.workflow_status` enum** (schema-snapshot-2026-05-26.sql:109–123) — 14 values:

```
draft → estimate_requested → estimate_created → quote_generated → quote_sent
  ↓
due_diligence_in_process → due_diligence_approved | due_diligence_rejected
  ↓
signed → addended → active
  ↓
inactive | rejected_by_client
```

**`quotes.status`** (quotes.interface.ts:5–11) — 6 values:
- `pending`, `approved`, `signed`, `rejected`, `signed_and_addended`, `cancelled`

**`projectsAsyncThunk.ts:395`** has the comment:
> "Async thunk for fetching active projects (pre-signature: draft through quote_sent)"

So "Active Projects" in the codebase already means "everything that isn't signed yet." That's the diligence pipeline. But the page that consumes this (`ActiveProjectsPage.tsx`) doesn't render that distinction — the rep just sees a flat list with no workflow_status badge or filter.

### Enum drift between DB and frontend

Source of truth divergence (will bite us eventually):

| In DB enum | In frontend type | Missing on the other side |
|---|---|---|
| `inactive` | — | DB-only |
| `addended` | — | DB-only |
| `rejected_by_client` | — | DB-only |
| — | `due_diligence_completed` | Frontend-only |
| — | `signed_and_addended` | Frontend-only |
| — | `cancelled` | Frontend-only |

Pick one source of truth. Easiest: regenerate frontend types from DB (Supabase `generate_typescript_types`), then reconcile naming.

---

## 2. The 13 diligence fields — where they live, what's broken

| # | Field | Stored at | Editable in UI | Blocks contract gen? | Default |
|---|---|---|---|---|---|
| 1 | Legal rep (name, ID, role…) | `client_legal_representatives.*` | ClientForm.tsx:318–400, QuoteWizardStep1Client.tsx | No | Silent NULL |
| 2 | NIT / tax_number | `clients.tax_number` | ClientForm.tsx:235–239 | No | NULL |
| 3 | Billing address | `clients.billing_address` | ClientForm.tsx:241–245 | No | NULL |
| 4 | Client Drive folder | `clients.g_drive_folder_id` | ClientForm.tsx:267–271 | No | NULL |
| 5 | **Project Drive folder** | `projects.g_drive_folder_id` | **No UI** — admin-only | **YES — hard block** | NULL → generic post-click error |
| 6 | Insurance (insurer, policy#, cert#, status) | `projects.insurance_*` | EditOperationsProjectForm.tsx:71–73 | No | NULL |
| 7 | Grid access + electricity provider | `projects.electric_grid_access`, `electricity_provider` | EditOperationsProjectForm.tsx:75–76 | No | NULL |
| 8 | Contract installation dates | `projects.contract_installation_*` | EditOperationsProjectForm.tsx:108–109, 238–244 | No | NULL |
| 9 | **Solar technician** | `projects.maintenance_provider_id` | **No UI** — admin-only | No | NULL |
| 10 | Maintenance schedule rows | `project_maintenances.*` | MaintenancesPage.tsx (edit only, not create) | No | Auto-seeded by 2026-05-27 migration |
| 11 | Provider payment schedule | `provider_payment_schedules.*` | ProjectPhasesManager (auto-selected, not editable) | No | Silently inherits provider default |
| 12 | Equipment line items | `estimate_equipment.*` | EstimateEquipmentManager.tsx:18–120 | No | No defaults |
| 13 | Phases + phase costs | `project_phases.*` | ProjectPhasesManager in ActiveProjectDetails | No | No defaults |

**Pattern of the gaps:**
- **Two fields have no UI at all** (#5 project Drive folder, #9 technician). #5 is also the only hard gate — so the rep gets a contract-gen error for a field they have no UI to set. That's the worst combination in the table.
- **The one hard gate fails silently and post-click.** Generic error string ("Project has no g_drive_folder_id set") via contractGenerationMessages.ts, only surfaced after the rep clicks Generate Contract.
- **Six fields are scattered across four forms** (ClientForm, QuoteWizardStep1Client, EditOperationsProjectForm, MaintenancesPage, ActiveProjectDetails). No single "is this project ready?" view.
- **Provider payment schedule auto-selects silently** — rep never sees what schedule was chosen. Bites later when payments don't match expectations.

---

## 3. Status transitions — what's automatic, what's manual

**Nothing is automatic.** Every state transition is a human typing into a form:

- `quote.status` → only changes via `updateQuote` thunk (quotesAsyncThunk.ts:67–78). No cascade from elsewhere.
- `project.workflow_status` → only changes via the dropdown at `EditOperationsProjectForm.tsx:212–217`. Free-form picker across all 14 values, **zero transition validation** (a rep can jump from `draft` directly to `signed` without ever passing through diligence).
- Contract signing → sets `contracts.signed_on`, but does NOT cascade to `quote.status='signed'` or `project.workflow_status='signed'`. Both must be manually flipped after.

This is consistent with Albedo's "logic in code, not DB" convention — but in the *frontend* code, the cascade is also missing. Sign-time should at minimum bump quote → signed and project → signed in the same operation.

---

## 4. Rep navigation — the five journeys

Mapped these against the actual route table (MainLayout.tsx:258–289) and concrete table/filter code.

### Journey 1 — "Re-run last week's quote with different assumptions"
**Path:** Mis Cotizaciones → search by ref/client/estimate → click row → /quote-wizard/4 pre-loaded.
**Friction:** Works well. Five-field search (MyQuotesPage.tsx:85–89). Only gap: no date range filter — "last Thursday" requires manual scan.

### Journey 2 — "Edit client phone from a few days ago"
**Path:** Clientes → search → /clients/:client_reference → edit form.
**Friction:** Works. Client list doesn't show associated projects/quotes count, so reps can't visually identify "my busy client."

### Journey 3 — "Find a draft project I started and continue Step 2"
**Path:** Ventas → /projects/active (ProjectsTable, mode="active") → filter by sale_type/date/country/cost/status/currency.
**Friction:** **Critical.**
- `workflow_status` is in the data but **not surfaced as a filter** (ProjectsTable.tsx:59–77). Rep cannot say "show me only my drafts."
- Clicking a draft navigates to ActiveProjectDetails — there is **no "Continue Setup" CTA** linking back to `/quote-wizard/2?projectRef=...`. The wizard *would* resume cleanly (QuoteWizard.tsx:126–180 already restores from URL params), but the button to get there doesn't exist.
- Net: drafts feel like dead ends.

### Journey 4 — "Quote was approved, navigate to the project page to mark progress"
**Path:** From QuoteWizardStep4Explorer → browser back, or manually retype URL.
**Friction:** No "View Project" link from the calculator. The calculator shows no project status / contract status, so a rep working on a signed quote can't see they need to update the project status from inside the calculator.

### Journey 5 — "Show me everything I'm working on right now"
**Path:** None. There is no unified WIP view.
**Friction:** Reps must check `/my-quotes` (filter status=pending), `/projects/active` (no draft filter), and infer estimate state from project drill-downs. Three context switches for one mental question.

---

## 5. Recommended UX changes (ordered by leverage)

### A — Single panel: "Diligencia" on the project detail page
The highest-impact single change. New collapsed panel on ActiveProjectDetails listing all 13 diligence fields with:
- Per-field status: ✅ Complete / ⚠ Missing / 🟦 Optional
- Inline "Edit" link to the right form (no need to navigate to ClientForm + EditOperationsProjectForm + MaintenancesPage separately)
- Completion progress bar at top ("8/13 listo para firmar")
- **Pre-flight check on Generate Contract:** if Project Drive folder is missing, disable the button and show the missing-field list inline (don't wait for the post-click failure)

This single panel solves: the silent-NULL problem, the scattering problem, the "ready to sign?" question, and the "no UI for technician / Drive folder" gap (just add inputs in this panel — no need for a separate admin tool).

### B — Workflow status badge + filter on `/projects/active`
ProjectsTable already has the data. Add:
- A status badge column (color-coded per workflow_status, mirroring MyQuotesPage's quote-status badges)
- A filter chip strip at the top: "Borradores | En diligencia | Listos para firmar | Firmados | Todos"
- Default filter to "En proceso" (everything pre-signed)

### C — "Continue Setup" CTA on ActiveProjectDetails for non-completed projects
If `workflow_status` ∈ {draft, estimate_requested, estimate_created, quote_generated, quote_sent}, show a prominent button "Continuar configuración" linking to the right wizard step:
- draft → Step 1
- estimate_requested / estimate_created → Step 3
- quote_generated / quote_sent → Step 4

### D — Bidirectional links between quote calculator and project page
From QuoteWizardStep4Explorer: small "Ver proyecto →" link in the header. From ActiveProjectDetails: existing quote list rows already navigate well; just verify they do.

### E — Reconcile DB enum vs frontend type for `workflow_status`
Pick a side (recommend DB), regenerate frontend types, fix all consumers. This is an independent task and a footgun.

### F — Auto-cascade quote signing
When `contracts.signed_on` is set, in the same transaction: set `quotes.status='signed'` and `projects.workflow_status='signed'`. Either as a Supabase trigger (this is one of the exception cases — denormalized-field-kept-in-sync) or in the contract-generator Lambda. Document in `database/triggers/` per the CLAUDE.md rule.

### G — Date-range filter on MyQuotesPage and ProjectsTable
Low cost, addresses "last Thursday" friction.

### H — "Mi trabajo" sidebar item (optional, broader)
Unified WIP dashboard: drafts + pending quotes + estimates-in-progress, sorted by last_modified. Bigger ask — defer if A–G are enough.

---

## 6. Draft GitHub issues (ready to file)

Suggesting these for GitHub Project #2 under a new milestone or label ("UX — Diligencia"):

1. **Diligence panel on project detail** — Add a "Diligencia" panel on ActiveProjectDetails listing all 13 pre-sign fields with status badges, inline edit, and progress bar. Pre-flight check before Generate Contract. (Owner: Jake, est: 1–2 days)
2. **Add Project Drive folder + Solar technician UI** — Two fields currently have no UI. Add inputs to the Diligence panel from #1. (Owner: Jake, est: 2 hr)
3. **Workflow status filter on /projects/active** — Add status badge column + filter chip strip. Default to "En proceso". (Owner: Jake, est: 3 hr)
4. **"Continuar configuración" CTA** — On ActiveProjectDetails for non-completed projects, link to right wizard step based on workflow_status. (Owner: Jake, est: 1 hr)
5. **Quote ↔ project bidirectional links** — "Ver proyecto" link in QuoteWizardStep4Explorer header. (Owner: Jake, est: 30 min)
6. **Reconcile workflow_status enum (DB vs frontend)** — Regenerate types from DB, fix consumers, document decisions. (Owner: Jake or Ian, est: 2–3 hr)
7. **Auto-cascade quote signing → quote/project status** — On contracts.signed_on set, cascade `quotes.status` and `projects.workflow_status`. Trigger or Lambda. (Owner: Daniel? Jake? est: 2 hr)
8. **Date range filter on MyQuotesPage and ProjectsTable** — Low-priority polish. (Owner: Jake, est: 2 hr)
9. **Workflow status transition validation** — Replace free dropdown at EditOperationsProjectForm.tsx:212 with a list of valid next states given current state. (Owner: Jake, est: 2 hr; optional, post-cutover)

---

## 7. Caveats

- This is a code audit, not a runtime audit. No browser testing happened overnight. Friction observations are derived from reading components, route tables, and feature flags — they don't account for how the UI *feels* in practice. A 30-min Playwright pass before shipping any of these would be worth it.
- Spanish/English mismatches in field labels weren't audited — most of the codebase is bilingual (`es.json` primary, `en.json` fallback) and the proposed Spanish strings above are my draft, not pulled from an existing translation.
- Permissions / RLS not in scope here — none of the proposed changes alter who can edit what. The diligence panel just consolidates existing capabilities.
- I did not check whether the contract-generator Lambda *also* needs Project Drive folder (vs only the client folder). The error message implies it does. Worth a quick verification before A ships.

---

## 8. What I'd ship first

If pre-cutover priorities only allow one item: **A (Diligence panel).** It solves the silent failure, makes the diligence stage visible for the first time, and is one self-contained component on one page. Everything else is independently valuable but optional for D-day.

If two items: A + F (auto-cascade signing). F removes manual cleanup steps every time a project signs, which compounds across however many projects sign in the first weeks after cutover.

If three: A + F + B (workflow_status filter). B finally makes drafts findable.
