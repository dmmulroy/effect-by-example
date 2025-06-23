# MetricPolling: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem MetricPolling Solves

Traditional metric collection often requires manual updates scattered throughout your codebase, leading to:

```typescript
// Traditional approach - scattered metric updates
const processRequest = async (request: Request) => {
  metricsClient.increment('requests.total')
  const start = Date.now()
  
  try {
    const result = await handleRequest(request)
    metricsClient.timing('request.duration', Date.now() - start)
    metricsClient.increment('requests.success')
    return result
  } catch (error) {
    metricsClient.increment('requests.error')
    throw error
  }
}

// Manual system monitoring
setInterval(() => {
  const memUsage = process.memoryUsage()
  metricsClient.gauge('memory.used', memUsage.heapUsed)
  metricsClient.gauge('memory.total', memUsage.heapTotal)
}, 5000)

// Scattered business metrics
const updateInventory = async (item: Item) => {
  await inventoryDb.update(item)
  // Easy to forget this manual update
  metricsClient.gauge('inventory.count', await inventoryDb.count())
}
```

This approach leads to:
- **Scattered Updates** - Metric collection logic mixed with business logic
- **Manual Coordination** - Developers must remember to update metrics
- **Inconsistent Timing** - No standardized polling intervals
- **Resource Waste** - Inefficient one-off polling implementations

### The MetricPolling Solution

MetricPolling provides a declarative way to define metrics that automatically poll for updates on a schedule:

```typescript
import { MetricPolling, Metric, Schedule, Duration, Effect } from "effect"

// Declarative metric polling - clean, automated, composable
const memoryUsagePolling = MetricPolling.make(
  Metric.gauge("memory_used", { description: "Current memory usage in bytes" }),
  Effect.sync(() => process.memoryUsage().heapUsed)
)

const activeConnectionsPolling = MetricPolling.make(
  Metric.gauge("active_connections", { description: "Number of active connections" }),
  Effect.gen(function* () {
    const connectionPool = yield* ConnectionPool
    return yield* connectionPool.getActiveCount()
  })
)
```

### Key Concepts

**MetricPolling**: A combination of a metric and an effect that polls for updates to that metric automatically

**Polling Effect**: An Effect that produces values to feed into the metric - runs on a schedule

**Launch**: Starting a MetricPolling as a background fiber with a specific schedule

## Basic Usage Patterns

### Pattern 1: Simple System Metric Polling

```typescript
import { MetricPolling, Metric, Schedule, Duration, Effect } from "effect"

// Create a metric that polls system memory usage
const memoryPolling = MetricPolling.make(
  Metric.gauge("system_memory_mb", { description: "System memory usage in MB" }),
  Effect.sync(() => Math.round(process.memoryUsage().heapUsed / 1024 / 1024))
)

// Launch the polling with a 5-second interval
const program = Effect.gen(function* () {
  const fiber = yield* memoryPolling.pipe(
    MetricPolling.launch(Schedule.spaced(Duration.seconds(5)))
  )
  
  // Your application logic here
  yield* Effect.sleep(Duration.minutes(1))
  
  // Polling runs automatically in the background
  yield* fiber.interrupt()
})
```

### Pattern 2: Database Metric Polling

```typescript
import { MetricPolling, Metric, Schedule, Duration, Effect } from "effect"

// Poll database connection count
const dbConnectionsPolling = MetricPolling.make(
  Metric.gauge("db_connections", { description: "Active database connections" }),
  Effect.gen(function* () {
    const database = yield* Database
    return yield* database.getActiveConnectionCount()
  })
)

// Poll pending job count
const pendingJobsPolling = MetricPolling.make(
  Metric.gauge("pending_jobs", { description: "Number of pending background jobs" }),
  Effect.gen(function* () {
    const jobQueue = yield* JobQueue
    return yield* jobQueue.getPendingCount()
  })
)
```

### Pattern 3: Custom Business Metric Polling

