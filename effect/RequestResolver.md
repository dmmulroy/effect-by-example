# RequestResolver: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem RequestResolver Solves

Traditional data fetching approaches lead to several critical performance and maintainability issues:

```typescript
// Traditional approach - N+1 query problem
async function getUsersWithPosts(userIds: Array<string>) {
  const users = []
  
  // Sequential database calls - terrible performance!
  for (const userId of userIds) {
    const user = await db.getUser(userId)          // Query 1, 2, 3...
    const posts = await db.getPostsByUser(userId) // Query N+1, N+2, N+3...
    users.push({ ...user, posts })
  }
  
  return users
}

// What happens with 100 users? 200 database queries!
```

This approach leads to:
- **N+1 Query Problem** - Exponential database calls destroying performance
- **No Deduplication** - Same requests executed multiple times wastefully  
- **Poor Caching** - No intelligent request batching or result caching
- **Error Handling Complexity** - Manual coordination of failures across requests
- **Resource Exhaustion** - Uncontrolled concurrent requests overwhelming systems

### The RequestResolver Solution

RequestResolver transforms your data fetching into an intelligent, optimized system that automatically batches, deduplicates, and caches requests:

```typescript
import { Effect, Request, RequestResolver } from "effect"

// Define your data request
class GetUser extends Request.TaggedClass("GetUser")<
  User,                    // Success type
  DatabaseError,           // Error type  
  { readonly userId: string }
> {}

// Create an intelligent resolver that batches requests
const userResolver = RequestResolver.makeBatched((requests: Array<GetUser>) =>
  Effect.gen(function* () {
    // Single optimized database call for ALL users!
    const userIds = requests.map(req => req.userId)
    const users = yield* Database.getUsers(userIds)
    
    // Fulfill each request with its corresponding user
    for (const request of requests) {
      const user = users.find(u => u.id === request.userId)
      if (user) {
        yield* Request.succeed(request, user)
      } else {
        yield* Request.fail(request, new UserNotFoundError(request.userId))
      }
    }
  })
)

// Usage is simple - Effect handles batching automatically!
const getUser = (userId: string) => 
  Effect.request(new GetUser({ userId }), userResolver)
```

### Key Concepts

**Automatic Batching**: Multiple requests made within the same event loop are automatically grouped into efficient batch operations

**Request Deduplication**: Identical requests are deduplicated - only one actual fetch occurs even with multiple callers

**Intelligent Caching**: Built-in request-level caching prevents redundant operations while respecting cache invalidation

**Error Isolation**: Individual request failures don't crash the entire batch - precise error handling per request

**Resource Management**: Smart concurrency control prevents system overload while maximizing throughput

## Basic Usage Patterns

### Pattern 1: Simple Request Definition

```typescript
import { Effect, Request, RequestResolver } from "effect"

// Step 1: Define your request type
class GetProduct extends Request.TaggedClass("GetProduct")<
  Product,              // What you get back
  ProductNotFoundError, // What can go wrong
  { readonly productId: string }
> {}

// Step 2: Create resolver from a single-request function
const productResolver = RequestResolver.fromEffect((request: GetProduct) =>
  Effect.gen(function* () {
    const db = yield* Database
    const product = yield* db.products.findById(request.productId)
    
    if (!product) {
      return yield* Effect.fail(new ProductNotFoundError(request.productId))
    }
    
    return product
  })
)

// Step 3: Use it like any other Effect
const getProduct = (productId: string) =>
  Effect.request(new GetProduct({ productId }), productResolver)
```

### Pattern 2: Batched Request Processing

```typescript
// Define requests that benefit from batching
class GetUserProfile extends Request.TaggedClass("GetUserProfile")<
  UserProfile,
  DatabaseError,
  { readonly userId: string }
> {}

// Create a batched resolver for optimal database usage
const userProfileResolver = RequestResolver.makeBatched((requests: Array<GetUserProfile>) =>
  Effect.gen(function* () {
    const userIds = requests.map(req => req.userId)
    
    // Single efficient database query
    const profiles = yield* Database.batchGetUserProfiles(userIds)
    
    // Fulfill each request with its result
    for (const request of requests) {
      const profile = profiles.get(request.userId)
      if (profile) {
        yield* Request.succeed(request, profile)
      } else {
        yield* Request.fail(request, new DatabaseError(`Profile not found: ${request.userId}`))
      }
    }
  })
)

// Usage automatically batches multiple concurrent calls
const getUserProfile = (userId: string) =>
  Effect.request(new GetUserProfile({ userId }), userProfileResolver)
```

### Pattern 3: Tagged Request Unions

```typescript
// Multiple related request types
class GetUser extends Request.TaggedClass("GetUser")<User, never, { id: string }> {}
class GetUserPosts extends Request.TaggedClass("GetUserPosts")<Array<Post>, never, { userId: string }> {}
class GetUserFollowers extends Request.TaggedClass("GetUserFollowers")<Array<User>, never, { userId: string }> {}

type UserRequest = GetUser | GetUserPosts | GetUserFollowers

// Handle different request types with a single resolver
const userDataResolver = RequestResolver.fromEffectTagged<UserRequest>()(
  {
    GetUser: (requests: Array<GetUser>) =>
      Effect.gen(function* () {
        const users = yield* Database.getUsers(requests.map(r => r.id))
        return users
      }),
      
    GetUserPosts: (requests: Array<GetUserPosts>) =>
      Effect.gen(function* () {
        const userIds = requests.map(r => r.userId)
        const posts = yield* Database.getPostsByUsers(userIds)
        return requests.map(req => posts.filter(p => p.userId === req.userId))
      }),
      
    GetUserFollowers: (requests: Array<GetUserFollowers>) =>
      Effect.gen(function* () {
        const userIds = requests.map(r => r.userId)
        const followers = yield* Database.getFollowersByUsers(userIds)
        return requests.map(req => followers.filter(f => f.followingId === req.userId))
      })
  }
)
```

