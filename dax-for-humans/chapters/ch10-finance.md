# Chapter 10: Finance

## Core Idea
Beyond DAX's 50+ built-in financial functions, this chapter's real value is in **granularity-mismatch resolution** (comparing monthly budgets to daily actuals, reconstructing period values from cumulative totals, allocating date-ranged amounts across months) and correctly handling **semi-additive measures** (currency conversion, where the "right" rate depends on which row/date is in context, not a single global rate).

## Frameworks Introduced
- **Solve-for-any-variable calculators (What-If parameters)**: create three Power BI "New Parameter → Numeric Range" inputs (Gross Margin, Revenue, Cost), each generating a table + a `SELECTEDVALUE`-based measure, then write three measures that each solve the GM/Revenue/Cost algebraic identity for the "missing" variable given the other two selected slicer values, `MROUND`ed to the parameter's increment.
- **Semi-additive currency conversion**: for each transaction row, look up the *most recent exchange rate on or before that row's date* (`MAXX(FILTER(rates, date <= rowDate & from/to match), date)`, a double-lookup), fall back to rate = 1 via `COALESCE` when no cross-rate exists (same-currency), then `SUMX(rate × amount)` — this must be computed **per row**, not as a single global rate, because it's semi-additive (Ch2 Measure Totals family): a naive single "current rate" measure would misstate historical rows.
- **Granularity reconciliation (budget↔actuals)**: to compare a monthly-grain budget against daily-grain actuals, either (a) roll actuals up to monthly (`FORMAT(date,"mmm-yyyy")` grouping key) or (b) spread the monthly budget down to daily (`budget / DAY(EOMONTH(date,0))`) — pick the direction based on which visual grain you need; cumulative ("running") versions of both use the same day-elapsed-fraction proration idea as Ch9's cumulative PV/EV/AC.
- **Reverse-engineering period values from cumulative data**: when only running/YTD totals are stored (not period deltas), recover the period value via `CurrentCumulative - PreviousCumulative` (a double-lookup on the previous row, `ALL` + FILTER to `month - 1`) — the generic "de-accumulate a running total" technique, reusable anywhere only cumulative data is available.
- **Non-recursive compound interest (DAX has no true recursion)**: instead of iteratively compounding year-over-year (which DAX can't express directly), compute **each year's contribution independently** — for each investment year, multiply its principal by the product (`PRODUCTX`) of `(1+rate)` across every year from investment through the target year — then sum all years' contributions. Converts an inherently recursive problem into an aggregate-of-independent-terms problem.

## Key Concepts
- **Gross Margin identities**: `%GM = (Revenue-Cost)/Revenue`; `Revenue = Cost/(1-%GM)`; `Cost = Revenue×(1-%GM)` — three rearrangements of one equation, each useful depending on which two values are known.
- **COALESCE(v1, v2, ...)**: returns the first non-blank value — used here as a cleaner alternative to `IF(x=BLANK(), fallback, x)`, though the book notes the explicit IF form is arguably more in line with No-CALCULATE's "fewest functions" philosophy.
- **DATEVALUE pitfalls**: converts a text date string to a real date, but format ambiguity matters — `"mmm-yyyy"` ("Jan-2024") parses correctly to the 1st of that month, while `"mmm-yy"` ("Jan-24") is misparsed as day 24 of the *current* year, not year 2024.
- **APTR (Accounts Payable Turnover Ratio)**: `TotalSupplierPurchases / AverageAP` where `AverageAP = (BeginningAP + EndingAP)/2` — a liquidity metric; industry-healthy range cited as 6–10 turns/year.
- **Modified Dietz Return**: `(EndValue - StartValue - NetFlows) / (StartValue + WeightedFlows)`, where each flow is weighted by `(daysRemainingAfterFlow / totalDays)` — approximates IRR without XIRR's computational cost and without needing a valuation at every flow date.
- **XIRR**: DAX's built-in true internal-rate-of-return function, taking a `DATATABLE` of (date, cashflow) pairs — gives a more precise but more computationally intensive answer than Modified Dietz (42.16% vs. 42.52% in the book's example).
- **FV(rate, periods, payment, presentValue)**: DAX's native future-value function (financial-function family); `POWER(1+r, t)` is the manual equivalent for simple compounding without periodic payments.

