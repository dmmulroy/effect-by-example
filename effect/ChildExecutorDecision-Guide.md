# ChildExecutorDecision: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem ChildExecutorDecision Solves

When working with concurrent stream processing and channel operations in Effect, you often need fine-grained control over how child executors behave during merge operations. Traditional approaches lead to rigid execution models that can't adapt to different workloads:

```typescript
// Traditional approach - fixed behavior
const mergeStreams = Stream.mergeAll(streams, { concurrency: 4 })
// No control over when to continue, yield, or close individual streams
// All streams follow the same execution pattern
// Can lead to resource contention or inefficient scheduling
```

This approach leads to:
- **Inflexible Scheduling** - No control over when child streams should yield execution
- **Resource Waste** - Cannot close streams early when conditions are met
- **Poor Load Balancing** - No way to dynamically adjust execution priority
- **Limited Customization** - Cannot implement custom merge strategies

### The ChildExecutorDecision Solution

ChildExecutorDecision provides granular control over child executor behavior in concurrent channel operations, enabling dynamic scheduling decisions based on runtime conditions.

```typescript
import { ChildExecutorDecision, Channel, Option } from "effect"

// Fine-grained control over child executor behavior
const customMergeLogic = (emittedValue: unknown): ChildExecutorDecision.ChildExecutorDecision => {
  if (shouldContinueProcessing(emittedValue)) {
    return ChildExecutorDecision.Continue()
  }
  if (shouldYieldToOthers(emittedValue)) {
    return ChildExecutorDecision.Yield()
  }
  return ChildExecutorDecision.Close(emittedValue)
}
```

### Key Concepts

**Continue**: Instructs the child executor to keep processing the current substream without yielding execution to other concurrent streams.

**Yield**: Tells the child executor to pause processing and allow other concurrent streams to execute, implementing cooperative multitasking.

**Close**: Terminates the current substream with a final value and transfers execution to the next available substream.

## Basic Usage Patterns

### Pattern 1: Basic Decision Creation

```typescript
import { ChildExecutorDecision } from "effect"

// Create basic decisions
const continueDecision = ChildExecutorDecision.Continue()
const yieldDecision = ChildExecutorDecision.Yield()
const closeDecision = ChildExecutorDecision.Close("final-value")

// Type guards for decision checking
const decision: ChildExecutorDecision.ChildExecutorDecision = ChildExecutorDecision.Continue()

if (ChildExecutorDecision.isContinue(decision)) {
  console.log("Decision is to continue processing")
}

if (ChildExecutorDecision.isYield(decision)) {
  console.log("Decision is to yield execution")
}

if (ChildExecutorDecision.isClose(decision)) {
  console.log("Decision is to close with value:", decision.value)
}
```

### Pattern 2: Pattern Matching on Decisions

```typescript
import { ChildExecutorDecision } from "effect"

const handleDecision = (decision: ChildExecutorDecision.ChildExecutorDecision): string => {
  return ChildExecutorDecision.match(decision, {
    onContinue: () => "Continue processing current stream",
    onYield: () => "Yield to allow other streams to process",
    onClose: (value) => `Close stream with final value: ${value}`
  })
}

// Usage
const decision = ChildExecutorDecision.Yield()
const result = handleDecision(decision) // "Yield to allow other streams to process"
```

### Pattern 3: Conditional Decision Logic

```typescript
import { ChildExecutorDecision, Option } from "effect"

const makeDecisionBasedOnCondition = (
  condition: Option.Option<boolean>
): ChildExecutorDecision.ChildExecutorDecision => {
  return Option.match(condition, {
    onNone: () => ChildExecutorDecision.Yield(),
    onSome: (hasData) => hasData 
      ? ChildExecutorDecision.Continue()
      : ChildExecutorDecision.Close("no-data")
  })
}
```

## Real-World Examples

### Example 1: Load-Balanced Stream Processing

This example shows how to implement intelligent load balancing for concurrent stream processing based on system load.

