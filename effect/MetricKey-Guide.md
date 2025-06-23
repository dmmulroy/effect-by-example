# MetricKey: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem MetricKey Solves

When building applications, developers often struggle with organizing and identifying metrics across different services and components. Traditional approaches lead to chaos:

```typescript
// Traditional approach - inconsistent metric naming and organization
const userLoginCounter = createCounter('user_login')
const userLoginHistogram = createHistogram('userLoginTime') // Different naming convention!
const userRegisterCounter = createCounter('user_register_events') // Different format!
const apiResponseTimeHistogram = createHistogram('api.response.time', { buckets: [0.1, 0.5, 1.0] })
const dbQueryTimeHistogram = createHistogram('db.query.time', { buckets: [0.01, 0.1, 1.0] }) // Different boundaries!

// Problems with this approach:
// - Inconsistent naming conventions
// - Conflicting metric definitions with same names
// - No type safety or validation
// - Difficult to organize and query metrics
// - Manual tag management and potential conflicts
```

This approach leads to:
- **Metric Conflicts** - Same metric names with different types or configurations
- **Inconsistent Organization** - No standard way to categorize or tag metrics
- **Type Safety Issues** - Runtime errors when metric types don't match usage
- **Poor Discoverability** - Hard to find and understand available metrics
- **Manual Tag Management** - Error-prone manual handling of metric labels

### The MetricKey Solution

MetricKey provides a type-safe, collision-free way to define and organize metrics with guaranteed uniqueness and proper categorization:

```typescript
import { MetricKey, MetricLabel } from "effect"

// Type-safe metric key creation with automatic conflict detection
const userLoginCounter = MetricKey.counter("user_login", {
  description: "Number of user login attempts"
})

const userLoginDuration = MetricKey.histogram("user_login_duration", 
  MetricBoundaries.exponential(0.01, 2.0, 10), 
  "Time taken for user login process"
)

// Impossible to create conflicting metrics - MetricKey prevents this at compile time
const duplicateKey = MetricKey.counter("user_login") // Same name, compatible type - shares state
// const conflictingKey = MetricKey.gauge("user_login") // Would be a different key due to type difference
```

### Key Concepts

**Unique Identity**: Each MetricKey has a unique identity based on name, type, tags, and configuration - preventing accidental metric conflicts

**Type Safety**: MetricKeys enforce type compatibility between metric definitions and usage at compile time

**Hierarchical Organization**: Built-in support for tags and labels enables organized metric hierarchies and efficient querying

## Basic Usage Patterns

### Pattern 1: Basic Metric Key Creation

```typescript
import { MetricKey, MetricBoundaries } from "effect"

// Counter for tracking events
const requestCounter = MetricKey.counter("http_requests_total", {
  description: "Total number of HTTP requests"
})

// Gauge for current values
const activeConnectionsGauge = MetricKey.gauge("active_connections", {
  description: "Number of currently active connections"
})

// Histogram for measuring distributions
const responseTimeHistogram = MetricKey.histogram(
  "http_response_time_seconds",
  MetricBoundaries.linear(0.1, 0.1, 10),
  "HTTP response time distribution"
)
```

### Pattern 2: Tagged Metric Keys

```typescript
import { MetricKey, MetricLabel } from "effect"

// Base metric key
const baseCounter = MetricKey.counter("api_requests")

// Add tags to create specialized versions
const getRequestsCounter = baseCounter.pipe(
  MetricKey.tagged("method", "GET"),
  MetricKey.tagged("endpoint", "/users")
)

const postRequestsCounter = baseCounter.pipe(
  MetricKey.tagged("method", "POST"),
  MetricKey.tagged("endpoint", "/users")
)

// Or add multiple tags at once
const taggedCounter = MetricKey.taggedWithLabels(baseCounter, [
  MetricLabel.make("method", "GET"),
  MetricLabel.make("service", "user-service"),
  MetricLabel.make("version", "v1")
])
```

### Pattern 3: Metric Key Types and Validation

```typescript
import { MetricKey, MetricKeyType } from "effect"

// Type-safe metric key creation
const userCounter: MetricKey.Counter<number> = MetricKey.counter("users")
const bigIntCounter: MetricKey.Counter<bigint> = MetricKey.counter("large_values", { bigint: true })

// Frequency table for categorical data
const errorTypeFreq: MetricKey.Frequency = MetricKey.frequency("error_types", {
  description: "Distribution of error types",
  preregisteredWords: ["ValidationError", "NetworkError", "DatabaseError"]
})

// Validation and type checking
const isCounterKey = MetricKeyType.isCounterKey(userCounter.keyType) // true
const isGaugeKey = MetricKeyType.isGaugeKey(userCounter.keyType) // false
```

## Real-World Examples

### Example 1: API Monitoring System

