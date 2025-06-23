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

When building domain-specific types that need to integrate seamlessly with the Effect ecosystem, developers often face a challenge: their custom classes can't be used directly in Effect workflows without explicit conversion. This leads to verbose, error-prone code that breaks the natural flow of Effect operations.

```typescript
// Traditional approach - manual Effect wrapping
class DatabaseConnection {
  connect(): Promise<void> { /* ... */ }
  query(sql: string): Promise<Row[]> { /* ... */ }
}

// Usage requires manual wrapping everywhere
const workflow = Effect.gen(function* () {
  const db = new DatabaseConnection()
  const connectEffect = Effect.fromPromise(() => db.connect()) // Manual wrapping
  yield* connectEffect
  const queryEffect = Effect.fromPromise(() => db.query("SELECT * FROM users")) // More wrapping
  return yield* queryEffect
})
```

This approach leads to:
- **Boilerplate Overhead** - Every method call requires manual Effect wrapping
- **Inconsistent APIs** - Some operations return Effects, others return Promises
- **Lost Type Safety** - Manual wrapping can lose important type information
- **Poor Composability** - Objects don't integrate naturally with Effect pipelines

### The Effectable Solution

Effectable provides base classes that make custom types behave like native Effect values, enabling seamless integration with the Effect ecosystem while maintaining full type safety and composability.

```typescript
import { Effectable, Effect } from "effect"

class DatabaseConnection extends Effectable.Class<void> {
  commit() {
    return this.connect()
  }
  
  private connect(): Effect.Effect<void> {
    return Effect.fromPromise(() => this.client.connect())
  }
  
  query(sql: string): Effect.Effect<Row[]> {
    return Effect.fromPromise(() => this.client.query(sql))
  }
}

// Usage is natural and composable
const workflow = Effect.gen(function* () {
  const db = new DatabaseConnection()
  yield* db // Automatically becomes an Effect!
  return yield* db.query("SELECT * FROM users")
}).pipe(
  Effect.withSpan("database.workflow")
)
```

### Key Concepts

**Effectable.Class**: Base class that makes instances behave like Effect values with identity-based equality

**Effectable.StructuralClass**: Base class with structural equality for immutable value types

**commit() method**: The core method that defines what Effect the instance represents when used in Effect workflows

## Basic Usage Patterns

### Pattern 1: Resource-Based Effectable

```typescript
import { Effectable, Effect, Context } from "effect"

// A connection that represents its initialization as an Effect
class HttpClient extends Effectable.Class<HttpClient> {
  constructor(private config: ClientConfig) {
    super()
  }
  
  // The commit method defines what happens when this instance is yielded
  commit(): Effect.Effect<HttpClient> {
    return Effect.gen(function* () {
      yield* Effect.log(`Initializing HTTP client with ${this.config.baseUrl}`)
      yield* this.validateConfig()
      return this
    }.bind(this))
  }
  
  private validateConfig(): Effect.Effect<void> {
    return this.config.apiKey 
      ? Effect.succeed(undefined)
      : Effect.fail(new Error("API key required"))
  }
  
  get(path: string): Effect.Effect<Response> {
    return Effect.fromPromise(() => 
      fetch(`${this.config.baseUrl}${path}`, {
        headers: { Authorization: `Bearer ${this.config.apiKey}` }
      })
    )
  }
}

// Usage - the instance can be yielded directly
const workflow = Effect.gen(function* () {
  const client = new HttpClient({ baseUrl: "https://api.example.com", apiKey: "key" })
  const initializedClient = yield* client // commit() is called automatically
  return yield* initializedClient.get("/users")
})
```

### Pattern 2: Command-Style Effectable

```typescript
import { Effectable, Effect } from "effect"

// Commands that represent their execution as Effects
class SendEmail extends Effectable.StructuralClass<void> {
  constructor(
    public readonly to: string,
    public readonly subject: string,
    public readonly body: string
  ) {
    super()
  }
  
  commit(): Effect.Effect<void> {
    return Effect.gen(function* () {
      yield* Effect.log(`Sending email to ${this.to}`)
      yield* Effect.sleep("100 millis") // Simulate sending
      yield* Effect.log(`Email sent successfully`)
    })
  }
}

class CreateUser extends Effectable.StructuralClass<User> {
  constructor(public readonly userData: UserData) {
    super()
  }
  
  commit(): Effect.Effect<User> {
    return Effect.gen(function* () {
      const validatedData = yield* this.validate()
      const user = yield* this.saveToDatabase(validatedData)
      yield* new SendEmail(
        user.email, 
        "Welcome!", 
        "Thanks for joining us!"
      ) // Nested Effectable usage
      return user
    })
  }
  
  private validate(): Effect.Effect<UserData> {
    return this.userData.email.includes("@")
      ? Effect.succeed(this.userData)
      : Effect.fail(new Error("Invalid email"))
  }
  
  private saveToDatabase(data: UserData): Effect.Effect<User> {
    return Effect.succeed({ id: crypto.randomUUID(), ...data })
  }
}
```

