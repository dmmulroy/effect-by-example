# Effectable: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem Effectable Solves

When building Effect-based applications, you often need to create custom abstractions that work seamlessly with Effect's `yield*` syntax and the broader Effect ecosystem. Without Effectable, creating such abstractions requires verbose boilerplate and manual implementation of multiple type IDs and protocols.

```typescript
// Traditional approach - problematic custom Effect-like type
class CustomRef<A> {
  constructor(private value: A) {}
  
  // Doesn't work with yield* syntax
  get(): Effect.Effect<A> {
    return Effect.succeed(this.value)
  }
  
  // Can't be used in Effect.gen
  set(value: A): Effect.Effect<void> {
    this.value = value
    return Effect.void
  }
}

// Usage requires verbose Effect.flatMap chains
const program = pipe(
  customRef.get(),
  Effect.flatMap(value => 
    Effect.flatMap(customRef.set(value + 1), () =>
      customRef.get()
    )
  )
)
```

This approach leads to:
- **Poor Ergonomics** - Cannot use `yield*` syntax for clean, imperative-style code
- **Ecosystem Fragmentation** - Custom types don't integrate with Effect tooling (tracing, etc.)
- **Type Safety Issues** - Manual protocol implementation is error-prone

### The Effectable Solution

Effectable provides base classes and protocols to create Effect-like abstractions that work seamlessly with the Effect ecosystem.

```typescript
import { Effectable, Effect } from "effect"

class CustomRef<A> extends Effectable.Class<A> {
  constructor(private value: A) {
    super()
  }
  
  // The commit method defines what happens when yielded
  commit(): Effect.Effect<A> {
    return Effect.succeed(this.value)
  }
  
  set(value: A): Effect.Effect<void> {
    this.value = value
    return Effect.void
  }
}

// Now works seamlessly with Effect.gen
const program = Effect.gen(function* () {
  const value = yield* customRef  // Clean yield* syntax!
  yield* customRef.set(value + 1)
  return yield* customRef
})
```

### Key Concepts

**Effectable.Class**: Base class for creating Effect-like abstractions that can be yielded from

**Effectable.StructuralClass**: Base class for types that should have structural equality (useful for errors)

**commit() method**: Defines the Effect that runs when your type is yielded from

## Basic Usage Patterns

### Pattern 1: Simple Effect-like Wrapper

```typescript
import { Effectable, Effect } from "effect"

class Computation<A> extends Effectable.Class<A> {
  constructor(private effect: Effect.Effect<A>) {
    super()
  }
  
  commit(): Effect.Effect<A> {
    return this.effect
  }
}

// Usage with yield*
const program = Effect.gen(function* () {
  const result = yield* new Computation(Effect.succeed(42))
  return result * 2
})
```

### Pattern 2: Stateful Effect-like Type

```typescript
import { Effectable, Effect, Ref } from "effect"

class Counter extends Effectable.Class<number> {
  constructor(private ref: Ref.Ref<number>) {
    super()
  }
  
  commit(): Effect.Effect<number> {
    return Ref.get(this.ref)
  }
  
  increment(): Effect.Effect<void> {
    return Ref.update(this.ref, n => n + 1)
  }
  
  decrement(): Effect.Effect<void> {
    return Ref.update(this.ref, n => n - 1)
  }
}

// Helper constructor
const makeCounter = (initial: number) => Effect.gen(function* () {
  const ref = yield* Ref.make(initial)
  return new Counter(ref)
})
```

### Pattern 3: Error Types with StructuralClass

```typescript
import { Effectable, Effect } from "effect"

class ValidationError extends Effectable.StructuralClass<never, ValidationError> {
  readonly _tag = "ValidationError"
  
  constructor(
    readonly field: string,
    readonly message: string
  ) {
    super()
  }
  
  commit(): Effect.Effect<never, ValidationError> {
    return Effect.fail(this)
  }
}

// Usage in validation logic
const validateEmail = (email: string) => Effect.gen(function* () {
  if (!email.includes("@")) {
    yield* new ValidationError("email", "Must contain @")
  }
  return email
})
```

## Real-World Examples

### Example 1: Database Connection Pool

A connection pool that can be yielded to get a connection, with automatic resource management.

