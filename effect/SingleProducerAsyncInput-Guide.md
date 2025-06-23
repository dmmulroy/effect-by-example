# SingleProducerAsyncInput: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem SingleProducerAsyncInput Solves

When building async streaming systems, a common challenge is coordinating between a single producer and multiple consumers with specific constraints:

```typescript
// Traditional approach - using regular queues or observables
class TraditionalAsyncBuffer<T> {
  private buffer: T[] = []
  private consumers: Array<(value: T) => void> = []
  
  produce(value: T) {
    // Problem 1: No backpressure - producer can outpace consumers
    this.buffer.push(value)
    this.notifyConsumers()
  }
  
  consume(callback: (value: T) => void) {
    // Problem 2: All consumers see all values (broadcasting)
    this.consumers.push(callback)
  }
  
  private notifyConsumers() {
    // Problem 3: No coordination between producer and consumers
    // Problem 4: Complex error handling and completion logic
    while (this.buffer.length > 0 && this.consumers.length > 0) {
      const value = this.buffer.shift()!
      this.consumers.forEach(callback => callback(value))
    }
  }
}
```

This approach leads to:
- **Unbounded Buffering** - Producer can overwhelm consumers with data
- **Broadcasting Semantics** - All consumers receive all values instead of work distribution
- **No Backpressure** - No mechanism to slow down production based on consumption
- **Complex State Management** - Manual coordination of producer/consumer lifecycle
- **Poor Error Handling** - Errors don't propagate cleanly between producer and consumers

### The SingleProducerAsyncInput Solution

SingleProducerAsyncInput provides an MVar-like abstraction specifically designed for single-producer, multiple-consumer scenarios with built-in coordination:

```typescript
import { SingleProducerAsyncInput, Effect, Channel } from "effect"

// Create a coordinated async input system
const createAsyncInput = <T>() => SingleProducerAsyncInput.make<never, T, void>()

// Producer waits for consumers to be ready
const producer = Effect.gen(function* () {
  const input = yield* createAsyncInput<string>()
  
  // Emit waits for a consumer to pick up the value
  yield* input.emit("data-1")
  yield* input.emit("data-2")
  yield* input.done()
  
  return input
})
```

### Key Concepts

**Single Producer**: Only one producer can emit values, but multiple consumers can read from it

**Buffer Size 1**: Maintains at most one value in the buffer, preventing unbounded memory growth

**Work Distribution**: Each emitted element is consumed by exactly one consumer (not broadcasted)

**Backpressure**: Producer waits for consumer to pick up values before emitting new ones

**Terminal States**: Done and error signals persist and can be read by multiple consumers

## Basic Usage Patterns

### Pattern 1: Creating and Basic Operations

```typescript
import { SingleProducerAsyncInput, Effect, Console } from "effect"

// Create a single producer async input
const basicExample = Effect.gen(function* () {
  const input = yield* SingleProducerAsyncInput.make<never, string, void>()
  
  // Producer side - emit values
  yield* input.emit("hello")
  yield* input.emit("world")
  yield* input.done()
  
  return input
})
```

### Pattern 2: Producer/Consumer Coordination

```typescript
import { SingleProducerAsyncInput, Effect, Fiber } from "effect"

const producerConsumerExample = Effect.gen(function* () {
  const input = yield* SingleProducerAsyncInput.make<never, number, string>()
  
  // Start consumer fiber
  const consumerFiber = yield* Effect.fork(
    Effect.gen(function* () {
      while (true) {
        const result = yield* input.takeWith(
          (cause) => Effect.logError(`Consumer error: ${cause}`),
          (value) => Effect.log(`Consumed: ${value}`),
          (done) => Effect.log(`Done: ${done}`)
        )
        yield* result
      }
    })
  )
  
  // Producer emits values with coordination
  yield* input.emit(1)
  yield* input.emit(2) 
  yield* input.emit(3)
  yield* input.done("completed")
  
  yield* Fiber.join(consumerFiber)
})
```

### Pattern 3: Integration with Channels

```typescript
import { SingleProducerAsyncInput, Channel, Stream, Effect } from "effect"

const channelIntegration = Effect.gen(function* () {
  const input = yield* SingleProducerAsyncInput.make<never, string, void>()
  
  // Create a channel from the async input
  const channel = Channel.fromInput(input)
  
  // Convert to stream for processing
  const stream = Stream.fromChannel(channel)
  
  // Start processing in background
  const processFiber = yield* Effect.fork(
    stream.pipe(
      Stream.tap(value => Console.log(`Processing: ${value}`)),
      Stream.runDrain
    )
  )
  
  // Feed data to the input
  yield* input.emit("task-1")
  yield* input.emit("task-2")
  yield* input.done()
  
  yield* Fiber.join(processFiber)
})
```

## Real-World Examples

### Example 1: HTTP Request Processing Pipeline

Managing incoming HTTP requests with backpressure to prevent server overload:

```typescript
import { 
  SingleProducerAsyncInput, 
  Effect, 
  Channel, 
  Stream, 
  Queue, 
  Fiber,
  Data,
  Layer,
  Context
} from "effect"

// Domain model
class HttpRequest extends Data.Class<{
  id: string
  method: string
  path: string
  body: string
}> {}

class HttpResponse extends Data.Class<{
  requestId: string
  status: number
  body: string
}> {}

class ProcessingError extends Data.TaggedError("ProcessingError")<{
  requestId: string
  message: string
}> {}

// HTTP Request Processor Service
class HttpProcessor extends Context.Tag("HttpProcessor")<
  HttpProcessor,
  {
    processRequest: (request: HttpRequest) => Effect.Effect<HttpResponse, ProcessingError>
  }
>() {}

const makeHttpProcessor = Effect.gen(function* () {
  const processRequest = (request: HttpRequest) =>
    Effect.gen(function* () {
      // Simulate processing time
      yield* Effect.sleep("100 millis")
      
      if (request.path === "/error") {
        return yield* Effect.fail(new ProcessingError({
          requestId: request.id,
          message: "Simulated processing error"
        }))
      }
      
      return new HttpResponse({
        requestId: request.id,
        status: 200,
        body: `Processed ${request.method} ${request.path}`
      })
    })
  
  return HttpProcessor.of({ processRequest })
})

// HTTP Request Pipeline using SingleProducerAsyncInput
const createRequestPipeline = Effect.gen(function* () {
  const processor = yield* HttpProcessor
  
  // Create async input for incoming requests
  const requestInput = yield* SingleProducerAsyncInput.make<
    never, 
    HttpRequest, 
    void
  >()
  
  // Create response queue for completed requests
  const responseQueue = yield* Queue.bounded<
    Effect.Effect<HttpResponse, ProcessingError>
  >(100)
  
  // Convert input to stream and process requests
  const requestStream = Stream.fromChannel(Channel.fromInput(requestInput))
  
  // Start request processing pipeline
  const processingFiber = yield* Effect.fork(
    requestStream.pipe(
      Stream.mapEffect(request => 
        processor.processRequest(request).pipe(
          Effect.tap(response => 
            Effect.log(`✅ Processed request ${response.requestId}`)
          ),
          Effect.tapError(error =>
            Effect.logError(`❌ Failed to process request ${error.requestId}: ${error.message}`)
          )
        )
      ),
      Stream.tap(responseEffect => Queue.offer(responseQueue, responseEffect)),
      Stream.runDrain
    )
  )
  
  // Response handler
  const responseHandler = Effect.gen(function* () {
    while (true) {
      const responseEffect = yield* Queue.take(responseQueue)
      const result = yield* Effect.either(responseEffect)
      
      yield* result.pipe(
        Effect.match({
          onLeft: error => Effect.log(`Response error: ${error.message}`),
          onRight: response => Effect.log(`Response ready: ${response.status}`)
        })
      )
    }
  })
  
  const responseHandlerFiber = yield* Effect.fork(responseHandler)
  
  // API for adding requests (with natural backpressure)
  const addRequest = (request: HttpRequest) =>
    Effect.gen(function* () {
      yield* Effect.log(`📨 Queuing request ${request.id}`)
      // This will wait if consumers can't keep up (backpressure)
      yield* requestInput.emit(request)
      yield* Effect.log(`✅ Request ${request.id} accepted`)
    })
  
  const shutdown = Effect.gen(function* () {
    yield* requestInput.done()
    yield* Queue.shutdown(responseQueue)
    yield* Fiber.join(processingFiber)
    yield* Fiber.interrupt(responseHandlerFiber)
  })
  
  return { addRequest, shutdown }
})

// Usage example
const httpPipelineExample = Effect.gen(function* () {
  const pipeline = yield* createRequestPipeline
  
  // Simulate incoming requests
  const requests = [
    new HttpRequest({ id: "1", method: "GET", path: "/users", body: "" }),
    new HttpRequest({ id: "2", method: "POST", path: "/users", body: '{"name":"John"}' }),
    new HttpRequest({ id: "3", method: "GET", path: "/error", body: "" }),
    new HttpRequest({ id: "4", method: "DELETE", path: "/users/1", body: "" })
  ]
  
  // Send requests (they'll be processed with natural backpressure)
  for (const request of requests) {
    yield* pipeline.addRequest(request)
  }
  
  // Wait a bit for processing to complete
  yield* Effect.sleep("1 second")
  
  // Shutdown gracefully
  yield* pipeline.shutdown
}).pipe(
  Effect.provide(Layer.succeed(HttpProcessor, makeHttpProcessor))
)
```

### Example 2: File Processing with Progress Tracking

Processing large files chunk by chunk with progress reporting:

```typescript
import { 
  SingleProducerAsyncInput, 
  Effect, 
  Stream, 
  Channel,
  Ref,
  Chunk,
  Data,
  Context
} from "effect"

// Domain models
class FileChunk extends Data.Class<{
  chunkId: number
  data: Uint8Array
  isLast: boolean
}> {}

class ProcessingProgress extends Data.Class<{
  totalChunks: number
  processedChunks: number
  bytesProcessed: number
  isComplete: boolean
}> {}

class FileProcessingError extends Data.TaggedError("FileProcessingError")<{
  chunkId: number
  reason: string
}> {}

// File Processor Service
class FileProcessor extends Context.Tag("FileProcessor")<
  FileProcessor,
  {
    processChunk: (chunk: FileChunk) => Effect.Effect<Uint8Array, FileProcessingError>
    reportProgress: (progress: ProcessingProgress) => Effect.Effect<void>
  }
>() {}

const makeFileProcessor = Effect.gen(function* () {
  const processChunk = (chunk: FileChunk) =>
    Effect.gen(function* () {
      // Simulate chunk processing (e.g., compression, encryption)
      yield* Effect.sleep("50 millis")
      
      if (chunk.data.length === 0) {
        return yield* Effect.fail(new FileProcessingError({
          chunkId: chunk.chunkId,
          reason: "Empty chunk"
        }))
      }
      
      // Simulate processing transformation
      const processed = new Uint8Array(chunk.data.length)
      for (let i = 0; i < chunk.data.length; i++) {
        processed[i] = chunk.data[i] ^ 0xFF // Simple XOR transformation
      }
      
      return processed
    })
  
  const reportProgress = (progress: ProcessingProgress) =>
    Effect.log(
      `📊 Progress: ${progress.processedChunks}/${progress.totalChunks} chunks ` +
      `(${Math.round((progress.processedChunks / progress.totalChunks) * 100)}%) ` +
      `- ${progress.bytesProcessed} bytes processed`
    )
  
  return FileProcessor.of({ processChunk, reportProgress })
})

// File Processing Pipeline
const createFileProcessingPipeline = (totalChunks: number) =>
  Effect.gen(function* () {
    const processor = yield* FileProcessor
    
    // Create async input for file chunks
    const chunkInput = yield* SingleProducerAsyncInput.make<
      never,
      FileChunk,
      void
    >()
    
    // Progress tracking
    const progressRef = yield* Ref.make(new ProcessingProgress({
      totalChunks,
      processedChunks: 0,
      bytesProcessed: 0,
      isComplete: false
    }))
    
    // Processed chunks storage
    const processedChunks = yield* Ref.make<Chunk.Chunk<Uint8Array>>(Chunk.empty())
    
    // Create processing stream
    const chunkStream = Stream.fromChannel(Channel.fromInput(chunkInput))
    
    // Start chunk processing
    const processingFiber = yield* Effect.fork(
      chunkStream.pipe(
        Stream.mapEffect(chunk =>
          Effect.gen(function* () {
            const processedData = yield* processor.processChunk(chunk)
            
            // Update progress
            yield* Ref.update(progressRef, progress => new ProcessingProgress({
              ...progress,
              processedChunks: progress.processedChunks + 1,
              bytesProcessed: progress.bytesProcessed + chunk.data.length,
              isComplete: chunk.isLast && progress.processedChunks === totalChunks - 1
            }))
            
            // Store processed chunk
            yield* Ref.update(processedChunks, chunks => 
              Chunk.append(chunks, processedData)
            )
            
            // Report progress
            const currentProgress = yield* Ref.get(progressRef)
            yield* processor.reportProgress(currentProgress)
            
            return processedData
          })
        ),
        Stream.runDrain
      )
    )
    
    // Progress monitoring
    const progressMonitor = Effect.gen(function* () {
      while (true) {
        yield* Effect.sleep("500 millis")
        const progress = yield* Ref.get(progressRef)
        
        if (progress.isComplete) {
          yield* Effect.log("🎉 File processing completed!")
          break
        }
      }
    })
    
    const progressFiber = yield* Effect.fork(progressMonitor)
    
    // API for feeding chunks
    const addChunk = (chunkId: number, data: Uint8Array, isLast = false) =>
      Effect.gen(function* () {
        const chunk = new FileChunk({ chunkId, data, isLast })
        yield* chunkInput.emit(chunk)
      })
    
    const finish = Effect.gen(function* () {
      yield* chunkInput.done()
      yield* Fiber.join(processingFiber)
      yield* Fiber.interrupt(progressFiber)
      
      return yield* Ref.get(processedChunks)
    })
    
    return { addChunk, finish }
  })

// Usage example
const fileProcessingExample = Effect.gen(function* () {
  // Simulate a file with 5 chunks
  const totalChunks = 5
  const pipeline = yield* createFileProcessingPipeline(totalChunks)
  
  // Generate sample file chunks
  const generateChunkData = (chunkId: number) => {
    const data = new Uint8Array(1024) // 1KB chunks
    for (let i = 0; i < data.length; i++) {
      data[i] = (chunkId * 256 + i) % 256
    }
    return data
  }
  
  // Feed chunks to the pipeline
  for (let i = 0; i < totalChunks; i++) {
    const data = generateChunkData(i)
    const isLast = i === totalChunks - 1
    
    yield* pipeline.addChunk(i, data, isLast)
    
    // Simulate realistic chunk arrival timing
    yield* Effect.sleep("100 millis")
  }
  
  // Wait for completion and get results
  const processedChunks = yield* pipeline.finish
  
  yield* Effect.log(`📁 Processing complete! Generated ${Chunk.size(processedChunks)} processed chunks`)
  
  return processedChunks
}).pipe(
  Effect.provide(Layer.succeed(FileProcessor, makeFileProcessor))
)
```

