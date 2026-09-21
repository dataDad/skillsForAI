# Chapter 12: Distance and Space

## Core Idea
DAX has no spatial function library — no ATAN2, no distance/bearing/geodesic support — so this chapter builds a small reusable geometry toolkit (a hand-rolled ATAN2, Haversine distance, bearing) from core trig functions, then applies similar "build the missing primitive" thinking to non-spatial-but-structurally-similar problems: transitive closure (graph reachability) and 3D bin-packing (box-size matching).

## Frameworks Introduced
- **Hand-rolled ATAN2 via quadrant SWITCH**: DAX's `ATAN` only covers -90°to90° (two quadrants); reconstruct full four-quadrant `ATAN2(y,x)` with a `SWITCH(TRUE(), ...)` covering the sign combinations of x and y (positive x; negative x with y≥0; negative x with y<0; x=0 with y>0; x=0 with y<0), each branch using the right ATAN-plus-or-minus-PI adjustment. This single reusable formula block underlies every subsequent spatial calculation in the chapter (polar-coordinate theta, Haversine distance's central angle, and bearing).
- **Haversine great-circle distance**: convert lat/long degrees to radians, compute the haversine of the angular difference, then `distance = EarthRadius × 2×ATAN2(√a, √(1-a))` — the standard formula for real-world distance between two lat/long points, expressed entirely in DAX primitives plus the hand-rolled ATAN2.
- **Bearing (direction of travel)**: a second application of the same hand-rolled ATAN2, computed from `y = cos(lat2)×sin(Δlong)` and `x = cos(lat1)×sin(lat2) − sin(lat1)×cos(lat2)×cos(Δlong)`; convert "relative" bearing to "true" bearing via `MOD(bearing + 360, 360)`.
- **Dimension-order-normalization for matching (Box Sizes)**: when comparing two sets of dimensions (item vs. box) that may not share the same length/width/height *labeling* convention, sort each set into `__Measure1 ≤ __Measure2 ≤ __Measure3` (smallest/middle/largest) before comparing — this makes "does the item fit in the box" a simple three-way `≤` comparison regardless of how either was originally labeled.
- **Bounded transitive closure**: since DAX table operations don't recurse arbitrarily, compute graph reachability (`From→To` chains) by repeating the same "find all `To`s reachable from the current `To` set" ADDCOLUMNS/FILTER/UNION step **once per desired hop depth** — a 2-hop closure needs the block applied twice, a 3-hop closure three times, and so on; there's no way to compute an unbounded-depth closure without deciding a maximum hop count in advance.

## Key Concepts
- **Polar coordinates (r, θ)**: `r = √(x²+y²)` (hypotenuse); `θ = ATAN2(y,x)` in degrees via `DEGREES()`.
- **RADIANS / DEGREES**: required conversions since DAX trig functions operate in radians but geographic/navigation data is usually in degrees.
- **Eastings/Northings → Lat/Long**: a large, verbatim-ported closed-form geodetic transformation (Ordnance Survey grid formulas) — included as an example of DAX handling genuinely heavy scientific-computing formulas when needed, not something to derive from first principles.
- **"Near" — box vs. radius**: a bounding-box near-test (`|Δx| ≤ radius && |Δy| ≤ radius`) is cheaper but overinclusive (catches corner points a box-diagonal away); a true radius test adds a `Distance = SQRT((x1-x)²+(y1-y)²)` column and filters `Distance ≤ radius` — pick box for speed, radius for correctness, based on the use case.
- **Transitive closure**: the graph-reachability concept — "what can be reached from node X by following any number of direct edges" — computed here via repeated UNION/FILTER passes (see Frameworks) rather than a native recursive function, since DAX lacks one.
- **Box Sizes / bin-fit matching**: `MINX`/`MAXX`/`EXCEPT` over a 3-element inline table `{h, w, l}` is the trick for sorting exactly three values without a general sort function — `__Measure1 = MIN`, `__Measure3 = MAX`, `__Measure2 = the one EXCEPT'd out of the other two`.

## Mental Models
- **Any time a "missing" DAX function is well-defined mathematically (ATAN2, MODE from Ch5, distance formulas), it's usually cheaper to hand-roll it once as a reusable SWITCH/VAR block than to work around its absence in every downstream formula** — this chapter treats ATAN2 as a one-time investment reused three times (polar, distance, bearing).
- **Matching or comparing multi-dimensional data (box dimensions) is unreliable until you normalize to a canonical order** — sort each entity's dimensions (smallest→largest) before comparing, exactly as you would before comparing unordered sets.
- **DAX table recursion is bounded, not arbitrary**: recognize "how many hops/levels do I need?" up front for any transitive/graph-reachability problem, then unroll that many repetitions of the same expansion step — there's no way to ask for "however many hops it takes" generically.

