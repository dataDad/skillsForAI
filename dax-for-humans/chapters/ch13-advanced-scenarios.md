# Chapter 13: Advanced Scenarios

## Core Idea
DAX's real power isn't limited to computing metrics — **disconnected tables** (tables with no data-model relationship to the rest of the semantic model) plus DAX measures can *simulate* relationships, invert slicer behavior, impose arbitrary filter logic, restructure a Matrix visual's own layout, and even generate visual elements (SVG) — because the "relationship" is defined entirely in a measure's DAX logic rather than in the model's static join graph.

## Frameworks Introduced
- **Disconnected-table-as-controller**: a table with no relationship to the fact table, used purely to drive a slicer or parameter; a measure reads the disconnected table's selected value(s) and decides how to filter/compute against the (unrelated) fact table — the relationship exists only inside the measure's logic, so it can be arbitrarily complex (not limited to single-column exact-match joins like real relationships).
- **NOT Slicer / NOT Aggregator**: invert a slicer's normal "show selected" behavior into "hide selected" by testing `IF(currentValue IN disconnectedSelectedValues, BLANK(), 1)` and using that as a visual-level filter (`show items when value = 1`), or the aggregator variant that sums only the *unselected* category's rows.
- **Complex Selector**: generalizes the NOT Slicer idea — any measure that returns 1/0 (or a value) per row of a visual, used as that visual's filter, can impose arbitrary custom selection logic (e.g. "show the selected week and the four weeks before it") that no built-in slicer/filter interaction could express.
- **AND Slicer (cohort intersection)**: default slicer selections OR together ("show rows matching any selected value"); to instead require *all* selected values to apply to the same entity (e.g. "patients diagnosed with every selected diagnosis code"), count each entity's matching-selected-values and compare to the total number of values selected — only entities whose count equals the full selection pass.
- **Custom Matrix Hierarchy**: when a Matrix visual's native column/row hierarchy can't express what you want (e.g. "show a Current-Year and Last-Year total column, but only at the grand-total level, not per month"), replace the Matrix's native column field with a disconnected table encoding your desired display structure (label + sort order + level), then write one measure with a `SWITCH`/`ISINSCOPE` cascade that computes the right value depending on which level of the custom hierarchy is in scope.
- **Dynamic Granularity Scale**: show recent data at fine granularity (week) and older data at coarse granularity (quarter/year) in the *same* visual/axis by building a calendar column that labels each date according to a rule ("if in the current quarter, label by week; else if in the current year, label by quarter; else label by year") and a matching numeric sort column, then aggregate (e.g. AVERAGEX) at that computed granularity.

## Key Concepts
- **Why disconnected tables matter**: native Power BI relationships only propagate filters via exact single-column matches — they can't express "this slicer selection changes which rows are summed based on custom logic." A disconnected table sidesteps this: the table supplies the slicer's UI, and a measure supplies all the "relationship" logic, unconstrained by relationship mechanics.
- **CALCULATETABLE**: CALCULATE's table-returning sibling — used in the chapter's most advanced AND-slicer variant to restore an outer row context after DISTINCT (which normally ignores row context but preserves filter context) — the book uses this specifically to contrast its complexity against the simpler COUNTROWS-based AND slicer, reinforcing the No CALCULATE preference.
- **GENERATE's context-propagation nuance**: for *related* tables, GENERATE evaluates the second table's expression within each row's context of the first table — this subtlety (not simply "Cartesian product") is what makes the advanced Cohort/AND-slicer formula work (and what makes it hard to read).
- **ISINSCOPE cascades for hierarchy-level detection**: used to determine which level of a custom (or native) row/column hierarchy is currently active in a Matrix cell, driving which calculation branch applies — extends Ch2's ISINSCOPE introduction to a full multi-branch SWITCH.
- **SVG-via-DAX**: build an SVG XML string (header + shape markup + footer) inside a measure, set the measure's Data Category to "Image URL", and Power BI renders it inline in Table/Matrix visuals — enables conditional icons (red-dot flags), animated elements (`<animate>`), and dynamically-scored graphics (star ratings) entirely from DAX string concatenation.

## Mental Models
- Whenever you catch yourself wanting a slicer/visual to behave in a way Power BI's native interaction model doesn't support (invert selection, require intersection instead of union, restrict to certain hierarchy levels only), the fix is almost always **a disconnected table for the UI + a measure that encodes the custom logic** — this is the chapter's single unifying idea across NOT slicers, AND slicers, complex selectors, and custom hierarchies.
- A Matrix visual's native hierarchy shows every measure at every level by default; to selectively suppress/show values by level, **replace the native hierarchy columns with a disconnected table whose rows encode exactly the levels/labels you want**, then branch on `ISINSCOPE` per level.
- Dynamic granularity is "labeling determines aggregation grain" — build the label (and a matching sort key) with a rule that changes based on recency, and any aggregation using that label automatically operates at mixed granularity without special-casing the visual.

