# Micro: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem Micro Solves

Modern JavaScript applications often need Effect-like capabilities—structured error handling, composability, and type safety—but without the full Effect runtime overhead. Traditional approaches force you to choose between heavy frameworks or reinventing async patterns:

```typescript
// Traditional approach - manual error handling and composition
async function processUserData(userId: string): Promise<ProcessedUser | null> {
  try {
    const user = await fetchUser(userId);
    if (!user) return null;
    
    const validation = await validateUser(user);
    if (!validation.isValid) {
      throw new Error(`Validation failed: ${validation.errors.join(', ')}`);
    }
    
    const enriched = await enrichUserData(user);
    const processed = await processData(enriched);
    
    return processed;
  } catch (error) {
    console.error('Processing failed:', error);
    // How do we handle different error types?
    // How do we retry? Add timeouts? Compose operations?
    throw error;
  }
}

// Bundle includes full Effect runtime even for simple operations
import { Effect } from "effect"

const processUser = (userId: string) =>
  Effect.gen(function* () {
    const user = yield* fetchUser(userId)
    const validated = yield* validateUser(user)  
    const enriched = yield* enrichUserData(validated)
    return yield* processData(enriched)
  })
```

This approach leads to:
- **Bundle Bloat** - Full Effect runtime for simple async operations
- **Complexity Overhead** - Heavy abstractions for lightweight use cases  
- **Library Constraints** - Can't provide Effect-like APIs without runtime dependencies

### The Micro Solution

Micro provides Effect's composability and type safety with minimal bundle impact, starting at **5kb gzipped**:

```typescript
import { Micro } from "effect"

const processUser = (userId: string) =>
  Micro.gen(function* () {
    const user = yield* fetchUser(userId)
    const validated = yield* validateUser(user)
    const enriched = yield* enrichUserData(validated)
    return yield* processData(enriched)
  }).pipe(
    Micro.catchTag("NotFoundError", () => Micro.succeed(null)),
    Micro.timeout(5000),
    Micro.retry({ schedule: Micro.scheduleExponential(100) })
  )

// Same composability, fraction of the bundle size
```

### Key Concepts

**Micro<Success, Error, Requirements>**: A lightweight Effect-like type that represents an asynchronous computation with structured error handling and dependency injection.

**MicroExit**: Result type that captures success/failure outcomes, similar to Effect's Exit but streamlined.

**MicroCause**: Simplified cause tracking for failures, defects, and interruptions without full Effect runtime overhead.

## Basic Usage Patterns

### Pattern 1: Creating Micro Effects

```typescript
import { Micro } from "effect"

// From values
const success = Micro.succeed(42)
const failure = Micro.fail("Something went wrong")

// From synchronous computations
const computed = Micro.sync(() => Math.random() * 100)

// From promises
const fromPromise = Micro.promise(() => fetch("/api/data"))

// From promises with error handling
const safePromise = Micro.tryPromise({
  try: () => fetch("/api/users/123").then(r => r.json()),
  catch: (error) => new Error(`Fetch failed: ${error}`)
})
```

### Pattern 2: Running Micro Effects

```typescript
import { Micro } from "effect"

const effect = Micro.succeed("Hello, Micro!")

// Run synchronously (for pure computations)
const syncResult = Micro.runSync(effect)
console.log(syncResult) // "Hello, Micro!"

// Run as Promise
Micro.runPromise(effect).then(console.log) // "Hello, Micro!"

// Run with exit information
Micro.runPromiseExit(effect).then(exit => {
  if (exit._tag === "Success") {
    console.log("Success:", exit.value)
  } else {
    console.log("Failure:", exit.cause)
  }
})
```

### Pattern 3: Composing Operations

```typescript
import { Micro } from "effect"

const pipeline = Micro.gen(function* () {
  const data = yield* fetchData()
  const validated = yield* validateData(data)
  const processed = yield* processData(validated)
  return processed
}).pipe(
  Micro.timeout(3000),
  Micro.retry({ schedule: Micro.scheduleSpaced(500) }),
  Micro.catchAll(error => Micro.succeed({ error: error.message }))
)
```

## Real-World Examples

### Example 1: Lightweight HTTP Client Library

Building a minimal HTTP client that libraries can embed without runtime overhead:

```typescript
import { Micro } from "effect"

// Error types for structured error handling
class HttpError extends Micro.TaggedError("HttpError")<{
  status: number
  message: string
  url: string
}> {}

class TimeoutError extends Micro.TaggedError("TimeoutError")<{
  timeout: number
  url: string
}> {}

class NetworkError extends Micro.TaggedError("NetworkError")<{
  cause: unknown
  url: string
}> {}

// HTTP client with Micro
interface HttpClient {
  get<T>(url: string, options?: RequestInit): Micro.Micro<T, HttpError | TimeoutError | NetworkError>
  post<T>(url: string, data: unknown, options?: RequestInit): Micro.Micro<T, HttpError | TimeoutError | NetworkError>
}

const createClient = (baseTimeout: number = 5000): HttpClient => ({
  get: <T>(url: string, options?: RequestInit) =>
    Micro.tryPromise({
      try: () => fetch(url, options),
      catch: (cause) => new NetworkError({ cause, url })
    }).pipe(
      Micro.timeout(baseTimeout),
      Micro.catchTag("TimeoutException", () => 
        new TimeoutError({ timeout: baseTimeout, url })
      ),
      Micro.flatMap(response => 
        response.ok
          ? Micro.tryPromise({
              try: () => response.json() as Promise<T>,
              catch: (cause) => new NetworkError({ cause, url })
            })
          : Micro.fail(new HttpError({ 
              status: response.status, 
              message: response.statusText, 
              url 
            }))
      )
    ),

  post: <T>(url: string, data: unknown, options?: RequestInit) =>
    Micro.tryPromise({
      try: () => fetch(url, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(data),
        ...options
      }),
      catch: (cause) => new NetworkError({ cause, url })
    }).pipe(
      Micro.timeout(baseTimeout),
      Micro.catchTag("TimeoutException", () => 
        new TimeoutError({ timeout: baseTimeout, url })
      ),
      Micro.flatMap(response => 
        response.ok
          ? Micro.tryPromise({
              try: () => response.json() as Promise<T>,
              catch: (cause) => new NetworkError({ cause, url })
            })
          : Micro.fail(new HttpError({ 
              status: response.status, 
              message: response.statusText, 
              url 
            }))
      )
    )
})

// Usage - library consumers get structured error handling
const client = createClient(3000)

const fetchUser = (id: string) =>
  client.get<User>(`/api/users/${id}`).pipe(
    Micro.retry({ 
      schedule: Micro.scheduleExponential(100)
    }),
    Micro.catchTag("HttpError", error =>
      error.status === 404 
        ? Micro.succeed(null)
        : Micro.fail(error)
    )
  )
```

### Example 2: Performance-Critical Data Processing

When bundle size matters for client-side data processing:

```typescript
import { Micro } from "effect"

// Processing pipeline for real-time analytics
interface MetricEvent {
  timestamp: number
  type: string
  value: number
  tags: Record<string, string>
}

interface ProcessedMetric {
  aggregatedValue: number
  normalizedTags: Record<string, string>
  bucket: string
  processed_at: number
}

class ValidationError extends Micro.TaggedError("ValidationError")<{
  field: string
  value: unknown
  reason: string
}> {}

class ProcessingError extends Micro.TaggedError("ProcessingError")<{
  stage: string
  event: MetricEvent
  cause: unknown
}> {}

// Lightweight processing pipeline
const processMetrics = (events: MetricEvent[]) =>
  Micro.forEach(events, processEvent, { concurrency: 10 }).pipe(
    Micro.map(results => results.filter(Boolean)), // Remove nulls
    Micro.catchAll(error => {
      console.error("Processing failed:", error)
      return Micro.succeed([])
    })
  )

const processEvent = (event: MetricEvent): Micro.Micro<ProcessedMetric | null, ValidationError | ProcessingError> =>
  Micro.gen(function* () {
    // Validation stage
    const validated = yield* validateEvent(event)
    
    // Normalization stage  
    const normalized = yield* normalizeEvent(validated)
    
    // Aggregation stage
    const aggregated = yield* aggregateEvent(normalized)
    
    return {
      aggregatedValue: aggregated.value,
      normalizedTags: aggregated.tags,
      bucket: getBucket(aggregated.timestamp),
      processed_at: Date.now()
    }
  }).pipe(
    Micro.catchAll(error => {
      if (error._tag === "ValidationError") {
        // Skip invalid events
        return Micro.succeed(null)
      }
      return Micro.fail(error)
    })
  )

const validateEvent = (event: MetricEvent) =>
  Micro.gen(function* () {
    if (typeof event.timestamp !== 'number' || event.timestamp <= 0) {
      return yield* Micro.fail(new ValidationError({
        field: 'timestamp',
        value: event.timestamp,
        reason: 'Must be positive number'
      }))
    }
    
    if (typeof event.value !== 'number' || isNaN(event.value)) {
      return yield* Micro.fail(new ValidationError({
        field: 'value', 
        value: event.value,
        reason: 'Must be valid number'
      }))
    }
    
    return event
  })

const normalizeEvent = (event: MetricEvent) =>
  Micro.sync(() => ({
    ...event,
    tags: Object.fromEntries(
      Object.entries(event.tags)
        .map(([k, v]) => [k.toLowerCase().trim(), v.toString().trim()])
        .filter(([k, v]) => k && v)
    )
  }))

const aggregateEvent = (event: MetricEvent) =>
  Micro.sync(() => {
    // Apply aggregation logic based on event type
    const multiplier = event.type === 'counter' ? 1 : 
                     event.type === 'gauge' ? 1 :
                     event.type === 'histogram' ? 0.1 : 1
    
    return {
      ...event,
      value: event.value * multiplier
    }
  })

const getBucket = (timestamp: number): string => {
  const date = new Date(timestamp)
  const hour = Math.floor(date.getHours())
  return `${date.toISOString().split('T')[0]}-${hour.toString().padStart(2, '0')}`
}
```

