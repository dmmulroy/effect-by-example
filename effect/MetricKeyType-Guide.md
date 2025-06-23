# MetricKeyType: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem MetricKeyType Solves

When building monitoring systems, developers often struggle with type-unsafe metric creation, runtime errors from incompatible metric types, and confusion about which metric type to use for different data patterns:

```typescript
// Traditional approach - type-unsafe and error-prone
const metrics = {
  requestCount: createMetric('counter'), // No compile-time validation
  responseTime: createMetric('histogram'), // Wrong type could be passed
  memoryUsage: createMetric('gauge'), // No configuration validation
}

// Runtime errors when using wrong input types
metrics.requestCount.record("invalid") // TypeError at runtime
metrics.responseTime.record([1, 2, 3]) // Unexpected structure
```

This approach leads to:
- **Runtime Type Errors** - Wrong input types cause crashes instead of compile-time validation
- **Configuration Drift** - Metric boundaries and settings can become inconsistent
- **Poor Developer Experience** - No IntelliSense or type guidance for metric-specific configurations
- **Maintenance Overhead** - Hard to refactor metrics when requirements change

### The MetricKeyType Solution

MetricKeyType provides compile-time type safety for metric definitions, ensuring each metric type has the correct input/output types and configuration:

```typescript
import { MetricKeyType, MetricBoundaries, Duration } from "effect"

// Type-safe metric key definitions with proper configurations
const counterKeyType = MetricKeyType.counter<number>()
const gaugeKeyType = MetricKeyType.gauge<number>()
const histogramKeyType = MetricKeyType.histogram(
  MetricBoundaries.linear({ start: 0, width: 10, count: 11 })
)
const summaryKeyType = MetricKeyType.summary({
  maxAge: Duration.hours(1),
  maxSize: 1000,
  error: 0.01,
  quantiles: [0.5, 0.95, 0.99]
})
const frequencyKeyType = MetricKeyType.frequency()
```

### Key Concepts

**Metric Key Type**: A type-safe specification that defines what kind of data a metric accepts and what aggregated state it produces

**Input/Output Types**: Each metric key type has specific input types (what you record) and output types (the aggregated metric state)

**Type Safety**: Prevents runtime errors by ensuring only compatible data types can be used with each metric type

## Basic Usage Patterns

### Pattern 1: Creating Basic Metric Key Types

```typescript
import { MetricKeyType } from "effect"

// Counter - tracks cumulative values (numbers or bigints)
const requestCounterType = MetricKeyType.counter<number>()
const errorCounterType = MetricKeyType.counter<bigint>()

// Gauge - tracks current values that can go up or down
const memoryGaugeType = MetricKeyType.gauge<number>()
const diskSpaceGaugeType = MetricKeyType.gauge<bigint>()

// Frequency - counts occurrences of string values
const userAgentFrequencyType = MetricKeyType.frequency()
```

### Pattern 2: Configuring Histogram Key Types

```typescript
import { MetricKeyType, MetricBoundaries } from "effect"

// Linear boundaries for response times (0-100ms in 10ms increments)
const responseTimeType = MetricKeyType.histogram(
  MetricBoundaries.linear({ start: 0, width: 10, count: 11 })
)

// Exponential boundaries for memory allocation sizes
const memoryAllocationType = MetricKeyType.histogram(
  MetricBoundaries.exponential({ start: 1024, factor: 2, count: 10 })
)

// Custom boundaries for specific business metrics
const orderValueType = MetricKeyType.histogram(
  MetricBoundaries.fromIterable([10, 50, 100, 500, 1000, 5000])
)
```

### Pattern 3: Configuring Summary Key Types

```typescript
import { MetricKeyType, Duration } from "effect"

// High-precision latency tracking
const latencyType = MetricKeyType.summary({
  maxAge: Duration.minutes(10),    // Keep 10 minutes of data
  maxSize: 1000,                   // Store up to 1000 samples
  error: 0.01,                     // 1% error margin for quantiles
  quantiles: [0.50, 0.95, 0.99]   // Track median, 95th, and 99th percentiles
})

// Long-term trend analysis
const businessMetricType = MetricKeyType.summary({
  maxAge: Duration.hours(24),      // Keep 24 hours of data
  maxSize: 5000,                   // Store more samples for accuracy
  error: 0.001,                    // Higher precision
  quantiles: [0.5, 0.75, 0.90, 0.95, 0.99, 0.999]
})
```

## Real-World Examples