```typescript
// Poll inventory levels
const inventoryPolling = MetricPolling.make(
  Metric.gauge("inventory_total_items", { description: "Total items in inventory" }),
  Effect.gen(function* () {
    const inventory = yield* InventoryService
    const items = yield* inventory.getAllItems()
    return items.length
  })
)

// Launch with retry policy for resilience
const launchInventoryPolling = inventoryPolling.pipe(
  MetricPolling.retry(Schedule.exponential(Duration.millis(100)).pipe(
    Schedule.intersect(Schedule.recurs(3))
  )),
  MetricPolling.launch(Schedule.spaced(Duration.seconds(30)))
)
```

## Real-World Examples

### Example 1: Application Performance Monitoring

Monitor key application performance metrics automatically:

```typescript
import { MetricPolling, Metric, Schedule, Duration, Effect, Layer } from "effect"

// Memory usage polling
const memoryPolling = MetricPolling.make(
  Metric.gauge("app_memory_mb", { description: "Application memory usage" }),
  Effect.sync(() => Math.round(process.memoryUsage().heapUsed / 1024 / 1024))
)

// CPU usage polling (simplified)
const cpuPolling = MetricPolling.make(
  Metric.gauge("app_cpu_percent", { description: "CPU usage percentage" }),
  Effect.gen(function* () {
    const usage = yield* Effect.promise(() => getCpuUsage())
    return Math.round(usage * 100)
  })
)

// Event loop lag polling
const eventLoopLagPolling = MetricPolling.make(
  Metric.gauge("event_loop_lag_ms", { description: "Event loop lag in milliseconds" }),
  Effect.gen(function* () {
    const start = process.hrtime()
    yield* Effect.sleep(Duration.millis(0))
    const delta = process.hrtime(start)
    return Math.round(delta[0] * 1000 + delta[1] / 1e6)
  })
)

// Application monitoring service
const makeAppMonitoringService = Effect.gen(function* () {
  // Launch all performance metrics with different intervals
  const memoryFiber = yield* memoryPolling.pipe(
    MetricPolling.launch(Schedule.spaced(Duration.seconds(10)))
  )
  
  const cpuFiber = yield* cpuPolling.pipe(
    MetricPolling.launch(Schedule.spaced(Duration.seconds(5)))
  )
  
  const lagFiber = yield* eventLoopLagPolling.pipe(
    MetricPolling.launch(Schedule.spaced(Duration.seconds(1)))
  )
  
  return {
    stop: Effect.gen(function* () {
      yield* memoryFiber.interrupt()
      yield* cpuFiber.interrupt()
      yield* lagFiber.interrupt()
    })
  } as const
})

// Helper function to get CPU usage
const getCpuUsage = (): Promise<number> => {
  return new Promise((resolve) => {
    const startUsage = process.cpuUsage()
    const startTime = process.hrtime()
    
    setTimeout(() => {
      const endUsage = process.cpuUsage(startUsage)
      const endTime = process.hrtime(startTime)
      
      const userTime = endUsage.user / 1000
      const systemTime = endUsage.system / 1000
      const totalTime = (endTime[0] * 1000000 + endTime[1] / 1000) / 1000
      
      resolve((userTime + systemTime) / totalTime)
    }, 100)
  })
}
```

### Example 2: Database Connection Pool Monitoring

Monitor database health and connection pool metrics:

