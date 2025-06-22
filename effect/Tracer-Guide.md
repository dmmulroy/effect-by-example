# Tracer: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem Tracer Solves

Traditional application observability in distributed systems relies on logs and metrics, but these approaches fall short when tracking request flows across multiple services and understanding performance bottlenecks in complex operations.

```typescript
// Traditional approach - limited observability
async function processOrder(orderId: string) {
  console.log(`Processing order ${orderId}`)
  
  try {
    const user = await getUserById(userId)
    console.log(`Retrieved user ${user.id}`)
    
    const payment = await processPayment(order.total)
    console.log(`Payment processed: ${payment.id}`)
    
    const inventory = await updateInventory(order.items)
    console.log(`Inventory updated`)
    
    await sendConfirmationEmail(user.email)
    console.log(`Confirmation sent`)
    
  } catch (error) {
    console.error(`Order processing failed: ${error.message}`)
    throw error
  }
}
```

This approach leads to:
- **Limited Context** - No correlation between related operations across services
- **Poor Performance Visibility** - Cannot identify bottlenecks or understand timing relationships
- **Difficult Debugging** - No structured way to trace request flow through distributed systems
- **Missing Causality** - Cannot understand parent-child relationships between operations
- **Fragmented Monitoring** - Logs scattered across services without unified view

### The Tracer Solution

Effect's Tracer module provides comprehensive distributed tracing capabilities with automatic context propagation, hierarchical span management, and seamless integration with observability platforms like OpenTelemetry.

```typescript
import { Effect, Tracer } from "effect"

// Declarative tracing with automatic context propagation
const processOrder = (orderId: string) => Effect.gen(function* () {
  yield* Effect.annotateCurrentSpan("order.id", orderId)
  yield* Effect.log("Processing order", { orderId })
  
  const user = yield* getUserById(userId).pipe(
    Effect.withSpan("get-user", { attributes: { "user.id": userId } })
  )
  
  const payment = yield* processPayment(order.total).pipe(
    Effect.withSpan("process-payment", { 
      attributes: { "payment.amount": order.total },
      kind: "client" 
    })
  )
  
  const inventory = yield* updateInventory(order.items).pipe(
    Effect.withSpan("update-inventory", { kind: "internal" })
  )
  
  yield* sendConfirmationEmail(user.email).pipe(
    Effect.withSpan("send-email", { 
      attributes: { "email.recipient": user.email },
      kind: "producer" 
    })
  )
  
  return { orderId, status: "completed" }
}).pipe(
  Effect.withSpan("process-order", {
    attributes: { "service.name": "order-service" }
  })
)
```

### Key Concepts

**Span**: A single unit of work representing an operation with timing data, metadata, and hierarchical relationships

**Trace**: A collection of spans that represents the complete journey of a request through your system

**Context Propagation**: Automatic passing of trace context between operations and across service boundaries

**Span Annotations**: Key-value metadata attached to spans for rich contextual information

**Span Links**: Connections between spans in different traces for complex distributed scenarios

## Basic Usage Patterns

### Pattern 1: Creating Basic Spans

```typescript
import { Effect } from "effect"

// Simple span creation
const fetchUserData = (userId: string) => Effect.gen(function* () {
  yield* Effect.sleep("100 millis") // Simulate database call
  return { id: userId, name: "John Doe", email: "john@example.com" }
}).pipe(
  Effect.withSpan("fetch-user-data")
)

// Span with attributes
const authenticateUser = (credentials: UserCredentials) => Effect.gen(function* () {
  yield* Effect.annotateCurrentSpan("user.email", credentials.email)
  yield* Effect.annotateCurrentSpan("auth.method", "password")
  
  const isValid = yield* validateCredentials(credentials)
  yield* Effect.annotateCurrentSpan("auth.success", isValid)
  
  return isValid
}).pipe(
  Effect.withSpan("authenticate-user", {
    attributes: { "service.component": "authentication" }
  })
)
```

### Pattern 2: Nested Spans with Hierarchy

```typescript
// Parent-child span relationships
const processUserRegistration = (userData: UserData) => Effect.gen(function* () {
  yield* Effect.log("Starting user registration")
  
  // Child span for validation
  const validatedData = yield* validateUserData(userData).pipe(
    Effect.withSpan("validate-user-data", {
      attributes: { "validation.fields": Object.keys(userData).length }
    })
  )
  
  // Child span for database operations
  const user = yield* createUserInDatabase(validatedData).pipe(
    Effect.withSpan("create-user-db", {
      kind: "internal",
      attributes: { "db.operation": "insert", "db.table": "users" }
    })
  )
  
  // Child span for email service
  yield* sendWelcomeEmail(user.email).pipe(
    Effect.withSpan("send-welcome-email", {
      kind: "producer",
      attributes: { "email.template": "welcome", "email.provider": "sendgrid" }
    })
  )
  
  return user
}).pipe(
  Effect.withSpan("process-user-registration", {
    attributes: { "service.name": "user-service", "operation.type": "registration" }
  })
)
```

### Pattern 3: Error Handling with Tracing

```typescript
// Spans automatically capture error information
const riskyOperation = (input: string) => Effect.gen(function* () {
  yield* Effect.annotateCurrentSpan("input.value", input)
  
  if (input.length === 0) {
    yield* Effect.fail(new Error("Input cannot be empty"))
  }
  
  if (input.length > 100) {
    yield* Effect.fail(new Error("Input too long"))
  }
  
  return input.toUpperCase()
}).pipe(
  Effect.withSpan("risky-operation", {
    attributes: { "operation.type": "transformation" }
  }),
  Effect.catchAll((error) => Effect.gen(function* () {
    yield* Effect.annotateCurrentSpan("error.handled", true)
    yield* Effect.log("Operation failed, using fallback", { error: error.message })
    return "DEFAULT_VALUE"
  }))
)
```

## Real-World Examples

### Example 1: E-commerce Order Processing Pipeline

