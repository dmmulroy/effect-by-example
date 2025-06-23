# TestAnnotations: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem TestAnnotations Solves

Testing complex applications often requires tracking metadata across test execution: performance metrics, test classifications, retry counts, fiber lifecycle information, and custom annotations for debugging and reporting. Traditional testing approaches struggle with this:

```typescript
// Traditional approach - fragmented test metadata
let testMetrics = {}
let retryCount = 0
let taggedTests = new Set()

beforeEach(() => {
  testMetrics = { start: Date.now() }
  retryCount = 0
})

afterEach(() => {
  if (retryCount > 0) {
    console.log(`Test retried ${retryCount} times`)
  }
  console.log(`Test took ${Date.now() - testMetrics.start}ms`)
})

it('should process data', () => {
  taggedTests.add('integration')
  // Test implementation with scattered metadata tracking
})
```

This approach leads to:
- **Scattered State Management** - Metadata spread across global variables and test hooks
- **Type Unsafe Annotations** - No compile-time guarantees about annotation structure
- **Poor Composability** - Difficult to combine annotations from different test layers
- **Manual Aggregation** - Complex logic required to collect and merge test metadata

### The TestAnnotations Solution

TestAnnotations provides a structured, type-safe service for managing test metadata that composes naturally with Effect's service system:

```typescript
import { TestAnnotations, TestAnnotation, TestServices } from "effect"

// Type-safe, composable test annotations
const performanceTest = Effect.gen(function* () {
  // Annotate test with custom metadata
  yield* TestServices.annotate(TestAnnotation.tagged, HashSet.make("performance", "critical"))
  
  // Track custom metrics
  const startTime = yield* TestServices.annotate(customStartTime, Date.now())
  
  // Your test logic here
  const result = yield* processLargeDataset()
  
  // Annotations automatically combine and aggregate
  return result
})
```

### Key Concepts

**TestAnnotation**: A typed annotation definition with an identifier, initial value, and combine function for aggregating values across test execution.

**TestAnnotationMap**: A collection of annotations that automatically handles type-safe storage and retrieval of different annotation types.

**TestAnnotations Service**: The main service interface providing `get` and `annotate` operations for managing test metadata throughout test execution.

## Basic Usage Patterns

### Pattern 1: Setup and Basic Annotation

```typescript
import { Effect, TestAnnotation, TestServices, HashSet } from "effect"

// Create custom annotation types
const executionTime = TestAnnotation.make(
  "execution_time", 
  0, 
  (a: number, b: number) => Math.max(a, b)
)

const testCategories = TestAnnotation.make(
  "categories",
  HashSet.empty<string>(),
  (a, b) => HashSet.union(a, b)
)

// Basic annotation usage
const annotatedTest = Effect.gen(function* () {
  // Add test categories
  yield* TestServices.annotate(testCategories, HashSet.make("unit", "fast"))
  
  // Record execution time
  yield* TestServices.annotate(executionTime, 150)
  
  // Retrieve current annotations
  const categories = yield* TestServices.get(testCategories)
  const maxTime = yield* TestServices.get(executionTime)
  
  console.log(`Categories: ${Array.from(categories).join(", ")}`)
  console.log(`Max execution time: ${maxTime}ms`)
})
```

### Pattern 2: Built-in Annotations

```typescript
import { TestAnnotation } from "effect"

// Using predefined annotations
const retryTest = Effect.gen(function* () {
  // Track test retries
  yield* TestServices.annotate(TestAnnotation.retried, 1)
  
  // Add test tags
  yield* TestServices.annotate(
    TestAnnotation.tagged, 
    HashSet.make("flaky", "network")
  )
  
  // Mark as repeated test
  yield* TestServices.annotate(TestAnnotation.repeated, 1)
  
  // Your test logic
  const result = yield* performNetworkOperation()
  return result
})
```

### Pattern 3: Annotation Retrieval and Reporting

```typescript
// Helper for generating test reports
const generateTestReport = Effect.gen(function* () {
  const retryCount = yield* TestServices.get(TestAnnotation.retried)
  const tags = yield* TestServices.get(TestAnnotation.tagged)
  const repeatCount = yield* TestServices.get(TestAnnotation.repeated)
  
  return {
    retries: retryCount,
    tags: Array.from(tags),
    repeats: repeatCount,
    timestamp: new Date().toISOString()
  }
})
```

