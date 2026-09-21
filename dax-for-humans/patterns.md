# Patterns — DAX For Humans

## The No CALCULATE Pattern
**When to use**: any measure that filters rows and aggregates a value — the default shape for nearly every measure in this book.
**How**: `VAR __Table = FILTER(source, condition)` → `VAR __Result = XAggregator(__Table, [Column])` → `RETURN __Result`.
**Trade-offs**: more verbose than CALCULATE for simple cases; every step is independently inspectable/debuggable via RETURN-swap. (Ch 1)

## RETURN-Swap Debugging
**When to use**: a measure's output looks wrong and you need to inspect an intermediate step.
**How**: temporarily change `RETURN __Result` to `RETURN COUNTROWS(__TableVar)` or `RETURN TOCSV(__TableVar)`; revert once diagnosed.
**Trade-offs**: none — this is the primary reason the No CALCULATE style is preferred; not available with nested CALCULATE, whose steps can't be decomposed without changing the result. (Ch 2, Ch 16)

## Measure Totals Fix (semi-additive rebuild)
**When to use**: a Table/Matrix visual's Total row doesn't equal the sum of its displayed rows (non-additive expressions: `SUM(x) - constant`, ratios, per-row rates).
**How**: `VAR __Table = SUMMARIZECOLUMNS(groupingCols, "__Value", [BrokenMeasure])` → `SUMX(__Table, [__Value])`.
**Trade-offs**: requires knowing the visual's exact grouping columns in advance. (Ch 2, reused in Ch 5, 7, 9, 10)

## Offset-Based Date Intelligence
**When to use**: any period-to-date, previous-period, or rolling-average calculation — replaces native time-intelligence functions.
**How**: add an offset column per date grain (year/quarter/month/week) to the calendar table (0=current, negative=past); filter `ALL('Calendar')` to the target offset value or range.
**Trade-offs**: requires building offset columns up front; works with fiscal calendars, weeks, and single-table models where native functions fail. (Ch 3)

## Double Lookup
**When to use**: need to find a value based on a key that itself must first be looked up (previous period, previous row, MODE, min/max-matching lookups).
**How**: `VAR __Key = <lookup the key, e.g. MAX(Date)>` → `VAR __Result = <filter again on __Key, then MAXX/MINX to get final value>`.
**Trade-offs**: none; foundational and reused throughout the book. (Ch 2, reused constantly)

## Text-to-Table
**When to use**: any problem requiring table operations (FILTER, INTERSECT, CONCATENATEX) on the characters/words of a string.
**How**: `SUBSTITUTE(text, delimiter, "|")` → `PATHLENGTH`/`PATHITEM`, or `GENERATESERIES(1, LEN(text), 1)` + `MID` for char-by-char.
**Trade-offs**: none; a core building block reused for extraction, DAX Index, fuzzy matching, and more. (Ch 4)

## Attribute Extraction (locate-bound-extract)
**When to use**: pulling a labeled value out of unstructured/messy text where the label's position and presence vary.
**How**: `SEARCH` for the label → `SEARCH` for the next delimiter, defaulting to `LEN(text)+1` if none found → `MID` between start and end boundary.
**Trade-offs**: bespoke per label format; doesn't generalize to arbitrary grammars. (Ch 4)

## Reasonable-Maximum Search-Chaining
**When to use**: a pattern may occur an unknown number of times (0 to N) in a string.
**How**: chain N `SEARCH` calls, each starting after the previous match's position; wrap each single value with `{ }` and `UNION` into a table; `FILTER` out blanks.
**Trade-offs**: N must be chosen as a practical upper bound, not derived dynamically. (Ch 4)

## Safe Division
**When to use**: any division where the divisor isn't a literal constant.
**How**: `DIVIDE(numerator, denominator, alternateResult)` instead of `/`.
**Trade-offs**: none; always prefer DIVIDE. (Ch 5)

## Weighted Average
**When to use**: any weighted-average KPI (GPA, portfolio returns, WACC, CPI).
**How**: `DIVIDE(SUMX(table, value*weight), SUMX(table, weight), 0)`.
**Trade-offs**: dramatically simpler than Power BI's built-in "weighted average by category" quick measure. (Ch 5)

