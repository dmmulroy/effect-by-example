# TestSized: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem TestSized Solves

Property-based testing and data generation often need to control the size complexity of generated test data, but traditional approaches lack sophisticated size management:

```typescript
// Traditional approach - hard-coded test data sizes
describe("list operations", () => {
  it("should handle small lists", () => {
    const smallList = [1, 2, 3]
    expect(processItems(smallList)).toBeDefined()
  })
  
  it("should handle large lists", () => {
    const largeList = Array.from({ length: 1000 }, (_, i) => i)
    expect(processItems(largeList)).toBeDefined()
    // But what about different sizes? Edge cases? Performance boundaries?
  })
})
```

This approach leads to:
- **Fixed Size Testing** - Only tests specific, predetermined sizes
- **Missing Size Boundaries** - Fails to test critical size thresholds (0, 1, 100, 1000+)
- **Performance Blind Spots** - Doesn't validate performance characteristics across size ranges
- **Poor Size Scaling** - No systematic way to control complexity in generated test data

### The TestSized Solution

TestSized provides sophisticated size control for property-based testing and data generation:

```typescript
import { Effect, TestSized, TestServices, Random, Arbitrary, FastCheck } from "effect"

// Control test data size dynamically
const sizedListTest = Effect.gen(function* () {
  const currentSize = yield* TestServices.size
  const items = yield* generateItemsOfSize(currentSize)
  const result = yield* processItems(items)
  
  // Size-aware assertions
  if (currentSize === 0) {
    assert(result.isEmpty)
  } else if (currentSize < 10) {
    assert(result.length <= currentSize * 2)
  } else {
    assert(result.processingTime < currentSize * 10)
  }
}).pipe(
  TestServices.withSize(50) // Run with size 50
)
```

### Key Concepts

**Size Parameter**: A numeric value that controls the complexity/scale of generated test data

**Size Context**: TestSized provides size as ambient context throughout test execution

**Size Scaling**: Generators and operations can scale their behavior based on current size

## Basic Usage Patterns

### Pattern 1: Getting Current Size

```typescript
import { Effect, TestServices } from "effect"

// Access the current test size
const checkCurrentSize = Effect.gen(function* () {
  const size = yield* TestServices.size
  console.log(`Current test size: ${size}`)
  return size
})

// Default size is 100 (configured in TestServices.liveServices)
Effect.runSync(checkCurrentSize.pipe(
  Effect.provide(TestServices.liveServices)
))
// Output: Current test size: 100
```

### Pattern 2: Running Tests with Specific Size

```typescript
import { Effect, TestServices } from "effect"

const sizeDependentTest = Effect.gen(function* () {
  const size = yield* TestServices.size
  
  // Generate data based on size
  const data = Array.from({ length: size }, (_, i) => i)
  const result = yield* processData(data)
  
  return { size, dataLength: data.length, result }
})

// Run with custom size
const testResult = Effect.runSync(
  sizeDependentTest.pipe(
    TestServices.withSize(25),
    Effect.provide(TestServices.liveServices)
  )
)
// testResult: { size: 25, dataLength: 25, result: ... }
```

### Pattern 3: Size-Aware Data Generation

```typescript
import { Effect, TestServices, Random } from "effect"

const generateSizedData = Effect.gen(function* () {
  const size = yield* TestServices.size
  const random = yield* Random.Random
  
  // Generate arrays based on current size
  const arrayLength = Math.max(1, Math.floor(size / 10))
  const items = []
  
  for (let i = 0; i < arrayLength; i++) {
    const value = yield* random.nextIntBetween(0, size)
    items.push(value)
  }
  
  return items
})

// Generate data with different sizes
const smallData = Effect.runSync(
  generateSizedData.pipe(
    TestServices.withSize(10),
    Effect.provide(TestServices.liveServices)
  )
)
// smallData: [1] (length 1, values 0-10)

const largeData = Effect.runSync(
  generateSizedData.pipe(
    TestServices.withSize(500),
    Effect.provide(TestServices.liveServices)
  )
)
// largeData: [234, 456, 123, 789, 12] (length 50, values 0-500)
```

## Real-World Examples

### Example 1: Database Performance Testing

Testing database operations across different data scales:

```typescript
import { Effect, TestServices, Random, Array as Arr } from "effect"

interface User {
  readonly id: number
  readonly name: string
  readonly email: string
  readonly createdAt: Date
}

interface DatabaseService {
  readonly insertUsers: (users: ReadonlyArray<User>) => Effect.Effect<void>
  readonly queryUsers: (limit?: number) => Effect.Effect<ReadonlyArray<User>>
  readonly deleteAllUsers: Effect.Effect<void>
}

const DatabaseService = Context.GenericTag<DatabaseService>("DatabaseService")

// Size-aware user generation
const generateUsers = Effect.gen(function* () {
  const size = yield* TestServices.size
  const random = yield* Random.Random
  const userCount = Math.max(1, Math.floor(size / 5)) // Scale user count with size
  
  const users: User[] = []
  for (let i = 0; i < userCount; i++) {
    const id = yield* random.nextIntBetween(1, size * 10)
    const nameLength = Math.max(3, Math.floor(size / 20))
    const name = Array.from({ length: nameLength }, () => 
      String.fromCharCode(97 + Math.floor(Math.random() * 26))
    ).join('')
    
    users.push({
      id,
      name,
      email: `${name}@example.com`,
      createdAt: new Date()
    })
  }
  
  return users
})

// Database performance test with size scaling
const databasePerformanceTest = Effect.gen(function* () {
  const db = yield* DatabaseService
  const users = yield* generateUsers
  const startTime = Date.now()
  
  // Insert users
  yield* db.insertUsers(users)
  const insertTime = Date.now() - startTime
  
  // Query users
  const queryStart = Date.now()
  const queriedUsers = yield* db.queryUsers()
  const queryTime = Date.now() - queryStart
  
  // Cleanup
  yield* db.deleteAllUsers
  
  return {
    userCount: users.length,
    insertTime,
    queryTime,
    totalTime: insertTime + queryTime
  }
})

// Test with different scales
const runScalabilityTests = Effect.gen(function* () {
  const sizes = [10, 50, 100, 500, 1000]
  const results = []
  
  for (const size of sizes) {
    const result = yield* databasePerformanceTest.pipe(
      TestServices.withSize(size)
    )
    results.push({ size, ...result })
  }
  
  return results
}).pipe(
  Effect.provide(TestServices.liveServices),
  Effect.provide(mockDatabaseLayer)
)

// Results show performance scaling:
// [
//   { size: 10, userCount: 2, insertTime: 5, queryTime: 2, totalTime: 7 },
//   { size: 50, userCount: 10, insertTime: 15, queryTime: 8, totalTime: 23 },
//   { size: 100, userCount: 20, insertTime: 35, queryTime: 18, totalTime: 53 },
//   ...
// ]
```

### Example 2: API Load Testing with Size-Based Scaling

Testing API endpoints with increasing request volumes:

```typescript
import { Effect, TestServices, Array as Arr, Duration, Schedule } from "effect"

interface ApiClient {
  readonly makeRequest: (endpoint: string, payload: unknown) => Effect.Effect<Response>
}

const ApiClient = Context.GenericTag<ApiClient>("ApiClient")

// Size-aware request generation
const generateApiRequests = Effect.gen(function* () {
  const size = yield* TestServices.size
  const requestCount = Math.min(size, 1000) // Cap at 1000 requests
  const batchSize = Math.max(1, Math.floor(size / 10))
  
  const requests = Arr.range(0, requestCount - 1).map(i => ({
    endpoint: `/api/users/${i}`,
    payload: { 
      data: `request-${i}`,
      timestamp: Date.now(),
      size: Math.floor(size / 100) // Payload size scales with test size
    }
  }))
  
  return { requests, batchSize }
})

// Load test with concurrent batches
const apiLoadTest = Effect.gen(function* () {
  const client = yield* ApiClient
  const { requests, batchSize } = yield* generateApiRequests
  const startTime = Date.now()
  
  // Process requests in batches
  const batches = Arr.chunksOf(requests, batchSize)
  let successCount = 0
  let errorCount = 0
  
  for (const batch of batches) {
    const batchResults = yield* Effect.allWith(
      batch.map(req => 
        client.makeRequest(req.endpoint, req.payload).pipe(
          Effect.map(() => "success" as const),
          Effect.catchAll(() => Effect.succeed("error" as const))
        )
      ),
      { concurrency: "unbounded" }
    )
    
    successCount += batchResults.filter(r => r === "success").length
    errorCount += batchResults.filter(r => r === "error").length
  }
  
  const totalTime = Date.now() - startTime
  
  return {
    totalRequests: requests.length,
    batchSize,
    successCount,
    errorCount,
    totalTime,
    requestsPerSecond: requests.length / (totalTime / 1000)
  }
})

// Progressive load testing
const progressiveLoadTest = Effect.gen(function* () {
  const testSizes = [5, 25, 50, 100, 250, 500]
  const results = []
  
  for (const size of testSizes) {
    console.log(`Running load test with size: ${size}`)
    
    const result = yield* apiLoadTest.pipe(
      TestServices.withSize(size),
      Effect.timeout(Duration.seconds(30))
    )
    
    results.push({ size, ...result })
    
    // Brief pause between tests
    yield* Effect.sleep(Duration.seconds(1))
  }
  
  return results
}).pipe(
  Effect.provide(TestServices.liveServices),
  Effect.provide(mockApiClientLayer)
)
```

