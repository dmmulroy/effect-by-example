# SynchronizedRef: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem SynchronizedRef Solves

Managing concurrent state updates that require effectful operations creates complex synchronization challenges. Traditional approaches with locks, mutexes, or atomic operations become unwieldy when you need to perform effects like database queries, API calls, or file operations as part of state updates:

```typescript
// Traditional approach - complex synchronization with effects
let cache = new Map<string, UserData>();
let cacheLock = new Mutex();

async function getUserWithCache(userId: string): Promise<UserData> {
  await cacheLock.acquire();
  
  try {
    const cached = cache.get(userId);
    if (cached) {
      return cached;
    }
    
    // Problem: We need to release lock during async operation
    // but this creates a race condition
    cacheLock.release();
    const userData = await fetchUserFromDatabase(userId);
    
    await cacheLock.acquire();
    // Another request might have cached this while we were fetching
    if (!cache.has(userId)) {
      cache.set(userId, userData);
    }
    return userData;
  } finally {
    cacheLock.release();
  }
}

// Concurrent requests create race conditions:
Promise.all([
  getUserWithCache('user-1'),
  getUserWithCache('user-1'), // Might fetch twice
  getUserWithCache('user-1')  // Might fetch three times
]);
```

This approach leads to:
- **Race Conditions** - Multiple concurrent updates can corrupt state
- **Complex Lock Management** - Acquiring/releasing locks around async operations is error-prone
- **Deadlock Potential** - Nested locks or forgotten releases cause deadlocks
- **Performance Issues** - Blocking threads while waiting for effects reduces concurrency

### The SynchronizedRef Solution

SynchronizedRef provides atomic, effectful state updates that eliminate race conditions while maintaining high concurrency:

```typescript
import { Effect, SynchronizedRef } from "effect"

// Create a synchronized cache reference
const makeUserCache = Effect.gen(function* () {
  return yield* SynchronizedRef.make(new Map<string, UserData>())
})

const getUserWithCache = (userId: string, cache: SynchronizedRef.SynchronizedRef<Map<string, UserData>>) => {
  return SynchronizedRef.updateEffect(cache, (currentCache) => 
    Effect.gen(function* () {
      if (currentCache.has(userId)) {
        return currentCache; // No update needed
      }
      
      // Effect runs atomically - no race conditions
      const userData = yield* fetchUserFromDatabase(userId)
      return new Map(currentCache).set(userId, userData)
    })
  ).pipe(
    Effect.flatMap(() => SynchronizedRef.get(cache)),
    Effect.map(cache => cache.get(userId)!)
  )
}

// Concurrent requests are automatically synchronized
const program = Effect.gen(function* () {
  const cache = yield* makeUserCache
  
  // All requests are coordinated - only one database fetch per user
  const results = yield* Effect.all([
    getUserWithCache('user-1', cache),
    getUserWithCache('user-1', cache),
    getUserWithCache('user-1', cache)
  ], { concurrency: "unbounded" })
  
  return results
})
```

### Key Concepts

**SynchronizedRef**: A mutable reference that supports atomic, effectful updates to shared state in concurrent environments.

**Atomic Effects**: Effects that run as part of state updates are guaranteed to complete without interference from other concurrent updates.

**Sequential Consistency**: Updates are applied in order, ensuring predictable state transitions even under high concurrency.

## Basic Usage Patterns

### Pattern 1: Creating and Reading SynchronizedRef

```typescript
import { Effect, SynchronizedRef } from "effect"

// Create a SynchronizedRef with initial value
const createCounter = Effect.gen(function* () {
  const counter = yield* SynchronizedRef.make(0)
  return counter
})

// Read current value
const readCounter = (counter: SynchronizedRef.SynchronizedRef<number>) =>
  SynchronizedRef.get(counter)

const program = Effect.gen(function* () {
  const counter = yield* createCounter
  const value = yield* readCounter(counter)
  yield* Effect.log(`Current count: ${value}`)
})
```

### Pattern 2: Simple State Updates

```typescript
// Update with pure function
const increment = (counter: SynchronizedRef.SynchronizedRef<number>) =>
  SynchronizedRef.update(counter, (n) => n + 1)

// Update and get new value
const incrementAndGet = (counter: SynchronizedRef.SynchronizedRef<number>) =>
  SynchronizedRef.updateAndGet(counter, (n) => n + 1)

// Set specific value
const reset = (counter: SynchronizedRef.SynchronizedRef<number>) =>
  SynchronizedRef.set(counter, 0)
```

### Pattern 3: Effectful Updates

```typescript
// Update with effect that might fail
const fetchAndUpdateUser = (
  userRef: SynchronizedRef.SynchronizedRef<Option.Option<User>>,
  userId: string
) =>
  SynchronizedRef.updateEffect(userRef, () =>
    Effect.gen(function* () {
      const user = yield* fetchUser(userId)
      return Option.some(user)
    })
  )

// Modify and return a result
const processOrderAndGetTotal = (
  orderRef: SynchronizedRef.SynchronizedRef<Order>,
  item: OrderItem
) =>
  SynchronizedRef.modifyEffect(orderRef, (order) =>
    Effect.gen(function* () {
      const updatedOrder = { ...order, items: [...order.items, item] }
      const total = yield* calculateOrderTotal(updatedOrder)
      return [total, updatedOrder] as const
    })
  )
```

## Real-World Examples

### Example 1: Concurrent Rate Limiting

A rate limiter that tracks requests per user and enforces limits using database persistence:

