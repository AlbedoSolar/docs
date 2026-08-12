# Addendum robustness — review packet (August 2026)

*Written 2026-08-12 for peer review of the addendum work shipped 2026-08-11/12. Self-contained: background, findings, changes with commits, how to verify each claim yourself, and the decisions most worth challenging.*

## Background — what an addendum is here

An adenda replaces a project's signed quote with a new one. Since 2026-07-20 it's an automated flow: duplicate estimate → approve variant *as adenda* (stamps `replaces_quote_id` / `original_quote_id` / `addendum_number`) → generate contract → sign. At signature, the old quote, its cash flows, and its estimate get `addended_at = signing date` (**the seam**). Rule: old payments **≤ seam stay active** (real history for invoicing/loan tape); payments **after the seam are nullified**; the new schedule governs from there. Invariant: the active flows of a chain form **one timeline** — no month billed twice, no gap.

- Seam ruling: `docs/quote-generation/6-business-decisions-log.md` (2026-07-20)
- User procedure: `wiki/14-adendas.md`
- DB/code split rationale: infra `database/addendum-supersession.md`

## What triggered this work

Finanzas (Cynthia) reported "no adenda process exists" on 2026-08-11. Investigation showed the process existed and had been tried once — **810-07 El Machetón, 2026-07-21: 15 contract drafts, none signable.** Two silent failures: the grace anchor was filled with the *original* contract's 2025-10 start (payment 1 landed 10 months in the past) and the legal fee wasn't overridden to 0 (the engine re-computes legal every generation → re-charged Q2,098.21 the client had already paid). No UI surface said anything was wrong.

## Findings (each verified, not inferred)

1. **Sign-time supersession was three separate PostgREST writes**, each failure swallowed into `console.error` with "signing continues". A partial failure = quote stamped but flows alive (or inverse) = two live schedules, read as truth by every consumer. No error tracking exists, so nobody would know.
2. **Seam boundary disagreement**: canonical `v_cash_flows_all_states` used `addended_at > payment_date` (payment ON seam day = dead) while `mv_monthly_cash_flows_v1.is_nullified` used `payment_date > addended_at` (same row = alive). The ruling says alive.
3. **One weak test, invisible when failing**: the only invariant-ish dbt test checked payment_number uniqueness; the daily dbt CI had been red for 8+ consecutive days on unrelated test debt, so any new failure was indistinguishable noise.
4. **A second, legacy seam writer**: `ProjectSummary` had a browser-side "Addend" button stamping `addended_at` on a quote + ALL its flows with a user-picked date — sequential browser writes, no supersession pairing. Used once ever (2026-01-29).
5. **Contract modal traps** (the 810-07 killers): free-text grace anchor with no validation/preview; no warning when an adenda quote still carries legal costs.
6. **Zoho gap (NOT yet fixed — known open item)**: daily invoicing runs *inside* Zoho off the `cm_flujo_de_pagos` replica; nothing updates it on supersession, so an old schedule keeps invoicing after an adenda signs. Their own integration doc: "addendum semantics… undefined."

## What shipped

| Commit | Repo/branch | Contents |
|---|---|---|
| `23deb0a5` | infra/staging | `ContractPreGenerationModal`: payment-1 month preview; explicit-confirm gate on backdated grace anchors (resets on date change); amber warning when `addendum_number > 0` and `generation_parameters.legal_fee_override ≠ 0`; de-voseo'd help text |
| `f27517a1` | infra/staging | **`public.supersede_addendum_chain(bigint, date)`** — the three seam stamps in one transaction, `FOR UPDATE` on both quotes, idempotent, EXECUTE service_role only; **`public.signing_failures`** (durable failure records, 3 stages); **`public.v_addendum_chain_health`** (the invariant as a view — 4 issue kinds); SDK wrapper repointed to the RPC (signature unchanged); Lambda records failures + post-sign verification returning `chain_health_issues`; APD red banner (live query, self-clearing); legacy "Addend" button removed |
| `0836e89b` | infra/staging | Seam boundary `>` → `>=` in `v_cash_flows_all_states` (+ `inactive_reason` `<=` → `<`); deleted the never-wired `dbt-run.yml` stub |
| `b570c97` | dbt/main | `test_addendum_chain_health` — SELECT from the health view, any row fails; source registered |
| `8588c0f` | dbt/main | Invariant test as its own `if: always()` CI step (readable even while the general test step is red) |
| `3d07d13` | dbt/main | Nightly cron = tests only; full rebuild monthly (1st) + on dispatch |
| `00db2e5` | docs/main | `wiki/14-adendas.md` — the procedure, in Spanish, mapped to Finanzas' 9-step doc |

