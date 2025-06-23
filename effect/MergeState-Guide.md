# MergeState: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem MergeState Solves

When merging two concurrent streams or channels in traditional approaches, managing the lifecycle and coordination becomes complex and error-prone:

```typescript
// Traditional approach - manual state tracking
class ManualMergeManager {
  private leftRunning = false
  private rightRunning = false
  private leftResult: any = null
  private rightResult: any = null

  async merge(leftStream: AsyncIterable<any>, rightStream: AsyncIterable<any>) {
    // Complex state management with race conditions
    this.leftRunning = true
    this.rightRunning = true
    
    // Manual coordination - prone to bugs
    const leftPromise = this.processLeft(leftStream)
    const rightPromise = this.processRight(rightStream)
    
    // Error-prone cleanup and result handling
    try {
      await Promise.race([leftPromise, rightPromise])
      // Complex state transitions...
    } catch (error) {
      // Manual cleanup required
    }
  }
}
```

This approach leads to:
- **Race Conditions** - Manual state management creates timing issues
- **Resource Leaks** - Difficult to properly cleanup when one side completes
- **Complex Error Handling** - No clear error propagation strategy
- **State Confusion** - Hard to track which side is running vs completed

### The MergeState Solution

MergeState provides a type-safe, composable way to manage the lifecycle of concurrent merge operations:

```typescript
import { MergeState, Effect, Fiber, Either } from "effect"

// Clean, type-safe state management
const createMergeOperation = <A, B>(
  leftFiber: Fiber.Fiber<Either.Either<A, string>, Error>,
  rightFiber: Fiber.Fiber<Either.Either<B, string>, Error>
): MergeState.MergeState<never, Error, Error, Error, A | B, string, string, string> =>
  MergeState.BothRunning(leftFiber, rightFiber)
```

### Key Concepts

**BothRunning**: Represents the state where both sides of a merge are actively executing concurrently

**LeftDone**: Represents the state where the left side has completed, with a continuation function for handling the right side's completion

**RightDone**: Represents the state where the right side has completed, with a continuation function for handling the left side's completion

## Basic Usage Patterns

### Pattern 1: Creating Initial Merge State

```typescript
import { MergeState, Effect, Fiber, Either } from "effect"

// Setup two concurrent operations
const createConcurrentOperations = Effect.gen(function* () {
  const leftFiber = yield* Effect.fork(
    Effect.succeed(Either.right("left-result"))
  )
  const rightFiber = yield* Effect.fork(
    Effect.succeed(Either.right("right-result"))
  )
  
  return MergeState.BothRunning(leftFiber, rightFiber)
})
```

### Pattern 2: State Transitions

```typescript
// Handle when left side completes first
const handleLeftCompletion = (
  rightFiber: Fiber.Fiber<Either.Either<string, number>, Error>
) => 
  MergeState.LeftDone((rightExit) =>
    Effect.gen(function* () {
      const rightResult = yield* Effect.fromExit(rightExit)
      return `Left done, right result: ${rightResult}`
    })
  )

// Handle when right side completes first  
const handleRightCompletion = (
  leftFiber: Fiber.Fiber<Either.Either<string, number>, Error>
) =>
  MergeState.RightDone((leftExit) =>
    Effect.gen(function* () {
      const leftResult = yield* Effect.fromExit(leftExit)
      return `Right done, left result: ${leftResult}`
    })
  )
```

### Pattern 3: State Pattern Matching

```typescript
const processMessage = (state: MergeState.MergeState<never, Error, Error, Error, string, number, number, string>) =>
  MergeState.match(state, {
    onBothRunning: (left, right) => `Both sides running: ${left} + ${right}`,
    onLeftDone: (continuation) => "Left side completed, waiting for right",
    onRightDone: (continuation) => "Right side completed, waiting for left"
  })
```

## Real-World Examples

### Example 1: Database and Cache Merge

Managing concurrent database and cache operations while ensuring proper cleanup:

```typescript
import { MergeState, Effect, Fiber, Either, Exit } from "effect"

interface DatabaseResult {
  data: string[]
  fromCache: boolean
}

interface CacheService {
  get: (key: string) => Effect.Effect<string[], Error>
}

interface DatabaseService {
  query: (sql: string) => Effect.Effect<string[], Error>
}

const fetchUserData = (userId: string) =>
  Effect.gen(function* () {
    const cacheService = yield* CacheService
    const dbService = yield* DatabaseService
    
    // Start both operations concurrently
    const cacheFiber = yield* Effect.fork(
      cacheService.get(`user:${userId}`).pipe(
        Effect.map(data => Either.right({ data, fromCache: true }))
      )
    )
    
    const dbFiber = yield* Effect.fork(
      dbService.query(`SELECT * FROM users WHERE id = '${userId}'`).pipe(
        Effect.map(data => Either.right({ data, fromCache: false }))
      )
    )
    
    // Create merge state to coordinate results
    const initialState = MergeState.BothRunning(cacheFiber, dbFiber)
    
    return yield* coordinateDataFetch(initialState)
  })

const coordinateDataFetch = (
  state: MergeState.MergeState<
    never, 
    Error, 
    Error, 
    Error, 
    DatabaseResult, 
    DatabaseResult, 
    DatabaseResult, 
    DatabaseResult
  >
) =>
  MergeState.match(state, {
    onBothRunning: (cacheFiber, dbFiber) =>
      Effect.gen(function* () {
        // Race the operations - first one wins
        const result = yield* Fiber.raceWith(cacheFiber, dbFiber, {
          onSelfDone: (cacheExit, dbFiber) =>
            Effect.gen(function* () {
              yield* Fiber.interrupt(dbFiber) // Cancel DB if cache wins
              return yield* Effect.fromExit(cacheExit)
            }),
          onOtherDone: (dbExit, cacheFiber) =>
            Effect.gen(function* () {
              yield* Fiber.interrupt(cacheFiber) // Cancel cache if DB wins
              return yield* Effect.fromExit(dbExit)
            })
        })
        return result
      }),
    onLeftDone: (continuation) =>
      Effect.gen(function* () {
        // Cache completed, handle DB result
        return yield* continuation(Exit.succeed({ data: [], fromCache: false }))
      }),
    onRightDone: (continuation) =>
      Effect.gen(function* () {
        // DB completed, handle cache result  
        return yield* continuation(Exit.succeed({ data: [], fromCache: true }))
      })
  })
```

### Example 2: API Response Merging with Fallback

Combining primary and fallback API responses with proper error handling:

```typescript
import { MergeState, Effect, Fiber, Either, Exit, Array as Arr } from "effect"

interface ApiResponse {
  status: "success" | "error"
  data: any[]
  source: "primary" | "fallback"
}

interface ApiService {
  fetchPrimary: (endpoint: string) => Effect.Effect<any[], Error>
  fetchFallback: (endpoint: string) => Effect.Effect<any[], Error>
}

const fetchWithFallback = (endpoint: string) =>
  Effect.gen(function* () {
    const apiService = yield* ApiService
    
    // Start both API calls
    const primaryFiber = yield* Effect.fork(
      apiService.fetchPrimary(endpoint).pipe(
        Effect.map(data => Either.right({ status: "success" as const, data, source: "primary" as const })),
        Effect.catchAll(error => 
          Effect.succeed(Either.left(error))
        )
      )
    )
    
    const fallbackFiber = yield* Effect.fork(
      apiService.fetchFallback(endpoint).pipe(
        Effect.map(data => Either.right({ status: "success" as const, data, source: "fallback" as const })),
        Effect.catchAll(error =>
          Effect.succeed(Either.left(error))
        )
      )
    )
    
    const mergeState = MergeState.BothRunning(primaryFiber, fallbackFiber)
    return yield* handleApiMerge(mergeState)
  })

const handleApiMerge = (
  state: MergeState.MergeState<never, never, never, Error, ApiResponse, ApiResponse, ApiResponse, ApiResponse>
) =>
  MergeState.match(state, {
    onBothRunning: (primaryFiber, fallbackFiber) =>
      Effect.gen(function* () {
        // Primary wins if successful, otherwise use fallback
        return yield* Fiber.raceWith(primaryFiber, fallbackFiber, {
          onSelfDone: (primaryExit, fallbackFiber) =>
            Effect.gen(function* () {
              const primaryResult = yield* Effect.fromExit(primaryExit)
              if (Either.isRight(primaryResult)) {
                yield* Fiber.interrupt(fallbackFiber)
                return primaryResult.right
              }
              // Primary failed, wait for fallback
              const fallbackResult = yield* Fiber.join(fallbackFiber)
              return Either.isRight(fallbackResult) 
                ? fallbackResult.right 
                : { status: "error" as const, data: [], source: "fallback" as const }
            }),
          onOtherDone: (fallbackExit, primaryFiber) =>
            Effect.gen(function* () {
              // Still prefer primary even if fallback completes first
              const primaryResult = yield* Fiber.join(primaryFiber)
              if (Either.isRight(primaryResult)) {
                return primaryResult.right
              }
              const fallbackResult = yield* Effect.fromExit(fallbackExit)
              return Either.isRight(fallbackResult)
                ? fallbackResult.right
                : { status: "error" as const, data: [], source: "primary" as const }
            })
        })
      }),
    onLeftDone: (continuation) =>
      Effect.gen(function* () {
        // Primary is done, handle fallback completion
        return yield* continuation(Exit.succeed({ 
          status: "success" as const, 
          data: [], 
          source: "fallback" as const 
        }))
      }),
    onRightDone: (continuation) =>
      Effect.gen(function* () {
        // Fallback is done, handle primary completion
        return yield* continuation(Exit.succeed({ 
          status: "success" as const, 
          data: [], 
          source: "primary" as const 
        }))
      })
  })
```

