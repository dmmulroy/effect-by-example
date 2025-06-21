# Queue: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem Queue Solves

Modern applications need to handle asynchronous communication between different parts of the system - producers and consumers operating at different rates, background processing, work distribution, and buffering. Traditional approaches using arrays, event emitters, or manual coordination quickly become problematic:

```typescript
// Traditional approach - manual producer-consumer coordination
class ManualQueue<T> {
  private items: T[] = []
  private consumers: Array<(item: T) => void> = []
  private maxSize: number
  
  constructor(maxSize: number = 100) {
    this.maxSize = maxSize
  }
  
  // Producer side - what happens when queue is full?
  offer(item: T): boolean {
    if (this.items.length >= this.maxSize) {
      return false // Dropped! No backpressure mechanism
    }
    
    this.items.push(item)
    
    // Notify waiting consumers - but race conditions!
    if (this.consumers.length > 0) {
      const consumer = this.consumers.shift()
      const nextItem = this.items.shift()
      if (consumer && nextItem) {
        consumer(nextItem)
      }
    }
    
    return true
  }
  
  // Consumer side - what if no items available?
  take(): Promise<T> {
    return new Promise((resolve) => {
      if (this.items.length > 0) {
        const item = this.items.shift()!
        resolve(item)
      } else {
        // Add to waiting consumers - but what about memory leaks?
        this.consumers.push(resolve)
      }
    })
  }
}

// Usage problems
const queue = new ManualQueue<string>(5)

// Producer overwhelms consumer
async function producer() {
  for (let i = 0; i < 100; i++) {
    const success = queue.offer(`item-${i}`)
    if (!success) {
      console.log(`Dropped item-${i}`) // Lost data!
    }
  }
}

// Slow consumer
async function consumer() {
  while (true) {
    const item = await queue.take()
    // What about graceful shutdown?
    // What about error handling?
    await processItem(item)
  }
}

// Event-based coordination gets complex quickly
class EventBasedQueue extends EventEmitter {
  private items: any[] = []
  
  offer(item: any) {
    this.items.push(item)
    this.emit('item-added', item)
  }
  
  take() {
    if (this.items.length > 0) {
      return Promise.resolve(this.items.shift())
    }
    
    return new Promise(resolve => {
      this.once('item-added', () => {
        resolve(this.items.shift())
      })
    })
  }
  
  // How do we handle shutdown? Cleanup? Backpressure?
  // Error handling across async boundaries becomes nightmare
}
```

This approach leads to:
- **No backpressure** - Fast producers overwhelm slow consumers
- **Race conditions** - Concurrent access without proper synchronization  
- **Memory leaks** - Unbounded growth of waiting consumers
- **Poor error handling** - Errors don't propagate correctly
- **No graceful shutdown** - Difficult to coordinate cleanup

### The Queue Solution

Effect's Queue provides type-safe, backpressure-aware, asynchronous communication with built-in coordination:

```typescript
import { Effect, Queue, Fiber } from "effect"

// Type-safe queue with automatic backpressure
const producer = (queue: Queue.Queue<string>) =>
  Effect.gen(function* () {
    for (let i = 0; i < 100; i++) {
      // Automatically suspends when queue is full (backpressure)
      yield* Queue.offer(queue, `item-${i}`)
      console.log(`Produced item-${i}`)
    }
  })

const consumer = (queue: Queue.Queue<string>) =>
  Effect.gen(function* () {
    yield* Effect.forever(
      Effect.gen(function* () {
        // Automatically suspends when queue is empty
        const item = yield* Queue.take(queue)
        yield* processItem(item)
        console.log(`Consumed ${item}`)
      })
    )
  })

// Clean producer-consumer pattern
const producerConsumerPipeline = Effect.gen(function* () {
  // Bounded queue with automatic backpressure
  const queue = yield* Queue.bounded<string>(10)
  
  // Fork producer and consumer
  const producerFiber = yield* Effect.fork(producer(queue))
  const consumerFiber = yield* Effect.fork(consumer(queue))
  
  // Wait for producer to finish
  yield* Fiber.join(producerFiber)
  
  // Graceful shutdown
  yield* Queue.shutdown(queue)
  yield* Fiber.interrupt(consumerFiber)
})
```

### Key Concepts

**Queue**: A type-safe, asynchronous data structure for producer-consumer communication with built-in backpressure.

**Backpressure**: Automatic flow control where producers suspend when queue is full, preventing overwhelming consumers.

**Bounded/Unbounded**: Queues can have size limits (bounded) or grow indefinitely (unbounded).

**Offer/Take**: Core operations for adding and removing items, with automatic suspension and resumption.

**Shutdown**: Coordinated cleanup that interrupts all waiting fibers and prevents new operations.

## Basic Usage Patterns

### Pattern 1: Simple Producer-Consumer

```typescript
import { Effect, Queue, Fiber } from "effect"

interface WorkItem {
  id: string
  data: any
  priority: number
}

const processWorkItem = (item: WorkItem): Effect.Effect<void> =>
  Effect.gen(function* () {
    console.log(`Processing work item ${item.id}`)
    // Simulate work
    yield* Effect.sleep(`${100 + Math.random() * 200}ms`)
    console.log(`Completed work item ${item.id}`)
  })

// Simple bounded queue pattern
const simpleWorkQueue = Effect.gen(function* () {
  const queue = yield* Queue.bounded<WorkItem>(5)
  
  // Producer: generates work items
  const producer = Effect.gen(function* () {
    for (let i = 0; i < 10; i++) {
      const workItem: WorkItem = {
        id: `work-${i}`,
        data: { value: i * 10 },
        priority: Math.floor(Math.random() * 3)
      }
      
      yield* Queue.offer(queue, workItem)
      console.log(`Queued work item ${workItem.id}`)
    }
  })
  
  // Consumer: processes work items
  const consumer = Effect.gen(function* () {
    yield* Effect.forever(
      Effect.gen(function* () {
        const workItem = yield* Queue.take(queue)
        yield* processWorkItem(workItem)
      })
    )
  })
  
  // Run producer and consumer concurrently
  const producerFiber = yield* Effect.fork(producer)
  const consumerFiber = yield* Effect.fork(consumer)
  
  // Wait for producer to finish
  yield* Fiber.join(producerFiber)
  
  // Give consumer time to finish remaining items
  yield* Effect.sleep("1 second")
  
  // Shutdown gracefully
  yield* Fiber.interrupt(consumerFiber)
  yield* Queue.shutdown(queue)
})
```

### Pattern 2: Multiple Producers and Consumers

```typescript
import { Effect, Queue, Fiber } from "effect"

interface Task {
  id: string
  type: "urgent" | "normal" | "low"
  payload: any
  createdAt: number
}

const processTask = (task: Task, workerId: string): Effect.Effect<void> =>
  Effect.gen(function* () {
    const processingTime = task.type === "urgent" ? 100 : 
                          task.type === "normal" ? 300 : 500
    
    console.log(`Worker ${workerId} processing ${task.type} task ${task.id}`)
    yield* Effect.sleep(`${processingTime}ms`)
    console.log(`Worker ${workerId} completed task ${task.id}`)
  })

const multiProducerConsumer = Effect.gen(function* () {
  const taskQueue = yield* Queue.bounded<Task>(20)
  
  // Multiple producers generating different types of tasks
  const createProducer = (producerId: string, taskType: Task["type"], count: number) =>
    Effect.gen(function* () {
      for (let i = 0; i < count; i++) {
        const task: Task = {
          id: `${producerId}-${i}`,
          type: taskType,
          payload: { data: `payload-${i}` },
          createdAt: Date.now()
        }
        
        yield* Queue.offer(taskQueue, task)
        console.log(`Producer ${producerId} created ${taskType} task ${task.id}`)
        
        // Variable production rate
        yield* Effect.sleep(`${Math.random() * 100}ms`)
      }
    })
  
  // Multiple consumers with different capabilities
  const createConsumer = (workerId: string) =>
    Effect.gen(function* () {
      yield* Effect.forever(
        Effect.gen(function* () {
          const task = yield* Queue.take(taskQueue)
          yield* processTask(task, workerId)
        })
      )
    })
  
  // Start multiple producers
  const producerFibers = yield* Effect.all([
    Effect.fork(createProducer("urgent-producer", "urgent", 5)),
    Effect.fork(createProducer("normal-producer", "normal", 8)),
    Effect.fork(createProducer("batch-producer", "low", 10))
  ])
  
  // Start multiple consumers
  const consumerFibers = yield* Effect.all([
    Effect.fork(createConsumer("worker-1")),
    Effect.fork(createConsumer("worker-2")),
    Effect.fork(createConsumer("worker-3"))
  ])
  
  // Wait for all producers to finish
  yield* Effect.all(producerFibers.map(fiber => Fiber.join(fiber)))
  console.log("All producers finished")
  
  // Let consumers finish remaining tasks
  yield* Effect.sleep("2 seconds")
  
  // Cleanup
  yield* Effect.all(consumerFibers.map(fiber => Fiber.interrupt(fiber)))
  yield* Queue.shutdown(taskQueue)
})
```

### Pattern 3: Queue with Different Overflow Strategies

