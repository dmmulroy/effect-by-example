# TestServices: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem TestServices Solves

Testing Effect applications presents unique challenges when dealing with services that interact with the outside world - time, random numbers, file systems, and network calls. Traditional testing approaches often lead to:

```typescript
// Traditional approach - unreliable, slow, and brittle tests
import { test } from 'vitest'

test('user session expires after 1 hour', async () => {
  const user = await createUser()
  const session = await loginUser(user)
  
  // ❌ Actually wait 1 hour - test takes forever!
  await new Promise(resolve => setTimeout(resolve, 3600000))
  
  const isValid = await validateSession(session.token)
  expect(isValid).toBe(false)
})

test('retry failed API calls 3 times', async () => {
  const mockApi = jest.fn()
    .mockRejectedValueOnce(new Error('Network error'))
    .mockRejectedValueOnce(new Error('Network error'))
    .mockRejectedValueOnce(new Error('Network error'))
    .mockResolvedValueOnce({ data: 'success' })
  
  // ❌ Complex mocking setup, brittle assertions
  const result = await retryApiCall(mockApi)
  expect(mockApi).toHaveBeenCalledTimes(4)
})
```

This approach leads to:
- **Slow Tests** - Waiting for real time to pass or network calls to complete
- **Flaky Tests** - Tests that pass sometimes and fail other times due to timing
- **Complex Mocking** - Intricate setup required to mock external dependencies
- **Poor Isolation** - Tests that interfere with each other through shared state

### The TestServices Solution

TestServices provides a clean, composable way to control and mock the runtime environment of your Effect applications:

```typescript
import { TestServices, TestClock, Effect } from "effect"

// ✅ Fast, deterministic, and reliable
const testUserSessionExpiry = Effect.gen(function* () {
  const user = yield* createUser()
  const session = yield* loginUser(user)
  
  // Instantly advance time by 1 hour
  yield* TestClock.adjust(Duration.hours(1))
  
  const isValid = yield* validateSession(session.token)
  expect(isValid).toBe(false)
}).pipe(
  // Provide test environment with controlled time
  Effect.provide(TestServices.liveServices)
)
```

### Key Concepts

**TestAnnotations**: Track test metadata, timing, and custom annotations throughout test execution

**TestLive**: Access to real "live" services when you need actual implementations (like console output)

**TestSized**: Control the size parameter for property-based testing and data generation

**TestConfig**: Configure test behavior like retry counts, sample sizes, and shrinking parameters

## Basic Usage Patterns

### Pattern 1: Basic Test Environment Setup

```typescript
import { Effect, TestServices } from "effect"
import { describe, it } from "@effect/vitest"

// Simplest usage - provide test services to an effect
const myTest = Effect.gen(function* () {
  // Your test logic here
  const result = yield* someEffectThatNeedsTestServices()
  expect(result).toBe("expected")
}).pipe(
  Effect.provide(TestServices.liveServices)
)

describe("My Feature", () => {
  it.effect("should work correctly", () => myTest)
})
```

### Pattern 2: Accessing Test Annotations

```typescript
import { TestServices, TestAnnotation, Effect } from "effect"

const testWithAnnotations = Effect.gen(function* () {
  // Add custom annotations to track test progress
  yield* TestServices.annotate(TestAnnotation.tagged("phase"), "setup")
  
  const user = yield* createUser()
  
  yield* TestServices.annotate(TestAnnotation.tagged("phase"), "execution")
  const result = yield* processUser(user)
  
  yield* TestServices.annotate(TestAnnotation.tagged("phase"), "cleanup")
  yield* cleanupUser(user.id)
  
  // Access annotations
  const phase = yield* TestServices.get(TestAnnotation.tagged("phase"))
  console.log(`Current phase: ${phase}`)
  
  return result
})
```

### Pattern 3: Using Live Services When Needed

```typescript
import { TestServices, Effect, Console } from "effect"

const testWithLiveServices = Effect.gen(function* () {
  // Run most of the test with test services
  const result = yield* simulateUserWorkflow()
  
  // Use live services for actual console output
  yield* TestServices.provideLive(
    Console.log(`Test completed with result: ${result}`)
  )
  
  return result
})
```

## Real-World Examples

### Example 1: Testing Time-Dependent Business Logic

Consider a subscription service where users get trial periods and need to handle expiration:

```typescript
import { 
  Effect, 
  TestServices, 
  TestClock, 
  Duration, 
  Context,
  Layer 
} from "effect"

// Domain model
interface User {
  readonly id: string
  readonly email: string
  readonly subscriptionStatus: "trial" | "active" | "expired"
  readonly trialEndDate: Date
}

interface SubscriptionService {
  readonly createTrialUser: (email: string) => Effect.Effect<User>
  readonly checkSubscriptionStatus: (userId: string) => Effect.Effect<User>
  readonly sendExpirationNotice: (user: User) => Effect.Effect<void>
}

const SubscriptionService = Context.GenericTag<SubscriptionService>("SubscriptionService")

// Implementation
const makeSubscriptionService = Effect.gen(function* () {
  const users = new Map<string, User>()
  
  const createTrialUser = (email: string): Effect.Effect<User> => 
    Effect.gen(function* () {
      const now = yield* Effect.clockWith(clock => clock.currentTimeMillis)
      const user: User = {
        id: `user-${Math.random()}`,
        email,
        subscriptionStatus: "trial",
        trialEndDate: new Date(now + Duration.toMillis(Duration.days(14)))
      }
      users.set(user.id, user)
      return user
    })
  
  const checkSubscriptionStatus = (userId: string): Effect.Effect<User> =>
    Effect.gen(function* () {
      const user = users.get(userId)
      if (!user) {
        return yield* Effect.fail(new Error(`User ${userId} not found`))
      }
      
      const now = yield* Effect.clockWith(clock => clock.currentTimeMillis)
      if (now > user.trialEndDate.getTime() && user.subscriptionStatus === "trial") {
        const expiredUser = { ...user, subscriptionStatus: "expired" as const }
        users.set(userId, expiredUser)
        return expiredUser
      }
      
      return user
    })
  
  const sendExpirationNotice = (user: User): Effect.Effect<void> =>
    TestServices.provideLive(
      Effect.log(`Sending expiration notice to ${user.email}`)
    )
  
  return { createTrialUser, checkSubscriptionStatus, sendExpirationNotice }
})

const SubscriptionServiceLive = Layer.effect(SubscriptionService, makeSubscriptionService)

// Test: Trial expiration workflow
const testTrialExpiration = Effect.gen(function* () {
  const service = yield* SubscriptionService
  
  // Create a trial user
  const user = yield* service.createTrialUser("user@example.com")
  expect(user.subscriptionStatus).toBe("trial")
  
  // Fast-forward 10 days - still in trial
  yield* TestClock.adjust(Duration.days(10))
  const stillTrial = yield* service.checkSubscriptionStatus(user.id)
  expect(stillTrial.subscriptionStatus).toBe("trial")
  
  // Fast-forward 5 more days - trial expired
  yield* TestClock.adjust(Duration.days(5))
  const expired = yield* service.checkSubscriptionStatus(user.id)
  expect(expired.subscriptionStatus).toBe("expired")
  
  // Send notification
  yield* service.sendExpirationNotice(expired)
  
  return expired
}).pipe(
  Effect.provide(SubscriptionServiceLive),
  Effect.provide(TestServices.liveServices)
)
```

### Example 2: Testing Retry Logic and Network Resilience

Testing network failures and retry mechanisms:

```typescript
import { 
  Effect, 
  TestServices, 
  TestClock, 
  Duration, 
  Schedule, 
  Context,
  Layer,
  Random
} from "effect"

// Domain model
interface ApiClient {
  readonly fetchUserData: (userId: string) => Effect.Effect<UserData, ApiError>
}

interface UserData {
  readonly id: string
  readonly name: string
  readonly lastLogin: Date
}

class ApiError {
  readonly _tag = "ApiError"
  constructor(public readonly message: string, public readonly retryable: boolean) {}
}

const ApiClient = Context.GenericTag<ApiClient>("ApiClient")

// Mock implementation that simulates network failures
const makeMockApiClient = Effect.gen(function* () {
  let callCount = 0
  
  const fetchUserData = (userId: string): Effect.Effect<UserData, ApiError> =>
    Effect.gen(function* () {
      callCount++
      
      // Simulate network failures for first 2 calls
      if (callCount <= 2) {
        yield* TestServices.annotate(
          TestAnnotation.tagged("api-call"), 
          `Attempt ${callCount}: Network failure`
        )
        return yield* Effect.fail(
          new ApiError("Network timeout", true)
        )
      }
      
      // Third call succeeds
      yield* TestServices.annotate(
        TestAnnotation.tagged("api-call"), 
        `Attempt ${callCount}: Success`
      )
      
      const now = yield* Effect.clockWith(clock => clock.currentTimeMillis)
      return {
        id: userId,
        name: "John Doe",
        lastLogin: new Date(now)
      }
    })
  
  return { fetchUserData }
})

const MockApiClientLive = Layer.effect(ApiClient, makeMockApiClient)

// Retry service with exponential backoff
const fetchUserDataWithRetry = (userId: string) =>
  Effect.gen(function* () {
    const client = yield* ApiClient
    
    return yield* client.fetchUserData(userId).pipe(
      Effect.retry(
        Schedule.exponential(Duration.seconds(1)).pipe(
          Schedule.intersect(Schedule.recurs(3))
        )
      ),
      Effect.catchAll(error => {
        // After all retries failed, log and re-throw
        return TestServices.provideLive(
          Effect.log(`Failed to fetch user data after retries: ${error.message}`)
        ).pipe(
          Effect.flatMap(() => Effect.fail(error))
        )
      })
    )
  })

// Test: Retry mechanism with controlled time
const testRetryMechanism = Effect.gen(function* () {
  // Track the start time
  const startTime = yield* TestClock.currentTimeMillis
  
  // Attempt to fetch user data (will fail twice, then succeed)
  const userData = yield* fetchUserDataWithRetry("user-123")
  
  expect(userData.id).toBe("user-123")
  expect(userData.name).toBe("John Doe")
  
  // Verify that time advanced due to retries
  const endTime = yield* TestClock.currentTimeMillis
  const elapsedTime = endTime - startTime
  
  // Should have waited: 1s + 2s = 3s total for the two retries
  expect(elapsedTime).toBeGreaterThanOrEqual(Duration.toMillis(Duration.seconds(3)))
  
  // Check annotations
  const apiCallHistory = yield* TestServices.get(TestAnnotation.tagged("api-call"))
  expect(apiCallHistory).toContain("Network failure")
  expect(apiCallHistory).toContain("Success")
  
  return userData
}).pipe(
  Effect.provide(MockApiClientLive),
  Effect.provide(TestServices.liveServices)
)
```

### Example 3: Property-Based Testing with TestSized

Testing data generation and validation with different sizes:

```typescript
import { 
  Effect, 
  TestServices, 
  FastCheck, 
  Array as Arr,
  Context,
  Layer 
} from "effect"

// Domain model
interface Product {
  readonly id: string
  readonly name: string
  readonly price: number
  readonly category: string
}

interface ProductService {
  readonly validateProducts: (products: ReadonlyArray<Product>) => Effect.Effect<ReadonlyArray<Product>, ValidationError>
  readonly calculateTotal: (products: ReadonlyArray<Product>) => Effect.Effect<number>
}

class ValidationError {
  readonly _tag = "ValidationError"
  constructor(public readonly message: string) {}
}

const ProductService = Context.GenericTag<ProductService>("ProductService")

// Implementation
const makeProductService = Effect.gen(function* () {
  const validateProducts = (products: ReadonlyArray<Product>): Effect.Effect<ReadonlyArray<Product>, ValidationError> =>
    Effect.gen(function* () {
      // Validate each product
      for (const product of products) {
        if (product.price < 0) {
          return yield* Effect.fail(new ValidationError(`Invalid price for ${product.name}: ${product.price}`))
        }
        if (product.name.length === 0) {
          return yield* Effect.fail(new ValidationError(`Empty name for product ${product.id}`))
        }
      }
      return products
    })
  
  const calculateTotal = (products: ReadonlyArray<Product>): Effect.Effect<number> =>
    Effect.gen(function* () {
      const validated = yield* validateProducts(products)
      return validated.reduce((sum, product) => sum + product.price, 0)
    })
  
  return { validateProducts, calculateTotal }
})

const ProductServiceLive = Layer.effect(ProductService, makeProductService)

// Property-based test generator
const generateProduct = (size: number) =>
  FastCheck.record({
    id: FastCheck.string({ minLength: 1, maxLength: 20 }),
    name: FastCheck.string({ minLength: 1, maxLength: Math.max(10, size) }),
    price: FastCheck.float({ min: 0.01, max: size * 100 }),
    category: FastCheck.constantFrom("electronics", "books", "clothing", "food")
  })

const generateProducts = (size: number) =>
  FastCheck.array(generateProduct(size), { minLength: 1, maxLength: size })

// Test with different sizes
const testProductValidation = Effect.gen(function* () {
  const service = yield* ProductService
  const currentSize = yield* TestServices.size
  
  yield* TestServices.annotate(
    TestAnnotation.tagged("test-size"), 
    `Testing with size: ${currentSize}`
  )
  
  // Generate products based on current test size
  const products = yield* Effect.sync(() => 
    FastCheck.sample(generateProducts(currentSize), 1)[0]
  )
  
  // Test validation
  const validatedProducts = yield* service.validateProducts(products)
  expect(validatedProducts.length).toBe(products.length)
  
  // Test total calculation
  const total = yield* service.calculateTotal(products)
  const expectedTotal = products.reduce((sum, p) => sum + p.price, 0)
  expect(total).toBe(expectedTotal)
  
  // Log results with live services
  yield* TestServices.provideLive(
    Effect.log(`Tested ${products.length} products with total: $${total.toFixed(2)}`)
  )
  
  return { products: validatedProducts, total }
})

// Run tests with different sizes
const testWithDifferentSizes = Effect.gen(function* () {
  const results = []
  
  // Test with small size
  const smallResult = yield* TestServices.withSize(5)(testProductValidation)
  results.push(smallResult)
  
  // Test with medium size  
  const mediumResult = yield* TestServices.withSize(20)(testProductValidation)
  results.push(mediumResult)
  
  // Test with large size
  const largeResult = yield* TestServices.withSize(100)(testProductValidation)
  results.push(largeResult)
  
  return results
}).pipe(
  Effect.provide(ProductServiceLive),
  Effect.provide(TestServices.liveServices)
)
```

