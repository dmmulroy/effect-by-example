# MergeDecision: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem MergeDecision Solves

When working with concurrent streams or channels in traditional approaches, deciding what happens when one stream completes before another is often hardcoded and inflexible. This leads to rigid merging behavior that doesn't adapt to your specific use cases.

```typescript
// Traditional approach - hardcoded merge behavior
const combineStreams = async (stream1, stream2) => {
  // What happens when stream1 finishes first?
  // What if stream2 has an error?
  // How do we customize the merge behavior?
  
  // Typically results in:
  // - Fixed "wait for both" or "first wins" behavior
  // - No way to customize based on completion results
  // - Error handling mixed with merge logic
  // - Different implementations for different scenarios
}
```

This approach leads to:
- **Inflexible Merging** - Cannot adapt behavior based on how streams complete
- **Mixed Concerns** - Error handling and merge logic are tangled together
- **Code Duplication** - Different merge scenarios require separate implementations
- **Poor Composability** - Hard to reuse merge strategies across different contexts

### The MergeDecision Solution

MergeDecision provides a way to declaratively specify what should happen when streams or channels complete during a merge operation, separating the merge strategy from the merge implementation.

```typescript
import { MergeDecision, Channel, Effect, Exit } from "effect"

// Clean, declarative merge decisions
const waitForBoth = MergeDecision.Await((exit: Exit.Exit<string, Error>) => 
  Exit.isSuccess(exit) ? Effect.succeed(exit.value) : Effect.fail(exit.cause)
)

const takeFirst = MergeDecision.Done(Effect.succeed("first-wins"))

const customDecision = MergeDecision.Await((exit) => 
  Effect.gen(function* () {
    if (Exit.isSuccess(exit)) {
      const result = yield* processResult(exit.value)
      return yield* continueOrStop(result)
    }
    return yield* handleError(exit.cause)
  })
)
```

### Key Concepts

**Done Decision**: Immediately complete the merge with a specific effect - no waiting for the other stream

**Await Decision**: Wait for the other stream to complete, then process both results with a custom function

**AwaitConst Decision**: Wait for the other stream but ignore its result, returning a constant effect

## Basic Usage Patterns

### Pattern 1: Immediate Completion (Done)

```typescript
import { MergeDecision, Effect, Channel, Exit } from "effect"

// Complete immediately when one stream finishes
const immediateCompletion = <A>(result: A) =>
  MergeDecision.Done(Effect.succeed(result))

// Example: First stream to complete wins
const firstWins = Channel.mergeWith(leftChannel, {
  other: rightChannel,
  onSelfDone: (exit) => MergeDecision.Done(Effect.succeed("left-won")),
  onOtherDone: (exit) => MergeDecision.Done(Effect.succeed("right-won"))
})
```

### Pattern 2: Wait and Process (Await)

```typescript
// Wait for the other stream and combine results
const waitAndCombine = <A, B>(
  leftResult: A,
  combiner: (left: A, right: B) => Effect.Effect<string>
) =>
  MergeDecision.Await((rightExit: Exit.Exit<B, Error>) =>
    Exit.isSuccess(rightExit) 
      ? combiner(leftResult, rightExit.value)
      : Effect.fail(new Error("Right stream failed"))
  )

// Example: Combine results from both streams
const combineResults = Channel.mergeWith(numbersChannel, {
  other: stringsChannel,
  onSelfDone: (exit) => 
    Exit.isSuccess(exit) 
      ? MergeDecision.Await((otherExit) =>
          Exit.isSuccess(otherExit)
            ? Effect.succeed(`Numbers: ${exit.value}, Strings: ${otherExit.value}`)
            : Effect.fail(otherExit.cause)
        )
      : MergeDecision.Done(Effect.fail(exit.cause)),
  onOtherDone: (exit) =>
    Exit.isSuccess(exit)
      ? MergeDecision.Await((selfExit) =>
          Exit.isSuccess(selfExit)
            ? Effect.succeed(`Strings: ${exit.value}, Numbers: ${selfExit.value}`)
            : Effect.fail(selfExit.cause)
        )
      : MergeDecision.Done(Effect.fail(exit.cause))
})
```

### Pattern 3: Constant Awaiting (AwaitConst)

