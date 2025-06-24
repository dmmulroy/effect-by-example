# TestLive: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem TestLive Solves

Testing applications often requires a delicate balance between controlled test environments and realistic behavior. Traditional testing approaches either mock everything (losing real-world behavior) or use live services everywhere (creating unpredictable, slow tests):

```typescript
// Traditional approach - All mocked (unrealistic)
test('file processing', async () => {
  const mockFileSystem = {
    readFile: jest.fn().mockResolvedValue('mock content'),
    writeFile: jest.fn().mockResolvedValue(undefined)
  };
  
  // Test passes but doesn't validate real I/O behavior
  await processFile(mockFileSystem, 'test.txt');
  expect(mockFileSystem.readFile).toHaveBeenCalled();
});

// Traditional approach - All live (unpredictable)
test('timeout handling', async () => {
  // This test is flaky because it depends on real time
  const start = Date.now();
  try {
    await operationWithTimeout(100);
  } catch (error) {
    const elapsed = Date.now() - start;
    expect(elapsed).toBeGreaterThan(100); // Flaky!
  }
});

// Traditional approach - Mixed environment (confusing)
test('database with real clock', async () => {
  const mockDb = createMockDatabase();
  // Using real Date.now() but mocked database
  // Results in inconsistent test behavior
  const result = await processWithTimestamp(mockDb);
  expect(result.timestamp).toBeCloseTo(Date.now(), -2);
});
```

This approach leads to:
- **Unrealistic Testing** - Mocked services don't behave like real implementations
- **Flaky Tests** - Live services introduce timing and environmental dependencies  
- **Inconsistent Behavior** - Mixed mock/live environments create unpredictable results
- **Poor Performance** - Tests that need real I/O or network calls are slow

### The TestLive Solution

TestLive provides controlled access to real Effect services when you need them, while keeping your test environment predictable and fast:

```typescript
import { Effect, TestServices, Console, Clock } from "effect"

// TestLive lets you selectively use real services
const testWithRealConsole = Effect.gen(function* () {
  // This runs with test services (fast, predictable)
  const testTime = yield* Clock.currentTimeMillis
  
  // But we can access real console when needed
  yield* TestServices.provideLive(
    Console.log(`Test started at: ${testTime}`)
  )
  
  // Back to test environment
  yield* Clock.sleep("1 second") // Instant in tests
  const endTime = yield* Clock.currentTimeMillis
  
  return { testTime, endTime }
})
```

### Key Concepts

**TestLive Service**: A service that provides access to real Effect default services (Clock, Console, Random, etc.) from within test environments.

**Live Context Switching**: The ability to temporarily switch from test services to live services for specific operations while maintaining test isolation.

**Selective Realism**: Using real services only where necessary (like console output, file I/O) while keeping time, randomness, and other services controlled in tests.

## Basic Usage Patterns

### Pattern 1: Accessing Live Services

```typescript
import { Effect, TestServices, Console } from "effect"

// Access the live service for real console output
const debugTest = Effect.gen(function* () {
  const result = yield* someComplexOperation()
  
  // Print to real console for debugging
  yield* TestServices.provideLive(
    Console.log(`Debug: operation result = ${JSON.stringify(result)}`)
  )
  
  return result
})
```

### Pattern 2: Mixed Test and Live Operations

```typescript
import { Effect, TestServices, Clock, Random } from "effect"

const mixedEnvironmentTest = Effect.gen(function* () {
  // Use test clock (instant, controlled)
  yield* Clock.sleep("5 minutes")
  const testTime = yield* Clock.currentTimeMillis
  
  // But use real random for actual randomness
  const realRandom = yield* TestServices.provideLive(
    Random.nextInt
  )
  
  return { testTime, realRandom }
})
```

### Pattern 3: Creating Custom Live Contexts

```typescript
import { Effect, TestServices, Layer, Context } from "effect"

class Logger extends Context.Tag("Logger")<Logger, {
  log: (message: string) => Effect.Effect<void>
}>() {}

const LiveLogger = Layer.succeed(Logger, {
  log: (message) => TestServices.provideLive(
    Console.log(`[LIVE] ${message}`)
  )
})

const testWithLiveLogging = Effect.gen(function* () {
  const logger = yield* Logger
  yield* logger.log("This goes to real console")
}).pipe(
  Effect.provide(LiveLogger)
)
```