```typescript
import { Effect, Queue } from "effect"

interface LogEntry {
  level: "info" | "warn" | "error"
  message: string
  timestamp: number
  source: string
}

// Different queue strategies for different use cases
const queueStrategiesExample = Effect.gen(function* () {
  // Bounded queue - blocks when full (backpressure)
  const criticalQueue = yield* Queue.bounded<LogEntry>(100)
  
  // Dropping queue - drops new items when full
  const auditQueue = yield* Queue.dropping<LogEntry>(50)
  
  // Sliding queue - drops old items to make room for new ones
  const metricsQueue = yield* Queue.sliding<LogEntry>(30)
  
  // Unbounded queue - grows indefinitely
  const debugQueue = yield* Queue.unbounded<LogEntry>()
  
  const generateLogEntry = (level: LogEntry["level"], message: string, source: string): LogEntry => ({
    level,
    message,
    timestamp: Date.now(),
    source
  })
  
  // Simulate high-volume logging
  const logProducer = Effect.gen(function* () {
    for (let i = 0; i < 200; i++) {
      const entry = generateLogEntry(
        i % 10 === 0 ? "error" : i % 5 === 0 ? "warn" : "info",
        `Log message ${i}`,
        `service-${i % 3}`
      )
      
      // Send to appropriate queue based on level
      if (entry.level === "error") {
        yield* Queue.offer(criticalQueue, entry) // Will block if full
      } else if (entry.level === "warn") {
        yield* Queue.offer(auditQueue, entry) // Will drop if full
      } else {
        yield* Queue.offer(metricsQueue, entry) // Will slide if full
      }
      
      // Also send all to debug
      yield* Queue.offer(debugQueue, entry)
      
      yield* Effect.sleep("10 millis")
    }
  })
  
  // Different consumers for different priorities
  const criticalConsumer = Effect.gen(function* () {
    yield* Effect.forever(
      Effect.gen(function* () {
        const entry = yield* Queue.take(criticalQueue)
        console.log(`CRITICAL: ${entry.message}`)
        yield* Effect.sleep("50 millis") // Fast processing
      })
    )
  })
  
  const auditConsumer = Effect.gen(function* () {
    yield* Effect.forever(
      Effect.gen(function* () {
        const entry = yield* Queue.take(auditQueue)
        console.log(`AUDIT: ${entry.message}`)
        yield* Effect.sleep("100 millis") // Medium processing
      })
    )
  })
  
  const metricsConsumer = Effect.gen(function* () {
    // Batch processing for metrics
    yield* Effect.forever(
      Effect.gen(function* () {
        const batch = yield* Queue.takeUpTo(metricsQueue, 10)
        if (batch.length > 0) {
          console.log(`METRICS: Processed batch of ${batch.length} entries`)
        }
        yield* Effect.sleep("200 millis")
      })
    )
  })
  
  // Start all producers and consumers
  const producerFiber = yield* Effect.fork(logProducer)
  const consumerFibers = yield* Effect.all([
    Effect.fork(criticalConsumer),
    Effect.fork(auditConsumer),
    Effect.fork(metricsConsumer)
  ])
  
  // Wait for producer
  yield* Fiber.join(producerFiber)
  
  // Let consumers finish
  yield* Effect.sleep("1 second")
  
  // Check queue sizes
  const criticalSize = yield* Queue.size(criticalQueue)
  const auditSize = yield* Queue.size(auditQueue)
  const metricsSize = yield* Queue.size(metricsQueue)
  const debugSize = yield* Queue.size(debugQueue)
  
  console.log(`Final queue sizes - Critical: ${criticalSize}, Audit: ${auditSize}, Metrics: ${metricsSize}, Debug: ${debugSize}`)
  
  // Cleanup
  yield* Effect.all(consumerFibers.map(fiber => Fiber.interrupt(fiber)))
})
```

## Real-World Examples

### Example 1: High-Performance Job Processing System

A job processing system that handles different job types with priorities, retries, and monitoring.

```typescript
import { Effect, Queue, Fiber, Ref, Schedule } from "effect"

interface Job {
  id: string
  type: string
  priority: number
  payload: any
  attempts: number
  maxAttempts: number
  createdAt: number
  scheduledAt?: number
}

interface JobResult {
  jobId: string
  success: boolean
  result?: any
  error?: string
  completedAt: number
  duration: number
}

interface ProcessorMetrics {
  processed: number
  failed: number
  retried: number
  averageProcessingTime: number
}

class JobProcessor {
  private jobQueue: Queue.Queue<Job>
  private retryQueue: Queue.Queue<Job>
  private resultQueue: Queue.Queue<JobResult>
  private metrics: Ref.Ref<ProcessorMetrics>
  private isShuttingDown: Ref.Ref<boolean>
  
  constructor(
    jobQueue: Queue.Queue<Job>,
    retryQueue: Queue.Queue<Job>,
    resultQueue: Queue.Queue<JobResult>,
    metrics: Ref.Ref<ProcessorMetrics>,
    isShuttingDown: Ref.Ref<boolean>
  ) {
    this.jobQueue = jobQueue
    this.retryQueue = retryQueue
    this.resultQueue = resultQueue
    this.metrics = metrics
    this.isShuttingDown = isShuttingDown
  }
  
  static make = (queueSize: number = 1000) =>
    Effect.gen(function* () {
      const jobQueue = yield* Queue.bounded<Job>(queueSize)
      const retryQueue = yield* Queue.bounded<Job>(queueSize / 2)
      const resultQueue = yield* Queue.bounded<JobResult>(queueSize)
      const metrics = yield* Ref.make<ProcessorMetrics>({
        processed: 0,
        failed: 0,
        retried: 0,
        averageProcessingTime: 0
      })
      const isShuttingDown = yield* Ref.make(false)
      
      const processor = new JobProcessor(
        jobQueue, retryQueue, resultQueue, metrics, isShuttingDown
      )
      
      // Start background workers
      yield* Effect.fork(processor.jobWorker())
      yield* Effect.fork(processor.retryWorker())
      yield* Effect.fork(processor.metricsReporter())
      
      return processor
    })
  
  submitJob = (job: Omit<Job, 'attempts' | 'createdAt'>) =>
    Effect.gen(function* () {
      const fullJob: Job = {
        ...job,
        attempts: 0,
        createdAt: Date.now()
      }
      
      yield* Queue.offer(this.jobQueue, fullJob)
      return fullJob.id
    })
  
  private processJob = (job: Job): Effect.Effect<any> =>
    Effect.gen(function* () {
      // Simulate different job types with different processing times and failure rates
      const processingTime = job.type === "fast" ? 100 : 
                           job.type === "normal" ? 500 : 
                           job.type === "slow" ? 2000 : 300
      
      const failureRate = job.type === "unreliable" ? 0.3 : 0.1
      
      yield* Effect.sleep(`${processingTime}ms`)
      
      if (Math.random() < failureRate) {
        yield* Effect.fail(new Error(`Job ${job.id} failed randomly`))
      }
      
      return { result: `Processed ${job.type} job ${job.id}`, data: job.payload }
    })
  
  private jobWorker = () =>
    Effect.gen(function* () {
      yield* Effect.forever(
        Effect.gen(function* () {
          // Check if shutting down
          const shuttingDown = yield* Ref.get(this.isShuttingDown)
          if (shuttingDown) {
            return
          }
          
          const job = yield* Queue.take(this.jobQueue)
          const startTime = Date.now()
          
          const result = yield* this.processJob(job).pipe(
            Effect.either,
            Effect.tap(() => this.updateMetrics(Date.now() - startTime, true)),
            Effect.catchAll(() => this.updateMetrics(Date.now() - startTime, false))
          )
          
          const jobResult: JobResult = {
            jobId: job.id,
            success: result._tag === "Right",
            result: result._tag === "Right" ? result.right : undefined,
            error: result._tag === "Left" ? result.left.message : undefined,
            completedAt: Date.now(),
            duration: Date.now() - startTime
          }
          
          if (result._tag === "Left" && job.attempts < job.maxAttempts) {
            // Retry logic
            const retryJob: Job = {
              ...job,
              attempts: job.attempts + 1,
              scheduledAt: Date.now() + Math.pow(2, job.attempts) * 1000 // Exponential backoff
            }
            
            yield* Queue.offer(this.retryQueue, retryJob)
            yield* Ref.update(this.metrics, m => ({ ...m, retried: m.retried + 1 }))
          } else {
            // Final result
            yield* Queue.offer(this.resultQueue, jobResult)
          }
        })
      )
    })
  
  private retryWorker = () =>
    Effect.gen(function* () {
      yield* Effect.forever(
        Effect.gen(function* () {
          const job = yield* Queue.take(this.retryQueue)
          
          // Wait until scheduled time
          if (job.scheduledAt) {
            const waitTime = job.scheduledAt - Date.now()
            if (waitTime > 0) {
              yield* Effect.sleep(`${waitTime}ms`)
            }
          }
          
          // Move back to main queue
          yield* Queue.offer(this.jobQueue, job)
        })
      )
    })
  
  private updateMetrics = (duration: number, success: boolean) =>
    Ref.update(this.metrics, current => {
      const newProcessed = current.processed + 1
      const newFailed = success ? current.failed : current.failed + 1
      const newAverage = (current.averageProcessingTime * current.processed + duration) / newProcessed
      
      return {
        processed: newProcessed,
        failed: newFailed,
        retried: current.retried,
        averageProcessingTime: newAverage
      }
    })
  
  private metricsReporter = () =>
    Effect.gen(function* () {
      yield* Effect.forever(
        Effect.gen(function* () {
          const metrics = yield* Ref.get(this.metrics)
          const jobQueueSize = yield* Queue.size(this.jobQueue)
          const retryQueueSize = yield* Queue.size(this.retryQueue)
          const resultQueueSize = yield* Queue.size(this.resultQueue)
          
          console.log("Job Processor Metrics:", {
            ...metrics,
            queueSizes: {
              jobs: jobQueueSize,
              retries: retryQueueSize,
              results: resultQueueSize
            }
          })
          
          yield* Effect.sleep("10 seconds")
        })
      )
    })
  
  getResults = (count: number = 10) =>
    Queue.takeUpTo(this.resultQueue, count)
  
  shutdown = () =>
    Effect.gen(function* () {
      yield* Ref.set(this.isShuttingDown, true)
      
      // Wait a bit for current jobs to finish
      yield* Effect.sleep("2 seconds")
      
      // Shutdown queues
      yield* Queue.shutdown(this.jobQueue)
      yield* Queue.shutdown(this.retryQueue)
      yield* Queue.shutdown(this.resultQueue)
      
      const finalMetrics = yield* Ref.get(this.metrics)
      console.log("Final metrics:", finalMetrics)
    })
}

// Usage example
const jobProcessingExample = Effect.gen(function* () {
  const processor = yield* JobProcessor.make(500)
  
  // Submit various types of jobs
  const jobTypes = ["fast", "normal", "slow", "unreliable"]
  const jobPromises = []
  
  for (let i = 0; i < 100; i++) {
    const jobType = jobTypes[i % jobTypes.length]
    const jobId = yield* processor.submitJob({
      id: `job-${i}`,
      type: jobType,
      priority: Math.floor(Math.random() * 5),
      payload: { data: `job-${i}-data` },
      maxAttempts: 3
    })
    jobPromises.push(jobId)
  }
  
  console.log(`Submitted ${jobPromises.length} jobs`)
  
  // Let the system process jobs
  yield* Effect.sleep("30 seconds")
  
  // Get some results
  const results = yield* processor.getResults(20)
  console.log(`Retrieved ${results.length} results`)
  
  // Shutdown gracefully
  yield* processor.shutdown()
})
```

### Example 2: Real-time Event Streaming Pipeline

A real-time event processing pipeline with multiple stages, filtering, and aggregation.

