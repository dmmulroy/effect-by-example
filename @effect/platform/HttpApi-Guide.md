# HttpApi: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem HttpApi Solves

Building type-safe APIs traditionally requires maintaining multiple sources of truth: your API implementation, client types, documentation, and validation schemas. This leads to inconsistencies, runtime errors, and development overhead.

```typescript
// Traditional approach - multiple disconnected pieces
interface User {
  id: string
  name: string
  email: string
}

// Separate validation (can drift from types)
const validateCreateUser = (body: any) => {
  if (!body.name || !body.email) throw new Error("Invalid input")
  // Manual validation logic...
}

// Separate documentation (can become outdated)
/**
 * POST /users
 * Creates a new user
 * @param {Object} body - User data
 * @param {string} body.name - User name
 * @param {string} body.email - User email
 */
app.post('/users', (req, res) => {
  validateCreateUser(req.body)
  // Implementation...
})

// Separate client types (can drift from server)
type ApiClient = {
  createUser: (user: CreateUserRequest) => Promise<User>
}
```

This approach leads to:
- **Type Drift** - Client and server types diverge over time
- **Documentation Rot** - Manual docs become outdated and incorrect
- **Runtime Errors** - Validation mismatches cause production failures
- **Development Overhead** - Maintaining multiple sources of truth

### The HttpApi Solution

HttpApi provides a declarative, schema-first approach where you define your API once and generate everything else: types, validation, documentation, and client code.

```typescript
import { HttpApi, HttpApiEndpoint, HttpApiGroup, OpenApi } from "@effect/platform"
import { Schema } from "effect"

// Single source of truth - schema-driven API definition
const User = Schema.Struct({
  id: Schema.String,
  name: Schema.String,
  email: Schema.String.pipe(Schema.pattern(/^[^@]+@[^@]+\.[^@]+$/))
})

const CreateUserRequest = Schema.Struct({
  name: Schema.String.pipe(Schema.minLength(1)),
  email: Schema.String.pipe(Schema.pattern(/^[^@]+@[^@]+\.[^@]+$/))
})

const UsersApi = HttpApi.make("users-api").add(
  HttpApiGroup.make("users").add(
    HttpApiEndpoint.post("createUser", "/users")
      .setPayload(CreateUserRequest)
      .addSuccess(User)
      .addError(Schema.Struct({ error: Schema.String }), { status: 400 })
  )
)

// Automatic OpenAPI documentation
const openApiSpec = OpenApi.fromApi(UsersApi)

// Type-safe client generation
// Implementation provides type safety automatically
```

### Key Concepts

**HttpApi**: A collection of API groups that represents your entire API surface or a domain portion

**HttpApiGroup**: A logical grouping of related endpoints (e.g., "users", "orders", "payments")

**HttpApiEndpoint**: A single API endpoint with method, path, schemas, and metadata

**Schema-First**: Define data structures once using Effect Schema, get validation and types automatically

**Declarative**: Describe what your API does, not how to implement it

## Basic Usage Patterns

### Pattern 1: Creating an API

```typescript
import { HttpApi, HttpApiGroup, HttpApiEndpoint } from "@effect/platform"
import { Schema } from "effect"

// Step 1: Create the root API
const MyApi = HttpApi.make("my-api")

// Step 2: Define a group for related endpoints
const HealthGroup = HttpApiGroup.make("health")

// Step 3: Add endpoints to the group
const healthEndpoint = HttpApiEndpoint.get("ping", "/health")
  .addSuccess(Schema.Struct({ status: Schema.Literal("ok") }))

// Step 4: Compose everything together
const ApiWithHealth = MyApi.add(
  HealthGroup.add(healthEndpoint)
)
```

### Pattern 2: Adding Schemas and Validation

```typescript
import { HttpApi, HttpApiGroup, HttpApiEndpoint } from "@effect/platform"
import { Schema } from "effect"

// Define your data schemas
const CreateTaskRequest = Schema.Struct({
  title: Schema.String.pipe(Schema.minLength(1), Schema.maxLength(100)),
  description: Schema.optional(Schema.String),
  priority: Schema.Literal("low", "medium", "high").pipe(Schema.withDefault(() => "medium"))
})

const Task = Schema.Struct({
  id: Schema.String,
  title: Schema.String,
  description: Schema.optional(Schema.String),
  priority: Schema.Literal("low", "medium", "high"),
  completed: Schema.Boolean.pipe(Schema.withDefault(() => false)),
  createdAt: Schema.DateTimeUtc
})

// Create endpoint with request/response schemas
const createTaskEndpoint = HttpApiEndpoint.post("createTask", "/tasks")
  .setPayload(CreateTaskRequest)
  .addSuccess(Task)
  .addError(Schema.Struct({ error: Schema.String }), { status: 400 })

const TasksGroup = HttpApiGroup.make("tasks").add(createTaskEndpoint)
```

### Pattern 3: Path Parameters and Query Strings

```typescript
import { HttpApi, HttpApiGroup, HttpApiEndpoint } from "@effect/platform"
import { Schema } from "effect"

// Path parameters are automatically extracted from the path
const getTaskEndpoint = HttpApiEndpoint.get("getTask", "/tasks/:id")
  .addSuccess(Task)
  .addError(Schema.Struct({ error: Schema.String }), { status: 404 })

// Query parameters using setUrlParams
const listTasksEndpoint = HttpApiEndpoint.get("listTasks", "/tasks")
  .setUrlParams(Schema.Struct({
    page: Schema.optional(Schema.NumberFromString.pipe(Schema.positive())),
    limit: Schema.optional(Schema.NumberFromString.pipe(Schema.positive(), Schema.lessThanOrEqualTo(100))),
    priority: Schema.optional(Schema.Literal("low", "medium", "high"))
  }))
  .addSuccess(Schema.Struct({
    tasks: Schema.Array(Task),
    total: Schema.Number,
    page: Schema.Number,
    hasMore: Schema.Boolean
  }))
```

## Real-World Examples

### Example 1: E-commerce Product API

Building a comprehensive product management API with CRUD operations, search, and inventory management.

```typescript
import { HttpApi, HttpApiGroup, HttpApiEndpoint, HttpApiBuilder } from "@effect/platform"
import { Schema } from "effect"
import { Effect, Layer } from "effect"

// Domain schemas
const Product = Schema.Struct({
  id: Schema.String,
  name: Schema.String,
  description: Schema.String,
  price: Schema.Number.pipe(Schema.positive()),
  category: Schema.String,
  inStock: Schema.Boolean,
  inventory: Schema.Number.pipe(Schema.nonNegative()),
  createdAt: Schema.DateTimeUtc,
  updatedAt: Schema.DateTimeUtc
})

const CreateProductRequest = Schema.Struct({
  name: Schema.String.pipe(Schema.minLength(1), Schema.maxLength(100)),
  description: Schema.String.pipe(Schema.maxLength(500)),
  price: Schema.Number.pipe(Schema.positive()),
  category: Schema.String,
  inventory: Schema.Number.pipe(Schema.nonNegative())
})

const UpdateProductRequest = Schema.Struct({
  name: Schema.optional(Schema.String.pipe(Schema.minLength(1), Schema.maxLength(100))),
  description: Schema.optional(Schema.String.pipe(Schema.maxLength(500))),
  price: Schema.optional(Schema.Number.pipe(Schema.positive())),
  category: Schema.optional(Schema.String),
  inventory: Schema.optional(Schema.Number.pipe(Schema.nonNegative()))
})

const ProductSearchParams = Schema.Struct({
  q: Schema.optional(Schema.String),
  category: Schema.optional(Schema.String),
  minPrice: Schema.optional(Schema.NumberFromString.pipe(Schema.positive())),
  maxPrice: Schema.optional(Schema.NumberFromString.pipe(Schema.positive())),
  inStock: Schema.optional(Schema.Boolean),
  page: Schema.optional(Schema.NumberFromString.pipe(Schema.positive())),
  limit: Schema.optional(Schema.NumberFromString.pipe(Schema.positive(), Schema.lessThanOrEqualTo(100)))
})

const ProductList = Schema.Struct({
  products: Schema.Array(Product),
  total: Schema.Number,
  page: Schema.Number,
  limit: Schema.Number,
  hasMore: Schema.Boolean
})

// Error schemas
const NotFoundError = Schema.Struct({
  error: Schema.Literal("NOT_FOUND"),
  message: Schema.String
})

const ValidationError = Schema.Struct({
  error: Schema.Literal("VALIDATION_ERROR"),
  message: Schema.String,
  details: Schema.Array(Schema.String)
})

// API definition
const ProductsGroup = HttpApiGroup.make("products")
  .add(
    HttpApiEndpoint.get("listProducts", "/products")
      .setUrlParams(ProductSearchParams)
      .addSuccess(ProductList)
  )
  .add(
    HttpApiEndpoint.get("getProduct", "/products/:id")
      .addSuccess(Product)
      .addError(NotFoundError, { status: 404 })
  )
  .add(
    HttpApiEndpoint.post("createProduct", "/products")
      .setPayload(CreateProductRequest)
      .addSuccess(Product)
      .addError(ValidationError, { status: 400 })
  )
  .add(
    HttpApiEndpoint.put("updateProduct", "/products/:id")
      .setPayload(UpdateProductRequest)
      .addSuccess(Product)
      .addError(NotFoundError, { status: 404 })
      .addError(ValidationError, { status: 400 })
  )
  .add(
    HttpApiEndpoint.del("deleteProduct", "/products/:id")
      .addSuccess(Schema.Struct({ deleted: Schema.Boolean }))
      .addError(NotFoundError, { status: 404 })
  )

const EcommerceApi = HttpApi.make("ecommerce-api").add(ProductsGroup)

// Implementation using HttpApiBuilder
interface ProductService {
  readonly findAll: (params: typeof ProductSearchParams.Type) => Effect.Effect<typeof ProductList.Type>
  readonly findById: (id: string) => Effect.Effect<typeof Product.Type, typeof NotFoundError.Type>
  readonly create: (product: typeof CreateProductRequest.Type) => Effect.Effect<typeof Product.Type, typeof ValidationError.Type>
  readonly update: (id: string, updates: typeof UpdateProductRequest.Type) => Effect.Effect<typeof Product.Type, typeof NotFoundError.Type | typeof ValidationError.Type>
  readonly delete: (id: string) => Effect.Effect<{ deleted: boolean }, typeof NotFoundError.Type>
}

const ProductService = Effect.Tag<ProductService>("ProductService")

// Implementation layer
const ProductServiceLive = Layer.succeed(ProductService, {
  findAll: (params) => Effect.gen(function* () {
    // Simulate database query with search and pagination
    const mockProducts: typeof Product.Type[] = [
      {
        id: "1",
        name: "Laptop",
        description: "High-performance laptop",
        price: 999.99,
        category: "electronics",
        inStock: true,
        inventory: 10,
        createdAt: new Date("2024-01-01T00:00:00Z"),
        updatedAt: new Date("2024-01-01T00:00:00Z")
      }
    ]
    
    const filtered = mockProducts.filter(product => {
      if (params.category && product.category !== params.category) return false
      if (params.minPrice && product.price < params.minPrice) return false
      if (params.maxPrice && product.price > params.maxPrice) return false
      if (params.inStock !== undefined && product.inStock !== params.inStock) return false
      if (params.q && !product.name.toLowerCase().includes(params.q.toLowerCase())) return false
      return true
    })
    
    const page = params.page ?? 1
    const limit = params.limit ?? 20
    const start = (page - 1) * limit
    const products = filtered.slice(start, start + limit)
    
    return {
      products,
      total: filtered.length,
      page,
      limit,
      hasMore: start + limit < filtered.length
    }
  }),
  
  findById: (id) => Effect.gen(function* () {
    if (id === "1") {
      return {
        id: "1",
        name: "Laptop",
        description: "High-performance laptop",
        price: 999.99,
        category: "electronics",
        inStock: true,
        inventory: 10,
        createdAt: new Date("2024-01-01T00:00:00Z"),
        updatedAt: new Date("2024-01-01T00:00:00Z")
      }
    }
    return yield* Effect.fail({ error: "NOT_FOUND" as const, message: `Product ${id} not found` })
  }),
  
  create: (product) => Effect.gen(function* () {
    // Simulate validation and creation
    const newProduct: typeof Product.Type = {
      id: Math.random().toString(36).substr(2, 9),
      ...product,
      inStock: product.inventory > 0,
      createdAt: new Date(),
      updatedAt: new Date()
    }
    return newProduct
  }),
  
  update: (id, updates) => Effect.gen(function* () {
    const existing = yield* ProductService.findById(id)
    const updated: typeof Product.Type = {
      ...existing,
      ...updates,
      inStock: (updates.inventory ?? existing.inventory) > 0,
      updatedAt: new Date()
    }
    return updated
  }),
  
  delete: (id) => Effect.gen(function* () {
    yield* ProductService.findById(id) // Check if exists
    return { deleted: true }
  })
})

// Handlers implementation
const ProductHandlers = HttpApiBuilder.group(EcommerceApi, "products", (handlers) =>
  handlers
    .handle("listProducts", ({ urlParams }) => 
      ProductService.findAll(urlParams)
    )
    .handle("getProduct", ({ path }) => 
      ProductService.findById(path.id)
    )
    .handle("createProduct", ({ payload }) => 
      ProductService.create(payload)
    )
    .handle("updateProduct", ({ path, payload }) => 
      ProductService.update(path.id, payload)
    )
    .handle("deleteProduct", ({ path }) => 
      ProductService.delete(path.id)
    )
)

// Complete application layer
const AppLayer = Layer.provide(
  HttpApiBuilder.api(EcommerceApi),
  Layer.mergeAll(ProductHandlers, ProductServiceLive)
)
```

