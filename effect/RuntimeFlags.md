# RuntimeFlags: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem RuntimeFlags Solves

In production applications, you need fine-grained control over runtime behavior without changing code. Traditional approaches hardcode execution settings or require application restarts:

```typescript
// Traditional approach - hardcoded execution behavior
class TaskProcessor {
  private interruptible = true;
  private metricsEnabled = process.env.NODE_ENV === 'production';
  private debugMode = process.env.DEBUG === 'true';

  async processTask(task: Task): Promise<Result> {
    if (this.debugMode) {
      console.log('Processing task:', task.id);
    }
    
    // No way to dynamically change interruption behavior
    // Metrics collection is always on/off per environment
    // Cannot tune performance characteristics at runtime
    
    const startTime = this.metricsEnabled ? Date.now() : 0;
    try {
      const result = await this.executeTask(task);
      if (this.metricsEnabled) {
        this.recordMetric('task.duration', Date.now() - startTime);
      }
      return result;
    } catch (error) {
      if (this.interruptible && this.shouldCancel()) {
        throw new CancellationError();
      }
      throw error;
    }
  }
}
```

This approach leads to:
- **Inflexible Runtime Behavior** - Cannot adjust execution characteristics without code changes
- **Environment Lock-in** - Settings are fixed per deployment environment
- **Poor Observability Control** - Cannot dynamically enable/disable metrics or debugging
- **Testing Difficulties** - Hard to test different execution scenarios
- **Performance Trade-offs** - Cannot optimize for different workload patterns

### The RuntimeFlags Solution

RuntimeFlags provide dynamic, composable control over Effect runtime behavior without changing application code:

```typescript
import { Effect, RuntimeFlags, RuntimeFlagsPatch } from "effect"

// Clean, declarative runtime behavior control
const processTask = (task: Task) => Effect.gen(function* () {
  const flags = yield* Effect.getRuntimeFlags
  
  // Behavior adapts based on current runtime flags
  if (RuntimeFlags.runtimeMetrics(flags)) {
    yield* Effect.logInfo(`Processing task: ${task.id}`)
  }
  
  const result = yield* executeTask(task)
  return result
}).pipe(
  // Execution characteristics can be modified per use case
  Effect.withRuntimeFlagsPatch(RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics)),
  Effect.timeout("30s") // Respects interruption flags
)
```

### Key Concepts

**RuntimeFlags**: A set of bit flags that control Effect runtime behavior - interruption, metrics, supervision, cooperative yielding, and wind-down mode.

**RuntimeFlagsPatch**: Describes changes to runtime flags that can be applied temporarily or permanently to modify execution behavior.

**Dynamic Control**: Runtime flags can be inspected and modified during execution, allowing adaptive behavior based on current conditions.

## Basic Usage Patterns

### Pattern 1: Inspecting Current Runtime Flags

```typescript
import { Effect, RuntimeFlags } from "effect"

const inspectRuntimeBehavior = Effect.gen(function* () {
  const flags = yield* Effect.getRuntimeFlags
  
  const canInterrupt = RuntimeFlags.interruptible(flags)
  const metricsEnabled = RuntimeFlags.runtimeMetrics(flags)
  const debugMode = RuntimeFlags.opSupervision(flags)
  
  yield* Effect.logInfo(`Runtime configuration:`, {
    interruptible: canInterrupt,
    metrics: metricsEnabled,
    debugging: debugMode
  })
  
  return { canInterrupt, metricsEnabled, debugMode }
})
```

### Pattern 2: Temporarily Modifying Runtime Flags

```typescript
import { Effect, RuntimeFlags, RuntimeFlagsPatch } from "effect"

const processWithMetrics = <A, E, R>(effect: Effect.Effect<A, E, R>) => {
  return effect.pipe(
    Effect.withRuntimeFlagsPatch(
      RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics)
    )
  )
}

const debuggableEffect = <A, E, R>(effect: Effect.Effect<A, E, R>) => {
  return effect.pipe(
    Effect.withRuntimeFlagsPatch(
      RuntimeFlagsPatch.enable(RuntimeFlags.OpSupervision)
    )
  )
}
```

### Pattern 3: Permanently Updating Runtime Flags

```typescript
import { Effect, RuntimeFlags, RuntimeFlagsPatch } from "effect"

const enableProductionOptimizations = Effect.gen(function* () {
  // Enable metrics collection for monitoring
  yield* Effect.patchRuntimeFlags(
    RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics)
  )
  
  // Disable detailed operation supervision for performance
  yield* Effect.patchRuntimeFlags(
    RuntimeFlagsPatch.disable(RuntimeFlags.OpSupervision)
  )
  
  yield* Effect.logInfo("Production optimizations enabled")
})
```

## Real-World Examples

### Example 1: Adaptive HTTP Request Processing

HTTP services need different runtime characteristics based on request type and system load:

```typescript
import { Effect, RuntimeFlags, RuntimeFlagsPatch, Duration } from "effect"
import { HttpApi, HttpServer } from "@effect/platform"

// Domain models
interface ApiRequest {
  readonly id: string
  readonly type: 'critical' | 'background' | 'batch'
  readonly payload: unknown
}

interface ProcessingMetrics {
  readonly requestId: string
  readonly duration: number
  readonly flags: RuntimeFlags.RuntimeFlags
}

// Request processor with adaptive runtime behavior
const processApiRequest = (request: ApiRequest) => Effect.gen(function* () {
  const startTime = yield* Effect.clockWith(clock => clock.currentTimeMillis)
  
  // Adapt runtime flags based on request type
  const runtimePatch = yield* determineRuntimeFlags(request.type)
  
  const result = yield* Effect.gen(function* () {
    yield* Effect.logInfo(`Processing ${request.type} request: ${request.id}`)
    
    // Simulate processing based on request type
    if (request.type === 'critical') {
      return yield* processCriticalRequest(request)
    } else if (request.type === 'background') {
      return yield* processBackgroundRequest(request)
    } else {
      return yield* processBatchRequest(request)
    }
  }).pipe(
    Effect.withRuntimeFlagsPatch(runtimePatch),
    Effect.timeout(getTimeoutForRequestType(request.type))
  )
  
  // Collect metrics if enabled
  const flags = yield* Effect.getRuntimeFlags
  if (RuntimeFlags.runtimeMetrics(flags)) {
    const endTime = yield* Effect.clockWith(clock => clock.currentTimeMillis)
    yield* recordProcessingMetrics({
      requestId: request.id,
      duration: endTime - startTime,
      flags
    })
  }
  
  return result
})

const determineRuntimeFlags = (requestType: ApiRequest['type']) => Effect.gen(function* () {
  switch (requestType) {
    case 'critical':
      // Critical requests: metrics + supervision for debugging, no cooperative yielding
      return RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics).pipe(
        RuntimeFlagsPatch.andThen(RuntimeFlagsPatch.enable(RuntimeFlags.OpSupervision)),
        RuntimeFlagsPatch.andThen(RuntimeFlagsPatch.disable(RuntimeFlags.CooperativeYielding))
      )
    
    case 'background':
      // Background requests: cooperative yielding enabled, minimal overhead
      return RuntimeFlagsPatch.enable(RuntimeFlags.CooperativeYielding).pipe(
        RuntimeFlagsPatch.andThen(RuntimeFlagsPatch.disable(RuntimeFlags.OpSupervision))
      )
    
    case 'batch':
      // Batch requests: all optimizations for throughput
      return RuntimeFlagsPatch.disable(RuntimeFlags.OpSupervision).pipe(
        RuntimeFlagsPatch.andThen(RuntimeFlagsPatch.disable(RuntimeFlags.RuntimeMetrics)),
        RuntimeFlagsPatch.andThen(RuntimeFlagsPatch.enable(RuntimeFlags.CooperativeYielding))
      )
  }
})

const processCriticalRequest = (request: ApiRequest) => Effect.gen(function* () {
  // High-priority processing with full observability
  yield* Effect.logDebug("Starting critical request processing")
  
  // Simulate intensive processing
  yield* Effect.sleep(Duration.millis(100))
  
  return { status: 'completed', data: request.payload }
})

const processBackgroundRequest = (request: ApiRequest) => Effect.gen(function* () {
  // Yield frequently to allow other fibers to run
  for (let i = 0; i < 10; i++) {
    yield* Effect.yieldNow()
    yield* Effect.sleep(Duration.millis(10))
  }
  
  return { status: 'background_processed', data: request.payload }
})

const processBatchRequest = (request: ApiRequest) => Effect.gen(function* () {
  // Optimized for throughput, minimal overhead
  yield* Effect.sleep(Duration.millis(50))
  return { status: 'batch_processed', data: request.payload }
})

const getTimeoutForRequestType = (type: ApiRequest['type']): Duration.DurationInput => {
  switch (type) {
    case 'critical': return Duration.seconds(5)
    case 'background': return Duration.minutes(2)
    case 'batch': return Duration.minutes(10)
  }
}

const recordProcessingMetrics = (metrics: ProcessingMetrics) => 
  Effect.logInfo("Request processed", metrics)
```

### Example 2: Database Operations with Dynamic Performance Tuning

Database operations need different runtime characteristics based on query complexity and system load:

```typescript
import { Effect, RuntimeFlags, RuntimeFlagsPatch, Context } from "effect"

// Services
interface DatabaseConfig {
  readonly maxConnections: number
  readonly queryTimeout: Duration.DurationInput
  readonly enableQueryMetrics: boolean
}

interface SystemLoad {
  readonly cpuUsage: number
  readonly memoryUsage: number
  readonly activeConnections: number
}

const DatabaseConfig = Context.GenericTag<DatabaseConfig>("DatabaseConfig")
const SystemLoad = Context.GenericTag<SystemLoad>("SystemLoad")

// Database operation with adaptive runtime flags
const executeQuery = <T>(
  query: string,
  params: ReadonlyArray<unknown> = []
) => Effect.gen(function* () {
  const config = yield* DatabaseConfig
  const systemLoad = yield* SystemLoad
  
  // Determine optimal runtime flags based on system conditions
  const runtimePatch = yield* adaptRuntimeForSystemLoad(systemLoad, config)
  
  return yield* Effect.gen(function* () {
    const flags = yield* Effect.getRuntimeFlags
    
    // Log query execution if supervision is enabled
    if (RuntimeFlags.opSupervision(flags)) {
      yield* Effect.logDebug(`Executing query: ${query}`, { params })
    }
    
    // Simulate query execution
    const result = yield* simulateQueryExecution(query, params)
    
    // Record metrics if enabled
    if (RuntimeFlags.runtimeMetrics(flags)) {
      yield* recordQueryMetrics(query, result)
    }
    
    return result
  }).pipe(
    Effect.withRuntimeFlagsPatch(runtimePatch),
    Effect.timeout(config.queryTimeout)
  )
})

const adaptRuntimeForSystemLoad = (
  systemLoad: SystemLoad,
  config: DatabaseConfig
) => Effect.gen(function* () {
  const isHighLoad = systemLoad.cpuUsage > 80 || systemLoad.memoryUsage > 85
  const isMediumLoad = systemLoad.cpuUsage > 60 || systemLoad.memoryUsage > 70
  
  if (isHighLoad) {
    // High load: disable expensive features, enable cooperative yielding
    yield* Effect.logInfo("High system load detected, optimizing for performance")
    return RuntimeFlagsPatch.disable(RuntimeFlags.OpSupervision).pipe(
      RuntimeFlagsPatch.andThen(RuntimeFlagsPatch.disable(RuntimeFlags.RuntimeMetrics)),
      RuntimeFlagsPatch.andThen(RuntimeFlagsPatch.enable(RuntimeFlags.CooperativeYielding))
    )
  } else if (isMediumLoad) {
    // Medium load: enable cooperative yielding, selective metrics
    return RuntimeFlagsPatch.enable(RuntimeFlags.CooperativeYielding).pipe(
      RuntimeFlagsPatch.andThen(
        config.enableQueryMetrics 
          ? RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics)
          : RuntimeFlagsPatch.disable(RuntimeFlags.RuntimeMetrics)
      )
    )
  } else {
    // Low load: enable full observability
    return RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics).pipe(
      RuntimeFlagsPatch.andThen(RuntimeFlagsPatch.enable(RuntimeFlags.OpSupervision))
    )
  }
})

const simulateQueryExecution = <T>(
  query: string, 
  params: ReadonlyArray<unknown>
): Effect.Effect<T> => Effect.gen(function* () {
  // Simulate different query complexities
  const complexity = query.includes('JOIN') ? 'complex' : 
                    query.includes('WHERE') ? 'medium' : 'simple'
  
  const delay = complexity === 'complex' ? 200 : 
                complexity === 'medium' ? 50 : 20
  
  yield* Effect.sleep(Duration.millis(delay))
  return { rows: [], affectedRows: 0 } as T
})

const recordQueryMetrics = (query: string, result: unknown) =>
  Effect.logInfo("Query executed", { 
    query: query.substring(0, 100), 
    resultSize: JSON.stringify(result).length 
  })
```

### Example 3: Testing Framework with Configurable Runtime Behavior

Testing scenarios require different runtime configurations to verify behavior under various conditions:

```typescript
import { Effect, RuntimeFlags, RuntimeFlagsPatch, TestContext, Layer } from "effect"

// Test configuration types
interface TestScenario {
  readonly name: string
  readonly runtimeFlags: RuntimeFlags.RuntimeFlags
  readonly expectedBehavior: 'should_interrupt' | 'should_complete' | 'should_timeout'
}

// Test helpers for runtime flag scenarios
const createTestScenario = (
  name: string,
  flags: RuntimeFlags.RuntimeFlags,
  expectedBehavior: TestScenario['expectedBehavior']
): TestScenario => ({ name, runtimeFlags: flags, expectedBehavior })

const testRuntimeFlagScenarios = Effect.gen(function* () {
  const scenarios: ReadonlyArray<TestScenario> = [
    createTestScenario(
      "interruption enabled",
      RuntimeFlags.make(RuntimeFlags.Interruption, RuntimeFlags.RuntimeMetrics),
      'should_interrupt'
    ),
    createTestScenario(
      "wind-down mode",
      RuntimeFlags.make(RuntimeFlags.WindDown, RuntimeFlags.RuntimeMetrics),
      'should_complete'
    ),
    createTestScenario(
      "cooperative yielding",
      RuntimeFlags.make(RuntimeFlags.CooperativeYielding, RuntimeFlags.RuntimeMetrics),
      'should_timeout'
    ),
    createTestScenario(
      "production optimized",
      RuntimeFlags.make(RuntimeFlags.Interruption),
      'should_interrupt'
    )
  ]
  
  // Run each test scenario
  for (const scenario of scenarios) {
    yield* runTestScenario(scenario)
  }
})

const runTestScenario = (scenario: TestScenario) => Effect.gen(function* () {
  yield* Effect.logInfo(`Running test scenario: ${scenario.name}`)
  
  // Create test effect that runs with specific runtime flags
  const testEffect = createLongRunningTask().pipe(
    Effect.withRuntimeFlagsPatch(
      RuntimeFlagsPatch.diff(RuntimeFlags.none, scenario.runtimeFlags)
    )
  )
  
  // Execute test with timeout and interruption
  const result = yield* Effect.race(
    testEffect,
    Effect.sleep(Duration.seconds(1)).pipe(Effect.interrupt)
  ).pipe(
    Effect.timeout(Duration.seconds(2)),
    Effect.either
  )
  
  // Verify the result matches expected behavior
  yield* verifyTestResult(scenario, result)
})

const createLongRunningTask = () => Effect.gen(function* () {
  // Simulate a task that can be interrupted or completed
  for (let i = 0; i < 100; i++) {
    const flags = yield* Effect.getRuntimeFlags
    
    // Check if we should yield to other fibers
    if (RuntimeFlags.cooperativeYielding(flags)) {
      yield* Effect.yieldNow()
    }
    
    // Check if we're in wind-down mode
    if (RuntimeFlags.windDown(flags)) {
      yield* Effect.logDebug("Completing task in wind-down mode")
      yield* Effect.sleep(Duration.millis(10))
      continue
    }
    
    // Normal processing
    yield* Effect.sleep(Duration.millis(20))
  }
  
  return "task completed"
})

const verifyTestResult = (
  scenario: TestScenario,
  result: Either.Either<unknown, string>
) => Effect.gen(function* () {
  const success = Either.isRight(result)
  const shouldComplete = scenario.expectedBehavior === 'should_complete'
  
  if (success === shouldComplete) {
    yield* Effect.logInfo(`✓ Test passed: ${scenario.name}`)
  } else {
    yield* Effect.logError(`✗ Test failed: ${scenario.name}`, {
      expected: scenario.expectedBehavior,
      actual: success ? 'completed' : 'interrupted/timeout'
    })
  }
})

// Test layer that provides different runtime configurations
const TestRuntimeLayer = Layer.effectDiscard(Effect.gen(function* () {
  yield* Effect.logInfo("Setting up test runtime environment")
  
  // Enable metrics and supervision for testing
  yield* Effect.patchRuntimeFlags(
    RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics).pipe(
      RuntimeFlagsPatch.andThen(RuntimeFlagsPatch.enable(RuntimeFlags.OpSupervision))
    )
  )
}))
```

## Advanced Features Deep Dive

### RuntimeFlagsPatch: Composable Flag Changes

RuntimeFlagsPatch allows you to describe changes to runtime flags that can be composed and applied:

#### Basic RuntimeFlagsPatch Usage

```typescript
import { RuntimeFlags, RuntimeFlagsPatch } from "effect"

// Create patches for individual flags
const enableMetrics = RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics)
const disableSupervision = RuntimeFlagsPatch.disable(RuntimeFlags.OpSupervision)

// Compose patches
const productionPatch = RuntimeFlagsPatch.andThen(enableMetrics, disableSupervision)

// Apply patches to runtime flags
const currentFlags = RuntimeFlags.make(RuntimeFlags.Interruption)
const updatedFlags = RuntimeFlags.patch(currentFlags, productionPatch)
```

#### Advanced RuntimeFlagsPatch: Dynamic Configuration