```typescript
import { Effect, Queue, Fiber, Ref, Chunk } from "effect"

interface RawEvent {
  id: string
  type: string
  source: string
  timestamp: number
  data: any
}

interface ProcessedEvent {
  id: string
  type: string
  source: string
  timestamp: number
  processedAt: number
  enrichedData: any
  tags: string[]
}

interface AggregatedEvent {
  windowStart: number
  windowEnd: number
  source: string
  eventCount: number
  eventTypes: Record<string, number>
  summary: any
}

class EventPipeline {
  private rawQueue: Queue.Queue<RawEvent>
  private filteredQueue: Queue.Queue<RawEvent>
  private processedQueue: Queue.Queue<ProcessedEvent>
  private aggregatedQueue: Queue.Queue<AggregatedEvent>
  private errorQueue: Queue.Queue<{ event: RawEvent, error: string }>
  
  constructor(
    rawQueue: Queue.Queue<RawEvent>,
    filteredQueue: Queue.Queue<RawEvent>,
    processedQueue: Queue.Queue<ProcessedEvent>,
    aggregatedQueue: Queue.Queue<AggregatedEvent>,
    errorQueue: Queue.Queue<{ event: RawEvent, error: string }>
  ) {
    this.rawQueue = rawQueue
    this.filteredQueue = filteredQueue
    this.processedQueue = processedQueue
    this.aggregatedQueue = aggregatedQueue
    this.errorQueue = errorQueue
  }
  
  static make = () =>
    Effect.gen(function* () {
      const rawQueue = yield* Queue.bounded<RawEvent>(1000)
      const filteredQueue = yield* Queue.bounded<RawEvent>(800)
      const processedQueue = yield* Queue.bounded<ProcessedEvent>(600)
      const aggregatedQueue = yield* Queue.bounded<AggregatedEvent>(100)
      const errorQueue = yield* Queue.bounded<{ event: RawEvent, error: string }>(200)
      
      const pipeline = new EventPipeline(
        rawQueue, filteredQueue, processedQueue, aggregatedQueue, errorQueue
      )
      
      // Start pipeline stages
      yield* Effect.fork(pipeline.filterStage())
      yield* Effect.fork(pipeline.processingStage())
      yield* Effect.fork(pipeline.aggregationStage())
      yield* Effect.fork(pipeline.errorHandler())
      
      return pipeline
    })
  
  ingestEvent = (event: RawEvent) =>
    Queue.offer(this.rawQueue, event).pipe(
      Effect.timeout("100 millis"),
      Effect.catchAll(() => 
        Effect.sync(() => console.warn("Raw queue full, dropping event"))
      )
    )
  
  private filterStage = () =>
    Effect.gen(function* () {
      yield* Effect.forever(
        Effect.gen(function* () {
          const event = yield* Queue.take(this.rawQueue)
          
          // Filter logic - only process certain event types
          const allowedTypes = ["user_action", "system_event", "error", "metric"]
          const allowedSources = ["web", "mobile", "api", "background"]
          
          if (allowedTypes.includes(event.type) && allowedSources.includes(event.source)) {
            yield* Queue.offer(this.filteredQueue, event)
          } else {
            console.log(`Filtered out event ${event.id} (type: ${event.type}, source: ${event.source})`)
          }
        })
      )
    })
  
  private processingStage = () =>
    Effect.gen(function* () {
      // Multiple workers for parallel processing
      const workers = Array.from({ length: 3 }, (_, i) => 
        Effect.fork(this.processingWorker(`worker-${i}`))
      )
      
      yield* Effect.all(workers)
    })
  
  private processingWorker = (workerId: string) =>
    Effect.gen(function* () {
      yield* Effect.forever(
        Effect.gen(function* () {
          const event = yield* Queue.take(this.filteredQueue)
          
          const processedEvent = yield* this.enrichEvent(event).pipe(
            Effect.catchAll(error => 
              Effect.gen(function* () {
                yield* Queue.offer(this.errorQueue, { event, error: error.message })
                return null
              })
            )
          )
          
          if (processedEvent) {
            yield* Queue.offer(this.processedQueue, processedEvent)
          }
        })
      )
    })
  
  private enrichEvent = (event: RawEvent): Effect.Effect<ProcessedEvent> =>
    Effect.gen(function* () {
      // Simulate enrichment with external data
      yield* Effect.sleep("50 millis")
      
      // Simulate occasional failures
      if (Math.random() < 0.05) {
        yield* Effect.fail(new Error("Enrichment service unavailable"))
      }
      
      const enrichedData = {
        ...event.data,
        userAgent: event.source === "web" ? "Chrome/91.0" : undefined,
        geoLocation: event.source === "mobile" ? { country: "US", city: "SF" } : undefined,
        enrichedAt: Date.now()
      }
      
      const tags = []
      if (event.type === "error") tags.push("alert", "high-priority")
      if (event.type === "user_action") tags.push("behavioral")
      if (event.source === "mobile") tags.push("mobile")
      
      return {
        id: event.id,
        type: event.type,
        source: event.source,
        timestamp: event.timestamp,
        processedAt: Date.now(),
        enrichedData,
        tags
      }
    })
  
  private aggregationStage = () =>
    Effect.gen(function* () {
      const windowSize = 10000 // 10 second windows
      const aggregationBuffer = yield* Ref.make<Map<string, ProcessedEvent[]>>(new Map())
      
      yield* Effect.forever(
        Effect.gen(function* () {
          // Collect events for batch processing
          const events = yield* Queue.takeUpTo(this.processedQueue, 50)
          
          if (events.length > 0) {
            // Group by source and time window
            const now = Date.now()
            const windowStart = Math.floor(now / windowSize) * windowSize
            
            const eventsBySource = Chunk.reduce(events, new Map<string, ProcessedEvent[]>(), (acc, event) => {
              const key = `${event.source}-${windowStart}`
              const existing = acc.get(key) || []
              acc.set(key, [...existing, event])
              return acc
            })
            
            // Create aggregations
            for (const [key, sourceEvents] of eventsBySource.entries()) {
              const [source] = key.split('-')
              
              const aggregation: AggregatedEvent = {
                windowStart,
                windowEnd: windowStart + windowSize,
                source,
                eventCount: sourceEvents.length,
                eventTypes: sourceEvents.reduce((acc, event) => {
                  acc[event.type] = (acc[event.type] || 0) + 1
                  return acc
                }, {} as Record<string, number>),
                summary: {
                  avgProcessingTime: sourceEvents.reduce((sum, e) => 
                    sum + (e.processedAt - e.timestamp), 0) / sourceEvents.length,
                  tagCounts: sourceEvents.flatMap(e => e.tags).reduce((acc, tag) => {
                    acc[tag] = (acc[tag] || 0) + 1
                    return acc
                  }, {} as Record<string, number>)
                }
              }
              
              yield* Queue.offer(this.aggregatedQueue, aggregation)
            }
          }
          
          yield* Effect.sleep("1 second")
        })
      )
    })
  
  private errorHandler = () =>
    Effect.gen(function* () {
      yield* Effect.forever(
        Effect.gen(function* () {
          const errorInfo = yield* Queue.take(this.errorQueue)
          console.error(`Error processing event ${errorInfo.event.id}:`, errorInfo.error)
          
          // Could implement dead letter queue, alerting, etc.
          
          yield* Effect.sleep("100 millis")
        })
      )
    })
  
  getAggregations = (count: number = 10) =>
    Queue.takeUpTo(this.aggregatedQueue, count)
  
  getMetrics = () =>
    Effect.gen(function* () {
      return {
        queueSizes: {
          raw: yield* Queue.size(this.rawQueue),
          filtered: yield* Queue.size(this.filteredQueue),
          processed: yield* Queue.size(this.processedQueue),
          aggregated: yield* Queue.size(this.aggregatedQueue),
          errors: yield* Queue.size(this.errorQueue)
        }
      }
    })
  
  shutdown = () =>
    Effect.gen(function* () {
      yield* Queue.shutdown(this.rawQueue)
      yield* Queue.shutdown(this.filteredQueue)
      yield* Queue.shutdown(this.processedQueue)
      yield* Queue.shutdown(this.aggregatedQueue)
      yield* Queue.shutdown(this.errorQueue)
    })
}

// Usage example
const eventPipelineExample = Effect.gen(function* () {
  const pipeline = yield* EventPipeline.make()
  
  // Generate stream of events
  const eventGenerator = Effect.gen(function* () {
    yield* Effect.forever(
      Effect.gen(function* () {
        const sources = ["web", "mobile", "api", "background", "unknown"]
        const types = ["user_action", "system_event", "error", "metric", "spam"]
        
        const event: RawEvent = {
          id: Math.random().toString(36).substr(2, 9),
          type: types[Math.floor(Math.random() * types.length)],
          source: sources[Math.floor(Math.random() * sources.length)],
          timestamp: Date.now(),
          data: {
            userId: Math.floor(Math.random() * 10000),
            sessionId: Math.random().toString(36).substr(2, 9),
            value: Math.random() * 100
          }
        }
        
        yield* pipeline.ingestEvent(event)
        
        // Variable event rate
        yield* Effect.sleep(`${Math.random() * 100}ms`)
      })
    )
  })
  
  // Start event generation
  const generatorFiber = yield* Effect.fork(eventGenerator)
  
  // Monitor metrics
  const metricsMonitor = Effect.gen(function* () {
    yield* Effect.forever(
      Effect.gen(function* () {
        const metrics = yield* pipeline.getMetrics()
        const aggregations = yield* pipeline.getAggregations(5)
        
        console.log("Pipeline Metrics:", metrics)
        console.log(`Recent aggregations: ${aggregations.length}`)
        
        yield* Effect.sleep("5 seconds")
      })
    )
  })
  
  const metricsFiber = yield* Effect.fork(metricsMonitor)
  
  // Run for a while
  yield* Effect.sleep("30 seconds")
  
  // Stop generation and shutdown
  yield* Fiber.interrupt(generatorFiber)
  yield* Fiber.interrupt(metricsFiber)
  yield* pipeline.shutdown()
  
  console.log("Event pipeline demo completed")
})
```

### Example 3: Distributed Task Coordination

A distributed task system that coordinates work across multiple workers with load balancing and failover.

