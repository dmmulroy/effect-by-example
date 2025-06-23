# TestAnnotationMap: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem TestAnnotationMap Solves

Managing test metadata at scale is a complex challenge. Traditional testing approaches force developers to handle test annotations through scattered, type-unsafe mechanisms:

```typescript
// Traditional approach - scattered test metadata
const testResults = {
  ignored: 3,
  repeated: 5,
  tags: ["slow", "integration"],
  customMetrics: { executionTime: 1500 }
}

// Problems with this approach:
// - No type safety between annotation keys and values
// - Difficult to combine results from multiple test suites
// - Manual tracking of different annotation types
// - No built-in merging strategies for complex data
```

This approach leads to:
- **Type Safety Issues** - No compile-time guarantees about annotation keys and value types
- **Complex Aggregation** - Manual logic required to combine test results across suites
- **Inconsistent Metadata** - Different parts of the codebase use different annotation formats
- **Maintenance Burden** - Changes to annotation structure require updates throughout the codebase

### The TestAnnotationMap Solution

TestAnnotationMap provides a type-safe, composable way to manage collections of test annotations with built-in combination strategies:

```typescript
import { TestAnnotationMap, TestAnnotation } from "effect"

// Type-safe annotations with automatic combination logic
const testMap = TestAnnotationMap.empty().pipe(
  TestAnnotationMap.annotate(TestAnnotation.ignored, 3),
  TestAnnotationMap.annotate(TestAnnotation.repeated, 5),
  TestAnnotationMap.annotate(TestAnnotation.tagged, ["slow", "integration"])
)

// Retrieve values with type safety
const ignoredCount = TestAnnotationMap.get(testMap, TestAnnotation.ignored) // number
const tags = TestAnnotationMap.get(testMap, TestAnnotation.tagged) // HashSet<string>
```

### Key Concepts

**TestAnnotationMap**: A type-safe container that maps TestAnnotation keys to their corresponding values, with built-in combination strategies

**Annotation Combination**: Automatic merging of annotation values when combining maps, using each annotation's defined combination logic

**Type Safety**: Compile-time guarantees that annotation keys and values are correctly paired and handled

## Basic Usage Patterns

### Pattern 1: Creating and Populating Annotation Maps

```typescript
import { TestAnnotationMap, TestAnnotation, HashSet } from "effect"

// Start with an empty map
const emptyMap = TestAnnotationMap.empty()

// Add annotations one by one
const populatedMap = TestAnnotationMap.empty().pipe(
  TestAnnotationMap.annotate(TestAnnotation.ignored, 2),
  TestAnnotationMap.annotate(TestAnnotation.repeated, 3),
  TestAnnotationMap.annotate(TestAnnotation.tagged, HashSet.make("slow", "database"))
)

// Create from existing HashMap
import { HashMap } from "effect"

const existingData = HashMap.empty().pipe(
  HashMap.set(TestAnnotation.ignored, 5),
  HashMap.set(TestAnnotation.retried, 1)
)

const mapFromData = TestAnnotationMap.make(existingData)
```

### Pattern 2: Retrieving Annotation Values

```typescript
import { TestAnnotationMap, TestAnnotation } from "effect"

const testMap = TestAnnotationMap.empty().pipe(
  TestAnnotationMap.annotate(TestAnnotation.ignored, 3),
  TestAnnotationMap.annotate(TestAnnotation.repeated, 2)
)

// Get values with automatic fallback to initial values
const ignoredCount = TestAnnotationMap.get(testMap, TestAnnotation.ignored) // 3
const retriedCount = TestAnnotationMap.get(testMap, TestAnnotation.retried) // 0 (initial value)

// Pattern for safe value access
const getTestMetric = <A>(map: TestAnnotationMap.TestAnnotationMap, annotation: TestAnnotation.TestAnnotation<A>): A =>
  TestAnnotationMap.get(map, annotation)
```

### Pattern 3: Updating and Overwriting Values

```typescript
import { TestAnnotationMap, TestAnnotation } from "effect"

const initialMap = TestAnnotationMap.empty().pipe(
  TestAnnotationMap.annotate(TestAnnotation.ignored, 5)
)

// Update with combination logic (adds to existing value)
const updatedMap = TestAnnotationMap.update(
  initialMap,
  TestAnnotation.ignored,
  (current) => current + 3
) // ignored count becomes 8

// Overwrite completely (replaces existing value)
const overwrittenMap = TestAnnotationMap.overwrite(
  initialMap,
  TestAnnotation.ignored,
  10
) // ignored count becomes 10

// Annotate adds to existing value using annotation's combine function
const annotatedMap = TestAnnotationMap.annotate(
  initialMap,
  TestAnnotation.ignored,
  2
) // ignored count becomes 7 (5 + 2)
```

## Real-World Examples

### Example 1: Test Suite Results Aggregation

Managing test results across multiple test suites requires combining annotations from different sources:

