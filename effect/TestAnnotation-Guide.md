# TestAnnotation: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem TestAnnotation Solves

When running test suites, developers often struggle with organizing, categorizing, and analyzing test results. Traditional testing frameworks provide limited metadata capabilities, making it difficult to:

```typescript
// Traditional approach - limited test metadata
describe("User Service", () => {
  it("should create user", async () => {
    // No way to tag this test as integration, slow, or requiring database
    // No way to track performance metrics or retry counts
    // Limited reporting and filtering capabilities
  })
})
```

This approach leads to:
- **Poor Test Organization** - No systematic way to categorize tests by type, performance, or environment
- **Limited Analytics** - Difficulty tracking test patterns, retry rates, and performance metrics
- **Inflexible Reporting** - Cannot filter or group tests by business requirements
- **Missing Context** - Test results lack rich metadata for debugging and analysis

### The TestAnnotation Solution

TestAnnotation provides a structured way to attach rich metadata to tests, enabling powerful categorization, reporting, and analysis capabilities.

```typescript
import { TestAnnotation, TestServices } from "effect"

// Rich, structured test metadata
const performanceTest = TestAnnotation.make(
  "performance",
  { slow: false, benchmark: false },
  (a, b) => ({ slow: a.slow || b.slow, benchmark: a.benchmark || b.benchmark })
)

const testWithMetadata = Effect.gen(function* () {
  // Add structured metadata to tests
  yield* TestServices.annotate(TestAnnotation.tagged, HashSet.fromIterable(["integration", "database"]))
  yield* TestServices.annotate(performanceTest, { slow: true, benchmark: false })
  
  // Your test logic here...
  const result = yield* createUser({ name: "Alice", email: "alice@example.com" })
  return result
})
```

### Key Concepts

**TestAnnotation**: A structured metadata container that defines how test information is stored and combined across test runs.

**Annotation Map**: A collection of annotations attached to a specific test execution, providing rich context and categorization.

**Built-in Annotations**: Pre-defined annotations like `tagged`, `ignored`, `repeated`, and `retried` for common testing scenarios.

## Basic Usage Patterns

### Pattern 1: Using Built-in Annotations

```typescript
import { Effect, TestAnnotation, TestServices, HashSet } from "effect"

// Tag tests for categorization and filtering
const taggedTest = Effect.gen(function* () {
  yield* TestServices.annotate(
    TestAnnotation.tagged,
    HashSet.fromIterable(["integration", "slow", "database"])
  )
  
  // Your test implementation
  const result = yield* performDatabaseOperation()
  return result
})

// Track ignored tests
const conditionalTest = Effect.gen(function* () {
  const shouldRun = yield* checkEnvironment()
  
  if (!shouldRun) {
    yield* TestServices.annotate(TestAnnotation.ignored, 1)
    return "skipped"
  }
  
  return yield* runTest()
})
```

### Pattern 2: Creating Custom Annotations

```typescript
// Custom annotation for test environment metadata
const environmentAnnotation = TestAnnotation.make(
  "environment",
  { type: "unit", database: false, external: false },
  (a, b) => ({
    type: b.type !== "unit" ? b.type : a.type,
    database: a.database || b.database,
    external: a.external || b.external
  })
)

// Custom annotation for performance tracking
const performanceAnnotation = TestAnnotation.make(
  "performance",
  { duration: 0, memory: 0 },
  (a, b) => ({
    duration: Math.max(a.duration, b.duration),
    memory: Math.max(a.memory, b.memory)
  })
)
```

### Pattern 3: Reading Annotation Values

```typescript
const examineTestMetadata = Effect.gen(function* () {
  // Get current test tags
  const tags = yield* TestServices.get(TestAnnotation.tagged)
  
  // Get retry count
  const retryCount = yield* TestServices.get(TestAnnotation.retried)
  
  // Get custom annotation values
  const performance = yield* TestServices.get(performanceAnnotation)
  
  console.log({ tags: HashSet.toArray(tags), retryCount, performance })
})
```

