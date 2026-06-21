---
name: grillmap
description: Three-phase skill for understanding and documenting a project — a relentless one-question-at-a-time interview that sharpens vague language and captures decisions (writing CONTEXT.md and docs/adr/ inline), then a lightweight knowledge graph built from what the interview surfaced (docs/GRAPH.md, Mermaid), then a human-readable handoff overview synthesizing both (docs/OVERVIEW.md). Runs as one continuous flow from a single invocation. Use when the user wants to stress-test a plan or get properly oriented in an existing project, and end up with documentation + a structural map + a readable handoff doc, not just one of the three.
---

# grillmap

Three phases, one flow: grill the plan into precision, map what got resolved into a graph, then write up the whole thing as something a new person could actually read. Triggered by a single invocation; the three phases run back to back without the user needing to advance them manually.

## Phase 1 — Grill

Interview the user relentlessly about every aspect of the plan or project until shared understanding is reached. Walk each branch of the design tree, resolving dependencies between decisions one at a time. For every question, give a recommended answer alongside it.

Ask one question at a time. Wait for feedback before asking the next. If a question can be answered by exploring the codebase, explore the codebase instead of asking.

### File layout

Most projects have a single context:

```
/
├── CONTEXT.md
├── docs/
│   ├── adr/
│   │   ├── 0001-event-sourced-orders.md
│   │   └── 0002-postgres-for-write-model.md
│   ├── GRAPH.md       (written in Phase 2)
│   └── OVERVIEW.md     (written in Phase 3)
└── src/
```

If `CONTEXT-MAP.md` exists at the root, the project has multiple contexts, each with its own `CONTEXT.md` and `docs/adr/`. The map points to where each one lives. In a multi-context project, run Phase 1 against the context currently being discussed. Phases 2 and 3 are scoped to that same context too, including file location: `docs/GRAPH.md` and `docs/OVERVIEW.md` go inside that context's own folder (e.g. `src/billing/docs/GRAPH.md`), next to its `CONTEXT.md` and `docs/adr/` — not at the project root. A decision that touches more than one context (e.g. a boundary between ordering and billing) gets noted in the current context's files; don't write into another context's `CONTEXT.md` in the same session unless the user explicitly redirects there.

Create files lazily — only when there's something to write. No `CONTEXT.md`, `docs/adr/`, `docs/GRAPH.md`, or `docs/OVERVIEW.md` exists until the first thing that belongs in it actually shows up.

### During the interview

**Challenge against the glossary.** When a term conflicts with what's already in `CONTEXT.md`, call it out immediately: "Your glossary defines 'cancellation' as X, but you seem to mean Y — which is it?"

**Sharpen fuzzy language.** When a term is vague or overloaded, propose a precise canonical one: "You're saying 'account' — do you mean the Customer or the User? Those are different things."

**Stress-test with scenarios.** When a domain relationship comes up, invent concrete edge-case scenarios that force precision about the boundary between concepts.

**Cross-reference with the code.** When the user states how something works, check whether the code agrees. Surface contradictions directly: "Your code cancels entire Orders, but you just said partial cancellation is possible — which is right?"

**Update `CONTEXT.md` inline.** The moment a term resolves, write it. Don't batch updates. Keep `CONTEXT.md` free of implementation detail — only terms meaningful to someone who understands the domain, not the code.

**Offer ADRs sparingly.** Only propose writing an ADR when all three hold:
1. Hard to reverse — changing course later would cost something real.
2. Surprising without context — a future reader would wonder why this was done this way.
3. The result of a real trade-off — genuine alternatives existed and one was picked for specific reasons.

If any of the three is missing, don't write one.

### Tracking progress