### Pattern 3: Service-Style Effectable

```typescript
import { Effectable, Effect, Context } from "effect"

interface DatabaseConfig {
  connectionString: string
  maxConnections: number
}

class DatabaseService extends Effectable.Class<DatabaseService> {
  constructor(private config: DatabaseConfig) {
    super()
  }
  
  commit(): Effect.Effect<DatabaseService> {
    return Effect.gen(function* () {
      yield* Effect.log("Connecting to database...")
      // Simulate connection setup
      yield* Effect.sleep("500 millis")
      yield* Effect.log("Database connected successfully")
      return this
    })
  }
  
  findUser(id: string): Effect.Effect<User | null> {
    return Effect.succeed({ id, name: "John Doe", email: "john@example.com" })
  }
  
  createUser(userData: UserData): Effect.Effect<User> {
    return Effect.succeed({ id: crypto.randomUUID(), ...userData })
  }
}

// Create as a service
const DatabaseLive = Context.GenericTag<DatabaseService>("@app/Database")
  .of(new DatabaseService({ 
    connectionString: "postgresql://localhost:5432/mydb",
    maxConnections: 10 
  }))
```

## Real-World Examples

### Example 1: Task Queue with Retry Logic

A task queue that automatically handles retry logic and failure tracking.

```typescript
import { Effectable, Effect, Schedule, Cause } from "effect"

interface TaskResult<A> {
  success: boolean
  result?: A
  error?: unknown
  attempts: number
}

class RetryableTask<A, E> extends Effectable.Class<TaskResult<A>> {
  constructor(
    private task: Effect.Effect<A, E>,
    private maxRetries: number = 3,
    private taskName: string = "unknown"
  ) {
    super()
  }
  
  commit(): Effect.Effect<TaskResult<A>> {
    return Effect.gen(function* () {
      yield* Effect.log(`Starting task: ${this.taskName}`)
      
      const result = yield* this.task.pipe(
        Effect.retry(Schedule.exponential("100 millis").pipe(
          Schedule.compose(Schedule.recurs(this.maxRetries))
        )),
        Effect.either,
        Effect.map((either) => either._tag === "Right" 
          ? { success: true, result: either.value, attempts: 1 }
          : { success: false, error: either.left, attempts: this.maxRetries + 1 }
        )
      )
      
      if (result.success) {
        yield* Effect.log(`Task ${this.taskName} completed successfully`)
      } else {
        yield* Effect.logError(`Task ${this.taskName} failed after ${result.attempts} attempts`)
      }
      
      return result
    }.bind(this))
  }
}

// Usage in a workflow
const processOrders = Effect.gen(function* () {
  const fetchOrders = new RetryableTask(
    Effect.fromPromise(() => fetch("/api/orders").then(r => r.json())),
    5,
    "fetch-orders"
  )
  
  const enrichOrders = new RetryableTask(
    Effect.fromPromise(() => enrichOrderData(orders)),
    3,
    "enrich-orders"  
  )
  
  const ordersResult = yield* fetchOrders
  if (!ordersResult.success) {
    return Effect.fail("Could not fetch orders")
  }
  
  const enrichedResult = yield* enrichOrders
  return { orders: ordersResult.result, enriched: enrichedResult.success }
})
```

### Example 2: Configuration Manager with Hot Reloading

A configuration system that can reload itself and notify dependent services.

