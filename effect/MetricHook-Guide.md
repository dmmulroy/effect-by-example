# MetricHook: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem MetricHook Solves

When building observability into applications, developers often struggle with creating flexible, extensible metric collection systems. Traditional approaches lead to rigid metric implementations that are difficult to customize or extend:

```typescript
// Traditional approach - rigid metric collection
class SimpleCounter {
  private value = 0
  
  increment(amount: number): void {
    this.value += amount
    // Hard-coded behavior - no extensibility
    console.log(`Counter updated: ${this.value}`)
  }
  
  getValue(): number {
    return this.value
  }
}

// Problems with traditional approach:
const counter = new SimpleCounter()
counter.increment(5)
// - Cannot add custom logic during updates
// - Cannot compose multiple behaviors
// - Difficult to add monitoring, filtering, or transformation
// - No standardized interface for different metric types
```

This approach leads to:
- **Inflexibility** - Cannot customize metric behavior without modifying core logic
- **Poor Composability** - Cannot combine multiple metric behaviors or transformations
- **Maintenance Overhead** - Each metric type requires separate implementation and customization
- **Limited Extensibility** - Adding new behaviors requires modifying existing code

### The MetricHook Solution

MetricHook provides a powerful abstraction for creating customizable, composable metric collection hooks that can intercept and transform metric updates:

```typescript
import { MetricHook, MetricKey, MetricKeyType } from "effect"

// Create a flexible counter hook with custom behavior
const counterKey = MetricKey.counter("api_requests")

const counterHook = MetricHook.counter(counterKey).pipe(
  MetricHook.onUpdate((value) => {
    // Custom logic on every update
    console.log(`API request counter updated by: ${value}`)
  })
)

// Usage is clean and composable
counterHook.update(1) // Logs: "API request counter updated by: 1"
counterHook.update(3) // Logs: "API request counter updated by: 3"
const currentValue = counterHook.get() // Returns current counter state
```

### Key Concepts

**MetricHook<In, Out>**: A stateful metric aggregator that accepts updates of type `In` and maintains state of type `Out`, with customizable behavior through composition.

**Hook Composition**: Combine multiple behaviors using `.pipe()` with `onUpdate` and `onModify` to create sophisticated metric collection systems.

**Type Safety**: Strong typing ensures that updates match the expected input type and state queries return the correct output type.

## Basic Usage Patterns

### Pattern 1: Counter Hook Setup

```typescript
import { MetricHook, MetricKey, MetricKeyType } from "effect"

// Create a counter metric key
const requestCounterKey = MetricKey.counter("http_requests", {
  description: "Total HTTP requests processed"
})

// Create the hook
const requestCounterHook = MetricHook.counter(requestCounterKey)

// Basic usage
requestCounterHook.update(1) // Increment by 1
requestCounterHook.update(5) // Increment by 5

// Get current state
const currentCount = requestCounterHook.get()
console.log(`Total requests: ${currentCount.count}`)
```

### Pattern 2: Gauge Hook with Initial Value

```typescript
import { MetricHook, MetricKey } from "effect"

// Create gauge for active connections
const connectionsKey = MetricKey.gauge("active_connections", {
  description: "Current number of active connections"
})

// Initialize gauge with starting value
const connectionsHook = MetricHook.gauge(connectionsKey, 0)

// Update to specific value
connectionsHook.update(10) // Set to 10 connections

// Modify by delta (relative change)
connectionsHook.modify(5)  // Add 5 connections (now 15)
connectionsHook.modify(-3) // Remove 3 connections (now 12)

const currentConnections = connectionsHook.get()
console.log(`Active connections: ${currentConnections.value}`)
```

### Pattern 3: Histogram Hook for Latency Tracking

```typescript
import { MetricHook, MetricKey, MetricBoundaries } from "effect"

// Define latency buckets in milliseconds
const latencyBoundaries = MetricBoundaries.linear({ 
  start: 0, 
  width: 50, 
  count: 10 
}) // Creates buckets: 0, 50, 100, 150, ..., 450ms

const latencyKey = MetricKey.histogram("request_latency_ms", latencyBoundaries, {
  description: "HTTP request latency distribution"
})

const latencyHook = MetricHook.histogram(latencyKey)

// Record latency measurements
latencyHook.update(23)  // 23ms request
latencyHook.update(156) // 156ms request
latencyHook.update(89)  // 89ms request

const latencyStats = latencyHook.get()
console.log(`Total requests: ${latencyStats.count}`)
console.log(`Average latency: ${latencyStats.sum / latencyStats.count}ms`)
console.log(`Min latency: ${latencyStats.min}ms`)
console.log(`Max latency: ${latencyStats.max}ms`)
```

## Real-World Examples

### Example 1: API Gateway Metrics with Custom Monitoring

Building a comprehensive API gateway monitoring system with custom alerting and logging:

```typescript
import { MetricHook, MetricKey, Effect, Console } from "effect"

// Define metrics for API gateway
interface ApiGatewayMetrics {
  readonly requestCounter: MetricHook.Counter<number>
  readonly responseTimeHistogram: MetricHook.Histogram
  readonly errorCounter: MetricHook.Counter<number>
  readonly activeConnectionsGauge: MetricHook.Gauge<number>
}

const createApiGatewayMetrics = (): ApiGatewayMetrics => {
  // Request counter with alerting
  const requestCounter = MetricHook.counter(
    MetricKey.counter("api_requests_total")
  ).pipe(
    MetricHook.onUpdate((count) => {
      if (count > 100) {
        console.log(`HIGH TRAFFIC ALERT: ${count} requests in burst`)
      }
    })
  )

  // Response time histogram with percentile monitoring
  const responseTimeHistogram = MetricHook.histogram(
    MetricKey.histogram("api_response_time_ms", 
      MetricBoundaries.exponential({ start: 1, factor: 2, count: 12 })
    )
  ).pipe(
    MetricHook.onUpdate((responseTime) => {
      if (responseTime > 5000) {
        console.error(`SLOW RESPONSE ALERT: ${responseTime}ms`)
      }
    })
  )

  // Error counter with threshold monitoring
  const errorCounter = MetricHook.counter(
    MetricKey.counter("api_errors_total")
  ).pipe(
    MetricHook.onUpdate((errorCount) => {
      console.warn(`Error occurred. Total errors: ${errorCount}`)
    })
  )

  // Active connections gauge
  const activeConnectionsGauge = MetricHook.gauge(
    MetricKey.gauge("api_active_connections"), 
    0
  )

  return {
    requestCounter,
    responseTimeHistogram,
    errorCounter,
    activeConnectionsGauge
  }
}

// Usage in API request handler
const handleApiRequest = (metrics: ApiGatewayMetrics) => {
  return Effect.gen(function* () {
    const startTime = Date.now()
    
    // Increment active connections
    metrics.activeConnectionsGauge.modify(1)
    
    try {
      // Increment request counter
      metrics.requestCounter.update(1)
      
      // Simulate API processing
      yield* Effect.sleep("100 millis")
      
      // Record response time
      const responseTime = Date.now() - startTime
      metrics.responseTimeHistogram.update(responseTime)
      
      return { status: "success", data: "API response" }
    } catch (error) {
      // Record error
      metrics.errorCounter.update(1)
      throw error
    } finally {
      // Decrement active connections
      metrics.activeConnectionsGauge.modify(-1)
    }
  })
}

// Create and use metrics
const metrics = createApiGatewayMetrics()

// Simulate API requests
Effect.runPromise(
  Effect.gen(function* () {
    yield* handleApiRequest(metrics)
    yield* handleApiRequest(metrics)
    yield* handleApiRequest(metrics)
    
    // Get current metrics
    const requests = metrics.requestCounter.get()
    const connections = metrics.activeConnectionsGauge.get()
    const responseStats = metrics.responseTimeHistogram.get()
    
    yield* Console.log(`Total requests: ${requests.count}`)
    yield* Console.log(`Active connections: ${connections.value}`)
    yield* Console.log(`Average response time: ${responseStats.sum / responseStats.count}ms`)
  })
)
```

### Example 2: Database Connection Pool Monitoring

Creating a comprehensive database connection pool monitoring system:

```typescript
import { MetricHook, MetricKey, MetricBoundaries, Effect } from "effect"

interface DatabaseMetrics {
  readonly connectionPoolSize: MetricHook.Gauge<number>
  readonly activeQueries: MetricHook.Gauge<number>
  readonly queryDuration: MetricHook.Histogram
  readonly connectionErrors: MetricHook.Counter<number>
  readonly queryTypes: MetricHook.Frequency
}

const createDatabaseMetrics = (): DatabaseMetrics => {
  return {
    connectionPoolSize: MetricHook.gauge(
      MetricKey.gauge("db_connection_pool_size"), 
      0
    ).pipe(
      MetricHook.onUpdate((size) => {
        if (size > 80) {
          console.warn(`Connection pool nearly full: ${size}/100`)
        }
      })
    ),

    activeQueries: MetricHook.gauge(
      MetricKey.gauge("db_active_queries"), 
      0
    ),

    queryDuration: MetricHook.histogram(
      MetricKey.histogram("db_query_duration_ms",
        MetricBoundaries.exponential({ start: 1, factor: 1.5, count: 15 })
      )
    ).pipe(
      MetricHook.onUpdate((duration) => {
        if (duration > 10000) {
          console.error(`SLOW QUERY ALERT: ${duration}ms`)
        }
      })
    ),

    connectionErrors: MetricHook.counter(
      MetricKey.counter("db_connection_errors")
    ).pipe(
      MetricHook.onUpdate((errors) => {
        console.error(`Database connection error occurred. Total: ${errors}`)
      })
    ),

    queryTypes: MetricHook.frequency(
      MetricKey.frequency("db_query_types", ["SELECT", "INSERT", "UPDATE", "DELETE"])
    )
  }
}

// Database connection pool simulator
class DatabasePool {
  constructor(private metrics: DatabaseMetrics) {}

  executeQuery = (queryType: string, query: string) => {
    return Effect.gen(function* () {
      const startTime = Date.now()
      
      // Track active query
      this.metrics.activeQueries.modify(1)
      
      // Record query type
      this.metrics.queryTypes.update(queryType)
      
      try {
        // Simulate query execution
        const executionTime = Math.random() * 1000 + 50
        yield* Effect.sleep(`${executionTime} millis`)
        
        // Record query duration
        const duration = Date.now() - startTime
        this.metrics.queryDuration.update(duration)
        
        return { result: "Query executed successfully" }
      } catch (error) {
        this.metrics.connectionErrors.update(1)
        throw error
      } finally {
        // Decrease active queries
        this.metrics.activeQueries.modify(-1)
      }
    })
  }

  addConnection = () => {
    this.metrics.connectionPoolSize.modify(1)
  }

  removeConnection = () => {
    this.metrics.connectionPoolSize.modify(-1)
  }
}

// Usage example
const dbMetrics = createDatabaseMetrics()
const dbPool = new DatabasePool(dbMetrics)

Effect.runPromise(
  Effect.gen(function* () {
    // Initialize connection pool
    dbPool.addConnection()
    dbPool.addConnection()
    dbPool.addConnection()
    
    // Execute various queries
    yield* dbPool.executeQuery("SELECT", "SELECT * FROM users")
    yield* dbPool.executeQuery("INSERT", "INSERT INTO users ...")
    yield* dbPool.executeQuery("UPDATE", "UPDATE users SET ...")
    yield* dbPool.executeQuery("SELECT", "SELECT COUNT(*) FROM orders")
    
    // Get metrics summary
    const poolSize = dbMetrics.connectionPoolSize.get()
    const queryStats = dbMetrics.queryDuration.get()
    const queryFrequencies = dbMetrics.queryTypes.get()
    
    console.log(`Pool size: ${poolSize.value}`)
    console.log(`Total queries: ${queryStats.count}`)
    console.log(`Average query time: ${queryStats.sum / queryStats.count}ms`)
    console.log("Query type frequencies:", queryFrequencies.occurrences)
  })
)
```

### Example 3: Real-Time Application Performance Monitoring

Building a comprehensive APM system for monitoring application performance:

```typescript
import { MetricHook, MetricKey, Effect, Schedule } from "effect"

interface ApplicationMetrics {
  readonly memoryUsage: MetricHook.Gauge<number>
  readonly cpuUsage: MetricHook.Gauge<number>
  readonly requestThroughput: MetricHook.Counter<number>
  readonly errorRate: MetricHook.Counter<number>
  readonly responseTimePercentiles: MetricHook.Summary
}

const createApplicationMetrics = (): ApplicationMetrics => {
  return {
    memoryUsage: MetricHook.gauge(
      MetricKey.gauge("app_memory_usage_mb"), 
      0
    ).pipe(
      MetricHook.onUpdate((usage) => {
        if (usage > 1000) {
          console.warn(`HIGH MEMORY USAGE: ${usage}MB`)
        }
      })
    ),

    cpuUsage: MetricHook.gauge(
      MetricKey.gauge("app_cpu_usage_percent"), 
      0
    ).pipe(
      MetricHook.onUpdate((usage) => {
        if (usage > 80) {
          console.warn(`HIGH CPU USAGE: ${usage}%`)
        }
      })
    ),

    requestThroughput: MetricHook.counter(
      MetricKey.counter("app_requests_per_second")
    ),

    errorRate: MetricHook.counter(
      MetricKey.counter("app_errors_per_second")
    ),

    responseTimePercentiles: MetricHook.summary(
      MetricKey.summary("app_response_time_ms", {
        maxAge: "60 seconds",
        maxSize: 1000,
        error: 0.01,
        quantiles: [0.5, 0.75, 0.95, 0.99]
      })
    )
  }
}

// Performance monitoring service
class PerformanceMonitor {
  constructor(private metrics: ApplicationMetrics) {}

  // Collect system metrics
  collectSystemMetrics = Effect.gen(function* () {
    // Simulate memory usage collection
    const memoryUsage = Math.random() * 800 + 200 // 200-1000 MB
    this.metrics.memoryUsage.update(memoryUsage)

    // Simulate CPU usage collection
    const cpuUsage = Math.random() * 60 + 20 // 20-80%
    this.metrics.cpuUsage.update(cpuUsage)
  })

  // Track request performance
  trackRequest = (responseTime: number, isError: boolean = false) => {
    this.metrics.requestThroughput.update(1)
    this.metrics.responseTimePercentiles.update([responseTime, Date.now()])
    
    if (isError) {
      this.metrics.errorRate.update(1)
    }
  }

  // Generate performance report
  generateReport = Effect.gen(function* () {
    const memory = this.metrics.memoryUsage.get()
    const cpu = this.metrics.cpuUsage.get()
    const throughput = this.metrics.requestThroughput.get()
    const errors = this.metrics.errorRate.get()
    const responseTime = this.metrics.responseTimePercentiles.get()

    return {
      timestamp: new Date().toISOString(),
      system: {
        memoryUsageMB: memory.value,
        cpuUsagePercent: cpu.value
      },
      requests: {
        totalRequests: throughput.count,
        totalErrors: errors.count,
        errorRate: errors.count / throughput.count * 100,
        responseTime: {
          count: responseTime.count,
          min: responseTime.min,
          max: responseTime.max,
          average: responseTime.sum / responseTime.count,
          percentiles: Object.fromEntries(responseTime.quantiles)
        }
      }
    }
  })
}

// Usage in application monitoring
const appMetrics = createApplicationMetrics()
const performanceMonitor = new PerformanceMonitor(appMetrics)

// Simulate application monitoring
const monitoringProgram = Effect.gen(function* () {
  // Start periodic system metrics collection
  const systemMetricsSchedule = Schedule.spaced("5 seconds")
  
  yield* Effect.fork(
    performanceMonitor.collectSystemMetrics.pipe(
      Effect.repeat(systemMetricsSchedule)
    )
  )

  // Simulate incoming requests with varying performance
  for (let i = 0; i < 50; i++) {
    const responseTime = Math.random() * 500 + 50 // 50-550ms
    const isError = Math.random() < 0.05 // 5% error rate
    
    performanceMonitor.trackRequest(responseTime, isError)
    
    yield* Effect.sleep("100 millis")
  }

  // Generate and display performance report
  const report = yield* performanceMonitor.generateReport
  
  console.log("=== Performance Report ===")
  console.log(JSON.stringify(report, null, 2))
})

Effect.runPromise(monitoringProgram)
```

