# Fiber: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem Fiber Solves

Modern JavaScript applications need to handle concurrent operations effectively - multiple API calls, background processing, real-time updates, and user interactions. Traditional approaches using Promises and async/await quickly become unwieldy when you need:

```typescript
// Traditional approach - managing concurrent operations with Promises
async function processOrdersConcurrently(orders: Order[]): Promise<ProcessedOrder[]> {
  const results: ProcessedOrder[] = []
  const errors: Error[] = []
  
  // Process orders concurrently but how do we control concurrency?
  const promises = orders.map(async (order) => {
    try {
      // What if we need to cancel this operation?
      const validated = await validateOrder(order)
      const processed = await processPayment(validated)
      const shipped = await scheduleShipping(processed)
      return shipped
    } catch (error) {
      errors.push(error as Error)
      throw error
    }
  })
  
  // All-or-nothing approach - one failure kills everything
  try {
    return await Promise.all(promises)
  } catch (error) {
    // How do we handle partial failures?
    // How do we cancel remaining operations?
    // How do we clean up resources?
    throw new Error(`Failed to process orders: ${errors.length} errors`)
  }
}

// What about graceful shutdown?
async function backgroundProcessor(): Promise<void> {
  let shouldStop = false
  
  // Global state for cancellation - not ideal
  process.on('SIGTERM', () => {
    shouldStop = true
  })
  
  while (!shouldStop) {
    try {
      const batch = await fetchNextBatch()
      // How do we cancel this if shutdown is requested?
      await processBatch(batch)
      await new Promise(resolve => setTimeout(resolve, 1000))
    } catch (error) {
      console.error('Background processing error:', error)
      // Should we continue? How do we implement backoff?
    }
  }
}
```

This approach leads to:
- **Uncontrolled concurrency** - No way to limit concurrent operations
- **Poor cancellation** - Difficult to interrupt running operations
- **Resource leaks** - No structured cleanup mechanism
- **Complex error handling** - All-or-nothing failure modes
- **Debugging nightmares** - No visibility into fiber lifecycles

### The Fiber Solution

Fibers provide lightweight, controllable virtual threads with structured concurrency, resource management, and interruption:

```typescript
import { Effect, Fiber, Queue } from "effect"

// Clean, structured concurrent processing
const processOrdersConcurrently = (orders: Order[]) =>
  Effect.gen(function* () {
    // Fork each order processing into its own fiber
    const fibers = yield* Effect.all(
      orders.map(order => 
        Effect.fork(
          Effect.gen(function* () {
            const validated = yield* validateOrder(order)
            const processed = yield* processPayment(validated)
            const shipped = yield* scheduleShipping(processed)
            return shipped
          })
        )
      )
    )
    
    // Wait for all to complete with structured error handling
    const results = yield* Effect.all(
      fibers.map(fiber => Fiber.join(fiber))
    )
    
    return results
  })

// Graceful background processing with proper cancellation
const backgroundProcessor = Effect.gen(function* () {
  yield* Effect.forever(
    Effect.gen(function* () {
      const batch = yield* fetchNextBatch()
      yield* processBatch(batch)
      yield* Effect.sleep("1 second")
    })
  )
}).pipe(
  // Automatically handles interruption and cleanup
  Effect.interruptible,
  Effect.retry(Schedule.exponential("100 millis").pipe(Schedule.compose(Schedule.recurs(3))))
)
```

### Key Concepts

**Fiber**: A lightweight virtual thread that represents the execution of an Effect. Fibers can be forked, joined, interrupted, and composed.

**Fork**: Creating a new fiber by running an Effect concurrently in the background.

**Join**: Waiting for a fiber to complete and retrieving its result.

**Interrupt**: Gracefully stopping a fiber and running all cleanup code.

**Structured Concurrency**: Child fibers are automatically managed by their parent, ensuring no resource leaks.

## Basic Usage Patterns

### Pattern 1: Fork and Join

```typescript
import { Effect, Fiber } from "effect"

// Basic fork and join pattern
const forkAndJoin = Effect.gen(function* () {
  // Fork a computation into a new fiber
  const fiber = yield* Effect.fork(
    Effect.gen(function* () {
      yield* Effect.sleep("1 second")
      return "Hello from fiber!"
    })
  )
  
  // Do other work while fiber runs
  console.log("Doing other work...")
  yield* Effect.sleep("500 millis")
  
  // Join the fiber and get its result
  const result = yield* Fiber.join(fiber)
  console.log(result) // "Hello from fiber!"
  
  return result
})
```

