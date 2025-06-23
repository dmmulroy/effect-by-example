# KeyedPool: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem KeyedPool Solves

Traditional resource pooling solutions require separate pools for each resource type or configuration. This leads to complex pool management, memory waste, and potential resource leaks when working with resources that need to be organized by key.

```typescript
// Traditional approach - multiple separate pools
const databasePool = new Pool(() => createDatabaseConnection())
const redisPool = new Pool(() => createRedisConnection())
const s3Pool = new Pool(() => createS3Client())

// Issues: 
// - Separate management for each pool
// - No unified resource lifecycle
// - Memory waste with fixed pools
// - Complex cleanup logic
```

This approach leads to:
- **Resource Fragmentation** - Each resource type requires its own pool management
- **Memory Inefficiency** - Fixed pools allocate maximum resources even when unused
- **Complex Lifecycle Management** - No unified way to clean up resources
- **Poor Scalability** - Difficult to add new resource types or configurations

### The KeyedPool Solution

KeyedPool provides key-based resource pooling where each key maintains its own pool of resources, with unified lifecycle management and automatic cleanup.

```typescript
import { KeyedPool, Effect, Scope } from "effect"

// Create a keyed pool for database connections by region
const connectionPool = KeyedPool.make({
  acquire: (region: string) => createDatabaseConnection(region),
  size: 10
})

// Use resources by key - pools are created on-demand
const useDatabase = (region: string) => Effect.gen(function* () {
  const db = yield* KeyedPool.get(connectionPool, region)
  return yield* db.query("SELECT * FROM users")
})
```

### Key Concepts

**KeyedPool<K, A, E>**: A pool of pools, where each key `K` maintains its own pool of resources `A`

**Scoped Resources**: Resources are automatically managed within scopes, ensuring proper cleanup

**Lazy Pool Creation**: Pools for specific keys are created only when first accessed

## Basic Usage Patterns

### Pattern 1: Simple Fixed-Size Pool

```typescript
import { KeyedPool, Effect, Scope } from "effect"

// Create a pool with fixed size for all keys
const httpClientPool = KeyedPool.make({
  acquire: (baseUrl: string) => Effect.gen(function* () {
    console.log(`Creating HTTP client for ${baseUrl}`)
    return {
      baseUrl,
      request: (path: string) => Effect.succeed(`Response from ${baseUrl}${path}`)
    }
  }),
  size: 5
})

// Usage within a scope
const makeRequest = (baseUrl: string, path: string) => Effect.gen(function* () {
  const client = yield* KeyedPool.get(httpClientPool, baseUrl)
  return yield* client.request(path)
}).pipe(
  Effect.scoped
)
```

### Pattern 2: Variable Pool Sizes by Key

```typescript
// Different pool sizes based on key characteristics
const adaptivePool = KeyedPool.makeWith({
  acquire: (config: { service: string; priority: "high" | "normal" }) => 
    createServiceConnection(config),
  size: (config) => config.priority === "high" ? 20 : 5
})

const useService = (service: string, priority: "high" | "normal") => Effect.gen(function* () {
  const connection = yield* KeyedPool.get(adaptivePool, { service, priority })
  return yield* connection.execute()
}).pipe(
  Effect.scoped
)
```

### Pattern 3: Time-to-Live Management

```typescript
// Pools with TTL for automatic resource refresh
const cachingPool = KeyedPool.makeWithTTL({
  acquire: (cacheKey: string) => createCacheConnection(cacheKey),
  min: 2,
  max: 10,
  timeToLive: "5 minutes"
})
```

## Real-World Examples

### Example 1: Multi-Region Database Connections

Managing database connections across different regions with automatic failover and resource pooling.

