# HttpClient: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem HttpClient Solves

Building robust HTTP clients in TypeScript traditionally involves juggling multiple concerns:

```typescript
// Traditional approach - problematic code
async function fetchUser(id: string) {
  try {
    const response = await fetch(`https://api.example.com/users/${id}`, {
      headers: {
        'Authorization': 'Bearer ' + getToken(), // Token might be undefined
        'Content-Type': 'application/json'
      }
    })
    
    if (!response.ok) {
      // Limited error information
      throw new Error(`HTTP ${response.status}`)
    }
    
    const data = await response.json()
    // No type safety - runtime errors possible
    return data as User // Type assertion, not validation
  } catch (error) {
    // Lost context about the original request
    console.error('Failed to fetch user', error)
    // No retry logic
    // No timeout handling
    // No request/response logging
    throw error
  }
}

// Retry logic requires wrapper functions
async function fetchWithRetry(fn: () => Promise<any>, retries = 3) {
  // Complex retry implementation
  // No exponential backoff
  // No selective retry based on error type
}
```

This approach leads to:
- **Type Unsafety** - Runtime errors from malformed responses
- **Poor Error Context** - Lost information about failed requests
- **Complex Retry Logic** - Manual implementation of retry strategies
- **No Built-in Tracing** - Difficult to debug in production
- **Authentication Boilerplate** - Repeated auth logic across requests

### The HttpClient Solution

Effect's HttpClient provides a composable, type-safe HTTP client with built-in error handling, retries, and tracing:

```typescript
import { HttpClient, HttpClientRequest, HttpClientResponse } from "@effect/platform"
import { Effect, Schema } from "effect"

// Type-safe schema for validation
const User = Schema.Struct({
  id: Schema.String,
  name: Schema.String,
  email: Schema.String
})

// Clean, composable solution
export const fetchUser = (id: string) => Effect.gen(function* () {
  const client = yield* HttpClient.HttpClient
  const response = yield* client.get(`/users/${id}`)
  return yield* HttpClientResponse.schemaBodyJson(User)(response)
}).pipe(
  Effect.retry({ times: 3, schedule: "exponential" }),
  Effect.withSpan("user.fetch", { attributes: { userId: id } })
)
```

### Key Concepts

**HttpClient**: A service that executes HTTP requests, providing a functional interface for all HTTP methods.

**HttpClientRequest**: An immutable request builder that allows you to construct requests with headers, body, and other options.

**HttpClientResponse**: The response type that includes methods for parsing and validating response bodies.

**Layers**: Effect's dependency injection system that allows you to provide different HTTP client implementations (Node.js, browser, test).

## Basic Usage Patterns

### Pattern 1: Setup/Initialization

```typescript
import { HttpClient, HttpClientRequest } from "@effect/platform"
import { BunHttpClient } from "@effect/platform-bun"
import { NodeHttpClient } from "@effect/platform-node"
import { FetchHttpClient } from "@effect/platform"
import { Effect, Layer } from "effect"

// Choose the appropriate client for your platform
const HttpClientLive = NodeHttpClient.layer  // For Node.js
// const HttpClientLive = BunHttpClient.layer  // For Bun
// const HttpClientLive = FetchHttpClient.layer  // For browsers

// Basic GET request
const getExample = Effect.gen(function* () {
  const client = yield* HttpClient.HttpClient
  const response = yield* client.get("https://jsonplaceholder.typicode.com/posts/1")
  return yield* response.json
})

// Run with dependencies
const runnable = getExample.pipe(
  Effect.provide(HttpClientLive)
)
```

### Pattern 2: Core HTTP Methods

```typescript
import { HttpClient, HttpClientRequest } from "@effect/platform"
import { Effect } from "effect"

const httpMethods = Effect.gen(function* () {
  const client = yield* HttpClient.HttpClient
  
  // GET request
  const getResponse = yield* client.get("/users")
  
  // POST request with JSON body
  const postResponse = yield* client.post("/users").pipe(
    HttpClientRequest.bodyJson({
      name: "John Doe",
      email: "john@example.com"
    }),
    client.execute
  )
  
  // PUT request
  const putResponse = yield* client.put("/users/1").pipe(
    HttpClientRequest.bodyJson({
      name: "Jane Doe",
      email: "jane@example.com"
    }),
    client.execute
  )
  
  // DELETE request
  const deleteResponse = yield* client.del("/users/1")
  
  return { getResponse, postResponse, putResponse, deleteResponse }
})
```

### Pattern 3: Request Configuration

```typescript
import { HttpClient, HttpClientRequest, Headers } from "@effect/platform"
import { Effect } from "effect"

const configuredRequest = Effect.gen(function* () {
  const client = yield* HttpClient.HttpClient
  
  // Configure headers, query params, and timeout
  const response = yield* HttpClientRequest.get("/api/data").pipe(
    HttpClientRequest.setHeader("X-API-Key", "secret-key"),
    HttpClientRequest.setHeaders(Headers.fromInput({
      "Accept": "application/json",
      "X-Custom-Header": "value"
    })),
    HttpClientRequest.setUrlParam("page", "1"),
    HttpClientRequest.setUrlParams({ limit: "10", sort: "desc" }),
    client.execute
  )
  
  return yield* response.json
})
```

## Real-World Examples

### Example 1: REST API Client with Authentication

Building a complete REST API client with authentication, error handling, and type safety:

```typescript
import { HttpClient, HttpClientRequest, HttpClientResponse, HttpClientError } from "@effect/platform"
import { Effect, Schema, Layer, Context, Redacted } from "effect"

// API Configuration
interface ApiConfig {
  readonly baseUrl: string
  readonly apiKey: Redacted.Redacted<string>
}

const ApiConfig = Context.GenericTag<ApiConfig>("ApiConfig")

// Domain models with validation
const User = Schema.Struct({
  id: Schema.Number,
  email: Schema.String,
  name: Schema.String,
  createdAt: Schema.DateFromString
})

const PaginatedResponse = <A, I>(schema: Schema.Schema<A, I>) =>
  Schema.Struct({
    data: Schema.Array(schema),
    total: Schema.Number,
    page: Schema.Number,
    pageSize: Schema.Number
  })

// Custom error types
class ApiError extends Schema.TaggedError<ApiError>()("ApiError", {
  status: Schema.Number,
  message: Schema.String,
  endpoint: Schema.String
}) {}

