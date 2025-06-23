# FiberSet: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem FiberSet Solves

Managing collections of concurrent operations is a common challenge in applications. Without proper coordination, we end up with scattered fiber management, resource leaks, and difficult cleanup:

```typescript
// Traditional approach - problematic fiber management
import { Effect, Fiber } from "effect"

const processItems = (items: string[]) => Effect.gen(function* () {
  const fibers: Fiber.RuntimeFiber<void, Error>[] = []
  
  // Fork multiple operations
  for (const item of items) {
    const fiber = yield* Effect.fork(processItem(item))
    fibers.push(fiber) // Manual tracking
  }
  
  // Manual cleanup - error-prone
  try {
    yield* Effect.forEach(fibers, Fiber.join)
  } finally {
    // What if some fibers are still running?
    // What about resource cleanup?
    // How do we handle partial failures?
  }
})
```

This approach leads to:
- **Resource Leaks** - Fibers may not be properly interrupted on scope closure
- **Complex Error Handling** - Managing failures across multiple fibers is difficult  
- **Manual Lifecycle Management** - Tracking, joining, and cleaning up fibers manually
- **No Supervision** - No centralized way to monitor or control fiber collections

### The FiberSet Solution

FiberSet provides a managed collection for fibers with automatic lifecycle management and supervision:

```typescript
import { Effect, FiberSet } from "effect"

const processItems = (items: string[]) => Effect.gen(function* () {
  const set = yield* FiberSet.make()
  
  // Fork operations into the managed set
  yield* Effect.forEach(items, (item) => 
    FiberSet.run(set, processItem(item))
  )
  
  // Automatic cleanup when scope closes
  // Proper error propagation
  // Resource management handled automatically
}).pipe(Effect.scoped)
```

### Key Concepts

**FiberSet**: A managed collection of fibers that automatically handles lifecycle, cleanup, and supervision

**Scoped Resource**: FiberSet integrates with Effect's scope system for automatic resource management

**Supervision Strategy**: Controls how errors from child fibers affect the parent and other fibers in the set

**Runtime Integration**: Provides utilities for creating runtime functions backed by fiber sets

## Basic Usage Patterns

### Pattern 1: Creating and Managing a FiberSet

```typescript
import { Effect, FiberSet } from "effect"

// Create a FiberSet within a scope
const basicExample = Effect.gen(function* () {
  const set = yield* FiberSet.make<string, Error>()
  
  // Add some work to the set
  yield* FiberSet.run(set, Effect.succeed("task 1"))
  yield* FiberSet.run(set, Effect.succeed("task 2"))
  yield* FiberSet.run(set, Effect.succeed("task 3"))
  
  // Check current size
  const size = yield* FiberSet.size(set)
  console.log(`Active fibers: ${size}`)
  
  // Wait for all to complete
  yield* FiberSet.awaitEmpty(set)
}).pipe(Effect.scoped) // Automatic cleanup on scope exit
```

### Pattern 2: Fork Operations with Error Handling

```typescript
const processWithErrors = Effect.gen(function* () {
  const set = yield* FiberSet.make<number, string>()
  
  // Fork operations that might fail
  yield* FiberSet.run(set, Effect.succeed(1))
  yield* FiberSet.run(set, Effect.fail("Something went wrong"))
  yield* FiberSet.run(set, Effect.succeed(3))
  
  // Handle the first error that occurs
  const result = yield* FiberSet.join(set).pipe(
    Effect.catchAll((error) => {
      console.log(`First error: ${error}`)
      return Effect.succeed("handled")
    })
  )
  
  return result
}).pipe(Effect.scoped)
```

### Pattern 3: Bulk Operations on Fiber Collections

```typescript
const bulkProcessing = (tasks: Array<Effect.Effect<string, Error>>) => 
  Effect.gen(function* () {
    const set = yield* FiberSet.make<string, Error>()
    
    // Fork all tasks into the set
    yield* Effect.forEach(tasks, (task) => FiberSet.run(set, task))
    
    // Clear all running fibers if needed
    // yield* FiberSet.clear(set) // Interrupts all
    
    // Or wait for completion
    yield* FiberSet.awaitEmpty(set)
    
    console.log("All tasks completed")
  }).pipe(Effect.scoped)
```

## Real-World Examples

### Example 1: HTTP Request Processing Server

A web server needs to handle multiple concurrent requests while managing their lifecycles:

