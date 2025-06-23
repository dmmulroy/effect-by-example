# LayerMap: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem LayerMap Solves

Managing dynamic service configurations and runtime dependency injection can be challenging in large applications. Traditional approaches lead to several problems:

```typescript
// Traditional approach - static service configuration
class DatabaseService {
  private static connections = new Map<string, Database>()
  
  static async getConnection(tenant: string): Promise<Database> {
    // Manual connection management
    if (!this.connections.has(tenant)) {
      const config = await loadTenantConfig(tenant)
      const db = new Database(config)
      this.connections.set(tenant, db)
    }
    return this.connections.get(tenant)!
  }
  
  // Manual cleanup - easy to forget!
  static async cleanup() {
    for (const [, db] of this.connections) {
      await db.close()
    }
    this.connections.clear()
  }
}

// Problems with this approach:
// - No automatic resource cleanup
// - Difficult to test with different configurations
// - No composability with Effect's dependency system
// - Manual error handling
// - Memory leaks if cleanup is forgotten
```

This approach leads to:
- **Resource Leaks** - No automatic cleanup when services are no longer needed
- **Configuration Drift** - Hard to manage different service configurations dynamically
- **Testing Complexity** - Difficult to swap implementations for testing
- **Poor Composability** - Doesn't integrate with Effect's dependency system

### The LayerMap Solution

LayerMap provides a type-safe, resource-managed way to dynamically create and manage service layers based on keys.

```typescript
import { Context, Effect, Layer, LayerMap } from "effect"

// Define your service interface
class DatabaseService extends Context.Tag("DatabaseService")<DatabaseService, {
  readonly query: (sql: string) => Effect.Effect<any[], DatabaseError>
  readonly close: Effect.Effect<void>
}>() {}

// Create a LayerMap service for multi-tenant databases
class DatabaseMap extends LayerMap.Service<DatabaseMap>()("DatabaseMap", {
  lookup: (tenantId: string) =>
    Layer.scoped(
      DatabaseService,
      Effect.gen(function* () {
        const config = yield* loadTenantConfig(tenantId)
        const connection = yield* openDatabaseConnection(config)
        
        return {
          query: (sql: string) => executeQuery(connection, sql),
          close: closeDatabaseConnection(connection)
        }
      })
    ),
  idleTimeToLive: "5 minutes" // Automatic cleanup after 5 minutes of inactivity
}) {}
```

### Key Concepts

**Dynamic Layer Creation**: Layers are created on-demand based on keys, allowing runtime service configuration.

**Resource Management**: Built on RcMap (Reference Counted Map) for automatic resource acquisition and cleanup.

**Idle Time-to-Live**: Unused layers are automatically cleaned up after a configurable idle period.

## Basic Usage Patterns

### Pattern 1: Service Factory with LayerMap

```typescript
import { Context, Effect, Layer, LayerMap } from "effect"

// Define a service that needs different configurations
class CacheService extends Context.Tag("CacheService")<CacheService, {
  readonly get: (key: string) => Effect.Effect<string | null>
  readonly set: (key: string, value: string) => Effect.Effect<void>
}>() {}

// Create a LayerMap for different cache configurations
class CacheMap extends LayerMap.Service<CacheMap>()("CacheMap", {
  lookup: (region: string) =>
    Layer.succeed(CacheService, {
      get: (key: string) => 
        Effect.succeed(`cached-value-from-${region}-for-${key}`),
      set: (key: string, value: string) => 
        Effect.log(`Setting ${key}=${value} in ${region}`)
    }),
  idleTimeToLive: "10 minutes"
}) {}
```

### Pattern 2: Using the LayerMap

```typescript
// Use the LayerMap to provide region-specific cache services
const program = Effect.gen(function* () {
  // Access the cache service - it will be created for "us-east-1" if not exists
  const cache = yield* CacheService
  
  yield* cache.set("user:123", "john_doe")
  const value = yield* cache.get("user:123")
  
  yield* Effect.log(`Retrieved: ${value}`)
}).pipe(
  // Provide the specific region's cache layer
  Effect.provide(CacheMap.get("us-east-1"))
)

// Run with the CacheMap service
const runnable = program.pipe(
  Effect.provide(CacheMap.Default)
)
```

### Pattern 3: Runtime Layer Access

```typescript
// Access layers at runtime for advanced use cases
const dynamicProgram = Effect.gen(function* () {
  const cacheMap = yield* CacheMap
  
  // Get a runtime for a specific region
  const runtime = yield* cacheMap.runtime("eu-west-1")
  
  // Use the runtime directly
  const result = yield* Runtime.runSync(runtime)(
    Effect.gen(function* () {
      const cache = yield* CacheService
      return yield* cache.get("dynamic-key")
    })
  )
  
  yield* Effect.log(`Dynamic result: ${result}`)
})
```

## Real-World Examples

### Example 1: Multi-Tenant Database Access

Multi-tenant applications need isolated database access per tenant with automatic resource management.

