# MetricRegistry: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem MetricRegistry Solves

When building production applications, managing metrics across different components becomes challenging. Traditional approaches often lead to scattered metric collection, inconsistent metric naming, and difficulty in aggregating application-wide statistics.

```typescript
// Traditional approach - scattered metric management
class UserService {
  private userCreationCount = 0
  private userLoginCount = 0
  
  createUser() {
    this.userCreationCount++
    // Manual metric tracking
  }
  
  loginUser() {
    this.userLoginCount++
    // Manual metric tracking
  }
}

class OrderService {
  private orderCount = 0
  private orderTotal = 0
  
  createOrder(amount: number) {
    this.orderCount++
    this.orderTotal += amount
    // Manual metric tracking
  }
}

// No centralized way to collect all metrics
// Difficult to export metrics consistently
// No type safety or metric discovery
```

This approach leads to:
- **Scattered Metrics** - Metrics spread across different services without centralization
- **Inconsistent Collection** - Each service implements its own metric tracking
- **No Aggregation** - Difficulty in getting application-wide metric snapshots
- **Poor Observability** - No unified way to export metrics to monitoring systems

### The MetricRegistry Solution

MetricRegistry provides a centralized registry for all application metrics, enabling unified metric collection, aggregation, and export.

```typescript
import { Effect, MetricRegistry, MetricKey, Metric } from "effect"

// Create a centralized registry
const registry = MetricRegistry.make()

// Define metric keys
const userCreations = MetricKey.counter("user_creations")
const activeUsers = MetricKey.gauge("active_users")
const requestDuration = MetricKey.histogram("request_duration_ms", [10, 50, 100, 500, 1000])

// Get metric hooks from registry
const userCreationMetric = registry.getCounter(userCreations)
const activeUserMetric = registry.getGauge(activeUsers)
const durationMetric = registry.getHistogram(requestDuration)

// Centralized snapshot collection
const snapshot = registry.snapshot()
```

### Key Concepts

**MetricRegistry**: A centralized store that manages all application metrics and provides unified access to metric hooks

**MetricHook**: The actual metric implementation that handles updates and value retrieval for a specific metric key

**Snapshot**: A point-in-time collection of all registered metrics and their current values, useful for metric export

## Basic Usage Patterns

### Pattern 1: Registry Creation and Basic Registration

```typescript
import { MetricRegistry, MetricKey } from "effect"

// Create a new metric registry
const registry = MetricRegistry.make()

// Define metric keys
const requestCount = MetricKey.counter("http_requests_total")
const responseTime = MetricKey.histogram("http_response_time_ms", [10, 50, 100, 500, 1000])
const activeConnections = MetricKey.gauge("active_connections")

// Get metric hooks from registry (automatically registers them)
const requestCounter = registry.getCounter(requestCount)
const responseTimeHistogram = registry.getHistogram(responseTime)
const connectionsGauge = registry.getGauge(activeConnections)
```

### Pattern 2: Updating Metrics

```typescript
// Update counter metrics
requestCounter.update(1) // Increment by 1
requestCounter.update(5) // Increment by 5

// Update histogram metrics
responseTimeHistogram.update(45) // Record a 45ms response time
responseTimeHistogram.update(120) // Record a 120ms response time

// Update gauge metrics
connectionsGauge.update(10) // Set gauge to 10
connectionsGauge.update(15) // Set gauge to 15
```

### Pattern 3: Collecting Metric Snapshots

```typescript
// Get snapshot of all metrics in the registry
const snapshot = registry.snapshot()

snapshot.forEach(pair => {
  console.log(`Metric: ${pair.metricKey.name}`)
  console.log(`Value:`, pair.metricState)
  console.log(`Tags:`, pair.metricKey.tags)
})
```

## Real-World Examples

### Example 1: HTTP Server Metrics Collection

Building a comprehensive metrics system for an HTTP server that tracks requests, response times, and error rates.

