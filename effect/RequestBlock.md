# RequestBlock: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples) 
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem RequestBlock Solves

When building data-driven applications, developers frequently face the N+1 query problem and inefficient request processing:

```typescript
// Traditional approach - multiple individual requests
async function loadUserProfiles(userIds: string[]) {
  const profiles = []
  // This creates N database calls - very inefficient!
  for (const id of userIds) {
    const profile = await fetchUserProfile(id) // Separate network call each time
    profiles.push(profile)
  }
  return profiles
}

// Another common anti-pattern - sequential processing
async function loadOrderDetails(orderIds: string[]) {
  const orders = []
  for (const id of orderIds) {
    const order = await loadOrder(id)        // Wait for each
    const items = await loadOrderItems(id)   // Then wait for items
    const customer = await loadCustomer(order.customerId) // Then customer
    orders.push({ order, items, customer })
  }
  return orders // All processed sequentially - slow!
}
```

This approach leads to:
- **Performance Issues** - Multiple round-trips to data sources
- **Resource Waste** - Inability to batch similar requests together
- **Poor Scalability** - Sequential processing limits throughput
- **Complex Optimization** - Manual request batching is error-prone

### The RequestBlock Solution

RequestBlock captures a collection of blocked requests as a structured data type, preserving information about which requests can be parallelized and which must be sequential, enabling maximum batching while maintaining ordering guarantees.

```typescript
import { RequestBlock, Request, RequestResolver, Effect, Console } from "effect"

// Effect automatically optimizes request patterns using RequestBlock
const optimizedUserLoad = Effect.gen(function* () {
  // These will be automatically batched together
  const [profile1, profile2, profile3] = yield* Effect.all([
    loadUserProfile("user1"),
    loadUserProfile("user2"), 
    loadUserProfile("user3")
  ])
  
  // This preserves the sequential dependency
  const orders = yield* loadUserOrders(profile1.id)
  const details = yield* loadOrderDetails(orders[0].id)
  
  return { profiles: [profile1, profile2, profile3], orders, details }
})
```

### Key Concepts

**RequestBlock Structure**: A union type representing different request execution patterns:
- `Empty` - No requests to execute
- `Single` - A single request to a data source
- `Par` - Two blocks that can execute in parallel
- `Seq` - Two blocks that must execute sequentially

**Batching Optimization**: RequestBlock automatically identifies requests to the same data source that can be batched together for efficiency.

**Ordering Preservation**: Sequential dependencies are preserved while maximizing parallelism where possible.

## Basic Usage Patterns

### Pattern 1: Creating Single Request Blocks

```typescript
import { RequestBlock, Request, RequestResolver, Effect } from "effect"

// Define a request type
interface GetUser extends Request.Request<User, UserNotFoundError> {
  readonly _tag: "GetUser"
  readonly id: string
}

const GetUser = Request.tagged<GetUser>("GetUser")

// Create a RequestResolver
const UserResolver = RequestResolver.makeBatched(
  (requests: NonEmptyArray<GetUser>) => Effect.gen(function* () {
    const ids = requests.map(req => req.id)
    const users = yield* fetchUsersBatch(ids)
    
    requests.forEach((request, index) => {
      const user = users[index]
      if (user) {
        Request.succeed(request, user)
      } else {
        Request.fail(request, new UserNotFoundError(request.id))
      }
    })
  })
)

// Create a single request block
const singleBlock = RequestBlock.single(UserResolver, GetUser({ id: "123" }))
```

### Pattern 2: Combining Request Blocks in Parallel

```typescript
import { RequestBlock } from "effect"

// Combine requests that can execute in parallel
const parallelBlock = RequestBlock.parallel(
  RequestBlock.single(UserResolver, GetUser({ id: "user1" })),
  RequestBlock.single(UserResolver, GetUser({ id: "user2" }))
)

// Multiple parallel combinations
const multiParallel = RequestBlock.parallel(
  RequestBlock.parallel(
    RequestBlock.single(UserResolver, GetUser({ id: "user1" })),
    RequestBlock.single(UserResolver, GetUser({ id: "user2" }))
  ),
  RequestBlock.parallel(
    RequestBlock.single(UserResolver, GetUser({ id: "user3" })),
    RequestBlock.single(UserResolver, GetUser({ id: "user4" }))
  )
)
```

### Pattern 3: Sequential Request Dependencies

```typescript
import { RequestBlock } from "effect" 

// Create sequential dependencies where later requests depend on earlier ones
const sequentialBlock = RequestBlock.sequential(
  RequestBlock.single(UserResolver, GetUser({ id: "user1" })),
  RequestBlock.single(OrderResolver, GetUserOrders({ userId: "user1" }))
)

// Complex sequential patterns
const complexSequential = RequestBlock.sequential(
  // First: Load user in parallel
  RequestBlock.parallel(
    RequestBlock.single(UserResolver, GetUser({ id: "user1" })),
    RequestBlock.single(UserResolver, GetUser({ id: "user2" }))
  ),
  // Then: Load their orders (depends on users being loaded)
  RequestBlock.parallel(
    RequestBlock.single(OrderResolver, GetUserOrders({ userId: "user1" })),
    RequestBlock.single(OrderResolver, GetUserOrders({ userId: "user2" }))
  )
)
```

## Real-World Examples

### Example 1: E-commerce Dashboard Data Loading

Building an e-commerce dashboard that needs to load user profiles, order history, and product details efficiently.

