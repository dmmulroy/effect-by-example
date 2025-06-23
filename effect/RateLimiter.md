# RateLimiter: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem RateLimiter Solves

Traditional rate limiting approaches often lack composability, type safety, and proper resource management. They typically involve complex state management, timing logic, and don't integrate well with Effect's structured concurrency model.

```typescript
// Traditional approach - manual rate limiting
let requests = 0
let lastReset = Date.now()
const LIMIT = 10
const WINDOW = 1000

async function rateLimitedCall(fn: () => Promise<any>) {
  const now = Date.now()
  if (now - lastReset > WINDOW) {
    requests = 0
    lastReset = now
  }
  
  if (requests >= LIMIT) {
    const waitTime = WINDOW - (now - lastReset)
    await new Promise(resolve => setTimeout(resolve, waitTime))
    return rateLimitedCall(fn) // Recursive call
  }
  
  requests++
  return await fn()
}
```

This approach leads to:
- **Complex State Management** - Manual tracking of requests and timing
- **Race Conditions** - Concurrent access can corrupt the request counter
- **Poor Composability** - Difficult to combine multiple rate limiters
- **Resource Leaks** - No proper cleanup of timers and state
- **Testing Challenges** - Hard to test time-dependent behavior

### The RateLimiter Solution

RateLimiter provides composable, type-safe rate limiting with automatic resource management and multiple algorithms for different use cases.

```typescript
import { Effect, RateLimiter, Scope } from "effect"

// Create a rate limiter with token bucket algorithm
const apiLimiter = Effect.scoped(
  RateLimiter.make({
    limit: 10,
    interval: "1 seconds",
    algorithm: "token-bucket"
  })
)

// Use the rate limiter to protect API calls
const rateLimitedApiCall = Effect.gen(function* () {
  const limiter = yield* apiLimiter
  return yield* limiter(
    Effect.tryPromise(() => fetch("/api/data"))
  )
})
```

### Key Concepts

**Rate Limiter**: A function that takes an Effect and returns a rate-limited version of that Effect

**Token Bucket Algorithm**: Allows bursts up to the limit, then spreads remaining requests evenly over the interval

**Fixed Window Algorithm**: Allows up to the limit during each fixed time window, then resets

**Cost-Based Limiting**: Different operations can consume different amounts of the rate limit "budget"

## Basic Usage Patterns

### Pattern 1: Simple Rate Limiting

```typescript
import { Effect, RateLimiter } from "effect"

// Create a basic rate limiter - 5 requests per second
const createBasicLimiter = RateLimiter.make({
  limit: 5,
  interval: "1 seconds"
})

const protectedOperation = Effect.scoped(
  Effect.gen(function* () {
    const limiter = yield* createBasicLimiter
    
    // Wrap any Effect with rate limiting
    return yield* limiter(
      Effect.sync(() => console.log("Rate limited operation"))
    )
  })
)
```

### Pattern 2: Choosing Rate Limiting Algorithms

```typescript
// Token bucket - allows bursts, then steady rate
const tokenBucketLimiter = RateLimiter.make({
  limit: 10,
  interval: "1 seconds",
  algorithm: "token-bucket" // Default
})

// Fixed window - strict limit per time window
const fixedWindowLimiter = RateLimiter.make({
  limit: 10,
  interval: "1 seconds", 
  algorithm: "fixed-window"
})

const demonstrateAlgorithms = Effect.scoped(
  Effect.gen(function* () {
    const tokenBucket = yield* tokenBucketLimiter
    const fixedWindow = yield* fixedWindowLimiter
    
    // Token bucket allows immediate burst of 10 requests
    // then 1 request every 100ms
    yield* Effect.all(
      Array.from({ length: 10 }, () => 
        tokenBucket(Effect.sync(() => "burst"))
      )
    )
    
    // Fixed window allows 10 requests per second window
    // then blocks until next window
    yield* fixedWindow(Effect.sync(() => "windowed"))
  })
)
```

### Pattern 3: Cost-Based Rate Limiting

```typescript
import { Function as Fn } from "effect"

const createCreditLimiter = Effect.scoped(
  Effect.gen(function* () {
    // 1000 credits per hour
    const baseLimiter = yield* RateLimiter.make({
      limit: 1000,
      interval: "1 hours"
    })
    
    // Different operations cost different amounts
    const queryLimiter = Fn.compose(baseLimiter, RateLimiter.withCost(1))
    const mutationLimiter = Fn.compose(baseLimiter, RateLimiter.withCost(5))
    const heavyQueryLimiter = Fn.compose(baseLimiter, RateLimiter.withCost(10))
    
    return { queryLimiter, mutationLimiter, heavyQueryLimiter }
  })
)
```

