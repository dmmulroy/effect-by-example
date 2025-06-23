# UpstreamPullRequest: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem UpstreamPullRequest Solves

When building complex streaming applications, you often need to coordinate data flow between upstream producers and downstream consumers. Traditional approaches to flow control and backpressure management are fragmented and error-prone:

```typescript
// Traditional approach - manual flow control
class StreamProcessor {
  private buffer: any[] = []
  private downstreamConnections = 0
  
  async processUpstream(data: any) {
    // Manual tracking of downstream state
    if (this.downstreamConnections === 0) {
      // What do we do? Drop? Buffer? Error?
      console.warn("No downstream consumers")
      return
    }
    
    // Manual buffer management
    this.buffer.push(data)
    
    // Complex coordination logic
    if (this.shouldPullMore()) {
      await this.requestMore()
    }
  }
  
  private shouldPullMore(): boolean {
    // Complex heuristics that are hard to get right
    return this.buffer.length < 10 && this.downstreamConnections > 0
  }
}
```

This approach leads to:
- **Coordination Complexity** - Manual tracking of upstream/downstream state
- **Error-Prone Logic** - Hard to determine when to pull more data
- **Resource Leaks** - Difficult to manage buffers and connections properly
- **Non-Composable** - Each component needs its own flow control logic

### The UpstreamPullRequest Solution

UpstreamPullRequest provides a type-safe, composable way to represent and handle upstream data requests in Channel-based streaming:

```typescript
import { UpstreamPullRequest, UpstreamPullStrategy } from "effect"

// Type-safe upstream request handling
const handleUpstreamRequest = (
  request: UpstreamPullRequest.UpstreamPullRequest<string>
): UpstreamPullStrategy.UpstreamPullStrategy<string> => {
  return UpstreamPullRequest.match(request, {
    onPulled: (value) => {
      // Data was successfully pulled from upstream
      return UpstreamPullStrategy.PullAfterNext(Option.none())
    },
    onNoUpstream: (activeDownstreamCount) => {
      // No upstream available, decide strategy based on downstream count
      return activeDownstreamCount > 0
        ? UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())
        : UpstreamPullStrategy.PullAfterNext(Option.none())
    }
  })
}
```

### Key Concepts

**UpstreamPullRequest**: A discriminated union representing the result of an upstream pull operation - either data was received (`Pulled`) or no upstream source is available (`NoUpstream`).

**Pulled**: Indicates successful data retrieval from upstream with the actual value.

**NoUpstream**: Indicates no upstream source is available, but provides the count of active downstream consumers to inform decision-making.

## Basic Usage Patterns

### Pattern 1: Creating Pull Requests

```typescript
import { UpstreamPullRequest } from "effect"

// Create a successful pull request with data
const successfulPull = UpstreamPullRequest.Pulled("some data")

// Create a no-upstream request with downstream count
const noUpstreamPull = UpstreamPullRequest.NoUpstream(3) // 3 active downstream consumers
```

### Pattern 2: Type-Safe Request Handling

```typescript
import { UpstreamPullRequest, UpstreamPullStrategy, Option } from "effect"

const processRequest = <A>(
  request: UpstreamPullRequest.UpstreamPullRequest<A>
): UpstreamPullStrategy.UpstreamPullStrategy<A> => {
  return UpstreamPullRequest.match(request, {
    onPulled: (value: A) => {
      console.log("Received data:", value)
      return UpstreamPullStrategy.PullAfterNext(Option.none())
    },
    onNoUpstream: (activeDownstreamCount: number) => {
      console.log(`No upstream, ${activeDownstreamCount} downstream consumers`)
      return UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())
    }
  })
}
```

### Pattern 3: Request Inspection

```typescript
import { UpstreamPullRequest } from "effect"

const inspectRequest = <A>(request: UpstreamPullRequest.UpstreamPullRequest<A>): void => {
  if (UpstreamPullRequest.isPulled(request)) {
    console.log("Got data:", request.value)
  } else if (UpstreamPullRequest.isNoUpstream(request)) {
    console.log("No upstream, active downstream:", request.activeDownstreamCount)
  }
}
```

## Real-World Examples

### Example 1: HTTP Stream Processing