```typescript
import { Effect, SynchronizedRef, Schedule, DateTime } from "effect"

interface RateLimitState {
  readonly requestCounts: Map<string, number>
  readonly lastReset: DateTime.DateTime
  readonly limit: number
  readonly windowMs: number
}

const createRateLimiter = (limit: number, windowMs: number) =>
  Effect.gen(function* () {
    const initialState: RateLimitState = {
      requestCounts: new Map(),
      lastReset: yield* DateTime.now,
      limit,
      windowMs
    }
    
    return yield* SynchronizedRef.make(initialState)
  })

const checkRateLimit = (
  rateLimiter: SynchronizedRef.SynchronizedRef<RateLimitState>,
  userId: string
) =>
  SynchronizedRef.modifyEffect(rateLimiter, (state) =>
    Effect.gen(function* () {
      const now = yield* DateTime.now
      const resetNeeded = DateTime.greaterThan(
        now,
        DateTime.add(state.lastReset, { millis: state.windowMs })
      )
      
      let newState = state
      
      // Reset window if needed
      if (resetNeeded) {
        yield* Effect.log(`Resetting rate limit window at ${DateTime.formatIso(now)}`)
        newState = {
          ...state,
          requestCounts: new Map(),
          lastReset: now
        }
      }
      
      const currentCount = newState.requestCounts.get(userId) ?? 0
      const newCount = currentCount + 1
      
      if (newCount > state.limit) {
        return [
          { allowed: false, remainingRequests: 0, resetTime: DateTime.add(newState.lastReset, { millis: state.windowMs }) },
          newState
        ] as const
      }
      
      const updatedState = {
        ...newState,
        requestCounts: new Map(newState.requestCounts).set(userId, newCount)
      }
      
      return [
        { 
          allowed: true, 
          remainingRequests: state.limit - newCount,
          resetTime: DateTime.add(newState.lastReset, { millis: state.windowMs })
        },
        updatedState
      ] as const
    })
  )

// Usage in concurrent API handler
const handleApiRequest = (userId: string, rateLimiter: SynchronizedRef.SynchronizedRef<RateLimitState>) =>
  Effect.gen(function* () {
    const rateCheck = yield* checkRateLimit(rateLimiter, userId)
    
    if (!rateCheck.allowed) {
      return yield* Effect.fail(new RateLimitExceededError({
        userId,
        resetTime: rateCheck.resetTime
      }))
    }
    
    yield* Effect.log(`Processing request for ${userId}, ${rateCheck.remainingRequests} remaining`)
    return yield* processApiRequest(userId)
  })

// Concurrent requests are properly rate limited
const testConcurrentRequests = Effect.gen(function* () {
  const rateLimiter = yield* createRateLimiter(5, 60000) // 5 requests per minute
  
  const requests = Arr.makeBy(10, (i) => 
    handleApiRequest(`user-${i % 3}`, rateLimiter) // 3 users making requests
  )
  
  yield* Effect.all(requests, { 
    concurrency: "unbounded",
    batching: false 
  }).pipe(
    Effect.catchAll(() => Effect.void) // Ignore rate limit errors for demo
  )
})
```

### Example 2: Connection Pool Management

A database connection pool that manages connections with health checks and automatic recovery:

```typescript
interface PooledConnection {
  readonly id: string
  readonly connection: DatabaseConnection
  readonly isHealthy: boolean
  readonly lastUsed: DateTime.DateTime
  readonly createdAt: DateTime.DateTime
}

interface ConnectionPoolState {
  readonly available: PooledConnection[]
  readonly inUse: Map<string, PooledConnection>
  readonly maxConnections: number
  readonly healthCheckInterval: number
}

const createConnectionPool = (maxConnections: number) =>
  Effect.gen(function* () {
    const initialState: ConnectionPoolState = {
      available: [],
      inUse: new Map(),
      maxConnections,
      healthCheckInterval: 30000
    }
    
    return yield* SynchronizedRef.make(initialState)
  })

const acquireConnection = (pool: SynchronizedRef.SynchronizedRef<ConnectionPoolState>) =>
  SynchronizedRef.modifyEffect(pool, (state) =>
    Effect.gen(function* () {
      const now = yield* DateTime.now
      
      // Try to use existing healthy connection
      const healthyConnection = state.available.find(conn => conn.isHealthy)
      
      if (healthyConnection) {
        const updatedAvailable = state.available.filter(conn => conn.id !== healthyConnection.id)
        const updatedInUse = new Map(state.inUse).set(healthyConnection.id, {
          ...healthyConnection,
          lastUsed: now
        })
        
        return [
          healthyConnection,
          { ...state, available: updatedAvailable, inUse: updatedInUse }
        ] as const
      }
      
      // Create new connection if under limit
      const totalConnections = state.available.length + state.inUse.size
      if (totalConnections < state.maxConnections) {
        const newConnection = yield* createNewDatabaseConnection()
        const pooledConnection: PooledConnection = {
          id: yield* generateConnectionId(),
          connection: newConnection,
          isHealthy: true,
          lastUsed: now,
          createdAt: now
        }
        
        const updatedInUse = new Map(state.inUse).set(pooledConnection.id, pooledConnection)
        
        return [
          pooledConnection,
          { ...state, inUse: updatedInUse }
        ] as const
      }
      
      // Pool exhausted
      return yield* Effect.fail(new ConnectionPoolExhaustedError({
        maxConnections: state.maxConnections,
        currentConnections: totalConnections
      }))
    })
  )

const releaseConnection = (
  pool: SynchronizedRef.SynchronizedRef<ConnectionPoolState>,
  connectionId: string,
  isHealthy: boolean = true
) =>
  SynchronizedRef.updateEffect(pool, (state) =>
    Effect.gen(function* () {
      const connection = state.inUse.get(connectionId)
      if (!connection) {
        return state // Connection not in use
      }
      
      const updatedInUse = new Map(state.inUse)
      updatedInUse.delete(connectionId)
      
      if (isHealthy) {
        const updatedConnection = { ...connection, isHealthy: true }
        return {
          ...state,
          available: [...state.available, updatedConnection],
          inUse: updatedInUse
        }
      } else {
        // Close unhealthy connection
        yield* closeConnection(connection.connection)
        return { ...state, inUse: updatedInUse }
      }
    })
  )

// Health check background task
const performHealthCheck = (pool: SynchronizedRef.SynchronizedRef<ConnectionPoolState>) =>
  SynchronizedRef.updateEffect(pool, (state) =>
    Effect.gen(function* () {
      const healthChecks = state.available.map(conn =>
        checkConnectionHealth(conn.connection).pipe(
          Effect.map(isHealthy => ({ ...conn, isHealthy })),
          Effect.catchAll(() => Effect.succeed({ ...conn, isHealthy: false }))
        )
      )
      
      const updatedConnections = yield* Effect.all(healthChecks, { concurrency: 5 })
      
      return { ...state, available: updatedConnections }
    })
  )

// Usage with automatic resource management
const withConnection = <R, E, A>(
  pool: SynchronizedRef.SynchronizedRef<ConnectionPoolState>,
  operation: (conn: DatabaseConnection) => Effect.Effect<A, E, R>
) =>
  Effect.gen(function* () {
    const pooledConnection = yield* acquireConnection(pool)
    
    return yield* operation(pooledConnection.connection).pipe(
      Effect.ensuring(releaseConnection(pool, pooledConnection.id, true)),
      Effect.catchAll((error) =>
        releaseConnection(pool, pooledConnection.id, false).pipe(
          Effect.flatMap(() => Effect.fail(error))
        )
      )
    )
  })
```