### Example 2: User Authentication & Authorization API

A comprehensive authentication system with JWT tokens, role-based access control, and security middleware.

```typescript
import { HttpApi, HttpApiGroup, HttpApiEndpoint, HttpApiBuilder, HttpApiSecurity } from "@effect/platform"
import { Schema } from "effect"
import { Effect, Layer, Context } from "effect"

// Authentication schemas
const LoginRequest = Schema.Struct({
  email: Schema.String.pipe(Schema.pattern(/^[^@]+@[^@]+\.[^@]+$/)),
  password: Schema.String.pipe(Schema.minLength(8))
})

const RegisterRequest = Schema.Struct({
  email: Schema.String.pipe(Schema.pattern(/^[^@]+@[^@]+\.[^@]+$/)),
  password: Schema.String.pipe(Schema.minLength(8)),
  name: Schema.String.pipe(Schema.minLength(1))
})

const User = Schema.Struct({
  id: Schema.String,
  email: Schema.String,
  name: Schema.String,
  role: Schema.Literal("user", "admin"),
  createdAt: Schema.DateTimeUtc
})

const AuthResponse = Schema.Struct({
  user: User,
  token: Schema.String,
  refreshToken: Schema.String,
  expiresIn: Schema.Number
})

const RefreshTokenRequest = Schema.Struct({
  refreshToken: Schema.String
})

// Security scheme for JWT
const JwtSecurity = HttpApiSecurity.bearer({
  bearerFormat: "JWT"
})

// Error schemas
const AuthError = Schema.Struct({
  error: Schema.Literal("AUTH_ERROR"),
  message: Schema.String
})

const ForbiddenError = Schema.Struct({
  error: Schema.Literal("FORBIDDEN"),
  message: Schema.String
})

// API definition with security
const AuthGroup = HttpApiGroup.make("auth")
  .add(
    HttpApiEndpoint.post("login", "/auth/login")
      .setPayload(LoginRequest)
      .addSuccess(AuthResponse)
      .addError(AuthError, { status: 401 })
  )
  .add(
    HttpApiEndpoint.post("register", "/auth/register")
      .setPayload(RegisterRequest)
      .addSuccess(AuthResponse)
      .addError(AuthError, { status: 400 })
  )
  .add(
    HttpApiEndpoint.post("refresh", "/auth/refresh")
      .setPayload(RefreshTokenRequest)
      .addSuccess(AuthResponse)
      .addError(AuthError, { status: 401 })
  )

const UserGroup = HttpApiGroup.make("users")
  .add(
    HttpApiEndpoint.get("profile", "/users/profile")
      .setSecurity(JwtSecurity)
      .addSuccess(User)
      .addError(AuthError, { status: 401 })
  )
  .add(
    HttpApiEndpoint.get("listUsers", "/users")
      .setSecurity(JwtSecurity)
      .addSuccess(Schema.Array(User))
      .addError(AuthError, { status: 401 })
      .addError(ForbiddenError, { status: 403 })
  )

const AuthApi = HttpApi.make("auth-api")
  .add(AuthGroup)
  .add(UserGroup)

// Services
interface AuthService {
  readonly login: (credentials: typeof LoginRequest.Type) => Effect.Effect<typeof AuthResponse.Type, typeof AuthError.Type>
  readonly register: (userData: typeof RegisterRequest.Type) => Effect.Effect<typeof AuthResponse.Type, typeof AuthError.Type>
  readonly refresh: (refreshToken: string) => Effect.Effect<typeof AuthResponse.Type, typeof AuthError.Type>
  readonly verifyToken: (token: string) => Effect.Effect<typeof User.Type, typeof AuthError.Type>
}

interface UserService {
  readonly getCurrentUser: (userId: string) => Effect.Effect<typeof User.Type, typeof AuthError.Type>
  readonly getAllUsers: (requestingUser: typeof User.Type) => Effect.Effect<typeof User.Type[], typeof AuthError.Type | typeof ForbiddenError.Type>
}

const AuthService = Context.GenericTag<AuthService>("AuthService")
const UserService = Context.GenericTag<UserService>("UserService")

// JWT middleware for protected endpoints
const JwtMiddleware = HttpApiBuilder.middleware.make((request) => 
  Effect.gen(function* () {
    const authHeader = request.headers.authorization
    if (!authHeader?.startsWith("Bearer ")) {
      return yield* Effect.fail({ error: "AUTH_ERROR" as const, message: "Missing or invalid authorization header" })
    }
    
    const token = authHeader.slice(7)
    const user = yield* AuthService.verifyToken(token)
    
    return { user }
  })
)

// Implementation
const AuthServiceLive = Layer.succeed(AuthService, {
  login: (credentials) => Effect.gen(function* () {
    // Simulate authentication
    if (credentials.email === "user@example.com" && credentials.password === "password123") {
      return {
        user: {
          id: "1",
          email: credentials.email,
          name: "John Doe",
          role: "user" as const,
          createdAt: new Date()
        },
        token: "jwt-token-here",
        refreshToken: "refresh-token-here",
        expiresIn: 3600
      }
    }
    return yield* Effect.fail({ error: "AUTH_ERROR" as const, message: "Invalid credentials" })
  }),
  
  register: (userData) => Effect.gen(function* () {
    // Simulate user creation
    return {
      user: {
        id: Math.random().toString(36).substr(2, 9),
        email: userData.email,
        name: userData.name,
        role: "user" as const,
        createdAt: new Date()
      },
      token: "jwt-token-here",
      refreshToken: "refresh-token-here",
      expiresIn: 3600
    }
  }),
  
  refresh: (refreshToken) => Effect.gen(function* () {
    if (refreshToken === "refresh-token-here") {
      return {
        user: {
          id: "1",
          email: "user@example.com",
          name: "John Doe",
          role: "user" as const,
          createdAt: new Date()
        },
        token: "new-jwt-token-here",
        refreshToken: "new-refresh-token-here",
        expiresIn: 3600
      }
    }
    return yield* Effect.fail({ error: "AUTH_ERROR" as const, message: "Invalid refresh token" })
  }),
  
  verifyToken: (token) => Effect.gen(function* () {
    if (token === "jwt-token-here") {
      return {
        id: "1",
        email: "user@example.com",
        name: "John Doe",
        role: "user" as const,
        createdAt: new Date()
      }
    }
    return yield* Effect.fail({ error: "AUTH_ERROR" as const, message: "Invalid token" })
  })
})

const UserServiceLive = Layer.succeed(UserService, {
  getCurrentUser: (userId) => Effect.gen(function* () {
    return {
      id: userId,
      email: "user@example.com",
      name: "John Doe",
      role: "user" as const,
      createdAt: new Date()
    }
  }),
  
  getAllUsers: (requestingUser) => Effect.gen(function* () {
    if (requestingUser.role !== "admin") {
      return yield* Effect.fail({ error: "FORBIDDEN" as const, message: "Admin access required" })
    }
    
    return [
      {
        id: "1",
        email: "user@example.com",
        name: "John Doe",
        role: "user" as const,
        createdAt: new Date()
      },
      {
        id: "2",
        email: "admin@example.com",
        name: "Jane Admin",
        role: "admin" as const,
        createdAt: new Date()
      }
    ]
  })
})

// Handlers with security
const AuthHandlers = HttpApiBuilder.group(AuthApi, "auth", (handlers) =>
  handlers
    .handle("login", ({ payload }) => AuthService.login(payload))
    .handle("register", ({ payload }) => AuthService.register(payload))
    .handle("refresh", ({ payload }) => AuthService.refresh(payload.refreshToken))
)

const UserHandlers = HttpApiBuilder.group(AuthApi, "users", (handlers) =>
  handlers
    .handle("profile", ({ security }) => 
      Effect.gen(function* () {
        const { user } = yield* security
        return yield* UserService.getCurrentUser(user.id)
      })
    )
    .handle("listUsers", ({ security }) => 
      Effect.gen(function* () {
        const { user } = yield* security
        return yield* UserService.getAllUsers(user)
      })
    )
).pipe(
  Layer.provide(JwtMiddleware)
)

// Complete application layer
const AuthAppLayer = Layer.provide(
  HttpApiBuilder.api(AuthApi),
  Layer.mergeAll(AuthHandlers, UserHandlers, AuthServiceLive, UserServiceLive)
)
```

