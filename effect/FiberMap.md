# FiberMap: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem FiberMap Solves

When building concurrent applications, you often need to manage multiple long-running fibers identified by keys. Traditional approaches lead to several problems:

```typescript
// Traditional approach - manual fiber management
const userConnections = new Map<string, Fiber.RuntimeFiber<void, Error>>()

function addConnection(userId: string, effect: Effect.Effect<void, Error>) {
  return Effect.gen(function* () {
    // Manual cleanup of existing fiber
    const existing = userConnections.get(userId)
    if (existing) {
      yield* Fiber.interrupt(existing)
    }
    
    // Manual fiber tracking
    const fiber = yield* Effect.fork(effect)
    userConnections.set(userId, fiber)
    
    // Manual cleanup when fiber completes - easy to forget!
    fiber.addObserver(() => {
      userConnections.delete(userId)
    })
  })
}

// Manual cleanup on application shutdown
function cleanup() {
  return Effect.gen(function* () {
    for (const [, fiber] of userConnections) {
      yield* Fiber.interrupt(fiber)
    }
    userConnections.clear()
  })
}
```

This approach leads to:
- **Memory Leaks** - Forgetting to remove completed fibers from the map
- **Resource Management Complexity** - Manual cleanup of fibers on scope exit
- **Race Conditions** - Concurrent access to the fiber map without proper synchronization
- **Error Propagation Issues** - Difficulty in handling errors from multiple fibers

### The FiberMap Solution

FiberMap provides a managed collection of keyed fibers with automatic cleanup and lifecycle management:

```typescript
import { Effect, FiberMap } from "effect"

function addConnection(map: FiberMap<string>, userId: string, effect: Effect.Effect<void, Error>) {
  // Automatic cleanup of existing fiber + automatic removal when complete
  return FiberMap.run(map, userId, effect)
}

// Usage with automatic cleanup
const program = Effect.gen(function* () {
  const map = yield* FiberMap.make<string>()
  
  yield* addConnection(map, "user1", Effect.never)
  yield* addConnection(map, "user2", Effect.never)
  
  // All fibers automatically interrupted when scope closes
}).pipe(Effect.scoped)
```

### Key Concepts

**FiberMap<K, A, E>**: A managed collection of fibers indexed by keys of type `K`, where each fiber produces `A` or fails with `E`

**Automatic Cleanup**: Fibers are automatically removed from the map when they complete, and all fibers are interrupted when the scope closes

**Key Replacement**: Adding a fiber with an existing key automatically interrupts the previous fiber

## Basic Usage Patterns

### Pattern 1: Creating and Using a FiberMap

```typescript
import { Effect, FiberMap } from "effect"

// Create a FiberMap for string keys
const program = Effect.gen(function* () {
  const map = yield* FiberMap.make<string>()
  
  // Run effects with automatic fiber management
  yield* FiberMap.run(map, "task-1", Effect.log("Task 1 running"))
  yield* FiberMap.run(map, "task-2", Effect.log("Task 2 running"))
  
  // Check map size
  const size = yield* FiberMap.size(map)
  yield* Effect.log(`Map contains ${size} fibers`)
}).pipe(Effect.scoped) // Fibers cleaned up when scope exits
```

### Pattern 2: Replacing Fibers by Key

```typescript
const connectionManager = Effect.gen(function* () {
  const connections = yield* FiberMap.make<string>()
  
  // Start connection for user
  yield* FiberMap.run(connections, "user123", simulateConnection("user123"))
  
  // Replace connection (previous fiber is automatically interrupted)
  yield* FiberMap.run(connections, "user123", simulateConnection("user123"))
  
  yield* Effect.log("Connection replaced seamlessly")
}).pipe(Effect.scoped)
```

### Pattern 3: Conditional Fiber Creation