### Example 3: Memory Usage Testing with Size Control

Testing memory consumption patterns across different data sizes:

```typescript
import { Effect, TestServices, Ref, Array as Arr } from "effect"

// Memory-intensive data structure
class DataProcessor {
  private cache = new Map<string, unknown>()
  private buffer: unknown[] = []
  
  constructor(private maxSize: number) {}
  
  addItem(key: string, data: unknown): void {
    if (this.cache.size < this.maxSize) {
      this.cache.set(key, data)
      this.buffer.push(data)
    }
  }
  
  process(): unknown[] {
    const result = Array.from(this.cache.values())
    return result.concat(this.buffer)
  }
  
  getMemoryUsage(): number {
    return this.cache.size + this.buffer.length
  }
  
  clear(): void {
    this.cache.clear()
    this.buffer.length = 0
  }
}

// Size-based memory testing
const memoryUsageTest = Effect.gen(function* () {
  const size = yield* TestServices.size
  const processor = new DataProcessor(size * 10)
  
  // Generate data proportional to size
  const itemCount = size * 2
  const itemSize = Math.max(10, size / 10) // Each item's complexity
  
  const startMemory = process.memoryUsage().heapUsed
  
  // Add items to processor
  for (let i = 0; i < itemCount; i++) {
    const data = {
      id: i,
      payload: Array(itemSize).fill(0).map(() => Math.random()),
      metadata: {
        timestamp: Date.now(),
        size: itemSize,
        index: i
      }
    }
    processor.addItem(`item-${i}`, data)
  }
  
  // Process data
  const results = processor.process()
  const processorMemory = processor.getMemoryUsage()
  const endMemory = process.memoryUsage().heapUsed
  const memoryDelta = endMemory - startMemory
  
  // Cleanup
  processor.clear()
  
  return {
    testSize: size,
    itemCount,
    itemSize,
    resultsLength: results.length,
    processorMemory,
    memoryDeltaKB: Math.round(memoryDelta / 1024),
    memoryPerItem: Math.round(memoryDelta / itemCount)
  }
})

// Memory scaling analysis
const memoryScalingAnalysis = Effect.gen(function* () {
  const sizes = [10, 25, 50, 100, 250, 500]
  const results = []
  
  for (const size of sizes) {
    // Force garbage collection if available
    if (global.gc) global.gc()
    
    const result = yield* memoryUsageTest.pipe(
      TestServices.withSize(size)
    )
    
    results.push(result)
    
    // Log immediate results
    console.log(
      `Size ${size}: ${result.itemCount} items, ` +
      `${result.memoryDeltaKB}KB total, ` +
      `${result.memoryPerItem} bytes/item`
    )
  }
  
  // Calculate memory growth patterns
  const growthAnalysis = results.map((result, index) => {
    if (index === 0) return { ...result, growthFactor: 1 }
    
    const previous = results[index - 1]
    const growthFactor = result.memoryDeltaKB / previous.memoryDeltaKB
    
    return { ...result, growthFactor }
  })
  
  return growthAnalysis
}).pipe(
  Effect.provide(TestServices.liveServices)
)

// Example output:
// [
//   { testSize: 10, itemCount: 20, memoryDeltaKB: 45, memoryPerItem: 2304, growthFactor: 1 },
//   { testSize: 25, itemCount: 50, memoryDeltaKB: 112, memoryPerItem: 2304, growthFactor: 2.49 },
//   { testSize: 50, itemCount: 100, memoryDeltaKB: 224, memoryPerItem: 2304, growthFactor: 2.0 },
//   ...
// ]
```

## Advanced Features Deep Dive

### Feature 1: Dynamic Size Manipulation

Advanced size control during test execution:

```typescript
import { Effect, TestServices, Ref } from "effect"

// Dynamic size adjustment during test execution
const dynamicSizeTest = Effect.gen(function* () {
  const results = yield* Ref.make<Array<{ size: number; result: unknown }>>([])
  
  // Start with small size
  let currentResult = yield* someOperation().pipe(
    TestServices.withSize(10)
  )
  
  yield* Ref.update(results, prev => [...prev, { size: 10, result: currentResult }])
  
  // Gradually increase size based on previous results
  for (const size of [25, 50, 100, 200]) {
    currentResult = yield* someOperation().pipe(
      TestServices.withSize(size)
    )
    
    yield* Ref.update(results, prev => [...prev, { size, result: currentResult }])
    
    // Early termination if performance degrades
    if (shouldStopTesting(currentResult)) {
      break
    }
  }
  
  return yield* Ref.get(results)
})

const shouldStopTesting = (result: unknown): boolean => {
  // Implementation-specific logic
  return false
}

const someOperation = (): Effect.Effect<unknown> =>
  Effect.succeed("operation result")
```

### Feature 2: Size-Aware Property Testing Integration

Combining TestSized with FastCheck for sophisticated property testing:

```typescript
import { Effect, TestServices, FastCheck, Arbitrary, Schema } from "effect"

// Schema that scales with size
const createSizedSchema = (size: number) => Schema.Struct({
  id: Schema.Number,
  items: Schema.Array(
    Schema.String.pipe(
      Schema.minLength(1),
      Schema.maxLength(Math.max(5, size / 10))
    )
  ).pipe(
    Schema.minItems(0),
    Schema.maxItems(Math.max(1, size / 5))
  ),
  metadata: Schema.Record({
    key: Schema.String,
    value: Schema.Union(Schema.String, Schema.Number)
  }).pipe(
    Schema.maxEntries(Math.max(1, size / 20))
  )
})

// Size-aware property testing
const sizedPropertyTest = Effect.gen(function* () {
  const size = yield* TestServices.size
  const schema = createSizedSchema(size)
  const arbitrary = Arbitrary.make(schema)
  
  const property = FastCheck.property(arbitrary, (data) => {
    // Properties that should hold regardless of size
    return (
      data.items.length <= size / 5 &&
      data.items.every(item => item.length <= size / 10) &&
      Object.keys(data.metadata).length <= size / 20
    )
  })
  
  // Run property test with size-appropriate sample count
  const sampleCount = Math.min(100, Math.max(10, size))
  
  return yield* Effect.sync(() => 
    FastCheck.assert(property, { numRuns: sampleCount })
  )
})

// Multi-size property validation
const multiSizePropertyValidation = Effect.gen(function* () {
  const sizes = [5, 20, 50, 100, 500]
  const results = []
  
  for (const size of sizes) {
    const startTime = Date.now()
    
    yield* sizedPropertyTest.pipe(
      TestServices.withSize(size)
    )
    
    const duration = Date.now() - startTime
    results.push({ size, duration, status: "passed" })
  }
  
  return results
}).pipe(
  Effect.provide(TestServices.liveServices),
  Effect.catchAll(error => 
    Effect.succeed([{ error: error.toString(), status: "failed" }])
  )
)
```

### Feature 3: Custom Size Strategies

Implementing custom size progression strategies:

```typescript
import { Effect, TestServices, Array as Arr } from "effect"

// Size progression strategies
const SizeStrategy = {
  linear: (start: number, end: number, steps: number): number[] =>
    Arr.range(0, steps - 1).map(i => 
      start + Math.floor((end - start) * i / (steps - 1))
    ),
  
  exponential: (start: number, end: number, steps: number): number[] =>
    Arr.range(0, steps - 1).map(i => 
      Math.floor(start * Math.pow(end / start, i / (steps - 1)))
    ),
  
  fibonacci: (maxSize: number): number[] => {
    const sizes = [1, 1]
    while (sizes[sizes.length - 1] < maxSize) {
      const next = sizes[sizes.length - 1] + sizes[sizes.length - 2]
      if (next > maxSize) break
      sizes.push(next)
    }
    return sizes.slice(2) // Remove initial 1, 1
  }
}

// Apply size strategy to tests
const runWithSizeStrategy = <A, E, R>(
  test: Effect.Effect<A, E, R>,
  strategy: number[]
) => Effect.gen(function* () {
  const results = []
  
  for (const size of strategy) {
    const result = yield* test.pipe(
      TestServices.withSize(size),
      Effect.map(result => ({ size, result })),
      Effect.catchAll(error => Effect.succeed({ size, error }))
    )
    
    results.push(result)
  }
  
  return results
})

// Example usage with different strategies
const performanceTest = Effect.gen(function* () {
  const size = yield* TestServices.size
  
  // Simulate some work that scales with size
  const work = Array.from({ length: size }, (_, i) => i * 2)
  const result = work.reduce((sum, val) => sum + val, 0)
  
  return { processedItems: work.length, sum: result }
})

const compareStrategies = Effect.gen(function* () {
  const linearResults = yield* runWithSizeStrategy(
    performanceTest,
    SizeStrategy.linear(10, 1000, 10)
  )
  
  const exponentialResults = yield* runWithSizeStrategy(
    performanceTest,
    SizeStrategy.exponential(10, 1000, 10)
  )
  
  const fibonacciResults = yield* runWithSizeStrategy(
    performanceTest,
    SizeStrategy.fibonacci(1000)
  )
  
  return {
    linear: linearResults,
    exponential: exponentialResults,
    fibonacci: fibonacciResults
  }
}).pipe(
  Effect.provide(TestServices.liveServices)
)
```