```typescript
import { TestAnnotationMap, TestAnnotation, HashSet, Effect } from "effect"

// Individual test suite results
const unitTestResults = TestAnnotationMap.empty().pipe(
  TestAnnotationMap.annotate(TestAnnotation.ignored, 2),
  TestAnnotationMap.annotate(TestAnnotation.repeated, 1),
  TestAnnotationMap.annotate(TestAnnotation.tagged, HashSet.make("unit", "fast"))
)

const integrationTestResults = TestAnnotationMap.empty().pipe(
  TestAnnotationMap.annotate(TestAnnotation.ignored, 1),
  TestAnnotationMap.annotate(TestAnnotation.retried, 3),
  TestAnnotationMap.annotate(TestAnnotation.tagged, HashSet.make("integration", "slow"))
)

const e2eTestResults = TestAnnotationMap.empty().pipe(
  TestAnnotationMap.annotate(TestAnnotation.repeated, 2),
  TestAnnotationMap.annotate(TestAnnotation.retried, 1),
  TestAnnotationMap.annotate(TestAnnotation.tagged, HashSet.make("e2e", "slow"))
)

// Combine all results automatically
const combinedResults = TestAnnotationMap.combine(
  TestAnnotationMap.combine(unitTestResults, integrationTestResults),
  e2eTestResults
)

// Extract final metrics
const totalIgnored = TestAnnotationMap.get(combinedResults, TestAnnotation.ignored) // 3
const totalRepeated = TestAnnotationMap.get(combinedResults, TestAnnotation.repeated) // 3
const totalRetried = TestAnnotationMap.get(combinedResults, TestAnnotation.retried) // 4
const allTags = TestAnnotationMap.get(combinedResults, TestAnnotation.tagged) 
// HashSet containing: unit, fast, integration, slow, e2e

// Helper for test reporting
const generateTestReport = (results: TestAnnotationMap.TestAnnotationMap) =>
  Effect.gen(function* () {
    const ignored = TestAnnotationMap.get(results, TestAnnotation.ignored)
    const repeated = TestAnnotationMap.get(results, TestAnnotation.repeated)
    const retried = TestAnnotationMap.get(results, TestAnnotation.retried)
    const tags = TestAnnotationMap.get(results, TestAnnotation.tagged)
    
    return {
      summary: {
        ignored,
        repeated,
        retried,
        totalTests: ignored + repeated + retried
      },
      tags: HashSet.toArray(tags),
      success: ignored === 0 && retried === 0
    }
  })
```

### Example 2: Performance Metrics Collection

Collecting and aggregating performance metrics across test runs:

```typescript
import { TestAnnotationMap, TestAnnotation, HashSet, Effect } from "effect"

// Custom performance annotations
const ExecutionTime = TestAnnotation.make<number>(
  "execution-time-ms",
  0,
  (a, b) => a + b
)

const MemoryUsage = TestAnnotation.make<number>(
  "memory-usage-mb",
  0,
  Math.max // Keep the highest memory usage
)

const DatabaseQueries = TestAnnotation.make<number>(
  "database-queries",
  0,
  (a, b) => a + b
)

// Performance tracking during test execution
const trackPerformanceMetrics = (
  testName: string,
  executionTimeMs: number,
  memoryUsageMb: number,
  dbQueries: number
): TestAnnotationMap.TestAnnotationMap =>
  TestAnnotationMap.empty().pipe(
    TestAnnotationMap.annotate(ExecutionTime, executionTimeMs),
    TestAnnotationMap.annotate(MemoryUsage, memoryUsageMb),
    TestAnnotationMap.annotate(DatabaseQueries, dbQueries),
    TestAnnotationMap.annotate(TestAnnotation.tagged, HashSet.make(testName))
  )

// Simulate collecting metrics from multiple test runs
const testRun1 = trackPerformanceMetrics("user-creation", 150, 45, 3)
const testRun2 = trackPerformanceMetrics("order-processing", 300, 78, 8)
const testRun3 = trackPerformanceMetrics("payment-flow", 200, 52, 5)

// Aggregate performance data
const performanceReport = TestAnnotationMap.combine(
  TestAnnotationMap.combine(testRun1, testRun2),
  testRun3
)

// Generate performance insights
const analyzePerformance = (metrics: TestAnnotationMap.TestAnnotationMap) =>
  Effect.gen(function* () {
    const totalExecutionTime = TestAnnotationMap.get(metrics, ExecutionTime)
    const peakMemoryUsage = TestAnnotationMap.get(metrics, MemoryUsage)
    const totalDbQueries = TestAnnotationMap.get(metrics, DatabaseQueries)
    const testsRun = TestAnnotationMap.get(metrics, TestAnnotation.tagged)
    
    const testCount = HashSet.size(testsRun)
    const avgExecutionTime = totalExecutionTime / testCount
    const avgDbQueries = totalDbQueries / testCount
    
    return {
      performance: {
        totalExecutionTime,
        averageExecutionTime: avgExecutionTime,
        peakMemoryUsage,
        totalDatabaseQueries: totalDbQueries,
        averageDatabaseQueries: avgDbQueries
      },
      recommendations: {
        slowTests: avgExecutionTime > 200,
        highMemoryUsage: peakMemoryUsage > 60,
        excessiveDbQueries: avgDbQueries > 5
      },
      testsAnalyzed: HashSet.toArray(testsRun)
    }
  })
```

### Example 3: Test Environment Configuration Tracking

Managing and combining test configuration metadata across different environments:

```typescript
import { TestAnnotationMap, TestAnnotation, HashSet, Effect } from "effect"

// Custom configuration annotations
const Environment = TestAnnotation.make<HashSet.HashSet<string>>(
  "test-environments",
  HashSet.empty(),
  HashSet.union
)

const Features = TestAnnotation.make<HashSet.HashSet<string>>(
  "feature-flags",
  HashSet.empty(),
  HashSet.union
)

const Browsers = TestAnnotation.make<HashSet.HashSet<string>>(
  "browsers-tested",
  HashSet.empty(),
  HashSet.union
)

// Configuration tracking for different test scenarios
const createConfigMap = (
  environments: string[],
  features: string[],
  browsers: string[]
): TestAnnotationMap.TestAnnotationMap =>
  TestAnnotationMap.empty().pipe(
    TestAnnotationMap.annotate(Environment, HashSet.fromIterable(environments)),
    TestAnnotationMap.annotate(Features, HashSet.fromIterable(features)),
    TestAnnotationMap.annotate(Browsers, HashSet.fromIterable(browsers))
  )

// Different test configurations
const smokeTestConfig = createConfigMap(
  ["staging", "production"],
  ["core-features"],
  ["chrome", "firefox"]
)

const regressionTestConfig = createConfigMap(
  ["development", "staging"],
  ["core-features", "experimental-features", "beta-features"],
  ["chrome", "firefox", "safari", "edge"]
)

const performanceTestConfig = createConfigMap(
  ["performance"],
  ["core-features"],
  ["chrome"]
)

// Combine all configurations to get complete coverage overview
const fullTestCoverage = TestAnnotationMap.combine(
  TestAnnotationMap.combine(smokeTestConfig, regressionTestConfig),
  performanceTestConfig
)

// Analyze test coverage
const analyzeCoverage = (config: TestAnnotationMap.TestAnnotationMap) =>
  Effect.gen(function* () {
    const environments = TestAnnotationMap.get(config, Environment)
    const features = TestAnnotationMap.get(config, Features)
    const browsers = TestAnnotationMap.get(config, Browsers)
    
    const coverage = {
      environments: HashSet.toArray(environments),
      features: HashSet.toArray(features),
      browsers: HashSet.toArray(browsers),
      metrics: {
        environmentCount: HashSet.size(environments),
        featureCount: HashSet.size(features),
        browserCount: HashSet.size(browsers)
      }
    }
    
    // Coverage quality checks
    const hasProductionCoverage = HashSet.has(environments, "production")
    const hasMultipleBrowsers = HashSet.size(browsers) >= 3
    const hasFeatureCoverage = HashSet.size(features) >= 2
    
    return {
      coverage,
      quality: {
        production: hasProductionCoverage,
        crossBrowser: hasMultipleBrowsers,
        featureComplete: hasFeatureCoverage,
        overall: hasProductionCoverage && hasMultipleBrowsers && hasFeatureCoverage
      }
    }
  })
```

## Advanced Features Deep Dive

### Feature 1: Custom Annotation Creation

Creating custom annotations with sophisticated combination strategies for domain-specific testing needs.

#### Basic Custom Annotation Usage

```typescript
import { TestAnnotationMap, TestAnnotation, Effect } from "effect"

// Simple counter annotation
const ApiCalls = TestAnnotation.make<number>(
  "api-calls",
  0,
  (a, b) => a + b
)

// Duration tracking with max combination
const MaxDuration = TestAnnotation.make<number>(
  "max-duration-ms",
  0,
  Math.max
)

const basicMap = TestAnnotationMap.empty().pipe(
  TestAnnotationMap.annotate(ApiCalls, 5),
  TestAnnotationMap.annotate(MaxDuration, 1500)
)
```

#### Real-World Custom Annotation Example

```typescript
import { TestAnnotationMap, TestAnnotation, HashMap, HashSet } from "effect"

// Complex annotation for API endpoint coverage
interface EndpointCoverage {
  readonly endpoint: string
  readonly methods: HashSet.HashSet<string>
  readonly statusCodes: HashSet.HashSet<number>
  readonly responseTime: number
}

const combineEndpointCoverage = (
  a: HashMap.HashMap<string, EndpointCoverage>,
  b: HashMap.HashMap<string, EndpointCoverage>
): HashMap.HashMap<string, EndpointCoverage> => {
  let result = a
  
  for (const [endpoint, coverage] of b) {
    if (HashMap.has(result, endpoint)) {
      const existing = HashMap.get(result, endpoint)!
      const combined: EndpointCoverage = {
        endpoint,
        methods: HashSet.union(existing.methods, coverage.methods),
        statusCodes: HashSet.union(existing.statusCodes, coverage.statusCodes),
        responseTime: Math.max(existing.responseTime, coverage.responseTime)
      }
      result = HashMap.set(result, endpoint, combined)
    } else {
      result = HashMap.set(result, endpoint, coverage)
    }
  }
  
  return result
}

const ApiCoverage = TestAnnotation.make<HashMap.HashMap<string, EndpointCoverage>>(
  "api-endpoint-coverage",
  HashMap.empty(),
  combineEndpointCoverage
)

// Helper to create endpoint coverage
const createEndpointCoverage = (
  endpoint: string,
  method: string,
  statusCode: number,
  responseTime: number
): EndpointCoverage => ({
  endpoint,
  methods: HashSet.make(method),
  statusCodes: HashSet.make(statusCode),
  responseTime
})

// Track API coverage during tests
const trackApiCall = (
  map: TestAnnotationMap.TestAnnotationMap,
  endpoint: string,
  method: string,
  statusCode: number,
  responseTime: number
): TestAnnotationMap.TestAnnotationMap => {
  const coverage = HashMap.make([
    endpoint,
    createEndpointCoverage(endpoint, method, statusCode, responseTime)
  ])
  
  return TestAnnotationMap.annotate(map, ApiCoverage, coverage)
}
```

#### Advanced Custom Annotation: Test Complexity Metrics

