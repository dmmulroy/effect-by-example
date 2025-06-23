# FiberRefsPatch: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem FiberRefsPatch Solves

Imagine you're building a system that needs to manage fiber-local state efficiently. Traditional approaches force you to choose between:

```typescript
// Traditional approach 1: Copy entire state (inefficient)
const newFiberRefs = { ...oldFiberRefs, logger: newLogger, metrics: newMetrics }

// Traditional approach 2: Mutate state directly (unsafe)
fiberRefs.logger = newLogger
fiberRefs.metrics = newMetrics
```

This approach leads to:
- **Memory Waste** - Copying entire fiber state when only small changes are needed
- **Performance Issues** - Full state copies on every update are expensive
- **State Drift** - Direct mutations can cause inconsistent state across fibers
- **Complex Synchronization** - Managing state propagation becomes error-prone

### The FiberRefsPatch Solution

FiberRefsPatch captures changes to fiber-local state as immutable, composable patches that can be efficiently applied and combined:

```typescript
import { FiberRefsPatch } from "effect"

// Capture changes as a patch instead of copying state
const patch = FiberRefsPatch.diff(oldFiberRefs, newFiberRefs)

// Apply patch efficiently to target fiber state
const updatedRefs = FiberRefsPatch.patch(patch, fiberId, targetFiberRefs)

// Combine multiple patches for batch operations
const combinedPatch = FiberRefsPatch.combine(patch1, patch2)
```

### Key Concepts

**Patch**: A representation of changes to fiber-local state that can be applied, combined, and reversed

**Diff**: The operation that computes the differences between two fiber state snapshots

**Apply**: The operation that efficiently applies a patch to existing fiber state

**Composition**: The ability to combine multiple patches into a single, atomic operation

## Basic Usage Patterns

### Pattern 1: Creating Patches from State Changes

```typescript
import { Effect, FiberRef, FiberRefsPatch } from "effect"

// Create some fiber-local state
const makeAppState = Effect.gen(function* () {
  const logLevel = yield* FiberRef.make("info")
  const requestId = yield* FiberRef.make("")
  const userId = yield* FiberRef.make<string | null>(null)
  
  return { logLevel, requestId, userId }
})

// Capture state changes as a patch
const captureConfigChanges = (newLogLevel: string, newRequestId: string) =>
  Effect.gen(function* () {
    const { logLevel, requestId } = yield* makeAppState
    
    return yield* Effect.diffFiberRefs(
      Effect.gen(function* () {
        yield* FiberRef.set(logLevel, newLogLevel)
        yield* FiberRef.set(requestId, newRequestId)
        return "changes applied"
      })
    )
  })
```

### Pattern 2: Applying Patches to Fiber State

```typescript
import { Effect, FiberRefsPatch } from "effect"

// Apply a captured patch to current fiber
const applyStatePatch = (patch: FiberRefsPatch.FiberRefsPatch) =>
  Effect.patchFiberRefs(patch)

// Usage in a workflow
const processWithPatch = (patch: FiberRefsPatch.FiberRefsPatch) =>
  Effect.gen(function* () {
    // Apply the patch to current fiber
    yield* Effect.patchFiberRefs(patch)
    
    // Continue with patched state
    return yield* doSomeWork()
  })

const doSomeWork = () => Effect.succeed("work completed")
```

### Pattern 3: Combining Multiple Patches

```typescript
import { FiberRefsPatch } from "effect"

// Helper to combine multiple patches into one atomic update
const combinePatchUpdates = (
  patches: ReadonlyArray<FiberRefsPatch.FiberRefsPatch>
) => patches.reduce(FiberRefsPatch.combine, FiberRefsPatch.empty)

// Usage for batch operations
const batchStateUpdates = (
  logPatch: FiberRefsPatch.FiberRefsPatch,
  metricsPatch: FiberRefsPatch.FiberRefsPatch,
  tracingPatch: FiberRefsPatch.FiberRefsPatch
) => {
  const combinedPatch = FiberRefsPatch.combine(
    FiberRefsPatch.combine(logPatch, metricsPatch),
    tracingPatch
  )
  
  return Effect.patchFiberRefs(combinedPatch)
}
```

## Real-World Examples

### Example 1: Request Context Management

Managing request-scoped state efficiently across service layers:

```typescript
import { Context, Effect, FiberRef, FiberRefsPatch, Layer } from "effect"

// Request context state
interface RequestContext {
  readonly requestId: string
  readonly userId: string
  readonly traceId: string
  readonly correlationId: string
}

// Service for managing request context
class RequestContextService extends Context.Tag("RequestContextService")<
  RequestContextService,
  {
    readonly setContext: (context: RequestContext) => Effect.Effect<FiberRefsPatch.FiberRefsPatch>
    readonly updateContext: (
      update: Partial<RequestContext>
    ) => Effect.Effect<FiberRefsPatch.FiberRefsPatch>
    readonly getCurrentContext: () => Effect.Effect<RequestContext>
    readonly withContext: <A, E, R>(
      context: RequestContext,
      effect: Effect.Effect<A, E, R>
    ) => Effect.Effect<A, E, R>
  }
>() {}

// Implementation that uses FiberRefsPatch for efficient updates
const makeRequestContextService = Effect.gen(function* () {
  const requestIdRef = yield* FiberRef.make("")
  const userIdRef = yield* FiberRef.make("")
  const traceIdRef = yield* FiberRef.make("")
  const correlationIdRef = yield* FiberRef.make("")

  const setContext = (context: RequestContext) =>
    Effect.diffFiberRefs(
      Effect.gen(function* () {
        yield* FiberRef.set(requestIdRef, context.requestId)
        yield* FiberRef.set(userIdRef, context.userId)
        yield* FiberRef.set(traceIdRef, context.traceId)
        yield* FiberRef.set(correlationIdRef, context.correlationId)
      })
    ).pipe(Effect.map(([patch]) => patch))

  const updateContext = (update: Partial<RequestContext>) =>
    Effect.gen(function* () {
      const current = yield* getCurrentContext()
      const newContext: RequestContext = { ...current, ...update }
      return yield* setContext(newContext)
    })

  const getCurrentContext = () =>
    Effect.gen(function* () {
      const requestId = yield* FiberRef.get(requestIdRef)
      const userId = yield* FiberRef.get(userIdRef)
      const traceId = yield* FiberRef.get(traceIdRef)
      const correlationId = yield* FiberRef.get(correlationIdRef)
      
      return { requestId, userId, traceId, correlationId }
    })

  const withContext = <A, E, R>(
    context: RequestContext,
    effect: Effect.Effect<A, E, R>
  ) =>
    Effect.gen(function* () {
      const patch = yield* setContext(context)
      return yield* Effect.patchFiberRefs(patch).pipe(
        Effect.zipRight(effect)
      )
    })

  return RequestContextService.of({
    setContext,
    updateContext,
    getCurrentContext,
    withContext
  })
})

// Layer that provides the service
const RequestContextLive = Layer.effect(RequestContextService, makeRequestContextService)

// Usage in HTTP request handler
const handleRequest = (requestData: { id: string; userId: string }) =>
  Effect.gen(function* () {
    const contextService = yield* RequestContextService
    
    // Create request context patch
    const contextPatch = yield* contextService.setContext({
      requestId: requestData.id,
      userId: requestData.userId,
      traceId: `trace-${requestData.id}`,
      correlationId: `corr-${Date.now()}`
    })
    
    // Process request with context
    return yield* Effect.patchFiberRefs(contextPatch).pipe(
      Effect.zipRight(processRequest())
    )
  })

const processRequest = () =>
  Effect.gen(function* () {
    const contextService = yield* RequestContextService
    const context = yield* contextService.getCurrentContext()
    
    console.log(`Processing request ${context.requestId} for user ${context.userId}`)
    return `Request ${context.requestId} processed successfully`
  })
```

### Example 2: Layer State Isolation with Performance Optimization

Using FiberRefsPatch to efficiently manage layer-specific state without full context copying:

```typescript
import { Context, Effect, FiberRef, FiberRefsPatch, Layer, Scope } from "effect"

interface DatabaseConfig {
  readonly connectionString: string
  readonly maxConnections: number
  readonly queryTimeout: number
}

interface CacheConfig {
  readonly ttl: number
  readonly maxEntries: number
  readonly evictionPolicy: "lru" | "fifo"
}

// Service that efficiently manages layer configuration
class LayerConfigManager extends Context.Tag("LayerConfigManager")<
  LayerConfigManager,
  {
    readonly captureConfigChanges: <A, E, R>(
      effect: Effect.Effect<A, E, R>
    ) => Effect.Effect<[FiberRefsPatch.FiberRefsPatch, A], E, R>
    readonly applyConfigSnapshot: (
      patch: FiberRefsPatch.FiberRefsPatch,
      scope: Scope.Scope
    ) => Effect.Effect<void>
    readonly createIsolatedLayer: <I, E, R>(
      baseLayer: Layer.Layer<I, E, R>,
      configPatch: FiberRefsPatch.FiberRefsPatch
    ) => Layer.Layer<I, E, R>
  }
>() {}

const makeLayerConfigManager = Effect.gen(function* () {
  const dbConfigRef = yield* FiberRef.make<DatabaseConfig>({
    connectionString: "default://localhost",
    maxConnections: 10,
    queryTimeout: 5000
  })
  
  const cacheConfigRef = yield* FiberRef.make<CacheConfig>({
    ttl: 300000,
    maxEntries: 1000,
    evictionPolicy: "lru"
  })

  const captureConfigChanges = <A, E, R>(effect: Effect.Effect<A, E, R>) =>
    Effect.diffFiberRefs(effect)

  const applyConfigSnapshot = (patch: FiberRefsPatch.FiberRefsPatch, scope: Scope.Scope) =>
    Effect.gen(function* () {
      // Apply patch and set up revert on scope cleanup
      yield* Effect.patchFiberRefs(patch)
      
      // Capture current state to revert later
      const currentRefs = yield* Effect.fiberRefs
      const revertPatch = FiberRefsPatch.diff(currentRefs, currentRefs)
      
      yield* Scope.addFinalizer(
        scope,
        Effect.patchFiberRefs(revertPatch)
      )
    })

  const createIsolatedLayer = <I, E, R>(
    baseLayer: Layer.Layer<I, E, R>,
    configPatch: FiberRefsPatch.FiberRefsPatch
  ) =>
    Layer.scopedContext(
      Effect.gen(function* () {
        const scope = yield* Effect.scope
        yield* applyConfigSnapshot(configPatch, scope)
        return yield* Layer.buildWithScope(baseLayer, scope)
      })
    )

  return LayerConfigManager.of({
    captureConfigChanges,
    applyConfigSnapshot,
    createIsolatedLayer
  })
})

// Usage: Creating tenant-specific database layers
const createTenantDatabaseLayer = (tenantId: string) =>
  Effect.gen(function* () {
    const configManager = yield* LayerConfigManager
    
    // Capture tenant-specific config changes
    const [configPatch] = yield* configManager.captureConfigChanges(
      Effect.gen(function* () {
        const dbConfigRef = yield* FiberRef.make<DatabaseConfig>({
          connectionString: "default://localhost",
          maxConnections: 10,
          queryTimeout: 5000
        })
        
        // Update config for this tenant
        yield* FiberRef.update(dbConfigRef, config => ({
          ...config,
          connectionString: `tenant://db-${tenantId}`,
          maxConnections: tenantId === "premium" ? 50 : 20
        }))
      })
    )
    
    // Create isolated layer with tenant config
    return configManager.createIsolatedLayer(BaseDatabaseLayer, configPatch)
  })

