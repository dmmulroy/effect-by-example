# TPriorityQueue: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem TPriorityQueue Solves

Traditional priority queues in concurrent programming suffer from race conditions and inconsistent ordering:

```typescript
// Traditional approach - race conditions and lost priorities
class UnsafePriorityQueue<T> {
  private items: Array<{ value: T; priority: number }> = []
  
  enqueue(value: T, priority: number) {
    this.items.push({ value, priority })
    this.items.sort((a, b) => b.priority - a.priority) // Not atomic!
  }
  
  dequeue(): T | undefined {
    return this.items.shift()?.value // Race condition with enqueue!
  }
}

// Multiple concurrent operations
const queue = new UnsafePriorityQueue<string>()
Promise.all([
  Promise.resolve().then(() => queue.enqueue("low", 1)),
  Promise.resolve().then(() => queue.enqueue("high", 10)),
  Promise.resolve().then(() => queue.dequeue())
])
// Result is unpredictable - operations may interfere with each other
```

This approach leads to:
- **Race Conditions** - Concurrent enqueue/dequeue operations interfering
- **Lost Priorities** - Items inserted during sort operations may be misplaced
- **Inconsistent Ordering** - Priority order not maintained under concurrent access
- **Complex Locking** - Manual synchronization adding complexity and deadlock risk

### The TPriorityQueue Solution

TPriorityQueue provides a transactional priority queue that maintains ordering atomically:

```typescript
import { STM, TPriorityQueue, Order } from "effect"

// Safe concurrent operations with TPriorityQueue
const program = STM.gen(function* () {
  const queue = yield* TPriorityQueue.make(Order.number)("low", "high", "medium")
  
  // All operations maintain priority order atomically
  yield* TPriorityQueue.offer(queue, "urgent")
  yield* TPriorityQueue.offer(queue, "normal")
  
  const highestPriority = yield* TPriorityQueue.take(queue)
  const nextHighest = yield* TPriorityQueue.peek(queue)
  
  return { highestPriority, nextHighest }
})
```

### Key Concepts

**Priority Ordering**: TPriorityQueue uses an Order instance to determine element priority, ensuring consistent ordering.

**Transactional Operations**: All operations are atomic within STM transactions, preventing race conditions.

**Efficient Implementation**: Maintains heap-based priority ordering for optimal performance.

## Basic Usage Patterns

### Pattern 1: Creating TPriorityQueues

```typescript
import { STM, TPriorityQueue, Order } from "effect"

// Numeric priority queue (higher numbers = higher priority)
const createNumericQueue = STM.gen(function* () {
  return yield* TPriorityQueue.make(Order.number)(10, 5, 15, 1)
})

// String priority queue (alphabetical order)
const createStringQueue = STM.gen(function* () {
  return yield* TPriorityQueue.make(Order.string)("zebra", "apple", "banana")
})

// Custom priority queue with objects
interface Task {
  readonly name: string
  readonly priority: number
  readonly deadline: Date
}

const taskOrder = Order.make<Task>((a, b) => {
  // Higher priority number = higher priority
  // Earlier deadline = higher priority (if same priority)
  if (a.priority !== b.priority) {
    return b.priority - a.priority
  }
  return a.deadline.getTime() - b.deadline.getTime()
})

const createTaskQueue = STM.gen(function* () {
  return yield* TPriorityQueue.make(taskOrder)()
})

// Empty queue
const createEmpty = STM.gen(function* () {
  return yield* TPriorityQueue.empty<number>(Order.number)
})
```

### Pattern 2: Basic Queue Operations

```typescript
// Adding elements
const addOperations = (queue: TPriorityQueue.TPriorityQueue<number>) => STM.gen(function* () {
  yield* TPriorityQueue.offer(queue, 42)
  yield* TPriorityQueue.offerAll(queue, [1, 2, 3, 10, 5])
  
  return yield* TPriorityQueue.size(queue)
})

// Retrieving elements
const retrieveOperations = (queue: TPriorityQueue.TPriorityQueue<number>) => STM.gen(function* () {
  const highest = yield* TPriorityQueue.take(queue) // Removes highest priority
  const nextHighest = yield* TPriorityQueue.peek(queue) // Peeks without removing
  const peekOption = yield* TPriorityQueue.peekOption(queue) // Safe peek
  const takeOption = yield* TPriorityQueue.takeOption(queue) // Safe take
  
  return { highest, nextHighest, peekOption, takeOption }
})

// Bulk operations
const bulkOperations = (queue: TPriorityQueue.TPriorityQueue<number>) => STM.gen(function* () {
  const allItems = yield* TPriorityQueue.takeAll(queue) // Removes all items
  const upToFive = yield* TPriorityQueue.takeUpTo(queue, 5) // Takes up to 5 items
  const asArray = yield* TPriorityQueue.toArray(queue) // Non-destructive conversion
  
  return { allItems, upToFive, asArray }
})
```

### Pattern 3: Queue State and Filtering

```typescript
// State checking
const stateOperations = (queue: TPriorityQueue.TPriorityQueue<number>) => STM.gen(function* () {
  const isEmpty = yield* TPriorityQueue.isEmpty(queue)
  const isNonEmpty = yield* TPriorityQueue.isNonEmpty(queue)
  const size = yield* TPriorityQueue.size(queue)
  
  return { isEmpty, isNonEmpty, size }
})

// Filtering operations
const filterOperations = (queue: TPriorityQueue.TPriorityQueue<number>) => STM.gen(function* () {
  // Remove items matching predicate
  yield* TPriorityQueue.removeIf(queue, (item) => item < 5)
  
  // Keep only items matching predicate
  yield* TPriorityQueue.retainIf(queue, (item) => item % 2 === 0)
  
  return yield* TPriorityQueue.toArray(queue)
})
```