```typescript
// Wait for the other stream but use a predetermined result
const awaitWithConstant = <A>(constantResult: A) =>
  MergeDecision.AwaitConst(Effect.succeed(constantResult))

// Example: Always return "completed" regardless of the other stream's result
const alwaysCompleted = Channel.mergeWith(dataChannel, {
  other: metadataChannel,
  onSelfDone: () => MergeDecision.AwaitConst(Effect.succeed("data-completed")),
  onOtherDone: () => MergeDecision.AwaitConst(Effect.succeed("metadata-completed"))
})
```

## Real-World Examples

### Example 1: Event Stream Coordination

Coordinate multiple event streams where some events are critical and others are optional.

```typescript
import { MergeDecision, Channel, Effect, Exit, Stream } from "effect"

interface CriticalEvent {
  readonly _tag: "CriticalEvent"
  readonly id: string
  readonly data: unknown
}

interface OptionalEvent {
  readonly _tag: "OptionalEvent"
  readonly id: string
  readonly metadata: Record<string, unknown>
}

// Helper to create decision based on event criticality
const createEventDecision = (isCritical: boolean) =>
  (exit: Exit.Exit<unknown, Error>): MergeDecision.MergeDecision<never, Error, unknown, Error, string> => {
    if (isCritical) {
      // Critical events must succeed, fail entire merge if they fail
      return Exit.isSuccess(exit)
        ? MergeDecision.AwaitConst(Effect.succeed("critical-event-processed"))
        : MergeDecision.Done(Effect.fail(new Error("Critical event failed")))
    } else {
      // Optional events: continue regardless, log failures
      return MergeDecision.Await((otherExit) =>
        Effect.gen(function* () {
          if (Exit.isFailure(exit)) {
            yield* Effect.logWarning("Optional event failed", exit.cause)
          }
          if (Exit.isFailure(otherExit)) {
            yield* Effect.logWarning("Other stream failed", otherExit.cause)
          }
          return "events-processed"
        })
      )
    }
  }

// Process critical and optional event streams
const processEventStreams = (
  criticalEvents: Stream.Stream<CriticalEvent, Error>,
  optionalEvents: Stream.Stream<OptionalEvent, Error>
) => {
  const criticalChannel = Stream.toChannel(criticalEvents)
  const optionalChannel = Stream.toChannel(optionalEvents)

  return Channel.mergeWith(criticalChannel, {
    other: optionalChannel,
    onSelfDone: createEventDecision(true),    // Critical stream
    onOtherDone: createEventDecision(false)   // Optional stream
  })
}
```

### Example 2: Data Processing Pipeline

Process data streams with different priorities and error handling strategies.

