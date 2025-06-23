# RuntimeFlagsPatch: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem RuntimeFlagsPatch Solves

When building Effect applications, you often need to temporarily modify runtime behavior for specific operations. Traditional approaches might involve:

```typescript
// Traditional approach - manually managing flags
const originalFlags = await Effect.getRuntimeFlags()
await Effect.patchRuntimeFlags(RuntimeFlagsPatch.enable(RuntimeFlags.OpSupervision))
try {
  // Do work with supervision enabled
  await someOperation()
} finally {
  // Restore original flags
  await Effect.patchRuntimeFlags(RuntimeFlags.diff(originalFlags, await Effect.getRuntimeFlags()))
}
```

This approach leads to:
- **Boilerplate Code** - Manual save/restore of runtime flags
- **Error-Prone Logic** - Easy to forget cleanup or make mistakes in diff calculations
- **Poor Composability** - Hard to combine multiple flag modifications safely
- **Resource Leaks** - Flags may not be restored if errors occur

### The RuntimeFlagsPatch Solution

RuntimeFlagsPatch provides efficient, composable, and safe runtime flag modifications:

```typescript
import { Effect, RuntimeFlags, RuntimeFlagsPatch } from "effect"

// Create a patch that enables supervision
const supervisionPatch = RuntimeFlagsPatch.enable(RuntimeFlags.OpSupervision)

// Apply temporarily with automatic cleanup
const result = someOperation().pipe(
  Effect.withRuntimeFlagsPatch(supervisionPatch)
)
```

### Key Concepts

**RuntimeFlagsPatch**: An efficient representation of runtime flag modifications that can be applied, combined, and inverted

**Patch Operations**: Enable/disable flags, combine patches, and create inverses for rollback

**Scope Management**: Automatic cleanup and restoration of flags when effects complete

## Basic Usage Patterns

### Pattern 1: Single Flag Operations

```typescript
import { Effect, RuntimeFlags, RuntimeFlagsPatch } from "effect"

// Enable a single flag
const enableMetrics = RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics)

// Disable a single flag  
const disableInterruption = RuntimeFlagsPatch.disable(RuntimeFlags.Interruption)

// Apply a patch to current fiber
const applyPatch = Effect.patchRuntimeFlags(enableMetrics)
```

### Pattern 2: Scoped Flag Modifications

```typescript
// Apply patch temporarily to a specific effect
const withMetrics = <A, E, R>(effect: Effect.Effect<A, E, R>) =>
  effect.pipe(
    Effect.withRuntimeFlagsPatch(RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics))
  )

// Use the scoped modification
const result = withMetrics(
  Effect.gen(function* () {
    // This code runs with runtime metrics enabled
    const data = yield* fetchData()
    return yield* processData(data)
  })
)
```

### Pattern 3: Patch Composition

```typescript
// Combine multiple patches
const debugPatch = RuntimeFlagsPatch.enable(RuntimeFlags.OpSupervision).pipe(
  RuntimeFlagsPatch.andThen(RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics))
)

// Alternative composition using both
const comboPatch = RuntimeFlagsPatch.both(
  RuntimeFlagsPatch.enable(RuntimeFlags.OpSupervision),
  RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics)
)
```

## Real-World Examples

### Example 1: Database Transaction Monitoring

```typescript
import { Effect, RuntimeFlags, RuntimeFlagsPatch } from "effect"

// Database operation that needs monitoring
const executeTransaction = (query: string) => Effect.gen(function* () {
  const db = yield* DatabaseService
  const result = yield* db.execute(query)
  return result
})

// Create monitoring patch for database operations
const dbMonitoringPatch = RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics).pipe(
  RuntimeFlagsPatch.andThen(RuntimeFlagsPatch.enable(RuntimeFlags.OpSupervision))
)

// Apply monitoring to database operations
export const monitoredDbOperation = <A, E, R>(
  operation: Effect.Effect<A, E, R>
): Effect.Effect<A, E, R> =>
  operation.pipe(
    Effect.withRuntimeFlagsPatch(dbMonitoringPatch),
    Effect.withSpan("database.operation")
  )

// Usage in application
const createUser = (userData: UserData) => Effect.gen(function* () {
  const query = `INSERT INTO users (name, email) VALUES (?, ?)`
  return yield* executeTransaction(query).pipe(
    monitoredDbOperation
  )
})
```

### Example 2: Development vs Production Flag Management