```typescript
import { Effect, MetricKey, MetricBoundaries, MetricLabel } from "effect"

// Comprehensive API monitoring with organized metric keys
class ApiMetrics {
  // Request metrics with consistent naming
  static readonly requests = MetricKey.counter("api_requests_total", {
    description: "Total number of API requests"
  })
  
  static readonly responseTime = MetricKey.histogram(
    "api_response_time_seconds",
    MetricBoundaries.exponential(0.001, 2.0, 15),
    "API response time distribution"
  )
  
  static readonly activeRequests = MetricKey.gauge("api_active_requests", {
    description: "Number of currently processing requests"
  })
  
  // Error tracking
  static readonly errors = MetricKey.counter("api_errors_total", {
    description: "Total number of API errors"
  })
  
  static readonly errorTypes = MetricKey.frequency("api_error_types", {
    description: "Distribution of API error types",
    preregisteredWords: ["validation", "authorization", "rate_limit", "internal", "timeout"]
  })

  // Create tagged versions for specific endpoints
  static forEndpoint(endpoint: string, method: string) {
    return {
      requests: this.requests.pipe(
        MetricKey.tagged("endpoint", endpoint),
        MetricKey.tagged("method", method)
      ),
      responseTime: this.responseTime.pipe(
        MetricKey.tagged("endpoint", endpoint),
        MetricKey.tagged("method", method)
      ),
      activeRequests: this.activeRequests.pipe(
        MetricKey.tagged("endpoint", endpoint),
        MetricKey.tagged("method", method)
      ),
      errors: this.errors.pipe(
        MetricKey.tagged("endpoint", endpoint),
        MetricKey.tagged("method", method)
      )
    }
  }
}

// Usage in request handling
const handleApiRequest = (endpoint: string, method: string) => {
  const metrics = ApiMetrics.forEndpoint(endpoint, method)
  
  return Effect.gen(function* () {
    // Increment active requests
    yield* Effect.log(`Handling ${method} ${endpoint}`)
    
    // Process request with metrics
    const startTime = Date.now()
    const result = yield* processRequest(endpoint, method)
    const duration = Date.now() - startTime
    
    // Record metrics using the type-safe keys
    yield* Effect.succeed(duration / 1000) // Convert to seconds
    
    return result
  }).pipe(
    Effect.catchAll((error) => Effect.gen(function* () {
      // Record error metrics
      yield* Effect.log(`Error in ${method} ${endpoint}: ${error}`)
      return Effect.fail(error)
    }))
  )
}
```

### Example 2: Database Connection Pool Monitoring

```typescript
import { Effect, MetricKey, MetricBoundaries, Duration } from "effect"

// Database metrics with proper organization
class DatabaseMetrics {
  static readonly connectionPool = {
    // Pool state metrics
    active: MetricKey.gauge("db_connections_active", {
      description: "Number of active database connections"
    }),
    
    idle: MetricKey.gauge("db_connections_idle", {
      description: "Number of idle database connections"
    }),
    
    waiting: MetricKey.gauge("db_connections_waiting", {
      description: "Number of requests waiting for connections"
    }),
    
    // Pool operation metrics
    acquired: MetricKey.counter("db_connections_acquired_total", {
      description: "Total number of connections acquired from pool"
    }),
    
    released: MetricKey.counter("db_connections_released_total", {
      description: "Total number of connections released to pool"
    }),
    
    // Performance metrics
    acquisitionTime: MetricKey.histogram(
      "db_connection_acquisition_seconds",
      MetricBoundaries.exponential(0.001, 2.0, 12),
      "Time taken to acquire connection from pool"
    )
  }
  
  static readonly queries = {
    // Query performance
    duration: MetricKey.histogram(
      "db_query_duration_seconds",
      MetricBoundaries.exponential(0.0001, 2.0, 16),
      "Database query execution time"
    ),
    
    count: MetricKey.counter("db_queries_total", {
      description: "Total number of database queries executed"
    }),
    
    errors: MetricKey.counter("db_query_errors_total", {
      description: "Total number of database query errors"
    }),
    
    // Query type distribution
    types: MetricKey.frequency("db_query_types", {
      description: "Distribution of database query types",
      preregisteredWords: ["SELECT", "INSERT", "UPDATE", "DELETE", "TRANSACTION"]
    })
  }

  // Create database-specific metrics
  static forDatabase(database: string) {
    const dbLabel = MetricLabel.make("database", database)
    
    return {
      connectionPool: {
        active: this.connectionPool.active.pipe(MetricKey.taggedWithLabels([dbLabel])),
        idle: this.connectionPool.idle.pipe(MetricKey.taggedWithLabels([dbLabel])),
        waiting: this.connectionPool.waiting.pipe(MetricKey.taggedWithLabels([dbLabel])),
        acquired: this.connectionPool.acquired.pipe(MetricKey.taggedWithLabels([dbLabel])),
        released: this.connectionPool.released.pipe(MetricKey.taggedWithLabels([dbLabel])),
        acquisitionTime: this.connectionPool.acquisitionTime.pipe(MetricKey.taggedWithLabels([dbLabel]))
      },
      queries: {
        duration: this.queries.duration.pipe(MetricKey.taggedWithLabels([dbLabel])),
        count: this.queries.count.pipe(MetricKey.taggedWithLabels([dbLabel])),
        errors: this.queries.errors.pipe(MetricKey.taggedWithLabels([dbLabel])),
        types: this.queries.types.pipe(MetricKey.taggedWithLabels([dbLabel]))
      }
    }
  }
}
```

### Example 3: User Activity Tracking System

