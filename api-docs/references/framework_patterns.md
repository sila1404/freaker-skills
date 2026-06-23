# Framework Patterns for Route & Schema Detection

This file is a **lookup table of examples**, not an exhaustive list to match against. The underlying detection logic is always the same four steps:

1. Find where a URL path/method gets registered to a handler function
2. Follow that registration to the handler function body
3. Find where the handler reads/validates the request (this is the request schema)
4. Find every place the handler produces a response, success or error (this is the response schema + status codes)

If you encounter a framework not listed below, apply these four steps directly — every framework has *some* mechanism for each step, even if the exact syntax differs. Read a few real route definitions in the repo before assuming you understand the pattern; frameworks often have more than one valid way to register a route (e.g., decorators vs. explicit registration calls in the same Python codebase).

## Detecting the framework

Check the dependency manifest first — it's faster and more reliable than guessing from syntax:

| Manifest file | Language | Look for |
|---|---|---|
| `go.mod` | Go | `gin-gonic/gin`, `labstack/echo`, `gorilla/mux`, or plain `net/http` |
| `package.json` | Node/TypeScript | `express`, `fastify`, `@nestjs/core`, `koa`, `hapi`, `next` |
| `requirements.txt` / `pyproject.toml` | Python | `flask`, `fastapi`, `django`, `tornado`, `aiohttp` |
| `Cargo.toml` | Rust | `actix-web`, `axum`, `rocket`, `warp` |
| `pom.xml` / `build.gradle` | Java/Kotlin | `spring-boot-starter-web`, `io.javalin`, `io.micronaut` |
| `Gemfile` | Ruby | `rails`, `sinatra`, `grape` |
| `composer.json` | PHP | `laravel/framework`, `slim/slim`, `symfony/symfony` |
| `*.csproj` | C# | `Microsoft.AspNetCore.*` |

## Route-registration patterns by framework

**Go — Gin**
```go
r.GET("/users/:id", getUser)
rg := r.Group("/api")
rg.POST("/users", createUser)
```
Request shape: struct with `binding:"..."` tags (`ShouldBindJSON`). Response: `c.JSON(status, ...)` calls — grep all of them per handler for full status-code coverage.

**Go — Echo**
```go
e.GET("/users/:id", getUser)
```
Request: struct + `c.Bind()` + `validator` tags. Response: `c.JSON(status, ...)`.

**Go — net/http (stdlib)**
```go
mux.HandleFunc("GET /users/{id}", getUser)  // Go 1.22+ pattern routing
```
No built-in validation — check manually for `if`-based checks at the top of the handler. Response: `w.WriteHeader(status)` + `json.NewEncoder(w).Encode(...)`.

**Node — Express**
```js
app.get('/users/:id', getUser)
router.post('/users', validateBody(schema), createUser)
```
Request shape: often a separate validation middleware (`zod`, `joi`, `express-validator`) — check for a schema object passed into middleware, not just the handler itself. Response: `res.status(code).json(...)`.

**Node — Fastify**
```js
fastify.post('/users', { schema: { body: userSchema, response: { 200: responseSchema } } }, createUser)
```
Fastify routes often declare the request/response JSON Schema **inline in the route options** — this is a gift, since it's already structured. Prefer reading `schema.body`/`schema.response` over inferring from handler code.

**Node — NestJS**
```ts
@Controller('users')
class UsersController {
  @Get(':id')
  getUser(@Param('id') id: string) { ... }

  @Post()
  createUser(@Body() dto: CreateUserDto) { ... }
}
```
Request shape: the DTO class (`CreateUserDto`), often with `class-validator` decorators (`@IsString()`, `@IsOptional()`). Response: return type of the method, or `@ApiResponse()` Swagger decorators if present — those are often already half-written docs, read them.

**Python — FastAPI**
```python
@app.post("/users", response_model=UserResponse)
def create_user(payload: CreateUserRequest):
    ...
```
This is the easiest case: Pydantic models (`CreateUserRequest`, `UserResponse`) are already a complete, typed schema. FastAPI also auto-generates OpenAPI — check for an existing `/openapi.json` route or exported spec before redoing this work by hand.

**Python — Flask**
```python
@app.route("/users/<int:id>", methods=["GET"])
def get_user(id): ...
```
No built-in validation — check for `request.get_json()` + manual checks, or a `marshmallow`/`pydantic` schema used explicitly inside the handler.

**Python — Django (DRF)**
```python
class UserViewSet(viewsets.ModelViewSet):
    serializer_class = UserSerializer
```
Request/response shape: the `Serializer` class — read its fields directly, including `required=False` and `validators=[...]`.

**Rust — Axum**
```rust
.route("/users/:id", get(get_user))
.route("/users", post(create_user))
```
Request shape: a struct deriving `Deserialize`, often extracted via `Json<CreateUserRequest>` in the handler signature. Response: struct deriving `Serialize`, or explicit `(StatusCode, Json(...))` tuples for error branches.

**Rust — Actix-web**
```rust
#[post("/users")]
async fn create_user(payload: web::Json<CreateUserRequest>) -> impl Responder { ... }
```
Similar to Axum — look at the `web::Json<T>` type parameter for the request shape.

**Java — Spring**
```java
@PostMapping("/users")
public ResponseEntity<UserResponse> createUser(@RequestBody @Valid CreateUserRequest req) { ... }
```
Request shape: the `@RequestBody` class, with `@NotNull`/`@Size`/etc. Bean Validation annotations marking required/optional. Response: the generic type of `ResponseEntity<T>`, plus any `@ExceptionHandler` methods for error shapes — check the whole controller class for those, they're easy to miss.

**Ruby — Rails**
```ruby
# config/routes.rb
post '/users', to: 'users#create'

# app/controllers/users_controller.rb
def create
  user = User.new(user_params)
  if user.save
    render json: user, status: :created
  else
    render json: { errors: user.errors }, status: :unprocessable_entity
  end
end
```
Request shape: `user_params` (strong params) shows accepted fields; model validations (`validates :email, presence: true`) show required/format rules — check the model, not just the controller.

**PHP — Laravel**
```php
Route::post('/users', [UserController::class, 'store']);

public function store(StoreUserRequest $request) { ... }
```
Request shape: the `FormRequest` class (`StoreUserRequest`) has a `rules()` method — that's the validation contract, read it directly.

## GraphQL

Regardless of language, prefer the **schema definition** over resolver code:
```graphql
type Query {
  user(id: ID!): User
}
type Mutation {
  createUser(input: CreateUserInput!): User!
}
input CreateUserInput {
  email: String!
  name: String
}
```
`!` means required/non-null — this is unambiguous in the schema, unlike some REST validation idioms. Document queries and mutations separately. Resolver code (the function implementing each field) shows behavior but the schema is authoritative for shape.

## When nothing matches

If the framework genuinely isn't covered above:
1. Find the package name from the manifest file, and search for "how does `<package>` register routes" — most frameworks document this clearly, and a single example route in the actual codebase combined with that context is enough to extrapolate the rest.
2. Look for any existing route in the codebase and trace it manually: where is the path string, what function does it call, what does that function read from the request object, what does it write to the response.
3. If genuinely stuck on validation rules (no clear schema/validation layer exists), say so explicitly in the docs rather than guessing — e.g., "No explicit validation found; the handler will accept any JSON body and may fail at runtime on missing fields."