## Real-World Examples

### Example 1: Task Scheduling System

```typescript
import { Effect, STM, TPriorityQueue, TRef, Order, Clock, Schedule } from "effect"

interface ScheduledTask {
  readonly id: string
  readonly name: string
  readonly priority: number
  readonly deadline: Date
  readonly work: () => Effect.Effect<unknown>
  readonly retryCount: number
  readonly maxRetries: number
}

interface TaskResult {
  readonly taskId: string
  readonly status: "completed" | "failed" | "retried"
  readonly result?: unknown
  readonly error?: unknown
  readonly completedAt: Date
}

class TaskScheduler {
  constructor(
    private pendingQueue: TPriorityQueue.TPriorityQueue<ScheduledTask>,
    private completedTasks: TRef.TRef<readonly TaskResult[]>,
    private isRunning: TRef.TRef<boolean>
  ) {}

  static make = (): Effect.Effect<TaskScheduler> => {
    const taskOrder = Order.make<ScheduledTask>((a, b) => {
      // Higher priority first
      if (a.priority !== b.priority) {
        return b.priority - a.priority
      }
      // Earlier deadline first (if same priority)
      return a.deadline.getTime() - b.deadline.getTime()
    })

    return STM.gen(function* () {
      const pendingQueue = yield* TPriorityQueue.empty<ScheduledTask>(taskOrder)
      const completedTasks = yield* TRef.make<readonly TaskResult[]>([])
      const isRunning = yield* TRef.make(false)
      
      return new TaskScheduler(pendingQueue, completedTasks, isRunning)
    }).pipe(STM.commit)
  }

  scheduleTask = (
    id: string,
    name: string,
    priority: number,
    deadline: Date,
    work: () => Effect.Effect<unknown>,
    maxRetries: number = 3
  ): Effect.Effect<void> => {
    const task: ScheduledTask = {
      id,
      name,
      priority,
      deadline,
      work,
      retryCount: 0,
      maxRetries
    }

    return STM.commit(TPriorityQueue.offer(this.pendingQueue, task))
  }

  private processNextTask = (): Effect.Effect<boolean> =>
    STM.gen(function* () {
      const isEmpty = yield* TPriorityQueue.isEmpty(this.pendingQueue)
      if (isEmpty) {
        return false
      }
      
      const task = yield* TPriorityQueue.take(this.pendingQueue)
      return task
    }).pipe(
      STM.commit,
      Effect.flatMap((taskOrFalse) => {
        if (typeof taskOrFalse === "boolean") {
          return Effect.succeed(false)
        }
        
        return this.executeTask(taskOrFalse).pipe(Effect.as(true))
      })
    )

  private executeTask = (task: ScheduledTask): Effect.Effect<void> =>
    Effect.gen(function* () {
      const startTime = new Date()
      
      const result = yield* task.work().pipe(
        Effect.timeout("30 seconds"),
        Effect.either
      )
      
      if (result._tag === "Left") {
        // Task failed, check if we should retry
        if (task.retryCount < task.maxRetries) {
          const retryTask: ScheduledTask = {
            ...task,
            retryCount: task.retryCount + 1,
            // Lower priority for retries to avoid blocking new tasks
            priority: Math.max(1, task.priority - 1)
          }
          
          yield* STM.commit(TPriorityQueue.offer(this.pendingQueue, retryTask))
          
          const retryResult: TaskResult = {
            taskId: task.id,
            status: "retried",
            error: result.left,
            completedAt: new Date()
          }
          
          yield* STM.commit(
            TRef.update(this.completedTasks, (tasks) => [...tasks, retryResult])
          )
        } else {
          // Max retries exceeded
          const failedResult: TaskResult = {
            taskId: task.id,
            status: "failed",
            error: result.left,
            completedAt: new Date()
          }
          
          yield* STM.commit(
            TRef.update(this.completedTasks, (tasks) => [...tasks, failedResult])
          )
        }
      } else {
        // Task succeeded
        const successResult: TaskResult = {
          taskId: task.id,
          status: "completed",
          result: result.right,
          completedAt: new Date()
        }
        
        yield* STM.commit(
          TRef.update(this.completedTasks, (tasks) => [...tasks, successResult])
        )
      }
    })

  start = (): Effect.Effect<void> =>
    STM.gen(function* () {
      const running = yield* TRef.get(this.isRunning)
      if (running) {
        return // Already running
      }
      
      yield* TRef.set(this.isRunning, true)
    }).pipe(
      STM.commit,
      Effect.flatMap(() =>
        Effect.gen(function* () {
          while (true) {
            const running = yield* STM.commit(TRef.get(this.isRunning))
            if (!running) break
            
            const processed = yield* this.processNextTask()
            if (!processed) {
              yield* Effect.sleep("100 millis") // Wait before checking again
            }
          }
        }).pipe(Effect.fork, Effect.asVoid)
      )
    )

  stop = (): Effect.Effect<void> =>
    STM.commit(TRef.set(this.isRunning, false))

  getQueueStats = (): Effect.Effect<{
    pendingTasks: number
    completedTasks: number
    nextTask: ScheduledTask | null
  }> =>
    STM.gen(function* () {
      const pendingTasks = yield* TPriorityQueue.size(this.pendingQueue)
      const completedResults = yield* TRef.get(this.completedTasks)
      const nextTask = yield* TPriorityQueue.peekOption(this.pendingQueue).pipe(
        STM.map((option) => option._tag === "Some" ? option.value : null)
      )
      
      return {
        pendingTasks,
        completedTasks: completedResults.length,
        nextTask
      }
    }).pipe(STM.commit)

  getCompletedTasks = (): Effect.Effect<readonly TaskResult[]> =>
    STM.commit(TRef.get(this.completedTasks))
}

// Usage
const taskSchedulingExample = Effect.gen(function* () {
  const scheduler = yield* TaskScheduler.make()
  
  // Start the scheduler
  yield* scheduler.start()
  
  // Schedule various tasks with different priorities and deadlines
  yield* scheduler.scheduleTask(
    "task-1",
    "High Priority Task",
    10,
    new Date(Date.now() + 5000),
    () => Effect.gen(function* () {
      yield* Effect.sleep("200 millis")
      return "High priority result"
    })
  )
  
  yield* scheduler.scheduleTask(
    "task-2",
    "Low Priority Task",
    1,
    new Date(Date.now() + 10000),
    () => Effect.gen(function* () {
      yield* Effect.sleep("100 millis")
      return "Low priority result"
    })
  )
  
  yield* scheduler.scheduleTask(
    "task-3",
    "Urgent Task",
    15,
    new Date(Date.now() + 1000),
    () => Effect.gen(function* () {
      yield* Effect.sleep("50 millis")
      return "Urgent result"
    })
  )
  
  yield* scheduler.scheduleTask(
    "task-4",
    "Failing Task",
    5,
    new Date(Date.now() + 3000),
    () => Effect.fail(new Error("This task always fails")),
    2 // Max 2 retries
  )
  
  // Wait for tasks to process
  yield* Effect.sleep("3 seconds")
  
  const stats = yield* scheduler.getQueueStats()
  const completedTasks = yield* scheduler.getCompletedTasks()
  
  yield* scheduler.stop()
  
  return { stats, completedTasks }
})
```

