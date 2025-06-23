# TSet: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem TSet Solves

Managing concurrent access to shared sets in multi-fiber applications requires careful synchronization to prevent race conditions. Traditional approaches using locks and mutexes can lead to deadlocks, performance bottlenecks, and error-prone code:

```typescript
// Traditional approach - prone to race conditions and deadlocks
let sharedSet = new Set<number>()
let lock = new Mutex()

async function addToSet(value: number) {
  await lock.acquire()
  try {
    sharedSet.add(value)
  } finally {
    lock.release()
  }
}

async function removeFromSet(value: number) {
  await lock.acquire()
  try {
    sharedSet.delete(value)
  } finally {
    lock.release()
  }
}

// Complex operations require manual coordination
async function moveElement(from: Set<number>, to: Set<number>, value: number) {
  // Risk of deadlock if not careful with lock ordering
  // Risk of partial failure leaving inconsistent state
}
```

This approach leads to:
- **Race Conditions** - Multiple fibers can corrupt the set state
- **Deadlock Potential** - Complex operations with multiple locks can deadlock
- **Error Prone** - Manual lock management is difficult to get right
- **Poor Composability** - Hard to combine operations atomically

### The TSet Solution

TSet (Transactional Set) provides a lock-free, composable solution using Software Transactional Memory (STM). All operations are automatically atomic and composable:

```typescript
import { TSet, STM, Effect } from "effect"

// Create and manipulate sets transactionally
const program = Effect.gen(function* () {
  const userSet = yield* STM.commit(TSet.make("alice", "bob"))
  
  // All operations are automatically atomic
  yield* STM.commit(TSet.add(userSet, "charlie"))
  
  const hasAlice = yield* STM.commit(TSet.has(userSet, "alice"))
  const size = yield* STM.commit(TSet.size(userSet))
  
  return { hasAlice, size }
})
```

### Key Concepts

**Transactional Memory**: All TSet operations execute within STM transactions, providing atomicity and composability without manual locking

**Lock-Free Concurrency**: TSet uses optimistic concurrency control, automatically retrying transactions when conflicts occur

**Composable Operations**: Multiple TSet operations can be combined into single atomic transactions that either all succeed or all fail

## Basic Usage Patterns

### Pattern 1: Creating and Initializing TSets

```typescript
import { TSet, STM, Effect } from "effect"

// Empty set
const createEmptySet = STM.commit(TSet.empty<string>())

// Set with initial values
const createPopulatedSet = STM.commit(TSet.make("apple", "banana", "cherry"))

// From existing iterable
const createFromArray = STM.commit(TSet.fromIterable([1, 2, 3, 4, 5]))

const program = Effect.gen(function* () {
  const emptySet = yield* createEmptySet
  const fruitSet = yield* createPopulatedSet
  const numberSet = yield* createFromArray
  
  const emptySize = yield* STM.commit(TSet.size(emptySet))
  const fruitSize = yield* STM.commit(TSet.size(fruitSet))
  const numberSize = yield* STM.commit(TSet.size(numberSet))
  
  return { emptySize, fruitSize, numberSize } // { 0, 3, 5 }
})
```

### Pattern 2: Basic Set Operations

```typescript
import { TSet, STM, Effect } from "effect"

const setOperations = Effect.gen(function* () {
  const userSet = yield* STM.commit(TSet.make("alice", "bob"))
  
  // Add elements
  yield* STM.commit(TSet.add(userSet, "charlie"))
  yield* STM.commit(TSet.add(userSet, "alice")) // Duplicates ignored
  
  // Check membership
  const hasAlice = yield* STM.commit(TSet.has(userSet, "alice"))
  const hasDave = yield* STM.commit(TSet.has(userSet, "dave"))
  
  // Remove elements
  yield* STM.commit(TSet.remove(userSet, "bob"))
  
  // Get final state
  const finalUsers = yield* STM.commit(TSet.toArray(userSet))
  const finalSize = yield* STM.commit(TSet.size(userSet))
  
  return { hasAlice, hasDave, finalUsers, finalSize }
})
```