```typescript
import { Effectable, Effect, Pool, Context } from "effect"

class DatabaseConnection extends Effectable.Class<Connection, DatabaseError, never> {
  constructor(private pool: Pool.Pool<Connection, DatabaseError>) {
    super()
  }
  
  commit(): Effect.Effect<Connection, DatabaseError> {
    return Pool.get(this.pool)
  }
  
  query<T>(sql: string): Effect.Effect<T[], DatabaseError> {
    return Effect.gen(function* () {
      const connection = yield* this
      return yield* runQuery(connection, sql)
    })
  }
  
  transaction<T>(
    operation: Effect.Effect<T, DatabaseError, never>
  ): Effect.Effect<T, DatabaseError> {
    return Effect.gen(function* () {
      const connection = yield* this
      yield* beginTransaction(connection)
      
      const result = yield* operation.pipe(
        Effect.catchAllCause(cause => 
          Effect.gen(function* () {
            yield* rollback(connection)
            return yield* Effect.failCause(cause)
          })
        )
      )
      
      yield* commit(connection)
      return result
    })
  }
}

// Service creation
const makeDatabaseService = Effect.gen(function* () {
  const pool = yield* Pool.make({
    acquire: acquireConnection(),
    size: 10
  })
  return new DatabaseConnection(pool)
})

// Usage in application
const userService = Effect.gen(function* () {
  const db = yield* DatabaseConnection
  
  // Direct yielding gets a connection
  const connection = yield* db
  
  // Or use helper methods
  const users = yield* db.query<User>("SELECT * FROM users")
  
  return yield* db.transaction(Effect.gen(function* () {
    yield* db.query("INSERT INTO users ...")
    return yield* db.query("SELECT LAST_INSERT_ID()")
  }))
})
```

### Example 2: HTTP Client with Request Context

An HTTP client that maintains request context and can be yielded for making requests.

```typescript
import { Effectable, Effect, Context, Logger } from "effect"

class HttpClient extends Effectable.Class<
  HttpResponse,
  HttpError,
  Logger.Logger
> {
  constructor(
    private baseUrl: string,
    private defaultHeaders: Record<string, string> = {}
  ) {
    super()
  }
  
  commit(): Effect.Effect<HttpResponse, HttpError, Logger.Logger> {
    // Default GET to base URL when yielded directly
    return this.get("/")
  }
  
  get(path: string, headers?: Record<string, string>): Effect.Effect<
    HttpResponse,
    HttpError,
    Logger.Logger
  > {
    return this.request("GET", path, undefined, headers)
  }
  
  post<T>(
    path: string,
    body: T,
    headers?: Record<string, string>
  ): Effect.Effect<HttpResponse, HttpError, Logger.Logger> {
    return this.request("POST", path, body, headers)
  }
  
  private request<T>(
    method: string,
    path: string,
    body?: T,
    headers: Record<string, string> = {}
  ): Effect.Effect<HttpResponse, HttpError, Logger.Logger> {
    return Effect.gen(function* () {
      const logger = yield* Logger.Logger
      
      yield* Logger.info(`${method} ${this.baseUrl}${path}`)
      
      const response = yield* Effect.tryPromise({
        try: () => fetch(`${this.baseUrl}${path}`, {
          method,
          headers: { ...this.defaultHeaders, ...headers },
          body: body ? JSON.stringify(body) : undefined
        }),
        catch: error => new HttpError("NetworkError", String(error))
      })
      
      if (!response.ok) {
        return yield* Effect.fail(
          new HttpError("HttpError", `${response.status}: ${response.statusText}`)
        )
      }
      
      const data = yield* Effect.tryPromise({
        try: () => response.json(),
        catch: error => new HttpError("ParseError", String(error))
      })
      
      yield* Logger.info(`Response received: ${response.status}`)
      
      return {
        status: response.status,
        headers: Object.fromEntries(response.headers.entries()),
        data
      }
    })
  }
}

class HttpError extends Effectable.StructuralClass<never, HttpError> {
  readonly _tag = "HttpError"
  
  constructor(
    readonly type: "NetworkError" | "HttpError" | "ParseError",
    readonly message: string
  ) {
    super()
  }
  
  commit(): Effect.Effect<never, HttpError> {
    return Effect.fail(this)
  }
}

// Service setup
const ApiClient = Context.GenericTag<HttpClient>("ApiClient")

const makeApiService = Effect.gen(function* () {
  return new HttpClient("https://api.example.com", {
    "Authorization": "Bearer token",
    "Content-Type": "application/json"
  })
})

// Usage
const userApi = Effect.gen(function* () {
  const client = yield* ApiClient
  
  // Get user data
  const response = yield* client.get("/users/123")
  
  // Create new user
  const newUser = yield* client.post("/users", {
    name: "John Doe",
    email: "john@example.com"
  })
  
  return newUser.data
}).pipe(
  Effect.provideService(ApiClient, makeApiService)
)
```