## Practical Patterns & Best Practices

### Pattern 1: Size-Aware Test Helpers

Create reusable helpers that adapt to test size:

```typescript
import { Effect, TestServices, Random, Array as Arr } from "effect"

// Helper: Generate size-appropriate test data
const generateTestData = <A>(
  generator: (size: number) => Effect.Effect<A>
) => Effect.gen(function* () {
  const size = yield* TestServices.size
  return yield* generator(size)
})

// Helper: Size-scaled string generation
const generateSizedString = generateTestData((size) => 
  Effect.gen(function* () {
    const length = Math.max(1, Math.floor(size / 10))
    const random = yield* Random.Random
    let result = ""
    
    for (let i = 0; i < length; i++) {
      const charCode = yield* random.nextIntBetween(97, 122) // a-z
      result += String.fromCharCode(charCode)
    }
    
    return result
  })
)

// Helper: Size-scaled array generation
const generateSizedArray = <A>(
  itemGenerator: Effect.Effect<A>
) => generateTestData((size) => 
  Effect.gen(function* () {
    const length = Math.max(1, Math.floor(size / 5))
    const items = []
    
    for (let i = 0; i < length; i++) {
      const item = yield* itemGenerator
      items.push(item)
    }
    
    return items
  })
)

// Helper: Performance assertion based on size
const assertPerformance = <A>(
  operation: Effect.Effect<A>,
  timeoutPerSizeUnit: number = 1 // ms per size unit
) => Effect.gen(function* () {
  const size = yield* TestServices.size
  const timeout = size * timeoutPerSizeUnit
  const startTime = Date.now()
  
  const result = yield* operation.pipe(
    Effect.timeout(Duration.millis(timeout))
  )
  
  const duration = Date.now() - startTime
  
  if (duration > timeout) {
    throw new Error(`Operation took ${duration}ms, expected < ${timeout}ms for size ${size}`)
  }
  
  return { result, duration, size, timeout }
})

// Usage example
const testWithSizedHelpers = Effect.gen(function* () {
  const testString = yield* generateSizedString
  const testArray = yield* generateSizedArray(Effect.succeed(Math.random()))
  
  const performanceResult = yield* assertPerformance(
    processData(testString, testArray),
    2 // 2ms per size unit
  )
  
  return {
    stringLength: testString.length,
    arrayLength: testArray.length,
    performance: performanceResult
  }
})
```

### Pattern 2: Size Boundaries Testing

Test critical size boundaries systematically:

```typescript
import { Effect, TestServices, Array as Arr } from "effect"

// Define critical size boundaries
const SizeBoundaries = {
  empty: 0,
  single: 1,
  small: 10,
  medium: 100,
  large: 1000,
  xlarge: 10000
} as const

// Test behavior at boundaries
const boundaryTest = <A, E>(
  testFn: Effect.Effect<A, E, TestServices>,
  boundaries: ReadonlyArray<number> = Object.values(SizeBoundaries)
) => Effect.gen(function* () {
  const results = []
  
  for (const size of boundaries) {
    const result = yield* testFn.pipe(
      TestServices.withSize(size),
      Effect.map(result => ({ boundary: size, result, status: "success" as const })),
      Effect.catchAll(error => Effect.succeed({ 
        boundary: size, 
        error: error.toString(), 
        status: "failed" as const 
      }))
    )
    
    results.push(result)
  }
  
  return results
})

// Boundary-aware data structure test
const dataStructureTest = Effect.gen(function* () {
  const size = yield* TestServices.size
  const data = Arr.range(0, size - 1)
  
  // Different behaviors at different boundaries
  if (size === 0) {
    return { behavior: "empty", items: [], processingTime: 0 }
  } else if (size === 1) {
    return { behavior: "single", items: data, processingTime: 1 }
  } else if (size <= 10) {
    return { behavior: "small_optimized", items: data, processingTime: size }
  } else if (size <= 100) {
    return { behavior: "medium_batch", items: data, processingTime: size * 1.5 }
  } else {
    return { behavior: "large_streaming", items: data.slice(0, 100), processingTime: size * 0.5 }
  }
})

// Run boundary tests
const runBoundaryTests = boundaryTest(dataStructureTest).pipe(
  Effect.provide(TestServices.liveServices)
)
```