const BaseDatabaseLayer = Layer.succeed(Context.Tag<"Database">(), "database-service")
```

### Example 3: Distributed State Synchronization

Using FiberRefsPatch for efficient state synchronization across distributed fibers:

```typescript
import { Effect, FiberRef, FiberRefsPatch, Queue, Ref } from "effect"

interface StateSync {
  readonly nodeId: string
  readonly version: number
  readonly patches: ReadonlyArray<FiberRefsPatch.FiberRefsPatch>
}

// Distributed state synchronization service
class DistributedStateService extends Context.Tag("DistributedStateService")<
  DistributedStateService,
  {
    readonly broadcastChanges: (patch: FiberRefsPatch.FiberRefsPatch) => Effect.Effect<void>
    readonly syncFromRemote: (sync: StateSync) => Effect.Effect<void>
    readonly startSyncWorker: () => Effect.Effect<void, never, never>
    readonly createSyncPoint: () => Effect.Effect<StateSync>
  }
>() {}

const makeDistributedStateService = (nodeId: string) =>
  Effect.gen(function* () {
    const versionRef = yield* Ref.make(0)
    const patchQueueRef = yield* Ref.make<ReadonlyArray<FiberRefsPatch.FiberRefsPatch>>([])
    const syncQueue = yield* Queue.unbounded<StateSync>()

    const broadcastChanges = (patch: FiberRefsPatch.FiberRefsPatch) =>
      Effect.gen(function* () {
        // Add to local patch queue for batching
        yield* Ref.update(patchQueueRef, patches => [...patches, patch])
        
        // Increment version
        const newVersion = yield* Ref.updateAndGet(versionRef, v => v + 1)
        
        // Create sync message
        const patches = yield* Ref.get(patchQueueRef)
        const syncMessage: StateSync = { nodeId, version: newVersion, patches }
        
        // Queue for broadcasting (would normally send to network)
        yield* Queue.offer(syncQueue, syncMessage)
        
        // Clear patch queue after broadcast
        yield* Ref.set(patchQueueRef, [])
      })

    const syncFromRemote = (sync: StateSync) =>
      Effect.gen(function* () {
        if (sync.nodeId === nodeId) return // Skip own messages
        
        // Combine all patches into single atomic update
        const combinedPatch = sync.patches.reduce(
          FiberRefsPatch.combine,
          FiberRefsPatch.empty
        )
        
        // Apply the combined patch atomically
        yield* Effect.patchFiberRefs(combinedPatch)
        
        // Update local version if remote is newer
        const currentVersion = yield* Ref.get(versionRef)
        if (sync.version > currentVersion) {
          yield* Ref.set(versionRef, sync.version)
        }
      })

    const startSyncWorker = () =>
      Effect.gen(function* () {
        while (true) {
          const syncMessage = yield* Queue.take(syncQueue)
          yield* syncFromRemote(syncMessage)
          
          // In real implementation, would broadcast to network here
          console.log(`Syncing state from node ${syncMessage.nodeId}`)
        }
      }).pipe(Effect.forever, Effect.forkDaemon, Effect.asVoid)

    const createSyncPoint = () =>
      Effect.gen(function* () {
        const version = yield* Ref.get(versionRef)
        const patches = yield* Ref.get(patchQueueRef)
        
        return { nodeId, version, patches }
      })

    return DistributedStateService.of({
      broadcastChanges,
      syncFromRemote,
      startSyncWorker,
      createSyncPoint
    })
  })

// Usage in distributed application
const distributedWorkflow = (nodeId: string) =>
  Effect.gen(function* () {
    const stateService = yield* DistributedStateService
    const userPrefsRef = yield* FiberRef.make({ theme: "light", lang: "en" })
    
    // Start background sync worker
    yield* stateService.startSyncWorker()
    
    // Make local state changes and broadcast them
    const [patch] = yield* Effect.diffFiberRefs(
      Effect.gen(function* () {
        yield* FiberRef.update(userPrefsRef, prefs => ({
          ...prefs,
          theme: "dark"
        }))
      })
    )
    
    // Broadcast the change to other nodes
    yield* stateService.broadcastChanges(patch)
    
    return "State synchronized across nodes"
  })
```

## Advanced Features Deep Dive

### Feature 1: Efficient Patch Composition

FiberRefsPatch supports efficient composition through the `AndThen` operation, allowing multiple patches to be combined without intermediate state materialization.

#### Basic Patch Composition

```typescript
import { FiberRefsPatch } from "effect"

// Compose patches sequentially
const createComposedPatch = (
  patch1: FiberRefsPatch.FiberRefsPatch,
  patch2: FiberRefsPatch.FiberRefsPatch,
  patch3: FiberRefsPatch.FiberRefsPatch
) => {
  // Patches are combined in order - patch1, then patch2, then patch3
  return FiberRefsPatch.combine(
    FiberRefsPatch.combine(patch1, patch2),
    patch3
  )
}

