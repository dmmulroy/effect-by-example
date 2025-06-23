# TArray: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem TArray Solves

Traditional shared arrays in concurrent programming are a source of race conditions and data corruption:

```typescript
// Traditional approach - race conditions and data corruption
const sharedArray = [1, 2, 3, 4, 5]

// Multiple concurrent operations
Promise.all([
  Promise.resolve().then(() => sharedArray.push(6)),
  Promise.resolve().then(() => sharedArray.splice(1, 1)),
  Promise.resolve().then(() => sharedArray[0] = 99)
])

// Result is unpredictable - operations may interfere with each other
```

This approach leads to:
- **Race Conditions** - Multiple threads modifying array simultaneously
- **Data Corruption** - Inconsistent state during concurrent operations
- **Complex Locking** - Manual synchronization primitives needed
- **Deadlock Risk** - Complex locking schemes can cause deadlocks

### The TArray Solution

TArray provides a transactional array that guarantees consistency across concurrent operations:

```typescript
import { STM, TArray } from "effect"

// Safe concurrent operations with TArray
const program = STM.gen(function* () {
  const tarray = yield* TArray.fromIterable([1, 2, 3, 4, 5])
  
  // All operations are atomic and consistent
  yield* TArray.update(tarray, 0, (x) => x + 99)
  yield* STM.forEach([6, 7, 8], (item) => TArray.update(tarray, tarray.length, () => item))
  
  return yield* TArray.toArray(tarray)
})
```

### Key Concepts

**Transactional Array**: A mutable array that can only be modified within STM transactions, ensuring atomicity and consistency.

**STM Operations**: All TArray operations return STM effects that must be executed within a transaction context.

**Composability**: TArray operations can be composed with other STM data structures for complex atomic operations.

## Basic Usage Patterns

### Pattern 1: Creating TArrays

```typescript
import { STM, TArray } from "effect"

// From initial values
const fromValues = STM.gen(function* () {
  return yield* TArray.make(1, 2, 3, 4, 5)
})

// From iterable
const fromIterable = STM.gen(function* () {
  const numbers = [10, 20, 30, 40, 50]
  return yield* TArray.fromIterable(numbers)
})

// Empty array
const empty = STM.gen(function* () {
  return yield* TArray.empty<number>()
})
```

### Pattern 2: Basic Array Operations

```typescript
// Reading values
const readOperations = (tarray: TArray.TArray<number>) => STM.gen(function* () {
  const size = yield* TArray.size(tarray)
  const firstElement = yield* TArray.get(tarray, 0)
  const headOption = yield* TArray.headOption(tarray)
  const allElements = yield* TArray.toArray(tarray)
  
  return { size, firstElement, headOption, allElements }
})

// Writing values
const writeOperations = (tarray: TArray.TArray<number>) => STM.gen(function* () {
  yield* TArray.update(tarray, 0, (x) => x * 2)
  yield* TArray.transform(tarray, (arr) => arr.map(x => x + 1))
  
  return yield* TArray.toArray(tarray)
})
```

### Pattern 3: Functional Operations

```typescript
// Search and filter operations
const searchOperations = (tarray: TArray.TArray<number>) => STM.gen(function* () {
  const found = yield* TArray.findFirst(tarray, (x) => x > 10)
  const foundIndex = yield* TArray.findFirstIndex(tarray, (x) => x > 10)
  const count = yield* TArray.count(tarray, (x) => x % 2 === 0)
  const hasEven = yield* TArray.some(tarray, (x) => x % 2 === 0)
  const allPositive = yield* TArray.every(tarray, (x) => x > 0)
  
  return { found, foundIndex, count, hasEven, allPositive }
})
```

## Real-World Examples

### Example 1: Thread-Safe Shopping Cart