### Example 3: Promise-Based Library Migration

Gradually migrating from Promise-based APIs to structured error handling:

```typescript
import { Micro } from "effect"

// Legacy Promise-based database client
interface LegacyDb {
  findUser(id: string): Promise<User | null>
  updateUser(id: string, data: Partial<User>): Promise<User>
  deleteUser(id: string): Promise<boolean>
}

// Micro wrapper for structured error handling
class DbError extends Micro.TaggedError("DbError")<{
  operation: string
  details: string
  originalError?: unknown
}> {}

class UserNotFoundError extends Micro.TaggedError("UserNotFoundError")<{
  userId: string
}> {}

interface UserService {
  getUser(id: string): Micro.Micro<User, UserNotFoundError | DbError>
  updateUser(id: string, data: Partial<User>): Micro.Micro<User, UserNotFoundError | DbError>
  deleteUser(id: string): Micro.Micro<boolean, UserNotFoundError | DbError>
}

const createUserService = (db: LegacyDb): UserService => ({
  getUser: (id: string) =>
    Micro.tryPromise({
      try: () => db.findUser(id),
      catch: (error) => new DbError({
        operation: 'findUser',
        details: 'Database query failed',
        originalError: error
      })
    }).pipe(
      Micro.flatMap(user => 
        user 
          ? Micro.succeed(user)
          : Micro.fail(new UserNotFoundError({ userId: id }))
      )
    ),

  updateUser: (id: string, data: Partial<User>) =>
    Micro.gen(function* () {
      // Check user exists first
      yield* Micro.tryPromise({
        try: () => db.findUser(id),
        catch: (error) => new DbError({
          operation: 'findUser',
          details: 'Failed to verify user exists',
          originalError: error
        })
      }).pipe(
        Micro.flatMap(user =>
          user 
            ? Micro.succeed(user)
            : Micro.fail(new UserNotFoundError({ userId: id }))
        )
      )
      
      // Perform update
      return yield* Micro.tryPromise({
        try: () => db.updateUser(id, data),
        catch: (error) => new DbError({
          operation: 'updateUser',
          details: 'Update operation failed',
          originalError: error
        })
      })
    }),

  deleteUser: (id: string) =>
    Micro.gen(function* () {
      // Check user exists first
      yield* Micro.tryPromise({
        try: () => db.findUser(id),
        catch: (error) => new DbError({
          operation: 'findUser', 
          details: 'Failed to verify user exists',
          originalError: error
        })
      }).pipe(
        Micro.flatMap(user =>
          user
            ? Micro.succeed(user)
            : Micro.fail(new UserNotFoundError({ userId: id }))
        )
      )
      
      // Perform deletion
      return yield* Micro.tryPromise({
        try: () => db.deleteUser(id),
        catch: (error) => new DbError({
          operation: 'deleteUser',
          details: 'Delete operation failed', 
          originalError: error
        })
      })
    })
})

// Usage with structured error handling
const userService = createUserService(legacyDb)

const handleUserUpdate = (userId: string, updates: Partial<User>) =>
  userService.updateUser(userId, updates).pipe(
    Micro.catchTag("UserNotFoundError", error =>
      Micro.fail(new Error(`User ${error.userId} not found`))
    ),
    Micro.catchTag("DbError", error =>
      Micro.fail(new Error(`Database error in ${error.operation}: ${error.details}`))
    ),
    Micro.retry({ 
      schedule: Micro.scheduleExponential(100)
    })
  )
```

## Advanced Features Deep Dive

### Feature 1: MicroScope - Resource Management

