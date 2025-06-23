# Request: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem Request Solves

Modern applications constantly fetch data from external sources - databases, APIs, file systems, and remote services. Traditional approaches to data fetching often lead to inefficient, error-prone patterns:

```typescript
// Traditional approach - ad-hoc data fetching
const fetchUser = async (id: number) => {
  // No batching - N+1 query problem
  const response = await fetch(`/api/users/${id}`)
  return response.json()
}

const fetchUserPosts = async (userId: number) => {
  // Duplicate requests not deduplicated
  const response = await fetch(`/api/users/${userId}/posts`)
  return response.json()
}

// Manual cache management
const cache = new Map()
const fetchUserWithCache = async (id: number) => {
  if (cache.has(id)) return cache.get(id)
  const user = await fetchUser(id)
  cache.set(id, user)
  return user
}
```

This approach leads to:
- **N+1 Query Problems** - Multiple individual requests instead of batched queries
- **Request Duplication** - Same data fetched multiple times concurrently
- **Manual Caching** - Complex cache invalidation and consistency issues
- **Poor Error Handling** - No unified error handling across data sources
- **No Request Optimization** - Cannot analyze and optimize request patterns

### The Request Solution

Request provides a declarative abstraction for data fetching with automatic batching, deduplication, and caching:

```typescript
import { Effect, Request, RequestResolver } from "effect"

// Define what data you want - not how to get it
class GetUser extends Request.Class<User, UserNotFoundError, {
  readonly id: number
}> {}

class GetUserPosts extends Request.Class<ReadonlyArray<Post>, PostFetchError, {
  readonly userId: number
}> {}

// Define how to resolve requests efficiently
const userResolver = RequestResolver.makeBatched((requests: ReadonlyArray<GetUser>) =>
  Effect.gen(function* () {
    // Automatically batched - single query for multiple users
    const users = yield* database.getUsersByIds(requests.map(r => r.id))
    
    // Complete all requests in the batch
    yield* Effect.forEach(requests, (request) => {
      const user = users.find(u => u.id === request.id)
      return user 
        ? Request.succeed(request, user)
        : Request.fail(request, new UserNotFoundError({ id: request.id }))
    }, { discard: true })
  })
)
```

### Key Concepts

**Request**: A declarative description of data you want to fetch, containing the request parameters and type information for success/error cases

**RequestResolver**: The implementation that knows how to efficiently fetch the requested data, with automatic batching and error handling

**Automatic Batching**: Multiple requests of the same type are automatically collected and executed together for efficiency

**Request Deduplication**: Identical concurrent requests are deduplicated and share the same result

## Basic Usage Patterns

### Pattern 1: Simple Request Definition

```typescript
import { Request, Effect } from "effect"

// Define a request for user data
class GetUser extends Request.Class<User, UserNotFoundError, {
  readonly id: number
}> {}

// Usage - declarative data fetching
const getUser = (id: number) => new GetUser({ id })

// Type-safe request with inferred success/error types
const userEffect: Effect.Effect<User, UserNotFoundError> = 
  Effect.request(getUser(123), userResolver)
```

### Pattern 2: Tagged Requests with Schema

```typescript
import { Schema } from "effect"

// Using Schema's TaggedRequest for serializable requests
class GetUserById extends Schema.TaggedRequest<GetUserById>()("GetUserById", {
  failure: Schema.String, // Error type
  success: User.schema,   // Success type schema
  payload: {
    id: Schema.Number     // Request parameters
  }
}) {}

// Enhanced with serialization and validation
const request = new GetUserById({ id: 42 })
```

### Pattern 3: Basic RequestResolver

```typescript
import { RequestResolver, Effect } from "effect"

// Simple resolver for individual requests
const getUserResolver = RequestResolver.fromEffect((request: GetUser) =>
  Effect.gen(function* () {
    const user = yield* database.findUser(request.id)
    if (!user) {
      return yield* Effect.fail(new UserNotFoundError({ id: request.id }))
    }
    return user
  })
)

// Execute the request
const program = Effect.gen(function* () {
  const user = yield* Effect.request(new GetUser({ id: 1 }), getUserResolver)
  console.log(`Found user: ${user.name}`)
})
```

## Real-World Examples

### Example 1: E-commerce Product Catalog