```typescript
const taskScheduler = Effect.gen(function* () {
  const tasks = yield* FiberMap.make<string>()
  
  // Only create if missing
  yield* FiberMap.run(tasks, "periodic-sync", periodicSync, {
    onlyIfMissing: true
  })
  
  // This won't replace the existing fiber
  yield* FiberMap.run(tasks, "periodic-sync", periodicSync, {
    onlyIfMissing: true
  })
}).pipe(Effect.scoped)
```

## Real-World Examples

### Example 1: WebSocket Connection Manager

Managing multiple WebSocket connections per user with automatic cleanup:

```typescript
import { Effect, FiberMap, Ref, Schedule } from "effect"

interface WebSocketConnection {
  readonly userId: string
  readonly send: (message: string) => Effect.Effect<void>
  readonly close: () => Effect.Effect<void>
}

const createConnectionManager = () => {
  return Effect.gen(function* () {
    const connections = yield* FiberMap.make<string>()
    const messageQueue = yield* Ref.make<Map<string, Array<string>>>(new Map())
    
    const addConnection = (connection: WebSocketConnection) => {
      const connectionEffect = Effect.gen(function* () {
        yield* Effect.log(`Connection opened for user ${connection.userId}`)
        
        // Send queued messages
        const queue = yield* Ref.get(messageQueue)
        const userMessages = queue.get(connection.userId) ?? []
        
        yield* Effect.forEach(userMessages, (message) =>
          connection.send(message)
        )
        
        // Clear queue for this user
        yield* Ref.update(messageQueue, (map) => {
          const newMap = new Map(map)
          newMap.delete(connection.userId)
          return newMap
        })
        
        // Keep connection alive with heartbeat
        yield* Effect.repeat(
          connection.send("ping"),
          Schedule.fixed("30 seconds")
        )
      }).pipe(
        Effect.onExit((exit) =>
          Effect.gen(function* () {
            yield* Effect.log(`Connection closed for user ${connection.userId}`)
            yield* connection.close()
          })
        )
      )
      
      return FiberMap.run(connections, connection.userId, connectionEffect)
    }
    
    const removeConnection = (userId: string) =>
      FiberMap.remove(connections, userId)
    
    const sendMessage = (userId: string, message: string) =>
      Effect.gen(function* () {
        const fiber = yield* FiberMap.unsafeGet(connections, userId)
        
        if (fiber._tag === "Some") {
          // Connection exists, send directly
          const connection = yield* getConnectionFromFiber(fiber.value)
          yield* connection.send(message)
        } else {
          // Queue message for when user connects
          yield* Ref.update(messageQueue, (map) => {
            const newMap = new Map(map)
            const existing = newMap.get(userId) ?? []
            newMap.set(userId, [...existing, message])
            return newMap
          })
        }
      })
    
    const getActiveUsers = () =>
      Effect.gen(function* () {
        const size = yield* FiberMap.size(connections)
        const users: string[] = []
        for (const [userId] of connections) {
          users.push(userId)
        }
        return { count: size, users }
      })
    
    return {
      addConnection,
      removeConnection,
      sendMessage,
      getActiveUsers
    } as const
  })
}

// Usage
const webSocketServer = Effect.gen(function* () {
  const manager = yield* createConnectionManager()
  
  // Simulate connections
  yield* manager.addConnection({
    userId: "user1",
    send: (msg) => Effect.log(`Sending to user1: ${msg}`),
    close: () => Effect.log("Closing user1 connection")
  })
  
  yield* manager.sendMessage("user1", "Hello!")
  
  const stats = yield* manager.getActiveUsers()
  yield* Effect.log(`Active users: ${stats.count}`)
  
}).pipe(Effect.scoped)
```

### Example 2: Background Task Orchestration

Managing multiple background tasks with different lifecycles:

```typescript
import { Effect, FiberMap, Schedule, Ref } from "effect"

interface TaskConfig {
  readonly name: string
  readonly interval: Duration.Duration
  readonly maxRetries: number
  readonly effect: Effect.Effect<void, Error>
}

const createTaskOrchestrator = () => {
  return Effect.gen(function* () {
    const tasks = yield* FiberMap.make<string>()
    const taskStats = yield* Ref.make<Map<string, TaskStats>>(new Map())
    
    const registerTask = (config: TaskConfig) => {
      const taskEffect = Effect.gen(function* () {
        yield* Effect.log(`Starting task: ${config.name}`)
        
        yield* Effect.repeat(
          Effect.gen(function* () {
            const startTime = yield* Effect.clockWith(_ => _.currentTimeMillis)
            
            yield* config.effect.pipe(
              Effect.catchAll((error) =>
                Effect.gen(function* () {
                  yield* updateTaskStats(config.name, (stats) => ({
                    ...stats,
                    failures: stats.failures + 1,
                    lastError: Option.some(error)
                  }))
                  return Effect.fail(error)
                })
              ),
              Effect.retry(Schedule.exponential("1 second").pipe(
                Schedule.compose(Schedule.recurs(config.maxRetries))
              )),
              Effect.tap(() =>
                Effect.gen(function* () {
                  const endTime = yield* Effect.clockWith(_ => _.currentTimeMillis)
                  yield* updateTaskStats(config.name, (stats) => ({
                    ...stats,
                    successes: stats.successes + 1,
                    lastRun: Option.some(new Date(endTime)),
                    averageRuntime: calculateAverage(stats.runtimes, endTime - startTime)
                  }))
                })
              )
            )
          }),
          Schedule.fixed(config.interval)
        )
      }).pipe(
        Effect.onInterrupt(() =>
          Effect.log(`Task interrupted: ${config.name}`)
        )
      )
      
      return FiberMap.run(tasks, config.name, taskEffect, {
        propagateInterruption: false // Don't fail the whole map if one task fails
      })
    }
    
    const stopTask = (name: string) =>
      FiberMap.remove(tasks, name)
    
    const getTaskStats = () =>
      Ref.get(taskStats)
    
    const updateTaskStats = (name: string, update: (stats: TaskStats) => TaskStats) =>
      Ref.update(taskStats, (map) => {
        const newMap = new Map(map)
        const current = newMap.get(name) ?? defaultTaskStats()
        newMap.set(name, update(current))
        return newMap
      })
    
    const restartTask = (name: string, config: TaskConfig) =>
      Effect.gen(function* () {
        yield* stopTask(name)
        yield* Effect.sleep("100 millis") // Brief pause
        yield* registerTask(config)
      })
    
    return {
      registerTask,
      stopTask,
      restartTask,
      getTaskStats
    } as const
  })
}

interface TaskStats {
  readonly successes: number
  readonly failures: number
  readonly lastRun: Option.Option<Date>
  readonly lastError: Option.Option<Error>
  readonly runtimes: Array<number>
  readonly averageRuntime: number
}

const defaultTaskStats = (): TaskStats => ({
  successes: 0,
  failures: 0,
  lastRun: Option.none(),
  lastError: Option.none(),
  runtimes: [],
  averageRuntime: 0
})

// Usage
const backgroundServices = Effect.gen(function* () {
  const orchestrator = yield* createTaskOrchestrator()
  
  // Register various background tasks
  yield* orchestrator.registerTask({
    name: "data-sync",
    interval: Duration.minutes(5),
    maxRetries: 3,
    effect: syncDataFromAPI
  })
  
  yield* orchestrator.registerTask({
    name: "cleanup",
    interval: Duration.hours(1),
    maxRetries: 1,
    effect: cleanupTempFiles
  })
  
  yield* orchestrator.registerTask({
    name: "health-check",
    interval: Duration.seconds(30),
    maxRetries: 0,
    effect: performHealthCheck
  })
  
  // Let tasks run for a while
  yield* Effect.sleep("10 minutes")
  
  // Check stats
  const stats = yield* orchestrator.getTaskStats()
  yield* Effect.log(`Task statistics: ${JSON.stringify(Array.from(stats.entries()))}`)
  
}).pipe(Effect.scoped)
```

