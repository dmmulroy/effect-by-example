# MutableHashMap: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem MutableHashMap Solves

While Effect promotes immutability and functional programming, there are scenarios where mutable data structures provide critical performance benefits. Traditional approaches to high-performance caching, real-time state management, and performance-critical lookups often require mutable hash maps:

```typescript
// Traditional approach - JavaScript Map with manual thread safety
class PerformanceCache {
  private cache = new Map<string, { value: any; timestamp: number }>()
  private locks = new Map<string, Promise<void>>()
  
  async get(key: string): Promise<any> {
    // Manual locking for concurrent access
    const existingLock = this.locks.get(key)
    if (existingLock) {
      await existingLock
    }
    
    const entry = this.cache.get(key)
    if (!entry) return undefined
    
    // Manual expiration checking
    if (Date.now() - entry.timestamp > 60000) {
      this.cache.delete(key)
      return undefined
    }
    
    return entry.value
  }
  
  async set(key: string, value: any): Promise<void> {
    // Create lock to prevent concurrent modifications
    const lockPromise = this.waitForLock(key)
    this.locks.set(key, lockPromise)
    
    await lockPromise
    this.cache.set(key, { value, timestamp: Date.now() })
    this.locks.delete(key)
  }
  
  private async waitForLock(key: string): Promise<void> {
    const existingLock = this.locks.get(key)
    if (existingLock) {
      await existingLock
    }
  }
}

// Immutable HashMap approach - performance overhead for frequent updates
import { HashMap } from "effect"

const updateCache = (cache: HashMap.HashMap<string, CacheEntry>, key: string, value: any) => {
  // Every update creates new HashMap instance
  return cache.pipe(
    HashMap.set(key, { value, timestamp: Date.now() })
  )
}

// In hot paths with thousands of updates per second, this creates memory pressure
let cache = HashMap.empty<string, CacheEntry>()
for (let i = 0; i < 10000; i++) {
  cache = updateCache(cache, `key-${i}`, `value-${i}`)
  // 10,000 HashMap instances created
}
```

This approach leads to:
- **Performance bottlenecks** - Immutable structures create overhead in hot paths
- **Memory pressure** - Excessive allocations in high-frequency update scenarios  
- **Complex concurrency** - Manual locking and thread safety management
- **Limited integration** - Difficult to compose with Effect's ecosystem

### The MutableHashMap Solution

MutableHashMap provides controlled mutability within Effect's type system, offering high-performance mutable operations while maintaining Effect's composability and safety guarantees:

```typescript
import { MutableHashMap, Effect } from "effect"

// High-performance mutable hash map with Effect integration
const createPerformanceCache = Effect.gen(function* () {
  const cache = MutableHashMap.empty<string, { value: string; timestamp: number }>()
  
  const get = (key: string) => Effect.gen(function* () {
    const entry = MutableHashMap.get(cache, key)
    
    if (!entry._tag || entry._tag === "None") {
      return undefined
    }
    
    const now = yield* Effect.clockWith(clock => Effect.succeed(clock.unsafeCurrentTimeMillis()))
    
    // Check expiration
    if (now - entry.value.timestamp > 60000) {
      MutableHashMap.remove(cache, key)
      return undefined
    }
    
    return entry.value.value
  })
  
  const set = (key: string, value: string) => Effect.gen(function* () {
    const now = yield* Effect.clockWith(clock => Effect.succeed(clock.unsafeCurrentTimeMillis()))
    MutableHashMap.set(cache, key, { value, timestamp: now })
  })
  
  const stats = Effect.sync(() => ({
    size: MutableHashMap.size(cache),
    keys: MutableHashMap.keys(cache)
  }))
  
  return { get, set, stats } as const
})
```

### Key Concepts

**Controlled Mutability**: MutableHashMap provides mutable operations within Effect's controlled environment, ensuring predictable behavior while maintaining high performance.

**Thread-Safe Operations**: All MutableHashMap operations are designed to work safely within Effect's fiber-based concurrency model.

**Effect Integration**: Seamlessly composes with other Effect modules like Clock, Logger, and Metrics for comprehensive applications.

## Basic Usage Patterns

### Pattern 1: Creating and Initializing

```typescript
import { MutableHashMap, Effect } from "effect"

// Empty mutable hash map
const emptyCache = MutableHashMap.empty<string, number>()

// Create with initial entries
const initializedCache = MutableHashMap.make(
  ["key1", 100],
  ["key2", 200],
  ["key3", 300]
)

// Create from iterable
const fromArrayCache = MutableHashMap.fromIterable([
  ["user:1", { name: "Alice", age: 30 }],
  ["user:2", { name: "Bob", age: 25 }],
  ["user:3", { name: "Charlie", age: 35 }]
])
```

### Pattern 2: Basic Operations

```typescript
import { MutableHashMap, Effect, Option } from "effect"

const basicOperations = Effect.gen(function* () {
  const cache = MutableHashMap.empty<string, string>()
  
  // Set values
  MutableHashMap.set(cache, "greeting", "Hello")
  MutableHashMap.set(cache, "farewell", "Goodbye")
  
  // Get values (returns Option)
  const greeting = MutableHashMap.get(cache, "greeting")
  // greeting is Option.some("Hello")
  
  const missing = MutableHashMap.get(cache, "missing")
  // missing is Option.none()
  
  // Check existence
  const hasGreeting = MutableHashMap.has(cache, "greeting") // true
  const hasMissing = MutableHashMap.has(cache, "missing")   // false
  
  // Get size and check if empty
  const currentSize = MutableHashMap.size(cache) // 2
  const isEmpty = MutableHashMap.isEmpty(cache)  // false
  
  return { greeting, hasGreeting, currentSize, isEmpty }
})
```

### Pattern 3: Advanced Mutations

