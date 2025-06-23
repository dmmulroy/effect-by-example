# GlobalValue: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem GlobalValue Solves

In modern JavaScript development, especially with hot-reloading frameworks like Next.js and Remix, or when mixing CommonJS and ESM modules, global state can be unexpectedly recreated multiple times. This leads to:

```typescript
// Traditional approach - problematic singleton
class DatabaseConnectionPool {
  private static instance: DatabaseConnectionPool
  private connections: Connection[] = []

  static getInstance() {
    if (!this.instance) {
      this.instance = new DatabaseConnectionPool()
    }
    return this.instance
  }

  private constructor() {
    // Expensive initialization
    this.initializeConnections()
  }
}

// Problems arise with hot-reloading:
const pool1 = DatabaseConnectionPool.getInstance() // First load
// Hot reload occurs...
const pool2 = DatabaseConnectionPool.getInstance() // New instance!
// pool1 !== pool2, connections are lost
```

This approach leads to:
- **Resource Duplication** - Multiple instances of expensive resources like connection pools, caches, or registries
- **State Loss** - Hot-reloading destroys accumulated state and configuration
- **Memory Leaks** - Orphaned instances aren't garbage collected properly
- **Inconsistent State** - Different parts of the application reference different instances

### The GlobalValue Solution

GlobalValue ensures that a single instance of a value is created globally, even across module reloads or mixed import environments:

```typescript
import { globalValue } from "effect/GlobalValue"

// This pool persists across hot-reloads and mixed imports
const connectionPool = globalValue(
  Symbol.for("app/database/connectionPool"),
  () => new DatabaseConnectionPool()
)

// Always returns the same instance, regardless of module reloading
const getPool = () => connectionPool
```

### Key Concepts

**Global Store**: A versioned global store identified by the Effect library version, ensuring isolation between different Effect versions

**Lazy Initialization**: Values are computed only when first accessed using the provided compute function

**Identity-Based Caching**: Uses any value as an identifier (typically Symbols) to uniquely identify cached values

## Basic Usage Patterns

### Pattern 1: Simple Global Cache

```typescript
import { globalValue } from "effect/GlobalValue"

// Create a persistent cache that survives hot-reloads
const userCache = globalValue(
  Symbol.for("app/cache/users"),
  () => new Map<string, User>()
)

// Usage is straightforward
const getUser = (id: string): User | undefined => userCache.get(id)
const setUser = (id: string, user: User): void => userCache.set(id, user)
```

### Pattern 2: Configuration Registry

```typescript
import { globalValue } from "effect/GlobalValue"

interface AppConfig {
  apiUrl: string
  timeout: number
  retries: number
}

const appConfig = globalValue(
  Symbol.for("app/config"),
  (): AppConfig => ({
    apiUrl: process.env.API_URL ?? "http://localhost:3000",
    timeout: Number(process.env.TIMEOUT) || 5000,
    retries: Number(process.env.RETRIES) || 3
  })
)

// Access configuration anywhere in the application
const config = appConfig
```

### Pattern 3: Service Registry with Effect Integration

```typescript
import { globalValue } from "effect/GlobalValue"
import * as Context from "effect/Context"
import * as FiberRef from "effect/FiberRef"
import * as Effect from "effect/Effect"

// Global service instances for dependency injection
const serviceRegistry = globalValue(
  Symbol.for("app/services/registry"),
  () => new Map<string, unknown>()
)

// Helper for service registration
const registerService = <T>(key: string, service: T): void => {
  serviceRegistry.set(key, service)
}

// Effect-based service lookup
const getService = <T>(key: string): Effect.Effect<T, Error> =>
  Effect.sync(() => {
    const service = serviceRegistry.get(key)
    if (!service) {
      throw new Error(`Service not found: ${key}`)
    }
    return service as T
  })
```

## Real-World Examples

### Example 1: Database Connection Pool Management

Modern applications need persistent database connections that survive development hot-reloads and maintain connection limits.