Resource management without full Effect runtime overhead:

#### Basic MicroScope Usage

```typescript
import { Micro } from "effect"

// Create and manage resources
const managedResource = Micro.gen(function* () {
  const scope = yield* Micro.scopeMake
  
  // Add cleanup logic
  yield* scope.addFinalizer(() => 
    Micro.sync(() => console.log("Cleaning up database connection"))
  )
  
  const connection = yield* acquireDbConnection()
  return { connection, scope }
})
```

#### Real-World MicroScope Example

```typescript
import { Micro } from "effect"

interface DatabaseConnection {
  query<T>(sql: string, params?: unknown[]): Promise<T[]>
  close(): Promise<void>
}

interface FileHandle {
  write(data: string): Promise<void>
  close(): Promise<void>
}

// Resource management for data export operation
const exportUserData = (userId: string, outputPath: string) =>
  Micro.scoped(
    Micro.gen(function* () {
      // Acquire database connection
      const db = yield* Micro.acquireRelease(
        Micro.tryPromise({
          try: () => createDbConnection(),
          catch: (error) => new Error(`DB connection failed: ${error}`)
        }),
        (connection) => Micro.promise(() => connection.close())
      )
      
      // Acquire file handle
      const file = yield* Micro.acquireRelease(
        Micro.tryPromise({
          try: () => createFileHandle(outputPath),
          catch: (error) => new Error(`File creation failed: ${error}`)
        }),
        (handle) => Micro.promise(() => handle.close())
      )
      
      // Perform export operation
      const userData = yield* Micro.tryPromise({
        try: () => db.query('SELECT * FROM users WHERE id = ?', [userId]),
        catch: (error) => new Error(`Query failed: ${error}`)
      })
      
      const dataJson = JSON.stringify(userData, null, 2)
      
      yield* Micro.tryPromise({
        try: () => file.write(dataJson),
        catch: (error) => new Error(`Write failed: ${error}`)
      })
      
      return `Exported ${userData.length} records to ${outputPath}`
    })
  )

// Resources are automatically cleaned up on success or failure
```

#### Advanced MicroScope: Connection Pooling

```typescript
import { Micro } from "effect"

interface PooledConnection {
  connection: DatabaseConnection
  returnToPool: () => Micro.Micro<void>
}

const createConnectionPool = (maxConnections: number) => {
  const pool: DatabaseConnection[] = []
  const inUse = new Set<DatabaseConnection>()
  
  const acquireConnection = (): Micro.Micro<PooledConnection, Error> =>
    Micro.gen(function* () {
      if (pool.length > 0) {
        const connection = pool.pop()!
        inUse.add(connection)
        
        return {
          connection,
          returnToPool: () => Micro.sync(() => {
            inUse.delete(connection)
            pool.push(connection)
          })
        }
      }
      
      if (inUse.size < maxConnections) {
        const connection = yield* Micro.tryPromise({
          try: () => createDbConnection(),
          catch: (error) => new Error(`Connection creation failed: ${error}`)
        })
        
        inUse.add(connection)
        
        return {
          connection,
          returnToPool: () => Micro.sync(() => {
            inUse.delete(connection)
            pool.push(connection)
          })
        }
      }
      
      return yield* Micro.fail(new Error("Connection pool exhausted"))
    })
  
  return { acquireConnection }
}

// Usage with automatic resource management
const withPooledConnection = <T, E>(
  pool: ReturnType<typeof createConnectionPool>,
  operation: (conn: DatabaseConnection) => Micro.Micro<T, E>
) =>
  Micro.acquireUseRelease(
    pool.acquireConnection(),
    ({ connection }) => operation(connection),
    ({ returnToPool }) => returnToPool()
  )
```

### Feature 2: MicroSchedule - Lightweight Scheduling

Custom scheduling policies without full Effect runtime:

#### Basic MicroSchedule Usage

```typescript
import { Micro, Option } from "effect"

// Custom schedule that backs off exponentially but caps at max delay
const cappedExponential = (
  initialDelay: number, 
  maxDelay: number
): Micro.MicroSchedule => 
  (attempt: number, elapsed: number) => {
    if (attempt > 10) return Option.none() // Stop after 10 attempts
    
    const delay = Math.min(initialDelay * Math.pow(2, attempt - 1), maxDelay)
    return Option.some(delay)
  }

// Usage
const retryWithCappedBackoff = <T, E>(effect: Micro.Micro<T, E>) =>
  effect.pipe(
    Micro.retry({ 
      schedule: cappedExponential(100, 5000) 
    })
  )
```

#### Real-World MicroSchedule Example

