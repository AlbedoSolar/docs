# Default Quotes & Client Offers

## Overview

The default quotes system automatically generates three financing tiers (Gold, Silver, Bronze) for each estimate, then enables sales reps to create client-facing offer pages for any tier.

## Quote Tier Solving

### Goal

Find three tiers of quote offerings with different IRR targets:
- **Gold**: IRR ~14% (best return for Albedo, shortest term)
- **Silver**: IRR ~13%
- **Bronze**: IRR ~12% (lowest return, longest term, easiest for client)

### How It Works

1. The system reads the estimate's **monthly solar savings** — this becomes the target monthly payment for all tiers.

2. For each tier, the system finds the **loan term** (in months) where the achieved IRR is closest to the target. At each candidate term:
   - The backend goal-seeks for the APR that produces a monthly payment equal to the client's solar savings
   - The achieved IRR is extracted from the resulting KPIs
   - The term with the closest IRR match is selected

3. **Search strategy**: Two-pass brute force in the `quote-solver` edge function:
   - Pass 1: Coarse scan every 6 months from 12 to 180 months (~28 evaluations)
   - Pass 2: Fine scan every month within ±6 months of the best coarse match (~12 evaluations per tier)

   Binary search was attempted but doesn't work because the IRR-vs-term relationship is **not monotonic** when monthly payment is fixed and APR floats.