```typescript
import { Effect, FiberSet, Context, Layer } from "effect"

// Request processing service
interface RequestProcessor {
  readonly process: (request: Request) => Effect.Effect<Response, ProcessingError>
}

const RequestProcessor = Context.GenericTag<RequestProcessor>("RequestProcessor")

interface Request {
  readonly id: string
  readonly data: string
}

interface Response {
  readonly id: string
  readonly result: string
}

class ProcessingError {
  readonly _tag = "ProcessingError"
  constructor(readonly message: string, readonly requestId: string) {}
}

// HTTP Server with FiberSet for request management
const makeHttpServer = Effect.gen(function* () {
  const processor = yield* RequestProcessor
  const requestSet = yield* FiberSet.make<Response, ProcessingError>()
  
  const handleRequest = (request: Request) => Effect.gen(function* () {
    console.log(`Starting request ${request.id}`)
    
    // Fork request processing into the managed set
    const fiber = yield* FiberSet.run(
      requestSet, 
      processor.process(request).pipe(
        Effect.tapBoth({
          onFailure: (error) => Effect.log(`Request ${request.id} failed: ${error.message}`),
          onSuccess: (response) => Effect.log(`Request ${request.id} completed`)
        })
      )
    )
    
    return fiber
  })
  
  const shutdown = Effect.gen(function* () {
    yield* Effect.log("Shutting down server...")
    const activeCount = yield* FiberSet.size(requestSet)
    yield* Effect.log(`Interrupting ${activeCount} active requests`)
    // FiberSet automatically interrupts all fibers when scope closes
  })
  
  const getActiveRequestCount = FiberSet.size(requestSet)
  
  return {
    handleRequest,
    shutdown,
    getActiveRequestCount
  } as const
}).pipe(Effect.scoped)

// Usage example
const serverExample = Effect.gen(function* () {
  const server = yield* makeHttpServer
  
  // Simulate incoming requests
  const requests = [
    { id: "req-1", data: "user data 1" },
    { id: "req-2", data: "user data 2" },
    { id: "req-3", data: "user data 3" }
  ]
  
  // Handle requests concurrently
  yield* Effect.forEach(
    requests, 
    (req) => server.handleRequest(req),
    { concurrency: "unbounded" }
  )
  
  // Check active requests
  const active = yield* server.getActiveRequestCount
  console.log(`Active requests: ${active}`)
  
  // Server shutdown handles cleanup automatically
})

// Implementation layer
const RequestProcessorLive = Layer.succeed(RequestProcessor, {
  process: (request) => Effect.gen(function* () {
    // Simulate async processing
    yield* Effect.sleep(Math.random() * 1000)
    
    if (request.data.includes("error")) {
      return yield* Effect.fail(
        new ProcessingError("Invalid data", request.id)
      )
    }
    
    return {
      id: request.id,
      result: `Processed: ${request.data}`
    }
  })
})

const runServerExample = serverExample.pipe(
  Effect.provide(RequestProcessorLive),
  Effect.scoped
)
```

### Example 2: Background Job Processing System

A job processing system that manages worker fibers with different priorities and error handling:

```typescript
import { Effect, FiberSet, Queue, Schedule } from "effect"

interface Job {
  readonly id: string
  readonly type: "email" | "report" | "cleanup"
  readonly payload: unknown
  readonly priority: number
}

interface JobResult {
  readonly jobId: string
  readonly status: "completed" | "failed"
  readonly result?: unknown
  readonly error?: string
}

class JobProcessor {
  constructor(
    private readonly highPrioritySet: FiberSet.FiberSet<JobResult, never>,
    private readonly lowPrioritySet: FiberSet.FiberSet<JobResult, never>
  ) {}
  
  processJob = (job: Job): Effect.Effect<JobResult, never> => {
    const targetSet = job.priority > 5 ? this.highPrioritySet : this.lowPrioritySet
    
    const jobEffect = Effect.gen(function* () {
      yield* Effect.log(`Processing job ${job.id} (${job.type})`)
      
      // Simulate job processing based on type
      switch (job.type) {
        case "email":
          yield* Effect.sleep(100) // Fast
          return { jobId: job.id, status: "completed" as const, result: "Email sent" }
        
        case "report":
          yield* Effect.sleep(5000) // Slow
          return { jobId: job.id, status: "completed" as const, result: "Report generated" }
        
        case "cleanup":
          yield* Effect.sleep(2000) // Medium
          if (Math.random() < 0.3) {
            throw new Error("Cleanup failed")
          }
          return { jobId: job.id, status: "completed" as const, result: "Cleanup done" }
        
        default:
          return { jobId: job.id, status: "failed" as const, error: "Unknown job type" }
      }
    }).pipe(
      Effect.catchAll((error) => 
        Effect.succeed({
          jobId: job.id,
          status: "failed" as const,
          error: error.message
        })
      )
    )
    
    return FiberSet.run(targetSet, jobEffect).pipe(
      Effect.map(() => ({ jobId: job.id, status: "queued" as const }))
    )
  }
  
  getStats = Effect.gen(() => {
    const highPriorityCount = yield* FiberSet.size(this.highPrioritySet)
    const lowPriorityCount = yield* FiberSet.size(this.lowPrioritySet)
    
    return {
      highPriority: highPriorityCount,
      lowPriority: lowPriorityCount,
      total: highPriorityCount + lowPriorityCount
    }
  })
  
  shutdown = Effect.gen(() => {
    yield* Effect.log("Shutting down job processor...")
    const stats = yield* this.getStats
    yield* Effect.log(`Interrupting ${stats.total} active jobs`)
    // Fiber sets will be cleaned up when scope closes
  })
}

const makeJobProcessor = Effect.gen(function* () {
  const highPrioritySet = yield* FiberSet.make<JobResult, never>()
  const lowPrioritySet = yield* FiberSet.make<JobResult, never>()
  
  return new JobProcessor(highPrioritySet, lowPrioritySet)
}).pipe(Effect.scoped)

// Usage example
const jobProcessingExample = Effect.gen(function* () {
  const processor = yield* makeJobProcessor
  
  // Create sample jobs
  const jobs: Job[] = [
    { id: "job-1", type: "email", payload: {}, priority: 8 },
    { id: "job-2", type: "report", payload: {}, priority: 3 },
    { id: "job-3", type: "cleanup", payload: {}, priority: 6 },
    { id: "job-4", type: "email", payload: {}, priority: 9 },
    { id: "job-5", type: "report", payload: {}, priority: 2 }
  ]
  
  // Process jobs
  yield* Effect.forEach(jobs, processor.processJob)
  
  // Monitor progress
  yield* Effect.repeat(
    Effect.gen(function* () {
      const stats = yield* processor.getStats
      yield* Effect.log(`Active jobs - High: ${stats.highPriority}, Low: ${stats.lowPriority}`)
      
      if (stats.total === 0) {
        yield* Effect.log("All jobs completed!")
        return false // Stop repeating
      }
      return true
    }),
    Schedule.fixed(1000)
  ).pipe(
    Effect.race(Effect.sleep(10000)) // Max 10 seconds
  )
}).pipe(Effect.scoped)
```

### Example 3: Real-time Data Pipeline with Backpressure

A data processing pipeline that manages streaming operations with proper backpressure handling:

```typescript
import { Effect, FiberSet, Stream, Queue, Ref } from "effect"

interface DataEvent {
  readonly id: string
  readonly timestamp: number
  readonly data: string
  readonly source: string
}

interface ProcessedEvent {
  readonly originalId: string
  readonly processedAt: number
  readonly result: string
  readonly processingTime: number
}

interface PipelineMetrics {
  readonly processed: number
  readonly failed: number
  readonly activeWorkers: number
}

const makeDataPipeline = (maxConcurrency: number = 10) => Effect.gen(function* () {
  const workerSet = yield* FiberSet.make<ProcessedEvent, never>()
  const metrics = yield* Ref.make<PipelineMetrics>({ 
    processed: 0, 
    failed: 0, 
    activeWorkers: 0 
  })
  
  const processEvent = (event: DataEvent): Effect.Effect<ProcessedEvent, never> => {
    const startTime = Date.now()
    
    return Effect.gen(function* () {
      // Update active worker count
      yield* Ref.update(metrics, m => ({ ...m, activeWorkers: m.activeWorkers + 1 }))
      
      try {
        // Simulate processing based on data complexity
        const processingTime = event.data.length * 10
        yield* Effect.sleep(processingTime)
        
        // Simulate occasional failures
        if (event.data.includes("error")) {
          throw new Error(`Processing failed for ${event.id}`)
        }
        
        const result: ProcessedEvent = {
          originalId: event.id,
          processedAt: Date.now(),
          result: `Processed: ${event.data.toUpperCase()}`,
          processingTime: Date.now() - startTime
        }
        
        yield* Ref.update(metrics, m => ({ ...m, processed: m.processed + 1 }))
        return result
        
      } catch (error) {
        yield* Ref.update(metrics, m => ({ ...m, failed: m.failed + 1 }))
        return {
          originalId: event.id,
          processedAt: Date.now(),
          result: `Failed: ${(error as Error).message}`,
          processingTime: Date.now() - startTime
        }
      }
    }).pipe(
      Effect.ensuring(
        Ref.update(metrics, m => ({ ...m, activeWorkers: m.activeWorkers - 1 }))
      ),
      Effect.catchAll((error) => 
        Effect.succeed({
          originalId: event.id,
          processedAt: Date.now(),
          result: `Error: ${error.message}`,
          processingTime: Date.now() - startTime
        })
      )
    )
  }
  
  const submitEvent = (event: DataEvent): Effect.Effect<void> => Effect.gen(function* () {
    const currentWorkers = yield* FiberSet.size(workerSet)
    
    if (currentWorkers >= maxConcurrency) {
      // Apply backpressure - wait for some workers to complete
      yield* Effect.sleep(100)
      return yield* submitEvent(event) // Retry
    }
    
    // Fork the processing into our managed set
    yield* FiberSet.run(workerSet, processEvent(event))
  })
  
  const processStream = (
    eventStream: Stream.Stream<DataEvent, never>
  ): Effect.Effect<void, never> => 
    Stream.runForEach(eventStream, submitEvent)
  
  const getMetrics = Ref.get(metrics)
  
  const waitForCompletion = FiberSet.awaitEmpty(workerSet)
  
  return {
    submitEvent,
    processStream,
    getMetrics,
    waitForCompletion
  } as const
}).pipe(Effect.scoped)

// Usage example
const pipelineExample = Effect.gen(function* () {
  const pipeline = yield* makeDataPipeline(5) // Max 5 concurrent workers
  
  // Create a stream of events
  const eventStream = Stream.fromIterable([
    { id: "evt-1", timestamp: Date.now(), data: "quick data", source: "api" },
    { id: "evt-2", timestamp: Date.now(), data: "complex data that takes longer", source: "webhook" },
    { id: "evt-3", timestamp: Date.now(), data: "error data", source: "api" },
    { id: "evt-4", timestamp: Date.now(), data: "normal processing", source: "queue" },
    { id: "evt-5", timestamp: Date.now(), data: "batch data processing", source: "file" }
  ]).pipe(
    Stream.schedule(Schedule.spaced(200)) // Emit every 200ms
  )
  
  // Start processing and monitor
  const processingFiber = yield* Effect.fork(pipeline.processStream(eventStream))
  
  // Monitor metrics
  const monitoringFiber = yield* Effect.fork(
    Effect.repeat(
      Effect.gen(function* () {
        const metrics = yield* pipeline.getMetrics
        yield* Effect.log(
          `Metrics - Processed: ${metrics.processed}, Failed: ${metrics.failed}, Active: ${metrics.activeWorkers}`
        )
      }),
      Schedule.fixed(500)
    )
  )
  
  // Wait for stream to complete
  yield* Fiber.join(processingFiber)
  
  // Wait for all workers to finish
  yield* pipeline.waitForCompletion
  
  // Stop monitoring
  yield* Fiber.interrupt(monitoringFiber)
  
  // Final metrics
  const finalMetrics = yield* pipeline.getMetrics
  yield* Effect.log(`Final metrics: ${JSON.stringify(finalMetrics)}`)
}).pipe(Effect.scoped)
```

## Advanced Features Deep Dive

### Feature 1: Runtime Functions

FiberSet provides utilities for creating runtime functions that are backed by managed fiber collections:

#### Basic Runtime Usage

```typescript
import { Effect, FiberSet, Runtime } from "effect"

const runtimeExample = Effect.gen(function* () {
  // Create a runtime function backed by a FiberSet
  const runWithFiberSet = yield* FiberSet.makeRuntime<never, string, Error>()
  
  // Use it to fork effects that are automatically managed
  const fiber1 = runWithFiberSet(Effect.succeed("task 1"))
  const fiber2 = runWithFiberSet(Effect.succeed("task 2"))
  const fiber3 = runWithFiberSet(Effect.fail(new Error("task 3 failed")))
  
  // Fibers are automatically cleaned up when scope closes
  yield* Effect.sleep(1000)
}).pipe(Effect.scoped)
```

#### Real-World Runtime Example: Event Handler System

