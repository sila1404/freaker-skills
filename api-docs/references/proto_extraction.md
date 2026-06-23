# Extracting Documentation From `.proto` Files

A `.proto` file is already a structured contract — read it directly rather than inferring behavior from the generated stubs or the servicer implementation. The implementation can have bugs; the `.proto` is the agreed contract.

## Mapping proto constructs to documentation

| `.proto` construct | Maps to |
|---|---|
| `service Foo { rpc Bar(...) }` | One documented endpoint/operation per `rpc` line |
| `message` | One schema (OpenAPI) / one field table (Markdown) |
| Field with no `repeated`/`optional` | Required field in proto3 if it's part of a `oneof`, otherwise treat presence semantics per the comments — proto3 scalar fields don't distinguish "unset" from "zero value" unless explicitly marked `optional` |
| `repeated <type>` | Array of `<type>` |
| `enum` | Enum of string names (use the name, not the integer value, in docs — the integer is a wire-format detail) |
| Field comments (`//` above a field) | Pull directly into the field description — proto comments are often the only place intent is documented at all |

## Worked example

Given this `.proto`:

```protobuf
enum Action {
  ACTION_UNSPECIFIED  = 0;
  SMILE               = 1;
  HEAD_TURN_LEFT      = 2;
  HEAD_TURN_RIGHT     = 3;
  EYES_CLOSED         = 4;
}

message AnalyzeRequest {
  // Base64-encoded JPEG/PNG frame from the frontend
  string image_base64 = 1;
  // The action the backend is currently expecting
  Action expected_action = 2;
}

message AnalyzeResponse {
  bool   valid       = 1;
  float  confidence  = 2;
  string reason      = 3;
  Action detected_action = 4;
}

service FaceAnalyzer {
  rpc Analyze(AnalyzeRequest) returns (AnalyzeResponse);
}
```

**Markdown output:**

```markdown
### FaceAnalyzer.Analyze (gRPC)

Analyzes a single video frame against an expected liveness-challenge action.

**Request**

| Field            | Type   | Description                              |
|-------------------|--------|-------------------------------------------|
| image_base64      | string | Base64-encoded JPEG/PNG frame             |
| expected_action   | Action | The action the backend expects (see enum) |

**Response**

| Field            | Type    | Description                                   |
|-------------------|---------|-------------------------------------------------|
| valid             | bool    | Whether the expected action was detected      |
| confidence        | float   | Confidence score, 0.0–1.0                     |
| reason            | string  | Human-readable reason when valid=false        |
| detected_action   | Action  | The action actually detected (may differ)     |

**Action enum**: `SMILE`, `HEAD_TURN_LEFT`, `HEAD_TURN_RIGHT`, `EYES_CLOSED`, `ACTION_UNSPECIFIED`
```

**OpenAPI output** (using the gRPC convention from `openapi_template.md`):

```yaml
/grpc/FaceAnalyzer/Analyze:
  post:
    summary: Analyze a video frame against an expected challenge action
    description: >
      Internal gRPC call (FaceAnalyzer.Analyze) — documented here for
      contract visibility, not externally reachable as HTTP.
    operationId: faceAnalyzerAnalyze
    requestBody:
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/AnalyzeRequest'
    responses:
      '200':
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/AnalyzeResponse'

components:
  schemas:
    Action:
      type: string
      enum: [ACTION_UNSPECIFIED, SMILE, HEAD_TURN_LEFT, HEAD_TURN_RIGHT, EYES_CLOSED]

    AnalyzeRequest:
      type: object
      required: [image_base64, expected_action]
      properties:
        image_base64:
          type: string
          description: Base64-encoded JPEG/PNG frame from the frontend
        expected_action:
          $ref: '#/components/schemas/Action'

    AnalyzeResponse:
      type: object
      properties:
        valid:
          type: boolean
        confidence:
          type: number
          format: float
        reason:
          type: string
        detected_action:
          $ref: '#/components/schemas/Action'
```

## Things to watch for

- **Don't trust the generated Go/Python stub comments over the `.proto` comments** — stubs are generated and may have stale or auto-generated comments that don't reflect the real `.proto` source comments. Always read the `.proto` file itself.
- **`oneof` blocks** mean exactly one of the contained fields will be set — document this explicitly, since it's not obvious from a flat field list.
- **If the same `.proto` message is reused across multiple RPCs**, define it once in `components/schemas` (OpenAPI) and reference it everywhere, rather than redefining it per-endpoint.