// More readable with pipe-style composition
const composePatchesPipe = (
  patches: ReadonlyArray<FiberRefsPatch.FiberRefsPatch>
) => patches.reduce(FiberRefsPatch.combine, FiberRefsPatch.empty)
```

#### Real-World Patch Composition Example

```typescript
import { Effect, FiberRef, FiberRefsPatch } from "effect"

// Transaction-like state management using patch composition
const createTransactionalUpdate = () =>
  Effect.gen(function* () {
    const accountBalanceRef = yield* FiberRef.make(1000)
    const transactionLogRef = yield* FiberRef.make<string[]>([])
    const lastUpdatedRef = yield* FiberRef.make(new Date())

    // Helper to capture individual operation patches
    const captureOperation = <A>(
      operation: Effect.Effect<A, any, any>,
      description: string
    ) =>
      Effect.diffFiberRefs(
        Effect.gen(function* () {
          const result = yield* operation
          yield* FiberRef.update(transactionLogRef, log => [...log, description])
          yield* FiberRef.set(lastUpdatedRef, new Date())
          return result
        })
      ).pipe(Effect.map(([patch, result]) => ({ patch, result, description })))

    // Compose multiple operations into atomic transaction
    const executeTransaction = (amount: number) =>
      Effect.gen(function* () {
        // Capture each operation as a separate patch
        const debitOp = yield* captureOperation(
          FiberRef.update(accountBalanceRef, balance => balance - amount),
          `Debit: $${amount}`
        )
        
        const feesOp = yield* captureOperation(
          FiberRef.update(accountBalanceRef, balance => balance - 2.50),
          "Transaction fee: $2.50"
        )
        
        const auditOp = yield* captureOperation(
          Effect.succeed("audit-logged"),
          "Audit trail updated"
        )

        // Combine all patches into single atomic update
        const transactionPatch = FiberRefsPatch.combine(
          FiberRefsPatch.combine(debitOp.patch, feesOp.patch),
          auditOp.patch
        )

        // Apply the entire transaction atomically
        yield* Effect.patchFiberRefs(transactionPatch)
        
        return {
          operations: [debitOp.description, feesOp.description, auditOp.description],
          finalBalance: yield* FiberRef.get(accountBalanceRef)
        }
      })

    return { executeTransaction, accountBalanceRef, transactionLogRef }
  })
```

### Feature 2: Patch Reversal and Rollback

FiberRefsPatch supports creating inverse patches for rollback scenarios.

#### Advanced Rollback Pattern

```typescript
import { Effect, FiberRef, FiberRefsPatch, Ref } from "effect"

// Rollback-capable state manager
const createRollbackableState = <T>(initialValue: T) =>
  Effect.gen(function* () {
    const stateRef = yield* FiberRef.make(initialValue)
    const checkpointStack = yield* Ref.make<ReadonlyArray<FiberRefsPatch.FiberRefsPatch>>([])

    const createCheckpoint = () =>
      Effect.gen(function* () {
        const currentRefs = yield* Effect.fiberRefs
        
        // Create a no-op patch as checkpoint marker
        const checkpointPatch = FiberRefsPatch.empty
        
        yield* Ref.update(checkpointStack, stack => [...stack, checkpointPatch])
        return currentRefs
      })

    const updateWithRollback = <A>(
      update: Effect.Effect<A, any, any>
    ) =>
      Effect.gen(function* () {
        // Capture current state before update
        const beforeRefs = yield* Effect.fiberRefs
        
        // Execute the update and capture changes
        const [patch, result] = yield* Effect.diffFiberRefs(update)
        
        // Create reverse patch for rollback
        const afterRefs = yield* Effect.fiberRefs
        const rollbackPatch = FiberRefsPatch.diff(afterRefs, beforeRefs)
        
        // Store rollback patch
        yield* Ref.update(checkpointStack, stack => [...stack, rollbackPatch])
        
        return { result, rollbackPatch }
      })

    const rollbackToLastCheckpoint = () =>
      Effect.gen(function* () {
        const stack = yield* Ref.get(checkpointStack)
        
        if (stack.length === 0) {
          return Effect.fail(new Error("No checkpoints available"))
        }
        
        const rollbackPatch = stack[stack.length - 1]
        
        // Apply rollback patch
        yield* Effect.patchFiberRefs(rollbackPatch)
        
        // Remove used checkpoint
        yield* Ref.update(checkpointStack, stack => stack.slice(0, -1))
        
        return "Rollback completed"
      })

    return {
      stateRef,
      createCheckpoint,
      updateWithRollback,
      rollbackToLastCheckpoint
    }
  })
```

### Feature 3: Memory-Efficient State Propagation

FiberRefsPatch enables memory-efficient state propagation across fiber boundaries.

#### Optimized State Inheritance

```typescript
import { Effect, Fiber, FiberRef, FiberRefsPatch } from "effect"

