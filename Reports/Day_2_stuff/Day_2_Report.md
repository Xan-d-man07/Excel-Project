# Day 2 Report – Library Insights Dashboard (Excel Project)

**Date:** 2025-07-30  
**Goal:** Connect tables, create core DAX measures, validate with a Pivot + slicers, and resolve join/key issues.

---

## 1) Relationships (Power Pivot)
In the Data tab select datamodel and Manage data model
![Select Data Model](./Day_2_screenshots/1-select_data_data-model.png)

Select and Drag the cursor from Books_tbl ISBN to Checkouts_tbl ISBN
![Link Books → Checkouts (ISBN)](./Day_2_screenshots/2-link_book_to_checkouts_isbns.png)

- **Books_tbl[ISBN] (1) → Checkouts_tbl[ISBN] (*)**
![Result of Link](./Day_2_screenshots/3-result_of_link.png)

Do the same with MemberId from Meber_tbl to Checkouts_tbl
- **Members_tbl[MemberID] (1) → Checkouts_tbl[MemberID] (*)**
- ![Link Checkouts → Members (MemberID)](./Day_2_screenshots/4-link_checkout_to_members_memberID.png)
- Verified arrows show **1 → \*** toward `Checkouts_tbl` in Diagram View.
![MemberID Link](./Day_2_screenshots/5-memberID_link.png)
  

---

## 2) Core Measures (DAX)

> Added in **Power Pivot → Data View (marked 1 on the diagram) → Calculation Area(marked 2)**.  
>This opens a spread spreadsheet at the bottom(marked 3)
>There you enter each of the below formulas in a cell.
>For ease of arrangement and organization they go from left to right
![Show Calculation Area](./Day_2_screenshots/6-calculation_area.png)



**Checkouts_tbl**
```DAX
Total Checkouts := DISTINCTCOUNT ( Checkouts_tbl[TxnID] )

Active Members  := DISTINCTCOUNT ( Checkouts_tbl[MemberID] )

Active Titles   := DISTINCTCOUNT ( Checkouts_tbl[ISBN] )

Average Days Out :=
AVERAGEX (
    FILTER ( Checkouts_tbl, NOT ISBLANK ( Checkouts_tbl[DaysOut] ) ),
    Checkouts_tbl[DaysOut]
)

Overdue Count :=
VAR TodayDate = TODAY()
RETURN
COUNTROWS (
    FILTER (
        Checkouts_tbl,
        ( NOT ISBLANK ( Checkouts_tbl[ReturnDate] ) && Checkouts_tbl[ReturnDate] > Checkouts_tbl[DueDate] )
        || ( ISBLANK ( Checkouts_tbl[ReturnDate] ) && TodayDate > Checkouts_tbl[DueDate] )
    )
)

Overdue % := DIVIDE ( [Overdue Count], [Total Checkouts] )
```

![Insert Total Copies + Format](./Day_2_screenshots/7-insert_Total_Copies_formula_change_format.png)
![Insert Active Members](./Day_2_screenshots/8-insert_Active_Members_formula.png)
![Insert Active Titles](./Day_2_screenshots/9-Active_Titles_formula.png)
![Average Days Out – pt1](./Day_2_screenshots/10-Avg_days_out_pt1.png)
![Average Days Out – pt2](./Day_2_screenshots/11-Avg_days_pt2.png)

The end results looks like this
![Finished Formulas (Checkouts)](./Day_2_screenshots/12-finished_formulas_checkout.png)

**In Books_tbl**
The same process is applied in this table
```DAX
Total Copies  := SUM ( Books_tbl[CopiesOwned] )
Turnover Rate := DIVIDE ( [Total Checkouts], [Total Copies] )
```

![Total Copies Formula](./Day_2_screenshots/13-total_copies_formula.png)
![Finished Formulas (Books)](./Day_2_screenshots/14-finished_formulas_books.png)

Formats
---
- Counts: Number (0 decimals)
- Average Days Out: Number (1–2 decimals)
- Overdue %: Percentage (1 decimal)
- Turnover Rate: % (1 decimal) or Number (2 decimals)
  
