# TMap: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem TMap Solves

Traditional shared maps in concurrent programming suffer from race conditions and inconsistent state:

```typescript
// Traditional approach - race conditions and data corruption
const sharedMap = new Map<string, number>()

// Multiple concurrent operations
Promise.all([
  Promise.resolve().then(() => sharedMap.set("counter", (sharedMap.get("counter") || 0) + 1)),
  Promise.resolve().then(() => sharedMap.set("counter", (sharedMap.get("counter") || 0) + 1)),
  Promise.resolve().then(() => sharedMap.delete("other-key"))
])

// Result is unpredictable - increments may be lost due to race conditions
```

This approach leads to:
- **Lost Updates** - Concurrent read-modify-write operations interfering
- **Inconsistent Reads** - Reading partial state during modifications
- **Complex Locking** - Manual synchronization adding complexity and deadlock risk
- **No Atomicity** - No way to perform multiple map operations atomically

### The TMap Solution

TMap provides a transactional map that guarantees consistency across concurrent operations:

```typescript
import { STM, TMap } from "effect"

// Safe concurrent operations with TMap
const program = STM.gen(function* () {
  const tmap = yield* TMap.empty<string, number>()
  
  // All operations are atomic and consistent
  yield* TMap.set(tmap, "counter", 0)
  yield* TMap.updateWith(tmap, "counter", (value) => value ? value + 1 : 1)
  yield* TMap.updateWith(tmap, "counter", (value) => value ? value + 1 : 1)
  
  return yield* TMap.get(tmap, "counter")
})
```

### Key Concepts

**Transactional Map**: A mutable key-value store that can only be modified within STM transactions, ensuring atomicity and consistency.

**Atomic Operations**: All TMap operations are atomic, meaning either all operations in a transaction succeed or none do.

**Composability**: TMap operations compose naturally with other STM data structures for complex atomic operations.

## Basic Usage Patterns

### Pattern 1: Creating and Initializing TMaps

```typescript
import { STM, TMap } from "effect"

// Empty map
const createEmpty = STM.gen(function* () {
  return yield* TMap.empty<string, number>()
})

// From entries
const createFromEntries = STM.gen(function* () {
  return yield* TMap.make(
    ["apple", 5],
    ["banana", 3],
    ["orange", 8]
  )
})

// From iterable
const createFromIterable = STM.gen(function* () {
  const entries = [["key1", 100], ["key2", 200]] as const
  return yield* TMap.fromIterable(entries)
})
```

### Pattern 2: Basic Map Operations

```typescript
// Reading operations
const readOperations = (tmap: TMap.TMap<string, number>) => STM.gen(function* () {
  const value = yield* TMap.get(tmap, "key1")
  const valueOrDefault = yield* TMap.getOrElse(tmap, "key2", () => 0)
  const hasKey = yield* TMap.has(tmap, "key3")
  const isEmpty = yield* TMap.isEmpty(tmap)
  const size = yield* TMap.size(tmap)
  
  return { value, valueOrDefault, hasKey, isEmpty, size }
})

// Writing operations
const writeOperations = (tmap: TMap.TMap<string, number>) => STM.gen(function* () {
  yield* TMap.set(tmap, "new-key", 42)
  yield* TMap.setIfAbsent(tmap, "conditional-key", 99)
  yield* TMap.remove(tmap, "old-key")
  yield* TMap.updateWith(tmap, "counter", (value) => value ? value + 1 : 1)
  
  return yield* TMap.toHashMap(tmap)
})
```

### Pattern 3: Bulk Operations

```typescript
// Bulk read operations
const bulkReads = (tmap: TMap.TMap<string, number>) => STM.gen(function* () {
  const keys = yield* TMap.keys(tmap)
  const values = yield* TMap.values(tmap)
  const entries = yield* TMap.toArray(tmap)
  const asHashMap = yield* TMap.toHashMap(tmap)
  
  return { keys, values, entries, asHashMap }
})

// Bulk modifications
const bulkModifications = (tmap: TMap.TMap<string, number>) => STM.gen(function* () {
  // Remove multiple keys
  yield* TMap.removeAll(tmap, ["key1", "key2", "key3"])
  
  // Remove conditionally
  yield* TMap.removeIf(tmap, (key, value) => value < 10)
  
  // Keep only matching entries
  yield* TMap.retainIf(tmap, (key, value) => key.startsWith("important-"))
  
  return yield* TMap.size(tmap)
})
```