```typescript
import { MetricPolling, Metric, Schedule, Duration, Effect, Context } from "effect"

// Database connection pool monitoring
interface ConnectionPool {
  readonly getActiveConnections: () => Effect.Effect<number>
  readonly getIdleConnections: () => Effect.Effect<number>
  readonly getMaxConnections: () => Effect.Effect<number>
  readonly getAverageQueryTime: () => Effect.Effect<number>
}

const ConnectionPool = Context.GenericTag<ConnectionPool>("ConnectionPool")

// Active connections polling
const activeConnectionsPolling = MetricPolling.make(
  Metric.gauge("db_active_connections", { description: "Active database connections" }),
  Effect.gen(function* () {
    const pool = yield* ConnectionPool
    return yield* pool.getActiveConnections()
  })
)

// Idle connections polling
const idleConnectionsPolling = MetricPolling.make(
  Metric.gauge("db_idle_connections", { description: "Idle database connections" }),
  Effect.gen(function* () {
    const pool = yield* ConnectionPool
    return yield* pool.getIdleConnections()
  })
)

// Connection pool utilization
const poolUtilizationPolling = MetricPolling.make(
  Metric.gauge("db_pool_utilization_percent", { description: "Connection pool utilization" }),
  Effect.gen(function* () {
    const pool = yield* ConnectionPool
    const active = yield* pool.getActiveConnections()
    const max = yield* pool.getMaxConnections()
    return Math.round((active / max) * 100)
  })
)

// Average query time polling
const queryTimePolling = MetricPolling.make(
  Metric.gauge("db_avg_query_time_ms", { description: "Average query execution time" }),
  Effect.gen(function* () {
    const pool = yield* ConnectionPool
    return yield* pool.getAverageQueryTime()
  })
)

// Combined database monitoring
const databaseMonitoring = MetricPolling.collectAll([
  activeConnectionsPolling,
  idleConnectionsPolling,
  poolUtilizationPolling,
  queryTimePolling
])

// Launch database monitoring with error handling
const launchDatabaseMonitoring = databaseMonitoring.pipe(
  MetricPolling.retry(
    Schedule.exponential(Duration.seconds(1)).pipe(
      Schedule.intersect(Schedule.recurs(5))
    )
  ),
  MetricPolling.launch(Schedule.spaced(Duration.seconds(15)))
).pipe(
  Effect.catchAll((error) => 
    Effect.logError("Database monitoring failed", error).pipe(
      Effect.zipRight(Effect.unit)
    )
  )
)
```

### Example 3: Business Metrics Monitoring

Track business-specific metrics like user activity and transaction volumes:

```typescript
import { MetricPolling, Metric, Schedule, Duration, Effect, Context } from "effect"

// Business services
interface UserService {
  readonly getActiveUserCount: () => Effect.Effect<number>
  readonly getRegisteredUserCount: () => Effect.Effect<number>
}

interface OrderService {
  readonly getPendingOrderCount: () => Effect.Effect<number>
  readonly getTodayOrderCount: () => Effect.Effect<number>
  readonly getTodayRevenue: () => Effect.Effect<number>
}

const UserService = Context.GenericTag<UserService>("UserService")
const OrderService = Context.GenericTag<OrderService>("OrderService")

// Active users polling
const activeUsersPolling = MetricPolling.make(
  Metric.gauge("active_users", { description: "Currently active users" }),
  Effect.gen(function* () {
    const userService = yield* UserService
    return yield* userService.getActiveUserCount()
  })
)

// Pending orders polling
const pendingOrdersPolling = MetricPolling.make(
  Metric.gauge("pending_orders", { description: "Orders awaiting processing" }),
  Effect.gen(function* () {
    const orderService = yield* OrderService
    return yield* orderService.getPendingOrderCount()
  })
)

// Daily revenue polling
const dailyRevenuePolling = MetricPolling.make(
  Metric.gauge("daily_revenue_cents", { description: "Today's revenue in cents" }),
  Effect.gen(function* () {
    const orderService = yield* OrderService
    const revenue = yield* orderService.getTodayRevenue()
    return Math.round(revenue * 100) // Convert to cents
  })
)

// Order processing rate
const orderRatePolling = MetricPolling.make(
  Metric.gauge("orders_per_hour", { description: "Orders processed per hour" }),
  Effect.gen(function* () {
    const orderService = yield* OrderService
    const todayOrders = yield* orderService.getTodayOrderCount()
    const hoursToday = new Date().getHours() + 1
    return Math.round(todayOrders / hoursToday)
  })
)

// Business intelligence dashboard metrics
const businessMetrics = Effect.gen(function* () {
  // Different polling intervals for different metrics
  const activeUsersFiber = yield* activeUsersPolling.pipe(
    MetricPolling.launch(Schedule.spaced(Duration.minutes(1)))
  )
  
  const pendingOrdersFiber = yield* pendingOrdersPolling.pipe(
    MetricPolling.launch(Schedule.spaced(Duration.seconds(30)))
  )
  
  const revenueFiber = yield* dailyRevenuePolling.pipe(
    MetricPolling.launch(Schedule.spaced(Duration.minutes(5)))
  )
  
  const orderRateFiber = yield* orderRatePolling.pipe(
    MetricPolling.launch(Schedule.spaced(Duration.minutes(10)))
  )
  
  return {
    fibers: [activeUsersFiber, pendingOrdersFiber, revenueFiber, orderRateFiber],
    stopAll: Effect.forEach(
      [activeUsersFiber, pendingOrdersFiber, revenueFiber, orderRateFiber],
      (fiber) => fiber.interrupt()
    )
  } as const
})
```