## Real-World Examples

### Example 1: Performance Test Suite with Metrics

```typescript
import { Effect, TestAnnotation, TestServices, HashSet, Layer } from "effect"

// Custom annotations for performance testing
const executionTime = TestAnnotation.make(
  "execution_time",
  0,
  (a: number, b: number) => Math.max(a, b)
)

const memoryUsage = TestAnnotation.make(
  "memory_usage",
  0,
  (a: number, b: number) => Math.max(a, b)
)

const performanceThreshold = TestAnnotation.make(
  "performance_threshold",
  "unknown" as "fast" | "medium" | "slow" | "unknown",
  (a, b) => a === "unknown" ? b : a
)

// Performance test implementation
const performanceTest = (testName: string, threshold: number) =>
  Effect.gen(function* () {
    // Start performance tracking
    const startTime = performance.now()
    const startMemory = process.memoryUsage().heapUsed
    
    // Classify performance expectations
    const classification = threshold < 100 ? "fast" : 
                          threshold < 1000 ? "medium" : "slow"
    
    yield* TestServices.annotate(performanceThreshold, classification)
    yield* TestServices.annotate(
      TestAnnotation.tagged, 
      HashSet.make("performance", classification)
    )
    
    // Run the actual test
    const result = yield* simulateHeavyComputation()
    
    // Record performance metrics
    const endTime = performance.now()
    const endMemory = process.memoryUsage().heapUsed
    const executionMs = Math.round(endTime - startTime)
    const memoryMB = Math.round((endMemory - startMemory) / 1024 / 1024)
    
    yield* TestServices.annotate(executionTime, executionMs)
    yield* TestServices.annotate(memoryUsage, memoryMB)
    
    // Validate against threshold
    if (executionMs > threshold) {
      yield* TestServices.annotate(
        TestAnnotation.tagged, 
        HashSet.make("slow", "needs-optimization")
      )
    }
    
    return { result, executionMs, memoryMB }
  })

// Helper for simulating work
const simulateHeavyComputation = Effect.gen(function* () {
  yield* Effect.sleep("100 millis")
  return Array.from({ length: 1000 }, (_, i) => i * i).reduce((a, b) => a + b, 0)
})

// Performance report generation
const generatePerformanceReport = Effect.gen(function* () {
  const maxExecution = yield* TestServices.get(executionTime)
  const maxMemory = yield* TestServices.get(memoryUsage)
  const threshold = yield* TestServices.get(performanceThreshold)
  const tags = yield* TestServices.get(TestAnnotation.tagged)
  
  return {
    performance: {
      maxExecutionTime: `${maxExecution}ms`,
      maxMemoryUsage: `${maxMemory}MB`,
      classification: threshold,
      tags: Array.from(tags),
      passed: !tags.has("slow")
    }
  }
})
```

### Example 2: Integration Test with Service Dependencies

```typescript
import { Effect, Context, Layer, TestServices, TestAnnotation } from "effect"

// Service dependency annotations
const serviceAnnotation = TestAnnotation.make(
  "required_services",
  [] as string[],
  (a: string[], b: string[]) => [...new Set([...a, ...b])]
)

const integrationScope = TestAnnotation.make(
  "integration_scope",
  "unit" as "unit" | "integration" | "e2e",
  (a, b) => b !== "unit" ? b : a
)

// Mock services for testing
interface DatabaseService {
  readonly findUser: (id: string) => Effect.Effect<{ id: string; name: string }>
}

interface EmailService {
  readonly sendEmail: (to: string, subject: string) => Effect.Effect<void>
}

const DatabaseService = Context.GenericTag<DatabaseService>("DatabaseService")
const EmailService = Context.GenericTag<EmailService>("EmailService")

// Integration test with service tracking
const userNotificationTest = Effect.gen(function* () {
  // Track integration scope and required services
  yield* TestServices.annotate(integrationScope, "integration")
  yield* TestServices.annotate(serviceAnnotation, ["database", "email"])
  yield* TestServices.annotate(
    TestAnnotation.tagged,
    HashSet.make("integration", "user-management", "notifications")
  )
  
  // Use services (automatically tracked)
  const database = yield* DatabaseService
  const email = yield* EmailService
  
  // Test business logic
  const user = yield* database.findUser("user-123")
  yield* email.sendEmail(user.name, "Welcome!")
  
  return { success: true, userId: user.id }
}).pipe(
  Effect.provide(Layer.mergeAll(
    Layer.succeed(DatabaseService, {
      findUser: (id) => Effect.succeed({ id, name: `User ${id}` })
    }),
    Layer.succeed(EmailService, {
      sendEmail: (to, subject) => Effect.logInfo(`Email sent to ${to}: ${subject}`)
    })
  ))
)

// Integration test report
const generateIntegrationReport = Effect.gen(function* () {
  const scope = yield* TestServices.get(integrationScope)
  const services = yield* TestServices.get(serviceAnnotation)
  const tags = yield* TestServices.get(TestAnnotation.tagged)
  const retries = yield* TestServices.get(TestAnnotation.retried)
  
  return {
    testScope: scope,
    requiredServices: services,
    tags: Array.from(tags),
    retryCount: retries,
    complexity: services.length > 2 ? "high" : "medium"
  }
})
```