```typescript
import { Effect, Layer } from "effect"
import { NodeSdk } from "@effect/opentelemetry"
import { BatchSpanProcessor } from "@opentelemetry/sdk-trace-base"
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http"

// Domain models
interface Order {
  readonly id: string
  readonly userId: string
  readonly items: ReadonlyArray<OrderItem>
  readonly total: number
  readonly status: OrderStatus
}

interface OrderItem {
  readonly productId: string
  readonly quantity: number
  readonly price: number
}

type OrderStatus = "pending" | "confirmed" | "processing" | "shipped" | "delivered"

// Services with tracing
const OrderService = {
  validateOrder: (order: Order) => Effect.gen(function* () {
    yield* Effect.annotateCurrentSpan("order.id", order.id)
    yield* Effect.annotateCurrentSpan("order.item_count", order.items.length)
    yield* Effect.annotateCurrentSpan("order.total", order.total)
    
    // Validate order structure
    if (order.items.length === 0) {
      yield* Effect.fail(new Error("Order must contain at least one item"))
    }
    
    if (order.total <= 0) {
      yield* Effect.fail(new Error("Order total must be positive"))
    }
    
    // Validate inventory availability
    for (const item of order.items) {
      const available = yield* InventoryService.checkAvailability(item.productId, item.quantity)
      if (!available) {
        yield* Effect.fail(new Error(`Insufficient inventory for product ${item.productId}`))
      }
    }
    
    return order
  }).pipe(
    Effect.withSpan("validate-order", {
      kind: "internal",
      attributes: { "service.component": "order-validation" }
    })
  ),

  processPayment: (order: Order) => Effect.gen(function* () {
    yield* Effect.annotateCurrentSpan("payment.amount", order.total)
    yield* Effect.annotateCurrentSpan("payment.currency", "USD")
    
    // Simulate payment gateway call
    yield* Effect.sleep("150 millis")
    const paymentId = `payment_${Math.random().toString(36).substr(2, 9)}`
    
    yield* Effect.annotateCurrentSpan("payment.id", paymentId)
    yield* Effect.annotateCurrentSpan("payment.status", "completed")
    
    return { id: paymentId, amount: order.total, status: "completed" as const }
  }).pipe(
    Effect.withSpan("process-payment", {
      kind: "client",
      attributes: { 
        "service.component": "payment-gateway",
        "external.service": "stripe"
      }
    })
  ),

  updateInventory: (items: ReadonlyArray<OrderItem>) => Effect.gen(function* () {
    yield* Effect.annotateCurrentSpan("inventory.items_count", items.length)
    
    const results = yield* Effect.all(
      items.map(item => 
        InventoryService.decrementStock(item.productId, item.quantity).pipe(
          Effect.withSpan("decrement-stock", {
            attributes: { 
              "product.id": item.productId,
              "inventory.quantity": item.quantity
            }
          })
        )
      ),
      { concurrency: "inherit" }
    )
    
    yield* Effect.annotateCurrentSpan("inventory.updated_count", results.length)
    return results
  }).pipe(
    Effect.withSpan("update-inventory", {
      kind: "internal",
      attributes: { "service.component": "inventory-management" }
    })
  ),

  sendOrderConfirmation: (order: Order, userId: string) => Effect.gen(function* () {
    yield* Effect.annotateCurrentSpan("email.recipient_id", userId)
    yield* Effect.annotateCurrentSpan("email.template", "order-confirmation")
    
    const user = yield* UserService.getUser(userId)
    yield* Effect.annotateCurrentSpan("email.recipient", user.email)
    
    // Simulate email sending
    yield* Effect.sleep("50 millis")
    
    yield* Effect.annotateCurrentSpan("email.sent", true)
    yield* Effect.log("Order confirmation sent", { 
      orderId: order.id, 
      recipient: user.email 
    })
  }).pipe(
    Effect.withSpan("send-order-confirmation", {
      kind: "producer",
      attributes: { 
        "service.component": "email-service",
        "external.service": "sendgrid"
      }
    })
  )
}

// Supporting services
const InventoryService = {
  checkAvailability: (productId: string, quantity: number) => 
    Effect.succeed(true).pipe(
      Effect.delay("20 millis"),
      Effect.withSpan("check-inventory-availability", {
        attributes: { "product.id": productId, "quantity.requested": quantity }
      })
    ),

  decrementStock: (productId: string, quantity: number) =>
    Effect.succeed({ productId, newStock: 100 - quantity }).pipe(
      Effect.delay("30 millis"),
      Effect.withSpan("decrement-stock", {
        attributes: { "product.id": productId, "quantity.decremented": quantity }
      })
    )
}

const UserService = {
  getUser: (userId: string) =>
    Effect.succeed({ id: userId, email: `user${userId}@example.com` }).pipe(
      Effect.delay("25 millis"),
      Effect.withSpan("get-user", {
        attributes: { "user.id": userId }
      })
    )
}

// Main order processing workflow
const processOrder = (order: Order) => Effect.gen(function* () {
  yield* Effect.annotateCurrentSpan("order.id", order.id)
  yield* Effect.annotateCurrentSpan("service.name", "order-service")
  yield* Effect.log("Starting order processing", { orderId: order.id })
  
  // Step 1: Validate order
  const validatedOrder = yield* OrderService.validateOrder(order)
  
  // Step 2: Process payment
  const payment = yield* OrderService.processPayment(validatedOrder)
  yield* Effect.annotateCurrentSpan("payment.id", payment.id)
  
  // Step 3: Update inventory
  yield* OrderService.updateInventory(validatedOrder.items)
  
  // Step 4: Send confirmation
  yield* OrderService.sendOrderConfirmation(validatedOrder, validatedOrder.userId)
  
  const processedOrder = { ...validatedOrder, status: "confirmed" as const }
  yield* Effect.annotateCurrentSpan("order.final_status", processedOrder.status)
  
  return processedOrder
}).pipe(
  Effect.withSpan("process-order", {
    attributes: { 
      "service.name": "order-service",
      "operation.type": "order-processing",
      "service.version": "1.2.0"
    }
  })
)

// OpenTelemetry configuration
const TracingLayer = NodeSdk.layer(() => ({
  resource: { 
    serviceName: "ecommerce-order-service",
    serviceVersion: "1.2.0"
  },
  spanProcessor: new BatchSpanProcessor(
    new OTLPTraceExporter({
      url: "http://localhost:4318/v1/traces"
    })
  )
}))

// Example usage
const sampleOrder: Order = {
  id: "order_123",
  userId: "user_456",
  items: [
    { productId: "prod_789", quantity: 2, price: 29.99 },
    { productId: "prod_012", quantity: 1, price: 49.99 }
  ],
  total: 109.97,
  status: "pending"
}

// Run with tracing
const program = processOrder(sampleOrder).pipe(
  Effect.provide(TracingLayer),
  Effect.tapError(error => Effect.log("Order processing failed", { error: error.message }))
)
```

### Example 2: Microservices Communication with Distributed Tracing