### Example 3: HTTP Request Deduplication

Prevent duplicate HTTP requests for the same resource using FiberMap:

```typescript
import { Effect, FiberMap, HttpClient } from "effect"

interface CacheEntry<A> {
  readonly data: A
  readonly timestamp: number
  readonly ttl: number
}

const createRequestDeduplicator = <R>(httpClient: HttpClient.HttpClient) => {
  return Effect.gen(function* () {
    const inFlightRequests = yield* FiberMap.make<string>()
    const cache = yield* Ref.make<Map<string, CacheEntry<unknown>>>(new Map())
    
    const request = <A>(
      url: string,
      options: {
        readonly decode: (response: unknown) => Effect.Effect<A, Error>
        readonly ttl?: Duration.Duration
        readonly deduplicate?: boolean
      }
    ): Effect.Effect<A, Error, R> => {
      return Effect.gen(function* () {
        const cacheKey = `${url}:${JSON.stringify(options)}`
        const now = yield* Effect.clockWith(_ => _.currentTimeMillis)
        const ttlMs = Duration.toMillis(options.ttl ?? Duration.minutes(5))
        
        // Check cache first
        const cacheMap = yield* Ref.get(cache)
        const cached = cacheMap.get(cacheKey)
        
        if (cached && (now - cached.timestamp) < cached.ttl) {
          yield* Effect.log(`Cache hit for ${url}`)
          return cached.data as A
        }
        
        // Check if request is already in flight
        if (options.deduplicate !== false) {
          const existingFiber = yield* FiberMap.unsafeGet(inFlightRequests, cacheKey)
          
          if (existingFiber._tag === "Some") {
            yield* Effect.log(`Deduplicating request for ${url}`)
            return yield* Fiber.join(existingFiber.value) as Effect.Effect<A, Error>
          }
        }
        
        // Make the request
        const requestEffect = Effect.gen(function* () {
          yield* Effect.log(`Making HTTP request to ${url}`)
          
          const response = yield* HttpClient.get(url).pipe(
            Effect.provideService(HttpClient.HttpClient, httpClient),
            Effect.flatMap((response) => response.json),
            Effect.flatMap(options.decode)
          )
          
          // Update cache
          yield* Ref.update(cache, (map) => {
            const newMap = new Map(map)
            newMap.set(cacheKey, {
              data: response,
              timestamp: now,
              ttl: ttlMs
            })
            return newMap
          })
          
          return response
        }).pipe(
          Effect.onExit(() =>
            // Remove from in-flight requests when done
            Effect.sync(() => {
              // This will be handled automatically by FiberMap
            })
          )
        )
        
        return yield* FiberMap.run(inFlightRequests, cacheKey, requestEffect).pipe(
          Effect.flatMap(Fiber.join)
        )
      })
    }
    
    const clearCache = () =>
      Ref.set(cache, new Map())
    
    const getCacheStats = () =>
      Effect.gen(function* () {
        const cacheMap = yield* Ref.get(cache)
        const inFlight = yield* FiberMap.size(inFlightRequests)
        
        return {
          cachedEntries: cacheMap.size,
          inFlightRequests: inFlight
        }
      })
    
    return {
      request,
      clearCache,
      getCacheStats
    } as const
  })
}

// Usage
const apiClient = Effect.gen(function* () {
  const httpClient = yield* HttpClient.HttpClient
  const deduplicator = yield* createRequestDeduplicator(httpClient)
  
  // These requests will be deduplicated
  const responses = yield* Effect.all([
    deduplicator.request("/api/users", {
      decode: (data) => Effect.succeed(data as User[])
    }),
    deduplicator.request("/api/users", {
      decode: (data) => Effect.succeed(data as User[])
    }),
    deduplicator.request("/api/users", {
      decode: (data) => Effect.succeed(data as User[])
    })
  ], { concurrency: "unbounded" })
  
  const stats = yield* deduplicator.getCacheStats()
  yield* Effect.log(`API stats: ${JSON.stringify(stats)}`)
  
  return responses[0] // All responses will be identical
}).pipe(Effect.scoped)
```

