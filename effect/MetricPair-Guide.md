# MetricPair: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem MetricPair Solves

Traditional metric systems often struggle with associating metric definitions with their current state values, leading to fragmented monitoring setups:

```typescript
// Traditional approach - scattered metric handling
class MetricsCollector {
  private counters = new Map<string, number>()
  private gauges = new Map<string, number>()
  
  increment(name: string, value = 1) {
    this.counters.set(name, (this.counters.get(name) || 0) + value)
  }
  
  setGauge(name: string, value: number) {
    this.gauges.set(name, value)
  }
  
  // Problematic: No association between metric definition and state
  exportMetrics() {
    const result: any[] = []
    
    for (const [name, value] of this.counters) {
      result.push({ type: 'counter', name, value })
    }
    
    for (const [name, value] of this.gauges) {
      result.push({ type: 'gauge', name, value })
    }
    
    return result
  }
}

// Manual state management with no type safety
const collector = new MetricsCollector()
collector.increment('http_requests_total')
collector.setGauge('active_connections', 42)
```

This approach leads to:
- **State Fragmentation** - Metric definitions and their values exist separately
- **Type Unsafety** - No compile-time verification that state matches metric type
- **Manual Synchronization** - Developers must manually ensure metric keys match state types
- **Export Complexity** - Complex logic needed to combine definitions with current state

### The MetricPair Solution

MetricPair creates a typed association between a metric's definition (MetricKey) and its current state (MetricState), ensuring type safety and simplifying metric collection:

```typescript
import { Effect, Metric, MetricKey, MetricPair } from "effect"

// MetricPair automatically associates definition with state
const createRequestCounter = Effect.gen(function* () {
  const requestKey = MetricKey.counter("http_requests_total", {
    description: "Total number of HTTP requests"
  })
  
  // Create metric state directly
  const currentState = MetricState.counter(0)
  
  // MetricPair provides type-safe association
  return MetricPair.make(requestKey, currentState)
})
```

### Key Concepts

**MetricKey**: The metric definition containing name, type, and metadata (tags, description)

**MetricState**: The current value(s) of a metric (counter value, gauge reading, histogram buckets)

**MetricPair**: A type-safe container that associates a MetricKey with its corresponding MetricState

## Basic Usage Patterns

### Pattern 1: Creating Basic MetricPairs

```typescript
import { Effect, Metric, MetricKey, MetricPair, MetricState } from "effect"

// Create a counter metric pair
const createCounterPair = Effect.gen(function* () {
  const counterKey = MetricKey.counter("api_calls_total")
  const counterState = MetricState.counter(5) // Current count: 5
  
  return MetricPair.make(counterKey, counterState)
})

// Create a gauge metric pair  
const createGaugePair = Effect.gen(function* () {
  const gaugeKey = MetricKey.gauge("memory_usage_bytes", {
    description: "Current memory usage in bytes"
  })
  const gaugeState = MetricState.gauge(1048576) // 1MB
  
  return MetricPair.make(gaugeKey, gaugeState)
})
```

### Pattern 2: Extracting Data from MetricPairs

```typescript
// Working with MetricPair data
const inspectMetricPair = (pair: MetricPair.MetricPair.Untyped) => Effect.gen(function* () {
  const metricName = pair.metricKey.name
  const metricType = pair.metricKey.keyType
  const currentState = pair.metricState
  
  yield* Effect.log(`Metric: ${metricName}, Type: ${metricType._tag}`)
  
  // Type-safe state access based on metric type
  if (metricType._tag === "Counter") {
    yield* Effect.log(`Counter value: ${currentState.count}`)
  } else if (metricType._tag === "Gauge") {
    yield* Effect.log(`Gauge value: ${currentState.value}`)
  }
})
```

### Pattern 3: Snapshot Collection

```typescript
// Collecting multiple metric pairs as a snapshot
const createMetricsSnapshot = Effect.gen(function* () {
  // Get all metric pairs from the registry
  const snapshot = yield* Metric.snapshot
  
  yield* Effect.log(`Collected ${snapshot.length} metrics`)
  
  // Process each metric pair
  yield* Effect.forEach(snapshot, inspectMetricPair)
  
  return snapshot
})
```

## Real-World Examples