### Example 3: Real-Time Metrics Aggregation

A metrics system that aggregates real-time data from multiple sources with periodic persistence:

```typescript
interface MetricPoint {
  readonly timestamp: DateTime.DateTime
  readonly value: number
  readonly tags: Record<string, string>
}

interface AggregatedMetrics {
  readonly count: number
  readonly sum: number
  readonly min: number
  readonly max: number
  readonly average: number
  readonly tags: Record<string, string>
}

interface MetricsState {
  readonly points: MetricPoint[]
  readonly aggregated: Map<string, AggregatedMetrics>
  readonly lastPersisted: DateTime.DateTime
  readonly bufferSize: number
}

const createMetricsCollector = (bufferSize: number = 1000) =>
  Effect.gen(function* () {
    const initialState: MetricsState = {
      points: [],
      aggregated: new Map(),
      lastPersisted: yield* DateTime.now,
      bufferSize
    }
    
    return yield* SynchronizedRef.make(initialState)
  })

const recordMetric = (
  collector: SynchronizedRef.SynchronizedRef<MetricsState>,
  metricName: string,
  value: number,
  tags: Record<string, string> = {}
) =>
  SynchronizedRef.updateEffect(collector, (state) =>
    Effect.gen(function* () {
      const now = yield* DateTime.now
      const newPoint: MetricPoint = { timestamp: now, value, tags }
      
      // Add to buffer
      const updatedPoints = [...state.points, newPoint]
      
      // Update aggregation
      const aggregationKey = `${metricName}:${JSON.stringify(tags)}`
      const current = state.aggregated.get(aggregationKey)
      
      const newAggregated = current
        ? {
            count: current.count + 1,
            sum: current.sum + value,
            min: Math.min(current.min, value),
            max: Math.max(current.max, value),
            average: (current.sum + value) / (current.count + 1),
            tags
          }
        : {
            count: 1,
            sum: value,
            min: value,
            max: value,
            average: value,
            tags
          }
      
      const updatedAggregated = new Map(state.aggregated).set(aggregationKey, newAggregated)
      
      // Check if buffer needs flushing
      if (updatedPoints.length >= state.bufferSize) {
        yield* Effect.log(`Flushing ${updatedPoints.length} metric points to storage`)
        yield* persistMetricPoints(updatedPoints)
        
        return {
          ...state,
          points: [],
          aggregated: updatedAggregated,
          lastPersisted: now
        }
      }
      
      return {
        ...state,
        points: updatedPoints,
        aggregated: updatedAggregated
      }
    })
  )

const getMetricsSummary = (collector: SynchronizedRef.SynchronizedRef<MetricsState>) =>
  SynchronizedRef.get(collector).pipe(
    Effect.map(state => ({
      bufferedPoints: state.points.length,
      aggregatedMetrics: Object.fromEntries(state.aggregated),
      lastPersisted: state.lastPersisted
    }))
  )

// Background flush process
const createPeriodicFlush = (
  collector: SynchronizedRef.SynchronizedRef<MetricsState>,
  intervalMs: number = 30000
) =>
  Effect.gen(function* () {
    yield* SynchronizedRef.updateEffect(collector, (state) =>
      Effect.gen(function* () {
        if (state.points.length > 0) {
          yield* Effect.log(`Periodic flush: ${state.points.length} points`)
          yield* persistMetricPoints(state.points)
          
          return {
            ...state,
            points: [],
            lastPersisted: yield* DateTime.now
          }
        }
        return state
      })
    )
  }).pipe(
    Effect.repeat(Schedule.fixed(intervalMs)),
    Effect.fork
  )

// Concurrent metric recording
const simulateMetricsLoad = (collector: SynchronizedRef.SynchronizedRef<MetricsState>) =>
  Effect.gen(function* () {
    const metricsGenerators = Arr.makeBy(10, (i) =>
      Effect.gen(function* () {
        yield* recordMetric(collector, "request_duration", Math.random() * 100, {
          service: `service-${i % 3}`,
          endpoint: `/api/v${(i % 2) + 1}`
        })
        yield* recordMetric(collector, "error_count", Math.random() < 0.1 ? 1 : 0, {
          service: `service-${i % 3}`
        })
      }).pipe(
        Effect.repeat(Schedule.spaced(100)), // Record every 100ms
        Effect.fork
      )
    )
    
    const generators = yield* Effect.all(metricsGenerators)
    
    // Let it run for 5 seconds
    yield* Effect.sleep(5000)
    
    // Stop generators and show summary
    yield* Effect.all(generators.map(fiber => fiber.interrupt))
    return yield* getMetricsSummary(collector)
  })
```

