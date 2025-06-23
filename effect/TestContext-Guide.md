# TestContext: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem TestContext Solves

When writing tests, you often need to control and mock various aspects of the runtime environment - time, randomness, service dependencies, and configuration. Traditional approaches lead to fragile, hard-to-maintain test code:

```typescript
// Traditional approach - problematic test setup
describe("UserService", () => {
  let originalConsoleLog: any
  let originalDateNow: any
  let mockDb: any
  
  beforeEach(() => {
    // Manual mocking of global functions
    originalConsoleLog = console.log
    console.log = jest.fn()
    
    // Time mocking with jest
    originalDateNow = Date.now
    Date.now = jest.fn(() => 1234567890)
    
    // Service mocking
    mockDb = {
      users: {
        create: jest.fn(),
        find: jest.fn()
      }
    }
  })
  
  afterEach(() => {
    // Manual cleanup
    console.log = originalConsoleLog
    Date.now = originalDateNow
    jest.clearAllMocks()
  })
  
  test("creates user with timestamp", async () => {
    // Test implementation with manual mocks
  })
})
```

This approach leads to:
- **Global State Pollution** - Tests can interfere with each other through shared mocks
- **Manual Cleanup** - Easy to forget teardown, causing test pollution
- **Type Unsafety** - Mocks often bypass TypeScript's type checking
- **Brittle Tests** - Changes to implementation details break tests

### The TestContext Solution

Effect's TestContext provides a composable, type-safe test environment that automatically manages test services, time control, and configuration:

```typescript
import { Effect, TestContext, TestClock, TestServices } from "effect"
import { it } from "@effect/vitest"

// Clean, type-safe test with automatic service management
it.effect("creates user with timestamp", () =>
  Effect.gen(function* () {
    // TestContext automatically provides test services
    const now = yield* TestClock.currentTimeMillis
    const user = yield* createUser({ name: "Alice" })
    
    expect(user.createdAt).toBe(now)
  })
)
```

### Key Concepts

**TestContext**: A Layer that provides all test-specific services (TestClock, TestRandom, TestConfig, etc.) in a single, composable unit.

**TestServices**: The collection of services provided by TestContext: annotations, live service access, size configuration, and test config.

**Test Isolation**: Each test runs in its own fiber with isolated state, preventing cross-test pollution.

## Basic Usage Patterns

### Pattern 1: Using TestContext with @effect/vitest

```typescript
import { Effect, TestContext } from "effect"
import { it, expect } from "@effect/vitest"

// The it.effect helper automatically provides TestContext
it.effect("basic test with test services", () =>
  Effect.gen(function* () {
    // All test services are available
    const config = yield* Effect.testConfig
    expect(config.repeats).toBe(100)
  })
)
```

### Pattern 2: Accessing Test Services

```typescript
import { Effect, TestServices, TestClock, TestRandom } from "effect"
import { it } from "@effect/vitest"

it.effect("using test services", () =>
  Effect.gen(function* () {
    // Access the test clock for time control
    yield* TestClock.set(new Date("2024-01-01"))
    const time = yield* TestClock.currentTimeMillis
    
    // Access test random for deterministic randomness
    const random = yield* TestRandom.next
    
    // Access test annotations for metadata
    yield* TestServices.annotate("testId", "user-creation-001")
    
    // Access test configuration
    const samples = yield* TestServices.samples
    expect(samples).toBe(200)
  })
)
```

### Pattern 3: Live Service Access

```typescript
import { Effect, TestServices, Console } from "effect"
import { it } from "@effect/vitest"

it.effect("accessing real services in tests", () =>
  Effect.gen(function* () {
    // Run effect with real/live services
    yield* TestServices.provideLive(
      Console.log("This uses the real console")
    )
    
    // Or use provideWithLive for transformations
    const result = yield* TestServices.provideWithLive(
      Effect.succeed(42),
      (effect) => Effect.map(effect, n => n * 2)
    )
    
    expect(result).toBe(84)
  })
)
```

## Real-World Examples

### Example 1: Testing Time-Dependent Business Logic

Testing a user session that expires after 30 minutes of inactivity:

```typescript
import { Effect, TestClock, TestContext, Duration, Option } from "effect"
import { it, expect } from "@effect/vitest"

interface Session {
  userId: string
  lastActivity: number
  data: Record<string, unknown>
}

class SessionService extends Effect.Service<SessionService>()("SessionService", {
  effect: Effect.gen(function* () {
    const sessions = new Map<string, Session>()
    
    const create = (userId: string) => Effect.gen(function* () {
      const now = yield* Effect.clock.currentTimeMillis
      const session: Session = {
        userId,
        lastActivity: now,
        data: {}
      }
      sessions.set(userId, session)
      return session
    })
    
    const get = (userId: string) => Effect.gen(function* () {
      const session = sessions.get(userId)
      if (!session) return Option.none()
      
      const now = yield* Effect.clock.currentTimeMillis
      const elapsed = now - session.lastActivity
      
      // Session expires after 30 minutes
      if (elapsed > Duration.toMillis(Duration.minutes(30))) {
        sessions.delete(userId)
        return Option.none()
      }
      
      return Option.some(session)
    })
    
    const touch = (userId: string) => Effect.gen(function* () {
      const session = sessions.get(userId)
      if (!session) return Option.none()
      
      const now = yield* Effect.clock.currentTimeMillis
      session.lastActivity = now
      return Option.some(session)
    })
    
    return { create, get, touch } as const
  })
}) {}

it.effect("session expires after 30 minutes of inactivity", () =>
  Effect.gen(function* () {
    const service = yield* SessionService
    
    // Create a session at current time
    yield* TestClock.set(new Date("2024-01-01T10:00:00Z"))
    const session = yield* service.create("user-123")
    
    // Session should exist after 29 minutes
    yield* TestClock.adjust(Duration.minutes(29))
    const active = yield* service.get("user-123")
    expect(Option.isSome(active)).toBe(true)
    
    // Session should expire after 31 minutes
    yield* TestClock.adjust(Duration.minutes(2))
    const expired = yield* service.get("user-123")
    expect(Option.isNone(expired)).toBe(true)
  }).pipe(
    Effect.provide(SessionService.Default)
  )
)

it.effect("touching session resets expiry", () =>
  Effect.gen(function* () {
    const service = yield* SessionService
    
    yield* TestClock.set(new Date("2024-01-01T10:00:00Z"))
    yield* service.create("user-123")
    
    // After 29 minutes, touch the session
    yield* TestClock.adjust(Duration.minutes(29))
    yield* service.touch("user-123")
    
    // Wait another 29 minutes - session should still be active
    yield* TestClock.adjust(Duration.minutes(29))
    const active = yield* service.get("user-123")
    expect(Option.isSome(active)).toBe(true)
  }).pipe(
    Effect.provide(SessionService.Default)
  )
)
```

### Example 2: Testing Service Dependencies with Mocks

Testing a notification service that depends on email and SMS services:

```typescript
import { Effect, Context, Layer, Queue, TestServices, Array as Arr } from "effect"
import { it, expect, describe } from "@effect/vitest"

// Service interfaces
interface EmailService {
  send(to: string, subject: string, body: string): Effect.Effect<void>
}

interface SmsService {
  send(to: string, message: string): Effect.Effect<void>
}

interface NotificationService {
  notify(userId: string, message: string): Effect.Effect<void>
}

// Service tags
class EmailService extends Context.Tag("EmailService")<EmailService, EmailService>() {}
class SmsService extends Context.Tag("SmsService")<SmsService, SmsService>() {}
class NotificationService extends Context.Tag("NotificationService")<NotificationService, NotificationService>() {}

// Test implementation with recording
const makeTestEmailService = Effect.gen(function* () {
  const sent = yield* Queue.unbounded<{ to: string; subject: string; body: string }>()
  
  const send = (to: string, subject: string, body: string) =>
    Queue.offer(sent, { to, subject, body })
  
  const getSent = Queue.takeAll(sent)
  
  return { send, getSent } as const
})

const makeTestSmsService = Effect.gen(function* () {
  const sent = yield* Queue.unbounded<{ to: string; message: string }>()
  
  const send = (to: string, message: string) =>
    Queue.offer(sent, { to, message })
  
  const getSent = Queue.takeAll(sent)
  
  return { send, getSent } as const
})

// Real notification service implementation
const NotificationServiceLive = Layer.effect(
  NotificationService,
  Effect.gen(function* () {
    const email = yield* EmailService
    const sms = yield* SmsService
    
    const notify = (userId: string, message: string) =>
      Effect.gen(function* () {
        // Send both email and SMS notifications
        yield* Effect.all([
          email.send(`${userId}@example.com`, "Notification", message),
          sms.send(`+1${userId}`, message)
        ], { concurrency: "unbounded" })
      })
    
    return { notify }
  })
)

describe("NotificationService", () => {
  it.effect("sends both email and SMS notifications", () =>
    Effect.gen(function* () {
      const emailService = yield* makeTestEmailService
      const smsService = yield* makeTestSmsService
      const notifications = yield* NotificationService
      
      // Send a notification
      yield* notifications.notify("555-1234", "Your order has shipped!")
      
      // Verify email was sent
      const emails = yield* emailService.getSent
      expect(emails).toHaveLength(1)
      expect(emails[0]).toEqual({
        to: "555-1234@example.com",
        subject: "Notification",
        body: "Your order has shipped!"
      })
      
      // Verify SMS was sent
      const messages = yield* smsService.getSent
      expect(messages).toHaveLength(1)
      expect(messages[0]).toEqual({
        to: "+1555-1234",
        message: "Your order has shipped!"
      })
    }).pipe(
      Effect.provide(
        Layer.mergeAll(
          Layer.effect(EmailService, makeTestEmailService),
          Layer.effect(SmsService, makeTestSmsService),
          NotificationServiceLive
        )
      )
    )
  )
})
```

### Example 3: Testing with Test Annotations and Metadata

Using test annotations to track test execution and gather metrics:

```typescript
import { Effect, TestServices, TestAnnotation, TestAnnotationMap, Duration, Option } from "effect"
import { it, expect, afterAll } from "@effect/vitest"

// Custom test annotations
const testDuration = TestAnnotation.make<Duration.Duration>(
  "duration",
  Duration.zero,
  Duration.sum
)

const testCategory = TestAnnotation.make<Array<string>>(
  "category",
  [],
  Arr.appendAll
)

const testPriority = TestAnnotation.make<"high" | "medium" | "low">(
  "priority",
  "medium",
  (a, b) => a === "high" || b === "high" ? "high" : a === "low" && b === "low" ? "low" : "medium"
)

// Helper to measure test duration
const timed = <A, E, R>(effect: Effect.Effect<A, E, R>) =>
  Effect.gen(function* () {
    const start = yield* Effect.clock.currentTimeMillis
    const result = yield* effect
    const end = yield* Effect.clock.currentTimeMillis
    
    yield* TestServices.annotate(testDuration, Duration.millis(end - start))
    
    return result
  })

it.effect("high priority database test", () =>
  Effect.gen(function* () {
    yield* TestServices.annotate(testCategory, ["database", "integration"])
    yield* TestServices.annotate(testPriority, "high")
    
    yield* timed(
      Effect.gen(function* () {
        // Simulate database operation
        yield* Effect.sleep(Duration.millis(100))
        
        // Test logic here
        expect(true).toBe(true)
      })
    )
  })
)

it.effect("low priority utility test", () =>
  Effect.gen(function* () {
    yield* TestServices.annotate(testCategory, ["utility"])
    yield* TestServices.annotate(testPriority, "low")
    
    yield* timed(
      Effect.gen(function* () {
        // Quick utility test
        yield* Effect.sleep(Duration.millis(10))
        expect(true).toBe(true)
      })
    )
  })
)

// Collect and report test metrics after all tests
afterAll(() =>
  Effect.gen(function* () {
    const annotations = yield* TestServices.annotations()
    
    // In a real scenario, you might want to aggregate annotations across tests
    // This is a simplified example
    yield* TestServices.provideLive(
      Effect.gen(function* () {
        yield* Effect.log("Test execution summary generated")
      })
    )
  }).pipe(
    Effect.provide(TestContext),
    Effect.runPromise
  )
)
```