```typescript
import { MutableHashMap, Effect, Option } from "effect"

const advancedMutations = Effect.gen(function* () {
  const counters = MutableHashMap.empty<string, number>()
  
  // Initialize counters
  MutableHashMap.set(counters, "visits", 0)
  MutableHashMap.set(counters, "errors", 0)
  
  // Modify existing values
  MutableHashMap.modify(counters, "visits", count => count + 1)
  MutableHashMap.modify(counters, "visits", count => count + 5)
  
  // Conditional modifications with modifyAt
  MutableHashMap.modifyAt(counters, "new-counter", (existing) => {
    return Option.isSome(existing) 
      ? Option.some(existing.value + 1)
      : Option.some(1)
  })
  
  // Remove entries
  MutableHashMap.remove(counters, "errors")
  
  // Get all keys and values
  const allKeys = MutableHashMap.keys(counters)
  const allValues = MutableHashMap.values(counters)
  
  return { counters, allKeys, allValues }
})
```

## Real-World Examples

### Example 1: High-Performance Application Cache

A high-throughput web application needs efficient caching for database query results with automatic expiration:

```typescript
import { MutableHashMap, Effect, Clock, Logger, Schedule, Duration } from "effect"

interface CacheEntry<T> {
  readonly value: T
  readonly timestamp: number
  readonly hits: number
}

const createApplicationCache = <K, V>(
  options: {
    readonly maxSize: number
    readonly ttlMillis: number
    readonly cleanupIntervalMillis: number
  }
) => Effect.gen(function* () {
  const cache = MutableHashMap.empty<K, CacheEntry<V>>()
  const logger = yield* Logger.Logger
  
  // Cache operations
  const get = (key: K) => Effect.gen(function* () {
    const now = yield* Clock.currentTimeMillis
    const entry = MutableHashMap.get(cache, key)
    
    if (Option.isNone(entry)) {
      yield* Logger.debug(`Cache miss for key: ${String(key)}`)
      return Option.none<V>()
    }
    
    // Check expiration
    if (now - entry.value.timestamp > options.ttlMillis) {
      MutableHashMap.remove(cache, key)
      yield* Logger.debug(`Cache entry expired for key: ${String(key)}`)
      return Option.none<V>()
    }
    
    // Update hit counter
    MutableHashMap.modify(cache, key, current => ({
      ...current,
      hits: current.hits + 1
    }))
    
    yield* Logger.debug(`Cache hit for key: ${String(key)} (hits: ${entry.value.hits + 1})`)
    return Option.some(entry.value.value)
  })
  
  const set = (key: K, value: V) => Effect.gen(function* () {
    const now = yield* Clock.currentTimeMillis
    
    // Check size limit and evict if necessary
    if (MutableHashMap.size(cache) >= options.maxSize) {
      yield* evictLeastRecentlyUsed()
    }
    
    MutableHashMap.set(cache, key, {
      value,
      timestamp: now,
      hits: 0
    })
    
    yield* Logger.debug(`Cache set for key: ${String(key)}`)
  })
  
  const evictLeastRecentlyUsed = Effect.gen(function* () {
    const entries = Array.from(cache)
    if (entries.length === 0) return
    
    // Find entry with oldest timestamp and lowest hits
    let lruKey = entries[0][0]
    let lruEntry = entries[0][1]
    
    for (const [key, entry] of entries) {
      if (entry.timestamp < lruEntry.timestamp || 
          (entry.timestamp === lruEntry.timestamp && entry.hits < lruEntry.hits)) {
        lruKey = key
        lruEntry = entry
      }
    }
    
    MutableHashMap.remove(cache, lruKey)
    yield* Logger.debug(`Evicted LRU entry: ${String(lruKey)}`)
  })
  
  const cleanup = Effect.gen(function* () {
    const now = yield* Clock.currentTimeMillis
    const keysToRemove: K[] = []
    
    // Collect expired entries
    MutableHashMap.forEach(cache, (entry, key) => {
      if (now - entry.timestamp > options.ttlMillis) {
        keysToRemove.push(key)
      }
    })
    
    // Remove expired entries
    keysToRemove.forEach(key => MutableHashMap.remove(cache, key))
    
    if (keysToRemove.length > 0) {
      yield* Logger.info(`Cleaned up ${keysToRemove.length} expired cache entries`)
    }
  })
  
  const stats = Effect.sync(() => ({
    size: MutableHashMap.size(cache),
    isEmpty: MutableHashMap.isEmpty(cache),
    keys: MutableHashMap.keys(cache).slice(0, 10) // Sample of keys
  }))
  
  // Start cleanup task
  const cleanupTask = cleanup.pipe(
    Effect.repeat(Schedule.fixed(Duration.millis(options.cleanupIntervalMillis))),
    Effect.fork
  )
  
  yield* cleanupTask
  
  return { get, set, stats, cleanup } as const
})

// Usage example
const applicationCache = createApplicationCache<string, UserProfile>({
  maxSize: 10000,
  ttlMillis: 300000, // 5 minutes
  cleanupIntervalMillis: 60000 // 1 minute
}).pipe(
  Effect.provide(Logger.minimumLogLevel(Logger.LogLevel.Debug))
)
```

### Example 2: Real-Time Event Aggregation

A real-time analytics system that aggregates events by user and metric type with high-frequency updates:

```typescript
import { MutableHashMap, Effect, Clock, Ref, Schedule, Duration } from "effect"

interface EventMetrics {
  readonly count: number
  readonly sum: number
  readonly min: number
  readonly max: number
  readonly lastUpdated: number
}

const createEventAggregator = Effect.gen(function* () {
  const userMetrics = MutableHashMap.empty<string, MutableHashMap.MutableHashMap<string, EventMetrics>>()
  const totalEvents = yield* Ref.make(0)
  
  const recordEvent = (userId: string, metricType: string, value: number) => Effect.gen(function* () {
    const now = yield* Clock.currentTimeMillis
    
    // Get or create user metrics map
    const userMap = MutableHashMap.get(userMetrics, userId)
    let metricsMap: MutableHashMap.MutableHashMap<string, EventMetrics>
    
    if (Option.isNone(userMap)) {
      metricsMap = MutableHashMap.empty<string, EventMetrics>()
      MutableHashMap.set(userMetrics, userId, metricsMap)
    } else {
      metricsMap = userMap.value
    }
    
    // Update or create metric
    MutableHashMap.modifyAt(metricsMap, metricType, (existing) => {
      if (Option.isNone(existing)) {
        return Option.some({
          count: 1,
          sum: value,
          min: value,
          max: value,
          lastUpdated: now
        })
      }
      
      const current = existing.value
      return Option.some({
        count: current.count + 1,
        sum: current.sum + value,
        min: Math.min(current.min, value),
        max: Math.max(current.max, value),
        lastUpdated: now
      })
    })
    
    // Update total counter
    yield* Ref.update(totalEvents, count => count + 1)
  })
  
  const getUserMetrics = (userId: string) => Effect.gen(function* () {
    const userMap = MutableHashMap.get(userMetrics, userId)
    
    if (Option.isNone(userMap)) {
      return new Map<string, EventMetrics>()
    }
    
    const result = new Map<string, EventMetrics>()
    MutableHashMap.forEach(userMap.value, (metrics, metricType) => {
      result.set(metricType, metrics)
    })
    
    return result
  })
  
  const getTopUsers = (limit: number = 10) => Effect.gen(function* () {
    const userStats: Array<{ userId: string; totalEvents: number; metrics: number }> = []
    
    MutableHashMap.forEach(userMetrics, (metricsMap, userId) => {
      let totalEvents = 0
      let metricsCount = 0
      
      MutableHashMap.forEach(metricsMap, (metrics, _) => {
        totalEvents += metrics.count
        metricsCount += 1
      })
      
      userStats.push({ userId, totalEvents, metrics: metricsCount })
    })
    
    return userStats
      .sort((a, b) => b.totalEvents - a.totalEvents)
      .slice(0, limit)
  })
  
  const getGlobalStats = Effect.gen(function* () {
    const total = yield* Ref.get(totalEvents)
    const uniqueUsers = MutableHashMap.size(userMetrics)
    
    let totalMetricTypes = 0
    MutableHashMap.forEach(userMetrics, (metricsMap, _) => {
      totalMetricTypes += MutableHashMap.size(metricsMap)
    })
    
    return {
      totalEvents: total,
      uniqueUsers,
      totalMetricTypes,
      avgMetricsPerUser: uniqueUsers > 0 ? totalMetricTypes / uniqueUsers : 0
    }
  })
  
  const pruneOldMetrics = (maxAgeMillis: number) => Effect.gen(function* () {
    const now = yield* Clock.currentTimeMillis
    const cutoff = now - maxAgeMillis
    let prunedCount = 0
    
    const usersToRemove: string[] = []
    
    MutableHashMap.forEach(userMetrics, (metricsMap, userId) => {
      const metricsToRemove: string[] = []
      
      MutableHashMap.forEach(metricsMap, (metrics, metricType) => {
        if (metrics.lastUpdated < cutoff) {
          metricsToRemove.push(metricType)
          prunedCount++
        }
      })
      
      metricsToRemove.forEach(metricType => {
        MutableHashMap.remove(metricsMap, metricType)
      })
      
      if (MutableHashMap.isEmpty(metricsMap)) {
        usersToRemove.push(userId)
      }
    })
    
    usersToRemove.forEach(userId => {
      MutableHashMap.remove(userMetrics, userId)
    })
    
    return { prunedCount, usersRemoved: usersToRemove.length }
  })
  
  return {
    recordEvent,
    getUserMetrics,
    getTopUsers,
    getGlobalStats,
    pruneOldMetrics
  } as const
})

// Usage in high-frequency scenario
const processEventStream = Effect.gen(function* () {
  const aggregator = yield* createEventAggregator
  
  // Simulate high-frequency events
  const events = [
    { userId: "user1", metric: "page_views", value: 1 },
    { userId: "user1", metric: "clicks", value: 3 },
    { userId: "user2", metric: "page_views", value: 1 },
    { userId: "user1", metric: "page_views", value: 1 },
    { userId: "user2", metric: "purchases", value: 29.99 }
  ]
  
  // Process events concurrently
  yield* Effect.forEach(
    events,
    event => aggregator.recordEvent(event.userId, event.metric, event.value),
    { concurrency: "unbounded" }
  )
  
  const stats = yield* aggregator.getGlobalStats
  const topUsers = yield* aggregator.getTopUsers(5)
  
  return { stats, topUsers }
})
```

### Example 3: Session Management with Automatic Cleanup

A web application session manager that tracks user sessions with automatic expiration and cleanup:

```typescript
import { MutableHashMap, Effect, Clock, Logger, Schedule, Duration, Random } from "effect"

interface SessionData {
  readonly userId: string
  readonly createdAt: number
  readonly lastAccessed: number
  readonly data: Record<string, unknown>
  readonly ipAddress: string
}

const createSessionManager = Effect.gen(function* () {
  const sessions = MutableHashMap.empty<string, SessionData>()
  const logger = yield* Logger.Logger
  
  const generateSessionId = Effect.gen(function* () {
    const timestamp = yield* Clock.currentTimeMillis
    const random = yield* Random.nextInt
    return `session_${timestamp}_${Math.abs(random)}`
  })
  
  const createSession = (userId: string, ipAddress: string, initialData: Record<string, unknown> = {}) => 
    Effect.gen(function* () {
      const sessionId = yield* generateSessionId
      const now = yield* Clock.currentTimeMillis
      
      const sessionData: SessionData = {
        userId,
        createdAt: now,
        lastAccessed: now,
        data: initialData,
        ipAddress
      }
      
      MutableHashMap.set(sessions, sessionId, sessionData)
      
      yield* Logger.info(`Created session ${sessionId} for user ${userId}`)
      return sessionId
    })
  
  const getSession = (sessionId: string) => Effect.gen(function* () {
    const session = MutableHashMap.get(sessions, sessionId)
    
    if (Option.isNone(session)) {
      yield* Logger.debug(`Session not found: ${sessionId}`)
      return Option.none<SessionData>()
    }
    
    const now = yield* Clock.currentTimeMillis
    
    // Update last accessed time
    MutableHashMap.modify(sessions, sessionId, current => ({
      ...current,
      lastAccessed: now
    }))
    
    yield* Logger.debug(`Session accessed: ${sessionId}`)
    return Option.some(session.value)
  })
  
  const updateSessionData = (sessionId: string, updates: Record<string, unknown>) => 
    Effect.gen(function* () {
      const existing = MutableHashMap.get(sessions, sessionId)
      
      if (Option.isNone(existing)) {
        yield* Logger.warn(`Attempted to update non-existent session: ${sessionId}`)
        return false
      }
      
      const now = yield* Clock.currentTimeMillis
      
      MutableHashMap.modify(sessions, sessionId, current => ({
        ...current,
        lastAccessed: now,
        data: { ...current.data, ...updates }
      }))
      
      yield* Logger.debug(`Updated session data: ${sessionId}`)
      return true
    })
  
  const destroySession = (sessionId: string) => Effect.gen(function* () {
    const existed = MutableHashMap.has(sessions, sessionId)
    
    if (existed) {
      MutableHashMap.remove(sessions, sessionId)
      yield* Logger.info(`Destroyed session: ${sessionId}`)
    }
    
    return existed
  })
  
  const cleanupExpiredSessions = (maxIdleTimeMillis: number) => Effect.gen(function* () {
    const now = yield* Clock.currentTimeMillis
    const expiredSessions: string[] = []
    
    MutableHashMap.forEach(sessions, (session, sessionId) => {
      if (now - session.lastAccessed > maxIdleTimeMillis) {
        expiredSessions.push(sessionId)
      }
    })
    
    expiredSessions.forEach(sessionId => {
      MutableHashMap.remove(sessions, sessionId)
    })
    
    if (expiredSessions.length > 0) {
      yield* Logger.info(`Cleaned up ${expiredSessions.length} expired sessions`)
    }
    
    return expiredSessions.length
  })
  
  const getSessionStats = Effect.gen(function* () {
    const now = yield* Clock.currentTimeMillis
    const allSessions = Array.from(sessions)
    
    const userSessions = new Map<string, number>()
    let totalIdleTime = 0
    
    allSessions.forEach(([sessionId, session]) => {
      const userCount = userSessions.get(session.userId) || 0
      userSessions.set(session.userId, userCount + 1)
      totalIdleTime += now - session.lastAccessed
    })
    
    return {
      totalSessions: MutableHashMap.size(sessions),
      uniqueUsers: userSessions.size,
      avgIdleTime: allSessions.length > 0 ? totalIdleTime / allSessions.length : 0,
      userSessionCounts: Array.from(userSessions.entries())
        .sort(([, a], [, b]) => b - a)
        .slice(0, 10)
    }
  })
  
  const getUserSessions = (userId: string) => Effect.gen(function* () {
    const userSessions: Array<{ sessionId: string; session: SessionData }> = []
    
    MutableHashMap.forEach(sessions, (session, sessionId) => {
      if (session.userId === userId) {
        userSessions.push({ sessionId, session })
      }
    })
    
    return userSessions.sort((a, b) => b.session.lastAccessed - a.session.lastAccessed)
  })
  
  // Start automatic cleanup
  const startCleanupTask = (intervalMillis: number, maxIdleTimeMillis: number) => {
    return cleanupExpiredSessions(maxIdleTimeMillis).pipe(
      Effect.repeat(Schedule.fixed(Duration.millis(intervalMillis))),
      Effect.fork
    )
  }
  
  return {
    createSession,
    getSession,
    updateSessionData,
    destroySession,
    cleanupExpiredSessions,
    getSessionStats,
    getUserSessions,
    startCleanupTask
  } as const
})

// Usage example with automatic cleanup
const sessionManagerExample = Effect.gen(function* () {
  const sessionManager = yield* createSessionManager
  
  // Start cleanup task (every 5 minutes, expire after 30 minutes)
  yield* sessionManager.startCleanupTask(300000, 1800000)
  
  // Create some sessions
  const sessionId1 = yield* sessionManager.createSession("user1", "192.168.1.100", { theme: "dark" })
  const sessionId2 = yield* sessionManager.createSession("user2", "192.168.1.101", { language: "en" })
  
  // Update session data
  yield* sessionManager.updateSessionData(sessionId1, { lastPage: "/dashboard" })
  
  // Get session statistics
  const stats = yield* sessionManager.getSessionStats
  
  return { sessionId1, sessionId2, stats }
}).pipe(
  Effect.provide(Logger.pretty)
)
```

## Advanced Features Deep Dive

### Feature 1: Hash-based Key Distribution

MutableHashMap uses hash-based key distribution to optimize performance for different key types:

#### Basic Hash Distribution

```typescript
import { MutableHashMap, Equal, Hash } from "effect"

// For primitive keys (strings, numbers), MutableHashMap uses referential storage
const primitiveCache = MutableHashMap.empty<string, number>()
MutableHashMap.set(primitiveCache, "key1", 100)
MutableHashMap.set(primitiveCache, "key2", 200)

// For complex keys, it uses hash buckets
class UserId implements Equal.Equal {
  constructor(readonly value: string) {}
  
  [Hash.symbol]() {
    return Hash.hash(this.value)
  }
  
  [Equal.symbol](that: unknown): boolean {
    return that instanceof UserId && this.value === that.value
  }
}

const userCache = MutableHashMap.empty<UserId, UserProfile>()
MutableHashMap.set(userCache, new UserId("user1"), { name: "Alice", email: "alice@example.com" })
```

#### Real-World Hash Distribution Example

```typescript
import { MutableHashMap, Equal, Hash, Effect } from "effect"

// Custom key type for database records
class DatabaseKey implements Equal.Equal {
  constructor(
    readonly table: string,
    readonly id: string,
    readonly version: number = 1
  ) {}
  
  [Hash.symbol]() {
    // Custom hash combining all fields
    return Hash.combine(
      Hash.hash(this.table),
      Hash.combine(
        Hash.hash(this.id),
        Hash.hash(this.version)
      )
    )
  }
  
  [Equal.symbol](that: unknown): boolean {
    return that instanceof DatabaseKey &&
           this.table === that.table &&
           this.id === that.id &&
           this.version === that.version
  }
  
  toString(): string {
    return `${this.table}:${this.id}:v${this.version}`
  }
}

const createDatabaseCache = Effect.gen(function* () {
  const cache = MutableHashMap.empty<DatabaseKey, unknown>()
  
  const set = <T>(table: string, id: string, data: T, version = 1) => {
    const key = new DatabaseKey(table, id, version)
    MutableHashMap.set(cache, key, data)
    return key
  }
  
  const get = <T>(key: DatabaseKey): T | undefined => {
    const result = MutableHashMap.get(cache, key)
    return Option.isSome(result) ? result.value as T : undefined
  }
  
  const getLatestVersion = <T>(table: string, id: string): T | undefined => {
    // Find the highest version for this table/id combination
    let latestKey: DatabaseKey | undefined
    let latestData: T | undefined
    
    MutableHashMap.forEach(cache, (data, key) => {
      if (key instanceof DatabaseKey && 
          key.table === table && 
          key.id === id) {
        if (!latestKey || key.version > latestKey.version) {
          latestKey = key
          latestData = data as T
        }
      }
    })
    
    return latestData
  }
  
  const invalidateTable = (table: string) => {
    const keysToRemove: DatabaseKey[] = []
    
    MutableHashMap.forEach(cache, (_, key) => {
      if (key instanceof DatabaseKey && key.table === table) {
        keysToRemove.push(key)
      }
    })
    
    keysToRemove.forEach(key => MutableHashMap.remove(cache, key))
    return keysToRemove.length
  }
  
  return { set, get, getLatestVersion, invalidateTable } as const
})
```