```typescript
import { Context, Effect, Layer, LayerMap, Scope } from "effect"
import { NodeRuntime } from "@effect/platform-node"

// Database configuration and errors
interface DatabaseConfig {
  readonly host: string
  readonly port: number
  readonly database: string
  readonly credentials: {
    readonly username: string
    readonly password: string
  }
}

class DatabaseError extends Error {
  readonly _tag = "DatabaseError"
}

class ConfigError extends Error {
  readonly _tag = "ConfigError"
}

// Database service definition
class DatabaseService extends Context.Tag("DatabaseService")<DatabaseService, {
  readonly query: <A>(sql: string) => Effect.Effect<A[], DatabaseError>
  readonly transaction: <A, E, R>(
    effect: Effect.Effect<A, E, R>
  ) => Effect.Effect<A, E | DatabaseError, R>
}>() {}

// Configuration service
class ConfigService extends Context.Tag("ConfigService")<ConfigService, {
  readonly getTenantConfig: (tenantId: string) => Effect.Effect<DatabaseConfig, ConfigError>
}>() {}

// Multi-tenant database LayerMap
class TenantDatabaseMap extends LayerMap.Service<TenantDatabaseMap>()("TenantDatabaseMap", {
  lookup: (tenantId: string) =>
    Layer.scoped(
      DatabaseService,
      Effect.gen(function* () {
        // Load tenant-specific configuration
        const configService = yield* ConfigService
        const config = yield* configService.getTenantConfig(tenantId)
        
        // Create database connection with automatic cleanup
        const connection = yield* Effect.acquireRelease(
          Effect.gen(function* () {
            yield* Effect.log(`Opening database connection for tenant: ${tenantId}`)
            // Simulate database connection
            return {
              host: config.host,
              port: config.port,
              database: config.database,
              isConnected: true
            }
          }),
          (conn) => Effect.gen(function* () {
            yield* Effect.log(`Closing database connection for tenant: ${tenantId}`)
            // Simulate connection cleanup
          })
        )
        
        return {
          query: <A>(sql: string) =>
            Effect.gen(function* () {
              if (!connection.isConnected) {
                return yield* Effect.fail(new DatabaseError("Connection closed"))
              }
              yield* Effect.log(`Executing query for ${tenantId}: ${sql}`)
              // Simulate query execution
              return [] as A[]
            }),
          
          transaction: <A, E, R>(effect: Effect.Effect<A, E, R>) =>
            Effect.gen(function* () {
              yield* Effect.log(`Starting transaction for tenant: ${tenantId}`)
              const result = yield* effect
              yield* Effect.log(`Committing transaction for tenant: ${tenantId}`)
              return result
            }).pipe(
              Effect.catchAll((error) =>
                Effect.gen(function* () {
                  yield* Effect.log(`Rolling back transaction for tenant: ${tenantId}`)
                  return yield* Effect.fail(error)
                })
              )
            )
        }
      })
    ),
  
  // Clean up idle tenant connections after 2 minutes
  idleTimeToLive: "2 minutes",
  
  // Provide required dependencies
  dependencies: [
    Layer.succeed(ConfigService, {
      getTenantConfig: (tenantId: string) =>
        Effect.succeed({
          host: `${tenantId}-db.company.com`,
          port: 5432,
          database: `tenant_${tenantId}`,
          credentials: {
            username: `user_${tenantId}`,
            password: `password_${tenantId}`
          }
        })
    })
  ]
}) {}

// Business logic using tenant-specific database
const processUserData = (tenantId: string, userId: string) =>
  Effect.gen(function* () {
    const db = yield* DatabaseService
    
    // Query user data
    const users = yield* db.query<{ id: string; name: string }>(
      `SELECT id, name FROM users WHERE id = '${userId}'`
    )
    
    if (users.length === 0) {
      return yield* Effect.fail(new Error("User not found"))
    }
    
    // Update user in a transaction
    yield* db.transaction(
      Effect.gen(function* () {
        yield* db.query(`UPDATE users SET last_access = NOW() WHERE id = '${userId}'`)
        yield* db.query(`INSERT INTO access_log (user_id, timestamp) VALUES ('${userId}', NOW())`)
      })
    )
    
    return users[0]
  }).pipe(
    // Use tenant-specific database layer
    Effect.provide(TenantDatabaseMap.get(tenantId))
  )

// Example usage
const application = Effect.gen(function* () {
  // Process users from different tenants
  const tenant1User = yield* processUserData("tenant1", "user123")
  const tenant2User = yield* processUserData("tenant2", "user456")
  
  yield* Effect.log(`Processed users: ${tenant1User.name}, ${tenant2User.name}`)
}).pipe(
  Effect.provide(TenantDatabaseMap.Default),
  Effect.catchAll((error) => Effect.log(`Application error: ${error.message}`))
)
```

### Example 2: Feature Flag Service with Environment-Specific Configuration

Different environments (dev, staging, prod) need different feature flag configurations.