```typescript
import { Effect, STM, TArray } from "effect"

interface CartItem {
  readonly id: string
  readonly name: string
  readonly price: number
  readonly quantity: number
}

class ShoppingCart {
  constructor(private items: TArray.TArray<CartItem>) {}

  static make = (): Effect.Effect<ShoppingCart> =>
    STM.gen(function* () {
      const items = yield* TArray.empty<CartItem>()
      return new ShoppingCart(items)
    }).pipe(STM.commit)

  addItem = (item: CartItem): Effect.Effect<void> =>
    STM.gen(function* () {
      const existingIndex = yield* TArray.findFirstIndex(
        this.items,
        (existing) => existing.id === item.id
      )
      
      if (existingIndex._tag === "Some") {
        // Update existing item quantity
        yield* TArray.updateSTM(this.items, existingIndex.value, (existing) =>
          STM.succeed({ ...existing, quantity: existing.quantity + item.quantity })
        )
      } else {
        // Add new item
        const currentSize = yield* TArray.size(this.items)
        yield* TArray.update(this.items, currentSize, () => item)
      }
    }).pipe(STM.commit)

  removeItem = (itemId: string): Effect.Effect<boolean> =>
    STM.gen(function* () {
      const index = yield* TArray.findFirstIndex(
        this.items,
        (item) => item.id === itemId
      )
      
      if (index._tag === "Some") {
        yield* TArray.transform(this.items, (items) =>
          items.filter(item => item.id !== itemId)
        )
        return true
      }
      return false
    }).pipe(STM.commit)

  getTotal = (): Effect.Effect<number> =>
    STM.gen(function* () {
      return yield* TArray.reduce(
        this.items,
        0,
        (acc, item) => acc + (item.price * item.quantity)
      )
    }).pipe(STM.commit)

  getItems = (): Effect.Effect<readonly CartItem[]> =>
    TArray.toArray(this.items).pipe(STM.commit)
}

// Usage
const shoppingExample = Effect.gen(function* () {
  const cart = yield* ShoppingCart.make()
  
  // Multiple concurrent operations
  yield* Effect.all([
    cart.addItem({ id: "1", name: "Laptop", price: 999, quantity: 1 }),
    cart.addItem({ id: "2", name: "Mouse", price: 25, quantity: 2 }),
    cart.addItem({ id: "1", name: "Laptop", price: 999, quantity: 1 }) // Increments quantity
  ], { concurrency: "unbounded" })
  
  const items = yield* cart.getItems()
  const total = yield* cart.getTotal()
  
  return { items, total }
})
```

### Example 2: Concurrent Task Queue Processor