```typescript
import { Effect, Queue, Fiber, Ref, Schedule } from "effect"

interface Task {
  id: string
  type: string
  priority: number
  data: any
  assignedWorker?: string
  attempts: number
  maxAttempts: number
  timeout: number
  createdAt: number
}

interface Worker {
  id: string
  capacity: number
  currentLoad: number
  lastHeartbeat: number
  capabilities: string[]
  status: "active" | "busy" | "offline"
}

interface TaskResult {
  taskId: string
  workerId: string
  success: boolean
  result?: any
  error?: string
  duration: number
  completedAt: number
}

class DistributedTaskCoordinator {
  private taskQueue: Queue.Queue<Task>
  private workerAssignments: Queue.Queue<{ task: Task, workerId: string }>
  private resultQueue: Queue.Queue<TaskResult>
  private failedTaskQueue: Queue.Queue<Task>
  
  private workers: Ref.Ref<Map<string, Worker>>
  private activeTasks: Ref.Ref<Map<string, { task: Task, workerId: string, startTime: number }>>
  
  constructor(
    taskQueue: Queue.Queue<Task>,
    workerAssignments: Queue.Queue<{ task: Task, workerId: string }>,
    resultQueue: Queue.Queue<TaskResult>,
    failedTaskQueue: Queue.Queue<Task>,
    workers: Ref.Ref<Map<string, Worker>>,
    activeTasks: Ref.Ref<Map<string, { task: Task, workerId: string, startTime: number }>>
  ) {
    this.taskQueue = taskQueue
    this.workerAssignments = workerAssignments
    this.resultQueue = resultQueue
    this.failedTaskQueue = failedTaskQueue
    this.workers = workers
    this.activeTasks = activeTasks
  }
  
  static make = () =>
    Effect.gen(function* () {
      const taskQueue = yield* Queue.bounded<Task>(1000)
      const workerAssignments = yield* Queue.bounded<{ task: Task, workerId: string }>(500)
      const resultQueue = yield* Queue.bounded<TaskResult>(1000)
      const failedTaskQueue = yield* Queue.bounded<Task>(200)
      
      const workers = yield* Ref.make(new Map<string, Worker>())
      const activeTasks = yield* Ref.make(new Map<string, { task: Task, workerId: string, startTime: number }>())
      
      const coordinator = new DistributedTaskCoordinator(
        taskQueue, workerAssignments, resultQueue, failedTaskQueue, workers, activeTasks
      )
      
      // Start coordinator processes
      yield* Effect.fork(coordinator.taskScheduler())
      yield* Effect.fork(coordinator.workerMonitor())
      yield* Effect.fork(coordinator.taskMonitor())
      yield* Effect.fork(coordinator.failureHandler())
      
      return coordinator
    })
  
  registerWorker = (worker: Worker) =>
    Effect.gen(function* () {
      yield* Ref.update(this.workers, workers => 
        new Map(workers.set(worker.id, { ...worker, lastHeartbeat: Date.now() }))
      )
      console.log(`Worker ${worker.id} registered with capabilities: ${worker.capabilities.join(', ')}`)
    })
  
  submitTask = (task: Omit<Task, 'attempts' | 'createdAt'>) =>
    Effect.gen(function* () {
      const fullTask: Task = {
        ...task,
        attempts: 0,
        createdAt: Date.now()
      }
      
      yield* Queue.offer(this.taskQueue, fullTask)
      return fullTask.id
    })
  
  private taskScheduler = () =>
    Effect.gen(function* () {
      yield* Effect.forever(
        Effect.gen(function* () {
          const task = yield* Queue.take(this.taskQueue)
          const worker = yield* this.selectWorker(task)
          
          if (worker) {
            yield* Queue.offer(this.workerAssignments, { task, workerId: worker.id })
            
            // Update worker load
            yield* Ref.update(this.workers, workers => {
              const updated = new Map(workers)
              const w = updated.get(worker.id)
              if (w) {
                updated.set(worker.id, { ...w, currentLoad: w.currentLoad + 1 })
              }
              return updated
            })
            
            // Track active task
            yield* Ref.update(this.activeTasks, tasks => 
              new Map(tasks.set(task.id, { task, workerId: worker.id, startTime: Date.now() }))
            )
          } else {
            // No available worker, put task back in queue
            yield* Effect.sleep("1 second")
            yield* Queue.offer(this.taskQueue, task)
          }
        })
      )
    })
  
  private selectWorker = (task: Task): Effect.Effect<Worker | null> =>
    Effect.gen(function* () {
      const workers = yield* Ref.get(this.workers)
      
      // Filter workers by capability and availability
      const availableWorkers = Array.from(workers.values()).filter(worker => 
        worker.status === "active" &&
        worker.currentLoad < worker.capacity &&
        worker.capabilities.includes(task.type) &&
        Date.now() - worker.lastHeartbeat < 30000 // 30 second heartbeat timeout
      )
      
      if (availableWorkers.length === 0) {
        return null
      }
      
      // Load balancing: prefer workers with lower load, higher priority tasks get better workers
      availableWorkers.sort((a, b) => {
        const loadDiff = a.currentLoad - b.currentLoad
        if (loadDiff !== 0) return loadDiff
        return b.capacity - a.capacity // Prefer higher capacity workers
      })
      
      return availableWorkers[0]
    })
  
  private workerMonitor = () =>
    Effect.gen(function* () {
      yield* Effect.forever(
        Effect.gen(function* () {
          const workers = yield* Ref.get(this.workers)
          const now = Date.now()
          
          // Check for offline workers
          const offlineWorkers = Array.from(workers.entries()).filter(([_, worker]) =>
            now - worker.lastHeartbeat > 60000 // 1 minute timeout
          )
          
          if (offlineWorkers.length > 0) {
            console.log(`Detected ${offlineWorkers.length} offline workers`)
            
            // Mark workers as offline and reassign their tasks
            yield* Ref.update(this.workers, currentWorkers => {
              const updated = new Map(currentWorkers)
              for (const [workerId] of offlineWorkers) {
                const worker = updated.get(workerId)
                if (worker) {
                  updated.set(workerId, { ...worker, status: "offline", currentLoad: 0 })
                }
              }
              return updated
            })
            
            // Reassign tasks from offline workers
            yield* this.reassignTasksFromWorkers(offlineWorkers.map(([id]) => id))
          }
          
          yield* Effect.sleep("10 seconds")
        })
      )
    })
  
  private taskMonitor = () =>
    Effect.gen(function* () {
      yield* Effect.forever(
        Effect.gen(function* () {
          const activeTasks = yield* Ref.get(this.activeTasks)
          const now = Date.now()
          
          // Check for timed out tasks
          const timedOutTasks = Array.from(activeTasks.entries()).filter(([_, { task, startTime }]) =>
            now - startTime > task.timeout
          )
          
          if (timedOutTasks.length > 0) {
            console.log(`Found ${timedOutTasks.length} timed out tasks`)
            
            for (const [taskId, { task, workerId }] of timedOutTasks) {
              // Create timeout result
              const result: TaskResult = {
                taskId,
                workerId,
                success: false,
                error: "Task timeout",
                duration: now - activeTasks.get(taskId)!.startTime,
                completedAt: now
              }
              
              yield* Queue.offer(this.resultQueue, result)
              
              // Move to failed queue for retry
              const retryTask: Task = { ...task, attempts: task.attempts + 1 }
              yield* Queue.offer(this.failedTaskQueue, retryTask)
              
              // Clean up
              yield* Ref.update(this.activeTasks, tasks => {
                const updated = new Map(tasks)
                updated.delete(taskId)
                return updated
              })
              
              // Update worker load
              yield* Ref.update(this.workers, workers => {
                const updated = new Map(workers)
                const worker = updated.get(workerId)
                if (worker) {
                  updated.set(workerId, { ...worker, currentLoad: Math.max(0, worker.currentLoad - 1) })
                }
                return updated
              })
            }
          }
          
          yield* Effect.sleep("5 seconds")
        })
      )
    })
  
  private failureHandler = () =>
    Effect.gen(function* () {
      yield* Effect.forever(
        Effect.gen(function* () {
          const failedTask = yield* Queue.take(this.failedTaskQueue)
          
          if (failedTask.attempts < failedTask.maxAttempts) {
            // Retry with exponential backoff
            const delay = Math.min(1000 * Math.pow(2, failedTask.attempts - 1), 30000)
            yield* Effect.sleep(`${delay}ms`)
            yield* Queue.offer(this.taskQueue, failedTask)
            console.log(`Retrying task ${failedTask.id} (attempt ${failedTask.attempts})`)
          } else {
            console.error(`Task ${failedTask.id} failed permanently after ${failedTask.attempts} attempts`)
          }
        })
      )
    })
  
  private reassignTasksFromWorkers = (workerIds: string[]) =>
    Effect.gen(function* () {
      const activeTasks = yield* Ref.get(this.activeTasks)
      
      const tasksToReassign = Array.from(activeTasks.entries())
        .filter(([_, { workerId }]) => workerIds.includes(workerId))
        .map(([_, { task }]) => task)
      
      if (tasksToReassign.length > 0) {
        console.log(`Reassigning ${tasksToReassign.length} tasks from offline workers`)
        
        for (const task of tasksToReassign) {
          const retryTask: Task = { ...task, attempts: task.attempts + 1, assignedWorker: undefined }
          yield* Queue.offer(this.taskQueue, retryTask)
          
          // Clean up active tasks
          yield* Ref.update(this.activeTasks, tasks => {
            const updated = new Map(tasks)
            updated.delete(task.id)
            return updated
          })
        }
      }
    })
  
  // Worker heartbeat
  updateWorkerHeartbeat = (workerId: string) =>
    Ref.update(this.workers, workers => {
      const updated = new Map(workers)
      const worker = updated.get(workerId)
      if (worker) {
        updated.set(workerId, { ...worker, lastHeartbeat: Date.now() })
      }
      return updated
    })
  
  // Simulate worker completing task
  completeTask = (taskResult: TaskResult) =>
    Effect.gen(function* () {
      yield* Queue.offer(this.resultQueue, taskResult)
      
      // Clean up active task
      yield* Ref.update(this.activeTasks, tasks => {
        const updated = new Map(tasks)
        updated.delete(taskResult.taskId)
        return updated
      })
      
      // Update worker load
      yield* Ref.update(this.workers, workers => {
        const updated = new Map(workers)
        const worker = updated.get(taskResult.workerId)
        if (worker) {
          updated.set(taskResult.workerId, { 
            ...worker, 
            currentLoad: Math.max(0, worker.currentLoad - 1),
            lastHeartbeat: Date.now()
          })
        }
        return updated
      })
    })
  
  getSystemMetrics = () =>
    Effect.gen(function* () {
      const workers = yield* Ref.get(this.workers)
      const activeTasks = yield* Ref.get(this.activeTasks)
      
      const workerStats = Array.from(workers.values()).reduce((acc, worker) => {
        acc[worker.status] = (acc[worker.status] || 0) + 1
        return acc
      }, {} as Record<string, number>)
      
      return {
        workers: {
          total: workers.size,
          byStatus: workerStats,
          totalCapacity: Array.from(workers.values()).reduce((sum, w) => sum + w.capacity, 0),
          totalLoad: Array.from(workers.values()).reduce((sum, w) => sum + w.currentLoad, 0)
        },
        tasks: {
          active: activeTasks.size,
          queued: yield* Queue.size(this.taskQueue),
          pending: yield* Queue.size(this.workerAssignments),
          failed: yield* Queue.size(this.failedTaskQueue),
          results: yield* Queue.size(this.resultQueue)
        }
      }
    })
}

// Usage example
const distributedTaskExample = Effect.gen(function* () {
  const coordinator = yield* DistributedTaskCoordinator.make()
  
  // Register workers
  yield* coordinator.registerWorker({
    id: "worker-1",
    capacity: 5,
    currentLoad: 0,
    lastHeartbeat: Date.now(),
    capabilities: ["cpu-intensive", "data-processing"],
    status: "active"
  })
  
  yield* coordinator.registerWorker({
    id: "worker-2",
    capacity: 3,
    currentLoad: 0,
    lastHeartbeat: Date.now(),
    capabilities: ["io-intensive", "data-processing"],
    status: "active"
  })
  
  // Submit tasks
  const taskTypes = ["cpu-intensive", "io-intensive", "data-processing"]
  for (let i = 0; i < 20; i++) {
    yield* coordinator.submitTask({
      id: `task-${i}`,
      type: taskTypes[i % taskTypes.length],
      priority: Math.floor(Math.random() * 5),
      data: { workload: i * 10 },
      maxAttempts: 3,
      timeout: 10000
    })
  }
  
  // Simulate task completion
  const taskSimulator = Effect.gen(function* () {
    yield* Effect.forever(
      Effect.gen(function* () {
        // Simulate random task completion
        if (Math.random() < 0.3) {
          const result: TaskResult = {
            taskId: `task-${Math.floor(Math.random() * 20)}`,
            workerId: Math.random() < 0.5 ? "worker-1" : "worker-2",
            success: Math.random() < 0.8,
            result: Math.random() < 0.8 ? { output: "success" } : undefined,
            error: Math.random() < 0.8 ? undefined : "Processing failed",
            duration: Math.random() * 5000,
            completedAt: Date.now()
          }
          
          yield* coordinator.completeTask(result)
        }
        
        yield* Effect.sleep("1 second")
      })
    )
  })
  
  const simulatorFiber = yield* Effect.fork(taskSimulator)
  
  // Monitor system
  const monitor = Effect.gen(function* () {
    yield* Effect.forever(
      Effect.gen(function* () {
        const metrics = yield* coordinator.getSystemMetrics()
        console.log("System Metrics:", JSON.stringify(metrics, null, 2))
        yield* Effect.sleep("5 seconds")
      })
    )
  })
  
  const monitorFiber = yield* Effect.fork(monitor)
  
  // Run for a while
  yield* Effect.sleep("30 seconds")
  
  yield* Fiber.interrupt(simulatorFiber)
  yield* Fiber.interrupt(monitorFiber)
  
  console.log("Distributed task coordination demo completed")
})
```