```typescript
import { Context, Effect, Layer, LayerMap } from "effect"

interface FeatureFlags {
  readonly enableNewUI: boolean
  readonly maxUploadSize: number
  readonly enableBetaFeatures: boolean
}

class FeatureFlagService extends Context.Tag("FeatureFlagService")<FeatureFlagService, {
  readonly getFlag: (flag: keyof FeatureFlags) => Effect.Effect<boolean | number>
  readonly getAllFlags: Effect.Effect<FeatureFlags>
  readonly refreshFlags: Effect.Effect<void>
}>() {}

// Environment-specific feature flag configurations
class EnvironmentFlagMap extends LayerMap.Service<EnvironmentFlagMap>()("EnvironmentFlagMap", {
  lookup: (environment: "development" | "staging" | "production") => {
    const configs: Record<typeof environment, FeatureFlags> = {
      development: {
        enableNewUI: true,
        maxUploadSize: 100_000_000, // 100MB for dev
        enableBetaFeatures: true
      },
      staging: {
        enableNewUI: true,
        maxUploadSize: 50_000_000, // 50MB for staging
        enableBetaFeatures: true
      },
      production: {
        enableNewUI: false, // Conservative in prod
        maxUploadSize: 10_000_000, // 10MB for prod
        enableBetaFeatures: false
      }
    }
    
    return Layer.succeed(FeatureFlagService, {
      getFlag: (flag: keyof FeatureFlags) =>
        Effect.succeed(configs[environment][flag]),
      
      getAllFlags: Effect.succeed(configs[environment]),
      
      refreshFlags: Effect.gen(function* () {
        yield* Effect.log(`Refreshing feature flags for ${environment}`)
        // In real implementation, this might fetch from a remote service
      })
    })
  },
  
  idleTimeToLive: "30 minutes"
}) {}

// Helper to determine current environment
const getCurrentEnvironment = Effect.gen(function* () {
  const env = process.env.NODE_ENV || "development"
  return env as "development" | "staging" | "production"
})

// Business logic that adapts based on feature flags
const uploadFile = (fileSize: number, fileName: string) =>
  Effect.gen(function* () {
    const flags = yield* FeatureFlagService
    
    // Check file size against environment-specific limit
    const maxSize = yield* flags.getFlag("maxUploadSize") as Effect.Effect<number>
    const maxSizeValue = yield* maxSize
    
    if (fileSize > maxSizeValue) {
      return yield* Effect.fail(
        new Error(`File ${fileName} (${fileSize} bytes) exceeds maximum size ${maxSizeValue} bytes`)
      )
    }
    
    // Use new UI if enabled
    const useNewUI = yield* flags.getFlag("enableNewUI") as Effect.Effect<boolean>
    const uiVersion = (yield* useNewUI) ? "v2" : "v1"
    
    yield* Effect.log(`Uploading ${fileName} using UI ${uiVersion}`)
    
    return {
      fileName,
      fileSize,
      uiVersion,
      uploadedAt: new Date().toISOString()
    }
  })

// Application that automatically uses environment-appropriate configuration
const fileUploadApp = Effect.gen(function* () {
  const environment = yield* getCurrentEnvironment
  
  // Upload files with environment-specific limits
  const result1 = yield* uploadFile(5_000_000, "document.pdf").pipe(
    Effect.provide(EnvironmentFlagMap.get(environment))
  )
  
  const result2 = yield* uploadFile(50_000_000, "video.mp4").pipe(
    Effect.provide(EnvironmentFlagMap.get(environment))
  )
  
  yield* Effect.log(`Upload results: ${JSON.stringify([result1, result2], null, 2)}`)
}).pipe(
  Effect.provide(EnvironmentFlagMap.Default),
  Effect.catchAll((error) => 
    Effect.log(`Upload failed: ${error.message}`)
  )
)
```

### Example 3: API Client Pool with Rate Limiting

Manage different API clients with per-client rate limiting and automatic connection pooling.

```typescript
import { Context, Effect, Layer, LayerMap, Schedule } from "effect"

interface APIResponse<T> {
  readonly data: T
  readonly status: number
  readonly headers: Record<string, string>
}

class RateLimitError extends Error {
  readonly _tag = "RateLimitError"
}

class APIClientService extends Context.Tag("APIClientService")<APIClientService, {
  readonly get: <T>(path: string) => Effect.Effect<APIResponse<T>, RateLimitError>
  readonly post: <T, B>(path: string, body: B) => Effect.Effect<APIResponse<T>, RateLimitError>
  readonly getRateLimit: Effect.Effect<{ remaining: number; resetAt: Date }>
}>() {}

// Configuration for different API clients
interface APIClientConfig {
  readonly baseUrl: string
  readonly apiKey: string
  readonly rateLimit: number // requests per minute
  readonly timeout: number // milliseconds
}

class APIClientMap extends LayerMap.Service<APIClientMap>()("APIClientMap", {
  lookup: (clientId: string) => {
    const configs: Record<string, APIClientConfig> = {
      "github": {
        baseUrl: "https://api.github.com",
        apiKey: process.env.GITHUB_TOKEN || "",
        rateLimit: 60, // 60 requests per minute
        timeout: 10000
      },
      "stripe": {
        baseUrl: "https://api.stripe.com/v1",
        apiKey: process.env.STRIPE_SECRET_KEY || "",
        rateLimit: 100, // 100 requests per minute
        timeout: 15000
      },
      "slack": {
        baseUrl: "https://slack.com/api",
        apiKey: process.env.SLACK_BOT_TOKEN || "",
        rateLimit: 50, // 50 requests per minute
        timeout: 8000
      }
    }
    
    const config = configs[clientId]
    if (!config) {
      return Layer.fail(new Error(`Unknown API client: ${clientId}`))
    }
    
    return Layer.scoped(
      APIClientService,
      Effect.gen(function* () {
        // Rate limiting state
        let requestCount = 0
        let windowStart = new Date()
        
        const checkRateLimit = Effect.gen(function* () {
          const now = new Date()
          const minutesSinceWindowStart = (now.getTime() - windowStart.getTime()) / (1000 * 60)
          
          // Reset window if more than a minute has passed
          if (minutesSinceWindowStart >= 1) {
            requestCount = 0
            windowStart = now
          }
          
          if (requestCount >= config.rateLimit) {
            const resetAt = new Date(windowStart.getTime() + 60 * 1000)
            return yield* Effect.fail(
              new RateLimitError(
                `Rate limit exceeded for ${clientId}. Reset at ${resetAt.toISOString()}`
              )
            )
          }
          
          requestCount++
        })
        
        // Simulated HTTP client
        const makeRequest = <T>(method: string, path: string, body?: any) =>
          Effect.gen(function* () {
            yield* checkRateLimit
            
            yield* Effect.log(`${method} ${config.baseUrl}${path}`)
            
            // Simulate network delay
            yield* Effect.sleep("100 millis")
            
            // Simulate response
            const response: APIResponse<T> = {
              data: {} as T,
              status: 200,
              headers: {
                "x-ratelimit-remaining": String(config.rateLimit - requestCount),
                "x-ratelimit-reset": String(Math.floor(windowStart.getTime() / 1000) + 60)
              }
            }
            
            return response
          }).pipe(
            Effect.timeout(`${config.timeout} millis`),
            Effect.retry(Schedule.exponential("1 second").pipe(Schedule.compose(Schedule.recurs(3))))
          )
        
        // Resource cleanup
        yield* Effect.addFinalizer(() =>
          Effect.log(`Cleaning up API client: ${clientId}`)
        )
        
        return {
          get: <T>(path: string) => makeRequest<T>("GET", path),
          post: <T, B>(path: string, body: B) => makeRequest<T>("POST", path, body),
          getRateLimit: Effect.succeed({
            remaining: config.rateLimit - requestCount,
            resetAt: new Date(windowStart.getTime() + 60 * 1000)
          })
        }
      })
    )
  },
  
  idleTimeToLive: "15 minutes"
}) {}

// Business logic using multiple API clients
const syncUserData = (userId: string) =>
  Effect.gen(function* () {
    // Get user from GitHub
    const githubUser = yield* Effect.gen(function* () {
      const client = yield* APIClientService
      return yield* client.get<{ login: string; email: string }>(`/users/${userId}`)
    }).pipe(
      Effect.provide(APIClientMap.get("github"))
    )
    
    // Create customer in Stripe
    const stripeCustomer = yield* Effect.gen(function* () {
      const client = yield* APIClientService
      return yield* client.post<{ id: string }, { email: string }>("/customers", {
        email: githubUser.data.email
      })
    }).pipe(
      Effect.provide(APIClientMap.get("stripe"))
    )
    
    // Send notification to Slack
    yield* Effect.gen(function* () {
      const client = yield* APIClientService
      yield* client.post<{}, { text: string }>("/chat.postMessage", {
        text: `New user synced: ${githubUser.data.login} (${stripeCustomer.data.id})`
      })
    }).pipe(
      Effect.provide(APIClientMap.get("slack"))
    )
    
    return {
      githubUser: githubUser.data,
      stripeCustomerId: stripeCustomer.data.id
    }
  })

// Monitor rate limits across all clients
const checkAllRateLimits = Effect.gen(function* () {
  const apiMap = yield* APIClientMap
  
  const clients = ["github", "stripe", "slack"]
  
  for (const clientId of clients) {
    const runtime = yield* apiMap.runtime(clientId)
    
    const rateLimit = yield* Runtime.runSync(runtime)(
      Effect.gen(function* () {
        const client = yield* APIClientService
        return yield* client.getRateLimit
      })
    )
    
    yield* Effect.log(
      `${clientId}: ${rateLimit.remaining} requests remaining, resets at ${rateLimit.resetAt.toISOString()}`
    )
  }
})

// Application demonstrating multiple API client usage
const apiIntegrationApp = Effect.gen(function* () {
  // Sync multiple users in parallel
  const syncResults = yield* Effect.all([
    syncUserData("user1"),
    syncUserData("user2"),
    syncUserData("user3")
  ], { concurrency: 2 })
  
  yield* Effect.log(`Synced ${syncResults.length} users`)
  
  // Check rate limit status
  yield* checkAllRateLimits
}).pipe(
  Effect.provide(APIClientMap.Default),
  Effect.catchAll((error) =>
    Effect.log(`API integration error: ${error.message}`)
  )
)
```