## Advanced Features Deep Dive

### Feature 1: Test Annotations and Metadata Tracking

Test annotations provide a way to collect metadata and structured logging throughout your test execution:

#### Basic Annotation Usage

```typescript
import { TestServices, TestAnnotation, Effect } from "effect"

// Create custom annotation keys
const TimingAnnotation = TestAnnotation.tagged<number>("timing")
const OperationAnnotation = TestAnnotation.tagged<string>("operation")
const ErrorCountAnnotation = TestAnnotation.number("errors")

const testWithDetailedAnnotations = Effect.gen(function* () {
  const startTime = Date.now()
  
  yield* TestServices.annotate(OperationAnnotation, "database-setup")
  yield* setupDatabase()
  
  yield* TestServices.annotate(OperationAnnotation, "user-creation")
  try {
    yield* createUsers()
  } catch (error) {
    yield* TestServices.annotate(ErrorCountAnnotation, 1)
    throw error
  }
  
  yield* TestServices.annotate(OperationAnnotation, "test-execution")
  const result = yield* runBusinessLogic()
  
  const endTime = Date.now()
  yield* TestServices.annotate(TimingAnnotation, endTime - startTime)
  
  // Access accumulated annotations
  const finalOperation = yield* TestServices.get(OperationAnnotation)
  const totalErrors = yield* TestServices.get(ErrorCountAnnotation)
  const executionTime = yield* TestServices.get(TimingAnnotation)
  
  expect(finalOperation).toBe("test-execution")
  expect(totalErrors).toBe(0)
  expect(executionTime).toBeGreaterThan(0)
  
  return result
})
```

#### Real-World Annotation Example: API Testing

```typescript
import { TestServices, TestAnnotation, Effect, pipe } from "effect"

// Custom annotations for API testing
const RequestCountAnnotation = TestAnnotation.number("request-count")
const ResponseTimeAnnotation = TestAnnotation.tagged<number[]>("response-times")
const EndpointAnnotation = TestAnnotation.tagged<string>("endpoint")

const trackApiCall = <A, E, R>(
  endpoint: string,
  effect: Effect.Effect<A, E, R>
): Effect.Effect<A, E, R> =>
  Effect.gen(function* () {
    yield* TestServices.annotate(EndpointAnnotation, endpoint)
    
    const startTime = Date.now()
    const result = yield* effect
    const endTime = Date.now()
    const responseTime = endTime - startTime
    
    // Increment request count
    const currentCount = yield* TestServices.get(RequestCountAnnotation)
    yield* TestServices.annotate(RequestCountAnnotation, currentCount + 1)
    
    // Track response times
    const currentTimes = yield* TestServices.get(ResponseTimeAnnotation)
    yield* TestServices.annotate(ResponseTimeAnnotation, [...currentTimes, responseTime])
    
    return result
  })

const testApiPerformance = Effect.gen(function* () {
  // Multiple API calls with tracking
  yield* trackApiCall("/users", fetchUsers())
  yield* trackApiCall("/products", fetchProducts())
  yield* trackApiCall("/orders", fetchOrders())
  
  // Analyze results
  const totalRequests = yield* TestServices.get(RequestCountAnnotation)
  const responseTimes = yield* TestServices.get(ResponseTimeAnnotation)
  const averageResponseTime = responseTimes.reduce((a, b) => a + b, 0) / responseTimes.length
  
  expect(totalRequests).toBe(3)
  expect(averageResponseTime).toBeLessThan(1000) // Less than 1 second average
  
  // Log summary with live services
  yield* TestServices.provideLive(
    Effect.log(`API Performance: ${totalRequests} requests, avg ${averageResponseTime}ms`)
  )
})
```