## Real-World Examples

### Example 1: Test Suite Organization with Categories

```typescript
import { Effect, TestAnnotation, TestServices, HashSet } from "effect"

// Define test categories
const testCategory = TestAnnotation.make(
  "category",
  { primary: "unit", secondary: [] as string[] },
  (a, b) => ({
    primary: b.primary,
    secondary: [...a.secondary, ...b.secondary]
  })
)

// User service tests with rich categorization
const createUserTest = Effect.gen(function* () {
  // Categorize as integration test requiring database
  yield* TestServices.annotate(
    TestAnnotation.tagged,
    HashSet.fromIterable(["user-service", "integration", "database"])
  )
  
  yield* TestServices.annotate(testCategory, {
    primary: "integration",
    secondary: ["database", "user-management"]
  })
  
  // Test implementation
  const user = { name: "Alice", email: "alice@example.com", role: "user" }
  const result = yield* createUser(user)
  
  // Verify results
  expect(result.id).toBeDefined()
  expect(result.name).toBe("Alice")
  
  return result
})

const validateUserTest = Effect.gen(function* () {
  // Categorize as unit test with validation focus
  yield* TestServices.annotate(
    TestAnnotation.tagged,
    HashSet.fromIterable(["user-service", "unit", "validation"])
  )
  
  yield* TestServices.annotate(testCategory, {
    primary: "unit",
    secondary: ["validation", "business-rules"]
  })
  
  // Test validation logic
  const invalidUser = { name: "", email: "invalid-email" }
  const result = yield* validateUser(invalidUser).pipe(
    Effect.flip, // Expect this to fail
    Effect.map(error => error.message)
  )
  
  expect(result).toContain("Invalid email format")
  return result
})
```

### Example 2: Performance Testing with Benchmarking

```typescript
// Define performance annotation
const performanceMetrics = TestAnnotation.make(
  "performance",
  { 
    startTime: 0, 
    endTime: 0, 
    duration: 0, 
    memoryBefore: 0, 
    memoryAfter: 0,
    operations: 0
  },
  (a, b) => ({
    startTime: Math.min(a.startTime || Date.now(), b.startTime || Date.now()),
    endTime: Math.max(a.endTime, b.endTime),
    duration: a.duration + b.duration,
    memoryBefore: a.memoryBefore || b.memoryBefore,
    memoryAfter: Math.max(a.memoryAfter, b.memoryAfter),
    operations: a.operations + b.operations
  })
)

const benchmarkDataProcessing = Effect.gen(function* () {
  // Tag as performance benchmark
  yield* TestServices.annotate(
    TestAnnotation.tagged,
    HashSet.fromIterable(["performance", "benchmark", "data-processing"])
  )
  
  // Record start metrics
  const startTime = Date.now()
  const memoryBefore = process.memoryUsage().heapUsed
  
  yield* TestServices.annotate(performanceMetrics, {
    startTime,
    endTime: 0,
    duration: 0,
    memoryBefore,
    memoryAfter: 0,
    operations: 0
  })
  
  // Perform intensive data processing
  const data = Array.from({ length: 100000 }, (_, i) => ({ id: i, value: Math.random() }))
  const processed = yield* processLargeDataset(data)
  
  // Record end metrics
  const endTime = Date.now()
  const memoryAfter = process.memoryUsage().heapUsed
  const duration = endTime - startTime
  
  yield* TestServices.annotate(performanceMetrics, {
    startTime: 0,
    endTime,
    duration,
    memoryBefore: 0,
    memoryAfter,
    operations: data.length
  })
  
  // Assert performance requirements
  expect(duration).toBeLessThan(5000) // Less than 5 seconds
  expect(processed.length).toBe(data.length)
  
  return { processed, duration, memoryUsed: memoryAfter - memoryBefore }
})
```

### Example 3: Test Environment and Dependency Tracking