## Real-World Examples

### Example 1: E-commerce Product Catalog

Modern e-commerce applications need to efficiently load product data, reviews, and inventory across multiple requests:

```typescript
import { Effect, Request, RequestResolver, Array as Arr } from "effect"

// Domain models
interface Product {
  readonly id: string
  readonly name: string
  readonly price: number
  readonly categoryId: string
}

interface ProductReview {
  readonly id: string
  readonly productId: string
  readonly rating: number
  readonly comment: string
}

interface InventoryLevel {
  readonly productId: string
  readonly quantity: number
  readonly reserved: number
}

// Request definitions
class GetProduct extends Request.TaggedClass("GetProduct")<
  Product,
  ProductNotFoundError,
  { readonly productId: string }
> {}

class GetProductReviews extends Request.TaggedClass("GetProductReviews")<
  Array<ProductReview>,
  DatabaseError,
  { readonly productId: string }
> {}

class GetInventoryLevel extends Request.TaggedClass("GetInventoryLevel")<
  InventoryLevel,
  InventoryError,
  { readonly productId: string }
> {}

// Optimized batch resolvers
const productResolver = RequestResolver.makeBatched((requests: Array<GetProduct>) =>
  Effect.gen(function* () {
    const productIds = requests.map(req => req.productId)
    const products = yield* ProductDatabase.batchGet(productIds)
    
    for (const request of requests) {
      const product = products.find(p => p.id === request.productId)
      if (product) {
        yield* Request.succeed(request, product)
      } else {
        yield* Request.fail(request, new ProductNotFoundError(request.productId))
      }
    }
  }).pipe(
    Effect.withSpan("product.batch_get", {
      attributes: { "batch.size": requests => requests.length }
    })
  )
)

const reviewsResolver = RequestResolver.makeBatched((requests: Array<GetProductReviews>) =>
  Effect.gen(function* () {
    const productIds = requests.map(req => req.productId)
    const allReviews = yield* ReviewDatabase.getByProducts(productIds)
    
    for (const request of requests) {
      const reviews = allReviews.filter(r => r.productId === request.productId)
      yield* Request.succeed(request, reviews)
    }
  }).pipe(
    Effect.withSpan("reviews.batch_get")
  )
)

const inventoryResolver = RequestResolver.makeBatched((requests: Array<GetInventoryLevel>) =>
  Effect.gen(function* () {
    const productIds = requests.map(req => req.productId)
    const inventory = yield* InventoryService.batchCheck(productIds)
    
    for (const request of requests) {
      const level = inventory.get(request.productId)
      if (level) {
        yield* Request.succeed(request, level)
      } else {
        yield* Request.fail(request, new InventoryError(`No inventory data: ${request.productId}`))
      }
    }
  }).pipe(
    Effect.withSpan("inventory.batch_check")
  )
)

// High-level operations that automatically batch
const getProduct = (productId: string) =>
  Effect.request(new GetProduct({ productId }), productResolver)

const getProductReviews = (productId: string) =>
  Effect.request(new GetProductReviews({ productId }), reviewsResolver)

const getInventoryLevel = (productId: string) =>
  Effect.request(new GetInventoryLevel({ productId }), inventoryResolver)

// Compose complex data fetching - automatically optimized!
const getProductDetails = (productId: string) =>
  Effect.gen(function* () {
    // These three requests can run concurrently and will be batched
    const [product, reviews, inventory] = yield* Effect.all([
      getProduct(productId),
      getProductReviews(productId),
      getInventoryLevel(productId)
    ], { concurrency: "unbounded", batching: true })
    
    const averageRating = reviews.length > 0 
      ? reviews.reduce((sum, r) => sum + r.rating, 0) / reviews.length
      : 0
      
    const isInStock = inventory.quantity > inventory.reserved
    
    return {
      product,
      reviews,
      inventory,
      averageRating,
      isInStock
    }
  })

// Usage: Load multiple products efficiently
const getProductCatalog = (productIds: Array<string>) =>
  Effect.all(
    productIds.map(getProductDetails),
    { concurrency: "unbounded", batching: true }
  )
```

### Example 2: Social Media Feed Generation

Social media applications require efficient fetching of user data, posts, and social connections:

```typescript
import { Effect, Request, RequestResolver, Array as Arr } from "effect"

// Domain models
interface User {
  readonly id: string
  readonly username: string
  readonly displayName: string
  readonly avatarUrl: string
}

interface Post {
  readonly id: string
  readonly authorId: string
  readonly content: string
  readonly createdAt: Date
  readonly likes: number
}

interface Following {
  readonly followerId: string
  readonly followingId: string
}

// Request definitions with proper error handling
class GetUser extends Request.TaggedClass("GetUser")<
  User,
  UserNotFoundError,
  { readonly userId: string }
> {}

class GetUserPosts extends Request.TaggedClass("GetUserPosts")<
  Array<Post>,
  DatabaseError,
  { readonly userId: string; readonly limit?: number }
> {}

class GetUserFollowing extends Request.TaggedClass("GetUserFollowing")<
  Array<string>,
  DatabaseError,
  { readonly userId: string }
> {}

class GetPostLikes extends Request.TaggedClass("GetPostLikes")<
  number,
  DatabaseError,
  { readonly postId: string }
> {}

// Intelligent resolvers with monitoring
const userResolver = RequestResolver.makeBatched((requests: Array<GetUser>) =>
  Effect.gen(function* () {
    const userIds = requests.map(req => req.userId)
    
    // Single database query for all users
    const users = yield* UserDatabase.batchGet(userIds)
    
    for (const request of requests) {
      const user = users.find(u => u.id === request.userId)
      if (user) {
        yield* Request.succeed(request, user)
      } else {
        yield* Request.fail(request, new UserNotFoundError(request.userId))
      }
    }
  }).pipe(
    Effect.withSpan("users.batch_get", {
      attributes: { "batch.size": requests => requests.length }
    })
  )
).pipe(
  RequestResolver.identified("UserResolver")
)

const postsResolver = RequestResolver.makeBatched((requests: Array<GetUserPosts>) =>
  Effect.gen(function* () {
    const userIds = requests.map(req => req.userId)
    const maxLimit = Math.max(...requests.map(req => req.limit ?? 10))
    
    // Optimized query for all users' posts
    const allPosts = yield* PostDatabase.getByUsers(userIds, maxLimit)
    
    for (const request of requests) {
      const userPosts = allPosts
        .filter(p => p.authorId === request.userId)
        .slice(0, request.limit ?? 10)
      yield* Request.succeed(request, userPosts)
    }
  }).pipe(
    Effect.withSpan("posts.batch_get")
  )
)

const followingResolver = RequestResolver.makeBatched((requests: Array<GetUserFollowing>) =>
  Effect.gen(function* () {
    const userIds = requests.map(req => req.userId)
    const followingData = yield* FollowDatabase.getFollowingByUsers(userIds)
    
    for (const request of requests) {
      const following = followingData
        .filter(f => f.followerId === request.userId)
        .map(f => f.followingId)
      yield* Request.succeed(request, following)
    }
  })
)

// High-level operations
const getUser = (userId: string) =>
  Effect.request(new GetUser({ userId }), userResolver)

const getUserPosts = (userId: string, limit?: number) =>
  Effect.request(new GetUserPosts({ userId, limit }), postsResolver)

const getUserFollowing = (userId: string) =>
  Effect.request(new GetUserFollowing({ userId }), followingResolver)

// Complex feed generation with automatic optimization
const generateFeed = (userId: string, limit: number = 20) =>
  Effect.gen(function* () {
    // Get user's following list
    const following = yield* getUserFollowing(userId)
    
    if (following.length === 0) {
      return []
    }
    
    // Get posts from all followed users - automatically batched!
    const allPosts = yield* Effect.all(
      following.map(followingId => getUserPosts(followingId, 5)),
      { concurrency: "unbounded", batching: true }
    )
    
    // Flatten and sort by creation date
    const feedPosts = allPosts
      .flat()
      .sort((a, b) => b.createdAt.getTime() - a.createdAt.getTime())
      .slice(0, limit)
    
    // Get author information for each post - automatically batched!
    const authors = yield* Effect.all(
      feedPosts.map(post => getUser(post.authorId)),
      { concurrency: "unbounded", batching: true }
    )
    
    // Combine posts with author data
    return feedPosts.map((post, index) => ({
      post,
      author: authors[index]
    }))
  }).pipe(
    Effect.withSpan("feed.generate", {
      attributes: { "user.id": userId, "feed.limit": limit }
    })
  )
```

### Example 3: API Gateway with Multiple Backend Services

Enterprise applications often need to aggregate data from multiple microservices efficiently:

```typescript
import { Effect, Request, RequestResolver, Duration, Array as Arr } from "effect"

// Service models
interface UserProfile {
  readonly userId: string
  readonly email: string
  readonly preferences: Record<string, unknown>
}

interface OrderHistory {
  readonly userId: string
  readonly orders: Array<Order>
  readonly totalSpent: number
}

interface RecommendationData {
  readonly userId: string
  readonly recommendations: Array<Product>
  readonly personalizedOffers: Array<Offer>
}

// Request definitions for different services
class GetUserProfile extends Request.TaggedClass("GetUserProfile")<
  UserProfile,
  ServiceError,
  { readonly userId: string }
> {}

class GetOrderHistory extends Request.TaggedClass("GetOrderHistory")<
  OrderHistory,
  ServiceError,
  { readonly userId: string; readonly months?: number }
> {}

class GetRecommendations extends Request.TaggedClass("GetRecommendations")<
  RecommendationData,
  ServiceError,
  { readonly userId: string }
> {}

// Service-specific resolvers with timeouts and retries
const profileServiceResolver = RequestResolver.makeBatched((requests: Array<GetUserProfile>) =>
  Effect.gen(function* () {
    const userIds = requests.map(req => req.userId)
    
    // Call user service with batch API
    const profiles = yield* ProfileService.batchGetProfiles(userIds)
    
    for (const request of requests) {
      const profile = profiles.find(p => p.userId === request.userId)
      if (profile) {
        yield* Request.succeed(request, profile)
      } else {
        yield* Request.fail(request, new ServiceError(`Profile not found: ${request.userId}`))
      }
    }
  }).pipe(
    Effect.timeout(Duration.seconds(5)),
    Effect.retry({ times: 2, delay: Duration.millis(100) }),
    Effect.withSpan("profile_service.batch_get")
  )
).pipe(
  RequestResolver.identified("ProfileService")
)

const orderServiceResolver = RequestResolver.makeBatched((requests: Array<GetOrderHistory>) =>
  Effect.gen(function* () {
    // Group requests by months parameter for efficient batching
    const requestsByMonths = Arr.groupBy(requests, req => req.months ?? 12)
    
    for (const [months, groupedRequests] of Object.entries(requestsByMonths)) {
      const userIds = groupedRequests.map(req => req.userId)
      const orderHistories = yield* OrderService.batchGetOrderHistory(userIds, Number(months))
      
      for (const request of groupedRequests) {
        const history = orderHistories.find(h => h.userId === request.userId)
        if (history) {
          yield* Request.succeed(request, history)
        } else {
          yield* Request.fail(request, new ServiceError(`Order history not found: ${request.userId}`))
        }
      }
    }
  }).pipe(
    Effect.timeout(Duration.seconds(10)),
    Effect.retry({ times: 3, delay: Duration.millis(200) }),
    Effect.withSpan("order_service.batch_get")
  )
)

const recommendationServiceResolver = RequestResolver.makeBatched((requests: Array<GetRecommendations>) =>
  Effect.gen(function* () {
    const userIds = requests.map(req => req.userId)
    
    // ML service call for personalized recommendations
    const recommendations = yield* RecommendationService.batchGetRecommendations(userIds)
    
    for (const request of requests) {
      const userRecs = recommendations.find(r => r.userId === request.userId)
      if (userRecs) {
        yield* Request.succeed(request, userRecs)
      } else {
        // Fallback to empty recommendations rather than failing
        yield* Request.succeed(request, {
          userId: request.userId,
          recommendations: [],
          personalizedOffers: []
        })
      }
    }
  }).pipe(
    Effect.timeout(Duration.seconds(3)),
    Effect.retry({ times: 1 }),
    Effect.withSpan("recommendation_service.batch_get")
  )
)

// High-level API functions
const getUserProfile = (userId: string) =>
  Effect.request(new GetUserProfile({ userId }), profileServiceResolver)

const getOrderHistory = (userId: string, months?: number) =>
  Effect.request(new GetOrderHistory({ userId, months }), orderServiceResolver)

const getRecommendations = (userId: string) =>
  Effect.request(new GetRecommendations({ userId }), recommendationServiceResolver)

// Comprehensive user dashboard data aggregation
const getUserDashboard = (userId: string) =>
  Effect.gen(function* () {
    // All three service calls happen concurrently and are automatically batched
    const [profile, orderHistory, recommendations] = yield* Effect.all([
      getUserProfile(userId),
      getOrderHistory(userId, 6), // Last 6 months
      getRecommendations(userId)
    ], { concurrency: "unbounded", batching: true })
    
    // Derive additional insights
    const isVipCustomer = orderHistory.totalSpent > 1000
    const hasActiveOffers = recommendations.personalizedOffers.length > 0
    
    return {
      profile,
      orderHistory,
      recommendations,
      insights: {
        isVipCustomer,
        hasActiveOffers,
        lastOrderDate: orderHistory.orders[0]?.createdAt
      }
    }
  }).pipe(
    Effect.withSpan("user_dashboard.aggregate", {
      attributes: { "user.id": userId }
    })
  )

// Bulk dashboard generation for admin views
const getBulkUserDashboards = (userIds: Array<string>) =>
  Effect.all(
    userIds.map(getUserDashboard),
    { concurrency: "unbounded", batching: true }
  )
```

