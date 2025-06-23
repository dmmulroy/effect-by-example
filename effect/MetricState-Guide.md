# MetricState: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem MetricState Solves

Traditional metric systems often struggle with representing and manipulating metric values consistently across different metric types, leading to fragmented state management:

```typescript
// Traditional approach - inconsistent metric state handling
class MetricStateManager {
  private counterValues = new Map<string, number>()
  private gaugeValues = new Map<string, number>()
  private histogramValues = new Map<string, { buckets: number[], count: number, sum: number }>()
  
  updateCounter(name: string, increment: number) {
    this.counterValues.set(name, (this.counterValues.get(name) || 0) + increment)
  }
  
  setGauge(name: string, value: number) {
    this.gaugeValues.set(name, value)
  }
  
  updateHistogram(name: string, value: number) {
    const existing = this.histogramValues.get(name) || { buckets: [], count: 0, sum: 0 }
    // Complex bucket logic...
    this.histogramValues.set(name, { ...existing, count: existing.count + 1, sum: existing.sum + value })
  }
  
  // Problematic: Different export logic for each type
  exportMetrics() {
    const counters = Array.from(this.counterValues.entries()).map(([name, value]) => ({ name, type: 'counter', value }))
    const gauges = Array.from(this.gaugeValues.entries()).map(([name, value]) => ({ name, type: 'gauge', value }))
    const histograms = Array.from(this.histogramValues.entries()).map(([name, data]) => ({ name, type: 'histogram', ...data }))
    
    return [...counters, ...gauges, ...histograms]
  }
}
```

This approach leads to:
- **Type Inconsistency** - Different storage and handling logic for each metric type
- **State Fragmentation** - Metric values scattered across different data structures
- **Manual Serialization** - Complex logic needed to export metric states uniformly
- **No State Validation** - No guarantees that metric state values are valid for their type

### The MetricState Solution

MetricState provides a unified, type-safe representation of metric values that works consistently across all metric types:

```typescript
import { MetricState, Equal, Effect } from "effect"

// Unified state representation for all metric types
const createMetricStates = Effect.gen(function* () {
  // Counter state - tracks cumulative values
  const requestCount = MetricState.counter(1247)
  
  // Gauge state - tracks point-in-time values
  const memoryUsage = MetricState.gauge(1048576) // 1MB
  
  // Histogram state - tracks value distributions
  const responseTime = MetricState.histogram({
    buckets: [[0.1, 5], [0.5, 15], [1.0, 8], [2.0, 2]],
    count: 30,
    min: 0.05,
    max: 1.8,
    sum: 18.5
  })
  
  // Summary state - tracks quantiles over time
  const latencyPercentiles = MetricState.summary({
    error: 0.01,
    quantiles: [[0.5, Option.some(0.2)], [0.95, Option.some(0.8)], [0.99, Option.some(1.2)]],
    count: 100,
    min: 0.1,
    max: 2.1,
    sum: 50.0
  })
  
  // Frequency state - tracks occurrence counts
  const errorTypes = MetricState.frequency(
    new Map([
      ["validation_error", 12],
      ["network_timeout", 5],
      ["database_error", 3]
    ])
  )
  
  return { requestCount, memoryUsage, responseTime, latencyPercentiles, errorTypes }
})
```

### Key Concepts

**Counter State**: Represents monotonically increasing values with a `count` field

**Gauge State**: Represents point-in-time measurements with a `value` field

**Histogram State**: Represents value distributions with `buckets`, `count`, `min`, `max`, and `sum` fields

**Summary State**: Represents quantile calculations with `quantiles`, `error`, `count`, `min`, `max`, and `sum` fields

**Frequency State**: Represents occurrence counts with an `occurrences` map of string keys to numeric counts

## Basic Usage Patterns

### Pattern 1: Creating Basic Metric States

```typescript
import { MetricState, Option } from "effect"

// Create counter states for different numeric types
const numberCounter = MetricState.counter(42)
const bigintCounter = MetricState.counter(1000000n)

// Create gauge states for current measurements
const cpuUsage = MetricState.gauge(0.75) // 75% CPU usage
const connectionCount = MetricState.gauge(150n) // 150 active connections

// Create frequency state for categorical data
const httpMethods = MetricState.frequency(
  new Map([
    ["GET", 450],
    ["POST", 120],
    ["PUT", 35],
    ["DELETE", 8]
  ])
)
```

### Pattern 2: Working with Complex Metric States

```typescript
// Create histogram state for response time tracking
const createResponseTimeHistogram = () => {
  return MetricState.histogram({
    // [boundary, count] pairs showing distribution
    buckets: [
      [0.1, 25],  // 25 requests under 100ms
      [0.5, 45],  // 45 requests under 500ms
      [1.0, 15],  // 15 requests under 1s
      [2.0, 8],   // 8 requests under 2s
      [5.0, 2]    // 2 requests under 5s
    ],
    count: 95,          // total request count
    min: 0.023,         // fastest request
    max: 3.456,         // slowest request
    sum: 58.234         // total time for all requests
  })
}

// Create summary state with quantile tracking
const createLatencySummary = () => {
  return MetricState.summary({
    error: 0.01,        // 1% error tolerance
    quantiles: [
      [0.5, Option.some(0.234)],   // 50th percentile (median)
      [0.95, Option.some(1.123)],  // 95th percentile
      [0.99, Option.some(2.567)],  // 99th percentile
      [0.999, Option.none()]       // 99.9th percentile (not calculated yet)
    ],
    count: 1000,
    min: 0.001,
    max: 5.678,
    sum: 450.123
  })
}
```

### Pattern 3: State Type Checking and Validation

```typescript
import { MetricState, Effect } from "effect"

// Type-safe state inspection
const inspectMetricState = (state: MetricState.MetricState.Untyped) => Effect.gen(function* () {
  if (MetricState.isCounterState(state)) {
    yield* Effect.log(`Counter value: ${state.count}`)
  } else if (MetricState.isGaugeState(state)) {
    yield* Effect.log(`Gauge value: ${state.value}`)
  } else if (MetricState.isHistogramState(state)) {
    yield* Effect.log(`Histogram: ${state.count} samples, avg: ${state.sum / state.count}`)
  } else if (MetricState.isSummaryState(state)) {
    yield* Effect.log(`Summary: ${state.count} samples, quantiles: ${state.quantiles.length}`)
  } else if (MetricState.isFrequencyState(state)) {
    yield* Effect.log(`Frequency: ${state.occurrences.size} unique values`)
  }
})
```

## Real-World Examples

### Example 1: Application Performance Monitoring System

Building a comprehensive APM system that manages different types of metric states for performance tracking:

```typescript
import { MetricState, Effect, Array as Arr, Option, HashMap } from "effect"

// Performance metrics state management
class PerformanceMetrics {
  private states = new Map<string, MetricState.MetricState.Untyped>()
  
  // Initialize common performance metric states
  static create = Effect.gen(function* () {
    const metrics = new PerformanceMetrics()
    
    // Request counting
    metrics.states.set("http_requests_total", MetricState.counter(0))
    
    // Response time distribution
    metrics.states.set("http_response_time", MetricState.histogram({
      buckets: [
        [0.005, 0], [0.01, 0], [0.025, 0], [0.05, 0],
        [0.1, 0], [0.25, 0], [0.5, 0], [1.0, 0], [2.5, 0], [5.0, 0]
      ],
      count: 0,
      min: 0,
      max: 0,
      sum: 0
    }))
    
    // Memory usage tracking
    metrics.states.set("memory_usage_bytes", MetricState.gauge(0))
    
    // Error frequency tracking
    metrics.states.set("error_frequency", MetricState.frequency(new Map()))
    
    // Latency percentiles
    metrics.states.set("latency_percentiles", MetricState.summary({
      error: 0.01,
      quantiles: [
        [0.5, Option.none()],
        [0.95, Option.none()],
        [0.99, Option.none()]
      ],
      count: 0,
      min: 0,
      max: 0,
      sum: 0
    }))
    
    return metrics
  })
  
  // Update request metrics
  updateRequestMetrics = (responseTime: number, statusCode: number) => Effect.gen(function* () {
    // Increment request counter
    const currentCount = this.states.get("http_requests_total")
    if (MetricState.isCounterState(currentCount)) {
      this.states.set("http_requests_total", MetricState.counter(currentCount.count + 1))
    }
    
    // Update response time histogram
    const histogram = this.states.get("http_response_time")
    if (MetricState.isHistogramState(histogram)) {
      const newHistogram = this.updateHistogramWithValue(histogram, responseTime)
      this.states.set("http_response_time", newHistogram)
    }
    
    // Track errors in frequency map
    if (statusCode >= 400) {
      const errorType = statusCode >= 500 ? "server_error" : "client_error"
      const frequency = this.states.get("error_frequency")
      if (MetricState.isFrequencyState(frequency)) {
        const newOccurrences = new Map(frequency.occurrences)
        newOccurrences.set(errorType, (newOccurrences.get(errorType) || 0) + 1)
        this.states.set("error_frequency", MetricState.frequency(newOccurrences))
      }
    }
    
    yield* Effect.log(`Updated metrics for ${responseTime}ms response (${statusCode})`)
  })
  
  // Helper to update histogram state with new value
  private updateHistogramWithValue = (
    histogram: MetricState.MetricState.Histogram,
    value: number
  ): MetricState.MetricState.Histogram => {
    const newBuckets = histogram.buckets.map(([boundary, count]) => 
      [boundary, value <= boundary ? count + 1 : count] as const
    )
    
    return MetricState.histogram({
      buckets: newBuckets,
      count: histogram.count + 1,
      min: histogram.count === 0 ? value : Math.min(histogram.min, value),
      max: histogram.count === 0 ? value : Math.max(histogram.max, value),
      sum: histogram.sum + value
    })
  }
  
  // Generate performance report from current states
  generateReport = Effect.gen(function* () {
    const report: Record<string, any> = {}
    
    for (const [name, state] of this.states) {
      if (MetricState.isCounterState(state)) {
        report[name] = { type: "counter", value: state.count }
      } else if (MetricState.isGaugeState(state)) {
        report[name] = { type: "gauge", value: state.value }
      } else if (MetricState.isHistogramState(state)) {
        report[name] = {
          type: "histogram",
          count: state.count,
          average: state.count > 0 ? state.sum / state.count : 0,
          min: state.min,
          max: state.max,
          buckets: state.buckets
        }
      } else if (MetricState.isSummaryState(state)) {
        report[name] = {
          type: "summary",
          count: state.count,
          quantiles: state.quantiles.map(([q, v]) => [q, Option.getOrNull(v)])
        }
      } else if (MetricState.isFrequencyState(state)) {
        report[name] = {
          type: "frequency",
          occurrences: Object.fromEntries(state.occurrences)
        }
      }
    }
    
    return report
  })
  
  // Get all metric states for export
  getAllStates = () => Array.from(this.states.entries())
}

// Usage in HTTP server
const trackHttpRequest = (path: string, method: string, responseTime: number, statusCode: number) => Effect.gen(function* () {
  const metrics = yield* PerformanceMetrics.create
  
  yield* metrics.updateRequestMetrics(responseTime, statusCode)
  
  // Update memory usage gauge
  const memoryUsage = process.memoryUsage().heapUsed
  metrics.states.set("memory_usage_bytes", MetricState.gauge(memoryUsage))
  
  return yield* metrics.generateReport
})
```

### Example 2: Database Performance Monitoring

Comprehensive database metrics using different MetricState types to track query performance:

```typescript
import { MetricState, Effect, Schedule, Duration, Option } from "effect"

// Database metrics state manager
class DatabaseMetrics {
  private queryStates = new Map<string, MetricState.MetricState.Untyped>()
  private connectionStates = new Map<string, MetricState.MetricState.Untyped>()
  
  static create = Effect.gen(function* () {
    const metrics = new DatabaseMetrics()
    
    // Query performance metrics
    metrics.queryStates.set("queries_total", MetricState.counter(0))
    metrics.queryStates.set("query_duration_histogram", MetricState.histogram({
      buckets: [
        [0.001, 0], [0.005, 0], [0.01, 0], [0.05, 0],
        [0.1, 0], [0.5, 0], [1.0, 0], [5.0, 0], [10.0, 0]
      ],
      count: 0,
      min: 0,
      max: 0,
      sum: 0
    }))
    
    metrics.queryStates.set("query_types", MetricState.frequency(new Map([
      ["SELECT", 0],
      ["INSERT", 0],
      ["UPDATE", 0],
      ["DELETE", 0]
    ])))
    
    // Connection pool metrics
    metrics.connectionStates.set("active_connections", MetricState.gauge(0))
    metrics.connectionStates.set("connection_pool_size", MetricState.gauge(10))
    
    // Connection lifetime tracking
    metrics.connectionStates.set("connection_lifetime", MetricState.summary({
      error: 0.05,
      quantiles: [
        [0.5, Option.none()],
        [0.9, Option.none()],
        [0.95, Option.none()],
        [0.99, Option.none()]
      ],
      count: 0,
      min: 0,
      max: 0,
      sum: 0
    }))
    
    return metrics
  })
  
  // Track a database query
  recordQuery = (queryType: string, durationSeconds: number, isSuccess: boolean) => Effect.gen(function* () {
    // Update query counter
    const queryCount = this.queryStates.get("queries_total")
    if (MetricState.isCounterState(queryCount)) {
      this.queryStates.set("queries_total", MetricState.counter(queryCount.count + 1))
    }
    
    // Update query duration histogram
    const durationHist = this.queryStates.get("query_duration_histogram")
    if (MetricState.isHistogramState(durationHist)) {
      const updatedHist = this.addValueToHistogram(durationHist, durationSeconds)
      this.queryStates.set("query_duration_histogram", updatedHist)
    }
    
    // Update query type frequency
    const queryTypes = this.queryStates.get("query_types")
    if (MetricState.isFrequencyState(queryTypes)) {
      const newOccurrences = new Map(queryTypes.occurrences)
      newOccurrences.set(queryType, (newOccurrences.get(queryType) || 0) + 1)
      this.queryStates.set("query_types", MetricState.frequency(newOccurrences))
    }
    
    yield* Effect.log(`Recorded ${queryType} query: ${durationSeconds}s (${isSuccess ? 'success' : 'failure'})`)
  })
  
  // Update connection pool metrics
  updateConnectionMetrics = (activeConnections: number, poolSize: number) => Effect.gen(function* () {
    this.connectionStates.set("active_connections", MetricState.gauge(activeConnections))
    this.connectionStates.set("connection_pool_size", MetricState.gauge(poolSize))
    
    yield* Effect.log(`Connection pool: ${activeConnections}/${poolSize} active`)
  })
  
  // Record connection lifetime for summary statistics
  recordConnectionLifetime = (lifetimeSeconds: number) => Effect.gen(function* () {
    const summary = this.connectionStates.get("connection_lifetime")
    if (MetricState.isSummaryState(summary)) {
      const updatedSummary = this.addValueToSummary(summary, lifetimeSeconds)
      this.connectionStates.set("connection_lifetime", updatedSummary)
    }
  })
  
  // Helper to add value to histogram
  private addValueToHistogram = (
    histogram: MetricState.MetricState.Histogram,
    value: number
  ): MetricState.MetricState.Histogram => {
    const newBuckets = histogram.buckets.map(([boundary, count]) => 
      [boundary, value <= boundary ? count + 1 : count] as const
    )
    
    return MetricState.histogram({
      buckets: newBuckets,
      count: histogram.count + 1,
      min: histogram.count === 0 ? value : Math.min(histogram.min, value),
      max: histogram.count === 0 ? value : Math.max(histogram.max, value),
      sum: histogram.sum + value
    })
  }
  
  // Helper to add value to summary
  private addValueToSummary = (
    summary: MetricState.MetricState.Summary,
    value: number
  ): MetricState.MetricState.Summary => {
    // Simplified summary update - in real implementation, you'd maintain
    // a sliding window and recalculate quantiles
    return MetricState.summary({
      error: summary.error,
      quantiles: summary.quantiles, // Would be recalculated in practice
      count: summary.count + 1,
      min: summary.count === 0 ? value : Math.min(summary.min, value),
      max: summary.count === 0 ? value : Math.max(summary.max, value),
      sum: summary.sum + value
    })
  }
  
  // Export all database metrics
  exportMetrics = Effect.gen(function* () {
    const allStates = new Map([
      ...this.queryStates,
      ...this.connectionStates
    ])
    
    const exported: Record<string, any> = {}
    
    for (const [name, state] of allStates) {
      if (MetricState.isCounterState(state)) {
        exported[name] = { type: "counter", value: state.count }
      } else if (MetricState.isGaugeState(state)) {
        exported[name] = { type: "gauge", value: state.value }
      } else if (MetricState.isHistogramState(state)) {
        exported[name] = {
          type: "histogram",
          samples: state.count,
          average: state.count > 0 ? state.sum / state.count : 0,
          p95: this.calculateP95FromHistogram(state),
          buckets: Object.fromEntries(state.buckets)
        }
      } else if (MetricState.isSummaryState(state)) {
        exported[name] = {
          type: "summary",
          samples: state.count,
          quantiles: Object.fromEntries(
            state.quantiles.map(([q, v]) => [q, Option.getOrNull(v)])
          )
        }
      } else if (MetricState.isFrequencyState(state)) {
        exported[name] = {
          type: "frequency",
          total: Array.from(state.occurrences.values()).reduce((a, b) => a + b, 0),
          breakdown: Object.fromEntries(state.occurrences)
        }
      }
    }
    
    return exported
  })
  
  private calculateP95FromHistogram = (histogram: MetricState.MetricState.Histogram): number => {
    const p95Count = histogram.count * 0.95
    let cumulativeCount = 0
    
    for (const [boundary, count] of histogram.buckets) {
      cumulativeCount += count
      if (cumulativeCount >= p95Count) {
        return boundary
      }
    }
    
    return histogram.max
  }
}

// Usage example
const trackDatabaseOperation = (operation: string, query: string) => Effect.gen(function* () {
  const metrics = yield* DatabaseMetrics.create
  const startTime = yield* Effect.succeed(Date.now())
  
  // Simulate database operation
  yield* Effect.sleep(Duration.millis(Math.random() * 1000))
  
  const endTime = yield* Effect.succeed(Date.now())
  const durationSeconds = (endTime - startTime) / 1000
  
  // Record the operation
  yield* metrics.recordQuery(operation, durationSeconds, true)
  yield* metrics.updateConnectionMetrics(8, 10)
  
  // Export current metrics
  const currentMetrics = yield* metrics.exportMetrics
  yield* Effect.log(`Database metrics: ${JSON.stringify(currentMetrics, null, 2)}`)
  
  return currentMetrics
})
```