### Example 1: API Performance Monitoring

Building a comprehensive API monitoring system with different metric types for different performance aspects:

```typescript
import { MetricKeyType, MetricBoundaries, Duration, Effect } from "effect"

// API request counting with incremental-only counter
const apiRequestKeyType = MetricKeyType.counter<number>()

// Response time distribution with histogram
const responseTimeKeyType = MetricKeyType.histogram(
  MetricBoundaries.linear({ start: 0, width: 50, count: 20 }) // 0-1000ms in 50ms buckets
)

// Current active connections with gauge
const activeConnectionsKeyType = MetricKeyType.gauge<number>()

// High-precision latency percentiles with summary
const latencyPercentilesKeyType = MetricKeyType.summary({
  maxAge: Duration.minutes(5),
  maxSize: 1000,
  error: 0.01,
  quantiles: [0.5, 0.95, 0.99]
})

// Track different HTTP status codes with frequency
const statusCodeKeyType = MetricKeyType.frequency({
  preregisteredWords: ["200", "404", "500", "503"] // Pre-register common codes
})

// Example API monitoring service
const createApiMonitoringMetrics = Effect.gen(function* () {
  return {
    requestCount: apiRequestKeyType,
    responseTime: responseTimeKeyType,
    activeConnections: activeConnectionsKeyType,
    latencyPercentiles: latencyPercentilesKeyType,
    statusCodes: statusCodeKeyType
  } as const
})
```

### Example 2: Database Connection Pool Monitoring

Monitoring database connection pool health with appropriate metric types:

```typescript
import { MetricKeyType, MetricBoundaries, Duration } from "effect"

// Pool size tracking with gauge (current active connections)
const poolSizeKeyType = MetricKeyType.gauge<number>()

// Connection wait time distribution
const waitTimeKeyType = MetricKeyType.histogram(
  MetricBoundaries.exponential({ start: 1, factor: 2, count: 15 }) // 1ms to ~32 seconds
)

// Query execution time percentiles
const queryTimeKeyType = MetricKeyType.summary({
  maxAge: Duration.minutes(10),
  maxSize: 2000,
  error: 0.005, // Higher precision for database metrics
  quantiles: [0.5, 0.9, 0.95, 0.99]
})

// Connection lifecycle events
const connectionEventsKeyType = MetricKeyType.frequency({
  preregisteredWords: ["created", "destroyed", "timeout", "error"]
})

// Total connections created (ever-increasing counter)
const totalConnectionsKeyType = MetricKeyType.counter<bigint>()

const createDatabaseMetrics = {
  poolSize: poolSizeKeyType,
  waitTime: waitTimeKeyType,
  queryTime: queryTimeKeyType,
  connectionEvents: connectionEventsKeyType,
  totalConnections: totalConnectionsKeyType
} as const
```

### Example 3: E-commerce Business Metrics

Tracking business-critical metrics with domain-specific configurations:

```typescript
import { MetricKeyType, MetricBoundaries, Duration } from "effect"

// Order value distribution (business intelligence)
const orderValueKeyType = MetricKeyType.histogram(
  MetricBoundaries.fromIterable([
    10, 25, 50, 100, 200, 500, 1000, 2000, 5000, 10000
  ])
)

// Cart abandonment tracking by stage
const cartEventsKeyType = MetricKeyType.frequency({
  preregisteredWords: [
    "item_added", "item_removed", "checkout_started", 
    "payment_failed", "order_completed", "cart_abandoned"
  ]
})

// Customer session duration percentiles
const sessionDurationKeyType = MetricKeyType.summary({
  maxAge: Duration.hours(1),
  maxSize: 5000,
  error: 0.02,
  quantiles: [0.25, 0.5, 0.75, 0.9, 0.95]
})

// Current active users (real-time gauge)
const activeUsersKeyType = MetricKeyType.gauge<number>()

// Revenue tracking (precise big number counter)
const revenueKeyType = MetricKeyType.counter<bigint>()

const createEcommerceMetrics = {
  orderValues: orderValueKeyType,
  cartEvents: cartEventsKeyType,
  sessionDuration: sessionDurationKeyType,
  activeUsers: activeUsersKeyType,
  revenue: revenueKeyType
} as const
```

## Advanced Features Deep Dive

### Feature 1: Type Variance and Safety

MetricKeyType uses sophisticated type variance to ensure input and output types are correctly constrained:

#### Understanding Input/Output Type Relationships