```typescript
import { Micro, Option } from "effect"

// Advanced scheduling for API rate limiting
interface RateLimitConfig {
  requestsPerMinute: number
  burstLimit: number
  backoffMultiplier: number
}

const createRateLimitSchedule = (config: RateLimitConfig): Micro.MicroSchedule => {
  const windowSize = 60_000 // 1 minute in milliseconds
  const requestInterval = windowSize / config.requestsPerMinute
  
  return (attempt: number, elapsed: number) => {
    // Allow burst initially
    if (attempt <= config.burstLimit) {
      return Option.some(0)
    }
    
    // After burst, enforce rate limiting
    const expectedDelay = requestInterval * config.backoffMultiplier
    const adjustedDelay = Math.min(expectedDelay * Math.pow(1.5, attempt - config.burstLimit), 30_000)
    
    // Stop retrying after 5 minutes
    if (elapsed > 300_000) {
      return Option.none()
    }
    
    return Option.some(adjustedDelay)
  }
}

// Circuit breaker pattern with scheduling
const createCircuitBreakerSchedule = (
  failureThreshold: number,
  recoveryTimeout: number
): Micro.MicroSchedule => {
  let consecutiveFailures = 0
  let lastFailureTime = 0
  
  return (attempt: number, elapsed: number) => {
    const now = Date.now()
    
    // If we've exceeded threshold, enter "open" state
    if (consecutiveFailures >= failureThreshold) {
      const timeSinceLastFailure = now - lastFailureTime
      
      // Stay open for recovery timeout
      if (timeSinceLastFailure < recoveryTimeout) {
        return Option.none() // Circuit is open, don't retry
      }
      
      // Try to close circuit (half-open state)
      consecutiveFailures = 0
      return Option.some(0)
    }
    
    // Normal exponential backoff
    if (attempt > 5) {
      consecutiveFailures += 1
      lastFailureTime = now
      return Option.none()
    }
    
    return Option.some(Math.min(1000 * Math.pow(2, attempt - 1), 10_000))
  }
}
```

### Feature 3: MicroCause - Structured Error Tracking

Simplified cause tracking for debugging and monitoring:

#### Basic MicroCause Usage

```typescript
import { Micro } from "effect"

const debugEffect = <T, E>(effect: Micro.Micro<T, E>) =>
  effect.pipe(
    Micro.tapErrorCause(cause => 
      Micro.sync(() => {
        switch (cause._tag) {
          case "Fail":
            console.error("Expected error:", cause.error)
            break
          case "Die": 
            console.error("Unexpected error:", cause.defect)
            break
          case "Interrupt":
            console.error("Operation interrupted")
            break
        }
      })
    )
  )
```

#### Real-World MicroCause Example

```typescript
import { Micro } from "effect"

interface ErrorMetrics {
  expectedErrors: number
  unexpectedErrors: number
  interruptions: number
  totalOperations: number
}

// Error tracking and metrics collection
const createErrorTracker = () => {
  const metrics: ErrorMetrics = {
    expectedErrors: 0,
    unexpectedErrors: 0,
    interruptions: 0,
    totalOperations: 0
  }
  
  const trackError = (cause: Micro.MicroCause<unknown>) =>
    Micro.sync(() => {
      metrics.totalOperations += 1
      
      switch (cause._tag) {
        case "Fail":
          metrics.expectedErrors += 1
          // Send to monitoring system
          sendMetric("error.expected", 1, {
            error_type: typeof cause.error === 'object' && cause.error && '_tag' in cause.error 
              ? (cause.error as any)._tag 
              : 'unknown'
          })
          break
          
        case "Die":
          metrics.unexpectedErrors += 1
          // Send to error tracking system  
          sendMetric("error.unexpected", 1, {
            error_message: String(cause.defect)
          })
          break
          
        case "Interrupt":
          metrics.interruptions += 1
          sendMetric("operation.interrupted", 1)
          break
      }
    })
  
  const withTracking = <T, E>(effect: Micro.Micro<T, E>) =>
    effect.pipe(
      Micro.tap(() => Micro.sync(() => metrics.totalOperations += 1)),
      Micro.tapErrorCause(trackError)
    )
  
  return { withTracking, getMetrics: () => metrics }
}

// Usage in production monitoring
const errorTracker = createErrorTracker()

const monitoredApiCall = (url: string) =>
  errorTracker.withTracking(
    Micro.tryPromise({
      try: () => fetch(url).then(r => r.json()),
      catch: (error) => new Error(`API call failed: ${error}`)
    }).pipe(
      Micro.timeout(5000),
      Micro.retry({ schedule: Micro.scheduleExponential(200) })
    )
  )

// Helper function for metrics (implementation would depend on your monitoring system)
const sendMetric = (name: string, value: number, tags?: Record<string, string>) => {
  // Implementation for your monitoring system (DataDog, Prometheus, etc.)
  console.log(`Metric: ${name} = ${value}`, tags)
}
```