### Example 3: Business Intelligence Dashboard

Building a business metrics system that tracks KPIs using various MetricState types:

```typescript
import { MetricState, Effect, Array as Arr, Option, Schedule, Duration } from "effect"

// Business metrics state management
class BusinessIntelligence {
  private revenueStates = new Map<string, MetricState.MetricState.Untyped>()
  private customerStates = new Map<string, MetricState.MetricState.Untyped>()
  private orderStates = new Map<string, MetricState.MetricState.Untyped>()
  
  static create = Effect.gen(function* () {
    const bi = new BusinessIntelligence()
    
    // Revenue tracking
    bi.revenueStates.set("total_revenue", MetricState.gauge(0))
    bi.revenueStates.set("daily_revenue", MetricState.gauge(0))
    bi.revenueStates.set("revenue_by_product", MetricState.frequency(new Map()))
    
    // Customer metrics
    bi.customerStates.set("total_customers", MetricState.counter(0))
    bi.customerStates.set("active_customers", MetricState.gauge(0))
    bi.customerStates.set("customer_acquisition_cost", MetricState.summary({
      error: 0.05,
      quantiles: [
        [0.25, Option.none()],
        [0.5, Option.none()],
        [0.75, Option.none()],
        [0.95, Option.none()]
      ],
      count: 0,
      min: 0,
      max: 0,
      sum: 0
    }))
    
    // Order analytics
    bi.orderStates.set("orders_total", MetricState.counter(0))
    bi.orderStates.set("order_value_distribution", MetricState.histogram({
      buckets: [
        [10, 0], [25, 0], [50, 0], [100, 0], [250, 0],
        [500, 0], [1000, 0], [2500, 0], [5000, 0]
      ],
      count: 0,
      min: 0,
      max: 0,
      sum: 0
    }))
    
    bi.orderStates.set("order_status_frequency", MetricState.frequency(new Map([
      ["pending", 0],
      ["processing", 0],
      ["shipped", 0],
      ["delivered", 0],
      ["cancelled", 0]
    ])))
    
    return bi
  })
  
  // Record a new customer acquisition
  recordCustomerAcquisition = (acquisitionCost: number) => Effect.gen(function* () {
    // Increment total customers
    const totalCustomers = this.customerStates.get("total_customers")
    if (MetricState.isCounterState(totalCustomers)) {
      this.customerStates.set("total_customers", MetricState.counter(totalCustomers.count + 1))
    }
    
    // Update acquisition cost summary
    const costSummary = this.customerStates.get("customer_acquisition_cost")
    if (MetricState.isSummaryState(costSummary)) {
      const updatedSummary = MetricState.summary({
        error: costSummary.error,
        quantiles: costSummary.quantiles, // Would recalculate in practice
        count: costSummary.count + 1,
        min: costSummary.count === 0 ? acquisitionCost : Math.min(costSummary.min, acquisitionCost),
        max: costSummary.count === 0 ? acquisitionCost : Math.max(costSummary.max, acquisitionCost),
        sum: costSummary.sum + acquisitionCost
      })
      this.customerStates.set("customer_acquisition_cost", updatedSummary)
    }
    
    yield* Effect.log(`New customer acquired with cost: $${acquisitionCost}`)
  })
  
  // Record a new order
  recordOrder = (orderId: string, value: number, productCategory: string, status: string) => Effect.gen(function* () {
    // Increment order counter
    const orderCount = this.orderStates.get("orders_total")
    if (MetricState.isCounterState(orderCount)) {
      this.orderStates.set("orders_total", MetricState.counter(orderCount.count + 1))
    }
    
    // Update order value histogram
    const valueHist = this.orderStates.get("order_value_distribution")
    if (MetricState.isHistogramState(valueHist)) {
      const newBuckets = valueHist.buckets.map(([boundary, count]) => 
        [boundary, value <= boundary ? count + 1 : count] as const
      )
      
      const updatedHist = MetricState.histogram({
        buckets: newBuckets,
        count: valueHist.count + 1,
        min: valueHist.count === 0 ? value : Math.min(valueHist.min, value),
        max: valueHist.count === 0 ? value : Math.max(valueHist.max, value),
        sum: valueHist.sum + value
      })
      this.orderStates.set("order_value_distribution", updatedHist)
    }
    
    // Update revenue by product frequency
    const revenueByProduct = this.revenueStates.get("revenue_by_product")
    if (MetricState.isFrequencyState(revenueByProduct)) {
      const newOccurrences = new Map(revenueByProduct.occurrences)
      newOccurrences.set(productCategory, (newOccurrences.get(productCategory) || 0) + value)
      this.revenueStates.set("revenue_by_product", MetricState.frequency(newOccurrences))
    }
    
    // Update order status frequency
    const statusFreq = this.orderStates.get("order_status_frequency")
    if (MetricState.isFrequencyState(statusFreq)) {
      const newOccurrences = new Map(statusFreq.occurrences)
      newOccurrences.set(status, (newOccurrences.get(status) || 0) + 1)
      this.orderStates.set("order_status_frequency", MetricState.frequency(newOccurrences))
    }
    
    // Update total revenue
    const totalRevenue = this.revenueStates.get("total_revenue")
    if (MetricState.isGaugeState(totalRevenue)) {
      this.revenueStates.set("total_revenue", MetricState.gauge(totalRevenue.value + value))
    }
    
    yield* Effect.log(`Order ${orderId}: $${value} (${productCategory}, ${status})`)
  })
  
  // Generate comprehensive business report
  generateBusinessReport = Effect.gen(function* () {
    const report: any = {
      timestamp: new Date().toISOString(),
      revenue: {},
      customers: {},
      orders: {}
    }
    
    // Process revenue metrics
    for (const [name, state] of this.revenueStates) {
      if (MetricState.isGaugeState(state)) {
        report.revenue[name] = state.value
      } else if (MetricState.isFrequencyState(state)) {
        report.revenue[name] = Object.fromEntries(state.occurrences)
      }
    }
    
    // Process customer metrics
    for (const [name, state] of this.customerStates) {
      if (MetricState.isCounterState(state)) {
        report.customers[name] = state.count
      } else if (MetricState.isGaugeState(state)) {
        report.customers[name] = state.value
      } else if (MetricState.isSummaryState(state)) {
        report.customers[name] = {
          count: state.count,
          average: state.count > 0 ? state.sum / state.count : 0,
          quantiles: Object.fromEntries(
            state.quantiles.map(([q, v]) => [q, Option.getOrNull(v)])
          )
        }
      }
    }
    
    // Process order metrics
    for (const [name, state] of this.orderStates) {
      if (MetricState.isCounterState(state)) {
        report.orders[name] = state.count
      } else if (MetricState.isHistogramState(state)) {
        report.orders[name] = {
          total_orders: state.count,
          total_value: state.sum,
          average_order_value: state.count > 0 ? state.sum / state.count : 0,
          min_order: state.min,
          max_order: state.max,
          distribution: Object.fromEntries(state.buckets)
        }
      } else if (MetricState.isFrequencyState(state)) {
        report.orders[name] = Object.fromEntries(state.occurrences)
      }
    }
    
    // Calculate derived KPIs
    const totalCustomers = this.customerStates.get("total_customers")
    const totalRevenue = this.revenueStates.get("total_revenue")
    
    if (MetricState.isCounterState(totalCustomers) && MetricState.isGaugeState(totalRevenue)) {
      report.kpis = {
        revenue_per_customer: totalCustomers.count > 0 ? totalRevenue.value / totalCustomers.count : 0,
        total_customers: totalCustomers.count,
        total_revenue: totalRevenue.value
      }
    }
    
    return report
  })
  
  // Simulate daily metrics update
  updateDailyMetrics = Effect.gen(function* () {
    // Simulate some business activity
    yield* this.recordCustomerAcquisition(25.50)
    yield* this.recordOrder("ORD-001", 149.99, "electronics", "pending")
    yield* this.recordOrder("ORD-002", 79.99, "books", "shipped")
    yield* this.recordOrder("ORD-003", 299.99, "electronics", "delivered")
    
    // Update active customers gauge (would come from actual database query)
    this.customerStates.set("active_customers", MetricState.gauge(1250))
    
    // Update daily revenue (would be calculated from actual orders)
    this.revenueStates.set("daily_revenue", MetricState.gauge(2400.50))
    
    yield* Effect.log("Daily metrics updated")
  })
}

// Usage example with scheduled updates
const runBusinessIntelligence = Effect.gen(function* () {
  const bi = yield* BusinessIntelligence.create
  
  // Initial data load
  yield* bi.updateDailyMetrics
  
  // Generate initial report
  const report = yield* bi.generateBusinessReport
  yield* Effect.log(`Business Report: ${JSON.stringify(report, null, 2)}`)
  
  // Schedule periodic updates
  const scheduler = bi.updateDailyMetrics.pipe(
    Effect.repeat(Schedule.fixed(Duration.hours(1)))
  )
  
  yield* scheduler
})
```

