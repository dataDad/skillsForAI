# Chapter 2: More Core Concepts

## Core Idea
CALCULATE collapses filtering and aggregation into a single opaque step, which is exactly why it's hard to debug; the No CALCULATE style keeps those steps as separate, inspectable variables — and this chapter builds the rest of the core DAX toolkit (lookups, grouping, columns, logic, information functions) on top of that same VAR/RETURN discipline.

## Frameworks Introduced
- **No CALCULATE vs. CALCULATE (debuggability argument)**: `CALCULATE(<expr>, <filter>)` is "a fancy FILTER function" — it changes context while performing a calculation in one step. Functionally equivalent to filter-then-aggregate, but because it's one step you can't inspect its intermediate state; nesting `CALCULATE` inside `CALCULATE` compounds this.
  - When to use No CALCULATE instead: whenever you want a calculation you can debug by swapping the `RETURN` target.
  - How: write `VAR __Table = FILTER(...)` then aggregate with an X function, exactly as in Ch1.
- **The RETURN-swap debugging technique**: because each step of a No CALCULATE measure is a named variable, you can debug by temporarily changing what `RETURN` outputs.
  - How: swap `RETURN __Result` for `RETURN COUNTROWS(__Table)` to check row counts, or `RETURN TOCSV(__Table)` to see the actual rows of a table variable rendered as text in a Card/Table visual. Put it back to `__Result` when done.
- **The Measure Totals Problem fix ("Banana Pickle Math")**: when a measure's Total row doesn't equal the sum of its displayed rows (common with non-additive expressions like `SUM(...) - 2`).
  - When to use: any measure whose grand total looks wrong in a Table/Matrix visual (rare in other visual types).
  - How: build a `__Table` VAR with `SUMMARIZECOLUMNS` that regroups the data exactly as the visual does, include a column that evaluates the broken measure per group, then `SUMX` over that table and return the result.

## Key Concepts
- **CALCULATE**: takes a scalar expression and a filter clause; changes filter context in one step instead of filter-then-aggregate.
- **TOCSV**: renders a table (up to `MaxRows`, default 10) as delimited text — the primary tool for visually inspecting a table variable's contents in a Card/Table visual.
- **ALL**: removes all filters (internal and external to the visual) from a table, so the calculation sees every row.
- **ALLSELECTED**: removes filters internal to the visual but keeps filters external to it (e.g. from slicers) — subtly different from ALL, and the difference only shows once a slicer/other visual is added.
- **Auto-exist**: a Power BI Desktop default behavior where two or more active filters on columns from the *same* table are intersected rather than applied independently — explains why an `ALL()`-based running total can still appear restricted by a slicer on a related column from the same table.
- **SUMMARIZE / GROUPBY / SUMMARIZECOLUMNS**: three different functions that all group rows and add aggregated columns; SUMMARIZECOLUMNS is generally preferred (faster, purpose-built) but doesn't let new columns reference other computed columns from the same call — use ADDCOLUMNS layered on top for that.
- **HASONEVALUE**: returns True if exactly one distinct value of a column remains in the current context (e.g. false on a Total row where many values collapse together).
- **ISINSCOPE**: returns True when a column is the active level of a row/column hierarchy in a visual; only reliable when tested from the bottom of the hierarchy upward, since a `SWITCH(TRUE(), ...)` stops at the first match and a higher-level column is technically "in scope" at every level below it too.
- **IN operator / CONTAINSROW**: tests whether a value (or row of values) exists in a specified table; `{ "A", "B" }` is DAX's inline table-constructor syntax.
- **UNION / EXCEPT / INTERSECT / CROSSJOIN**: append, subtract, intersect, and Cartesian-join two tables respectively (UNION/EXCEPT/INTERSECT require matching column counts).

## Mental Models
- Use **FILTER + MINX/MAXX** instead of `LOOKUPVALUE` when you need to fetch a specific value — it's one fewer function to learn, and it degrades gracefully (returns the extremum instead of erroring) rather than requiring a guaranteed-single-value.
- Think of **"double lookup"** as a two-step pattern: first look up a key value (e.g. `MAX(Date)`), then filter again on that key to retrieve the final value. This recurs constantly (previous period, previous row, running totals).
- Use **ALL for calculations that should ignore the surrounding visual entirely** (grand totals, running totals across all data) and **ALLSELECTED for calculations that should respect user-applied filters/slicers but ignore only the visual's own internal grouping**.