## Advanced Features Deep Dive

### Feature 1: Queue Composition and Routing

Building complex queue topologies for sophisticated data flow patterns.

#### Fan-out and Fan-in Patterns

```typescript
import { Effect, Queue, Fiber } from "effect"

interface Message {
  id: string
  type: string
  payload: any
  routing: string[]
}

class MessageRouter {
  private inputQueue: Queue.Queue<Message>
  private outputQueues: Map<string, Queue.Queue<Message>>
  
  constructor(
    inputQueue: Queue.Queue<Message>,
    outputQueues: Map<string, Queue.Queue<Message>>
  ) {
    this.inputQueue = inputQueue
    this.outputQueues = outputQueues
  }
  
  static make = (outputRoutes: string[]) =>
    Effect.gen(function* () {
      const inputQueue = yield* Queue.bounded<Message>(100)
      const outputQueues = new Map<string, Queue.Queue<Message>>()
      
      // Create output queues for each route
      for (const route of outputRoutes) {
        const queue = yield* Queue.bounded<Message>(50)
        outputQueues.set(route, queue)
      }
      
      const router = new MessageRouter(inputQueue, outputQueues)
      
      // Start routing
      yield* Effect.fork(router.routingProcessor())
      
      return router
    })
  
  send = (message: Message) =>
    Queue.offer(this.inputQueue, message)
  
  getQueue = (route: string) =>
    Effect.succeed(this.outputQueues.get(route))
  
  private routingProcessor = () =>
    Effect.gen(function* () {
      yield* Effect.forever(
        Effect.gen(function* () {
          const message = yield* Queue.take(this.inputQueue)
          
          // Fan-out: send to all specified routes
          const routingPromises = message.routing.map(route => {
            const queue = this.outputQueues.get(route)
            return queue ? Queue.offer(queue, message) : Effect.unit
          })
          
          yield* Effect.all(routingPromises, { concurrency: "unbounded" })
        })
      )
    })
}

// Usage example
const messageRoutingExample = Effect.gen(function* () {
  const router = yield* MessageRouter.make(["analytics", "billing", "notifications"])
  
  // Consumers for different routes
  const createConsumer = (route: string) =>
    Effect.gen(function* () {
      const queue = yield* router.getQueue(route)
      if (!queue) return
      
      yield* Effect.forever(
        Effect.gen(function* () {
          const message = yield* Queue.take(queue)
          console.log(`${route} processed message ${message.id}`)
          yield* Effect.sleep("100 millis")
        })
      )
    })
  
  // Start consumers
  const consumers = yield* Effect.all([
    Effect.fork(createConsumer("analytics")),
    Effect.fork(createConsumer("billing")),
    Effect.fork(createConsumer("notifications"))
  ])
  
  // Send messages with different routing
  const messages: Message[] = [
    { id: "msg-1", type: "order", payload: {}, routing: ["analytics", "billing"] },
    { id: "msg-2", type: "signup", payload: {}, routing: ["analytics", "notifications"] },
    { id: "msg-3", type: "payment", payload: {}, routing: ["billing"] },
    { id: "msg-4", type: "error", payload: {}, routing: ["analytics", "notifications"] }
  ]
  
  for (const message of messages) {
    yield* router.send(message)
  }
  
  yield* Effect.sleep("2 seconds")
  yield* Effect.all(consumers.map(fiber => Fiber.interrupt(fiber)))
})
```

#### Priority Queues

```typescript
import { Effect, Queue, Ref } from "effect"

interface PriorityItem<T> {
  item: T
  priority: number
  insertOrder: number
}

class PriorityQueue<T> {
  private items: Ref.Ref<PriorityItem<T>[]>
  private insertCounter: Ref.Ref<number>
  private waitingTakers: Ref.Ref<Array<(item: T) => void>>
  
  constructor(
    items: Ref.Ref<PriorityItem<T>[]>,
    insertCounter: Ref.Ref<number>,
    waitingTakers: Ref.Ref<Array<(item: T) => void>>
  ) {
    this.items = items
    this.insertCounter = insertCounter
    this.waitingTakers = waitingTakers
  }
  
  static make = <T>() =>
    Effect.gen(function* () {
      const items = yield* Ref.make<PriorityItem<T>[]>([])
      const insertCounter = yield* Ref.make(0)
      const waitingTakers = yield* Ref.make<Array<(item: T) => void>>([])
      
      return new PriorityQueue(items, insertCounter, waitingTakers)
    })
  
  offer = (item: T, priority: number) =>
    Effect.gen(function* () {
      const insertOrder = yield* Ref.updateAndGet(this.insertCounter, n => n + 1)
      const priorityItem: PriorityItem<T> = { item, priority, insertOrder }
      
      yield* Ref.update(this.items, items => {
        const newItems = [...items, priorityItem]
        // Sort by priority (higher first), then by insertion order
        newItems.sort((a, b) => {
          if (a.priority !== b.priority) {
            return b.priority - a.priority
          }
          return a.insertOrder - b.insertOrder
        })
        return newItems
      })
      
      // Notify waiting takers
      const takers = yield* Ref.get(this.waitingTakers)
      if (takers.length > 0) {
        const taker = takers[0]
        yield* Ref.update(this.waitingTakers, t => t.slice(1))
        
        const topItem = yield* Ref.updateAndGet(this.items, items => {
          if (items.length > 0) {
            return items.slice(1)
          }
          return items
        })
        
        if (topItem.length < (yield* Ref.get(this.items)).length + 1) {
          taker(priorityItem.item)
        }
      }
    })
  
  take = () =>
    Effect.gen(function* () {
      const items = yield* Ref.get(this.items)
      
      if (items.length > 0) {
        const topItem = items[0]
        yield* Ref.update(this.items, items => items.slice(1))
        return topItem.item
      } else {
        // Wait for an item
        return yield* Effect.async<T>((resume) => {
          Ref.update(this.waitingTakers, takers => [...takers, resume]).pipe(
            Effect.runSync
          )
        })
      }
    })
  
  size = () =>
    Effect.gen(function* () {
      const items = yield* Ref.get(this.items)
      return items.length
    })
}

// Usage example
const priorityQueueExample = Effect.gen(function* () {
  const pQueue = yield* PriorityQueue.make<string>()
  
  // Consumer
  const consumer = Effect.gen(function* () {
    yield* Effect.forever(
      Effect.gen(function* () {
        const item = yield* pQueue.take()
        console.log(`Processing: ${item}`)
        yield* Effect.sleep("200 millis")
      })
    )
  })
  
  const consumerFiber = yield* Effect.fork(consumer)
  
  // Producer with different priorities
  yield* pQueue.offer("Low priority task 1", 1)
  yield* pQueue.offer("High priority task 1", 5)
  yield* pQueue.offer("Medium priority task 1", 3)
  yield* pQueue.offer("High priority task 2", 5)
  yield* pQueue.offer("Low priority task 2", 1)
  yield* pQueue.offer("Medium priority task 2", 3)
  
  yield* Effect.sleep("3 seconds")
  yield* Fiber.interrupt(consumerFiber)
})
```

### Feature 2: Queue Monitoring and Observability

Comprehensive monitoring for queue health and performance.

#### Queue Metrics and Health Monitoring

