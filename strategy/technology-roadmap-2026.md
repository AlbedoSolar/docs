# Technology & AI Roadmap — 2026

Albedo's technology strategy for the next 6–12 months: where we are, where we're going, and how AI accelerates our mission.

---

## Where We Are Today

Albedo has built a fully integrated solar financing platform that handles the complete lifecycle from client intake through quoting, contract signing, payment management, and impact reporting. The platform runs on four layers:

| Layer | Technology | Role |
|-------|-----------|------|
| Frontend | React + Redux + Supabase Client | User-facing quoting tool, project management, reporting |
| Business Logic | Supabase Edge Functions (Deno/TypeScript) | Quote generation, rate calculations, estimate computations, interest/amortization |
| Database | Supabase PostgreSQL + RLS + dbt | Data integrity, row-level security, audit trails, analytics views |
| Integrations | AWS Lambda | Asana, Zoho Books, QuickBase sync, electricity bill scraping |

### Key capabilities already operational

- **End-to-end quoting engine** — From client data to full cash flow schedule with KPIs (IRR, APR, monthly payments), including multi-currency support (GTQ, USD, HNL)
- **Impact reporting** — Automated calculation of installed MW, lifetime kWh production, CO2 avoided, solar savings, and social impact metrics across the entire portfolio
- **Financial modeling** — Monthly cash flows with interest amortization, insurance, maintenance, and commission cost schedules; IVA calculations per country
- **Addendum management** — Contract modifications with cash flow consolidation across original and amended terms
- **Role-based access control** — Permission system with field-level policies (hidden, read-only, editable per role)
- **RAG-based data chat** — AI-powered Q&A over internal documentation using OpenAI embeddings (live)
- **Electricity bill scraping pipeline** — Automated extraction from utility portals (EEGSA, ENERGUATE, ENEE)

### Architecture designed for AI

Our edge functions already behave like AI-ready tools: typed inputs, typed outputs, stateless, single-concern. We've documented a phased plan to expose them via MCP (Model Context Protocol), the emerging standard for connecting AI agents to external tools and data sources. This means every calculation we've built — quoting, rates, estimates, financial projections — can be called by an AI agent with minimal additional engineering.

---

## The Vision: Document to Quote in Minutes

Today, generating a solar financing quote requires a salesperson to manually enter client information, consumption data, equipment specifications, and financial parameters across multiple screens. This process takes 15–20 minutes per quote, requires training, and is error-prone.

Our target state:

```
Client sends electricity bill (PDF or photo)
                    │
                    ▼
    ┌───────────────────────────────┐
    │   AI Document Extraction      │
    │   (Claude Vision API)         │
    │                               │
    │   → Monthly consumption (kWh) │
    │   → Electricity rate ($/kWh)  │
    │   → Client name / tax ID     │
    │   → Address / municipality    │
    │   → Electricity distributor   │
    └───────────────┬───────────────┘
                    │
                    ▼
    ┌───────────────────────────────┐
    │   Intelligent System Sizing    │
    │                               │
    │   → Match consumption to      │
    │     optimal panel config      │
    │   → Look up location rates    │
    │     (insurance, maintenance,  │
    │      legal fees, commissions) │
    │   → Apply country defaults    │
    └───────────────┬───────────────┘
                    │
                    ▼
    ┌───────────────────────────────┐
    │   Auto-Generate 3 Quotes      │
    │   at IRR 12%, 13%, 14%        │
    │                               │
    │   → Full cash flow schedules  │
    │   → Monthly payment options   │
    │   → Savings projections       │
    │   → Impact metrics            │
    └───────────────┬───────────────┘
                    │
                    ▼
    Sales rep reviews, adjusts if
    needed, and sends to client
```

This compresses the quoting process from 20 minutes to under 2 minutes, eliminates data entry errors, and lets the sales team focus on the client relationship rather than the spreadsheet.

---

## Roadmap

### Phase 1: Foundation (Months 1–3)

#### 1.1 Electricity Bill Data Extraction

**What:** Build an AI-powered document processing pipeline that extracts structured data from Central American electricity bills (recibos de luz).

**How it works:**
- Client sends a PDF or phone photo of their electricity bill via WhatsApp, email, or the app
- Claude's vision capability reads the document — handling both clean utility PDFs and imperfect phone photos
- Extracts key fields: monthly consumption (kWh), electricity rate, client name, NIT (tax ID), address, distributor, account number
- Output is validated against known distributor formats (EEGSA, ENERGUATE in Guatemala; ENEE in Honduras; distributors in El Salvador)

**Why it matters:** This is the single highest-friction step in the sales process. Every quote starts with "send me your electricity bill" — automating the extraction removes the bottleneck.

**Built on:** Existing electricity bill scraping infrastructure (we already understand these document formats), Claude Vision API, Supabase Edge Functions.

#### 1.2 Role-Based Access for Sales Team

**What:** Complete the field-level permission system and launch the quoting tool to the core sales team.

