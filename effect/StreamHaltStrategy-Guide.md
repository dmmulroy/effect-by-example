# StreamHaltStrategy: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem StreamHaltStrategy Solves

When merging multiple streams in concurrent applications, determining when the merged stream should terminate becomes critical for resource management and application behavior. Traditional approaches often result in unpredictable termination behavior, resource leaks, or premature shutdowns.

```typescript
// Traditional approach - unpredictable termination
import { Readable } from 'stream'

function mergeStreams(stream1: Readable, stream2: Readable) {
  const results: any[] = []
  let finished = 0
  
  const handleEnd = () => {
    finished++
    // When should we consider the merge complete?
    // After first stream ends? After both? What about errors?
    if (finished >= 2) {
      // This logic is scattered and error-prone
      return results
    }
  }
  
  stream1.on('data', (data) => results.push(data))
  stream1.on('end', handleEnd)
  stream1.on('error', (err) => {
    // How do we handle errors from one stream?
    // Should the other stream continue?
    throw err
  })
  
  stream2.on('data', (data) => results.push(data))
  stream2.on('end', handleEnd)
  stream2.on('error', (err) => {
    // Duplicated error handling logic
    throw err
  })
  
  return results
}
```

This approach leads to:
- **Unpredictable Termination** - No clear strategy for when merged streams should halt
- **Resource Leaks** - Streams may continue running when they should be cleaned up
- **Complex Error Handling** - Difficult to determine how errors should propagate
- **Race Conditions** - Timing-dependent behavior in concurrent stream operations

### The StreamHaltStrategy Solution

Effect's StreamHaltStrategy provides explicit, composable control over how merged streams should terminate, ensuring predictable behavior and proper resource cleanup.

```typescript
import { Stream, StreamHaltStrategy, Effect } from "effect"

// The Effect solution - explicit, predictable, composable
const mergeStreamsWithStrategy = (strategy: StreamHaltStrategy.HaltStrategy) => {
  const stream1 = Stream.fromIterable([1, 2, 3]).pipe(
    Stream.schedule(Schedule.spaced("100 millis"))
  )
  
  const stream2 = Stream.fromIterable([4, 5, 6]).pipe(
    Stream.schedule(Schedule.spaced("200 millis"))
  )
  
  return Stream.merge(stream1, stream2, { haltStrategy: strategy })
}
```

### Key Concepts

**HaltStrategy**: A strategy that determines when a merged stream should terminate based on the termination status of its constituent streams.

**Left**: The merged stream terminates when the left (first) stream terminates, regardless of the right stream's status.

**Right**: The merged stream terminates when the right (second) stream terminates, regardless of the left stream's status.

**Both**: The merged stream terminates only when both streams have terminated (default behavior).

**Either**: The merged stream terminates when either stream terminates, whichever happens first.

## Basic Usage Patterns

### Pattern 1: Default Behavior (Both)

```typescript
import { Stream, Effect, Schedule } from "effect"

// Default behavior - wait for both streams to complete
const mergeBothComplete = Effect.gen(function* () {
  const numbers = Stream.fromIterable([1, 2, 3]).pipe(
    Stream.schedule(Schedule.spaced("100 millis"))
  )
  
  const letters = Stream.fromIterable(['a', 'b']).pipe(
    Stream.schedule(Schedule.spaced("150 millis"))
  )
  
  // Without specifying haltStrategy, defaults to "Both"
  const merged = Stream.merge(numbers, letters)
  
  return yield* Stream.runCollect(merged)
})

// Result: Both streams run to completion
// Output: Chunk([1, 'a', 2, 'b', 3])
```

### Pattern 2: Explicit Strategy Configuration

```typescript
import { StreamHaltStrategy } from "effect"

// Explicitly configure halt strategy
const mergeWithExplicitStrategy = Effect.gen(function* () {
  const fastStream = Stream.fromIterable([1, 2, 3, 4, 5]).pipe(
    Stream.schedule(Schedule.spaced("50 millis"))
  )
  
  const slowStream = Stream.fromIterable(['a', 'b']).pipe(
    Stream.schedule(Schedule.spaced("200 millis"))
  )
  
  // Terminate when either stream finishes
  const merged = Stream.merge(fastStream, slowStream, {
    haltStrategy: StreamHaltStrategy.Either
  })
  
  return yield* Stream.runCollect(merged)
})

// Result: Terminates when slowStream finishes first
// Output: Chunk([1, 'a', 2, 'b'])
```