```typescript
// Environment configuration annotation
const testEnvironment = TestAnnotation.make(
  "environment",
  {
    database: "none",
    external_apis: false,
    docker: false,
    ci: false,
    browser: "none"
  },
  (a, b) => ({
    database: b.database !== "none" ? b.database : a.database,
    external_apis: a.external_apis || b.external_apis,
    docker: a.docker || b.docker,
    ci: a.ci || b.ci,
    browser: b.browser !== "none" ? b.browser : a.browser
  })
)

// Dependency requirements annotation
const testDependencies = TestAnnotation.make(
  "dependencies",
  [] as string[],
  (a, b) => [...new Set([...a, ...b])]
)

const integrationTestWithEnvironment = Effect.gen(function* () {
  // Specify environment requirements
  yield* TestServices.annotate(testEnvironment, {
    database: "postgresql",
    external_apis: true,
    docker: true,
    ci: process.env.CI === "true",
    browser: "none"
  })
  
  // Track required dependencies
  yield* TestServices.annotate(testDependencies, [
    "postgresql",
    "redis",
    "stripe-api",
    "sendgrid-api"
  ])
  
  // Tag for filtering and reporting
  yield* TestServices.annotate(
    TestAnnotation.tagged,
    HashSet.fromIterable(["integration", "payment", "external"])
  )
  
  // Test implementation requiring multiple services
  const order = { 
    userId: "user-123", 
    items: [{ productId: "prod-456", quantity: 2 }],
    total: 29.99
  }
  
  const result = yield* Effect.gen(function* () {
    // Process payment with external API
    const payment = yield* processPayment(order.total)
    
    // Store order in database
    const savedOrder = yield* saveOrder({ ...order, paymentId: payment.id })
    
    // Send confirmation email
    yield* sendOrderConfirmation(savedOrder)
    
    return savedOrder
  })
  
  expect(result.id).toBeDefined()
  expect(result.status).toBe("confirmed")
  
  return result
})
```

## Advanced Features Deep Dive

### Feature 1: Custom Annotation Combiners

The combine function in TestAnnotation determines how values merge when annotations are combined across test runs or nested contexts.

#### Basic Combiner Usage

```typescript
// Simple additive combiner for counting
const requestCount = TestAnnotation.make(
  "requests",
  0,
  (a, b) => a + b
)

// Set union combiner for collecting unique values
const uniqueErrors = TestAnnotation.make(
  "errors",
  new Set<string>(),
  (a, b) => new Set([...a, ...b])
)
```

#### Real-World Combiner Example: Test Metrics

```typescript
interface TestMetrics {
  assertions: number
  apiCalls: number
  databaseQueries: number
  executionTime: number
  errorCount: number
  warnings: string[]
}

const testMetrics = TestAnnotation.make(
  "metrics",
  {
    assertions: 0,
    apiCalls: 0,
    databaseQueries: 0,
    executionTime: 0,
    errorCount: 0,
    warnings: []
  } as TestMetrics,
  (a, b) => ({
    assertions: a.assertions + b.assertions,
    apiCalls: a.apiCalls + b.apiCalls,
    databaseQueries: a.databaseQueries + b.databaseQueries,
    executionTime: Math.max(a.executionTime, b.executionTime), // Use maximum time
    errorCount: a.errorCount + b.errorCount,
    warnings: [...a.warnings, ...b.warnings] // Collect all warnings
  })
)

const complexTestWithMetrics = Effect.gen(function* () {
  // Track test execution metrics
  const startTime = Date.now()
  
  // Perform multiple operations and track each
  yield* TestServices.annotate(testMetrics, { 
    assertions: 1, apiCalls: 0, databaseQueries: 0, 
    executionTime: 0, errorCount: 0, warnings: [] 
  })
  
  const user = yield* fetchUser("user-123")
  yield* TestServices.annotate(testMetrics, { 
    assertions: 0, apiCalls: 1, databaseQueries: 1, 
    executionTime: 0, errorCount: 0, warnings: [] 
  })
  
  const orders = yield* fetchUserOrders(user.id)
  yield* TestServices.annotate(testMetrics, { 
    assertions: 0, apiCalls: 1, databaseQueries: 1, 
    executionTime: 0, errorCount: 0, warnings: [] 
  })
  
  // Record final execution time
  const endTime = Date.now()
  yield* TestServices.annotate(testMetrics, { 
    assertions: 0, apiCalls: 0, databaseQueries: 0, 
    executionTime: endTime - startTime, errorCount: 0, warnings: [] 
  })
  
  return { user, orders }
})
```