// Efficient parent-child state inheritance
const createStateInheritanceSystem = () =>
  Effect.gen(function* () {
    const parentConfigRef = yield* FiberRef.make({ 
      logLevel: "info", 
      timeout: 5000, 
      retries: 3 
    })

    // Spawn child fiber with selective state inheritance
    const spawnChildWithPartialState = <A, E>(
      childEffect: Effect.Effect<A, E, any>,
      stateOverrides: Partial<{ logLevel: string; timeout: number; retries: number }>
    ) =>
      Effect.gen(function* () {
        // Capture minimal patch for child-specific overrides
        const [overridePatch] = yield* Effect.diffFiberRefs(
          Effect.gen(function* () {
            if (stateOverrides.logLevel) {
              yield* FiberRef.update(parentConfigRef, config => ({
                ...config,
                logLevel: stateOverrides.logLevel!
              }))
            }
            if (stateOverrides.timeout) {
              yield* FiberRef.update(parentConfigRef, config => ({
                ...config,
                timeout: stateOverrides.timeout!
              }))
            }
            if (stateOverrides.retries) {
              yield* FiberRef.update(parentConfigRef, config => ({
                ...config,
                retries: stateOverrides.retries!
              }))
            }
          })
        )

        // Fork child with minimal state patch
        return yield* Effect.gen(function* () {
          yield* Effect.patchFiberRefs(overridePatch)
          return yield* childEffect
        }).pipe(Effect.fork)
      })

    // Bulk spawn optimization
    const spawnMultipleChildren = <A, E>(
      children: ReadonlyArray<{
        effect: Effect.Effect<A, E, any>
        overrides: Partial<{ logLevel: string; timeout: number; retries: number }>
      }>
    ) =>
      Effect.gen(function* () {
        // Pre-compute patches for all children
        const childPatches = yield* Effect.all(
          children.map(({ overrides }) =>
            Effect.diffFiberRefs(
              Effect.gen(function* () {
                if (overrides.logLevel) {
                  yield* FiberRef.update(parentConfigRef, config => ({
                    ...config,
                    logLevel: overrides.logLevel!
                  }))
                }
                if (overrides.timeout) {
                  yield* FiberRef.update(parentConfigRef, config => ({
                    ...config,
                    timeout: overrides.timeout!
                  }))
                }
                if (overrides.retries) {
                  yield* FiberRef.update(parentConfigRef, config => ({
                    ...config,
                    retries: overrides.retries!
                  }))
                }
              })
            ).pipe(Effect.map(([patch]) => patch))
          )
        )

        // Fork all children with their respective patches
        return yield* Effect.all(
          children.map(({ effect }, index) =>
            Effect.gen(function* () {
              yield* Effect.patchFiberRefs(childPatches[index])
              return yield* effect
            }).pipe(Effect.fork)
          )
        )
      })

    return {
      spawnChildWithPartialState,
      spawnMultipleChildren,
      parentConfigRef
    }
  })
```

## Practical Patterns & Best Practices

### Pattern 1: Batch State Updates

Efficient batching of multiple state changes into single atomic operations:

```typescript
import { Effect, FiberRef, FiberRefsPatch, Array as Arr } from "effect"

// Batch state updater utility
const createBatchStateUpdater = () => {
  const batchUpdates = <T extends Record<string, any>>(
    updates: ReadonlyArray<{
      ref: FiberRef.FiberRef<T[keyof T]>
      value: T[keyof T]
    }>
  ) =>
    Effect.gen(function* () {
      // Capture all updates as individual patches
      const patches = yield* Effect.all(
        updates.map(({ ref, value }) =>
          Effect.diffFiberRefs(FiberRef.set(ref, value)).pipe(
            Effect.map(([patch]) => patch)
          )
        )
      )

      // Combine all patches into single atomic update
      const batchPatch = patches.reduce(FiberRefsPatch.combine, FiberRefsPatch.empty)
      
      // Apply batch update atomically
      yield* Effect.patchFiberRefs(batchPatch)
      
      return patches.length
    })

  const batchFunctionalUpdates = <T extends Record<string, any>>(
    updates: ReadonlyArray<{
      ref: FiberRef.FiberRef<T[keyof T]>
      update: (current: T[keyof T]) => T[keyof T]
    }>
  ) =>
    Effect.gen(function* () {
      const patches = yield* Effect.all(
        updates.map(({ ref, update }) =>
          Effect.diffFiberRefs(FiberRef.update(ref, update)).pipe(
            Effect.map(([patch]) => patch)
          )
        )
      )

      const batchPatch = patches.reduce(FiberRefsPatch.combine, FiberRefsPatch.empty)
      yield* Effect.patchFiberRefs(batchPatch)
      
      return patches.length
    })

  return { batchUpdates, batchFunctionalUpdates }
}

// Usage example
const userSettingsUpdate = Effect.gen(function* () {
  const themeRef = yield* FiberRef.make("light")
  const languageRef = yield* FiberRef.make("en")
  const notificationsRef = yield* FiberRef.make(true)
  
  const { batchUpdates } = createBatchStateUpdater()
  
  // Update all settings atomically
  const updatesCount = yield* batchUpdates([
    { ref: themeRef, value: "dark" },
    { ref: languageRef, value: "es" },
    { ref: notificationsRef, value: false }
  ])
  
  console.log(`Applied ${updatesCount} settings updates atomically`)
  
  return {
    theme: yield* FiberRef.get(themeRef),
    language: yield* FiberRef.get(languageRef),
    notifications: yield* FiberRef.get(notificationsRef)
  }
})
```

### Pattern 2: State Migration and Versioning

Managing state schema changes and migrations using patches:

```typescript
import { Effect, FiberRef, FiberRefsPatch, Schema } from "effect"

// State versioning system
interface StateV1 {
  readonly version: 1
  readonly username: string
  readonly email: string
}

