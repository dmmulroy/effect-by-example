# TDeferred: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem TDeferred Solves

Traditional synchronization primitives in concurrent programming often lead to complex coordination patterns:

```typescript
// Traditional approach - complex callback and promise coordination
let resultStorage: { value?: any; error?: any; resolved: boolean } = { resolved: false }
const callbacks: Array<(value: any) => void> = []

// Multiple concurrent operations waiting for a result
const waitForResult = (): Promise<any> => {
  return new Promise((resolve, reject) => {
    if (resultStorage.resolved) {
      if (resultStorage.error) reject(resultStorage.error)
      else resolve(resultStorage.value)
    } else {
      callbacks.push((value) => {
        if (resultStorage.error) reject(resultStorage.error)
        else resolve(value)
      })
    }
  })
}

// Producer sets the result
const setResult = (value: any) => {
  resultStorage = { value, resolved: true }
  callbacks.forEach(cb => cb(value))
}
```

This approach leads to:
- **Race Conditions** - Multiple threads checking and setting state simultaneously
- **Memory Leaks** - Callbacks not properly cleaned up
- **Complex Error Handling** - Managing error propagation across waiters
- **No Composability** - Difficult to combine with other concurrent operations

### The TDeferred Solution

TDeferred provides a transactional one-time variable for safe coordination within STM transactions:

```typescript
import { STM, TDeferred } from "effect"

// Safe coordination with TDeferred
const program = STM.gen(function* () {
  const deferred = yield* TDeferred.make<string, never>()
  
  // Multiple operations can safely await the same result
  const waiter1 = TDeferred.await(deferred)
  const waiter2 = TDeferred.await(deferred)
  
  // Producer sets the value once
  yield* TDeferred.succeed(deferred, "Hello, World!")
  
  // All waiters receive the same value atomically
  const result1 = yield* waiter1
  const result2 = yield* waiter2
  
  return { result1, result2 }
})
```

### Key Concepts

**One-Time Variable**: TDeferred can only be completed once, either with a success value or failure.

**Transactional Coordination**: All operations are atomic within STM transactions, preventing race conditions.

**Composability**: TDeferred operations compose naturally with other STM operations for complex coordination patterns.

## Basic Usage Patterns

### Pattern 1: Creating and Completing TDeferred

```typescript
import { STM, TDeferred } from "effect"

// Create deferred with success type
const createDeferred = STM.gen(function* () {
  return yield* TDeferred.make<string, never>()
})

// Create deferred with error type
const createFallibleDeferred = STM.gen(function* () {
  return yield* TDeferred.make<string, Error>()
})

// Complete with success
const completeWithSuccess = (deferred: TDeferred.TDeferred<string, never>) =>
  TDeferred.succeed(deferred, "Success!")

// Complete with failure
const completeWithFailure = (deferred: TDeferred.TDeferred<string, Error>) =>
  TDeferred.fail(deferred, new Error("Something went wrong"))
```

### Pattern 2: Awaiting and Polling

```typescript
// Await completion (blocks until completed)
const awaitCompletion = (deferred: TDeferred.TDeferred<string, Error>) => STM.gen(function* () {
  const result = yield* TDeferred.await(deferred)
  return result
})

// Poll for completion (non-blocking)
const pollCompletion = (deferred: TDeferred.TDeferred<string, Error>) => STM.gen(function* () {
  const result = yield* TDeferred.poll(deferred)
  
  if (result._tag === "None") {
    return "Not yet completed"
  } else if (result.value._tag === "Left") {
    return `Failed with: ${result.value.left.message}`
  } else {
    return `Completed with: ${result.value.right}`
  }
})
```

### Pattern 3: Multiple Waiters Pattern

```typescript
// Multiple operations waiting for the same result
const multipleWaiters = STM.gen(function* () {
  const deferred = yield* TDeferred.make<number, never>()
  
  // Create multiple waiters
  const waiter1 = TDeferred.await(deferred)
  const waiter2 = TDeferred.await(deferred)
  const waiter3 = TDeferred.await(deferred)
  
  // Complete the deferred
  yield* TDeferred.succeed(deferred, 42)
  
  // All waiters get the same result
  const results = yield* STM.all([waiter1, waiter2, waiter3])
  
  return results // [42, 42, 42]
})
```

