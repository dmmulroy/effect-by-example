# ExecutionStrategy: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem ExecutionStrategy Solves

When building applications with Effect, you often need to process multiple operations that could run either sequentially or in parallel. Without proper execution control, you might end up with:

```typescript
// Traditional approach - no control over execution
const processFiles = async (files: string[]) => {
  const results = []
  
  // Always sequential - slow for independent operations
  for (const file of files) {
    results.push(await processFile(file))
  }
  
  // Or always parallel - might overwhelm resources
  return Promise.all(files.map(file => processFile(file)))
}
```

This approach leads to:
- **Resource Management Issues** - Unbounded parallelism can overwhelm system resources
- **Performance Problems** - Sequential execution when parallelism would be beneficial
- **No Flexibility** - Can't adapt execution strategy based on context or load
- **Error Handling Complexity** - Different error propagation patterns for different execution modes

### The ExecutionStrategy Solution

ExecutionStrategy provides a unified way to control how multiple effects are executed, offering three distinct strategies that can be chosen based on your specific needs:

```typescript
import { ExecutionStrategy, Effect } from "effect"

// Sequential execution - one after another
const sequential = ExecutionStrategy.sequential

// Unlimited parallel execution
const parallel = ExecutionStrategy.parallel

// Bounded parallel execution - up to N concurrent operations
const boundedParallel = ExecutionStrategy.parallelN(4)
```

### Key Concepts

**Sequential Strategy**: Executes effects one after another, preserving order and minimizing resource usage.

**Parallel Strategy**: Executes all effects concurrently without limits, maximizing throughput for independent operations.

**ParallelN Strategy**: Executes effects with controlled concurrency, balancing performance and resource management.

## Basic Usage Patterns

### Pattern 1: Strategy Creation

```typescript
import { ExecutionStrategy } from "effect"

// Create execution strategies
const sequential = ExecutionStrategy.sequential
const parallel = ExecutionStrategy.parallel
const limitedParallel = ExecutionStrategy.parallelN(3)
```

### Pattern 2: Strategy Inspection

```typescript
import { ExecutionStrategy } from "effect"

const inspectStrategy = (strategy: ExecutionStrategy.ExecutionStrategy) => {
  if (ExecutionStrategy.isSequential(strategy)) {
    console.log("Sequential execution")
  } else if (ExecutionStrategy.isParallel(strategy)) {
    console.log("Unlimited parallel execution")
  } else if (ExecutionStrategy.isParallelN(strategy)) {
    console.log(`Parallel execution with limit: ${strategy.parallelism}`)
  }
}
```

### Pattern 3: Strategy Matching

```typescript
import { ExecutionStrategy } from "effect"

const getDescription = (strategy: ExecutionStrategy.ExecutionStrategy): string =>
  ExecutionStrategy.match(strategy, {
    onSequential: () => "One at a time",
    onParallel: () => "All at once",
    onParallelN: (n) => `Up to ${n} at once`
  })
```

## Real-World Examples

### Example 1: File Processing Pipeline

Managing file processing with different execution strategies based on file size and system load:

```typescript
import { Effect, ExecutionStrategy } from "effect"
import { FileSystem } from "@effect/platform"

interface FileProcessor {
  readonly processFile: (path: string) => Effect.Effect<ProcessResult, ProcessError>
  readonly getFileSize: (path: string) => Effect.Effect<number, FileError>
}

interface ProcessResult {
  readonly path: string
  readonly size: number
  readonly duration: number
}

class ProcessError {
  readonly _tag = "ProcessError"
  constructor(readonly cause: unknown, readonly path: string) {}
}

class FileError {
  readonly _tag = "FileError" 
  constructor(readonly cause: unknown, readonly path: string) {}
}

// Smart file processing with strategy selection
const createFileProcessor = Effect.gen(function* () {
  const fs = yield* FileSystem.FileSystem

  const processFile = (path: string): Effect.Effect<ProcessResult, ProcessError> =>
    Effect.gen(function* () {
      const start = Date.now()
      const content = yield* fs.readFileString(path)
      const size = content.length
      
      // Simulate processing
      yield* Effect.sleep("100 millis")
      
      return {
        path,
        size,
        duration: Date.now() - start
      }
    }).pipe(
      Effect.catchAll((cause) => Effect.fail(new ProcessError(cause, path)))
    )

  const getFileSize = (path: string): Effect.Effect<number, FileError> =>
    fs.stat(path).pipe(
      Effect.map(stats => stats.size),
      Effect.catchAll((cause) => Effect.fail(new FileError(cause, path)))
    )

  return { processFile, getFileSize } as const satisfies FileProcessor
})

// Strategy selector based on file characteristics
const selectStrategy = (files: readonly string[]) =>
  Effect.gen(function* () {
    const processor = yield* createFileProcessor
    
    // Get total size of all files
    const sizes = yield* Effect.forEach(files, processor.getFileSize, { 
      concurrency: "unbounded" 
    })
    const totalSize = sizes.reduce((sum, size) => sum + size, 0)
    
    // Choose strategy based on total workload
    if (totalSize < 1024 * 1024) { // < 1MB total
      return ExecutionStrategy.parallel
    } else if (totalSize < 10 * 1024 * 1024) { // < 10MB total
      return ExecutionStrategy.parallelN(4)
    } else {
      return ExecutionStrategy.sequential
    }
  })

// Process files with adaptive strategy
const processFiles = (files: readonly string[]) =>
  Effect.gen(function* () {
    const strategy = yield* selectStrategy(files)
    const processor = yield* createFileProcessor
    
    console.log(`Processing ${files.length} files with strategy: ${getDescription(strategy)}`)
    
    return yield* Effect.forEach(files, processor.processFile, {
      concurrency: strategyToConcurrency(strategy)
    })
  })

// Helper to convert ExecutionStrategy to Effect concurrency
const strategyToConcurrency = (strategy: ExecutionStrategy.ExecutionStrategy) =>
  ExecutionStrategy.match(strategy, {
    onSequential: () => 1 as const,
    onParallel: () => "unbounded" as const,
    onParallelN: (n) => n
  })
```

### Example 2: API Rate Limiting

Controlling API requests to respect rate limits while maximizing throughput:

```typescript
import { Effect, ExecutionStrategy, Schedule, Ref } from "effect"

interface ApiClient {
  readonly request: (url: string) => Effect.Effect<ApiResponse, ApiError>
}

interface ApiResponse {
  readonly data: unknown
  readonly headers: Record<string, string>
}

class ApiError {
  readonly _tag = "ApiError"
  constructor(readonly status: number, readonly message: string) {}
}

class RateLimitExceeded {
  readonly _tag = "RateLimitExceeded"
  constructor(readonly retryAfter: number) {}
}

// Rate-limited API client with adaptive execution
const createRateLimitedClient = (baseUrl: string, maxRequestsPerSecond: number) =>
  Effect.gen(function* () {
    const requestCounter = yield* Ref.make(0)
    const lastReset = yield* Ref.make(Date.now())

    const request = (url: string): Effect.Effect<ApiResponse, ApiError> =>
      Effect.gen(function* () {
        // Simulate HTTP request
        yield* Effect.sleep("50 millis")
        
        const random = Math.random()
        if (random < 0.1) {
          return yield* Effect.fail(new ApiError(500, "Server error"))
        }
        
        return {
          data: { url, timestamp: Date.now() },
          headers: { "x-rate-limit-remaining": "100" }
        }
      })

    // Strategy based on current rate limit usage
    const getOptimalStrategy = Effect.gen(function* () {
      const now = Date.now()
      const lastResetTime = yield* Ref.get(lastReset)
      const currentCount = yield* Ref.get(requestCounter)
      
      // Reset counter if more than 1 second has passed
      if (now - lastResetTime > 1000) {
        yield* Ref.set(requestCounter, 0)
        yield* Ref.set(lastReset, now)
        return ExecutionStrategy.parallelN(maxRequestsPerSecond)
      }
      
      const remainingQuota = maxRequestsPerSecond - currentCount
      
      if (remainingQuota <= 0) {
        return ExecutionStrategy.sequential
      } else if (remainingQuota < maxRequestsPerSecond / 2) {
        return ExecutionStrategy.parallelN(Math.max(1, Math.floor(remainingQuota / 2)))
      } else {
        return ExecutionStrategy.parallelN(remainingQuota)
      }
    })

    return { request, getOptimalStrategy } as const satisfies ApiClient & {
      getOptimalStrategy: Effect.Effect<ExecutionStrategy.ExecutionStrategy>
    }
  })

// Batch API requests with rate limiting
const fetchUrlsBatch = (urls: readonly string[], maxRequestsPerSecond: number) =>
  Effect.gen(function* () {
    const client = yield* createRateLimitedClient("https://api.example.com", maxRequestsPerSecond)
    const strategy = yield* client.getOptimalStrategy
    
    console.log(`Fetching ${urls.length} URLs with strategy: ${getDescription(strategy)}`)
    
    return yield* Effect.forEach(urls, client.request, {
      concurrency: strategyToConcurrency(strategy)
    })
  }).pipe(
    Effect.retry(Schedule.exponential("100 millis").compose(Schedule.recurs(3)))
  )
```