```typescript
import { Effect, STM, TArray, Queue, Fiber } from "effect"

interface Task {
  readonly id: string
  readonly payload: unknown
  readonly priority: number
  readonly createdAt: Date
}

interface TaskResult {
  readonly taskId: string
  readonly result: unknown
  readonly processedAt: Date
  readonly duration: number
}

class TaskProcessor {
  constructor(
    private pendingTasks: TArray.TArray<Task>,
    private completedTasks: TArray.TArray<TaskResult>,
    private failedTasks: TArray.TArray<{ task: Task; error: unknown }>
  ) {}

  static make = (): Effect.Effect<TaskProcessor> =>
    STM.gen(function* () {
      const pending = yield* TArray.empty<Task>()
      const completed = yield* TArray.empty<TaskResult>()
      const failed = yield* TArray.empty<{ task: Task; error: unknown }>()
      return new TaskProcessor(pending, completed, failed)
    }).pipe(STM.commit)

  addTask = (task: Task): Effect.Effect<void> =>
    STM.gen(function* () {
      const size = yield* TArray.size(this.pendingTasks)
      yield* TArray.update(this.pendingTasks, size, () => task)
    }).pipe(STM.commit)

  private processTask = (task: Task): Effect.Effect<TaskResult> =>
    Effect.gen(function* () {
      const startTime = Date.now()
      
      // Simulate task processing
      yield* Effect.sleep("100 millis")
      const result = `Processed task ${task.id} with payload: ${JSON.stringify(task.payload)}`
      
      const endTime = Date.now()
      return {
        taskId: task.id,
        result,
        processedAt: new Date(),
        duration: endTime - startTime
      }
    })

  private processNextTask = (): Effect.Effect<boolean> =>
    STM.gen(function* () {
      const tasks = yield* TArray.toArray(this.pendingTasks)
      
      if (tasks.length === 0) {
        return false
      }
      
      // Find highest priority task
      const sortedTasks = tasks.sort((a, b) => b.priority - a.priority)
      const nextTask = sortedTasks[0]
      
      // Remove task from pending
      yield* TArray.transform(this.pendingTasks, (arr) =>
        arr.filter(t => t.id !== nextTask.id)
      )
      
      return nextTask
    }).pipe(
      STM.commit,
      Effect.flatMap((taskOrFalse) => {
        if (typeof taskOrFalse === "boolean") {
          return Effect.succeed(false)
        }
        
        return this.processTask(taskOrFalse).pipe(
          Effect.flatMap((result) =>
            STM.gen(function* () {
              const size = yield* TArray.size(this.completedTasks)
              yield* TArray.update(this.completedTasks, size, () => result)
            }).pipe(STM.commit)
          ),
          Effect.catchAll((error) =>
            STM.gen(function* () {
              const size = yield* TArray.size(this.failedTasks)
              yield* TArray.update(this.failedTasks, size, () => ({ task: taskOrFalse, error }))
            }).pipe(STM.commit)
          ),
          Effect.as(true)
        )
      })
    )

  startProcessing = (): Effect.Effect<Fiber.RuntimeFiber<never, void>> =>
    Effect.gen(function* () {
      const processLoop = Effect.gen(function* () {
        while (true) {
          const processed = yield* this.processNextTask()
          if (!processed) {
            yield* Effect.sleep("50 millis") // Wait before checking again
          }
        }
      })
      
      return yield* Effect.fork(processLoop)
    })

  getStats = (): Effect.Effect<{
    pending: number
    completed: number
    failed: number
    results: readonly TaskResult[]
  }> =>
    STM.gen(function* () {
      const pending = yield* TArray.size(this.pendingTasks)
      const completed = yield* TArray.size(this.completedTasks)
      const failed = yield* TArray.size(this.failedTasks)
      const results = yield* TArray.toArray(this.completedTasks)
      
      return { pending, completed, failed, results }
    }).pipe(STM.commit)
}

// Usage
const taskProcessingExample = Effect.gen(function* () {
  const processor = yield* TaskProcessor.make()
  
  // Start processing fiber
  const processingFiber = yield* processor.startProcessing()
  
  // Add tasks concurrently
  yield* Effect.all([
    processor.addTask({ id: "1", payload: { data: "first" }, priority: 1, createdAt: new Date() }),
    processor.addTask({ id: "2", payload: { data: "second" }, priority: 3, createdAt: new Date() }),
    processor.addTask({ id: "3", payload: { data: "third" }, priority: 2, createdAt: new Date() })
  ], { concurrency: "unbounded" })
  
  // Wait for processing
  yield* Effect.sleep("1 second")
  
  const stats = yield* processor.getStats()
  
  // Clean up
  yield* Fiber.interrupt(processingFiber)
  
  return stats
})
```

### Example 3: Event Sourcing with TArray

