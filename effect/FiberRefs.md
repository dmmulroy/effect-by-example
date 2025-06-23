# FiberRefs: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem FiberRefs Solves

Managing collections of fiber-local state across complex async workflows is challenging with traditional approaches:

```typescript
// Traditional approach - manual state tracking
class ManualStateManager {
  private state = new Map<string, any>()
  
  async passToAsyncOperation() {
    // ❌ State doesn't automatically propagate
    // ❌ Manual serialization/deserialization needed  
    // ❌ No type safety across fiber boundaries
    const serialized = JSON.stringify(Object.fromEntries(this.state))
    return await someAsyncWork(serialized)
  }
  
  async forkOperation() {
    // ❌ Child operations lose parent state
    // ❌ No automatic merging of child changes back to parent
    return await Promise.all([
      this.childOperation1(),
      this.childOperation2() // These don't see each other's state changes
    ])
  }
}
```

This approach leads to:
- **State Isolation Issues** - Child fibers lose access to parent fiber state
- **Manual Propagation** - Must manually serialize/deserialize state across boundaries
- **No Merge Strategy** - Changes in child fibers don't automatically propagate back
- **Type Unsafety** - No compile-time guarantees about state structure
- **Performance Overhead** - Inefficient copying and serialization

### The FiberRefs Solution

FiberRefs provides a type-safe, performant collection for managing multiple FiberRef values with automatic propagation and merging:

```typescript
import { Effect, FiberRef, FiberRefs } from "effect"

// ✅ Type-safe collection of fiber-local state
export const createApplicationState = Effect.gen(function* () {
  const userId = yield* FiberRef.make<string | null>(null)
  const requestId = yield* FiberRef.make<string>("")
  const logLevel = yield* FiberRef.make<"debug" | "info" | "error">("info")
  
  // Get current fiber state as a collection
  const currentRefs = yield* Effect.getFiberRefs
  return { userId, requestId, logLevel, currentRefs }
})
```

### Key Concepts

**FiberRefs Collection**: A map-like structure that holds multiple FiberRef values with their fiber-specific values and history

**Automatic Propagation**: When fibers fork, their FiberRefs are automatically inherited by child fibers

**Smart Merging**: When child fibers complete, their FiberRef changes are merged back using each FiberRef's join strategy

## Basic Usage Patterns

### Pattern 1: Creating and Reading Collections

```typescript
import { Effect, FiberRef, FiberRefs } from "effect"

export const basicFiberRefsUsage = Effect.gen(function* () {
  // Create some FiberRefs
  const userFiberRef = yield* FiberRef.make("anonymous")
  const countFiberRef = yield* FiberRef.make(0)
  
  // Set values
  yield* FiberRef.set(userFiberRef, "alice")
  yield* FiberRef.set(countFiberRef, 42)
  
  // Get the current collection of all FiberRefs
  const fiberRefs = yield* Effect.getFiberRefs
  
  // Read specific values from the collection
  const user = FiberRefs.getOrDefault(fiberRefs, userFiberRef)
  const count = FiberRefs.getOrDefault(fiberRefs, countFiberRef)
  
  console.log({ user, count }) // { user: "alice", count: 42 }
  return { user, count }
})
```

### Pattern 2: Propagating State Across Fiber Boundaries

```typescript
import { Effect, FiberRef, FiberRefs, Fiber } from "effect"

export const propagateStateExample = Effect.gen(function* () {
  const contextRef = yield* FiberRef.make("main-context")
  
  // Set state in parent fiber
  yield* FiberRef.set(contextRef, "parent-value")
  
  // Fork a child fiber - state automatically propagates
  const childFiber = yield* Effect.fork(
    Effect.gen(function* () {
      // Child sees parent's state
      const value = yield* FiberRef.get(contextRef)
      console.log("Child sees:", value) // "parent-value"
      
      // Child modifies state
      yield* FiberRef.set(contextRef, "child-modified")
      
      // Return current fiber refs for parent to inherit
      return yield* Effect.getFiberRefs
    })
  )
  
  const childRefs = yield* Fiber.join(childFiber)
  
  // Parent can inherit child's changes
  yield* Effect.setFiberRefs(childRefs)
  
  const finalValue = yield* FiberRef.get(contextRef)
  console.log("Parent after inherit:", finalValue) // "child-modified"
})
```

### Pattern 3: Batch Operations on Multiple FiberRefs

```typescript
import { Effect, FiberRef, FiberRefs } from "effect"

export const batchOperationsExample = Effect.gen(function* () {
  // Create multiple FiberRefs
  const refs = yield* Effect.all({
    user: FiberRef.make(""),
    session: FiberRef.make(""),
    permissions: FiberRef.make<string[]>([])
  })
  
  // Get current collection
  const currentRefs = yield* Effect.getFiberRefs
  
  // Read multiple values at once
  const currentState = {
    user: FiberRefs.getOrDefault(currentRefs, refs.user),
    session: FiberRefs.getOrDefault(currentRefs, refs.session),
    permissions: FiberRefs.getOrDefault(currentRefs, refs.permissions)
  }
  
  return currentState
})
```

## Real-World Examples

### Example 1: Request Context Management

Managing request-scoped data across async operations in a web server:

```typescript
import { Effect, FiberRef, FiberRefs, Context } from "effect"

// Define request context FiberRefs
export const makeRequestContext = Effect.gen(function* () {
  const requestId = yield* FiberRef.make("")
  const userId = yield* FiberRef.make<string | null>(null)
  const traceId = yield* FiberRef.make("")
  const startTime = yield* FiberRef.make(Date.now())
  
  return { requestId, userId, traceId, startTime } as const
})

// Service for managing request context
export interface RequestContextService {
  readonly setContext: (context: {
    requestId: string
    userId?: string
    traceId: string
  }) => Effect.Effect<void>
  readonly getContext: Effect.Effect<{
    requestId: string
    userId: string | null
    traceId: string
    duration: number
  }>
  readonly getCurrentRefs: Effect.Effect<FiberRefs.FiberRefs>
}

export const RequestContextServiceLive = Effect.gen(function* () {
  const refs = yield* makeRequestContext
  
  const setContext = (context: {
    requestId: string
    userId?: string
    traceId: string
  }) => Effect.gen(function* () {
    yield* FiberRef.set(refs.requestId, context.requestId)
    yield* FiberRef.set(refs.traceId, context.traceId)
    if (context.userId) {
      yield* FiberRef.set(refs.userId, context.userId)
    }
    yield* FiberRef.set(refs.startTime, Date.now())
  })
  
  const getContext = Effect.gen(function* () {
    const fiberRefs = yield* Effect.getFiberRefs
    const requestId = FiberRefs.getOrDefault(fiberRefs, refs.requestId)
    const userId = FiberRefs.getOrDefault(fiberRefs, refs.userId)
    const traceId = FiberRefs.getOrDefault(fiberRefs, refs.traceId)
    const startTime = FiberRefs.getOrDefault(fiberRefs, refs.startTime)
    
    return {
      requestId,
      userId,
      traceId,
      duration: Date.now() - startTime
    }
  })
  
  const getCurrentRefs = Effect.getFiberRefs
  
  return {
    setContext,
    getContext,
    getCurrentRefs
  } as const satisfies RequestContextService
}).pipe(
  Effect.map((impl) => impl)
)

// Usage in HTTP request handler
export const handleRequest = (request: { id: string; userId?: string }) =>
  Effect.gen(function* () {
    const contextService = yield* RequestContextServiceLive
    const crypto = yield* Effect.serviceOption(Effect.context<crypto.WebCrypto>())
    
    // Set request context
    yield* contextService.setContext({
      requestId: request.id,
      userId: request.userId,
      traceId: crypto ? crypto.randomUUID() : Math.random().toString()
    })
    
    // Process request - context automatically available in all child operations
    const result = yield* processBusinessLogic()
    
    // Get final context for logging
    const context = yield* contextService.getContext
    console.log("Request completed:", context)
    
    return result
  })

const processBusinessLogic = Effect.gen(function* () {
  // This operation automatically has access to request context
  const contextService = yield* RequestContextServiceLive
  const context = yield* contextService.getContext
  
  console.log(`Processing request ${context.requestId} for user ${context.userId}`)
  
  // Fork child operations - they inherit context
  const results = yield* Effect.all([
    processStep1(),
    processStep2(),
    processStep3()
  ], { concurrency: "unbounded" })
  
  return results
})

const processStep1 = Effect.gen(function* () {
  const contextService = yield* RequestContextServiceLive
  const context = yield* contextService.getContext
  
  console.log(`Step 1 executing for request ${context.requestId}`)
  yield* Effect.sleep("100 millis")
  return "step1-result"
})

const processStep2 = Effect.gen(function* () {
  const contextService = yield* RequestContextServiceLive
  const context = yield* contextService.getContext
  
  console.log(`Step 2 executing for request ${context.requestId}`)
  yield* Effect.sleep("200 millis")
  return "step2-result"
})

const processStep3 = Effect.gen(function* () {
  const contextService = yield* RequestContextServiceLive  
  const context = yield* contextService.getContext
  
  console.log(`Step 3 executing for request ${context.requestId}`)
  yield* Effect.sleep("150 millis")
  return "step3-result"
})
```

### Example 2: Multi-Tenant Application State

Managing tenant-specific configuration and permissions across operations:

```typescript
import { Effect, FiberRef, FiberRefs, Array as Arr } from "effect"

// Tenant-specific state management
export interface TenantConfig {
  readonly tenantId: string
  readonly features: readonly string[]
  readonly limits: {
    readonly apiCallsPerMinute: number
    readonly storageQuotaBytes: number
  }
  readonly customSettings: Record<string, unknown>
}

export const makeTenantState = Effect.gen(function* () {
  const currentTenant = yield* FiberRef.make<TenantConfig | null>(null)
  const apiCallCount = yield* FiberRef.make(0)
  const operationStack = yield* FiberRef.make<readonly string[]>([])
  
  return { currentTenant, apiCallCount, operationStack } as const
})

export interface TenantService {
  readonly setTenant: (config: TenantConfig) => Effect.Effect<void>
  readonly getCurrentTenant: Effect.Effect<TenantConfig | null>
  readonly incrementApiCalls: Effect.Effect<number>
  readonly pushOperation: (operation: string) => Effect.Effect<void>
  readonly popOperation: Effect.Effect<string | null>
  readonly getTenantRefs: Effect.Effect<FiberRefs.FiberRefs>
  readonly applyTenantRefs: (refs: FiberRefs.FiberRefs) => Effect.Effect<void>
}

export const TenantServiceLive = Effect.gen(function* () {
  const refs = yield* makeTenantState
  
  const setTenant = (config: TenantConfig) =>
    FiberRef.set(refs.currentTenant, config)
  
  const getCurrentTenant = FiberRef.get(refs.currentTenant)
  
  const incrementApiCalls = Effect.gen(function* () {
    const current = yield* FiberRef.get(refs.apiCallCount)
    const newCount = current + 1
    yield* FiberRef.set(refs.apiCallCount, newCount)
    return newCount
  })
  
  const pushOperation = (operation: string) => Effect.gen(function* () {
    const current = yield* FiberRef.get(refs.operationStack)
    yield* FiberRef.set(refs.operationStack, [operation, ...current])
  })
  
  const popOperation = Effect.gen(function* () {
    const current = yield* FiberRef.get(refs.operationStack)
    if (Arr.isNonEmptyReadonlyArray(current)) {
      const [head, ...tail] = current
      yield* FiberRef.set(refs.operationStack, tail)
      return head
    }
    return null
  })
  
  const getTenantRefs = Effect.getFiberRefs
  
  const applyTenantRefs = (fiberRefs: FiberRefs.FiberRefs) =>
    Effect.setFiberRefs(fiberRefs)
  
  return {
    setTenant,
    getCurrentTenant,
    incrementApiCalls,
    pushOperation,
    popOperation,
    getTenantRefs,
    applyTenantRefs
  } as const satisfies TenantService
}).pipe(Effect.map(impl => impl))

// Usage in multi-tenant API
export const processApiRequest = (
  tenantId: string,
  operation: string,
  handler: Effect.Effect<unknown>
) => Effect.gen(function* () {
  const tenantService = yield* TenantServiceLive
  
  // Load tenant configuration
  const tenantConfig = yield* loadTenantConfig(tenantId)
  yield* tenantService.setTenant(tenantConfig)
  
  // Track operation
  yield* tenantService.pushOperation(operation)
  yield* tenantService.incrementApiCalls()
  
  // Execute with tenant context
  try {
    // Capture current tenant state
    const tenantRefs = yield* tenantService.getTenantRefs
    
    // Fork operation with tenant context
    const result = yield* Effect.fork(
      Effect.gen(function* () {
        // Child fiber inherits tenant state
        const childTenantService = yield* TenantServiceLive
        const tenant = yield* childTenantService.getCurrentTenant
        
        if (!tenant) {
          return yield* Effect.fail(new Error("No tenant context"))
        }
        
        // Check permissions
        yield* checkTenantPermissions(tenant, operation)
        
        // Execute handler
        return yield* handler
      })
    ).pipe(Fiber.join)
    
    return result
  } finally {
    yield* tenantService.popOperation()
  }
})

const loadTenantConfig = (tenantId: string): Effect.Effect<TenantConfig> =>
  Effect.succeed({
    tenantId,
    features: ["feature-a", "feature-b"],
    limits: {
      apiCallsPerMinute: 1000,
      storageQuotaBytes: 1024 * 1024 * 100 // 100MB
    },
    customSettings: {
      theme: "dark",
      language: "en"
    }
  })

const checkTenantPermissions = (
  tenant: TenantConfig,
  operation: string
): Effect.Effect<void> =>
  Effect.gen(function* () {
    if (operation === "admin" && !tenant.features.includes("admin-access")) {
      return yield* Effect.fail(
        new Error(`Tenant ${tenant.tenantId} lacks admin access`)
      )
    }
    // Additional permission checks...
  })
```