### Feature 2: Annotation Maps and Batch Operations

TestAnnotationMap provides powerful ways to work with multiple annotations simultaneously.

#### Working with Annotation Maps

```typescript
import { TestAnnotationMap } from "effect"

const createTestReport = Effect.gen(function* () {
  // Get all current annotations
  const annotations = yield* TestServices.annotations()
  const annotationMap = yield* annotations.ref.get
  
  // Extract specific annotation values
  const tags = TestAnnotationMap.get(annotationMap, TestAnnotation.tagged)
  const retryCount = TestAnnotationMap.get(annotationMap, TestAnnotation.retried)
  const metrics = TestAnnotationMap.get(annotationMap, testMetrics)
  
  // Create comprehensive test report
  const report = {
    tags: HashSet.toArray(tags),
    retryCount,
    metrics,
    timestamp: new Date().toISOString(),
    testId: crypto.randomUUID()
  }
  
  return report
})
```

### Feature 3: Advanced Annotation Patterns

#### Hierarchical Test Organization

```typescript
// Test suite annotation for hierarchical organization
const testSuite = TestAnnotation.make(
  "suite",
  { name: "", path: [] as string[], level: 0 },
  (a, b) => ({
    name: b.name || a.name,
    path: b.path.length > 0 ? b.path : a.path,
    level: Math.max(a.level, b.level)
  })
)

const createNestedTestSuite = (suiteName: string, path: string[]) => 
  Effect.gen(function* () {
    yield* TestServices.annotate(testSuite, {
      name: suiteName,
      path,
      level: path.length
    })
    
    return Effect.succeed(`Test suite: ${suiteName}`)
  })

// Usage in nested test structure
const userServiceTests = Effect.gen(function* () {
  yield* createNestedTestSuite("User Service", ["services", "user"])
  
  // Individual test inherits suite metadata
  const createUserTest = Effect.gen(function* () {
    yield* TestServices.annotate(testSuite, {
      name: "Create User Test",
      path: ["services", "user", "create"],
      level: 3
    })
    
    // Test implementation...
    return yield* testCreateUser()
  })
  
  return yield* createUserTest
})
```

## Practical Patterns & Best Practices

### Pattern 1: Test Categorization Helper

```typescript
// Helper for consistent test categorization
const categorizeTest = (config: {
  type: "unit" | "integration" | "e2e"
  tags: string[]
  environment?: {
    database?: string
    external?: boolean
    browser?: boolean
  }
  performance?: {
    slow?: boolean
    benchmark?: boolean
  }
}) => Effect.gen(function* () {
  // Apply standard tags
  const allTags = [config.type, ...config.tags]
  yield* TestServices.annotate(
    TestAnnotation.tagged,
    HashSet.fromIterable(allTags)
  )
  
  // Apply environment configuration
  if (config.environment) {
    yield* TestServices.annotate(testEnvironment, {
      database: config.environment.database || "none",
      external_apis: config.environment.external || false,
      docker: false,
      ci: false,
      browser: config.environment.browser ? "chrome" : "none"
    })
  }
  
  // Apply performance markers
  if (config.performance) {
    yield* TestServices.annotate(performanceTest, {
      slow: config.performance.slow || false,
      benchmark: config.performance.benchmark || false
    })
  }
})

// Usage example
const databaseIntegrationTest = Effect.gen(function* () {
  yield* categorizeTest({
    type: "integration",
    tags: ["database", "user-service"],
    environment: {
      database: "postgresql",
      external: false
    }
  })
  
  // Test implementation
  return yield* testDatabaseOperations()
})
```