```typescript
import { Effect, Context, Layer } from "effect"
import { HttpClient, HttpClientRequest } from "@effect/platform"

// Service interfaces
interface UserService {
  readonly getUser: (id: string) => Effect.Effect<User, UserNotFoundError>
  readonly updateUserPreferences: (id: string, prefs: UserPreferences) => Effect.Effect<void>
}

interface NotificationService {
  readonly sendEmail: (email: EmailRequest) => Effect.Effect<void>
  readonly sendPush: (push: PushRequest) => Effect.Effect<void>
}

interface AnalyticsService {
  readonly trackEvent: (event: AnalyticsEvent) => Effect.Effect<void>
}

// Domain models
interface User {
  readonly id: string
  readonly email: string
  readonly name: string
  readonly preferences: UserPreferences
}

interface UserPreferences {
  readonly emailNotifications: boolean
  readonly pushNotifications: boolean
  readonly theme: "light" | "dark"
}

// Error types
class UserNotFoundError extends Error {
  readonly _tag = "UserNotFoundError"
  constructor(userId: string) {
    super(`User not found: ${userId}`)
  }
}

// Service implementations with distributed tracing
const makeUserService = Effect.gen(function* () {
  const httpClient = yield* HttpClient.HttpClient

  const getUser = (id: string) => Effect.gen(function* () {
    yield* Effect.annotateCurrentSpan("user.id", id)
    yield* Effect.annotateCurrentSpan("service.target", "user-service")
    
    const request = HttpClientRequest.get(`/api/users/${id}`).pipe(
      HttpClientRequest.setHeader("X-Trace-Id", "current-trace-id"),
      HttpClientRequest.setHeader("Service-Name", "profile-service")
    )
    
    const response = yield* httpClient.execute(request).pipe(
      Effect.mapError(() => new UserNotFoundError(id)),
      Effect.withSpan("http-get-user", {
        kind: "client",
        attributes: {
          "http.method": "GET",
          "http.url": `/api/users/${id}`,
          "service.target": "user-service"
        }
      })
    )
    
    const user = yield* Effect.tryPromise(() => response.json() as Promise<User>)
    yield* Effect.annotateCurrentSpan("user.email", user.email)
    
    return user
  }).pipe(
    Effect.withSpan("get-user", {
      attributes: { "service.operation": "get-user" }
    })
  )

  const updateUserPreferences = (id: string, prefs: UserPreferences) => Effect.gen(function* () {
    yield* Effect.annotateCurrentSpan("user.id", id)
    yield* Effect.annotateCurrentSpan("preferences.email_enabled", prefs.emailNotifications)
    yield* Effect.annotateCurrentSpan("preferences.push_enabled", prefs.pushNotifications)
    
    const request = HttpClientRequest.put(`/api/users/${id}/preferences`).pipe(
      HttpClientRequest.setBody(JSON.stringify(prefs)),
      HttpClientRequest.setHeader("Content-Type", "application/json")
    )
    
    yield* httpClient.execute(request).pipe(
      Effect.withSpan("http-update-preferences", {
        kind: "client",
        attributes: {
          "http.method": "PUT",
          "http.url": `/api/users/${id}/preferences`,
          "service.target": "user-service"
        }
      })
    )
  }).pipe(
    Effect.withSpan("update-user-preferences", {
      attributes: { "service.operation": "update-preferences" }
    })
  )

  return { getUser, updateUserPreferences } as const satisfies UserService
})

const makeNotificationService = Effect.gen(function* () {
  const httpClient = yield* HttpClient.HttpClient

  const sendEmail = (emailRequest: EmailRequest) => Effect.gen(function* () {
    yield* Effect.annotateCurrentSpan("email.recipient", emailRequest.to)
    yield* Effect.annotateCurrentSpan("email.template", emailRequest.template)
    
    yield* Effect.sleep("100 millis") // Simulate email processing
    yield* Effect.log("Email sent", { recipient: emailRequest.to })
  }).pipe(
    Effect.withSpan("send-email", {
      kind: "producer",
      attributes: {
        "service.target": "email-service",
        "email.provider": "sendgrid"
      }
    })
  )

  const sendPush = (pushRequest: PushRequest) => Effect.gen(function* () {
    yield* Effect.annotateCurrentSpan("push.user_id", pushRequest.userId)
    yield* Effect.annotateCurrentSpan("push.message", pushRequest.message)
    
    yield* Effect.sleep("50 millis") // Simulate push processing
    yield* Effect.log("Push notification sent", { userId: pushRequest.userId })
  }).pipe(
    Effect.withSpan("send-push", {
      kind: "producer",
      attributes: {
        "service.target": "push-service",
        "push.provider": "firebase"
      }
    })
  )

  return { sendEmail, sendPush } as const satisfies NotificationService
})

const makeAnalyticsService = Effect.gen(function* () {
  const trackEvent = (event: AnalyticsEvent) => Effect.gen(function* () {
    yield* Effect.annotateCurrentSpan("analytics.event_type", event.type)
    yield* Effect.annotateCurrentSpan("analytics.user_id", event.userId)
    
    yield* Effect.sleep("20 millis") // Simulate analytics processing
    yield* Effect.log("Analytics event tracked", { eventType: event.type })
  }).pipe(
    Effect.withSpan("track-event", {
      kind: "producer",
      attributes: {
        "service.target": "analytics-service",
        "analytics.provider": "mixpanel"
      }
    })
  )

  return { trackEvent } as const satisfies AnalyticsService
})

// Main business logic with cross-service tracing
const updateUserProfilePreferences = (
  userId: string, 
  newPreferences: UserPreferences
) => Effect.gen(function* () {
  yield* Effect.annotateCurrentSpan("user.id", userId)
  yield* Effect.annotateCurrentSpan("operation.type", "profile-update")
  
  const userService = yield* makeUserService
  const notificationService = yield* makeNotificationService  
  const analyticsService = yield* makeAnalyticsService
  
  // Get current user
  const user = yield* userService.getUser(userId)
  yield* Effect.annotateCurrentSpan("user.email", user.email)
  
  // Update preferences
  yield* userService.updateUserPreferences(userId, newPreferences)
  
  // Send notifications based on new preferences
  const notifications = []
  if (newPreferences.emailNotifications) {
    notifications.push(
      notificationService.sendEmail({
        to: user.email,
        template: "preferences-updated",
        data: { userName: user.name }
      })
    )
  }
  
  if (newPreferences.pushNotifications) {
    notifications.push(
      notificationService.sendPush({
        userId: user.id,
        message: "Your preferences have been updated"
      })
    )
  }
  
  // Send notifications concurrently
  if (notifications.length > 0) {
    yield* Effect.all(notifications, { concurrency: "inherit" }).pipe(
      Effect.withSpan("send-notifications", {
        attributes: { "notifications.count": notifications.length }
      })
    )
  }
  
  // Track analytics event
  yield* analyticsService.trackEvent({
    type: "preferences_updated",
    userId: user.id,
    properties: {
      emailEnabled: newPreferences.emailNotifications,
      pushEnabled: newPreferences.pushNotifications,
      theme: newPreferences.theme
    }
  })
  
  return { success: true, user: { ...user, preferences: newPreferences } }
}).pipe(
  Effect.withSpan("update-user-profile-preferences", {
    attributes: {
      "service.name": "profile-service",
      "operation.type": "preference-update",
      "service.version": "2.1.0"
    }
  })
)
```

### Example 3: Performance Monitoring and Custom Metrics