```typescript
import { Effect, MetricRegistry, MetricKey, Layer } from "effect"
import { Array as Arr } from "effect"

// Define HTTP server metrics
const httpMetrics = {
  requestCount: MetricKey.counter("http_requests_total"),
  responseTime: MetricKey.histogram("http_response_time_ms", [10, 50, 100, 500, 1000, 5000]),
  activeRequests: MetricKey.gauge("http_active_requests"),
  errorCount: MetricKey.counter("http_errors_total"),
  requestSize: MetricKey.histogram("http_request_size_bytes", [100, 1000, 10000, 100000])
}

interface HttpMetricsService {
  readonly recordRequest: (method: string, path: string) => Effect.Effect<void>
  readonly recordResponse: (statusCode: number, duration: number) => Effect.Effect<void>
  readonly recordError: (error: string) => Effect.Effect<void>
  readonly getSnapshot: () => Effect.Effect<ReadonlyArray<{ name: string; value: unknown }>>
}

const makeHttpMetricsService = Effect.gen(function* () {
  const registry = MetricRegistry.make()
  
  // Get metric hooks
  const requestCounter = registry.getCounter(httpMetrics.requestCount)
  const responseTimeHistogram = registry.getHistogram(httpMetrics.responseTime)
  const activeRequestsGauge = registry.getGauge(httpMetrics.activeRequests)
  const errorCounter = registry.getCounter(httpMetrics.errorCount)
  const requestSizeHistogram = registry.getHistogram(httpMetrics.requestSize)

  const recordRequest = (method: string, path: string) => Effect.gen(function* () {
    // Create tagged metric key for this specific request
    const taggedRequestCount = httpMetrics.requestCount.pipe(
      MetricKey.tagged("method", method),
      MetricKey.tagged("path", path)
    )
    const taggedCounter = registry.getCounter(taggedRequestCount)
    
    taggedCounter.update(1)
    activeRequestsGauge.modify(current => current + 1)
  })

  const recordResponse = (statusCode: number, duration: number) => Effect.gen(function* () {
    const statusClass = Math.floor(statusCode / 100) + "xx"
    
    responseTimeHistogram.update(duration)
    activeRequestsGauge.modify(current => Math.max(0, current - 1))
    
    if (statusCode >= 400) {
      const taggedErrorCount = httpMetrics.errorCount.pipe(
        MetricKey.tagged("status_code", statusCode.toString()),
        MetricKey.tagged("status_class", statusClass)
      )
      const taggedErrorCounter = registry.getCounter(taggedErrorCount)
      taggedErrorCounter.update(1)
    }
  })

  const recordError = (error: string) => Effect.gen(function* () {
    const taggedErrorCount = httpMetrics.errorCount.pipe(
      MetricKey.tagged("error_type", error)
    )
    const taggedErrorCounter = registry.getCounter(taggedErrorCount)
    taggedErrorCounter.update(1)
  })

  const getSnapshot = () => Effect.gen(function* () {
    const snapshot = registry.snapshot()
    return snapshot.map(pair => ({
      name: pair.metricKey.name,
      value: pair.metricState,
      tags: pair.metricKey.tags
    }))
  })

  return { recordRequest, recordResponse, recordError, getSnapshot }
}).pipe(
  Effect.catchAll(error => Effect.die(`Failed to create HTTP metrics service: ${error}`))
)

const HttpMetricsService = Layer.effect("HttpMetricsService", makeHttpMetricsService)

// Usage in HTTP handlers
const handleRequest = (method: string, path: string, handler: () => Effect.Effect<Response>) =>
  Effect.gen(function* () {
    const metricsService = yield* Effect.service<HttpMetricsService>()
    const startTime = Date.now()
    
    yield* metricsService.recordRequest(method, path)
    
    try {
      const response = yield* handler()
      const duration = Date.now() - startTime
      yield* metricsService.recordResponse(response.status, duration)
      return response
    } catch (error) {
      const duration = Date.now() - startTime
      yield* metricsService.recordResponse(500, duration)
      yield* metricsService.recordError(error.constructor.name)
      throw error
    }
  })
```

### Example 2: Database Connection Pool Metrics

Monitoring database connection pool health and performance with centralized metrics.

```typescript
import { Effect, MetricRegistry, MetricKey, Schedule, Layer } from "effect"
import { Array as Arr } from "effect"

// Database pool metrics definition
const dbPoolMetrics = {
  activeConnections: MetricKey.gauge("db_pool_active_connections"),
  idleConnections: MetricKey.gauge("db_pool_idle_connections"),
  totalConnections: MetricKey.gauge("db_pool_total_connections"),
  connectionWaitTime: MetricKey.histogram("db_pool_connection_wait_ms", [1, 5, 10, 50, 100, 500]),
  queryDuration: MetricKey.histogram("db_query_duration_ms", [1, 10, 50, 100, 500, 1000, 5000]),
  connectionErrors: MetricKey.counter("db_pool_connection_errors"),
  queryErrors: MetricKey.counter("db_query_errors")
}

interface DbPoolMetricsService {
  readonly recordConnectionAcquired: (waitTime: number) => Effect.Effect<void>
  readonly recordConnectionReleased: () => Effect.Effect<void>
  readonly recordQuery: (table: string, operation: string, duration: number) => Effect.Effect<void>
  readonly recordConnectionError: (error: string) => Effect.Effect<void>
  readonly updatePoolStats: (active: number, idle: number, total: number) => Effect.Effect<void>
  readonly startMetricsCollection: () => Effect.Effect<void>
}

const makeDbPoolMetricsService = Effect.gen(function* () {
  const registry = MetricRegistry.make()
  
  // Get metric hooks
  const activeConnectionsGauge = registry.getGauge(dbPoolMetrics.activeConnections)
  const idleConnectionsGauge = registry.getGauge(dbPoolMetrics.idleConnections)
  const totalConnectionsGauge = registry.getGauge(dbPoolMetrics.totalConnections)
  const connectionWaitHistogram = registry.getHistogram(dbPoolMetrics.connectionWaitTime)
  const queryDurationHistogram = registry.getHistogram(dbPoolMetrics.queryDuration)
  const connectionErrorCounter = registry.getCounter(dbPoolMetrics.connectionErrors)
  const queryErrorCounter = registry.getCounter(dbPoolMetrics.queryErrors)

  const recordConnectionAcquired = (waitTime: number) => Effect.gen(function* () {
    connectionWaitHistogram.update(waitTime)
  })

  const recordConnectionReleased = () => Effect.gen(function* () {
    // Connection released - metrics updated via pool stats
  })

  const recordQuery = (table: string, operation: string, duration: number) => Effect.gen(function* () {
    const taggedQueryDuration = dbPoolMetrics.queryDuration.pipe(
      MetricKey.tagged("table", table),
      MetricKey.tagged("operation", operation)
    )
    const taggedHistogram = registry.getHistogram(taggedQueryDuration)
    taggedHistogram.update(duration)
  })

  const recordConnectionError = (error: string) => Effect.gen(function* () {
    const taggedConnectionError = dbPoolMetrics.connectionErrors.pipe(
      MetricKey.tagged("error_type", error)
    )
    const taggedCounter = registry.getCounter(taggedConnectionError)
    taggedCounter.update(1)
  })

  const updatePoolStats = (active: number, idle: number, total: number) => Effect.gen(function* () {
    activeConnectionsGauge.update(active)
    idleConnectionsGauge.update(idle)
    totalConnectionsGauge.update(total)
  })

  const startMetricsCollection = () => Effect.gen(function* () {
    // Start a background task to periodically collect pool metrics
    yield* Effect.forever(
      Effect.gen(function* () {
        // Simulate getting pool stats from actual pool
        const poolStats = yield* getPoolStats()
        yield* updatePoolStats(poolStats.active, poolStats.idle, poolStats.total)
      }).pipe(
        Effect.delay("5 seconds")
      )
    ).pipe(
      Effect.forkDaemon
    )
  })

  return {
    recordConnectionAcquired,
    recordConnectionReleased,
    recordQuery,
    recordConnectionError,
    updatePoolStats,
    startMetricsCollection,
    registry // Expose registry for snapshot access
  }
})

// Helper function to simulate getting pool stats
const getPoolStats = () => Effect.succeed({
  active: Math.floor(Math.random() * 20),
  idle: Math.floor(Math.random() * 10),
  total: 30
})

const DbPoolMetricsService = Layer.effect("DbPoolMetricsService", makeDbPoolMetricsService)
```