// API Client implementation
const makeApiClient = Effect.gen(function* () {
  const client = yield* HttpClient.HttpClient
  const config = yield* ApiConfig
  
  // Add authentication and base URL to all requests
  const authenticatedClient = client.pipe(
    HttpClient.mapRequest(HttpClientRequest.prependUrl(config.baseUrl)),
    HttpClient.mapRequest(HttpClientRequest.bearerToken(config.apiKey))
  )
  
  // Helper for handling API errors
  const handleApiError = (endpoint: string) => 
    HttpClient.catchAll((error) => Effect.gen(function* () {
      if (HttpClientError.isHttpClientError(error)) {
        const response = error.response
        const body = yield* response.json
        return yield* Effect.fail(new ApiError({
          status: response.status,
          message: body.message || "Unknown error",
          endpoint
        }))
      }
      return yield* Effect.fail(error)
    }))
  
  // User operations
  const getUsers = (page = 1, pageSize = 10) => Effect.gen(function* () {
    const response = yield* authenticatedClient.get("/users").pipe(
      HttpClientRequest.setUrlParams({ page: String(page), pageSize: String(pageSize) })
    )
    return yield* HttpClientResponse.schemaBodyJson(PaginatedResponse(User))(response)
  }).pipe(
    handleApiError("GET /users"),
    Effect.withSpan("api.getUsers", { attributes: { page, pageSize } })
  )
  
  const getUser = (id: number) => Effect.gen(function* () {
    const response = yield* authenticatedClient.get(`/users/${id}`)
    return yield* HttpClientResponse.schemaBodyJson(User)(response)
  }).pipe(
    handleApiError(`GET /users/${id}`),
    Effect.withSpan("api.getUser", { attributes: { userId: id } })
  )
  
  const createUser = (data: typeof User.Type) => Effect.gen(function* () {
    const response = yield* authenticatedClient.post("/users").pipe(
      HttpClientRequest.schemaBodyJson(User)(data)
    )
    return yield* HttpClientResponse.schemaBodyJson(User)(response)
  }).pipe(
    handleApiError("POST /users"),
    Effect.withSpan("api.createUser")
  )
  
  const updateUser = (id: number, data: Partial<typeof User.Type>) => Effect.gen(function* () {
    const response = yield* authenticatedClient.patch(`/users/${id}`).pipe(
      HttpClientRequest.bodyJson(data)
    )
    return yield* HttpClientResponse.schemaBodyJson(User)(response)
  }).pipe(
    handleApiError(`PATCH /users/${id}`),
    Effect.withSpan("api.updateUser", { attributes: { userId: id } })
  )
  
  const deleteUser = (id: number) => Effect.gen(function* () {
    yield* authenticatedClient.del(`/users/${id}`)
    return { success: true }
  }).pipe(
    handleApiError(`DELETE /users/${id}`),
    Effect.withSpan("api.deleteUser", { attributes: { userId: id } })
  )
  
  return {
    getUsers,
    getUser,
    createUser,
    updateUser,
    deleteUser
  } as const
})

// Create service layer
const ApiClient = Context.GenericTag<Effect.Effect.Success<typeof makeApiClient>>("ApiClient")

const ApiClientLive = Layer.effect(ApiClient, makeApiClient).pipe(
  Layer.provide(HttpClient.layer)
)

// Usage example
const program = Effect.gen(function* () {
  const api = yield* ApiClient
  
  // Create a new user
  const newUser = yield* api.createUser({
    email: "new@example.com",
    name: "New User",
    createdAt: new Date()
  })
  
  // Get paginated users
  const users = yield* api.getUsers(1, 20)
  console.log(`Found ${users.total} users`)
  
  // Update user
  const updated = yield* api.updateUser(newUser.id, { name: "Updated Name" })
  
  return updated
})
```

### Example 2: File Upload with Progress Tracking

Implementing file uploads with progress tracking and resumable uploads:

```typescript
import { HttpClient, HttpClientRequest, HttpClientError } from "@effect/platform"
import { Effect, Stream, Schema, Chunk, Ref } from "effect"
import * as NodeFs from "@effect/platform-node/NodeFileSystem"
import * as Path from "@effect/platform/Path"

// Upload progress tracking
interface UploadProgress {
  readonly bytesUploaded: number
  readonly totalBytes: number
  readonly percentage: number
}

const UploadProgress = Context.GenericTag<Ref.Ref<UploadProgress>>("UploadProgress")

// File upload response
const UploadResponse = Schema.Struct({
  id: Schema.String,
  filename: Schema.String,
  size: Schema.Number,
  url: Schema.String
})

// Chunked upload implementation
const uploadFile = (filePath: string, chunkSize = 1024 * 1024) => Effect.gen(function* () {
  const client = yield* HttpClient.HttpClient
  const fs = yield* NodeFs.FileSystem
  const path = yield* Path.Path
  const progressRef = yield* UploadProgress
  
  // Get file info
  const stats = yield* fs.stat(filePath)
  const filename = path.basename(filePath)
  const totalSize = Number(stats.size)
  
  // Initialize upload session
  const initResponse = yield* client.post("/uploads/init").pipe(
    HttpClientRequest.bodyJson({
      filename,
      size: totalSize,
      chunkSize
    })
  )
  const { uploadId } = yield* initResponse.json as Effect.Effect<{ uploadId: string }>
  
  // Upload file in chunks
  const fileStream = yield* fs.stream(filePath, { chunkSize })
  let offset = 0
  
  const uploadChunks = Stream.mapAccum(fileStream, 0, (chunkIndex, chunk) => {
    const chunkData = Chunk.toUint8Array(chunk)
    const currentOffset = offset
    offset += chunkData.length
    
    return Effect.gen(function* () {
      // Upload chunk
      const response = yield* client.put(`/uploads/${uploadId}/chunks/${chunkIndex}`).pipe(
        HttpClientRequest.bodyUint8Array(chunkData),
        HttpClientRequest.setHeader("Content-Range", `bytes ${currentOffset}-${offset - 1}/${totalSize}`)
      )
      
      // Update progress
      yield* progressRef.update((prev) => ({
        bytesUploaded: offset,
        totalBytes: totalSize,
        percentage: Math.round((offset / totalSize) * 100)
      }))
      
      return response
    }).pipe(
      Effect.retry({
        times: 3,
        schedule: "exponential",
        while: (error) => {
          // Only retry on network errors, not on 4xx errors
          if (HttpClientError.isHttpClientError(error)) {
            return error.response.status >= 500
          }
          return true
        }
      }),
      Effect.map(() => [chunkIndex + 1, { chunkIndex, bytesUploaded: offset }] as const)
    )
  })
  
  // Process all chunks
  yield* Stream.runDrain(uploadChunks)
  
  // Complete upload
  const completeResponse = yield* client.post(`/uploads/${uploadId}/complete`)
  return yield* HttpClientResponse.schemaBodyJson(UploadResponse)(completeResponse)
}).pipe(
  Effect.withSpan("file.upload", { attributes: { filename: Path.basename(filePath) } })
)

// Usage with progress monitoring
const uploadWithProgress = (filePath: string) => Effect.gen(function* () {
  const progressRef = yield* Ref.make<UploadProgress>({
    bytesUploaded: 0,
    totalBytes: 0,
    percentage: 0
  })
  
  // Monitor progress in background
  const monitorFiber = yield* progressRef.changes.pipe(
    Stream.tap((progress) => 
      Effect.sync(() => console.log(`Upload progress: ${progress.percentage}%`))
    ),
    Stream.runDrain,
    Effect.fork
  )
  
  const result = yield* uploadFile(filePath).pipe(
    Effect.provideService(UploadProgress, progressRef)
  )
  
  yield* Fiber.interrupt(monitorFiber)
  return result
})
```

### Example 3: GraphQL Client with Caching

Building a GraphQL client with request caching and batch query support:

```typescript
import { HttpClient, HttpClientRequest, HttpClientResponse } from "@effect/platform"
import { Effect, Schema, Cache, Duration, Request, RequestResolver } from "effect"

// GraphQL types
const GraphQLRequest = Schema.Struct({
  query: Schema.String,
  variables: Schema.optional(Schema.Record(Schema.String, Schema.Unknown)),
  operationName: Schema.optional(Schema.String)
})

const GraphQLResponse = <A, I>(dataSchema: Schema.Schema<A, I>) =>
  Schema.Struct({
    data: Schema.NullOr(dataSchema),
    errors: Schema.optional(Schema.Array(Schema.Struct({
      message: Schema.String,
      path: Schema.optional(Schema.Array(Schema.Union(Schema.String, Schema.Number))),
      extensions: Schema.optional(Schema.Record(Schema.String, Schema.Unknown))
    })))
  })

// GraphQL error
class GraphQLError extends Schema.TaggedError<GraphQLError>()("GraphQLError", {
  errors: Schema.Array(Schema.Struct({
    message: Schema.String,
    path: Schema.optional(Schema.Array(Schema.Union(Schema.String, Schema.Number)))
  }))
}) {}

// Request batching implementation
interface GraphQLBatchRequest extends Request.Request<unknown, GraphQLError> {
  readonly _tag: "GraphQLBatchRequest"
  readonly query: string
  readonly variables?: Record<string, unknown>
  readonly operationName?: string
}

