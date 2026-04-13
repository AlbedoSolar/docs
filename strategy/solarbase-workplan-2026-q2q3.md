# Solarbase Development Work Plan — Q2-Q3 2026

## Overview

Solarbase is Albedo's core platform for solar lease origination, portfolio management, and reporting. This work plan covers four parallel tracks needed to consolidate operations onto Solarbase and retire legacy systems.

---

## Track 1: Quoting Tool — Complete & Launch to Sales Team

**Goal:** Make Solarbase the primary tool for creating and managing quotes, replacing QuickBase for the sales team.

**Current state:** The quoting wizard (4-step flow) is functional for internal use. Default quote generation (Gold/Silver/Bronze tiers), client offer pages, and the KPI calculator exist. UX audit fixes are complete. Not yet launched to the full sales team.

### Remaining work

| Item | Description | Priority |
|---|---|---|
| **Default quotes UX** | Finalize the Gold/Silver/Bronze card layout with commission scores, monthly payment, and "Send Offer" flow. Backend calls to quote-generator already work. | High |
| **Launch to core sales** | Configure field-level permissions (PolicyFormField system is built), run QA with sales team, training. | High |
| **Client offer refinements** | Add monthly interest breakdown and total monthly payment (con IVA) to the client-facing offer page (BW-2, BW-3). | Medium |
| **PDF quote generation** | Generate downloadable quote PDFs. Draft view exists, needs a sustainable rendering approach. | Medium |
| **Commission calculations** | Implement commission logic with margin types (add/subtract), IVA handling, multiple sales people. | Medium |
| **Addendum workflow** | Simplify addendum creation to a single atomic operation instead of 3+ manual steps. Fix the one-signed-quote trigger. | Medium |
| **Equipment configuration** | Replicate QB equipment setup. Long-term: pull from OpenSolar. | Low |
| **Wider sales launch** | After core sales is validated. Additional training, field policies for expanded roles. | Low |

---

## Track 2: Operations — Move Day-to-Day Workflows to Solarbase

**Goal:** Operations team manages active projects, installations, and post-install workflows in Solarbase instead of QuickBase + Asana.

**Current state:** Operations view exists (OperationsProjectsPage) with project table, filtering, and basic project details. Signed project import from QB works (with known provider mapping issues). The view is functional for read-only monitoring but not yet a full workflow tool.

### Remaining work

| Item | Description | Priority |
|---|---|---|
| **Project status tracking** | Add installation milestones, inspection dates, grid connection status. Currently only workflow_status (draft/signed/addended/cancelled). | High |
| **Zoho payment status** | Surface actual payment status (paid to date, current balance, DPD) on each project's detail page. Data exists in zoho_invoices — needs a UI component. | High |
| **Retail price in USD** | Show insured value on operations table (OPS-2). Needs exchange rate decision. | Medium |
| **KPI view on project page** | Read-only summary of IRR, payback, NPV on ActiveProjectDetails. Calculator exists, just needs a display component. | Medium |
| **Process flows** | Define and implement the operational workflow states (pre-install → install → post-install → monitoring) with assigned tasks and notifications. | Medium |
| **CO2 avoidance tracking** | Yearly CO2 avoidance calculation per project (BW-7). Solar production × grid emission factor. | Low |
| **Equipment depreciation** | Add depreciation column to equipment list view (BW-4). Data exists in DB. | Low |

---

## Track 3: Zoho Books Integration — Bidirectional Sync

**Goal:** Reliable, automated data flow between Solarbase and Zoho Books for invoicing, payments, and portfolio reporting.

**Current state:** One-way sync exists (Supabase → Zoho: contacts, projects, cash flows). Invoice and payment data flows Zoho → Supabase via CSV exports (manual). The Debita loan tape reports now work for Guatemala + Honduras using this data. No automated reverse sync yet.

### Remaining work

| Item | Description | Priority |
|---|---|---|
| **Automated invoice sync** | Build a Lambda/edge function that pulls invoices from Zoho Books API on a schedule (daily). Replace the manual CSV export process. Endpoints are identified. | High |
| **Automated payment sync** | Same for customer payments. Link payments to invoices and projects automatically using the Zoho Project ID chain. | High |
| **Invoice-to-project mapping** | Use the Zoho Project ID → project_code direct chain as the primary mapping. Fall back to heuristics only when Project ID is missing. Fix the 120 currently unmapped GT invoices. | High |
| **Honduras accounting sync** | Formalize the HN accounting CSV import process. Currently manual via `honduras_fifo_loader.py`. Long-term: connect to HN's accounting system directly or move HN to Zoho. | Medium |
| **Loan tape automation** | Currently dbt views + manual export. Goal: one-click report generation from the platform, or scheduled delivery. | Medium |
| **El Salvador data** | SV has ~4 projects. Need the facturas-y-pagos export from the SV accounting team (not the financial statements they sent). Small scope. | Low |
| **Zoho project linking** | Work with the Zoho/finance team to ensure ALL invoices in Zoho have a Project ID. The ~26% of 2025 invoices without Project IDs forced heuristic matching that caused misattributed invoices. | Low |

---

## Track 4: Sunset QuickBase

**Goal:** Decommission QuickBase as a primary data source. Solarbase becomes the system of record.

**Current state:** QuickBase is still used for initial project/client data entry by some teams. The QB import Lambda (`quickbase-import-kickoff`) pulls signed projects into Supabase. Provider data sync has known bugs (wrong field ID for Record ID#2). Some teams still reference QB for day-to-day work.

### Remaining work

| Item | Description | Priority |
|---|---|---|
| **Identify remaining QB-only workflows** | Audit which teams still use QB daily and for what. Map each workflow to its Solarbase equivalent (or build it). | High |
| **Fix QB provider sync** | The `ProveedoresTableFields.RecordId = 2` SDK enum bug causes duplicate providers on every import. Fix to `9` (Record ID#2). Also fix the provider resolver that didn't fire for the Quelectriza case. | Medium |
| **Data migration plan** | Define which historical QB data needs to be in Supabase and which can be archived. Currently only signed projects are imported. | Medium |
| **Parallel run period** | Run both systems in parallel with data validation checks. Define exit criteria for turning off QB (e.g., zero logins for 30 days, all active projects in Solarbase). | Medium |
| **Training & change management** | Train all teams on Solarbase equivalents of their QB workflows. | Medium |
| **Archive & decommission** | Export final QB data, archive, cancel subscription. | Low |

---

## Dependencies Between Tracks

```
Track 1 (Quoting)     → Track 4 (Sunset QB): Sales needs quoting in Solarbase before QB can go
Track 2 (Operations)   → Track 3 (Zoho): Ops needs payment status, which requires Zoho sync
Track 3 (Zoho)         → Track 2 (Operations): Automated sync enables real-time ops dashboards
Track 4 (Sunset QB)    → Track 1 + 2: Can't sunset until both sales and ops are on Solarbase
```

## Suggested Sequencing

**Now → June 2026 (8 weeks):**
- Launch quoting to core sales (Track 1 — high-priority items)
- Build automated Zoho invoice + payment sync (Track 3 — unlocks everything downstream)
- Audit remaining QB workflows (Track 4)

**July → September 2026 (12 weeks):**
- Operations workflow buildout (Track 2 — with Zoho data now flowing)
- Commission calculations, addendum workflow, PDF quotes (Track 1 — medium-priority)
- QB parallel run, training, data migration (Track 4)
- Automate loan tape delivery (Track 3)

**October 2026+:**
- Wider sales launch (Track 1)
- QB decommission (Track 4)
- AI/MCP layer on top of stable platform (per technology roadmap Phase 2-3)
