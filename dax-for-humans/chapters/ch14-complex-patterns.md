# Chapter 14: Complex Patterns

## Core Idea
This chapter is proof-of-concept that DAX — without CALCULATE and without native recursion, loops, or guaranteed sort order — can still solve problems long considered impossible in the language: recreating missing Excel statistical functions (GAMMA, TRIMMEAN), fuzzy text matching, a sortable index, streak detection, and multi-column/dynamic-width aggregation. Each solution is a creative reapplication of techniques from earlier chapters (text-to-table, running totals, while-loop emulation), not new DAX primitives.

## Frameworks Introduced
- **Numerical approximation in place of true recursion**: DAX cannot implement a genuinely recursive mathematical definition (the gamma function's integral). Use a known closed-form numerical approximation instead (Lanczos approximation for GAMMA) — the general lesson: when a formula is inherently recursive, look for an established non-recursive numerical method (the book also cites Runge-Kutta for differential equations) rather than trying to force DAX into recursion.
- **Decomposing an aggregate into its algebraic components (TRIMMEAN)**: not every statistic can be computed by "filter a table then AVERAGEX it" — TRIMMEAN requires ranking values, determining a symmetric trim count (rounded down to an even number, Excel-specific rounding), building cumulative-count-from-each-end columns via a double concurrent while-loop (Ch11 technique), and finally computing `(Sum − TrimmedBottom − TrimmedTop) / (Count − 2×Trim)` directly rather than through AVERAGEX. Lesson: recognize when a "mean" is actually "sum divided by count" decomposed into filtered pieces, not a single AVERAGEX call.
- **Custom fuzzy text matching without Power Query**: expand each candidate string into a table of its own left-anchored substrings of increasing length (3 chars, 4 chars, ... full length) via GENERATESERIES+LEFT, SEARCH each substring against the target text, and take the *longest* substring that matches — approximates fuzzy matching with pure DAX; supplement with formal algorithms (Jaccard similarity via character-set INTERSECT/UNION ratio, Levenshtein-distance approximation via EXCEPT-based character difference count) when a more standard algorithm is preferred, but the book found its custom substring-growth approach outperformed both formal algorithms on real data.
- **DAX Index (a sortable, stable row index)**: previously considered impossible because DAX has no guaranteed row order outside sort-aware functions — solved by converting the table to a `CONCATENATEX`-built path string (optionally with its own ASC/DESC sort parameters), then rebuilding a table via `GENERATESERIES` + `PATHITEM` where the series position **is** the guaranteed index. Handles duplicates naturally (each duplicate gets its own path position) and extends to multi-column tables by concatenating column values with a private delimiter (e.g. `~`) before pathifying, then splitting back apart after.
- **Streak Detection ("Cthulhu")**: find how many consecutive rows (by an ordering column) belong to the same "group" ending at the current row.
  - How (general-purpose, non-consecutive-index-safe version — "Bride of Cthulhu"): find `__StreakStart` = the largest index value, among all rows with a *different* group and a smaller index than the current row, via `MAXX(FILTER(ALL(table), index<current && group<>currentGroup), index)`; then count rows with `index > __StreakStart && index <= current` — that count is the current streak length.
  - Extends to "Longest Streak" by computing this per-row as a column, then taking the max (optionally per group, using HASONEVALUE to detect single- vs. multi-group context).
- **Multi-column / dynamic-width aggregation**: to aggregate across many similarly-named columns (Jan/Feb/Mar/..., or 144 timestamped sensor-reading columns), either brute-force `UNION` of `SELECTCOLUMNS` calls (fine for a handful of columns) or, for large/variable column counts, dynamically discover and unpivot columns via `TOCSV` (renders the whole table, including headers, as a path-like text blob) + text-parsing (locate where "info" columns end and "data" columns begin by counting commas before the first colon in a timestamp-style header) + the text-to-table technique, entirely without hardcoding column names.

## Key Concepts
- **Lanczos approximation**: the standard numerical method for computing the gamma function without infinite recursion — includes a fast-path (`FACT(z-1)`) when the input is an integer, since gamma equals factorial there.
- **TRIMMEAN's rounding rule**: rounds the excluded-point count down to the nearest *even* number for top/bottom symmetry — explicitly not the same as MROUND, which doesn't match Excel's documented TRIMMEAN rounding behavior.
- **Jaccard similarity**: `|intersection| / |union|` of two strings' character sets (position-aligned in this implementation, found empirically to outperform position-ignorant matching).
- **Levenshtein distance (approximated)**: edit-distance-like count via `EXCEPT` between the two strings' character tables (a simplification of true Levenshtein, which formally counts minimum insert/delete/substitute operations).
- **PATHITEM / PATHLENGTH / CONCATENATEX with sort parameters**: the toolset underlying DAX Index — CONCATENATEX's 4th/5th parameters (expression, sort order) impose an explicit, guaranteed order into a path string that GENERATESERIES+PATHITEM can then read back positionally.
- **TOCSV as a table-to-text serialization tool**: beyond its Ch2 debugging use, TOCSV (with header row control) is repurposed here as the entry point for a fully dynamic, column-name-agnostic multi-column unpivot/aggregate.

## Mental Models
- When a problem is "impossible" in DAX (recursion, guaranteed order, missing built-in functions), the fix is almost always **route the data through a text/path representation where the needed guarantee (order, structure) can be encoded explicitly**, then convert back to a table — this single idea underlies DAX Index, the multi-column dynamic aggregation, and echoes the text-to-table technique from Ch4 used throughout the book.
- Streak detection reframes "how long is my current run" as **"how far back until the group last changed"** — a single MAXX/FILTER lookup for the boundary, then a COUNTROWS between that boundary and the current row; this is more robust than trying to increment a running counter row-by-row (which DAX can't do imperatively anyway).
- Not every "compute a statistic over a filtered table" problem fits the FILTER-then-X-aggregator mold directly (TRIMMEAN) — when it doesn't, break the statistic into its raw algebraic components (sum, count, adjustments) and assemble the final division yourself.

## Anti-patterns
- **Trying to implement genuinely recursive definitions directly in DAX**: doesn't work — substitute an established non-recursive numerical approximation (Lanczos for gamma, Runge-Kutta for differential equations) instead of fighting the language.
- **Using MROUND to replicate Excel's TRIMMEAN exclusion-count rounding**: produces different (wrong) results — Excel's specific "round down to nearest even" rule for TRIMMEAN must be implemented explicitly (`IF(ISODD(points), points-1, points)`), not assumed equivalent to MROUND.
- **Assuming position-ignorant character-set comparison always beats position-aligned comparison for fuzzy matching**: the book explicitly found the opposite in practical testing for their Jaccard implementation — validate fuzzy-matching heuristics against real data rather than assuming the "more sophisticated" approach wins.
- **Hardcoding column names for wide/many-column unpivot-and-aggregate**: works for a handful of columns (Jan–Apr) but doesn't scale to 144+ columns or variable schemas — use the dynamic TOCSV-based technique instead when column count is large or not fixed in advance.

## Reference Tables

| Problem | Core technique |
|---|---|
| GAMMA (missing recursive function) | Lanczos numerical approximation, integer fast-path via FACT |
| TRIMMEAN (missing Excel function) | rank + symmetric trim count + cumulative bottom/top counts (double while-loop) + direct sum/count division |
| Fuzzy matching (custom) | growing left-anchored substrings + SEARCH + longest-match selection, with tunable thresholds |
| Fuzzy matching (Jaccard) | `COUNTROWS(INTERSECT(charsA,charsB)) / COUNTROWS(UNION(charsA,charsB))` |
| Fuzzy matching (Levenshtein-approx) | `COUNTROWS(EXCEPT(longerChars, shorterChars))` |
| DAX Index (sortable, dedupe-safe) | CONCATENATEX(sorted path) → GENERATESERIES + PATHITEM, series position = index |
| Streak length | `MAXX(FILTER(ALL, idx<current && group<>currentGroup), idx)` boundary, then COUNTROWS between boundary and current |
| Multi-column aggregation (few cols) | UNION of SELECTCOLUMNS per column |
| Multi-column aggregation (many/dynamic cols) | TOCSV → locate info/data column boundary via comma-counting → text-to-table → X-aggregate |

## Worked Example
DAX Index — building a guaranteed, sortable row index where none is natively possible, including safe handling of duplicate values:
```
DAX Index Duplicates Sorted Table =
    VAR __Table = UNION('Index', 'Index')                            -- duplicated for demo
    VAR __Path = CONCATENATEX(__Table, [Product], "|", [Product], ASC)  -- sorted path
    VAR __Result =
        ADDCOLUMNS(
            SELECTCOLUMNS(GENERATESERIES(1, COUNTROWS(__Table), 1), "Index", [Value]),
            "Product", PATHITEM(__Path, [Index])
        )
RETURN
    __Result
```
`CONCATENATEX`'s 4th/5th parameters (`[Product], ASC`) impose an explicit sort order into the pipe-delimited path — something no other DAX function outside a query's ORDERBY can guarantee. `GENERATESERIES(1, N, 1)` then supplies a genuinely sequential index whose row-by-row `PATHITEM` lookup reads back the sorted, duplicate-safe order. Each duplicate "Apple" gets its own distinct index (1 and 2) rather than collapsing.

## Key Takeaways
1. When DAX lacks a needed built-in (GAMMA, MODE from Ch5, TRIMMEAN, fuzzy matching, a sortable index), look for a known numerical/algorithmic substitute rather than assuming the problem is unsolvable.
2. Route data through a CONCATENATEX-built (optionally sorted) path string whenever you need a guarantee — order, uniqueness handling, structure — DAX tables don't natively provide.
3. Streak/run-length detection is a boundary lookup (last group-change position) plus a row count, not an imperative increment.
4. Decompose statistics that don't fit FILTER-then-X-aggregator (like TRIMMEAN) into their raw sum/count components and assemble manually.
5. For wide or variable-schema tables, use TOCSV + text parsing to build fully dynamic, column-name-agnostic aggregation instead of hardcoding column references.
6. Every technique in this chapter — however elaborate — still avoids CALCULATE entirely, reinforcing the book's central thesis that CALCULATE is never structurally required, even for the hardest problems.

## Connects To
- **Ch 4 (Text)**: text-to-table (GENERATESERIES + PATHITEM/MID) is the direct foundation of DAX Index, fuzzy matching, and multi-column aggregation.
- **Ch 5 (Numbers)**: MODE's "missing function, build it" framing directly parallels GAMMA and TRIMMEAN here.
- **Ch 11 (Operations)**: TRIMMEAN's double concurrent while-loop reuses the running-total/threshold-crossing while-loop emulation from Ch11 directly.
- **Ch 16 (AI, Debugging, and CALCULATE)**: this chapter's closing argument ("CALCULATE was never required, even here") sets up the book's final deep-dive into why CALCULATE is avoidable.