### Feature 2: Live Service Integration

The TestLive service allows you to selectively use real implementations while keeping the rest of your test environment controlled:

#### Advanced Live Service Usage

```typescript
import { TestServices, Effect, Console, Logger, Clock } from "effect"

const testWithSelectiveLiveServices = Effect.gen(function* () {
  // Use test services for timing control
  yield* TestClock.adjust(Duration.minutes(5))
  
  // Use live services for console output and logging
  yield* TestServices.provideLive(
    Effect.gen(function* () {
      yield* Console.log("Starting integration test...")
      yield* Logger.info("Test environment initialized")
      
      const timestamp = yield* Clock.currentTimeMillis
      yield* Console.log(`Test started at: ${new Date(timestamp).toISOString()}`)
    })
  )
  
  // Back to test services for controlled behavior
  const result = yield* simulateUserInteractions()
  
  // Live services for final reporting
  yield* TestServices.provideLive(
    Console.log(`Integration test completed: ${JSON.stringify(result)}`)
  )
  
  return result
})
```

#### Custom Live Service Implementation

```typescript
import { TestServices, Effect, Context, Layer } from "effect"

// Create a custom service that needs both test and live behavior
interface NotificationService {
  readonly sendEmail: (to: string, subject: string) => Effect.Effect<void>
  readonly logSentEmails: () => Effect.Effect<ReadonlyArray<string>>
}

const NotificationService = Context.GenericTag<NotificationService>("NotificationService")

const makeTestNotificationService = Effect.gen(function* () {
  const sentEmails: string[] = []
  
  const sendEmail = (to: string, subject: string): Effect.Effect<void> =>
    Effect.gen(function* () {
      // Store email in test state
      sentEmails.push(`${to}: ${subject}`)
      
      // Use live services for actual logging
      yield* TestServices.provideLive(
        Effect.log(`📧 Email sent to ${to}: ${subject}`)
      )
    })
  
  const logSentEmails = (): Effect.Effect<ReadonlyArray<string>> =>
    Effect.succeed(sentEmails)
  
  return { sendEmail, logSentEmails }
})

const TestNotificationServiceLive = Layer.effect(NotificationService, makeTestNotificationService)

const testEmailNotifications = Effect.gen(function* () {
  const notificationService = yield* NotificationService
  
  // Send test emails
  yield* notificationService.sendEmail("user@example.com", "Welcome!")
  yield* notificationService.sendEmail("admin@example.com", "New user registered")
  
  // Verify emails were "sent"
  const sentEmails = yield* notificationService.logSentEmails()
  expect(sentEmails).toHaveLength(2)
  expect(sentEmails[0]).toContain("Welcome!")
  
  return sentEmails
}).pipe(
  Effect.provide(TestNotificationServiceLive),
  Effect.provide(TestServices.liveServices)
)
```

### Feature 3: TestConfig and Customizable Test Behavior

TestConfig allows you to customize retry behavior, sampling, and other test parameters:

#### Advanced TestConfig Usage

```typescript
import { TestServices, Effect, Schedule, Duration } from "effect"

// Create custom test configuration
const customTestConfig = TestServices.testConfigLayer({
  repeats: 5,      // Run each test 5 times for stability
  retries: 2,      // Retry failed tests up to 2 times
  samples: 50,     // Use 50 samples for property-based tests
  shrinks: 500     // Try up to 500 shrinkings to minimize failures
})

const testWithCustomConfig = Effect.gen(function* () {
  // Access current test configuration
  const config = yield* TestServices.testConfig
  const repeats = yield* TestServices.repeats
  const retries = yield* TestServices.retries
  
  yield* TestServices.annotate(
    TestAnnotation.tagged("config"),
    `repeats: ${repeats}, retries: ${retries}`
  )
  
  // Use config values in your test logic
  const results = []
  for (let i = 0; i < repeats; i++) {
    const result = yield* runSingleTest()
    results.push(result)
  }
  
  return results
}).pipe(
  Effect.provide(customTestConfig),
  Effect.provide(TestServices.liveServices)
)
```