```typescript
import { KeyedPool, Effect, Scope, Layer } from "effect"

interface DatabaseConnection {
  readonly region: string
  readonly query: (sql: string) => Effect.Effect<unknown[], DatabaseError>
  readonly close: () => Effect.Effect<void>
}

class DatabaseError extends Error {
  readonly _tag = "DatabaseError"
}

// Create region-specific database pools
const createDatabasePool = KeyedPool.makeWith({
  acquire: (region: string) => Effect.gen(function* () {
    console.log(`Establishing connection to ${region} database`)
    
    // Simulate connection creation with potential failure
    if (Math.random() < 0.1) {
      yield* Effect.fail(new DatabaseError(`Failed to connect to ${region}`))
    }
    
    const connection: DatabaseConnection = {
      region,
      query: (sql: string) => Effect.gen(function* () {
        console.log(`Executing query in ${region}: ${sql}`)
        // Simulate query execution
        yield* Effect.sleep("100 millis")
        return [{ id: 1, name: "User" }]
      }),
      close: () => Effect.gen(function* () {
        console.log(`Closing connection to ${region}`)
      })
    }
    
    return connection
  }).pipe(
    Effect.acquireRelease(
      (connection) => connection.close()
    )
  ),
  size: (region) => region === "us-east-1" ? 20 : 10 // More connections for primary region
})

// Business logic using the pool
const getUsersByRegion = (region: string, userId: number) => Effect.gen(function* () {
  const db = yield* KeyedPool.get(createDatabasePool, region)
  const users = yield* db.query(`SELECT * FROM users WHERE id = ${userId}`)
  return users
}).pipe(
  Effect.scoped,
  Effect.retry({ times: 3 }), // Retry on connection failures
  Effect.timeout("30 seconds")
)

// Multi-region query with fallback
const getUserWithFallback = (userId: number) => Effect.gen(function* () {
  const primaryRegion = "us-east-1"
  const fallbackRegion = "us-west-2"
  
  return yield* getUsersByRegion(primaryRegion, userId).pipe(
    Effect.catchAll(() => getUsersByRegion(fallbackRegion, userId))
  )
})
```

### Example 2: HTTP Client Pool for Microservices

Managing HTTP clients for different microservices with load balancing and circuit breaking.

```typescript
import { KeyedPool, Effect, Scope, Schedule } from "effect"

interface ServiceConfig {
  readonly name: string
  readonly baseUrl: string
  readonly timeout: string
}

interface HttpClient {
  readonly config: ServiceConfig
  readonly get: (path: string) => Effect.Effect<unknown, HttpError>
  readonly post: (path: string, body: unknown) => Effect.Effect<unknown, HttpError>
}

class HttpError extends Error {
  readonly _tag = "HttpError"
  constructor(readonly status: number, message: string) {
    super(message)
  }
}

// Create service-specific HTTP client pools
const createHttpClientPool = KeyedPool.makeWithTTL({
  acquire: (config: ServiceConfig) => Effect.gen(function* () {
    console.log(`Creating HTTP client for ${config.name}`)
    
    const client: HttpClient = {
      config,
      get: (path: string) => Effect.gen(function* () {
        console.log(`GET ${config.baseUrl}${path}`)
        // Simulate HTTP request
        yield* Effect.sleep("200 millis")
        if (Math.random() < 0.05) {
          yield* Effect.fail(new HttpError(500, "Service unavailable"))
        }
        return { data: `Response from ${config.name}` }
      }),
      post: (path: string, body: unknown) => Effect.gen(function* () {
        console.log(`POST ${config.baseUrl}${path}`, body)
        yield* Effect.sleep("300 millis")
        return { success: true }
      })
    }
    
    return client
  }),
  min: 2,
  max: 15,
  timeToLive: "10 minutes" // Refresh clients periodically
})

// Service registry pattern
const serviceConfigs = {
  userService: { name: "user-service", baseUrl: "https://api.users.com", timeout: "5 seconds" },
  orderService: { name: "order-service", baseUrl: "https://api.orders.com", timeout: "10 seconds" },
  paymentService: { name: "payment-service", baseUrl: "https://api.payments.com", timeout: "15 seconds" }
} as const

type ServiceName = keyof typeof serviceConfigs

// Business operations using the pool
const callService = <T>(
  serviceName: ServiceName,
  operation: (client: HttpClient) => Effect.Effect<T, HttpError>
) => Effect.gen(function* () {
  const config = serviceConfigs[serviceName]
  const client = yield* KeyedPool.get(createHttpClientPool, config)
  return yield* operation(client)
}).pipe(
  Effect.scoped,
  Effect.retry(Schedule.exponential("1 second").pipe(Schedule.compose(Schedule.recurs(3)))),
  Effect.timeout("30 seconds")
)

// Example business logic
const processOrder = (userId: number, items: unknown[]) => Effect.gen(function* () {
  // Get user information
  const user = yield* callService("userService", (client) => 
    client.get(`/users/${userId}`)
  )
  
  // Create order
  const order = yield* callService("orderService", (client) =>
    client.post("/orders", { userId, items })
  )
  
  // Process payment
  const payment = yield* callService("paymentService", (client) =>
    client.post("/payments", { orderId: order.id, amount: 100 })
  )
  
  return { user, order, payment }
})
```

