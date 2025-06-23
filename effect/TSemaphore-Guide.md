# TSemaphore: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem TSemaphore Solves

Resource limitation and coordination in concurrent applications often requires careful synchronization. Traditional approaches can lead to race conditions, deadlocks, or inefficient resource utilization.

```typescript
// Traditional approach - using locks or promises with shared state
let activeConnections = 0
const maxConnections = 3

async function processRequest(request: Request) {
  // Race condition: multiple fibers could check and increment simultaneously
  if (activeConnections >= maxConnections) {
    throw new Error("Too many active connections")
  }
  activeConnections++
  
  try {
    return await handleRequest(request)
  } finally {
    // Memory leak risk: decrement might not execute on error
    activeConnections--
  }
}
```

This approach leads to:
- **Race Conditions** - Multiple threads can check and modify shared state simultaneously
- **Resource Leaks** - Failed cleanup when exceptions occur
- **Deadlock Potential** - Complex locking scenarios can cause system freezes
- **Poor Composability** - Hard to combine with other synchronization primitives

### The TSemaphore Solution

TSemaphore provides transactional memory-based semaphores that guarantee atomic operations and automatic cleanup within STM transactions.

```typescript
import { Effect, STM, TSemaphore } from "effect"

const processRequest = (request: Request, semaphore: TSemaphore.TSemaphore) =>
  Effect.gen(function* () {
    // Atomic acquisition with automatic cleanup
    return yield* TSemaphore.withPermit(semaphore)(
      handleRequest(request)
    )
  })
```

### Key Concepts

**TSemaphore**: A transactional semaphore that manages permits within STM transactions, ensuring atomic acquire/release operations.

**Permits**: Virtual tokens that control access to resources - when permits are unavailable, operations automatically retry.

**STM Integration**: All operations are composable within Software Transactional Memory, providing atomicity and consistency guarantees.

## Basic Usage Patterns

### Pattern 1: Creating a TSemaphore

```typescript
import { Effect, STM, TSemaphore } from "effect"

// Create a semaphore with 3 permits inside an STM transaction
const makeSemaphore = STM.gen(function* () {
  return yield* TSemaphore.make(3)
})

// Use in an Effect
const program = Effect.gen(function* () {
  const semaphore = yield* STM.commit(makeSemaphore)
  return semaphore
})
```

### Pattern 2: Basic Resource Protection

```typescript
import { Effect, STM, TSemaphore } from "effect"

const protectedOperation = (semaphore: TSemaphore.TSemaphore) =>
  Effect.gen(function* () {
    // Acquire permit, run operation, automatically release
    return yield* TSemaphore.withPermit(semaphore)(
      Effect.gen(function* () {
        yield* Effect.log("Performing protected operation")
        yield* Effect.sleep("1 seconds")
        return "operation complete"
      })
    )
  })
```

### Pattern 3: Manual Permit Management

```typescript
import { Effect, STM, TSemaphore } from "effect"

const manualSemaphoreUsage = (semaphore: TSemaphore.TSemaphore) =>
  Effect.gen(function* () {
    // Manual acquire within STM
    yield* STM.commit(TSemaphore.acquire(semaphore))
    
    try {
      yield* Effect.log("Critical section")
      return "result"
    } finally {
      // Manual release within STM
      yield* STM.commit(TSemaphore.release(semaphore))
    }
  })
```

## Real-World Examples

### Example 1: Database Connection Pool Management

Managing a limited pool of database connections to prevent overwhelming the database server.

