# Chapter 7: Customers

## Core Idea
This chapter opens the book's "Scenarios" segment: nearly every customer KPI here (new/lost/returning customers, churn, LTV, CAC, open tickets, funnel drop-off, market-basket "better together", ACV, sales-after-event) is the *same* No CALCULATE pattern from Ch1 — build a virtual table via SUMMARIZE/FILTER/GENERATE/EXCEPT/INTERSECT, then aggregate over it with an X-function — applied to a new business question each time.

## Frameworks Introduced
- **Set-comparison customer segmentation (New/Lost/Returning)**: get the distinct customer-ID set for the current period (`__Current`) and a reference prior period (`__Previous`), then use table set operators to classify:
  - New = `EXCEPT(__Current, __Previous)` (in current, never seen before)
  - Lost = `EXCEPT(__Previous, __Current)` with `__Previous` scoped to *just* last month (never seen this period)
  - Returning = `EXCEPT(INTERSECT(__Current, __PriorHistory), __PreviousMonth)` (seen historically and now, but NOT last month)
  - Growth Rate = `DIVIDE(COUNTROWS(EXCEPT(current, priorMonth)), COUNTROWS(priorMonth))`
  - Churn Rate = `DIVIDE(priorMonthCount - COUNTROWS(INTERSECT(current, priorMonth)), priorMonthCount)`