## Advanced Features Deep Dive

### Feature 1: Test Configuration Management

TestContext provides sophisticated test configuration through TestConfig:

#### Basic Test Configuration Usage

```typescript
import { Effect, TestServices, TestConfig } from "effect"
import { it, expect } from "@effect/vitest"

it.effect("accessing test configuration", () =>
  Effect.gen(function* () {
    const config = yield* TestServices.testConfig
    
    expect(config.repeats).toBe(100)  // Number of test repetitions
    expect(config.retries).toBe(100)  // Number of retries for flaky tests
    expect(config.samples).toBe(200)  // Samples for property tests
    expect(config.shrinks).toBe(1000) // Max shrinking iterations
  })
)
```

#### Real-World Test Configuration Example

```typescript
import { Effect, TestServices, Layer, TestConfig } from "effect"
import { it, layer } from "@effect/vitest"

// Custom test configuration for different test suites
const performanceTestConfig = TestConfig.make({
  repeats: 1000,    // More repetitions for performance tests
  retries: 10,      // Fewer retries
  samples: 500,     // More samples for statistical significance
  shrinks: 100      // Less shrinking for performance
})

const integrationTestConfig = TestConfig.make({
  repeats: 10,      // Fewer repetitions
  retries: 200,     // More retries for flaky network tests
  samples: 100,     // Standard samples
  shrinks: 1000     // Full shrinking for debugging
})

// Apply configuration to test suites
layer(TestServices.testConfigLayer(performanceTestConfig))("performance tests", (it) => {
  it.effect("load test with many samples", () =>
    Effect.gen(function* () {
      const config = yield* TestServices.testConfig
      const samples = yield* TestServices.samples
      
      // Run performance test with configured samples
      const results = yield* Effect.forEach(
        Arr.range(0, samples),
        () => measureOperationLatency(),
        { concurrency: "unbounded" }
      )
      
      const avgLatency = Arr.reduce(results, 0, (a, b) => a + b) / results.length
      expect(avgLatency).toBeLessThan(100)
    })
  )
})
```

#### Advanced Test Configuration: Dynamic Configuration

```typescript
import { Effect, TestServices, Config, Layer } from "effect"

// Load test configuration from environment
const testConfigFromEnv = Layer.effect(
  TestConfig.TestConfig,
  Effect.gen(function* () {
    const repeats = yield* Config.withDefault(Config.integer("TEST_REPEATS"), 100)
    const retries = yield* Config.withDefault(Config.integer("TEST_RETRIES"), 100)
    const samples = yield* Config.withDefault(Config.integer("TEST_SAMPLES"), 200)
    const shrinks = yield* Config.withDefault(Config.integer("TEST_SHRINKS"), 1000)
    
    return TestConfig.make({ repeats, retries, samples, shrinks })
  })
)
```

### Feature 2: Test Service Isolation with Sized

The TestSized service provides size-based configuration for generative tests:

#### Basic Sized Usage

```typescript
import { Effect, TestServices } from "effect"
import { it, expect } from "@effect/vitest"

it.effect("using test size for data generation", () =>
  Effect.gen(function* () {
    const size = yield* TestServices.size
    
    // Generate data proportional to test size
    const users = yield* generateUsers(size)
    expect(users).toHaveLength(size)
  })
)

const generateUsers = (count: number) =>
  Effect.succeed(
    Arr.range(0, count).map(i => ({
      id: `user-${i}`,
      name: `User ${i}`,
      email: `user${i}@example.com`
    }))
  )
```

#### Real-World Sized Example: Stress Testing