## Real-World Examples

### Example 1: Resource Initialization Coordinator

```typescript
import { Effect, STM, TDeferred, Layer, Context } from "effect"

interface Database {
  readonly query: (sql: string) => Effect.Effect<unknown>
  readonly close: () => Effect.Effect<void>
}

interface Cache {
  readonly get: (key: string) => Effect.Effect<unknown>
  readonly set: (key: string, value: unknown) => Effect.Effect<void>
}

class ApplicationServices {
  constructor(
    private dbDeferred: TDeferred.TDeferred<Database, Error>,
    private cacheDeferred: TDeferred.TDeferred<Cache, Error>
  ) {}

  static make = (): Effect.Effect<ApplicationServices> =>
    STM.gen(function* () {
      const dbDeferred = yield* TDeferred.make<Database, Error>()
      const cacheDeferred = yield* TDeferred.make<Cache, Error>()
      return new ApplicationServices(dbDeferred, cacheDeferred)
    }).pipe(STM.commit)

  // Initialize database (can be called from multiple places)
  initializeDatabase = (connectionString: string): Effect.Effect<void> =>
    Effect.gen(function* () {
      // Simulate database initialization
      yield* Effect.sleep("200 millis")
      
      const database: Database = {
        query: (sql: string) => Effect.succeed(`Result for: ${sql}`),
        close: () => Effect.succeed(void 0)
      }
      
      yield* STM.commit(TDeferred.succeed(this.dbDeferred, database))
    }).pipe(
      Effect.catchAll((error) =>
        STM.commit(TDeferred.fail(this.dbDeferred, error as Error))
      )
    )

  // Initialize cache (can be called from multiple places)
  initializeCache = (config: { host: string; port: number }): Effect.Effect<void> =>
    Effect.gen(function* () {
      // Simulate cache initialization
      yield* Effect.sleep("100 millis")
      
      const cache: Cache = {
        get: (key: string) => Effect.succeed(`cached-${key}`),
        set: (key: string, value: unknown) => Effect.succeed(void 0)
      }
      
      yield* STM.commit(TDeferred.succeed(this.cacheDeferred, cache))
    }).pipe(
      Effect.catchAll((error) =>
        STM.commit(TDeferred.fail(this.cacheDeferred, error as Error))
      )
    )

  // Wait for database to be ready
  getDatabase = (): Effect.Effect<Database, Error> =>
    STM.commit(TDeferred.await(this.dbDeferred))

  // Wait for cache to be ready
  getCache = (): Effect.Effect<Cache, Error> =>
    STM.commit(TDeferred.await(this.cacheDeferred))

  // Wait for both services to be ready
  waitForServices = (): Effect.Effect<{ database: Database; cache: Cache }, Error> =>
    STM.gen(function* () {
      const database = yield* TDeferred.await(this.dbDeferred)
      const cache = yield* TDeferred.await(this.cacheDeferred)
      return { database, cache }
    }).pipe(STM.commit)
}

// Usage
const resourceInitializationExample = Effect.gen(function* () {
  const services = yield* ApplicationServices.make()
  
  // Multiple components trying to initialize services concurrently
  const initFiber = yield* Effect.all([
    services.initializeDatabase("postgres://localhost:5432/app"),
    services.initializeCache({ host: "localhost", port: 6379 }),
    services.initializeDatabase("postgres://localhost:5432/app"), // Duplicate call - safe!
  ], { concurrency: "unbounded" }).pipe(Effect.fork)
  
  // Multiple components waiting for services
  const usersFiber = yield* Effect.fork(
    Effect.gen(function* () {
      const db = yield* services.getDatabase()
      return yield* db.query("SELECT * FROM users")
    })
  )
  
  const productsFiber = yield* Effect.fork(
    Effect.gen(function* () {
      const { database, cache } = yield* services.waitForServices()
      const products = yield* database.query("SELECT * FROM products")
      yield* cache.set("products", products)
      return products
    })
  )
  
  // Wait for all operations
  yield* Effect.await(initFiber)
  const usersResult = yield* Effect.await(usersFiber)
  const productsResult = yield* Effect.await(productsFiber)
  
  return { users: usersResult, products: productsResult }
})
```