### Example 3: Event Store with Replay Capability

An event store that can be yielded to append events and replay event streams.

```typescript
import { Effectable, Effect, Stream, Chunk, Ref } from "effect"

class EventStore<E> extends Effectable.Class<
  readonly E[],
  EventStoreError,
  never
> {
  constructor(
    private events: Ref.Ref<Chunk.Chunk<E>>,
    private position: Ref.Ref<number>
  ) {
    super()
  }
  
  commit(): Effect.Effect<readonly E[], EventStoreError> {
    return this.getEvents()
  }
  
  append(event: E): Effect.Effect<void, EventStoreError> {
    return Effect.gen(function* () {
      yield* Ref.update(this.events, events => 
        Chunk.append(events, event)
      )
    })
  }
  
  appendBatch(events: readonly E[]): Effect.Effect<void, EventStoreError> {
    return Effect.gen(function* () {
      yield* Ref.update(this.events, existing => 
        Chunk.appendAll(existing, events)
      )
    })
  }
  
  getEvents(): Effect.Effect<readonly E[], EventStoreError> {
    return Effect.gen(function* () {
      const events = yield* Ref.get(this.events)
      return Chunk.toReadonlyArray(events)
    })
  }
  
  getEventsFrom(position: number): Effect.Effect<readonly E[], EventStoreError> {
    return Effect.gen(function* () {
      const events = yield* Ref.get(this.events)
      return Chunk.toReadonlyArray(Chunk.drop(events, position))
    })
  }
  
  replay(): Stream.Stream<E, EventStoreError> {
    return Stream.fromIterableEffect(this.getEvents())
  }
  
  replayFrom(position: number): Stream.Stream<E, EventStoreError> {
    return Stream.fromIterableEffect(this.getEventsFrom(position))
  }
  
  getCurrentPosition(): Effect.Effect<number, EventStoreError> {
    return Effect.gen(function* () {
      const events = yield* Ref.get(this.events)
      return Chunk.size(events)
    })
  }
}

class EventStoreError extends Effectable.StructuralClass<never, EventStoreError> {
  readonly _tag = "EventStoreError"
  
  constructor(readonly message: string) {
    super()
  }
  
  commit(): Effect.Effect<never, EventStoreError> {
    return Effect.fail(this)
  }
}

// Factory function
const makeEventStore = <E>() => Effect.gen(function* () {
  const events = yield* Ref.make(Chunk.empty<E>())
  const position = yield* Ref.make(0)
  return new EventStore(events, position)
})

// Usage in event sourcing system
interface OrderEvent {
  type: "OrderCreated" | "OrderShipped" | "OrderCancelled"
  orderId: string
  timestamp: Date
  data: unknown
}

const orderEventStore = Effect.gen(function* () {
  const eventStore = yield* makeEventStore<OrderEvent>()
  
  // Append events
  yield* eventStore.append({
    type: "OrderCreated",
    orderId: "order-123",
    timestamp: new Date(),
    data: { customerId: "customer-456" }
  })
  
  // Get all events by yielding the store directly
  const allEvents = yield* eventStore
  
  // Replay events as a stream
  const eventStream = eventStore.replay()
  
  yield* Stream.runForEach(eventStream, event => 
    Effect.logInfo(`Processing event: ${event.type}`)
  )
  
  return eventStore
})
```

## Advanced Features Deep Dive

### Feature 1: Integration with Effect Type IDs

Effectable classes automatically implement multiple Effect type IDs, making them compatible with various Effect ecosystem tools.

#### Basic Type ID Implementation