- **Per-row-then-aggregate (recurring guard against Ch2/Ch5's Measure Totals Problem)**: whenever a KPI multiplies/combines two columns (cost×clicks for CAC, price-like TCV/Years for ACV), compute the combined value **per row first** via ADDCOLUMNS, then SUMX/AVERAGEX the table — never aggregate the source columns separately and combine the aggregates.
- **Inventing missing intermediate rows (Open Tickets / date-span expansion)**: when a fact table only records start/end events (ticket opened/closed) but you need a value *per day in between*, use `GENERATE(factTable, FILTER(datesTable, [Date] between the row's start/end))` to materialize one row per (event, date) pair, then COUNTROWS/aggregate over that expanded table.
- **Pairwise co-occurrence ("Better Together")**: to find which items are purchased together, build the Cartesian product of the distinct item list with itself via `GENERATE`, filter to `__Item1 < __Item2` (removes self-pairs and duplicate reversed pairs), then for each pair count orders (via SUMMARIZE on Order_ID) containing *either* item and filter to orders containing *both* (count > 1).

## Key Concepts
- **EXCEPT / INTERSECT as classification tools**: the entire New/Lost/Returning/Churn/Growth family reduces to picking the right two sets and the right set operator — memorize this mapping rather than each formula.
- **NPS (Net Promoter Score)**: `(% promoters [score>8] − % detractors [score<7]) × 100`, range -100 to 100 (a trademarked term — Bain & Company / NICE Systems / Fred Reichheld).
- **LTV (Lifetime Value)**: `(AvgPurchaseValue × AvgPurchaseFrequency × AvgCustomerLifespan) / DistinctCustomers`, where `AvgCustomerLifespan = 1 / YearlyChurnRate` and `AvgPurchaseFrequency = TotalPurchases / YearsSpanned`.
- **CAC (Customer Acquisition Cost)**: total marketing/overhead cost divided by new customers acquired — built by summing a per-row cost column (`clicks × costPerClick`), never by multiplying summed columns.
- **Funnel Drop-off Rate vs. Abandonment Rate**: Drop-off compares each step to the *immediately preceding* step (`__PrevStep = __CurrStep - 1`); Abandonment compares every step back to the *first* step (`__PrevStep = 1`) — same formula skeleton, only which "previous" count is hardcoded differs.
- **ACV (Annual Contract Value)**: `TCV / MAX(Years, 1)` per contract (floors sub-year contracts at 1 year since it's an *annual* measure), averaged across customers — the total must be the average of the per-customer values, not `SUM(TCV)/SUM(Years)`.
- **EARLIER()**: used as a shorthand inside nested ADDCOLUMNS to reference an outer row's column value without a nested VAR/RETURN block — used in Sales After Event to compare each row's product against the current outer-table row.

## Mental Models
- Ask **"what are the two sets, and what set operation classifies them?"** before writing any customer-segmentation measure — EXCEPT for "in A but not B", INTERSECT for "in both A and B".
- When a metric needs a value that's "per unit combination, then aggregated" (cost per click × clicks, price × quantity, TCV / years), always build the per-row value as an ADDCOLUMNS'd virtual table column first — this is the Measure Totals Problem showing up in nearly every scenario chapter from here on.
- **A relationship's filter direction matters for which table's context flows into a calculation** — Sales After Event depends on visiting the Visits table to establish `__MostRecentVisit`, which only works because the relationship is single-direction from Visits.

## Anti-patterns
- **Multiplying summed columns instead of summing a per-row product**: `SUM(Clicks) * SUM(CostPerClick)` (600 × 350 = 210,000) is wildly wrong vs. the correct per-row sum (70,000) — always ADDCOLUMNS a per-row product/ratio, then SUMX.
- **Excluding all open (unclosed) tickets from "average time to close"**: skews the metric by ignoring tickets that have been open unusually long; the book explicitly flags this as a judgment call with real trade-offs in both directions, not a settled default.
- **Forgetting the Measure Totals fix on a KPI's Total row**: several measures in this chapter (Sales After Visit, ACV) display blank/wrong totals until wrapped in the SUMMARIZE-then-SUMX pattern from Ch2 — always check the Total row, not just individual rows.

## Reference Tables

| KPI | Formula shape |
|---|---|
| New Customers | `COUNTROWS(EXCEPT(currentIDs, allPriorIDs))` |
| Lost Customers | `COUNTROWS(EXCEPT(lastMonthIDs, currentIDs))` |
| Returning Customers | `COUNTROWS(EXCEPT(INTERSECT(currentIDs, allPriorIDs), lastMonthIDs))` |
| Churn Rate | `DIVIDE(lastMonthCount - COUNTROWS(INTERSECT(current, lastMonth)), lastMonthCount)` |
| Growth Rate | `DIVIDE(COUNTROWS(EXCEPT(current, lastMonth)), lastMonthCount)` |
| NPS | `(pct(score>8) - pct(score<7)) * 100` |
| LTV | `AvgPurchaseValue * AvgPurchaseFrequency * (1/YearlyChurnRate) / DistinctCustomers` |
| CAC | `DIVIDE(SUMX(table, "cost per row"), SUMX(table, newCustomersPerRow))` |
| Drop-off Rate | `DIVIDE(prevStepCount - currStepCount, prevStepCount)`, prevStep = currStep - 1 |
| Abandonment Rate | Same, but prevStep hardcoded to step 1 |
| Better Together | Cartesian self-join, dedupe via `Item1 < Item2`, count orders containing both |
| ACV | `AVERAGE(per-row: TCV / MAX(Years, 1))` |
| Open Tickets (per day) | `GENERATE(tickets, matching date range)` → COUNTROWS per date |
| Avg Time to Close | `AVERAGEX(tickets, EffectiveCloseDate - OpenedDate + 1)` |

## Worked Example
"Open Tickets per day" — inventing intermediate rows a source table doesn't record:
```
Tickets Open =
    VAR __Tickets =
        ADDCOLUMNS(
            'Open Tickets',
            "__EffectiveDate",
            SWITCH( TRUE(),
                [Closed Date] <> BLANK(), [Closed Date],
                TODAY() >= [Opened Date], TODAY(),
                [Opened Date] + 1
            )
        )
    VAR __Table =
        SELECTCOLUMNS(
            GENERATE(
                __Tickets,
                FILTER( 'Dates', [Date] >= [Opened Date] && [Date] <= [__EffectiveDate] )
            ),
            "__ID", [Ticket Num]
        )
    VAR __Result = COUNTROWS( __Table )
RETURN
    __Result
```
`__EffectiveDate` handles the still-open case (blank Closed Date → today or opened+1) so every ticket has a bounded date range. `GENERATE` then produces one row per (ticket, date-in-range) pair — turning a 2-column start/end table into a fully expanded per-day table any COUNTROWS/aggregate can consume. Debug by swapping `RETURN __Result` for `RETURN TOCSV(__Table)` (the Ch2 debugging technique) to see exactly which ticket IDs are open on a given date.

## Key Takeaways
1. Customer segmentation KPIs (new/lost/returning/churn/growth) are all EXCEPT/INTERSECT set operations on two ID sets — identify the sets first, the formula follows.
2. Whenever a KPI combines two columns multiplicatively or divisively, compute per row via ADDCOLUMNS before aggregating — this recurs in CAC, ACV, and beyond.
3. GENERATE lets you materialize implied intermediate rows (open-ticket date spans) that a source table doesn't explicitly store.
4. Pairwise/market-basket analysis is a self-join (GENERATE against itself) filtered to break symmetry (`A < B`).
5. Check Total rows separately from individual rows — several of this chapter's measures need the SUMMARIZE-then-SUMX Measure Totals fix to total correctly.
6. This chapter is proof-by-repetition that the Ch1 "filter into a table, aggregate with X" pattern scales to real, non-trivial business KPIs without CALCULATE.

## Connects To
- **Ch 1 (No CALCULATE Pattern)**: every measure in this chapter is a direct instance of it.
- **Ch 2 (Measure Totals Problem)**: reused explicitly for AC/CAC, ACV, and Sales After Visit totals.
- **Ch 3 (offsets, EOMONTH)**: churn/growth rate prior-period boundaries use EOMONTH exactly as in period-to-date measures.
- **Ch 8 (Human Resources)**: reuses the same set-comparison and per-row-aggregation patterns for turnover, absenteeism, and other HR KPIs.