```typescript
import { MetricKeyType, MetricState } from "effect"

// Extract input and output types from a metric key type
type CounterInput = MetricKeyType.InType<MetricKeyType.Counter<number>>    // number
type CounterOutput = MetricKeyType.OutType<MetricKeyType.Counter<number>>  // MetricState.Counter<number>

type HistogramInput = MetricKeyType.InType<MetricKeyType.Histogram>        // number
type HistogramOutput = MetricKeyType.OutType<MetricKeyType.Histogram>      // MetricState.Histogram

type FrequencyInput = MetricKeyType.InType<MetricKeyType.Frequency>        // string
type FrequencyOutput = MetricKeyType.OutType<MetricKeyType.Frequency>      // MetricState.Frequency
```

#### Real-World Type Safety Example

```typescript
import { MetricKeyType, MetricKey, MetricState } from "effect"

// Create a type-safe metric key factory
const createTypedMetricKey = <T extends MetricKeyType.Untyped>(
  name: string,
  keyType: T,
  description?: string
): MetricKey<T> => {
  return MetricKey.make(name, keyType, description)
}

// Usage with compile-time type safety
const performanceMetrics = Effect.gen(function* () {
  const responseTimeKey = createTypedMetricKey(
    "api_response_time",
    MetricKeyType.histogram(MetricBoundaries.linear({ start: 0, width: 10, count: 11 })),
    "API response time distribution"
  )
  
  // TypeScript knows this accepts numbers and returns HistogramState
  const metric = yield* Metric.fromMetricKey(responseTimeKey)
  
  return { responseTime: metric }
})
```

### Feature 2: Advanced Histogram Configurations

#### Choosing the Right Boundary Strategy

```typescript
import { MetricKeyType, MetricBoundaries } from "effect"

// Linear boundaries - uniform distribution
const uniformResponseTime = MetricKeyType.histogram(
  MetricBoundaries.linear({ start: 0, width: 100, count: 10 })
  // Buckets: 0-100, 100-200, ..., 900-1000, 1000+
)

// Exponential boundaries - for metrics with wide ranges
const memoryUsage = MetricKeyType.histogram(
  MetricBoundaries.exponential({ start: 1024, factor: 2, count: 12 })
  // Buckets: 1KB, 2KB, 4KB, 8KB, ..., 2GB, 4GB+
)

// Custom boundaries - business-specific ranges
const orderSizes = MetricKeyType.histogram(
  MetricBoundaries.fromIterable([1, 5, 10, 25, 50, 100, 250, 500])
  // Buckets based on business logic
)
```

#### Advanced Histogram Pattern: Multi-Scale Monitoring

```typescript
import { MetricKeyType, MetricBoundaries, Effect } from "effect"

// Create histogram key types for different time scales
const createMultiScaleLatencyMetrics = Effect.gen(function* () {
  // Fine-grained for real-time monitoring (milliseconds)
  const realtimeLatency = MetricKeyType.histogram(
    MetricBoundaries.linear({ start: 0, width: 10, count: 20 }) // 0-200ms
  )
  
  // Medium-grained for API monitoring (hundreds of milliseconds)
  const apiLatency = MetricKeyType.histogram(
    MetricBoundaries.linear({ start: 0, width: 100, count: 30 }) // 0-3000ms
  )
  
  // Coarse-grained for batch job monitoring (seconds)
  const batchLatency = MetricKeyType.histogram(
    MetricBoundaries.exponential({ start: 1000, factor: 2, count: 10 }) // 1s-1024s
  )
  
  return {
    realtime: realtimeLatency,
    api: apiLatency,
    batch: batchLatency
  } as const
})
```

### Feature 3: Summary Configuration Strategies

#### Precision vs Memory Trade-offs

```typescript
import { MetricKeyType, Duration } from "effect"

// High-precision, short-term monitoring
const highPrecisionLatency = MetricKeyType.summary({
  maxAge: Duration.minutes(5),     // Short window for real-time alerts
  maxSize: 10000,                  // Large sample size for accuracy
  error: 0.001,                    // 0.1% error for high precision
  quantiles: [0.5, 0.9, 0.95, 0.99, 0.999]
})

// Balanced precision, medium-term trends
const balancedMetric = MetricKeyType.summary({
  maxAge: Duration.minutes(30),    // 30-minute window
  maxSize: 2000,                   // Reasonable memory usage
  error: 0.01,                     // 1% error is acceptable
  quantiles: [0.5, 0.95, 0.99]    // Key percentiles only
})

// Low-precision, long-term analysis
const trendAnalysis = MetricKeyType.summary({
  maxAge: Duration.hours(24),      // Full day of data
  maxSize: 1000,                   // Limited samples to save memory
  error: 0.05,                     // 5% error for trends
  quantiles: [0.5, 0.9, 0.95]     // Basic percentiles
})
```