## Advanced Features Deep Dive

### Feature 1: Metric State Composition and Merging

Combine multiple metric states of the same type to create aggregated views.

#### Basic State Merging

```typescript
import { MetricState, Equal } from "effect"

// Helper to merge counter states
const mergeCounterStates = (
  states: MetricState.MetricState.Counter<number>[]
): MetricState.MetricState.Counter<number> => {
  const totalCount = states.reduce((sum, state) => sum + state.count, 0)
  return MetricState.counter(totalCount)
}

// Helper to merge gauge states (taking the latest or average)
const mergeGaugeStates = (
  states: MetricState.MetricState.Gauge<number>[],
  strategy: "latest" | "average" | "max" | "min" = "latest"
): MetricState.MetricState.Gauge<number> => {
  if (states.length === 0) return MetricState.gauge(0)
  
  switch (strategy) {
    case "latest":
      return states[states.length - 1]
    case "average":
      const avg = states.reduce((sum, state) => sum + state.value, 0) / states.length
      return MetricState.gauge(avg)
    case "max":
      const max = Math.max(...states.map(state => state.value))
      return MetricState.gauge(max)
    case "min":
      const min = Math.min(...states.map(state => state.value))
      return MetricState.gauge(min)
  }
}

// Helper to merge frequency states
const mergeFrequencyStates = (
  states: MetricState.MetricState.Frequency[]
): MetricState.MetricState.Frequency => {
  const merged = new Map<string, number>()
  
  for (const state of states) {
    for (const [key, count] of state.occurrences) {
      merged.set(key, (merged.get(key) || 0) + count)
    }
  }
  
  return MetricState.frequency(merged)
}
```

#### Real-World State Aggregation Example

```typescript
// Multi-region metrics aggregation
const aggregateRegionalMetrics = Effect.gen(function* () {
  // Simulate metrics from different regions
  const usWestMetrics = {
    requests: MetricState.counter(1500),
    activeUsers: MetricState.gauge(250),
    errorTypes: MetricState.frequency(new Map([
      ["timeout", 5],
      ["validation", 12],
      ["auth", 3]
    ]))
  }
  
  const usEastMetrics = {
    requests: MetricState.counter(2100),
    activeUsers: MetricState.gauge(380),
    errorTypes: MetricState.frequency(new Map([
      ["timeout", 8],
      ["validation", 15],
      ["database", 4]
    ]))
  }
  
  const euMetrics = {
    requests: MetricState.counter(950),
    activeUsers: MetricState.gauge(120),
    errorTypes: MetricState.frequency(new Map([
      ["timeout", 2],
      ["validation", 7],
      ["network", 1]
    ]))
  }
  
  // Aggregate counter states (total requests across regions)
  const totalRequests = mergeCounterStates([
    usWestMetrics.requests,
    usEastMetrics.requests,
    euMetrics.requests
  ])
  
  // Aggregate gauge states (total active users)
  const totalActiveUsers = mergeGaugeStates([
    usWestMetrics.activeUsers,
    usEastMetrics.activeUsers,
    euMetrics.activeUsers
  ], "average") // or use sum by adding values
  
  // Aggregate frequency states (all error types combined)
  const allErrorTypes = mergeFrequencyStates([
    usWestMetrics.errorTypes,
    usEastMetrics.errorTypes,
    euMetrics.errorTypes
  ])
  
  yield* Effect.log(`Global requests: ${totalRequests.count}`)
  yield* Effect.log(`Average active users: ${totalActiveUsers.value}`)
  yield* Effect.log(`Error distribution: ${JSON.stringify(Object.fromEntries(allErrorTypes.occurrences))}`)
  
  return {
    totalRequests,
    totalActiveUsers,
    allErrorTypes
  }
})
```

### Feature 2: Metric State Serialization and Deserialization

Convert metric states to/from JSON for storage, transport, and persistence.

#### Serialization Helpers

```typescript
// Serialize metric state to JSON
const serializeMetricState = (state: MetricState.MetricState.Untyped): any => {
  if (MetricState.isCounterState(state)) {
    return {
      type: "counter",
      count: state.count
    }
  } else if (MetricState.isGaugeState(state)) {
    return {
      type: "gauge",
      value: state.value
    }
  } else if (MetricState.isHistogramState(state)) {
    return {
      type: "histogram",
      buckets: state.buckets,
      count: state.count,
      min: state.min,
      max: state.max,
      sum: state.sum
    }
  } else if (MetricState.isSummaryState(state)) {
    return {
      type: "summary",
      error: state.error,
      quantiles: state.quantiles.map(([q, v]) => [q, Option.getOrNull(v)]),
      count: state.count,
      min: state.min,
      max: state.max,
      sum: state.sum
    }
  } else if (MetricState.isFrequencyState(state)) {
    return {
      type: "frequency",
      occurrences: Object.fromEntries(state.occurrences)
    }
  }
  
  throw new Error("Unknown metric state type")
}

// Deserialize JSON to metric state
const deserializeMetricState = (json: any): MetricState.MetricState.Untyped => {
  switch (json.type) {
    case "counter":
      return MetricState.counter(json.count)
    case "gauge":
      return MetricState.gauge(json.value)
    case "histogram":
      return MetricState.histogram({
        buckets: json.buckets,
        count: json.count,
        min: json.min,
        max: json.max,
        sum: json.sum
      })
    case "summary":
      return MetricState.summary({
        error: json.error,
        quantiles: json.quantiles.map(([q, v]: [number, number | null]) => 
          [q, v === null ? Option.none() : Option.some(v)]
        ),
        count: json.count,
        min: json.min,
        max: json.max,
        sum: json.sum
      })
    case "frequency":
      return MetricState.frequency(new Map(Object.entries(json.occurrences)))
    default:
      throw new Error(`Unknown metric state type: ${json.type}`)
  }
}
```

#### Real-World Persistence Example

```typescript
// Metric state persistence service
const createMetricStatePersistence = Effect.gen(function* () {
  const saveToFile = (states: Map<string, MetricState.MetricState.Untyped>, filename: string) => Effect.gen(function* () {
    const serialized = Object.fromEntries(
      Array.from(states.entries()).map(([name, state]) => [
        name,
        {
          ...serializeMetricState(state),
          timestamp: Date.now()
        }
      ])
    )
    
    const json = JSON.stringify(serialized, null, 2)
    
    // In real implementation, would use FileSystem.writeFileString
    yield* Effect.log(`Saving metrics to ${filename}: ${json.length} bytes`)
    
    return json
  })
  
  const loadFromFile = (filename: string) => Effect.gen(function* () {
    // In real implementation, would use FileSystem.readFileString
    const mockData = {
      "http_requests": {
        type: "counter",
        count: 1500,
        timestamp: Date.now() - 3600000
      },
      "memory_usage": {
        type: "gauge",
        value: 1048576,
        timestamp: Date.now() - 1800000
      },
      "response_times": {
        type: "histogram",
        buckets: [[0.1, 10], [0.5, 25], [1.0, 8], [2.0, 2]],
        count: 45,
        min: 0.05,
        max: 1.8,
        sum: 22.5,
        timestamp: Date.now() - 900000
      }
    }
    
    const states = new Map<string, MetricState.MetricState.Untyped>()
    
    for (const [name, data] of Object.entries(mockData)) {
      const state = deserializeMetricState(data)
      states.set(name, state)
    }
    
    yield* Effect.log(`Loaded ${states.size} metric states from ${filename}`)
    
    return states
  })
  
  const snapshotMetrics = (states: Map<string, MetricState.MetricState.Untyped>) => Effect.gen(function* () {
    const timestamp = new Date().toISOString()
    const filename = `metrics-snapshot-${timestamp}.json`
    
    const json = yield* saveToFile(states, filename)
    
    return { filename, size: json.length, timestamp }
  })
  
  return { saveToFile, loadFromFile, snapshotMetrics } as const
})
```