**How it works:**
- PolicyFormField component (already built) enforces hidden/read-only/editable states per field per role
- Configure policies for the sales role: hide sensitive financial margins, lock rate parameters, show only client-facing fields
- Sales staff can create quotes without seeing or modifying internal pricing assumptions

**Why it matters:** Scaling the team from power users to the full sales force requires guardrails. This is the gate to broader adoption.

**Status:** Database tables and UI component built; needs configuration, testing, and rollout.

#### 1.3 Default Quote Generation

**What:** Automatically generate 3 quote variants at different return rates (IRR 12%, 13%, 14%) for every estimate, with commissions displayed inline.

**Why it matters:** Sales reps currently generate one quote at a time and manually adjust rates. Auto-generating options with visible commission impact incentivizes higher-margin deals and speeds up the sales conversation.

**Built on:** Existing `quote-generator` edge function — this is orchestration, not new calculation logic.

### Phase 2: AI Integration (Months 3–6)

#### 2.1 MCP Agent Layer — Read-Only Tools

**What:** Expose the quoting and analytics platform to AI agents via MCP (Model Context Protocol).

**Tools in this phase:**
| MCP Tool | Edge Function | Purpose |
|----------|--------------|---------|
| `get_quote_rates` | `quote-rates` | Look up location-based rates for any project |
| `calculate_estimate` | `estimate-calculations` | Compute estimate defaults with override logic |
| `generate_quote` | `quote-generator` | Full quote generation with KPIs and schedules |
| `calculate_interest` | `calculate-interest` | XIRR, IRR, amortization computations |
| `get_project` | Supabase read | Retrieve project details |
| `list_quotes` | Supabase read | List quotes for a project |
| `get_impact_summary` | dbt view | Portfolio-level impact metrics |
| `get_portfolio_financials` | dbt view | Accounts receivable, cash flow summaries |

**Why it matters:** Once these tools exist, any AI interface — Claude, a chatbot, a WhatsApp integration — can answer questions about the portfolio, generate quotes, and run financial scenarios without custom code for each interface.

#### 2.2 Natural-Language Portfolio Analytics

**What:** Extend the existing `data-chat` RAG system to answer quantitative questions about the portfolio using the `data_chat` dbt schema views.

**Example queries:**
- "How many projects did we sign in Q1 2026?"
- "What's our total installed MW in Guatemala?"
- "Which projects have overdue payments?"
- "What's the average IRR across our signed portfolio?"
- "Generate our Q1 impact numbers"

**Built on:** Existing `data-chat` edge function, existing `data_chat` schema with currency-specific views (`v_projects_gtq`, `v_projects_usd`), Supabase pgvector for embeddings.

**Why it matters:** Leadership and donors can query the portfolio directly instead of waiting for someone to run reports. Real-time visibility into portfolio health, impact, and financial performance.

#### 2.3 Automated Impact Reporting

**What:** AI-generated impact reports pulling from the dbt impact models, formatted for donor presentations.

**Data sources (already built):**
- `mart_impact_overall_summary` — MW installed, CO2 avoided, total savings, social impact percentages
- `mart_impact_quarterly` — Quarter-over-quarter growth in clients, projects, and impact categories
- `mart_impact_projects_summary` — Per-project production, savings, and degradation calculations

**Output:** A formatted report (PDF or slide deck) showing portfolio impact with visualizations — generated on demand or on a quarterly schedule.

**Why it matters:** Impact reporting is currently a manual process. Automating it means reports are always current, always consistent, and available on demand — including minutes before a donor meeting.

#### 2.4 Document-to-Quote Pipeline (Full Integration)

**What:** Connect the bill extraction (Phase 1.1) with the MCP agent layer (Phase 2.1) into a single end-to-end flow.

**The complete flow:**
1. Client sends recibo de luz via WhatsApp or email
2. AI extracts consumption, rate, location, and client data
3. Agent calls `get_quote_rates` to look up location-specific rates
4. Agent calls system sizing logic to match consumption to optimal equipment configuration
5. Agent calls `generate_quote` three times at different IRRs
6. Sales rep receives a pre-populated quote package with three options, ready to review and send

**What makes this possible:** Every step in this chain already exists as a discrete, tested edge function. The AI layer is orchestration — calling existing tools in sequence — not new business logic.

### Phase 3: Intelligence (Months 6–12)

#### 3.1 Smart Payment Monitoring & Collections

**What:** AI-powered payment health monitoring using cash flow and Zoho integration data.

**Capabilities:**
- Flag overdue payments based on cash flow schedules
- Predict default risk from payment patterns (late frequency, partial payments)
- Auto-generate follow-up tasks in Asana for collections
- Alert account managers when a client's payment behavior changes

**Built on:** Monthly cash flow data (already in Supabase), Zoho Books integration (already on AWS Lambda), Asana integration (already operational).

**Why it matters:** Portfolio health is the #1 concern for lenders and donors. Proactive monitoring reduces defaults and demonstrates operational maturity.

#### 3.2 Electricity Bill Intelligence

**What:** Extract structured data from scraped electricity bills to improve savings projections.