```typescript
import { Effectable, Effect } from "effect"

class CustomEffectable<A> extends Effectable.Class<A> {
  commit(): Effect.Effect<A> {
    return Effect.succeed(this.value)
  }
  
  constructor(private value: A) {
    super()
  }
}

// Automatically implements:
// - EffectTypeId (works with Effect.gen)
// - StreamTypeId (works with Stream operations)
// - SinkTypeId (works with Sink operations)  
// - ChannelTypeId (works with Channel operations)
```

#### Real-World Type ID Example

```typescript
import { Effectable, Effect, Stream, Equal, Hash } from "effect"

class TimedValue<A> extends Effectable.Class<A> {
  constructor(
    private value: A,
    private timestamp: Date = new Date()
  ) {
    super()
  }
  
  commit(): Effect.Effect<A> {
    return Effect.succeed(this.value)
  }
  
  age(): number {
    return Date.now() - this.timestamp.getTime()
  }
  
  isExpired(ttl: number): boolean {
    return this.age() > ttl
  }
  
  // Works as a Stream source
  toStream(): Stream.Stream<A> {
    return Stream.fromEffect(this)
  }
  
  // Equal implementation for structural comparison
  [Equal.symbol](that: unknown): boolean {
    return that instanceof TimedValue &&
           Equal.equals(this.value, that.value) &&
           this.timestamp.getTime() === that.timestamp.getTime()
  }
  
  [Hash.symbol](): number {
    return Hash.hash([this.value, this.timestamp.getTime()])
  }
}

// Usage demonstrating cross-ecosystem compatibility
const timedProgram = Effect.gen(function* () {
  const timedValue = new TimedValue("hello", new Date())
  
  // Works with Effect.gen
  const value = yield* timedValue
  
  // Works with Stream
  const stream = timedValue.toStream()
  yield* Stream.runForEach(stream, v => Effect.logInfo(v))
  
  return value
})
```

### Feature 2: Advanced Error Handling with StructuralClass

StructuralClass provides structural equality for error types, making error handling more predictable.

#### Complex Error Hierarchy

```typescript
import { Effectable, Effect, Data } from "effect"

// Base error class
abstract class ServiceError extends Effectable.StructuralClass<never, ServiceError> {
  abstract readonly _tag: string
  abstract readonly message: string
  
  commit(): Effect.Effect<never, ServiceError> {
    return Effect.fail(this)
  }
}

// Specific error types
class ValidationError extends ServiceError {
  readonly _tag = "ValidationError"
  
  constructor(
    readonly field: string,
    readonly message: string,
    readonly value: unknown
  ) {
    super()
  }
}

class DatabaseError extends ServiceError {
  readonly _tag = "DatabaseError"
  
  constructor(
    readonly message: string,
    readonly query?: string,
    readonly cause?: unknown
  ) {
    super()
  }
}

class NetworkError extends ServiceError {
  readonly _tag = "NetworkError"
  
  constructor(
    readonly message: string,
    readonly url: string,
    readonly status?: number
  ) {
    super()
  }
}

// Error recovery patterns
const createUser = (userData: unknown) => Effect.gen(function* () {
  // Validation
  if (typeof userData !== "object" || !userData) {
    yield* new ValidationError("userData", "Must be an object", userData)
  }
  
  const user = userData as { name?: string; email?: string }
  
  if (!user.name) {
    yield* new ValidationError("name", "Name is required", user.name)
  }
  
  if (!user.email || !user.email.includes("@")) {
    yield* new ValidationError("email", "Valid email is required", user.email)
  }
  
  // Database operation
  const result = yield* saveUser(user).pipe(
    Effect.catchTag("DatabaseError", dbError => 
      Effect.fail(new DatabaseError(
        `Failed to save user: ${dbError.message}`,
        "INSERT INTO users...",
        dbError
      ))
    )
  )
  
  return result
}).pipe(
  Effect.catchTags({
    ValidationError: error => Effect.logError(`Validation failed: ${error.message}`)
      .pipe(Effect.zipRight(Effect.fail(error))),
    DatabaseError: error => Effect.logError(`Database error: ${error.message}`)
      .pipe(Effect.zipRight(Effect.fail(error))),
    NetworkError: error => Effect.logError(`Network error: ${error.message}`)
      .pipe(Effect.zipRight(Effect.fail(error)))
  })
)
```

### Feature 3: Performance Optimization Patterns

Effectable types can be optimized for specific use cases through careful implementation.