```typescript
import { Effect, STM, TSemaphore, Layer, Context } from "effect"

// Database connection interface
interface DatabaseConnection {
  readonly id: string
  query: (sql: string) => Effect.Effect<unknown[]>
  close: () => Effect.Effect<void>
}

// Connection pool service
class ConnectionPool extends Context.Tag("ConnectionPool")<
  ConnectionPool,
  {
    readonly acquire: Effect.Effect<DatabaseConnection>
    readonly release: (conn: DatabaseConnection) => Effect.Effect<void>
    readonly withConnection: <A, E, R>(
      f: (conn: DatabaseConnection) => Effect.Effect<A, E, R>
    ) => Effect.Effect<A, E, R>
  }
>() {}

// Implementation with TSemaphore
const makeConnectionPool = (maxConnections: number) =>
  Effect.gen(function* () {
    const semaphore = yield* STM.commit(TSemaphore.make(maxConnections))
    const connections = new Set<DatabaseConnection>()
    
    const acquire = Effect.gen(function* () {
      // Wait for available permit
      yield* STM.commit(TSemaphore.acquire(semaphore))
      
      // Create or reuse connection
      const connection: DatabaseConnection = {
        id: `conn-${Date.now()}`,
        query: (sql: string) => Effect.succeed([]),
        close: () => Effect.void
      }
      
      connections.add(connection)
      return connection
    })
    
    const release = (conn: DatabaseConnection) =>
      Effect.gen(function* () {
        connections.delete(conn)
        yield* conn.close()
        yield* STM.commit(TSemaphore.release(semaphore))
      })
    
    const withConnection = <A, E, R>(
      f: (conn: DatabaseConnection) => Effect.Effect<A, E, R>
    ): Effect.Effect<A, E, R> =>
      Effect.gen(function* () {
        const conn = yield* acquire
        try {
          return yield* f(conn)
        } finally {
          yield* release(conn)
        }
      })
    
    return { acquire, release, withConnection }
  })

// Usage in application
const userRepository = Effect.gen(function* () {
  const pool = yield* ConnectionPool
  
  const findUser = (id: string) =>
    pool.withConnection((conn) =>
      Effect.gen(function* () {
        yield* Effect.log(`Querying user ${id}`)
        const results = yield* conn.query(`SELECT * FROM users WHERE id = ${id}`)
        return results[0] as { id: string; name: string }
      })
    )
  
  return { findUser }
})

// Layer providing the connection pool
const ConnectionPoolLive = Layer.effect(
  ConnectionPool,
  makeConnectionPool(5)
)
```

### Example 2: Rate Limiting API Requests

Implementing a rate limiter for external API calls using TSemaphore with time-based permit replenishment.

```typescript
import { Effect, STM, TSemaphore, Schedule, Fiber } from "effect"

interface RateLimiter {
  readonly execute: <A, E, R>(
    operation: Effect.Effect<A, E, R>
  ) => Effect.Effect<A, E, R>
  readonly availablePermits: Effect.Effect<number>
}

const makeRateLimiter = (
  maxRequests: number,
  windowMs: number
): Effect.Effect<RateLimiter> =>
  Effect.gen(function* () {
    const semaphore = yield* STM.commit(TSemaphore.make(maxRequests))
    
    // Background fiber to replenish permits
    const replenishFiber = yield* Effect.fork(
      Effect.gen(function* () {
        while (true) {
          yield* Effect.sleep(`${windowMs} millis`)
          
          // Replenish permits up to maximum
          yield* STM.commit(
            STM.gen(function* () {
              const available = yield* TSemaphore.available(semaphore)
              const toAdd = Math.max(0, maxRequests - available)
              if (toAdd > 0) {
                yield* TSemaphore.releaseN(semaphore, toAdd)
              }
            })
          )
        }
      })
    )
    
    const execute = <A, E, R>(
      operation: Effect.Effect<A, E, R>
    ): Effect.Effect<A, E, R> =>
      TSemaphore.withPermit(semaphore)(operation)
    
    const availablePermits = STM.commit(TSemaphore.available(semaphore))
    
    // Cleanup on scope close
    yield* Effect.addFinalizer(() => Fiber.interrupt(replenishFiber))
    
    return { execute, availablePermits }
  })

// External API service with rate limiting
interface ExternalApiService {
  readonly fetchUserData: (userId: string) => Effect.Effect<UserData>
  readonly fetchOrderHistory: (userId: string) => Effect.Effect<Order[]>
}

interface UserData {
  readonly id: string
  readonly name: string
  readonly email: string
}

interface Order {
  readonly id: string
  readonly amount: number
  readonly date: string
}

const makeExternalApiService = Effect.gen(function* () {
  // Rate limit: 10 requests per minute
  const rateLimiter = yield* makeRateLimiter(10, 60_000)
  
  const fetchUserData = (userId: string): Effect.Effect<UserData> =>
    rateLimiter.execute(
      Effect.gen(function* () {
        yield* Effect.log(`Fetching user data for ${userId}`)
        yield* Effect.sleep("500 millis") // Simulate API call
        
        return {
          id: userId,
          name: `User ${userId}`,
          email: `user${userId}@example.com`
        }
      })
    )
  
  const fetchOrderHistory = (userId: string): Effect.Effect<Order[]> =>
    rateLimiter.execute(
      Effect.gen(function* () {
        yield* Effect.log(`Fetching order history for ${userId}`)
        yield* Effect.sleep("800 millis") // Simulate API call
        
        return [
          { id: "order-1", amount: 99.99, date: "2024-01-01" },
          { id: "order-2", amount: 149.99, date: "2024-01-15" }
        ]
      })
    )
  
  return { fetchUserData, fetchOrderHistory }
})
```

