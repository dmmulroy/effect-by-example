# TQueue: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem TQueue Solves

In concurrent applications, sharing data between fibers often leads to complex synchronization issues, race conditions, and deadlocks. Traditional approaches using locks and mutable state are error-prone and hard to reason about:

```typescript
// Traditional approach - complex and error-prone
class UnsafeQueue<T> {
  private items: T[] = []
  private mutex = new Mutex()
  
  async enqueue(item: T): Promise<void> {
    await this.mutex.lock()
    try {
      this.items.push(item)
    } finally {
      this.mutex.unlock()
    }
  }
  
  async dequeue(): Promise<T | undefined> {
    await this.mutex.lock()
    try {
      return this.items.shift()
    } finally {
      this.mutex.unlock()
    }
  }
}
```

This approach leads to:
- **Lock contention** - Multiple fibers waiting for the same lock
- **Deadlock risks** - Complex lock ordering can cause system freezes
- **No composability** - Can't combine operations atomically
- **Manual back pressure** - Must implement capacity limits manually

### The TQueue Solution

TQueue provides a transactional queue implementation that works within Software Transactional Memory (STM), offering composable, deadlock-free concurrent operations:

```typescript
import { TQueue, STM, Effect } from "effect"

// Create a bounded queue with automatic back pressure
const program = Effect.gen(function* () {
  const queue = yield* STM.commit(TQueue.bounded<string>(10))
  
  // Producer fiber - automatically blocks when queue is full
  const producer = STM.commit(
    TQueue.offer(queue, "message")
  )
  
  // Consumer fiber - automatically blocks when queue is empty
  const consumer = STM.commit(
    TQueue.take(queue)
  )
  
  return { producer, consumer }
})
```

### Key Concepts

**Transactional Operations**: All TQueue operations run within STM transactions, ensuring atomicity and consistency

**Back Pressure Strategies**: Different queue types handle capacity limits differently (bounded, dropping, sliding)

**Composability**: TQueue operations can be composed with other STM operations in a single transaction

## Basic Usage Patterns

### Creating Different Queue Types

```typescript
import { TQueue, STM, Effect } from "effect"

// Bounded queue - blocks producers when full
const createBoundedQueue = STM.commit(
  TQueue.bounded<string>(100)
)

// Unbounded queue - never blocks producers
const createUnboundedQueue = STM.commit(
  TQueue.unbounded<string>()
)

// Dropping queue - drops new items when full
const createDroppingQueue = STM.commit(
  TQueue.dropping<string>(50)
)

// Sliding queue - drops old items when full
const createSlidingQueue = STM.commit(
  TQueue.sliding<string>(50)
)
```

### Basic Producer/Consumer Pattern

```typescript
import { TQueue, STM, Effect, Fiber } from "effect"

const producerConsumer = Effect.gen(function* () {
  // Create a bounded queue
  const queue = yield* STM.commit(TQueue.bounded<number>(10))
  
  // Producer fiber
  const producer = yield* Effect.fork(
    Effect.forever(
      STM.commit(TQueue.offer(queue, Math.random())).pipe(
        Effect.zipRight(Effect.sleep("100 millis"))
      )
    )
  )
  
  // Consumer fiber
  const consumer = yield* Effect.fork(
    Effect.forever(
      STM.commit(TQueue.take(queue)).pipe(
        Effect.tap(value => Effect.log(`Consumed: ${value}`)),
        Effect.zipRight(Effect.sleep("200 millis"))
      )
    )
  )
  
  // Run for 5 seconds
  yield* Effect.sleep("5 seconds")
  yield* Fiber.interrupt(producer)
  yield* Fiber.interrupt(consumer)
})
```

### Batch Operations

```typescript
import { TQueue, STM, Effect, Array as Arr } from "effect"

const batchOperations = Effect.gen(function* () {
  const queue = yield* STM.commit(TQueue.bounded<string>(100))
  
  // Offer multiple items atomically
  yield* STM.commit(
    TQueue.offerAll(queue, ["item1", "item2", "item3"])
  )
  
  // Take all available items
  const allItems = yield* STM.commit(TQueue.takeAll(queue))
  console.log(`Got ${Arr.length(allItems)} items`)
  
  // Take up to N items
  yield* STM.commit(
    TQueue.offerAll(queue, Arr.range(1, 20).pipe(Arr.map(String)))
  )
  const someitems = yield* STM.commit(TQueue.takeUpTo(queue, 5))
  console.log(`Took ${Arr.length(someitems)} items`)
})
```

