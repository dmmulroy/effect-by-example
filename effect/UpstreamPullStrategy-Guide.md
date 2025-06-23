# UpstreamPullStrategy: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem UpstreamPullStrategy Solves

When building streaming applications that process data from upstream sources, one of the most critical decisions is **when to pull more data**. Traditional streaming approaches often lead to:

```typescript
// Traditional approach - naive pull strategy
class SimpleStream<T> {
  private buffer: T[] = []
  
  async process() {
    while (true) {
      // ❌ Always pull immediately - can overwhelm upstream
      const data = await this.pullFromUpstream()
      this.buffer.push(data)
      
      // ❌ No coordination between pulling and processing
      this.processBuffer()
    }
  }
}
```

This approach leads to:
- **Buffer Overflow** - Pulling too aggressively can overwhelm memory
- **Inefficient Batching** - No coordination between pull timing and processing needs
- **Resource Waste** - Upstream resources are consumed whether downstream is ready or not
- **Backpressure Issues** - No feedback mechanism to control flow

### The UpstreamPullStrategy Solution

UpstreamPullStrategy provides precise control over **when** to request more data from upstream sources, enabling efficient streaming with proper backpressure management.

```typescript
import { UpstreamPullStrategy, Option } from "effect"

// ✅ Coordinated pull strategy - pull after processing current item
const pullAfterNext = UpstreamPullStrategy.PullAfterNext(Option.none())

// ✅ Batch-oriented pull strategy - pull after all queued items are processed
const pullAfterAll = UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())
```

### Key Concepts

**PullAfterNext**: Pull immediately after processing the next available item - optimal for low-latency, item-by-item processing

**PullAfterAllEnqueued**: Pull only after all currently enqueued items have been processed - optimal for batch processing and memory efficiency

**EmitSeparator**: Optional value emitted between pulls to provide coordination signals or delimiters

## Basic Usage Patterns

### Pattern 1: Low-Latency Processing

```typescript
import { UpstreamPullStrategy, Option, Channel, Effect } from "effect"

// Pull immediately after each item for minimal latency
const lowLatencyStrategy = UpstreamPullStrategy.PullAfterNext(Option.none())

const processItemImmediately = <T>(source: Channel.Channel<T>) =>
  source.pipe(
    Channel.concatMapWithCustom(
      (item) => Channel.write(processItem(item)),
      () => {},
      () => {},
      () => lowLatencyStrategy,
      () => Channel.ChildExecutorDecision.Continue()
    )
  )

const processItem = <T>(item: T): T => {
  // Immediate processing logic
  return item
}
```

### Pattern 2: Batch Processing

```typescript
// Pull only after processing all enqueued items
const batchStrategy = UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())

const processBatch = <T>(source: Channel.Channel<T>) =>
  source.pipe(
    Channel.concatMapWithCustom(
      (item) => Channel.write(item),
      () => {},
      () => {},
      () => batchStrategy,
      () => Channel.ChildExecutorDecision.Continue()
    )
  )
```

### Pattern 3: Strategy with Separators

```typescript
// Emit separators between pull operations
const separatorStrategy = UpstreamPullStrategy.PullAfterNext(
  Option.some("---SEPARATOR---")
)

const processWithSeparators = <T>(source: Channel.Channel<T>) =>
  source.pipe(
    Channel.concatMapWithCustom(
      (item) => Channel.writeAll(item, "---SEPARATOR---"),
      () => {},
      () => {},
      () => separatorStrategy,
      () => Channel.ChildExecutorDecision.Continue()
    )
  )
```

## Real-World Examples

### Example 1: Database Record Processing

Efficiently process database records with adaptive pull strategies based on processing complexity.

```typescript
import { UpstreamPullStrategy, Channel, Effect, Option, Array as Arr } from "effect"

interface DatabaseRecord {
  readonly id: string
  readonly data: unknown
  readonly processingTime: "fast" | "slow"
}

const createAdaptiveDatabaseProcessor = () => {
  return Effect.gen(function* () {
    const fastStrategy = UpstreamPullStrategy.PullAfterNext(Option.none())
    const batchStrategy = UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())
    
    const processRecord = (record: DatabaseRecord) =>
      Channel.gen(function* () {
        // Simulate processing
        yield* Channel.fromEffect(
          Effect.sleep(record.processingTime === "fast" ? "10 millis" : "100 millis")
        )
        return { ...record, processed: true }
      })
    
    const adaptiveProcessor = (records: Channel.Channel<DatabaseRecord>) =>
      records.pipe(
        Channel.concatMapWithCustom(
          processRecord,
          () => {},
          () => {},
          (request) => UpstreamPullRequest.match(request, {
            onPulled: (record) => 
              // Use different strategies based on record type
              record.processingTime === "fast" ? fastStrategy : batchStrategy,
            onNoUpstream: () => batchStrategy
          }),
          () => Channel.ChildExecutorDecision.Continue()
        )
      )
    
    return { adaptiveProcessor }
  })
}
```