```typescript
import { TestAnnotationMap, TestAnnotation, Array as Arr, pipe } from "effect"

interface ComplexityMetrics {
  readonly cyclomaticComplexity: number
  readonly linesOfCode: number
  readonly dependencies: HashSet.HashSet<string>
  readonly assertions: number
}

const combineComplexityMetrics = (a: ComplexityMetrics, b: ComplexityMetrics): ComplexityMetrics => ({
  cyclomaticComplexity: Math.max(a.cyclomaticComplexity, b.cyclomaticComplexity),
  linesOfCode: a.linesOfCode + b.linesOfCode,
  dependencies: HashSet.union(a.dependencies, b.dependencies),
  assertions: a.assertions + b.assertions
})

const TestComplexity = TestAnnotation.make<ComplexityMetrics>(
  "test-complexity",
  {
    cyclomaticComplexity: 0,
    linesOfCode: 0,
    dependencies: HashSet.empty(),
    assertions: 0
  },
  combineComplexityMetrics
)

// Calculate complexity score for prioritizing test maintenance
const calculateComplexityScore = (metrics: ComplexityMetrics): number => {
  const dependencyWeight = HashSet.size(metrics.dependencies) * 2
  const complexityWeight = metrics.cyclomaticComplexity * 3
  const locWeight = Math.floor(metrics.linesOfCode / 10)
  const assertionPenalty = metrics.assertions > 20 ? 5 : 0
  
  return dependencyWeight + complexityWeight + locWeight + assertionPenalty
}

// Helper for complexity analysis
const analyzeTestComplexity = (map: TestAnnotationMap.TestAnnotationMap) =>
  Effect.gen(function* () {
    const complexity = TestAnnotationMap.get(map, TestComplexity)
    const score = calculateComplexityScore(complexity)
    
    return {
      metrics: complexity,
      score,
      priority: score > 15 ? "high" : score > 8 ? "medium" : "low",
      recommendations: {
        refactorSuggested: score > 15,
        splitTestSuggested: complexity.linesOfCode > 100,
        reduceDependencies: HashSet.size(complexity.dependencies) > 5,
        simplifyAssertions: complexity.assertions > 20
      }
    }
  })
```

### Feature 2: Map Combination Strategies

Understanding and leveraging different combination approaches for complex testing scenarios.

#### Sequential Combination for Pipeline Testing

```typescript
import { TestAnnotationMap, TestAnnotation, Array as Arr, pipe, Effect } from "effect"

// Custom annotation for pipeline stage tracking
const PipelineStages = TestAnnotation.make<string[]>(
  "pipeline-stages",
  [],
  (a, b) => [...a, ...b]
)

const StageResults = TestAnnotation.make<Record<string, boolean>>(
  "stage-results",
  {},
  (a, b) => ({ ...a, ...b })
)

// Simulate testing different pipeline stages
const createStageResult = (
  stage: string,
  success: boolean,
  dependencies: string[] = []
): TestAnnotationMap.TestAnnotationMap =>
  TestAnnotationMap.empty().pipe(
    TestAnnotationMap.annotate(PipelineStages, [stage]),
    TestAnnotationMap.annotate(StageResults, { [stage]: success }),
    TestAnnotationMap.annotate(TestAnnotation.tagged, HashSet.fromIterable(dependencies))
  )

// Test pipeline execution
const buildStage = createStageResult("build", true, ["nodejs", "typescript"])
const testStage = createStageResult("test", true, ["jest", "coverage"])
const deployStage = createStageResult("deploy", false, ["docker", "kubernetes"])

// Combine pipeline results
const pipelineResults = pipe(
  buildStage,
  TestAnnotationMap.combine(testStage),
  TestAnnotationMap.combine(deployStage)
)

// Analyze pipeline health
const analyzePipeline = (results: TestAnnotationMap.TestAnnotationMap) =>
  Effect.gen(function* () {
    const stages = TestAnnotationMap.get(results, PipelineStages)
    const stageResults = TestAnnotationMap.get(results, StageResults)
    const dependencies = TestAnnotationMap.get(results, TestAnnotation.tagged)
    
    const failedStages = stages.filter(stage => !stageResults[stage])
    const successRate = (stages.length - failedStages.length) / stages.length
    
    return {
      pipeline: {
        stages: stages.length,
        successful: stages.length - failedStages.length,
        failed: failedStages.length,
        successRate
      },
      failures: failedStages,
      dependencies: HashSet.toArray(dependencies),
      status: failedStages.length === 0 ? "success" : "failure"
    }
  })
```

#### Parallel Combination for Load Testing

```typescript
import { TestAnnotationMap, TestAnnotation, Effect } from "effect"

// Load testing annotations
const ConcurrentUsers = TestAnnotation.make<number>(
  "concurrent-users",
  0,
  Math.max // Track peak concurrent users
)

const TotalRequests = TestAnnotation.make<number>(
  "total-requests",
  0,
  (a, b) => a + b
)

const ResponseTimes = TestAnnotation.make<number[]>(
  "response-times",
  [],
  (a, b) => [...a, ...b]
)

// Simulate load test results from different worker threads
const createLoadTestResult = (
  users: number,
  requests: number,
  responseTimes: number[]
): TestAnnotationMap.TestAnnotationMap =>
  TestAnnotationMap.empty().pipe(
    TestAnnotationMap.annotate(ConcurrentUsers, users),
    TestAnnotationMap.annotate(TotalRequests, requests),
    TestAnnotationMap.annotate(ResponseTimes, responseTimes)
  )

// Results from parallel load test workers
const worker1Results = createLoadTestResult(50, 1000, [120, 145, 98, 156])
const worker2Results = createLoadTestResult(75, 1500, [134, 167, 112, 189])
const worker3Results = createLoadTestResult(60, 1200, [108, 142, 156, 178])

// Combine parallel load test results
const aggregatedLoadTest = pipe(
  worker1Results,
  TestAnnotationMap.combine(worker2Results),
  TestAnnotationMap.combine(worker3Results)
)

// Analyze load test performance
const analyzeLoadTest = (results: TestAnnotationMap.TestAnnotationMap) =>
  Effect.gen(function* () {
    const peakUsers = TestAnnotationMap.get(results, ConcurrentUsers)
    const totalRequests = TestAnnotationMap.get(results, TotalRequests)
    const responseTimes = TestAnnotationMap.get(results, ResponseTimes)
    
    const avgResponseTime = responseTimes.reduce((a, b) => a + b, 0) / responseTimes.length
    const p95ResponseTime = responseTimes.sort((a, b) => a - b)[Math.floor(responseTimes.length * 0.95)]
    const requestsPerSecond = totalRequests / 60 // Assuming 1-minute test
    
    return {
      loadTest: {
        peakConcurrentUsers: peakUsers,
        totalRequests,
        averageResponseTime: avgResponseTime,
        p95ResponseTime,
        requestsPerSecond,
        throughput: requestsPerSecond * peakUsers
      },
      performance: {
        acceptable: avgResponseTime < 200 && p95ResponseTime < 500,
        responseTimeGrade: avgResponseTime < 100 ? "A" : avgResponseTime < 200 ? "B" : "C",
        scalabilityScore: Math.min(100, (requestsPerSecond / peakUsers) * 100)
      }
    }
  })
```

