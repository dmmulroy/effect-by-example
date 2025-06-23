# FiberHandle: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem FiberHandle Solves

Managing long-running background tasks in concurrent applications is notoriously difficult. Traditional approaches often lead to memory leaks, resource exhaustion, and unpredictable fiber lifecycles:

```typescript
// Traditional approach - manual fiber management
let backgroundFiber: Fiber.RuntimeFiber<void, never> | undefined

const startBackgroundWork = () => {
  // Previous fiber might still be running!
  backgroundFiber = Effect.runFork(longRunningTask)
}

const stopBackgroundWork = () => {
  // Manual cleanup required
  if (backgroundFiber) {
    Effect.runSync(Fiber.interrupt(backgroundFiber))
  }
}

// Easy to forget cleanup, leading to resource leaks
// No automatic interruption when scope closes
// Race conditions with concurrent access
```

This approach leads to:
- **Resource Leaks** - Fibers continue running after they're no longer needed
- **Race Conditions** - Multiple fibers competing for the same resources
- **Manual Cleanup** - Developers must remember to interrupt fibers
- **Scope Misalignment** - Fiber lifetimes don't align with business logic scopes

### The FiberHandle Solution

FiberHandle provides a managed container for a single fiber with automatic lifecycle management:

```typescript
import { Effect, FiberHandle, Scope } from "effect"

const managedBackgroundWork = Effect.gen(function* () {
  const handle = yield* FiberHandle.make()
  
  // Start background work - automatically managed
  yield* FiberHandle.run(handle, longRunningTask)
  
  // Start different work - previous fiber automatically interrupted
  yield* FiberHandle.run(handle, differentTask)
  
  // No manual cleanup needed - automatic interruption on scope close
}).pipe(Effect.scoped)
```

### Key Concepts

**FiberHandle**: A scoped container that holds at most one fiber, automatically interrupting it when replaced or when the scope closes

**Automatic Interruption**: When a new fiber is added to the handle, the previous fiber is interrupted unless explicitly configured otherwise

**Scope Integration**: FiberHandles integrate with Effect's scope system, ensuring fibers are cleaned up when their containing scope closes

## Basic Usage Patterns

### Pattern 1: Creating and Using a FiberHandle

```typescript
import { Effect, FiberHandle } from "effect"

const basicUsage = Effect.gen(function* () {
  // Create a fiber handle within a scope
  const handle = yield* FiberHandle.make<string, never>()
  
  // Run an effect in the handle
  const fiber = yield* FiberHandle.run(handle, Effect.succeed("Hello, World!"))
  
  // Get the result
  const result = yield* Fiber.await(fiber)
  console.log(result) // "Hello, World!"
}).pipe(Effect.scoped)
```

### Pattern 2: Replacing Running Fibers

```typescript
const fiberReplacement = Effect.gen(function* () {
  const handle = yield* FiberHandle.make<number, never>()
  
  // Start first task
  yield* FiberHandle.run(handle, Effect.delay(Effect.succeed(1), "1 second"))
  
  // Start second task - first task is automatically interrupted
  yield* FiberHandle.run(handle, Effect.delay(Effect.succeed(2), "500 millis"))
  
  // Only the second task will complete
}).pipe(Effect.scoped)
```

### Pattern 3: Conditional Fiber Management

```typescript
const conditionalManagement = Effect.gen(function* () {
  const handle = yield* FiberHandle.make<void, never>()
  
  // Start a long-running task
  yield* FiberHandle.run(handle, Effect.never)
  
  // Try to start another task, but only if handle is empty
  yield* FiberHandle.run(handle, Effect.succeed(void 0), {
    onlyIfMissing: true // This won't run because handle already has a fiber
  })
}).pipe(Effect.scoped)
```

## Real-World Examples

### Example 1: Background Data Synchronization

Managing periodic data synchronization with automatic cleanup:

```typescript
import { Effect, FiberHandle, Schedule, Ref } from "effect"

interface DatabaseSyncService {
  readonly sync: Effect.Effect<void, SyncError>
  readonly start: Effect.Effect<void, never>
  readonly stop: Effect.Effect<void, never>
  readonly isRunning: Effect.Effect<boolean, never>
}

const makeDatabaseSyncService = Effect.gen(function* () {
  const handle = yield* FiberHandle.make<void, SyncError>()
  const isRunningRef = yield* Ref.make(false)
  
  const syncLoop = Effect.gen(function* () {
    yield* Ref.set(isRunningRef, true)
    yield* Effect.log("Starting database sync loop")
    
    yield* Effect.repeat(
      Effect.gen(function* () {
        yield* Effect.log("Syncing database...")
        yield* databaseClient.sync()
        yield* Effect.log("Database sync completed")
      }),
      Schedule.fixed("30 seconds")
    )
  }).pipe(
    Effect.ensuring(Ref.set(isRunningRef, false)),
    Effect.onInterrupt(() => Effect.log("Database sync interrupted"))
  )
  
  const start = FiberHandle.run(handle, syncLoop, { onlyIfMissing: true }).pipe(
    Effect.asVoid
  )
  
  const stop = FiberHandle.clear(handle)
  
  const isRunning = Ref.get(isRunningRef)
  
  return {
    sync: databaseClient.sync(),
    start,
    stop,
    isRunning
  } as const satisfies DatabaseSyncService
})

// Usage in application
const applicationWithSync = Effect.gen(function* () {
  const syncService = yield* makeDatabaseSyncService
  
  // Start background synchronization
  yield* syncService.start
  
  // Do other application work
  yield* handleUserRequests()
  
  // Sync service automatically stops when scope closes
}).pipe(
  Effect.scoped,
  Effect.provide(DatabaseClientLayer)
)
```

### Example 2: Timeout Management for Operations

Managing operation timeouts with fiber handles for clean cancellation:

```typescript
import { Effect, FiberHandle, Duration } from "effect"

interface TimeoutManager {
  readonly startTimeout: <A, E>(
    operation: Effect.Effect<A, E>,
    timeout: Duration.DurationInput,
    onTimeout: () => Effect.Effect<void, never>
  ) => Effect.Effect<A, E | TimeoutError>
  readonly cancelTimeout: Effect.Effect<void, never>
}

class TimeoutError {
  readonly _tag = "TimeoutError"
  constructor(readonly duration: Duration.Duration) {}
}

const makeTimeoutManager = Effect.gen(function* () {
  const timeoutHandle = yield* FiberHandle.make<void, never>()
  
  const startTimeout = <A, E>(
    operation: Effect.Effect<A, E>,
    timeout: Duration.DurationInput,
    onTimeout: () => Effect.Effect<void, never>
  ) => Effect.gen(function* () {
    // Start timeout fiber
    yield* FiberHandle.run(
      timeoutHandle,
      Effect.gen(function* () {
        yield* Effect.sleep(timeout)
        yield* onTimeout()
        yield* Effect.fail(new TimeoutError(Duration.decode(timeout)))
      })
    )
    
    // Race the operation against the timeout
    const result = yield* Effect.race(
      operation,
      FiberHandle.join(timeoutHandle)
    )
    
    // Clear timeout on successful completion
    yield* FiberHandle.clear(timeoutHandle)
    
    return result
  })
  
  const cancelTimeout = FiberHandle.clear(timeoutHandle)
  
  return { startTimeout, cancelTimeout } as const satisfies TimeoutManager
})

// Usage for API calls with timeout
const apiCallWithTimeout = Effect.gen(function* () {
  const timeoutManager = yield* makeTimeoutManager
  
  const result = yield* timeoutManager.startTimeout(
    httpClient.get("/api/data"),
    "10 seconds",
    () => Effect.log("API call timed out, cleaning up resources")
  )
  
  return result
}).pipe(
  Effect.scoped,
  Effect.catchTag("TimeoutError", (error) =>
    Effect.gen(function* () {
      yield* Effect.log(`Operation timed out after ${Duration.format(error.duration)}`)
      return yield* Effect.fail(new ApiTimeoutError())
    })
  )
)
```

### Example 3: Dynamic Worker Pool Management

