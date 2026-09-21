# Glossary — DAX For Humans

**ACV (Annual Contract Value)** — average per-customer annual revenue from a contract; `TCV / MAX(Years, 1)`, averaged across customers, not `SUM(TCV)/SUM(Years)` (Ch 7).

**ALL** — removes all filters (internal and external to the visual) from a table. (Ch 2)

**ALLSELECTED** — removes filters internal to the visual but keeps filters external to it (e.g. slicers). (Ch 2)

**APTR (Accounts Payable Turnover Ratio)** — `SupplierPurchases / AverageAccountsPayable`; liquidity KPI, healthy range ~6–10. (Ch 10)

**ATAN2** — four-quadrant arctangent, absent from DAX; hand-rolled via a quadrant-aware SWITCH. (Ch 12)

**Auto-exist** — Power BI default where two+ active filters on columns from the same table are intersected rather than applied independently. (Ch 2)

**Bradford Factor** — `Instances² × DaysAbsent`; weights frequent short absences more than infrequent long ones. (Ch 8)

**CALCULATE** — takes a scalar expression and a filter clause; a "fancy FILTER function" that changes context in one step. Nested CALCULATE calls have the innermost filter silently override the outer, unless KEEPFILTERS forces an intersection. (Ch 2, Ch 16)

**CALCULATETABLE** — CALCULATE's table-returning sibling. (Ch 13)

**Chelsie Eiden's Duration** — packed-digit duration formatting technique (`Days*1000000 + Hours*10000 + ...` + custom format string `00:00:00:00`) that keeps a duration numeric (aggregatable) while displaying as D:H:M:S. (Ch 6)

**Churn Rate** — `DIVIDE(lastMonthCount − COUNTROWS(INTERSECT(current, lastMonth)), lastMonthCount)`. (Ch 7)

**CONCATENATEX** — iterates a table, concatenating an expression per row with a delimiter and optional sort parameters. (Ch 4)

**Complex Selector** — a measure returning 0/1 (or a value) used as a visual-level filter to impose arbitrary custom selection logic. (Ch 13)

**Context** — the filters currently affecting a DAX calculation; comes from row position, visuals (internal/external), or the DAX expression itself. (Ch 1)

**Context transition** — CALCULATE's mechanism for converting row context into filter context. (Ch 16)

**DAX Index** — a guaranteed, sortable row index built by converting a table to a CONCATENATEX path (optionally sorted) then rebuilding via GENERATESERIES + PATHITEM. (Ch 14)

**DIVIDE** — safe division with an optional fallback for divide-by-zero; preferred over `/` except with literal-constant divisors. (Ch 5)

**Double lookup** — a two-step pattern: look up a key value (e.g. MAX(Date)), then filter again on that key to retrieve the final value. Foundational to previous-period, previous-row, and MODE-style calculations. (Ch 2)

**DOS (Days of Supply)** — `EndingInventory / AverageDailyDemand`. (Ch 11)

**EARLIER** — inside a nested ADDCOLUMNS/FILTER, refers to the outer row's value of a column; a shorthand alternative to nested VAR/RETURN for running totals and next/previous-row lookups. (Ch 11)

**EOMONTH** — returns the end-of-month date N months from a given date; `EOMONTH(d,-7)+1` gives the first of the month six months prior.

**ETR (Employee Turnover Rate)** — `Termed / ((StartHeadcount + EndHeadcount)/2)`. (Ch 8)

**EVALUATEANDLOG** — logs an expression's intermediate result via Analysis Services/SQL Profiler tracing while still returning its value; a "debug print" for DAX. (Ch 16)

**EV / PV / AC (Earned Value / Planned Value / Actual Cost)** — Earned Value Management metrics; EV = %Complete × PV (or milestone-credit alt), PV = Work×HourlyCost, AC = HoursLogged×HourlyCost. (Ch 9)

**FTE (Full-Time Equivalent)** — `TotalHoursWorked / MaximumFullTimeHoursInPeriod`. (Ch 8)

**Filter context** — the set of filters applied to a table/column during evaluation, as opposed to row context.

**FILTERS / ISFILTERED / ISCROSSFILTERED** — functions reporting which columns are currently filtered and by what values, used for diagnosing context issues. (Ch 16)

**Gamma function** — statistical function missing from DAX (extends factorial to non-integers); implemented via the Lanczos numerical approximation since DAX cannot recurse. (Ch 14)

**GENERATE** — Cartesian product between two tables, with the second evaluated in the row context of the first; the mechanism behind "inventing" missing intermediate rows (Open Tickets, Overlap, Overworked). (Ch 7, Ch 9)

**Gini Coefficient** — 0 (perfect equality) to 1 (perfect inequality), computed from the area under a Lorenz curve via a Riemann sum. (Ch 8)

**GROUPBY** — groups a table, requiring an X-aggregator with `CURRENTGROUP()` for aggregated columns.

**HASONEVALUE** — returns True if exactly one distinct value of a column remains in the current context. (Ch 2)

**HCVA (Human Capital Value Added)** — `(Revenue − (TotalCost − FullyLoadedEmploymentCost)) / FTE`, pro-rated per employee's active-days fraction. (Ch 8)

**Haversine formula** — great-circle distance between two lat/long points on a sphere. (Ch 12)

**ISINSCOPE** — returns True when a column is the active level of a hierarchy in a visual; must be tested bottom-of-hierarchy to top inside a SWITCH TRUE to work correctly. (Ch 2, Ch 13)

**Jaccard similarity** — `|intersection| / |union|` of two strings' character sets, a fuzzy-matching technique. (Ch 14)

**Kaplan-Meier estimator** — `Π(1 − d(i)/n(i))` over time increments; survival-curve statistic repurposed for employee tenure. (Ch 8)