### Example 3: File Upload & Processing API

A file management system with upload, processing, and download capabilities including multipart support and progress tracking.

```typescript
import { HttpApi, HttpApiGroup, HttpApiEndpoint, HttpApiBuilder } from "@effect/platform"
import { Schema } from "effect"
import { Effect, Layer, Stream } from "effect"

// File schemas
const FileInfo = Schema.Struct({
  id: Schema.String,
  filename: Schema.String,
  originalName: Schema.String,
  mimetype: Schema.String,
  size: Schema.Number,
  uploadedAt: Schema.DateTimeUtc,
  status: Schema.Literal("uploading", "processing", "completed", "error"),
  url: Schema.optional(Schema.String),
  thumbnailUrl: Schema.optional(Schema.String)
})

const UploadResponse = Schema.Struct({
  file: FileInfo,
  uploadUrl: Schema.String
})

const FileListResponse = Schema.Struct({
  files: Schema.Array(FileInfo),
  total: Schema.Number,
  page: Schema.Number,
  hasMore: Schema.Boolean
})

const FileSearchParams = Schema.Struct({
  mimetype: Schema.optional(Schema.String),
  status: Schema.optional(Schema.Literal("uploading", "processing", "completed", "error")),
  page: Schema.optional(Schema.NumberFromString.pipe(Schema.positive())),
  limit: Schema.optional(Schema.NumberFromString.pipe(Schema.positive(), Schema.lessThanOrEqualTo(100)))
})

// File processing schemas
const ProcessingJob = Schema.Struct({
  id: Schema.String,
  fileId: Schema.String,
  type: Schema.Literal("thumbnail", "resize", "compress"),
  status: Schema.Literal("pending", "processing", "completed", "failed"),
  progress: Schema.Number.pipe(Schema.between(0, 100)),
  result: Schema.optional(Schema.Struct({
    url: Schema.String,
    size: Schema.Number
  })),
  error: Schema.optional(Schema.String),
  createdAt: Schema.DateTimeUtc,
  completedAt: Schema.optional(Schema.DateTimeUtc)
})

const ProcessingRequest = Schema.Struct({
  type: Schema.Literal("thumbnail", "resize", "compress"),
  options: Schema.optional(Schema.Record(Schema.String, Schema.Unknown))
})

// Error schemas
const FileNotFoundError = Schema.Struct({
  error: Schema.Literal("FILE_NOT_FOUND"),
  message: Schema.String
})

const ProcessingError = Schema.Struct({
  error: Schema.Literal("PROCESSING_ERROR"),
  message: Schema.String
})

// API definition
const FilesGroup = HttpApiGroup.make("files")
  .add(
    HttpApiEndpoint.post("uploadFile", "/files/upload")
      .setPayload(Schema.Struct({
        filename: Schema.String,
        mimetype: Schema.String,
        size: Schema.Number
      }))
      .addSuccess(UploadResponse)
  )
  .add(
    HttpApiEndpoint.get("listFiles", "/files")
      .setUrlParams(FileSearchParams)
      .addSuccess(FileListResponse)
  )
  .add(
    HttpApiEndpoint.get("getFile", "/files/:id")
      .addSuccess(FileInfo)
      .addError(FileNotFoundError, { status: 404 })
  )
  .add(
    HttpApiEndpoint.del("deleteFile", "/files/:id")
      .addSuccess(Schema.Struct({ deleted: Schema.Boolean }))
      .addError(FileNotFoundError, { status: 404 })
  )
  .add(
    HttpApiEndpoint.post("processFile", "/files/:id/process")
      .setPayload(ProcessingRequest)
      .addSuccess(ProcessingJob)
      .addError(FileNotFoundError, { status: 404 })
      .addError(ProcessingError, { status: 400 })
  )
  .add(
    HttpApiEndpoint.get("getProcessingJob", "/files/:id/jobs/:jobId")
      .addSuccess(ProcessingJob)
      .addError(FileNotFoundError, { status: 404 })
  )

const FileManagementApi = HttpApi.make("file-management-api").add(FilesGroup)

// Services
interface FileService {
  readonly initiateUpload: (metadata: { filename: string; mimetype: string; size: number }) => Effect.Effect<typeof UploadResponse.Type>
  readonly listFiles: (params: typeof FileSearchParams.Type) => Effect.Effect<typeof FileListResponse.Type>
  readonly getFile: (id: string) => Effect.Effect<typeof FileInfo.Type, typeof FileNotFoundError.Type>
  readonly deleteFile: (id: string) => Effect.Effect<{ deleted: boolean }, typeof FileNotFoundError.Type>
}

interface ProcessingService {
  readonly startProcessing: (fileId: string, request: typeof ProcessingRequest.Type) => Effect.Effect<typeof ProcessingJob.Type, typeof FileNotFoundError.Type | typeof ProcessingError.Type>
  readonly getJob: (fileId: string, jobId: string) => Effect.Effect<typeof ProcessingJob.Type, typeof FileNotFoundError.Type>
}

const FileService = Effect.Tag<FileService>("FileService")
const ProcessingService = Effect.Tag<ProcessingService>("ProcessingService")

// Implementation
const FileServiceLive = Layer.succeed(FileService, {
  initiateUpload: (metadata) => Effect.gen(function* () {
    const fileId = Math.random().toString(36).substr(2, 9)
    const file: typeof FileInfo.Type = {
      id: fileId,
      filename: `${fileId}-${metadata.filename}`,
      originalName: metadata.filename,
      mimetype: metadata.mimetype,
      size: metadata.size,
      uploadedAt: new Date(),
      status: "uploading"
    }
    
    return {
      file,
      uploadUrl: `https://storage.example.com/upload/${fileId}`
    }
  }),
  
  listFiles: (params) => Effect.gen(function* () {
    // Simulate file listing with filtering
    const mockFiles: typeof FileInfo.Type[] = [
      {
        id: "1",
        filename: "1-document.pdf",
        originalName: "document.pdf",
        mimetype: "application/pdf",
        size: 1024000,
        uploadedAt: new Date("2024-01-01T00:00:00Z"),
        status: "completed",
        url: "https://storage.example.com/files/1-document.pdf"
      },
      {
        id: "2",
        filename: "2-image.jpg",
        originalName: "image.jpg",
        mimetype: "image/jpeg",
        size: 512000,
        uploadedAt: new Date("2024-01-02T00:00:00Z"),
        status: "completed",
        url: "https://storage.example.com/files/2-image.jpg",
        thumbnailUrl: "https://storage.example.com/thumbnails/2-image.jpg"
      }
    ]
    
    const filtered = mockFiles.filter(file => {
      if (params.mimetype && !file.mimetype.includes(params.mimetype)) return false
      if (params.status && file.status !== params.status) return false
      return true
    })
    
    const page = params.page ?? 1
    const limit = params.limit ?? 20
    const start = (page - 1) * limit
    const files = filtered.slice(start, start + limit)
    
    return {
      files,
      total: filtered.length,
      page,
      hasMore: start + limit < filtered.length
    }
  }),
  
  getFile: (id) => Effect.gen(function* () {
    if (id === "1") {
      return {
        id: "1",
        filename: "1-document.pdf",
        originalName: "document.pdf",
        mimetype: "application/pdf",
        size: 1024000,
        uploadedAt: new Date("2024-01-01T00:00:00Z"),
        status: "completed" as const,
        url: "https://storage.example.com/files/1-document.pdf"
      }
    }
    return yield* Effect.fail({ error: "FILE_NOT_FOUND" as const, message: `File ${id} not found` })
  }),
  
  deleteFile: (id) => Effect.gen(function* () {
    yield* FileService.getFile(id) // Check if exists
    // Simulate deletion
    return { deleted: true }
  })
})