```typescript
import { Effect, Queue, Ref, Metric, Schedule } from "effect"

interface QueueMetrics {
  totalOffered: number
  totalTaken: number
  currentSize: number
  maxSizeReached: number
  averageWaitTime: number
  throughputPerSecond: number
  lastResetTime: number
}

class MonitoredQueue<T> {
  private queue: Queue.Queue<T>
  private metrics: Ref.Ref<QueueMetrics>
  private offerTimes: Ref.Ref<Map<string, number>>
  
  // Metrics
  private sizeGauge: Metric.Metric.Gauge<number>
  private throughputCounter: Metric.Metric.Counter<number>
  private waitTimeHistogram: Metric.Metric.Histogram<number>
  
  constructor(
    queue: Queue.Queue<T>,
    metrics: Ref.Ref<QueueMetrics>,
    offerTimes: Ref.Ref<Map<string, number>>,
    sizeGauge: Metric.Metric.Gauge<number>,
    throughputCounter: Metric.Metric.Counter<number>,
    waitTimeHistogram: Metric.Metric.Histogram<number>
  ) {
    this.queue = queue
    this.metrics = metrics
    this.offerTimes = offerTimes
    this.sizeGauge = sizeGauge
    this.throughputCounter = throughputCounter
    this.waitTimeHistogram = waitTimeHistogram
  }
  
  static make = <T>(capacity: number, name: string) =>
    Effect.gen(function* () {
      const queue = yield* Queue.bounded<T>(capacity)
      const metrics = yield* Ref.make<QueueMetrics>({
        totalOffered: 0,
        totalTaken: 0,
        currentSize: 0,
        maxSizeReached: 0,
        averageWaitTime: 0,
        throughputPerSecond: 0,
        lastResetTime: Date.now()
      })
      const offerTimes = yield* Ref.make(new Map<string, number>())
      
      // Create metrics
      const sizeGauge = Metric.gauge(`${name}_queue_size`)
      const throughputCounter = Metric.counter(`${name}_throughput`)
      const waitTimeHistogram = Metric.histogram(`${name}_wait_time_ms`, 
        Metric.Histogram.Boundaries.exponential({
          start: 1,
          factor: 2,
          count: 10
        })
      )
      
      const monitoredQueue = new MonitoredQueue(
        queue, metrics, offerTimes, sizeGauge, throughputCounter, waitTimeHistogram
      )
      
      // Start monitoring
      yield* Effect.fork(monitoredQueue.monitoringLoop())
      
      return monitoredQueue
    })
  
  offer = (item: T) =>
    Effect.gen(function* () {
      const itemId = Math.random().toString(36).substr(2, 9)
      const offerTime = Date.now()
      
      yield* Ref.update(this.offerTimes, times => new Map(times.set(itemId, offerTime)))
      
      const wrappedItem = { item, itemId } as any
      yield* Queue.offer(this.queue, wrappedItem)
      
      // Update metrics
      yield* Ref.update(this.metrics, m => ({
        ...m,
        totalOffered: m.totalOffered + 1
      }))
      
      yield* Metric.increment(this.throughputCounter)
    })
  
  take = (): Effect.Effect<T> =>
    Effect.gen(function* () {
      const wrappedItem = yield* Queue.take(this.queue) as any
      const takeTime = Date.now()
      
      // Calculate wait time
      const offerTimes = yield* Ref.get(this.offerTimes)
      const offerTime = offerTimes.get(wrappedItem.itemId)
      
      if (offerTime) {
        const waitTime = takeTime - offerTime
        yield* Metric.update(this.waitTimeHistogram, waitTime)
        
        // Update average wait time
        yield* Ref.update(this.metrics, m => {
          const newAverage = (m.averageWaitTime * m.totalTaken + waitTime) / (m.totalTaken + 1)
          return {
            ...m,
            totalTaken: m.totalTaken + 1,
            averageWaitTime: newAverage
          }
        })
        
        // Clean up offer time tracking
        yield* Ref.update(this.offerTimes, times => {
          const newTimes = new Map(times)
          newTimes.delete(wrappedItem.itemId)
          return newTimes
        })
      }
      
      return wrappedItem.item
    })
  
  private monitoringLoop = () =>
    Effect.gen(function* () {
      yield* Effect.forever(
        Effect.gen(function* () {
          const size = yield* Queue.size(this.queue)
          
          // Update size gauge
          yield* Metric.set(this.sizeGauge, size)
          
          // Update metrics
          yield* Ref.update(this.metrics, m => ({
            ...m,
            currentSize: size,
            maxSizeReached: Math.max(m.maxSizeReached, size)
          }))
          
          yield* Effect.sleep("1 second")
        })
      )
    })
  
  getMetrics = () => Ref.get(this.metrics)
  
  resetMetrics = () =>
    Effect.gen(function* () {
      yield* Ref.set(this.metrics, {
        totalOffered: 0,
        totalTaken: 0,
        currentSize: yield* Queue.size(this.queue),
        maxSizeReached: 0,
        averageWaitTime: 0,
        throughputPerSecond: 0,
        lastResetTime: Date.now()
      })
    })
  
  // Health check
  getHealthStatus = () =>
    Effect.gen(function* () {
      const metrics = yield* this.getMetrics()
      const currentSize = yield* Queue.size(this.queue)
      
      // Define health criteria
      const isHealthy = 
        currentSize < 100 && // Not too backlogged
        metrics.averageWaitTime < 5000 && // Wait time under 5 seconds
        (Date.now() - metrics.lastResetTime < 300000 || metrics.totalTaken > 0) // Active within 5 minutes
      
      return {
        healthy: isHealthy,
        metrics,
        issues: [
          ...(currentSize >= 100 ? ["Queue backlogged"] : []),
          ...(metrics.averageWaitTime >= 5000 ? ["High wait times"] : []),
          ...(Date.now() - metrics.lastResetTime >= 300000 && metrics.totalTaken === 0 ? ["No activity"] : [])
        ]
      }
    })
}

// Usage example
const monitoredQueueExample = Effect.gen(function* () {
  const monitoredQueue = yield* MonitoredQueue.make<string>(50, "work_queue")
  
  // Producer
  const producer = Effect.gen(function* () {
    for (let i = 0; i < 100; i++) {
      yield* monitoredQueue.offer(`task-${i}`)
      yield* Effect.sleep(`${Math.random() * 100}ms`)
    }
  })
  
  // Consumer
  const consumer = Effect.gen(function* () {
    yield* Effect.forever(
      Effect.gen(function* () {
        const task = yield* monitoredQueue.take()
        console.log(`Processing: ${task}`)
        yield* Effect.sleep(`${100 + Math.random() * 200}ms`)
      })
    )
  })
  
  // Health monitor
  const healthMonitor = Effect.gen(function* () {
    yield* Effect.forever(
      Effect.gen(function* () {
        const health = yield* monitoredQueue.getHealthStatus()
        console.log(`Queue Health: ${health.healthy ? "HEALTHY" : "UNHEALTHY"}`)
        if (!health.healthy) {
          console.log("Issues:", health.issues)
        }
        console.log("Metrics:", health.metrics)
        
        yield* Effect.sleep("5 seconds")
      })
    )
  })
  
  // Start all
  const producerFiber = yield* Effect.fork(producer)
  const consumerFiber = yield* Effect.fork(consumer)
  const healthFiber = yield* Effect.fork(healthMonitor)
  
  yield* Fiber.join(producerFiber)
  yield* Effect.sleep("5 seconds")
  
  yield* Fiber.interrupt(consumerFiber)
  yield* Fiber.interrupt(healthFiber)
})
```

### Feature 3: Queue Persistence and Recovery

Implementing durable queues that survive application restarts.

#### Persistent Queue with File System

```typescript
import { Effect, Queue, Ref, Schedule } from "effect"
import * as fs from "fs/promises"

interface PersistedItem<T> {
  id: string
  data: T
  timestamp: number
}

interface PersistenceConfig {
  filePath: string
  syncInterval: number
  maxMemoryItems: number
}

class PersistentQueue<T> {
  private memoryQueue: Queue.Queue<PersistedItem<T>>
  private persistedItems: Ref.Ref<PersistedItem<T>[]>
  private config: PersistenceConfig
  private nextId: Ref.Ref<number>
  
  constructor(
    memoryQueue: Queue.Queue<PersistedItem<T>>,
    persistedItems: Ref.Ref<PersistedItem<T>[]>,
    config: PersistenceConfig,
    nextId: Ref.Ref<number>
  ) {
    this.memoryQueue = memoryQueue
    this.persistedItems = persistedItems
    this.config = config
    this.nextId = nextId
  }
  
  static make = <T>(config: PersistenceConfig) =>
    Effect.gen(function* () {
      const memoryQueue = yield* Queue.bounded<PersistedItem<T>>(config.maxMemoryItems)
      const persistedItems = yield* Ref.make<PersistedItem<T>[]>([])
      const nextId = yield* Ref.make(0)
      
      const persistentQueue = new PersistentQueue(memoryQueue, persistedItems, config, nextId)
      
      // Load existing items
      yield* persistentQueue.loadFromDisk()
      
      // Start persistence worker
      yield* Effect.fork(persistentQueue.persistenceWorker())
      
      return persistentQueue
    })
  
  offer = (item: T) =>
    Effect.gen(function* () {
      const id = yield* Ref.updateAndGet(this.nextId, n => n + 1)
      const persistedItem: PersistedItem<T> = {
        id: id.toString(),
        data: item,
        timestamp: Date.now()
      }
      
      // Try to add to memory queue first
      const offered = yield* Queue.offer(this.memoryQueue, persistedItem).pipe(
        Effect.timeout("100 millis"),
        Effect.option
      )
      
      if (offered._tag === "None") {
        // Memory queue full, add directly to persistence
        yield* Ref.update(this.persistedItems, items => [...items, persistedItem])
        yield* this.syncToDisk()
      }
    })
  
  take = (): Effect.Effect<T> =>
    Effect.gen(function* () {
      // Try memory queue first
      const memoryItem = yield* Queue.poll(this.memoryQueue)
      
      if (memoryItem._tag === "Some") {
        return memoryItem.value.data
      }
      
      // Check persisted items
      const persistedItems = yield* Ref.get(this.persistedItems)
      if (persistedItems.length > 0) {
        const item = persistedItems[0]
        yield* Ref.update(this.persistedItems, items => items.slice(1))
        yield* this.syncToDisk()
        return item.data
      }
      
      // Wait for new item
      const item = yield* Queue.take(this.memoryQueue)
      return item.data
    })
  
  private loadFromDisk = () =>
    Effect.gen(function* () {
      try {
        const data = yield* Effect.promise(() => fs.readFile(this.config.filePath, 'utf-8'))
        const items: PersistedItem<T>[] = JSON.parse(data)
        
        // Load items into memory queue and persistence
        const memoryItems = items.slice(0, this.config.maxMemoryItems)
        const remainingItems = items.slice(this.config.maxMemoryItems)
        
        for (const item of memoryItems) {
          yield* Queue.offer(this.memoryQueue, item)
        }
        
        yield* Ref.set(this.persistedItems, remainingItems)
        
        // Update next ID
        const maxId = items.reduce((max, item) => 
          Math.max(max, parseInt(item.id) || 0), 0
        )
        yield* Ref.set(this.nextId, maxId + 1)
        
        console.log(`Loaded ${items.length} items from disk`)
      } catch (error) {
        if ((error as any).code !== 'ENOENT') {
          console.error('Failed to load from disk:', error)
        }
      }
    })
  
  private syncToDisk = () =>
    Effect.gen(function* () {
      const memoryItems: PersistedItem<T>[] = []
      
      // Drain memory queue to get items
      while (true) {
        const item = yield* Queue.poll(this.memoryQueue)
        if (item._tag === "None") break
        memoryItems.push(item.value)
      }
      
      // Restore items to memory queue
      for (const item of memoryItems) {
        yield* Queue.offer(this.memoryQueue, item)
      }
      
      // Combine with persisted items
      const persistedItems = yield* Ref.get(this.persistedItems)
      const allItems = [...memoryItems, ...persistedItems]
      
      // Write to disk
      yield* Effect.promise(() => 
        fs.writeFile(this.config.filePath, JSON.stringify(allItems, null, 2))
      )
    })
  
  private persistenceWorker = () =>
    Effect.gen(function* () {
      yield* Effect.repeat(
        this.syncToDisk(),
        Schedule.fixed(`${this.config.syncInterval}ms`)
      )
    })
  
  size = () =>
    Effect.gen(function* () {
      const memorySize = yield* Queue.size(this.memoryQueue)
      const persistedItems = yield* Ref.get(this.persistedItems)
      return memorySize + persistedItems.length
    })
  
  shutdown = () =>
    Effect.gen(function* () {
      // Final sync
      yield* this.syncToDisk()
      yield* Queue.shutdown(this.memoryQueue)
    })
}

// Usage example
const persistentQueueExample = Effect.gen(function* () {
  const queue = yield* PersistentQueue.make<string>({
    filePath: './queue-data.json',
    syncInterval: 5000,
    maxMemoryItems: 100
  })
  
  // Producer
  const producer = Effect.gen(function* () {
    for (let i = 0; i < 50; i++) {
      yield* queue.offer(`persistent-item-${i}`)
      console.log(`Offered item ${i}`)
      yield* Effect.sleep("200 millis")
    }
  })
  
  // Consumer
  const consumer = Effect.gen(function* () {
    yield* Effect.forever(
      Effect.gen(function* () {
        const item = yield* queue.take()
        console.log(`Consumed: ${item}`)
        yield* Effect.sleep("300 millis")
      })
    )
  })
  
  const producerFiber = yield* Effect.fork(producer)
  const consumerFiber = yield* Effect.fork(consumer)
  
  yield* Fiber.join(producerFiber)
  yield* Effect.sleep("5 seconds")
  
  yield* Fiber.interrupt(consumerFiber)
  yield* queue.shutdown()
  
  console.log("Persistent queue demo completed")
})
```