#### Lazy Evaluation Pattern

```typescript
import { Effectable, Effect, Ref } from "effect"

class LazyComputation<A> extends Effectable.Class<A> {
  private memoized: Ref.Ref<Option.Option<A>> | null = null
  
  constructor(private computation: Effect.Effect<A>) {
    super()
  }
  
  commit(): Effect.Effect<A> {
    return Effect.gen(function* () {
      if (!this.memoized) {
        this.memoized = yield* Ref.make(Option.none<A>())
      }
      
      const cached = yield* Ref.get(this.memoized)
      
      if (Option.isSome(cached)) {
        return cached.value
      }
      
      const result = yield* this.computation
      yield* Ref.set(this.memoized, Option.some(result))
      return result
    })
  }
  
  // Reset the memoization
  reset(): Effect.Effect<void> {
    return this.memoized 
      ? Ref.set(this.memoized, Option.none<A>())
      : Effect.void
  }
}

// Usage for expensive computations
const expensiveOperation = new LazyComputation(
  Effect.gen(function* () {
    yield* Effect.sleep("1 second")
    yield* Effect.logInfo("Computing expensive result...")
    return Math.random() * 1000
  })
)

const program = Effect.gen(function* () {
  // First call: computes the value
  const result1 = yield* expensiveOperation
  
  // Second call: returns cached value
  const result2 = yield* expensiveOperation
  
  console.log(result1 === result2) // true
  
  // Reset and compute again
  yield* expensiveOperation.reset()
  const result3 = yield* expensiveOperation
  
  console.log(result1 === result3) // false (new computation)
})
```

## Practical Patterns & Best Practices

### Pattern 1: Resource Management with Effectable

```typescript
import { Effectable, Effect, Scope } from "effect"

class ManagedResource<T> extends Effectable.Class<T, ResourceError, Scope.Scope> {
  constructor(
    private acquire: Effect.Effect<T, ResourceError>,
    private release: (resource: T) => Effect.Effect<void, never>
  ) {
    super()
  }
  
  commit(): Effect.Effect<T, ResourceError, Scope.Scope> {
    return Effect.acquireRelease(this.acquire, this.release)
  }
  
  // Helper to use the resource within a scope
  use<A, E>(
    operation: (resource: T) => Effect.Effect<A, E>
  ): Effect.Effect<A, E | ResourceError> {
    return Effect.scoped(
      Effect.gen(function* () {
        const resource = yield* this
        return yield* operation(resource)
      })
    )
  }
}

// File handle example
const openFile = (path: string) => new ManagedResource(
  Effect.tryPromise({
    try: () => fs.open(path, "r"),
    catch: error => new ResourceError("FileOpenError", String(error))
  }),
  handle => Effect.tryPromise({
    try: () => handle.close(),
    catch: () => void 0 // Ignore close errors
  }).pipe(Effect.orElse(() => Effect.void))
)

// Usage
const readFileContents = (path: string) =>
  openFile(path).use(handle =>
    Effect.tryPromise({
      try: () => handle.readFile(),
      catch: error => new ResourceError("FileReadError", String(error))
    })
  )
```

### Pattern 2: Composable Configuration

```typescript
import { Effectable, Effect, Context } from "effect"

class ConfigValue<T> extends Effectable.Class<T, ConfigError> {
  constructor(
    private key: string,
    private parse: (raw: string) => Effect.Effect<T, ConfigError>,
    private defaultValue?: T
  ) {
    super()
  }
  
  commit(): Effect.Effect<T, ConfigError> {
    return Effect.gen(function* () {
      const raw = process.env[this.key]
      
      if (!raw) {
        if (this.defaultValue !== undefined) {
          return this.defaultValue
        }
        return yield* Effect.fail(
          new ConfigError("MissingConfig", `Environment variable ${this.key} is required`)
        )
      }
      
      return yield* this.parse(raw)
    })
  }
  
  withDefault(value: T): ConfigValue<T> {
    return new ConfigValue(this.key, this.parse, value)
  }
  
  optional(): ConfigValue<Option.Option<T>> {
    return new ConfigValue(
      this.key,
      raw => Effect.succeed(Option.some(raw)).pipe(
        Effect.flatMap(opt => 
          Option.match(opt, {
            onNone: () => Effect.succeed(Option.none<T>()),
            onSome: value => this.parse(value).pipe(Effect.map(Option.some))
          })
        )
      ),
      Option.none()
    )
  }
}

// Config builders
const stringConfig = (key: string) => 
  new ConfigValue(key, value => Effect.succeed(value))

const numberConfig = (key: string) =>
  new ConfigValue(key, value => {
    const parsed = Number(value)
    return isNaN(parsed)
      ? Effect.fail(new ConfigError("ParseError", `${key} must be a number`))
      : Effect.succeed(parsed)
  })

const booleanConfig = (key: string) =>
  new ConfigValue(key, value => 
    Effect.succeed(value.toLowerCase() === "true")
  )

// Application configuration
const config = Effect.gen(function* () {
  const port = yield* numberConfig("PORT").withDefault(3000)
  const host = yield* stringConfig("HOST").withDefault("localhost")
  const debug = yield* booleanConfig("DEBUG").withDefault(false)
  const databaseUrl = yield* stringConfig("DATABASE_URL")
  
  return { port, host, debug, databaseUrl }
})
```