### Example 3: Redis Cache Pool with Cluster Support

Managing Redis connections across different clusters and databases with automatic failover.

```typescript
import { KeyedPool, Effect, Scope, Duration } from "effect"

interface RedisConfig {
  readonly cluster: string
  readonly database: number
  readonly keyPrefix?: string
}

interface RedisConnection {
  readonly config: RedisConfig
  readonly get: (key: string) => Effect.Effect<string | null, RedisError>
  readonly set: (key: string, value: string, ttl?: Duration.Duration) => Effect.Effect<void, RedisError>
  readonly del: (key: string) => Effect.Effect<boolean, RedisError>
  readonly pipeline: () => RedisPipeline
}

interface RedisPipeline {
  readonly get: (key: string) => RedisPipeline
  readonly set: (key: string, value: string) => RedisPipeline
  readonly exec: () => Effect.Effect<unknown[], RedisError>
}

class RedisError extends Error {
  readonly _tag = "RedisError"
}

// Create Redis connection pool by cluster and database
const createRedisPool = KeyedPool.makeWithTTLBy({
  acquire: (config: RedisConfig) => Effect.gen(function* () {
    console.log(`Connecting to Redis cluster: ${config.cluster}, DB: ${config.database}`)
    
    // Simulate connection with potential failure
    if (Math.random() < 0.02) {
      yield* Effect.fail(new RedisError(`Failed to connect to ${config.cluster}`))
    }
    
    const connection: RedisConnection = {
      config,
      get: (key: string) => Effect.gen(function* () {
        const fullKey = config.keyPrefix ? `${config.keyPrefix}:${key}` : key
        console.log(`GET ${fullKey} from ${config.cluster}`)
        yield* Effect.sleep("10 millis")
        return Math.random() > 0.5 ? `value-${key}` : null
      }),
      set: (key: string, value: string, ttl?: Duration.Duration) => Effect.gen(function* () {
        const fullKey = config.keyPrefix ? `${config.keyPrefix}:${key}` : key
        console.log(`SET ${fullKey} = ${value} in ${config.cluster}`)
        yield* Effect.sleep("15 millis")
      }),
      del: (key: string) => Effect.gen(function* () {
        const fullKey = config.keyPrefix ? `${config.keyPrefix}:${key}` : key
        console.log(`DEL ${fullKey} from ${config.cluster}`)
        yield* Effect.sleep("10 millis")
        return true
      }),
      pipeline: () => {
        const commands: { op: string; key: string; value?: string }[] = []
        
        const pipeline: RedisPipeline = {
          get: (key: string) => {
            commands.push({ op: "GET", key })
            return pipeline
          },
          set: (key: string, value: string) => {
            commands.push({ op: "SET", key, value })
            return pipeline
          },
          exec: () => Effect.gen(function* () {
            console.log(`Executing pipeline with ${commands.length} commands in ${config.cluster}`)
            yield* Effect.sleep("50 millis")
            return commands.map(() => "OK")
          })
        }
        
        return pipeline
      }
    }
    
    return connection
  }).pipe(
    Effect.acquireRelease(
      (connection) => Effect.gen(function* () {
        console.log(`Closing Redis connection to ${connection.config.cluster}`)
      })
    )
  ),
  min: (config) => config.cluster.includes("prod") ? 5 : 2,
  max: (config) => config.cluster.includes("prod") ? 25 : 10,
  timeToLive: (config) => 
    config.cluster.includes("prod") ? "30 minutes" : "10 minutes"
})

// Cache service implementation
const CacheService = {
  get: (cluster: string, database: number, key: string) => Effect.gen(function* () {
    const config: RedisConfig = { cluster, database }
    const redis = yield* KeyedPool.get(createRedisPool, config)
    return yield* redis.get(key)
  }).pipe(Effect.scoped),
  
  set: (cluster: string, database: number, key: string, value: string, ttl?: Duration.Duration) => Effect.gen(function* () {
    const config: RedisConfig = { cluster, database }
    const redis = yield* KeyedPool.get(createRedisPool, config)
    return yield* redis.set(key, value, ttl)
  }).pipe(Effect.scoped),
  
  mget: (cluster: string, database: number, keys: string[]) => Effect.gen(function* () {
    const config: RedisConfig = { cluster, database }
    const redis = yield* KeyedPool.get(createRedisPool, config)
    const pipeline = redis.pipeline()
    
    keys.forEach(key => pipeline.get(key))
    const results = yield* pipeline.exec()
    
    return keys.map((key, index) => ({
      key,
      value: results[index] as string | null
    }))
  }).pipe(Effect.scoped)
}

// Example usage with multiple clusters
const distributedCacheGet = (key: string) => Effect.gen(function* () {
  const primaryCluster = "redis-prod-us-east"
  const fallbackCluster = "redis-prod-us-west"
  
  // Try primary cluster first
  const primaryResult = yield* CacheService.get(primaryCluster, 0, key).pipe(
    Effect.catchAll(() => Effect.succeed(null))
  )
  
  if (primaryResult !== null) {
    return primaryResult
  }
  
  // Fallback to secondary cluster
  return yield* CacheService.get(fallbackCluster, 0, key)
})
```