## Advanced Features Deep Dive

### Feature 1: Metric Aggregation with collectAll

Combine multiple related metrics into a single polling operation for efficiency:

#### Basic collectAll Usage

```typescript
import { MetricPolling, Metric, Effect } from "effect"

// Individual system metrics
const cpuPolling = MetricPolling.make(
  Metric.gauge("cpu_percent", { description: "CPU usage" }),
  Effect.sync(() => getCpuUsage())
)

const memoryPolling = MetricPolling.make(
  Metric.gauge("memory_mb", { description: "Memory usage" }),
  Effect.sync(() => getMemoryUsage())
)

const diskPolling = MetricPolling.make(
  Metric.gauge("disk_percent", { description: "Disk usage" }),
  Effect.sync(() => getDiskUsage())
)

// Collect all system metrics into one polling operation
const systemMetrics = MetricPolling.collectAll([
  cpuPolling,
  memoryPolling,
  diskPolling
])
```

#### Real-World collectAll Example

```typescript
// Database metrics collection
const databaseMetrics = Effect.gen(function* () {
  const database = yield* Database
  
  // Collect multiple database metrics in a single query
  const stats = yield* database.getConnectionStats()
  
  return {
    activeConnections: stats.active,
    idleConnections: stats.idle,
    queuedQueries: stats.queued,
    averageQueryTime: stats.avgQueryTime
  }
})

// Create individual polling metrics that share the same data source
const dbActivePolling = MetricPolling.make(
  Metric.gauge("db_active", { description: "Active DB connections" }),
  databaseMetrics.pipe(Effect.map(stats => stats.activeConnections))
)

const dbIdlePolling = MetricPolling.make(
  Metric.gauge("db_idle", { description: "Idle DB connections" }),
  databaseMetrics.pipe(Effect.map(stats => stats.idleConnections))
)

const dbQueuedPolling = MetricPolling.make(
  Metric.gauge("db_queued", { description: "Queued DB queries" }),
  databaseMetrics.pipe(Effect.map(stats => stats.queuedQueries))
)

// More efficient: collect all at once to avoid duplicate database calls
const allDbMetrics = MetricPolling.collectAll([
  dbActivePolling,
  dbIdlePolling,
  dbQueuedPolling
])
```

### Feature 2: Resilient Polling with Retry Policies

Add retry logic to handle transient polling failures:

#### Basic Retry Usage

```typescript
import { MetricPolling, Schedule, Duration } from "effect"

const resilientPolling = MetricPolling.make(
  Metric.gauge("external_api_response_time", { description: "API response time" }),
  Effect.gen(function* () {
    const response = yield* Effect.promise(() => fetch("https://api.example.com/health"))
    return response.ok ? 200 : 500
  })
).pipe(
  MetricPolling.retry(
    Schedule.exponential(Duration.millis(100)).pipe(
      Schedule.intersect(Schedule.recurs(3))
    )
  )
)
```

#### Advanced Retry: Different Strategies for Different Errors