```typescript
import { Effect, Metric, Random } from "effect"

// Custom metrics for performance monitoring
const DatabaseQueryDuration = Metric.histogram(
  "database_query_duration_seconds",
  "Duration of database queries in seconds"
)

const CacheHitCounter = Metric.counter(
  "cache_hits_total", 
  "Total number of cache hits"
)

const CacheMissCounter = Metric.counter(
  "cache_misses_total",
  "Total number of cache misses"
)

const ActiveConnectionsGauge = Metric.gauge(
  "active_connections",
  "Number of active database connections"
)

// Performance-monitored data access layer
const DataService = {
  findUserById: (id: string) => Effect.gen(function* () {
    yield* Effect.annotateCurrentSpan("user.id", id)
    yield* Effect.annotateCurrentSpan("operation.type", "database-query")
    
    const startTime = yield* Effect.sync(() => performance.now())
    
    // Simulate cache check
    const cacheHit = yield* Random.nextBoolean
    
    if (cacheHit) {
      yield* CacheHitCounter.increment()
      yield* Effect.annotateCurrentSpan("cache.hit", true)
      yield* Effect.sleep("5 millis") // Fast cache response
      
      const user = { id, name: `User ${id}`, email: `user${id}@example.com` }
      return user
    } else {
      yield* CacheMissCounter.increment()
      yield* Effect.annotateCurrentSpan("cache.hit", false)
      
      // Simulate database query
      const queryDuration = yield* Random.nextIntBetween(50, 200)
      yield* Effect.sleep(`${queryDuration} millis`)
      
      const endTime = yield* Effect.sync(() => performance.now())
      const durationSeconds = (endTime - startTime) / 1000
      
      yield* DatabaseQueryDuration.update(durationSeconds)
      yield* Effect.annotateCurrentSpan("db.query_duration_ms", queryDuration)
      yield* Effect.annotateCurrentSpan("db.table", "users")
      yield* Effect.annotateCurrentSpan("db.operation", "SELECT")
      
      const user = { id, name: `User ${id}`, email: `user${id}@example.com` }
      return user
    }
  }).pipe(
    Effect.withSpan("find-user-by-id", {
      kind: "internal",
      attributes: {
        "service.component": "data-layer",
        "db.system": "postgresql"
      }
    })
  ),

  updateConnectionCount: (delta: number) => Effect.gen(function* () {
    yield* ActiveConnectionsGauge.increment(delta)
    yield* Effect.annotateCurrentSpan("connections.delta", delta)
  }).pipe(
    Effect.withSpan("update-connection-count", {
      attributes: { "service.component": "connection-pool" }
    })
  )
}

// Service with performance monitoring
const createPerformanceMonitoredService = Effect.gen(function* () {
  // Initialize connection pool
  yield* DataService.updateConnectionCount(10) // Starting with 10 connections
  
  const processUserRequest = (userId: string) => Effect.gen(function* () {
    yield* Effect.annotateCurrentSpan("user.id", userId)
    yield* Effect.annotateCurrentSpan("service.component", "user-request-handler")
    
    // Add connection tracking
    yield* DataService.updateConnectionCount(1) // Acquire connection
    
    const user = yield* DataService.findUserById(userId)
    yield* Effect.annotateCurrentSpan("user.found", !!user)
    
    // Simulate some processing time
    const processingTime = yield* Random.nextIntBetween(10, 50)
    yield* Effect.sleep(`${processingTime} millis`)
    yield* Effect.annotateCurrentSpan("processing.time_ms", processingTime)
    
    yield* DataService.updateConnectionCount(-1) // Release connection
    
    return { user, processedAt: new Date() }
  }).pipe(
    Effect.withSpan("process-user-request", {
      attributes: {
        "service.name": "user-service",
        "operation.type": "user-request"
      }
    }),
    Effect.ensuring(
      DataService.updateConnectionCount(-1) // Ensure connection is always released
    )
  )

  return { processUserRequest }
})

// Load testing simulation with tracing
const simulateLoad = (userCount: number, requestsPerUser: number) => Effect.gen(function* () {
  yield* Effect.annotateCurrentSpan("load_test.user_count", userCount)
  yield* Effect.annotateCurrentSpan("load_test.requests_per_user", requestsPerUser)
  
  const service = yield* createPerformanceMonitoredService
  
  const users = Array.from({ length: userCount }, (_, i) => `user_${i + 1}`)
  
  const allRequests = users.flatMap(userId => 
    Array.from({ length: requestsPerUser }, (_, requestIndex) =>
      service.processUserRequest(userId).pipe(
        Effect.withSpan(`user-request-${requestIndex + 1}`, {
          attributes: {
            "request.user_id": userId,
            "request.sequence": requestIndex + 1
          }
        })
      )
    )
  )
  
  const startTime = yield* Effect.sync(() => performance.now())
  
  const results = yield* Effect.all(allRequests, { 
    concurrency: 5 // Limit concurrent requests
  }).pipe(
    Effect.withSpan("execute-all-requests", {
      attributes: { 
        "requests.total_count": allRequests.length,
        "requests.concurrency": 5
      }
    })
  )
  
  const endTime = yield* Effect.sync(() => performance.now())
  const totalDuration = endTime - startTime
  
  yield* Effect.annotateCurrentSpan("load_test.duration_ms", totalDuration)
  yield* Effect.annotateCurrentSpan("load_test.requests_completed", results.length)
  yield* Effect.annotateCurrentSpan("load_test.requests_per_second", 
    (results.length / totalDuration) * 1000
  )
  
  return {
    totalRequests: results.length,
    durationMs: totalDuration,
    requestsPerSecond: (results.length / totalDuration) * 1000
  }
}).pipe(
  Effect.withSpan("simulate-load-test", {
    attributes: {
      "test.type": "load-test",
      "service.name": "user-service"
    }
  })
)
```

## Advanced Features Deep Dive

### Span Links and Cross-Trace Relationships

Span links allow you to create relationships between spans that exist in different traces, enabling complex distributed scenarios.

```typescript
import { Effect, Tracer } from "effect"

// Creating span links for batch processing scenarios
const processBatchWithLinks = (batchId: string, items: ReadonlyArray<BatchItem>) => Effect.gen(function* () {
  yield* Effect.annotateCurrentSpan("batch.id", batchId)
  yield* Effect.annotateCurrentSpan("batch.size", items.length)
  
  const itemProcessingPromises = items.map((item, index) =>
    processIndividualItem(item).pipe(
      Effect.withSpan(`process-item-${index}`, {
        links: [
          // Link back to the batch processing span
          {
            _tag: "SpanLink" as const,
            span: yield* Effect.serviceOption(Tracer.ParentSpan),
            attributes: { 
              "link.type": "batch-parent",
              "batch.item_index": index 
            }
          }
        ]
      })
    )
  )
  
  return yield* Effect.all(itemProcessingPromises, { concurrency: 3 })
}).pipe(
  Effect.withSpan("process-batch", {
    attributes: { "processing.type": "batch" }
  })
)

// Cross-service span linking
const handleWebhookWithTracking = (webhookData: WebhookData) => Effect.gen(function* () {
  // Extract trace context from webhook headers if available
  const incomingTraceId = webhookData.headers["x-trace-id"]
  const incomingSpanId = webhookData.headers["x-span-id"]
  
  if (incomingTraceId && incomingSpanId) {
    const externalSpan = Tracer.externalSpan({
      traceId: incomingTraceId,
      spanId: incomingSpanId,
      sampled: true
    })
    
    yield* Effect.annotateCurrentSpan("webhook.source_trace_id", incomingTraceId)
    yield* Effect.annotateCurrentSpan("webhook.source_span_id", incomingSpanId)
  }
  
  // Process webhook with context from originating service
  return yield* processWebhookPayload(webhookData.payload)
}).pipe(
  Effect.withSpan("handle-webhook", {
    kind: "server",
    attributes: { 
      "webhook.source": webhookData.source,
      "webhook.event_type": webhookData.eventType
    }
  })
)
```

