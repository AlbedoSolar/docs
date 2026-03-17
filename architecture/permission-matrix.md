# Permission Matrix

Source of truth for RLS write policies across all public tables. SELECT access is granted to all authenticated users unless noted otherwise.

## Roles

| Role | Description |
|---|---|
| admin | Full access (role_id = 1) |
| sales | Sales team — can create and manage quotes, estimates, clients |
| operations | Operations team — can update/delete projects and sites |

## Helper Functions

| Function | Purpose |
|---|---|
| `is_admin()` | Returns true if user has role_id = 1 |
| `user_has_role(role_name text)` | Returns true if user has the specified role |
| `is_sales_or_admin()` | Shorthand for `user_has_role('sales') OR is_admin()` |
| `estimate_is_signed(est_id bigint)` | Returns true if estimate has any quote with status `signed` or `signed_and_addended` |

## Business Tables

### Quote/Estimate Tables (signed-record protection)

| Table | INSERT | UPDATE | DELETE | Signed Protection |
|---|---|---|---|---|
| `quotes` | sales, admin | sales, admin | sales, admin | Yes — quote with status `signed` or `signed_and_addended` is immutable |
| `estimates` | sales, admin | sales, admin | sales, admin | Yes — estimate with any signed quote is immutable |
| `estimate_equipment` | sales, admin | sales, admin | sales, admin | Yes — inherits from parent estimate |
| `estimate_affiliates` | sales, admin | sales, admin | sales, admin | Yes — inherits from parent estimate |
| `monthly_cash_flows` | sales, admin | sales, admin | sales, admin | Yes — inherits from parent quote |

### Project/Site/Client Tables (no signed protection)

| Table | INSERT | UPDATE | DELETE | Notes |
|---|---|---|---|---|
| `projects` | sales, admin | sales, ops, admin | sales, ops, admin | Operations can update/delete but not create |
| `sites` | sales, admin | sales, ops, admin | sales, ops, admin | Operations can update/delete but not create |
| `clients` | sales, admin | sales, admin | sales, admin | — |

## Other Tables

> Add rows here as RLS policies are created for additional tables.

| Table | INSERT | UPDATE | DELETE | Notes |
|---|---|---|---|---|
| | | | | |