## Unique Ranking (tie-free, virtual-table-safe)
**When to use**: RANKX/RANK produce ties you need to break uniquely, including against virtual tables.
**How**: concatenate identifying columns into a composite row key; `CONCATENATEX(table, key, "|", sortExpr, sortOrder)` to build a sorted path; `GENERATESERIES` + `PATHITEM` to read back a strictly-increasing rank.
**Trade-offs**: more complex than RANK/RANKX; reach for it only when true uniqueness (no ties) is required. (Ch 5)

## Set-Comparison Segmentation
**When to use**: classifying entities as new/lost/returning, or computing churn/growth rate.
**How**: get distinct ID sets for current and reference periods; `EXCEPT(current, reference)` = new/gained; `EXCEPT(reference, current)` = lost; `INTERSECT` for retained/matching.
**Trade-offs**: none; the standard toolkit for period-over-period cohort analysis. (Ch 7)

## Invented Intermediate Rows (GENERATE expansion)
**When to use**: a fact table records only start/end events (ticket open/close, meeting start/end) but you need a value per intervening day/minute.
**How**: `GENERATE(factTable, FILTER(dateOrTimeSeries, within the row's start/end range))` to materialize one row per (event, date/time) pair.
**Trade-offs**: can be expensive at fine granularity (minute-level) on large data; use SELECTCOLUMNS to trim columns first. (Ch 7, 9)

## Partial-Period Clamping
**When to use**: computing a rate/ratio over a period where entities may start or end outside that period (employee turnover, absenteeism, HCVA).
**How**: `__Min = IF(entityStart < periodStart, periodStart, entityStart)`; `__Max = IF(entityEnd > periodEnd, periodEnd, entityEnd)`; measure the clamped range.
**Trade-offs**: none; essential for period-partial entities. (Ch 8)

## Four-Scenario Interval Overlap
**When to use**: testing whether a date range (absence, event) overlaps an arbitrary reporting window.
**How**: OR four conditions — contained-within, overlaps-start, overlaps-end, spans-entirely.
**Trade-offs**: verbose but complete; memorize as a template. (Ch 8)

## While-Loop Emulation
**When to use**: sequential accumulate-until-threshold problems (FIFO/LIFO fulfillment, bin allocation) — DAX has no loop construct.
**How**: build a table of candidate "iterations"; add a running-total column via `EARLIER`; add a decrement column (`runningTotal - target`); `MINX(FILTER(table, decrement>=0), [key])` finds the crossing point.
**Trade-offs**: doesn't guarantee row order for ties — pair with an explicit composite sort key when priority matters. (Ch 11)

## Minute/Day-Resolution Overlap Collapsing
**When to use**: computing true unique time/capacity covered by potentially-overlapping intervals (meetings, resource allocation).
**How**: `GENERATESERIES` at the needed grain (e.g. `1/24/60` for minutes) → `GENERATE` against all intervals, flag coverage per unit → `GROUPBY` + `MAXX(CURRENTGROUP(), flag)` to collapse overlaps to one flag per unit → sum.
**Trade-offs**: naive duration-summing overcounts overlaps; this is the correct alternative. (Ch 9)

## Hand-Rolled ATAN2
**When to use**: any angle/bearing/distance calculation — DAX's ATAN only covers two quadrants.
**How**: `SWITCH(TRUE(), x>0, ATAN(y/x), x<0&&y>=0, ATAN(y/x)+PI(), x<0&&y<0, ATAN(y/x)-PI(), x=0&&y>0, PI()/2, x=0&&y<0, -PI()/2, BLANK())`.
**Trade-offs**: none; reuse this block verbatim for polar coordinates, Haversine distance, and bearing. (Ch 12)

## Dimension-Order Normalization
**When to use**: comparing multi-dimensional entities (box dimensions) that may not share the same labeling convention.
**How**: sort each entity's dimensions into (min, mid, max) via `MINX`/`MAXX`/`EXCEPT` over a small inline table `{a,b,c}`; compare all three pairwise.
**Trade-offs**: only practical for small, fixed dimension counts (e.g. 3). (Ch 12)