```typescript
import { Effect, TestServices, Duration, Metric } from "effect"
import { it } from "@effect/vitest"

const requestLatency = Metric.histogram(
  "request_latency",
  Metric.Histogram.Boundaries.exponential(1, 2, 10)
)

it.effect("stress test with configurable load", () =>
  Effect.gen(function* () {
    // Use different sizes for different test scenarios
    yield* TestServices.withSize(10)(  // Light load
      stressTest("light load")
    )
    
    yield* TestServices.withSize(100)( // Medium load
      stressTest("medium load")
    )
    
    yield* TestServices.withSize(1000)( // Heavy load
      stressTest("heavy load")
    )
  })
)

const stressTest = (scenario: string) =>
  Effect.gen(function* () {
    const size = yield* TestServices.size
    const service = yield* ApiService
    
    yield* Effect.log(`Running ${scenario} with ${size} concurrent requests`)
    
    const results = yield* Effect.forEach(
      Arr.range(0, size),
      (i) => 
        service.makeRequest(`/api/test/${i}`).pipe(
          Effect.tap((duration) => Metric.update(requestLatency, duration)),
          Effect.catchAll(() => Effect.succeed(Duration.infinity))
        ),
      { concurrency: size }
    )
    
    const successful = results.filter(d => d !== Duration.infinity).length
    const successRate = (successful / size) * 100
    
    yield* Effect.log(`${scenario}: ${successRate}% success rate`)
    expect(successRate).toBeGreaterThan(95)
  })
```

#### Advanced Sized: Property-Based Testing Integration

```typescript
import { Effect, TestServices, Schema } from "effect"
import { it } from "@effect/vitest"

const User = Schema.Struct({
  id: Schema.String,
  name: Schema.String,
  age: Schema.Number.pipe(Schema.between(0, 150)),
  email: Schema.String.pipe(Schema.pattern(/^[^@]+@[^@]+$/))
})

it.effect.prop(
  "user validation handles generated data",
  { user: User },
  ({ user }) =>
    Effect.gen(function* () {
      const size = yield* TestServices.size
      
      // Generate more test cases for larger sizes
      const variations = yield* Effect.forEach(
        Arr.range(0, Math.max(1, size / 10)),
        (i) => ({
          ...user,
          email: `${user.name.toLowerCase().replace(/\s/g, ".")}${i}@example.com`
        })
      )
      
      // All variations should be valid
      for (const variant of variations) {
        const result = Schema.decodeUnknownEither(User)(variant)
        expect(Either.isRight(result)).toBe(true)
      }
    })
)
```

## Practical Patterns & Best Practices

### Pattern 1: Test Service Factory Pattern

Create reusable test service factories for consistent mocking:

```typescript
import { Effect, Context, Layer, Ref, TestServices } from "effect"

// Generic test service factory
const makeTestService = <S, A>(
  tag: Context.Tag<S, A>,
  make: Effect.Effect<A>
) => Layer.scoped(
  tag,
  Effect.gen(function* () {
    const service = yield* make
    const annotations = yield* TestServices.annotations()
    
    // Annotate test with service creation
    yield* annotations.annotate(
      TestAnnotation.make("mocked-services", [] as string[], Arr.appendAll),
      [tag.key]
    )
    
    return service
  })
)

// Reusable mock database service
const makeTestDatabase = () => makeTestService(
  Database,
  Effect.gen(function* () {
    const data = yield* Ref.make(new Map<string, unknown>())
    
    return {
      get: (key: string) => Ref.get(data).pipe(
        Effect.map(map => Option.fromNullable(map.get(key)))
      ),
      set: (key: string, value: unknown) => Ref.update(data, map => {
        map.set(key, value)
        return map
      }),
      delete: (key: string) => Ref.update(data, map => {
        map.delete(key)
        return map
      }),
      clear: () => Ref.set(data, new Map())
    }
  })
)
```

### Pattern 2: Test Environment Builder

Build complex test environments compositionally:

```typescript
import { Effect, Layer, TestContext } from "effect"

// Test environment builder
class TestEnvironment {
  private layers: Layer.Layer<any, any, any>[] = []
  
  withService<S, E>(layer: Layer.Layer<S, E, TestServices.TestServices>) {
    this.layers.push(layer)
    return this
  }
  
  withConfig(config: Partial<TestConfig.TestConfig>) {
    this.layers.push(
      TestServices.testConfigLayer(
        TestConfig.make({
          repeats: config.repeats ?? 100,
          retries: config.retries ?? 100,
          samples: config.samples ?? 200,
          shrinks: config.shrinks ?? 1000
        })
      )
    )
    return this
  }
  
  withSize(size: number) {
    this.layers.push(TestServices.sizedLayer(size))
    return this
  }
  
  build() {
    return this.layers.reduce(
      (acc, layer) => Layer.provideMerge(acc, layer),
      TestContext
    )
  }
}

// Usage
const testEnv = new TestEnvironment()
  .withService(makeTestDatabase())
  .withService(makeTestCache())
  .withConfig({ retries: 50, samples: 500 })
  .withSize(200)
  .build()

layer(testEnv)("integration tests", (it) => {
  it.effect("complex test with full environment", () =>
    Effect.gen(function* () {
      // All configured services are available
      const db = yield* Database
      const cache = yield* Cache
      const size = yield* TestServices.size
      
      expect(size).toBe(200)
    })
  )
})
```

## Integration Examples

### Integration with @effect/vitest

The most common integration is with @effect/vitest for structured testing:

```typescript
import { Effect, Layer, TestContext } from "effect"
import { describe, expect, it, layer } from "@effect/vitest"

// Service definitions
class UserRepository extends Context.Tag("UserRepository")<
  UserRepository,
  {
    create(user: User.New): Effect.Effect<User.Entity, DatabaseError>
    findById(id: string): Effect.Effect<Option.Option<User.Entity>, DatabaseError>
    findByEmail(email: string): Effect.Effect<Option.Option<User.Entity>, DatabaseError>
  }
>() {}

// Test implementation with recording
const TestUserRepository = Layer.effect(
  UserRepository,
  Effect.gen(function* () {
    const users = yield* Ref.make<Map<string, User.Entity>>(new Map())
    const annotations = yield* TestServices.annotations()
    
    return {
      create: (user: User.New) =>
        Effect.gen(function* () {
          const id = yield* Random.nextIntBetween(1000, 9999).pipe(
            Effect.map(String)
          )
          const entity = { ...user, id, createdAt: yield* Clock.currentTimeMillis }
          yield* Ref.update(users, map => new Map(map).set(id, entity))
          
          // Track operations in test annotations
          yield* annotations.annotate(
            TestAnnotation.make("db-operations", [] as string[], Arr.appendAll),
            [`CREATE:${id}`]
          )
          
          return entity
        }),
        
      findById: (id: string) =>
        Effect.gen(function* () {
          const map = yield* Ref.get(users)
          yield* annotations.annotate(
            TestAnnotation.make("db-operations", [] as string[], Arr.appendAll),
            [`FIND:${id}`]
          )
          return Option.fromNullable(map.get(id))
        }),
        
      findByEmail: (email: string) =>
        Effect.gen(function* () {
          const map = yield* Ref.get(users)
          const user = Arr.findFirst(
            Array.from(map.values()),
            u => u.email === email
          )
          yield* annotations.annotate(
            TestAnnotation.make("db-operations", [] as string[], Arr.appendAll),
            [`FIND_BY_EMAIL:${email}`]
          )
          return user
        })
    }
  })
)

describe("UserService Integration Tests", () => {
  // Share test repository across tests in this suite
  layer(TestUserRepository)((it) => {
    it.effect("creates and retrieves users", () =>
      Effect.gen(function* () {
        const repo = yield* UserRepository
        
        // Create a user
        const created = yield* repo.create({
          name: "Alice",
          email: "alice@example.com"
        })
        
        // Retrieve by ID
        const found = yield* repo.findById(created.id)
        expect(Option.isSome(found)).toBe(true)
        expect(Option.getOrNull(found)?.name).toBe("Alice")
        
        // Retrieve by email
        const foundByEmail = yield* repo.findByEmail("alice@example.com")
        expect(Option.isSome(foundByEmail)).toBe(true)
      })
    )
    
    it.effect("tracks database operations", () =>
      Effect.gen(function* () {
        const repo = yield* UserRepository
        
        yield* repo.create({ name: "Bob", email: "bob@example.com" })
        const created = yield* repo.create({ name: "Carol", email: "carol@example.com" })
        yield* repo.findById(created.id)
        yield* repo.findByEmail("bob@example.com")
        
        // Check recorded operations
        const operations = yield* TestServices.get(
          TestAnnotation.make("db-operations", [] as string[], Arr.appendAll)
        )
        
        expect(operations).toHaveLength(4)
        expect(operations.filter(op => op.startsWith("CREATE"))).toHaveLength(2)
        expect(operations.filter(op => op.startsWith("FIND"))).toHaveLength(2)
      })
    )
  })
})
```