```typescript
import { Effect, MetricKey, MetricBoundaries, MetricLabel, Duration } from "effect"

// User activity metrics with comprehensive tracking
class UserActivityMetrics {
  // User session metrics
  static readonly sessions = {
    started: MetricKey.counter("user_sessions_started_total", {
      description: "Total number of user sessions started"
    }),
    
    ended: MetricKey.counter("user_sessions_ended_total", {
      description: "Total number of user sessions ended"
    }),
    
    active: MetricKey.gauge("user_sessions_active", {
      description: "Number of currently active user sessions"
    }),
    
    duration: MetricKey.histogram(
      "user_session_duration_seconds",
      MetricBoundaries.exponential(60, 2.0, 12), // Start at 1 minute
      "Distribution of user session durations"
    )
  }
  
  // User action metrics
  static readonly actions = {
    count: MetricKey.counter("user_actions_total", {
      description: "Total number of user actions performed"
    }),
    
    types: MetricKey.frequency("user_action_types", {
      description: "Distribution of user action types",
      preregisteredWords: [
        "login", "logout", "view_page", "click_button", 
        "submit_form", "download_file", "upload_file", "search"
      ]
    }),
    
    responseTime: MetricKey.histogram(
      "user_action_response_time_seconds",
      MetricBoundaries.linear(0.1, 0.1, 20),
      "Time taken to respond to user actions"
    )
  }
  
  // Feature usage metrics
  static readonly features = {
    usage: MetricKey.counter("user_feature_usage_total", {
      description: "Total number of feature usages by users"
    }),
    
    uniqueUsers: MetricKey.gauge("user_feature_unique_users", {
      description: "Number of unique users using features"
    })
  }

  // Create user-segment specific metrics
  static forUserSegment(segment: string, plan: string = "free") {
    const segmentTags = [
      MetricLabel.make("user_segment", segment),
      MetricLabel.make("plan", plan)
    ]
    
    return {
      sessions: {
        started: this.sessions.started.pipe(MetricKey.taggedWithLabels(segmentTags)),
        ended: this.sessions.ended.pipe(MetricKey.taggedWithLabels(segmentTags)),
        active: this.sessions.active.pipe(MetricKey.taggedWithLabels(segmentTags)),
        duration: this.sessions.duration.pipe(MetricKey.taggedWithLabels(segmentTags))
      },
      actions: {
        count: this.actions.count.pipe(MetricKey.taggedWithLabels(segmentTags)),
        types: this.actions.types.pipe(MetricKey.taggedWithLabels(segmentTags)),
        responseTime: this.actions.responseTime.pipe(MetricKey.taggedWithLabels(segmentTags))
      },
      features: {
        usage: this.features.usage.pipe(MetricKey.taggedWithLabels(segmentTags)),
        uniqueUsers: this.features.uniqueUsers.pipe(MetricKey.taggedWithLabels(segmentTags))
      }
    }
  }

  // Create feature-specific metrics
  static forFeature(featureName: string) {
    const featureTag = MetricLabel.make("feature", featureName)
    
    return {
      usage: this.features.usage.pipe(MetricKey.taggedWithLabels([featureTag])),
      uniqueUsers: this.features.uniqueUsers.pipe(MetricKey.taggedWithLabels([featureTag])),
      responseTime: this.actions.responseTime.pipe(MetricKey.taggedWithLabels([featureTag]))
    }
  }
}

// Usage in user activity tracking
const trackUserSession = (userId: string, userSegment: string, plan: string) => 
  Effect.gen(function* () {
    const metrics = UserActivityMetrics.forUserSegment(userSegment, plan)
    const sessionStart = Date.now()
    
    // Start session tracking
    yield* Effect.log(`Starting session for user ${userId}`)
    
    // Session management with metrics
    return {
      trackAction: (actionType: string) => Effect.gen(function* () {
        const actionStart = Date.now()
        yield* Effect.log(`User ${userId} performed action: ${actionType}`)
        
        // Simulate action processing
        yield* Effect.sleep(Duration.millis(Math.random() * 100))
        
        const actionDuration = Date.now() - actionStart
        yield* Effect.succeed(actionDuration / 1000)
      }),
      
      endSession: () => Effect.gen(function* () {
        const sessionDuration = Date.now() - sessionStart
        yield* Effect.log(`Ending session for user ${userId}`)
        yield* Effect.succeed(sessionDuration / 1000)
      })
    }
  })
```

## Advanced Features Deep Dive

### Metric Key Equality and Hashing

MetricKeys use structural equality based on all their components - name, type, description, and tags:

```typescript
import { Equal, MetricKey, MetricLabel } from "effect"

const createMetricKeyComparison = () => Effect.gen(function* () {
  // Same metric keys are equal
  const counter1 = MetricKey.counter("requests")
  const counter2 = MetricKey.counter("requests")
  
  const areEqual = Equal.equals(counter1, counter2) // true
  
  // Different types create different keys
  const counterKey = MetricKey.counter("metric")
  const gaugeKey = MetricKey.gauge("metric")
  
  const typesEqual = Equal.equals(counterKey, gaugeKey) // false
  
  // Tags affect equality
  const baseKey = MetricKey.counter("api_requests")
  const taggedKey = baseKey.pipe(MetricKey.tagged("method", "GET"))
  
  const tagsEqual = Equal.equals(baseKey, taggedKey) // false
  
  // Same tags in different order are equal
  const tagged1 = baseKey.pipe(
    MetricKey.tagged("method", "GET"),
    MetricKey.tagged("endpoint", "/users")
  )
  
  const tagged2 = baseKey.pipe(
    MetricKey.tagged("endpoint", "/users"),
    MetricKey.tagged("method", "GET")
  )
  
  const orderEqual = Equal.equals(tagged1, tagged2) // true
  
  return { areEqual, typesEqual, tagsEqual, orderEqual }
})
```

### Complex Metric Key Hierarchies

