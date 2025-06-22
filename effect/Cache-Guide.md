# Cache: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem Cache Solves

In modern applications, expensive operations like database queries, API calls, and computations are often repeated unnecessarily. Without proper caching, applications suffer from performance bottlenecks, redundant work, and resource waste.

```typescript
// Traditional approach - problematic and inefficient
class UserService {
  private cache = new Map<string, User>()
  
  async getUser(id: string): Promise<User> {
    // No deduplication - concurrent requests cause multiple DB calls
    if (!this.cache.has(id)) {
      const user = await this.database.findUser(id) // Multiple concurrent calls!
      this.cache.set(id, user)
      return user
    }
    
    // No TTL - stale data persists indefinitely
    return this.cache.get(id)!
  }
  
  // Manual cache management - error-prone
  invalidateUser(id: string) {
    this.cache.delete(id)
  }
}
```

This approach leads to:
- **Race Conditions** - Multiple concurrent requests for the same key cause duplicate work
- **Memory Leaks** - No automatic eviction based on capacity or TTL
- **Stale Data** - No built-in expiration mechanism
- **Error Handling** - Failed lookups aren't properly cached or handled
- **Manual Management** - Complex invalidation logic scattered throughout code

### The Cache Solution

Effect's Cache module provides a concurrent-safe, composable caching solution with automatic deduplication, TTL support, and comprehensive error handling.

```typescript
import { Cache, Effect, Duration } from "effect"

// Declarative cache with automatic deduplication and TTL
const userCache = Cache.make({
  capacity: 1000,
  timeToLive: Duration.minutes(5),
  lookup: (id: string) => Effect.gen(function* () {
    // Only computed once per key, even with concurrent access
    const user = yield* Database.findUser(id)
    return user
  })
})

const program = Effect.gen(function* () {
  const cache = yield* userCache
  
  // Concurrent requests automatically deduplicated
  const users = yield* Effect.all([
    cache.get("user-123"),
    cache.get("user-123"),
    cache.get("user-123")
  ], { concurrency: "unbounded" })
  
  // Only one database call made, result shared across all requests
  console.log("All users:", users) // [User, User, User] - same instance
})
```

### Key Concepts

**Cache<Key, Value, Error>**: A thread-safe, concurrent cache that deduplicates requests and manages value lifecycles with TTL and capacity constraints.

**Lookup Function**: A function that computes cache values on-demand, returning an Effect that can handle both sync and async operations with proper error handling.

**Deduplication**: Multiple concurrent requests for the same key result in only one lookup computation, with all requesters receiving the same result.

**TTL (Time To Live)**: Automatic expiration of cached values after a specified duration, ensuring data freshness.

**LRU Eviction**: Least Recently Used eviction policy when cache reaches capacity, automatically managing memory usage.

## Basic Usage Patterns

### Pattern 1: Simple Cache Creation

```typescript
import { Cache, Effect, Duration } from "effect"

// Create a basic cache for expensive computations
const fibonacciCache = Cache.make({
  capacity: 100,
  timeToLive: Duration.hours(1),
  lookup: (n: number) => Effect.gen(function* () {
    // Simulate expensive computation
    yield* Effect.sleep(Duration.milliseconds(100))
    
    if (n <= 1) return n
    
    const cache = yield* fibonacciCache
    const prev1 = yield* cache.get(n - 1)
    const prev2 = yield* cache.get(n - 2)
    
    return prev1 + prev2
  })
})

const program = Effect.gen(function* () {
  const cache = yield* fibonacciCache
  
  const result = yield* cache.get(10)
  console.log(`Fibonacci(10) = ${result}`)
  
  // Subsequent calls are instant (cached)
  const cachedResult = yield* cache.get(10)
  console.log(`Cached result: ${cachedResult}`)
})
```

### Pattern 2: Database Entity Caching

```typescript
import { Cache, Effect, Duration } from "effect"

interface User {
  readonly id: string
  readonly name: string
  readonly email: string
}

// Cache with service dependency
const createUserCache = (database: Database) => Cache.make({
  capacity: 1000,
  timeToLive: Duration.minutes(10),
  lookup: (userId: string) => Effect.gen(function* () {
    console.log(`Loading user ${userId} from database`)
    const user = yield* database.users.findById(userId)
    
    if (!user) {
      return yield* Effect.fail(new UserNotFoundError(userId))
    }
    
    return user
  })
})

const program = Effect.gen(function* () {
  const database = yield* Database
  const userCache = yield* createUserCache(database)
  
  // First call hits database
  const user1 = yield* userCache.get("user-123")
  
  // Second call uses cache
  const user2 = yield* userCache.get("user-123")
  
  console.log("Same instance:", user1 === user2) // true
})
```