```typescript
interface EventHandler {
  readonly handleEvent: (event: unknown) => void
}

const makeEventHandler = Effect.gen(function* () {
  // Create a promise-based runtime for handling events
  const runEvent = yield* FiberSet.makeRuntimePromise<never, void, Error>()
  
  const handleEvent = (event: unknown) => {
    // Fork event processing and get a Promise back
    const promise = runEvent(
      Effect.gen(function* () {
        yield* Effect.log(`Processing event: ${JSON.stringify(event)}`)
        yield* Effect.sleep(100) // Simulate async work
        yield* Effect.log("Event processed successfully")
      })
    )
    
    // Handle promise in traditional callback style if needed
    promise.catch((error) => {
      console.error("Event processing failed:", error)
    })
  }
  
  return { handleEvent } satisfies EventHandler
}).pipe(Effect.scoped)
```

### Feature 2: Interruption Propagation Control

Control how interruptions are handled within the fiber set:

#### Basic Interruption Control

```typescript
const interruptionExample = Effect.gen(function* () {
  const set = yield* FiberSet.make<string, never>()
  
  // Fork with interruption propagation disabled
  const fiber1 = yield* FiberSet.run(
    set,
    Effect.never, // This will run indefinitely
    { propagateInterruption: false } // Don't propagate interruptions to the set
  )
  
  // Fork with interruption propagation enabled (default)
  const fiber2 = yield* FiberSet.run(
    set,
    Effect.never,
    { propagateInterruption: true } // Interruptions will affect the set
  )
  
  yield* Effect.sleep(100)
  
  // Interrupting fiber1 won't affect the FiberSet
  yield* Fiber.interrupt(fiber1)
  
  // The set's deferred won't be completed by fiber1's interruption
  const isDone = yield* Deferred.isDone(set.deferred)
  console.log(`Set deferred completed: ${isDone}`) // false
  
  // But interrupting fiber2 will complete the set's deferred
  yield* Fiber.interrupt(fiber2)
}).pipe(Effect.scoped)
```

#### Advanced: Selective Error Propagation

```typescript
const selectiveErrorHandling = Effect.gen(function* () {
  const criticalSet = yield* FiberSet.make<string, Error>()
  const backgroundSet = yield* FiberSet.make<string, Error>()
  
  // Critical tasks - failures should stop everything
  yield* FiberSet.run(
    criticalSet,
    Effect.fail(new Error("Critical failure")),
    { propagateInterruption: true }
  )
  
  // Background tasks - failures should be logged but not propagate
  yield* FiberSet.run(
    backgroundSet,
    Effect.fail(new Error("Background task failed")),
    { propagateInterruption: false }
  )
  
  // Handle critical failures
  const criticalResult = yield* FiberSet.join(criticalSet).pipe(
    Effect.catchAll((error) => {
      console.log(`Critical system failure: ${error.message}`)
      return Effect.succeed("System shutdown initiated")
    })
  )
  
  // Background tasks won't affect the main flow
  yield* FiberSet.awaitEmpty(backgroundSet)
  
  return criticalResult
}).pipe(Effect.scoped)
```

### Feature 3: Direct Fiber Management

For advanced use cases, you can directly add existing fibers to a FiberSet:

#### Adding External Fibers

```typescript
const externalFiberExample = Effect.gen(function* () {
  const set = yield* FiberSet.make<string, Error>()
  
  // Create fibers outside the set
  const externalFiber1 = yield* Effect.fork(
    Effect.gen(function* () {
      yield* Effect.sleep(1000)
      return "External task 1"
    })
  )
  
  const externalFiber2 = yield* Effect.fork(
    Effect.gen(function* () {
      yield* Effect.sleep(2000)
      return "External task 2"
    })
  )
  
  // Add them to the set for management
  yield* FiberSet.add(set, externalFiber1)
  yield* FiberSet.add(set, externalFiber2)
  
  // Now they're managed by the FiberSet
  const size = yield* FiberSet.size(set)
  console.log(`Managing ${size} external fibers`)
  
  // Wait for all to complete
  yield* FiberSet.awaitEmpty(set)
}).pipe(Effect.scoped)
```

## Practical Patterns & Best Practices

### Pattern 1: Graceful Shutdown with Timeout