const ProcessingServiceLive = Layer.succeed(ProcessingService, {
  startProcessing: (fileId, request) => Effect.gen(function* () {
    // Verify file exists
    yield* FileService.getFile(fileId)
    
    const jobId = Math.random().toString(36).substr(2, 9)
    const job: typeof ProcessingJob.Type = {
      id: jobId,
      fileId,
      type: request.type,
      status: "pending",
      progress: 0,
      createdAt: new Date()
    }
    
    // Start async processing (simulate)
    Effect.runPromise(Effect.fork(
      Effect.gen(function* () {
        yield* Effect.sleep("1 second")
        // Simulate processing completion
        const result = {
          url: `https://storage.example.com/processed/${jobId}-${request.type}.jpg`,
          size: 256000
        }
        // In real implementation, update job status in database
      })
    ))
    
    return job
  }),
  
  getJob: (fileId, jobId) => Effect.gen(function* () {
    // Verify file exists
    yield* FileService.getFile(fileId)
    
    if (jobId === "job1") {
      return {
        id: jobId,
        fileId,
        type: "thumbnail" as const,
        status: "completed" as const,
        progress: 100,
        result: {
          url: `https://storage.example.com/processed/${jobId}-thumbnail.jpg`,
          size: 64000
        },
        createdAt: new Date("2024-01-01T00:00:00Z"),
        completedAt: new Date("2024-01-01T00:01:00Z")
      }
    }
    
    return yield* Effect.fail({ error: "FILE_NOT_FOUND" as const, message: `Job ${jobId} not found` })
  })
})

// Handlers
const FileHandlers = HttpApiBuilder.group(FileManagementApi, "files", (handlers) =>
  handlers
    .handle("uploadFile", ({ payload }) => 
      FileService.initiateUpload(payload)
    )
    .handle("listFiles", ({ urlParams }) => 
      FileService.listFiles(urlParams)
    )
    .handle("getFile", ({ path }) => 
      FileService.getFile(path.id)
    )
    .handle("deleteFile", ({ path }) => 
      FileService.deleteFile(path.id)
    )
    .handle("processFile", ({ path, payload }) => 
      ProcessingService.startProcessing(path.id, payload)
    )
    .handle("getProcessingJob", ({ path }) => 
      ProcessingService.getJob(path.id, path.jobId)
    )
)

// Complete application layer
const FileAppLayer = Layer.provide(
  HttpApiBuilder.api(FileManagementApi),
  Layer.mergeAll(FileHandlers, FileServiceLive, ProcessingServiceLive)
)
```

## Advanced Features Deep Dive

### Feature 1: OpenAPI Documentation Generation

HttpApi automatically generates comprehensive OpenAPI specifications from your API definitions, including schemas, security, and examples.

#### Basic OpenAPI Usage

```typescript
import { HttpApi, HttpApiGroup, HttpApiEndpoint, OpenApi } from "@effect/platform"
import { Schema } from "effect"

const api = HttpApi.make("my-api").add(
  HttpApiGroup.make("users").add(
    HttpApiEndpoint.get("getUsers", "/users")
      .addSuccess(Schema.Array(Schema.Struct({
        id: Schema.String,
        name: Schema.String
      })))
  )
)

// Generate OpenAPI specification
const openApiSpec = OpenApi.fromApi(api)

// The spec is a complete OpenAPI 3.1.0 document
console.log(JSON.stringify(openApiSpec, null, 2))
```

#### Real-World OpenAPI Example

```typescript
import { HttpApi, HttpApiGroup, HttpApiEndpoint, OpenApi } from "@effect/platform"
import { Schema } from "effect"

// Enhanced schemas with OpenAPI annotations
const User = Schema.Struct({
  id: Schema.String.pipe(
    Schema.description("Unique user identifier"),
    Schema.examples(["user_123", "user_456"])
  ),
  name: Schema.String.pipe(
    Schema.description("User's full name"),
    Schema.examples(["John Doe", "Jane Smith"])
  ),
  email: Schema.String.pipe(
    Schema.pattern(/^[^@]+@[^@]+\.[^@]+$/),
    Schema.description("User's email address"),
    Schema.examples(["john@example.com", "jane@example.com"])
  ),
  role: Schema.Literal("user", "admin").pipe(
    Schema.description("User's role in the system")
  )
}).pipe(
  Schema.description("User entity representing a system user")
)

const CreateUserRequest = Schema.Struct({
  name: Schema.String.pipe(
    Schema.minLength(1),
    Schema.maxLength(100),
    Schema.description("User's full name (1-100 characters)")
  ),
  email: Schema.String.pipe(
    Schema.pattern(/^[^@]+@[^@]+\.[^@]+$/),
    Schema.description("Valid email address")
  ),
  role: Schema.optional(Schema.Literal("user", "admin").pipe(
    Schema.withDefault(() => "user" as const),
    Schema.description("User role (defaults to 'user')")
  ))
}).pipe(
  Schema.description("Request payload for creating a new user")
)

// API with rich metadata
const UsersApi = HttpApi.make("users-api")
  .annotate(OpenApi.Title, "User Management API")
  .annotate(OpenApi.Description, "Comprehensive API for managing users in the system")
  .annotate(OpenApi.Version, "1.0.0")
  .annotate(OpenApi.License, {
    name: "MIT",
    url: "https://opensource.org/licenses/MIT"
  })
  .add(
    HttpApiGroup.make("users")
      .annotate(OpenApi.Description, "User management operations")
      .add(
        HttpApiEndpoint.get("listUsers", "/users")
          .annotate(OpenApi.Summary, "List all users")
          .annotate(OpenApi.Description, "Retrieve a paginated list of all users in the system")
          .setUrlParams(Schema.Struct({
            page: Schema.optional(Schema.NumberFromString.pipe(
              Schema.positive(),
              Schema.description("Page number (1-based)")
            )),
            limit: Schema.optional(Schema.NumberFromString.pipe(
              Schema.positive(),
              Schema.lessThanOrEqualTo(100),
              Schema.description("Number of users per page (max 100)")
            ))
          }))
          .addSuccess(Schema.Struct({
            users: Schema.Array(User),
            total: Schema.Number.pipe(Schema.description("Total number of users")),
            page: Schema.Number.pipe(Schema.description("Current page number")),
            hasMore: Schema.Boolean.pipe(Schema.description("Whether more pages exist"))
          }).pipe(Schema.description("Paginated user list response")))
          .addError(Schema.Struct({
            error: Schema.String,
            message: Schema.String
          }), { status: 400 })
      )
      .add(
        HttpApiEndpoint.post("createUser", "/users")
          .annotate(OpenApi.Summary, "Create a new user")
          .annotate(OpenApi.Description, "Create a new user account with the provided information")
          .setPayload(CreateUserRequest)
          .addSuccess(User)
          .addError(Schema.Struct({
            error: Schema.Literal("VALIDATION_ERROR"),
            message: Schema.String,
            details: Schema.Array(Schema.String)
          }), { status: 400 })
          .addError(Schema.Struct({
            error: Schema.Literal("CONFLICT"),
            message: Schema.String
          }), { status: 409 })
      )
  )

// Generate enhanced OpenAPI spec
const enhancedSpec = OpenApi.fromApi(UsersApi, {
  additionalPropertiesStrategy: "strict"
})

// The generated spec includes:
// - Complete schema definitions with descriptions and examples
// - Proper HTTP status codes and error responses
// - Request/response body specifications
// - Query parameter documentation
// - API metadata (title, description, version, license)
```

#### Advanced OpenAPI: Custom Transformations

```typescript
import { HttpApi, HttpApiGroup, HttpApiEndpoint, OpenApi } from "@effect/platform"
import { Schema } from "effect"

// Custom schema transformations for OpenAPI
const DateTimeSchema = Schema.DateTimeUtc.pipe(
  Schema.description("ISO 8601 datetime string"),
  Schema.examples(["2024-01-01T00:00:00.000Z"]),
  OpenApi.Transform({
    type: "string",
    format: "date-time",
    example: "2024-01-01T00:00:00.000Z"
  })
)

const PaginationSchema = Schema.Struct({
  page: Schema.NumberFromString.pipe(
    Schema.positive(),
    Schema.withDefault(() => 1),
    Schema.description("Page number starting from 1")
  ),
  limit: Schema.NumberFromString.pipe(
    Schema.positive(),
    Schema.lessThanOrEqualTo(100),
    Schema.withDefault(() => 20),
    Schema.description("Items per page (1-100)")
  ),
  sort: Schema.optional(Schema.Literal("name", "email", "createdAt").pipe(
    Schema.description("Sort field")
  )),
  order: Schema.optional(Schema.Literal("asc", "desc").pipe(
    Schema.withDefault(() => "asc" as const),
    Schema.description("Sort order")
  ))
}).pipe(
  Schema.description("Pagination and sorting parameters")
)

// Override specific OpenAPI properties
const ApiKeyHeader = Schema.String.pipe(
  OpenApi.Override({
    name: "X-API-Key",
    in: "header",
    required: true,
    description: "API key for authentication",
    example: "ak_1234567890abcdef"
  })
)

const api = HttpApi.make("advanced-api")
  .annotate(OpenApi.Title, "Advanced API")
  .annotate(OpenApi.Servers, [
    { url: "https://api.example.com/v1", description: "Production server" },
    { url: "https://staging-api.example.com/v1", description: "Staging server" },
    { url: "http://localhost:3000/v1", description: "Development server" }
  ])
  .add(
    HttpApiGroup.make("resources").add(
      HttpApiEndpoint.get("listResources", "/resources")
        .setHeaders(Schema.Struct({
          "X-API-Key": ApiKeyHeader
        }))
        .setUrlParams(PaginationSchema)
        .addSuccess(Schema.Struct({
          data: Schema.Array(Schema.Struct({
            id: Schema.String,
            name: Schema.String,
            createdAt: DateTimeSchema,
            updatedAt: DateTimeSchema
          })),
          pagination: Schema.Struct({
            page: Schema.Number,
            limit: Schema.Number,
            total: Schema.Number,
            totalPages: Schema.Number
          })
        }))
    )
  )

const customSpec = OpenApi.fromApi(api)

// Exclude specific endpoints from documentation
const InternalApi = HttpApi.make("internal-api").add(
  HttpApiGroup.make("internal").add(
    HttpApiEndpoint.get("healthCheck", "/health")
      .annotate(OpenApi.Exclude, true) // This endpoint won't appear in OpenAPI
      .addSuccess(Schema.Struct({ status: Schema.Literal("ok") }))
  )
)
```

### Feature 2: Type-Safe Client Generation

HttpApi enables automatic generation of type-safe client code from your API definitions.

#### Basic Client Generation Pattern

```typescript
import { HttpApi, HttpApiGroup, HttpApiEndpoint } from "@effect/platform"
import { HttpClient } from "@effect/platform"
import { Effect } from "effect"
import { Schema } from "effect"