```typescript
import { Effect, RuntimeFlags, RuntimeFlagsPatch, Config } from "effect"

// Configuration-driven runtime flags
interface RuntimeConfig {
  readonly enableMetrics: boolean
  readonly enableSupervision: boolean
  readonly enableCooperativeYielding: boolean
  readonly enableInterruption: boolean
}

const createRuntimePatchFromConfig = (config: RuntimeConfig) => Effect.gen(function* () {
  let patch = RuntimeFlagsPatch.empty
  
  // Build patch based on configuration
  if (config.enableMetrics) {
    patch = RuntimeFlagsPatch.andThen(patch, RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics))
  } else {
    patch = RuntimeFlagsPatch.andThen(patch, RuntimeFlagsPatch.disable(RuntimeFlags.RuntimeMetrics))
  }
  
  if (config.enableSupervision) {
    patch = RuntimeFlagsPatch.andThen(patch, RuntimeFlagsPatch.enable(RuntimeFlags.OpSupervision))
  }
  
  if (config.enableCooperativeYielding) {
    patch = RuntimeFlagsPatch.andThen(patch, RuntimeFlagsPatch.enable(RuntimeFlags.CooperativeYielding))
  }
  
  if (!config.enableInterruption) {
    patch = RuntimeFlagsPatch.andThen(patch, RuntimeFlagsPatch.disable(RuntimeFlags.Interruption))
  }
  
  return patch
})

// Apply configuration to current fiber
const applyRuntimeConfig = (config: RuntimeConfig) => Effect.gen(function* () {
  const patch = yield* createRuntimePatchFromConfig(config)
  yield* Effect.patchRuntimeFlags(patch)
  
  const updatedFlags = yield* Effect.getRuntimeFlags
  yield* Effect.logInfo("Runtime configuration applied", {
    metrics: RuntimeFlags.runtimeMetrics(updatedFlags),
    supervision: RuntimeFlags.opSupervision(updatedFlags),
    cooperative: RuntimeFlags.cooperativeYielding(updatedFlags),
    interruption: RuntimeFlags.interruption(updatedFlags)
  })
})

// Create reusable runtime modifier
const withRuntimeConfig = <A, E, R>(
  config: RuntimeConfig
) => (effect: Effect.Effect<A, E, R>): Effect.Effect<A, E, R> => Effect.gen(function* () {
  const patch = yield* createRuntimePatchFromConfig(config)
  return yield* effect.pipe(Effect.withRuntimeFlagsPatch(patch))
})
```

### Runtime Flag Inspection and Diffing

RuntimeFlags provide utilities for inspecting and comparing flag states:

```typescript
import { Effect, RuntimeFlags, RuntimeFlagsPatch } from "effect"

const analyzeRuntimeChanges = Effect.gen(function* () {
  const initialFlags = yield* Effect.getRuntimeFlags
  
  // Apply some changes
  yield* Effect.patchRuntimeFlags(
    RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics)
  )
  
  const updatedFlags = yield* Effect.getRuntimeFlags
  
  // Create a diff showing what changed
  const diff = RuntimeFlags.diff(initialFlags, updatedFlags)
  
  // Inspect the changes
  yield* Effect.logInfo("Runtime flags analysis", {
    initial: RuntimeFlags.render(initialFlags),
    updated: RuntimeFlags.render(updatedFlags),
    diff: RuntimeFlagsPatch.render(diff)
  })
  
  // Get individual flag states
  const flagStates = {
    interruption: RuntimeFlags.interruption(updatedFlags),
    metrics: RuntimeFlags.runtimeMetrics(updatedFlags),
    supervision: RuntimeFlags.opSupervision(updatedFlags),
    windDown: RuntimeFlags.windDown(updatedFlags),
    cooperative: RuntimeFlags.cooperativeYielding(updatedFlags)
  }
  
  yield* Effect.logInfo("Individual flag states", flagStates)
  
  return { initialFlags, updatedFlags, diff, flagStates }
})
```

## Practical Patterns & Best Practices

### Pattern 1: Environment-Based Runtime Configuration

```typescript
import { Effect, RuntimeFlags, RuntimeFlagsPatch, Config, Layer } from "effect"

// Environment-aware runtime configuration
const EnvironmentRuntimeLayer = Layer.effectDiscard(Effect.gen(function* () {
  const environment = yield* Config.string("NODE_ENV").pipe(Config.withDefault("development"))
  const enableDebug = yield* Config.boolean("ENABLE_DEBUG").pipe(Config.withDefault(false))
  const enableMetrics = yield* Config.boolean("ENABLE_METRICS").pipe(Config.withDefault(false))
  
  const patch = yield* createEnvironmentPatch(environment, enableDebug, enableMetrics)
  yield* Effect.patchRuntimeFlags(patch)
  
  yield* Effect.logInfo(`Runtime configured for ${environment} environment`)
}))

const createEnvironmentPatch = (
  environment: string,
  enableDebug: boolean,
  enableMetrics: boolean
) => Effect.gen(function* () {
  switch (environment) {
    case 'production':
      // Production: metrics enabled, minimal supervision
      return RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics).pipe(
        RuntimeFlagsPatch.andThen(RuntimeFlagsPatch.disable(RuntimeFlags.OpSupervision)),
        RuntimeFlagsPatch.andThen(RuntimeFlagsPatch.enable(RuntimeFlags.CooperativeYielding))
      )
    
    case 'development':
      // Development: full observability
      return RuntimeFlagsPatch.enable(RuntimeFlags.OpSupervision).pipe(
        RuntimeFlagsPatch.andThen(
          enableMetrics 
            ? RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics)
            : RuntimeFlagsPatch.disable(RuntimeFlags.RuntimeMetrics)
        ),
        RuntimeFlagsPatch.andThen(
          enableDebug
            ? RuntimeFlagsPatch.enable(RuntimeFlags.OpSupervision)
            : RuntimeFlagsPatch.empty
        )
      )
    
    case 'test':
      // Testing: deterministic behavior
      return RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics).pipe(
        RuntimeFlagsPatch.andThen(RuntimeFlagsPatch.enable(RuntimeFlags.OpSupervision)),
        RuntimeFlagsPatch.andThen(RuntimeFlagsPatch.disable(RuntimeFlags.CooperativeYielding))
      )
    
    default:
      return RuntimeFlagsPatch.empty
  }
})
```

