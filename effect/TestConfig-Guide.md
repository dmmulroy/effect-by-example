# TestConfig: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem TestConfig Solves

Testing complex applications requires precise control over various test execution parameters. Traditional testing approaches often hardcode these values or scatter them across different configuration files, leading to:

```typescript
// Traditional approach - scattered configuration
const REPEATS = 10;
const RETRIES = 5;
const SAMPLES = 50;
const SHRINKS = 100;

// Different tests with different needs
test('flaky network test', async () => {
  let attempts = 0;
  while (attempts < RETRIES) {
    try {
      await makeNetworkCall();
      break;
    } catch (error) {
      attempts++;
      if (attempts === RETRIES) throw error;
    }
  }
});

// Property-based test with manual shrinking
test('property test', () => {
  for (let i = 0; i < SAMPLES; i++) {
    const input = generateRandomInput();
    try {
      assert(property(input));
    } catch (error) {
      // Manual shrinking logic...
      let shrinkAttempts = 0;
      while (shrinkAttempts < SHRINKS) {
        // Complex shrinking implementation
      }
    }
  }
});
```

This approach leads to:
- **Configuration Drift** - Different test files using different values
- **Hard to Maintain** - Scattered configuration across the codebase
- **Environment-Specific Issues** - No easy way to adjust for different environments
- **Inconsistent Test Behavior** - Tests behave differently in different contexts

### The TestConfig Solution

TestConfig provides a centralized, composable service for managing test execution parameters:

```typescript
import { TestConfig, TestServices, Effect } from "effect"

// Clean, centralized configuration
const testConfig = TestConfig.make({
  repeats: 100,    // Repeat tests for stability
  retries: 10,     // Retry flaky tests
  samples: 200,    // Property-based test samples
  shrinks: 1000    // Shrinking attempts for failures
})

// Tests automatically use the configuration
const stableTest = Effect.gen(function* () {
  const config = yield* TestServices.testConfig
  // Test logic with automatic retry/repeat behavior
})
```

### Key Concepts

**Repeats**: Number of times to repeat tests to ensure stability - useful for detecting intermittent failures and race conditions.

**Retries**: Number of times to retry flaky tests - essential for tests that interact with external services or resources.

**Samples**: Number of samples to check for property-based testing - determines how thoroughly random inputs are tested.

**Shrinks**: Maximum number of shrinking attempts to minimize large failures - helps find the minimal failing case in property-based tests.

## Basic Usage Patterns

### Pattern 1: Creating a TestConfig

```typescript
import { TestConfig } from "effect"

// Create a basic test configuration
const basicConfig = TestConfig.make({
  repeats: 50,
  retries: 5,
  samples: 100,
  shrinks: 500
})

// Create environment-specific configurations
const developmentConfig = TestConfig.make({
  repeats: 10,
  retries: 3,
  samples: 50,
  shrinks: 100
})

const productionConfig = TestConfig.make({
  repeats: 200,
  retries: 20,
  samples: 1000,
  shrinks: 5000
})
```

### Pattern 2: Accessing Configuration in Tests

```typescript
import { Effect, TestServices } from "effect"

// Access the entire configuration
const getConfigExample = Effect.gen(function* () {
  const config = yield* TestServices.testConfig
  console.log(`Using ${config.samples} samples for property testing`)
  return config
})

// Access specific configuration values
const getRetriesExample = Effect.gen(function* () {
  const retries = yield* TestServices.retries
  console.log(`Will retry flaky tests ${retries} times`)
  return retries
})

// Access multiple values
const getMultipleValues = Effect.gen(function* () {
  const repeats = yield* TestServices.repeats
  const samples = yield* TestServices.samples
  return { repeats, samples }
})
```

### Pattern 3: Providing Configuration to Tests

```typescript
import { Effect, TestServices, Layer } from "effect"

// Using withTestConfig for specific tests
const testWithCustomConfig = Effect.gen(function* () {
  // Test logic here
  yield* Effect.log("Running with custom config")
}).pipe(
  TestServices.withTestConfig(TestConfig.make({
    repeats: 5,
    retries: 2,
    samples: 25,
    shrinks: 50
  }))
)

// Using Layer for broader configuration
const customConfigLayer = TestServices.testConfigLayer({
  repeats: 75,
  retries: 15,
  samples: 300,
  shrinks: 1500
})

const testWithLayer = Effect.gen(function* () {
  const config = yield* TestServices.testConfig
  yield* Effect.log(`Layer config: ${config.samples} samples`)
}).pipe(
  Effect.provide(customConfigLayer)
)
```