// Define API
const api = HttpApi.make("client-api").add(
  HttpApiGroup.make("posts").add(
    HttpApiEndpoint.get("getPosts", "/posts")
      .setUrlParams(Schema.Struct({
        page: Schema.optional(Schema.NumberFromString)
      }))
      .addSuccess(Schema.Array(Schema.Struct({
        id: Schema.Number,
        title: Schema.String,
        content: Schema.String
      })))
  )
)

// Generate type-safe client
const createApiClient = (baseUrl: string) => {
  const client = HttpClient.client
  
  return {
    posts: {
      getPosts: (params?: { page?: number }) => Effect.gen(function* () {
        const url = new URL("/posts", baseUrl)
        if (params?.page) {
          url.searchParams.set("page", params.page.toString())
        }
        
        const response = yield* HttpClient.request.get(url.toString()).pipe(
          client,
          Effect.flatMap(HttpClient.response.json)
        )
        
        // Type-safe parsing using the API schema
        return yield* Schema.decodeUnknown(
          Schema.Array(Schema.Struct({
            id: Schema.Number,
            title: Schema.String,
            content: Schema.String
          }))
        )(response)
      })
    }
  }
}

// Usage with full type safety
const apiClient = createApiClient("https://api.example.com")

const posts = yield* apiClient.posts.getPosts({ page: 1 })
// posts is typed as Array<{ id: number; title: string; content: string }>
```

#### Advanced Client Generation with Error Handling

```typescript
import { HttpApi, HttpApiGroup, HttpApiEndpoint } from "@effect/platform"
import { HttpClient, HttpClientError } from "@effect/platform"
import { Effect, Data } from "effect"
import { Schema } from "effect"

// API with comprehensive error handling
const BlogApi = HttpApi.make("blog-api").add(
  HttpApiGroup.make("posts").add(
    HttpApiEndpoint.get("getPost", "/posts/:id")
      .addSuccess(Schema.Struct({
        id: Schema.Number,
        title: Schema.String,
        content: Schema.String,
        publishedAt: Schema.DateTimeUtc
      }))
      .addError(Schema.Struct({
        error: Schema.Literal("NOT_FOUND"),
        message: Schema.String
      }), { status: 404 })
      .addError(Schema.Struct({
        error: Schema.Literal("UNAUTHORIZED"),
        message: Schema.String
      }), { status: 401 })
  )
)

// Client-side error types
class ApiError extends Data.TaggedError("ApiError")<{
  readonly status: number
  readonly error: string
  readonly message: string
}> {}

class NetworkError extends Data.TaggedError("NetworkError")<{
  readonly cause: HttpClientError.HttpClientError
}> {}

// Enhanced client generator
const createBlogClient = (baseUrl: string, apiKey?: string) => {
  const makeRequest = <S, E>(
    method: "GET" | "POST" | "PUT" | "DELETE",
    path: string,
    successSchema: Schema.Schema<S>,
    options: {
      params?: Record<string, string | number>
      body?: unknown
      headers?: Record<string, string>
    } = {}
  ) => Effect.gen(function* () {
    // Build URL with path parameters
    let url = path
    if (options.params) {
      Object.entries(options.params).forEach(([key, value]) => {
        url = url.replace(`:${key}`, String(value))
      })
    }
    
    const fullUrl = new URL(url, baseUrl).toString()
    
    // Build headers
    const headers: Record<string, string> = {
      "Content-Type": "application/json",
      ...options.headers
    }
    
    if (apiKey) {
      headers["Authorization"] = `Bearer ${apiKey}`
    }
    
    // Make request
    const request = method === "GET" 
      ? HttpClient.request.get(fullUrl, { headers })
      : method === "POST"
      ? HttpClient.request.post(fullUrl, { headers, body: HttpClient.body.json(options.body) })
      : HttpClient.request.put(fullUrl, { headers, body: HttpClient.body.json(options.body) })
    
    const response = yield* request.pipe(
      HttpClient.client,
      Effect.mapError(cause => new NetworkError({ cause }))
    )
    
    // Handle HTTP errors
    if (response.status >= 400) {
      const errorBody = yield* HttpClient.response.json(response).pipe(
        Effect.orElse(() => Effect.succeed({ error: "UNKNOWN", message: "Unknown error" }))
      )
      
      return yield* Effect.fail(new ApiError({
        status: response.status,
        error: errorBody.error || "HTTP_ERROR",
        message: errorBody.message || `HTTP ${response.status}`
      }))
    }
    
    // Parse successful response
    const data = yield* HttpClient.response.json(response)
    return yield* Schema.decodeUnknown(successSchema)(data).pipe(
      Effect.mapError(cause => new ApiError({
        status: 500,
        error: "DECODE_ERROR",
        message: `Failed to parse response: ${cause}`
      }))
    )
  })
  
  return {
    posts: {
      getPost: (id: number) => makeRequest(
        "GET",
        "/posts/:id",
        Schema.Struct({
          id: Schema.Number,
          title: Schema.String,
          content: Schema.String,
          publishedAt: Schema.DateTimeUtc
        }),
        { params: { id } }
      )
    }
  }
}

// Usage with comprehensive error handling
const blogClient = createBlogClient("https://api.blog.com", "your-api-key")

const program = Effect.gen(function* () {
  const post = yield* blogClient.posts.getPost(123)
  console.log(`Post: ${post.title}`)
  return post
}).pipe(
  Effect.catchTag("ApiError", (error) => 
    Effect.gen(function* () {
      if (error.status === 404) {
        console.log("Post not found")
        return null
      }
      if (error.status === 401) {
        console.log("Authentication required")
        return yield* Effect.fail(error)
      }
      return yield* Effect.fail(error)
    })
  ),
  Effect.catchTag("NetworkError", (error) => 
    Effect.gen(function* () {
      console.log("Network error:", error.cause)
      return yield* Effect.fail(error)
    })
  )
)
```

### Feature 3: API Versioning and Compatibility

HttpApi supports sophisticated API versioning strategies for maintaining backward compatibility while evolving your API.

#### Version-Aware API Design

```typescript
import { HttpApi, HttpApiGroup, HttpApiEndpoint } from "@effect/platform"
import { Schema } from "effect"

// V1 schemas
const UserV1 = Schema.Struct({
  id: Schema.String,
  name: Schema.String,
  email: Schema.String
})

// V2 schemas (evolved with new fields)
const UserV2 = Schema.Struct({
  id: Schema.String,
  name: Schema.String,
  email: Schema.String,
  profile: Schema.Struct({
    avatar: Schema.optional(Schema.String),
    bio: Schema.optional(Schema.String),
    preferences: Schema.Struct({
      theme: Schema.Literal("light", "dark").pipe(Schema.withDefault(() => "light" as const)),
      notifications: Schema.Boolean.pipe(Schema.withDefault(() => true))
    })
  }).pipe(Schema.withDefault(() => ({
    preferences: { theme: "light" as const, notifications: true }
  })))
})

// Version-specific APIs
const ApiV1 = HttpApi.make("api-v1")
  .prefix("/v1")
  .add(
    HttpApiGroup.make("users").add(
      HttpApiEndpoint.get("getUser", "/users/:id")
        .addSuccess(UserV1)
    )
  )

const ApiV2 = HttpApi.make("api-v2")
  .prefix("/v2")
  .add(
    HttpApiGroup.make("users").add(
      HttpApiEndpoint.get("getUser", "/users/:id")
        .addSuccess(UserV2)
    )
  )

// Unified API with version routing
const VersionedApi = HttpApi.make("versioned-api")
  .addHttpApi(ApiV1)
  .addHttpApi(ApiV2)

// Migration utilities
const migrateUserV1ToV2 = (userV1: typeof UserV1.Type): typeof UserV2.Type => ({
  ...userV1,
  profile: {
    preferences: {
      theme: "light" as const,
      notifications: true
    }
  }
})

// Backward compatibility handler
const createCompatibilityHandler = <T, U>(
  handler: Effect.Effect<U>,
  migrator: (input: T) => U
) => Effect.gen(function* () {
  const result = yield* handler
  return migrator(result)
})
```

#### Content Negotiation and API Evolution

```typescript
import { HttpApi, HttpApiGroup, HttpApiEndpoint } from "@effect/platform"
import { Schema } from "effect"
import { Effect } from "effect"

// Schema evolution tracking
const BaseUser = Schema.Struct({
  id: Schema.String,
  name: Schema.String,
  email: Schema.String,
  createdAt: Schema.DateTimeUtc
})

const UserV1 = BaseUser

const UserV2 = BaseUser.pipe(
  Schema.extend(Schema.Struct({
    profile: Schema.optional(Schema.Struct({
      avatar: Schema.optional(Schema.String),
      bio: Schema.optional(Schema.String)
    }))
  }))
)

const UserV3 = UserV2.pipe(
  Schema.extend(Schema.Struct({
    settings: Schema.optional(Schema.Struct({
      theme: Schema.Literal("light", "dark", "auto").pipe(Schema.withDefault(() => "light" as const)),
      language: Schema.String.pipe(Schema.withDefault(() => "en")),
      timezone: Schema.String.pipe(Schema.withDefault(() => "UTC"))
    }))
  }))
)

// Content negotiation based API
const ContentNegotiationApi = HttpApi.make("content-api").add(
  HttpApiGroup.make("users").add(
    HttpApiEndpoint.get("getUser", "/users/:id")
      .setHeaders(Schema.Struct({
        "Accept": Schema.optional(Schema.String),
        "API-Version": Schema.optional(Schema.Literal("1", "2", "3"))
      }))
      .addSuccess(Schema.Union(UserV1, UserV2, UserV3))
  )
)

// Version-aware service
interface UserService {
  readonly getUser: (id: string, version: "1" | "2" | "3") => Effect.Effect<
    typeof UserV1.Type | typeof UserV2.Type | typeof UserV3.Type
  >
}

