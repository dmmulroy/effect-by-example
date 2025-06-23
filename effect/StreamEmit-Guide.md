# StreamEmit: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem StreamEmit Solves

Creating streams from asynchronous callbacks, push-based data sources, and external event emitters is a common challenge in modern applications. Traditional approaches often lead to complex state management, memory leaks, and difficult error handling.

```typescript
// Traditional Node.js EventEmitter approach - error-prone
import { EventEmitter } from 'events'

class DataStream extends EventEmitter {
  private buffer: any[] = []
  private closed = false
  
  async processData() {
    try {
      while (!this.closed) {
        const data = await this.fetchData()
        this.buffer.push(data)
        this.emit('data', data)
      }
    } catch (error) {
      this.emit('error', error)
    } finally {
      this.emit('end')
    }
  }
  
  close() {
    this.closed = true
  }
}

// Usage is complex and error-prone
const stream = new DataStream()
stream.on('data', (data) => console.log(data))
stream.on('error', (error) => console.error(error))
stream.on('end', () => console.log('Done'))
stream.processData() // Fire and forget - hard to manage lifecycle

// RxJS Subject approach - manual resource management
import { Subject } from 'rxjs'

const createRxJSStream = () => {
  const subject = new Subject<number>()
  
  const interval = setInterval(() => {
    subject.next(Math.random())
  }, 1000)
  
  // Manual cleanup required - easy to forget
  setTimeout(() => {
    clearInterval(interval)
    subject.complete()
  }, 5000)
  
  return subject
}
```

This approach leads to:
- **Resource Leaks** - Manual cleanup often forgotten or handled incorrectly
- **Complex State Management** - Tracking emission state across multiple callbacks
- **Error Handling Fragmentation** - Errors scattered throughout callback chains
- **Backpressure Issues** - No built-in flow control for overwhelming consumers

### The StreamEmit Solution

Effect's StreamEmit module provides a powerful, type-safe interface for creating streams from push-based data sources with automatic resource management and built-in backpressure handling.

```typescript
import { Stream, Effect, Chunk, StreamEmit } from "effect"

// The Effect solution - safe, composable, with automatic resource management
const createEffectStream = Stream.asyncPush<string>((emit) =>
  Effect.acquireRelease(
    Effect.gen(function* () {
      yield* Effect.log("Starting data source")
      return setInterval(() => {
        emit.single(`data-${Date.now()}`)
      }, 1000)
    }),
    (handle) => Effect.gen(function* () {
      yield* Effect.log("Cleaning up data source")
      clearInterval(handle)
    })
  )
)
```

### Key Concepts

**Emit**: An asynchronous callback interface that can be called multiple times to emit values, handle errors, or signal stream termination with built-in type safety.

**EmitOps**: A collection of helper methods for common emission patterns including single values, chunks, effects, and error handling.

**EmitOpsPush**: A specialized interface for push-based streams that provides immediate feedback about emission success through boolean return values.

## Basic Usage Patterns

### Pattern 1: Simple Async Stream Creation

```typescript
import { Stream, Effect, Chunk, StreamEmit } from "effect"

// Basic async stream with manual emission control
const simpleAsyncStream = Stream.async<number>(
  (emit: StreamEmit.Emit<never, never, number, void>) => {
    let count = 0
    
    const interval = setInterval(() => {
      if (count < 5) {
        emit.single(count++)
      } else {
        emit.end() // Signal stream completion
        clearInterval(interval)
      }
    }, 1000)
    
    // Optional cleanup function
    return Effect.sync(() => clearInterval(interval))
  }
)
```

### Pattern 2: Push-Based Stream with Resource Management

```typescript
import { Stream, Effect, Scope } from "effect"

// Push-based stream with automatic resource cleanup
const pushBasedStream = Stream.asyncPush<string>((emit) =>
  Effect.acquireRelease(
    Effect.gen(function* () {
      const websocket = new WebSocket('wss://example.com/data')
      
      websocket.onmessage = (event) => {
        emit.single(event.data)
      }
      
      websocket.onerror = (error) => {
        emit.fail(new Error(`WebSocket error: ${error}`))
      }
      
      websocket.onclose = () => {
        emit.end()
      }
      
      return websocket
    }),
    (ws) => Effect.sync(() => ws.close())
  )
)
```

