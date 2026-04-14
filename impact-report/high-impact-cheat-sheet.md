# High Social Impact — Cheat Sheet

Quick reference for how "high social impact" is defined, where the data comes from, and how it's calculated.

---

## Definition

A project is **high social impact** if it meets **any one** of these six criteria:

| Criteria | Field in DB | Source |
|---|---|---|
| Women-led business | `projects.women_led` | Intake form (manual) |
| Youth-led business | `projects.youth_led` | Intake form (manual) |
| Rural area | `projects.rural_area` | Intake form (manual) |
| Educational institution | `projects.educational_institution` | Intake form (manual) |
| Non-profit organization | `projects.non_profit` | Intake form (manual) |
| Impoverished area | `municipalities.in_poorest_20_percent` | Derived from external poverty data |

The first five are collected during project onboarding. The sixth is derived automatically from the municipality where the project is located.

---

## How "Impoverished Area" Works

The rule is: **a project is in an impoverished area if its municipality is in the poorest 20% of municipalities in that country**.

This is computed by the DBT view `int_projects_mega_view`, which derives `impoverished_area_calculated` from the `municipalities.in_poorest_20_percent` boolean:
- `true` → "Yes"
- `false` → "No"
- `NULL` → "Not Sure"

The `municipalities` table also stores `in_poorest_25_percent`, `in_poorest_33_percent`, and `in_poorest_50_percent` for alternative thresholds, though only the 20% threshold is currently used in impact reporting.

---

## Poverty Data Sources by Country

### Guatemala
- **Source**: Unknown (pre-dates current documentation). Already populated when system was built.
- **Coverage**: 340 municipalities, all have `in_poorest_20_percent` set.

### Honduras
- **Source**: UNDP Human Development Index (IDH), 2009 data
- **Downloaded from**: [Humanitarian Data Exchange (HumData)](https://data.humdata.org/dataset/honduras-indice-de-desarrollo-humano-2009)
- **Methodology**: All 298 municipalities ranked by IDH (ascending). Bottom 20% by IDH (≤ 0.597) = 59 municipalities flagged as poorest.
- **Coverage**: 298 of 299 municipalities matched. 1 unmatched (San Miguel Guancapla).
- **Poorest departments**: Lempira (16), Copán (8), Intibucá (7), Ocotepeque (5), El Paraíso (5)
- **Limitation**: Data is from 2009. A 2022 UNDP Atlas exists but is not available as structured download.

### El Salvador
- **Source**: FISDL/FLACSO Mapa de Pobreza (~2004 methodology)
- **Downloaded from**: [ArcGIS feature service](https://esri-sv.maps.arcgis.com/home/item.html?id=0bc142780f3e44f39d8bbdf7ed9f9116) hosted by Esri El Salvador
- **Methodology**: All 260 municipalities ranked by poverty rate (descending). Top 20% by rate (≥ 64.7%) = 52 municipalities flagged as poorest. Aligns closely with FISDL's 46 "Pobreza Extrema Severa" municipalities (17.7%).
- **Coverage**: 260 of 263 municipalities matched. 3 unmatched (Jujutla, Zacatecas, San Antonio Masahuat).
- **Poorest departments**: Chalatenango, Morazán, Sonsonate, Usulután
- **Limitation**: Data is from ~2004. A 2022 World Bank study exists but the data table is not publicly downloadable. El Salvador restructured to 44 municipalities in 2024, but our projects use the old 262.

---

## Possible Values

All six impact fields use the same value set:

| Value | Meaning |
|---|---|
| `Yes` | Confirmed positive |
| `No` | Confirmed negative |
| `Not Sure` | Unknown / not yet answered |
| `NULL` | Never set (e.g., new project with no intake data, or no poverty data for municipality) |

---

## How the Percentage is Calculated

### Per-category (e.g., "Women-led %")

```
numerator   = COUNT(projects WHERE women_led = 'Yes')
denominator = COUNT(projects WHERE women_led IN ('Yes', 'No'))
percentage  = numerator / denominator * 100
```

Projects with `NULL` or `Not Sure` are **excluded from the denominator** — they don't count as "no".

### High impact overall (any category)

```
numerator   = COUNT(projects WHERE any of the 6 flags = 'Yes')
denominator = depends on approach (see below)
```

**Two valid approaches for the denominator**:

| Approach | Denominator | Use when |
|---|---|---|
| Strict (excl. Not Sure) | Projects where ALL 6 fields are `Yes` or `No` | You want the most accurate rate, but sample size shrinks for recent years |
| Inclusive (incl. Not Sure) | All projects | You want the conservative floor — treats unknowns as "not high impact" |

For years 2021–2024, both approaches give the same result (no "Not Sure" values). For 2025+, they diverge because newer projects often have unresolved `women_led`, `youth_led`, and `rural_area` fields.

---

## SQL — High Impact % by Year

```sql
select
    extract(year from project_date)::int as year,
    count(*) as total_projects,
    count(*) filter (
        where women_led = 'Yes'
           or youth_led = 'Yes'
           or rural_area = 'Yes'
           or educational_institution = 'Yes'
           or non_profit = 'Yes'
           or impoverished_area_calculated = 'Yes'
    ) as high_impact,
    round(
        100.0 * count(*) filter (
            where women_led = 'Yes'
               or youth_led = 'Yes'
               or rural_area = 'Yes'
               or educational_institution = 'Yes'
               or non_profit = 'Yes'
               or impoverished_area_calculated = 'Yes'
        ) / nullif(count(*) filter (
            where women_led in ('Yes', 'No')
              and youth_led in ('Yes', 'No')
              and rural_area in ('Yes', 'No')
              and educational_institution in ('Yes', 'No')
              and non_profit in ('Yes', 'No')
              and impoverished_area_calculated in ('Yes', 'No')
        ), 0), 1
    ) as pct_excl_not_sure,
    round(
        100.0 * count(*) filter (
            where women_led = 'Yes'
               or youth_led = 'Yes'
               or rural_area = 'Yes'
               or educational_institution = 'Yes'
               or non_profit = 'Yes'
               or impoverished_area_calculated = 'Yes'
        ) / count(*), 1
    ) as pct_incl_not_sure
from analytics.int_projects_mega_view
group by 1
order by 1;
```

---

## Key DBT Models

| Model | Schema | What it does |
|---|---|---|
| `int_projects_mega_view` | analytics | Joins projects with municipalities to derive `impoverished_area_calculated`. Source for quarterly rollup. |
| `mart_impact_projects_summary` | analytics | Per-project impact KPIs (kW, CO2, savings, impact flags) |
| `mart_impact_overall_summary` | analytics | Single-row org-wide summary with impact percentages |
| `mart_impact_quarterly` | analytics | Quarterly cumulative and new project/client counts by impact category |

---

## Detailed Source Documentation

Full methodology, name matching details, and raw data files: [`database/migrations/2026-04-14-seed-hn-sv-poverty-data-SOURCES.md`](/albedo-automations-infra/database/migrations/2026-04-14-seed-hn-sv-poverty-data-SOURCES.md)

---

*Last updated: 2026-04-14*