### Pattern 2: Multiple Concurrent Operations

```typescript
import { Effect, Fiber } from "effect"

interface ApiResponse {
  data: string
  timestamp: number
}

const fetchFromApi = (url: string): Effect.Effect<ApiResponse> =>
  Effect.gen(function* () {
    yield* Effect.sleep("200 millis") // Simulate network delay
    return {
      data: `Data from ${url}`,
      timestamp: Date.now()
    }
  })

const concurrentApiCalls = Effect.gen(function* () {
  // Fork multiple API calls
  const fiber1 = yield* Effect.fork(fetchFromApi("https://api1.com"))
  const fiber2 = yield* Effect.fork(fetchFromApi("https://api2.com"))
  const fiber3 = yield* Effect.fork(fetchFromApi("https://api3.com"))
  
  // Join all fibers and collect results
  const results = yield* Effect.all([
    Fiber.join(fiber1),
    Fiber.join(fiber2),
    Fiber.join(fiber3)
  ])
  
  return results
})
```

### Pattern 3: Interruption and Cleanup

```typescript
import { Effect, Fiber } from "effect"

const interruptibleTask = Effect.gen(function* () {
  const longRunningFiber = yield* Effect.fork(
    Effect.gen(function* () {
      yield* Effect.addFinalizer(() => 
        Effect.sync(() => console.log("Cleaning up resources..."))
      )
      
      // Simulate long-running work
      yield* Effect.forever(
        Effect.gen(function* () {
          console.log("Working...")
          yield* Effect.sleep("1 second")
        })
      )
    })
  )
  
  // Let it run for a bit
  yield* Effect.sleep("3 seconds")
  
  // Interrupt the fiber (cleanup will run automatically)
  const exit = yield* Fiber.interrupt(longRunningFiber)
  console.log("Fiber interrupted:", exit)
  
  return "Task completed"
})
```

## Real-World Examples

### Example 1: Concurrent File Processing

A common scenario where you need to process multiple files concurrently with proper error handling and resource management.

```typescript
import { Effect, Fiber, Ref } from "effect"
import * as fs from "fs/promises"

interface ProcessingResult {
  fileName: string
  lineCount: number
  wordCount: number
  success: boolean
  error?: string
}

const processFile = (filePath: string): Effect.Effect<ProcessingResult> =>
  Effect.gen(function* () {
    try {
      const content = yield* Effect.promise(() => fs.readFile(filePath, 'utf-8'))
      const lines = content.split('\n').length
      const words = content.split(/\s+/).filter(word => word.length > 0).length
      
      return {
        fileName: filePath,
        lineCount: lines,
        wordCount: words,
        success: true
      }
    } catch (error) {
      return {
        fileName: filePath,
        lineCount: 0,
        wordCount: 0,
        success: false,
        error: error instanceof Error ? error.message : 'Unknown error'
      }
    }
  })

const processFilesWithProgress = (filePaths: string[], maxConcurrency: number = 5) =>
  Effect.gen(function* () {
    const progressRef = yield* Ref.make(0)
    const resultsRef = yield* Ref.make<ProcessingResult[]>([])
    
    // Process files in batches to control concurrency
    const batches = []
    for (let i = 0; i < filePaths.length; i += maxConcurrency) {
      batches.push(filePaths.slice(i, i + maxConcurrency))
    }
    
    for (const batch of batches) {
      // Fork each file processing in the batch
      const fibers = yield* Effect.all(
        batch.map(filePath =>
          Effect.fork(
            Effect.gen(function* () {
              const result = yield* processFile(filePath)
              
              // Update progress atomically
              yield* Ref.update(progressRef, count => count + 1)
              yield* Ref.update(resultsRef, results => [...results, result])
              
              const currentProgress = yield* Ref.get(progressRef)
              console.log(`Progress: ${currentProgress}/${filePaths.length} files processed`)
              
              return result
            })
          )
        )
      )
      
      // Wait for current batch to complete
      yield* Effect.all(fibers.map(fiber => Fiber.join(fiber)))
    }
    
    const finalResults = yield* Ref.get(resultsRef)
    const successful = finalResults.filter(r => r.success).length
    const failed = finalResults.filter(r => !r.success).length
    
    console.log(`Processing complete: ${successful} successful, ${failed} failed`)
    return finalResults
  })

// Usage
const fileProcessingProgram = processFilesWithProgress([
  'file1.txt', 'file2.txt', 'file3.txt', 'file4.txt', 'file5.txt'
], 3)
```

