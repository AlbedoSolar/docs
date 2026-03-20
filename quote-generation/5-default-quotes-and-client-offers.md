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

## Architecture

| Component | Location | Purpose |
|-----------|----------|---------|
| `quote-solver` | `supabase/functions/quote-solver/` | Server-side tier solving (solve_tiers mode) |
| `quote-generator` | `supabase/functions/quote-generator/` | Individual quote generation (used by client offer) |
| `DefaultQuotesPage` | `frontend/quotes-app/src/pages/` | Internal tier generation UI |
| `DefaultQuoteCards` | `frontend/quotes-app/src/components/organisms/` | Gold/Silver/Bronze card display with offer creation |
| `ClientOfferPage` | `frontend/quotes-app/src/pages/` | Public client-facing offer page |
| `quote_offers` table | Supabase | Stores offer tokens, parameters, tracks views |
| `DefaultQuoteRegenerateForm` | `frontend/quotes-app/src/components/molecules/` | Parameter form (admin: advanced options) |