```typescript
import { 
  ChildExecutorDecision, 
  Channel, 
  Effect, 
  Stream, 
  Ref, 
  Schedule,
  pipe 
} from "effect"

interface ProcessingMetrics {
  readonly processed: number
  readonly errors: number
  readonly load: number
}

const createLoadBalancedProcessor = Effect.gen(function* () {
  const metrics = yield* Ref.make<ProcessingMetrics>({
    processed: 0,
    errors: 0,
    load: 0.0
  })

  const updateLoad = Effect.gen(function* () {
    const currentMetrics = yield* Ref.get(metrics)
    const newLoad = calculateSystemLoad(currentMetrics)
    yield* Ref.update(metrics, m => ({ ...m, load: newLoad }))
  })

  const makeProcessingDecision = (item: unknown): Effect.Effect<ChildExecutorDecision.ChildExecutorDecision> => {
    return Effect.gen(function* () {
      const currentMetrics = yield* Ref.get(metrics)
      
      // High load - yield to other streams
      if (currentMetrics.load > 0.8) {
        return ChildExecutorDecision.Yield()
      }
      
      // Error threshold reached - close this stream
      if (currentMetrics.errors > 10) {
        return ChildExecutorDecision.Close(`Too many errors: ${currentMetrics.errors}`)
      }
      
      // Normal processing - continue
      yield* Ref.update(metrics, m => ({ ...m, processed: m.processed + 1 }))
      return ChildExecutorDecision.Continue()
    })
  }

  const processStream = (items: ReadonlyArray<unknown>) => {
    return Stream.fromIterable(items).pipe(
      Stream.mapEffect(processItem),
      Stream.tap(() => updateLoad)
    )
  }

  return { processStream, makeProcessingDecision, metrics }
})

const calculateSystemLoad = (metrics: ProcessingMetrics): number => {
  // Simulate load calculation based on processed items and errors
  return Math.min(1.0, (metrics.processed * 0.01) + (metrics.errors * 0.1))
}

const processItem = (item: unknown): Effect.Effect<string> => {
  return Effect.gen(function* () {
    // Simulate processing with potential errors
    if (Math.random() < 0.1) {
      yield* Effect.fail(`Processing failed for item: ${item}`)
    }
    return `Processed: ${item}`
  })
}
```

### Example 2: Priority-Based Stream Merging

This example demonstrates how to use ChildExecutorDecision to implement priority-based processing where high-priority streams get more execution time.