Both migrations (`2026-08-11-addendum-supersession-atomicity.sql`, `2026-08-11-cash-flows-seam-boundary-inclusive.sql`) are **applied to prod**. CHANGELOG entries exist for both. The deployed app bundle (`s3://albedo-quotes-app`, which deploys from staging pushes) contains the guardrails — verified in the served JS.

## Verify it yourself

```sql
-- Invariant view: empty = every chain coherent (baseline 2026-08-11: 0 rows)
SELECT * FROM v_addendum_chain_health;

-- RPC full-path test against real data, harmless (rolled back):
BEGIN;
SELECT supersede_addendum_chain(20, CURRENT_DATE);   -- 810-07 adenda: stamps quote 33884 + 53 flows + estimate
SELECT supersede_addendum_chain(20, CURRENT_DATE);   -- idempotent: {"ran": false, "reason": "already_superseded"}
ROLLBACK;

-- Boundary fix changed nothing retroactively:
SELECT COUNT(*) FROM monthly_cash_flows
WHERE addended_at IS NOT NULL AND addended_at = payment_date::timestamptz;  -- 0
```

```bash
cd dbt/main && ./venv/bin/dbt test --select test_addendum_chain_health   # PASS ~0.4s
```

Evidence for the 810-07 story: contracts 465–484 (all unsigned, `grace_period_start_date = 2025-10-01`); contract 484's `monthly_cash_flows` JSON has payment 1 = 2025-10-01 with `legal_costs = 2098.21`.

## Decisions to challenge (where review adds most value)

1. **Supersession writes moved into a Postgres function** despite the "logic in code, not DB" rule. Claimed justification: PostgREST can't express transactions; semantics stay documented in code; same exception class as the governance write RPCs. Disagree → the revert path is the old TS three-update implementation.
2. **Sign never aborts on supersession/materialization failure** (kept from the original design: the client signed on paper). Now recorded + surfaced instead of silent. Alternative view: a failed seam should block signing.
3. **APD banner reads the live view, not `signing_failures`** — self-clearing, no resolution bookkeeping. Cost: the failures table has no UI and relies on someone querying it.
4. **`double_billed_month` only counts rows with `monthly_payment > 0`** — deliberate, so a payment-0 down-payment row can share a month with the old schedule's last payment. Is that heuristic too loose?
5. **`seam_gap` only scans FK-linked chains** — the legacy chains (577-*, 981-*) without `replaces_quote_id` are not gap-checked until the linkage backfill happens.
6. **Legacy "Addend" button removed** rather than fixed — based on single-use-ever evidence.
7. **Seam boundary `>=`** — a payment exactly on the signing date counts as history (paid under the old contract). Check this matches finance's intuition.

## Open items (known, deliberate, unshipped)

- **Zoho flag-and-skip** (workstream 1): on supersession, flag post-seam `cm_flujo_de_pagos` rows; Deluge daily engine skips flagged; reconciliation lists flagged-but-invoiced for nota de crédito. Until then the handoff is manual.
- **dbt current-vs-original split** (workstream 3): the Debita loan tape reads the *original* quote's interest rate (`mart_debita_loan_tape_v2.sql:226`) — wrong the day an adenda restates a rate. 39 model files to classify.
- **Server-side validation** (workstream 5): grace-anchor and legal-override guards are frontend-only; the Lambda accepts anything.
- 15 stale El Machetón drafts not voided (`contracts.voided_on` exists but nothing reads/writes it); legacy chains lack FK linkage; the daily dbt test debt (8 failing tests) predates this work and is untriaged.
- Prod CDK env gets the new Lambda code at the next staging→main promotion; the app the team uses already has it (deploys from staging).
