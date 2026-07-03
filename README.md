# State Employee Credit Card — Power BI Project (PBIP)

A hand-authored Power BI Project (PBIP, file-based format with TMDL semantic model and PBIR report) built from the Delaware State Employee Credit Card transaction export. No Power BI Desktop / GUI was used to build this — every TMDL, Power Query (M), and PBIR JSON file was authored directly.

## What's in this repo

```
StateEmployeeCreditCard.pbip                      <- double-click / open this in Power BI Desktop
data/
  State_Employee_Credit_Card.csv                  <- copy of the source CSV (see "Source data" below)
StateEmployeeCreditCard.SemanticModel/            <- TMDL semantic model
  definition.pbism
  definition/
    database.tmdl
    model.tmdl
    expressions.tmdl                              <- ProjectFolder parameter used by Power Query
    relationships.tmdl
    tables/
      Transactions.tmdl                           <- fact table + Power Query M + measures
      Customer.tmdl                               <- customer (proxy) dimension + Power Query M + measure
      Date.tmdl                                   <- calculated date table, marked as the model's Date Table
StateEmployeeCreditCard.Report/                   <- PBIR report
  definition.pbir
  definition/
    report.json, version.json
    pages/
      pages.json
      TransactionReportPage/                      <- "Credit Card Transaction Report"
      CustomerReportPage/                          <- "Credit Card Customer Report"
```

## Source data and real schema discovered

Source file: `State_Employee_Credit_Card.csv` (~26 MB, 196,806 transaction rows + header). This is a Delaware state-employee purchasing-card (P-Card) transaction export, **not** a consumer credit-card dataset. Profiling the raw file (`wc -l`, `head`, and a pandas pass) found these actual columns — there is no header row ambiguity, but the real column names/types differ from a typical "credit card" schema:

| Source column      | Type (raw)   | Notes discovered while profiling |
|---------------------|--------------|-----------------------------------|
| `FISCAL_YEAR`       | integer      | Single value (2019) for the whole file |
| `FISCAL_PERIOD`     | integer      | 1–12, state fiscal period |
| `DEPT_NAME`         | text         | State agency/department, 64 distinct values |
| `DIV_NAME`          | text         | Division within the department, 275 distinct values |
| `MERCHANT`          | text         | Raw merchant name, 46,578 distinct values |
| `CAT_DESCR`         | text         | Merchant category description (like an MCC description), 245 distinct values |
| `TRANS_DT`          | text (mm/dd/yyyy) | Transaction date; real data range **2018-06-01 to 2019-06-27** |
| `MERCHANDISE_AMT`   | text         | Dollar amount; **~13k values use thousands-separator commas** (e.g. `"3,427.80"`) and must be cleaned before converting to a number; range **-$29,971.39 to $180,286.63** (negative = refunds/credits); no nulls |

There is **no cardholder ID, no age/gender/demographic field, and no credit-limit field** anywhere in the source file — these are simply not part of this dataset.

## Assumptions & substitutions made (read this before trusting the "customer"/"demographic"/"credit limit" numbers)

Because the source has none of the cardholder-level fields the original ask assumed, the model uses documented proxies instead of inventing fake data:

| Requested concept | Not available because... | Proxy used instead |
|---|---|---|
| Individual cardholder / customer ID | No cardholder ID column exists | **`CustomerKey` = `DeptName \| DivName`** — each distinct Department+Division combination (291 of them) is treated as one "customer" (a cardholder cost-center group), since P-cards are issued and managed at the division level in this data |
| Credit limit | No credit-limit field exists | **`CreditLimitProxy`** on the `Customer` table = 3× the largest single transaction ever observed for that department/division. This is a rough, clearly-labeled estimate of spending capacity derived from observed behavior, not a real limit |
| Age / gender demographics | No demographic fields exist | **`DeptName` (department/agency)** is used as the demographic-equivalent bucket everywhere a demographic split was requested (histogram, trend-by-gender chart, slicer). This is the substitution explicitly suggested when demographic columns are absent |
| Average transaction frequency | No per-person frequency exists | Computed per `CustomerKey` (department/division) as total transaction count, used both as a DAX measure (`Average Transaction Frequency` = transactions ÷ customers) and as the X-axis of the scatter chart |
| "Transaction type" for the weekly bar chart / pie chart | No separate "type" field distinct from category | `CategoryDescription` (`CAT_DESCR`) is used as the transaction type/category dimension throughout. A derived `CategoryGroup` column buckets everything outside the top 5 categories into "Other" so legends stay readable; the same top-N-bucket technique produces `DeptGroup` for the department trend chart |

