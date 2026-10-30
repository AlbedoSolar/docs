# dbt Tests Register

Every automated check that runs against production data in the `dbt` repo, one line each, with a link to its SQL. The tests are the nightly's eyes: when one fails, the nightly run goes red and someone has to look. This file says, for each test, **what it checks, why it exists, what to do when it fails**, and where its code is.

**When they run.** `.github/workflows/dbt-run.yml` runs `dbt test` every night at 06:00 UTC (in practice 10:00–12:30) after rebuilding the impact chain and the finance tables; the chain-health and EEFFs tests run again as their own steps. The monthly run on the 1st does the same after a full rebuild.

**Severity.** *error*: a row returned fails the build (red nightly). *warn*: the rows are listed in the log and the build stays green; a warning means "look at this list", not "something is broken". Every warn says in its header when it should become an error.

**Reading a failure.** The log shows `FAIL n test_name` (or `WARN n`). Run `dbt test --select test_name --store-failures` or open the compiled SQL to see the rows; the output columns are designed for triage (they name the project, quote or invoice and, where the test has several checks, which one fired). A red nightly is not always a data bug: check for timeouts first (`main/OPERATIONS.md`).

**Keeping this file current.** A test is not done until it has a row here. The rule lives in the dbt repo's `CLAUDE.md`: a new or changed singular test ships with its register row and entry in the same PR, and the entry's "baseline" is the count and date at creation. The `dbt` repo is private: the links open for team members only.


## Register

### Finance invariants