### Example 3: Business Metrics Aggregation System

A comprehensive business metrics system that tracks user engagement, feature usage, and business KPIs.

```typescript
import { Effect, MetricRegistry, MetricKey, Duration, Layer } from "effect"
import { Array as Arr } from "effect"

// Business metrics definition
const businessMetrics = {
  userRegistrations: MetricKey.counter("user_registrations_total"),
  userLogins: MetricKey.counter("user_logins_total"),
  featureUsage: MetricKey.counter("feature_usage_total"),
  sessionDuration: MetricKey.histogram("user_session_duration_minutes", [1, 5, 15, 30, 60, 120]),
  activeUsers: MetricKey.gauge("active_users_current"),
  revenueGenerated: MetricKey.counter("revenue_generated_cents", { bigint: true }),
  conversionRate: MetricKey.gauge("conversion_rate_percent"),
  errorFrequency: MetricKey.frequency("error_types")
}

interface BusinessMetricsService {
  readonly recordUserRegistration: (source: string, userType: string) => Effect.Effect<void>
  readonly recordUserLogin: (userId: string, method: string) => Effect.Effect<void>
  readonly recordFeatureUsage: (feature: string, userId: string) => Effect.Effect<void>
  readonly recordSessionEnd: (userId: string, durationMinutes: number) => Effect.Effect<void>
  readonly updateActiveUsers: (count: number) => Effect.Effect<void>
  readonly recordRevenue: (amountCents: bigint, source: string) => Effect.Effect<void>
  readonly updateConversionRate: (rate: number) => Effect.Effect<void>
  readonly recordError: (errorType: string) => Effect.Effect<void>
  readonly generateBusinessReport: () => Effect.Effect<BusinessReport>
}

interface BusinessReport {
  readonly timestamp: Date
  readonly userMetrics: {
    totalRegistrations: number
    totalLogins: number
    activeUsers: number
    avgSessionDuration: number
  }
  readonly revenueMetrics: {
    totalRevenue: bigint
    conversionRate: number
  }
  readonly featureMetrics: ReadonlyArray<{ feature: string; usage: number }>
  readonly errorMetrics: ReadonlyArray<{ errorType: string; frequency: number }>
}

const makeBusinessMetricsService = Effect.gen(function* () {
  const registry = MetricRegistry.make()
  
  // Get metric hooks
  const userRegistrationsCounter = registry.getCounter(businessMetrics.userRegistrations)
  const userLoginsCounter = registry.getCounter(businessMetrics.userLogins)
  const featureUsageCounter = registry.getCounter(businessMetrics.featureUsage)
  const sessionDurationHistogram = registry.getHistogram(businessMetrics.sessionDuration)
  const activeUsersGauge = registry.getGauge(businessMetrics.activeUsers)
  const revenueCounter = registry.getCounter(businessMetrics.revenueGenerated)
  const conversionRateGauge = registry.getGauge(businessMetrics.conversionRate)
  const errorFrequency = registry.getFrequency(businessMetrics.errorFrequency)

  const recordUserRegistration = (source: string, userType: string) => Effect.gen(function* () {
    const taggedRegistrations = businessMetrics.userRegistrations.pipe(
      MetricKey.tagged("source", source),
      MetricKey.tagged("user_type", userType)
    )
    const taggedCounter = registry.getCounter(taggedRegistrations)
    taggedCounter.update(1)
    userRegistrationsCounter.update(1)
  })

  const recordUserLogin = (userId: string, method: string) => Effect.gen(function* () {
    const taggedLogins = businessMetrics.userLogins.pipe(
      MetricKey.tagged("method", method)
    )
    const taggedCounter = registry.getCounter(taggedLogins)
    taggedCounter.update(1)
    userLoginsCounter.update(1)
  })

  const recordFeatureUsage = (feature: string, userId: string) => Effect.gen(function* () {
    const taggedFeatureUsage = businessMetrics.featureUsage.pipe(
      MetricKey.tagged("feature", feature),
      MetricKey.tagged("user_id", userId)
    )
    const taggedCounter = registry.getCounter(taggedFeatureUsage)
    taggedCounter.update(1)
    featureUsageCounter.update(1)
  })

  const recordSessionEnd = (userId: string, durationMinutes: number) => Effect.gen(function* () {
    sessionDurationHistogram.update(durationMinutes)
  })

  const updateActiveUsers = (count: number) => Effect.gen(function* () {
    activeUsersGauge.update(count)
  })

  const recordRevenue = (amountCents: bigint, source: string) => Effect.gen(function* () {
    const taggedRevenue = businessMetrics.revenueGenerated.pipe(
      MetricKey.tagged("source", source)
    )
    const taggedCounter = registry.getCounter(taggedRevenue)
    taggedCounter.update(amountCents)
    revenueCounter.update(amountCents)
  })

  const updateConversionRate = (rate: number) => Effect.gen(function* () {
    conversionRateGauge.update(rate)
  })

  const recordError = (errorType: string) => Effect.gen(function* () {
    errorFrequency.update(errorType)
  })

  const generateBusinessReport = () => Effect.gen(function* () {
    const snapshot = registry.snapshot()
    
    // Extract metrics from snapshot
    const userRegistrations = findMetricValue(snapshot, "user_registrations_total") as number || 0
    const userLogins = findMetricValue(snapshot, "user_logins_total") as number || 0
    const activeUsers = findMetricValue(snapshot, "active_users_current") as number || 0
    const totalRevenue = findMetricValue(snapshot, "revenue_generated_cents") as bigint || BigInt(0)
    const conversionRate = findMetricValue(snapshot, "conversion_rate_percent") as number || 0
    
    // Calculate average session duration from histogram
    const sessionHistogram = findMetricValue(snapshot, "user_session_duration_minutes")
    const avgSessionDuration = calculateAvgFromHistogram(sessionHistogram)
    
    // Get feature usage metrics
    const featureMetrics = extractFeatureMetrics(snapshot)
    
    // Get error frequency metrics
    const errorMetrics = extractErrorMetrics(snapshot)

    return {
      timestamp: new Date(),
      userMetrics: {
        totalRegistrations: userRegistrations,
        totalLogins: userLogins,
        activeUsers,
        avgSessionDuration
      },
      revenueMetrics: {
        totalRevenue,
        conversionRate
      },
      featureMetrics,
      errorMetrics
    }
  })

  return {
    recordUserRegistration,
    recordUserLogin,
    recordFeatureUsage,
    recordSessionEnd,
    updateActiveUsers,
    recordRevenue,
    updateConversionRate,
    recordError,
    generateBusinessReport,
    registry
  }
})

// Helper functions for report generation
const findMetricValue = (snapshot: any[], metricName: string) => {
  const metric = snapshot.find(pair => pair.metricKey.name === metricName)
  return metric?.metricState?.value
}

const calculateAvgFromHistogram = (histogram: any) => {
  // Simplified calculation - in practice would use actual histogram data
  return histogram?.mean || 0
}

const extractFeatureMetrics = (snapshot: any[]) => {
  return snapshot
    .filter(pair => pair.metricKey.name === "feature_usage_total")
    .map(pair => ({
      feature: pair.metricKey.tags.find((tag: any) => tag.key === "feature")?.value || "unknown",
      usage: pair.metricState.value
    }))
}

const extractErrorMetrics = (snapshot: any[]) => {
  return snapshot
    .filter(pair => pair.metricKey.name === "error_types")
    .flatMap(pair => 
      Object.entries(pair.metricState.occurrences || {}).map(([errorType, frequency]) => ({
        errorType,
        frequency: frequency as number
      }))
    )
}

const BusinessMetricsService = Layer.effect("BusinessMetricsService", makeBusinessMetricsService)
```