```typescript
const advancedRetryPolling = MetricPolling.make(
  Metric.gauge("service_health_score", { description: "Service health score 0-100" }),
  Effect.gen(function* () {
    const healthCheck = yield* ServiceHealthCheck
    return yield* healthCheck.getHealthScore()
  })
).pipe(
  MetricPolling.retry(
    Schedule.exponential(Duration.seconds(1), 2.0).pipe(
      Schedule.intersect(Schedule.recurs(5)),
      Schedule.intersect(Schedule.spaced(Duration.seconds(10)))
    )
  )
)
```

### Feature 3: Metric Composition with zip

Combine related metrics that should be updated together:

#### Basic zip Usage

```typescript
const requestMetrics = MetricPolling.zip(
  MetricPolling.make(
    Metric.counter("requests_total", { description: "Total requests" }),
    Effect.gen(function* () {
      const stats = yield* RequestStats
      return yield* stats.getTotalRequests()
    })
  ),
  MetricPolling.make(
    Metric.gauge("requests_per_second", { description: "Current RPS" }),
    Effect.gen(function* () {
      const stats = yield* RequestStats
      return yield* stats.getCurrentRPS()
    })
  )
)
```

#### Real-World zip Example: Cache Metrics

```typescript
// Cache hit rate and total operations
const cacheHitPolling = MetricPolling.make(
  Metric.gauge("cache_hit_rate_percent", { description: "Cache hit rate" }),
  Effect.gen(function* () {
    const cache = yield* CacheService
    const stats = yield* cache.getStats()
    return Math.round((stats.hits / (stats.hits + stats.misses)) * 100)
  })
)

const cacheOpsPolling = MetricPolling.make(
  Metric.counter("cache_operations_total", { description: "Total cache operations" }),
  Effect.gen(function* () {
    const cache = yield* CacheService
    const stats = yield* cache.getStats()
    return stats.hits + stats.misses
  })
)

// Zip them together to ensure consistent timing
const cacheMetrics = MetricPolling.zip(cacheHitPolling, cacheOpsPolling)

// Launch combined metrics
const launchCacheMetrics = cacheMetrics.pipe(
  MetricPolling.launch(Schedule.spaced(Duration.seconds(30)))
)
```

## Practical Patterns & Best Practices

### Pattern 1: Hierarchical Metric Organization

```typescript
// Organize metrics by domain and create helper functions
const createDatabaseMetrics = (database: Database) => {
  const connectionMetrics = MetricPolling.collectAll([
    MetricPolling.make(
      Metric.gauge("db_connections_active", { description: "Active connections" }),
      database.getActiveConnections()
    ),
    MetricPolling.make(
      Metric.gauge("db_connections_idle", { description: "Idle connections" }),
      database.getIdleConnections()
    )
  ])
  
  const performanceMetrics = MetricPolling.collectAll([
    MetricPolling.make(
      Metric.gauge("db_query_time_avg_ms", { description: "Average query time" }),
      database.getAverageQueryTime()
    ),
    MetricPolling.make(
      Metric.gauge("db_slow_queries_count", { description: "Slow queries count" }),
      database.getSlowQueryCount()
    )
  ])
  
  return {
    connections: connectionMetrics,
    performance: performanceMetrics,
    all: MetricPolling.collectAll([connectionMetrics, performanceMetrics])
  }
}

// Usage
const launchDatabaseMonitoring = Effect.gen(function* () {
  const database = yield* Database
  const metrics = createDatabaseMetrics(database)
  
  return yield* metrics.all.pipe(
    MetricPolling.launch(Schedule.spaced(Duration.seconds(10)))
  )
})
```

### Pattern 2: Conditional Metric Polling

```typescript
// Poll different metrics based on application state
const createConditionalMetrics = (isProduction: boolean) => {
  const baseMetrics = [
    MetricPolling.make(
      Metric.gauge("app_uptime_seconds", { description: "Application uptime" }),
      Effect.sync(() => Math.floor(process.uptime()))
    )
  ]
  
  const productionMetrics = isProduction ? [
    MetricPolling.make(
      Metric.gauge("detailed_memory_stats", { description: "Detailed memory stats" }),
      Effect.sync(() => process.memoryUsage().external)
    ),
    MetricPolling.make(
      Metric.gauge("gc_stats", { description: "Garbage collection stats" }),
      Effect.sync(() => getGCStats())
    )
  ] : []
  
  return MetricPolling.collectAll([...baseMetrics, ...productionMetrics])
}

// Helper function for GC stats
const getGCStats = (): number => {
  // Simplified GC stats collection
  const stats = process.memoryUsage()
  return stats.heapUsed / stats.heapTotal
}
```