## Practical Patterns & Best Practices

### Pattern 1: Test Environment Factory

Create reusable test environments for different scenarios:

```typescript
import { TestServices, Layer, Effect, Context } from "effect"

// Create a comprehensive test environment
const createTestEnvironment = (config: {
  timeControl?: boolean
  liveLogging?: boolean
  customSize?: number
  customConfig?: {
    repeats?: number
    retries?: number
    samples?: number
    shrinks?: number
  }
}) => {
  const baseLayer = TestServices.liveServices
  
  let testLayer = Layer.succeed(Context.empty(), baseLayer)
  
  if (config.customSize) {
    testLayer = Layer.provideMerge(testLayer, TestServices.sizedLayer(config.customSize))
  }
  
  if (config.customConfig) {
    testLayer = Layer.provideMerge(
      testLayer, 
      TestServices.testConfigLayer({
        repeats: config.customConfig.repeats ?? 100,
        retries: config.customConfig.retries ?? 100,
        samples: config.customConfig.samples ?? 200,
        shrinks: config.customConfig.shrinks ?? 1000
      })
    )
  }
  
  return testLayer
}

// Usage examples
const fastTestEnvironment = createTestEnvironment({
  timeControl: true,
  customSize: 10,
  customConfig: { repeats: 1, retries: 0, samples: 10, shrinks: 10 }
})

const thoroughTestEnvironment = createTestEnvironment({
  liveLogging: true,
  customSize: 100,
  customConfig: { repeats: 10, retries: 3, samples: 1000, shrinks: 5000 }
})

const debugTestEnvironment = createTestEnvironment({
  timeControl: true,
  liveLogging: true,
  customSize: 5
})
```

### Pattern 2: Test Lifecycle Management

Manage test setup, execution, and cleanup with TestServices:

```typescript
import { TestServices, Effect, Scope, TestAnnotation } from "effect"

const TestPhaseAnnotation = TestAnnotation.tagged<string>("test-phase")

const withTestLifecycle = <A, E, R>(
  testName: string,
  setup: Effect.Effect<void, E, R>,
  test: Effect.Effect<A, E, R>,
  cleanup: Effect.Effect<void, E, R>
): Effect.Effect<A, E, R> =>
  Effect.gen(function* () {
    yield* TestServices.annotate(TestPhaseAnnotation, "setup")
    yield* TestServices.provideLive(Effect.log(`🚀 Starting test: ${testName}`))
    
    yield* setup
    
    yield* TestServices.annotate(TestPhaseAnnotation, "execution")
    const result = yield* test
    
    yield* TestServices.annotate(TestPhaseAnnotation, "cleanup")
    yield* cleanup
    
    yield* TestServices.provideLive(Effect.log(`✅ Completed test: ${testName}`))
    
    return result
  }).pipe(
    Effect.catchAll(error => 
      Effect.gen(function* () {
        yield* TestServices.annotate(TestPhaseAnnotation, "error")
        yield* TestServices.provideLive(Effect.log(`❌ Test failed: ${testName}`))
        yield* cleanup // Ensure cleanup runs even on failure
        return yield* Effect.fail(error)
      })
    )
  )

// Usage
const testUserWorkflow = withTestLifecycle(
  "User Registration Workflow",
  // Setup
  Effect.gen(function* () {
    yield* initializeDatabase()
    yield* seedTestData()
  }),
  // Test
  Effect.gen(function* () {
    const user = yield* registerUser("test@example.com")
    const profile = yield* createUserProfile(user.id)
    return { user, profile }
  }),
  // Cleanup
  Effect.gen(function* () {
    yield* clearDatabase()
    yield* resetTestData()
  })
)
```

### Pattern 3: Conditional Test Services

Switch between test and live services based on conditions:

```typescript
import { TestServices, Effect, Context } from "effect"

const withConditionalServices = <A, E, R>(
  condition: boolean,
  effect: Effect.Effect<A, E, R>
): Effect.Effect<A, E, R> =>
  condition 
    ? TestServices.provideLive(effect)
    : effect

const testWithEnvironmentAwareness = Effect.gen(function* () {
  const isCI = process.env.CI === "true"
  const isDebug = process.env.DEBUG === "true"
  
  // Use live services in CI for better debugging
  yield* withConditionalServices(
    isCI,
    Effect.log("Running in CI environment")
  )
  
  // Enable verbose logging in debug mode
  if (isDebug) {
    yield* TestServices.annotate(
      TestAnnotation.tagged("debug"), 
      "Verbose logging enabled"
    )
  }
  
  const result = yield* runMainTest()
  
  // Always use live services for final reporting in CI
  yield* withConditionalServices(
    isCI,
    Effect.log(`Test result: ${JSON.stringify(result)}`)
  )
  
  return result
})
```