### Example 2: Real-time Data Pipeline

Building a real-time data processing pipeline that can handle backpressure and graceful shutdown.

```typescript
import { Effect, Fiber, Queue, Ref, Schedule } from "effect"

interface DataPoint {
  id: string
  value: number
  timestamp: number
}

interface ProcessedData {
  id: string
  processedValue: number
  processingTime: number
}

// Simulated data source
const dataProducer = (queue: Queue.Queue<DataPoint>) =>
  Effect.gen(function* () {
    let counter = 0
    
    yield* Effect.forever(
      Effect.gen(function* () {
        const dataPoint: DataPoint = {
          id: `data-${counter++}`,
          value: Math.random() * 100,
          timestamp: Date.now()
        }
        
        // Offer data to queue (with backpressure)
        yield* Queue.offer(queue, dataPoint)
        console.log(`Produced: ${dataPoint.id}`)
        
        // Vary production rate
        yield* Effect.sleep(`${Math.random() * 1000}ms`)
      })
    )
  })

// Data processor
const dataProcessor = (
  inputQueue: Queue.Queue<DataPoint>,
  outputQueue: Queue.Queue<ProcessedData>
) =>
  Effect.gen(function* () {
    yield* Effect.forever(
      Effect.gen(function* () {
        // Take data from input queue
        const dataPoint = yield* Queue.take(inputQueue)
        
        // Simulate processing time
        const startTime = Date.now()
        yield* Effect.sleep(`${100 + Math.random() * 200}ms`)
        
        const processedData: ProcessedData = {
          id: dataPoint.id,
          processedValue: dataPoint.value * 2,
          processingTime: Date.now() - startTime
        }
        
        // Offer processed data to output queue
        yield* Queue.offer(outputQueue, processedData)
        console.log(`Processed: ${processedData.id} (${processedData.processingTime}ms)`)
      })
    )
  })

// Data consumer
const dataConsumer = (queue: Queue.Queue<ProcessedData>) =>
  Effect.gen(function* () {
    yield* Effect.forever(
      Effect.gen(function* () {
        // Take processed data
        const processedData = yield* Queue.take(queue)
        
        // Simulate consumption (e.g., save to database)
        console.log(`Consumed: ${processedData.id} -> ${processedData.processedValue}`)
        
        yield* Effect.sleep("50 millis")
      })
    )
  })

// Main pipeline
const dataPipeline = Effect.gen(function* () {
  // Create queues with backpressure
  const inputQueue = yield* Queue.bounded<DataPoint>(10)
  const outputQueue = yield* Queue.bounded<ProcessedData>(10)
  
  // Fork all pipeline stages
  const producerFiber = yield* Effect.fork(dataProducer(inputQueue))
  const processorFiber = yield* Effect.fork(dataProcessor(inputQueue, outputQueue))
  const consumerFiber = yield* Effect.fork(dataConsumer(outputQueue))
  
  // Let pipeline run for 10 seconds
  yield* Effect.sleep("10 seconds")
  
  // Graceful shutdown
  console.log("Shutting down pipeline...")
  
  // Interrupt all fibers (cleanup will happen automatically)
  yield* Fiber.interrupt(producerFiber)
  yield* Fiber.interrupt(processorFiber)
  yield* Fiber.interrupt(consumerFiber)
  
  // Shutdown queues
  yield* Queue.shutdown(inputQueue)
  yield* Queue.shutdown(outputQueue)
  
  console.log("Pipeline shutdown complete")
})
```

### Example 3: Resilient Web Scraper

A web scraper that handles failures, implements retry logic, and manages concurrent requests.