## Real-World Examples

### Example 1: Thread-Safe In-Memory Cache

```typescript
import { Effect, STM, TMap, Clock } from "effect"

interface CacheEntry<T> {
  readonly value: T
  readonly timestamp: number
  readonly ttl: number
}

class InMemoryCache<K, V> {
  constructor(private cache: TMap.TMap<K, CacheEntry<V>>) {}

  static make = <K, V>(): Effect.Effect<InMemoryCache<K, V>> =>
    STM.gen(function* () {
      const cache = yield* TMap.empty<K, CacheEntry<V>>()
      return new InMemoryCache(cache)
    }).pipe(STM.commit)

  get = (key: K): Effect.Effect<V | null> =>
    Effect.gen(function* () {
      const now = yield* Clock.currentTimeMillis
      
      return yield* STM.gen(function* () {
        const entry = yield* TMap.get(this.cache, key)
        
        if (!entry) {
          return null
        }
        
        // Check if expired
        if (now > entry.timestamp + entry.ttl) {
          yield* TMap.remove(this.cache, key)
          return null
        }
        
        return entry.value
      }).pipe(STM.commit)
    })

  set = (key: K, value: V, ttlMs: number = 300000): Effect.Effect<void> =>
    Effect.gen(function* () {
      const timestamp = yield* Clock.currentTimeMillis
      const entry: CacheEntry<V> = { value, timestamp, ttl: ttlMs }
      
      yield* STM.commit(TMap.set(this.cache, key, entry))
    })

  getOrCompute = <E>(
    key: K,
    compute: () => Effect.Effect<V, E>,
    ttlMs: number = 300000
  ): Effect.Effect<V, E> =>
    Effect.gen(function* () {
      const cached = yield* this.get(key)
      
      if (cached !== null) {
        return cached
      }
      
      const computed = yield* compute()
      yield* this.set(key, computed, ttlMs)
      return computed
    })

  invalidate = (key: K): Effect.Effect<boolean> =>
    STM.gen(function* () {
      const existed = yield* TMap.has(this.cache, key)
      yield* TMap.remove(this.cache, key)
      return existed
    }).pipe(STM.commit)

  invalidatePattern = (predicate: (key: K, value: V) => boolean): Effect.Effect<number> =>
    STM.gen(function* () {
      let removed = 0
      
      yield* TMap.removeIf(this.cache, (key, entry) => {
        const shouldRemove = predicate(key, entry.value)
        if (shouldRemove) removed++
        return shouldRemove
      })
      
      return removed
    }).pipe(STM.commit)

  cleanup = (): Effect.Effect<number> =>
    Effect.gen(function* () {
      const now = yield* Clock.currentTimeMillis
      
      return yield* STM.gen(function* () {
        let cleaned = 0
        
        yield* TMap.removeIf(this.cache, (key, entry) => {
          const expired = now > entry.timestamp + entry.ttl
          if (expired) cleaned++
          return expired
        })
        
        return cleaned
      }).pipe(STM.commit)
    })

  getStats = (): Effect.Effect<{
    size: number
    keys: readonly K[]
    oldestEntry: number | null
  }> =>
    Effect.gen(function* () {
      const now = yield* Clock.currentTimeMillis
      
      return yield* STM.gen(function* () {
        const size = yield* TMap.size(this.cache)
        const keys = yield* TMap.keys(this.cache)
        
        let oldestTimestamp: number | null = null
        
        yield* TMap.forEach(this.cache, (entry) =>
          STM.gen(function* () {
            if (oldestTimestamp === null || entry.timestamp < oldestTimestamp) {
              oldestTimestamp = entry.timestamp
            }
          })
        )
        
        return { size, keys, oldestEntry: oldestTimestamp }
      }).pipe(STM.commit)
    })
}

// Usage
const cacheExample = Effect.gen(function* () {
  const cache = yield* InMemoryCache.make<string, { data: string; count: number }>()
  
  // Simulate expensive computation
  const expensiveComputation = (id: string) =>
    Effect.gen(function* () {
      yield* Effect.sleep("100 millis")
      return { data: `Computed data for ${id}`, count: Math.floor(Math.random() * 100) }
    })
  
  // Multiple concurrent cache operations
  yield* Effect.all([
    cache.getOrCompute("user-1", () => expensiveComputation("user-1"), 5000),
    cache.getOrCompute("user-2", () => expensiveComputation("user-2"), 5000),
    cache.getOrCompute("user-1", () => expensiveComputation("user-1"), 5000), // Should hit cache
  ], { concurrency: "unbounded" })
  
  // Check cache stats
  const stats = yield* cache.getStats()
  
  // Cleanup expired entries
  yield* Effect.sleep("6 seconds")
  const cleanedCount = yield* cache.cleanup()
  
  const finalStats = yield* cache.getStats()
  
  return { initialStats: stats, cleanedCount, finalStats }
})
```