```typescript
import { Effect, UpstreamPullRequest, UpstreamPullStrategy, Option, Channel } from "effect"

interface HttpChunk {
  readonly data: Uint8Array
  readonly timestamp: number
}

interface HttpStreamProcessor {
  readonly processChunk: (chunk: HttpChunk) => Effect.Effect<void>
  readonly getBackpressureStatus: () => Effect.Effect<boolean>
}

const createHttpStreamProcessor = (): Effect.Effect<HttpStreamProcessor> => {
  return Effect.gen(function* () {
    const backpressureRef = yield* Effect.ref(false)
    
    const processChunk = (chunk: HttpChunk): Effect.Effect<void> => {
      return Effect.gen(function* () {
        const isBackpressured = yield* backpressureRef.get
        
        if (isBackpressured) {
          yield* Effect.logInfo(`Buffering chunk: ${chunk.data.length} bytes`)
        } else {
          yield* Effect.logInfo(`Processing chunk: ${chunk.data.length} bytes`)
        }
        
        // Simulate processing time
        yield* Effect.sleep("10 millis")
      })
    }
    
    const getBackpressureStatus = (): Effect.Effect<boolean> => backpressureRef.get
    
    return { processChunk, getBackpressureStatus }
  })
}

const handleHttpUpstreamRequest = (
  processor: HttpStreamProcessor
) => (
  request: UpstreamPullRequest.UpstreamPullRequest<HttpChunk>
): Effect.Effect<UpstreamPullStrategy.UpstreamPullStrategy<HttpChunk>> => {
  return Effect.gen(function* () {
    const backpressured = yield* processor.getBackpressureStatus()
    
    return UpstreamPullRequest.match(request, {
      onPulled: (chunk) => {
        // Process the chunk
        Effect.runFork(processor.processChunk(chunk))
        
        // Decide pull strategy based on backpressure
        return backpressured
          ? UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())
          : UpstreamPullStrategy.PullAfterNext(Option.none())
      },
      onNoUpstream: (activeDownstreamCount) => {
        // No more HTTP data available
        if (activeDownstreamCount > 0) {
          // Still have consumers, wait for them to finish
          return UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())
        } else {
          // No consumers, can pull immediately when available
          return UpstreamPullStrategy.PullAfterNext(Option.none())
        }
      }
    })
  })
}
```

### Example 2: Database Streaming with Batch Processing

```typescript
import { Effect, UpstreamPullRequest, UpstreamPullStrategy, Option, Chunk, Array as Arr } from "effect"

interface DatabaseRecord {
  readonly id: string
  readonly data: Record<string, unknown>
  readonly timestamp: Date
}

interface BatchProcessor {
  readonly batchSize: number
  readonly currentBatch: Array<DatabaseRecord>
  readonly processBatch: (records: Array<DatabaseRecord>) => Effect.Effect<void>
}

const createBatchProcessor = (batchSize: number): Effect.Effect<BatchProcessor> => {
  return Effect.gen(function* () {
    const batchRef = yield* Effect.ref<Array<DatabaseRecord>>([])
    
    const processBatch = (records: Array<DatabaseRecord>): Effect.Effect<void> => {
      return Effect.gen(function* () {
        yield* Effect.logInfo(`Processing batch of ${records.length} records`)
        
        // Simulate batch processing
        yield* Effect.forEach(records, (record) => 
          Effect.gen(function* () {
            yield* Effect.logDebug(`Processing record: ${record.id}`)
            yield* Effect.sleep("5 millis")
          })
        )
        
        yield* Effect.logInfo("Batch processing completed")
      })
    }
    
    const getCurrentBatch = (): Effect.Effect<Array<DatabaseRecord>> => batchRef.get
    
    const addToBatch = (record: DatabaseRecord): Effect.Effect<boolean> => {
      return Effect.gen(function* () {
        const current = yield* batchRef.get
        const updated = [...current, record]
        
        if (updated.length >= batchSize) {
          yield* batchRef.set([])
          yield* processBatch(updated)
          return true // Batch was processed
        } else {
          yield* batchRef.set(updated)
          return false // Still accumulating
        }
      })
    }
    
    return {
      batchSize,
      get currentBatch() {
        return Effect.runSync(getCurrentBatch())
      },
      processBatch: addToBatch
    }
  })
}

const handleDatabaseUpstreamRequest = (
  processor: BatchProcessor
) => (
  request: UpstreamPullRequest.UpstreamPullRequest<DatabaseRecord>
): Effect.Effect<UpstreamPullStrategy.UpstreamPullStrategy<DatabaseRecord>> => {
  return Effect.gen(function* () {
    return UpstreamPullRequest.match(request, {
      onPulled: (record) => {
        return Effect.gen(function* () {
          const batchCompleted = yield* processor.processBatch(record)
          
          // If batch was completed, pull more aggressively
          // Otherwise, wait for batch to fill up
          return batchCompleted
            ? UpstreamPullStrategy.PullAfterNext(Option.none())
            : UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())
        })
      },
      onNoUpstream: (activeDownstreamCount) => {
        // No more database records available
        return Effect.gen(function* () {
          const currentBatch = processor.currentBatch
          
          if (currentBatch.length > 0) {
            // Process remaining records in partial batch
            yield* processor.processBatch(currentBatch[0]) // Trigger batch processing
          }
          
          // Strategy depends on whether we still have downstream consumers
          return activeDownstreamCount > 0
            ? UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())
            : UpstreamPullStrategy.PullAfterNext(Option.none())
        })
      }
    })
  })
}
```