## Practical Patterns & Best Practices

### Pattern 1: Graceful Degradation with Queue Overflow

```typescript
import { Effect, Queue, Ref } from "effect"

interface QueueWithFallback<T> {
  primaryQueue: Queue.Queue<T>
  fallbackHandler: (item: T) => Effect.Effect<void>
  degradationMetrics: Ref.Ref<{
    totalOffers: number
    fallbackCount: number
    lastFallbackTime: number
  }>
}

const createQueueWithFallback = <T>(
  primaryCapacity: number,
  fallbackHandler: (item: T) => Effect.Effect<void>
) =>
  Effect.gen(function* () {
    const primaryQueue = yield* Queue.bounded<T>(primaryCapacity)
    const degradationMetrics = yield* Ref.make({
      totalOffers: 0,
      fallbackCount: 0,
      lastFallbackTime: 0
    })
    
    return {
      primaryQueue,
      fallbackHandler,
      degradationMetrics
    }
  })

const offerWithFallback = <T>(
  queueWithFallback: QueueWithFallback<T>,
  item: T,
  timeoutMs: number = 100
) =>
  Effect.gen(function* () {
    yield* Ref.update(queueWithFallback.degradationMetrics, m => ({
      ...m,
      totalOffers: m.totalOffers + 1
    }))
    
    // Try primary queue with timeout
    const offered = yield* Queue.offer(queueWithFallback.primaryQueue, item).pipe(
      Effect.timeout(`${timeoutMs}ms`),
      Effect.option
    )
    
    if (offered._tag === "None") {
      // Primary queue full or slow, use fallback
      yield* Ref.update(queueWithFallback.degradationMetrics, m => ({
        ...m,
        fallbackCount: m.fallbackCount + 1,
        lastFallbackTime: Date.now()
      }))
      
      yield* queueWithFallback.fallbackHandler(item)
      console.log("Used fallback handler for item")
    }
  })

// Usage example
const gracefulDegradationExample = Effect.gen(function* () {
  // Fallback: log to file or external service
  const fallbackHandler = (item: string) =>
    Effect.gen(function* () {
      console.log(`FALLBACK: ${item}`)
      // In real scenario: write to file, send to external queue, etc.
      yield* Effect.sleep("10 millis")
    })
  
  const queueWithFallback = yield* createQueueWithFallback(5, fallbackHandler)
  
  // Fast producer
  const producer = Effect.gen(function* () {
    for (let i = 0; i < 20; i++) {
      yield* offerWithFallback(queueWithFallback, `item-${i}`, 50)
      yield* Effect.sleep("10 millis")
    }
  })
  
  // Slow consumer
  const consumer = Effect.gen(function* () {
    yield* Effect.forever(
      Effect.gen(function* () {
        const item = yield* Queue.take(queueWithFallback.primaryQueue)
        console.log(`Primary consumed: ${item}`)
        yield* Effect.sleep("200 millis")
      })
    )
  })
  
  const producerFiber = yield* Effect.fork(producer)
  const consumerFiber = yield* Effect.fork(consumer)
  
  yield* Fiber.join(producerFiber)
  yield* Effect.sleep("2 seconds")
  
  const metrics = yield* Ref.get(queueWithFallback.degradationMetrics)
  console.log("Degradation metrics:", metrics)
  
  yield* Fiber.interrupt(consumerFiber)
})
```

### Pattern 2: Queue-Based Rate Limiting

```typescript
import { Effect, Queue, Ref, Schedule } from "effect"

interface RateLimitedQueue<T> {
  requestQueue: Queue.Queue<T>
  outputQueue: Queue.Queue<T>
  rateLimit: number
  windowMs: number
}

const createRateLimitedQueue = <T>(
  capacity: number,
  rateLimit: number,
  windowMs: number = 1000
) =>
  Effect.gen(function* () {
    const requestQueue = yield* Queue.bounded<T>(capacity)
    const outputQueue = yield* Queue.bounded<T>(capacity)
    
    const rateLimitedQueue: RateLimitedQueue<T> = {
      requestQueue,
      outputQueue,
      rateLimit,
      windowMs
    }
    
    // Start rate limiting processor
    yield* Effect.fork(rateLimitingProcessor(rateLimitedQueue))
    
    return rateLimitedQueue
  })

const rateLimitingProcessor = <T>(queue: RateLimitedQueue<T>) =>
  Effect.gen(function* () {
    const processedInWindow = yield* Ref.make(0)
    const windowStart = yield* Ref.make(Date.now())
    
    yield* Effect.forever(
      Effect.gen(function* () {
        const now = Date.now()
        const currentWindowStart = yield* Ref.get(windowStart)
        
        // Reset window if needed
        if (now - currentWindowStart >= queue.windowMs) {
          yield* Ref.set(windowStart, now)
          yield* Ref.set(processedInWindow, 0)
        }
        
        const currentProcessed = yield* Ref.get(processedInWindow)
        
        if (currentProcessed < queue.rateLimit) {
          // Can process more items in this window
          const item = yield* Queue.poll(queue.requestQueue)
          
          if (item._tag === "Some") {
            yield* Queue.offer(queue.outputQueue, item.value)
            yield* Ref.update(processedInWindow, n => n + 1)
          } else {
            // No items to process, small delay
            yield* Effect.sleep("10 millis")
          }
        } else {
          // Rate limit reached, wait for next window
          const remainingWindow = queue.windowMs - (now - currentWindowStart)
          yield* Effect.sleep(`${Math.max(remainingWindow, 10)}ms`)
        }
      })
    )
  })

// Usage example
const rateLimitingExample = Effect.gen(function* () {
  const rateLimitedQueue = yield* createRateLimitedQueue<string>(100, 5, 1000) // 5 per second
  
  // Producer - tries to send 20 items quickly
  const producer = Effect.gen(function* () {
    for (let i = 0; i < 20; i++) {
      yield* Queue.offer(rateLimitedQueue.requestQueue, `request-${i}`)
      console.log(`Queued request-${i}`)
      yield* Effect.sleep("50 millis")
    }
  })
  
  // Consumer - processes rate-limited items
  const consumer = Effect.gen(function* () {
    yield* Effect.forever(
      Effect.gen(function* () {
        const item = yield* Queue.take(rateLimitedQueue.outputQueue)
        console.log(`Processed: ${item} at ${new Date().toISOString()}`)
      })
    )
  })
  
  const producerFiber = yield* Effect.fork(producer)
  const consumerFiber = yield* Effect.fork(consumer)
  
  yield* Fiber.join(producerFiber)
  yield* Effect.sleep("10 seconds") // Watch rate limiting in action
  
  yield* Fiber.interrupt(consumerFiber)
})
```

### Pattern 3: Dead Letter Queue Pattern