## Advanced Features Deep Dive

### Resource Management with RcMap

LayerMap is built on top of RcMap (Reference Counted Map), which provides sophisticated resource management:

```typescript
import { Context, Effect, Layer, LayerMap, Scope } from "effect"

class ExpensiveResource extends Context.Tag("ExpensiveResource")<ExpensiveResource, {
  readonly compute: (input: string) => Effect.Effect<string>
  readonly getStats: Effect.Effect<{ created: Date; usageCount: number }>
}>() {}

// LayerMap that creates expensive resources with tracking
class ResourceMap extends LayerMap.Service<ResourceMap>()("ResourceMap", {
  lookup: (key: string) =>
    Layer.scoped(
      ExpensiveResource,
      Effect.gen(function* () {
        const createdAt = new Date()
        let usageCount = 0
        
        yield* Effect.log(`Creating expensive resource for key: ${key}`)
        
        // Simulate expensive initialization
        yield* Effect.sleep("2 seconds")
        
        // Cleanup when resource is no longer needed
        yield* Effect.addFinalizer(() => 
          Effect.log(`Destroying expensive resource for key: ${key} (used ${usageCount} times)`)
        )
        
        return {
          compute: (input: string) =>
            Effect.gen(function* () {
              usageCount++
              yield* Effect.log(`Computing with resource ${key} (usage #${usageCount})`)
              return `${key}:${input}:computed`
            }),
          
          getStats: Effect.succeed({
            created: createdAt,
            usageCount
          })
        }
      })
    ),
  
  idleTimeToLive: "30 seconds" // Short TTL for demonstration
}) {}

// Demonstrate resource sharing and cleanup
const resourceSharingDemo = Effect.gen(function* () {
  const resourceMap = yield* ResourceMap
  
  // Multiple concurrent accesses to the same resource
  const results = yield* Effect.all([
    // These will share the same resource instance
    Effect.scoped(
      Effect.gen(function* () {
        const resource = yield* resourceMap.get("shared-key")
        return yield* Layer.build(resource).pipe(
          Effect.flatMap(() => ExpensiveResource),
          Effect.flatMap((r) => r.compute("task1"))
        )
      })
    ),
    
    Effect.scoped(
      Effect.gen(function* () {
        const resource = yield* resourceMap.get("shared-key")
        return yield* Layer.build(resource).pipe(
          Effect.flatMap(() => ExpensiveResource),
          Effect.flatMap((r) => r.compute("task2"))
        )
      })
    )
  ], { concurrency: 2 })
  
  yield* Effect.log(`Results: ${results.join(", ")}`)
  
  // Wait for resource to be cleaned up due to idle timeout
  yield* Effect.sleep("35 seconds")
  
  // This will create a new resource instance
  yield* Effect.scoped(
    Effect.gen(function* () {
      const resource = yield* resourceMap.get("shared-key")
      return yield* Layer.build(resource).pipe(
        Effect.flatMap(() => ExpensiveResource),
        Effect.flatMap((r) => r.compute("task3"))
      )
    })
  )
}).pipe(
  Effect.provide(ResourceMap.Default)
)
```

### Runtime Layer Access Patterns

LayerMap provides runtime access to individual layers, enabling advanced patterns:

```typescript
import { Context, Effect, Layer, LayerMap, Runtime } from "effect"

