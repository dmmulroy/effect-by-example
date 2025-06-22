# HttpRouter: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem HttpRouter Solves

Building HTTP APIs in Node.js traditionally involves verbose routing setups, manual parameter parsing, inconsistent error handling, and complex middleware composition. Consider this typical Express.js approach:

```typescript
// Traditional Express approach - verbose and error-prone
app.get('/users/:id', async (req, res) => {
  try {
    const id = parseInt(req.params.id) // Manual validation
    if (isNaN(id)) {
      return res.status(400).json({ error: 'Invalid ID' })
    }
    
    const user = await getUserById(id)
    if (!user) {
      return res.status(404).json({ error: 'User not found' })
    }
    
    res.json(user)
  } catch (error) {
    console.error(error)
    res.status(500).json({ error: 'Internal server error' })
  }
})
```

This approach leads to:
- **Boilerplate Overload** - Repetitive error handling and validation code
- **Type Unsafe** - No compile-time guarantees about request/response types
- **Inconsistent Error Handling** - Manual error management across routes
- **Complex Middleware** - Difficult to compose and reason about middleware chains

### The HttpRouter Solution

HttpRouter provides a type-safe, composable approach to building HTTP APIs with automatic error handling and seamless integration with Effect's ecosystem:

```typescript
import { HttpRouter, HttpServer } from "@effect/platform"
import { Schema } from "@effect/schema"
import { Effect, Layer } from "effect"

const GetUserParams = Schema.Struct({
  id: Schema.NumberFromString
})

const api = HttpRouter.empty.pipe(
  HttpRouter.get("/users/:id", 
    Effect.gen(function* () {
      const { id } = yield* HttpRouter.schemaParams(GetUserParams)
      const user = yield* getUserById(id)
      return user
    })
  )
)
```

### Key Concepts

**Route Handler**: A function that processes HTTP requests and returns responses through Effect computations

**Path Parameters**: Dynamic segments in URLs (e.g., `/users/:id`) automatically extracted and validated

**Schema Integration**: Automatic request/response validation using @effect/schema for type safety

**Middleware Composition**: Reusable request processing logic that can be composed declaratively

**Error Boundaries**: Automatic error handling with structured error types and recovery strategies

## Basic Usage Patterns

### Pattern 1: Simple Route Definition

```typescript
import { HttpRouter, HttpServer } from "@effect/platform"
import { Effect, Layer } from "effect"

// Define a basic GET route
const router = HttpRouter.empty.pipe(
  HttpRouter.get("/health", Effect.succeed({ status: "ok", timestamp: Date.now() }))
)

// Create HTTP server
const ServerLive = HttpServer.serve().pipe(
  Layer.provide(HttpServer.layer({ port: 3000 }))
)

// Run the application
const app = router.pipe(
  HttpServer.serve(),
  Layer.launch
)
```

### Pattern 2: Route with Path Parameters

```typescript
import { Schema } from "@effect/schema"

const UserIdParams = Schema.Struct({
  id: Schema.NumberFromString
})

const router = HttpRouter.empty.pipe(
  HttpRouter.get("/users/:id", 
    Effect.gen(function* () {
      const { id } = yield* HttpRouter.schemaParams(UserIdParams)
      return { userId: id, name: `User ${id}` }
    })
  )
)
```

### Pattern 3: POST Route with Request Body

```typescript
const CreateUserBody = Schema.Struct({
  name: Schema.NonEmptyString,
  email: Schema.String.pipe(Schema.pattern(/^[^\s@]+@[^\s@]+\.[^\s@]+$/)),
  age: Schema.Number.pipe(Schema.between(0, 120))
})

const router = HttpRouter.empty.pipe(
  HttpRouter.post("/users",
    Effect.gen(function* () {
      const userData = yield* HttpRouter.schemaJson(CreateUserBody)
      const newUser = yield* createUser(userData)
      return newUser
    })
  )
)
```

## Real-World Examples

### Example 1: User Management API

A complete CRUD API for user management with validation and error handling:

```typescript
import { HttpRouter, HttpServer } from "@effect/platform"
import { Schema } from "@effect/schema"
import { Effect, Layer, Data } from "effect"

// Domain Models
const User = Schema.Struct({
  id: Schema.Number,
  name: Schema.NonEmptyString,
  email: Schema.String.pipe(Schema.pattern(/^[^\s@]+@[^\s@]+\.[^\s@]+$/)),
  age: Schema.Number.pipe(Schema.between(0, 120)),
  createdAt: Schema.DateFromString,
  updatedAt: Schema.DateFromString
})

const CreateUserRequest = Schema.Struct({
  name: Schema.NonEmptyString,
  email: Schema.String.pipe(Schema.pattern(/^[^\s@]+@[^\s@]+\.[^\s@]+$/)),
  age: Schema.Number.pipe(Schema.between(0, 120))
})

const UpdateUserRequest = Schema.Struct({
  name: Schema.optional(Schema.NonEmptyString),
  email: Schema.optional(Schema.String.pipe(Schema.pattern(/^[^\s@]+@[^\s@]+\.[^\s@]+$/))),
  age: Schema.optional(Schema.Number.pipe(Schema.between(0, 120)))
})

const UserIdParams = Schema.Struct({
  id: Schema.NumberFromString
})

// Error Types
class UserNotFoundError extends Data.TaggedError("UserNotFoundError")<{
  readonly userId: number
}> {}

class UserAlreadyExistsError extends Data.TaggedError("UserAlreadyExistsError")<{
  readonly email: string
}> {}

// Service Interface
interface UserService {
  readonly getUser: (id: number) => Effect.Effect<User.Type, UserNotFoundError>
  readonly createUser: (data: CreateUserRequest.Type) => Effect.Effect<User.Type, UserAlreadyExistsError>
  readonly updateUser: (id: number, data: UpdateUserRequest.Type) => Effect.Effect<User.Type, UserNotFoundError>
  readonly deleteUser: (id: number) => Effect.Effect<void, UserNotFoundError>
  readonly listUsers: () => Effect.Effect<Array<User.Type>>
}

const UserService = Effect.Tag<UserService>("UserService")

// API Router
const userRouter = HttpRouter.empty.pipe(
  // GET /users - List all users
  HttpRouter.get("/users", 
    Effect.gen(function* () {
      const userService = yield* UserService
      const users = yield* userService.listUsers()
      return { users, count: users.length }
    })
  ),
  
  // GET /users/:id - Get user by ID
  HttpRouter.get("/users/:id",
    Effect.gen(function* () {
      const { id } = yield* HttpRouter.schemaParams(UserIdParams)
      const userService = yield* UserService
      const user = yield* userService.getUser(id)
      return user
    })
  ),
  
  // POST /users - Create new user
  HttpRouter.post("/users",
    Effect.gen(function* () {
      const userData = yield* HttpRouter.schemaJson(CreateUserRequest)
      const userService = yield* UserService
      const newUser = yield* userService.createUser(userData)
      return newUser
    })
  ),
  
  // PUT /users/:id - Update user
  HttpRouter.put("/users/:id",
    Effect.gen(function* () {
      const { id } = yield* HttpRouter.schemaParams(UserIdParams)
      const updateData = yield* HttpRouter.schemaJson(UpdateUserRequest)
      const userService = yield* UserService
      const updatedUser = yield* userService.updateUser(id, updateData)
      return updatedUser
    })
  ),
  
  // DELETE /users/:id - Delete user
  HttpRouter.del("/users/:id",
    Effect.gen(function* () {
      const { id } = yield* HttpRouter.schemaParams(UserIdParams)
      const userService = yield* UserService
      yield* userService.deleteUser(id)
      return { message: "User deleted successfully" }
    })
  )
).pipe(
  // Global error handling for this router
  HttpRouter.catchTag("UserNotFoundError", (error) => 
    Effect.succeed({ error: "User not found", userId: error.userId })
  ),
  HttpRouter.catchTag("UserAlreadyExistsError", (error) =>
    Effect.succeed({ error: "User already exists", email: error.email })
  )
)
```