```typescript
// Configuration-based flag management
const createEnvironmentPatch = (env: "development" | "production") => {
  const basePatch = RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics)
  
  if (env === "development") {
    return basePatch.pipe(
      RuntimeFlagsPatch.andThen(RuntimeFlagsPatch.enable(RuntimeFlags.OpSupervision))
    )
  }
  
  return basePatch
}

// Application startup with environment-specific flags
const startApplication = Effect.gen(function* () {
  const config = yield* Config.Config
  const envPatch = createEnvironmentPatch(config.environment)
  
  return yield* Effect.gen(function* () {
    yield* Logger.log("Starting application...")
    yield* setupRoutes()
    yield* startServer()
    yield* Logger.log("Application started successfully")
  }).pipe(
    Effect.patchRuntimeFlags(envPatch)
  )
})
```

### Example 3: Critical Section with Interruption Control

```typescript
// Critical operations that should not be interrupted
const criticalOperation = <A, E, R>(
  operation: Effect.Effect<A, E, R>,
  description: string
): Effect.Effect<A, E, R> =>
  Effect.gen(function* () {
    yield* Logger.info(`Starting critical operation: ${description}`)
    
    const result = yield* operation.pipe(
      Effect.withRuntimeFlagsPatch(RuntimeFlagsPatch.disable(RuntimeFlags.Interruption))
    )
    
    yield* Logger.info(`Completed critical operation: ${description}`)
    return result
  })

// Usage for file operations
const saveUserData = (data: UserData) =>
  criticalOperation(
    Effect.gen(function* () {
      yield* validateData(data)
      yield* writeToDatabase(data)
      yield* updateCache(data) 
      yield* sendNotification(data)
    }),
    "user-data-save"
  )
```

## Advanced Features Deep Dive

### Feature 1: Patch Inversion and Rollback

Patch inversion allows you to create the opposite of a patch, useful for manual rollback scenarios.

#### Basic Inversion Usage

```typescript
// Create a patch that enables metrics
const enablePatch = RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics)

// Create the inverse patch that disables metrics
const disablePatch = RuntimeFlagsPatch.inverse(enablePatch)

// Manual application and rollback
const manualFlagManagement = Effect.gen(function* () {
  yield* Effect.patchRuntimeFlags(enablePatch)
  
  try {
    // Do work with metrics enabled
    yield* performMetricedOperation()
  } finally {
    // Manually rollback
    yield* Effect.patchRuntimeFlags(disablePatch)
  }
})
```

#### Real-World Inversion Example

```typescript
// Advanced debugging system with rollback capability
class DebugSession {
  private patches: Array<RuntimeFlagsPatch.RuntimeFlagsPatch> = []
  
  enableDebugMode = Effect.gen(function* () {
    const debugPatch = RuntimeFlagsPatch.enable(RuntimeFlags.OpSupervision).pipe(
      RuntimeFlagsPatch.andThen(RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics))
    )
    
    this.patches.push(debugPatch)
    yield* Effect.patchRuntimeFlags(debugPatch)
    yield* Logger.info("Debug mode enabled")
  })
  
  rollbackAll = Effect.gen(function* () {
    for (const patch of this.patches.reverse()) {
      yield* Effect.patchRuntimeFlags(RuntimeFlagsPatch.inverse(patch))
    }
    this.patches.length = 0
    yield* Logger.info("All debug patches rolled back")
  })
}
```

### Feature 2: Patch Composition Strategies

RuntimeFlagsPatch supports multiple composition strategies for different use cases.

#### Sequential Composition with andThen

```typescript
// Apply patches in sequence
const sequentialPatch = RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics).pipe(
  RuntimeFlagsPatch.andThen(RuntimeFlagsPatch.enable(RuntimeFlags.OpSupervision)),
  RuntimeFlagsPatch.andThen(RuntimeFlagsPatch.disable(RuntimeFlags.CooperativeYielding))
)

// Useful for building complex patch chains
const createDevelopmentPatch = () =>
  RuntimeFlagsPatch.empty.pipe(
    RuntimeFlagsPatch.andThen(RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics)),
    RuntimeFlagsPatch.andThen(RuntimeFlagsPatch.enable(RuntimeFlags.OpSupervision))
  )
```

#### Parallel Composition with both and either

```typescript
// Both patches must be active (intersection)
const strictPatch = RuntimeFlagsPatch.both(
  RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics),
  RuntimeFlagsPatch.enable(RuntimeFlags.OpSupervision)
)

// Either patch can be active (union)
const lenientPatch = RuntimeFlagsPatch.either(
  RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics),
  RuntimeFlagsPatch.enable(RuntimeFlags.OpSupervision)
)
```

### Feature 3: Conditional Patch Application

```typescript
// Helper for conditional patch application
const conditionalPatch = <A, E, R>(
  condition: boolean,
  patch: RuntimeFlagsPatch.RuntimeFlagsPatch,
  effect: Effect.Effect<A, E, R>
): Effect.Effect<A, E, R> =>
  condition
    ? effect.pipe(Effect.withRuntimeFlagsPatch(patch))
    : effect

// Usage with environment-based conditions
const runWithOptionalDebug = <A, E, R>(
  effect: Effect.Effect<A, E, R>,
  debugEnabled: boolean
) =>
  conditionalPatch(
    debugEnabled,
    RuntimeFlagsPatch.enable(RuntimeFlags.OpSupervision),
    effect
  )
```