### Example 3: Worker Pool Coordination

Coordinating a pool of worker fibers processing tasks from a queue, ensuring optimal resource utilization.

```typescript
import { Effect, STM, TSemaphore, TQueue, Fiber, Array as Arr } from "effect"

interface Task {
  readonly id: string
  readonly payload: unknown
  readonly priority: number
}

interface WorkerPool {
  readonly submitTask: (task: Task) => Effect.Effect<void>
  readonly shutdown: Effect.Effect<void>
  readonly getStats: Effect.Effect<{
    activeWorkers: number
    queuedTasks: number
    processedTasks: number
  }>
}

const makeWorkerPool = (
  workerCount: number,
  processTask: (task: Task) => Effect.Effect<void>
): Effect.Effect<WorkerPool> =>
  Effect.gen(function* () {
    const semaphore = yield* STM.commit(TSemaphore.make(workerCount))
    const taskQueue = yield* STM.commit(TQueue.unbounded<Task>())
    const processedCount = yield* STM.commit(STM.make(0))
    
    // Worker function
    const worker = Effect.gen(function* () {
      while (true) {
        // Get next task from queue
        const task = yield* STM.commit(TQueue.take(taskQueue))
        
        // Process task with semaphore protection
        yield* TSemaphore.withPermit(semaphore)(
          Effect.gen(function* () {
            yield* Effect.log(`Worker processing task ${task.id}`)
            yield* processTask(task)
            yield* STM.commit(STM.update(processedCount, n => n + 1))
            yield* Effect.log(`Task ${task.id} completed`)
          })
        )
      }
    })
    
    // Start worker fibers
    const workerFibers = yield* Effect.forEach(
      Arr.range(1, workerCount),
      () => Effect.fork(worker),
      { concurrency: "unbounded" }
    )
    
    const submitTask = (task: Task) =>
      STM.commit(TQueue.offer(taskQueue, task))
    
    const shutdown = Effect.gen(function* () {
      yield* Effect.log("Shutting down worker pool...")
      yield* Fiber.interruptAll(workerFibers)
      yield* Effect.log("Worker pool shutdown complete")
    })
    
    const getStats = Effect.gen(function* () {
      const queueSize = yield* STM.commit(TQueue.size(taskQueue))
      const processed = yield* STM.commit(STM.get(processedCount))
      const available = yield* STM.commit(TSemaphore.available(semaphore))
      
      return {
        activeWorkers: workerCount - available,
        queuedTasks: queueSize,
        processedTasks: processed
      }
    })
    
    return { submitTask, shutdown, getStats }
  })

// Usage example
const workerPoolExample = Effect.gen(function* () {
  const processTask = (task: Task) =>
    Effect.gen(function* () {
      // Simulate work with variable duration based on priority
      const duration = task.priority * 100
      yield* Effect.sleep(`${duration} millis`)
      
      if (Math.random() < 0.1) {
        yield* Effect.fail(new Error(`Task ${task.id} failed`))
      }
    }).pipe(
      Effect.catchAll((error) =>
        Effect.log(`Task failed: ${error.message}`)
      )
    )
  
  const pool = yield* makeWorkerPool(3, processTask)
  
  // Submit multiple tasks
  yield* Effect.forEach(
    Arr.range(1, 10),
    (i) => pool.submitTask({
      id: `task-${i}`,
      payload: { data: `Task ${i} data` },
      priority: Math.floor(Math.random() * 5) + 1
    }),
    { concurrency: "unbounded" }
  )
  
  // Monitor stats
  yield* Effect.fork(
    Effect.repeat(
      Effect.gen(function* () {
        const stats = yield* pool.getStats
        yield* Effect.log(
          `Stats - Active: ${stats.activeWorkers}, ` +
          `Queued: ${stats.queuedTasks}, ` +
          `Processed: ${stats.processedTasks}`
        )
      }),
      Schedule.fixed("2 seconds")
    )
  )
  
  // Let it run for a while
  yield* Effect.sleep("15 seconds")
  
  yield* pool.shutdown
})
```