class MetricsService extends Context.Tag("MetricsService")<MetricsService, {
  readonly increment: (metric: string) => Effect.Effect<void>
  readonly gauge: (metric: string, value: number) => Effect.Effect<void>
  readonly getMetrics: Effect.Effect<Record<string, number>>
}>() {}

class MetricsMap extends LayerMap.Service<MetricsMap>()("MetricsMap", {
  lookup: (namespace: string) => {
    const metrics = new Map<string, number>()
    
    return Layer.succeed(MetricsService, {
      increment: (metric: string) =>
        Effect.sync(() => {
          const key = `${namespace}.${metric}`
          metrics.set(key, (metrics.get(key) || 0) + 1)
        }),
      
      gauge: (metric: string, value: number) =>
        Effect.sync(() => {
          const key = `${namespace}.${metric}`
          metrics.set(key, value)
        }),
      
      getMetrics: Effect.sync(() => Object.fromEntries(metrics))
    })
  }
}) {}

// Advanced pattern: Cross-namespace metric aggregation
const aggregateMetrics = Effect.gen(function* () {
  const metricsMap = yield* MetricsMap
  
  const namespaces = ["api", "database", "cache"]
  
  // Get runtimes for all namespaces
  const runtimes = yield* Effect.all(
    namespaces.map(ns => 
      Effect.map(metricsMap.runtime(ns), runtime => ({ namespace: ns, runtime }))
    )
  )
  
  // Collect metrics from all namespaces
  const allMetrics = yield* Effect.all(
    runtimes.map(({ namespace, runtime }) =>
      Runtime.runSync(runtime)(
        Effect.gen(function* () {
          const metrics = yield* MetricsService
          const data = yield* metrics.getMetrics
          return { namespace, data }
        })
      )
    )
  )
  
  // Aggregate metrics
  const aggregated = allMetrics.reduce((acc, { namespace, data }) => {
    Object.entries(data).forEach(([key, value]) => {
      acc[key] = (acc[key] || 0) + value
    })
    return acc
  }, {} as Record<string, number>)
  
  yield* Effect.log(`Aggregated metrics: ${JSON.stringify(aggregated, null, 2)}`)
  
  return aggregated
}).pipe(
  Effect.provide(MetricsMap.Default)
)
```

### Layer Invalidation and Refresh

Control layer lifecycle with manual invalidation:

```typescript
import { Context, Effect, Layer, LayerMap } from "effect"

class CacheService extends Context.Tag("CacheService")<CacheService, {
  readonly get: (key: string) => Effect.Effect<string | null>
  readonly set: (key: string, value: string) => Effect.Effect<void>
  readonly clear: Effect.Effect<void>
}>() {}

class CacheMap extends LayerMap.Service<CacheMap>()("CacheMap", {
  lookup: (region: string) => {
    const cache = new Map<string, string>()
    const createdAt = new Date()
    
    return Layer.succeed(CacheService, {
      get: (key: string) => 
        Effect.gen(function* () {
          const value = cache.get(key) || null
          yield* Effect.log(`[${region}] GET ${key} = ${value} (cache age: ${Date.now() - createdAt.getTime()}ms)`)
          return value
        }),
      
      set: (key: string, value: string) =>
        Effect.gen(function* () {
          cache.set(key, value)
          yield* Effect.log(`[${region}] SET ${key} = ${value}`)
        }),
      
      clear: Effect.gen(function* () {
        const size = cache.size
        cache.clear()
        yield* Effect.log(`[${region}] CLEAR (removed ${size} items)`)
      })
    })
  },
  
  idleTimeToLive: "1 minute"
}) {}

// Demonstrate cache invalidation and refresh
const cacheInvalidationDemo = Effect.gen(function* () {
  const cacheMap = yield* CacheMap
  
  // Use cache in us-east-1
  yield* Effect.gen(function* () {
    const cache = yield* CacheService
    yield* cache.set("user:123", "john")
    const value = yield* cache.get("user:123")
    yield* Effect.log(`Retrieved: ${value}`)
  }).pipe(
    Effect.provide(CacheMap.get("us-east-1"))
  )
  
  // Use the same cache again - should reuse the layer
  yield* Effect.gen(function* () {
    const cache = yield* CacheService
    const value = yield* cache.get("user:123")
    yield* Effect.log(`Retrieved again: ${value}`)
  }).pipe(
    Effect.provide(CacheMap.get("us-east-1"))
  )
  
  // Manually invalidate the us-east-1 cache
  yield* Effect.log("Invalidating us-east-1 cache...")
  yield* cacheMap.invalidate("us-east-1")
  
  // Next access will create a new cache instance
  yield* Effect.gen(function* () {
    const cache = yield* CacheService
    const value = yield* cache.get("user:123") // Will be null - new cache instance
    yield* Effect.log(`After invalidation: ${value}`)
    
    // Set the value again
    yield* cache.set("user:123", "john_refreshed")
    const newValue = yield* cache.get("user:123")
    yield* Effect.log(`New value: ${newValue}`)
  }).pipe(
    Effect.provide(CacheMap.get("us-east-1"))
  )
}).pipe(
  Effect.provide(CacheMap.Default)
)
```

## Practical Patterns & Best Practices

### Pattern 1: Configuration-Driven LayerMap

Create LayerMaps that read their configuration from external sources:

```typescript
import { Context, Effect, Layer, LayerMap, Config } from "effect"