### Example 3: Test Suite with Fiber Supervision

```typescript
import { Effect, Fiber, TestServices, TestAnnotation } from "effect"

// Custom annotation for concurrent test tracking
const concurrentOperations = TestAnnotation.make(
  "concurrent_operations",
  0,
  (a: number, b: number) => a + b
)

// Concurrent test with fiber supervision
const concurrentDataProcessing = Effect.gen(function* () {
  yield* TestServices.annotate(
    TestAnnotation.tagged,
    HashSet.make("concurrent", "data-processing")
  )
  
  // Start multiple concurrent operations
  const fiber1 = yield* Effect.fork(processChunk(1))
  const fiber2 = yield* Effect.fork(processChunk(2))
  const fiber3 = yield* Effect.fork(processChunk(3))
  
  yield* TestServices.annotate(concurrentOperations, 3)
  
  // Wait for all fibers
  const results = yield* Effect.all([
    Fiber.join(fiber1),
    Fiber.join(fiber2),
    Fiber.join(fiber3)
  ])
  
  // Check supervised fibers
  const supervisedFibers = yield* TestServices.supervisedFibers()
  console.log(`Supervised ${supervisedFibers.size} fibers during test`)
  
  return results
})

const processChunk = (chunkId: number) => Effect.gen(function* () {
  yield* Effect.sleep("50 millis")
  yield* Effect.logInfo(`Processed chunk ${chunkId}`)
  return `chunk-${chunkId}-result`
})

// Concurrency report
const generateConcurrencyReport = Effect.gen(function* () {
  const operations = yield* TestServices.get(concurrentOperations)
  const supervisedFibers = yield* TestServices.supervisedFibers()
  const tags = yield* TestServices.get(TestAnnotation.tagged)
  
  return {
    concurrentOperations: operations,
    activeFibers: supervisedFibers.size,
    tags: Array.from(tags),
    concurrencyLevel: operations > 5 ? "high" : operations > 2 ? "medium" : "low"
  }
})
```

## Advanced Features Deep Dive

### Custom Annotation Types with Complex Combining

```typescript
import { Equal, Hash } from "effect"

// Complex annotation for test execution statistics
interface TestStats extends Equal.Equal {
  readonly assertions: number
  readonly errors: string[]
  readonly warnings: string[]
  readonly duration: number
}

const TestStats = {
  make: (stats: Omit<TestStats, typeof Equal.symbol | typeof Hash.symbol>): TestStats => ({
    ...stats,
    [Equal.symbol](that: unknown): boolean {
      return TestStats.isTestStats(that) &&
        this.assertions === that.assertions &&
        Equal.equals(this.errors, that.errors) &&
        Equal.equals(this.warnings, that.warnings) &&
        this.duration === that.duration
    },
    [Hash.symbol](): number {
      return Hash.combine(
        Hash.hash(stats.assertions),
        Hash.combine(Hash.hash(stats.errors), Hash.hash(stats.warnings))
      )
    }
  }),
  
  isTestStats: (u: unknown): u is TestStats =>
    typeof u === "object" && u !== null && Equal.symbol in u,
  
  empty: {
    assertions: 0,
    errors: [],
    warnings: [],
    duration: 0,
    [Equal.symbol](that: unknown): boolean {
      return TestStats.isTestStats(that) && 
        that.assertions === 0 && 
        that.errors.length === 0 && 
        that.warnings.length === 0 && 
        that.duration === 0
    },
    [Hash.symbol](): number {
      return Hash.hash("empty-test-stats")
    }
  } as TestStats,
  
  combine: (a: TestStats, b: TestStats): TestStats => TestStats.make({
    assertions: a.assertions + b.assertions,
    errors: [...a.errors, ...b.errors],
    warnings: [...a.warnings, ...b.warnings],
    duration: Math.max(a.duration, b.duration)
  })
}

const testStatsAnnotation = TestAnnotation.make(
  "test_stats",
  TestStats.empty,
  TestStats.combine
)

// Advanced usage with complex annotations
const complexTest = Effect.gen(function* () {
  yield* TestServices.annotate(
    testStatsAnnotation,
    TestStats.make({
      assertions: 5,
      errors: ["network timeout"],
      warnings: ["deprecated API used"],
      duration: 1500
    })
  )
  
  // Simulate more test operations
  yield* TestServices.annotate(
    testStatsAnnotation,
    TestStats.make({
      assertions: 3,
      errors: [],
      warnings: ["slow query"],
      duration: 800
    })
  )
  
  const finalStats = yield* TestServices.get(testStatsAnnotation)
  
  return {
    totalAssertions: finalStats.assertions,
    allErrors: finalStats.errors,
    allWarnings: finalStats.warnings,
    maxDuration: finalStats.duration
  }
})
```