## Real-World Examples

### Example 1: Database Integration Testing

Testing database operations often requires retries due to connection issues and repeats to ensure data consistency:

```typescript
import { Effect, TestServices, TestConfig } from "effect"
import { sql } from "@effect/sql"

// Database-specific test configuration
const databaseTestConfig = TestConfig.make({
  repeats: 20,     // Ensure transactions are consistent
  retries: 10,     // Handle connection timeouts
  samples: 100,    // Test various data combinations
  shrinks: 200     // Find minimal failing queries
})

// Database connection test with automatic retries
const testDatabaseConnection = Effect.gen(function* () {
  const retries = yield* TestServices.retries
  
  const connectWithRetry = (attempt: number): Effect.Effect<boolean> => {
    if (attempt >= retries) {
      return Effect.fail(new Error(`Failed to connect after ${retries} attempts`))
    }
    
    return Effect.gen(function* () {
      const connection = yield* sql.connect()
      yield* sql.query("SELECT 1 as test")
      yield* sql.close(connection)
      return true
    }).pipe(
      Effect.catchAll(() => connectWithRetry(attempt + 1))
    )
  }
  
  return yield* connectWithRetry(0)
}).pipe(
  Effect.provide(TestServices.testConfigLayer(databaseTestConfig))
)

// Test data consistency with automatic repeats
const testDataConsistency = Effect.gen(function* () {
  const repeats = yield* TestServices.repeats
  const results: boolean[] = []
  
  for (let i = 0; i < repeats; i++) {
    const isConsistent = yield* Effect.gen(function* () {
      // Insert test data
      yield* sql.query("INSERT INTO users (name) VALUES ('test')")
      
      // Read and verify
      const users = yield* sql.query("SELECT * FROM users WHERE name = 'test'")
      
      // Cleanup
      yield* sql.query("DELETE FROM users WHERE name = 'test'")
      
      return users.length === 1
    })
    
    results.push(isConsistent)
  }
  
  const consistencyRate = results.filter(Boolean).length / results.length
  return consistencyRate > 0.95 // 95% consistency required
})
```

### Example 2: API Testing with Property-Based Testing

Testing API endpoints with various inputs requires careful configuration of samples and shrinking:

```typescript
import { Effect, TestServices, FastCheck } from "effect"

// API-specific test configuration
const apiTestConfig = TestConfig.make({
  repeats: 5,       // Light repeats for API tests
  retries: 15,      // High retries for network issues
  samples: 500,     // Thorough property testing
  shrinks: 2000     // Extensive shrinking for edge cases
})

// Property-based API testing
const testApiValidation = Effect.gen(function* () {
  const samples = yield* TestServices.samples
  const shrinks = yield* TestServices.shrinks
  
  const userArbitrary = FastCheck.record({
    name: FastCheck.string({ minLength: 1, maxLength: 100 }),
    email: FastCheck.emailAddress(),
    age: FastCheck.integer({ min: 18, max: 120 })
  })
  
  const property = (user: { name: string; email: string; age: number }) =>
    Effect.gen(function* () {
      const response = yield* apiClient.post("/users", user)
      
      // Properties that should always hold
      const hasId = response.id !== undefined
      const nameMatches = response.name === user.name
      const emailMatches = response.email === user.email
      const ageMatches = response.age === user.age
      
      return hasId && nameMatches && emailMatches && ageMatches
    })
  
  // Run property-based test with configured samples
  for (let i = 0; i < samples; i++) {
    const user = FastCheck.sample(userArbitrary, 1)[0]
    
    const result = yield* property(user).pipe(
      Effect.catchAll((error) => {
        // Use configured shrinking to find minimal failing case
        return shrinkFailingCase(user, error, shrinks)
      })
    )
    
    if (!result) {
      return Effect.fail(new Error(`Property violated for user: ${JSON.stringify(user)}`))
    }
  }
  
  return true
}).pipe(
  Effect.provide(TestServices.testConfigLayer(apiTestConfig))
)

// Helper function for shrinking
const shrinkFailingCase = (
  originalInput: any,
  error: unknown,
  maxShrinks: number
): Effect.Effect<boolean> => {
  // Simplified shrinking logic
  return Effect.gen(function* () {
    let currentInput = originalInput
    let shrinkAttempts = 0
    
    while (shrinkAttempts < maxShrinks) {
      const shrunkInput = shrinkInput(currentInput)
      
      const stillFails = yield* property(shrunkInput).pipe(
        Effect.map(() => false),
        Effect.catchAll(() => Effect.succeed(true))
      )
      
      if (stillFails) {
        currentInput = shrunkInput
      }
      
      shrinkAttempts++
    }
    
    yield* Effect.log(`Minimal failing case: ${JSON.stringify(currentInput)}`)
    return false
  })
}

const shrinkInput = (input: any): any => {
  // Implement input shrinking logic
  return {
    ...input,
    name: input.name.length > 1 ? input.name.slice(0, -1) : input.name,
    age: input.age > 18 ? input.age - 1 : input.age
  }
}
```

