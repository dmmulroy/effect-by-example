# Deferred: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem Deferred Solves

Imagine you're building a real-time application where multiple parts of your system need to wait for a single event to complete - like waiting for user authentication, file processing, or API responses. Traditional approaches often lead to complex callback management or Promise-based solutions that lack type safety and composability:

```typescript
// Traditional approach - complex callback coordination
let authResult: User | null = null
const waitingCallbacks: Array<(user: User) => void> = []
const errorCallbacks: Array<(error: Error) => void> = []

function authenticateUser(token: string) {
  // Complex state management
  performAuth(token, (user) => {
    authResult = user
    waitingCallbacks.forEach(cb => cb(user))
    waitingCallbacks.length = 0
  }, (error) => {
    errorCallbacks.forEach(cb => cb(error))
    errorCallbacks.length = 0
  })
}

function waitForAuth(onSuccess: (user: User) => void, onError: (error: Error) => void) {
  if (authResult) {
    onSuccess(authResult)
  } else {
    waitingCallbacks.push(onSuccess)
    errorCallbacks.push(onError)
  }
}
```

This approach leads to:
- **Race Conditions** - Multiple fibers trying to complete the same operation
- **Memory Leaks** - Callbacks accumulating without proper cleanup
- **Type Unsafety** - No compile-time guarantees about success/failure types
- **Error Handling Complexity** - Managing multiple error scenarios manually

### The Deferred Solution

Deferred provides a type-safe, fiber-aware coordination primitive that allows exactly one completion with automatic cleanup and structured concurrency:

```typescript
import { Deferred, Effect } from "effect"

// Create a typed promise-like coordination primitive
const program = Effect.gen(function* () {
  const authDeferred = yield* Deferred.make<User, AuthError>()
  
  // Multiple fibers can safely wait
  const waitForAuth = Deferred.await(authDeferred)
  
  // Only one completion succeeds
  yield* Deferred.succeed(authDeferred, authenticatedUser)
  
  return yield* waitForAuth
})
```

### Key Concepts

**Single Assignment**: A Deferred can be completed exactly once, making it perfect for coordinating one-time events across multiple fibers.

**Fiber-Safe Coordination**: Multiple fibers can safely await the same Deferred without race conditions or resource leaks.

**Type-Safe Results**: Both success and error types are tracked at compile time, ensuring robust error handling.

## Basic Usage Patterns

### Pattern 1: Creating and Completing a Deferred

```typescript
import { Deferred, Effect } from "effect"

// Create a new Deferred for coordinating a string result
const basicExample = Effect.gen(function* () {
  const deferred = yield* Deferred.make<string>()
  
  // Complete the deferred with a value
  yield* Deferred.succeed(deferred, "Hello, Effect!")
  
  // Await the result
  const result = yield* Deferred.await(deferred)
  
  console.log(result) // "Hello, Effect!"
  return result
})
```

### Pattern 2: Error Handling with Deferred

```typescript
import { Deferred, Effect } from "effect"

// Create a Deferred that can fail with a specific error type
const errorExample = Effect.gen(function* () {
  const deferred = yield* Deferred.make<number, string>()
  
  // Complete with failure
  yield* Deferred.fail(deferred, "Something went wrong")
  
  // Await with error handling
  const result = yield* Deferred.await(deferred).pipe(
    Effect.catchAll((error) => {
      console.log(`Caught error: ${error}`)
      return Effect.succeed(-1)
    })
  )
  
  return result
})
```

### Pattern 3: Multiple Waiters

```typescript
import { Deferred, Effect, Fiber } from "effect"

// Multiple fibers waiting for the same result
const multipleWaitersExample = Effect.gen(function* () {
  const deferred = yield* Deferred.make<string>()
  
  // Start multiple fibers that wait for the result
  const fiber1 = yield* Effect.fork(
    Deferred.await(deferred).pipe(
      Effect.map(value => `Fiber 1 got: ${value}`)
    )
  )
  
  const fiber2 = yield* Effect.fork(
    Deferred.await(deferred).pipe(
      Effect.map(value => `Fiber 2 got: ${value}`)
    )
  )
  
  // Complete the deferred - both fibers will receive the result
  yield* Deferred.succeed(deferred, "shared result")
  
  const result1 = yield* Fiber.join(fiber1)
  const result2 = yield* Fiber.join(fiber2)
  
  return [result1, result2]
})
```

## Real-World Examples

### Example 1: Authentication Token Management

Modern applications often need to coordinate authentication across multiple concurrent operations:

```typescript
import { Deferred, Effect, Layer, Context, Ref } from "effect"

// Define our domain types
interface User {
  readonly id: string
  readonly email: string
  readonly permissions: ReadonlyArray<string>
}

class AuthError extends Error {
  readonly _tag = "AuthError"
  constructor(message: string) {
    super(message)
  }
}

// Authentication service interface
class AuthService extends Context.Tag("AuthService")<
  AuthService,
  {
    readonly authenticateToken: (token: string) => Effect.Effect<User, AuthError>
  }
>() {}

// Token manager that coordinates authentication across the app
class TokenManager extends Context.Tag("TokenManager")<
  TokenManager,
  {
    readonly authenticate: (token: string) => Effect.Effect<User, AuthError>
    readonly getCurrentUser: () => Effect.Effect<User, AuthError>
  }
>() {}

const makeTokenManager = Effect.gen(function* () {
  const authService = yield* AuthService
  const currentAuthRef = yield* Ref.make<Deferred.Deferred<User, AuthError> | null>(null)
  
  const authenticate = (token: string) => Effect.gen(function* () {
    // Check if authentication is already in progress
    const existingAuth = yield* Ref.get(currentAuthRef)
    if (existingAuth !== null) {
      // Another fiber is already authenticating, wait for it
      return yield* Deferred.await(existingAuth)
    }
    
    // Start new authentication
    const authDeferred = yield* Deferred.make<User, AuthError>()
    yield* Ref.set(currentAuthRef, authDeferred)
    
    // Perform authentication in background
    yield* Effect.fork(
      authService.authenticateToken(token).pipe(
        Effect.flatMap(user => Deferred.succeed(authDeferred, user)),
        Effect.catchAll(error => Deferred.fail(authDeferred, error)),
        Effect.ensuring(Ref.set(currentAuthRef, null)) // Clean up on completion
      )
    )
    
    return yield* Deferred.await(authDeferred)
  })
  
  const getCurrentUser = () => Effect.gen(function* () {
    const currentAuth = yield* Ref.get(currentAuthRef)
    if (currentAuth === null) {
      return yield* Effect.fail(new AuthError("No authentication in progress"))
    }
    return yield* Deferred.await(currentAuth)
  })
  
  return { authenticate, getCurrentUser } as const
})

// Usage in application
const applicationExample = Effect.gen(function* () {
  const tokenManager = yield* TokenManager
  
  // Multiple concurrent operations that need authentication
  const operations = [
    Effect.gen(function* () {
      const user = yield* tokenManager.authenticate("user-token-123")
      return `Operation 1 completed for user: ${user.email}`
    }),
    Effect.gen(function* () {
      const user = yield* tokenManager.getCurrentUser()
      return `Operation 2 completed for user: ${user.email}`
    }),
    Effect.gen(function* () {
      const user = yield* tokenManager.getCurrentUser()
      return `Operation 3 completed for user: ${user.email}`
    })
  ]
  
  // All operations will coordinate through the same authentication
  return yield* Effect.all(operations, { concurrency: "unbounded" })
})

// Layer implementation
const TokenManagerLive = Layer.effect(
  TokenManager,
  makeTokenManager
)

const AuthServiceLive = Layer.succeed(
  AuthService,
  {
    authenticateToken: (token: string) =>
      Effect.gen(function* () {
        // Simulate authentication delay
        yield* Effect.sleep("100 millis")
        if (token === "user-token-123") {
          return {
            id: "user-123",
            email: "user@example.com",
            permissions: ["read", "write"]
          }
        }
        return yield* Effect.fail(new AuthError("Invalid token"))
      })
  }
)

const MainLayer = Layer.empty.pipe(
  Layer.provide(AuthServiceLive),
  Layer.provide(TokenManagerLive)
)
```

### Example 2: File Processing Pipeline

Coordinate file processing operations where multiple workers need to wait for file availability:

```typescript
import { Deferred, Effect, Queue, Ref, Array as Arr } from "effect"

interface FileData {
  readonly id: string
  readonly content: string
  readonly metadata: Record<string, unknown>
}

class ProcessingError extends Error {
  readonly _tag = "ProcessingError"
}

// File processor that coordinates multiple processing stages
const createFileProcessor = Effect.gen(function* () {
  const processingQueue = yield* Queue.bounded<{
    readonly fileId: string
    readonly deferred: Deferred.Deferred<FileData, ProcessingError>
  }>(100)
  
  const activeJobs = yield* Ref.make<Map<string, Deferred.Deferred<FileData, ProcessingError>>>(new Map())
  
  // Background worker that processes files
  const worker = Effect.gen(function* () {
    while (true) {
      const job = yield* Queue.take(processingQueue)
      
      // Simulate file processing
      const processedFile = yield* Effect.gen(function* () {
        yield* Effect.sleep("500 millis")
        
        // Simulate processing logic
        if (job.fileId.includes("error")) {
          return yield* Effect.fail(new ProcessingError(`Failed to process ${job.fileId}`))
        }
        
        return {
          id: job.fileId,
          content: `Processed content for ${job.fileId}`,
          metadata: { processedAt: Date.now() }
        }
      }).pipe(
        Effect.catchAll(error => 
          Deferred.fail(job.deferred, error as ProcessingError).pipe(
            Effect.as(undefined)
          )
        ),
        Effect.flatMap(result => 
          result ? Deferred.succeed(job.deferred, result) : Effect.void
        )
      )
      
      // Clean up completed job
      yield* Ref.update(activeJobs, map => {
        const newMap = new Map(map)
        newMap.delete(job.fileId)
        return newMap
      })
    }
  }).pipe(
    Effect.fork,
    Effect.scoped
  )
  
  const processFile = (fileId: string) => Effect.gen(function* () {
    const jobs = yield* Ref.get(activeJobs)
    
    // Check if file is already being processed
    const existingJob = jobs.get(fileId)
    if (existingJob) {
      return yield* Deferred.await(existingJob)
    }
    
    // Create new processing job
    const deferred = yield* Deferred.make<FileData, ProcessingError>()
    
    yield* Ref.update(activeJobs, map => 
      new Map(map).set(fileId, deferred)
    )
    
    yield* Queue.offer(processingQueue, { fileId, deferred })
    
    return yield* Deferred.await(deferred)
  })
  
  const getProcessingStatus = (fileId: string) => Effect.gen(function* () {
    const jobs = yield* Ref.get(activeJobs)
    const job = jobs.get(fileId)
    
    if (!job) {
      return { status: "not_found" as const }
    }
    
    const isComplete = yield* Deferred.isDone(job)
    return { 
      status: isComplete ? ("complete" as const) : ("processing" as const) 
    }
  })
  
  return { processFile, getProcessingStatus, worker } as const
})

// Usage example
const fileProcessingExample = Effect.gen(function* () {
  const processor = yield* createFileProcessor
  
  // Start the background worker
  yield* processor.worker
  
  // Process multiple files concurrently
  const fileIds = ["file1.txt", "file2.txt", "file3.txt", "error-file.txt"]
  
  const results = yield* Effect.all(
    Arr.map(fileIds, fileId =>
      processor.processFile(fileId).pipe(
        Effect.either // Convert failures to Either for easier handling
      )
    ),
    { concurrency: "unbounded" }
  )
  
  return results.map((result, index) => ({
    fileId: fileIds[index],
    result: result._tag === "Right" ? result.right : result.left
  }))
}).pipe(
  Effect.scoped // Ensure proper cleanup of background worker
)
```

### Example 3: Real-Time Data Synchronization

Coordinate real-time updates across multiple subscribers using WebSocket connections:

```typescript
import { Deferred, Effect, Ref, Stream, Chunk, Fiber } from "effect"

interface DataUpdate {
  readonly id: string
  readonly timestamp: number
  readonly payload: Record<string, unknown>
}

class ConnectionError extends Error {
  readonly _tag = "ConnectionError"
}

// WebSocket manager that coordinates real-time updates
const createWebSocketManager = (url: string) => Effect.gen(function* () {
  const connectionStatus = yield* Ref.make<"disconnected" | "connecting" | "connected">("disconnected")
  const connectionDeferred = yield* Ref.make<Deferred.Deferred<WebSocket, ConnectionError> | null>(null)
  const subscribers = yield* Ref.make<Set<(update: DataUpdate) => Effect.Effect<void>>>(new Set())
  
  const connect = Effect.gen(function* () {
    const status = yield* Ref.get(connectionStatus)
    
    if (status === "connected") {
      const existingConnection = yield* Ref.get(connectionDeferred)
      if (existingConnection) {
        return yield* Deferred.await(existingConnection)
      }
    }
    
    if (status === "connecting") {
      const pendingConnection = yield* Ref.get(connectionDeferred)
      if (pendingConnection) {
        return yield* Deferred.await(pendingConnection)
      }
    }
    
    // Start new connection
    yield* Ref.set(connectionStatus, "connecting")
    const newDeferred = yield* Deferred.make<WebSocket, ConnectionError>()
    yield* Ref.set(connectionDeferred, newDeferred)
    
    // Simulate WebSocket connection
    yield* Effect.fork(
      Effect.gen(function* () {
        yield* Effect.sleep("1 second") // Simulate connection delay
        
        // Create mock WebSocket
        const mockSocket = {
          send: (data: string) => console.log(`Sending: ${data}`),
          close: () => console.log("Connection closed"),
          addEventListener: (event: string, handler: Function) => {
            if (event === "message") {
              // Simulate periodic updates
              const interval = setInterval(() => {
                handler({
                  data: JSON.stringify({
                    id: Math.random().toString(36),
                    timestamp: Date.now(),
                    payload: { value: Math.random() }
                  })
                })
              }, 2000)
              
              setTimeout(() => clearInterval(interval), 30000) // Stop after 30 seconds
            }
          }
        } as any
        
        yield* Ref.set(connectionStatus, "connected")
        yield* Deferred.succeed(newDeferred, mockSocket)
        
        // Start message handling
        yield* handleIncomingMessages(mockSocket)
      }).pipe(
        Effect.catchAll(error => 
          Deferred.fail(newDeferred, new ConnectionError(error.message))
        ),
        Effect.ensuring(
          Effect.gen(function* () {
            yield* Ref.set(connectionStatus, "disconnected")
            yield* Ref.set(connectionDeferred, null)
          })
        )
      )
    )
    
    return yield* Deferred.await(newDeferred)
  })
  
  const handleIncomingMessages = (socket: WebSocket) => Effect.gen(function* () {
    socket.addEventListener("message", (event: MessageEvent) => {
      Effect.gen(function* () {
        const update: DataUpdate = JSON.parse(event.data)
        const currentSubscribers = yield* Ref.get(subscribers)
        
        // Notify all subscribers
        yield* Effect.forEach(
          Array.from(currentSubscribers),
          (subscriber) => subscriber(update),
          { concurrency: "unbounded" }
        )
      }).pipe(
        Effect.catchAll(error => {
          console.error("Error handling message:", error)
          return Effect.void
        }),
        Effect.runFork
      )
    })
  })
  
  const subscribe = (handler: (update: DataUpdate) => Effect.Effect<void>) => Effect.gen(function* () {
    yield* Ref.update(subscribers, set => new Set(set).add(handler))
    
    // Ensure connection is established
    yield* connect()
    
    // Return cleanup function
    return () => Ref.update(subscribers, set => {
      const newSet = new Set(set)
      newSet.delete(handler)
      return newSet
    })
  })
  
  return { connect, subscribe } as const
})

// Usage example
const webSocketExample = Effect.gen(function* () {
  const wsManager = yield* createWebSocketManager("wss://api.example.com/realtime")
  
  // Create multiple subscribers
  const cleanup1 = yield* wsManager.subscribe((update) =>
    Effect.gen(function* () {
      console.log(`Subscriber 1 received:`, update)
    })
  )
  
  const cleanup2 = yield* wsManager.subscribe((update) =>
    Effect.gen(function* () {
      console.log(`Subscriber 2 received:`, update)
      // Process update in database
      yield* Effect.sleep("100 millis") // Simulate DB operation
    })
  )
  
  // Let it run for a while
  yield* Effect.sleep("10 seconds")
  
  // Clean up subscribers
  yield* cleanup1
  yield* cleanup2
  
  return "WebSocket example completed"
})
```