```typescript
const gracefulShutdown = <A, E>(
  fiberSet: FiberSet.FiberSet<A, E>,
  timeoutMs: number
) => Effect.gen(function* () {
  yield* Effect.log("Initiating graceful shutdown...")
  
  const shutdownResult = yield* FiberSet.awaitEmpty(fiberSet).pipe(
    Effect.timeout(timeoutMs),
    Effect.catchAll(() => {
      return Effect.gen(function* () {
        yield* Effect.log(`Timeout reached, forcing shutdown...`)
        const remaining = yield* FiberSet.size(fiberSet)
        yield* Effect.log(`Interrupting ${remaining} remaining fibers`)
        yield* FiberSet.clear(fiberSet)
        return "forced"
      })
    }),
    Effect.map(() => "graceful")
  )
  
  yield* Effect.log(`Shutdown completed: ${shutdownResult}`)
  return shutdownResult
})

// Usage in a service
const serviceWithGracefulShutdown = Effect.gen(function* () {
  const workerSet = yield* FiberSet.make()
  
  // Start background workers
  yield* Effect.forEach(
    Array.range(1, 5),
    (i) => FiberSet.run(workerSet, longRunningTask(i))
  )
  
  // Register shutdown handler
  yield* Effect.addFinalizer(() => gracefulShutdown(workerSet, 5000))
  
  // Service is ready
  yield* Effect.log("Service started with graceful shutdown support")
  yield* Effect.never // Keep service running
}).pipe(Effect.scoped)
```

### Pattern 2: Circuit Breaker with FiberSet

```typescript
interface CircuitBreakerState {
  readonly failures: number
  readonly lastFailure: number
  readonly state: "closed" | "open" | "half-open"
}

const makeCircuitBreaker = (
  maxFailures: number,
  resetTimeoutMs: number
) => Effect.gen(function* () {
  const state = yield* Ref.make<CircuitBreakerState>({
    failures: 0,
    lastFailure: 0,
    state: "closed"
  })
  
  const workerSet = yield* FiberSet.make<unknown, Error>()
  
  const execute = <A, E>(
    effect: Effect.Effect<A, E>
  ): Effect.Effect<A, E | Error> => Effect.gen(function* () {
    const currentState = yield* Ref.get(state)
    const now = Date.now()
    
    // Check if circuit should be reset
    if (
      currentState.state === "open" && 
      now - currentState.lastFailure > resetTimeoutMs
    ) {
      yield* Ref.set(state, { ...currentState, state: "half-open" })
    }
    
    // Reject if circuit is open
    if (currentState.state === "open") {
      return yield* Effect.fail(new Error("Circuit breaker is open"))
    }
    
    // Execute the effect through the FiberSet
    const result = yield* FiberSet.run(workerSet, effect).pipe(
      Effect.flatMap(Fiber.join),
      Effect.tapBoth({
        onFailure: () => Effect.gen(function* () {
          const current = yield* Ref.get(state)
          const newFailures = current.failures + 1
          
          if (newFailures >= maxFailures) {
            yield* Ref.set(state, {
              failures: newFailures,
              lastFailure: now,
              state: "open"
            })
            yield* Effect.log("Circuit breaker opened")
          } else {
            yield* Ref.update(state, s => ({ ...s, failures: newFailures }))
          }
        }),
        onSuccess: () => {
          // Reset on success
          if (currentState.state === "half-open") {
            return Ref.set(state, { failures: 0, lastFailure: 0, state: "closed" })
          }
          return Effect.void
        }
      })
    )
    
    return result
  })
  
  const getState = Ref.get(state)
  const reset = Ref.set(state, { failures: 0, lastFailure: 0, state: "closed" })
  
  return { execute, getState, reset } as const
}).pipe(Effect.scoped)
```

### Pattern 3: Resource Pool with FiberSet