### Example 2: Task Completion Signaling System

```typescript
import { Effect, STM, TDeferred, TRef, TArray, Fiber } from "effect"

interface Task {
  readonly id: string
  readonly name: string
  readonly dependencies: readonly string[]
  readonly work: () => Effect.Effect<unknown>
}

interface TaskResult {
  readonly taskId: string
  readonly result: unknown
  readonly completedAt: Date
}

class TaskOrchestrator {
  constructor(
    private tasks: Map<string, Task>,
    private completionSignals: Map<string, TDeferred.TDeferred<TaskResult, Error>>,
    private completedTasks: TRef.TRef<Map<string, TaskResult>>
  ) {}

  static make = (tasks: readonly Task[]): Effect.Effect<TaskOrchestrator> =>
    STM.gen(function* () {
      const completedTasks = yield* TRef.make(new Map<string, TaskResult>())
      const completionSignals = new Map<string, TDeferred.TDeferred<TaskResult, Error>>()
      
      // Create completion signals for all tasks
      for (const task of tasks) {
        const signal = yield* TDeferred.make<TaskResult, Error>()
        completionSignals.set(task.id, signal)
      }
      
      const tasksMap = new Map(tasks.map(task => [task.id, task]))
      return new TaskOrchestrator(tasksMap, completionSignals, completedTasks)
    }).pipe(STM.commit)

  // Wait for specific task completion
  waitForTask = (taskId: string): Effect.Effect<TaskResult, Error> => {
    const signal = this.completionSignals.get(taskId)
    if (!signal) {
      return Effect.fail(new Error(`Task ${taskId} not found`))
    }
    return STM.commit(TDeferred.await(signal))
  }

  // Wait for multiple tasks
  waitForTasks = (taskIds: readonly string[]): Effect.Effect<readonly TaskResult[], Error> =>
    Effect.all(taskIds.map(id => this.waitForTask(id)))

  // Execute a single task
  private executeTask = (task: Task): Effect.Effect<void> =>
    Effect.gen(function* () {
      // Wait for dependencies to complete
      if (task.dependencies.length > 0) {
        yield* this.waitForTasks(task.dependencies)
      }
      
      // Execute the task
      const result = yield* task.work()
      const taskResult: TaskResult = {
        taskId: task.id,
        result,
        completedAt: new Date()
      }
      
      // Signal completion
      const signal = this.completionSignals.get(task.id)!
      yield* STM.gen(function* () {
        yield* TDeferred.succeed(signal, taskResult)
        yield* TRef.update(this.completedTasks, (map) => 
          new Map(map).set(task.id, taskResult)
        )
      }).pipe(STM.commit)
    }).pipe(
      Effect.catchAll((error) => {
        const signal = this.completionSignals.get(task.id)!
        return STM.commit(TDeferred.fail(signal, error as Error))
      })
    )

  // Execute all tasks
  executeAll = (): Effect.Effect<Map<string, TaskResult>, Error> =>
    Effect.gen(function* () {
      // Start all tasks concurrently
      const fibers = yield* Effect.all(
        Array.from(this.tasks.values()).map(task =>
          Effect.fork(this.executeTask(task))
        )
      )
      
      // Wait for all to complete
      yield* Effect.all(fibers.map(fiber => Effect.await(fiber)))
      
      // Return completed tasks
      return yield* STM.commit(TRef.get(this.completedTasks))
    })

  // Get completion status
  getCompletionStatus = (): Effect.Effect<{
    completed: readonly string[]
    pending: readonly string[]
    failed: readonly string[]
  }> =>
    STM.gen(function* () {
      const completed: string[] = []
      const pending: string[] = []
      const failed: string[] = []
      
      for (const [taskId, signal] of this.completionSignals) {
        const status = yield* TDeferred.poll(signal)
        
        if (status._tag === "None") {
          pending.push(taskId)
        } else if (status.value._tag === "Left") {
          failed.push(taskId)
        } else {
          completed.push(taskId)
        }
      }
      
      return { completed, pending, failed }
    }).pipe(STM.commit)
}

// Usage
const taskOrchestrationExample = Effect.gen(function* () {
  const tasks: Task[] = [
    {
      id: "setup",
      name: "Setup Environment",
      dependencies: [],
      work: () => Effect.gen(function* () {
        yield* Effect.sleep("100 millis")
        return "Environment ready"
      })
    },
    {
      id: "fetch-data",
      name: "Fetch Data",
      dependencies: ["setup"],
      work: () => Effect.gen(function* () {
        yield* Effect.sleep("200 millis")
        return { data: [1, 2, 3, 4, 5] }
      })
    },
    {
      id: "process-data",
      name: "Process Data",
      dependencies: ["fetch-data"],
      work: () => Effect.gen(function* () {
        yield* Effect.sleep("150 millis")
        return "Data processed"
      })
    },
    {
      id: "send-notification",
      name: "Send Notification",
      dependencies: ["setup"],
      work: () => Effect.gen(function* () {
        yield* Effect.sleep("50 millis")
        return "Notification sent"
      })
    },
    {
      id: "cleanup",
      name: "Cleanup",
      dependencies: ["process-data", "send-notification"],
      work: () => Effect.gen(function* () {
        yield* Effect.sleep("75 millis")
        return "Cleanup complete"
      })
    }
  ]
  
  const orchestrator = yield* TaskOrchestrator.make(tasks)
  
  // Start execution and wait for specific milestone
  const executionFiber = yield* Effect.fork(orchestrator.executeAll())
  
  // Wait for data processing to complete before proceeding
  const processResult = yield* orchestrator.waitForTask("process-data")
  console.log("Data processing completed:", processResult)
  
  // Check overall status
  const status = yield* orchestrator.getCompletionStatus()
  console.log("Current status:", status)
  
  // Wait for all tasks to complete
  const allResults = yield* Effect.await(executionFiber)
  
  return { processResult, finalStatus: status, allResults }
})
```

