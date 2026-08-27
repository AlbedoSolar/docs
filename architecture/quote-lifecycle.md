# Quote lifecycle — states, column semantics, and enforcement

Status: **governing doc** (Jake's ruling 2026-08-26, from the post-QB audit).
The quotes table holds two entities sharing one lifecycle, and this document
declares what each status means, which transitions are legal, what each
pricing column means *per stage*, and where each rule is enforced. When a
flow and this doc disagree, one of them is wrong — figure out which before
shipping.

## What the quotes table is

One row per **financing structure ever proposed for an estimate**, with a
lifecycle. Two entities share the table:

- **Candidate** (`pending`, and `reverted`/`rejected` as re-workable
  variants): a saved calculadora pricing session — one way of financing the
  estimate. Mutable, reopenable, comparable. Candidates are load-bearing:
  every offer is generated FROM a quote (`quote_offers.quote_id`), the
  Gold/Silver/Bronze flow creates three per estimate, and the calculadora
  reopen flow edits them. As of 2026-08-26: 1,560 pending rows = 81% of the
  table.
- **Deal** (`approved`, `signed`, `signed_and_addended`): a contractual or
  contract-bound record. Progressively immutable (see column semantics).

We deliberately did NOT split these into separate tables. The
read-through-views architecture (see `read-through-views.md`) provides the
entity separation logically: deal consumers read stage-scoped views and can
never see candidates. A physical split would re-point every offer FK and
rebuild the calculadora save flow for a distinction the views already make.

## Statuses and legal transitions

Statuses in production: `pending`, `approved`, `signed`,
`signed_and_addended`, `reverted`, `rejected`, `cancelled`, `annulled`.

| From | To | Via (the ONLY legal path) |
|---|---|---|
| — | `pending` | calculadora save (quote-generator); QB import (legacy) |
| `pending` | `approved` | `approve-quote-variant` edge fn (base variant), or child quote INSERTed directly as `approved` (non-base variant) |
| `pending` | `rejected` / `cancelled` | APD status menu |
| `approved` | `signed` | contract-generator final-sign path ONLY (materializes cash flows, runs addendum supersession). The APD menu deliberately does not offer it. |
| `approved` | `pending` (base) / `reverted` (non-base) | `disapprove-quote-variant`, or displaced by a newer approval in the same project |
| `signed` | `signed_and_addended` | `supersede_addendum_chain` RPC at addendum signing ONLY |
| `signed`/`signed_and_addended` | `annulled` | `annul_project()` RPC ONLY (atomic: seam stamp, contract void, workflow move). Never via a plain status write — see `database/annulment.md`. |
| `rejected` | `pending` / `cancelled` | APD status menu |
| `cancelled`, `annulled`, `signed_and_addended` | — | terminal (`annulled` reverses only via `unannul_project()`) |

At most ONE `approved` quote per project at a time (enforced by
`approve-quote-variant`'s revert sweep). At most one signed original per
project (dbt invariant B3).

## Column semantics per stage — the rule that was missing

Pricing columns on quotes (`annual_interest_rate`, `monthly_interest_rate`,
`retail_price_project_currency`, and pricing keys inside
`generation_parameters`):

1. **Candidate stage — PROVISIONAL.** The calculadora's goal-seek output.
   Its payment mode returns "the exact rate the user's target implies" — a
   goal-seeking artifact, not a commitment. Nothing downstream may treat a
   candidate's pricing as contractual.
2. **At approval — STAMPED from the approved offer cell.** Ruling
   2026-08-26: *the offer's solved rate is the true rate.* The offer is
   calculated once and stored statically (`quote_offers.variant_matrix`);
   the approved cell builds the signed schedule and is the contract's
   pricing provenance. Approval copies the cell's `solved_annual_rate` (and
   offer linkage) onto the quote. The cell remains the immutable provenance
   record; the quote row is the queryable surface.
3. **At signing — FROZEN.** RLS signed-protection (`quote_is_signed()` in
   the UPDATE policies) blocks edits; contract snapshots freeze document
   terms independently.

Known enforcement gap (open task): `approve-quote-variant`'s
`isApprovingBase` branch updates status only and skips the stamp — the
child-quote branch stamps correctly. Until fixed, base-variant approvals
leave the calculadora artifact in place. Four signed quotes were backfilled
for exactly this on 2026-08-26
(`database/migrations/2026-08-26-backfill-quote-rates-from-approved-offer.sql`).

The standing enforcement that the stamp matches reality is dbt's
`test_interest_payments_match` (stored rate vs signed schedule, nightly).

## Related fields with the same shape

`quotes.contract_signing_date` went through this exact lifecycle reasoning
first: truth lives on `contracts.signed_on`, the quote column is a
trigger-maintained mirror (`trg_sync_quote_signing_date`) slated for
`_legacy` rename. See `database/triggers/quote-signing-date-sync.md` and the
2026-08-25 decisions-log entry. When auditing further quote columns, ask the
same question: *is this the canonical home, a stamped mirror, or a fossil?*

## Reads

Deal reads go through stage-scoped views (`stg_signed_quotes` in dbt,
`v_active_projects` / `v_operations_projects` / `v_project_signing` in the
app) per `read-through-views.md`. Raw-table reads remain legitimate for
by-id lookups and PostgREST embed targets only.

## Open governance items

- **Candidate retention**: no policy exists for the 1,500+ pending rows
  (dead tier-trio siblings, abandoned sessions). Needs a ruling: archive
  after N months without an offer? Never? (Owner: Jake.)
- **Base-path stamping fix**: queued task (approve-quote-variant).
- **`reverted` semantics**: only ever written by the approval sweep; nothing
  reads it distinctly from `pending`. Candidate for consolidation or
  documentation of intent.