```typescript
import { Effect, STM, TArray, Clock } from "effect"

interface Event {
  readonly id: string
  readonly type: string
  readonly payload: unknown
  readonly timestamp: number
  readonly version: number
}

interface Snapshot<T> {
  readonly data: T
  readonly version: number
  readonly timestamp: number
}

class EventStore<T> {
  constructor(
    private events: TArray.TArray<Event>,
    private snapshots: TArray.TArray<Snapshot<T>>
  ) {}

  static make = <T>(): Effect.Effect<EventStore<T>> =>
    STM.gen(function* () {
      const events = yield* TArray.empty<Event>()
      const snapshots = yield* TArray.empty<Snapshot<T>>()
      return new EventStore(events, snapshots)
    }).pipe(STM.commit)

  appendEvent = (type: string, payload: unknown): Effect.Effect<Event> =>
    Effect.gen(function* () {
      const timestamp = yield* Clock.currentTimeMillis
      const eventId = `${timestamp}-${Math.random().toString(36).substr(2, 9)}`
      
      return yield* STM.gen(function* () {
        const currentVersion = yield* TArray.size(this.events)
        const event: Event = {
          id: eventId,
          type,
          payload,
          timestamp,
          version: currentVersion + 1
        }
        
        const size = yield* TArray.size(this.events)
        yield* TArray.update(this.events, size, () => event)
        return event
      }).pipe(STM.commit)
    })

  getEvents = (fromVersion?: number): Effect.Effect<readonly Event[]> =>
    STM.gen(function* () {
      const allEvents = yield* TArray.toArray(this.events)
      return fromVersion ? allEvents.filter(e => e.version >= fromVersion) : allEvents
    }).pipe(STM.commit)

  createSnapshot = (data: T): Effect.Effect<Snapshot<T>> =>
    Effect.gen(function* () {
      const timestamp = yield* Clock.currentTimeMillis
      
      return yield* STM.gen(function* () {
        const version = yield* TArray.size(this.events)
        const snapshot: Snapshot<T> = { data, version, timestamp }
        
        // Keep only the latest snapshot
        yield* TArray.transform(this.snapshots, () => [snapshot])
        return snapshot
      }).pipe(STM.commit)
    })

  getLatestSnapshot = (): Effect.Effect<Snapshot<T> | null> =>
    STM.gen(function* () {
      const snapshots = yield* TArray.toArray(this.snapshots)
      return snapshots.length > 0 ? snapshots[snapshots.length - 1] : null
    }).pipe(STM.commit)

  replayEvents = <R>(
    reducer: (state: R, event: Event) => R,
    initialState: R,
    fromVersion?: number
  ): Effect.Effect<R> =>
    Effect.gen(function* () {
      const events = yield* this.getEvents(fromVersion)
      return events.reduce(reducer, initialState)
    })
}

// Usage with a simple counter
interface CounterState {
  readonly count: number
  readonly lastOperation: string
}

const eventSourcingExample = Effect.gen(function* () {
  const eventStore = yield* EventStore.make<CounterState>()
  
  // Append events concurrently
  yield* Effect.all([
    eventStore.appendEvent("INCREMENT", { amount: 1 }),
    eventStore.appendEvent("INCREMENT", { amount: 5 }),
    eventStore.appendEvent("DECREMENT", { amount: 2 }),
    eventStore.appendEvent("INCREMENT", { amount: 3 })
  ], { concurrency: "unbounded" })
  
  // Replay events to build current state
  const currentState = yield* eventStore.replayEvents(
    (state: CounterState, event: Event): CounterState => {
      switch (event.type) {
        case "INCREMENT":
          return {
            count: state.count + (event.payload as { amount: number }).amount,
            lastOperation: "increment"
          }
        case "DECREMENT":
          return {
            count: state.count - (event.payload as { amount: number }).amount,
            lastOperation: "decrement"
          }
        default:
          return state
      }
    },
    { count: 0, lastOperation: "none" }
  )
  
  // Create snapshot
  const snapshot = yield* eventStore.createSnapshot(currentState)
  
  const allEvents = yield* eventStore.getEvents()
  
  return { currentState, snapshot, eventCount: allEvents.length }
})
```

## Advanced Features Deep Dive

### Feature 1: STM Transformations and Computations

TArray supports both synchronous and STM-based transformations for complex operations:

#### Basic Transformations

```typescript
const basicTransforms = (tarray: TArray.TArray<number>) => STM.gen(function* () {
  // Transform all elements
  yield* TArray.transform(tarray, (arr) => arr.map(x => x * 2))
  
  // Update specific element
  yield* TArray.update(tarray, 0, (x) => x + 10)
  
  return yield* TArray.toArray(tarray)
})
```

#### STM-based Transformations

