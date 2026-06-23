# api-docs

Generate, write, and keep API documentation in sync — covering REST, gRPC, GraphQL, or plain RPC endpoints in any language or web framework. Produces both human-readable Markdown reference docs and an OpenAPI 3.1 YAML spec.

See [SKILL.md](./SKILL.md) for the actual behavior definition — that's the file an agent reads.

## What it does, briefly

- **Generate from code**: detects the API style and framework, reads the real contract the code exposes, then writes Markdown + OpenAPI YAML together so they never drift.
- **Keep docs in sync**: diffs the current code contract against existing docs, updates only what changed, and flags breaking changes.
- **Write docs first**: when there's no code yet, writes the OpenAPI YAML as the spec to build against, then derives Markdown from it.

Reference templates live in [`references/`](./references/):

- `framework_patterns.md` — route-registration and validation idioms across languages/frameworks
- `openapi_template.md` — structure and conventions for the OpenAPI output
- `markdown_template.md` — structure and conventions for the Markdown output
- `proto_extraction.md` — how to read `.proto` files for gRPC docs

## Installing

```bash
npx skills add sila1404/freaker-skills --skill api-docs
```

## Usage

```
/api-docs                              # generate or sync API docs for the current project
/api-docs document this endpoint       # focus on one endpoint
/api-docs check if the docs are still accurate
```

Output goes to `docs/api.md` and `docs/openapi.yaml` by default, unless the project already keeps docs elsewhere.

## License

MIT, same as the rest of [freaker-skills](../).