## Advanced Features Deep Dive

### Feature 1: Dynamic Metric Registration

MetricRegistry automatically registers metrics when you request them via getter methods, enabling dynamic metric creation based on runtime conditions.

#### Basic Dynamic Registration

```typescript
import { Effect, MetricRegistry, MetricKey } from "effect"

const registry = MetricRegistry.make()

const createDynamicMetrics = (userId: string, feature: string) => Effect.gen(function* () {
  // Metrics are automatically registered when first accessed
  const userFeatureCounter = MetricKey.counter(`user_${userId}_${feature}_usage`)
  const counter = registry.getCounter(userFeatureCounter)
  
  counter.update(1)
  return counter
})
```

#### Real-World Dynamic Registration Example

```typescript
interface MetricManager {
  readonly getOrCreateCounter: (name: string, tags: Record<string, string>) => Effect.Effect<any>
  readonly getOrCreateHistogram: (name: string, boundaries: ReadonlyArray<number>) => Effect.Effect<any>
  readonly getAllMetrics: () => Effect.Effect<ReadonlyArray<string>>
}

const makeMetricManager = Effect.gen(function* () {
  const registry = MetricRegistry.make()
  const metricCache = new Map<string, any>()

  const getOrCreateCounter = (name: string, tags: Record<string, string>) => Effect.gen(function* () {
    const cacheKey = `${name}_${Object.entries(tags).map(([k, v]) => `${k}:${v}`).join("_")}`
    
    if (metricCache.has(cacheKey)) {
      return metricCache.get(cacheKey)!
    }

    let metricKey = MetricKey.counter(name)
    for (const [key, value] of Object.entries(tags)) {
      metricKey = metricKey.pipe(MetricKey.tagged(key, value))
    }

    const counter = registry.getCounter(metricKey)
    metricCache.set(cacheKey, counter)
    return counter
  })

  const getOrCreateHistogram = (name: string, boundaries: ReadonlyArray<number>) => Effect.gen(function* () {
    const cacheKey = `${name}_histogram_${boundaries.join("_")}`
    
    if (metricCache.has(cacheKey)) {
      return metricCache.get(cacheKey)!
    }

    const metricKey = MetricKey.histogram(name, boundaries)
    const histogram = registry.getHistogram(metricKey)
    metricCache.set(cacheKey, histogram)
    return histogram
  })

  const getAllMetrics = () => Effect.gen(function* () {
    const snapshot = registry.snapshot()
    return snapshot.map(pair => pair.metricKey.name)
  })

  return { getOrCreateCounter, getOrCreateHistogram, getAllMetrics }
})
```