## Anti-patterns
- **Debugging CALCULATE-based measures by breaking them into steps**: doesn't work — restructuring a single `CALCULATE` into two separate `CALCULATE` calls changes the actual result, so CALCULATE logic must be reasoned about as a black box or inspected with third-party tools instead.
- **Testing ISINSCOPE from the top of a hierarchy down**: a `SWITCH(TRUE(), ISINSCOPE(HigherLevelCol), ..., ISINSCOPE(LowerLevelCol), ...)` will always match the higher-level test first (it's "in scope" everywhere below it too) and never reach the lower-level branch — always order ISINSCOPE tests from bottom of hierarchy to top.
- **Adding computed columns directly inside SUMMARIZECOLUMNS that reference other columns from the same call**: fails with a "cannot be determined" error, because those columns aren't yet materialized inside the function call — wrap with `ADDCOLUMNS` (or a `VAR` holding the SUMMARIZECOLUMNS table, then a second `VAR` with ADDCOLUMNS) instead.

## Reference Tables

| Function | Purpose | Note |
|---|---|---|
| `SUMMARIZE('T', [Col], "Name", expr)` | Group + aggregate | Older, most flexible |
| `GROUPBY('T', [Col], "Name", SUMX(CURRENTGROUP(), expr))` | Group + aggregate | Requires X-aggregator + `CURRENTGROUP()` |
| `SUMMARIZECOLUMNS('T'[Col], "Name", expr)` | Group + aggregate | Generally preferred; cannot reference sibling computed columns |
| `ALL('Table')` | Remove all filters | Ignores visual entirely |
| `ALLSELECTED('Table')` | Remove internal-visual filters only | Respects external slicers |

## Worked Example
Fixing the Measure Totals Problem for `Sum Total Cost 2 = SUM('Table'[Total Cost]) - 2` (a per-row-correct, total-wrong measure):
```
Total Sum Cost 2 =
    VAR __Table =
        SUMMARIZECOLUMNS(
            'Table'[Item],
            "__TotalCost", [Sum Total Cost 2]
        )
    VAR __Result = SUMX( __Table, [__TotalCost] )
RETURN
    __Result
```
This regroups the data exactly as the Table visual does (by Item), evaluates the broken measure once per group into `__TotalCost`, then sums those already-correct per-row values — so the Total row now equals 60.80 (the true sum of the individual rows) instead of the naive 66.80 CALCULATE-style total would show. Swap `RETURN __Result` for `RETURN TOCSV(__Table)` to see the intermediate per-group values while debugging.

## Key Takeaways
1. CALCULATE is not banned because it's slow or wrong — it's avoided because it's a single opaque step that can't be decomposed for debugging; No CALCULATE keeps every step as an inspectable variable.
2. When a measure's output looks wrong, swap the RETURN target to `COUNTROWS(__Table)` or `TOCSV(__Table)` on the relevant variable before assuming the logic itself is broken.
3. Prefer `FILTER` + `MINX`/`MAXX` over `LOOKUPVALUE` for value lookups — fewer functions, more general, and composes with the rest of the No CALCULATE pattern.
4. ALL vs. ALLSELECTED is the lever for "ignore everything" vs. "ignore only this visual's own grouping" — pick based on whether user-applied slicers should still apply.
5. Prefer SUMMARIZECOLUMNS for grouping; reach for ADDCOLUMNS (layered via a VAR) only when a new column must reference another column computed in the same summarization.
6. Order ISINSCOPE tests bottom-of-hierarchy to top-of-hierarchy inside a SWITCH TRUE.
7. When a Table/Matrix visual's Total row doesn't match the sum of its rows, rebuild the visual's own grouping as a `SUMMARIZECOLUMNS` table variable, evaluate the measure per group, and SUMX over it.

## Connects To
- **Ch 1**: directly extends the No CALCULATE / VAR / RETURN pattern and the "double lookup" technique introduced there.
- **Ch 16 (AI, Debugging, and CALCULATE)**: returns to CALCULATE's "black box" nature in depth, including why nested CALCULATE can't be split into steps without changing results.
- **Running totals, previous-period comparisons**: the ALL/ALLSELECTED + double-lookup patterns here are reused throughout the Dates and Calendars chapter (Ch 3) and the scenario chapters.