## Advanced Features Deep Dive

### Feature 1: Custom MetricHook Creation

Create completely custom metric hooks for specialized use cases:

#### Basic Custom Hook Usage

```typescript
import { MetricHook } from "effect"

// Create a custom "Top-K" hook that tracks the most frequent values
const createTopKHook = <T>(k: number) => {
  const frequencies = new Map<T, number>()
  
  return MetricHook.make<T, Map<T, number>>({
    get: () => frequencies,
    update: (value: T) => {
      const current = frequencies.get(value) ?? 0
      frequencies.set(value, current + 1)
      
      // Keep only top K values
      if (frequencies.size > k) {
        const sorted = Array.from(frequencies.entries())
          .sort(([, a], [, b]) => b - a)
          .slice(0, k)
        
        frequencies.clear()
        sorted.forEach(([key, val]) => frequencies.set(key, val))
      }
    },
    modify: (value: T) => {
      // Same as update for this use case
      const current = frequencies.get(value) ?? 0
      frequencies.set(value, current + 1)
    }
  })
}

// Usage
const topUrlsHook = createTopKHook<string>(5)

// Track URL hits
topUrlsHook.update("/api/users")
topUrlsHook.update("/api/orders")
topUrlsHook.update("/api/users")
topUrlsHook.update("/api/products")
topUrlsHook.update("/api/users")

const topUrls = topUrlsHook.get()
console.log("Top URLs:", Array.from(topUrls.entries()))
```

#### Advanced Custom Hook: Sliding Window Statistics

```typescript
import { MetricHook } from "effect"

interface SlidingWindowStats {
  readonly count: number
  readonly sum: number
  readonly average: number
  readonly min: number
  readonly max: number
  readonly values: ReadonlyArray<number>
}

const createSlidingWindowHook = (windowSize: number) => {
  const values: number[] = []
  
  const calculateStats = (): SlidingWindowStats => {
    if (values.length === 0) {
      return {
        count: 0,
        sum: 0,
        average: 0,
        min: 0,
        max: 0,
        values: []
      }
    }
    
    const sum = values.reduce((acc, val) => acc + val, 0)
    return {
      count: values.length,
      sum,
      average: sum / values.length,
      min: Math.min(...values),
      max: Math.max(...values),
      values: [...values]
    }
  }
  
  return MetricHook.make<number, SlidingWindowStats>({
    get: calculateStats,
    update: (value: number) => {
      values.push(value)
      if (values.length > windowSize) {
        values.shift() // Remove oldest value
      }
    },
    modify: (value: number) => {
      // For sliding window, modify behaves same as update
      values.push(value)
      if (values.length > windowSize) {
        values.shift()
      }
    }
  })
}

// Usage for tracking last 10 response times
const responseTimeWindow = createSlidingWindowHook(10)

// Add some response times
[120, 85, 200, 150, 95, 300, 180, 110, 90, 250, 170].forEach(time => {
  responseTimeWindow.update(time)
})

const stats = responseTimeWindow.get()
console.log(`Response Time Stats (last ${stats.count} requests):`)
console.log(`Average: ${stats.average.toFixed(2)}ms`)
console.log(`Min: ${stats.min}ms, Max: ${stats.max}ms`)
```