## Advanced Features Deep Dive

### Feature 1: Multi-Permit Operations

TSemaphore supports acquiring multiple permits atomically, useful for operations requiring varying amounts of resources.

#### Basic Multi-Permit Usage

```typescript
import { Effect, STM, TSemaphore } from "effect"

const multiPermitExample = Effect.gen(function* () {
  const semaphore = yield* STM.commit(TSemaphore.make(10))
  
  // Small operation needs 2 permits
  const smallOperation = TSemaphore.withPermits(semaphore, 2)(
    Effect.gen(function* () {
      yield* Effect.log("Running small operation")
      yield* Effect.sleep("1 seconds")
      return "small result"
    })
  )
  
  // Large operation needs 5 permits
  const largeOperation = TSemaphore.withPermits(semaphore, 5)(
    Effect.gen(function* () {
      yield* Effect.log("Running large operation")
      yield* Effect.sleep("3 seconds")
      return "large result"
    })
  )
  
  // Run operations concurrently
  const results = yield* Effect.all([
    smallOperation,
    smallOperation,
    largeOperation
  ], { concurrency: "unbounded" })
  
  yield* Effect.log(`Results: ${JSON.stringify(results)}`)
})
```

#### Real-World Multi-Permit Example: Memory Pool Manager

```typescript
interface MemoryPool {
  readonly allocate: (size: number) => Effect.Effect<Buffer>
  readonly deallocate: (buffer: Buffer) => Effect.Effect<void>
  readonly stats: Effect.Effect<{ allocated: number; available: number }>
}

const makeMemoryPool = (totalMemoryMB: number): Effect.Effect<MemoryPool> =>
  Effect.gen(function* () {
    // Each permit represents 1MB of memory
    const semaphore = yield* STM.commit(TSemaphore.make(totalMemoryMB))
    const allocatedBuffers = new WeakMap<Buffer, number>()
    
    const allocate = (sizeMB: number) =>
      Effect.gen(function* () {
        if (sizeMB <= 0) {
          yield* Effect.fail(new Error("Invalid allocation size"))
        }
        
        // Acquire permits equal to required memory
        yield* STM.commit(TSemaphore.acquireN(semaphore, sizeMB))
        
        const buffer = Buffer.alloc(sizeMB * 1024 * 1024)
        allocatedBuffers.set(buffer, sizeMB)
        
        yield* Effect.log(`Allocated ${sizeMB}MB buffer`)
        return buffer
      })
    
    const deallocate = (buffer: Buffer) =>
      Effect.gen(function* () {
        const size = allocatedBuffers.get(buffer)
        if (!size) {
          yield* Effect.fail(new Error("Buffer not found in pool"))
        }
        
        allocatedBuffers.delete(buffer)
        yield* STM.commit(TSemaphore.releaseN(semaphore, size))
        yield* Effect.log(`Deallocated ${size}MB buffer`)
      })
    
    const stats = Effect.gen(function* () {
      const available = yield* STM.commit(TSemaphore.available(semaphore))
      return {
        allocated: totalMemoryMB - available,
        available
      }
    })
    
    return { allocate, deallocate, stats }
  })
```