## Advanced Features Deep Dive

### Feature 1: Pool Size Configuration

KeyedPool provides flexible pool sizing strategies for different use cases.

#### Basic Fixed Size

```typescript
// Same size for all keys
const fixedPool = KeyedPool.make({
  acquire: (key: string) => createResource(key),
  size: 10
})
```

#### Dynamic Size by Key

```typescript
// Different sizes based on key characteristics
const dynamicPool = KeyedPool.makeWith({
  acquire: (config: { service: string; tier: "premium" | "standard" }) => 
    createConnection(config),
  size: (config) => config.tier === "premium" ? 50 : 10
})
```

#### Real-World Dynamic Sizing Example

```typescript
interface ServiceConfig {
  readonly name: string
  readonly criticality: "critical" | "important" | "normal"
  readonly expectedLoad: "high" | "medium" | "low"
}

const intelligentPool = KeyedPool.makeWith({
  acquire: (config: ServiceConfig) => createServiceConnection(config),
  size: (config) => {
    const baseSize = {
      critical: 20,
      important: 10,
      normal: 5
    }[config.criticality]
    
    const loadMultiplier = {
      high: 2,
      medium: 1.5,
      low: 1
    }[config.expectedLoad]
    
    return Math.ceil(baseSize * loadMultiplier)
  }
})
```

### Feature 2: Time-to-Live Management

Automatic resource refresh and pool shrinking based on time and usage patterns.

#### Basic TTL Configuration

```typescript
const ttlPool = KeyedPool.makeWithTTL({
  acquire: (key: string) => createResource(key),
  min: 2,  // Minimum pool size
  max: 20, // Maximum pool size
  timeToLive: "15 minutes" // Resources older than this are refreshed
})
```

#### Advanced TTL with Key-Based Configuration

```typescript
interface CacheConfig {
  readonly region: string
  readonly dataType: "user" | "product" | "session"
}

const adaptiveTTLPool = KeyedPool.makeWithTTLBy({
  acquire: (config: CacheConfig) => createCacheConnection(config),
  min: (config) => config.dataType === "session" ? 5 : 2,
  max: (config) => config.dataType === "user" ? 50 : 20,
  timeToLive: (config) => {
    const baseTTL = {
      user: "1 hour",
      product: "30 minutes", 
      session: "5 minutes"
    }[config.dataType]
    
    // Shorter TTL for non-primary regions
    return config.region === "primary" ? baseTTL : "5 minutes"
  }
})
```

### Feature 3: Resource Invalidation and Health Checks

Proactive resource management with health monitoring and automatic invalidation.

