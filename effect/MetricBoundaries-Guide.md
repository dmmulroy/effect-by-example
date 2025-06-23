# MetricBoundaries: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem MetricBoundaries Solves

When measuring performance and observability metrics in applications, you often need to track distributions of values rather than just counts or averages. Traditional approaches to histogram configuration lead to several critical issues:

```typescript
// Traditional approach - manual bucket configuration
const responseTimeBuckets = [10, 50, 100, 250, 500, 1000, 2500, 5000, 10000]
const memoryUsageBuckets = [1024, 2048, 4096, 8192, 16384, 32768, 65536]

// Problems with this approach:
// 1. Manual configuration is error-prone
// 2. Hard to maintain consistent bucket strategies
// 3. No type safety for boundary values
// 4. Difficult to adapt boundaries based on requirements
// 5. No automatic +Infinity boundary handling
```

This approach leads to:
- **Inconsistent Metrics** - Different histograms use different bucket strategies
- **Poor Precision** - Manually chosen boundaries often don't match data distribution
- **Maintenance Burden** - Every histogram requires custom boundary configuration
- **Missing Edge Cases** - Forgetting the +Infinity boundary for overflow values

### The MetricBoundaries Solution

MetricBoundaries provides a type-safe, composable way to define histogram bucket boundaries with built-in strategies for common distribution patterns:

```typescript
import { MetricBoundaries, Metric } from "effect"

// Linear boundaries for evenly distributed values
const responseTimeHistogram = Metric.histogram(
  "http_request_duration_seconds",
  MetricBoundaries.linear({ start: 0.01, width: 0.05, count: 20 })
)

// Exponential boundaries for exponentially distributed values  
const memoryUsageHistogram = Metric.histogram(
  "memory_usage_bytes",
  MetricBoundaries.exponential({ start: 1024, factor: 2, count: 10 })
)

// Custom boundaries from iterable
const customBoundaries = MetricBoundaries.fromIterable([10, 25, 50, 100, 250, 500])
```

### Key Concepts

**Linear Boundaries**: Creates evenly spaced bucket boundaries - ideal for normally distributed values like response times or queue lengths.

**Exponential Boundaries**: Creates exponentially increasing bucket boundaries - perfect for power-law distributed values like memory usage or file sizes.

**Automatic Infinity Boundary**: All boundary configurations automatically include a +Infinity bucket to capture overflow values.

## Basic Usage Patterns

### Pattern 1: Linear Boundaries for Response Times

```typescript
import { Effect, Metric, MetricBoundaries } from "effect"

// Create linear boundaries from 0 to 1 second in 50ms increments
const responseTimeBoundaries = MetricBoundaries.linear({
  start: 0,      // Start at 0ms
  width: 0.05,   // 50ms increments  
  count: 21      // 21 buckets (0, 0.05, 0.1, ..., 1.0, +∞)
})

const responseTimeHistogram = Metric.histogram(
  "http_response_time_seconds",
  responseTimeBoundaries,
  "Distribution of HTTP response times"
)
```

### Pattern 2: Exponential Boundaries for Memory Usage

```typescript
// Create exponential boundaries for memory measurements
const memoryBoundaries = MetricBoundaries.exponential({
  start: 1024,   // Start at 1KB
  factor: 2,     // Double each bucket
  count: 10      // 10 buckets (1KB, 2KB, 4KB, ..., 512KB, +∞)
})

const memoryHistogram = Metric.histogram(
  "process_memory_bytes", 
  memoryBoundaries,
  "Process memory usage distribution"
)
```

### Pattern 3: Custom Boundaries from Business Requirements

```typescript
// Create boundaries based on SLA requirements
const slaBasedBoundaries = MetricBoundaries.fromIterable([
  0.1,   // Fast response (100ms)
  0.5,   // Acceptable response (500ms) 
  1.0,   // Slow response (1s)
  5.0,   // Very slow response (5s)
  30.0   // Timeout threshold (30s)
  // +∞ automatically added
])

const slaHistogram = Metric.histogram(
  "sla_response_time_seconds",
  slaBasedBoundaries,
  "Response times measured against SLA thresholds"
)
```

