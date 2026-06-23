---
name: api-docs
description: Generate, write, and keep API documentation in sync — covering REST/gRPC/GraphQL/RPC endpoints, request/response shapes, error codes, and auth, in any programming language or web framework. Produces both human-readable Markdown and an OpenAPI 3.1 YAML spec. Use this skill whenever the user asks to document an API, write API docs, create an OpenAPI/Swagger spec, generate endpoint reference docs, update docs after changing a route/handler, or review whether docs match the actual code — regardless of what language or framework the codebase uses. Trigger this even if the user just says "document this endpoint" or "add docs for the new route" without mentioning OpenAPI or Markdown explicitly — those are the two outputs this skill always considers.
---

# API Documentation

Produces and maintains API documentation from code in **any language and any web framework** — Go, Python, Node/TypeScript, Rust, Java, Ruby, PHP, C#, or anything else — across **any API style**: REST, gRPC, GraphQL, or plain RPC. Two synchronized outputs:

1. **Markdown** — human-readable reference docs (`references/markdown_template.md` for the structure)
2. **OpenAPI 3.1 YAML** — machine-readable spec (`references/openapi_template.md` for the structure)

Both outputs describe the same API — never let them drift. If you update one, update the other in the same pass.

This skill works by **reading the actual contract the code exposes**, not by pattern-matching one specific framework's syntax. The detection step below is deliberately framework-agnostic: identify the API style first, then the framework, then read its idioms — using `references/framework_patterns.md` as a lookup table, not a hardcoded list to match against. If a framework isn't in that file, apply the same underlying logic (find route registration → find handler → find request/response shape → find error branches) and add what you learn to the reference file for next time.

## Read existing project context first

Before touching code or writing anything, skim whatever the project already says about itself — **README, `CONTEXT.md`, `docs/`, `CONTRIBUTING.md`, ADRs, a `CHANGELOG`, inline package-level doc comments.** `CONTEXT.md` specifically has become a common convention for project/business context meant for AI agents and new contributors alike — always check for it explicitly, don't rely on stumbling onto it. This isn't optional busywork; it's the fastest way to avoid writing technically-correct-but-misleading docs.

What to look for specifically:
- **What the API/project is for** — a one-line description here saves you from inferring intent purely from route names, which is often ambiguous (is `/cancel` a soft-delete or a state transition? the README usually says).
- **Domain vocabulary and naming conventions already in use** — if the project calls something a "session" everywhere in its docs, don't rename it to "challenge" in your output just because the variable name in code is `challenge`. Match the project's existing vocabulary so the docs feel native, not bolted on.
- **Business rules that aren't obvious from code alone** — a comment or README line explaining *why* a behavior exists (e.g., "todos can be reopened but not edited once completed, to preserve audit history") is worth lifting into the docs verbatim-in-spirit, since code alone shows *what* happens, not *why* — and the "why" is often exactly what an API consumer needs to use the endpoint correctly rather than by trial and error.
- **Existing terminology for errors/statuses** — if docs already describe an error as "the session is locked," reuse that phrase rather than inventing a new one like "the session is frozen."
- **An existing doc structure or style to match** — if the project already has other Markdown docs, skim one to match heading style, tone, and level of formality, so the new API docs don't look like they came from a different project.

**If project docs and code disagree** (stale doc describing removed behavior, or a comment that no longer matches the implementation): trust the code for what the API *does*, but flag the discrepancy to the user rather than silently picking one — a stale doc is itself useful signal that something changed without the docs being updated, which is exactly the kind of drift this skill exists to catch.

Don't over-invest here — this is a fast skim for context, not a full audit. A quick check like `ls README* CONTEXT* CHANGELOG* CONTRIBUTING* 2>/dev/null` and a peek into `docs/` covers most projects in seconds. If the project has no docs at all, that's not a problem — just proceed straight to reading the code.

## Decide which mode you're in