4. **Result**: Three tiers with different solved terms, different APRs, but roughly the same monthly payment (equal to the client's savings).

### Technical Details

- **Endpoint**: `quote-solver` edge function, mode `solve_tiers`
- **Authentication**: Requires JWT (verify_jwt = true)
- **APR floor**: Not applied in solve_tiers mode — the monthly payment goal-seek already constrains the economics
- **IRR cap**: Values above 100% are treated as failed results (the 12000% sentinel from `irrBisection` upper bound)
- **Down payment**: 0% for tier solving (most conservative case)
- **Grace period**: 3 months (fixed)

## Client Offers

### Goal

Present the client with financing options that vary by term length and down payment, all based on the selected tier's economics.

### How It Works

When a sales rep clicks "Crear Oferta" on a tier, the system:

1. Creates a `quote_offers` record with:
   - The tier's solved term as `base_term_months`
   - The tier's solved APR and achieved IRR in `generation_parameters`
   - A unique `token` for the public URL

2. The client offer page (`/offer/:token`) generates up to 6 variants:
   - 3 terms: base term, base - 12 months, base + 12 months
   - 2 down payments: 0% and 10%

3. **Each variant uses the tier's solved APR as a fixed rate** — no goal-seeking on the client offer page.
   - With a fixed APR, monthly payments vary naturally by term and down payment
   - The achieved IRR is at or above the tier's target for all variants

4. Failed variants (where the term is too short) and down payment options where all terms fail are hidden automatically.

### What the Client Sees

- A professional offer page with:
  - Project and equipment details
  - Financial comparison table showing monthly payment, insurance, maintenance, and interest across all viable options
  - Solar savings summary
  - Collapsible 30-year payment plan tables per variant
  - Terms and conditions

- The client naturally gravitates toward the option where monthly payment < monthly savings (immediate cost reduction).

### Sales Flow

1. Rep generates Gold/Silver/Bronze tiers (one button click)
2. Rep starts with Gold offer (highest return for Albedo) — clicks "Crear Oferta", copies link, sends via WhatsApp
3. If client doesn't bite, rep falls back to Silver, then Bronze
4. Each tier is a separate URL — client only sees one tier at a time

## Quote Explorer (V2)

### Goal

Replace the fixed Gold/Silver/Bronze tiers with an interactive calculator where the sales rep directly controls monthly payment, term, and down payment, sees the resulting IRR in real-time, and generates offer sheets from the explored values.

### How It Works

The Quote Explorer page (`DefaultQuotesPageV2`) presents three input dials and a live IRR display:

1. **Inputs**: Monthly payment, term (months), and down payment (%)
2. **Output**: IRR updates reactively as the user adjusts any input (400ms debounce)
3. **IRR Lock**: User can fix the IRR at a desired value. When locked:
   - Changing the term solves for the monthly payment that maintains the locked IRR
   - Changing the monthly payment scans for the term that maintains the locked IRR
   - Changing the down payment solves for the monthly payment that maintains the locked IRR

### Initial Values

On page load, the system calls `irr-calculator` in `solve_for_payment` mode with `target_irr: 16` and `term_months: 60` to find a starting monthly payment that produces a Gold-tier IRR. This gives the user a meaningful starting point rather than zeros.

### IRR Tiers (Color Coding)

- **Gold** (>= 16%): amber — best return for Albedo
- **Silver** (>= 14%): slate
- **Bronze** (>= 12%): orange
- **Red** (< 12%): red — below minimum threshold

### Savings Comparison

The page displays the estimate's monthly solar savings and compares it to the current monthly payment. The payment card and a comparison bar turn green when payment <= savings (client saves money from day one) or red when payment exceeds savings.

### Offer Sheet Generation

1. User clicks "Generar Oferta" to configure the offer
2. A config panel shows the current down payment (forced as one option) and a second editable down payment value
3. The system calls `quote-solver` in `generate_offer` mode: holds IRR constant, varies term (base +/- 12 months) and down payment options
4. An inline preview matrix shows monthly payments for each term x down payment combination
5. "Crear Enlace Compartible" inserts into `quote_offers` and generates a public URL
6. The public offer page (`/offer/:token`) is the same `ClientOfferPage` used by the V1 flow

### Backend: `irr-calculator` Edge Function

**Location**: `supabase/functions/irr-calculator/`

A lightweight endpoint designed for responsive UI interactions. Three modes:

#### Mode: `forward` (default)

Given monthly payment + term, returns the resulting IRR.

- **Input**: `{ estimate_reference, monthly_payment, term_months, down_payment_percentage, grace_period, ... }`
- **Process**: Goal-seeks APR such that the calculated payment matches the input, then runs the full NII forward calc to extract IRR
- **Output**: `{ irr, solved_annual_rate, monthly_payment, term_months, retail_price }`

#### Mode: `solve_for_payment`

Given a target IRR + term, returns the monthly payment that achieves it.

- **Input**: `{ mode: "solve_for_payment", estimate_reference, target_irr, term_months, ... }`
- **Process**: Goal-seeks APR such that IRR matches the target, then extracts the resulting monthly payment from KPIs
- **Output**: `{ irr, monthly_payment, term_months, solved_annual_rate }`

#### Mode: `solve_for_term`

Given a target IRR + monthly payment, scans terms to find the closest match.

- **Input**: `{ mode: "solve_for_term", estimate_reference, target_irr, monthly_payment, min_term, max_term, ... }`
- **Process**: Two-pass brute force (same strategy as `quote-solver` solve_tiers):
  - Pass 1: Coarse scan every 6 months
  - Pass 2: Fine scan every month within +/- 6 of the best coarse match
- **Output**: `{ irr, monthly_payment, term_months, solved_annual_rate }`

#### Internal Architecture

The function is structured in three layers to separate DB queries from computation:

1. **`loadSharedData`** — Runs all DB queries once per request: estimate context, config, tax/insurance/maintenance rates, installation schedule
2. **`buildTermContext`** — Computes term-dependent values (retail price, legal fee, commission schedule, goal-seek args) without any DB calls. Called once per term in forward/solve_for_payment modes, or per-term in the solve_for_term scan
3. **`runForwardCalc`** — Runs the full NII pipeline (lease table, services table, amortization, KPI extraction) for a given APR and term. Pure computation

All service parameters (grace period, WACC, insurance/maintenance rates, etc.) are resolved once into a `ServiceParams` struct at the top of the handler and passed through all layers. This eliminates duplication and ensures consistency.

#### Error Handling

- 422 errors include a Spanish-language `hint` field with actionable guidance (e.g., minimum viable payment at the given term)
- IRR values above 100% are returned as `null` with an explanatory error
- The `solve_for_term` scan logs failed term evaluations with `console.warn` but continues scanning

### V1 vs V2 Coexistence

Both pages exist in the codebase simultaneously:
- **V2** (active): `/projects/:ref/default-quotes/:id` renders `DefaultQuotesPageV2`
- **V1** (preserved): `/projects/:ref/default-quotes-v1/:id` renders `DefaultQuotesPage`

To switch back, swap the components in the route definitions in `AppRouter.tsx`.

## Architecture

| Component | Location | Purpose |
|-----------|----------|---------|
| `quote-solver` | `supabase/functions/quote-solver/` | Server-side tier solving (solve_tiers) and offer generation (generate_offer) |
| `irr-calculator` | `supabase/functions/irr-calculator/` | Lightweight interactive IRR calculation (forward, solve_for_payment, solve_for_term) |
| `quote-generator` | `supabase/functions/quote-generator/` | Individual quote generation (used by client offer page) |
| `DefaultQuotesPage` | `frontend/quotes-app/src/pages/` | V1: Gold/Silver/Bronze tier generation UI |
| `DefaultQuotesPageV2` | `frontend/quotes-app/src/pages/` | V2: Interactive quote explorer with dials and IRR lock |
| `DefaultQuoteCards` | `frontend/quotes-app/src/components/organisms/` | V1: Gold/Silver/Bronze card display with offer creation |
| `ClientOfferPage` | `frontend/quotes-app/src/pages/` | Public client-facing offer page (shared by V1 and V2) |
| `quote_offers` table | Supabase | Stores offer tokens, parameters, tracks views |
| `DefaultQuoteRegenerateForm` | `frontend/quotes-app/src/components/molecules/` | V1: Parameter form (admin: advanced options) |
