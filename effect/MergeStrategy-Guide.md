# MergeStrategy: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem MergeStrategy Solves

When working with concurrent data streams or channels, you inevitably face the challenge of handling backpressure and buffer overflow. Traditional approaches often lead to memory leaks, dropped data, or system instability:

```typescript
// Traditional approach - problematic concurrent stream handling
const traditionalMerge = async (streams: AsyncIterable<Data>[]) => {
  const results: Data[] = []
  const promises = streams.map(async (stream) => {
    for await (const item of stream) {
      results.push(item) // Race conditions, memory issues
    }
  })
  await Promise.all(promises) // No backpressure control
  return results
}
```

This approach leads to:
- **Memory Exhaustion** - Unlimited buffering can cause out-of-memory errors
- **Race Conditions** - Concurrent access to shared state without coordination
- **No Backpressure Control** - Fast producers can overwhelm slow consumers
- **Data Loss** - No strategy for handling overflow situations

### The MergeStrategy Solution

MergeStrategy provides controlled, predictable behavior for merging concurrent streams with built-in backpressure handling:

```typescript
import { MergeStrategy } from "effect"
import { Channel, Stream } from "effect"

// Controlled merge with backpressure
const controlledMerge = <A>(streams: Stream.Stream<A>[]) =>
  Stream.mergeAll(streams, {
    concurrency: 4,
    bufferSize: 32,
    mergeStrategy: MergeStrategy.BackPressure()
  })

// Or with buffer sliding for high-throughput scenarios
const highThroughputMerge = <A>(streams: Stream.Stream<A>[]) =>
  Stream.mergeAll(streams, {
    concurrency: 8,
    bufferSize: 16,
    mergeStrategy: MergeStrategy.BufferSliding()
  })
```

### Key Concepts

**BackPressure**: Slows down producers when consumers cannot keep up, ensuring system stability by preventing buffer overflow.

**BufferSliding**: Drops oldest buffered items when the buffer is full, maintaining throughput at the cost of potentially losing data.

**Merge Operations**: Combining multiple concurrent streams while respecting the chosen backpressure strategy.

## Basic Usage Patterns

### Pattern 1: BackPressure Strategy

```typescript
import { MergeStrategy } from "effect"

// Create a backpressure strategy
const backPressure = MergeStrategy.BackPressure()

// Use with channel operations
const mergeWithBackPressure = <A>(channels: Channel.Channel<A>[]) =>
  Channel.mergeAll(channels, {
    concurrency: 4,
    bufferSize: 16,
    mergeStrategy: backPressure
  })
```

### Pattern 2: BufferSliding Strategy

```typescript
import { MergeStrategy } from "effect"

// Create a buffer sliding strategy
const bufferSliding = MergeStrategy.BufferSliding()

// Use for high-throughput scenarios
const mergeWithSliding = <A>(channels: Channel.Channel<A>[]) =>
  Channel.mergeAll(channels, {
    concurrency: 8,
    bufferSize: 32,
    mergeStrategy: bufferSliding
  })
```

### Pattern 3: Strategy Selection

```typescript
import { MergeStrategy } from "effect"

// Pattern match on strategy types
const getStrategyDescription = (strategy: MergeStrategy.MergeStrategy): string =>
  MergeStrategy.match(strategy, {
    onBackPressure: () => "Applies backpressure to maintain data integrity",
    onBufferSliding: () => "Slides buffer to maintain throughput"
  })

// Type guards for strategy checking
const isBackPressureStrategy = (strategy: MergeStrategy.MergeStrategy): boolean =>
  MergeStrategy.isBackPressure(strategy)

const isBufferSlidingStrategy = (strategy: MergeStrategy.MergeStrategy): boolean =>
  MergeStrategy.isBufferSliding(strategy)
```

## Real-World Examples

### Example 1: Load Balancing Web Requests

Handle multiple API endpoints with controlled concurrency and backpressure:

```typescript
import { Effect, Stream, MergeStrategy, Logger } from "effect"
import { Array as Arr, Function as Fn } from "effect"

interface ApiEndpoint {
  readonly url: string
  readonly priority: number
}

interface ApiResponse {
  readonly endpoint: string
  readonly data: unknown
  readonly timestamp: number
}

const makeApiCall = (endpoint: ApiEndpoint): Effect.Effect<ApiResponse, Error> =>
  Effect.gen(function* () {
    const response = yield* Effect.tryPromise({
      try: () => fetch(endpoint.url).then(res => res.json()),
      catch: (error) => new Error(`API call failed: ${error}`)
    })
    
    return {
      endpoint: endpoint.url,
      data: response,
      timestamp: Date.now()
    }
  }).pipe(
    Effect.withSpan("api.call", { attributes: { "api.url": endpoint.url } })
  )

// Load balancer with backpressure for critical systems
const criticalSystemLoadBalancer = (endpoints: ApiEndpoint[]) => {
  const streams = endpoints.map(endpoint =>
    Stream.fromEffect(makeApiCall(endpoint))
  )

  return Stream.mergeAll(streams, {
    concurrency: 3, // Controlled concurrency for critical systems
    bufferSize: 8,
    mergeStrategy: MergeStrategy.BackPressure() // Ensure no data loss
  }).pipe(
    Stream.tap(response => 
      Logger.info(`Received response from ${response.endpoint}`)
    )
  )
}

// High-throughput load balancer for analytics
const analyticsLoadBalancer = (endpoints: ApiEndpoint[]) => {
  const streams = endpoints.map(endpoint =>
    Stream.fromEffect(makeApiCall(endpoint))
  )

  return Stream.mergeAll(streams, {
    concurrency: 10, // Higher concurrency for throughput
    bufferSize: 64,
    mergeStrategy: MergeStrategy.BufferSliding() // Prefer throughput over completeness
  })
}
```

### Example 2: Real-Time Data Processing Pipeline

Process multiple data streams with different strategies based on data criticality:

```typescript
import { Effect, Stream, MergeStrategy, Queue } from "effect"

interface DataPoint {
  readonly source: string
  readonly value: number
  readonly timestamp: number
  readonly critical: boolean
}

interface ProcessedData {
  readonly source: string
  readonly aggregatedValue: number
  readonly processedAt: number
}

const processDataPoint = (dataPoint: DataPoint): Effect.Effect<ProcessedData> =>
  Effect.gen(function* () {
    // Simulate processing time
    yield* Effect.sleep("100 millis")
    
    return {
      source: dataPoint.source,
      aggregatedValue: dataPoint.value * 1.5,
      processedAt: Date.now()
    }
  })

// Critical data processing with strict ordering and no data loss
const processCriticalData = (criticalStreams: Stream.Stream<DataPoint>[]) =>
  Stream.mergeAll(criticalStreams, {
    concurrency: 2, // Lower concurrency for critical data
    bufferSize: 16,
    mergeStrategy: MergeStrategy.BackPressure() // Never drop critical data
  }).pipe(
    Stream.mapEffect(processDataPoint),
    Stream.tap(processed => 
      Effect.logInfo(`Critical data processed: ${processed.source}`)
    )
  )

// Analytics data processing with high throughput
const processAnalyticsData = (analyticsStreams: Stream.Stream<DataPoint>[]) =>
  Stream.mergeAll(analyticsStreams, {
    concurrency: 8, // Higher concurrency for analytics
    bufferSize: 128,
    mergeStrategy: MergeStrategy.BufferSliding() // OK to drop some analytics data
  }).pipe(
    Stream.mapEffect(processDataPoint),
    Stream.buffer({ capacity: 256 })
  )

// Combined processing pipeline
const dataProcessingPipeline = (
  criticalStreams: Stream.Stream<DataPoint>[],
  analyticsStreams: Stream.Stream<DataPoint>[]
) => Effect.gen(function* () {
  const criticalProcessor = processCriticalData(criticalStreams)
  const analyticsProcessor = processAnalyticsData(analyticsStreams)
  
  // Merge processed streams with different priorities
  const combinedStream = Stream.merge(criticalProcessor, analyticsProcessor)
  
  return yield* Stream.runCollect(combinedStream)
})
```

### Example 3: Financial Trading System

Handle market data feeds with different latency requirements:

```typescript
import { Effect, Stream, MergeStrategy, Ref, Duration } from "effect"

interface MarketData {
  readonly symbol: string
  readonly price: number
  readonly volume: number
  readonly timestamp: number
  readonly exchange: string
}

interface OrderExecution {
  readonly orderId: string
  readonly symbol: string
  readonly executedPrice: number
  readonly executedAt: number
}

// Ultra-low latency trading feeds - never drop data
const tradingDataProcessor = (tradingFeeds: Stream.Stream<MarketData>[]) =>
  Stream.mergeAll(tradingFeeds, {
    concurrency: 4,
    bufferSize: 32,
    mergeStrategy: MergeStrategy.BackPressure() // Critical: never lose trading data
  }).pipe(
    Stream.filter(data => data.volume > 1000), // Only high-volume trades
    Stream.tap(data => 
      Effect.logDebug(`High-volume trade: ${data.symbol} at ${data.price}`)
    )
  )

// Market analytics feeds - can drop old data for latest updates
const analyticsDataProcessor = (analyticsFeeds: Stream.Stream<MarketData>[]) =>
  Stream.mergeAll(analyticsFeeds, {
    concurrency: 12,
    bufferSize: 256,
    mergeStrategy: MergeStrategy.BufferSliding() // OK to drop old analytics data
  }).pipe(
    Stream.groupByKey(data => data.symbol),
    Stream.map(([symbol, symbolStream]) =>
      symbolStream.pipe(
        Stream.debounce(Duration.millis(100)), // Debounce for analytics
        Stream.map(data => ({ symbol, latestPrice: data.price }))
      )
    ),
    Stream.flatten({ concurrency: "unbounded" })
  )
```

## Advanced Features Deep Dive

### Feature 1: Strategy Pattern Matching

Pattern matching allows for flexible strategy-based behavior:

#### Basic Strategy Matching

```typescript
import { MergeStrategy } from "effect"

const configureBuffer = (strategy: MergeStrategy.MergeStrategy): number =>
  MergeStrategy.match(strategy, {
    onBackPressure: () => 16, // Conservative buffer for backpressure
    onBufferSliding: () => 128 // Larger buffer for sliding
  })
```

#### Real-World Strategy Selection

```typescript
import { Effect, Config } from "effect"

interface StreamConfig {
  readonly maxLatencyMs: number
  readonly dataImportance: "critical" | "normal" | "analytics"
  readonly throughputRequirement: "low" | "medium" | "high"
}

const selectMergeStrategy = (config: StreamConfig): MergeStrategy.MergeStrategy => {
  if (config.dataImportance === "critical") {
    return MergeStrategy.BackPressure()
  }
  
  if (config.throughputRequirement === "high" && config.maxLatencyMs < 100) {
    return MergeStrategy.BufferSliding()
  }
  
  return MergeStrategy.BackPressure()
}

// Configuration-driven stream processor
const createStreamProcessor = <A>(
  streams: Stream.Stream<A>[],
  config: StreamConfig
) => {
  const strategy = selectMergeStrategy(config)
  const bufferSize = MergeStrategy.match(strategy, {
    onBackPressure: () => Math.min(config.maxLatencyMs / 10, 64),
    onBufferSliding: () => Math.max(config.maxLatencyMs / 5, 128)
  })
  
  return Stream.mergeAll(streams, {
    concurrency: config.throughputRequirement === "high" ? 8 : 4,
    bufferSize,
    mergeStrategy: strategy
  })
}
```

### Feature 2: Advanced Strategy Composition

#### Strategy-Based Resource Management

```typescript
import { Effect, Resource, Scope } from "effect"

interface ResourceConfig {
  readonly maxConnections: number
  readonly strategy: MergeStrategy.MergeStrategy
}

const createResourcePool = <R>(
  resourceFactory: Effect.Effect<R>,
  config: ResourceConfig
) => Effect.gen(function* () {
  const connectionPool = yield* Resource.make(
    Effect.succeed(new Set<R>()),
    (pool) => Effect.forEach(pool, resource => Effect.succeed(resource), {
      concurrency: "unbounded"
    })
  )
  
  return {
    acquire: Effect.gen(function* () {
      const pool = yield* connectionPool
      if (pool.size >= config.maxConnections) {
        return yield* MergeStrategy.match(config.strategy, {
          onBackPressure: () => Effect.retry(Effect.sleep("100 millis"), { times: 10 }),
          onBufferSliding: () => Effect.fail(new Error("Pool exhausted"))
        })
      }
      
      const resource = yield* resourceFactory
      pool.add(resource)
      return resource
    }),
    
    release: (resource: R) => Effect.gen(function* () {
      const pool = yield* connectionPool
      pool.delete(resource)
    })
  }
})
```

## Practical Patterns & Best Practices

### Pattern 1: Adaptive Strategy Selection