### Example 3: Performance and Load Testing

Performance tests require specific configuration for different types of measurements:

```typescript
import { Effect, TestServices, Duration, Ref } from "effect"

// Performance test configuration
const performanceTestConfig = TestConfig.make({
  repeats: 100,     // Multiple runs for statistical significance
  retries: 3,       // Few retries for performance tests
  samples: 1000,    // Large sample size for load testing
  shrinks: 50       // Limited shrinking for performance tests
})

// Load testing with configurable sample size
const testApiLoadHandling = Effect.gen(function* () {
  const samples = yield* TestServices.samples
  const repeats = yield* TestServices.repeats
  
  const responseTimes: number[] = []
  const errorCount = yield* Ref.make(0)
  
  // Perform load test with configured sample size
  const loadTest = Effect.gen(function* () {
    const startTime = Date.now()
    
    const requests = Array.from({ length: samples }, (_, i) =>
      Effect.gen(function* () {
        const requestStart = Date.now()
        
        const result = yield* apiClient.get(`/data/${i}`).pipe(
          Effect.catchAll((error) => {
            return Ref.update(errorCount, n => n + 1).pipe(
              Effect.as(null)
            )
          })
        )
        
        const requestEnd = Date.now()
        return requestEnd - requestStart
      })
    )
    
    const times = yield* Effect.all(requests, { concurrency: 50 })
    const validTimes = times.filter(time => time !== null) as number[]
    
    return {
      averageResponseTime: validTimes.reduce((a, b) => a + b, 0) / validTimes.length,
      totalTime: Date.now() - startTime,
      successRate: validTimes.length / samples
    }
  })
  
  // Run the load test multiple times for statistical significance
  for (let i = 0; i < repeats; i++) {
    const result = yield* loadTest
    responseTimes.push(result.averageResponseTime)
    
    yield* Effect.log(`Run ${i + 1}: ${result.averageResponseTime}ms avg, ${result.successRate * 100}% success`)
  }
  
  const overallAverage = responseTimes.reduce((a, b) => a + b, 0) / responseTimes.length
  const errors = yield* Ref.get(errorCount)
  
  return {
    averageResponseTime: overallAverage,
    totalErrors: errors,
    stability: responseTimes.every(time => Math.abs(time - overallAverage) < overallAverage * 0.2)
  }
}).pipe(
  Effect.provide(TestServices.testConfigLayer(performanceTestConfig))
)

// Memory leak detection with repeated execution
const testMemoryLeaks = Effect.gen(function* () {
  const repeats = yield* TestServices.repeats
  const initialMemory = process.memoryUsage().heapUsed
  
  for (let i = 0; i < repeats; i++) {
    yield* Effect.gen(function* () {
      // Perform memory-intensive operation
      const largeArray = new Array(10000).fill(0).map((_, idx) => ({ id: idx, data: Math.random() }))
      
      // Process the data
      const processed = largeArray.map(item => ({
        ...item,
        computed: item.data * 2
      }))
      
      // Cleanup should happen automatically
      return processed.length
    })
    
    // Force garbage collection between iterations
    if (global.gc) {
      global.gc()
    }
  }
  
  const finalMemory = process.memoryUsage().heapUsed
  const memoryGrowth = (finalMemory - initialMemory) / initialMemory
  
  return memoryGrowth < 0.1 // Less than 10% memory growth allowed
})
```

## Advanced Features Deep Dive

### Feature 1: Environment-Specific Configuration

TestConfig enables different configurations for different environments, allowing tests to adapt to their execution context.

#### Basic Environment Configuration