## Real-World Examples

### Example 1: API Response Time Monitoring

Monitoring API response times across different endpoints requires histogram boundaries that capture both fast and slow responses effectively.

```typescript
import { Effect, Metric, MetricBoundaries, Duration } from "effect"

// Linear boundaries optimized for web API response times (0-2000ms)
const apiResponseBoundaries = MetricBoundaries.linear({
  start: 0.01,   // 10ms minimum meaningful response time
  width: 0.1,    // 100ms increments
  count: 20      // Covers 0.01s to 2.01s range
})

const createApiResponseHistogram = (endpoint: string) =>
  Metric.histogram(
    `api_response_duration_seconds`,
    apiResponseBoundaries,
    `Response time distribution for API endpoints`
  ).pipe(
    Metric.tagged("endpoint", endpoint),
    Metric.tagged("service", "api-gateway")
  )

// Usage in API handlers
const measureApiResponse = <A, E, R>(
  endpoint: string, 
  effect: Effect.Effect<A, E, R>
): Effect.Effect<A, E, R> => 
  Effect.gen(function* () {
    const histogram = createApiResponseHistogram(endpoint)
    const startTime = yield* Effect.sync(() => Date.now())
    const result = yield* effect
    const endTime = yield* Effect.sync(() => Date.now())
    const duration = (endTime - startTime) / 1000 // Convert to seconds
    yield* Metric.update(histogram, duration)
    return result
  })

// Example usage
const getUserProfile = (userId: string) =>
  Effect.gen(function* () {
    // Simulate database lookup
    yield* Effect.sleep(Duration.millis(150))
    return { id: userId, name: "John Doe", email: "john@example.com" }
  }).pipe(
    effect => measureApiResponse("/users/:id", effect)
  )
```

### Example 2: Database Query Performance Analysis

Database queries often have exponential performance characteristics - most queries are fast, but some can be very slow.

```typescript
import { Effect, Metric, MetricBoundaries } from "effect"

// Exponential boundaries for database query times
const dbQueryBoundaries = MetricBoundaries.exponential({
  start: 0.001,  // 1ms minimum
  factor: 2.5,   // 2.5x growth factor for good granularity
  count: 12      // Covers 1ms to ~60 seconds
})

const createDbQueryHistogram = (queryType: string) =>
  Metric.histogram(
    "database_query_duration_seconds",
    dbQueryBoundaries,
    "Database query execution time distribution"
  ).pipe(
    Metric.tagged("query_type", queryType),
    Metric.tagged("database", "postgres")
  )

// Helper for measuring query performance
const measureQuery = <A, E, R>(
  queryType: string,
  query: Effect.Effect<A, E, R>
): Effect.Effect<A, E, R> =>
  Effect.gen(function* () {
    const histogram = createDbQueryHistogram(queryType)
    return yield* Metric.trackDuration(histogram, query)
  })

// Example query implementations
const getUserById = (id: string) =>
  Effect.gen(function* () {
    // Simulate SELECT query
    yield* Effect.sleep(Duration.millis(5))
    return { id, name: "User", email: "user@example.com" }
  }).pipe(
    query => measureQuery("select", query)
  )

const generateReport = (filters: unknown) =>
  Effect.gen(function* () {
    // Simulate complex analytical query
    yield* Effect.sleep(Duration.millis(2500))
    return { reportId: "report-123", rows: 10000 }
  }).pipe(
    query => measureQuery("analytics", query)
  )
```

### Example 3: File Size Distribution Monitoring

File upload services need to track file size distributions to optimize storage and bandwidth allocation.