**KEEPFILTERS** — modifies CALCULATE so the inner filter intersects with, rather than overrides, the outer filter. (Ch 16)

**Lanczos approximation** — numerical method for computing the gamma function without recursion. (Ch 14)

**LTV (Lifetime Value)** — `AvgPurchaseValue × AvgPurchaseFrequency × (1/YearlyChurnRate) / DistinctCustomers`. (Ch 7)

**Measure Totals Problem ("Banana Pickle Math")** — when a measure's Total row doesn't equal the sum of its rows (common with non-additive expressions); fixed by rebuilding the visual's grouping as a SUMMARIZECOLUMNS table variable and SUMX-ing over it. (Ch 2)

**MEDIAN in calculated columns** — MEDIAN/MEDIANX/PERCENTILE(X) functions error in calculated columns (variant-type error); fix with `CONVERT(..., DOUBLE)`. (Ch 5)

**MOD decimal bug** — MOD misbehaves with decimal divisors; fixed with a custom ROUND-based "floating mod." (Ch 5)

**MODE** — missing from DAX; built via SUMMARIZE + COUNTROWS + filter-to-max (a double lookup). (Ch 5)

**MTBF / MTTR (Mean Time Between/To Failure/Repair)** — reliability KPIs; MTBF via fixed-window or actual-uptime (next-row-gap) methods; MTTR = `AVERAGEX(DATEDIFF(RepairStarted, RepairCompleted, HOUR))`. (Ch 11)

**NETWORKDAYS** — counts working days (Mon–Fri) between two dates; cannot represent a 7-day work week.

**No CALCULATE Pattern** — the book's core reusable formula shape: filter a table into a VAR, aggregate over it with an X-function, RETURN the result — avoids CALCULATE entirely. (Ch 1)

**NPS (Net Promoter Score)** — `(%promoters[score>8] − %detractors[score<7]) × 100`. (Ch 7)

**OCT / OLT (Order Cycle Time / Order Lead Time)** — `AVERAGEX` of date differences per order, grouped via GROUPBY+MINX/MAXX. (Ch 11)

**OEE (Overall Equipment Effectiveness)** — `Availability × Performance × Quality`. (Ch 11)

**Offset** — a signed integer distance from the current period (0=current, -1=previous, etc.) for a given date grain; the foundational technique replacing DAX's native time-intelligence functions. (Ch 3)

**OTIF (On Time In Full)** — an order counts only if every line shipped full quantity and arrived by the due date. (Ch 11)

**PATHITEM / PATHLENGTH** — DAX hierarchy-path functions repurposed for text-to-table conversion (split on a delimiter converted to pipe characters). (Ch 4)

**PRODUCTX** — the multiplicative counterpart to SUMX; used for running products (Kaplan-Meier survival, non-recursive compound interest). (Ch 8, Ch 10)

**RANK / RANKX / RANK.EQ** — DAX's three ranking functions; RANK (newest) supports `PARTITIONBY` for grouped ranking without RANKX's manual filtering. (Ch 5)

**Reasonable maximum search-chaining** — chaining SEARCH calls, each starting after the previous match, to find an unknown number of pattern occurrences. (Ch 4)

**RELATED / RELATEDTABLE** — pull a column value from the "one" side of a relationship, or a table from the "many" side. (Ch 9)

**RETURN-swap debugging** — swapping a measure's RETURN target to an intermediate VAR or `TOCSV(tableVar)` to inspect intermediate calculation steps. (Ch 2, Ch 16)

**Row context** — the implicit "current row" context in which a column reference like `[Price]` resolves. (Ch 1)

**Semi-additive measure** — a measure (like currency conversion) that must be computed per row, not with a single global aggregate, because its correct value depends on which row/date is in context. (Ch 10)

**SUMMARIZE / SUMMARIZECOLUMNS / GROUPBY** — three functions that group rows and add aggregated columns; SUMMARIZECOLUMNS is generally preferred but can't reference sibling computed columns within the same call. (Ch 2)

**SV / CV (Schedule Variance / Cost Variance)** — `EV−PV` and `EV−AC` respectively, in Earned Value Management. (Ch 9)

**TOCSV** — renders a table as delimited text (up to MaxRows, default 10); the primary tool for visually inspecting a table variable's contents while debugging. (Ch 2)

**TREATAS** — applies a table's values as a virtual relationship filter onto another table's column, without a physical relationship.

**Text-to-table** — converting a string into a one-row-per-character/word table via GENERATESERIES + PATHITEM/MID, unlocking full table operations on text. (Ch 4)

**TRIMMEAN** — Excel function missing from DAX; mean after trimming a symmetric percentage of extreme values, rebuilt via ranking + cumulative counts + direct sum/count math (not AVERAGEX). (Ch 14)

**Unique Ranking** — tie-free ranking via a composite row-identifier string turned into a sorted CONCATENATEX path, read back positionally — works even on virtual tables. (Ch 5)

**VAR / RETURN** — the modern way to structure DAX as named, ordered steps instead of nested function calls; the syntactic backbone of the No CALCULATE approach. (Ch 1)

**Weighted Average** — `DIVIDE(SUMX(table, value×weight), SUMX(table, weight))`. (Ch 5)

**While-loop emulation** — encoding a sequential accumulate-until-threshold problem as table rows with a running-total column (via EARLIER) and a decrement/exit column, then finding the first row where the threshold is crossed via MINX/FILTER. (Ch 11)

**X aggregator** — the "X" suffix versions of aggregation functions (SUMX, AVERAGEX, MINX, MAXX, COUNTX) that take a table expression plus a per-row scalar expression; more general than the plain aggregators. (Ch 1)