```typescript
import { Effect, Request, RequestResolver, Data, Array as Arr } from "effect"

// Domain models
interface User {
  readonly id: string
  readonly name: string
  readonly email: string
}

interface Order {
  readonly id: string
  readonly userId: string
  readonly total: number
  readonly items: OrderItem[]
}

interface Product {
  readonly id: string
  readonly name: string
  readonly price: number
}

interface OrderItem {
  readonly productId: string
  readonly quantity: number
}

// Error types
class UserNotFoundError extends Data.TaggedError("UserNotFoundError")<{
  readonly userId: string
}> {}

class OrderNotFoundError extends Data.TaggedError("OrderNotFoundError")<{
  readonly orderId: string
}> {}

// Request definitions
interface GetUser extends Request.Request<User, UserNotFoundError> {
  readonly _tag: "GetUser"
  readonly id: string
}
const GetUser = Request.tagged<GetUser>("GetUser")

interface GetUserOrders extends Request.Request<Order[], never> {
  readonly _tag: "GetUserOrders"
  readonly userId: string
}
const GetUserOrders = Request.tagged<GetUserOrders>("GetUserOrders")

interface GetProduct extends Request.Request<Product, never> {
  readonly _tag: "GetProduct"
  readonly id: string
}
const GetProduct = Request.tagged<GetProduct>("GetProduct")

// RequestResolvers with batching
const UserResolver = RequestResolver.makeBatched(
  (requests: NonEmptyArray<GetUser>) => Effect.gen(function* () {
    yield* Console.log(`Batching ${requests.length} user requests`)
    const ids = requests.map(req => req.id)
    const users = yield* fetchUsersBatch(ids)
    
    requests.forEach(request => {
      const user = users.find(u => u.id === request.id)
      if (user) {
        Request.succeed(request, user)
      } else {
        Request.fail(request, new UserNotFoundError({ userId: request.id }))
      }
    })
  })
)

const OrderResolver = RequestResolver.makeBatched(
  (requests: NonEmptyArray<GetUserOrders>) => Effect.gen(function* () {
    yield* Console.log(`Batching ${requests.length} order requests`)
    const userIds = requests.map(req => req.userId)
    const ordersMap = yield* fetchOrdersByUsersBatch(userIds)
    
    requests.forEach(request => {
      const orders = ordersMap.get(request.userId) ?? []
      Request.succeed(request, orders)
    })
  })
)

const ProductResolver = RequestResolver.makeBatched(
  (requests: NonEmptyArray<GetProduct>) => Effect.gen(function* () {
    yield* Console.log(`Batching ${requests.length} product requests`)
    const ids = requests.map(req => req.id)
    const products = yield* fetchProductsBatch(ids)
    
    requests.forEach(request => {
      const product = products.find(p => p.id === request.id)
      if (product) {
        Request.succeed(request, product)
      }
    })
  })
)

// Dashboard data loading with automatic batching
const loadDashboardData = (userIds: string[]) => Effect.gen(function* () {
  // Step 1: Load all users in parallel (batched automatically)
  const users = yield* Effect.all(
    userIds.map(id => Effect.request(GetUser({ id }), UserResolver))
  )
  
  // Step 2: Load orders for all users (batched automatically)
  const orders = yield* Effect.all(
    users.map(user => Effect.request(GetUserOrders({ userId: user.id }), OrderResolver))
  )
  
  // Step 3: Extract all unique product IDs and load products (batched)
  const allProductIds = orders.flatMap(userOrders => 
    userOrders.flatMap(order => order.items.map(item => item.productId))
  )
  const uniqueProductIds = Array.from(new Set(allProductIds))
  
  const products = yield* Effect.all(
    uniqueProductIds.map(id => Effect.request(GetProduct({ id }), ProductResolver))
  )
  
  return { users, orders, products }
}).pipe(
  Effect.withSpan("dashboard.load"),
  Effect.provide(UserResolver),
  Effect.provide(OrderResolver),
  Effect.provide(ProductResolver)
)

// Mock data sources (simulate API calls)
const fetchUsersBatch = (ids: string[]): Effect.Effect<User[]> => 
  Effect.gen(function* () {
    yield* Effect.sleep("100 millis") // Simulate network delay
    return ids.map(id => ({ id, name: `User ${id}`, email: `user${id}@example.com` }))
  })

const fetchOrdersByUsersBatch = (userIds: string[]): Effect.Effect<Map<string, Order[]>> =>
  Effect.gen(function* () {
    yield* Effect.sleep("150 millis")
    const ordersMap = new Map<string, Order[]>()
    userIds.forEach(userId => {
      ordersMap.set(userId, [
        {
          id: `order-${userId}-1`,
          userId,
          total: 100,
          items: [{ productId: "product1", quantity: 2 }]
        }
      ])
    })
    return ordersMap
  })

const fetchProductsBatch = (ids: string[]): Effect.Effect<Product[]> =>
  Effect.gen(function* () {
    yield* Effect.sleep("80 millis")
    return ids.map(id => ({ id, name: `Product ${id}`, price: 50 }))
  })
```

### Example 2: Social Media Feed Aggregation

Building a social media feed that aggregates posts, user info, and interaction data efficiently.

```typescript
import { Effect, Request, RequestResolver, Data, Array as Arr } from "effect"

// Domain models
interface Post {
  readonly id: string
  readonly authorId: string
  readonly content: string
  readonly timestamp: Date
}

interface UserProfile {
  readonly id: string
  readonly username: string
  readonly avatar: string
}

interface PostInteractions {
  readonly postId: string
  readonly likes: number
  readonly comments: number
  readonly shares: number
}

// Request types
interface GetPost extends Request.Request<Post, never> {
  readonly _tag: "GetPost"
  readonly id: string
}
const GetPost = Request.tagged<GetPost>("GetPost")

interface GetUserProfile extends Request.Request<UserProfile, never> {
  readonly _tag: "GetUserProfile"
  readonly userId: string
}
const GetUserProfile = Request.tagged<GetUserProfile>("GetUserProfile")

interface GetPostInteractions extends Request.Request<PostInteractions, never> {
  readonly _tag: "GetPostInteractions"  
  readonly postId: string
}
const GetPostInteractions = Request.tagged<GetPostInteractions>("GetPostInteractions")

// Efficient resolvers
const PostResolver = RequestResolver.makeBatched(
  (requests: NonEmptyArray<GetPost>) => Effect.gen(function* () {
    const postIds = requests.map(req => req.id)
    const posts = yield* fetchPostsBatch(postIds)
    
    requests.forEach(request => {
      const post = posts.find(p => p.id === request.id)
      if (post) Request.succeed(request, post)
    })
  })
)

const UserProfileResolver = RequestResolver.makeBatched(
  (requests: NonEmptyArray<GetUserProfile>) => Effect.gen(function* () {
    const userIds = requests.map(req => req.userId)
    const profiles = yield* fetchUserProfilesBatch(userIds)
    
    requests.forEach(request => {
      const profile = profiles.find(p => p.id === request.userId)
      if (profile) Request.succeed(request, profile)
    })
  })
)

const InteractionsResolver = RequestResolver.makeBatched(
  (requests: NonEmptyArray<GetPostInteractions>) => Effect.gen(function* () {
    const postIds = requests.map(req => req.postId)
    const interactions = yield* fetchInteractionsBatch(postIds)
    
    requests.forEach(request => {
      const interaction = interactions.find(i => i.postId === request.postId)
      if (interaction) Request.succeed(request, interaction)
    })
  })
)

// Feed loading with intelligent batching
interface FeedItem {
  readonly post: Post
  readonly author: UserProfile
  readonly interactions: PostInteractions
}

const loadFeed = (postIds: string[]): Effect.Effect<FeedItem[]> => Effect.gen(function* () {
  // Load all posts in parallel (batched)
  const posts = yield* Effect.all(
    postIds.map(id => Effect.request(GetPost({ id }), PostResolver))
  )
  
  // Extract unique author IDs and load profiles (batched)
  const authorIds = Array.from(new Set(posts.map(post => post.authorId)))
  const profiles = yield* Effect.all(
    authorIds.map(userId => Effect.request(GetUserProfile({ userId }), UserProfileResolver))
  )
  
  // Load interactions for all posts (batched)
  const interactions = yield* Effect.all(
    posts.map(post => Effect.request(GetPostInteractions({ postId: post.id }), InteractionsResolver))
  )
  
  // Combine data into feed items
  const profileMap = new Map(profiles.map(p => [p.id, p]))
  
  return posts.map((post, index) => ({
    post,
    author: profileMap.get(post.authorId)!,
    interactions: interactions[index]
  }))
}).pipe(
  Effect.provide(PostResolver),
  Effect.provide(UserProfileResolver), 
  Effect.provide(InteractionsResolver)
)

// Mock data sources
const fetchPostsBatch = (ids: string[]): Effect.Effect<Post[]> =>
  Effect.succeed(ids.map(id => ({
    id,
    authorId: `author-${id}`,
    content: `Post content ${id}`,
    timestamp: new Date()
  })))

const fetchUserProfilesBatch = (ids: string[]): Effect.Effect<UserProfile[]> =>
  Effect.succeed(ids.map(id => ({
    id,
    username: `user_${id}`,
    avatar: `https://avatar.com/${id}`
  })))