### Feature 2: Scoped Permit Management

TSemaphore integrates with Effect's resource management system for automatic cleanup.

#### Basic Scoped Usage

```typescript
import { Effect, STM, TSemaphore, Scope } from "effect"

const scopedSemaphoreExample = Effect.gen(function* () {
  const semaphore = yield* STM.commit(TSemaphore.make(3))
  
  // Automatic permit acquisition and release within scope
  yield* Effect.scoped(
    Effect.gen(function* () {
      // Acquire permit for the scope duration
      yield* TSemaphore.withPermitScoped(semaphore)
      
      yield* Effect.log("Inside scoped resource")
      yield* Effect.sleep("2 seconds")
      
      // Permit automatically released when scope closes
    })
  )
  
  yield* Effect.log("Scope closed, permit released")
})
```

#### Advanced Scoped: Resource Lifecycle Manager

```typescript
interface ManagedResource {
  readonly id: string
  readonly cleanup: () => Effect.Effect<void>
  readonly use: <A>(f: (resource: ManagedResource) => Effect.Effect<A>) => Effect.Effect<A>
}

const makeManagedResource = (id: string): Effect.Effect<ManagedResource> =>
  Effect.gen(function* () {
    yield* Effect.log(`Creating resource ${id}`)
    
    const cleanup = () =>
      Effect.gen(function* () {
        yield* Effect.log(`Cleaning up resource ${id}`)
        yield* Effect.sleep("100 millis")
      })
    
    const use = <A>(f: (resource: ManagedResource) => Effect.Effect<A>) =>
      f({ id, cleanup, use })
    
    return { id, cleanup, use }
  })

const resourceManager = Effect.gen(function* () {
  // Limit concurrent resources to 2
  const semaphore = yield* STM.commit(TSemaphore.make(2))
  
  const withManagedResource = <A>(
    resourceId: string,
    f: (resource: ManagedResource) => Effect.Effect<A>
  ): Effect.Effect<A> =>
    Effect.scoped(
      Effect.gen(function* () {
        // Acquire permit and resource within same scope
        yield* TSemaphore.withPermitScoped(semaphore)
        
        const resource = yield* Effect.acquireRelease(
          makeManagedResource(resourceId),
          (resource) => resource.cleanup()
        )
        
        return yield* f(resource)
      })
    )
  
  // Use multiple resources concurrently
  const tasks = Arr.range(1, 5).map((i) =>
    withManagedResource(`resource-${i}`, (resource) =>
      Effect.gen(function* () {
        yield* Effect.log(`Using ${resource.id}`)
        yield* Effect.sleep("3 seconds")
        return `Result from ${resource.id}`
      })
    )
  )
  
  const results = yield* Effect.all(tasks, { concurrency: "unbounded" })
  yield* Effect.log(`All results: ${JSON.stringify(results)}`)
})
```

### Feature 3: STM Composition and Coordination

TSemaphore operations compose naturally with other STM primitives for complex coordination patterns.

#### Coordinated Resource Access