### Example 2: API Rate Limiting

Control API request rates using pull strategies to respect rate limits.

```typescript
interface ApiRequest {
  readonly url: string
  readonly priority: "high" | "low"
}

interface ApiResponse {
  readonly data: unknown
  readonly remaining: number
}

const createRateLimitedProcessor = (rateLimitPerSecond: number) => {
  return Effect.gen(function* () {
    const makeRequest = (request: ApiRequest): Effect.Effect<ApiResponse> =>
      Effect.gen(function* () {
        // Simulate API call
        const response = yield* Effect.tryPromise(() => 
          fetch(request.url).then(r => r.json())
        )
        return {
          data: response,
          remaining: rateLimitPerSecond - 1 // Simplified
        }
      })
    
    const rateLimitedStrategy = (remaining: number) =>
      remaining > rateLimitPerSecond * 0.8 
        ? UpstreamPullStrategy.PullAfterNext(Option.none())
        : UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())
    
    const processRequests = (requests: Channel.Channel<ApiRequest>) =>
      requests.pipe(
        Channel.concatMapWithCustom(
          (request) => Channel.fromEffect(makeRequest(request)),
          () => {},
          () => {},
          (pullRequest) => UpstreamPullRequest.match(pullRequest, {
            onPulled: () => UpstreamPullStrategy.PullAfterNext(Option.none()),
            onNoUpstream: () => UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())
          }),
          (response) => response.remaining > 0 
            ? Channel.ChildExecutorDecision.Continue()
            : Channel.ChildExecutorDecision.Yield()
        )
      )
    
    return { processRequests }
  })
}
```

### Example 3: Memory-Conscious Stream Processing

Process large datasets while managing memory usage through strategic pull timing.

```typescript
interface DataChunk {
  readonly size: number
  readonly data: ReadonlyArray<unknown>
}

const createMemoryEfficientProcessor = (maxMemory: number) => {
  return Effect.gen(function* () {
    let currentMemoryUsage = 0
    
    const memoryStrategy = (chunkSize: number) => {
      const wouldExceedMemory = currentMemoryUsage + chunkSize > maxMemory
      return wouldExceedMemory
        ? UpstreamPullStrategy.PullAfterAllEnqueued(Option.some("MEMORY_WAIT"))
        : UpstreamPullStrategy.PullAfterNext(Option.none())
    }
    
    const processChunk = (chunk: DataChunk) =>
      Channel.gen(function* () {
        currentMemoryUsage += chunk.size
        
        // Process the chunk
        const processedData = yield* Channel.fromEffect(
          Effect.sync(() => 
            chunk.data.map(item => ({ item, timestamp: Date.now() }))
          )
        )
        
        // Release memory
        currentMemoryUsage = Math.max(0, currentMemoryUsage - chunk.size)
        
        return { processedData, memoryUsed: currentMemoryUsage }
      })
    
    const memoryEfficientProcessor = (chunks: Channel.Channel<DataChunk>) =>
      chunks.pipe(
        Channel.concatMapWithCustom(
          processChunk,
          () => {},
          () => {},
          (pullRequest) => UpstreamPullRequest.match(pullRequest, {
            onPulled: (chunk) => memoryStrategy(chunk.size),
            onNoUpstream: () => UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())
          }),
          () => Channel.ChildExecutorDecision.Continue()
        )
      )
    
    return { memoryEfficientProcessor, getCurrentMemoryUsage: () => currentMemoryUsage }
  })
}
```

## Advanced Features Deep Dive

### Feature 1: Dynamic Strategy Selection

Choose pull strategies dynamically based on runtime conditions.

#### Basic Dynamic Strategy Usage

```typescript
const createDynamicStrategy = <T>(
  condition: (item: T) => boolean
) => (item: T) =>
  condition(item)
    ? UpstreamPullStrategy.PullAfterNext(Option.none())
    : UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())
```

#### Real-World Dynamic Strategy Example

