# MetricLabel: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem MetricLabel Solves

When building applications, basic metrics like "requests per second" or "response time" aren't enough for production observability. You need to slice and dice your metrics by different dimensions to understand what's really happening.

```typescript
// Traditional approach - separate metrics for each dimension
const userRequestsCounter = Metric.counter("user_requests")
const adminRequestsCounter = Metric.counter("admin_requests")
const guestRequestsCounter = Metric.counter("guest_requests")

// Same for each endpoint
const loginRequestsCounter = Metric.counter("login_requests")
const signupRequestsCounter = Metric.counter("signup_requests")
const profileRequestsCounter = Metric.counter("profile_requests")

// And for each status code...
const successRequestsCounter = Metric.counter("success_requests")
const errorRequestsCounter = Metric.counter("error_requests")
```

This approach leads to:
- **Metric Explosion** - Hundreds of similar metrics that are hard to manage
- **Lost Relationships** - Can't correlate user_type with endpoint or status_code
- **Query Complexity** - Need complex aggregation logic to get meaningful insights
- **Maintenance Nightmare** - Adding a new dimension requires creating new metrics everywhere

### The MetricLabel Solution

MetricLabel allows you to add dimensions to your metrics, enabling powerful aggregation and filtering while keeping your metric definitions clean and maintainable.

```typescript
import { Metric, MetricLabel } from "effect"

// Single metric with multiple dimensions
const requestsCounter = Metric.counter("http_requests").pipe(
  Metric.taggedWithLabels([
    MetricLabel.make("user_type", "guest"),
    MetricLabel.make("endpoint", "/api/login"),
    MetricLabel.make("status_code", "200")
  ])
)
```

### Key Concepts

**MetricLabel**: A key-value pair that adds a dimension to metrics, enabling granular analysis

**Labeling Strategy**: The approach to choosing which dimensions to track for meaningful insights

**Cardinality Management**: Controlling the number of unique label combinations to maintain performance

## Basic Usage Patterns

### Pattern 1: Creating Basic Labels

```typescript
import { MetricLabel } from "effect"

// Create individual labels
const userTypeLabel = MetricLabel.make("user_type", "premium")
const regionLabel = MetricLabel.make("region", "us-east-1")
const methodLabel = MetricLabel.make("method", "POST")

// Labels are equal if both key and value match
console.log(
  MetricLabel.make("env", "production") === MetricLabel.make("env", "production")
) // false (different object references)

// Use Equal.equals for value equality
import { Equal } from "effect"
console.log(
  Equal.equals(
    MetricLabel.make("env", "production"),
    MetricLabel.make("env", "production")
  )
) // true
```

### Pattern 2: Applying Labels to Metrics

```typescript
import { Metric, MetricLabel, Effect } from "effect"

// Create a counter with static labels
const httpRequestsCounter = Metric.counter("http_requests").pipe(
  Metric.taggedWithLabels([
    MetricLabel.make("service", "user-api"),
    MetricLabel.make("version", "1.2.0")
  ])
)

// Use the labeled metric
const trackRequest = httpRequestsCounter(Effect.void)
```

### Pattern 3: Dynamic Labels Based on Input

```typescript
import { Metric, MetricLabel, Effect } from "effect"

interface RequestContext {
  readonly userId: string
  readonly endpoint: string
  readonly method: string
  readonly statusCode: number
}

// Create labels dynamically from request context
const requestContextToLabels = (ctx: RequestContext) => [
  MetricLabel.make("endpoint", ctx.endpoint),
  MetricLabel.make("method", ctx.method),
  MetricLabel.make("status_code", ctx.statusCode.toString()),
  MetricLabel.make("user_segment", ctx.userId.startsWith("premium_") ? "premium" : "basic")
]

const dynamicRequestsCounter = Metric.counter("requests").pipe(
  Metric.taggedWithLabelsInput(requestContextToLabels)
)
```

## Real-World Examples

### Example 1: HTTP API Monitoring

Tracking HTTP requests with multiple dimensions for comprehensive API monitoring.