### Pattern 3: String-based Strategy Input

```typescript
// Using string input for convenience
const mergeWithStringStrategy = Effect.gen(function* () {
  const primary = Stream.fromIterable([1, 2, 3])
  const secondary = Stream.fromIterable(['x', 'y', 'z', 'w'])
  
  // String input is automatically converted to HaltStrategy
  const merged = Stream.merge(primary, secondary, {
    haltStrategy: "left" // Equivalent to StreamHaltStrategy.Left
  })
  
  return yield* Stream.runCollect(merged)
})

// Result: Terminates when primary stream finishes
```

## Real-World Examples

### Example 1: Real-time Data Processing with Fallback

Real-time applications often need to merge a primary data stream with a fallback stream, terminating when the primary source completes.

```typescript
import { Stream, Effect, Schedule, Random } from "effect"

// Simulate real-time sensor data
const createSensorStream = (sensorId: string, interval: number) =>
  Stream.repeatEffect(
    Random.nextIntBetween(20, 30).pipe(
      Effect.map(temp => ({
        sensorId,
        temperature: temp,
        timestamp: Date.now()
      }))
    )
  ).pipe(
    Stream.take(10), // Simulate finite data
    Stream.schedule(Schedule.spaced(`${interval} millis`))
  )

// Real-world sensor data processing
const processSensorData = Effect.gen(function* () {
  const primarySensor = createSensorStream("SENSOR_001", 100)
  const backupSensor = createSensorStream("BACKUP_001", 150)
  
  // Use Left strategy - stop when primary sensor stops
  const sensorData = Stream.merge(primarySensor, backupSensor, {
    haltStrategy: StreamHaltStrategy.Left
  }).pipe(
    Stream.tap(reading => 
      Effect.log(`Received: ${reading.sensorId} - ${reading.temperature}°C`)
    ),
    Stream.filter(reading => reading.temperature > 25), // Alert threshold
    Stream.map(reading => ({
      ...reading,
      alert: reading.temperature > 28 ? "HIGH" : "NORMAL"
    }))
  )
  
  return yield* Stream.runCollect(sensorData)
})

// Result: Processes data until primary sensor stops, regardless of backup status
```

### Example 2: API Rate Limiting with Multiple Endpoints

When working with multiple API endpoints with different rate limits, you may want to terminate processing when the fastest endpoint completes its quota.

```typescript
import { Stream, Effect, Schedule, HttpClient } from "effect"

interface ApiResponse {
  endpoint: string
  data: unknown[]
  timestamp: number
}

// Simulate API endpoints with different response times
const createApiStream = (endpoint: string, delay: number, maxRequests: number) =>
  Stream.range(1, maxRequests).pipe(
    Stream.mapEffect(requestId =>
      Effect.gen(function* () {
        // Simulate API call delay
        yield* Effect.sleep(`${delay} millis`)
        return {
          endpoint,
          data: [`data_${requestId}`],
          timestamp: Date.now()
        } satisfies ApiResponse
      })
    ),
    Stream.schedule(Schedule.spaced(`${delay} millis`))
  )

// Process multiple API endpoints
const processMultipleApis = Effect.gen(function* () {
  const fastApi = createApiStream("fast-api", 100, 5)
  const thoroughApi = createApiStream("thorough-api", 300, 15)
  
  // Use Either strategy - stop when first API completes its quota
  const apiData = Stream.merge(fastApi, thoroughApi, {
    haltStrategy: StreamHaltStrategy.Either
  }).pipe(
    Stream.tap(response =>
      Effect.log(`API Response from ${response.endpoint}: ${response.data.length} items`)
    ),
    Stream.groupBy({
      key: response => response.endpoint,
      buffer: 10
    }),
    Stream.map(([endpoint, responses]) => ({
      endpoint,
      totalResponses: responses.length,
      totalItems: responses.reduce((sum, r) => sum + r.data.length, 0)
    }))
  )
  
  return yield* Stream.runCollect(apiData)
})

// Result: Stops processing when the faster API completes, saving resources
```