const GraphQLBatchRequest = Request.tagged<GraphQLBatchRequest>("GraphQLBatchRequest")

// GraphQL client implementation
const makeGraphQLClient = (endpoint: string) => Effect.gen(function* () {
  const client = yield* HttpClient.HttpClient
  
  // Batch resolver for combining multiple queries
  const batchResolver = RequestResolver.makeBatched<GraphQLBatchRequest>(
    (requests: Array<GraphQLBatchRequest>) => Effect.gen(function* () {
      // Combine multiple queries into a single request
      const batchedQuery = requests.map((req, index) => ({
        id: String(index),
        query: req.query,
        variables: req.variables,
        operationName: req.operationName
      }))
      
      const response = yield* client.post(endpoint).pipe(
        HttpClientRequest.bodyJson(batchedQuery)
      )
      
      const results = yield* response.json as Effect.Effect<Array<any>>
      
      // Map results back to individual requests
      return requests.map((req, index) => {
        const result = results[index]
        if (result.errors) {
          return Request.completeEffect(req, Effect.fail(new GraphQLError({ errors: result.errors })))
        }
        return Request.completeEffect(req, Effect.succeed(result.data))
      })
    }).pipe(
      Effect.catchAll((error) => 
        Effect.succeed(requests.map(req => 
          Request.completeEffect(req, Effect.fail(new GraphQLError({ 
            errors: [{ message: "Batch request failed" }] 
          })))
        ))
      )
    ),
    { window: Duration.millis(10), maxBatchSize: 50 }
  )
  
  // Cache for query results
  const queryCache = yield* Cache.make({
    capacity: 100,
    timeToLive: Duration.minutes(5),
    lookup: (key: string) => Effect.succeed(null as any)
  })
  
  // Execute GraphQL query with caching
  const query = <A, I>(
    queryString: string,
    schema: Schema.Schema<A, I>,
    variables?: Record<string, unknown>,
    options?: { skipCache?: boolean }
  ) => Effect.gen(function* () {
    const cacheKey = JSON.stringify({ query: queryString, variables })
    
    // Check cache first
    if (!options?.skipCache) {
      const cached = yield* queryCache.get(cacheKey)
      if (cached !== null) return cached as A
    }
    
    // Execute query
    const response = yield* client.post(endpoint).pipe(
      HttpClientRequest.schemaBodyJson(GraphQLRequest)({
        query: queryString,
        variables
      })
    )
    
    const result = yield* HttpClientResponse.schemaBodyJson(GraphQLResponse(schema))(response)
    
    if (result.errors) {
      return yield* Effect.fail(new GraphQLError({ errors: result.errors }))
    }
    
    if (result.data === null) {
      return yield* Effect.fail(new GraphQLError({ 
        errors: [{ message: "No data returned" }] 
      }))
    }
    
    // Cache successful result
    if (!options?.skipCache) {
      yield* queryCache.set(cacheKey, result.data)
    }
    
    return result.data
  }).pipe(
    Effect.withSpan("graphql.query", { 
      attributes: { 
        "graphql.operation": queryString.match(/^(query|mutation|subscription)\s+(\w+)/)?.[2] || "anonymous" 
      } 
    })
  )
  
  // Execute mutation (never cached)
  const mutation = <A, I>(
    mutationString: string,
    schema: Schema.Schema<A, I>,
    variables?: Record<string, unknown>
  ) => query(mutationString, schema, variables, { skipCache: true })
  
  // Batch query execution
  const batchQuery = <A, I>(
    queryString: string,
    schema: Schema.Schema<A, I>,
    variables?: Record<string, unknown>
  ) => Effect.request(
    GraphQLBatchRequest({ query: queryString, variables }),
    batchResolver
  ).pipe(
    Effect.flatMap(data => Schema.decodeUnknown(schema)(data))
  )
  
  return { query, mutation, batchQuery, clearCache: queryCache.invalidateAll }
})

// Usage example
const UserQuery = Schema.Struct({
  user: Schema.Struct({
    id: Schema.String,
    name: Schema.String,
    email: Schema.String,
    posts: Schema.Array(Schema.Struct({
      id: Schema.String,
      title: Schema.String,
      createdAt: Schema.DateFromString
    }))
  })
})

const program = Effect.gen(function* () {
  const graphql = yield* makeGraphQLClient("https://api.example.com/graphql")
  
  // Execute a query with caching
  const userData = yield* graphql.query(
    `query GetUser($id: ID!) {
      user(id: $id) {
        id
        name
        email
        posts {
          id
          title
          createdAt
        }
      }
    }`,
    UserQuery,
    { id: "123" }
  )
  
  // Execute a mutation
  const CreateUserResponse = Schema.Struct({
    createUser: Schema.Struct({
      id: Schema.String,
      name: Schema.String
    })
  })
  
  const newUser = yield* graphql.mutation(
    `mutation CreateUser($input: CreateUserInput!) {
      createUser(input: $input) {
        id
        name
      }
    }`,
    CreateUserResponse,
    { 
      input: { 
        name: "John Doe", 
        email: "john@example.com" 
      } 
    }
  )
  
  return { userData, newUser }
})
```

## Advanced Features Deep Dive

### Feature 1: Retry Strategies

HttpClient provides sophisticated retry capabilities with customizable strategies:

#### Basic Retry Usage

```typescript
import { HttpClient, HttpClientError } from "@effect/platform"
import { Effect, Schedule, Duration } from "effect"

const retryExample = Effect.gen(function* () {
  const client = yield* HttpClient.HttpClient
  
  // Simple retry with exponential backoff
  const response1 = yield* client.get("/api/data").pipe(
    Effect.retry({
      times: 3,
      schedule: Schedule.exponential(Duration.seconds(1))
    })
  )
  
  // Retry only on specific errors
  const response2 = yield* client.get("/api/data").pipe(
    Effect.retry({
      while: (error) => {
        if (HttpClientError.isHttpClientError(error)) {
          // Retry on 5xx errors and network errors
          return error.response.status >= 500
        }
        return true // Retry on other errors
      },
      schedule: Schedule.fibonacci(Duration.seconds(1))
    })
  )
  
  return { response1, response2 }
})
```

#### Real-World Retry Example

```typescript
import { HttpClient, HttpClientError } from "@effect/platform"
import { Effect, Schedule, Duration, Metric } from "effect"

// Metrics for monitoring retries
const retryCounter = Metric.counter("http_client_retries", {
  description: "Number of HTTP request retries"
})

const retryHistogram = Metric.histogram("http_client_retry_duration", {
  description: "Duration of retry attempts",
  boundaries: Metric.Histogram.Boundaries.exponential(0.1, 2, 10)
})

// Custom retry policy with circuit breaker pattern
const makeResilientClient = Effect.gen(function* () {
  const baseClient = yield* HttpClient.HttpClient
  
  // Track consecutive failures
  const failureCount = yield* Ref.make(0)
  const circuitOpen = yield* Ref.make(false)
  
  const resilientClient = baseClient.pipe(
    // Add retry logic
    HttpClient.retry({
      while: (error, attempt) => Effect.gen(function* () {
        // Check circuit breaker
        const isOpen = yield* circuitOpen.get
        if (isOpen) {
          return false // Don't retry if circuit is open
        }
        
        if (HttpClientError.isHttpClientError(error)) {
          const status = error.response.status
          
          // Don't retry on client errors
          if (status >= 400 && status < 500) {
            return false
          }
          
          // Retry on server errors and network issues
          if (status >= 500 || status === 0) {
            yield* retryCounter.increment()
            
            // Open circuit after 5 consecutive failures
            if (attempt >= 5) {
              yield* circuitOpen.set(true)
              yield* Effect.delay(Duration.minutes(1)).pipe(
                Effect.andThen(circuitOpen.set(false)),
                Effect.fork
              )
              return false
            }
            
            return true
          }
        }
        
        return true
      }),
      schedule: Schedule.decorateJittered(
        Schedule.exponential(Duration.seconds(1), 2).pipe(
          Schedule.upTo(Duration.seconds(30))
        ),
        0.1 // Add 10% jitter
      )
    }),
    // Reset failure count on success
    HttpClient.tap(() => failureCount.set(0)),
    // Track retry duration
    HttpClient.tapError(() => 
      Metric.timerUpdate(retryHistogram, Date.now())
    )
  )
  
  return resilientClient
})
```

#### Advanced Retry: Idempotency Keys

```typescript
import { HttpClient, HttpClientRequest } from "@effect/platform"
import { Effect, Random, Ref } from "effect"