```typescript
import { Effect, Fiber, Ref, Schedule } from "effect"

interface ScrapingResult {
  url: string
  title?: string
  content?: string
  success: boolean
  error?: string
  attempts: number
}

const scrapeUrl = (url: string): Effect.Effect<ScrapingResult> =>
  Effect.gen(function* () {
    // Simulate HTTP request
    yield* Effect.sleep(`${Math.random() * 1000}ms`)
    
    // Simulate random failures
    if (Math.random() < 0.3) {
      yield* Effect.fail(new Error(`Failed to fetch ${url}`))
    }
    
    return {
      url,
      title: `Title for ${url}`,
      content: `Content from ${url}`,
      success: true,
      attempts: 1
    }
  })

const scrapeWithRetry = (url: string): Effect.Effect<ScrapingResult> =>
  Effect.gen(function* () {
    const attemptsRef = yield* Ref.make(0)
    
    const result = yield* pipe(
      Effect.tap(scrapeUrl(url), () => Ref.update(attemptsRef, n => n + 1))
    ).pipe(
      Effect.retry(
        pipe(
          Schedule.exponential("100 millis")
        ).pipe(
          Schedule.compose(Schedule.recurs(3))
        ).pipe(
          Schedule.tapOutput(() => Effect.sync(() => console.log(`Retrying ${url}...`)))
        )
      )
    ).pipe(
      Effect.catchAll(error =>
        Effect.gen(function* () {
          const attempts = yield* Ref.get(attemptsRef)
          return {
            url,
            success: false,
            error: error.message,
            attempts
          }
        })
      )
    )
    
    const finalAttempts = yield* Ref.get(attemptsRef)
    return { ...result, attempts: finalAttempts }
  })

const concurrentScraper = (urls: string[], maxConcurrency: number = 3) =>
  Effect.gen(function* () {
    // Fork all scraping operations
    const fibers = yield* Effect.all(
      urls.map(url => Effect.fork(scrapeWithRetry(url))),
      { concurrency: maxConcurrency }
    )
    
    // Join all fibers to get results
    const results = yield* Effect.all(
      fibers.map(fiber => Fiber.join(fiber))
    )
    
    const successful = results.filter(r => r.success).length
    const failed = results.filter(r => !r.success).length
    
    console.log(`Scraping complete: ${successful} successful, ${failed} failed`)
    return results
  })

// Usage
const scrapingProgram = concurrentScraper([
  'https://example1.com',
  'https://example2.com',
  'https://example3.com',
  'https://example4.com',
  'https://example5.com'
], 2)
```

## Advanced Features Deep Dive

### Feature 1: Fiber Composition and Orchestration

Fibers can be composed using various combinators to create sophisticated concurrency patterns.

#### Basic Fiber Composition

```typescript
import { Effect, Fiber } from "effect"

const composeFibers = Effect.gen(function* () {
  const fiber1 = yield* Effect.fork(Effect.succeed("Hello"))
  const fiber2 = yield* Effect.fork(Effect.succeed("World"))
  
  // Zip fibers together
  const combinedFiber = Fiber.zip(fiber1, fiber2)
  const [result1, result2] = yield* Fiber.join(combinedFiber)
  
  return `${result1} ${result2}` // "Hello World"
})
```

#### Advanced Fiber Orchestration

```typescript
import { Effect, Fiber, Duration } from "effect"

interface Task {
  id: string
  duration: Duration.Duration
  result: string
}

const executeTask = (task: Task): Effect.Effect<string> =>
  Effect.gen(function* () {
    yield* Effect.sleep(task.duration)
    return `Task ${task.id}: ${task.result}`
  })

// Complex orchestration pattern
const orchestrateTasks = (tasks: Task[]) =>
  Effect.gen(function* () {
    // Fork all tasks
    const fibers = yield* Effect.all(
      tasks.map(task => Effect.fork(executeTask(task)))
    )
    
    // Wait for first completion
    const firstCompleted = yield* Fiber.raceAll(fibers)
    const firstResult = yield* Fiber.join(firstCompleted)
    
    console.log(`First task completed: ${firstResult}`)
    
    // Continue with remaining tasks
    const remainingFibers = fibers.filter(f => f !== firstCompleted)
    const remainingResults = yield* Effect.all(
      remainingFibers.map(fiber => Fiber.join(fiber))
    )
    
    return [firstResult, ...remainingResults]
  })

// Helper function for racing fibers
const Fiber = {
  // ... existing Fiber functions
  raceAll: <A>(fibers: Fiber.Fiber<A>[]) =>
    Effect.gen(function* () {
      // Use Effect.race to find first completion
      const effects = fibers.map(fiber => Fiber.join(fiber))
      return yield* Effect.raceAll(effects)
    })
}
```