```typescript
const stmTransforms = (tarray: TArray.TArray<number>) => STM.gen(function* () {
  // Transform with STM effects
  yield* TArray.transformSTM(tarray, (arr) =>
    STM.succeed(arr.filter(x => x > 0))
  )
  
  // Update with STM computation
  yield* TArray.updateSTM(tarray, 0, (x) =>
    STM.gen(function* () {
      // Complex computation within STM
      const factor = yield* STM.succeed(x > 10 ? 2 : 1)
      return x * factor
    })
  )
  
  return yield* TArray.toArray(tarray)
})
```

#### Advanced Search and Aggregation

```typescript
const advancedOperations = (tarray: TArray.TArray<number>) => STM.gen(function* () {
  // Find with STM computation
  const complexFind = yield* TArray.findFirstSTM(tarray, (x) =>
    STM.gen(function* () {
      // Complex predicate logic
      const threshold = yield* STM.succeed(10)
      return x > threshold && x % 2 === 0
    })
  )
  
  // Reduce with STM
  const stmSum = yield* TArray.reduceSTM(
    tarray,
    STM.succeed(0),
    (acc, value) => STM.succeed(acc + value)
  )
  
  // Count with STM predicate
  const evenCount = yield* TArray.countSTM(tarray, (x) =>
    STM.succeed(x % 2 === 0)
  )
  
  return { complexFind, stmSum, evenCount }
})
```

### Feature 2: Composition with Other STM Data Structures

```typescript
import { STM, TArray, TRef, TMap } from "effect"

const compositeOperations = STM.gen(function* () {
  const tarray = yield* TArray.fromIterable([1, 2, 3, 4, 5])
  const counter = yield* TRef.make(0)
  const cache = yield* TMap.empty<string, number>()
  
  // Atomic operation across multiple STM structures
  yield* TArray.forEach(tarray, (value, index) =>
    STM.gen(function* () {
      // Update counter
      yield* TRef.update(counter, (c) => c + 1)
      
      // Cache the value
      yield* TMap.set(cache, `item-${index}`, value * 2)
      
      // Update array element
      yield* TArray.update(tarray, index, (x) => x + 10)
    })
  )
  
  const finalArray = yield* TArray.toArray(tarray)
  const finalCounter = yield* TRef.get(counter)
  const cacheEntries = yield* TMap.toArray(cache)
  
  return { finalArray, finalCounter, cacheEntries }
})
```

## Practical Patterns & Best Practices

### Pattern 1: Safe Concurrent Modifications

```typescript
// Helper for safe array modifications
const safeArrayOperation = <A, R>(
  tarray: TArray.TArray<A>,
  operation: (arr: readonly A[]) => { newArray: readonly A[]; result: R }
): STM.STM<R> =>
  STM.gen(function* () {
    const currentArray = yield* TArray.toArray(tarray)
    const { newArray, result } = operation(currentArray)
    yield* TArray.transform(tarray, () => newArray)
    return result
  })

// Usage
const safeBulkUpdate = (tarray: TArray.TArray<number>) =>
  safeArrayOperation(tarray, (arr) => ({
    newArray: arr.map(x => x * 2).filter(x => x > 0),
    result: arr.length
  }))
```

### Pattern 2: Batch Operations

```typescript
// Helper for batch operations
const batchOperations = <A>(
  tarray: TArray.TArray<A>,
  operations: readonly ((arr: TArray.TArray<A>) => STM.STM<void>)[]
): STM.STM<void> =>
  STM.gen(function* () {
    for (const operation of operations) {
      yield* operation(tarray)
    }
  })

// Usage
const multipleBatchUpdates = (tarray: TArray.TArray<number>) =>
  batchOperations(tarray, [
    (arr) => TArray.update(arr, 0, (x) => x + 1),
    (arr) => TArray.transform(arr, (a) => a.concat([99])),
    (arr) => TArray.update(arr, 1, (x) => x * 2)
  ])
```

### Pattern 3: Conditional Operations