## Real-World Examples

### Example 1: API Rate Limiting for External Service

Building a service that calls external APIs with different rate limits per endpoint.

```typescript
import { Effect, RateLimiter, Schedule, Layer, Context } from "effect"
import { Function as Fn } from "effect"

// Define the external API service
class ExternalApiService extends Context.Tag("ExternalApiService")<
  ExternalApiService,
  {
    readonly getUserData: (userId: string) => Effect.Effect<UserData, ApiError>
    readonly updateUser: (userId: string, data: Partial<UserData>) => Effect.Effect<void, ApiError>
    readonly searchUsers: (query: string) => Effect.Effect<UserData[], ApiError>
  }
>() {}

interface UserData {
  readonly id: string
  readonly name: string
  readonly email: string
}

class ApiError {
  readonly _tag = "ApiError"
  constructor(readonly message: string, readonly status?: number) {}
}

// Create rate limiters for different API endpoints
const makeApiRateLimiters = Effect.scoped(
  Effect.gen(function* () {
    // GitHub API: 5000 requests per hour for authenticated requests
    const primaryLimiter = yield* RateLimiter.make({
      limit: 5000,
      interval: "1 hours",
      algorithm: "token-bucket"
    })
    
    // Secondary rate limit: 60 requests per minute per endpoint
    const secondaryLimiter = yield* RateLimiter.make({
      limit: 60,
      interval: "1 minutes",
      algorithm: "fixed-window"
    })
    
    // Compose both limiters - respects both constraints
    const combinedLimiter = Fn.compose(primaryLimiter, secondaryLimiter)
    
    // Different endpoints have different costs
    const getUserLimiter = Fn.compose(combinedLimiter, RateLimiter.withCost(1))
    const updateUserLimiter = Fn.compose(combinedLimiter, RateLimiter.withCost(3))
    const searchLimiter = Fn.compose(combinedLimiter, RateLimiter.withCost(2))
    
    return { getUserLimiter, updateUserLimiter, searchLimiter }
  })
)

// Implementation with proper error handling and retries
const makeExternalApiService = Effect.gen(function* () {
  const { getUserLimiter, updateUserLimiter, searchLimiter } = yield* makeApiRateLimiters
  
  const baseUrl = "https://api.external-service.com"
  
  const makeRequest = <T>(
    url: string,
    options?: RequestInit
  ): Effect.Effect<T, ApiError> =>
    Effect.tryPromise({
      try: () => fetch(`${baseUrl}${url}`, options).then(res => {
        if (!res.ok) {
          throw new ApiError(`HTTP ${res.status}: ${res.statusText}`, res.status)
        }
        return res.json()
      }),
      catch: (error) => new ApiError(
        error instanceof Error ? error.message : "Unknown error"
      )
    }).pipe(
      Effect.retry(
        Schedule.exponential("100 millis") 
          .pipe(Schedule.intersect(Schedule.recurs(3)))
      )
    )
  
  const getUserData = (userId: string) =>
    getUserLimiter(makeRequest<UserData>(`/users/${userId}`))
  
  const updateUser = (userId: string, data: Partial<UserData>) =>
    updateUserLimiter(
      makeRequest<void>(`/users/${userId}`, {
        method: "PUT",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(data)
      })
    )
  
  const searchUsers = (query: string) =>
    searchLimiter(
      makeRequest<UserData[]>(`/users/search?q=${encodeURIComponent(query)}`)
    )
  
  return ExternalApiService.of({ getUserData, updateUser, searchUsers })
})

// Layer for the service
const ExternalApiServiceLive = Layer.effect(ExternalApiService, makeExternalApiService)

// Usage example
const processUsers = Effect.gen(function* () {
  const api = yield* ExternalApiService
  
  // These calls are automatically rate limited
  const user1 = yield* api.getUserData("user1")
  const user2 = yield* api.getUserData("user2") 
  
  // Update operations cost more credits
  yield* api.updateUser("user1", { name: "Updated Name" })
  
  // Search operations
  const searchResults = yield* api.searchUsers("john")
  
  return { user1, user2, searchResults }
}).pipe(
  Effect.provide(ExternalApiServiceLive)
)
```

### Example 2: Database Connection Rate Limiting

Protecting database resources from being overwhelmed while maintaining good performance.