```typescript
import { Effectable, Effect, Ref, Fiber, Schedule } from "effect"

interface AppConfig {
  database: { url: string; poolSize: number }
  api: { port: number; rateLimit: number }
  features: { enableNewUI: boolean; enableAnalytics: boolean }
}

class ConfigManager extends Effectable.Class<AppConfig> {
  private configRef: Ref.Ref<AppConfig>
  private watcherFiber: Fiber.Fiber<never>
  
  constructor(private configPath: string) {
    super()
  }
  
  commit(): Effect.Effect<AppConfig> {
    return Effect.gen(function* () {
      // Load initial configuration
      const initialConfig = yield* this.loadConfig()
      this.configRef = yield* Ref.make(initialConfig)
      
      // Start file watcher for hot reloading
      this.watcherFiber = yield* this.startConfigWatcher().pipe(
        Effect.fork
      )
      
      yield* Effect.log(`Configuration loaded from ${this.configPath}`)
      return initialConfig
    }.bind(this))
  }
  
  private loadConfig(): Effect.Effect<AppConfig> {
    return Effect.fromPromise(async () => {
      const content = await import(this.configPath)
      return content.default as AppConfig
    }).pipe(
      Effect.catchAll(() => Effect.succeed({
        database: { url: "postgresql://localhost:5432/app", poolSize: 10 },
        api: { port: 3000, rateLimit: 100 },
        features: { enableNewUI: false, enableAnalytics: true }
      }))
    )
  }
  
  private startConfigWatcher(): Effect.Effect<never> {
    return Effect.gen(function* () {
      yield* Effect.repeat(
        Effect.gen(function* () {
          const newConfig = yield* this.loadConfig()
          const currentConfig = yield* Ref.get(this.configRef)
          
          if (JSON.stringify(newConfig) !== JSON.stringify(currentConfig)) {
            yield* Ref.set(this.configRef, newConfig)
            yield* Effect.log("Configuration reloaded")
          }
        }.bind(this)),
        Schedule.fixed("5 seconds")
      )
    }.bind(this))
  }
  
  getCurrentConfig(): Effect.Effect<AppConfig> {
    return Ref.get(this.configRef)
  }
  
  updateFeatureFlag(flag: keyof AppConfig["features"], enabled: boolean): Effect.Effect<void> {
    return Ref.update(this.configRef, config => ({
      ...config,
      features: { ...config.features, [flag]: enabled }
    }))
  }
}

// Usage in application setup
const startApplication = Effect.gen(function* () {
  const config = new ConfigManager("./config/app.json")
  const appConfig = yield* config // Loads and starts watching
  
  yield* Effect.log(`Starting server on port ${appConfig.api.port}`)
  
  // Use config throughout the application
  const currentConfig = yield* config.getCurrentConfig()
  if (currentConfig.features.enableAnalytics) {
    yield* Effect.log("Analytics enabled")
  }
})
```

### Example 3: Distributed Lock with Automatic Cleanup

A distributed lock implementation that ensures proper cleanup even on failures.

```typescript
import { Effectable, Effect, Schedule, Clock, Duration } from "effect"

interface LockOptions {
  ttl: Duration.Duration
  retryInterval: Duration.Duration  
  maxRetries: number
}

class DistributedLock extends Effectable.Class<DistributedLock> {
  private lockId: string
  private isLocked = false
  
  constructor(
    private lockKey: string,
    private redisClient: RedisClient,
    private options: LockOptions = {
      ttl: Duration.seconds(30),
      retryInterval: Duration.millis(100),
      maxRetries: 50
    }
  ) {
    super()
    this.lockId = `${lockKey}:${crypto.randomUUID()}`
  }
  
  commit(): Effect.Effect<DistributedLock> {
    return Effect.gen(function* () {
      // Attempt to acquire the lock with retries
      const acquired = yield* this.acquireLock().pipe(
        Effect.retry(Schedule.spaced(this.options.retryInterval).pipe(
          Schedule.compose(Schedule.recurs(this.options.maxRetries))
        )),
        Effect.either
      )
      
      if (acquired._tag === "Left") {
        return yield* Effect.fail(new Error(`Could not acquire lock: ${this.lockKey}`))
      }
      
      this.isLocked = true
      yield* Effect.log(`Acquired distributed lock: ${this.lockKey}`)
      
      // Set up automatic cleanup
      yield* this.scheduleCleanup().pipe(Effect.fork)
      
      return this
    }.bind(this))
  }
  
  private acquireLock(): Effect.Effect<boolean> {
    return Effect.fromPromise(() => 
      this.redisClient.set(
        `lock:${this.lockKey}`, 
        this.lockId, 
        "PX", 
        Duration.toMillis(this.options.ttl),
        "NX"
      )
    ).pipe(
      Effect.map(result => result === "OK")
    )
  }
  
  private scheduleCleanup(): Effect.Effect<void> {
    return Effect.gen(function* () {
      // Wait for TTL - 5 seconds, then refresh or cleanup
      yield* Effect.sleep(Duration.subtract(this.options.ttl, Duration.seconds(5)))
      
      if (this.isLocked) {
        const extended = yield* this.extendLock()
        if (extended) {
          yield* this.scheduleCleanup() // Recursively schedule next cleanup
        }
      }
    }.bind(this))
  }
  
  private extendLock(): Effect.Effect<boolean> {
    const script = `
      if redis.call("GET", KEYS[1]) == ARGV[1] then
        return redis.call("PEXPIRE", KEYS[1], ARGV[2])
      else
        return 0
      end
    `
    
    return Effect.fromPromise(() => 
      this.redisClient.eval(script, 1, `lock:${this.lockKey}`, this.lockId, Duration.toMillis(this.options.ttl))
    ).pipe(
      Effect.map(result => result === 1)
    )
  }
  
  release(): Effect.Effect<void> {
    return Effect.gen(function* () {
      if (!this.isLocked) return
      
      const script = `
        if redis.call("GET", KEYS[1]) == ARGV[1] then
          return redis.call("DEL", KEYS[1])
        else
          return 0
        end
      `
      
      yield* Effect.fromPromise(() => 
        this.redisClient.eval(script, 1, `lock:${this.lockKey}`, this.lockId)
      )
      
      this.isLocked = false
      yield* Effect.log(`Released distributed lock: ${this.lockKey}`)
    }.bind(this))
  }
}

// Usage with automatic cleanup via Effect.ensuring
const criticalSection = Effect.gen(function* () {
  const lock = new DistributedLock("user:123:update", redisClient)
  
  return yield* Effect.gen(function* () {
    const acquiredLock = yield* lock
    
    // Perform critical operations
    yield* Effect.log("Performing critical database updates...")
    yield* Effect.sleep("2 seconds")
    yield* Effect.log("Critical operations completed")
    
    return "success"
  }).pipe(
    Effect.ensuring(lock.release()) // Always cleanup, even on failure
  )
})
```

