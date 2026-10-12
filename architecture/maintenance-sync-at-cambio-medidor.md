# Maintenance rows at Cambio Medidor — design

**Status:** ruled 2026-10-12 (Jake with Javi). Ready to build once the V-rule
question in §8 is answered.
**Supersedes:** the 2026-10-09 draft of this file, which placed the seed on
the client-contract signing path. That trigger is dropped.
**Scope:** `public.project_maintenances` gets its rows when ops registers
the **Cambio Medidor** milestone on the ops page. Includes the milestone
catalog reorder requested in the same conversation.
**Not in scope:** the ops proposal for a visit date + derived statuses
(separate design), 577-01's through-2031 rows (awaiting the team).

## 1. Problem

`project_maintenances` has only ever been filled by one-off SQL seeds
(2026-04-21, 2026-05-27, 2026-10-09). Nothing runs when a project reaches
the point where maintenance should be scheduled, so projects go missing from
the Mantenimientos page until someone notices — ops did on 2026-05-27 and
again on 2026-10-08.

## 2. Decisions (2026-10-12)

1. **Trigger = Cambio Medidor.** The maintenance schedule is created when
   ops registers `fecha_cambio_medidor` in the milestones panel of the ops
   page. Not at client signing, not at Socio signing, not at Entrega
   Cliente. (Socio signing has no data: 0 of 75 partner contracts carry a
   signing date. Entrega Cliente is entered late: 1 of the 19 projects
   signed since June has it.)
2. **Confirm first.** Before the milestone saves, ops sees what will be
   created and confirms.
3. **Anchor = the Cambio Medidor date.** First maintenance 12 months later,
   one per contract year.
4. **Existing 230 projects keep their current dates.** Their rows stay
   anchored on Entrega Cliente / first payment. No re-anchor.
5. **Correcting the Cambio Medidor date moves the maintenance dates with it**
   while no maintenance is completed, and the UI says so while it happens.
6. **Accepted trade-off:** a project signed from now on does not appear on
   the Mantenimientos page until Cambio Medidor is registered.

## 3. The rule (single definition, in TypeScript)

Same three pieces as the 2026-10-09 seed, with the anchor swapped. After
this ships the TypeScript function is canonical; the SQL seeds are history.

| Piece | Rule |
|---|---|
| Who | the project's live quote is signed Financiado (not deleted, not addended; estimate not deleted) and the project has no `project_maintenances` rows |
| How many years | `1 .. ceil(max payment_number / 12)` of the live quote's cash flows; no end-of-schedule cap (ruled 2026-10-09) |
| Scheduled date | `fecha_cambio_medidor + year × 12 months` |
| Cost | cash-flow rows with `maintenance_cost ≥ 0.01`, ascending by `payment_number`; `first_paid_year = ceil(first charge's payment_number / 12)`; Nth charge → year `first_paid_year + N − 1`; other years `NULL` |
| Flags | `albedo_pays = cost ≥ 0.01`; `is_warranty_covered = NOT albedo_pays` |
| Currency | `estimates.project_currency` |
| Status | `Pendiente`, `actual_date` NULL |

Writes: one `INSERT … ON CONFLICT (project_id, maintenance_year) DO NOTHING`
with the service-role client. INSERT on the table is already
`service_role`-only (2026-06-22), which is exactly this caller.

## 4. Where it runs

Edge function `governance-milestones` (supabase repo), which already owns
every milestone write: it validates the V-rules, writes through the
`governance_register_milestone` RPC, and logs to `governance_audit_log`.

New shared module `supabase/functions/_shared/maintenance/seed.ts`
exporting:

- `planProjectMaintenances(supabase, projectId, anchorDate)` → the rows
  that would be created (or `{ skipped: reason }`), without writing. Reasons:
  `contado`, `not_signed`, `already_seeded`, `no_cash_flows`.
- `seedProjectMaintenances(supabase, projectId, anchorDate)` → writes the
  plan, returns the rows inserted.
- `reanchorProjectMaintenances(supabase, projectId, newAnchorDate)` → moves
  `scheduled_date` on every row that is not `Completado`; returns the count
  and the rows left untouched.

Two new `op` values on the function, plus a change to `register`/`modify`:

| Call | What happens |
|---|---|
| `op=preview_maintenances` (`project_id`, `fecha`) | returns the plan for the confirm dialog: number of years, first and last date, Albedo-pays years with cost and currency. Read-only. |
| `op=register`, `tipo_hito=fecha_cambio_medidor` | existing flow, then **seed**. Response gains `maintenances: { created: n, first: date, last: date }` or `{ skipped: reason }`. |
| `op=modify`, `tipo_hito=fecha_cambio_medidor`, new `fecha` | existing flow, then **re-anchor** if no row is `Completado`. Response gains `maintenances: { moved: n }` or `{ skipped: 'completed_rows', completed: n }`. |
| `op=modify` as *anulación* (fecha → null) | milestone flow only. Rows stay. Response says so. |

Ordering inside `register`: validate V-rules → RPC insert → audit log →
seed. A seed failure after the milestone is saved does **not** undo the
milestone: it returns `200` with `maintenances: { error }` so the UI can say
"hito registrado, pero el calendario de mantenimientos no se creó". The
failure is also written to `signing_failures` with
`stage = 'seed_project_maintenances'` (the table is the existing home for
"a business side-effect failed after the main write"; the name is historical).
The same function can be replayed by hand for that project.