### Example 3: Transaction Context with Rollback

Managing transactional state that can be rolled back across nested operations:

```typescript
import { Effect, FiberRef, FiberRefs, Ref } from "effect"

export interface TransactionState {
  readonly transactionId: string
  readonly changes: readonly string[]
  readonly rollbackActions: readonly (() => Effect.Effect<void>)[]
  readonly committed: boolean
}

export const makeTransactionRefs = Effect.gen(function* () {
  const transactionState = yield* FiberRef.make<TransactionState | null>(null)
  const changeCounter = yield* FiberRef.make(0)
  const isolationLevel = yield* FiberRef.make<"read-committed" | "serializable">("read-committed")
  
  return { transactionState, changeCounter, isolationLevel } as const
})

export interface TransactionService {
  readonly beginTransaction: (id: string) => Effect.Effect<void>
  readonly addChange: (description: string, rollback: () => Effect.Effect<void>) => Effect.Effect<void>
  readonly commitTransaction: Effect.Effect<void>
  readonly rollbackTransaction: Effect.Effect<void>
  readonly getCurrentTransaction: Effect.Effect<TransactionState | null>
  readonly forkWithTransaction: <A, E, R>(
    effect: Effect.Effect<A, E, R>
  ) => Effect.Effect<A, E, R>
}

export const TransactionServiceLive = Effect.gen(function* () {
  const refs = yield* makeTransactionRefs
  
  const beginTransaction = (id: string) => Effect.gen(function* () {
    const newTransaction: TransactionState = {
      transactionId: id,
      changes: [],
      rollbackActions: [],
      committed: false
    }
    yield* FiberRef.set(refs.transactionState, newTransaction)
    yield* FiberRef.set(refs.changeCounter, 0)
  })
  
  const addChange = (description: string, rollback: () => Effect.Effect<void>) =>
    Effect.gen(function* () {
      const current = yield* FiberRef.get(refs.transactionState)
      const counter = yield* FiberRef.get(refs.changeCounter)
      
      if (!current) {
        return yield* Effect.fail(new Error("No active transaction"))
      }
      
      const updated: TransactionState = {
        ...current,
        changes: [...current.changes, description],
        rollbackActions: [...current.rollbackActions, rollback]
      }
      
      yield* FiberRef.set(refs.transactionState, updated)
      yield* FiberRef.set(refs.changeCounter, counter + 1)
    })
  
  const commitTransaction = Effect.gen(function* () {
    const transaction = yield* FiberRef.get(refs.transactionState)
    
    if (!transaction) {
      return yield* Effect.fail(new Error("No active transaction"))
    }
    
    const committed: TransactionState = {
      ...transaction,
      committed: true
    }
    
    yield* FiberRef.set(refs.transactionState, committed)
    console.log(`Transaction ${transaction.transactionId} committed with ${transaction.changes.length} changes`)
  })
  
  const rollbackTransaction = Effect.gen(function* () {
    const transaction = yield* FiberRef.get(refs.transactionState)
    
    if (!transaction) {
      return yield* Effect.fail(new Error("No active transaction"))
    }
    
    console.log(`Rolling back transaction ${transaction.transactionId}...`)
    
    // Execute rollback actions in reverse order
    const reversedActions = transaction.rollbackActions.slice().reverse()
    for (const action of reversedActions) {
      yield* action()
    }
    
    yield* FiberRef.set(refs.transactionState, null)
    yield* FiberRef.set(refs.changeCounter, 0)
    
    console.log(`Transaction ${transaction.transactionId} rolled back`)
  })
  
  const getCurrentTransaction = FiberRef.get(refs.transactionState)
  
  const forkWithTransaction = <A, E, R>(
    effect: Effect.Effect<A, E, R>
  ) => Effect.gen(function* () {
    // Capture current transaction state
    const currentRefs = yield* Effect.getFiberRefs
    
    return yield* Effect.fork(
      Effect.gen(function* () {
        // Child fiber inherits transaction state
        yield* Effect.setFiberRefs(currentRefs)
        return yield* effect
      })
    ).pipe(Fiber.join)
  })
  
  return {
    beginTransaction,
    addChange,
    commitTransaction,
    rollbackTransaction,
    getCurrentTransaction,
    forkWithTransaction
  } as const satisfies TransactionService
}).pipe(Effect.map(impl => impl))

// Usage example
export const performTransactionalWork = Effect.gen(function* () {
  const txService = yield* TransactionServiceLive
  
  // Begin transaction
  yield* txService.beginTransaction("tx-001")
  
  try {
    // Perform some work with rollback capabilities
    yield* performStep1(txService)
    yield* performStep2(txService)
    
    // Fork concurrent operations that inherit transaction
    const results = yield* Effect.all([
      txService.forkWithTransaction(performStep3()),
      txService.forkWithTransaction(performStep4())
    ])
    
    // Commit if all successful
    yield* txService.commitTransaction()
    return results
    
  } catch (error) {
    // Rollback on any error
    yield* txService.rollbackTransaction()
    throw error
  }
})

const performStep1 = (txService: TransactionService) =>
  Effect.gen(function* () {
    yield* txService.addChange(
      "Created user record",
      () => Effect.log("Deleting user record...")
    )
    console.log("Step 1: User created")
  })

const performStep2 = (txService: TransactionService) =>
  Effect.gen(function* () {
    yield* txService.addChange(
      "Updated permissions",
      () => Effect.log("Reverting permissions...")
    )
    console.log("Step 2: Permissions updated")
  })

const performStep3 = () =>
  Effect.gen(function* () {
    const txService = yield* TransactionServiceLive
    yield* txService.addChange(
      "Sent notification",
      () => Effect.log("Canceling notification...")
    )
    console.log("Step 3: Notification sent")
    return "step3-result"
  })

const performStep4 = () =>
  Effect.gen(function* () {
    const txService = yield* TransactionServiceLive
    yield* txService.addChange(
      "Updated cache",
      () => Effect.log("Invalidating cache...")
    )
    console.log("Step 4: Cache updated")
    return "step4-result"
  })
```

## Advanced Features Deep Dive

### Feature 1: FiberRefs Forking and Joining

Understanding how FiberRef values are inherited and merged across fiber boundaries:

#### Basic Forking Usage

```typescript
import { Effect, FiberRef, FiberRefs, Fiber, FiberId } from "effect"

export const forkingExample = Effect.gen(function* () {
  const counterRef = yield* FiberRef.make(0)
  
  // Set initial value
  yield* FiberRef.set(counterRef, 10)
  
  // Fork child fiber
  const childFiber = yield* Effect.fork(
    Effect.gen(function* () {
      // Child inherits parent's value
      const inherited = yield* FiberRef.get(counterRef)
      console.log("Child inherited:", inherited) // 10
      
      // Child modifies its copy
      yield* FiberRef.set(counterRef, 25)
      
      return yield* FiberRef.get(counterRef)
    })
  )
  
  // Parent value unchanged during child execution
  const parentValue = yield* FiberRef.get(counterRef)
  console.log("Parent during child:", parentValue) // 10
  
  const childResult = yield* Fiber.join(childFiber)
  console.log("Child result:", childResult) // 25
  
  // Parent still has original value (no automatic merging)
  const finalParentValue = yield* FiberRef.get(counterRef)
  console.log("Parent after join:", finalParentValue) // 10
})
```

#### Advanced Forking: Manual State Management

```typescript
import { Effect, FiberRef, FiberRefs, Fiber, FiberId } from "effect"

export const manualFiberRefsManagement = Effect.gen(function* () {
  const stateRef = yield* FiberRef.make({ count: 0, data: "initial" })
  
  // Parent sets initial state
  yield* FiberRef.set(stateRef, { count: 5, data: "parent-data" })
  
  // Fork with explicit FiberRefs management
  const childFiber = yield* Effect.fork(
    Effect.gen(function* () {
      // Get current fiber refs to pass to other operations
      const currentRefs = yield* Effect.getFiberRefs
      
      // Child can inspect its inherited state
      const childState = FiberRefs.getOrDefault(currentRefs, stateRef)
      console.log("Child inherited state:", childState)
      
      // Modify child state
      yield* FiberRef.set(stateRef, { count: childState.count + 10, data: "child-data" })
      
      // Return the modified FiberRefs for parent to potentially use
      return yield* Effect.getFiberRefs
    })
  )
  
  const childRefs = yield* Fiber.join(childFiber)
  
  // Parent can choose to inherit child's state
  const childState = FiberRefs.getOrDefault(childRefs, stateRef)
  console.log("Child final state:", childState) // { count: 15, data: "child-data" }
  
  // Parent can apply child's FiberRefs if desired
  yield* Effect.setFiberRefs(childRefs)
  
  const finalState = yield* FiberRef.get(stateRef)
  console.log("Parent after inheriting:", finalState) // { count: 15, data: "child-data" }
})
```

### Feature 2: Collection Operations and Querying

Efficiently working with large collections of FiberRefs:

#### Inspecting and Filtering FiberRefs

```typescript
import { Effect, FiberRef, FiberRefs, HashSet } from "effect"

export const fiberRefsInspection = Effect.gen(function* () {
  // Create several FiberRefs with different purposes
  const userRef = yield* FiberRef.make("anonymous")
  const sessionRef = yield* FiberRef.make("")
  const debugModeRef = yield* FiberRef.make(false)
  const counterRef = yield* FiberRef.make(0)
  
  // Set some values
  yield* FiberRef.set(userRef, "alice")
  yield* FiberRef.set(sessionRef, "sess-123")
  yield* FiberRef.set(debugModeRef, true)
  yield* FiberRef.set(counterRef, 42)
  
  // Get all FiberRefs
  const allRefs = yield* Effect.getFiberRefs
  
  // Get set of all FiberRef keys
  const fiberRefSet = FiberRefs.fiberRefs(allRefs)
  console.log("Total FiberRefs:", HashSet.size(fiberRefSet))
  
  // Query specific values
  const userValue = FiberRefs.get(allRefs, userRef)
  console.log("User value:", userValue) // Some("alice")
  
  const nonExistentRef = yield* FiberRef.make("not-set")
  const missingValue = FiberRefs.get(allRefs, nonExistentRef)
  console.log("Missing value:", missingValue) // None
  
  // Get with default
  const userValueOrDefault = FiberRefs.getOrDefault(allRefs, userRef)
  const defaultValue = FiberRefs.getOrDefault(allRefs, nonExistentRef)
  console.log("User or default:", userValueOrDefault) // "alice"
  console.log("Default for non-existent:", defaultValue) // "not-set" (initial value)
  
  return {
    totalRefs: HashSet.size(fiberRefSet),
    userValue: userValueOrDefault,
    debugMode: FiberRefs.getOrDefault(allRefs, debugModeRef),
    counter: FiberRefs.getOrDefault(allRefs, counterRef)
  }
})
```

#### Bulk Updates and Transformations

```typescript
import { Effect, FiberRef, FiberRefs, FiberId } from "effect"

export const bulkFiberRefsOperations = Effect.gen(function* () {
  const refs = yield* Effect.all({
    metric1: FiberRef.make(0),
    metric2: FiberRef.make(0),
    metric3: FiberRef.make(0),
    lastUpdate: FiberRef.make(Date.now())
  })
  
  // Get current FiberRefs
  const currentRefs = yield* Effect.getFiberRefs
  
  // Create a new FiberRefs with bulk updates
  const fiberId = FiberId.make(1, Date.now())
  const updatedRefs = currentRefs.pipe(
    FiberRefs.updateAs({
      fiberId,
      fiberRef: refs.metric1,
      value: 100
    }),
    FiberRefs.updateAs({
      fiberId,
      fiberRef: refs.metric2,
      value: 200
    }),
    FiberRefs.updateAs({
      fiberId,
      fiberRef: refs.metric3,
      value: 300
    }),
    FiberRefs.updateAs({
      fiberId,
      fiberRef: refs.lastUpdate,
      value: Date.now()
    })
  )
  
  // Apply the bulk updates
  yield* Effect.setFiberRefs(updatedRefs)
  
  // Verify updates
  const results = {
    metric1: yield* FiberRef.get(refs.metric1),
    metric2: yield* FiberRef.get(refs.metric2),
    metric3: yield* FiberRef.get(refs.metric3),
    lastUpdate: yield* FiberRef.get(refs.lastUpdate)
  }
  
  console.log("Bulk update results:", results)
  return results
})
```

### Feature 3: Advanced Joining Strategies

Understanding how FiberRef values merge when child fibers complete:

#### Custom Join Behavior with FiberRef Configuration