### Feature 2: Metric Snapshots and Export

The snapshot functionality provides a point-in-time view of all metrics, essential for metric export and monitoring integration.

#### Advanced Snapshot Processing

```typescript
import { Effect, MetricRegistry, Schedule } from "effect"
import { Array as Arr } from "effect"

interface MetricExporter {
  readonly exportMetrics: (format: "prometheus" | "json" | "influxdb") => Effect.Effect<string>
  readonly scheduleExport: (interval: Duration.DurationInput) => Effect.Effect<void>
  readonly getMetricHistory: (metricName: string) => Effect.Effect<ReadonlyArray<MetricSnapshot>>
}

interface MetricSnapshot {
  readonly timestamp: Date
  readonly name: string
  readonly value: unknown
  readonly tags: ReadonlyArray<{ key: string; value: string }>
}

const makeMetricExporter = (registry: MetricRegistry.MetricRegistry) => Effect.gen(function* () {
  const metricHistory = new Map<string, MetricSnapshot[]>()

  const exportMetrics = (format: "prometheus" | "json" | "influxdb") => Effect.gen(function* () {
    const snapshot = registry.snapshot()
    
    switch (format) {
      case "prometheus":
        return formatPrometheus(snapshot)
      case "json":
        return formatJSON(snapshot)
      case "influxdb":
        return formatInfluxDB(snapshot)
      default:
        return yield* Effect.fail(new Error(`Unsupported format: ${format}`))
    }
  })

  const scheduleExport = (interval: Duration.DurationInput) => Effect.gen(function* () {
    yield* Effect.forever(
      Effect.gen(function* () {
        const snapshot = registry.snapshot()
        
        // Store snapshot in history
        snapshot.forEach(pair => {
          const metricSnapshot: MetricSnapshot = {
            timestamp: new Date(),
            name: pair.metricKey.name,
            value: pair.metricState,
            tags: pair.metricKey.tags.map(tag => ({ key: tag.key, value: tag.value }))
          }
          
          const history = metricHistory.get(pair.metricKey.name) || []
          history.push(metricSnapshot)
          
          // Keep only last 100 snapshots per metric
          if (history.length > 100) {
            history.splice(0, history.length - 100)
          }
          
          metricHistory.set(pair.metricKey.name, history)
        })
        
        // Export to external system
        const prometheusFormat = yield* exportMetrics("prometheus")
        yield* sendToPrometheusGateway(prometheusFormat)
      }).pipe(
        Effect.delay(interval)
      )
    ).pipe(
      Effect.forkDaemon
    )
  })

  const getMetricHistory = (metricName: string) => Effect.gen(function* () {
    return metricHistory.get(metricName) || []
  })

  return { exportMetrics, scheduleExport, getMetricHistory }
})

// Format functions
const formatPrometheus = (snapshot: any[]) => Effect.gen(function* () {
  const lines = snapshot.map(pair => {
    const labels = pair.metricKey.tags
      .map((tag: any) => `${tag.key}="${tag.value}"`)
      .join(",")
    const labelsStr = labels ? `{${labels}}` : ""
    
    return `${pair.metricKey.name}${labelsStr} ${pair.metricState.value || 0}`
  })
  
  return lines.join("\n")
})

const formatJSON = (snapshot: any[]) => Effect.gen(function* () {
  const metrics = snapshot.map(pair => ({
    name: pair.metricKey.name,
    value: pair.metricState.value,
    tags: Object.fromEntries(
      pair.metricKey.tags.map((tag: any) => [tag.key, tag.value])
    ),
    timestamp: new Date().toISOString()
  }))
  
  return JSON.stringify({ metrics, timestamp: new Date().toISOString() }, null, 2)
})

const formatInfluxDB = (snapshot: any[]) => Effect.gen(function* () {
  const lines = snapshot.map(pair => {
    const tags = pair.metricKey.tags
      .map((tag: any) => `${tag.key}=${tag.value}`)
      .join(",")
    const tagsStr = tags ? `,${tags}` : ""
    
    return `${pair.metricKey.name}${tagsStr} value=${pair.metricState.value || 0} ${Date.now() * 1000000}`
  })
  
  return lines.join("\n")
})

const sendToPrometheusGateway = (data: string) => Effect.gen(function* () {
  // Implementation would send to actual Prometheus push gateway
  console.log("Sending to Prometheus:", data)
})
```

## Practical Patterns & Best Practices

### Pattern 1: Centralized Metric Configuration

Create a centralized configuration system for all application metrics to ensure consistency and discoverability.