## Real-World Examples

### Example 1: File Processing with Real I/O but Controlled Time

```typescript
import { Effect, TestServices, Clock, Console } from "effect"
import * as NodeFileSystem from "@effect/platform-node/FileSystem"

interface FileProcessor {
  processFile: (path: string) => Effect.Effect<ProcessingResult, FileError>
}

const FileProcessor = Context.Tag<FileProcessor>()

interface ProcessingResult {
  path: string
  size: number
  processedAt: number
  content: string
}

interface FileError {
  readonly _tag: "FileNotFound" | "ProcessingError"
  readonly path: string
  readonly cause?: unknown
}

const makeFileProcessor = Effect.gen(function* () {
  return {
    processFile: (path: string) => Effect.gen(function* () {
      // Use real file system for actual file I/O
      const content = yield* TestServices.provideLive(
        NodeFileSystem.FileSystem.pipe(
          Effect.flatMap(fs => fs.readFileString(path)),
          Effect.mapError((cause): FileError => ({
            _tag: "FileNotFound",
            path,
            cause
          }))
        )
      )
      
      // Use test clock for predictable timestamps
      const processedAt = yield* Clock.currentTimeMillis
      
      // Real console output for monitoring
      yield* TestServices.provideLive(
        Console.log(`Processed file: ${path} (${content.length} chars)`)
      )
      
      return {
        path,
        size: content.length,
        processedAt,
        content: content.toUpperCase()
      }
    })
  }
})

// Test with real files but controlled time
const testFileProcessor = Effect.gen(function* () {
  const processor = yield* FileProcessor
  
  // Create a test file (in real filesystem)
  yield* TestServices.provideLive(
    NodeFileSystem.FileSystem.pipe(
      Effect.flatMap(fs => fs.writeFileString("test.txt", "hello world"))
    )
  )
  
  const result = yield* processor.processFile("test.txt")
  
  // Time is controlled by test clock
  expect(result.processedAt).toBe(0) // Test clock starts at 0
  expect(result.content).toBe("HELLO WORLD")
  expect(result.size).toBe(11)
  
  // Cleanup
  yield* TestServices.provideLive(
    NodeFileSystem.FileSystem.pipe(
      Effect.flatMap(fs => fs.remove("test.txt"))
    )
  )
}).pipe(
  Effect.provide(Layer.succeed(FileProcessor, makeFileProcessor))
)
```

### Example 2: API Testing with Real HTTP but Controlled Timing

```typescript
import { Effect, TestServices, Clock, Schedule, Console } from "effect"
import * as Http from "@effect/platform/HttpClient"

interface ApiClient {
  fetchWithRetry: <A>(
    url: string, 
    decoder: (response: unknown) => A
  ) => Effect.Effect<A, ApiError>
}

const ApiClient = Context.Tag<ApiClient>()

interface ApiError {
  readonly _tag: "NetworkError" | "DecodingError" | "TimeoutError"
  readonly url: string
  readonly cause?: unknown
}

const makeApiClient = Effect.gen(function* () {
  return {
    fetchWithRetry: <A>(url: string, decoder: (response: unknown) => A) =>
      Effect.gen(function* () {
        // Real HTTP request
        const response = yield* TestServices.provideLive(
          Http.request.get(url).pipe(
            Http.client.execute,
            Effect.flatMap(Http.response.json),
            Effect.mapError((cause): ApiError => ({
              _tag: "NetworkError",
              url,
              cause
            }))
          )
        )
        
        // Controlled retry timing
        const retrySchedule = Schedule.exponential("100 millis").pipe(
          Schedule.intersect(Schedule.recurs(3))
        )
        
        const decoded = yield* Effect.try({
          try: () => decoder(response),
          catch: (cause): ApiError => ({
            _tag: "DecodingError", 
            url,
            cause
          })
        }).pipe(
          Effect.retry(retrySchedule),
          Effect.timeoutFail({
            duration: "30 seconds",
            onTimeout: (): ApiError => ({ _tag: "TimeoutError", url })
          })
        )
        
        return decoded
      })
  }
})

// Test with real API calls but controlled timing
const testApiWithRetry = Effect.gen(function* () {
  const api = yield* ApiClient
  
  // Log real request timing
  yield* TestServices.provideLive(
    Console.log("Starting API test with real HTTP requests")
  )
  
  const startTime = yield* Clock.currentTimeMillis
  
  const result = yield* api.fetchWithRetry(
    "https://jsonplaceholder.typicode.com/posts/1",
    (response: any) => ({
      id: response.id,
      title: response.title
    })
  )
  
  const endTime = yield* Clock.currentTimeMillis
  
  // Time measurements are predictable in tests
  expect(endTime - startTime).toBe(0) // Instantaneous in test environment
  expect(result.id).toBe(1)
  expect(typeof result.title).toBe("string")
  
  yield* TestServices.provideLive(
    Console.log(`API test completed: ${JSON.stringify(result)}`)
  )
}).pipe(
  Effect.provide(Layer.succeed(ApiClient, makeApiClient))
)
```