```typescript
import { Effect, FiberRef, FiberRefs, Fiber } from "effect"

// Create FiberRef with custom join behavior for accumulating values
export const createAccumulatingRef = <A>(initial: A, combine: (parent: A, child: A) => A) =>
  FiberRef.make(initial, {
    fork: (parent) => parent, // Child starts with parent's value
    join: combine            // Custom merge strategy
  })

export const customJoiningExample = Effect.gen(function* () {
  // Counter that accumulates child values
  const accumulatingCounter = yield* createAccumulatingRef(
    0,
    (parent, child) => parent + child
  )
  
  // List that merges child additions
  const accumulatingList = yield* createAccumulatingRef(
    [] as string[],
    (parent, child) => [...parent, ...child]
  )
  
  // Set initial parent values
  yield* FiberRef.set(accumulatingCounter, 10)
  yield* FiberRef.set(accumulatingList, ["parent-item"])
  
  // Fork multiple children that modify state
  const childFibers = yield* Effect.all([
    Effect.fork(
      Effect.gen(function* () {
        yield* FiberRef.update(accumulatingCounter, (n) => n + 5)
        yield* FiberRef.update(accumulatingList, (list) => [...list, "child1-item"])
        return "child1-done"
      })
    ),
    Effect.fork(
      Effect.gen(function* () {
        yield* FiberRef.update(accumulatingCounter, (n) => n + 3)
        yield* FiberRef.update(accumulatingList, (list) => [...list, "child2-item"])
        return "child2-done"
      })
    ),
    Effect.fork(
      Effect.gen(function* () {
        yield* FiberRef.update(accumulatingCounter, (n) => n + 7)
        yield* FiberRef.update(accumulatingList, (list) => [...list, "child3-item"])
        return "child3-done"
      })
    )
  ])
  
  // Wait for all children and join their FiberRefs
  const childResults = yield* Effect.all(
    childFibers.map(Fiber.join)
  )
  
  // Check final values after automatic joining
  const finalCounter = yield* FiberRef.get(accumulatingCounter)
  const finalList = yield* FiberRef.get(accumulatingList)
  
  console.log("Final counter:", finalCounter) // Should be 10 + 5 + 3 + 7 = 25
  console.log("Final list:", finalList) // Should include all items
  
  return {
    childResults,
    finalCounter,
    finalList
  }
})
```

## Practical Patterns & Best Practices

### Pattern 1: FiberRefs Service Factory

Creating reusable services that encapsulate FiberRefs management:

```typescript
import { Effect, FiberRef, FiberRefs, Context } from "effect"

// Generic FiberRefs service factory
export const makeFiberRefsService = <T extends Record<string, unknown>>(
  initialState: T
) => {
  const make = Effect.gen(function* () {
    // Create FiberRefs for each property
    const refs = {} as { [K in keyof T]: FiberRef.FiberRef<T[K]> }
    
    for (const [key, value] of Object.entries(initialState)) {
      refs[key as keyof T] = yield* FiberRef.make(value)
    }
    
    const getState = Effect.gen(function* () {
      const fiberRefs = yield* Effect.getFiberRefs
      const state = {} as T
      
      for (const [key, ref] of Object.entries(refs)) {
        state[key as keyof T] = FiberRefs.getOrDefault(fiberRefs, ref as FiberRef.FiberRef<any>)
      }
      
      return state
    })
    
    const setState = (newState: Partial<T>) => Effect.gen(function* () {
      for (const [key, value] of Object.entries(newState)) {
        if (key in refs && value !== undefined) {
          yield* FiberRef.set(refs[key as keyof T], value)
        }
      }
    })
    
    const updateState = (updater: (current: T) => Partial<T>) => Effect.gen(function* () {
      const current = yield* getState
      const updates = updater(current)
      yield* setState(updates)
    })
    
    const captureRefs = Effect.getFiberRefs
    
    const restoreRefs = (fiberRefs: FiberRefs.FiberRefs) =>
      Effect.setFiberRefs(fiberRefs)
    
    return {
      refs,
      getState,
      setState,
      updateState,
      captureRefs,
      restoreRefs
    } as const
  })
  
  return make
}

// Usage example: Application state service
interface AppState {
  userId: string | null
  sessionId: string
  theme: "light" | "dark"
  language: string
  lastActivity: number
}

export const AppStateService = makeFiberRefsService<AppState>({
  userId: null,
  sessionId: "",
  theme: "light",
  language: "en",
  lastActivity: Date.now()
})

// Usage in application
export const handleUserLogin = (userId: string, sessionId: string) =>
  Effect.gen(function* () {
    const appState = yield* AppStateService
    
    yield* appState.setState({
      userId,
      sessionId,
      lastActivity: Date.now()
    })
    
    // Fork background tasks that inherit state
    yield* Effect.fork(
      updateUserPreferences()
    )
    
    yield* Effect.fork(
      logUserActivity()
    )
    
    return yield* appState.getState()
  })

const updateUserPreferences = Effect.gen(function* () {
  const appState = yield* AppStateService
  const state = yield* appState.getState()
  
  if (state.userId) {
    // Load user preferences and update theme/language
    yield* appState.updateState((current) => ({
      theme: "dark", // User's preferred theme
      language: "es" // User's preferred language
    }))
  }
})

const logUserActivity = Effect.gen(function* () {
  const appState = yield* AppStateService
  const state = yield* appState.getState()
  
  console.log(`User ${state.userId} active in session ${state.sessionId}`)
  
  // Update activity timestamp periodically
  yield* appState.updateState(() => ({
    lastActivity: Date.now()
  }))
})
```

### Pattern 2: Scoped FiberRefs Management

Managing FiberRefs within specific scopes with automatic cleanup:

```typescript
import { Effect, FiberRef, FiberRefs, Scope } from "effect"

// Helper for scoped FiberRefs operations
export const withScopedFiberRefs = <A, E, R>(
  setup: Effect.Effect<FiberRefs.FiberRefs, E, R>,
  operation: Effect.Effect<A, E, R>
): Effect.Effect<A, E, R> =>
  Effect.gen(function* () {
    // Capture current state
    const originalRefs = yield* Effect.getFiberRefs
    
    try {
      // Setup scoped state
      const scopedRefs = yield* setup
      yield* Effect.setFiberRefs(scopedRefs)
      
      // Execute operation with scoped state
      return yield* operation
    } finally {
      // Restore original state
      yield* Effect.setFiberRefs(originalRefs)
    }
  })

// Example: Database transaction scope
export const withTransactionScope = <A, E, R>(
  transactionId: string,
  operation: Effect.Effect<A, E, R>
): Effect.Effect<A, E, R> => {
  const setupTransactionRefs = Effect.gen(function* () {
    const txIdRef = yield* FiberRef.make("")
    const txActiveRef = yield* FiberRef.make(false)
    const txStartTimeRef = yield* FiberRef.make(0)
    
    yield* FiberRef.set(txIdRef, transactionId)
    yield* FiberRef.set(txActiveRef, true)
    yield* FiberRef.set(txStartTimeRef, Date.now())
    
    return yield* Effect.getFiberRefs
  })
  
  return withScopedFiberRefs(setupTransactionRefs, operation)
}

// Usage
export const performDatabaseWork = withTransactionScope(
  "tx-001",
  Effect.gen(function* () {
    // This code runs with transaction FiberRefs active
    yield* databaseOperation1()
    yield* databaseOperation2()
    
    // Transaction context is automatically cleaned up when scope exits
    return "work-completed"
  })
)

const databaseOperation1 = Effect.gen(function* () {
  // Can access transaction context via standard FiberRef operations
  const currentRefs = yield* Effect.getFiberRefs
  console.log("Operation 1 executing in transaction context")
})

const databaseOperation2 = Effect.gen(function* () {
  const currentRefs = yield* Effect.getFiberRefs
  console.log("Operation 2 executing in transaction context")
})
```