## Practical Patterns & Best Practices

### Pattern 1: Error Boundary Pattern

```typescript
import { Micro } from "effect"

// Create reusable error boundaries for different domains
const createErrorBoundary = <E extends { _tag: string }>(
  errorName: string,
  fallback: unknown
) => <T, R>(effect: Micro.Micro<T, E, R>): Micro.Micro<T | typeof fallback, never, R> =>
  effect.pipe(
    Micro.catchAll(error => {
      console.error(`${errorName} boundary caught:`, error)
      
      // Send to monitoring
      sendErrorToMonitoring(errorName, error)
      
      return Micro.succeed(fallback)
    })
  )

// Domain-specific error boundaries
const withUserServiceBoundary = createErrorBoundary("UserService", null)
const withPaymentBoundary = createErrorBoundary("Payment", { status: "failed" })

// Usage
const safeUserLookup = (id: string) =>
  withUserServiceBoundary(
    userService.getUser(id)
  )

const safePaymentProcessing = (payment: PaymentRequest) =>
  withPaymentBoundary(
    paymentService.processPayment(payment)
  )

const sendErrorToMonitoring = (domain: string, error: unknown) => {
  // Implementation depends on monitoring system
  console.error(`[${domain}] Error:`, error)
}
```

### Pattern 2: Resource Pool Pattern

```typescript
import { Micro } from "effect"

interface Resource<T> {
  value: T
  isHealthy: () => boolean
  cleanup: () => Promise<void>
}

const createResourcePool = <T>(
  factory: () => Promise<T>,
  healthCheck: (resource: T) => boolean,
  maxSize: number = 10,
  idleTimeout: number = 30_000
) => {
  const available: Array<{ resource: T; lastUsed: number }> = []
  const inUse = new Set<T>()
  
  // Cleanup idle resources periodically
  setInterval(() => {
    const now = Date.now()
    for (let i = available.length - 1; i >= 0; i--) {
      const item = available[i]
      if (now - item.lastUsed > idleTimeout) {
        available.splice(i, 1)
        // Cleanup in background
        Promise.resolve().then(() => {
          if ('cleanup' in item.resource && typeof item.resource.cleanup === 'function') {
            return (item.resource as any).cleanup()
          }
        }).catch(console.error)
      }
    }
  }, idleTimeout / 2)
  
  const acquire = (): Micro.Micro<T, Error> =>
    Micro.gen(function* () {
      // Try to reuse healthy resource
      for (let i = 0; i < available.length; i++) {
        const item = available[i]
        if (healthCheck(item.resource)) {
          available.splice(i, 1)
          inUse.add(item.resource)
          return item.resource
        }
      }
      
      // Create new resource if under limit
      if (inUse.size < maxSize) {
        const resource = yield* Micro.tryPromise({
          try: factory,
          catch: (error) => new Error(`Resource creation failed: ${error}`)
        })
        
        inUse.add(resource)
        return resource
      }
      
      return yield* Micro.fail(new Error("Resource pool exhausted"))
    })
  
  const release = (resource: T) =>
    Micro.sync(() => {
      inUse.delete(resource)
      if (healthCheck(resource)) {
        available.push({ resource, lastUsed: Date.now() })
      }
    })
  
  const withResource = <TResult, E>(
    operation: (resource: T) => Micro.Micro<TResult, E>
  ) =>
    Micro.acquireUseRelease(
      acquire(),
      operation,
      release
    )
  
  return { withResource, acquire, release }
}

// Usage example
interface DbConnection {
  query: (sql: string) => Promise<unknown[]>
  isConnected: () => boolean
  close: () => Promise<void>
}

const dbPool = createResourcePool(
  () => createDbConnection(),
  (conn: DbConnection) => conn.isConnected(),
  5, // max 5 connections
  60_000 // 1 minute idle timeout
)

const queryWithPool = (sql: string) =>
  dbPool.withResource(connection =>
    Micro.tryPromise({
      try: () => connection.query(sql),
      catch: (error) => new Error(`Query failed: ${error}`)
    })
  )
```

### Pattern 3: Async Iterator Integration