### Example 2: Real-Time Metrics Aggregator

```typescript
import { Effect, STM, TMap, TRef, Schedule } from "effect"

interface MetricData {
  readonly count: number
  readonly sum: number
  readonly min: number
  readonly max: number
  readonly lastUpdate: number
}

interface MetricUpdate {
  readonly name: string
  readonly value: number
  readonly timestamp: number
  readonly tags?: Record<string, string>
}

class MetricsAggregator {
  constructor(
    private metrics: TMap.TMap<string, MetricData>,
    private taggedMetrics: TMap.TMap<string, TMap.TMap<string, MetricData>>
  ) {}

  static make = (): Effect.Effect<MetricsAggregator> =>
    STM.gen(function* () {
      const metrics = yield* TMap.empty<string, MetricData>()
      const taggedMetrics = yield* TMap.empty<string, TMap.TMap<string, MetricData>>()
      return new MetricsAggregator(metrics, taggedMetrics)
    }).pipe(STM.commit)

  recordMetric = (update: MetricUpdate): Effect.Effect<void> =>
    STM.gen(function* () {
      const metricKey = update.name
      
      // Update main metric
      yield* TMap.updateWith(this.metrics, metricKey, (existing) => {
        if (!existing) {
          return {
            count: 1,
            sum: update.value,
            min: update.value,
            max: update.value,
            lastUpdate: update.timestamp
          }
        }
        
        return {
          count: existing.count + 1,
          sum: existing.sum + update.value,
          min: Math.min(existing.min, update.value),
          max: Math.max(existing.max, update.value),
          lastUpdate: update.timestamp
        }
      })
      
      // Update tagged metrics if tags are provided
      if (update.tags) {
        for (const [tagKey, tagValue] of Object.entries(update.tags)) {
          const taggedKey = `${metricKey}.${tagKey}`
          
          // Get or create tagged metric map
          let taggedMap = yield* TMap.get(this.taggedMetrics, taggedKey)
          if (!taggedMap) {
            taggedMap = yield* TMap.empty<string, MetricData>()
            yield* TMap.set(this.taggedMetrics, taggedKey, taggedMap)
          }
          
          // Update tagged metric
          yield* TMap.updateWith(taggedMap, tagValue, (existing) => {
            if (!existing) {
              return {
                count: 1,
                sum: update.value,
                min: update.value,
                max: update.value,
                lastUpdate: update.timestamp
              }
            }
            
            return {
              count: existing.count + 1,
              sum: existing.sum + update.value,
              min: Math.min(existing.min, update.value),
              max: Math.max(existing.max, update.value),
              lastUpdate: update.timestamp
            }
          })
        }
      }
    }).pipe(STM.commit)

  getMetric = (name: string): Effect.Effect<MetricData | null> =>
    STM.commit(TMap.get(this.metrics, name))

  getTaggedMetric = (name: string, tagKey: string, tagValue: string): Effect.Effect<MetricData | null> =>
    STM.gen(function* () {
      const taggedKey = `${name}.${tagKey}`
      const taggedMap = yield* TMap.get(this.taggedMetrics, taggedKey)
      
      if (!taggedMap) {
        return null
      }
      
      return yield* TMap.get(taggedMap, tagValue)
    }).pipe(STM.commit)

  getAllMetrics = (): Effect.Effect<Record<string, MetricData & { average: number }>> =>
    STM.gen(function* () {
      const entries = yield* TMap.toArray(this.metrics)
      const result: Record<string, MetricData & { average: number }> = {}
      
      for (const [name, data] of entries) {
        result[name] = {
          ...data,
          average: data.count > 0 ? data.sum / data.count : 0
        }
      }
      
      return result
    }).pipe(STM.commit)

  getTopMetrics = (limit: number, sortBy: "count" | "sum" | "average" = "count"): Effect.Effect<Array<{
    name: string
    data: MetricData & { average: number }
  }>> =>
    STM.gen(function* () {
      const entries = yield* TMap.toArray(this.metrics)
      
      const enrichedEntries = entries.map(([name, data]) => ({
        name,
        data: {
          ...data,
          average: data.count > 0 ? data.sum / data.count : 0
        }
      }))
      
      const sortedEntries = enrichedEntries.sort((a, b) => {
        switch (sortBy) {
          case "count":
            return b.data.count - a.data.count
          case "sum":
            return b.data.sum - a.data.sum
          case "average":
            return b.data.average - a.data.average
          default:
            return 0
        }
      })
      
      return sortedEntries.slice(0, limit)
    }).pipe(STM.commit)

  resetMetrics = (olderThan?: number): Effect.Effect<number> =>
    STM.gen(function* () {
      let resetCount = 0
      
      if (olderThan) {
        // Reset metrics older than specified timestamp
        yield* TMap.removeIf(this.metrics, (name, data) => {
          const shouldReset = data.lastUpdate < olderThan
          if (shouldReset) resetCount++
          return shouldReset
        })
        
        // Reset tagged metrics
        yield* TMap.removeIf(this.taggedMetrics, (taggedKey, taggedMap) => {
          return STM.gen(function* () {
            const entries = yield* TMap.toArray(taggedMap)
            const hasOldEntries = entries.some(([_, data]) => data.lastUpdate < olderThan)
            
            if (hasOldEntries) {
              yield* TMap.removeIf(taggedMap, (_, data) => data.lastUpdate < olderThan)
              const remainingSize = yield* TMap.size(taggedMap)
              return remainingSize === 0
            }
            
            return false
          })
        })
      } else {
        // Reset all metrics
        const currentSize = yield* TMap.size(this.metrics)
        yield* TMap.removeAll(this.metrics, yield* TMap.keys(this.metrics))
        yield* TMap.removeAll(this.taggedMetrics, yield* TMap.keys(this.taggedMetrics))
        resetCount = currentSize
      }
      
      return resetCount
    }).pipe(STM.commit)
}

// Usage
const metricsExample = Effect.gen(function* () {
  const aggregator = yield* MetricsAggregator.make()
  
  // Simulate metrics collection
  const simulateMetrics = Effect.gen(function* () {
    for (let i = 0; i < 100; i++) {
      const timestamp = Date.now()
      
      // API response times
      yield* aggregator.recordMetric({
        name: "api.response_time",
        value: Math.random() * 1000,
        timestamp,
        tags: {
          endpoint: `/api/v${Math.floor(Math.random() * 3) + 1}/users`,
          method: Math.random() > 0.5 ? "GET" : "POST"
        }
      })
      
      // Error counts
      if (Math.random() > 0.9) {
        yield* aggregator.recordMetric({
          name: "api.errors",
          value: 1,
          timestamp,
          tags: {
            type: Math.random() > 0.5 ? "validation" : "server"
          }
        })
      }
      
      yield* Effect.sleep("10 millis")
    }
  })
  
  // Run metrics collection
  yield* simulateMetrics
  
  // Get aggregated results
  const allMetrics = yield* aggregator.getAllMetrics()
  const topMetrics = yield* aggregator.getTopMetrics(5, "average")
  
  // Get specific tagged metric
  const getResponseTimes = yield* aggregator.getTaggedMetric(
    "api.response_time",
    "method",
    "GET"
  )
  
  return {
    allMetrics,
    topMetrics,
    getResponseTimes
  }
})
```