### Pattern 3: FiberRefs State Diffing and Synchronization

Advanced pattern for tracking and synchronizing FiberRefs changes:

```typescript
import { Effect, FiberRef, FiberRefs, Array as Arr, HashMap } from "effect"

// State diff utilities for FiberRefs
export const FiberRefsDiff = {
  // Create a snapshot of current FiberRefs state
  snapshot: Effect.gen(function* () {
    const fiberRefs = yield* Effect.getFiberRefs
    const fiberRefSet = FiberRefs.fiberRefs(fiberRefs)
    
    const snapshot = new Map<FiberRef.FiberRef<any>, any>()
    
    for (const ref of fiberRefSet) {
      const value = FiberRefs.getOrDefault(fiberRefs, ref)
      snapshot.set(ref, value)
    }
    
    return snapshot
  }),
  
  // Compare two snapshots and return differences
  compare: (
    before: Map<FiberRef.FiberRef<any>, any>,
    after: Map<FiberRef.FiberRef<any>, any>
  ) => {
    const changes: Array<{
      ref: FiberRef.FiberRef<any>
      type: "added" | "modified" | "deleted"
      beforeValue?: any
      afterValue?: any
    }> = []
    
    // Check for additions and modifications
    for (const [ref, afterValue] of after) {
      if (!before.has(ref)) {
        changes.push({ ref, type: "added", afterValue })
      } else {
        const beforeValue = before.get(ref)
        if (beforeValue !== afterValue) {
          changes.push({ ref, type: "modified", beforeValue, afterValue })
        }
      }
    }
    
    // Check for deletions
    for (const [ref, beforeValue] of before) {
      if (!after.has(ref)) {
        changes.push({ ref, type: "deleted", beforeValue })
      }
    }
    
    return changes
  },
  
  // Apply changes to create a new FiberRefs
  applyChanges: (
    baseFiberRefs: FiberRefs.FiberRefs,
    changes: Array<{
      ref: FiberRef.FiberRef<any>
      type: "added" | "modified" | "deleted"
      afterValue?: any
    }>
  ) => Effect.gen(function* () {
    let currentRefs = baseFiberRefs
    
    for (const change of changes) {
      if (change.type === "added" || change.type === "modified") {
        currentRefs = FiberRefs.updateAs(currentRefs, {
          fiberId: yield* Effect.fiberId,
          fiberRef: change.ref,
          value: change.afterValue
        })
      } else if (change.type === "deleted") {
        currentRefs = FiberRefs.delete(currentRefs, change.ref)
      }
    }
    
    return currentRefs
  })
}

// Usage: State synchronization service
export interface StateSyncService {
  readonly captureState: Effect.Effect<Map<FiberRef.FiberRef<any>, any>>
  readonly syncState: (targetState: Map<FiberRef.FiberRef<any>, any>) => Effect.Effect<void>
  readonly watchChanges: <A, E, R>(
    operation: Effect.Effect<A, E, R>
  ) => Effect.Effect<{
    result: A
    changes: Array<{
      ref: FiberRef.FiberRef<any>
      type: "added" | "modified" | "deleted"
      beforeValue?: any
      afterValue?: any
    }>
  }>
}

export const StateSyncServiceLive: Effect.Effect<StateSyncService> = Effect.gen(function* () {
  const captureState = FiberRefsDiff.snapshot
  
  const syncState = (targetState: Map<FiberRef.FiberRef<any>, any>) =>
    Effect.gen(function* () {
      const currentState = yield* captureState
      const changes = FiberRefsDiff.compare(currentState, targetState)
      
      const currentRefs = yield* Effect.getFiberRefs
      const updatedRefs = yield* FiberRefsDiff.applyChanges(
        currentRefs,
        changes.map(change => ({
          ref: change.ref,
          type: change.type,
          afterValue: change.afterValue
        }))
      )
      
      yield* Effect.setFiberRefs(updatedRefs)
    })
  
  const watchChanges = <A, E, R>(
    operation: Effect.Effect<A, E, R>
  ) => Effect.gen(function* () {
    const beforeState = yield* captureState
    const result = yield* operation
    const afterState = yield* captureState
    
    const changes = FiberRefsDiff.compare(beforeState, afterState)
    
    return { result, changes }
  })
  
  return { captureState, syncState, watchChanges } as const satisfies StateSyncService
})

// Usage example
export const demonstrateStateSync = Effect.gen(function* () {
  const syncService = yield* StateSyncServiceLive
  
  // Create some FiberRefs
  const counterRef = yield* FiberRef.make(0)
  const messageRef = yield* FiberRef.make("")
  
  // Capture initial state
  const initialState = yield* syncService.captureState
  
  // Perform operations and watch changes
  const { result, changes } = yield* syncService.watchChanges(
    Effect.gen(function* () {
      yield* FiberRef.set(counterRef, 42)
      yield* FiberRef.set(messageRef, "hello world")
      return "operations-complete"
    })
  )
  
  console.log("Operation result:", result)
  console.log("Detected changes:", changes.length)
  
  // Reset to initial state
  yield* syncService.syncState(initialState)
  
  // Verify reset
  const finalCounter = yield* FiberRef.get(counterRef)
  const finalMessage = yield* FiberRef.get(messageRef)
  
  console.log("After reset - counter:", finalCounter, "message:", finalMessage)
})
```

## Integration Examples

### Integration with HTTP Server and Client

Using FiberRefs to manage request/response context across HTTP operations:

```typescript
import { Effect, FiberRef, FiberRefs, Context } from "effect"

// HTTP context management with FiberRefs
export interface HttpContext {
  readonly requestId: string
  readonly method: string
  readonly path: string
  readonly headers: Record<string, string>
  readonly startTime: number
  readonly userAgent?: string
  readonly clientIp?: string
}

export const makeHttpContextRefs = Effect.gen(function* () {
  const requestId = yield* FiberRef.make("")
  const method = yield* FiberRef.make("")
  const path = yield* FiberRef.make("")
  const headers = yield* FiberRef.make<Record<string, string>>({})
  const startTime = yield* FiberRef.make(0)
  const userAgent = yield* FiberRef.make<string | undefined>(undefined)
  const clientIp = yield* FiberRef.make<string | undefined>(undefined)
  
  return { requestId, method, path, headers, startTime, userAgent, clientIp } as const
})

export interface HttpContextService {
  readonly setRequestContext: (context: HttpContext) => Effect.Effect<void>
  readonly getRequestContext: Effect.Effect<HttpContext>
  readonly getRequestDuration: Effect.Effect<number>
  readonly withRequestContext: <A, E, R>(
    context: HttpContext,
    operation: Effect.Effect<A, E, R>
  ) => Effect.Effect<A, E, R>
}

export const HttpContextServiceLive = Effect.gen(function* () {
  const refs = yield* makeHttpContextRefs
  
  const setRequestContext = (context: HttpContext) => Effect.gen(function* () {
    yield* FiberRef.set(refs.requestId, context.requestId)
    yield* FiberRef.set(refs.method, context.method)
    yield* FiberRef.set(refs.path, context.path)
    yield* FiberRef.set(refs.headers, context.headers)
    yield* FiberRef.set(refs.startTime, context.startTime)
    yield* FiberRef.set(refs.userAgent, context.userAgent)
    yield* FiberRef.set(refs.clientIp, context.clientIp)
  })
  
  const getRequestContext = Effect.gen(function* () {
    const fiberRefs = yield* Effect.getFiberRefs
    
    return {
      requestId: FiberRefs.getOrDefault(fiberRefs, refs.requestId),
      method: FiberRefs.getOrDefault(fiberRefs, refs.method),
      path: FiberRefs.getOrDefault(fiberRefs, refs.path),
      headers: FiberRefs.getOrDefault(fiberRefs, refs.headers),
      startTime: FiberRefs.getOrDefault(fiberRefs, refs.startTime),
      userAgent: FiberRefs.getOrDefault(fiberRefs, refs.userAgent),
      clientIp: FiberRefs.getOrDefault(fiberRefs, refs.clientIp)
    } as HttpContext
  })
  
  const getRequestDuration = Effect.gen(function* () {
    const context = yield* getRequestContext
    return Date.now() - context.startTime
  })
  
  const withRequestContext = <A, E, R>(
    context: HttpContext,
    operation: Effect.Effect<A, E, R>
  ): Effect.Effect<A, E, R> => Effect.gen(function* () {
    // Capture current FiberRefs
    const originalRefs = yield* Effect.getFiberRefs
    
    try {
      // Set request context
      yield* setRequestContext(context)
      
      // Execute operation with context
      return yield* operation
    } finally {
      // Restore original context
      yield* Effect.setFiberRefs(originalRefs)
    }
  })
  
  return {
    setRequestContext,
    getRequestContext,
    getRequestDuration,
    withRequestContext
  } as const satisfies HttpContextService
}).pipe(Effect.map(impl => impl))

// HTTP middleware that sets up request context
export const withHttpContextMiddleware = <A, E, R>(
  request: {
    id: string
    method: string
    path: string
    headers: Record<string, string>
  },
  handler: Effect.Effect<A, E, R>
): Effect.Effect<A, E, R> => Effect.gen(function* () {
  const httpContext = yield* HttpContextServiceLive
  
  const context: HttpContext = {
    requestId: request.id,
    method: request.method,
    path: request.path,
    headers: request.headers,
    startTime: Date.now(),
    userAgent: request.headers["user-agent"],
    clientIp: request.headers["x-forwarded-for"] || request.headers["x-real-ip"]
  }
  
  return yield* httpContext.withRequestContext(context, handler)
})

// Usage in HTTP handlers
export const handleApiRequest = (request: {
  id: string
  method: string
  path: string
  headers: Record<string, string>
  body?: any
}) => withHttpContextMiddleware(
  request,
  Effect.gen(function* () {
    const httpContext = yield* HttpContextServiceLive
    
    // All operations automatically have access to request context
    yield* logRequestStart()
    
    // Process the request
    const result = yield* processApiRequest(request.body)
    
    // Log completion with timing
    yield* logRequestComplete()
    
    return result
  })
)

const logRequestStart = Effect.gen(function* () {
  const httpContext = yield* HttpContextServiceLive
  const context = yield* httpContext.getRequestContext
  
  console.log(`[${context.requestId}] ${context.method} ${context.path} - Start`)
})

const logRequestComplete = Effect.gen(function* () {
  const httpContext = yield* HttpContextServiceLive
  const context = yield* httpContext.getRequestContext
  const duration = yield* httpContext.getRequestDuration
  
  console.log(`[${context.requestId}] ${context.method} ${context.path} - Complete (${duration}ms)`)
})

const processApiRequest = (body: any) => Effect.gen(function* () {
  const httpContext = yield* HttpContextServiceLive
  const context = yield* httpContext.getRequestContext
  
  // Fork multiple operations that inherit request context
  const results = yield* Effect.all([
    validateRequest(body),
    checkPermissions(),
    processBusinessLogic(body)
  ], { concurrency: 2 })
  
  return {
    requestId: context.requestId,
    results,
    timestamp: Date.now()
  }
})

const validateRequest = (body: any) => Effect.gen(function* () {
  const httpContext = yield* HttpContextServiceLive
  const context = yield* httpContext.getRequestContext
  
  console.log(`[${context.requestId}] Validating request`)
  yield* Effect.sleep("50 millis")
  return { valid: true }
})

const checkPermissions = () => Effect.gen(function* () {
  const httpContext = yield* HttpContextServiceLive
  const context = yield* httpContext.getRequestContext
  
  console.log(`[${context.requestId}] Checking permissions`)
  yield* Effect.sleep("30 millis")
  return { authorized: true }
})

const processBusinessLogic = (body: any) => Effect.gen(function* () {
  const httpContext = yield* HttpContextServiceLive
  const context = yield* httpContext.getRequestContext
  
  console.log(`[${context.requestId}] Processing business logic`)
  yield* Effect.sleep("100 millis")
  return { processed: true, data: body }
})
```

### Testing Strategies

Comprehensive testing approaches for FiberRefs-dependent code:

```typescript
import { Effect, FiberRef, FiberRefs, TestContext, TestServices } from "effect"

// Test utilities for FiberRefs
export const FiberRefsTestUtils = {
  // Create isolated test environment with fresh FiberRefs
  withCleanFiberRefs: <A, E, R>(
    test: Effect.Effect<A, E, R>
  ): Effect.Effect<A, E, R> =>
    Effect.gen(function* () {
      // Start with empty FiberRefs
      const cleanRefs = FiberRefs.empty()
      yield* Effect.setFiberRefs(cleanRefs)
      return yield* test
    }),
  
  // Setup test FiberRefs with specific values
  withTestFiberRefs: <T extends Record<string, any>>(
    setup: T,
    refs: { [K in keyof T]: FiberRef.FiberRef<T[K]> }
  ) => <A, E, R>(
    test: Effect.Effect<A, E, R>
  ): Effect.Effect<A, E, R> =>
    Effect.gen(function* () {
      // Set all test values
      for (const [key, value] of Object.entries(setup)) {
        if (key in refs) {
          yield* FiberRef.set(refs[key as keyof T], value)
        }
      }
      
      return yield* test
    }),
  
  // Assert FiberRef state matches expected values
  assertFiberRefsState: <T extends Record<string, any>>(
    expected: Partial<T>,
    refs: { [K in keyof T]: FiberRef.FiberRef<T[K]> }
  ) => Effect.gen(function* () {
    const fiberRefs = yield* Effect.getFiberRefs
    
    for (const [key, expectedValue] of Object.entries(expected)) {
      if (key in refs) {
        const actualValue = FiberRefs.getOrDefault(fiberRefs, refs[key as keyof T])
        if (actualValue !== expectedValue) {
          return yield* Effect.fail(
            new Error(`FiberRef ${key}: expected ${expectedValue}, got ${actualValue}`)
          )
        }
      }
    }
  }),
  
  // Capture FiberRefs snapshot for later comparison
  captureSnapshot: Effect.gen(function* () {
    const fiberRefs = yield* Effect.getFiberRefs
    const fiberRefSet = FiberRefs.fiberRefs(fiberRefs)
    
    const snapshot: Array<{ ref: FiberRef.FiberRef<any>; value: any }> = []
    
    for (const ref of fiberRefSet) {
      const value = FiberRefs.getOrDefault(fiberRefs, ref)
      snapshot.push({ ref, value })
    }
    
    return snapshot
  })
}

// Example test suite
export const testRequestContextService = Effect.gen(function* () {
  // Test setup with clean FiberRefs
  yield* FiberRefsTestUtils.withCleanFiberRefs(
    Effect.gen(function* () {
      // Create test FiberRefs
      const refs = yield* makeHttpContextRefs
      const service = yield* HttpContextServiceLive
      
      // Test 1: Setting and getting context
      yield* FiberRefsTestUtils.withTestFiberRefs(
        {
          requestId: "test-123",
          method: "GET",
          path: "/api/test",
          headers: { "content-type": "application/json" },
          startTime: Date.now(),
          userAgent: "test-agent",
          clientIp: "127.0.0.1"
        },
        refs
      )(
        Effect.gen(function* () {
          const context = yield* service.getRequestContext
          
          console.assert(context.requestId === "test-123")
          console.assert(context.method === "GET")
          console.assert(context.path === "/api/test")
          
          console.log("✓ Context setting/getting test passed")
        })
      )
      
      // Test 2: Context isolation in child fibers
      yield* testContextIsolation(service, refs)
      
      // Test 3: Context inheritance and merging
      yield* testContextInheritance(service, refs)
      
      console.log("✓ All FiberRefs tests passed")
    })
  )
})

const testContextIsolation = (
  service: HttpContextService,
  refs: Awaited<ReturnType<typeof makeHttpContextRefs>>
) => Effect.gen(function* () {
  // Set parent context
  yield* service.setRequestContext({
    requestId: "parent-123",
    method: "POST",
    path: "/parent",
    headers: {},
    startTime: Date.now()
  })
  
  // Fork child with different context
  const childResult = yield* Effect.fork(
    service.withRequestContext(
      {
        requestId: "child-456",
        method: "GET",
        path: "/child",
        headers: {},
        startTime: Date.now()
      },
      Effect.gen(function* () {
        const childContext = yield* service.getRequestContext
        return childContext.requestId
      })
    )
  ).pipe(Fiber.join)
  
  // Verify parent context unchanged
  const parentContext = yield* service.getRequestContext
  
  console.assert(childResult === "child-456")
  console.assert(parentContext.requestId === "parent-123")
  
  console.log("✓ Context isolation test passed")
})

const testContextInheritance = (
  service: HttpContextService,
  refs: Awaited<ReturnType<typeof makeHttpContextRefs>>
) => Effect.gen(function* () {
  // Set initial context
  yield* service.setRequestContext({
    requestId: "inherit-test",
    method: "PUT",
    path: "/inherit",
    headers: { "x-test": "value" },
    startTime: Date.now()
  })
  
  // Fork child that inherits and modifies context
  const childFiber = yield* Effect.fork(
    Effect.gen(function* () {
      // Child inherits parent context
      const inherited = yield* service.getRequestContext
      console.assert(inherited.requestId === "inherit-test")
      
      // Child can access inherited values
      console.assert(inherited.headers["x-test"] === "value")
      
      // Return child's final FiberRefs
      return yield* Effect.getFiberRefs
    })
  )
  
  const childRefs = yield* Fiber.join(childFiber)
  
  // Verify inheritance worked
  const childRequestId = FiberRefs.getOrDefault(childRefs, refs.requestId)
  console.assert(childRequestId === "inherit-test")
  
  console.log("✓ Context inheritance test passed")
})

// Integration test with real HTTP-like scenario
export const testHttpRequestFlow = Effect.gen(function* () {
  const requests = [
    { id: "req-1", method: "GET", path: "/users", headers: {} },
    { id: "req-2", method: "POST", path: "/users", headers: {} },
    { id: "req-3", method: "PUT", path: "/users/123", headers: {} }
  ]
  
  // Process multiple requests concurrently
  const results = yield* Effect.all(
    requests.map(req => handleApiRequest(req)),
    { concurrency: "unbounded" }
  )
  
  console.log("Processed requests:", results.length)
  
  // Verify each request maintained its own context
  for (let i = 0; i < results.length; i++) {
    const result = results[i]
    const expectedId = requests[i].id
    console.assert(result.requestId === expectedId)
  }
  
  console.log("✓ Concurrent request processing test passed")
})
```

## Conclusion

FiberRefs provides **type-safe fiber-local state management**, **automatic propagation across fiber boundaries**, and **intelligent merging strategies** for complex concurrent applications.

Key benefits:
- **Automatic State Propagation**: Child fibers inherit parent state without manual serialization
- **Smart Merging**: FiberRef values are merged back using configurable join strategies  
- **Type Safety**: Compile-time guarantees about state structure and types
- **Performance**: Efficient copy-on-write semantics avoid unnecessary duplication
- **Composability**: Works seamlessly with Effect's concurrency and error handling

FiberRefs is ideal for managing request context, transaction state, configuration values, and any other fiber-local data that needs to flow through your application's concurrent operations while maintaining isolation and type safety.