```typescript
import { Effect, STM, TSemaphore, TRef, TQueue } from "effect"

interface ResourceCoordinator {
  readonly requestResource: (
    priority: number,
    requiredPermits: number
  ) => Effect.Effect<string>
  readonly getQueueStatus: Effect.Effect<{
    waiting: number
    available: number
  }>
}

const makeResourceCoordinator = (
  totalPermits: number
): Effect.Effect<ResourceCoordinator> =>
  Effect.gen(function* () {
    const semaphore = yield* STM.commit(TSemaphore.make(totalPermits))
    const requestCounter = yield* STM.commit(TRef.make(0))
    const priorityQueue = yield* STM.commit(TQueue.unbounded<{
      id: string
      priority: number
      permits: number
      deferred: Effect.Deferred<string>
    }>())
    
    // Background processor that handles requests by priority
    const processor = Effect.gen(function* () {
      while (true) {
        const request = yield* STM.commit(TQueue.take(priorityQueue))
        
        // Try to acquire permits - will retry if not available
        yield* STM.commit(TSemaphore.acquireN(semaphore, request.permits))
        
        // Grant the resource
        yield* Effect.Deferred.succeed(
          request.deferred,
          `Resource-${request.id} (${request.permits} permits)`
        )
        
        // Simulate resource usage time
        yield* Effect.sleep("2 seconds")
        
        // Release permits
        yield* STM.commit(TSemaphore.releaseN(semaphore, request.permits))
      }
    })
    
    yield* Effect.fork(processor)
    
    const requestResource = (priority: number, requiredPermits: number) =>
      Effect.gen(function* () {
        const id = yield* STM.commit(
          STM.gen(function* () {
            const counter = yield* TRef.get(requestCounter)
            yield* TRef.set(requestCounter, counter + 1)
            return counter.toString()
          })
        )
        
        const deferred = yield* Effect.Deferred.make<string>()
        
        // Add request to priority queue
        yield* STM.commit(
          TQueue.offer(priorityQueue, {
            id,
            priority,
            permits: requiredPermits,
            deferred
          })
        )
        
        // Wait for resource allocation
        return yield* Effect.Deferred.await(deferred)
      })
    
    const getQueueStatus = Effect.gen(function* () {
      const waiting = yield* STM.commit(TQueue.size(priorityQueue))
      const available = yield* STM.commit(TSemaphore.available(semaphore))
      return { waiting, available }
    })
    
    return { requestResource, getQueueStatus }
  })
```

## Practical Patterns & Best Practices

### Pattern 1: Graceful Degradation with Circuit Breaker

```typescript
interface CircuitBreakerSemaphore {
  readonly execute: <A, E, R>(
    operation: Effect.Effect<A, E, R>
  ) => Effect.Effect<A, E | Error, R>
  readonly getState: Effect.Effect<"CLOSED" | "OPEN" | "HALF_OPEN">
}

const makeCircuitBreakerSemaphore = (
  permits: number,
  failureThreshold: number,
  timeoutMs: number
): Effect.Effect<CircuitBreakerSemaphore> =>
  Effect.gen(function* () {
    const semaphore = yield* STM.commit(TSemaphore.make(permits))
    const failureCount = yield* STM.commit(TRef.make(0))
    const lastFailureTime = yield* STM.commit(TRef.make<number | null>(null))
    const state = yield* STM.commit(TRef.make<"CLOSED" | "OPEN" | "HALF_OPEN">("CLOSED"))
    
    const updateState = STM.gen(function* () {
      const failures = yield* TRef.get(failureCount)
      const lastFailure = yield* TRef.get(lastFailureTime)
      const currentTime = Date.now()
      
      if (failures >= failureThreshold) {
        if (lastFailure && (currentTime - lastFailure) > timeoutMs) {
          yield* TRef.set(state, "HALF_OPEN")
        } else {
          yield* TRef.set(state, "OPEN")
        }
      } else {
        yield* TRef.set(state, "CLOSED")
      }
    })
    
    const execute = <A, E, R>(
      operation: Effect.Effect<A, E, R>
    ): Effect.Effect<A, E | Error, R> =>
      Effect.gen(function* () {
        yield* STM.commit(updateState)
        const currentState = yield* STM.commit(TRef.get(state))
        
        if (currentState === "OPEN") {
          yield* Effect.fail(new Error("Circuit breaker is OPEN"))
        }
        
        return yield* TSemaphore.withPermit(semaphore)(operation).pipe(
          Effect.tapDefect(() =>
            STM.commit(
              STM.gen(function* () {
                yield* TRef.update(failureCount, n => n + 1)
                yield* TRef.set(lastFailureTime, Date.now())
              })
            )
          ),
          Effect.tap(() =>
            STM.commit(TRef.set(failureCount, 0))
          )
        )
      })
    
    const getState = STM.commit(
      STM.gen(function* () {
        yield* updateState
        return yield* TRef.get(state)
      })
    )
    
    return { execute, getState }
  })
```

### Pattern 2: Adaptive Semaphore with Load Monitoring