### Real-World Advanced: Test Environment Annotations

```typescript
// Environment-specific annotations
const testEnvironment = TestAnnotation.make(
  "test_environment",
  {
    platform: "unknown" as "node" | "browser" | "react-native" | "unknown",
    version: "unknown",
    ci: false,
    features: [] as string[]
  },
  (a, b) => ({
    platform: b.platform !== "unknown" ? b.platform : a.platform,
    version: b.version !== "unknown" ? b.version : a.version,
    ci: a.ci || b.ci,
    features: [...new Set([...a.features, ...b.features])]
  })
)

const resourceUsage = TestAnnotation.make(
  "resource_usage", 
  { cpu: 0, memory: 0, network: 0 },
  (a, b) => ({
    cpu: Math.max(a.cpu, b.cpu),
    memory: Math.max(a.memory, b.memory), 
    network: a.network + b.network
  })
)

// Environment-aware test setup
const environmentAwareTest = Effect.gen(function* () {
  // Detect and annotate environment
  const platform = typeof window !== "undefined" ? "browser" : "node"
  const isCI = process.env.CI === "true"
  
  yield* TestServices.annotate(testEnvironment, {
    platform,
    version: process.version || "unknown",
    ci: isCI,
    features: ["async", "promises", "modules"]
  })
  
  // Different behavior based on environment
  if (platform === "browser") {
    yield* TestServices.annotate(
      TestAnnotation.tagged,
      HashSet.make("browser", "dom")
    )
    yield* testDOMInteraction()
  } else {
    yield* TestServices.annotate(
      TestAnnotation.tagged,
      HashSet.make("node", "filesystem")
    )
    yield* testFileSystem()
  }
  
  // Track resource usage
  yield* TestServices.annotate(resourceUsage, {
    cpu: 45, // percentage
    memory: 128, // MB
    network: 2048 // bytes
  })
})

const testDOMInteraction = Effect.succeed("DOM test completed")
const testFileSystem = Effect.succeed("File system test completed")
```

## Practical Patterns & Best Practices

### Pattern 1: Test Classification and Routing

```typescript
// Helper for test classification
const classifyTest = (
  category: "unit" | "integration" | "e2e",
  priority: "low" | "medium" | "high" | "critical",
  tags: string[]
) => Effect.gen(function* () {
  yield* TestServices.annotate(
    TestAnnotation.tagged,
    HashSet.fromIterable([category, priority, ...tags])
  )
  
  // Add category-specific annotations
  if (category === "e2e") {
    yield* TestServices.annotate(integrationScope, "e2e")
    yield* TestServices.annotate(serviceAnnotation, ["all"])
  }
  
  if (priority === "critical") {
    yield* TestServices.annotate(
      TestAnnotation.tagged,
      HashSet.make("smoke-test", "blocking")
    )
  }
})

// Smart test runner helper
const createTestRunner = <A, E>(
  testName: string,
  test: Effect.Effect<A, E>,
  classification: {
    category: "unit" | "integration" | "e2e"
    priority: "low" | "medium" | "high" | "critical"
    tags: string[]
    timeout?: number
    retries?: number
  }
) => Effect.gen(function* () {
  // Apply classification
  yield* classifyTest(classification.category, classification.priority, classification.tags)
  
  // Configure based on classification
  const effectWithTimeout = classification.timeout
    ? Effect.timeout(test, `${classification.timeout} millis`)
    : test
  
  // Add retry logic for flaky tests
  const effectWithRetries = classification.retries
    ? Effect.retry(effectWithTimeout, { times: classification.retries })
    : effectWithTimeout
  
  // Execute with error tracking
  const result = yield* Effect.either(effectWithRetries)
  
  if (Either.isLeft(result)) {
    yield* TestServices.annotate(
      TestAnnotation.tagged,
      HashSet.make("failed", "needs-investigation")
    )
  }
  
  return result
})
```