### Example 3: Circuit Breaker with TDeferred

```typescript
import { Effect, STM, TDeferred, TRef, Clock } from "effect"

type CircuitState = "Closed" | "Open" | "HalfOpen"

interface CircuitBreakerConfig {
  readonly failureThreshold: number
  readonly successThreshold: number
  readonly timeout: number
}

class CircuitBreaker<A, E> {
  constructor(
    private config: CircuitBreakerConfig,
    private state: TRef.TRef<CircuitState>,
    private failureCount: TRef.TRef<number>,
    private successCount: TRef.TRef<number>,
    private lastFailureTime: TRef.TRef<number>,
    private nextAttemptSignal: TRef.TRef<TDeferred.TDeferred<void, never> | null>
  ) {}

  static make = <A, E>(config: CircuitBreakerConfig): Effect.Effect<CircuitBreaker<A, E>> =>
    STM.gen(function* () {
      const state = yield* TRef.make<CircuitState>("Closed")
      const failureCount = yield* TRef.make(0)
      const successCount = yield* TRef.make(0)
      const lastFailureTime = yield* TRef.make(0)
      const nextAttemptSignal = yield* TRef.make<TDeferred.TDeferred<void, never> | null>(null)
      
      return new CircuitBreaker(config, state, failureCount, successCount, lastFailureTime, nextAttemptSignal)
    }).pipe(STM.commit)

  private shouldAllowRequest = (): STM.STM<boolean> =>
    STM.gen(function* () {
      const currentState = yield* TRef.get(this.state)
      
      switch (currentState) {
        case "Closed":
          return true
        case "Open": {
          const lastFailure = yield* TRef.get(this.lastFailureTime)
          const now = Date.now()
          
          if (now - lastFailure >= this.config.timeout) {
            // Transition to half-open and create signal for next attempt
            yield* TRef.set(this.state, "HalfOpen")
            yield* TRef.set(this.successCount, 0)
            const signal = yield* TDeferred.make<void, never>()
            yield* TRef.set(this.nextAttemptSignal, signal)
            return true
          }
          return false
        }
        case "HalfOpen": {
          // Only one request allowed in half-open state
          const signal = yield* TRef.get(this.nextAttemptSignal)
          if (signal) {
            // Wait for the current attempt to complete
            yield* TDeferred.await(signal)
          }
          return true
        }
      }
    })

  private recordSuccess = (): STM.STM<void> =>
    STM.gen(function* () {
      const currentState = yield* TRef.get(this.state)
      
      if (currentState === "HalfOpen") {
        const newSuccessCount = yield* TRef.updateAndGet(this.successCount, (c) => c + 1)
        
        if (newSuccessCount >= this.config.successThreshold) {
          // Transition back to closed
          yield* TRef.set(this.state, "Closed")
          yield* TRef.set(this.failureCount, 0)
          yield* TRef.set(this.successCount, 0)
          
          // Signal that the attempt completed
          const signal = yield* TRef.get(this.nextAttemptSignal)
          if (signal) {
            yield* TDeferred.succeed(signal, void 0)
            yield* TRef.set(this.nextAttemptSignal, null)
          }
        }
      } else if (currentState === "Closed") {
        yield* TRef.set(this.failureCount, 0)
      }
    })

  private recordFailure = (): STM.STM<void> =>
    STM.gen(function* () {
      const currentState = yield* TRef.get(this.state)
      const newFailureCount = yield* TRef.updateAndGet(this.failureCount, (c) => c + 1)
      
      if (currentState === "Closed" && newFailureCount >= this.config.failureThreshold) {
        // Transition to open
        yield* TRef.set(this.state, "Open")
        yield* TRef.set(this.lastFailureTime, Date.now())
      } else if (currentState === "HalfOpen") {
        // Go back to open
        yield* TRef.set(this.state, "Open")
        yield* TRef.set(this.lastFailureTime, Date.now())
        
        // Signal that the attempt completed
        const signal = yield* TRef.get(this.nextAttemptSignal)
        if (signal) {
          yield* TDeferred.succeed(signal, void 0)
          yield* TRef.set(this.nextAttemptSignal, null)
        }
      }
    })

  execute = (operation: Effect.Effect<A, E>): Effect.Effect<A, E | Error> =>
    Effect.gen(function* () {
      const canProceed = yield* STM.commit(this.shouldAllowRequest())
      
      if (!canProceed) {
        return yield* Effect.fail(new Error("Circuit breaker is open"))
      }
      
      return yield* operation.pipe(
        Effect.tap(() => STM.commit(this.recordSuccess())),
        Effect.catchAll((error) =>
          Effect.gen(function* () {
            yield* STM.commit(this.recordFailure())
            return yield* Effect.fail(error)
          })
        )
      )
    })

  getState = (): Effect.Effect<{
    state: CircuitState
    failureCount: number
    successCount: number
  }> =>
    STM.gen(function* () {
      const state = yield* TRef.get(this.state)
      const failureCount = yield* TRef.get(this.failureCount)
      const successCount = yield* TRef.get(this.successCount)
      
      return { state, failureCount, successCount }
    }).pipe(STM.commit)
}

// Usage
const circuitBreakerExample = Effect.gen(function* () {
  const circuitBreaker = yield* CircuitBreaker.make<string, Error>({
    failureThreshold: 3,
    successThreshold: 2,
    timeout: 1000
  })
  
  // Simulate an unreliable service
  let attemptCount = 0
  const unreliableService = Effect.gen(function* () {
    attemptCount++
    yield* Effect.sleep("100 millis")
    
    // Fail first 4 attempts, then succeed
    if (attemptCount <= 4) {
      return yield* Effect.fail(new Error(`Service failure ${attemptCount}`))
    }
    return `Success on attempt ${attemptCount}`
  })
  
  const results: Array<{ attempt: number; result: string }> = []
  
  // Make multiple requests
  for (let i = 1; i <= 8; i++) {
    const result = yield* circuitBreaker.execute(unreliableService).pipe(
      Effect.either,
      Effect.tap(() => Effect.sleep("200 millis"))
    )
    
    const state = yield* circuitBreaker.getState()
    
    results.push({
      attempt: i,
      result: result._tag === "Left" 
        ? `Error: ${result.left.message} (State: ${state.state})`
        : `Success: ${result.right} (State: ${state.state})`
    })
  }
  
  return results
})
```