```typescript
import { Effect, Request, RequestResolver, Schema } from "effect"

// Domain models
class Product extends Schema.Class<Product>("Product")({
  id: Schema.Number,
  name: Schema.String,
  price: Schema.Number,
  categoryId: Schema.Number
}) {}

class Category extends Schema.Class<Category>("Category")({
  id: Schema.Number,
  name: Schema.String
}) {}

// Request definitions
class GetProduct extends Request.Class<Product, ProductNotFoundError, {
  readonly id: number
}> {}

class GetCategory extends Request.Class<Category, CategoryNotFoundError, {
  readonly id: number
}> {}

class GetProductsByCategory extends Request.Class<ReadonlyArray<Product>, never, {
  readonly categoryId: number
}> {}

// Efficient batched resolvers
const productResolver = RequestResolver.makeBatched((requests: ReadonlyArray<GetProduct>) =>
  Effect.gen(function* () {
    const productIds = requests.map(r => r.id)
    const products = yield* database.products.findByIds(productIds)
    
    yield* Effect.forEach(requests, (request) => {
      const product = products.find(p => p.id === request.id)
      return product
        ? Request.succeed(request, product)
        : Request.fail(request, new ProductNotFoundError({ id: request.id }))
    }, { discard: true })
  })
)

const categoryResolver = RequestResolver.makeBatched((requests: ReadonlyArray<GetCategory>) =>
  Effect.gen(function* () {
    const categoryIds = requests.map(r => r.id)
    const categories = yield* database.categories.findByIds(categoryIds)
    
    yield* Effect.forEach(requests, (request) => {
      const category = categories.find(c => c.id === request.id)
      return category
        ? Request.succeed(request, category)
        : Request.fail(request, new CategoryNotFoundError({ id: request.id }))
    }, { discard: true })
  })
)

const productsByCategoryResolver = RequestResolver.fromEffect((request: GetProductsByCategory) =>
  database.products.findByCategory(request.categoryId)
)

// Business logic with automatic batching
const getProductWithCategory = (productId: number) =>
  Effect.gen(function* () {
    // These requests are automatically batched and cached
    const product = yield* Effect.request(new GetProduct({ id: productId }), productResolver)
    const category = yield* Effect.request(new GetCategory({ id: product.categoryId }), categoryResolver)
    
    return { product, category }
  })

// Usage - automatically optimizes multiple concurrent requests
const catalogPage = Effect.gen(function* () {
  // All products fetched in a single batch
  // All categories fetched in a single batch
  const productDetails = yield* Effect.forEach(
    [1, 2, 3, 4, 5], 
    getProductWithCategory,
    { concurrency: "unbounded" }
  )
  
  return productDetails
})
```

### Example 2: Social Media Feed System

```typescript
import { Effect, Request, RequestResolver, Array as Arr } from "effect"

// Request definitions for social media domain
class GetUser extends Request.Class<User, UserNotFoundError, {
  readonly id: string
}> {}

class GetPost extends Request.Class<Post, PostNotFoundError, {
  readonly id: string
}> {}

class GetUserFollowers extends Request.Class<ReadonlyArray<string>, never, {
  readonly userId: string
}> {}

class GetUserPosts extends Request.Class<ReadonlyArray<Post>, never, {
  readonly userId: string
  readonly limit: number
}> {}

// Resolvers with different batching strategies
const userResolver = RequestResolver.makeBatched((requests: ReadonlyArray<GetUser>) =>
  Effect.gen(function* () {
    // Single database query for all requested users
    const userIds = requests.map(r => r.id)
    const users = yield* userService.getBatch(userIds)
    
    yield* Effect.forEach(requests, (request) => {
      const user = users.find(u => u.id === request.id)
      return user
        ? Request.succeed(request, user)
        : Request.fail(request, new UserNotFoundError({ userId: request.id }))
    }, { discard: true })
  })
)

const postResolver = RequestResolver.makeBatched((requests: ReadonlyArray<GetPost>) =>
  Effect.gen(function* () {
    const postIds = requests.map(r => r.id)
    const posts = yield* postService.getBatch(postIds)
    
    yield* Effect.forEach(requests, (request) => {
      const post = posts.find(p => p.id === request.id)
      return post
        ? Request.succeed(request, post)
        : Request.fail(request, new PostNotFoundError({ postId: request.id }))
    }, { discard: true })
  })
)

// Individual request resolvers for complex queries
const userFollowersResolver = RequestResolver.fromEffect((request: GetUserFollowers) =>
  socialGraphService.getFollowers(request.userId)
)

const userPostsResolver = RequestResolver.makeBatched((requests: ReadonlyArray<GetUserPosts>) =>
  Effect.gen(function* () {
    // Group by limit for efficient batching
    const requestsByLimit = Arr.groupBy(requests, r => r.limit)
    
    yield* Effect.forEach(Object.entries(requestsByLimit), ([limit, reqs]) => {
      const userIds = reqs.map(r => r.userId)
      return Effect.gen(function* () {
        const postsMap = yield* postService.getUserPostsBatch(userIds, Number(limit))
        
        yield* Effect.forEach(reqs, (request) => {
          const posts = postsMap.get(request.userId) ?? []
          return Request.succeed(request, posts)
        }, { discard: true })
      })
    }, { discard: true })
  })
)

// Complex business logic with automatic optimization
const buildUserFeed = (userId: string, limit: number = 10) =>
  Effect.gen(function* () {
    // Get user's followers (single request)
    const followers = yield* Effect.request(
      new GetUserFollowers({ userId }), 
      userFollowersResolver
    )
    
    // Get posts from followed users (automatically batched by limit)
    const followerPosts = yield* Effect.forEach(
      followers.slice(0, 20), // Limit to recent followers
      (followerId) => Effect.request(
        new GetUserPosts({ userId: followerId, limit: 5 }),
        userPostsResolver
      ),
      { concurrency: "unbounded" }
    )
    
    // Flatten and sort posts
    const allPosts = followerPosts.flat().sort((a, b) => 
      new Date(b.createdAt).getTime() - new Date(a.createdAt).getTime()
    )
    
    // Get author details for posts (automatically batched)
    const postsWithAuthors = yield* Effect.forEach(
      allPosts.slice(0, limit),
      (post) => Effect.gen(function* () {
        const author = yield* Effect.request(
          new GetUser({ id: post.authorId }),
          userResolver
        )
        return { post, author }
      }),
      { concurrency: "unbounded" }
    )
    
    return postsWithAuthors
  })
```