```typescript
interface PooledResource {
  readonly id: string
  readonly resource: unknown
  readonly inUse: boolean
  readonly createdAt: number
}

const makeResourcePool = <R>(
  createResource: Effect.Effect<R>,
  maxSize: number,
  maxIdleMs: number
) => Effect.gen(function* () {
  const pool = yield* Ref.make<PooledResource[]>([])
  const maintenanceSet = yield* FiberSet.make()
  
  // Start maintenance fiber
  yield* FiberSet.run(maintenanceSet, 
    Effect.repeat(
      Effect.gen(function* () {
        const now = Date.now()
        yield* Ref.update(pool, resources => 
          resources.filter(r => 
            r.inUse || (now - r.createdAt) < maxIdleMs
          )
        )
        yield* Effect.log("Pool maintenance completed")
      }),
      Schedule.fixed(30000) // Run every 30 seconds
    )
  )
  
  const acquire = Effect.gen(function* () {
    const resources = yield* Ref.get(pool)
    const available = resources.find(r => !r.inUse)
    
    if (available) {
      yield* Ref.update(pool, rs => 
        rs.map(r => r.id === available.id ? { ...r, inUse: true } : r)
      )
      return available.resource as R
    }
    
    if (resources.length < maxSize) {
      const resource = yield* createResource
      const pooledResource: PooledResource = {
        id: `resource-${Date.now()}-${Math.random()}`,
        resource,
        inUse: true,
        createdAt: Date.now()
      }
      
      yield* Ref.update(pool, rs => [...rs, pooledResource])
      return resource
    }
    
    // Pool is full, wait and retry
    yield* Effect.sleep(100)
    return yield* acquire
  })
  
  const release = (resource: R) => Effect.gen(function* () {
    yield* Ref.update(pool, resources =>
      resources.map(r => 
        r.resource === resource ? { ...r, inUse: false } : r
      )
    )
  })
  
  const withResource = <A, E>(
    use: (resource: R) => Effect.Effect<A, E>
  ): Effect.Effect<A, E> => Effect.gen(function* () {
    const resource = yield* acquire
    return yield* Effect.ensuring(
      use(resource),
      release(resource)
    )
  })
  
  const getStats = Effect.gen(function* () {
    const resources = yield* Ref.get(pool)
    return {
      total: resources.length,
      available: resources.filter(r => !r.inUse).length,
      inUse: resources.filter(r => r.inUse).length
    }
  })
  
  return { acquire, release, withResource, getStats } as const
}).pipe(Effect.scoped)
```

## Integration Examples

### Integration with HTTP Server

Building on real-world Effect platform usage:

```typescript
import { Effect, FiberSet, Context, Layer } from "effect"

// HTTP request handler with FiberSet management
interface HttpServer {
  readonly start: Effect.Effect<void>
  readonly stop: Effect.Effect<void>
  readonly getActiveConnections: Effect.Effect<number>
}

const HttpServer = Context.GenericTag<HttpServer>("HttpServer")

const makeHttpServer = (port: number) => Effect.gen(function* () {
  const connectionSet = yield* FiberSet.make<void, Error>()
  
  const handleConnection = (connectionId: string) => Effect.gen(function* () {
    yield* Effect.log(`New connection: ${connectionId}`)
    
    // Simulate connection handling
    yield* Effect.sleep(Math.random() * 5000)
    
    yield* Effect.log(`Connection closed: ${connectionId}`)
  })
  
  const start = Effect.gen(function* () {
    yield* Effect.log(`Starting HTTP server on port ${port}`)
    
    // Simulate incoming connections
    yield* Effect.forEach(
      Array.range(1, 10),
      (i) => FiberSet.run(
        connectionSet, 
        handleConnection(`conn-${i}`)
      ),
      { concurrency: "unbounded" }
    )
    
    yield* Effect.log("HTTP server started")
  })
  
  const stop = Effect.gen(function* () {
    yield* Effect.log("Stopping HTTP server...")
    const activeConnections = yield* FiberSet.size(connectionSet)
    yield* Effect.log(`Closing ${activeConnections} active connections`)
    // FiberSet will handle interrupting all connections
  })
  
  const getActiveConnections = FiberSet.size(connectionSet)
  
  return { start, stop, getActiveConnections } satisfies HttpServer
}).pipe(Effect.scoped)

const HttpServerLive = Layer.effect(
  HttpServer,
  makeHttpServer(8080)
)

// Usage
const serverApp = Effect.gen(function* () {
  const server = yield* HttpServer
  
  yield* server.start
  
  // Monitor connections
  yield* Effect.repeat(
    Effect.gen(function* () {
      const connections = yield* server.getActiveConnections
      yield* Effect.log(`Active connections: ${connections}`)
    }),
    Schedule.fixed(1000)
  ).pipe(
    Effect.timeout(10000) // Run for 10 seconds
  )
}).pipe(
  Effect.provide(HttpServerLive),
  Effect.scoped
)
```

### Integration with Streaming Data