Managing a pool of workers that can be dynamically started and stopped:

```typescript
import { Effect, FiberHandle, Array as Arr, Ref, Queue } from "effect"

interface WorkerPool<A> {
  readonly submit: (task: A) => Effect.Effect<void, never>
  readonly resize: (newSize: number) => Effect.Effect<void, never>
  readonly shutdown: Effect.Effect<void, never>
  readonly status: Effect.Effect<WorkerPoolStatus, never>
}

interface WorkerPoolStatus {
  readonly activeWorkers: number
  readonly queuedTasks: number
  readonly isShutdown: boolean
}

const makeWorkerPool = <A>(
  processor: (task: A) => Effect.Effect<void, ProcessingError>,
  initialSize: number = 3
): Effect.Effect<WorkerPool<A>, never, Scope.Scope> => Effect.gen(function* () {
  const taskQueue = yield* Queue.unbounded<A>()
  const workerHandles = yield* Ref.make<ReadonlyArray<FiberHandle<void, ProcessingError>>>(
    Arr.empty()
  )
  const isShutdownRef = yield* Ref.make(false)
  
  const createWorker = Effect.gen(function* () {
    const handle = yield* FiberHandle.make<void, ProcessingError>()
    
    const workerLoop = Effect.gen(function* () {
      while (true) {
        const task = yield* Queue.take(taskQueue)
        yield* processor(task)
      }
    }).pipe(
      Effect.forever,
      Effect.onInterrupt(() => Effect.log("Worker shutting down"))
    )
    
    yield* FiberHandle.run(handle, workerLoop)
    return handle
  })
  
  const startWorkers = (count: number) => Effect.gen(function* () {
    const newHandles = yield* Effect.all(
      Arr.replicate(createWorker, count),
      { concurrency: "unbounded" }
    )
    
    yield* Ref.update(workerHandles, Arr.appendAll(newHandles))
  })
  
  const stopWorkers = (count: number) => Effect.gen(function* () {
    const handles = yield* Ref.get(workerHandles)
    const toStop = Arr.take(handles, count)
    const remaining = Arr.drop(handles, count)
    
    yield* Effect.all(
      Arr.map(toStop, FiberHandle.clear),
      { concurrency: "unbounded" }
    )
    
    yield* Ref.set(workerHandles, remaining)
  })
  
  // Initialize workers
  yield* startWorkers(initialSize)
  
  const submit = (task: A) => Effect.gen(function* () {
    const isShutdown = yield* Ref.get(isShutdownRef)
    if (isShutdown) {
      return yield* Effect.fail(new Error("Worker pool is shutdown"))
    }
    yield* Queue.offer(taskQueue, task)
  }).pipe(Effect.orDie)
  
  const resize = (newSize: number) => Effect.gen(function* () {
    const currentHandles = yield* Ref.get(workerHandles)
    const currentSize = Arr.length(currentHandles)
    
    if (newSize > currentSize) {
      yield* startWorkers(newSize - currentSize)
    } else if (newSize < currentSize) {
      yield* stopWorkers(currentSize - newSize)
    }
  })
  
  const shutdown = Effect.gen(function* () {
    yield* Ref.set(isShutdownRef, true)
    yield* resize(0)
    yield* Queue.shutdown(taskQueue)
  })
  
  const status = Effect.gen(function* () {
    const handles = yield* Ref.get(workerHandles)
    const queueSize = yield* Queue.size(taskQueue)
    const isShutdown = yield* Ref.get(isShutdownRef)
    
    return {
      activeWorkers: Arr.length(handles),
      queuedTasks: queueSize,
      isShutdown
    } as const satisfies WorkerPoolStatus
  })
  
  return { submit, resize, shutdown, status } as const satisfies WorkerPool<A>
})

// Usage example
const imageProcessingPool = Effect.gen(function* () {
  const pool = yield* makeWorkerPool(
    (image: ImageTask) => processImage(image),
    5 // Start with 5 workers
  )
  
  // Submit tasks
  yield* pool.submit({ path: "/images/photo1.jpg", format: "webp" })
  yield* pool.submit({ path: "/images/photo2.jpg", format: "webp" })
  
  // Dynamically resize based on load
  const status = yield* pool.status
  if (status.queuedTasks > 10) {
    yield* pool.resize(status.activeWorkers + 2)
  }
  
  // Pool automatically shuts down when scope closes
}).pipe(Effect.scoped)
```