### Example 3: GraphQL-Style Data Loading

```typescript
import { Effect, Request, RequestResolver, Option } from "effect"

// Nested data structure requests
class GetOrder extends Request.Class<Order, OrderNotFoundError, {
  readonly id: string
}> {}

class GetCustomer extends Request.Class<Customer, CustomerNotFoundError, {
  readonly id: string
}> {}

class GetOrderItems extends Request.Class<ReadonlyArray<OrderItem>, never, {
  readonly orderId: string
}> {}

class GetProduct extends Request.Class<Product, ProductNotFoundError, {
  readonly id: string
}> {}

// DataLoader pattern for GraphQL-like field resolution
const createDataLoaders = () => {
  const orderResolver = RequestResolver.makeBatched((requests: ReadonlyArray<GetOrder>) =>
    Effect.gen(function* () {
      const orderIds = requests.map(r => r.id)
      const orders = yield* orderService.findByIds(orderIds)
      
      yield* Effect.forEach(requests, (request) => {
        const order = orders.find(o => o.id === request.id)
        return order
          ? Request.succeed(request, order)
          : Request.fail(request, new OrderNotFoundError({ orderId: request.id }))
      }, { discard: true })
    })
  )

  const customerResolver = RequestResolver.makeBatched((requests: ReadonlyArray<GetCustomer>) =>
    Effect.gen(function* () {
      const customerIds = requests.map(r => r.id)
      const customers = yield* customerService.findByIds(customerIds)
      
      yield* Effect.forEach(requests, (request) => {
        const customer = customers.find(c => c.id === request.id)
        return customer
          ? Request.succeed(request, customer)
          : Request.fail(request, new CustomerNotFoundError({ customerId: request.id }))
      }, { discard: true })
    })
  )

  return { orderResolver, customerResolver }
}

// GraphQL-style field resolvers
const resolveOrderWithDetails = (orderId: string) =>
  Effect.gen(function* () {
    const { orderResolver, customerResolver } = createDataLoaders()
    
    // Get base order
    const order = yield* Effect.request(new GetOrder({ id: orderId }), orderResolver)
    
    // Resolve nested fields in parallel
    const [customer, orderItems] = yield* Effect.all([
      Effect.request(new GetCustomer({ id: order.customerId }), customerResolver),
      Effect.request(new GetOrderItems({ orderId }), orderItemsResolver)
    ])
    
    // Resolve product details for each item
    const itemsWithProducts = yield* Effect.forEach(
      orderItems,
      (item) => Effect.gen(function* () {
        const product = yield* Effect.request(
          new GetProduct({ id: item.productId }),
          productResolver
        )
        return { ...item, product }
      }),
      { concurrency: "unbounded" }
    )
    
    return {
      order,
      customer,
      items: itemsWithProducts
    }
  })
```

## Advanced Features Deep Dive

### Feature 1: Request Caching and Deduplication

Request automatically handles caching and deduplication to prevent redundant work:

```typescript
import { Effect, Request, RequestResolver } from "effect"

class ExpensiveComputation extends Request.Class<number, never, {
  readonly input: number
}> {}

const expensiveResolver = RequestResolver.fromEffect((request: ExpensiveComputation) =>
  Effect.gen(function* () {
    console.log(`Computing for input: ${request.input}`)
    yield* Effect.sleep("1 second") // Simulate expensive work
    return request.input * request.input
  })
)

// Deduplication example
const deduplicationDemo = Effect.gen(function* () {
  const request1 = new ExpensiveComputation({ input: 5 })
  const request2 = new ExpensiveComputation({ input: 5 }) // Same parameters
  
  // Both requests execute concurrently but only one computation happens
  const [result1, result2] = yield* Effect.all([
    Effect.request(request1, expensiveResolver),
    Effect.request(request2, expensiveResolver)
  ])
  
  console.log(`Results: ${result1}, ${result2}`) // Both are 25
  // Only logs "Computing for input: 5" once
})

// Cache configuration
const cachedProgram = Effect.gen(function* () {
  // First request - computed and cached
  const result1 = yield* Effect.request(
    new ExpensiveComputation({ input: 10 }),
    expensiveResolver
  )
  
  // Second request - served from cache
  const result2 = yield* Effect.request(
    new ExpensiveComputation({ input: 10 }),
    expensiveResolver
  )
  
  return [result1, result2]
}).pipe(
  Effect.withRequestCaching(true) // Enable caching
)
```

