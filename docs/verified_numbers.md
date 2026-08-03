# Verified numbers

Every figure the workbook produces was checked against BigQuery on **2026-08-03**.
Nothing in this repository is typed in by hand — the workbook computes it, and the
warehouse confirms it.

Reproduce the control totals with [`sql/02_control_totals.sql`](../sql/02_control_totals.sql).

## Control totals — window 2024-01-01 to 2025-12-31

| Metric | BigQuery | Excel workbook | Match |
|---|---:|---:|:--:|
| Line items | 57,542 | 57,542 | yes |
| Distinct orders | 39,743 | 39,743 | yes |
| Revenue | 3,426,968.93 | 3,426,968.93 | yes |
| Profit | 1,777,550.79 | 1,777,544.97 | see note |
| Gross margin | 51.87% | 51.9% | yes |
| Median item price | 39.99 | 39.99 | yes |
| Items sold at a loss | 0 | 0 | yes |

### Note on the 5.82 profit difference

The two profit figures differ by **5.82 on 1.78M — 0.0003%**.

The export rounds `cost` and `profit` to two decimals *per line item*
(`ROUND(oi.sale_price - p.cost, 2)`), because that is what a currency column
should look like in a spreadsheet a human will read. BigQuery's control query
sums the unrounded values. Rounding 57,542 rows before summing, rather than
after, accumulates that much drift.

This is expected and correct for a reporting extract. It is recorded here rather
than quietly ignored, because "the spreadsheet and the warehouse disagree by a
few units" is exactly the kind of thing that destroys trust in a dashboard when
someone discovers it later on their own.

### Note on the order count

Counting distinct orders **per year and adding the two years** gives 39,775.
Counting distinct orders **across the whole window** gives 39,743.

The 32-order gap is orders whose line items straddle 31 December / 1 January.
Per-year counting sees them in both years. The workbook reports the window-level
figure, 39,743, and attributes each order to the year of its first line item —
so its per-year figures (15,733 + 24,010) add up to the window total.

## Headline results

| Metric | Value |
|---|---:|
| Revenue | 3,426,968.93 |
| Profit | 1,777,544.97 |
| Gross margin | 51.9% |
| Orders | 39,743 |
| Average order value | 86.23 |
| Average item price | 59.56 |
| Items per order | 1.45 |
| Line items | 57,542 |

### Year over year

| Year | Revenue | Profit | Margin | Orders | AOV |
|---|---:|---:|---:|---:|---:|
| 2024 | 1,344,840.86 | 697,549.61 | 51.9% | 15,733 | 85.48 |
| 2025 | 2,082,128.07 | 1,079,995.36 | 51.9% | 24,010 | 86.72 |
| **Growth** | **+54.8%** | +54.8% | flat | +52.6% | +1.5% |

Revenue grew 54.8%, and almost all of it came from **order volume** (+52.6%),
not from customers spending more per order (+1.5%). Margin held flat at 51.9%
across both years — growth did not come from discounting.

### Top categories

| # | Category | Revenue | Cumulative share |
|---|---|---:|---:|
| 1 | Outerwear & Coats | 406,127.85 | 11.9% |
| 2 | Jeans | 403,382.36 | 23.6% |
| 3 | Sweaters | 263,429.71 | 31.3% |
| 4 | Swim | 209,554.38 | 37.4% |
| 5 | Fashion Hoodies & Sweatshirts | 202,226.64 | 43.3% |

26 categories in total. The top 5 carry 43.3% of revenue — a long tail, not a
Pareto 80/20 curve. Assortment decisions here cannot lean on a handful of lines.

### Item price distribution

| Band | Items | Share |
|---|---:|---:|
| up to 10 | 2,769 | 4.8% |
| 10 to 25 | 14,658 | 25.5% |
| 25 to 50 | 18,145 | 31.5% |
| 50 to 75 | 9,026 | 15.7% |
| 75 to 100 | 4,896 | 8.5% |
| 100 to 150 | 4,130 | 7.2% |
| 150 to 250 | 2,931 | 5.1% |
| 250 to 500 | 863 | 1.5% |
| above 500 | 124 | 0.2% |

Percentiles: P50 = 39.99 · P75 = 69.95 · P90 = 128.27 · P99 = 299.95.

Nearly 62% of items sell below 50. The catalogue is priced for volume, and the
handful of items above 500 (0.2%) are rare enough that they should be excluded
from averages used for target-setting.

## Data quality findings

Three things in this source are worth flagging, because each one would silently
distort a report built on it:

**1. Country names are not canonical.** The `users` table contains both
`Germany` and `Deutschland`, and uses `Brasil` rather than `Brazil`. Grouping by
the raw column splits Germany across two rows and under-reports it. The workbook
carries a `country_raw -> country_clean` mapping on the `Lookups` sheet and
resolves it with INDEX/MATCH. Consolidated, Germany is **139,844.20**.

**2. Cancelled and Returned items are present in `order_items`.** They are
excluded at the source query. Leaving them in would inflate revenue with sales
that never completed.

**3. No item in this window sells at a loss** — 0 of 57,542. This is a property
of the generated dataset, not a finding about the business, and it is stated here
so nobody reads the 100% "sold at a profit" tile as a real commercial insight.
It also means margin analysis on this data cannot demonstrate loss-making
segments; the [Tableau project](https://github.com/ANAPBORGES/tableau-sales-profitability-dashboard)
uses Superstore for that, where 10 of 49 states genuinely lose money.
