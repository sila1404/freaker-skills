---
name: explainfn
description: Adds brief, complexity-scaled explanatory comments above functions — one line for simple functions, more structured detail (params/returns/etc) for complex ones, using each language's native doc-comment convention (docstrings, JSDoc, Rustdoc, etc). By default, skips functions that are already self-explanatory from their name and signature; documents everything in scope when the user explicitly asks for full coverage. Updates existing comments that no longer match the function's actual behavior instead of leaving them stale. Use when the user wants functions documented, commented, or explained, wants to add docstrings/JSDoc, or asks what a function does and wants that captured inline.
---

# explainfn

Add a comment above a function that tells a reader what it does, scaled to how much explanation the function actually needs — not a fixed template applied everywhere.

## Step 1 — Decide which functions get a comment

**Default (no explicit scope given):** skip functions that are self-explanatory from name and signature alone. A getter, a trivial wrapper, a one-line delegation — if the name and signature already say everything the body does, leave it alone. Document the rest.

**User explicitly points at scope** ("document every function in this file," "add docstrings to all of `utils.py`," "comment this specific function"): document everything in that scope, including trivial ones. An explicit ask overrides the default — don't apply the self-explanatory skip when the user asked for full coverage.

When in doubt about which mode applies, default to selective — only switch to full coverage on a clear, explicit signal.

## Step 2 — Decide how much to write

A function is **complex** if any of these hold:
- It has a branch, loop, or conditional in the body.
- It takes three or more parameters.
- Its return value isn't obvious from the name and signature (e.g. returns a transformed/computed value, a tuple, an object whose shape isn't implied by the name).

**Simple** (none of the above): one line. State what it does. Skip params/returns sections entirely — a one-liner that needs a Returns section wasn't simple.

**Complex** (any of the above): more structure, but still no padding. Use the relevant pieces only:
- One-line summary first, always.
- Params — only document a param if understanding it requires reading the function body. If the name and type (when present) already tell the caller everything, skip it — don't restate `def foo(name: str)` as "name: the name, type str." This applies with or without type hints: the test is "would a caller need to read the body to know what to pass," not "does it have a type annotation."
- Returns — only if the return value needs explanation beyond its type.
- Raises/Throws — only if the function deliberately raises something a caller needs to handle, not every theoretical exception.
- Example — only if the calling convention is genuinely non-obvious (e.g. unusual argument order, a callback shape, required setup). Most functions don't need one.
- **Unused parameter** — if a parameter is declared but never referenced anywhere in the function body, don't silently document it as if it does something, and don't silently skip mentioning it either. Flag it explicitly (e.g. a short "Note: `x` is unused in the current implementation" line) so the discrepancy is visible rather than papered over. This is a documentation task, not a refactor — don't remove the parameter or otherwise change the signature, just surface what's actually true.

A complex function's comment should still be scannable in a few seconds. If a section would just restate the signature, cut the section.

## Step 3 — Use the language's native convention

Detect the language from the file extension and write in its standard doc-comment format, not a generic `//` block:

| Language | Convention | Marker |
|---|---|---|
| Python | Docstring (Google or NumPy style — match what the file/project already uses; default to Google if neither is established) | `"""..."""` directly under the `def` line |
| JavaScript / TypeScript | JSDoc | `/** ... */` above the function, `@param`/`@returns` tags |
| Java | Javadoc | `/** ... */` above the method, `@param`/`@return`/`@throws` |
| C# | XML doc comments | `/// <summary>`, `/// <param>`, `/// <returns>` |
| Go | Go doc comment | Plain `//` directly above, starting with the function name |
| Rust | Rustdoc | `///` above the function, `# Examples` section if used |
| C / C++ | Doxygen-style | `/** ... */` with `@brief`/`@param`/`@return`, or plain `//` for simple one-liners if the project doesn't use Doxygen |
| Ruby | YARD | `# ...` above the method, `@param`/`@return` tags |

If a project already has an established docstring style that differs from the table (e.g. NumPy-style Python, or a custom JSDoc tag set), match what's already there instead of the default — consistency with the existing codebase wins over the table.

If the language isn't listed, use that language's most common idiomatic doc-comment style; if genuinely unclear, ask rather than guess at an unfamiliar convention.

## Step 4 — Handle existing comments

If a function already has a comment or docstring:

- **Still accurate** — leave it alone. Don't rewrite working documentation just to restyle it, unless the user asked for a format change.
- **Outdated or wrong** (describes behavior the function no longer has, references removed params, wrong return type, etc.) — update it in place to match the function's actual current behavior. Don't leave stale documentation standing, and don't just delete it without replacing it.
- **Present but missing pieces** (e.g. has a summary but no Params section on a function that's since grown a third parameter) — fill in what's missing, keep what's accurate.

## What this skill does not do

- Doesn't change function logic, signatures, or behavior — comments only.
- Doesn't write module-level docs, README content, or architecture notes — see a project-overview-style skill for that; this is strictly per-function.
- Doesn't add comments inside a function body explaining individual lines — that's a different kind of comment (explaining *how*); this skill writes the one comment above the function explaining *what it does* and, when complex, *what it expects and returns*.