## Advanced Features Deep Dive

### Feature 1: Conditional Updates with updateSome

updateSome allows you to conditionally update state only when certain conditions are met:

#### Basic updateSome Usage

```typescript
import { Effect, SynchronizedRef, Option } from "effect"

interface UserSession {
  readonly userId: string
  readonly isActive: boolean
  readonly lastActivity: DateTime.DateTime
}

const updateLastActivityIfActive = (
  sessionRef: SynchronizedRef.SynchronizedRef<UserSession>
) =>
  SynchronizedRef.updateSome(sessionRef, (session) =>
    session.isActive
      ? Option.some({ ...session, lastActivity: DateTime.now })
      : Option.none()
  )
```

#### Real-World updateSome Example

```typescript
// Circuit breaker that only updates failure count when not already open
interface CircuitBreakerState {
  readonly state: "closed" | "half-open" | "open"
  readonly failureCount: number
  readonly lastFailure: Option.Option<DateTime.DateTime>
  readonly threshold: number
}

const recordFailure = (
  circuitRef: SynchronizedRef.SynchronizedRef<CircuitBreakerState>
) =>
  SynchronizedRef.updateSomeEffect(circuitRef, (current) =>
    current.state === "open"
      ? Effect.succeed(Option.none()) // Don't update if already open
      : Effect.gen(function* () {
          const now = yield* DateTime.now
          const newFailureCount = current.failureCount + 1
          
          if (newFailureCount >= current.threshold) {
            yield* Effect.log("Circuit breaker opened due to failures")
            return Option.some({
              ...current,
              state: "open" as const,
              failureCount: newFailureCount,
              lastFailure: Option.some(now)
            })
          }
          
          return Option.some({
            ...current,
            failureCount: newFailureCount,
            lastFailure: Option.some(now)
          })
        })
  )
```

#### Advanced updateSome: Optimistic Concurrency Control

```typescript
interface VersionedDocument {
  readonly id: string
  readonly version: number
  readonly content: string
  readonly lastModified: DateTime.DateTime
}

const updateDocumentWithOptimisticLocking = (
  docRef: SynchronizedRef.SynchronizedRef<VersionedDocument>,
  expectedVersion: number,
  newContent: string
) =>
  SynchronizedRef.updateSomeEffect(docRef, (current) =>
    current.version !== expectedVersion
      ? Effect.succeed(Option.none()) // Version mismatch, no update
      : Effect.gen(function* () {
          // Simulate saving to database with version check
          const saved = yield* saveDocumentToDatabase({
            ...current,
            version: current.version + 1,
            content: newContent,
            lastModified: yield* DateTime.now
          })
          
          return Option.some(saved)
        })
  ).pipe(
    Effect.flatMap(Option.match({
      onNone: () => Effect.fail(new OptimisticLockError({ expectedVersion, actualVersion: -1 })),
      onSome: Effect.succeed
    }))
  )
```

### Feature 2: Atomic Read-Modify-Write with modifyEffect

modifyEffect allows you to atomically read, transform, and return a result in one operation:

#### Basic modifyEffect Usage

```typescript
const withdrawFromAccount = (
  accountRef: SynchronizedRef.SynchronizedRef<Account>,
  amount: number
) =>
  SynchronizedRef.modifyEffect(accountRef, (account) =>
    Effect.gen(function* () {
      if (account.balance < amount) {
        return yield* Effect.fail(new InsufficientFundsError({
          requested: amount,
          available: account.balance
        }))
      }
      
      const newBalance = account.balance - amount
      const updatedAccount = { ...account, balance: newBalance }
      const transaction = { amount, newBalance, timestamp: yield* DateTime.now }
      
      return [transaction, updatedAccount] as const
    })
  )
```

#### Real-World modifyEffect Example