All of this is implemented transparently in Power Query / DAX (not hidden) — see the column descriptions in the TMDL files or open the model in Power BI Desktop and hover over any field for its description.

## Power Query (M) transformations — `Transactions` table

Implemented entirely in `StateEmployeeCreditCard.SemanticModel/definition/tables/Transactions.tmdl`, the M script:

1. Reads the CSV from `data/State_Employee_Credit_Card.csv` via the `ProjectFolder` Power Query parameter (see "Manual steps" below).
2. Promotes headers and sets initial types.
3. Strips thousands-separator commas from `MERCHANDISE_AMT` and converts it to a proper number; parses `TRANS_DT` (mm/dd/yyyy, `en-US` locale) into a real `date`.
4. Trims text columns and renames all columns to business-friendly names (`DeptName`, `TransactionAmount`, etc.).
5. Extracts date parts into new columns: `TransactionYear`, `TransactionMonth`/`TransactionMonthName`, `TransactionWeek` (week-of-year), `TransactionWeekdayNum`/`TransactionWeekdayName`, and a sortable `YearMonth` (yyyy-MM) key.
6. Builds the `CustomerKey` proxy (`DeptName | DivName`).
7. Groups by `CategoryDescription` to compute a per-category mean and standard deviation of `TransactionAmount`, joins those statistics back onto every row, and adds an **anomaly-flagging column**: `IsAnomalyFlag` = true when a transaction's amount is more than **3 standard deviations** from its category's mean **and** the category has at least 10 observed transactions (avoids false positives in tiny categories). The signed distance is kept in `AnomalyZScore` for the conditional-formatting data bars. This flags **2,789 of 196,806 transactions (≈1.4%)**.
8. Buckets the top-5 categories and top-5 departments by volume into `CategoryGroup` / `DeptGroup` (everything else becomes `"Other"`) so chart legends stay readable.

The `Customer` table's M query references the `Transactions` query directly and groups it down to one row per `CustomerKey`, computing `TransactionCount`, `TotalSpend`, `AvgTransactionAmount`, `MaxTransactionAmount`, and the `CreditLimitProxy`. The `Date` table's M query also references `Transactions` to build a contiguous calendar (full years) spanning the real data range.

## Semantic model

- **Transactions** (fact, ~196.8k rows) — one row per card transaction, fully typed columns, hidden helper columns (`CategoryMeanAmount`, `CategoryStdDevAmount`, `CategoryTransactionCount`, month/weekday numbers) kept for sorting/anomaly math but hidden from the field list.
- **Customer** (dimension, 291 rows) — one row per department/division "customer" proxy, related to Transactions via `CustomerKey`.
- **Date** (calculated date table, marked as the official *Date Table* via `dataCategory: Time` + `IsDateTable` annotation) — contiguous daily calendar, related to Transactions via `TransactionDate` → `Date`. Enables proper time-intelligence and chronological axis sorting (`MonthName` sorted by `MonthNumber`, `WeekdayName` sorted by `WeekdayNumber`).

**Relationships:** `Date[Date]` (1) → `Transactions[TransactionDate]` (*), and `Customer[CustomerKey]` (1) → `Transactions[CustomerKey]` (*).