const fetchInteractionsBatch = (postIds: string[]): Effect.Effect<PostInteractions[]> =>
  Effect.succeed(postIds.map(postId => ({
    postId,
    likes: Math.floor(Math.random() * 100),
    comments: Math.floor(Math.random() * 20),
    shares: Math.floor(Math.random() * 10)
  })))
```

### Example 3: Multi-Tenant Data Processing Pipeline

Building a data processing pipeline that handles requests from multiple tenants with different access patterns and performance requirements.

```typescript
import { Effect, Request, RequestResolver, Data, Context, Layer } from "effect"

// Tenant context
interface TenantId {
  readonly id: string
  readonly tier: "basic" | "premium" | "enterprise"
}

class TenantContext extends Context.Tag("TenantContext")<TenantContext, TenantId>() {}

// Data models
interface TenantData {
  readonly tenantId: string
  readonly records: DataRecord[]
}

interface DataRecord {
  readonly id: string
  readonly tenantId: string
  readonly data: unknown
  readonly processedAt?: Date
}

interface ProcessingResult {
  readonly recordId: string
  readonly status: "success" | "failed"
  readonly result?: unknown
  readonly error?: string
}

// Request types with tenant awareness
interface GetTenantData extends Request.Request<TenantData, never> {
  readonly _tag: "GetTenantData"
  readonly tenantId: string
  readonly limit?: number
}
const GetTenantData = Request.tagged<GetTenantData>("GetTenantData")

interface ProcessRecord extends Request.Request<ProcessingResult, never> {
  readonly _tag: "ProcessRecord"
  readonly record: DataRecord
}
const ProcessRecord = Request.tagged<ProcessRecord>("ProcessRecord")

// Tenant-aware resolvers with different batching strategies
const TenantDataResolver = RequestResolver.makeBatched(
  (requests: NonEmptyArray<GetTenantData>) => Effect.gen(function* () {
    const tenant = yield* TenantContext
    
    // Group requests by tenant for efficient batching
    const tenantGroups = requests.reduce((acc, req) => {
      if (!acc[req.tenantId]) acc[req.tenantId] = []
      acc[req.tenantId].push(req)
      return acc
    }, {} as Record<string, GetTenantData[]>)
    
    // Process each tenant's requests
    for (const [tenantId, tenantRequests] of Object.entries(tenantGroups)) {
      const tenantData = yield* fetchTenantDataBatch(tenantId, tenantRequests.length)
      
      tenantRequests.forEach((request, index) => {
        const data = tenantData[index] || { tenantId, records: [] }
        Request.succeed(request, data)
      })
    }
  })
)

const ProcessingResolver = RequestResolver.makeBatched(
  (requests: NonEmptyArray<ProcessRecord>) => Effect.gen(function* () {
    const tenant = yield* TenantContext
    
    // Adjust batch size based on tenant tier
    const batchSize = tenant.tier === "enterprise" ? 100 : 
                     tenant.tier === "premium" ? 50 : 20
    
    const batches = Arr.chunksOf(requests, batchSize)
    
    yield* Effect.all(
      batches.map(batch => Effect.gen(function* () {
        const results = yield* processRecordsBatch(batch.map(req => req.record))
        
        batch.forEach((request, index) => {
          Request.succeed(request, results[index])
        })
      })),
      { concurrency: tenant.tier === "enterprise" ? 4 : 2 }
    )
  })
)

// Multi-tenant processing pipeline
const processTenantWorkload = (tenantId: string, recordLimit: number = 1000) => 
  Effect.gen(function* () {
    const tenant = yield* TenantContext
    
    // Load tenant data with automatic batching
    const tenantData = yield* Effect.request(
      GetTenantData({ tenantId, limit: recordLimit }),
      TenantDataResolver
    )
    
    // Process records in batches based on tenant tier
    const processingResults = yield* Effect.all(
      tenantData.records.map(record => 
        Effect.request(ProcessRecord({ record }), ProcessingResolver)
      ),
      { 
        concurrency: tenant.tier === "enterprise" ? "unbounded" : 10,
        batching: true 
      }
    )
    
    return {
      tenantId,
      totalRecords: tenantData.records.length,
      successCount: processingResults.filter(r => r.status === "success").length,
      failureCount: processingResults.filter(r => r.status === "failed").length,
      results: processingResults
    }
  }).pipe(
    Effect.withSpan("tenant.process_workload", {
      attributes: { "tenant.id": tenantId, "tenant.tier": tenant.tier }
    })
  )

// Layer setup for different tenant tiers
const makeTenantLayer = (tenantId: string, tier: TenantId["tier"]) =>
  Layer.succeed(TenantContext, { id: tenantId, tier })

// Usage example
const processMultipleTenants = Effect.gen(function* () {
  const tenants = [
    { id: "tenant1", tier: "basic" as const },
    { id: "tenant2", tier: "premium" as const },
    { id: "tenant3", tier: "enterprise" as const }
  ]
  
  const results = yield* Effect.all(
    tenants.map(tenant => 
      processTenantWorkload(tenant.id, 500).pipe(
        Effect.provide(makeTenantLayer(tenant.id, tenant.tier)),
        Effect.provide(TenantDataResolver),
        Effect.provide(ProcessingResolver)
      )
    ),
    { concurrency: 3 }
  )
  
  return results
})

// Mock implementations
const fetchTenantDataBatch = (tenantId: string, count: number): Effect.Effect<TenantData[]> =>
  Effect.succeed(
    Array.from({ length: count }, (_, i) => ({
      tenantId,
      records: Array.from({ length: 10 }, (_, j) => ({
        id: `record-${tenantId}-${i}-${j}`,
        tenantId,
        data: { value: Math.random() }
      }))
    }))
  )