// Service that varies by environment
class NotificationService extends Context.Tag("NotificationService")<NotificationService, {
  readonly sendEmail: (to: string, subject: string, body: string) => Effect.Effect<void>
  readonly sendSMS: (to: string, message: string) => Effect.Effect<void>
}>() {}

// Configuration for different environments
const notificationConfig = Config.nested("notifications")(
  Config.all({
    emailProvider: Config.string("email_provider"),
    smsProvider: Config.string("sms_provider"),
    enableEmail: Config.boolean("enable_email"),
    enableSMS: Config.boolean("enable_sms")
  })
)

class NotificationMap extends LayerMap.Service<NotificationMap>()("NotificationMap", {
  lookup: (environment: string) =>
    Layer.effect(
      NotificationService,
      Effect.gen(function* () {
        // Load configuration for this environment
        const config = yield* notificationConfig
        
        const emailEnabled = config.enableEmail
        const smsEnabled = config.enableSMS
        
        return {
          sendEmail: (to: string, subject: string, body: string) =>
            emailEnabled
              ? Effect.log(`[${environment}] Email to ${to}: ${subject}`)
              : Effect.log(`[${environment}] Email disabled, skipping: ${subject}`),
          
          sendSMS: (to: string, message: string) =>
            smsEnabled
              ? Effect.log(`[${environment}] SMS to ${to}: ${message}`)
              : Effect.log(`[${environment}] SMS disabled, skipping: ${message}`)
        }
      })
    )
}) {}

// Helper to automatically determine environment
const withEnvironmentNotifications = <A, E, R>(
  effect: Effect.Effect<A, E, R | NotificationService>
): Effect.Effect<A, E, R | NotificationMap> =>
  Effect.gen(function* () {
    const env = process.env.NODE_ENV || "development"
    return yield* effect.pipe(
      Effect.provide(NotificationMap.get(env))
    )
  })
```

### Pattern 2: Hierarchical LayerMap Dependencies

Create LayerMaps that depend on other LayerMaps:

```typescript
import { Context, Effect, Layer, LayerMap } from "effect"

// Base data layer
class DatabaseService extends Context.Tag("DatabaseService")<DatabaseService, {
  readonly query: (sql: string) => Effect.Effect<any[]>
}>() {}

class DatabaseMap extends LayerMap.Service<DatabaseMap>()("DatabaseMap", {
  lookup: (tenant: string) =>
    Layer.succeed(DatabaseService, {
      query: (sql: string) => 
        Effect.log(`[DB:${tenant}] ${sql}`).pipe(
          Effect.as([])
        )
    })
}) {}

// Business logic layer that depends on database layer
class UserService extends Context.Tag("UserService")<UserService, {
  readonly getUser: (id: string) => Effect.Effect<{ id: string; name: string } | null>
  readonly createUser: (name: string) => Effect.Effect<{ id: string; name: string }>
}>() {}

class UserServiceMap extends LayerMap.Service<UserServiceMap>()("UserServiceMap", {
  lookup: (tenant: string) =>
    Layer.effect(
      UserService,
      Effect.gen(function* () {
        const db = yield* DatabaseService
        
        return {
          getUser: (id: string) =>
            Effect.gen(function* () {
              const rows = yield* db.query(`SELECT * FROM users WHERE id = '${id}'`)
              return rows.length > 0 ? { id, name: `User${id}` } : null
            }),
          
          createUser: (name: string) =>
            Effect.gen(function* () {
              const id = Math.random().toString(36).substr(2, 9)
              yield* db.query(`INSERT INTO users (id, name) VALUES ('${id}', '${name}')`)
              return { id, name }
            })
        }
      })
    ).pipe(
      // Depend on the tenant-specific database
      Layer.provide(DatabaseMap.get(tenant))
    ),
  
  // Provide the database map as a dependency
  dependencies: [DatabaseMap.Default]
}) {}

// Usage with hierarchical dependencies
const userManagementDemo = Effect.gen(function* () {
  // Create a user in tenant1
  const newUser = yield* Effect.gen(function* () {
    const userService = yield* UserService
    return yield* userService.createUser("Alice")
  }).pipe(
    Effect.provide(UserServiceMap.get("tenant1"))
  )
  
  // Retrieve the user from tenant1
  const retrievedUser = yield* Effect.gen(function* () {
    const userService = yield* UserService
    return yield* userService.getUser(newUser.id)
  }).pipe(
    Effect.provide(UserServiceMap.get("tenant1"))
  )
  
  yield* Effect.log(`Created and retrieved user: ${JSON.stringify(retrievedUser)}`)
}).pipe(
  Effect.provide(UserServiceMap.Default)
)
```

### Pattern 3: Testing with LayerMap

Create test doubles using LayerMap for isolated testing:

```typescript
import { Context, Effect, Layer, LayerMap } from "effect"

// External service interface
class PaymentGatewayService extends Context.Tag("PaymentGatewayService")<PaymentGatewayService, {
  readonly processPayment: (amount: number, cardToken: string) => Effect.Effect<{ transactionId: string; status: "success" | "failed" }>
  readonly refundPayment: (transactionId: string) => Effect.Effect<{ refundId: string; status: "success" | "failed" }>
}>() {}

