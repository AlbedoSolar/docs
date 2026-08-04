# Reads go through section views

**Status:** agreed 2026-08-04 (Jake). Adopted incrementally, not big-bang.

## The rule

> Every app read goes through a view named `v_<section>`. Tables are for writes, by id.

That's the whole rule. Everything below is how to apply it and what it costs.

## Why

Correctness filters currently live at the call site, which means every call site
has to remember them. They don't.

The case that forced this: `estimates.deleted_at` had existed for a long time and
51 rows carried it, but only 12 of 30 read paths honoured it. Project 1037-01
rendered 13 estimates on the project page when exactly 1 was live; 2449-01
rendered 9 of 2. Both the project page and wizard step 3 read through the same
function, so the sales team saw it daily. Fixing it meant hand-auditing 30 call
sites and making a judgment call on each — with no guarantee the next feature
doesn't add a 31st.

In a view, `deleted_at IS NULL` is one line nobody can forget.

The same shape applies to the signed filter, the original-vs-latest quote choice,
and the IVA fallback — all three already flagged as drift risks in the
2026-07-24 dbt audit.

## What it looks like

Before:

```ts
.from('estimates')
.select('*, project_phases(...), estimate_vendors(...), estimate_affiliates(...)')
.eq('project_id', projectId)
.is('deleted_at', null)          // every call site must remember this
```

After:

```ts
.from('v_project_estimates').eq('project_id', projectId)
```

## Scope

- **One view per section, not per table.** Several views may read the same base
  tables — expect a few built on `projects` alone (project page, operations,
  finances). That's fine and expected.
- **Writes are unchanged.** They go to tables, keyed by id. Views fix read
  consistency; they do not fix write consistency, and they are not expected to.
- **Convenience reads may keep hitting tables.** Lookups by id
  (`getEstimateById`, reference lookups) buy nothing from a view and cost the
  PostgREST embedding problem below. Leave them.

Already following this pattern: `v_active_projects`, `v_operations_projects`,
`v_estimate_calcs`, `v_offer_sheet_phases`. The change is making it the default
rather than the exception.

## Conventions

1. **Name it `v_<section>`.** `ian_*` is reserved for analysis views that the app
   does not read.
2. **`security_invoker = true`,** stated explicitly in the migration. Definer only
   with a written reason in the same file. See the RLS note below for why this
   matters even though RLS is currently permissive.
3. **`GRANT SELECT ... TO authenticated`** — not `PUBLIC`. Anon access is granted
   only to views that a public route actually needs, and only after checking what
   columns it exposes.
4. **`EXPLAIN` it against production row counts before it ships.** See the index
   note below.
5. **Ship it as a `CREATE OR REPLACE VIEW` migration** with a CHANGELOG entry,
   like any other database change.

## Deferred: restrictive read RLS

Decided 2026-08-04. Read RLS stays permissive for now. The team is small, most
people should be able to see most things, and the UI is what boxes people into
their area.

**Revisit when access broadens** — new roles, external/partner logins, or anyone
who should not see all financial data. At that point the work is writing policies,
not rewriting views, *provided* views were created with `security_invoker = true`.
A definer view runs as its owner and bypasses RLS on its base tables entirely, so
a definer view built today would have to be rebuilt then. Setting the flag now is
free; changing twenty views later is not.

This deferral covers **authenticated** reads only. It is not a decision to leave
data open to `anon` — see the separate anon exposure on `clients`, `projects`,
`quotes`, `estimates` and `users`, which predates this document and is tracked
independently.

## Two costs to know about

### PostgREST embedding

PostgREST builds nested selects from foreign keys, and **views have no foreign
keys**. `getEstimatesByProjectId` currently selects a ~60-line nested tree
(`estimates(*, project_phases(...), estimate_vendors(..., user_codes(...)),
estimate_affiliates(..., affiliates(...)))`). Pointed at a view, those embeds stop
resolving.

Options are flattening the tree into the view, or declaring computed
relationships. Both are per-view work, not one-time setup. This is the main reason
section views should be flat and wide rather than deeply nested, and the main
reason embed-target tables keep their direct reads.

### Views hide missing indexes

When the query is in the code, a slow page gives you something to read. When it's
`.from('v_active_projects')`, the joins are invisible at the call site — a missing
index surfaces as "the page is slow" with nothing to look at.

Two multipliers: views get reused, so one bad join slows every section reading
through it; and stacked views can stop the planner pushing predicates down, so a
`WHERE` gets applied after a scan instead of before.

Precedent: `v_active_projects` ran 8.8s until `estimates(project_id)` was indexed,
then 0.2s — 44x, from one index nobody could see from the frontend.

As of 2026-08-04 there are 15 unindexed foreign keys on hot tables. The ones most
likely to matter as views take over: `quote_offers.estimate_id`,
`quote_offers.project_id`, `estimates.user_code_id`, `estimates.provider_id`,
`projects.site_id`.

## Enforcement

The rule is only worth writing down if the wrong thing gets caught. In order of
how little it depends on anyone remembering:

1. **CI check** — fail the build when code reads a table that a section view
   already covers, with an explicit allowlist for by-id lookups and embed targets.
   The allowlist is the useful artifact: every entry becomes a documented
   exception instead of an accident. Not yet built.
2. **This document, referenced from CLAUDE.md** — phrased as a named artifact
   (`v_<section>` is the read path) rather than a principle, because the rules
   that stick in this codebase are the ones with something to point at:
   `DataTable`, `QuoteWizardStep3Estimate`, `computeRetailPreviewFromPhases`.
3. **`REVOKE SELECT` on the base table** once *every* read for it has moved. Then
   a raw read fails immediately rather than shipping. Only viable for leaf tables —
   `estimates` can't be revoked while it is an embed target and a by-id lookup.

**The middle state is the risk.** Half-migrated means two conventions coexist and
you cannot tell which one a given read follows by looking at it. The CI allowlist
is what makes the middle state legible, which is why it should land early rather
than last.