const processRecordsBatch = (records: DataRecord[]): Effect.Effect<ProcessingResult[]> =>
  Effect.gen(function* () {
    yield* Effect.sleep("50 millis") // Simulate processing time
    return records.map(record => ({
      recordId: record.id,
      status: Math.random() > 0.1 ? "success" : "failed" as const,
      result: { processed: true, value: Math.random() }
    }))
  })
```

## Advanced Features Deep Dive

### Feature 1: Request Block Transformation

RequestBlock provides powerful transformation capabilities through the `reduce` function and custom reducers.

#### Basic Reduction Usage

```typescript
import { RequestBlock } from "effect"

// Define a custom reducer to analyze request patterns
const AnalysisReducer: RequestBlock.RequestBlock.Reducer<{
  totalRequests: number
  dataSources: Set<string>
  maxDepth: number
}> = {
  emptyCase: () => ({
    totalRequests: 0,
    dataSources: new Set(),
    maxDepth: 0
  }),
  
  singleCase: (dataSource, request) => ({
    totalRequests: 1,
    dataSources: new Set([dataSource.toString()]),
    maxDepth: 1
  }),
  
  parCase: (left, right) => ({
    totalRequests: left.totalRequests + right.totalRequests,
    dataSources: new Set([...left.dataSources, ...right.dataSources]),
    maxDepth: Math.max(left.maxDepth, right.maxDepth)
  }),
  
  seqCase: (left, right) => ({
    totalRequests: left.totalRequests + right.totalRequests,
    dataSources: new Set([...left.dataSources, ...right.dataSources]),
    maxDepth: left.maxDepth + right.maxDepth
  })
}

// Analyze request block complexity
const analyzeRequestBlock = (block: RequestBlock.RequestBlock) => {
  return RequestBlock.reduce(block, AnalysisReducer)
}
```

#### Real-World Transformation Example

```typescript
import { RequestBlock, RequestResolver, Effect } from "effect"

// Transform request resolvers to add caching
const CachingTransformer = <A>(
  cache: Map<string, A>
) => (
  resolver: RequestResolver.RequestResolver<A>
): RequestResolver.RequestResolver<A> => {
  return RequestResolver.make((request) => Effect.gen(function* () {
    const cacheKey = JSON.stringify(request)
    const cached = cache.get(cacheKey)
    
    if (cached) {
      yield* Console.log(`Cache hit for ${cacheKey}`)
      return cached
    }
    
    const result = yield* resolver(request)
    cache.set(cacheKey, result)
    yield* Console.log(`Cached result for ${cacheKey}`)
    return result
  }))
}

// Apply caching to all resolvers in a request block
const addCachingToBlock = (
  block: RequestBlock.RequestBlock,
  cache: Map<string, any>
): RequestBlock.RequestBlock => {
  return RequestBlock.mapRequestResolvers(
    block,
    CachingTransformer(cache)
  )
}

// Usage example
const optimizeWithCaching = Effect.gen(function* () {
  const cache = new Map()
  
  const originalBlock = RequestBlock.parallel(
    RequestBlock.single(UserResolver, GetUser({ id: "123" })),
    RequestBlock.single(UserResolver, GetUser({ id: "456" }))
  )
  
  const cachedBlock = addCachingToBlock(originalBlock, cache)
  
  // First execution populates cache
  yield* Effect.runRequestBlock(cachedBlock)
  
  // Second execution uses cache
  yield* Effect.runRequestBlock(cachedBlock)
})
```

### Feature 2: Advanced Batching Strategies

Understanding how RequestBlock optimizes request execution through intelligent batching.

#### Batching Analysis

```typescript
import { RequestBlock, Effect, Console } from "effect" 

// Create a reducer that tracks batching opportunities
const BatchingAnalyzer: RequestBlock.RequestBlock.Reducer<{
  batchGroups: Map<string, number>
  parallelGroups: number
  sequentialChains: number
}> = {
  emptyCase: () => ({
    batchGroups: new Map(),
    parallelGroups: 0,
    sequentialChains: 0
  }),
  
  singleCase: (dataSource, request) => {
    const sourceName = dataSource.toString()
    const batchGroups = new Map([[sourceName, 1]])
    return {
      batchGroups,
      parallelGroups: 0,
      sequentialChains: 0
    }
  },
  
  parCase: (left, right) => {
    const batchGroups = new Map(left.batchGroups)
    for (const [source, count] of right.batchGroups) {
      batchGroups.set(source, (batchGroups.get(source) || 0) + count)
    }
    
    return {
      batchGroups,
      parallelGroups: left.parallelGroups + right.parallelGroups + 1,
      sequentialChains: left.sequentialChains + right.sequentialChains
    }
  },
  
  seqCase: (left, right) => {
    const batchGroups = new Map(left.batchGroups)
    for (const [source, count] of right.batchGroups) {
      batchGroups.set(source, (batchGroups.get(source) || 0) + count)
    }
    
    return {
      batchGroups,
      parallelGroups: left.parallelGroups + right.parallelGroups,
      sequentialChains: left.sequentialChains + right.sequentialChains + 1
    }
  }
}

// Analyze batching efficiency
const analyzeBatching = (block: RequestBlock.RequestBlock) => {
  const analysis = RequestBlock.reduce(block, BatchingAnalyzer)
  
  return {
    ...analysis,
    totalBatchableRequests: Array.from(analysis.batchGroups.values()).reduce((a, b) => a + b, 0),
    averageBatchSize: analysis.batchGroups.size > 0 
      ? Array.from(analysis.batchGroups.values()).reduce((a, b) => a + b, 0) / analysis.batchGroups.size 
      : 0,
    batchingEfficiency: analysis.batchGroups.size > 0
      ? Array.from(analysis.batchGroups.values()).filter(count => count > 1).length / analysis.batchGroups.size
      : 0
  }
}
```

#### Dynamic Batch Size Optimization

```typescript
import { RequestResolver, Effect, Ref } from "effect"