## Practical Patterns & Best Practices

### Pattern 1: Patch Builder Pattern

```typescript
// Reusable patch builder for common scenarios
class RuntimePatchBuilder {
  private patches: RuntimeFlagsPatch.RuntimeFlagsPatch[] = []
  
  withMetrics() {
    this.patches.push(RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics))
    return this
  }
  
  withSupervision() {
    this.patches.push(RuntimeFlagsPatch.enable(RuntimeFlags.OpSupervision))
    return this
  }
  
  withoutInterruption() {
    this.patches.push(RuntimeFlagsPatch.disable(RuntimeFlags.Interruption))
    return this
  }
  
  withoutYielding() {
    this.patches.push(RuntimeFlagsPatch.disable(RuntimeFlags.CooperativeYielding))
    return this
  }
  
  build(): RuntimeFlagsPatch.RuntimeFlagsPatch {
    return this.patches.reduce(
      (acc, patch) => acc.pipe(RuntimeFlagsPatch.andThen(patch)),
      RuntimeFlagsPatch.empty
    )
  }
}

// Usage
const debugPatch = new RuntimePatchBuilder()
  .withMetrics()
  .withSupervision()
  .build()

const criticalPatch = new RuntimePatchBuilder()
  .withoutInterruption()
  .withoutYielding()
  .build()
```

### Pattern 2: Patch Registry Pattern

```typescript
// Centralized patch management
const RuntimePatches = {
  // Development patches
  fullDebug: RuntimeFlagsPatch.enable(RuntimeFlags.OpSupervision).pipe(
    RuntimeFlagsPatch.andThen(RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics))
  ),
  
  // Production patches
  metricsOnly: RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics),
  
  // Critical operation patches
  uninterruptible: RuntimeFlagsPatch.disable(RuntimeFlags.Interruption),
  
  // Performance patches
  fastPath: RuntimeFlagsPatch.disable(RuntimeFlags.CooperativeYielding).pipe(
    RuntimeFlagsPatch.andThen(RuntimeFlagsPatch.disable(RuntimeFlags.OpSupervision))
  ),
  
  // Compose patches
  debugCritical: RuntimeFlagsPatch.both(
    RuntimeFlagsPatch.enable(RuntimeFlags.OpSupervision),
    RuntimeFlagsPatch.disable(RuntimeFlags.Interruption)
  )
} as const

// Usage with type safety
const performCriticalDebugOperation = <A, E, R>(
  operation: Effect.Effect<A, E, R>
): Effect.Effect<A, E, R> =>
  operation.pipe(
    Effect.withRuntimeFlagsPatch(RuntimePatches.debugCritical)
  )
```

### Pattern 3: Scoped Patch Management

```typescript
// Scoped patch application with resource management
const withScopedFlags = <A, E, R>(
  patch: RuntimeFlagsPatch.RuntimeFlagsPatch,
  effect: Effect.Effect<A, E, R>
): Effect.Effect<A, E, R> =>
  Effect.gen(function* () {
    yield* Effect.withRuntimeFlagsPatchScoped(patch)
    return yield* effect
  }).pipe(
    Effect.scoped
  )

// Layer-based flag management
const DebugFlagsLayer = Layer.effectDiscard(
  Effect.withRuntimeFlagsPatchScoped(
    RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics).pipe(
      RuntimeFlagsPatch.andThen(RuntimeFlagsPatch.enable(RuntimeFlags.OpSupervision))
    )
  )
)

// Use with layers
const debugApp = mainApplication.pipe(
  Effect.provide(DebugFlagsLayer)
)
```

## Integration Examples

### Integration with Configuration Systems

```typescript
import { Config, Effect, RuntimeFlags, RuntimeFlagsPatch } from "effect"

// Configuration-driven patch creation
const createConfigPatch = Effect.gen(function* () {
  const metricsEnabled = yield* Config.boolean("METRICS_ENABLED").pipe(
    Config.withDefault(true)
  )
  const supervisionEnabled = yield* Config.boolean("SUPERVISION_ENABLED").pipe(
    Config.withDefault(false)
  )
  const interruptionEnabled = yield* Config.boolean("INTERRUPTION_ENABLED").pipe(
    Config.withDefault(true)
  )
  
  let patch = RuntimeFlagsPatch.empty
  
  if (metricsEnabled) {
    patch = patch.pipe(RuntimeFlagsPatch.andThen(
      RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics)
    ))
  }
  
  if (supervisionEnabled) {
    patch = patch.pipe(RuntimeFlagsPatch.andThen(
      RuntimeFlagsPatch.enable(RuntimeFlags.OpSupervision)
    ))
  }
  
  if (!interruptionEnabled) {
    patch = patch.pipe(RuntimeFlagsPatch.andThen(
      RuntimeFlagsPatch.disable(RuntimeFlags.Interruption)
    ))
  }
  
  return patch
})

// Application startup with config-driven flags
const startWithConfig = Effect.gen(function* () {
  const configPatch = yield* createConfigPatch
  
  return yield* Effect.gen(function* () {
    yield* Logger.info("Starting application with configured runtime flags")
    yield* startServer()
    yield* registerShutdownHooks()
  }).pipe(
    Effect.patchRuntimeFlags(configPatch)
  )
})
```