### Example 1: Application Performance Monitoring

Building a comprehensive APM system that tracks request metrics with type-safe state management:

```typescript
import { Effect, Metric, MetricKey, MetricPair, MetricState, Duration, HashMap } from "effect"

// Define application metrics with proper types
class AppMetrics {
  static readonly requestsTotal = MetricKey.counter("http_requests_total", {
    description: "Total HTTP requests processed"
  })
  
  static readonly requestDuration = MetricKey.histogram("http_request_duration_seconds", {
    description: "HTTP request duration in seconds",
    boundaries: [0.1, 0.5, 1.0, 2.0, 5.0]
  })
  
  static readonly activeConnections = MetricKey.gauge("http_active_connections", {
    description: "Currently active HTTP connections"
  })
}

// Service that collects performance metrics
const makePerformanceCollector = Effect.gen(function* () {
  const collectMetrics = Effect.gen(function* () {
    const snapshot = yield* Metric.snapshot
    
    // Group metrics by type for structured reporting
    const metricsByType = HashMap.empty<string, MetricPair.MetricPair.Untyped[]>()
    
    for (const pair of snapshot) {
      const metricType = pair.metricKey.keyType._tag
      const existing = HashMap.get(metricsByType, metricType)
      
      if (existing._tag === "Some") {
        HashMap.set(metricsByType, metricType, [...existing.value, pair])
      } else {
        HashMap.set(metricsByType, metricType, [pair])
      }
    }
    
    return metricsByType
  })
  
  const exportPrometheusFormat = (pairs: MetricPair.MetricPair.Untyped[]) => Effect.gen(function* () {
    const lines: string[] = []
    
    for (const pair of pairs) {
      const { metricKey, metricState } = pair
      const metricType = metricKey.keyType
      
      // Type-safe export based on metric type
      switch (metricType._tag) {
        case "Counter": {
          lines.push(`# TYPE ${metricKey.name} counter`)
          lines.push(`${metricKey.name} ${metricState.count}`)
          break
        }
        case "Gauge": {
          lines.push(`# TYPE ${metricKey.name} gauge`)
          lines.push(`${metricKey.name} ${metricState.value}`)
          break
        }
        case "Histogram": {
          lines.push(`# TYPE ${metricKey.name} histogram`)
          // Export histogram buckets and count
          for (const [boundary, count] of metricState.buckets) {
            lines.push(`${metricKey.name}_bucket{le="${boundary}"} ${count}`)
          }
          lines.push(`${metricKey.name}_count ${metricState.count}`)
          lines.push(`${metricKey.name}_sum ${metricState.sum}`)
          break
        }
      }
    }
    
    return lines.join('\n')
  })
  
  return { collectMetrics, exportPrometheusFormat } as const
})
```

### Example 2: Database Connection Pool Monitoring

Creating a database monitoring system that tracks connection pool metrics:

```typescript
import { Effect, Metric, MetricKey, MetricPair, MetricState, Schedule, Layer, Context } from "effect"

// Database metrics definition
class DatabaseMetrics {
  static readonly poolSize = MetricKey.gauge("db_connection_pool_size", {
    description: "Current database connection pool size"
  })
  
  static readonly activeConnections = MetricKey.gauge("db_active_connections", {
    description: "Active database connections"
  })
  
  static readonly queryDuration = MetricKey.histogram("db_query_duration_seconds", {
    description: "Database query execution time",
    boundaries: [0.001, 0.01, 0.1, 1.0, 10.0]
  })
  
  static readonly queryErrors = MetricKey.counter("db_query_errors_total", {
    description: "Total database query errors"
  }).pipe(
    MetricKey.tagged("error_type", "connection_timeout")
  )
}

// Database monitoring service
interface DatabaseMonitor {
  readonly collectPoolMetrics: Effect.Effect<MetricPair.MetricPair.Untyped[]>
  readonly startPeriodicCollection: Effect.Effect<void>
}

const DatabaseMonitor = Context.GenericTag<DatabaseMonitor>("@app/DatabaseMonitor")

