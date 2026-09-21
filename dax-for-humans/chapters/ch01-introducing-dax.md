# Chapter 1: Introducing DAX

## Core Idea
DAX is taught around CALCULATE by tradition, but you can write almost any DAX calculation with a single repeatable pattern — filter a table into a variable, then aggregate over it with an X-function — without ever touching CALCULATE.

## Frameworks Introduced
- **The No CALCULATE Pattern**: the book's central, reusable formula shape for building measures.
  - When to use: any measure that needs to filter rows and then aggregate a value — i.e. most DAX measures.
  - How:
    1. Create one or more `VAR`s for any inputs/constants the calculation needs (e.g. a value to exclude).
    2. Create a table `VAR` that filters (and/or groups) the rows the calculation needs, typically with `FILTER`.
    3. Use an "X aggregator" (`SUMX`, `MINX`, `MAXX`, `AVERAGEX`, `COUNTX`, …) over that table variable to produce the final scalar, assigned to a `__Result` variable and returned via `RETURN`.

## Key Concepts
- **DAX**: Data Analysis eXpressions — Power BI's formula language; "thinks" in tables/rows/columns, not cells (unlike Excel).
- **Column vs. Measure**: columns are computed once (at creation/refresh) and live in one table; measures are computed dynamically per query context and can conceptually live in any table.
- **Context**: the set of filters currently affecting a calculation. Comes from row position (row context, for columns), from visuals (internal/external filter context), or from filters written into the DAX itself.
- **Row context**: when a column calculation references `[ColumnName]`, it implicitly means "the value of that column on the current row."
- **X aggregator**: the "X" suffix versions of aggregation functions (`SUMX`, `AVERAGEX`, `MINX`, `MAXX`, `COUNTX`) that take a table expression as the first argument and a per-row scalar expression as the second — more flexible than the non-X versions, which only take a column.
- **VAR / RETURN**: the modern way to structure DAX as named, ordered steps instead of deeply nested function calls.
- **DAX calculated table**: a table produced by a DAX formula (e.g. via `FILTER`), created with New table.

## Mental Models
- Think of DAX as operating over **tables, rows, and columns** — never an individual "cell" the way Excel does. To get a single value you must filter a table down to a row, then pull a column.
- Think of a measure's value as always answering "given the filters currently in effect (from visuals, slicers, or the DAX itself), what does this calculation evaluate to?" — the same measure formula returns different numbers in different visual contexts.
- `SUM('Table'[Price])` and `SUMX('Table', 'Table'[Price])` are equivalent — the plain aggregator is just sugar syntax over the X version, which is why the X version is the more general tool to reach for.

## Anti-patterns
- **Learning DAX by starting with CALCULATE**: the author compares this to "trying to learn physics by starting with quantum mechanics" — CALCULATE is one of the most complex functions in DAX and is not needed to write most calculations.
- **Nesting functions instead of using VAR/RETURN**: repeats sub-expressions (e.g. the same `FILTER(...)` twice in one formula), hurts readability, can hurt performance, and makes debugging much harder.
- **Skipping the underscore naming convention for variables**: plain names like `Table` can collide with reserved words; Microsoft's own internal DAX queries use double-underscore-prefixed names (`__DS0Core`, `__Table`) — following this convention keeps variables unambiguous and matches how Performance Analyzer output reads.

## Reference Tables

| DAX Concept | Excel Analogue | Key Difference |
|---|---|---|
| Table/row/column | Cell / cell range | DAX has no direct "cell" reference; you filter to a row, then select a column |
| Column (calculated) | Formula filled down a column | Computed once at creation/refresh; static until refresh |
| Measure | — (no direct Excel equivalent) | Computed dynamically per query/visual context |

## Worked Example
Starting from a simple `Table` with columns `Item, Price, Quantity, Date`:
1. **Calculated column** — `Total Cost = [Price] * [Quantity]` (row context: multiplies each row's own Price and Quantity).
2. **Measure** — `Average Total Cost = AVERAGE('Table'[Total Cost])`. Placed in a Card visual it shows 12.57 (the overall average); clicking "Pickle" in a chart filters context and the card recalculates to 11.97 — demonstrating that measures, unlike columns, respond live to filter context.
3. **Explicit filter** — `Table 2 = FILTER('Table', 'Table'[Item] = "Banana")` creates a new calculated table of just the Banana rows. `&&` (and), `||` (or), `<>` (not) combine conditions, e.g. `FILTER('Table', 'Table'[Item] <> "Pickle")`.
4. **Applying the full pattern** — a measure that sums Total Cost for every row except "Pickle":
   ```
   Sum Total Cost No Pickle =
       VAR __ExcludeItem = "Pickle"
       VAR __Table = FILTER( 'Table', 'Table'[Item] <> __ExcludeItem )
       VAR __Result = SUMX( __Table, [Total Cost] )
   RETURN
       __Result
   ```
   Result: 38.89. This is the exact three-step shape (input VAR → filtered table VAR → X-aggregate → RETURN) that recurs throughout the rest of the book.

## Key Takeaways
1. Master the three-step pattern (filter into a table VAR, aggregate with an X function, return the result) — it solves the large majority of DAX measure requirements without CALCULATE.
2. Columns are static (computed at refresh); measures are dynamic (computed per filter context) — pick the right one deliberately.
3. Prefer X-aggregators over plain aggregators for flexibility; a plain `SUM` is just `SUMX` in disguise.
4. Use VAR/RETURN for every non-trivial measure: it avoids repeated sub-expressions, reads as ordered steps, and is far easier to debug than nested functions.
5. Prefix variable names with `_` or `__` (matching Microsoft's own internal DAX query convention) to avoid reserved-word collisions and keep formulas scannable.
6. Context (the filters affecting a calculation) can come from inside a visual, outside a visual (slicers/other visuals), or from the DAX expression itself — always ask "what's filtering this calculation right now?"

## Connects To
- **Ch 2**: builds directly on this pattern, contrasting it explicitly with the CALCULATE-based style and introducing more core functions used inside the same VAR/RETURN shape.
- **The No CALCULATE Pattern**: reused, extended, and applied to real-world KPIs in every scenario chapter (Customers, Human Resources, Projects, Finance, Operations, Distance and Space).