### Example 3: Stream Processing Pipeline

Managing multiple data processing streams with backpressure:

```typescript
import { MergeState, Effect, Fiber, Either, Exit, Stream, Queue, Array as Arr } from "effect"

interface ProcessingResult {
  processed: number
  errors: number
  source: string
}

interface StreamProcessor {
  processA: (data: ReadonlyArray<string>) => Effect.Effect<ProcessingResult, Error>
  processB: (data: ReadonlyArray<string>) => Effect.Effect<ProcessingResult, Error>
}

const processDataStreams = (inputData: ReadonlyArray<string>) =>
  Effect.gen(function* () {
    const processor = yield* StreamProcessor
    const queue = yield* Queue.bounded<ProcessingResult>(100)
    
    // Split data for parallel processing
    const dataA = Arr.take(inputData, Math.ceil(inputData.length / 2))
    const dataB = Arr.drop(inputData, Math.ceil(inputData.length / 2))
    
    // Start processing fibers
    const processorAFiber = yield* Effect.fork(
      processor.processA(dataA).pipe(
        Effect.map(result => Either.right({ ...result, source: "processor-a" }))
      )
    )
    
    const processorBFiber = yield* Effect.fork(
      processor.processB(dataB).pipe(
        Effect.map(result => Either.right({ ...result, source: "processor-b" }))
      )
    )
    
    const mergeState = MergeState.BothRunning(processorAFiber, processorBFiber)
    return yield* handleStreamProcessing(mergeState, queue)
  })

const handleStreamProcessing = (
  state: MergeState.MergeState<never, Error, Error, Error, ProcessingResult, ProcessingResult, ProcessingResult, ProcessingResult>,
  resultQueue: Queue.Queue<ProcessingResult>
) =>
  MergeState.match(state, {
    onBothRunning: (processorA, processorB) =>
      Effect.gen(function* () {
        // Collect results as they complete
        const results: ProcessingResult[] = []
        
        return yield* Fiber.raceWith(processorA, processorB, {
          onSelfDone: (aExit, bFiber) =>
            Effect.gen(function* () {
              const aResult = yield* Effect.fromExit(aExit)
              if (Either.isRight(aResult)) {
                yield* Queue.offer(resultQueue, aResult.right)
                results.push(aResult.right)
              }
              
              // Wait for B to complete
              const bResult = yield* Fiber.join(bFiber)
              if (Either.isRight(bResult)) {
                yield* Queue.offer(resultQueue, bResult.right)
                results.push(bResult.right)
              }
              
              return combineResults(results)
            }),
          onOtherDone: (bExit, aFiber) =>
            Effect.gen(function* () {
              const bResult = yield* Effect.fromExit(bExit)
              if (Either.isRight(bResult)) {
                yield* Queue.offer(resultQueue, bResult.right)
                results.push(bResult.right)
              }
              
              // Wait for A to complete
              const aResult = yield* Fiber.join(aFiber)
              if (Either.isRight(aResult)) {
                yield* Queue.offer(resultQueue, aResult.right)
                results.push(aResult.right)
              }
              
              return combineResults(results)
            })
        })
      }),
    onLeftDone: (continuation) =>
      Effect.gen(function* () {
        // Processor A done, finalize with B result
        return yield* continuation(Exit.succeed({
          processed: 0,
          errors: 0,
          source: "processor-b"
        }))
      }),
    onRightDone: (continuation) =>
      Effect.gen(function* () {
        // Processor B done, finalize with A result
        return yield* continuation(Exit.succeed({
          processed: 0,
          errors: 0,
          source: "processor-a"
        }))
      })
  })

const combineResults = (results: ReadonlyArray<ProcessingResult>): ProcessingResult => ({
  processed: results.reduce((sum, r) => sum + r.processed, 0),
  errors: results.reduce((sum, r) => sum + r.errors, 0),
  source: "combined"
})
```