interface StateV2 {
  readonly version: 2
  readonly user: {
    readonly name: string
    readonly email: string
    readonly profile: {
      readonly avatar?: string
    }
  }
}

type AppState = StateV1 | StateV2

const createMigrationSystem = () => {
  const stateRef = yield* FiberRef.make<AppState>({
    version: 1,
    username: "defaultUser",
    email: "user@example.com"
  })

  // Migration functions
  const migrateV1ToV2 = (v1State: StateV1): StateV2 => ({
    version: 2,
    user: {
      name: v1State.username,
      email: v1State.email,
      profile: {}
    }
  })

  // Create migration patch
  const createMigrationPatch = (targetVersion: number) =>
    Effect.gen(function* () {
      const currentState = yield* FiberRef.get(stateRef)
      
      let migratedState: AppState = currentState
      
      if (currentState.version === 1 && targetVersion >= 2) {
        migratedState = migrateV1ToV2(currentState as StateV1)
      }
      
      // Create patch for migration
      const [migrationPatch] = yield* Effect.diffFiberRefs(
        FiberRef.set(stateRef, migratedState)
      )
      
      return migrationPatch
    })

  // Apply migration with rollback capability
  const migrateWithRollback = (targetVersion: number) =>
    Effect.gen(function* () {
      const beforeState = yield* FiberRef.get(stateRef)
      const beforeRefs = yield* Effect.fiberRefs
      
      // Create and apply migration patch
      const migrationPatch = yield* createMigrationPatch(targetVersion)
      yield* Effect.patchFiberRefs(migrationPatch)
      
      const afterRefs = yield* Effect.fiberRefs
      const rollbackPatch = FiberRefsPatch.diff(afterRefs, beforeRefs)
      
      return {
        beforeState,
        afterState: yield* FiberRef.get(stateRef),
        rollback: () => Effect.patchFiberRefs(rollbackPatch)
      }
    })

  return {
    stateRef,
    createMigrationPatch,
    migrateWithRollback
  }
}
```

### Pattern 3: Performance-Optimized State Synchronization

Minimizing memory allocations and CPU overhead in high-frequency state updates:

```typescript
import { Effect, FiberRef, FiberRefsPatch, Ref, Schedule } from "effect"

// High-performance state sync system
const createHighPerfStateSync = () =>
  Effect.gen(function* () {
    const metricsRef = yield* FiberRef.make({
      requestCount: 0,
      errorCount: 0,
      avgResponseTime: 0
    })
    
    const pendingPatchesRef = yield* Ref.make<ReadonlyArray<FiberRefsPatch.FiberRefsPatch>>([])
    const flushInProgress = yield* Ref.make(false)

    // Optimized batch collection
    const addPendingUpdate = (patch: FiberRefsPatch.FiberRefsPatch) =>
      Effect.gen(function* () {
        yield* Ref.update(pendingPatchesRef, patches => [...patches, patch])
        
        // Trigger flush if we have enough patches or if none in progress
        const patches = yield* Ref.get(pendingPatchesRef)
        const inProgress = yield* Ref.get(flushInProgress)
        
        if (patches.length >= 10 || (!inProgress && patches.length > 0)) {
          yield* flushPendingUpdates()
        }
      })

    // Optimized flush operation
    const flushPendingUpdates = () =>
      Effect.gen(function* () {
        const alreadyFlushing = yield* Ref.getAndSet(flushInProgress, true)
        if (alreadyFlushing) return 0 // Skip if already flushing
        
        const patches = yield* Ref.getAndSet(pendingPatchesRef, [])
        
        if (patches.length === 0) {
          yield* Ref.set(flushInProgress, false)
          return 0
        }
        
        // Combine patches efficiently
        const combinedPatch = patches.reduce(FiberRefsPatch.combine, FiberRefsPatch.empty)
        
        // Apply combined patch
        yield* Effect.patchFiberRefs(combinedPatch)
        
        yield* Ref.set(flushInProgress, false)
        return patches.length
      })

    // Periodic flush to handle low-frequency updates
    const startPeriodicFlush = () =>
      flushPendingUpdates().pipe(
        Effect.repeat(Schedule.fixed("100 millis")),
        Effect.forkDaemon,
        Effect.asVoid
      )

    // High-frequency metric updates
    const updateMetrics = (
      requestDelta: number = 0,
      errorDelta: number = 0,
      responseTime?: number
    ) =>
      Effect.gen(function* () {
        const [patch] = yield* Effect.diffFiberRefs(
          FiberRef.update(metricsRef, current => ({
            requestCount: current.requestCount + requestDelta,
            errorCount: current.errorCount + errorDelta,
            avgResponseTime: responseTime ?? current.avgResponseTime
          }))
        )
        
        yield* addPendingUpdate(patch)
      })

    return {
      updateMetrics,
      flushPendingUpdates,
      startPeriodicFlush,
      metricsRef
    }
  })
```

## Integration Examples

### Integration with Layers and Runtime

FiberRefsPatch integrates seamlessly with Effect's Layer system for efficient dependency injection:

```typescript
import { Context, Effect, FiberRef, FiberRefsPatch, Layer, Runtime } from "effect"

// Service configuration using FiberRefsPatch
class DatabaseService extends Context.Tag("DatabaseService")<
  DatabaseService,
  {
    readonly query: <T>(sql: string) => Effect.Effect<T>
    readonly getConnectionInfo: () => Effect.Effect<string>
  }
>() {}

class LoggerService extends Context.Tag("LoggerService")<
  LoggerService,
  {
    readonly log: (message: string) => Effect.Effect<void>
    readonly getLevel: () => Effect.Effect<string>
  }