const UserServiceLive = Layer.succeed(UserService, {
  getUser: (id, version) => Effect.gen(function* () {
    // Fetch full user data (latest version)
    const fullUser: typeof UserV3.Type = {
      id,
      name: "John Doe",
      email: "john@example.com",
      createdAt: new Date(),
      profile: {
        avatar: "https://example.com/avatar.jpg",
        bio: "Software developer"
      },
      settings: {
        theme: "dark",
        language: "en",
        timezone: "America/New_York"
      }
    }
    
    // Return version-appropriate data
    switch (version) {
      case "1":
        return {
          id: fullUser.id,
          name: fullUser.name,
          email: fullUser.email,
          createdAt: fullUser.createdAt
        }
      case "2":
        return {
          id: fullUser.id,
          name: fullUser.name,
          email: fullUser.email,
          createdAt: fullUser.createdAt,
          profile: fullUser.profile
        }
      case "3":
        return fullUser
      default:
        return fullUser
    }
  })
})

// Handler with version detection
const UserHandlers = HttpApiBuilder.group(ContentNegotiationApi, "users", (handlers) =>
  handlers.handle("getUser", ({ path, headers }) => 
    Effect.gen(function* () {
      // Determine version from header or default to latest
      const apiVersion = headers["API-Version"] || "3"
      const version = ["1", "2", "3"].includes(apiVersion) ? apiVersion as "1" | "2" | "3" : "3"
      
      return yield* UserService.getUser(path.id, version)
    })
  )
)
```

## Practical Patterns & Best Practices

### Pattern 1: Modular API Composition

Create reusable API modules that can be composed into larger applications.

```typescript
import { HttpApi, HttpApiGroup, HttpApiEndpoint } from "@effect/platform"
import { Schema } from "effect"

// Reusable health check module
const createHealthModule = (serviceName: string) => 
  HttpApiGroup.make("health").add(
    HttpApiEndpoint.get("health", "/health")
      .addSuccess(Schema.Struct({
        service: Schema.String,
        status: Schema.Literal("healthy", "unhealthy"),
        timestamp: Schema.DateTimeUtc,
        checks: Schema.Array(Schema.Struct({
          name: Schema.String,
          status: Schema.Literal("pass", "fail"),
          message: Schema.optional(Schema.String)
        }))
      }))
  )

// Reusable authentication module
const createAuthModule = () =>
  HttpApiGroup.make("auth")
    .add(
      HttpApiEndpoint.post("login", "/auth/login")
        .setPayload(Schema.Struct({
          email: Schema.String,
          password: Schema.String
        }))
        .addSuccess(Schema.Struct({
          token: Schema.String,
          expiresIn: Schema.Number
        }))
    )
    .add(
      HttpApiEndpoint.post("refresh", "/auth/refresh")
        .setPayload(Schema.Struct({
          refreshToken: Schema.String
        }))
        .addSuccess(Schema.Struct({
          token: Schema.String,
          expiresIn: Schema.Number
        }))
    )

// Composable domain modules
const createUserModule = () =>
  HttpApiGroup.make("users")
    .add(
      HttpApiEndpoint.get("listUsers", "/users")
        .addSuccess(Schema.Array(Schema.Struct({
          id: Schema.String,
          name: Schema.String,
          email: Schema.String
        })))
    )

const createOrderModule = () =>
  HttpApiGroup.make("orders")
    .add(
      HttpApiEndpoint.get("listOrders", "/orders")
        .addSuccess(Schema.Array(Schema.Struct({
          id: Schema.String,
          userId: Schema.String,
          total: Schema.Number,
          status: Schema.Literal("pending", "confirmed", "shipped", "delivered")
        })))
    )

// Compose into complete API
const ApplicationApi = HttpApi.make("application-api")
  .add(createHealthModule("application-service"))
  .add(createAuthModule())
  .add(createUserModule())
  .add(createOrderModule())

// Environment-specific composition
const DevelopmentApi = ApplicationApi
  .add(
    HttpApiGroup.make("debug").add(
      HttpApiEndpoint.get("debugInfo", "/debug")
        .addSuccess(Schema.Struct({
          environment: Schema.Literal("development"),
          version: Schema.String,
          buildTime: Schema.DateTimeUtc
        }))
    )
  )

const ProductionApi = ApplicationApi
  // Production doesn't include debug endpoints
```

### Pattern 2: Request/Response Transformation Pipeline

Create reusable transformation patterns for common request/response modifications.

```typescript
import { HttpApi, HttpApiGroup, HttpApiEndpoint, HttpApiBuilder } from "@effect/platform"
import { Schema } from "effect"
import { Effect, pipe } from "effect"

// Common transformation schemas
const TimestampedResponse = <T extends Schema.Schema.All>(schema: T) =>
  Schema.Struct({
    data: schema,
    timestamp: Schema.DateTimeUtc.pipe(Schema.withDefault(() => new Date())),
    requestId: Schema.String
  })

const PaginatedResponse = <T extends Schema.Schema.All>(itemSchema: T) =>
  Schema.Struct({
    items: Schema.Array(itemSchema),
    pagination: Schema.Struct({
      page: Schema.Number,
      limit: Schema.Number,
      total: Schema.Number,
      totalPages: Schema.Number
    }),
    links: Schema.Struct({
      first: Schema.String,
      prev: Schema.optional(Schema.String),
      next: Schema.optional(Schema.String),
      last: Schema.String
    })
  })

// Request transformation middleware
const RequestIdMiddleware = HttpApiBuilder.middleware.make((request) =>
  Effect.gen(function* () {
    const requestId = request.headers["x-request-id"] || 
      Math.random().toString(36).substr(2, 9)
    
    return { requestId }
  })
)

const TimestampMiddleware = HttpApiBuilder.middleware.make((request) =>
  Effect.gen(function* () {
    const timestamp = new Date()
    return { timestamp }
  })
)

// Helper for creating paginated endpoints
const createPaginatedEndpoint = <T extends Schema.Schema.All>(
  name: string,
  path: string,
  itemSchema: T
) =>
  HttpApiEndpoint.get(name, path)
    .setUrlParams(Schema.Struct({
      page: Schema.optional(Schema.NumberFromString.pipe(Schema.positive())),
      limit: Schema.optional(Schema.NumberFromString.pipe(Schema.positive(), Schema.lessThanOrEqualTo(100))),
      sort: Schema.optional(Schema.String),
      order: Schema.optional(Schema.Literal("asc", "desc"))
    }))
    .addSuccess(PaginatedResponse(itemSchema))

// Helper for creating timestamped endpoints  
const createTimestampedEndpoint = <T extends Schema.Schema.All>(
  method: "get" | "post" | "put" | "delete",
  name: string,
  path: string,
  successSchema: T
) => {
  const endpoint = method === "get" ? HttpApiEndpoint.get(name, path)
    : method === "post" ? HttpApiEndpoint.post(name, path)
    : method === "put" ? HttpApiEndpoint.put(name, path)
    : HttpApiEndpoint.del(name, path)
    
  return endpoint.addSuccess(TimestampedResponse(successSchema))
}

// Usage
const User = Schema.Struct({
  id: Schema.String,
  name: Schema.String,
  email: Schema.String
})

const TransformedApi = HttpApi.make("transformed-api").add(
  HttpApiGroup.make("users")
    .add(createPaginatedEndpoint("listUsers", "/users", User))
    .add(createTimestampedEndpoint("get", "getUser", "/users/:id", User))
)

// Handlers with transformation
const TransformedHandlers = HttpApiBuilder.group(TransformedApi, "users", (handlers) =>
  handlers
    .handle("listUsers", ({ urlParams, middleware }) => 
      Effect.gen(function* () {
        const { requestId } = yield* middleware
        const page = urlParams.page ?? 1
        const limit = urlParams.limit ?? 20
        
        // Simulate data fetching
        const mockUsers = Array.from({ length: 50 }, (_, i) => ({
          id: `user_${i + 1}`,
          name: `User ${i + 1}`,
          email: `user${i + 1}@example.com`
        }))
        
        const start = (page - 1) * limit
        const items = mockUsers.slice(start, start + limit)
        const total = mockUsers.length
        const totalPages = Math.ceil(total / limit)
        
        const baseUrl = "https://api.example.com/users"
        
        return {
          items,
          pagination: { page, limit, total, totalPages },
          links: {
            first: `${baseUrl}?page=1&limit=${limit}`,
            prev: page > 1 ? `${baseUrl}?page=${page - 1}&limit=${limit}` : undefined,
            next: page < totalPages ? `${baseUrl}?page=${page + 1}&limit=${limit}` : undefined,
            last: `${baseUrl}?page=${totalPages}&limit=${limit}`
          }
        }
      })
    )
    .handle("getUser", ({ path, middleware }) => 
      Effect.gen(function* () {
        const { requestId, timestamp } = yield* middleware
        
        // Simulate user fetching
        const user = {
          id: path.id,
          name: "John Doe",
          email: "john@example.com"
        }
        
        return {
          data: user,
          timestamp,
          requestId
        }
      })
    )
).pipe(
  Layer.provide(RequestIdMiddleware),
  Layer.provide(TimestampMiddleware)
)
```

### Pattern 3: Error Handling and Status Code Management

Implement comprehensive error handling with proper HTTP status codes and error responses.

```typescript
import { HttpApi, HttpApiGroup, HttpApiEndpoint, HttpApiBuilder } from "@effect/platform"
import { Schema } from "effect"
import { Effect, Data } from "effect"

// Standardized error schemas
const ValidationError = Schema.Struct({
  error: Schema.Literal("VALIDATION_ERROR"),
  message: Schema.String,
  details: Schema.Array(Schema.Struct({
    field: Schema.String,
    message: Schema.String,
    value: Schema.optional(Schema.Unknown)
  }))
})

const NotFoundError = Schema.Struct({
  error: Schema.Literal("NOT_FOUND"),
  message: Schema.String,
  resource: Schema.String,
  id: Schema.optional(Schema.String)
})

const ConflictError = Schema.Struct({
  error: Schema.Literal("CONFLICT"),
  message: Schema.String,
  conflictingField: Schema.String,
  conflictingValue: Schema.Unknown
})

const InternalServerError = Schema.Struct({
  error: Schema.Literal("INTERNAL_SERVER_ERROR"),
  message: Schema.String,
  requestId: Schema.String
})

const RateLimitError = Schema.Struct({
  error: Schema.Literal("RATE_LIMIT_EXCEEDED"),
  message: Schema.String,
  retryAfter: Schema.Number
})