```typescript
import { MetricKey, MetricLabel, MetricBoundaries } from "effect"

// Create a comprehensive metric hierarchy for microservices
class ServiceMetrics {
  private constructor(
    private readonly serviceName: string,
    private readonly serviceVersion: string,
    private readonly environment: string
  ) {}
  
  static create(serviceName: string, serviceVersion: string, environment: string) {
    return new ServiceMetrics(serviceName, serviceVersion, environment)
  }
  
  // Base service tags applied to all metrics
  private get baseTags() {
    return [
      MetricLabel.make("service", this.serviceName),
      MetricLabel.make("version", this.serviceVersion),
      MetricLabel.make("environment", this.environment)
    ]
  }
  
  // HTTP metrics with service context
  get http() {
    const httpTags = [...this.baseTags, MetricLabel.make("layer", "http")]
    
    return {
      requests: MetricKey.counter("http_requests_total", {
        description: "Total HTTP requests"
      }).pipe(MetricKey.taggedWithLabels(httpTags)),
      
      duration: MetricKey.histogram(
        "http_request_duration_seconds",
        MetricBoundaries.exponential(0.001, 2.0, 15),
        "HTTP request duration"
      ).pipe(MetricKey.taggedWithLabels(httpTags)),
      
      errors: MetricKey.counter("http_errors_total", {
        description: "Total HTTP errors"
      }).pipe(MetricKey.taggedWithLabels(httpTags)),
      
      // Method-specific metrics
      forMethod: (method: string) => {
        const methodTags = [...httpTags, MetricLabel.make("method", method)]
        return {
          requests: MetricKey.counter("http_requests_total").pipe(
            MetricKey.taggedWithLabels(methodTags)
          ),
          duration: MetricKey.histogram("http_request_duration_seconds", 
            MetricBoundaries.exponential(0.001, 2.0, 15)
          ).pipe(MetricKey.taggedWithLabels(methodTags)),
          errors: MetricKey.counter("http_errors_total").pipe(
            MetricKey.taggedWithLabels(methodTags)
          )
        }
      }
    }
  }
  
  // Database metrics with service context
  get database() {
    const dbTags = [...this.baseTags, MetricLabel.make("layer", "database")]
    
    return {
      connections: {
        active: MetricKey.gauge("db_connections_active").pipe(
          MetricKey.taggedWithLabels(dbTags)
        ),
        pool: MetricKey.gauge("db_connection_pool_size").pipe(
          MetricKey.taggedWithLabels(dbTags)
        )
      },
      
      queries: {
        count: MetricKey.counter("db_queries_total").pipe(
          MetricKey.taggedWithLabels(dbTags)
        ),
        duration: MetricKey.histogram("db_query_duration_seconds",
          MetricBoundaries.exponential(0.0001, 2.0, 16)
        ).pipe(MetricKey.taggedWithLabels(dbTags)),
        errors: MetricKey.counter("db_query_errors_total").pipe(
          MetricKey.taggedWithLabels(dbTags)
        )
      },
      
      // Table-specific metrics
      forTable: (tableName: string) => {
        const tableTags = [...dbTags, MetricLabel.make("table", tableName)]
        return {
          queries: MetricKey.counter("db_queries_total").pipe(
            MetricKey.taggedWithLabels(tableTags)
          ),
          duration: MetricKey.histogram("db_query_duration_seconds",
            MetricBoundaries.exponential(0.0001, 2.0, 16)
          ).pipe(MetricKey.taggedWithLabels(tableTags))
        }
      }
    }
  }
  
  // Business logic metrics
  get business() {
    const businessTags = [...this.baseTags, MetricLabel.make("layer", "business")]
    
    return {
      operations: MetricKey.counter("business_operations_total").pipe(
        MetricKey.taggedWithLabels(businessTags)
      ),
      
      processingTime: MetricKey.histogram("business_operation_duration_seconds",
        MetricBoundaries.linear(0.01, 0.01, 100)
      ).pipe(MetricKey.taggedWithLabels(businessTags)),
      
      errors: MetricKey.counter("business_errors_total").pipe(
        MetricKey.taggedWithLabels(businessTags)
      ),
      
      // Operation-specific metrics
      forOperation: (operationName: string) => {
        const opTags = [...businessTags, MetricLabel.make("operation", operationName)]
        return {
          count: MetricKey.counter("business_operations_total").pipe(
            MetricKey.taggedWithLabels(opTags)
          ),
          duration: MetricKey.histogram("business_operation_duration_seconds",
            MetricBoundaries.linear(0.01, 0.01, 100)
          ).pipe(MetricKey.taggedWithLabels(opTags)),
          success: MetricKey.counter("business_operations_success_total").pipe(
            MetricKey.taggedWithLabels(opTags)
          ),
          errors: MetricKey.counter("business_errors_total").pipe(
            MetricKey.taggedWithLabels(opTags)
          )
        }
      }
    }
  }
}

// Usage
const userServiceMetrics = ServiceMetrics.create("user-service", "1.2.3", "production")
const paymentServiceMetrics = ServiceMetrics.create("payment-service", "2.1.0", "production")

const userOperationMetrics = userServiceMetrics.business.forOperation("create_user")
const paymentOperationMetrics = paymentServiceMetrics.business.forOperation("process_payment")
```

### Metric Key Type Guards and Validation

```typescript
import { MetricKey, MetricKeyType } from "effect"

// Type-safe metric key validation and usage
const validateMetricKey = <T extends MetricKey.Untyped>(key: T) => Effect.gen(function* () {
  // Check if it's a valid metric key
  if (!MetricKey.isMetricKey(key)) {
    return yield* Effect.fail(new Error("Invalid metric key"))
  }
  
  // Type-specific validation
  if (MetricKeyType.isCounterKey(key.keyType)) {
    yield* Effect.log(`Counter key: ${key.name}`)
    // Counter-specific logic
  } else if (MetricKeyType.isGaugeKey(key.keyType)) {
    yield* Effect.log(`Gauge key: ${key.name}`)
    // Gauge-specific logic  
  } else if (MetricKeyType.isHistogramKey(key.keyType)) {
    yield* Effect.log(`Histogram key: ${key.name} with boundaries`)
    // Histogram-specific logic
  } else if (MetricKeyType.isFrequencyKey(key.keyType)) {
    yield* Effect.log(`Frequency key: ${key.name}`)
    // Frequency-specific logic
  } else if (MetricKeyType.isSummaryKey(key.keyType)) {
    yield* Effect.log(`Summary key: ${key.name}`)
    // Summary-specific logic
  }
  
  return key
})

// Dynamic metric key creation with validation
const createDynamicMetricKey = (
  name: string,
  type: "counter" | "gauge" | "histogram",
  options: Record<string, unknown> = {}
) => Effect.gen(function* () {
  switch (type) {
    case "counter":
      return MetricKey.counter(name, {
        description: options.description as string,
        bigint: options.bigint as boolean,
        incremental: options.incremental as boolean
      })
    
    case "gauge":
      return MetricKey.gauge(name, {
        description: options.description as string,
        bigint: options.bigint as boolean
      })
    
    case "histogram":
      if (!options.boundaries) {
        return yield* Effect.fail(new Error("Histogram requires boundaries"))
      }
      return MetricKey.histogram(
        name,
        options.boundaries as any,
        options.description as string
      )
    
    default:
      return yield* Effect.fail(new Error(`Unsupported metric type: ${type}`))
  }
})
```

## Practical Patterns & Best Practices

### Pattern 1: Metric Key Factories

Create reusable factories for consistent metric key creation across your application:

```typescript
import { MetricKey, MetricBoundaries, MetricLabel } from "effect"

// Factory for creating consistent HTTP metrics
const createHttpMetrics = (serviceName: string) => {
  const baseLabels = [MetricLabel.make("service", serviceName)]
  
  return {
    // Request metrics
    requests: MetricKey.counter("http_requests_total", {
      description: `Total HTTP requests for ${serviceName}`
    }).pipe(MetricKey.taggedWithLabels(baseLabels)),
    
    // Response time metrics with appropriate boundaries
    responseTime: MetricKey.histogram(
      "http_response_time_seconds",
      MetricBoundaries.exponential(0.001, 2.0, 15), // 1ms to ~32s
      `HTTP response time for ${serviceName}`
    ).pipe(MetricKey.taggedWithLabels(baseLabels)),
    
    // Error metrics
    errors: MetricKey.counter("http_errors_total", {
      description: `Total HTTP errors for ${serviceName}`
    }).pipe(MetricKey.taggedWithLabels(baseLabels)),
    
    // Active request gauge
    activeRequests: MetricKey.gauge("http_active_requests", {
      description: `Currently active HTTP requests for ${serviceName}`
    }).pipe(MetricKey.taggedWithLabels(baseLabels))
  }
}

// Factory for database metrics
const createDatabaseMetrics = (serviceName: string, databaseType: "postgres" | "redis" | "mongodb") => {
  const baseLabels = [
    MetricLabel.make("service", serviceName),
    MetricLabel.make("database_type", databaseType)
  ]
  
  return {
    connections: {
      active: MetricKey.gauge("db_connections_active", {
        description: `Active ${databaseType} connections for ${serviceName}`
      }).pipe(MetricKey.taggedWithLabels(baseLabels)),
      
      poolSize: MetricKey.gauge("db_connection_pool_size", {
        description: `${databaseType} connection pool size for ${serviceName}`
      }).pipe(MetricKey.taggedWithLabels(baseLabels))
    },
    
    operations: {
      count: MetricKey.counter("db_operations_total", {
        description: `Total ${databaseType} operations for ${serviceName}`
      }).pipe(MetricKey.taggedWithLabels(baseLabels)),
      
      duration: MetricKey.histogram(
        "db_operation_duration_seconds",
        // Different boundaries for different database types
        databaseType === "redis" 
          ? MetricBoundaries.exponential(0.0001, 2.0, 12) // Faster operations
          : MetricBoundaries.exponential(0.001, 2.0, 15),  // Slower operations
        `${databaseType} operation duration for ${serviceName}`
      ).pipe(MetricKey.taggedWithLabels(baseLabels)),
      
      errors: MetricKey.counter("db_operation_errors_total", {
        description: `Total ${databaseType} operation errors for ${serviceName}`
      }).pipe(MetricKey.taggedWithLabels(baseLabels))
    }
  }
}

// Usage
const userServiceHttp = createHttpMetrics("user-service")
const paymentServiceHttp = createHttpMetrics("payment-service")
const userServiceDb = createDatabaseMetrics("user-service", "postgres")
const sessionServiceDb = createDatabaseMetrics("session-service", "redis")
```

### Pattern 2: Metric Key Composition and Inheritance

Build complex metric hierarchies through composition:

```typescript
import { MetricKey, MetricLabel, MetricBoundaries } from "effect"

// Base metric definitions
const baseMetrics = {
  requests: MetricKey.counter("requests_total"),
  errors: MetricKey.counter("errors_total"),
  duration: MetricKey.histogram("operation_duration_seconds", 
    MetricBoundaries.exponential(0.001, 2.0, 15)
  )
}

// Compose service-specific metrics
const createServiceMetrics = (serviceName: string, environment: string) => {
  const serviceTags = [
    MetricLabel.make("service", serviceName),
    MetricLabel.make("environment", environment)
  ]
  
  return {
    requests: baseMetrics.requests.pipe(MetricKey.taggedWithLabels(serviceTags)),
    errors: baseMetrics.errors.pipe(MetricKey.taggedWithLabels(serviceTags)),
    duration: baseMetrics.duration.pipe(MetricKey.taggedWithLabels(serviceTags))
  }
}

// Further compose feature-specific metrics
const createFeatureMetrics = (serviceMetrics: ReturnType<typeof createServiceMetrics>, feature: string) => {
  const featureTag = MetricLabel.make("feature", feature)
  
  return {
    requests: serviceMetrics.requests.pipe(MetricKey.taggedWithLabels([featureTag])),
    errors: serviceMetrics.errors.pipe(MetricKey.taggedWithLabels([featureTag])),
    duration: serviceMetrics.duration.pipe(MetricKey.taggedWithLabels([featureTag]))
  }
}

// Create endpoint-specific metrics
const createEndpointMetrics = (
  featureMetrics: ReturnType<typeof createFeatureMetrics>, 
  method: string, 
  path: string
) => {
  const endpointTags = [
    MetricLabel.make("method", method),
    MetricLabel.make("path", path)
  ]
  
  return {
    requests: featureMetrics.requests.pipe(MetricKey.taggedWithLabels(endpointTags)),
    errors: featureMetrics.errors.pipe(MetricKey.taggedWithLabels(endpointTags)),
    duration: featureMetrics.duration.pipe(MetricKey.taggedWithLabels(endpointTags))
  }
}

// Usage - building a complete metric hierarchy
const userService = createServiceMetrics("user-service", "production")
const authFeature = createFeatureMetrics(userService, "authentication")
const loginEndpoint = createEndpointMetrics(authFeature, "POST", "/auth/login")
const logoutEndpoint = createEndpointMetrics(authFeature, "POST", "/auth/logout")

// Now each endpoint has fully qualified metrics:
// - service: user-service
// - environment: production  
// - feature: authentication
// - method: POST
// - path: /auth/login or /auth/logout
```

### Pattern 3: Metric Key Validation and Error Handling

Implement robust validation for metric key creation:

```typescript
import { Effect, MetricKey, MetricBoundaries } from "effect"

// Validation helpers
const validateMetricName = (name: string): Effect.Effect<string, Error> =>
  Effect.gen(function* () {
    if (!name || name.trim().length === 0) {
      return yield* Effect.fail(new Error("Metric name cannot be empty"))
    }
    
    if (!/^[a-zA-Z][a-zA-Z0-9_]*$/.test(name)) {
      return yield* Effect.fail(new Error("Invalid metric name format. Must start with letter and contain only letters, numbers, and underscores"))
    }
    
    if (name.length > 100) {
      return yield* Effect.fail(new Error("Metric name too long. Maximum 100 characters"))
    }
    
    return name
  })

const validateDescription = (description?: string): Effect.Effect<string | undefined, Error> =>
  Effect.gen(function* () {
    if (description && description.length > 500) {
      return yield* Effect.fail(new Error("Description too long. Maximum 500 characters"))
    }
    return description
  })

const validateTags = (tags: Array<{ key: string; value: string }>): Effect.Effect<Array<{ key: string; value: string }>, Error> =>
  Effect.gen(function* () {
    if (tags.length > 20) {
      return yield* Effect.fail(new Error("Too many tags. Maximum 20 tags allowed"))
    }
    
    for (const tag of tags) {
      if (!tag.key || !tag.value) {
        return yield* Effect.fail(new Error("Tag key and value cannot be empty"))
      }
      
      if (tag.key.length > 50 || tag.value.length > 200) {
        return yield* Effect.fail(new Error("Tag key/value too long"))
      }
    }
    
    // Check for duplicate tag keys
    const keys = tags.map(t => t.key)
    const uniqueKeys = new Set(keys)
    if (keys.length !== uniqueKeys.size) {
      return yield* Effect.fail(new Error("Duplicate tag keys not allowed"))
    }
    
    return tags
  })

// Safe metric key creation
const createValidatedCounter = (
  name: string,
  description?: string,
  tags: Array<{ key: string; value: string }> = []
): Effect.Effect<MetricKey.Counter<number>, Error> =>
  Effect.gen(function* () {
    const validName = yield* validateMetricName(name)
    const validDescription = yield* validateDescription(description)
    const validTags = yield* validateTags(tags)
    
    const baseKey = MetricKey.counter(validName, {
      description: validDescription
    })
    
    // Apply tags if provided
    if (validTags.length === 0) {
      return baseKey
    }
    
    const metricLabels = validTags.map(tag => MetricLabel.make(tag.key, tag.value))
    return baseKey.pipe(MetricKey.taggedWithLabels(metricLabels))
  })

const createValidatedHistogram = (
  name: string,
  boundaries: MetricBoundaries.MetricBoundaries,
  description?: string,
  tags: Array<{ key: string; value: string }> = []
): Effect.Effect<MetricKey.Histogram, Error> =>
  Effect.gen(function* () {
    const validName = yield* validateMetricName(name)
    const validDescription = yield* validateDescription(description)
    const validTags = yield* validateTags(tags)
    
    // Validate boundaries
    if (!boundaries || boundaries.values.length === 0) {
      return yield* Effect.fail(new Error("Histogram boundaries cannot be empty"))
    }
    
    const baseKey = MetricKey.histogram(validName, boundaries, validDescription)
    
    if (validTags.length === 0) {
      return baseKey
    }
    
    const metricLabels = validTags.map(tag => MetricLabel.make(tag.key, tag.value))
    return baseKey.pipe(MetricKey.taggedWithLabels(metricLabels))
  })

// Usage with error handling
const safeMetricCreation = Effect.gen(function* () {
  // This will succeed
  const validCounter = yield* createValidatedCounter(
    "user_requests",
    "Number of user requests",
    [{ key: "service", value: "user-api" }]
  )
  
  // This will fail with validation error
  const invalidCounter = yield* createValidatedCounter(
    "", // Empty name
    "Description"
  ).pipe(
    Effect.catchAll(error => Effect.log(`Counter creation failed: ${error.message}`))
  )
  
  return validCounter
})
```

## Integration Examples

### Integration with Prometheus