### Pattern 2: Conditional Runtime Flag Management

```typescript
import { Effect, RuntimeFlags, RuntimeFlagsPatch } from "effect"

// Helper for conditional flag management
const conditionalRuntimeFlags = <A, E, R>(
  condition: (flags: RuntimeFlags.RuntimeFlags) => boolean,
  whenTrue: RuntimeFlagsPatch.RuntimeFlagsPatch,
  whenFalse: RuntimeFlagsPatch.RuntimeFlagsPatch = RuntimeFlagsPatch.empty
) => (effect: Effect.Effect<A, E, R>): Effect.Effect<A, E, R> => Effect.gen(function* () {
  const currentFlags = yield* Effect.getRuntimeFlags
  const patch = condition(currentFlags) ? whenTrue : whenFalse
  return yield* effect.pipe(Effect.withRuntimeFlagsPatch(patch))
})

// Usage examples
const withMetricsIfNotEnabled = conditionalRuntimeFlags(
  (flags) => !RuntimeFlags.runtimeMetrics(flags),
  RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics)
)

const withSupervisionIfDebugMode = conditionalRuntimeFlags(
  (flags) => RuntimeFlags.runtimeMetrics(flags), // Use metrics as debug indicator
  RuntimeFlagsPatch.enable(RuntimeFlags.OpSupervision),
  RuntimeFlagsPatch.disable(RuntimeFlags.OpSupervision)
)

// Composite helper
const smartRuntimeOptimization = <A, E, R>(
  effect: Effect.Effect<A, E, R>
): Effect.Effect<A, E, R> => effect.pipe(
  withMetricsIfNotEnabled,
  withSupervisionIfDebugMode
)
```

### Pattern 3: Runtime Flag Scoping and Restoration

```typescript
import { Effect, RuntimeFlags, RuntimeFlagsPatch, Scope } from "effect"

// Scoped runtime flag changes
const withScopedRuntimeFlags = <A, E, R>(
  patch: RuntimeFlagsPatch.RuntimeFlagsPatch
) => (effect: Effect.Effect<A, E, R>): Effect.Effect<A, E, R> => Effect.gen(function* () {
  return yield* Effect.acquireUseRelease(
    // Acquire: save current flags and apply patch
    Effect.gen(function* () {
      const originalFlags = yield* Effect.getRuntimeFlags
      yield* Effect.patchRuntimeFlags(patch)
      return originalFlags
    }),
    // Use: run the effect with modified flags
    () => effect,
    // Release: restore original flags
    (originalFlags) => Effect.gen(function* () {
      const currentFlags = yield* Effect.getRuntimeFlags
      const restorePatch = RuntimeFlags.diff(currentFlags, originalFlags)
      yield* Effect.patchRuntimeFlags(restorePatch)
    })
  )
})

// Usage example
const temporarilyDisableInterruption = <A, E, R>(
  effect: Effect.Effect<A, E, R>
): Effect.Effect<A, E, R> => effect.pipe(
  withScopedRuntimeFlags(RuntimeFlagsPatch.disable(RuntimeFlags.Interruption))
)

const criticalSection = Effect.gen(function* () {
  yield* Effect.logInfo("Entering critical section")
  
  // This section cannot be interrupted
  yield* temporarilyDisableInterruption(Effect.gen(function* () {
    yield* Effect.logInfo("Performing critical operations")
    yield* Effect.sleep(Duration.seconds(2))
    yield* Effect.logInfo("Critical operations completed")
  }))
  
  yield* Effect.logInfo("Exiting critical section")
})
```

## Integration Examples

### Integration with HTTP Server and Metrics