### Feature 2: Advanced Batching Strategies

Control how requests are batched for optimal performance:

```typescript
import { RequestResolver, Effect, Array as Arr } from "effect"

// Custom batching with size limits
const limitedBatchResolver = RequestResolver.makeBatched((requests: ReadonlyArray<GetUser>) =>
  Effect.gen(function* () {
    // Process in chunks to avoid overwhelming the database
    const chunks = Arr.chunksOf(requests, 50) // Max 50 requests per batch
    
    yield* Effect.forEach(chunks, (chunk) =>
      Effect.gen(function* () {
        const userIds = chunk.map(r => r.id)
        const users = yield* database.getUsersBatch(userIds)
        
        yield* Effect.forEach(chunk, (request) => {
          const user = users.find(u => u.id === request.id)
          return user
            ? Request.succeed(request, user)
            : Request.fail(request, new UserNotFoundError({ id: request.id }))
        }, { discard: true })
      }),
      { concurrency: 3 } // Process 3 chunks concurrently
    , { discard: true })
  })
)

// Conditional batching based on request properties
const smartBatchingResolver = RequestResolver.makeBatched((requests: ReadonlyArray<GetUserPosts>) =>
  Effect.gen(function* () {
    // Group requests by similar characteristics for efficient batching
    const grouped = Arr.groupBy(requests, (r) => `${r.limit}-${r.includePrivate}`)
    
    yield* Effect.forEach(Object.values(grouped), (group) =>
      Effect.gen(function* () {
        const userIds = group.map(r => r.userId)
        const config = { 
          limit: group[0].limit, 
          includePrivate: group[0].includePrivate 
        }
        
        const postsMap = yield* postService.getUserPostsBatch(userIds, config)
        
        yield* Effect.forEach(group, (request) => {
          const posts = postsMap.get(request.userId) ?? []
          return Request.succeed(request, posts)
        }, { discard: true })
      }),
      { concurrency: 5 }
    , { discard: true })
  })
)

// Batch size optimization
const optimizedResolver = RequestResolver.makeBatched((requests: ReadonlyArray<GetUser>) =>
  Effect.gen(function* () {
    const batchSize = Math.min(requests.length, 100) // Optimal batch size
    const chunks = Arr.chunksOf(requests, batchSize)
    
    yield* Effect.forEach(chunks, processBatch, { 
      concurrency: Math.ceil(requests.length / batchSize)
    }, { discard: true })
  })
).pipe(
  RequestResolver.batchN(200) // Maximum concurrent requests
)
```

### Feature 3: Error Handling and Retries

Sophisticated error handling patterns for reliable data fetching:

```typescript
import { Effect, Request, RequestResolver, Schedule } from "effect"

// Error types
class ServiceUnavailableError extends Error {
  readonly _tag = "ServiceUnavailableError"
}

class RateLimitError extends Error {
  readonly _tag = "RateLimitError"
  constructor(readonly retryAfter: number) { super() }
}

// Resilient resolver with retries
const resilientResolver = RequestResolver.fromEffect((request: GetUser) =>
  Effect.gen(function* () {
    const user = yield* userService.getUser(request.id)
    return user
  }).pipe(
    // Retry on transient failures
    Effect.retry(
      Schedule.exponential("100 millis").pipe(
        Schedule.intersect(Schedule.recurs(3)),
        Schedule.whileInput((error) => 
          error._tag === "ServiceUnavailableError" ||
          error._tag === "NetworkError"
        )
      )
    ),
    // Handle rate limiting
    Effect.catchTag("RateLimitError", (error) =>
      Effect.sleep(`${error.retryAfter} seconds`).pipe(
        Effect.flatMap(() => userService.getUser(request.id))
      )
    )
  )
)

// Partial failure handling in batched requests
const robustBatchResolver = RequestResolver.makeBatched((requests: ReadonlyArray<GetUser>) =>
  Effect.gen(function* () {
    const userIds = requests.map(r => r.id)
    
    // Attempt batch request with fallback to individual requests
    const batchResult = yield* userService.getUsersBatch(userIds).pipe(
      Effect.catchAll((batchError) => 
        // Fallback to individual requests if batch fails
        Effect.forEach(userIds, (id) => 
          userService.getUser(id).pipe(
            Effect.map(user => ({ id, user: Option.some(user) })),
            Effect.catchAll(() => Effect.succeed({ id, user: Option.none() }))
          )
        ).pipe(
          Effect.map(results => 
            results.reduce((acc, { id, user }) => {
              if (Option.isSome(user)) {
                acc.set(id, user.value)
              }
              return acc
            }, new Map<number, User>())
          )
        )
      )
    )
    
    // Complete requests with results or appropriate errors
    yield* Effect.forEach(requests, (request) => {
      const user = batchResult.get(request.id)
      return user
        ? Request.succeed(request, user)
        : Request.fail(request, new UserNotFoundError({ id: request.id }))
    }, { discard: true })
  })
)
```