### Feature 2: Advanced Mutation Patterns

#### Atomic Updates with modifyAt

```typescript
import { MutableHashMap, Option, Effect } from "effect"

const createAtomicCounter = Effect.gen(function* () {
  const counters = MutableHashMap.empty<string, number>()
  
  // Atomic increment - creates counter if it doesn't exist
  const increment = (key: string, delta = 1) => {
    MutableHashMap.modifyAt(counters, key, (current) => {
      const newValue = Option.isSome(current) ? current.value + delta : delta
      return Option.some(newValue)
    })
    
    return MutableHashMap.get(counters, key)
  }
  
  // Atomic decrement with minimum value
  const decrementWithFloor = (key: string, delta = 1, minimum = 0) => {
    let result: Option.Option<number> = Option.none()
    
    MutableHashMap.modifyAt(counters, key, (current) => {
      if (Option.isNone(current)) {
        return Option.none() // Don't create if doesn't exist
      }
      
      const newValue = Math.max(current.value - delta, minimum)
      result = Option.some(newValue)
      return result
    })
    
    return result
  }
  
  // Conditional set - only if current value meets condition
  const setIf = (key: string, newValue: number, condition: (current?: number) => boolean) => {
    let updated = false
    
    MutableHashMap.modifyAt(counters, key, (current) => {
      const currentValue = Option.isSome(current) ? current.value : undefined
      
      if (condition(currentValue)) {
        updated = true
        return Option.some(newValue)
      }
      
      return current
    })
    
    return updated
  }
  
  return { increment, decrementWithFloor, setIf, counters } as const
})

// Usage examples
const atomicCounterExample = Effect.gen(function* () {
  const counter = yield* createAtomicCounter
  
  // Increment creates counter
  const count1 = counter.increment("api-calls", 5)
  console.log(count1) // Option.some(5)
  
  // Further increments
  counter.increment("api-calls", 3)
  const count2 = counter.increment("api-calls", 2)
  console.log(count2) // Option.some(10)
  
  // Conditional updates
  const wasUpdated = counter.setIf("api-calls", 0, (current) => 
    current !== undefined && current > 5
  )
  console.log(wasUpdated) // true
  
  return MutableHashMap.get(counter.counters, "api-calls")
})
```

#### Advanced Bulk Operations

```typescript
import { MutableHashMap, Effect, Array as Arr } from "effect"

const createBulkOperations = <K, V>() => {
  
  const bulkSet = (map: MutableHashMap.MutableHashMap<K, V>, entries: Array<[K, V]>) => {
    entries.forEach(([key, value]) => {
      MutableHashMap.set(map, key, value)
    })
  }
  
  const bulkModify = (
    map: MutableHashMap.MutableHashMap<K, V>, 
    keys: Array<K>, 
    modifier: (value: V) => V
  ) => {
    const results: Array<{ key: K; updated: boolean }> = []
    
    keys.forEach(key => {
      const existing = MutableHashMap.get(map, key)
      if (Option.isSome(existing)) {
        MutableHashMap.set(map, key, modifier(existing.value))
        results.push({ key, updated: true })
      } else {
        results.push({ key, updated: false })
      }
    })
    
    return results
  }
  
  const bulkRemove = (map: MutableHashMap.MutableHashMap<K, V>, keys: Array<K>) => {
    const removed: Array<K> = []
    
    keys.forEach(key => {
      if (MutableHashMap.has(map, key)) {
        MutableHashMap.remove(map, key)
        removed.push(key)
      }
    })
    
    return removed
  }
  
  const filterAndModify = (
    map: MutableHashMap.MutableHashMap<K, V>,
    predicate: (value: V, key: K) => boolean,
    modifier: (value: V, key: K) => V
  ) => {
    const keysToModify: Array<K> = []
    
    MutableHashMap.forEach(map, (value, key) => {
      if (predicate(value, key)) {
        keysToModify.push(key)
      }
    })
    
    keysToModify.forEach(key => {
      MutableHashMap.modify(map, key, (value) => modifier(value, key))
    })
    
    return keysToModify.length
  }
  
  return { bulkSet, bulkModify, bulkRemove, filterAndModify } as const
}

// Usage example
const bulkOperationsExample = Effect.gen(function* () {
  const operations = createBulkOperations<string, number>()
  const numbers = MutableHashMap.empty<string, number>()
  
  // Bulk set initial values
  operations.bulkSet(numbers, [
    ["one", 1],
    ["two", 2], 
    ["three", 3],
    ["four", 4],
    ["five", 5]
  ])
  
  // Bulk modify even numbers
  const modifyResults = operations.bulkModify(
    numbers,
    ["two", "four", "six"], // six doesn't exist
    value => value * 10
  )
  
  // Filter and modify values > 10
  const modifiedCount = operations.filterAndModify(
    numbers,
    (value) => value > 10,
    (value) => value + 100
  )
  
  const finalState = Array.from(numbers)
  
  return { modifyResults, modifiedCount, finalState }
})
```

## Practical Patterns & Best Practices

### Pattern 1: Safe Mutation Within Effect Context