#### Advanced Summary: SLA Monitoring Pattern

```typescript
import { MetricKeyType, Duration, Effect } from "effect"

// Create SLA-focused summary configurations
const createSLAMetrics = Effect.gen(function* () {
  // API SLA monitoring (99.9% uptime, <100ms p95 latency)
  const apiSLA = MetricKeyType.summary({
    maxAge: Duration.minutes(15),   // SLA evaluation window
    maxSize: 5000,
    error: 0.001,                   // High precision for SLA compliance
    quantiles: [0.95, 0.99, 0.999] // Focus on tail latencies
  })
  
  // Database SLA monitoring (99.5% success rate, <50ms p99 latency)
  const dbSLA = MetricKeyType.summary({
    maxAge: Duration.minutes(10),
    maxSize: 3000,
    error: 0.005,
    quantiles: [0.99, 0.995, 0.999]
  })
  
  return { api: apiSLA, database: dbSLA } as const
})
```

## Practical Patterns & Best Practices

### Pattern 1: Metric Key Type Factory

Create reusable factories for common metric configurations:

```typescript
import { MetricKeyType, MetricBoundaries, Duration } from "effect"

// Factory for creating standard latency histograms
const createLatencyHistogram = (maxLatencyMs: number, bucketCount: number = 20) => {
  return MetricKeyType.histogram(
    MetricBoundaries.linear({
      start: 0,
      width: maxLatencyMs / bucketCount,
      count: bucketCount
    })
  )
}

// Factory for creating standard percentile summaries
const createPercentileSummary = (
  windowMinutes: number,
  sampleSize: number = 1000,
  quantiles: readonly number[] = [0.5, 0.95, 0.99]
) => {
  return MetricKeyType.summary({
    maxAge: Duration.minutes(windowMinutes),
    maxSize: sampleSize,
    error: 0.01,
    quantiles
  })
}

// Factory for pre-configured frequency metrics
const createEventFrequency = (preregisteredEvents?: readonly string[]) => {
  return MetricKeyType.frequency({
    preregisteredWords: preregisteredEvents
  })
}

// Usage examples
const httpLatency = createLatencyHistogram(2000) // 0-2000ms histogram
const businessMetrics = createPercentileSummary(60, 5000) // 1-hour window
const userEvents = createEventFrequency(['login', 'logout', 'purchase'])
```

### Pattern 2: Domain-Specific Metric Types

Define metric types tailored to specific business domains:

```typescript
import { MetricKeyType, MetricBoundaries, Duration } from "effect"

// E-commerce domain metrics
const EcommerceMetrics = {
  // Order value buckets based on business tiers
  orderValue: MetricKeyType.histogram(
    MetricBoundaries.fromIterable([10, 25, 50, 100, 250, 500, 1000, 2500])
  ),
  
  // Customer journey events
  customerEvents: MetricKeyType.frequency({
    preregisteredWords: [
      'page_view', 'product_view', 'add_to_cart', 'remove_from_cart',
      'checkout_start', 'payment_info', 'order_complete', 'order_cancel'
    ]
  }),
  
  // Session duration with business-relevant percentiles
  sessionDuration: MetricKeyType.summary({
    maxAge: Duration.hours(2),
    maxSize: 2000,
    error: 0.02,
    quantiles: [0.25, 0.5, 0.75, 0.9] // Business quartiles
  })
} as const

// Infrastructure domain metrics
const InfrastructureMetrics = {
  // CPU usage percentage (0-100)
  cpuUsage: MetricKeyType.histogram(
    MetricBoundaries.linear({ start: 0, width: 5, count: 20 })
  ),
  
  // Memory allocation sizes (bytes)
  memoryAllocation: MetricKeyType.histogram(
    MetricBoundaries.exponential({ start: 1024, factor: 2, count: 16 })
  ),
  
  // Network request latency (microseconds to seconds)
  networkLatency: MetricKeyType.summary({
    maxAge: Duration.minutes(5),
    maxSize: 1000,
    error: 0.01,
    quantiles: [0.5, 0.9, 0.95, 0.99]
  })
} as const
```

### Pattern 3: Metric Type Validation and Guards