```typescript
import { 
  ChildExecutorDecision, 
  Channel, 
  Effect, 
  Stream, 
  Queue, 
  Ref,
  pipe 
} from "effect"

interface PriorityItem {
  readonly data: unknown
  readonly priority: number // 1-10, 10 being highest
  readonly streamId: string
}

const createPriorityProcessor = Effect.gen(function* () {
  const streamPriorities = yield* Ref.make<Map<string, number>>(new Map())
  const executionCounts = yield* Ref.make<Map<string, number>>(new Map())

  const updateStreamPriority = (streamId: string, priority: number) => {
    return Ref.update(streamPriorities, map => new Map(map).set(streamId, priority))
  }

  const makePriorityDecision = (
    item: PriorityItem
  ): Effect.Effect<ChildExecutorDecision.ChildExecutorDecision> => {
    return Effect.gen(function* () {
      const priorities = yield* Ref.get(streamPriorities)
      const counts = yield* Ref.get(executionCounts)
      
      const currentPriority = priorities.get(item.streamId) ?? 1
      const currentCount = counts.get(item.streamId) ?? 0
      
      // Update execution count
      yield* Ref.update(executionCounts, map => 
        new Map(map).set(item.streamId, currentCount + 1)
      )

      // High priority items get more continuous processing
      if (currentPriority >= 8) {
        return ChildExecutorDecision.Continue()
      }

      // Medium priority - yield after processing several items
      if (currentPriority >= 5 && currentCount % 3 === 0) {
        return ChildExecutorDecision.Yield()
      }

      // Low priority - yield frequently to allow others
      if (currentPriority < 5 && currentCount % 2 === 0) {
        return ChildExecutorDecision.Yield()
      }

      // Special condition - close if item indicates completion
      if (isCompletionMarker(item.data)) {
        return ChildExecutorDecision.Close(`Stream ${item.streamId} completed`)
      }

      return ChildExecutorDecision.Continue()
    })
  }

  const createPriorityStream = (
    items: ReadonlyArray<PriorityItem>
  ): Stream.Stream<PriorityItem> => {
    return Stream.fromIterable(items).pipe(
      Stream.tap(item => updateStreamPriority(item.streamId, item.priority))
    )
  }

  return { 
    createPriorityStream, 
    makePriorityDecision, 
    updateStreamPriority,
    streamPriorities,
    executionCounts 
  }
})

const isCompletionMarker = (data: unknown): boolean => {
  return typeof data === 'string' && data.includes('COMPLETE')
}

// Usage example
const runPriorityProcessing = Effect.gen(function* () {
  const processor = yield* createPriorityProcessor

  const highPriorityItems: ReadonlyArray<PriorityItem> = [
    { data: "urgent-task-1", priority: 9, streamId: "urgent" },
    { data: "urgent-task-2", priority: 9, streamId: "urgent" },
    { data: "COMPLETE", priority: 9, streamId: "urgent" }
  ]

  const lowPriorityItems: ReadonlyArray<PriorityItem> = [
    { data: "background-task-1", priority: 3, streamId: "background" },
    { data: "background-task-2", priority: 3, streamId: "background" },
    { data: "background-task-3", priority: 3, streamId: "background" }
  ]

  const urgentStream = processor.createPriorityStream(highPriorityItems)
  const backgroundStream = processor.createPriorityStream(lowPriorityItems)

  return Stream.mergeAll([urgentStream, backgroundStream], { 
    concurrency: 2,
    bufferSize: 10 
  }).pipe(
    Stream.runCollect
  )
})
```

### Example 3: Resource-Aware Stream Processing

This example shows how to use ChildExecutorDecision to manage resource consumption and implement backpressure in stream processing.

```typescript
import { 
  ChildExecutorDecision, 
  Channel, 
  Effect, 
  Stream, 
  Ref, 
  Clock,
  Schedule,
  pipe 
} from "effect"

interface ResourceManager {
  readonly memoryUsage: Ref.Ref<number>
  readonly cpuUsage: Ref.Ref<number>
  readonly activeConnections: Ref.Ref<number>
  readonly maxMemory: number
  readonly maxCpu: number
  readonly maxConnections: number
}

const createResourceManager = (
  maxMemory: number = 1000,
  maxCpu: number = 80,
  maxConnections: number = 100
): Effect.Effect<ResourceManager> => {
  return Effect.gen(function* () {
    const memoryUsage = yield* Ref.make(0)
    const cpuUsage = yield* Ref.make(0)
    const activeConnections = yield* Ref.make(0)

    return {
      memoryUsage,
      cpuUsage,
      activeConnections,
      maxMemory,
      maxCpu,
      maxConnections
    }
  })
}

const makeResourceAwareDecision = (
  resourceManager: ResourceManager,
  itemSize: number
): Effect.Effect<ChildExecutorDecision.ChildExecutorDecision> => {
  return Effect.gen(function* () {
    const memory = yield* Ref.get(resourceManager.memoryUsage)
    const cpu = yield* Ref.get(resourceManager.cpuUsage)
    const connections = yield* Ref.get(resourceManager.activeConnections)

    // Critical resource usage - close stream to prevent system overload
    if (memory > resourceManager.maxMemory * 0.95) {
      return ChildExecutorDecision.Close("Memory limit exceeded")
    }

    if (cpu > resourceManager.maxCpu * 0.95) {
      return ChildExecutorDecision.Close("CPU limit exceeded")
    }

    if (connections > resourceManager.maxConnections * 0.95) {
      return ChildExecutorDecision.Close("Connection limit exceeded")
    }

    // High resource usage - yield to reduce load
    if (memory > resourceManager.maxMemory * 0.8 || 
        cpu > resourceManager.maxCpu * 0.8 ||
        connections > resourceManager.maxConnections * 0.8) {
      return ChildExecutorDecision.Yield()
    }

    // Normal usage - continue processing
    yield* Ref.update(resourceManager.memoryUsage, m => m + itemSize)
    yield* Ref.update(resourceManager.activeConnections, c => c + 1)
    
    return ChildExecutorDecision.Continue()
  })
}

const simulateResourceUsage = (
  resourceManager: ResourceManager
): Effect.Effect<void> => {
  return Effect.gen(function* () {
    // Simulate fluctuating CPU usage
    const newCpu = Math.random() * 100
    yield* Ref.set(resourceManager.cpuUsage, newCpu)

    // Simulate memory cleanup
    yield* Ref.update(resourceManager.memoryUsage, m => Math.max(0, m - 10))
    
    // Simulate connection cleanup
    yield* Ref.update(resourceManager.activeConnections, c => Math.max(0, c - 1))
  })
}

// Usage with concurrent streams
const runResourceAwareProcessing = Effect.gen(function* () {
  const resourceManager = yield* createResourceManager(500, 70, 50)

  const createManagedStream = (items: ReadonlyArray<unknown>, itemSize: number) => {
    return Stream.fromIterable(items).pipe(
      Stream.mapEffect(item => 
        Effect.gen(function* () {
          const decision = yield* makeResourceAwareDecision(resourceManager, itemSize)
          yield* simulateResourceUsage(resourceManager)
          return { item, decision }
        })
      )
    )
  }

  const largeItemStream = createManagedStream(
    ["large-file-1.zip", "large-file-2.zip", "large-file-3.zip"], 
    100
  )
  
  const smallItemStream = createManagedStream(
    ["small-file-1.txt", "small-file-2.txt", "small-file-3.txt"], 
    10
  )

  return Stream.mergeAll([largeItemStream, smallItemStream], { 
    concurrency: 2 
  }).pipe(
    Stream.take(10),
    Stream.runCollect
  )
})
```