## Real-World Examples

### Example 1: Task Processing System

A robust task processing system with multiple workers and priority handling:

```typescript
import { TQueue, STM, Effect, Fiber, Array as Arr, Option } from "effect"

interface Task {
  id: string
  priority: number
  payload: string
}

const taskProcessingSystem = Effect.gen(function* () {
  // High and low priority queues
  const highPriorityQueue = yield* STM.commit(TQueue.bounded<Task>(50))
  const lowPriorityQueue = yield* STM.commit(TQueue.bounded<Task>(100))
  
  // Task distributor
  const distributeTask = (task: Task) =>
    task.priority >= 5
      ? TQueue.offer(highPriorityQueue, task)
      : TQueue.offer(lowPriorityQueue, task)
  
  // Worker that prioritizes high priority tasks
  const createWorker = (workerId: number) =>
    Effect.forever(
      STM.commit(
        STM.gen(function* () {
          // Try high priority first
          const highPriority = yield* TQueue.poll(highPriorityQueue)
          
          const task = Option.isSome(highPriority)
            ? highPriority.value
            : yield* TQueue.take(lowPriorityQueue)
          
          return task
        })
      ).pipe(
        Effect.flatMap(task =>
          Effect.gen(function* () {
            yield* Effect.log(`Worker ${workerId} processing task ${task.id}`)
            // Simulate task processing
            yield* Effect.sleep(`${Math.random() * 1000} millis`)
            yield* Effect.log(`Worker ${workerId} completed task ${task.id}`)
          })
        )
      )
    )
  
  // Create multiple workers
  const workers = yield* Effect.all(
    Arr.range(1, 3).pipe(
      Arr.map(id => Effect.fork(createWorker(id)))
    )
  )
  
  // Submit tasks
  yield* Effect.all(
    Arr.range(1, 20).pipe(
      Arr.map(i => ({
        id: `task-${i}`,
        priority: Math.floor(Math.random() * 10),
        payload: `Task data ${i}`
      })),
      Arr.map(task =>
        STM.commit(distributeTask(task)).pipe(
          Effect.tap(() => Effect.log(`Submitted ${task.id} with priority ${task.priority}`))
        )
      )
    ),
    { concurrency: "unbounded" }
  )
  
  // Let workers process
  yield* Effect.sleep("10 seconds")
  
  // Graceful shutdown
  yield* Effect.all(workers.map(Fiber.interrupt))
})
```

### Example 2: Rate-Limited API Client

Using TQueue to implement rate limiting for API requests:

```typescript
import { TQueue, STM, Effect, Schedule, Fiber, Duration } from "effect"

interface ApiRequest {
  id: string
  url: string
  onComplete: (result: string) => Effect.Effect<void>
}

const rateLimitedApiClient = Effect.gen(function* () {
  // Queue for pending requests
  const requestQueue = yield* STM.commit(TQueue.bounded<ApiRequest>(1000))
  
  // Rate limiter - allows N requests per window
  const createRateLimiter = (requestsPerSecond: number) =>
    Effect.gen(function* () {
      const tokens = yield* STM.commit(TQueue.bounded<void>(requestsPerSecond))
      
      // Fill with initial tokens
      yield* STM.commit(
        STM.forEach(
          Arr.replicate(void 0, requestsPerSecond),
          () => TQueue.offer(tokens, void 0)
        )
      )
      
      // Token refill fiber
      yield* Effect.fork(
        Effect.forever(
          Effect.delay(
            STM.commit(TQueue.offer(tokens, void 0)),
            Duration.millis(1000 / requestsPerSecond)
          )
        )
      )
      
      return tokens
    })
  
  const rateLimiter = yield* createRateLimiter(10) // 10 requests per second
  
  // API worker
  const apiWorker = Effect.forever(
    Effect.gen(function* () {
      // Wait for token and request
      const [_, request] = yield* STM.commit(
        STM.all([
          TQueue.take(rateLimiter),
          TQueue.take(requestQueue)
        ])
      )
      
      // Make API call
      yield* Effect.log(`Processing request ${request.id}`)
      const result = yield* Effect.tryPromise({
        try: () => fetch(request.url).then(r => r.text()),
        catch: (error) => new Error(`API request failed: ${error}`)
      })
      
      // Complete request
      yield* request.onComplete(result)
    })
  )
  
  // Start workers
  const workers = yield* Effect.all(
    Arr.range(1, 5).pipe(
      Arr.map(() => Effect.fork(apiWorker))
    )
  )
  
  // Submit requests
  const submitRequest = (url: string) =>
    Effect.gen(function* () {
      const id = `req-${Math.random()}`
      const result = yield* Effect.Deferred.make<string, Error>()
      
      yield* STM.commit(
        TQueue.offer(requestQueue, {
          id,
          url,
          onComplete: (value) => Effect.Deferred.succeed(result, value)
        })
      )
      
      return yield* Effect.Deferred.await(result)
    })
  
  // Example usage
  const results = yield* Effect.all(
    Arr.range(1, 30).pipe(
      Arr.map(i => submitRequest(`https://api.example.com/data/${i}`))
    ),
    { concurrency: "unbounded" }
  )
  
  yield* Effect.all(workers.map(Fiber.interrupt))
})
```

### Example 3: Event Stream Processing

Building an event streaming system with buffering and batching:

```typescript
import { TQueue, STM, Effect, Stream, Chunk, Fiber } from "effect"