// Idempotency key management
const IdempotencyKeys = Context.GenericTag<Ref.Ref<Map<string, string>>>("IdempotencyKeys")

const withIdempotency = <R, E>(
  request: Effect.Effect<HttpClientRequest.HttpClientRequest, E, R>
) => Effect.gen(function* () {
  const keys = yield* IdempotencyKeys
  const keysMap = yield* keys.get
  
  // Generate or retrieve idempotency key
  const requestId = yield* Effect.sync(() => {
    const method = request.method
    const url = request.url
    const body = JSON.stringify(request.body)
    const requestKey = `${method}:${url}:${body}`
    
    const existing = keysMap.get(requestKey)
    if (existing) return existing
    
    const newKey = yield* Random.nextUuid
    keysMap.set(requestKey, newKey)
    return newKey
  })
  
  return HttpClientRequest.setHeader(request, "Idempotency-Key", requestId)
})

// Usage with payment processing
const processPayment = (amount: number, currency: string) => Effect.gen(function* () {
  const client = yield* HttpClient.HttpClient
  
  const response = yield* client.post("/payments").pipe(
    HttpClientRequest.bodyJson({ amount, currency }),
    withIdempotency,
    client.execute,
    Effect.retry({
      times: 5,
      schedule: Schedule.exponential(Duration.seconds(2)),
      while: (error) => {
        // Safe to retry with idempotency key
        return HttpClientError.isHttpClientError(error) && 
               error.response.status >= 500
      }
    })
  )
  
  return yield* response.json
})
```

### Feature 2: Request/Response Transformation

HttpClient provides powerful transformation capabilities for both requests and responses:

#### Basic Transformation Usage

```typescript
import { HttpClient, HttpClientRequest, HttpClientResponse } from "@effect/platform"
import { Effect, Logger } from "effect"

const transformExample = Effect.gen(function* () {
  const client = yield* HttpClient.HttpClient
  
  // Transform requests
  const clientWithAuth = client.pipe(
    HttpClient.mapRequest((request) =>
      HttpClientRequest.setHeader(request, "X-API-Version", "2.0")
    ),
    HttpClient.mapRequestEffect((request) => Effect.gen(function* () {
      const token = yield* getAuthToken() // Some effect to get token
      return HttpClientRequest.bearerToken(request, token)
    }))
  )
  
  // Transform responses
  const clientWithLogging = clientWithAuth.pipe(
    HttpClient.transformResponse((response) => Effect.gen(function* () {
      yield* Logger.info("Response received", {
        status: response.status,
        url: response.url
      })
      return response
    }))
  )
  
  return clientWithLogging
})
```

#### Real-World Transformation Example

```typescript
import { HttpClient, HttpClientRequest, HttpClientResponse, HttpClientError } from "@effect/platform"
import { Effect, Schema, DateTime, Logger, Metric } from "effect"
import * as Crypto from "node:crypto"

// Response envelope unwrapping
const ApiEnvelope = <A, I>(schema: Schema.Schema<A, I>) =>
  Schema.Struct({
    success: Schema.Boolean,
    data: schema,
    meta: Schema.optional(Schema.Struct({
      requestId: Schema.String,
      timestamp: Schema.DateFromString,
      version: Schema.String
    }))
  })

// Request signing for API authentication
const signRequest = (secret: string) => 
  HttpClient.mapRequestEffect((request) => Effect.gen(function* () {
    const timestamp = Date.now().toString()
    const method = request.method
    const url = request.url
    const body = request.body ? JSON.stringify(request.body) : ""
    
    // Create signature
    const message = `${method}\n${url}\n${timestamp}\n${body}`
    const signature = Crypto
      .createHmac("sha256", secret)
      .update(message)
      .digest("hex")
    
    return request.pipe(
      HttpClientRequest.setHeader("X-Timestamp", timestamp),
      HttpClientRequest.setHeader("X-Signature", signature)
    )
  }))

// Response metrics collection
const responseMetrics = {
  duration: Metric.histogram("http_response_duration_ms"),
  statusCodes: Metric.counter("http_response_status_codes")
}

const withMetrics = HttpClient.transformResponse((response) => Effect.gen(function* () {
  const duration = response.headers["x-response-time"]
  if (duration) {
    yield* responseMetrics.duration.update(Number(duration))
  }
  
  yield* responseMetrics.statusCodes.increment({
    status: String(response.status),
    method: response.request.method
  })
  
  return response
}))

// Automatic response unwrapping and validation
const unwrapApiResponse = <A, I>(schema: Schema.Schema<A, I>) =>
  (response: HttpClientResponse.HttpClientResponse) => Effect.gen(function* () {
    const envelope = yield* HttpClientResponse.schemaBodyJson(ApiEnvelope(schema))(response)
    
    if (!envelope.success) {
      return yield* Effect.fail(new Error("API request failed"))
    }
    
    // Log metadata
    if (envelope.meta) {
      yield* Logger.debug("API response metadata", envelope.meta)
    }
    
    return envelope.data
  })

// Complete API client with all transformations
const makeEnhancedClient = (apiSecret: string) => Effect.gen(function* () {
  const baseClient = yield* HttpClient.HttpClient
  
  return baseClient.pipe(
    // Add request signing
    signRequest(apiSecret),
    // Add default headers
    HttpClient.mapRequest((request) =>
      request.pipe(
        HttpClientRequest.setHeader("Accept", "application/json"),
        HttpClientRequest.setHeader("X-Client-Version", "1.0.0")
      )
    ),
    // Add response metrics
    withMetrics,
    // Add request/response logging in development
    HttpClient.tap((response) => Effect.gen(function* () {
      const isDev = yield* Effect.config("NODE_ENV").pipe(
        Effect.map(env => env === "development"),
        Effect.orElse(() => Effect.succeed(false))
      )
      
      if (isDev) {
        yield* Logger.debug("HTTP Request", {
          method: response.request.method,
          url: response.request.url,
          status: response.status
        })
      }
    }))
  )
})

// Usage with automatic unwrapping
const getProducts = Effect.gen(function* () {
  const client = yield* makeEnhancedClient("my-api-secret")
  
  const Product = Schema.Struct({
    id: Schema.String,
    name: Schema.String,
    price: Schema.Number
  })
  
  const response = yield* client.get("/products")
  return yield* unwrapApiResponse(Schema.Array(Product))(response)
})
```

### Feature 3: Streaming and Large File Handling

HttpClient supports streaming for handling large files and real-time data:

#### Basic Streaming Usage

```typescript
import { HttpClient, HttpClientRequest, HttpClientResponse } from "@effect/platform"
import { Effect, Stream, Chunk } from "effect"
import * as NodeFs from "@effect/platform-node/NodeFileSystem"