## Advanced Features Deep Dive

### Feature 1: Composing with Other STM Operations

TDeferred operations can be composed with other STM structures for complex coordination:

```typescript
import { STM, TDeferred, TRef, TMap } from "effect"

const compositeCoordination = STM.gen(function* () {
  const deferred = yield* TDeferred.make<string, never>()
  const counter = yield* TRef.make(0)
  const results = yield* TMap.empty<string, number>()
  
  // Complex atomic operation
  const operation = STM.gen(function* () {
    // Wait for deferred value
    const value = yield* TDeferred.await(deferred)
    
    // Update counter
    const newCount = yield* TRef.updateAndGet(counter, (c) => c + 1)
    
    // Store result
    yield* TMap.set(results, value, newCount)
    
    return { value, count: newCount }
  })
  
  // Complete deferred in parallel with operation
  const completer = TDeferred.succeed(deferred, "computed-value")
  
  // Both operations are atomic
  yield* STM.all([operation, completer])
  
  const finalResults = yield* TMap.toArray(results)
  return finalResults
})
```

### Feature 2: Done Operations with Exit Values

```typescript
import { STM, TDeferred, Exit } from "effect"

const exitOperations = STM.gen(function* () {
  const deferred = yield* TDeferred.make<string, Error>()
  
  // Complete with Exit value (success or failure)
  const successExit = Exit.succeed("Success value")
  const failureExit = Exit.fail(new Error("Failure value"))
  
  yield* TDeferred.done(deferred, successExit)
  
  const result = yield* TDeferred.await(deferred)
  return result
})
```

