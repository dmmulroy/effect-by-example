# Metric: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem Metric Solves

Observability is critical for production applications, but traditional approaches to metrics collection suffer from significant issues:

```typescript
// Traditional approach - scattered, inconsistent metrics
let requestCount = 0;
let totalResponseTime = 0;
const errorsByType = new Map<string, number>();

function handleRequest(req: Request) {
  const start = Date.now();
  requestCount++; // Race condition with concurrent requests
  
  try {
    const result = processRequest(req);
    totalResponseTime += Date.now() - start; // Memory leak potential
    return result;
  } catch (error) {
    const errorType = error.constructor.name;
    errorsByType.set(errorType, (errorsByType.get(errorType) || 0) + 1);
    throw error;
  }
}

// Separate logging/reporting logic scattered throughout
console.log(`Requests: ${requestCount}, Avg Response: ${totalResponseTime / requestCount}ms`);
```

This approach leads to:
- **Race Conditions** - Concurrent access to shared counters
- **Memory Leaks** - Unbounded metric storage and processing
- **Inconsistent Collection** - Metrics scattered across the codebase
- **Poor Composability** - Different metric types require different handling
- **No Type Safety** - Runtime errors from incorrect metric updates

### The Metric Solution

Effect's Metric module provides thread-safe, composable, and type-safe metrics that integrate seamlessly with your application logic:

```typescript
import { Metric, Effect } from "effect"

// Type-safe metric definitions
const requestCounter = Metric.counter("http_requests_total", {
  description: "Total number of HTTP requests"
})

const responseTimer = Metric.timer("http_request_duration_milliseconds", 
  "HTTP request duration in milliseconds"
)

const errorFrequency = Metric.frequency("http_errors", {
  description: "Frequency of HTTP errors by type"
})

// Composable, thread-safe application logic
export const handleRequest = (req: Request) => Effect.gen(function* () {
  const response = yield* processRequest(req).pipe(
    Metric.trackDuration(responseTimer),
    Metric.trackAll(requestCounter, 1),
    Effect.catchTag("ValidationError", (error) => 
      Metric.update(errorFrequency, "validation").pipe(
        Effect.zipRight(Effect.fail(error))
      )
    ),
    Effect.catchTag("DatabaseError", (error) =>
      Metric.update(errorFrequency, "database").pipe(
        Effect.zipRight(Effect.fail(error))
      )
    )
  )
  return response
})
```

### Key Concepts

**Counter**: Monotonically increasing values for counting events (requests, errors, tasks completed)

**Gauge**: Point-in-time measurements that can go up or down (memory usage, active connections, queue size)

**Histogram**: Distribution of values within configured buckets (response times, request sizes, processing durations)

**Timer**: Special histogram optimized for duration measurements with automatic time unit handling

**Summary**: Statistical summaries with quantiles over sliding time windows (95th percentile response times)

**Frequency**: Counts occurrences of different string values (error types, user agents, endpoints)

## Basic Usage Patterns

### Pattern 1: Counter Setup

```typescript
import { Metric, Effect } from "effect"

// Basic counter for tracking events
const pageViewCounter = Metric.counter("page_views_total", {
  description: "Total number of page views"
})

// Increment counter in your application logic
const trackPageView = (page: string) => Effect.gen(function* () {
  yield* Metric.increment(pageViewCounter)
  // Your business logic here
  return `Viewed ${page}`
})
```

### Pattern 2: Gauge for Current State

```typescript
// Gauge for tracking current values
const activeConnectionsGauge = Metric.gauge("active_connections", {
  description: "Number of active database connections"
})

const connectionManager = Effect.gen(function* () {
  // Update gauge when connections change
  yield* Metric.set(activeConnectionsGauge, 10)
  
  // Increment when new connection opens
  yield* Metric.increment(activeConnectionsGauge)
  
  // Decrement when connection closes  
  yield* Metric.incrementBy(activeConnectionsGauge, -1)
})
```

### Pattern 3: Timer for Duration Tracking

```typescript
// Timer automatically tracks Duration values
const databaseQueryTimer = Metric.timer("db_query_duration_milliseconds",
  "Time spent executing database queries"
)

const executeQuery = (sql: string) => Effect.gen(function* () {
  const result = yield* runDatabaseQuery(sql)
  return result
}).pipe(
  Metric.trackDuration(databaseQueryTimer)
)
```

## Real-World Examples

### Example 1: E-commerce API Metrics

Complete metrics setup for an e-commerce API with request tracking, error monitoring, and performance measurement:

```typescript
import { Metric, Effect, MetricBoundaries } from "effect"

// Request metrics
const httpRequestsTotal = Metric.counter("http_requests_total", {
  description: "Total HTTP requests by method and status"
})

const httpRequestDuration = Metric.histogram(
  "http_request_duration_seconds",
  MetricBoundaries.exponential({ 
    start: 0.001, // 1ms
    factor: 2, 
    count: 12 // Up to ~4 seconds
  }),
  "HTTP request duration in seconds"
)

// Business metrics
const ordersPlaced = Metric.counter("orders_placed_total", {
  description: "Total orders placed"
})

const orderValue = Metric.histogram(
  "order_value_dollars",
  MetricBoundaries.linear({
    start: 0,
    width: 50, // $50 buckets
    count: 20 // Up to $1000
  }),
  "Order value in dollars"
)

const inventoryLevels = Metric.gauge("inventory_items", {
  description: "Current inventory levels by product"
})

// Error tracking
const errorsByType = Metric.frequency("errors_by_type", {
  description: "Errors categorized by type"
})

// API handler with comprehensive metrics
export const createOrderHandler = (orderData: OrderData) => Effect.gen(function* () {
  // Track the request
  yield* Metric.increment(httpRequestsTotal.pipe(
    Metric.tagged("method", "POST"),
    Metric.tagged("endpoint", "/orders")
  ))
  
  const startTime = yield* Effect.clock.pipe(Effect.map(clock => clock.currentTimeMillis))
  
  // Validate order
  const validOrder = yield* validateOrder(orderData).pipe(
    Effect.catchTag("ValidationError", (error) =>
      Metric.update(errorsByType, "validation_error").pipe(
        Effect.zipRight(Effect.fail(error))
      )
    )
  )
  
  // Check inventory
  const inventoryCheck = yield* checkInventory(validOrder).pipe(
    Effect.catchTag("OutOfStockError", (error) =>
      Metric.update(errorsByType, "out_of_stock").pipe(
        Effect.zipRight(Effect.fail(error))
      )
    )
  )
  
  // Process payment
  const payment = yield* processPayment(validOrder).pipe(
    Effect.catchTag("PaymentError", (error) =>
      Metric.update(errorsByType, "payment_failed").pipe(
        Effect.zipRight(Effect.fail(error))
      )
    )
  )
  
  // Create order
  const order = yield* createOrder(validOrder, payment)
  
  // Update business metrics
  yield* Metric.increment(ordersPlaced)
  yield* Metric.update(orderValue, order.totalAmount)
  
  // Update inventory levels
  for (const item of order.items) {
    yield* Metric.incrementBy(inventoryLevels.pipe(
      Metric.tagged("product_id", item.productId)
    ), -item.quantity)
  }
  
  // Record successful request
  const endTime = yield* Effect.clock.pipe(Effect.map(clock => clock.currentTimeMillis))
  const duration = (endTime - startTime) / 1000 // Convert to seconds
  
  yield* Metric.update(httpRequestDuration.pipe(
    Metric.tagged("method", "POST"),
    Metric.tagged("endpoint", "/orders"),
    Metric.tagged("status", "200")
  ), duration)
  
  return order
}).pipe(
  Effect.catchAll((error) => 
    Effect.gen(function* () {
      // Record failed request metrics
      const endTime = yield* Effect.clock.pipe(Effect.map(clock => clock.currentTimeMillis))
      const duration = (endTime - startTime) / 1000
      
      yield* Metric.update(httpRequestDuration.pipe(
        Metric.tagged("method", "POST"),
        Metric.tagged("endpoint", "/orders"),
        Metric.tagged("status", "500")
      ), duration)
      
      return yield* Effect.fail(error)
    })
  )
)
```

### Example 2: Background Job Processing Metrics

Comprehensive metrics for a background job processing system:

```typescript
import { Metric, Effect, Schedule, Queue } from "effect"

// Job processing metrics
const jobsProcessed = Metric.counter("jobs_processed_total", {
  description: "Total jobs processed by type and status"
})

const jobProcessingDuration = Metric.timer("job_processing_duration_milliseconds",
  "Time spent processing jobs"
)

const jobQueueSize = Metric.gauge("job_queue_size", {
  description: "Current number of jobs in queue"
})

const jobFailureRate = Metric.summary({
  name: "job_failure_rate",
  maxAge: "5 minutes",
  maxSize: 1000,
  error: 0.01,
  quantiles: [0.5, 0.95, 0.99],
  description: "Job failure rate percentiles"
})

// Worker resource metrics
const workerMemoryUsage = Metric.gauge("worker_memory_bytes", {
  description: "Memory usage by worker process"
})

const activeWorkers = Metric.gauge("active_workers", {
  description: "Number of active worker processes"
})

interface Job {
  readonly id: string
  readonly type: string
  readonly payload: unknown
  readonly retryCount: number
  readonly createdAt: Date
}

// Job processor with metrics
export const processJob = (job: Job) => Effect.gen(function* () {
  // Update queue size (decrement as we take job)
  yield* Metric.incrementBy(jobQueueSize.pipe(
    Metric.tagged("queue", job.type)
  ), -1)
  
  // Increment active workers
  yield* Metric.increment(activeWorkers.pipe(
    Metric.tagged("worker_id", yield* Effect.fiberDescriptor.pipe(
      Effect.map(descriptor => descriptor.id.toString())
    ))
  ))
  
  const jobResult = yield* Effect.suspend(() => {
    // Simulate job processing based on type
    switch (job.type) {
      case "email":
        return sendEmail(job.payload as EmailPayload)
      case "report":
        return generateReport(job.payload as ReportPayload)
      case "cleanup":
        return cleanupData(job.payload as CleanupPayload)
      default:
        return Effect.fail(new Error(`Unknown job type: ${job.type}`))
    }
  }).pipe(
    // Track processing duration
    Metric.trackDuration(jobProcessingDuration.pipe(
      Metric.tagged("job_type", job.type)
    )),
    // Handle success
    Effect.tap(() => 
      Metric.increment(jobsProcessed.pipe(
        Metric.tagged("job_type", job.type),
        Metric.tagged("status", "success")
      ))
    ),
    // Handle failures
    Effect.catchAll((error) => Effect.gen(function* () {
      // Record failure metrics
      yield* Metric.increment(jobsProcessed.pipe(
        Metric.tagged("job_type", job.type),
        Metric.tagged("status", "failed")
      ))
      
      // Update failure rate (1 = failure, 0 = success)
      yield* Metric.update(jobFailureRate.pipe(
        Metric.tagged("job_type", job.type)
      ), 1)
      
      return yield* Effect.fail(error)
    })),
    // Always decrement active workers
    Effect.ensuring(
      Metric.incrementBy(activeWorkers.pipe(
        Metric.tagged("worker_id", yield* Effect.fiberDescriptor.pipe(
          Effect.map(descriptor => descriptor.id.toString())
        ))
      ), -1)
    )
  )
  
  // Record successful completion
  yield* Metric.update(jobFailureRate.pipe(
    Metric.tagged("job_type", job.type)
  ), 0)
  
  return jobResult
})

// Job queue worker with resource monitoring
export const startJobWorker = (jobQueue: Queue.Queue<Job>) => Effect.gen(function* () {
  // Monitor memory usage periodically
  const memoryMonitor = Effect.gen(function* () {
    const memoryUsage = process.memoryUsage()
    yield* Metric.set(workerMemoryUsage, memoryUsage.heapUsed)
  }).pipe(
    Effect.repeat(Schedule.fixed("30 seconds")),
    Effect.fork
  )
  
  yield* memoryMonitor
  
  // Process jobs continuously
  yield* Effect.gen(function* () {
    const job = yield* Queue.take(jobQueue)
    yield* processJob(job)
  }).pipe(
    Effect.forever,
    Effect.catchAllCause(Effect.logError)
  )
})
```

### Example 3: Database Connection Pool Metrics

Detailed monitoring for a database connection pool:

```typescript
import { Metric, Effect, Pool, Duration } from "effect"

// Connection pool metrics
const poolSize = Metric.gauge("db_pool_size", {
  description: "Current database connection pool size"
})

const poolAcquisitionDuration = Metric.timer("db_pool_acquisition_duration_milliseconds",
  "Time to acquire connection from pool"
)

const poolUtilization = Metric.gauge("db_pool_utilization_percent", {
  description: "Database pool utilization as percentage"
})

const connectionLifetime = Metric.histogram(
  "db_connection_lifetime_seconds",
  MetricBoundaries.exponential({
    start: 1, // 1 second
    factor: 2,
    count: 10 // Up to ~17 minutes
  }),
  "Database connection lifetime"
)

const queryExecutionTime = Metric.timer("db_query_execution_time_milliseconds",
  "Database query execution time"
)

const queryErrorsByType = Metric.frequency("db_query_errors", {
  description: "Database query errors by type",
  preregisteredWords: ["timeout", "connection_lost", "syntax_error", "constraint_violation"]
})

interface DatabaseConnection {
  readonly id: string
  readonly createdAt: Date
  execute<T>(query: string, params?: unknown[]): Effect.Effect<T, DatabaseError>
  close(): Effect.Effect<void>
}

interface DatabaseError {
  readonly _tag: string
  readonly message: string
  readonly code?: string
}

// Create connection pool with metrics
export const createDatabasePool = (config: {
  minSize: number
  maxSize: number
  connectionTimeout: Duration.Duration
}) => Effect.gen(function* () {
  const pool = yield* Pool.make({
    acquire: Effect.gen(function* () {
      const connection = yield* createDatabaseConnection().pipe(
        Metric.trackDuration(poolAcquisitionDuration),
        Effect.tap(() => 
          Metric.increment(poolSize.pipe(
            Metric.tagged("state", "active")
          ))
        )
      )
      return connection
    }),
    size: config.maxSize
  })
  
  // Monitor pool utilization
  const utilizationMonitor = Effect.gen(function* () {
    const activeConnections = yield* Pool.size(pool)
    const utilization = (activeConnections / config.maxSize) * 100
    yield* Metric.set(poolUtilization, utilization)
  }).pipe(
    Effect.repeat(Schedule.fixed("10 seconds")),
    Effect.fork
  )
  
  yield* utilizationMonitor
  
  return pool
})

// Execute query with comprehensive metrics
export const executeQuery = <T>(
  pool: Pool.Pool<DatabaseConnection, never>,
  query: string,
  params?: unknown[]
) => Effect.gen(function* () {
  const startTime = yield* Effect.clock.pipe(Effect.map(clock => clock.currentTimeMillis))
  
  const result = yield* Pool.get(pool).pipe(
    Effect.flatMap(connection => 
      connection.execute<T>(query, params).pipe(
        Metric.trackDuration(queryExecutionTime.pipe(
          Metric.tagged("query_type", extractQueryType(query))
        )),
        Effect.catchTag("TimeoutError", (error) =>
          Metric.update(queryErrorsByType, "timeout").pipe(
            Effect.zipRight(Effect.fail(error))
          )
        ),
        Effect.catchTag("ConnectionError", (error) =>
          Metric.update(queryErrorsByType, "connection_lost").pipe(
            Effect.zipRight(Effect.fail(error))
          )
        ),
        Effect.catchTag("SyntaxError", (error) =>
          Metric.update(queryErrorsByType, "syntax_error").pipe(
            Effect.zipRight(Effect.fail(error))
          )
        ),
        Effect.catchTag("ConstraintError", (error) =>
          Metric.update(queryErrorsByType, "constraint_violation").pipe(
            Effect.zipRight(Effect.fail(error))
          )
        ),
        Effect.ensuring(() => {
          // Track connection lifetime when returned to pool
          const endTime = Date.now()
          const lifetimeSeconds = (endTime - connection.createdAt.getTime()) / 1000
          return Metric.update(connectionLifetime, lifetimeSeconds)
        })
      )
    )
  )
  
  return result
})

// Helper to extract query type for metrics
const extractQueryType = (query: string): string => {
  const trimmed = query.trim().toUpperCase()
  if (trimmed.startsWith('SELECT')) return 'select'
  if (trimmed.startsWith('INSERT')) return 'insert'
  if (trimmed.startsWith('UPDATE')) return 'update'
  if (trimmed.startsWith('DELETE')) return 'delete'
  return 'other'
}
```

