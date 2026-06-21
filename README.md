# freaker-skills

A personal collection of [Agent Skills](https://github.com/vercel-labs/skills) — reusable instruction sets that extend what an AI agent does, installable with `npx skills`.

## Why this exists

Honestly, no big reason. I had some free time, was kinda bored, had nothing better to do, and figured I'd mess around with making my own agent skill instead of just scrolling my phone. Started with one idea, got curious how far I could take it, and ended up with this. Nothing fancy, just something fun to build and tinker with when I've got spare time.

## What's in here

| Skill | What it does |
|---|---|
| [`tersedev`](./tersedev) | Lazy-senior-developer persona — minimal code, terse prose, test-driven workflow for non-trivial work. See its own README for details and credits. |
| [`grillmap`](./grillmap) | Three-phase skill: relentless interview to sharpen a plan or project (writing `CONTEXT.md`/`docs/adr/` along the way), a lightweight knowledge graph built from what the interview surfaced, then a human-readable overview synthesizing both. See its own README for details and credits. |

More skills may be added here over time as I build out other personal-workflow behaviors — this repo is just my personal shelf for these, not a themed collection.

## Installing a skill from here

```bash
npx skills add sila1404/freaker-skills
```

Use `--list` to see everything currently in the repo:

```bash
npx skills add sila1404/freaker-skills --list
```

## License

MIT — see [LICENSE](./LICENSE). Individual skills may credit additional sources in their own README; that credit stands regardless of this repo's license.