```typescript
import { Effect, Ref, Metrics } from "effect"

// Helper for runtime strategy adaptation
const createAdaptiveStrategy = () => Effect.gen(function* () {
  const latencyRef = yield* Ref.make(0)
  const throughputRef = yield* Ref.make(0)
  
  const updateMetrics = (latency: number, throughput: number) =>
    Effect.gen(function* () {
      yield* Ref.set(latencyRef, latency)
      yield* Ref.set(throughputRef, throughput)
    })
  
  const getCurrentStrategy = () => Effect.gen(function* () {
    const latency = yield* Ref.get(latencyRef)
    const throughput = yield* Ref.get(throughputRef)
    
    // Adapt strategy based on current conditions
    if (latency > 500 && throughput < 100) {
      return MergeStrategy.BufferSliding() // Prefer throughput
    }
    
    return MergeStrategy.BackPressure() // Default to safety
  })
  
  return { updateMetrics, getCurrentStrategy }
})

// Usage in adaptive stream processing
const adaptiveStreamProcessor = <A>(streams: Stream.Stream<A>[]) =>
  Effect.gen(function* () {
    const adapter = yield* createAdaptiveStrategy()
    
    return Stream.fromEffect(adapter.getCurrentStrategy).pipe(
      Stream.flatMap(strategy =>
        Stream.mergeAll(streams, {
          concurrency: 4,
          bufferSize: 32,
          mergeStrategy: strategy
        })
      )
    )
  })
```

### Pattern 2: Strategy-Based Error Handling

```typescript
import { Effect, Cause } from "effect"

// Error handling based on merge strategy
const handleMergeError = <E>(
  strategy: MergeStrategy.MergeStrategy,
  error: E
): Effect.Effect<void, E> =>
  MergeStrategy.match(strategy, {
    onBackPressure: () => 
      Effect.logError("Backpressure error - will retry").pipe(
        Effect.zipRight(Effect.fail(error))
      ),
    onBufferSliding: () =>
      Effect.logWarning("Buffer sliding error - continuing").pipe(
        Effect.zipRight(Effect.void)
      )
  })

// Resilient stream with strategy-aware error handling
const resilientStreamProcessor = <A, E>(
  streams: Stream.Stream<A, E>[],
  strategy: MergeStrategy.MergeStrategy
) =>
  Stream.mergeAll(streams, {
    concurrency: 4,
    bufferSize: 32,
    mergeStrategy: strategy
  }).pipe(
    Stream.catchAll(error => Stream.fromEffect(handleMergeError(strategy, error)))
  )
```

### Pattern 3: Performance Monitoring

```typescript
import { Effect, Metrics, Schedule } from "effect"

// Strategy performance tracking
const createStrategyMetrics = (strategyName: string) => {
  const throughput = Metrics.counter(`merge_strategy_throughput_${strategyName}`)
  const backpressureEvents = Metrics.counter(`merge_strategy_backpressure_${strategyName}`)
  const bufferUtilization = Metrics.gauge(`merge_strategy_buffer_utilization_${strategyName}`)
  
  return { throughput, backpressureEvents, bufferUtilization }
}

// Monitored merge operation
const monitoredMerge = <A>(
  streams: Stream.Stream<A>[],
  strategy: MergeStrategy.MergeStrategy
) => Effect.gen(function* () {
  const strategyName = MergeStrategy.match(strategy, {
    onBackPressure: () => "backpressure",
    onBufferSliding: () => "buffer_sliding"
  })
  
  const metrics = createStrategyMetrics(strategyName)
  
  return Stream.mergeAll(streams, {
    concurrency: 4,
    bufferSize: 32,
    mergeStrategy: strategy
  }).pipe(
    Stream.tap(() => Effect.succeed(metrics.throughput).pipe(Effect.flatMap(Metrics.increment))),
    Stream.tapErrorCause(cause => 
      Effect.succeed(metrics.backpressureEvents).pipe(Effect.flatMap(Metrics.increment))
    )
  )
})
```

## Integration Examples

### Integration with HTTP Client Pools

```typescript
import { HttpClient, HttpClientRequest } from "@effect/platform"
import { Effect, Stream, MergeStrategy } from "effect"

interface ApiRequest {
  readonly url: string
  readonly method: "GET" | "POST" | "PUT" | "DELETE"
  readonly body?: unknown
  readonly priority: "high" | "normal" | "low"
}

// HTTP client with strategy-based request handling
const createHttpClientPool = (strategy: MergeStrategy.MergeStrategy) =>
  Effect.gen(function* () {
    const client = yield* HttpClient.HttpClient
    
    const makeRequest = (request: ApiRequest) =>
      Effect.gen(function* () {
        const httpRequest = HttpClientRequest.make(request.method)(request.url)
        return yield* client.execute(httpRequest)
      })
    
    const processRequests = (requests: Stream.Stream<ApiRequest>) =>
      requests.pipe(
        Stream.groupByKey(req => req.priority),
        Stream.map(([priority, requestStream]) => {
          const concurrency = priority === "high" ? 8 : 4
          const bufferSize = priority === "high" ? 64 : 32
          
          return requestStream.pipe(
            Stream.mapEffect(makeRequest, { concurrency }),
            Stream.buffer({ capacity: bufferSize, strategy: 
              MergeStrategy.match(strategy, {
                onBackPressure: () => "suspend" as const,
                onBufferSliding: () => "dropping" as const
              })
            })
          )
        }),
        Stream.mergeAll({
          concurrency: 3,
          bufferSize: 16,
          mergeStrategy: strategy
        })
      )
    
    return { processRequests }
  })
```