```typescript
import { Effect, Metric, MetricBoundaries } from "effect"

// Combined strategy: linear for small files, exponential for large files
const createFileSizeBoundaries = () => {
  const smallFileBoundaries = [
    1024,      // 1KB
    5120,      // 5KB  
    10240,     // 10KB
    51200,     // 50KB
    102400,    // 100KB
    512000,    // 500KB
    1048576    // 1MB
  ]
  
  const largeBoundaries = MetricBoundaries.exponential({
    start: 2 * 1048576,  // 2MB
    factor: 2,           // Double each bucket
    count: 8             // Up to 256MB
  }).values.slice(0, -1) // Remove the auto-added infinity
  
  return MetricBoundaries.fromIterable([
    ...smallFileBoundaries,
    ...largeBoundaries
  ])
}

const fileSizeHistogram = Metric.histogram(
  "file_upload_size_bytes",
  createFileSizeBoundaries(),
  "Distribution of uploaded file sizes"
)

const trackFileUpload = (fileSize: number, fileType: string) =>
  Effect.gen(function* () {
    const taggedHistogram = fileSizeHistogram.pipe(
      Metric.tagged("file_type", fileType),
      Metric.tagged("upload_method", "multipart")
    )
    yield* Metric.update(taggedHistogram, fileSize)
  })

// Usage example
const uploadFile = (file: { size: number; type: string; data: Uint8Array }) =>
  Effect.gen(function* () {
    // Simulate file processing
    yield* Effect.sleep(Duration.millis(file.size / 1000))
    
    // Track the upload metrics
    yield* trackFileUpload(file.size, file.type)
    
    return { fileId: "file-123", uploadedAt: new Date() }
  })
```

## Advanced Features Deep Dive

### Feature 1: Boundary Value Analysis

Understanding how boundary values are calculated and used is crucial for optimal histogram configuration.

#### Basic Boundary Value Generation

```typescript
import { MetricBoundaries } from "effect"

// Linear boundaries generate evenly spaced values
const linearBoundaries = MetricBoundaries.linear({
  start: 0,
  width: 10,
  count: 5
})

// Resulting values: [0, 10, 20, 30, +∞]
console.log(linearBoundaries.values)

// Exponential boundaries generate exponentially increasing values
const exponentialBoundaries = MetricBoundaries.exponential({
  start: 1,
  factor: 2,
  count: 5  
})

// Resulting values: [1, 2, 4, 8, +∞]
console.log(exponentialBoundaries.values)
```

#### Real-World Boundary Analysis Example

```typescript
import { Effect, MetricBoundaries, pipe } from "effect"

// Helper to analyze boundary effectiveness
const analyzeBoundaries = (
  boundaries: MetricBoundaries.MetricBoundaries,
  sampleData: ReadonlyArray<number>
) =>
  Effect.gen(function* () {
    const bucketCounts = new Map<number, number>()
    
    // Initialize bucket counts
    for (const boundary of boundaries.values) {
      bucketCounts.set(boundary, 0)
    }
    
    // Distribute sample data into buckets
    for (const value of sampleData) {
      for (const boundary of boundaries.values) {
        if (value <= boundary) {
          bucketCounts.set(boundary, (bucketCounts.get(boundary) || 0) + 1)
          break
        }
      }
    }
    
    // Calculate distribution statistics
    const totalSamples = sampleData.length
    const bucketStats = Array.from(bucketCounts.entries()).map(([boundary, count]) => ({
      boundary,
      count,
      percentage: (count / totalSamples) * 100,
      cumulative: 0 // Will be calculated below
    }))
    
    // Calculate cumulative percentages
    let cumulative = 0
    for (const stat of bucketStats) {
      cumulative += stat.count
      stat.cumulative = (cumulative / totalSamples) * 100
    }
    
    return bucketStats
  })

// Example: Analyze response time boundaries
const analyzeResponseTimeBoundaries = Effect.gen(function* () {
  const boundaries = MetricBoundaries.linear({
    start: 0.1,
    width: 0.1,
    count: 10
  })
  
  // Sample response time data (in seconds)
  const responseTimeSamples = [
    0.05, 0.12, 0.18, 0.25, 0.33, 0.41, 0.52, 0.68, 0.75, 0.89,
    0.95, 1.12, 1.25, 1.41, 1.68, 2.15, 2.87, 3.41, 4.25, 5.12
  ]
  
  const stats = yield* analyzeBoundaries(boundaries, responseTimeSamples)
  
  console.log("Boundary Analysis:")
  for (const stat of stats) {
    console.log(
      `≤${stat.boundary}s: ${stat.count} samples (${stat.percentage.toFixed(1)}%, cumulative: ${stat.cumulative.toFixed(1)}%)`
    )
  }
  
  return stats
})
```