### Example 3: Event Stream Processing with Multiple Consumers

Building an event processing system where different consumers handle different types of events:

```typescript
import { 
  SingleProducerAsyncInput, 
  Effect, 
  Stream, 
  Channel,
  Fiber,
  Data,
  Context,
  Queue,
  Match
} from "effect"

// Event types
abstract class BaseEvent extends Data.TaggedClass("BaseEvent")<{
  id: string
  timestamp: number
  source: string
}> {}

class UserEvent extends BaseEvent.pipe(Data.TaggedClass("UserEvent"))<{
  userId: string
  action: "login" | "logout" | "signup"
}> {}

class OrderEvent extends BaseEvent.pipe(Data.TaggedClass("OrderEvent"))<{
  orderId: string
  amount: number
  status: "created" | "paid" | "cancelled"
}> {}

class SystemEvent extends BaseEvent.pipe(Data.TaggedClass("SystemEvent"))<{
  level: "info" | "warning" | "error"
  message: string
}> {}

type DomainEvent = UserEvent | OrderEvent | SystemEvent

// Event handlers
class UserEventHandler extends Context.Tag("UserEventHandler")<
  UserEventHandler,
  {
    handleUserEvent: (event: UserEvent) => Effect.Effect<void>
  }
>() {}

class OrderEventHandler extends Context.Tag("OrderEventHandler")<
  OrderEventHandler,
  {
    handleOrderEvent: (event: OrderEvent) => Effect.Effect<void>
  }
>() {}

class SystemEventHandler extends Context.Tag("SystemEventHandler")<
  SystemEventHandler,
  {
    handleSystemEvent: (event: SystemEvent) => Effect.Effect<void>
  }
>() {}

// Create event handlers
const makeUserEventHandler = Effect.succeed(
  UserEventHandler.of({
    handleUserEvent: (event) =>
      Effect.log(`👤 User ${event.userId} performed ${event.action}`)
  })
)

const makeOrderEventHandler = Effect.succeed(
  OrderEventHandler.of({
    handleOrderEvent: (event) =>
      Effect.log(`💰 Order ${event.orderId}: ${event.status} ($${event.amount})`)
  })
)

const makeSystemEventHandler = Effect.succeed(
  SystemEventHandler.of({
    handleSystemEvent: (event) =>
      Effect.log(`🔧 System ${event.level}: ${event.message}`)
  })
)

// Multi-consumer event processing system
const createEventProcessingSystem = Effect.gen(function* () {
  const userHandler = yield* UserEventHandler
  const orderHandler = yield* OrderEventHandler
  const systemHandler = yield* SystemEventHandler
  
  // Create async input for events
  const eventInput = yield* SingleProducerAsyncInput.make<never, DomainEvent, void>()
  
  // Create separate queues for each event type
  const userEventQueue = yield* Queue.bounded<UserEvent>(50)
  const orderEventQueue = yield* Queue.bounded<OrderEvent>(50)
  const systemEventQueue = yield* Queue.bounded<SystemEvent>(50)
  
  // Event distributor - reads from input and routes to appropriate queues
  const eventDistributor = Effect.gen(function* () {
    const eventStream = Stream.fromChannel(Channel.fromInput(eventInput))
    
    return yield* eventStream.pipe(
      Stream.tap(event => 
        Match.value(event).pipe(
          Match.when({ _tag: "UserEvent" }, (e) => Queue.offer(userEventQueue, e)),
          Match.when({ _tag: "OrderEvent" }, (e) => Queue.offer(orderEventQueue, e)),
          Match.when({ _tag: "SystemEvent" }, (e) => Queue.offer(systemEventQueue, e)),
          Match.exhaustive
        )
      ),
      Stream.runDrain
    )
  })
  
  // Consumer for user events
  const userEventConsumer = Effect.gen(function* () {
    while (true) {
      const event = yield* Queue.take(userEventQueue)
      yield* userHandler.handleUserEvent(event)
    }
  })
  
  // Consumer for order events
  const orderEventConsumer = Effect.gen(function* () {
    while (true) {
      const event = yield* Queue.take(orderEventQueue)
      yield* orderHandler.handleOrderEvent(event)
    }
  })
  
  // Consumer for system events
  const systemEventConsumer = Effect.gen(function* () {
    while (true) {
      const event = yield* Queue.take(systemEventQueue)
      yield* systemHandler.handleSystemEvent(event)
    }
  })
  
  // Start all consumers
  const distributorFiber = yield* Effect.fork(eventDistributor)
  const userConsumerFiber = yield* Effect.fork(userEventConsumer)
  const orderConsumerFiber = yield* Effect.fork(orderEventConsumer)
  const systemConsumerFiber = yield* Effect.fork(systemEventConsumer)
  
  const publishEvent = (event: DomainEvent) =>
    Effect.gen(function* () {
      yield* Effect.log(`📝 Publishing ${event._tag} event: ${event.id}`)
      yield* eventInput.emit(event)
    })
  
  const shutdown = Effect.gen(function* () {
    yield* eventInput.done()
    yield* Queue.shutdown(userEventQueue)
    yield* Queue.shutdown(orderEventQueue)
    yield* Queue.shutdown(systemEventQueue)
    
    yield* Fiber.join(distributorFiber)
    yield* Fiber.interrupt(userConsumerFiber)
    yield* Fiber.interrupt(orderConsumerFiber)
    yield* Fiber.interrupt(systemConsumerFiber)
  })
  
  return { publishEvent, shutdown }
})

// Usage example
const eventProcessingExample = Effect.gen(function* () {
  const eventSystem = yield* createEventProcessingSystem
  
  // Generate sample events
  const events: DomainEvent[] = [
    new UserEvent({
      id: "u1",
      timestamp: Date.now(),
      source: "auth-service",
      userId: "user123",
      action: "login"
    }),
    new OrderEvent({
      id: "o1", 
      timestamp: Date.now(),
      source: "order-service",
      orderId: "order456",
      amount: 99.99,
      status: "created"
    }),
    new SystemEvent({
      id: "s1",
      timestamp: Date.now(), 
      source: "monitoring",
      level: "warning",
      message: "High memory usage detected"
    }),
    new UserEvent({
      id: "u2",
      timestamp: Date.now(),
      source: "auth-service", 
      userId: "user789",
      action: "signup"
    }),
    new OrderEvent({
      id: "o2",
      timestamp: Date.now(),
      source: "order-service",
      orderId: "order456",
      amount: 99.99,
      status: "paid"
    })
  ]
  
  // Publish events with realistic timing
  for (const event of events) {
    yield* eventSystem.publishEvent(event)
    yield* Effect.sleep("200 millis")
  }
  
  // Let events process
  yield* Effect.sleep("1 second")
  
  // Shutdown system
  yield* eventSystem.shutdown
}).pipe(
  Effect.provide(Layer.mergeAll(
    Layer.succeed(UserEventHandler, makeUserEventHandler),
    Layer.succeed(OrderEventHandler, makeOrderEventHandler), 
    Layer.succeed(SystemEventHandler, makeSystemEventHandler)
  ))
)
```