### Example 2: Event Processing Pipeline

```typescript
import { Effect, STM, TPriorityQueue, TRef, Order, Chunk } from "effect"

interface Event {
  readonly id: string
  readonly type: string
  readonly priority: number
  readonly timestamp: number
  readonly payload: unknown
  readonly source: string
}

interface ProcessedEvent {
  readonly originalEvent: Event
  readonly processedAt: number
  readonly processingTimeMs: number
  readonly result: unknown
}

interface EventProcessor {
  readonly name: string
  readonly canProcess: (event: Event) => boolean
  readonly process: (event: Event) => Effect.Effect<unknown>
  readonly priority: number
}

class EventProcessingPipeline {
  constructor(
    private eventQueue: TPriorityQueue.TPriorityQueue<Event>,
    private processors: readonly EventProcessor[],
    private processedEvents: TRef.TRef<readonly ProcessedEvent[]>,
    private deadLetterQueue: TPriorityQueue.TPriorityQueue<Event>
  ) {}

  static make = (processors: readonly EventProcessor[]): Effect.Effect<EventProcessingPipeline> => {
    const eventOrder = Order.make<Event>((a, b) => {
      // Higher priority first
      if (a.priority !== b.priority) {
        return b.priority - a.priority
      }
      // Earlier timestamp first (if same priority)
      return a.timestamp - b.timestamp
    })

    return STM.gen(function* () {
      const eventQueue = yield* TPriorityQueue.empty<Event>(eventOrder)
      const deadLetterQueue = yield* TPriorityQueue.empty<Event>(eventOrder)
      const processedEvents = yield* TRef.make<readonly ProcessedEvent[]>([])
      
      return new EventProcessingPipeline(
        eventQueue,
        processors,
        processedEvents,
        deadLetterQueue
      )
    }).pipe(STM.commit)
  }

  publishEvent = (event: Event): Effect.Effect<void> =>
    STM.commit(TPriorityQueue.offer(this.eventQueue, event))

  publishEvents = (events: readonly Event[]): Effect.Effect<void> =>
    STM.commit(TPriorityQueue.offerAll(this.eventQueue, events))

  private findProcessor = (event: Event): EventProcessor | null => {
    // Find the highest priority processor that can handle this event
    const suitableProcessors = this.processors
      .filter(p => p.canProcess(event))
      .sort((a, b) => b.priority - a.priority)
    
    return suitableProcessors[0] || null
  }

  private processEvent = (event: Event): Effect.Effect<void> =>
    Effect.gen(function* () {
      const processor = this.findProcessor(event)
      
      if (!processor) {
        // No processor can handle this event, send to dead letter queue
        yield* STM.commit(TPriorityQueue.offer(this.deadLetterQueue, event))
        return
      }
      
      const startTime = Date.now()
      
      const result = yield* processor.process(event).pipe(
        Effect.timeout("10 seconds"),
        Effect.either
      )
      
      const endTime = Date.now()
      
      if (result._tag === "Left") {
        // Processing failed, send to dead letter queue
        yield* STM.commit(TPriorityQueue.offer(this.deadLetterQueue, event))
      } else {
        // Processing succeeded
        const processedEvent: ProcessedEvent = {
          originalEvent: event,
          processedAt: endTime,
          processingTimeMs: endTime - startTime,
          result: result.right
        }
        
        yield* STM.commit(
          TRef.update(this.processedEvents, (events) => [...events, processedEvent])
        )
      }
    })

  processNextBatch = (batchSize: number = 10): Effect.Effect<number> =>
    STM.gen(function* () {
      const events = yield* TPriorityQueue.takeUpTo(this.eventQueue, batchSize)
      return events
    }).pipe(
      STM.commit,
      Effect.flatMap((events) =>
        Effect.gen(function* () {
          if (events.length === 0) {
            return 0
          }
          
          // Process events concurrently
          yield* Effect.all(
            events.map(event => this.processEvent(event)),
            { concurrency: 5 }
          )
          
          return events.length
        })
      )
    )

  drainQueue = (): Effect.Effect<number> =>
    Effect.gen(function* () {
      let totalProcessed = 0
      
      while (true) {
        const processed = yield* this.processNextBatch(50)
        if (processed === 0) break
        totalProcessed += processed
      }
      
      return totalProcessed
    })

  getStats = (): Effect.Effect<{
    pendingEvents: number
    processedEvents: number
    deadLetterEvents: number
    nextEvent: Event | null
  }> =>
    STM.gen(function* () {
      const pendingEvents = yield* TPriorityQueue.size(this.eventQueue)
      const deadLetterEvents = yield* TPriorityQueue.size(this.deadLetterQueue)
      const processedEventsArray = yield* TRef.get(this.processedEvents)
      const nextEvent = yield* TPriorityQueue.peekOption(this.eventQueue).pipe(
        STM.map((option) => option._tag === "Some" ? option.value : null)
      )
      
      return {
        pendingEvents,
        processedEvents: processedEventsArray.length,
        deadLetterEvents,
        nextEvent
      }
    }).pipe(STM.commit)

  getProcessedEvents = (): Effect.Effect<readonly ProcessedEvent[]> =>
    STM.commit(TRef.get(this.processedEvents))

  getDeadLetterEvents = (): Effect.Effect<readonly Event[]> =>
    STM.commit(TPriorityQueue.toArray(this.deadLetterQueue))
}

// Usage
const eventProcessingExample = Effect.gen(function* () {
  // Define event processors
  const processors: EventProcessor[] = [
    {
      name: "User Event Processor",
      priority: 10,
      canProcess: (event) => event.type.startsWith("user."),
      process: (event) => Effect.gen(function* () {
        yield* Effect.sleep("50 millis")
        return `Processed user event: ${event.id}`
      })
    },
    {
      name: "Payment Event Processor",
      priority: 15, // Higher priority
      canProcess: (event) => event.type.startsWith("payment."),
      process: (event) => Effect.gen(function* () {
        yield* Effect.sleep("100 millis")
        return `Processed payment event: ${event.id}`
      })
    },
    {
      name: "Analytics Event Processor",
      priority: 5,
      canProcess: (event) => event.type.startsWith("analytics."),
      process: (event) => Effect.gen(function* () {
        yield* Effect.sleep("25 millis")
        return `Processed analytics event: ${event.id}`
      })
    },
    {
      name: "Generic Event Processor",
      priority: 1, // Lowest priority, catches everything
      canProcess: () => true,
      process: (event) => Effect.gen(function* () {
        yield* Effect.sleep("30 millis")
        return `Processed generic event: ${event.id}`
      })
    }
  ]
  
  const pipeline = yield* EventProcessingPipeline.make(processors)
  
  // Generate sample events with different priorities
  const sampleEvents: Event[] = [
    {
      id: "evt-1",
      type: "user.login",
      priority: 5,
      timestamp: Date.now(),
      payload: { userId: "user-123" },
      source: "web-app"
    },
    {
      id: "evt-2",
      type: "payment.completed",
      priority: 10, // High priority
      timestamp: Date.now() + 1000,
      payload: { amount: 99.99, orderId: "order-456" },
      source: "payment-service"
    },
    {
      id: "evt-3",
      type: "analytics.page_view",
      priority: 1, // Low priority
      timestamp: Date.now() + 2000,
      payload: { page: "/dashboard", userId: "user-123" },
      source: "web-app"
    },
    {
      id: "evt-4",
      type: "payment.failed",
      priority: 15, // Critical priority
      timestamp: Date.now() + 500,
      payload: { error: "insufficient_funds", orderId: "order-789" },
      source: "payment-service"
    },
    {
      id: "evt-5",
      type: "system.health_check",
      priority: 8,
      timestamp: Date.now() + 1500,
      payload: { status: "healthy" },
      source: "monitoring"
    }
  ]
  
  // Publish events
  yield* pipeline.publishEvents(sampleEvents)
  
  // Process all events
  const processedCount = yield* pipeline.drainQueue()
  
  // Get results
  const stats = yield* pipeline.getStats()
  const processedEvents = yield* pipeline.getProcessedEvents()
  const deadLetterEvents = yield* pipeline.getDeadLetterEvents()
  
  return {
    processedCount,
    stats,
    processedEvents: processedEvents.map(pe => ({
      id: pe.originalEvent.id,
      type: pe.originalEvent.type,
      processingTimeMs: pe.processingTimeMs,
      result: pe.result
    })),
    deadLetterCount: deadLetterEvents.length
  }
})
```