## Advanced Features Deep Dive

### Feature 1: Pattern Matching with Complex Logic

Pattern matching allows you to handle different decision types with sophisticated branching logic.

#### Basic Pattern Matching Usage

```typescript
import { ChildExecutorDecision, pipe } from "effect"

const analyzeDecision = (decision: ChildExecutorDecision.ChildExecutorDecision) => {
  return pipe(
    decision,
    ChildExecutorDecision.match({
      onContinue: () => ({
        action: "continue",
        shouldLog: true,
        nextStep: "process-next-item"
      }),
      onYield: () => ({
        action: "yield",
        shouldLog: false,
        nextStep: "schedule-later"
      }),
      onClose: (value) => ({
        action: "close",
        shouldLog: true,
        nextStep: "cleanup",
        finalValue: value
      })
    })
  )
}
```

#### Real-World Pattern Matching Example

```typescript
import { ChildExecutorDecision, Effect, Logger, Metric, pipe } from "effect"

interface DecisionMetrics {
  readonly continueCount: Metric.Counter
  readonly yieldCount: Metric.Counter
  readonly closeCount: Metric.Counter
}

const createDecisionHandler = (metrics: DecisionMetrics) => {
  return (decision: ChildExecutorDecision.ChildExecutorDecision) => {
    return pipe(
      decision,
      ChildExecutorDecision.match({
        onContinue: () => Effect.gen(function* () {
          yield* Metric.increment(metrics.continueCount)
          yield* Logger.info("Stream continuing execution")
          return "CONTINUE"
        }),
        onYield: () => Effect.gen(function* () {
          yield* Metric.increment(metrics.yieldCount)
          yield* Logger.debug("Stream yielding execution")
          return "YIELD"
        }),
        onClose: (value) => Effect.gen(function* () {
          yield* Metric.increment(metrics.closeCount)
          yield* Logger.info(`Stream closing with value: ${value}`)
          return `CLOSE:${value}`
        })
      })
    )
  }
}
```

#### Advanced Pattern: Decision Composition