const makeDatabaseMonitor = Effect.gen(function* () {
  const collectPoolMetrics = Effect.gen(function* () {
    // Simulate collecting actual database metrics
    const poolSize = yield* Effect.succeed(10)
    const activeConnections = yield* Effect.succeed(7)
    
    // Create metric pairs with current state
    const poolSizePair = MetricPair.make(
      DatabaseMetrics.poolSize,
      MetricState.gauge(poolSize)
    )
    
    const activeConnectionsPair = MetricPair.make(
      DatabaseMetrics.activeConnections,
      MetricState.gauge(activeConnections)
    )
    
    return [poolSizePair, activeConnectionsPair]
  })
  
  const startPeriodicCollection = Effect.gen(function* () {
    const collector = collectPoolMetrics.pipe(
      Effect.flatMap((pairs) => Effect.gen(function* () {
        yield* Effect.log(`Collected ${pairs.length} database metrics`)
        
        // Log each metric pair for debugging
        yield* Effect.forEach(pairs, (pair) => Effect.gen(function* () {
          const name = pair.metricKey.name
          const type = pair.metricKey.keyType._tag
          yield* Effect.log(`${name} (${type}): ${JSON.stringify(pair.metricState)}`)
        }))
      })),
      Effect.repeat(Schedule.fixed(Duration.seconds(30))) // Collect every 30 seconds
    )
    
    yield* collector
  })
  
  return DatabaseMonitor.of({ collectPoolMetrics, startPeriodicCollection })
})

// Layer for the database monitor
const DatabaseMonitorLive = Layer.effect(DatabaseMonitor, makeDatabaseMonitor)
```

### Example 3: Business Metrics Dashboard

Building a business intelligence system that tracks KPIs using MetricPair:

```typescript
import { Effect, Metric, MetricKey, MetricPair, MetricState, Array as Arr, Option, DateTime } from "effect"

// Business KPI metrics
class BusinessMetrics {
  static readonly revenue = MetricKey.gauge("business_revenue_usd", {
    description: "Current revenue in USD"
  }).pipe(
    MetricKey.tagged("period", "daily")
  )
  
  static readonly customerCount = MetricKey.gauge("business_customers_total", {
    description: "Total number of customers"
  })
  
  static readonly orderValue = MetricKey.histogram("business_order_value_usd", {
    description: "Order value distribution in USD",
    boundaries: [10, 50, 100, 500, 1000, 5000]
  })
}

// Business metrics collector with MetricPair management
const makeBusinessDashboard = Effect.gen(function* () {
  const createKPISnapshot = Effect.gen(function* () {
    const allMetrics = yield* Metric.snapshot
    
    // Filter to business metrics only
    const businessMetrics = Arr.filter(allMetrics, (pair) =>
      pair.metricKey.name.startsWith("business_")
    )
    
    return businessMetrics
  })
  
  const calculateKPIs = (metrics: MetricPair.MetricPair.Untyped[]) => Effect.gen(function* () {
    const kpis: Record<string, any> = {}
    
    for (const pair of metrics) {
      const { metricKey, metricState } = pair
      
      switch (metricKey.name) {
        case "business_revenue_usd": {
          kpis.revenue = metricState.value
          break
        }
        case "business_customers_total": {
          kpis.customerCount = metricState.value
          break
        }
        case "business_order_value_usd": {
          // Calculate average order value from histogram
          const totalOrders = metricState.count
          const totalValue = metricState.sum
          kpis.averageOrderValue = totalOrders > 0 ? totalValue / totalOrders : 0
          break
        }
      }
    }
    
    return kpis
  })
  
  const generateDashboardData = Effect.gen(function* () {
    const snapshot = yield* createKPISnapshot
    const kpis = yield* calculateKPIs(snapshot)
    const timestamp = yield* Effect.succeed(DateTime.now())
    
    return {
      timestamp,
      kpis,
      metricCount: snapshot.length,
      rawMetrics: snapshot
    }
  })
  
  return { createKPISnapshot, calculateKPIs, generateDashboardData } as const
})
```

## Advanced Features Deep Dive

### Feature 1: Type-Safe State Management

MetricPair ensures that metric state always matches the expected type for the metric key.

#### Basic Type Safety

```typescript
import { MetricKey, MetricPair, MetricState } from "effect"