### Pattern 3: Atomic Compound Operations

```typescript
import { TSet, STM, Effect } from "effect"

const atomicOperations = Effect.gen(function* () {
  const sourceSet = yield* STM.commit(TSet.make(1, 2, 3, 4, 5))
  const targetSet = yield* STM.commit(TSet.empty<number>())
  
  // Move elements atomically - either all succeed or all fail
  const moveEvenNumbers = STM.gen(function* () {
    const sourceArray = yield* TSet.toArray(sourceSet)
    
    for (const value of sourceArray) {
      if (value % 2 === 0) {
        yield* TSet.remove(sourceSet, value)
        yield* TSet.add(targetSet, value)
      }
    }
  })
  
  yield* STM.commit(moveEvenNumbers)
  
  const sourceResult = yield* STM.commit(TSet.toArray(sourceSet))
  const targetResult = yield* STM.commit(TSet.toArray(targetSet))
  
  return { source: sourceResult, target: targetResult }
})
```

## Real-World Examples

### Example 1: User Session Management

Managing active user sessions in a web application requires thread-safe operations for login, logout, and session validation:

```typescript
import { TSet, STM, Effect, Context } from "effect"

interface User {
  readonly id: string
  readonly username: string
  readonly lastActivity: Date
}

class SessionManager extends Context.Tag("SessionManager")<
  SessionManager,
  {
    readonly activeSessions: TSet.TSet<User>
    readonly login: (user: User) => Effect.Effect<void>
    readonly logout: (userId: string) => Effect.Effect<void>
    readonly isActive: (userId: string) => Effect.Effect<boolean>
    readonly getActiveSessions: () => Effect.Effect<Array<User>>
    readonly cleanupInactive: (timeoutMs: number) => Effect.Effect<number>
  }
>() {}

const makeSessionManager = Effect.gen(function* () {
  const activeSessions = yield* STM.commit(TSet.empty<User>())
  
  const login = (user: User) => 
    STM.commit(TSet.add(activeSessions, user))
  
  const logout = (userId: string) => 
    STM.commit(
      TSet.removeIf(activeSessions, (user) => user.id === userId, { discard: true })
    )
  
  const isActive = (userId: string) => 
    STM.commit(
      TSet.reduce(activeSessions, false, (found, user) => 
        found || user.id === userId
      )
    )
  
  const getActiveSessions = () => 
    STM.commit(TSet.toArray(activeSessions))
  
  const cleanupInactive = (timeoutMs: number) => Effect.gen(function* () {
    const now = new Date()
    const cutoff = new Date(now.getTime() - timeoutMs)
    
    const removedUsers = yield* STM.commit(
      TSet.removeIf(
        activeSessions, 
        (user) => user.lastActivity < cutoff
      )
    )
    
    return removedUsers.length
  })
  
  return {
    activeSessions,
    login,
    logout,
    isActive,
    getActiveSessions,
    cleanupInactive
  } as const
})

// Usage example
const sessionExample = Effect.gen(function* () {
  const sessionManager = yield* makeSessionManager
  
  // Multiple concurrent logins
  yield* Effect.all([
    sessionManager.login({ id: "1", username: "alice", lastActivity: new Date() }),
    sessionManager.login({ id: "2", username: "bob", lastActivity: new Date() }),
    sessionManager.login({ id: "3", username: "charlie", lastActivity: new Date() })
  ], { concurrency: "unbounded" })
  
  const isAliceActive = yield* sessionManager.isActive("1")
  const activeSessions = yield* sessionManager.getActiveSessions()
  
  return { isAliceActive, sessionCount: activeSessions.length }
}).pipe(
  Effect.provideService(SessionManager, yield* makeSessionManager)
)
```

### Example 2: Distributed Task Queue

A thread-safe task queue that supports multiple producers and consumers:

```typescript
import { TSet, STM, Effect, Context, Fiber } from "effect"

interface Task {
  readonly id: string
  readonly type: string
  readonly payload: unknown
  readonly priority: number
  readonly createdAt: Date
}

class TaskQueue extends Context.Tag("TaskQueue")<
  TaskQueue,
  {
    readonly pendingTasks: TSet.TSet<Task>
    readonly activeTasks: TSet.TSet<Task>
    readonly completedTasks: TSet.TSet<Task>
    readonly enqueue: (task: Task) => Effect.Effect<void>
    readonly dequeue: () => Effect.Effect<Task>
    readonly complete: (taskId: string) => Effect.Effect<void>
    readonly getStats: () => Effect.Effect<{
      pending: number
      active: number
      completed: number
    }>
  }
>() {}

const makeTaskQueue = Effect.gen(function* () {
  const pendingTasks = yield* STM.commit(TSet.empty<Task>())
  const activeTasks = yield* STM.commit(TSet.empty<Task>())
  const completedTasks = yield* STM.commit(TSet.empty<Task>())
  
  const enqueue = (task: Task) => 
    STM.commit(TSet.add(pendingTasks, task))
  
  // Atomically move highest priority task from pending to active
  const dequeue = () => STM.commit(
    STM.gen(function* () {
      const tasks = yield* TSet.toArray(pendingTasks)
      
      if (tasks.length === 0) {
        return yield* STM.retry() // Wait for tasks to become available
      }
      
      // Find highest priority task
      const highestPriorityTask = tasks.reduce((highest, current) =>
        current.priority > highest.priority ? current : highest
      )
      
      yield* TSet.remove(pendingTasks, highestPriorityTask)
      yield* TSet.add(activeTasks, highestPriorityTask)
      
      return highestPriorityTask
    })
  )
  
  const complete = (taskId: string) => STM.commit(
    STM.gen(function* () {
      const activeTask = yield* TSet.takeFirst(
        activeTasks,
        (task) => task.id === taskId ? task : undefined
      )
      
      yield* TSet.add(completedTasks, activeTask)
    })
  )
  
  const getStats = () => Effect.gen(function* () {
    const [pending, active, completed] = yield* Effect.all([
      STM.commit(TSet.size(pendingTasks)),
      STM.commit(TSet.size(activeTasks)),
      STM.commit(TSet.size(completedTasks))
    ])
    
    return { pending, active, completed }
  })
  
  return {
    pendingTasks,
    activeTasks,
    completedTasks,
    enqueue,
    dequeue,
    complete,
    getStats
  } as const
})

// Worker that processes tasks
const createWorker = (workerId: string, taskQueue: TaskQueue) => 
  Effect.gen(function* () {
    while (true) {
      try {
        const task = yield* taskQueue.dequeue()
        
        // Simulate task processing
        yield* Effect.sleep(100)
        yield* Effect.log(`Worker ${workerId} completed task ${task.id}`)
        
        yield* taskQueue.complete(task.id)
      } catch (error) {
        yield* Effect.logError(`Worker ${workerId} error: ${error}`)
      }
    }
  }).pipe(
    Effect.forever,
    Effect.fork
  )
```

### Example 3: Real-Time Event Subscription System

Managing real-time subscriptions where clients can subscribe/unsubscribe from topics:

```typescript
import { TSet, STM, Effect, Context } from "effect"

interface Subscription {
  readonly clientId: string
  readonly topic: string
  readonly subscribedAt: Date
}

interface Event {
  readonly topic: string
  readonly data: unknown
  readonly timestamp: Date
}

class EventHub extends Context.Tag("EventHub")<
  EventHub,
  {
    readonly subscriptions: TSet.TSet<Subscription>
    readonly subscribe: (clientId: string, topic: string) => Effect.Effect<void>
    readonly unsubscribe: (clientId: string, topic: string) => Effect.Effect<void>
    readonly publish: (event: Event) => Effect.Effect<Array<string>>
    readonly getSubscribers: (topic: string) => Effect.Effect<Array<string>>
    readonly getClientSubscriptions: (clientId: string) => Effect.Effect<Array<string>>
  }
>() {}

const makeEventHub = Effect.gen(function* () {
  const subscriptions = yield* STM.commit(TSet.empty<Subscription>())
  
  const subscribe = (clientId: string, topic: string) => STM.commit(
    TSet.add(subscriptions, {
      clientId,
      topic,
      subscribedAt: new Date()
    })
  )
  
  const unsubscribe = (clientId: string, topic: string) => STM.commit(
    TSet.removeIf(
      subscriptions,
      (sub) => sub.clientId === clientId && sub.topic === topic,
      { discard: true }
    )
  )
  
  const publish = (event: Event) => STM.commit(
    TSet.reduce(
      subscriptions,
      [] as Array<string>,
      (clients, subscription) =>
        subscription.topic === event.topic
          ? [...clients, subscription.clientId]
          : clients
    )
  )
  
  const getSubscribers = (topic: string) => STM.commit(
    TSet.reduce(
      subscriptions,
      [] as Array<string>,
      (clients, subscription) =>
        subscription.topic === topic
          ? [...clients, subscription.clientId]
          : clients
    )
  )
  
  const getClientSubscriptions = (clientId: string) => STM.commit(
    TSet.reduce(
      subscriptions,
      [] as Array<string>,
      (topics, subscription) =>
        subscription.clientId === clientId
          ? [...topics, subscription.topic]
          : topics
    )
  )
  
  return {
    subscriptions,
    subscribe,
    unsubscribe,
    publish,
    getSubscribers,
    getClientSubscriptions
  } as const
})
```

## Advanced Features Deep Dive

### Set Operations: Union, Intersection, and Difference

TSet provides atomic set operations that modify sets in place:

```typescript
import { TSet, STM, Effect } from "effect"

const setOperationsExample = Effect.gen(function* () {
  const setA = yield* STM.commit(TSet.make(1, 2, 3, 4))
  const setB = yield* STM.commit(TSet.make(3, 4, 5, 6))
  const setC = yield* STM.commit(TSet.make(1, 2, 3, 4))
  
  // Union: setA becomes setA ∪ setB
  yield* STM.commit(TSet.union(setA, setB))
  const unionResult = yield* STM.commit(TSet.toArray(setA))
  
  // Reset setA
  yield* STM.commit(TSet.fromIterable([1, 2, 3, 4])).pipe(
    STM.flatMap((newSet) => 
      STM.gen(function* () {
        yield* TSet.removeAll(setA, yield* TSet.toArray(setA))
        yield* TSet.union(setA, newSet)
      })
    )
  )
  
  // Intersection: setA becomes setA ∩ setB  
  yield* STM.commit(TSet.intersection(setA, setB))
  const intersectionResult = yield* STM.commit(TSet.toArray(setA))
  
  // Difference: setC becomes setC - setB
  yield* STM.commit(TSet.difference(setC, setB))
  const differenceResult = yield* STM.commit(TSet.toArray(setC))
  
  return {
    union: unionResult,        // [1, 2, 3, 4, 5, 6]
    intersection: intersectionResult, // [3, 4]
    difference: differenceResult      // [1, 2]
  }
})
```

### Conditional Operations with takeFirst and takeSome

These operations allow you to atomically extract elements that match specific conditions:

```typescript
import { TSet, STM, Effect, Option } from "effect"

interface User {
  readonly id: string
  readonly role: "admin" | "user" | "guest"
  readonly priority: number
}

const conditionalOperationsExample = Effect.gen(function* () {
  const userQueue = yield* STM.commit(TSet.fromIterable([
    { id: "1", role: "user" as const, priority: 1 },
    { id: "2", role: "admin" as const, priority: 10 },
    { id: "3", role: "guest" as const, priority: 0 },
    { id: "4", role: "admin" as const, priority: 8 },
    { id: "5", role: "user" as const, priority: 3 }
  ]))
  
  // Take the first admin user (removes from set)
  const firstAdmin = yield* STM.commit(
    TSet.takeFirst(userQueue, (user) => 
      user.role === "admin" ? Option.some(user) : Option.none()
    )
  )
  
  // Take all remaining users with priority > 2
  const highPriorityUsers = yield* STM.commit(
    TSet.takeSome(userQueue, (user) =>
      user.priority > 2 ? Option.some(user) : Option.none()
    )
  )
  
  const remainingUsers = yield* STM.commit(TSet.toArray(userQueue))
  
  return {
    firstAdmin,
    highPriorityUsers,
    remaining: remainingUsers
  }
})
```