interface Event {
  type: string
  timestamp: number
  data: unknown
}

const eventStreamProcessing = Effect.gen(function* () {
  // Event queue with sliding window
  const eventQueue = yield* STM.commit(TQueue.sliding<Event>(1000))
  
  // Dead letter queue for failed events
  const deadLetterQueue = yield* STM.commit(TQueue.bounded<Event>(100))
  
  // Event producer
  const eventProducer = Effect.fork(
    Effect.forever(
      Effect.gen(function* () {
        const event: Event = {
          type: Math.random() > 0.5 ? "click" : "view",
          timestamp: Date.now(),
          data: { userId: Math.floor(Math.random() * 1000) }
        }
        
        yield* STM.commit(TQueue.offer(eventQueue, event))
        yield* Effect.sleep(`${Math.random() * 100} millis`)
      })
    )
  )
  
  // Create a stream from the queue
  const eventStream = Stream.fromTQueue(eventQueue)
  
  // Process events in batches
  const processedStream = eventStream.pipe(
    Stream.groupedWithin(50, "1 second"),
    Stream.mapEffect(batch =>
      Effect.gen(function* () {
        yield* Effect.log(`Processing batch of ${Chunk.size(batch)} events`)
        
        // Process each event
        const results = yield* Effect.forEach(
          batch,
          (event) =>
            Effect.gen(function* () {
              // Simulate processing that might fail
              if (Math.random() > 0.9) {
                yield* STM.commit(TQueue.offer(deadLetterQueue, event))
                return Effect.fail(`Failed to process event ${event.timestamp}`)
              }
              
              return Effect.succeed({
                ...event,
                processed: true
              })
            }).pipe(
              Effect.either
            ),
          { concurrency: 5 }
        )
        
        const successful = results.filter(result => result._tag === "Right").length
        yield* Effect.log(`Batch complete: ${successful}/${Chunk.size(batch)} succeeded`)
        
        return results
      })
    )
  )
  
  // Start event producer
  const producer = yield* eventProducer
  
  // Run stream processor
  const processor = yield* Effect.fork(
    Stream.runDrain(processedStream)
  )
  
  // Monitor dead letter queue
  const deadLetterMonitor = yield* Effect.fork(
    Effect.forever(
      Effect.gen(function* () {
        const size = yield* STM.commit(TQueue.size(deadLetterQueue))
        if (size > 0) {
          yield* Effect.log(`Dead letter queue size: ${size}`)
        }
        yield* Effect.sleep("5 seconds")
      })
    )
  )
  
  // Run for 30 seconds
  yield* Effect.sleep("30 seconds")
  
  // Shutdown
  yield* Fiber.interrupt(producer)
  yield* STM.commit(TQueue.shutdown(eventQueue))
  yield* Fiber.join(processor)
  yield* Fiber.interrupt(deadLetterMonitor)
  
  // Report dead letters
  const deadLetters = yield* STM.commit(TQueue.takeAll(deadLetterQueue))
  yield* Effect.log(`Total dead letters: ${Arr.length(deadLetters)}`)
})
```

## Advanced Features Deep Dive

### Queue Introspection and Monitoring

TQueue provides several methods to inspect queue state without modifying it:

```typescript
import { TQueue, STM, Effect } from "effect"