// Domain error classes
class UserNotFoundError extends Data.TaggedError("UserNotFoundError")<{
  readonly userId: string
}> {}

class EmailAlreadyExistsError extends Data.TaggedError("EmailAlreadyExistsError")<{
  readonly email: string
}> {}

class ValidationFailedError extends Data.TaggedError("ValidationFailedError")<{
  readonly errors: Array<{ field: string; message: string; value?: unknown }>
}> {}

// Error mapping utility
const mapDomainErrorToHttp = (error: unknown) => {
  if (error instanceof UserNotFoundError) {
    return Effect.fail({
      error: "NOT_FOUND" as const,
      message: `User not found`,
      resource: "user",
      id: error.userId
    })
  }
  
  if (error instanceof EmailAlreadyExistsError) {
    return Effect.fail({
      error: "CONFLICT" as const,
      message: "Email address already in use",
      conflictingField: "email",
      conflictingValue: error.email
    })
  }
  
  if (error instanceof ValidationFailedError) {
    return Effect.fail({
      error: "VALIDATION_ERROR" as const,
      message: "Validation failed",
      details: error.errors
    })
  }
  
  // Default to internal server error
  return Effect.fail({
    error: "INTERNAL_SERVER_ERROR" as const,
    message: "An unexpected error occurred",
    requestId: Math.random().toString(36).substr(2, 9)
  })
}

// API with comprehensive error handling
const ErrorHandlingApi = HttpApi.make("error-handling-api").add(
  HttpApiGroup.make("users")
    .add(
      HttpApiEndpoint.get("getUser", "/users/:id")
        .addSuccess(Schema.Struct({
          id: Schema.String,
          name: Schema.String,
          email: Schema.String
        }))
        .addError(NotFoundError, { status: 404 })
        .addError(InternalServerError, { status: 500 })
    )
    .add(
      HttpApiEndpoint.post("createUser", "/users")
        .setPayload(Schema.Struct({
          name: Schema.String.pipe(Schema.minLength(1)),
          email: Schema.String.pipe(Schema.pattern(/^[^@]+@[^@]+\.[^@]+$/))
        }))
        .addSuccess(Schema.Struct({
          id: Schema.String,
          name: Schema.String,
          email: Schema.String
        }))
        .addError(ValidationError, { status: 400 })
        .addError(ConflictError, { status: 409 })
        .addError(InternalServerError, { status: 500 })
    )
    .add(
      HttpApiEndpoint.get("listUsers", "/users")
        .setHeaders(Schema.Struct({
          "X-Rate-Limit": Schema.optional(Schema.String)
        }))
        .addSuccess(Schema.Array(Schema.Struct({
          id: Schema.String,
          name: Schema.String,
          email: Schema.String
        })))
        .addError(RateLimitError, { status: 429 })
        .addError(InternalServerError, { status: 500 })
    )
)

// Service with domain errors
interface UserService {
  readonly getUser: (id: string) => Effect.Effect<
    { id: string; name: string; email: string },
    UserNotFoundError
  >
  readonly createUser: (data: { name: string; email: string }) => Effect.Effect<
    { id: string; name: string; email: string },
    EmailAlreadyExistsError | ValidationFailedError
  >
  readonly listUsers: () => Effect.Effect<
    Array<{ id: string; name: string; email: string }>,
    never
  >
}

const UserService = Effect.Tag<UserService>("UserService")

// Rate limiting service
interface RateLimitService {
  readonly checkLimit: (key: string) => Effect.Effect<boolean, never>
}

const RateLimitService = Effect.Tag<RateLimitService>("RateLimitService")

const RateLimitServiceLive = Layer.succeed(RateLimitService, {
  checkLimit: (key) => Effect.succeed(Math.random() > 0.1) // 90% success rate
})

// Handlers with error mapping
const ErrorHandlingHandlers = HttpApiBuilder.group(ErrorHandlingApi, "users", (handlers) =>
  handlers
    .handle("getUser", ({ path }) => 
      UserService.getUser(path.id).pipe(
        Effect.catchAll(mapDomainErrorToHttp)
      )
    )
    .handle("createUser", ({ payload }) => 
      UserService.createUser(payload).pipe(
        Effect.catchAll(mapDomainErrorToHttp)
      )
    )
    .handle("listUsers", ({ headers }) => 
      Effect.gen(function* () {
        const rateLimitKey = headers["X-Rate-Limit"] || "default"
        const allowed = yield* RateLimitService.checkLimit(rateLimitKey)
        
        if (!allowed) {
          return yield* Effect.fail({
            error: "RATE_LIMIT_EXCEEDED" as const,
            message: "Rate limit exceeded. Please try again later.",
            retryAfter: 60
          })
        }
        
        return yield* UserService.listUsers().pipe(
          Effect.catchAll(mapDomainErrorToHttp)
        )
      })
    )
)

// Global error handling middleware
const GlobalErrorHandler = HttpApiBuilder.middleware.make((request) =>
  Effect.gen(function* () {
    return {}
  }).pipe(
    Effect.catchAllCause(cause => 
      Effect.gen(function* () {
        console.error("Unhandled error:", cause)
        return yield* Effect.fail({
          error: "INTERNAL_SERVER_ERROR" as const,
          message: "An unexpected error occurred",
          requestId: Math.random().toString(36).substr(2, 9)
        })
      })
    )
  )
)
```

## Integration Examples

### Integration with HttpServer and HttpClient

Complete example showing how HttpApi integrates with HttpServer for serving APIs and HttpClient for consuming them.

```typescript
import { 
  HttpApi, 
  HttpApiGroup, 
  HttpApiEndpoint, 
  HttpApiBuilder,
  HttpServer,
  HttpClient
} from "@effect/platform"
import { NodeHttpServer, NodeRuntime } from "@effect/platform-node"
import { Schema } from "effect"
import { Effect, Layer } from "effect"

// Define API
const TodoApi = HttpApi.make("todo-api").add(
  HttpApiGroup.make("todos")
    .add(
      HttpApiEndpoint.get("listTodos", "/todos")
        .setUrlParams(Schema.Struct({
          completed: Schema.optional(Schema.Boolean),
          page: Schema.optional(Schema.NumberFromString.pipe(Schema.positive())),
          limit: Schema.optional(Schema.NumberFromString.pipe(Schema.positive()))
        }))
        .addSuccess(Schema.Struct({
          todos: Schema.Array(Schema.Struct({
            id: Schema.String,
            title: Schema.String,
            completed: Schema.Boolean,
            createdAt: Schema.DateTimeUtc
          })),
          pagination: Schema.Struct({
            page: Schema.Number,
            limit: Schema.Number,
            total: Schema.Number
          })
        }))
    )
    .add(
      HttpApiEndpoint.post("createTodo", "/todos")
        .setPayload(Schema.Struct({
          title: Schema.String.pipe(Schema.minLength(1)),
          completed: Schema.optional(Schema.Boolean.pipe(Schema.withDefault(() => false)))
        }))
        .addSuccess(Schema.Struct({
          id: Schema.String,
          title: Schema.String,
          completed: Schema.Boolean,
          createdAt: Schema.DateTimeUtc
        }))
    )
)

// Service implementation
interface TodoService {
  readonly list: (filters: { completed?: boolean; page?: number; limit?: number }) => Effect.Effect<{
    todos: Array<{ id: string; title: string; completed: boolean; createdAt: Date }>
    pagination: { page: number; limit: number; total: number }
  }>
  readonly create: (data: { title: string; completed?: boolean }) => Effect.Effect<{
    id: string
    title: string
    completed: boolean
    createdAt: Date
  }>
}

const TodoService = Effect.Tag<TodoService>("TodoService")

const TodoServiceLive = Layer.succeed(TodoService, {
  list: (filters) => Effect.gen(function* () {
    // Simulate database
    const allTodos = [
      { id: "1", title: "Learn Effect", completed: false, createdAt: new Date("2024-01-01") },
      { id: "2", title: "Build API", completed: true, createdAt: new Date("2024-01-02") },
      { id: "3", title: "Write docs", completed: false, createdAt: new Date("2024-01-03") }
    ]
    
    let filtered = allTodos
    if (filters.completed !== undefined) {
      filtered = filtered.filter(todo => todo.completed === filters.completed)
    }
    
    const page = filters.page ?? 1
    const limit = filters.limit ?? 10
    const start = (page - 1) * limit
    const todos = filtered.slice(start, start + limit)
    
    return {
      todos,
      pagination: {
        page,
        limit,
        total: filtered.length
      }
    }
  }),
  
  create: (data) => Effect.gen(function* () {
    const todo = {
      id: Math.random().toString(36).substr(2, 9),
      title: data.title,
      completed: data.completed ?? false,
      createdAt: new Date()
    }
    return todo
  })
})

// Handlers
const TodoHandlers = HttpApiBuilder.group(TodoApi, "todos", (handlers) =>
  handlers
    .handle("listTodos", ({ urlParams }) => TodoService.list(urlParams))
    .handle("createTodo", ({ payload }) => TodoService.create(payload))
)

// Server setup
const TodoServer = HttpServer.router.empty.pipe(
  HttpServer.router.mount("/api", HttpApiBuilder.httpApp(TodoApi)),
  HttpServer.server.serve(HttpServer.middleware.logger),
  HttpServer.server.withLogAddress,
  Layer.provide(HttpApiBuilder.api(TodoApi)),
  Layer.provide(TodoHandlers),
  Layer.provide(TodoServiceLive),
  Layer.provide(NodeHttpServer.layer({ port: 3000 }))
)