### Example 3: WebSocket Message Coordination

```typescript
import { Effect, UpstreamPullRequest, UpstreamPullStrategy, Option, Queue } from "effect"

interface WebSocketMessage {
  readonly type: "data" | "ping" | "pong" | "close"
  readonly payload: string
  readonly timestamp: number
}

interface WebSocketHandler {
  readonly messageQueue: Queue.Queue<WebSocketMessage>
  readonly activeConnections: number
  readonly sendMessage: (message: WebSocketMessage) => Effect.Effect<void>
}

const createWebSocketHandler = (): Effect.Effect<WebSocketHandler> => {
  return Effect.gen(function* () {
    const messageQueue = yield* Queue.bounded<WebSocketMessage>(1000)
    const connectionCount = yield* Effect.ref(0)
    
    const sendMessage = (message: WebSocketMessage): Effect.Effect<void> => {
      return Effect.gen(function* () {
        const connections = yield* connectionCount.get
        
        if (connections === 0) {
          yield* Effect.logWarning("No active connections to send message")
          return
        }
        
        yield* Queue.offer(messageQueue, message)
        yield* Effect.logInfo(`Queued message type: ${message.type}`)
      })
    }
    
    const getActiveConnections = (): Effect.Effect<number> => connectionCount.get
    
    return {
      messageQueue,
      get activeConnections() {
        return Effect.runSync(getActiveConnections())
      },
      sendMessage
    }
  })
}

const handleWebSocketUpstreamRequest = (
  handler: WebSocketHandler
) => (
  request: UpstreamPullRequest.UpstreamPullRequest<WebSocketMessage>
): UpstreamPullStrategy.UpstreamPullStrategy<WebSocketMessage> => {
  return UpstreamPullRequest.match(request, {
    onPulled: (message) => {
      // Successfully received a message from upstream
      Effect.runFork(handler.sendMessage(message))
      
      // Determine pull strategy based on message type
      return message.type === "close"
        ? UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())
        : UpstreamPullStrategy.PullAfterNext(Option.none())
    },
    onNoUpstream: (activeDownstreamCount) => {
      // No upstream WebSocket messages available
      const hasActiveConnections = handler.activeConnections > 0
      
      if (hasActiveConnections && activeDownstreamCount > 0) {
        // Still have both upstream and downstream connections
        return UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())
      } else {
        // Either no connections or no downstream consumers
        return UpstreamPullStrategy.PullAfterNext(Option.none())
      }
    }
  })
}
```

## Advanced Features Deep Dive

### Feature 1: Conditional Pull Strategies

Understanding when to use different pull strategies based on upstream request state:

#### Basic Pull Strategy Selection

```typescript
import { UpstreamPullRequest, UpstreamPullStrategy, Option } from "effect"

const basicPullStrategy = <A>(
  request: UpstreamPullRequest.UpstreamPullRequest<A>
): UpstreamPullStrategy.UpstreamPullStrategy<A> => {
  return UpstreamPullRequest.match(request, {
    onPulled: (_value) => {
      // Data available - pull after processing current item
      return UpstreamPullStrategy.PullAfterNext(Option.none())
    },
    onNoUpstream: (_activeDownstreamCount) => {
      // No data available - wait for all queued items to be processed
      return UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())
    }
  })
}
```