```typescript
import { Effect, RuntimeFlags, RuntimeFlagsPatch, Layer } from "effect"
import { HttpApi, HttpServer, HttpRouter } from "@effect/platform"

// HTTP server with runtime flag-based behavior
const createAdaptiveHttpServer = Effect.gen(function* () {
  const api = yield* HttpApi.make({ title: "Adaptive API" })
  
  const router = HttpRouter.empty.pipe(
    HttpRouter.get("/health", Effect.gen(function* () {
      const flags = yield* Effect.getRuntimeFlags
      
      return HttpApi.response.json({
        status: "healthy",
        runtime: {
          metrics: RuntimeFlags.runtimeMetrics(flags),
          supervision: RuntimeFlags.opSupervision(flags),
          cooperative: RuntimeFlags.cooperativeYielding(flags)
        }
      })
    })),
    
    HttpRouter.post("/configure-runtime", Effect.gen(function* () {
      const body = yield* HttpApi.request.schemaBodyJson(ConfigurationSchema)
      const patch = yield* createPatchFromRequest(body)
      
      yield* Effect.patchRuntimeFlags(patch)
      
      const updatedFlags = yield* Effect.getRuntimeFlags
      
      return HttpApi.response.json({
        message: "Runtime configuration updated",
        flags: RuntimeFlags.render(updatedFlags)
      })
    })),
    
    HttpRouter.get("/intensive/:duration", Effect.gen(function* () {
      const params = yield* HttpApi.request.schemaParams(
        Schema.Struct({ duration: Schema.NumberFromString })
      )
      
      // Adapt behavior based on current runtime flags
      const result = yield* performIntensiveTask(Duration.seconds(params.duration)).pipe(
        adaptTaskToRuntimeFlags
      )
      
      return HttpApi.response.json(result)
    }))
  )
  
  return HttpServer.serve(HttpApi.api(api).pipe(HttpApi.addRouter(router)))
})

const adaptTaskToRuntimeFlags = <A, E, R>(
  effect: Effect.Effect<A, E, R>
): Effect.Effect<A, E, R> => Effect.gen(function* () {
  const flags = yield* Effect.getRuntimeFlags
  
  // If cooperative yielding is enabled, break work into smaller chunks
  if (RuntimeFlags.cooperativeYielding(flags)) {
    return yield* effect.pipe(
      Effect.withRuntimeFlagsPatch(RuntimeFlagsPatch.enable(RuntimeFlags.CooperativeYielding))
    )
  }
  
  // If metrics are enabled, add detailed tracing
  if (RuntimeFlags.runtimeMetrics(flags)) {
    return yield* effect.pipe(
      Effect.withSpan("intensive-task"),
      Effect.withRuntimeFlagsPatch(RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics))
    )
  }
  
  return yield* effect
})

const performIntensiveTask = (duration: Duration.Duration) => Effect.gen(function* () {
  const chunks = Math.ceil(Duration.toMillis(duration) / 100)
  let progress = 0
  
  for (let i = 0; i < chunks; i++) {
    const flags = yield* Effect.getRuntimeFlags
    
    // Yield if cooperative yielding is enabled
    if (RuntimeFlags.cooperativeYielding(flags)) {
      yield* Effect.yieldNow()
    }
    
    yield* Effect.sleep(Duration.millis(100))
    progress = ((i + 1) / chunks) * 100
    
    // Log progress if supervision is enabled
    if (RuntimeFlags.opSupervision(flags)) {
      yield* Effect.logDebug(`Task progress: ${progress.toFixed(1)}%`)
    }
  }
  
  return { completed: true, progress: 100 }
})

// Configuration schema
const ConfigurationSchema = Schema.Struct({
  enableMetrics: Schema.optional(Schema.Boolean),
  enableSupervision: Schema.optional(Schema.Boolean), 
  enableCooperativeYielding: Schema.optional(Schema.Boolean)
})

const createPatchFromRequest = (config: Schema.Schema.Type<typeof ConfigurationSchema>) => Effect.gen(function* () {
  let patch = RuntimeFlagsPatch.empty
  
  if (config.enableMetrics !== undefined) {
    patch = RuntimeFlagsPatch.andThen(
      patch,
      config.enableMetrics 
        ? RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics)
        : RuntimeFlagsPatch.disable(RuntimeFlags.RuntimeMetrics)
    )
  }
  
  if (config.enableSupervision !== undefined) {
    patch = RuntimeFlagsPatch.andThen(
      patch,
      config.enableSupervision
        ? RuntimeFlagsPatch.enable(RuntimeFlags.OpSupervision)
        : RuntimeFlagsPatch.disable(RuntimeFlags.OpSupervision)
    )
  }
  
  if (config.enableCooperativeYielding !== undefined) {
    patch = RuntimeFlagsPatch.andThen(
      patch,
      config.enableCooperativeYielding
        ? RuntimeFlagsPatch.enable(RuntimeFlags.CooperativeYielding)
        : RuntimeFlagsPatch.disable(RuntimeFlags.CooperativeYielding)
    )
  }
  
  return patch
})
```

### Integration with Testing Framework