>() {}

// Configuration layer that uses FiberRefsPatch
const ConfigurableApplicationLayer = Layer.effect(
  Context.Tag<"App">(),
  Effect.gen(function* () {
    // Create configuration FiberRefs
    const dbUrlRef = yield* FiberRef.make("postgresql://localhost:5432/dev")
    const logLevelRef = yield* FiberRef.make("info")
    const featureFlagsRef = yield* FiberRef.make<Record<string, boolean>>({})

    // Create a configuration snapshot function
    const captureConfig = () =>
      Effect.gen(function* () {
        const dbUrl = yield* FiberRef.get(dbUrlRef)
        const logLevel = yield* FiberRef.get(logLevelRef)
        const featureFlags = yield* FiberRef.get(featureFlagsRef)
        
        return { dbUrl, logLevel, featureFlags }
      })

    // Configuration update with patch capture
    const updateConfigWithPatch = (updates: {
      dbUrl?: string
      logLevel?: string
      featureFlags?: Record<string, boolean>
    }) =>
      Effect.diffFiberRefs(
        Effect.gen(function* () {
          if (updates.dbUrl) {
            yield* FiberRef.set(dbUrlRef, updates.dbUrl)
          }
          if (updates.logLevel) {
            yield* FiberRef.set(logLevelRef, updates.logLevel)
          }
          if (updates.featureFlags) {
            yield* FiberRef.set(featureFlagsRef, updates.featureFlags)
          }
        })
      )

    // Create child runtime with specific configuration
    const createChildRuntime = (configOverrides: {
      dbUrl?: string
      logLevel?: string
      featureFlags?: Record<string, boolean>
    }) =>
      Effect.gen(function* () {
        const [configPatch] = yield* updateConfigWithPatch(configOverrides)
        
        return yield* Effect.runtime<any>().pipe(
          Effect.map(runtime => ({
            ...runtime,
            // Apply config patch to child runtime
            fiberRefs: FiberRefsPatch.patch(configPatch, runtime.runtimeFlags, runtime.fiberRefs)
          }))
        )
      })

    return Context.Tag<"App">().of({
      captureConfig,
      updateConfigWithPatch,
      createChildRuntime
    })
  })
)

// Usage with different configurations per tenant
const runTenantWorkflow = (tenantId: string) =>
  Effect.gen(function* () {
    const app = yield* Context.Tag<"App">()
    
    // Create tenant-specific runtime
    const tenantRuntime = yield* app.createChildRuntime({
      dbUrl: `postgresql://localhost:5432/tenant_${tenantId}`,
      logLevel: tenantId === "premium" ? "debug" : "info",
      featureFlags: {
        advancedFeatures: tenantId === "premium",
        betaFeatures: false
      }
    })

    // Run tenant workflow with isolated configuration
    return yield* Effect.provide(
      tenantWorkflow(tenantId),
      Layer.succeed(Context.Tag<"Runtime">(), tenantRuntime)
    )
  })

const tenantWorkflow = (tenantId: string) =>
  Effect.gen(function* () {
    const app = yield* Context.Tag<"App">()
    const config = yield* app.captureConfig()
    
    console.log(`Tenant ${tenantId} using DB: ${config.dbUrl}`)
    console.log(`Log level: ${config.logLevel}`)
    
    return `Workflow completed for tenant ${tenantId}`
  })
```

### Testing Strategies

Comprehensive testing patterns for FiberRefsPatch-based systems:

```typescript
import { Effect, FiberRef, FiberRefsPatch, TestServices } from "effect"
import { describe, it, expect } from "vitest"

// Test utilities for FiberRefsPatch
const createTestUtils = () => {
  const captureStateSnapshot = () =>
    Effect.gen(function* () {
      const refs = yield* Effect.fiberRefs
      return { refs, timestamp: Date.now() }
    })

  const assertStateEquals = (
    snapshot1: { refs: any; timestamp: number },
    snapshot2: { refs: any; timestamp: number }
  ) => {
    // Compare relevant FiberRef values
    expect(snapshot1.refs).toEqual(snapshot2.refs)
  }

  const createMockPatch = (changes: Record<string, any>) =>
    Effect.gen(function* () {
      const tempRef = yield* FiberRef.make({})
      
      const [patch] = yield* Effect.diffFiberRefs(
        FiberRef.set(tempRef, changes)
      )
      
      return patch
    })

  return {
    captureStateSnapshot,
    assertStateEquals,
    createMockPatch
  }
}