## Anti-patterns
- **Reaching for ATAN alone on spatial/navigational data**: silently wrong outside two quadrants (-90° to 90°) — always use the full ATAN2 SWITCH block for anything involving direction/bearing/angle-between-points.
- **Using a bounding-box "near" test when true radial proximity is required**: overincludes points that are diagonally farther than the radius (a box's corners extend √2× farther than its half-width) — use the radius+SQRT distance test when correctness matters more than performance.
- **Assuming a fixed-depth transitive closure formula generalizes to arbitrary graph depth**: it doesn't — a 2-hop formula misses nodes 3+ hops away; explicitly decide and hardcode the maximum depth needed, or accept the closure is partial.

## Reference Tables

| Calculation | Formula shape |
|---|---|
| ATAN2(y,x) | quadrant-aware SWITCH(TRUE(), ...) over sign(x), sign(y) |
| Polar r | `SQRT(x²+y²)` |
| Polar θ | `DEGREES(ATAN2(y,x))` |
| Haversine distance | `EarthRadius × 2×ATAN2(√a, √(1-a))`, a = haversine of Δlat/Δlong |
| Bearing | `DEGREES(ATAN2(y,x))` with y,x from lat/long cross terms; true = `MOD(bearing+360, 360)` |
| Near (box) | `|Δx|≤r && |Δy|≤r` |
| Near (radius) | box test + `SQRT(Δx²+Δy²) ≤ r` |
| Transitive closure (N hops) | repeat UNION/FILTER expansion step N times |
| Box fit | normalize both item and box dims to (min,mid,max), compare all three `≤` |

## Worked Example
The reusable hand-rolled ATAN2 block, the chapter's foundational technique reused in polar coordinates, Haversine distance, and bearing:
```
ATAN2 =
    VAR __x = <x value>
    VAR __y = <y value>
    VAR __Result =
        SWITCH( TRUE(),
            __x > 0, ATAN(__y/__x),
            __x < 0 && __y >= 0, ATAN(__y/__x) + PI(),
            __x < 0 && __y < 0, ATAN(__y/__x) - PI(),
            __x = 0 && __y > 0, PI()/2,
            __x = 0 && __y < 0, PI()/2 * -1,
            BLANK()
        )
RETURN
    __Result
```
Every subsequent spatial formula in the chapter (`theta`, Haversine's central angle `c`, bearing) is this exact SWITCH block with different `__x`/`__y` inputs substituted in — recognizing that pattern means you only need to learn it once.

## Key Takeaways
1. Hand-roll ATAN2 once via a quadrant-aware SWITCH; reuse it for every subsequent angle/bearing/distance calculation rather than re-deriving quadrant logic each time.
2. Haversine distance and bearing are direct formula translations into DAX, requiring RADIANS/DEGREES conversions at the boundaries.
3. Normalize multi-dimensional values (sort smallest→largest) before comparing across differently-labeled dimension sets — the Box Sizes fit-test trick.
4. DAX transitive closure requires manually unrolling one expansion pass per hop depth — decide the required depth up front.
5. Choose bounding-box vs. true-radius "near" tests deliberately based on whether overinclusion at the corners is acceptable.
6. Not every problem in this chapter is the No CALCULATE virtual-table pattern — but none of them need CALCULATE either; the book's real point is that CALCULATE is never required, not that every problem looks identical.

## Connects To
- **Ch 5 (Numbers)**: trig functions (SIN/COS/TAN, RADIANS/DEGREES) introduced there are the building blocks used throughout this chapter.
- **Ch 4 (Text)**: EXCEPT/inline-table `{ }` tricks used for the Box Sizes dimension-sorting reuse Ch4/Ch5's small-set manipulation techniques.
- **Ch 11 (Operations)**: transitive closure's repeated-expansion technique is conceptually the same "unroll a bounded loop" idea as Ch11's while-loop emulation, applied to graph traversal instead of accumulation.
- **Ch 14 (Complex Patterns)**: DAX INDEX and other advanced techniques later in the book continue this chapter's spirit of building missing primitives from scratch.