### Feature 2: Fiber Supervision and Lifecycle Management

Understanding how fibers are supervised and their lifecycle patterns.

#### Automatic Supervision

```typescript
import { Effect, Fiber, Ref } from "effect"

const supervisedFibers = Effect.gen(function* () {
  const parentFiber = yield* Effect.fork(
    Effect.gen(function* () {
      console.log("Parent fiber started")
      
      // Child fibers are automatically supervised
      const child1 = yield* Effect.fork(
        Effect.gen(function* () {
          yield* Effect.addFinalizer(() => 
            Effect.sync(() => console.log("Child 1 cleanup"))
          )
          yield* Effect.forever(
            Effect.gen(function* () {
              console.log("Child 1 working...")
              yield* Effect.sleep("1 second")
            })
          )
        })
      )
      
      const child2 = yield* Effect.fork(
        Effect.gen(function* () {
          yield* Effect.addFinalizer(() => 
            Effect.sync(() => console.log("Child 2 cleanup"))
          )
          yield* Effect.sleep("3 seconds")
          return "Child 2 result"
        })
      )
      
      // Wait for child2, then interrupt everything
      const result = yield* Fiber.join(child2)
      console.log(result)
      
      // When parent ends, child1 is automatically interrupted
      return "Parent completed"
    })
  )
  
  return yield* Fiber.join(parentFiber)
})
```

#### Daemon Fibers

```typescript
import { Effect, Fiber } from "effect"

const daemonFiberExample = Effect.gen(function* () {
  // Fork a daemon fiber that runs independently
  const daemonFiber = yield* Effect.forkDaemon(
    Effect.gen(function* () {
      yield* Effect.forever(
        Effect.gen(function* () {
          console.log("Daemon fiber working...")
          yield* Effect.sleep("2 seconds")
        })
      )
    })
  )
  
  console.log("Main fiber doing work...")
  yield* Effect.sleep("5 seconds")
  console.log("Main fiber completed")
  
  // Daemon continues running until explicitly interrupted
  yield* Effect.sleep("2 seconds")
  yield* Fiber.interrupt(daemonFiber)
  
  return "Program completed"
})
```

### Feature 3: Advanced Interruption Patterns

Sophisticated interruption handling for complex scenarios.

#### Interruption with Resource Cleanup

```typescript
import { Effect, Fiber, Ref } from "effect"

interface Resource {
  id: string
  acquired: boolean
}

const acquireResource = (id: string): Effect.Effect<Resource> =>
  Effect.gen(function* () {
    console.log(`Acquiring resource ${id}...`)
    yield* Effect.sleep("100 millis")
    return { id, acquired: true }
  })

const releaseResource = (resource: Resource): Effect.Effect<void> =>
  Effect.gen(function* () {
    console.log(`Releasing resource ${resource.id}...`)
    yield* Effect.sleep("50 millis")
  })

const interruptibleWithCleanup = Effect.gen(function* () {
  const resourcesRef = yield* Ref.make<Resource[]>([])
  
  const fiber = yield* Effect.fork(
    Effect.gen(function* () {
      // Acquire multiple resources
      for (let i = 0; i < 5; i++) {
        const resource = yield* acquireResource(`resource-${i}`)
        yield* Ref.update(resourcesRef, resources => [...resources, resource])
        
        // Add cleanup finalizer for each resource
        yield* Effect.addFinalizer(() => releaseResource(resource))
        
        yield* Effect.sleep("200 millis")
      }
      
      // Long-running work
      yield* Effect.forever(
        Effect.gen(function* () {
          console.log("Working with resources...")
          yield* Effect.sleep("500 millis")
        })
      )
    })
  )
  
  // Let it run for a bit
  yield* Effect.sleep("1 second")
  
  // Interrupt - all finalizers will run
  console.log("Interrupting fiber...")
  const exit = yield* Fiber.interrupt(fiber)
  
  const finalResources = yield* Ref.get(resourcesRef)
  console.log(`Cleaned up ${finalResources.length} resources`)
  
  return exit
})
```