## Practical Patterns & Best Practices

### Pattern 1: Type-Safe Annotation Factories

```typescript
import { TestAnnotationMap, TestAnnotation, HashSet, Effect } from "effect"

// Factory pattern for creating consistent annotation maps
interface TestMetadata {
  readonly suite: string
  readonly category: string
  readonly priority: "low" | "medium" | "high"
  readonly tags: readonly string[]
}

const createTestAnnotationMap = (metadata: TestMetadata): TestAnnotationMap.TestAnnotationMap => {
  const priorityWeight = metadata.priority === "high" ? 3 : metadata.priority === "medium" ? 2 : 1
  
  return TestAnnotationMap.empty().pipe(
    TestAnnotationMap.annotate(TestAnnotation.tagged, HashSet.fromIterable([
      metadata.suite,
      metadata.category,
      metadata.priority,
      ...metadata.tags
    ])),
    // Custom priority annotation
    TestAnnotationMap.annotate(
      TestAnnotation.make<number>("priority-weight", 0, Math.max),
      priorityWeight
    )
  )
}

// Reusable annotation builder
const TestAnnotationBuilder = {
  create: (metadata: TestMetadata) => createTestAnnotationMap(metadata),
  
  withIgnored: (map: TestAnnotationMap.TestAnnotationMap, count: number) =>
    TestAnnotationMap.annotate(map, TestAnnotation.ignored, count),
    
  withRepeated: (map: TestAnnotationMap.TestAnnotationMap, count: number) =>
    TestAnnotationMap.annotate(map, TestAnnotation.repeated, count),
    
  withRetried: (map: TestAnnotationMap.TestAnnotationMap, count: number) =>
    TestAnnotationMap.annotate(map, TestAnnotation.retried, count),
    
  combine: (maps: readonly TestAnnotationMap.TestAnnotationMap[]) =>
    maps.reduce(TestAnnotationMap.combine, TestAnnotationMap.empty())
}

// Usage example
const unitTestMap = TestAnnotationBuilder.create({
  suite: "unit-tests",
  category: "fast",
  priority: "high",
  tags: ["core", "isolated"]
}).pipe(
  (map) => TestAnnotationBuilder.withIgnored(map, 2),
  (map) => TestAnnotationBuilder.withRepeated(map, 1)
)
```

### Pattern 2: Annotation Map Validation and Health Checks

```typescript
import { TestAnnotationMap, TestAnnotation, HashSet, Effect, Array as Arr } from "effect"

// Health check patterns for annotation maps
interface TestHealthMetrics {
  readonly totalTests: number
  readonly failureRate: number
  readonly retryRate: number
  readonly coverage: number
}

const calculateHealthMetrics = (map: TestAnnotationMap.TestAnnotationMap): TestHealthMetrics => {
  const ignored = TestAnnotationMap.get(map, TestAnnotation.ignored)
  const repeated = TestAnnotationMap.get(map, TestAnnotation.repeated)
  const retried = TestAnnotationMap.get(map, TestAnnotation.retried)
  const tags = TestAnnotationMap.get(map, TestAnnotation.tagged)
  
  const totalTests = ignored + repeated + retried
  const failureRate = totalTests > 0 ? (ignored + retried) / totalTests : 0
  const retryRate = totalTests > 0 ? retried / totalTests : 0
  const coverage = HashSet.size(tags) // Simple coverage based on tag diversity
  
  return {
    totalTests,
    failureRate,
    retryRate,
    coverage
  }
}

// Validation rules for test health
const validateTestHealth = (metrics: TestHealthMetrics) =>
  Effect.gen(function* () {
    const issues: string[] = []
    const warnings: string[] = []
    
    if (metrics.failureRate > 0.1) {
      issues.push(`High failure rate: ${(metrics.failureRate * 100).toFixed(1)}%`)
    } else if (metrics.failureRate > 0.05) {
      warnings.push(`Elevated failure rate: ${(metrics.failureRate * 100).toFixed(1)}%`)
    }
    
    if (metrics.retryRate > 0.2) {
      issues.push(`Excessive retry rate: ${(metrics.retryRate * 100).toFixed(1)}%`)
    } else if (metrics.retryRate > 0.1) {
      warnings.push(`High retry rate: ${(metrics.retryRate * 100).toFixed(1)}%`)
    }
    
    if (metrics.totalTests === 0) {
      issues.push("No tests executed")
    } else if (metrics.totalTests < 10) {
      warnings.push("Low test count may indicate incomplete coverage")
    }
    
    if (metrics.coverage < 3) {
      warnings.push("Limited test coverage categories")
    }
    
    return {
      status: issues.length > 0 ? "unhealthy" : warnings.length > 0 ? "warning" : "healthy",
      issues,
      warnings,
      metrics,
      score: Math.max(0, 100 - (issues.length * 25) - (warnings.length * 10))
    }
  })

// Comprehensive test suite analysis
const analyzeTestSuite = (maps: readonly TestAnnotationMap.TestAnnotationMap[]) =>
  Effect.gen(function* () {
    const combinedMap = TestAnnotationBuilder.combine(maps)
    const metrics = calculateHealthMetrics(combinedMap)
    const health = yield* validateTestHealth(metrics)
    
    // Additional insights
    const suiteAnalysis = {
      summary: {
        totalSuites: maps.length,
        averageTestsPerSuite: metrics.totalTests / maps.length,
        healthScore: health.score
      },
      recommendations: [
        ...(health.score < 70 ? ["Review and improve test stability"] : []),
        ...(metrics.retryRate > 0.15 ? ["Investigate flaky tests"] : []),
        ...(metrics.coverage < 5 ? ["Expand test coverage categories"] : []),
        ...(metrics.totalTests < maps.length * 5 ? ["Consider adding more test cases"] : [])
      ]
    }
    
    return {
      health,
      analysis: suiteAnalysis,
      rawMetrics: metrics
    }
  })
```