## Mental Models
- Whenever a rate/ratio measure must be correct **both per-row and in total** (currency conversion is the clearest case), assume it's semi-additive and build it with the Ch2 Measure Totals pattern (per-row ADDCOLUMNS computation, then SUMX) by default — don't wait to discover the total is wrong.
- Granularity mismatches (budget/actual, YTD/monthly) are resolved by **picking a direction to convert and building the lookup/allocation formula for that direction** — either coarse-to-fine (spread a month's budget over its days) or fine-to-coarse (roll daily actuals up to the month) — never by comparing the two at mismatched grains directly.
- **"DAX can't recurse" is a design constraint to route around, not a wall**: decompose a recursive-looking problem (compounding across years) into independent per-entity contributions that can each be computed directly and then summed — this is the same "avoid nested nested lookups" spirit as the No CALCULATE philosophy applied to a genuinely different kind of problem (temporal recursion rather than filter/aggregate composition).

## Anti-patterns
- **Using a single global "current" exchange rate for a currency-conversion measure**: works for the grand total by luck but misstates every historical row — currency conversion is semi-additive and needs per-row rate lookup (Ch2 pattern).
- **Assuming `"mmm-yy"` and `"mmm-yyyy"` behave the same in DATEVALUE**: they don't — `"mmm-yy"` silently produces a wrong date (day-of-month = the 2-digit year number, year = current system year).
- **Comparing budget and actuals at mismatched granularity without reconciling**: a monthly budget number plotted directly against daily actuals is not a meaningful comparison — always roll one up or spread one down first.
- **Trying to hand-simulate iterative compounding year by year in DAX**: DAX has no loop/recursion construct for this; restructure the calculation as an aggregate of independent per-period contributions instead (see Compound Interest's `PRODUCTX`-based non-recursive formula).

## Reference Tables

| KPI / Technique | Formula shape |
|---|---|
| %GM / Revenue / Cost solver | Algebraic rearrangement of `%GM=(Rev-Cost)/Rev`, `MROUND`ed to parameter increment |
| Currency conversion (semi-additive) | Per-row: most-recent-rate-on-or-before lookup × Amount, `COALESCE`(rate,1), then SUMX |
| Periodic billing revenue | `GENERATE` billing rows against matching Year-Month rows in Dates, SUMX(Amount) per bucket |
| Reverse YTD (de-accumulate) | `CurrentCumulative − MAXX(FILTER(ALL, priorPeriod), Cumulative)` |
| Budget↔Actuals (coarse→fine) | `MonthlyBudget / DAY(EOMONTH(date,0))` |
| Budget↔Actuals (fine→coarse) | `SUMX(FILTER(daily, monthKey = targetMonthKey), amount)` |
| APTR | `SupplierPurchases / ((BeginningAP+EndingAP)/2)` |
| Modified Dietz Return | `(B−A−ΣF) / (A + Σ(weight×F))`, weight = days-remaining/totalDays |
| Compound Interest (non-recursive) | `SUMX(perYearPrincipal, principal × PRODUCTX(ratesFromInvestYearToTargetYear, 1+rate))` |

## Worked Example
Non-recursive multi-year compound interest with varying annual investments and rates — the chapter's most instructive "DAX has no recursion, route around it" technique:
```
Future Value Complex =
    VAR __Year = MAX('Calendar'[Year])
    VAR __Table =
        ADDCOLUMNS(
            SUMMARIZE( FILTER(ALL('Calendar'[Year]), [Year] <= __Year), [Year],
                "__Amount", SUM('Investments'[Investment]) ),
            "__TotalAmount",
                VAR __CurrentYear = [Year]
                VAR __CompoundInterest =
                    PRODUCTX(
                        FILTER(ALL('Interest Rates'), YEAR([Date]) >= __CurrentYear && YEAR([Date]) <= __Year),
                        1 + [Rate]
                    )
                RETURN [__Amount] * __CompoundInterest
        )
    VAR __Result = SUMX(__Table, [__TotalAmount])
RETURN
    __Result
```
Each investment year's principal is compounded forward independently by the product of every intervening year's `(1+rate)` factor — so year 2021's $10,000 and year 2023's $7,500 are each grown by their own applicable rate-product and then summed, avoiding any need to "remember" a running balance across iterations.

## Key Takeaways
1. Solve-for-any-variable calculators are just algebraic rearrangements wired to What-If parameter slicers via SELECTEDVALUE.
2. Currency conversion (and any rate lookup that varies per transaction) is semi-additive — always compute per row via ADDCOLUMNS before SUMX, per the Ch2 Measure Totals pattern.
3. Reconciling mismatched granularity (budget vs. actuals, YTD vs. monthly) means explicitly choosing a direction (spread or roll-up) and building that specific lookup/allocation formula.
4. De-accumulating a cumulative/YTD series into period deltas is a previous-row double-lookup, same shape as Ch3's previous-period patterns.
5. DAX's lack of recursion is solved by decomposing recursive-looking problems (compounding) into independent per-entity contributions summed together, not by trying to simulate a loop.
6. Watch DATEVALUE format string subtleties — `"mmm-yy"` silently misparses.

## Connects To
- **Ch 2 (Measure Totals)**: currency conversion is a direct real-world application of the semi-additive-measure fix.
- **Ch 3 (offsets, previous-period lookups)**: Reverse YTD reuses the exact "filter ALL to prior period, MAXX lookup" idiom.
- **Ch 9 (Projects/EVM)**: cumulative budget-vs-actual proration echoes Cumulative PV/EV/AC's elapsed-fraction technique.
- **Ch 11 (Operations)**: continues the KPI-recipe format into operational metrics next.