describe("FiberRefsPatch", () => {
  it("should efficiently combine multiple patches", () =>
    Effect.gen(function* () {
      const testRef1 = yield* FiberRef.make("initial1")
      const testRef2 = yield* FiberRef.make("initial2")
      const { captureStateSnapshot } = createTestUtils()

      // Capture initial state
      const initialState = yield* captureStateSnapshot()

      // Create individual patches
      const [patch1] = yield* Effect.diffFiberRefs(
        FiberRef.set(testRef1, "updated1")
      )
      
      const [patch2] = yield* Effect.diffFiberRefs(
        FiberRef.set(testRef2, "updated2")
      )

      // Test patch combination
      const combinedPatch = FiberRefsPatch.combine(patch1, patch2)
      yield* Effect.patchFiberRefs(combinedPatch)

      // Verify both changes applied
      const value1 = yield* FiberRef.get(testRef1)
      const value2 = yield* FiberRef.get(testRef2)
      
      expect(value1).toBe("updated1")
      expect(value2).toBe("updated2")
    }).pipe(Effect.provide(TestServices.TestContext)))

  it("should handle patch reversal correctly", () =>
    Effect.gen(function* () {
      const testRef = yield* FiberRef.make("original")
      
      // Apply a change and capture patch
      const beforeRefs = yield* Effect.fiberRefs
      yield* FiberRef.set(testRef, "modified")
      const afterRefs = yield* Effect.fiberRefs
      
      // Create reverse patch
      const reversePatch = FiberRefsPatch.diff(afterRefs, beforeRefs)
      
      // Apply reverse patch
      yield* Effect.patchFiberRefs(reversePatch)
      
      // Verify state reverted
      const currentValue = yield* FiberRef.get(testRef)
      expect(currentValue).toBe("original")
    }).pipe(Effect.provide(TestServices.TestContext)))

  it("should maintain performance with large patch compositions", () =>
    Effect.gen(function* () {
      const startTime = Date.now()
      
      // Create many FiberRefs and patches
      const refs = yield* Effect.all(
        Array.from({ length: 100 }, (_, i) => FiberRef.make(`value${i}`))
      )
      
      const patches = yield* Effect.all(
        refs.map((ref, i) =>
          Effect.diffFiberRefs(FiberRef.set(ref, `updated${i}`)).pipe(
            Effect.map(([patch]) => patch)
          )
        )
      )
      
      // Combine all patches
      const megaPatch = patches.reduce(FiberRefsPatch.combine, FiberRefsPatch.empty)
      
      // Apply combined patch
      yield* Effect.patchFiberRefs(megaPatch)
      
      const endTime = Date.now()
      const duration = endTime - startTime
      
      // Performance assertion
      expect(duration).toBeLessThan(100) // Should complete in <100ms
      
      // Verify all updates applied
      const values = yield* Effect.all(refs.map(FiberRef.get))
      values.forEach((value, i) => {
        expect(value).toBe(`updated${i}`)
      })
    }).pipe(Effect.provide(TestServices.TestContext)))
})

// Property-based testing for FiberRefsPatch laws
describe("FiberRefsPatch Laws", () => {
  it("should satisfy associativity law for patch combination", () =>
    Effect.gen(function* () {
      const { createMockPatch } = createTestUtils()
      
      const patchA = yield* createMockPatch({ a: 1 })
      const patchB = yield* createMockPatch({ b: 2 })
      const patchC = yield* createMockPatch({ c: 3 })
      
      // (A + B) + C should equal A + (B + C)
      const leftAssoc = FiberRefsPatch.combine(
        FiberRefsPatch.combine(patchA, patchB),
        patchC
      )
      
      const rightAssoc = FiberRefsPatch.combine(
        patchA,
        FiberRefsPatch.combine(patchB, patchC)
      )
      
      // Apply both and verify same result
      const testRef = yield* FiberRef.make({})
      
      const beforeRefs1 = yield* Effect.fiberRefs
      yield* Effect.patchFiberRefs(leftAssoc)
      const afterRefs1 = yield* Effect.fiberRefs
      
      // Reset state
      yield* Effect.setFiberRefs(beforeRefs1)
      
      const beforeRefs2 = yield* Effect.fiberRefs
      yield* Effect.patchFiberRefs(rightAssoc)
      const afterRefs2 = yield* Effect.fiberRefs
      
      // Should produce same result
      expect(afterRefs1).toEqual(afterRefs2)
    }).pipe(Effect.provide(TestServices.TestContext)))

  it("should satisfy identity law for empty patch", () =>
    Effect.gen(function* () {
      const testRef = yield* FiberRef.make("test-value")
      const { createMockPatch } = createTestUtils()
      
      const somePatch = yield* createMockPatch({ test: "data" })
      
      // empty + patch = patch + empty = patch
      const leftIdentity = FiberRefsPatch.combine(FiberRefsPatch.empty, somePatch)
      const rightIdentity = FiberRefsPatch.combine(somePatch, FiberRefsPatch.empty)
      
      const beforeRefs = yield* Effect.fiberRefs
      
      // Test left identity
      yield* Effect.patchFiberRefs(leftIdentity)
      const afterLeft = yield* Effect.fiberRefs
      
      // Reset and test right identity
      yield* Effect.setFiberRefs(beforeRefs)
      yield* Effect.patchFiberRefs(rightIdentity)
      const afterRight = yield* Effect.fiberRefs
      
      // Reset and test original patch
      yield* Effect.setFiberRefs(beforeRefs)
      yield* Effect.patchFiberRefs(somePatch)
      const afterOriginal = yield* Effect.fiberRefs
      
      expect(afterLeft).toEqual(afterOriginal)
      expect(afterRight).toEqual(afterOriginal)
    }).pipe(Effect.provide(TestServices.TestContext)))
})
```

## Conclusion

FiberRefsPatch provides efficient state change representation, memory-optimized state propagation, and composable state updates for Effect applications.

Key benefits:
- **Memory Efficiency**: Patches represent only changes, not full state copies
- **Performance Optimization**: Batch multiple state changes into atomic operations  
- **Composability**: Combine and sequence patches for complex state management
- **Reliability**: Atomic updates prevent partial state corruption
- **Debugging**: Patches provide audit trail of state changes

FiberRefsPatch is essential when building high-performance Effect applications that require efficient fiber-local state management, especially in scenarios involving frequent state updates, state synchronization across fibers, or complex state inheritance patterns.