### Example 3: Resource Pool with Priority Allocation

```typescript
import { Effect, STM, TPriorityQueue, TRef, TMap, Order } from "effect"

interface ResourceRequest {
  readonly id: string
  readonly requesterType: "critical" | "normal" | "background"
  readonly resourceType: string
  readonly priority: number
  readonly requestedAt: number
  readonly timeout: number
}

interface Resource {
  readonly id: string
  readonly type: string
  readonly isAvailable: boolean
  readonly allocatedTo?: string
  readonly allocatedAt?: number
}

interface ResourceAllocation {
  readonly requestId: string
  readonly resourceId: string
  readonly allocatedAt: number
  readonly expiresAt: number
}

class PriorityResourcePool {
  constructor(
    private requestQueue: TPriorityQueue.TPriorityQueue<ResourceRequest>,
    private resources: TMap.TMap<string, Resource>,
    private allocations: TMap.TMap<string, ResourceAllocation>
  ) {}

  static make = (resources: readonly Resource[]): Effect.Effect<PriorityResourcePool> => {
    const requestOrder = Order.make<ResourceRequest>((a, b) => {
      // Higher priority first
      if (a.priority !== b.priority) {
        return b.priority - a.priority
      }
      // Earlier request first (if same priority)
      return a.requestedAt - b.requestedAt
    })

    return STM.gen(function* () {
      const requestQueue = yield* TPriorityQueue.empty<ResourceRequest>(requestOrder)
      const resourceMap = yield* TMap.fromIterable(
        resources.map(r => [r.id, r] as const)
      )
      const allocations = yield* TMap.empty<string, ResourceAllocation>()
      
      return new PriorityResourcePool(requestQueue, resourceMap, allocations)
    }).pipe(STM.commit)
  }

  // Calculate priority based on requester type and urgency
  private calculatePriority = (
    requesterType: ResourceRequest["requesterType"],
    urgency: number = 1
  ): number => {
    const basePriority = {
      critical: 100,
      normal: 50,
      background: 10
    }[requesterType]
    
    return basePriority + urgency
  }

  requestResource = (
    requestId: string,
    requesterType: ResourceRequest["requesterType"],
    resourceType: string,
    urgency: number = 1,
    timeoutMs: number = 30000
  ): Effect.Effect<string> => {
    const priority = this.calculatePriority(requesterType, urgency)
    const request: ResourceRequest = {
      id: requestId,
      requesterType,
      resourceType,
      priority,
      requestedAt: Date.now(),
      timeout: timeoutMs
    }

    return STM.commit(TPriorityQueue.offer(this.requestQueue, request)).pipe(
      Effect.flatMap(() => this.processRequests()),
      Effect.flatMap(() => this.waitForAllocation(requestId, timeoutMs))
    )
  }

  private waitForAllocation = (requestId: string, timeoutMs: number): Effect.Effect<string> =>
    Effect.gen(function* () {
      const startTime = Date.now()
      
      while (Date.now() - startTime < timeoutMs) {
        const allocation = yield* STM.commit(TMap.get(this.allocations, requestId))
        
        if (allocation) {
          return allocation.resourceId
        }
        
        yield* Effect.sleep("100 millis")
      }
      
      // Timeout - remove request from queue if still there
      yield* STM.commit(
        TPriorityQueue.removeIf(this.requestQueue, (req) => req.id === requestId)
      )
      
      return yield* Effect.fail(new Error(`Resource request ${requestId} timed out`))
    })

  private processRequests = (): Effect.Effect<void> =>
    STM.gen(function* () {
      const isEmpty = yield* TPriorityQueue.isEmpty(this.requestQueue)
      if (isEmpty) return
      
      // Get next highest priority request
      const request = yield* TPriorityQueue.take(this.requestQueue)
      
      // Find available resource of the requested type
      const availableResource = yield* this.findAvailableResource(request.resourceType)
      
      if (availableResource) {
        // Allocate resource
        const allocation: ResourceAllocation = {
          requestId: request.id,
          resourceId: availableResource.id,
          allocatedAt: Date.now(),
          expiresAt: Date.now() + 300000 // 5 minutes default allocation
        }
        
        // Update resource as allocated
        const allocatedResource: Resource = {
          ...availableResource,
          isAvailable: false,
          allocatedTo: request.id,
          allocatedAt: Date.now()
        }
        
        yield* TMap.set(this.resources, availableResource.id, allocatedResource)
        yield* TMap.set(this.allocations, request.id, allocation)
      } else {
        // No available resource, put request back (it will be processed later)
        yield* TPriorityQueue.offer(this.requestQueue, request)
      }
    }).pipe(STM.commit)

  private findAvailableResource = (resourceType: string): STM.STM<Resource | null> =>
    STM.gen(function* () {
      const allResources = yield* TMap.toArray(this.resources)
      
      for (const [_, resource] of allResources) {
        if (resource.type === resourceType && resource.isAvailable) {
          return resource
        }
      }
      
      return null
    })

  releaseResource = (requestId: string): Effect.Effect<void> =>
    STM.gen(function* () {
      const allocation = yield* TMap.get(this.allocations, requestId)
      
      if (!allocation) {
        return // No allocation found
      }
      
      // Mark resource as available
      const resource = yield* TMap.get(this.resources, allocation.resourceId)
      if (resource) {
        const availableResource: Resource = {
          ...resource,
          isAvailable: true,
          allocatedTo: undefined,
          allocatedAt: undefined
        }
        yield* TMap.set(this.resources, allocation.resourceId, availableResource)
      }
      
      // Remove allocation
      yield* TMap.remove(this.allocations, requestId)
    }).pipe(
      STM.commit,
      Effect.flatMap(() => this.processRequests()) // Process waiting requests
    )

  cleanupExpiredAllocations = (): Effect.Effect<number> =>
    STM.gen(function* () {
      const now = Date.now()
      const allAllocations = yield* TMap.toArray(this.allocations)
      let cleanedCount = 0
      
      for (const [requestId, allocation] of allAllocations) {
        if (now > allocation.expiresAt) {
          // Release expired allocation
          const resource = yield* TMap.get(this.resources, allocation.resourceId)
          if (resource) {
            const availableResource: Resource = {
              ...resource,
              isAvailable: true,
              allocatedTo: undefined,
              allocatedAt: undefined
            }
            yield* TMap.set(this.resources, allocation.resourceId, availableResource)
          }
          
          yield* TMap.remove(this.allocations, requestId)
          cleanedCount++
        }
      }
      
      return cleanedCount
    }).pipe(
      STM.commit,
      Effect.tap(() => this.processRequests()) // Process waiting requests
    )

  getStats = (): Effect.Effect<{
    totalResources: number
    availableResources: number
    allocatedResources: number
    pendingRequests: number
    allocations: readonly ResourceAllocation[]
  }> =>
    STM.gen(function* () {
      const allResources = yield* TMap.toArray(this.resources)
      const totalResources = allResources.length
      const availableResources = allResources.filter(([_, r]) => r.isAvailable).length
      const allocatedResources = totalResources - availableResources
      
      const pendingRequests = yield* TPriorityQueue.size(this.requestQueue)
      const allAllocations = yield* TMap.toArray(this.allocations)
      
      return {
        totalResources,
        availableResources,
        allocatedResources,
        pendingRequests,
        allocations: allAllocations.map(([_, allocation]) => allocation)
      }
    }).pipe(STM.commit)

  getPendingRequests = (): Effect.Effect<readonly ResourceRequest[]> =>
    STM.commit(TPriorityQueue.toArray(this.requestQueue))
}

// Usage
const resourcePoolExample = Effect.gen(function* () {
  // Create resource pool with mixed resource types
  const resources: Resource[] = [
    { id: "db-1", type: "database", isAvailable: true },
    { id: "db-2", type: "database", isAvailable: true },
    { id: "cache-1", type: "cache", isAvailable: true },
    { id: "cache-2", type: "cache", isAvailable: true },
    { id: "worker-1", type: "compute", isAvailable: true },
    { id: "worker-2", type: "compute", isAvailable: true },
    { id: "worker-3", type: "compute", isAvailable: true }
  ]
  
  const pool = yield* PriorityResourcePool.make(resources)
  
  // Simulate concurrent resource requests with different priorities
  const requests = Effect.all([
    // Critical requests (highest priority)
    pool.requestResource("req-1", "critical", "database", 10),
    pool.requestResource("req-2", "critical", "cache", 8),
    
    // Normal requests
    pool.requestResource("req-3", "normal", "compute", 3),
    pool.requestResource("req-4", "normal", "database", 2),
    
    // Background requests (lowest priority)
    pool.requestResource("req-5", "background", "compute", 1),
    pool.requestResource("req-6", "background", "cache", 1),
  ], { concurrency: "unbounded" })
  
  // Run requests and capture results
  const allocatedResources = yield* requests.pipe(Effect.either)
  
  // Check stats after allocation
  const statsAfterAllocation = yield* pool.getStats()
  
  // Release some resources
  yield* pool.releaseResource("req-1")
  yield* pool.releaseResource("req-3")
  
  // Check stats after release
  const statsAfterRelease = yield* pool.getStats()
  
  // Try more requests to see priority queueing
  const moreRequests = Effect.all([
    pool.requestResource("req-7", "critical", "database", 15),
    pool.requestResource("req-8", "background", "database", 1),
    pool.requestResource("req-9", "normal", "database", 5),
  ], { concurrency: "unbounded" })
  
  const moreAllocations = yield* moreRequests.pipe(Effect.either)
  
  const finalStats = yield* pool.getStats()
  const pendingRequests = yield* pool.getPendingRequests()
  
  return {
    initialAllocations: allocatedResources,
    statsAfterAllocation,
    statsAfterRelease,
    moreAllocations,
    finalStats,
    pendingRequests: pendingRequests.map(req => ({
      id: req.id,
      type: req.requesterType,
      priority: req.priority,
      resourceType: req.resourceType
    }))
  }
})
```