## Advanced Features Deep Dive

### Feature 1: Request Caching and Deduplication

RequestResolver provides sophisticated caching mechanisms to eliminate redundant requests and optimize performance:

#### Basic Request Caching

```typescript
import { Effect, Request, RequestResolver, Cache } from "effect"

class GetExpensiveComputation extends Request.TaggedClass("GetExpensiveComputation")<
  ComputationResult,
  ComputationError,
  { readonly input: string }
> {}

const expensiveResolver = RequestResolver.fromEffect((request: GetExpensiveComputation) =>
  Effect.gen(function* () {
    // Simulate expensive computation
    yield* Effect.sleep(Duration.seconds(2))
    yield* Effect.log(`Computing for input: ${request.input}`)
    
    const result = yield* performExpensiveComputation(request.input)
    return result
  })
)

const getComputation = (input: string) =>
  Effect.request(
    new GetExpensiveComputation({ input }),
    expensiveResolver
  )

// Enable request caching - identical requests are deduplicated
const cachedProgram = Effect.gen(function* () {
  // These three calls will result in only ONE actual computation
  const [result1, result2, result3] = yield* Effect.all([
    getComputation("same-input"),
    getComputation("same-input"), 
    getComputation("same-input")
  ], { concurrency: "unbounded", batching: true })
  
  yield* Effect.log(`All results equal: ${result1 === result2 && result2 === result3}`)
}).pipe(
  Effect.withRequestCaching(true) // Enable request-level caching
)
```

#### Advanced Cache Configuration

```typescript
// Custom cache with TTL and size limits
const createCachedResolver = <A extends Request.Request<any, any>>(
  baseResolver: RequestResolver.RequestResolver<A>,
  options: {
    readonly timeToLive: Duration.DurationInput
    readonly maxSize: number
  }
) =>
  Effect.gen(function* () {
    const cache = yield* Cache.make({
      capacity: options.maxSize,
      timeToLive: options.timeToLive,
      lookup: (request: A) => Effect.request(request, baseResolver)
    })
    
    return RequestResolver.fromEffect((request: A) =>
      Cache.get(cache, request)
    )
  })

// Usage with custom cache settings
const cachedUserResolver = yield* createCachedResolver(userResolver, {
  timeToLive: Duration.minutes(5),
  maxSize: 1000
})
```

### Feature 2: Error Handling and Retry Strategies

RequestResolver integrates seamlessly with Effect's error handling for robust data fetching:

#### Granular Error Handling

```typescript
import { Effect, Request, RequestResolver, Schedule } from "effect"

// Define specific error types
class NetworkError extends Data.TaggedClass("NetworkError")<{
  readonly cause: string
  readonly retryable: boolean
}> {}

class ValidationError extends Data.TaggedClass("ValidationError")<{
  readonly field: string
  readonly message: string
}> {}

class GetUserWithRetry extends Request.TaggedClass("GetUserWithRetry")<
  User,
  NetworkError | ValidationError,
  { readonly userId: string }
> {}

const resilientUserResolver = RequestResolver.makeBatched((requests: Array<GetUserWithRetry>) =>
  Effect.gen(function* () {
    const userIds = requests.map(req => req.userId)
    
    try {
      const users = yield* UserApi.batchGet(userIds).pipe(
        Effect.retry(
          Schedule.exponential(Duration.millis(100)).pipe(
            Schedule.compose(Schedule.recurs(3))
          )
        ),
        Effect.catchTag("HttpError", (error) => 
          new NetworkError({ 
            cause: error.message, 
            retryable: error.status >= 500 
          })
        )
      )
      
      for (const request of requests) {
        const user = users.find(u => u.id === request.userId)
        if (user) {
          // Validate user data
          const validationResult = yield* validateUser(user)
          if (validationResult._tag === "Right") {
            yield* Request.succeed(request, validationResult.right)
          } else {
            yield* Request.fail(request, new ValidationError({
              field: validationResult.left.field,
              message: validationResult.left.message
            }))
          }
        } else {
          yield* Request.fail(request, new NetworkError({
            cause: `User ${request.userId} not found`,
            retryable: false
          }))
        }
      }
    } catch (error) {
      // Handle batch-level failures
      for (const request of requests) {
        yield* Request.fail(request, new NetworkError({
          cause: String(error),
          retryable: true
        }))
      }
    }
  })
)
```