### Feature 3: Metric State Validation and Type Safety

Ensure metric states are valid and properly typed at runtime.

#### State Validation

```typescript
// Comprehensive metric state validation
const validateMetricState = (state: MetricState.MetricState.Untyped, expectedType?: string) => Effect.gen(function* () {
  const validations: string[] = []
  
  if (MetricState.isCounterState(state)) {
    if (expectedType && expectedType !== "counter") {
      return yield* Effect.fail(new Error(`Expected ${expectedType}, got counter`))
    }
    
    if (state.count < 0) {
      validations.push("Counter value cannot be negative")
    }
    
    if (!Number.isFinite(state.count)) {
      validations.push("Counter value must be finite")
    }
  } else if (MetricState.isGaugeState(state)) {
    if (expectedType && expectedType !== "gauge") {
      return yield* Effect.fail(new Error(`Expected ${expectedType}, got gauge`))
    }
    
    if (!Number.isFinite(state.value)) {
      validations.push("Gauge value must be finite")
    }
  } else if (MetricState.isHistogramState(state)) {
    if (expectedType && expectedType !== "histogram") {
      return yield* Effect.fail(new Error(`Expected ${expectedType}, got histogram`))
    }
    
    if (state.count < 0) {
      validations.push("Histogram count cannot be negative")
    }
    
    if (state.min > state.max) {
      validations.push("Histogram min cannot be greater than max")
    }
    
    if (state.buckets.length === 0) {
      validations.push("Histogram must have at least one bucket")
    }
    
    // Validate bucket boundaries are sorted
    for (let i = 1; i < state.buckets.length; i++) {
      if (state.buckets[i][0] <= state.buckets[i-1][0]) {
        validations.push("Histogram bucket boundaries must be in ascending order")
      }
    }
    
    // Validate bucket counts
    for (const [boundary, count] of state.buckets) {
      if (count < 0) {
        validations.push(`Bucket count for boundary ${boundary} cannot be negative`)
      }
    }
  } else if (MetricState.isSummaryState(state)) {
    if (expectedType && expectedType !== "summary") {
      return yield* Effect.fail(new Error(`Expected ${expectedType}, got summary`))
    }
    
    if (state.error <= 0 || state.error >= 1) {
      validations.push("Summary error must be between 0 and 1")
    }
    
    if (state.count < 0) {
      validations.push("Summary count cannot be negative")
    }
    
    if (state.min > state.max) {
      validations.push("Summary min cannot be greater than max")
    }
    
    // Validate quantiles
    for (const [quantile, value] of state.quantiles) {
      if (quantile < 0 || quantile > 1) {
        validations.push(`Quantile ${quantile} must be between 0 and 1`)
      }
      
      if (Option.isSome(value) && (value.value < state.min || value.value > state.max)) {
        validations.push(`Quantile value ${value.value} is outside min/max range`)
      }
    }
  } else if (MetricState.isFrequencyState(state)) {
    if (expectedType && expectedType !== "frequency") {
      return yield* Effect.fail(new Error(`Expected ${expectedType}, got frequency`))
    }
    
    for (const [key, count] of state.occurrences) {
      if (count < 0) {
        validations.push(`Frequency count for '${key}' cannot be negative`)
      }
      
      if (!Number.isInteger(count)) {
        validations.push(`Frequency count for '${key}' must be an integer`)
      }
    }
  }
  
  if (validations.length > 0) {
    return yield* Effect.fail(new Error(`Metric state validation failed: ${validations.join(", ")}`))
  }
  
  yield* Effect.log("Metric state validation passed")
  return state
})

// Type-safe metric state builder with validation
const buildValidatedMetricState = <T extends "counter" | "gauge" | "histogram" | "summary" | "frequency">(
  type: T,
  data: any
) => Effect.gen(function* () {
  const state = (() => {
    switch (type) {
      case "counter":
        return MetricState.counter(data as number)
      case "gauge":
        return MetricState.gauge(data as number)
      case "histogram":
        return MetricState.histogram(data)
      case "summary":
        return MetricState.summary(data)
      case "frequency":
        return MetricState.frequency(data as Map<string, number>)
      default:
        throw new Error(`Unknown metric type: ${type}`)
    }
  })()
  
  const validated = yield* validateMetricState(state, type)
  
  return validated
})
```

## Practical Patterns & Best Practices

### Pattern 1: Metric State Factory for Consistent Creation

Create reusable factories that ensure consistent metric state creation with validation:

```typescript
// Centralized metric state factory
const createMetricStateFactory = Effect.gen(function* () {
  const createCounterState = (initialValue: number = 0) => Effect.gen(function* () {
    if (initialValue < 0) {
      return yield* Effect.fail(new Error("Counter initial value cannot be negative"))
    }
    
    const state = MetricState.counter(initialValue)
    yield* Effect.log(`Created counter state with value: ${initialValue}`)
    return state
  })
  
  const createGaugeState = (initialValue: number = 0) => Effect.gen(function* () {
    if (!Number.isFinite(initialValue)) {
      return yield* Effect.fail(new Error("Gauge initial value must be finite"))
    }
    
    const state = MetricState.gauge(initialValue)
    yield* Effect.log(`Created gauge state with value: ${initialValue}`)
    return state
  })
  
  const createHistogramState = (bucketBoundaries: number[]) => Effect.gen(function* () {
    if (bucketBoundaries.length === 0) {
      return yield* Effect.fail(new Error("Histogram must have at least one bucket"))
    }
    
    // Ensure boundaries are sorted
    const sortedBoundaries = [...bucketBoundaries].sort((a, b) => a - b)
    const buckets = sortedBoundaries.map(boundary => [boundary, 0] as const)
    
    const state = MetricState.histogram({
      buckets,
      count: 0,
      min: 0,
      max: 0,
      sum: 0
    })
    
    yield* Effect.log(`Created histogram state with ${buckets.length} buckets`)
    return state
  })
  
  const createSummaryState = (quantiles: number[], error: number = 0.01) => Effect.gen(function* () {
    if (error <= 0 || error >= 1) {
      return yield* Effect.fail(new Error("Summary error must be between 0 and 1"))
    }
    
    const quantileSpecs = quantiles.map(q => [q, Option.none()] as const)
    
    const state = MetricState.summary({
      error,
      quantiles: quantileSpecs,
      count: 0,
      min: 0,
      max: 0,
      sum: 0
    })
    
    yield* Effect.log(`Created summary state with ${quantiles.length} quantiles`)
    return state
  })
  
  const createFrequencyState = (initialKeys: string[] = []) => Effect.gen(function* () {
    const occurrences = new Map(initialKeys.map(key => [key, 0]))
    
    const state = MetricState.frequency(occurrences)
    yield* Effect.log(`Created frequency state with ${initialKeys.length} initial keys`)
    return state
  })
  
  return {
    createCounterState,
    createGaugeState,
    createHistogramState,
    createSummaryState,
    createFrequencyState
  } as const
})

// Usage example
const setupApplicationMetrics = Effect.gen(function* () {
  const factory = yield* createMetricStateFactory
  
  // Create standard HTTP metrics
  const httpRequestCount = yield* factory.createCounterState()
  const currentMemoryUsage = yield* factory.createGaugeState()
  const responseTimeDistribution = yield* factory.createHistogramState([0.1, 0.5, 1.0, 2.0, 5.0])
  const latencyQuantiles = yield* factory.createSummaryState([0.5, 0.95, 0.99])
  const errorTypes = yield* factory.createFrequencyState(["timeout", "validation", "network"])
  
  return {
    httpRequestCount,
    currentMemoryUsage,
    responseTimeDistribution,
    latencyQuantiles,
    errorTypes
  }
})
```

### Pattern 2: Metric State Update Helpers

Create composable helpers for updating metric states safely:

```typescript
// Safe metric state update utilities
const createMetricStateUpdaters = Effect.gen(function* () {
  const updateCounterState = (
    current: MetricState.MetricState.Counter<number>,
    increment: number
  ) => Effect.gen(function* () {
    if (increment < 0) {
      return yield* Effect.fail(new Error("Counter increment cannot be negative"))
    }
    
    const newValue = current.count + increment
    if (!Number.isFinite(newValue)) {
      return yield* Effect.fail(new Error("Counter overflow"))
    }
    
    return MetricState.counter(newValue)
  })
  
  const updateGaugeState = (
    current: MetricState.MetricState.Gauge<number>,
    newValue: number
  ) => Effect.gen(function* () {
    if (!Number.isFinite(newValue)) {
      return yield* Effect.fail(new Error("Gauge value must be finite"))
    }
    
    return MetricState.gauge(newValue)
  })
  
  const updateHistogramState = (
    current: MetricState.MetricState.Histogram,
    value: number
  ) => Effect.gen(function* () {
    if (!Number.isFinite(value)) {
      return yield* Effect.fail(new Error("Histogram value must be finite"))
    }
    
    // Update buckets
    const newBuckets = current.buckets.map(([boundary, count]) => 
      [boundary, value <= boundary ? count + 1 : count] as const
    )
    
    // Update aggregates
    const newCount = current.count + 1
    const newMin = current.count === 0 ? value : Math.min(current.min, value)
    const newMax = current.count === 0 ? value : Math.max(current.max, value)
    const newSum = current.sum + value
    
    return MetricState.histogram({
      buckets: newBuckets,
      count: newCount,
      min: newMin,
      max: newMax,
      sum: newSum
    })
  })
  
  const updateFrequencyState = (
    current: MetricState.MetricState.Frequency,
    key: string,
    increment: number = 1
  ) => Effect.gen(function* () {
    if (increment < 0) {
      return yield* Effect.fail(new Error("Frequency increment cannot be negative"))
    }
    
    const newOccurrences = new Map(current.occurrences)
    newOccurrences.set(key, (newOccurrences.get(key) || 0) + increment)
    
    return MetricState.frequency(newOccurrences)
  })
  
  const batchUpdateHistogram = (
    current: MetricState.MetricState.Histogram,
    values: number[]
  ) => Effect.gen(function* () {
    if (values.length === 0) {
      return current
    }
    
    // Validate all values first
    for (const value of values) {
      if (!Number.isFinite(value)) {
        return yield* Effect.fail(new Error(`Invalid histogram value: ${value}`))
      }
    }
    
    // Update buckets for all values
    const newBuckets = current.buckets.map(([boundary, count]) => {
      const additionalCount = values.filter(v => v <= boundary).length
      return [boundary, count + additionalCount] as const
    })
    
    // Update aggregates
    const newCount = current.count + values.length
    const newMin = current.count === 0 ? Math.min(...values) : Math.min(current.min, ...values)
    const newMax = current.count === 0 ? Math.max(...values) : Math.max(current.max, ...values)
    const newSum = current.sum + values.reduce((sum, v) => sum + v, 0)
    
    return MetricState.histogram({
      buckets: newBuckets,
      count: newCount,
      min: newMin,
      max: newMax,
      sum: newSum
    })
  })
  
  return {
    updateCounterState,
    updateGaugeState,
    updateHistogramState,
    updateFrequencyState,
    batchUpdateHistogram
  } as const
})

// Usage in metric collection
const collectHttpMetrics = (responseTime: number, statusCode: number) => Effect.gen(function* () {
  const updaters = yield* createMetricStateUpdaters
  
  // Sample current states (would come from metric registry)
  let requestCount = MetricState.counter(100)
  let responseHist = MetricState.histogram({
    buckets: [[0.1, 10], [0.5, 20], [1.0, 5]],
    count: 35,
    min: 0.05,
    max: 0.8,
    sum: 15.2
  })
  let errorFreq = MetricState.frequency(new Map([["4xx", 5], ["5xx", 2]]))
  
  // Update metrics
  requestCount = yield* updaters.updateCounterState(requestCount, 1)
  responseHist = yield* updaters.updateHistogramState(responseHist, responseTime)
  
  if (statusCode >= 400) {
    const errorType = statusCode >= 500 ? "5xx" : "4xx"
    errorFreq = yield* updaters.updateFrequencyState(errorFreq, errorType)
  }
  
  yield* Effect.log(`Updated HTTP metrics: ${requestCount.count} requests, latest response: ${responseTime}ms`)
  
  return { requestCount, responseHist, errorFreq }
})
```

### Pattern 3: Metric State Comparison and Equality

Compare metric states for changes, equality, and analysis:

```typescript
// Metric state comparison utilities
const createMetricStateComparators = Effect.gen(function* () {
  const compareCounterStates = (
    state1: MetricState.MetricState.Counter<number>,
    state2: MetricState.MetricState.Counter<number>
  ) => ({
    equal: Equal.equals(state1, state2),
    difference: state2.count - state1.count,
    percentChange: state1.count === 0 ? 0 : ((state2.count - state1.count) / state1.count) * 100
  })
  
  const compareGaugeStates = (
    state1: MetricState.MetricState.Gauge<number>,
    state2: MetricState.MetricState.Gauge<number>
  ) => ({
    equal: Equal.equals(state1, state2),
    difference: state2.value - state1.value,
    percentChange: state1.value === 0 ? 0 : ((state2.value - state1.value) / state1.value) * 100
  })
  
  const compareHistogramStates = (
    state1: MetricState.MetricState.Histogram,
    state2: MetricState.MetricState.Histogram
  ) => ({
    equal: Equal.equals(state1, state2),
    countDifference: state2.count - state1.count,
    sumDifference: state2.sum - state1.sum,
    averageChange: {
      before: state1.count > 0 ? state1.sum / state1.count : 0,
      after: state2.count > 0 ? state2.sum / state2.count : 0
    },
    bucketChanges: state1.buckets.map(([boundary, count1], index) => {
      const count2 = state2.buckets[index]?.[1] || 0
      return {
        boundary,
        countDifference: count2 - count1,
        percentChange: count1 === 0 ? 0 : ((count2 - count1) / count1) * 100
      }
    })
  })
  
  const compareFrequencyStates = (
    state1: MetricState.MetricState.Frequency,
    state2: MetricState.MetricState.Frequency
  ) => {
    const allKeys = new Set([...state1.occurrences.keys(), ...state2.occurrences.keys()])
    const keyChanges = Array.from(allKeys).map(key => {
      const count1 = state1.occurrences.get(key) || 0
      const count2 = state2.occurrences.get(key) || 0
      return {
        key,
        countDifference: count2 - count1,
        percentChange: count1 === 0 ? 0 : ((count2 - count1) / count1) * 100
      }
    })
    
    return {
      equal: Equal.equals(state1, state2),
      keyChanges,
      newKeys: Array.from(allKeys).filter(key => !state1.occurrences.has(key)),
      removedKeys: Array.from(state1.occurrences.keys()).filter(key => !state2.occurrences.has(key))
    }
  }
  
  const analyzeStateChanges = (
    before: MetricState.MetricState.Untyped,
    after: MetricState.MetricState.Untyped
  ) => Effect.gen(function* () {
    if (MetricState.isCounterState(before) && MetricState.isCounterState(after)) {
      const comparison = compareCounterStates(before, after)
      yield* Effect.log(`Counter change: ${comparison.difference} (${comparison.percentChange.toFixed(2)}%)`)
      return comparison
    } else if (MetricState.isGaugeState(before) && MetricState.isGaugeState(after)) {
      const comparison = compareGaugeStates(before, after)
      yield* Effect.log(`Gauge change: ${comparison.difference} (${comparison.percentChange.toFixed(2)}%)`)
      return comparison
    } else if (MetricState.isHistogramState(before) && MetricState.isHistogramState(after)) {
      const comparison = compareHistogramStates(before, after)
      yield* Effect.log(`Histogram change: ${comparison.countDifference} samples, avg: ${comparison.averageChange.before.toFixed(3)} -> ${comparison.averageChange.after.toFixed(3)}`)
      return comparison
    } else if (MetricState.isFrequencyState(before) && MetricState.isFrequencyState(after)) {
      const comparison = compareFrequencyStates(before, after)
      yield* Effect.log(`Frequency change: ${comparison.newKeys.length} new keys, ${comparison.removedKeys.length} removed keys`)
      return comparison
    } else {
      return yield* Effect.fail(new Error("Cannot compare different metric state types"))
    }
  })
  
  return {
    compareCounterStates,
    compareGaugeStates,
    compareHistogramStates,
    compareFrequencyStates,
    analyzeStateChanges
  } as const
})

// Usage in monitoring system
const monitorMetricStateChanges = Effect.gen(function* () {
  const comparators = yield* createMetricStateComparators
  
  // Simulate metric state changes over time
  const beforeState = MetricState.counter(100)
  const afterState = MetricState.counter(150)
  
  const changes = yield* comparators.analyzeStateChanges(beforeState, afterState)
  
  // Alert on significant changes
  if ('percentChange' in changes && Math.abs(changes.percentChange) > 20) {
    yield* Effect.log(`⚠️  Significant metric change detected: ${changes.percentChange.toFixed(2)}%`)
  }
  
  return changes
})
```