```typescript
import { Effect, TestServices, Config } from "effect"

// Define environment-specific configurations
const getEnvironmentConfig = Effect.gen(function* () {
  const environment = yield* Config.string("NODE_ENV").pipe(
    Config.withDefault("development")
  )
  
  switch (environment) {
    case "development":
      return TestConfig.make({
        repeats: 5,
        retries: 3,
        samples: 50,
        shrinks: 100
      })
    
    case "ci":
      return TestConfig.make({
        repeats: 50,
        retries: 10,
        samples: 200,
        shrinks: 500
      })
    
    case "production":
      return TestConfig.make({
        repeats: 100,
        retries: 20,
        samples: 1000,
        shrinks: 2000
      })
    
    default:
      return TestConfig.make({
        repeats: 10,
        retries: 5,
        samples: 100,
        shrinks: 200
      })
  }
})

// Create a layer that provides environment-specific configuration
const environmentConfigLayer = Layer.effect(
  TestConfig.TestConfig,
  getEnvironmentConfig
)
```

#### Real-World Environment Configuration Example

```typescript
// CI/CD pipeline configuration
const ciConfiguration = Effect.gen(function* () {
  const isCI = yield* Config.boolean("CI").pipe(
    Config.withDefault(false)
  )
  
  const timeout = yield* Config.duration("TEST_TIMEOUT").pipe(
    Config.withDefault(Duration.seconds(30))
  )
  
  const parallel = yield* Config.boolean("PARALLEL_TESTS").pipe(
    Config.withDefault(true)
  )
  
  if (isCI) {
    return TestConfig.make({
      repeats: parallel ? 20 : 50,
      retries: 15,
      samples: 300,
      shrinks: 1000
    })
  } else {
    return TestConfig.make({
      repeats: 5,
      retries: 3,
      samples: 50,
      shrinks: 100
    })
  }
})

// Docker environment configuration
const dockerTestConfiguration = Effect.gen(function* () {
  const inDocker = yield* Config.boolean("DOCKER").pipe(
    Config.withDefault(false)
  )
  
  const memory = yield* Config.integer("MEMORY_LIMIT").pipe(
    Config.withDefault(512)
  )
  
  // Adjust configuration based on available resources
  const scaleFactor = memory / 1024 // Scale based on available memory
  
  return TestConfig.make({
    repeats: Math.floor(20 * scaleFactor),
    retries: inDocker ? 20 : 10, // More retries in containerized environments
    samples: Math.floor(200 * scaleFactor),
    shrinks: Math.floor(500 * scaleFactor)
  })
})
```

### Feature 2: Dynamic Configuration Adjustment

TestConfig can be dynamically adjusted based on test results and system conditions.

#### Adaptive Configuration

```typescript
import { Effect, TestServices, Ref, Metric } from "effect"

// Adaptive configuration based on test performance
const createAdaptiveConfig = Effect.gen(function* () {
  const successRate = yield* Ref.make(1.0)
  const averageResponseTime = yield* Ref.make(100)
  
  const adjustConfig = Effect.gen(function* () {
    const currentSuccessRate = yield* Ref.get(successRate)
    const currentResponseTime = yield* Ref.get(averageResponseTime)
    
    // Adjust retries based on success rate
    const retries = currentSuccessRate < 0.8 ? 20 : 
                   currentSuccessRate < 0.9 ? 10 : 5
    
    // Adjust samples based on response time
    const samples = currentResponseTime > 1000 ? 50 :
                   currentResponseTime > 500 ? 100 : 200
    
    // Adjust repeats based on overall stability
    const repeats = currentSuccessRate > 0.95 && currentResponseTime < 200 ? 5 : 20
    
    return TestConfig.make({
      repeats,
      retries,
      samples,
      shrinks: 500
    })
  })
  
  return { adjustConfig, successRate, averageResponseTime }
})

// Test that adapts its configuration based on results
const adaptiveTest = Effect.gen(function* () {
  const { adjustConfig, successRate, averageResponseTime } = yield* createAdaptiveConfig
  
  // Initial test run
  let currentConfig = yield* adjustConfig
  
  for (let iteration = 0; iteration < 10; iteration++) {
    const testResult = yield* Effect.gen(function* () {
      const startTime = Date.now()
      const samples = yield* TestServices.samples
      
      let successes = 0
      
      for (let i = 0; i < samples; i++) {
        const success = yield* performTest().pipe(
          Effect.map(() => true),
          Effect.catchAll(() => Effect.succeed(false))
        )
        
        if (success) successes++
      }
      
      const endTime = Date.now()
      const responseTime = (endTime - startTime) / samples
      
      return {
        successRate: successes / samples,
        averageResponseTime: responseTime
      }
    }).pipe(
      TestServices.withTestConfig(currentConfig)
    )
    
    // Update metrics
    yield* Ref.set(successRate, testResult.successRate)
    yield* Ref.set(averageResponseTime, testResult.averageResponseTime)
    
    // Adjust configuration for next iteration
    currentConfig = yield* adjustConfig
    
    yield* Effect.log(`Iteration ${iteration}: ${testResult.successRate * 100}% success, ${testResult.averageResponseTime}ms avg`)
  }
  
  return yield* Ref.get(successRate)
})

const performTest = (): Effect.Effect<void> => {
  // Simulate a test that might fail
  return Effect.gen(function* () {
    const random = Math.random()
    if (random < 0.1) {
      yield* Effect.fail(new Error("Random test failure"))
    }
    yield* Effect.sleep(Duration.millis(Math.random() * 200))
  })
}
```