### Pattern 3: Effect-Based Stream Creation

```typescript
import { Stream, Effect } from "effect"

// Stream that integrates with Effect-based operations
const effectBasedStream = Stream.asyncEffect<string, string>(
  (emit) => Effect.gen(function* () {
    const config = yield* ConfigService
    const database = yield* DatabaseService
    
    // Set up database change listener
    const listener = (change: DatabaseChange) => {
      emit.fromEffect(
        Effect.gen(function* () {
          const processedData = yield* processChange(change)
          return processedData
        })
      )
    }
    
    yield* database.onChanges(listener)
    
    return () => database.removeListener(listener)
  })
)
```

## Real-World Examples

### Example 1: Event-Driven Data Processing

Building a stream that processes server-sent events with error handling and backpressure control.

```typescript
import { Stream, Effect, Schedule, Context, Layer } from "effect"

// Define our services
interface EventSourceConfig {
  readonly url: string
  readonly reconnectDelay: string
}

const EventSourceConfig = Context.GenericTag<EventSourceConfig>("EventSourceConfig")

interface Logger {
  readonly info: (message: string) => Effect.Effect<void>
  readonly error: (message: string, error?: unknown) => Effect.Effect<void>
}

const Logger = Context.GenericTag<Logger>("Logger")

// Create a resilient event stream
const createEventStream = <T>(
  parseEvent: (data: string) => Effect.Effect<T, Error>
) => Stream.asyncPush<T, Error>((emit) =>
  Effect.gen(function* () {
    const config = yield* EventSourceConfig
    const logger = yield* Logger
    
    yield* logger.info(`Connecting to event stream: ${config.url}`)
    
    const eventSource = new EventSource(config.url)
    
    // Handle successful messages
    eventSource.onmessage = (event) => {
      Effect.runPromise(
        parseEvent(event.data).pipe(
          Effect.flatMap((parsed) => Effect.sync(() => emit.single(parsed))),
          Effect.catchAll((error) => 
            Effect.gen(function* () {
              yield* logger.error("Failed to parse event", error)
              emit.fail(error)
            })
          )
        )
      )
    }
    
    // Handle connection errors
    eventSource.onerror = (error) => {
      Effect.runPromise(
        logger.error("EventSource error", error).pipe(
          Effect.andThen(() => emit.fail(new Error("EventSource connection failed")))
        )
      )
    }
    
    return eventSource
  }).pipe(
    Effect.acquireRelease(
      (eventSource) => Effect.gen(function* () {
        const logger = yield* Logger
        yield* logger.info("Closing event stream connection")
        eventSource.close()
      })
    )
  )
)

// Usage with automatic reconnection
const processEvents = createEventStream((data: string) =>
  Effect.try({
    try: () => JSON.parse(data) as { id: string; type: string; payload: unknown },
    catch: (error) => new Error(`Invalid JSON: ${error}`)
  })
).pipe(
  Stream.retry(Schedule.exponential("1 second").pipe(Schedule.compose(Schedule.recurs(5)))),
  Stream.tap((event) => Effect.log(`Processing event: ${event.id}`))
)
```

### Example 2: Real-Time Metrics Collection

Creating a metrics stream that aggregates data from multiple sources with configurable emission strategies.