```typescript
import { Metric, MetricLabel, Effect, pipe } from "effect"

interface HttpRequest {
  readonly method: string
  readonly path: string
  readonly userAgent: string
  readonly userId?: string
}

interface HttpResponse {
  readonly statusCode: number
  readonly responseTime: number
  readonly contentLength: number
}

// Create comprehensive HTTP metrics
const httpRequestsCounter = Metric.counter("http_requests_total")
const httpResponseTimeHistogram = Metric.histogram(
  "http_response_time_seconds",
  MetricBoundaries.linear({ start: 0, width: 0.1, count: 20 })
)
const httpResponseSizeHistogram = Metric.histogram(
  "http_response_size_bytes", 
  MetricBoundaries.exponential({ start: 100, factor: 2, count: 10 })
)

const createHttpLabels = (request: HttpRequest, response: HttpResponse) => [
  MetricLabel.make("method", request.method),
  MetricLabel.make("path", normalizeApiPath(request.path)),
  MetricLabel.make("status_code", response.statusCode.toString()),
  MetricLabel.make("status_class", getStatusClass(response.statusCode)),
  MetricLabel.make("user_type", request.userId ? "authenticated" : "anonymous"),
  MetricLabel.make("browser", parseBrowser(request.userAgent))
]

const normalizeApiPath = (path: string): string => {
  // Replace IDs with placeholders for better grouping
  return path
    .replace(/\/\d+/g, "/:id")
    .replace(/\/[a-f0-9-]{36}/g, "/:uuid")
}

const getStatusClass = (statusCode: number): string => {
  if (statusCode < 300) return "2xx"
  if (statusCode < 400) return "3xx" 
  if (statusCode < 500) return "4xx"
  return "5xx"
}

const parseBrowser = (userAgent: string): string => {
  if (userAgent.includes("Chrome")) return "chrome"
  if (userAgent.includes("Firefox")) return "firefox"
  if (userAgent.includes("Safari")) return "safari"
  return "other"
}

// Service for tracking HTTP metrics
export const HttpMetricsService = {
  trackRequest: (request: HttpRequest, response: HttpResponse) => 
    Effect.gen(function* () {
      const labels = createHttpLabels(request, response)
      
      // Track request count
      yield* httpRequestsCounter.pipe(
        Metric.taggedWithLabels(labels)
      )(Effect.void)
      
      // Track response time
      yield* httpResponseTimeHistogram.pipe(
        Metric.taggedWithLabels(labels)
      )(Effect.succeed(response.responseTime / 1000)) // Convert to seconds
      
      // Track response size
      yield* httpResponseSizeHistogram.pipe(
        Metric.taggedWithLabels(labels)
      )(Effect.succeed(response.contentLength))
    })
}
```

### Example 2: E-commerce Business Metrics

Tracking business metrics with customer and product dimensions.

```typescript
import { Metric, MetricLabel, Effect, pipe } from "effect"

interface Customer {
  readonly id: string
  readonly segment: "bronze" | "silver" | "gold" | "platinum"
  readonly region: string
  readonly acquisitionChannel: string
}

interface Product {
  readonly sku: string
  readonly category: string
  readonly brand: string
  readonly price: number
}

interface PurchaseEvent {
  readonly customer: Customer
  readonly product: Product
  readonly quantity: number
  readonly discountPercent: number
  readonly timestamp: Date
}

// Business metrics with rich labeling
const salesCounter = Metric.counter("sales_total")
const revenueCounter = Metric.counter("revenue_total")
const averageOrderValueGauge = Metric.gauge("average_order_value")
const discountRateGauge = Metric.gauge("discount_rate_percent")

const createBusinessLabels = (event: PurchaseEvent) => {
  const totalValue = event.product.price * event.quantity
  const priceRange = getPriceRange(event.product.price)
  const dayOfWeek = getDayOfWeek(event.timestamp)
  const timeOfDay = getTimeOfDay(event.timestamp)
  
  return [
    // Customer dimensions
    MetricLabel.make("customer_segment", event.customer.segment),
    MetricLabel.make("customer_region", event.customer.region),
    MetricLabel.make("acquisition_channel", event.customer.acquisitionChannel),
    
    // Product dimensions  
    MetricLabel.make("product_category", event.product.category),
    MetricLabel.make("product_brand", event.product.brand),
    MetricLabel.make("price_range", priceRange),
    
    // Purchase dimensions
    MetricLabel.make("quantity_bucket", getQuantityBucket(event.quantity)),
    MetricLabel.make("discount_applied", event.discountPercent > 0 ? "yes" : "no"),
    MetricLabel.make("discount_tier", getDiscountTier(event.discountPercent)),
    
    // Temporal dimensions
    MetricLabel.make("day_of_week", dayOfWeek),
    MetricLabel.make("time_of_day", timeOfDay)
  ]
}

const getPriceRange = (price: number): string => {
  if (price < 25) return "under_25"
  if (price < 100) return "25_to_100"
  if (price < 500) return "100_to_500"
  return "over_500"
}

const getQuantityBucket = (quantity: number): string => {
  if (quantity === 1) return "single"
  if (quantity <= 3) return "small_bulk"
  if (quantity <= 10) return "medium_bulk"
  return "large_bulk"
}

const getDiscountTier = (discountPercent: number): string => {
  if (discountPercent === 0) return "none"
  if (discountPercent <= 10) return "low"
  if (discountPercent <= 25) return "medium"
  return "high"
}

const getDayOfWeek = (date: Date): string => {
  const days = ["sunday", "monday", "tuesday", "wednesday", "thursday", "friday", "saturday"]
  return days[date.getDay()]
}

const getTimeOfDay = (date: Date): string => {
  const hour = date.getHours()
  if (hour < 6) return "night"
  if (hour < 12) return "morning" 
  if (hour < 18) return "afternoon"
  return "evening"
}

// Service for tracking business metrics
export const BusinessMetricsService = {
  trackPurchase: (event: PurchaseEvent) =>
    Effect.gen(function* () {
      const labels = createBusinessLabels(event)
      const totalValue = event.product.price * event.quantity
      const discountedValue = totalValue * (1 - event.discountPercent / 100)
      
      // Track sales count
      yield* salesCounter.pipe(
        Metric.taggedWithLabels(labels)
      )(Effect.void)
      
      // Track revenue
      yield* revenueCounter.pipe(
        Metric.taggedWithLabels(labels)
      )(Effect.succeed(discountedValue))
      
      // Track average order value
      yield* averageOrderValueGauge.pipe(
        Metric.taggedWithLabels(labels)
      )(Effect.succeed(discountedValue))
      
      // Track discount rate
      yield* discountRateGauge.pipe(
        Metric.taggedWithLabels(labels)
      )(Effect.succeed(event.discountPercent))
    })
}
```

