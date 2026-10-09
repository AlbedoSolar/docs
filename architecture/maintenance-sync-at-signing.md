# Maintenance rows at signing — design

**Status:** proposal, 2026-10-09. Awaiting Jake's go.
**Scope:** `public.project_maintenances` gets its rows the moment a Financiado
quote signs, from the same code path that materializes the cash flows.
**Not in scope:** the ops proposal for a visit date + derived statuses (separate
design), 577-01's through-2031 rows (awaiting the team).

## 1. Problem

`project_maintenances` has only ever been filled by one-off SQL seeds
(2026-04-21, 2026-05-27, 2026-10-09). Nothing runs at signing, so every
project signed after the last seed is invisible on the Mantenimientos page
until someone notices — ops did on 2026-05-27 (5 projects) and again on
2026-10-08 (19 projects, 113 rows). The gap reopens with the next signature.

The rules themselves are settled and live (see §3). What is missing is a
home in code and a trigger point.

## 2. Decision in one paragraph

Add `seedProjectMaintenancesForSignedQuote(quoteId)` to the shared
`@albedo/supabase-sdk` client, next to
`materializeMonthlyCashFlowsForSignedQuote`, and call it from the
contract-generator's final-sign path right after the cash flows exist and the
addendum seam is stamped. It is idempotent, never blocks a signature, and
records any failure in `signing_failures`. A nightly dbt guard test fails
when a signed Financiado project has no rows, so a missed seed is seen the
next morning instead of months later. No trigger, no cron sweep for now.

## 3. The rule (single definition, moved to TypeScript)

Three pieces, each already ruled and in production since June. The 2026-10-09
migration is the last SQL copy; after this ships the TypeScript function is
canonical and the SQL files are history.

| Piece | Rule | Source |
|---|---|---|
| Who | signed Financiado quote, not deleted, not addended; estimate not deleted; project has no rows yet | eligibility used by every seed |
| How many years | `1 .. ceil(max payment_number / 12)` of the live quote's cash flows. **No end-of-schedule cap** (ruled 2026-10-09) | `2026-10-09-seed-missing-project-maintenances.sql` |
| Scheduled date | `COALESCE(projects.client_delivery_date, first cash-flow month with monthly_payment > 0) + year × 12 months` | Scenario A, Fernanda 2026-06-04; `2026-06-07-anchor-maintenance-on-client-delivery-date.sql` |
| Cost | cash-flow rows with `maintenance_cost ≥ 0.01`, ascending by `payment_number`; `first_paid_year = ceil(first charge's payment_number / 12)`; Nth charge → year `first_paid_year + N − 1`; other years `NULL` | ordinal mapping, `2026-06-12-fix-cost-mapping-ordinal-aligned.sql` |
| Flags | `albedo_pays = cost ≥ 0.01`; `is_warranty_covered = NOT albedo_pays` | `2026-06-12-treat-near-zero-cost-as-warranty.sql` |
| Currency | `estimates.project_currency` | `2026-07-15-project-maintenances-cost-currency.sql` |
| Status | `Pendiente`; `actual_date` NULL | — |

Reads: the quote's `monthly_cash_flows` rows directly (not the chain view).
At signing the live quote owns the whole forward schedule, which is what
maintenance is scheduled against. The addendum case is handled separately
(§5).

Writes: one `INSERT … ON CONFLICT (project_id, maintenance_year) DO NOTHING`
through the service-role client. RLS already restricts INSERT to
`service_role` (2026-06-22), which is exactly this caller.

## 4. Placement in the sign path

`packages/lambdas/contract-generator/src/controllers.ts`, `updateContract`,
inside the `signedOn && !isPartnerContract` block, after
`supersedeAddendumChain` and before the chain-health check:

1. `updateQuoteStatus(quote, 'signed')` — existing
2. `materializeMonthlyCashFlowsForSignedQuote(...)` — existing
3. `supersedeAddendumChain(...)` — existing
4. **`seedProjectMaintenancesForSignedQuote(quote)`** — new
5. post-sign chain-health verification — existing

Why here and not earlier: step 4 needs the cash-flow rows (step 2) and, for
an addendum, the predecessor already marked `addended_at` (step 3) so the
eligibility query sees the right live quote.

Why not a trigger: the rule reads cash flows, estimates, projects and the
addendum chain, and §5 needs a judgement call. That is business logic; the
code-over-Postgres rule keeps it in TypeScript. Only the sign path and the
one-off seeds write these rows, so one call site covers it.

Failure handling, same pattern as the cash-flow step:

- Wrapped in its own `try/catch`. A failure never aborts the signature — the
  client signed on paper and blocking the record helps no one.