## Advanced Features Deep Dive

### Feature 1: Structural vs Identity Equality

Effectable provides two base classes with different equality semantics, crucial for proper Effect behavior.

#### Basic Equality Usage

```typescript
import { Effectable, Effect, Equal } from "effect"

// Identity-based equality (default)
class Connection extends Effectable.Class<string> {
  constructor(private url: string) { super() }
  commit() { return Effect.succeed(this.url) }
}

// Structural equality for value types
class ConfigValue extends Effectable.StructuralClass<string> {
  constructor(private value: string) { super() }
  commit() { return Effect.succeed(this.value) }
}

// Demonstrate the difference
const conn1 = new Connection("localhost")
const conn2 = new Connection("localhost")

const config1 = new ConfigValue("debug")
const config2 = new ConfigValue("debug")

console.log(Equal.equals(conn1, conn2))   // false - different instances
console.log(Equal.equals(config1, config2)) // true - same structure
```

#### Real-World Equality Example

```typescript
// Identity-based: Each database connection is unique
class DatabaseConnection extends Effectable.Class<DatabaseConnection> {
  constructor(
    private pool: ConnectionPool,
    private connectionId: string
  ) {
    super()
  }
  
  commit(): Effect.Effect<DatabaseConnection> {
    return Effect.gen(function* () {
      yield* Effect.log(`Establishing connection ${this.connectionId}`)
      return this
    })
  }
}

// Structural: HTTP requests with same parameters are equivalent
class HttpRequest extends Effectable.StructuralClass<Response> {
  constructor(
    public readonly method: string,
    public readonly url: string,
    public readonly headers: Record<string, string> = {}
  ) {
    super()
  }
  
  commit(): Effect.Effect<Response> {
    return Effect.fromPromise(() => 
      fetch(this.url, { method: this.method, headers: this.headers })
    )
  }
}

// Usage with caching - structural equality enables effective caching
const requestCache = new Map<HttpRequest, Effect.Effect<Response>>()

const cachedRequest = (req: HttpRequest): Effect.Effect<Response> => {
  // Thanks to structural equality, same requests will have same hash
  const cached = requestCache.get(req)
  if (cached) {
    return cached.pipe(Effect.tap(() => Effect.log("Cache hit!")))
  }
  
  const effect = req.commit().pipe(
    Effect.tap(() => Effect.log("Cache miss - executing request"))
  )
  requestCache.set(req, effect)
  return effect
}
```

### Feature 2: Advanced commit() Patterns

The commit method can implement sophisticated initialization and resource management patterns.

#### Lazy Initialization Pattern