## Advanced Features Deep Dive

### Feature 1: Completion Strategies

Deferred offers multiple ways to complete a deferred value, each with different use cases and performance characteristics.

#### Basic Completion with `succeed` and `fail`

```typescript
import { Deferred, Effect } from "effect"

const basicCompletionExample = Effect.gen(function* () {
  const successDeferred = yield* Deferred.make<string>()
  const errorDeferred = yield* Deferred.make<string, Error>()
  
  // Complete with success
  const wasSuccessful = yield* Deferred.succeed(successDeferred, "completed!")
  console.log(`Success completion: ${wasSuccessful}`) // true
  
  // Complete with failure
  const wasFailureSet = yield* Deferred.fail(errorDeferred, new Error("something failed"))
  console.log(`Failure completion: ${wasFailureSet}`) // true
  
  // Attempting to complete again returns false
  const attemptAgain = yield* Deferred.succeed(successDeferred, "won't work")
  console.log(`Second completion attempt: ${attemptAgain}`) // false
})
```

#### Effect-based Completion with `complete` and `completeWith`

```typescript
import { Deferred, Effect, Ref } from "effect"

const effectCompletionExample = Effect.gen(function* () {
  const counter = yield* Ref.make(0)
  const memoizedDeferred = yield* Deferred.make<number>()
  const nonMemoizedDeferred = yield* Deferred.make<number>()
  
  // complete() memoizes the effect result
  yield* Deferred.complete(
    memoizedDeferred,
    Ref.updateAndGet(counter, n => n + 1)
  )
  
  // completeWith() re-executes the effect for each await
  yield* Deferred.completeWith(
    nonMemoizedDeferred,
    Ref.updateAndGet(counter, n => n + 1)
  )
  
  // Both deferreds await the same effect, but with different behavior
  const memoized1 = yield* Deferred.await(memoizedDeferred)
  const memoized2 = yield* Deferred.await(memoizedDeferred)
  
  const nonMemoized1 = yield* Deferred.await(nonMemoizedDeferred)
  const nonMemoized2 = yield* Deferred.await(nonMemoizedDeferred)
  
  console.log(`Memoized: ${memoized1}, ${memoized2}`) // Same value
  console.log(`Non-memoized: ${nonMemoized1}, ${nonMemoized2}`) // Different values
})
```

### Feature 2: Polling and Status Checking

Monitor deferred completion status without blocking the current fiber.

#### Checking Completion Status