```typescript
import { Effect, MetricKey, MetricBoundaries, MetricLabel } from "effect"

// Prometheus-compatible metric key creation
class PrometheusMetrics {
  // Prometheus naming conventions
  private static toPrometheusName(name: string): string {
    return name
      .toLowerCase()
      .replace(/[^a-z0-9_]/g, '_')
      .replace(/^[^a-z]/, 'metric_$&')
      .replace(/_+/g, '_')
  }
  
  // Create HTTP metrics following Prometheus conventions
  static createHttpMetrics(namespace: string = "http") {
    return {
      // Standard HTTP request counter
      requestsTotal: MetricKey.counter(`${namespace}_requests_total`, {
        description: "Total number of HTTP requests"
      }),
      
      // HTTP request duration histogram with standard buckets
      requestDurationSeconds: MetricKey.histogram(
        `${namespace}_request_duration_seconds`,
        MetricBoundaries.linear(0.005, 0.005, 12) // .005, .01, .025, .05, .075, .1, .25, .5, .75, 1.0, 2.5, 5.0, 10.0
          .pipe(boundaries => MetricBoundaries.fromArray([
            ...boundaries.values,
            ...MetricBoundaries.exponential(0.005, 2, 10).values
          ])),
        "Duration of HTTP requests"
      ),
      
      // HTTP request size histogram
      requestSizeBytes: MetricKey.histogram(
        `${namespace}_request_size_bytes`,
        MetricBoundaries.exponential(1, 10, 8), // 1, 10, 100, 1K, 10K, 100K, 1M, 10M, 100M
        "Size of HTTP requests in bytes"
      ),
      
      // HTTP response size histogram  
      responseSizeBytes: MetricKey.histogram(
        `${namespace}_response_size_bytes`,
        MetricBoundaries.exponential(1, 10, 8),
        "Size of HTTP responses in bytes"
      ),
      
      // Currently active requests
      requestsActive: MetricKey.gauge(`${namespace}_requests_active`, {
        description: "Currently active HTTP requests"
      })
    }
  }
  
  // Create database metrics following Prometheus conventions
  static createDatabaseMetrics(namespace: string = "db") {
    return {
      // Connection pool metrics
      connectionsActive: MetricKey.gauge(`${namespace}_connections_active`, {
        description: "Currently active database connections"
      }),
      
      connectionsIdle: MetricKey.gauge(`${namespace}_connections_idle`, {
        description: "Currently idle database connections"  
      }),
      
      connectionsWaiting: MetricKey.gauge(`${namespace}_connections_waiting`, {
        description: "Currently waiting database connection requests"
      }),
      
      // Query metrics
      queriesTotal: MetricKey.counter(`${namespace}_queries_total`, {
        description: "Total number of database queries"
      }),
      
      queryDurationSeconds: MetricKey.histogram(
        `${namespace}_query_duration_seconds`,
        MetricBoundaries.exponential(0.0001, 4, 15), // 0.1ms to ~16s
        "Duration of database queries"
      ),
      
      // Transaction metrics
      transactionsTotal: MetricKey.counter(`${namespace}_transactions_total`, {
        description: "Total number of database transactions"
      }),
      
      transactionDurationSeconds: MetricKey.histogram(
        `${namespace}_transaction_duration_seconds`,
        MetricBoundaries.exponential(0.001, 4, 12), // 1ms to ~16s
        "Duration of database transactions"
      )
    }
  }
  
  // Create application metrics
  static createApplicationMetrics(appName: string) {
    const namespace = this.toPrometheusName(appName)
    
    return {
      // Application info
      info: MetricKey.gauge(`${namespace}_info`, {
        description: `Information about ${appName} application`
      }),
      
      // Uptime
      uptimeSeconds: MetricKey.counter(`${namespace}_uptime_seconds_total`, {
        description: `Total uptime of ${appName} in seconds`
      }),
      
      // Business metrics
      businessOperationsTotal: MetricKey.counter(`${namespace}_business_operations_total`, {
        description: `Total number of business operations in ${appName}`
      }),
      
      businessOperationDurationSeconds: MetricKey.histogram(
        `${namespace}_business_operation_duration_seconds`,
        MetricBoundaries.exponential(0.01, 2, 12),
        `Duration of business operations in ${appName}`
      ),
      
      // Error metrics
      errorsTotal: MetricKey.counter(`${namespace}_errors_total`, {
        description: `Total number of errors in ${appName}`
      }),
      
      // Resource usage
      memoryUsageBytes: MetricKey.gauge(`${namespace}_memory_usage_bytes`, {
        description: `Memory usage of ${appName} in bytes`
      }),
      
      cpuUsagePercent: MetricKey.gauge(`${namespace}_cpu_usage_percent`, {
        description: `CPU usage of ${appName} as percentage`
      })
    }
  }
}

// Example usage with proper Prometheus labels
const createPrometheusCompatibleService = (serviceName: string, version: string, environment: string) => {
  const httpMetrics = PrometheusMetrics.createHttpMetrics()  
  const dbMetrics = PrometheusMetrics.createDatabaseMetrics()
  const appMetrics = PrometheusMetrics.createApplicationMetrics(serviceName)
  
  // Apply standard Prometheus labels
  const standardLabels = [
    MetricLabel.make("service", serviceName),
    MetricLabel.make("version", version),
    MetricLabel.make("environment", environment),
    MetricLabel.make("instance", `${serviceName}-${process.pid}`)
  ]
  
  return {
    http: {
      requestsTotal: httpMetrics.requestsTotal.pipe(MetricKey.taggedWithLabels(standardLabels)),
      requestDurationSeconds: httpMetrics.requestDurationSeconds.pipe(MetricKey.taggedWithLabels(standardLabels)),
      requestsActive: httpMetrics.requestsActive.pipe(MetricKey.taggedWithLabels(standardLabels))
    },
    
    database: {
      connectionsActive: dbMetrics.connectionsActive.pipe(MetricKey.taggedWithLabels(standardLabels)),
      queriesTotal: dbMetrics.queriesTotal.pipe(MetricKey.taggedWithLabels(standardLabels)),
      queryDurationSeconds: dbMetrics.queryDurationSeconds.pipe(MetricKey.taggedWithLabels(standardLabels))
    },
    
    application: {
      info: appMetrics.info.pipe(MetricKey.taggedWithLabels([
        ...standardLabels,
        MetricLabel.make("build_info", "1.0.0"),
        MetricLabel.make("git_commit", "abc123def")
      ])),
      uptimeSeconds: appMetrics.uptimeSeconds.pipe(MetricKey.taggedWithLabels(standardLabels)),
      errorsTotal: appMetrics.errorsTotal.pipe(MetricKey.taggedWithLabels(standardLabels))
    }
  }
}
```

### Integration with Custom Metric Registries