## Advanced Features Deep Dive

### Feature 1: Backpressure and Flow Control

SingleProducerAsyncInput provides natural backpressure by making producers wait for consumers:

#### Basic Backpressure Usage

```typescript
import { SingleProducerAsyncInput, Effect, Fiber } from "effect"

const backpressureExample = Effect.gen(function* () {
  const input = yield* SingleProducerAsyncInput.make<never, string, void>()
  
  // Slow consumer (processes 1 item per second)
  const slowConsumer = Effect.gen(function* () {
    let count = 0
    while (true) {
      const result = yield* input.takeWith(
        () => Effect.void,
        (value) => Effect.gen(function* () {
          count++
          yield* Effect.log(`🐌 Slow consumer processing: ${value} (${count})`)
          yield* Effect.sleep("1 second") // Simulate slow processing
        }),
        () => Effect.log("Slow consumer done")
      )
      yield* result
    }
  })
  
  // Fast producer (tries to emit every 100ms)
  const fastProducer = Effect.gen(function* () {
    for (let i = 1; i <= 5; i++) {
      yield* Effect.log(`🚀 Producer attempting to emit: item-${i}`)
      const startTime = Date.now()
      
      yield* input.emit(`item-${i}`)
      
      const endTime = Date.now()
      yield* Effect.log(`✅ Producer emitted item-${i} after ${endTime - startTime}ms`)
      
      yield* Effect.sleep("100 millis")
    }
    yield* input.done()
  })
  
  const consumerFiber = yield* Effect.fork(slowConsumer)
  yield* fastProducer
  yield* Fiber.join(consumerFiber)
})

// This will show the producer waiting for the slow consumer
```

#### Advanced Backpressure: Rate-Limited Producer

```typescript
import { 
  SingleProducerAsyncInput, 
  Effect, 
  Ref, 
  Schedule,
  Duration,
  Queue
} from "effect"

const createRateLimitedProducer = <T>(
  input: SingleProducerAsyncInput.AsyncInputProducer<never, T, void>,
  maxRate: number, // items per second
  windowSize: Duration.Duration = Duration.seconds(1)
) =>
  Effect.gen(function* () {
    const timestamps = yield* Ref.make<number[]>([])
    
    const rateLimitedEmit = (item: T) =>
      Effect.gen(function* () {
        const now = Date.now()
        const windowStart = now - Duration.toMillis(windowSize)
        
        // Clean old timestamps
        yield* Ref.update(timestamps, times => 
          times.filter(time => time > windowStart)
        )
        
        const currentTimes = yield* Ref.get(timestamps)
        
        if (currentTimes.length >= maxRate) {
          // Wait until we can emit
          const oldestTime = Math.min(...currentTimes)
          const waitTime = oldestTime + Duration.toMillis(windowSize) - now
          
          if (waitTime > 0) {
            yield* Effect.log(`⏳ Rate limiting: waiting ${waitTime}ms`)
            yield* Effect.sleep(Duration.millis(waitTime))
          }
        }
        
        // Emit the item
        yield* input.emit(item)
        yield* Ref.update(timestamps, times => [...times, Date.now()])
      })
    
    return { rateLimitedEmit }
  })

const rateLimitExample = Effect.gen(function* () {
  const input = yield* SingleProducerAsyncInput.make<never, string, void>()
  const { rateLimitedEmit } = yield* createRateLimitedProducer(input, 2) // 2 items per second
  
  // Consumer
  const consumer = Effect.gen(function* () {
    const stream = Stream.fromChannel(Channel.fromInput(input))
    return yield* stream.pipe(
      Stream.tap(value => Effect.log(`📦 Consumed: ${value}`)),
      Stream.runDrain
    )
  })
  
  // Producer with rate limiting
  const producer = Effect.gen(function* () {
    for (let i = 1; i <= 6; i++) {
      yield* Effect.log(`🎯 Attempting to emit item-${i}`)
      yield* rateLimitedEmit(`item-${i}`)
      yield* Effect.log(`✅ Emitted item-${i}`)
    }
    yield* input.done()
  })
  
  const consumerFiber = yield* Effect.fork(consumer)
  yield* producer
  yield* Fiber.join(consumerFiber)
})
```