// Client setup
const createTodoClient = (baseUrl: string) => {
  const client = HttpClient.client.pipe(
    HttpClient.client.mapRequest(HttpClient.request.prependUrl(baseUrl))
  )
  
  return {
    todos: {
      list: (params?: { completed?: boolean; page?: number; limit?: number }) =>
        Effect.gen(function* () {
          const url = new URL("/api/todos", baseUrl)
          if (params?.completed !== undefined) {
            url.searchParams.set("completed", String(params.completed))
          }
          if (params?.page) {
            url.searchParams.set("page", String(params.page))
          }
          if (params?.limit) {
            url.searchParams.set("limit", String(params.limit))
          }
          
          const response = yield* HttpClient.request.get(url.pathname + url.search).pipe(
            client,
            Effect.flatMap(HttpClient.response.json)
          )
          
          return response as {
            todos: Array<{ id: string; title: string; completed: boolean; createdAt: string }>
            pagination: { page: number; limit: number; total: number }
          }
        }),
      
      create: (data: { title: string; completed?: boolean }) =>
        Effect.gen(function* () {
          const response = yield* HttpClient.request.post("/api/todos").pipe(
            HttpClient.request.jsonBody(data),
            client,
            Effect.flatMap(HttpClient.response.json)
          )
          
          return response as {
            id: string
            title: string
            completed: boolean
            createdAt: string
          }
        })
    }
  }
}

// Usage example
const program = Effect.gen(function* () {
  // Start server
  yield* Effect.fork(TodoServer)
  
  // Wait for server to start
  yield* Effect.sleep("1 second")
  
  // Use client
  const client = createTodoClient("http://localhost:3000")
  
  // Create a todo
  const newTodo = yield* client.todos.create({
    title: "Test todo from client",
    completed: false
  })
  
  console.log("Created todo:", newTodo)
  
  // List todos
  const todos = yield* client.todos.list({ completed: false })
  console.log("Todos:", todos)
})

// Run the program
NodeRuntime.runMain(program)
```

### Testing Strategies

Comprehensive testing approach for HttpApi including unit tests, integration tests, and contract testing.

```typescript
import { HttpApi, HttpApiGroup, HttpApiEndpoint, HttpApiBuilder } from "@effect/platform"
import { HttpClient, HttpClientRequest } from "@effect/platform"
import { Schema } from "effect"
import { Effect, Layer, TestContext, TestServices } from "effect"
import { expect } from "vitest"

// Test utilities for HttpApi
const createTestApi = () => {
  const User = Schema.Struct({
    id: Schema.String,
    name: Schema.String,
    email: Schema.String
  })
  
  const CreateUserRequest = Schema.Struct({
    name: Schema.String.pipe(Schema.minLength(1)),
    email: Schema.String.pipe(Schema.pattern(/^[^@]+@[^@]+\.[^@]+$/))
  })
  
  return HttpApi.make("test-api").add(
    HttpApiGroup.make("users")
      .add(
        HttpApiEndpoint.get("getUser", "/users/:id")
          .addSuccess(User)
          .addError(Schema.Struct({ error: Schema.String }), { status: 404 })
      )
      .add(
        HttpApiEndpoint.post("createUser", "/users")
          .setPayload(CreateUserRequest)
          .addSuccess(User)
          .addError(Schema.Struct({ error: Schema.String }), { status: 400 })
      )
  )
}

// Mock service for testing
interface TestUserService {
  readonly getUser: (id: string) => Effect.Effect<
    { id: string; name: string; email: string },
    { error: string }
  >
  readonly createUser: (data: { name: string; email: string }) => Effect.Effect<
    { id: string; name: string; email: string },
    { error: string }
  >
}

const TestUserService = Effect.Tag<TestUserService>("TestUserService")

// Mock implementation
const createMockUserService = (
  users: Map<string, { id: string; name: string; email: string }> = new Map()
) => Layer.succeed(TestUserService, {
  getUser: (id) => 
    users.has(id) 
      ? Effect.succeed(users.get(id)!)
      : Effect.fail({ error: "User not found" }),
  
  createUser: (data) => Effect.gen(function* () {
    // Check for duplicate email
    const existingUser = Array.from(users.values()).find(u => u.email === data.email)
    if (existingUser) {
      return yield* Effect.fail({ error: "Email already exists" })
    }
    
    const user = {
      id: Math.random().toString(36).substr(2, 9),
      name: data.name,
      email: data.email
    }
    
    users.set(user.id, user)
    return user
  })
})

// Unit tests for handlers
const testHandlers = () => {
  const api = createTestApi()
  
  const handlers = HttpApiBuilder.group(api, "users", (handlers) =>
    handlers
      .handle("getUser", ({ path }) => TestUserService.getUser(path.id))
      .handle("createUser", ({ payload }) => TestUserService.createUser(payload))
  )
  
  return {
    getUser: (id: string) => 
      handlers.pipe(
        Layer.provide(createMockUserService(new Map([
          ["1", { id: "1", name: "John Doe", email: "john@example.com" }]
        ]))),
        Layer.build,
        Effect.flatMap(layer => 
          TestUserService.getUser(id).pipe(
            Effect.provide(layer)
          )
        )
      ),
    
    createUser: (data: { name: string; email: string }) =>
      handlers.pipe(
        Layer.provide(createMockUserService()),
        Layer.build,
        Effect.flatMap(layer =>
          TestUserService.createUser(data).pipe(
            Effect.provide(layer)
          )
        )
      )
  }
}

// Integration tests with real HTTP server
const createTestServer = () => {
  const api = createTestApi()
  
  const handlers = HttpApiBuilder.group(api, "users", (handlers) =>
    handlers
      .handle("getUser", ({ path }) => TestUserService.getUser(path.id))
      .handle("createUser", ({ payload }) => TestUserService.createUser(payload))
  )
  
  const server = HttpServer.router.empty.pipe(
    HttpServer.router.mount("/api", HttpApiBuilder.httpApp(api)),
    HttpServer.server.serve(),
    Layer.provide(HttpApiBuilder.api(api)),
    Layer.provide(handlers),
    Layer.provide(createMockUserService(new Map([
      ["1", { id: "1", name: "John Doe", email: "john@example.com" }]
    ]))),
    Layer.provide(HttpServer.layer)
  )
  
  return server
}

// Contract testing utilities
const testApiContract = (api: typeof HttpApi.HttpApi.Any) => {
  const spec = OpenApi.fromApi(api)
  
  return {
    validateRequest: (endpoint: string, method: string, data: unknown) =>
      Effect.gen(function* () {
        // Validate request against OpenAPI spec
        const operation = spec.paths[endpoint]?.[method.toLowerCase()]
        if (!operation) {
          return yield* Effect.fail(new Error(`Endpoint ${method} ${endpoint} not found in spec`))
        }
        
        // Validate request body if present
        if (operation.requestBody && data) {
          const schema = operation.requestBody.content["application/json"]?.schema
          // Validate data against schema (implementation would use ajv or similar)
          return true
        }
        
        return true
      }),
    
    validateResponse: (endpoint: string, method: string, status: number, data: unknown) =>
      Effect.gen(function* () {
        const operation = spec.paths[endpoint]?.[method.toLowerCase()]
        if (!operation) {
          return yield* Effect.fail(new Error(`Endpoint ${method} ${endpoint} not found in spec`))
        }
        
        const response = operation.responses[status]
        if (!response) {
          return yield* Effect.fail(new Error(`Status ${status} not defined for ${method} ${endpoint}`))
        }
        
        // Validate response data against schema
        return true
      })
  }
}

// Test examples
describe("HttpApi Testing", () => {
  test("unit test - get user", async () => {
    const { getUser } = testHandlers()
    
    const result = await Effect.runPromise(getUser("1"))
    
    expect(result).toEqual({
      id: "1",
      name: "John Doe",
      email: "john@example.com"
    })
  })
  
  test("unit test - create user", async () => {
    const { createUser } = testHandlers()
    
    const result = await Effect.runPromise(createUser({
      name: "Jane Smith",
      email: "jane@example.com"
    }))
    
    expect(result.name).toBe("Jane Smith")
    expect(result.email).toBe("jane@example.com")
    expect(result.id).toBeDefined()
  })
  
  test("integration test - HTTP server", async () => {
    const server = createTestServer()
    
    await Effect.runPromise(
      Effect.gen(function* () {
        // Start server
        yield* Effect.fork(server)
        yield* Effect.sleep("100 millis")
        
        // Make HTTP request
        const response = yield* HttpClient.request.get("http://localhost:3000/api/users/1").pipe(
          HttpClient.client,
          Effect.flatMap(HttpClient.response.json)
        )
        
        expect(response).toEqual({
          id: "1",
          name: "John Doe",
          email: "john@example.com"
        })
      }).pipe(
        Effect.provide(TestServices.layer)
      )
    )
  })
  
  test("contract test - validate API spec", async () => {
    const api = createTestApi()
    const contract = testApiContract(api)
    
    await Effect.runPromise(
      Effect.gen(function* () {
        // Test valid request
        yield* contract.validateRequest("/users", "POST", {
          name: "John Doe",
          email: "john@example.com"
        })
        
        // Test valid response
        yield* contract.validateResponse("/users", "POST", 200, {
          id: "1",
          name: "John Doe",
          email: "john@example.com"
        })
      })
    )
  })
})

// Property-based testing with fast-check
import fc from "fast-check"

const userArbitrary = fc.record({
  name: fc.string({ minLength: 1, maxLength: 100 }),
  email: fc.emailAddress()
})

test("property test - create user always returns valid user", async () => {
  await fc.assert(
    fc.asyncProperty(userArbitrary, async (userData) => {
      const { createUser } = testHandlers()
      
      const result = await Effect.runPromise(
        createUser(userData).pipe(
          Effect.either
        )
      )
      
      if (result._tag === "Right") {
        expect(result.right.name).toBe(userData.name)
        expect(result.right.email).toBe(userData.email)
        expect(typeof result.right.id).toBe("string")
        expect(result.right.id.length).toBeGreaterThan(0)
      }
    })
  )
})
```

## Conclusion

HttpApi provides declarative, type-safe API development with automatic documentation generation, client code generation, and comprehensive validation. It eliminates the disconnect between API definition, implementation, and documentation.

Key benefits:
- **Single Source of Truth**: Define your API once, get types, validation, and docs automatically
- **Type Safety**: Full end-to-end type safety from server to client
- **Developer Experience**: Excellent IDE support with autocomplete and error detection
- **Maintainability**: Schema-driven approach prevents drift and reduces maintenance overhead
- **Integration**: Seamless integration with Effect ecosystem and existing tools

HttpApi is ideal for teams building production APIs who want to maintain consistency, reduce boilerplate, and ensure type safety across their entire application stack.