```typescript
class LazyService extends Effectable.Class<LazyService> {
  private initialized = false
  private initializationPromise?: Promise<void>
  
  constructor(private config: ServiceConfig) {
    super()
  }
  
  commit(): Effect.Effect<LazyService> {
    return Effect.gen(function* () {
      if (this.initialized) {
        return this
      }
      
      // Ensure single initialization even with concurrent access
      if (!this.initializationPromise) {
        this.initializationPromise = this.performInitialization()
      }
      
      yield* Effect.fromPromise(() => this.initializationPromise!)
      this.initialized = true
      
      return this
    }.bind(this))
  }
  
  private async performInitialization(): Promise<void> {
    // Expensive initialization logic
    await new Promise(resolve => setTimeout(resolve, 1000))
  }
}
```

#### Conditional Initialization Pattern

```typescript
class ConditionalResource extends Effectable.Class<ConditionalResource> {
  constructor(
    private condition: () => boolean,
    private onTrue: Effect.Effect<void>,
    private onFalse: Effect.Effect<void>
  ) {
    super()
  }
  
  commit(): Effect.Effect<ConditionalResource> {
    return Effect.gen(function* () {
      const shouldInitialize = this.condition()
      
      if (shouldInitialize) {
        yield* this.onTrue
        yield* Effect.log("Resource initialized via true path")
      } else {
        yield* this.onFalse  
        yield* Effect.log("Resource initialized via false path")
      }
      
      return this
    }.bind(this))
  }
}

// Usage - initialization depends on runtime conditions
const conditionalSetup = new ConditionalResource(
  () => process.env.NODE_ENV === "production",
  Effect.log("Setting up production resources"),
  Effect.log("Setting up development resources")
)
```

## Practical Patterns & Best Practices

### Pattern 1: Builder Pattern with Effectable

Create fluent APIs that maintain Effect compatibility throughout the building process.

```typescript
class QueryBuilder extends Effectable.StructuralClass<string> {
  constructor(
    private tableName: string,
    private selectFields: string[] = ["*"],
    private whereConditions: string[] = [],
    private joinClauses: string[] = [],
    private orderByFields: string[] = [],
    private limitValue?: number
  ) {
    super()
  }
  
  select(...fields: string[]): QueryBuilder {
    return new QueryBuilder(
      this.tableName,
      fields,
      this.whereConditions,
      this.joinClauses,
      this.orderByFields,
      this.limitValue
    )
  }
  
  where(condition: string): QueryBuilder {
    return new QueryBuilder(
      this.tableName,
      this.selectFields,
      [...this.whereConditions, condition],
      this.joinClauses,
      this.orderByFields,
      this.limitValue
    )
  }
  
  join(table: string, on: string): QueryBuilder {
    return new QueryBuilder(
      this.tableName,
      this.selectFields,
      this.whereConditions,
      [...this.joinClauses, `JOIN ${table} ON ${on}`],
      this.orderByFields,
      this.limitValue
    )
  }
  
  orderBy(field: string): QueryBuilder {
    return new QueryBuilder(
      this.tableName,
      this.selectFields,
      this.whereConditions,
      this.joinClauses,
      [...this.orderByFields, field],
      this.limitValue
    )
  }
  
  limit(count: number): QueryBuilder {
    return new QueryBuilder(
      this.tableName,
      this.selectFields,
      this.whereConditions,
      this.joinClauses,
      this.orderByFields,
      count
    )
  }
  
  commit(): Effect.Effect<string> {
    return Effect.succeed(this.buildQuery())
  }
  
  private buildQuery(): string {
    let query = `SELECT ${this.selectFields.join(", ")} FROM ${this.tableName}`
    
    if (this.joinClauses.length > 0) {
      query += ` ${this.joinClauses.join(" ")}`
    }
    
    if (this.whereConditions.length > 0) {
      query += ` WHERE ${this.whereConditions.join(" AND ")}`
    }
    
    if (this.orderByFields.length > 0) {
      query += ` ORDER BY ${this.orderByFields.join(", ")}`
    }
    
    if (this.limitValue !== undefined) {
      query += ` LIMIT ${this.limitValue}`
    }
    
    return query
  }
}

// Helper for starting queries
const query = (table: string) => new QueryBuilder(table)

// Usage - fluent API that can be yielded at any point
const userQuery = Effect.gen(function* () {
  const sqlQuery = yield* query("users")
    .select("id", "name", "email")
    .join("profiles", "users.id = profiles.user_id")
    .where("users.active = true")
    .where("profiles.verified = true")
    .orderBy("users.created_at")
    .limit(50) // This can be yielded to get the SQL string
  
  yield* Effect.log(`Generated query: ${sqlQuery}`)
  return sqlQuery
})
```