### Example 3: Database Testing with Live Connections but Test Transactions

```typescript
import { Effect, TestServices, Clock, Console } from "effect"
import * as SqlClient from "@effect/sql/SqlClient"

interface UserRepository {
  createUser: (user: NewUser) => Effect.Effect<User, RepositoryError>
  findUser: (id: string) => Effect.Effect<User | null, RepositoryError>
}

const UserRepository = Context.Tag<UserRepository>()

interface NewUser {
  name: string
  email: string
}

interface User extends NewUser {
  id: string
  createdAt: number
}

interface RepositoryError {
  readonly _tag: "DatabaseError" | "UserNotFound"
  readonly cause?: unknown
}

const makeUserRepository = Effect.gen(function* () {
  const sql = yield* SqlClient.SqlClient
  
  return {
    createUser: (user: NewUser) => Effect.gen(function* () {
      const id = yield* Effect.sync(() => crypto.randomUUID())
      
      // Use test clock for predictable timestamps
      const createdAt = yield* Clock.currentTimeMillis
      
      // Real database operation
      yield* TestServices.provideLive(
        sql.execute("INSERT INTO users (id, name, email, created_at) VALUES (?, ?, ?, ?)")
          (id, user.name, user.email, createdAt)
      ).pipe(
        Effect.mapError((cause): RepositoryError => ({
          _tag: "DatabaseError",
          cause
        }))
      )
      
      // Log to real console for debugging
      yield* TestServices.provideLive(
        Console.log(`Created user: ${user.email} at ${createdAt}`)
      )
      
      return { ...user, id, createdAt }
    }),
    
    findUser: (id: string) => Effect.gen(function* () {
      const rows = yield* TestServices.provideLive(
        sql.execute("SELECT * FROM users WHERE id = ?")
          (id)
      ).pipe(
        Effect.mapError((cause): RepositoryError => ({
          _tag: "DatabaseError",
          cause
        }))
      )
      
      if (rows.length === 0) return null
      
      const row = rows[0]
      return {
        id: row.id,
        name: row.name,
        email: row.email,
        createdAt: row.created_at
      }
    })
  }
})

// Test with real database but controlled time and isolated transactions
const testUserRepository = Effect.gen(function* () {
  const repo = yield* UserRepository
  
  // All database operations happen in a real transaction
  // but timing is controlled by test clock
  yield* SqlClient.SqlClient.pipe(
    Effect.flatMap(sql => TestServices.provideLive(sql.begin)),
    Effect.flatMap(() => Effect.gen(function* () {
      const newUser = { name: "Test User", email: "test@example.com" }
      
      const created = yield* repo.createUser(newUser)
      expect(created.createdAt).toBe(0) // Test clock time
      
      const found = yield* repo.findUser(created.id)
      expect(found).toEqual(created)
      
      // Advance test time
      yield* Clock.sleep("1 hour")
      
      const secondUser = { name: "User 2", email: "user2@example.com" }
      const created2 = yield* repo.createUser(secondUser)
      
      // Second user has later timestamp
      expect(created2.createdAt).toBe(3600000) // 1 hour in milliseconds
      
      yield* TestServices.provideLive(
        Console.log("Database test completed successfully")
      )
    })),
    Effect.flatMap(() => 
      SqlClient.SqlClient.pipe(
        Effect.flatMap(sql => TestServices.provideLive(sql.rollback))
      )
    )
  )
}).pipe(
  Effect.provide(UserRepository.layer)
)
```