## Integration Examples

### Integration with Prometheus Export

Export MetricState values in Prometheus format for monitoring dashboards:

```typescript
import { MetricState, Effect, Array as Arr } from "effect"

// Prometheus exporter for metric states
const createPrometheusExporter = Effect.gen(function* () {
  const formatCounterState = (
    name: string,
    state: MetricState.MetricState.Counter<number>,
    labels: Record<string, string> = {}
  ) => {
    const labelStr = Object.entries(labels)
      .map(([k, v]) => `${k}="${v}"`)
      .join(",")
    const labelSuffix = labelStr ? `{${labelStr}}` : ""
    
    return [
      `# TYPE ${name} counter`,
      `${name}${labelSuffix} ${state.count}`
    ].join("\n")
  }
  
  const formatGaugeState = (
    name: string,
    state: MetricState.MetricState.Gauge<number>,
    labels: Record<string, string> = {}
  ) => {
    const labelStr = Object.entries(labels)
      .map(([k, v]) => `${k}="${v}"`)
      .join(",")
    const labelSuffix = labelStr ? `{${labelStr}}` : ""
    
    return [
      `# TYPE ${name} gauge`,
      `${name}${labelSuffix} ${state.value}`
    ].join("\n")
  }
  
  const formatHistogramState = (
    name: string,
    state: MetricState.MetricState.Histogram,
    labels: Record<string, string> = {}
  ) => {
    const labelStr = Object.entries(labels)
      .map(([k, v]) => `${k}="${v}"`)
      .join(",")
    const labelPrefix = labelStr ? `{${labelStr},` : "{"
    
    const lines = [`# TYPE ${name} histogram`]
    
    // Export buckets
    for (const [boundary, count] of state.buckets) {
      lines.push(`${name}_bucket${labelPrefix}le="${boundary}"} ${count}`)
    }
    
    // Export +Inf bucket
    lines.push(`${name}_bucket${labelPrefix}le="+Inf"} ${state.count}`)
    
    // Export count and sum
    const labelSuffix = labelStr ? `{${labelStr}}` : ""
    lines.push(`${name}_count${labelSuffix} ${state.count}`)
    lines.push(`${name}_sum${labelSuffix} ${state.sum}`)
    
    return lines.join("\n")
  }
  
  const formatSummaryState = (
    name: string,
    state: MetricState.MetricState.Summary,
    labels: Record<string, string> = {}
  ) => {
    const labelStr = Object.entries(labels)
      .map(([k, v]) => `${k}="${v}"`)
      .join(",")
    const labelPrefix = labelStr ? `{${labelStr},` : "{"
    
    const lines = [`# TYPE ${name} summary`]
    
    // Export quantiles
    for (const [quantile, value] of state.quantiles) {
      if (Option.isSome(value)) {
        lines.push(`${name}${labelPrefix}quantile="${quantile}"} ${value.value}`)
      }
    }
    
    // Export count and sum
    const labelSuffix = labelStr ? `{${labelStr}}` : ""
    lines.push(`${name}_count${labelSuffix} ${state.count}`)
    lines.push(`${name}_sum${labelSuffix} ${state.sum}`)
    
    return lines.join("\n")
  }
  
  const exportMetricState = (
    name: string,
    state: MetricState.MetricState.Untyped,
    labels: Record<string, string> = {}
  ) => Effect.gen(function* () {
    if (MetricState.isCounterState(state)) {
      return formatCounterState(name, state, labels)
    } else if (MetricState.isGaugeState(state)) {
      return formatGaugeState(name, state, labels)
    } else if (MetricState.isHistogramState(state)) {
      return formatHistogramState(name, state, labels)
    } else if (MetricState.isSummaryState(state)) {
      return formatSummaryState(name, state, labels)
    } else if (MetricState.isFrequencyState(state)) {
      // Convert frequency to multiple gauge metrics
      const lines = [`# TYPE ${name} gauge`]
      for (const [key, count] of state.occurrences) {
        const labelStr = Object.entries({...labels, value: key})
          .map(([k, v]) => `${k}="${v}"`)
          .join(",")
        lines.push(`${name}{${labelStr}} ${count}`)
      }
      return lines.join("\n")
    } else {
      return yield* Effect.fail(new Error(`Unknown metric state type for ${name}`))
    }
  })
  
  const exportMultipleStates = (states: Map<string, MetricState.MetricState.Untyped>) => Effect.gen(function* () {
    const exported: string[] = []
    
    for (const [name, state] of states) {
      const formatted = yield* exportMetricState(name, state)
      exported.push(formatted)
    }
    
    return exported.join("\n\n")
  })
  
  return { exportMetricState, exportMultipleStates } as const
})

// Usage example
const exportApplicationMetrics = Effect.gen(function* () {
  const exporter = yield* createPrometheusExporter
  
  // Sample application metrics
  const metrics = new Map([
    ["http_requests_total", MetricState.counter(1500)],
    ["memory_usage_bytes", MetricState.gauge(1048576)],
    ["http_request_duration_seconds", MetricState.histogram({
      buckets: [[0.1, 100], [0.5, 200], [1.0, 50], [2.0, 10]],
      count: 360,
      min: 0.05,
      max: 1.8,
      sum: 180.5
    })],
    ["error_types", MetricState.frequency(new Map([
      ["timeout", 5],
      ["validation", 12],
      ["database", 3]
    ]))]
  ])
  
  const prometheusOutput = yield* exporter.exportMultipleStates(metrics)
  
  yield* Effect.log("Prometheus metrics export:")
  yield* Effect.log(prometheusOutput)
  
  return prometheusOutput
})
```

### Integration with Custom Metric Storage

Store and retrieve metric states from various storage backends:

```typescript
import { MetricState, Effect, Context, Layer } from "effect"

// Storage interface for metric states
interface MetricStateStorage {
  readonly store: (key: string, state: MetricState.MetricState.Untyped) => Effect.Effect<void>
  readonly retrieve: (key: string) => Effect.Effect<MetricState.MetricState.Untyped>
  readonly list: () => Effect.Effect<Array<[string, MetricState.MetricState.Untyped]>>
  readonly delete: (key: string) => Effect.Effect<void>
}

const MetricStateStorage = Context.GenericTag<MetricStateStorage>("@app/MetricStateStorage")

// In-memory storage implementation
const makeInMemoryStorage = Effect.gen(function* () {
  const storage = yield* Effect.succeed(new Map<string, MetricState.MetricState.Untyped>())
  
  const store = (key: string, state: MetricState.MetricState.Untyped) => Effect.gen(function* () {
    storage.set(key, state)
    yield* Effect.log(`Stored metric state: ${key}`)
  })
  
  const retrieve = (key: string) => Effect.gen(function* () {
    const state = storage.get(key)
    if (!state) {
      return yield* Effect.fail(new Error(`Metric state not found: ${key}`))
    }
    return state
  })
  
  const list = Effect.gen(function* () {
    return Array.from(storage.entries())
  })
  
  const deleteState = (key: string) => Effect.gen(function* () {
    const deleted = storage.delete(key)
    if (!deleted) {
      return yield* Effect.fail(new Error(`Metric state not found: ${key}`))
    }
    yield* Effect.log(`Deleted metric state: ${key}`)
  })
  
  return MetricStateStorage.of({
    store,
    retrieve,
    list,
    delete: deleteState
  })
})

// Redis-like storage implementation (mock)
const makeRedisStorage = Effect.gen(function* () {
  const store = (key: string, state: MetricState.MetricState.Untyped) => Effect.gen(function* () {
    const serialized = JSON.stringify(serializeMetricState(state))
    // Mock Redis call: yield* RedisClient.set(key, serialized)
    yield* Effect.log(`Stored to Redis: ${key} = ${serialized}`)
  })
  
  const retrieve = (key: string) => Effect.gen(function* () {
    // Mock Redis call: const serialized = yield* RedisClient.get(key)
    const mockSerialized = '{"type":"counter","count":42}'
    
    if (!mockSerialized) {
      return yield* Effect.fail(new Error(`Metric state not found in Redis: ${key}`))
    }
    
    const parsed = JSON.parse(mockSerialized)
    return deserializeMetricState(parsed)
  })
  
  const list = Effect.gen(function* () {
    // Mock Redis call: const keys = yield* RedisClient.keys("metric:*")
    const mockKeys = ["metric:requests", "metric:memory"]
    
    const entries: Array<[string, MetricState.MetricState.Untyped]> = []
    for (const key of mockKeys) {
      const state = yield* retrieve(key)
      entries.push([key, state])
    }
    
    return entries
  })
  
  const deleteState = (key: string) => Effect.gen(function* () {
    // Mock Redis call: yield* RedisClient.del(key)
    yield* Effect.log(`Deleted from Redis: ${key}`)
  })
  
  return MetricStateStorage.of({
    store,
    retrieve,
    list,
    delete: deleteState
  })
})