### Example 3: Session Management System

```typescript
import { Effect, STM, TMap, TRef, Clock, Random } from "effect"

interface Session {
  readonly id: string
  readonly userId: string
  readonly createdAt: number
  readonly lastAccessed: number
  readonly expiresAt: number
  readonly data: Record<string, unknown>
}

interface SessionUpdate {
  readonly lastAccessed?: number
  readonly expiresAt?: number
  readonly data?: Record<string, unknown>
}

class SessionManager {
  constructor(
    private sessions: TMap.TMap<string, Session>,
    private userSessions: TMap.TMap<string, Set<string>>,
    private sessionCounter: TRef.TRef<number>
  ) {}

  static make = (): Effect.Effect<SessionManager> =>
    STM.gen(function* () {
      const sessions = yield* TMap.empty<string, Session>()
      const userSessions = yield* TMap.empty<string, Set<string>>()
      const sessionCounter = yield* TRef.make(0)
      return new SessionManager(sessions, userSessions, sessionCounter)
    }).pipe(STM.commit)

  createSession = (
    userId: string,
    expirationMs: number = 3600000, // 1 hour
    initialData: Record<string, unknown> = {}
  ): Effect.Effect<Session> =>
    Effect.gen(function* () {
      const now = yield* Clock.currentTimeMillis
      const sessionId = yield* Random.nextIntBetween(100000, 999999).pipe(
        Effect.map(n => `session_${userId}_${n}_${now}`)
      )
      
      const session: Session = {
        id: sessionId,
        userId,
        createdAt: now,
        lastAccessed: now,
        expiresAt: now + expirationMs,
        data: initialData
      }
      
      return yield* STM.gen(function* () {
        // Store session
        yield* TMap.set(this.sessions, sessionId, session)
        
        // Track user sessions
        const existingSessions = yield* TMap.getOrElse(
          this.userSessions,
          userId,
          () => new Set<string>()
        )
        const updatedSessions = new Set(existingSessions).add(sessionId)
        yield* TMap.set(this.userSessions, userId, updatedSessions)
        
        // Update counter
        yield* TRef.update(this.sessionCounter, (c) => c + 1)
        
        return session
      }).pipe(STM.commit)
    })

  getSession = (sessionId: string): Effect.Effect<Session | null> =>
    Effect.gen(function* () {
      const now = yield* Clock.currentTimeMillis
      
      return yield* STM.gen(function* () {
        const session = yield* TMap.get(this.sessions, sessionId)
        
        if (!session) {
          return null
        }
        
        // Check if expired
        if (now > session.expiresAt) {
          yield* this.removeSessionInternal(sessionId, session.userId)
          return null
        }
        
        // Update last accessed
        const updatedSession = { ...session, lastAccessed: now }
        yield* TMap.set(this.sessions, sessionId, updatedSession)
        
        return updatedSession
      }).pipe(STM.commit)
    })

  updateSession = (sessionId: string, update: SessionUpdate): Effect.Effect<Session | null> =>
    STM.gen(function* () {
      const session = yield* TMap.get(this.sessions, sessionId)
      
      if (!session) {
        return null
      }
      
      const updatedSession: Session = {
        ...session,
        ...update,
        data: update.data ? { ...session.data, ...update.data } : session.data
      }
      
      yield* TMap.set(this.sessions, sessionId, updatedSession)
      return updatedSession
    }).pipe(STM.commit)

  private removeSessionInternal = (sessionId: string, userId: string): STM.STM<void> =>
    STM.gen(function* () {
      // Remove session
      yield* TMap.remove(this.sessions, sessionId)
      
      // Update user sessions
      const userSessionsSet = yield* TMap.get(this.userSessions, userId)
      if (userSessionsSet) {
        const updatedSet = new Set(userSessionsSet)
        updatedSet.delete(sessionId)
        
        if (updatedSet.size === 0) {
          yield* TMap.remove(this.userSessions, userId)
        } else {
          yield* TMap.set(this.userSessions, userId, updatedSet)
        }
      }
      
      // Update counter
      yield* TRef.update(this.sessionCounter, (c) => c - 1)
    })

  removeSession = (sessionId: string): Effect.Effect<boolean> =>
    STM.gen(function* () {
      const session = yield* TMap.get(this.sessions, sessionId)
      
      if (!session) {
        return false
      }
      
      yield* this.removeSessionInternal(sessionId, session.userId)
      return true
    }).pipe(STM.commit)

  removeAllUserSessions = (userId: string): Effect.Effect<number> =>
    STM.gen(function* () {
      const userSessionsSet = yield* TMap.get(this.userSessions, userId)
      
      if (!userSessionsSet) {
        return 0
      }
      
      let removedCount = 0
      for (const sessionId of userSessionsSet) {
        yield* TMap.remove(this.sessions, sessionId)
        yield* TRef.update(this.sessionCounter, (c) => c - 1)
        removedCount++
      }
      
      yield* TMap.remove(this.userSessions, userId)
      return removedCount
    }).pipe(STM.commit)

  cleanupExpiredSessions = (): Effect.Effect<number> =>
    Effect.gen(function* () {
      const now = yield* Clock.currentTimeMillis
      
      return yield* STM.gen(function* () {
        let cleanedCount = 0
        const allSessions = yield* TMap.toArray(this.sessions)
        
        for (const [sessionId, session] of allSessions) {
          if (now > session.expiresAt) {
            yield* this.removeSessionInternal(sessionId, session.userId)
            cleanedCount++
          }
        }
        
        return cleanedCount
      }).pipe(STM.commit)
    })

  getUserSessions = (userId: string): Effect.Effect<readonly Session[]> =>
    STM.gen(function* () {
      const sessionIds = yield* TMap.getOrElse(
        this.userSessions,
        userId,
        () => new Set<string>()
      )
      
      const sessions: Session[] = []
      for (const sessionId of sessionIds) {
        const session = yield* TMap.get(this.sessions, sessionId)
        if (session) {
          sessions.push(session)
        }
      }
      
      return sessions
    }).pipe(STM.commit)

  getStats = (): Effect.Effect<{
    totalSessions: number
    totalUsers: number
    averageSessionsPerUser: number
    oldestSession: Session | null
  }> =>
    STM.gen(function* () {
      const totalSessions = yield* TRef.get(this.sessionCounter)
      const totalUsers = yield* TMap.size(this.userSessions)
      const averageSessionsPerUser = totalUsers > 0 ? totalSessions / totalUsers : 0
      
      let oldestSession: Session | null = null
      const allSessions = yield* TMap.toArray(this.sessions)
      
      for (const [_, session] of allSessions) {
        if (!oldestSession || session.createdAt < oldestSession.createdAt) {
          oldestSession = session
        }
      }
      
      return { totalSessions, totalUsers, averageSessionsPerUser, oldestSession }
    }).pipe(STM.commit)
}

// Usage
const sessionExample = Effect.gen(function* () {
  const sessionManager = yield* SessionManager.make()
  
  // Create multiple sessions for different users
  const user1Session1 = yield* sessionManager.createSession("user1", 5000, { role: "admin" })
  const user1Session2 = yield* sessionManager.createSession("user1", 5000, { role: "admin" })
  const user2Session1 = yield* sessionManager.createSession("user2", 5000, { role: "user" })
  
  // Simulate session usage
  yield* Effect.sleep("1 second")
  
  // Get and update sessions
  const retrievedSession = yield* sessionManager.getSession(user1Session1.id)
  yield* sessionManager.updateSession(user1Session1.id, {
    data: { lastAction: "view-dashboard", timestamp: Date.now() }
  })
  
  // Get user sessions
  const user1Sessions = yield* sessionManager.getUserSessions("user1")
  
  // Check stats
  const initialStats = yield* sessionManager.getStats()
  
  // Cleanup expired sessions
  yield* Effect.sleep("6 seconds")
  const cleanedCount = yield* sessionManager.cleanupExpiredSessions()
  
  const finalStats = yield* sessionManager.getStats()
  
  return {
    user1Sessions: user1Sessions.length,
    initialStats,
    cleanedCount,
    finalStats
  }
})
```