## Advanced Features Deep Dive

### Feature 1: Runtime Capture with makeRuntime

The `makeRuntime` function creates a runtime-capturing function that can execute effects with the current runtime context:

#### Basic Runtime Usage

```typescript
const runtimeExample = Effect.gen(function* () {
  // Create a runtime function backed by a FiberHandle
  const run = yield* FiberHandle.makeRuntime<DatabaseService>()
  
  // Execute effects with the current runtime
  const fiber1 = run(Effect.andThen(DatabaseService, db => db.query("SELECT * FROM users")))
  const fiber2 = run(Effect.andThen(DatabaseService, db => db.query("SELECT * FROM orders")))
  
  // Second query interrupts the first
  const result = yield* Fiber.await(fiber2)
  return result
}).pipe(
  Effect.scoped,
  Effect.provide(DatabaseServiceLive)
)
```

#### Real-World Runtime Example: Event Stream Processor

```typescript
const makeEventStreamProcessor = Effect.gen(function* () {
  const run = yield* FiberHandle.makeRuntime<EventBus | Logger>()
  
  const processEventStream = (streamName: string) => Effect.gen(function* () {
    const eventBus = yield* EventBus
    const logger = yield* Logger
    
    yield* logger.info(`Starting event stream processor for: ${streamName}`)
    
    yield* Effect.repeat(
      Effect.gen(function* () {
        const events = yield* eventBus.pollEvents(streamName)
        yield* Effect.forEach(events, processEvent, { concurrency: 5 })
      }),
      Schedule.fixed("1 second")
    )
  })
  
  return {
    startStream: (streamName: string) => run(processEventStream(streamName)),
    stopCurrentStream: () => Effect.void // Automatic via FiberHandle
  } as const
})
```

### Feature 2: Promise Integration with makeRuntimePromise

For integrating with Promise-based APIs while maintaining fiber lifecycle management:

```typescript
const promiseIntegration = Effect.gen(function* () {
  const runPromise = yield* FiberHandle.makeRuntimePromise<HttpClient>()
  
  // Convert Effect to Promise while maintaining fiber management
  const fetchData = async (url: string): Promise<ApiResponse> => {
    return runPromise(
      Effect.gen(function* () {
        const client = yield* HttpClient
        return yield* client.get(url)
      })
    )
  }
  
  // Use in existing Promise-based code
  const result = await fetchData("/api/users")
  return result
}).pipe(
  Effect.scoped,
  Effect.provide(HttpClientLive)
)
```

### Feature 3: Advanced Error Propagation

Control how errors are propagated from managed fibers:

#### Propagation Control Example

```typescript
const errorPropagationExample = Effect.gen(function* () {
  const handle = yield* FiberHandle.make<void, DatabaseError>()
  
  // With propagateInterruption: false (default)
  // Interruptions won't propagate to the handle's deferred
  yield* FiberHandle.run(
    handle,
    Effect.gen(function* () {
      yield* Effect.sleep("1 second")
      yield* Effect.fail(new DatabaseError("Connection lost"))
    }),
    { propagateInterruption: false }
  )
  
  // With propagateInterruption: true
  // All errors including interruptions will propagate
  yield* FiberHandle.run(
    handle,
    criticalOperation,
    { propagateInterruption: true }
  )
  
  // Join will fail if any managed fiber fails
  yield* FiberHandle.join(handle)
}).pipe(Effect.scoped)
```

## Practical Patterns & Best Practices

### Pattern 1: Resource-Aware Fiber Management