## Practical Patterns & Best Practices

### Pattern 1: Request Composition and Dependency Resolution

```typescript
// Helper for dependent requests
const withDependencies = <A, B, E1, E2>(
  firstRequest: Effect.Effect<A, E1>,
  dependentRequest: (a: A) => Effect.Effect<B, E2>
): Effect.Effect<B, E1 | E2> =>
  Effect.gen(function* () {
    const first = yield* firstRequest
    const second = yield* dependentRequest(first)
    return second
  })

// Complex dependency resolution
const resolveUserProfile = (userId: string) =>
  Effect.gen(function* () {
    // Primary user data
    const user = yield* Effect.request(new GetUser({ id: userId }), userResolver)
    
    // Dependent requests based on user data
    const [preferences, recentActivity, socialConnections] = yield* Effect.all([
      Effect.request(
        new GetUserPreferences({ userId: user.id }),
        preferencesResolver
      ),
      Effect.request(
        new GetRecentActivity({ userId: user.id, days: 30 }),
        activityResolver
      ),
      Effect.request(
        new GetSocialConnections({ userId: user.id }),
        socialResolver
      )
    ])
    
    // Further dependent requests
    const connectionDetails = yield* Effect.forEach(
      socialConnections.slice(0, 10), // Limit for performance
      (connection) => Effect.request(
        new GetUser({ id: connection.userId }),
        userResolver
      ),
      { concurrency: 5 }
    )
    
    return {
      user,
      preferences,
      recentActivity,
      socialConnections: connectionDetails
    }
  })
```

### Pattern 2: Request Factory Pattern

```typescript
// Request factory for complex request creation
const createRequestFactory = <TRequest, TParams>(
  RequestClass: new (params: TParams) => TRequest,
  resolver: RequestResolver<TRequest>
) => {
  return {
    // Single request
    get: (params: TParams) => Effect.request(new RequestClass(params), resolver),
    
    // Batch requests
    getMany: (paramsArray: ReadonlyArray<TParams>) =>
      Effect.forEach(
        paramsArray,
        (params) => Effect.request(new RequestClass(params), resolver),
        { concurrency: "unbounded" }
      ),
    
    // Optional request (doesn't fail on not found)
    getOptional: (params: TParams) =>
      Effect.request(new RequestClass(params), resolver).pipe(
        Effect.option
      ),
    
    // Cached request with TTL
    getCached: (params: TParams, ttl: Duration.DurationInput) =>
      Effect.request(new RequestClass(params), resolver).pipe(
        Effect.cached(ttl)
      )
  }
}

// Usage
const userRequests = createRequestFactory(GetUser, userResolver)
const productRequests = createRequestFactory(GetProduct, productResolver)

const example = Effect.gen(function* () {
  // Clean, consistent API
  const user = yield* userRequests.get({ id: "123" })
  const products = yield* productRequests.getMany([
    { id: "p1" }, { id: "p2" }, { id: "p3" }
  ])
  const optionalUser = yield* userRequests.getOptional({ id: "maybe-exists" })
  
  return { user, products, optionalUser }
})
```

### Pattern 3: Request Middleware and Instrumentation