### Example 2: E-commerce Product API with Pagination

An API for managing products with search, filtering, and pagination:

```typescript
import { HttpRouter } from "@effect/platform"
import { Schema } from "@effect/schema"
import { Effect, Data } from "effect"

// Product Models
const Product = Schema.Struct({
  id: Schema.Number,
  name: Schema.NonEmptyString,
  description: Schema.String,
  price: Schema.Number.pipe(Schema.greaterThan(0)),
  category: Schema.String,
  inStock: Schema.Boolean,
  tags: Schema.Array(Schema.String),
  createdAt: Schema.DateFromString
})

const ProductSearchParams = Schema.Struct({
  page: Schema.optional(Schema.NumberFromString.pipe(Schema.greaterThan(0))),
  limit: Schema.optional(Schema.NumberFromString.pipe(Schema.between(1, 100))),
  category: Schema.optional(Schema.String),
  minPrice: Schema.optional(Schema.NumberFromString.pipe(Schema.greaterThanOrEqualTo(0))),
  maxPrice: Schema.optional(Schema.NumberFromString.pipe(Schema.greaterThanOrEqualTo(0))),
  inStock: Schema.optional(Schema.BooleanFromString),
  search: Schema.optional(Schema.String)
})

const ProductIdParams = Schema.Struct({
  id: Schema.NumberFromString
})

// Service Interface
interface ProductService {
  readonly searchProducts: (params: ProductSearchParams.Type) => Effect.Effect<{
    products: Array<Product.Type>
    total: number
    page: number
    limit: number
  }>
  readonly getProduct: (id: number) => Effect.Effect<Product.Type, ProductNotFoundError>
}

class ProductNotFoundError extends Data.TaggedError("ProductNotFoundError")<{
  readonly productId: number
}> {}

const ProductService = Effect.Tag<ProductService>("ProductService")

// Product API Router
const productRouter = HttpRouter.empty.pipe(
  // GET /products - Search and filter products
  HttpRouter.get("/products",
    Effect.gen(function* () {
      const searchParams = yield* HttpRouter.schemaParams(ProductSearchParams)
      const productService = yield* ProductService
      const result = yield* productService.searchProducts(searchParams)
      return {
        data: result.products,
        pagination: {
          page: result.page,
          limit: result.limit,
          total: result.total,
          pages: Math.ceil(result.total / result.limit)
        }
      }
    })
  ),
  
  // GET /products/:id - Get product by ID
  HttpRouter.get("/products/:id",
    Effect.gen(function* () {
      const { id } = yield* HttpRouter.schemaParams(ProductIdParams)
      const productService = yield* ProductService
      const product = yield* productService.getProduct(id)
      return product
    })
  )
).pipe(
  HttpRouter.catchTag("ProductNotFoundError", (error) =>
    Effect.succeed({ error: "Product not found", productId: error.productId })
  )
)
```

### Example 3: Blog API with Nested Routes and Authentication

A blog API demonstrating route nesting, authentication middleware, and complex relationships:

```typescript
import { HttpRouter } from "@effect/platform"
import { Schema } from "@effect/schema"
import { Effect, Data, Context } from "effect"

// Auth Models
const AuthUser = Schema.Struct({
  id: Schema.Number,
  username: Schema.String,
  role: Schema.Literal("admin", "author", "reader")
})

const AuthToken = Schema.Struct({
  token: Schema.String
})

// Blog Models
const BlogPost = Schema.Struct({
  id: Schema.Number,
  title: Schema.NonEmptyString,
  content: Schema.String,
  authorId: Schema.Number,
  author: AuthUser,
  published: Schema.Boolean,
  tags: Schema.Array(Schema.String),
  createdAt: Schema.DateFromString,
  updatedAt: Schema.DateFromString
})

const Comment = Schema.Struct({
  id: Schema.Number,
  postId: Schema.Number,
  authorId: Schema.Number,
  author: AuthUser,
  content: Schema.String,
  createdAt: Schema.DateFromString
})

const CreatePostRequest = Schema.Struct({
  title: Schema.NonEmptyString,
  content: Schema.String,
  tags: Schema.Array(Schema.String),
  published: Schema.Boolean
})

const CreateCommentRequest = Schema.Struct({
  content: Schema.NonEmptyString
})

// Error Types
class UnauthorizedError extends Data.TaggedError("UnauthorizedError")<{
  readonly message: string
}> {}

class PostNotFoundError extends Data.TaggedError("PostNotFoundError")<{
  readonly postId: number
}> {}

// Authentication Middleware
const authenticateUser = Effect.gen(function* () {
  const request = yield* HttpRouter.HttpServerRequest
  const authHeader = request.headers.authorization
  
  if (!authHeader?.startsWith("Bearer ")) {
    return yield* new UnauthorizedError({ message: "Missing or invalid token" })
  }
  
  const token = authHeader.substring(7)
  const authService = yield* AuthService
  const user = yield* authService.validateToken(token)
  return user
})

// Services
interface AuthService {
  readonly validateToken: (token: string) => Effect.Effect<AuthUser.Type, UnauthorizedError>
}

interface BlogService {
  readonly createPost: (userId: number, data: CreatePostRequest.Type) => Effect.Effect<BlogPost.Type>
  readonly getPost: (id: number) => Effect.Effect<BlogPost.Type, PostNotFoundError>
  readonly getUserPosts: (userId: number) => Effect.Effect<Array<BlogPost.Type>>
  readonly addComment: (postId: number, userId: number, data: CreateCommentRequest.Type) => Effect.Effect<Comment.Type, PostNotFoundError>
  readonly getPostComments: (postId: number) => Effect.Effect<Array<Comment.Type>, PostNotFoundError>
}

const AuthService = Effect.Tag<AuthService>("AuthService")
const BlogService = Effect.Tag<BlogService>("BlogService")

// Current User Context
const CurrentUser = Context.GenericTag<AuthUser.Type>("CurrentUser")

// Blog Router
const blogRouter = HttpRouter.empty.pipe(
  // POST /posts - Create new post (authenticated)
  HttpRouter.post("/posts",
    Effect.gen(function* () {
      const user = yield* authenticateUser
      const postData = yield* HttpRouter.schemaJson(CreatePostRequest)
      const blogService = yield* BlogService
      const newPost = yield* blogService.createPost(user.id, postData)
      return newPost
    })
  ),
  
  // GET /posts/:id - Get post by ID
  HttpRouter.get("/posts/:id",
    Effect.gen(function* () {
      const { id } = yield* HttpRouter.schemaParams(Schema.Struct({ id: Schema.NumberFromString }))
      const blogService = yield* BlogService
      const post = yield* blogService.getPost(id)
      return post
    })
  ),
  
  // GET /users/:userId/posts - Get user's posts
  HttpRouter.get("/users/:userId/posts",
    Effect.gen(function* () {
      const { userId } = yield* HttpRouter.schemaParams(Schema.Struct({ userId: Schema.NumberFromString }))
      const blogService = yield* BlogService
      const posts = yield* blogService.getUserPosts(userId)
      return { posts, count: posts.length }
    })
  ),
  
  // POST /posts/:id/comments - Add comment to post (authenticated)
  HttpRouter.post("/posts/:id/comments",
    Effect.gen(function* () {
      const user = yield* authenticateUser
      const { id } = yield* HttpRouter.schemaParams(Schema.Struct({ id: Schema.NumberFromString }))
      const commentData = yield* HttpRouter.schemaJson(CreateCommentRequest)
      const blogService = yield* BlogService
      const comment = yield* blogService.addComment(id, user.id, commentData)
      return comment
    })
  ),
  
  // GET /posts/:id/comments - Get post comments
  HttpRouter.get("/posts/:id/comments",
    Effect.gen(function* () {
      const { id } = yield* HttpRouter.schemaParams(Schema.Struct({ id: Schema.NumberFromString }))
      const blogService = yield* BlogService
      const comments = yield* blogService.getPostComments(id)
      return { comments, count: comments.length }
    })
  )
).pipe(
  // Error handling
  HttpRouter.catchTag("UnauthorizedError", (error) =>
    Effect.succeed({ error: error.message })
  ),
  HttpRouter.catchTag("PostNotFoundError", (error) =>
    Effect.succeed({ error: "Post not found", postId: error.postId })
  )
)
```