Use refinement functions to validate metric key types at runtime:

```typescript
import { MetricKeyType, MetricBoundaries } from "effect"

// Type guards for different metric key types
const validateMetricKeyType = {
  isCounter: (keyType: MetricKeyType.Untyped): keyType is MetricKeyType.Counter<number | bigint> => {
    return MetricKeyType.isCounterKey(keyType)
  },
  
  isGauge: (keyType: MetricKeyType.Untyped): keyType is MetricKeyType.Gauge<number | bigint> => {
    return MetricKeyType.isGaugeKey(keyType)
  },
  
  isHistogram: (keyType: MetricKeyType.Untyped): keyType is MetricKeyType.Histogram => {
    return MetricKeyType.isHistogramKey(keyType)
  },
  
  isSummary: (keyType: MetricKeyType.Untyped): keyType is MetricKeyType.Summary => {
    return MetricKeyType.isSummaryKey(keyType)
  },
  
  isFrequency: (keyType: MetricKeyType.Untyped): keyType is MetricKeyType.Frequency => {
    return MetricKeyType.isFrequencyKey(keyType)
  }
} as const

// Configuration validator
const validateHistogramBoundaries = (boundaries: MetricBoundaries.MetricBoundaries) => {
  const values = boundaries.values
  if (values.length === 0) {
    throw new Error("Histogram must have at least one boundary")
  }
  
  // Ensure boundaries are sorted
  for (let i = 1; i < values.length; i++) {
    if (values[i] <= values[i - 1]) {
      throw new Error("Histogram boundaries must be in ascending order")
    }
  }
  
  return boundaries
}

// Safe metric key type creation
const createValidatedHistogram = (boundaries: number[]) => {
  const validatedBoundaries = validateHistogramBoundaries(
    MetricBoundaries.fromIterable(boundaries)
  )
  return MetricKeyType.histogram(validatedBoundaries)
}
```

## Integration Examples

### Integration with Effect Metric System

```typescript
import { 
  MetricKeyType, MetricKey, Metric, Effect, 
  MetricBoundaries, Duration 
} from "effect"

// Create a complete monitoring system using MetricKeyType
const createApplicationMetrics = Effect.gen(function* () {
  // Define key types
  const requestCountType = MetricKeyType.counter<number>()
  const responseTimeType = MetricKeyType.histogram(
    MetricBoundaries.linear({ start: 0, width: 50, count: 20 })
  )
  const errorRateType = MetricKeyType.summary({
    maxAge: Duration.minutes(5),
    maxSize: 1000,
    error: 0.01,
    quantiles: [0.95, 0.99]
  })
  
  // Create metric keys
  const requestCountKey = MetricKey.make("http_requests_total", requestCountType)
  const responseTimeKey = MetricKey.make("http_response_time", responseTimeType)
  const errorRateKey = MetricKey.make("http_error_rate", errorRateType)
  
  // Create actual metrics
  const requestCounter = yield* Metric.fromMetricKey(requestCountKey)
  const responseTimer = yield* Metric.fromMetricKey(responseTimeKey)
  const errorTracker = yield* Metric.fromMetricKey(errorRateKey)
  
  return {
    requestCounter,
    responseTimer,
    errorTracker
  } as const
})

// Usage in application middleware
const httpMetricsMiddleware = <A, E, R>(
  effect: Effect.Effect<A, E, R>
): Effect.Effect<A, E, R> => {
  return Effect.gen(function* () {
    const metrics = yield* createApplicationMetrics
    const startTime = yield* Effect.clockWith(clock => clock.currentTimeMillis)
    
    try {
      // Track request
      yield* metrics.requestCounter(Effect.succeed(1))
      
      const result = yield* effect
      
      // Track successful response time
      const endTime = yield* Effect.clockWith(clock => clock.currentTimeMillis)
      const duration = endTime - startTime
      yield* metrics.responseTimer(Effect.succeed(duration))
      
      return result
    } catch (error) {
      // Track error
      yield* metrics.errorTracker(Effect.succeed(1))
      throw error
    }
  })
}
```

### Integration with Custom Monitoring Systems