## Integration Examples

### Integration with Popular Libraries

#### Express.js Server Integration

```typescript
import { Effectable, Effect, Context, Logger } from "effect"
import express from "express"

class ExpressServer extends Effectable.Class<
  express.Application,
  ServerError,
  Logger.Logger
> {
  constructor(private port: number) {
    super()
  }
  
  commit(): Effect.Effect<express.Application, ServerError, Logger.Logger> {
    return Effect.gen(function* () {
      const logger = yield* Logger.Logger
      const app = express()
      
      app.use(express.json())
      
      // Add Effect middleware
      app.use((req, res, next) => {
        req.context = Context.empty()
        next()
      })
      
      yield* Effect.tryPromise({
        try: () => new Promise<void>((resolve, reject) => {
          const server = app.listen(this.port, (error?: Error) => {
            if (error) reject(error)
            else resolve()
          })
        }),
        catch: error => new ServerError("StartupError", String(error))
      })
      
      yield* Logger.info(`Server started on port ${this.port}`)
      return app
    })
  }
  
  route<A, E>(
    path: string,
    method: "GET" | "POST" | "PUT" | "DELETE",
    handler: (req: express.Request) => Effect.Effect<A, E, Logger.Logger>
  ): Effect.Effect<void, ServerError, Logger.Logger> {
    return Effect.gen(function* () {
      const app = yield* this
      
      app[method.toLowerCase()](path, async (req, res) => {
        const result = await Effect.runPromise(
          handler(req).pipe(
            Effect.provideService(Logger.Logger, Logger.make())
          )
        )
        
        res.json(result)
      })
    })
  }
}

// Usage
const server = new ExpressServer(3000)

const setupServer = Effect.gen(function* () {
  // Start server
  yield* server
  
  // Add routes
  yield* server.route("/users/:id", "GET", req => 
    Effect.succeed({ id: req.params.id, name: "John Doe" })
  )
  
  yield* server.route("/users", "POST", req =>
    Effect.gen(function* () {
      const user = req.body
      // Validation and database logic here
      return { id: "new-id", ...user }
    })
  )
})
```

#### Prisma Database Integration