#### Advanced Boundary: Custom Distribution Matching

```typescript
// Create boundaries that match expected data distribution
const createPercentileBoundaries = (percentiles: ReadonlyArray<{ percentile: number; value: number }>) =>
  MetricBoundaries.fromIterable(percentiles.map(p => p.value))

// Example: Create boundaries based on historical P50, P90, P95, P99
const createSLABoundaries = () => {
  const historicalPercentiles = [
    { percentile: 50, value: 0.25 },   // P50: 250ms
    { percentile: 90, value: 0.8 },    // P90: 800ms  
    { percentile: 95, value: 1.5 },    // P95: 1.5s
    { percentile: 99, value: 5.0 },    // P99: 5s
    { percentile: 99.9, value: 30.0 }  // P99.9: 30s
  ]
  
  return createPercentileBoundaries(historicalPercentiles)
}

const slaBoundariesHistogram = Metric.histogram(
  "api_response_sla_seconds",
  createSLABoundaries(),
  "API response times measured against historical SLA percentiles"
)
```

### Feature 2: Boundary Optimization Strategies

Different workload patterns require different boundary strategies for optimal metric precision and storage efficiency.

#### Performance-Oriented Linear Boundaries

```typescript
// Optimized for consistent performance monitoring
const createPerformanceBoundaries = (maxExpectedValue: number, granularity: number) =>
  MetricBoundaries.linear({
    start: 0,
    width: maxExpectedValue / granularity,
    count: granularity + 1
  })

// High-precision boundaries for critical performance metrics
const criticalPathBoundaries = createPerformanceBoundaries(2.0, 40) // 50ms precision up to 2s

const criticalPathHistogram = Metric.histogram(
  "critical_path_duration_seconds",
  criticalPathBoundaries,
  "Critical path execution time with high precision"
)
```

#### Resource-Aware Exponential Boundaries  

```typescript
// Optimized for resource usage patterns (memory, disk, network)
const createResourceBoundaries = (baseUnit: number, maxSize: number) => {
  const steps = Math.ceil(Math.log2(maxSize / baseUnit))
  return MetricBoundaries.exponential({
    start: baseUnit,
    factor: 2,
    count: steps + 1
  })
}

// Memory usage boundaries from 1MB to 1GB
const memoryBoundaries = createResourceBoundaries(1024 * 1024, 1024 * 1024 * 1024)

const memoryUsageHistogram = Metric.histogram(
  "jvm_memory_usage_bytes",
  memoryBoundaries,
  "JVM memory usage distribution"
)
```

## Practical Patterns & Best Practices

### Pattern 1: Adaptive Boundary Configuration

```typescript
import { Effect, Config, MetricBoundaries, Metric } from "effect"

// Configuration-driven boundary creation
const createConfigurableBoundaries = Effect.gen(function* () {
  const boundaryType = yield* Config.string("METRIC_BOUNDARY_TYPE").pipe(
    Config.withDefault("linear")
  )
  
  const startValue = yield* Config.number("METRIC_BOUNDARY_START").pipe(
    Config.withDefault(0)
  )
  
  const endValue = yield* Config.number("METRIC_BOUNDARY_END").pipe(
    Config.withDefault(1)
  )
  
  const bucketCount = yield* Config.number("METRIC_BOUNDARY_BUCKETS").pipe(
    Config.withDefault(10)
  )
  
  if (boundaryType === "exponential") {
    const factor = yield* Config.number("METRIC_BOUNDARY_FACTOR").pipe(
      Config.withDefault(2)
    )
    
    return MetricBoundaries.exponential({
      start: startValue,
      factor,
      count: bucketCount
    })
  }
  
  return MetricBoundaries.linear({
    start: startValue,
    width: (endValue - startValue) / (bucketCount - 1),
    count: bucketCount
  })
})

// Usage with environment-based configuration
const createAdaptiveHistogram = (name: string, description: string) =>
  Effect.gen(function* () {
    const boundaries = yield* createConfigurableBoundaries
    return Metric.histogram(name, boundaries, description)
  })
```