```typescript
import { MetricKeyType, Effect, MetricState } from "effect"

// Custom monitoring adapter
class PrometheusMetricAdapter {
  private keyTypeRegistry = new Map<string, MetricKeyType.Untyped>()
  
  registerMetricKeyType(name: string, keyType: MetricKeyType.Untyped) {
    this.keyTypeRegistry.set(name, keyType)
  }
  
  // Convert Effect metric states to Prometheus format
  formatMetricState = (name: string, state: MetricState.Untyped): string => {
    const keyType = this.keyTypeRegistry.get(name)
    
    if (!keyType) {
      throw new Error(`Unknown metric key type for ${name}`)
    }
    
    if (MetricKeyType.isCounterKey(keyType)) {
      const counterState = state as MetricState.Counter<number>
      return `# TYPE ${name} counter\n${name} ${counterState.count}`
    }
    
    if (MetricKeyType.isGaugeKey(keyType)) {
      const gaugeState = state as MetricState.Gauge<number>
      return `# TYPE ${name} gauge\n${name} ${gaugeState.value}`
    }
    
    if (MetricKeyType.isHistogramKey(keyType)) {
      const histogramState = state as MetricState.Histogram
      let output = `# TYPE ${name} histogram\n`
      
      // Add bucket counts
      for (const [boundary, count] of histogramState.buckets) {
        const label = boundary === Infinity ? '+Inf' : boundary.toString()
        output += `${name}_bucket{le="${label}"} ${count}\n`
      }
      
      // Add sum and count
      output += `${name}_sum ${histogramState.sum}\n`
      output += `${name}_count ${histogramState.count}\n`
      
      return output
    }
    
    throw new Error(`Unsupported metric key type for ${name}`)
  }
}

// Usage with custom adapter
const setupPrometheusMonitoring = Effect.gen(function* () {
  const adapter = new PrometheusMetricAdapter()
  
  // Register metric key types
  const requestCountType = MetricKeyType.counter<number>()
  const memoryUsageType = MetricKeyType.gauge<number>()
  const latencyType = MetricKeyType.histogram(
    MetricBoundaries.exponential({ start: 1, factor: 2, count: 10 })
  )
  
  adapter.registerMetricKeyType("requests_total", requestCountType)
  adapter.registerMetricKeyType("memory_bytes", memoryUsageType)
  adapter.registerMetricKeyType("request_duration_seconds", latencyType)
  
  return adapter
})
```

### Testing Strategies

```typescript
import { MetricKeyType, Effect, TestServices } from "effect"

// Test utilities for metric key types
const createTestMetricKeyTypes = {
  counter: <A extends number | bigint>() => MetricKeyType.counter<A>(),
  gauge: <A extends number | bigint>() => MetricKeyType.gauge<A>(),
  histogram: () => MetricKeyType.histogram(
    MetricBoundaries.fromIterable([1, 5, 10, 50, 100])
  ),
  summary: () => MetricKeyType.summary({
    maxAge: Duration.seconds(10),
    maxSize: 100,
    error: 0.01,
    quantiles: [0.5, 0.95]
  }),
  frequency: () => MetricKeyType.frequency()
}

// Property-based testing for metric key types
const testMetricKeyTypeProperties = Effect.gen(function* () {
  const counterType = createTestMetricKeyTypes.counter<number>()
  const gaugeType = createTestMetricKeyTypes.gauge<number>()
  const histogramType = createTestMetricKeyTypes.histogram()
  
  // Test type guards
  yield* Effect.sync(() => {
    console.assert(MetricKeyType.isCounterKey(counterType))
    console.assert(MetricKeyType.isGaugeKey(gaugeType))
    console.assert(MetricKeyType.isHistogramKey(histogramType))
    console.assert(!MetricKeyType.isCounterKey(gaugeType))
  })
  
  // Test input/output type extraction
  type CounterInput = MetricKeyType.InType<typeof counterType>  // number
  type GaugeOutput = MetricKeyType.OutType<typeof gaugeType>    // MetricState.Gauge<number>
  
  console.log("All metric key type tests passed")
})

// Run tests with Effect TestServices
const runMetricKeyTypeTests = testMetricKeyTypeProperties.pipe(
  Effect.provide(TestServices.TestServices)
)
```

## Conclusion

MetricKeyType provides type-safe metric definitions, compile-time validation, and proper input/output type relationships for robust monitoring systems.

Key benefits:
- **Type Safety**: Prevents runtime errors by ensuring metric types match their usage patterns
- **Configuration Validation**: Validates metric-specific settings like histogram boundaries and summary quantiles at compile time
- **Developer Experience**: Provides IntelliSense and type guidance for metric creation and usage

MetricKeyType is essential when building reliable monitoring systems that need to evolve safely over time while maintaining type correctness across metric collection, aggregation, and reporting.