```typescript
import { Effect, MetricKey, MetricLabel, HashMap, Ref } from "effect"

// Custom metric registry using MetricKey as the key
interface MetricEntry {
  readonly key: MetricKey.Untyped
  readonly value: unknown
  readonly lastUpdated: Date
  readonly updateCount: number
}

class CustomMetricRegistry {
  constructor(private readonly storage: Ref.Ref<HashMap.HashMap<string, MetricEntry>>) {}
  
  static create(): Effect.Effect<CustomMetricRegistry, never> {
    return Effect.gen(function* () {
      const storage = yield* Ref.make(HashMap.empty<string, MetricEntry>())
      return new CustomMetricRegistry(storage)
    })
  }
  
  // Register a metric key
  register<T extends MetricKey.Untyped>(key: T, initialValue: unknown): Effect.Effect<void, Error> {
    return Effect.gen(function* () {
      const keyId = this.getKeyId(key)
      const entry: MetricEntry = {
        key,
        value: initialValue,
        lastUpdated: new Date(),
        updateCount: 0
      }
      
      yield* Ref.update(this.storage, HashMap.set(keyId, entry))
      yield* Effect.log(`Registered metric: ${key.name}`)
    })
  }
  
  // Update metric value
  update<T extends MetricKey.Untyped>(key: T, value: unknown): Effect.Effect<void, Error> {
    return Effect.gen(function* () {
      const keyId = this.getKeyId(key)
      const currentMap = yield* Ref.get(this.storage)
      
      const existingEntry = HashMap.get(currentMap, keyId)
      if (!existingEntry) {
        return yield* Effect.fail(new Error(`Metric not registered: ${key.name}`))
      }
      
      const updatedEntry: MetricEntry = {
        ...existingEntry,
        value,
        lastUpdated: new Date(),
        updateCount: existingEntry.updateCount + 1
      }
      
      yield* Ref.update(this.storage, HashMap.set(keyId, updatedEntry))
    })
  }
  
  // Get metric value
  get<T extends MetricKey.Untyped>(key: T): Effect.Effect<unknown, Error> {
    return Effect.gen(function* () {
      const keyId = this.getKeyId(key)
      const currentMap = yield* Ref.get(this.storage)
      
      const entry = HashMap.get(currentMap, keyId)
      if (!entry) {
        return yield* Effect.fail(new Error(`Metric not found: ${key.name}`))
      }
      
      return entry.value
    })
  }
  
  // List all metrics matching a pattern
  listByPattern(namePattern: RegExp): Effect.Effect<Array<MetricEntry>, never> {
    return Effect.gen(function* () {
      const currentMap = yield* Ref.get(this.storage)
      const entries = Array.from(HashMap.values(currentMap))
      
      return entries.filter(entry => namePattern.test(entry.key.name))
    })
  }
  
  // List metrics by tag
  listByTag(tagKey: string, tagValue?: string): Effect.Effect<Array<MetricEntry>, never> {
    return Effect.gen(function* () {
      const currentMap = yield* Ref.get(this.storage)
      const entries = Array.from(HashMap.values(currentMap))
      
      return entries.filter(entry => {
        const hasTag = entry.key.tags.some(tag => {
          if (tagValue) {
            return tag.key === tagKey && tag.value === tagValue
          }
          return tag.key === tagKey
        })
        return hasTag
      })
    })
  }
  
  // Export metrics in a specific format
  export(format: "json" | "prometheus"): Effect.Effect<string, Error> {
    return Effect.gen(function* () {
      const currentMap = yield* Ref.get(this.storage)
      const entries = Array.from(HashMap.values(currentMap))
      
      switch (format) {
        case "json":
          return JSON.stringify(entries.map(entry => ({
            name: entry.key.name,
            description: entry.key.description,
            tags: entry.key.tags.map(tag => ({ key: tag.key, value: tag.value })),
            value: entry.value,
            lastUpdated: entry.lastUpdated.toISOString(),
            updateCount: entry.updateCount
          })), null, 2)
        
        case "prometheus":
          return entries.map(entry => {
            const tags = entry.key.tags.length > 0 
              ? `{${entry.key.tags.map(tag => `${tag.key}="${tag.value}"`).join(",")}}`
              : ""
            return `${entry.key.name}${tags} ${entry.value}`
          }).join("\n")
        
        default:
          return yield* Effect.fail(new Error(`Unsupported format: ${format}`))
      }
    })
  }
  
  private getKeyId(key: MetricKey.Untyped): string {
    // Create a unique identifier for the metric key
    const tagString = key.tags
      .map(tag => `${tag.key}=${tag.value}`)
      .sort()
      .join(",")
    
    return `${key.name}:${key.keyType}:${tagString}`
  }
}

// Usage example
const useCustomRegistry = Effect.gen(function* () {
  const registry = yield* CustomMetricRegistry.create()
  
  // Create and register metrics
  const requestCounter = MetricKey.counter("http_requests").pipe(
    MetricKey.tagged("method", "GET"),
    MetricKey.tagged("endpoint", "/users")
  )
  
  const responseTimeHistogram = MetricKey.histogram(
    "http_response_time",
    MetricBoundaries.exponential(0.001, 2, 10)
  ).pipe(
    MetricKey.tagged("method", "GET"),
    MetricKey.tagged("endpoint", "/users")
  )
  
  // Register metrics
  yield* registry.register(requestCounter, 0)
  yield* registry.register(responseTimeHistogram, { buckets: [], count: 0, sum: 0 })
  
  // Update metrics
  yield* registry.update(requestCounter, 42)
  yield* registry.update(responseTimeHistogram, { buckets: [1, 5, 10], count: 16, sum: 23.4 })
  
  // Query metrics
  const counterValue = yield* registry.get(requestCounter)
  const httpMetrics = yield* registry.listByTag("method", "GET")
  const allMetrics = yield* registry.listByPattern(/^http_/)
  
  // Export metrics
  const jsonExport = yield* registry.export("json")
  const prometheusExport = yield* registry.export("prometheus")
  
  yield* Effect.log(`Counter value: ${counterValue}`)
  yield* Effect.log(`HTTP metrics count: ${httpMetrics.length}`)
  yield* Effect.log(`JSON export:\n${jsonExport}`)
  yield* Effect.log(`Prometheus export:\n${prometheusExport}`)
})
```

## Conclusion

MetricKey provides type-safe metric organization, conflict prevention, and hierarchical structuring for complex applications.

Key benefits:
- **Conflict Prevention**: Unique keys prevent metric definition conflicts and ensure consistency
- **Type Safety**: Compile-time validation of metric types and usage patterns
- **Hierarchical Organization**: Tag-based organization enables efficient metric querying and management
- **Integration Ready**: Seamless integration with monitoring systems like Prometheus and custom registries

MetricKey is essential when building applications that require comprehensive observability, consistent metric organization, and type-safe metric handling across distributed systems and microservices architectures.