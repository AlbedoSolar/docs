# Quote Generation Process: Narrative Walkthrough

This document describes, in plain English, what happens when a quote is generated in the Albedo system. No formulas here — just what happens, why, and where the data comes from at each step. For exact formulas and edge cases, see the [Calculation Reference](./2-calculation-reference.md).

---

## Table of Contents

1. [The Big Picture](#the-big-picture)
2. [Step 1: Client](#step-1-client)
3. [Step 2: Project](#step-2-project)
4. [Step 3: Estimate](#step-3-estimate)
5. [Step 4: Quote (KPI Calculation)](#step-4-quote-kpi-calculation)
6. [What Happens When You Hit "Calculate"](#what-happens-when-you-hit-calculate)
7. [What Happens When You Save a Quote](#what-happens-when-you-save-a-quote)
8. [The Two Calculation Engines](#the-two-calculation-engines)
9. [Addendums](#addendums)

---

## The Big Picture

Albedo finances solar panel installations. A client wants solar panels. A provider installs them. Albedo pays the provider and structures a lease for the client — monthly payments over several years, with insurance and maintenance baked in.

The quote generation process takes one core input — **how much does it cost the provider to install the system** — and builds a complete financial package around it: the retail price the client will pay, the monthly lease payment, insurance and maintenance charges, commission payouts to the sales team, and financial KPIs that tell Albedo whether the deal is profitable.

The QuoteWizard is a 4-step frontend form that walks the user through this process:

1. **Client** — Who is the customer?
2. **Project** — Where is the installation? What kind of project?
3. **Estimate** — What does the installation cost? Who's installing it? Who sold the deal?
4. **Quote** — Given all of that, what are the financial terms?

Steps 1-3 collect data. Step 4 runs the calculations.

---

## Step 1: Client

**What happens:** The user either selects an existing client or creates a new one.

**Data collected:**
- Legal name
- Email
- Tax number (NIT)
- Client reference (auto-generated, e.g., "CLI-01")
- Optional parent client (for subsidiaries)

**Where it's stored:** `clients` table in Supabase.

**Why it matters:** The client is the entity that will sign the lease contract and make monthly payments. The client reference becomes the root of the reference chain used throughout the system (client -> project -> estimate -> quote).

---

## Step 2: Project

**What happens:** The user either selects an existing project for this client or creates a new one. A project represents one physical solar installation at one location.

**Data collected:**
- Project name and description
- Site location: Country, then Department (state/province), then Municipality (city)
- Optional: link to an existing site record
- Project sector classification
- Impact flags: women-led, non-profit, rural area, youth-led, educational institution, impoverished area

**Where it's stored:** `projects` table, with location data from `countries`, `departments`, and `municipalities` tables.

**Why the location matters so much:** The country, department, and municipality directly determine:
- **Tax rate** (IVA): Guatemala is 12%, El Salvador 13%, Honduras 15%
- **Insurance rate**: Varies by department, region, or country depending on what data exists
- **Maintenance rate**: Varies by provider + department/region/country

So the project location is not just metadata — it's a financial input.

**Reference format:** `{clientReference}-{projectNumber}`, e.g., "CLI-01-01" for the first project of client CLI-01.

---

## Step 3: Estimate

**What happens:** This is where the financial inputs begin. The user builds an estimate that captures everything about the installation: what it costs, who's providing it, who sold it, and what commission structure applies.

### The Installation Cost

The most important input is the **project cost in partner contract currency** — this is what the provider charges to install the solar system. It comes from the sum of all **project phases**.

**Project phases** replaced what used to be a single provider/cost field. Now each phase has:
- A **phase type** (All, Structure, Installation, Maintenance, Expansion)
- A **provider** (the company doing the work)
- A **phase cost** (in the partner's contract currency)
- A **partner contract currency** (can differ per phase)

Multiple providers can each handle different phases of the same project. The system converts all phase costs into the project currency using exchange rates from the `exchange_rates` table (most recent rate on or before a reference date).

### The Sales Team and Commissions

Each estimate has a sales team, tracked in `estimate_vendors`:
- **Primary vendor** (required) — the main salesperson
- **Secondary vendor** (optional)
- **Sales director** (optional)

And optional affiliate partners, tracked in `estimate_affiliates`:
- **Aliado** (partner/ally) affiliate
- **Embajador** (ambassador) affiliate

Each of these roles has a commission percentage. The commission structure is determined by the **commission type** — a record in `commission_types` that defines percentage breakdowns for each role (primary vendor %, secondary vendor %, sales director %, aliado %, embajador %).

**How the commission type is selected:** The frontend finds the intersection of commission types shared by all selected vendors and affiliates, then filters for exact role-column matches. If a secondary vendor is selected, the commission type must have a non-null `secondary_vendor_percentage`. If only one commission type survives the filtering, it's auto-selected. Otherwise the user picks from a dropdown.

### Equipment

The estimate also tracks equipment (solar panels, inverters, batteries, etc.):
- Equipment type and subtype
- Brand
- Provider
- Quantity

This is informational — equipment details don't feed into the financial calculations.

### Other Estimate Fields

- **Monthly solar production** (kWh, partner estimate)
- **Price of electricity per kWh**
- **Project currency** (what the client pays in)
- **Power bill currency** (what the client's utility charges in)
- **Project start date**

**Reference format:** `{projectReference}-{estimateNumber}`, e.g., "CLI-01-01-01".

---

## Step 4: Quote (KPI Calculation)

**What happens:** This is where the system takes everything from the estimate and runs the full financial model. The user configures loan terms and financial parameters, hits "Calculate," and gets back a table of KPIs and a month-by-month cashflow projection.

### What the Frontend Does Before the User Sees the Form

When Step 4 loads, several things happen automatically:

1. **Provider selection:** If the estimate has multiple phases with different providers, the frontend selects the provider from the phase with the highest cost — this is typically the primary installer who handles the bulk of the work and maintenance. If multiple providers are detected, a warning banner is shown to the user. The selected provider's `quote_margin_type` ("Add" or "Subtract") and `standard_wholesale_margin` are fetched from the `providers` table to determine how the retail price is derived from the installation cost.

2. **Site location resolution:** If the project has a linked site, the system uses the site's department and municipality for more precise rate lookups. Otherwise it falls back to the project's location fields.

3. **Quote rates fetch:** The frontend calls the `quote-rates` endpoint with the location, the selected provider (highest-cost phase), and asset cost. This returns:
   - The location-specific maintenance rate
   - The location-specific insurance rate
   - The calculated legal fee
   - The retail price before and after commission (a preview)
   - The total commission cost

These values pre-populate the form.

### The Form Fields

The user sees a form with these parameters (many pre-filled from the rates fetch):

**Loan structure:**
- Down payment percentage (default: 3%)
- Annual interest rate (default: 18.2%, or calculated from the recommended APR)
- Loan term in months (default: 63)
- Grace period in months (default: 3)
- Purchase option percentage (default: 1% — this is the lease-to-own buyout at the end)

**Operating costs:**
- Insurance percentage (fetched from location, default fallback: 1.7%)
- Insurance deflation rate (default: 0.96, meaning 4% annual decrease)
- Maintenance rate (fetched from location+provider, default fallback: 2.8%)
- Maintenance premium (default: 10% markup on calculated maintenance costs)
- Maintenance inflation rate (default: 5% annual increase)
- Manual override toggles for both insurance and maintenance

**Financial metrics:**
- WACC (weighted average cost of capital, default: 10%)
- Risk-free rate (default: 4%)
- Target margin (default: 0%)
- Risk level (Low/Medium/High)

**Read-only display fields:**
- Retail price before commission
- Retail price after commission
- Commission costs (read-only if calculated from backend)
- Legal costs

---

## What Happens When You Hit "Calculate"

The frontend sends all form data plus the `estimate_reference` to the **quote-generator** edge function. Here's what the backend does:

### 1. Load the Estimate Context

The backend loads the full estimate from the database, including:
- All project phases with their costs, currencies, and provider margin data
- The commission type with all role percentages
- The project's site location (country, department, municipality)
- The sales team (estimate_vendors) and affiliates (estimate_affiliates)
- Exchange rates for currency conversion

Each phase cost is converted from its partner contract currency to the project currency.

### 2. Calculate the Retail Price

This is the most important derived number. The retail price is **not a direct input** — it's calculated from the installation cost using the provider's margin system.

**There are two margin types:**

**"Add" type:** The installation cost represents the provider's share after their margin. To get the retail price, the system reverses the margin: it divides the cost by the margin percentage (adjusted) to arrive at a larger base retail price. Then commission is added on top.

Think of it this way: if the provider's margin is 15%, and they quoted $10,000, that $10,000 is 85% of the retail price. So the retail price is $10,000 / 0.85 = $11,765. Then commission gets added.

**"Subtract" type:** The installation cost IS the retail base price. Commission is not added separately — it's already factored in.

After the base retail price is calculated per phase (and summed across all phases), a **discount** can be applied:
- If the discount value is less than 1, it's treated as a percentage (e.g., 0.10 = 10% off)
- If the discount value is 1 or greater, it's treated as a flat amount to subtract

Then the **commission** is calculated as: the sum of all role commission percentages, multiplied by the discounted retail price, divided by 1.12 (removing IVA). This commission amount is added to the retail price for "Add" type margins only.

The final number — **retail price with commission** — is what the client pays for the system (before financing terms are applied).

### 3. Calculate the Legal Fee

The legal fee is a fixed amount based on the retail price and currency:

- **USD projects:** $150, $200, or $300 depending on whether the retail price is at/below $8,000, below $13,000, or above
- **GTQ projects:** Q1,175, Q1,600, or Q2,350 at similar thresholds (Q60,000 and Q100,000)
- **Other currencies:** 5% of the partner contract cost

### 4. Calculate the Down Payment

Either provided directly by the user, or: retail price * down payment percentage from the estimate's `down_payment_1_percentage` field.

### 5. Calculate the Installation Schedule

This determines when Albedo pays the provider. Each phase has a **provider payment schedule** that splits the cost into three periods (e.g., 50% upfront, 40% at milestone, 10% on completion). Phase costs are converted to project currency and multiplied by these payout percentages to produce a 3-element array of payment amounts.

If a phase doesn't have a provider payment schedule, it falls back to the default split from the config (50% / 40% / 10%).

### 6. Build the Commission Payout Schedule

Commission is not paid all at once. It's split into two tranches:
- **80%** is paid at month 1 (immediately after contract signing)
- **20%** is paid at the end of the grace period

If the grace period is only 1 month (so both tranches would fall on the same month), the full 100% is paid in month 1.

The total commission is: sum of all role percentages * the retail price (before commission is added).

### 7. Generate the Lease Table

A standard amortization schedule. The balance being financed is: retail price minus the down payment. During the grace period, no payments are made but interest accrues on the balance. After the grace period, regular monthly payments begin using a standard annuity formula. The final month includes the purchase option amount (1% of retail price by default).

### 8. Generate the Services Table

Insurance and maintenance costs are calculated separately from the lease.

**Insurance:**
- Annual cost = installation cost * insurance rate
- Each year, the cost deflates (multiplied by the deflation rate, e.g., 0.96)
- Insurance is assessed every 12 months, starting at month 4 (configurable)
- The total insurance cost across all years gets a premium markup (default: 1.8%)
- The total (with premium) is divided equally across all payment months (from the end of grace period to the end of the loan term)
- The result is a flat monthly insurance payment

**Maintenance:**
- First maintenance visit cost = installation cost * maintenance rate
- Maintenance starts at month 24 (configurable maintenance-free period) with visits every 12 months
- After year 3 (configurable), inflation applies at 5% per year
- The total maintenance cost gets a premium markup (default: 10%)
- Like insurance, the total is spread into equal monthly payments across the payment period
- But the denominator for this spread uses `term - gracePeriod + 1`, not the start month — this is intentional to "allow for discounts" (matches the legacy R code)

### 9. Combine Everything

The lease table, services table, commission schedule, and installation schedule are merged into a single month-by-month table. Each row now has: lease payment, insurance payment, maintenance payment, commission cost, installation cost, legal fee, and down payment.

### 10. Calculate the Full Amortization with Debt and Tax

For each month:

**Income side:**
- Lease income = lease payment + down payment + legal fee + IVA
- Services income = insurance payment + maintenance payment
- Total income = lease + services

**Expense side:**
- Lease expense = installation cost + commission cost + debt cost + legal fee + IVA
- Services expense = actual maintenance cost + actual insurance cost
- Total expense = lease + services

**Debt tracking:** Albedo itself borrows money to fund these installations. The system tracks Albedo's debt balance:
- Debt starts with the installation costs (paid to the provider)
- Each month, the debt grows by (previous balance + new costs - payments received) * (WACC / 12)
- Debt cost each month = that growth amount
- Debt balance is floored at zero (if income exceeds costs, debt doesn't go negative)

**Net Income (NII):** Income minus expense, calculated for three scenarios:
- **Leasing only:** Just the lease payments vs. installation + commission + debt costs
- **With Services (Servicios):** Adds insurance and maintenance payments vs. actual insurance and maintenance costs
- **Total:** Everything including IVA/tax

### 11. Calculate Financial KPIs

From the NII table, the system derives:

- **Income / Expense / Profit** — summed across all months for each scenario
- **NPV (Net Present Value)** — each month's NII discounted at the risk-free rate (4% annual, divided by 12 for monthly)
- **IRR (Internal Rate of Return)** — the monthly rate that makes the NPV of the NII stream equal zero, annualized by multiplying by 12 and converting to percentage
- **Gross Margin** — (profit / expense) * 100
- **Payback Period** — the first month where cumulative NII becomes non-negative (months 0-2 are excluded from consideration and default to 9999)
- **Monthly Payment** — sampled at month 10 (the lease payment, or the total payment with IVA for the services/total columns)
- **APR** — the annual rate used
- **Sale Markup** — ((retail price / installation cost) - 1) * 100

These KPIs are returned in a table with columns: Leasing, Servicios (Services), Total.

---

## What Happens When You Save a Quote

After reviewing the KPIs and cashflow table, the user saves the quote. The frontend:

1. **Generates a quote reference:** `{estimateReference}-{quoteNumber}`, e.g., "CLI-01-01-01-01"

2. **Determines the sale type:**
   - If down payment percentage >= 100% → "Contado" (cash sale, no financing)
   - Otherwise → "Financiado" (financed/leased)

3. **Calculates the first payment date:** project start date + grace period months

4. **Extracts the monthly interest rate** from the IRR KPI in the "Total w IVA" column, converting annual to monthly

5. **Maps the cashflow table to monthly_cash_flows records** — one row per month with the payment date, all payment components (principal, interest, insurance, maintenance, etc.)

6. **Saves to the database:**
   - Creates a `quotes` record
   - Creates `monthly_cash_flows` records (one per month)

All amounts saved to the database are **without IVA/VAT**. IVA is calculated on-the-fly when needed.

---

## The Two Calculation Engines

The quote-generator has two modes:

### Estimate Reference Mode (Primary)

Used by the QuoteWizard. You provide an `estimate_reference` and the backend loads everything it needs from the database — phases, providers, commissions, location, exchange rates. It uses the **integration flow** calculation path: separate lease table + services table, combined, then amortized. This is the more detailed engine.

### Manual Mode (Fallback/Admin)

Used for ad-hoc calculations where you provide all inputs directly (asset_cost, commission_percentages, etc.) without referencing an estimate. It uses the **NII table** calculation path, which is a single unified table rather than separate lease + services tables. The formulas are similar but structured differently.

The frontend always uses Estimate Reference Mode. Manual Mode exists for internal testing and API consumers.

### Key Difference Between the Two Engines

In Estimate Reference Mode:
- Installation costs come from provider payment schedules (actual payout timing)
- The retail price is calculated from phase costs and provider margins
- Commission is based on the estimate's actual vendor/affiliate structure
- Insurance and maintenance use the actual installation total (from phase costs in project currency)

In Manual Mode:
- Asset cost is provided directly
- The retail price is derived from the asset cost + provider margin (if provided)
- Commission percentages are provided as a flat array
- Insurance and maintenance use the provided asset cost directly

---

## Addendums

A quote can be "addended" — replaced with a new version while keeping a link to the original. When an addendum is created:

1. The current quote gets a timestamp in `addended_at`
2. A new quote is created with `addendum_number` incremented
3. The `original_quote_id` links back to the first quote in the chain
4. Monthly cash flows for the old quote are soft-deleted (marked with `addended_at`)
5. New monthly cash flows are created for the new quote

Multiple signed quotes per project are now allowed through this addendum system.

---

## Configuration System

Many of the default values described above can be overridden through a configuration system:

- The `quote_generator_configs` table holds configuration records
- One config is marked as `is_active` (the global default)
- Individual users can be mapped to specific configs via `quote_generator_user_active_configs`
- User config overrides global config, which overrides hardcoded defaults

This allows different sales teams or markets to use different default parameters without code changes.