```typescript
import { Effect, RateLimiter, Pool, Context, Layer } from "effect"

// Database connection interface
interface DatabaseConnection {
  readonly query: <T>(sql: string, params?: unknown[]) => Effect.Effect<T[], DbError>
  readonly execute: (sql: string, params?: unknown[]) => Effect.Effect<void, DbError>
  readonly close: () => Effect.Effect<void>
}

class DbError {
  readonly _tag = "DbError"
  constructor(readonly message: string, readonly query?: string) {}
}

class DatabaseService extends Context.Tag("DatabaseService")<
  DatabaseService,
  {
    readonly queryUsers: (limit?: number) => Effect.Effect<UserData[], DbError>
    readonly createUser: (data: Omit<UserData, "id">) => Effect.Effect<UserData, DbError>
    readonly updateUser: (id: string, data: Partial<UserData>) => Effect.Effect<void, DbError>
    readonly deleteUser: (id: string) => Effect.Effect<void, DbError>
  }
>() {}

// Create rate-limited database operations
const makeDatabaseService = Effect.gen(function* () {
  // Connection pool
  const connectionPool = yield* Pool.make({
    acquire: Effect.sync(() => ({
      query: <T>(sql: string, params?: unknown[]) => 
        Effect.tryPromise({
          try: () => executeQuery<T>(sql, params),
          catch: (error) => new DbError(
            error instanceof Error ? error.message : "Query failed",
            sql
          )
        }),
      execute: (sql: string, params?: unknown[]) =>
        Effect.tryPromise({
          try: () => executeCommand(sql, params),
          catch: (error) => new DbError(
            error instanceof Error ? error.message : "Command failed", 
            sql
          )
        }),
      close: () => Effect.sync(() => {
        // Close connection logic
      })
    } satisfies DatabaseConnection)),
    size: 10
  })
  
  // Rate limiters for different operation types
  const queryLimiter = yield* RateLimiter.make({
    limit: 100, // 100 queries per second
    interval: "1 seconds",
    algorithm: "token-bucket"
  })
  
  const writeLimiter = yield* RateLimiter.make({
    limit: 20, // 20 writes per second 
    interval: "1 seconds",
    algorithm: "fixed-window"
  })
  
  // Heavy operations (reports, bulk operations) are more restricted
  const heavyQueryLimiter = yield* RateLimiter.make({
    limit: 5,
    interval: "1 seconds",
    algorithm: "token-bucket"
  })
  
  const withConnection = <A, E>(
    operation: (conn: DatabaseConnection) => Effect.Effect<A, E>
  ) => Pool.get(connectionPool).pipe(
    Effect.flatMap(operation),
    Effect.scoped
  )
  
  const queryUsers = (limit = 50) =>
    queryLimiter(
      withConnection(conn => 
        conn.query<UserData>(
          "SELECT id, name, email FROM users LIMIT ?", 
          [limit]
        )
      )
    )
  
  const createUser = (data: Omit<UserData, "id">) =>
    writeLimiter(
      withConnection(conn => Effect.gen(function* () {
        yield* conn.execute(
          "INSERT INTO users (name, email) VALUES (?, ?)",
          [data.name, data.email]
        )
        
        const [user] = yield* conn.query<UserData>(
          "SELECT id, name, email FROM users WHERE email = ?",
          [data.email]
        )
        
        return user
      }))
    )
  
  const updateUser = (id: string, data: Partial<UserData>) =>
    writeLimiter(
      withConnection(conn => {
        const updates = Object.entries(data)
          .filter(([_, value]) => value !== undefined)
          .map(([key]) => `${key} = ?`)
          .join(", ")
        
        if (updates.length === 0) {
          return Effect.void
        }
        
        const values = Object.values(data).filter(v => v !== undefined)
        return conn.execute(
          `UPDATE users SET ${updates} WHERE id = ?`,
          [...values, id]
        )
      })
    )
  
  const deleteUser = (id: string) =>
    writeLimiter(
      withConnection(conn => 
        conn.execute("DELETE FROM users WHERE id = ?", [id])
      )
    )
  
  return DatabaseService.of({
    queryUsers,
    createUser, 
    updateUser,
    deleteUser
  })
})

// Simulate database operations
const executeQuery = <T>(sql: string, params?: unknown[]): Promise<T[]> =>
  new Promise(resolve => {
    setTimeout(() => {
      console.log(`Executing query: ${sql}`, params)
      resolve([] as T[])
    }, Math.random() * 50 + 10)
  })

const executeCommand = (sql: string, params?: unknown[]): Promise<void> =>
  new Promise(resolve => {
    setTimeout(() => {
      console.log(`Executing command: ${sql}`, params)
      resolve()
    }, Math.random() * 100 + 20)
  })

const DatabaseServiceLive = Layer.scoped(
  DatabaseService,
  makeDatabaseService
)

// Usage with automatic rate limiting
const processUserData = Effect.gen(function* () {
  const db = yield* DatabaseService
  
  // These operations are automatically rate limited
  const users = yield* db.queryUsers(10)
  
  yield* db.createUser({
    name: "New User",
    email: "new@example.com"
  })
  
  if (users.length > 0) {
    yield* db.updateUser(users[0].id, { name: "Updated User" })
  }
  
  return users
}).pipe(
  Effect.provide(DatabaseServiceLive)
)
```