- Failure is recorded in `signing_failures` with
  `stage = 'seed_project_maintenances'` and the quote/contract ids, so it
  shows up where the other sign-time failures already do.
- Contado quotes: the function returns early with `{ skipped: 'contado' }`.
  Partner contracts never reach this block.
- Re-running (rep corrects `signed_on`, or the seed is replayed by hand) is a
  no-op on projects that already have rows.

## 5. Adendas: never re-cost silently

When the quote being signed has `replaces_quote_id`, the project already has
rows from the original signing. Three options were considered:

| Option | Effect | Verdict |
|---|---|---|
| Re-cost existing rows from the new schedule | 1037-01's adenda carried zero maintenance charges; this would have quietly zeroed five paid years | rejected |
| Leave rows untouched | Correct for most adendas (payment holidays, grace months) but wrong when the term changes | not enough alone |
| Leave rows untouched **and flag when the new schedule disagrees** | Keeps history; a human decides | **chosen** |

Concretely, on an addendum signing the function:

1. Computes what the rows *would* be from the new live quote (same rule as §3).
2. Compares against the existing rows on `maintenance_year`: number of years,
   and cost per year (0.01 tolerance).
3. If identical → nothing to do.
4. If different → inserts nothing, changes nothing, and records a
   `signing_failures` row with `stage = 'maintenance_recost_review'` and the
   diff in `context` (years added/removed, cost deltas). Finanzas/ops resolve
   it by hand, with a migration if rows change.

Years the new schedule *adds* beyond the existing last year (a term
extension) are included in the diff, not auto-inserted — same reasoning.

## 6. Things that deliberately do not happen

- **Delivery date set later.** Rows anchored on the first payment month do
  not move when ops fills `client_delivery_date` afterwards (679-01 precedent,
  2026-06; restated 2026-10-09 for the 19). A re-anchor on that event is a
  possible later feature; it is not part of this change.
- **Annulment.** `annul_project` does not touch `project_maintenances`; the
  Mantenimientos page already excludes annulled projects. Pending rows on an
  annulled project stay as they are. If ops wants them visibly cancelled,
  that is a status, not a delete.
- **Contado** projects get no rows.
- **No nightly sweep job.** A scheduled Lambda that re-runs the seed for
  uncovered projects is easy to add later (precedent:
  `performance-reports-stack.ts`, 06:00 cron). The guard test in §7 is
  enough to start with; add the sweep only if the guard fires more than
  once.

## 7. Detection: a dbt guard test

New singular test in the dbt repo, run nightly with the others:

> signed Financiado projects (live quote, not deleted, not addended, project
> not annulled) with zero `project_maintenances` rows → must return 0 rows.

Baseline today: 0 (after the 2026-10-09 seed). Failing the nightly run is the
alert; the fix is to replay the seed function for the listed projects. This
is the same shape as `test_addendum_chain_linkage`.

## 8. Rollout

1. **infra** — `supabase-sdk/src/client.ts`: the function + a unit test whose
   fixture replays the 19 projects of the 2026-10-09 seed and asserts the
   function reproduces all 113 rows (dates, costs, flags, currency) exactly.
   That test is the proof that the TypeScript copy equals the SQL rule.
2. **infra** — `contract-generator/src/controllers.ts`: the call in
   `updateContract`; `signing_failures` gains the two new `stage` values (no
   schema change; the column is text).
3. **dbt** — the guard test.
4. **docs** — wiki `13-mantenimientos.md` "Relación con otras entidades" still
   says the schedule comes from lease payments; update to the §3 rule.
   CHANGELOG entry in infra for the behaviour change (no DB migration).
5. Ship infra to `staging`; test with one staging signature; promote.

Estimate: one working day for 1–2, an hour each for 3–4.

## 9. Open questions

1. **Addendum diff threshold.** Flag on any cost difference ≥ 0.01, or only
   on a change in the number of years? Proposal: any difference — adendas are
   rare (eight chains in total), a false flag costs a minute.
2. **Replaying the seed by hand.** When a guard fires, who runs the replay —
   a small CLI script in infra (`scripts/seed-project-maintenances.ts
   --project 2574-02`), or Jake via the SDK in a session? Proposal: the
   script, so it is not a Jake-only operation.
3. **Years 1–2 on partner-covered projects.** The rule derives warranty years
   from the cash flows (years without a charge). Providers whose contracts
   say "no free years" (the July Ecolumen/Multienergías backfills) were fixed
   by hand. Leave as is, or pull the free-year count from `provider_terms`?
   Out of scope here; noted so it is not forgotten.