// Test implementations with different behaviors
class PaymentGatewayTestMap extends LayerMap.Service<PaymentGatewayTestMap>()("PaymentGatewayTestMap", {
  lookup: (scenario: "success" | "failure" | "timeout" | "partial_failure") => {
    switch (scenario) {
      case "success":
        return Layer.succeed(PaymentGatewayService, {
          processPayment: (amount: number, cardToken: string) =>
            Effect.succeed({
              transactionId: `txn_${Math.random().toString(36).substr(2, 9)}`,
              status: "success" as const
            }),
          
          refundPayment: (transactionId: string) =>
            Effect.succeed({
              refundId: `ref_${Math.random().toString(36).substr(2, 9)}`,
              status: "success" as const
            })
        })
      
      case "failure":
        return Layer.succeed(PaymentGatewayService, {
          processPayment: (amount: number, cardToken: string) =>
            Effect.succeed({
              transactionId: `txn_${Math.random().toString(36).substr(2, 9)}`,
              status: "failed" as const
            }),
          
          refundPayment: (transactionId: string) =>
            Effect.succeed({
              refundId: `ref_${Math.random().toString(36).substr(2, 9)}`,
              status: "failed" as const
            })
        })
      
      case "timeout":
        return Layer.succeed(PaymentGatewayService, {
          processPayment: (amount: number, cardToken: string) =>
            Effect.sleep("10 seconds").pipe(
              Effect.as({
                transactionId: `txn_timeout`,
                status: "failed" as const
              })
            ),
          
          refundPayment: (transactionId: string) =>
            Effect.sleep("10 seconds").pipe(
              Effect.as({
                refundId: `ref_timeout`,
                status: "failed" as const
              })
            )
        })
      
      case "partial_failure":
        let callCount = 0
        return Layer.succeed(PaymentGatewayService, {
          processPayment: (amount: number, cardToken: string) =>
            Effect.sync(() => {
              callCount++
              if (callCount === 1) {
                return {
                  transactionId: `txn_fail_${callCount}`,
                  status: "failed" as const
                }
              }
              return {
                transactionId: `txn_success_${callCount}`,
                status: "success" as const
              }
            }),
          
          refundPayment: (transactionId: string) =>
            Effect.succeed({
              refundId: `ref_${Math.random().toString(36).substr(2, 9)}`,
              status: "success" as const
            })
        })
      
      default:
        return Layer.fail(new Error(`Unknown test scenario: ${scenario}`))
    }
  }
}) {}

// Business logic to test
const processOrderPayment = (orderId: string, amount: number, cardToken: string) =>
  Effect.gen(function* () {
    const paymentGateway = yield* PaymentGatewayService
    
    yield* Effect.log(`Processing payment for order ${orderId}: $${amount}`)
    
    const result = yield* paymentGateway.processPayment(amount, cardToken)
    
    if (result.status === "success") {
      yield* Effect.log(`Payment successful: ${result.transactionId}`)
      return { orderId, transactionId: result.transactionId, status: "paid" as const }
    } else {
      yield* Effect.log(`Payment failed: ${result.transactionId}`)
      return { orderId, transactionId: result.transactionId, status: "failed" as const }
    }
  })

// Test suite using different scenarios
const paymentTestSuite = Effect.gen(function* () {
  // Test successful payment
  const successResult = yield* processOrderPayment("order1", 100, "card_123").pipe(
    Effect.provide(PaymentGatewayTestMap.get("success"))
  )
  
  // Test failed payment
  const failureResult = yield* processOrderPayment("order2", 200, "card_456").pipe(
    Effect.provide(PaymentGatewayTestMap.get("failure"))
  )
  
  // Test timeout with timeout handling
  const timeoutResult = yield* processOrderPayment("order3", 300, "card_789").pipe(
    Effect.provide(PaymentGatewayTestMap.get("timeout")),
    Effect.timeout("2 seconds"),
    Effect.catchAll(() => Effect.succeed({ orderId: "order3", transactionId: "timeout", status: "timeout" as const }))
  )
  
  // Test retry logic with partial failure
  const retryResult = yield* processOrderPayment("order4", 400, "card_retry").pipe(
    Effect.provide(PaymentGatewayTestMap.get("partial_failure")),
    Effect.retry({ times: 2 })
  )
  
  yield* Effect.log("Test Results:")
  yield* Effect.log(`Success: ${JSON.stringify(successResult)}`)
  yield* Effect.log(`Failure: ${JSON.stringify(failureResult)}`)
  yield* Effect.log(`Timeout: ${JSON.stringify(timeoutResult)}`)
  yield* Effect.log(`Retry: ${JSON.stringify(retryResult)}`)
}).pipe(
  Effect.provide(PaymentGatewayTestMap.Default)
)
```

## Integration Examples

### Integration with Effect Platform HTTP

Combine LayerMap with HTTP servers for tenant-specific routing:

```typescript
import { Context, Effect, Layer, LayerMap } from "effect"
import { HttpApi, HttpApp, HttpMiddleware, HttpRouter, HttpServer } from "@effect/platform"
import { NodeHttpServer } from "@effect/platform-node"

// Tenant-specific service
class TenantConfigService extends Context.Tag("TenantConfigService")<TenantConfigService, {
  readonly getConfig: Effect.Effect<{
    readonly name: string
    readonly features: string[]
    readonly limits: { readonly maxUsers: number; readonly maxRequests: number }
  }>
}>() {}

class TenantConfigMap extends LayerMap.Service<TenantConfigMap>()("TenantConfigMap", {
  lookup: (tenantId: string) => {
    const configs = {
      "startup": { name: "Startup Plan", features: ["basic"], limits: { maxUsers: 10, maxRequests: 1000 } },
      "business": { name: "Business Plan", features: ["basic", "advanced"], limits: { maxUsers: 100, maxRequests: 10000 } },
      "enterprise": { name: "Enterprise Plan", features: ["basic", "advanced", "premium"], limits: { maxUsers: 1000, maxRequests: 100000 } }
    }
    
    const config = configs[tenantId as keyof typeof configs] || configs.startup
    
    return Layer.succeed(TenantConfigService, {
      getConfig: Effect.succeed(config)
    })
  }
}) {}