### Example 3: Database Migration Control

Managing database migrations with different execution strategies for safety:

```typescript
import { Effect, ExecutionStrategy } from "effect"

interface Migration {
  readonly id: string
  readonly name: string
  readonly risk: "low" | "medium" | "high"
  readonly dependencies: readonly string[]
  readonly execute: Effect.Effect<void, MigrationError>
}

class MigrationError {
  readonly _tag = "MigrationError"
  constructor(
    readonly migrationId: string,
    readonly cause: unknown
  ) {}
}

// Migration executor with risk-based strategy selection
const createMigrationExecutor = () => {
  const selectStrategyByRisk = (migrations: readonly Migration[]) => {
    const risks = migrations.map(m => m.risk)
    
    if (risks.some(risk => risk === "high")) {
      // High-risk migrations must run sequentially
      return ExecutionStrategy.sequential
    } else if (risks.some(risk => risk === "medium")) {
      // Medium-risk migrations run with limited parallelism
      return ExecutionStrategy.parallelN(2)
    } else {
      // Low-risk migrations can run in parallel
      return ExecutionStrategy.parallelN(4)
    }
  }

  const executeMigrations = (migrations: readonly Migration[]) =>
    Effect.gen(function* () {
      // Sort by dependencies first
      const sortedMigrations = topologicalSort(migrations)
      const strategy = selectStrategyByRisk(sortedMigrations)
      
      console.log(`Executing ${migrations.length} migrations with strategy: ${getDescription(strategy)}`)
      
      return yield* Effect.forEach(
        sortedMigrations,
        (migration) => migration.execute.pipe(
          Effect.catchAll((cause) => 
            Effect.fail(new MigrationError(migration.id, cause))
          ),
          Effect.withSpan("migration.execute", {
            attributes: {
              "migration.id": migration.id,
              "migration.risk": migration.risk
            }
          })
        ),
        { concurrency: strategyToConcurrency(strategy) }
      )
    })

  return { executeMigrations, selectStrategyByRisk } as const
}

// Helper function for dependency sorting
const topologicalSort = (migrations: readonly Migration[]): Migration[] => {
  // Simple topological sort implementation
  const visited = new Set<string>()
  const result: Migration[] = []
  
  const visit = (migration: Migration) => {
    if (visited.has(migration.id)) return
    
    visited.add(migration.id)
    
    // Visit dependencies first
    for (const depId of migration.dependencies) {
      const dep = migrations.find(m => m.id === depId)
      if (dep) visit(dep)
    }
    
    result.push(migration)
  }
  
  for (const migration of migrations) {
    visit(migration)
  }
  
  return result
}
```

## Advanced Features Deep Dive

### Feature 1: Finalizer Execution Control

ExecutionStrategy can control how finalizers (cleanup operations) are executed within scoped workflows:

#### Basic Finalizer Control

```typescript
import { Effect, ExecutionStrategy, Scope } from "effect"

const createResourceWithFinalizerStrategy = (strategy: ExecutionStrategy.ExecutionStrategy) =>
  Effect.gen(function* () {
    return yield* Effect.finalizersMask(strategy)((restore) =>
      Effect.gen(function* () {
        const resource1 = yield* Effect.acquireRelease(
          Effect.succeed("Resource 1"),
          () => restore(Effect.log("Cleaning up Resource 1"))
        )
        
        const resource2 = yield* Effect.acquireRelease(
          Effect.succeed("Resource 2"), 
          () => restore(Effect.log("Cleaning up Resource 2"))
        )
        
        const resource3 = yield* Effect.acquireRelease(
          Effect.succeed("Resource 3"),
          () => restore(Effect.log("Cleaning up Resource 3"))
        )
        
        return [resource1, resource2, resource3] as const
      })
    )
  })
```