### Integration with Database Operations

```typescript
import { SqlClient } from "@effect/sql"
import { Effect, Stream, MergeStrategy, Pool } from "effect"

interface DatabaseQuery {
  readonly sql: string
  readonly params: readonly unknown[]
  readonly timeout: number
}

// Database connection pool with merge strategies
const createDatabaseProcessor = (
  connectionPool: Pool.Pool<SqlClient.SqlClient, Error>,
  strategy: MergeStrategy.MergeStrategy
) => {
  const executeQuery = (query: DatabaseQuery) =>
    Pool.use(connectionPool, client =>
      client.execute(query.sql, query.params).pipe(
        Effect.timeout(Duration.millis(query.timeout))
      )
    )
  
  const processBatch = (queries: Stream.Stream<DatabaseQuery>) =>
    Stream.mergeAll([queries], {
      concurrency: MergeStrategy.match(strategy, {
        onBackPressure: () => 4, // Conservative for database connections
        onBufferSliding: () => 8  // Higher throughput when dropping is acceptable
      }),
      bufferSize: 32,
      mergeStrategy: strategy
    }).pipe(
      Stream.mapEffect(executeQuery),
      Stream.tapError(error => 
        Effect.logError(`Database query failed: ${error}`)
      )
    )
  
  return { processBatch }
}
```

### Testing Strategies

```typescript
import { Effect, TestClock, TestServices, Fiber } from "effect"
import { describe, it, expect } from "@effect/vitest"

// Test helpers for merge strategy behavior
const createTestStream = <A>(values: A[], delay: Duration.Duration) =>
  Stream.fromIterable(values).pipe(
    Stream.tap(() => TestClock.adjust(delay))
  )

describe("MergeStrategy", () => {
  it.effect("BackPressure maintains all data", () =>
    Effect.gen(function* () {
      const stream1 = createTestStream([1, 2, 3], Duration.millis(100))
      const stream2 = createTestStream([4, 5, 6], Duration.millis(50))
      
      const merged = Stream.mergeAll([stream1, stream2], {
        concurrency: 2,
        bufferSize: 2, // Small buffer to test backpressure
        mergeStrategy: MergeStrategy.BackPressure()
      })
      
      const result = yield* Stream.runCollect(merged).pipe(
        Effect.provide(TestServices.TestServices)
      )
      
      expect(Chunk.toArray(result).sort()).toEqual([1, 2, 3, 4, 5, 6])
    })
  )
  
  it.effect("BufferSliding may drop data under pressure", () =>
    Effect.gen(function* () {
      const fastStream = createTestStream(Array.from({ length: 100 }, (_, i) => i), Duration.millis(1))
      const slowConsumer = Stream.tap(fastStream, () => TestClock.adjust(Duration.millis(50)))
      
      const merged = Stream.mergeAll([slowConsumer], {
        concurrency: 1,
        bufferSize: 5, // Very small buffer
        mergeStrategy: MergeStrategy.BufferSliding()
      })
      
      const result = yield* Stream.runCollect(merged).pipe(
        Effect.provide(TestServices.TestServices)
      )
      
      // With buffer sliding, we expect fewer than 100 items
      expect(Chunk.size(result)).toBeLessThan(100)
    })
  )
})
```

## Conclusion

MergeStrategy provides **predictable backpressure control**, **flexible buffer management**, and **composable stream coordination** for concurrent data processing.

Key benefits:
- **Backpressure Control**: Prevents system overload by controlling data flow between producers and consumers
- **Buffer Management**: Provides clear strategies for handling buffer overflow scenarios  
- **Composability**: Integrates seamlessly with Effect's Channel and Stream ecosystems

Use MergeStrategy when you need reliable, controlled merging of concurrent streams with predictable behavior under load. Choose BackPressure for critical data that cannot be lost, and BufferSliding for high-throughput scenarios where some data loss is acceptable.