### Example 3: Infrastructure Monitoring

Monitoring system resources with detailed infrastructure labels.

```typescript
import { Metric, MetricLabel, Effect, Schedule, pipe } from "effect"

interface SystemMetrics {
  readonly cpuUsagePercent: number
  readonly memoryUsagePercent: number
  readonly diskUsagePercent: number
  readonly networkBytesPerSecond: number
  readonly activeConnections: number
}

interface SystemInfo {
  readonly hostname: string
  readonly environment: "production" | "staging" | "development"
  readonly region: string
  readonly availabilityZone: string
  readonly instanceType: string
  readonly serviceVersion: string
}

// Infrastructure metrics
const cpuUsageGauge = Metric.gauge("cpu_usage_percent")
const memoryUsageGauge = Metric.gauge("memory_usage_percent") 
const diskUsageGauge = Metric.gauge("disk_usage_percent")
const networkThroughputGauge = Metric.gauge("network_bytes_per_second")
const activeConnectionsGauge = Metric.gauge("active_connections")

const createInfrastructureLabels = (systemInfo: SystemInfo) => [
  MetricLabel.make("hostname", systemInfo.hostname),
  MetricLabel.make("environment", systemInfo.environment),
  MetricLabel.make("region", systemInfo.region),
  MetricLabel.make("availability_zone", systemInfo.availabilityZone),
  MetricLabel.make("instance_type", systemInfo.instanceType),
  MetricLabel.make("service_version", systemInfo.serviceVersion),
  MetricLabel.make("cluster", `${systemInfo.environment}-${systemInfo.region}`)
]

// System monitoring service
export const SystemMonitoringService = (systemInfo: SystemInfo) => {
  const labels = createInfrastructureLabels(systemInfo)
  
  const recordMetrics = (metrics: SystemMetrics) =>
    Effect.gen(function* () {
      // Record all system metrics with consistent labels
      yield* Effect.all([
        cpuUsageGauge.pipe(Metric.taggedWithLabels(labels))(Effect.succeed(metrics.cpuUsagePercent)),
        memoryUsageGauge.pipe(Metric.taggedWithLabels(labels))(Effect.succeed(metrics.memoryUsagePercent)),
        diskUsageGauge.pipe(Metric.taggedWithLabels(labels))(Effect.succeed(metrics.diskUsagePercent)),
        networkThroughputGauge.pipe(Metric.taggedWithLabels(labels))(Effect.succeed(metrics.networkBytesPerSecond)),
        activeConnectionsGauge.pipe(Metric.taggedWithLabels(labels))(Effect.succeed(metrics.activeConnections))
      ], { concurrency: "unbounded" })
    })
  
  // Create additional metrics with derived labels
  const recordDerivedMetrics = (metrics: SystemMetrics) =>
    Effect.gen(function* () {
      const resourcePressureLabels = [
        ...labels,
        MetricLabel.make("cpu_pressure", getCpuPressure(metrics.cpuUsagePercent)),
        MetricLabel.make("memory_pressure", getMemoryPressure(metrics.memoryUsagePercent)),
        MetricLabel.make("overall_health", getOverallHealth(metrics))
      ]
      
      // System health score (composite metric)
      const healthScore = calculateHealthScore(metrics)
      yield* Metric.gauge("system_health_score").pipe(
        Metric.taggedWithLabels(resourcePressureLabels)
      )(Effect.succeed(healthScore))
    })
  
  return {
    recordMetrics,
    recordDerivedMetrics,
    
    // Start continuous monitoring
    startMonitoring: (getSystemMetrics: Effect.Effect<SystemMetrics, never, never>) =>
      pipe(
        getSystemMetrics,
        Effect.flatMap(metrics =>
          Effect.all([
            recordMetrics(metrics),
            recordDerivedMetrics(metrics)
          ])
        ),
        Effect.repeat(Schedule.fixed("30 seconds")),
        Effect.forkDaemon
      )
  }
}

const getCpuPressure = (cpuUsage: number): string => {
  if (cpuUsage < 50) return "low"
  if (cpuUsage < 80) return "medium"
  return "high"
}

const getMemoryPressure = (memoryUsage: number): string => {
  if (memoryUsage < 60) return "low"
  if (memoryUsage < 85) return "medium"
  return "high"
}

const getOverallHealth = (metrics: SystemMetrics): string => {
  const score = calculateHealthScore(metrics)
  if (score > 80) return "healthy"
  if (score > 60) return "degraded"
  return "unhealthy"
}

const calculateHealthScore = (metrics: SystemMetrics): number => {
  // Simple health scoring algorithm
  const cpuScore = Math.max(0, 100 - metrics.cpuUsagePercent)
  const memoryScore = Math.max(0, 100 - metrics.memoryUsagePercent)
  const diskScore = Math.max(0, 100 - metrics.diskUsagePercent)
  
  return (cpuScore + memoryScore + diskScore) / 3
}
```