### Pattern 2: Conditional Test Execution

```typescript
// Helper for conditional test execution based on environment
const conditionalTest = <A, E, R>(
  condition: Effect.Effect<boolean, E, R>,
  test: Effect.Effect<A, E, R>,
  reason: string
) => Effect.gen(function* () {
  const shouldRun = yield* condition
  
  if (!shouldRun) {
    yield* TestServices.annotate(TestAnnotation.ignored, 1)
    yield* TestServices.annotate(
      TestAnnotation.tagged,
      HashSet.fromIterable(["skipped"])
    )
    
    console.log(`Test skipped: ${reason}`)
    return Option.none<A>()
  }
  
  const result = yield* test
  return Option.some(result)
})

// Usage for environment-dependent tests
const externalApiTest = conditionalTest(
  Effect.sync(() => process.env.NODE_ENV !== "ci"),
  Effect.gen(function* () {
    yield* categorizeTest({
      type: "integration",
      tags: ["external-api", "payment"],
      environment: { external: true }
    })
    
    return yield* testPaymentProcessing()
  }),
  "External API tests disabled in CI environment"
)
```

### Pattern 3: Test Retry Logic with Annotations

```typescript
// Enhanced retry logic that tracks attempts
const retryableTest = <A, E, R>(
  test: Effect.Effect<A, E, R>,
  maxRetries: number = 3
) => Effect.gen(function* () {
  let attempt = 0
  
  const attemptTest = (): Effect.Effect<A, E, R> => Effect.gen(function* () {
    attempt++
    
    if (attempt > 1) {
      yield* TestServices.annotate(TestAnnotation.retried, 1)
      console.log(`Test retry attempt ${attempt}/${maxRetries + 1}`)
    }
    
    return yield* test
  })
  
  return yield* attemptTest().pipe(
    Effect.retry(Schedule.recurs(maxRetries))
  )
})

// Usage with flaky external service
const flaky
test = retryableTest(
  Effect.gen(function* () {
    yield* categorizeTest({
      type: "integration",
      tags: ["external", "flaky"],
      environment: { external: true }
    })
    
    // This might fail due to network issues
    return yield* callExternalService()
  }),
  3 // Retry up to 3 times
)
```

## Integration Examples

### Integration with Vitest Testing Framework

```typescript
import { describe, it, expect } from "vitest"
import { Effect, TestServices, TestAnnotation, HashSet } from "effect"

// Helper to run Effect tests with annotations
const runEffectTest = <A>(
  name: string,
  test: Effect.Effect<A, never, never>
) => {
  return it(name, async () => {
    const result = await Effect.runPromise(
      test.pipe(
        Effect.provide(TestServices.liveServices)
      )
    )
    return result
  })
}

// Test suite with rich annotations
describe("User Service Integration Tests", () => {
  runEffectTest("should create user with validation", Effect.gen(function* () {
    yield* TestServices.annotate(
      TestAnnotation.tagged,
      HashSet.fromIterable(["user-service", "validation", "integration"])
    )
    
    yield* TestServices.annotate(testEnvironment, {
      database: "postgresql",
      external_apis: false,
      docker: true,
      ci: process.env.CI === "true",
      browser: "none"
    })
    
    const user = { name: "Alice", email: "alice@example.com" }
    const result = yield* createUser(user)
    
    expect(result.id).toBeDefined()
    expect(result.name).toBe("Alice")
    expect(result.email).toBe("alice@example.com")
    
    return result
  }))
  
  runEffectTest("should handle user creation errors", Effect.gen(function* () {
    yield* TestServices.annotate(
      TestAnnotation.tagged,
      HashSet.fromIterable(["user-service", "error-handling", "unit"])
    )
    
    const invalidUser = { name: "", email: "invalid" }
    
    const result = yield* createUser(invalidUser).pipe(
      Effect.flip,
      Effect.map(error => error.message)
    )
    
    expect(result).toContain("Invalid user data")
    return result
  }))
})
```