```typescript
import { MetricRegistry, MetricKey } from "effect"

// Centralized metric definitions
export const AppMetrics = {
  // HTTP metrics
  Http: {
    requestCount: MetricKey.counter("http_requests_total"),
    responseTime: MetricKey.histogram("http_response_time_ms", [10, 50, 100, 500, 1000, 5000]),
    activeRequests: MetricKey.gauge("http_active_requests"),
    errorRate: MetricKey.gauge("http_error_rate_percent")
  },
  
  // Database metrics
  Database: {
    connectionPoolActive: MetricKey.gauge("db_pool_active_connections"),
    connectionPoolIdle: MetricKey.gauge("db_pool_idle_connections"),
    queryDuration: MetricKey.histogram("db_query_duration_ms", [1, 10, 50, 100, 500, 1000]),
    queryErrors: MetricKey.counter("db_query_errors_total")
  },
  
  // Business metrics
  Business: {
    userRegistrations: MetricKey.counter("user_registrations_total"),
    activeUsers: MetricKey.gauge("active_users_current"),
    revenueGenerated: MetricKey.counter("revenue_generated_cents", { bigint: true }),
    featureUsage: MetricKey.counter("feature_usage_total")
  },
  
  // System metrics
  System: {
    memoryUsage: MetricKey.gauge("system_memory_usage_bytes", { bigint: true }),
    cpuUsage: MetricKey.gauge("system_cpu_usage_percent"),
    diskUsage: MetricKey.gauge("system_disk_usage_bytes", { bigint: true }),
    uptime: MetricKey.gauge("system_uptime_seconds")
  }
} as const

// Centralized registry factory
export const createAppMetricRegistry = () => {
  const registry = MetricRegistry.make()
  
  // Pre-register commonly used metrics
  const hooks = {
    http: {
      requestCount: registry.getCounter(AppMetrics.Http.requestCount),
      responseTime: registry.getHistogram(AppMetrics.Http.responseTime),
      activeRequests: registry.getGauge(AppMetrics.Http.activeRequests),
      errorRate: registry.getGauge(AppMetrics.Http.errorRate)
    },
    database: {
      connectionPoolActive: registry.getGauge(AppMetrics.Database.connectionPoolActive),
      connectionPoolIdle: registry.getGauge(AppMetrics.Database.connectionPoolIdle),
      queryDuration: registry.getHistogram(AppMetrics.Database.queryDuration),
      queryErrors: registry.getCounter(AppMetrics.Database.queryErrors)
    },
    business: {
      userRegistrations: registry.getCounter(AppMetrics.Business.userRegistrations),
      activeUsers: registry.getGauge(AppMetrics.Business.activeUsers),
      revenueGenerated: registry.getCounter(AppMetrics.Business.revenueGenerated),
      featureUsage: registry.getCounter(AppMetrics.Business.featureUsage)
    },
    system: {
      memoryUsage: registry.getGauge(AppMetrics.System.memoryUsage),
      cpuUsage: registry.getGauge(AppMetrics.System.cpuUsage),
      diskUsage: registry.getGauge(AppMetrics.System.diskUsage),
      uptime: registry.getGauge(AppMetrics.System.uptime)
    }
  }
  
  return { registry, hooks }
}
```

### Pattern 2: Metric Middleware and Decorators

Create reusable middleware patterns for automatic metric collection around common operations.

```typescript
import { Effect, MetricRegistry } from "effect"

// Function decorator for automatic timing metrics
const withTiming = <Args extends ReadonlyArray<unknown>, R, E, A>(
  name: string,
  fn: (...args: Args) => Effect.Effect<A, E, R>
) => {
  const registry = MetricRegistry.make()
  const timingHistogram = registry.getHistogram(
    MetricKey.histogram(`${name}_duration_ms`, [1, 10, 50, 100, 500, 1000, 5000])
  )
  
  return (...args: Args): Effect.Effect<A, E, R> => {
    return Effect.gen(function* () {
      const start = Date.now()
      try {
        const result = yield* fn(...args)
        const duration = Date.now() - start
        timingHistogram.update(duration)
        return result
      } catch (error) {
        const duration = Date.now() - start
        timingHistogram.update(duration)
        throw error
      }
    })
  }
}

// Request counting middleware
const withRequestCounting = <Args extends ReadonlyArray<unknown>, R, E, A>(
  name: string,
  fn: (...args: Args) => Effect.Effect<A, E, R>
) => {
  const registry = MetricRegistry.make()
  const requestCounter = registry.getCounter(MetricKey.counter(`${name}_requests_total`))
  const errorCounter = registry.getCounter(MetricKey.counter(`${name}_errors_total`))
  
  return (...args: Args): Effect.Effect<A, E, R> => {
    return Effect.gen(function* () {
      requestCounter.update(1)
      try {
        return yield* fn(...args)
      } catch (error) {
        errorCounter.update(1)
        throw error
      }
    })
  }
}

// Combined metrics decorator
const withMetrics = <Args extends ReadonlyArray<unknown>, R, E, A>(
  name: string,
  fn: (...args: Args) => Effect.Effect<A, E, R>
) => {
  return withTiming(name, withRequestCounting(name, fn))
}

// Usage example
const databaseQuery = withMetrics(
  "database_user_lookup",
  (userId: string) => Effect.gen(function* () {
    // Simulate database query
    yield* Effect.sleep("100 millis")
    return { id: userId, name: "John Doe" }
  })
)
```

## Integration Examples

### Integration with Prometheus and Grafana

Comprehensive integration with Prometheus for metric collection and Grafana for visualization.