### Pattern 3: API Response Caching

```typescript
import { Cache, Effect, Duration, HttpClient } from "effect"

interface WeatherData {
  readonly city: string
  readonly temperature: number
  readonly conditions: string
}

const weatherCache = Cache.make({
  capacity: 50,
  timeToLive: Duration.minutes(15), // Weather data expires in 15 minutes
  lookup: (cityCode: string) => Effect.gen(function* () {
    const client = yield* HttpClient.HttpClient
    
    const response = yield* client.get(`/weather/${cityCode}`).pipe(
      Effect.flatMap(HttpClient.response.json),
      Effect.mapError(error => new WeatherApiError(cityCode, error))
    )
    
    return response as WeatherData
  })
})

const getWeather = (city: string) => Effect.gen(function* () {
  const cache = yield* weatherCache
  return yield* cache.get(city)
})
```

## Real-World Examples

### Example 1: User Profile Service with Permissions

```typescript
import { Cache, Effect, Duration, Layer } from "effect"

interface UserProfile {
  readonly id: string
  readonly name: string
  readonly role: string
  readonly permissions: ReadonlyArray<string>
}

class UserNotFoundError extends Error {
  readonly _tag = "UserNotFoundError"
  constructor(readonly userId: string) {
    super(`User not found: ${userId}`)
  }
}

class PermissionDeniedError extends Error {
  readonly _tag = "PermissionDeniedError"
  constructor(readonly userId: string, readonly permission: string) {
    super(`User ${userId} lacks permission: ${permission}`)
  }
}

// Service interfaces
interface Database {
  readonly users: {
    readonly findById: (id: string) => Effect.Effect<UserProfile | null, DatabaseError>
  }
}
const Database = Context.GenericTag<Database>("Database")

interface Logger {
  readonly info: (message: string) => Effect.Effect<void>
}
const Logger = Context.GenericTag<Logger>("Logger")

// Cache factory with proper service dependencies
const makeUserProfileCache = Effect.gen(function* () {
  const database = yield* Database
  const logger = yield* Logger
  
  return yield* Cache.make({
    capacity: 5000,
    timeToLive: Duration.minutes(30),
    lookup: (userId: string) => Effect.gen(function* () {
      yield* logger.info(`Loading user profile: ${userId}`)
      
      const profile = yield* database.users.findById(userId).pipe(
        Effect.mapError(error => new DatabaseError(`Failed to load user ${userId}`, error))
      )
      
      if (!profile) {
        return yield* Effect.fail(new UserNotFoundError(userId))
      }
      
      yield* logger.info(`User profile loaded: ${userId}`)
      return profile
    })
  })
})

// Business logic using the cache
const getUserWithPermissionCheck = (userId: string, requiredPermission: string) => 
  Effect.gen(function* () {
    const cache = yield* makeUserProfileCache
    const profile = yield* cache.get(userId)
    
    if (!profile.permissions.includes(requiredPermission)) {
      return yield* Effect.fail(new PermissionDeniedError(userId, requiredPermission))
    }
    
    return profile
  })

// Usage in request handler
const handleUserRequest = (userId: string) => Effect.gen(function* () {
  // Multiple permission checks use cached profile
  const profile = yield* getUserWithPermissionCheck(userId, "read:profile")
  const canEdit = yield* Effect.either(
    getUserWithPermissionCheck(userId, "edit:profile")
  )
  
  return {
    profile,
    canEdit: canEdit._tag === "Right"
  }
}).pipe(
  Effect.catchTag("UserNotFoundError", () => 
    Effect.succeed({ error: "User not found" })
  ),
  Effect.catchTag("PermissionDeniedError", () => 
    Effect.succeed({ error: "Access denied" })
  )
)
```

### Example 2: Multi-Level Cache with Fallback Strategy