### Pattern 2: Annotation-Based Test Selection

```typescript
// Test filter based on annotations
const shouldRunTest = (
  environment: "development" | "ci" | "production",
  availableTime: "short" | "medium" | "long"
) => Effect.gen(function* () {
  const tags = yield* TestServices.get(TestAnnotation.tagged)
  const scope = yield* TestServices.get(integrationScope)
  
  // Environment-based filtering
  if (environment === "ci" && tags.has("manual-only")) {
    return false
  }
  
  if (environment === "production" && scope === "e2e") {
    return false
  }
  
  // Time-based filtering
  if (availableTime === "short" && tags.has("slow")) {
    return false
  }
  
  if (availableTime === "short" && scope === "e2e") {
    return false
  }
  
  return true
})

// Conditional test execution
const conditionalTest = <A, E>(
  test: Effect.Effect<A, E>,
  condition: Effect.Effect<boolean>
) => Effect.gen(function* () {
  const shouldRun = yield* condition
  
  if (!shouldRun) {
    yield* TestServices.annotate(TestAnnotation.ignored, 1)
    yield* TestServices.annotate(
      TestAnnotation.tagged,
      HashSet.make("skipped", "conditional")
    )
    return undefined as A
  }
  
  return yield* test
})
```

### Pattern 3: Comprehensive Test Reporting

```typescript
// Complete test report generator
const generateCompleteTestReport = Effect.gen(function* () {
  // Gather all annotation data
  const tags = yield* TestServices.get(TestAnnotation.tagged)
  const retries = yield* TestServices.get(TestAnnotation.retried)
  const repeats = yield* TestServices.get(TestAnnotation.repeated)
  const ignored = yield* TestServices.get(TestAnnotation.ignored)
  const supervisedFibers = yield* TestServices.supervisedFibers()
  
  // Custom annotations
  const executionTime = yield* TestServices.get(executionTime).pipe(
    Effect.orElse(() => Effect.succeed(0))
  )
  const memoryUsage = yield* TestServices.get(memoryUsage).pipe(
    Effect.orElse(() => Effect.succeed(0))
  )
  
  // Categorize test results
  const categories = Array.from(tags).filter(tag => 
    ["unit", "integration", "e2e"].includes(tag)
  )
  
  const priorities = Array.from(tags).filter(tag =>
    ["low", "medium", "high", "critical"].includes(tag)
  )
  
  const status = tags.has("failed") ? "failed" : 
                tags.has("skipped") ? "skipped" : "passed"
  
  return {
    status,
    categories: categories.length > 0 ? categories : ["uncategorized"],
    priorities: priorities.length > 0 ? priorities : ["medium"],
    tags: Array.from(tags),
    metrics: {
      retries,
      repeats,
      ignored,
      executionTime: `${executionTime}ms`,
      memoryUsage: `${memoryUsage}MB`,
      activeFibers: supervisedFibers.size
    },
    timestamp: new Date().toISOString(),
    reliability: retries > 2 ? "flaky" : retries > 0 ? "unstable" : "stable"
  }
})

// Batch report for multiple tests
const generateBatchReport = (testResults: Array<Awaited<ReturnType<typeof generateCompleteTestReport>>>) => {
  const summary = testResults.reduce((acc, result) => ({
    total: acc.total + 1,
    passed: acc.passed + (result.status === "passed" ? 1 : 0),
    failed: acc.failed + (result.status === "failed" ? 1 : 0),
    skipped: acc.skipped + (result.status === "skipped" ? 1 : 0),
    totalRetries: acc.totalRetries + result.metrics.retries,
    flakyTests: acc.flakyTests + (result.reliability === "flaky" ? 1 : 0)
  }), {
    total: 0,
    passed: 0,
    failed: 0,
    skipped: 0,
    totalRetries: 0,
    flakyTests: 0
  })
  
  return {
    summary,
    successRate: `${Math.round((summary.passed / summary.total) * 100)}%`,
    reliability: summary.flakyTests / summary.total < 0.1 ? "excellent" : 
                summary.flakyTests / summary.total < 0.25 ? "good" : "needs-improvement",
    testResults
  }
}
```