```typescript
import { globalValue } from "effect/GlobalValue"
import * as Effect from "effect/Effect"
import * as Layer from "effect/Layer"
import * as Context from "effect/Context"

interface DatabasePool {
  readonly query: <T>(sql: string, params?: unknown[]) => Effect.Effect<T, Error>
  readonly transaction: <T>(
    fn: (tx: DatabasePool) => Effect.Effect<T, Error>
  ) => Effect.Effect<T, Error>
  readonly close: () => Effect.Effect<void, Error>
}

const DatabasePool = Context.GenericTag<DatabasePool>("app/DatabasePool")

// Global connection pool that persists across reloads
const createDatabasePool = globalValue(
  Symbol.for("app/database/pool"),
  () => Effect.gen(function* () {
    console.log("Creating new database pool...") // Only logs once
    
    const config = yield* Effect.sync(() => ({
      host: process.env.DB_HOST ?? "localhost",
      port: Number(process.env.DB_PORT) || 5432,
      database: process.env.DB_NAME ?? "myapp",
      maxConnections: 20
    }))

    // Simulate expensive pool creation
    const pool = yield* Effect.sync(() => ({
      query: <T>(sql: string, params?: unknown[]) =>
        Effect.sync(() => {
          console.log(`Executing: ${sql}`)
          return {} as T
        }),
      
      transaction: <T>(fn: (tx: DatabasePool) => Effect.Effect<T, Error>) =>
        Effect.gen(function* () {
          console.log("Starting transaction")
          const result = yield* fn(pool)
          console.log("Committing transaction")
          return result
        }),
      
      close: () => Effect.sync(() => {
        console.log("Closing database pool")
      })
    }))

    return pool
  })
)

// Layer that provides the global database pool
export const DatabasePoolLive = Layer.effect(
  DatabasePool,
  createDatabasePool
)

// Usage in business logic
export const getUserById = (id: string) =>
  Effect.gen(function* () {
    const db = yield* DatabasePool
    const user = yield* db.query<User>(
      "SELECT * FROM users WHERE id = $1",
      [id]
    )
    return user
  })
```

### Example 2: HTTP Client with Global Configuration

API clients often need shared configuration, request interceptors, and connection pooling that should persist across module reloads.

```typescript
import { globalValue } from "effect/GlobalValue"
import * as Effect from "effect/Effect"
import * as Context from "effect/Context"
import * as FiberRef from "effect/FiberRef"

interface HttpClientConfig {
  readonly baseUrl: string
  readonly timeout: number
  readonly retries: number
  readonly headers: Record<string, string>
}

interface HttpClient {
  readonly get: <T>(url: string) => Effect.Effect<T, Error>
  readonly post: <T>(url: string, body: unknown) => Effect.Effect<T, Error>
  readonly put: <T>(url: string, body: unknown) => Effect.Effect<T, Error>
  readonly delete: (url: string) => Effect.Effect<void, Error>
}

const HttpClient = Context.GenericTag<HttpClient>("app/HttpClient")

// Global client configuration that survives hot-reloads
const createHttpClient = globalValue(
  Symbol.for("app/http/client"),
  () => Effect.gen(function* () {
    console.log("Initializing HTTP client...") // Only runs once
    
    const config: HttpClientConfig = {
      baseUrl: process.env.API_BASE_URL ?? "https://api.example.com",
      timeout: Number(process.env.API_TIMEOUT) || 10000,
      retries: Number(process.env.API_RETRIES) || 3,
      headers: {
        "User-Agent": "MyApp/1.0",
        "Accept": "application/json",
        "Content-Type": "application/json"
      }
    }

    // Request interceptor for auth tokens
    const currentAuthToken = yield* Effect.sync(() =>
      FiberRef.unsafeMake<string | null>(null)
    )

    const makeRequest = <T>(
      method: string,
      url: string,
      body?: unknown
    ): Effect.Effect<T, Error> =>
      Effect.gen(function* () {
        const token = yield* FiberRef.get(currentAuthToken)
        const headers = { ...config.headers }
        
        if (token) {
          headers["Authorization"] = `Bearer ${token}`
        }

        // Simulate HTTP request
        const fullUrl = `${config.baseUrl}${url}`
        console.log(`${method} ${fullUrl}`)
        
        if (body) {
          console.log("Request body:", body)
        }
        
        return {} as T
      })

    const client: HttpClient = {
      get: <T>(url: string) => makeRequest<T>("GET", url),
      post: <T>(url: string, body: unknown) => makeRequest<T>("POST", url, body),
      put: <T>(url: string, body: unknown) => makeRequest<T>("PUT", url, body),
      delete: (url: string) => makeRequest<void>("DELETE", url)
    }

    return { client, currentAuthToken }
  })
)

// Layer and helper functions
export const HttpClientLive = Layer.effect(
  HttpClient,
  createHttpClient.pipe(Effect.map(({ client }) => client))
)

export const setAuthToken = (token: string) =>
  Effect.gen(function* () {
    const { currentAuthToken } = yield* createHttpClient
    yield* FiberRef.set(currentAuthToken, token)
  })

// Usage in application services
export const fetchUserProfile = (userId: string) =>
  Effect.gen(function* () {
    const client = yield* HttpClient
    const profile = yield* client.get<UserProfile>(`/users/${userId}`)
    return profile
  })
```

### Example 3: Application Metrics Registry