```typescript
interface HealthCheckableResource {
  readonly id: string
  readonly isHealthy: () => Effect.Effect<boolean>
  readonly use: () => Effect.Effect<unknown>
}

const healthMonitoredPool = KeyedPool.make({
  acquire: (endpoint: string) => Effect.gen(function* () {
    const resource: HealthCheckableResource = {
      id: `resource-${Date.now()}`,
      isHealthy: () => Effect.gen(function* () {
        // Simulate health check
        yield* Effect.sleep("100 millis")
        return Math.random() > 0.1 // 90% healthy
      }),
      use: () => Effect.succeed("operation result")
    }
    return resource
  }),
  size: 10
})

// Health check pattern with automatic invalidation
const useHealthyResource = (endpoint: string) => Effect.gen(function* () {
  const resource = yield* KeyedPool.get(healthMonitoredPool, endpoint)
  
  const isHealthy = yield* resource.isHealthy()
  if (!isHealthy) {
    // Invalidate unhealthy resource
    yield* KeyedPool.invalidate(healthMonitoredPool, resource)
    // Get a fresh resource
    const freshResource = yield* KeyedPool.get(healthMonitoredPool, endpoint)
    return yield* freshResource.use()
  }
  
  return yield* resource.use()
}).pipe(
  Effect.scoped,
  Effect.retry({ times: 2 })
)
```

## Practical Patterns & Best Practices

### Pattern 1: Pool Monitoring and Metrics

```typescript
import { Ref, Effect, Scope } from "effect"

interface PoolMetrics {
  readonly totalAcquisitions: number
  readonly currentlyActive: number
  readonly errors: number
}

const createMonitoredPool = <K, A, E>(
  name: string,
  acquire: (key: K) => Effect.Effect<A, E, Scope.Scope>
) => Effect.gen(function* () {
  const metrics = yield* Ref.make<PoolMetrics>({
    totalAcquisitions: 0,
    currentlyActive: 0,
    errors: 0
  })
  
  const pool = yield* KeyedPool.make({
    acquire: (key: K) => acquire(key).pipe(
      Effect.tap(() => Ref.update(metrics, m => ({
        ...m,
        totalAcquisitions: m.totalAcquisitions + 1,
        currentlyActive: m.currentlyActive + 1
      }))),
      Effect.acquireRelease(() => 
        Ref.update(metrics, m => ({
          ...m,
          currentlyActive: m.currentlyActive - 1
        }))
      ),
      Effect.tapError(() => 
        Ref.update(metrics, m => ({ ...m, errors: m.errors + 1 }))
      )
    ),
    size: 10
  })
  
  const getMetrics = () => Ref.get(metrics)
  
  // Periodic metrics logging
  const startMetricsLogging = Effect.gen(function* () {
    while (true) {
      yield* Effect.sleep("30 seconds")
      const currentMetrics = yield* getMetrics()
      console.log(`Pool ${name} metrics:`, currentMetrics)
    }
  }).pipe(
    Effect.fork,
    Effect.scoped
  )
  
  return { pool, getMetrics, startMetricsLogging }
})
```

### Pattern 2: Circuit Breaker Integration

```typescript
interface CircuitBreakerState {
  readonly failures: number
  readonly lastFailure: number
  readonly state: "closed" | "open" | "half-open"
}

const createCircuitBreakerPool = <K, A, E>(
  acquire: (key: K) => Effect.Effect<A, E, Scope.Scope>,
  failureThreshold: number = 5,
  timeout: Duration.Duration = Duration.seconds(30)
) => Effect.gen(function* () {
  const circuitStates = yield* Ref.make(new Map<K, CircuitBreakerState>())
  
  const getCircuitState = (key: K) => Effect.gen(function* () {
    const states = yield* Ref.get(circuitStates)
    return states.get(key) ?? { failures: 0, lastFailure: 0, state: "closed" as const }
  })
  
  const updateCircuitState = (key: K, update: (state: CircuitBreakerState) => CircuitBreakerState) =>
    Ref.update(circuitStates, states => {
      const currentState = states.get(key) ?? { failures: 0, lastFailure: 0, state: "closed" as const }
      const newState = update(currentState)
      return new Map(states).set(key, newState)
    })
  
  const pool = yield* KeyedPool.make({
    acquire: (key: K) => Effect.gen(function* () {
      const state = yield* getCircuitState(key)
      const now = Date.now()
      
      // Check if circuit should be opened
      if (state.state === "open" && (now - state.lastFailure) < Duration.toMillis(timeout)) {
        yield* Effect.fail(new Error("Circuit breaker is open"))
      }
      
      // Try to acquire resource
      const resource = yield* acquire(key).pipe(
        Effect.tap(() => {
          // Reset on success
          if (state.failures > 0) {
            return updateCircuitState(key, () => ({ failures: 0, lastFailure: 0, state: "closed" }))
          }
          return Effect.void
        }),
        Effect.tapError(() => {
          return updateCircuitState(key, s => ({
            failures: s.failures + 1,
            lastFailure: now,
            state: s.failures + 1 >= failureThreshold ? "open" : s.state
          }))
        })
      )
      
      return resource
    }),
    size: 10
  })
  
  return pool
})
```