## Advanced Features Deep Dive

### Route Mounting and Composition

HttpRouter supports mounting sub-routers at specific paths, enabling modular API design:

#### Basic Route Mounting

```typescript
const apiV1 = HttpRouter.empty.pipe(
  HttpRouter.get("/users", getUsersHandler),
  HttpRouter.get("/products", getProductsHandler)
)

const apiV2 = HttpRouter.empty.pipe(
  HttpRouter.get("/users", getUsersV2Handler),
  HttpRouter.get("/products", getProductsV2Handler)
)

const mainRouter = HttpRouter.empty.pipe(
  HttpRouter.mount("/api/v1", apiV1),
  HttpRouter.mount("/api/v2", apiV2)
)
```

#### Real-World Modular API Example

```typescript
// Auth Module
const authRouter = HttpRouter.empty.pipe(
  HttpRouter.post("/login", loginHandler),
  HttpRouter.post("/register", registerHandler),
  HttpRouter.post("/refresh", refreshTokenHandler),
  HttpRouter.post("/logout", logoutHandler)
)

// User Module  
const userRouter = HttpRouter.empty.pipe(
  HttpRouter.get("/profile", getUserProfileHandler),
  HttpRouter.put("/profile", updateUserProfileHandler),
  HttpRouter.get("/settings", getUserSettingsHandler),
  HttpRouter.put("/settings", updateUserSettingsHandler)
)

// Admin Module
const adminRouter = HttpRouter.empty.pipe(
  HttpRouter.get("/users", listAllUsersHandler),
  HttpRouter.delete("/users/:id", deleteUserHandler),
  HttpRouter.get("/analytics", getAnalyticsHandler)
).pipe(
  // Admin-only middleware
  HttpRouter.use((app) => 
    Effect.gen(function* () {
      const user = yield* authenticateUser
      if (user.role !== "admin") {
        return yield* new UnauthorizedError({ message: "Admin access required" })
      }
      return yield* app
    })
  )
)

// Main Application Router
const appRouter = HttpRouter.empty.pipe(
  HttpRouter.mount("/auth", authRouter),
  HttpRouter.mount("/user", userRouter.pipe(HttpRouter.use(authMiddleware))),
  HttpRouter.mount("/admin", adminRouter),
  HttpRouter.get("/health", healthCheckHandler)
)
```

### Middleware Composition and Ordering

HttpRouter middleware executes in a specific order and can be composed at different levels:

#### Global Middleware

```typescript
const withLogging = (app: HttpRouter.HttpRouter) =>
  app.pipe(
    HttpRouter.use((handler) =>
      Effect.gen(function* () {
        const request = yield* HttpRouter.HttpServerRequest
        const start = Date.now()
        console.log(`${request.method} ${request.url} - Started`)
        
        const result = yield* handler
        
        const duration = Date.now() - start
        console.log(`${request.method} ${request.url} - Completed in ${duration}ms`)
        
        return result
      })
    )
  )

const withCors = (app: HttpRouter.HttpRouter) =>
  app.pipe(
    HttpRouter.use((handler) =>
      Effect.gen(function* () {
        const response = yield* handler
        // Add CORS headers
        return response.pipe(
          HttpServerResponse.setHeader("Access-Control-Allow-Origin", "*"),
          HttpServerResponse.setHeader("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS"),
          HttpServerResponse.setHeader("Access-Control-Allow-Headers", "Content-Type, Authorization")
        )
      })
    )
  )

const app = HttpRouter.empty.pipe(
  // Add routes
  HttpRouter.get("/api/data", dataHandler),
  // Apply middleware
  withLogging,
  withCors
)
```

#### Route-Specific Middleware