### Feature 2: Hook Composition and Transformation

Compose multiple hooks and transform their behavior:

#### onUpdate Hook Composition

```typescript
import { MetricHook, MetricKey, Console, Effect } from "effect"

// Create a counter with multiple update behaviors
const requestCounter = MetricHook.counter(
  MetricKey.counter("api_requests")
).pipe(
  // Add logging
  MetricHook.onUpdate((count) => {
    console.log(`Request count updated by: ${count}`)
  }),
  // Add alerting
  MetricHook.onUpdate((count) => {
    if (count > 10) {
      console.warn(`Burst detected: ${count} requests at once`)
    }
  }),
  // Add metrics forwarding (simulate sending to external system)
  MetricHook.onUpdate((count) => {
    // Simulate sending to external monitoring system
    Effect.runSync(
      Effect.gen(function* () {
        yield* Console.log(`Forwarding metric to external system: ${count}`)
      })
    )
  })
)

// Usage triggers all composed behaviors
requestCounter.update(5)  // Logs all three behaviors
requestCounter.update(15) // Includes burst alert
```

#### onModify Hook for Relative Updates

```typescript
import { MetricHook, MetricKey } from "effect"

// Track server capacity with custom modify behavior
const serverCapacity = MetricHook.gauge(
  MetricKey.gauge("server_capacity_percent"), 
  0
).pipe(
  MetricHook.onModify((delta) => {
    console.log(`Server capacity changed by: ${delta > 0 ? '+' : ''}${delta}%`)
  }),
  MetricHook.onUpdate((absolute) => {
    if (absolute > 90) {
      console.error(`CRITICAL: Server capacity at ${absolute}%`)
    } else if (absolute > 75) {
      console.warn(`WARNING: Server capacity at ${absolute}%`)
    }
  })
)

// Demonstrate different update patterns
serverCapacity.update(60)  // Set to 60% - triggers onUpdate
serverCapacity.modify(20)  // Add 20% - triggers both onModify and onUpdate
serverCapacity.modify(-30) // Subtract 30% - triggers both behaviors
```

### Feature 3: Advanced Metric Aggregation Patterns

#### Multi-Dimensional Metrics

```typescript
import { MetricHook, MetricKey } from "effect"

// Track metrics across multiple dimensions
interface MultiDimensionalMetrics {
  readonly byRegion: Map<string, MetricHook.Counter<number>>
  readonly byService: Map<string, MetricHook.Histogram>
  readonly byUser: Map<string, MetricHook.Frequency>
}

const createMultiDimensionalMetrics = (): MultiDimensionalMetrics => {
  return {
    byRegion: new Map(),
    byService: new Map(), 
    byUser: new Map()
  }
}

const getOrCreateRegionCounter = (
  metrics: MultiDimensionalMetrics, 
  region: string
): MetricHook.Counter<number> => {
  if (!metrics.byRegion.has(region)) {
    const counter = MetricHook.counter(
      MetricKey.counter(`requests_by_region_${region}`)
    ).pipe(
      MetricHook.onUpdate((count) => {
        console.log(`Region ${region} requests: +${count}`)
      })
    )
    metrics.byRegion.set(region, counter)
  }
  return metrics.byRegion.get(region)!
}

const getOrCreateServiceHistogram = (
  metrics: MultiDimensionalMetrics,
  service: string
): MetricHook.Histogram => {
  if (!metrics.byService.has(service)) {
    const histogram = MetricHook.histogram(
      MetricKey.histogram(`latency_by_service_${service}`,
        MetricBoundaries.exponential({ start: 1, factor: 2, count: 10 })
      )
    )
    metrics.byService.set(service, histogram)
  }
  return metrics.byService.get(service)!
}

// Usage for multi-dimensional tracking
const multiMetrics = createMultiDimensionalMetrics()

// Track requests by region
getOrCreateRegionCounter(multiMetrics, "us-east-1").update(5)
getOrCreateRegionCounter(multiMetrics, "eu-west-1").update(3)
getOrCreateRegionCounter(multiMetrics, "us-east-1").update(2) // +2 more

// Track latency by service
getOrCreateServiceHistogram(multiMetrics, "user-service").update(150)
getOrCreateServiceHistogram(multiMetrics, "order-service").update(89)
getOrCreateServiceHistogram(multiMetrics, "user-service").update(200)

// Generate multi-dimensional report
const generateMultiDimensionalReport = (metrics: MultiDimensionalMetrics) => {
  const report = {
    byRegion: {} as Record<string, number>,
    byService: {} as Record<string, { count: number, avgMs: number }>
  }
  
  // Aggregate by region
  metrics.byRegion.forEach((counter, region) => {
    const state = counter.get()
    report.byRegion[region] = state.count
  })
  
  // Aggregate by service
  metrics.byService.forEach((histogram, service) => {
    const state = histogram.get()
    report.byService[service] = {
      count: state.count,
      avgMs: state.sum / state.count
    }
  })
  
  return report
}

console.log("Multi-dimensional Report:", generateMultiDimensionalReport(multiMetrics))
```