```typescript
import { Stream, Effect, Ref, Queue, Fiber, Exit } from "effect"

interface MetricPoint {
  readonly name: string
  readonly value: number
  readonly timestamp: number
  readonly tags: Record<string, string>
}

interface MetricsCollector {
  readonly collect: Effect.Effect<MetricPoint[]>
  readonly interval: number
}

const MetricsCollector = Context.GenericTag<MetricsCollector>("MetricsCollector")

// Advanced metrics stream with batching and flow control
const createMetricsStream = (batchSize: number = 100) =>
  Stream.asyncPush<MetricPoint[], Error>((emit) =>
    Effect.gen(function* () {
      const collector = yield* MetricsCollector
      const batchRef = yield* Ref.make<MetricPoint[]>([])
      const queue = yield* Queue.bounded<MetricPoint>(1000)
      
      // Producer fiber - collects metrics at regular intervals
      const producerFiber = yield* Effect.gen(function* () {
        while (true) {
          const metrics = yield* collector.collect
          yield* Queue.offerAll(queue, metrics)
          yield* Effect.sleep(`${collector.interval} millis`)
        }
      }).pipe(
        Effect.catchAllCause((cause) =>
          Effect.gen(function* () {
            yield* Effect.log(`Metrics collection failed: ${cause}`)
            emit.halt(cause)
          })
        ),
        Effect.fork
      )
      
      // Consumer fiber - batches and emits metrics
      const consumerFiber = yield* Effect.gen(function* () {
        while (true) {
          const metric = yield* Queue.take(queue)
          const currentBatch = yield* Ref.get(batchRef)
          const newBatch = [...currentBatch, metric]
          
          if (newBatch.length >= batchSize) {
            const success = emit.array(newBatch)
            if (success) {
              yield* Ref.set(batchRef, [])
            } else {
              // Backpressure detected - slow down collection
              yield* Effect.sleep("100 millis")
            }
          } else {
            yield* Ref.set(batchRef, newBatch)
          }
        }
      }).pipe(
        Effect.catchAllCause((cause) =>
          Effect.gen(function* () {
            yield* Effect.log(`Metrics emission failed: ${cause}`)
            emit.halt(cause)
          })
        ),
        Effect.fork
      )
      
      return { producerFiber, consumerFiber, queue }
    }).pipe(
      Effect.acquireRelease(({ producerFiber, consumerFiber, queue }) =>
        Effect.gen(function* () {
          yield* Fiber.interrupt(producerFiber)
          yield* Fiber.interrupt(consumerFiber)
          yield* Queue.shutdown(queue)
          yield* Effect.log("Metrics collection stopped")
        })
      )
    )
  )

// Usage with processing pipeline
const metricsProcessing = createMetricsStream(50).pipe(
  Stream.tap((batch) => Effect.log(`Processing ${batch.length} metrics`)),
  Stream.mapEffect((batch) =>
    Effect.gen(function* () {
      // Process batch (e.g., send to monitoring system)
      const processed = batch.map(metric => ({
        ...metric,
        processed: true,
        processedAt: Date.now()
      }))
      return processed
    })
  ),
  Stream.buffer({ capacity: 10, strategy: "dropping" })
)
```

### Example 3: File System Watcher Stream

Implementing a file system watcher that monitors directory changes with proper resource cleanup.

```typescript
import { Stream, Effect, Chunk } from "effect"
import * as fs from "fs"
import * as path from "path"

interface FileEvent {
  readonly type: 'created' | 'modified' | 'deleted'
  readonly path: string
  readonly timestamp: number
}

interface WatcherConfig {
  readonly directory: string
  readonly recursive: boolean
  readonly filter?: (path: string) => boolean
}

const createFileWatcherStream = (config: WatcherConfig) =>
  Stream.asyncPush<FileEvent, Error>((emit) =>
    Effect.gen(function* () {
      yield* Effect.log(`Starting file watcher for: ${config.directory}`)
      
      // Verify directory exists
      const dirExists = yield* Effect.tryPromise({
        try: () => fs.promises.access(config.directory),
        catch: (error) => new Error(`Directory not accessible: ${error}`)
      })
      
      const watcher = fs.watch(
        config.directory,
        { recursive: config.recursive },
        (eventType, filename) => {
          if (!filename) return
          
          const fullPath = path.join(config.directory, filename)
          
          // Apply filter if provided
          if (config.filter && !config.filter(fullPath)) {
            return
          }
          
          const fileEvent: FileEvent = {
            type: eventType as 'created' | 'modified',
            path: fullPath,
            timestamp: Date.now()
          }
          
          const success = emit.single(fileEvent)
          if (!success) {
            // Handle backpressure - could implement buffering here
            console.warn(`Dropped file event for ${fullPath} due to backpressure`)
          }
        }
      )
      
      watcher.on('error', (error) => {
        emit.fail(new Error(`File watcher error: ${error.message}`))
      })
      
      return watcher
    }).pipe(
      Effect.acquireRelease((watcher) =>
        Effect.gen(function* () {
          yield* Effect.log("Stopping file watcher")
          watcher.close()
        })
      )
    )
  )

// Usage with file processing pipeline
const processFileChanges = (directory: string) =>
  createFileWatcherStream({
    directory,
    recursive: true,
    filter: (path) => path.endsWith('.json') || path.endsWith('.yaml')
  }).pipe(
    Stream.debounce("500 millis"), // Debounce rapid changes
    Stream.mapEffect((event) =>
      Effect.gen(function* () {
        // Read and validate file content
        const content = yield* Effect.tryPromise({
          try: () => fs.promises.readFile(event.path, 'utf8'),
          catch: (error) => new Error(`Failed to read ${event.path}: ${error}`)
        })
        
        return {
          ...event,
          content,
          size: content.length
        }
      })
    ),
    Stream.tap((event) => 
      Effect.log(`Processed file change: ${event.type} ${event.path} (${event.size} bytes)`)
    )
  )
```