```typescript
import { pipe } from "effect"

const requireAuth = (handler: HttpRouter.Route.Handler) =>
  Effect.gen(function* () {
    const user = yield* authenticateUser
    return yield* handler.pipe(
      Effect.provideService(CurrentUser, user)
    )
  })

const requireRole = (role: string) => (handler: HttpRouter.Route.Handler) =>
  Effect.gen(function* () {
    const user = yield* CurrentUser
    if (user.role !== role) {
      return yield* new UnauthorizedError({ message: `${role} access required` })
    }
    return yield* handler
  })

const protectedRouter = HttpRouter.empty.pipe(
  HttpRouter.get("/admin/users", 
    pipe(
      Effect.gen(function* () {
        const users = yield* getAllUsers()
        return users
      }),
      requireAuth,
      requireRole("admin")
    )
  ),
  HttpRouter.get("/profile",
    Effect.gen(function* () {
      const user = yield* CurrentUser
      return user
    }).pipe(requireAuth)
  )
)
```

### Advanced Parameter Handling

#### Complex Path Parameters

```typescript
const ComplexParams = Schema.Struct({
  userId: Schema.NumberFromString,
  resourceId: Schema.NumberFromString,
  action: Schema.Literal("view", "edit", "delete")
})

const resourceRouter = HttpRouter.empty.pipe(
  HttpRouter.get("/users/:userId/resources/:resourceId/:action",
    Effect.gen(function* () {
      const { userId, resourceId, action } = yield* HttpRouter.schemaParams(ComplexParams)
      
      const user = yield* getUserById(userId)
      const resource = yield* getResourceById(resourceId)
      
      // Check permissions based on action
      yield* checkPermissions(user, resource, action)
      
      return { user, resource, action, timestamp: Date.now() }
    })
  )
)
```

#### Query Parameter Validation

```typescript
const SearchParams = Schema.Struct({
  q: Schema.optional(Schema.String),
  page: Schema.optional(Schema.NumberFromString.pipe(Schema.greaterThan(0))),
  limit: Schema.optional(Schema.NumberFromString.pipe(Schema.between(1, 100))),
  sortBy: Schema.optional(Schema.Literal("name", "date", "price")),
  sortOrder: Schema.optional(Schema.Literal("asc", "desc")),
  tags: Schema.optional(Schema.Array(Schema.String)),
  dateFrom: Schema.optional(Schema.DateFromString),
  dateTo: Schema.optional(Schema.DateFromString)
})

const searchRouter = HttpRouter.empty.pipe(
  HttpRouter.get("/search",
    Effect.gen(function* () {
      const params = yield* HttpRouter.schemaParams(SearchParams)
      
      // Build search query
      const query = {
        text: params.q,
        pagination: {
          page: params.page ?? 1,
          limit: params.limit ?? 20
        },
        sorting: {
          field: params.sortBy ?? "date",
          order: params.sortOrder ?? "desc"
        },
        filters: {
          tags: params.tags,
          dateRange: params.dateFrom && params.dateTo ? {
            from: params.dateFrom,
            to: params.dateTo
          } : undefined
        }
      }
      
      const results = yield* performSearch(query)
      return results
    })
  )
)
```

## Practical Patterns & Best Practices

### Pattern 1: Service Layer Integration

```typescript
// Create a helper for common service integration patterns
const withService = <T>(tag: Context.Tag<T, T>) => 
  <A, E, R>(effect: (service: T) => Effect.Effect<A, E, R>) =>
    Effect.gen(function* () {
      const service = yield* tag
      return yield* effect(service)
    })

// Usage in routes
const userRouter = HttpRouter.empty.pipe(
  HttpRouter.get("/users/:id",
    withService(UserService)((userService) =>
      Effect.gen(function* () {
        const { id } = yield* HttpRouter.schemaParams(UserIdParams)
        const user = yield* userService.getUser(id)
        return user
      })
    )
  )
)
```

### Pattern 2: Standardized Error Responses