### Pattern 2: Resource Pool Management

Manage pools of resources with automatic cleanup and health checking.

```typescript
interface PooledResource {
  id: string
  isHealthy(): Promise<boolean>
  cleanup(): Promise<void>
}

class ResourcePool<T extends PooledResource> extends Effectable.Class<T> {
  private available: T[] = []
  private inUse = new Set<T>()
  private healthCheckInterval?: NodeJS.Timeout
  
  constructor(
    private createResource: () => Promise<T>,
    private maxSize: number = 10,
    private healthCheckIntervalMs: number = 30000
  ) {
    super()
  }
  
  commit(): Effect.Effect<T> {
    return Effect.gen(function* () {
      // Initialize pool if needed
      if (!this.healthCheckInterval) {
        yield* this.initializePool()
      }
      
      // Get or create resource
      const resource = yield* this.acquireResource()
      
      // Set up automatic return to pool
      yield* Effect.addFinalizer(() => this.releaseResource(resource))
      
      return resource
    }.bind(this))
  }
  
  private initializePool(): Effect.Effect<void> {
    return Effect.gen(function* () {
      // Start health checking
      this.healthCheckInterval = setInterval(() => {
        this.performHealthCheck().pipe(Effect.runFork)
      }, this.healthCheckIntervalMs)
      
      // Pre-populate pool
      yield* Effect.forEach(
        Array.from({ length: Math.min(3, this.maxSize) }, (_, i) => i),
        () => Effect.fromPromise(() => this.createResource()).pipe(
          Effect.map(resource => this.available.push(resource)),
          Effect.catchAll(() => Effect.succeed(undefined))
        ),
        { concurrency: "unbounded" }
      )
      
      yield* Effect.log(`Resource pool initialized with ${this.available.length} resources`)
    }.bind(this))
  }
  
  private acquireResource(): Effect.Effect<T> {
    return Effect.gen(function* () {
      // Try to get from available pool
      if (this.available.length > 0) {
        const resource = this.available.pop()!
        this.inUse.add(resource)
        return resource
      }
      
      // Create new if under max size
      if (this.inUse.size < this.maxSize) {
        const resource = yield* Effect.fromPromise(() => this.createResource())
        this.inUse.add(resource)
        return resource
      }
      
      // Wait for resource to become available
      return yield* Effect.gen(function* () {
        yield* Effect.sleep("100 millis")
        return yield* this.acquireResource()
      }.bind(this))
    }.bind(this))
  }
  
  private releaseResource(resource: T): Effect.Effect<void> {
    return Effect.gen(function* () {
      this.inUse.delete(resource)
      
      // Check if resource is still healthy
      const healthy = yield* Effect.fromPromise(() => resource.isHealthy()).pipe(
        Effect.catchAll(() => Effect.succeed(false))
      )
      
      if (healthy) {
        this.available.push(resource)
      } else {
        yield* Effect.fromPromise(() => resource.cleanup()).pipe(
          Effect.catchAll(() => Effect.succeed(undefined))
        )
      }
    }.bind(this))
  }
  
  private performHealthCheck(): Effect.Effect<void> {
    return Effect.gen(function* () {
      // Check available resources
      const healthyResources: T[] = []
      
      yield* Effect.forEach(
        this.available,
        (resource) => Effect.gen(function* () {
          const healthy = yield* Effect.fromPromise(() => resource.isHealthy()).pipe(
            Effect.catchAll(() => Effect.succeed(false))
          )
          
          if (healthy) {
            healthyResources.push(resource)
          } else {
            yield* Effect.fromPromise(() => resource.cleanup()).pipe(
              Effect.catchAll(() => Effect.succeed(undefined))
            )
          }
        }),
        { concurrency: "unbounded" }
      )
      
      this.available = healthyResources
    }.bind(this))
  }
  
  shutdown(): Effect.Effect<void> {
    return Effect.gen(function* () {
      if (this.healthCheckInterval) {
        clearInterval(this.healthCheckInterval)
      }
      
      const allResources = [...this.available, ...this.inUse]
      
      yield* Effect.forEach(
        allResources,
        (resource) => Effect.fromPromise(() => resource.cleanup()).pipe(
          Effect.catchAll(() => Effect.succeed(undefined))
        ),
        { concurrency: "unbounded" }
      )
      
      this.available = []
      this.inUse.clear()
      
      yield* Effect.log("Resource pool shut down")
    }.bind(this))
  }
}

// Usage with scoped resource management
const withDatabaseConnection = <A>(
  work: (db: DatabaseConnection) => Effect.Effect<A>
): Effect.Effect<A> => {
  const pool = new ResourcePool(
    () => createDatabaseConnection(),
    5, // max 5 connections
    30000 // health check every 30s
  )
  
  return Effect.gen(function* () {
    const connection = yield* pool
    return yield* work(connection) 
  }).pipe(
    Effect.scoped // Automatic cleanup
  )
}
```