```typescript
// Helper for conditional array operations
const conditionalUpdate = <A>(
  tarray: TArray.TArray<A>,
  condition: (arr: readonly A[]) => boolean,
  operation: (arr: TArray.TArray<A>) => STM.STM<void>
): STM.STM<boolean> =>
  STM.gen(function* () {
    const current = yield* TArray.toArray(tarray)
    if (condition(current)) {
      yield* operation(tarray)
      return true
    }
    return false
  })

// Usage
const conditionalSort = (tarray: TArray.TArray<number>) =>
  conditionalUpdate(
    tarray,
    (arr) => arr.length > 3,
    (arr) => TArray.transform(arr, (a) => [...a].sort((x, y) => x - y))
  )
```

## Integration Examples

### Integration with Effect Queue

```typescript
import { Effect, STM, TArray, Queue } from "effect"

const integrationWithQueue = Effect.gen(function* () {
  const queue = yield* Queue.bounded<number>(100)
  const tarray = yield* TArray.empty<number>().pipe(STM.commit)
  
  // Producer: Send items to queue and track in TArray
  const producer = Effect.gen(function* () {
    for (let i = 1; i <= 10; i++) {
      yield* Queue.offer(queue, i)
      yield* STM.gen(function* () {
        const size = yield* TArray.size(tarray)
        yield* TArray.update(tarray, size, () => i)
      }).pipe(STM.commit)
      yield* Effect.sleep("100 millis")
    }
  })
  
  // Consumer: Process queue items and update TArray
  const consumer = Effect.gen(function* () {
    while (true) {
      const item = yield* Queue.take(queue)
      yield* STM.gen(function* () {
        const index = yield* TArray.findFirstIndex(tarray, (x) => x === item)
        if (index._tag === "Some") {
          yield* TArray.update(tarray, index.value, (x) => x * 100)
        }
      }).pipe(STM.commit)
    }
  })
  
  // Run producer and consumer concurrently
  yield* Effect.race(
    producer,
    Effect.delay(consumer, "50 millis")
  )
  
  return yield* TArray.toArray(tarray).pipe(STM.commit)
})
```

### Testing Strategies

```typescript
import { Effect, STM, TArray, TestClock, TestContext } from "effect"

// Test utility for TArray operations
const createTestArray = <A>(initialValues: readonly A[]) =>
  TArray.fromIterable(initialValues).pipe(STM.commit)

// Property-based testing helper
const testArrayProperty = <A>(
  property: (arr: TArray.TArray<A>) => STM.STM<boolean>,
  initialValues: readonly A[]
) =>
  Effect.gen(function* () {
    const tarray = yield* createTestArray(initialValues)
    return yield* property(tarray).pipe(STM.commit)
  })

// Example test
const testArrayConsistency = testArrayProperty(
  (tarray) => STM.gen(function* () {
    const initialSize = yield* TArray.size(tarray)
    yield* TArray.update(tarray, 0, (x) => x)
    const finalSize = yield* TArray.size(tarray)
    return initialSize === finalSize
  }),
  [1, 2, 3, 4, 5]
)

// Concurrent test scenario
const testConcurrentOperations = Effect.gen(function* () {
  const tarray = yield* createTestArray([1, 2, 3, 4, 5])
  
  // Run multiple concurrent operations
  yield* Effect.all([
    STM.commit(TArray.update(tarray, 0, (x) => x + 1)),
    STM.commit(TArray.update(tarray, 1, (x) => x + 2)),
    STM.commit(TArray.update(tarray, 2, (x) => x + 3))
  ], { concurrency: "unbounded" })
  
  const result = yield* STM.commit(TArray.toArray(tarray))
  
  // Verify all operations were applied atomically
  return result
}).pipe(
  Effect.provide(TestContext.TestContext)
)
```

## Conclusion

TArray provides thread-safe, transactional array operations for concurrent programming, atomic state management, and composable data transformations.

Key benefits:
- **Atomicity**: All operations are atomic and consistent across concurrent access
- **Composability**: Seamlessly integrates with other STM data structures
- **Type Safety**: Full TypeScript support with type inference and validation

TArray is ideal for scenarios requiring shared mutable arrays in concurrent environments, event sourcing systems, and complex state management where consistency is critical.