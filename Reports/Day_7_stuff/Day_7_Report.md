# Day 7 – Inventory Risk: Stale Titles & Potential Shortages

**Date:** 2025-09-08  
**Goal:** Add a *staleness* parameter and risk measures to highlight books that haven’t circulated recently and titles that may be under‑stocked. Build pivots/slicers so the window (90/120/180/365 days) can be changed without editing formulas.

---

## Project Goal (restated)
Create an Excel Library Insights Dashboard that pulls raw CSVs with **Power Query**, models the data in **Power Pivot** (Data Model), and delivers interactive **PivotTables/PivotCharts** with **DAX measures** and **KPIs**—so librarians can track circulation, inventory health, and service quality.

---

## What’s been done so far
- **Day 1:** Folder structure, imported CSVs with Power Query, cleaned types, loaded to the Data Model.  
- **Day 2:** Related tables (Books, Members, Checkouts), baseline measures (Total Checkouts, Total Copies, Overdue %, Avg Days Out).  
- **Day 3:** Built **Calendar_tbl** from actual transaction date range, marked as Date Table; added timeline & time splits.  
- **Day 4–5:** Segmentation and KPIs; formatting measures for consumption (percent, decimals).  
- **Day 6:** Member/branch perspectives, sanity checks on relationships and filters.

Today we added risk‑focused measures and a slicer‑driven *stale window* to surface problem titles.

---

## Summary of today’s activity
- Created a small **StaleDays** dimension (90/120/180/365) and added it to the Data Model.
- Wrote a parameterized measure **Stale Days Selected** using the slicer’s selected value.
- Built **Recent Checkouts (N)** that counts loans in the last *N* days, honoring timeline/calendar filters.
- Added **Demand per Copy (N)** and **Is Potential Shortage (N/Yes‑No)**.
- (Optional) Added **Is Stale Title?** for titles with *no* recent circulation in the selected window.
- Built two pivots:  
  1) **Stale Titles** (filter = *Is Stale Title? = 1*) and  
  2) **Potential Shortages** with **Top 10 by Demand per Copy**.  
- Connected the **StaleDays slicer** to both pivots and formatted units.

---

## Step‑by‑step

### 1) Create *StaleDays* and add to the Data Model
1. On an empty worksheet, enter a one‑column table:
   - Header: `StaleDays`
   - Rows: `90`, `120`, `180`, `365`
2. **Home → Format as Table** → name it **StaleDays_tbl**.  
3. **Data → From Table/Range** → *Only Create Connection* + **Add to Data Model** (so it’s visible in Power Pivot).

> **Why a dimension table?** A tiny, maintainable list drives a slicer. You avoid hard‑coding N inside measures and give users control.

---

### 2) Parameterize the window
Create a **measure** in the Power Pivot **Calculation Area** (any table is fine; many teams store utility measures in a helper table):

```DAX
Stale Days Selected :=
VAR v = SELECTEDVALUE ( StaleDays_tbl[StaleDays] )
RETURN IF ( ISBLANK ( v ), 90, v )      -- default to 90 if nothing selected
```

---

### 3) Count recent loans (N days)
Use the Calendar to respect report filters/timeline. If there’s no active date context, fall back to **TODAY()**.

```DAX
Recent Checkouts (N) :=
VAR n = [Stale Days Selected]
VAR EndDate =
    IF ( ISBLANK ( MAX ( Calendar_tbl[Date] ) ),
         TODAY(),
         MAX ( Calendar_tbl[Date] ) )
VAR StartDate = EndDate - n
RETURN
    CALCULATE (
        DISTINCTCOUNT ( Checkouts_tbl[TxnID] ),
        DATESBETWEEN ( Calendar_tbl[Date], StartDate, EndDate )
    )
```

> **Note:** Excel Power Pivot doesn’t support `COALESCE` in some versions; the `IF(ISBLANK(...), TODAY(), ...)` pattern is a compatible substitute.

---

### 4) Risk measures
```DAX
Demand per Copy (N) :=
DIVIDE ( [Recent Checkouts (N)], [Total Copies] )

Is Potential Shortage (N) :=
VAR threshold = 0.50           -- tweak as policy requires
RETURN IF ( [Demand per Copy (N)] > threshold, 1, 0 )

Is Potential Shortage :=
IF ( [Is Potential Shortage (N)] = 1, "Yes", "No" )
```

**(Optional) Flag truly stale titles** (no loans in the N‑day window):
```DAX
Is Stale Title? :=
VAR n = [Stale Days Selected]
VAR EndDate =
    IF ( ISBLANK ( MAX ( Calendar_tbl[Date] ) ),
         TODAY(),
         MAX ( Calendar_tbl[Date] ) )
VAR StartDate = EndDate - n
VAR recent =
    CALCULATE (
        DISTINCTCOUNT ( Checkouts_tbl[TxnID] ),
        DATESBETWEEN ( Calendar_tbl[Date], StartDate, EndDate )
    )
RETURN IF ( recent = 0, 1, 0 )
```

Format:
- **Demand per Copy (N):** Decimal (2).  
- **Is Potential Shortage (N):** Whole Number.  
- **Is Potential Shortage / Is Stale Title?:** keep as text *Yes/No* or use numeric for filters.

---

### 5) Build the pivots
**Pivot A – Stale Titles**  
- Rows: `Books_tbl[Title]`  
- Filters: `Is Stale Title? = 1`  
- Values: `Recent Checkouts (N)` (optional add others)

**Pivot B – Potential Shortages**  
- Rows: `Books_tbl[Title]`  
- Values: `Recent Checkouts (N)`, `Total Copies`, `Demand per Copy (N)`, `Is Potential Shortage (N)`  
- Apply **Top 10**: Right‑click a title → *Filter* → *Top 10…* → *Top* **10** by **Demand per Copy (N)**.

**Connect the StaleDays slicer**
- Insert Slicer from any *Data Model‑backed Pivot*: **PivotTable Analyze → Insert Slicer → StaleDays_tbl[StaleDays]**.  
- With the slicer selected: **Slicer → Report Connections** → check both pivots.  
  > If *Report Connections* is greyed out, the slicer was created from a normal worksheet table—delete it and re‑insert from a Pivot that uses the Data Model.

---

## Why today matters
- Turns the dashboard from *descriptive* to **actionable** by revealing:
  - **Stale stock** (titles not moving in N days).  
  - **Shortage risk** (high demand relative to copies).  
- The **StaleDays** parameter makes the view flexible for monthly, quarterly, or annual audits—no code changes required.

---

## Troubleshooting & tips
- **Hashes (####)** in a pivot cell → widen the column or change number format.  
- **Slicer not affecting a pivot** → use *Report Connections* and confirm both pivots are Data‑Model based.  
- **Top 10 not responding** → ensure you set *Top by* the correct **measure** (Demand per Copy), not a field count.  
- **Date filtering** → Calendar_tbl must be *Marked as Date Table* and related to Checkouts_tbl via the date you’re slicing (typically `OutDate`).

---

## Next steps
- Tune the shortage threshold by branch/genre.  
- Add conditional formatting to highlight *Yes* in **Is Potential Shortage**.  
- Create a small “action list” Pivot (Title, Branch, CopiesOwned, Suggested Copies).

