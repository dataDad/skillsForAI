# Chapter 4: Text

## Core Idea
Text manipulation in DAX is a toolkit built almost entirely from `SEARCH`/`FIND` (locate), `MID`/`LEFT`/`RIGHT` (extract), and `SUBSTITUTE` (replace/count) — combined with the "text-to-table" technique (turn a string into a one-row-per-character/word table via `GENERATESERIES` + `PATHITEM`/`MID`) to unlock full DAX table operations on text.

## Frameworks Introduced
- **Text-to-Table**: convert a delimited string (or any string, char-by-char) into a DAX table so the full table toolkit (FILTER, INTERSECT, ADDCOLUMNS, CONCATENATEX...) applies to text problems.
  - How (word-level): `SUBSTITUTE(text, " ", "|")` → `PATHLENGTH`/`PATHITEM` walk the pipe-delimited path. How (char-level): `GENERATESERIES(1, LEN(text), 1)` + `ADDCOLUMNS(..., "Char", MID(text, [Value], 1))`.
  - When to use: counting/checking specific characters (vowel count via `INTERSECT` with a set), per-word transforms (capitalization), token replacement (lookup-table substitution), building anonymized/scrambled text.
- **Attribute Extraction from unstructured strings**: locate a labeled attribute ("Name: ", "Color: ") anywhere in a free-text string and pull just its value, tolerant of the attribute being absent or positioned at the string's end.
  - How: `SEARCH` for the label → if found, `SEARCH` for the next delimiter (comma) starting from that position → if no delimiter found, treat `LEN(text)+1` as the end boundary → `MID` from `label position + label length` for `(end boundary - start)` characters.
- **Counting occurrences via SUBSTITUTE length-delta**: `LEN(original) - LEN(SUBSTITUTE(original, target, ""))`, divided by `LEN(target)` when the target is more than one character (add `+1` when counting delimiters to count items instead of separators).
- **"Reasonable maximum" search-chaining**: when a pattern (e.g. a number starting with "1") could occur multiple times in a string and you don't know how many, chain `SEARCH` calls, each starting after the previous match's position, up to a practical maximum count — then `UNION` the wrapped single-value results `{ __S1 }, { __S2 }, ...` into one table and `FILTER` out blanks.

