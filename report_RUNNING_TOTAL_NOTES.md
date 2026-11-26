# Running Total — company template revision

Open the existing `report.pbip`, then select **Running Total**.

## Scope protection

The approved **QTD Trading Dashboard** page was not edited. Its entire page
directory, every other existing page, report-level settings, themes, logo
resources and the PBIP pointer remain byte-for-byte unchanged against the
pre-edit backup. All existing physical measures, calculation items and
Snowflake import partitions were preserved. The original local imported
cache was not overwritten by the file edits.

## Format

- Landscape 1920 × 1080, Fit to Page, matching the supplied page proportions.
- Logo, centered Running Total title and the exact subtitle:
  `Cumulative Net Sales and Units within the selected quarter`.
- Quarter, Department, Region and Location Type filters, in that order.
  Existing selections and synchronization groups are preserved; the filters
  do not reduce one another's options. Use their right-side dropdown arrows.
- Two side-by-side charts: `Running Total – Net Sales (CY vs PY)` and
  `Running Total – Units (CY vs PY)`.
- Day-of-quarter X axes; Net Sales in millions, Units in thousands.
  Current Year is solid teal; Past Year is dashed orange. Each chart has a
  dotted reporting-cutoff marker, captioned `today` as in the supplied format.
- Two tables headed `Net Sales – Running Total` and `Units – Running Total`.
  Each has exactly **Date, CY Value, CY Run Tot, PY Run Tot, YoY %**.
- Dates use `dd MMM`; values use whole-number display formatting; YoY % uses
  a signed single-decimal percentage without triangles, with green/red text.
- Dark teal table headers, cream alternating rows, orange top rules and
  aligned rectangular panels. Tables scroll internally for additional days.

Browser fullscreen buttons, navigation arrows and browser scrollbars in the
reference screenshot are not report artwork and were not drawn into the page.

## Data semantics

The screenshot is a layout reference, not a source of financial figures.
Daily values and running totals come from the existing transaction-date model.
PY retains the existing ISO-comparable-date mapping.

The existing elapsed-period rule is preserved: both CY and PY stop at the
same reporting cutoff. PY is not extended into unelapsed days just to copy
the sample line's shape. The cutoff marker represents the selected quarter's
reporting-through day, not an invented interior point. It is at quarter end
for a completed historical quarter such as the default 2026 Q1.

Only additive, running-page helpers were introduced:

- Hidden day-of-quarter/cutoff/quarter-length columns.
- `Running Layout` calculation group for the chart series and four table
  value columns; five physical measures are still retained.
- `Running Dates`, a calculated date list filtered one-way by the disconnected
  Quarter Selector. It is intentionally **not related to DIM_DATE or FCT_SALES**.
  This bounds each daily table to the selected quarter without changing the
  QTD page's current/preceding-quarter behaviour.

## Validation

Microsoft's TMDL serializer parsed the model and the new PBIR files passed
published JSON schemas. Existing saved Desktop 2.15 visual definitions were
preserved, not downgraded. Desktop's local DAX engine verified:

| Check | Result |
| --- | ---: |
| Imported sales rows | 86,652 |
| Imported Net Sales | 941,254.40 |
| Unchanged 2026 Q1 QTD Net Sales | 107,125.50 |
| 1 Jan daily Net Sales | 1,034.90 |
| 2 Jan CY running Net Sales | 2,235.10 |
| 31 Mar CY running Net Sales | 107,125.50 |
| 31 Mar PY running Net Sales | 99,461.50 |
| 31 Mar running YoY % | 7.705494% |
| Date rows selected for 2026 Q1 | 90 |
| Formatted table grid cells for 2026 Q1 | 360 |

Also checked: a negative daily running comparison returns red, a positive one
green; Q4 chart results are blank after reporting day 49 for both series.
The new objects were calculated in memory for testing, with no Snowflake
refresh. Save in Desktop after reviewing to persist its updated local cache.

## Backup

`report_before_running_template_20261126T040254Z` contains the complete saved
project immediately before this change, including a SHA-256 manifest and the
original cache. Close Desktop before restoring the three linked components
(`report.pbip`, `report.Report`, `report.SemanticModel`) together.