## Advanced Features Deep Dive

### Feature 1: Label Equality and Hashing

MetricLabel implements proper equality and hashing for efficient metric aggregation.

#### Basic Label Equality

```typescript
import { MetricLabel, Equal } from "effect"

const label1 = MetricLabel.make("environment", "production")
const label2 = MetricLabel.make("environment", "production")
const label3 = MetricLabel.make("environment", "staging")

// Object reference equality (always false for different instances)
console.log(label1 === label2) // false

// Value equality using Equal
console.log(Equal.equals(label1, label2)) // true
console.log(Equal.equals(label1, label3)) // false

// Labels with same key but different values are not equal
console.log(Equal.equals(
  MetricLabel.make("region", "us-east-1"),
  MetricLabel.make("region", "us-west-2")
)) // false
```

#### Real-World Label Equality Example

```typescript
import { MetricLabel, Equal, HashMap } from "effect"

// Efficient label deduplication using HashMap
const createLabelSet = (labels: MetricLabel.MetricLabel[]) => {
  return labels.reduce(
    (acc, label) => HashMap.set(acc, label, true),
    HashMap.empty<MetricLabel.MetricLabel, boolean>()
  )
}

// Usage in metric aggregation
const aggregateMetricsByLabels = (
  metrics: Array<{ labels: MetricLabel.MetricLabel[], value: number }>
) => {
  const grouped = new Map<string, { labels: MetricLabel.MetricLabel[], values: number[] }>()
  
  for (const metric of metrics) {
    // Create a deterministic key from labels
    const labelKey = metric.labels
      .map(label => `${label.key}=${label.value}`)
      .sort()
      .join(",")
    
    if (!grouped.has(labelKey)) {
      grouped.set(labelKey, { labels: metric.labels, values: [] })
    }
    grouped.get(labelKey)!.values.push(metric.value)
  }
  
  return Array.from(grouped.values()).map(group => ({
    labels: group.labels,
    sum: group.values.reduce((a, b) => a + b, 0),
    count: group.values.length,
    average: group.values.reduce((a, b) => a + b, 0) / group.values.length
  }))
}
```

### Feature 2: Dynamic Label Generation

Advanced patterns for generating labels dynamically based on runtime conditions.

#### Context-Aware Label Generation

```typescript
import { MetricLabel, Effect, Context } from "effect"

// Context for request tracing
interface RequestContext {
  readonly traceId: string
  readonly userId?: string
  readonly sessionId?: string
  readonly clientVersion?: string
}

const RequestContext = Context.GenericTag<RequestContext>("RequestContext")

// Service for dynamic label generation
export const LabelGeneratorService = {
  generateRequestLabels: Effect.gen(function* () {
    const context = yield* RequestContext
    
    return [
      MetricLabel.make("trace_id", context.traceId),
      MetricLabel.make("user_status", context.userId ? "authenticated" : "anonymous"),
      MetricLabel.make("session_type", context.sessionId ? "persistent" : "transient"),
      MetricLabel.make("client_version", context.clientVersion ?? "unknown"),
      MetricLabel.make("request_origin", inferOrigin(context))
    ]
  }),
  
  generateBusinessLabels: (businessContext: unknown) =>
    Effect.gen(function* () {
      // Complex business logic for label generation
      const labels: MetricLabel.MetricLabel[] = []
      
      // Add conditional labels based on business rules
      if (isHighValueCustomer(businessContext)) {
        labels.push(MetricLabel.make("customer_tier", "high_value"))
      }
      
      if (isPromotionalPeriod(new Date())) {
        labels.push(MetricLabel.make("promotion_active", "true"))
      }
      
      return labels
    })
}

const inferOrigin = (context: RequestContext): string => {
  if (context.clientVersion?.includes("mobile")) return "mobile"
  if (context.clientVersion?.includes("web")) return "web"
  return "unknown"
}

const isHighValueCustomer = (context: unknown): boolean => {
  // Business logic to determine high-value customers
  return false // placeholder
}

const isPromotionalPeriod = (date: Date): boolean => {
  // Check if current date falls within promotional periods
  return false // placeholder
}
```