```typescript
interface AdaptiveSemaphore {
  readonly withPermit: <A, E, R>(
    operation: Effect.Effect<A, E, R>
  ) => Effect.Effect<A, E, R>
  readonly getMetrics: Effect.Effect<{
    currentPermits: number
    averageWaitTime: number
    throughput: number
  }>
}

const makeAdaptiveSemaphore = (
  initialPermits: number,
  minPermits: number,
  maxPermits: number
): Effect.Effect<AdaptiveSemaphore> =>
  Effect.gen(function* () {
    const semaphore = yield* STM.commit(TSemaphore.make(initialPermits))
    const waitTimes = yield* STM.commit(TRef.make<number[]>([]))
    const completionTimes = yield* STM.commit(TRef.make<number[]>([]))
    
    // Background adjustment based on metrics
    const adjuster = Effect.gen(function* () {
      while (true) {
        yield* Effect.sleep("5 seconds")
        
        yield* STM.commit(
          STM.gen(function* () {
            const waits = yield* TRef.get(waitTimes)
            const completions = yield* TRef.get(completionTimes)
            
            if (waits.length > 10) {
              const avgWait = waits.reduce((a, b) => a + b, 0) / waits.length
              const current = yield* TSemaphore.available(semaphore)
              
              // Increase permits if average wait time is high
              if (avgWait > 1000 && current < maxPermits) {
                yield* TSemaphore.release(semaphore)
              }
              // Decrease permits if system is underutilized
              else if (avgWait < 100 && current > minPermits) {
                yield* TSemaphore.acquire(semaphore)
              }
              
              // Clear old metrics
              yield* TRef.set(waitTimes, [])
              yield* TRef.set(completionTimes, [])
            }
          })
        )
      }
    })
    
    yield* Effect.fork(adjuster)
    
    const withPermit = <A, E, R>(
      operation: Effect.Effect<A, E, R>
    ): Effect.Effect<A, E, R> =>
      Effect.gen(function* () {
        const startWait = Date.now()
        
        return yield* TSemaphore.withPermit(semaphore)(
          Effect.gen(function* () {
            const waitTime = Date.now() - startWait
            const startExec = Date.now()
            
            const result = yield* operation
            
            const execTime = Date.now() - startExec
            
            // Record metrics
            yield* STM.commit(
              STM.gen(function* () {
                yield* TRef.update(waitTimes, times => [...times, waitTime].slice(-100))
                yield* TRef.update(completionTimes, times => [...times, execTime].slice(-100))
              })
            )
            
            return result
          })
        )
      })
    
    const getMetrics = STM.commit(
      STM.gen(function* () {
        const currentPermits = yield* TSemaphore.available(semaphore)
        const waits = yield* TRef.get(waitTimes)
        const completions = yield* TRef.get(completionTimes)
        
        const averageWaitTime = waits.length > 0 
          ? waits.reduce((a, b) => a + b, 0) / waits.length 
          : 0
        
        const throughput = completions.length > 0
          ? 1000 / (completions.reduce((a, b) => a + b, 0) / completions.length)
          : 0
        
        return { currentPermits, averageWaitTime, throughput }
      })
    )
    
    return { withPermit, getMetrics }
  })
```

## Integration Examples

### Integration with Effect HTTP Server

```typescript
import { Effect, STM, TSemaphore, Layer } from "effect"
import * as Http from "@effect/platform/HttpApi"

// Service for handling HTTP requests with semaphore protection
interface RequestHandler {
  readonly handleRequest: (request: Http.HttpServerRequest) => Effect.Effect<Http.HttpServerResponse>
}

const RequestHandlerLive = Layer.effect(
  RequestHandler,
  Effect.gen(function* () {
    // Limit concurrent request processing
    const semaphore = yield* STM.commit(TSemaphore.make(10))
    
    const handleRequest = (request: Http.HttpServerRequest) =>
      TSemaphore.withPermit(semaphore)(
        Effect.gen(function* () {
          const url = request.url
          yield* Effect.log(`Processing request to ${url}`)
          
          // Simulate request processing
          yield* Effect.sleep("500 millis")
          
          const body = JSON.stringify({
            message: "Request processed successfully",
            timestamp: new Date().toISOString(),
            url
          })
          
          return Http.response.json({ body })
        })
      )
    
    return { handleRequest }
  })
)

// HTTP server setup with semaphore integration
const serverProgram = Effect.gen(function* () {
  const handler = yield* RequestHandler
  
  const api = Http.api.make().pipe(
    Http.api.get("/protected", handler.handleRequest)
  )
  
  yield* Effect.log("Server started with semaphore protection")
  
  // Keep server running
  yield* Effect.never
}).pipe(
  Effect.provide(RequestHandlerLive)
)
```

