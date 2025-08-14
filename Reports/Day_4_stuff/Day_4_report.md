# Day 4 Report — Time Intelligence, Trends, Comparison KPIs, and “Problem Titles”

**Date:** 2025-08-13  
**Goal:** Add time-aware measures (MTD/YTD/PM/PY/YoY), build a monthly trend, create comparison KPI cards, and surface “Problem Titles” with simple rules and formatting.

---
# Project Goal (restated)
Build a lightweight Library Insights Dashboard in Excel that shows what’s being checked out, who’s using the library, and how inventory is performing—using Power Query for data prep, Power Pivot / Data Model for relationships & DAX measures, and PivotTables/Charts with slicers & a timeline for interactive analysis.

# What’s Been Done So Far
Data & Setup
- Created a clean repo/folder structure and an Excel workbook LibraryDashboard.xlsx.
- Imported Books, Members, and Checkouts CSVs with Power Query, fixed data types, and added a BasePath parameter so file paths don’t break.

Cleaning & Helper Columns
- Added DaysOut in Checkouts_tbl: Duration.Days(ReturnDate - OutDate) (blank when not returned).
- Trimmed/standardized key text columns where needed.

Data Model
- Loaded all tables to the Data Model.
- Created relationships:
    - Books_tbl[ISBN] (1) → Checkouts_tbl[ISBN] (*)
    - Members_tbl[MemberID] (1) → Checkouts_tbl[MemberID] (*)
- Built a proper Calendar_tbl (continuous date range from all transaction dates) and marked it as Date Table; related Calendar_tbl[Date] → Checkouts_tbl[OutDate].
- Added a small Books_dim (reference table) to use Branch/Genre as clean slicers without double-counting.

Core DAX Measures
- Total Checkouts = DISTINCTCOUNT(Checkouts_tbl[TxnID])
- Total Copies = SUM(Books_tbl[CopiesOwned])
- Average Days Out = AVERAGEX(FILTER(Checkouts_tbl, NOT ISBLANK([DaysOut])), [DaysOut])
- Overdue Count = late returns + currently late
- Overdue % = DIVIDE([Overdue Count], [Total Checkouts])
- Turnover Rate = DIVIDE([Total Checkouts], [Total Copies])

Reports & Interactivity
- Main Pivot with Genre and values (checkouts, copies, turnover, avg days, overdue %).
- Slicers for Branch (dimension), MemberType, Genre; Timeline on Calendar_tbl[Date].
- A “Top 10 Titles” view and a clustered column chart, all wired to slicers/timeline.


# Today’s Activity (bulleted)
- Created/confirmed the base measure Last Checkout Date:
    - CALCULATE( MAX( Checkouts_tbl[OutDate] ) )

- Built Days Since Last Checkout (robust, no VAR needed):
    - IF( ISBLANK([Last Checkout Date]), BLANK(), DATEDIFF([Last Checkout Date], TODAY(), DAY) )
    - Ensured it was entered in the Calculation Area (measure), not as a calculated column.
    - Formatted as Whole Number; sanity-checked in a quick pivot by Title.

- Troubleshot the “syntax/VAR” error by:
    - Verifying the dependent measure existed.
    - Confirming the measure was created in the bottom grid (Calculation Area).
    - Avoiding VAR/RETURN for maximum compatibility.

- Verified the measure respects slicers and the timeline (it recalculates per current filter context).

- Kept dashboard tidy: consistent number formats, meaningful display names, and checked that Top 10 & charts respond to filters.

## 1) Time-Intelligence Measures (Power Pivot → Calculation Area)

> These rely on `Calendar_tbl` being marked as the **Date Table** and related to `Checkouts_tbl[OutDate]`.  
> Create the measures in **Power Pivot → Data View → Calculation Area** (home table `Checkouts_tbl` is fine).

Add the following to Checkout_tbl calculation area 

### 1.1 Month-to-Date, Prior Month, and MoM %

```DAX
Total Checkouts MTD :=
CALCULATE ( [Total Checkouts], DATESMTD ( Calendar_tbl[Date] ) )

Total Checkouts PM :=
CALCULATE ( [Total Checkouts], DATEADD ( Calendar_tbl[Date], -1, MONTH ) )

Total Checkouts MoM % :=
VAR Curr = [Total Checkouts]
VAR Prev = [Total Checkouts PM]
RETURN IF ( ISBLANK ( Prev ), BLANK(), DIVIDE ( Curr - Prev, Prev ) )
```

Format:
- MTD, PM → Number (0)
- MoM % → Percentage (1 decimal)