### Feature 3: Conditional Completion Patterns

```typescript
const conditionalCompletion = <A, E>(
  deferred: TDeferred.TDeferred<A, E>,
  condition: () => STM.STM<boolean>,
  successValue: A,
  failureValue: E
): STM.STM<void> =>
  STM.gen(function* () {
    const shouldSucceed = yield* condition()
    
    if (shouldSucceed) {
      yield* TDeferred.succeed(deferred, successValue)
    } else {
      yield* TDeferred.fail(deferred, failureValue)
    }
  })

// Usage
const conditionalExample = STM.gen(function* () {
  const deferred = yield* TDeferred.make<string, Error>()
  const condition = STM.succeed(Math.random() > 0.5)
  
  yield* conditionalCompletion(
    deferred,
    () => condition,
    "Lucky!",
    new Error("Unlucky!")
  )
  
  return yield* TDeferred.await(deferred)
})
```

## Practical Patterns & Best Practices

### Pattern 1: Timeout with TDeferred

```typescript
// Helper for timeout operations
const withTimeout = <A, E>(
  deferred: TDeferred.TDeferred<A, E>,
  timeoutMs: number,
  timeoutValue: E
): STM.STM<A, E> =>
  STM.gen(function* () {
    const timeoutDeferred = yield* TDeferred.make<never, E>()
    
    // Set up timeout
    Effect.sleep(`${timeoutMs} millis`).pipe(
      Effect.flatMap(() => STM.commit(TDeferred.fail(timeoutDeferred, timeoutValue))),
      Effect.fork
    )
    
    // Race between completion and timeout
    return yield* STM.race(
      TDeferred.await(deferred),
      TDeferred.await(timeoutDeferred)
    )
  })
```

### Pattern 2: Broadcast Pattern

```typescript
// Helper for broadcasting to multiple deferreds
const broadcast = <A>(
  deferreds: readonly TDeferred.TDeferred<A, never>[],
  value: A
): STM.STM<void> =>
  STM.gen(function* () {
    for (const deferred of deferreds) {
      yield* TDeferred.succeed(deferred, value)
    }
  })

// Usage
const broadcastExample = STM.gen(function* () {
  const deferreds = yield* STM.all([
    TDeferred.make<string, never>(),
    TDeferred.make<string, never>(),
    TDeferred.make<string, never>()
  ])
  
  // Broadcast value to all
  yield* broadcast(deferreds, "Broadcast message")
  
  // All deferreds now have the same value
  const results = yield* STM.all(deferreds.map(d => TDeferred.await(d)))
  return results
})
```

### Pattern 3: Barrier Synchronization