>The method to change the format of the calculations (i.e to percentages, decimals etc.) is shown in the screen shots above


## 3) Pivot + Interactivity
**Pivot setup (Insert → PivotTable → From Data Model)**
![insert pivot](./Day_2_screenshots/20-insert_pivot_table.png)

- Rows: Books_tbl -> select Genre deonted as  Books_tbl[Genre]
- Values: [Total Checkouts], [Total Copies], [Turnover Rate], [Average Days Out], [Overdue %]
- The part shown in the red marker in the picture below

![Pivot Fields](./Day_2_screenshots/15-pivot_fields.png)


- Select any cell in the pivot table which opens the 'PivotTable Analyze tab' and the Design Tab
- Design: Report Layout = Tabular, Subtotals = Do Not Show
![Tabular layout](./Day_2_screenshots/21-select_tabular_layout.png)
![NO subtotals](./Day_2_screenshots/22-select_doNot_show_subTotal.png)

**Slicers / Timeline**
>Insert Slicers
- Select any cell in the pivot table
- PivotTable Analyze -> Filter -> Insert Slicer
- Slicer: Books_tbl[Branch] (drives both inventory & transactions via the Books→Checkouts relationship)
- Slicer: Members_tbl[MemberType]
  
![Insert Slicers](./Day_2_screenshots/16-insert_slicers.png)

>Insert Timeline
PivotTable Analyze -> Filter -> Insert Timeline
- Timeline: Checkouts_tbl[OutDate]
![Insert Timeline](./Day_2_screenshots/18-insert_timeline.png)


The Final Result of the day looks like this
![Timeline Results](./Day_2_screenshots/19-timeline_results.png)



## 4) Issues Detected
The following issue was detected and resolved for the above notation to be achived, So no official documentation of the error exists
---
Issue observed: Checkout measures appeared only on the (blank) row when sliced by Books_tbl[Genre].
Root cause: Checkouts.csv used ISBN and MemberID values that did not exist in Books.csv/Members.csv.

Fix applied:

Generated a different Checkouts.csv where each row’s ISBN is sampled from Books.csv[ISBN] and MemberID from Members.csv[MemberID], and then rplaced the original file with this.

Replaced Raw_Data/Checkouts.csv with the fixed file and Refresh All. This way all the previous work done on the origianl file is carried over to the new one, without issues
![Refesh all](./Day_2_screenshots/23-refresh_all.png)

Result: all Genre rows now populate; (blank) either disappears or is minimal.

## 5)  QA Checks & Expected Behavior
Branch slicer (Books_tbl[Branch]) filters both [Total Copies] and checkout measures — confirmed.

Timeline (OutDate) affects activity measures ([Total Checkouts], [Average Days Out], [Overdue %]) but not [Total Copies].

Turnover Rate = Total Checkouts ÷ Total Copies validates when spot‑checked.

No lingering (blank) in Genre after key fix (or filtered out for presentation).

## 6) Summary of Day 2 and why it matters
**What we did today**
- Turned the three cleaned tables from Day 1 into a working model by creating relationships:
- Books_tbl[ISBN] (1) → Checkouts_tbl[ISBN] (*)
- Members_tbl[MemberID] (1) → Checkouts_tbl[MemberID] (*)
- Built core DAX measures (formulas) for circulation and service KPIs.
- Assembled a PivotTable + slicers/timeline to explore the data.
- Fixed a join issue by aligning keys in Checkouts.csv so every checkout references a real book and member.
  
**How it carries over from Day 1**
- Day 1 delivered clean tables and a key calculated column DaysOut.
- Day 2 used those clean columns to (1) relate the tables, (2) aggregate the data into business metrics, and (3) visualize it interactively.
- Together they form a classic star schema: Books and Members (lookup tables) filter the Checkouts fact table.


## 7) Why we had to…
### a) Connect ISBN and MemberID
- Relationships let filters flow from Books (e.g., Genre, Branch) and Members (e.g., MemberType) into Checkouts.
- Without matching keys, Genre/MemberType can’t filter transactions; you get blank or misleading results.
- Correct joins turn row-level records into meaningful aggregated KPIs per Genre, Branch, MemberType, or time.