```typescript
import { Cache, Effect, Duration, Option } from "effect"

interface ProductData {
  readonly id: string
  readonly name: string
  readonly price: number
  readonly inventory: number
}

// L1 Cache: In-memory, very fast, short TTL
const l1ProductCache = Cache.make({
  capacity: 100, // Small capacity for hot items
  timeToLive: Duration.minutes(2),
  lookup: (productId: string) => Effect.gen(function* () {
    console.log(`L1 Cache miss: ${productId}`)
    const l2Cache = yield* l2ProductCache
    return yield* l2Cache.get(productId)
  })
})

// L2 Cache: Redis-backed, medium TTL
const l2ProductCache = Cache.make({
  capacity: 10000,
  timeToLive: Duration.minutes(15),
  lookup: (productId: string) => Effect.gen(function* () {
    console.log(`L2 Cache miss: ${productId}`)
    const redis = yield* Redis
    
    // Try Redis first
    const cached = yield* redis.get(`product:${productId}`).pipe(
      Effect.map(Option.fromNullable),
      Effect.map(Option.map(JSON.parse))
    )
    
    if (Option.isSome(cached)) {
      console.log(`Redis hit: ${productId}`)
      return cached.value as ProductData
    }
    
    // Fallback to database
    const database = yield* Database
    const product = yield* database.products.findById(productId)
    
    if (!product) {
      return yield* Effect.fail(new ProductNotFoundError(productId))
    }
    
    // Warm Redis cache
    yield* redis.set(
      `product:${productId}`,
      JSON.stringify(product),
      Duration.minutes(30)
    )
    
    return product
  })
})

// Business logic with automatic cache warming
const getProduct = (productId: string) => Effect.gen(function* () {
  const l1Cache = yield* l1ProductCache
  return yield* l1Cache.get(productId)
})

const getProductsWithInventoryCheck = (productIds: ReadonlyArray<string>) =>
  Effect.gen(function* () {
    // Fetch all products concurrently (automatically deduplicated)
    const products = yield* Effect.all(
      productIds.map(getProduct),
      { concurrency: "unbounded" }
    )
    
    // Filter products with available inventory
    return products.filter(product => product.inventory > 0)
  }).pipe(
    Effect.catchTag("ProductNotFoundError", () => Effect.succeed([]))
  )
```

### Example 3: Configuration Cache with Hot Reloading

```typescript
import { Cache, Effect, Duration, Schedule, Ref } from "effect"

interface AppConfig {
  readonly features: {
    readonly enableNewUI: boolean
    readonly maxFileSize: number
    readonly allowedDomains: ReadonlyArray<string>
  }
  readonly limits: {
    readonly rateLimit: number
    readonly maxConnections: number
  }
  readonly version: string
}

// Versioned configuration cache
const configCache = Effect.gen(function* () {
  const versionRef = yield* Ref.make(0)
  
  return yield* Cache.makeWith({
    capacity: 10,
    lookup: (configKey: string) => Effect.gen(function* () {
      const configService = yield* ConfigService
      const logger = yield* Logger
      
      yield* logger.info(`Loading configuration: ${configKey}`)
      
      const config = yield* configService.loadConfig(configKey)
      const currentVersion = yield* Ref.get(versionRef)
      
      yield* logger.info(`Configuration loaded: ${configKey} (v${currentVersion})`)
      
      return { ...config, version: currentVersion.toString() }
    }),
    timeToLive: (exit) => {
      // Shorter TTL for failed configurations
      return Exit.isSuccess(exit) 
        ? Duration.minutes(10)
        : Duration.seconds(30)
    }
  })
})

// Hot reload mechanism
const startConfigWatcher = Effect.gen(function* () {
  const cache = yield* configCache
  const configService = yield* ConfigService
  const logger = yield* Logger
  
  // Watch for configuration changes
  const watchChanges = Effect.gen(function* () {
    const hasChanges = yield* configService.checkForChanges()
    
    if (hasChanges) {
      yield* logger.info("Configuration changes detected, invalidating cache")
      yield* cache.invalidateAll
    }
  })
  
  // Schedule periodic checks
  return yield* watchChanges.pipe(
    Effect.repeat(Schedule.fixed(Duration.seconds(30))),
    Effect.fork
  )
})

// Helper for feature flag checking with caching
const isFeatureEnabled = (feature: keyof AppConfig["features"]) =>
  Effect.gen(function* () {
    const cache = yield* configCache
    const config = yield* cache.get("app-config")
    
    return config.features[feature]
  })

// Usage in request handling
const handleFileUpload = (file: File) => Effect.gen(function* () {
  const maxSize = yield* Effect.gen(function* () {
    const cache = yield* configCache
    const config = yield* cache.get("app-config")
    return config.features.maxFileSize
  })
  
  if (file.size > maxSize) {
    return yield* Effect.fail(new FileTooLargeError(file.size, maxSize))
  }
  
  const newUIEnabled = yield* isFeatureEnabled("enableNewUI")
  
  return {
    uploadResult: yield* processFileUpload(file),
    useNewUI: newUIEnabled
  }
})
```

## Advanced Features Deep Dive

### Feature 1: Cache Statistics and Monitoring