const streamingExample = Effect.gen(function* () {
  const client = yield* HttpClient.HttpClient
  const fs = yield* NodeFs.FileSystem
  
  // Stream response body
  const response = yield* client.get("/large-file.zip")
  const stream = HttpClientResponse.stream(response)
  
  // Process stream in chunks
  yield* stream.pipe(
    Stream.tap((chunk) => 
      Effect.sync(() => console.log(`Received ${chunk.length} bytes`))
    ),
    Stream.run(fs.sink("/tmp/downloaded-file.zip"))
  )
})
```

#### Real-World Streaming Example

```typescript
import { HttpClient, HttpClientRequest, HttpClientResponse } from "@effect/platform"
import { Effect, Stream, Chunk, Queue, Fiber, Ref } from "effect"
import * as NodeFs from "@effect/platform-node/NodeFileSystem"
import * as Path from "@effect/platform/Path"

// Download manager with pause/resume support
interface DownloadState {
  readonly status: "idle" | "downloading" | "paused" | "completed" | "error"
  readonly bytesDownloaded: number
  readonly totalBytes: number
  readonly speed: number // bytes per second
}

class DownloadManager extends Context.Tag("DownloadManager")<
  DownloadManager,
  {
    readonly download: (url: string, destination: string) => Effect.Effect<void>
    readonly pause: Effect.Effect<void>
    readonly resume: Effect.Effect<void>
    readonly getState: Effect.Effect<DownloadState>
  }
>() {}

const makeDownloadManager = Effect.gen(function* () {
  const client = yield* HttpClient.HttpClient
  const fs = yield* NodeFs.FileSystem
  const path = yield* Path.Path
  
  const state = yield* Ref.make<DownloadState>({
    status: "idle",
    bytesDownloaded: 0,
    totalBytes: 0,
    speed: 0
  })
  
  const pauseSignal = yield* Queue.unbounded<"pause" | "resume">()
  const currentDownload = yield* Ref.make<Fiber.RuntimeFiber<void, any> | null>(null)
  
  const download = (url: string, destination: string) => Effect.gen(function* () {
    // Check if file exists and get current size for resume
    const existingSize = yield* fs.stat(destination).pipe(
      Effect.map(stats => Number(stats.size)),
      Effect.orElse(() => Effect.succeed(0))
    )
    
    // Update state
    yield* state.update(s => ({ ...s, status: "downloading", bytesDownloaded: existingSize }))
    
    // Create request with range header for resume
    const request = existingSize > 0
      ? HttpClientRequest.get(url).pipe(
          HttpClientRequest.setHeader("Range", `bytes=${existingSize}-`)
        )
      : HttpClientRequest.get(url)
    
    const response = yield* client.execute(request)
    
    // Get total size from headers
    const contentLength = response.headers["content-length"]
    const totalBytes = contentLength 
      ? Number(contentLength) + existingSize 
      : 0
    
    yield* state.update(s => ({ ...s, totalBytes }))
    
    // Speed calculation
    const startTime = Date.now()
    const lastSpeedUpdate = yield* Ref.make(Date.now())
    const lastBytes = yield* Ref.make(existingSize)
    
    // Download stream with pause support
    const downloadStream = HttpClientResponse.stream(response).pipe(
      Stream.tap((chunk) => Effect.gen(function* () {
        const chunkSize = chunk.length
        
        // Update bytes downloaded
        yield* state.update(s => ({ 
          ...s, 
          bytesDownloaded: s.bytesDownloaded + chunkSize 
        }))
        
        // Update speed every second
        const now = Date.now()
        const lastUpdate = yield* lastSpeedUpdate.get
        if (now - lastUpdate >= 1000) {
          const bytes = yield* Effect.map(state.get, s => s.bytesDownloaded)
          const lastBytesValue = yield* lastBytes.get
          const speed = (bytes - lastBytesValue) / ((now - lastUpdate) / 1000)
          
          yield* state.update(s => ({ ...s, speed }))
          yield* lastSpeedUpdate.set(now)
          yield* lastBytes.set(bytes)
        }
      })),
      // Interruptible stream for pause/resume
      Stream.interruptWhen(
        Queue.take(pauseSignal).pipe(
          Effect.flatMap(signal => 
            signal === "pause" 
              ? Effect.succeed(true) 
              : Effect.succeed(false)
          )
        )
      )
    )
    
    // Write to file (append mode for resume)
    const sink = existingSize > 0
      ? fs.sink(destination, { flag: "a" })
      : fs.sink(destination)
    
    yield* downloadStream.pipe(
      Stream.run(sink),
      Effect.ensuring(
        state.update(s => ({ 
          ...s, 
          status: s.bytesDownloaded === s.totalBytes ? "completed" : "paused" 
        }))
      )
    )
  }).pipe(
    Effect.catchAll((error) => 
      state.update(s => ({ ...s, status: "error" })).pipe(
        Effect.andThen(Effect.fail(error))
      )
    )
  )
  
  return {
    download: (url: string, destination: string) => Effect.gen(function* () {
      const fiber = yield* Effect.fork(download(url, destination))
      yield* currentDownload.set(fiber)
      yield* Fiber.join(fiber)
    }),
    
    pause: Effect.gen(function* () {
      yield* pauseSignal.offer("pause")
      yield* state.update(s => ({ ...s, status: "paused" }))
    }),
    
    resume: Effect.gen(function* () {
      yield* pauseSignal.offer("resume")
      const current = yield* currentDownload.get
      if (current) {
        yield* Fiber.join(current)
      }
    }),
    
    getState: state.get
  }
})

// Server-sent events (SSE) streaming
const sseClient = (url: string) => Effect.gen(function* () {
  const client = yield* HttpClient.HttpClient
  
  const response = yield* client.get(url).pipe(
    HttpClientRequest.setHeader("Accept", "text/event-stream")
  )
  
  const eventStream = HttpClientResponse.stream(response).pipe(
    Stream.decodeText("utf-8"),
    Stream.splitLines,
    Stream.mapAccum("", (buffer, line) => {
      if (line === "") {
        // Empty line signals end of event
        const event = parseSSEEvent(buffer)
        return [[], event]
      } else {
        // Accumulate lines
        return [[buffer, line].filter(Boolean).join("\n"), null]
      }
    }),
    Stream.filter(event => event !== null),
    Stream.map(event => event!)
  )
  
  return eventStream
})

const parseSSEEvent = (data: string): { event?: string; data: string; id?: string } | null => {
  if (!data) return null
  
  const lines = data.split("\n")
  const event: any = {}
  
  for (const line of lines) {
    const [field, ...valueParts] = line.split(":")
    const value = valueParts.join(":").trim()
    
    switch (field) {
      case "event":
        event.event = value
        break
      case "data":
        event.data = value
        break
      case "id":
        event.id = value
        break
    }
  }
  
  return event.data ? event : null
}

// Usage example
const downloadLargeFile = Effect.gen(function* () {
  const manager = yield* DownloadManager
  
  // Start download
  const downloadFiber = yield* manager.download(
    "https://example.com/large-file.zip",
    "/tmp/large-file.zip"
  ), Effect.fork)
  
  // Monitor progress
  const monitorFiber = yield* Effect.repeat(
    manager.getState.pipe(
      Effect.tap(state => 
        Effect.sync(() => {
          const progress = (state.bytesDownloaded / state.totalBytes) * 100
          const speedMB = state.speed / (1024 * 1024)
          console.log(
            `Progress: ${progress.toFixed(2)}% | ` +
            `Speed: ${speedMB.toFixed(2)} MB/s | ` +
            `Status: ${state.status}`
          )
        })
      )
    ),
    Schedule.fixed(Duration.seconds(1))
  ), Effect.fork)
  
  // Simulate pause after 5 seconds
  yield* Effect.sleep(Duration.seconds(5))
  yield* manager.pause
  console.log("Download paused")
  
  // Resume after 2 seconds
  yield* Effect.sleep(Duration.seconds(2))
  yield* manager.resume
  console.log("Download resumed")
  
  // Wait for completion
  yield* Fiber.join(downloadFiber)
  yield* Fiber.interrupt(monitorFiber)
}).pipe(
  Effect.provide(Layer.effect(DownloadManager, makeDownloadManager))
)
```

## Practical Patterns & Best Practices

### Pattern 1: API Client Factory

Create reusable API client factories with consistent error handling and configuration:

```typescript
import { HttpClient, HttpClientRequest, HttpClientResponse, HttpClientError } from "@effect/platform"
import { Effect, Layer, Context, Config, Duration, Logger } from "effect"