## Advanced Features Deep Dive

### Feature 1: Custom Ordering and Complex Priorities

TPriorityQueue supports sophisticated ordering strategies for complex priority scenarios:

```typescript
import { Order } from "effect"

interface ComplexTask {
  readonly id: string
  readonly priority: number
  readonly deadline: Date
  readonly estimatedDuration: number
  readonly dependencies: readonly string[]
  readonly resourceRequirement: number
}

// Multi-criteria ordering
const complexTaskOrder = Order.make<ComplexTask>((a, b) => {
  // 1. Priority (higher = better)
  if (a.priority !== b.priority) {
    return b.priority - a.priority
  }
  
  // 2. Deadline urgency (earlier = better)
  const aUrgency = a.deadline.getTime() - Date.now()
  const bUrgency = b.deadline.getTime() - Date.now()
  if (aUrgency !== bUrgency) {
    return aUrgency - bUrgency
  }
  
  // 3. Resource efficiency (lower requirement = better)
  if (a.resourceRequirement !== b.resourceRequirement) {
    return a.resourceRequirement - b.resourceRequirement
  }
  
  // 4. Duration (shorter = better)
  return a.estimatedDuration - b.estimatedDuration
})

const advancedOrdering = STM.gen(function* () {
  const queue = yield* TPriorityQueue.empty<ComplexTask>(complexTaskOrder)
  
  const tasks: ComplexTask[] = [
    {
      id: "task-a",
      priority: 5,
      deadline: new Date(Date.now() + 10000),
      estimatedDuration: 1000,
      dependencies: [],
      resourceRequirement: 3
    },
    {
      id: "task-b",
      priority: 5, // Same priority as task-a
      deadline: new Date(Date.now() + 5000), // But earlier deadline
      estimatedDuration: 1000,
      dependencies: [],
      resourceRequirement: 3
    }
  ]
  
  yield* TPriorityQueue.offerAll(queue, tasks)
  
  // task-b should come first due to earlier deadline
  const first = yield* TPriorityQueue.take(queue)
  const second = yield* TPriorityQueue.take(queue)
  
  return { first: first.id, second: second.id }
})
```