### Feature 3: Configuration Composition and Inheritance

TestConfig supports composition patterns for building complex configuration hierarchies.

#### Configuration Composition

```typescript
// Base configurations for different test types
const baseConfigs = {
  unit: TestConfig.make({
    repeats: 1,
    retries: 3,
    samples: 50,
    shrinks: 100
  }),
  
  integration: TestConfig.make({
    repeats: 5,
    retries: 10,
    samples: 100,
    shrinks: 200
  }),
  
  e2e: TestConfig.make({
    repeats: 10,
    retries: 20,
    samples: 50,
    shrinks: 100
  })
}

// Configuration modifiers
const configModifiers = {
  highThroughput: (config: TestConfig.TestConfig): TestConfig.TestConfig => 
    TestConfig.make({
      ...config,
      samples: config.samples * 2,
      repeats: Math.max(1, Math.floor(config.repeats / 2))
    }),
  
  unstableEnvironment: (config: TestConfig.TestConfig): TestConfig.TestConfig =>
    TestConfig.make({
      ...config,
      retries: config.retries * 2,
      repeats: config.repeats * 2
    }),
  
  quickFeedback: (config: TestConfig.TestConfig): TestConfig.TestConfig =>
    TestConfig.make({
      ...config,
      repeats: Math.max(1, Math.floor(config.repeats / 4)),
      samples: Math.max(10, Math.floor(config.samples / 2)),
      shrinks: Math.max(50, Math.floor(config.shrinks / 2))
    })
}

// Compose configurations
const createComposedConfig = (
  baseType: keyof typeof baseConfigs,
  modifiers: Array<keyof typeof configModifiers>
): TestConfig.TestConfig => {
  const base = baseConfigs[baseType]
  
  return modifiers.reduce(
    (config, modifier) => configModifiers[modifier](config),
    base
  )
}

// Usage examples
const unstableIntegrationConfig = createComposedConfig('integration', ['unstableEnvironment'])
const quickE2EConfig = createComposedConfig('e2e', ['quickFeedback'])
const highThroughputUnitConfig = createComposedConfig('unit', ['highThroughput'])
```

## Practical Patterns & Best Practices

### Pattern 1: Configuration Validation and Constraints