const queueMonitoring = Effect.gen(function* () {
  const queue = yield* STM.commit(TQueue.bounded<string>(10))
  
  // Add some items
  yield* STM.commit(
    TQueue.offerAll(queue, ["a", "b", "c"])
  )
  
  // Check queue state
  const state = yield* STM.commit(
    STM.gen(function* () {
      const size = yield* TQueue.size(queue)
      const capacity = yield* TQueue.capacity(queue)
      const isEmpty = yield* TQueue.isEmpty(queue)
      const isFull = yield* TQueue.isFull(queue)
      const isShutdown = yield* TQueue.isShutdown(queue)
      
      // Peek at next item without removing
      const nextItem = yield* TQueue.peekOption(queue)
      
      return {
        size,
        capacity,
        isEmpty,
        isFull,
        isShutdown,
        nextItem
      }
    })
  )
  
  console.log("Queue state:", state)
})
```

### Selective Taking with Seek

Use `seek` to find and take specific items from the queue:

```typescript
import { TQueue, STM, Effect, Option } from "effect"

interface Message {
  id: string
  recipient: string
  content: string
}

const selectiveConsumer = Effect.gen(function* () {
  const queue = yield* STM.commit(TQueue.unbounded<Message>())
  
  // Add messages
  yield* STM.commit(
    TQueue.offerAll(queue, [
      { id: "1", recipient: "alice", content: "Hello Alice" },
      { id: "2", recipient: "bob", content: "Hello Bob" },
      { id: "3", recipient: "alice", content: "How are you?" }
    ])
  )
  
  // Take only messages for Alice
  const aliceMessage = yield* STM.commit(
    TQueue.seek(queue, msg => msg.recipient === "alice")
  )
  
  console.log("Found message for Alice:", aliceMessage)
  
  // Remaining messages
  const remaining = yield* STM.commit(TQueue.takeAll(queue))
  console.log("Remaining messages:", remaining)
})
```

### Transactional Composition

Compose TQueue operations with other STM operations atomically:

```typescript
import { TQueue, TRef, STM, Effect } from "effect"

interface Job {
  id: string
  status: "pending" | "processing" | "completed"
}

const transactionalJobSystem = Effect.gen(function* () {
  const jobQueue = yield* STM.commit(TQueue.bounded<Job>(100))
  const processingCount = yield* STM.commit(TRef.make(0))
  const completedCount = yield* STM.commit(TRef.make(0))
  
  // Atomically take job and update counters
  const takeJobForProcessing = STM.gen(function* () {
    const job = yield* TQueue.take(jobQueue)
    yield* TRef.update(processingCount, n => n + 1)
    return { ...job, status: "processing" as const }
  })
  
  // Atomically complete job and update counters
  const completeJob = (job: Job) =>
    STM.gen(function* () {
      yield* TRef.update(processingCount, n => n - 1)
      yield* TRef.update(completedCount, n => n + 1)
      return { ...job, status: "completed" as const }
    })
  
  // Worker process
  const worker = Effect.forever(
    STM.commit(takeJobForProcessing).pipe(
      Effect.flatMap(job =>
        Effect.gen(function* () {
          yield* Effect.log(`Processing job ${job.id}`)
          yield* Effect.sleep("1 second")
          const completed = yield* STM.commit(completeJob(job))
          yield* Effect.log(`Completed job ${completed.id}`)
        })
      )
    )
  )
  
  // Start workers
  const workers = yield* Effect.all(
    Arr.range(1, 3).pipe(
      Arr.map(() => Effect.fork(worker))
    )
  )
  
  // Submit jobs
  yield* Effect.all(
    Arr.range(1, 10).pipe(
      Arr.map(i => ({
        id: `job-${i}`,
        status: "pending" as const
      })),
      Arr.map(job => STM.commit(TQueue.offer(jobQueue, job)))
    )
  )
  
  // Monitor progress
  const monitor = yield* Effect.fork(
    Effect.repeat(
      STM.commit(
        STM.gen(function* () {
          const processing = yield* TRef.get(processingCount)
          const completed = yield* TRef.get(completedCount)
          const queued = yield* TQueue.size(jobQueue)
          return { queued, processing, completed }
        })
      ).pipe(
        Effect.tap(stats => Effect.log(`Stats: ${JSON.stringify(stats)}`))
      ),
      Schedule.spaced("2 seconds")
    )
  )
  
  // Run for a while
  yield* Effect.sleep("15 seconds")
  
  // Cleanup
  yield* Effect.all(workers.map(Fiber.interrupt))
  yield* Fiber.interrupt(monitor)
})
```

## Practical Patterns & Best Practices

### Pattern 1: Priority Queue Implementation

Build a priority queue system using multiple TQueues:

```typescript
import { TQueue, STM, Effect, Option, Array as Arr } from "effect"