Cache statistics provide valuable insights into cache performance and help optimize cache configuration.

#### Basic Statistics Usage

```typescript
import { Cache, Effect, Duration } from "effect"

const monitoredCache = Cache.make({
  capacity: 1000,
  timeToLive: Duration.minutes(10),
  lookup: (key: string) => Effect.succeed(`value-${key}`)
})

const analyzeCachePerformance = Effect.gen(function* () {
  const cache = yield* monitoredCache
  
  // Perform some cache operations
  yield* Effect.all([
    cache.get("key1"),
    cache.get("key2"),
    cache.get("key1"), // This will be a cache hit
    cache.get("key3")
  ])
  
  // Get comprehensive statistics
  const stats = yield* cache.cacheStats
  
  const hitRate = stats.hits / (stats.hits + stats.misses)
  
  console.log(`Cache Statistics:`)
  console.log(`- Total entries: ${stats.size}`)
  console.log(`- Cache hits: ${stats.hits}`)
  console.log(`- Cache misses: ${stats.misses}`)
  console.log(`- Hit rate: ${(hitRate * 100).toFixed(2)}%`)
  
  return stats
})
```

#### Real-World Statistics with Alerting

```typescript
import { Cache, Effect, Duration, Schedule, Ref } from "effect"

// Statistics tracker with alerting
const createMonitoredCache = <K, V, E>(
  name: string,
  options: {
    capacity: number
    timeToLive: Duration.DurationInput
    lookup: (key: K) => Effect.Effect<V, E>
  }
) => Effect.gen(function* () {
  const cache = yield* Cache.make(options)
  const metricsRef = yield* Ref.make({
    totalRequests: 0,
    averageHitRate: 0,
    lastAlertTime: 0
  })
  
  const monitoredGet = (key: K) => Effect.gen(function* () {
    const result = yield* cache.get(key)
    
    // Update metrics
    yield* Ref.update(metricsRef, metrics => ({
      ...metrics,
      totalRequests: metrics.totalRequests + 1
    }))
    
    return result
  })
  
  const checkCacheHealth = Effect.gen(function* () {
    const stats = yield* cache.cacheStats
    const metrics = yield* Ref.get(metricsRef)
    
    const hitRate = stats.hits / (stats.hits + stats.misses)
    const now = Date.now()
    
    // Alert if hit rate drops below threshold
    if (hitRate < 0.7 && now - metrics.lastAlertTime > 300000) { // 5 minutes
      const logger = yield* Logger
      yield* logger.warn(
        `Cache ${name} hit rate low: ${(hitRate * 100).toFixed(2)}%`
      )
      
      yield* Ref.update(metricsRef, m => ({ ...m, lastAlertTime: now }))
    }
    
    yield* Ref.update(metricsRef, m => ({
      ...m,
      averageHitRate: (m.averageHitRate + hitRate) / 2
    }))
  })
  
  // Start health monitoring
  const monitor = yield* checkCacheHealth.pipe(
    Effect.repeat(Schedule.fixed(Duration.seconds(60))),
    Effect.fork
  )
  
  return { get: monitoredGet, cache, monitor, metrics: metricsRef }
})
```

### Feature 2: Advanced TTL Strategies

Effect Cache supports dynamic TTL based on the result of lookup operations, enabling sophisticated caching strategies.

#### TTL Based on Success/Failure

```typescript
import { Cache, Effect, Duration, Exit } from "effect"

// Different TTL for successful vs failed lookups
const resilientApiCache = Cache.makeWith({
  capacity: 1000,
  lookup: (endpoint: string) => Effect.gen(function* () {
    const httpClient = yield* HttpClient.HttpClient
    
    const response = yield* httpClient.get(endpoint).pipe(
      Effect.timeout(Duration.seconds(5)),
      Effect.retry(Schedule.exponential(Duration.milliseconds(100)).pipe(
        Schedule.recurs(3)
      ))
    )
    
    return yield* HttpClient.response.json(response)
  }),
  timeToLive: (exit) => {
    if (Exit.isSuccess(exit)) {
      // Cache successful responses for longer
      return Duration.minutes(15)
    } else {
      // Cache failures for shorter time to allow retry
      return Duration.seconds(30)
    }
  }
})

// TTL based on data freshness requirements
const stockPriceCache = Cache.makeWith({
  capacity: 500,
  lookup: (symbol: string) => Effect.gen(function* () {
    const marketData = yield* MarketDataService
    const price = yield* marketData.getCurrentPrice(symbol)
    
    return {
      symbol,
      price: price.value,
      timestamp: Date.now(),
      volatility: price.volatility
    }
  }),
  timeToLive: (exit) => {
    if (Exit.isFailure(exit)) {
      return Duration.seconds(10) // Quick retry on failures
    }
    
    const result = exit.value
    
    // Highly volatile stocks get shorter TTL
    if (result.volatility > 0.05) {
      return Duration.seconds(30)
    }
    
    // Stable stocks can be cached longer
    return Duration.minutes(5)
  }
})
```