## Practical Patterns & Best Practices

### Pattern 1: Bounded Concurrency with Semaphore

```typescript
import { Effect, Fiber, Semaphore } from "effect"

const boundedConcurrentProcessing = <A, B>(
  items: A[],
  processor: (item: A) => Effect.Effect<B>,
  maxConcurrency: number
) =>
  Effect.gen(function* () {
    // Create semaphore to limit concurrency
    const semaphore = yield* Semaphore.make(maxConcurrency)
    
    // Process items with bounded concurrency
    const fibers = yield* Effect.all(
      items.map(item =>
        Effect.fork(
          Semaphore.withPermits(semaphore, 1, processor(item))
        )
      )
    )
    
    // Join all fibers
    return yield* Effect.all(
      fibers.map(fiber => Fiber.join(fiber))
    )
  })

// Usage example
const processUrls = boundedConcurrentProcessing(
  ['url1', 'url2', 'url3', 'url4', 'url5'],
  (url: string) => Effect.gen(function* () {
    console.log(`Processing ${url}`)
    yield* Effect.sleep("1 second")
    return `Result for ${url}`
  }),
  2 // Max 2 concurrent operations
)
```

### Pattern 2: Producer-Consumer with Backpressure

```typescript
import { Effect, Fiber, Queue, Ref } from "effect"

const producerConsumerPattern = <T>(
  producer: () => Effect.Effect<T>,
  consumer: (item: T) => Effect.Effect<void>,
  queueSize: number = 10
) =>
  Effect.gen(function* () {
    const queue = yield* Queue.bounded<T>(queueSize)
    const processedCount = yield* Ref.make(0)
    
    // Producer fiber
    const producerFiber = yield* Effect.fork(
      Effect.gen(function* () {
        yield* Effect.forever(
          Effect.gen(function* () {
            const item = yield* producer()
            yield* Queue.offer(queue, item) // Blocks if queue is full
          })
        )
      })
    )
    
    // Consumer fiber
    const consumerFiber = yield* Effect.fork(
      Effect.gen(function* () {
        yield* Effect.forever(
          Effect.gen(function* () {
            const item = yield* Queue.take(queue)
            yield* consumer(item)
            yield* Ref.update(processedCount, n => n + 1)
          })
        )
      })
    )
    
    // Return control handles
    return {
      stop: () => Effect.gen(function* () {
        yield* Fiber.interrupt(producerFiber)
        yield* Fiber.interrupt(consumerFiber)
        yield* Queue.shutdown(queue)
        const finalCount = yield* Ref.get(processedCount)
        return finalCount
      }),
      getProcessedCount: () => Ref.get(processedCount),
      getQueueSize: () => Queue.size(queue)
    }
  })
```

### Pattern 3: Graceful Shutdown Manager