## Advanced Features Deep Dive

### Feature 1: Emission Control and Backpressure

StreamEmit provides sophisticated control over data emission with built-in backpressure handling through buffer strategies and emission feedback.

#### Basic Emission Control

```typescript
import { Stream, Effect, Chunk } from "effect"

// Different emission methods for various use cases
const demonstrateEmissionMethods = Stream.async<string | number>(
  (emit) => {
    let counter = 0
    
    const interval = setInterval(() => {
      counter++
      
      if (counter <= 3) {
        // Emit single values
        emit.single(`Message ${counter}`)
      } else if (counter <= 6) {
        // Emit chunks for batch processing
        emit.chunk(Chunk.make(`Batch ${counter}`, `Extra ${counter}`))
      } else if (counter <= 9) {
        // Emit from Effect computations
        emit.fromEffect(
          Effect.gen(function* () {
            const computed = yield* Effect.succeed(counter * 10)
            return `Computed: ${computed}`
          })
        )
      } else {
        // Signal completion
        emit.end()
        clearInterval(interval)
      }
    }, 500)
    
    return Effect.sync(() => clearInterval(interval))
  }
)
```

#### Advanced Backpressure Handling

```typescript
import { Stream, Effect, Ref, Queue } from "effect"

// Custom backpressure handling with adaptive emission rates
const createAdaptiveStream = <T>(
  source: () => Effect.Effect<T>,
  baseInterval: number = 1000
) =>
  Stream.asyncPush<T>((emit) =>
    Effect.gen(function* () {
      const intervalRef = yield* Ref.make(baseInterval)
      const backpressureCount = yield* Ref.make(0)
      
      const producer = yield* Effect.gen(function* () {
        while (true) {
          const data = yield* source()
          const success = emit.single(data)
          
          if (!success) {
            // Backpressure detected - slow down
            const count = yield* Ref.updateAndGet(backpressureCount, n => n + 1)
            const newInterval = baseInterval * Math.pow(1.5, Math.min(count, 5))
            yield* Ref.set(intervalRef, newInterval)
            yield* Effect.log(`Backpressure detected, slowing to ${newInterval}ms`)
          } else {
            // Reset backpressure counter on successful emission
            yield* Ref.set(backpressureCount, 0)
            yield* Ref.set(intervalRef, baseInterval)
          }
          
          const currentInterval = yield* Ref.get(intervalRef)
          yield* Effect.sleep(`${currentInterval} millis`)
        }
      }).pipe(Effect.fork)
      
      return producer
    }).pipe(
      Effect.acquireRelease((fiber) => Fiber.interrupt(fiber))
    )
  )
```

### Feature 2: Error Handling Strategies

StreamEmit provides multiple ways to handle errors during stream emission, from simple failures to complex cause handling.

#### Comprehensive Error Handling