### Transactional Transformations

Transform all elements in a set atomically:

```typescript
import { TSet, STM, Effect } from "effect"

const transformationExample = Effect.gen(function* () {
  const priceSet = yield* STM.commit(TSet.make(10.0, 15.5, 20.0, 25.99))
  
  // Apply 10% discount to all prices
  yield* STM.commit(
    TSet.transform(priceSet, (price) => price * 0.9)
  )
  
  const discountedPrices = yield* STM.commit(TSet.toArray(priceSet))
  
  // Complex transformation with STM effects
  const complexTransformation = STM.gen(function* () {
    yield* TSet.transformSTM(priceSet, (price) => 
      STM.gen(function* () {
        // Simulate external price validation
        if (price < 1.0) {
          return yield* STM.fail("Price too low")
        }
        return Math.round(price * 100) / 100 // Round to 2 decimal places
      })
    )
  })
  
  yield* STM.commit(complexTransformation)
  const finalPrices = yield* STM.commit(TSet.toArray(priceSet))
  
  return { discountedPrices, finalPrices }
})
```

## Practical Patterns & Best Practices

### Pattern 1: Atomic Multi-Set Operations

```typescript
import { TSet, STM, Effect } from "effect"

// Helper for atomic operations across multiple sets
const atomicMultiSetOperation = <A>(
  operation: (sets: Array<TSet.TSet<A>>) => STM.STM<void>
) => (...sets: Array<TSet.TSet<A>>) => STM.commit(operation(sets))

// Move element between sets atomically
const moveElement = <A>(
  from: TSet.TSet<A>,
  to: TSet.TSet<A>,
  element: A
) => STM.gen(function* () {
  const hasElement = yield* TSet.has(from, element)
  if (hasElement) {
    yield* TSet.remove(from, element)
    yield* TSet.add(to, element)
  }
})

// Swap elements between two sets
const swapElements = <A>(
  setA: TSet.TSet<A>,
  setB: TSet.TSet<A>,
  elementA: A,
  elementB: A
) => STM.gen(function* () {
  const hasA = yield* TSet.has(setA, elementA)
  const hasB = yield* TSet.has(setB, elementB)
  
  if (hasA && hasB) {
    yield* TSet.remove(setA, elementA)
    yield* TSet.remove(setB, elementB)
    yield* TSet.add(setA, elementB)
    yield* TSet.add(setB, elementA)
  }
})

// Usage
const multiSetExample = Effect.gen(function* () {
  const inbox = yield* STM.commit(TSet.make("task1", "task2", "task3"))
  const processing = yield* STM.commit(TSet.empty<string>())
  const completed = yield* STM.commit(TSet.empty<string>())
  
  // Move task through workflow
  yield* STM.commit(moveElement(inbox, processing, "task1"))
  yield* STM.commit(moveElement(processing, completed, "task1"))
  
  const [inboxTasks, processingTasks, completedTasks] = yield* Effect.all([
    STM.commit(TSet.toArray(inbox)),
    STM.commit(TSet.toArray(processing)),
    STM.commit(TSet.toArray(completed))
  ])
  
  return { inbox: inboxTasks, processing: processingTasks, completed: completedTasks }
})
```

### Pattern 2: TSet-based Cache with TTL