## Advanced Features Deep Dive

### Feature 1: Metric Composition and Transformation

Combine multiple metrics and transform their inputs/outputs for complex monitoring scenarios.

#### Basic Metric Composition

```typescript
import { Metric } from "effect"

// Combine metrics using zip
const requestMetrics = Metric.zip(
  Metric.counter("requests_total"),
  Metric.timer("request_duration")
)

// Use the combined metric
const trackRequest = <A, E, R>(effect: Effect.Effect<A, E, R>) =>
  effect.pipe(requestMetrics([1, Duration.millis(0)]))
```

#### Real-World Composition Example

```typescript
// Complex e-commerce checkout metrics
const checkoutMetrics = Effect.gen(function* () {
  const stepCounter = Metric.counter("checkout_steps_completed")
  const valueHistogram = Metric.histogram("checkout_values", 
    MetricBoundaries.exponential({ start: 1, factor: 2, count: 15 })
  )
  const errorFreq = Metric.frequency("checkout_errors")
  
  // Compose into a single metric for checkout tracking
  const composedMetric = Metric.zip(
    Metric.zip(stepCounter, valueHistogram),
    errorFreq
  )
  
  return composedMetric
})

// Transform metric inputs for specific use cases
const userActivityMetric = Metric.counter("user_activity").pipe(
  Metric.mapInput((activity: UserActivity) => {
    // Transform complex user activity into simple count
    switch (activity.type) {
      case "login": return 2
      case "purchase": return 5  
      case "view": return 1
      default: return 0
    }
  }),
  Metric.tagged("activity_weight", "calculated")
)
```

#### Advanced Metric: Custom Business KPI Tracking

```typescript
// Custom metric for tracking business KPIs with weighted scoring
interface BusinessEvent {
  readonly type: "signup" | "purchase" | "retention" | "referral"
  readonly value: number
  readonly userId: string
  readonly timestamp: Date
}

const businessValueMetric = Metric.counter("business_value_score").pipe(
  Metric.mapInput((event: BusinessEvent) => {
    // Weight different events by business importance
    const weights = {
      signup: 10,
      purchase: 50,
      retention: 25,
      referral: 15
    }
    return event.value * weights[event.type]
  }),
  Metric.tagged("metric_type", "business_kpi")
)

const trackBusinessEvent = (event: BusinessEvent) => Effect.gen(function* () {
  // Update primary business metric
  yield* Metric.update(businessValueMetric.pipe(
    Metric.tagged("event_type", event.type),
    Metric.tagged("user_segment", yield* getUserSegment(event.userId))
  ), event)
  
  // Also track in time-series for trending
  yield* Metric.update(businessValueMetric.pipe(
    Metric.tagged("period", getTimePeriod(event.timestamp)),
    Metric.withNow
  ), event)
})
```

### Feature 2: Dynamic Tagging and Contextual Metrics

Create metrics that adapt their tags based on runtime context and input data.

#### Dynamic Tag Generation

```typescript
// Metric that tags based on request context
const apiMetrics = Metric.counter("api_requests").pipe(
  Metric.taggedWithLabelsInput((request: ApiRequest) => [
    MetricLabel.make("method", request.method),
    MetricLabel.make("endpoint", normalizeEndpoint(request.path)),
    MetricLabel.make("user_tier", request.user?.tier ?? "anonymous"),
    MetricLabel.make("region", request.headers["cf-ipcountry"] ?? "unknown")
  ])
)

// Contextual error tracking
const contextualErrorMetric = Metric.frequency("contextual_errors").pipe(
  Metric.taggedWithLabelsInput((error: AppError) => [
    MetricLabel.make("error_type", error.name),
    MetricLabel.make("severity", error.severity),
    MetricLabel.make("component", error.component),
    MetricLabel.make("user_session", error.context.sessionId),
    MetricLabel.make("feature_flag", error.context.experimentVariant ?? "control")
  ])
)
```

