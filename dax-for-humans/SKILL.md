---
name: dax-for-humans
description: "Knowledge base from \"DAX For Humans\" by Greg Deckler. Use when writing or reviewing DAX for Power BI, applying the No CALCULATE pattern, building date-intelligence/customer/HR/finance/operations KPIs, debugging DAX measures, or deciding when (and when not) to use CALCULATE."
---

<!-- argument-hint: [topic, framework name, or chapter number] -->

# DAX For Humans
**Author**: Greg Deckler | **Pages**: ~565 | **Chapters**: 16 | **Generated**: 2026-09-21

## How to Use This Skill

- **Without arguments** — load the core frameworks below for reference while writing DAX.
- **With a topic** — ask about `offsets`, `churn`, `while loop`, `CALCULATE`, or another indexed topic; I find and read the relevant chapter file.
- **With a chapter** — ask for `ch03` or `dates`; I load that specific chapter.
- **Browse** — ask "what chapters do you have?" to see the full index.

When you ask about a topic not covered in Core Frameworks below, I will read
the relevant chapter file before answering.

---

## Core Frameworks & Mental Models

**The book's thesis**: DAX is traditionally taught starting with `CALCULATE`, the most complex function in the language — comparable to "teaching physics by starting with quantum mechanics." Instead, nearly every DAX problem in this book — from a basic measure to gamma-function approximation, fuzzy text matching, and while-loop emulation — is solved with **one repeatable pattern** and zero uses of `CALCULATE`.

### The No CALCULATE Pattern (the whole book in one shape)
```
VAR __Table = FILTER(<source>, <condition>)     -- or SUMMARIZE/GENERATE/GROUPBY
VAR __Result = SUMX(__Table, <expr>)            -- or AVERAGEX/MAXX/MINX/COUNTX/PRODUCTX
RETURN __Result
```
Build a virtual table by filtering/grouping, then aggregate it with an **X-aggregator** (`SUMX`, `AVERAGEX`, `MAXX`, `MINX`, `COUNTX`, `PRODUCTX`) — which take a table plus a per-row expression, unlike the non-X aggregators which only take a column. Prefer X-versions generally: `SUM(col)` is literally sugar for `SUMX(table, col)`.

### Why: debuggability, not dogma
Every step of a No CALCULATE measure is a named `VAR`, so you can debug by swapping what `RETURN` outputs — `RETURN COUNTROWS(__Table)` to check row counts, or `RETURN TOCSV(__Table)` to see the actual rows rendered as text. **This is the core justification for the whole approach**: `CALCULATE(expr, filter)` does the same filter-then-aggregate work in one opaque step that can't be decomposed — nested `CALCULATE` calls can't even be split into separate statements without changing the result (the innermost filter silently *overrides* the outer one, unless `KEEPFILTERS` forces an intersection — a rule you must simply know, not derive). CALCULATE also carries seven filter-modifier functions (`ALL`, `ALLEXCEPT`, `KEEPFILTERS`, `USERELATIONSHIP`, `CROSSFILTER`, `REMOVEFILTERS`, `ALLNOBLANKROW`) that form their own mini-language, usable mostly only inside CALCULATE.

**CALCULATE is not "banned."** Use it once you've internalized its rules and accept the debugging trade-off — e.g. re-evaluating an existing measure in a slightly modified context. Neither approach is inherently faster: No CALCULATE won a 94%-faster YTD benchmark in one scenario, but a CALCULATE+`USERELATIONSHIP` measure outperformed (and out-scaled) an equivalent No CALCULATE emulation at 730M rows in another. Validate at real data volumes rather than assuming either side always wins.

### Offsets replace time intelligence
DAX's ~30 native time-intelligence functions (`TOTALYTD`, `PREVIOUSMONTH`, ...) assume a standard calendar, can't do weeks, and don't work in single-table models. Instead: add **offset columns** per date grain to the calendar (0 = current period, −1 = previous, etc.), then `FILTER(ALL('Calendar'), [offset] = target)`. "To-date" vs. "full period" is controlled by the `__MaxDate` variable; "current" vs. "previous" by the offset value. This one technique underlies YTD, QTD, MTD, WTD, previous-period, previous-period-to-date, and rolling averages — and handles fiscal calendars and weeks, which native functions can't.

### The double lookup
Look up a key value (`MAX(Date)`), then filter again on that key to get the final value (`MAXX(FILTER(table, [Date]=__Key), [Value])`). This two-step shape recurs everywhere: previous-row/period lookups, MODE (DAX has no native MODE), reverse-engineering period deltas from cumulative data, and more.

### When DAX seems to lack a feature, don't assume it's impossible
DAX has no recursion, no guaranteed row order, no loop construct, and is missing functions Excel has (GAMMA, TRIMMEAN, MODE, ATAN2). The book's recurring move: **decompose the problem, or route it through a text/path representation that can encode the missing guarantee.**
- **No recursion** → substitute a non-recursive numerical method (Lanczos approximation for GAMMA) or decompose into independent per-entity contributions summed together (multi-year compound interest).
- **No guaranteed order** → `CONCATENATEX(table, expr, delim, sortExpr, sortOrder)` builds a sorted path string; `GENERATESERIES` + `PATHITEM` reads it back positionally — this is how a genuinely sortable **DAX Index** becomes possible, and the basis of text-to-table generally.
- **No loop** → encode "iterations" as table rows with a running-total column (via `EARLIER`) and a decrement/exit column; `MINX(FILTER(table, exit>=0), [key])` finds where the "loop" would exit (FIFO/LIFO fulfillment, bin-picking).
- **Missing function** → hand-roll it once as a reusable block (ATAN2 via quadrant `SWITCH`) and reuse everywhere needed.