Metrics collection systems need global registries that persist across reloads to maintain accurate counts and measurements.

```typescript
import { globalValue } from "effect/GlobalValue"
import * as Effect from "effect/Effect"
import * as Context from "effect/Context"
import * as Ref from "effect/Ref"

interface MetricValue {
  readonly count: number
  readonly sum: number
  readonly min: number
  readonly max: number
  readonly lastUpdated: Date
}

interface MetricsRegistry {
  readonly counter: (name: string, value?: number) => Effect.Effect<void>
  readonly gauge: (name: string, value: number) => Effect.Effect<void>
  readonly histogram: (name: string, value: number) => Effect.Effect<void>
  readonly getMetric: (name: string) => Effect.Effect<MetricValue | null>
  readonly getAllMetrics: () => Effect.Effect<Record<string, MetricValue>>
  readonly reset: (name?: string) => Effect.Effect<void>
}

const MetricsRegistry = Context.GenericTag<MetricsRegistry>("app/MetricsRegistry")

// Global metrics that persist through development cycles
const createMetricsRegistry = globalValue(
  Symbol.for("app/metrics/registry"),
  () => Effect.gen(function* () {
    console.log("Creating metrics registry...") // Only logs once
    
    const metricsRef = yield* Ref.make(new Map<string, MetricValue>())

    const updateMetric = (
      name: string,
      updater: (existing: MetricValue | undefined) => MetricValue
    ) =>
      Ref.update(metricsRef, metrics => {
        const existing = metrics.get(name)
        const updated = updater(existing)
        return new Map(metrics).set(name, updated)
      })

    const registry: MetricsRegistry = {
      counter: (name: string, value = 1) =>
        updateMetric(name, existing => ({
          count: (existing?.count ?? 0) + value,
          sum: (existing?.sum ?? 0) + value,
          min: Math.min(existing?.min ?? value, value),
          max: Math.max(existing?.max ?? value, value),
          lastUpdated: new Date()
        })),

      gauge: (name: string, value: number) =>
        updateMetric(name, existing => ({
          count: (existing?.count ?? 0) + 1,
          sum: value, // Gauge shows current value
          min: Math.min(existing?.min ?? value, value),
          max: Math.max(existing?.max ?? value, value),
          lastUpdated: new Date()
        })),

      histogram: (name: string, value: number) =>
        updateMetric(name, existing => ({
          count: (existing?.count ?? 0) + 1,
          sum: (existing?.sum ?? 0) + value,
          min: Math.min(existing?.min ?? value, value),
          max: Math.max(existing?.max ?? value, value),
          lastUpdated: new Date()
        })),

      getMetric: (name: string) =>
        Ref.get(metricsRef).pipe(
          Effect.map(metrics => metrics.get(name) ?? null)
        ),

      getAllMetrics: () =>
        Ref.get(metricsRef).pipe(
          Effect.map(metrics => Object.fromEntries(metrics.entries()))
        ),

      reset: (name?: string) =>
        name 
          ? Ref.update(metricsRef, metrics => {
              const newMetrics = new Map(metrics)
              newMetrics.delete(name)
              return newMetrics
            })
          : Ref.set(metricsRef, new Map())
    }

    return registry
  })
)

// Layer for dependency injection
export const MetricsRegistryLive = Layer.effect(
  MetricsRegistry,
  createMetricsRegistry
)

// Helper functions for common metrics patterns
export const trackApiCall = (endpoint: string) =>
  Effect.gen(function* () {
    const metrics = yield* MetricsRegistry
    const start = Date.now()
    
    return {
      success: () => Effect.gen(function* () {
        const duration = Date.now() - start
        yield* metrics.counter(`api.${endpoint}.requests`)
        yield* metrics.counter(`api.${endpoint}.success`)
        yield* metrics.histogram(`api.${endpoint}.duration`, duration)
      }),
      
      error: () => Effect.gen(function* () {
        const duration = Date.now() - start
        yield* metrics.counter(`api.${endpoint}.requests`)
        yield* metrics.counter(`api.${endpoint}.errors`)
        yield* metrics.histogram(`api.${endpoint}.duration`, duration)
      })
    }
  })

// Usage in API handlers
export const getUsersHandler = Effect.gen(function* () {
  const tracker = yield* trackApiCall("users.list")
  
  try {
    const users = yield* fetchUsers()
    yield* tracker.success()
    return users
  } catch (error) {
    yield* tracker.error()
    throw error
  }
})
```

## Advanced Features Deep Dive

### Global Value Identity and Versioning

GlobalValue uses a versioned global store to ensure compatibility across different Effect library versions:

```typescript
import { globalValue } from "effect/GlobalValue"

// The global store is versioned to prevent conflicts
// between different Effect versions in the same runtime
const myValue = globalValue(
  Symbol.for("myapp/feature/cache"),
  () => new Map<string, unknown>()
)

// Even if multiple versions of Effect are loaded,
// each version gets its own isolated global store
```

### Custom Identity Strategies

While Symbols are recommended, any value can serve as an identity key:

```typescript
import { globalValue } from "effect/GlobalValue"

// String-based keys (less safe due to potential collisions)
const stringKeyCache = globalValue(
  "app.cache.users",
  () => new Map<string, User>()
)

// Object-based keys (useful for complex scenarios)
const configKey = { module: "database", feature: "pool" }
const dbPool = globalValue(configKey, () => createDatabasePool())

// Class-based keys (advanced pattern)
class ServiceRegistry {
  static readonly KEY = Symbol.for("ServiceRegistry")
}

const registry = globalValue(
  ServiceRegistry.KEY,
  () => new Map<string, unknown>()
)
```

### Lazy Initialization Patterns

GlobalValue supports complex initialization logic including async operations:

```typescript
import { globalValue } from "effect/GlobalValue"
import * as Effect from "effect/Effect"

// Async initialization with Effect
const asyncResource = globalValue(
  Symbol.for("app/resources/async"),
  () => Effect.gen(function* () {
    console.log("Starting async initialization...")
    
    // Simulate async setup
    yield* Effect.sleep("1 second")
    
    const config = yield* loadConfiguration()
    const connection = yield* establishConnection(config)
    const cache = yield* initializeCache()
    
    return {
      config,
      connection,
      cache,
      initialized: true
    }
  })
)

// Usage requires running the Effect
const useAsyncResource = Effect.gen(function* () {
  const resource = yield* asyncResource
  // Use the initialized resource
  return resource
})
```

### Error Handling in Global Values

Handle initialization failures gracefully:

```typescript
import { globalValue } from "effect/GlobalValue"
import * as Effect from "effect/Effect"
import * as Option from "effect/Option"

// Fallback pattern for failed initialization
const resilientCache = globalValue(
  Symbol.for("app/cache/resilient"),
  () => {
    try {
      // Attempt primary cache initialization
      return createRedisCache()
    } catch (error) {
      console.warn("Redis cache failed, falling back to memory cache:", error)
      // Fallback to in-memory cache
      return new Map<string, unknown>()
    }
  }
)

// Option-based error handling
const optionalResource = globalValue(
  Symbol.for("app/resources/optional"),
  () => {
    try {
      const resource = createExpensiveResource()
      return Option.some(resource)
    } catch (error) {
      console.error("Resource initialization failed:", error)
      return Option.none()
    }
  }
)

// Usage with Option
const useOptionalResource = Effect.gen(function* () {
  const resourceOption = optionalResource
  
  if (Option.isSome(resourceOption)) {
    return yield* processWithResource(resourceOption.value)
  } else {
    return yield* processWithoutResource()
  }
})
```

## Practical Patterns & Best Practices

### Pattern 1: Service Factory with Dependencies

Create reusable service factories that manage their own global instances:

```typescript
import { globalValue } from "effect/GlobalValue"
import * as Effect from "effect/Effect"
import * as Context from "effect/Context"

// Generic service factory pattern
const createServiceFactory = <T>(
  key: symbol,
  factory: () => Effect.Effect<T, Error>
) => {
  const service = globalValue(key, factory)
  return service
}

// Email service factory
interface EmailService {
  readonly send: (to: string, subject: string, body: string) => Effect.Effect<void, Error>
  readonly sendBulk: (emails: Email[]) => Effect.Effect<void, Error>
}

const EmailService = Context.GenericTag<EmailService>("app/EmailService")

const createEmailService = () =>
  Effect.gen(function* () {
    const config = yield* Effect.sync(() => ({
      smtp: {
        host: process.env.SMTP_HOST ?? "localhost",
        port: Number(process.env.SMTP_PORT) || 587,
        secure: process.env.SMTP_SECURE === "true"
      }
    }))

    const transport = yield* Effect.sync(() => 
      createSMTPTransport(config.smtp)
    )

    return {
      send: (to: string, subject: string, body: string) =>
        Effect.sync(() => {
          console.log(`Sending email to ${to}: ${subject}`)
          // Use transport to send email
        }),
      
      sendBulk: (emails: Email[]) =>
        Effect.sync(() => {
          console.log(`Sending ${emails.length} bulk emails`)
          // Use transport for bulk sending
        })
    } satisfies EmailService
  })

// Global email service instance
export const emailService = createServiceFactory(
  Symbol.for("app/services/email"),
  createEmailService
)

export const EmailServiceLive = Layer.effect(EmailService, emailService)
```