```typescript
interface WorkItem {
  readonly type: "cpu_intensive" | "io_bound" | "lightweight"
  readonly data: unknown
}

const workloadAwareStrategy = (item: WorkItem, queueSize: number) => {
  // Strategy selection based on work type and current load
  if (item.type === "lightweight") {
    return UpstreamPullStrategy.PullAfterNext(Option.none())
  }
  
  if (item.type === "cpu_intensive" && queueSize > 10) {
    return UpstreamPullStrategy.PullAfterAllEnqueued(Option.some("CPU_BACKOFF"))
  }
  
  return UpstreamPullStrategy.PullAfterNext(Option.none())
}

const processWorkItems = (
  items: Channel.Channel<WorkItem>,
  getQueueSize: () => number
) =>
  items.pipe(
    Channel.concatMapWithCustom(
      (item) => Channel.fromEffect(processWorkItem(item)),
      () => {},
      () => {},
      (pullRequest) => UpstreamPullRequest.match(pullRequest, {
        onPulled: (item) => workloadAwareStrategy(item, getQueueSize()),
        onNoUpstream: () => UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())
      }),
      () => Channel.ChildExecutorDecision.Continue()
    )
  )

const processWorkItem = (item: WorkItem): Effect.Effect<{ result: unknown }> =>
  Effect.gen(function* () {
    const delay = item.type === "cpu_intensive" ? "100 millis" :
                  item.type === "io_bound" ? "50 millis" : "10 millis"
    
    yield* Effect.sleep(delay)
    return { result: `processed-${item.type}` }
  })
```

### Feature 2: Separator-Based Coordination

Use separators to coordinate between different processing stages.

#### Advanced Separator Usage

```typescript
const createCoordinatedProcessor = <T, R>(
  processor: (item: T) => Effect.Effect<R>,
  separatorValue: string
) => {
  const strategyWithSeparators = UpstreamPullStrategy.PullAfterNext(
    Option.some(separatorValue)
  )
  
  return (source: Channel.Channel<T>) =>
    source.pipe(
      Channel.concatMapWithCustom(
        (item) => Channel.fromEffect(processor(item)),
        () => {},
        () => {},
        () => strategyWithSeparators,
        (result) => Channel.ChildExecutorDecision.Continue()
      )
    )
}

// Usage with coordination signals
const coordinated = createCoordinatedProcessor(
  (data: string) => Effect.succeed(data.toUpperCase()),
  "---BATCH_COMPLETE---"
)
```

## Practical Patterns & Best Practices

### Pattern 1: Strategy Factory Pattern

Create reusable strategy factories for common scenarios.

```typescript
const StrategyFactory = {
  // High-throughput processing
  highThroughput: () => UpstreamPullStrategy.PullAfterNext(Option.none()),
  
  // Memory-conscious processing
  memoryConscious: () => UpstreamPullStrategy.PullAfterAllEnqueued(Option.none()),
  
  // Coordinated with separators
  withSeparator: <T>(separator: T) => 
    UpstreamPullStrategy.PullAfterNext(Option.some(separator)),
  
  // Adaptive based on load
  adaptive: (currentLoad: number, threshold: number) =>
    currentLoad > threshold
      ? UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())
      : UpstreamPullStrategy.PullAfterNext(Option.none()),
  
  // Rate-limited
  rateLimited: (requestsPerSecond: number, currentRate: number) =>
    currentRate > requestsPerSecond * 0.8
      ? UpstreamPullStrategy.PullAfterAllEnqueued(Option.some("RATE_LIMIT"))
      : UpstreamPullStrategy.PullAfterNext(Option.none())
}

// Usage
const processWithFactory = <T>(
  source: Channel.Channel<T>,
  strategyName: keyof typeof StrategyFactory
) => {
  const strategy = StrategyFactory[strategyName]()
  
  return source.pipe(
    Channel.concatMapWithCustom(
      (item) => Channel.write(item),
      () => {},
      () => {},
      () => strategy,
      () => Channel.ChildExecutorDecision.Continue()
    )
  )
}
```

### Pattern 2: Strategy Composition

Combine multiple strategies for complex scenarios.

```typescript
const createCompositeStrategy = <T>(
  strategies: ReadonlyArray<{
    condition: (item: T, context: unknown) => boolean
    strategy: UpstreamPullStrategy.UpstreamPullStrategy<unknown>
  }>,
  defaultStrategy: UpstreamPullStrategy.UpstreamPullStrategy<unknown>
) => (item: T, context: unknown) => {
  const matchingStrategy = strategies.find(s => s.condition(item, context))
  return matchingStrategy?.strategy ?? defaultStrategy
}

// Usage example
const compositeStrategy = createCompositeStrategy([
  {
    condition: (item: WorkItem) => item.type === "cpu_intensive",
    strategy: UpstreamPullStrategy.PullAfterAllEnqueued(Option.some("CPU_WAIT"))
  },
  {
    condition: (item: WorkItem) => item.type === "lightweight",
    strategy: UpstreamPullStrategy.PullAfterNext(Option.none())
  }
], UpstreamPullStrategy.PullAfterNext(Option.none()))
```