### Example 3: File Processing with Error Recovery

Processing multiple files concurrently with graceful degradation when some files fail.

```typescript
import { Stream, Effect, Cause } from "effect"
import * as NodeFS from "@effect/platform-node/NodeFileSystem"

interface FileProcessingResult {
  filename: string
  status: "success" | "error"
  content?: string
  error?: string
}

// Process files with error recovery
const processFiles = (filePaths: string[]) => Effect.gen(function* () {
  const fileStreams = filePaths.map((filePath, index) =>
    Stream.fromEffect(
      Effect.gen(function* () {
        try {
          const content = yield* NodeFS.NodeFileSystem.readFileString(filePath)
          return {
            filename: filePath,
            status: "success" as const,
            content: content.slice(0, 100) // Preview
          }
        } catch (error) {
          return {
            filename: filePath,
            status: "error" as const,
            error: String(error)
          }
        }
      }).pipe(
        Effect.catchAll(error => 
          Effect.succeed({
            filename: filePath,
            status: "error" as const,
            error: Cause.pretty(error)
          })
        )
      )
    )
  )
  
  // Process all files, but use Right strategy for priority file
  const [priorityFile, ...otherFiles] = fileStreams
  
  if (otherFiles.length === 0) {
    return yield* Stream.runCollect(priorityFile)
  }
  
  const combinedOtherFiles = otherFiles.reduce((acc, stream) =>
    Stream.merge(acc, stream, { haltStrategy: StreamHaltStrategy.Both })
  )
  
  // Use Right strategy - continue until priority file and all others complete
  const allResults = Stream.merge(priorityFile, combinedOtherFiles, {
    haltStrategy: StreamHaltStrategy.Both
  }).pipe(
    Stream.tap(result =>
      result.status === "error" 
        ? Effect.logError(`Failed to process ${result.filename}: ${result.error}`)
        : Effect.log(`Successfully processed ${result.filename}`)
    )
  )
  
  return yield* Stream.runCollect(allResults)
})

// Example usage
const processProjectFiles = processFiles([
  "./package.json",    // Priority file
  "./README.md",
  "./src/index.ts",
  "./nonexistent.txt"  // This will error gracefully
])
```

## Advanced Features Deep Dive

### Strategy Pattern Matching

Use pattern matching to implement complex termination logic based on strategy type.

```typescript
import { StreamHaltStrategy, Effect } from "effect"

// Advanced strategy analysis
const analyzeStrategy = (strategy: StreamHaltStrategy.HaltStrategy) =>
  StreamHaltStrategy.match(strategy, {
    onLeft: () => Effect.log("Strategy: Terminate when left stream ends"),
    onRight: () => Effect.log("Strategy: Terminate when right stream ends"),
    onBoth: () => Effect.log("Strategy: Wait for both streams to complete"),
    onEither: () => Effect.log("Strategy: Terminate when any stream ends")
  })

// Helper for strategy-aware stream merging
const createStrategicMerge = <A, E, R, A2, E2, R2>(
  leftStream: Stream.Stream<A, E, R>,
  rightStream: Stream.Stream<A2, E2, R2>,
  strategy: StreamHaltStrategy.HaltStrategy
) => Effect.gen(function* () {
  yield* analyzeStrategy(strategy)
  
  const merged = Stream.merge(leftStream, rightStream, {
    haltStrategy: strategy
  })
  
  return yield* Stream.runCollect(merged)
})
```

### Dynamic Strategy Selection

Select strategies based on runtime conditions for adaptive behavior.