```typescript
import { Effect, Fiber, Ref, Deferred } from "effect"

class ShutdownManager {
  private fibers: Ref.Ref<Fiber.Fiber<any>[]>
  private shutdownDeferred: Deferred.Deferred<void>
  
  constructor(
    fibers: Ref.Ref<Fiber.Fiber<any>[]>,
    shutdownDeferred: Deferred.Deferred<void>
  ) {
    this.fibers = fibers
    this.shutdownDeferred = shutdownDeferred
  }
  
  static make = Effect.gen(function* () {
    const fibers = yield* Ref.make<Fiber.Fiber<any>[]>([])
    const shutdownDeferred = yield* Deferred.make<void>()
    return new ShutdownManager(fibers, shutdownDeferred)
  })
  
  register = <A>(effect: Effect.Effect<A>) =>
    Effect.gen(function* () {
      const fiber = yield* Effect.fork(effect)
      yield* Ref.update(this.fibers, fibers => [...fibers, fiber])
      return fiber
    })
  
  shutdown = (timeoutMs: number = 5000) =>
    Effect.gen(function* () {
      console.log("Initiating graceful shutdown...")
      
      // Signal shutdown to all fibers
      yield* Deferred.succeed(this.shutdownDeferred, undefined)
      
      const allFibers = yield* Ref.get(this.fibers)
      
      // Try graceful shutdown first
      const shutdownResults = yield* Effect.all(
        allFibers.map(fiber => 
          pipe(
            Fiber.interrupt(fiber)
          ).pipe(
            Effect.timeout(`${timeoutMs}ms`)
          ).pipe(
            Effect.option
          )
        )
      )
      
      const gracefulCount = shutdownResults.filter(result => result._tag === "Some").length
      const forcedCount = allFibers.length - gracefulCount
      
      console.log(`Shutdown complete: ${gracefulCount} graceful, ${forcedCount} forced`)
      return { gracefulCount, forcedCount }
    })
  
  isShuttingDown = () => Deferred.poll(this.shutdownDeferred)
}

// Usage example
const managedApplication = Effect.gen(function* () {
  const shutdownManager = yield* ShutdownManager.make
  
  // Register background tasks
  yield* shutdownManager.register(
    Effect.gen(function* () {
      yield* Effect.forever(
        Effect.gen(function* () {
          const isShuttingDown = yield* shutdownManager.isShuttingDown()
          if (isShuttingDown._tag === "Some") {
            console.log("Task 1 detected shutdown, cleaning up...")
            return
          }
          
          console.log("Task 1 working...")
          yield* Effect.sleep("1 second")
        })
      )
    })
  )
  
  // Simulate running for a while
  yield* Effect.sleep("5 seconds")
  
  // Shutdown gracefully
  return yield* shutdownManager.shutdown(3000)
})
```

## Integration Examples

### Integration with Express.js

```typescript
import { Effect, Fiber, Queue, Ref } from "effect"
import express from "express"

interface JobRequest {
  id: string
  data: any
  resolve: (result: any) => void
  reject: (error: Error) => void
}

const createJobProcessor = Effect.gen(function* () {
  const jobQueue = yield* Queue.unbounded<JobRequest>()
  const activeJobs = yield* Ref.make<Map<string, Fiber.Fiber<any>>>(new Map())
  
  // Job processor fiber
  const processorFiber = yield* Effect.fork(
    Effect.forever(
      Effect.gen(function* () {
        const job = yield* Queue.take(jobQueue)
        
        const jobFiber = yield* Effect.fork(
          Effect.gen(function* () {
            try {
              // Simulate job processing
              yield* Effect.sleep("2 seconds")
              const result = `Processed job ${job.id}`
              job.resolve(result)
            } catch (error) {
              job.reject(error as Error)
            }
          })
        )
        
        // Track active job
        yield* Ref.update(activeJobs, jobs => 
          new Map(jobs.set(job.id, jobFiber))
        )
        
        // Clean up when job completes
        yield* Effect.fork(
          Effect.gen(function* () {
            yield* Fiber.await(jobFiber)
            yield* Ref.update(activeJobs, jobs => {
              const newJobs = new Map(jobs)
              newJobs.delete(job.id)
              return newJobs
            })
          })
        )
      })
    )
  )
  
  const submitJob = (id: string, data: any): Promise<any> =>
    new Promise((resolve, reject) => {
      const job: JobRequest = { id, data, resolve, reject }
      Effect.runPromise(Queue.offer(jobQueue, job))
    })
  
  const getActiveJobs = () =>
    Effect.runPromise(
      Effect.gen(function* () {
        const jobs = yield* Ref.get(activeJobs)
        return Array.from(jobs.keys())
      })
    )
  
  const shutdown = () =>
    Effect.runPromise(
      Effect.gen(function* () {
        yield* Fiber.interrupt(processorFiber)
        yield* Queue.shutdown(jobQueue)
      })
    )
  
  return { submitJob, getActiveJobs, shutdown }
})

// Express integration
const app = express()
app.use(express.json())

Effect.runPromise(createJobProcessor).then(jobProcessor => {
  app.post('/jobs', async (req, res) => {
    try {
      const jobId = Math.random().toString(36).substr(2, 9)
      const result = await jobProcessor.submitJob(jobId, req.body)
      res.json({ success: true, jobId, result })
    } catch (error) {
      res.status(500).json({ success: false, error: error.message })
    }
  })
  
  app.get('/jobs', async (req, res) => {
    try {
      const activeJobs = await jobProcessor.getActiveJobs()
      res.json({ activeJobs })
    } catch (error) {
      res.status(500).json({ error: error.message })
    }
  })
  
  // Graceful shutdown
  process.on('SIGTERM', () => {
    console.log('Shutting down...')
    jobProcessor.shutdown().then(() => {
      process.exit(0)
    })
  })
  
  app.listen(3000, () => {
    console.log('Server running on port 3000')
  })
})
```