#### Real-World Finalizer Example

```typescript
import { Effect, ExecutionStrategy } from "effect"

interface DatabaseConnection {
  readonly query: (sql: string) => Effect.Effect<unknown[]>
  readonly close: Effect.Effect<void>
}

interface FileHandle {
  readonly write: (data: string) => Effect.Effect<void>
  readonly close: Effect.Effect<void>
}

// Resource cleanup with different strategies
const processDataWithCleanup = (
  cleanupStrategy: ExecutionStrategy.ExecutionStrategy,
  data: readonly string[]
) =>
  Effect.gen(function* () {
    return yield* Effect.finalizersMask(cleanupStrategy)((restore) =>
      Effect.gen(function* () {
        // Acquire multiple resources
        const db = yield* Effect.acquireRelease(
          openDatabaseConnection(),
          (conn) => restore(conn.close.pipe(
            Effect.tap(() => Effect.log("Database connection closed"))
          ))
        )
        
        const logFile = yield* Effect.acquireRelease(
          openLogFile("process.log"),
          (file) => restore(file.close.pipe(
            Effect.tap(() => Effect.log("Log file closed"))
          ))
        )
        
        const tempFiles = yield* Effect.forEach(
          data,
          (item, index) => Effect.acquireRelease(
            createTempFile(`temp_${index}.txt`),
            (file) => restore(file.close.pipe(
              Effect.tap(() => Effect.log(`Temp file ${index} closed`))
            ))
          )
        )
        
        // Process data
        const results = yield* Effect.forEach(
          data,
          (item, index) => processItem(item, db, tempFiles[index], logFile)
        )
        
        return results
      })
    )
  })

// Mock implementations
const openDatabaseConnection = (): Effect.Effect<DatabaseConnection> =>
  Effect.succeed({
    query: (sql: string) => Effect.succeed([]),
    close: Effect.log("Closing database connection")
  })

const openLogFile = (path: string): Effect.Effect<FileHandle> =>
  Effect.succeed({
    write: (data: string) => Effect.log(`Writing to log: ${data}`),
    close: Effect.log(`Closing log file: ${path}`)
  })

const createTempFile = (name: string): Effect.Effect<FileHandle> =>
  Effect.succeed({
    write: (data: string) => Effect.log(`Writing to temp file ${name}: ${data}`),
    close: Effect.log(`Closing temp file: ${name}`)
  })

const processItem = (
  item: string,
  db: DatabaseConnection,
  tempFile: FileHandle,
  logFile: FileHandle
) =>
  Effect.gen(function* () {
    yield* logFile.write(`Processing: ${item}`)
    yield* tempFile.write(`Data: ${item}`)
    const result = yield* db.query(`SELECT * FROM items WHERE name = '${item}'`)
    return { item, result }
  })
```

### Feature 2: Advanced Strategy Composition

Combining execution strategies for complex workflows:

```typescript
import { Effect, ExecutionStrategy, pipe } from "effect"

// Strategy composition for multi-stage processing
const createProcessingPipeline = <A, B, C>(
  items: readonly A[],
  stage1: (item: A) => Effect.Effect<B>,
  stage2: (item: B) => Effect.Effect<C>,
  strategies: {
    readonly stage1: ExecutionStrategy.ExecutionStrategy
    readonly stage2: ExecutionStrategy.ExecutionStrategy
    readonly cleanup: ExecutionStrategy.ExecutionStrategy
  }
) =>
  Effect.gen(function* () {
    // Stage 1: Initial processing
    const stage1Results = yield* Effect.forEach(
      items,
      stage1,
      { concurrency: strategyToConcurrency(strategies.stage1) }
    )
    
    // Stage 2: Secondary processing with different strategy
    const stage2Results = yield* Effect.forEach(
      stage1Results,
      stage2,
      { concurrency: strategyToConcurrency(strategies.stage2) }
    )
    
    return stage2Results
  }).pipe(
    Effect.finalizersMask(strategies.cleanup)((restore) => 
      Effect.succeed(stage2Results => stage2Results)
    )
  )
```