#### Circuit Breaker Pattern

```typescript
// Implement circuit breaker for external service calls
const createCircuitBreakerResolver = <A extends Request.Request<any, any>>(
  baseResolver: RequestResolver.RequestResolver<A>,
  options: {
    readonly maxFailures: number
    readonly resetTimeout: Duration.DurationInput
  }
) =>
  Effect.gen(function* () {
    const failures = yield* Ref.make(0)
    const lastFailure = yield* Ref.make(Option.none<Date>())
    const isOpen = yield* Ref.make(false)
    
    const checkCircuit = Effect.gen(function* () {
      const open = yield* Ref.get(isOpen)
      if (!open) return true
      
      const lastFail = yield* Ref.get(lastFailure)
      if (Option.isNone(lastFail)) return true
      
      const resetTime = new Date(lastFail.value.getTime() + Duration.toMillis(options.resetTimeout))
      if (new Date() > resetTime) {
        yield* Ref.set(isOpen, false)
        yield* Ref.set(failures, 0)
        return true
      }
      
      return false
    })
    
    return RequestResolver.fromEffect((request: A) =>
      Effect.gen(function* () {
        const canProceed = yield* checkCircuit
        if (!canProceed) {
          return yield* Effect.fail(new ServiceUnavailableError("Circuit breaker is open"))
        }
        
        const result = yield* Effect.request(request, baseResolver)
        
        // Reset failure count on success
        yield* Ref.set(failures, 0)
        return result
      }).pipe(
        Effect.catchAll((error) =>
          Effect.gen(function* () {
            const currentFailures = yield* Ref.updateAndGet(failures, n => n + 1)
            yield* Ref.set(lastFailure, Option.some(new Date()))
            
            if (currentFailures >= options.maxFailures) {
              yield* Ref.set(isOpen, true)
              yield* Effect.log("Circuit breaker opened due to excessive failures")
            }
            
            return yield* Effect.fail(error)
          })
        )
      )
    )
  })
```

### Feature 3: Performance Monitoring and Observability

RequestResolver integrates with Effect's observability features for comprehensive monitoring:

#### Request Metrics and Tracing

```typescript
import { Effect, Metrics, RequestResolver } from "effect"

// Define custom metrics
const requestDuration = Metrics.histogram("request_duration_ms", {
  description: "Duration of request processing in milliseconds",
  boundaries: [1, 5, 10, 25, 50, 100, 250, 500, 1000, 2500, 5000]
})

const requestCount = Metrics.counter("request_total", {
  description: "Total number of requests processed"
})

const batchSize = Metrics.histogram("batch_size", {
  description: "Size of request batches",
  boundaries: [1, 2, 5, 10, 25, 50, 100]
})

// Enhanced resolver with comprehensive monitoring
const monitoredResolver = <A extends Request.Request<any, any>>(
  name: string,
  baseResolver: RequestResolver.RequestResolver<A>
) =>
  RequestResolver.makeBatched((requests: Array<A>) =>
    Effect.gen(function* () {
      const startTime = Date.now()
      
      // Record batch size
      yield* Metrics.increment(batchSize, requests.length)
      yield* Metrics.increment(requestCount, requests.length)
      
      // Execute the actual resolution with tracing
      yield* baseResolver.runAll([[...requests.map(req => ({ request: req } as any))]])
      
      // Record duration
      const duration = Date.now() - startTime
      yield* Metrics.set(requestDuration, duration)
      
      yield* Effect.log(`Batch processed: ${requests.length} requests in ${duration}ms`)
    }).pipe(
      Effect.withSpan(`resolver.${name}`, {
        attributes: {
          "batch.size": requests.length,
          "resolver.name": name
        }
      })
    )
  )

// Usage with monitoring
const monitoredUserResolver = monitoredResolver("user", userResolver)
```

#### Performance Analytics

```typescript
// Advanced performance tracking
const createAnalyticsResolver = <A extends Request.Request<any, any>>(
  name: string,
  baseResolver: RequestResolver.RequestResolver<A>
) =>
  Effect.gen(function* () {
    const stats = yield* Ref.make({
      totalRequests: 0,
      totalBatches: 0,
      averageBatchSize: 0,
      totalDuration: 0,
      averageDuration: 0,
      errorRate: 0,
      errors: 0
    })
    
    const updateStats = (batchSize: number, duration: number, hadError: boolean) =>
      Ref.update(stats, current => {
        const newTotalRequests = current.totalRequests + batchSize
        const newTotalBatches = current.totalBatches + 1
        const newTotalDuration = current.totalDuration + duration
        const newErrors = current.errors + (hadError ? 1 : 0)
        
        return {
          totalRequests: newTotalRequests,
          totalBatches: newTotalBatches,
          averageBatchSize: newTotalRequests / newTotalBatches,
          totalDuration: newTotalDuration,
          averageDuration: newTotalDuration / newTotalBatches,
          errorRate: (newErrors / newTotalBatches) * 100,
          errors: newErrors
        }
      })
    
    const getStats = () => Ref.get(stats)
    
    const resolver = RequestResolver.aroundRequests(
      baseResolver,
      (requests) =>
        Effect.gen(function* () {
          const startTime = Date.now()
          return { startTime, batchSize: requests.length }
        }),
      (requests, { startTime, batchSize }) =>
        Effect.gen(function* () {
          const duration = Date.now() - startTime
          yield* updateStats(batchSize, duration, false)
        }).pipe(
          Effect.catchAll((error) =>
            Effect.gen(function* () {
              const duration = Date.now() - startTime
              yield* updateStats(batchSize, duration, true)
              return yield* Effect.fail(error)
            })
          )
        )
    )
    
    return { resolver, getStats }
  })
```