## Advanced Features Deep Dive

### Feature 1: Advanced Search and Filtering

TMap provides powerful search and filtering capabilities with both synchronous and STM-based predicates:

```typescript
const advancedSearch = (tmap: TMap.TMap<string, number>) => STM.gen(function* () {
  // Find first matching entry
  const firstEven = yield* TMap.findFirst(tmap, (key, value) => value % 2 === 0)
  
  // Find with STM computation
  const complexFind = yield* TMap.findSTM(tmap, (key, value) =>
    STM.gen(function* () {
      // Complex predicate with STM logic
      const threshold = yield* STM.succeed(50)
      return value > threshold && key.startsWith("important")
    })
  )
  
  // Find all matching entries
  const allLarge = yield* TMap.findAll(tmap, (key, value) => value > 100)
  
  return { firstEven, complexFind, allLarge }
})
```

### Feature 2: Reduction and Aggregation

```typescript
const aggregationOperations = (tmap: TMap.TMap<string, number>) => STM.gen(function* () {
  // Simple reduction
  const sum = yield* TMap.reduce(
    tmap,
    0,
    (acc, key, value) => acc + value
  )
  
  // STM-based reduction
  const stmSum = yield* TMap.reduceSTM(
    tmap,
    STM.succeed(0),
    (acc, key, value) => STM.succeed(acc + value)
  )
  
  // Complex aggregation
  const stats = yield* TMap.reduce(
    tmap,
    { count: 0, sum: 0, min: Infinity, max: -Infinity },
    (acc, key, value) => ({
      count: acc.count + 1,
      sum: acc.sum + value,
      min: Math.min(acc.min, value),
      max: Math.max(acc.max, value)
    })
  )
  
  return { sum, stmSum, stats }
})
```