**DAX measures:**

| Measure | Table | DAX |
|---|---|---|
| Total Transaction Volume | Transactions | `COUNTROWS(Transactions)` |
| Total Transaction Amount | Transactions | `SUM(Transactions[TransactionAmount])` |
| Average Transaction Value | Transactions | `AVERAGE(Transactions[TransactionAmount])` |
| Total Customers | Transactions | `DISTINCTCOUNT(Transactions[CustomerKey])` |
| Average Transaction Frequency | Transactions | `DIVIDE([Total Transaction Volume], [Total Customers], 0)` |
| Anomaly Count / Anomaly Rate | Transactions | Count and % of rows where `IsAnomalyFlag = TRUE()` |
| Anomaly Background Color | Transactions | Drives the table's conditional background formatting |
| Average Credit Limit | Customer | `AVERAGE(Customer[CreditLimitProxy])` |

## Report pages

### Page 1 — "Credit Card Transaction Report"
- **Clustered column chart** — Weekly Transaction Volume by Category (`TransactionWeek` × `CategoryGroup`, top-5 categories + Other)
- **Line chart** — Transaction Amount Trend Over Time (`Date[YearMonth]` × Total Transaction Amount)
- **Pie chart** — Transaction Category Distribution (`CategoryGroup` × Total Transaction Volume)
- **Table** with conditional formatting — full transaction detail; every column's background is driven by the `Anomaly Background Color` measure (highlights anomaly-flagged rows in red), and the `AnomalyZScore` column additionally shows data bars

### Page 2 — "Credit Card Customer Report"
- **KPI cards** — Total Customers, Average Credit Limit (proxy), Average Transaction Value, Total Transaction Volume
- **Slicers** — Time Period (`Date[Date]`), Transaction Type (`CategoryDescription`), Department (demographic proxy, `DeptName`)
- **Horizontal bar chart** — Transactions by Department (demographic-equivalent histogram)
- **Line chart** — Transaction Trend by Department (gender-trend proxy; top-5 departments + Other over `Date[YearMonth]`)
- **Scatter chart** — Transaction Frequency (proxy) vs. Credit Limit (proxy) by customer, bubble size = Total Spend

## How to open in Power BI Desktop

1. Copy/clone this whole folder to the machine that has Power BI Desktop installed (keep the `.pbip` file, both `.SemanticModel`/`.Report` folders, and `data/` together).
2. Double-click `StateEmployeeCreditCard.pbip`. Power BI Desktop will open the report and the semantic model together.
3. **If the project folder path is different from `/Users/skeshav/Projects/powerbi-project`** (e.g. you copied it to Windows or a different folder), update the `ProjectFolder` Power Query parameter once: **Transform data → Manage Parameters → ProjectFolder** → set it to the new absolute path of the project's root folder (the folder containing the `data` subfolder), using forward slashes (e.g. `C:/Users/you/StateEmployeeCreditCard`). Then select **Home → Refresh**.
4. Even if the path didn't change, Power BI Desktop needs to run Power Query at least once: select **Home → Refresh** after opening so the semantic model loads data from `data/State_Employee_Credit_Card.csv`.
5. Both report pages should then render with live data.

### Manual steps / known limitations
- **Refresh required on first open** — Power Query only runs when you refresh; the PBIP ships with no cached data.
- **`ProjectFolder` parameter** — Power Query cannot express a true "relative to the .pbip" path, so a text parameter holding the absolute project folder is used instead (see step 3 above). This is the standard, documented workaround for portability in PBIP projects.
- **Large table visual** — the anomaly-detail table on page 1 is bound to all ~196.8k transaction rows; Power BI Desktop will page/virtualize it, but initial rendering may take a few seconds on first load.
- Lineage tags / GUIDs for tables and relationships were authored by hand with simple, stable placeholder values; Power BI Desktop will keep them as-is (it does not require them to look like real GUIDs).