```typescript
// Dynamic strategy selection based on stream characteristics
const selectOptimalStrategy = <A, B>(
  leftStream: Stream.Stream<A, never, never>,
  rightStream: Stream.Stream<B, never, never>,
  leftPriority: number,
  rightPriority: number
): StreamHaltStrategy.HaltStrategy => {
  if (leftPriority > rightPriority) {
    return StreamHaltStrategy.Left
  } else if (rightPriority > leftPriority) {
    return StreamHaltStrategy.Right
  } else {
    return StreamHaltStrategy.Both
  }
}

// Adaptive stream processing
const adaptiveStreamMerge = <A, B>(
  config: {
    leftStream: Stream.Stream<A, never, never>
    rightStream: Stream.Stream<B, never, never>
    leftPriority: number
    rightPriority: number
    timeoutMs?: number
  }
) => Effect.gen(function* () {
  const strategy = selectOptimalStrategy(
    config.leftStream,
    config.rightStream,
    config.leftPriority,
    config.rightPriority
  )
  
  let mergedStream = Stream.merge(config.leftStream, config.rightStream, {
    haltStrategy: strategy
  })
  
  // Add timeout if specified
  if (config.timeoutMs) {
    mergedStream = mergedStream.pipe(
      Stream.timeoutFail(
        () => new Error(`Stream merge timed out after ${config.timeoutMs}ms`),
        `${config.timeoutMs} millis`
      )
    )
  }
  
  return yield* Stream.runCollect(mergedStream)
})
```

### Complex Multi-Stream Scenarios

Handle complex scenarios with multiple streams and nested merge operations.

```typescript
// Complex multi-stream merging with different strategies
const complexStreamOrchestration = Effect.gen(function* () {
  // Create different types of streams
  const criticalStream = Stream.fromIterable(['CRITICAL_1', 'CRITICAL_2'])
  const normalStream = Stream.fromIterable(['NORMAL_1', 'NORMAL_2', 'NORMAL_3'])
  const backgroundStream = Stream.fromIterable(['BG_1', 'BG_2', 'BG_3', 'BG_4'])
  
  // First merge: Critical + Normal (wait for critical to complete)
  const primaryFlow = Stream.merge(criticalStream, normalStream, {
    haltStrategy: StreamHaltStrategy.Left
  })
  
  // Second merge: Primary + Background (either completes the flow)
  const completeFlow = Stream.merge(primaryFlow, backgroundStream, {
    haltStrategy: StreamHaltStrategy.Either
  })
  
  return yield* Stream.runCollect(completeFlow)
})
```

## Practical Patterns & Best Practices

### Pattern 1: Graceful Shutdown Helper

Create reusable utilities for graceful stream termination.

```typescript
// Graceful shutdown utility
const createGracefulMerge = <A, E, R, A2, E2, R2>(
  primaryStream: Stream.Stream<A, E, R>,
  fallbackStream: Stream.Stream<A2, E2, R2>,
  options: {
    strategy?: StreamHaltStrategy.HaltStrategyInput
    timeoutMs?: number
    onShutdown?: Effect.Effect<void, never, never>
  } = {}
) => Effect.gen(function* () {
  const strategy = options.strategy ? 
    StreamHaltStrategy.fromInput(options.strategy) : 
    StreamHaltStrategy.Both
  
  let merged = Stream.merge(primaryStream, fallbackStream, {
    haltStrategy: strategy
  })
  
  // Add timeout if specified
  if (options.timeoutMs) {
    merged = merged.pipe(
      Stream.timeoutFail(
        () => new Error('Stream merge timeout'),
        `${options.timeoutMs} millis`
      )
    )
  }
  
  // Add shutdown hook
  if (options.onShutdown) {
    merged = merged.pipe(
      Stream.onDone(() => options.onShutdown!)
    )
  }
  
  return merged
})

// Usage example
const gracefulDataProcessing = Effect.gen(function* () {
  const primaryData = Stream.fromIterable([1, 2, 3])
  const backupData = Stream.fromIterable([4, 5, 6])
  
  const stream = yield* createGracefulMerge(primaryData, backupData, {
    strategy: "left",
    timeoutMs: 5000,
    onShutdown: Effect.log("Stream processing completed")
  })
  
  return yield* Stream.runCollect(stream)
})
```

### Pattern 2: Strategy Composition

Compose multiple strategies for complex termination behavior.