```typescript
import { Effect, Schema } from "effect"

// Configuration validation schema
const TestConfigSchema = Schema.Struct({
  repeats: Schema.Number.pipe(
    Schema.int(),
    Schema.positive(),
    Schema.lessThanOrEqualTo(1000)
  ),
  retries: Schema.Number.pipe(
    Schema.int(),
    Schema.nonNegative(),
    Schema.lessThanOrEqualTo(100)
  ),
  samples: Schema.Number.pipe(
    Schema.int(),
    Schema.positive(),
    Schema.lessThanOrEqualTo(10000)
  ),
  shrinks: Schema.Number.pipe(
    Schema.int(),
    Schema.nonNegative(),
    Schema.lessThanOrEqualTo(10000)
  )
})

// Safe configuration creation with validation
const createValidatedConfig = (params: {
  readonly repeats: number
  readonly retries: number
  readonly samples: number
  readonly shrinks: number
}): Effect.Effect<TestConfig.TestConfig, Schema.ParseResult.ParseError> => {
  return Effect.gen(function* () {
    const validated = yield* Schema.decodeUnknown(TestConfigSchema)(params)
    return TestConfig.make(validated)
  })
}

// Configuration builder with constraints
const configBuilder = {
  create: () => ({
    repeats: 1,
    retries: 0,
    samples: 1,
    shrinks: 0
  }),
  
  withRepeats: (repeats: number) => (config: any) => ({
    ...config,
    repeats: Math.max(1, Math.min(1000, repeats))
  }),
  
  withRetries: (retries: number) => (config: any) => ({
    ...config,
    retries: Math.max(0, Math.min(100, retries))
  }),
  
  withSamples: (samples: number) => (config: any) => ({
    ...config,
    samples: Math.max(1, Math.min(10000, samples))
  }),
  
  withShrinks: (shrinks: number) => (config: any) => ({
    ...config,
    shrinks: Math.max(0, Math.min(10000, shrinks))
  }),
  
  build: (config: any) => createValidatedConfig(config)
}

// Usage with fluent API
const safeConfig = yield* configBuilder
  .create()
  .pipe(
    configBuilder.withRepeats(50),
    configBuilder.withRetries(10),
    configBuilder.withSamples(200),
    configBuilder.withShrinks(500),
    configBuilder.build
  )
```

### Pattern 2: Test Suite Configuration Management

```typescript
import { Effect, Layer, Context } from "effect"

// Test suite configuration service
class TestSuiteConfig extends Context.Tag("TestSuiteConfig")<
  TestSuiteConfig,
  {
    readonly name: string
    readonly config: TestConfig.TestConfig
    readonly metadata: {
      readonly timeout: Duration.Duration
      readonly tags: ReadonlyArray<string>
      readonly parallel: boolean
    }
  }
>() {}

// Helper for creating test suite configurations
const createTestSuite = (
  name: string,
  config: TestConfig.TestConfig,
  options: {
    readonly timeout?: Duration.Duration
    readonly tags?: ReadonlyArray<string>
    readonly parallel?: boolean
  } = {}
) => {
  return Layer.succeed(TestSuiteConfig, {
    name,
    config,
    metadata: {
      timeout: options.timeout ?? Duration.seconds(30),
      tags: options.tags ?? [],
      parallel: options.parallel ?? true
    }
  })
}

// Pre-configured test suites
const testSuites = {
  // Fast unit tests
  unit: createTestSuite(
    "Unit Tests",
    TestConfig.make({
      repeats: 1,
      retries: 2,
      samples: 25,
      shrinks: 50
    }),
    {
      timeout: Duration.seconds(5),
      tags: ["unit", "fast"],
      parallel: true
    }
  ),
  
  // Integration tests with external services
  integration: createTestSuite(
    "Integration Tests",
    TestConfig.make({
      repeats: 3,
      retries: 10,
      samples: 100,
      shrinks: 200
    }),
    {
      timeout: Duration.seconds(30),
      tags: ["integration", "external"],
      parallel: false
    }
  ),
  
  // End-to-end browser tests
  e2e: createTestSuite(
    "E2E Tests",
    TestConfig.make({
      repeats: 2,
      retries: 5,
      samples: 10,
      shrinks: 50
    }),
    {
      timeout: Duration.minutes(5),
      tags: ["e2e", "browser"],
      parallel: false
    }
  )
}

// Test runner that uses suite configuration
const runTestSuite = <A, E>(
  suiteName: keyof typeof testSuites,
  test: Effect.Effect<A, E>
): Effect.Effect<A, E> => {
  return Effect.gen(function* () {
    const suite = yield* TestSuiteConfig
    
    yield* Effect.log(`Running ${suite.name} with config:`)
    yield* Effect.log(`  Repeats: ${suite.config.repeats}`)
    yield* Effect.log(`  Retries: ${suite.config.retries}`)
    yield* Effect.log(`  Samples: ${suite.config.samples}`)
    yield* Effect.log(`  Timeout: ${Duration.toMillis(suite.metadata.timeout)}ms`)
    
    return yield* test.pipe(
      TestServices.withTestConfig(suite.config),
      Effect.timeout(suite.metadata.timeout)
    )
  }).pipe(
    Effect.provide(testSuites[suiteName])
  )
}
```

### Pattern 3: Configuration Monitoring and Metrics

