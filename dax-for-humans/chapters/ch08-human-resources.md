# Chapter 8: Human Resources

## Core Idea
HR KPIs (turnover, absenteeism, satisfaction, HCVA, utilization, Kaplan-Meier survival, Gini pay equality) are all still the Ch1 "filter into a virtual table, aggregate with X" pattern — the added complexity here comes entirely from correctly handling **partial-period membership** (employees hired/terminated mid-period) and **sentinel/placeholder dates**, not from new DAX primitives.

## Frameworks Introduced
- **Partial-period clamping**: whenever an entity (employee, absence) may start or end outside the analysis period, clamp its effective range with `__Min = IF(start < periodStart, periodStart, start)` and `__Max = IF(end > periodEnd, periodEnd, end)`, then measure the clamped range (NETWORKDAYS, day-count) instead of the entity's full raw range — reused in Absenteeism, HCVA, and (implicitly) Bradford Factor.
- **Sentinel-date pattern**: use a far-future placeholder (e.g. `12/31/9999`) instead of BLANK for "not yet terminated" — simplifies filter logic (`[Term Date] > __End` works directly) at the cost of needing to know the sentinel value; blank/null term dates would require extra `OR ISBLANK(...)` branches.
- **Four-scenario interval overlap filter**: to find records (absences) overlapping an arbitrary period when both the record and the period have their own start/end, OR together the four ways two intervals can overlap: contained-within, overlapping-the-start, overlapping-the-end, and spanning-the-whole-period. This generalizes to any "does interval A intersect window B" problem.
- **Running-product-over-time (Kaplan-Meier survival)**: analogous to a running total (Ch2/Ch3) but with `PRODUCTX` instead of `SUMX` — build a per-time-increment table of `1 - d(i)/n(i)` (non-survival rate) and multiply across all increments up to the current one.
- **Lorenz Curve / Gini Coefficient via Riemann sum**: approximate the area under an empirical distribution curve by treating each percentile step as a unit-width rectangle and summing (`SUMX`) its height — then `Gini = A / (A+B)` where `B` is that summed area and `A = TotalTriangleArea - B`.

## Key Concepts
- **ETR (Employee Turnover Rate)**: `Termed / ((StartingHeadcount + EndingHeadcount)/2)` — headcounts computed with `ALL('Employees')` regardless of any Dates relationship, as a defensive habit for more complex models.
- **NETWORKDAYS(start, end, [weekend_code], [holidays])**: excludes weekends by default; the weekend-code parameter (table in this chapter) can shift which 1–2 days count as "weekend" but **cannot represent a 7-day work week** — for round-the-clock operations, use raw day-count `(__Max - __Min) * 1.` or `COUNTROWS(EXCEPT(GENERATESERIES(min,max,1), holidayDates))` instead.
- **Bradford Factor**: `Instances² × TotalDaysAbsent` — deliberately weights *frequent* short absences more heavily than infrequent long ones (same total days absent, more instances → higher score).
- **HCVA (Human Capital Value Added)**: `(Revenue - (TotalCost - FullyLoadedEmploymentCost)) / FTE`, where each employee's cost and FTE contribution are pro-rated by the fraction of the period they were actively employed (partial-period clamping again).
- **FTE (Full-Time Equivalent)**: `TotalHoursWorked / MaximumFullTimeHoursInPeriod` — the definition of "maximum full-time hours" is organization-specific (accounting for PTO/holidays reduces the nominal 2,080 hours/year figure).
- **Utilization %**: `BillableHours / (NETWORKDAYS(period) × DailyWorkHours × HeadcountInPeriod)`.
- **Kaplan-Meier estimator**: `Survival(t) = Π(1 - d(i)/n(i))` for all time increments i ≤ t, where d(i) = count exiting at exactly increment i, n(i) = count still "at risk" (survived past i) — a clinical-trials statistic repurposed here for employee-tenure survival curves.
- **Gini Coefficient**: 0 = perfect equality, 1 = perfect inequality; computed from the Lorenz curve (cumulative % of income held by the bottom X% of a population ranked by pay).
- **PRODUCTX(table, expr)**: the multiplicative counterpart to SUMX — evaluates expr per row and multiplies (rather than sums) the results.

## Mental Models
- Treat every "rate over a period with entities that may not span the full period" KPI (turnover, absenteeism, HCVA, utilization) as **clamp each entity's range to the period, then aggregate the clamped ranges** — this one habit handles the majority of real-world HR-period edge cases.
- The four-way interval-overlap OR clause is worth memorizing as a template: *contained*, *overlaps-start*, *overlaps-end*, *spans-entirely* — any two-date-range-vs-window intersection problem needs all four.
- Kaplan-Meier and running totals are the same shape (per-increment table → fold across increments) with the fold operator changed from `+` (SUMX) to `×` (PRODUCTX) — recognizing "is this additive or multiplicative accumulation?" tells you which X-function to reach for.

