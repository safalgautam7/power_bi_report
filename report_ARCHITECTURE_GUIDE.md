# Understanding the QTD and Running Total Report

**A follow-along guide to the actual measures, calculation groups and model design**

This guide describes the definitions in `report.SemanticModel/definition`,
read on 26 November 2026. It explains the existing implementation; it is not
a proposal to rebuild the report. Creating this document does not change
either dashboard.

> **The central idea:** business measures calculate the basic numbers;
> calculation groups choose **what to calculate**, **which dates to use**,
> **which comparison to show**, and **how to arrange the company template**.

## Contents

1. [The architecture at a glance](#1-the-architecture-at-a-glance)
2. [The sales model and filter context](#2-the-sales-model-and-filter-context)
3. [The five physical measures](#3-the-five-physical-measures)
4. [What a calculation group actually does](#4-what-a-calculation-group-actually-does)
5. [Metric: which business number?](#5-metric-which-business-number)
6. [Time Comparison: current, PY and growth](#6-time-comparison-current-py-and-growth)
7. [Period Scope: which date window?](#7-period-scope-which-date-window)
8. [Running Columns: the original running-table adapter](#8-running-columns-the-original-running-table-adapter)
9. [Template Columns: the approved QTD layout](#9-template-columns-the-approved-qtd-layout)
10. [Running Layout: the company running-total format](#10-running-layout-the-company-running-total-format)
11. [Helper tables and calculated columns](#11-helper-tables-and-calculated-columns)
12. [Precedence: how the groups cooperate](#12-precedence-how-the-groups-cooperate)
13. [How each page is connected to the model](#13-how-each-page-is-connected-to-the-model)
14. [Walk through one table cell](#14-walk-through-one-table-cell)
15. [Formatting and slicer interactions](#15-formatting-and-slicer-interactions)
16. [Follow-along learning exercises](#16-follow-along-learning-exercises)
17. [Maintenance rules and troubleshooting](#17-maintenance-rules-and-troubleshooting)
18. [A short explanation you can present](#18-a-short-explanation-you-can-present)
19. [Where to find the full source](#19-where-to-find-the-full-source)

---

## 1. The architecture at a glance

The current model contains:

| Object | Number | Purpose |
| --- | ---: | --- |
| Imported business tables | 4 | Date, Item, Location and sales-line facts |
| Physical measures | 5 | Three base measures, one reporting entry point, one colour helper |
| Calculation groups | 6 | Reusable metrics, comparisons, date windows and page layouts |
| Other calculated/helper tables | 5 | Settings, quarter selection, running dates, Top N and department labels |

Calculation groups themselves appear as tables in Model view. Therefore,
**15 tables in the model does not mean 15 imported business tables**.

```text
Snowflake sales mart
        │
        ▼
DIM_DATE / DIM_ITEM / DIM_LOCATION ──filter──► FCT_SALES
                                                  │
                                        Base business measures
                                                  │
                                           [Trading Metric]
                                                  │
                     ┌────────────────────────────┼──────────────────────┐
                     ▼                            ▼                      ▼
                 QTD page                  Running Total         Department Analysis
              Template Columns             Running Layout           Metric items
                     │                            │                      │
                     └──────── Period Scope / Time Comparison / Metric ─┘

[YoY Status] supplies text colours to visuals, not sales values.
```

The model separates four questions:

| Question | Object that answers it |
| --- | --- |
| What business calculation do I want? | `Metric` |
| Current, previous year, difference or growth rate? | `Time Comparison` |
| Selected quarter, preceding quarters or cumulative dates? | `Period Scope` |
| Which combinations and labels belong in this company template? | `Template Columns`, `Running Layout` and the retained `Running Columns` |

**Why this design?** It avoids separate physical measures for every possible
combination, such as Net Sales PY QTD, Units PY QTD and Net Sales PY Running
Total. The reusable logic is centralized instead.

It is a trade-off: fewer physical measures, but more calculation-group logic
to understand. The groups are real calculations, not a way to eliminate the
underlying complexity.

---

## 2. The sales model and filter context

### 2.1 The four business tables

| Table | What it represents | Typical reporting fields |
| --- | --- | --- |
| `FCT_SALES` | One sales line, not necessarily one whole transaction | Transaction ID, units, gross amount, discount amount |
| `DIM_DATE` | Shared reporting calendar | Calendar date, quarter, month, ISO week, comparable PY date |
| `DIM_ITEM` | Product classifications, including historical versions | Department and Class |
| `DIM_LOCATION` | Store/location classifications, including historical versions | Region, District, Location Type and stable Location ID |

The three business relationships are:

```text
DIM_DATE[DATE_KEY]         1 ──► * FCT_SALES[DATE_KEY]
DIM_ITEM[ITEM_SK]          1 ──► * FCT_SALES[ITEM_SK]
DIM_LOCATION[LOCATION_SK]  1 ──► * FCT_SALES[LOCATION_SK]
```

The intended active filter direction is dimension to fact. The shared Date
table is marked as a date table. Reporting follows transaction dates through
the fact's date key, not the file-load date.

### 2.2 What does “filter context” mean?

It means the set of filters under which a measure is evaluated.

For example, a cell can mean:

```text
Department = Beverages
Region = East
Selected quarter = 2026 Q1
Displayed month = January 2026
```

The product and location filters reach the fact through relationships.
The selected quarter is handled separately by DAX because the quarter
selector deliberately does not filter the sales star directly.

Measures are recalculated for each cell; they are not precomputed values
copied into the report at refresh.

### 2.3 Totals recalculate; they do not always add visible rows

For example:

```text
Month A: sales 100, transactions 10 → ABV 10
Month B: sales 900, transactions 30 → ABV 30

Quarter ABV = 1,000 / 40 = 25
```

The quarter ABV is neither `10 + 30` nor the simple average of 10 and 30.
Distinct transaction counts can also overlap across product groups, so
department transaction counts should not automatically be added together.

Historical facts reference the item/location version used for that sale.
Do not globally filter the dimensions to `IS_CURRENT = TRUE`; that can
remove valid historical sales.

---

## 3. The five physical measures

A **physical measure** here means an actual measure object in the model,
as opposed to a calculation item.

| Measure | Home table | Role |
| --- | --- | --- |
| `[Net Sales]` | FCT_SALES | Base sales amount |
| `[Units]` | DIM_DATE | Base quantity |
| `[Transactioncount]` | FCT_SALES | Base distinct transaction count |
| `[Trading Metric]` | FCT_SALES | Entry point used by reporting visuals |
| `[YoY Status]` | FCT_SALES | Conditional font-colour helper |

### 3.1 Net Sales

```dax
Net Sales =
    SUM(FCT_SALES[GROSS_SALES_AMOUNT])
        - SUM(FCT_SALES[DISCOUNT_AMOUNT])
```

**Why it exists:** it is the core reusable definition of sales after discount.
Keeping it as a base measure avoids repeating this subtraction everywhere.

Although the fact also exposes a net amount column, this measure explicitly
implements the required Gross Sales minus Discount definition.

### 3.2 Units

```dax
Units =
    SUM(FCT_SALES[UNITS])
```

**Why it exists:** quantities are reused by Units, UPT, ASP and running totals.

**Why is its home table DIM_DATE?** Power BI does not allow a measure and a
column in the same table to have case-insensitively identical names. The fact
already has `UNITS`. The measure was housed on DIM_DATE to retain the desired
name without renaming the imported source column.

The home table is organizational. It does **not** mean Units counts dates:
the formula still sums `FCT_SALES[UNITS]` and respects the sales filter context.

### 3.3 Transactioncount

```dax
Transactioncount =
    DISTINCTCOUNT(FCT_SALES[TRANSACTION_ID])
```

**Why not COUNTROWS?** A transaction can have multiple sales lines.

```text
Transaction A: three lines
Transaction B: two lines

Sales-line count = 5
Transactioncount = 2
```

The calculation assumes transaction IDs are globally unique, as required by
the source pipeline's key contract.

The physical measure's exact name is `Transactioncount`. The company's
display label `Transaction_count` is a presentation alias, not another measure.

### 3.4 Trading Metric

```dax
Trading Metric =
    [Net Sales]
```

**Why create a measure that appears to duplicate Net Sales?** It gives the
calculation groups one controlled reporting entry point.

Most numeric report visuals use this measure. Depending on calculation-item
filters, the same measure can return Units, ABV, PY Net Sales or a running total.
Special caption items can also make it return text for headers and filters.

The base measures remain useful for independent checks without being
transformed by every reporting calculation group.

### 3.5 YoY Status

This measure returns colour codes, not a numeric growth KPI:

| Result | Meaning |
| --- | --- |
| `#188038` | Favourable change: green |
| `#C62828` | Unfavourable change: red |
| `#6B7280` | Zero change: grey |
| `#303638` | Ordinary/non-comparison text |

Its complete source is in `FCT_SALES.tmdl`. Read it as this sequence:

1. Identify the comparison column being formatted.
2. Identify the metric and required period scope.
3. Build the current and ISO-aligned previous-year date sets.
4. Recalculate the two values using the applicable business formula.
5. Determine the direction of change.
6. Reverse **favourability**, but not the arithmetic sign, for Discount/Discount %.
7. Return the appropriate colour code.

For example, higher sales are normally green; higher Discount % is red.

**Why does it repeat some business formulas?** Formatting a different measure
through guarded calculation groups is not the same as directly evaluating
`[Trading Metric]`. The helper explicitly calculates CY/PY to avoid an
incorrect colour caused by relying on nested measure substitution.

This creates a maintenance dependency: if a supported metric definition
changes, check its matching formula inside `[YoY Status]` as well.

> A displayed YoY % comes from the comparison logic. `[YoY Status]` only
> tells the visual which colour to use.

---

## 4. What a calculation group actually does

A calculation group is a model table containing named **calculation items**.
Selecting/filtering one of its items changes the expression evaluated for
an eligible measure.

For example, the actual `Metric → Units` expression is:

```dax
IF(
    ISSELECTEDMEASURE([Trading Metric]),
    [Units],
    SELECTEDMEASURE()
)
```

Read that as:

> “If this visual is evaluating Trading Metric, use Units instead.
> Otherwise, leave the selected measure alone.”

### The important functions

| Function | Meaning in this model |
| --- | --- |
| `SELECTEDMEASURE()` | The measure expression being transformed |
| `ISSELECTEDMEASURE(...)` | A guard restricting which measure objects may be transformed |
| `SELECTEDMEASUREFORMATSTRING()` | Inherit the underlying measure's display format |
| `SELECTEDVALUE(...)` | Read a single selected item, quarter or date |
| `CALCULATE(...)` | Evaluate an expression after changing filter context |
| `REMOVEFILTERS(...)` | Remove specified filters, not necessarily all report filters |
| `TREATAS(...)` | Apply a table of values as filters on another column |
| `VALUES(...)` | Capture the distinct values visible in the current context |
| `DIVIDE(...)` | Divide safely, returning blank by default for a zero denominator |

Most items guard `[Trading Metric]`. `Running Layout` additionally handles
`[YoY Status]` where the new running format needs colour behaviour.

Do not paste a calculation-item expression as an ordinary new measure:
`SELECTEDMEASURE()` is intended for calculation-group expressions.

---

## 5. Metric: which business number?

**Table/column:** `Metric[Metric]`  
**Precedence:** 10  
**Source:** `Metric.tmdl`

### 5.1 Core calculation items

| Item | Business calculation | Why it is useful |
| --- | --- | --- |
| Net Sales | `[Net Sales]` | Sales after discount |
| Units | `[Units]` | Quantity sold |
| Transactioncount | `[Transactioncount]` | Number of distinct transactions |
| ABV | Net Sales / Transactioncount | Average basket value under the current filters |
| UPT | Units / Transactioncount | Units per transaction |
| Discount % | Discount amount / Gross Sales amount | Discount rate |
| Gross Sales | Sum of gross amount | Sales before discounts |
| Discount | Sum of discount amount | Total discount given |
| ASP | Net Sales / Units | Average selling price per unit |
| Active Locations | Stable location IDs with non-zero aggregated Net Sales | Stores contributing non-zero net sales in the period |
| Sales per Location | Net Sales / Active Locations | Sales productivity per active location |

For example, the actual ABV item is:

```dax
IF(
    ISSELECTEDMEASURE([Trading Metric]),
    DIVIDE([Net Sales], [Transactioncount]),
    SELECTEDMEASURE()
)
```

Ratios are recomputed in each date/product/location context, including totals.
They are not calculated by adding or averaging the displayed row ratios.

Active Locations counts `LOCATION_ID`, not SCD version key `LOCATION_SK`.
A location with net sales that aggregate to zero is excluded; a negative,
non-zero amount still meets this specific definition.

### 5.2 Advanced calculation items

| Item | What it does | Reason for creating it |
| --- | --- | --- |
| Contribution % | Department Net Sales divided by the selected-department total | Shows department share while retaining external selections |
| Like-for-Like Net Sales | CY sales from locations that traded in both compared periods | Removes the effect of a different store cohort |
| Like-for-Like PY Net Sales | PY sales from that same location cohort | Provides the matching comparison, not a different cohort |
| Top N Department Net Sales | Returns a selected top department or the combined Other remainder | Supports the Top N + Other chart without extra physical measures |

#### Contribution %

```dax
DIVIDE(
    [Net Sales],
    CALCULATE(
        [Net Sales],
        ALLSELECTED(DIM_ITEM[DEPARTMENT_NAME])
    )
)
```

This is the business expression inside its Trading Metric guard.
`ALLSELECTED` removes the individual department row restriction while retaining
the external selection. This version is designed for **Department rows**;
do not assume the same denominator is correct for arbitrary row dimensions.

#### Like-for-like

The two items:

1. Capture current dates and their comparable PY dates.
2. Find stable location IDs with transactions in CY.
3. Find stable location IDs with transactions in PY.
4. Use `INTERSECT` to keep locations present in both sets.
5. Calculate CY or PY sales using that same cohort.

Store eligibility is tested across all products by removing DIM_ITEM filters.
The resulting sales still respect product selections. Location filters remain
in effect. This is an explicit business choice, not the only possible LFL policy.

LFL trading eligibility uses transactions; Active Locations uses non-zero
period Net Sales. Those two definitions intentionally answer different questions.

#### Top N + Other

The item reads `Top N[Value]`, scores eligible departments by Net Sales and uses
`TOPN`. Department ID is a secondary sort key to make ties deterministic.
`EXCEPT` identifies the remainder, which is displayed as Other.

The calculation preserves the selected period and other business filters.
Top N plus Other should reconcile to selected-period Net Sales. Other may
be larger than one individual department because it combines several.

Do not add a separate built-in Top N filter that removes the Other category.

### 5.3 Why are these items rather than separate measures?

One reporting entry point can expose all 15 business metrics, while the same
period and comparison logic can be reused for eligible metrics.

---

## 6. Time Comparison: current, PY and growth

**Table/column:** `'Time Comparison'[Comparison]`  
**Precedence:** 20  
**Items:** Current, PY, YoY, YoY %

| Item | Meaning |
| --- | --- |
| Current | Leave the selected-period metric unchanged |
| PY | Recalculate the metric on comparable previous ISO-year dates |
| YoY | Current minus PY; Discount % is expressed in percentage points |
| YoY % | `(Current − PY) / PY` |

### 6.1 PY is ISO-aligned, not simply date minus one year

The warehouse calendar supplies `PY_COMPARABLE_DATE`: previous ISO week-year,
same ISO week and weekday.

```text
Current date:     01 Jan 2026
Comparable PY:    02 Jan 2025
```

That is why this implementation does not simply use `SAMEPERIODLASTYEAR`.

The essential mapping is:

```dax
VAR DatesPY =
    FILTER(
        VALUES(DIM_DATE[PY_COMPARABLE_DATE]),
        NOT ISBLANK(DIM_DATE[PY_COMPARABLE_DATE])
    )
RETURN
    CALCULATE(
        SELECTEDMEASURE(),
        REMOVEFILTERS(DIM_DATE),
        TREATAS(DatesPY, DIM_DATE[CALENDAR_DATE])
    )
```

This is an explanatory extract. The actual items also contain eligibility,
coverage, blank-handling and ratio logic. Keep those checks in the real model.

### 6.2 Why remove calendar filters before applying PY dates?

If January 2026 remained filtered while PY dates from 2025 were applied,
the intersection could be empty. The original calendar context must be replaced,
while Department, Region and other business filters are retained.

### 6.3 Ratios and missing data

ABV, UPT, ASP, Discount % and Sales per Location are recalculated in the PY
context. PY ABV, for example, means PY Net Sales / PY Transactioncount.

Comparison coverage requires a complete mapped date set inside the imported
data range. Incomplete coverage returns blank, including unmatched ISO-week
cases. For additive metrics, missing sales in an otherwise covered comparison
can be treated as zero; missing ratio denominators are not invented.

These are mapping/range checks, not proof that every source file was loaded.
They assume the imported dataset has passed its upstream completeness checks.

### 6.4 Percentage points are not relative percentage growth

```text
Discount rate CY = 9%
Discount rate PY = 8%

YoY difference = +1 percentage point
YoY %          = (9% − 8%) / 8% = +12.5%
```

The Discount % YoY item multiplies the decimal-rate difference by 100 so it
can be formatted as `pp` or `pts YoY`. The Discount KPI uses this point change,
not the relative YoY % used by the four other main KPI comparisons.

### 6.5 Intentionally blocked combinations

Generic PY/YoY items return blank for Contribution %, Top N Department Net Sales
and the Like-for-Like items. LFL has its own explicit matching CY/PY items;
applying generic PY would rebuild the comparison around a different period.

---

## 7. Period Scope: which date window?

**Table/column:** `'Period Scope'[Scope]`  
**Precedence:** 30

| Item | Dates used |
| --- | --- |
| Selected QTD | Selected quarter start through the earlier of quarter end and data cutoff |
| Previous 2 Quarters | Six months before selected quarter start through the day before that start |
| Quarter Running Total | Selected quarter start through the date currently being evaluated |

For a selected 2026 Q1:

```text
Selected quarter:       01 Jan–31 Mar 2026
Previous two quarters:  01 Jul–31 Dec 2025
```

For a selected 2026 Q4 with a data cutoff of 18 November:

```text
Selected QTD:           01 Oct–18 Nov 2026
Previous two quarters:  01 Apr–30 Sep 2026
```

### 7.1 Preserving each month/week cell

Selected QTD and Previous 2 Quarters capture `VALUES(DIM_DATE[CALENDAR_DATE])`
inside the cell before clearing calendar filters. They intersect those dates
with the desired window and reapply the resulting date set.

This is why January and February do not both repeat the full quarter value.
An ISO week crossing two calendar months contributes only the dates belonging
to each month; the underlying sales are not duplicated in the quarter total.

### 7.2 The actual running-total expression

The `Quarter Running Total` calculation item is:

```dax
IF(
    NOT ISSELECTEDMEASURE([Trading Metric]),
    SELECTEDMEASURE(),

    VAR QStart = SELECTEDVALUE('Quarter Selector'[Quarter Start])
    VAR QEnd = SELECTEDVALUE('Quarter Selector'[Quarter End])
    VAR Cutoff = MAX('Report Settings'[As Of Date])
    VAR PointDate = MAX(DIM_DATE[CALENDAR_DATE])
    VAR DatesToPoint =
        FILTER(
            ALL(DIM_DATE),
            DIM_DATE[CALENDAR_DATE] >= QStart
                && DIM_DATE[CALENDAR_DATE] <= PointDate
        )
    RETURN
        IF(
            HASONEVALUE('Quarter Selector'[Quarter Key])
                && PointDate >= QStart
                && PointDate <= MIN(QEnd, Cutoff),
            CALCULATE(
                SELECTEDMEASURE(),
                REMOVEFILTERS(DIM_DATE),
                DatesToPoint
            )
        )
)
```

Read it in this order:

1. Read the selected quarter's boundaries.
2. Read the current plotted/table date.
3. Build all calendar dates from quarter start through that date.
4. Replace the single-date calendar filter with this cumulative range.
5. Keep product/location selections.
6. Return blank outside the selected quarter or after the reporting cutoff.

```text
Daily sales:       100, 150, 200
Running sales:    100, 250, 450
```

Clearing the whole Date table also removes month filters that could otherwise
incorrectly reset a quarter running total at the start of each month.

**Important:** PY maps this already-limited current-period date window.
Both CY and PY therefore use the same elapsed-period cutoff.

---

## 8. Running Columns: the original running-table adapter

**Table/column:** `'Running Columns'[Column]`  
**Precedence:** 40

| Original item | Scope selected | Comparison selected |
| --- | --- | --- |
| CY Value | Selected QTD | Current |
| CY Running Total | Quarter Running Total | Current |
| PY Running Total | Quarter Running Total | PY |
| YoY Running Total | Quarter Running Total | YoY |
| YoY % Running Total | Quarter Running Total | YoY % |

It turns lower-level scope/comparison selections into useful named columns.

**Why retain it after creating Running Layout?** The current company-format
tables use Running Layout instead, but the retained group still participates
in the `[YoY Status]` colour bridge for the new YoY % column.

It also preserves the earlier general-purpose running-column definitions.
Do not delete it simply because its column is absent from the latest tables.

---

## 9. Template Columns: the approved QTD layout

**Table/column:** `'Template Columns'[Column]`  
**Precedence:** 50

The company template does not repeat Current/PY/YoY/YoY % under every metric.
It mixes selected combinations in one row. This group creates those combinations.

### 9.1 Numeric table items

| Template item | Metric selected | Comparison selected |
| --- | --- | --- |
| Net Sales | Net Sales | Current |
| PY Net Sales | Net Sales | PY |
| YoY | Net Sales | YoY |
| YoY % | Net Sales | YoY % |
| Units | Units | Current |
| Transaction_count | Transactioncount | Current |
| ABV | ABV | Current |
| Disc % | Discount % | Current |

The visual supplies Selected QTD or Previous 2 Quarters as its scope.

For example, the actual PY Net Sales item is:

```dax
IF(
    ISSELECTEDMEASURE([Trading Metric]),
    IF(
        ISINSCOPE(DIM_DATE[QUARTER_LABEL]),
        CALCULATE(
            SELECTEDMEASURE(),
            REMOVEFILTERS('Metric'),
            REMOVEFILTERS('Time Comparison'),
            'Metric'[Metric] = "Net Sales",
            'Time Comparison'[Comparison] = "PY"
        )
    ),
    SELECTEDMEASURE()
)
```

The `ISINSCOPE` guard blanks the extra overall grand total outside the quarter
hierarchy. Quarter/month subtotal cells still recalculate normally.

**Consequence:** these numeric layout items are intended for the QTD hierarchy
matrix, not a stand-alone card with no quarter row in scope.

### 9.2 Header and formatting items

| Item | Purpose |
| --- | --- |
| Quarter Header | Returns the selected quarter label and its full start/end dates as text |
| KPI Change | Keeps the value expression, but changes its format to include arrows and YoY text |

Quarter Header displays the **full quarter boundaries**, not necessarily the
partial-quarter data cutoff. The calculations still respect Reporting Through.

KPI Change formats ordinary growth as `▲ +0.0% YoY` / `▼ -0.0% YoY`.
For Discount %, it uses `pts YoY` because that KPI's comparison is point change.

### 9.3 The six slicer-caption items

- Slicer Quarter
- Slicer Department
- Slicer Class
- Slicer Region
- Slicer District
- Slicer Location Type

These return text such as `Department: All` or `Quarter: 2026 Q1`.
They use `ISFILTERED`, `FILTERS` and `CONCATENATEX` to describe explicit choices.

They are **captions**, not the filtering mechanism. The native dropdowns and
their interactions apply the selections. Caption cards support the company's
inline filter appearance; the dropdown arrow remains the control to use.

---

## 10. Running Layout: the company running-total format

**Table/column:** `'Running Layout'[Column]`  
**Precedence:** 60

This is the page-specific adapter added for the supplied Running Total format.
It has four table items and two chart-series items.

### 10.1 The four table items

| Item | Result | Lower-level selections |
| --- | --- | --- |
| CY Value | Value for that individual day | Selected QTD + Current |
| CY Run Tot | Quarter-to-date cumulative value at that day | Quarter Running Total + Current |
| PY Run Tot | Cumulative ISO-comparable PY value | Quarter Running Total + PY |
| YoY % | Growth between the two running totals | Quarter Running Total + YoY % |

Together with the Date row header, these produce exactly five displayed columns.

Each item:

1. Reads the row's `'Running Dates'[Date]` into `_Point`.
2. Checks that it is a single date within the quarter/reporting cutoff.
3. Clears DIM_DATE filters and maps `_Point` to `DIM_DATE[CALENDAR_DATE]` using TREATAS.
4. Chooses the required scope and comparison.
5. Lets the existing Metric item calculate Net Sales or Units.

For example, the actual CY Run Tot item is:

```dax
IF(
    ISSELECTEDMEASURE([Trading Metric], [YoY Status]),
    VAR _Point = SELECTEDVALUE('Running Dates'[Date])
    VAR _Start = SELECTEDVALUE('Quarter Selector'[Quarter Start])
    VAR _End =
        MIN(
            SELECTEDVALUE('Quarter Selector'[Quarter End]),
            MAX('Report Settings'[As Of Date])
        )
    RETURN
        IF(
            HASONEVALUE('Running Dates'[Date])
                && _Point >= _Start
                && _Point <= _End,
            IF(
                ISSELECTEDMEASURE([YoY Status]),
                "#303638",
                CALCULATE(
                    SELECTEDMEASURE(),
                    REMOVEFILTERS(DIM_DATE),
                    TREATAS({_Point}, DIM_DATE[CALENDAR_DATE]),
                    REMOVEFILTERS('Period Scope'),
                    REMOVEFILTERS('Time Comparison'),
                    'Period Scope'[Scope] = "Quarter Running Total",
                    'Time Comparison'[Comparison] = "Current"
                )
            )
        ),
    SELECTEDMEASURE()
)
```

The YoY % item additionally routes `[YoY Status]` through the retained
Running Columns item `YoY % Running Total`. That reuses the existing colour
logic after mapping the helper date to the real calendar.

The other three columns get ordinary dark text. The date guard also avoids
meaningless sums of cumulative values in an extra total row.

### 10.2 Chart items: Current Year and Past Year

The chart X-axis is `DIM_DATE[Running Day of Quarter]`, not a date label.
Day 20 occurs in many quarters, so the item first reconstructs the actual date:

```dax
VAR _Start = SELECTEDVALUE('Quarter Selector'[Quarter Start])
VAR _Day = SELECTEDVALUE(DIM_DATE[Running Day of Quarter])
VAR _Date = _Start + _Day - 1
```

It then maps that date onto the real calendar and selects Quarter Running
Total plus Current or PY. This avoids incorrectly using the maximum calendar
date from all the years that happen to contain day 20.

**Chart scaling is explicit:**

```dax
DIVIDE(
    _Value,
    IF(SELECTEDVALUE('Metric'[Metric]) = "Units", 1000, 1000000)
)
```

- Net Sales chart values are in millions.
- Units chart values are in thousands.
- Tables still show the unscaled amounts/counts.

For example, table sales of 107,125.50 plot as 0.1071255 on a Net Sales (M) axis.
This is a deliberate presentation conversion; the fact and base measures
are unchanged. Do not apply another millions/thousands division on the axis.

These chart items are designed for Net Sales and Units. Extend the scaling
logic deliberately before using them for a different metric.

### 10.3 Why a new layout group instead of editing the QTD design?

It isolates the running-page labels, date mapping and chart scaling. The
approved QTD visual definitions and existing calculation items did not need
to be rewritten.

---

## 11. Helper tables and calculated columns

Helper tables are not extra sales facts. Most are intentionally disconnected
from the sales star and are read explicitly by DAX.

### 11.1 Report Settings

This calculated table stores:

| Column | Definition |
| --- | --- |
| Data Start Date | Minimum imported transaction date |
| Data End Date | Maximum imported transaction date |
| As Of Date | Earlier of TODAY() and Data End Date |

Its actual table expression is:

```dax
VAR _FirstDate = MIN(FCT_SALES[TRANSACTION_DATE])
VAR _LastDate = MAX(FCT_SALES[TRANSACTION_DATE])
RETURN
    ROW(
        "Data Start Date", _FirstDate,
        "Data End Date", _LastDate,
        "As Of Date", MIN(TODAY(), _LastDate)
    )
```

**Why?** The report should not calculate through dates beyond the imported data.
These values are global reporting settings, not recalculated per Department
selection. As a calculated table, it updates during model processing/refresh,
not on every slicer click. A maximum transaction date is not a production
publication/completeness watermark by itself.

### 11.2 Quarter Selector

This calculated table contains quarters overlapping the imported reporting
range and stores Quarter Key, Quarter, Quarter Start, Quarter End and Reporting
Through. Reporting Through is the earlier of quarter end and As Of Date.

**Why not directly use DIM_DATE[QUARTER_LABEL] as the slicer?** A related
quarter filter would remove preceding quarters before the second QTD matrix
could display them. The selector instead supplies parameters to Period Scope.

It does not have a filtering relationship to DIM_DATE or FCT_SALES.
Its one helper relationship is explained below.

### 11.3 Running Dates

Its table expression is:

```dax
SELECTCOLUMNS(
    DIM_DATE,
    "Date", DIM_DATE[CALENDAR_DATE],
    "Quarter Key", DIM_DATE[QUARTER_KEY]
)
```

It contains the calendar date list at refresh. It is **not rebuilt for each
slicer choice**. The following one-way relationship restricts its rows at
query time:

```text
Quarter Selector[Quarter Key]  1 ──► * Running Dates[Quarter Key]
```

There is deliberately no relationship from Running Dates to DIM_DATE or
FCT_SALES. Running Layout uses TREATAS to map a displayed helper date to the
real calendar only when evaluating a running-page value.

**Why create this table?** The daily tables can start with only the selected
quarter's dates instead of evaluating the full multi-year calendar, including
formatting, before removing irrelevant rows. For 2026 Q1 that means 90 dates.

This improves query behaviour without changing the QTD page's disconnected
quarter-selection design. Do not connect this helper to the sales star as
a shortcut; that could make the selected quarter filter away preceding periods.

### 11.4 Top N

```dax
GENERATESERIES(1, 10, 1)
```

This disconnected numeric parameter supplies N to the Top N metric item.
`SELECTEDVALUE('Top N'[Value], 3)` provides a default of 3. It does not require
an additional parameter-value measure.

### 11.5 Department Axis

This disconnected table contains Department ID/name pairs plus a synthetic
`__OTHER__` ID labelled Other. The Top N calculation maps the selected axis
category to real DIM_ITEM departments using TREATAS.

**Why a separate axis?** Other is not an actual department in the source data.
It is a reporting category representing the remainder.

### 11.6 Additional calendar/display columns

| Column | Where | Purpose |
| --- | --- | --- |
| ISO Year Week | DIM_DATE | Unambiguous label such as 2026-W01 for general analysis |
| Template Month | DIM_DATE | Company label such as Jan 2026, sorted by year/month key |
| Template Week | DIM_DATE | Company label such as W01; adds the ISO year at calendar-year boundaries |
| Template Week Sort | DIM_DATE | Orders boundary weeks correctly within calendar months |
| Running Day of Quarter | DIM_DATE | `DATEDIFF(Quarter Start, Calendar Date, DAY) + 1` |
| Running Cutoff Day | Quarter Selector | Day number of Reporting Through |
| Running Quarter Length | Quarter Selector | Total days in the selected quarter, for the chart X-axis limit |

The chart's dotted marker is captioned `today` to follow the supplied design,
but its position is **the reporting-through day**. For a completed historical
quarter it is at quarter end; it is not an invented point inside the quarter.

---

## 12. Precedence: how the groups cooperate

| Group | Precedence | Main responsibility |
| --- | ---: | --- |
| Metric | 10 | Business expression |
| Time Comparison | 20 | Comparable dates and growth |
| Period Scope | 30 | Current-period date window |
| Running Columns | 40 | Original running-column adapter / colour bridge |
| Template Columns | 50 | QTD company-layout adapter and captions |
| Running Layout | 60 | Current running-page adapter |

**Higher precedence has priority when groups combine.** Think of the
higher-precedence item as an outer expression that can select filters for
lower-precedence groups. This is expression composition, not a procedural
program that simply executes six steps in table order.

For a QTD PY Net Sales cell, the intended nesting is conceptually:

```text
Template Columns chooses Net Sales + PY
    Period Scope identifies the current-period dates for this row
        Time Comparison maps those dates to PY
            Metric supplies the Net Sales business expression
```

For a running table cell:

```text
Running Layout maps the helper-row date and selects scope/comparison
    Period Scope expands that date into quarter-start-through-point dates
        Time Comparison keeps them as CY or maps them to PY
            Metric supplies Net Sales or Units
```

Not all groups are actively selected on every visual. Normally the QTD
Template Columns layout and the Running Layout are alternatives, not two
layout groups to filter simultaneously.

Changing precedence can alter results even when each individual expression
still looks reasonable. Validate combinations, not just isolated items.

---

## 13. How each page is connected to the model

### 13.1 Approved QTD page

**Main KPI value cards:**

```text
Value:       [Trading Metric]
Metric:      Net Sales / Units / Transactioncount / ABV / Discount %
Scope:       Selected QTD
Comparison:  Current
```

**KPI change cards:** same metric/scope, with:

```text
Comparison:      YoY % for the first four; YoY for Discount %
Template item:   KPI Change
Font colour:    [YoY Status]
```

**Current-quarter matrix:**

```text
Rows:     DIM_DATE[QUARTER_LABEL] → [Template Month] → [Template Week]
Columns:  Template Columns[Column]
Values:   [Trading Metric]
Scope:    Selected QTD
Items:    the eight numeric company-template items
```

**Last-two-quarters matrix:** Quarter → Month rows; Previous 2 Quarters scope;
the first six numeric template items, excluding ABV and Disc %.

### 13.2 Running Total page

**Charts:**

```text
X-axis:   DIM_DATE[Running Day of Quarter]
Y-axis:   [Trading Metric]
Series:   Running Layout[Column], limited to Current Year and Past Year
Metric:   Net Sales for one chart, Units for the other
```

**Daily tables:**

```text
Rows:     Running Dates[Date]
Columns:  Running Layout[Column]
Values:   [Trading Metric]
Items:    CY Value, CY Run Tot, PY Run Tot, YoY %
Metric:   Net Sales for one table, Units for the other
Colour:   [YoY Status], routed by Running Layout
```

Use **Running Dates[Date]** for these tables. Simply swapping in
DIM_DATE[CALENDAR_DATE] bypasses the helper-row mapping expected by the items.

### 13.3 Department Analysis

- Top N chart: Department Axis labels, Metric = Top N Department Net Sales.
- Contribution/LFL matrix: real department rows and the applicable Metric items.
- Scope = Selected QTD; Comparison = Current for the advanced items.

---

## 14. Walk through one table cell

Imagine the Net Sales table row for 2 January in a selected 2026 Q1.

### CY Value

1. The quarter selector filters Running Dates to Q1.
2. The row supplies 2 January as its helper date.
3. Running Layout maps that date to DIM_DATE and selects Selected QTD + Current.
4. Period Scope retains that one valid date.
5. Metric selects Net Sales.
6. The result is **that day's sales only**.

### CY Run Tot

The same mapping happens, but Running Layout selects Quarter Running Total.
Period Scope replaces the single-day filter with 1–2 January.

Previously validated fixture values illustrate the difference:

```text
1 Jan daily Net Sales = 1,034.90
2 Jan daily Net Sales = 1,200.20

2 Jan CY Value       = 1,200.20
2 Jan CY Run Tot     = 2,235.10
```

The table rounds these for display; the underlying amounts retain decimals.

### PY Run Tot

1. Build the same elapsed current window: 1–2 January 2026.
2. Map each date using PY_COMPARABLE_DATE.
3. Evaluate Net Sales across those mapped dates.

The validated PY running value at that point was 2,063.70.

### YoY %

```text
(2,235.10 − 2,063.70) / 2,063.70 = approximately +8.3%
```

The value comes from comparison logic. The green font comes separately from
YoY Status. These fixture figures are checks/examples, not hardcoded constants;
a different imported snapshot or filter selection can change them.

---

## 15. Formatting and slicer interactions

### 15.1 Dynamic formats do not usually turn numbers into text

Calculation items use a separate `formatStringDefinition`, for example:

```dax
IF(
    ISSELECTEDMEASURE([Trading Metric]),
    "0.0%",
    SELECTEDMEASUREFORMATSTRING()
)
```

This keeps the value numeric. Charts and sorting can still use the number.
QTD percentage changes may show triangle symbols via the format string;
the running tables use signed percentages without triangles to match their design.

`FORMAT` is used appropriately for text labels, such as dates in Quarter
Header and the calculated month/week labels. The chart M/K conversion is a
separate explicit numeric scaling operation, as explained above.

### 15.2 Colour and direction are different concepts

```text
Sales up        → up triangle, green
Sales down      → down triangle, red
Discount % up   → up triangle, red
Discount % down → down triangle, green
```

The sign is not reversed just to obtain a favourable colour.

### 15.3 Slicer independence is configured in the report

Single-direction star relationships are not sufficient to prevent two slicers
on the same dimension from reducing one another's options.

The report uses visual interactions:

- Slicer → other slicer: None.
- Slicer → intended metric visual: Filter.
- Slicer → its own inline caption: Filter.
- Other slicers → that caption: None.
- Chart/table → slicer: None.

Quarter, Department, Region and Location Type use the shared synchronization
groups across the relevant pages. Class and District are not silently carried
into Running Total through hidden synchronized slicers.

The chosen business filters still intersect when calculating values. An
incompatible Department/Class combination can legitimately produce no sales.

---

## 16. Follow-along learning exercises

Use a copy of the project if you want to experiment. The approved pages do
not need to be redesigned to learn the model.

### Exercise 1 — Find the three base formulas

1. Open Model view → Model explorer.
2. Find Net Sales and Transactioncount under FCT_SALES.
3. Find Units under DIM_DATE.
4. Read the formulas and explain why COUNTROWS would not replace DISTINCTCOUNT.

### Exercise 2 — Watch one reporting measure change meaning

1. On a temporary page in the copy, add a matrix.
2. Put `Metric[Metric]` in Rows and `[Trading Metric]` in Values.
3. Inspect Net Sales, Units and ABV.
4. Notice that you did not create a separate value field for every row.

Without a Period Scope/quarter selection, this is a base diagnostic view of
the filtered imported data, not automatically the selected quarter.

### Exercise 3 — Add the date-window layer

1. Add the Quarter Selector slicer and select one quarter.
2. Filter Period Scope to Selected QTD.
3. Filter Time Comparison to Current.
4. Confirm that Trading Metric now responds to the quarter.

### Exercise 4 — Compare correctly

1. Put Time Comparison[Comparison] in matrix Columns.
2. Keep a core metric such as Net Sales or ABV.
3. Compare Current, PY, YoY and YoY %.
4. Explain why ABV PY is recalculated from PY sales/transactions.

### Exercise 5 — Trace the actual running table

1. Select the Net Sales running table in the copied report.
2. Confirm Rows = Running Dates[Date], Columns = Running Layout[Column].
3. Confirm Values = Trading Metric and the Metric filter = Net Sales.
4. Open Running Layout → CY Run Tot and locate TREATAS.
5. Open Period Scope → Quarter Running Total and locate DatesToPoint.
6. Explain why the first maps a date while the second expands a date range.

### Exercise 6 — Prove that chart scaling is presentation-specific

1. Inspect a running table amount.
2. Inspect its corresponding chart endpoint.
3. Convert the chart's M/K units back to raw values.
4. Explain why changing the base Net Sales/Units measures to divide by M/K
   would incorrectly scale other visuals as well.

### Exercise 7 — Inspect the isolated relationship

Find Quarter Selector → Running Dates. Confirm there is no relationship from
that helper to DIM_DATE or FCT_SALES. Explain how the running table still
filters sales through its explicit TREATAS mapping.

---

## 17. Maintenance rules and troubleshooting

| Symptom/change | What to check |
| --- | --- |
| Quarter slicer appears ineffective | Is the visual using Trading Metric plus an appropriate scope/layout item? Base measures deliberately bypass these transformations. |
| Previous two quarters disappear | A related DIM_DATE quarter/page filter may be intersecting away the required dates. |
| QTD layout item is blank in a card | Numeric Template Columns items require the quarter hierarchy to be in scope. Use Metric/Scope/Comparison directly for ordinary cards. |
| Running table is blank | Use Running Dates[Date], select one quarter and the correct four Running Layout items. Check that calculated objects are processed. |
| Running chart is blank or wrong | Use Running Day of Quarter and the Current Year/Past Year Running Layout items together. |
| Running chart amount seems one million times too small | Read the Net Sales (M) axis; table values are intentionally unscaled. Avoid double scaling. |
| PY is blank | Check imported date-range coverage and ISO mapping before assuming the calculation is broken. |
| Running total resets each month | A month/date filter may remain in place; preserve the full Date-table filter replacement in Period Scope. |
| Colour is wrong after changing a metric | Review the matching business formula in YoY Status and the layout's colour routing. |
| Want to remove Running Columns | It is still referenced by the Running Layout colour bridge; inspect dependencies first. |
| Add a new business metric | Add its Metric item and format; consider ratio/PY rules and colour-helper support before adding it to a layout. |
| Change a company column label | Update the relevant layout adapter/visual mapping, not the physical base measure unnecessarily. |
| Change a calculation-group precedence | Test combined scope + comparison + metric results and formats; do not treat precedence as cosmetic. |

Additional rules:

- Keep stable LOCATION_ID for store cohorts and counts; avoid counting SCD versions.
- Keep actual trading-date logic separate from load timestamps.
- Do not use FORMAT to turn core numeric KPIs into text.
- Do not replace missing history with fabricated PY values.
- Do not sum running-total rows to produce a quarter total; use the final valid point.
- Do not alter the approved QTD page while changing the running-page layout.
- Calculated tables/columns update at processing/refresh; measures/items evaluate
  under query filter context. Refresh timing and slicer timing are different.
- New calculated objects can need model calculation before display. This is
  different from intentionally refreshing the imported Snowflake business data.

---

## 18. A short explanation you can present

> “The report uses a sales star schema with Date, Item and Location filtering
> the sales fact. I kept three base business measures and used Trading Metric
> as the reporting entry point. Calculation groups select the metric, the
> date window and the comparison, so I do not need separate measures for
> every CY/PY/running combination. Two layout groups then arrange those results
> into the company's QTD and Running Total formats. A separate colour helper
> handles favourable and unfavourable changes. The running tables use an
> isolated date list mapped back to the real calendar, which keeps their
> queries focused without changing the approved QTD page.”

For just the running total:

> “For each displayed day, I map it to the selected quarter's calendar date.
> CALCULATE then evaluates the metric over all dates from quarter start
> through that day instead of only that day's sales. PY uses the same elapsed
> window, mapped to comparable ISO dates.”

---

## 19. Where to find the full source

In Desktop, use **Model view → Model explorer** to inspect measures and
calculation items. File names below are relative to this guide in Documents.

| Object | Source file |
| --- | --- |
| Net Sales, Transactioncount, Trading Metric, YoY Status | [FCT_SALES.tmdl](report.SemanticModel/definition/tables/FCT_SALES.tmdl) |
| Units and calendar/display helper columns | [DIM_DATE.tmdl](report.SemanticModel/definition/tables/DIM_DATE.tmdl) |
| Business metric items | [Metric.tmdl](report.SemanticModel/definition/tables/Metric.tmdl) |
| Current/PY/YoY comparison items | [Time Comparison.tmdl](<report.SemanticModel/definition/tables/Time Comparison.tmdl>) |
| Selected/preceding/running date windows | [Period Scope.tmdl](<report.SemanticModel/definition/tables/Period Scope.tmdl>) |
| Original running-column adapter | [Running Columns.tmdl](<report.SemanticModel/definition/tables/Running Columns.tmdl>) |
| Approved QTD layout and captions | [Template Columns.tmdl](<report.SemanticModel/definition/tables/Template Columns.tmdl>) |
| Current running-page layout | [Running Layout.tmdl](<report.SemanticModel/definition/tables/Running Layout.tmdl>) |
| Global reporting dates | [Report Settings.tmdl](<report.SemanticModel/definition/tables/Report Settings.tmdl>) |
| Quarter selection and chart-boundary helpers | [Quarter Selector.tmdl](<report.SemanticModel/definition/tables/Quarter Selector.tmdl>) |
| Isolated running date list | [Running Dates.tmdl](<report.SemanticModel/definition/tables/Running Dates.tmdl>) |
| Top N parameter | [Top N.tmdl](<report.SemanticModel/definition/tables/Top N.tmdl>) |
| Department/Other labels | [Department Axis.tmdl](<report.SemanticModel/definition/tables/Department Axis.tmdl>) |
| Relationships | [relationships.tmdl](report.SemanticModel/definition/relationships.tmdl) |
| Model table references and options | [model.tmdl](report.SemanticModel/definition/model.tmdl) |

**Suggested reading order:** base measures → Metric → Period Scope → Time
Comparison → Template Columns → Running Layout → Running Dates/relationships
→ YoY Status and the retained Running Columns bridge → advanced Metric items.

When a label appears to be a “measure” on screen, first check whether it is
actually a physical measure, a calculation item, a renamed field or a caption.
That distinction is the key to understanding this report.