## Practical Patterns & Best Practices

### Pattern 1: Hierarchical Data Loading

Efficiently load nested data structures while maintaining optimal query patterns:

```typescript
// Helper for loading tree-like data structures
const createHierarchicalLoader = <TId, TNode>(
  getChildren: (parentId: TId) => Effect.Effect<Array<TNode>, DatabaseError>,
  getNodeId: (node: TNode) => TId,
  maxDepth: number = 3
) => {
  class GetNodeChildren extends Request.TaggedClass("GetNodeChildren")<
    Array<TNode>,
    DatabaseError,
    { readonly parentId: TId; readonly depth: number }
  > {}
  
  const childrenResolver = RequestResolver.makeBatched((requests: Array<GetNodeChildren>) =>
    Effect.gen(function* () {
      // Group by depth to optimize database access patterns
      const requestsByDepth = Arr.groupBy(requests, req => req.depth)
      
      for (const [depth, depthRequests] of Object.entries(requestsByDepth)) {
        const parentIds = depthRequests.map(req => req.parentId)
        const allChildren = yield* Effect.all(
          parentIds.map(getChildren),
          { concurrency: "unbounded", batching: true }
        )
        
        for (let i = 0; i < depthRequests.length; i++) {
          yield* Request.succeed(depthRequests[i], allChildren[i])
        }
      }
    })
  )
  
  const loadHierarchy = (parentId: TId, depth: number = 0): Effect.Effect<Array<TNode>, DatabaseError> =>
    Effect.gen(function* () {
      if (depth >= maxDepth) return []
      
      const children = yield* Effect.request(
        new GetNodeChildren({ parentId, depth }),
        childrenResolver
      )
      
      // Recursively load grandchildren
      const grandchildren = yield* Effect.all(
        children.map(child => loadHierarchy(getNodeId(child), depth + 1)),
        { concurrency: "unbounded", batching: true }
      )
      
      return children.map((child, index) => ({
        ...child,
        children: grandchildren[index]
      }))
    })
  
  return { loadHierarchy }
}

// Usage for organization hierarchy
const orgLoader = createHierarchicalLoader(
  (managerId: string) => OrganizationDb.getDirectReports(managerId),
  (employee: Employee) => employee.id,
  5 // Max 5 levels deep
)

const getOrganizationChart = (ceoId: string) =>
  orgLoader.loadHierarchy(ceoId)
```

### Pattern 2: Conditional Request Loading

Load data conditionally based on previous results while maintaining batching efficiency:

```typescript
// Advanced conditional data loading pattern
const createConditionalLoader = <TRequest extends Request.Request<any, any>, TCondition>(
  baseResolver: RequestResolver.RequestResolver<TRequest>,
  shouldLoad: (condition: TCondition) => boolean,
  getCondition: (request: TRequest) => Effect.Effect<TCondition, never>
) =>
  RequestResolver.makeBatched((requests: Array<TRequest>) =>
    Effect.gen(function* () {
      // First, evaluate conditions for all requests
      const conditions = yield* Effect.all(
        requests.map(getCondition),
        { concurrency: "unbounded" }
      )
      
      // Filter requests that should proceed
      const requestsToProcess = requests.filter((_, index) => 
        shouldLoad(conditions[index])
      )
      
      // Process valid requests through base resolver
      if (requestsToProcess.length > 0) {
        yield* baseResolver.runAll([requestsToProcess.map(req => ({ request: req } as any))])
      }
      
      // Handle skipped requests
      for (let i = 0; i < requests.length; i++) {
        if (!shouldLoad(conditions[i])) {
          yield* Request.succeed(requests[i], null) // or appropriate default
        }
      }
    })
  )

// Usage for premium feature access
class GetPremiumData extends Request.TaggedClass("GetPremiumData")<
  PremiumData | null,
  never,
  { readonly userId: string }
> {}

const premiumDataResolver = createConditionalLoader(
  basePremiumResolver,
  (user: User) => user.subscriptionTier === "premium",
  (request: GetPremiumData) => getUser(request.userId)
)
```

### Pattern 3: Request Transformation and Adaptation

Transform requests to work with different backend APIs or data formats:

```typescript
// Request transformation for API versioning
const createVersionedResolver = <TRequest extends Request.Request<any, any>>(
  v1Resolver: RequestResolver.RequestResolver<TRequest>,
  v2Resolver: RequestResolver.RequestResolver<TRequest>,
  useV2: (request: TRequest) => boolean
) =>
  RequestResolver.eitherWith(
    v2Resolver,
    v1Resolver,
    (entry: Request.Entry<TRequest>) =>
      useV2(entry.request) 
        ? Either.left(entry)  // Use v2
        : Either.right(entry) // Use v1
  )

// Multi-backend resolver pattern
const createMultiBackendResolver = <TRequest extends Request.Request<any, any>>(
  backends: Record<string, RequestResolver.RequestResolver<TRequest>>,
  selectBackend: (request: TRequest) => string
) =>
  RequestResolver.makeBatched((requests: Array<TRequest>) =>
    Effect.gen(function* () {
      // Group requests by backend
      const requestsByBackend = Arr.groupBy(requests, selectBackend)
      
      // Process each backend group
      for (const [backendName, backendRequests] of Object.entries(requestsByBackend)) {
        const resolver = backends[backendName]
        if (resolver) {
          yield* resolver.runAll([backendRequests.map(req => ({ request: req } as any))])
        } else {
          // Handle unknown backend
          for (const request of backendRequests) {
            yield* Request.fail(request, new Error(`Unknown backend: ${backendName}`))
          }
        }
      }
    })
  )

// Usage for multi-region data access
const multiRegionResolver = createMultiBackendResolver(
  {
    "us-east": usEastResolver,
    "eu-west": euWestResolver,
    "asia-pacific": asiaPacificResolver
  },
  (request: GetRegionalData) => request.region
)
```