interface PriorityQueue<A> {
  offer: (item: A, priority: number) => STM.STM<void>
  take: STM.STM<A>
  takeAll: STM.STM<Array<A>>
}

const makePriorityQueue = <A>(levels: number): Effect.Effect<PriorityQueue<A>> =>
  Effect.gen(function* () {
    // Create a queue for each priority level
    const queues = yield* Effect.all(
      Arr.range(0, levels - 1).pipe(
        Arr.map(() => STM.commit(TQueue.unbounded<A>()))
      )
    )
    
    const offer = (item: A, priority: number): STM.STM<void> => {
      const level = Math.max(0, Math.min(levels - 1, priority))
      return TQueue.offer(queues[level], item)
    }
    
    const take: STM.STM<A> = STM.gen(function* () {
      // Try queues in priority order
      for (let i = levels - 1; i >= 0; i--) {
        const item = yield* TQueue.poll(queues[i])
        if (Option.isSome(item)) {
          return item.value
        }
      }
      
      // If all empty, block on highest priority
      return yield* TQueue.take(queues[levels - 1])
    })
    
    const takeAll: STM.STM<Array<A>> = STM.gen(function* () {
      const results: A[] = []
      
      // Collect from all queues in priority order
      for (let i = levels - 1; i >= 0; i--) {
        const items = yield* TQueue.takeAll(queues[i])
        results.push(...items)
      }
      
      return results
    })
    
    return { offer, take, takeAll }
  })

// Usage example
const priorityExample = Effect.gen(function* () {
  const pq = yield* makePriorityQueue<string>(3)
  
  // Add items with different priorities
  yield* STM.commit(
    STM.all([
      pq.offer("low priority", 0),
      pq.offer("high priority", 2),
      pq.offer("medium priority", 1),
      pq.offer("another high", 2)
    ])
  )
  
  // Take items - high priority first
  const item1 = yield* STM.commit(pq.take)
  const item2 = yield* STM.commit(pq.take)
  
  console.log("First:", item1)  // "high priority"
  console.log("Second:", item2) // "another high"
})
```

### Pattern 2: Circuit Breaker Queue

Implement circuit breaker pattern with TQueue:

```typescript
import { TQueue, TRef, STM, Effect, Duration } from "effect"

interface CircuitBreakerQueue<A> {
  offer: (item: A) => Effect.Effect<boolean>
  take: Effect.Effect<A>
  getState: STM.STM<"closed" | "open" | "half-open">
}

const makeCircuitBreakerQueue = <A>(
  capacity: number,
  failureThreshold: number,
  resetTimeout: Duration.Duration
): Effect.Effect<CircuitBreakerQueue<A>> =>
  Effect.gen(function* () {
    const queue = yield* STM.commit(TQueue.bounded<A>(capacity))
    const failureCount = yield* STM.commit(TRef.make(0))
    const state = yield* STM.commit(TRef.make<"closed" | "open" | "half-open">("closed"))
    const lastFailureTime = yield* STM.commit(TRef.make(0))
    
    const checkAndUpdateState = STM.gen(function* () {
      const currentState = yield* TRef.get(state)
      const failures = yield* TRef.get(failureCount)
      const lastFailure = yield* TRef.get(lastFailureTime)
      const now = Date.now()
      
      if (currentState === "open") {
        const elapsed = now - lastFailure
        if (elapsed > Duration.toMillis(resetTimeout)) {
          yield* TRef.set(state, "half-open")
          yield* TRef.set(failureCount, 0)
        }
      } else if (failures >= failureThreshold) {
        yield* TRef.set(state, "open")
        yield* TRef.set(lastFailureTime, now)
      }
      
      return yield* TRef.get(state)
    })
    
    const offer = (item: A): Effect.Effect<boolean> =>
      STM.commit(
        STM.gen(function* () {
          const currentState = yield* checkAndUpdateState
          
          if (currentState === "open") {
            return false
          }
          
          yield* TQueue.offer(queue, item)
          
          if (currentState === "half-open") {
            yield* TRef.set(state, "closed")
          }
          
          return true
        })
      )
    
    const take: Effect.Effect<A> =
      STM.commit(TQueue.take(queue)).pipe(
        Effect.tapError(() =>
          STM.commit(
            TRef.update(failureCount, n => n + 1)
          )
        )
      )
    
    const getState = TRef.get(state)
    
    return { offer, take, getState }
  })