## Advanced Features Deep Dive

### Feature 1: Custom Live Service Creation

TestLive allows you to create custom services that selectively use live implementations while maintaining test isolation.

#### Basic Custom Live Service

```typescript
import { Effect, TestServices, Context, Layer } from "effect"

class EmailService extends Context.Tag("EmailService")<EmailService, {
  send: (to: string, subject: string, body: string) => Effect.Effect<void, EmailError>
  validate: (email: string) => Effect.Effect<boolean>
}>() {}

interface EmailError {
  readonly _tag: "SendError" | "ValidationError"
  readonly email: string
  readonly cause?: unknown
}

// Service that uses live console for email "sending" but test services for validation
const LiveEmailService = Layer.succeed(EmailService, {
  send: (to, subject, body) => Effect.gen(function* () {
    // Validate using test services (instant)
    const isValid = yield* EmailService.pipe(
      Effect.flatMap(service => service.validate(to))
    )
    
    if (!isValid) {
      return yield* Effect.fail({
        _tag: "ValidationError" as const,
        email: to
      })
    }
    
    // "Send" email using real console output
    yield* TestServices.provideLive(
      Console.log(`📧 Email sent to: ${to}`)
        .pipe(Effect.delay("100 millis")) // Real delay for demonstration
    )
  }),
  
  validate: (email) => Effect.succeed(email.includes("@"))
})
```

#### Advanced Custom Live Service with Resource Management

```typescript
import { Effect, TestServices, Queue, Ref, Schedule } from "effect"

class NotificationService extends Context.Tag("NotificationService")<
  NotificationService,
  {
    queue: (notification: Notification) => Effect.Effect<void>
    process: () => Effect.Effect<void, never, never>
    getStats: () => Effect.Effect<NotificationStats>
  }
>() {}

interface Notification {
  type: "email" | "sms" | "push"
  recipient: string
  message: string
  priority: "low" | "medium" | "high"
}

interface NotificationStats {
  processed: number
  failed: number
  queued: number
}

const makeNotificationService = Effect.gen(function* () {
  const queue = yield* Queue.bounded<Notification>(100)
  const processed = yield* Ref.make(0)
  const failed = yield* Ref.make(0)
  
  return {
    queue: (notification: Notification) => Queue.offer(queue, notification),
    
    process: () => Effect.gen(function* () {
      const notification = yield* Queue.take(queue)
      
      // Real console output for monitoring
      yield* TestServices.provideLive(
        Console.log(`Processing ${notification.type} to ${notification.recipient}`)
      )
      
      const success = yield* Effect.gen(function* () {
        // Simulate processing with test services (controlled timing)
        const delay = notification.priority === "high" ? "100 millis" : "500 millis"
        yield* Clock.sleep(delay)
        
        // Real random for realistic failure simulation
        const shouldFail = yield* TestServices.provideLive(
          Random.nextBoolean.pipe(Effect.map(b => b && notification.priority === "low"))
        )
        
        return !shouldFail
      })
      
      if (success) {
        yield* Ref.update(processed, n => n + 1)
        yield* TestServices.provideLive(
          Console.log(`✅ Successfully processed ${notification.type}`)
        )
      } else {
        yield* Ref.update(failed, n => n + 1)
        yield* TestServices.provideLive(
          Console.log(`❌ Failed to process ${notification.type}`)
        )
      }
    }).pipe(
      Effect.forever,
      Effect.forkDaemon
    ),
    
    getStats: () => Effect.gen(function* () {
      const processedCount = yield* Ref.get(processed)
      const failedCount = yield* Ref.get(failed)
      const queuedCount = yield* Queue.size(queue)
      
      return {
        processed: processedCount,
        failed: failedCount,
        queued: queuedCount
      }
    })
  }
})

const NotificationServiceLive = Layer.effect(NotificationService, makeNotificationService)
```