```typescript
import { Deferred, Effect, Option } from "effect"

const statusCheckingExample = Effect.gen(function* () {
  const deferred = yield* Deferred.make<string>()
  
  // Check if deferred is completed
  const isInitiallyDone = yield* Deferred.isDone(deferred)
  console.log(`Initially done: ${isInitiallyDone}`) // false
  
  // Poll for result without blocking
  const initialPoll = yield* Deferred.poll(deferred)
  console.log(`Initial poll result: ${Option.isNone(initialPoll) ? "None" : "Some"}`) // None
  
  // Complete the deferred
  yield* Deferred.succeed(deferred, "completed")
  
  // Check status again
  const isDoneAfterCompletion = yield* Deferred.isDone(deferred)
  console.log(`Done after completion: ${isDoneAfterCompletion}`) // true
  
  // Poll again - now returns Some
  const pollAfterCompletion = yield* Deferred.poll(deferred)
  if (Option.isSome(pollAfterCompletion)) {
    const result = yield* pollAfterCompletion.value
    console.log(`Poll result: ${result}`) // "completed"
  }
})
```

#### Real-World Polling: Health Check Monitor

```typescript
import { Deferred, Effect, Schedule, Option } from "effect"

interface HealthStatus {
  readonly service: string
  readonly status: "healthy" | "unhealthy"
  readonly lastCheck: number
}

const createHealthMonitor = (serviceName: string) => Effect.gen(function* () {
  const healthDeferred = yield* Deferred.make<HealthStatus, Error>()
  
  // Start health checking in background
  const healthChecker = Effect.gen(function* () {
    const checkHealth = Effect.gen(function* () {
      // Simulate health check
      yield* Effect.sleep("100 millis")
      const isHealthy = Math.random() > 0.3 // 70% chance of being healthy
      
      const status: HealthStatus = {
        service: serviceName,
        status: isHealthy ? "healthy" : "unhealthy",
        lastCheck: Date.now()
      }
      
      if (isHealthy) {
        yield* Deferred.succeed(healthDeferred, status)
      } else {
        yield* Deferred.fail(healthDeferred, new Error(`${serviceName} is unhealthy`))
      }
    })
    
    yield* checkHealth.pipe(
      Effect.retry(Schedule.exponential("1 second").pipe(Schedule.upTo("30 seconds"))),
      Effect.catchAll(() => Effect.void) // Keep trying even after failures
    )
  })
  
  const waitForHealth = (timeout: Duration.Duration = Duration.seconds(10)) => Effect.gen(function* () {
    const healthCheck = Deferred.await(healthDeferred).pipe(
      Effect.timeoutFail(() => new Error(`Health check timeout for ${serviceName}`), timeout)
    )
    
    return yield* healthCheck
  })
  
  const getCurrentStatus = Effect.gen(function* () {
    const poll = yield* Deferred.poll(healthDeferred)
    
    if (Option.isNone(poll)) {
      return { status: "checking" as const }
    }
    
    const result = yield* poll.value.pipe(Effect.either)
    return result._tag === "Right" 
      ? { status: "complete" as const, health: result.right }
      : { status: "failed" as const, error: result.left }
  })
  
  return { waitForHealth, getCurrentStatus, healthChecker } as const
})
```

### Feature 3: Interruption and Cleanup

Handle interruption scenarios and ensure proper resource cleanup.

#### Interruption Handling

```typescript
import { Deferred, Effect, Fiber } from "effect"

const interruptionExample = Effect.gen(function* () {
  const deferred = yield* Deferred.make<string>()
  
  // Start a fiber that waits for the deferred
  const waitingFiber = yield* Effect.fork(
    Deferred.await(deferred).pipe(
      Effect.onInterrupt(() => 
        Effect.gen(function* () {
          console.log("Waiting fiber was interrupted")
        })
      )
    )
  )
  
  // Let it wait a bit
  yield* Effect.sleep("100 millis")
  
  // Interrupt the deferred - this will interrupt all waiting fibers
  const wasInterrupted = yield* Deferred.interrupt(deferred)
  console.log(`Deferred was interrupted: ${wasInterrupted}`)
  
  // The waiting fiber should be interrupted
  const fiberResult = yield* Fiber.await(waitingFiber)
  console.log(`Fiber result:`, fiberResult)
})
```

## Practical Patterns & Best Practices

### Pattern 1: Resource Initialization

Create a pattern for lazy resource initialization that multiple consumers can safely access:

```typescript
import { Deferred, Effect, Ref, Layer, Context } from "effect"

// Generic resource initializer
const createResourceInitializer = <R, E, A>(
  resourceName: string,
  initializeResource: Effect.Effect<A, E, R>
) => Effect.gen(function* () {
  const initializationState = yield* Ref.make<
    | { _tag: "NotStarted" }
    | { _tag: "Initializing"; deferred: Deferred.Deferred<A, E> }
    | { _tag: "Initialized"; resource: A }
  >({ _tag: "NotStarted" })
  
  const getResource = Effect.gen(function* () {
    const currentState = yield* Ref.get(initializationState)
    
    switch (currentState._tag) {
      case "Initialized":
        return currentState.resource
        
      case "Initializing":
        return yield* Deferred.await(currentState.deferred)
        
      case "NotStarted":
        const deferred = yield* Deferred.make<A, E>()
        
        const wasSet = yield* Ref.compareAndSet(
          initializationState,
          currentState,
          { _tag: "Initializing", deferred }
        )
        
        if (!wasSet) {
          // Another fiber started initialization, wait for it
          const newState = yield* Ref.get(initializationState)
          if (newState._tag === "Initializing") {
            return yield* Deferred.await(newState.deferred)
          } else if (newState._tag === "Initialized") {
            return newState.resource
          } else {
            // Retry the whole process
            return yield* getResource
          }
        }
        
        // We won the race, initialize the resource
        yield* Effect.fork(
          initializeResource.pipe(
            Effect.flatMap(resource => 
              Ref.set(initializationState, { _tag: "Initialized", resource }).pipe(
                Effect.flatMap(() => Deferred.succeed(deferred, resource))
              )
            ),
            Effect.catchAll(error =>
              Ref.set(initializationState, { _tag: "NotStarted" }).pipe(
                Effect.flatMap(() => Deferred.fail(deferred, error))
              )
            )
          )
        )
        
        return yield* Deferred.await(deferred)
    }
  })
  
  return { getResource } as const
})

// Example usage with database connection
interface DatabaseConnection {
  readonly host: string
  readonly connectionId: string
  readonly query: (sql: string) => Effect.Effect<unknown[]>
}

class DatabaseConnectionError extends Error {
  readonly _tag = "DatabaseConnectionError"
}

const makeDatabaseService = Effect.gen(function* () {
  const initializer = yield* createResourceInitializer(
    "DatabaseConnection",
    Effect.gen(function* () {
      console.log("Initializing database connection...")
      yield* Effect.sleep("2 seconds") // Simulate connection time
      
      const connection: DatabaseConnection = {
        host: "localhost:5432",
        connectionId: Math.random().toString(36),
        query: (sql: string) => Effect.gen(function* () {
          yield* Effect.sleep("100 millis") // Simulate query time
          return [{ result: `Query result for: ${sql}` }]
        })
      }
      
      console.log(`Database connected with ID: ${connection.connectionId}`)
      return connection
    }).pipe(
      Effect.mapError(() => new DatabaseConnectionError("Failed to connect to database"))
    )
  )
  
  return { getConnection: initializer.getResource } as const
})

// Usage example
const databaseExample = Effect.gen(function* () {
  const dbService = yield* makeDatabaseService
  
  // Multiple concurrent operations that need the database
  const operations = Array.from({ length: 5 }, (_, i) =>
    Effect.gen(function* () {
      const connection = yield* dbService.getConnection()
      const results = yield* connection.query(`SELECT * FROM users WHERE id = ${i}`)
      return { operation: i, results }
    })
  )
  
  // All operations will share the same database connection
  return yield* Effect.all(operations, { concurrency: "unbounded" })
})
```

### Pattern 2: Circuit Breaker with Deferred

Implement a circuit breaker that coordinates failure states across multiple calls:

```typescript
import { Deferred, Effect, Ref, Schedule, Clock } from "effect"

type CircuitState = 
  | { _tag: "Closed" }
  | { _tag: "Open"; openedAt: number; recoveryDeferred: Deferred.Deferred<void> }
  | { _tag: "HalfOpen"; testDeferred: Deferred.Deferred<boolean> }

const createCircuitBreaker = <R, E, A>(
  operation: Effect.Effect<A, E, R>,
  options: {
    readonly failureThreshold: number
    readonly recoveryTimeout: Duration.Duration
    readonly resetSuccessThreshold: number
  }
) => Effect.gen(function* () {
  const state = yield* Ref.make<CircuitState>({ _tag: "Closed" })
  const failureCount = yield* Ref.make(0)
  const successCount = yield* Ref.make(0)
  
  const executeWithCircuitBreaker = Effect.gen(function* () {
    const currentState = yield* Ref.get(state)
    
    switch (currentState._tag) {
      case "Closed":
        return yield* executeOperation()
        
      case "Open":
        // Check if recovery timeout has passed
        const now = yield* Clock.currentTimeMillis
        const timeOpen = now - currentState.openedAt
        
        if (timeOpen >= Duration.toMillis(options.recoveryTimeout)) {
          // Transition to half-open
          const testDeferred = yield* Deferred.make<boolean>()
          const newState: CircuitState = { _tag: "HalfOpen", testDeferred }
          
          const wasSet = yield* Ref.compareAndSet(state, currentState, newState)
          if (wasSet) {
            // We won the race to test, execute the operation
            return yield* executeTestOperation(testDeferred)
          } else {
            // Another fiber is testing, wait for the result
            const updatedState = yield* Ref.get(state)
            if (updatedState._tag === "HalfOpen") {
              const testResult = yield* Deferred.await(updatedState.testDeferred)
              if (testResult) {
                // Test succeeded, circuit is now closed
                return yield* executeOperation()
              } else {
                // Test failed, circuit is still open
                return yield* Effect.fail(new Error("Circuit breaker is open"))
              }
            } else {
              // State changed, retry
              return yield* executeWithCircuitBreaker
            }
          }
        } else {
          // Still in recovery timeout, wait for recovery
          return yield* Deferred.await(currentState.recoveryDeferred).pipe(
            Effect.flatMap(() => executeWithCircuitBreaker)
          )
        }
        
      case "HalfOpen":
        // Wait for the test to complete
        const testResult = yield* Deferred.await(currentState.testDeferred)
        if (testResult) {
          return yield* executeOperation()
        } else {
          return yield* Effect.fail(new Error("Circuit breaker is open"))
        }
    }
  })
  
  const executeOperation = () => operation.pipe(
    Effect.tap(() => Ref.set(failureCount, 0)), // Reset failure count on success
    Effect.catchAll(error => 
      Ref.updateAndGet(failureCount, n => n + 1).pipe(
        Effect.flatMap(failures => {
          if (failures >= options.failureThreshold) {
            return openCircuit().pipe(
              Effect.flatMap(() => Effect.fail(error))
            )
          }
          return Effect.fail(error)
        })
      )
    )
  )
  
  const executeTestOperation = (testDeferred: Deferred.Deferred<boolean>) => 
    operation.pipe(
      Effect.flatMap(result => 
        Ref.updateAndGet(successCount, n => n + 1).pipe(
          Effect.flatMap(successes => {
            if (successes >= options.resetSuccessThreshold) {
              // Close circuit
              return Ref.set(state, { _tag: "Closed" }).pipe(
                Effect.flatMap(() => Ref.set(successCount, 0)),
                Effect.flatMap(() => Deferred.succeed(testDeferred, true)),
                Effect.as(result)
              )
            } else {
              // Need more successes
              return Deferred.succeed(testDeferred, true).pipe(
                Effect.as(result)
              )
            }
          })
        )
      ),
      Effect.catchAll(error => 
        openCircuit().pipe(
          Effect.flatMap(() => Deferred.succeed(testDeferred, false)),
          Effect.flatMap(() => Effect.fail(error))
        )
      )
    )
  
  const openCircuit = Effect.gen(function* () {
    const now = yield* Clock.currentTimeMillis
    const recoveryDeferred = yield* Deferred.make<void>()
    
    yield* Ref.set(state, { _tag: "Open", openedAt: now, recoveryDeferred })
    yield* Ref.set(failureCount, 0)
    yield* Ref.set(successCount, 0)
    
    // Schedule recovery
    yield* Effect.fork(
      Effect.sleep(options.recoveryTimeout).pipe(
        Effect.flatMap(() => Deferred.succeed(recoveryDeferred, undefined))
      )
    )
  })
  
  const getState = Effect.gen(function* () {
    const currentState = yield* Ref.get(state)
    const failures = yield* Ref.get(failureCount)
    
    return {
      state: currentState._tag,
      failureCount: failures,
      ...(currentState._tag === "Open" && {
        openedAt: currentState.openedAt
      })
    }
  })
  
  return { execute: executeWithCircuitBreaker, getState } as const
})
```

## Integration Examples

### Integration with Stream Processing

Coordinate stream operations using Deferred for backpressure and synchronization:

```typescript
import { Deferred, Effect, Stream, Queue, Ref, Chunk } from "effect"

// Stream processor that coordinates multiple stages with backpressure
const createStreamProcessor = <A, B, E>(
  transform: (item: A) => Effect.Effect<B, E>,
  options: {
    readonly bufferSize: number
    readonly maxConcurrency: number
  }
) => Effect.gen(function* () {
  const inputQueue = yield* Queue.bounded<A>(options.bufferSize)
  const outputQueue = yield* Queue.bounded<B>(options.bufferSize)
  const processingSlots = yield* Ref.make(options.maxConcurrency)
  const completionDeferred = yield* Deferred.make<void>()
  
  // Worker pool that processes items
  const workers = Array.from({ length: options.maxConcurrency }, () =>
    Effect.gen(function* () {
      while (true) {
        const item = yield* Queue.take(inputQueue)
        
        const result = yield* transform(item).pipe(
          Effect.catchAll(error => {
            console.error("Processing error:", error)
            return Effect.fail(error)
          })
        )
        
        yield* Queue.offer(outputQueue, result)
      }
    }).pipe(
      Effect.forever,
      Effect.fork
    )
  )
  
  const processStream = (input: Stream.Stream<A>) => 
    Stream.fromEffect(
      Effect.gen(function* () {
        // Start workers
        yield* Effect.all(workers)
        
        // Feed input stream to queue
        yield* Effect.fork(
          Stream.runForEach(input, item => Queue.offer(inputQueue, item)).pipe(
            Effect.ensuring(Queue.shutdown(inputQueue))
          )
        )
        
        // Return output stream
        return Stream.fromQueue(outputQueue).pipe(
          Stream.takeUntil(() => Queue.isShutdown(inputQueue))
        )
      })
    ).pipe(
      Stream.flatten
    )
  
  return { processStream } as const
})

// Example usage with real-time data processing
const streamProcessingExample = Effect.gen(function* () {
  const processor = yield* createStreamProcessor(
    (data: { id: string; value: number }) => Effect.gen(function* () {
      // Simulate processing delay
      yield* Effect.sleep("50 millis")
      
      return {
        id: data.id,
        processedValue: data.value * 2,
        processedAt: Date.now()
      }
    }),
    {
      bufferSize: 100,
      maxConcurrency: 5
    }
  )
  
  // Create input stream
  const inputStream = Stream.range(1, 20).pipe(
    Stream.map(i => ({ id: `item-${i}`, value: i })),
    Stream.schedule(Schedule.spaced("10 millis")) // Emit every 10ms
  )
  
  // Process and collect results
  const results = yield* processor.processStream(inputStream).pipe(
    Stream.runCollect
  )
  
  return Chunk.toReadonlyArray(results)
})
```