#### Advanced Conditional Strategy

```typescript
import { UpstreamPullRequest, UpstreamPullStrategy, Option, Effect } from "effect"

interface StreamMetrics {
  readonly bufferSize: number
  readonly processingRate: number
  readonly backpressureThreshold: number
}

const adaptivePullStrategy = (metrics: StreamMetrics) => <A>(
  request: UpstreamPullRequest.UpstreamPullRequest<A>
): UpstreamPullStrategy.UpstreamPullStrategy<A> => {
  return UpstreamPullRequest.match(request, {
    onPulled: (value) => {
      // Adaptive strategy based on current metrics
      const isBackpressured = metrics.bufferSize > metrics.backpressureThreshold
      const hasHighProcessingRate = metrics.processingRate > 100 // messages/sec
      
      if (isBackpressured) {
        // High buffer usage - wait for queue to drain
        return UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())
      } else if (hasHighProcessingRate) {
        // High processing rate - pull aggressively
        return UpstreamPullStrategy.PullAfterNext(Option.none())
      } else {
        // Balanced approach
        return UpstreamPullStrategy.PullAfterNext(Option.none())
      }
    },
    onNoUpstream: (activeDownstreamCount) => {
      // Strategy based on downstream demand
      return activeDownstreamCount > metrics.backpressureThreshold
        ? UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())
        : UpstreamPullStrategy.PullAfterNext(Option.none())
    }
  })
}
```

### Feature 2: Request State Management

Managing complex upstream request state across multiple processing stages:

```typescript
import { UpstreamPullRequest, Effect, Ref } from "effect"

interface RequestTracker<A> {
  readonly totalRequests: number
  readonly successfulPulls: number
  readonly noUpstreamCount: number
  readonly lastValue: A | null
}

const createRequestTracker = <A>(): Effect.Effect<Ref.Ref<RequestTracker<A>>> => {
  return Effect.ref<RequestTracker<A>>({
    totalRequests: 0,
    successfulPulls: 0,
    noUpstreamCount: 0,
    lastValue: null
  })
}

const trackRequest = <A>(
  tracker: Ref.Ref<RequestTracker<A>>
) => (
  request: UpstreamPullRequest.UpstreamPullRequest<A>
): Effect.Effect<void> => {
  return Effect.gen(function* () {
    const current = yield* tracker.get
    
    const updated = UpstreamPullRequest.match(request, {
      onPulled: (value): RequestTracker<A> => ({
        ...current,
        totalRequests: current.totalRequests + 1,
        successfulPulls: current.successfulPulls + 1,
        lastValue: value
      }),
      onNoUpstream: (_activeDownstreamCount): RequestTracker<A> => ({
        ...current,
        totalRequests: current.totalRequests + 1,
        noUpstreamCount: current.noUpstreamCount + 1
      })
    })
    
    yield* tracker.set(updated)
  })
}

const getRequestStats = <A>(
  tracker: Ref.Ref<RequestTracker<A>>
): Effect.Effect<{
  successRate: number
  upstreamAvailability: number
  hasRecentData: boolean
}> => {
  return Effect.gen(function* () {
    const stats = yield* tracker.get
    
    const successRate = stats.totalRequests > 0 
      ? stats.successfulPulls / stats.totalRequests 
      : 0
      
    const upstreamAvailability = stats.totalRequests > 0
      ? (stats.totalRequests - stats.noUpstreamCount) / stats.totalRequests
      : 0
      
    const hasRecentData = stats.lastValue !== null
    
    return { successRate, upstreamAvailability, hasRecentData }
  })
}
```

## Practical Patterns & Best Practices

### Pattern 1: Request Multiplexing