```typescript
import { Effectable, Effect, Context } from "effect"
import { PrismaClient } from "@prisma/client"

class PrismaDatabase extends Effectable.Class<
  PrismaClient,
  DatabaseError,
  never
> {
  private client: PrismaClient | null = null
  
  commit(): Effect.Effect<PrismaClient, DatabaseError> {
    return Effect.gen(function* () {
      if (!this.client) {
        this.client = new PrismaClient()
        yield* Effect.tryPromise({
          try: () => this.client!.$connect(),
          catch: error => new DatabaseError("ConnectionError", String(error))
        })
      }
      return this.client
    })
  }
  
  disconnect(): Effect.Effect<void, DatabaseError> {
    return Effect.gen(function* () {
      if (this.client) {
        yield* Effect.tryPromise({
          try: () => this.client!.$disconnect(),
          catch: error => new DatabaseError("DisconnectionError", String(error))
        })
        this.client = null
      }
    })
  }
  
  query<T>(operation: (client: PrismaClient) => Promise<T>): Effect.Effect<T, DatabaseError> {
    return Effect.gen(function* () {
      const client = yield* this
      return yield* Effect.tryPromise({
        try: () => operation(client),
        catch: error => new DatabaseError("QueryError", String(error))
      })
    })
  }
  
  transaction<T>(
    operations: (client: PrismaClient) => Promise<T>
  ): Effect.Effect<T, DatabaseError> {
    return Effect.gen(function* () {
      const client = yield* this
      return yield* Effect.tryPromise({
        try: () => client.$transaction(async (tx) => operations(tx)),
        catch: error => new DatabaseError("TransactionError", String(error))
      })
    })
  }
}

// Service setup
const Database = Context.GenericTag<PrismaDatabase>("Database")

const makeDatabaseService = Effect.gen(function* () {
  return new PrismaDatabase()
})

// Repository pattern
const UserRepository = {
  findById: (id: string) => Effect.gen(function* () {
    const db = yield* Database
    return yield* db.query(client => 
      client.user.findUnique({ where: { id } })
    )
  }),
  
  create: (userData: { name: string; email: string }) => Effect.gen(function* () {
    const db = yield* Database
    return yield* db.query(client =>
      client.user.create({ data: userData })
    )
  }),
  
  createWithProfile: (
    userData: { name: string; email: string },
    profileData: { bio: string }
  ) => Effect.gen(function* () {
    const db = yield* Database
    
    return yield* db.transaction(async client => {
      const user = await client.user.create({ data: userData })
      const profile = await client.profile.create({
        data: { ...profileData, userId: user.id }
      })
      return { user, profile }
    })
  })
}
```

### Testing Strategies

#### Test Utilities for Effectable Types

```typescript
import { Effectable, Effect, TestContext, it } from "effect"

// Mock implementation
class MockHttpClient extends Effectable.Class<HttpResponse> {
  private responses: Map<string, HttpResponse> = new Map()
  
  constructor() {
    super()
  }
  
  commit(): Effect.Effect<HttpResponse> {
    return this.get("/")
  }
  
  mockResponse(url: string, response: HttpResponse): void {
    this.responses.set(url, response)
  }
  
  get(url: string): Effect.Effect<HttpResponse> {
    const response = this.responses.get(url)
    return response 
      ? Effect.succeed(response)
      : Effect.fail(new HttpError("NotMocked", `No mock for ${url}`))
  }
}

// Test helper
const withMockClient = <A, E>(
  test: (client: MockHttpClient) => Effect.Effect<A, E>
): Effect.Effect<A, E> => Effect.gen(function* () {
  const client = new MockHttpClient()
  return yield* test(client)
})

// Property-based testing
const testCustomRef = Effect.gen(function* () {
  const ref = new CustomRef(0)
  
  // Test yielding returns current value
  const initial = yield* ref
  expect(initial).toBe(0)
  
  // Test setting updates value
  yield* ref.set(42)
  const updated = yield* ref
  expect(updated).toBe(42)
  
  // Test multiple operations
  yield* ref.set(10)
  yield* ref.set(20)
  const final = yield* ref
  expect(final).toBe(20)
})

// Integration test
const testDatabaseIntegration = Effect.gen(function* () {
  const db = yield* makeDatabaseService
  
  // Test connection
  const client = yield* db
  expect(client).toBeDefined()
  
  // Test query
  const result = yield* db.query(client => 
    client.user.findMany({ take: 1 })
  )
  expect(Array.isArray(result)).toBe(true)
}).pipe(
  Effect.provideLayer(TestContext.TestContext.default)
)
```

## Conclusion

Effectable provides a powerful foundation for creating Effect-like abstractions that integrate seamlessly with the Effect ecosystem. It enables clean, composable APIs that work with `yield*` syntax while maintaining type safety and ecosystem compatibility.

Key benefits:
- **Ergonomic Integration**: Custom types work naturally with Effect.gen and yield* syntax
- **Ecosystem Compatibility**: Automatic integration with Effect tooling, tracing, and type IDs
- **Type Safety**: Full TypeScript support with proper variance and error handling
- **Performance**: Optimized base classes with minimal overhead
- **Structural Equality**: StructuralClass provides predictable equality semantics for errors

Use Effectable when building custom Effect-like abstractions, domain-specific wrappers around external libraries, or stateful components that need to integrate cleanly with Effect-based applications.