### Example 3: HTTP Server with Rate Limiting Middleware

Building HTTP middleware that applies different rate limits based on user authentication and endpoint sensitivity.

```typescript
import { Effect, RateLimiter, Context, Layer, Option, HashMap } from "effect"
import { Function as Fn } from "effect"

// HTTP types
interface HttpRequest {
  readonly url: string
  readonly method: string
  readonly headers: Record<string, string>
  readonly body?: string
}

interface HttpResponse {
  readonly status: number
  readonly headers?: Record<string, string>
  readonly body?: string
}

type HttpHandler = (request: HttpRequest) => Effect.Effect<HttpResponse, HttpError>

class HttpError {
  constructor(
    readonly status: number,
    readonly message: string
  ) {}
}

// Rate limiting service
class RateLimitingService extends Context.Tag("RateLimitingService")<
  RateLimitingService,
  {
    readonly checkRateLimit: (
      identifier: string,
      endpoint: string
    ) => Effect.Effect<boolean, never>
  }
>() {}

// User authentication service
class AuthService extends Context.Tag("AuthService")<
  AuthService,
  {
    readonly authenticateRequest: (request: HttpRequest) => Effect.Effect<Option.Option<string>, never>
  }
>() {}

const makeRateLimitingService = Effect.scoped(
  Effect.gen(function* () {
    // Different rate limiters for different user tiers and endpoints
    const rateLimiters = yield* Effect.all({
      // Anonymous users - very limited
      anonymous: RateLimiter.make({
        limit: 10,
        interval: "1 minutes",
        algorithm: "fixed-window"
      }),
      
      // Authenticated users - more generous
      authenticated: RateLimiter.make({
        limit: 100,
        interval: "1 minutes", 
        algorithm: "token-bucket"
      }),
      
      // Premium users - highest limits
      premium: RateLimiter.make({
        limit: 1000,
        interval: "1 minutes",
        algorithm: "token-bucket"
      }),
      
      // Sensitive endpoints (admin, payment) - strict limits
      sensitive: RateLimiter.make({
        limit: 5,
        interval: "1 minutes",
        algorithm: "fixed-window"
      })
    })
    
    // Track rate limiters per user
    const userLimiters = HashMap.empty<string, RateLimiter.RateLimiter>()
    
    const getRateLimiterForUser = (
      userType: "anonymous" | "authenticated" | "premium",
      endpoint: string
    ) => {
      // Sensitive endpoints always use strict limiting
      if (endpoint.startsWith("/admin") || endpoint.startsWith("/payment")) {
        return Fn.compose(rateLimiters.sensitive, rateLimiters[userType])
      }
      
      return rateLimiters[userType]
    }
    
    const checkRateLimit = (identifier: string, endpoint: string) =>
      Effect.gen(function* () {
        // Determine user type from identifier
        const userType = identifier.startsWith("premium_") ? "premium" :
                        identifier.startsWith("user_") ? "authenticated" :
                        "anonymous"
        
        const limiter = getRateLimiterForUser(userType, endpoint)
        
        // Try to acquire rate limit permission
        // Using a no-op effect to test the rate limiter
        const testEffect = Effect.sync(() => true)
        
        return yield* limiter(testEffect).pipe(
          Effect.map(() => true),
          Effect.catchAll(() => Effect.succeed(false))
        )
      })
    
    return RateLimitingService.of({ checkRateLimit })
  })
)

const makeAuthService = Effect.sync(() =>
  AuthService.of({
    authenticateRequest: (request: HttpRequest) => {
      const authHeader = request.headers["authorization"]
      if (!authHeader) {
        return Effect.succeed(Option.none())
      }
      
      // Simple token parsing (in real app, verify JWT etc.)
      if (authHeader.startsWith("Bearer premium_")) {
        return Effect.succeed(Option.some("premium_user"))
      } else if (authHeader.startsWith("Bearer user_")) {
        return Effect.succeed(Option.some("user_authenticated"))
      }
      
      return Effect.succeed(Option.none())
    }
  })
)

// Rate limiting middleware
const withRateLimit = (handler: HttpHandler): HttpHandler =>
  (request: HttpRequest) => Effect.gen(function* () {
    const rateLimiter = yield* RateLimitingService
    const auth = yield* AuthService
    
    // Get user identifier
    const userId = yield* auth.authenticateRequest(request)
    const identifier = Option.match(userId, {
      onNone: () => request.headers["x-forwarded-for"] || "anonymous",
      onSome: (id) => id
    })
    
    // Check rate limit
    const allowed = yield* rateLimiter.checkRateLimit(identifier, request.url)
    
    if (!allowed) {
      return {
        status: 429,
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          error: "Rate limit exceeded",
          retryAfter: 60
        })
      }
    }
    
    // Proceed with actual handler
    return yield* handler(request)
  })

// Example handlers
const getUserHandler: HttpHandler = (request) =>
  Effect.succeed({
    status: 200,
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ message: "User data" })
  })

const adminHandler: HttpHandler = (request) =>
  Effect.succeed({
    status: 200,
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ message: "Admin data" })
  })

// Set up services
const RateLimitingServiceLive = Layer.scoped(
  RateLimitingService,
  makeRateLimitingService
)

const AuthServiceLive = Layer.effect(AuthService, makeAuthService)

const ServicesLayer = Layer.merge(RateLimitingServiceLive, AuthServiceLive)

// Example server usage
const processRequest = (request: HttpRequest) => {
  const handler = request.url.startsWith("/admin") 
    ? withRateLimit(adminHandler)
    : withRateLimit(getUserHandler)
  
  return handler(request)
}.pipe(
  Effect.provide(ServicesLayer)
)

// Example requests
const testRateLimit = Effect.gen(function* () {
  // Anonymous user - should be rate limited quickly
  const anonymousRequest: HttpRequest = {
    url: "/api/users",
    method: "GET", 
    headers: {}
  }
  
  // Premium user - should have high limits
  const premiumRequest: HttpRequest = {
    url: "/api/users",
    method: "GET",
    headers: { authorization: "Bearer premium_abc123" }
  }
  
  // Admin endpoint - should be heavily rate limited
  const adminRequest: HttpRequest = {
    url: "/admin/users",
    method: "GET",
    headers: { authorization: "Bearer user_def456" }
  }
  
  // Test rapid requests
  yield* Effect.all([
    processRequest(anonymousRequest),
    processRequest(premiumRequest),
    processRequest(adminRequest)
  ])
})
```