### Debugging toolkit
1. **RETURN-swap** (covers ~90% of cases): change `RETURN __Result` to a scalar VAR or `TOCSV(__TableVar)`.
2. **Context inspection**: `FILTERS()`, `ISFILTERED()`, `ISCROSSFILTERED()`, or the visual's own Filters icon.
3. **EVALUATEANDLOG** + SQL Profiler for deep intermediate-value tracing (production measures should not keep this).
4. **AI-assisted**: attach the semantic model's BIM file (Power BI Project `.pbip` save, or Tabular Editor export) before prompting an LLM — dramatically improves generated-DAX usability over prompting without schema context.

### Performance, briefly
You rarely need Storage-Engine/Formula-Engine internals. Practical wins, in the order that mattered most in the book's worked 99%-faster example: `SWITCH(TRUE(), ...)` over nested IF → order logical tests to eliminate the most rows first → consolidate dependent measures → filter early via an explicit `FILTER` rather than inside SWITCH/IF → drop unnecessary `VALUE()`/`VALUES()` → simplify the logic's conceptual shape, not just its syntax.

---

## Chapter Index

| # | Title | Key Frameworks |
|---|-------|----------------|
| [ch01](chapters/ch01-introducing-dax.md) | Introducing DAX | The No CALCULATE Pattern, row vs. filter context |
| [ch02](chapters/ch02-more-core-concepts.md) | More Core Concepts | RETURN-swap debugging, Measure Totals Problem, ALL/ALLSELECTED |
| [ch03](chapters/ch03-dates-and-calendars.md) | Dates and Calendars | Offset Pattern, period-to-date, rolling averages |
| [ch04](chapters/ch04-text.md) | Text | Text-to-table, attribute extraction, collation/UNICHAR |
| [ch05](chapters/ch05-numbers.md) | Numbers | DIVIDE, rounding families, MODE, weighted average, unique ranking |
| [ch06](chapters/ch06-time-and-duration.md) | Time and Duration | Chelsie Eiden's Duration, time-as-fraction, Unix/UTC conversion |
| [ch07](chapters/ch07-customers.md) | Customers | EXCEPT/INTERSECT segmentation, invented intermediate rows |
| [ch08](chapters/ch08-human-resources.md) | Human Resources | Partial-period clamping, four-scenario interval overlap |
| [ch09](chapters/ch09-projects.md) | Projects | EVM (PV/EV/AC/SV/CV), minute-resolution overlap collapsing |
| [ch10](chapters/ch10-finance.md) | Finance | Semi-additive measures, granularity reconciliation, non-recursive compounding |
| [ch11](chapters/ch11-operations.md) | Operations | While-loop emulation, EARLIER, composite sort keys |
| [ch12](chapters/ch12-distance-and-space.md) | Distance and Space | Hand-rolled ATAN2, Haversine, transitive closure |
| [ch13](chapters/ch13-advanced-scenarios.md) | Advanced Scenarios | Disconnected tables, NOT/AND slicers, custom Matrix hierarchy, SVG |
| [ch14](chapters/ch14-complex-patterns.md) | Complex Patterns | GAMMA, TRIMMEAN, fuzzy matching, DAX Index, streaks |
| [ch15](chapters/ch15-optimizing-performance.md) | Optimizing Performance | SE/FE model, iterative refactoring, filter-early |
| [ch16](chapters/ch16-ai-debugging-and-calculate.md) | AI, Debugging, and CALCULATE | BIM-grounded AI prompting, CALCULATE's mini-language |

## Topic Index

- **AI-assisted DAX** → ch16
- **ATAN2 / bearing / distance** → ch12
- **CALCULATE (what it is, why avoided, when OK)** → ch02, ch15, ch16
- **Churn / retention / LTV / CAC** → ch07
- **Circular dependency** → ch16
- **Compound interest (non-recursive)** → ch10
- **Currency conversion (semi-additive)** → ch10
- **DAX Index** → ch14
- **Debugging (RETURN-swap, TOCSV, context)** → ch02, ch16
- **Disconnected tables / NOT slicer / AND slicer** → ch13
- **Earned Value Management (PV/EV/AC/SV/CV)** → ch09
- **Fuzzy matching** → ch14
- **GAMMA / TRIMMEAN (missing Excel functions)** → ch14
- **Gini coefficient / Kaplan-Meier** → ch08
- **MODE, MEDIAN quirks, MOD bug** → ch05
- **Offsets / period-to-date / rolling average** → ch03
- **Optimization (SE/FE, filter-early, SWITCH)** → ch15
- **OTIF / OCT / MTBF / OEE** → ch11
- **RANKX vs RANK vs RANK.EQ / unique ranking** → ch05
- **Streaks (run-length detection)** → ch14
- **SVG images via DAX** → ch13
- **Text-to-table / string extraction** → ch04
- **Time zones / Unix timestamps / duration formatting** → ch06
- **Transitive closure** → ch12
- **While-loop emulation / FIFO-LIFO fulfillment** → ch11

## Supporting Files

- [glossary.md](glossary.md) — all key terms with definitions and chapter references
- [patterns.md](patterns.md) — every named technique with When-to-use/How/Trade-offs
- [cheatsheet.md](cheatsheet.md) — decision rules, quick-reference tables, formula skeletons

---

## Scope & Limits

This skill covers the book's content only — extracted structure and synthesized summaries, not the book's text verbatim. For hands-on implementation, combine with the actual semantic model you're working in (table/column names, relationships) — none of this skill's examples know your model. This is a personal reference derived from a purchased, copyrighted book; do not redistribute or publish the generated chapter files.