// Metric state manager using storage
const createMetricStateManager = Effect.gen(function* () {
  const storage = yield* MetricStateStorage
  
  const saveMetricState = (name: string, state: MetricState.MetricState.Untyped) => 
    storage.store(name, state)
  
  const loadMetricState = (name: string) => 
    storage.retrieve(name)
  
  const getAllMetricStates = () => 
    storage.list()
  
  const updateCounterMetric = (name: string, increment: number) => Effect.gen(function* () {
    const currentState = yield* storage.retrieve(name).pipe(
      Effect.catchAll(() => Effect.succeed(MetricState.counter(0)))
    )
    
    if (!MetricState.isCounterState(currentState)) {
      return yield* Effect.fail(new Error(`Expected counter state for ${name}`))
    }
    
    const newState = MetricState.counter(currentState.count + increment)
    yield* storage.store(name, newState)
    
    return newState
  })
  
  const updateGaugeMetric = (name: string, value: number) => Effect.gen(function* () {
    const newState = MetricState.gauge(value)
    yield* storage.store(name, newState)
    return newState
  })
  
  const snapshotAllMetrics = Effect.gen(function* () {
    const allStates = yield* storage.list()
    const timestamp = new Date().toISOString()
    
    yield* Effect.log(`Snapshot taken at ${timestamp}: ${allStates.length} metrics`)
    
    return {
      timestamp,
      metrics: allStates,
      count: allStates.length
    }
  })
  
  return {
    saveMetricState,
    loadMetricState,
    getAllMetricStates,
    updateCounterMetric,
    updateGaugeMetric,
    snapshotAllMetrics
  } as const
})

// Layers for different storage implementations
const InMemoryStorageLayer = Layer.effect(MetricStateStorage, makeInMemoryStorage)
const RedisStorageLayer = Layer.effect(MetricStateStorage, makeRedisStorage)

// Usage example
const useMetricStateManager = Effect.gen(function* () {
  const manager = yield* createMetricStateManager
  
  // Update some metrics
  yield* manager.updateCounterMetric("http_requests", 1)
  yield* manager.updateGaugeMetric("memory_usage", 1048576)
  
  // Take a snapshot
  const snapshot = yield* manager.snapshotAllMetrics
  
  yield* Effect.log(`Snapshot contains ${snapshot.count} metrics`)
  
  return snapshot
}).pipe(
  Effect.provide(InMemoryStorageLayer) // or RedisStorageLayer
)
```

### Testing Strategies

Comprehensive testing approaches for MetricState-based systems:

```typescript
// Test utilities for MetricState
const createMetricStateTestUtils = Effect.gen(function* () {
  const createTestCounter = (value: number) => 
    MetricState.counter(value)
  
  const createTestGauge = (value: number) => 
    MetricState.gauge(value)
  
  const createTestHistogram = (samples: number[]) => Effect.gen(function* () {
    const boundaries = [0.1, 0.5, 1.0, 2.0, 5.0]
    const buckets = boundaries.map(boundary => [
      boundary,
      samples.filter(s => s <= boundary).length
    ] as const)
    
    return MetricState.histogram({
      buckets,
      count: samples.length,
      min: samples.length > 0 ? Math.min(...samples) : 0,
      max: samples.length > 0 ? Math.max(...samples) : 0,
      sum: samples.reduce((sum, s) => sum + s, 0)
    })
  })
  
  const assertCounterState = (
    state: MetricState.MetricState.Untyped,
    expectedCount: number
  ) => Effect.gen(function* () {
    if (!MetricState.isCounterState(state)) {
      return yield* Effect.fail(new Error("Expected counter state"))
    }
    
    if (state.count !== expectedCount) {
      return yield* Effect.fail(
        new Error(`Expected count ${expectedCount}, got ${state.count}`)
      )
    }
    
    yield* Effect.log(`✓ Counter assertion passed: ${expectedCount}`)
  })
  
  const assertGaugeState = (
    state: MetricState.MetricState.Untyped,
    expectedValue: number,
    tolerance: number = 0.01
  ) => Effect.gen(function* () {
    if (!MetricState.isGaugeState(state)) {
      return yield* Effect.fail(new Error("Expected gauge state"))
    }
    
    if (Math.abs(state.value - expectedValue) > tolerance) {
      return yield* Effect.fail(
        new Error(`Expected value ${expectedValue} ± ${tolerance}, got ${state.value}`)
      )
    }
    
    yield* Effect.log(`✓ Gauge assertion passed: ${expectedValue}`)
  })
  
  const assertHistogramState = (
    state: MetricState.MetricState.Untyped,
    expectedCount: number,
    expectedSum?: number
  ) => Effect.gen(function* () {
    if (!MetricState.isHistogramState(state)) {
      return yield* Effect.fail(new Error("Expected histogram state"))
    }
    
    if (state.count !== expectedCount) {
      return yield* Effect.fail(
        new Error(`Expected count ${expectedCount}, got ${state.count}`)
      )
    }
    
    if (expectedSum !== undefined && Math.abs(state.sum - expectedSum) > 0.01) {
      return yield* Effect.fail(
        new Error(`Expected sum ${expectedSum}, got ${state.sum}`)
      )
    }
    
    yield* Effect.log(`✓ Histogram assertion passed: ${expectedCount} samples`)
  })
  
  const mockMetricStateManager = (initialStates: Map<string, MetricState.MetricState.Untyped>) => {
    const states = new Map(initialStates)
    
    return {
      getState: (name: string) => states.get(name),
      setState: (name: string, state: MetricState.MetricState.Untyped) => {
        states.set(name, state)
      },
      getAllStates: () => Array.from(states.entries()),
      reset: () => states.clear()
    }
  }
  
  return {
    createTestCounter,
    createTestGauge,
    createTestHistogram,
    assertCounterState,
    assertGaugeState,
    assertHistogramState,
    mockMetricStateManager
  } as const
})

// Example test suite
const testMetricStateOperations = Effect.gen(function* () {
  const utils = yield* createMetricStateTestUtils
  
  // Test counter state creation and updates
  yield* Effect.log("Testing counter states...")
  const counter = utils.createTestCounter(0)
  yield* utils.assertCounterState(counter, 0)
  
  const incrementedCounter = MetricState.counter(counter.count + 5)
  yield* utils.assertCounterState(incrementedCounter, 5)
  
  // Test gauge state operations
  yield* Effect.log("Testing gauge states...")
  const gauge = utils.createTestGauge(42.5)
  yield* utils.assertGaugeState(gauge, 42.5)
  
  // Test histogram state creation
  yield* Effect.log("Testing histogram states...")
  const histogram = yield* utils.createTestHistogram([0.1, 0.3, 0.7, 1.2, 0.5])
  yield* utils.assertHistogramState(histogram, 5, 2.8)
  
  // Test with mock manager
  yield* Effect.log("Testing with mock manager...")
  const mockManager = utils.mockMetricStateManager(new Map([
    ["test_counter", MetricState.counter(10)],
    ["test_gauge", MetricState.gauge(3.14)]
  ]))
  
  const counterState = mockManager.getState("test_counter")
  if (counterState) {
    yield* utils.assertCounterState(counterState, 10)
  }
  
  const gaugeState = mockManager.getState("test_gauge")
  if (gaugeState) {
    yield* utils.assertGaugeState(gaugeState, 3.14)
  }
  
  yield* Effect.log("All MetricState tests passed!")
})
```

## Conclusion

MetricState provides a unified, type-safe foundation for representing metric values across all metric types in Effect applications, enabling robust monitoring and observability systems.

Key benefits:
- **Type Safety**: Compile-time guarantees that metric state values match their expected types
- **Unified Interface**: Consistent API for working with different metric types (counters, gauges, histograms, summaries, frequencies)
- **Composability**: States can be merged, compared, and transformed using functional patterns
- **Serialization Ready**: Built-in support for JSON serialization and deserialization for persistence
- **Integration Friendly**: Easy integration with monitoring systems like Prometheus, storage backends, and custom dashboards

MetricState is essential when building production monitoring systems, analytics dashboards, or any application requiring reliable metric collection and state management with strong type safety guarantees.