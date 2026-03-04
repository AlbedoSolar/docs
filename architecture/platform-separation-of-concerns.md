# Platform Architecture: Separation of Concerns

How responsibilities are divided across AWS, Supabase, and the frontend — and where new development should go.

---

## Table of Contents

1. [Current Architecture Overview](#1-current-architecture-overview)
2. [Layer Responsibilities](#2-layer-responsibilities)
3. [Supabase Edge Functions](#3-supabase-edge-functions)
4. [AWS Lambda Functions](#4-aws-lambda-functions)
5. [Database Layer (Supabase PostgreSQL)](#5-database-layer-supabase-postgresql)
6. [Frontend (React App)](#6-frontend-react-app)
7. [Migration Direction: AWS → Supabase](#7-migration-direction-aws--supabase)
8. [MCP Server Compatibility](#8-mcp-server-compatibility)

---

## 1. Current Architecture Overview

The platform spans four runtime layers:

```
┌─────────────────────────────────────────────────────────┐
│                    Frontend (React)                      │
│          Vite + Redux + Supabase Client SDK              │
└────────────┬──────────────────────┬─────────────────────┘
             │                      │
     Supabase Client SDK      HTTP (fetch)
     (realtime, auth, CRUD)         │
             │               ┌──────┴──────┐
             ▼               ▼              ▼
┌──────────────────┐  ┌───────────┐  ┌───────────────┐
│    Supabase DB   │  │ Supabase  │  │  AWS API GW   │
│   (PostgreSQL)   │  │   Edge    │  │  + Lambda     │
│                  │  │ Functions │  │               │
│ • Tables + RLS   │  │           │  │ • Asana       │
│ • Triggers       │  │ • Quote   │  │ • Zoho        │
│ • Functions      │  │   gen     │  │ • QuickBase   │
│ • Realtime       │  │ • Rates   │  │ • Scraping    │
│                  │  │ • Calcs   │  │ • Auth        │
└──────────────────┘  └───────────┘  └───────────────┘
```

**Key principle:** New business logic goes into Supabase Edge Functions. AWS Lambda is retained only for integrations with external services that require specific networking, secrets, or long-running execution.

---

## 2. Layer Responsibilities

| Layer | Owns | Does NOT Own |
|---|---|---|
| **Frontend** | User interaction, form orchestration, display logic, client-side validation | Business calculations, data integrity rules, secrets |
| **Supabase Edge Functions** | Business calculations, domain logic, orchestration of multi-step computations | Persistent state, data integrity, external service auth |
| **Supabase Database** | Data integrity, row-level security, audit trails, referential constraints, change tracking | Business display logic, external API calls |
| **AWS Lambda** | External service integrations (Asana, Zoho, QuickBase), scheduled jobs, scraping pipelines | Core business calculations, CRUD operations |

---

## 3. Supabase Edge Functions

Edge functions are the **primary compute layer for business logic**. Each function serves a single domain concern.

| Function | Domain | Purpose |
|---|---|---|
| `quote-generator` | Quote generation | Full quote engine: retail projections, NII table, KPIs, APR goal-seek, lease/amortization schedules |
| `quote-rates` | Rate lookup | Location-based rate resolution (maintenance, insurance, legal fees, retail projections) for the quote wizard |
| `estimate-calculations` | Estimate defaults | Applies override-or-default logic for estimate-level fields (solar tax, legal costs, commission, asset transfer) |
| `calculate-interest` | Financial math | XIRR/IRR, amortization, and monthly cashflow computations |
| `fetch-exchange-rates` | Currency | Fetches live exchange rates from CurrencyFreaks and persists to DB |
| `data-chat` | AI/search | RAG-based Q&A over documentation using OpenAI embeddings |
| `import-raw-data-from-quickbase` | Data sync | Imports QuickBase table records into Supabase |

### Shared modules (`_shared/`)

Edge functions share logic through a `_shared/` directory (Deno convention):

- **quote-helpers.ts** — Config loading, provider pricing, tax/insurance/maintenance rate resolution, legal fee tiers, commission schedules, estimate context loading
- **quote-calculator.ts** — NII table generation, KPI computation, recommended APR
- **quote-goal-seek.ts** — APR goal-seek via bisection/Newton methods
- **quote-integration-flow.ts** — Lease calculation, servicios table, amortization, integration KPI rollup
- **quote-utils.ts** — CORS, JSON responses, Supabase admin client, type coercion

### When to add a new edge function vs. extend an existing one

- **New function** when the domain concern is distinct (different inputs, different callers, different lifecycle). Example: `estimate-calculations` is separate from `quote-rates` because they serve different workflows — estimate creation vs. quote wizard — with different input shapes.
- **Extend existing** when you're adding a closely related output to an existing call that shares the same inputs and caller. Example: adding a new rate field to `quote-rates` that depends on the same location inputs.

---

## 4. AWS Lambda Functions

Lambdas remain for workloads that require AWS-specific capabilities. **No new business calculation logic should be added here.**

### Active — External integrations

| Lambda | Purpose |
|---|---|
| `asana-webhook-handler` | Processes inbound Asana webhook events |
| `asana-webhook-creator` | Registers/manages Asana webhook subscriptions |
| `asana-automated-messages-handler` | Sends automated Asana task messages |
| `zoho-integrations` | Zoho Books API integration (invoices, contacts) |
| `supabase-client-handler` | Server-side Supabase operations requiring service role key from AWS |
| `auth` | Authentication token handling |

### Active — Scraping pipeline

| Lambda | Purpose |
|---|---|
| `electricity-bills-scraper-dispatcher` | Schedules scraping jobs via EventBridge |
| `electricity-bills-scraper-authenticator` | Handles scraper auth sessions |
| `electricity-bills-scraper-worker` | Executes individual scraping tasks |
| `electricity-bills-scraper-consolidator` | Aggregates scraped results |
| `electricity-bills-gdrive-uploader` | Uploads consolidated bills to Google Drive |

### Legacy — Migrated or deprecated

| Lambda | Status |
|---|---|
| `quote-generator` | **Migrated** to Supabase edge function |
| `calculations-service` | **Migrated** to `estimate-calculations` + `quote-rates` edge functions |
| `quickbase-import-kickoff` | Largely replaced by `import-raw-data-from-quickbase` edge function |

---

## 5. Database Layer (Supabase PostgreSQL)

The database enforces **data integrity and auditability**, not business logic.

### What belongs in the database

- **Row-level security (RLS)** — Access control per user/role
- **Referential integrity** — Foreign keys, unique constraints, check constraints
- **Audit triggers** — Automatic logging of state changes (e.g., `trigger_quote_status_history`, `trigger_track_project_created`, `trigger_track_estimate_created`)
- **Business rule triggers** — Invariants that must hold regardless of caller (e.g., `trg_check_one_signed_quote_per_project`)
- **Change tracking** — The `track_database_change()` function logs mutations across ~20 tables to `database_change_log`
- **Default assignment triggers** — e.g., `set_project_phase_payment_schedule` auto-assigns payment schedules

### What does NOT belong in the database

- Multi-step calculations (use edge functions)
- External API calls (use Lambda or edge functions)
- Display formatting or UI-specific transformations (use frontend)

---

## 6. Frontend (React App)

The frontend is a **thin orchestration layer**. It calls backend services and renders results.

### Service layer (`src/services/`)

Each service file maps to a single backend concern:

| Service | Backend | Transport |
|---|---|---|
| `supabase-data-service.ts` | Supabase DB | Supabase Client SDK |
| `quote-rates-service.ts` | `quote-rates` edge function | HTTP + Supabase anon key |
| `estimate-calculations-service.ts` | `estimate-calculations` edge function | HTTP + Supabase anon key |
| `currency-service.ts` | Supabase DB | Supabase Client SDK |
| `project-service.ts` | Supabase DB | Supabase Client SDK |
| `addendum-service.ts` | Supabase DB | Supabase Client SDK |
| `activity-logging-service.ts` | Supabase DB | Supabase Client SDK |
| `pipeline-timeline-service.ts` | Supabase DB | Supabase Client SDK |

### Frontend rules

1. **No business calculations in the frontend.** All calculations happen in edge functions or the database. The frontend only renders results and passes user inputs.
2. **Supabase auth headers on all edge function calls.** Use the `VITE_SUPABASE_ANON_KEY` as both `apikey` and `Authorization: Bearer` headers.
3. **Service factory pattern.** `data-service.factory.ts` + `data-service.interface.ts` allow swapping data service implementations.

---

## 7. Migration Direction: AWS → Supabase

### Completed migrations

| What | From | To | Date |
|---|---|---|---|
| Quote generation engine | `quote-generator` Lambda (Python) | `quote-generator` edge function (TypeScript) | 2025 |
| Quote rate lookups | `calculations-service` Lambda | `quote-rates` edge function | 2025 |
| Estimate calculations | `calculations-service` Lambda | `estimate-calculations` edge function | 2026-03 |
| QuickBase import | `quickbase-import-kickoff` Lambda | `import-raw-data-from-quickbase` edge function | 2025 |

### Remaining on AWS (no migration planned)

These stay on AWS because they depend on AWS-specific services or have requirements that edge functions cannot satisfy:

- **Asana integration** — Webhook endpoints registered with Asana require stable URLs and specific auth flows. SQS queues provide retry/backoff.
- **Zoho integration** — OAuth token management stored in SSM Parameter Store.
- **Electricity bill scraping** — Multi-stage pipeline using EventBridge scheduling, SQS queues, S3 storage, and Google Drive uploads. Long-running, stateful.
- **Auth Lambda** — Tied to API Gateway authorizers.

### Decision criteria for new features

Ask these questions when deciding where a new feature lives:

1. **Does it call an external API that requires AWS secrets or networking?** → AWS Lambda
2. **Does it enforce a data integrity rule that must hold regardless of caller?** → Database trigger
3. **Does it compute business values from inputs?** → Supabase Edge Function
4. **Does it only display or collect data?** → Frontend

---

## 8. MCP Server Compatibility

An [MCP (Model Context Protocol)](https://modelcontextprotocol.io) server would allow AI agents to interact with Albedo's data and operations through a standardized tool interface. The current architecture is well-positioned for this if we follow a few principles going forward.

### Why edge functions map well to MCP tools

Each Supabase Edge Function already behaves like an MCP tool:
- **Typed inputs** — JSON request body with a defined schema
- **Typed outputs** — JSON response with a predictable shape
- **Stateless** — No server-side session; all context comes from the request
- **Single concern** — One function per domain operation

An MCP server would wrap these functions as tools:

```
MCP Tool: "calculate_estimate_values"
├── Input schema  → EstimateCalculationInputs (already defined)
├── Execute       → POST /functions/v1/estimate-calculations
└── Output schema → EstimateCalculatedValues (already defined)

MCP Tool: "get_quote_rates"
├── Input schema  → QuoteRatesRequest (already defined)
├── Execute       → POST /functions/v1/quote-rates
└── Output schema → QuoteRatesResponse (already defined)

MCP Tool: "generate_quote"
├── Input schema  → QuoteGeneratorRequest
├── Execute       → POST /functions/v1/quote-generator
└── Output schema → Full quote with KPIs, schedules, etc.
```

### Design guidelines for MCP readiness

Follow these when building new edge functions or modifying existing ones:

1. **Explicit input/output types.** Every edge function should have TypeScript interfaces for its request and response shapes. These become the MCP tool's `inputSchema` and output description. The frontend service files already define these (e.g., `EstimateCalculationInputs`, `QuoteRatesRequest`).

2. **No implicit context.** Edge functions should not depend on ambient state (cookies, sessions, URL path parameters). All inputs should come from the JSON body. This is already the case — all current edge functions accept a single `POST` body.

3. **Granular functions over monoliths.** Keep functions focused on one operation. An MCP agent should be able to call `get_quote_rates` without also triggering quote generation. The current separation (`quote-rates` vs. `estimate-calculations` vs. `quote-generator`) follows this principle.

4. **Idempotent reads, explicit writes.** Functions that compute values (rates, calculations) should be side-effect-free. Functions that modify state (future: create estimate, update project) should be clearly labeled and separate. This lets an MCP server mark tools as "read-only" vs. "mutating" for agent safety.

5. **Error responses with structured messages.** Return `{ "error": "human-readable message" }` with appropriate HTTP status codes. MCP agents need to distinguish between invalid inputs (400), auth failures (401), and server errors (500).

6. **Document tool semantics in code.** Add a JSDoc comment at the top of each edge function's `index.ts` describing what it does, its required inputs, and what it returns. These comments become the MCP tool descriptions that help agents choose the right tool.

### MCP server implementation plan

When ready to build the MCP server:

**Phase 1 — Read-only tools (low risk)**
- Wrap existing edge functions as MCP tools: `get_quote_rates`, `calculate_estimate_values`, `calculate_interest`, `generate_quote`
- Add Supabase read-only tools: `get_project`, `get_estimate`, `list_quotes`, `get_exchange_rates`
- These are safe for agents to call freely

**Phase 2 — Write tools (requires guardrails)**
- `create_estimate`, `update_estimate`, `create_project`
- Wrap existing `supabase-data-service` methods
- Add confirmation/approval mechanisms for destructive operations

**Phase 3 — Integration tools**
- Expose Asana/Zoho/QuickBase operations through MCP
- These proxy to existing AWS Lambdas
- Require explicit user authorization per-operation

### Architecture for the MCP server itself

```
┌──────────────────────────────────────────────────┐
│                  MCP Server                       │
│          (Supabase Edge Function or               │
│           standalone Node.js service)             │
│                                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│  │ Read     │  │ Compute  │  │ Integration  │   │
│  │ Tools    │  │ Tools    │  │ Tools        │   │
│  │          │  │          │  │              │   │
│  │ Supabase │  │ Edge Fn  │  │ AWS Lambda   │   │
│  │ Client   │  │ HTTP     │  │ via API GW   │   │
│  └────┬─────┘  └────┬─────┘  └──────┬───────┘   │
│       │              │               │            │
└───────┼──────────────┼───────────────┼────────────┘
        ▼              ▼               ▼
   Supabase DB    Edge Functions    AWS Lambdas
```

The MCP server itself could be:
- **Another Supabase Edge Function** — simplest deployment, shares the same auth context, but limited to Deno runtime and edge function execution limits
- **Standalone Node.js service** — more flexibility for long-running MCP sessions, can use the Supabase service role key and AWS SDK directly

The right choice depends on how MCP clients will connect (stdio vs. HTTP/SSE) and whether sessions need to persist beyond a single request.