```typescript
import { MergeDecision, Channel, Effect, Exit, Stream, Queue } from "effect"

interface DataBatch {
  readonly id: string
  readonly priority: "high" | "normal" | "low"
  readonly items: ReadonlyArray<unknown>
}

interface ProcessingResult {
  readonly processed: number
  readonly failed: number
  readonly duration: number
}

// Create priority-aware merge decisions
const createPriorityDecision = (priority: "high" | "normal" | "low") =>
  (exit: Exit.Exit<ProcessingResult, Error>) => {
    return Effect.gen(function* () {
      if (Exit.isFailure(exit)) {
        // High priority failures stop everything
        if (priority === "high") {
          return yield* MergeDecision.Done(Effect.fail(
            new Error("High priority processing failed")
          ))
        }
        
        // Normal/low priority failures are logged but don't stop processing
        yield* Effect.logWarning(`${priority} priority processing failed`, exit.cause)
        return yield* MergeDecision.AwaitConst(Effect.succeed({
          processed: 0,
          failed: 1,
          duration: 0
        }))
      }

      const result = exit.value
      
      // High priority always takes precedence
      if (priority === "high") {
        return yield* MergeDecision.Done(Effect.succeed(result))
      }

      // Wait for the other stream and combine results
      return yield* MergeDecision.Await((otherExit: Exit.Exit<ProcessingResult, Error>) =>
        Effect.gen(function* () {
          if (Exit.isFailure(otherExit)) {
            return result // Use our result if other failed
          }

          const otherResult = otherExit.value
          return {
            processed: result.processed + otherResult.processed,
            failed: result.failed + otherResult.failed,
            duration: Math.max(result.duration, otherResult.duration)
          }
        })
      )
    })
  }

// Process data streams with priority handling
const processDataStreams = (
  highPriorityData: Stream.Stream<DataBatch, Error>,
  normalPriorityData: Stream.Stream<DataBatch, Error>
) => {
  const processStream = (stream: Stream.Stream<DataBatch, Error>) =>
    stream.pipe(
      Stream.mapEffect((batch) =>
        Effect.gen(function* () {
          const start = yield* Effect.clock.currentTimeMillis
          const processed = yield* processBatch(batch)
          const end = yield* Effect.clock.currentTimeMillis
          
          return {
            processed: processed.length,
            failed: batch.items.length - processed.length,
            duration: end - start
          }
        })
      ),
      Stream.toChannel
    )

  return Channel.mergeWith(processStream(highPriorityData), {
    other: processStream(normalPriorityData),
    onSelfDone: createPriorityDecision("high"),
    onOtherDone: createPriorityDecision("normal")
  })
}

// Helper function for processing batches
const processBatch = (batch: DataBatch): Effect.Effect<ReadonlyArray<unknown>, Error> =>
  Effect.gen(function* () {
    // Simulate batch processing
    yield* Effect.sleep(`${batch.items.length * 10} millis`)
    
    // Simulate some failures for non-high priority
    if (batch.priority !== "high" && Math.random() < 0.1) {
      return yield* Effect.fail(new Error("Processing failed"))
    }
    
    return batch.items
  })
```

### Example 3: Resource Monitoring System

Monitor multiple resource streams with different strategies for handling completion and failures.

```typescript
import { MergeDecision, Channel, Effect, Exit, Stream, Metrics } from "effect"

interface ResourceMetrics {
  readonly resource: string
  readonly cpu: number
  readonly memory: number
  readonly timestamp: number
}

interface AlertConfig {
  readonly resource: string
  readonly thresholds: {
    readonly cpu: number
    readonly memory: number
  }
  readonly critical: boolean
}

// Create monitoring decision based on resource criticality
const createMonitoringDecision = (config: AlertConfig) =>
  (exit: Exit.Exit<ResourceMetrics, Error>) => {
    return Effect.gen(function* () {
      if (Exit.isFailure(exit)) {
        if (config.critical) {
          // Critical resource monitoring failure triggers immediate alert
          yield* Effect.logError(`Critical resource ${config.resource} monitoring failed`)
          return yield* MergeDecision.Done(Effect.fail(
            new Error(`Critical monitoring failure for ${config.resource}`)
          ))
        } else {
          // Non-critical failure: log and continue
          yield* Effect.logWarning(`Resource ${config.resource} monitoring failed`)
    
          return yield* MergeDecision.Await((otherExit) =>
            Exit.isSuccess(otherExit)
              ? Effect.succeed(otherExit.value)
              : Effect.succeed({
                  resource: "fallback",
                  cpu: 0,
                  memory: 0,
                  timestamp: Date.now()
                } as ResourceMetrics)
          )
        }
      }

      const metrics = exit.value
      
      // Check thresholds
      const cpuAlert = metrics.cpu > config.thresholds.cpu
      const memoryAlert = metrics.memory > config.thresholds.memory

      if (cpuAlert || memoryAlert) {
        yield* Effect.logWarning(
          `Resource ${config.resource} threshold exceeded`,
          { cpu: metrics.cpu, memory: metrics.memory }
        )
        
        if (config.critical && (cpuAlert || memoryAlert)) {
          // Critical resource with threshold breach
          return yield* MergeDecision.Done(Effect.succeed(metrics))
        }
      }

      // Normal case: wait for other resource and combine
      return yield* MergeDecision.Await((otherExit: Exit.Exit<ResourceMetrics, Error>) =>
        Effect.gen(function* () {
          if (Exit.isFailure(otherExit)) {
            return metrics // Use our metrics if other failed
          }

          const otherMetrics = otherExit.value
          
          // Return combined metrics
          return {
            resource: `${metrics.resource}+${otherMetrics.resource}`,
            cpu: Math.max(metrics.cpu, otherMetrics.cpu),
            memory: Math.max(metrics.memory, otherMetrics.memory),
            timestamp: Math.max(metrics.timestamp, otherMetrics.timestamp)
          }
        })
      )
    })
  }

// Monitor multiple resources with different strategies
const monitorResources = (
  primaryResource: { stream: Stream.Stream<ResourceMetrics, Error>, config: AlertConfig },
  secondaryResource: { stream: Stream.Stream<ResourceMetrics, Error>, config: AlertConfig }
) => {
  return Channel.mergeWith(Stream.toChannel(primaryResource.stream), {
    other: Stream.toChannel(secondaryResource.stream),
    onSelfDone: createMonitoringDecision(primaryResource.config),
    onOtherDone: createMonitoringDecision(secondaryResource.config)
  })
}

// Usage example
const monitoringSystem = monitorResources(
  {
    stream: Stream.fromIterable([
      { resource: "database", cpu: 85, memory: 70, timestamp: Date.now() },
      { resource: "database", cpu: 90, memory: 75, timestamp: Date.now() + 1000 }
    ]),
    config: {
      resource: "database",
      thresholds: { cpu: 80, memory: 85 },
      critical: true
    }
  },
  {
    stream: Stream.fromIterable([
      { resource: "cache", cpu: 60, memory: 40, timestamp: Date.now() },
      { resource: "cache", cpu: 65, memory: 45, timestamp: Date.now() + 1000 }
    ]),
    config: {
      resource: "cache", 
      thresholds: { cpu: 70, memory: 60 },
      critical: false
    }
  }
)
```