### Pattern 3: Metric Polling with Resource Management

```typescript
// Ensure proper resource cleanup when polling stops
const createResourceAwareMetrics = Effect.gen(function* () {
  const resourceMonitor = yield* Effect.acquireRelease(
    Effect.gen(function* () {
      const monitor = yield* Effect.sync(() => createResourceMonitor())
      yield* Effect.logInfo("Resource monitor started")
      return monitor
    }),
    (monitor) => Effect.gen(function* () {
      yield* Effect.sync(() => monitor.cleanup())
      yield* Effect.logInfo("Resource monitor stopped")
    })
  )
  
  const resourcePolling = MetricPolling.make(
    Metric.gauge("resource_usage", { description: "Resource usage percentage" }),
    Effect.sync(() => resourceMonitor.getUsagePercent())
  )
  
  return resourcePolling
})

// Usage with proper scoping
const launchResourceMonitoring = Effect.scoped(
  Effect.gen(function* () {
    const polling = yield* createResourceAwareMetrics
    const fiber = yield* polling.pipe(
      MetricPolling.launch(Schedule.spaced(Duration.seconds(5)))
    )
    
    // Monitor runs until scope is closed
    return fiber
  })
)

// Placeholder for resource monitor
const createResourceMonitor = () => ({
  getUsagePercent: () => Math.random() * 100,
  cleanup: () => { /* cleanup logic */ }
})
```

## Integration Examples

### Integration with Prometheus Metrics

```typescript
import { MetricPolling, Metric, Schedule, Duration, Effect, Layer } from "effect"

// Prometheus-compatible metric service
interface PrometheusMetrics {
  readonly register: (name: string, help: string, type: 'gauge' | 'counter') => Effect.Effect<void>
  readonly updateGauge: (name: string, value: number) => Effect.Effect<void>
  readonly incrementCounter: (name: string, value: number) => Effect.Effect<void>
  readonly getMetricsText: () => Effect.Effect<string>
}

const PrometheusMetrics = Context.GenericTag<PrometheusMetrics>("PrometheusMetrics")

// Create metrics that export to Prometheus
const createPrometheusPolling = <T>(
  name: string,
  description: string,
  pollEffect: Effect.Effect<T>,
  valueExtractor: (value: T) => number
) => {
  return Effect.gen(function* () {
    const prometheus = yield* PrometheusMetrics
    
    // Register the metric with Prometheus
    yield* prometheus.register(name, description, 'gauge')
    
    // Create Effect metric polling
    const polling = MetricPolling.make(
      Metric.gauge(name, { description }),
      pollEffect.pipe(
        Effect.tap(value => 
          prometheus.updateGauge(name, valueExtractor(value))
        ),
        Effect.map(valueExtractor)
      )
    )
    
    return polling
  })
}

// Usage example
const applicationMetrics = Effect.gen(function* () {
  const memoryPolling = yield* createPrometheusPolling(
    "app_memory_usage_bytes",
    "Application memory usage in bytes",
    Effect.sync(() => process.memoryUsage()),
    (usage) => usage.heapUsed
  )
  
  const requestPolling = yield* createPrometheusPolling(
    "http_requests_total",
    "Total HTTP requests",
    Effect.gen(function* () {
      const httpService = yield* HttpService
      return yield* httpService.getRequestCount()
    }),
    (count) => count
  )
  
  return MetricPolling.collectAll([memoryPolling, requestPolling])
})

// Launch with Prometheus export endpoint
const launchWithPrometheus = Effect.gen(function* () {
  const metrics = yield* applicationMetrics
  const fiber = yield* metrics.pipe(
    MetricPolling.launch(Schedule.spaced(Duration.seconds(15)))
  )
  
  // Start HTTP server for Prometheus scraping
  const server = yield* startPrometheusServer()
  
  return { metricsFiber: fiber, server }
})

// Simplified HTTP service interface
interface HttpService {
  readonly getRequestCount: () => Effect.Effect<number>
}

const HttpService = Context.GenericTag<HttpService>("HttpService")

// Simplified Prometheus server
const startPrometheusServer = () => 
  Effect.gen(function* () {
    const prometheus = yield* PrometheusMetrics
    // Start HTTP server that serves prometheus.getMetricsText() on /metrics
    return yield* Effect.sync(() => ({ port: 9090 }))
  })
```