```typescript
// Inventory management with reservation system
interface InventoryItem {
  readonly productId: string
  readonly available: number
  readonly reserved: number
  readonly total: number
}

const reserveInventory = (
  inventoryRef: SynchronizedRef.SynchronizedRef<Map<string, InventoryItem>>,
  productId: string,
  quantity: number
) =>
  SynchronizedRef.modifyEffect(inventoryRef, (inventory) =>
    Effect.gen(function* () {
      const item = inventory.get(productId)
      if (!item) {
        return yield* Effect.fail(new ProductNotFoundError({ productId }))
      }
      
      if (item.available < quantity) {
        return yield* Effect.fail(new InsufficientInventoryError({
          productId,
          requested: quantity,
          available: item.available
        }))
      }
      
      // Record reservation in external system
      const reservationId = yield* createInventoryReservation(productId, quantity)
      
      const updatedItem = {
        ...item,
        available: item.available - quantity,
        reserved: item.reserved + quantity
      }
      
      const updatedInventory = new Map(inventory).set(productId, updatedItem)
      
      const reservation = {
        reservationId,
        productId,
        quantity,
        timestamp: yield* DateTime.now
      }
      
      return [reservation, updatedInventory] as const
    })
  )
```

### Feature 3: Coordinated Multi-Reference Updates

For complex state that spans multiple SynchronizedRefs, coordination patterns ensure consistency:

```typescript
interface TransferResult {
  readonly fromBalance: number
  readonly toBalance: number
  readonly transactionId: string
}

const transferBetweenAccounts = (
  fromAccountRef: SynchronizedRef.SynchronizedRef<Account>,
  toAccountRef: SynchronizedRef.SynchronizedRef<Account>,
  amount: number
) =>
  Effect.gen(function* () {
    // Use a deterministic ordering to prevent deadlocks
    const refs = [fromAccountRef, toAccountRef].sort((a, b) => 
      a.toString().localeCompare(b.toString())
    )
    
    // Coordinate updates using nested modifyEffect
    return yield* SynchronizedRef.modifyEffect(refs[0], (firstAccount) =>
      SynchronizedRef.modifyEffect(refs[1], (secondAccount) =>
        Effect.gen(function* () {
          // Determine which account is from/to after sorting
          const isFromFirst = refs[0] === fromAccountRef
          const fromAccount = isFromFirst ? firstAccount : secondAccount
          const toAccount = isFromFirst ? secondAccount : firstAccount
          
          if (fromAccount.balance < amount) {
            return yield* Effect.fail(new InsufficientFundsError({
              requested: amount,
              available: fromAccount.balance
            }))
          }
          
          const transactionId = yield* generateTransactionId()
          
          // Record transaction in external system
          yield* recordTransaction({
            id: transactionId,
            fromAccount: fromAccount.id,
            toAccount: toAccount.id,
            amount,
            timestamp: yield* DateTime.now
          })
          
          const updatedFromAccount = {
            ...fromAccount,
            balance: fromAccount.balance - amount
          }
          
          const updatedToAccount = {
            ...toAccount,
            balance: toAccount.balance + amount
          }
          
          const result: TransferResult = {
            fromBalance: updatedFromAccount.balance,
            toBalance: updatedToAccount.balance,
            transactionId
          }
          
          const newFirstAccount = isFromFirst ? updatedFromAccount : updatedToAccount
          const newSecondAccount = isFromFirst ? updatedToAccount : updatedFromAccount
          
          return [result, [newFirstAccount, newSecondAccount]] as const
        })
      )
    )
  })
```

## Practical Patterns & Best Practices

### Pattern 1: State Machine Implementation

```typescript
type OrderState = "pending" | "processing" | "shipped" | "delivered" | "cancelled"

interface OrderStateMachine {
  readonly orderId: string
  readonly state: OrderState
  readonly history: Array<{ state: OrderState; timestamp: DateTime.DateTime }>
  readonly metadata: Record<string, unknown>
}

const createOrderStateMachine = (orderId: string) =>
  Effect.gen(function* () {
    const initialState: OrderStateMachine = {
      orderId,
      state: "pending",
      history: [{ state: "pending", timestamp: yield* DateTime.now }],
      metadata: {}
    }
    
    return yield* SynchronizedRef.make(initialState)
  })

const transitionOrderState = (
  orderRef: SynchronizedRef.SynchronizedRef<OrderStateMachine>,
  newState: OrderState,
  metadata: Record<string, unknown> = {}
) => {
  const validTransitions: Record<OrderState, OrderState[]> = {
    pending: ["processing", "cancelled"],
    processing: ["shipped", "cancelled"],
    shipped: ["delivered"],
    delivered: [],
    cancelled: []
  }
  
  return SynchronizedRef.updateEffect(orderRef, (current) =>
    Effect.gen(function* () {
      const allowedStates = validTransitions[current.state]
      
      if (!allowedStates.includes(newState)) {
        return yield* Effect.fail(new InvalidStateTransitionError({
          from: current.state,
          to: newState,
          allowed: allowedStates
        }))
      }
      
      // Perform state-specific side effects
      yield* performStateTransitionEffects(current.state, newState, current.orderId)
      
      const now = yield* DateTime.now
      
      return {
        ...current,
        state: newState,
        history: [...current.history, { state: newState, timestamp: now }],
        metadata: { ...current.metadata, ...metadata }
      }
    })
  )
}

const performStateTransitionEffects = (
  fromState: OrderState,
  toState: OrderState,
  orderId: string
) =>
  Effect.gen(function* () {
    if (toState === "processing") {
      yield* Effect.log(`Starting order processing for ${orderId}`)
      yield* notifyWarehouse(orderId)
    } else if (toState === "shipped") {
      yield* Effect.log(`Order ${orderId} shipped`)
      yield* sendShippingNotification(orderId)
    } else if (toState === "delivered") {
      yield* Effect.log(`Order ${orderId} delivered`)
      yield* sendDeliveryConfirmation(orderId)
    } else if (toState === "cancelled") {
      yield* Effect.log(`Order ${orderId} cancelled`)
      yield* processRefund(orderId)
    }
  })
```