## Advanced Features Deep Dive

### Feature 1: Rate Limiter Composition

Rate limiters can be composed to enforce multiple constraints simultaneously.

#### Basic Rate Limiter Composition

```typescript
import { Function as Fn, Effect, RateLimiter } from "effect"

const composedRateLimiters = Effect.scoped(
  Effect.gen(function* () {
    // Global rate limit: 1000 requests per hour
    const globalLimiter = yield* RateLimiter.make({
      limit: 1000,
      interval: "1 hours",
      algorithm: "token-bucket"
    })
    
    // Burst protection: 10 requests per second
    const burstLimiter = yield* RateLimiter.make({
      limit: 10,
      interval: "1 seconds", 
      algorithm: "fixed-window"
    })
    
    // Composed limiter respects both constraints
    const combinedLimiter = Fn.compose(globalLimiter, burstLimiter)
    
    return combinedLimiter
  })
)
```

#### Real-World Composition Example

```typescript
// Multi-tier rate limiting for different API endpoints
const createApiRateLimiters = Effect.scoped(
  Effect.gen(function* () {
    // Base limiters
    const globalLimiter = yield* RateLimiter.make({
      limit: 10000,
      interval: "1 hours"
    })
    
    const perMinuteLimiter = yield* RateLimiter.make({
      limit: 200,
      interval: "1 minutes"
    })
    
    const perSecondLimiter = yield* RateLimiter.make({
      limit: 5,
      interval: "1 seconds"
    })
    
    // Compose base limiters
    const baseLimiter = Fn.compose(
      globalLimiter,
      Fn.compose(perMinuteLimiter, perSecondLimiter)
    )
    
    // Endpoint-specific limiters with different costs
    const searchLimiter = Fn.compose(baseLimiter, RateLimiter.withCost(2))
    const uploadLimiter = Fn.compose(baseLimiter, RateLimiter.withCost(10))
    const analyticsLimiter = Fn.compose(baseLimiter, RateLimiter.withCost(5))
    
    return {
      baseLimiter,
      searchLimiter, 
      uploadLimiter,
      analyticsLimiter
    }
  })
)
```

### Feature 2: Dynamic Cost Calculation

Rate limiters can have costs calculated dynamically based on the request or operation.