```typescript
import { ChildExecutorDecision, Effect, Array as Arr, pipe } from "effect"

const combineDecisions = (
  decisions: ReadonlyArray<ChildExecutorDecision.ChildExecutorDecision>
): ChildExecutorDecision.ChildExecutorDecision => {
  const hasContinue = decisions.some(ChildExecutorDecision.isContinue)
  const hasClose = decisions.some(ChildExecutorDecision.isClose)
  const hasYield = decisions.some(ChildExecutorDecision.isYield)

  // Priority: Close > Continue > Yield
  if (hasClose) {
    const closeDecision = decisions.find(ChildExecutorDecision.isClose)!
    return ChildExecutorDecision.Close(closeDecision.value)
  }

  if (hasContinue) {
    return ChildExecutorDecision.Continue()
  }

  return ChildExecutorDecision.Yield()
}

const applyDecisionPolicy = (
  decisions: ReadonlyArray<ChildExecutorDecision.ChildExecutorDecision>,
  policy: "majority" | "priority" | "unanimous"
): ChildExecutorDecision.ChildExecutorDecision => {
  switch (policy) {
    case "majority":
      return getMajorityDecision(decisions)
    case "priority":
      return combineDecisions(decisions)
    case "unanimous":
      return getUnanimousDecision(decisions)
  }
}

const getMajorityDecision = (
  decisions: ReadonlyArray<ChildExecutorDecision.ChildExecutorDecision>
): ChildExecutorDecision.ChildExecutorDecision => {
  const counts = {
    continue: decisions.filter(ChildExecutorDecision.isContinue).length,
    yield: decisions.filter(ChildExecutorDecision.isYield).length,
    close: decisions.filter(ChildExecutorDecision.isClose).length
  }

  const max = Math.max(counts.continue, counts.yield, counts.close)
  
  if (counts.close === max) {
    const closeDecision = decisions.find(ChildExecutorDecision.isClose)!
    return ChildExecutorDecision.Close(closeDecision.value)
  }
  
  if (counts.continue === max) {
    return ChildExecutorDecision.Continue()
  }
  
  return ChildExecutorDecision.Yield()
}

const getUnanimousDecision = (
  decisions: ReadonlyArray<ChildExecutorDecision.ChildExecutorDecision>
): ChildExecutorDecision.ChildExecutorDecision => {
  if (decisions.length === 0) {
    return ChildExecutorDecision.Yield()
  }

  const first = decisions[0]
  const allSame = decisions.every(decision => {
    if (ChildExecutorDecision.isContinue(first) && ChildExecutorDecision.isContinue(decision)) {
      return true
    }
    if (ChildExecutorDecision.isYield(first) && ChildExecutorDecision.isYield(decision)) {
      return true
    }
    if (ChildExecutorDecision.isClose(first) && ChildExecutorDecision.isClose(decision)) {
      return decision.value === first.value
    }
    return false
  })

  return allSame ? first : ChildExecutorDecision.Yield()
}
```

### Feature 2: Dynamic Decision Making with State

Implement stateful decision making that adapts based on historical execution patterns.

#### Stateful Decision Maker