### Pattern 2: Event Sourcing with SynchronizedRef

```typescript
interface Event {
  readonly id: string
  readonly type: string
  readonly payload: unknown
  readonly timestamp: DateTime.DateTime
}

interface EventStore {
  readonly events: Event[]
  readonly snapshots: Map<string, unknown>
  readonly lastSnapshotAt: Option.Option<DateTime.DateTime>
}

const createEventStore = Effect.gen(function* () {
  const initialStore: EventStore = {
    events: [],
    snapshots: new Map(),
    lastSnapshotAt: Option.none()
  }
  
  return yield* SynchronizedRef.make(initialStore)
})

const appendEvent = (
  store: SynchronizedRef.SynchronizedRef<EventStore>,
  eventType: string,
  payload: unknown
) =>
  SynchronizedRef.updateEffect(store, (currentStore) =>
    Effect.gen(function* () {
      const event: Event = {
        id: yield* generateEventId(),
        type: eventType,
        payload,
        timestamp: yield* DateTime.now
      }
      
      // Persist event to external storage
      yield* persistEvent(event)
      
      const updatedEvents = [...currentStore.events, event]
      
      // Create snapshot if needed (every 100 events)
      if (updatedEvents.length % 100 === 0) {
        const snapshot = yield* createSnapshot(updatedEvents)
        const snapshotKey = `snapshot-${Math.floor(updatedEvents.length / 100)}`
        
        return {
          ...currentStore,
          events: updatedEvents,
          snapshots: new Map(currentStore.snapshots).set(snapshotKey, snapshot),
          lastSnapshotAt: Option.some(yield* DateTime.now)
        }
      }
      
      return {
        ...currentStore,
        events: updatedEvents
      }
    })
  )

const replayEvents = (
  store: SynchronizedRef.SynchronizedRef<EventStore>,
  fromEvent: number = 0
) =>
  SynchronizedRef.get(store).pipe(
    Effect.flatMap(eventStore =>
      Effect.gen(function* () {
        const eventsToReplay = eventStore.events.slice(fromEvent)
        
        return yield* Effect.reduce(
          eventsToReplay,
          {} as Record<string, unknown>, // Initial state
          (state, event) => applyEvent(state, event)
        )
      })
    )
  )
```

### Pattern 3: Resource Pool with Health Monitoring

```typescript
interface PooledResource<T> {
  readonly id: string
  readonly resource: T
  readonly healthStatus: "healthy" | "degraded" | "unhealthy"
  readonly lastHealthCheck: DateTime.DateTime
  readonly createdAt: DateTime.DateTime
  readonly usageCount: number
}

interface ResourcePool<T> {
  readonly available: PooledResource<T>[]
  readonly inUse: Map<string, PooledResource<T>>
  readonly maxSize: number
  readonly healthCheckInterval: number
}

const createResourcePool = <T>(
  maxSize: number,
  resourceFactory: () => Effect.Effect<T, never, never>
) =>
  Effect.gen(function* () {
    const initialPool: ResourcePool<T> = {
      available: [],
      inUse: new Map(),
      maxSize,
      healthCheckInterval: 30000
    }
    
    const poolRef = yield* SynchronizedRef.make(initialPool)
    
    // Start health check background process
    yield* startHealthCheckProcess(poolRef, resourceFactory)
    
    return poolRef
  })

const acquireResource = <T>(
  pool: SynchronizedRef.SynchronizedRef<ResourcePool<T>>,
  resourceFactory: () => Effect.Effect<T, never, never>
) =>
  SynchronizedRef.modifyEffect(pool, (currentPool) =>
    Effect.gen(function* () {
      // Try to find healthy available resource
      const healthyResource = currentPool.available.find(r => r.healthStatus === "healthy")
      
      if (healthyResource) {
        const updatedAvailable = currentPool.available.filter(r => r.id !== healthyResource.id)
        const resourceInUse = {
          ...healthyResource,
          usageCount: healthyResource.usageCount + 1
        }
        const updatedInUse = new Map(currentPool.inUse).set(healthyResource.id, resourceInUse)
        
        return [
          healthyResource.resource,
          {
            ...currentPool,
            available: updatedAvailable,
            inUse: updatedInUse
          }
        ] as const
      }
      
      // Create new resource if under limit
      const totalResources = currentPool.available.length + currentPool.inUse.size
      if (totalResources < currentPool.maxSize) {
        const newResource = yield* resourceFactory()
        const pooledResource: PooledResource<T> = {
          id: yield* generateResourceId(),
          resource: newResource,
          healthStatus: "healthy",
          lastHealthCheck: yield* DateTime.now,
          createdAt: yield* DateTime.now,
          usageCount: 1
        }
        
        const updatedInUse = new Map(currentPool.inUse).set(pooledResource.id, pooledResource)
        
        return [
          newResource,
          {
            ...currentPool,
            inUse: updatedInUse
          }
        ] as const
      }
      
      // Pool exhausted
      return yield* Effect.fail(new ResourcePoolExhaustedError({ maxSize: currentPool.maxSize }))
    })
  )

const startHealthCheckProcess = <T>(
  pool: SynchronizedRef.SynchronizedRef<ResourcePool<T>>,
  resourceFactory: () => Effect.Effect<T, never, never>
) =>
  Effect.gen(function* () {
    yield* SynchronizedRef.updateEffect(pool, (currentPool) =>
      Effect.gen(function* () {
        const now = yield* DateTime.now
        
        // Check health of available resources
        const healthChecks = currentPool.available.map(resource =>
          checkResourceHealth(resource.resource).pipe(
            Effect.map(isHealthy => ({
              ...resource,
              healthStatus: isHealthy ? "healthy" as const : "unhealthy" as const,
              lastHealthCheck: now
            })),
            Effect.catchAll(() => Effect.succeed({
              ...resource,
              healthStatus: "unhealthy" as const,
              lastHealthCheck: now
            }))
          )
        )
        
        const updatedResources = yield* Effect.all(healthChecks, { concurrency: 5 })
        
        // Remove unhealthy resources and replace if needed
        const healthyResources = updatedResources.filter(r => r.healthStatus !== "unhealthy")
        const removedCount = updatedResources.length - healthyResources.length
        
        if (removedCount > 0) {
          yield* Effect.log(`Removed ${removedCount} unhealthy resources from pool`)
        }
        
        return {
          ...currentPool,
          available: healthyResources
        }
      })
    )
  }).pipe(
    Effect.repeat(Schedule.spaced(30000)), // Check every 30 seconds
    Effect.fork
  )
```