#### Real-World Dynamic Tagging Example

```typescript
// A/B testing metrics that automatically tag by experiment
interface ExperimentContext {
  readonly experimentId: string
  readonly variant: string
  readonly userId: string
  readonly sessionId: string
}

const experimentMetrics = {
  conversionRate: Metric.counter("experiment_conversions").pipe(
    Metric.taggedWithLabelsInput((context: ExperimentContext) => [
      MetricLabel.make("experiment_id", context.experimentId),
      MetricLabel.make("variant", context.variant),
      MetricLabel.make("user_cohort", getUserCohort(context.userId))
    ])
  ),
  
  engagementTime: Metric.timer("experiment_engagement_time").pipe(
    Metric.taggedWithLabelsInput((context: ExperimentContext) => [
      MetricLabel.make("experiment_id", context.experimentId),
      MetricLabel.make("variant", context.variant),
      MetricLabel.make("session_type", getSessionType(context.sessionId))
    ])
  )
}

// Track experiment events with automatic tagging
export const trackExperimentEvent = (
  event: "view" | "click" | "conversion",
  context: ExperimentContext
) => Effect.gen(function* () {
  switch (event) {
    case "conversion":
      yield* Metric.increment(experimentMetrics.conversionRate.pipe(
        Metric.tagged("event_type", "conversion")
      ))(context)
      break
    case "click":
      yield* Metric.increment(experimentMetrics.conversionRate.pipe(
        Metric.tagged("event_type", "engagement") 
      ))(context)
      break
  }
})
```

### Feature 3: Polling Metrics and Live Monitoring

Set up metrics that automatically poll system resources and external services.

#### System Resource Polling

```typescript
import { MetricPolling, Effect, Schedule } from "effect"

// Automatically poll system metrics
const systemMetrics = Effect.gen(function* () {
  const cpuUsage = Metric.gauge("system_cpu_percent")
  const memoryUsage = Metric.gauge("system_memory_bytes")  
  const diskSpace = Metric.gauge("system_disk_free_bytes")
  
  // Create polling metrics that update automatically
  const cpuPolling = MetricPolling.make(
    cpuUsage,
    Effect.suspend(() => Effect.succeed(getCpuUsage()))
  )
  
  const memoryPolling = MetricPolling.make(
    memoryUsage,
    Effect.suspend(() => Effect.succeed(process.memoryUsage().heapUsed))
  )
  
  const diskPolling = MetricPolling.make(
    diskSpace,
    Effect.suspend(() => getDiskFreeSpace("/"))
  )
  
  // Start all polling with different schedules
  yield* Effect.all([
    MetricPolling.launch(cpuPolling, Schedule.fixed("5 seconds")),
    MetricPolling.launch(memoryPolling, Schedule.fixed("10 seconds")),
    MetricPolling.launch(diskPolling, Schedule.fixed("60 seconds"))
  ], { concurrency: "unbounded" })
})
```

#### External Service Health Polling

```typescript
// Monitor external service health automatically
const serviceHealthMetrics = Effect.gen(function* () {
  const serviceLatency = Metric.gauge("external_service_latency_ms")
  const serviceAvailability = Metric.gauge("external_service_availability")
  
  // Health check that measures latency and availability
  const healthCheck = (serviceName: string, endpoint: string) => Effect.gen(function* () {
    const start = yield* Effect.clock.pipe(Effect.map(clock => clock.currentTimeMillis))
    
    const result = yield* Effect.tryPromise({
      try: () => fetch(endpoint, { method: "HEAD", timeout: 5000 }),
      catch: () => new Error("Health check failed")
    }).pipe(
      Effect.map(() => ({ available: true, latency: Date.now() - start })),
      Effect.catchAll(() => Effect.succeed({ available: false, latency: 5000 }))
    )
    
    // Update metrics with service name tags
    yield* Metric.set(serviceLatency.pipe(
      Metric.tagged("service", serviceName)
    ), result.latency)
    
    yield* Metric.set(serviceAvailability.pipe(
      Metric.tagged("service", serviceName)
    ), result.available ? 1 : 0)
    
    return result
  })
  
  // Poll multiple services
  const services = [
    { name: "payment-api", endpoint: "https://api.payment.com/health" },
    { name: "user-service", endpoint: "https://users.internal.com/health" },
    { name: "inventory-db", endpoint: "https://inventory.db.internal.com/ping" }
  ]
  
  // Start health check polling for all services
  yield* Effect.all(
    services.map(service => 
      healthCheck(service.name, service.endpoint).pipe(
        Effect.repeat(Schedule.fixed("30 seconds")),
        Effect.fork
      )
    ),
    { concurrency: "unbounded" }
  )
})
```

## Practical Patterns & Best Practices

### Pattern 1: Metric Factory for Consistent Naming

Create reusable factories that ensure consistent metric naming and tagging across your application:

```typescript
// Centralized metric factory with naming conventions
export const createMetrics = (serviceName: string, version: string) => {
  const baseLabels = [
    MetricLabel.make("service", serviceName),
    MetricLabel.make("version", version)
  ]
  
  const withBaseLabels = <Type, In, Out>(metric: Metric<Type, In, Out>) =>
    metric.pipe(Metric.taggedWithLabels(baseLabels))
  
  return {
    // HTTP metrics
    httpRequests: withBaseLabels(Metric.counter("http_requests_total", {
      description: "Total HTTP requests by endpoint and status"
    })),
    
    httpDuration: withBaseLabels(Metric.histogram(
      "http_request_duration_seconds",
      MetricBoundaries.exponential({ start: 0.001, factor: 2, count: 15 }),
      "HTTP request duration in seconds"
    )),
    
    // Business metrics  
    businessEvents: withBaseLabels(Metric.counter("business_events_total", {
      description: "Business events by type"
    })),
    
    // Error metrics
    errors: withBaseLabels(Metric.frequency("errors_by_type", {
      description: "Errors categorized by type and component"
    })),
    
    // Resource metrics
    memoryUsage: withBaseLabels(Metric.gauge("memory_usage_bytes", {
      description: "Current memory usage in bytes"
    })),
    
    activeConnections: withBaseLabels(Metric.gauge("active_connections", {
      description: "Number of active connections"
    }))
  }
}

// Usage across different services
const userServiceMetrics = createMetrics("user-service", "1.2.3")
const orderServiceMetrics = createMetrics("order-service", "2.1.0")
```

### Pattern 2: Middleware for Automatic Metric Collection

Create middleware that automatically collects metrics without cluttering business logic:

```typescript
// HTTP middleware for automatic request metrics
export const createMetricsMiddleware = (metrics: ReturnType<typeof createMetrics>) => {
  return <A, E, R>(effect: Effect.Effect<A, E, R>) => Effect.gen(function* () {
    const request = yield* HttpRequest.current
    const startTime = yield* Effect.clock.pipe(Effect.map(clock => clock.currentTimeMillis))
    
    const result = yield* effect.pipe(
      Effect.catchAll((error) => Effect.gen(function* () {
        // Record error metrics
        yield* Metric.update(metrics.errors, error.name).pipe(
          Metric.tagged("endpoint", request.url.pathname),
          Metric.tagged("method", request.method)
        )
        
        // Record failed request metrics
        const endTime = yield* Effect.clock.pipe(Effect.map(clock => clock.currentTimeMillis))
        const durationSeconds = (endTime - startTime) / 1000
        
        yield* Metric.update(metrics.httpDuration.pipe(
          Metric.tagged("method", request.method),
          Metric.tagged("endpoint", request.url.pathname),
          Metric.tagged("status", "500")
        ), durationSeconds)
        
        yield* Metric.increment(metrics.httpRequests.pipe(
          Metric.tagged("method", request.method),
          Metric.tagged("endpoint", request.url.pathname),
          Metric.tagged("status", "500")
        ))
        
        return yield* Effect.fail(error)
      }))
    )
    
    // Record successful request metrics
    const endTime = yield* Effect.clock.pipe(Effect.map(clock => clock.currentTimeMillis))
    const durationSeconds = (endTime - startTime) / 1000
    
    yield* Metric.update(metrics.httpDuration.pipe(
      Metric.tagged("method", request.method),
      Metric.tagged("endpoint", request.url.pathname),  
      Metric.tagged("status", "200")
    ), durationSeconds)
    
    yield* Metric.increment(metrics.httpRequests.pipe(
      Metric.tagged("method", request.method),
      Metric.tagged("endpoint", request.url.pathname),
      Metric.tagged("status", "200")
    ))
    
    return result
  })
}

// Business logic remains clean
export const getUserProfile = (userId: string) => Effect.gen(function* () {
  const user = yield* Database.findUser(userId)
  const profile = yield* buildUserProfile(user)
  return profile
}).pipe(createMetricsMiddleware(userServiceMetrics))
```

### Pattern 3: Custom Metric Aggregation Helpers

Build helpers that encapsulate common metric aggregation patterns:

```typescript
// Helper for tracking operation success/failure rates
export const trackOperationMetrics = <A, E>(
  operation: string,
  metrics: ReturnType<typeof createMetrics>
) => {
  return <R>(effect: Effect.Effect<A, E, R>) => Effect.gen(function* () {
    const successCounter = metrics.businessEvents.pipe(
      Metric.tagged("operation", operation),
      Metric.tagged("status", "success")
    )
    
    const failureCounter = metrics.businessEvents.pipe(
      Metric.tagged("operation", operation),
      Metric.tagged("status", "failure")
    )
    
    const result = yield* effect.pipe(
      Metric.trackDuration(metrics.httpDuration.pipe(
        Metric.tagged("operation", operation)
      )),
      Effect.tap(() => Metric.increment(successCounter)),
      Effect.catchAll((error) => Effect.gen(function* () {
        yield* Metric.increment(failureCounter)
        yield* Metric.update(metrics.errors, error.name || "unknown_error").pipe(
          Metric.tagged("operation", operation)
        )
        return yield* Effect.fail(error)
      }))
    )
    
    return result
  })
}

// Helper for batch operation metrics
export const trackBatchMetrics = <A, E>(
  batchName: string,
  items: ReadonlyArray<A>,
  processor: (item: A) => Effect.Effect<unknown, E>,
  metrics: ReturnType<typeof createMetrics>
) => Effect.gen(function* () {
  const batchSize = items.length
  const startTime = yield* Effect.clock.pipe(Effect.map(clock => clock.currentTimeMillis))
  
  // Track batch size
  yield* Metric.update(metrics.businessEvents.pipe(
    Metric.tagged("batch_name", batchName),
    Metric.tagged("event_type", "batch_started")
  ), batchSize)
  
  // Process items and collect results
  const results = yield* Effect.allSuccesses(
    items.map(item => processor(item).pipe(
      Effect.either,
      Effect.tap(result => {
        const eventType = result._tag === "Right" ? "item_success" : "item_failure"
        return Metric.increment(metrics.businessEvents.pipe(
          Metric.tagged("batch_name", batchName),
          Metric.tagged("event_type", eventType)
        ))
      })
    ))
  )
  
  const successCount = results.filter(r => r._tag === "Right").length
  const failureCount = results.filter(r => r._tag === "Left").length
  
  // Record final batch metrics
  const endTime = yield* Effect.clock.pipe(Effect.map(clock => clock.currentTimeMillis))
  const durationMs = endTime - startTime
  
  yield* Effect.all([
    Metric.update(metrics.httpDuration.pipe(
      Metric.tagged("batch_name", batchName),
      Metric.tagged("operation", "batch_processing")
    ), durationMs / 1000),
    
    Metric.set(metrics.activeConnections.pipe(
      Metric.tagged("batch_name", batchName),
      Metric.tagged("metric_type", "success_rate")
    ), Math.round((successCount / batchSize) * 100))
  ])
  
  return { successCount, failureCount, totalTime: durationMs }
})

// Usage examples
const processUserRegistrations = (users: User[]) =>
  trackBatchMetrics("user_registration", users, processRegistration, userServiceMetrics)

const sendNotifications = (notifications: Notification[]) =>
  trackOperationMetrics("send_notification", userServiceMetrics)(
    Effect.forEach(notifications, sendSingleNotification, { concurrency: 5 })
  )
```

## Integration Examples

### Integration with Prometheus and Grafana

Set up Effect Metrics with Prometheus for visualization in Grafana:

```typescript
import * as NodeSdk from "@effect/opentelemetry/NodeSdk"
import { PrometheusExporter } from "@opentelemetry/exporter-prometheus"
import { MeterProvider } from "@opentelemetry/sdk-metrics"
import { Effect, Layer } from "effect"

// Prometheus integration layer
export const PrometheusMetricsLayer = Layer.effectDiscard(
  Effect.gen(function* () {
    // Configure Prometheus exporter
    const prometheusExporter = new PrometheusExporter({
      port: 9090,
      endpoint: "/metrics"
    }, () => {
      console.log("Prometheus metrics server started on :9090/metrics")
    })
    
    // Create OpenTelemetry configuration
    yield* NodeSdk.layer(() => ({
      resource: {
        serviceName: process.env.SERVICE_NAME || "effect-app",
        serviceVersion: process.env.SERVICE_VERSION || "1.0.0"
      },
      metricReader: prometheusExporter,
      instrumentations: []
    }))
  })
)

// Application with Prometheus metrics
const app = Effect.gen(function* () {
  const metrics = createMetrics("my-service", "1.0.0")
  
  // Your application logic with metrics
  const server = HttpServer.serve(
    HttpRouter.empty.pipe(
      HttpRouter.get("/health", Effect.succeed(HttpResponse.text("OK"))),
      HttpRouter.get("/api/users/:id", getUserHandler),
      HttpRouter.post("/api/users", createUserHandler)
    )
  ).pipe(
    createMetricsMiddleware(metrics)
  )
  
  yield* server
}).pipe(
  Effect.provide(PrometheusMetricsLayer)
)
```

### Integration with Application Performance Monitoring (APM)

Connect Effect Metrics with APM tools like New Relic or DataDog:

```typescript
// Custom metric exporter for APM integration
export const createAPMExporter = (config: {
  apiKey: string
  endpoint: string
  serviceName: string
}) => {
  const exportMetrics = (metrics: MetricPair.MetricPair.Untyped[]) => 
    Effect.gen(function* () {
      const payload = metrics.map(pair => ({
        name: pair.metricKey.name,
        type: pair.metricKey.keyType._tag,
        value: extractMetricValue(pair.metricState),
        tags: pair.metricKey.tags.reduce((acc, tag) => ({
          ...acc,
          [tag.key]: tag.value
        }), {} as Record<string, string>),
        timestamp: Date.now(),
        service: config.serviceName
      }))
      
      yield* HttpClient.post(config.endpoint, {
        headers: {
          "Authorization": `Bearer ${config.apiKey}`,
          "Content-Type": "application/json"
        },
        body: HttpBody.json(payload)
      }).pipe(
        Effect.catchAll(error => 
          Effect.logError(`Failed to export metrics to APM: ${error}`)
        )
      )
    })
  
  // Export metrics every 60 seconds
  return Effect.gen(function* () {
    yield* Effect.gen(function* () {
      const snapshot = yield* Metric.snapshot
      yield* exportMetrics(snapshot)
    }).pipe(
      Effect.repeat(Schedule.fixed("60 seconds")),
      Effect.fork
    )
  })
}

// Business logic with APM integration
const businessApp = Effect.gen(function* () {
  const metrics = createMetrics("business-service", "2.1.0")
  
  // Start APM exporter
  yield* createAPMExporter({
    apiKey: process.env.APM_API_KEY!,
    endpoint: "https://api.apm-provider.com/v1/metrics",
    serviceName: "business-service"
  })
  
  // Your business logic with metrics
  yield* runBusinessOperations(metrics)
})
```