### Feature 2: Conditional Operations and Bulk Processing

```typescript
const conditionalOperations = (queue: TPriorityQueue.TPriorityQueue<number>) => STM.gen(function* () {
  // Conditional offer based on queue state
  const size = yield* TPriorityQueue.size(queue)
  if (size < 10) {
    yield* TPriorityQueue.offer(queue, 99)
  }
  
  // Take only if queue has enough items
  const hasEnoughItems = yield* TPriorityQueue.size(queue).pipe(
    STM.map(s => s >= 3)
  )
  
  if (hasEnoughItems) {
    const batch = yield* TPriorityQueue.takeUpTo(queue, 3)
    return batch
  }
  
  return []
})

// Bulk processing with filtering
const bulkProcessing = (queue: TPriorityQueue.TPriorityQueue<number>) => STM.gen(function* () {
  // Process all items above threshold
  const highPriorityItems = yield* TPriorityQueue.takeWhile(
    queue,
    (item) => item > 50
  )
  
  // Remove low priority items without processing
  yield* TPriorityQueue.removeIf(queue, (item) => item < 10)
  
  return highPriorityItems
})

// Helper function for takeWhile (not built-in)
const takeWhile = <A>(
  queue: TPriorityQueue.TPriorityQueue<A>,
  predicate: (item: A) => boolean
): STM.STM<readonly A[]> =>
  STM.gen(function* () {
    const result: A[] = []
    
    while (true) {
      const peeked = yield* TPriorityQueue.peekOption(queue)
      
      if (peeked._tag === "None") {
        break
      }
      
      if (!predicate(peeked.value)) {
        break
      }
      
      const taken = yield* TPriorityQueue.take(queue)
      result.push(taken)
    }
    
    return result
  })
```