## Advanced Features Deep Dive

### Feature 1: Fiber Lifecycle Management

FiberMap provides sophisticated control over fiber lifecycles with options for interruption handling and error propagation.

#### Basic Lifecycle Control

```typescript
const lifecycleDemo = Effect.gen(function* () {
  const map = yield* FiberMap.make<string>()
  
  // Standard behavior: fiber replaced on duplicate key
  yield* FiberMap.run(map, "task", longRunningTask)
  yield* FiberMap.run(map, "task", anotherTask) // First task interrupted
  
  // Conditional creation: only if missing
  yield* FiberMap.run(map, "task", yetAnotherTask, {
    onlyIfMissing: true // Won't interrupt existing fiber
  })
})
```

#### Advanced Interruption Control

```typescript
const interruptionDemo = Effect.gen(function* () {
  const map = yield* FiberMap.make<string>()
  
  // Control error propagation
  yield* FiberMap.run(map, "critical-task", criticalTask, {
    propagateInterruption: true // Failure propagates to FiberMap
  })
  
  yield* FiberMap.run(map, "background-task", backgroundTask, {
    propagateInterruption: false // Failure stays isolated
  })
  
  // Wait for all fibers or first failure
  yield* FiberMap.join(map)
})
```

### Feature 2: Runtime Integration

Create runtime-backed execution functions for high-performance scenarios.

#### Creating Runtime Functions

```typescript
const runtimeIntegration = Effect.gen(function* () {
  // Create a runtime-backed runner
  const run = yield* FiberMap.makeRuntime<never, string>()
  
  // Use like a regular function (no Effect wrapper needed)
  const fiber1 = run("task-1", Effect.succeed("result-1"))
  const fiber2 = run("task-2", Effect.succeed("result-2"))
  
  // Fibers are automatically managed
  const result1 = yield* Fiber.join(fiber1)
  const result2 = yield* Fiber.join(fiber2)
  
  return [result1, result2]
}).pipe(Effect.scoped)
```

#### Promise-Based Runtime

```typescript
const promiseRuntime = Effect.gen(function* () {
  const runPromise = yield* FiberMap.makeRuntimePromise<never, string>()
  
  // Returns promises instead of fibers
  const promise1 = runPromise("async-task", Effect.succeed("async-result"))
  const promise2 = runPromise("another-task", Effect.succeed("another-result"))
  
  // Convert back to Effect if needed
  const result1 = yield* Effect.promise(() => promise1)
  const result2 = yield* Effect.promise(() => promise2)
  
  return [result1, result2]
}).pipe(Effect.scoped)
```

### Feature 3: Advanced Querying and Monitoring

#### Fiber Inspection and Management

```typescript
const monitoringExample = Effect.gen(function* () {
  const map = yield* FiberMap.make<string>()
  
  // Add some fibers
  yield* FiberMap.run(map, "task-1", Effect.sleep("10 seconds"))
  yield* FiberMap.run(map, "task-2", Effect.sleep("5 seconds"))
  
  // Check if specific fiber exists
  const hasTask1 = yield* FiberMap.has(map, "task-1")
  yield* Effect.log(`Task 1 exists: ${hasTask1}`)
  
  // Get specific fiber for inspection
  const task1Fiber = yield* FiberMap.get(map, "task-1").pipe(
    Effect.catchTag("NoSuchElement", () => Effect.succeed(null))
  )
  
  if (task1Fiber) {
    const fiberId = task1Fiber.id()
    yield* Effect.log(`Task 1 fiber ID: ${fiberId}`)
  }
  
  // Monitor map size
  const initialSize = yield* FiberMap.size(map)
  yield* Effect.log(`Initial size: ${initialSize}`)
  
  // Wait for fibers to complete naturally
  yield* FiberMap.awaitEmpty(map)
  
  const finalSize = yield* FiberMap.size(map)
  yield* Effect.log(`Final size: ${finalSize}`)
}).pipe(Effect.scoped)
```