// HTTP API that uses tenant-specific configuration
const tenantApi = HttpApi.make("tenant-api").pipe(
  HttpApi.get("config", "/config", {
    response: {
      200: {
        name: "string",
        features: ["string"],
        limits: { maxUsers: "number", maxRequests: "number" }
      }
    }
  })
)

const tenantRouter = HttpRouter.make(
  HttpApi.implement(tenantApi)({
    config: Effect.gen(function* () {
      const configService = yield* TenantConfigService
      return yield* configService.getConfig
    })
  })
)

// Middleware to extract tenant ID and provide appropriate layer
const tenantMiddleware = HttpMiddleware.make((app) =>
  Effect.gen(function* () {
    const request = yield* HttpRouter.currentRequest
    const tenantId = request.headers["x-tenant-id"] || "startup"
    
    // Provide tenant-specific layer to the request
    return yield* app.pipe(
      Effect.provide(TenantConfigMap.get(tenantId))
    )
  })
)

// Main HTTP application
const httpApp = HttpApp.empty.pipe(
  HttpApp.mountApp("/api", tenantRouter),
  HttpApp.use(tenantMiddleware)
)

// Server with LayerMap support
const server = HttpServer.serve(httpApp).pipe(
  Effect.provide(TenantConfigMap.Default),
  Effect.provide(NodeHttpServer.layer({ port: 3000 }))
)
```

### Integration with Streams and Subscriptions

Use LayerMap for per-subscription stream processing:

```typescript
import { Context, Effect, Layer, LayerMap, Stream, Queue } from "effect"

interface Event {
  readonly id: string
  readonly type: string
  readonly data: any
  readonly timestamp: Date
}

class EventStreamService extends Context.Tag("EventStreamService")<EventStreamService, {
  readonly subscribe: (filter: (event: Event) => boolean) => Stream.Stream<Event>
  readonly publish: (event: Event) => Effect.Effect<void>
}>() {}

class EventStreamMap extends LayerMap.Service<EventStreamMap>()("EventStreamMap", {
  lookup: (streamId: string) =>
    Layer.scoped(
      EventStreamService,
      Effect.gen(function* () {
        // Create a queue for this stream
        const eventQueue = yield* Queue.unbounded<Event>()
        const subscribers = new Set<Queue.Enqueue<Event>>()
        
        yield* Effect.log(`Creating event stream: ${streamId}`)
        
        // Cleanup on scope close
        yield* Effect.addFinalizer(() =>
          Effect.gen(function* () {
            yield* Effect.log(`Closing event stream: ${streamId}`)
            yield* Queue.shutdown(eventQueue)
          })
        )
        
        return {
          subscribe: (filter: (event: Event) => boolean) =>
            Stream.fromQueue(eventQueue).pipe(
              Stream.filter(filter),
              Stream.tap((event) => Effect.log(`[${streamId}] Event: ${event.type}`))
            ),
          
          publish: (event: Event) =>
            Effect.gen(function* () {
              yield* Effect.log(`[${streamId}] Publishing: ${event.type}`)
              yield* Queue.offer(eventQueue, event)
            })
        }
      })
    )
}) {}

// Example: Multi-room chat system
const chatDemo = Effect.gen(function* () {
  const streamMap = yield* EventStreamMap
  
  // Create event streams for different chat rooms
  const room1Events = yield* streamMap.get("room1")
  const room2Events = yield* streamMap.get("room2")
  
  // Subscribe to messages in room1
  const room1Subscription = Stream.runForeach(
    Layer.build(room1Events).pipe(
      Effect.flatMap(() => EventStreamService),
      Effect.map((service) => service.subscribe((event) => event.type === "message")),
      Stream.unwrap
    ),
    (event) => Effect.log(`Room1 Message: ${event.data.text}`)
  ).pipe(
    Effect.fork
  )
  
  // Subscribe to all events in room2
  const room2Subscription = Stream.runForeach(
    Layer.build(room2Events).pipe(
      Effect.flatMap(() => EventStreamService),
      Effect.map((service) => service.subscribe(() => true)),
      Stream.unwrap
    ),
    (event) => Effect.log(`Room2 Event: ${event.type}`)
  ).pipe(
    Effect.fork
  )
  
  // Publish some events
  yield* Effect.gen(function* () {
    const room1Service = yield* EventStreamService
    yield* room1Service.publish({
      id: "1",
      type: "message",
      data: { text: "Hello room1!" },
      timestamp: new Date()
    })
  }).pipe(
    Effect.provide(room1Events)
  )
  
  yield* Effect.gen(function* () {
    const room2Service = yield* EventStreamService
    yield* room2Service.publish({
      id: "2",
      type: "user-joined",
      data: { userId: "user123" },
      timestamp: new Date()
    })
  }).pipe(
    Effect.provide(room2Events)
  )
  
  // Let events process
  yield* Effect.sleep("1 second")
  
  // Clean up subscriptions
  yield* room1Subscription.pipe(Effect.flatMap(Fiber.interrupt))
  yield* room2Subscription.pipe(Effect.flatMap(Fiber.interrupt))
}).pipe(
  Effect.provide(EventStreamMap.Default)
)
```

## Conclusion

LayerMap provides **dynamic service management**, **automatic resource cleanup**, and **type-safe dependency injection** for complex Effect applications.

Key benefits:
- **Dynamic Configuration**: Create services based on runtime keys (tenants, environments, regions)
- **Resource Management**: Automatic cleanup with reference counting and idle timeouts
- **Type Safety**: Full TypeScript integration with Effect's type system
- **Composability**: Seamless integration with Effect's Layer and Context systems
- **Testing**: Easy test double injection for different scenarios

LayerMap is ideal when you need to manage multiple variants of the same service type, handle multi-tenant applications, or implement configuration-driven dependency injection patterns.