```typescript
import { UpstreamPullRequest, UpstreamPullStrategy, Option, Effect, Array as Arr } from "effect"

interface MultiplexedRequest<A> {
  readonly requests: ReadonlyArray<UpstreamPullRequest.UpstreamPullRequest<A>>
  readonly strategy: "first-wins" | "all-or-nothing" | "best-effort"
}

const handleMultiplexedRequests = <A>(
  multiplexed: MultiplexedRequest<A>
): UpstreamPullStrategy.UpstreamPullStrategy<A> => {
  const { requests, strategy } = multiplexed
  
  switch (strategy) {
    case "first-wins": {
      // Use the first successful pull, ignore the rest
      const firstPulled = requests.find(UpstreamPullRequest.isPulled)
      if (firstPulled) {
        return UpstreamPullStrategy.PullAfterNext(Option.none())
      }
      
      // All are NoUpstream - sum downstream counts
      const totalDownstream = requests
        .filter(UpstreamPullRequest.isNoUpstream)
        .reduce((sum, req) => sum + req.activeDownstreamCount, 0)
        
      return totalDownstream > 0
        ? UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())
        : UpstreamPullStrategy.PullAfterNext(Option.none())
    }
    
    case "all-or-nothing": {
      // Only proceed if all requests have data
      const allPulled = requests.every(UpstreamPullRequest.isPulled)
      if (allPulled) {
        return UpstreamPullStrategy.PullAfterNext(Option.none())
      }
      
      // Wait for all upstream sources to be ready
      return UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())
    }
    
    case "best-effort": {
      // Process any available data
      const hasSomeData = requests.some(UpstreamPullRequest.isPulled)
      return hasSomeData
        ? UpstreamPullStrategy.PullAfterNext(Option.none())
        : UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())
    }
  }
}
```

### Pattern 2: Error Recovery and Retry

```typescript
import { UpstreamPullRequest, UpstreamPullStrategy, Option, Effect, Schedule } from "effect"

interface RetryableRequest<A> {
  readonly request: UpstreamPullRequest.UpstreamPullRequest<A>
  readonly retryCount: number
  readonly maxRetries: number
  readonly lastError?: Error
}

const createRetryableRequestHandler = <A>(maxRetries: number = 3) => {
  return (request: UpstreamPullRequest.UpstreamPullRequest<A>): Effect.Effect<UpstreamPullStrategy.UpstreamPullStrategy<A>> => {
    return Effect.gen(function* () {
      const retryableRequest: RetryableRequest<A> = {
        request,
        retryCount: 0,
        maxRetries
      }
      
      return yield* handleRetryableRequest(retryableRequest)
    })
  }
}

const handleRetryableRequest = <A>(
  retryable: RetryableRequest<A>
): Effect.Effect<UpstreamPullStrategy.UpstreamPullStrategy<A>> => {
  return Effect.gen(function* () {
    return UpstreamPullRequest.match(retryable.request, {
      onPulled: (value) => {
        // Successful pull - reset retry count and continue
        return UpstreamPullStrategy.PullAfterNext(Option.none())
      },
      onNoUpstream: (activeDownstreamCount) => {
        // Check if we should retry
        if (retryable.retryCount < retryable.maxRetries && activeDownstreamCount > 0) {
          // Retry with backoff
          const backoffStrategy = UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())
          
          yield* Effect.logInfo(`Retry attempt ${retryable.retryCount + 1}/${retryable.maxRetries}`)
          yield* Effect.sleep(`${Math.pow(2, retryable.retryCount)} seconds`)
          
          return backoffStrategy
        } else {
          // Max retries reached or no downstream consumers
          yield* Effect.logWarning(`Max retries reached or no downstream consumers`)
          return UpstreamPullStrategy.PullAfterNext(Option.none())
        }
      }
    })
  })
}
```

### Pattern 3: Priority-Based Request Handling