```typescript
import { Effect, Metric, TestServices } from "effect"

// Metrics for test configuration effectiveness
const testMetrics = {
  successRate: Metric.gauge("test_success_rate", { description: "Test success rate" }),
  averageRetries: Metric.gauge("test_average_retries", { description: "Average retries per test" }),
  configUsage: Metric.counter("test_config_usage", { description: "Config usage frequency" }),
  shrinkingEffectiveness: Metric.histogram("shrinking_effectiveness", {
    description: "Effectiveness of shrinking in finding minimal failing cases"
  })
}

// Configuration monitoring wrapper
const withConfigMonitoring = <A, E, R>(
  testName: string,
  effect: Effect.Effect<A, E, R>
): Effect.Effect<A, E, R> => {
  return Effect.gen(function* () {
    const config = yield* TestServices.testConfig
    const startTime = Date.now()
    
    // Track configuration usage
    yield* testMetrics.configUsage.pipe(
      Metric.tagged("test_name", testName),
      Metric.tagged("config_type", getConfigType(config))
    )
    
    let retryCount = 0
    const monitored = effect.pipe(
      Effect.retry({
        times: config.retries,
        schedule: Schedule.exponential(Duration.millis(100))
      }),
      Effect.tapError(() => Effect.sync(() => retryCount++))
    )
    
    const result = yield* monitored.pipe(
      Effect.catchAll((error) => {
        // Record failure metrics
        return Effect.gen(function* () {
          yield* testMetrics.successRate.pipe(
            Metric.set(0),
            Metric.tagged("test_name", testName)
          )
          
          yield* testMetrics.averageRetries.pipe(
            Metric.set(retryCount),
            Metric.tagged("test_name", testName)
          )
          
          return yield* Effect.fail(error)
        })
      }),
      Effect.tap(() => {
        // Record success metrics
        return Effect.gen(function* () {
          yield* testMetrics.successRate.pipe(
            Metric.set(1),
            Metric.tagged("test_name", testName)
          )
          
          yield* testMetrics.averageRetries.pipe(
            Metric.set(retryCount),
            Metric.tagged("test_name", testName)
          )
        })
      })
    )
    
    const endTime = Date.now()
    const duration = endTime - startTime
    
    yield* Effect.log(`Test ${testName} completed in ${duration}ms with ${retryCount} retries`)
    
    return result
  })
}

const getConfigType = (config: TestConfig.TestConfig): string => {
  if (config.repeats === 1 && config.samples < 50) return "unit"
  if (config.retries > 10) return "integration"
  if (config.samples > 500) return "property"
  return "standard"
}

// Usage in tests
const monitoredTest = withConfigMonitoring(
  "api_validation_test",
  Effect.gen(function* () {
    // Test implementation
    yield* Effect.log("Running API validation test")
    return "success"
  })
)
```

## Integration Examples

### Integration with Vitest Testing Framework

```typescript
import { describe, it, expect } from "@effect/vitest"
import { Effect, TestServices, TestConfig } from "effect"

// Custom test configuration for Vitest
const vitestConfig = TestConfig.make({
  repeats: 10,
  retries: 5,
  samples: 100,
  shrinks: 200
})

describe("User API", () => {
  it.effect("should handle user creation with retries", () =>
    Effect.gen(function* () {
      const retries = yield* TestServices.retries
      
      const createUser = (attempt: number): Effect.Effect<User> => {
        if (attempt >= retries) {
          return Effect.fail(new Error("Max retries exceeded"))
        }
        
        return Effect.gen(function* () {
          const response = yield* fetch("/api/users", {
            method: "POST",
            body: JSON.stringify({ name: "John", email: "john@example.com" })
          })
          
          if (!response.ok) {
            return yield* createUser(attempt + 1)
          }
          
          return yield* response.json()
        })
      }
      
      const user = yield* createUser(0)
      expect(user.name).toBe("John")
      expect(user.email).toBe("john@example.com")
    }).pipe(
      Effect.provide(TestServices.testConfigLayer(vitestConfig))
    )
  )
  
  it.effect("should validate user properties", () =>
    Effect.gen(function* () {
      const samples = yield* TestServices.samples
      
      for (let i = 0; i < samples; i++) {
        const user = generateRandomUser()
        const isValid = yield* validateUser(user)
        expect(isValid).toBe(true)
      }
    })
  )
})

interface User {
  id: string
  name: string
  email: string
}

const generateRandomUser = (): User => ({
  id: Math.random().toString(36),
  name: `User${Math.floor(Math.random() * 1000)}`,
  email: `user${Math.floor(Math.random() * 1000)}@example.com`
})

const validateUser = (user: User): Effect.Effect<boolean> =>
  Effect.succeed(
    user.name.length > 0 &&
    user.email.includes("@") &&
    user.id.length > 0
  )
```

