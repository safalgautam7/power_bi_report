# Official company template

Open `report.pbip`, with `report.Report` and `report.SemanticModel` alongside it.

## Current layout

- 1920 × 1080 landscape, Fit to Page; no vertical page scrolling.
- Existing logo; centered title and dynamic selected-quarter date line.
- Six inline filters in the supplied order. The native dropdown chevron on
  the right opens each filter. The Quarter filter uses the teal treatment.
- Five cream KPI blocks with orange top rules: Net Sales, Units,
  Transaction_count, ABV and Discount %.
- Four KPI comparisons use YoY %. Discount % uses percentage-point change
  (`pts YoY`); increases are adverse/red, decreases favourable/green.
- Current-quarter table: Period, Net Sales, PY Net Sales, YoY, YoY %, Units,
  Transaction_count, ABV, Disc %.
- Last-two-quarters table: Period, Net Sales, PY Net Sales, YoY, YoY %, Units,
  Transaction_count. Quarters are newest first; months are chronological.
- Month labels are `Jan 2026`; week labels are `W01`. At calendar/ISO-year
  boundaries the ISO year is added to remove ambiguity and retain correct
  ordering. January starts expanded; other months can be expanded using +.
- Teal table headers, cream alternating rows, orange rules, and coloured
  changes/triangles. Extra combined grand totals are blanked because the
  quarter subtotal already supplies the required total.
- Additional analysis is on `Department Analysis`, not on the official
  dashboard. `Running Total` remains a separate required page.

Default quarter is **2026 Q1**, as in the reference. Values remain calculated
from the imported data, not copied from the template's illustrative figures.
Quarter selection is synchronized across pages; other slicers do not filter
one another. Use the right-side chevrons to operate the inline filters.

## Data and validation

All four Snowflake import partitions, the original local imported cache,
logo/theme assets and PBIP pointer were preserved byte-for-byte relative to
the latest backup. No source refresh was issued.

Verified using Desktop's local DAX engine:

| Item | Result |
| --- | ---: |
| Imported sales lines | 86,652 |
| Full imported Net Sales | 941,254.40 |
| 2026 Q1 Net Sales | 107,125.50 |
| ISO-aligned PY Net Sales | 99,461.50 |
| YoY | 7,664.00 |
| YoY % | 7.705494% |
| Units | 29,625 |
| Transactioncount | 9,841 |
| ABV | 10.885632 |
| Discount % | 0.210895% |

The physical measure remains named `Transactioncount`; only its displayed
template caption is `Transaction_count`. There are still five physical
measures. A fifth calculation group, `Template Columns`, supplies the mixed
table columns and text captions without creating dozens of measures.

Some older PY periods have incomplete coverage because the imported sales
start on 9 November 2024. Those comparisons intentionally remain blank.
The tables include every actual month/week, so their internal scrollbars can
appear when there are more rows than the illustrative template shows. No
business rows were suppressed just to reproduce the sample's row count.

The Desktop rendering was inspected and the new TMDL/PBIR definitions were
structurally validated. Native dropdown arrows, expand/collapse controls,
and authoring-only controls are Power BI UI, not copied screenshot artwork.
The browser full-screen notification in the reference is not report content.
Final corporate pixel-level approval remains a visual review task.

## Latest backup

`report_before_official_template_20261125T114055Z` contains the complete saved
project immediately before this template revision, including the local cache
and a SHA-256 manifest. `report_backup_20261125T111221Z` is the original
pre-build backup. Close Desktop before restoring all three linked components.