## Key Concepts
- **FIND vs. SEARCH**: identical signature (`(find_text, within_text, start, not_found_value)`); FIND is case-sensitive, SEARCH is not.
- **CONTAINSSTRING vs. CONTAINSSTRINGEXACT**: boolean containment test; the EXACT variant is case-sensitive.
- **REPLACE(text, start, num_chars, new_text)**: positional replacement (vs. SUBSTITUTE's value-based replacement).
- **SUBSTITUTE's 4th parameter**: instance number to replace, counted left-to-right; combine with a computed "total instances" count to replace counting from the right instead.
- **PATH functions** (`PATHLENGTH`, `PATHITEM`): designed for DAX hierarchy paths (pipe-delimited strings) but repurposed here as a string-splitting mechanism.
- **CONCATENATEX**: iterates a table, concatenating an expression per row with a delimiter, and (optionally) an order-by expression — the standard way to turn a table back into dynamic display text.
- **Collation / case-insensitivity**: Power BI Desktop, the Power BI service, and Azure Analysis Services all use case-insensitive collation that **cannot be changed** — duplicate strings differing only by case collapse to the casing of the first-encountered instance. (On-prem Analysis Services can change this.)
- **UNICHAR(8203)**: the zero-width space character — the standard workaround for forcing case-sensitive-looking distinctness, by injecting it adjacent to specific capital letters so two differently-cased strings no longer compare as identical.
- **UNICODE / UNICHAR**: convert a character to/from its decimal Unicode code point — used here to anonymize/scramble text.
- **USERPRINCIPALNAME() / HOUR(NOW())**: retrieve viewer identity and current hour for personalized/dynamic greetings.

## Mental Models
- Treat every "find one thing in a messy string" problem as **locate → bound → extract**: find the start position, find (or default) the end position, then `MID` between them.
- When a string might contain **zero, one, or many** matches, don't special-case — build a table of candidate matches (even blank ones), filter out the invalid/blank ones, and let table operations (COUNTROWS, MINX/MAXX) resolve the "how many, which one" logic uniformly.
- Case-insensitivity is a **storage-layer fact**, not a formula bug — if case must be preserved for display or comparison, it has to be encoded as an actual character difference (zero-width space), not assumed from casing alone.

## Anti-patterns
- **Assuming DAX preserves the casing of each individual row of duplicate-but-differently-cased text**: it doesn't — verify with `{ "The", "the", "ThE", "tHe" }` as a calculated table; all four rows display identically (the casing of whichever value was encountered first). Any case-preservation requirement needs the UNICHAR(8203) workaround, one substitution per capital letter that must be distinguished — expensive and rarely worth it except for demonstrably important cases.
- **Using LOOKUPVALUE or a single-shot SEARCH when a pattern's cardinality is unknown**: build the reasonable-maximum candidate table instead so 0/1/many all resolve through the same filter+aggregate logic rather than needing separate code paths.
- **Assuming INT/rounding-style truncation is safe for validation math**: use `ISERROR(VALUE(text))` to validate that a text fragment is purely numeric (the Phone Number Verifier's core validation trick) rather than trying to pattern-match digits directly.

## Reference Tables

| Function | Purpose |
|---|---|
| `LEFT(text, n)` / `RIGHT(text, n)` / `MID(text, start, n)` | Positional extraction |
| `LEN(text)` | Character count |
| `FIND` / `SEARCH` | Locate substring position (case-sensitive / insensitive) |
| `CONTAINSSTRING` / `CONTAINSSTRINGEXACT` | Boolean containment (insensitive / sensitive) |
| `REPLACE(text, start, n, new)` | Positional replace |
| `SUBSTITUTE(text, old, new, [instance])` | Value-based replace, optional instance number |
| `UPPER` / `LOWER` | Case conversion |
| `CONCATENATEX(table, expr, delim, order_expr)` | Table → delimited text |
| `GENERATESERIES(start, end, step)` | Numeric series table (used to drive char/word iteration) |
| `PATHLENGTH` / `PATHITEM` | Split/read a pipe-delimited path string |
| `UNICHAR(n)` / `UNICODE(char)` | Code point ↔ character |
| `ISERROR(VALUE(text))` | Test whether text is purely numeric |

Conditional-formatting color formats supported by "Field value" formatting: named colors (Windows' 141 names), 3/6/8-digit hex (8-digit adds alpha), `RGB(r,g,b)`, `RGBA(r,g,b,a)`, `HSL(h,s%,l%)`, `HSLA(h,s%,l%,a)`.

## Worked Example
Extracting a labeled attribute (`Name:`, `Color:`, `Number:`) from free-form text where the attribute may appear anywhere, in any case, or be absent:
```
Name =
    VAR __Find = "Name: "
    VAR __Pos = SEARCH( __Find, [Column1], 1, BLANK() )
    VAR __CommaPos = IF( __Pos = BLANK(), BLANK(), SEARCH( ",", [Column1], __Pos, BLANK() ) )
    VAR __CommaPos2 = IF( __CommaPos = BLANK(), LEN([Column1]) + 1, __CommaPos )
    VAR __Result =
        IF(
            __Pos = BLANK(),
            BLANK(),
            VAR __Len = LEN( __Find )
            VAR __Start = __Pos + __Len
            VAR __Result = MID( [Column1], __Start, __CommaPos2 - __Start )
            RETURN __Result
        )
RETURN
    __Result
```
Key trick: `__CommaPos2` defaults to `LEN(text) + 1` when no trailing comma exists, so the same `MID` extraction formula works whether the attribute is mid-string (bounded by a comma) or at the very end (bounded by the string's own length). The same skeleton (change only `__Find`) produces the Color and Number extractors.

## Key Takeaways
1. Master locate-bound-extract (SEARCH → determine end boundary with a fallback → MID) — it solves the majority of "pull a labeled value out of messy text" problems.
2. Text-to-table (via PATH functions or GENERATESERIES + MID) turns any string problem into a table problem, unlocking FILTER/INTERSECT/ADDCOLUMNS/CONCATENATEX.
3. Count occurrences via length-delta after SUBSTITUTE-to-blank, dividing by the target's length for multi-character targets.
4. Remember collation is case-insensitive everywhere except on-prem Analysis Services — don't rely on case being preserved for distinct-but-same-spelled values; use UNICHAR(8203) only when case genuinely must round-trip.
5. Validate numeric-looking text fragments with `ISERROR(VALUE(text))` rather than manual character-class checks.
6. When a match could occur 0, 1, or many times, build a candidate table (chained SEARCHes wrapped and UNIONed) and filter/aggregate — don't special-case cardinality.

## Connects To
- **Ch 1/Ch 2**: reuses VAR/RETURN, nested VAR/RETURN blocks, FILTER + X-aggregators, and SWITCH TRUE throughout.
- **Ch 9 (Human Resources) / Ch 13 (Advanced Scenarios)**: text-to-table and fuzzy-matching techniques recur in later scenario chapters (e.g. Complex Patterns' Fuzzy Matching in Ch 14 builds directly on this chapter's string-as-table approach).
- **Numbers (Ch 5)**: the next chapter's `ISERROR(VALUE(...))` numeric validation pattern is introduced here first (Phone Number Verifier).