## Integration Examples

### Integration with Effect Services

Create Effectable types that work seamlessly with Effect's service system.

```typescript
import { Effectable, Effect, Context, Layer } from "effect"

// Service interface
interface EmailService {
  readonly send: (to: string, subject: string, body: string) => Effect.Effect<void>
  readonly sendBulk: (emails: EmailData[]) => Effect.Effect<void>
}

const EmailService = Context.GenericTag<EmailService>("@app/EmailService")

// Effectable implementation that can be used as a service
class SmtpEmailService extends Effectable.Class<SmtpEmailService> implements EmailService {
  constructor(private config: SmtpConfig) {
    super()
  }
  
  commit(): Effect.Effect<SmtpEmailService> {
    return Effect.gen(function* () {
      yield* Effect.log("Initializing SMTP connection...")
      yield* this.connectToSmtp()
      yield* Effect.log("SMTP service ready")
      return this
    }.bind(this))
  }
  
  private connectToSmtp(): Effect.Effect<void> {
    return Effect.fromPromise(async () => {
      // SMTP connection logic
      await new Promise(resolve => setTimeout(resolve, 500))
    })
  }
  
  send(to: string, subject: string, body: string): Effect.Effect<void> {
    return Effect.gen(function* () {
      yield* Effect.log(`Sending email to ${to}: ${subject}`)
      yield* Effect.sleep("100 millis") // Simulate sending
    })
  }
  
  sendBulk(emails: EmailData[]): Effect.Effect<void> {
    return Effect.forEach(
      emails,
      (email) => this.send(email.to, email.subject, email.body),
      { concurrency: 5 }
    ).pipe(
      Effect.asVoid
    )
  }
}

// Create service layer that initializes the Effectable
const EmailServiceLive = Layer.effect(
  EmailService,
  Effect.gen(function* () {
    const smtpConfig = { host: "smtp.example.com", port: 587 }
    const service = new SmtpEmailService(smtpConfig)
    return yield* service // Automatic initialization via commit()
  })
)

// Usage in application
const sendWelcomeEmail = (user: User): Effect.Effect<void> =>
  Effect.gen(function* () {
    const emailService = yield* EmailService
    return yield* emailService.send(
      user.email,
      "Welcome!",
      `Hello ${user.name}, welcome to our service!`
    )
  })

const program = sendWelcomeEmail({ name: "John", email: "john@example.com" }).pipe(
  Effect.provide(EmailServiceLive)
)
```

### Integration with External Libraries

Bridge non-Effect libraries using Effectable for seamless integration.

```typescript
import { Effectable, Effect } from "effect"
import Redis from "ioredis"

// Wrap Redis client as an Effectable
class RedisClient extends Effectable.Class<RedisClient> {
  private client?: Redis
  
  constructor(private config: Redis.RedisOptions) {
    super()
  }
  
  commit(): Effect.Effect<RedisClient> {
    return Effect.gen(function* () {
      this.client = new Redis(this.config)
      
      // Wait for connection
      yield* Effect.fromPromise(() => 
        new Promise<void>((resolve, reject) => {
          this.client!.on('connect', () => resolve())
          this.client!.on('error', reject)
        })
      )
      
      yield* Effect.log("Redis client connected")
      return this
    }.bind(this))
  }
  
  get(key: string): Effect.Effect<string | null> {
    return Effect.fromPromise(() => this.client!.get(key))
  }
  
  set(key: string, value: string, ttlSeconds?: number): Effect.Effect<void> {
    return Effect.fromPromise(async () => {
      if (ttlSeconds) {
        await this.client!.setex(key, ttlSeconds, value)
      } else {
        await this.client!.set(key, value)
      }
    })
  }
  
  delete(key: string): Effect.Effect<number> {
    return Effect.fromPromise(() => this.client!.del(key))
  }
  
  disconnect(): Effect.Effect<void> {
    return Effect.fromPromise(() => this.client!.quit()).pipe(
      Effect.asVoid
    )
  }
}

// Usage with automatic resource management
const cacheWorkflow = Effect.gen(function* () {
  const redis = new RedisClient({ host: "localhost", port: 6379 })
  const client = yield* redis // Automatically connects
  
  yield* client.set("user:123", JSON.stringify({ name: "John" }), 3600)
  const userData = yield* client.get("user:123")
  
  return userData ? JSON.parse(userData) : null
}).pipe(
  Effect.ensuring(Effect.gen(function* () {
    // Cleanup happens automatically due to Effectable lifecycle
  }))
)
```

