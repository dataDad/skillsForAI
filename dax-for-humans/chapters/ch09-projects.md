# Chapter 9: Projects

## Core Idea
Earned Value Management (PV/EV/AC/SV/CV) and related project metrics (burndown, overlap, overallocation) all reduce to two techniques: **prorating a total by elapsed-time-fraction** (idealized burndown, cumulative PV/EV/AC), and **materializing a fine-grained time series (minute or day) to correctly resolve overlaps** (meeting overlap, resource overallocation) instead of naively summing durations.

## Frameworks Introduced
- **Idealized-vs-Actual Burndown**: `Idealized = TotalWork − (TotalWork/TotalDays) × ElapsedDays` (linear expectation) vs. `Actual = TotalWork − CumulativeHoursLogged` (real consumption filtered by date) — plot both to see ahead/behind-schedule visually.
- **PV/EV/AC via per-row cost, then SUMX (the CAC/ACV pattern again)**: `PV = SUMX(Assignments, Work × RELATED(HourlyCost))`; `AC = SUMX(Hours, HoursLogged × RELATED(HourlyCost))`; `EV = SUMX(Project, %Complete × PV_per_task)` — always multiply per-row before aggregating, never multiply pre-summed totals.
- **Cumulative-value-to-date via task-level time-proration**: for each task, compute `__TaskDays` (its total duration) and `__Days` (days elapsed since its start, capped at "today"), then prorate: `if fully elapsed, use full task value; else value × (__Days/__TaskDays)`. Applied identically to Cumulative PV, Cumulative EV, and Cumulative AC (AC uses actual logged hours through the date instead of proration).
- **Minute-resolution overlap resolution**: to correctly compute total *unique* time covered by possibly-overlapping intervals (meetings), don't sum durations (double-counts overlaps) — instead `GENERATESERIES` every minute across the full span, `GENERATE` against all intervals, mark each minute as covered (1) or not (0) per interval, `GROUPBY` per minute taking `MAXX` of the coverage flags (collapses multiple 1s from overlapping meetings into one), then sum the collapsed minutes.
- **Overallocation detection**: analogous to the minute-resolution overlap technique but at day granularity — build a (employee × weekday) grid, and for each cell sum `8 hours × count of active tasks assigned to that employee on that day`, so days with more simultaneously-active tasks show hours exceeding an 8-hour (or 40-hour weekly) capacity threshold.

## Key Concepts
- **PV (Planned Value / BCWS)**: baseline budget — `Work × HourlyCost`, summed.
- **EV (Earned Value / BCWP)**: budget "earned" based on completion — `%Complete × PV`; alternate methodologies use fixed milestone credit instead (e.g. 20% credit for any in-progress task regardless of actual % complete) — which method to use is a project-methodology choice, not a DAX correctness question.
- **AC (Actual Cost)**: `HoursLogged × HourlyCost`, summed, filtered to date ≤ reporting date.
- **SV (Schedule Variance)** = `EV − PV`: positive = ahead of schedule, negative = behind.
- **CV (Cost Variance)** = `EV − AC`: positive = under budget, negative = over budget.
- **SV in days**: found by locating the date on the *planned* curve whose PV equals the *current* EV value (minimum absolute difference lookup) and comparing to today's date — converts a dollar variance into a schedule-days variance.
- **RELATED() / RELATEDTABLE()**: pull a column value from the "one" side of a relationship (RELATED) or a table of matching rows from the "many" side (RELATEDTABLE) — used throughout to join Hourly Cost onto Assignments/Hours rows.
- **GROUPBY + CURRENTGROUP() + MAXX**: the mechanism for collapsing multiple overlapping per-minute "covered" flags into a single 0/1 per minute (any meeting covering this minute ⇒ minute counts once).
- **SELECTCOLUMNS to avoid ambiguous self-referencing column names**: when a virtual table's column shares a name with a column in a table being compared against it, DAX can resolve the reference to the wrong table — rename via SELECTCOLUMNS before comparing (`'Tasks'[Employee Id] = [__EmployeeId]`, not `= [Employee Id]`, which would ambiguously resolve back to Tasks itself).

## Mental Models
- Every EVM "current" or "cumulative" measure is really: **build a per-date/per-task table of prorated values, then look up or sum the relevant slice** — Current SV$/CV use the same "filter to non-blank, take the max date, look up that row's value" idiom as Ch3's period-to-date measures.
- **Overlapping intervals cannot be summed directly** — anytime you need "total unique time/capacity covered by potentially-overlapping ranges," drop to the finest necessary time grain (minute, day) and resolve coverage per unit before aggregating. This is the same "invent the missing rows" idea as Ch7's Open Tickets, generalized to handle overlap collapsing (via MAXX/GROUPBY) rather than just counting.
- Renaming columns via SELECTCOLUMNS before a same-named-column comparison isn't cosmetic — it changes which table DAX resolves the comparison against.