### Testing Strategies with Different Configurations

```typescript
import { Effect, TestServices, Layer } from "effect"

// Testing strategy factory
const createTestingStrategy = (
  type: "smoke" | "regression" | "load" | "chaos"
) => {
  const configs = {
    smoke: TestConfig.make({
      repeats: 1,
      retries: 2,
      samples: 10,
      shrinks: 25
    }),
    
    regression: TestConfig.make({
      repeats: 20,
      retries: 10,
      samples: 200,
      shrinks: 500
    }),
    
    load: TestConfig.make({
      repeats: 5,
      retries: 3,
      samples: 1000,
      shrinks: 100
    }),
    
    chaos: TestConfig.make({
      repeats: 50,
      retries: 20,
      samples: 500,
      shrinks: 1000
    })
  }
  
  return {
    config: configs[type],
    layer: TestServices.testConfigLayer(configs[type]),
    
    runTest: <A, E>(
      name: string,
      test: Effect.Effect<A, E>
    ): Effect.Effect<A, E> => {
      return Effect.gen(function* () {
        yield* Effect.log(`Running ${type} test: ${name}`)
        
        const config = yield* TestServices.testConfig
        yield* Effect.log(`Config - Repeats: ${config.repeats}, Retries: ${config.retries}`)
        
        return yield* test
      }).pipe(
        Effect.provide(configs[type] as any)
      )
    }
  }
}

// Usage in different test scenarios
const smokeTestStrategy = createTestingStrategy("smoke")
const regressionTestStrategy = createTestingStrategy("regression")
const loadTestStrategy = createTestingStrategy("load")

// Example tests with different strategies
const runSmokeTests = smokeTestStrategy.runTest(
  "Basic API Health Check",
  Effect.gen(function* () {
    const response = yield* fetch("/health")
    return response.status === 200
  })
)

const runRegressionTests = regressionTestStrategy.runTest(
  "Feature Regression Suite",
  Effect.gen(function* () {
    const samples = yield* TestServices.samples
    const repeats = yield* TestServices.repeats
    
    let failures = 0
    
    for (let repeat = 0; repeat < repeats; repeat++) {
      for (let sample = 0; sample < samples; sample++) {
        const testData = generateTestData(sample)
        const result = yield* processTestData(testData).pipe(
          Effect.catchAll(() => Effect.sync(() => failures++))
        )
      }
    }
    
    const failureRate = failures / (samples * repeats)
    return failureRate < 0.01 // Less than 1% failure rate
  })
)

const runLoadTests = loadTestStrategy.runTest(
  "API Load Test",
  Effect.gen(function* () {
    const samples = yield* TestServices.samples
    
    const requests = Array.from({ length: samples }, (_, i) =>
      Effect.gen(function* () {
        const start = Date.now()
        yield* fetch(`/api/data/${i}`)
        return Date.now() - start
      })
    )
    
    const responseTimes = yield* Effect.all(requests, { concurrency: 100 })
    const averageTime = responseTimes.reduce((a, b) => a + b, 0) / responseTimes.length
    
    return averageTime < 1000 // Average response time under 1 second
  })
)

const generateTestData = (seed: number) => ({
  id: seed,
  data: `test-data-${seed}`,
  timestamp: Date.now()
})

const processTestData = (data: any): Effect.Effect<boolean> =>
  Effect.succeed(data.id >= 0 && data.data.includes("test"))
```

## Conclusion

TestConfig provides centralized, composable configuration management for test execution parameters in Effect applications. It enables consistent test behavior across different environments while maintaining flexibility for specific testing needs.

Key benefits:
- **Centralized Configuration**: Single source of truth for test execution parameters
- **Environment Adaptability**: Different configurations for development, CI, and production environments  
- **Composable Design**: Build complex configurations from simple building blocks
- **Integration Ready**: Works seamlessly with Effect's testing ecosystem and external frameworks

TestConfig is essential when building robust test suites that need to handle varying execution contexts, from quick development feedback loops to comprehensive CI/CD pipeline validation.