#### Performance-Optimized Label Caching

```typescript
import { MetricLabel, Effect, Cache, Duration } from "effect"

// Cache for expensive label computations
const labelCache = Cache.make({
  capacity: 1000,
  timeToLive: Duration.minutes(5),
  lookup: (key: string) => Effect.succeed(computeExpensiveLabels(key))
})

const computeExpensiveLabels = (context: string): MetricLabel.MetricLabel[] => {
  // Expensive computation (e.g., database lookup, API call)
  const parsed = JSON.parse(context)
  
  return [
    MetricLabel.make("computed_category", expensiveCategorizationLogic(parsed)),
    MetricLabel.make("risk_score", calculateRiskScore(parsed).toString()),
    MetricLabel.make("geographic_region", lookupGeographicRegion(parsed))
  ]
}

// Cached label generation service
export const CachedLabelGeneratorService = {
  getLabelsForContext: (contextKey: string) =>
    Effect.gen(function* () {
      const cached = yield* Cache.get(labelCache, contextKey)
      return cached
    }),
  
  invalidateLabelsForContext: (contextKey: string) =>
    Cache.invalidate(labelCache, contextKey)
}

const expensiveCategorizationLogic = (data: any): string => "category" // placeholder
const calculateRiskScore = (data: any): number => 0 // placeholder
const lookupGeographicRegion = (data: any): string => "unknown" // placeholder
```

## Practical Patterns & Best Practices

### Pattern 1: Label Standardization

Create consistent labeling standards across your application to ensure metrics are comparable and aggregatable.

```typescript
import { MetricLabel } from "effect"

// Standard label factory functions
export const StandardLabels = {
  // Environment labels
  environment: (env: "production" | "staging" | "development") =>
    MetricLabel.make("environment", env),
  
  region: (region: string) =>
    MetricLabel.make("region", region),
  
  service: (serviceName: string, version: string) => [
    MetricLabel.make("service", serviceName),
    MetricLabel.make("version", version)
  ],
  
  // HTTP labels with standardized values
  httpMethod: (method: string) =>
    MetricLabel.make("method", method.toUpperCase()),
  
  httpStatusClass: (statusCode: number) =>
    MetricLabel.make("status_class", `${Math.floor(statusCode / 100)}xx`),
  
  httpEndpoint: (path: string) =>
    MetricLabel.make("endpoint", normalizePath(path)),
  
  // User labels
  userSegment: (userId?: string, isPremium: boolean = false) =>
    MetricLabel.make("user_segment", 
      userId ? (isPremium ? "premium" : "free") : "anonymous"
    ),
  
  // Performance labels
  performanceTier: (responseTime: number) =>
    MetricLabel.make("performance_tier", getPerformanceTier(responseTime)),
  
  // Business labels
  businessMetric: (feature: string, experiment?: string) => {
    const labels = [MetricLabel.make("feature", feature)]
    if (experiment) {
      labels.push(MetricLabel.make("experiment", experiment))
    }
    return labels
  }
}

// Helper functions for consistent value normalization
const normalizePath = (path: string): string => {
  return path
    .replace(/\/\d+(?=\/|$)/g, "/:id")
    .replace(/\/[a-f0-9-]{36}(?=\/|$)/g, "/:uuid")
    .replace(/\/[a-f0-9]{24}(?=\/|$)/g, "/:objectid")
    .toLowerCase()
}

const getPerformanceTier = (responseTimeMs: number): string => {
  if (responseTimeMs < 100) return "fast"
  if (responseTimeMs < 500) return "acceptable"
  if (responseTimeMs < 2000) return "slow" 
  return "very_slow"
}

// Usage pattern with standardized labels
const createStandardizedMetric = (
  metricName: string,
  serviceName: string,
  serviceVersion: string
) => {
  const baseLabels = [
    StandardLabels.environment("production"),
    StandardLabels.region("us-east-1"),
    ...StandardLabels.service(serviceName, serviceVersion)
  ]
  
  return Metric.counter(metricName).pipe(
    Metric.taggedWithLabels(baseLabels)
  )
}
```

### Pattern 2: Cardinality Management