## Integration Examples

### Integration with Vitest Testing Framework

```typescript
import { describe, it, expect, beforeEach, afterEach } from "vitest"
import { Effect, TestServices, TestAnnotation, Layer } from "effect"

// Vitest integration helper
const runEffectTest = async <A>(
  effect: Effect.Effect<A, any, any>,
  timeout = 5000
): Promise<A> => {
  return await Effect.runPromise(
    effect.pipe(
      Effect.provide(TestServices.liveServices),
      Effect.timeout(`${timeout} millis`)
    )
  )
}

describe("User Service", () => {
  let testReport: any
  
  beforeEach(async () => {
    // Initialize test annotations for each test
    testReport = null
  })
  
  afterEach(async () => {
    // Generate report after each test
    testReport = await runEffectTest(generateCompleteTestReport())
    console.log("Test Report:", JSON.stringify(testReport, null, 2))
  })
  
  it("should create user successfully", async () => {
    const result = await runEffectTest(
      Effect.gen(function* () {
        yield* classifyTest("unit", "high", ["user-creation", "database"])
        
        // Your test logic
        const user = yield* createUser({ name: "John", email: "john@example.com" })
        
        expect(user.id).toBeDefined()
        expect(user.name).toBe("John")
        
        return user
      })
    )
    
    expect(result.name).toBe("John")
  })
  
  it("should handle user creation failure", async () => {
    await expect(
      runEffectTest(
        Effect.gen(function* () {
          yield* classifyTest("unit", "medium", ["error-handling", "validation"])
          
          // Test error case
          return yield* createUser({ name: "", email: "invalid" })
        })
      )
    ).rejects.toThrow()
  })
})

// Mock user creation for testing
const createUser = (data: { name: string; email: string }) =>
  Effect.gen(function* () {
    if (!data.name || !data.email.includes("@")) {
      return yield* Effect.fail(new Error("Invalid user data"))
    }
    
    yield* Effect.sleep("10 millis") // Simulate async operation
    return {
      id: Math.random().toString(),
      name: data.name,
      email: data.email,
      createdAt: new Date()
    }
  })
```

### Integration with Jest and Custom Matchers

```typescript
import { Effect, TestServices, TestAnnotation } from "effect"

// Custom Jest matchers for Effect testing
declare global {
  namespace jest {
    interface Matchers<R> {
      toHaveAnnotation<A>(annotation: TestAnnotation.TestAnnotation<A>, expected: A): R
      toBeTaggedWith(tags: string[]): R
      toHaveExecutedWithin(maxTime: number): R
    }
  }
}

// Extend Jest matchers
expect.extend({
  toHaveAnnotation<A>(
    effectTest: Effect.Effect<any>,
    annotation: TestAnnotation.TestAnnotation<A>,
    expected: A
  ) {
    return Effect.runSync(
      Effect.gen(function* () {
        yield* effectTest
        const actual = yield* TestServices.get(annotation)
        
        const pass = Equal.equals(actual, expected)
        
        return {
          pass,
          message: () => pass
            ? `Expected annotation ${annotation.identifier} not to equal ${expected}`
            : `Expected annotation ${annotation.identifier} to equal ${expected}, got ${actual}`
        }
      }).pipe(Effect.provide(TestServices.liveServices))
    )
  },
  
  toBeTaggedWith(effectTest: Effect.Effect<any>, expectedTags: string[]) {
    return Effect.runSync(
      Effect.gen(function* () {
        yield* effectTest
        const actualTags = yield* TestServices.get(TestAnnotation.tagged)
        
        const hasAllTags = expectedTags.every(tag => actualTags.has(tag))
        
        return {
          pass: hasAllTags,
          message: () => hasAllTags
            ? `Expected test not to have tags ${expectedTags.join(", ")}`
            : `Expected test to have tags ${expectedTags.join(", ")}, got ${Array.from(actualTags).join(", ")}`
        }
      }).pipe(Effect.provide(TestServices.liveServices))
    )
  }
})

// Usage with Jest
describe("Effect TestAnnotations with Jest", () => {
  it("should properly annotate performance tests", async () => {
    const performanceTest = Effect.gen(function* () {
      yield* TestServices.annotate(
        TestAnnotation.tagged,
        HashSet.make("performance", "critical")
      )
      
      yield* TestServices.annotate(executionTime, 150)
      
      // Simulate test work
      yield* Effect.sleep("100 millis")
      
      return "performance test completed"
    })
    
    await expect(performanceTest).toBeTaggedWith(["performance", "critical"])
    await expect(performanceTest).toHaveAnnotation(executionTime, 150)
  })
})
```