### Pattern 3: Annotation Map Serialization and Persistence

```typescript
import { TestAnnotationMap, TestAnnotation, HashSet, HashMap, Effect, Array as Arr } from "effect"

// Serialization for persistent test reporting
interface SerializedAnnotationMap {
  readonly annotations: Record<string, unknown>
  readonly timestamp: number
  readonly version: string
}

const serializeAnnotationMap = (map: TestAnnotationMap.TestAnnotationMap): SerializedAnnotationMap => {
  // Extract known annotations
  const annotations: Record<string, unknown> = {
    ignored: TestAnnotationMap.get(map, TestAnnotation.ignored),
    repeated: TestAnnotationMap.get(map, TestAnnotation.repeated),
    retried: TestAnnotationMap.get(map, TestAnnotation.retried),
    tagged: HashSet.toArray(TestAnnotationMap.get(map, TestAnnotation.tagged))
  }
  
  return {
    annotations,
    timestamp: Date.now(),
    version: "1.0.0"
  }
}

const deserializeAnnotationMap = (serialized: SerializedAnnotationMap): TestAnnotationMap.TestAnnotationMap => {
  const { annotations } = serialized
  
  return TestAnnotationMap.empty().pipe(
    TestAnnotationMap.annotate(TestAnnotation.ignored, annotations.ignored as number || 0),
    TestAnnotationMap.annotate(TestAnnotation.repeated, annotations.repeated as number || 0),
    TestAnnotationMap.annotate(TestAnnotation.retried, annotations.retried as number || 0),
    TestAnnotationMap.annotate(
      TestAnnotation.tagged,
      HashSet.fromIterable(annotations.tagged as string[] || [])
    )
  )
}

// Persistent test history tracking
interface TestHistoryEntry {
  readonly id: string
  readonly map: TestAnnotationMap.TestAnnotationMap
  readonly timestamp: number
  readonly metadata: Record<string, unknown>
}

const TestHistoryManager = {
  create: () => {
    const history: TestHistoryEntry[] = []
    
    return {
      add: (id: string, map: TestAnnotationMap.TestAnnotationMap, metadata: Record<string, unknown> = {}) =>
        Effect.gen(function* () {
          const entry: TestHistoryEntry = {
            id,
            map,
            timestamp: Date.now(),
            metadata
          }
          history.push(entry)
          return entry
        }),
      
      get: (id: string) =>
        Effect.gen(function* () {
          return history.find(entry => entry.id === id)
        }),
      
      getHistory: (limit?: number) =>
        Effect.gen(function* () {
          const sorted = [...history].sort((a, b) => b.timestamp - a.timestamp)
          return limit ? sorted.slice(0, limit) : sorted
        }),
      
      analyze: (timeRange?: { start: number; end: number }) =>
        Effect.gen(function* () {
          const relevantEntries = timeRange
            ? history.filter(e => e.timestamp >= timeRange.start && e.timestamp <= timeRange.end)
            : history
          
          if (relevantEntries.length === 0) {
            return { trend: "no-data", entries: 0 }
          }
          
          const combinedMap = relevantEntries
            .map(e => e.map)
            .reduce(TestAnnotationMap.combine, TestAnnotationMap.empty())
          
          const metrics = calculateHealthMetrics(combinedMap)
          
          // Trend analysis
          const recentEntries = relevantEntries.slice(-5)
          const oldEntries = relevantEntries.slice(0, 5)
          
          if (recentEntries.length >= 2 && oldEntries.length >= 2) {
            const recentMetrics = calculateHealthMetrics(
              recentEntries.map(e => e.map).reduce(TestAnnotationMap.combine, TestAnnotationMap.empty())
            )
            const oldMetrics = calculateHealthMetrics(
              oldEntries.map(e => e.map).reduce(TestAnnotationMap.combine, TestAnnotationMap.empty())
            )
            
            const trend = recentMetrics.failureRate < oldMetrics.failureRate ? "improving" : 
                         recentMetrics.failureRate > oldMetrics.failureRate ? "declining" : "stable"
            
            return {
              trend,
              entries: relevantEntries.length,
              current: metrics,
              comparison: {
                recent: recentMetrics,
                historical: oldMetrics
              }
            }
          }
          
          return {
            trend: "insufficient-data" as const,
            entries: relevantEntries.length,
            current: metrics
          }
        })
    }
  }
}
```

## Integration Examples

### Integration with Effect Testing Framework