## Anti-patterns
- **Summing individual meeting/task durations to get "total time spent"**: double (or more) counts overlapping periods — the book's example shows 7 hours of nominal meeting duration collapsing to 3.5 actual hours once overlap is resolved.
- **Comparing an unrenamed virtual-table column against a same-named source-table column**: silently resolves to the wrong table's value — always disambiguate column names (SELECTCOLUMNS or a distinct alias) before such comparisons.
- **Treating "% complete" as the only valid earned-value method**: some methodologies fix in-progress credit (e.g. flat 20%) regardless of reported completion — know which your organization uses before hardcoding the proportional-% formula.

## Reference Tables

| Metric | Formula |
|---|---|
| PV | `SUMX(Assignments, Work × RELATED(HourlyCost))` |
| EV (proportional) | `SUMX(Project, %Complete × PV_per_task)` |
| EV (milestone-credit alt) | `SWITCH(%=1, PV; %>0, PV×0.2; 0)` |
| AC | `SUMX(Hours ≤ reportingDate, HoursLogged × RELATED(HourlyCost))` |
| SV$ | `Cumulative EV − Cumulative PV` |
| CV | `Cumulative EV − Cumulative AC` |
| SV (days) | date-lookup: find PV-curve date matching current EV, subtract from today |
| Idealized Burndown | `TotalWork − (TotalWork/TotalProjectDays) × ElapsedDays` |
| Actual Burndown | `TotalWork − SUM(HoursLogged ≤ date)` |
| Meeting Overlap (true unique hours) | per-minute GENERATE + GROUPBY/MAXX coverage flag, SUM/60 |
| Overallocation | per-employee-per-day grid, SUMX(active-task-count × 8hrs) vs. capacity threshold |

## Worked Example
Resolving overlapping meetings into true unique hours covered — the chapter's clearest illustration of "don't sum overlapping intervals directly":
```
Overlap =
    VAR __Start = MIN('Meetings'[Start])
    VAR __End = MAX('Meetings'[End])
    VAR __Table =
        GROUPBY(
            ADDCOLUMNS(
                GENERATE( GENERATESERIES(__Start, __End, 1/24/60), ALL('Meetings') ),
                "__Include", IF([Value] >= [Start] && [Value] <= [End], 1, 0)
            ),
            [Value],
            "__Minute", MAXX(CURRENTGROUP(), [__Include])
        )
    VAR __Result = SUMX(__Table, [__Minute]) / 60
RETURN
    __Result
```
`GENERATESERIES(__Start, __End, 1/24/60)` produces one row per minute (the time-fraction-of-day trick from Ch6). `GENERATE` cross-joins every minute against every meeting, flagging whether that minute falls inside each meeting. `GROUPBY`+`MAXX(CURRENTGROUP(), ...)` collapses each minute's (possibly several) flags down to one — "was this minute covered by *any* meeting?" Summing those collapsed flags and dividing by 60 gives true unique hours (3.5, not the naive 7-hour sum of individual meeting durations).

## Key Takeaways
1. PV/EV/AC/SV/CV all follow "multiply per row, then SUMX" — never aggregate-then-multiply.
2. Cumulative-to-date project metrics prorate each task's value by elapsed-fraction-of-its-own-duration, not the whole project's duration.
3. "Current" value lookups (Current SV$, Current CV) reuse the Ch3 idiom: filter to non-blank dates, take MAX, look up that row.
4. Overlapping time intervals must be resolved at a fine time grain (GENERATE + GROUPBY/MAXX) before summing — naive duration-summing overcounts overlaps.
5. Overallocation is the same fine-grain-resolution technique applied at day granularity to detect when simultaneous task assignments exceed daily/weekly capacity.
6. Rename virtual-table columns before comparing them against same-named source-table columns to avoid DAX resolving the reference to the wrong table.

## Connects To
- **Ch 1–3**: still the No CALCULATE virtual-table-then-X-aggregate pattern; cumulative-to-date proration echoes Ch3's period-to-date offset thinking.
- **Ch 6 (Time and Duration)**: the `1/24/60` minute-fraction constant and GENERATESERIES-driven time expansion reuse Ch6's time-as-decimal-fraction foundation directly.
- **Ch 7 (Open Tickets)**: the GENERATE-based "invent missing intermediate rows" technique is the direct ancestor of both the minute-resolution overlap and day-resolution overallocation techniques here.
- **Ch 11 (Operations)**: schedule/duration-style KPIs (delivery dates, cycle time) reuse the elapsed-time proration pattern from EVM.