## Advanced Features Deep Dive

### Feature 1: Dynamic Decision Making

Create merge decisions that adapt based on runtime conditions and stream results.

#### Basic Dynamic Decision Usage

```typescript
import { MergeDecision, Effect, Exit } from "effect"

// Decision that changes behavior based on result content
const createAdaptiveDecision = <A>(
  successThreshold: number,
  onBelowThreshold: Effect.Effect<string>,
  onAboveThreshold: Effect.Effect<string>
) =>
  (exit: Exit.Exit<A, Error>) => {
    if (Exit.isFailure(exit)) {
      return MergeDecision.Done(Effect.fail(exit.cause))
    }

    const value = exit.value
    
    // Adapt decision based on the result
    if (typeof value === 'number' && value < successThreshold) {
      return MergeDecision.Done(onBelowThreshold)
    }
    
    return MergeDecision.Await((otherExit) =>
      Exit.isSuccess(otherExit) 
        ? onAboveThreshold
        : Effect.succeed("partial-success")
    )
  }
```

#### Real-World Dynamic Decision Example

```typescript
// Load balancer that adapts decisions based on response times
interface ServiceResponse {
  readonly serviceId: string
  readonly responseTime: number
  readonly payload: unknown
}

const createLoadBalancerDecision = (responseTimeThreshold: number) =>
  (exit: Exit.Exit<ServiceResponse, Error>) => {
    return Effect.gen(function* () {
      if (Exit.isFailure(exit)) {
        // Service failed, wait for backup service
        return yield* MergeDecision.Await((backupExit) =>
          Exit.isSuccess(backupExit)
            ? Effect.succeed(backupExit.value)
            : Effect.fail(new Error("All services failed"))
        )
      }

      const response = exit.value
      
      if (response.responseTime < responseTimeThreshold) {
        // Fast response, use immediately
        yield* Effect.logInfo(`Fast response from ${response.serviceId}`)
        return yield* MergeDecision.Done(Effect.succeed(response))
      } else {
        // Slow response, wait for the other service to see if it's faster
        yield* Effect.logInfo(`Slow response from ${response.serviceId}, waiting for alternative`)
        
        return yield* MergeDecision.Await((otherExit) =>
          Effect.gen(function* () {
            if (Exit.isFailure(otherExit)) {
              // Other service failed, use our slow response
              return response
            }

            const otherResponse = otherExit.value
            
            // Choose the faster response
            return otherResponse.responseTime < response.responseTime 
              ? otherResponse 
              : response
          })
        )
      }
    })
  }
```

#### Advanced Dynamic Decision: Circuit Breaker Pattern