### Pattern 2: Configuration Management with Hot-Reloading

Manage application configuration that persists across development reloads:

```typescript
import { globalValue } from "effect/GlobalValue"
import * as Effect from "effect/Effect"
import * as Ref from "effect/Ref"

interface AppConfig {
  readonly database: {
    readonly host: string
    readonly port: number
    readonly name: string
  }
  readonly api: {
    readonly baseUrl: string
    readonly timeout: number
  }
  readonly features: {
    readonly enableLogging: boolean
    readonly enableMetrics: boolean
  }
}

interface ConfigManager {
  readonly getConfig: () => Effect.Effect<AppConfig>
  readonly updateConfig: (updater: (config: AppConfig) => AppConfig) => Effect.Effect<void>
  readonly reloadConfig: () => Effect.Effect<void>
  readonly watchConfig: (callback: (config: AppConfig) => void) => Effect.Effect<void>
}

const createConfigManager = globalValue(
  Symbol.for("app/config/manager"),
  () => Effect.gen(function* () {
    const loadConfig = (): AppConfig => ({
      database: {
        host: process.env.DB_HOST ?? "localhost",
        port: Number(process.env.DB_PORT) || 5432,
        name: process.env.DB_NAME ?? "myapp"
      },
      api: {
        baseUrl: process.env.API_BASE_URL ?? "http://localhost:3000",
        timeout: Number(process.env.API_TIMEOUT) || 5000
      },
      features: {
        enableLogging: process.env.ENABLE_LOGGING !== "false",
        enableMetrics: process.env.ENABLE_METRICS !== "false"
      }
    })

    const configRef = yield* Ref.make(loadConfig())
    const watchers = yield* Ref.make<Array<(config: AppConfig) => void>>([])

    const notifyWatchers = (config: AppConfig) =>
      Effect.gen(function* () {
        const watcherList = yield* Ref.get(watchers)
        yield* Effect.sync(() => {
          watcherList.forEach(callback => callback(config))
        })
      })

    const manager: ConfigManager = {
      getConfig: () => Ref.get(configRef),
      
      updateConfig: (updater: (config: AppConfig) => AppConfig) =>
        Effect.gen(function* () {
          const oldConfig = yield* Ref.get(configRef)
          const newConfig = updater(oldConfig)
          yield* Ref.set(configRef, newConfig)
          yield* notifyWatchers(newConfig)
        }),
      
      reloadConfig: () =>
        Effect.gen(function* () {
          const newConfig = loadConfig()
          yield* Ref.set(configRef, newConfig)
          yield* notifyWatchers(newConfig)
          console.log("Configuration reloaded")
        }),
      
      watchConfig: (callback: (config: AppConfig) => void) =>
        Ref.update(watchers, list => [...list, callback])
    }

    return manager
  })
)

// Helper functions for common config operations
export const getConfig = () =>
  Effect.gen(function* () {
    const manager = yield* createConfigManager
    return yield* manager.getConfig()
  })

export const getDatabaseConfig = () =>
  getConfig().pipe(Effect.map(config => config.database))

export const getApiConfig = () =>
  getConfig().pipe(Effect.map(config => config.api))
```

### Pattern 3: Resource Pool Management

Manage expensive resources like database connections, HTTP clients, or file handles:

```typescript
import { globalValue } from "effect/GlobalValue"
import * as Effect from "effect/Effect"
import * as Queue from "effect/Queue"
import * as Ref from "effect/Ref"

interface Resource<T> {
  readonly value: T
  readonly isHealthy: () => boolean
  readonly close: () => Effect.Effect<void>
}

interface ResourcePool<T> {
  readonly acquire: () => Effect.Effect<Resource<T>, Error>
  readonly release: (resource: Resource<T>) => Effect.Effect<void>
  readonly drain: () => Effect.Effect<void>
  readonly size: () => Effect.Effect<number>
  readonly healthy: () => Effect.Effect<number>
}

const createResourcePool = <T>(
  name: string,
  factory: () => Effect.Effect<T, Error>,
  options: {
    readonly min: number
    readonly max: number
    readonly healthCheck?: (resource: T) => boolean
  }
) =>
  globalValue(
    Symbol.for(`app/pool/${name}`),
    () => Effect.gen(function* () {
      console.log(`Creating resource pool: ${name}`)
      
      const available = yield* Queue.unbounded<Resource<T>>()
      const inUse = yield* Ref.make(new Set<Resource<T>>())

      const createResource = () =>
        Effect.gen(function* () {
          const value = yield* factory()
          const resource: Resource<T> = {
            value,
            isHealthy: () => options.healthCheck?.(value) ?? true,
            close: () => Effect.sync(() => {
              console.log(`Closing resource in pool: ${name}`)
              // Cleanup logic here
            })
          }
          return resource
        })

      // Pre-populate pool with minimum resources
      yield* Effect.gen(function* () {
        for (let i = 0; i < options.min; i++) {
          const resource = yield* createResource()
          yield* Queue.offer(available, resource)
        }
      })

      const pool: ResourcePool<T> = {
        acquire: () =>
          Effect.gen(function* () {
            const resource = yield* Queue.take(available)
            yield* Ref.update(inUse, set => new Set(set).add(resource))
            
            if (!resource.isHealthy()) {
              yield* resource.close()
              yield* pool.release(resource)
              return yield* pool.acquire() // Retry with fresh resource
            }
            
            return resource
          }),

        release: (resource: Resource<T>) =>
          Effect.gen(function* () {
            yield* Ref.update(inUse, set => {
              const newSet = new Set(set)
              newSet.delete(resource)
              return newSet
            })
            
            if (resource.isHealthy()) {
              yield* Queue.offer(available, resource)
            } else {
              yield* resource.close()
              // Create replacement if below minimum
              const currentSize = yield* Queue.size(available)
              if (currentSize < options.min) {
                const newResource = yield* createResource()
                yield* Queue.offer(available, newResource)
              }
            }
          }),

        drain: () =>
          Effect.gen(function* () {
            console.log(`Draining resource pool: ${name}`)
            
            // Close all available resources
            const availableResources = yield* Queue.takeAll(available)
            yield* Effect.forEach(availableResources, resource => resource.close())
            
            // Close all in-use resources
            const inUseSet = yield* Ref.get(inUse)
            yield* Effect.forEach(Array.from(inUseSet), resource => resource.close())
            yield* Ref.set(inUse, new Set())
          }),

        size: () =>
          Effect.gen(function* () {
            const availableCount = yield* Queue.size(available)
            const inUseSet = yield* Ref.get(inUse)
            return availableCount + inUseSet.size
          }),

        healthy: () =>
          Effect.gen(function* () {
            const availableResources = yield* Queue.takeAll(available)
            const healthyResources = availableResources.filter(r => r.isHealthy())
            
            // Put healthy resources back
            yield* Effect.forEach(healthyResources, r => Queue.offer(available, r))
            
            // Close unhealthy resources
            const unhealthyResources = availableResources.filter(r => !r.isHealthy())
            yield* Effect.forEach(unhealthyResources, r => r.close())
            
            return healthyResources.length
          })
      }

      return pool
    })
  )

// Database connection pool example
export const databasePool = createResourcePool(
  "database",
  () => Effect.sync(() => ({
    connectionId: Math.random().toString(36),
    lastUsed: Date.now(),
    query: (sql: string) => console.log(`Executing: ${sql}`)
  })),
  {
    min: 5,
    max: 20,
    healthCheck: (conn) => Date.now() - conn.lastUsed < 30000 // 30 second timeout
  }
)

// Usage pattern
export const withDatabaseConnection = <T, E>(
  operation: (connection: { query: (sql: string) => void }) => Effect.Effect<T, E>
) =>
  Effect.gen(function* () {
    const pool = yield* databasePool
    const resource = yield* pool.acquire()
    
    try {
      const result = yield* operation(resource.value)
      return result
    } finally {
      yield* pool.release(resource)
    }
  })
```

## Integration Examples

### Integration with Layer and Runtime

GlobalValue works seamlessly with Effect's dependency injection system:

```typescript
import { globalValue } from "effect/GlobalValue"
import * as Effect from "effect/Effect"
import * as Layer from "effect/Layer"
import * as Runtime from "effect/Runtime"
import * as Context from "effect/Context"

// Service definitions
interface Logger {
  readonly info: (message: string) => Effect.Effect<void>
  readonly error: (message: string, error?: unknown) => Effect.Effect<void>
}

interface Database {
  readonly query: <T>(sql: string) => Effect.Effect<T, Error>
}

interface HttpServer {
  readonly start: (port: number) => Effect.Effect<void, Error>
  readonly stop: () => Effect.Effect<void>
}

const Logger = Context.GenericTag<Logger>("app/Logger")
const Database = Context.GenericTag<Database>("app/Database")
const HttpServer = Context.GenericTag<HttpServer>("app/HttpServer")

// Global runtime that persists across reloads
const createApplicationRuntime = globalValue(
  Symbol.for("app/runtime"),
  () => {
    console.log("Creating application runtime...")
    
    // Logger implementation
    const LoggerLive = Layer.succeed(Logger, {
      info: (message: string) =>
        Effect.sync(() => console.log(`[INFO] ${message}`)),
      error: (message: string, error?: unknown) =>
        Effect.sync(() => console.error(`[ERROR] ${message}`, error))
    })

    // Database implementation
    const DatabaseLive = Layer.succeed(Database, {
      query: <T>(sql: string) =>
        Effect.sync(() => {
          console.log(`Query: ${sql}`)
          return {} as T
        })
    })

    // HTTP Server implementation
    const HttpServerLive = Layer.succeed(HttpServer, {
      start: (port: number) =>
        Effect.sync(() => console.log(`Server starting on port ${port}`)),
      stop: () =>
        Effect.sync(() => console.log("Server stopping"))
    })

    // Compose all layers
    const ApplicationLayer = Layer.empty.pipe(
      Layer.provide(LoggerLive),
      Layer.provide(DatabaseLive),
      Layer.provide(HttpServerLive)
    )

    return Runtime.make(ApplicationLayer)
  }
)

// Application bootstrap
export const startApplication = Effect.gen(function* () {
  const runtime = yield* createApplicationRuntime
  
  // Your application logic using the runtime
  const program = Effect.gen(function* () {
    const logger = yield* Logger
    const database = yield* Database
    const server = yield* HttpServer

    yield* logger.info("Application starting...")
    yield* database.query("SELECT 1")
    yield* server.start(3000)
    yield* logger.info("Application started successfully")
  })

  // Run the program with the global runtime
  return yield* program.pipe(Runtime.runPromise(runtime))
})
```

### Integration with Testing Frameworks

GlobalValue can be managed in testing environments to ensure test isolation:

```typescript
import { globalValue } from "effect/GlobalValue"
import * as Effect from "effect/Effect"
import * as Ref from "effect/Ref"

// Test utilities for managing global state
interface TestContext {
  readonly cleanup: () => Effect.Effect<void>
  readonly reset: () => Effect.Effect<void>
  readonly snapshot: () => Effect.Effect<unknown>
  readonly restore: (snapshot: unknown) => Effect.Effect<void>
}

const createTestContext = globalValue(
  Symbol.for("test/context"),
  () => Effect.gen(function* () {
    const cleanupTasks = yield* Ref.make<Array<() => Effect.Effect<void>>>([])
    const snapshots = yield* Ref.make<Map<string, unknown>>(new Map())

    const context: TestContext = {
      cleanup: () =>
        Effect.gen(function* () {
          const tasks = yield* Ref.get(cleanupTasks)
          yield* Effect.forEach(tasks.reverse(), task => task())
          yield* Ref.set(cleanupTasks, [])
        }),

      reset: () =>
        Effect.gen(function* () {
          yield* context.cleanup()
          yield* Ref.set(snapshots, new Map())
        }),

      snapshot: () =>
        Effect.gen(function* () {
          const currentSnapshots = yield* Ref.get(snapshots)
          return Object.fromEntries(currentSnapshots)
        }),

      restore: (snapshot: unknown) =>
        Effect.gen(function* () {
          if (typeof snapshot === "object" && snapshot !== null) {
            const entries = Object.entries(snapshot)
            yield* Ref.set(snapshots, new Map(entries))
          }
        })
    }

    return context
  })
)

// Test helper functions
export const withTestCleanup = <T, E>(
  operation: () => Effect.Effect<T, E>
) =>
  Effect.gen(function* () {
    const testContext = yield* createTestContext
    
    try {
      return yield* operation()
    } finally {
      yield* testContext.cleanup()
    }
  })

export const addTestCleanup = (cleanup: () => Effect.Effect<void>) =>
  Effect.gen(function* () {
    const testContext = yield* createTestContext
    const cleanupTasks = yield* Ref.get(testContext.cleanup as any) // Type assertion for internal access
    yield* Ref.set(cleanupTasks as any, [...cleanupTasks, cleanup])
  })

// Usage in tests
export const testDatabaseOperations = Effect.gen(function* () {
  // Setup test database
  const db = yield* setupTestDatabase()
  yield* addTestCleanup(() => db.close())

  // Run test operations
  const user = yield* db.createUser({ name: "Test User" })
  const retrieved = yield* db.getUser(user.id)

  // Test assertions would go here
  console.assert(retrieved.name === "Test User")
})

// Run test with automatic cleanup
export const runTest = (test: Effect.Effect<void, Error>) =>
  withTestCleanup(() => test).pipe(
    Effect.catchAll(error => 
      Effect.sync(() => console.error("Test failed:", error))
    )
  )
```

### Integration with Third-Party Libraries