```typescript
const createResourceAwareFiberHandle = <R, E, A>(
  resource: Effect.Effect<R, E>,
  cleanup: (resource: R) => Effect.Effect<void, never>
) => Effect.gen(function* () {
  const handle = yield* FiberHandle.make<A, E>()
  const resourceRef = yield* Ref.make<Option.Option<R>>(Option.none())
  
  const runWithResource = <XE extends E, XA extends A>(
    effect: (resource: R) => Effect.Effect<XA, XE>
  ) => Effect.gen(function* () {
    const fiber = yield* FiberHandle.run(
      handle,
      Effect.gen(function* () {
        const res = yield* resource
        yield* Ref.set(resourceRef, Option.some(res))
        
        return yield* effect(res).pipe(
          Effect.ensuring(
            Effect.gen(function* () {
              yield* cleanup(res)
              yield* Ref.set(resourceRef, Option.none())
            })
          )
        )
      })
    )
    
    return fiber
  })
  
  const getCurrentResource = Effect.flatMap(
    Ref.get(resourceRef),
    Option.match({
      onNone: () => Effect.fail(new Error("No resource available")),
      onSome: Effect.succeed
    })
  )
  
  return { runWithResource, getCurrentResource, handle } as const
})
```

### Pattern 2: Graceful Shutdown Pattern

```typescript
const makeGracefulShutdownHandle = <A, E>() => Effect.gen(function* () {
  const handle = yield* FiberHandle.make<A, E>()
  const shutdownSignal = yield* Deferred.make<void, never>()
  
  const runWithGracefulShutdown = <XE extends E, XA extends A>(
    effect: Effect.Effect<XA, XE>,
    onShutdown: Effect.Effect<void, never> = Effect.void
  ) => Effect.gen(function* () {
    const fiber = yield* FiberHandle.run(
      handle,
      Effect.race(
        effect,
        Effect.gen(function* () {
          yield* Deferred.await(shutdownSignal)
          yield* onShutdown
          return yield* Effect.interrupt
        })
      )
    )
    
    return fiber
  })
  
  const gracefulShutdown = Effect.gen(function* () {
    yield* Deferred.succeed(shutdownSignal, void 0)
    yield* FiberHandle.awaitEmpty(handle)
  })
  
  return { runWithGracefulShutdown, gracefulShutdown } as const
})
```

### Pattern 3: Fiber Handle Composition

```typescript
const composeFiberHandles = <A, B, E>(
  handleA: FiberHandle<A, E>,
  handleB: FiberHandle<B, E>
) => {
  const runBoth = <XE extends E>(
    effectA: Effect.Effect<A, XE>,
    effectB: Effect.Effect<B, XE>
  ) => Effect.gen(function* () {
    const fiberA = yield* FiberHandle.run(handleA, effectA)
    const fiberB = yield* FiberHandle.run(handleB, effectB)
    
    return yield* Fiber.all([fiberA, fiberB])
  })
  
  const clearBoth = Effect.all([
    FiberHandle.clear(handleA),
    FiberHandle.clear(handleB)
  ])
  
  const joinBoth = Effect.all([
    FiberHandle.join(handleA),
    FiberHandle.join(handleB)
  ])
  
  return { runBoth, clearBoth, joinBoth } as const
}
```

## Integration Examples

### Integration with HTTP Server

```typescript
import { HttpServer, HttpRouter, HttpMiddleware } from "@effect/platform"

const makeServerWithBackgroundTasks = Effect.gen(function* () {
  const backgroundHandle = yield* FiberHandle.make<void, never>()
  
  // Background task that runs alongside the server
  const backgroundTask = Effect.gen(function* () {
    yield* Effect.log("Starting background maintenance task")
    yield* Effect.repeat(
      Effect.gen(function* () {
        yield* cleanupExpiredSessions()
        yield* updateMetrics()
        yield* Effect.log("Background maintenance completed")
      }),
      Schedule.fixed("5 minutes")
    )
  })
  
  const router = HttpRouter.empty.pipe(
    HttpRouter.get("/health", 
      Effect.gen(function* () {
        const isBackgroundRunning = Option.isSome(FiberHandle.unsafeGet(backgroundHandle))
        return HttpResponse.json({
          status: "healthy",
          backgroundTask: isBackgroundRunning ? "running" : "stopped"
        })
      })
    ),
    HttpRouter.post("/admin/restart-background",
      Effect.gen(function* () {
        yield* FiberHandle.run(backgroundHandle, backgroundTask)
        return HttpResponse.json({ message: "Background task restarted" })
      })
    )
  )
  
  const server = HttpServer.make(router).pipe(
    HttpServer.withLogAddress,
    HttpServer.serve()
  )
  
  return Effect.gen(function* () {
    // Start background task
    yield* FiberHandle.run(backgroundHandle, backgroundTask)
    
    // Start server
    yield* server
  })
}).pipe(
  Effect.scoped,
  Effect.provide(HttpServer.layer({ port: 3000 }))
)
```