```typescript
import { ChildExecutorDecision, Effect, Ref, Array as Arr, pipe } from "effect"

interface DecisionHistory {
  readonly decisions: ReadonlyArray<ChildExecutorDecision.ChildExecutorDecision>
  readonly timestamps: ReadonlyArray<number>
  readonly maxHistory: number
}

const createStatefulDecisionMaker = Effect.gen(function* () {
  const history = yield* Ref.make<DecisionHistory>({
    decisions: [],
    timestamps: [],
    maxHistory: 100
  })

  const recordDecision = (decision: ChildExecutorDecision.ChildExecutorDecision) => {
    return Effect.gen(function* () {
      const now = Date.now()
      yield* Ref.update(history, h => ({
        ...h,
        decisions: Arr.append(h.decisions, decision).slice(-h.maxHistory),
        timestamps: Arr.append(h.timestamps, now).slice(-h.maxHistory)
      }))
    })
  }

  const makeAdaptiveDecision = (
    currentCondition: unknown
  ): Effect.Effect<ChildExecutorDecision.ChildExecutorDecision> => {
    return Effect.gen(function* () {
      const h = yield* Ref.get(history)
      
      const recentDecisions = h.decisions.slice(-10)
      const yieldRatio = recentDecisions.filter(ChildExecutorDecision.isYield).length / recentDecisions.length
      const continueRatio = recentDecisions.filter(ChildExecutorDecision.isContinue).length / recentDecisions.length

      let decision: ChildExecutorDecision.ChildExecutorDecision

      // Adaptive logic based on recent patterns
      if (yieldRatio > 0.7) {
        // Too much yielding, try to continue more
        decision = ChildExecutorDecision.Continue()
      } else if (continueRatio > 0.8) {
        // Too much continuing, balance with yielding
        decision = ChildExecutorDecision.Yield()
      } else {
        // Balanced approach based on current condition
        decision = makeBasicDecision(currentCondition)
      }

      yield* recordDecision(decision)
      return decision
    })
  }

  const getDecisionStats = Effect.gen(function* () {
    const h = yield* Ref.get(history)
    const total = h.decisions.length
    
    if (total === 0) {
      return { continue: 0, yield: 0, close: 0, total: 0 }
    }

    return {
      continue: h.decisions.filter(ChildExecutorDecision.isContinue).length / total,
      yield: h.decisions.filter(ChildExecutorDecision.isYield).length / total,
      close: h.decisions.filter(ChildExecutorDecision.isClose).length / total,
      total
    }
  })

  return { makeAdaptiveDecision, getDecisionStats, recordDecision }
})

const makeBasicDecision = (condition: unknown): ChildExecutorDecision.ChildExecutorDecision => {
  if (typeof condition === 'string' && condition.includes('error')) {
    return ChildExecutorDecision.Close(condition)
  }
  if (typeof condition === 'number' && condition > 0.5) {
    return ChildExecutorDecision.Continue()
  }
  return ChildExecutorDecision.Yield()
}
```

## Practical Patterns & Best Practices

### Pattern 1: Decision Factory Pattern

Create reusable decision factories for common scenarios.

```typescript
import { ChildExecutorDecision, Effect, pipe } from "effect"

interface DecisionFactoryConfig {
  readonly continueThreshold: number
  readonly yieldThreshold: number
  readonly closeOnError: boolean
  readonly maxProcessingTime: number
}

const createDecisionFactory = (config: DecisionFactoryConfig) => {
  return {
    forLoad: (currentLoad: number): ChildExecutorDecision.ChildExecutorDecision => {
      if (currentLoad > config.continueThreshold) {
        return ChildExecutorDecision.Yield()
      }
      return ChildExecutorDecision.Continue()
    },

    forError: (error: unknown): ChildExecutorDecision.ChildExecutorDecision => {
      if (config.closeOnError) {
        return ChildExecutorDecision.Close(`Error: ${error}`)
      }
      return ChildExecutorDecision.Yield()
    },

    forTime: (processingTime: number): ChildExecutorDecision.ChildExecutorDecision => {
      if (processingTime > config.maxProcessingTime) {
        return ChildExecutorDecision.Close(`Timeout after ${processingTime}ms`)
      }
      if (processingTime > config.maxProcessingTime * 0.8) {
        return ChildExecutorDecision.Yield()
      }
      return ChildExecutorDecision.Continue()
    },

    composite: (
      load: number, 
      error: unknown | null, 
      time: number
    ): ChildExecutorDecision.ChildExecutorDecision => {
      if (error && config.closeOnError) {
        return ChildExecutorDecision.Close(`Error: ${error}`)
      }
      if (time > config.maxProcessingTime) {
        return ChildExecutorDecision.Close(`Timeout after ${time}ms`)
      }
      if (load > config.continueThreshold || time > config.maxProcessingTime * 0.8) {
        return ChildExecutorDecision.Yield()
      }
      return ChildExecutorDecision.Continue()
    }
  }
}

// Usage
const factory = createDecisionFactory({
  continueThreshold: 0.7,
  yieldThreshold: 0.5,
  closeOnError: true,
  maxProcessingTime: 5000
})

const decision = factory.composite(0.8, null, 1000) // Returns Yield
```