```typescript
import { Ref } from "effect"

interface CircuitState {
  readonly failures: number
  readonly lastFailure: number
  readonly isOpen: boolean
}

const createCircuitBreakerDecision = (
  circuitState: Ref.Ref<CircuitState>,
  maxFailures: number,
  timeoutMs: number
) =>
  (exit: Exit.Exit<unknown, Error>) => {
    return Effect.gen(function* () {
      const state = yield* Ref.get(circuitState)
      const now = Date.now()

      if (Exit.isFailure(exit)) {
        // Update circuit state on failure
        const newFailures = state.failures + 1
        const shouldOpen = newFailures >= maxFailures
        
        yield* Ref.set(circuitState, {
          failures: newFailures,
          lastFailure: now,
          isOpen: shouldOpen
        })

        if (shouldOpen) {
          yield* Effect.logWarning("Circuit breaker opened due to failures")
          return yield* MergeDecision.Done(Effect.fail(
            new Error("Circuit breaker is open")
          ))
        }

        // Still under threshold, wait for backup
        return yield* MergeDecision.Await((backupExit) =>
          Exit.isSuccess(backupExit)
            ? Effect.succeed(backupExit.value)
            : Effect.fail(new Error("Both services failed"))
        )
      }

      // Success case
      if (state.isOpen && (now - state.lastFailure) > timeoutMs) {
        // Reset circuit after timeout
        yield* Ref.set(circuitState, {
          failures: 0,
          lastFailure: 0,
          isOpen: false
        })
        yield* Effect.logInfo("Circuit breaker reset")
      }

      return yield* MergeDecision.Done(Effect.succeed(exit.value))
    })
  }
```

### Feature 2: Conditional Merge Strategies

Build complex merge strategies that can switch between different behaviors based on stream characteristics.

#### Strategy Pattern with MergeDecision

```typescript
// Define merge strategies as reusable functions
const MergeStrategies = {
  // Race: first to complete wins
  race: <A, E>() => 
    (exit: Exit.Exit<A, E>): MergeDecision.MergeDecision<never, E, A, E, A> =>
      MergeDecision.Done(Effect.suspend(() => exit)),

  // WaitBoth: wait for both and combine
  waitBoth: <A, B, C>(combiner: (a: A, b: B) => C) =>
    (exit: Exit.Exit<A, Error>) =>
      Exit.isSuccess(exit)
        ? MergeDecision.Await((otherExit: Exit.Exit<B, Error>) =>
            Exit.isSuccess(otherExit)
              ? Effect.succeed(combiner(exit.value, otherExit.value))
              : Effect.fail(otherExit.cause)
          )
        : MergeDecision.Done(Effect.fail(exit.cause)),

  // Failover: use backup if primary fails
  failover: <A, E>() =>
    (exit: Exit.Exit<A, E>) =>
      Exit.isSuccess(exit)
        ? MergeDecision.Done(Effect.succeed(exit.value))
        : MergeDecision.Await((backupExit) => Effect.suspend(() => backupExit)),

  // Timeout: wait for other but only for a limited time
  timeout: <A, E>(timeoutMs: number, fallback: A) =>
    (exit: Exit.Exit<A, E>) =>
      Exit.isSuccess(exit)
        ? MergeDecision.Await((otherExit) =>
            Effect.race(
              Effect.suspend(() => otherExit),
              Effect.delay(Effect.succeed(fallback), `${timeoutMs} millis`)
            )
          )
        : MergeDecision.Done(Effect.fail(exit.cause))
}

// Usage: Create adaptive merges based on configuration
const createAdaptiveMerge = <A, B, C>(
  strategy: "race" | "waitBoth" | "failover" | "timeout",
  config?: { combiner?: (a: A, b: B) => C; timeoutMs?: number; fallback?: A }
) => {
  switch (strategy) {
    case "race":
      return MergeStrategies.race<A, Error>()
    case "waitBoth":
      return MergeStrategies.waitBoth(config?.combiner ?? ((a, b) => [a, b] as C))
    case "failover":
      return MergeStrategies.failover<A, Error>()
    case "timeout":
      return MergeStrategies.timeout(config?.timeoutMs ?? 5000, config?.fallback!)
  }
}
```

## Practical Patterns & Best Practices

