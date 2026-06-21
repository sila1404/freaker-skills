# tersedev

A lazy-senior-developer engineering persona for AI coding agents: minimal code, terse communication, and a real test-driven workflow for anything beyond a trivial fix. One dial (`lite | full | ultra`) controls all three at once.

See [SKILL.md](./SKILL.md) for the actual behavior definition — that's the file an agent reads.

## What it does, briefly

- **Code**: climbs a ladder (skip it → stdlib → native feature → existing dependency → one-liner → minimal code) before writing anything, marking deliberate shortcuts with a `// tersedev:` comment naming the ceiling and the upgrade path.
- **Process**: auto-runs a test-driven workflow (one test → minimal code → repeat, vertical slices only) for non-trivial work, while trivial fixes skip straight to a single runnable check.
- **Prose**: drops articles, filler, and hedging from conversational text, with an automatic override back to full sentences for security warnings, irreversible actions, and anything genuinely ambiguous when compressed.

## How it was made

This skill is a deliberate merge of three separate, pre-existing public skills, combined into one self-contained skill so they'd work together instead of as three things stacked on top of each other. Credit where it's due:

- **[Ponytail](https://github.com/DietrichGebert/ponytail)** (by DietrichGebert) — the lazy-senior-developer / YAGNI-ladder / minimal-code philosophy. The "climb the ladder, mark shortcuts with a comment, code-first output" rules in `tersedev` originate here.
- **[Caveman](https://github.com/JuliusBrussee/caveman)** (by JuliusBrussee) — the terse, fragment-friendly prose style with an auto-clarity override for safety-critical or ambiguous moments. The "drop articles/filler/hedging, full sentences for warnings" rules in `tersedev` originate here.
- **[Test-Driven Development](https://github.com/mattpocock/skills/tree/main/skills/engineering/tdd)** — the `tdd` skill from [mattpocock/skills](https://github.com/mattpocock/skills) (by Matt Pocock). The red/green/refactor workflow, the ban on writing all tests before any implementation ("horizontal slicing"), and the public-interface-only testing philosophy all come from there. The entire [Test-Driven Workflow](./SKILL.md#test-driven-workflow) section in `tersedev` is built on it.

`tersedev` doesn't reference these by name in its own SKILL.md — each rule was rewritten to stand on its own so the skill works for anyone who installs it, without requiring them to know its history. This README is where that history lives instead.

The merge design (when TDD auto-fires vs. when the lighter single-check rule applies, how the planning step compresses to one line, how the three original intensity dials collapse into one) was worked out interactively, then verified against a series of test prompts — including a trivial-vs-non-trivial boundary case, a level switch mid-task, and a "stop" command mid-test-cycle — before being finalized.

Packaged and validated using [Anthropic's `skill-creator`](https://github.com/anthropics/skills) (`quick_validate.py`, `package_skill.py`).

## Installing

```bash
npx skills add sila1404/freaker-skills --skill tersedev
```

## Usage

```
/tersedev full        # activate (default level)
/tersedev lite|ultra   # switch level
stop tersedev          # deactivate, revert to normal mode
```

## License

MIT, same as the rest of [freaker-skills](../). [Ponytail](https://github.com/DietrichGebert/ponytail), [Caveman](https://github.com/JuliusBrussee/caveman), and [mattpocock/skills](https://github.com/mattpocock/skills) (the `tdd` source) retain whatever license their own repos specify; check each upstream repo directly if redistributing this skill matters for your use case.