#### Iteration and Batch Operations

```typescript
const batchOperations = Effect.gen(function* () {
  const map = yield* FiberMap.make<string>()
  
  // Add multiple fibers
  const tasks = ["task-1", "task-2", "task-3", "task-4"]
  yield* Effect.forEach(tasks, (taskId) =>
    FiberMap.run(map, taskId, Effect.sleep(Duration.seconds(Math.random() * 10)))
  )
  
  // Iterate over all fibers
  const fiberInfo: Array<{ key: string, fiberId: string }> = []
  for (const [key, fiber] of map) {
    fiberInfo.push({
      key,
      fiberId: fiber.id().toString()
    })
  }
  
  yield* Effect.log(`Active fibers: ${JSON.stringify(fiberInfo)}`)
  
  // Clear all fibers at once
  yield* FiberMap.clear(map)
  
  const clearedSize = yield* FiberMap.size(map)
  yield* Effect.log(`Size after clear: ${clearedSize}`)
}).pipe(Effect.scoped)
```

## Practical Patterns & Best Practices

### Pattern 1: Supervised Fiber Pool

Create a robust fiber pool with supervision and automatic restart capabilities:

```typescript
const createSupervisedPool = <K, A, E>(
  supervisor: (key: K, error: E) => Effect.Effect<boolean> // Return true to restart
) => {
  return Effect.gen(function* () {
    const pool = yield* FiberMap.make<K, A, E>()
    const restartCounts = yield* Ref.make<Map<K, number>>(new Map())
    
    const addSupervised = (
      key: K,
      effect: Effect.Effect<A, E>,
      maxRestarts: number = 3
    ) => {
      const supervisedEffect = effect.pipe(
        Effect.catchAll((error) =>
          Effect.gen(function* () {
            const shouldRestart = yield* supervisor(key, error)
            
            if (shouldRestart) {
              const currentCount = yield* Ref.get(restartCounts).pipe(
                Effect.map(map => map.get(key) ?? 0)
              )
              
              if (currentCount < maxRestarts) {
                yield* Ref.update(restartCounts, map => {
                  const newMap = new Map(map)
                  newMap.set(key, currentCount + 1)
                  return newMap
                })
                
                yield* Effect.log(`Restarting ${key} (attempt ${currentCount + 1})`)
                yield* Effect.sleep("1 second") // Brief delay before restart
                return yield* addSupervised(key, effect, maxRestarts)
              }
            }
            
            return Effect.fail(error)
          }).pipe(Effect.flatten)
        )
      )
      
      return FiberMap.run(pool, key, supervisedEffect)
    }
    
    return { pool, addSupervised }
  })
}

// Usage
const supervisedServices = Effect.gen(function* () {
  const supervisor = (key: string, error: unknown) =>
    Effect.gen(function* () {
      yield* Effect.log(`Service ${key} failed: ${error}`)
      return Effect.succeed(true) // Always restart
    })
  
  const { addSupervised } = yield* createSupervisedPool(supervisor)
  
  yield* addSupervised("database-connection", maintainDatabaseConnection)
  yield* addSupervised("message-processor", processMessages)
  yield* addSupervised("health-monitor", monitorHealth)
  
}).pipe(Effect.scoped)
```

### Pattern 2: Rate-Limited Fiber Execution

Control the rate of fiber creation and execution:

```typescript
const createRateLimitedMap = <K>(
  maxConcurrent: number,
  rateLimitPerSecond: number
) => {
  return Effect.gen(function* () {
    const map = yield* FiberMap.make<K>()
    const semaphore = yield* Effect.makeSemaphore(maxConcurrent)
    const rateLimiter = yield* Effect.makeRateLimiter({
      requests: rateLimitPerSecond,
      duration: Duration.seconds(1)
    })
    
    const runRateLimited = <A, E, R>(
      key: K,
      effect: Effect.Effect<A, E, R>
    ): Effect.Effect<Fiber.RuntimeFiber<A, E>, never, R> => {
      const limitedEffect = Effect.gen(function* () {
        yield* rateLimiter
        yield* semaphore.withPermits(1)(effect)
      })
      
      return FiberMap.run(map, key, limitedEffect)
    }
    
    return { map, runRateLimited }
  })
}

// Usage
const apiRequestManager = Effect.gen(function* () {
  const { runRateLimited } = yield* createRateLimitedMap<string>(
    10, // Max 10 concurrent requests
    100 // Max 100 requests per second
  )
  
  // These will be rate-limited automatically
  yield* runRateLimited("fetch-user-1", fetchUser("user1"))
  yield* runRateLimited("fetch-user-2", fetchUser("user2"))
  yield* runRateLimited("fetch-user-3", fetchUser("user3"))
  
}).pipe(Effect.scoped)
```

### Pattern 3: Hierarchical Fiber Management

Organize fibers in a hierarchical structure with cascading cleanup:

```typescript
interface FiberHierarchy<K> {
  readonly parent: FiberMap<K>
  readonly children: Map<K, FiberMap<string>>
}

const createHierarchicalManager = <K>() => {
  return Effect.gen(function* () {
    const parent = yield* FiberMap.make<K>()
    const children = yield* Ref.make<Map<K, FiberMap<string>>>(new Map())
    
    const addParent = (key: K, effect: Effect.Effect<void>) =>
      FiberMap.run(parent, key, effect)
    
    const addChild = (parentKey: K, childKey: string, effect: Effect.Effect<void>) =>
      Effect.gen(function* () {
        const childrenMap = yield* Ref.get(children)
        let childMap = childrenMap.get(parentKey)
        
        if (!childMap) {
          childMap = yield* FiberMap.make<string>()
          yield* Ref.update(children, map => {
            const newMap = new Map(map)
            newMap.set(parentKey, childMap!)
            return newMap
          })
        }
        
        yield* FiberMap.run(childMap, childKey, effect)
      })
    
    const removeParent = (key: K) =>
      Effect.gen(function* () {
        // First clean up all children
        const childrenMap = yield* Ref.get(children)
        const childMap = childrenMap.get(key)
        
        if (childMap) {
          yield* FiberMap.clear(childMap)
          yield* Ref.update(children, map => {
            const newMap = new Map(map)
            newMap.delete(key)
            return newMap
          })
        }
        
        // Then remove parent
        yield* FiberMap.remove(parent, key)
      })
    
    return {
      addParent,
      addChild,
      removeParent,
      parent,
      children: Ref.get(children)
    }
  })
}
```

## Integration Examples

### Integration with HTTP Server

Build a robust HTTP server with per-request fiber management:

```typescript
import { Effect, FiberMap, HttpServer } from "@effect/platform"

const createHttpServerWithFiberManagement = () => {
  return Effect.gen(function* () {
    const requestFibers = yield* FiberMap.make<string>()
    
    const app = HttpServer.router.empty.pipe(
      HttpServer.router.post("/api/process", 
        Effect.gen(function* () {
          const request = yield* HttpServer.request.HttpServerRequest
          const requestId = yield* Effect.sync(() => crypto.randomUUID())
          
          // Process request in managed fiber
          const processingFiber = yield* FiberMap.run(
            requestFibers,
            requestId,
            processLongRunningRequest(request)
          )
          
          // Return request ID for tracking
          return HttpServer.response.json({ requestId })
        })
      ),
      HttpServer.router.get("/api/status/:requestId",
        Effect.gen(function* () {
          const requestId = yield* HttpServer.params.fromPath({ requestId: HttpServer.params.string })
          const fiber = yield* FiberMap.unsafeGet(requestFibers, requestId.requestId)
          
          if (fiber._tag === "Some") {
            const poll = fiber.value.unsafePoll()
            return HttpServer.response.json({
              status: poll ? "completed" : "processing",
              result: poll ? poll : null
            })
          } else {
            return HttpServer.response.json({ status: "not_found" })
          }
        })
      ),
      HttpServer.router.delete("/api/cancel/:requestId",
        Effect.gen(function* () {
          const requestId = yield* HttpServer.params.fromPath({ requestId: HttpServer.params.string })
          yield* FiberMap.remove(requestFibers, requestId.requestId)
          return HttpServer.response.json({ status: "cancelled" })
        })
      )
    )
    
    return HttpServer.serve(app, { port: 3000 })
  }).pipe(Effect.scoped)
}
```