### Testing Strategies

Comprehensive testing approaches for metrics-enabled applications:

```typescript
// Test utilities for metrics
export const createTestMetrics = () => {
  const capturedMetrics = new Map<string, unknown[]>()
  
  const createMockMetric = <Type, In, Out>(name: string) => {
    const metric = Metric.counter(name) // or appropriate type
    
    // Capture updates for testing
    return metric.pipe(
      Metric.mapInput((input: In) => {
        const existing = capturedMetrics.get(name) || []
        capturedMetrics.set(name, [...existing, input])
        return input
      })
    )
  }
  
  return {
    createMockMetric,
    getCapturedUpdates: (metricName: string) => capturedMetrics.get(metricName) || [],
    clearCaptured: () => capturedMetrics.clear(),
    getAllCaptured: () => Object.fromEntries(capturedMetrics.entries())
  }
}

// Test example
describe("User Service Metrics", () => {
  it("should track user registration metrics", async () => {
    const testMetrics = createTestMetrics()
    const mockMetrics = {
      userRegistrations: testMetrics.createMockMetric<any, number, any>("user_registrations"),
      registrationErrors: testMetrics.createMockMetric<any, string, any>("registration_errors")
    }
    
    const registerUser = (userData: UserData) => Effect.gen(function* () {
      const result = yield* validateAndCreateUser(userData).pipe(
        Effect.tap(() => Metric.increment(mockMetrics.userRegistrations)),
        Effect.catchTag("ValidationError", (error) =>
          Metric.update(mockMetrics.registrationErrors, "validation").pipe(
            Effect.zipRight(Effect.fail(error))
          )
        )
      )
      return result
    })
    
    // Test successful registration
    await Effect.runPromise(registerUser(validUserData))
    
    expect(testMetrics.getCapturedUpdates("user_registrations")).toEqual([1])
    expect(testMetrics.getCapturedUpdates("registration_errors")).toEqual([])
    
    // Test failed registration
    await Effect.runPromise(
      registerUser(invalidUserData).pipe(Effect.either)
    )
    
    expect(testMetrics.getCapturedUpdates("registration_errors")).toEqual(["validation"])
  })
  
  it("should measure operation duration", async () => {
    const timer = Metric.timer("operation_duration")
    let capturedDuration: Duration.Duration | null = null
    
    const mockTimer = timer.pipe(
      Metric.mapInput((duration: Duration.Duration) => {
        capturedDuration = duration
        return duration
      })
    )
    
    const slowOperation = Effect.gen(function* () {
      yield* Effect.sleep("100 millis")
      return "completed"
    }).pipe(
      Metric.trackDuration(mockTimer)
    )
    
    await Effect.runPromise(slowOperation)
    
    expect(capturedDuration).not.toBeNull()
    expect(Duration.toMillis(capturedDuration!)).toBeGreaterThanOrEqual(100)
  })
})

// Property-based testing for metrics
describe("Metric Properties", () => {
  it("counter should only increase", async () => {
    const counter = Metric.counter("test_counter")
    
    const property = fc.property(
      fc.array(fc.nat(100), { minLength: 1, maxLength: 50 }),
      async (increments) => {
        const initialValue = await Effect.runPromise(Metric.value(counter))
        
        for (const increment of increments) {
          await Effect.runPromise(Metric.incrementBy(counter, increment))
        }
        
        const finalValue = await Effect.runPromise(Metric.value(counter))
        const expectedIncrease = increments.reduce((sum, inc) => sum + inc, 0)
        
        expect(finalValue.count).toEqual(initialValue.count + expectedIncrease)
      }
    )
    
    await fc.assert(property)
  })
})
```

## Conclusion

Metric provides type-safe, composable, and thread-safe metrics collection for Effect applications, eliminating the common pitfalls of traditional monitoring approaches.

Key benefits:
- **Type Safety**: Compile-time guarantees for metric types and operations
- **Composability**: Combine and transform metrics using familiar Effect patterns  
- **Thread Safety**: Concurrent metric updates without race conditions or data corruption
- **Integration Ready**: Seamless integration with popular monitoring platforms
- **Performance**: Efficient metric collection with minimal runtime overhead
- **Testing Support**: Comprehensive testing utilities for metrics-enabled applications

Effect Metrics shine when you need reliable, scalable observability that integrates naturally with your Effect-based application architecture, providing production-ready monitoring without sacrificing type safety or composability.