## Integration Examples

### Integration with Layer System

```typescript
import { Context, Layer } from "effect"

// Service interface
interface MetricsService {
  readonly increment: (metric: string, tags?: Record<string, string>) => Effect.Effect<void>
  readonly decrement: (metric: string, tags?: Record<string, string>) => Effect.Effect<void>
  readonly gauge: (metric: string, value: number, tags?: Record<string, string>) => Effect.Effect<void>
  readonly getSummary: () => Effect.Effect<Record<string, number>>
}

const MetricsService = Context.GenericTag<MetricsService>("MetricsService")

// Implementation using SynchronizedRef
const makeMetricsService = Effect.gen(function* () {
  const metricsRef = yield* SynchronizedRef.make(new Map<string, number>())
  
  const increment = (metric: string, tags: Record<string, string> = {}) => {
    const key = createMetricKey(metric, tags)
    return SynchronizedRef.update(metricsRef, (metrics) =>
      new Map(metrics).set(key, (metrics.get(key) ?? 0) + 1)
    )
  }
  
  const decrement = (metric: string, tags: Record<string, string> = {}) => {
    const key = createMetricKey(metric, tags)
    return SynchronizedRef.update(metricsRef, (metrics) =>
      new Map(metrics).set(key, Math.max(0, (metrics.get(key) ?? 0) - 1))
    )
  }
  
  const gauge = (metric: string, value: number, tags: Record<string, string> = {}) => {
    const key = createMetricKey(metric, tags)
    return SynchronizedRef.update(metricsRef, (metrics) =>
      new Map(metrics).set(key, value)
    )
  }
  
  const getSummary = () =>
    SynchronizedRef.get(metricsRef).pipe(
      Effect.map(metrics => Object.fromEntries(metrics))
    )
  
  return { increment, decrement, gauge, getSummary } as const satisfies MetricsService
})

// Layer providing the service
const MetricsServiceLive = Layer.effect(MetricsService, makeMetricsService)

// Usage in application
const applicationLogic = Effect.gen(function* () {
  const metrics = yield* MetricsService
  
  yield* metrics.increment("requests.total", { endpoint: "/api/users" })
  yield* metrics.gauge("memory.usage", 85.5)
  
  const summary = yield* metrics.getSummary()
  yield* Effect.log("Current metrics", summary)
})

const program = applicationLogic.pipe(
  Effect.provide(MetricsServiceLive)
)
```

### Integration with Streams

```typescript
import { Stream, Sink } from "effect"

// Real-time data aggregation with SynchronizedRef
interface StreamingAggregator<T> {
  readonly add: (item: T) => Effect.Effect<void>
  readonly getState: () => Effect.Effect<T[]>
  readonly stream: Stream.Stream<T[]>
}

const createStreamingAggregator = <T>(
  windowSize: number = 100
): Effect.Effect<StreamingAggregator<T>> =>
  Effect.gen(function* () {
    const bufferRef = yield* SynchronizedRef.make<T[]>([])
    const subscribersRef = yield* SynchronizedRef.make<Set<Sink.Sink<void, never, T[], never, never>>>(new Set())
    
    const add = (item: T) =>
      SynchronizedRef.updateEffect(bufferRef, (buffer) =>
        Effect.gen(function* () {
          const newBuffer = [...buffer, item]
          
          if (newBuffer.length >= windowSize) {
            // Notify subscribers
            const subscribers = yield* SynchronizedRef.get(subscribersRef)
            yield* Effect.all(
              Arr.fromIterable(subscribers).map(sink =>
                Stream.fromIterable(newBuffer).pipe(
                  Stream.run(sink),
                  Effect.catchAll(() => Effect.void) // Handle subscriber errors gracefully
                )
              ),
              { concurrency: "unbounded" }
            )
            
            return [] // Reset buffer
          }
          
          return newBuffer
        })
      )
    
    const getState = () => SynchronizedRef.get(bufferRef)
    
    const stream = Stream.async<T[]>((emit, signal) => {
      const sink = Sink.forEach((batch: T[]) => emit.single(batch))
      
      return SynchronizedRef.update(subscribersRef, subscribers =>
        new Set(subscribers).add(sink)
      ).pipe(
        Effect.ensuring(
          SynchronizedRef.update(subscribersRef, subscribers => {
            const newSubscribers = new Set(subscribers)
            newSubscribers.delete(sink)
            return newSubscribers
          })
        )
      )
    })
    
    return { add, getState, stream } as const
  })

// Usage with Stream processing
const processDataStream = Effect.gen(function* () {
  const aggregator = yield* createStreamingAggregator<number>(5)
  
  // Producer: Add data points
  const producer = Effect.gen(function* () {
    for (let i = 0; i < 20; i++) {
      yield* aggregator.add(Math.random() * 100)
      yield* Effect.sleep(100)
    }
  }).pipe(Effect.fork)
  
  // Consumer: Process batches
  const consumer = aggregator.stream.pipe(
    Stream.tap(batch => Effect.log(`Processing batch of ${batch.length} items: ${batch.join(", ")}`)),
    Stream.take(4), // Take 4 batches
    Stream.runCollect
  )
  
  yield* producer
  const results = yield* consumer
  
  return results
})
```