```typescript
import { Effect, RequestResolver, Metrics, Logger } from "effect"

// Request timing middleware
const withTiming = <A, R>(
  resolver: RequestResolver<A, R>,
  name: string
): RequestResolver<A, R> =>
  RequestResolver.aroundRequests(
    resolver,
    (requests) => Effect.gen(function* () {
      const startTime = yield* Effect.sync(() => Date.now())
      yield* Logger.info(`Starting ${name} batch`, { 
        requestCount: requests.length 
      })
      return { startTime, requestCount: requests.length }
    }),
    (requests, { startTime, requestCount }) => Effect.gen(function* () {
      const duration = Date.now() - startTime
      
      yield* Logger.info(`Completed ${name} batch`, {
        requestCount,
        duration: `${duration}ms`
      })
      
      // Record metrics
      yield* Metrics.counter(`${name}_requests_total`).pipe(
        Metrics.increment(requestCount)
      )
      
      yield* Metrics.histogram(`${name}_duration_ms`).pipe(
        Metrics.update(duration)
      )
    })
  )

// Request validation middleware
const withValidation = <A extends Request.Request<any, any>, R>(
  resolver: RequestResolver<A, R>,
  validate: (request: A) => Effect.Effect<void, ValidationError>
): RequestResolver<A, R> =>
  RequestResolver.aroundRequests(
    resolver,
    (requests) => Effect.gen(function* () {
      // Validate all requests before processing
      yield* Effect.forEach(requests, validate, { discard: true })
    }),
    () => Effect.void
  )

// Circuit breaker pattern
const withCircuitBreaker = <A, R>(
  resolver: RequestResolver<A, R>,
  options: {
    readonly maxFailures: number
    readonly resetTimeout: Duration.DurationInput
  }
): Effect.Effect<RequestResolver<A, R>, never, Scope.Scope> =>
  Effect.gen(function* () {
    const state = yield* Ref.make<{
      failures: number
      lastFailureTime: number
      isOpen: boolean
    }>({ failures: 0, lastFailureTime: 0, isOpen: false })
    
    return RequestResolver.around(
      resolver,
      Effect.gen(function* () {
        const current = yield* Ref.get(state)
        
        if (current.isOpen) {
          const elapsed = Date.now() - current.lastFailureTime
          if (elapsed < Duration.toMillis(options.resetTimeout)) {
            return yield* Effect.fail(new CircuitBreakerOpenError())
          }
          // Reset circuit breaker
          yield* Ref.set(state, { failures: 0, lastFailureTime: 0, isOpen: false })
        }
      }),
      (result) => Effect.gen(function* () {
        if (Effect.isFailure(result)) {
          yield* Ref.update(state, (current) => ({
            failures: current.failures + 1,
            lastFailureTime: Date.now(),
            isOpen: current.failures + 1 >= options.maxFailures
          }))
        } else {
          // Reset on success
          yield* Ref.set(state, { failures: 0, lastFailureTime: 0, isOpen: false })
        }
      })
    )
  })

// Combine middleware
const instrumentedResolver = Effect.gen(function* () {
  const baseResolver = userResolver
  const timedResolver = withTiming(baseResolver, "user")
  const validatedResolver = withValidation(timedResolver, validateUserRequest)
  const circuitBreakerResolver = yield* withCircuitBreaker(validatedResolver, {
    maxFailures: 5,
    resetTimeout: "30 seconds"
  })
  
  return circuitBreakerResolver
})
```

## Integration Examples

### Integration with HTTP APIs

```typescript
import { Effect, Request, RequestResolver, HttpClient, HttpClientRequest } from "effect"

// API request models
class GetGitHubUser extends Request.Class<GitHubUser, GitHubError, {
  readonly username: string
}> {}

class GetGitHubRepos extends Request.Class<ReadonlyArray<GitHubRepo>, GitHubError, {
  readonly username: string
}> {}

// HTTP client-based resolver
const gitHubResolver = RequestResolver.makeBatched((requests: ReadonlyArray<GetGitHubUser>) =>
  Effect.gen(function* () {
    const httpClient = yield* HttpClient.HttpClient
    
    // Batch requests using GitHub's batch API or parallel requests
    const responses = yield* Effect.forEach(
      requests,
      (request) => HttpClientRequest.get(`https://api.github.com/users/${request.username}`).pipe(
        httpClient,
        Effect.flatMap(response => response.json),
        Effect.mapError(() => new GitHubError({ message: "User not found" })),
        Effect.map(user => ({ request, user }))
      ),
      { concurrency: 10 } // GitHub rate limit consideration
    )
    
    // Complete all requests
    yield* Effect.forEach(responses, ({ request, user }) =>
      Request.succeed(request, user as GitHubUser)
    , { discard: true })
  })
)

// REST API integration with error handling
const restApiResolver = <TRequest extends Request.Request<any, any>>(
  endpoint: (request: TRequest) => string,
  transform: (data: unknown) => Request.Request.Success<TRequest>
) => RequestResolver.fromEffect((request: TRequest) =>
  Effect.gen(function* () {
    const httpClient = yield* HttpClient.HttpClient
    
    const response = yield* HttpClientRequest.get(endpoint(request)).pipe(
      HttpClient.filterStatusOk(httpClient),
      Effect.flatMap(response => response.json),
      Effect.mapError((error) => new ApiError({ cause: error }))
    )
    
    return transform(response)
  })
)
```

### Integration with Database Systems

```typescript
import { Effect, Request, RequestResolver, SqlClient } from "@effect/sql"

