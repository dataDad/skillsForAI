# Cheatsheet — DAX For Humans

## Core Decision Rule
**When writing any DAX measure, default to: `VAR __Table = FILTER(...)` → `VAR __Result = XAggregator(__Table, [Col])` → `RETURN __Result`.** Reach for CALCULATE only when you've accepted its debugging cost (see below) and want the convenience of re-evaluating an existing measure in a modified context.

## Debugging Decision Tree
1. Result looks wrong → swap `RETURN __Result` for `RETURN COUNTROWS(__TableVar)` (row count) or `RETURN TOCSV(__TableVar)` (contents) on the suspect variable.
2. Need to know what's filtering a visual right now → check the visual's Filters icon, or build a measure with `FILTERS()`/`ISFILTERED()`.
3. Need to trace an expression across a refresh/query → `EVALUATEANDLOG` + SQL Profiler trace.
4. Debugging with AI → attach the model's BIM file before prompting; never prompt without schema context.
5. Circular dependency error on a calculated column → check for CALCULATE inside it; switch to MAXX/FILTER/EARLIER, or add ALL(...) inside the CALCULATE.

## CALCULATE vs. No CALCULATE
| Situation | Use |
|---|---|
| Default / anything you'll need to debug or extend | No CALCULATE (VAR/FILTER/X-aggregator) |
| Re-evaluating an existing measure in a slightly different context, and you've internalized CALCULATE's rules | CALCULATE is fine |
| Nested CALCULATE calls | Know: innermost filter **overrides** the outer, unless `KEEPFILTERS` forces intersection — never assume filters simply combine |
| High-cardinality performance-critical relationship switching (e.g. `USERELATIONSHIP`) | CALCULATE can outperform (and out-scale) a No CALCULATE emulation — validate with real data volume |
| CALCULATE in a calculated column | Risk of circular-dependency errors; prefer MAXX+FILTER+EARLIER |

## Date Intelligence: Which Technique
| Need | Use |
|---|---|
| Any period-to-date / previous-period / rolling average | Offset columns (Ch3), not TOTALYTD/PREVIOUSMONTH/etc. |
| Fiscal calendar, week periods, or single-table model | Offsets — native time intelligence can't do these reliably |
| "To-date" (partial) vs. "full period" | Controlled by `__MaxDate` (today vs. period-end) |
| "Current" vs. "previous" | Controlled by offset value (0 vs. −1) |
| Rolling N-period average | Offset filtered to a *range*, grouped, AVERAGEX |

## Rounding: Which Function
| Need | Function |
|---|---|
| Truncate toward zero (handles negatives correctly) | `TRUNC` |
| Truncate toward −∞ (INT(-2.1) = -3, likely wrong for negatives) | `INT` — avoid unless value is known non-negative |
| Round to N decimals, standard rounding | `ROUND` |
| Round away from / toward zero explicitly | `ROUNDUP` / `ROUNDDOWN` |
| Round to nearest multiple of M | `MROUND` (same-sign only) / `CEILING`, `ISO.CEILING`, `FLOOR` |
| MEDIAN/MEDIANX/PERCENTILE(X) in a calculated column | Wrap with `CONVERT(..., DOUBLE)` |
| MOD with a decimal divisor | Don't use MOD directly — build the ROUND-based "floating mod" |

## Text: Which Function
| Need | Function |
|---|---|
| Locate substring, case-sensitive / insensitive | `FIND` / `SEARCH` |
| Boolean contains, case-sensitive / insensitive | `CONTAINSSTRINGEXACT` / `CONTAINSSTRING` |
| Count occurrences of a substring | `LEN(text) - LEN(SUBSTITUTE(text, target, ""))`, ÷ `LEN(target)` if multi-char |
| Turn a string into a table (per word/char) | `SUBSTITUTE`→pipe→`PATHITEM`, or `GENERATESERIES`+`MID` |
| Validate text is purely numeric | `ISERROR(VALUE(text))` |
| Preserve case in a case-insensitive engine | `UNICHAR(8203)` (zero-width space) injected near capitals — expensive, use sparingly |

## Thresholds & Defaults Cited in the Book
- APTR healthy range: **6–10** turns/year.
- FTE full-time hours/year baseline: **2,080** (40hrs×52wks), often reduced to **~1,896** after PTO/holidays.
- NPS bands: detractors 0–6, passives 7–8, promoters 9–10.
- Gamma/GAMMA Lanczos approximation accuracy: ~12–13 decimal places.
- TRIMMEAN exclusion rounding: down to nearest **even** number (not MROUND-equivalent).

## Tells & Smells
- Measure's Total row doesn't match the sum of its rows → Measure Totals Problem; rebuild via SUMMARIZECOLUMNS+SUMX (Ch2).
- `0 = BLANK()` is TRUE but `0 == BLANK()` is FALSE → know this before trusting a comparison involving possible blanks/zeros.
- COUNTROWS on an empty FILTER result returns BLANK, not 0 → add `+ 0` if you want zero-period rows to render in a chart; omit it if you want them excluded.
- A measure using `SUM(colA) * SUM(colB)` for a per-transaction cost/value → almost certainly the Measure Totals Problem in disguise; compute per row first.
- ISINSCOPE tests failing unexpectedly in a SWITCH TRUE → check test order; must go bottom-of-hierarchy to top.
- RANKX/RANK producing ties you didn't expect → decide if Dense vs. Skip is the actual issue, or if you need true tie-free Unique Ranking.
- A slow measure with nested IFs → try SWITCH(TRUE()) first; usually the single biggest win.
- A slow measure with SWITCH/IF logic gating a big table → move the elimination test into an explicit early FILTER instead.
- DATEVALUE on `"mmm-yy"` strings → silently wrong (misparses as day=year, year=current) — use `"mmm-yyyy"`.
- Same text differing only by case collapsing to one casing → Power BI/AS collation is case-insensitive everywhere except on-prem Analysis Services.

## Quick Reference: Core Formula Skeletons
```
-- No CALCULATE pattern
VAR __Table = FILTER(<source>, <condition>)
VAR __Result = SUMX(__Table, <expr>)     -- or AVERAGEX/MAXX/MINX/COUNTX/PRODUCTX
RETURN __Result

-- Double lookup
VAR __Key = MAX(<column>)                 -- or MAXX(FILTER(...), ...)
VAR __Result = MAXX(FILTER(<source>, [Key]=__Key), [TargetColumn])
RETURN __Result

-- Measure Totals fix
VAR __Table = SUMMARIZECOLUMNS(<groupCols>, "__V", [BrokenMeasure])
VAR __Result = SUMX(__Table, [__V])
RETURN __Result

-- Offset-based period-to-date
VAR __Offset = IF(HASONEVALUE('Cal'[Year]), MAX('Cal'[CurrYearOffset]), 0)
VAR __MaxDate = IF(__Offset=0, TODAY(), MAX('Cal'[Date]))
VAR __Table = SUMMARIZE(FILTER('Cal', [Date]<=__MaxDate && [CurrYearOffset]=__Offset), [Date], "__V", SUM('Fact'[Value]))
VAR __Result = SUMX(__Table, [__V])
RETURN __Result
```