## Advanced Features Deep Dive

### Feature 1: State Inspection and Debugging

Understanding and debugging merge state transitions:

#### Basic State Inspection

```typescript
import { MergeState, Effect } from "effect"

const inspectMergeState = <Env, Err, Err1, Err2, Elem, Done, Done1, Done2>(
  state: MergeState.MergeState<Env, Err, Err1, Err2, Elem, Done, Done1, Done2>
): string => {
  if (MergeState.isBothRunning(state)) {
    return "Both sides are actively running"
  }
  if (MergeState.isLeftDone(state)) {
    return "Left side completed, waiting for right side"
  }
  if (MergeState.isRightDone(state)) {
    return "Right side completed, waiting for left side"
  }
  return "Unknown state"
}
```

#### Real-World State Debugging Example

```typescript
const debugMergeOperation = <A, B>(
  leftOperation: Effect.Effect<A, Error>,
  rightOperation: Effect.Effect<B, Error>
) =>
  Effect.gen(function* () {
    const startTime = Date.now()
    
    const leftFiber = yield* Effect.fork(
      leftOperation.pipe(
        Effect.map(result => Either.right(result)),
        Effect.tap(() => Effect.log("Left operation completed"))
      )
    )
    
    const rightFiber = yield* Effect.fork(
      rightOperation.pipe(
        Effect.map(result => Either.right(result)),
        Effect.tap(() => Effect.log("Right operation completed"))
      )
    )
    
    const initialState = MergeState.BothRunning(leftFiber, rightFiber)
    
    yield* Effect.log(`Merge operation started: ${inspectMergeState(initialState)}`)
    
    const result = yield* monitorMergeProgress(initialState, startTime)
    
    yield* Effect.log(`Merge operation completed in ${Date.now() - startTime}ms`)
    
    return result
  })

const monitorMergeProgress = <A, B>(
  state: MergeState.MergeState<never, Error, Error, Error, A | B, A, B, A | B>,
  startTime: number
) =>
  MergeState.match(state, {
    onBothRunning: (left, right) =>
      Effect.gen(function* () {
        yield* Effect.log("Monitoring both fibers...")
        
        return yield* Fiber.raceWith(left, right, {
          onSelfDone: (leftExit, rightFiber) =>
            Effect.gen(function* () {
              yield* Effect.log(`Left completed after ${Date.now() - startTime}ms`)
              yield* Fiber.interrupt(rightFiber)
              return yield* Effect.fromExit(leftExit)
            }),
          onOtherDone: (rightExit, leftFiber) =>
            Effect.gen(function* () {
              yield* Effect.log(`Right completed after ${Date.now() - startTime}ms`)
              yield* Fiber.interrupt(leftFiber)
              return yield* Effect.fromExit(rightExit)
            })
        })
      }),
    onLeftDone: (continuation) =>
      Effect.gen(function* () {
        yield* Effect.log("Left done, processing right completion...")
        return yield* continuation(Exit.succeed({} as B))
      }),
    onRightDone: (continuation) =>
      Effect.gen(function* () {
        yield* Effect.log("Right done, processing left completion...")
        return yield* continuation(Exit.succeed({} as A))
      })
  })
```