### Pattern 1: Error-Resilient Merging

```typescript
// Helper for creating error-resilient merge decisions
const createResilientDecision = <A, E>(
  onSuccess: (value: A) => MergeDecision.MergeDecision<never, E, A, E, A>,
  onFailure: (cause: E) => MergeDecision.MergeDecision<never, E, A, E, A>,
  maxRetries: number = 3
) => {
  return (exit: Exit.Exit<A, E>, retryCount: number = 0): MergeDecision.MergeDecision<never, E, A, E, A> => {
    if (Exit.isSuccess(exit)) {
      return onSuccess(exit.value)
    }

    if (retryCount < maxRetries) {
      // Retry by waiting for the other stream
      return MergeDecision.Await((otherExit) =>
        Effect.gen(function* () {
          if (Exit.isSuccess(otherExit)) {
            return otherExit.value
          }
          // Both failed, but we can retry
          yield* Effect.logWarning(`Retry ${retryCount + 1}/${maxRetries}`)
          return yield* Effect.fail(otherExit.cause)
        })
      )
    }

    return onFailure(exit.cause)
  }
}

// Usage
const resilientMergeDecision = createResilientDecision(
  (value) => MergeDecision.Done(Effect.succeed(value)),
  (cause) => MergeDecision.Done(Effect.fail(new Error("All retries exhausted"))),
  3
)
```

### Pattern 2: Priority-Based Merging

```typescript
// Priority system for merge decisions
interface PriorityConfig<A> {
  readonly priority: number
  readonly processor: (value: A) => Effect.Effect<A>
  readonly onConflict: "takeHigher" | "takeLower" | "combine"
}

const createPriorityDecision = <A>(config: PriorityConfig<A>) =>
  (exit: Exit.Exit<{ value: A; priority: number }, Error>) => {
    if (Exit.isFailure(exit)) {
      return MergeDecision.Await((otherExit) => Effect.suspend(() => otherExit))
    }

    const { value, priority } = exit.value

    return MergeDecision.Await((otherExit: Exit.Exit<{ value: A; priority: number }, Error>) =>
      Effect.gen(function* () {
        if (Exit.isFailure(otherExit)) {
          return yield* config.processor(value)
        }

        const other = otherExit.value
        
        switch (config.onConflict) {
          case "takeHigher":
            return priority > other.priority ? value : other.value
          case "takeLower":
            return priority < other.priority ? value : other.value
          case "combine":
            // Custom combine logic would go here
            return yield* config.processor(value)
        }
      })
    )
  }

// Helper to wrap values with priority
const withPriority = <A>(value: A, priority: number) => ({ value, priority })
```

### Pattern 3: Resource-Aware Merging

```typescript
// Resource-aware decisions that consider system state
interface ResourceState {
  readonly memoryUsed: number
  readonly cpuUsed: number
  readonly activeConnections: number
}

const createResourceAwareDecision = <A>(
  getResourceState: Effect.Effect<ResourceState>,
  thresholds: {
    readonly memory: number
    readonly cpu: number
    readonly connections: number
  }
) =>
  (exit: Exit.Exit<A, Error>) => {
    return Effect.gen(function* () {
      const resources = yield* getResourceState

      if (Exit.isFailure(exit)) {
        // Check if we have resources to wait for recovery
        const canWaitForRecovery = 
          resources.memoryUsed < thresholds.memory * 0.8 &&
          resources.cpuUsed < thresholds.cpu * 0.8

        if (canWaitForRecovery) {
          return yield* MergeDecision.Await((otherExit) => Effect.suspend(() => otherExit))
        } else {
          return yield* MergeDecision.Done(Effect.fail(exit.cause))
        }
      }

      // Success case - check if we should wait for the other stream
      const shouldWaitForOther = 
        resources.memoryUsed < thresholds.memory &&
        resources.cpuUsed < thresholds.cpu &&
        resources.activeConnections < thresholds.connections

      if (shouldWaitForOther) {
        return yield* MergeDecision.Await((otherExit) =>
          Exit.isSuccess(otherExit)
            ? Effect.succeed(otherExit.value)
            : Effect.succeed(exit.value)
        )
      } else {
        // Resource constrained, take what we have
        yield* Effect.logInfo("Resource constrained, completing immediately")
        return yield* MergeDecision.Done(Effect.succeed(exit.value))
      }
    })
  }
```