### Pattern 3: Strategy Monitoring

Monitor and adjust strategies based on performance metrics.

```typescript
interface StrategyMetrics {
  readonly pullCount: number
  readonly averageProcessingTime: number
  readonly memoryUsage: number
  readonly queueSize: number
}

const createMonitoredStrategy = () => {
  return Effect.gen(function* () {
    const metrics = yield* Ref.make<StrategyMetrics>({
      pullCount: 0,
      averageProcessingTime: 0,
      memoryUsage: 0,
      queueSize: 0
    })
    
    const updateMetrics = (update: Partial<StrategyMetrics>) =>
      Ref.update(metrics, current => ({ ...current, ...update }))
    
    const adaptiveStrategy = () =>
      Effect.gen(function* () {
        const current = yield* Ref.get(metrics)
        
        // Adapt strategy based on metrics
        if (current.memoryUsage > 0.8) {
          return UpstreamPullStrategy.PullAfterAllEnqueued(Option.some("HIGH_MEMORY"))
        }
        
        if (current.queueSize > 100) {
          return UpstreamPullStrategy.PullAfterAllEnqueued(Option.some("QUEUE_FULL"))
        }
        
        return UpstreamPullStrategy.PullAfterNext(Option.none())
      })
    
    return { adaptiveStrategy, updateMetrics, getMetrics: () => Ref.get(metrics) }
  })
}
```

## Integration Examples

### Integration with Stream Processing

```typescript
import { Stream, UpstreamPullStrategy, Channel, Effect } from "effect"

const createStreamWithPullStrategy = <T, R>(
  stream: Stream.Stream<T>,
  processor: (item: T) => Effect.Effect<R>,
  strategy: UpstreamPullStrategy.UpstreamPullStrategy<unknown>
) =>
  stream.pipe(
    Stream.toChannel,
    Channel.concatMapWithCustom(
      (item) => Channel.fromEffect(processor(item)),
      () => {},
      () => {},
      () => strategy,
      () => Channel.ChildExecutorDecision.Continue()
    ),
    Channel.toStream
  )

// Usage with different strategies
const processStreamFast = <T>(
  stream: Stream.Stream<T>,
  processor: (item: T) => Effect.Effect<T>
) =>
  createStreamWithPullStrategy(
    stream,
    processor,
    UpstreamPullStrategy.PullAfterNext(Option.none())
  )

const processStreamBatched = <T>(
  stream: Stream.Stream<T>,
  processor: (item: T) => Effect.Effect<T>
) =>
  createStreamWithPullStrategy(
    stream,
    processor,
    UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())
  )
```

### Testing Strategies

```typescript
import { Effect, TestContext, Ref } from "effect"

const testPullStrategy = Effect.gen(function* () {
  const testData = [1, 2, 3, 4, 5]
  const pullCounts = yield* Ref.make(0)
  
  const trackingStrategy = UpstreamPullStrategy.PullAfterNext(Option.none())
  
  const processor = (item: number) =>
    Effect.gen(function* () {
      yield* Ref.update(pullCounts, n => n + 1)
      return item * 2
    })
  
  const testChannel = Channel.fromIterable(testData).pipe(
    Channel.concatMapWithCustom(
      (item) => Channel.fromEffect(processor(item)),
      () => {},
      () => {},
      () => trackingStrategy,
      () => Channel.ChildExecutorDecision.Continue()
    )
  )
  
  const [results, finalCount] = yield* Effect.all([
    Channel.runCollect(testChannel),
    Ref.get(pullCounts)
  ])
  
  // Verify strategy behavior
  return {
    processedResults: results[0],
    pullCount: finalCount,
    originalData: testData
  }
})

// Run tests
const runStrategyTests = () =>
  Effect.gen(function* () {
    const testResult = yield* testPullStrategy
    
    console.log("Test Results:", testResult)
    // Assert expected behavior based on strategy
  }).pipe(
    Effect.provide(TestContext.live)
  )
```

## Conclusion

UpstreamPullStrategy provides **precise control over data flow timing**, **efficient resource utilization**, and **adaptive backpressure management** for high-performance streaming applications.

Key benefits:
- **Flow Control**: Fine-grained control over when to request more data from upstream
- **Resource Efficiency**: Prevent memory overflow and optimize processing patterns  
- **Adaptability**: Dynamic strategy selection based on runtime conditions

UpstreamPullStrategy is essential when building streaming applications that need to balance throughput, memory usage, and processing efficiency. Use PullAfterNext for low-latency scenarios and PullAfterAllEnqueued for batch processing and memory-conscious applications.