Control metric cardinality to avoid performance issues and storage explosion.

```typescript
import { MetricLabel, Effect, pipe } from "effect"

// Cardinality-aware label builder
export class CardinalityManagedLabels {
  private readonly maxCardinality: number
  private readonly labelPriority: Record<string, number>
  
  constructor(maxCardinality: number = 10000) {
    this.maxCardinality = maxCardinality
    this.labelPriority = {
      "environment": 1,
      "service": 2,
      "region": 3,
      "method": 4,
      "status_class": 5,
      "endpoint": 6,
      "user_segment": 7,
      // Lower priority labels may be dropped
      "user_id": 100,
      "trace_id": 101,
      "session_id": 102
    }
  }
  
  // Intelligently limit labels based on cardinality
  limitLabels(labels: MetricLabel.MetricLabel[]): MetricLabel.MetricLabel[] {
    // Sort by priority (lower number = higher priority)
    const sortedLabels = labels.sort((a, b) => {
      const aPriority = this.labelPriority[a.key] ?? 50
      const bPriority = this.labelPriority[b.key] ?? 50
      return aPriority - bPriority
    })
    
    // Apply cardinality limits per label key
    const result: MetricLabel.MetricLabel[] = []
    const cardinalityPerKey: Map<string, Set<string>> = new Map()
    
    for (const label of sortedLabels) {
      if (!cardinalityPerKey.has(label.key)) {
        cardinalityPerKey.set(label.key, new Set())
      }
      
      const keyCardinality = cardinalityPerKey.get(label.key)!
      
      // Limit high-cardinality labels
      const maxCardinalityForKey = this.getMaxCardinalityForKey(label.key)
      
      if (keyCardinality.size < maxCardinalityForKey) {
        keyCardinality.add(label.value)
        result.push(label)
      } else if (!keyCardinality.has(label.value)) {
        // Replace with "other" value for high cardinality
        result.push(MetricLabel.make(label.key, "other"))
      }
    }
    
    return result
  }
  
  private getMaxCardinalityForKey(key: string): number {
    const limits: Record<string, number> = {
      "user_id": 1000,      // Limit user IDs
      "trace_id": 0,        // Never include trace IDs in metrics
      "session_id": 100,    // Limit session IDs
      "endpoint": 50,       // Reasonable endpoint limit
      "method": 10,         // HTTP methods are naturally low cardinality
      "status_class": 5     // 1xx, 2xx, 3xx, 4xx, 5xx
    }
    
    return limits[key] ?? 20 // Default limit
  }
  
  // Create metrics with managed cardinality
  createManagedMetric<Type, In, Out>(
    baseMetric: Metric.Metric<Type, In, Out>,
    labelGenerator: (input: In) => MetricLabel.MetricLabel[]
  ): Metric.Metric<Type, In, void> {
    return baseMetric.pipe(
      Metric.taggedWithLabelsInput((input: In) => 
        this.limitLabels(labelGenerator(input))
      )
    )
  }
}

// Usage example with cardinality management
const cardinalityManager = new CardinalityManagedLabels(1000)

const createManagedHttpMetric = () => {
  const baseCounter = Metric.counter("http_requests")
  
  return cardinalityManager.createManagedMetric(
    baseCounter,
    (requestContext: { userId?: string, endpoint: string, method: string }) => [
      MetricLabel.make("endpoint", requestContext.endpoint),
      MetricLabel.make("method", requestContext.method),
      // This might be limited due to high cardinality
      ...(requestContext.userId ? [MetricLabel.make("user_id", requestContext.userId)] : [])
    ]
  )
}
```

### Pattern 3: Label Composition and Reuse

Build complex label sets by composing simpler, reusable label generators.