This skill handles three distinct situations. Identify which one applies before doing anything else code-related (you've already skimmed project context above regardless of mode):

| Situation | Signal | What to do |
|---|---|---|
| **Generate from scratch** | No existing docs, or user says "document this API" | Go to [Generating from code](#generating-from-code) |
| **Sync after a code change** | Docs already exist, code just changed (new endpoint, changed field, etc.) | Go to [Keeping docs in sync](#keeping-docs-in-sync) |
| **Write from scratch (no code yet)** | User is designing an API before implementing it | Go to [Writing docs first (design mode)](#writing-docs-first-design-mode) |

If unclear, check whether docs already exist in the repo (`grep -ril "openapi\|swagger" .` or look for `docs/api.md`) before asking the user.

## Generating from code

1. **Identify the API style first** — this determines where the contract lives:
   - **gRPC**: look for `.proto` files anywhere in the repo. If found, the `.proto` is the contract — read it directly rather than reverse-engineering from generated stubs or handler code in any language.
   - **GraphQL**: look for `.graphql`/`.gql` schema files, or a `schema.graphql`, or in-code schema builders (e.g., `gql\`...\`` template literals, `@ObjectType` decorators). The schema definition is the contract.
   - **REST/HTTP**: no schema file — the contract lives in route registration + handler code. This is the common case and needs framework detection (next step).
   - **OpenAPI/Swagger already exists**: if `openapi.yaml`, `swagger.json`, or similar is already present and current, treat it as a strong source of truth to cross-check against, not just code.

   Multiple styles can coexist in one repo (e.g., a public REST API in front of internal gRPC services, like a BFF) — document each style with the conventions that fit it, and note the relationship between them.

2. **For REST/HTTP, detect the framework before searching for routes.** Don't assume a syntax — check what's actually imported/required first:
   - Check the dependency manifest: `go.mod`, `package.json`, `requirements.txt`/`pyproject.toml`, `Cargo.toml`, `pom.xml`/`build.gradle`, `Gemfile`, `composer.json`, `*.csproj`, etc.
   - Match the framework name found there against `references/framework_patterns.md`, which has route-registration syntax for common frameworks per language (Gin/Echo/net-http for Go; Express/Fastify/NestJS for Node; Flask/FastAPI/Django for Python; Actix/Axum for Rust; Spring for Java; Rails for Ruby; and others).
   - **If the framework isn't listed there**, don't give up — apply the general pattern instead: search for whatever construct registers a URL path to a function (often named `route`, `handle`, `Router`, `@app.<verb>`, `#[<verb>(...)]`, or similar), then follow that to the function body it points to. Every web framework has *some* mechanism for this; the names vary but the structure doesn't.

3. **For each endpoint found, extract:**
   - Method + path (HTTP/REST), or RPC name (gRPC), or query/mutation name (GraphQL)
   - Request fields: name, type, required/optional, and validation rules. Validation idioms vary by language/framework (Go struct tags, Python type hints + Pydantic/marshmallow, TypeScript interfaces + zod/class-validator, Rust struct derives, Java annotations) — `references/framework_patterns.md` has examples, but the underlying question is always the same: *what does the handler reject before doing real work?*
   - Response fields: same, plus every status code/error state the handler can actually produce
   - Side effects worth noting (e.g., "creates a session with a TTL", "this call is idempotent")

   Don't guess at validation rules or error codes — read the actual handler code path, including every error branch, regardless of language. A docs skill that documents the happy path only is actively misleading.

4. **Write both outputs together** using the templates in `references/`. Generate the OpenAPI YAML first (it's the stricter format and forces you to nail down types), then derive the Markdown from it — this order prevents the two from silently disagreeing on a type or required field.

5. **Cross-check before finishing**: every path/operation in the OpenAPI YAML should have a matching section in the Markdown, and vice versa. Every field type in one should match the other exactly (e.g., don't call something a `string` in one and `integer` in the other). Two specific things worth checking mechanically rather than trusting a visual skim:
   - **Terminology**: re-check both outputs against any naming/terminology rules you found in step 0 — it's easy to apply a vocabulary rule (e.g., "say 'link' not 'shortlink'") inconsistently across a long document, getting it right in one section and slipping back into the codebase's internal naming in another (a title, a heading, a stray sentence). A quick `grep` for the term you were told *not* to use, across both output files, catches this in seconds.
   - **Structural consistency in the YAML itself**: if you're identifying operations via a `description:` field for some schemas, make sure you did it for *all* of them, not just the first few before the file got long — a YAML comment (`#`) is invisible to any tool that actually parses the file, so information that exists only in a comment effectively doesn't exist for codegen/linting purposes. Grep for your own comment markers vs. your own description fields and confirm the count matches.

## Keeping docs in sync

Triggered when the user changed code and wants docs updated, or asks you to "check if the docs are still accurate."

1. **Diff the contract, not the implementation.** Compare the current schema (`.proto`, `.graphql`) or route/handler signatures against what's documented. Look specifically for:
   - New endpoints/RPCs/queries not yet documented
   - Removed endpoints still documented (stale — flag for removal, don't just delete silently)
   - Changed field names, types, or required/optional status
   - New error codes or status codes in the handler that aren't in the docs

2. **Update minimally.** Don't rewrite sections that are still accurate just because you're touching the file — this makes diffs hard to review. Change only what actually changed.

3. **Call out breaking changes explicitly.** If a field became required, renamed, or removed, say so plainly to the user — this is the kind of thing that breaks frontend integrations silently if missed. Use a short callout like:
   > ⚠️ Breaking change: `session_id` is now required in `POST /challenge/verify` (was optional).

4. Update both Markdown and YAML in the same pass, same as generation mode.

## Writing docs first (design mode)

Triggered when there's no code yet — the user is designing the API and wants docs as the spec to build against.

1. **Interview briefly if the shape is unclear**: what resource/action is this, what does the client send, what should come back on success and on each failure mode. Don't over-ask — if the user already described the shape in conversation, use that.
2. Write the OpenAPI YAML as the source of truth (it's meant for exactly this — designing a contract before implementation).
3. Derive Markdown from it.
4. Tell the user this is a proposed contract — once they implement it, run this skill again in **sync mode** to verify the implementation actually matches what was designed.

## Error documentation (don't skip this)

Every endpoint needs its failure modes documented, not just its happy path. At minimum, cover:
- Validation errors (HTTP 400 / gRPC `InvalidArgument` / GraphQL validation errors in the `errors` array)
- Auth/session errors (401/403/410, or the equivalent in non-HTTP styles)
- Upstream/dependency failures (502/503 — e.g., a downstream service, database, or sidecar being unreachable)
- What the error response body actually looks like in this specific codebase — error envelope shape varies a lot by framework (some return `{"error": "..."}`, others `{"message": "...", "code": "..."}`, others a typed error class serialized differently) — copy the real shape from the code, don't assume a generic one

## Output locations

- Markdown → `docs/api.md` (create `docs/` if it doesn't exist) unless the user specifies otherwise
- OpenAPI YAML → `docs/openapi.yaml`
- If the project already has docs elsewhere (check for `docs/`, `api/`, or a `swagger.yaml` at root), use the existing location instead of inventing a new one

## Reference files

- `references/framework_patterns.md` — route-registration and validation idioms for common frameworks across languages, plus the general detection logic to fall back on for anything not listed
- `references/openapi_template.md` — structure and conventions for the OpenAPI output, including how to document gRPC- or GraphQL-derived endpoints in OpenAPI form (neither has a native OpenAPI mapping, so this defines the convention to use consistently)
- `references/markdown_template.md` — structure and conventions for the Markdown output
- `references/proto_extraction.md` — how to read a `.proto` file and map enums/messages/services to documentation fields, with a worked example (gRPC-specific; for GraphQL schemas, the same idea applies — read the `.graphql` SDL directly rather than inferring from resolver code)