### Feature 2: Error Handling and Recovery

SingleProducerAsyncInput provides sophisticated error handling with proper propagation:

#### Error Propagation Pattern

```typescript
import { 
  SingleProducerAsyncInput, 
  Effect, 
  Cause,
  Exit,
  Data
} from "effect"

class ProcessingError extends Data.TaggedError("ProcessingError")<{
  message: string
  code: number
}> {}

const errorHandlingExample = Effect.gen(function* () {
  const input = yield* SingleProducerAsyncInput.make<ProcessingError, string, void>()
  
  // Consumer with error handling
  const consumerWithRecovery = Effect.gen(function* () {
    while (true) {
      const result = yield* input.takeWith(
        (cause) => Effect.gen(function* () {
          yield* Effect.log(`❌ Consumer received error: ${Cause.pretty(cause)}`)
          
          // Handle different error types
          return yield* Cause.failureOption(cause).pipe(
            Effect.map(failure => 
              Match.value(failure).pipe(
                Match.when({ _tag: "ProcessingError" }, error => 
                  Effect.log(`🔧 Handling processing error: ${error.message} (code: ${error.code})`)
                ),
                Match.orElse(() => 
                  Effect.log(`🚨 Unknown error type`)
                )
              )
            ),
            Effect.getOrElse(() => 
              Effect.log("💥 System error - considering restart")
            )
          )
        }),
        (value) => Effect.log(`✅ Successfully processed: ${value}`),
        () => Effect.log("🏁 Processing completed normally")
      )
      
      yield* result
    }
  })
  
  // Producer that might fail
  const unreliableProducer = Effect.gen(function* () {
    const items = ["good-1", "good-2", "bad-item", "good-3", "good-4"]
    
    for (const item of items) {
      if (item === "bad-item") {
        // Send error instead of item
        yield* input.error(Cause.fail(new ProcessingError({
          message: "Failed to process bad item",
          code: 500
        })))
      } else {
        yield* input.emit(item)
      }
      
      yield* Effect.sleep("200 millis")
    }
    
    yield* input.done()
  })
  
  const consumerFiber = yield* Effect.fork(consumerWithRecovery)
  yield* unreliableProducer
  yield* Fiber.join(consumerFiber)
})
```

#### Circuit Breaker Pattern

```typescript
import { 
  SingleProducerAsyncInput, 
  Effect, 
  Ref, 
  Schedule,
  Data,
  Duration
} from "effect"

class CircuitBreakerError extends Data.TaggedError("CircuitBreakerError")<{
  reason: string
}> {}

const createCircuitBreaker = <E, A>(
  failureThreshold: number,
  resetTimeout: Duration.Duration
) =>
  Effect.gen(function* () {
    const state = yield* Ref.make<{
      failures: number
      lastFailure: number | null
      state: "closed" | "open" | "half-open"
    }>({
      failures: 0,
      lastFailure: null,
      state: "closed"
    })
    
    const execute = <R>(effect: Effect.Effect<A, E, R>) =>
      Effect.gen(function* () {
        const currentState = yield* Ref.get(state)
        const now = Date.now()
        
        // Check if we should transition from open to half-open
        if (
          currentState.state === "open" && 
          currentState.lastFailure !== null &&
          now - currentState.lastFailure > Duration.toMillis(resetTimeout)
        ) {
          yield* Ref.update(state, s => ({ ...s, state: "half-open" }))
          yield* Effect.log("🔄 Circuit breaker: OPEN → HALF-OPEN")
        }
        
        // Reject if circuit is open
        if (currentState.state === "open") {
          return yield* Effect.fail(new CircuitBreakerError({
            reason: "Circuit breaker is OPEN"
          }))
        }
        
        // Try to execute the effect
        const result = yield* Effect.either(effect)
        
        return yield* result.pipe(
          Effect.match({
            onLeft: (error) => Effect.gen(function* () {
              // Record failure
              yield* Ref.update(state, s => ({
                failures: s.failures + 1,
                lastFailure: now,
                state: s.failures + 1 >= failureThreshold ? "open" : s.state
              }))
              
              const newState = yield* Ref.get(state)
              if (newState.state === "open") {
                yield* Effect.log("🚫 Circuit breaker: CLOSED → OPEN")
              }
              
              return yield* Effect.fail(error)
            }),
            onRight: (value) => Effect.gen(function* () {
              // Reset on success
              yield* Ref.update(state, s => ({
                failures: 0,
                lastFailure: null,
                state: "closed"
              }))
              
              if (currentState.state === "half-open") {
                yield* Effect.log("✅ Circuit breaker: HALF-OPEN → CLOSED")
              }
              
              return value
            })
          })
        )
      })
    
    return { execute }
  })

const circuitBreakerExample = Effect.gen(function* () {
  const input = yield* SingleProducerAsyncInput.make<
    ProcessingError | CircuitBreakerError, 
    string, 
    void
  >()
  
  const circuitBreaker = yield* createCircuitBreaker<ProcessingError, string>(
    3, // Fail after 3 consecutive failures
    Duration.seconds(2) // Reset after 2 seconds
  )
  
  // Simulate unreliable processing
  let attempts = 0
  const unreliableProcessing = (value: string) =>
    Effect.gen(function* () {
      attempts++
      
      // Fail first 4 attempts, then succeed
      if (attempts <= 4 && value.includes("flaky")) {
        return yield* Effect.fail(new ProcessingError({
          message: `Processing failed for ${value} (attempt ${attempts})`,
          code: 500
        }))
      }
      
      return `processed-${value}`
    })
  
  // Consumer with circuit breaker
  const protectedConsumer = Effect.gen(function* () {
    while (true) {
      const result = yield* input.takeWith(
        (cause) => Effect.log(`❌ Consumer error: ${Cause.pretty(cause)}`),
        (value) => 
          circuitBreaker.execute(unreliableProcessing(value)).pipe(
            Effect.tap(result => Effect.log(`✅ Protected processing: ${result}`)),
            Effect.catchAll(error =>
              Effect.log(`⚡ Circuit breaker protected us from: ${error.message}`)
            )
          ),
        () => Effect.log("🏁 Consumer done")
      )
      
      yield* result
      yield* Effect.sleep("100 millis")
    }
  })
  
  // Producer
  const producer = Effect.gen(function* () {
    const items = [
      "flaky-item-1", "flaky-item-2", "flaky-item-3", 
      "flaky-item-4", "flaky-item-5", "good-item"
    ]
    
    for (const item of items) {
      yield* input.emit(item)
      yield* Effect.sleep("300 millis")
    }
    
    yield* input.done()
  })
  
  const consumerFiber = yield* Effect.fork(protectedConsumer)
  yield* producer
  yield* Fiber.join(consumerFiber)
})
```