```typescript
// Helper for dynamic cost calculation
const withDynamicCost = <A, E, R>(
  costCalculator: (input: A) => number
) => (effect: Effect.Effect<A, E, R>): Effect.Effect<A, E, R> =>
  Effect.gen(function* () {
    const result = yield* effect
    const cost = costCalculator(result)
    return yield* RateLimiter.withCost(cost)(Effect.succeed(result))
  })

// Example: File upload with size-based costs  
interface FileUpload {
  readonly filename: string
  readonly size: number
  readonly content: Uint8Array
}

const processFileUpload = (upload: FileUpload) =>
  Effect.scoped(
    Effect.gen(function* () {
      const limiter = yield* RateLimiter.make({
        limit: 1000, // 1000 MB per hour
        interval: "1 hours"
      })
      
      // Cost based on file size (1 credit per MB)
      const uploadCost = Math.ceil(upload.size / (1024 * 1024))
      
      return yield* limiter(
        Effect.sync(() => processFile(upload)).pipe(
          RateLimiter.withCost(uploadCost)
        )
      )
    })
  )

const processFile = (upload: FileUpload): string => {
  // File processing logic
  return `Processed ${upload.filename}`
}
```

### Feature 3: Advanced Error Handling and Backpressure

```typescript
import { Schedule, Cause } from "effect"

// Rate limiter with sophisticated retry and backpressure
const createResilientApiClient = Effect.scoped(
  Effect.gen(function* () {
    const rateLimiter = yield* RateLimiter.make({
      limit: 100,
      interval: "1 minutes",
      algorithm: "token-bucket"
    })
    
    const makeApiCall = <T>(
      url: string,
      options?: RequestInit
    ): Effect.Effect<T, ApiError> =>
      rateLimiter(
        Effect.tryPromise({
          try: () => fetch(url, options).then(res => res.json()),
          catch: (error) => new ApiError(
            error instanceof Error ? error.message : "Unknown error"
          )
        })
      ).pipe(
        // Retry with exponential backoff on rate limit
        Effect.retry(
          Schedule.exponential("1 seconds")
            .pipe(Schedule.intersect(Schedule.recurs(5)))
            .pipe(Schedule.whileInput((error: ApiError) => 
              error.message.includes("rate limit") ||
              error.message.includes("429")
            ))
        ),
        // Add circuit breaker behavior
        Effect.timeout("30 seconds"),
        Effect.tapError((error) => 
          Effect.sync(() => console.error("API call failed:", error))
        )
      )
    
    return { makeApiCall }
  })
)

// Example with graceful degradation
const getDataWithFallback = <T>(
  primaryUrl: string,
  fallbackData: T
): Effect.Effect<T, never> =>
  Effect.scoped(
    Effect.gen(function* () {
      const client = yield* createResilientApiClient
      
      return yield* client.makeApiCall<T>(primaryUrl).pipe(
        Effect.catchAll((error) =>
          Effect.gen(function* () {
            yield* Effect.logWarning(`Primary API failed: ${error.message}`)
            yield* Effect.logInfo("Using fallback data")
            return fallbackData
          })
        )
      )
    })
  )
```

## Practical Patterns & Best Practices

### Pattern 1: Rate Limiter Factory with Configuration

```typescript
interface RateLimitConfig {
  readonly tier: "free" | "pro" | "enterprise"
  readonly endpoint: "query" | "mutation" | "upload" | "admin"
}

const createConfigurableRateLimiter = (config: RateLimitConfig) =>
  Effect.scoped(
    Effect.gen(function* () {
      // Base limits by tier
      const tierLimits = {
        free: { limit: 100, interval: "1 hours" as const },
        pro: { limit: 1000, interval: "1 hours" as const },
        enterprise: { limit: 10000, interval: "1 hours" as const }
      }
      
      // Endpoint cost multipliers
      const endpointCosts = {
        query: 1,
        mutation: 2,
        upload: 10,
        admin: 5
      }
      
      const baseLimiter = yield* RateLimiter.make({
        ...tierLimits[config.tier],
        algorithm: "token-bucket"
      })
      
      const cost = endpointCosts[config.endpoint]
      return Fn.compose(baseLimiter, RateLimiter.withCost(cost))
    })
  )

// Usage
const freeUserQueryLimiter = createConfigurableRateLimiter({
  tier: "free",
  endpoint: "query"
})

const enterpriseUploadLimiter = createConfigurableRateLimiter({
  tier: "enterprise", 
  endpoint: "upload"
})
```

### Pattern 2: Rate Limiter Middleware for Effect-based HTTP Frameworks