```typescript
import { Stream, Effect, Cause, Exit } from "effect"

class StreamError extends Error {
  readonly _tag = "StreamError"
  constructor(
    readonly message: string,
    readonly code: string,
    readonly cause?: unknown
  ) {
    super(message)
  }
}

// Advanced error handling with recovery strategies
const createResilientStream = <T>(
  producer: () => Effect.Effect<T, Error>
) =>
  Stream.async<T, StreamError>((emit) => {
    let retryCount = 0
    const maxRetries = 3
    
    const tryEmit = () => {
      Effect.runPromise(
        producer().pipe(
          Effect.flatMap((data) => Effect.sync(() => emit.single(data))),
          Effect.catchAll((error) =>
            Effect.gen(function* () {
              retryCount++
              
              if (retryCount <= maxRetries) {
                yield* Effect.log(`Retry ${retryCount}/${maxRetries} after error: ${error.message}`)
                yield* Effect.sleep(`${retryCount * 1000} millis`)
                setTimeout(tryEmit, 0) // Schedule retry
              } else {
                // Max retries exceeded - fail the stream
                const streamError = new StreamError(
                  `Failed after ${maxRetries} retries`,
                  "MAX_RETRIES_EXCEEDED",
                  error
                )
                emit.fail(streamError)
              }
            })
          ),
          Effect.catchAllDefect((defect) =>
            Effect.gen(function* () {
              yield* Effect.log(`Unrecoverable defect: ${defect}`)
              emit.die(defect)
            })
          )
        )
      )
    }
    
    // Start the emission process
    tryEmit()
    
    const interval = setInterval(tryEmit, 5000)
    return Effect.sync(() => clearInterval(interval))
  })

// Usage with exit handling
const processWithExitHandling = <T>(
  stream: Stream.Stream<T, StreamError>
) =>
  stream.pipe(
    Stream.mapEffect((data) =>
      Effect.gen(function* () {
        // Process data and handle exit states
        const result = yield* processData(data)
        return Exit.succeed(result)
      }).pipe(
        Effect.catchAll((error) => Effect.succeed(Exit.fail(error)))
      )
    ),
    Stream.mapEffect((exit) =>
      Exit.match(exit, {
        onFailure: (error) => Effect.log(`Processing failed: ${error}`),
        onSuccess: (result) => Effect.succeed(result)
      })
    )
  )
```

### Feature 3: Resource-Safe Stream Composition

StreamEmit integrates seamlessly with Effect's resource management system for safe stream composition.

#### Complex Resource Management

```typescript
import { Stream, Effect, Scope, Resource } from "effect"

interface DatabaseConnection {
  readonly query: (sql: string) => Effect.Effect<unknown[]>
  readonly close: Effect.Effect<void>
}

interface RedisClient {
  readonly get: (key: string) => Effect.Effect<string | null>
  readonly disconnect: Effect.Effect<void>
}

// Multi-resource stream with proper cleanup coordination
const createDataSyncStream = Stream.asyncScoped<unknown[], Error>((emit) =>
  Effect.gen(function* () {
    // Acquire multiple resources
    const db = yield* Resource.make(
      connectToDatabase(),
      (conn) => conn.close
    )
    
    const redis = yield* Resource.make(
      connectToRedis(),
      (client) => client.disconnect
    )
    
    yield* Effect.log("Starting data synchronization")
    
    // Set up change stream
    const syncData = () =>
      Effect.gen(function* () {
        // Get data from database
        const dbData = yield* db.query("SELECT * FROM recent_changes")
        
        // Check cache for processing flags
        const cacheKey = `sync_${Date.now()}`
        const cached = yield* redis.get(cacheKey)
        
        if (!cached) {
          emit.fromEffectChunk(
            Effect.succeed(Chunk.fromIterable(dbData))
          )
        }
      }).pipe(
        Effect.catchAll((error) =>
          Effect.gen(function* () {
            yield* Effect.log(`Sync error: ${error}`)
            emit.fail(error)
          })
        )
      )
    
    // Start periodic sync
    const syncInterval = setInterval(() => {
      Effect.runPromise(syncData())
    }, 10000)
    
    // Return cleanup function
    return Effect.sync(() => {
      clearInterval(syncInterval)
    })
  })
)
```

