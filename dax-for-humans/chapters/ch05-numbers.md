# Chapter 5: Numbers

## Core Idea
DAX has a surprisingly large number-function surface (rounding alone has 11 functions) with real quirks (MOD and MEDIAN both misbehave in specific situations) and missing pieces (no MODE function) — this chapter maps the safe defaults and gives No-CALCULATE-style workarounds for each gap.

## Frameworks Introduced
- **Safe division**: always use `DIVIDE(numerator, denominator, alternate_result)` instead of `/`, unless the divisor is a hardcoded constant — the third parameter avoids divide-by-zero errors outright. For integer division protected against zero, use `INT(DIVIDE(n, d, BLANK()))` (or `TRUNC(...)` if negatives are possible), not `QUOTIENT` (which has no error-handling parameter).
- **Aggregating a measure (extends the Ch2 Measure Totals fix)**: you cannot `SUM([SomeMeasure])`; instead rebuild the visual's own grouping as a `SUMMARIZE`/`SUMMARIZECOLUMNS` table variable with a column evaluating the measure per group, then aggregate over that table.
- **Weighted Average**: `DIVIDE(SUMX(table, value * weight), SUMX(table, weight), 0)` — two X-aggregations and a DIVIDE, dramatically simpler than Power BI's own built-in "Weighted average by category" quick-measure (which nests CALCULATE/KEEPFILTERS).
- **Linear Interpolation for gaps in known data**: for a target x with no known y, find the nearest known `x0 < x` and `x1 > x` (via FILTER + MAXX/MINX double-lookups) and their y-values `y0`, `y1`, then apply `y0 + (x - x0) * DIVIDE(y1 - y0, x1 - x0)`.
- **Unique Ranking (no ties)**: build a composite row-identifier string (concatenate the columns that make a row unique), turn the table into a sorted path via `CONCATENATEX(..., delimiter, sort_expr, sort_order)`, then GENERATESERIES + PATHITEM to assign a strictly increasing rank per path position — works against virtual tables, unlike RANKX/RANK.