### Testing Strategies

Create testable Effectable types with dependency injection and mocking capabilities.

```typescript
import { Effectable, Effect, TestContext, TestClock } from "effect"

// Abstract base for testability
abstract class HttpClient extends Effectable.Class<HttpClient> {
  abstract get(url: string): Effect.Effect<Response>
  abstract post(url: string, data: unknown): Effect.Effect<Response>
}

// Production implementation  
class FetchHttpClient extends HttpClient {
  commit(): Effect.Effect<FetchHttpClient> {
    return Effect.succeed(this) // No initialization needed
  }
  
  get(url: string): Effect.Effect<Response> {
    return Effect.fromPromise(() => fetch(url))
  }
  
  post(url: string, data: unknown): Effect.Effect<Response> {
    return Effect.fromPromise(() => 
      fetch(url, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(data)
      })
    )
  }
}

// Test implementation
class MockHttpClient extends HttpClient {
  private responses = new Map<string, Response>()
  private requests: Array<{ method: string; url: string; data?: unknown }> = []
  
  commit(): Effect.Effect<MockHttpClient> {
    return Effect.succeed(this)
  }
  
  mockResponse(url: string, response: Response): void {
    this.responses.set(url, response)
  }
  
  get(url: string): Effect.Effect<Response> {
    return Effect.gen(function* () {
      this.requests.push({ method: "GET", url })
      
      const response = this.responses.get(url)
      if (!response) {
        return yield* Effect.fail(new Error(`No mock response for GET ${url}`))
      }
      
      return response
    }.bind(this))
  }
  
  post(url: string, data: unknown): Effect.Effect<Response> {
    return Effect.gen(function* () {
      this.requests.push({ method: "POST", url, data })
      
      const response = this.responses.get(url)
      if (!response) {
        return yield* Effect.fail(new Error(`No mock response for POST ${url}`))
      }
      
      return response
    }.bind(this))
  }
  
  getRequests() {
    return this.requests
  }
}

// Service using HttpClient
class UserService extends Effectable.Class<UserService> {
  constructor(private httpClient: HttpClient) {
    super()
  }
  
  commit(): Effect.Effect<UserService> {
    return Effect.gen(function* () {
      yield* this.httpClient // Initialize HTTP client
      return this
    })
  }
  
  getUser(id: string): Effect.Effect<User> {
    return Effect.gen(function* () {
      const response = yield* this.httpClient.get(`/api/users/${id}`)
      const json = yield* Effect.fromPromise(() => response.json())
      return json as User
    })
  }
}

// Test usage
const testUserService = Effect.gen(function* () {
  const mockClient = new MockHttpClient()
  mockClient.mockResponse("/api/users/123", new Response(
    JSON.stringify({ id: "123", name: "John Doe" })
  ))
  
  const userService = new UserService(mockClient)
  const service = yield* userService
  
  const user = yield* service.getUser("123")
  
  // Verify behavior
  const requests = mockClient.getRequests()
  console.log(`Made ${requests.length} requests`)
  
  return user
}).pipe(
  Effect.provide(TestContext.TestContext)
)
```

## Conclusion

Effectable provides a powerful foundation for creating custom types that integrate seamlessly with the Effect ecosystem, enabling elegant APIs that maintain type safety and composability.

Key benefits:
- **Seamless Integration**: Custom types behave like native Effect values
- **Type Safety**: Full TypeScript support with proper inference
- **Resource Management**: Built-in patterns for initialization and cleanup
- **Composability**: Works naturally with Effect pipelines and service layers
- **Testability**: Easy to mock and test with dependency injection

Use Effectable when you need to create domain-specific types that should feel like first-class citizens in Effect workflows, such as database connections, external service clients, or complex business domain objects that encapsulate both data and behavior.