### Feature 3: Integration with Other STM Structures

```typescript
import { STM, TPriorityQueue, TRef, TMap } from "effect"

const complexCoordination = STM.gen(function* () {
  const taskQueue = yield* TPriorityQueue.make(Order.number)(5, 10, 1, 8)
  const completedTasks = yield* TRef.make<readonly number[]>([])
  const taskMetrics = yield* TMap.empty<string, number>()
  
  // Complex atomic operation
  const processTask = STM.gen(function* () {
    const isEmpty = yield* TPriorityQueue.isEmpty(taskQueue)
    if (isEmpty) {
      return null
    }
    
    const task = yield* TPriorityQueue.take(taskQueue)
    
    // Update completed tasks
    yield* TRef.update(completedTasks, (tasks) => [...tasks, task])
    
    // Update metrics
    yield* TMap.updateWith(taskMetrics, "totalProcessed", (count) => (count || 0) + 1)
    yield* TMap.updateWith(taskMetrics, "totalValue", (sum) => (sum || 0) + task)
    
    return task
  })
  
  // Process multiple tasks atomically
  const results: number[] = []
  for (let i = 0; i < 3; i++) {
    const result = yield* processTask
    if (result !== null) {
      results.push(result)
    }
  }
  
  const finalCompleted = yield* TRef.get(completedTasks)
  const finalMetrics = yield* TMap.toHashMap(taskMetrics)
  
  return { results, finalCompleted, finalMetrics }
})
```

## Practical Patterns & Best Practices

### Pattern 1: Priority Adjustment Strategies

```typescript
// Helper for dynamic priority adjustment
const adjustPriorities = <A>(
  queue: TPriorityQueue.TPriorityQueue<A & { priority: number }>,
  adjustment: (item: A & { priority: number }) => number
): STM.STM<void> =>
  STM.gen(function* () {
    const allItems = yield* TPriorityQueue.takeAll(queue)
    const adjustedItems = allItems.map(item => ({
      ...item,
      priority: adjustment(item)
    }))
    yield* TPriorityQueue.offerAll(queue, adjustedItems)
  })

// Age-based priority boosting
const boostOldTasks = <A extends { priority: number; timestamp: number }>(
  queue: TPriorityQueue.TPriorityQueue<A>,
  ageThresholdMs: number,
  boost: number
): STM.STM<number> =>
  STM.gen(function* () {
    const now = Date.now()
    let boostedCount = 0
    
    yield* adjustPriorities(queue, (item) => {
      const age = now - item.timestamp
      if (age > ageThresholdMs) {
        boostedCount++
        return item.priority + boost
      }
      return item.priority
    })
    
    return boostedCount
  })
```

### Pattern 2: Queue Monitoring and Health Checks

```typescript
// Helper for queue health monitoring
const monitorQueueHealth = <A>(
  queue: TPriorityQueue.TPriorityQueue<A>,
  maxSize: number,
  maxAge?: number
): STM.STM<{
  isHealthy: boolean
  size: number
  issues: readonly string[]
}> =>
  STM.gen(function* () {
    const size = yield* TPriorityQueue.size(queue)
    const issues: string[] = []
    
    if (size > maxSize) {
      issues.push(`Queue size ${size} exceeds maximum ${maxSize}`)
    }
    
    if (maxAge && size > 0) {
      const items = yield* TPriorityQueue.toArray(queue)
      const now = Date.now()
      const hasOldItems = items.some(item => 
        'timestamp' in item && 
        typeof item.timestamp === 'number' && 
        now - item.timestamp > maxAge
      )
      
      if (hasOldItems) {
        issues.push(`Queue contains items older than ${maxAge}ms`)
      }
    }
    
    return {
      isHealthy: issues.length === 0,
      size,
      issues
    }
  })
```

### Pattern 3: Fair Scheduling with Priority Groups

```typescript
// Helper for fair scheduling across priority groups
const fairScheduler = <A extends { group: string; priority: number }>(
  queue: TPriorityQueue.TPriorityQueue<A>,
  maxItemsPerGroup: number
): STM.STM<readonly A[]> =>
  STM.gen(function* () {
    const allItems = yield* TPriorityQueue.toArray(queue)
    const groupCounts = new Map<string, number>()
    const selectedItems: A[] = []
    
    // Sort by priority within groups
    const sortedItems = [...allItems].sort((a, b) => b.priority - a.priority)
    
    for (const item of sortedItems) {
      const currentCount = groupCounts.get(item.group) || 0
      
      if (currentCount < maxItemsPerGroup) {
        selectedItems.push(item)
        groupCounts.set(item.group, currentCount + 1)
        
        // Remove from queue
        yield* TPriorityQueue.removeIf(queue, (queueItem) => 
          queueItem === item
        )
      }
    }
    
    return selectedItems
  })
```