### Integration with Testing Framework

Use FiberMap for managing test fixtures and cleanup:

```typescript
import { Effect, FiberMap, TestServices } from "effect"

const createTestHarness = () => {
  return Effect.gen(function* () {
    const testFibers = yield* FiberMap.make<string>()
    const testResources = yield* Ref.make<Map<string, unknown>>(new Map())
    
    const startTestResource = <A>(
      name: string,
      resource: Effect.Effect<A>,
      cleanup?: (resource: A) => Effect.Effect<void>
    ) =>
      Effect.gen(function* () {
        const resourceFiber = yield* FiberMap.run(
          testFibers,
          name,
          Effect.gen(function* () {
            const res = yield* resource
            yield* Ref.update(testResources, map => {
              const newMap = new Map(map)
              newMap.set(name, res)
              return newMap
            })
            
            // Keep resource alive
            yield* Effect.never
          }).pipe(
            Effect.onInterrupt(() =>
              cleanup ? 
                Effect.gen(function* () {
                  const resourcesMap = yield* Ref.get(testResources)
                  const res = resourcesMap.get(name) as A
                  if (res) {
                    yield* cleanup(res)
                  }
                }) :
                Effect.void
            )
          )
        )
        
        // Wait for resource to be ready
        yield* Effect.sleep("10 millis")
        
        return resourceFiber
      })
    
    const getTestResource = <A>(name: string): Effect.Effect<A, Error> =>
      Effect.gen(function* () {
        const resourcesMap = yield* Ref.get(testResources)
        const resource = resourcesMap.get(name)
        
        if (!resource) {
          return yield* Effect.fail(new Error(`Test resource '${name}' not found`))
        }
        
        return resource as A
      })
    
    const cleanupAllResources = () =>
      FiberMap.clear(testFibers)
    
    return {
      startTestResource,
      getTestResource,
      cleanupAllResources
    }
  })
}

// Usage in tests
const testSuite = Effect.gen(function* () {
  const harness = yield* createTestHarness()
  
  // Start test database
  yield* harness.startTestResource(
    "database",
    createTestDatabase(),
    (db) => db.close()
  )
  
  // Start test server
  yield* harness.startTestResource(
    "server",
    createTestServer(),
    (server) => server.shutdown()
  )
  
  // Run tests
  yield* Effect.gen(function* () {
    const db = yield* harness.getTestResource<Database>("database")
    const server = yield* harness.getTestResource<Server>("server")
    
    // Your test logic here
    yield* runTestsWithResources(db, server)
  })
  
  // Cleanup happens automatically when scope closes
}).pipe(
  Effect.scoped,
  Effect.provide(TestServices.TestServices)
)
```

## Conclusion

FiberMap provides powerful concurrent fiber management, automatic resource cleanup, and sophisticated lifecycle control for building robust Effect applications.

Key benefits:
- **Automatic Cleanup**: Fibers are automatically removed when completed and interrupted on scope closure
- **Key-Based Organization**: Organize concurrent operations by meaningful keys with automatic replacement
- **Error Isolation**: Control error propagation between fibers with fine-grained options

FiberMap is ideal for managing WebSocket connections, background tasks, HTTP request deduplication, worker pools, and any scenario requiring keyed concurrent fiber management with automatic lifecycle handling.