### Pattern 2: Decision Chain Pattern

Chain multiple decision criteria together for complex logic.

```typescript
import { ChildExecutorDecision, Effect, Option, pipe } from "effect"

type DecisionCriteria<T> = (input: T) => Option.Option<ChildExecutorDecision.ChildExecutorDecision>

const createDecisionChain = <T>(...criteria: ReadonlyArray<DecisionCriteria<T>>) => {
  return (input: T): ChildExecutorDecision.ChildExecutorDecision => {
    for (const criterion of criteria) {
      const decision = criterion(input)
      if (Option.isSome(decision)) {
        return decision.value
      }
    }
    // Default decision if no criteria match
    return ChildExecutorDecision.Continue()
  }
}

// Example criteria
const errorCriterion: DecisionCriteria<{ error?: string }> = (input) => {
  return input.error 
    ? Option.some(ChildExecutorDecision.Close(`Error: ${input.error}`))
    : Option.none()
}

const loadCriterion: DecisionCriteria<{ load: number }> = (input) => {
  return input.load > 0.8
    ? Option.some(ChildExecutorDecision.Yield())
    : Option.none()
}

const timeCriterion: DecisionCriteria<{ processingTime: number }> = (input) => {
  return input.processingTime > 10000
    ? Option.some(ChildExecutorDecision.Close(`Timeout: ${input.processingTime}ms`))
    : Option.none()
}

// Usage
const decisionChain = createDecisionChain(
  errorCriterion,
  timeCriterion,
  loadCriterion
)

const decision = decisionChain({
  error: undefined,
  load: 0.9,
  processingTime: 5000
}) // Returns Yield due to high load
```

## Integration Examples

### Integration with Channel Custom Operations

```typescript
import { 
  ChildExecutorDecision, 
  Channel, 
  Effect, 
  Stream, 
  UpstreamPullStrategy,
  UpstreamPullRequest,
  Option,
  pipe 
} from "effect"

const createCustomMergeChannel = <A>(
  decisionMaker: (item: A) => ChildExecutorDecision.ChildExecutorDecision
) => {
  return <InElem, OutErr, InErr, OutDone, InDone, Env>(
    source: Channel.Channel<A, InElem, OutErr, InErr, OutDone, InDone, Env>
  ) => {
    return Channel.concatMapWithCustom(
      source,
      (item: A) => Channel.write(item),
      (done1: OutDone, done2: OutDone) => done1, // combineInners
      (done1: OutDone, done2: OutDone) => done1, // combineAll
      (pullRequest: UpstreamPullRequest.UpstreamPullRequest<A>) => 
        UpstreamPullRequest.match(pullRequest, {
          onPulled: () => UpstreamPullStrategy.PullAfterNext(Option.none()),
          onNoUpstream: () => UpstreamPullStrategy.PullAfterAllEnqueued(Option.none())
        }),
      decisionMaker // This is where ChildExecutorDecision is used
    )
  }
}

// Usage with Stream
const createSmartMergeStream = <A>(
  streams: ReadonlyArray<Stream.Stream<A>>,
  decisionMaker: (item: A) => ChildExecutorDecision.ChildExecutorDecision
) => {
  const channels = streams.map(stream => Stream.toChannel(stream))
  const mergedChannel = channels.reduce((acc, channel) => 
    Channel.mergeWith(acc, channel, {
      onSelfDone: (exit) => Channel.fromExit(exit),
      onOtherDone: (exit) => Channel.fromExit(exit)
    })
  )
  
  return Channel.toStream(createCustomMergeChannel(decisionMaker)(mergedChannel))
}
```

### Testing Strategies