GlobalValue helps bridge Effect with traditional JavaScript libraries:

```typescript
import { globalValue } from "effect/GlobalValue"
import * as Effect from "effect/Effect"
import * as Layer from "effect/Layer"
import * as Context from "effect/Context"

// Example: Redis client integration
interface RedisClient {
  readonly get: (key: string) => Effect.Effect<string | null, Error>
  readonly set: (key: string, value: string, ttl?: number) => Effect.Effect<void, Error>
  readonly del: (key: string) => Effect.Effect<number, Error>
  readonly exists: (key: string) => Effect.Effect<boolean, Error>
}

const RedisClient = Context.GenericTag<RedisClient>("app/RedisClient")

// Global Redis connection that survives hot-reloads
const createRedisClient = globalValue(
  Symbol.for("app/redis/client"),
  () => Effect.gen(function* () {
    console.log("Connecting to Redis...")
    
    // Simulate Redis client creation (replace with actual redis library)
    const client = {
      connected: true,
      get: async (key: string) => `value-${key}`,
      set: async (key: string, value: string, ttl?: number) => {
        console.log(`Redis SET ${key} = ${value}${ttl ? ` (TTL: ${ttl}s)` : ''}`)
      },
      del: async (key: string) => {
        console.log(`Redis DEL ${key}`)
        return 1
      },
      exists: async (key: string) => true
    }

    // Wrap client methods with Effect
    const effectClient: RedisClient = {
      get: (key: string) =>
        Effect.tryPromise({
          try: () => client.get(key),
          catch: (error) => new Error(`Redis GET failed: ${error}`)
        }),

      set: (key: string, value: string, ttl?: number) =>
        Effect.tryPromise({
          try: () => client.set(key, value, ttl),
          catch: (error) => new Error(`Redis SET failed: ${error}`)
        }),

      del: (key: string) =>
        Effect.tryPromise({
          try: () => client.del(key),
          catch: (error) => new Error(`Redis DEL failed: ${error}`)
        }),

      exists: (key: string) =>
        Effect.tryPromise({
          try: () => client.exists(key),
          catch: (error) => new Error(`Redis EXISTS failed: ${error}`)
        })
    }

    return effectClient
  })
)

export const RedisClientLive = Layer.effect(RedisClient, createRedisClient)

// Cache service using Redis
interface CacheService {
  readonly get: <T>(key: string) => Effect.Effect<T | null, Error>
  readonly set: <T>(key: string, value: T, ttl?: number) => Effect.Effect<void, Error>
  readonly invalidate: (key: string) => Effect.Effect<void, Error>
  readonly clear: (pattern?: string) => Effect.Effect<void, Error>
}

const CacheService = Context.GenericTag<CacheService>("app/CacheService")

export const CacheServiceLive = Layer.effect(
  CacheService,
  Effect.gen(function* () {
    const redis = yield* RedisClient

    const service: CacheService = {
      get: <T>(key: string) =>
        redis.get(key).pipe(
          Effect.map(value => value ? JSON.parse(value) as T : null),
          Effect.catchAll(() => Effect.succeed(null))
        ),

      set: <T>(key: string, value: T, ttl?: number) =>
        redis.set(key, JSON.stringify(value), ttl),

      invalidate: (key: string) =>
        redis.del(key).pipe(Effect.asVoid),

      clear: (pattern?: string) =>
        Effect.gen(function* () {
          // In a real implementation, you'd use Redis SCAN with pattern matching
          console.log(`Clearing cache with pattern: ${pattern ?? '*'}`)
        })
    }

    return service
  })
)

// Usage in application services
export const getUserFromCache = (userId: string) =>
  Effect.gen(function* () {
    const cache = yield* CacheService
    const cacheKey = `user:${userId}`
    
    const cached = yield* cache.get<User>(cacheKey)
    if (cached) {
      return cached
    }

    // Cache miss - fetch from database
    const user = yield* fetchUserFromDatabase(userId)
    yield* cache.set(cacheKey, user, 300) // 5 minute TTL
    
    return user
  })
```

## Conclusion

GlobalValue provides **persistent global state management**, **hot-reload resilience**, and **resource efficiency** for Effect applications.

Key benefits:
- **Development Productivity**: Maintains application state through hot-reloads, eliminating repetitive re-initialization
- **Resource Management**: Prevents duplication of expensive resources like database connections and HTTP clients  
- **Module Safety**: Works reliably across mixed CommonJS/ESM environments and multiple module imports
- **Effect Integration**: Seamlessly integrates with Effect's dependency injection and runtime systems

GlobalValue is essential when building Effect applications that need persistent global state, shared resources, or resilience against module reloading in development environments.