## Practical Patterns & Best Practices

### Pattern 1: Emission Rate Limiting

```typescript
import { Stream, Effect, Ref } from "effect"

// Rate-limited emission helper
const createRateLimitedEmitter = <T>(
  emitsPerSecond: number
) => {
  const tokenBucket = (capacity: number, refillRate: number) =>
    Effect.gen(function* () {
      const tokensRef = yield* Ref.make(capacity)
      const lastRefillRef = yield* Ref.make(Date.now())
      
      const tryEmit = yield* Effect.gen(function* () {
        const now = Date.now()
        const lastRefill = yield* Ref.get(lastRefillRef)
        const timeDelta = now - lastRefill
        
        // Refill tokens based on time passed
        const tokensToAdd = Math.floor((timeDelta / 1000) * refillRate)
        if (tokensToAdd > 0) {
          yield* Ref.update(tokensRef, tokens => Math.min(capacity, tokens + tokensToAdd))
          yield* Ref.set(lastRefillRef, now)
        }
        
        const tokens = yield* Ref.get(tokensRef)
        if (tokens > 0) {
          yield* Ref.update(tokensRef, n => n - 1)
          return true
        }
        return false
      })
      
      return { tryEmit }
    })
  
  return Stream.asyncPush<T>((emit) =>
    Effect.gen(function* () {
      const { tryEmit } = yield* tokenBucket(emitsPerSecond, emitsPerSecond)
      const queue: T[] = []
      
      const processQueue = () =>
        Effect.gen(function* () {
          if (queue.length > 0) {
            const canEmit = yield* tryEmit
            if (canEmit) {
              const item = queue.shift()!
              emit.single(item)
            }
          }
        })
      
      const interval = setInterval(() => {
        Effect.runPromise(processQueue())
      }, 100)
      
      return {
        enqueue: (item: T) => queue.push(item),
        cleanup: () => clearInterval(interval)
      }
    }).pipe(
      Effect.acquireRelease(({ cleanup }) => Effect.sync(cleanup))
    )
  )
}
```

### Pattern 2: Stream Health Monitoring

```typescript
import { Stream, Effect, Ref, Metric } from "effect"

// Stream health monitoring with metrics
const withHealthMonitoring = <T, E, R>(
  stream: Stream.Stream<T, E, R>,
  name: string
) => {
  const emissionCounter = Metric.counter(`${name}_emissions_total`)
  const errorCounter = Metric.counter(`${name}_errors_total`)
  const backpressureCounter = Metric.counter(`${name}_backpressure_total`)
  
  return Stream.asyncPush<T, E>((emit) =>
    Effect.gen(function* () {
      const healthRef = yield* Ref.make({
        lastEmission: Date.now(),
        totalEmissions: 0,
        errors: 0,
        backpressureEvents: 0
      })
      
      const monitoredEmit = {
        single: (value: T) => {
          const success = emit.single(value)
          
          Effect.runSync(
            Effect.gen(function* () {
              if (success) {
                yield* Metric.increment(emissionCounter)
                yield* Ref.update(healthRef, h => ({
                  ...h,
                  lastEmission: Date.now(),
                  totalEmissions: h.totalEmissions + 1
                }))
              } else {
                yield* Metric.increment(backpressureCounter)
                yield* Ref.update(healthRef, h => ({
                  ...h,
                  backpressureEvents: h.backpressureEvents + 1
                }))
              }
            })
          )
          
          return success
        },
        
        fail: (error: E) => {
          Effect.runSync(
            Effect.gen(function* () {
              yield* Metric.increment(errorCounter)
              yield* Ref.update(healthRef, h => ({
                ...h,
                errors: h.errors + 1
              }))
            })
          )
          
          emit.fail(error)
        },
        
        end: () => emit.end()
      }
      
      // Health check fiber
      const healthCheckFiber = yield* Effect.gen(function* () {
        while (true) {
          const health = yield* Ref.get(healthRef)
          const timeSinceLastEmission = Date.now() - health.lastEmission
          
          if (timeSinceLastEmission > 30000) { // 30 seconds
            yield* Effect.log(`Stream ${name} appears stalled - last emission ${timeSinceLastEmission}ms ago`)
          }
          
          yield* Effect.sleep("10 seconds")
        }
      }).pipe(Effect.fork)
      
      return { healthCheckFiber, monitoredEmit }
    }).pipe(
      Effect.acquireRelease(({ healthCheckFiber }) => Fiber.interrupt(healthCheckFiber))
    )
  )
}
```