### Feature 2: Scoped Live Services

Control the lifetime of live services using Effect's scoped resource management.

#### Scoped File Operations

```typescript
import { Effect, TestServices, Scope } from "effect"

const withTempDirectory = <A, E, R>(
  effect: (tempDir: string) => Effect.Effect<A, E, R>
): Effect.Effect<A, E | Error, R | Scope.Scope> =>
  Effect.gen(function* () {
    // Create temp directory using live file system
    const tempDir = yield* TestServices.provideLive(
      Effect.sync(() => {
        const path = `/tmp/test-${Date.now()}-${Math.random()}`
        require('fs').mkdirSync(path, { recursive: true })
        return path
      })
    )
    
    // Add cleanup to scope
    yield* Effect.addFinalizer(() =>
      TestServices.provideLive(
        Effect.sync(() => {
          require('fs').rmSync(tempDir, { recursive: true, force: true })
        }).pipe(
          Effect.catchAll(() => Effect.void), // Ignore cleanup errors
          Effect.tap(() => Console.log(`Cleaned up temp directory: ${tempDir}`))
        )
      )
    )
    
    // Log directory creation
    yield* TestServices.provideLive(
      Console.log(`Created temp directory: ${tempDir}`)
    )
    
    return yield* effect(tempDir)
  })

// Usage with automatic cleanup
const testWithTempFiles = Effect.scoped(
  withTempDirectory((tempDir) => Effect.gen(function* () {
    // Write test files using live I/O
    yield* TestServices.provideLive(
      Effect.sync(() => {
        require('fs').writeFileSync(`${tempDir}/test1.txt`, "content 1")
        require('fs').writeFileSync(`${tempDir}/test2.txt`, "content 2")
      })
    )
    
    // Test clock operations (instant)
    const startTime = yield* Clock.currentTimeMillis
    yield* Clock.sleep("5 minutes")
    const endTime = yield* Clock.currentTimeMillis
    
    expect(endTime - startTime).toBe(300000) // 5 minutes in test time
    
    // Verify files exist using live I/O
    const files = yield* TestServices.provideLive(
      Effect.sync(() => require('fs').readdirSync(tempDir))
    )
    
    expect(files).toEqual(["test1.txt", "test2.txt"])
  }))
)
```

#### Scoped Network Resources

```typescript
import { Effect, TestServices, Scope, Deferred } from "effect"

const withTestServer = <A, E, R>(
  port: number,
  effect: (baseUrl: string) => Effect.Effect<A, E, R>
): Effect.Effect<A, E | Error, R | Scope.Scope> =>
  Effect.gen(function* () {
    const serverReady = yield* Deferred.make<void>()
    
    // Start real HTTP server
    const server = yield* TestServices.provideLive(
      Effect.async<any, Error>((resume) => {
        const express = require('express')
        const app = express()
        
        app.get('/health', (req: any, res: any) => res.json({ status: 'ok' }))
        app.get('/time', (req: any, res: any) => res.json({ time: Date.now() }))
        
        const server = app.listen(port, () => {
          Deferred.succeed(serverReady, undefined)
          resume(Effect.succeed(server))
        })
        
        server.on('error', (error: Error) => {
          resume(Effect.fail(error))
        })
      })
    )
    
    // Add server cleanup to scope
    yield* Effect.addFinalizer(() =>
      TestServices.provideLive(
        Effect.sync(() => server.close()).pipe(
          Effect.tap(() => Console.log(`Stopped test server on port ${port}`))
        )
      )
    )
    
    // Wait for server to be ready
    yield* Deferred.await(serverReady)
    
    yield* TestServices.provideLive(
      Console.log(`Test server started on port ${port}`)
    )
    
    return yield* effect(`http://localhost:${port}`)
  })