### Integration with Testing

Use Deferred for coordinating test scenarios and asserting concurrent behavior:

```typescript
import { Deferred, Effect, TestClock, Clock, Ref, Fiber } from "effect"

// Test utilities for concurrent behavior
const createTestCoordinator = Effect.gen(function* () {
  const events = yield* Ref.make<Array<{ event: string; timestamp: number }>>([])
  const barriers = yield* Ref.make<Map<string, Deferred.Deferred<void>>>(new Map())
  
  const recordEvent = (event: string) => Effect.gen(function* () {
    const timestamp = yield* Clock.currentTimeMillis
    yield* Ref.update(events, events => [...events, { event, timestamp }])
  })
  
  const waitForBarrier = (name: string) => Effect.gen(function* () {
    const currentBarriers = yield* Ref.get(barriers)
    const existingBarrier = currentBarriers.get(name)
    
    if (existingBarrier) {
      return yield* Deferred.await(existingBarrier)
    }
    
    const newBarrier = yield* Deferred.make<void>()
    yield* Ref.update(barriers, map => new Map(map).set(name, newBarrier))
    
    return yield* Deferred.await(newBarrier)
  })
  
  const releaseBarrier = (name: string) => Effect.gen(function* () {
    const currentBarriers = yield* Ref.get(barriers)
    const barrier = currentBarriers.get(name)
    
    if (barrier) {
      yield* Deferred.succeed(barrier, undefined)
      yield* Ref.update(barriers, map => {
        const newMap = new Map(map)
        newMap.delete(name)
        return newMap
      })
    }
  })
  
  const getEvents = Ref.get(events)
  
  return { recordEvent, waitForBarrier, releaseBarrier, getEvents } as const
})

// Test example
const concurrentBehaviorTest = Effect.gen(function* () {
  const coordinator = yield* createTestCoordinator
  
  // Simulate concurrent operations that need coordination
  const operation1 = Effect.gen(function* () {
    yield* coordinator.recordEvent("op1-start")
    yield* Effect.sleep("100 millis")
    yield* coordinator.recordEvent("op1-barrier")
    yield* coordinator.waitForBarrier("sync-point")
    yield* coordinator.recordEvent("op1-after-barrier")
    yield* Effect.sleep("50 millis")
    yield* coordinator.recordEvent("op1-end")
  })
  
  const operation2 = Effect.gen(function* () {
    yield* coordinator.recordEvent("op2-start")
    yield* Effect.sleep("150 millis")
    yield* coordinator.recordEvent("op2-barrier")
    yield* coordinator.waitForBarrier("sync-point")
    yield* coordinator.recordEvent("op2-after-barrier")
    yield* Effect.sleep("75 millis")
    yield* coordinator.recordEvent("op2-end")
  })
  
  // Start both operations
  const fiber1 = yield* Effect.fork(operation1)
  const fiber2 = yield* Effect.fork(operation2)
  
  // Let them run until they hit the barrier
  yield* Effect.sleep("200 millis")
  
  // Release the barrier
  yield* coordinator.releaseBarrier("sync-point")
  
  // Wait for completion
  yield* Fiber.join(fiber1)
  yield* Fiber.join(fiber2)
  
  // Verify the event sequence
  const events = yield* coordinator.getEvents
  
  return events.map(e => e.event) // Return just event names for assertion
}).pipe(
  Effect.provide(TestClock.defaultTestClock)
)
```

## Conclusion

Deferred provides robust fiber-safe coordination, type-safe asynchronous workflows, and structured concurrency management for Effect applications.

Key benefits:
- **Single Assignment Semantics**: Prevents race conditions and ensures consistent state across multiple waiters
- **Type-Safe Error Handling**: Compile-time guarantees about success and failure types with full Effect error model support
- **Structured Concurrency**: Automatic cleanup and proper resource management through Effect's structured concurrency model

Use Deferred when you need to coordinate one-time events across multiple fibers, implement synchronization primitives, or build higher-level concurrent abstractions that require reliable state management and type safety.