## Practical Patterns & Best Practices

### Pattern 1: Resource Management with Cleanup

```typescript
import { 
  SingleProducerAsyncInput, 
  Effect, 
  Scope,
  Resource,
  Ref,
  Data
} from "effect"

class ManagedResource extends Data.Class<{
  id: string
  data: string
  cleanup: () => Effect.Effect<void>
}> {}

const createManagedProducer = <T>(
  resourceFactory: (item: T) => Effect.Effect<ManagedResource, never, Scope.Scope>
) =>
  Effect.gen(function* () {
    const input = yield* SingleProducerAsyncInput.make<never, ManagedResource, void>()
    const activeResources = yield* Ref.make<Map<string, ManagedResource>>(new Map())
    
    const emit = (item: T) =>
      Effect.scoped(
        Effect.gen(function* () {
          const resource = yield* resourceFactory(item)
          
          // Track the resource
          yield* Ref.update(activeResources, resources => 
            new Map(resources).set(resource.id, resource)
          )
          
          // Emit to consumers
          yield* input.emit(resource)
          
          // Set up cleanup when scope closes
          yield* Scope.addFinalizer(
            Effect.gen(function* () {
              yield* resource.cleanup()
              yield* Ref.update(activeResources, resources => {
                const newMap = new Map(resources)
                newMap.delete(resource.id)
                return newMap
              })
              yield* Effect.log(`🧹 Cleaned up resource: ${resource.id}`)
            })
          )
        })
      )
    
    const done = Effect.gen(function* () {
      yield* input.done()
      
      // Cleanup any remaining resources
      const resources = yield* Ref.get(activeResources)
      for (const [id, resource] of resources) {
        yield* resource.cleanup()
        yield* Effect.log(`🧹 Final cleanup of resource: ${id}`)
      }
    })
    
    return { emit, done, input }
  })

// Usage pattern
const resourceManagementExample = Effect.gen(function* () {
  const { emit, done, input } = yield* createManagedProducer((data: string) =>
    Effect.gen(function* () {
      const resource = new ManagedResource({
        id: `resource-${Math.random().toString(36).slice(2)}`,
        data,
        cleanup: () => Effect.log(`Cleaning up resource for: ${data}`)
      })
      
      yield* Effect.log(`📦 Created resource: ${resource.id}`)
      return resource
    })
  )
  
  // Consumer that processes resources
  const consumer = Effect.gen(function* () {
    const stream = Stream.fromChannel(Channel.fromInput(input))
    return yield* stream.pipe(
      Stream.tap(resource => 
        Effect.log(`⚙️ Processing resource: ${resource.id} with data: ${resource.data}`)
      ),
      Stream.take(3), // Process only first 3 resources
      Stream.runDrain
    )
  })
  
  const consumerFiber = yield* Effect.fork(consumer)
  
  // Emit resources
  yield* emit("data-1")
  yield* emit("data-2") 
  yield* emit("data-3")
  yield* emit("data-4") // This won't be processed
  
  yield* Fiber.join(consumerFiber)
  yield* done
})
```

### Pattern 2: Batching and Buffering

```typescript
import { 
  SingleProducerAsyncInput, 
  Effect, 
  Stream,
  Channel,
  Chunk,
  Ref,
  Schedule,
  Duration
} from "effect"

const createBatchingProducer = <T>(
  batchSize: number,
  flushInterval: Duration.Duration
) =>
  Effect.gen(function* () {
    const input = yield* SingleProducerAsyncInput.make<never, Chunk.Chunk<T>, void>()
    const buffer = yield* Ref.make<Chunk.Chunk<T>>(Chunk.empty())
    
    const flush = Effect.gen(function* () {
      const currentBuffer = yield* Ref.get(buffer)
      
      if (Chunk.size(currentBuffer) > 0) {
        yield* input.emit(currentBuffer)
        yield* Ref.set(buffer, Chunk.empty())
        yield* Effect.log(`📦 Flushed batch of ${Chunk.size(currentBuffer)} items`)
      }
    })
    
    // Periodic flush
    const flushScheduler = Effect.repeat(
      flush,
      Schedule.spaced(flushInterval)
    )
    
    const flushFiber = yield* Effect.fork(flushScheduler)
    
    const add = (item: T) =>
      Effect.gen(function* () {
        yield* Ref.update(buffer, buf => Chunk.append(buf, item))
        const currentSize = yield* Ref.get(buffer).pipe(
          Effect.map(Chunk.size)
        )
        
        // Flush if batch is full
        if (currentSize >= batchSize) {
          yield* flush
        }
      })
    
    const done = Effect.gen(function* () {
      yield* flush // Final flush
      yield* input.done()
      yield* Fiber.interrupt(flushFiber)
    })
    
    return { add, done, input }
  })

const batchingExample = Effect.gen(function* () {
  const { add, done, input } = yield* createBatchingProducer<string>(
    3, // Batch size of 3
    Duration.seconds(1) // Flush every second
  )
  
  // Consumer that processes batches
  const batchConsumer = Effect.gen(function* () {
    const stream = Stream.fromChannel(Channel.fromInput(input))
    
    return yield* stream.pipe(
      Stream.tap(batch => 
        Effect.log(`🏗️ Processing batch of ${Chunk.size(batch)} items: ${Chunk.toReadonlyArray(batch).join(", ")}`)
      ),
      Stream.runDrain
    )
  })
  
  const consumerFiber = yield* Effect.fork(batchConsumer)
  
  // Add items with various timing
  yield* add("item-1")
  yield* add("item-2")
  yield* add("item-3") // This should trigger a batch flush
  
  yield* Effect.sleep("500 millis")
  
  yield* add("item-4")
  yield* add("item-5")
  
  yield* Effect.sleep("1.2 seconds") // This should trigger time-based flush
  
  yield* add("item-6")
  
  yield* done // Final flush
  yield* Fiber.join(consumerFiber)
})
```