### Feature 3: Cache Refresh Strategies

The `refresh` method allows updating cached values without invalidating them, providing continuous service during updates.

#### Background Refresh Pattern

```typescript
import { Cache, Effect, Duration, Schedule, Fiber } from "effect"

const createAutoRefreshCache = <K, V, E>(
  name: string,
  options: {
    capacity: number
    timeToLive: Duration.DurationInput
    refreshInterval: Duration.DurationInput
    lookup: (key: K) => Effect.Effect<V, E>
  }
) => Effect.gen(function* () {
  const cache = yield* Cache.make({
    capacity: options.capacity,
    timeToLive: options.timeToLive,
    lookup: options.lookup
  })
  
  const activeKeys = yield* Ref.make<Set<K>>(new Set())
  
  // Track accessed keys for proactive refresh
  const get = (key: K) => Effect.gen(function* () {
    yield* Ref.update(activeKeys, keys => new Set([...keys, key]))
    return yield* cache.get(key)
  })
  
  // Background refresh job
  const refreshJob = Effect.gen(function* () {
    const keys = yield* Ref.get(activeKeys)
    const logger = yield* Logger
    
    yield* logger.info(`Refreshing ${keys.size} active cache entries for ${name}`)
    
    // Refresh all active keys concurrently
    yield* Effect.all(
      Array.from(keys).map(key => 
        cache.refresh(key).pipe(
          Effect.catchAll(error => {
            console.warn(`Failed to refresh ${name} cache for key:`, key, error)
            return Effect.unit
          })
        )
      ),
      { concurrency: 10 }
    )
  })
  
  // Start background refresh scheduler
  const scheduler = yield* refreshJob.pipe(
    Effect.repeat(Schedule.fixed(options.refreshInterval)),
    Effect.fork
  )
  
  return { get, cache, scheduler }
})

// Usage for critical data that needs high availability
const criticalDataCache = createAutoRefreshCache("user-sessions", {
  capacity: 10000,
  timeToLive: Duration.minutes(30),
  refreshInterval: Duration.minutes(5), // Refresh every 5 minutes
  lookup: (sessionId: string) => Effect.gen(function* () {
    const sessionStore = yield* SessionStore
    return yield* sessionStore.getSession(sessionId)
  })
})
```

## Practical Patterns & Best Practices

### Pattern 1: Cache Warming and Preloading

```typescript
import { Cache, Effect, Duration, Chunk, Array as Arr } from "effect"

// Cache warming utility
const warmCache = <K, V, E>(
  cache: Cache.Cache<K, V, E>,
  keys: ReadonlyArray<K>,
  options: {
    concurrency?: number
    batchSize?: number
  } = {}
) => Effect.gen(function* () {
  const { concurrency = 10, batchSize = 50 } = options
  const logger = yield* Logger
  
  yield* logger.info(`Warming cache with ${keys.length} keys`)
  
  // Process in batches to avoid overwhelming the system
  const batches = Chunk.toReadonlyArray(Chunk.chunksOf(Chunk.fromIterable(keys), batchSize))
  
  for (const batch of batches) {
    yield* Effect.all(
      batch.map(key => 
        cache.get(key).pipe(
          Effect.catchAll(error => {
            console.warn("Cache warming failed for key:", key, error)
            return Effect.unit
          })
        )
      ),
      { concurrency }
    )
    
    // Small delay between batches
    yield* Effect.sleep(Duration.milliseconds(100))
  }
  
  const stats = yield* cache.cacheStats
  yield* logger.info(`Cache warmed: ${stats.size} entries loaded`)
})

// Application startup cache warming
const applicationStartup = Effect.gen(function* () {
  const userCache = yield* createUserCache()
  const productCache = yield* createProductCache()
  
  // Load critical data during startup
  const criticalUserIds = ["admin-1", "system-user", "default-user"]
  const popularProductIds = yield* getPopularProductIds()
  
  yield* Effect.all([
    warmCache(userCache, criticalUserIds),
    warmCache(productCache, popularProductIds, { concurrency: 20 })
  ])
  
  console.log("Application startup completed with warmed caches")
})
```

