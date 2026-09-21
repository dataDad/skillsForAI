# Chapter 3: Dates and Calendars

## Core Idea
DAX's ~30 built-in "time intelligence" functions (TOTALYTD, PREVIOUSMONTH, etc.) are fundamentally limited — they assume a standard Gregorian calendar, can't handle weeks, and don't work with single-table models — so the No CALCULATE approach replaces all of them with one general technique: **offsets**.

## Frameworks Introduced
- **The Offset Pattern**: represent every date as a signed integer distance from the current period (0 = current, -1 = one period ago, +1 = one period ahead), for whatever period granularity you need (year/quarter/month/week/day).
  - When to use: any period-over-period, period-to-date, rolling-average, or previous-period calculation — replaces the entire time-intelligence function family.
  - How: add an offset column per grain to your calendar table (e.g. `CurrYearOffset`, `CurrMonthOffset`, `CurrWeekOffset` — Melissa de Korte's `fnDateTable` Power Query function generates ~60 such columns including fiscal and ISO variants), then filter `ALL('Calendar')` down to rows where the offset column equals the target offset (0 for current, -1 for previous, a range for rolling windows).
  - Why it beats time intelligence: works with fiscal/custom calendars (a fiscal offset column is just built the same way, referenced from a different start month), works with weeks (which time intelligence functions don't support at all), works in single-table models (an offset can be computed dynamically inline, no separate date table required), and every step is an inspectable VAR.
- **Generalized Period-to-Date**: `VAR __Offset` (from the in-scope offset column, defaulting to 0 if multiple periods are in context) → `VAR __MaxDate` (today if current period, else the max date in that historical period) → `VAR __Table` = SUMMARIZE/FILTER the calendar to dates ≤ `__MaxDate` in that offset → SUMX/aggregate.
- **Generalized Previous-Period-to-Date**: same shape as period-to-date, but `__Offset` is one less than current, and `__MaxDate` is computed with `EDATE` (or a day-count difference for quarters/weeks) so the previous period is truncated to the *same relative point* as today, not the full period.
- **Rolling N-Period Average**: filter the offset column to a *range* (`__MinOffset` to `__MaxOffset`), group by the period label, then aggregate across periods with an X function (AVERAGEX for a rolling average, but SUMX/MEDIANX etc. work identically).

## Key Concepts
- **Measure table**: a dedicated empty table (via a blank query, an empty Enter Data query, or `{ "" }` as a calculated table) used to organize measures instead of scattering them across fact tables; hide its dummy column so Power BI treats it as a measures-only table.
- **Dates as numbers**: a DAX date is a whole number of days since Dec 30, 1899 — multiplying/adding by 1 reveals the underlying serial number; this is why dates support arithmetic (`Date2 - Date1` = day count).
- **Offset**: a signed integer distance from the current period for a given grain — the book's core date-handling primitive (see Frameworks).
- **EDATE(date, months)**: shifts a date forward/backward by whole months — the key tool for aligning "previous period to date" cutoffs.
- **EOMONTH(date, months)**: returns end-of-month N months away; `EOMONTH(date, -7) + 1` gives the first of the month 6 months prior.
- **Mark as date table**: a Power BI setting (right-click table → Mark as date table) that native time-intelligence functions require to work correctly — not needed for the offset approach, but harmless to set.
- **Auto-exist** (carried over from Ch2): affects why offset-based YTD-style measures can appear to respect slicers even without `ALLSELECTED`.
- **Julian Day**: a continuous day count from 4713 BC used in scientific contexts — conceptually just another offset system, which the author uses as further proof that offsets are the "proper" way to do date math.

## Mental Models
- Think of the calendar as a **number line per grain**: year offsets, quarter offsets, month offsets, week offsets all exist simultaneously as columns on the same calendar table, and you pick whichever grain's number line the current calculation needs.
- **"Full period" vs. "to-date" is just a `<=` vs `<` and a `__MaxDate` choice** — nearly every variant in this chapter (YTD vs. previous-YTD vs. previous-full-year) is the same `SUMMARIZE(FILTER(ALL('Calendar'), ...), ...)` skeleton with only the offset value and max-date expression changing.
- Dropping `- 1` from an `__Offset` calculation turns a "previous period" measure into a "current period" measure — the offset value is the single dial that shifts which period you're looking at.

## Anti-patterns
- **Using TOTALYTD/PREVIOUSMONTH/etc. for anything beyond a trivial standard-calendar report**: they silently return full-period sums instead of to-date sums for non-current periods (verified by comparing `TOTALYTD` output against `SUM(Value)` for a past year — they match, exposing that TOTALYTD isn't actually doing a to-date calculation for historical years), they don't support fiscal calendars (except TOTALYTD, partially), they don't support weeks at all, and they require a real date table (fail or behave oddly in single-table models).
- **Forgetting leap-year day-count drift in rolling to-date averages**: comparing "day 313 of a leap year" to "day 313 of a non-leap year" silently compares Nov 8 to Nov 9 — fix by computing the max date via `__MinDate + (Today - __MinCurrentPeriodDate)` (a date-arithmetic offset) rather than a raw day-of-year number.
- **Using INT() for negative number truncation**: `INT(-2.1)` returns -3 (rounds toward negative infinity), not -2 — use `TRUNC()` when truncating values that may be negative (relevant in the Julian Day conversion formulas).

## Reference Tables

| Calculation | Offset formula shape | Native DAX equivalent | Native DAX limitation exposed |
|---|---|---|---|
| Year-to-date | `FILTER(ALL('Calendar'), [Date]<=MaxDate && [CurrYearOffset]=0)` | `TOTALYTD` | Blank in a Card visual with no date in context; ignores fiscal start except via 4th param |
| Quarter-to-date | same, `CurrQuarterOffset` | `TOTALQTD` | No fiscal-calendar parameter at all |
| Month-to-date | same, `CurrMonthOffset` | `TOTALMTD` | No fiscal-calendar parameter |
| Week-to-date | same, `CurrWeekOffset` | *(none)* | Time intelligence has no week functions |
| Previous year (full) | `[CurrYearOffset] = -1` (via `MAX(offset)-1`), `ALL('Calendar')` | `PREVIOUSYEAR` inside `CALCULATE` | Standard calendar only |
| Previous quarter/month (full) | same pattern with respective offset column | `PREVIOUSQUARTER` / `PREVIOUSMONTH` | Standard calendar only |
| Previous period **to date** | offset - 1, plus `EDATE`/day-count-aligned `__MaxDate` | *(no direct native equivalent)* | — |
| Rolling N-period average | offset column filtered to a **range** `[__MinOffset, __MaxOffset]`, grouped, `AVERAGEX` | *(no direct native equivalent)* | — |

## Worked Example
Generalized Year-to-Date, the pattern every other period-to-date/previous-period measure in this chapter is a variation of:
```
Year To Date 3 =
    VAR __Offset =
        IF( HASONEVALUE('Calendar'[Year]), MAX('Calendar'[CurrYearOffset]), 0 )
    VAR __Today = TODAY()
    VAR __MaxDate = IF( __Offset = 0, __Today, MAX('Calendar'[Date]) )
    VAR __Table =
        SUMMARIZE(
            FILTER( 'Calendar', [Date] <= __MaxDate && [CurrYearOffset] = __Offset ),
            [Date],
            "__Value", SUM('Table'[Value])
        )
    VAR __Result = SUMX( __Table, [__Value] )
RETURN
    __Result
```
- `__Offset` resolves the *current* year in context (0 if a single year is filtered, else defaults to the present year).
- `__MaxDate` caps the range at today for the current year, or at the full last date of the period for past years — this is what makes it behave identically to `TOTALYTD` for the current year while still being explicit and debuggable for past years.
- Swap `CurrYearOffset` for `CurrQuarterOffset`/`CurrMonthOffset`/`CurrWeekOffset` (with `EDATE`/day-count adjustments for "to-date" truncation) to get every other period-to-date/previous-period variant in the chapter — including week-to-date, which native DAX cannot do at all.

## Key Takeaways
1. Build an offset column per date grain (year/quarter/month/week) on the calendar table; every date-intelligence calculation in the rest of the book is a filter on one of these columns.
2. Offsets work with fiscal calendars, weeks, and single-table models — the three cases where native time-intelligence functions break down or don't exist.
3. "To-date" vs. "full period" is controlled by the `__MaxDate` variable (today vs. period-end); "current" vs. "previous" is controlled by the offset value (0 vs. -1) — learn these two dials and you can derive every variant.
4. Verify a new offset-based measure against its native time-intelligence counterpart where one exists (TOTALYTD, PREVIOUSYEAR, etc.) — the chapter uses this comparison repeatedly to expose where the native function's behavior is actually wrong or inconsistent.
5. Watch for leap-year drift when comparing "to-date" windows across years by day-of-year count; align by date arithmetic (`__MinDate + (Today - __MinCurrentDate)`) instead of raw day numbers.
6. Use `TRUNC` instead of `INT` whenever a calculation might produce a negative number.
7. Rolling-period calculations are just the period-to-date pattern with the offset filter widened to a range instead of a single value.

## Connects To
- **Ch 1 / Ch 2**: every date measure here is a direct application of the No CALCULATE VAR/FILTER/X-aggregator pattern and the ALL/ALLSELECTED and double-lookup techniques.
- **Ch 6 (Time and Duration)**: extends offset-style thinking to intra-day time calculations.
- **Scenario chapters (Customers, HR, Projects, Finance, Operations)**: reuse period-to-date, previous-period, and rolling-average measures as building blocks for KPIs like churn, YoY growth, and trailing averages.
- **Previous Row or Occurrence**: the `MAXX(FILTER(ALL(...), [Date] < __Current), [Date])` double-lookup pattern here is the direct ancestor of the "previous row" techniques reused in Ch 12 (Distance/Streaks) and elsewhere.