## Key Concepts
- **DIVIDE(num, denom, [alt])**: the safe-division standard; prefer it over `/` whenever the denominator isn't a literal constant.
- **INT vs. TRUNC**: identical for positive numbers; for negatives, `INT` rounds toward negative infinity (`INT(-2.1) = -3`) while `TRUNC` truncates toward zero (`TRUNC(-2.1) = -2`) — use TRUNC when negatives are possible.
- **ROUND/ROUNDUP/ROUNDDOWN**: ROUNDUP/ROUNDDOWN round away-from-zero/toward-zero respectively (not "more positive"/"more negative" — a common misreading for negative inputs); ROUND uses standard half-away-from-zero rounding; a negative second parameter rounds left of the decimal point.
- **MROUND(value, multiple)**: rounds to nearest multiple of the second parameter; **errors if the two parameters have different signs** — guard with `IF([Value] < 0, MROUND([Value], -m), MROUND([Value], m))`.
- **CEILING vs. ISO.CEILING**: identical except when both the value and multiple are negative — CEILING rounds away from zero, ISO.CEILING rounds toward zero; CEILING also errors on a negative multiple with a positive value (ISO.CEILING doesn't).
- **`=` vs. `==`**: `0 = BLANK()` is TRUE; `0 == BLANK()` is FALSE — knowing this "saves hours of troubleshooting" when a comparison behaves unexpectedly around blanks/zeros.
- **STDEVX.P/VARX.P vs. .S variants**: `.P` treats the data as the full population, `.S` as a sample — different denominators, different results.
- **RANKX vs. RANK.EQ vs. RANK**: RANKX (oldest, most flexible, most confusing — up to 5 params); RANK.EQ (Excel-style port); RANK (newest, recommended default — takes a tie-mode (`DENSE`/`SKIP`), a table/axis, an `ORDERBY(...)` clause, and an optional `PARTITIONBY(...)` for per-group ranking without RANKX's manual filtering gymnastics).
- **Dense vs. Skip ranking**: Skip (default) leaves gaps after ties (1,2,2,5 — mirrors "how many rows outrank me + 1"); Dense has no gaps (1,2,2,3).
- **LINEST / LINESTX**: least-squares linear regression, returning a table with `Slope`, `Intercept`, `CoefficientOfDetermination` (0–1, correlation strength) columns — first parameter is the dependent (y) variable, second the independent (x).
- **FORMAT(value, format_string, [locale])**: converts a number to formatted text using either predefined names ("Currency", "Percent", "Scientific"...) or custom format strings (`0`, `#`, `.`, `%`, `,`, `\`, `"..."` as placeholders); measures additionally support **Dynamic format strings** — a DAX expression that computes the format string itself (e.g. switch between `"0"` and a percent format depending on the measure's own value).

## Mental Models
- Think of DAX's rounding functions as **two families**: "round to N decimal places" (ROUND/ROUNDUP/ROUNDDOWN, TRUNC) vs. "round to the nearest multiple of M" (MROUND, CEILING/ISO.CEILING, FLOOR) — pick the family by what you're actually snapping to.
- **MEDIAN, MEDIANX, and the PERCENTILE family return a "variant" type that Power BI rejects in calculated *columns*** (though not measures) — when you hit the cryptic "Expressions that yield variant data-type cannot be used to define calculated columns" error, the fix is usually just `CONVERT(MEDIAN(...), DOUBLE)` to force a concrete type; only reach for the full path-based mathematical reconstruction of median when CONVERT isn't sufficient for the situation.
- **MODE doesn't exist in DAX** despite ~100 statistical functions — build it as SUMMARIZE-count-then-filter-to-max, exactly the "double lookup" shape from Ch1–Ch3.

## Anti-patterns
- **Using MOD directly with a decimal divisor**: MOD has a floating-point precision bug with decimals (alternates incorrectly between values) — build a custom "floating mod" via `ROUND(DIVIDE(value, divisor), significant_digits)` minus its truncation, times the divisor, falling back to plain MOD only when both operands are true integers.
- **Using QUOTIENT for safe integer division**: it lacks an error-fallback parameter (unlike DIVIDE); wrap DIVIDE with INT/TRUNC instead.
- **Assuming RANKX/RANK.EQ can rank per-group without manual filtering**: use RANK with `PARTITIONBY(...)` instead — replicating grouped ranking with RANKX is "daunting."
- **Multiplying pre-aggregated measures (`SUM(Price) * SUM(Quantity)`) and expecting correct row-level totals**: this reproduces the Measure Totals Problem (Ch2) in a very common real-world shape — fix with the SUMMARIZE-then-SUMX rebuild.

## Reference Tables

| Function | Rounds toward | Errors on |
|---|---|---|
| `INT` | -∞ | — |
| `TRUNC` | 0 | — |
| `ROUND(v, n)` | nearest (half away from 0) | — |
| `ROUNDUP(v, n)` | away from 0 | — |
| `ROUNDDOWN(v, n)` | toward 0 | — |
| `MROUND(v, m)` | nearest multiple of m | sign(v) ≠ sign(m) |
| `CEILING(v, m)` | up to multiple of m | negative m with positive v |
| `ISO.CEILING(v, m)` | up to multiple of m (toward 0 when both negative) | — |
| `FLOOR(v, m)` | down to multiple of m | same constraints as CEILING |
| `EVEN` / `ODD` | away from 0, to nearest even/odd int | — |
| `CURRENCY(v)` | ≈ `ROUND(v, 4)` in currency type | — |

| Custom format character | Meaning |
|---|---|
| `0` | digit or zero |
| `#` | digit or nothing |
| `.` / `,` | decimal / thousands separator |
| `%` | ×100 and show % |
| `\` | escape next character |
| `"ABC"` | literal text |

## Worked Example
Building a MODE function DAX lacks natively, with a tie-aware upgrade:
```
Improved Mode =
    VAR __Table = SELECTCOLUMNS( 'Mode Table', "__Value", [Value] )
    VAR __Summarized = SUMMARIZE( __Table, [__Value], "__Count", COUNTROWS('Mode Table') )
    VAR __Max = MAXX( __Summarized, [__Count] )
    VAR __Modes = FILTER( __Summarized, [__Count] = __Max )
    VAR __ModeCount = COUNTROWS( __Modes )
    VAR __Result = IF( __ModeCount > 5, "Many", CONCATENATEX( __Modes, [__Value], "," ) )
RETURN
    __Result
```
Group-count-filter-to-max is the same "double lookup" shape used for previous-period date lookups (Ch3) and value extraction (Ch4) — group the data, find the maximum of the grouping column, then filter back down to rows matching that maximum. Here it additionally handles ties by returning all tied values (capped at "Many" past 5) instead of picking one arbitrarily.

## Key Takeaways
1. Default to `DIVIDE(...)` over `/` everywhere except literal-constant divisors.
2. Know the two rounding families (decimal-place vs. nearest-multiple) and pick deliberately; watch MROUND/CEILING's sign-mismatch error cases.
3. MOD has a real decimal-precision bug — build the ROUND-based "floating mod" fix when dividing by non-integers.
4. MEDIAN/MEDIANX/PERCENTILE(X) functions can't be used directly in calculated columns (variant-type error) — wrap with `CONVERT(..., DOUBLE)`.
5. DAX has no MODE function — build it with SUMMARIZE + COUNTROWS + filter-to-max, and consider the tie-aware version by default.
6. Prefer RANK over RANKX for new work — its `PARTITIONBY` clause makes grouped ranking trivial where RANKX requires manual FILTER gymnastics; reach for the CONCATENATEX/PATHITEM technique only when true tie-free unique ranks are required.
7. Aggregating a measure across grouped rows is the Measure Totals Problem in disguise — rebuild the grouping as a table variable and SUMX over it.
8. Weighted average is just `DIVIDE(SUMX(value*weight), SUMX(weight))` — don't reach for Power BI's built-in quick measure, which is needlessly complex.

## Connects To
- **Ch 2**: the Measure Totals Problem fix is directly reused for "aggregating a measure" here.
- **Ch 3/Ch 4**: the double-lookup (filter → find extremum → filter again) and text-to-table/path techniques recur in Mode, Unique Ranking, and Linear Interpolation.
- **Ch 9 (Human Resources) / Ch 15 (Complex Patterns)**: weighted averages, ranking, and regression resurface in KPI calculations (e.g. Kaplan-Meier, TRIMMEAN) later in the book.