### Pattern 2: Cache Invalidation Strategies

```typescript
import { Cache, Effect, Duration, Ref, Array as Arr } from "effect"

// Tag-based cache invalidation
const createTaggedCache = <K, V, E>(
  options: {
    capacity: number
    timeToLive: Duration.DurationInput
    lookup: (key: K) => Effect.Effect<V, E>
    getTags: (key: K, value: V) => ReadonlyArray<string>
  }
) => Effect.gen(function* () {
  const cache = yield* Cache.make({
    capacity: options.capacity,
    timeToLive: options.timeToLive,
    lookup: options.lookup
  })
  
  const tagMap = yield* Ref.make<Map<string, Set<K>>>(new Map())
  
  const get = (key: K) => Effect.gen(function* () {
    const value = yield* cache.get(key)
    const tags = options.getTags(key, value)
    
    // Update tag mappings
    yield* Ref.update(tagMap, map => {
      const newMap = new Map(map)
      tags.forEach(tag => {
        const keySet = newMap.get(tag) || new Set()
        keySet.add(key)
        newMap.set(tag, keySet)
      })
      return newMap
    })
    
    return value
  })
  
  const invalidateByTag = (tag: string) => Effect.gen(function* () {
    const map = yield* Ref.get(tagMap)
    const keysToInvalidate = map.get(tag)
    
    if (keysToInvalidate) {
      yield* Effect.all(
        Arr.fromIterable(keysToInvalidate).map(key => cache.invalidate(key))
      )
      
      // Clean up tag mappings
      yield* Ref.update(tagMap, m => {
        const newMap = new Map(m)
        newMap.delete(tag)
        return newMap
      })
    }
  })
  
  return { get, invalidateByTag, cache }
})

// Usage with user data that can be invalidated by organization
const userCache = createTaggedCache({
  capacity: 5000,
  timeToLive: Duration.minutes(15),
  lookup: (userId: string) => Effect.gen(function* () {
    const database = yield* Database
    return yield* database.users.findById(userId)
  }),
  getTags: (userId, user) => [`org:${user.organizationId}`, `role:${user.role}`]
})

// Invalidate all users in an organization
const handleOrganizationUpdate = (orgId: string) => Effect.gen(function* () {
  const cache = yield* userCache
  yield* cache.invalidateByTag(`org:${orgId}`)
  console.log(`Invalidated all cached users for organization: ${orgId}`)
})
```

### Pattern 3: Cache Metrics and Observability

```typescript
import { Cache, Effect, Duration, Ref, Schedule, Metric } from "effect"

// Comprehensive cache observability
const createObservableCache = <K, V, E>(
  name: string,
  options: {
    capacity: number
    timeToLive: Duration.DurationInput
    lookup: (key: K) => Effect.Effect<V, E>
  }
) => Effect.gen(function* () {
  // Create metrics
  const hitCounter = yield* Metric.counter(`cache_hits_total`)
  const missCounter = yield* Metric.counter(`cache_misses_total`)
  const lookupDuration = yield* Metric.histogram(`cache_lookup_duration_ms`)
  const cacheSize = yield* Metric.gauge(`cache_size`)
  
  const cache = yield* Cache.make({
    capacity: options.capacity,
    timeToLive: options.timeToLive,
    lookup: (key: K) => Effect.gen(function* () {
      const start = yield* Effect.clockWith(clock => clock.currentTimeMillis)
      const result = yield* options.lookup(key)
      const end = yield* Effect.clockWith(clock => clock.currentTimeMillis)
      
      yield* Metric.update(lookupDuration, end - start)
      yield* Metric.increment(missCounter)
      
      return result
    })
  })
  
  const get = (key: K) => Effect.gen(function* () {
    const exists = yield* cache.contains(key)
    
    if (exists) {
      yield* Metric.increment(hitCounter)
    }
    
    const result = yield* cache.get(key)
    
    // Update size metric
    const currentSize = yield* cache.size
    yield* Metric.set(cacheSize, currentSize)
    
    return result
  })
  
  // Periodic metrics reporting
  const reportMetrics = Effect.gen(function* () {
    const stats = yield* cache.cacheStats
    const logger = yield* Logger
    
    const hitRate = stats.hits / (stats.hits + stats.misses) || 0
    
    yield* logger.info(`Cache ${name} metrics:`, {
      size: stats.size,
      hits: stats.hits,
      misses: stats.misses,
      hitRate: (hitRate * 100).toFixed(2) + "%"
    })
  })
  
  const metricsScheduler = yield* reportMetrics.pipe(
    Effect.repeat(Schedule.fixed(Duration.minutes(5))),
    Effect.fork
  )
  
  return { get, cache, metricsScheduler }
})
```