**Capabilities:**
- Track actual consumption patterns over time (seasonal variation, growth trends)
- Detect rate changes by distributor and update pricing models
- Compare actual solar production vs. projections per project
- Feed consumption data back into the production multiplier (currently hardcoded at 0.13 kWh/W/month for Guatemala) to calibrate per-region

**Built on:** Existing scraping pipeline (5 Lambda functions), bill data in Supabase, impact report models with degradation calculations.

**Why it matters:** Moves savings projections from estimates to actuals. Better projections mean better underwriting, which means lower risk for donors.

#### 3.3 Predictive Portfolio Modeling

**What:** Build models that predict portfolio performance based on historical data.

**Capabilities:**
- Actual vs. projected production comparison per project
- Regional production multiplier calibration from real data
- Equipment degradation validation (is 0.5%/year accurate?)
- Forward-looking portfolio value projections with confidence intervals

**Data required:** 12+ months of actual production data for enough projects to be statistically meaningful. The dbt impact models already calculate `electricity_production_to_date` vs. `total_electricity_production` — this extends that comparison.

**Why it matters:** Donors want to know "are your projections reliable?" Moving from assumed multipliers to data-driven predictions answers that question definitively.

#### 3.4 MCP Agent Layer — Write Tools

**What:** Allow AI agents to create and modify records with appropriate guardrails.

**Tools in this phase:**
- `create_project` — with validation and duplicate detection
- `create_estimate` — with equipment matching and default application
- `update_project_status` — workflow transitions with confirmation
- `create_invoice` — via Zoho integration

**Guardrails:** All write operations require explicit user confirmation. The MCP server marks these tools as "mutating" so agents present changes for review before execution.

#### 3.5 Client Self-Service Portal

**What:** A simple interface where clients can see their payment schedule, solar production, and savings.

**Built on:** Same Supabase data, same dbt views, Supabase realtime for live updates. Minimal new backend — it's a new frontend consuming existing data.

**Why it matters:** Reduces support burden, increases transparency, and demonstrates professionalism to clients and donors alike.

---

## Technology Stack

| Capability | Technology | Why |
|-----------|-----------|-----|
| Document extraction | Claude Vision API | Best-in-class multimodal understanding; handles photos of bills, not just clean PDFs |
| AI agent orchestration | Claude + MCP | Our edge functions are already designed as MCP-compatible tools |
| Embeddings & RAG | Supabase pgvector + OpenAI embeddings (existing) | Already operational in `data-chat`; migration to Claude embeddings possible |
| Analytics models | dbt | Already powering impact, finance, and app views with 51 models |
| Realtime data | Supabase Realtime | Already used in the frontend for live updates |
| External integrations | AWS Lambda | Retained for Asana, Zoho, QuickBase, scraping — no migration needed |
| PDF/report generation | To be selected | Options: Puppeteer, react-pdf, or Google Slides API (already have Drive integration) |

---

## Impact on Operations

| Metric | Today | With Roadmap |
|--------|-------|-------------|
| Time to generate a quote | 15–20 minutes | < 2 minutes |
| Data entry errors in quotes | Common (manual input) | Near zero (AI extraction + validation) |
| Impact report generation | Manual, hours | On-demand, seconds |
| Payment monitoring | Reactive (check when asked) | Proactive (AI alerts) |
| Sales team onboarding | Weeks (complex tool) | Days (guided AI flow) |
| Portfolio queries | Ask analyst, wait | Natural language, instant |
| Savings projection accuracy | Estimated (static multiplier) | Calibrated (actual production data) |

---

## Investment & Dependencies

### What we've already built (sunk cost, operational)
- Complete quoting engine with 7 edge functions
- Impact, finance, and analytics dbt models (51 models)
- Permission system (database + UI components)
- Electricity bill scraping pipeline
- Asana, Zoho, QuickBase integrations
- RAG-based data chat prototype

### What Phase 1 requires
- Claude API access (Vision + text) — usage-based pricing
- 2–4 weeks engineering for bill extraction + validation
- 1–2 weeks for permission configuration + sales launch
- No new infrastructure — runs on existing Supabase + AWS

### What Phases 2–3 require
- MCP server deployment (Supabase Edge Function or standalone Node.js)
- Expanded Claude API usage for agent orchestration
- dbt model extensions for payment monitoring views
- Optional: report generation tooling (Puppeteer or Google Slides API)
- 12+ months of production data for predictive modeling (data collection already underway)

---

## Summary

Albedo's technology platform is transitioning from a manual-entry quoting tool to an AI-powered solar financing engine. The foundation is built: every business calculation, every rate lookup, every financial projection already exists as a discrete, testable, API-callable function. The roadmap layers AI on top of this foundation — not to replace the team, but to make each person dramatically more productive.

The near-term focus is clear: **turn a photo of an electricity bill into a complete financing proposal in under two minutes.** Everything else — portfolio analytics, impact reporting, payment monitoring, predictive modeling — flows naturally from the same architecture and data.