```typescript
import { Effect, Queue, Ref, Schedule } from "effect"

interface ProcessingResult<T> {
  item: T
  success: boolean
  error?: string
  attempts: number
}

interface DeadLetterQueueSystem<T> {
  mainQueue: Queue.Queue<T>
  retryQueue: Queue.Queue<{ item: T, attempts: number }>
  deadLetterQueue: Queue.Queue<T>
  maxRetries: number
  processor: (item: T) => Effect.Effect<void>
}

const createDeadLetterQueueSystem = <T>(
  capacity: number,
  maxRetries: number,
  processor: (item: T) => Effect.Effect<void>
) =>
  Effect.gen(function* () {
    const mainQueue = yield* Queue.bounded<T>(capacity)
    const retryQueue = yield* Queue.bounded<{ item: T, attempts: number }>(capacity)
    const deadLetterQueue = yield* Queue.bounded<T>(capacity / 2)
    
    const system: DeadLetterQueueSystem<T> = {
      mainQueue,
      retryQueue,
      deadLetterQueue,
      maxRetries,
      processor
    }
    
    // Start processors
    yield* Effect.fork(mainProcessor(system))
    yield* Effect.fork(retryProcessor(system))
    
    return system
  })

const mainProcessor = <T>(system: DeadLetterQueueSystem<T>) =>
  Effect.gen(function* () {
    yield* Effect.forever(
      Effect.gen(function* () {
        const item = yield* Queue.take(system.mainQueue)
        
        const result = yield* system.processor(item).pipe(
          Effect.either,
          Effect.map(either => ({
            item,
            success: either._tag === "Right",
            error: either._tag === "Left" ? either.left.message : undefined,
            attempts: 1
          }))
        )
        
        if (!result.success && result.attempts <= system.maxRetries) {
          // Send to retry queue
          yield* Queue.offer(system.retryQueue, { 
            item: result.item, 
            attempts: result.attempts 
          })
          console.log(`Retry queued for item (attempt ${result.attempts})`)
        } else if (!result.success) {
          // Send to dead letter queue
          yield* Queue.offer(system.deadLetterQueue, result.item)
          console.error(`Item sent to dead letter queue after ${result.attempts} attempts`)
        } else {
          console.log(`Item processed successfully`)
        }
      })
    )
  })

const retryProcessor = <T>(system: DeadLetterQueueSystem<T>) =>
  Effect.gen(function* () {
    yield* Effect.forever(
      Effect.gen(function* () {
        const retryItem = yield* Queue.take(system.retryQueue)
        
        // Exponential backoff
        const backoffMs = Math.min(1000 * Math.pow(2, retryItem.attempts - 1), 30000)
        yield* Effect.sleep(`${backoffMs}ms`)
        
        const result = yield* system.processor(retryItem.item).pipe(
          Effect.either,
          Effect.map(either => ({
            item: retryItem.item,
            success: either._tag === "Right",
            error: either._tag === "Left" ? either.left.message : undefined,
            attempts: retryItem.attempts + 1
          }))
        )
        
        if (!result.success && result.attempts <= system.maxRetries) {
          // Retry again
          yield* Queue.offer(system.retryQueue, { 
            item: result.item, 
            attempts: result.attempts 
          })
          console.log(`Retry queued again for item (attempt ${result.attempts})`)
        } else if (!result.success) {
          // Give up, send to dead letter queue
          yield* Queue.offer(system.deadLetterQueue, result.item)
          console.error(`Item sent to dead letter queue after ${result.attempts} attempts`)
        } else {
          console.log(`Item processed successfully on retry`)
        }
      })
    )
  })

// Usage example
const deadLetterQueueExample = Effect.gen(function* () {
  // Flaky processor that fails randomly
  const flakyProcessor = (item: string) =>
    Effect.gen(function* () {
      console.log(`Processing ${item}`)
      yield* Effect.sleep("100 millis")
      
      if (Math.random() < 0.6) { // 60% failure rate
        yield* Effect.fail(new Error(`Random failure for ${item}`))
      }
    })
  
  const system = yield* createDeadLetterQueueSystem(50, 3, flakyProcessor)
  
  // Producer
  const producer = Effect.gen(function* () {
    for (let i = 0; i < 10; i++) {
      yield* Queue.offer(system.mainQueue, `task-${i}`)
      yield* Effect.sleep("200 millis")
    }
  })
  
  // Dead letter handler
  const deadLetterHandler = Effect.gen(function* () {
    yield* Effect.forever(
      Effect.gen(function* () {
        const deadItem = yield* Queue.take(system.deadLetterQueue)
        console.log(`DEAD LETTER: ${deadItem} - manual intervention required`)
        
        // In real scenario: log to monitoring system, create alerts, etc.
      })
    )
  })
  
  const producerFiber = yield* Effect.fork(producer)
  const deadLetterFiber = yield* Effect.fork(deadLetterHandler)
  
  yield* Fiber.join(producerFiber)
  yield* Effect.sleep("10 seconds") // Let retries finish
  
  yield* Fiber.interrupt(deadLetterFiber)
  
  const deadLetterSize = yield* Queue.size(system.deadLetterQueue)
  console.log(`Final dead letter queue size: ${deadLetterSize}`)
})
```

## Integration Examples

### Integration with WebSocket Connections

```typescript
import { Effect, Queue, Fiber } from "effect"
import WebSocket from "ws"

interface WebSocketMessage {
  type: string
  data: any
  clientId: string
  timestamp: number
}

const createWebSocketHandler = (port: number) =>
  Effect.gen(function* () {
    const incomingQueue = yield* Queue.bounded<WebSocketMessage>(1000)
    const outgoingQueue = yield* Queue.bounded<WebSocketMessage>(1000)
    const clients = yield* Ref.make(new Map<string, WebSocket>())
    
    const wss = new WebSocket.Server({ port })
    
    // Handle new connections
    wss.on('connection', (ws, req) => {
      const clientId = Math.random().toString(36).substr(2, 9)
      
      Effect.runPromise(
        Effect.gen(function* () {
          yield* Ref.update(clients, c => new Map(c.set(clientId, ws)))
          console.log(`Client ${clientId} connected`)
          
          ws.on('message', (data) => {
            const message: WebSocketMessage = {
              type: 'incoming',
              data: JSON.parse(data.toString()),
              clientId,
              timestamp: Date.now()
            }
            
            Effect.runPromise(Queue.offer(incomingQueue, message))
          })
          
          ws.on('close', () => {
            Effect.runPromise(
              Ref.update(clients, c => {
                const updated = new Map(c)
                updated.delete(clientId)
                return updated
              })
            )
            console.log(`Client ${clientId} disconnected`)
          })
        })
      )
    })
    
    // Outgoing message processor
    const outgoingProcessor = Effect.gen(function* () {
      yield* Effect.forever(
        Effect.gen(function* () {
          const message = yield* Queue.take(outgoingQueue)
          const clientMap = yield* Ref.get(clients)
          const client = clientMap.get(message.clientId)
          
          if (client && client.readyState === WebSocket.OPEN) {
            client.send(JSON.stringify(message.data))
          }
        })
      )
    })
    
    yield* Effect.fork(outgoingProcessor)
    
    return {
      incomingQueue,
      outgoingQueue,
      broadcast: (data: any) =>
        Effect.gen(function* () {
          const clientMap = yield* Ref.get(clients)
          yield* Effect.all(
            Array.from(clientMap.entries()).map(([clientId, ws]) =>
              Queue.offer(outgoingQueue, {
                type: 'broadcast',
                data,
                clientId,
                timestamp: Date.now()
              })
            )
          )
        })
    }
  })

// Usage
const webSocketExample = Effect.gen(function* () {
  const handler = yield* createWebSocketHandler(8080)
  
  // Message processor
  const messageProcessor = Effect.gen(function* () {
    yield* Effect.forever(
      Effect.gen(function* () {
        const message = yield* Queue.take(handler.incomingQueue)
        console.log(`Processing message from ${message.clientId}:`, message.data)
        
        // Echo back
        yield* Queue.offer(handler.outgoingQueue, {
          type: 'echo',
          data: { echo: message.data },
          clientId: message.clientId,
          timestamp: Date.now()
        })
      })
    )
  })
  
  yield* Effect.fork(messageProcessor)
  
  console.log("WebSocket server started on port 8080")
  yield* Effect.sleep("30 seconds")
})
```

### Testing Strategies

```typescript
import { Effect, Queue, TestContext, TestClock, Ref } from "effect"
import { describe, it, expect } from "@effect/vitest"

describe("Queue Testing", () => {
  it("should handle backpressure correctly", () =>
    Effect.gen(function* () {
      const queue = yield* Queue.bounded<number>(2)
      const offered = yield* Ref.make<number[]>([])
      
      // Fill queue to capacity
      yield* Queue.offer(queue, 1)
      yield* Queue.offer(queue, 2)
      
      // This should not complete immediately due to backpressure
      const fiber = yield* Effect.fork(
        Effect.gen(function* () {
          yield* Queue.offer(queue, 3)
          yield* Ref.update(offered, arr => [...arr, 3])
        })
      )
      
      // Verify offer is blocked
      yield* TestClock.adjust("100 millis")
      const offeredSoFar = yield* Ref.get(offered)
      expect(offeredSoFar).toEqual([])
      
      // Make space in queue
      const taken = yield* Queue.take(queue)
      expect(taken).toBe(1)
      
      // Now the blocked offer should complete
      yield* Fiber.join(fiber)
      const finalOffered = yield* Ref.get(offered)
      expect(finalOffered).toEqual([3])
    }).pipe(Effect.provide(TestContext.TestContext))
  )
  
  it("should test producer-consumer scenarios", () =>
    Effect.gen(function* () {
      const queue = yield* Queue.bounded<string>(5)
      const results = yield* Ref.make<string[]>([])
      
      // Producer
      const producer = Effect.gen(function* () {
        for (let i = 0; i < 5; i++) {
          yield* Queue.offer(queue, `item-${i}`)
          yield* TestClock.adjust("100 millis")
        }
      })
      
      // Consumer
      const consumer = Effect.gen(function* () {
        for (let i = 0; i < 5; i++) {
          const item = yield* Queue.take(queue)
          yield* Ref.update(results, arr => [...arr, item])
          yield* TestClock.adjust("150 millis")
        }
      })
      
      // Run concurrently
      yield* Effect.all([producer, consumer], { concurrency: "unbounded" })
      
      const finalResults = yield* Ref.get(results)
      expect(finalResults).toEqual([
        "item-0", "item-1", "item-2", "item-3", "item-4"
      ])
    }).pipe(Effect.provide(TestContext.TestContext))
  )
  
  it("should test queue shutdown behavior", () =>
    Effect.gen(function* () {
      const queue = yield* Queue.bounded<number>(10)
      const shutdownCaught = yield* Ref.make(false)
      
      // Consumer that will be interrupted by shutdown
      const consumer = Effect.gen(function* () {
        yield* Queue.take(queue) // This will be interrupted
      }).pipe(
        Effect.catchAll(() => Ref.set(shutdownCaught, true))
      )
      
      const consumerFiber = yield* Effect.fork(consumer)
      
      // Shutdown queue while consumer is waiting
      yield* Queue.shutdown(queue)
      yield* Fiber.join(consumerFiber)
      
      const wasCaught = yield* Ref.get(shutdownCaught)
      expect(wasCaught).toBe(true)
    })
  )
})

// Property-based testing
const testQueueProperties = Effect.gen(function* () {
  // Test FIFO property
  const queue = yield* Queue.bounded<number>(100)
  const items = Array.from({ length: 50 }, (_, i) => i)
  
  // Offer all items
  yield* Effect.all(items.map(item => Queue.offer(queue, item)))
  
  // Take all items and verify order
  const results = []
  for (let i = 0; i < items.length; i++) {
    const item = yield* Queue.take(queue)
    results.push(item)
  }
  
  expect(results).toEqual(items)
})
```

Queue provides essential building blocks for asynchronous communication in Effect applications. By embracing type-safe, backpressure-aware queues, you get predictable flow control and robust producer-consumer patterns.

Key benefits:
- **Built-in Backpressure**: Automatic flow control prevents overwhelming consumers
- **Type Safety**: Full TypeScript integration with compile-time guarantees
- **Graceful Shutdown**: Coordinated cleanup that handles all waiting fibers
- **Composable Patterns**: Build complex data flows from simple queue operations
- **Testing Support**: Comprehensive testing utilities for queue behavior

Use Queue when you need producer-consumer communication, work distribution, buffering, rate limiting, or any scenario involving asynchronous data flow between different parts of your application.