### Pattern 3: Resource Warm-up and Pre-loading

```typescript
const createPreloadedPool = <K extends string>(
  keys: readonly K[],
  acquire: (key: K) => Effect.Effect<unknown, unknown, Scope.Scope>
) => Effect.gen(function* () {
  const pool = yield* KeyedPool.make({ acquire, size: 5 })
  
  // Pre-warm pools for known keys
  const warmupEffect = Effect.forEach(
    keys,
    (key) => Effect.gen(function* () {
      console.log(`Pre-warming pool for key: ${key}`)
      // Acquire and immediately release to populate the pool
      yield* KeyedPool.get(pool, key).pipe(
        Effect.scoped,
        Effect.catchAll(() => Effect.void) // Ignore warm-up failures
      )
    }),
    { concurrency: "unbounded", discard: true }
  )
  
  // Start warm-up in background
  yield* Effect.fork(warmupEffect)
  
  return pool
})

// Usage example
const preloadedHttpPool = createPreloadedPool(
  ["api.users.com", "api.orders.com", "api.payments.com"] as const,
  (baseUrl) => createHttpClient(baseUrl)
)
```

## Integration Examples

### Integration with Effect Services

```typescript
import { Context, Layer, Effect } from "effect"

// Define service interfaces
interface DatabaseService {
  readonly getUser: (id: number) => Effect.Effect<User, DatabaseError>
  readonly createUser: (user: NewUser) => Effect.Effect<User, DatabaseError>
}

interface CacheService {
  readonly get: (key: string) => Effect.Effect<string | null, CacheError>
  readonly set: (key: string, value: string) => Effect.Effect<void, CacheError>
}

// Service tags
const DatabaseService = Context.GenericTag<DatabaseService>("DatabaseService")
const CacheService = Context.GenericTag<CacheService>("CacheService")

// Keyed pool-based implementations
const DatabaseServiceLive = Layer.scoped(
  DatabaseService,
  Effect.gen(function* () {
    const pool = yield* KeyedPool.make({
      acquire: (region: string) => createDatabaseConnection(region),
      size: 10
    })
    
    return DatabaseService.of({
      getUser: (id: number) => Effect.gen(function* () {
        const region = yield* getUserRegion(id)
        const db = yield* KeyedPool.get(pool, region)
        return yield* db.query(`SELECT * FROM users WHERE id = ${id}`)
      }).pipe(Effect.scoped),
      
      createUser: (user: NewUser) => Effect.gen(function* () {
        const region = yield* determineUserRegion(user)
        const db = yield* KeyedPool.get(pool, region)
        return yield* db.insert("users", user)
      }).pipe(Effect.scoped)
    })
  })
)

const CacheServiceLive = Layer.scoped(
  CacheService,
  Effect.gen(function* () {
    const pool = yield* KeyedPool.makeWithTTL({
      acquire: (cluster: string) => createCacheConnection(cluster),
      min: 2,
      max: 15,
      timeToLive: "10 minutes"
    })
    
    return CacheService.of({
      get: (key: string) => Effect.gen(function* () {
        const cluster = yield* getClusterForKey(key)
        const cache = yield* KeyedPool.get(pool, cluster)
        return yield* cache.get(key)
      }).pipe(Effect.scoped),
      
      set: (key: string, value: string) => Effect.gen(function* () {
        const cluster = yield* getClusterForKey(key)
        const cache = yield* KeyedPool.get(pool, cluster)
        return yield* cache.set(key, value)
      }).pipe(Effect.scoped)
    })
  })
)

// Combined application layer
const AppLayer = Layer.empty.pipe(
  Layer.provide(DatabaseServiceLive),
  Layer.provide(CacheServiceLive)
)
```