### Pattern 3: Priority-Based Processing

```typescript
import { 
  SingleProducerAsyncInput, 
  Effect, 
  Stream,
  Channel,
  Data,
  Order,
  Array as Arr,
  Ref
} from "effect"

class PriorityItem<T> extends Data.Class<{
  priority: number // Higher number = higher priority
  data: T
  timestamp: number
}> {}

const priorityOrder = <T>(): Order.Order<PriorityItem<T>> =>
  Order.make((a, b) => {
    // First compare by priority (higher priority first)
    if (a.priority !== b.priority) {
      return b.priority - a.priority
    }
    // Then by timestamp (older first for same priority)
    return a.timestamp - b.timestamp
  })

const createPriorityProducer = <T>() =>
  Effect.gen(function* () {
    const input = yield* SingleProducerAsyncInput.make<never, PriorityItem<T>, void>()
    const priorityBuffer = yield* Ref.make<ReadonlyArray<PriorityItem<T>>>([])
    
    const emit = (data: T, priority = 0) =>
      Effect.gen(function* () {
        const item = new PriorityItem({
          priority,
          data,
          timestamp: Date.now()
        })
        
        // Add to priority buffer and sort
        yield* Ref.update(priorityBuffer, buffer => 
          Arr.sort([...buffer, item], priorityOrder())
        )
        
        yield* Effect.log(`📥 Queued item with priority ${priority}: ${JSON.stringify(data)}`)
      })
    
    const startProcessing = Effect.gen(function* () {
      while (true) {
        const buffer = yield* Ref.get(priorityBuffer)
        
        if (buffer.length > 0) {
          const [nextItem, ...remaining] = buffer
          yield* Ref.set(priorityBuffer, remaining)
          yield* input.emit(nextItem)
        } else {
          // Wait a bit if buffer is empty
          yield* Effect.sleep("100 millis")
        }
      }
    })
    
    const done = Effect.gen(function* () {
      // Process remaining items
      const buffer = yield* Ref.get(priorityBuffer)
      for (const item of buffer) {
        yield* input.emit(item)
      }
      yield* input.done()
    })
    
    return { emit, startProcessing, done, input }
  })

const priorityProcessingExample = Effect.gen(function* () {
  const { emit, startProcessing, done, input } = yield* createPriorityProducer<string>()
  
  // Consumer
  const consumer = Effect.gen(function* () {
    const stream = Stream.fromChannel(Channel.fromInput(input))
    
    return yield* stream.pipe(
      Stream.tap(item => 
        Effect.log(`⚡ Processing priority ${item.priority} item: ${item.data}`)
      ),
      Stream.runDrain
    )
  })
  
  const consumerFiber = yield* Effect.fork(consumer)
  const processingFiber = yield* Effect.fork(startProcessing)
  
  // Add items with different priorities (they should be processed in priority order)
  yield* emit("low-priority-task", 1)
  yield* emit("normal-task", 5)
  yield* emit("urgent-task", 10)
  yield* emit("another-low-task", 2)
  yield* emit("critical-task", 15)
  yield* emit("medium-task", 7)
  
  yield* Effect.sleep("500 millis") // Let some processing happen
  
  yield* Fiber.interrupt(processingFiber)
  yield* done
  yield* Fiber.join(consumerFiber)
})
```

## Integration Examples

### Integration with HTTP Servers

```typescript
import { 
  SingleProducerAsyncInput, 
  Effect, 
  Stream,
  Channel,
  Data,
  Context,
  Layer
} from "effect"

// Simulate HTTP request/response types
class HttpRequest extends Data.Class<{
  id: string
  method: string
  path: string
  body: unknown
}> {}

class HttpResponse extends Data.Class<{
  requestId: string
  status: number
  headers: Record<string, string>
  body: unknown
}> {}

// HTTP Handler service
class HttpHandler extends Context.Tag("HttpHandler")<
  HttpHandler,
  {
    handleRequest: (request: HttpRequest) => Effect.Effect<HttpResponse>
  }
>() {}

const makeHttpHandler = Effect.succeed(
  HttpHandler.of({
    handleRequest: (request) =>
      Effect.gen(function* () {
        yield* Effect.sleep("50 millis") // Simulate processing
        
        return new HttpResponse({
          requestId: request.id,
          status: 200,
          headers: { "Content-Type": "application/json" },
          body: { message: `Handled ${request.method} ${request.path}` }
        })
      })
  })
)

// HTTP Server using SingleProducerAsyncInput
const createHttpServer = Effect.gen(function* () {
  const handler = yield* HttpHandler
  
  const requestInput = yield* SingleProducerAsyncInput.make<never, HttpRequest, void>()
  const responseMap = yield* Ref.make<Map<string, HttpResponse>>(new Map())
  
  // Request processing pipeline
  const requestProcessor = Effect.gen(function* () {
    const requestStream = Stream.fromChannel(Channel.fromInput(requestInput))
    
    return yield* requestStream.pipe(
      Stream.mapEffect(request =>
        Effect.gen(function* () {
          const response = yield* handler.handleRequest(request)
          
          // Store response for retrieval
          yield* Ref.update(responseMap, map => 
            new Map(map).set(request.id, response)
          )
          
          yield* Effect.log(`📨 Processed request ${request.id}`)
          return response
        })
      ),
      Stream.runDrain
    )
  })
  
  const processorFiber = yield* Effect.fork(requestProcessor)
  
  // Server API
  const handleRequest = (request: HttpRequest) =>
    Effect.gen(function* () {
      // Submit request for processing (with backpressure)
      yield* requestInput.emit(request)
      
      // Poll for response (in real implementation, this would be event-driven)
      const response = yield* Effect.repeat(
        Effect.gen(function* () {
          const responses = yield* Ref.get(responseMap)
          const response = responses.get(request.id)
          
          if (response) {
            // Cleanup
            yield* Ref.update(responseMap, map => {
              const newMap = new Map(map)
              newMap.delete(request.id)
              return newMap
            })
            return response
          }
          
          return yield* Effect.fail("Response not ready")
        }),
        Schedule.spaced(Duration.millis(10)).pipe(
          Schedule.whileInput((input: string) => input === "Response not ready"),
          Schedule.compose(Schedule.recurs(100)) // Max 100 attempts
        )
      )
      
      return response
    })
  
  const shutdown = Effect.gen(function* () {
    yield* requestInput.done()
    yield* Fiber.join(processorFiber)
  })
  
  return { handleRequest, shutdown }
})

const httpServerExample = Effect.gen(function* () {
  const server = yield* createHttpServer
  
  // Simulate concurrent HTTP requests
  const requests = [
    new HttpRequest({ id: "req-1", method: "GET", path: "/users", body: null }),
    new HttpRequest({ id: "req-2", method: "POST", path: "/users", body: { name: "John" } }),
    new HttpRequest({ id: "req-3", method: "GET", path: "/posts", body: null })
  ]
  
  // Handle requests concurrently
  const responseFibers = yield* Effect.forEach(
    requests,
    request => Effect.fork(server.handleRequest(request)),
    { concurrency: "unbounded" }
  )
  
  const responses = yield* Effect.forEach(
    responseFibers,
    Fiber.join
  )
  
  yield* Effect.forEach(
    responses,
    response => Effect.log(`📥 Response ${response.requestId}: ${response.status}`)
  )
  
  yield* server.shutdown
}).pipe(
  Effect.provide(Layer.succeed(HttpHandler, makeHttpHandler))
)
```