## Integration Examples

### Integration with Popular Testing Frameworks

#### Vitest Integration

```typescript
import { Effect, TestServices } from "effect"
import { describe, it } from "@effect/vitest"

describe("User Service", () => {
  it.effect("should create and retrieve users", () =>
    Effect.gen(function* () {
      const userService = yield* UserService
      
      const user = yield* userService.create({
        email: "test@example.com",
        name: "Test User"
      })
      
      const retrieved = yield* userService.getById(user.id)
      expect(retrieved).toEqual(user)
      
      return user
    }).pipe(
      Effect.provide(UserServiceLive),
      Effect.provide(TestServices.liveServices)
    )
  )
  
  it.effect("should handle time-based operations", () =>
    Effect.gen(function* () {
      const userService = yield* UserService
      
      // Create user with trial period
      const user = yield* userService.createTrialUser("trial@example.com")
      expect(user.status).toBe("trial")
      
      // Fast-forward past trial period
      yield* TestClock.adjust(Duration.days(15))
      
      const expiredUser = yield* userService.checkStatus(user.id)
      expect(expiredUser.status).toBe("expired")
      
      return expiredUser
    }).pipe(
      Effect.provide(UserServiceLive),
      Effect.provide(TestServices.liveServices)
    )
  )
})
```

#### Jest Integration

```typescript
import { Effect, TestServices, Runtime } from "effect"

// Create a test runtime with TestServices
const testRuntime = Runtime.defaultRuntime.pipe(
  Runtime.provide(TestServices.liveServices)
)

const runTest = <A>(effect: Effect.Effect<A>) =>
  Runtime.runPromise(testRuntime)(effect)

describe("Product Service", () => {
  test("should calculate product totals correctly", async () => {
    const result = await runTest(
      Effect.gen(function* () {
        const products = [
          { id: "1", name: "Book", price: 29.99 },
          { id: "2", name: "Pen", price: 4.99 }
        ]
        
        const total = yield* calculateProductTotal(products)
        expect(total).toBe(34.98)
        
        return total
      })
    )
    
    expect(result).toBe(34.98)
  })
  
  test("should handle time-based pricing", async () => {
    await runTest(
      Effect.gen(function* () {
        const product = { id: "1", name: "Limited Offer", basePrice: 100 }
        
        // Regular price
        const regularPrice = yield* calculateDiscountedPrice(product)
        expect(regularPrice).toBe(100)
        
        // Flash sale price (50% off for 1 hour)
        yield* TestClock.adjust(Duration.hours(1))
        const salePrice = yield* calculateDiscountedPrice(product)
        expect(salePrice).toBe(50)
        
        // Back to regular price after sale
        yield* TestClock.adjust(Duration.hours(2))
        const backToRegular = yield* calculateDiscountedPrice(product)
        expect(backToRegular).toBe(100)
      })
    )
  })
})
```

### Integration with Property-Based Testing

```typescript
import { Effect, TestServices, FastCheck } from "effect"
import { describe, it } from "@effect/vitest"

// Property-based test helpers
const generateValidUser = FastCheck.record({
  email: FastCheck.emailAddress(),
  name: FastCheck.string({ minLength: 1, maxLength: 50 }),
  age: FastCheck.integer({ min: 18, max: 100 })
})

const generateUserList = (size: number) =>
  FastCheck.array(generateValidUser, { minLength: 1, maxLength: size })

describe("User Validation Properties", () => {
  it.effect("should validate any valid user", () =>
    Effect.gen(function* () {
      const currentSize = yield* TestServices.size
      
      // Generate test data based on current size
      const users = yield* Effect.sync(() =>
        FastCheck.sample(generateUserList(currentSize), 1)[0]
      )
      
      // Property: all valid users should pass validation
      for (const user of users) {
        const validationResult = yield* validateUser(user)
        expect(validationResult.isValid).toBe(true)
      }
      
      // Property: user count should be preserved
      const processedUsers = yield* processUserBatch(users)
      expect(processedUsers.length).toBe(users.length)
      
      yield* TestServices.annotate(
        TestAnnotation.tagged("property-test"),
        `Tested ${users.length} users with size ${currentSize}`
      )
      
      return users
    }).pipe(
      Effect.provide(TestServices.liveServices)
    )
  )
})

// Run property tests with different sizes
const runPropertyTestsWithSizes = Effect.gen(function* () {
  const sizes = [5, 20, 100]
  const results = []
  
  for (const size of sizes) {
    const result = yield* TestServices.withSize(size)(
      Effect.gen(function* () {
        const currentSize = yield* TestServices.size
        yield* TestServices.provideLive(
          Effect.log(`Running property tests with size: ${currentSize}`)
        )
        
        // Your property tests here
        return yield* runUserValidationProperties()
      })
    )
    results.push(result)
  }
  
  return results
}).pipe(
  Effect.provide(TestServices.liveServices)
)
```