### Pattern 3: Size-Based Test Configuration

Configure test behavior based on size context:

```typescript
import { Effect, TestServices, Context } from "effect"

// Size-based configuration
interface TestConfiguration {
  readonly timeout: number
  readonly retries: number
  readonly concurrency: number
  readonly samplingRate: number
}

const createSizeBasedConfig = (size: number): TestConfiguration => ({
  timeout: Math.max(1000, size * 10), // 10ms per size unit, min 1s
  retries: size < 50 ? 3 : size < 200 ? 2 : 1, // Fewer retries for larger tests
  concurrency: size < 100 ? 10 : size < 500 ? 5 : 2, // Less concurrency for larger tests
  samplingRate: size < 50 ? 1.0 : size < 200 ? 0.5 : 0.1 // Sample less for larger tests
})

const TestConfiguration = Context.GenericTag<TestConfiguration>("TestConfiguration")

// Size-aware test runner
const runSizeAwareTest = <A, E, R>(
  test: Effect.Effect<A, E, R & TestConfiguration>
) => Effect.gen(function* () {
  const size = yield* TestServices.size
  const config = createSizeBasedConfig(size)
  
  return yield* test.pipe(
    Effect.provide(Context.make(TestConfiguration, config)),
    Effect.timeout(Duration.millis(config.timeout))
  )
})

// Example test using size-based configuration
const configAwareTest = Effect.gen(function* () {
  const size = yield* TestServices.size
  const config = yield* TestConfiguration
  
  // Use configuration for test behavior
  const shouldSample = Math.random() < config.samplingRate
  if (!shouldSample && size > 100) {
    return { skipped: true, reason: "sampling" }
  }
  
  // Simulate concurrent operations
  const operations = Arr.range(0, Math.min(size, 100)).map(i =>
    Effect.delay(Effect.succeed(i * 2), Duration.millis(10))
  )
  
  const results = yield* Effect.allWith(operations, {
    concurrency: config.concurrency
  })
  
  return {
    size,
    config,
    resultsCount: results.length,
    skipped: false
  }
})

// Usage
const runConfiguredTest = runSizeAwareTest(configAwareTest).pipe(
  Effect.provide(TestServices.liveServices)
)
```

## Integration Examples

### Integration with Vitest and Property Testing

Combining TestSized with popular testing frameworks:

```typescript
import { Effect, TestServices, FastCheck, Arbitrary, Schema } from "effect"
import { describe, it, expect } from "vitest"

// Vitest integration helper
const runEffectTest = <A>(
  effect: Effect.Effect<A, never, TestServices>
): Promise<A> =>
  Effect.runPromise(effect.pipe(
    Effect.provide(TestServices.liveServices)
  ))

// Property-based test with size control
describe("Array operations with TestSized", () => {
  it("should handle arrays of different sizes", async () => {
    const testProperty = Effect.gen(function* () {
      const size = yield* TestServices.size
      
      // Create size-appropriate schema
      const arraySchema = Schema.Array(
        Schema.Number.pipe(Schema.int(), Schema.between(0, size))
      ).pipe(
        Schema.minItems(0),
        Schema.maxItems(Math.max(1, size / 10))
      )
      
      const arbitrary = Arbitrary.make(arraySchema)
      
      // Property: sorted array should have same length
      const property = FastCheck.property(arbitrary, (arr) => {
        const sorted = [...arr].sort((a, b) => a - b)
        return arr.length === sorted.length
      })
      
      return yield* Effect.sync(() => 
        FastCheck.assert(property, { numRuns: Math.min(100, size) })
      )
    })
    
    // Test with different sizes
    for (const size of [10, 50, 100]) {
      await runEffectTest(
        testProperty.pipe(TestServices.withSize(size))
      )
    }
  })
  
  it("should validate performance characteristics", async () => {
    const performanceTest = Effect.gen(function* () {
      const size = yield* TestServices.size
      const data = Array.from({ length: size }, (_, i) => i)
      
      const startTime = Date.now()
      const sorted = data.sort((a, b) => b - a) // Reverse sort
      const duration = Date.now() - startTime
      
      // Performance assertion based on size
      const maxExpectedTime = size * 0.01 // 0.01ms per item
      expect(duration).toBeLessThan(maxExpectedTime)
      
      return { size, duration, sorted: sorted.length }
    })
    
    const results = await runEffectTest(
      performanceTest.pipe(TestServices.withSize(10000))
    )
    
    expect(results.sorted).toBe(10000)
  })
})

// Custom test runner with size progression
const runSizeProgressionTest = <A>(
  testName: string,
  testEffect: Effect.Effect<A, never, TestServices>,
  sizes: number[] = [1, 10, 100, 1000]
) => {
  describe(testName, () => {
    sizes.forEach(size => {
      it(`should work with size ${size}`, async () => {
        const result = await runEffectTest(
          testEffect.pipe(TestServices.withSize(size))
        )
        expect(result).toBeDefined()
      })
    })
  })
}

// Usage of custom test runner
runSizeProgressionTest(
  "Data processing",
  Effect.gen(function* () {
    const size = yield* TestServices.size
    const processed = Array.from({ length: size }, (_, i) => i * 2)
    return processed.length
  })
)
```

### Integration with Database Testing

Using TestSized for database performance and scalability testing:

```typescript
import { Effect, TestServices, Layer, Context } from "effect"
import { Database } from "@effect/sql"

interface DatabaseTestService {
  readonly insertRecords: (count: number) => Effect.Effect<number>
  readonly queryRecords: (limit?: number) => Effect.Effect<Array<unknown>>
  readonly clearRecords: Effect.Effect<void>
}

const DatabaseTestService = Context.GenericTag<DatabaseTestService>("DatabaseTestService")

// Mock database implementation that scales with size
const mockDatabaseLayer = Layer.succeed(
  DatabaseTestService,
  DatabaseTestService.of({
    insertRecords: (count) => Effect.gen(function* () {
      // Simulate insert time proportional to count
      const insertTime = count * 0.1
      yield* Effect.sleep(Duration.millis(insertTime))
      return count
    }),
    
    queryRecords: (limit) => Effect.gen(function* () {
      const queryTime = (limit || 100) * 0.05
      yield* Effect.sleep(Duration.millis(queryTime))
      return Array.from({ length: limit || 100 }, (_, i) => ({ id: i }))
    }),
    
    clearRecords: Effect.succeed(void 0)
  })
)

// Database performance test with size scaling
const databaseScalabilityTest = Effect.gen(function* () {
  const size = yield* TestServices.size
  const db = yield* DatabaseTestService
  
  // Scale database operations with test size
  const recordCount = Math.max(1, size * 2)
  const queryLimit = Math.max(10, size)
  
  // Test insert performance
  const insertStart = Date.now()
  const insertedCount = yield* db.insertRecords(recordCount)
  const insertTime = Date.now() - insertStart
  
  // Test query performance
  const queryStart = Date.now()
  const records = yield* db.queryRecords(queryLimit)
  const queryTime = Date.now() - queryStart
  
  yield* db.clearRecords
  
  return {
    testSize: size,
    recordCount,
    queryLimit,
    insertedCount,
    queriedCount: records.length,
    insertTime,
    queryTime,
    totalTime: insertTime + queryTime
  }
})

// Database test suite with multiple sizes
const runDatabaseTestSuite = Effect.gen(function* () {
  const sizes = [5, 25, 50, 100, 250]
  const results = []
  
  for (const size of sizes) {
    const result = yield* databaseScalabilityTest.pipe(
      TestServices.withSize(size)
    )
    
    results.push(result)
    
    // Validate performance characteristics
    const expectedInsertTime = result.recordCount * 0.15 // 0.15ms per record + overhead
    const expectedQueryTime = result.queryLimit * 0.1 // 0.1ms per record + overhead
    
    if (result.insertTime > expectedInsertTime) {
      console.warn(`Insert time ${result.insertTime}ms exceeded expected ${expectedInsertTime}ms for size ${size}`)
    }
    
    if (result.queryTime > expectedQueryTime) {
      console.warn(`Query time ${result.queryTime}ms exceeded expected ${expectedQueryTime}ms for size ${size}`)
    }
  }
  
  return {
    testResults: results,
    performance: {
      avgInsertTimePerRecord: results.reduce((sum, r) => sum + r.insertTime / r.recordCount, 0) / results.length,
      avgQueryTimePerRecord: results.reduce((sum, r) => sum + r.queryTime / r.queriedCount, 0) / results.length
    }
  }
}).pipe(
  Effect.provide(TestServices.liveServices),
  Effect.provide(mockDatabaseLayer)
)
```