### Pattern 3: Stream Coordination

```typescript
// Coordinated stream emission with synchronization
const createCoordinatedStreams = <T>(
  sources: Array<() => Effect.Effect<T>>,
  coordinator: (items: T[]) => Effect.Effect<T[]>
) =>
  Stream.asyncPush<T[]>((emit) =>
    Effect.gen(function* () {
      const buffers = yield* Effect.forEach(sources, () => Ref.make<T[]>([]))
      const coordinationQueue = yield* Queue.bounded<number>(sources.length)
      
      // Create source streams
      const sourceFibers = yield* Effect.forEach(sources, (source, index) =>
        Effect.gen(function* () {
          while (true) {
            const item = yield* source()
            yield* Ref.update(buffers[index], buffer => [...buffer, item])
            yield* Queue.offer(coordinationQueue, index)
          }
        }).pipe(Effect.fork)
      )
      
      // Coordination fiber
      const coordinatorFiber = yield* Effect.gen(function* () {
        while (true) {
          yield* Queue.take(coordinationQueue) // Wait for any source to emit
          
          // Collect from all buffers
          const allBuffers = yield* Effect.forEach(buffers, Ref.get)
          const hasData = allBuffers.some(buffer => buffer.length > 0)
          
          if (hasData) {
            const allItems = allBuffers.flat()
            
            if (allItems.length > 0) {
              const coordinated = yield* coordinator(allItems)
              emit.array(coordinated)
              
              // Clear buffers
              yield* Effect.forEach(buffers, buffer => Ref.set(buffer, []))
            }
          }
        }
      }).pipe(Effect.fork)
      
      return { sourceFibers, coordinatorFiber }
    }).pipe(
      Effect.acquireRelease(({ sourceFibers, coordinatorFiber }) =>
        Effect.gen(function* () {
          yield* Effect.forEach(sourceFibers, Fiber.interrupt)
          yield* Fiber.interrupt(coordinatorFiber)
        })
      )
    )
  )
```

## Integration Examples

### Integration with HTTP Streaming

```typescript
import { Stream, Effect, Context, Layer } from "effect"
import { HttpClient, HttpClientRequest } from "@effect/platform"

// HTTP streaming integration
const createHttpStreamingClient = (url: string) =>
  Stream.asyncPush<string>((emit) =>
    Effect.gen(function* () {
      const client = yield* HttpClient.HttpClient
      
      const request = HttpClientRequest.get(url).pipe(
        HttpClientRequest.setHeader("Accept", "text/event-stream")
      )
      
      const response = yield* client.execute(request)
      const stream = response.stream
      
      // Process streaming response
      const reader = stream.getReader()
      const decoder = new TextDecoder()
      
      const readLoop = () =>
        Effect.gen(function* () {
          while (true) {
            const { done, value } = yield* Effect.tryPromise({
              try: () => reader.read(),
              catch: (error) => new Error(`Stream read error: ${error}`)
            })
            
            if (done) {
              emit.end()
              break
            }
            
            const chunk = decoder.decode(value, { stream: true })
            const lines = chunk.split('\n').filter(line => line.trim())
            
            for (const line of lines) {
              if (line.startsWith('data: ')) {
                const data = line.slice(6)
                emit.single(data)
              }
            }
          }
        }).pipe(
          Effect.catchAll((error) =>
            Effect.gen(function* () {
              yield* Effect.log(`HTTP stream error: ${error}`)
              emit.fail(error)
            })
          )
        )
      
      yield* Effect.fork(readLoop())
      
      return reader
    }).pipe(
      Effect.acquireRelease((reader) =>
        Effect.tryPromise({
          try: () => reader.cancel(),
          catch: () => new Error("Failed to cancel reader")
        }).pipe(Effect.orDie)
      )
    )
  )

// Usage with HTTP client layer
const httpStreamingLayer = Layer.effect(
  HttpClient.HttpClient,
  HttpClient.makeDefault
)

const processHttpStream = (url: string) =>
  createHttpStreamingClient(url).pipe(
    Stream.mapEffect((data) =>
      Effect.try({
        try: () => JSON.parse(data),
        catch: (error) => new Error(`Invalid JSON: ${error}`)
      })
    ),
    Stream.tap((event) => Effect.log(`Received event: ${JSON.stringify(event)}`))
  ).pipe(
    Stream.provideLayer(httpStreamingLayer)
  )
```