### Custom Tracer Implementation

```typescript
import { Effect, Context, Layer, Tracer, Option } from "effect"

// Custom tracer for specialized logging
interface CustomTracingContext {
  readonly sessionId: string
  readonly userId?: string
  readonly requestId: string
}

const CustomTracingContext = Context.GenericTag<CustomTracingContext>("CustomTracingContext")

const makeCustomTracer = Effect.gen(function* () {
  const context = yield* CustomTracingContext
  
  const customSpan = (
    name: string,
    parent: Option.Option<Tracer.AnySpan>,
    spanContext: Context.Context<never>,
    links: ReadonlyArray<Tracer.SpanLink>,
    startTime: bigint,
    kind: Tracer.SpanKind
  ): Tracer.Span => {
    const span = new CustomSpan(name, parent, spanContext, links, startTime, kind, context)
    return span
  }
  
  const customContext = <X>(f: () => X, fiber: any): X => {
    // Add custom context propagation logic
    return f()
  }
  
  return Tracer.make({
    span: customSpan,
    context: customContext
  })
})

class CustomSpan implements Tracer.Span {
  readonly _tag = "Span"
  readonly spanId: string
  readonly traceId: string
  readonly sampled = true
  
  status: Tracer.SpanStatus
  attributes: Map<string, unknown>
  events: Array<[string, bigint, Record<string, unknown>]> = []
  links: Array<Tracer.SpanLink>
  
  constructor(
    readonly name: string,
    readonly parent: Option.Option<Tracer.AnySpan>,
    readonly context: Context.Context<never>,
    links: Iterable<Tracer.SpanLink>,
    readonly startTime: bigint,
    readonly kind: Tracer.SpanKind,
    private readonly tracingContext: CustomTracingContext
  ) {
    this.status = { _tag: "Started", startTime }
    this.attributes = new Map()
    this.links = Array.from(links)
    
    // Generate IDs
    this.spanId = this.generateSpanId()
    this.traceId = parent._tag === "Some" ? parent.value.traceId : this.generateTraceId()
    
    // Add custom context attributes
    this.attributes.set("session.id", this.tracingContext.sessionId)
    this.attributes.set("request.id", this.tracingContext.requestId)
    if (this.tracingContext.userId) {
      this.attributes.set("user.id", this.tracingContext.userId)
    }
  }
  
  end(endTime: bigint, exit: any): void {
    this.status = {
      _tag: "Ended",
      endTime,
      exit,
      startTime: this.status.startTime
    }
    
    // Custom logging with enriched context
    console.log(`[TRACE] ${this.name}`, {
      spanId: this.spanId,
      traceId: this.traceId,
      sessionId: this.tracingContext.sessionId,
      requestId: this.tracingContext.requestId,
      userId: this.tracingContext.userId,
      duration: Number(endTime - this.status.startTime) / 1000000, // Convert to ms
      attributes: Object.fromEntries(this.attributes),
      events: this.events,
      status: this.status
    })
  }
  
  attribute(key: string, value: unknown): void {
    this.attributes.set(key, value)
  }
  
  event(name: string, startTime: bigint, attributes?: Record<string, unknown>): void {
    this.events.push([name, startTime, attributes ?? {}])
  }
  
  addLinks(links: ReadonlyArray<Tracer.SpanLink>): void {
    this.links.push(...links)
  }
  
  private generateSpanId(): string {
    return Math.random().toString(16).substring(2, 18)
  }
  
  private generateTraceId(): string {
    return Math.random().toString(16).substring(2, 34)
  }
}

const CustomTracerLayer = Layer.effect(
  Tracer.Tracer,
  makeCustomTracer
)

// Usage with custom tracer
const tracedApplication = (sessionId: string, requestId: string, userId?: string) => 
  Effect.gen(function* () {
    yield* Effect.log("Application started")
    
    const result = yield* someBusinessLogic().pipe(
      Effect.withSpan("business-operation")
    )
    
    return result
  }).pipe(
    Effect.provide(CustomTracerLayer),
    Effect.provideService(CustomTracingContext, { sessionId, requestId, userId })
  )
```

### Conditional Tracing and Sampling

```typescript
// Conditional tracing based on configuration or context
const ConditionalTracingConfig = Context.GenericTag<{
  readonly enabled: boolean
  readonly sampleRate: number
  readonly debugMode: boolean
}>("ConditionalTracingConfig")

const conditionalSpan = <A, E, R>(
  name: string, 
  effect: Effect.Effect<A, E, R>,
  options?: Tracer.SpanOptions
) => Effect.gen(function* () {
  const config = yield* ConditionalTracingConfig
  
  if (!config.enabled) {
    return yield* effect
  }
  
  // Sample based on rate
  const shouldSample = Math.random() < config.sampleRate
  if (!shouldSample && !config.debugMode) {
    return yield* effect
  }
  
  return yield* effect.pipe(
    Effect.withSpan(name, {
      ...options,
      attributes: {
        ...options?.attributes,
        "sampling.rate": config.sampleRate,
        "debug.mode": config.debugMode
      }
    })
  )
})

// Environment-aware tracing
const ProductionTracingLayer = Layer.succeed(ConditionalTracingConfig, {
  enabled: true,
  sampleRate: 0.1, // 10% sampling in production
  debugMode: false
})

const DevelopmentTracingLayer = Layer.succeed(ConditionalTracingConfig, {
  enabled: true,
  sampleRate: 1.0, // 100% sampling in development
  debugMode: true
})

const TracingLayer = process.env.NODE_ENV === "production" 
  ? ProductionTracingLayer 
  : DevelopmentTracingLayer
```

## Practical Patterns & Best Practices

### Pattern 1: Reusable Tracing Utilities