// Test with real server but controlled timing
const testWithServer = Effect.scoped(
  withTestServer(3001, (baseUrl) => Effect.gen(function* () {
    // Make real HTTP requests
    const healthCheck = yield* TestServices.provideLive(
      Http.request.get(`${baseUrl}/health`).pipe(
        Http.client.execute,
        Effect.flatMap(Http.response.json)
      )
    )
    
    expect(healthCheck).toEqual({ status: 'ok' })
    
    // Controlled timing for test duration
    const testStart = yield* Clock.currentTimeMillis
    yield* Clock.sleep("10 seconds")
    
    const timeResponse = yield* TestServices.provideLive(
      Http.request.get(`${baseUrl}/time`).pipe(
        Http.client.execute,
        Effect.flatMap(Http.response.json)
      )
    )
    
    // Server returns real time, but test tracks controlled time
    const testEnd = yield* Clock.currentTimeMillis
    expect(testEnd - testStart).toBe(10000) // Test time
    expect(timeResponse.time).toBeGreaterThan(Date.now() - 1000) // Real time
  }))
)
```

## Practical Patterns & Best Practices

### Pattern 1: Selective Service Replacement

Create helper functions to selectively replace specific services with live implementations:

```typescript
import { Effect, TestServices, Context, Layer } from "effect"

// Helper to run with live console but test clock
const withLiveConsole = <A, E, R>(
  effect: Effect.Effect<A, E, R>
): Effect.Effect<A, E, R> =>
  Effect.gen(function* () {
    const result = yield* effect
    // Any console operations within the effect use live console
    return result
  }).pipe(
    Effect.provideSomeLayer(
      Layer.succeed(Console.Console, {
        ...Console.defaultConsole,
        log: (...args) => TestServices.provideLive(Console.log(...args))
      })
    )
  )

// Helper to run with live random but test everything else
const withLiveRandom = <A, E, R>(
  effect: Effect.Effect<A, E, R>
): Effect.Effect<A, E, R> =>
  effect.pipe(
    Effect.provideSomeLayer(
      Layer.effect(Random.Random, 
        TestServices.provideLive(Effect.sync(() => Random.defaultRandom))
      )
    )
  )

// Combine multiple live services
const withLiveConsoleAndRandom = <A, E, R>(
  effect: Effect.Effect<A, E, R>
): Effect.Effect<A, E, R> =>
  effect.pipe(
    withLiveConsole,
    withLiveRandom
  )
```

### Pattern 2: Test Environment Configuration

Create configurable test environments that can easily switch between mocked and live services:

```typescript
interface TestEnvironment {
  readonly useLiveConsole: boolean
  readonly useLiveRandom: boolean
  readonly useLiveFileSystem: boolean
  readonly useLiveNetwork: boolean
}

const createTestEnvironment = (config: TestEnvironment) =>
  Layer.mergeAll(
    // Base test services
    TestServices.liveServices,
    
    // Conditional live services
    config.useLiveConsole 
      ? Layer.succeed(Console.Console, Console.defaultConsole) 
      : Layer.empty,
      
    config.useLiveRandom
      ? Layer.effect(Random.Random, TestServices.provideLive(Effect.succeed(Random.defaultRandom)))
      : Layer.empty,
      
    config.useLiveFileSystem
      ? NodeFileSystem.layer
      : Layer.empty,
      
    config.useLiveNetwork
      ? Http.client.layer
      : Layer.empty
  )

// Different test environments for different scenarios
const debugEnvironment = createTestEnvironment({
  useLiveConsole: true,
  useLiveRandom: false,
  useLiveFileSystem: false,
  useLiveNetwork: false
})

const integrationEnvironment = createTestEnvironment({
  useLiveConsole: true,
  useLiveRandom: true,
  useLiveFileSystem: true,
  useLiveNetwork: true
})

const unitTestEnvironment = createTestEnvironment({
  useLiveConsole: false,
  useLiveRandom: false,
  useLiveFileSystem: false,
  useLiveNetwork: false
})
```

### Pattern 3: Debugging Helpers

Use TestLive to create debugging utilities that provide insight into test execution:

```typescript
const withTestDebugger = <A, E, R>(
  testName: string,
  effect: Effect.Effect<A, E, R>
): Effect.Effect<A, E, R> =>
  Effect.gen(function* () {
    const startTime = yield* Clock.currentTimeMillis
    const realStartTime = Date.now()
    
    yield* TestServices.provideLive(
      Console.log(`🧪 Starting test: ${testName}`)
    )
    
    const result = yield* effect.pipe(
      Effect.catchAll(error => 
        TestServices.provideLive(
          Console.log(`❌ Test failed: ${testName} - ${JSON.stringify(error)}`)
        ).pipe(
          Effect.flatMap(() => Effect.fail(error))
        )
      )
    )
    
    const endTime = yield* Clock.currentTimeMillis
    const realEndTime = Date.now()
    
    yield* TestServices.provideLive(
      Console.log(
        `✅ Test completed: ${testName} ` +
        `(test time: ${endTime - startTime}ms, real time: ${realEndTime - realStartTime}ms)`
      )
    )
    
    return result
  })

