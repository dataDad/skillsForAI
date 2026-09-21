# skillsForAI

Personal collection of Claude/AI agent skills. **Private repo** — install command below works from any device with access to this repo.

## Skills

- [`dax-for-humans/`](dax-for-humans/) — DAX (Power BI) reference skill generated from *DAX For Humans* by Greg Deckler.

## Installing a skill from this repo (any device)

```bash
git clone https://github.com/dataDad/skillsForAI.git
cp -r skillsForAI/dax-for-humans ~/.claude/skills/dax-for-humans
```

Or, if using the cross-agent `skills` CLI:

```bash
npx skills add https://github.com/dataDad/skillsForAI --skill dax-for-humans
```