// Generic API client configuration
interface ApiClientConfig {
  readonly baseUrl: string
  readonly timeout: Duration.Duration
  readonly maxRetries: number
  readonly headers?: Record<string, string>
}

// Generic error type for API responses
class ApiResponseError<T = unknown> extends Schema.TaggedError<ApiResponseError<T>>()("ApiResponseError", {
  status: Schema.Number,
  message: Schema.String,
  details: Schema.optional(Schema.Unknown),
  request: Schema.Struct({
    method: Schema.String,
    url: Schema.String
  })
}) {}

// Create typed API client factory
const makeApiClientFactory = <ServiceName extends string>(
  serviceName: ServiceName,
  configPath: string
) => {
  const ServiceConfig = Context.GenericTag<ApiClientConfig>(`${serviceName}Config`)
  
  const ServiceConfigLive = Layer.effect(
    ServiceConfig,
    Effect.gen(function* () {
      const baseUrl = yield* Config.string(`${configPath}.baseUrl`)
      const timeout = yield* Config.duration(`${configPath}.timeout`)
      const maxRetries = yield* Config.integer(`${configPath}.maxRetries`)
      const headers = yield* Config.record(Config.string)(`${configPath}.headers`).pipe(
        Effect.orElse(() => Effect.succeed({}))
      )
      
      return { baseUrl, timeout, maxRetries, headers }
    })
  )
  
  const makeClient = Effect.gen(function* () {
    const config = yield* ServiceConfig
    const httpClient = yield* HttpClient.HttpClient
    
    const client = httpClient.pipe(
      // Base configuration
      HttpClient.mapRequest(HttpClientRequest.prependUrl(config.baseUrl)),
      HttpClient.mapRequest(request => 
        Object.entries(config.headers || {}).reduce(
          (req, [key, value]) => HttpClientRequest.setHeader(req, key, value),
          request
        )
      ),
      // Timeout
      HttpClient.timeout(config.timeout),
      // Retry with exponential backoff
      HttpClient.retry({
        times: config.maxRetries,
        schedule: Schedule.exponential(Duration.seconds(1)),
        while: (error) => {
          if (HttpClientError.isHttpClientError(error)) {
            return error.response.status >= 500 || error.response.status === 0
          }
          return true
        }
      }),
      // Error transformation
      HttpClient.catchAll((error) => Effect.gen(function* () {
        yield* Logger.error(`${serviceName} API error`, { error })
        
        if (HttpClientError.isHttpClientError(error)) {
          const response = error.response
          const body = yield* response.text.pipe(
            Effect.orElse(() => Effect.succeed("No response body"))
          )
          
          return yield* Effect.fail(new ApiResponseError({
            status: response.status,
            message: body,
            request: {
              method: error.request.method,
              url: error.request.url
            }
          }))
        }
        
        return yield* Effect.fail(error)
      })),
      // Logging
      HttpClient.tap((response) => 
        Logger.debug(`${serviceName} API response`, {
          method: response.request.method,
          url: response.request.url,
          status: response.status,
          duration: response.headers["x-response-time"]
        })
      )
    )
    
    // Helper for JSON responses with schema validation
    const requestJson = <I, A, E, R>(
      method: "GET" | "POST" | "PUT" | "PATCH" | "DELETE",
      path: string,
      schema: Schema.Schema<A, I>,
      options?: {
        body?: unknown
        params?: Record<string, string>
        headers?: Record<string, string>
      }
    ) => Effect.gen(function* () {
      let request = HttpClientRequest.make(method)(path)
      
      if (options?.body) {
        request = HttpClientRequest.bodyJson(request, options.body)
      }
      
      if (options?.params) {
        request = HttpClientRequest.setUrlParams(request, options.params)
      }
      
      if (options?.headers) {
        request = Object.entries(options.headers).reduce(
          (req, [key, value]) => HttpClientRequest.setHeader(req, key, value),
          request
        )
      }
      
      const response = yield* client.execute(request)
      return yield* HttpClientResponse.schemaBodyJson(schema)(response)
    }).pipe(
      Effect.withSpan(`${serviceName}.${method} ${path}`)
    )
    
    return {
      client,
      get: <I, A>(path: string, schema: Schema.Schema<A, I>, params?: Record<string, string>) =>
        requestJson("GET", path, schema, { params }),
      post: <I, A>(path: string, schema: Schema.Schema<A, I>, body: unknown) =>
        requestJson("POST", path, schema, { body }),
      put: <I, A>(path: string, schema: Schema.Schema<A, I>, body: unknown) =>
        requestJson("PUT", path, schema, { body }),
      patch: <I, A>(path: string, schema: Schema.Schema<A, I>, body: unknown) =>
        requestJson("PATCH", path, schema, { body }),
      delete: <I, A>(path: string, schema: Schema.Schema<A, I>) =>
        requestJson("DELETE", path, schema)
    }
  })
  
  return {
    ServiceConfig,
    ServiceConfigLive,
    makeClient,
    layer: Layer.effect(
      Context.GenericTag<ReturnType<typeof makeClient>>(`${serviceName}Client`),
      makeClient
    ), Layer.provide(ServiceConfigLive))
  }
}

// Usage: Create specific API clients
const UserApi = makeApiClientFactory("UserApi", "services.userApi")
const PaymentApi = makeApiClientFactory("PaymentApi", "services.paymentApi")

// Use in application
const program = Effect.gen(function* () {
  const userApi = yield* UserApi.makeClient
  const paymentApi = yield* PaymentApi.makeClient
  
  const User = Schema.Struct({
    id: Schema.String,
    name: Schema.String,
    email: Schema.String
  })
  
  const users = yield* userApi.get("/users", Schema.Array(User))
  const newUser = yield* userApi.post("/users", User, {
    name: "John Doe",
    email: "john@example.com"
  })
  
  return { users, newUser }
}).pipe(
  Effect.provide(UserApi.layer),
  Effect.provide(PaymentApi.layer)
)
```

### Pattern 2: OAuth 2.0 Authentication Flow

Implement a complete OAuth 2.0 authentication flow with token refresh:

```typescript
import { HttpClient, HttpClientRequest, HttpClientResponse } from "@effect/platform"
import { Effect, Ref, Layer, Context, Duration, Schedule, FiberRef } from "effect"

// OAuth token types
const OAuthToken = Schema.Struct({
  access_token: Schema.String,
  token_type: Schema.String,
  expires_in: Schema.Number,
  refresh_token: Schema.optional(Schema.String),
  scope: Schema.optional(Schema.String)
})

const TokenStorage = Schema.Struct({
  token: OAuthToken.Type,
  expiresAt: Schema.DateFromNumber
})

// OAuth configuration
interface OAuthConfig {
  readonly clientId: string
  readonly clientSecret: string
  readonly tokenUrl: string
  readonly authorizeUrl: string
  readonly redirectUri: string
  readonly scopes: ReadonlyArray<string>
}

const OAuthConfig = Context.GenericTag<OAuthConfig>("OAuthConfig")