```typescript
// Create consistent error response format
const ApiError = Schema.Struct({
  error: Schema.String,
  message: Schema.String,
  code: Schema.Number,
  timestamp: Schema.DateFromString,
  details: Schema.optional(Schema.Unknown)
})

const handleApiError = (error: unknown) => {
  if (error instanceof UserNotFoundError) {
    return Effect.succeed({
      error: "USER_NOT_FOUND",
      message: "The requested user was not found",
      code: 404,
      timestamp: new Date(),
      details: { userId: error.userId }
    })
  }
  
  if (error instanceof ValidationError) {
    return Effect.succeed({
      error: "VALIDATION_ERROR", 
      message: "Request validation failed",
      code: 400,
      timestamp: new Date(),
      details: { errors: error.errors }
    })
  }
  
  // Generic server error
  return Effect.succeed({
    error: "INTERNAL_ERROR",
    message: "An unexpected error occurred",
    code: 500,
    timestamp: new Date()
  })
}

// Apply to router
const router = HttpRouter.empty.pipe(
  // ... routes
  HttpRouter.catchAll(handleApiError)
)
```

### Pattern 3: Request/Response Transformation

```typescript
// Helper for standardized API responses
const ApiResponse = <T>(dataSchema: Schema.Schema<T>) => 
  Schema.Struct({
    data: dataSchema,
    success: Schema.Boolean,
    timestamp: Schema.DateFromString,
    requestId: Schema.String
  })

const withStandardResponse = <T>(data: T) => 
  Effect.gen(function* () {
    const requestId = yield* Effect.sync(() => crypto.randomUUID())
    return {
      data,
      success: true,
      timestamp: new Date(),
      requestId
    }
  })

// Usage in routes
const router = HttpRouter.empty.pipe(
  HttpRouter.get("/users",
    Effect.gen(function* () {
      const users = yield* getAllUsers()
      return yield* withStandardResponse(users)
    })
  )
)
```

### Pattern 4: Rate Limiting Per Route

```typescript
// Rate limiting middleware
interface RateLimiter {
  readonly checkLimit: (key: string, limit: number, window: number) => Effect.Effect<boolean>
}

const RateLimiter = Effect.Tag<RateLimiter>("RateLimiter")

const withRateLimit = (limit: number, windowMs: number) => 
  (handler: HttpRouter.Route.Handler) =>
    Effect.gen(function* () {
      const request = yield* HttpRouter.HttpServerRequest
      const rateLimiter = yield* RateLimiter
      
      const clientIp = request.headers["x-forwarded-for"] || request.remoteAddress || "unknown"
      const key = `${request.method}:${request.url}:${clientIp}`
      
      const allowed = yield* rateLimiter.checkLimit(key, limit, windowMs)
      
      if (!allowed) {
        return yield* new RateLimitError({ message: "Rate limit exceeded" })
      }
      
      return yield* handler
    })

// Apply to specific routes
const apiRouter = HttpRouter.empty.pipe(
  HttpRouter.post("/auth/login", 
    loginHandler.pipe(withRateLimit(5, 60000)) // 5 requests per minute
  ),
  HttpRouter.get("/api/data",
    dataHandler.pipe(withRateLimit(100, 60000)) // 100 requests per minute
  )
)
```

## Integration Examples

### Integration with HttpServer

```typescript
import { HttpRouter, HttpServer } from "@effect/platform"
import { NodeHttpServer } from "@effect/platform-node"
import { Effect, Layer } from "effect"

// Create the router
const router = HttpRouter.empty.pipe(
  HttpRouter.get("/api/users", getUsersHandler),
  HttpRouter.post("/api/users", createUserHandler)
)

// Create HTTP server layer
const ServerLive = HttpServer.serve(router).pipe(
  Layer.provide(NodeHttpServer.layer({ port: 3000 }))
)

// Run the application
const program = Effect.gen(function* () {
  yield* Effect.log("Starting HTTP server on port 3000")
  yield* Effect.never
}).pipe(
  Effect.provide(ServerLive),
  Effect.provide(DatabaseLive),
  Effect.provide(LoggerLive)
)

// Start the server
Effect.runPromise(program)
```

### Integration with Database Layer

