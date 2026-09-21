# Chapter 15: Optimizing Performance

## Core Idea
DAX performance work doesn't require understanding the Storage Engine (SE) and Formula Engine (FE) internals in depth — a small set of practical rewrite rules (SWITCH over nested IF, filter-first and filter-early, consolidate dependent measures, drop unnecessary functions) took one real-world measure from ~10 minutes to 6 seconds (99% faster), and the No CALCULATE approach frequently — but not universally — outperforms CALCULATE-based and native time-intelligence measures.

## Frameworks Introduced
- **The SE/FE mental model (just enough theory)**: the **Storage Engine** scans only needed columns from the compressed in-memory store, applies filters, and performs simple aggregations (SUM/COUNT/MIN/MAX) — fast, multi-threaded. The **Formula Engine** parses the DAX, manages row/filter context and context transitions, calls the SE for data, and handles anything the SE can't (FILTER, ADDCOLUMNS, iterators, complex row-by-row logic) — slower, single-threaded. **The optimization goal is to push as much work as possible into the SE and minimize FE-side row-by-row processing.**
- **Iterative measure refactoring as an optimization method**: rather than reasoning abstractly about SE/FE internals, the chapter's central worked example shows a *sequence* of eight successive rewrites of the same measure, each targeting one specific inefficiency and each independently measured — this "measure, change one thing, remeasure" loop is presented as the practical alternative to deep internals knowledge.
- **Filter-early, filter-often**: restructure logic so the cheapest, most row-eliminating test runs first (as a FILTER, not buried inside a SWITCH/IF), shrinking the row set the FE has to process on every subsequent step — repeatedly the single highest-leverage change across the chapter's example.

## Key Concepts
- **Performance Analyzer**: Power BI Desktop's built-in pane (View → Performance analyzer) reporting per-visual timing broken into DAX query / Visual display / Other / Evaluated parameters; only "DAX query" time is something your DAX code can influence. "Run in DAX query view" from a recorded result exposes the actual generated query (often very different from the measure's own DAX) for use in DAX Studio.
- **DAX Studio**: the standard free deep-inspection tool for SE-vs-FE timing breakdowns and query benchmarking — not covered in depth here, but flagged as the next-level tool once Performance Analyzer identifies a slow visual.
- **Power Optimizer (TruViz)**: a third-party AI-assisted tool that analyzes an entire semantic model (unused objects, relationships, measure dependencies) and proposes concrete fixes — mentioned as going beyond what manual profiling easily surfaces.
- **VALUE / VALUES avoidance**: `VALUE()` (text→number conversion) is "almost never a good idea" when the value is already numeric; prefer `DISTINCT` over `VALUES` in general — both were flagged as leftover habits from older DAX style that add FE overhead without benefit.
- **Measure fusion**: modern Power BI can sometimes optimize away the overhead of measures calling other measures, reducing (but not eliminating) the performance cost of "measure branching." Consolidating dependent measures into one still helped in the chapter's example (~7-10%).