## Practical Patterns & Best Practices

### Pattern 1: Metric Hook Factory

Create reusable factory functions for common metric patterns:

```typescript
import { MetricHook, MetricKey, MetricBoundaries } from "effect"

// Factory for creating standardized service metrics
const createServiceMetrics = (serviceName: string) => {
  const baseKey = (metric: string) => `service_${serviceName}_${metric}`
  
  return {
    requestCount: MetricHook.counter(
      MetricKey.counter(baseKey("requests_total"))
    ).pipe(
      MetricHook.onUpdate((count) => {
        if (count > 1000) {
          console.log(`${serviceName}: High traffic - ${count} requests`)
        }
      })
    ),
    
    responseTime: MetricHook.histogram(
      MetricKey.histogram(baseKey("response_time_ms"),
        MetricBoundaries.exponential({ start: 1, factor: 1.5, count: 12 })
      )
    ).pipe(
      MetricHook.onUpdate((time) => {
        if (time > 5000) {
          console.warn(`${serviceName}: Slow response - ${time}ms`)
        }
      })
    ),
    
    errorRate: MetricHook.counter(
      MetricKey.counter(baseKey("errors_total"))
    ).pipe(
      MetricHook.onUpdate(() => {
        console.error(`${serviceName}: Error occurred`)
      })
    ),
    
    activeConnections: MetricHook.gauge(
      MetricKey.gauge(baseKey("active_connections")), 
      0
    )
  }
}

// Usage
const userServiceMetrics = createServiceMetrics("user")
const orderServiceMetrics = createServiceMetrics("order")

// Each service has the same metric structure
userServiceMetrics.requestCount.update(1)
orderServiceMetrics.responseTime.update(120)
```

### Pattern 2: Metric Aggregation Pipeline

Create pipelines for complex metric aggregations:

```typescript
import { MetricHook, MetricKey, Effect } from "effect"

// Pipeline for processing and aggregating metrics
class MetricPipeline<T> {
  private hooks: Array<MetricHook<T, any>> = []
  
  constructor(private name: string) {}
  
  addHook<Out>(hook: MetricHook<T, Out>): this {
    this.hooks.push(hook)
    return this
  }
  
  process(value: T): void {
    console.log(`Pipeline ${this.name}: Processing value ${value}`)
    this.hooks.forEach(hook => hook.update(value))
  }
  
  getAggregatedStats() {
    return this.hooks.map(hook => ({
      type: hook.constructor.name,
      state: hook.get()
    }))
  }
}

// Create a comprehensive request processing pipeline
const requestPipeline = new MetricPipeline<number>("request_processing")
  .addHook(MetricHook.counter(MetricKey.counter("total_requests")))
  .addHook(MetricHook.histogram(
    MetricKey.histogram("request_sizes",
      MetricBoundaries.exponential({ start: 1, factor: 2, count: 10 })
    )
  ))
  .addHook(MetricHook.summary(
    MetricKey.summary("request_size_percentiles", {
      maxAge: "300 seconds",
      maxSize: 1000,
      error: 0.01,
      quantiles: [0.5, 0.95, 0.99]
    })
  ))

// Process requests through the pipeline
[1024, 2048, 512, 4096, 1536, 768, 8192, 256].forEach(size => {
  requestPipeline.process(size)
})

console.log("Pipeline Results:", requestPipeline.getAggregatedStats())
```

### Pattern 3: Conditional Metric Updates

Implement conditional logic for metric updates:

```typescript
import { MetricHook, MetricKey } from "effect"

// Create a smart metric hook that only updates under certain conditions
const createConditionalHook = <T>(
  baseHook: MetricHook<T, any>,
  condition: (value: T) => boolean,
  onSkip?: (value: T) => void
) => {
  return MetricHook.make<T, any>({
    get: () => baseHook.get(),
    update: (value: T) => {
      if (condition(value)) {
        baseHook.update(value)
      } else {
        onSkip?.(value)
      }
    },
    modify: (value: T) => {
      if (condition(value)) {
        baseHook.modify(value)
      } else {
        onSkip?.(value)
      }
    }
  })
}

// Only track response times above a threshold
const significantResponseTimes = createConditionalHook(
  MetricHook.histogram(
    MetricKey.histogram("significant_response_times",
      MetricBoundaries.linear({ start: 100, width: 100, count: 20 })
    )
  ),
  (responseTime: number) => responseTime > 100, // Only track > 100ms
  (skipped: number) => console.log(`Skipped fast response: ${skipped}ms`)
)

// Only track errors during business hours
const businessHoursErrors = createConditionalHook(
  MetricHook.counter(MetricKey.counter("business_hours_errors")),
  () => {
    const hour = new Date().getHours()
    return hour >= 9 && hour <= 17 // 9 AM to 5 PM
  },
  () => console.log("Error occurred outside business hours")
)

// Test the conditional hooks
significantResponseTimes.update(50)   // Skipped
significantResponseTimes.update(250)  // Recorded
significantResponseTimes.update(80)   // Skipped
significantResponseTimes.update(180)  // Recorded

businessHoursErrors.update(1) // Recorded or skipped based on current time
```