```typescript
// Generic rate limiting middleware
const rateLimitMiddleware = <R>(
  getRateLimiter: (request: HttpRequest) => Effect.Effect<RateLimiter.RateLimiter, never, R>
) => <A, E>(
  handler: (request: HttpRequest) => Effect.Effect<A, E, R>
) => (request: HttpRequest): Effect.Effect<A, E | HttpError, R> =>
  Effect.gen(function* () {
    const limiter = yield* getRateLimiter(request)
    
    return yield* limiter(handler(request)).pipe(
      Effect.mapError((error) => 
        error instanceof HttpError 
          ? error 
          : new HttpError(500, "Internal server error")
      )
    )
  })

// Request-based rate limiter selection
const selectRateLimiter = (request: HttpRequest) =>
  Effect.gen(function* () {
    const isAdmin = request.url.startsWith("/admin")
    const isUpload = request.method === "POST" && request.url.includes("/upload")
    
    if (isAdmin) {
      return yield* RateLimiter.make({
        limit: 10,
        interval: "1 minutes",
        algorithm: "fixed-window"
      })
    }
    
    if (isUpload) {
      return yield* RateLimiter.make({
        limit: 5,
        interval: "1 minutes", 
        algorithm: "token-bucket"
      })
    }
    
    return yield* RateLimiter.make({
      limit: 100,
      interval: "1 minutes",
      algorithm: "token-bucket"
    })
  })
```

### Pattern 3: Testing Rate Limiters

```typescript
import { TestClock } from "effect"

const testRateLimiter = Effect.gen(function* () {
  const limiter = yield* RateLimiter.make({
    limit: 3,
    interval: "1 seconds",
    algorithm: "fixed-window"
  })
  
  // Should allow 3 immediate calls
  const results1 = yield* Effect.all([
    limiter(Effect.succeed("call1")),
    limiter(Effect.succeed("call2")),
    limiter(Effect.succeed("call3"))
  ])
  
  console.log("First batch:", results1) // Should complete immediately
  
  // Fourth call should be delayed
  const delayedCall = yield* Effect.fork(
    limiter(Effect.succeed("call4"))
  )
  
  // Advance time to next window
  yield* TestClock.adjust("1 seconds")
  
  const result4 = yield* Fiber.join(delayedCall)
  console.log("Delayed call:", result4)
})

// Property-based testing helper
const testRateLimiterProperties = (
  algorithm: "fixed-window" | "token-bucket"
) => Effect.gen(function* () {
  const limit = 10
  const limiter = yield* RateLimiter.make({
    limit,
    interval: "1 seconds",
    algorithm
  })
  
  // Property: Should never exceed limit in initial burst
  const burstResults = yield* Effect.all(
    Array.from({ length: limit + 5 }, (_, i) =>
      Effect.fork(limiter(Effect.succeed(i)))
    )
  )
  
  const completedImmediately = burstResults.filter(
    fiber => /* check if completed immediately */ true
  )
  
  // Should be exactly `limit` calls completed immediately
  console.assert(
    completedImmediately.length === limit,
    `Expected ${limit} immediate completions, got ${completedImmediately.length}`
  )
})
```

## Integration Examples

### Integration with HTTP Servers (Fastify/Express style)

```typescript
import { NodeHttpServer, HttpRouter, HttpServerRequest } from "@effect/platform"
import { Layer } from "effect"

// HTTP server with built-in rate limiting
const createRateLimitedServer = Effect.gen(function* () {
  // Create rate limiters for different routes
  const rateLimiters = yield* Effect.all({
    api: RateLimiter.make({
      limit: 100,
      interval: "1 minutes",
      algorithm: "token-bucket"
    }),
    
    upload: RateLimiter.make({
      limit: 10,
      interval: "1 minutes",
      algorithm: "fixed-window"
    }),
    
    public: RateLimiter.make({
      limit: 1000,
      interval: "1 minutes", 
      algorithm: "token-bucket"
    })
  })
  
  const router = HttpRouter.empty.pipe(
    HttpRouter.get("/api/users", 
      Effect.gen(function* () {
        const request = yield* HttpServerRequest.HttpServerRequest
        
        return yield* rateLimiters.api(
          Effect.succeed({ users: ["Alice", "Bob"] })
        )
      })
    ),
    
    HttpRouter.post("/api/upload",
      Effect.gen(function* () {
        const request = yield* HttpServerRequest.HttpServerRequest
        
        return yield* rateLimiters.upload(
          Effect.succeed({ message: "Upload successful" })
        )
      })
    ),
    
    HttpRouter.get("/health",
      Effect.gen(function* () {
        return yield* rateLimiters.public(
          Effect.succeed({ status: "healthy" })
        )
      })
    )
  )
  
  return router
})

const ServerLive = Layer.scoped(
  "HttpServer",
  createRateLimitedServer
)
```