```typescript
import { Effect, FiberSet, Stream, Queue } from "effect"

// Stream processor with fiber management
const makeStreamProcessor = <A, B, E>(
  transform: (item: A) => Effect.Effect<B, E>,
  maxConcurrency: number = 10
) => Effect.gen(function* () {
  const processingSet = yield* FiberSet.make<B, E>()
  const resultQueue = yield* Queue.unbounded<B>()
  
  const processStream = (
    input: Stream.Stream<A, E>
  ): Stream.Stream<B, E> => 
    Stream.fromEffect(
      Effect.gen(function* () {
        // Process stream items through FiberSet
        yield* Stream.runForEach(
          input,
          (item) => Effect.gen(function* () {
            const currentSize = yield* FiberSet.size(processingSet)
            
            // Apply backpressure
            if (currentSize >= maxConcurrency) {
              yield* Effect.sleep(10)
            }
            
            // Fork processing
            yield* FiberSet.run(
              processingSet,
              transform(item).pipe(
                Effect.tap((result) => Queue.offer(resultQueue, result))
              )
            )
          })
        )
        
        // Wait for all processing to complete
        yield* FiberSet.awaitEmpty(processingSet)
        yield* Queue.shutdown(resultQueue)
      }).pipe(Effect.fork)
    ).pipe(
      Stream.flatMap(() => Stream.fromQueue(resultQueue))
    )
  
  return { processStream } as const
}).pipe(Effect.scoped)

// Usage example
const streamExample = Effect.gen(function* () {
  const processor = yield* makeStreamProcessor(
    (n: number) => Effect.gen(function* () {
      yield* Effect.sleep(100) // Simulate async processing  
      return n * 2
    }),
    5 // Max 5 concurrent operations
  )
  
  const inputStream = Stream.range(1, 20)
  const outputStream = processor.processStream(inputStream)
  
  const results = yield* Stream.runCollect(outputStream)
  yield* Effect.log(`Processed ${results.length} items`)
  
  return results
}).pipe(Effect.scoped)
```

### Testing Fiber Set Patterns

```typescript
import { Effect, FiberSet, TestClock, TestContext } from "effect"

// Test utilities for FiberSet
const FiberSetTestUtils = {
  // Wait for a specific number of fibers
  waitForFiberCount: <A, E>(
    set: FiberSet.FiberSet<A, E>, 
    expectedCount: number
  ) => Effect.gen(function* () {
    yield* Effect.repeatUntil(
      FiberSet.size(set),
      (size) => size === expectedCount
    )
  }),
  
  // Count fibers with timeout
  getFiberCountWithTimeout: <A, E>(
    set: FiberSet.FiberSet<A, E>,
    timeoutMs: number
  ) => FiberSet.size(set).pipe(
    Effect.timeout(timeoutMs),
    Effect.catchAll(() => Effect.succeed(-1))
  )
}

// Test example
const fiberSetTest = Effect.gen(function* () {
  const set = yield* FiberSet.make<string, Error>()
  
  // Fork some long-running tasks
  yield* FiberSet.run(set, Effect.sleep(1000).pipe(Effect.as("task1")))
  yield* FiberSet.run(set, Effect.sleep(2000).pipe(Effect.as("task2")))
  yield* FiberSet.run(set, Effect.sleep(3000).pipe(Effect.as("task3")))
  
  // Verify initial count
  const initialCount = yield* FiberSet.size(set)
  console.assert(initialCount === 3, "Should have 3 active fibers")
  
  // Advance test clock
  yield* TestClock.adjust(1000)
  
  // Wait for first task to complete
  yield* FiberSetTestUtils.waitForFiberCount(set, 2)
  
  // Advance more
  yield* TestClock.adjust(2000)
  
  // All should be complete
  yield* FiberSet.awaitEmpty(set)
  
  const finalCount = yield* FiberSet.size(set)
  console.assert(finalCount === 0, "All fibers should be complete")
}).pipe(
  Effect.provide(TestContext.TestContext),
  Effect.scoped
)
```

## Conclusion

FiberSet provides **automatic lifecycle management**, **supervision strategies**, and **resource cleanup** for collections of concurrent operations in Effect applications.

Key benefits:
- **Automatic Resource Management**: Fibers are automatically interrupted when scopes close, preventing resource leaks
- **Supervision & Error Handling**: Centralized error handling with configurable propagation strategies
- **Runtime Integration**: Seamlessly integrates with Effect's runtime system for advanced use cases
- **Composability**: Works naturally with other Effect constructs like Scope, Runtime, and Stream

FiberSet is essential when you need to manage multiple concurrent operations with proper cleanup, error handling, and resource management - common in web servers, background job processors, streaming applications, and any system requiring robust concurrency control.