```typescript
import { Micro } from "effect"

// Convert async iterables to Micro effects
const fromAsyncIterable = <T>(
  iterable: AsyncIterable<T>
): Micro.Micro<T[], Error> =>
  Micro.gen(function* () {
    const results: T[] = []
    
    const iterator = iterable[Symbol.asyncIterator]()
    
    while (true) {
      const { done, value } = yield* Micro.tryPromise({
        try: () => iterator.next(),
        catch: (error) => new Error(`Iterator failed: ${error}`)
      })
      
      if (done) break
      results.push(value)
    }
    
    return results
  })

// Process stream with backpressure
const processStream = <T, R>(
  stream: AsyncIterable<T>,
  processor: (item: T) => Micro.Micro<R, Error>,
  options: { concurrency?: number; batchSize?: number } = {}
) => {
  const { concurrency = 1, batchSize = 10 } = options
  
  return Micro.gen(function* () {
    const iterator = stream[Symbol.asyncIterator]()
    const results: R[] = []
    
    while (true) {
      // Collect batch
      const batch: T[] = []
      for (let i = 0; i < batchSize; i++) {
        const { done, value } = yield* Micro.tryPromise({
          try: () => iterator.next(),
          catch: (error) => new Error(`Stream read failed: ${error}`)
        })
        
        if (done) break
        batch.push(value)
      }
      
      if (batch.length === 0) break
      
      // Process batch with concurrency control
      const batchResults = yield* Micro.forEach(
        batch,
        processor,
        { concurrency }
      )
      
      results.push(...batchResults)
    }
    
    return results
  })
}

// Usage with real streams
async function* generateNumbers(count: number): AsyncGenerator<number> {
  for (let i = 0; i < count; i++) {
    await new Promise(resolve => setTimeout(resolve, 10))
    yield i
  }
}

const processNumber = (n: number) =>
  Micro.sync(() => n * 2).pipe(
    Micro.delay(Math.random() * 100) // Simulate variable processing time
  )

const processedStream = processStream(
  generateNumbers(100),
  processNumber,
  { concurrency: 5, batchSize: 10 }
)
```

## Integration Examples

### Integration with React

```typescript
import { Micro } from "effect"
import { useEffect, useState } from "react"

// Custom hook for Micro effects
function useMicro<T, E>(
  effect: Micro.Micro<T, E>,
  deps: React.DependencyList = []
) {
  const [state, setState] = useState<{
    data: T | null
    error: E | null
    loading: boolean
  }>({ data: null, error: null, loading: true })
  
  useEffect(() => {
    setState(prev => ({ ...prev, loading: true }))
    
    const fiber = Micro.runFork(effect)
    
    fiber.addObserver(exit => {
      if (exit._tag === "Success") {
        setState({ data: exit.value, error: null, loading: false })
      } else {
        const error = exit.cause._tag === "Fail" ? exit.cause.error : 
                     exit.cause._tag === "Die" ? exit.cause.defect :
                     "Operation interrupted"
        setState({ data: null, error: error as E, loading: false })
      }
    })
    
    return () => {
      Micro.fiberInterrupt(fiber)
    }
  }, deps)
  
  return state
}

// Usage in React components
interface User {
  id: string
  name: string
  email: string
}

const UserProfile: React.FC<{ userId: string }> = ({ userId }) => {
  const fetchUser = (id: string) =>
    Micro.tryPromise({
      try: () => fetch(`/api/users/${id}`).then(r => r.json()),
      catch: (error) => new Error(`Failed to fetch user: ${error}`)
    }).pipe(
      Micro.timeout(5000),
      Micro.retry({ schedule: Micro.scheduleExponential(500) })
    )
  
  const { data: user, error, loading } = useMicro(
    fetchUser(userId),
    [userId]
  )
  
  if (loading) return <div>Loading...</div>
  if (error) return <div>Error: {String(error)}</div>
  if (!user) return <div>No user found</div>
  
  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  )
}
```

### Testing Strategies