### b) Create formulas (measures)
- Raw columns answer “what happened per row?”; measures answer “how much/what rate over any slice?”
- DAX measures recalculate on the fly for any selection (branch, month, genre), enabling reusable KPIs across all pivots/charts.

### c) Use a PivotTable (with slicers/timeline)
- A Pivot reads directly from the Data Model, respects relationships, and applies measures correctly.
- Slicers and timelines make the model explorable—stakeholders can self-serve answers (“How did Sci‑Fi perform in West in March?”).

## 8) Formulas we created and why each matters
- Total Checkouts
    - DISTINCTCOUNT(Checkouts[TxnID])
    - What it tells you: overall demand/activity. The base denominator for several rates.

- Active Members
    - DISTINCTCOUNT(Checkouts[MemberID])
    - Why: breadth of engagement—how many unique patrons are participating in the period/segment.

- Active Titles
    - DISTINCTCOUNT(Checkouts[ISBN])
    - Why: collection utilization—how widely the collection is being touched (vs. concentrated on a few titles).

- Total Copies (from Books)
    - SUM(Books[CopiesOwned])
    - Why: inventory baseline. Needed to normalize activity by available copies.

- Average Days Out
    - AVERAGEX(FILTER(Checkouts, NOT ISBLANK([DaysOut])), [DaysOut])
    - Why: service/turnover tempo—how long items are in circulation. High values may signal long loan periods or bottlenecks.

- Overdue Count
    - COUNTROWS( late returns OR currently past due )
    - Why: service compliance and risk (patron experience, fines policy, availability). Feeds the rate below.

- Overdue %
    - DIVIDE([Overdue Count], [Total Checkouts])
    - Why: normalizes lateness by volume so you can compare across branches/genres/time.

- Turnover Rate (Books context)
    - DIVIDE([Total Checkouts], [Total Copies])
    - Why: headline efficiency metric—how often inventory circulates. Guides purchasing, weeding, and branch allocation.

Note: all of these measures are filter-aware—they instantly re‑compute by Genre, Branch (from Books), MemberType, and date (via timeline). That’s the power of the Day 1 cleaning + Day 2 relationships.

## 9) Extra Info on the Formulas used
The formulas used are called DAX measures

### What is a DAX measure?
DAX = Data Analysis Expressions. A measure is a DAX formula stored in the data model (Power Pivot/Power BI) that returns a single value for the current filters (slicers, rows, columns, timeline, etc.).

Think of a measure as a reusable KPI that’s recalculated on the fly for whatever context you put it in.

### Key traits
- Lives in the model, not in a worksheet cell.
- Evaluates per filter context (e.g., Genre = “Sci‑Fi”, Branch = “West”, March 2024).
- Aggregates rows (counts, sums, averages, ratios, time intel).
- Reusable across PivotTables, charts, and visuals.
- Where you create them
    - Power Pivot → Data View → Calculation Area, or
    - Power Pivot → New Measure… dialog.

## 10) Why today is important to the overall project
We moved from clean data to decision-ready insights.

The model now supports a dashboard where leaders can monitor demand, efficiency (turnover), timeliness (overdues, days out), and segment performance—all from one workbook.

The structure is extensible: tomorrow we can add KPI cards, charts, a Branch or Date dimension, and start answering deeper questions (e.g., “Top 10 titles to purchase more of by branch,” “Which member segments have higher overdue rates?”).

Next up: polish the Dashboard sheet (KPI cards + charts), add a Date/Branch dimension for richer time/branch analysis, and document insights.



## 11) Next Steps (Day 3 Preview)
Build a Dashboard sheet layout:

KPI cards: Total Checkouts, Total Copies, Turnover Rate, Overdue %, Avg Days Out.

Top 10 Titles by Turnover; Overdue by Genre/Branch.

Conditional Formatting on a titles table (e.g., highlight high DaysSinceLastCheckout).

Optional: add a Branch dimension table, and a Date table for richer time intelligence.