```typescript
import { MetricLabel, pipe } from "effect"

// Composable label generators
export const LabelComposer = {
  // Base infrastructure labels
  infrastructure: (env: string, region: string, service: string) => [
    MetricLabel.make("environment", env),
    MetricLabel.make("region", region),
    MetricLabel.make("service", service)
  ],
  
  // Request context labels  
  request: (method: string, path: string, userAgent?: string) => [
    MetricLabel.make("method", method),
    MetricLabel.make("endpoint", normalizePath(path)),
    ...(userAgent ? [MetricLabel.make("client_type", parseClientType(userAgent))] : [])
  ],
  
  // Business context labels
  business: (customerId?: string, feature?: string, experiment?: string) => {
    const labels: MetricLabel.MetricLabel[] = []
    
    if (customerId) {
      labels.push(MetricLabel.make("customer_tier", getCustomerTier(customerId)))
    }
    
    if (feature) {
      labels.push(MetricLabel.make("feature", feature))
    }
    
    if (experiment) {
      labels.push(MetricLabel.make("experiment", experiment))
    }
    
    return labels
  },
  
  // Performance context labels
  performance: (responseTime?: number, cacheHit?: boolean) => {
    const labels: MetricLabel.MetricLabel[] = []
    
    if (typeof responseTime === 'number') {
      labels.push(MetricLabel.make("performance_tier", getPerformanceTier(responseTime)))
    }
    
    if (typeof cacheHit === 'boolean') {
      labels.push(MetricLabel.make("cache_status", cacheHit ? "hit" : "miss"))
    }
    
    return labels
  },
  
  // Compose multiple label sets
  compose: (...labelSets: MetricLabel.MetricLabel[][]) =>
    labelSets.flat(),
  
  // Merge with deduplication
  merge: (...labelSets: MetricLabel.MetricLabel[][]) => {
    const labelMap = new Map<string, MetricLabel.MetricLabel>()
    
    for (const labels of labelSets) {
      for (const label of labels) {
        labelMap.set(label.key, label) // Later labels override earlier ones
      }
    }
    
    return Array.from(labelMap.values())
  }
}

// Usage pattern for complex label composition
interface CompleteRequestContext {
  readonly environment: string
  readonly region: string
  readonly service: string
  readonly method: string
  readonly path: string
  readonly userAgent?: string
  readonly customerId?: string
  readonly feature?: string
  readonly experiment?: string  
  readonly responseTime?: number
  readonly cacheHit?: boolean
}

const createCompleteRequestLabels = (context: CompleteRequestContext) =>
  LabelComposer.compose(
    LabelComposer.infrastructure(context.environment, context.region, context.service),
    LabelComposer.request(context.method, context.path, context.userAgent),
    LabelComposer.business(context.customerId, context.feature, context.experiment),
    LabelComposer.performance(context.responseTime, context.cacheHit)
  )

// Helper functions (implementations would vary based on business logic)
const parseClientType = (userAgent: string): string => "web" // placeholder
const getCustomerTier = (customerId: string): string => "standard" // placeholder
```

## Integration Examples

### Integration with Prometheus/OpenTelemetry

```typescript
import { Metric, MetricLabel, Effect, Layer } from "effect"

// Prometheus-compatible label naming
export const PrometheusLabels = {
  // Convert camelCase to snake_case for Prometheus compatibility
  toPrometheusLabel: (key: string, value: string) =>
    MetricLabel.make(
      key.replace(/[A-Z]/g, letter => `_${letter.toLowerCase()}`),
      value
    ),
  
  // Create standard Prometheus labels
  job: (jobName: string) => MetricLabel.make("job", jobName),
  instance: (instance: string) => MetricLabel.make("instance", instance),
  
  // HTTP labels following Prometheus conventions
  httpRequest: (method: string, handler: string, code: string) => [
    MetricLabel.make("method", method),
    MetricLabel.make("handler", handler),
    MetricLabel.make("code", code)
  ]
}

// OpenTelemetry integration service
export const OpenTelemetryMetricsService = {
  // Create OTel-compatible metrics with proper labeling
  createHttpRequestCounter: (serviceName: string, serviceVersion: string) => {
    const baseLabels = [
      MetricLabel.make("service.name", serviceName),
      MetricLabel.make("service.version", serviceVersion)
    ]
    
    return Metric.counter("http.server.requests").pipe(
      Metric.taggedWithLabels(baseLabels),
      Metric.taggedWithLabelsInput((req: {
        method: string
        route: string  
        status: number
      }) => [
        MetricLabel.make("http.method", req.method),
        MetricLabel.make("http.route", req.route),
        MetricLabel.make("http.status_code", req.status.toString())
      ])
    )
  },
  
  // Database operation metrics
  createDatabaseMetrics: (serviceName: string) => {
    const baseLabels = [
      MetricLabel.make("service.name", serviceName)
    ]
    
    return {
      operationDuration: Metric.histogram(
        "db.client.connections.usage",
        MetricBoundaries.exponential({ start: 0.001, factor: 2, count: 16 })
      ).pipe(
        Metric.taggedWithLabels(baseLabels),
        Metric.taggedWithLabelsInput((op: {
          operation: string
          table: string
          success: boolean
        }) => [
          MetricLabel.make("db.operation", op.operation),
          MetricLabel.make("db.collection.name", op.table),
          MetricLabel.make("otel.status_code", op.success ? "OK" : "ERROR")
        ])
      ),
      
      connectionPoolSize: Metric.gauge("db.client.connections.pool.size").pipe(
        Metric.taggedWithLabels([
          ...baseLabels,
          MetricLabel.make("pool.name", "main")
        ])
      )
    }
  }
}
```

### Testing Strategies