```typescript
import { Effect, RuntimeFlags, RuntimeFlagsPatch, TestContext, TestServices } from "effect"
import { test } from "@effect/vitest"

// Test utilities for runtime flag testing
const testWithRuntimeFlags = (
  name: string,
  flags: RuntimeFlags.RuntimeFlags,
  testBody: Effect.Effect<void, unknown, TestServices.TestServices>
) => {
  return test(name, () => 
    testBody.pipe(
      Effect.withRuntimeFlagsPatch(
        RuntimeFlags.diff(RuntimeFlags.none, flags)
      ),
      Effect.provide(TestContext.TestContext)
    )
  )
}

// Test suite demonstrating runtime flag behavior
describe("RuntimeFlags Integration Tests", () => {
  testWithRuntimeFlags(
    "should collect metrics when RuntimeMetrics is enabled",
    RuntimeFlags.make(RuntimeFlags.RuntimeMetrics, RuntimeFlags.OpSupervision),
    Effect.gen(function* () {
      const metrics: Array<string> = []
      
      // Mock metrics collection
      const collectMetric = (name: string) => Effect.gen(function* () {
        const flags = yield* Effect.getRuntimeFlags
        if (RuntimeFlags.runtimeMetrics(flags)) {
          metrics.push(name)
        }
      })
      
      yield* collectMetric("operation.start")
      yield* Effect.sleep(Duration.millis(100))
      yield* collectMetric("operation.end")
      
      // Verify metrics were collected
      expect(metrics).toEqual(["operation.start", "operation.end"])
    })
  )
  
  testWithRuntimeFlags(
    "should be interruptible when Interruption flag is enabled",
    RuntimeFlags.make(RuntimeFlags.Interruption),
    Effect.gen(function* () {
      const longRunningTask = Effect.gen(function* () {
        for (let i = 0; i < 100; i++) {
          yield* Effect.sleep(Duration.millis(10))
        }
        return "completed"
      })
      
      const result = yield* Effect.race(
        longRunningTask,
        Effect.sleep(Duration.millis(50)).pipe(Effect.interrupt)
      ).pipe(Effect.either)
      
      // Should be interrupted, not completed
      expect(Either.isLeft(result)).toBe(true)
    })
  )
  
  testWithRuntimeFlags(
    "should complete in wind-down mode even when interrupted",
    RuntimeFlags.make(RuntimeFlags.WindDown, RuntimeFlags.Interruption),
    Effect.gen(function* () {
      let completed = false
      
      const taskWithCleanup = Effect.gen(function* () {
        yield* Effect.sleep(Duration.millis(100))
        completed = true
        return "completed"
      })
      
      const result = yield* Effect.race(
        taskWithCleanup,
        Effect.sleep(Duration.millis(50)).pipe(Effect.interrupt)
      ).pipe(Effect.either)
      
      // Should complete despite interruption attempt
      expect(completed).toBe(true)
    })
  )
})

// Performance test with different runtime configurations
const benchmarkRuntimeConfigurations = Effect.gen(function* () {
  const configurations = [
    { name: "minimal", flags: RuntimeFlags.none },
    { name: "metrics-only", flags: RuntimeFlags.make(RuntimeFlags.RuntimeMetrics) },
    { name: "full-supervision", flags: RuntimeFlags.make(RuntimeFlags.OpSupervision, RuntimeFlags.RuntimeMetrics) },
    { name: "cooperative", flags: RuntimeFlags.make(RuntimeFlags.CooperativeYielding) }
  ]
  
  const benchmarkTask = Effect.gen(function* () {
    // Simulate work
    for (let i = 0; i < 1000; i++) {
      const flags = yield* Effect.getRuntimeFlags
      
      if (RuntimeFlags.cooperativeYielding(flags) && i % 100 === 0) {
        yield* Effect.yieldNow()
      }
      
      if (RuntimeFlags.opSupervision(flags) && i % 500 === 0) {
        yield* Effect.logDebug(`Progress: ${i}/1000`)
      }
    }
  })
  
  const results = []
  
  for (const config of configurations) {
    const start = yield* Effect.clockWith(clock => clock.currentTimeMillis)
    
    yield* benchmarkTask.pipe(
      Effect.withRuntimeFlagsPatch(
        RuntimeFlags.diff(RuntimeFlags.none, config.flags)
      )
    )
    
    const end = yield* Effect.clockWith(clock => clock.currentTimeMillis)
    const duration = end - start
    
    results.push({ configuration: config.name, duration })
    
    yield* Effect.logInfo(`${config.name}: ${duration}ms`)
  }
  
  return results
})
```

## Conclusion

RuntimeFlags provide dynamic, fine-grained control over Effect runtime behavior, enabling applications to adapt their execution characteristics based on environment, load conditions, and operational requirements.

Key benefits:
- **Dynamic Behavior Control**: Modify runtime characteristics without code changes or restarts
- **Performance Optimization**: Tune execution for different workload patterns and system conditions  
- **Enhanced Observability**: Selectively enable metrics and debugging based on operational needs
- **Testing Flexibility**: Verify behavior under different runtime configurations
- **Production Safety**: Enable safe experimentation with runtime optimizations

Use RuntimeFlags when you need to:
- Adapt application behavior based on system load or environment
- Enable/disable observability features dynamically
- Optimize performance for different deployment scenarios
- Test effects under various runtime conditions
- Implement feature flags at the runtime level