// Database-backed resolvers
const createDatabaseResolvers = (sql: SqlClient.SqlClient) => {
  const userResolver = RequestResolver.makeBatched((requests: ReadonlyArray<GetUser>) =>
    Effect.gen(function* () {
      const userIds = requests.map(r => r.id)
      
      // Single SQL query for batch
      const users = yield* sql`
        SELECT id, name, email, created_at 
        FROM users 
        WHERE id IN (${sql.in(userIds)})
      `.pipe(
        Effect.map(rows => rows.map(row => new User({
          id: row.id,
          name: row.name,
          email: row.email,
          createdAt: row.created_at
        })))
      )
      
      // Complete requests
      yield* Effect.forEach(requests, (request) => {
        const user = users.find(u => u.id === request.id)
        return user
          ? Request.succeed(request, user)
          : Request.fail(request, new UserNotFoundError({ id: request.id }))
      }, { discard: true })
    })
  )
  
  // Complex query with joins
  const orderWithDetailsResolver = RequestResolver.makeBatched((requests: ReadonlyArray<GetOrderDetails>) =>
    Effect.gen(function* () {
      const orderIds = requests.map(r => r.orderId)
      
      const results = yield* sql`
        SELECT 
          o.id as order_id,
          o.total,
          o.status,
          c.id as customer_id,
          c.name as customer_name,
          oi.id as item_id,
          oi.quantity,
          p.name as product_name,
          p.price
        FROM orders o
        JOIN customers c ON o.customer_id = c.id
        JOIN order_items oi ON o.id = oi.order_id
        JOIN products p ON oi.product_id = p.id
        WHERE o.id IN (${sql.in(orderIds)})
        ORDER BY o.id, oi.id
      `
      
      // Group results by order
      const orderMap = new Map<string, OrderDetails>()
      
      for (const row of results) {
        const orderId = row.order_id
        
        if (!orderMap.has(orderId)) {
          orderMap.set(orderId, {
            order: {
              id: orderId,
              total: row.total,
              status: row.status
            },
            customer: {
              id: row.customer_id,
              name: row.customer_name
            },
            items: []
          })
        }
        
        const orderDetails = orderMap.get(orderId)!
        orderDetails.items.push({
          id: row.item_id,
          quantity: row.quantity,
          product: {
            name: row.product_name,
            price: row.price
          }
        })
      }
      
      // Complete requests
      yield* Effect.forEach(requests, (request) => {
        const orderDetails = orderMap.get(request.orderId)
        return orderDetails
          ? Request.succeed(request, orderDetails)
          : Request.fail(request, new OrderNotFoundError({ orderId: request.orderId }))
      }, { discard: true })
    })
  )
  
  return { userResolver, orderWithDetailsResolver }
}
```

### Integration with Caching Systems

```typescript
import { Effect, Request, RequestResolver, Duration } from "effect"

// Redis-backed caching resolver
const withRedisCache = <TRequest extends Request.Request<any, any>>(
  resolver: RequestResolver<TRequest>,
  options: {
    readonly keyPrefix: string
    readonly ttl: Duration.DurationInput
    readonly serialize: (data: Request.Request.Success<TRequest>) => string
    readonly deserialize: (data: string) => Request.Request.Success<TRequest>
  }
) => RequestResolver.makeBatched((requests: ReadonlyArray<TRequest>) =>
  Effect.gen(function* () {
    const redis = yield* RedisClient
    
    // Check cache for all requests
    const cacheKeys = requests.map(r => `${options.keyPrefix}:${JSON.stringify(r)}`)
    const cachedValues = yield* redis.mget(cacheKeys)
    
    // Separate cached and uncached requests
    const uncachedRequests: TRequest[] = []
    const cachedResults: Array<{ request: TRequest; value: Request.Request.Success<TRequest> }> = []
    
    for (let i = 0; i < requests.length; i++) {
      const cachedValue = cachedValues[i]
      if (cachedValue) {
        cachedResults.push({
          request: requests[i],
          value: options.deserialize(cachedValue)
        })
      } else {
        uncachedRequests.push(requests[i])
      }
    }
    
    // Complete cached requests
    yield* Effect.forEach(cachedResults, ({ request, value }) =>
      Request.succeed(request, value)
    , { discard: true })
    
    // Process uncached requests
    if (uncachedRequests.length > 0) {
      yield* resolver.runAll([[...uncachedRequests.map(request => 
        Request.makeEntry({ 
          request, 
          result: Effect.never, // Will be completed by original resolver
          listeners: new Request.Listeners(),
          ownerId: Effect.fiberId(),
          state: { completed: false }
        })
      )]])
      
      // Cache the results (this would need to be implemented with hooks)
      // This is a simplified example - real implementation would need request completion hooks
    }
  })
)

