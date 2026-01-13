---
name: effect
description: Write idiomatic Effect TypeScript code. Use this skill when working with Effect, @effect/* packages, or projects using Effect for error handling, services, layers, schemas, and functional programming patterns.
---

# Effect TypeScript

Effect is a TypeScript library for building type-safe, composable, and production-grade applications. It provides structured concurrency, typed error handling, dependency injection via services/layers, and a rich standard library.

## Research Strategy

When you need Effect-specific information, use these methods in parallel for fastest results:

### 1. Search Effect MCP Documentation
Use the `effect_effect_docs_search` tool to search official docs:
```
effect_effect_docs_search({ query: "your search term" })
```
Then retrieve content with `effect_get_effect_doc({ documentId: <id> })`.

### 2. Search AnswerOverflow for Community Knowledge
The Effect Discord community (server ID: `795981131316985866`) is indexed on AnswerOverflow:
```
answeroverflow_search_answeroverflow({
  query: "your question",
  serverId: "795981131316985866"
})
```
Use `answeroverflow_get_thread_messages({ threadId: "<id>" })` to read full discussions.

### 3. Read Source Code Directly
Local Effect source repositories provide the most authoritative information:
- `~/.llms/effect` - Core Effect packages (effect, schema, platform, cli, sql, rpc, ai, etc.)
- `~/.llms/effect-atom` - Atom packages for frontend/reactive state (atom, atom-react, atom-vue, atom-livestore)

Pull latest before reading:
```bash
git -C ~/.llms/effect pull
git -C ~/.llms/effect-atom pull
```

### 4. Launch Parallel Research Agents
For complex questions, spin up multiple Task agents simultaneously:
```
Task({ description: "Research Effect services", prompt: "...", subagent_type: "explore" })
Task({ description: "Research Effect schema", prompt: "...", subagent_type: "explore" })
```

### Authoritative Sources
These individuals are highly reliable sources on Effect. Their posts and code can be trusted:
- Michael Arnaldi (creator)
- Giulio Canti
- Matt Pocock
- Kit Langton
- Patrick Roza
- Johannes Schickling

---

## Core Patterns

### Effect.gen for Sequential Code
Use `Effect.gen` with `yield*` for readable sequential code (like async/await for Promises):

```typescript
import { Effect } from "effect"

const program = Effect.gen(function* () {
  const data = yield* fetchData
  yield* Effect.logInfo(`Processing: ${data}`)
  return yield* processData(data)
})
```

### Effect.fn for Named, Traced Functions
Use `Effect.fn` for functions that return Effects. Provides call-site tracing and integrates with telemetry:

```typescript
import { Effect } from "effect"

const processUser = Effect.fn("processUser")(function* (userId: string) {
  yield* Effect.logInfo(`Processing user ${userId}`)
  const user = yield* getUser(userId)
  return yield* processData(user)
})
```

With transformation (retry, timeout):
```typescript
import { Effect, flow, Schedule } from "effect"

const fetchWithRetry = Effect.fn("fetchWithRetry")(
  function* (url: string) {
    return yield* fetchData(url)
  },
  flow(
    Effect.retry(Schedule.recurs(3)),
    Effect.timeout("5 seconds")
  )
)
```

### Pipe for Instrumentation
Use `.pipe()` to add cross-cutting concerns:

```typescript
import { Effect, Schedule } from "effect"

const program = fetchData.pipe(
  Effect.timeout("5 seconds"),
  Effect.retry(Schedule.exponential("100 millis").pipe(Schedule.compose(Schedule.recurs(3)))),
  Effect.tap((data) => Effect.logInfo(`Fetched: ${data}`)),
  Effect.withSpan("fetchData")
)
```

---

## Services & Layers

### Define Services with Context.Tag
Services are contracts defined with `Context.Tag`:

```typescript
import { Context, Effect } from "effect"

class Database extends Context.Tag("@app/Database")<
  Database,
  {
    readonly query: (sql: string) => Effect.Effect<unknown[]>
    readonly execute: (sql: string) => Effect.Effect<void>
  }
>() {}
```

**Rules:**
- Tag identifiers must be unique (use `@path/ServiceName` pattern)
- Service methods should have no dependencies (R = never)
- Use readonly properties

### Implement Services with Layers
Layers provide implementations:

```typescript
import { Effect, Layer } from "effect"

class Users extends Context.Tag("@app/Users")<
  Users,
  {
    readonly findById: (id: UserId) => Effect.Effect<User, UserNotFoundError>
  }
>() {
  static readonly layer = Layer.effect(
    Users,
    Effect.gen(function* () {
      const http = yield* HttpClient.HttpClient
      
      const findById = Effect.fn("Users.findById")(function* (id: UserId) {
        const response = yield* http.get(`/users/${id}`)
        return yield* HttpClientResponse.schemaBodyJson(User)(response)
      })

      return Users.of({ findById })
    })
  )
}
```

### Provide Layers Once at Entry Point
```typescript
const appLayer = userServiceLayer.pipe(
  Layer.provideMerge(databaseLayer),
  Layer.provideMerge(configLayer)
)

const main = program.pipe(Effect.provide(appLayer))
Effect.runPromise(main)
```

### Layer Memoization
Store parameterized layers in constants to avoid duplication:

```typescript
// Good: same reference, single pool
const postgresLayer = Postgres.layer({ url: "...", poolSize: 10 })
const appLayer = Layer.merge(
  UserRepo.layer.pipe(Layer.provide(postgresLayer)),
  OrderRepo.layer.pipe(Layer.provide(postgresLayer))
)

// Bad: different references, two pools
const badLayer = Layer.merge(
  UserRepo.layer.pipe(Layer.provide(Postgres.layer({ url: "..." }))),
  OrderRepo.layer.pipe(Layer.provide(Postgres.layer({ url: "..." })))
)
```

---

## Data Modeling with Schema

### Schema.Class for Records
```typescript
import { Schema } from "effect"

const UserId = Schema.String.pipe(Schema.brand("UserId"))
type UserId = typeof UserId.Type

class User extends Schema.Class<User>("User")({
  id: UserId,
  name: Schema.String,
  email: Schema.String,
  createdAt: Schema.Date,
}) {
  get displayName() {
    return `${this.name} (${this.email})`
  }
}
```

### Branded Types
Brand primitives to prevent mixing:

```typescript
const UserId = Schema.String.pipe(Schema.brand("UserId"))
const PostId = Schema.String.pipe(Schema.brand("PostId"))
const Email = Schema.String.pipe(Schema.brand("Email"))
const Port = Schema.Int.pipe(Schema.between(1, 65535), Schema.brand("Port"))
```

### Variants with Schema.TaggedClass
```typescript
import { Match, Schema } from "effect"

class Success extends Schema.TaggedClass<Success>()("Success", {
  value: Schema.Number,
}) {}

class Failure extends Schema.TaggedClass<Failure>()("Failure", {
  error: Schema.String,
}) {}

const Result = Schema.Union(Success, Failure)
type Result = typeof Result.Type

// Pattern match
const render = (result: Result) =>
  Match.valueTags(result, {
    Success: ({ value }) => `Got: ${value}`,
    Failure: ({ error }) => `Error: ${error}`,
  })
```

---

## Error Handling

### Schema.TaggedError for Domain Errors
```typescript
import { Schema } from "effect"

class ValidationError extends Schema.TaggedError<ValidationError>()(
  "ValidationError",
  {
    field: Schema.String,
    message: Schema.String,
  }
) {}

class NotFoundError extends Schema.TaggedError<NotFoundError>()(
  "NotFoundError",
  {
    resource: Schema.String,
    id: Schema.String,
  }
) {}
```

TaggedErrors are yieldable directly (no Effect.fail needed):
```typescript
const program = Effect.gen(function* () {
  if (!valid) {
    yield* ValidationError.make({ field: "email", message: "Invalid format" })
  }
  return data
})
```

### Recovery with catchTag/catchTags
```typescript
const recovered = program.pipe(
  Effect.catchTag("NotFoundError", (error) =>
    Effect.succeed(`Not found: ${error.id}`)
  )
)

const allRecovered = program.pipe(
  Effect.catchTags({
    NotFoundError: () => Effect.succeed("default"),
    ValidationError: () => Effect.succeed("invalid"),
  })
)
```

### Expected Errors vs Defects
- **Typed errors**: Domain failures callers can handle (validation, not found, permission denied)
- **Defects**: Unrecoverable bugs/invariant violations

Use `Effect.orDie` when recovery isn't possible:
```typescript
const main = Effect.gen(function* () {
  const config = yield* loadConfig.pipe(Effect.orDie)
})
```

### Schema.Defect for External Errors
Wrap unknown errors from external libraries:
```typescript
class ApiError extends Schema.TaggedError<ApiError>()(
  "ApiError",
  {
    endpoint: Schema.String,
    statusCode: Schema.Number,
    error: Schema.Defect,  // Wraps unknown errors
  }
) {}
```

---

## Testing with @effect/vitest

### Basic Setup
```typescript
import { Effect } from "effect"
import { describe, expect, it } from "@effect/vitest"

describe("Feature", () => {
  it.effect("works", () =>
    Effect.gen(function* () {
      const result = yield* Effect.succeed(42)
      expect(result).toBe(42)
    })
  )
})
```

### Test Variants
- `it.effect()` - Standard Effect tests
- `it.scoped()` - Tests with scoped resources (auto-cleanup)
- `it.live()` - Tests with real clock (no TestClock)

### Test Layers
Provide fresh layers per test:
```typescript
it.effect("isolated test", () =>
  Effect.gen(function* () {
    const db = yield* Database
    // ...
  }).pipe(Effect.provide(Database.testLayer))
)
```

### TestClock for Time
```typescript
import { Effect, Fiber, TestClock } from "effect"

it.effect("time-based", () =>
  Effect.gen(function* () {
    const fiber = yield* Effect.delay(Effect.succeed("done"), "10 seconds").pipe(Effect.fork)
    yield* TestClock.adjust("10 seconds")
    const result = yield* Fiber.join(fiber)
    expect(result).toBe("done")
  })
)
```

---

## Common Combinators

### Retry and Timeout
```typescript
const retryPolicy = Schedule.exponential("100 millis").pipe(
  Schedule.compose(Schedule.recurs(3))
)

const resilient = callApi.pipe(
  Effect.timeout("2 seconds"),
  Effect.retry(retryPolicy),
  Effect.timeout("10 seconds")  // Overall timeout
)
```

### Concurrency
```typescript
// Run effects in parallel
const results = yield* Effect.all([fetchA, fetchB, fetchC], { concurrency: "unbounded" })

// With limited concurrency
const results = yield* Effect.all(effects, { concurrency: 5 })
```

### Resource Management
```typescript
const program = Effect.acquireUseRelease(
  openFile("data.txt"),
  (file) => readFile(file),
  (file) => closeFile(file)
)
```

---

## Anti-Patterns

### Avoid
- Mixing `Effect.provide` throughout code (provide once at entry)
- Using raw `try/catch` instead of Effect error handling
- Creating layers inline multiple times (memoization breaks)
- Using `Effect.runSync/runPromise` deep in the call stack
- Ignoring typed errors with `Effect.orDie` when recovery is possible

### Prefer
- `Effect.fn` over plain functions returning Effects
- `Schema.TaggedError` over plain Error classes
- Branded types over raw primitives
- Services/Layers over global singletons
- `yield*` over `.pipe(Effect.flatMap(...))` for sequential code