```typescript
// Strategy composition utilities
const createConditionalStrategy = (
  condition: boolean,
  trueStrategy: StreamHaltStrategy.HaltStrategy,
  falseStrategy: StreamHaltStrategy.HaltStrategy
): StreamHaltStrategy.HaltStrategy => condition ? trueStrategy : falseStrategy

const createTimeBasedStrategy = (
  durationMs: number
): StreamHaltStrategy.HaltStrategy => {
  const startTime = Date.now()
  return Date.now() - startTime > durationMs ? 
    StreamHaltStrategy.Either : 
    StreamHaltStrategy.Both
}

// Compose strategies for adaptive behavior
const adaptiveProcessing = (isHighPriority: boolean) => Effect.gen(function* () {
  const dataStream = Stream.fromIterable([1, 2, 3, 4, 5])
  const monitorStream = Stream.fromIterable(['status', 'health', 'metrics'])
  
  const strategy = createConditionalStrategy(
    isHighPriority,
    StreamHaltStrategy.Left,  // High priority: focus on data
    StreamHaltStrategy.Both   // Normal priority: wait for both
  )
  
  const merged = Stream.merge(dataStream, monitorStream, {
    haltStrategy: strategy
  })
  
  return yield* Stream.runCollect(merged)
})
```

### Pattern 3: Error Recovery with Strategy Fallback

Implement error recovery patterns that adapt strategy based on failure conditions.

```typescript
// Error-aware strategy selection
const createResilientMerge = <A, E, R, A2, E2, R2>(
  primaryStream: Stream.Stream<A, E, R>,
  fallbackStream: Stream.Stream<A2, E2, R2>
) => Effect.gen(function* () {
  // First attempt: Both streams must complete
  const strictMerge = Stream.merge(primaryStream, fallbackStream, {
    haltStrategy: StreamHaltStrategy.Both
  }).pipe(
    Stream.timeout(`5 seconds`),
    Stream.catchAll(() => Stream.empty) // Convert timeout to empty stream
  )
  
  const strictResult = yield* Stream.runCollect(strictMerge)
  
  // If strict merge succeeded, return results
  if (strictResult.length > 0) {
    return strictResult
  }
  
  // Fallback: Accept results from either stream
  yield* Effect.log("Falling back to Either strategy due to timeout")
  
  const lenientMerge = Stream.merge(primaryStream, fallbackStream, {
    haltStrategy: StreamHaltStrategy.Either
  })
  
  return yield* Stream.runCollect(lenientMerge)
})

// Usage in error-prone environments
const resilientDataCollection = createResilientMerge(
  Stream.fromIterable([1, 2, 3]).pipe(
    Stream.schedule(Schedule.spaced("1 second"))
  ),
  Stream.fromIterable(['a', 'b', 'c']).pipe(
    Stream.schedule(Schedule.spaced("2 seconds"))
  )
)
```

## Integration Examples

### Integration with Effect Config

Use Effect's Config system to make halt strategies configurable.

```typescript
import { Config, Effect } from "effect"

// Configurable halt strategy
const HaltStrategyConfig = Config.literal("left", "right", "both", "either")("HALT_STRATEGY").pipe(
  Config.withDefault("both" as const)
)

const configurableStreamMerge = Effect.gen(function* () {
  const strategy = yield* HaltStrategyConfig
  
  const stream1 = Stream.fromIterable([1, 2, 3])
  const stream2 = Stream.fromIterable(['a', 'b', 'c'])
  
  const merged = Stream.merge(stream1, stream2, {
    haltStrategy: strategy
  })
  
  return yield* Stream.runCollect(merged)
})

// Set environment variable: HALT_STRATEGY=either
// The stream will use Either strategy
```

### Integration with Metrics and Monitoring

Monitor stream termination behavior using Effect's metrics system.