## Integration Examples

### Integration with HTTP Clients

Integrate RequestResolver with HTTP clients for efficient API data fetching:

```typescript
import { Effect, HttpClient, RequestResolver } from "effect"

// HTTP-based request resolver
const createHttpResolver = <TRequest extends Request.Request<any, any>>(
  baseUrl: string,
  endpoints: {
    readonly single: string
    readonly batch: string
  },
  mapRequest: (request: TRequest) => Record<string, unknown>,
  mapResponse: (response: unknown) => Request.Request.Success<TRequest>
) =>
  RequestResolver.makeBatched((requests: Array<TRequest>) =>
    Effect.gen(function* () {
      const client = yield* HttpClient.HttpClient
      
      if (requests.length === 1) {
        // Single request optimization
        const request = requests[0]
        const response = yield* client.get(`${baseUrl}${endpoints.single}`, {
          urlParams: mapRequest(request)
        }).pipe(
          HttpClient.withRetries(3),
          HttpClient.withTimeout(Duration.seconds(10))
        )
        
        const result = mapResponse(response)
        yield* Request.succeed(request, result)
      } else {
        // Batch request
        const batchPayload = requests.map(mapRequest)
        const response = yield* client.post(`${baseUrl}${endpoints.batch}`, {
          body: HttpBody.json({ requests: batchPayload })
        }).pipe(
          HttpClient.withRetries(2),
          HttpClient.withTimeout(Duration.seconds(30))
        )
        
        const results = response.results as Array<any>
        for (let i = 0; i < requests.length; i++) {
          const result = mapResponse(results[i])
          yield* Request.succeed(requests[i], result)
        }
      }
    }).pipe(
      Effect.withSpan("http_resolver.batch", {
        attributes: { 
          "batch.size": requests.length,
          "endpoint.base": baseUrl 
        }
      })
    )
  )

// Usage with REST API
const userHttpResolver = createHttpResolver(
  "https://api.example.com",
  {
    single: "/users/:id",
    batch: "/users/batch"
  },
  (request: GetUser) => ({ id: request.userId }),
  (response: any) => response.user
)
```

### Integration with Database ORMs

Integrate with popular database ORMs for efficient data access:

```typescript
import { Effect, RequestResolver } from "effect"
import { PrismaClient } from "@prisma/client"

// Prisma integration
const createPrismaResolver = <TModel extends keyof PrismaClient>(
  model: TModel,
  options: {
    readonly batchSize?: number
    readonly include?: Record<string, boolean>
  } = {}
) =>
  Effect.gen(function* () {
    const prisma = yield* Effect.service(PrismaClient)
    
    return RequestResolver.makeBatched((requests: Array<GetById>) =>
      Effect.gen(function* () {
        const ids = requests.map(req => req.id)
        
        // Use Prisma's efficient batch operations
        const records = yield* Effect.tryPromise({
          try: () => (prisma[model] as any).findMany({
            where: { id: { in: ids } },
            include: options.include
          }),
          catch: (error) => new DatabaseError(String(error))
        })
        
        // Map results back to requests
        for (const request of requests) {
          const record = records.find((r: any) => r.id === request.id)
          if (record) {
            yield* Request.succeed(request, record)
          } else {
            yield* Request.fail(request, new NotFoundError(`${String(model)} ${request.id} not found`))
          }
        }
      }).pipe(
        Effect.withSpan(`prisma.${String(model)}.batch_get`)
      )
    )
  })

// Usage
const userPrismaResolver = yield* createPrismaResolver("user", {
  include: { profile: true, posts: true }
})
```

### Integration with Caching Systems

Integrate with Redis and other caching systems for optimal performance:

```typescript
import { Effect, RequestResolver, Layer } from "effect"
import Redis from "ioredis"

// Redis-backed resolver with fallback to database
const createCachedResolver = <TRequest extends Request.Request<any, any>>(
  cacheKeyPrefix: string,
  baseResolver: RequestResolver.RequestResolver<TRequest>,
  getKey: (request: TRequest) => string,
  ttl: Duration.DurationInput = Duration.minutes(5)
) =>
  Effect.gen(function* () {
    const redis = yield* Effect.service(Redis)
    
    return RequestResolver.makeBatched((requests: Array<TRequest>) =>
      Effect.gen(function* () {
        // Try to get from cache first
        const cacheKeys = requests.map(req => `${cacheKeyPrefix}:${getKey(req)}`)
        const cachedResults = yield* Effect.tryPromise({
          try: () => redis.mget(...cacheKeys),
          catch: () => null
        })
        
        const missedRequests: Array<TRequest> = []
        const cacheHits: Array<{ request: TRequest; result: any }> = []
        
        // Identify cache hits and misses
        for (let i = 0; i < requests.length; i++) {
          const cached = cachedResults?.[i]
          if (cached) {
            try {
              const parsed = JSON.parse(cached)
              cacheHits.push({ request: requests[i], result: parsed })
            } catch {
              missedRequests.push(requests[i])
            }
          } else {
            missedRequests.push(requests[i])
          }
        }
        
        // Fulfill cache hits immediately
        for (const { request, result } of cacheHits) {
          yield* Request.succeed(request, result)
        }
        
        // Process cache misses through base resolver
        if (missedRequests.length > 0) {
          yield* baseResolver.runAll([missedRequests.map(req => ({ request: req } as any))])
          
          // Cache the results for future use
          const cacheOperations = missedRequests.map(request => {
            const key = `${cacheKeyPrefix}:${getKey(request)}`
            // Note: In real implementation, you'd need to get the result from the request
            return Effect.tryPromise({
              try: () => redis.setex(key, Duration.toSeconds(ttl), JSON.stringify(result)),
              catch: () => null // Ignore cache write failures
            })
          })
          
          yield* Effect.all(cacheOperations, { concurrency: "unbounded" })
        }
      }).pipe(
        Effect.withSpan("cached_resolver.batch", {
          attributes: {
            "cache.prefix": cacheKeyPrefix,
            "cache.hits": cacheHits.length,
            "cache.misses": missedRequests.length
          }
        })
      )
    )
  })

// Service layer for Redis
const RedisLive = Layer.effect(
  Redis,
  Effect.sync(() => new Redis({
    host: "localhost",
    port: 6379,
    retryDelayOnFailover: 100,
    maxRetriesPerRequest: 3
  }))
)

// Usage
const cachedUserResolver = yield* createCachedResolver(
  "users",
  baseUserResolver,
  (request: GetUser) => request.userId,
  Duration.minutes(15)
)
```