### Integration with Testing

```typescript
import { describe, it, expect } from "@effect/vitest"

describe("FiberHandle Testing Patterns", () => {
  it.scoped("should handle fiber lifecycle in tests", () =>
    Effect.gen(function* () {
      const handle = yield* FiberHandle.make<string, TestError>()
      const resultRef = yield* Ref.make<Option.Option<string>>(Option.none())
      
      // Start a task that sets a result
      yield* FiberHandle.run(
        handle,
        Effect.gen(function* () {
          yield* Effect.sleep("100 millis")
          yield* Ref.set(resultRef, Option.some("completed"))
          return "done"
        })
      )
      
      // Wait for completion
      yield* FiberHandle.awaitEmpty(handle)
      
      // Verify result
      const result = yield* Ref.get(resultRef)
      expect(Option.isSome(result)).toBe(true)
      expect(result.value).toBe("completed")
    })
  )
  
  it.scoped("should test interruption behavior", () =>
    Effect.gen(function* () {
      const handle = yield* FiberHandle.make<void, never>()
      const interruptedRef = yield* Ref.make(false)
      
      // Start a task that should be interrupted
      yield* FiberHandle.run(
        handle,
        Effect.onInterrupt(
          Effect.never,
          () => Ref.set(interruptedRef, true)
        )
      )
      
      // Start another task to interrupt the first
      yield* FiberHandle.run(handle, Effect.void)
      
      // Verify interruption occurred
      yield* TestClock.adjust("1 second")
      const wasInterrupted = yield* Ref.get(interruptedRef)
      expect(wasInterrupted).toBe(true)
    })
  )
})
```

### Integration with Stream Processing

```typescript
import { Stream, Sink } from "effect"

const makeStreamProcessor = <A, E>(
  source: Stream.Stream<A, E>,
  processor: (item: A) => Effect.Effect<void, ProcessingError>
) => Effect.gen(function* () {
  const processingHandle = yield* FiberHandle.make<void, E | ProcessingError>()
  
  const startProcessing = FiberHandle.run(
    processingHandle,
    Stream.run(
      source,
      Sink.forEach(processor)
    )
  )
  
  const stopProcessing = FiberHandle.clear(processingHandle)
  
  const restartProcessing = Effect.gen(function* () {
    yield* stopProcessing
    yield* startProcessing
  })
  
  return { startProcessing, stopProcessing, restartProcessing } as const
})

// Usage with event streams
const eventProcessor = Effect.gen(function* () {
  const eventStream = Stream.fromQueue(eventQueue)
  const processor = makeStreamProcessor(
    eventStream, 
    (event: DomainEvent) => handleDomainEvent(event)
  )
  
  yield* processor.startProcessing
  
  // Process events until scope closes
  yield* Effect.never
}).pipe(Effect.scoped)
```

## Conclusion

FiberHandle provides **automatic lifecycle management**, **scope integration**, and **safe concurrency** for managing single-fiber workloads in Effect applications.

Key benefits:
- **Automatic Cleanup**: Fibers are automatically interrupted when scopes close or when replaced
- **Resource Safety**: Prevents memory leaks and resource exhaustion from abandoned fibers
- **Concurrency Control**: Ensures only one fiber runs at a time per handle, eliminating race conditions

FiberHandle is ideal for managing background tasks, timeout operations, worker processes, and any scenario requiring controlled fiber lifecycle management with automatic cleanup guarantees.