// Adaptive batch resolver that adjusts batch size based on performance
const makeAdaptiveBatchResolver = <A extends Request.Request<any, any>>(
  processBatch: (requests: A[]) => Effect.Effect<void>,
  initialBatchSize: number = 10
) => {
  return Effect.gen(function* () {
    const batchSizeRef = yield* Ref.make(initialBatchSize)
    const performanceMetrics = yield* Ref.make({
      avgLatency: 0,
      successRate: 1,
      sampleCount: 0
    })
    
    return RequestResolver.makeBatched(
      (requests: NonEmptyArray<A>) => Effect.gen(function* () {
        const startTime = Date.now()
        const batchSize = yield* Ref.get(batchSizeRef)
        
        // Process in adaptive batch sizes
        const batches = Arr.chunksOf(requests, batchSize)
        let successCount = 0
        
        yield* Effect.all(
          batches.map(batch => Effect.gen(function* () {
            try {
              yield* processBatch(batch)
              successCount += batch.length
            } catch (error) {
              yield* Console.log(`Batch processing failed: ${error}`)
            }
          })),
          { concurrency: 2 }
        )
        
        // Update performance metrics
        const endTime = Date.now()
        const latency = endTime - startTime
        const successRate = successCount / requests.length
        
        yield* Ref.update(performanceMetrics, metrics => {
          const newSampleCount = metrics.sampleCount + 1
          const newAvgLatency = (metrics.avgLatency * metrics.sampleCount + latency) / newSampleCount
          const newSuccessRate = (metrics.successRate * metrics.sampleCount + successRate) / newSampleCount
          
          return {
            avgLatency: newAvgLatency,
            successRate: newSuccessRate,
            sampleCount: newSampleCount
          }
        })
        
        // Adjust batch size based on performance
        const metrics = yield* Ref.get(performanceMetrics)
        if (metrics.sampleCount > 5) {
          if (metrics.successRate > 0.95 && metrics.avgLatency < 100) {
            // Increase batch size for better throughput
            yield* Ref.update(batchSizeRef, size => Math.min(size * 1.2, 100))
          } else if (metrics.successRate < 0.8 || metrics.avgLatency > 500) {
            // Decrease batch size for better reliability
            yield* Ref.update(batchSizeRef, size => Math.max(size * 0.8, 5))
          }
        }
      })
    )
  })
}
```

### Feature 3: Request Block Composition Patterns

Advanced patterns for composing complex request blocks for sophisticated data loading scenarios.

#### Conditional Request Composition

```typescript
import { RequestBlock, Effect, Option } from "effect"

// Compose request blocks based on runtime conditions
const conditionalRequestComposition = <A, B>(
  condition: Effect.Effect<boolean>,
  primaryBlock: RequestBlock.RequestBlock,
  fallbackBlock: RequestBlock.RequestBlock
): Effect.Effect<RequestBlock.RequestBlock> => Effect.gen(function* () {
  const shouldUsePrimary = yield* condition
  
  if (shouldUsePrimary) {
    return primaryBlock
  } else {
    return fallbackBlock
  }
})

// Dynamic block building based on user permissions
const buildUserDataBlock = (userId: string, permissions: string[]) => Effect.gen(function* () {
  let block = RequestBlock.single(UserResolver, GetUser({ id: userId }))
  
  if (permissions.includes("read:orders")) {
    block = RequestBlock.parallel(
      block,
      RequestBlock.single(OrderResolver, GetUserOrders({ userId }))
    )
  }
  
  if (permissions.includes("read:analytics")) {
    block = RequestBlock.sequential(
      block,
      RequestBlock.single(AnalyticsResolver, GetUserAnalytics({ userId }))
    )
  }
  
  if (permissions.includes("read:admin")) {
    block = RequestBlock.parallel(
      block,
      RequestBlock.single(AdminResolver, GetUserAdminData({ userId }))
    )
  }
  
  return block
})
```

#### Request Block Pagination

```typescript
import { RequestBlock, Effect, Array as Arr } from "effect"

// Handle paginated data loading with request blocks
interface PaginatedRequest<T> {
  readonly page: number
  readonly pageSize: number
  readonly transform: (items: T[]) => RequestBlock.RequestBlock
}

const paginatedRequestBlock = <T>(
  requests: PaginatedRequest<T>[],
  loadPage: (page: number, pageSize: number) => Effect.Effect<T[]>
) => Effect.gen(function* () {
  // Load all pages in parallel
  const pages = yield* Effect.all(
    requests.map(req => loadPage(req.page, req.pageSize)),
    { concurrency: 5 }
  )
  
  // Transform each page into request blocks and compose
  const blocks = requests.map((req, index) => req.transform(pages[index]))
  
  // Combine all blocks in parallel
  return blocks.reduce((acc, block) => RequestBlock.parallel(acc, block), RequestBlock.empty)
})

// Usage example for loading user data across multiple pages
const loadUserDataAcrossPages = (userIds: string[]) => Effect.gen(function* () {
  const pageSize = 10
  const pages = Arr.chunksOf(userIds, pageSize)
  
  const paginatedRequests: PaginatedRequest<string>[] = pages.map((pageIds, pageIndex) => ({
    page: pageIndex,
    pageSize,
    transform: (ids: string[]) => {
      const userBlocks = ids.map(id => RequestBlock.single(UserResolver, GetUser({ id })))
      return userBlocks.reduce(
        (acc, block) => RequestBlock.parallel(acc, block), 
        RequestBlock.empty
      )
    }
  }))
  
  const composedBlock = yield* paginatedRequestBlock(
    paginatedRequests,
    (page, size) => Effect.succeed(pages[page] || [])
  )
  
  return composedBlock
})
```

## Practical Patterns & Best Practices

### Pattern 1: Request Deduplication

Eliminate duplicate requests automatically while preserving request semantics.

```typescript
import { RequestBlock, Effect, HashMap } from "effect"

// Helper to deduplicate requests in a block
const DeduplicationReducer: RequestBlock.RequestBlock.Reducer<{
  deduped: RequestBlock.RequestBlock
  seen: Set<string>
}> = {
  emptyCase: () => ({
    deduped: RequestBlock.empty,
    seen: new Set()
  }),
  
  singleCase: (dataSource, request) => {
    const key = JSON.stringify({ dataSource: dataSource.toString(), request })
    return {
      deduped: RequestBlock.single(dataSource, request),
      seen: new Set([key])
    }
  },
  
  parCase: (left, right) => {
    const combinedSeen = new Set([...left.seen, ...right.seen])
    return {
      deduped: RequestBlock.parallel(left.deduped, right.deduped),
      seen: combinedSeen
    }
  },
  
  seqCase: (left, right) => {
    const combinedSeen = new Set([...left.seen, ...right.seen])
    return {
      deduped: RequestBlock.sequential(left.deduped, right.deduped),
      seen: combinedSeen
    }
  }
}

// Smart request builder that prevents duplicates
class SmartRequestBuilder {
  private requests = new Map<string, RequestBlock.RequestBlock>()
  
  addRequest<A>(
    key: string,
    dataSource: RequestResolver.RequestResolver<A>,
    request: Request.Entry<A>
  ): this {
    if (!this.requests.has(key)) {
      this.requests.set(key, RequestBlock.single(dataSource, request))
    }
    return this
  }
  
  parallel(key1: string, key2: string): this {
    const block1 = this.requests.get(key1)
    const block2 = this.requests.get(key2)
    
    if (block1 && block2) {
      const combinedKey = `parallel:${key1}:${key2}`
      this.requests.set(combinedKey, RequestBlock.parallel(block1, block2))
    }
    
    return this
  }
  