### Pattern 2: Multi-Tier Boundary Strategy

```typescript
// Different boundaries for different service tiers
const createTieredBoundaries = (tier: "critical" | "standard" | "background") => {
  switch (tier) {
    case "critical":
      // High precision for critical services (every 10ms up to 1s)
      return MetricBoundaries.linear({
        start: 0.01,
        width: 0.01,
        count: 100
      })
      
    case "standard":
      // Moderate precision for standard services (every 50ms up to 5s)  
      return MetricBoundaries.linear({
        start: 0.05,
        width: 0.05,
        count: 100
      })
      
    case "background":
      // Lower precision for background services (exponential)
      return MetricBoundaries.exponential({
        start: 0.1,
        factor: 1.5,
        count: 15
      })
  }
}

// Service-aware histogram factory
const createServiceHistogram = (
  serviceName: string,
  tier: "critical" | "standard" | "background"
) => {
  const boundaries = createTieredBoundaries(tier)
  return Metric.histogram(
    `service_response_time_seconds`,
    boundaries,
    `Response time distribution for ${tier} tier service`
  ).pipe(
    Metric.tagged("service", serviceName),
    Metric.tagged("tier", tier)
  )
}

// Usage examples
const criticalServiceHistogram = createServiceHistogram("auth-service", "critical")
const standardServiceHistogram = createServiceHistogram("user-service", "standard")  
const backgroundServiceHistogram = createServiceHistogram("reporting-service", "background")
```

### Pattern 3: Boundary Validation and Testing

```typescript
import { Effect, pipe } from "effect"

// Boundary validation utilities
const validateBoundaries = (
  boundaries: MetricBoundaries.MetricBoundaries,
  expectedRange: { min: number; max: number },
  requiredPrecision: number
) =>
  Effect.gen(function* () {
    const values = boundaries.values
    const finiteValues = values.slice(0, -1) // Exclude +∞
    
    // Validate range coverage
    const minBoundary = Math.min(...finiteValues)
    const maxFiniteBoundary = Math.max(...finiteValues.filter(v => v !== Number.POSITIVE_INFINITY))
    
    if (minBoundary > expectedRange.min) {
      yield* Effect.fail(new Error(`Minimum boundary ${minBoundary} exceeds expected minimum ${expectedRange.min}`))
    }
    
    if (maxFiniteBoundary < expectedRange.max) {
      yield* Effect.fail(new Error(`Maximum boundary ${maxFiniteBoundary} below expected maximum ${expectedRange.max}`))
    }
    
    // Validate precision in critical range
    const criticalRangeValues = finiteValues.filter(
      v => v >= expectedRange.min && v <= expectedRange.max
    )
    
    const minGap = Math.min(
      ...criticalRangeValues.slice(1).map((v, i) => v - criticalRangeValues[i])
    )
    
    if (minGap > requiredPrecision) {
      yield* Effect.fail(new Error(`Boundary precision ${minGap} exceeds required precision ${requiredPrecision}`))
    }
    
    return {
      rangeMin: minBoundary,
      rangeMax: maxFiniteBoundary,
      precision: minGap,
      bucketCount: finiteValues.length,
      valid: true
    }
  })

// Test boundary configuration
const testResponseTimeBoundaries = Effect.gen(function* () {
  const boundaries = MetricBoundaries.linear({
    start: 0,
    width: 0.05,  // 50ms precision
    count: 21     // 0 to 1 second
  })
  
  const validation = yield* validateBoundaries(
    boundaries,
    { min: 0, max: 1.0 },    // Expected range: 0-1 second
    0.1                       // Required precision: 100ms
  )
  
  console.log("Boundary validation:", validation)
  return validation
})
```