### Testing Strategies

```typescript
import { MetricPolling, Metric, Effect, TestContext, TestClock } from "effect"

// Create testable metric polling
const createTestablePolling = (mockValue: number) => 
  MetricPolling.make(
    Metric.gauge("test_metric", { description: "Test metric" }),
    Effect.succeed(mockValue)
  )

// Test metric polling behavior
const testMetricPolling = Effect.gen(function* () {
  const polling = createTestablePolling(42)
  
  // Test single poll
  const value = yield* MetricPolling.poll(polling)
  console.log("Polled value:", value) // Should be 42
  
  // Test poll and update
  yield* MetricPolling.pollAndUpdate(polling)
  
  // Test with mock time
  const testProgram = Effect.gen(function* () {
    const fiber = yield* polling.pipe(
      MetricPolling.launch(Schedule.spaced(Duration.seconds(1)))
    )
    
    // Advance test clock
    yield* TestClock.adjust(Duration.seconds(5))
    
    // Verify polling occurred
    const metricValue = yield* Metric.value(polling.metric)
    console.log("Metric value after 5 seconds:", metricValue)
    
    yield* fiber.interrupt()
  })
  
  return yield* testProgram.pipe(
    Effect.provide(TestContext.TestContext)
  )
})

// Test error handling
const testPollingWithErrors = Effect.gen(function* () {
  let callCount = 0
  
  const flakyPolling = MetricPolling.make(
    Metric.gauge("flaky_metric", { description: "Flaky test metric" }),
    Effect.gen(function* () {
      callCount++
      if (callCount < 3) {
        return yield* Effect.fail(new Error("Temporary failure"))
      }
      return 100
    })
  ).pipe(
    MetricPolling.retry(
      Schedule.exponential(Duration.millis(10)).pipe(
        Schedule.intersect(Schedule.recurs(5))
      )
    )
  )
  
  // Should eventually succeed after retries
  const value = yield* MetricPolling.poll(flakyPolling)
  console.log("Final value after retries:", value) // Should be 100
})

// Property-based testing helper
const testPollingProperties = (
  polling: MetricPolling.MetricPolling<any, any, any, any, any>
) => 
  Effect.gen(function* () {
    // Test that polling is idempotent
    const value1 = yield* MetricPolling.poll(polling)
    const value2 = yield* MetricPolling.poll(polling)
    
    // For deterministic pollers, values should be the same
    if (value1 !== value2) {
      yield* Effect.logWarning("Polling is not deterministic")
    }
    
    // Test that pollAndUpdate doesn't throw
    yield* MetricPolling.pollAndUpdate(polling)
    
    return true
  })
```

## Conclusion

MetricPolling provides **automated metric collection**, **composable monitoring**, and **resource-efficient polling** for Effect applications.

Key benefits:
- **Declarative Monitoring**: Define what to measure, not how to measure it
- **Automatic Scheduling**: Built-in scheduling eliminates manual polling coordination
- **Composable Architecture**: Combine metrics efficiently with collectAll and zip
- **Resilient Operations**: Built-in retry policies handle transient failures
- **Resource Efficiency**: Shared polling effects and proper resource management

MetricPolling is ideal for applications that need continuous monitoring of system resources, business metrics, or external service health without the complexity of manual metric collection scattered throughout the codebase.