### Testing Strategies

Comprehensive testing strategies for RequestResolver-based systems:

```typescript
import { Effect, Layer, RequestResolver, TestContext } from "effect"

// Mock resolver for testing
const createMockResolver = <TRequest extends Request.Request<any, any>>(
  responses: Map<string, Request.Request.Success<TRequest> | Request.Request.Error<TRequest>>
) =>
  RequestResolver.makeBatched((requests: Array<TRequest>) =>
    Effect.gen(function* () {
      yield* Effect.sleep(Duration.millis(1)) // Simulate async behavior
      
      for (const request of requests) {
        const key = JSON.stringify(request)
        const response = responses.get(key)
        
        if (response instanceof Error) {
          yield* Request.fail(request, response)
        } else if (response !== undefined) {
          yield* Request.succeed(request, response)
        } else {
          yield* Request.fail(request, new Error(`No mock response for: ${key}`))
        }
      }
    })
  )

// Test helpers
const createTestSuite = () => {
  const mockUserData = new Map([
    [JSON.stringify(new GetUser({ userId: "1" })), { id: "1", name: "John" }],
    [JSON.stringify(new GetUser({ userId: "2" })), { id: "2", name: "Jane" }],
    [JSON.stringify(new GetUser({ userId: "invalid" })), new Error("User not found")]
  ])
  
  const mockResolver = createMockResolver(mockUserData)
  
  return {
    "should batch multiple requests": Effect.gen(function* () {
      const [user1, user2] = yield* Effect.all([
        Effect.request(new GetUser({ userId: "1" }), mockResolver),
        Effect.request(new GetUser({ userId: "2" }), mockResolver)
      ], { concurrency: "unbounded", batching: true })
      
      expect(user1.name).toBe("John")
      expect(user2.name).toBe("Jane")
    }),
    
    "should deduplicate identical requests": Effect.gen(function* () {
      let requestCount = 0
      const countingResolver = RequestResolver.makeBatched((requests: Array<GetUser>) =>
        Effect.gen(function* () {
          requestCount += requests.length
          yield* mockResolver.runAll([requests.map(req => ({ request: req } as any))])
        })
      )
      
      yield* Effect.all([
        Effect.request(new GetUser({ userId: "1" }), countingResolver),
        Effect.request(new GetUser({ userId: "1" }), countingResolver),
        Effect.request(new GetUser({ userId: "1" }), countingResolver)
      ], { concurrency: "unbounded", batching: true }).pipe(
        Effect.withRequestCaching(true)
      )
      
      expect(requestCount).toBe(1) // Only one actual request despite 3 calls
    }),
    
    "should handle individual request failures": Effect.gen(function* () {
      const result = yield* Effect.all([
        Effect.request(new GetUser({ userId: "1" }), mockResolver),
        Effect.request(new GetUser({ userId: "invalid" }), mockResolver).pipe(
          Effect.either
        )
      ], { concurrency: "unbounded", batching: true })
      
      expect(result[0].name).toBe("John")
      expect(Either.isLeft(result[1])).toBe(true)
    })
  }
}

// Property-based testing with fast-check
import fc from "fast-check"

const testBatchingBehavior = fc.asyncProperty(
  fc.array(fc.string(), { minLength: 1, maxLength: 100 }),
  async (userIds) => {
    const uniqueIds = [...new Set(userIds)]
    let actualRequests = 0
    
    const trackingResolver = RequestResolver.makeBatched((requests: Array<GetUser>) => {
      actualRequests += requests.length
      return Effect.all(
        requests.map(req => Request.succeed(req, { id: req.userId, name: `User ${req.userId}` }))
      )
    })
    
    await Effect.all(
      userIds.map(id => Effect.request(new GetUser({ userId: id }), trackingResolver)),
      { concurrency: "unbounded", batching: true }
    ).pipe(
      Effect.withRequestCaching(true),
      Effect.runPromise
    )
    
    // With deduplication, we should only make requests for unique IDs
    expect(actualRequests).toBe(uniqueIds.length)
  }
)
```

## Conclusion

RequestResolver provides intelligent data fetching optimization, automatic batching, and sophisticated caching for Effect applications.

Key benefits:
- **Performance Optimization**: Eliminates N+1 queries through automatic batching and deduplication
- **Resource Efficiency**: Intelligent request grouping prevents system overload while maximizing throughput
- **Developer Experience**: Simple, composable API that handles complex optimization concerns transparently

RequestResolver is essential for any Effect application that needs to efficiently fetch data from databases, APIs, or external services while maintaining clean, maintainable code architecture.