// Enhanced debugging with resource tracking
const withResourceTracking = <A, E, R>(
  effect: Effect.Effect<A, E, R>
): Effect.Effect<A, E, R> =>
  Effect.gen(function* () {
    const initialMemory = yield* TestServices.provideLive(
      Effect.sync(() => process.memoryUsage())
    )
    
    const result = yield* effect
    
    const finalMemory = yield* TestServices.provideLive(
      Effect.sync(() => process.memoryUsage())
    )
    
    const memoryDiff = {
      heapUsed: finalMemory.heapUsed - initialMemory.heapUsed,
      heapTotal: finalMemory.heapTotal - initialMemory.heapTotal,
      external: finalMemory.external - initialMemory.external
    }
    
    yield* TestServices.provideLive(
      Console.log(`📊 Memory usage: ${JSON.stringify(memoryDiff)}`)
    )
    
    return result
  })
```

### Pattern 4: Cross-Environment Testing

Test the same logic across different service implementations:

```typescript
const testAcrossEnvironments = <A>(
  testName: string,
  testLogic: Effect.Effect<A>,
  assertions: (result: A) => void
) =>
  Effect.gen(function* () {
    yield* TestServices.provideLive(
      Console.log(`Running ${testName} across different environments`)
    )
    
    // Test with full test services
    const testResult = yield* testLogic
    assertions(testResult)
    
    // Test with live random
    const liveRandomResult = yield* testLogic.pipe(withLiveRandom)
    assertions(liveRandomResult)
    
    // Test with live console (for debugging)
    const liveConsoleResult = yield* testLogic.pipe(withLiveConsole)
    assertions(liveConsoleResult)
    
    yield* TestServices.provideLive(
      Console.log(`✅ ${testName} passed in all environments`)
    )
  })
```

## Integration Examples

### Integration with Vitest

TestLive integrates seamlessly with Effect's Vitest integration for real-world testing scenarios:

```typescript
import { describe, it, expect } from "@effect/vitest"
import { Effect, TestServices, Console, Clock } from "effect"

describe("TestLive Integration", () => {
  it.effect("should handle mixed test and live services", () =>
    Effect.gen(function* () {
      // Test services for timing
      const start = yield* Clock.currentTimeMillis
      yield* Clock.sleep("1 hour")
      const end = yield* Clock.currentTimeMillis
      
      expect(end - start).toBe(3600000)
      
      // Live services for real output
      yield* TestServices.provideLive(
        Console.log("This appears in console during test")
      )
    })
  )
  
  it.live("should use all live services", () =>
    Effect.gen(function* () {
      // This test runs with live services by default
      const start = Date.now()
      yield* Effect.sleep("10 millis") // Real sleep
      const end = Date.now()
      
      expect(end - start).toBeGreaterThan(8)
      yield* Console.log("Real console output")
    })
  )
  
  it.scoped("should manage resources with TestLive", () =>
    Effect.gen(function* () {
      const resource = yield* Effect.acquireRelease(
        TestServices.provideLive(
          Effect.sync(() => {
            console.log("Acquiring real resource")
            return { id: "resource-123" }
          })
        ),
        (resource) => TestServices.provideLive(
          Effect.sync(() => {
            console.log(`Releasing ${resource.id}`)
          })
        )
      )
      
      expect(resource.id).toBe("resource-123")
    })
  )
})
```

### Integration with Property-Based Testing

Combine TestLive with Effect's property-based testing capabilities:

```typescript
import { Effect, TestServices, FastCheck, Console } from "effect"