```typescript
// Utility functions for common tracing patterns
const TracingUtils = {
  // Time an operation with automatic span creation
  timed: <A, E, R>(
    name: string,
    operation: Effect.Effect<A, E, R>,
    options?: Tracer.SpanOptions
  ) => Effect.gen(function* () {
    const startTime = yield* Effect.sync(() => performance.now())
    
    const result = yield* operation.pipe(
      Effect.withSpan(name, options)
    )
    
    const endTime = yield* Effect.sync(() => performance.now())
    const duration = endTime - startTime
    
    yield* Effect.annotateCurrentSpan("operation.duration_ms", duration)
    
    return result
  }),

  // Add standard service metadata to spans
  withServiceInfo: <A, E, R>(
    serviceName: string,
    version: string,
    operation: Effect.Effect<A, E, R>
  ) => operation.pipe(
    Effect.annotateSpans({
      "service.name": serviceName,
      "service.version": version,
      "telemetry.sdk.name": "effect",
      "telemetry.sdk.language": "typescript"
    })
  ),

  // Batch multiple operations with individual span tracking
  batch: <A, E, R>(
    name: string,
    operations: ReadonlyArray<Effect.Effect<A, E, R>>,
    options?: { concurrency?: number }
  ) => Effect.gen(function* () {
    yield* Effect.annotateCurrentSpan("batch.size", operations.length)
    yield* Effect.annotateCurrentSpan("batch.concurrency", options?.concurrency ?? "unbounded")
    
    const results = yield* Effect.all(
      operations.map((op, index) => 
        op.pipe(
          Effect.withSpan(`${name}-item-${index}`, {
            attributes: { "batch.item_index": index }
          })
        )
      ),
      { concurrency: options?.concurrency ?? "inherit" }
    )
    
    yield* Effect.annotateCurrentSpan("batch.completed_count", results.length)
    return results
  }).pipe(
    Effect.withSpan(`batch-${name}`, {
      attributes: { "operation.type": "batch" }
    })
  ),

  // Retry with tracing
  retryWithTracing: <A, E, R>(
    operation: Effect.Effect<A, E, R>,
    maxRetries: number = 3
  ) => {
    const attempt = (attemptNumber: number): Effect.Effect<A, E, R> =>
      operation.pipe(
        Effect.withSpan(`attempt-${attemptNumber}`, {
          attributes: { 
            "retry.attempt": attemptNumber,
            "retry.max_attempts": maxRetries 
          }
        }),
        Effect.catchAll((error) => {
          if (attemptNumber >= maxRetries) {
            return Effect.fail(error)
          }
          
          return Effect.gen(function* () {
            yield* Effect.annotateCurrentSpan("retry.will_retry", true)
            yield* Effect.sleep(`${attemptNumber * 100} millis`) // Exponential backoff
            return yield* attempt(attemptNumber + 1)
          })
        })
      )
    
    return attempt(1).pipe(
      Effect.withSpan("retry-operation", {
        attributes: { "operation.type": "retry" }
      })
    )
  }
}

// Usage examples
const exampleService = {
  fetchData: (id: string) => TracingUtils.timed(
    "fetch-data",
    Effect.gen(function* () {
      yield* Effect.sleep("100 millis")
      return { id, data: "sample" }
    }),
    { attributes: { "data.id": id } }
  ),

  processMultipleItems: (items: string[]) => TracingUtils.batch(
    "process-item",
    items.map(item => 
      Effect.gen(function* () {
        yield* Effect.sleep("50 millis")
        return `processed-${item}`
      })
    ),
    { concurrency: 3 }
  ).pipe(
    TracingUtils.withServiceInfo("data-service", "1.0.0")
  ),

  reliableFetch: (url: string) => TracingUtils.retryWithTracing(
    Effect.gen(function* () {
      // Simulate unreliable network call
      const success = yield* Effect.sync(() => Math.random() > 0.7)
      if (!success) {
        yield* Effect.fail(new Error("Network error"))
      }
      return { url, data: "fetched" }
    })
  )
}
```

### Pattern 2: Tracing Middleware for HTTP Services

```typescript
import { Effect, Layer } from "effect"

// HTTP request tracing middleware
const createTracingMiddleware = <R>(
  serviceName: string
) => {
  return <A, E>(
    handler: (request: HttpRequest) => Effect.Effect<A, E, R>
  ) => (request: HttpRequest) => Effect.gen(function* () {
    const traceId = request.headers["x-trace-id"] || generateTraceId()
    const spanId = request.headers["x-span-id"] || generateSpanId()
    const userAgent = request.headers["user-agent"] || "unknown"
    
    yield* Effect.annotateCurrentSpan("http.method", request.method)
    yield* Effect.annotateCurrentSpan("http.url", request.url)
    yield* Effect.annotateCurrentSpan("http.user_agent", userAgent)
    yield* Effect.annotateCurrentSpan("trace.id", traceId)
    yield* Effect.annotateCurrentSpan("span.id", spanId)
    
    if (request.headers["x-user-id"]) {
      yield* Effect.annotateCurrentSpan("user.id", request.headers["x-user-id"])
    }
    
    const startTime = yield* Effect.sync(() => performance.now())
    
    const result = yield* handler(request).pipe(
      Effect.tapError(error => 
        Effect.annotateCurrentSpan("http.error", error.message)
      ),
      Effect.tap(() => 
        Effect.annotateCurrentSpan("http.status", "success")
      )
    )
    
    const endTime = yield* Effect.sync(() => performance.now())
    yield* Effect.annotateCurrentSpan("http.duration_ms", endTime - startTime)
    
    return result
  }).pipe(
    Effect.withSpan(`${request.method} ${request.path}`, {
      kind: "server",
      attributes: {
        "service.name": serviceName,
        "http.route": request.path,
        "http.scheme": request.protocol
      }
    })
  )
}

// Database query tracing middleware  
const createDbTracingMiddleware = <A, E, R>(
  query: (sql: string, params?: any[]) => Effect.Effect<A, E, R>
) => (sql: string, params?: any[]) => Effect.gen(function* () {
  yield* Effect.annotateCurrentSpan("db.statement", sql)
  yield* Effect.annotateCurrentSpan("db.system", "postgresql")
  
  if (params) {
    yield* Effect.annotateCurrentSpan("db.params_count", params.length)
  }
  
  const result = yield* query(sql, params).pipe(
    Effect.tapError(error => 
      Effect.annotateCurrentSpan("db.error", error.message)
    )
  )
  
  yield* Effect.annotateCurrentSpan("db.success", true)
  
  return result
}).pipe(
  Effect.withSpan("db-query", {
    kind: "client",
    attributes: { "service.component": "database" }
  })
)
```

### Pattern 3: Structured Error Tracing