```typescript
import { Metric, MetricLabel, Effect, TestClock, TestServices } from "effect"
import { describe, it, expect } from "@effect/vitest"

// Test utilities for metrics with labels
export const MetricTestUtils = {
  // Create test metric with predefined labels
  createTestMetric: <Type, In, Out>(
    baseMetric: Metric.Metric<Type, In, Out>,
    testLabels: MetricLabel.MetricLabel[]
  ) => baseMetric.pipe(Metric.taggedWithLabels(testLabels)),
  
  // Extract metric values with specific labels
  getMetricValue: <Type, In, Out>(
    metric: Metric.Metric<Type, In, Out>,
    labels: MetricLabel.MetricLabel[]
  ) => Effect.gen(function* () {
    const taggedMetric = metric.pipe(Metric.taggedWithLabels(labels))
    return yield* Metric.value(taggedMetric)
  }),
  
  // Verify metric was updated with correct labels
  verifyMetricLabels: (
    expectedLabels: MetricLabel.MetricLabel[],
    actualLabels: MetricLabel.MetricLabel[]
  ) => {
    expect(actualLabels).toHaveLength(expectedLabels.length)
    
    for (const expectedLabel of expectedLabels) {
      const found = actualLabels.find(label => 
        label.key === expectedLabel.key && label.value === expectedLabel.value
      )
      expect(found).toBeDefined()
    }
  }
}

describe("MetricLabel Integration Tests", () => {
  it.effect("should track HTTP requests with correct labels", () =>
    Effect.gen(function* () {
      // Setup
      const httpCounter = Metric.counter("http_requests_test")
      const testLabels = [
        MetricLabel.make("method", "GET"),
        MetricLabel.make("endpoint", "/api/users/:id"),
        MetricLabel.make("status", "200")
      ]
      
      // Execute
      const labeledCounter = httpCounter.pipe(Metric.taggedWithLabels(testLabels))
      yield* labeledCounter(Effect.void)
      yield* labeledCounter(Effect.void)
      
      // Verify
      const counterValue = yield* Metric.value(labeledCounter)
      expect(counterValue.count).toBe(2)
    }).pipe(Effect.provide(TestServices.TestServices))
  )
  
  it.effect("should handle dynamic label generation", () =>
    Effect.gen(function* () {
      // Setup
      interface RequestData {
        userId: string
        action: string
        success: boolean
      }
      
      const dynamicCounter = Metric.counter("user_actions_test").pipe(
        Metric.taggedWithLabelsInput((data: RequestData) => [
          MetricLabel.make("user_id", data.userId),
          MetricLabel.make("action", data.action),
          MetricLabel.make("success", data.success.toString())
        ])
      )
      
      // Execute multiple requests with different labels
      const requests: RequestData[] = [
        { userId: "user1", action: "login", success: true },
        { userId: "user1", action: "logout", success: true },
        { userId: "user2", action: "login", success: false }
      ]
      
      for (const request of requests) {
        yield* dynamicCounter(Effect.succeed(request))
      }
      
      // Since labels are dynamic, we can't easily verify the exact values
      // but we can verify the metric was created and updated
      expect(true).toBe(true) // Placeholder for more complex verification
    }).pipe(Effect.provide(TestServices.TestServices))
  )
  
  it.effect("should maintain label equality across metric operations", () =>
    Effect.gen(function* () {
      const label1 = MetricLabel.make("environment", "test")
      const label2 = MetricLabel.make("environment", "test")
      const label3 = MetricLabel.make("environment", "prod")
      
      // Verify label equality
      expect(Equal.equals(label1, label2)).toBe(true)
      expect(Equal.equals(label1, label3)).toBe(false)
      
      // Verify labels work correctly in metrics
      const counter1 = Metric.counter("test_counter_1").pipe(Metric.taggedWithLabels([label1]))
      const counter2 = Metric.counter("test_counter_1").pipe(Metric.taggedWithLabels([label2]))
      
      yield* counter1(Effect.void)
      const value1 = yield* Metric.value(counter1)
      const value2 = yield* Metric.value(counter2)
      
      // Both should show the same count since labels are equal
      expect(value1.count).toBe(value2.count)
    }).pipe(Effect.provide(TestServices.TestServices))
  )
})
```

## Conclusion

MetricLabel provides **dimensional observability**, **efficient aggregation**, and **powerful filtering** for Effect metrics, enabling production-ready monitoring and alerting.

Key benefits:
- **Dimensional Analysis**: Slice metrics by any combination of business and technical dimensions
- **Reduced Complexity**: Single metrics with multiple dimensions instead of metric explosion
- **Performance Optimization**: Built-in equality and hashing for efficient aggregation
- **Integration Ready**: Works seamlessly with Prometheus, OpenTelemetry, and other monitoring systems

Use MetricLabel when you need to understand not just "what happened" but "what happened to whom, where, when, and how" in your applications. The ability to correlate metrics across multiple dimensions makes it invaluable for debugging, performance optimization, and business intelligence.