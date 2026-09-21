# Chapter 6: Time and Duration

## Core Idea
DAX has almost no native time/duration support (no duration type, only TIME/HOUR/MINUTE/SECOND/TIMEVALUE) — but because times are just fractional-day decimal numbers, nearly everything (duration math, time zones, Unix timestamps, work-hour calculations) can be built from plain arithmetic on those decimals plus the "Chelsie Eiden's Duration" formatting trick.

## Frameworks Introduced
- **Chelsie Eiden's Duration** (named for the Microsoft intern who devised it): the way to make a numeric duration *display* like `D:HH:MM:SS` while remaining a real, aggregatable number (not text).
  - How: decompose a decimal duration into `__Days`/`__Hours`/`__Minutes`/`__Seconds` via TRUNC/remainder arithmetic, then combine as `__Days*1000000 + __Hours*10000 + __Minutes*100 + __Seconds` (packed decimal digits, not concatenation), and apply a **Custom format string of `00:00:00:00`** to the measure. This keeps it a real number — so totals aggregate correctly, unlike a text-concatenated duration column.
  - When to use: any duration that needs to sum/total correctly in a visual while still displaying as D:H:M:S.
- **Seconds/Milliseconds ↔ Duration conversion**: repeated `TRUNC(remaining / unit_size)` + `MOD(remaining - whole*unit_size, unit_size)` steps, largest unit to smallest, feeding the same packed-digit "Chelsie Eiden" formatting.
- **Net Work Duration**: compute working minutes/hours between two datetimes, respecting a working-hours window (e.g. 7:30 AM–6 PM) and weekends.
  - How: `NETWORKDAYS` for the day count → classify start/end weekdays to determine how many of those days are "full" working days vs. partial → `DATEDIFF` for a full day's worth of minutes × full days, plus `DATEDIFF` for the partial start-day and end-day remainders → special-case same-day tickets.
- **Time Zone Conversion**: convert a source time to a destination time zone via UTC as an intermediate step: `UTCTime = SourceTime - SourceOffset/24`, then `DestTime = UTCTime + DestOffset/24` (division/multiplication by 24 converts hour-offsets to day-fraction units, consistent with the "time is a fraction of a day" fact from Time Basics).
- **Unix Timestamp Conversion**: `UTC = DATE(1970,1,1) + unix_seconds / (60*60*24)`, and the inverse `unix_seconds = (UTC_datetime - DATE(1970,1,1)) * 60*60*24`.

## Key Concepts
- **Time as a fraction of a day**: `TIME(17,21,36) * 1` reveals `.72` — DAX times are decimals between 0 and 1 representing the fraction of a 24-hour day elapsed, exactly like dates are whole-number day counts (Ch3).
- **TIME(h, m, s) rollover behavior**: seconds ≥60 roll into minutes, minutes ≥60 roll into hours, but hours ≥23 **cycle around the clock rather than rolling into the next day** — `TIME(41,21,36)` equals `TIME(17,21,36)`.
- **DATEDIFF(time1, time2, unit)**: supports HOUR/MINUTE/SECOND units (unlike DATEADD, which only supports DAY/MONTH/QUARTER/YEAR) — the one native function that handles sub-day time arithmetic directly.
- **Time table**: a CROSSJOIN of hour (0–23), minute (0–59), and second (0–59) series produces an 86,400-row time-of-day dimension table, analogous to a date/calendar table; recommended when source systems provide combined datetime stamps — split date and time into separate columns (each relating to its own dimension table) both for time-of-day analysis and to reduce column cardinality for better compression.
- **UTCNOW()**: current UTC datetime — combined with a stored UTC offset, the standard way to build a "last refreshed" timestamp for a report.
- **NETWORKDAYS**: counts working days (Mon–Fri) between two dates — has no native equivalent for working *hours*.

