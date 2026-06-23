# OpenAPI Template & Conventions

Use OpenAPI 3.1. Structure every spec this way:

```yaml
openapi: 3.1.0
info:
  title: <API name>
  version: <semver — bump on breaking changes>
  description: <one paragraph, what this API does and who calls it>

servers:
  - url: http://localhost:8080
    description: Local development

paths:
  /challenge/start:
    post:
      summary: <one line>
      description: <what it does, side effects, anything non-obvious>
      operationId: startChallenge
      requestBody:
        required: false
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/StartRequest'
      responses:
        '200':
          description: Session created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/StartResponse'
        '503':
          $ref: '#/components/responses/AnalyzerUnavailable'

components:
  schemas:
    StartResponse:
      type: object
      required: [session_id, actions, next_action, expires_at]
      properties:
        session_id:
          type: string
          format: uuid
        actions:
          type: array
          items:
            type: string
            enum: [smile, head_turn_left, head_turn_right, eyes_closed]
        next_action:
          type: string
        expires_at:
          type: string
          format: date-time

  responses:
    AnalyzerUnavailable:
      description: The face-analysis backend is unreachable
      content:
        application/json:
          schema:
            type: object
            properties:
              error:
                type: string
                example: "face analyzer unavailable"
```

## Conventions to follow consistently

- **`operationId`**: always camelCase, always present — many codegen tools require it.
- **Every property that's required in the Go struct (`binding:"required"`) must be in the schema's `required:` array.** Don't mark something optional in OpenAPI just because it's easier — check the actual validation tag.
- **Use `enum:` for any field with a fixed set of values** (action names, status strings) — don't just type it as `string`.
- **Document every status code the handler can actually return**, not just 200. Pull these from the handler's `c.JSON(http.StatusX, ...)` calls — every branch, including error branches.
- **Reusable error shapes go in `components/responses`**, referenced with `$ref` — don't repeat the same error schema inline in every path.

## Documenting gRPC services in OpenAPI (no native mapping — use this convention)

OpenAPI doesn't have a native concept of gRPC services. When a `.proto` service needs documenting and the project wants one unified OpenAPI file (e.g., to show the internal contract alongside the public REST API), use this convention rather than inventing your own each time:

- Represent each RPC as a `POST` path under a `/grpc/<ServiceName>/<MethodName>` namespace — this is documentation-only, not a real route, and should be noted as such in the path's `description`.
- Map `.proto` `message` → OpenAPI `schema` directly: each field becomes a property, `repeated` → `type: array`, nested messages → nested `$ref`.
- Map `.proto` `enum` → OpenAPI `enum` of strings using the enum value names (not the integer values) — integers from proto numbering are an implementation detail, not part of the documented contract.
- Always note in the path description which transport this actually uses, e.g.:
  > "Internal gRPC call (FaceAnalyzer.Analyze) — documented here for contract visibility, not externally reachable."

This keeps a single spec file as the source of truth for the whole system without pretending gRPC calls are REST endpoints.

## Documenting GraphQL in OpenAPI (different shape than gRPC — don't reuse that convention as-is)

GraphQL doesn't map onto OpenAPI's path-per-operation model the way gRPC does, because a real GraphQL API usually exposes a single transport endpoint (commonly `POST /graphql`) and the actual operations — queries, mutations, subscriptions — are selected by the request body, not the URL. Document it this way instead:

- Create **one path entry for the real transport endpoint** (`/graphql`), described once, noting that the operation is selected via the request body's GraphQL document, not the path.
- For each `Query`/`Mutation` field in the schema, add a `description` block under that one path explaining what operations exist — or, if the OpenAPI file needs to be queryable per-operation by tooling, use one schema component per operation input/output (e.g., `BorrowBookInput`, `BorrowResult`) even though they all live under the same single path. Pick whichever your downstream tooling needs; note the choice at the top of the file so it's not ambiguous later.
- Map GraphQL's `!` (non-null) directly to OpenAPI `required` — this is a clean, unambiguous mapping, unlike inferring "required" from REST validation idioms.
- Map GraphQL `enum` → OpenAPI `enum` of strings, same as the gRPC convention.
- **If the API uses a "result type" pattern for expected failures** (a payload type with a nullable success field alongside an error-code field, rather than throwing into the top-level `errors` array — a common deliberate pattern for business-rule failures as opposed to genuine exceptions) — document that pattern explicitly and do not describe these as HTTP-style error responses with status codes, since a GraphQL result type returns `200 OK` with the error encoded in the response body's shape, not the status code. Don't invent 4xx/5xx status codes for these the way you would for REST — there usually aren't any to map to.
- Note explicitly near the top of the relevant path/schema which transport this uses, e.g.:
  > "GraphQL endpoint — operations are selected by the request body (query/mutation document), not the URL path. Documented per-operation below for clarity, despite all sharing this one path."
- **Put each operation's identity in a real `description:` field on its schema, not just a YAML comment above it.** Comments (`#`) are invisible to any tool that parses the file — codegen, linters, documentation generators all see only the data model. If you write `# Mutation: borrowBook(...)` as a comment but never repeat that identity inside an actual `description:` string, a parser has no way to know which schema corresponds to which operation. Apply this consistently across every operation — it's easy to do it for the first few and drift into comment-only for the rest as the file gets long.

## Common mistakes to avoid

- Forgetting `format: date-time` / `format: uuid` on fields that have an obvious format — codegen tools use these.
- Inlining the same error schema repeatedly instead of using `components/responses`.
- Marking a field optional because the JSON tag has `omitempty` in Go — `omitempty` controls serialization, not whether the field is required on the way in. Check the validation tag, not the serialization tag.
- Mismatched casing between the YAML and the actual JSON wire field names — copy field names exactly from the Go struct's `json:"..."` tag, not the Go field name.