### Feature 3: Take Operations for Atomic Removal

```typescript
const takeOperations = (tmap: TMap.TMap<string, number>) => STM.gen(function* () {
  // Take first matching entry (removes from map)
  const firstLarge = yield* TMap.takeFirst(tmap, (key, value) => value > 100)
  
  // Take with STM computation
  const takenEntry = yield* TMap.takeFirstSTM(tmap, (key, value) =>
    STM.gen(function* () {
      const condition = yield* STM.succeed(value % 2 === 0)
      return condition
    })
  )
  
  // Take multiple entries
  const takenEntries = yield* TMap.takeSome(tmap, (key, value) => value < 10)
  
  return { firstLarge, takenEntry, takenEntries }
})
```

## Practical Patterns & Best Practices

### Pattern 1: Atomic Multi-Map Operations

```typescript
// Helper for atomic operations across multiple maps
const atomicMultiMapUpdate = <K, V>(
  operations: readonly {
    map: TMap.TMap<K, V>
    key: K
    update: (current: V | undefined) => V
  }[]
): STM.STM<void> =>
  STM.gen(function* () {
    for (const { map, key, update } of operations) {
      yield* TMap.updateWith(map, key, update)
    }
  })

// Usage
const multiMapExample = STM.gen(function* () {
  const inventory = yield* TMap.empty<string, number>()
  const reserved = yield* TMap.empty<string, number>()
  
  // Atomic reservation operation
  yield* atomicMultiMapUpdate([
    {
      map: inventory,
      key: "item-1",
      update: (current) => (current || 0) - 5
    },
    {
      map: reserved,
      key: "item-1",
      update: (current) => (current || 0) + 5
    }
  ])
  
  return { inventory: yield* TMap.toHashMap(inventory), reserved: yield* TMap.toHashMap(reserved) }
})
```