```typescript
import { MutableHashMap, Effect, Logger, Ref } from "effect"

// Helper to create isolated mutable operations
const createIsolatedMutation = <K, V, R, E, A>(
  operation: (map: MutableHashMap.MutableHashMap<K, V>) => Effect.Effect<A, E, R>
) => Effect.gen(function* () {
  const map = MutableHashMap.empty<K, V>()
  const result = yield* operation(map)
  return { result, finalState: Array.from(map) }
})

// Transaction-like pattern for multiple related mutations
const createTransactionalMutation = <K, V>() => {
  const withTransaction = <R, E, A>(
    map: MutableHashMap.MutableHashMap<K, V>,
    operations: Effect.Effect<A, E, R>
  ) => Effect.gen(function* () {
    // Take snapshot of current state
    const snapshot = new Map(Array.from(map))
    
    try {
      const result = yield* operations
      return result
    } catch (error) {
      // Rollback on error
      MutableHashMap.clear(map)
      snapshot.forEach((value, key) => {
        MutableHashMap.set(map, key, value)
      })
      throw error
    }
  })
  
  return { withTransaction }
}

// Usage pattern
const safeTransactionalUpdate = Effect.gen(function* () {
  const logger = yield* Logger.Logger
  const userCache = MutableHashMap.empty<string, UserProfile>()
  const { withTransaction } = createTransactionalMutation<string, UserProfile>()
  
  const updateMultipleUsers = withTransaction(userCache, Effect.gen(function* () {
    // These operations will be rolled back if any fail
    MutableHashMap.set(userCache, "user1", { name: "Alice", email: "alice@example.com" })
    MutableHashMap.set(userCache, "user2", { name: "Bob", email: "bob@example.com" })
    
    // Simulate potential failure
    const shouldFail = yield* Effect.succeed(false)
    if (shouldFail) {
      yield* Effect.fail(new Error("Operation failed"))
    }
    
    yield* Logger.info("Successfully updated multiple users")
    return MutableHashMap.size(userCache)
  }))
  
  const result = yield* updateMultipleUsers
  return result
}).pipe(
  Effect.provide(Logger.pretty)
)
```

### Pattern 2: Performance Monitoring and Metrics

```typescript
import { MutableHashMap, Effect, Metrics, Clock, Ref } from "effect"

const createMonitoredCache = <K, V>(name: string) => Effect.gen(function* () {
  const cache = MutableHashMap.empty<K, V>()
  
  // Metrics
  const hitCounter = Metrics.counter(`${name}_cache_hits`)
  const missCounter = Metrics.counter(`${name}_cache_misses`)
  const sizeGauge = Metrics.gauge(`${name}_cache_size`)
  const operationTimer = Metrics.timer(`${name}_cache_operation_duration`)
  
  const get = (key: K) => Effect.gen(function* () {
    const start = yield* Clock.currentTimeMillis
    const result = MutableHashMap.get(cache, key)
    const end = yield* Clock.currentTimeMillis
    
    yield* Metrics.timer.update(operationTimer, end - start)
    
    if (Option.isSome(result)) {
      yield* Metrics.counter.increment(hitCounter)
      return result
    } else {  
      yield* Metrics.counter.increment(missCounter)
      return Option.none<V>()
    }
  })
  
  const set = (key: K, value: V) => Effect.gen(function* () {
    const start = yield* Clock.currentTimeMillis
    MutableHashMap.set(cache, key, value)
    const end = yield* Clock.currentTimeMillis
    
    yield* Metrics.timer.update(operationTimer, end - start)
    yield* Metrics.gauge.set(sizeGauge, MutableHashMap.size(cache))
  })
  
  const stats = Effect.gen(function* () {
    return {
      size: MutableHashMap.size(cache),
      isEmpty: MutableHashMap.isEmpty(cache)
    }
  })
  
  return { get, set, stats } as const
})

// Monitoring usage
const monitoredCacheExample = Effect.gen(function* () {
  const cache = yield* createMonitoredCache<string, string>("user_sessions")
  
  // Operations are automatically monitored
  yield* cache.set("session1", "user_data_1")
  yield* cache.set("session2", "user_data_2")
  
  const hit = yield* cache.get("session1")
  const miss = yield* cache.get("session3")
  
  const stats = yield* cache.stats
  
  return { hit, miss, stats }
})
```

### Pattern 3: Thread-Safe Concurrent Access

```typescript
import { MutableHashMap, Effect, Semaphore, Duration, Schedule } from "effect"

const createConcurrentSafeCache = <K, V>(maxConcurrentOperations = 10) => Effect.gen(function* () {
  const cache = MutableHashMap.empty<K, V>()
  const semaphore = yield* Semaphore.make(maxConcurrentOperations)
  
  const safeGet = (key: K) => 
    Semaphore.withPermit(semaphore, Effect.sync(() => MutableHashMap.get(cache, key)))
  
  const safeSet = (key: K, value: V) =>
    Semaphore.withPermit(semaphore, Effect.sync(() => MutableHashMap.set(cache, key, value)))
  
  const safeModify = (key: K, f: (value: V) => V) =>
    Semaphore.withPermit(semaphore, Effect.sync(() => MutableHashMap.modify(cache, key, f)))
  
  const safeBulkOperation = <A>(operation: () => A) =>
    Semaphore.withPermit(semaphore, Effect.sync(operation))
  
  const getWithRetry = (key: K, maxRetries = 3) =>
    safeGet(key).pipe(
      Effect.retry(Schedule.exponential(Duration.millis(100)).pipe(
        Schedule.upTo(Duration.seconds(1)),
        Schedule.compose(Schedule.recurs(maxRetries))
      ))
    )
  
  return { 
    safeGet, 
    safeSet, 
    safeModify, 
    safeBulkOperation, 
    getWithRetry 
  } as const
})

// Concurrent usage example
const concurrentAccessExample = Effect.gen(function* () {
  const cache = yield* createConcurrentSafeCache<string, number>()
  
  // Simulate concurrent operations
  const concurrentOperations = Array.from({ length: 100 }, (_, i) => 
    cache.safeSet(`key${i}`, i * 2)
  )
  
  yield* Effect.forEach(concurrentOperations, op => op, { concurrency: "unbounded" })
  
  // Concurrent reads
  const concurrentReads = Array.from({ length: 50 }, (_, i) =>
    cache.safeGet(`key${i}`)
  )
  
  const results = yield* Effect.forEach(concurrentReads, read => read, { concurrency: 10 })
  
  return results.filter(Option.isSome).length
})
```