## Integration Examples

### Integration with Stream Processing

```typescript
import { Stream, Channel, MergeDecision, Effect, Exit } from "effect"

// Stream-specific merge decision helpers
const StreamMergeDecisions = {
  // Take elements until one stream completes
  takeUntilComplete: <A>(): typeof MergeDecision.Done<A, never, never> =>
    (exit: Exit.Exit<A, Error>) => MergeDecision.Done(Effect.suspend(() => exit)),

  // Combine stream results
  combineStreamResults: <A, B>(combiner: (a: A, b: B) => Effect.Effect<A | B>) =>
    (exit: Exit.Exit<A, Error>) =>
      Exit.isSuccess(exit)
        ? MergeDecision.Await((otherExit: Exit.Exit<B, Error>) =>
            Exit.isSuccess(otherExit)
              ? combiner(exit.value, otherExit.value)
              : Effect.succeed(exit.value)
          )
        : MergeDecision.Await((otherExit) => Effect.suspend(() => otherExit)),

  // Buffer until both streams complete
  bufferUntilBoth: <A>() =>
    (exit: Exit.Exit<ReadonlyArray<A>, Error>) =>
      Exit.isSuccess(exit)
        ? MergeDecision.Await((otherExit: Exit.Exit<ReadonlyArray<A>, Error>) =>
            Exit.isSuccess(otherExit)
              ? Effect.succeed([...exit.value, ...otherExit.value])
              : Effect.succeed(exit.value)
          )
        : MergeDecision.Done(Effect.fail(exit.cause))
}

// Usage with streams
const mergeDataStreams = (
  primaryData: Stream.Stream<string, Error>,
  secondaryData: Stream.Stream<string, Error>
) => {
  return Stream.fromChannel(
    Channel.mergeWith(Stream.toChannel(primaryData), {
      other: Stream.toChannel(secondaryData),
      onSelfDone: StreamMergeDecisions.combineStreamResults(
        (primary, secondary) => Effect.succeed(`${primary}-${secondary}`)
      ),
      onOtherDone: StreamMergeDecisions.combineStreamResults(
        (secondary, primary) => Effect.succeed(`${secondary}-${primary}`)
      )
    })
  )
}
```

### Integration with HTTP Services

```typescript
import { HttpClient, HttpClientRequest, MergeDecision, Effect, Exit } from "@effect/platform"

interface ServiceEndpoint {
  readonly url: string
  readonly timeout: number
  readonly retries: number
}

// HTTP-specific merge decisions
const createHttpMergeDecision = (
  primaryEndpoint: ServiceEndpoint,
  fallbackEndpoint: ServiceEndpoint
) =>
  (exit: Exit.Exit<unknown, Error>) => {
    return Effect.gen(function* () {
      if (Exit.isSuccess(exit)) {
        // Primary succeeded, check if we should wait for fallback for comparison
        const primaryResult = exit.value
        
        return yield* MergeDecision.Await((fallbackExit) =>
          Effect.gen(function* () {
            if (Exit.isFailure(fallbackExit)) {
              // Fallback failed, use primary result
              return primaryResult
            }
            
            // Both succeeded, could implement logic to choose best result
            // For example, choose based on response time, data freshness, etc.
            yield* Effect.logInfo("Both endpoints succeeded, using primary")
            return primaryResult
          })
        )
      } else {
        // Primary failed, wait for fallback
        yield* Effect.logWarning("Primary endpoint failed, waiting for fallback")
        
        return yield* MergeDecision.Await((fallbackExit) =>
          Exit.isSuccess(fallbackExit)
            ? Effect.succeed(fallbackExit.value)
            : Effect.fail(new Error("Both endpoints failed"))
        )
      }
    })
  }

// Usage with HTTP requests
const fetchWithFallback = (
  primaryUrl: string,
  fallbackUrl: string,
  client: HttpClient.HttpClient
) => {
  const makeRequest = (url: string) =>
    HttpClientRequest.get(url).pipe(
      client,
      Effect.map(response => response.json),
      Effect.scoped,
      Stream.fromEffect,
      Stream.toChannel
    )

  return Channel.mergeWith(makeRequest(primaryUrl), {
    other: makeRequest(fallbackUrl),
    onSelfDone: createHttpMergeDecision(
      { url: primaryUrl, timeout: 5000, retries: 2 },
      { url: fallbackUrl, timeout: 3000, retries: 1 }
    ),
    onOtherDone: createHttpMergeDecision(
      { url: fallbackUrl, timeout: 3000, retries: 1 },
      { url: primaryUrl, timeout: 5000, retries: 2 }
    )
  })
}
```