```typescript
import { TSet, STM, Effect, Schedule } from "effect"

interface CacheEntry<T> {
  readonly key: string
  readonly value: T
  readonly expiryTime: number
}

const makeTTLCache = <T>() => Effect.gen(function* () {
  const cache = yield* STM.commit(TSet.empty<CacheEntry<T>>())
  
  const put = (key: string, value: T, ttlMs: number) => STM.commit(
    STM.gen(function* () {
      // Remove existing entry
      yield* TSet.removeIf(cache, (entry) => entry.key === key, { discard: true })
      
      // Add new entry
      yield* TSet.add(cache, {
        key,
        value,
        expiryTime: Date.now() + ttlMs
      })
    })
  )
  
  const get = (key: string) => STM.commit(
    TSet.takeFirst(cache, (entry) => {
      if (entry.key === key) {
        return Date.now() < entry.expiryTime ? Option.some(entry.value) : Option.none()
      }
      return Option.none()
    })
  ).pipe(
    Effect.catchAll(() => Effect.succeed(Option.none<T>()))
  )
  
  const cleanup = () => STM.commit(
    TSet.removeIf(cache, (entry) => Date.now() >= entry.expiryTime, { discard: true })
  )
  
  // Auto-cleanup fiber
  const startCleanup = Effect.repeat(
    cleanup,
    Schedule.fixed(5000) // Clean every 5 seconds
  ).pipe(Effect.fork)
  
  return { put, get, cleanup, startCleanup }
})
```

### Pattern 3: Observer Pattern with TSets

```typescript
import { TSet, STM, Effect, Context } from "effect"

interface Observer<T> {
  readonly id: string
  readonly notify: (event: T) => Effect.Effect<void>
}

class Observable<T> extends Context.Tag("Observable")<
  Observable<T>,
  {
    readonly observers: TSet.TSet<Observer<T>>
    readonly subscribe: (observer: Observer<T>) => Effect.Effect<void>
    readonly unsubscribe: (observerId: string) => Effect.Effect<void>
    readonly notify: (event: T) => Effect.Effect<void>
  }
>() {}

const makeObservable = <T>() => Effect.gen(function* () {
  const observers = yield* STM.commit(TSet.empty<Observer<T>>())
  
  const subscribe = (observer: Observer<T>) => 
    STM.commit(TSet.add(observers, observer))
  
  const unsubscribe = (observerId: string) => 
    STM.commit(
      TSet.removeIf(observers, (obs) => obs.id === observerId, { discard: true })
    )
  
  const notify = (event: T) => Effect.gen(function* () {
    const currentObservers = yield* STM.commit(TSet.toArray(observers))
    
    yield* Effect.all(
      currentObservers.map((observer) => observer.notify(event)),
      { concurrency: "unbounded" }
    )
  })
  
  return { observers, subscribe, unsubscribe, notify }
})
```

## Integration Examples

### Integration with Effect Streams

```typescript
import { TSet, STM, Effect, Stream } from "effect"

// Convert TSet changes to a stream
const tsetToStream = <A>(tset: TSet.TSet<A>) => 
  Stream.repeatEffect(
    STM.commit(TSet.toArray(tset)).pipe(
      Effect.delay(100) // Poll every 100ms
    )
  )

// Process stream of set changes
const processSetChanges = Effect.gen(function* () {
  const activeUsers = yield* STM.commit(TSet.empty<string>())
  
  // Simulate user activity
  const addUsersFiber = Effect.gen(function* () {
    yield* Effect.sleep(500)
    yield* STM.commit(TSet.add(activeUsers, "alice"))
    yield* Effect.sleep(500)
    yield* STM.commit(TSet.add(activeUsers, "bob"))
    yield* Effect.sleep(500)
    yield* STM.commit(TSet.remove(activeUsers, "alice"))
  }).pipe(Effect.fork)
  
  // Stream the changes
  const changes = yield* tsetToStream(activeUsers).pipe(
    Stream.take(5),
    Stream.runCollect
  )
  
  yield* Fiber.join(addUsersFiber)
  
  return changes
})
```

### Integration with HTTP Server