### Feature 2: Advanced Error Handling Patterns

Sophisticated error handling and recovery strategies:

#### Error Recovery with State Transitions

```typescript
import { MergeState, Effect, Exit, Cause, Array as Arr } from "effect"

interface RetryableOperation<A> {
  operation: Effect.Effect<A, Error>
  maxRetries: number
  backoffMs: number
}

const resilientMerge = <A, B>(
  leftOp: RetryableOperation<A>,
  rightOp: RetryableOperation<B>
) =>
  Effect.gen(function* () {
    const leftFiber = yield* Effect.fork(
      retryWithBackoff(leftOp).pipe(
        Effect.map(result => Either.right(result))
      )
    )
    
    const rightFiber = yield* Effect.fork(
      retryWithBackoff(rightOp).pipe(
        Effect.map(result => Either.right(result))
      )
    )
    
    const mergeState = MergeState.BothRunning(leftFiber, rightFiber)
    return yield* handleResilientMerge(mergeState)
  })

const retryWithBackoff = <A>(config: RetryableOperation<A>): Effect.Effect<A, Error> =>
  config.operation.pipe(
    Effect.retry(
      Effect.scheduleWith(Effect.exponential(config.backoffMs), schedule => 
        schedule.pipe(Effect.take(config.maxRetries))
      )
    )
  )

const handleResilientMerge = <A, B>(
  state: MergeState.MergeState<never, Error, Error, Error, A | B, A, B, A | B>
) =>
  MergeState.match(state, {
    onBothRunning: (left, right) =>
      Effect.gen(function* () {
        return yield* Fiber.raceWith(left, right, {
          onSelfDone: (leftExit, rightFiber) =>
            Effect.gen(function* () {
              if (Exit.isFailure(leftExit)) {
                // Left failed, check if right can succeed
                const rightResult = yield* Fiber.join(rightFiber)
                if (Either.isRight(rightResult)) {
                  return rightResult.right
                }
                // Both failed
                return yield* Effect.fail(new Error("Both operations failed"))
              }
              
              yield* Fiber.interrupt(rightFiber)
              return yield* Effect.fromExit(leftExit)
            }),
          onOtherDone: (rightExit, leftFiber) =>
            Effect.gen(function* () {
              if (Exit.isFailure(rightExit)) {
                // Right failed, check if left can succeed
                const leftResult = yield* Fiber.join(leftFiber)
                if (Either.isRight(leftResult)) {
                  return leftResult.right
                }
                // Both failed
                return yield* Effect.fail(new Error("Both operations failed"))
              }
              
              yield* Fiber.interrupt(leftFiber)
              return yield* Effect.fromExit(rightExit)
            })
        })
      }),
    onLeftDone: (continuation) =>
      Effect.gen(function* () {
        // Left completed, handle right with fallback
        return yield* continuation(Exit.fail(new Error("Right operation timeout")))
      }),
    onRightDone: (continuation) =>
      Effect.gen(function* () {
        // Right completed, handle left with fallback
        return yield* continuation(Exit.fail(new Error("Left operation timeout")))
      })
  })
```

### Feature 3: Performance Optimization Patterns

Optimizing merge operations for high-throughput scenarios:

#### Batched Result Collection

