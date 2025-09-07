# Day 6 — Membership & Cohort Analysis
**Date:** 2025-09-06  
**Goal:** Connect member join dates to the Calendar table, add membership-focused measures, and build cohort-style pivots to see how new members and activity trend over time.

---

## Project Goal (restated)
Analyze public-library circulation and inventory with Excel (Power Query, Power Pivot/Data Model, DAX), producing a clean dashboard for titles, usage, timeliness, and membership growth.

---

## What’s been done so far
- **Day 1–2:** Imported/cleaned CSVs with Power Query, loaded them to the **Data Model**, created the *DaysOut* helper, verified keys, and built first pivots & slicers.  
- **Day 3:** Built a **Calendar_tbl** from checkout dates; linked it to **Checkouts_tbl**; added a timeline; introduced a **Books_dim (Branch)** slicer.  
- **Day 4:** Added core DAX measures (Total Checkouts, Total Copies, Overdue %, Average Days Out, Turnover) and formatted them properly.  
- **Day 5:** Created comparative measures (On-Time Return %, Median Days Out), and a KPI comparing **Total Checkouts** vs **Last Month Checkouts**; polished the dashboard visuals.

---

## Today’s highlights (quick bullets)
- Linked **Members_tbl[JoinDate] → Calendar_tbl[Date]** (one-to-many, single direction from Calendar).  
- Confirmed Calendar remains the single **Date** table (marked as Date Table).  
- Added membership measures:
  - **Members Joined** (new members in the selected period)
  - **Cumulative Members** (running total over time)
  - **Active Members** (members who checked out at least once in filter context)  
- Built cohort-style pivots (Join Month by MemberType; trend lines by Month/Year) and wired them to the existing **timeline** & slicers.  
- Documented how to interpret **solid vs. dotted** relationships and when to use `USERELATIONSHIP()`.

---

## Step-by-step (what I did today)

### 1) Validate the Calendar table
- Open **Power Pivot → Diagram View**.
- Ensure **Calendar_tbl** is still marked as Date (Power Pivot → *Mark as Date Table…* → **Date**).
- Confirm **one active link** already exists: **Calendar_tbl[Date] → Checkouts_tbl[OutDate]** (one→many, single direction).

### 2) Relate members to the Calendar
- In **Diagram View**, drag **Members_tbl[JoinDate]** to **Calendar_tbl[Date]**.
- In the **Edit Relationship** dialog:
  - Relationship type: **1 (Calendar) to * (Members)**.
  - **Single** filter direction from Calendar to Members.
  - Leave it **Active** (it won’t conflict because it connects *different* tables).
- Result: the **timeline** and Calendar fields now filter both **Checkouts** *and* **Members**.

> 🔎 **What the dotted line means**  
> Dotted lines in Power Pivot indicate an **inactive** relationship. You can keep alternate date links (e.g., DueDate or ReturnDate) as **inactive** and activate them **only inside a measure** using `USERELATIONSHIP()` when you need a different date lens.

### 3) Add membership measures (Power Pivot → Calculation Area)

> **Home table suggestion:** place these in **Members_tbl** (they work from any table, but keeping “member” measures there is tidy).

- **Members Joined**
  ```DAX
  Members Joined :=
  COUNTROWS ( Members_tbl )
  ```
  *Counts members in the current filter context (e.g., Month = March 2024 → joiners in March).*

- **Cumulative Members**
  ```DAX
  Cumulative Members :=
  VAR MaxDate =
      MAX ( Calendar_tbl[Date] )
  RETURN
      CALCULATE (
          [Members Joined],
          FILTER ( ALL ( Calendar_tbl[Date] ), Calendar_tbl[Date] <= MaxDate )
      )
  ```
  *Running total that shows growth over time.*

- **Active Members** *(checked out at least once in the filter context)*  
  (Requires the existing **Checkouts_tbl** relationship to Calendar.)
  ```DAX
  Active Members :=
  DISTINCTCOUNT ( Checkouts_tbl[MemberID] )
  ```
  *With Month/Year on rows, this shows how many distinct members borrowed items that period.*

*Optional:* If you created an **inactive** relationship from **Checkouts_tbl[ReturnDate] → Calendar_tbl[Date]**, you can define:
  ```DAX
  Checkouts by Return Date :=
  CALCULATE ( [Total Checkouts], USERELATIONSHIP ( Checkouts_tbl[ReturnDate], Calendar_tbl[Date] ) )
  ```
  *Lets you switch the date lens from OutDate to ReturnDate inside a measure.*

### 4) Build cohort/trend pivots
- **Pivot A – Joiners by Month & MemberType**
  - Rows: `Calendar_tbl[Year]`, `Calendar_tbl[MonthName]` (sort with `MonthIndex` if needed).
  - Columns: `Members_tbl[MemberType]`.
  - Values: **Members Joined**.
  - Connect the **timeline** (Calendar[Date]) and the **MemberType** slicer.

- **Pivot B – Cumulative Members**
  - Rows: `Calendar_tbl[Year]`, `Calendar_tbl[MonthName]`.
  - Values: **Cumulative Members**.
  - Insert a **PivotChart (Line)** to visualize growth.

- **Pivot C – Engagement**
  - Rows: `Calendar_tbl[MonthName]`.
  - Values: **Active Members**, **Total Checkouts**.
  - Use slicers (Genre, Branch_dim, MemberType) to compare segments.

### 5) Formatting & usability
- Set measure formats: **Whole Number** (Members Joined, Active Members, Cumulative Members).
- Sort months properly: use `Calendar_tbl[MonthIndex]` as **Sort by Column** for `MonthName`.
- Add explanatory labels (e.g., “New Members per Month”, “Total Members (Running)”).

---

## Why today matters
- **Calendar drives time-intelligence.** Linking **Members_tbl** to the same Calendar builds one coherent time axis for **both** circulation and membership.  
- **Cohort views** (joiners over time, cumulative growth) answer questions like:
  - Are we **acquiring** new members? At what pace?
  - Which **member types** are growing?
  - How does engagement (**Active Members**) track with checkouts?
- These insights complement the book & circulation KPIs, rounding out the dashboard to include **patron growth and participation**.

---

## Additional info (terms explained)

- **Active vs. Inactive relationship:**  
  Only one relationship between a table pair can be active for a given column pair. Inactive ones appear **dotted** and are ignored **unless** a DAX measure activates them with `USERELATIONSHIP()`.

- **Filter direction (single):**  
  Calendar filters Members and Checkouts (not vice versa). This keeps the model stable and prevents ambiguous filter paths.

- **Running total (Cumulative):**  
  Uses **ALL(Calendar)** to remove the row context month filter, then re-applies everything up to the current date.

---

### Next up (Day 7 idea)
- Retention/returning-member metrics (e.g., members who checked out in consecutive months).  
- Cross-overs: checkouts per **new vs. existing** members.  
- Final dashboard polish and README with screenshots.