// Multi-level caching
const multiLevelCache = <TRequest extends Request.Request<any, any>>(
  resolver: RequestResolver<TRequest>,
  cacheConfig: {
    readonly l1: { ttl: Duration.DurationInput; maxSize: number }
    readonly l2: { ttl: Duration.DurationInput; keyPrefix: string }
  }
) => Effect.gen(function* () {
  // L1: In-memory cache
  const memoryCache = yield* Cache.make({
    capacity: cacheConfig.l1.maxSize,
    timeToLive: cacheConfig.l1.ttl,
    lookup: (request: TRequest) => Effect.request(request, resolver)
  })
  
  // L2: Redis cache  
  const l2Resolver = withRedisCache(resolver, {
    keyPrefix: cacheConfig.l2.keyPrefix,
    ttl: cacheConfig.l2.ttl,
    serialize: JSON.stringify,
    deserialize: JSON.parse
  })
  
  // Combined resolver
  return RequestResolver.fromEffect((request: TRequest) =>
    Effect.gen(function* () {
      // Try L1 cache first
      const l1Result = yield* Cache.get(memoryCache, request).pipe(
        Effect.option
      )
      
      if (Option.isSome(l1Result)) {
        return l1Result.value
      }
      
      // Fallback to L2 cache + original resolver
      return yield* Effect.request(request, l2Resolver)
    })
  )
})
```

## Testing Request-Based Systems

```typescript
import { Effect, Request, RequestResolver, TestContext } from "effect"

// Test utilities for request-based systems
const createTestResolver = <TRequest extends Request.Request<any, any>>(
  responses: Map<string, Request.Request.Success<TRequest> | Request.Request.Error<TRequest>>
): RequestResolver<TRequest> =>
  RequestResolver.fromEffect((request: TRequest) => {
    const key = JSON.stringify(request)
    const response = responses.get(key)
    
    if (!response) {
      return Effect.fail(new Error(`No test response for request: ${key}`))
    }
    
    if (response instanceof Error) {
      return Effect.fail(response)
    }
    
    return Effect.succeed(response)
  })

// Test suite example
const testUserService = Effect.gen(function* () {
  // Setup test data
  const testUsers = new Map([
    [JSON.stringify(new GetUser({ id: "1" })), new User({ id: "1", name: "John" })],
    [JSON.stringify(new GetUser({ id: "2" })), new User({ id: "2", name: "Jane" })],
    [JSON.stringify(new GetUser({ id: "404" })), new UserNotFoundError({ id: "404" })]
  ])
  
  const testResolver = createTestResolver(testUsers)
  
  // Test successful request
  const user = yield* Effect.request(new GetUser({ id: "1" }), testResolver)
  console.assert(user.name === "John")
  
  // Test error case
  const errorResult = yield* Effect.request(
    new GetUser({ id: "404" }), 
    testResolver
  ).pipe(Effect.either)
  
  console.assert(Either.isLeft(errorResult))
  
  // Test batching behavior
  const [user1, user2] = yield* Effect.all([
    Effect.request(new GetUser({ id: "1" }), testResolver),
    Effect.request(new GetUser({ id: "2" }), testResolver)
  ])
  
  console.assert(user1.name === "John" && user2.name === "Jane")
})

// Property-based testing for request resolvers
const testResolverProperties = (resolver: RequestResolver<GetUser>) =>
  Effect.gen(function* () {
    // Test idempotency
    const request = new GetUser({ id: "test-id" })
    const [result1, result2] = yield* Effect.all([
      Effect.request(request, resolver),
      Effect.request(request, resolver)
    ])
    
    console.assert(Equal.equals(result1, result2), "Resolver should be idempotent")
    
    // Test batch consistency
    const requests = Array.from({ length: 10 }, (_, i) => new GetUser({ id: `user-${i}` }))
    
    const batchResults = yield* Effect.forEach(
      requests,
      (req) => Effect.request(req, resolver),
      { concurrency: "unbounded" }
    )
    
    const individualResults = yield* Effect.forEach(
      requests,
      (req) => Effect.request(req, resolver)
    )
    
    console.assert(
      batchResults.every((result, i) => Equal.equals(result, individualResults[i])),
      "Batch results should match individual results"
    )
  })
```

## Conclusion

Request provides a powerful abstraction for declarative, efficient data fetching with automatic batching, deduplication, and caching. It transforms imperative data fetching code into composable, type-safe operations that optimize themselves.

Key benefits:
- **Automatic Optimization**: Batching, deduplication, and caching happen automatically
- **Type Safety**: Full type inference for request parameters, success, and error types
- **Composability**: Requests compose naturally with Effect's concurrency and error handling
- **Performance**: Eliminates N+1 queries and redundant requests without manual optimization
- **Testability**: Clear separation between request definition and resolution logic

Request is ideal for applications that need to efficiently fetch data from multiple sources while maintaining clean, maintainable code. It's particularly powerful in GraphQL-style resolvers, microservice communication, and any scenario where data fetching patterns can benefit from automatic optimization.