```

## Integration Examples

### Integration with Stream

Convert between TQueue and Stream for powerful data processing:

```typescript
import { TQueue, Stream, STM, Effect, Chunk } from "effect"

const streamIntegration = Effect.gen(function* () {
  // Create source queue
  const sourceQueue = yield* STM.commit(TQueue.unbounded<number>())
  
  // Producer
  yield* Effect.fork(
    Effect.forever(
      STM.commit(TQueue.offer(sourceQueue, Math.random())).pipe(
        Effect.zipRight(Effect.sleep("100 millis"))
      )
    )
  )
  
  // Process with Stream
  const processedStream = Stream.fromTQueue(sourceQueue).pipe(
    Stream.map(n => n * 100),
    Stream.filter(n => n > 50),
    Stream.groupedWithin(10, "1 second"),
    Stream.mapConcatChunk(chunk =>
      Chunk.map(chunk, n => Math.floor(n))
    )
  )
  
  // Sink to another queue
  const sinkQueue = yield* STM.commit(TQueue.unbounded<number>())
  
  yield* Effect.fork(
    processedStream.pipe(
      Stream.run(
        Stream.toTQueue(sinkQueue)
      )
    )
  )
  
  // Consumer
  yield* Effect.repeat(
    STM.commit(TQueue.take(sinkQueue)).pipe(
      Effect.tap(value => Effect.log(`Processed value: ${value}`))
    ),
    { times: 20 }
  )
})
```

### Testing Strategies

Test TQueue-based systems with deterministic STM:

```typescript
import { TQueue, STM, Effect, TestClock, Fiber } from "effect"

describe("TQueue", () => {
  it("should handle back pressure correctly", () =>
    Effect.gen(function* () {
      const queue = yield* STM.commit(TQueue.bounded<number>(2))
      
      // Fill the queue
      yield* STM.commit(TQueue.offerAll(queue, [1, 2]))
      
      // This should block
      const blockedFiber = yield* Effect.fork(
        STM.commit(TQueue.offer(queue, 3))
      )
      
      // Check fiber is suspended
      yield* TestClock.adjust("100 millis")
      const isBlocked = yield* Fiber.poll(blockedFiber).pipe(
        Effect.map(Option.isNone)
      )
      
      expect(isBlocked).toBe(true)
      
      // Take one item to unblock
      yield* STM.commit(TQueue.take(queue))
      
      // Now the blocked offer should complete
      yield* Fiber.join(blockedFiber)
      
      const size = yield* STM.commit(TQueue.size(queue))
      expect(size).toBe(2)
    }).pipe(Effect.runPromise))
  
  it("should maintain FIFO order", () =>
    Effect.gen(function* () {
      const queue = yield* STM.commit(TQueue.unbounded<string>())
      
      // Add items
      const items = ["first", "second", "third"]
      yield* STM.commit(TQueue.offerAll(queue, items))
      
      // Take items
      const taken = yield* STM.commit(
        STM.forEach(items, () => TQueue.take(queue))
      )
      
      expect(taken).toEqual(items)
    }).pipe(Effect.runPromise))
})
```

## Conclusion

TQueue provides a powerful, composable solution for concurrent queue operations in Effect applications. Its integration with Software Transactional Memory ensures thread-safe operations without explicit locking.

Key benefits:
- **Composability**: Combine queue operations with other STM operations atomically
- **Type Safety**: Full type inference and compile-time guarantees
- **Back Pressure**: Built-in strategies for handling queue capacity
- **Deadlock Freedom**: STM ensures no deadlocks in concurrent operations

Use TQueue when you need thread-safe queues for producer-consumer patterns, work distribution, event streaming, or any concurrent data sharing scenario.