```typescript
import { Metric, Effect } from "effect"

// Define metrics for stream monitoring
const StreamTerminationMetric = Metric.counter("stream_terminations").pipe(
  Metric.tagged("strategy", "unknown")
)

const StreamProcessingDuration = Metric.histogram("stream_processing_duration_ms")

// Monitored stream merge
const monitoredStreamMerge = <A, B>(
  leftStream: Stream.Stream<A, never, never>,
  rightStream: Stream.Stream<B, never, never>,
  strategy: StreamHaltStrategy.HaltStrategy
) => Effect.gen(function* () {
  const startTime = Date.now()
  
  const strategyLabel = StreamHaltStrategy.match(strategy, {
    onLeft: () => "left",
    onRight: () => "right", 
    onBoth: () => "both",
    onEither: () => "either"
  })
  
  const merged = Stream.merge(leftStream, rightStream, {
    haltStrategy: strategy
  }).pipe(
    Stream.onDone(() => 
      StreamTerminationMetric.pipe(
        Metric.tagged("strategy", strategyLabel)
      ).pipe(
        Metric.increment()
      )
    )
  )
  
  const result = yield* Stream.runCollect(merged)
  
  const duration = Date.now() - startTime
  yield* StreamProcessingDuration(duration)
  
  return result
})
```

### Integration with Resource Management

Properly manage resources in streams with different halt strategies.

```typescript
import { Effect, Resource, Scope } from "effect"

// Resource-aware stream processing
const createResourceAwareStream = <A>(
  resourceName: string,
  data: Iterable<A>
) => Stream.acquireRelease(
  Effect.gen(function* () {
    yield* Effect.log(`Acquiring resource: ${resourceName}`)
    return { name: resourceName, data: Array.from(data) }
  }),
  (resource) => Effect.log(`Releasing resource: ${resource.name}`)
).pipe(
  Stream.flatMap(resource => 
    Stream.fromIterable(resource.data).pipe(
      Stream.tap(item => Effect.log(`Processing ${item} from ${resource.name}`))
    )
  )
)

// Safe resource management with halt strategies
const resourceSafeProcessing = Effect.gen(function* () {
  const resource1 = createResourceAwareStream("DB_CONNECTION", [1, 2, 3])
  const resource2 = createResourceAwareStream("FILE_HANDLE", ['a', 'b', 'c'])
  
  // Using Both strategy ensures both resources are properly cleaned up
  const processed = Stream.merge(resource1, resource2, {
    haltStrategy: StreamHaltStrategy.Both
  })
  
  return yield* Stream.runCollect(processed)
})

// This ensures all resources are properly acquired and released
// regardless of which stream finishes first
```

### Testing Strategies

Test different halt strategies to ensure correct behavior.

```typescript
import { TestClock, Duration } from "effect"

// Test halt strategy behavior
const testHaltStrategies = Effect.gen(function* () {
  const testCases = [
    { strategy: StreamHaltStrategy.Left, expected: "left-wins" },
    { strategy: StreamHaltStrategy.Right, expected: "right-wins" },
    { strategy: StreamHaltStrategy.Both, expected: "both-complete" },
    { strategy: StreamHaltStrategy.Either, expected: "first-wins" }
  ] as const
  
  for (const testCase of testCases) {
    yield* Effect.log(`Testing strategy: ${testCase.expected}`)
    
    const leftStream = Stream.fromIterable([1, 2]).pipe(
      Stream.schedule(Schedule.spaced("100 millis"))
    )
    
    const rightStream = Stream.fromIterable(['a', 'b', 'c']).pipe(
      Stream.schedule(Schedule.spaced("150 millis"))
    )
    
    const merged = Stream.merge(leftStream, rightStream, {
      haltStrategy: testCase.strategy
    })
    
    const result = yield* Stream.runCollect(merged)
    
    yield* Effect.log(`Strategy ${testCase.expected} result:`, result)
  }
})

// Run with TestClock for deterministic timing
const runStrategyTests = testHaltStrategies.pipe(
  Effect.provide(TestClock.make())
)
```

## Conclusion

StreamHaltStrategy provides explicit control over stream termination behavior, enabling predictable resource management and graceful shutdown handling in concurrent stream operations.

Key benefits:
- **Predictable Termination** - Explicit control over when merged streams should halt
- **Resource Safety** - Proper cleanup and resource management based on termination strategy  
- **Composable Design** - Strategies can be composed and configured for complex scenarios
- **Error Recovery** - Graceful degradation and fallback strategies for robust applications

StreamHaltStrategy is essential when building resilient streaming applications that need precise control over concurrent stream lifecycles, resource management, and graceful shutdown behavior.