### Pattern 2: Conditional Updates with Validation

```typescript
// Helper for conditional updates with validation
const conditionalUpdate = <K, V>(
  tmap: TMap.TMap<K, V>,
  key: K,
  condition: (current: V | undefined) => boolean,
  update: (current: V | undefined) => V
): STM.STM<boolean> =>
  STM.gen(function* () {
    const current = yield* TMap.get(tmap, key)
    
    if (condition(current)) {
      yield* TMap.set(tmap, key, update(current))
      return true
    }
    
    return false
  })

// Usage
const conditionalExample = STM.gen(function* () {
  const balances = yield* TMap.fromIterable([["alice", 100], ["bob", 50]])
  
  // Only update if balance is sufficient
  const transferred = yield* conditionalUpdate(
    balances,
    "alice",
    (balance) => (balance || 0) >= 25,
    (balance) => (balance || 0) - 25
  )
  
  if (transferred) {
    yield* TMap.updateWith(balances, "bob", (balance) => (balance || 0) + 25)
  }
  
  return { transferred, balances: yield* TMap.toHashMap(balances) }
})
```

### Pattern 3: Batch Processing with Error Handling

```typescript
// Helper for batch processing with error recovery
const batchProcess = <K, V, R, E>(
  tmap: TMap.TMap<K, V>,
  processor: (key: K, value: V) => STM.STM<R, E>,
  errorHandler: (key: K, value: V, error: E) => STM.STM<void>
): STM.STM<readonly R[]> =>
  STM.gen(function* () {
    const entries = yield* TMap.toArray(tmap)
    const results: R[] = []
    
    for (const [key, value] of entries) {
      const result = yield* processor(key, value).pipe(
        STM.catchAll((error) =>
          STM.gen(function* () {
            yield* errorHandler(key, value, error)
            return null as any
          })
        )
      )
      
      if (result !== null) {
        results.push(result)
      }
    }
    
    return results
  })
```