// Type system prevents mismatched state
const safeMetricCreation = () => {
  const counterKey = MetricKey.counter("requests")
  const gaugeKey = MetricKey.gauge("memory")
  
  // ✅ Correct: Counter state with counter key
  const validPair = MetricPair.make(counterKey, MetricState.counter(10))
  
  // ❌ Compiler error: Gauge state with counter key
  // const invalidPair = MetricPair.make(counterKey, MetricState.gauge(10))
  
  return validPair
}
```

#### Advanced Type Inference

```typescript
// Helper for creating typed metric pairs
const createTypedMetricPair = <T extends MetricKeyType.MetricKeyType<any, any>>(
  key: MetricKey.MetricKey<T>,
  stateValue: MetricKeyType.MetricKeyType.InType<T>
) => Effect.gen(function* () {
  // State type is automatically inferred from key type
  const state = yield* Effect.succeed(
    (() => {
      switch (key.keyType._tag) {
        case "Counter":
          return MetricState.counter(stateValue as number)
        case "Gauge":
          return MetricState.gauge(stateValue as number)
        case "Histogram":
          return MetricState.histogram({
            buckets: [[1, 0], [2, 0], [5, 0]],
            count: 0,
            min: 0,
            max: 0,
            sum: 0
          })
        default:
          throw new Error(`Unsupported metric type: ${key.keyType._tag}`)
      }
    })()
  )
  
  return MetricPair.make(key, state)
})
```

### Feature 2: Unsafe Construction for Dynamic Scenarios

Sometimes you need to create MetricPairs when types can't be statically verified.

#### Unsafe Creation Pattern

```typescript
// For dynamic metric creation where types are resolved at runtime
const createDynamicMetricPair = (
  metricName: string,
  metricType: string,
  value: unknown
) => Effect.gen(function* () {
  const key = yield* Effect.gen(function* () {
    switch (metricType) {
      case "counter":
        return MetricKey.counter(metricName)
      case "gauge":
        return MetricKey.gauge(metricName)
      default:
        return yield* Effect.fail(new Error(`Unknown metric type: ${metricType}`))
    }
  })
  
  const state = yield* Effect.gen(function* () {
    switch (metricType) {
      case "counter":
        return MetricState.counter(value as number)
      case "gauge":
        return MetricState.gauge(value as number)
      default:
        return yield* Effect.fail(new Error(`Unknown metric type: ${metricType}`))
    }
  })
  
  // Use unsafeMake when types can't be statically verified
  return MetricPair.unsafeMake(key, state)
})
```

#### Runtime Validation Pattern

```typescript
// Validate metric pairs at runtime
const validateMetricPair = (pair: MetricPair.MetricPair.Untyped) => Effect.gen(function* () {
  const keyType = pair.metricKey.keyType._tag
  const stateType = pair.metricState._tag
  
  // Ensure state type matches key type
  const isValid = (() => {
    switch (keyType) {
      case "Counter":
        return stateType === "CounterState"
      case "Gauge":
        return stateType === "GaugeState"
      case "Histogram":
        return stateType === "HistogramState"
      default:
        return false
    }
  })()
  
  if (!isValid) {
    return yield* Effect.fail(
      new Error(`Metric state type '${stateType}' doesn't match key type '${keyType}'`)
    )
  }
  
  return pair
})
```

## Practical Patterns & Best Practices

### Pattern 1: Metric Registry Integration

```typescript
// Helper for working with metric registries and pairs
const createMetricRegistryHelpers = Effect.gen(function* () {
  const snapshotByType = Effect.gen(function* () {
    const snapshot = yield* Metric.snapshot
    
    const grouped = Arr.groupBy(snapshot, (pair) => pair.metricKey.keyType._tag)
    
    return {
      counters: grouped["Counter"] || [],
      gauges: grouped["Gauge"] || [],
      histograms: grouped["Histogram"] || [],
      summaries: grouped["Summary"] || [],
      frequencies: grouped["Frequency"] || []
    }
  })
  
  const findMetricPair = (name: string) => Effect.gen(function* () {
    const snapshot = yield* Metric.snapshot
    
    return Arr.findFirst(snapshot, (pair) => pair.metricKey.name === name)
  })
  
  const filterByTag = (tagKey: string, tagValue: string) => Effect.gen(function* () {
    const snapshot = yield* Metric.snapshot
    
    return Arr.filter(snapshot, (pair) => {
      const tags = pair.metricKey.tags
      return Arr.some(tags, tag => tag.key === tagKey && tag.value === tagValue)
    })
  })
  
  return { snapshotByType, findMetricPair, filterByTag } as const
})
```

### Pattern 2: Metric Serialization and Export

```typescript
// Comprehensive metric export system
const createMetricExporter = Effect.gen(function* () {
  const toJSON = (pairs: MetricPair.MetricPair.Untyped[]) => Effect.gen(function* () {
    const jsonMetrics = Arr.map(pairs, (pair) => ({
      name: pair.metricKey.name,
      type: pair.metricKey.keyType._tag,
      description: pair.metricKey.description,
      tags: pair.metricKey.tags,
      state: pair.metricState,
      timestamp: DateTime.now()
    }))
    
    return JSON.stringify(jsonMetrics, null, 2)
  })
  
  const toPrometheus = (pairs: MetricPair.MetricPair.Untyped[]) => Effect.gen(function* () {
    const lines: string[] = []
    
    for (const pair of pairs) {
      const { metricKey, metricState } = pair
      const name = metricKey.name
      const type = metricKey.keyType._tag.toLowerCase()
      
      // Add help comment
      if (metricKey.description) {
        lines.push(`# HELP ${name} ${metricKey.description}`)
      }
      
      // Add type comment
      lines.push(`# TYPE ${name} ${type}`)
      
      // Add metric value(s)
      switch (type) {
        case "counter":
          lines.push(`${name} ${metricState.count}`)
          break
        case "gauge":
          lines.push(`${name} ${metricState.value}`)
          break
        case "histogram":
          // Export histogram buckets
          for (const [boundary, count] of metricState.buckets) {
            lines.push(`${name}_bucket{le="${boundary}"} ${count}`)
          }
          lines.push(`${name}_bucket{le="+Inf"} ${metricState.count}`)
          lines.push(`${name}_count ${metricState.count}`)
          lines.push(`${name}_sum ${metricState.sum}`)
          break
      }
      
      lines.push("") // Empty line between metrics
    }
    
    return lines.join("\n")
  })
  
  return { toJSON, toPrometheus } as const
})
```

## Integration Examples

### Integration with HTTP Monitoring

```typescript
import { Effect, Layer, HttpServer, MetricPair } from "effect"

