# dax-for-humans (Claude skill)

Agent skill generated from *DAX For Humans* by Greg Deckler with [book-to-skill](https://github.com/virgiliojr94/book-to-skill).

This is a **synthesized reference** — extracted frameworks, patterns, and summaries in my own words, not the book's text. It covers the book's central thesis (the "No CALCULATE" approach to writing debuggable DAX) plus chapter-by-chapter techniques for date intelligence, text, numbers, time, and domain KPIs (customers, HR, projects, finance, operations, distance/space), advanced scenarios, complex patterns, performance optimization, and AI-assisted DAX authoring/debugging.

**Private/personal use only** — derived from a purchased, copyrighted book. Not for redistribution.

## Install

```bash
git clone https://github.com/dataDad/skillsForAI.git
cp -r skillsForAI/dax-for-humans ~/.claude/skills/dax-for-humans
```

## Files

- `SKILL.md` — core frameworks (No CALCULATE pattern, offsets, double-lookup, debugging toolkit) + chapter/topic index (loads by default, ~4K tokens)
- `chapters/ch01–ch16` — one file per chapter, loaded on demand
- `glossary.md` — ~90 key terms with chapter references
- `patterns.md` — ~25 named techniques (When to use / How / Trade-offs)
- `cheatsheet.md` — decision rules, quick-reference tables, formula skeletons

Total size: ~28K words, but only `SKILL.md` loads by default — chapter files load only when a query needs that depth.