```typescript
// Helper for barrier synchronization
const createBarrier = (participantCount: number) => STM.gen(function* () {
  const barrier = yield* TDeferred.make<void, never>()
  const counter = yield* TRef.make(0)
  
  const wait = STM.gen(function* () {
    const newCount = yield* TRef.updateAndGet(counter, (c) => c + 1)
    
    if (newCount === participantCount) {
      yield* TDeferred.succeed(barrier, void 0)
    }
    
    yield* TDeferred.await(barrier)
  })
  
  return { wait }
})

// Usage
const barrierExample = Effect.gen(function* () {
  const { wait } = yield* STM.commit(createBarrier(3))
  
  // Three concurrent operations that synchronize
  const operations = [
    Effect.gen(function* () {
      yield* Effect.sleep("100 millis")
      yield* STM.commit(wait)
      return "Operation 1 completed"
    }),
    Effect.gen(function* () {
      yield* Effect.sleep("200 millis")
      yield* STM.commit(wait)
      return "Operation 2 completed"
    }),
    Effect.gen(function* () {
      yield* Effect.sleep("300 millis")
      yield* STM.commit(wait)
      return "Operation 3 completed"
    })
  ]
  
  // All operations will complete at roughly the same time
  return yield* Effect.all(operations, { concurrency: "unbounded" })
})
```

## Integration Examples

### Integration with Effect Fibers

```typescript
import { Effect, STM, TDeferred, Fiber } from "effect"

const fiberCoordination = Effect.gen(function* () {
  const resultDeferred = yield* STM.commit(TDeferred.make<string[], never>())
  
  // Create multiple worker fibers
  const workers = yield* Effect.all([
    Effect.fork(Effect.gen(function* () {
      yield* Effect.sleep("100 millis")
      return "Worker 1 result"
    })),
    Effect.fork(Effect.gen(function* () {
      yield* Effect.sleep("200 millis")
      return "Worker 2 result"
    })),
    Effect.fork(Effect.gen(function* () {
      yield* Effect.sleep("150 millis")
      return "Worker 3 result"
    }))
  ])
  
  // Coordinator fiber that collects results
  const coordinator = yield* Effect.fork(
    Effect.gen(function* () {
      const results = yield* Effect.all(workers.map(fiber => Effect.await(fiber)))
      yield* STM.commit(TDeferred.succeed(resultDeferred, results))
    })
  )
  
  // Consumer waiting for all results
  const allResults = yield* STM.commit(TDeferred.await(resultDeferred))
  
  yield* Effect.await(coordinator)
  
  return allResults
})
```

### Testing Strategies

```typescript
import { Effect, STM, TDeferred, TestClock, TestContext } from "effect"

// Test utility for TDeferred operations
const testDeferredCompletion = <A>(
  operation: (deferred: TDeferred.TDeferred<A, never>) => Effect.Effect<void>,
  expectedValue: A
) =>
  Effect.gen(function* () {
    const deferred = yield* STM.commit(TDeferred.make<A, never>())
    
    // Start operation
    yield* Effect.fork(operation(deferred))
    
    // Wait for completion
    const result = yield* STM.commit(TDeferred.await(deferred))
    
    return result === expectedValue
  })

// Test concurrent completion
const testConcurrentCompletion = Effect.gen(function* () {
  const deferred = yield* STM.commit(TDeferred.make<number, never>())
  
  // Multiple concurrent completers (only first should succeed)
  const completers = [
    Effect.delay(STM.commit(TDeferred.succeed(deferred, 1)), "100 millis"),
    Effect.delay(STM.commit(TDeferred.succeed(deferred, 2)), "200 millis"),
    Effect.delay(STM.commit(TDeferred.succeed(deferred, 3)), "300 millis")
  ]
  
  yield* Effect.all(completers, { concurrency: "unbounded" }).pipe(
    Effect.ignore // Ignore failures from duplicate completions
  )
  
  const result = yield* STM.commit(TDeferred.await(deferred))
  return result // Should be 1 (first to complete)
}).pipe(
  Effect.provide(TestContext.TestContext)
)
```

## Conclusion

TDeferred provides atomic one-time coordination for concurrent operations, resource initialization, and complex synchronization patterns.

Key benefits:
- **Atomic Coordination**: All operations are atomic within STM transactions
- **One-Time Semantics**: Prevents duplicate completion and ensures consistency
- **Composability**: Seamlessly integrates with other STM operations and Effect constructs

TDeferred is ideal for scenarios requiring one-time coordination, initialization barriers, and complex synchronization patterns where traditional promises and callbacks fall short.