```typescript
import { Micro } from "effect"
import { describe, it, expect, vi } from "vitest"

// Mock service for testing
class MockUserService {
  private users = new Map<string, User>()
  private shouldFail = false
  
  setUsers(users: User[]) {
    this.users.clear()
    users.forEach(user => this.users.set(user.id, user))
  }
  
  setShouldFail(shouldFail: boolean) {
    this.shouldFail = shouldFail
  }
  
  getUser(id: string): Micro.Micro<User, UserNotFoundError | DbError> {
    if (this.shouldFail) {
      return Micro.fail(new DbError({
        operation: 'getUser',
        details: 'Mock failure'
      }))
    }
    
    const user = this.users.get(id)
    return user 
      ? Micro.succeed(user)
      : Micro.fail(new UserNotFoundError({ userId: id }))
  }
}

describe("User Service", () => {
  it("should return user when found", async () => {
    const mockService = new MockUserService()
    const testUser = { id: "1", name: "John", email: "john@example.com" }
    
    mockService.setUsers([testUser])
    
    const result = await Micro.runPromise(mockService.getUser("1"))
    expect(result).toEqual(testUser)
  })
  
  it("should handle user not found", async () => {
    const mockService = new MockUserService()
    mockService.setUsers([])
    
    const result = await Micro.runPromiseExit(mockService.getUser("999"))
    
    expect(result._tag).toBe("Failure")
    if (result._tag === "Failure" && result.cause._tag === "Fail") {
      expect(result.cause.error).toBeInstanceOf(UserNotFoundError)
      expect(result.cause.error.userId).toBe("999")
    }
  })
  
  it("should retry on database errors", async () => {
    const mockService = new MockUserService()
    const testUser = { id: "1", name: "John", email: "john@example.com" }
    
    mockService.setUsers([testUser])
    mockService.setShouldFail(true)
    
    // Mock that fails twice then succeeds
    let callCount = 0
    const originalGetUser = mockService.getUser.bind(mockService)
    vi.spyOn(mockService, 'getUser').mockImplementation((id) => {
      callCount++
      if (callCount <= 2) {
        return Micro.fail(new DbError({
          operation: 'getUser',
          details: 'Temporary failure'
        }))
      }
      mockService.setShouldFail(false)
      return originalGetUser(id)
    })
    
    const effect = mockService.getUser("1").pipe(
      Micro.retry({ 
        schedule: Micro.scheduleRecurs(3)
      })
    )
    
    const result = await Micro.runPromise(effect)
    expect(result).toEqual(testUser)
    expect(callCount).toBe(3)
  })
  
  it("should timeout long-running operations", async () => {
    const slowEffect = Micro.gen(function* () {
      yield* Micro.sleep(2000) // 2 second delay
      return "result"
    })
    
    const timedEffect = slowEffect.pipe(
      Micro.timeout(100) // 100ms timeout
    )
    
    const result = await Micro.runPromiseExit(timedEffect)
    
    expect(result._tag).toBe("Failure")
    if (result._tag === "Failure" && result.cause._tag === "Fail") {
      expect(result.cause.error._tag).toBe("TimeoutException")
    }
  })
})

// Property-based testing with Micro
describe("Data Processing Pipeline", () => {
  it("should process all valid events", async () => {
    // Generate test data
    const validEvents: MetricEvent[] = Array.from({ length: 100 }, (_, i) => ({
      timestamp: Date.now() - i * 1000,
      type: ['counter', 'gauge', 'histogram'][i % 3],
      value: Math.random() * 100,
      tags: { service: 'test', instance: `${i % 5}` }
    }))
    
    const result = await Micro.runPromise(processMetrics(validEvents))
    
    expect(result).toHaveLength(100)
    result.forEach(processed => {
      expect(processed).toHaveProperty('aggregatedValue')
      expect(processed).toHaveProperty('normalizedTags')
      expect(processed).toHaveProperty('bucket')  
      expect(processed).toHaveProperty('processed_at')
    })
  })
  
  it("should handle mixed valid/invalid events", async () => {
    const mixedEvents: MetricEvent[] = [
      { timestamp: Date.now(), type: 'counter', value: 10, tags: { service: 'test' }},
      { timestamp: -1, type: 'counter', value: 20, tags: { service: 'test' }}, // Invalid timestamp
      { timestamp: Date.now(), type: 'gauge', value: NaN, tags: { service: 'test' }}, // Invalid value
      { timestamp: Date.now(), type: 'histogram', value: 30, tags: { service: 'test' }}
    ]
    
    const result = await Micro.runPromise(processMetrics(mixedEvents))
    
    // Should process valid events and skip invalid ones
    expect(result).toHaveLength(2)
  })
})
```

## Conclusion

Micro provides Effect's composability and type safety for lightweight use cases where bundle size matters. With starting at **5kb gzipped**, it enables libraries and applications to adopt Effect patterns without runtime overhead.

Key benefits:
- **Minimal Bundle Impact**: Fraction of full Effect runtime size
- **Effect Compatibility**: Same patterns and mental model as Effect
- **Production Ready**: Structured error handling, resource management, and scheduling

Use Micro when you need Effect-like capabilities in bundle-size-sensitive environments: client-side libraries, serverless functions, or applications where every kilobyte counts. For complex scenarios requiring Layer, Ref, Queue, or Deferred, stick with the full Effect runtime.