## Integration Examples

### Integration with HTTP Client for API Caching

```typescript
import { Cache, Effect, Duration, HttpClient, Context } from "effect"

// HTTP client with automatic response caching
interface HttpCacheConfig {
  readonly defaultTTL: Duration.DurationInput
  readonly maxCacheSize: number
  readonly cacheableStatuses: ReadonlyArray<number>
}

const HttpCacheConfig = Context.GenericTag<HttpCacheConfig>("HttpCacheConfig")

const createHttpCache = Effect.gen(function* () {
  const config = yield* HttpCacheConfig
  
  return yield* Cache.make({
    capacity: config.maxCacheSize,
    timeToLive: config.defaultTTL,
    lookup: (request: HttpClient.request.HttpClientRequest) => Effect.gen(function* () {
      const client = yield* HttpClient.HttpClient
      const response = yield* HttpClient.execute(request)(client)
      
      // Only cache successful responses
      if (!config.cacheableStatuses.includes(response.status)) {
        return yield* Effect.fail(new NonCacheableResponseError(response.status))
      }
      
      const body = yield* HttpClient.response.text(response)
      
      return {
        status: response.status,
        headers: response.headers,
        body,
        cachedAt: Date.now()
      }
    })
  })
})

// HTTP client wrapper with caching
const cachedHttpGet = (url: string, headers: Record<string, string> = {}) =>
  Effect.gen(function* () {
    const cache = yield* createHttpCache
    const request = HttpClient.request.get(url).pipe(
      HttpClient.request.setHeaders(headers)
    )
    
    const cacheKey = `${request.method}:${request.url}:${JSON.stringify(headers)}`
    
    return yield* cache.get(request)
  })

// Layer providing cached HTTP client
const HttpCacheLayer = Layer.effect(
  HttpCacheConfig,
  Effect.succeed({
    defaultTTL: Duration.minutes(10),
    maxCacheSize: 1000,
    cacheableStatuses: [200, 301, 302, 404]
  })
)

// Usage in application
const fetchUserProfile = (userId: string) => Effect.gen(function* () {
  const response = yield* cachedHttpGet(
    `https://api.example.com/users/${userId}`,
    { "Authorization": "Bearer token" }
  )
  
  return JSON.parse(response.body)
}).pipe(
  Effect.provide(HttpCacheLayer)
)
```

### Integration with Database ORM

```typescript
import { Cache, Effect, Duration, Layer } from "effect"

// Database query caching layer
interface QueryCache {
  readonly query: <T>(
    sql: string,
    params: ReadonlyArray<unknown>
  ) => Effect.Effect<T, DatabaseError>
}

const QueryCache = Context.GenericTag<QueryCache>("QueryCache")

const makeQueryCache = Effect.gen(function* () {
  const database = yield* Database
  
  const cache = yield* Cache.make({
    capacity: 10000,
    timeToLive: Duration.minutes(5),
    lookup: (query: { sql: string; params: ReadonlyArray<unknown> }) => 
      Effect.gen(function* () {
        console.log("Executing query:", query.sql)
        return yield* database.execute(query.sql, query.params)
      })
  })
  
  const query = <T>(sql: string, params: ReadonlyArray<unknown> = []) =>
    Effect.gen(function* () {
      const queryKey = { sql, params }
      return yield* cache.get(queryKey) as Effect.Effect<T, DatabaseError>
    })
  
  return { query } satisfies QueryCache
})

// Enhanced repository with caching
class UserRepository {
  constructor(private queryCache: QueryCache) {}
  
  findById = (id: string) => 
    this.queryCache.query<User>(
      "SELECT * FROM users WHERE id = ?",
      [id]
    )
  
  findByEmail = (email: string) =>
    this.queryCache.query<User>(
      "SELECT * FROM users WHERE email = ?",
      [email]
    )
  
  findActiveUsers = () =>
    this.queryCache.query<ReadonlyArray<User>>(
      "SELECT * FROM users WHERE active = true ORDER BY created_at DESC",
      []
    )
}

// Layer composition
const QueryCacheLayer = Layer.effect(QueryCache, makeQueryCache)

const UserRepositoryLayer = Layer.effect(
  UserRepository,
  Effect.gen(function* () {
    const queryCache = yield* QueryCache
    return new UserRepository(queryCache)
  })
).pipe(Layer.provide(QueryCacheLayer))