```typescript
import { MergeState, Effect, Fiber, Either, Ref, Array as Arr } from "effect"

interface BatchProcessor<T> {
  batchSize: number
  flushIntervalMs: number
  processor: (batch: ReadonlyArray<T>) => Effect.Effect<ReadonlyArray<T>, Error>
}

const createBatchedMerge = <A, B>(
  leftProcessor: BatchProcessor<A>,
  rightProcessor: BatchProcessor<B>
) =>
  Effect.gen(function* () {
    const leftBatch = yield* Ref.make<A[]>([])
    const rightBatch = yield* Ref.make<B[]>([])
    const results = yield* Ref.make<(A | B)[]>([])
    
    const leftFiber = yield* Effect.fork(
      processBatch(leftProcessor, leftBatch, results)
    )
    
    const rightFiber = yield* Effect.fork(
      processBatch(rightProcessor, rightBatch, results)
    )
    
    const mergeState = MergeState.BothRunning(leftFiber, rightFiber)
    return { state: mergeState, results }
  })

const processBatch = <T>(
  processor: BatchProcessor<T>,
  batchRef: Ref.Ref<T[]>,
  resultsRef: Ref.Ref<(T)[]>
): Effect.Effect<Either.Either<T[], ReadonlyArray<T>>, Error> =>
  Effect.gen(function* () {
    while (true) {
      const batch = yield* Ref.get(batchRef)
      
      if (batch.length >= processor.batchSize) {
        yield* Ref.set(batchRef, [])
        const processed = yield* processor.processor(batch)
        yield* Ref.update(resultsRef, current => [...current, ...processed])
      }
      
      yield* Effect.sleep(processor.flushIntervalMs)
    }
  }).pipe(
    Effect.map(results => Either.right(results))
  )
```

## Practical Patterns & Best Practices

### Pattern 1: Resource Cleanup Helper

```typescript
import { MergeState, Effect, Scope, Resource } from "effect"

const createManagedMerge = <A, B, R1, R2>(
  leftResource: Effect.Effect<A, Error, R1>,
  rightResource: Effect.Effect<B, Error, R2>
) =>
  Effect.scoped(
    Effect.gen(function* () {
      const scope = yield* Scope.make()
      
      const leftFiber = yield* Effect.fork(
        Scope.extend(leftResource, scope).pipe(
          Effect.map(result => Either.right(result))
        )
      )
      
      const rightFiber = yield* Effect.fork(
        Scope.extend(rightResource, scope).pipe(
          Effect.map(result => Either.right(result))
        )
      )
      
      const mergeState = MergeState.BothRunning(leftFiber, rightFiber)
      
      return yield* Effect.ensuring(
        processManagedMerge(mergeState),
        Scope.close(scope, Exit.succeed(undefined))
      )
    })
  )

const processManagedMerge = <A, B>(
  state: MergeState.MergeState<never, Error, Error, Error, A | B, A, B, A | B>
) =>
  MergeState.match(state, {
    onBothRunning: (left, right) =>
      Effect.gen(function* () {
        // Proper cleanup guaranteed by scoped resource management
        return yield* Fiber.raceWith(left, right, {
          onSelfDone: (leftExit, rightFiber) =>
            Effect.gen(function* () {
              yield* Fiber.interrupt(rightFiber)
              return yield* Effect.fromExit(leftExit)
            }),
          onOtherDone: (rightExit, leftFiber) =>
            Effect.gen(function* () {
              yield* Fiber.interrupt(leftFiber)
              return yield* Effect.fromExit(rightExit)
            })
        })
      }),
    onLeftDone: (continuation) => continuation(Exit.succeed({} as B)),
    onRightDone: (continuation) => continuation(Exit.succeed({} as A))
  })
```

### Pattern 2: Timeout and Cancellation Helper

```typescript
import { MergeState, Effect, Fiber, Schedule, Duration } from "effect"

const createTimeoutMerge = <A, B>(
  leftOp: Effect.Effect<A, Error>,
  rightOp: Effect.Effect<B, Error>,
  timeoutMs: number
) =>
  Effect.gen(function* () {
    const leftFiber = yield* Effect.fork(
      leftOp.pipe(
        Effect.timeout(Duration.millis(timeoutMs)),
        Effect.map(opt => opt ? Either.right(opt) : Either.left(new Error("Left timeout")))
      )
    )
    
    const rightFiber = yield* Effect.fork(
      rightOp.pipe(
        Effect.timeout(Duration.millis(timeoutMs)),
        Effect.map(opt => opt ? Either.right(opt) : Either.left(new Error("Right timeout")))
      )
    )
    
    const mergeState = MergeState.BothRunning(leftFiber, rightFiber)
    return yield* processTimeoutMerge(mergeState)
  })

const processTimeoutMerge = <A, B>(
  state: MergeState.MergeState<never, Error, Error, Error, A | B, A | undefined, B | undefined, A | B>
) =>
  MergeState.match(state, {
    onBothRunning: (left, right) =>
      Effect.gen(function* () {
        const timeoutFiber = yield* Effect.fork(
          Effect.sleep(Duration.seconds(30)).pipe(
            Effect.as(Either.left(new Error("Global timeout")))
          )
        )
        
        return yield* Fiber.raceWith(
          Fiber.race(left, right),
          timeoutFiber,
          {
            onSelfDone: (resultExit, timeoutFiber) =>
              Effect.gen(function* () {
                yield* Fiber.interrupt(timeoutFiber)
                return yield* Effect.fromExit(resultExit)
              }),
            onOtherDone: (timeoutExit, resultFiber) =>
              Effect.gen(function* () {
                yield* Fiber.interrupt(resultFiber)
                return yield* Effect.fail(new Error("Operation timed out"))
              })
          }
        )
      }),
    onLeftDone: (continuation) =>
      Effect.gen(function* () {
        return yield* continuation(Exit.succeed(undefined))
      }),
    onRightDone: (continuation) =>
      Effect.gen(function* () {
        return yield* continuation(Exit.succeed(undefined))
      })
  })
```