  sequential(key1: string, key2: string): this {
    const block1 = this.requests.get(key1)
    const block2 = this.requests.get(key2)
    
    if (block1 && block2) {
      const combinedKey = `sequential:${key1}:${key2}`
      this.requests.set(combinedKey, RequestBlock.sequential(block1, block2))
    }
    
    return this
  }
  
  build(key: string): Option.Option<RequestBlock.RequestBlock> {
    return Option.fromNullable(this.requests.get(key))
  }
}

// Usage example
const buildOptimizedRequests = Effect.gen(function* () {
  const builder = new SmartRequestBuilder()
  
  builder
    .addRequest("user1", UserResolver, GetUser({ id: "user1" }))
    .addRequest("user2", UserResolver, GetUser({ id: "user2" }))
    .addRequest("user1", UserResolver, GetUser({ id: "user1" })) // Duplicate - ignored
    .parallel("user1", "user2")
  
  const block = builder.build("parallel:user1:user2")
  
  return Option.match(block, {
    onNone: () => Effect.succeed("No requests to execute"),
    onSome: (requestBlock) => Effect.runRequestBlock(requestBlock)
  })
})
```

### Pattern 2: Request Priority and Scheduling

Implement request prioritization and intelligent scheduling within request blocks.

```typescript
import { RequestBlock, Effect, Schedule, Queue } from "effect"

// Priority-aware request resolver
interface PriorityRequest<A extends Request.Request<any, any>> {
  readonly request: A
  readonly priority: "high" | "medium" | "low"
  readonly deadline?: Date
}

const makePriorityResolver = <A extends Request.Request<any, any>>(
  processBatch: (requests: A[]) => Effect.Effect<void>
) => {
  return RequestResolver.make(
    (request: PriorityRequest<A>) => Effect.gen(function* () {
      const priorityQueue = yield* Queue.bounded<PriorityRequest<A>>(1000)
      
      // Add request to priority queue
      yield* Queue.offer(priorityQueue, request)
      
      // Process based on priority and deadline
      const processQueue = Effect.gen(function* () {
        const items = yield* Queue.takeAll(priorityQueue)
        
        // Sort by priority and deadline
        const sorted = items.sort((a, b) => {
          // High priority first
          if (a.priority !== b.priority) {
            const priorityOrder = { high: 3, medium: 2, low: 1 }
            return priorityOrder[b.priority] - priorityOrder[a.priority]
          }
          
          // Then by deadline
          if (a.deadline && b.deadline) {
            return a.deadline.getTime() - b.deadline.getTime()
          }
          
          return 0
        })
        
        // Process in priority order
        const requests = sorted.map(item => item.request)
        yield* processBatch(requests)
      })
      
      yield* processQueue
    })
  )
}

// Request block with priority scheduling
const priorityScheduledBlock = <A extends Request.Request<any, any>>(
  requests: PriorityRequest<A>[],
  resolver: RequestResolver.RequestResolver<A>
) => Effect.gen(function* () {
  // Group by priority
  const highPriority = requests.filter(r => r.priority === "high")
  const mediumPriority = requests.filter(r => r.priority === "medium")  
  const lowPriority = requests.filter(r => r.priority === "low")
  
  // Build hierarchical execution plan
  const highBlock = highPriority.length > 0 
    ? highPriority.map(pr => RequestBlock.single(resolver, pr.request))
        .reduce((acc, block) => RequestBlock.parallel(acc, block), RequestBlock.empty)
    : RequestBlock.empty
    
  const mediumBlock = mediumPriority.length > 0
    ? mediumPriority.map(pr => RequestBlock.single(resolver, pr.request))
        .reduce((acc, block) => RequestBlock.parallel(acc, block), RequestBlock.empty)
    : RequestBlock.empty
    
  const lowBlock = lowPriority.length > 0
    ? lowPriority.map(pr => RequestBlock.single(resolver, pr.request))
        .reduce((acc, block) => RequestBlock.parallel(acc, block), RequestBlock.empty)
    : RequestBlock.empty
  
  // Execute high priority first, then medium, then low
  return RequestBlock.sequential(
    RequestBlock.sequential(highBlock, mediumBlock),
    lowBlock
  )
})
```

### Pattern 3: Request Block Monitoring and Observability

Add comprehensive monitoring and observability to request block execution.

```typescript
import { RequestBlock, Effect, Metric, Console } from "effect"

// Metrics for request block monitoring
const RequestMetrics = {
  requestsTotal: Metric.counter("requests_total", {
    description: "Total number of requests processed"
  }),
  requestsDuration: Metric.histogram("requests_duration", {
    description: "Request processing duration in milliseconds"
  }),
  batchSize: Metric.histogram("batch_size", {
    description: "Number of requests in each batch"
  }),
  errors: Metric.counter("requests_errors", {
    description: "Number of request errors"
  })
}

// Observable request resolver wrapper
const makeObservableResolver = <A extends Request.Request<any, any>>(
  name: string,
  resolver: RequestResolver.RequestResolver<A>
): RequestResolver.RequestResolver<A> => {
  return RequestResolver.makeBatched(
    (requests: NonEmptyArray<A>) => Effect.gen(function* () {
      const startTime = Date.now()
      
      // Record batch size
      yield* Metric.increment(RequestMetrics.batchSize, requests.length)
      yield* Metric.increment(RequestMetrics.requestsTotal, requests.length)
      
      try {
        // Execute original resolver
        yield* resolver.runAll(requests)
        
        // Record success metrics
        const duration = Date.now() - startTime
        yield* Metric.set(RequestMetrics.requestsDuration, duration)
        
        yield* Console.log(
          `[${name}] Processed batch of ${requests.length} requests in ${duration}ms`
        )
      } catch (error) {
        // Record error metrics
        yield* Metric.increment(RequestMetrics.errors, requests.length)
        yield* Console.log(`[${name}] Batch processing failed: ${error}`)
        throw error
      }
    }).pipe(
      Effect.withSpan(`resolver.${name}`, {
        attributes: {
          "resolver.name": name,
          "batch.size": requests.length
        }
      })
    )
  )
}

// Request block execution with full observability
const executeWithObservability = (
  name: string,
  block: RequestBlock.RequestBlock
) => Effect.gen(function* () {
  const analysis = analyzeBatching(block)
  
  yield* Console.log(`[${name}] Executing request block:`, {
    totalRequests: analysis.totalBatchableRequests,
    dataSources: analysis.batchGroups.size,
    parallelGroups: analysis.parallelGroups,
    batchingEfficiency: `${(analysis.batchingEfficiency * 100).toFixed(1)}%`
  })
  
  const startTime = Date.now()
  
  yield* Effect.runRequestBlock(block).pipe(
    Effect.withSpan(`request_block.${name}`, {
      attributes: {
        "block.name": name,
        "block.total_requests": analysis.totalBatchableRequests,
        "block.data_sources": analysis.batchGroups.size,
        "block.parallel_groups": analysis.parallelGroups
      }
    }),
    Effect.tap(() => Effect.gen(function* () {
      const duration = Date.now() - startTime
      yield* Console.log(`[${name}] Completed in ${duration}ms`)
    }))
  )
})