```typescript
import { 
  ChildExecutorDecision, 
  Effect, 
  TestContext, 
  it, 
  expect,
  pipe 
} from "effect"

const testDecisionMaking = it.effect("should make correct decisions based on input", () =>
  Effect.gen(function* () {
    // Test Continue decision
    const continueDecision = ChildExecutorDecision.Continue()
    expect(ChildExecutorDecision.isContinue(continueDecision)).toBe(true)
    expect(ChildExecutorDecision.isYield(continueDecision)).toBe(false)
    expect(ChildExecutorDecision.isClose(continueDecision)).toBe(false)

    // Test Yield decision
    const yieldDecision = ChildExecutorDecision.Yield()
    expect(ChildExecutorDecision.isYield(yieldDecision)).toBe(true)
    expect(ChildExecutorDecision.isContinue(yieldDecision)).toBe(false)
    expect(ChildExecutorDecision.isClose(yieldDecision)).toBe(false)

    // Test Close decision
    const closeDecision = ChildExecutorDecision.Close("test-value")
    expect(ChildExecutorDecision.isClose(closeDecision)).toBe(true)
    expect(ChildExecutorDecision.isContinue(closeDecision)).toBe(false)
    expect(ChildExecutorDecision.isYield(closeDecision)).toBe(false)

    // Test pattern matching
    const matchResult = ChildExecutorDecision.match(closeDecision, {
      onContinue: () => "continue",
      onYield: () => "yield", 
      onClose: (value) => `close-${value}`
    })
    expect(matchResult).toBe("close-test-value")
  })
)

// Mock decision maker for testing
const createMockDecisionMaker = (decisions: ReadonlyArray<ChildExecutorDecision.ChildExecutorDecision>) => {
  let index = 0
  return (_item: unknown): ChildExecutorDecision.ChildExecutorDecision => {
    const decision = decisions[index % decisions.length]
    index++
    return decision
  }
}

const testDecisionSequence = it.effect("should handle decision sequences correctly", () =>
  Effect.gen(function* () {
    const decisions = [
      ChildExecutorDecision.Continue(),
      ChildExecutorDecision.Yield(),
      ChildExecutorDecision.Close("done")
    ]
    
    const mockDecisionMaker = createMockDecisionMaker(decisions)
    
    const results = [
      mockDecisionMaker("item1"),
      mockDecisionMaker("item2"), 
      mockDecisionMaker("item3")
    ]

    expect(ChildExecutorDecision.isContinue(results[0])).toBe(true)
    expect(ChildExecutorDecision.isYield(results[1])).toBe(true)
    expect(ChildExecutorDecision.isClose(results[2])).toBe(true)
  })
)

// Property-based testing with fast-check
import { Arbitrary, fc } from "effect"

const ChildExecutorDecisionArbitrary = fc.oneof(
  fc.constant(ChildExecutorDecision.Continue()),
  fc.constant(ChildExecutorDecision.Yield()),
  fc.string().map(s => ChildExecutorDecision.Close(s))
)

const testDecisionProperties = it.effect("decision pattern matching should be exhaustive", () =>
  Effect.gen(function* () {
    yield* Arbitrary.check(ChildExecutorDecisionArbitrary, (decision) => {
      const result = ChildExecutorDecision.match(decision, {
        onContinue: () => "continue",
        onYield: () => "yield",
        onClose: (value) => `close-${value}`
      })
      
      // Should always return a string result
      return typeof result === 'string' && result.length > 0
    })
  })
)
```

## Conclusion

ChildExecutorDecision provides fine-grained control over concurrent stream execution, enabling adaptive load balancing, priority-based processing, and resource-aware scheduling for Effect applications.

Key benefits:
- **Granular Control**: Make precise decisions about stream execution flow at the individual item level
- **Dynamic Adaptation**: Adjust processing behavior based on runtime conditions, load, and system state
- **Resource Management**: Implement sophisticated backpressure and resource-aware processing strategies
- **Composable Logic**: Chain and combine decision criteria for complex, maintainable execution policies

ChildExecutorDecision is essential when building high-performance concurrent applications that need to dynamically balance throughput, resource usage, and responsiveness based on changing conditions.