## Integration Examples

### Integration with Stream Processing

```typescript
import { MergeState, Stream, Effect, Fiber, Either, Queue } from "effect"

const mergeStreams = <A, B>(
  leftStream: Stream.Stream<A, Error>,
  rightStream: Stream.Stream<B, Error>
) =>
  Effect.gen(function* () {
    const outputQueue = yield* Queue.bounded<A | B>(1000)
    
    const leftFiber = yield* Effect.fork(
      Stream.runForEach(leftStream, item => Queue.offer(outputQueue, item)).pipe(
        Effect.map(() => Either.right("left-complete"))
      )
    )
    
    const rightFiber = yield* Effect.fork(
      Stream.runForEach(rightStream, item => Queue.offer(outputQueue, item)).pipe(
        Effect.map(() => Either.right("right-complete"))
      )
    )
    
    const mergeState = MergeState.BothRunning(leftFiber, rightFiber)
    
    return {
      outputStream: Stream.fromQueue(outputQueue),
      controller: mergeState
    }
  })
```

### Testing Strategies

```typescript
import { MergeState, Effect, TestContext, Clock, Fiber } from "effect"

const testMergeStateBehavior = Effect.gen(function* () {
  const testContext = yield* TestContext.TestContext
  
  // Create predictable test operations
  const slowOperation = Effect.sleep(Duration.seconds(2)).pipe(
    Effect.as("slow-result")
  )
  
  const fastOperation = Effect.sleep(Duration.seconds(1)).pipe(
    Effect.as("fast-result")
  )
  
  const leftFiber = yield* Effect.fork(
    slowOperation.pipe(Effect.map(result => Either.right(result)))
  )
  
  const rightFiber = yield* Effect.fork(
    fastOperation.pipe(Effect.map(result => Either.right(result)))
  )
  
  const mergeState = MergeState.BothRunning(leftFiber, rightFiber)
  
  // Assert initial state
  const isBothRunning = MergeState.isBothRunning(mergeState)
  yield* Effect.log(`Initial state is BothRunning: ${isBothRunning}`)
  
  // Advance test clock and observe state transitions
  yield* TestContext.advance(Duration.seconds(1))
  
  // Test state transitions
  const result = yield* MergeState.match(mergeState, {
    onBothRunning: (left, right) => Effect.succeed("still-running"),
    onLeftDone: (continuation) => Effect.succeed("left-done"),
    onRightDone: (continuation) => Effect.succeed("right-done")
  })
  
  yield* Effect.log(`Test result: ${result}`)
  return result
})
```

## Conclusion

MergeState provides **type-safe concurrent coordination**, **predictable state transitions**, and **composable error handling** for complex merge operations.

Key benefits:
- **Type Safety**: Compile-time guarantees about state transitions and data flow
- **Resource Management**: Automatic cleanup and proper fiber lifecycle management
- **Composability**: Works seamlessly with other Effect modules like Stream, Queue, and Scope

MergeState is essential when you need to coordinate multiple concurrent operations with complex completion semantics, proper error handling, and resource cleanup - particularly in stream processing, API aggregation, and distributed system coordination scenarios.