```typescript
import { TestAnnotationMap, TestAnnotation, Effect, TestContext, Layer } from "effect"

// Integration with Effect's testing utilities
const createTestAnnotationService = () => {
  const annotationMapRef = Effect.gen(function* () {
    const ref = yield* Effect.ref(TestAnnotationMap.empty())
    return ref
  })
  
  return {
    annotate: <A>(annotation: TestAnnotation.TestAnnotation<A>, value: A) =>
      Effect.gen(function* () {
        const ref = yield* annotationMapRef
        yield* Effect.updateRef(ref, TestAnnotationMap.annotate(annotation, value))
      }),
    
    get: <A>(annotation: TestAnnotation.TestAnnotation<A>) =>
      Effect.gen(function* () {
        const ref = yield* annotationMapRef
        const map = yield* Effect.readRef(ref)
        return TestAnnotationMap.get(map, annotation)
      }),
    
    getAll: () =>
      Effect.gen(function* () {
        const ref = yield* annotationMapRef
        return yield* Effect.readRef(ref)
      }),
    
    reset: () =>
      Effect.gen(function* () {
        const ref = yield* annotationMapRef
        yield* Effect.setRef(ref, TestAnnotationMap.empty())
      })
  }
}

// Test annotation layer
const TestAnnotationLayer = Layer.sync(
  TestContext.TestAnnotations,
  () => createTestAnnotationService()
)

// Example test with annotations
const exampleTest = Effect.gen(function* () {
  const annotations = yield* TestContext.TestAnnotations
  
  // Test setup with annotation tracking
  yield* annotations.annotate(TestAnnotation.tagged, HashSet.make("integration", "database"))
  
  // Simulate test execution
  const result = yield* Effect.try(() => {
    // Your test logic here
    return { success: true, duration: 150 }
  }).pipe(
    Effect.catchAll((error) =>
      Effect.gen(function* () {
        yield* annotations.annotate(TestAnnotation.retried, 1)
        return { success: false, error: String(error) }
      })
    )
  )
  
  if (result.success) {
    // Track successful test metrics
    const customDuration = TestAnnotation.make<number>("duration-ms", 0, Math.max)
    yield* annotations.annotate(customDuration, result.duration)
  }
  
  return result
}).pipe(
  Effect.provide(TestAnnotationLayer)
)
```

### Integration with Jest and Other Testing Frameworks

```typescript
import { TestAnnotationMap, TestAnnotation, HashSet, Effect } from "effect"

// Jest integration utilities
const EffectTestAnnotations = {
  // Global annotation map for Jest suite
  globalMap: TestAnnotationMap.empty(),
  
  // Jest reporter integration
  setupJestReporter: () => {
    const originalReporter = global.jasmine?.getEnv()?.reporter
    
    return {
      beforeTest: (testName: string, suiteName: string) => {
        const testMap = TestAnnotationMap.empty().pipe(
          TestAnnotationMap.annotate(
            TestAnnotation.tagged,
            HashSet.make(testName, suiteName, "jest")
          )
        )
        
        EffectTestAnnotations.globalMap = TestAnnotationMap.combine(
          EffectTestAnnotations.globalMap,
          testMap
        )
      },
      
      afterTest: (testName: string, result: { status: string; failureMessages?: string[] }) => {
        if (result.status === "failed") {
          EffectTestAnnotations.globalMap = TestAnnotationMap.annotate(
            EffectTestAnnotations.globalMap,
            TestAnnotation.retried,
            1
          )
        }
        
        if (result.status === "pending") {
          EffectTestAnnotations.globalMap = TestAnnotationMap.annotate(
            EffectTestAnnotations.globalMap,
            TestAnnotation.ignored,
            1
          )
        }
      },
      
      getReport: () => {
        const ignored = TestAnnotationMap.get(EffectTestAnnotations.globalMap, TestAnnotation.ignored)
        const retried = TestAnnotationMap.get(EffectTestAnnotations.globalMap, TestAnnotation.retried)
        const tags = TestAnnotationMap.get(EffectTestAnnotations.globalMap, TestAnnotation.tagged)
        
        return {
          summary: { ignored, retried, totalTags: HashSet.size(tags) },
          tags: HashSet.toArray(tags),
          map: EffectTestAnnotations.globalMap
        }
      }
    }
  },
  
  // Helper for Effect-based Jest tests
  runEffectTest: <E, A>(
    effect: Effect.Effect<A, E>,
    annotations: TestAnnotationMap.TestAnnotationMap = TestAnnotationMap.empty()
  ) => {
    return Effect.gen(function* () {
      // Merge test-specific annotations with global map
      EffectTestAnnotations.globalMap = TestAnnotationMap.combine(
        EffectTestAnnotations.globalMap,
        annotations
      )
      
      // Run the test effect
      const result = yield* effect
      
      return result
    }).pipe(
      Effect.catchAll((error) =>
        Effect.gen(function* () {
          // Track test failures
          EffectTestAnnotations.globalMap = TestAnnotationMap.annotate(
            EffectTestAnnotations.globalMap,
            TestAnnotation.retried,
            1
          )
          
          return Effect.fail(error)
        })
      )
    )
  }
}

// Usage with Jest
describe("User Management", () => {
  const reporter = EffectTestAnnotations.setupJestReporter()
  
  beforeEach(() => {
    reporter.beforeTest("user creation", "User Management")
  })
  
  afterEach(() => {
    // Jest will call this based on test results
    reporter.afterTest("user creation", { status: "passed" })
  })
  
  test("should create user successfully", async () => {
    const testAnnotations = TestAnnotationMap.empty().pipe(
      TestAnnotationMap.annotate(TestAnnotation.tagged, HashSet.make("user", "creation", "happy-path"))
    )
    
    const testEffect = Effect.gen(function* () {
      // Your Effect-based test logic here
      const user = yield* createUser({ name: "John", email: "john@example.com" })
      return user
    })
    
    const result = await Effect.runPromise(
      EffectTestAnnotations.runEffectTest(testEffect, testAnnotations)
    )
    
    expect(result).toBeDefined()
    expect(result.name).toBe("John")
  })
  
  afterAll(() => {
    const report = reporter.getReport()
    console.log("Test Execution Report:", report)
  })
})
```

### Testing Strategies