### 1.2 YTD, Prior Year, PYTD, and YoY %
```
Total Checkouts YTD :=
TOTALYTD ( [Total Checkouts], Calendar_tbl[Date] )

Total Checkouts PY :=
CALCULATE ( [Total Checkouts], SAMEPERIODLASTYEAR ( Calendar_tbl[Date] ) )

Total Checkouts PYTD :=
CALCULATE ( [Total Checkouts YTD], SAMEPERIODLASTYEAR ( Calendar_tbl[Date] ) )

Total Checkouts YoY % :=
VAR Curr = [Total Checkouts]
VAR Prev = [Total Checkouts PY]
RETURN IF ( ISBLANK ( Prev ), BLANK(), DIVIDE ( Curr - Prev, Prev ) )
```
Format:
- YTD, PY, PYTD → Number (0)
- YoY % → Percentage (1 decimal)


### 1.3 Rolling 30 Days (handy for a small trend card)
Total Checkouts 30D :=
CALCULATE (
    [Total Checkouts],
    DATESINPERIOD ( Calendar_tbl[Date], MAX ( Calendar_tbl[Date] ), -30, DAY )
)
Format: Number with 0 decimal places


## Why these matter:
- MTD/PM/MoM % compare the current month against last month to show direction of change.
- YTD/PY/PYTD/YoY % answer “Are we up or down vs last year?”—classic leadership questions.
- 30D gives a short-window pulse that ignores month boundaries.

## 2) Monthly Trend (Total Checkouts by Month)
- Insert → PivotTable → From Data Model on Dashboard.
- Rows: Calendar_tbl[YearMonth]
    - If you prefer MonthName, ensure in Calendar: MonthName → Sort by Column → Month (or use MonthIndex).

- Values: [Total Checkouts]

- Design: Subtotals Off; Grand Totals Off.

- PivotTable Analyze → PivotChart → Line (or Clustered Column).
    - Title: Monthly Checkouts
    - Axis number format (if column): Number (0) [Number format with 0 decimal places].

- Hook it up: With the trend pivot selected → Report Connections → check your Branch slicer, Genre, Member Type, and the Calendar Date timeline.

Why it matters: You can now see seasonality, spikes, or dips at a glance and slice the trend by Branch/Genre/Member Type.

## 3) Comparison KPI Cards (This Month vs Last; YTD vs PYTD)
- Insert → PivotTable → From Data Model (same Dashboard).
- Values only (no Rows/Columns):
    - [Total Checkouts MTD]
    - [Total Checkouts PM]
    - [Total Checkouts MoM %]
    - [Total Checkouts YTD]
    - [Total Checkouts PYTD]
    - [Total Checkouts YoY %]

- Tidy the pivot:
  - Design → Subtotals: Do Not Show
  - Grand Totals: Off for Rows and Columns
  - PivotTable Analyze → Field Headers: Off
  - Increase font size (16–20). Optionally use Shapes linked to the cells for “card” visuals.

- Formatting (in the measure, preferred):
  - Numbers: 0 decimals
  - Percentages: 1 decimal

- Connect: Report Connections → link to the same slicers/timeline.

Why it matters: These are leadership-friendly indicators that show trend and momentum (up/down vs prior periods) without reading full tables.

## 4) “Problem Titles” — Stale/Overdue Focus Table
### 4.1 Helper measures
```
Last Checkout Date :=
CALCULATE ( MAX ( Checkouts_tbl[OutDate] ) )

Days Since Last Checkout :=
VAR LastDate = [Last Checkout Date]
RETURN IF ( ISBLANK ( LastDate ), BLANK(), INT ( TODAY() - LastDate ) )

Total Checkouts 90D :=
CALCULATE (
    [Total Checkouts],
    DATESINPERIOD ( Calendar_tbl[Date], MAX ( Calendar_tbl[Date] ), -90, DAY )
)
```

Format:
- Days Since Last Checkout → Number (0)
- Total Checkouts 90D → Number (0)

### 4.2 Build the pivot
- Insert → PivotTable → From Data Model (Dashboard).
- Rows: Books_tbl[Title]
- Values: [Total Checkouts], [Total Checkouts 90D], [Days Since Last Checkout], [Overdue %]
- Focus filters (pick one or both):
    - Value Filters → Days Since Last Checkout Greater Than 30 (stale rule of thumb)
    - Sort by Days Since Last Checkout Largest to Smallest
- Conditional Formatting:
  - Days Since Last Checkout: 3-color scale (green→red)
  - Overdue %: Data bars or 3-color scale; Percentage (1 decimal)
- Connect: Report Connections → attach slicers/timeline.

Why it matters: This turns raw data into action—which titles to promote/purchase more of or where to address due-date behavior.

## 5) Why Day 4 matters to the project
- Elevates the dashboard from static totals to trend-aware, comparable KPIs.
- Enables month-over-month and year-over-year storytelling, crucial for planning and performance reviews.
- Adds a pragmatic, action-oriented view (“Problem Titles”) to guide weeding, purchasing, and policy tweaks.
- Keeps everything interactive—the same slicers/timeline drive every visual, so insights stay consistent and trustworthy.