Why the edge function and not a Postgres trigger on `project_milestones`:
the rule reads cash flows, estimates and the quote chain, and needs a
preview endpoint for the dialog. Business logic stays in TypeScript per the
house rule. There is exactly one write path for milestones, so one call site
covers it.

## 5. The ops page

`GovernanceTimelineWidget` (frontend `src/components/organisms/`), which
already has `handleRegister` and `handleModify` calling the edge function.

**Register, `fecha_cambio_medidor`:**

1. On "Registrar", call `op=preview_maintenances` first.
2. If the plan is `skipped` (Contado, already seeded, not signed): register
   as today, no dialog.
3. Otherwise open a confirm dialog:

   > **Crear calendario de mantenimientos**
   > Al registrar el cambio de medidor el DD/MM/AAAA se crearán N
   > mantenimientos para 2574-02:
   > · Primer mantenimiento: DD/MM/AAAA (12 meses después)
   > · Último: DD/MM/AAAA (año N)
   > · Años que paga Albedo: 2–5 (costo sin IVA: …)
   > Esta fecha también define las fechas de los siguientes años.
   > [Cancelar] [Registrar y crear mantenimientos]

4. On confirm, `op=register`. Toast: "Hito registrado · N mantenimientos
   creados".
5. If the response carries `maintenances.error`: toast "Hito registrado,
   pero el calendario de mantenimientos no se creó — avisa a sistemas."

**Modify, `fecha_cambio_medidor` with a new date:** the existing modify
modal (motivo required) gains one line when the project has rows:

> Al cambiar esta fecha se moverán las fechas de los N mantenimientos
> pendientes (primer mantenimiento: DD/MM/AAAA).

or, if any is completed:

> Este proyecto ya tiene mantenimientos completados; las fechas de los
> mantenimientos no se moverán.

Toast after save states what happened ("Hito modificado · 5 mantenimientos
reprogramados").

**Anulación:** modal line: "Los mantenimientos del proyecto se conservan."

All strings in `es.json` first, per the language rule.

## 6. Milestone catalog reorder

Requested in the same conversation. New `orden` after Fin Construcción:

| orden | Milestone | today |
|---|---|---|
| 5 | Recepción Póliza | 10 |
| 6 | Recepción Gestión Medidor | 8 |
| 7 | Cambio Medidor | 9 |
| 8 | Aprobación Calidad | 5 |
| 9 | Inicio Monitoreo | 6 |
| 10 | Entrega Cliente | 7 |

1–4 and 11–14 unchanged.

`orden` lives in four places that ship together:

1. frontend `src/governance/milestones.ts` (`MILESTONE_CATALOG`) — the panel
   renders from it;
2. `supabase/functions/_shared/governance/milestones.ts` — the mirror the
   edge function validates with;
3. infra `frontend/quotes-app/src/governance/milestones.ts` — the mirror
   kept during the frontend cutover;
4. table `public.milestone_catalog` — `v_project_milestone_stage` (estado)
   and `v_operations_projects` (próximo hito) read `orden` from it and
   follow a migration automatically.

Side effects of changing `orden`, both intended:

- **Estado** = highest-`orden` registered milestone that defines one. Today
  a project with Cambio Medidor shows "Medidor Bidireccional Instalado" even
  after Entrega Cliente; under the new order Entrega Cliente ranks higher.
- **Próximo hito** follows the new order.

## 7. Detection

New dbt singular test, nightly:

> projects with a registered `fecha_cambio_medidor`, a live signed Financiado
> quote, not annulled, and zero `project_maintenances` rows → must be empty.

Baseline today: to be measured before shipping (projects whose Cambio
Medidor predates this feature but that were seeded by the SQL batches will
have rows; any without rows are the backlog in §9).

## 8. Open: the V-rules

The sequence rules (spec §5, V1–V13) are preconditions keyed on milestone
*names*, not on `orden`, so the reorder does not change them by itself —
but two now contradict the new order:

| Rule | Today | Question for Javi |
|---|---|---|
| V9 | Recepción Póliza requires Aprobación Calidad | Póliza now comes before Calidad. What must precede Póliza — Fin Construcción? |
| V4 | Aprobación Calidad requires Fin Construcción | Should Calidad now require Cambio Medidor (and Póliza) first? |

V7 (Gestión Medidor requires Fin Construcción) and V8 (Cambio Medidor
requires Gestión Medidor) fit the new order. Projects signed before 2026
skip the V-rules entirely (ruling 2026-07-13), so the change only affects
new projects. The reorder and the rule changes ship in one PR set.

## 9. Rollout

1. **supabase** — `_shared/maintenance/seed.ts` + unit test that replays
   the 19 projects of the 2026-10-09 seed with their *first-payment* anchor
   and reproduces all 113 rows exactly (proves the TypeScript equals the
   SQL rule; the anchor is a parameter). `governance-milestones`: the three
   hooks in §4. Catalog `orden` in `_shared/governance/milestones.ts`.
2. **supabase** — migration: `milestone_catalog.orden` update; CHANGELOG
   entry in infra.
3. **frontend** — dialog + modify/anulación lines + strings; catalog mirror.
   Same change mirrored to the infra copy.
4. **dbt** — the guard test.
5. **docs** — wiki `13-mantenimientos.md` "Relación con otras entidades"
   (still says lease payments); a wiki section for the dialog.
6. **Backlog:** projects that already have Cambio Medidor registered but no
   rows (if any — measure). Seed them with the function, anchored on their
   Cambio Medidor date.

Estimate: one day for 1–2, half a day for 3, an hour each for 4–5.