### Testing Strategies

```typescript
import { Effect, Fiber, TestContext, TestClock } from "effect"
import { describe, it, expect } from "@effect/vitest"

describe("Fiber Testing", () => {
  it("should test concurrent operations", () =>
    Effect.gen(function* () {
      const results: string[] = []
      
      const task1 = Effect.gen(function* () {
        yield* Effect.sleep("100 millis")
        results.push("task1")
        return "task1"
      })
      
      const task2 = Effect.gen(function* () {
        yield* Effect.sleep("200 millis")
        results.push("task2")
        return "task2"
      })
      
      // Fork both tasks
      const fiber1 = yield* Effect.fork(task1)
      const fiber2 = yield* Effect.fork(task2)
      
      // Advance virtual time
      yield* TestClock.adjust("300 millis")
      
      // Join fibers
      const result1 = yield* Fiber.join(fiber1)
      const result2 = yield* Fiber.join(fiber2)
      
      expect(results).toEqual(["task1", "task2"])
      expect(result1).toBe("task1")
      expect(result2).toBe("task2")
    }).pipe(
      Effect.provide(TestContext.TestContext)
    )
  )
  
  it("should test fiber interruption", () =>
    Effect.gen(function* () {
      let cleanupCalled = false
      
      const interruptibleTask = Effect.gen(function* () {
        yield* Effect.addFinalizer(() => 
          Effect.sync(() => { cleanupCalled = true })
        )
        yield* Effect.sleep("1 second")
        return "should not reach here"
      })
      
      const fiber = yield* Effect.fork(interruptibleTask)
      yield* TestClock.adjust("500 millis")
      
      const exit = yield* Fiber.interrupt(fiber)
      
      expect(exit._tag).toBe("Failure")
      expect(cleanupCalled).toBe(true)
    }).pipe(
      Effect.provide(TestContext.TestContext)
    )
  )
  
  it("should test fiber composition", () =>
    Effect.gen(function* () {
      const fiber1 = yield* Effect.fork(Effect.succeed("Hello"))
      const fiber2 = yield* Effect.fork(Effect.succeed("World"))
      
      const combined = Fiber.zip(fiber1, fiber2)
      const [result1, result2] = yield* Fiber.join(combined)
      
      expect(result1).toBe("Hello")
      expect(result2).toBe("World")
    })
  )
})

// Property-based testing
const testFiberProperties = Effect.gen(function* () {
  // Test that forked fibers don't block parent
  const startTime = yield* Effect.sync(() => Date.now())
  
  const slowFiber = yield* Effect.fork(
    Effect.gen(function* () {
      yield* Effect.sleep("1 second")
      return "slow"
    })
  )
  
  const fastResult = yield* Effect.gen(function* () {
    yield* Effect.sleep("100 millis")
    return "fast"
  })
  
  const endTime = yield* Effect.sync(() => Date.now())
  const elapsed = endTime - startTime
  
  // Parent should complete quickly
  expect(elapsed).toBeLessThan(200)
  expect(fastResult).toBe("fast")
  
  // Slow fiber should still be running
  const slowResult = yield* Fiber.join(slowFiber)
  expect(slowResult).toBe("slow")
})
```

Fiber provides the foundation for building scalable, maintainable concurrent applications in Effect. By embracing structured concurrency, you get automatic resource management, graceful error handling, and predictable fiber lifecycles.

Key benefits:
- **Structured Concurrency**: Automatic cleanup and resource management
- **Interruptible Operations**: Graceful cancellation with cleanup guarantees
- **Composable Patterns**: Build complex concurrency from simple primitives
- **Type Safety**: Full TypeScript integration with error tracking
- **Testing Support**: Comprehensive testing utilities for concurrent code

Use Fiber when you need lightweight concurrency, background processing, parallel operations, or any scenario where traditional Promise-based approaches become unwieldy.