```typescript
// Enhanced error types with tracing context
abstract class TracedError extends Error {
  abstract readonly _tag: string
  readonly traceId?: string
  readonly spanId?: string
  readonly timestamp: Date = new Date()
  
  constructor(message: string, cause?: Error) {
    super(message)
    this.name = this._tag
    if (cause) {
      this.cause = cause
    }
  }
}

class ValidationError extends TracedError {
  readonly _tag = "ValidationError"
  
  constructor(
    public readonly field: string,
    public readonly value: unknown,
    message?: string
  ) {
    super(message || `Validation failed for field: ${field}`)
  }
}

class DatabaseError extends TracedError {
  readonly _tag = "DatabaseError"
  
  constructor(
    public readonly operation: string,
    public readonly table?: string,
    message?: string,
    cause?: Error
  ) {
    super(message || `Database operation failed: ${operation}`, cause)
  }
}

class NetworkError extends TracedError {
  readonly _tag = "NetworkError"
  
  constructor(
    public readonly url: string,
    public readonly statusCode?: number,
    message?: string,
    cause?: Error
  ) {
    super(message || `Network request failed: ${url}`, cause)
  }
}

// Error handling with automatic tracing
const handleTracedError = <E extends TracedError>(error: E) => Effect.gen(function* () {
  yield* Effect.annotateCurrentSpan("error.type", error._tag)
  yield* Effect.annotateCurrentSpan("error.message", error.message)
  yield* Effect.annotateCurrentSpan("error.timestamp", error.timestamp.toISOString())
  yield* Effect.annotateCurrentSpan("error.handled", true)
  
  // Add specific error context
  switch (error._tag) {
    case "ValidationError":
      yield* Effect.annotateCurrentSpan("validation.field", error.field)
      yield* Effect.annotateCurrentSpan("validation.value", String(error.value))
      break
    case "DatabaseError":
      yield* Effect.annotateCurrentSpan("db.operation", error.operation)
      if (error.table) {
        yield* Effect.annotateCurrentSpan("db.table", error.table)
      }
      break
    case "NetworkError":
      yield* Effect.annotateCurrentSpan("http.url", error.url)
      if (error.statusCode) {
        yield* Effect.annotateCurrentSpan("http.status_code", error.statusCode)
      }
      break
  }
  
  yield* Effect.log("Error occurred", {
    type: error._tag,
    message: error.message,
    traceId: error.traceId,
    spanId: error.spanId
  })
})

// Wrapping operations with structured error tracing
const withErrorTracing = <A, E extends TracedError, R>(
  operation: Effect.Effect<A, E, R>
) => operation.pipe(
  Effect.tapError(handleTracedError),
  Effect.sandbox,
  Effect.mapError(cause => {
    // Enhance error with current trace context if available
    return cause
  }),
  Effect.unsandbox
)
```

## Integration Examples

### Integration with OpenTelemetry and Observability Platforms

```typescript
import { NodeSdk } from "@effect/opentelemetry"
import { BatchSpanProcessor, ConsoleSpanExporter } from "@opentelemetry/sdk-trace-base"
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http"
import { JaegerExporter } from "@opentelemetry/exporter-jaeger"
import { Resource } from "@opentelemetry/resources"
import { SEMRESATTRS_SERVICE_NAME, SEMRESATTRS_SERVICE_VERSION } from "@opentelemetry/semantic-conventions"

// Multi-exporter setup for different environments
const createTracingLayer = (environment: "development" | "staging" | "production") => {
  const baseConfig = {
    resource: {
      serviceName: "my-effect-app",
      serviceVersion: "1.0.0",
      attributes: {
        [SEMRESATTRS_SERVICE_NAME]: "my-effect-app",
        [SEMRESATTRS_SERVICE_VERSION]: "1.0.0",
        "deployment.environment": environment
      }
    }
  }

  switch (environment) {
    case "development":
      return NodeSdk.layer(() => ({
        ...baseConfig,
        spanProcessor: new BatchSpanProcessor(new ConsoleSpanExporter())
      }))
    
    case "staging":
      return NodeSdk.layer(() => ({
        ...baseConfig,
        spanProcessor: new BatchSpanProcessor(
          new OTLPTraceExporter({
            url: "http://staging-collector:4318/v1/traces"
          })
        )
      }))
    
    case "production":
      return NodeSdk.layer(() => ({
        ...baseConfig,
        spanProcessor: new BatchSpanProcessor(
          new JaegerExporter({
            endpoint: "http://jaeger-collector:14268/api/traces"
          })
        )
      }))
  }
}

// Datadog integration
const DatadogTracingLayer = NodeSdk.layer(() => ({
  resource: { serviceName: "my-effect-app" },
  spanProcessor: new BatchSpanProcessor(
    new OTLPTraceExporter({
      url: "https://trace.agent.datadoghq.com/v0.4/traces",
      headers: {
        "DD-API-KEY": process.env.DATADOG_API_KEY!
      }
    })
  )
}))

// New Relic integration
const NewRelicTracingLayer = NodeSdk.layer(() => ({
  resource: { serviceName: "my-effect-app" },
  spanProcessor: new BatchSpanProcessor(
    new OTLPTraceExporter({
      url: "https://otlp.nr-data.net/v1/traces",
      headers: {
        "api-key": process.env.NEW_RELIC_LICENSE_KEY!
      }
    })
  )
}))

// Honeycomb integration
const HoneycombTracingLayer = NodeSdk.layer(() => ({
  resource: { serviceName: "my-effect-app" },
  spanProcessor: new BatchSpanProcessor(
    new OTLPTraceExporter({
      url: "https://api.honeycomb.io/v1/traces",
      headers: {
        "x-honeycomb-team": process.env.HONEYCOMB_API_KEY!,
        "x-honeycomb-dataset": "my-effect-app"
      }
    })
  )
}))

// Environment-aware layer selection
const TracingLayer = (() => {
  const env = process.env.NODE_ENV as "development" | "staging" | "production"
  const provider = process.env.TRACING_PROVIDER || "console"
  
  if (provider === "datadog") return DatadogTracingLayer
  if (provider === "newrelic") return NewRelicTracingLayer  
  if (provider === "honeycomb") return HoneycombTracingLayer
  
  return createTracingLayer(env)
})()
```

### Testing Strategies with Tracing