const testStringProcessing = FastCheck.property(
  FastCheck.string(),
  (input) => Effect.gen(function* () {
    // Log real test inputs for debugging
    yield* TestServices.provideLive(
      Console.log(`Testing with input: "${input}"`)
    )
    
    const processed = yield* processString(input)
    
    // Property: processed string should be same length
    expect(processed.length).toBe(input.length)
    
    // Property: should be idempotent
    const processedTwice = yield* processString(processed)
    expect(processedTwice).toBe(processed)
  })
)

const runPropertyTest = Effect.gen(function* () {
  yield* TestServices.provideLive(
    Console.log("Starting property-based test with live logging")
  )
  
  yield* testStringProcessing
  
  yield* TestServices.provideLive(
    Console.log("Property test completed successfully")
  )
})
```

### Integration with Test Containers

Use TestLive with test containers for integration testing:

```typescript
import { Effect, TestServices, Layer, Console } from "effect"

const withPostgresContainer = <A, E, R>(
  effect: (connectionString: string) => Effect.Effect<A, E, R>
): Effect.Effect<A, E | Error, R | Scope.Scope> =>
  Effect.gen(function* () {
    // Start real PostgreSQL container
    const container = yield* TestServices.provideLive(
      Effect.async<any, Error>((resume) => {
        const { GenericContainer } = require("testcontainers")
        
        new GenericContainer("postgres:15")
          .withEnvironment({
            POSTGRES_DB: "testdb",
            POSTGRES_USER: "testuser", 
            POSTGRES_PASSWORD: "testpass"
          })
          .withExposedPorts(5432)
          .start()
          .then((container: any) => {
            resume(Effect.succeed(container))
          })
          .catch((error: Error) => {
            resume(Effect.fail(error))
          })
      })
    )
    
    // Add container cleanup
    yield* Effect.addFinalizer(() =>
      TestServices.provideLive(
        Effect.promise(() => container.stop()).pipe(
          Effect.tap(() => Console.log("Stopped PostgreSQL container"))
        )
      )
    )
    
    const connectionString = `postgresql://testuser:testpass@${container.getHost()}:${container.getMappedPort(5432)}/testdb`
    
    yield* TestServices.provideLive(
      Console.log(`PostgreSQL container ready: ${connectionString}`)
    )
    
    return yield* effect(connectionString)
  })

// Test with real database container but controlled test time
const testDatabaseOperations = Effect.scoped(
  withPostgresContainer((connectionString) => Effect.gen(function* () {
    // Setup database with real connection but test timing
    const sql = yield* SqlClient.make({ 
      connectionString,
      transformQueryNames: SqlClient.transform.camelToSnake,
      transformResultNames: SqlClient.transform.snakeToCamel
    })
    
    // Create schema
    yield* TestServices.provideLive(
      sql.execute(`
        CREATE TABLE users (
          id UUID PRIMARY KEY,
          name TEXT NOT NULL,
          created_at BIGINT NOT NULL
        )
      `)
    )
    
    // Test with controlled time
    const startTime = yield* Clock.currentTimeMillis
    
    yield* sql.execute("INSERT INTO users VALUES (?, ?, ?)")
      (crypto.randomUUID(), "Test User", startTime)
    
    yield* Clock.sleep("1 day")
    const endTime = yield* Clock.currentTimeMillis
    
    const users = yield* sql.execute("SELECT * FROM users")
    
    expect(users).toHaveLength(1)
    expect(users[0].createdAt).toBe(startTime)
    expect(endTime - startTime).toBe(86400000) // 1 day in test time
  })).pipe(
    Effect.provide(SqlClient.layer)
  )
)
```

## Conclusion

TestLive provides the perfect balance between realistic testing and controlled test environments. It enables you to use real Effect services exactly when needed while maintaining the predictability and speed that makes tests reliable.

Key benefits:
- **Selective Realism**: Use live services only where necessary for meaningful tests
- **Controlled Environment**: Maintain predictable timing and deterministic behavior
- **Debugging Support**: Real console output and logging for test debugging
- **Resource Management**: Proper cleanup of live resources using Effect's scoped system
- **Composability**: Mix and match test and live services based on testing needs

TestLive is essential when you need to test real I/O operations, network requests, or other external integrations while keeping your test suite fast and reliable.