## Bounded Transitive Closure
**When to use**: graph reachability (which nodes are reachable from X via any number of hops) — DAX doesn't recurse arbitrarily.
**How**: repeat the "find all reachable-from-current-set" UNION/FILTER expansion step once per required hop depth.
**Trade-offs**: must decide a maximum hop count up front; a fixed-depth formula misses deeper connections. (Ch 12)

## Disconnected-Table-as-Controller
**When to use**: needing slicer/filter behavior beyond what native relationships support (NOT slicers, AND slicers, complex selectors, custom hierarchies).
**How**: create a table with no relationship to the fact table; write a measure that reads the disconnected table's selection and applies custom filter logic to the (unrelated) fact table.
**Trade-offs**: can have performance impact; the only way to achieve behaviors native relationships can't express. (Ch 13)

## Custom Matrix Hierarchy
**When to use**: a Matrix visual's native row/column hierarchy can't express the desired display (e.g. totals-only measures).
**How**: replace the native hierarchy field with a disconnected table encoding label+order+level; branch on `ISINSCOPE`/`SWITCH` per level in one measure.
**Trade-offs**: requires careful hierarchy-table design to preserve calculation context. (Ch 13)

## Dynamic Granularity Scale
**When to use**: showing recent data at fine granularity and older data at coarse granularity in one visual/axis.
**How**: build a calendar label + sort column whose granularity rule depends on recency (e.g. current quarter → week, current year → quarter, else → year); aggregate by that label.
**Trade-offs**: label/sort logic must be kept in sync. (Ch 13)

## Numerical Approximation for "Recursive" Math
**When to use**: a function is mathematically defined recursively (gamma function, differential equations) and DAX can't recurse.
**How**: substitute an established non-recursive numerical method (Lanczos approximation, Runge-Kutta).
**Trade-offs**: approximation, not exact — verify accuracy against a reference implementation. (Ch 14)

## DAX Index (guaranteed, sortable, dedupe-safe)
**When to use**: needing a stable row index with guaranteed order — otherwise impossible in DAX.
**How**: `CONCATENATEX(table, key, "|", sortExpr, sortOrder)` to build a sorted path; `GENERATESERIES(1, N, 1)` + `PATHITEM` to read back positionally.
**Trade-offs**: multi-column tables need an extra delimiter-based split step. (Ch 14)

## Streak / Run-Length Detection
**When to use**: counting consecutive same-group rows ending at the current row (Cthulhu pattern).
**How**: `__StreakStart = MAXX(FILTER(ALL(table), idx<current && group<>currentGroup), idx)`; `COUNTROWS(FILTER(ALL(table), idx>__StreakStart && idx<=current))`.
**Trade-offs**: the "Bride of Cthulhu" version works with non-consecutive index/date columns; earlier versions required consecutive indices. (Ch 14)

## Dynamic Multi-Column / Wide-Table Aggregation
**When to use**: aggregating across many (or a variable number of) similarly-shaped columns without hardcoding names.
**How**: for a few columns, `UNION` of `SELECTCOLUMNS` calls; for many/variable columns, `TOCSV` the table to text, locate info/data column boundary via comma-counting, then text-to-table and X-aggregate.
**Trade-offs**: the dynamic version is complex but scales to arbitrary column counts; brute-force UNION doesn't scale past a handful of columns. (Ch 14)

## AI-Assisted DAX with Model Context
**When to use**: generating or optimizing DAX with an LLM.
**How**: export the semantic model's BIM file (Power BI Project `.pbip` save, or Tabular Editor) and attach it to the prompt before asking for DAX; iterate with follow-up corrections.
**Trade-offs**: BIM files can be large — strip linguistic metadata first. (Ch 16)

## Iterative Performance Refactoring
**When to use**: a measure is too slow and you don't know why.
**How**: measure with Performance Analyzer → change one thing (SWITCH over nested IF, reorder filter tests, consolidate measures, filter earlier, drop VALUE/VALUES, simplify logic) → remeasure → repeat.
**Trade-offs**: results are scenario-specific; don't assume a technique that helped once always helps. (Ch 15)