## Integration Examples

### Integration with Effect Cache

```typescript
import { Cache, MutableHashMap, Effect, Duration, Logger } from "effect"

// Custom cache implementation using MutableHashMap for storage
const createCustomCacheStorage = <K, V>() => {
  const storage = MutableHashMap.empty<K, { value: V; timestamp: number }>()
  
  const makeCache = <R, E>(
    lookup: (key: K) => Effect.Effect<V, E, R>,
    options: {
      readonly ttl: Duration.Duration
      readonly capacity: number
    }
  ) => {
    return Cache.makeWith({
      capacity: options.capacity,
      timeToLive: options.ttl,
      lookup
    })
  }
  
  // Manual cache implementation using MutableHashMap
  const manualCache = {
    get: (key: K) => Effect.gen(function* () {
      const entry = MutableHashMap.get(storage, key)
      
      if (Option.isNone(entry)) {
        return Option.none<V>()
      }
      
      const now = yield* Effect.clockWith(clock => Effect.succeed(clock.unsafeCurrentTimeMillis()))
      const ttlMs = Duration.toMillis(options.ttl)
      
      if (now - entry.value.timestamp > ttlMs) {
        MutableHashMap.remove(storage, key)
        return Option.none<V>()
      }
      
      return Option.some(entry.value.value)
    }),
    
    set: (key: K, value: V) => Effect.gen(function* () {
      const now = yield* Effect.clockWith(clock => Effect.succeed(clock.unsafeCurrentTimeMillis()))
      MutableHashMap.set(storage, key, { value, timestamp: now })
    }),
    
    invalidate: (key: K) => Effect.sync(() => {
      MutableHashMap.remove(storage, key)
    })
  }
  
  return { makeCache, manualCache }
}

// Usage with both Effect Cache and manual implementation
const cacheIntegrationExample = Effect.gen(function* () {
  const logger = yield* Logger.Logger
  const { makeCache, manualCache } = createCustomCacheStorage<string, UserData>()
  
  // Using Effect's built-in Cache
  const officialCache = yield* makeCache(
    (userId: string) => Effect.gen(function* () {
      yield* Logger.info(`Loading user ${userId} from database`)
      // Simulate database call
      yield* Effect.sleep(Duration.millis(100))
      return { id: userId, name: `User ${userId}`, email: `${userId}@example.com` }
    }),
    {
      ttl: Duration.minutes(5),
      capacity: 1000
    }
  )
  
  // Using manual MutableHashMap cache
  const fetchWithManualCache = (userId: string) => Effect.gen(function* () {
    const cached = yield* manualCache.get(userId)
    
    if (Option.isSome(cached)) {
      yield* Logger.debug(`Cache hit for user ${userId}`)
      return cached.value
    }
    
    yield* Logger.info(`Cache miss, loading user ${userId}`)
    const userData = { id: userId, name: `User ${userId}`, email: `${userId}@example.com` }
    yield* manualCache.set(userId, userData)
    
    return userData
  })
  
  // Test both approaches
  const user1Official = yield* Cache.get(officialCache, "user1")
  const user1Manual = yield* fetchWithManualCache("user1")
  
  return { user1Official, user1Manual }
})
```

### Integration with Schema Validation

```typescript
import { Schema, MutableHashMap, Effect, Either } from "effect"

const UserSchema = Schema.Struct({
  id: Schema.String,
  email: Schema.String.pipe(Schema.pattern(/^[^\s@]+@[^\s@]+\.[^\s@]+$/)),
  age: Schema.Number.pipe(Schema.between(0, 150)),
  role: Schema.Union(Schema.Literal("admin"), Schema.Literal("user"), Schema.Literal("guest"))
})

type User = Schema.Schema.Type<typeof UserSchema>

const createValidatedUserCache = Effect.gen(function* () {
  const cache = MutableHashMap.empty<string, User>()
  const errors = MutableHashMap.empty<string, string>()
  
  const setUser = (id: string, userData: unknown) => Effect.gen(function* () {
    const parseResult = Schema.decodeUnknownEither(UserSchema)(userData)
    
    if (Either.isLeft(parseResult)) {
      const errorMessage = `Validation failed: ${parseResult.left}`
      MutableHashMap.set(errors, id, errorMessage)
      return Either.left(errorMessage)
    }
    
    const validUser = parseResult.right
    MutableHashMap.set(cache, id, validUser)
    
    // Clear any previous errors
    if (MutableHashMap.has(errors, id)) {
      MutableHashMap.remove(errors, id)
    }
    
    return Either.right(validUser)
  })
  
  const getUser = (id: string) => Effect.sync(() => {
    return MutableHashMap.get(cache, id)
  })
  
  const getUserError = (id: string) => Effect.sync(() => {
    return MutableHashMap.get(errors, id)
  })
  
  const updateUser = (id: string, updates: Partial<unknown>) => Effect.gen(function* () {
    const existing = MutableHashMap.get(cache, id)
    
    if (Option.isNone(existing)) {
      return Either.left("User not found")
    }
    
    const mergedData = { ...existing.value, ...updates }
    return yield* setUser(id, mergedData)
  })
  
  const getValidationStats = Effect.sync(() => ({
    validUsers: MutableHashMap.size(cache),
    erroredUsers: MutableHashMap.size(errors),
    totalAttempts: MutableHashMap.size(cache) + MutableHashMap.size(errors)
  }))
  
  return { 
    setUser, 
    getUser, 
    getUserError, 
    updateUser, 
    getValidationStats 
  } as const
})

// Usage example
const validatedCacheExample = Effect.gen(function* () {
  const userCache = yield* createValidatedUserCache
  
  // Valid user data
  const validResult = yield* userCache.setUser("user1", {
    id: "user1",
    email: "alice@example.com",
    age: 30,
    role: "admin"
  })
  
  // Invalid user data
  const invalidResult = yield* userCache.setUser("user2", {
    id: "user2",
    email: "invalid-email",
    age: -5,
    role: "invalid-role"
  })
  
  // Get results
  const user1 = yield* userCache.getUser("user1")
  const user2Error = yield* userCache.getUserError("user2")
  const stats = yield* userCache.getValidationStats
  
  return { validResult, invalidResult, user1, user2Error, stats }
})
```