```typescript
import { UpstreamPullRequest, UpstreamPullStrategy, Option, Effect } from "effect"

interface PriorityRequest<A> {
  readonly request: UpstreamPullRequest.UpstreamPullRequest<A>
  readonly priority: "high" | "medium" | "low"
  readonly timestamp: number
}

const createPriorityRequestHandler = <A>() => {
  return (priorityRequests: ReadonlyArray<PriorityRequest<A>>): UpstreamPullStrategy.UpstreamPullStrategy<A> => {
    // Sort by priority (high -> medium -> low) then by timestamp (oldest first)
    const sortedRequests = [...priorityRequests].sort((a, b) => {
      const priorityOrder = { high: 3, medium: 2, low: 1 }
      const priorityDiff = priorityOrder[b.priority] - priorityOrder[a.priority]
      
      if (priorityDiff !== 0) return priorityDiff
      return a.timestamp - b.timestamp
    })
    
    // Process the highest priority request first
    const highestPriorityRequest = sortedRequests[0]
    
    if (!highestPriorityRequest) {
      return UpstreamPullStrategy.PullAfterNext(Option.none())
    }
    
    return UpstreamPullRequest.match(highestPriorityRequest.request, {
      onPulled: (value) => {
        // High priority data available - process immediately
        return highestPriorityRequest.priority === "high"
          ? UpstreamPullStrategy.PullAfterNext(Option.none())
          : UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())
      },
      onNoUpstream: (activeDownstreamCount) => {
        // Check if we have other priority requests available
        const hasOtherRequests = sortedRequests.length > 1
        const hasHighPriorityDownstream = activeDownstreamCount > 0 && 
          sortedRequests.some(req => req.priority === "high")
        
        if (hasOtherRequests || hasHighPriorityDownstream) {
          return UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())
        } else {
          return UpstreamPullStrategy.PullAfterNext(Option.none())
        }
      }
    })
  }
}
```

## Integration Examples

### Integration with Channel and Stream

```typescript
import { Channel, Stream, UpstreamPullRequest, UpstreamPullStrategy, Option, Effect } from "effect"

// Custom channel that uses UpstreamPullRequest for flow control
const createBackpressureAwareChannel = <A>(
  bufferSize: number
): Channel.Channel<A, A, never, never, void, never, never> => {
  return Channel.suspend(() => {
    let currentBuffer: A[] = []
    
    const pullHandler = (
      request: UpstreamPullRequest.UpstreamPullRequest<A>
    ): UpstreamPullStrategy.UpstreamPullStrategy<A> => {
      return UpstreamPullRequest.match(request, {
        onPulled: (value) => {
          currentBuffer.push(value)
          
          // Emit buffered values if buffer is full
          if (currentBuffer.length >= bufferSize) {
            const toEmit = currentBuffer.splice(0, bufferSize)
            // Process buffered items
            return UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())
          }
          
          return UpstreamPullStrategy.PullAfterNext(Option.none())
        },
        onNoUpstream: (activeDownstreamCount) => {
          // Flush remaining buffer
          if (currentBuffer.length > 0 && activeDownstreamCount > 0) {
            const toEmit = currentBuffer.splice(0)
            // Process remaining items
          }
          
          return UpstreamPullStrategy.PullAfterNext(Option.none())
        }
      })
    }
    
    // Use the pull handler in channel implementation
    return Channel.identity<A>() // Simplified - real implementation would use pullHandler
  })
}

// Stream integration
const createStreamWithUpstreamControl = <A>(
  source: Stream.Stream<A>
): Stream.Stream<A> => {
  return Stream.fromChannel(
    source.pipe(
      Stream.toChannel,
      Channel.pipeTo(createBackpressureAwareChannel(10))
    )
  )
}

// Usage example
const controlledStream = createStreamWithUpstreamControl(
  Stream.range(1, 100).pipe(
    Stream.map(n => `Item ${n}`)
  )
)
```

### Integration with Effect Error Handling