### Testing Strategies

```typescript
import { Effect, TestContext, Ref } from "effect"

// Mock resource for testing
interface MockResource {
  readonly id: string
  readonly calls: number
  readonly call: () => Effect.Effect<string>
}

// Test utilities
const createTestPool = (behavior: "success" | "failure" | "slow" = "success") => 
  KeyedPool.make({
    acquire: (key: string) => Effect.gen(function* () {
      const resource: MockResource = {
        id: `${key}-${Date.now()}`,
        calls: 0,
        call: () => {
          resource.calls++
          
          switch (behavior) {
            case "failure":
              return Effect.fail(new Error(`Test failure for ${key}`))
            case "slow":
              return Effect.sleep("1 second").pipe(Effect.as(`Result from ${key}`))
            default:
              return Effect.succeed(`Result from ${key}`)
          }
        }
      }
      
      return resource
    }),
    size: 3
  })

// Test resource reuse
const testResourceReuse = Effect.gen(function* () {
  const pool = yield* createTestPool()
  const resourceIds = new Set<string>()
  
  // Make multiple requests with the same key
  for (let i = 0; i < 10; i++) {
    yield* Effect.scoped(
      Effect.gen(function* () {
        const resource = yield* KeyedPool.get(pool, "test-key")
        resourceIds.add(resource.id)
        yield* resource.call()
      })
    )
  }
  
  // Should reuse resources (fewer than 10 unique IDs)
  console.log(`Created ${resourceIds.size} unique resources for 10 requests`)
  return resourceIds.size <= 3 // Pool size is 3
})

// Test error handling and recovery
const testErrorRecovery = Effect.gen(function* () {
  const pool = yield* createTestPool("failure")
  
  const result = yield* Effect.scoped(
    KeyedPool.get(pool, "error-key").pipe(
      Effect.flatMap(resource => resource.call()),
      Effect.retry({ times: 2 }),
      Effect.catchAll(() => Effect.succeed("Recovered"))
    )
  )
  
  console.log("Error recovery result:", result)
  return result === "Recovered"
})

// Performance testing
const testConcurrentAccess = Effect.gen(function* () {
  const pool = yield* createTestPool()
  const startTime = Date.now()
  
  // 100 concurrent requests
  yield* Effect.forEach(
    Array.from({ length: 100 }, (_, i) => i),
    (i) => Effect.scoped(
      Effect.gen(function* () {
        const resource = yield* KeyedPool.get(pool, `key-${i % 5}`) // 5 different keys
        return yield* resource.call()
      })
    ),
    { concurrency: "unbounded", discard: true }
  )
  
  const duration = Date.now() - startTime
  console.log(`100 concurrent requests completed in ${duration}ms`)
  return duration
})

// Test suite
const runPoolTests = Effect.gen(function* () {
  console.log("Testing resource reuse...")
  const reuseResult = yield* testResourceReuse
  
  console.log("Testing error recovery...")
  const errorResult = yield* testErrorRecovery
  
  console.log("Testing concurrent access...")
  const perfResult = yield* testConcurrentAccess
  
  return {
    resourceReuse: reuseResult,
    errorRecovery: errorResult,
    concurrentPerformance: perfResult
  }
}).pipe(
  Effect.scoped
)
```

## Conclusion

KeyedPool provides powerful key-based resource pooling with automatic lifecycle management, flexible configuration, and seamless integration with Effect's ecosystem.

Key benefits:
- **Efficient Resource Management**: Pools are created on-demand and sized appropriately per key
- **Automatic Cleanup**: Resources are properly disposed when scopes close
- **Type Safety**: Full type inference and compile-time safety for resource types
- **Flexible Configuration**: Support for fixed sizes, dynamic sizing, and TTL-based management
- **Error Resilience**: Built-in retry mechanisms and circuit breaker patterns

KeyedPool is ideal for managing resources that need to be organized by key, such as database connections by region, HTTP clients by service, or cache connections by cluster. It eliminates the complexity of manual pool management while providing the performance benefits of resource pooling.