## Anti-patterns
- **Reaching for the CALCULATETABLE-based AND-slicer formula as a default**: it works, but requires deep understanding of GENERATE's context-propagation nuance and CALCULATETABLE's row-context-restoring behavior — the book presents it specifically to demonstrate that the No CALCULATE alternative (SUMMARIZE + COUNTROWS comparison) is dramatically easier to reason about and debug, and should be preferred.
- **Trying to solve "show these totals only at the grand-total level" via Matrix visual formatting/settings alone**: not natively possible — requires the disconnected custom-hierarchy-table technique.
- **Pasting SVG snippets from online sources without checking for HTML-escaped colons** (`data&colon;image/svg+xml` instead of `data:image/svg+xml`): breaks the image rendering silently; always verify the literal colon character is present.

## Reference Tables

| Technique | Core mechanism |
|---|---|
| NOT Slicer | disconnected table + `IF(value IN selected, BLANK(), 1)` as visual filter |
| NOT Aggregator | `SUMX(FILTER(table, NOT(category IN selected)), value)` |
| Complex Selector | any 0/1 measure used as a visual-level filter, arbitrary logic |
| AND Slicer (simple) | `SUMMARIZE` per-entity match-count = total selected count |
| AND Slicer (CALCULATETABLE variant) | GENERATE + EXCEPT + CALCULATETABLE row-context restore |
| Custom Matrix Hierarchy | disconnected hierarchy table + `SWITCH`/`ISINSCOPE` cascade measure |
| Dynamic Granularity Scale | calendar column with rule-based label + matching sort key |
| SVG image measure | string-built XML, Data Category = Image URL |

## Worked Example
The AND Slicer's simple (No CALCULATE) version — patients matching *every* selected diagnosis, not just any:
```
AND Slicer =
    VAR __Diagnoses = COUNTROWS(DISTINCT('Diagnoses'[Diagnosis]))   -- # selected in slicer
    VAR __Patients =
        SUMMARIZE(
            'Diagnoses', [Patient],
            "__Diagnoses", COUNTROWS(DISTINCT('Diagnoses'[Diagnosis]))  -- # matching this patient
        )
    VAR __Table = FILTER(__Patients, [__Diagnoses] = __Diagnoses)   -- only patients matching ALL
    VAR __Result = CONCATENATEX(__Table, [Patient], ", ")
RETURN
    __Result
```
Compare against the CALCULATETABLE-based "Cohort" variant the book also shows: functionally identical output, but requiring GENERATE's context-propagation nuance and CALCULATETABLE's row-context restoration to understand — the book uses this side-by-side comparison specifically to argue for preferring the simpler SUMMARIZE-based version whenever an equivalent No-CALCULATE formula exists.

## Key Takeaways
1. Disconnected tables + measures let you define "relationships" with arbitrary DAX logic, unconstrained by native single-column-exact-match relationships.
2. NOT slicers, AND slicers, and complex selectors are all the same idea: a disconnected table drives the UI, a measure computes a 0/1 (or value) used as a visual-level filter.
3. To restrict which Matrix hierarchy levels show which measures, replace the native hierarchy with a disconnected table and branch on ISINSCOPE per level.
4. Dynamic (mixed) granularity in one visual is achieved by computing a label + sort key whose granularity rule depends on recency, then aggregating by that label.
5. When both a CALCULATE-based and a No-CALCULATE formula solve the same problem, prefer the No-CALCULATE version — even when it requires more variables, it stays legible and debuggable, which the chapter demonstrates explicitly with the AND Slicer comparison.
6. SVG images built entirely in DAX (header + shape + footer string, Data Category = Image URL) enable conditional icons, animation, and dynamic scoring visuals without external tools.

## Connects To
- **Ch 2 (ISINSCOPE, HASONEVALUE)**: extended here into full multi-branch SWITCH cascades for hierarchy-level detection.
- **Ch 16 (AI, Debugging, and CALCULATE)**: the AND Slicer's CALCULATETABLE comparison foreshadows the book's deeper critique of CALCULATE's opacity.
- **Ch 4 (Text)**: SVG string-building is a direct application of DAX string concatenation (`&`) techniques.
- **Ch 14 (Complex Patterns)**: continues with similarly advanced, technique-dense problems (DAX INDEX, streaks, fuzzy matching).