// Usage with full monitoring
const monitoredExample = Effect.gen(function* () {
  // Create observable resolvers
  const observableUserResolver = makeObservableResolver("user", UserResolver)
  const observableOrderResolver = makeObservableResolver("order", OrderResolver)
  
  // Build request block
  const block = RequestBlock.parallel(
    RequestBlock.single(observableUserResolver, GetUser({ id: "user1" })),
    RequestBlock.parallel(
      RequestBlock.single(observableUserResolver, GetUser({ id: "user2" })),
      RequestBlock.single(observableOrderResolver, GetUserOrders({ userId: "user1" }))
    )
  )
  
  // Execute with full observability
  yield* executeWithObservability("dashboard_load", block)
}).pipe(
  Effect.provide(observableUserResolver),
  Effect.provide(observableOrderResolver)
)
```

## Integration Examples

### Integration with Popular Data Loading Libraries

Show how RequestBlock integrates with common data loading patterns and libraries.

```typescript
import { RequestBlock, Effect, Context, Layer } from "effect"

// Integration with GraphQL-style data loading
interface GraphQLContext {
  readonly loaders: {
    readonly user: (id: string) => Effect.Effect<User>
    readonly post: (id: string) => Effect.Effect<Post>
    readonly comment: (postId: string) => Effect.Effect<Comment[]>
  }
}

class GraphQLService extends Context.Tag("GraphQLService")<GraphQLService, GraphQLContext>() {}

// Convert GraphQL loaders to RequestResolvers  
const makeGraphQLRequestResolver = <K, V>(
  loader: (key: K) => Effect.Effect<V>
) => RequestResolver.makeBatched(
  (requests: NonEmptyArray<{ key: K }>) => Effect.gen(function* () {
    const results = yield* Effect.all(
      requests.map(req => loader(req.key)),
      { concurrency: 10 }
    )
    
    requests.forEach((request, index) => {
      Request.succeed(request, results[index])
    })
  })
)

// GraphQL query execution using RequestBlock
const executeGraphQLQuery = (query: {
  users: string[]
  posts: string[]
  commentsForPosts: string[]
}) => Effect.gen(function* () {
  const graphql = yield* GraphQLService
  
  // Create resolvers from GraphQL loaders
  const userResolver = makeGraphQLRequestResolver(graphql.loaders.user)
  const postResolver = makeGraphQLRequestResolver(graphql.loaders.post)
  const commentResolver = makeGraphQLRequestResolver(graphql.loaders.comment)
  
  // Build request block from GraphQL query
  const userBlocks = query.users.map(id => 
    RequestBlock.single(userResolver, { key: id })
  )
  
  const postBlocks = query.posts.map(id =>
    RequestBlock.single(postResolver, { key: id })
  )
  
  const commentBlocks = query.commentsForPosts.map(postId =>
    RequestBlock.single(commentResolver, { key: postId })
  )
  
  // Combine all requests optimally
  const allUserRequests = userBlocks.reduce(
    (acc, block) => RequestBlock.parallel(acc, block),
    RequestBlock.empty
  )
  
  const allPostRequests = postBlocks.reduce(
    (acc, block) => RequestBlock.parallel(acc, block),
    RequestBlock.empty
  )
  
  const allCommentRequests = commentBlocks.reduce(
    (acc, block) => RequestBlock.parallel(acc, block),
    RequestBlock.empty
  )
  
  // Execute users and posts in parallel, then comments (which may depend on posts)
  const queryBlock = RequestBlock.sequential(
    RequestBlock.parallel(allUserRequests, allPostRequests),
    allCommentRequests
  )
  
  yield* Effect.runRequestBlock(queryBlock)
})
```

### Integration with Caching Systems

Demonstrate how RequestBlock works with various caching strategies.

```typescript
import { RequestBlock, Effect, Duration, Schedule } from "effect"

// Multi-level caching integration
interface CacheConfig {
  readonly l1: {
    readonly maxSize: number
    readonly ttl: Duration.Duration
  }
  readonly l2: {
    readonly ttl: Duration.Duration
    readonly keyPrefix: string
  }
}

class CacheService extends Context.Tag("CacheService")<CacheService, {
  readonly l1Cache: Map<string, { value: any; expiry: number }>
  readonly l2Cache: {
    get: (key: string) => Effect.Effect<Option.Option<any>>
    set: (key: string, value: any, ttl: Duration.Duration) => Effect.Effect<void>
  }
  readonly config: CacheConfig
}>() {}

// Cache-aware request resolver
const makeCachedResolver = <A extends Request.Request<any, any>>(
  name: string,
  resolver: RequestResolver.RequestResolver<A>
) => Effect.gen(function* () {
  const cache = yield* CacheService
  
  return RequestResolver.makeBatched(
    (requests: NonEmptyArray<A>) => Effect.gen(function* () {
      const now = Date.now()
      const uncachedRequests: A[] = []
      const cachedResults = new Map<A, any>()
      
      // Check L1 cache
      for (const request of requests) {
        const cacheKey = `${name}:${JSON.stringify(request)}`
        const cached = cache.l1Cache.get(cacheKey)
        
        if (cached && cached.expiry > now) {
          cachedResults.set(request, cached.value)
        } else {
          // Check L2 cache
          const l2Result = yield* cache.l2Cache.get(
            `${cache.config.l2.keyPrefix}:${cacheKey}`
          )
          
          if (Option.isSome(l2Result)) {
            cachedResults.set(request, l2Result.value)
            // Warm L1 cache
            cache.l1Cache.set(cacheKey, {
              value: l2Result.value,
              expiry: now + Duration.toMillis(cache.config.l1.ttl)
            })
          } else {
            uncachedRequests.push(request)
          }
        }
      }
      
      // Process uncached requests
      if (uncachedRequests.length > 0) {
        yield* resolver.runAll(uncachedRequests)
        
        // Cache the results
        for (const request of uncachedRequests) {
          const cacheKey = `${name}:${JSON.stringify(request)}`
          // Note: In real implementation, you'd extract the result from the request
          const result = "mock_result" // This would be the actual result
          
          // Store in both cache levels
          cache.l1Cache.set(cacheKey, {
            value: result,
            expiry: now + Duration.toMillis(cache.config.l1.ttl)
          })
          
          yield* cache.l2Cache.set(
            `${cache.config.l2.keyPrefix}:${cacheKey}`,
            result,
            cache.config.l2.ttl
          )
        }
      }
      
      // Apply cached results
      for (const [request, result] of cachedResults) {
        Request.succeed(request, result)
      }
    })
  )
})