## Mental Models
- Once you accept "time is a decimal fraction of a day" (mirroring "date is an integer day count" from Ch3), every time/duration problem becomes ordinary arithmetic: add fractions to shift time, subtract and multiply by 86400 to get seconds, TRUNC/MOD to decompose into units.
- Duration display vs. duration value are separate concerns: **compute** in decimal (fraction-of-day) or integer-seconds/milliseconds for correct aggregation, and only convert to a packed-digit number + custom format string (or text, if aggregation doesn't matter) for **display**.
- Text-duration parsing (e.g. "1 day 6 hours 51 minutes") reuses the Ch4 text-to-table technique directly: split on spaces into a table, tag each token with a lowercased `__Key`, then look up each unit's numeric value in the row *before* its label token.

## Anti-patterns
- **Concatenating duration parts into text for display**: works for a single row but breaks totals/aggregation in visuals (blank or wrong) — use the packed-digit + custom-format-string approach instead so the value stays numeric.
- **Assuming DATEADD can shift by hours/minutes/seconds**: it can't (only DAY/MONTH/QUARTER/YEAR); build time-add manually via fractional arithmetic (`time + interval_count/24`, `/24/60`, or `/24/60/60`).
- **Computing NETWORKDAYS-style day counts without adjusting for partial start/end days**: a Monday→Wednesday range returns `NETWORKDAYS = 3`, but only Tuesday is a *full* working day — Monday and Wednesday are partial; failing to subtract those from the full-day count double-counts working time.

## Reference Tables

| Conversion | Core formula shape |
|---|---|
| Time → seconds | `(time2 - time1) * 24 * 60 * 60` |
| Decimal duration → D:H:M:S display | TRUNC/remainder cascade → packed digits → custom format `00:00:00:00` |
| Duration text ("D:H:M:S") → seconds | `SUBSTITUTE(":", "|")` → PATHITEM per segment × unit-seconds |
| Seconds → duration display | Same TRUNC/MOD cascade as decimal duration, largest→smallest unit |
| Unix seconds → UTC datetime | `DATE(1970,1,1) + unixSeconds/(60*60*24)` |
| UTC datetime → Unix seconds | `(datetime - DATE(1970,1,1)) * 60*60*24` |
| Time zone A → B | `(time - offsetA/24) + offsetB/24` |

## Worked Example
Converting a natural-language duration string like "1 Day 1 hour 51 Minutes" to total seconds, reusing Ch4's text-to-table technique:
```
Text Duration to Seconds =
    VAR __Separator = " "
    VAR __Text = MAX('Text Duration'[Text Duration])
    VAR __Count = LEN(__Text) - LEN(SUBSTITUTE(__Text, __Separator, "")) + 1
    VAR __Path = SUBSTITUTE(__Text, __Separator, "|")
    VAR __Table =
        ADDCOLUMNS(
            ADDCOLUMNS( GENERATESERIES(1, __Count, 1), "__Word", PATHITEM(__Path, [Value]) ),
            "__Key", LOWER([__Word])
        )
    VAR __DayPos = MAXX(FILTER(__Table, [__Key] = "day"), [Value]) - 1
    VAR __Days = MAXX(FILTER(__Table, [Value] = __DayPos), [__Word]) + 0
    -- (repeat __HourPos/__Hours, __MinPos/__Minutes, __SecPos/__Seconds the same way)
    VAR __Result = __Days*60*60*24 + __Hours*60*60 + __Minutes*60 + __Seconds
RETURN
    __Result
```
Because the numeric value always sits in the token *immediately before* its unit label ("1" before "Day"), each unit's value is found by locating the label's row position, subtracting 1, then looking up the word at that position — the same double-lookup shape used throughout the book, now driving a text-to-table result instead of a calendar table.

## Key Takeaways
1. Treat time as a fraction-of-day decimal (0–1) — every custom time calculation is arithmetic on that fraction.
2. Use the Chelsie Eiden packed-digit + custom-format-string technique whenever a duration must both aggregate correctly *and* display as D:H:M:S.
3. DATEDIFF is the one native function for sub-day time differences; DATEADD cannot shift by hour/minute/second — build that manually.
4. Net work duration requires explicitly correcting NETWORKDAYS' day count for partial start/end days before multiplying by a full working day's minutes.
5. Time zone conversion is just two additions/subtractions through UTC as a common intermediate.
6. Unix timestamp conversion is symmetric arithmetic anchored on `DATE(1970,1,1)`.
7. Text-duration parsing is a direct reapplication of Ch4's text-to-table + double-lookup pattern — locate each unit label's table row, then read the value one row earlier.

## Connects To
- **Ch 3 (Dates and Calendars)**: dates-as-integers and times-as-fractions are the same underlying representation; time tables mirror calendar tables.
- **Ch 4 (Text)**: text-duration parsing and the PATH-based split technique are reused verbatim.
- **Ch 2**: the Measure Totals-style "aggregate correctly, format for display" concern recurs in the Chelsie Eiden Duration technique.
- **Ch 11 (Operations)**: ticket/duration-style calculations (open tickets, delivery times) reuse Net Work Duration directly.