## Practical Patterns & Best Practices

### Pattern 1: Strategy Selection Helper

```typescript
import { ExecutionStrategy, Effect } from "effect"

interface WorkloadAnalysis {
  readonly itemCount: number
  readonly estimatedDuration: number
  readonly resourceIntensity: "low" | "medium" | "high"
  readonly errorTolerance: "strict" | "lenient"
}

const selectOptimalStrategy = (analysis: WorkloadAnalysis): ExecutionStrategy.ExecutionStrategy => {
  // For strict error tolerance, prefer sequential to fail fast
  if (analysis.errorTolerance === "strict") {
    return ExecutionStrategy.sequential
  }
  
  // For high resource intensity, limit concurrency
  if (analysis.resourceIntensity === "high") {
    return ExecutionStrategy.parallelN(2)
  }
  
  // For small workloads, parallel is fine
  if (analysis.itemCount <= 10) {
    return ExecutionStrategy.parallel
  }
  
  // For large workloads, use bounded parallelism
  const optimalConcurrency = Math.min(
    Math.ceil(analysis.itemCount / 5),
    8 // Never exceed 8 concurrent operations
  )
  
  return ExecutionStrategy.parallelN(optimalConcurrency)
}

// Usage helper for common patterns
const createProcessingConfig = (
  workload: WorkloadAnalysis,
  options?: {
    readonly maxConcurrency?: number
    readonly preferSequential?: boolean
  }
) => {
  if (options?.preferSequential) {
    return {
      strategy: ExecutionStrategy.sequential,
      concurrency: 1 as const
    }
  }
  
  const strategy = selectOptimalStrategy(workload)
  const maxConcurrency = options?.maxConcurrency ?? 8
  
  return {
    strategy,
    concurrency: ExecutionStrategy.match(strategy, {
      onSequential: () => 1 as const,
      onParallel: () => "unbounded" as const,
      onParallelN: (n) => Math.min(n, maxConcurrency)
    })
  }
}
```

### Pattern 2: Dynamic Strategy Adjustment

```typescript
import { Effect, ExecutionStrategy, Ref, Schedule } from "effect"

interface PerformanceMetrics {
  readonly throughput: number
  readonly errorRate: number
  readonly avgResponseTime: number
}

// Adaptive execution strategy that adjusts based on performance
const createAdaptiveExecutor = <A, B, E>(
  processor: (item: A) => Effect.Effect<B, E>
) =>
  Effect.gen(function* () {
    const currentStrategy = yield* Ref.make<ExecutionStrategy.ExecutionStrategy>(
      ExecutionStrategy.parallelN(4)
    )
    const metrics = yield* Ref.make<PerformanceMetrics>({
      throughput: 0,
      errorRate: 0,
      avgResponseTime: 0
    })

    const adjustStrategy = (currentMetrics: PerformanceMetrics) => {
      if (currentMetrics.errorRate > 0.1) {
        // High error rate - reduce concurrency
        return ExecutionStrategy.parallelN(2)
      } else if (currentMetrics.avgResponseTime > 1000) {
        // Slow responses - reduce concurrency
        return ExecutionStrategy.parallelN(3)
      } else if (currentMetrics.throughput > 100 && currentMetrics.errorRate < 0.01) {
        // Good performance - increase concurrency
        return ExecutionStrategy.parallelN(8)
      } else {
        // Maintain current strategy
        return ExecutionStrategy.parallelN(4)
      }
    }

    const processWithAdaptation = (items: readonly A[]) =>
      Effect.gen(function* () {
        const strategy = yield* Ref.get(currentStrategy)
        const startTime = Date.now()
        
        const results = yield* Effect.forEach(
          items,
          processor,
          { concurrency: strategyToConcurrency(strategy) }
        ).pipe(
          Effect.either
        )
        
        const endTime = Date.now()
        const duration = endTime - startTime
        
        // Update metrics
        const errors = results.filter(r => r._tag === "Left").length
        const newMetrics: PerformanceMetrics = {
          throughput: items.length / (duration / 1000),
          errorRate: errors / items.length,
          avgResponseTime: duration / items.length
        }
        
        yield* Ref.set(metrics, newMetrics)
        
        // Adjust strategy for next batch
        const newStrategy = adjustStrategy(newMetrics)
        yield* Ref.set(currentStrategy, newStrategy)
        
        return results
      })

    return { processWithAdaptation, getCurrentStrategy: Ref.get(currentStrategy) } as const
  })
```