Keep a running **open questions** list throughout Phase 1 — not just a mental note, an actual list maintained as the interview goes. Three kinds of answer go here instead of (or alongside) `CONTEXT.md`:
- **Tentative answers** — the user is unsure, hedges, or says something like "probably X, but never really came up." Don't write these to `CONTEXT.md` as settled fact. Park them in the open-questions list instead.
- **Known gaps** — a contradiction between stated intent and what the code does, where the user confirms it's a gap rather than a deliberate decision (e.g. "we need partial cancellation but the code doesn't support it yet"). These don't get an ADR (no decision was made), but they're too important to drop — park them here.
- **Explicitly deferred topics** — the user says "skip that for now" or similar.

This list is what Phase 3's "open questions or rough edges" section is built from — Phase 3 doesn't re-derive this by re-reading the conversation, it reads the list Phase 1 already kept.

### When to stop and move on

Keep interviewing until shared understanding is reached — don't stop on a fixed question count or an idle streak. The interview ends when the user says so (e.g. "that covers it," "let's move on," "stop here") or there's genuinely nothing left worth asking because every branch of the design tree has been walked. When the user signals they're done, don't ask a separate "ready to move to the graph step?" confirmation — treat their closing signal as the cue and state the transition in one line ("Grilling settled — building the graph now.").

## Phase 2 — Graph

Build a lightweight knowledge graph from what Phase 1 already surfaced. Don't re-explore the codebase from scratch — reuse what grilling already found.

**Nodes:** every resolved term in `CONTEXT.md`, plus every ADR (referenced by its decision, not its file number).

**Edges:** relationships stated or implied during the interview (e.g. "an Order has many LineItems," "Refund supersedes the original Payment"), plus real structural dependencies actually seen while exploring the codebase in Phase 1 (e.g. a module import, a foreign key, a function call chain) — not invented or guessed at if they weren't actually observed.

Render this as a Mermaid diagram and write it to `docs/GRAPH.md`, with a short legend underneath explaining what each node and edge type means. This file is for the agent (and anyone else) to consult later instead of re-deriving the project's shape from scratch — it should stand on its own without the interview transcript.

If the project is small enough that the graph would only have a handful of nodes, build it anyway — a small graph that's actually correct beats skipping the step. Only skip this phase if Phase 1 resolved nothing at all (e.g. the user stopped immediately, no `CONTEXT.md` entries and no ADRs exist): say so in one line, and skip Phase 3 as well — there's nothing to synthesize. Stop there rather than producing an empty or near-empty overview document.

## Phase 3 — Overview

Read `CONTEXT.md`, every ADR, and `docs/GRAPH.md`. Synthesize all three into one narrative document for a human who's never seen this project before and needs to get oriented fast.

Write `docs/OVERVIEW.md` containing:
- **What this project is** — a few sentences, plain language, no jargon that wasn't just defined.
- **Core vocabulary** — the terms from `CONTEXT.md` that matter most, introduced in an order that builds understanding (foundational concepts before the ones that depend on them), not just alphabetical or insertion order.
- **Key decisions and why** — the ADRs, summarized in a sentence or two each, in plain language, with a link back to the full ADR file for anyone who wants the detail.
- **How the pieces connect** — a short walkthrough of the graph from Phase 2, in prose: what depends on what, what the high-traffic nodes are, what's isolated.
- **Open questions or rough edges** — pulled from the open-questions list kept during Phase 1 (tentative answers, known gaps, deferred topics), plus anything the graph revealed as orphaned (a term with no relationships, an ADR that references something never defined).

This is prose for a person, not a data dump. It should be the one document someone reads before opening any code.

## If documentation already exists

If `CONTEXT.md` or `docs/adr/` already has content when grillmap is invoked, ask once whether to run the full interview against the existing material (treating it as the starting point to challenge and extend) or skip straight to rebuilding the graph and overview from what's already there. Don't assume either way.

## Boundaries

This skill writes documentation; it doesn't write or modify application code. If a question raised during Phase 1 reveals a bug or a contradiction worth fixing in code, surface it and ask whether to address it now or note it as an open question in Phase 3 — don't fix it unprompted mid-interview.