// OAuth client implementation
class OAuthClient extends Context.Tag("OAuthClient")<
  OAuthClient,
  {
    readonly authorize: (code: string) => Effect.Effect<typeof OAuthToken.Type>
    readonly refresh: Effect.Effect<typeof OAuthToken.Type>
    readonly withAuth: <A, E, R>(
      effect: Effect.Effect<A, E, R>
    ) => Effect.Effect<A, E | HttpClientError.HttpClientError, R>
  }
>() {}

const makeOAuthClient = Effect.gen(function* () {
  const config = yield* OAuthConfig
  const httpClient = yield* HttpClient.HttpClient
  const tokenRef = yield* Ref.make<TokenStorage | null>(null)
  
  // Token management
  const getStoredToken = tokenRef.get.pipe(
    Effect.flatMap(storage => 
      storage === null
        ? Effect.fail(new Error("No token available"))
        : Effect.succeed(storage)
    )
  )
  
  const isTokenExpired = (storage: TokenStorage) => {
    const now = new Date()
    const buffer = 5 * 60 * 1000 // 5 minute buffer
    return now.getTime() >= storage.expiresAt.getTime() - buffer
  }
  
  // Exchange authorization code for token
  const authorize = (code: string) => Effect.gen(function* () {
    const response = yield* httpClient.post(config.tokenUrl).pipe(
      HttpClientRequest.bodyUrlParams({
        grant_type: "authorization_code",
        code,
        client_id: config.clientId,
        client_secret: config.clientSecret,
        redirect_uri: config.redirectUri
      })
    )
    
    const token = yield* HttpClientResponse.schemaBodyJson(OAuthToken)(response)
    
    // Store token with expiration
    yield* tokenRef.set({
      token,
      expiresAt: new Date(Date.now() + token.expires_in * 1000)
    })
    
    return token
  })
  
  // Refresh access token
  const refresh = Effect.gen(function* () {
    const storage = yield* getStoredToken
    
    if (!storage.token.refresh_token) {
      return yield* Effect.fail(new Error("No refresh token available"))
    }
    
    const response = yield* httpClient.post(config.tokenUrl).pipe(
      HttpClientRequest.bodyUrlParams({
        grant_type: "refresh_token",
        refresh_token: storage.token.refresh_token,
        client_id: config.clientId,
        client_secret: config.clientSecret
      })
    )
    
    const token = yield* HttpClientResponse.schemaBodyJson(OAuthToken)(response)
    
    // Update stored token
    yield* tokenRef.set({
      token: {
        ...token,
        refresh_token: storage.token.refresh_token // Preserve refresh token if not returned
      },
      expiresAt: new Date(Date.now() + token.expires_in * 1000)
    })
    
    return token
  }).pipe(
    Effect.retry({
      times: 2,
      schedule: Schedule.exponential(Duration.seconds(1))
    })
  )
  
  // Get valid access token (refresh if needed)
  const getValidToken = Effect.gen(function* () {
    const storage = yield* getStoredToken
    
    if (isTokenExpired(storage)) {
      yield* refresh
      const newStorage = yield* getStoredToken
      return newStorage.token.access_token
    }
    
    return storage.token.access_token
  })
  
  // Create authorization URL
  const getAuthorizeUrl = (state?: string) => {
    const params = new URLSearchParams({
      client_id: config.clientId,
      redirect_uri: config.redirectUri,
      response_type: "code",
      scope: config.scopes.join(" "),
      ...(state ? { state } : {})
    })
    
    return `${config.authorizeUrl}?${params.toString()}`
  }
  
  // Fiber ref for current token
  const CurrentToken = FiberRef.unsafeMake<string | null>(null)
  
  // Execute effect with authentication
  const withAuth = <A, E, R>(
    effect: Effect.Effect<A, E, R>
  ) => Effect.gen(function* () {
    const token = yield* getValidToken
    yield* FiberRef.set(CurrentToken, token)
    
    return yield* effect.pipe(
      Effect.provide(
        Layer.succeed(HttpClient.HttpClient, 
          httpClient.pipe(
            HttpClient.mapRequest((request) => 
              HttpClientRequest.bearerToken(request, token)
            )
          )
        )
      )
    )
  }).pipe(
    Effect.retry({
      while: (error) => {
        // Retry on 401 with token refresh
        if (HttpClientError.isHttpClientError(error) && error.response.status === 401) {
          return Effect.orElse(Effect.as(refresh, true), () => Effect.succeed(false))
        }
        return Effect.succeed(false)
      },
      times: 1
    })
  )
  
  return {
    authorize,
    refresh,
    withAuth,
    getAuthorizeUrl
  }
})

// Create OAuth-enabled HTTP client
const makeOAuthHttpClient = Effect.gen(function* () {
  const oauth = yield* OAuthClient
  const baseClient = yield* HttpClient.HttpClient
  
  return baseClient.pipe(
    HttpClient.transformResponse((response) => Effect.gen(function* () {
      // Automatically retry on 401
      if (response.status === 401) {
        yield* oauth.refresh
        return yield* Effect.fail(HttpClientError.responseError(
          response.request,
          response,
          "Token expired"
        ))
      }
      return response
    }))
  )
})

// Usage example
const githubApiExample = Effect.gen(function* () {
  const oauth = yield* OAuthClient
  
  // Use OAuth-protected endpoint
  const repos = yield* oauth.withAuth(
    Effect.gen(function* () {
      const client = yield* HttpClient.HttpClient
      const response = yield* client.get("https://api.github.com/user/repos")
      return yield* response.json
    })
  )
  
  return repos
}).pipe(
  Effect.provide(Layer.succeed(OAuthConfig, {
    clientId: "your-client-id",
    clientSecret: "your-client-secret",
    tokenUrl: "https://github.com/login/oauth/access_token",
    authorizeUrl: "https://github.com/login/oauth/authorize",
    redirectUri: "http://localhost:3000/callback",
    scopes: ["repo", "user"]
  })),
  Effect.provide(Layer.effect(OAuthClient, makeOAuthClient))
)
```

## Integration Examples

### Integration with Express.js

Create an Express middleware that uses Effect HttpClient:

```typescript
import { HttpClient, HttpClientRequest } from "@effect/platform"
import { NodeHttpClient, NodeRuntime } from "@effect/platform-node"
import { Effect, Layer, Exit } from "effect"
import express from "express"

// Effect HTTP client middleware
const createEffectHttpMiddleware = (baseUrl: string) => {
  const HttpClientLive = NodeHttpClient.layer.pipe(
    Layer.provide(Layer.succeed(HttpClient.HttpClient, 
      HttpClient.HttpClient.pipe(
        HttpClient.mapRequest(HttpClientRequest.prependUrl(baseUrl))
      )
    ))
  )
  
  return (req: express.Request, res: express.Response, next: express.NextFunction) => {
    // Attach Effect HTTP client to request
    req.httpClient = {
      request: <A>(effect: Effect.Effect<A, any, HttpClient.HttpClient>) =>
        effect.pipe(
          Effect.provide(HttpClientLive),
          NodeRuntime.runPromiseExit
        ).then(exit => {
          if (Exit.isFailure(exit)) {
            throw exit.cause
          }
          return exit.value
        })
    }
    next()
  }
}

// Extend Express types
declare global {
  namespace Express {
    interface Request {
      httpClient: {
        request: <A>(effect: Effect.Effect<A, any, HttpClient.HttpClient>) => Promise<A>
      }
    }
  }
}

// Express app with Effect HttpClient
const app = express()
app.use(express.json())
app.use(createEffectHttpMiddleware("https://api.example.com"))

// Route using Effect HttpClient
app.get("/proxy/users/:id", async (req, res) => {
  try {
    const User = Schema.Struct({
      id: Schema.String,
      name: Schema.String,
      email: Schema.String
    })
    
    const user = await req.httpClient.request(
      Effect.gen(function* () {
        const client = yield* HttpClient.HttpClient
        const response = yield* client.get(`/users/${req.params.id}`)
        return yield* HttpClientResponse.schemaBodyJson(User)(response)
      })
    )
    
    res.json(user)
  } catch (error) {
    res.status(500).json({ error: "Failed to fetch user" })
  }
})