### Pattern 3: Error-Aware Strategy Selection

```typescript
import { Effect, ExecutionStrategy, Cause } from "effect"

// Strategy that adapts based on error types
const createErrorAwareProcessor = <A, B, E>(
  processor: (item: A) => Effect.Effect<B, E>,
  errorAnalyzer: (error: E) => "retriable" | "fatal" | "rate_limit"
) => {
  const processWithErrorHandling = (
    items: readonly A[],
    initialStrategy: ExecutionStrategy.ExecutionStrategy
  ) =>
    Effect.gen(function* () {
      let currentStrategy = initialStrategy
      let remainingItems = [...items]
      const results: B[] = []
      
      while (remainingItems.length > 0) {
        const batchSize = ExecutionStrategy.match(currentStrategy, {
          onSequential: () => 1,
          onParallel: () => remainingItems.length,
          onParallelN: (n) => Math.min(n, remainingItems.length)
        })
        
        const batch = remainingItems.slice(0, batchSize)
        remainingItems = remainingItems.slice(batchSize)
        
        const batchResults = yield* Effect.forEach(
          batch,
          processor,
          { concurrency: strategyToConcurrency(currentStrategy) }
        ).pipe(
          Effect.either
        )
        
        let shouldRetry = false
        const retriableItems: A[] = []
        
        for (let i = 0; i < batchResults.length; i++) {
          const result = batchResults[i]
          if (result._tag === "Right") {
            results.push(result.right)
          } else {
            const errorType = errorAnalyzer(result.left)
            
            switch (errorType) {
              case "retriable":
                retriableItems.push(batch[i])
                shouldRetry = true
                break
              case "rate_limit":
                // Switch to sequential for rate limiting
                currentStrategy = ExecutionStrategy.sequential
                retriableItems.push(batch[i])
                shouldRetry = true
                yield* Effect.sleep("1 second")
                break
              case "fatal":
                return yield* Effect.fail(result.left)
            }
          }
        }
        
        if (shouldRetry) {
          remainingItems = [...retriableItems, ...remainingItems]
        }
      }
      
      return results
    })

  return { processWithErrorHandling } as const
}
```

## Integration Examples

### Integration with Effect Runtime

```typescript
import { Effect, ExecutionStrategy, Runtime, FiberRef } from "effect"

// Custom runtime with execution strategy configuration
const createConfigurableRuntime = () => {
  const defaultStrategy = FiberRef.unsafeMake(ExecutionStrategy.parallelN(4))
  
  const runtime = Runtime.defaultRuntime.pipe(
    Runtime.provideService(
      "ExecutionStrategy",
      ExecutionStrategy.parallelN(4)
    )
  )
  
  const runWithStrategy = <A, E>(
    effect: Effect.Effect<A, E>,
    strategy: ExecutionStrategy.ExecutionStrategy
  ) =>
    effect.pipe(
      Effect.locally(defaultStrategy, strategy),
      Runtime.runPromise(runtime)
    )
  
  return { runtime, runWithStrategy, defaultStrategy } as const
}

// Usage with custom runtime
const processItemsWithRuntime = (items: readonly string[]) => {
  const { runWithStrategy } = createConfigurableRuntime()
  
  const processing = Effect.forEach(
    items,
    (item) => Effect.succeed(item.toUpperCase()),
    { concurrency: "inherit" }
  )
  
  // Run with different strategies
  return Promise.all([
    runWithStrategy(processing, ExecutionStrategy.sequential),
    runWithStrategy(processing, ExecutionStrategy.parallel),
    runWithStrategy(processing, ExecutionStrategy.parallelN(2))
  ])
}
```

### Testing Strategies