## Anti-patterns
- **Using BLANK/null for "still employed" instead of a sentinel date**: works, but every filter needs an extra `|| ISBLANK([Term Date])` branch — a sentinel date (with the team's agreement on its meaning) simplifies every downstream formula.
- **Averaging period-start and period-end headcount/cost as a shortcut for HCVA/FTE** (the book's "HCVA 2" alternative): simpler to write, but structurally less accurate than pro-rating each individual employee's actual days-active fraction — acceptable as a fast approximation, not as the default.
- **Forgetting the "+0" on COUNTROWS/FILTER-based measures that can return BLANK**: a measure that legitimately returns BLANK when a FILTER produces zero rows will be *invisible* in a Line/Bar chart by default — add `+0` deliberately when you want zero-value periods to render, and omit it deliberately when you want them excluded.
- **Trying to solve Gini/Lorenz curve area exactly via calculus**: the book explicitly rejects this ("you'd have to deal with differential equations so...hard pass") in favor of the Riemann-sum approximation — a reminder that a "good enough," debuggable DAX approximation beats a theoretically exact but impractical formula.

## Reference Tables

| KPI | Formula shape |
|---|---|
| ETR | `Termed / ((StartHeadcount + EndHeadcount)/2)` |
| Absenteeism | `AbsentWorkDays / TotalWorkDays`, both computed via clamped NETWORKDAYS |
| Bradford Factor | `Instances² × DaysAbsent` |
| Satisfaction | `SUM(SurveyAnswers) / (COUNTROWS × TopRating)` |
| HCVA | `(Revenue − (TotalCost − FullyLoadedCost)) / FTE`, both pro-rated |
| FTE | `TotalHoursWorked / MaxFullTimeHours` |
| Utilization % | `BillableHours / (NETWORKDAYS × DailyHours × Headcount)` |
| Kaplan-Meier Survival(t) | `PRODUCTX(incrementsUpToT, 1 - d(i)/n(i))` |
| Gini Coefficient | `A / (A+B)`, B = Riemann sum of Lorenz curve, A = triangle area − B |

## Worked Example
Absenteeism's four-scenario interval-overlap filter — the reusable template for "does this record's date range intersect the reporting window?":
```
FILTER(
    ALL('Absences'),
    [Employee] IN __EmployeeContext &&
    (
        ( [Start Date] >= __Start && [End Date] <= __End )                          -- contained
        || ( [Start Date] < __Start && [End Date] >= __Start && [End Date] <= __End )  -- overlaps start
        || ( [Start Date] >= __Start && [Start Date] <= __End && [End Date] > __End )  -- overlaps end
        || ( [Start Date] <= __Start && [End Date] >= __End )                        -- spans entirely
    )
)
```
Followed by clamping each surviving row's range with `__Min`/`__Max` and measuring `NETWORKDAYS(__Min, __Max)` — the same clamp-then-measure shape used for HCVA's per-employee active-days calculation.

## Key Takeaways
1. Clamp entity date ranges to the analysis period before measuring — the single most-repeated technique in this chapter (ETR, Absenteeism, HCVA).
2. Use a sentinel date for "still active/ongoing" rather than BLANK when you control the data model — it simplifies every downstream filter.
3. Memorize the four-scenario interval-overlap OR clause for any "does this date range intersect this window" problem.
4. NETWORKDAYS cannot represent a 7-day work week — fall back to raw day-count or EXCEPT-based holiday exclusion for round-the-clock operations.
5. PRODUCTX is SUMX's multiplicative sibling — reach for it whenever the accumulation across a time series should compound rather than add (survival curves, cumulative probabilities).
6. Add `+0` deliberately to control whether BLANK results (zero-count periods) render or are excluded in visuals — know which behavior you want, don't leave it to chance.
7. A Riemann-sum approximation (unit-width rectangles) is the practical way to compute "area under an empirical curve" in DAX — don't reach for calculus.

## Connects To
- **Ch 1–3**: every measure is still the No CALCULATE VAR/FILTER/X-aggregator pattern; NETWORKDAYS/EOMONTH-style date math extends Ch3's offset toolkit.
- **Ch 2 (Measure Totals)**: Satisfaction 2's incorrect total is fixed with the same SUMMARIZE-then-SUMX technique.
- **Ch 7 (Customers)**: the clamping technique here generalizes the partial-period thinking first seen in Customer churn/growth (previous-month boundaries via EOMONTH).
- **Ch 15 (Complex Patterns)**: TRIMMEAN and other statistical techniques later in the book build on the same "virtual table of per-entity computed values, then aggregate" shape as Gini/Kaplan-Meier here.
