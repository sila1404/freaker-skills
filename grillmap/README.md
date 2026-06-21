# grillmap

A three-phase skill: interview a plan or project into precision, build a lightweight knowledge graph from what the interview surfaced, then write up the whole thing as a handoff document a new person could actually read cold.

See [SKILL.md](./SKILL.md) for the actual behavior definition — that's the file an agent reads.

## What it does, briefly

**Phase 1 — Grill:** one question at a time, recommended answer attached, explores the codebase before asking anything answerable there. Challenges fuzzy language, cross-references claims against code, writes `CONTEXT.md` and `docs/adr/` inline as things resolve. Keeps going until you say it's covered, not on a fixed count.

**Phase 2 — Graph:** turns the resolved glossary and ADRs into nodes, turns stated relationships and real code dependencies into edges, renders it as Mermaid, saves it to `docs/GRAPH.md` so the agent has a durable map instead of re-deriving the project's shape every session.

**Phase 3 — Overview:** reads `CONTEXT.md`, every ADR, and the graph, then writes one narrative document — `docs/OVERVIEW.md` — covering what the project is, its core vocabulary in a sensible order, key decisions and why, how the pieces connect, and anything left open.

All three run from one invocation, back to back, no manual advancing between phases.

## How it relates to other skills

- **[grill-with-docs](https://github.com/mattpocock/skills/tree/main/skills/engineering/grill-with-docs)** (mattpocock/skills) is the direct source for Phase 1 — the one-question-at-a-time interview discipline, the glossary-challenging, the three-condition bar for writing an ADR, the `CONTEXT.md`/`docs/adr/` file layout. Phase 1 here is a close, faithful reimplementation of that skill's approach, not a wrapper around it — `grillmap` doesn't call or depend on `grill-with-docs` itself.
- **[Graphify](https://github.com/safishamsi/graphify)** (by Safi Shamsi) is the inspiration for Phase 2's existence, not its mechanism. The real Graphify is a full standalone tool — Tree-sitter parsing, NetworkX, Leiden clustering, a separate Python package you install and run. `grillmap`'s graph step borrows only the idea (a structural map of the project that an agent can consult instead of re-reading everything) and reimplements it as plain prompt instructions: build a small Mermaid diagram from what Phase 1 already learned, no parsing, no external tool, no install step.
- **[handoff](https://github.com/mattpocock/skills)** (mattpocock/skills) compacts a *conversation* so another agent session can continue it. `grillmap` documents a *project*, for a *person* to read. Different input, different audience, not a substitute for one another.

## Installing

```bash
npx skills add sila1404/freaker-skills --skill grillmap
```

## Usage

```
/grillmap                  # or: "grillmap this project" / "grillmap this plan"
```

If `CONTEXT.md` or `docs/adr/` already exist, it'll ask once whether to run the interview against the existing material or just rebuild the graph and overview from what's already there.

## License

MIT, same as the rest of [freaker-skills](../). [grill-with-docs](https://github.com/mattpocock/skills) and [Graphify](https://github.com/safishamsi/graphify) retain whatever license their own repos specify.