```typescript
import { SqlClient } from "@effect/sql"
import { Effect, Layer } from "effect"

// Database service
const DatabaseLive = Layer.effect(
  DatabaseService,
  Effect.gen(function* () {
    const sql = yield* SqlClient.SqlClient
    
    return {
      getUser: (id: number) =>
        sql`SELECT * FROM users WHERE id = ${id}`.pipe(
          Effect.flatMap(rows => 
            rows.length > 0 
              ? Effect.succeed(rows[0])
              : Effect.fail(new UserNotFoundError({ userId: id }))
          )
        ),
      
      createUser: (data: CreateUserRequest.Type) =>
        sql`INSERT INTO users (name, email, age) VALUES (${data.name}, ${data.email}, ${data.age}) RETURNING *`.pipe(
          Effect.map(rows => rows[0])
        )
    }
  })
)

// Router with database integration
const userRouter = HttpRouter.empty.pipe(
  HttpRouter.get("/users/:id",
    Effect.gen(function* () {
      const { id } = yield* HttpRouter.schemaParams(UserIdParams)
      const db = yield* DatabaseService
      const user = yield* db.getUser(id)
      return user
    })
  )
).pipe(
  Effect.provide(DatabaseLive)
)
```

### Testing Strategies

#### Unit Testing Individual Routes

```typescript
import { HttpRouter } from "@effect/platform"
import { Effect, Layer } from "effect"
import { describe, it, expect } from "@effect/vitest"

// Mock service for testing
const MockUserService = Layer.succeed(
  UserService,
  {
    getUser: (id: number) => 
      id === 1 
        ? Effect.succeed({ id: 1, name: "Test User", email: "test@example.com" })
        : Effect.fail(new UserNotFoundError({ userId: id })),
    
    createUser: (data) => 
      Effect.succeed({ id: 2, ...data, createdAt: new Date() })
  }
)

describe("User Router", () => {
  it("should get user by ID", () =>
    Effect.gen(function* () {
      const router = createUserRouter()
      const response = yield* router.pipe(
        HttpRouter.get("/users/1", getUserHandler),
        Effect.provide(MockUserService)
      )
      
      expect(response).toEqual({
        id: 1,
        name: "Test User", 
        email: "test@example.com"
      })
    })
  )
  
  it("should handle user not found", () =>
    Effect.gen(function* () {
      const router = createUserRouter()
      const result = yield* router.pipe(
        HttpRouter.get("/users/999", getUserHandler),
        Effect.provide(MockUserService),
        Effect.either
      )
      
      expect(result._tag).toBe("Left")
    })
  )
})
```

#### Integration Testing with Test Client

```typescript
import { HttpClient } from "@effect/platform"
import { Effect, Layer } from "effect"

// Create test client
const TestClient = HttpClient.layer.pipe(
  Layer.provide(HttpClient.layerTestClient)
)

describe("User API Integration", () => {
  it("should create and retrieve user", () =>
    Effect.gen(function* () {
      const client = yield* HttpClient.HttpClient
      
      // Create user
      const createResponse = yield* client.post("/api/users", {
        body: JSON.stringify({ 
          name: "John Doe", 
          email: "john@example.com",
          age: 25 
        })
      })
      
      const newUser = yield* createResponse.json
      expect(newUser.name).toBe("John Doe")
      
      // Get user
      const getResponse = yield* client.get(`/api/users/${newUser.id}`)
      const retrievedUser = yield* getResponse.json
      
      expect(retrievedUser.id).toBe(newUser.id)
      expect(retrievedUser.name).toBe("John Doe")
    }).pipe(
      Effect.provide(TestClient),
      Effect.provide(DatabaseLive)
    )
  )
})
```

## Conclusion

HttpRouter provides type-safe, composable HTTP routing for Effect applications, eliminating boilerplate and ensuring consistent error handling across your API.

Key benefits:
- **Type Safety**: Compile-time guarantees for request/response types through schema integration
- **Composability**: Declarative middleware composition and route mounting for modular APIs
- **Error Handling**: Structured error types with automatic transformation and recovery strategies

HttpRouter is ideal for building production-ready APIs that need strong type safety, comprehensive error handling, and seamless integration with Effect's ecosystem of services and layers.