// WebSocket proxy using Effect
import { WebSocketServer } from "ws"

const createWebSocketProxy = (targetUrl: string) => {
  const wss = new WebSocketServer({ noServer: true })
  
  wss.on("connection", (ws, req) => {
    const program = Effect.gen(function* () {
      const client = yield* HttpClient.HttpClient
      
      // Create WebSocket connection to target
      const targetWs = new WebSocket(targetUrl)
      
      // Proxy messages
      ws.on("message", (data) => {
        if (targetWs.readyState === WebSocket.OPEN) {
          targetWs.send(data)
        }
      })
      
      targetWs.on("message", (data) => {
        if (ws.readyState === WebSocket.OPEN) {
          ws.send(data)
        }
      })
      
      // Handle errors
      const handleError = (error: Error) => {
        console.error("WebSocket error:", error)
        ws.close()
        targetWs.close()
      }
      
      ws.on("error", handleError)
      targetWs.on("error", handleError)
      
      // Clean up on close
      ws.on("close", () => targetWs.close())
      targetWs.on("close", () => ws.close())
    })
    
    NodeRuntime.runFork(program.pipe(
      Effect.provide(NodeHttpClient.layer)
    ))
  })
  
  return wss
}
```

### Testing Strategies

Comprehensive testing approaches for HttpClient:

```typescript
import { HttpClient, HttpClientRequest, HttpClientResponse } from "@effect/platform"
import { Effect, Layer, TestClock, TestContext, Ref } from "effect"
import { describe, it, expect } from "vitest"

// Test utilities
const TestHttpClient = {
  // Create a mock HTTP client with predefined responses
  make: (responses: Map<string, () => Effect.Effect<HttpClientResponse.HttpClientResponse>>) =>
    HttpClient.make((request, url) => {
      const key = `${request.method} ${url.toString()}`
      const response = responses.get(key)
      
      if (!response) {
        return Effect.fail(HttpClientError.requestError(
          request,
          "No mock response defined"
        ))
      }
      
      return response()
    }),
  
  // Create a recording HTTP client
  makeRecording: () => Effect.gen(function* () {
    const requests = yield* Ref.make<Array<HttpClientRequest.HttpClientRequest>>([])
    
    const client = HttpClient.make((request) => {
      return Effect.gen(function* () {
        yield* requests.update(reqs => [...reqs, request])
        
        // Return a default response
        return HttpClientResponse.fromWeb(
          request,
          new Response(JSON.stringify({ mocked: true }), {
            status: 200,
            headers: { "Content-Type": "application/json" }
          })
        )
      })
    })
    
    return {
      client,
      getRequests: requests.get,
      clearRequests: requests.set([])
    }
  })
}

// Test example
describe("UserService", () => {
  it("should fetch user with retry on failure", () => 
    Effect.gen(function* () {
      // Setup test clock for controlling time
      yield* TestClock.adjust(Duration.zero)
      
      // Track retry attempts
      const attempts = yield* Ref.make(0)
      
      // Create mock client that fails twice then succeeds
      const mockClient = TestHttpClient.make(new Map([
        ["GET https://api.example.com/users/123", () => 
          Effect.gen(function* () {
            const attempt = yield* attempts.updateAndGet(n => n + 1)
            
            if (attempt < 3) {
              return yield* Effect.fail(HttpClientError.requestError(
                HttpClientRequest.get("/users/123"),
                "Network error"
              ))
            }
            
            return HttpClientResponse.fromWeb(
              HttpClientRequest.get("/users/123"),
              new Response(JSON.stringify({
                id: "123",
                name: "John Doe",
                email: "john@example.com"
              }), {
                status: 200,
                headers: { "Content-Type": "application/json" }
              })
            )
          })
        ]
      ]))
      
      // Run service with mock client
      const user = yield* getUserWithRetry("123").pipe(
        Effect.provide(Layer.succeed(HttpClient.HttpClient, mockClient))
      )
      
      // Verify result
      expect(user).toEqual({
        id: "123",
        name: "John Doe",
        email: "john@example.com"
      })
      
      // Verify retry attempts
      const finalAttempts = yield* attempts.get
      expect(finalAttempts).toBe(3)
    }).pipe(
      Effect.provide(TestContext.TestContext),
      Effect.runPromise
    )
  )
  
  it("should validate response schema", () =>
    Effect.gen(function* () {
      // Mock invalid response
      const mockClient = TestHttpClient.make(new Map([
        ["GET https://api.example.com/users/123", () =>
          Effect.succeed(
            HttpClientResponse.fromWeb(
              HttpClientRequest.get("/users/123"),
              new Response(JSON.stringify({
                id: "123",
                // Missing required fields
              }), {
                status: 200,
                headers: { "Content-Type": "application/json" }
              })
            )
          )
        ]
      ]))
      
      // Expect schema validation to fail
      const result = yield* Effect.either(
        getUser("123").pipe(
          Effect.provide(Layer.succeed(HttpClient.HttpClient, mockClient))
        )
      )
      
      expect(Exit.isFailure(result)).toBe(true)
    }), Effect.runPromise)
  )
})

// Integration test with real HTTP
describe("HttpClient Integration", () => {
  it("should handle real HTTP requests", () =>
    Effect.gen(function* () {
      const client = yield* HttpClient.HttpClient
      
      // Use httpbin.org for testing
      const response = yield* client.post("https://httpbin.org/post").pipe(
        HttpClientRequest.bodyJson({ test: "data" })
      )
      
      const body = yield* response.json as Effect.Effect<any>
      
      expect(body.json).toEqual({ test: "data" })
      expect(response.status).toBe(200)
    }).pipe(
      Effect.provide(FetchHttpClient.layer),
      Effect.runPromise
    )
  )
})

// Property-based testing
import * as fc from "fast-check"

describe("HttpClient Properties", () => {
  it("should maintain request/response correlation", () =>
    fc.assert(
      fc.asyncProperty(
        fc.record({
          method: fc.constantFrom("GET", "POST", "PUT", "DELETE"),
          path: fc.stringMatching(/^\/[a-z]+$/),
          headers: fc.dictionary(
            fc.stringMatching(/^[A-Za-z-]+$/),
            fc.string()
          )
        }),
        async (requestData) => {
          const program = Effect.gen(function* () {
            const recording = yield* TestHttpClient.makeRecording()
            
            // Make request
            yield* recording.client.execute(
              HttpClientRequest.make(requestData.method)(requestData.path).pipe(
                HttpClientRequest.setHeaders(requestData.headers)
              )
            )
            
            // Verify request was recorded correctly
            const requests = yield* recording.getRequests()
            expect(requests).toHaveLength(1)
            expect(requests[0].method).toBe(requestData.method)
            expect(requests[0].url).toContain(requestData.path)
          })
          
          await Effect.runPromise(program)
        }
      )
    )
  )
})
```

## Conclusion

HttpClient provides a powerful, type-safe, and composable solution for HTTP communication in Effect applications. 

Key benefits:
- **Type Safety**: Full TypeScript support with schema validation ensures runtime correctness
- **Composability**: Build complex HTTP workflows by combining simple Effect primitives
- **Built-in Features**: Retry strategies, timeout handling, request/response transformation, and tracing come standard
- **Error Handling**: Structured error types with full context for debugging
- **Cross-Platform**: Works seamlessly in Node.js, Bun, and browsers with platform-specific optimizations
- **Testing Support**: Easy to mock and test with provided utilities

HttpClient eliminates the boilerplate and complexity of traditional HTTP clients while providing enterprise-grade features for building robust, production-ready applications.