| Test | Checks that | Severity | Entry |
|---|---|---|---|
| [test_cash_flow_schedule_shape](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/finance_musts/test_cash_flow_schedule_shape.sql) | Native financed schedules have the shape of Jake's reconstruction sheet: enganche and gastos legales only on row 0; row-0 balance = retail sin IVA − enganche; every later balance = prior + interest − cuota − compra de activos (grace months capitalize); compra de activos only on the last row at 1 % of retail; one constant cuota after grace; seguro and mantenimiento constant on exactly the cuota rows. | error | [entry](#t-cash-flow-schedule-shape) |
| [test_month_0_closing_row](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/finance_musts/test_month_0_closing_row.sql) | On the row-0 closing row at signing, principal = enganche and interest = 0. | error | [entry](#t-month-0-closing-row) |
| [test_principal_and_interest_equal_monthly_payment](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/finance_musts/test_principal_and_interest_equal_monthly_payment.sql) | Every principal-bearing row of a schedule fits ONE payment identity: QB-era (down + asset transfer + cuota = principal + interest) or sign-time (cuota = principal + interest, asset transfer separate on the last row). | error | [entry](#t-principal-and-interest-equal-monthly-payment) |
| [test_interest_payments_match](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/finance_musts/test_interest_payments_match.sql) | Every row of a schedule fits ONE interest convention: flat monthly (principal × annual / 12) or compound daily proration (prior × ((1 + annual)^(days/365) − 1), stub periods, grace capitalizes). | error | [entry](#t-interest-payments-match) |
| [test_correctly_subtracting_principal](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/finance_musts/test_correctly_subtracting_principal.sql) | Remaining principal never increases from one payment to the next within a single quote's schedule (LAG partitioned by quote). | error | [entry](#t-correctly-subtracting-principal) |
| [test_remaining_principal_is_zero_at_end_of_loan](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/finance_musts/test_remaining_principal_is_zero_at_end_of_loan.sql) | The last scheduled payment of each project's CURRENT quote ends at remaining principal 0. | error | [entry](#t-remaining-principal-is-zero-at-end-of-loan) |
| [test_total_income_matches](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/finance_musts/test_total_income_matches.sql) | Across the whole adenda chain, what the client pays equals what was sold (the chain-root estimate's retail) plus interest, under either engine convention (tolerance 0.5). Negotiated restructures that change the total are listed in `seeds/negotiated_chain_adjustments.csv`. | error | [entry](#t-total-income-matches) |
| [test_all_financed_projects_have_interest](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/finance_musts/test_all_financed_projects_have_interest.sql) | Every signed Financiado project has non-zero interest across its cash flows. | error | [entry](#t-all-financed-projects-have-interest) |

### Accounts receivable

| Test | Checks that | Severity | Entry |
|---|---|---|---|
| [test_ar_principal_totals](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/accounts_receivable/test_ar_principal_totals.sql) | The built AR table (`int_official_accounts_receivable_by_project_gtq`) agrees row-for-row with an independent re-aggregation of its live inputs: short_term(project, month) = principal due in the next 12 months, with the model's own eligibility gate. | error | [entry](#t-ar-principal-totals) |

### Canonical parity

| Test | Checks that | Severity | Entry |
|---|---|---|---|
| [test_app_cash_flow_view_matches_tape](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/canonical_parity/test_app_cash_flow_view_matches_tape.sql) | `app.mv_monthly_cash_flows_v1` (what contracts, the signed page and Zoho read) neither invents nor loses money vs the stored tape: pass-through rows match column for column (0.005), split quotes keep per-quote cash sums, derived totals equal the sum of components. | error | [entry](#t-app-cash-flow-view-matches-tape) |
| [test_project_date_matches_canonical](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/canonical_parity/test_project_date_matches_canonical.sql) | `int_original_quotes.project_date` equals `public.v_project_signing.project_date` for every chain-root quote. | error | [entry](#t-project-date-matches-canonical) |
| [test_stg_cash_flows_match_canonical](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/canonical_parity/test_stg_cash_flows_match_canonical.sql) | Every row of `stg_signed_monthly_cash_flows` exists in canonical `v_cash_flows`. | error | [entry](#t-stg-cash-flows-match-canonical) |
| [test_stg_signed_projects_matches_canonical](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/canonical_parity/test_stg_signed_projects_matches_canonical.sql) | dbt's signed-project set (`stg_signed_quotes`) equals the canonical layer's (inlined in five `v_*` views). | error | [entry](#t-stg-signed-projects-matches-canonical) |

### Data health

| Test | Checks that | Severity | Entry |
|---|---|---|---|
| [test_data_consistency_invariants](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/data_health/test_data_consistency_invariants.sql) | Hard invariants from the July 2026 data review: a signed quote carries a signing date (A1), retail > 0 (A4), at most one signed original quote per project (B3). | error | [entry](#t-data-consistency-invariants) |
| [test_electricity_price_in_plausible_band](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/data_health/test_electricity_price_in_plausible_band.sql) | Signed projects whose electricity price, in USD, is outside 0.08–0.60 per kWh. | warn (flip to error once infra #491 is closed) | [entry](#t-electricity-price-in-plausible-band) |
| [test_every_signed_estimate_has_escalation_rate](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/data_health/test_every_signed_estimate_has_escalation_rate.sql) | Every signed estimate with panels resolves an electricity price inflation rate inside the sane band. | error | [entry](#t-every-signed-estimate-has-escalation-rate) |
| [test_country_carbon_assumptions_in_band](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/data_health/test_country_carbon_assumptions_in_band.sql) | `countries` carries the four CO2 inputs (gross grid factor, panel lifecycle factor, decarbonization rate, base year) and each sits in a plausible band. | error | [entry](#t-country-carbon-assumptions-in-band) |
| [test_rural_area_follows_gps](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/data_health/test_rural_area_follows_gps.sql) | Every signed project's `rural_area` in the impact mart equals the GPS rule on the site's density code; the manual flag is never used. | error | [entry](#t-rural-area-follows-gps) |
| [test_impact_flags_match_canonical_view](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/data_health/test_impact_flags_match_canonical_view.sql) | The six impact indicators in the impact summary equal the one home, `v_project_impact_flags`; no report reads the legacy project flags. | error | [entry](#t-impact-flags-match-canonical-view) |
| [test_signed_clients_have_impact_flags](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/data_health/test_signed_clients_have_impact_flags.sql) | Work queue: signed projects whose client has no Sí/No answer for women-led, youth-led, educational or non-profit. | warn | [entry](#t-signed-clients-have-impact-flags) |
| [test_vi_base_matches_impact_mart](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/data_health/test_vi_base_matches_impact_mart.sql) | The Valuing Impact input table agrees row for row with the impact mart on rural, kW and net savings, and stays inside VI's vocabulary. | error | [entry](#t-vi-base-matches-impact-mart) |
| [test_panel_degradation_in_range](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/data_health/test_panel_degradation_in_range.sql) | Solar panels have `annual_depreciation_percentage` in 0–0.02 and `first_year_production_loss_percentage` in 0–0.10 (fractions). | error | [entry](#t-panel-degradation-in-range) |
| [test_signed_projects_have_sector](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/data_health/test_signed_projects_have_sector.sql) | Every signed project has a `project_sector_id`. | error | [entry](#t-signed-projects-have-sector) |
| [test_backup_tables_expired](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/data_health/test_backup_tables_expired.sql) | Lists safety-copy tables (`_backup_YYYYMMDD_*`, `*_backup_YYYYMMDD`, `*_snapshot_YYYYMMDD`) older than 30 days. | warn | [entry](#t-backup-tables-expired) |
| [test_zoho_sync_ran_last_night](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/data_health/test_zoho_sync_ran_last_night.sql) | The nightly Zoho → Supabase sync ran: the newest run ended `completed` or `partial` and started within the last 25 hours. | error | [entry](#t-zoho-sync-ran-last-night) |

### EEFFs reports

| Test | Checks that | Severity | Entry |
|---|---|---|---|
| [test_eeffs_signed_projects_match_reports](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/eeffs/test_eeffs_signed_projects_match_reports.sql) | Every signed project is in the EEFFs report(s) it belongs to (Financiado → by_financed_project, non-GT also HN_ES_USD; Contado → by_paid_up_front_project), and nothing else is. | error | [entry](#t-eeffs-signed-projects-match-reports) |
| [test_eeffs_amounts_match_sources](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/eeffs/test_eeffs_amounts_match_sources.sql) | Per project, the report's flow rows summed across its month columns equal the cash-flow table for the same window (Jan-21 to Dec-30), and Precio Retail / Costo Proyecto equal the original estimate (tolerance 1.00). Covers Ingresos por Mantenimiento, and Pago Total includes maintenance, since 2026-10-29. | error | [entry](#t-eeffs-amounts-match-sources) |
| [test_eeffs_report_shape](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/eeffs/test_eeffs_report_shape.sql) | Exactly one row per (project, account), the 16 known accounts per project (13 until 2026-10-29; the three maintenance accounts were added then), no blank keys, no Guatemala rows in the HN/SV USD report. | error | [entry](#t-eeffs-report-shape) |
| [test_eeffs_signed_original_quote_sale_type_known](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/eeffs/test_eeffs_signed_original_quote_sale_type_known.sql) | Every signed project has exactly one original quote whose `sale_type` is exactly Financiado or Contado. | error | [entry](#t-eeffs-signed-original-quote-sale-type-known) |

### Adendas and signing

| Test | Checks that | Severity | Entry |
|---|---|---|---|
| [test_addendum_chain_health](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_addendum_chain_health.sql) | Active cash flows across an adenda chain form one timeline: no month billed twice, no gap at the seam, no half-fired supersession, no signed quote without a schedule. The rule lives in `public.v_addendum_chain_health`; the test asks "any rows?". | error | [entry](#t-addendum-chain-health) |
| [test_addendum_chain_linkage](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_addendum_chain_linkage.sql) | When an addended quote still has live pre-seam rows, its live successor carries `original_quote_id` (else the chain numbers as two sequences restarting at 1); no link points outside the project's signed set. | error | [entry](#t-addendum-chain-linkage) |
| [test_signing_failures_unresolved](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_signing_failures_unresolved.sql) | No row in `public.signing_failures` sits without `resolved_at`. | error | [entry](#t-signing-failures-unresolved) |
| [test_one_non_addended_flow_per_month_per_project](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_one_non_addended_flow_per_month_per_project.sql) | Each project has exactly one active (non-addended) cash flow per payment number. | error | [entry](#t-one-non-addended-flow-per-month-per-project) |
| [test_quotes_accounted_for](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_quotes_accounted_for.sql) | Every signed quote has cash-flow rows. | error | [entry](#t-quotes-accounted-for) |
| [test_all_quotes_have_contract_signing_date](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_all_quotes_have_contract_signing_date.sql) | Every signed quote has a `contract_signing_date`. | error | [entry](#t-all-quotes-have-contract-signing-date) |
| [test_no_future_contract_signing_dates](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_no_future_contract_signing_dates.sql) | No quote has a signing date in the future. | error | [entry](#t-no-future-contract-signing-dates) |
| [test_cash_flow_sanity_checks](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_cash_flow_sanity_checks.sql) | Contado schedules have no down payment and at most two payments unless every row carries a `payment_milestone` label; financed payment numbers don't skip; every quote's flows belong to one estimate. | error | [entry](#t-cash-flow-sanity-checks) |
| [test_data_consistency](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_data_consistency.sql) | Estimates point at the project their reference names; quote `sale_type` is Financiado or Contado. | error | [entry](#t-data-consistency) |
| [test_interest_rates_match_dbt_calculations](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_interest_rates_match_dbt_calculations.sql) | `quotes.monthly_interest_rate` equals the rate dbt derives in `stg_calculated__interest_rates` for the same project. | error | [entry](#t-interest-rates-match-dbt-calculations) |
| [test_exchange_rates_gap_free](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_exchange_rates_gap_free.sql) | The GTQ exchange-rate series has no missing date between its start and end, and the newest rate is at most 3 days old. | error | [entry](#t-exchange-rates-gap-free) |

### Maintenance

| Test | Checks that | Severity | Entry |
|---|---|---|---|
| [test_maintenances_exist_after_cambio_medidor](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_maintenances_exist_after_cambio_medidor.sql) | A signed Financiado project with the Cambio Medidor milestone registered has `project_maintenances` rows. | error | [entry](#t-maintenances-exist-after-cambio-medidor) |
| [test_maintenances_only_on_live_financiado](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_maintenances_only_on_live_financiado.sql) | Maintenance rows exist only on non-deleted projects with a live signed Financiado quote. | error | [entry](#t-maintenances-only-on-live-financiado) |
| [test_maintenance_years_contiguous](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_maintenance_years_contiguous.sql) | Each project's schedule is years 1..N with no gaps. | error | [entry](#t-maintenance-years-contiguous) |
| [test_maintenance_dates_share_one_anchor](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_maintenance_dates_share_one_anchor.sql) | `scheduled_date(N) = scheduled_date(1) + (N − 1) × 12 months` for every row of a project. | error | [entry](#t-maintenance-dates-share-one-anchor) |
| [test_maintenance_flags_consistent](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_maintenance_flags_consistent.sql) | `albedo_pays = cost_project_currency >= 0.01` and `is_warranty_covered = NOT albedo_pays`. | error | [entry](#t-maintenance-flags-consistent) |
| [test_maintenance_status_matches_actual_date](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_maintenance_status_matches_actual_date.sql) | `status = Completado` iff `actual_date` is set; only Pendiente / Completado are stored (Vencido is computed at read time). | error | [entry](#t-maintenance-status-matches-actual-date) |
| [test_maintenance_actual_date_not_future](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_maintenance_actual_date_not_future.sql) | `actual_date` (fecha real) is never in the future. | warn | [entry](#t-maintenance-actual-date-not-future) |

### Zoho sync

| Test | Checks that | Severity | Entry |
|---|---|---|---|
| [test_zoho_invoice_status_words](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/zoho_sync/test_zoho_invoice_status_words.sql) | `zoho_invoices.invoice_status` holds only Closed / Open / Overdue / Void / Draft. | error | [entry](#t-zoho-invoice-status-words) |
| [test_zoho_invoice_balance_fits_status](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/zoho_sync/test_zoho_invoice_balance_fits_status.sql) | For sync-sourced invoices: Closed ⇒ nothing left to pay; Open/Overdue ⇒ something left; 0 ≤ balance ≤ total; Overdue ⇒ due date passed; Open ⇒ due date not passed (2 days slack). | error | [entry](#t-zoho-invoice-balance-fits-status) |
| [test_zoho_invoice_lines_add_up](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/zoho_sync/test_zoho_invoice_lines_add_up.sql) | An invoice's lines (before tax) equal its total, either as-is or grossed up by one country's IVA (12/13/15 %), tolerance 0.10. | error | [entry](#t-zoho-invoice-lines-add-up) |
| [test_zoho_payments_not_duplicated](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/zoho_sync/test_zoho_payments_not_duplicated.sql) | The same Zoho payment is not stored twice against the same invoice. | error | [entry](#t-zoho-payments-not-duplicated) |
| [test_zoho_payments_point_at_an_invoice](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/zoho_sync/test_zoho_payments_point_at_an_invoice.sql) | Every sync-written payment names an invoice that exists in `zoho_invoices` ("Saldo de inicio del cliente" excepted: it is the opening balance, not an invoice). | error | [entry](#t-zoho-payments-point-at-an-invoice) |
| [test_zoho_project_links_agree](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/zoho_sync/test_zoho_project_links_agree.sql) | The three copies of "which project does this invoice belong to" (`invoice_project_map`, `zoho_invoices`, `lease_payments`) agree for sync-written rows, and the invoice's project exists with the right reference. | error | [entry](#t-zoho-project-links-agree) |
| [test_zoho_invoices_without_project](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/zoho_sync/test_zoho_invoices_without_project.sql) | Lists invoices the sync read in full but could not link to a project (void excluded). | warn | [entry](#t-zoho-invoices-without-project) |
| [test_zoho_payments_cover_amount_paid](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/zoho_sync/test_zoho_payments_cover_amount_paid.sql) | Stored payments (applied, or applied + tax withheld) are not short of Zoho's `amount_paid` (cash received) by more than 10 in the invoice's currency. | warn | [entry](#t-zoho-payments-cover-amount-paid) |
| [test_zoho_invoice_settlement_adds_up](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/zoho_sync/test_zoho_invoice_settlement_adds_up.sql) | For sync-sourced invoices whose detail was read: `total − balance` = cash received + credits applied + written off + tax withheld, within 1.00. A zero balance is not proof of cash. | warn | [entry](#t-zoho-invoice-settlement-adds-up) |
| [test_zoho_credit_applications_add_up](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/zoho_sync/test_zoho_credit_applications_add_up.sql) | The credit-note applications stored for an invoice (`zoho_credit_note_invoices`) add up to the credits Zoho reports on the invoice (`credits_applied`), within 0.10. | warn | [entry](#t-zoho-credit-applications-add-up) |
| [test_zoho_snapshot_is_current](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/zoho_sync/test_zoho_snapshot_is_current.sql) | Every `zoho_invoices` row has exactly one open version in `snapshots.snapshot_zoho_invoices`, and that version's check columns (status, balance, total, dates, cash, credits, write-off) equal the table. | error | [entry](#t-zoho-snapshot-is-current) |
| [test_zoho_snapshot_one_open_version_per_invoice](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/zoho_sync/test_zoho_snapshot_one_open_version_per_invoice.sql) | No invoice has two open versions or two versions that overlap in time in `snapshots.snapshot_zoho_invoices`. | error | [entry](#t-zoho-snapshot-one-open-version-per-invoice) |

### Generic column tests (schema.yml)

Declared in [`main/models/models_on_static_tables/schema.yml`](https://github.com/AlbedoSolar/dbt/blob/main/main/models/models_on_static_tables/schema.yml); dbt generates the SQL. All severity error.

| Model | Column | Test |
|---|---|---|
| `int_estimates_convert_currencies` | `id` | not_null |
| `int_estimates_convert_currencies` | `project_reference` | not_null |
| `int_monthly_cash_flows_convert_currencies` | `id` | not_null |
| `int_monthly_cash_flows_convert_currencies` | `quote_id` | not_null |
| `int_monthly_cash_flows_convert_currencies` | `payment_date` | not_null |
| `int_monthly_cash_flows_convert_currencies` | `payment_number` | not_null |
| `int_production_start_dates` | `quote_id` | not_null, unique |
| `int_production_start_dates` | `estimate_id` | not_null |
| `int_production_start_dates` | `project_id` | not_null |
| `int_production_start_dates` | `production_start_date` | not_null |
| `int_official_accounts_receivable_by_project_gtq` | `project_reference` | not_null |
| `int_official_accounts_receivable_by_project_gtq` | `month_start` | not_null |
| `mart_official_financial_projections` | `month_start` | not_null |
| `mart_official_historic_accounting_report` | `project_reference` | not_null |

---

## Entries

## Finance invariants

<a id="t-cash-flow-schedule-shape"></a>
### test_cash_flow_schedule_shape

**Checks that.** Native financed schedules have the shape of Jake's reconstruction sheet: enganche and gastos legales only on row 0; row-0 balance = retail sin IVA − enganche; every later balance = prior + interest − cuota − compra de activos (grace months capitalize); compra de activos only on the last row at 1 % of retail; one constant cuota after grace; seguro and mantenimiento constant on exactly the cuota rows.

**Why it exists.** Built from Jake's "cash flow simple.xlsx" (2026-09-25); one labelled row per offending quote/payment so triage reads like the sheet's FALSE cells. Interest is taken as given (two conventions are another test's job).

**When it fails.** Read the label: it names the sheet column that disagrees. Repair the tape row via a migration; if the engine wrote it, fix the engine first (see the 2026-09-25 tape repairs).

**Severity.** error. **Baseline.** 2026-09-25: 0 rows after the tape repairs. **Related.** Decisions log 2026-08-27, 2026-09-25.

**Source.** [`main/tests/tests_on_static_tables/finance_musts/test_cash_flow_schedule_shape.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/finance_musts/test_cash_flow_schedule_shape.sql)

<a id="t-month-0-closing-row"></a>
### test_month_0_closing_row

**Checks that.** On the row-0 closing row at signing, principal = enganche and interest = 0.

**Why it exists.** The enganche counts entirely as principal and no time has passed at signing. The sign-time materializer once wrote principal 0 on row 0 (fixed in supabase#28, 2026-09-25).

**When it fails.** Repair the tape row (pattern: infra migration 2026-09-25-month-0-principal-equals-enganche.sql) and check the materializer.

**Severity.** error. **Baseline.** 2026-09-25: 51 month-0 rows, 0 offenders. **Related.** Decisions log 2026-09-25 "No interest at month 0".

**Source.** [`main/tests/tests_on_static_tables/finance_musts/test_month_0_closing_row.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/finance_musts/test_month_0_closing_row.sql)

<a id="t-principal-and-interest-equal-monthly-payment"></a>
### test_principal_and_interest_equal_monthly_payment

**Checks that.** Every principal-bearing row of a schedule fits ONE payment identity: QB-era (down + asset transfer + cuota = principal + interest) or sign-time (cuota = principal + interest, asset transfer separate on the last row).

**Why it exists.** Two engines wrote signed cash flows with different conventions; both are legitimate. A schedule that fits neither is a real defect.

**When it fails.** Output names the quote and the failing-row counts per convention. Repair the rows that break the convention the rest of the schedule follows.

**Severity.** error. **Baseline.** 2026-08-25 census: 13,214 principal rows, all fit one convention. **Related.** Mirrors test_interest_payments_match.

**Source.** [`main/tests/tests_on_static_tables/finance_musts/test_principal_and_interest_equal_monthly_payment.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/finance_musts/test_principal_and_interest_equal_monthly_payment.sql)

<a id="t-interest-payments-match"></a>
### test_interest_payments_match

**Checks that.** Every row of a schedule fits ONE interest convention: flat monthly (principal × annual / 12) or compound daily proration (prior × ((1 + annual)^(days/365) − 1), stub periods, grace capitalizes).

**Why it exists.** Quote creation and the XIRR recalculation use different conventions; both exist in production (173 flat, 40 compound on 2026-08-18). Documented lump collections confirmed by Finanzas are excluded while they keep their documented value (`seeds/documented_lump_rows.csv`).

**When it fails.** Output gives bad-row counts per convention and the implied vs stored rate. A stored rate that disagrees with the schedule, a hand-crafted adenda, or an import artifact: fix the data, or document the lump row in the seed with Finanzas' sign-off.

**Severity.** error. **Baseline.** 2026-08-18 census; 0 rows on 2026-10-16. **Related.** Decisions log 2026-09-23 (interest accrued).

**Source.** [`main/tests/tests_on_static_tables/finance_musts/test_interest_payments_match.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/finance_musts/test_interest_payments_match.sql)

<a id="t-correctly-subtracting-principal"></a>
### test_correctly_subtracting_principal

**Checks that.** Remaining principal never increases from one payment to the next within a single quote's schedule (LAG partitioned by quote).

**Why it exists.** Adenda chains put two quotes' rows in one project; principal is only monotonic within one schedule.

**When it fails.** A balance that rises mid-schedule outside grace is a data edit; repair the row.

**Severity.** error. **Baseline.** 0 rows.

**Source.** [`main/tests/tests_on_static_tables/finance_musts/test_correctly_subtracting_principal.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/finance_musts/test_correctly_subtracting_principal.sql)

<a id="t-remaining-principal-is-zero-at-end-of-loan"></a>
### test_remaining_principal_is_zero_at_end_of_loan

**Checks that.** The last scheduled payment of each project's CURRENT quote ends at remaining principal 0.

**Why it exists.** An addended quote's last live row legitimately carries a balance (its pre-seam history stays live); only the current quote must amortize fully.

**When it fails.** Check the final row of the current quote; a non-zero balance means a truncated schedule or a wrong final row.

**Severity.** error. **Baseline.** 0 rows.

**Source.** [`main/tests/tests_on_static_tables/finance_musts/test_remaining_principal_is_zero_at_end_of_loan.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/finance_musts/test_remaining_principal_is_zero_at_end_of_loan.sql)

<a id="t-total-income-matches"></a>
### test_total_income_matches

**Checks that.** Across the whole adenda chain, what the client pays equals what was sold (the chain-root estimate's retail) plus interest, under either engine convention (tolerance 0.5). Negotiated restructures that change the total are listed in `seeds/negotiated_chain_adjustments.csv`.

**Why it exists.** Chain-aware since Jake's 2026-08-28 ruling: a mid-life adenda is healthy chain-wide even though its post-seam schedule alone does not match retail.

**When it fails.** Either the chain over/under-collects without a documented reason (fix the schedule) or a negotiated change is missing from the seed (add it with the ruling reference).

**Severity.** error. **Baseline.** 0 rows on 2026-08-28. **Related.** Decisions log 2026-08-25 (1499-01), 2026-09-01 (810-07), 2026-09-29 (1037-01).

**Source.** [`main/tests/tests_on_static_tables/finance_musts/test_total_income_matches.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/finance_musts/test_total_income_matches.sql)

<a id="t-all-financed-projects-have-interest"></a>
### test_all_financed_projects_have_interest

**Checks that.** Every signed Financiado project has non-zero interest across its cash flows.

**Why it exists.** A financed schedule with zero interest is an import or materialization error.

**When it fails.** Regenerate or repair the schedule.

**Severity.** error. **Baseline.** 0 rows.

**Source.** [`main/tests/tests_on_static_tables/finance_musts/test_all_financed_projects_have_interest.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/finance_musts/test_all_financed_projects_have_interest.sql)

## Accounts receivable

<a id="t-ar-principal-totals"></a>
### test_ar_principal_totals

**Checks that.** The built AR table (`int_official_accounts_receivable_by_project_gtq`) agrees row-for-row with an independent re-aggregation of its live inputs: short_term(project, month) = principal due in the next 12 months, with the model's own eligibility gate.

**Why it exists.** The AR models are tables and nothing rebuilds them automatically. On 2026-08-27 they were rebuilt from a months-stale intermediate; nothing noticed. Earlier versions asserted a floor that was false in three legitimate shapes.

**When it fails.** A stale table: `dbt run --select int_monthly_cash_flows_month_0_adjustment_converted_currencies+`. Otherwise a join fan-out or gate drift in the model.

**Severity.** error. **Baseline.** v3 2026-08-28: 0 rows. **Related.** dbt `main/OPERATIONS.md` (tables go stale).

**Source.** [`main/tests/tests_on_static_tables/accounts_receivable/test_ar_principal_totals.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/accounts_receivable/test_ar_principal_totals.sql)

## Canonical parity

<a id="t-app-cash-flow-view-matches-tape"></a>
### test_app_cash_flow_view_matches_tape

**Checks that.** `app.mv_monthly_cash_flows_v1` (what contracts, the signed page and Zoho read) neither invents nor loses money vs the stored tape: pass-through rows match column for column (0.005), split quotes keep per-quote cash sums, derived totals equal the sum of components.

**Why it exists.** The view's definition lives in two homes (dbt model and infra migration) that must stay byte-equivalent; the day they diverge in a way that changes numbers, this fails.

**When it fails.** Diff the two definitions; the failing class (pass_through / sum / derived_total) says where.

**Severity.** error. **Baseline.** 0 rows at v5 (2026-09-25). **Related.** Decisions log 2026-09-23/25, 2026-09-30.

**Source.** [`main/tests/tests_on_static_tables/canonical_parity/test_app_cash_flow_view_matches_tape.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/canonical_parity/test_app_cash_flow_view_matches_tape.sql)

<a id="t-project-date-matches-canonical"></a>
### test_project_date_matches_canonical

**Checks that.** `int_original_quotes.project_date` equals `public.v_project_signing.project_date` for every chain-root quote.

**Why it exists.** The one `project_date` rule is re-implemented in dbt for scan-cost reasons; this keeps the copy honest. Before unification the two definitions disagreed on 188 of 292 projects.

**When it fails.** Align the dbt re-implementation with the canonical view, never the reverse.

**Severity.** error. **Baseline.** 0 rows (2026-08-26). **Related.** Decisions log 2026-08-26.

**Source.** [`main/tests/tests_on_static_tables/canonical_parity/test_project_date_matches_canonical.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/canonical_parity/test_project_date_matches_canonical.sql)

<a id="t-stg-cash-flows-match-canonical"></a>
### test_stg_cash_flows_match_canonical

**Checks that.** Every row of `stg_signed_monthly_cash_flows` exists in canonical `v_cash_flows`.

**Why it exists.** Signedness is filtered by the consumer, not the view; this asserts the direction that must always hold rather than full equality.

**When it fails.** A stg row absent from the view means the staging filter widened or the view lost rows.

**Severity.** error. **Baseline.** 2026-07-30: 15,349 rows both sides. **Related.** Decisions log 2026-07-28 (layering).

**Source.** [`main/tests/tests_on_static_tables/canonical_parity/test_stg_cash_flows_match_canonical.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/canonical_parity/test_stg_cash_flows_match_canonical.sql)

<a id="t-stg-signed-projects-matches-canonical"></a>
### test_stg_signed_projects_matches_canonical

**Checks that.** dbt's signed-project set (`stg_signed_quotes`) equals the canonical layer's (inlined in five `v_*` views).

**Why it exists.** The signed rule lives in two places until it is promoted to one view; this fails the moment either side changes what "signed" means.

**When it fails.** Decide which side changed on purpose and align the other. Retire the test when the rule has one home.

**Severity.** error. **Baseline.** 2026-07-30: 281 = 281. **Related.** Decisions log 2026-07-24 (status semantics).

**Source.** [`main/tests/tests_on_static_tables/canonical_parity/test_stg_signed_projects_matches_canonical.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/canonical_parity/test_stg_signed_projects_matches_canonical.sql)

## Data health

<a id="t-data-consistency-invariants"></a>
### test_data_consistency_invariants

**Checks that.** Hard invariants from the July 2026 data review: a signed quote carries a signing date (A1), retail > 0 (A4), at most one signed original quote per project (B3).

**Why it exists.** Every invariant passed on 2026-07-12; a row means a regression.

**When it fails.** The `chk` column names the invariant; see `docs/data-quality/2026-07-data-review.md`.

**Severity.** error. **Baseline.** 2026-07-12: 0 rows.

**Source.** [`main/tests/tests_on_static_tables/data_health/test_data_consistency_invariants.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/data_health/test_data_consistency_invariants.sql)

<a id="t-electricity-price-in-plausible-band"></a>
### test_electricity_price_in_plausible_band

**Checks that.** Signed projects whose electricity price, in USD, is outside 0.08–0.60 per kWh.

**Why it exists.** 30-year savings = kWh × this price. Two incidents sat outside the band for months: USD prices tagged GTQ (savings 7.7× too low) and three projects inflating the June 2026 deck by ~USD 4.2M.

**When it fails.** Check the price and `electricity_price_currency` on the estimate; a price below ~1 tagged GTQ is a USD price with the wrong tag.

**Severity.** warn (flip to error once infra #491 is closed).

**Source.** [`main/tests/tests_on_static_tables/data_health/test_electricity_price_in_plausible_band.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/data_health/test_electricity_price_in_plausible_band.sql)

<a id="t-every-signed-estimate-has-escalation-rate"></a>
### test_every_signed_estimate_has_escalation_rate

**Checks that.** Every signed estimate with panels resolves an electricity price inflation rate inside the sane band.

**Why it exists.** The rate has no fallback by design; a NULL silently nulls 30-year savings. The 2026-09-18 incident was a percent typed into a fraction field.

**When it fails.** Give the project a site with a country (ops) or fix `countries.electricity_price_escalation_rate` (a fraction).

**Severity.** error. **Baseline.** 0 rows (2026-09-21). **Related.** Decisions log 2026-09-19.

**Source.** [`main/tests/tests_on_static_tables/data_health/test_every_signed_estimate_has_escalation_rate.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/data_health/test_every_signed_estimate_has_escalation_rate.sql)

<a id="t-country-carbon-assumptions-in-band"></a>
### test_country_carbon_assumptions_in_band

**Checks that.** The three operating countries carry all four CO2 inputs on `countries` — gross grid factor incl. upstream (0.30–1.20 t/MWh), panel lifecycle factor (0.02–0.12), decarbonization rate (0–3 %/yr), base year (2020–2035).

**Why it exists.** Every CO2 figure Albedo publishes is kWh × these four numbers (Valuing Impact method, adopted 2026-10-20). One wrong or missing value moves every impact report at once; a percent typed into the rate field (0.5 instead of 0.005) is the 2026-09-18 panel-degradation incident again.

**When it fails.** Check the `countries` row and the last migration that set it (`database_change_log`). The fix is the value, never the test.

**Severity.** error. **Baseline.** 0 rows (2026-10-20). **Related.** Decisions log 2026-10-20 "CO2 avoided follows the Valuing Impact method".

**Source.** [`main/tests/tests_on_static_tables/data_health/test_country_carbon_assumptions_in_band.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/data_health/test_country_carbon_assumptions_in_band.sql)

<a id="t-rural-area-follows-gps"></a>
### test_rural_area_follows_gps

**Checks that.** For every signed project, `rural_area` in the impact mart equals the GPS rule on the site's GHS-SMOD code: 11 or 12 = rural, any other code = urban, no coordinates = Not Sure. The manual flag on the project is never used.

**Why it exists.** The rule has been in force since 2026-07-09, but a September 2026 export to Valuing Impact was built from the manual flag and showed 37 rural projects where the GPS rule gives 105. The marts were right; this test keeps them right, and the VI export is now a model downstream of them.

**When it fails.** Someone changed the derivation in the impact mart. Restore the rule; do not edit project rows. A project with no coordinates stays Not Sure until ops captures them.

**Severity.** error. **Baseline.** 0 rows (2026-10-26, 306 signed projects, 277 with coordinates). **Related.** Decisions log 2026-07-09; infra #616 (retire the manual flags).

**Source.** [`main/tests/tests_on_static_tables/data_health/test_rural_area_follows_gps.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/data_health/test_rural_area_follows_gps.sql)

<a id="t-impact-flags-match-canonical-view"></a>
### test_impact_flags_match_canonical_view

**Checks that.** For every signed project, women-led, youth-led, educational, non-profit, rural and low-income area in the impact summary equal the values in `v_project_impact_flags`.

**Why it exists.** Since 2026-10-28 the six indicators have one home that pricing, the app and every report read. If a report is rewired back to the legacy project columns, the numbers we publish stop matching what the app shows and what discounts were given.

**When it fails.** A model reads the flags from somewhere else. Point it at `int_project_impact_flags`; never edit project rows to make it pass.

**Severity.** error. **Baseline.** 0 rows (2026-10-28). **Related.** Decisions log 2026-10-28 "Impact indicators have one home".

**Source.** [`main/tests/tests_on_static_tables/data_health/test_impact_flags_match_canonical_view.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/data_health/test_impact_flags_match_canonical_view.sql)

<a id="t-signed-clients-have-impact-flags"></a>
### test_signed_clients_have_impact_flags

**Checks that.** Lists signed projects whose client has no Sí/No answer for women-led, youth-led, educational institution or non-profit.

**Why it exists.** These answers belong to the client. Until every signed client is answered, the impact percentages carry unknowns and the legacy project columns cannot be retired (infra #616).

**When it fails.** It warns, it doesn't fail. Each row is a question for sales or ops: open the client (or the Impact page) and answer Sí / No for the four indicators.

**Severity.** warn; raise to error once it reaches 0. **Baseline.** 50 of 307 signed projects (2026-10-28). **Related.** Decisions log 2026-10-28.

**Source.** [`main/tests/tests_on_static_tables/data_health/test_signed_clients_have_impact_flags.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/data_health/test_signed_clients_have_impact_flags.sql)

<a id="t-vi-base-matches-impact-mart"></a>
### test_vi_base_matches_impact_mart

**Checks that.** The Valuing Impact input table (`mart_vi_base_de_datos_impacto`) has the same projects as the impact mart, the same kW, net savings equal to lifetime savings minus total paid, and an Ámbito territorial that is Rural, Urbana or Pendiente and matches the GPS-derived field.

**Why it exists.** That table is what Valuing Impact's societal-value model runs on. If it drifts from the impact mart, Albedo publishes two different portfolios. It replaces the hand-built export that caused the rural mismatch.

**When it fails.** Fix the mapping in the model; never patch the exported file by hand.

**Severity.** error. **Baseline.** 0 rows (2026-10-26). **Related.** dbt `CALCULATIONS.md` § "Valuing Impact input table".

**Source.** [`main/tests/tests_on_static_tables/data_health/test_vi_base_matches_impact_mart.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/data_health/test_vi_base_matches_impact_mart.sql)

<a id="t-panel-degradation-in-range"></a>
### test_panel_degradation_in_range

**Checks that.** Solar panels have `annual_depreciation_percentage` in 0–0.02 and `first_year_production_loss_percentage` in 0–0.10 (fractions).

**Why it exists.** Compounded over 30 years. A percent typed into the fraction field cut 24 projects' lifetime production to ~1/7 for two months and understated the Jul-31 investor export by ~12,900 t CO2 (infra #492).

**When it fails.** Correct the row in `public.equipments`.

**Severity.** error. **Baseline.** 2026-09-18: 0 rows after the fix.

**Source.** [`main/tests/tests_on_static_tables/data_health/test_panel_degradation_in_range.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/data_health/test_panel_degradation_in_range.sql)

<a id="t-signed-projects-have-sector"></a>
### test_signed_projects_have_sector

**Checks that.** Every signed project has a `project_sector_id`.

**Why it exists.** The sector feeds every per-sector total in impact and investor reports; a signed project without one drops out silently. QB used free text and curation stopped in Jan 2026.

**When it fails.** Set the sector on the project page (hints: `colloquial_name`, the client's other projects).

**Severity.** error. **Baseline.** 2026-10-19: 297 signed, 4 offenders (1998-01, 1998-02, 2078-01, 2225-01). **Related.** Decisions log 2026-10-16 (sector 3 rule).

**Source.** [`main/tests/tests_on_static_tables/data_health/test_signed_projects_have_sector.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/data_health/test_signed_projects_have_sector.sql)

<a id="t-backup-tables-expired"></a>
### test_backup_tables_expired

**Checks that.** Lists safety-copy tables (`_backup_YYYYMMDD_*`, `*_backup_YYYYMMDD`, `*_snapshot_YYYYMMDD`) older than 30 days.

**Why it exists.** Data-repair migrations snapshot rows and nothing cleans them up. A warning means "confirm the change stuck, update the ledger, drop the table".

**When it fails.** Per table: confirm, update infra `database/BACKUP-TABLES.md`, drop. Tables with an open drop condition stay listed.

**Severity.** warn. **Baseline.** 10 tables on 2026-10-16 (Loose Ends #587).

**Source.** [`main/tests/tests_on_static_tables/data_health/test_backup_tables_expired.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/data_health/test_backup_tables_expired.sql)

<a id="t-zoho-sync-ran-last-night"></a>
### test_zoho_sync_ran_last_night

**Checks that.** The nightly Zoho → Supabase sync ran: the newest run ended `completed` or `partial` and started within the last 25 hours.

**Why it exists.** `zoho_invoices` and `lease_payments` feed the Debita loan tape; a failed night only shows in `zoho_sync_runs` if someone looks.

**When it fails.** Read the run row's `status`, `stop_reason`, `error`. `failed`: re-run (`aws lambda invoke … {"trigger":"manual"}`). `skipped`: credential or `ZOHO_SYNC_ENABLED`. No run: the EventBridge rule.

**Severity.** error. **Baseline.** Green since 2026-09-29. **Related.** Infra `docs/zoho-books-integration/05-payments-sync-operations.md`.

**Source.** [`main/tests/tests_on_static_tables/data_health/test_zoho_sync_ran_last_night.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/data_health/test_zoho_sync_ran_last_night.sql)

## EEFFs reports

<a id="t-eeffs-signed-projects-match-reports"></a>
### test_eeffs_signed_projects_match_reports

**Checks that.** Every signed project is in the EEFFs report(s) it belongs to (Financiado → by_financed_project, non-GT also HN_ES_USD; Contado → by_paid_up_front_project), and nothing else is.

**Why it exists.** Finanzas pulls these monthly and mid-month; 2574-02 was missing from the Honduras report until a manual rebuild and nobody's test noticed. The marts are tables, so this also catches a stale build.

**When it fails.** `likely_cause` names the inner join that dropped the project: no original quote, no cash flows, sale_type unknown, no site/country, or stale table (`dbt run --select int_monthly_cash_flows_month_0_adjustment_converted_currencies+`).

**Severity.** error. **Baseline.** 2026-10-08: 300 signed = 230 financed + 70 contado, 0 rows. **Related.** Tag `eeffs`; run on its own in the nightly.

**Source.** [`main/tests/tests_on_static_tables/eeffs/test_eeffs_signed_projects_match_reports.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/eeffs/test_eeffs_signed_projects_match_reports.sql)

<a id="t-eeffs-amounts-match-sources"></a>
### test_eeffs_amounts_match_sources

**Checks that.** Per project, the report's flow rows summed across its month columns equal the cash-flow table for the same window (Jan-21 to Dec-30), and Precio Retail / Costo Proyecto equal the original estimate (tolerance 1.00). Since 2026-10-29 this includes Ingresos por Mantenimiento, and Pago Total includes the maintenance cuota ([decision](../quote-generation/6-business-decisions-log.md#e-eeffs-maintenance)).

**Why it exists.** Catches a month lost or double-counted by the pivot (MAX per month), a currency mix-up, or a signing month outside the window.

**When it fails.** Compare the project's rows with the cash-flow table; the window ruling (2026-10-08) keeps the Dec-30 cutoff.

**Severity.** error. **Baseline.** 2026-10-08: 0 rows, 230 GTQ + 36 USD projects. **Related.** Tag `eeffs`.

**Source.** [`main/tests/tests_on_static_tables/eeffs/test_eeffs_amounts_match_sources.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/eeffs/test_eeffs_amounts_match_sources.sql)

<a id="t-eeffs-report-shape"></a>
### test_eeffs_report_shape

**Checks that.** Exactly one row per (project, account), the 16 known accounts per project (13 until 2026-10-29; Ingresos por Mantenimiento and CxC - Mantenimiento CP/LP were added then), no blank keys, no Guatemala rows in the HN/SV USD report.

**Why it exists.** A join fan-out or a renamed account would ship silently because the pivot hides duplicates.

**When it fails.** Find the duplicate source row (usually a second original estimate) or restore the account name.

**Severity.** error. **Baseline.** 0 rows. **Related.** Tag `eeffs`.

**Source.** [`main/tests/tests_on_static_tables/eeffs/test_eeffs_report_shape.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/eeffs/test_eeffs_report_shape.sql)

<a id="t-eeffs-signed-original-quote-sale-type-known"></a>
### test_eeffs_signed_original_quote_sale_type_known

**Checks that.** Every signed project has exactly one original quote whose `sale_type` is exactly Financiado or Contado.

**Why it exists.** The marts route by exact string match; a NULL, typo or extra original quote drops or fans out a project. Upstream half of the coverage test.

**When it fails.** Fix the quote row.

**Severity.** error. **Baseline.** 0 rows. **Related.** Tag `eeffs`.

**Source.** [`main/tests/tests_on_static_tables/eeffs/test_eeffs_signed_original_quote_sale_type_known.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/eeffs/test_eeffs_signed_original_quote_sale_type_known.sql)

## Adendas and signing

<a id="t-addendum-chain-health"></a>
### test_addendum_chain_health

**Checks that.** Active cash flows across an adenda chain form one timeline: no month billed twice, no gap at the seam, no half-fired supersession, no signed quote without a schedule. The rule lives in `public.v_addendum_chain_health`; the test asks "any rows?".

**Why it exists.** Shared with the contract-generator's post-sign verification and the project page banner. Any row is a real broken chain.

**When it fails.** `issue` and `detail` name the quotes/months. Start with `signing_failures` for the same date; repair per infra `database/addendum-supersession.md`.

**Severity.** error. **Baseline.** 2026-08-11: 0 rows. **Related.** Run on its own as the nightly's invariant step. Decisions log 2026-07-20.

**Source.** [`main/tests/tests_on_static_tables/test_addendum_chain_health.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_addendum_chain_health.sql)

<a id="t-addendum-chain-linkage"></a>
### test_addendum_chain_linkage

**Checks that.** When an addended quote still has live pre-seam rows, its live successor carries `original_quote_id` (else the chain numbers as two sequences restarting at 1); no link points outside the project's signed set.

**Why it exists.** QB-era full-restatement chains need no link and are not flagged; the modern seam flow always stamps the link, and this catches the one that forgets.

**When it fails.** Stamp `original_quote_id` (and `replaces_quote_id`) per the 2026-09-04 backfill pattern.

**Severity.** error. **Baseline.** 2026-09-04: 0 rows after backfilling the 577-02/03 chains.

**Source.** [`main/tests/tests_on_static_tables/test_addendum_chain_linkage.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_addendum_chain_linkage.sql)

<a id="t-signing-failures-unresolved"></a>
### test_signing_failures_unresolved

**Checks that.** No row in `public.signing_failures` sits without `resolved_at`.

**Why it exists.** The sign path records a failed stage and continues instead of aborting; nobody read that table, and 2574-02 sat signed with zero cash-flow rows for 18 days.

**When it fails.** The `error` column says what broke. Re-run the stage (re-saving the signed date usually re-materializes), confirm the chain is clean, set `resolved_at`.

**Severity.** error. **Baseline.** 0 rows (2026-10-07).

**Source.** [`main/tests/tests_on_static_tables/test_signing_failures_unresolved.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_signing_failures_unresolved.sql)

<a id="t-one-non-addended-flow-per-month-per-project"></a>
### test_one_non_addended_flow_per_month_per_project

**Checks that.** Each project has exactly one active (non-addended) cash flow per payment number.

**Why it exists.** Two live rows for one payment would double-bill.

**When it fails.** Find the duplicate quote/row; usually a supersession that did not stamp `addended_at`.

**Severity.** error. **Baseline.** 0 rows.

**Source.** [`main/tests/tests_on_static_tables/test_one_non_addended_flow_per_month_per_project.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_one_non_addended_flow_per_month_per_project.sql)

<a id="t-quotes-accounted-for"></a>
### test_quotes_accounted_for

**Checks that.** Every signed quote has cash-flow rows.

**Why it exists.** Pending drafts may briefly lack flows by design; signed contracts never.

**When it fails.** Materialize (re-save the signed date) and check `signing_failures`.

**Severity.** error. **Baseline.** 0 rows.

**Source.** [`main/tests/tests_on_static_tables/test_quotes_accounted_for.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_quotes_accounted_for.sql)

<a id="t-all-quotes-have-contract-signing-date"></a>
### test_all_quotes_have_contract_signing_date

**Checks that.** Every signed quote has a `contract_signing_date`.

**Why it exists.** Since 2026-08-26 the column mirrors the client contract; a NULL means the date exists nowhere or the mirror broke.

**When it fails.** Fix the contract row or the sync trigger.

**Severity.** error. **Baseline.** 0 rows. **Related.** Decisions log 2026-10-02.

**Source.** [`main/tests/tests_on_static_tables/test_all_quotes_have_contract_signing_date.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_all_quotes_have_contract_signing_date.sql)

<a id="t-no-future-contract-signing-dates"></a>
### test_no_future_contract_signing_dates

**Checks that.** No quote has a signing date in the future.

**Why it exists.** A future date is a typo and shifts project dates and fiscal-year buckets.

**When it fails.** Correct the contract's `signed_on`.

**Severity.** error. **Baseline.** 0 rows.

**Source.** [`main/tests/tests_on_static_tables/test_no_future_contract_signing_dates.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_no_future_contract_signing_dates.sql)

<a id="t-cash-flow-sanity-checks"></a>
### test_cash_flow_sanity_checks

**Checks that.** Contado schedules have no down payment and at most two payments unless every row carries a `payment_milestone` label; financed payment numbers don't skip; every quote's flows belong to one estimate.

**Why it exists.** Milestone plans (since 2026-09-10) are a product feature; a >2-payment plan with an unlabelled row is the shape a data edit leaves. 1199-01 is a verified custom schedule (Javi, 2025-10-13).

**When it fails.** `test_description` names the failing check.

**Severity.** error. **Baseline.** 0 rows.

**Source.** [`main/tests/tests_on_static_tables/test_cash_flow_sanity_checks.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_cash_flow_sanity_checks.sql)

<a id="t-data-consistency"></a>
### test_data_consistency

**Checks that.** Estimates point at the project their reference names; quote `sale_type` is Financiado or Contado.

**Why it exists.** Cheap referential sanity.

**When it fails.** Fix the row.

**Severity.** error. **Baseline.** 0 rows.

**Source.** [`main/tests/tests_on_static_tables/test_data_consistency.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_data_consistency.sql)

<a id="t-interest-rates-match-dbt-calculations"></a>
### test_interest_rates_match_dbt_calculations

**Checks that.** `quotes.monthly_interest_rate` equals the rate dbt derives in `stg_calculated__interest_rates` for the same project.

**Why it exists.** Catches a stored rate that diverges from the schedule it describes.

**When it fails.** See test_interest_payments_match for the convention; correct the stored rate or the schedule.

**Severity.** error. **Related.** Joins on `project_reference` (legacy).

**Source.** [`main/tests/tests_on_static_tables/test_interest_rates_match_dbt_calculations.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_interest_rates_match_dbt_calculations.sql)

<a id="t-exchange-rates-gap-free"></a>
### test_exchange_rates_gap_free

**Checks that.** The GTQ exchange-rate series has no missing date between its start and end, and the newest rate is at most 3 days old.

**Why it exists.** Conversions join on exact date; 2406-01 vanished from the analytics lineage because its start date fell on one of 10 skipped days.

**When it fails.** `gap`: backfill the date (pattern: infra 2026-08-27 migration). `stale`: the daily sync died.

**Severity.** error. **Baseline.** 0 rows (2026-08-27).

**Source.** [`main/tests/tests_on_static_tables/test_exchange_rates_gap_free.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_exchange_rates_gap_free.sql)

## Maintenance

<a id="t-maintenances-exist-after-cambio-medidor"></a>
### test_maintenances_exist_after_cambio_medidor

**Checks that.** A signed Financiado project with the Cambio Medidor milestone registered has `project_maintenances` rows.

**Why it exists.** Since 2026-10-12 the milestone creates the schedule; a project without rows means the side effect failed or the milestone was written around the edge function.

**When it fails.** Replay the seed for the project; resolve the `signing_failures` row.

**Severity.** error. **Baseline.** 2026-10-12: 199 of 230 had the milestone, all 199 had rows. **Related.** Decisions log 2026-10-12.

**Source.** [`main/tests/tests_on_static_tables/test_maintenances_exist_after_cambio_medidor.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_maintenances_exist_after_cambio_medidor.sql)

<a id="t-maintenances-only-on-live-financiado"></a>
### test_maintenances_only_on_live_financiado

**Checks that.** Maintenance rows exist only on non-deleted projects with a live signed Financiado quote.

**Why it exists.** Contado clients own the system; unsigned projects have nothing to maintain; a reverted or soft-deleted quote should take its schedule with it.

**When it fails.** Delete the rows via a migration if the quote was reverted on purpose; otherwise fix the quote.

**Severity.** error. **Baseline.** 2026-10-12: 0 rows (230 projects).

**Source.** [`main/tests/tests_on_static_tables/test_maintenances_only_on_live_financiado.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_maintenances_only_on_live_financiado.sql)

<a id="t-maintenance-years-contiguous"></a>
### test_maintenance_years_contiguous

**Checks that.** Each project's schedule is years 1..N with no gaps.

**Why it exists.** The UNIQUE constraint prevents duplicates but not holes; a hand-deleted row leaves one.

**When it fails.** Re-seed the missing year with the project's anchor, or delete the strays and say so in the CHANGELOG.

**Severity.** error. **Baseline.** 2026-10-12: 0 rows.

**Source.** [`main/tests/tests_on_static_tables/test_maintenance_years_contiguous.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_maintenance_years_contiguous.sql)

<a id="t-maintenance-dates-share-one-anchor"></a>
### test_maintenance_dates_share_one_anchor

**Checks that.** `scheduled_date(N) = scheduled_date(1) + (N − 1) × 12 months` for every row of a project.

**Why it exists.** Both anchor rules (delivery date 2026-06-07, Cambio Medidor 2026-10-12) keep this spacing; a re-anchor moves every pending row together. A row off the grid is a hand edit.

**When it fails.** Recompute from the year-1 row, or re-anchor the whole project and record why.

**Severity.** error. **Baseline.** 2026-10-12: 1,310 / 1,310 on grid. **Related.** Decisions log 2026-10-12.

**Source.** [`main/tests/tests_on_static_tables/test_maintenance_dates_share_one_anchor.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_maintenance_dates_share_one_anchor.sql)

<a id="t-maintenance-flags-consistent"></a>
### test_maintenance_flags_consistent

**Checks that.** `albedo_pays = cost_project_currency >= 0.01` and `is_warranty_covered = NOT albedo_pays`.

**Why it exists.** The Mantenimientos page sums what Albedo owes by flag; a drifted flag misstates it.

**When it fails.** Re-derive both flags from the cost (the 2026-06-12 near-zero migration's UPDATE is idempotent).

**Severity.** error. **Baseline.** 2026-10-12: 0 rows.

**Source.** [`main/tests/tests_on_static_tables/test_maintenance_flags_consistent.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_maintenance_flags_consistent.sql)

<a id="t-maintenance-status-matches-actual-date"></a>
### test_maintenance_status_matches_actual_date

**Checks that.** `status = Completado` iff `actual_date` is set; only Pendiente / Completado are stored (Vencido is computed at read time).

**Why it exists.** Status is derived, not chosen; new derived states are computed, never stored.

**When it fails.** Set status from `actual_date`. If the stored vocabulary ever changes, update the test.

**Severity.** error. **Baseline.** 2026-10-12: 0 rows.

**Source.** [`main/tests/tests_on_static_tables/test_maintenance_status_matches_actual_date.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_maintenance_status_matches_actual_date.sql)

<a id="t-maintenance-actual-date-not-future"></a>
### test_maintenance_actual_date_not_future

**Checks that.** `actual_date` (fecha real) is never in the future.

**Why it exists.** A future value means ops noted a planned visit in the wrong field; that inflates the Completado count. The separate visit-date field now exists.

**When it fails.** Clear `actual_date` and put the date in `visit_date`. Raise to error once the three baseline rows are cleared.

**Severity.** warn. **Baseline.** 2026-10-12: 3 rows (990-01 / 990-06 / 990-08 year 2).

**Source.** [`main/tests/tests_on_static_tables/test_maintenance_actual_date_not_future.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/test_maintenance_actual_date_not_future.sql)

## Zoho sync

<a id="t-zoho-invoice-status-words"></a>
### test_zoho_invoice_status_words

**Checks that.** `zoho_invoices.invoice_status` holds only Closed / Open / Overdue / Void / Draft.

**Why it exists.** The Debita tape matches on these exact words; an API word (paid, sent, partially_paid…) landing here makes the tape silently stop counting.

**When it fails.** Correct `mapInvoiceStatus` in the sync and re-run.

**Severity.** error. **Baseline.** 0 rows.

**Source.** [`main/tests/tests_on_static_tables/zoho_sync/test_zoho_invoice_status_words.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/zoho_sync/test_zoho_invoice_status_words.sql)

<a id="t-zoho-invoice-balance-fits-status"></a>
### test_zoho_invoice_balance_fits_status

**Checks that.** For sync-sourced invoices: Closed ⇒ nothing left to pay; Open/Overdue ⇒ something left; 0 ≤ balance ≤ total; Overdue ⇒ due date passed; Open ⇒ due date not passed (2 days slack).

**Why it exists.** DPD and mora buckets are built from exactly these columns. The last rule is what tells us the sync's unpaid re-read (stage 1b) broke.

**When it fails.** Re-run the sync; if it persists, check stage 1b (reconcileUnpaid).

**Severity.** error. **Baseline.** 0 rows.

**Source.** [`main/tests/tests_on_static_tables/zoho_sync/test_zoho_invoice_balance_fits_status.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/zoho_sync/test_zoho_invoice_balance_fits_status.sql)

<a id="t-zoho-invoice-lines-add-up"></a>
### test_zoho_invoice_lines_add_up

**Checks that.** An invoice's lines (before tax) equal its total, either as-is or grossed up by one country's IVA (12/13/15 %), tolerance 0.10.

**Why it exists.** `v_zoho_invoice_lines` classifies charges by line (mora, seguro, enganche); a missing or leftover line makes those sums wrong while the invoice looks fine.

**When it fails.** Clear the invoice's `lines_synced_at` so the sync re-reads it.

**Severity.** error. **Baseline.** 0 rows.

**Source.** [`main/tests/tests_on_static_tables/zoho_sync/test_zoho_invoice_lines_add_up.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/zoho_sync/test_zoho_invoice_lines_add_up.sql)

<a id="t-zoho-payments-not-duplicated"></a>
### test_zoho_payments_not_duplicated

**Checks that.** The same Zoho payment is not stored twice against the same invoice.

**Why it exists.** `lease_payments` was filled three ways; a payment written under two ids would be summed twice.

**When it fails.** Keep the row whose `zoho_invoice_payment_id` Zoho returns; delete the other.

**Severity.** error. **Baseline.** 0 rows.

**Source.** [`main/tests/tests_on_static_tables/zoho_sync/test_zoho_payments_not_duplicated.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/zoho_sync/test_zoho_payments_not_duplicated.sql)

<a id="t-zoho-payments-point-at-an-invoice"></a>
### test_zoho_payments_point_at_an_invoice

**Checks that.** Every sync-written payment names an invoice that exists in `zoho_invoices` ("Saldo de inicio del cliente" excepted: it is the opening balance, not an invoice).

**Why it exists.** The payment tape joins payments to invoices for the due date; an orphan payment drops out of DPD.

**When it fails.** Re-run the sync; if the invoice never arrives, look it up in Zoho by number.

**Severity.** error. **Baseline.** 0 rows.

**Source.** [`main/tests/tests_on_static_tables/zoho_sync/test_zoho_payments_point_at_an_invoice.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/zoho_sync/test_zoho_payments_point_at_an_invoice.sql)

<a id="t-zoho-project-links-agree"></a>
### test_zoho_project_links_agree

**Checks that.** The three copies of "which project does this invoice belong to" (`invoice_project_map`, `zoho_invoices`, `lease_payments`) agree for sync-written rows, and the invoice's project exists with the right reference.

**Why it exists.** The loan tape takes the project from the invoice, the payment tape from the payment; drift credits a payment to one project and charges its invoice to another.

**When it fails.** The map is the truth; copy its project onto the invoice or payment. Seven manual-load payments from Feb 2026 are known and excluded (Loose Ends #587).

**Severity.** error. **Baseline.** 0 rows.

**Source.** [`main/tests/tests_on_static_tables/zoho_sync/test_zoho_project_links_agree.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/zoho_sync/test_zoho_project_links_agree.sql)

<a id="t-zoho-invoices-without-project"></a>
### test_zoho_invoices_without_project

**Checks that.** Lists invoices the sync read in full but could not link to a project (void excluded).

**Why it exists.** Known and waiting on a decision (Ian, 2026-09-28): pagaré interest, contado milestones, one-off services. Kept in view so a jump in the list is noticed.

**When it fails.** Add the link by hand in `invoice_project_map` (match_method manual) or name the project on the invoice in Zoho.

**Severity.** warn. **Baseline.** 2026-09-29: 27; 2026-10-16: 56 (Loose Ends #587).

**Source.** [`main/tests/tests_on_static_tables/zoho_sync/test_zoho_invoices_without_project.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/zoho_sync/test_zoho_invoices_without_project.sql)

<a id="t-zoho-payments-cover-amount-paid"></a>
### test_zoho_payments_cover_amount_paid

**Checks that.** Stored payments (applied, or applied + tax withheld) are not short of Zoho's `amount_paid` by more than 10 in the invoice's currency.

**Why it exists.** A gap is not always a missing payment: `amount_paid` is cash only (credits and write-offs are their own columns since 2026-10-21; see `test_zoho_invoice_settlement_adds_up`), and a payment entered against the customer's opening balance never reaches the invoice.

**When it fails.** Open the invoice in Zoho and read its payment history; if a payment is there and not here, re-run the sync and report it.

**Severity.** warn. **Baseline.** 2026-09-29: 2 invoices; 4 on 2026-10-16 (Loose Ends #587). **Related.** Infra docs `05-payments-sync-operations.md` (what amount_paid means).

**Source.** [`main/tests/tests_on_static_tables/zoho_sync/test_zoho_payments_cover_amount_paid.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/zoho_sync/test_zoho_payments_cover_amount_paid.sql)

<a id="t-zoho-invoice-settlement-adds-up"></a>
### test_zoho_invoice_settlement_adds_up

**Checks that.** For sync-sourced, non-void invoices whose detail has been read (`amount_paid IS NOT NULL`): `total − balance` equals `amount_paid` (cash received) + `credits_applied` + `write_off_amount` + the tax the client withheld (`lease_payments.withholding_tax_amount`), within 1.00 in the invoice's currency.

**Why it exists.** A credit note zeroes an invoice's balance in Zoho exactly like a payment does, with `payment_made = 0` (810-07's adenda holiday months, 2026). The loan tape must count that as a credit, not cash, and these four columns are the only way to tell. A row here means the detail read is stale or Zoho settled the invoice some way we do not store.

**When it fails.** `UPDATE zoho_invoices SET lines_synced_at = NULL WHERE zoho_invoice_number = '…';` and wait for the next sync run (or run it by hand: infra `05-payments-sync-operations.md`). If it persists, the invoice's payment history in Zoho shows what settled it.

**Severity.** warn; becomes error once it has been 0 for two weeks. **Baseline.** 2026-10-21: the 263 invoices queued for re-read by migration `20261020230000` clear over the first nights; expected 0 after that. **Related.** `v_project_payments.settled_by` (same arithmetic, per cuota).

**Source.** [`main/tests/tests_on_static_tables/zoho_sync/test_zoho_invoice_settlement_adds_up.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/zoho_sync/test_zoho_invoice_settlement_adds_up.sql)

<a id="t-zoho-credit-applications-add-up"></a>
### test_zoho_credit_applications_add_up

**Checks that.** For sync-sourced invoices with a detail read and any credit on either side: the sum of `zoho_credit_note_invoices.amount` for the invoice equals `zoho_invoices.credits_applied`, within 0.10.

**Why it exists.** Two readings of one fact, from two Zoho endpoints (the invoice detail and the credit note detail). They disagree when one side was read and the other not yet, or when a credit note was applied without showing up in the modified-since list.

**When it fails.** Re-run the sync (the credit stage queues the invoice for a re-read the same night). If the gap stays, open the credit note in Zoho and check which invoices it was applied to, then clear the invoice's `lines_synced_at`.

**Severity.** warn. **Baseline.** 2026-10-21: 0 after the first full credit-note run.

**Source.** [`main/tests/tests_on_static_tables/zoho_sync/test_zoho_credit_applications_add_up.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/zoho_sync/test_zoho_credit_applications_add_up.sql)
<a id="t-zoho-snapshot-is-current"></a>
### test_zoho_snapshot_is_current

**Checks that.** Every `public.zoho_invoices` row has exactly one open version (`dbt_valid_to IS NULL`) in `snapshots.snapshot_zoho_invoices`, and that version's check columns (status, balance, total, due date, last payment date, cash received, credits applied, write-off) equal the table.

**Why it exists.** The loan tape reads invoice state *as of a date* from this history (infra #541). The nightly runs `dbt snapshot --select snapshot_zoho_invoices` after the 04:05 UTC Zoho sync and right before the tests, so at test time the open versions must equal the table. A row means the snapshot step did not run or failed, or something wrote to `zoho_invoices` in between.

**When it fails.** `dbt snapshot --select snapshot_zoho_invoices` (safe to repeat: it inserts only what changed). Never `--full-refresh` it: that drops the history seeded from `database_change_log` (dbt `main/OPERATIONS.md`, "Zoho invoice history").

**Severity.** error. **Baseline.** 0 rows at creation (2026-10-21, after the seed). **Related.** `test_zoho_sync_ran_last_night`.

**Source.** [`main/tests/tests_on_static_tables/zoho_sync/test_zoho_snapshot_is_current.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/zoho_sync/test_zoho_snapshot_is_current.sql)

<a id="t-zoho-snapshot-one-open-version-per-invoice"></a>
### test_zoho_snapshot_one_open_version_per_invoice

**Checks that.** No invoice has more than one open version, and no two versions of the same invoice overlap in time, in `snapshots.snapshot_zoho_invoices`.

**Why it exists.** "State as of D" picks the version whose validity interval holds D; two open or overlapping versions make the answer ambiguous and double every as-of join. Happens if the seed is run twice, a hand-run `dbt snapshot` overlaps the nightly, or `zoho_invoice_number` stops being unique.

**When it fails.** Keep the version with the latest `dbt_valid_from`, close the others at that time, and find what wrote twice.

**Severity.** error. **Baseline.** 0 rows at creation (2026-10-21).

**Source.** [`main/tests/tests_on_static_tables/zoho_sync/test_zoho_snapshot_one_open_version_per_invoice.sql`](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/zoho_sync/test_zoho_snapshot_one_open_version_per_invoice.sql)

---

## Entry template

```
| [test_name](https://github.com/AlbedoSolar/dbt/blob/main/main/tests/tests_on_static_tables/<folder>/test_name.sql) | One sentence: the invariant. | error or warn | [entry](#t-name) |

<a id="t-name"></a>
### test_name

**Checks that.** The invariant, in one or two sentences.

**Why it exists.** The incident or risk behind it; what it protects (which report, which page).

**When it fails.** What the output columns mean and the repair, with the migration pattern if one exists.

**Severity.** error / warn (and, for warn, when it becomes error). **Baseline.** Count and date at creation. **Related.** Decisions-log entry, runbook or issue.

**Source.** The file link.
```

Register row under the right group, entry under the same group, both in the PR that adds or changes the test.