### Integration with Test Reporting Systems

```typescript
// Custom test reporter that utilizes annotations
interface TestReport {
  testId: string
  name: string
  tags: string[]
  environment: any
  performance: any
  result: "passed" | "failed" | "skipped"
  duration: number
  retryCount: number
  metadata: Record<string, any>
}

const generateTestReport = (testName: string) => Effect.gen(function* () {
  // Collect all annotation data
  const tags = yield* TestServices.get(TestAnnotation.tagged)
  const retryCount = yield* TestServices.get(TestAnnotation.retried)
  const ignoredCount = yield* TestServices.get(TestAnnotation.ignored)
  const environment = yield* TestServices.get(testEnvironment)
  const performance = yield* TestServices.get(performanceMetrics)
  
  const report: TestReport = {
    testId: crypto.randomUUID(),
    name: testName,
    tags: HashSet.toArray(tags),
    environment,
    performance,
    result: ignoredCount > 0 ? "skipped" : "passed",
    duration: performance.duration,
    retryCount,
    metadata: {
      timestamp: new Date().toISOString(),
      nodeVersion: process.version,
      platform: process.platform
    }
  }
  
  // Send to reporting system
  yield* sendToReportingSystem(report)
  
  return report
})

const sendToReportingSystem = (report: TestReport) => Effect.gen(function* () {
  // Integration with external reporting service
  if (process.env.TEST_REPORTING_ENDPOINT) {
    yield* Effect.tryPromise(() =>
      fetch(process.env.TEST_REPORTING_ENDPOINT!, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(report)
      })
    )
  }
  
  // Local file reporting
  yield* Effect.sync(() => {
    console.log(`Test Report: ${report.name}`)
    console.log(`  Tags: ${report.tags.join(", ")}`)
    console.log(`  Duration: ${report.duration}ms`)
    console.log(`  Retries: ${report.retryCount}`)
  })
})
```

### Integration with CI/CD Pipeline Filtering

```typescript
// CI/CD integration for selective test execution
const shouldRunTest = (requiredTags: string[]) => Effect.gen(function* () {
  const currentTags = yield* TestServices.get(TestAnnotation.tagged)
  const tagArray = HashSet.toArray(currentTags)
  
  const hasRequiredTags = requiredTags.every(tag => tagArray.includes(tag))
  const environment = yield* TestServices.get(testEnvironment)
  
  // Skip external API tests in CI
  if (process.env.CI === "true" && environment.external_apis) {
    return false
  }
  
  // Skip slow tests in PR builds
  if (process.env.BUILD_TYPE === "pr" && tagArray.includes("slow")) {
    return false
  }
  
  return hasRequiredTags
})

// Conditional test execution based on CI environment
const smartTest = <A, E, R>(
  name: string,
  test: Effect.Effect<A, E, R>,
  requiredTags: string[] = []
) => Effect.gen(function* () {
  const shouldRun = yield* shouldRunTest(requiredTags)
  
  if (!shouldRun) {
    yield* TestServices.annotate(TestAnnotation.ignored, 1)
    console.log(`Skipping test: ${name} (CI environment constraints)`)
    return Option.none<A>()
  }
  
  const result = yield* test
  yield* generateTestReport(name)
  
  return Option.some(result)
})
```

## Conclusion

TestAnnotation provides comprehensive metadata management for Effect-based testing, enabling rich test categorization, performance tracking, and intelligent test execution strategies.

Key benefits:
- **Rich Metadata** - Attach structured information to tests for better organization and reporting
- **Flexible Categorization** - Tag and filter tests by type, environment, performance characteristics
- **Advanced Analytics** - Track retry counts, performance metrics, and environmental dependencies
- **CI/CD Integration** - Enable intelligent test selection based on build context and requirements

TestAnnotation transforms basic test execution into a sophisticated testing platform with deep insights into test behavior, performance patterns, and environmental dependencies. Use it when you need more than simple pass/fail results and want to build intelligent, data-driven testing workflows.