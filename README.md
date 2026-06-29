# US Unemployment Dashboard

**End-to-end BI pipeline: Snowflake → Power BI**

US unemployment data from 1976 to 2025, loaded into Snowflake, modeled in Power BI, and published with a drillthrough page for monthly detail.

🔗 [View Live Report](https://app.powerbi.com/groups/me/reports/ee29b8bb-9631-4833-8dad-5b26998cecf1/bc9c4ba182d508685e7b?experience=power-bi)

---

## Project Summary

|  |  |
|--|--|
| **Data Source** | Kaggle — US unemployment by state, monthly, 1976–2025 |
| **Stack** | Snowflake · Power BI Desktop · Power BI Service |
| **Data Volume** | 31,747 fact rows · 53 dimension rows (50 states + DC + 2 metro areas) |
| **Status** | Complete — published to Power BI Service |

---

## Architecture

```
Kaggle CSV
    |
    v
Snowflake (Azure West US 2)
  Database: US_UNEMPLOYMENT
  Schema:   LABOR_STATS
    - dim_area       (53 rows — states, DC, LA County, NYC)
    - fact_unemployment  (31,747 rows — monthly, 1976–2025)
    |
    v
Power BI Desktop
  Native Snowflake connector · PAT authentication · Import mode
  Power Query: DATE column, Month Short column
  DAX: Calendar table, _Measures table
    |
    v
Power BI Service — Published Report
```

---

## Dashboard

### Page 1 — Summary

![US Unemployment Dashboard](https://github.com/Milad-Gerami/Snowflake_Unemployment_Dashboard/raw/main/US_Unemployment.png)

Three KPI cards snapshot 2025 metrics with year-over-year movement. The trend line covers the full 1976–2025 range — every major labor crisis is visible. Right-click any year on the trend line to drill through to monthly detail.

### Page 2 — Monthly Detail (Drillthrough)

![Monthly Detail](https://github.com/Milad-Gerami/Snowflake_Unemployment_Dashboard/raw/main/Monthly_Detail.png)

Triggered by right-clicking a year on Page 1. Shows monthly unemployment rate for that year, peak month, annual average, and a dashed reference line for the full historical average.

---

## Key Design Decisions

**DIVIDE instead of AVERAGE for rate KPIs** — rows are per state per month. Averaging rates across 51 areas weights Nebraska and California equally. `DIVIDE(total unemployed, total labor force)` gives the true population-weighted national rate.

**AREA_TYPE filter on all measures** — the dataset includes Los Angeles County and New York City as separate metro rows. Without filtering to `AREA_TYPE = "State"`, they double-count on top of California and New York. Filter lives in the measure, not the visual.

**Yearly granularity on trend line, not monthly** — 50 years × 12 months = 600 data points at page width is unreadable. Yearly (50 points) tells the trend cleanly. Monthly detail is available via drillthrough.

**Drillthrough instead of slicer for monthly detail** — a slicer only filters the date range, it doesn't change chart granularity. Drillthrough is the correct Power BI pattern: summary on Page 1, monthly breakdown on Page 2 when the user drills into a specific year.

**Percentage points (pp) not percent for YOY** — 4.0% to 4.2% is a 0.2 pp change, not a 0.2% change. Labeled accordingly on all KPI cards.

**Edit Interactions on KPI cards** — cards are hardcoded to 2025 snapshot metrics. Clicking a year on the trend line should not blank them out. Disconnected via Format → Edit Interactions → None.

---

## Data Model

**Snowflake tables loaded into Power BI via Import mode.**

Two columns added in Power Query on `FACT_UNEMPLOYMENT`:
- `DATE` — `Date.FromText(Text.From([YEAR]) & "-" & Text.From([MONTH]) & "-01")` — typed as Date, used for the relationship to Calendar
- `Month Short` — `Date.ToText(#date(1900, [MONTH], 1), "MMM")` — sorted by `MONTH` (numeric) for correct axis order

**Relationships:**
- `FACT_UNEMPLOYMENT[DATE]` → `Calendar[Date]` — many-to-one, active
- `FACT_UNEMPLOYMENT[FIPS_CODE]` → `DIM_AREA[FIPS_CODE]` — many-to-one, active

**Calendar table (DAX):**
```
Calendar = CALENDAR(DATE(1976,1,1), DATE(2025,12,31))
```

---

## DAX Measures

All measures in `_Measures` table.

```dax
Avg Unemployment Rate 2025 =
DIVIDE(
    CALCULATE(SUM(FACT_UNEMPLOYMENT[TOTAL_UNEMPLOYED]), FACT_UNEMPLOYMENT[YEAR] = 2025, DIM_AREA[AREA_TYPE] = "State"),
    CALCULATE(SUM(FACT_UNEMPLOYMENT[TOTAL_LABOR_FORCE]), FACT_UNEMPLOYMENT[YEAR] = 2025, DIM_AREA[AREA_TYPE] = "State")
)

Avg Unemployment Rate by Year =
DIVIDE(
    CALCULATE(SUM(FACT_UNEMPLOYMENT[TOTAL_UNEMPLOYED]), DIM_AREA[AREA_TYPE] = "State"),
    CALCULATE(SUM(FACT_UNEMPLOYMENT[TOTAL_LABOR_FORCE]), DIM_AREA[AREA_TYPE] = "State")
)

Unemployment Rate YOY Label =
VAR Diff = ([Avg Unemployment Rate 2025] - [Avg Unemployment Rate 2024]) * 100
RETURN
IF(Diff > 0, "▲ " & FORMAT(Diff, "0.0") & " pp", "▼ " & FORMAT(ABS(Diff), "0.0") & " pp")

Peak Month =
VAR PeakMonth =
    CALCULATETABLE(
        TOPN(1,
            SUMMARIZE(FACT_UNEMPLOYMENT, FACT_UNEMPLOYMENT[MONTH],
                "Rate", DIVIDE(SUM(FACT_UNEMPLOYMENT[TOTAL_UNEMPLOYED]), SUM(FACT_UNEMPLOYMENT[TOTAL_LABOR_FORCE]))),
            [Rate], DESC),
        DIM_AREA[AREA_TYPE] = "State"
    )
RETURN
"Peak: " & FORMAT(DATE(1900, MAXX(PeakMonth, FACT_UNEMPLOYMENT[MONTH]), 1), "MMMM")
```

---

## Validation

| Metric | Dashboard | Official BLS | Match |
|---|---|---|---|
| Unemployment Rate 2025 | 4.2% | 4.20% | ✅ |
| Unemployment Rate 2024 | 4.0% | 4.02% | ✅ |
| Participation Rate 2024 | 62.6% | 62.5% | ✅ |
| Labor Force 2025 | 171M | ~170M | ✅ |

---

## Portfolio

[milad-gerami.github.io](https://milad-gerami.github.io)