## Integration Examples

### Integration with Prometheus Metrics

```typescript
import { Effect, Metric, MetricBoundaries, Layer } from "effect"

// Prometheus-compatible boundary configurations
const createPrometheusBoundaries = (metricType: "duration" | "size" | "count") => {
  switch (metricType) {
    case "duration":
      // Prometheus default duration buckets (in seconds)
      return MetricBoundaries.fromIterable([
        0.005, 0.01, 0.025, 0.05, 0.075, 0.1, 0.25, 0.5, 0.75, 1.0, 2.5, 5.0, 7.5, 10.0
      ])
      
    case "size":
      // Prometheus default size buckets (in bytes)
      return MetricBoundaries.fromIterable([
        100, 1000, 10000, 100000, 1000000, 10000000, 100000000, 1000000000
      ])
      
    case "count":
      // Prometheus default count buckets
      return MetricBoundaries.fromIterable([
        1, 2, 5, 10, 20, 50, 100, 200, 500, 1000, 2000, 5000, 10000
      ])
  }
}

// HTTP server integration with Prometheus-compatible metrics
const createHttpMetrics = Effect.gen(function* () {
  const requestDuration = Metric.histogram(
    "http_request_duration_seconds",
    createPrometheusBoundaries("duration"),
    "HTTP request duration in seconds"
  )
  
  const requestSize = Metric.histogram(
    "http_request_size_bytes", 
    createPrometheusBoundaries("size"),
    "HTTP request size in bytes"
  )
  
  const responseSize = Metric.histogram(
    "http_response_size_bytes",
    createPrometheusBoundaries("size"), 
    "HTTP response size in bytes"
  )
  
  return { requestDuration, requestSize, responseSize } as const
})

// HTTP middleware with metrics
const withHttpMetrics = <A, E, R>(
  handler: Effect.Effect<A, E, R>
): Effect.Effect<A, E, R> =>
  Effect.gen(function* () {
    const metrics = yield* createHttpMetrics
    const startTime = yield* Effect.sync(() => Date.now())
    
    const result = yield* handler
    
    const endTime = yield* Effect.sync(() => Date.now())
    const duration = (endTime - startTime) / 1000
    
    yield* Metric.update(metrics.requestDuration, duration)
    
    return result
  })
```

### Integration with OpenTelemetry