### Integration with HTTP Load Testing

Testing HTTP services with size-controlled request volumes:

```typescript
import { Effect, TestServices, HttpClient, Array as Arr } from "effect"

// HTTP load testing with size-based scaling
const httpLoadTest = Effect.gen(function* () {
  const size = yield* TestServices.size
  const client = yield* HttpClient.HttpClient
  
  // Scale request parameters with size
  const requestCount = Math.min(size, 500) // Cap at 500 requests
  const concurrency = Math.max(1, Math.floor(size / 20)) // More concurrent requests for larger sizes
  const payloadSize = Math.max(100, size * 10) // Larger payloads for larger sizes
  
  // Generate size-appropriate payloads
  const payload = {
    data: Array.from({ length: payloadSize }, (_, i) => `item-${i}`),
    metadata: {
      testSize: size,
      timestamp: Date.now(),
      requestCount
    }
  }
  
  // Create batch of requests
  const requests = Arr.range(0, requestCount - 1).map(i =>
    HttpClient.post(`/api/test/${i}`, {
      body: HttpClient.body.json({ ...payload, requestId: i })
    }).pipe(
      HttpClient.fetchOk,
      Effect.timeout(Duration.seconds(30)),
      Effect.map(() => ({ requestId: i, status: "success" as const })),
      Effect.catchAll(error => Effect.succeed({ 
        requestId: i, 
        status: "failed" as const, 
        error: error.toString() 
      }))
    )
  )
  
  // Execute requests with controlled concurrency
  const startTime = Date.now()
  const results = yield* Effect.allWith(requests, { concurrency })
  const totalTime = Date.now() - startTime
  
  const successCount = results.filter(r => r.status === "success").length
  const failureCount = results.filter(r => r.status === "failed").length
  
  return {
    testSize: size,
    requestCount,
    concurrency,
    payloadSize,
    successCount,
    failureCount,
    totalTime,
    requestsPerSecond: requestCount / (totalTime / 1000),
    avgResponseTime: totalTime / requestCount
  }
})

// Progressive HTTP load testing
const progressiveHttpLoadTest = Effect.gen(function* () {
  const testSizes = [10, 50, 100, 200, 500]
  const results = []
  
  for (const size of testSizes) {
    console.log(`Starting HTTP load test with size: ${size}`)
    
    const result = yield* httpLoadTest.pipe(
      TestServices.withSize(size),
      Effect.timeout(Duration.minutes(2))
    )
    
    results.push(result)
    
    // Brief pause between test runs
    yield* Effect.sleep(Duration.seconds(2))
    
    // Early termination if failure rate is too high
    if (result.failureCount / result.requestCount > 0.1) {
      console.warn(`High failure rate at size ${size}, stopping progressive test`)
      break
    }
  }
  
  return {
    results,
    summary: {
      maxSuccessfulSize: Math.max(...results.filter(r => r.failureCount === 0).map(r => r.testSize)),
      avgRequestsPerSecond: results.reduce((sum, r) => sum + r.requestsPerSecond, 0) / results.length,
      totalRequests: results.reduce((sum, r) => sum + r.requestCount, 0)
    }
  }
}).pipe(
  Effect.provide(TestServices.liveServices),
  Effect.provide(HttpClient.layer)
)
```

## Conclusion

TestSized provides sophisticated size control for property-based testing and performance validation in Effect applications.

Key benefits:
- **Scalable Testing** - Systematically test behavior across different data sizes
- **Performance Validation** - Validate performance characteristics as data scales
- **Property-Based Integration** - Seamlessly integrates with FastCheck for comprehensive testing
- **Flexible Size Control** - Dynamic size adjustment during test execution

TestSized is essential when you need to validate how your applications behave with varying data scales, ensure performance characteristics across size boundaries, or implement sophisticated property-based testing strategies that adapt to data complexity.