### Testing Strategies

#### Strategy 1: Deterministic Testing with TestRandom

```typescript
import { Effect, TestRandom, Random } from "effect"
import { it, expect } from "@effect/vitest"

const shuffleArray = <A>(array: ReadonlyArray<A>): Effect.Effect<Array<A>> =>
  Effect.gen(function* () {
    const result = [...array]
    for (let i = result.length - 1; i > 0; i--) {
      const j = yield* Random.nextIntBetween(0, i + 1)
      ;[result[i], result[j]] = [result[j], result[i]]
    }
    return result
  })

it.effect("shuffle is deterministic with seed", () =>
  Effect.gen(function* () {
    yield* TestRandom.setSeed("test-seed-123")
    
    const input = [1, 2, 3, 4, 5]
    const result1 = yield* shuffleArray(input)
    
    // Reset seed and shuffle again
    yield* TestRandom.setSeed("test-seed-123")
    const result2 = yield* shuffleArray(input)
    
    // Same seed produces same shuffle
    expect(result1).toEqual(result2)
    
    // Different seed produces different shuffle
    yield* TestRandom.setSeed("different-seed")
    const result3 = yield* shuffleArray(input)
    expect(result3).not.toEqual(result1)
  })
)
```

#### Strategy 2: Time Travel Testing with TestClock

```typescript
import { Effect, TestClock, Schedule, Duration, Ref } from "effect"
import { it, expect } from "@effect/vitest"

it.effect("retry with exponential backoff", () =>
  Effect.gen(function* () {
    const attempts = yield* Ref.make<Array<number>>([])
    let attemptCount = 0
    
    const failingOperation = Effect.gen(function* () {
      const now = yield* Clock.currentTimeMillis
      yield* Ref.update(attempts, arr => [...arr, now])
      attemptCount++
      
      if (attemptCount < 4) {
        return yield* Effect.fail("Connection failed")
      }
      return "Success"
    })
    
    const startTime = yield* TestClock.currentTimeMillis
    
    // Run with exponential backoff
    const fiber = yield* failingOperation.pipe(
      Effect.retry(
        Schedule.exponential(Duration.seconds(1), 2).pipe(
          Schedule.compose(Schedule.recurs(5))
        )
      ),
      Effect.fork
    )
    
    // Advance time to trigger retries
    yield* TestClock.adjust(Duration.seconds(1))  // First retry at 1s
    yield* TestClock.adjust(Duration.seconds(2))  // Second retry at 3s
    yield* TestClock.adjust(Duration.seconds(4))  // Third retry at 7s
    
    const result = yield* Fiber.join(fiber)
    const attemptTimes = yield* Ref.get(attempts)
    
    expect(result).toBe("Success")
    expect(attemptTimes).toHaveLength(4)
    
    // Verify exponential backoff timing
    expect(attemptTimes[1] - attemptTimes[0]).toBe(1000)  // 1 second
    expect(attemptTimes[2] - attemptTimes[1]).toBe(2000)  // 2 seconds  
    expect(attemptTimes[3] - attemptTimes[2]).toBe(4000)  // 4 seconds
  })
)
```

## Conclusion

TestContext provides a comprehensive, type-safe foundation for testing Effect applications with proper isolation, deterministic behavior, and powerful service management.

Key benefits:
- **Automatic Isolation**: Each test runs in its own fiber with isolated services, preventing test pollution
- **Type Safety**: Full TypeScript support ensures mocks match service interfaces
- **Composability**: Layer-based architecture allows building complex test environments from simple pieces
- **Determinism**: Control over time, randomness, and configuration ensures reproducible tests

TestContext is essential for building maintainable, reliable test suites in Effect applications.