### Testing Strategies

```typescript
import { Stream, Effect, TestContext, TestClock, Ref } from "effect"
import { describe, it, expect } from "@effect/vitest"

// Testing StreamEmit-based streams
describe("StreamEmit Testing", () => {
  it("should handle emission control", () =>
    Effect.gen(function* () {
      const emittedItems = yield* Ref.make<number[]>([])
      
      const testStream = Stream.async<number>((emit) => {
        emit.single(1)
        emit.single(2)
        emit.single(3)
        emit.end()
      })
      
      yield* testStream.pipe(
        Stream.runForEach((item) => Ref.update(emittedItems, items => [...items, item]))
      )
      
      const result = yield* Ref.get(emittedItems)
      expect(result).toEqual([1, 2, 3])
    }).pipe(Effect.provide(TestContext.TestContext))
  )
  
  it("should handle backpressure correctly", () =>
    Effect.gen(function* () {
      const backpressureEvents = yield* Ref.make(0)
      
      const testStream = Stream.asyncPush<number>((emit) =>
        Effect.gen(function* () {
          for (let i = 0; i < 100; i++) {
            const success = emit.single(i)
            if (!success) {
              yield* Ref.update(backpressureEvents, n => n + 1)
            }
          }
          emit.end()
        })
      )
      
      // Consume with limited buffer
      yield* testStream.pipe(
        Stream.buffer({ capacity: 5, strategy: "dropping" }),
        Stream.runDrain
      )
      
      const events = yield* Ref.get(backpressureEvents)
      expect(events).toBeGreaterThan(0)
    }).pipe(Effect.provide(TestContext.TestContext))
  )
  
  it("should handle errors properly", () =>
    Effect.gen(function* () {
      const testStream = Stream.async<number, string>((emit) => {
        emit.single(1)
        emit.fail("Test error")
      })
      
      const result = yield* testStream.pipe(
        Stream.runCollect,
        Effect.either
      )
      
      expect(result._tag).toBe("Left")
      if (result._tag === "Left") {
        expect(result.left).toBe("Test error")
      }
    }).pipe(Effect.provide(TestContext.TestContext))
  )
})

// Property-based testing helpers
const generateStreamEmitTest = <T>(
  generator: Effect.Effect<T>,
  predicate: (items: T[]) => boolean
) =>
  Effect.gen(function* () {
    const items: T[] = []
    
    const testStream = Stream.asyncPush<T>((emit) =>
      Effect.gen(function* () {
        for (let i = 0; i < 50; i++) {
          const item = yield* generator
          items.push(item)
          emit.single(item)
        }
        emit.end()
      })
    )
    
    const collected = yield* Stream.runCollect(testStream)
    const result = predicate(Chunk.toReadonlyArray(collected))
    
    return { items, collected: Chunk.toReadonlyArray(collected), result }
  })
```

## Conclusion

StreamEmit provides powerful abstractions for creating streams from push-based data sources, external events, and asynchronous callbacks with built-in resource management and backpressure handling.

Key benefits:
- **Resource Safety** - Automatic cleanup of resources with Effect's resource management
- **Backpressure Control** - Built-in flow control mechanisms prevent memory issues
- **Type Safety** - Full type safety for emission patterns and error handling
- **Composability** - Seamless integration with other Effect modules and stream operations

StreamEmit is ideal when you need to create streams from external event sources, implement custom data producers, or integrate with push-based APIs while maintaining full control over emission timing and resource lifecycle.