## Integration Examples

### Integration with Effect Streams

```typescript
import { Effect, STM, TMap, Stream, Queue } from "effect"

const streamIntegration = Effect.gen(function* () {
  const tmap = yield* STM.commit(TMap.empty<string, number>())
  const updates = yield* Queue.unbounded<{ key: string; value: number }>()
  
  // Stream that processes updates and maintains TMap
  const updateStream = Stream.fromQueue(updates).pipe(
    Stream.mapEffect(({ key, value }) =>
      STM.commit(TMap.updateWith(tmap, key, (current) => (current || 0) + value))
    ),
    Stream.runDrain
  )
  
  // Producer sending updates
  const producer = Effect.gen(function* () {
    for (let i = 0; i < 10; i++) {
      yield* Queue.offer(updates, { key: `metric-${i % 3}`, value: i })
      yield* Effect.sleep("100 millis")
    }
    yield* Queue.shutdown(updates)
  })
  
  // Run stream and producer concurrently
  yield* Effect.race(updateStream, producer)
  
  return yield* STM.commit(TMap.toHashMap(tmap))
})
```

### Testing Strategies

```typescript
import { Effect, STM, TMap, TestClock, TestContext } from "effect"

// Test utility for TMap operations
const testMapOperations = <K, V>(
  initialData: readonly [K, V][],
  operations: (tmap: TMap.TMap<K, V>) => STM.STM<unknown>
) =>
  Effect.gen(function* () {
    const tmap = yield* STM.commit(TMap.fromIterable(initialData))
    yield* STM.commit(operations(tmap))
    return yield* STM.commit(TMap.toHashMap(tmap))
  })

// Concurrent operation test
const testConcurrentUpdates = Effect.gen(function* () {
  const tmap = yield* STM.commit(TMap.fromIterable([["counter", 0]]))
  
  // Multiple concurrent increments
  const increments = Array.from({ length: 100 }, (_, i) =>
    STM.commit(TMap.updateWith(tmap, "counter", (current) => (current || 0) + 1))
  )
  
  yield* Effect.all(increments, { concurrency: "unbounded" })
  
  const result = yield* STM.commit(TMap.get(tmap, "counter"))
  return result // Should be 100 (all increments applied atomically)
}).pipe(
  Effect.provide(TestContext.TestContext)
)

// Property-based testing
const testMapInvariants = (operations: readonly string[]) =>
  Effect.gen(function* () {
    const tmap = yield* STM.commit(TMap.empty<string, number>())
    
    // Apply operations
    for (const op of operations) {
      yield* STM.commit(TMap.set(tmap, op, 1))
    }
    
    // Verify invariants
    const size = yield* STM.commit(TMap.size(tmap))
    const keys = yield* STM.commit(TMap.keys(tmap))
    const uniqueOps = new Set(operations)
    
    return {
      sizeMatches: size === uniqueOps.size,
      keysMatch: keys.length === uniqueOps.size,
      allKeysPresent: [...uniqueOps].every(op => keys.includes(op))
    }
  })
```

## Conclusion

TMap provides thread-safe, transactional key-value storage for concurrent programming, caching systems, and atomic state management.

Key benefits:
- **Atomicity**: All operations are atomic and consistent across concurrent access
- **Rich API**: Comprehensive set of operations for searching, filtering, and aggregation
- **Composability**: Seamlessly integrates with other STM data structures and Effect constructs

TMap is ideal for scenarios requiring shared mutable mappings in concurrent environments, caching systems, and complex state management where consistency and atomicity are critical.