### Integration with Test Runners and CI/CD

```typescript
// CI/CD integration helpers
const createCITestSuite = (
  environment: "development" | "staging" | "production"
) => Effect.gen(function* () {
  // Configure test environment
  yield* TestServices.annotate(testEnvironment, {
    platform: "node",
    version: process.version,
    ci: true,
    features: ["ci", "automated", environment]
  })
  
  // Set appropriate tags for CI
  yield* TestServices.annotate(
    TestAnnotation.tagged,
    HashSet.make("ci", "automated", environment)
  )
  
  // Configure retry strategy for CI
  const retryStrategy = environment === "production" ? 3 : 1
  
  return { retryStrategy, environment }
})

// Test result aggregator for CI reporting
const aggregateTestResults = (results: Array<any>) => Effect.gen(function* () {
  const summary = results.reduce(
    (acc, result) => ({
      total: acc.total + 1,
      passed: acc.passed + (result.status === "passed" ? 1 : 0),
      failed: acc.failed + (result.status === "failed" ? 1 : 0),
      duration: acc.duration + (result.metrics?.executionTime || 0)
    }),
    { total: 0, passed: 0, failed: 0, duration: 0 }
  )
  
  // Generate CI-friendly output
  const ciReport = {
    testResults: {
      numTotalTests: summary.total,
      numPassedTests: summary.passed,
      numFailedTests: summary.failed,
      testResults: results
    },
    success: summary.failed === 0,
    coverageMap: {}, // Would integrate with coverage tools
    numTotalTestSuites: 1,
    numPassedTestSuites: summary.failed === 0 ? 1 : 0,
    numFailedTestSuites: summary.failed > 0 ? 1 : 0
  }
  
  // Output for CI systems
  if (process.env.CI) {
    console.log("::group::Test Results")
    console.log(JSON.stringify(ciReport, null, 2))
    console.log("::endgroup::")
  }
  
  return ciReport
})

// GitHub Actions integration
const createGitHubActionsOutput = (report: any) => Effect.gen(function* () {
  const outputs = [
    `tests-total=${report.testResults.numTotalTests}`,
    `tests-passed=${report.testResults.numPassedTests}`,
    `tests-failed=${report.testResults.numFailedTests}`,
    `success=${report.success}`
  ]
  
  // Set GitHub Actions outputs
  for (const output of outputs) {
    console.log(`::set-output name=${output}`)
  }
  
  // Create test summary
  const summary = `
## Test Results Summary
- **Total Tests**: ${report.testResults.numTotalTests}
- **Passed**: ${report.testResults.numPassedTests}
- **Failed**: ${report.testResults.numFailedTests}
- **Success Rate**: ${Math.round((report.testResults.numPassedTests / report.testResults.numTotalTests) * 100)}%
  `
  
  console.log(`::notice title=Test Summary::${summary}`)
  
  return summary
})
```

## Conclusion

TestAnnotations provides comprehensive test metadata management, structured error tracking, and performance monitoring for Effect applications.

Key benefits:
- **Type-Safe Metadata**: Compile-time guarantees for annotation structure and operations
- **Composable Architecture**: Natural integration with Effect's service and layer system  
- **Automatic Aggregation**: Built-in combining logic handles complex annotation merging across test execution
- **Rich Ecosystem**: Seamless integration with popular testing frameworks and CI/CD systems

TestAnnotations is essential for applications requiring comprehensive test observability, detailed performance tracking, complex test classification systems, or integration with sophisticated testing and monitoring infrastructure.