### Integration with Queue/Stream Processing

```typescript
import { Queue, Stream } from "effect"

// Rate-limited stream processing
const createRateLimitedProcessor = <A, B, E, R>(
  processor: (item: A) => Effect.Effect<B, E, R>,
  rateLimit: { limit: number; interval: Duration.DurationInput }
) => Effect.scoped(
  Effect.gen(function* () {
    const limiter = yield* RateLimiter.make({
      ...rateLimit,
      algorithm: "token-bucket"
    })
    
    const processItem = (item: A) =>
      limiter(processor(item))
    
    return { processItem }
  })
)

// Example: Rate-limited email sending
interface EmailMessage {
  readonly to: string
  readonly subject: string
  readonly body: string
}

const createEmailProcessor = Effect.scoped(
  Effect.gen(function* () {
    // Limit to 10 emails per minute to avoid provider limits
    const { processItem } = yield* createRateLimitedProcessor(
      (email: EmailMessage) => 
        Effect.tryPromise({
          try: () => sendEmail(email),
          catch: (error) => new Error(`Failed to send email: ${error}`)
        }),
      { limit: 10, interval: "1 minutes" }
    )
    
    const processEmailQueue = (queue: Queue.Queue<EmailMessage>) =>
      Stream.fromQueue(queue).pipe(
        Stream.tap(email => 
          Effect.log(`Processing email to ${email.to}`)
        ),
        Stream.mapEffect(processItem),
        Stream.tap(result => 
          Effect.log(`Email sent successfully`)
        ),
        Stream.runDrain
      )
    
    return { processEmailQueue }
  })
)

const sendEmail = (email: EmailMessage): Promise<void> => {
  // Simulate email sending
  return new Promise(resolve => {
    setTimeout(() => {
      console.log(`Sent email to ${email.to}: ${email.subject}`)
      resolve()
    }, 100)
  })
}
```

### Integration with Caching and Persistence

```typescript
import { Cache } from "effect"

// Rate-limited cache with fallback to database
const createCachedService = <K, V, E>(
  fetchFromDb: (key: K) => Effect.Effect<V, E>,
  cacheKey: (key: K) => string
) => Effect.scoped(
  Effect.gen(function* () {
    // Rate limit database queries
    const dbLimiter = yield* RateLimiter.make({
      limit: 50,
      interval: "1 minutes",
      algorithm: "token-bucket"
    })
    
    // Create cache with TTL
    const cache = yield* Cache.make({
      capacity: 1000,
      timeToLive: "5 minutes",
      lookup: (key: K) => dbLimiter(fetchFromDb(key))
    })
    
    const get = (key: K) => Cache.get(cache, key)
    
    const invalidate = (key: K) => Cache.invalidate(cache, key)
    
    const refresh = (key: K) => Cache.refresh(cache, key)
    
    return { get, invalidate, refresh }
  })
)

// Usage example
interface User {
  readonly id: string
  readonly name: string
  readonly email: string
}

const fetchUserFromDb = (userId: string): Effect.Effect<User, Error> =>
  Effect.tryPromise({
    try: () => queryDatabase(`SELECT * FROM users WHERE id = ?`, [userId]),
    catch: (error) => new Error(`Database query failed: ${error}`)
  })

const queryDatabase = (query: string, params: unknown[]): Promise<User> =>
  new Promise((resolve) => {
    // Simulate database query
    setTimeout(() => {
      resolve({
        id: params[0] as string,
        name: "John Doe",
        email: "john@example.com"
      })
    }, 50)
  })

const UserServiceLive = Layer.scoped(
  "UserService",
  createCachedService(fetchUserFromDb, (userId: string) => `user:${userId}`)
)
```

## Conclusion

RateLimiter provides type-safe, composable rate limiting with automatic resource management and multiple algorithms for different use cases.

Key benefits:
- **Composability**: Rate limiters can be easily combined to enforce multiple constraints
- **Resource Safety**: Automatic cleanup and proper resource management through Scope
- **Algorithm Choice**: Token bucket for bursty traffic, fixed window for strict limits
- **Cost-Based Limiting**: Different operations can consume different amounts of rate limit budget
- **Effect Integration**: Seamless integration with Effect's error handling, retries, and structured concurrency

RateLimiter is essential when you need to protect resources from being overwhelmed, integrate with external APIs that have rate limits, or implement fair usage policies in your applications.