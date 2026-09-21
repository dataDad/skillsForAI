# Chapter 11: Operations

## Core Idea
This chapter introduces the book's most advanced technique — **emulating a while-loop in a language with no loop construct** — by encoding loop "iterations" as table rows with a running total and an exit-condition column, then finding the row where the condition first flips. Simpler operational KPIs (OTIF, OCT, MTBF/MTTR, OEE, Days of Supply) are variations of prior patterns applied to supply-chain/manufacturing data.

## Frameworks Introduced
- **DAX "while loop" emulation**: since DAX has no loop construct, represent the loop's iterations as **rows of a table** (often generated from a sorted CONCATENATEX path, or GENERATESERIES), add a **running total column** (`SUMX(FILTER(table, [Index] <= EARLIER([Index])), value)`), then add a **decrement/exit column** (`runningTotal - target`), and find the **first row where the exit column crosses the threshold** (`MINX(FILTER(table, [exit] >= 0), [Index])`) — this row is where the "loop" would have exited.
  - When to use: any problem needing sequential accumulation-until-threshold (FIFO/LIFO fulfillment, bin-picking allocation) that would be a `while` loop in a procedural language.
- **EARLIER() for cleaner running-total/self-referencing columns**: inside a nested `ADDCOLUMNS`, `EARLIER([Col])` refers to the outer row's value of `[Col]` — a shorthand alternative to nested VAR/RETURN blocks when comparing a row against "all other rows relative to it" (used for running totals and next/previous-row lookups throughout this chapter).
- **Next/previous-row-value lookup (generalizing Ch3's double lookup)**: `MINX(FILTER(table, [key]=EARLIER([key]) && [orderCol] > EARLIER([orderCol])), [orderCol])` finds the next row for the same entity; pair with `DATEDIFF` to get gap/uptime between events (MTBF variant 2).
- **Priority-ordering via composite sort keys**: DAX has no reliable native row-ordering outside functions like `CONCATENATEX`'s own sort parameters — to break ties or enforce a specific fulfillment priority (most-orders-first, most-qty-first), concatenate the sort-relevant columns into one text/numeric key (`(Quantity & RIGHT(SalesOrder#,8)) * 1.`) and filter/compare on that composite key instead of the raw columns.

## Key Concepts
- **OTIF (On Time In Full)**: an order counts only if *every* line item shipped the full ordered quantity *and* arrived by the due date — computed by flagging each line 1/0, `GROUPBY`-ing to the order level (comparing line count to summed flags), then the percentage of orders where all lines passed.
- **OCT (Order Cycle Time) / OLT (Order Lead Time)**: `AVERAGEX` of `(latest shipped date − earliest received date) + 1` per order, grouped via `GROUPBY`+`MINX`/`MAXX`; OLT substitutes delivery date for shipped date — same formula shape, different date column.
- **FIFO/LIFO/priority-based fulfillment date**: the "while loop" technique applied to find the purchase-order ETA where cumulative incoming quantity first covers cumulative outstanding demand up to and including the current backorder.
- **MTBF (Mean Time Between Failure)**: two methods — (1) simple: `TotalOperatingHours / FailureCount` per machine, averaged; (2) actual-uptime: average of `(nextFailureStart − thisRepairCompleted)` gaps per machine via next-row lookup — method 2 doesn't need to assume a fixed operating-hours window.
- **MTTR (Mean Time To Repair)**: `AVERAGEX` of `DATEDIFF(RepairStarted, RepairCompleted, HOUR)`, excluding preventative maintenance ("PM") rows.
- **OEE (Overall Equipment Effectiveness)** = `Availability × Performance × Quality`, each a separate 0–1 ratio: Availability = `(TotalOperatingHours − TotalDowntimeHours)/TotalOperatingHours`; Performance = `ActualUnitsProduced / Capacity`; Quality = `GoodUnits / ActualUnits`.
- **DOS (Days of Supply)**: `EndingInventory / AverageDailyDemand`, where average daily demand is computed from cumulative average weekly demand up to the current week, divided by 7.
- **SELECTCOLUMNS as a performance discipline**: for measures against large operational fact tables, trimming a virtual table down to only the columns a calculation needs (via SELECTCOLUMNS before further ADDCOLUMNS/FILTER work) reduces memory pressure — a habit the book introduces here specifically because operational fact tables tend to be large.

## Mental Models
- Recognize a "while loop" problem by its shape: **accumulate a running quantity across a sorted sequence until it first meets/exceeds a target, then report where that happened** — this reframes FIFO/LIFO fulfillment, bin-picking, and similar sequential-allocation problems as a single reusable DAX idiom (running total column + threshold-crossing MINX/FILTER), rather than each needing bespoke logic.
- **DAX cannot enforce strict, stable row ordering outside specific sort-aware functions** (CONCATENATEX's order parameters, RANK's ORDERBY) — whenever "first," "next," or "priority" matters and ties are possible (same-date backorders), build an explicit composite tie-breaking key rather than relying on an assumed row order.
- MTBF's two calculation methods illustrate a recurring choice in this book: a **simple method with stated simplifying assumptions** (fixed operating window) vs. a **more accurate method built from actual inter-event gaps** (next-row lookup) — pick based on whether the assumption's cost (accuracy) is acceptable for the reporting need.

## Anti-patterns
- **Assuming DAX preserves any particular row order for "first"/"next" logic without an explicit key**: same-date backorders can be silently merged/mis-prioritized (the book explicitly flags this limitation in the FIFO/LIFO example) — always break ties with a composite sort key when exact ordering matters.
- **Treating MTBF's "simple" fixed-operating-window method as universally accurate**: it assumes every machine operated continuously across the same date range, which is rarely true in practice — prefer the actual-uptime (next-row-gap) method when machines have different install/operating histories, unless the simplifying assumption is explicitly acceptable.
- **Skipping SELECTCOLUMNS on large operational fact tables**: unlike small demo tables elsewhere in the book, large real-world operational fact tables make this optimization worth the extra verbosity — the book explicitly changes its own conventions here for this reason.

## Reference Tables

| KPI | Formula shape |
|---|---|
| OTIF | order passes iff every line: qty shipped = qty ordered AND delivered ≤ due date |
| OCT / OLT | `AVERAGEX(perOrder, (maxShipped/Delivered − minReceived) + 1)` |
| FIFO/LIFO Qty-to-Fulfill | `SUMX(FILTER(ALL(backorders), item=X && date <=/>= currentDate), qty)` |
| Fulfillment Date (while-loop) | running-total-of-incoming-supply crosses cumulative-demand threshold → date of crossing row |
| Simple MTBF | `AVERAGEX(perMachine, TotalOperatingHours / FailureCount)` |
| Actual-uptime MTBF | `AVERAGEX(failures, DATEDIFF(thisRepairCompleted, nextFailureStart, HOUR))` |
| MTTR | `AVERAGEX(nonPM repairs, DATEDIFF(RepairStarted, RepairCompleted, HOUR))` |
| OEE | `Availability × Performance × Quality` |
| Availability | `(TotalOperatingHours − TotalDowntime) / TotalOperatingHours` |
| Performance | `SUM(Actual) / SUM(Capacity)` |
| Quality | `SUM(Good) / SUM(Actual)` |
| DOS | `EndingInventory / (AverageWeeklyDemandToDate / 7)` |

## Worked Example
The "while loop" pattern in its clearest form — FIFO Delivery Date (find the purchase-order ETA at which cumulative incoming supply first covers demand):
```
FIFO Delivery Date =
    VAR __QtyToFulfill = [FIFO Qty to Fulfill]          -- cumulative demand target
    VAR __Table =
        ADDCOLUMNS(
            ADDCOLUMNS(
                SELECTCOLUMNS('Purchase Orders', "ETA", [ETA], "Quantity", [Quantity]),
                "__TotalQty", SUMX(FILTER('Purchase Orders', [ETA] <= EARLIER([ETA])), [Quantity])
            ),
            "__LoopCounter", [__TotalQty] - __QtyToFulfill
        )
    VAR __TargetDate = MINX(FILTER(__Table, [__LoopCounter] >= 0), [ETA])
    VAR __Result = IF(__QtyToFulfill = BLANK() || __TargetDate = BLANK(), BLANK(), __TargetDate)
RETURN
    __Result
```
`__TotalQty` is a running total (via EARLIER) of incoming purchase-order quantity ordered by ETA. `__LoopCounter` goes from negative to non-negative exactly at the ETA where cumulative supply first meets the demand target — `MINX(FILTER(..., >=0), [ETA])` finds that crossing point directly, without ever "iterating." Debug by returning `TOCSV(__Table)` instead of `__TargetDate` (the Ch2 technique) to see the running total and loop-counter columns explicitly.

## Key Takeaways
1. Emulate while-loops by building a running-total column (via EARLIER) plus a threshold/decrement column, then locate the first row where the threshold is crossed with MINX/FILTER.
2. EARLIER is the concise alternative to nested VAR/RETURN for self-referencing running totals and next/previous-row lookups.
3. When "first"/"priority" ordering matters and ties are possible, build an explicit composite sort key — DAX doesn't guarantee row order otherwise.
4. Trim virtual tables with SELECTCOLUMNS before further processing when working against large operational fact tables.
5. Prefer actual-uptime-based MTBF (next-row gap) over fixed-operating-window MTBF when machines don't share a uniform operating history.
6. OEE is a clean multiplicative composite of three independently-computable ratios (Availability × Performance × Quality) — decompose complex composite KPIs into their component ratios first.

## Connects To
- **Ch 4 (Text)**: the Order Fulfillment measure's sorted-locations-as-path technique (CONCATENATEX → PATHITEM) reuses text-to-table directly.
- **Ch 3 / Ch 9**: running totals and next/previous-row lookups extend the double-lookup and cumulative patterns established there.
- **Ch 12 (Distance and Space)**: continues with similarly complex, technique-dense operational-style problems (transitive closure, box packing) that also push past the basic No CALCULATE pattern.
- **Ch 14 (Complex Patterns)**: the DAX INDEX and Streaks patterns later in the book reuse this chapter's running-total/EARLIER techniques.