### Integration with Testing Frameworks

```typescript
// Testing utilities for runtime flag scenarios
export const TestUtils = {
  // Test with specific flags enabled
  withFlags: <A, E, R>(
    flags: RuntimeFlagsPatch.RuntimeFlagsPatch,
    test: Effect.Effect<A, E, R>
  ) => 
    test.pipe(Effect.withRuntimeFlagsPatch(flags)),
  
  // Test with metrics enabled
  withMetrics: <A, E, R>(test: Effect.Effect<A, E, R>) =>
    TestUtils.withFlags(
      RuntimeFlagsPatch.enable(RuntimeFlags.RuntimeMetrics),
      test
    ),
  
  // Test with supervision enabled
  withSupervision: <A, E, R>(test: Effect.Effect<A, E, R>) =>
    TestUtils.withFlags(
      RuntimeFlagsPatch.enable(RuntimeFlags.OpSupervision),
      test
    ),
  
  // Test without interruption
  uninterruptible: <A, E, R>(test: Effect.Effect<A, E, R>) =>
    TestUtils.withFlags(
      RuntimeFlagsPatch.disable(RuntimeFlags.Interruption),
      test
    )
}

// Usage in tests
describe("RuntimeFlags Integration", () => {
  it("should collect metrics during operation", () =>
    Effect.gen(function* () {
      const metrics = yield* performOperationWithMetrics()
      expect(metrics.operationCount).toBeGreaterThan(0)
    }).pipe(
      TestUtils.withMetrics,
      Effect.runPromise
    )
  )
  
  it("should handle uninterruptible operations", () =>
    Effect.gen(function* () {
      const result = yield* criticalOperation()
      expect(result).toBeDefined()
    }).pipe(
      TestUtils.uninterruptible,
      Effect.runPromise
    )
  )
})
```

### Integration with Observability

```typescript
// Observability-enhanced patch operations
const observablePatch = <A, E, R>(
  patch: RuntimeFlagsPatch.RuntimeFlagsPatch,
  operation: string,
  effect: Effect.Effect<A, E, R>
): Effect.Effect<A, E, R> =>
  Effect.gen(function* () {
    const enabledFlags = RuntimeFlagsPatch.enabledSet(patch)
    const disabledFlags = RuntimeFlagsPatch.disabledSet(patch)
    
    yield* Logger.debug(`Applying runtime flags patch for ${operation}`, {
      enabled: Array.from(enabledFlags),
      disabled: Array.from(disabledFlags)
    })
    
    const startTime = yield* Clock.currentTimeMillis
    
    const result = yield* effect.pipe(
      Effect.withRuntimeFlagsPatch(patch),
      Effect.withMetric(
        Metric.timer("runtime_flags_operation_duration").tagged("operation", operation)
      )
    )
    
    const endTime = yield* Clock.currentTimeMillis
    
    yield* Logger.debug(`Runtime flags patch completed for ${operation}`, {
      duration: endTime - startTime,
      success: true
    })
    
    return result
  })

// Usage with observability
const monitoredCriticalOperation = <A, E, R>(
  operation: Effect.Effect<A, E, R>
): Effect.Effect<A, E, R> =>
  observablePatch(
    RuntimeFlagsPatch.disable(RuntimeFlags.Interruption),
    "critical-operation",
    operation
  )
```

## Conclusion

RuntimeFlagsPatch provides efficient, composable, and safe runtime flag modifications for Effect applications. It enables fine-grained control over runtime behavior with automatic cleanup and comprehensive composition capabilities.

Key benefits:
- **Efficiency**: Patches are lightweight representations that can be applied with minimal overhead
- **Composability**: Multiple patches can be combined using various strategies (sequential, parallel, conditional)
- **Safety**: Automatic cleanup and resource management prevent flag leaks and inconsistent state
- **Flexibility**: Support for inversion, conditional application, and integration with configuration systems

RuntimeFlagsPatch is essential for applications that need dynamic runtime behavior control, debugging capabilities, performance optimization, or environment-specific configurations.