## Integration Examples

### Integration with Effect Fiber Management

```typescript
import { Effect, STM, TPriorityQueue, Fiber, Deferred } from "effect"

const fiberPoolIntegration = Effect.gen(function* () {
  const taskQueue = yield* STM.commit(TPriorityQueue.make(Order.number)())
  const workers = new Map<string, Fiber.RuntimeFiber<never, void>>()
  
  // Worker fiber that processes tasks
  const createWorker = (workerId: string) =>
    Effect.gen(function* () {
      while (true) {
        const task = yield* STM.gen(function* () {
          const isEmpty = yield* TPriorityQueue.isEmpty(taskQueue)
          if (isEmpty) {
            return null
          }
          return yield* TPriorityQueue.take(taskQueue)
        }).pipe(STM.commit)
        
        if (task === null) {
          yield* Effect.sleep("100 millis")
          continue
        }
        
        // Process task
        yield* Effect.sleep(`${task * 10} millis`)
        console.log(`Worker ${workerId} processed task ${task}`)
      }
    })
  
  // Start worker fibers
  for (let i = 0; i < 3; i++) {
    const workerId = `worker-${i}`
    const worker = yield* Effect.fork(createWorker(workerId))
    workers.set(workerId, worker)
  }
  
  // Add tasks with different priorities
  yield* STM.commit(TPriorityQueue.offerAll(taskQueue, [1, 10, 5, 8, 3, 15, 2]))
  
  // Let workers process tasks
  yield* Effect.sleep("2 seconds")
  
  // Cleanup workers
  for (const [_, worker] of workers) {
    yield* Fiber.interrupt(worker)
  }
  
  const remainingTasks = yield* STM.commit(TPriorityQueue.toArray(taskQueue))
  return { remainingTasks: remainingTasks.length }
})
```

### Testing Strategies

```typescript
import { Effect, STM, TPriorityQueue, Order, TestClock, TestContext } from "effect"

// Test utility for priority queue operations
const testPriorityOrdering = (items: readonly number[]) =>
  Effect.gen(function* () {
    const queue = yield* STM.commit(TPriorityQueue.empty<number>(Order.number))
    
    // Add items in random order
    const shuffledItems = [...items].sort(() => Math.random() - 0.5)
    yield* STM.commit(TPriorityQueue.offerAll(queue, shuffledItems))
    
    // Take all items and verify they come out in priority order
    const results: number[] = []
    while (true) {
      const item = yield* STM.commit(TPriorityQueue.takeOption(queue))
      if (item._tag === "None") break
      results.push(item.value)
    }
    
    // Verify ordering (descending for number order)
    const expectedOrder = [...items].sort((a, b) => b - a)
    return {
      input: items,
      output: results,
      isCorrectOrder: JSON.stringify(results) === JSON.stringify(expectedOrder)
    }
  })

// Concurrent operation test
const testConcurrentOperations = Effect.gen(function* () {
  const queue = yield* STM.commit(TPriorityQueue.empty<number>(Order.number))
  
  // Concurrent offers and takes
  const operations = [
    ...Array.from({ length: 10 }, (_, i) => 
      STM.commit(TPriorityQueue.offer(queue, i))
    ),
    ...Array.from({ length: 5 }, () => 
      STM.commit(TPriorityQueue.takeOption(queue))
    )
  ]
  
  const results = yield* Effect.all(operations, { concurrency: "unbounded" })
  
  const finalSize = yield* STM.commit(TPriorityQueue.size(queue))
  const remainingItems = yield* STM.commit(TPriorityQueue.toArray(queue))
  
  return {
    operationResults: results.filter(r => typeof r === 'object').length, // Successful takes
    finalSize,
    remainingItems
  }
}).pipe(
  Effect.provide(TestContext.TestContext)
)

// Property-based testing for priority invariant
const testPriorityInvariant = (operations: readonly { type: "offer" | "take"; value?: number }[]) =>
  Effect.gen(function* () {
    const queue = yield* STM.commit(TPriorityQueue.empty<number>(Order.number))
    const takenItems: number[] = []
    
    for (const op of operations) {
      if (op.type === "offer" && op.value !== undefined) {
        yield* STM.commit(TPriorityQueue.offer(queue, op.value))
      } else if (op.type === "take") {
        const item = yield* STM.commit(TPriorityQueue.takeOption(queue))
        if (item._tag === "Some") {
          takenItems.push(item.value)
        }
      }
    }
    
    // Verify that taken items are in descending order (priority order)
    const isValidOrder = takenItems.every((item, index) => 
      index === 0 || takenItems[index - 1] >= item
    )
    
    return {
      takenItems,
      isValidOrder,
      remainingSize: yield* STM.commit(TPriorityQueue.size(queue))
    }
  })
```

## Conclusion

TPriorityQueue provides thread-safe, transactional priority-based queuing for task scheduling, resource allocation, and event processing systems.

Key benefits:
- **Priority Ordering**: Maintains consistent priority order across concurrent operations
- **Atomicity**: All operations are atomic within STM transactions
- **Flexible Ordering**: Supports custom ordering strategies for complex priority scenarios
- **Composability**: Seamlessly integrates with other STM data structures

TPriorityQueue is ideal for scenarios requiring priority-based processing, task scheduling systems, and resource allocation where order and consistency are critical.