```typescript
import { TSet, STM, Effect, Context } from "effect"

interface ConnectionInfo {
  readonly id: string
  readonly ipAddress: string
  readonly connectedAt: Date
}

class ConnectionManager extends Context.Tag("ConnectionManager")<
  ConnectionManager,
  {
    readonly activeConnections: TSet.TSet<ConnectionInfo>
    readonly addConnection: (info: ConnectionInfo) => Effect.Effect<void>
    readonly removeConnection: (id: string) => Effect.Effect<void>
    readonly getActiveConnections: () => Effect.Effect<Array<ConnectionInfo>>
    readonly getConnectionCount: () => Effect.Effect<number>
  }
>() {}

const makeConnectionManager = Effect.gen(function* () {
  const activeConnections = yield* STM.commit(TSet.empty<ConnectionInfo>())
  
  const addConnection = (info: ConnectionInfo) => 
    STM.commit(TSet.add(activeConnections, info))
  
  const removeConnection = (id: string) => 
    STM.commit(
      TSet.removeIf(activeConnections, (conn) => conn.id === id, { discard: true })
    )
  
  const getActiveConnections = () => 
    STM.commit(TSet.toArray(activeConnections))
  
  const getConnectionCount = () => 
    STM.commit(TSet.size(activeConnections))
  
  return {
    activeConnections,
    addConnection,
    removeConnection,
    getActiveConnections,
    getConnectionCount
  } as const
})

// HTTP handler for connection stats
const connectionStatsHandler = Effect.gen(function* () {
  const connectionManager = yield* ConnectionManager
  
  const connections = yield* connectionManager.getActiveConnections()
  const count = yield* connectionManager.getConnectionCount()
  
  return {
    totalConnections: count,
    connections: connections.map(conn => ({
      id: conn.id,
      ipAddress: conn.ipAddress,
      duration: Date.now() - conn.connectedAt.getTime()
    }))
  }
}).pipe(
  Effect.provideService(ConnectionManager, yield* makeConnectionManager)
)
```

### Testing Strategies

```typescript
import { TSet, STM, Effect, TestContext } from "effect"

// Test helper for concurrent operations
const testConcurrentOperations = Effect.gen(function* () {
  const testSet = yield* STM.commit(TSet.empty<number>())
  
  // Simulate concurrent adds
  const addOperations = Array.from({ length: 100 }, (_, i) => 
    STM.commit(TSet.add(testSet, i))
  )
  
  yield* Effect.all(addOperations, { concurrency: "unbounded" })
  
  // Verify all elements were added exactly once
  const finalSize = yield* STM.commit(TSet.size(testSet))
  const finalElements = yield* STM.commit(TSet.toArray(testSet))
  
  return {
    expectedSize: 100,
    actualSize: finalSize,
    hasAllElements: finalElements.length === 100 && 
      finalElements.every(n => n >= 0 && n < 100)
  }
}).pipe(
  Effect.provide(TestContext.TestContext)
)

// Property-based test helper
const testSetInvariants = <A>(
  operations: Array<(set: TSet.TSet<A>) => STM.STM<void>>,
  initialValues: Array<A>
) => Effect.gen(function* () {
  const testSet = yield* STM.commit(TSet.fromIterable(initialValues))
  
  // Apply all operations
  for (const operation of operations) {
    yield* STM.commit(operation(testSet))
  }
  
  // Verify set invariants
  const elements = yield* STM.commit(TSet.toArray(testSet))
  const size = yield* STM.commit(TSet.size(testSet))
  const isEmpty = yield* STM.commit(TSet.isEmpty(testSet))
  
  return {
    sizeMatchesElements: size === elements.length,
    emptyConsistent: isEmpty === (size === 0),
    noDuplicates: new Set(elements).size === elements.length
  }
})
```

## Conclusion

TSet provides **lock-free concurrency**, **composable atomicity**, and **type-safe operations** for managing concurrent access to shared sets in Effect applications.

Key benefits:
- **Deadlock-Free**: STM eliminates the possibility of deadlocks through optimistic concurrency
- **Composable**: Multiple TSet operations can be combined into single atomic transactions
- **Type-Safe**: Full TypeScript support with compile-time guarantees

TSet is ideal for scenarios requiring thread-safe set operations, such as user session management, task queues, event subscription systems, and any application where multiple fibers need to safely manipulate shared collections.