```typescript
import { TestAnnotationMap, TestAnnotation, Effect, Array as Arr } from "effect"

// Property-based testing with annotations
const testAnnotationMapProperties = () => {
  const genAnnotationMap = () => {
    // Generate random annotation maps for property testing
    const randomIgnored = Math.floor(Math.random() * 10)
    const randomRepeated = Math.floor(Math.random() * 5)
    const randomTags = Array.from({ length: Math.floor(Math.random() * 5) }, 
      (_, i) => `tag${i}`)
    
    return TestAnnotationMap.empty().pipe(
      TestAnnotationMap.annotate(TestAnnotation.ignored, randomIgnored),
      TestAnnotationMap.annotate(TestAnnotation.repeated, randomRepeated),
      TestAnnotationMap.annotate(TestAnnotation.tagged, HashSet.fromIterable(randomTags))
    )
  }
  
  // Property: Combining maps is associative
  const testAssociativity = Effect.gen(function* () {
    const map1 = genAnnotationMap()
    const map2 = genAnnotationMap()
    const map3 = genAnnotationMap()
    
    const left = TestAnnotationMap.combine(TestAnnotationMap.combine(map1, map2), map3)
    const right = TestAnnotationMap.combine(map1, TestAnnotationMap.combine(map2, map3))
    
    // Compare key properties
    const leftIgnored = TestAnnotationMap.get(left, TestAnnotation.ignored)
    const rightIgnored = TestAnnotationMap.get(right, TestAnnotation.ignored)
    
    const leftRepeated = TestAnnotationMap.get(left, TestAnnotation.repeated)
    const rightRepeated = TestAnnotationMap.get(right, TestAnnotation.repeated)
    
    return {
      associativeIgnored: leftIgnored === rightIgnored,
      associativeRepeated: leftRepeated === rightRepeated,
      valid: leftIgnored === rightIgnored && leftRepeated === rightRepeated
    }
  })
  
  // Property: Empty map is identity element
  const testIdentity = Effect.gen(function* () {
    const map = genAnnotationMap()
    const empty = TestAnnotationMap.empty()
    
    const leftCombine = TestAnnotationMap.combine(empty, map)
    const rightCombine = TestAnnotationMap.combine(map, empty)
    
    const originalIgnored = TestAnnotationMap.get(map, TestAnnotation.ignored)
    const leftIgnored = TestAnnotationMap.get(leftCombine, TestAnnotation.ignored)
    const rightIgnored = TestAnnotationMap.get(rightCombine, TestAnnotation.ignored)
    
    return {
      leftIdentity: originalIgnored === leftIgnored,
      rightIdentity: originalIgnored === rightIgnored,
      valid: originalIgnored === leftIgnored && originalIgnored === rightIgnored
    }
  })
  
  return { testAssociativity, testIdentity }
}

// Mock/stub strategies for testing
const createMockAnnotationMap = (overrides: Partial<{
  ignored: number
  repeated: number
  retried: number
  tags: string[]
}> = {}): TestAnnotationMap.TestAnnotationMap => {
  const defaults = {
    ignored: 0,
    repeated: 0,
    retried: 0,
    tags: []
  }
  
  const config = { ...defaults, ...overrides }
  
  return TestAnnotationMap.empty().pipe(
    TestAnnotationMap.annotate(TestAnnotation.ignored, config.ignored),
    TestAnnotationMap.annotate(TestAnnotation.repeated, config.repeated),
    TestAnnotationMap.annotate(TestAnnotation.retried, config.retried),
    TestAnnotationMap.annotate(TestAnnotation.tagged, HashSet.fromIterable(config.tags))
  )
}

// Test utilities for annotation map testing
const TestAnnotationMapUtils = {
  createMock: createMockAnnotationMap,
  
  expectEqual: (actual: TestAnnotationMap.TestAnnotationMap, expected: TestAnnotationMap.TestAnnotationMap) =>
    Effect.gen(function* () {
      const actualIgnored = TestAnnotationMap.get(actual, TestAnnotation.ignored)
      const expectedIgnored = TestAnnotationMap.get(expected, TestAnnotation.ignored)
      
      const actualRepeated = TestAnnotationMap.get(actual, TestAnnotation.repeated)
      const expectedRepeated = TestAnnotationMap.get(expected, TestAnnotation.repeated)
      
      const actualRetried = TestAnnotationMap.get(actual, TestAnnotation.retried)
      const expectedRetried = TestAnnotationMap.get(expected, TestAnnotation.retried)
      
      return {
        ignoredMatch: actualIgnored === expectedIgnored,
        repeatedMatch: actualRepeated === expectedRepeated,
        retriedMatch: actualRetried === expectedRetried,
        allMatch: actualIgnored === expectedIgnored && 
                 actualRepeated === expectedRepeated && 
                 actualRetried === expectedRetried
      }
    }),
  
  snapshot: (map: TestAnnotationMap.TestAnnotationMap) => ({
    ignored: TestAnnotationMap.get(map, TestAnnotation.ignored),
    repeated: TestAnnotationMap.get(map, TestAnnotation.repeated),
    retried: TestAnnotationMap.get(map, TestAnnotation.retried),
    tags: HashSet.toArray(TestAnnotationMap.get(map, TestAnnotation.tagged))
  })
}
```

## Conclusion

TestAnnotationMap provides type-safe test metadata management, automatic annotation combination, and composable test result aggregation for large-scale testing scenarios.

Key benefits:
- **Type Safety**: Compile-time guarantees for annotation keys and values with no runtime type errors
- **Composable Architecture**: Seamless combination of test results from multiple sources using monoid properties
- **Extensible Design**: Custom annotations for domain-specific testing needs with flexible combination strategies

TestAnnotationMap is essential when you need to aggregate test metadata across multiple test suites, track complex test metrics with type safety, or build sophisticated test reporting and analysis systems that scale with your codebase.