## Integration Examples

### Integration with Express.js HTTP Server

```typescript
import { MetricHook, MetricKey, MetricBoundaries } from "effect"
import express from "express"

// HTTP server metrics
interface HttpServerMetrics {
  readonly requestCount: MetricHook.Counter<number>
  readonly responseTime: MetricHook.Histogram
  readonly statusCodes: MetricHook.Frequency
  readonly activeRequests: MetricHook.Gauge<number>
}

const createHttpServerMetrics = (): HttpServerMetrics => ({
  requestCount: MetricHook.counter(
    MetricKey.counter("http_requests_total")
  ),
  
  responseTime: MetricHook.histogram(
    MetricKey.histogram("http_request_duration_ms",
      MetricBoundaries.exponential({ start: 1, factor: 1.5, count: 15 })
    )
  ),
  
  statusCodes: MetricHook.frequency(
    MetricKey.frequency("http_response_status", 
      ["200", "201", "400", "401", "403", "404", "500", "502", "503"]
    )
  ),
  
  activeRequests: MetricHook.gauge(
    MetricKey.gauge("http_active_requests"), 
    0
  )
})

// Express middleware for metrics collection
const createMetricsMiddleware = (metrics: HttpServerMetrics) => {
  return (req: express.Request, res: express.Response, next: express.NextFunction) => {
    const startTime = Date.now()
    
    // Increment active requests
    metrics.activeRequests.modify(1)
    metrics.requestCount.update(1)
    
    // Hook into response finish event
    res.on('finish', () => {
      const duration = Date.now() - startTime
      
      // Record response time and status
      metrics.responseTime.update(duration)
      metrics.statusCodes.update(res.statusCode.toString())
      
      // Decrement active requests
      metrics.activeRequests.modify(-1)
    })
    
    next()
  }
}

// Create Express app with metrics
const app = express()
const httpMetrics = createHttpServerMetrics()

app.use(createMetricsMiddleware(httpMetrics))

// Sample routes
app.get('/api/users', (req, res) => {
  setTimeout(() => res.json({ users: [] }), Math.random() * 100)
})

app.get('/api/health', (req, res) => {
  res.json({ status: 'healthy' })
})

app.get('/metrics', (req, res) => {
  const metrics = {
    requests: httpMetrics.requestCount.get(),
    responseTime: httpMetrics.responseTime.get(),
    statusCodes: httpMetrics.statusCodes.get(),
    activeRequests: httpMetrics.activeRequests.get()
  }
  res.json(metrics)
})

// app.listen(3000, () => console.log('Server running on port 3000'))
```

### Integration with Prometheus Export

```typescript
import { MetricHook, MetricKey, Effect } from "effect"

// Prometheus-compatible metric exporter
class PrometheusExporter {
  private hooks = new Map<string, MetricHook<any, any>>()
  
  registerHook<In, Out>(name: string, hook: MetricHook<In, Out>): void {
    this.hooks.set(name, hook)
  }
  
  generatePrometheusFormat(): string {
    let output = ""
    
    this.hooks.forEach((hook, name) => {
      const state = hook.get()
      
      // Handle different metric types
      if ('count' in state && typeof state.count === 'number') {
        // Counter metric
        output += `# TYPE ${name} counter\n`
        output += `${name} ${state.count}\n\n`
      } else if ('value' in state && typeof state.value === 'number') {
        // Gauge metric
        output += `# TYPE ${name} gauge\n`
        output += `${name} ${state.value}\n\n`
      } else if ('buckets' in state && Array.isArray(state.buckets)) {
        // Histogram metric
        output += `# TYPE ${name} histogram\n`
        state.buckets.forEach(([boundary, count]: [number, number]) => {
          output += `${name}_bucket{le="${boundary}"} ${count}\n`
        })
        output += `${name}_bucket{le="+Inf"} ${state.count}\n`
        output += `${name}_sum ${state.sum}\n`
        output += `${name}_count ${state.count}\n\n`
      } else if ('occurrences' in state && state.occurrences instanceof Map) {
        // Frequency metric
        output += `# TYPE ${name} counter\n`
        state.occurrences.forEach((count, label) => {
          output += `${name}{label="${label}"} ${count}\n`
        })
        output += "\n"
      }
    })
    
    return output
  }
}

// Usage with application metrics
const prometheusExporter = new PrometheusExporter()

// Register metrics
const appRequestCounter = MetricHook.counter(MetricKey.counter("app_requests_total"))
const appResponseTime = MetricHook.histogram(
  MetricKey.histogram("app_response_time_seconds",
    MetricBoundaries.exponential({ start: 0.001, factor: 2, count: 12 })
  )
)
const appActiveConnections = MetricHook.gauge(MetricKey.gauge("app_active_connections"), 0)