```typescript
import { Effect, UpstreamPullRequest, UpstreamPullStrategy, Option } from "effect"

// Error types for upstream processing
class UpstreamError extends Error {
  readonly _tag = "UpstreamError"
  constructor(message: string, readonly cause?: Error) {
    super(message)
  }
}

class NoUpstreamError extends Error {
  readonly _tag = "NoUpstreamError"
  constructor(readonly activeDownstreamCount: number) {
    super(`No upstream available, ${activeDownstreamCount} downstream consumers`)
  }
}

const processUpstreamRequestWithErrorHandling = <A>(
  request: UpstreamPullRequest.UpstreamPullRequest<A>,
  processor: (value: A) => Effect.Effect<void, UpstreamError>
): Effect.Effect<UpstreamPullStrategy.UpstreamPullStrategy<A>, UpstreamError | NoUpstreamError> => {
  return Effect.gen(function* () {
    return UpstreamPullRequest.match(request, {
      onPulled: (value) => {
        return Effect.gen(function* () {
          // Process the value with error handling
          yield* processor(value).pipe(
            Effect.catchAll((error) => 
              Effect.gen(function* () {
                yield* Effect.logError(`Processing failed: ${error.message}`)
                return yield* Effect.fail(error)
              })
            )
          )
          
          return UpstreamPullStrategy.PullAfterNext(Option.none())
        })
      },
      onNoUpstream: (activeDownstreamCount) => {
        // Convert to typed error
        return Effect.fail(new NoUpstreamError(activeDownstreamCount))
      }
    })
  })
}

// Usage with proper error recovery
const robustUpstreamProcessor = <A>(
  request: UpstreamPullRequest.UpstreamPullRequest<A>,
  processor: (value: A) => Effect.Effect<void, UpstreamError>
): Effect.Effect<UpstreamPullStrategy.UpstreamPullStrategy<A>> => {
  return processUpstreamRequestWithErrorHandling(request, processor).pipe(
    Effect.catchTag("UpstreamError", (error) => 
      Effect.gen(function* () {
        yield* Effect.logWarning(`Upstream processing error: ${error.message}`)
        // Return a conservative pull strategy on error
        return UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())
      })
    ),
    Effect.catchTag("NoUpstreamError", (error) =>
      Effect.gen(function* () {
        yield* Effect.logInfo(`No upstream: ${error.message}`)
        // Return appropriate strategy based on downstream count
        return error.activeDownstreamCount > 0
          ? UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())
          : UpstreamPullStrategy.PullAfterNext(Option.none())
      })
    )
  )
}
```

### Testing Strategies

```typescript
import { Effect, UpstreamPullRequest, UpstreamPullStrategy, Option } from "effect"

// Test utilities for UpstreamPullRequest
const createTestRequest = <A>(value: A): UpstreamPullRequest.UpstreamPullRequest<A> => 
  UpstreamPullRequest.Pulled(value)

const createTestNoUpstream = (downstreamCount: number): UpstreamPullRequest.UpstreamPullRequest<never> =>
  UpstreamPullRequest.NoUpstream(downstreamCount)

// Property-based testing helpers
const testUpstreamRequestHandler = <A>(
  handler: (request: UpstreamPullRequest.UpstreamPullRequest<A>) => UpstreamPullStrategy.UpstreamPullStrategy<A>,
  testCases: Array<{
    input: UpstreamPullRequest.UpstreamPullRequest<A>
    expected: UpstreamPullStrategy.UpstreamPullStrategy<A>
  }>
): Effect.Effect<boolean> => {
  return Effect.gen(function* () {
    for (const testCase of testCases) {
      const result = handler(testCase.input)
      
      // Compare strategies (simplified comparison)
      const isEqual = JSON.stringify(result) === JSON.stringify(testCase.expected)
      
      if (!isEqual) {
        yield* Effect.logError(`Test failed for input: ${JSON.stringify(testCase.input)}`)
        return false
      }
    }
    
    yield* Effect.logInfo("All tests passed")
    return true
  })
}

// Example test suite
const testBasicRequestHandler = Effect.gen(function* () {
  const handler = (request: UpstreamPullRequest.UpstreamPullRequest<string>) => {
    return UpstreamPullRequest.match(request, {
      onPulled: (_) => UpstreamPullStrategy.PullAfterNext(Option.none()),
      onNoUpstream: (_) => UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())
    })
  }
  
  const testCases = [
    {
      input: createTestRequest("test"),
      expected: UpstreamPullStrategy.PullAfterNext(Option.none())
    },
    {
      input: createTestNoUpstream(5),
      expected: UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())
    }
  ]
  
  return yield* testUpstreamRequestHandler(handler, testCases)
})
```

## Conclusion

UpstreamPullRequest provides **type-safe upstream coordination**, **composable flow control**, and **predictable backpressure management** for Channel-based streaming applications.

Key benefits:
- **Type Safety**: Eliminates runtime errors in upstream/downstream coordination
- **Composability**: Integrates seamlessly with Effect's Channel and Stream modules
- **Flexibility**: Supports complex flow control patterns and adaptive strategies

UpstreamPullRequest is essential when building robust streaming applications that need fine-grained control over data flow, backpressure handling, and resource coordination between upstream producers and downstream consumers.