### Testing Strategies

```typescript
import { Effect, STM, TSemaphore, TestClock, TestContext } from "effect"
import { describe, it, expect } from "vitest"

// Test utilities for TSemaphore
const makeSemaphoreTest = (permits: number) =>
  Effect.gen(function* () {
    const semaphore = yield* STM.commit(TSemaphore.make(permits))
    const operations: Effect.Effect<string>[] = []
    const results: string[] = []
    
    // Helper to create tracked operations
    const makeOperation = (id: string, duration: string) =>
      TSemaphore.withPermit(semaphore)(
        Effect.gen(function* () {
          results.push(`${id}-start`)
          yield* TestClock.adjust(duration)
          results.push(`${id}-end`)
          return id
        })
      )
    
    return { semaphore, makeOperation, results }
  })

// Property-based testing helpers
const semaphoreProperties = {
  // Test that permits are properly acquired and released
  permitBookkeeping: (permits: number, operations: number) =>
    Effect.gen(function* () {
      const { semaphore, makeOperation } = yield* makeSemaphoreTest(permits)
      
      const ops = Array.from({ length: operations }, (_, i) =>
        makeOperation(`op-${i}`, "1 seconds")
      )
      
      // Run operations concurrently
      yield* Effect.all(ops, { concurrency: "unbounded" })
      
      // Verify all permits are released
      const available = yield* STM.commit(TSemaphore.available(semaphore))
      expect(available).toBe(permits)
    }),
  
  // Test concurrency limits are respected
  concurrencyLimit: (permits: number) =>
    Effect.gen(function* () {
      const { semaphore, makeOperation, results } = yield* makeSemaphoreTest(permits)
      
      // Start more operations than permits
      const ops = Array.from({ length: permits + 2 }, (_, i) =>
        makeOperation(`op-${i}`, "2 seconds")
      )
      
      yield* Effect.all(ops, { concurrency: "unbounded" })
      
      // Verify concurrency was limited
      const startEvents = results.filter(r => r.endsWith('-start'))
      const endEvents = results.filter(r => r.endsWith('-end'))
      
      // Should have exactly `permits` operations running initially
      expect(startEvents.slice(0, permits)).toHaveLength(permits)
      expect(endEvents.slice(0, permits)).toHaveLength(permits)
    })
}

// Example test suite
describe("TSemaphore", () => {
  it("should respect permit limits", () =>
    Effect.runPromise(
      semaphoreProperties.permitBookkeeping(3, 5).pipe(
        Effect.provide(TestContext.TestContext)
      )
    )
  )
  
  it("should limit concurrency", () =>
    Effect.runPromise(
      semaphoreProperties.concurrencyLimit(2).pipe(
        Effect.provide(TestContext.TestContext)
      )
    )
  )
})
```

## Conclusion

TSemaphore provides atomic, composable resource management for concurrent Effect applications through Software Transactional Memory. It offers automatic cleanup, seamless STM integration, and flexible permit management for sophisticated coordination patterns.

Key benefits:
- **Atomic Operations**: All permit operations are guaranteed atomic within STM transactions
- **Automatic Cleanup**: Resource management with guaranteed permit release even on failures
- **STM Composability**: Natural composition with other transactional data structures

TSemaphore is ideal for scenarios requiring precise concurrency control, resource pooling, rate limiting, and coordinated access to shared resources in Effect-based applications.