// Cache invalidation patterns
const invalidateCachePattern = (pattern: string) => Effect.gen(function* () {
  const cache = yield* CacheService
  
  // Invalidate L1 cache
  for (const key of cache.l1Cache.keys()) {
    if (key.includes(pattern)) {
      cache.l1Cache.delete(key)
    }
  }
  
  // Invalidate L2 cache (implementation dependent)
  yield* Console.log(`Invalidated cache pattern: ${pattern}`)
})
```

### Testing Strategies

Comprehensive testing approaches for RequestBlock-based systems.

```typescript
import { RequestBlock, Effect, TestContext, Clock, Deferred } from "effect"

// Test utilities for RequestBlock
const TestRequestUtils = {
  // Create a mock resolver that tracks calls
  mockResolver: <A extends Request.Request<any, any>>(
    name: string,
    responses: Map<string, any>
  ) => {
    const callLog: Array<{ timestamp: number; requests: A[] }> = []
    
    const resolver = RequestResolver.makeBatched(
      (requests: NonEmptyArray<A>) => Effect.gen(function* () {
        const timestamp = yield* Clock.currentTimeMillis
        callLog.push({ timestamp, requests: [...requests] })
        
        requests.forEach(request => {
          const key = JSON.stringify(request)
          const response = responses.get(key)
          if (response) {
            Request.succeed(request, response)
          } else {
            Request.fail(request, new Error(`No mock response for ${key}`))
          }
        })
      })
    )
    
    return { resolver, getCallLog: () => callLog }
  },
  
  // Simulate network delays
  withNetworkDelay: <A>(
    resolver: RequestResolver.RequestResolver<A>,
    delay: Duration.Duration
  ) => RequestResolver.make(
    (request) => Effect.gen(function* () {
      yield* Effect.sleep(delay)
      return yield* resolver.resolve(request)
    })
  ),
  
  // Test request block batching behavior
  testBatching: (
    block: RequestBlock.RequestBlock,
    expectedBatches: number
  ) => Effect.gen(function* () {
    const analysis = analyzeBatching(block)
    
    if (analysis.batchGroups.size !== expectedBatches) {
      yield* Effect.fail(
        new Error(
          `Expected ${expectedBatches} batches, got ${analysis.batchGroups.size}`
        )
      )
    }
    
    return analysis
  })
}

// Example test suite
const requestBlockTests = Effect.gen(function* () {
  // Test 1: Verify parallel requests are batched correctly
  yield* Effect.gen(function* () {
    const responses = new Map([
      ['{"_tag":"GetUser","id":"user1"}', { id: "user1", name: "User 1" }],
      ['{"_tag":"GetUser","id":"user2"}', { id: "user2", name: "User 2" }],
      ['{"_tag":"GetUser","id":"user3"}', { id: "user3", name: "User 3" }]
    ])
    
    const { resolver, getCallLog } = TestRequestUtils.mockResolver("test-user", responses)
    
    const block = RequestBlock.parallel(
      RequestBlock.parallel(
        RequestBlock.single(resolver, GetUser({ id: "user1" })),
        RequestBlock.single(resolver, GetUser({ id: "user2" }))
      ),
      RequestBlock.single(resolver, GetUser({ id: "user3" }))
    )
    
    yield* Effect.runRequestBlock(block)
    
    const callLog = getCallLog()
    if (callLog.length !== 1) {
      yield* Effect.fail(new Error(`Expected 1 batch call, got ${callLog.length}`))
    }
    
    if (callLog[0].requests.length !== 3) {
      yield* Effect.fail(new Error(`Expected 3 requests in batch, got ${callLog[0].requests.length}`))
    }
    
    yield* Console.log("✓ Parallel batching test passed")
  }),
  
  // Test 2: Verify sequential requests maintain order
  yield* Effect.gen(function* () {
    const responses = new Map([
      ['{"_tag":"GetUser","id":"user1"}', { id: "user1", name: "User 1" }],
      ['{"_tag":"GetUserOrders","userId":"user1"}', [{ id: "order1", userId: "user1" }]]
    ])
    
    const { resolver: userResolver, getCallLog: getUserCalls } = 
      TestRequestUtils.mockResolver("test-user", responses)
    const { resolver: orderResolver, getCallLog: getOrderCalls } = 
      TestRequestUtils.mockResolver("test-order", responses)
    
    const block = RequestBlock.sequential(
      RequestBlock.single(userResolver, GetUser({ id: "user1" })),
      RequestBlock.single(orderResolver, GetUserOrders({ userId: "user1" }))
    )
    
    yield* Effect.runRequestBlock(block)
    
    const userCalls = getUserCalls()
    const orderCalls = getOrderCalls()
    
    if (userCalls.length !== 1 || orderCalls.length !== 1) {
      yield* Effect.fail(new Error("Expected exactly one call to each resolver"))
    }
    
    if (userCalls[0].timestamp >= orderCalls[0].timestamp) {
      yield* Effect.fail(new Error("User request should complete before order request"))
    }
    
    yield* Console.log("✓ Sequential ordering test passed")
  }),
  
  // Test 3: Performance under load
  yield* Effect.gen(function* () {
    const responses = new Map()
    for (let i = 0; i < 1000; i++) {
      responses.set(
        `{"_tag":"GetUser","id":"user${i}"}`,
        { id: `user${i}`, name: `User ${i}` }
      )
    }
    
    const { resolver } = TestRequestUtils.mockResolver("load-test", responses)
    
    const userBlocks = Array.from({ length: 1000 }, (_, i) => 
      RequestBlock.single(resolver, GetUser({ id: `user${i}` }))
    )
    
    const massiveBlock = userBlocks.reduce(
      (acc, block) => RequestBlock.parallel(acc, block),
      RequestBlock.empty
    )
    
    const startTime = yield* Clock.currentTimeMillis
    yield* Effect.runRequestBlock(massiveBlock)
    const endTime = yield* Clock.currentTimeMillis
    
    const duration = endTime - startTime
    yield* Console.log(`✓ Load test completed in ${duration}ms`)
    
    if (duration > 1000) {
      yield* Effect.fail(new Error(`Load test took too long: ${duration}ms`))
    }
  })
}).pipe(
  Effect.provide(TestContext.TestContext),
  Effect.provide(Clock.ClockTag, Clock.make())
)
```

## Conclusion

RequestBlock provides powerful batching optimization, intelligent request scheduling, and maximum parallelization while preserving dependencies for efficient data loading in Effect applications.

Key benefits:
- **Automatic Batching**: Groups similar requests to the same data source automatically
- **Ordering Preservation**: Maintains sequential dependencies while maximizing parallelism  
- **Performance Optimization**: Reduces network round-trips and improves resource utilization

RequestBlock is essential when building data-intensive applications that need to efficiently load data from multiple sources while maintaining correctness and performance. It transforms complex data loading scenarios into declarative, composable, and highly optimized request execution plans.