### Integration with Testing

```typescript
import { 
  SingleProducerAsyncInput, 
  Effect, 
  TestServices,
  Ref,
  Array as Arr
} from "effect"

// Test utilities for SingleProducerAsyncInput
const createTestInput = <E, A, D>() =>
  Effect.gen(function* () {
    const input = yield* SingleProducerAsyncInput.make<E, A, D>()
    const emittedValues = yield* Ref.make<A[]>([])
    const errors = yield* Ref.make<E[]>([])
    const isCompleted = yield* Ref.make(false)
    
    // Consumer that captures all values for testing
    const testConsumer = Effect.gen(function* () {
      while (true) {
        const result = yield* input.takeWith(
          (cause) => Effect.gen(function* () {
            const failure = Cause.failureOption(cause)
            if (failure._tag === "Some") {
              yield* Ref.update(errors, errs => [...errs, failure.value])
            }
          }),
          (value) => Ref.update(emittedValues, values => [...values, value]),
          (_done) => Ref.set(isCompleted, true)
        )
        yield* result
        
        const completed = yield* Ref.get(isCompleted)
        if (completed) break
      }
    })
    
    const consumerFiber = yield* Effect.fork(testConsumer)
    
    // Test assertions
    const getEmittedValues = Ref.get(emittedValues)
    const getErrors = Ref.get(errors)
    const getIsCompleted = Ref.get(isCompleted)
    
    const waitForValue = (expectedValue: A) =>
      Effect.repeat(
        Effect.gen(function* () {
          const values = yield* getEmittedValues
          if (values.includes(expectedValue)) {
            return expectedValue
          }
          return yield* Effect.fail("Value not found")
        }),
        Schedule.spaced(Duration.millis(10)).pipe(
          Schedule.whileInput((input: string) => input === "Value not found"),
          Schedule.compose(Schedule.recurs(50))
        )
      )
    
    const waitForCompletion = Effect.repeat(
      Effect.gen(function* () {
        const completed = yield* getIsCompleted
        if (completed) return true
        return yield* Effect.fail("Not completed")
      }),
      Schedule.spaced(Duration.millis(10)).pipe(
        Schedule.whileInput((input: string) => input === "Not completed"),
        Schedule.compose(Schedule.recurs(100))
      )
    )
    
    const cleanup = Fiber.interrupt(consumerFiber)
    
    return {
      input,
      getEmittedValues,
      getErrors,
      getIsCompleted,
      waitForValue,
      waitForCompletion,
      cleanup
    }
  })

// Example test suite
const singleProducerAsyncInputTests = Effect.gen(function* () {
  yield* Effect.log("🧪 Running SingleProducerAsyncInput tests...")
  
  // Test 1: Basic emit and consume
  yield* Effect.log("Test 1: Basic emit and consume")
  const test1 = yield* createTestInput<never, string, void>()
  
  yield* test1.input.emit("hello")
  yield* test1.input.emit("world")
  yield* test1.input.done()
  
  yield* test1.waitForCompletion
  const values1 = yield* test1.getEmittedValues
  
  yield* Effect.log(`✅ Test 1 passed: ${JSON.stringify(values1)}`)
  yield* test1.cleanup
  
  // Test 2: Error handling
  yield* Effect.log("Test 2: Error handling")
  const test2 = yield* createTestInput<string, number, void>()
  
  yield* test2.input.emit(1)
  yield* test2.input.error(Cause.fail("test error"))
  
  yield* Effect.sleep("100 millis") // Give time for error processing
  
  const values2 = yield* test2.getEmittedValues
  const errors2 = yield* test2.getErrors
  
  yield* Effect.log(`✅ Test 2 passed: values=${JSON.stringify(values2)}, errors=${JSON.stringify(errors2)}`)
  yield* test2.cleanup
  
  // Test 3: Backpressure
  yield* Effect.log("Test 3: Backpressure simulation")
  const test3 = yield* createTestInput<never, string, void>()
  
  // Emit values rapidly in background
  const emitFiber = yield* Effect.fork(
    Effect.gen(function* () {
      for (let i = 1; i <= 5; i++) {
        yield* test3.input.emit(`item-${i}`)
        yield* Effect.log(`📤 Emitted item-${i}`)
      }
      yield* test3.input.done()
    })
  )
  
  // Values should be processed sequentially due to backpressure
  yield* test3.waitForCompletion
  yield* Fiber.join(emitFiber)
  
  const values3 = yield* test3.getEmittedValues
  yield* Effect.log(`✅ Test 3 passed: all items processed in order: ${JSON.stringify(values3)}`)
  yield* test3.cleanup
  
  yield* Effect.log("🎉 All tests completed!")
}).pipe(
  Effect.provide(TestServices.TestServices)
)
```

## Conclusion

SingleProducerAsyncInput provides **coordinated async communication**, **natural backpressure**, and **clean error handling** for single-producer, multiple-consumer scenarios.

Key benefits:
- **Backpressure Management**: Producers automatically wait for consumers, preventing memory overflow
- **Work Distribution**: Each value is consumed by exactly one consumer (not broadcasted)
- **Resource Efficiency**: Buffer size of 1 prevents unbounded memory growth  
- **Error Propagation**: Sophisticated error handling with proper cause propagation
- **Integration Ready**: Works seamlessly with Channel, Stream, and other Effect primitives

SingleProducerAsyncInput is ideal for scenarios where you need coordinated async processing with built-in flow control, such as request processing pipelines, event handling systems, and resource management workflows.