### Testing Strategies

```typescript
import { MutableHashMap, Effect, TestContext, TestClock, Duration } from "effect"

// Helper for testing MutableHashMap operations
const createTestHashMap = <K, V>() => {
  const createMockData = (count: number): Array<[K, V]> => {
    return Array.from({ length: count }, (_, i) => [
      `key${i}` as K,
      `value${i}` as V
    ])
  }
  
  const assertSizeEquals = (map: MutableHashMap.MutableHashMap<K, V>, expected: number) => 
    Effect.sync(() => {
      const actual = MutableHashMap.size(map)
      if (actual !== expected) {
        throw new Error(`Expected size ${expected}, got ${actual}`)
      }
    })
  
  const assertHasKey = (map: MutableHashMap.MutableHashMap<K, V>, key: K) =>
    Effect.sync(() => {
      if (!MutableHashMap.has(map, key)) {
        throw new Error(`Expected map to have key ${String(key)}`)
      }
    })
  
  const assertKeyValue = (map: MutableHashMap.MutableHashMap<K, V>, key: K, expectedValue: V) =>
    Effect.sync(() => {
      const actual = MutableHashMap.get(map, key)
      if (Option.isNone(actual)) {
        throw new Error(`Key ${String(key)} not found`)
      }
      if (actual.value !== expectedValue) {
        throw new Error(`Expected value ${String(expectedValue)}, got ${String(actual.value)}`)
      }
    })
  
  return { createMockData, assertSizeEquals, assertHasKey, assertKeyValue }
}

// Test suite example
const testMutableHashMapOperations = Effect.gen(function* () {
  const { createMockData, assertSizeEquals, assertHasKey, assertKeyValue } = 
    createTestHashMap<string, string>()
  
  // Test basic operations
  const map = MutableHashMap.empty<string, string>()
  
  // Test set and get
  MutableHashMap.set(map, "test", "value") 
  yield* assertHasKey(map, "test")
  yield* assertKeyValue(map, "test", "value")
  
  // Test bulk operations
  const mockData = createMockData(5)
  mockData.forEach(([key, value]) => {
    MutableHashMap.set(map, key, value)
  })
  
  yield* assertSizeEquals(map, 6) // 5 + 1 existing
  
  // Test modify
  MutableHashMap.modify(map, "key0", (value) => value.toUpperCase())
  yield* assertKeyValue(map, "key0", "VALUE0")
  
  // Test remove
  MutableHashMap.remove(map, "key1")
  yield* assertSizeEquals(map, 5)
  
  return "All tests passed"
})

// Property-based testing helper
const createPropertyTest = <K, V>() => {
  const testInvariant = (
    map: MutableHashMap.MutableHashMap<K, V>,
    operations: Array<{ type: "set" | "remove", key: K, value?: V }>
  ) => Effect.gen(function* () {
    let expectedSize = MutableHashMap.size(map)
    const expectedKeys = new Set(MutableHashMap.keys(map))
    
    for (const op of operations) {
      if (op.type === "set" && op.value !== undefined) {
        const hadKey = expectedKeys.has(op.key)
        MutableHashMap.set(map, op.key, op.value)
        expectedKeys.add(op.key)
        if (!hadKey) expectedSize++
      } else if (op.type === "remove") {
        const hadKey = expectedKeys.has(op.key)
        MutableHashMap.remove(map, op.key)
        expectedKeys.delete(op.key)
        if (hadKey) expectedSize--
      }
    }
    
    // Verify invariants
    yield* Effect.sync(() => {
      const actualSize = MutableHashMap.size(map)
      const actualKeys = new Set(MutableHashMap.keys(map))
      
      if (actualSize !== expectedSize) {
        throw new Error(`Size invariant failed: expected ${expectedSize}, got ${actualSize}`)
      }
      
      if (actualKeys.size !== expectedKeys.size) {
        throw new Error(`Keys count mismatch: expected ${expectedKeys.size}, got ${actualKeys.size}`)
      }
      
      for (const key of expectedKeys) {
        if (!actualKeys.has(key)) {
          throw new Error(`Missing expected key: ${String(key)}`)
        }
      }
    })
  })
  
  return { testInvariant }
}

// Time-based testing with TestClock
const testTimeBasedCache = Effect.gen(function* () {
  const cache = MutableHashMap.empty<string, { value: string; timestamp: number }>()
  
  const setWithTimestamp = (key: string, value: string) => Effect.gen(function* () {
    const now = yield* TestClock.currentTimeMillis
    MutableHashMap.set(cache, key, { value, timestamp: now })
  })
  
  const isExpired = (key: string, ttlMillis: number) => Effect.gen(function* () {
    const entry = MutableHashMap.get(cache, key)
    if (Option.isNone(entry)) return true
    
    const now = yield* TestClock.currentTimeMillis
    return now - entry.value.timestamp > ttlMillis
  })
  
  // Set initial value
  yield* setWithTimestamp("test", "value")
  
  // Advance time and check expiration
  yield* TestClock.adjust(Duration.seconds(30))
  const expired30s = yield* isExpired("test", 60000) // 1 minute TTL
  
  yield* TestClock.adjust(Duration.seconds(45))
  const expired75s = yield* isExpired("test", 60000)
  
  return { expired30s, expired75s } // should be false, true
}).pipe(
  Effect.provide(TestContext.TestContext)
)
```

## Conclusion

MutableHashMap provides controlled, high-performance mutability within Effect's functional programming paradigm, enabling efficient solutions for caching, real-time state management, and performance-critical applications.

Key benefits:
- **Performance optimization**: Eliminates immutability overhead in hot paths while maintaining Effect's safety guarantees
- **Seamless integration**: Works naturally with Effect's concurrency model, error handling, and resource management
- **Flexible operations**: Supports atomic updates, bulk operations, and custom hash-based key distribution
- **Memory efficiency**: Provides mutable operations without the memory pressure of creating new instances

MutableHashMap is ideal when you need the performance characteristics of mutable data structures while maintaining the composability and safety of Effect's ecosystem. Use it for high-frequency update scenarios, caching layers, session management, and real-time data aggregation where immutability would create performance bottlenecks.