prometheusExporter.registerHook("app_requests_total", appRequestCounter)
prometheusExporter.registerHook("app_response_time_seconds", appResponseTime)
prometheusExporter.registerHook("app_active_connections", appActiveConnections)

// Simulate some metric data
appRequestCounter.update(100)
appResponseTime.update(0.15)
appResponseTime.update(0.08)
appResponseTime.update(0.25)
appActiveConnections.update(5)

// Generate Prometheus format
console.log("=== Prometheus Metrics ===")
console.log(prometheusExporter.generatePrometheusFormat())
```

### Testing Strategies

```typescript
import { MetricHook, MetricKey, Effect } from "effect"

// Test utilities for metric hooks
class MetricHookTestUtils {
  static createTestCounter(name: string = "test_counter") {
    return MetricHook.counter(MetricKey.counter(name))
  }
  
  static createTestHistogram(name: string = "test_histogram") {
    return MetricHook.histogram(
      MetricKey.histogram(name, MetricBoundaries.linear({ start: 0, width: 10, count: 10 }))
    )
  }
  
  static simulateUpdates<T>(hook: MetricHook<T, any>, values: T[]): void {
    values.forEach(value => hook.update(value))
  }
  
  static captureHookUpdates<T>(hook: MetricHook<T, any>): Array<T> {
    const captured: T[] = []
    
    return MetricHook.make<T, any>({
      get: () => hook.get(),
      update: (value: T) => {
        captured.push(value)
        hook.update(value)
      },
      modify: (value: T) => {
        captured.push(value)
        hook.modify(value)
      }
    })
  }
}

// Test suite for metric hooks
const runMetricHookTests = Effect.gen(function* () {
  console.log("Running MetricHook Tests...\n")
  
  // Test 1: Counter functionality
  const counter = MetricHookTestUtils.createTestCounter()
  MetricHookTestUtils.simulateUpdates(counter, [1, 2, 3, 4, 5])
  
  const counterState = counter.get()
  console.log("Test 1 - Counter:")
  console.log(`Expected: 15, Actual: ${counterState.count}`)
  console.log(`Test 1 ${counterState.count === 15 ? 'PASSED' : 'FAILED'}\n`)
  
  // Test 2: Histogram distribution
  const histogram = MetricHookTestUtils.createTestHistogram()
  MetricHookTestUtils.simulateUpdates(histogram, [5, 15, 25, 35, 45])
  
  const histogramState = histogram.get()
  console.log("Test 2 - Histogram:")
  console.log(`Count: ${histogramState.count}`)
  console.log(`Sum: ${histogramState.sum}`)
  console.log(`Average: ${histogramState.sum / histogramState.count}`)
  console.log(`Test 2 ${histogramState.count === 5 && histogramState.sum === 125 ? 'PASSED' : 'FAILED'}\n`)
  
  // Test 3: Hook composition
  let updateCallCount = 0
  const composedCounter = MetricHook.counter(
    MetricKey.counter("composed_test")
  ).pipe(
    MetricHook.onUpdate(() => {
      updateCallCount++
    })
  )
  
  MetricHookTestUtils.simulateUpdates(composedCounter, [1, 1, 1])
  
  console.log("Test 3 - Hook Composition:")
  console.log(`Update calls: ${updateCallCount}`)
  console.log(`Test 3 ${updateCallCount === 3 ? 'PASSED' : 'FAILED'}\n`)
  
  // Test 4: Gauge modification
  const gauge = MetricHook.gauge(MetricKey.gauge("test_gauge"), 10)
  gauge.modify(5)   // Add 5 -> 15
  gauge.modify(-3)  // Subtract 3 -> 12
  gauge.update(20)  // Set to 20
  
  const gaugeState = gauge.get()
  console.log("Test 4 - Gauge:")
  console.log(`Expected: 20, Actual: ${gaugeState.value}`)
  console.log(`Test 4 ${gaugeState.value === 20 ? 'PASSED' : 'FAILED'}\n`)
  
  console.log("All tests completed!")
})

Effect.runPromise(runMetricHookTests)
```

## Conclusion

MetricHook provides powerful, composable metric collection capabilities that enable developers to build sophisticated monitoring and observability systems. By offering type-safe metric hooks with customizable behavior, it solves the common problems of inflexible metric systems while maintaining excellent performance and composability.

Key benefits:
- **Composability**: Chain multiple behaviors using `onUpdate` and `onModify` for complex metric processing pipelines
- **Type Safety**: Strong typing ensures metric updates and queries are type-safe at compile time  
- **Extensibility**: Create custom hooks for specialized use cases while maintaining consistent interfaces
- **Performance**: Efficient implementations of standard metric types (counters, gauges, histograms, summaries, frequencies)
- **Integration**: Easy integration with popular monitoring systems like Prometheus, Grafana, and custom dashboards

MetricHook is ideal for applications requiring sophisticated metric collection, real-time monitoring, performance analysis, and observability systems where flexibility and type safety are essential.