```typescript
import { Effect, TestContext, Layer } from "effect"
import { NodeSdk } from "@effect/opentelemetry"
import { InMemorySpanExporter, SimpleSpanProcessor } from "@opentelemetry/sdk-trace-base"

// In-memory tracing for tests
const createTestTracingLayer = () => {
  const spanExporter = new InMemorySpanExporter()
  
  const layer = NodeSdk.layer(() => ({
    resource: { serviceName: "test-service" },
    spanProcessor: new SimpleSpanProcessor(spanExporter)
  }))
  
  const getSpans = () => spanExporter.getFinishedSpans()
  const clearSpans = () => spanExporter.reset()
  
  return { layer, getSpans, clearSpans }
}

// Test utilities for span verification
const SpanTestUtils = {
  findSpanByName: (spans: any[], name: string) => 
    spans.find(span => span.name === name),
    
  assertSpanExists: (spans: any[], name: string) => {
    const span = SpanTestUtils.findSpanByName(spans, name)
    if (!span) {
      throw new Error(`Expected span '${name}' not found`)
    }
    return span
  },
  
  assertSpanAttribute: (span: any, key: string, expectedValue: any) => {
    const actualValue = span.attributes[key]
    if (actualValue !== expectedValue) {
      throw new Error(
        `Expected span attribute '${key}' to be '${expectedValue}', got '${actualValue}'`
      )
    }
  },
  
  assertSpanDuration: (span: any, minMs: number, maxMs: number) => {
    const durationMs = (span.endTime[0] - span.startTime[0]) * 1000 + 
                      (span.endTime[1] - span.startTime[1]) / 1000000
    if (durationMs < minMs || durationMs > maxMs) {
      throw new Error(
        `Expected span duration to be between ${minMs}ms and ${maxMs}ms, got ${durationMs}ms`
      )
    }
  }
}

// Example test cases
describe("Tracing Tests", () => {
  let testTracing: ReturnType<typeof createTestTracingLayer>
  
  beforeEach(() => {
    testTracing = createTestTracingLayer()
  })
  
  afterEach(() => {
    testTracing.clearSpans()
  })
  
  it("should create spans for service operations", async () => {
    const testService = {
      processData: (data: string) => Effect.gen(function* () {
        yield* Effect.sleep("100 millis")
        return data.toUpperCase()
      }).pipe(
        Effect.withSpan("process-data", {
          attributes: { "data.length": data.length }
        })
      )
    }
    
    const program = testService.processData("hello")
      .pipe(Effect.provide(testTracing.layer))
    
    const result = await Effect.runPromise(program)
    
    expect(result).toBe("HELLO")
    
    const spans = testTracing.getSpans()
    expect(spans).toHaveLength(1)
    
    const span = SpanTestUtils.assertSpanExists(spans, "process-data")
    SpanTestUtils.assertSpanAttribute(span, "data.length", 5)
    SpanTestUtils.assertSpanDuration(span, 90, 120)
  })
  
  it("should trace nested operations correctly", async () => {
    const nestedService = {
      parentOperation: () => Effect.gen(function* () {
        yield* Effect.sleep("50 millis")
        return yield* nestedService.childOperation()
      }).pipe(Effect.withSpan("parent-operation")),
      
      childOperation: () => Effect.gen(function* () {
        yield* Effect.sleep("30 millis")
        return "child result"
      }).pipe(Effect.withSpan("child-operation"))
    }
    
    const program = nestedService.parentOperation()
      .pipe(Effect.provide(testTracing.layer))
    
    await Effect.runPromise(program)
    
    const spans = testTracing.getSpans()
    expect(spans).toHaveLength(2)
    
    const parentSpan = SpanTestUtils.assertSpanExists(spans, "parent-operation")
    const childSpan = SpanTestUtils.assertSpanExists(spans, "child-operation")
    
    // Verify parent-child relationship
    expect(childSpan.parentSpanId).toBe(parentSpan.spanContext().spanId)
  })
  
  it("should capture errors in spans", async () => {
    const errorService = {
      failingOperation: () => Effect.gen(function* () {
        yield* Effect.sleep("25 millis")
        yield* Effect.fail(new Error("Something went wrong"))
      }).pipe(Effect.withSpan("failing-operation"))
    }
    
    const program = errorService.failingOperation()
      .pipe(
        Effect.provide(testTracing.layer),
        Effect.catchAll(() => Effect.void) // Handle error for test
      )
    
    await Effect.runPromise(program)
    
    const spans = testTracing.getSpans()
    const span = SpanTestUtils.assertSpanExists(spans, "failing-operation")
    
    expect(span.status.code).toBe(2) // ERROR status
    expect(span.status.message).toBe("Something went wrong")
  })
})
```

### Integration with Custom Observability Solutions

```typescript
// Custom observability platform integration
interface CustomObservabilityClient {
  readonly sendTrace: (trace: CustomTrace) => Effect.Effect<void>
  readonly sendMetrics: (metrics: CustomMetrics) => Effect.Effect<void>
}

interface CustomTrace {
  readonly traceId: string
  readonly spans: ReadonlyArray<CustomSpan>
  readonly service: string
  readonly timestamp: number
}

interface CustomSpan {
  readonly spanId: string
  readonly parentSpanId?: string
  readonly operationName: string
  readonly startTime: number
  readonly duration: number
  readonly tags: Record<string, any>
  readonly logs: ReadonlyArray<CustomSpanLog>
}

interface CustomSpanLog {
  readonly timestamp: number
  readonly level: "info" | "warn" | "error"
  readonly message: string
  readonly fields?: Record<string, any>
}

const createCustomObservabilityIntegration = (client: CustomObservabilityClient) => {
  const CustomSpanProcessor = class {
    private spans: CustomSpan[] = []
    
    onStart(span: any, parentContext: any): void {
      // Track span start
    }
    
    onEnd(span: any): void {
      const customSpan: CustomSpan = {
        spanId: span.spanContext().spanId,
        parentSpanId: span.parentSpanId,
        operationName: span.name,
        startTime: span.startTime[0] * 1000 + span.startTime[1] / 1000000,
        duration: (span.endTime[0] - span.startTime[0]) * 1000 + 
                  (span.endTime[1] - span.startTime[1]) / 1000000,
        tags: span.attributes,
        logs: span.events.map((event: any) => ({
          timestamp: event.time[0] * 1000 + event.time[1] / 1000000,
          level: event.attributes?.level || "info",
          message: event.name,
          fields: event.attributes
        }))
      }
      
      this.spans.push(customSpan)
      
      // Send span to custom platform
      Effect.runFork(
        client.sendTrace({
          traceId: span.spanContext().traceId,
          spans: [customSpan],
          service: "my-effect-app",
          timestamp: customSpan.startTime
        })
      )
    }
    
    shutdown(): Promise<void> {
      return Promise.resolve()
    }
    
    forceFlush(): Promise<void> {
      return Promise.resolve()
    }
  }
  
  return NodeSdk.layer(() => ({
    resource: { serviceName: "my-effect-app" },
    spanProcessor: new CustomSpanProcessor()
  }))
}

// APM-style tracing with custom metrics
const createAPMIntegration = () => {
  const transactions = new Map<string, {
    startTime: number
    endTime?: number
    spans: Array<{ name: string; duration: number }>
    errors: Array<{ message: string; stack?: string }>
  }>()
  
  const startTransaction = (name: string) => Effect.gen(function* () {
    const transactionId = `txn_${Math.random().toString(36).substr(2, 9)}`
    transactions.set(transactionId, {
      startTime: performance.now(),
      spans: [],
      errors: []
    })
    
    yield* Effect.annotateCurrentSpan("transaction.id", transactionId)
    yield* Effect.annotateCurrentSpan("transaction.name", name)
    
    return transactionId
  })
  
  const endTransaction = (transactionId: string) => Effect.gen(function* () {
    const transaction = transactions.get(transactionId)
    if (transaction) {
      transaction.endTime = performance.now()
      
      // Send APM data to monitoring service
      yield* Effect.log("Transaction completed", {
        transactionId,
        duration: transaction.endTime - transaction.startTime,
        spanCount: transaction.spans.length,
        errorCount: transaction.errors.length
      })
    }
  })
  
  return { startTransaction, endTransaction }
}
```

## Conclusion

Effect's Tracer module provides **comprehensive distributed tracing capabilities**, **automatic context propagation**, and **seamless observability integration** for complex Effect applications.

Key benefits:
- **Unified Observability** - Single tracing solution across all Effect operations with automatic context propagation
- **Performance Insights** - Detailed timing and performance data for identifying bottlenecks and optimizing critical paths  
- **Distributed Debugging** - Complete request flow visibility across services with hierarchical span relationships and error tracking
- **Production Ready** - Enterprise-grade OpenTelemetry integration supporting all major observability platforms and monitoring solutions

Use Effect Tracer when you need comprehensive observability for distributed systems, performance monitoring for complex workflows, or debugging capabilities for multi-service architectures. The declarative approach and automatic context propagation make it ideal for microservices, serverless applications, and any system where understanding request flow and timing relationships is critical for maintaining reliability and performance.