### Testing Strategies

```typescript
import { describe, it, expect } from "@effect/vitest"
import { TestClock } from "effect/TestClock" 

// Testing merge decisions in isolation
describe("MergeDecision", () => {
  it.effect("should handle successful completion", () =>
    Effect.gen(function* () {
      const decision = MergeDecision.Done(Effect.succeed("test-result"))
      
      const result = yield* MergeDecision.match(decision, {
        onDone: (effect) => effect,
        onAwait: () => Effect.succeed("unexpected")
      })
      
      expect(result).toBe("test-result")
    })
  )

  it.effect("should await other stream completion", () =>
    Effect.gen(function* () {
      const decision = MergeDecision.Await((exit: Exit.Exit<string, Error>) =>
        Exit.isSuccess(exit) 
          ? Effect.succeed(`awaited: ${exit.value}`)
          : Effect.fail(exit.cause)
      )
      
      const result = yield* MergeDecision.match(decision, {
        onDone: (effect) => effect,
        onAwait: (f) => f(Exit.succeed("other-result"))
      })
      
      expect(result).toBe("awaited: other-result")
    })
  )

  it.effect("should handle timeout scenarios", () =>
    Effect.gen(function* () {
      const timeoutDecision = MergeDecision.Await((exit: Exit.Exit<string, Error>) =>
        Effect.race(
          Effect.suspend(() => exit),
          Effect.delay(Effect.succeed("timeout"), "1 second")
        )
      )
      
      // Test with TestClock for deterministic timing
      const fiber = yield* Effect.fork(
        MergeDecision.match(timeoutDecision, {
          onDone: (effect) => effect,
          onAwait: (f) => f(Exit.succeed("slow-result"))
        })
      )
      
      yield* TestClock.adjust("2 seconds")
      const result = yield* Fiber.join(fiber)
      
      expect(result).toBe("timeout")
    })
  )
})

// Testing complete merge scenarios
describe("Channel Merging with MergeDecision", () => {
  it.effect("should merge channels with custom decisions", () =>
    Effect.gen(function* () {
      const leftChannel = Channel.fromEffect(Effect.succeed("left"))
      const rightChannel = Channel.fromEffect(Effect.succeed("right"))
      
      const mergedChannel = Channel.mergeWith(leftChannel, {
        other: rightChannel,
        onSelfDone: (exit) => MergeDecision.Await((otherExit) =>
          Effect.gen(function* () {
            const leftResult = Exit.isSuccess(exit) ? exit.value : "left-failed"
            const rightResult = Exit.isSuccess(otherExit) ? otherExit.value : "right-failed"
            return `${leftResult}+${rightResult}`
          })
        ),
        onOtherDone: (exit) => MergeDecision.Await((selfExit) =>
          Effect.gen(function* () {
            const rightResult = Exit.isSuccess(exit) ? exit.value : "right-failed"
            const leftResult = Exit.isSuccess(selfExit) ? selfExit.value : "left-failed"
            return `${leftResult}+${rightResult}`
          })
        )
      })
      
      const result = yield* Channel.runDrain(mergedChannel)
      expect(result).toBe("left+right")
    })
  )
})
```

## Conclusion

MergeDecision provides **declarative merge control**, **flexible error handling**, and **composable merge strategies** for stream and channel operations in Effect.

Key benefits:
- **Separation of Concerns**: Merge strategy is separate from merge implementation
- **Flexibility**: Adapt merge behavior based on runtime conditions and results  
- **Composability**: Build complex merge scenarios from simple decision primitives

MergeDecision is essential when you need fine-grained control over how concurrent streams or channels are merged, especially in scenarios involving error recovery, priority handling, or resource-aware processing.