```typescript
import { Effect, Metric, MetricBoundaries } from "effect"

// OpenTelemetry semantic convention compatible boundaries
const createOTelBoundaries = (instrument: "http.server.duration" | "http.client.duration" | "db.client.operation.duration") => {
  switch (instrument) {
    case "http.server.duration":
      // Optimized for web server response times (milliseconds)
      return MetricBoundaries.exponential({
        start: 0.005,  // 5ms
        factor: 2,
        count: 12      // Up to ~20 seconds
      })
      
    case "http.client.duration":
      // Optimized for HTTP client requests (seconds)
      return MetricBoundaries.linear({
        start: 0.01,   // 10ms
        width: 0.05,   // 50ms increments
        count: 40      // Up to 2 seconds
      })
      
    case "db.client.operation.duration":
      // Optimized for database operations (milliseconds)
      return MetricBoundaries.exponential({
        start: 0.001,  // 1ms
        factor: 2.5,
        count: 10      // Up to ~1 second
      })
  }
}

// OTel-compatible metric service
const OTelMetricsService = Effect.gen(function* () {
  const httpServerDuration = Metric.histogram(
    "http.server.duration",
    createOTelBoundaries("http.server.duration"),
    "Duration of HTTP server requests"
  )
  
  const httpClientDuration = Metric.histogram(
    "http.client.duration", 
    createOTelBoundaries("http.client.duration"),
    "Duration of HTTP client requests"
  )
  
  const dbOperationDuration = Metric.histogram(
    "db.client.operation.duration",
    createOTelBoundaries("db.client.operation.duration"),
    "Duration of database operations"
  )
  
  return {
    httpServerDuration,
    httpClientDuration, 
    dbOperationDuration,
    
    // Helper methods for common patterns
    measureHttpServer: (method: string, route: string) =>
      httpServerDuration.pipe(
        Metric.tagged("http.method", method),
        Metric.tagged("http.route", route)
      ),
      
    measureHttpClient: (method: string, url: string) =>
      httpClientDuration.pipe(
        Metric.tagged("http.method", method),
        Metric.tagged("http.url", url)
      ),
      
    measureDbOperation: (operation: string, table: string) =>
      dbOperationDuration.pipe(
        Metric.tagged("db.operation", operation),
        Metric.tagged("db.collection.name", table)
      )
  } as const
})

// Usage in application code
const trackDbQuery = <A, E, R>(
  operation: string,
  table: string,
  query: Effect.Effect<A, E, R>
): Effect.Effect<A, E, R> =>
  Effect.gen(function* () {
    const otelMetrics = yield* OTelMetricsService
    const histogram = otelMetrics.measureDbOperation(operation, table)
    return yield* Metric.trackDuration(histogram, query)
  })
```

### Testing Strategies

```typescript
import { Effect, Metric, MetricBoundaries, TestEnvironment } from "effect"

// Test utilities for boundary validation
const createTestBoundaries = () =>  
  MetricBoundaries.linear({
    start: 0,
    width: 0.1,
    count: 11  // 0, 0.1, 0.2, ..., 1.0, +∞
  })

const testHistogramBehavior = Effect.gen(function* () {
  const testHistogram = Metric.histogram(
    "test_histogram",
    createTestBoundaries(),
    "Test histogram for boundary validation"
  )
  
  // Test boundary edge cases
  const testValues = [0.05, 0.1, 0.15, 0.95, 1.0, 1.5, 2.0]
  
  for (const value of testValues) {
    yield* Metric.update(testHistogram, value)
  }
  
  const result = yield* Metric.value(testHistogram)
  
  // Validate histogram state
  console.log(`Histogram recorded ${result.count} observations`)
  console.log(`Sum: ${result.sum}, Min: ${result.min}, Max: ${result.max}`)
  
  return result
})

// Property-based testing for boundary configurations
const testBoundaryProperties = (boundaries: MetricBoundaries.MetricBoundaries) =>
  Effect.gen(function* () {
    const values = boundaries.values
    
    // Property: Values should be sorted in ascending order
    const sorted = [...values].sort((a, b) => a - b)
    if (!values.every((v, i) => v === sorted[i])) {
      yield* Effect.fail(new Error("Boundary values are not sorted"))
    }
    
    // Property: Should end with +Infinity
    if (values[values.length - 1] !== Number.POSITIVE_INFINITY) {
      yield* Effect.fail(new Error("Boundary values should end with +Infinity"))
    }
    
    // Property: No duplicate values
    const uniqueValues = new Set(values)
    if (uniqueValues.size !== values.length) {
      yield* Effect.fail(new Error("Boundary values contain duplicates"))
    }
    
    return { valid: true, boundaryCount: values.length }
  })
```

## Conclusion

MetricBoundaries provides **type-safe histogram configuration**, **optimal distribution matching**, and **seamless integration** with Effect's metrics system for comprehensive observability.

Key benefits:
- **Precision Control**: Linear and exponential strategies match your data's distribution patterns
- **Type Safety**: Compile-time validation prevents runtime metric configuration errors  
- **Composability**: Integrates seamlessly with Metric, Layer, and other Effect modules for robust observability pipelines

Use MetricBoundaries when you need precise, configurable histogram boundaries that adapt to your application's performance characteristics and observability requirements.