// Usage in service
const getUserWithFallback = (id: string) => Effect.gen(function* () {
  const userRepo = yield* UserRepository
  
  const user = yield* userRepo.findById(id).pipe(
    Effect.catchTag("NotFoundError", () => 
      userRepo.findByEmail(`${id}@fallback.com`)
    )
  )
  
  return user
}).pipe(Effect.provide(UserRepositoryLayer))
```

### Testing Strategies

```typescript
import { Cache, Effect, Duration, TestContext, TestClock } from "effect"

// Test utilities for cache behavior
const createTestCache = <K, V>(
  lookup: (key: K) => V,
  options: {
    capacity?: number
    timeToLive?: Duration.DurationInput
  } = {}
) => Cache.make({
  capacity: options.capacity ?? 100,
  timeToLive: options.timeToLive ?? Duration.minutes(5),
  lookup: (key: K) => Effect.succeed(lookup(key))
})

// Test cache hit behavior
const testCacheHits = Effect.gen(function* () {
  let callCount = 0
  const cache = yield* createTestCache((key: string) => {
    callCount++
    return `value-${key}-${callCount}`
  })
  
  // First call should hit lookup
  const result1 = yield* cache.get("test")
  console.assert(result1 === "value-test-1", "First call should hit lookup")
  console.assert(callCount === 1, "Lookup should be called once")
  
  // Second call should use cache
  const result2 = yield* cache.get("test")
  console.assert(result2 === "value-test-1", "Second call should use cached value")
  console.assert(callCount === 1, "Lookup should not be called again")
  
  const stats = yield* cache.cacheStats
  console.assert(stats.hits === 1, "Should have one cache hit")
  console.assert(stats.misses === 1, "Should have one cache miss")
})

// Test TTL behavior with TestClock
const testCacheTTL = Effect.gen(function* () {
  const cache = yield* createTestCache(
    (key: string) => `value-${key}`,
    { timeToLive: Duration.seconds(10) }
  )
  
  // Cache a value
  const result1 = yield* cache.get("test")
  console.assert(result1 === "value-test")
  
  // Advance time by 5 seconds (within TTL)
  yield* TestClock.adjust(Duration.seconds(5))
  const result2 = yield* cache.get("test")
  console.assert(result2 === "value-test", "Value should still be cached")
  
  // Advance time beyond TTL
  yield* TestClock.adjust(Duration.seconds(10))
  const result3 = yield* cache.get("test")
  console.assert(result3 === "value-test", "Value should be recomputed")
  
  const stats = yield* cache.cacheStats
  console.assert(stats.misses === 2, "Should have two cache misses")
}).pipe(Effect.provide(TestContext.TestContext))

// Mock cache for testing
const createMockCache = <K, V, E = never>(
  mockData: Map<K, V>
): Cache.Cache<K, V, E> => ({
  [Cache.CacheTypeId]: {
    _Key: undefined as any,
    _Error: undefined as any,
    _Value: undefined as any
  },
  get: (key: K) => mockData.has(key) 
    ? Effect.succeed(mockData.get(key)!)
    : Effect.fail("Not found" as E),
  getEither: (key: K) => mockData.has(key)
    ? Effect.succeed(Either.left(mockData.get(key)!))
    : Effect.succeed(Either.right(mockData.get(key)!)),
  refresh: (key: K) => Effect.unit,
  set: (key: K, value: V) => Effect.sync(() => {
    mockData.set(key, value)
  }),
  // ... other ConsumerCache methods
} as any)

// Integration test with mock
const testUserService = Effect.gen(function* () {
  const mockUsers = new Map([
    ["user-1", { id: "user-1", name: "Alice" }],
    ["user-2", { id: "user-2", name: "Bob" }]
  ])
  
  const mockCache = createMockCache(mockUsers)
  
  const service = new UserService(mockCache)
  const user = yield* service.getUser("user-1")
  
  console.assert(user.name === "Alice", "Should return mocked user")
})
```

## Conclusion

Cache provides efficient, concurrent-safe caching with automatic deduplication, TTL management, and comprehensive error handling for Effect applications.

Key benefits:
- **Concurrency Safety** - Multiple concurrent requests for the same key are automatically deduplicated
- **Flexible TTL** - Support for static and dynamic time-to-live strategies based on success/failure
- **Memory Management** - LRU eviction with configurable capacity limits prevents memory leaks
- **Observability** - Built-in statistics and metrics for monitoring cache performance
- **Composability** - Integrates seamlessly with other Effect modules and patterns

Use Effect Cache when you need high-performance caching with proper concurrency handling, automatic resource management, and comprehensive error handling in your Effect applications.