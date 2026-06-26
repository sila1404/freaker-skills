# explainfn

Adds a brief comment above a function explaining what it does — scaled to how complex the function actually is, written in whatever doc-comment convention the language natively uses.

See [SKILL.md](./SKILL.md) for the actual behavior definition — that's the file an agent reads.

## What it does, briefly

**Decides what gets a comment.** By default, skips functions that are already self-explanatory from their name and signature — a one-line getter doesn't need a docstring. If you explicitly ask for full coverage ("document every function in this file"), that overrides the default and everything in scope gets a comment, trivial or not.

**Scales the length to the function, not a template.** A function is treated as complex if it has a branch/loop/conditional, takes three or more parameters, or has a return value that isn't obvious from its name. Simple functions get one line. Complex functions get a short structured comment — but only the sections that actually add information; a parameter doesn't get explained if its name already tells a caller everything, and an unused parameter gets flagged rather than silently documented as if it does something.

**Writes in the language's native format.** Python gets a docstring, JS/TS gets JSDoc, Rust gets Rustdoc, and so on — not a generic `//` block dropped into every language. Matches an existing project convention (e.g. NumPy-style over Google-style) when one is already established.

**Fixes stale comments instead of leaving them.** If a function already has a comment that no longer matches what the code does, it gets updated in place rather than left wrong or just deleted.

## Installing

```bash
npx skills add sila1404/freaker-skills --skill explainfn
```

## Usage

```
/explainfn document the functions in this file          # selective: skips self-evident functions
/explainfn add docstrings to every function in utils.py # explicit: documents everything in scope
/explainfn comment this function                        # documents just the one pointed at
```

## License

MIT, same as the rest of [freaker-skills](../).