```typescript
import { Effect, MetricRegistry, Layer, Schedule } from "effect"
import { Array as Arr } from "effect"

interface PrometheusIntegration {
  readonly startMetricsServer: (port: number) => Effect.Effect<void>
  readonly pushToGateway: (gatewayUrl: string, jobName: string) => Effect.Effect<void>
  readonly scheduleMetricsPush: (gatewayUrl: string, interval: Duration.DurationInput) => Effect.Effect<void>
}

const makePrometheusIntegration = (registry: MetricRegistry.MetricRegistry) => Effect.gen(function* () {
  
  const formatPrometheusMetrics = () => Effect.gen(function* () {
    const snapshot = registry.snapshot()
    
    const metricLines = snapshot.flatMap(pair => {
      const metricName = pair.metricKey.name
      const labels = pair.metricKey.tags
      const value = pair.metricState
      
      // Handle different metric types
      if (isCounterState(value)) {
        const labelsStr = formatLabels(labels)
        return [
          `# HELP ${metricName} ${pair.metricKey.description || ""}`,
          `# TYPE ${metricName} counter`,
          `${metricName}${labelsStr} ${value.count}`
        ]
      } else if (isGaugeState(value)) {
        const labelsStr = formatLabels(labels)
        return [
          `# HELP ${metricName} ${pair.metricKey.description || ""}`,
          `# TYPE ${metricName} gauge`,
          `${metricName}${labelsStr} ${value.value}`
        ]
      } else if (isHistogramState(value)) {
        const labelsStr = formatLabels(labels)
        const bucketLines = value.buckets.map((bucket, index) => 
          `${metricName}_bucket${formatLabels([...labels, { key: "le", value: bucket.boundary.toString() }])} ${bucket.count}`
        )
        return [
          `# HELP ${metricName} ${pair.metricKey.description || ""}`,
          `# TYPE ${metricName} histogram`,
          ...bucketLines,
          `${metricName}_bucket${formatLabels([...labels, { key: "le", value: "+Inf" }])} ${value.count}`,
          `${metricName}_sum${labelsStr} ${value.sum}`,
          `${metricName}_count${labelsStr} ${value.count}`
        ]
      }
      return []
    })
    
    return metricLines.join("\n")
  })

  const startMetricsServer = (port: number) => Effect.gen(function* () {
    // Implementation would start HTTP server
    // This is a simplified example
    console.log(`Starting Prometheus metrics server on port ${port}`)
    
    // Simulate HTTP server that serves metrics at /metrics endpoint
    yield* Effect.forever(
      Effect.gen(function* () {
        const metricsData = yield* formatPrometheusMetrics()
        // In real implementation, this would be served via HTTP
        console.log("Metrics endpoint ready:", metricsData.split("\n").length, "lines")
      }).pipe(
        Effect.delay("30 seconds")
      )
    ).pipe(
      Effect.forkDaemon
    )
  })

  const pushToGateway = (gatewayUrl: string, jobName: string) => Effect.gen(function* () {
    const metricsData = yield* formatPrometheusMetrics()
    
    // In real implementation, would make HTTP POST to push gateway
    console.log(`Pushing metrics to ${gatewayUrl} for job ${jobName}`)
    console.log("Metrics data:", metricsData.substring(0, 200) + "...")
  })

  const scheduleMetricsPush = (gatewayUrl: string, interval: Duration.DurationInput) => Effect.gen(function* () {
    yield* Effect.forever(
      Effect.gen(function* () {
        yield* pushToGateway(gatewayUrl, "effect-app")
      }).pipe(
        Effect.delay(interval)
      )
    ).pipe(
      Effect.forkDaemon
    )
  })

  return { startMetricsServer, pushToGateway, scheduleMetricsPush }
})

// Helper functions
const formatLabels = (labels: ReadonlyArray<{ key: string; value: string }>) => {
  if (labels.length === 0) return ""
  const labelStr = labels.map(label => `${label.key}="${label.value}"`).join(",")
  return `{${labelStr}}`
}

const isCounterState = (state: any): boolean => {
  return state && typeof state.count === "number"
}

const isGaugeState = (state: any): boolean => {
  return state && typeof state.value === "number"
}

const isHistogramState = (state: any): boolean => {
  return state && Array.isArray(state.buckets)
}

// Layer for Prometheus integration
export const PrometheusIntegrationService = (registry: MetricRegistry.MetricRegistry) =>
  Layer.effect("PrometheusIntegration", makePrometheusIntegration(registry))
```

### Testing Strategies

Comprehensive testing approaches for metric collection and registry functionality.

```typescript
import { Effect, MetricRegistry, TestClock, Duration } from "effect"
import { describe, it, expect } from "vitest"