// HTTP metrics middleware that creates metric pairs
const createHttpMetricsMiddleware = Effect.gen(function* () {
  const httpMetrics = {
    requests: MetricKey.counter("http_requests_total").pipe(
      MetricKey.tagged("method", "GET"),
      MetricKey.tagged("status", "200")
    ),
    duration: MetricKey.histogram("http_request_duration_seconds", {
      boundaries: [0.1, 0.5, 1.0, 2.0, 5.0]
    })
  }
  
  const middleware = (request: HttpServer.Request) => Effect.gen(function* () {
    const startTime = yield* Effect.succeed(Date.now())
    
    // Process request (simplified)
    const response = yield* Effect.succeed({ status: 200, body: "OK" })
    
    const duration = (Date.now() - startTime) / 1000
    
    // Record metrics
    yield* Metric.increment(Metric.fromMetricKey(httpMetrics.requests))
    yield* Metric.update(Metric.fromMetricKey(httpMetrics.duration), duration)
    
    // Create snapshot including our metrics
    const snapshot = yield* Metric.snapshot
    const relevantPairs = Arr.filter(snapshot, (pair) => 
      pair.metricKey.name.startsWith("http_")
    )
    
    yield* Effect.log(`HTTP metrics collected: ${relevantPairs.length} pairs`)
    
    return response
  })
  
  return { middleware, httpMetrics } as const
})
```

### Integration with Custom Metric Storage

```typescript
// Custom storage backend for metric pairs
interface MetricStorage {
  readonly store: (pairs: MetricPair.MetricPair.Untyped[]) => Effect.Effect<void>
  readonly retrieve: (namePattern?: string) => Effect.Effect<MetricPair.MetricPair.Untyped[]>
}

const MetricStorage = Context.GenericTag<MetricStorage>("@app/MetricStorage")