## Mental Models
- Treat DAX optimization the same way you'd approach any performance problem: **measure first, change one thing, measure again** — the chapter's 8-step example is a worked demonstration of this loop, not a fixed checklist to apply blindly (the author explicitly notes these specific techniques won't always help in other scenarios).
- When comparing two working formulas with the same output, remember that **"more idiomatic No CALCULATE" is not automatically "faster"** — the CALCULATE vs No CALCULATE Act 2 example shows a No CALCULATE measure catastrophically failing to scale (resource-exceeded error at higher cardinality) where a CALCULATE-based measure using `USERELATIONSHIP` stayed fast, because CALCULATE's native relationship-switching capability let the SE do the heavy lifting that the No CALCULATE version had to emulate awkwardly in the FE.
- Ask **"how many rows does this eliminate, and how cheaply?"** for every logical test in a filter/SWITCH chain, and order tests accordingly — this single question drove several of the chapter's biggest speedups (No. of Orders 2→3, and 6→8).

## Anti-patterns
- **Nested IF statements for multi-branch logic**: `SWITCH(TRUE(), ...)` alone gave a ~90% speedup over equivalent nested IFs in the chapter's example — always prefer SWITCH(TRUE()) for 3+ branch conditional logic.
- **Splitting logic across multiple dependent measures ("measure branching") by default**: consolidating into a single measure gave a further ~7-10% gain in the example — the No CALCULATE philosophy's "single measure" preference has a measurable (if sometimes modest) performance basis, not just a readability one.
- **Testing/filtering broad conditions late, or inside SWITCH/IF rather than as an upfront FILTER**: moving the same logical test from inside a SWITCH branch to an explicit early FILTER call was consistently one of the largest wins (FILTER "is quite possibly the most powerful function in DAX when it comes to DAX optimization" — filter early and filter often).
- **Sprinkling VALUE()/VALUES() out of habit**: rarely adds correctness value in modern DAX and adds FE overhead — favor DISTINCT over VALUES, and avoid VALUE() on data that's already numeric.
- **Assuming the No CALCULATE approach always wins on performance**: Act 2's high-cardinality (730M row) scenario shows a case where CALCULATE + USERELATIONSHIP dramatically outperformed (and the No CALCULATE approach even failed outright at scale) — treat "No CALCULATE is usually faster" as a strong prior from Act 1, not an absolute rule, and validate at realistic data volumes.
- **Generalizing one optimization result to all scenarios**: the chapter explicitly warns that the specific techniques demonstrated "may not provide any benefit" in other circumstances — always validate with Performance Analyzer/DAX Studio on your own model rather than applying these rules unconditionally.

## Reference Tables

| Optimization step (from the chapter's worked example) | Change | Result |
|---|---|---|
| Baseline (nested IF, `VALUE()`, separate measures) | — | ~582,000 ms (9m 42s) |
| Nested IF → `SWITCH(TRUE(), ...)` | cleaner branching | 58,175 ms (−90%) |
| Reorder SWITCH tests (exclude-most-rows first) | test ordering | 27,424 ms (−50% more) |
| Consolidate dependent measures into one | single measure | ~25,000 ms (−7%) |
| Move exclusion test into an explicit early `FILTER` | filter before compute | 16,000 ms (−36% more) |
| Remove unnecessary `VALUE()` calls | drop dead function calls | 14,000 ms (−12% more) |
| Replace SWITCH with nested `FILTER` calls | FILTER over SWITCH | 10,000 ms (−28% more) |
| Simplify filter logic to its minimal form | logic simplification | 6,000 ms (−40% more; ~99% total) |

| Scenario | CALCULATE-based | No CALCULATE | Winner |
|---|---|---|---|
| YTD sales (Act 1, moderate data) | 309–653 ms | 43 ms | No CALCULATE (89–94% faster) |
| Cross-date-set ID exclusion (Act 2, 730M rows) | fast, scales with more dates selected | fails/errors at scale, or 27–63% slower when it works | CALCULATE + USERELATIONSHIP |

## Worked Example
The single highest-leverage rewrite in the chapter's main example — moving an exclusionary test from inside conditional branches to an explicit early FILTER, and simplifying the remaining logic to its essence:
```
Total Orders 8 =
    VAR __Date = MIN('DateTimeTable'[Date])
    VAR __Result =
        COUNTROWS(
            FILTER(
                'Tracking_History',
                'Tracking_History'[Start Date] < __Date && __Date < 'Tracking_History'[End Date]
            )
        )
RETURN
    __Result
```
Every earlier version (nested IFs, then SWITCH, then FILTER-plus-SWITCH) was computing the *same* logical answer — "does this order's date range contain the bucket date?" — through progressively more convoluted paths. Recognizing the whole four-branch AND/OR structure reduces to one inequality test (`start < date < end`) delivered the final, largest simplification: an order of magnitude faster than the already-optimized FILTER-based versions before it, and ~99% faster than the naive nested-IF original.

## Key Takeaways
1. You don't need deep SE/FE internals knowledge to optimize DAX — measure with Performance Analyzer, change one thing, remeasure.
2. Prefer `SWITCH(TRUE(), ...)` over nested IF for multi-branch logic — often the single biggest early win.
3. Order logical tests to eliminate/include the most rows as early and cheaply as possible.
4. Consolidate dependent measures into a single measure where reasonable.
5. FILTER, applied early, is "quite possibly the most powerful function in DAX" for optimization — push filtering before computation.
6. Drop unnecessary VALUE()/VALUES() calls; prefer DISTINCT over VALUES.
7. After mechanical optimizations, step back and ask whether the entire logical structure can be simplified to something conceptually smaller (as in Total Orders 8) — this often beats further micro-optimization of the existing structure.
8. No CALCULATE is usually — not always — faster; at very high cardinality, CALCULATE's native relationship-switching (USERELATIONSHIP) can outperform (and out-scale) a No CALCULATE emulation of the same logic. Validate with real data volumes before assuming either approach wins.

## Connects To
- **Ch 1–2 (No CALCULATE pattern)**: Act 1 provides the chapter's clearest performance justification for the book's overall philosophy — but Act 2 provides the necessary counter-example.
- **Ch 16 (AI, Debugging, and CALCULATE)**: continues directly from this chapter's CALCULATE-vs-No-CALCULATE performance comparison into a deeper treatment of when CALCULATE is actually appropriate.
- **Ch 3 (offsets, TOTALYTD)**: Act 1 reuses the offset-based YTD measure from Ch3 as the No CALCULATE performance baseline against CALCULATE and TOTALYTD.