### Testing Strategies

```typescript
import { TestContext, TestClock } from "effect"

// Test utilities for SynchronizedRef
const makeSynchronizedRefTestSuite = <T>(
  name: string,
  initialValue: T,
  operations: Array<{
    name: string
    operation: (ref: SynchronizedRef.SynchronizedRef<T>) => Effect.Effect<unknown>
    expectedValue?: T
  }>
) =>
  Effect.gen(function* () {
    yield* Effect.log(`Testing SynchronizedRef: ${name}`)
    
    const ref = yield* SynchronizedRef.make(initialValue)
    
    for (const { name: opName, operation, expectedValue } of operations) {
      yield* Effect.log(`  Running operation: ${opName}`)
      yield* operation(ref)
      
      if (expectedValue !== undefined) {
        const currentValue = yield* SynchronizedRef.get(ref)
        if (!Equal.equals(currentValue, expectedValue)) {
          yield* Effect.fail(new TestFailure({
            operation: opName,
            expected: expectedValue,
            actual: currentValue
          }))
        }
        yield* Effect.log(`    ✓ Value matches expected: ${JSON.stringify(expectedValue)}`)
      }
    }
    
    yield* Effect.log(`  All operations completed successfully`)
  })

// Concurrent operation testing
const testConcurrentUpdates = Effect.gen(function* () {
  const counter = yield* SynchronizedRef.make(0)
  
  // Run 100 concurrent increments
  const increments = Arr.makeBy(100, () =>
    SynchronizedRef.update(counter, n => n + 1)
  )
  
  yield* Effect.all(increments, { concurrency: "unbounded" })
  
  const finalValue = yield* SynchronizedRef.get(counter)
  
  if (finalValue !== 100) {
    yield* Effect.fail(new TestFailure({
      operation: "concurrent increments",
      expected: 100,
      actual: finalValue
    }))
  }
  
  yield* Effect.log("✓ Concurrent updates maintained consistency")
})

// Property-based testing with generators
const testSynchronizedRefProperties = Effect.gen(function* () {
  const testCases = yield* Effect.all([
    generateRandomOperations(50),
    generateRandomOperations(100),
    generateRandomOperations(200)
  ])
  
  for (const operations of testCases) {
    const ref = yield* SynchronizedRef.make(0)
    
    // Apply operations sequentially to get expected result
    let expectedValue = 0
    for (const op of operations) {
      expectedValue = applyOperation(expectedValue, op)
    }
    
    // Apply operations concurrently
    const effects = operations.map(op => 
      applyOperationToRef(ref, op)
    )
    
    yield* Effect.all(effects, { concurrency: "unbounded" })
    
    const actualValue = yield* SynchronizedRef.get(ref)
    
    if (actualValue !== expectedValue) {
      yield* Effect.fail(new PropertyTestFailure({
        operations,
        expected: expectedValue,
        actual: actualValue
      }))
    }
  }
  
  yield* Effect.log("✓ Property-based tests passed")
})

// Run all tests
const testSuite = Effect.gen(function* () {
  yield* makeSynchronizedRefTestSuite("Counter", 0, [
    { name: "increment", operation: ref => SynchronizedRef.update(ref, n => n + 1), expectedValue: 1 },
    { name: "decrement", operation: ref => SynchronizedRef.update(ref, n => n - 1), expectedValue: 0 },
    { name: "set to 10", operation: ref => SynchronizedRef.set(ref, 10), expectedValue: 10 }
  ])
  
  yield* testConcurrentUpdates
  yield* testSynchronizedRefProperties
  
  yield* Effect.log("All SynchronizedRef tests passed! ✓")
}).pipe(
  Effect.provide(TestContext.TestContext)
)
```

## Conclusion

SynchronizedRef provides **atomic effectful state management**, **concurrent safety**, and **composable concurrency patterns** for complex state coordination in Effect applications.

Key benefits:
- **Race-Free Updates**: Eliminates race conditions in concurrent state modifications
- **Effectful Operations**: Supports database queries, API calls, and other effects within state updates
- **High Performance**: Maintains concurrency while ensuring consistency
- **Composable**: Integrates seamlessly with other Effect modules and patterns

Use SynchronizedRef when you need to coordinate state changes that involve effects, manage shared resources with complex lifecycle requirements, or implement sophisticated concurrency patterns like rate limiting, connection pooling, or real-time data aggregation.