```typescript
import { Effect, ExecutionStrategy, TestServices, TestClock } from "effect"
import { expect, test } from "vitest"

// Test utilities for execution strategies
const createTestProcessor = (delay: number, failureRate: number = 0) =>
  (item: string) =>
    Effect.gen(function* () {
      yield* TestClock.adjust(delay)
      
      if (Math.random() < failureRate) {
        return yield* Effect.fail(new Error(`Failed to process ${item}`))
      }
      
      return `processed-${item}`
    })

test("ExecutionStrategy performance comparison", async () => {
  const items = ["a", "b", "c", "d", "e"]
  const processor = createTestProcessor(100) // 100ms delay per item
  
  const testSequential = Effect.gen(function* () {
    const start = yield* TestClock.currentTimeMillis
    const results = yield* Effect.forEach(items, processor, { concurrency: 1 })
    const end = yield* TestClock.currentTimeMillis
    return { results, duration: end - start }
  })
  
  const testParallel = Effect.gen(function* () {
    const start = yield* TestClock.currentTimeMillis
    const results = yield* Effect.forEach(items, processor, { concurrency: "unbounded" })
    const end = yield* TestClock.currentTimeMillis
    return { results, duration: end - start }
  })
  
  const program = Effect.gen(function* () {
    const sequential = yield* testSequential
    const parallel = yield* testParallel
    
    return { sequential, parallel }
  })
  
  const result = await Effect.runPromise(
    program.pipe(Effect.provide(TestServices.TestServices))
  )
  
  // Sequential should take 500ms (5 * 100ms)
  expect(result.sequential.duration).toBe(500)
  
  // Parallel should take 100ms (all concurrent)
  expect(result.parallel.duration).toBe(100)
  
  // Both should produce same results
  expect(result.sequential.results).toEqual(result.parallel.results)
})

test("ExecutionStrategy with bounded concurrency", async () => {
  const items = Array.from({ length: 10 }, (_, i) => `item-${i}`)
  const processor = createTestProcessor(100)
  
  const testBounded = Effect.gen(function* () {
    const start = yield* TestClock.currentTimeMillis
    const results = yield* Effect.forEach(items, processor, { concurrency: 3 })
    const end = yield* TestClock.currentTimeMillis
    return { results, duration: end - start }
  })
  
  const result = await Effect.runPromise(
    testBounded.pipe(Effect.provide(TestServices.TestServices))
  )
  
  // With 10 items, concurrency 3, and 100ms per item:
  // First 3 items: 0-100ms
  // Next 3 items: 100-200ms  
  // Next 3 items: 200-300ms
  // Last 1 item: 300-400ms
  expect(result.duration).toBe(400)
  expect(result.results).toHaveLength(10)
})
```

### Property-Based Testing

```typescript
import { Effect, ExecutionStrategy } from "effect"
import { test, expect } from "vitest"
import { fc } from "fast-check"

test("ExecutionStrategy properties", () => {
  fc.assert(
    fc.property(
      fc.array(fc.string(), { minLength: 1, maxLength: 20 }),
      fc.integer({ min: 1, max: 10 }),
      async (items, parallelism) => {
        const processor = (item: string) => Effect.succeed(item.toUpperCase())
        
        const sequential = await Effect.runPromise(
          Effect.forEach(items, processor, { concurrency: 1 })
        )
        
        const parallel = await Effect.runPromise(
          Effect.forEach(items, processor, { concurrency: "unbounded" })
        )
        
        const boundedParallel = await Effect.runPromise(
          Effect.forEach(items, processor, { concurrency: parallelism })
        )
        
        // All strategies should produce the same results
        expect(sequential).toEqual(parallel)
        expect(parallel).toEqual(boundedParallel)
        
        // Results should maintain input order
        expect(sequential).toEqual(items.map(item => item.toUpperCase()))
      }
    )
  )
})
```

## Conclusion

ExecutionStrategy provides fine-grained control over effect execution, resource management, and performance optimization for Effect-based applications.

Key benefits:
- **Performance Control**: Choose the right execution pattern for your workload
- **Resource Management**: Prevent resource exhaustion with bounded parallelism
- **Flexibility**: Adapt execution strategy based on runtime conditions

ExecutionStrategy is essential when you need to balance performance, resource usage, and reliability in concurrent Effect applications. Use sequential execution for strict ordering requirements, parallel execution for maximum throughput, and parallelN for controlled resource usage.