### Testing Strategies

#### Comprehensive Test Suite Pattern

```typescript
import { 
  Effect, 
  TestServices, 
  TestClock, 
  Duration, 
  Layer, 
  Context 
} from "effect"

// Test suite configuration
interface TestSuiteConfig {
  readonly enableTimeControl: boolean
  readonly enableLiveLogging: boolean
  readonly testSize: number
  readonly testRepeats: number
}

const createTestSuite = (config: TestSuiteConfig) => {
  const testLayer = Layer.mergeAll(
    TestServices.liveServices,
    TestServices.sizedLayer(config.testSize),
    TestServices.testConfigLayer({
      repeats: config.testRepeats,
      retries: 3,
      samples: config.testSize * 10,
      shrinks: 1000
    })
  )
  
  return {
    unitTests: (tests: Effect.Effect<void>[]) =>
      Effect.gen(function* () {
        yield* TestServices.annotate(TestAnnotation.tagged("suite"), "unit")
        
        for (const test of tests) {
          yield* test
        }
      }).pipe(
        Effect.provide(testLayer)
      ),
    
    integrationTests: (tests: Effect.Effect<void>[]) =>
      Effect.gen(function* () {
        yield* TestServices.annotate(TestAnnotation.tagged("suite"), "integration")
        
        if (config.enableLiveLogging) {
          yield* TestServices.provideLive(
            Effect.log("🔧 Starting integration tests...")
          )
        }
        
        for (const test of tests) {
          yield* test
        }
        
        if (config.enableLiveLogging) {
          yield* TestServices.provideLive(
            Effect.log("✅ Integration tests completed")
          )
        }
      }).pipe(
        Effect.provide(testLayer)
      ),
    
    performanceTests: (tests: Effect.Effect<void>[]) =>
      Effect.gen(function* () {
        yield* TestServices.annotate(TestAnnotation.tagged("suite"), "performance")
        
        const startTime = yield* TestClock.currentTimeMillis
        
        for (const test of tests) {
          yield* test
        }
        
        const endTime = yield* TestClock.currentTimeMillis
        const duration = endTime - startTime
        
        yield* TestServices.annotate(
          TestAnnotation.tagged("performance-duration"),
          duration.toString()
        )
        
        if (config.enableLiveLogging) {
          yield* TestServices.provideLive(
            Effect.log(`⚡ Performance tests completed in ${duration}ms`)
          )
        }
      }).pipe(
        Effect.provide(testLayer)
      )
  }
}

// Usage
const testSuite = createTestSuite({
  enableTimeControl: true,
  enableLiveLogging: true,
  testSize: 50,
  testRepeats: 3
})

const runAllTests = Effect.gen(function* () {
  yield* testSuite.unitTests([
    testUserValidation,
    testProductCalculations,
    testOrderProcessing
  ])
  
  yield* testSuite.integrationTests([
    testDatabaseIntegration,
    testApiIntegration,
    testEmailService
  ])
  
  yield* testSuite.performanceTests([
    testLargeDataProcessing,
    testConcurrentOperations,
    testMemoryUsage
  ])
})
```

## Conclusion

TestServices provides comprehensive testing infrastructure for Effect applications through controlled time manipulation, annotation tracking, selective live service access, and configurable test behavior.

Key benefits:
- **Deterministic Testing**: Control time, randomness, and external dependencies for reliable tests
- **Rich Metadata**: Track test execution with structured annotations and timing information
- **Flexible Service Management**: Choose between test and live implementations on a per-service basis
- **Scalable Test Configuration**: Adjust test parameters for different scenarios and environments

TestServices is essential when building robust Effect applications that require thorough testing of time-dependent behavior, retry mechanisms, property-based testing, and integration scenarios while maintaining fast, reliable, and deterministic test execution.