const makeInMemoryMetricStorage = Effect.gen(function* () {
  const storage = yield* Effect.succeed(new Map<string, MetricPair.MetricPair.Untyped>())
  
  const store = (pairs: MetricPair.MetricPair.Untyped[]) => Effect.gen(function* () {
    for (const pair of pairs) {
      storage.set(pair.metricKey.name, pair)
    }
    yield* Effect.log(`Stored ${pairs.length} metric pairs`)
  })
  
  const retrieve = (namePattern?: string) => Effect.gen(function* () {
    const allPairs = Array.from(storage.values())
    
    if (!namePattern) {
      return allPairs
    }
    
    const regex = new RegExp(namePattern)
    return Arr.filter(allPairs, (pair) => regex.test(pair.metricKey.name))
  })
  
  return MetricStorage.of({ store, retrieve })
})

// Service that periodically stores metrics
const makeMetricPersistence = Effect.gen(function* () {
  const storage = yield* MetricStorage
  
  const persistMetrics = Effect.gen(function* () {
    const snapshot = yield* Metric.snapshot
    yield* storage.store(snapshot)
  }).pipe(
    Effect.repeat(Schedule.fixed(Duration.minutes(1)))
  )
  
  return { persistMetrics }
})
```

### Testing Strategies

```typescript
// Test utilities for MetricPair
const createMetricTestHelpers = Effect.gen(function* () {
  const createTestMetricPair = (
    name: string,
    type: "counter" | "gauge" | "histogram",
    value: number
  ) => Effect.gen(function* () {
    const key = (() => {
      switch (type) {
        case "counter":
          return MetricKey.counter(name)
        case "gauge":
          return MetricKey.gauge(name)
        case "histogram":
          return MetricKey.histogram(name)
      }
    })()
    
    const state = (() => {
      switch (type) {
        case "counter":
          return MetricState.counter(value)
        case "gauge":
          return MetricState.gauge(value)
        case "histogram":
          const hist = MetricState.histogram()
          // Update histogram with value
          return hist
      }
    })()
    
    return MetricPair.make(key, state)
  })
  
  const assertMetricPair = (
    pair: MetricPair.MetricPair.Untyped,
    expectedName: string,
    expectedType: string
  ) => Effect.gen(function* () {
    const actualName = pair.metricKey.name
    const actualType = pair.metricKey.keyType._tag
    
    if (actualName !== expectedName) {
      return yield* Effect.fail(
        new Error(`Expected name '${expectedName}', got '${actualName}'`)
      )
    }
    
    if (actualType !== expectedType) {
      return yield* Effect.fail(
        new Error(`Expected type '${expectedType}', got '${actualType}'`)
      )
    }
    
    yield* Effect.log(`✓ MetricPair assertion passed: ${expectedName} (${expectedType})`)
  })
  
  const mockMetricSnapshot = (pairs: MetricPair.MetricPair.Untyped[]) => 
    Layer.succeed(Metric, {
      snapshot: Effect.succeed(pairs)
    } as any)
  
  return { createTestMetricPair, assertMetricPair, mockMetricSnapshot } as const
})

// Example test
const testMetricPairCreation = Effect.gen(function* () {
  const helpers = yield* createMetricTestHelpers
  
  // Test counter metric pair
  const counterPair = yield* helpers.createTestMetricPair("test_counter", "counter", 42)
  yield* helpers.assertMetricPair(counterPair, "test_counter", "Counter")
  
  // Test gauge metric pair
  const gaugePair = yield* helpers.createTestMetricPair("test_gauge", "gauge", 3.14)
  yield* helpers.assertMetricPair(gaugePair, "test_gauge", "Gauge")
  
  yield* Effect.log("All MetricPair tests passed!")
})
```

## Conclusion

MetricPair provides type-safe association between metric definitions and their state values, enabling robust monitoring systems in Effect applications.

Key benefits:
- **Type Safety**: Compile-time verification that metric state matches key type
- **Simplified Collection**: Unified interface for collecting metrics regardless of type
- **Registry Integration**: Seamless integration with Effect's metric registry system

MetricPair is essential when building monitoring systems, dashboards, or any application that needs to collect and export metrics with type safety and structural integrity.