describe("MetricRegistry Testing", () => {
  
  it("should correctly register and update counter metrics", async () => {
    const registry = MetricRegistry.make()
    const counterKey = MetricKey.counter("test_counter")
    const counter = registry.getCounter(counterKey)
    
    // Update counter
    counter.update(5)
    counter.update(3)
    
    // Take snapshot and verify
    const snapshot = registry.snapshot()
    const counterMetric = snapshot.find(pair => pair.metricKey.name === "test_counter")
    
    expect(counterMetric).toBeDefined()
    expect(counterMetric!.metricState.count).toBe(8)
  })

  it("should handle tagged metrics correctly", async () => {
    const registry = MetricRegistry.make()
    const baseCounterKey = MetricKey.counter("http_requests")
    
    // Create tagged versions
    const getCounter = baseCounterKey.pipe(MetricKey.tagged("method", "GET"))
    const postCounter = baseCounterKey.pipe(MetricKey.tagged("method", "POST"))
    
    const getMetric = registry.getCounter(getCounter)
    const postMetric = registry.getCounter(postCounter)
    
    getMetric.update(10)
    postMetric.update(5)
    
    const snapshot = registry.snapshot()
    expect(snapshot).toHaveLength(2)
    
    const getSnapshot = snapshot.find(pair => 
      pair.metricKey.tags.some(tag => tag.key === "method" && tag.value === "GET")
    )
    const postSnapshot = snapshot.find(pair => 
      pair.metricKey.tags.some(tag => tag.key === "method" && tag.value === "POST")
    )
    
    expect(getSnapshot!.metricState.count).toBe(10)
    expect(postSnapshot!.metricState.count).toBe(5)
  })

  it("should properly handle histogram metrics", async () => {
    const registry = MetricRegistry.make()
    const histogramKey = MetricKey.histogram("response_time", [10, 50, 100, 500])
    const histogram = registry.getHistogram(histogramKey)
    
    // Record some values
    histogram.update(5)   // Below first bucket
    histogram.update(25)  // In first bucket
    histogram.update(75)  // In second bucket
    histogram.update(200) // In third bucket
    histogram.update(800) // Above all buckets
    
    const snapshot = registry.snapshot()
    const histogramMetric = snapshot.find(pair => pair.metricKey.name === "response_time")
    
    expect(histogramMetric).toBeDefined()
    expect(histogramMetric!.metricState.count).toBe(5)
    expect(histogramMetric!.metricState.sum).toBe(1105)
  })

  it("should support metric registry snapshots with time-based testing", async () => {
    const testEffect = Effect.gen(function* () {
      const registry = MetricRegistry.make()
      const gaugeKey = MetricKey.gauge("cpu_usage")
      const gauge = registry.getGauge(gaugeKey)
      
      // Simulate CPU usage over time
      gauge.update(10)
      yield* TestClock.adjust(Duration.seconds(1))
      
      gauge.update(25)
      yield* TestClock.adjust(Duration.seconds(1))
      
      gauge.update(40)
      yield* TestClock.adjust(Duration.seconds(1))
      
      const snapshot = registry.snapshot()
      const cpuMetric = snapshot.find(pair => pair.metricKey.name === "cpu_usage")
      
      expect(cpuMetric!.metricState.value).toBe(40)
      return snapshot.length
    })
    
    const result = await Effect.runPromise(
      testEffect.pipe(Effect.provide(TestClock.TestClockService))
    )
    
    expect(result).toBe(1)
  })

  it("should handle concurrent metric updates correctly", async () => {
    const registry = MetricRegistry.make()
    const counterKey = MetricKey.counter("concurrent_counter")
    const counter = registry.getCounter(counterKey)
    
    // Simulate concurrent updates
    const updates = Array.from({ length: 100 }, (_, i) => 
      Effect.sync(() => counter.update(1))
    )
    
    await Effect.runPromise(Effect.all(updates, { concurrency: 10 }))
    
    const snapshot = registry.snapshot()
    const counterMetric = snapshot.find(pair => pair.metricKey.name === "concurrent_counter")
    
    expect(counterMetric!.metricState.count).toBe(100)
  })

  // Property-based testing example
  it("should maintain metric consistency across operations", async () => {
    const registry = MetricRegistry.make()
    const counterKey = MetricKey.counter("property_test_counter")
    const counter = registry.getCounter(counterKey)
    
    const testProperty = (updates: ReadonlyArray<number>) => Effect.gen(function* () {
      // Apply all updates
      for (const update of updates) {
        counter.update(update)
      }
      
      const snapshot = registry.snapshot()
      const counterMetric = snapshot.find(pair => pair.metricKey.name === "property_test_counter")
      const expectedTotal = updates.reduce((sum, val) => sum + val, 0)
      
      return counterMetric!.metricState.count === expectedTotal
    })
    
    // Test with various update sequences
    const testCases = [
      [1, 2, 3, 4, 5],
      [10, -5, 15, -3],
      [0, 0, 0, 1],
      [100]
    ]
    
    for (const testCase of testCases) {
      const result = await Effect.runPromise(testProperty(testCase))
      expect(result).toBe(true)
    }
  })
})

// Mock helpers for testing
export const createTestRegistry = () => {
  const registry = MetricRegistry.make()
  
  const getMetricValue = (name: string) => {
    const snapshot = registry.snapshot()
    const metric = snapshot.find(pair => pair.metricKey.name === name)
    return metric?.metricState
  }
  
  const getMetricCount = () => {
    return registry.snapshot().length
  }
  
  const resetRegistry = () => {
    // In a real implementation, you might need a reset method
    // For now, create a new registry
    return MetricRegistry.make()
  }
  
  return { registry, getMetricValue, getMetricCount, resetRegistry }
}
```

## Conclusion

MetricRegistry provides centralized metric management, unified collection interfaces, and comprehensive export capabilities for Effect applications.

Key benefits:
- **Centralized Management**: Single registry for all application metrics with automatic registration
- **Type-Safe Operations**: Strongly typed metric keys and hooks prevent runtime errors
- **Flexible Export**: Support for multiple export formats including Prometheus, JSON, and InfluxDB
- **Real-Time Snapshots**: Point-in-time metric collection for monitoring and alerting
- **Dynamic Registration**: Automatic metric registration based on runtime requirements

MetricRegistry is essential for production applications requiring comprehensive observability, metric aggregation, and integration with monitoring systems like Prometheus and Grafana.