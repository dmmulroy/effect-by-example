# HttpServer: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem HttpServer Solves

Building HTTP servers in Node.js traditionally involves significant boilerplate, manual error handling, and type-unsafe patterns:

```typescript
// Traditional Express approach - verbose and error-prone
import express from 'express'
import bodyParser from 'body-parser'

const app = express()
app.use(bodyParser.json())

// Manual error handling everywhere
app.get('/users/:id', async (req, res) => {
  try {
    const userId = req.params.id
    // No type safety for params
    const user = await db.users.findById(userId)
    if (!user) {
      res.status(404).json({ error: 'User not found' })
      return
    }
    res.json(user)
  } catch (error) {
    console.error(error)
    res.status(500).json({ error: 'Internal server error' })
  }
})

// Middleware composition is implicit and order-dependent
app.use(cors())
app.use(authMiddleware)
app.use(rateLimiter)

// No built-in dependency injection
const db = createDatabase() // Global singleton
```

This approach leads to:
- **Error-prone code** - Manual try-catch blocks everywhere
- **Type unsafety** - Request/response types are not enforced
- **Poor composability** - Middleware order matters and is implicit
- **Testing difficulties** - Hard to test in isolation
- **No dependency injection** - Global singletons and tight coupling

### The HttpServer Solution

Effect's HttpServer provides a type-safe, composable, and testable foundation for building HTTP servers:

```typescript
import { HttpRouter, HttpServer, HttpServerResponse } from "@effect/platform"
import { Effect, Layer } from "effect"

// Type-safe routing with automatic error handling
const router = HttpRouter.empty.pipe(
  HttpRouter.get("/users/:id", 
    Effect.gen(function* () {
      const { id } = yield* HttpRouter.params
      const db = yield* Database
      const user = yield* db.users.findById(id)
      return yield* HttpServerResponse.json(user)
    }).pipe(
      Effect.catchTag("NotFoundError", () => 
        HttpServerResponse.empty({ status: 404 })
      )
    )
  )
)

// Composable middleware with dependency injection
const server = HttpServer.serve(HttpMiddleware.logger).pipe(
  Layer.provide(HttpServer.layer({ port: 3000 })),
  Layer.provide(Database.layer)
)
```

### Key Concepts

**HttpServer**: The core server abstraction that handles HTTP requests and responses with full type safety and Effect integration.

**HttpRouter**: A composable routing system that supports path parameters, type-safe handlers, and automatic error propagation.

**HttpMiddleware**: Reusable request/response transformers that can add logging, authentication, CORS, and other cross-cutting concerns.

**HttpServerRequest**: Type-safe request abstraction with built-in parsers for headers, cookies, body, and query parameters.

**HttpServerResponse**: Composable response builders with support for JSON, HTML, streams, and files.

## Basic Usage Patterns

### Creating a Basic Server

```typescript
import { HttpRouter, HttpServer, HttpServerResponse } from "@effect/platform"
import { NodeHttpServer, NodeRuntime } from "@effect/platform-node"
import { Effect, Layer } from "effect"

// Create a simple router
const router = HttpRouter.empty.pipe(
  HttpRouter.get("/", 
    HttpServerResponse.text("Hello, World!")
  ),
  HttpRouter.get("/health",
    HttpServerResponse.json({ status: "ok", timestamp: new Date().toISOString() })
  )
)

// Create and run the server
const ServerLive = HttpRouter.toHttpApp(router).pipe(
  HttpServer.serve(HttpMiddleware.logger),
  Layer.provide(NodeHttpServer.layer({ port: 3000 }))
)

// Run the server
NodeRuntime.runMain(Layer.launch(ServerLive))
```

### Working with Request Data

```typescript
import { HttpRouter, HttpServerRequest, HttpServerResponse } from "@effect/platform"
import { Effect, Schema } from "effect"

// Define request schemas
const CreateUserBody = Schema.Struct({
  name: Schema.String,
  email: Schema.String.pipe(Schema.pattern(/^[^\s@]+@[^\s@]+\.[^\s@]+$/)),
  age: Schema.Number.pipe(Schema.positive())
})

// Type-safe request handling
const router = HttpRouter.empty.pipe(
  HttpRouter.post("/users",
    Effect.gen(function* () {
      // Parse and validate request body
      const body = yield* HttpServerRequest.schemaBodyJson(CreateUserBody)
      
      // Access headers
      const request = yield* HttpServerRequest.HttpServerRequest
      const authHeader = request.headers["authorization"]
      
      // Access query parameters
      const searchParams = yield* HttpRouter.searchParams
      const limit = searchParams.get("limit") ?? "10"
      
      // Return typed response
      return yield* HttpServerResponse.json({
        user: body,
        limit: parseInt(limit)
      })
    })
  )
)
```

### Middleware Composition

```typescript
import { HttpMiddleware, HttpRouter, HttpServerResponse } from "@effect/platform"
import { Effect, Layer } from "effect"

// Custom authentication middleware
const authMiddleware = HttpMiddleware.make((app) =>
  Effect.gen(function* () {
    const request = yield* HttpServerRequest.HttpServerRequest
    const token = request.headers["authorization"]?.replace("Bearer ", "")
    
    if (!token) {
      return yield* HttpServerResponse.unauthorized()
    }
    
    const user = yield* AuthService.verifyToken(token)
    return yield* app.pipe(
      Effect.provideService(CurrentUser, user)
    )
  })
)

// Apply middleware to router
const router = HttpRouter.empty.pipe(
  HttpRouter.get("/public", HttpServerResponse.text("Public route")),
  HttpRouter.use(authMiddleware),
  HttpRouter.get("/private", 
    Effect.gen(function* () {
      const user = yield* CurrentUser
      return yield* HttpServerResponse.json({ message: `Hello ${user.name}` })
    })
  )
)
```

## Real-World Examples

### Example 1: REST API with CRUD Operations

Building a complete REST API for a blog system with posts and comments:

```typescript
import { HttpRouter, HttpServerRequest, HttpServerResponse, HttpMiddleware } from "@effect/platform"
import { NodeHttpServer, NodeRuntime } from "@effect/platform-node"
import { Effect, Layer, Schema, Array as Arr } from "effect"

// Domain models
const Post = Schema.Struct({
  id: Schema.String,
  title: Schema.String,
  content: Schema.String,
  authorId: Schema.String,
  createdAt: Schema.Date,
  updatedAt: Schema.Date
})

const CreatePostBody = Schema.Struct({
  title: Schema.String.pipe(Schema.minLength(1)),
  content: Schema.String.pipe(Schema.minLength(10))
})

const UpdatePostBody = Schema.partial(CreatePostBody)

// Database service
interface PostsDatabase {
  readonly findAll: (options?: { limit?: number; offset?: number }) => 
    Effect.Effect<readonly Post.Type[]>
  readonly findById: (id: string) => 
    Effect.Effect<Post.Type, NotFoundError>
  readonly create: (data: CreatePostBody.Type & { authorId: string }) => 
    Effect.Effect<Post.Type>
  readonly update: (id: string, data: UpdatePostBody.Type) => 
    Effect.Effect<Post.Type, NotFoundError>
  readonly delete: (id: string) => 
    Effect.Effect<void, NotFoundError>
}

const PostsDatabase = Context.GenericTag<PostsDatabase>("PostsDatabase")

// Error types
class NotFoundError extends Data.TaggedError("NotFoundError")<{
  readonly resource: string
  readonly id: string
}> {}

// Posts router
const postsRouter = HttpRouter.prefixPath("/posts").pipe(
  // GET /posts - List all posts
  HttpRouter.get("/",
    Effect.gen(function* () {
      const searchParams = yield* HttpRouter.searchParams
      const limit = parseInt(searchParams.get("limit") ?? "10")
      const offset = parseInt(searchParams.get("offset") ?? "0")
      
      const db = yield* PostsDatabase
      const posts = yield* db.findAll({ limit, offset })
      
      return yield* HttpServerResponse.json({
        data: posts,
        pagination: { limit, offset, total: posts.length }
      })
    })
  ),
  
  // GET /posts/:id - Get single post
  HttpRouter.get("/:id",
    Effect.gen(function* () {
      const { id } = yield* HttpRouter.params
      const db = yield* PostsDatabase
      const post = yield* db.findById(id)
      return yield* HttpServerResponse.json(post)
    }).pipe(
      Effect.catchTag("NotFoundError", () =>
        HttpServerResponse.empty({ status: 404 })
      )
    )
  ),
  
  // POST /posts - Create new post
  HttpRouter.post("/",
    Effect.gen(function* () {
      const body = yield* HttpServerRequest.schemaBodyJson(CreatePostBody)
      const user = yield* CurrentUser
      const db = yield* PostsDatabase
      
      const post = yield* db.create({
        ...body,
        authorId: user.id
      })
      
      return yield* HttpServerResponse.json(post, { status: 201 })
    })
  ),
  
  // PUT /posts/:id - Update post
  HttpRouter.put("/:id",
    Effect.gen(function* () {
      const { id } = yield* HttpRouter.params
      const body = yield* HttpServerRequest.schemaBodyJson(UpdatePostBody)
      const db = yield* PostsDatabase
      
      const post = yield* db.update(id, body)
      return yield* HttpServerResponse.json(post)
    }).pipe(
      Effect.catchTag("NotFoundError", () =>
        HttpServerResponse.empty({ status: 404 })
      )
    )
  ),
  
  // DELETE /posts/:id - Delete post
  HttpRouter.del("/:id",
    Effect.gen(function* () {
      const { id } = yield* HttpRouter.params
      const db = yield* PostsDatabase
      
      yield* db.delete(id)
      return yield* HttpServerResponse.empty({ status: 204 })
    }).pipe(
      Effect.catchTag("NotFoundError", () =>
        HttpServerResponse.empty({ status: 404 })
      )
    )
  )
)

// Main application router with middleware
const appRouter = HttpRouter.empty.pipe(
  HttpRouter.use(HttpMiddleware.cors({
    allowedOrigins: ["http://localhost:3001"],
    allowedMethods: ["GET", "POST", "PUT", "DELETE"],
    allowCredentials: true
  })),
  HttpRouter.use(HttpMiddleware.logger),
  HttpRouter.mount("/api", 
    HttpRouter.empty.pipe(
      HttpRouter.use(authMiddleware),
      HttpRouter.mount("/posts", postsRouter)
    )
  )
)

// Server setup
const ServerLive = HttpRouter.toHttpApp(appRouter).pipe(
  HttpServer.serve(),
  Layer.provide(NodeHttpServer.layer({ port: 3000 })),
  Layer.provide(PostsDatabase.layer),
  Layer.provide(AuthService.layer)
)

NodeRuntime.runMain(Layer.launch(ServerLive))
```

### Example 2: File Upload and Static File Serving

Handling file uploads and serving static files with proper content types:

```typescript
import { HttpRouter, HttpServerRequest, HttpServerResponse, HttpMiddleware } from "@effect/platform"
import { NodeHttpServer } from "@effect/platform-node"
import { Effect, Layer, Stream, Chunk } from "effect"
import * as FileSystem from "@effect/platform/FileSystem"
import * as Path from "@effect/platform/Path"

// File upload handler
const uploadRouter = HttpRouter.prefixPath("/upload").pipe(
  HttpRouter.post("/",
    Effect.gen(function* () {
      const request = yield* HttpServerRequest.HttpServerRequest
      const contentType = request.headers["content-type"] ?? ""
      
      if (!contentType.includes("multipart/form-data")) {
        return yield* HttpServerResponse.unsupportedMediaType()
      }
      
      // Parse multipart form data
      const formData = yield* HttpServerRequest.multipartPersisted
      const uploadedFiles: Array<{ filename: string; path: string }> = []
      
      // Process each file
      for (const [field, part] of formData) {
        if (part._tag === "File") {
          const fs = yield* FileSystem.FileSystem
          const uploadDir = yield* Path.Path.map(path => path.join("uploads"))
          
          // Ensure upload directory exists
          yield* fs.makeDirectory(uploadDir, { recursive: true })
          
          // Generate unique filename
          const timestamp = Date.now()
          const filename = `${timestamp}-${part.filename}`
          const filepath = yield* Path.Path.map(path => 
            path.join(uploadDir, filename)
          )
          
          // Copy file to uploads directory
          yield* fs.copy(part.path, filepath)
          uploadedFiles.push({ filename, path: filepath })
        }
      }
      
      return yield* HttpServerResponse.json({
        message: "Files uploaded successfully",
        files: uploadedFiles
      })
    })
  )
)

// Static file server with proper content types
const staticFileServer = (basePath: string) =>
  HttpRouter.get("/*",
    Effect.gen(function* () {
      const request = yield* HttpServerRequest.HttpServerRequest
      const fs = yield* FileSystem.FileSystem
      const pathApi = yield* Path.Path
      
      // Extract file path from URL
      const urlPath = new URL(request.url, `http://${request.headers.host}`).pathname
      const filePath = pathApi.join(basePath, urlPath.slice(1))
      
      // Security: Prevent directory traversal
      const normalizedPath = pathApi.normalize(filePath)
      if (!normalizedPath.startsWith(pathApi.normalize(basePath))) {
        return yield* HttpServerResponse.forbidden()
      }
      
      // Check if file exists
      const exists = yield* fs.exists(filePath)
      if (!exists) {
        return yield* HttpServerResponse.empty({ status: 404 })
      }
      
      // Get file stats
      const stats = yield* fs.stat(filePath)
      if (stats.type !== "File") {
        return yield* HttpServerResponse.empty({ status: 404 })
      }
      
      // Determine content type
      const ext = pathApi.extname(filePath).toLowerCase()
      const contentTypes: Record<string, string> = {
        ".html": "text/html",
        ".css": "text/css",
        ".js": "application/javascript",
        ".json": "application/json",
        ".png": "image/png",
        ".jpg": "image/jpeg",
        ".jpeg": "image/jpeg",
        ".gif": "image/gif",
        ".svg": "image/svg+xml",
        ".pdf": "application/pdf"
      }
      
      const contentType = contentTypes[ext] ?? "application/octet-stream"
      
      // Stream file contents
      const stream = fs.stream(filePath, { chunkSize: 65536 })
      
      return yield* HttpServerResponse.stream(stream, {
        contentType,
        headers: {
          "cache-control": "public, max-age=3600",
          "last-modified": new Date(stats.mtime).toUTCString()
        }
      })
    })
  )

// Complete file handling server
const fileServer = HttpRouter.empty.pipe(
  HttpRouter.mount("/upload", uploadRouter),
  HttpRouter.mount("/static", staticFileServer("./public")),
  HttpRouter.get("/", HttpServerResponse.html(`
    <html>
      <body>
        <h1>File Upload Example</h1>
        <form action="/upload" method="post" enctype="multipart/form-data">
          <input type="file" name="file" multiple required>
          <button type="submit">Upload</button>
        </form>
      </body>
    </html>
  `))
)
```

### Example 3: WebSocket Integration

Real-time chat application with WebSocket support:

```typescript
import { HttpRouter, HttpServerRequest, HttpServerResponse, HttpServer } from "@effect/platform"
import { NodeHttpServer } from "@effect/platform-node"
import { Effect, Layer, Stream, Hub, Ref, HashMap } from "effect"

// Message types
interface ChatMessage {
  readonly id: string
  readonly userId: string
  readonly username: string
  readonly content: string
  readonly timestamp: Date
}

interface WebSocketConnection {
  readonly userId: string
  readonly username: string
  readonly send: (message: string) => Effect.Effect<void>
}

// Chat room service
class ChatRoom extends Context.Tag("ChatRoom")<
  ChatRoom,
  {
    readonly connections: Ref.Ref<HashMap.HashMap<string, WebSocketConnection>>
    readonly messages: Hub.Hub<ChatMessage>
    readonly join: (connection: WebSocketConnection) => Effect.Effect<void>
    readonly leave: (userId: string) => Effect.Effect<void>
    readonly broadcast: (message: ChatMessage) => Effect.Effect<void>
  }
>() {}

const ChatRoomLive = Layer.effect(
  ChatRoom,
  Effect.gen(function* () {
    const connections = yield* Ref.make(HashMap.empty<string, WebSocketConnection>())
    const messages = yield* Hub.unbounded<ChatMessage>()
    
    const join = (connection: WebSocketConnection) =>
      Effect.gen(function* () {
        yield* Ref.update(connections, HashMap.set(connection.userId, connection))
        
        // Send recent message history
        const recentMessages = yield* Hub.subscribe(messages).pipe(
          Stream.take(50),
          Stream.runCollect
        )
        
        yield* connection.send(JSON.stringify({
          type: "history",
          messages: recentMessages
        }))
      })
    
    const leave = (userId: string) =>
      Ref.update(connections, HashMap.remove(userId))
    
    const broadcast = (message: ChatMessage) =>
      Effect.gen(function* () {
        yield* Hub.publish(messages, message)
        
        const currentConnections = yield* Ref.get(connections)
        
        yield* Effect.forEach(
          HashMap.values(currentConnections),
          (connection) => connection.send(JSON.stringify({
            type: "message",
            data: message
          })),
          { concurrency: "unbounded", discard: true }
        )
      })
    
    return { connections, messages, join, leave, broadcast }
  })
)

// WebSocket upgrade handler
const websocketRouter = HttpRouter.get("/ws",
  Effect.gen(function* () {
    const request = yield* HttpServerRequest.HttpServerRequest
    const searchParams = yield* HttpRouter.searchParams
    
    const userId = searchParams.get("userId")
    const username = searchParams.get("username")
    
    if (!userId || !username) {
      return yield* HttpServerResponse.badRequest()
    }
    
    const chatRoom = yield* ChatRoom
    
    // Upgrade to WebSocket
    const socket = yield* HttpServerRequest.upgrade({
      onOpen: (ws) => 
        Effect.gen(function* () {
          const connection: WebSocketConnection = {
            userId,
            username,
            send: (message) => Effect.sync(() => ws.send(message))
          }
          
          yield* chatRoom.join(connection)
          yield* Effect.log(`User ${username} joined the chat`)
        }),
        
      onMessage: (ws, message) =>
        Effect.gen(function* () {
          const data = JSON.parse(message.toString())
          
          if (data.type === "message") {
            const chatMessage: ChatMessage = {
              id: crypto.randomUUID(),
              userId,
              username,
              content: data.content,
              timestamp: new Date()
            }
            
            yield* chatRoom.broadcast(chatMessage)
          }
        }),
        
      onClose: (ws) =>
        Effect.gen(function* () {
          yield* chatRoom.leave(userId)
          yield* Effect.log(`User ${username} left the chat`)
        })
    })
    
    return socket
  })
)

// HTTP endpoints for chat
const chatRouter = HttpRouter.empty.pipe(
  HttpRouter.get("/", HttpServerResponse.html(`
    <html>
      <head>
        <title>Effect Chat</title>
      </head>
      <body>
        <h1>Real-time Chat</h1>
        <div id="messages"></div>
        <input type="text" id="messageInput" placeholder="Type a message...">
        <button onclick="sendMessage()">Send</button>
        
        <script>
          const userId = crypto.randomUUID()
          const username = prompt("Enter your username:")
          const ws = new WebSocket(\`ws://localhost:3000/ws?userId=\${userId}&username=\${username}\`)
          
          ws.onmessage = (event) => {
            const data = JSON.parse(event.data)
            if (data.type === "message") {
              displayMessage(data.data)
            }
          }
          
          function sendMessage() {
            const input = document.getElementById("messageInput")
            ws.send(JSON.stringify({
              type: "message",
              content: input.value
            }))
            input.value = ""
          }
          
          function displayMessage(message) {
            const messages = document.getElementById("messages")
            messages.innerHTML += \`<p><b>\${message.username}:</b> \${message.content}</p>\`
          }
        </script>
      </body>
    </html>
  `)),
  HttpRouter.mount("/ws", websocketRouter)
)

// Run the chat server
const ChatServerLive = HttpRouter.toHttpApp(chatRouter).pipe(
  HttpServer.serve(HttpMiddleware.logger),
  Layer.provide(NodeHttpServer.layer({ port: 3000 })),
  Layer.provide(ChatRoomLive)
)

NodeRuntime.runMain(Layer.launch(ChatServerLive))
```

## Advanced Features Deep Dive

### Authentication and Authorization

Implementing JWT-based authentication with role-based access control:

```typescript
import { HttpMiddleware, HttpRouter, HttpServerRequest, HttpServerResponse } from "@effect/platform"
import { Effect, Context, Layer, Schema, Either } from "effect"
import * as JWT from "@effect/experimental/Jwt"

// User roles and permissions
const UserRole = Schema.Literal("admin", "user", "guest")
type UserRole = Schema.Schema.Type<typeof UserRole>

const User = Schema.Struct({
  id: Schema.String,
  email: Schema.String,
  role: UserRole
})
type User = Schema.Schema.Type<typeof User>

// Current user context
class CurrentUser extends Context.Tag("CurrentUser")<CurrentUser, User>() {}

// JWT service
class JwtService extends Context.Tag("JwtService")<
  JwtService,
  {
    readonly sign: (user: User) => Effect.Effect<string>
    readonly verify: (token: string) => Effect.Effect<User, UnauthorizedError>
  }
>() {}

class UnauthorizedError extends Data.TaggedError("UnauthorizedError")<{
  readonly reason: string
}> {}

// JWT service implementation
const JwtServiceLive = Layer.effect(
  JwtService,
  Effect.gen(function* () {
    const secret = yield* Config.string("JWT_SECRET")
    
    const sign = (user: User) =>
      JWT.encode({
        payload: user,
        secret,
        options: { expiresIn: "24h" }
      })
    
    const verify = (token: string) =>
      Effect.gen(function* () {
        const result = yield* JWT.verify(token, secret).pipe(
          Effect.mapError(() => new UnauthorizedError({ reason: "Invalid token" }))
        )
        
        return yield* Schema.decodeUnknown(User)(result.payload).pipe(
          Effect.mapError(() => new UnauthorizedError({ reason: "Invalid token payload" }))
        )
      })
    
    return { sign, verify }
  })
)

// Authentication middleware
const authenticate = HttpMiddleware.make((app) =>
  Effect.gen(function* () {
    const request = yield* HttpServerRequest.HttpServerRequest
    const authHeader = request.headers["authorization"]
    
    if (!authHeader?.startsWith("Bearer ")) {
      return yield* HttpServerResponse.unauthorized()
    }
    
    const token = authHeader.slice(7)
    const jwt = yield* JwtService
    const user = yield* jwt.verify(token).pipe(
      Effect.catchTag("UnauthorizedError", () =>
        Effect.fail(HttpServerResponse.unauthorized())
      )
    )
    
    return yield* app.pipe(
      Effect.provideService(CurrentUser, user)
    )
  })
)

// Authorization middleware factory
const authorize = (...allowedRoles: readonly UserRole[]) =>
  HttpMiddleware.make((app) =>
    Effect.gen(function* () {
      const user = yield* CurrentUser
      
      if (!allowedRoles.includes(user.role)) {
        return yield* HttpServerResponse.forbidden()
      }
      
      return yield* app
    })
  )

// Login endpoint
const authRouter = HttpRouter.empty.pipe(
  HttpRouter.post("/login",
    Effect.gen(function* () {
      const LoginBody = Schema.Struct({
        email: Schema.String.pipe(Schema.pattern(/^[^\s@]+@[^\s@]+\.[^\s@]+$/)),
        password: Schema.String.pipe(Schema.minLength(8))
      })
      
      const body = yield* HttpServerRequest.schemaBodyJson(LoginBody)
      const userService = yield* UserService
      
      // Verify credentials
      const user = yield* userService.verifyCredentials(body.email, body.password).pipe(
        Effect.catchTag("InvalidCredentials", () =>
          Effect.fail(HttpServerResponse.unauthorized())
        )
      )
      
      // Generate token
      const jwt = yield* JwtService
      const token = yield* jwt.sign(user)
      
      return yield* HttpServerResponse.json({
        token,
        user: {
          id: user.id,
          email: user.email,
          role: user.role
        }
      })
    })
  ),
  
  // Protected routes
  HttpRouter.use(authenticate),
  
  HttpRouter.get("/profile",
    Effect.gen(function* () {
      const user = yield* CurrentUser
      return yield* HttpServerResponse.json(user)
    })
  ),
  
  // Admin-only route
  HttpRouter.use(authorize("admin")),
  
  HttpRouter.get("/admin/users",
    Effect.gen(function* () {
      const userService = yield* UserService
      const users = yield* userService.listAll()
      return yield* HttpServerResponse.json(users)
    })
  )
)
```

### Rate Limiting and Request Validation

Implementing rate limiting with Redis and comprehensive request validation:

```typescript
import { HttpMiddleware, HttpRouter, HttpServerRequest, HttpServerResponse } from "@effect/platform"
import { Effect, RateLimiter, Duration, Schema } from "effect"
import * as Redis from "@effect/experimental/Redis"

// Rate limiter service
class RateLimiterService extends Context.Tag("RateLimiterService")<
  RateLimiterService,
  {
    readonly check: (key: string, limit: number, window: Duration.Duration) => 
      Effect.Effect<void, RateLimitExceeded>
  }
>() {}

class RateLimitExceeded extends Data.TaggedError("RateLimitExceeded")<{
  readonly limit: number
  readonly window: Duration.Duration
  readonly retryAfter: number
}> {}

// Redis-based rate limiter
const RateLimiterServiceLive = Layer.effect(
  RateLimiterService,
  Effect.gen(function* () {
    const redis = yield* Redis.Redis
    
    const check = (key: string, limit: number, window: Duration.Duration) =>
      Effect.gen(function* () {
        const windowMs = Duration.toMillis(window)
        const now = Date.now()
        const windowStart = now - windowMs
        
        // Remove old entries
        yield* redis.zRemRangeByScore(key, "-inf", windowStart.toString())
        
        // Count requests in window
        const count = yield* redis.zCard(key)
        
        if (count >= limit) {
          const oldestEntry = yield* redis.zRange(key, 0, 0)
          const oldestTime = oldestEntry[0] ? parseInt(oldestEntry[0]) : now
          const retryAfter = Math.ceil((oldestTime + windowMs - now) / 1000)
          
          return yield* Effect.fail(new RateLimitExceeded({
            limit,
            window,
            retryAfter
          }))
        }
        
        // Add current request
        yield* redis.zAdd(key, { score: now, member: now.toString() })
        yield* redis.expire(key, Math.ceil(windowMs / 1000))
      })
    
    return { check }
  })
)

// Rate limiting middleware factory
const rateLimitByIP = (limit: number, window: Duration.Duration) =>
  HttpMiddleware.make((app) =>
    Effect.gen(function* () {
      const request = yield* HttpServerRequest.HttpServerRequest
      const ip = request.headers["x-forwarded-for"] ?? 
                 request.socket.remoteAddress ?? 
                 "unknown"
      
      const rateLimiter = yield* RateLimiterService
      
      yield* rateLimiter.check(`rate_limit:${ip}`, limit, window).pipe(
        Effect.catchTag("RateLimitExceeded", (error) =>
          Effect.fail(
            HttpServerResponse.tooManyRequests({
              headers: {
                "retry-after": error.retryAfter.toString(),
                "x-ratelimit-limit": limit.toString(),
                "x-ratelimit-window": Duration.toSeconds(window).toString()
              }
            })
          )
        )
      )
      
      return yield* app
    })
  )

// Request validation schemas
const PaginationParams = Schema.Struct({
  page: Schema.optional(
    Schema.NumberFromString.pipe(
      Schema.positive(),
      Schema.int()
    ),
    { default: () => 1 }
  ),
  pageSize: Schema.optional(
    Schema.NumberFromString.pipe(
      Schema.positive(),
      Schema.int(),
      Schema.clamp(1, 100)
    ),
    { default: () => 20 }
  )
})

const SearchParams = Schema.extend(
  PaginationParams,
  Schema.Struct({
    q: Schema.optional(Schema.String.pipe(Schema.minLength(1))),
    sortBy: Schema.optional(Schema.Literal("name", "date", "popularity")),
    order: Schema.optional(Schema.Literal("asc", "desc"), { default: () => "asc" })
  })
)

// API with rate limiting and validation
const apiRouter = HttpRouter.empty.pipe(
  // Apply rate limiting
  HttpRouter.use(rateLimitByIP(100, Duration.minutes(1))),
  
  // Search endpoint with validation
  HttpRouter.get("/search",
    Effect.gen(function* () {
      const params = yield* HttpRouter.searchParams.pipe(
        Effect.flatMap(Schema.decodeUnknown(SearchParams))
      )
      
      const searchService = yield* SearchService
      const results = yield* searchService.search({
        query: params.q,
        page: params.page,
        pageSize: params.pageSize,
        sortBy: params.sortBy,
        order: params.order
      })
      
      return yield* HttpServerResponse.json({
        data: results.items,
        pagination: {
          page: params.page,
          pageSize: params.pageSize,
          total: results.total,
          totalPages: Math.ceil(results.total / params.pageSize)
        }
      })
    }).pipe(
      Effect.catchTag("ParseError", (error) =>
        HttpServerResponse.badRequest({
          body: JSON.stringify({ 
            error: "Invalid parameters",
            details: error.message 
          })
        })
      )
    )
  )
)
```

### CORS and Security Headers

Implementing comprehensive security headers and CORS configuration:

```typescript
import { HttpMiddleware, HttpRouter, HttpServerResponse } from "@effect/platform"
import { Effect, ReadonlyArray } from "effect"

// Security headers middleware
const securityHeaders = HttpMiddleware.make((app) =>
  app.pipe(
    Effect.map(response =>
      HttpServerResponse.setHeaders(response, {
        "x-content-type-options": "nosniff",
        "x-frame-options": "DENY",
        "x-xss-protection": "1; mode=block",
        "strict-transport-security": "max-age=31536000; includeSubDomains",
        "content-security-policy": "default-src 'self'",
        "referrer-policy": "strict-origin-when-cross-origin",
        "permissions-policy": "geolocation=(), microphone=(), camera=()"
      })
    )
  )
)

// Advanced CORS configuration
const corsConfig = {
  allowedOrigins: (origin: string) => {
    const allowedPatterns = [
      /^https:\/\/app\.example\.com$/,
      /^https:\/\/.*\.example\.com$/,
      /^http:\/\/localhost:\d+$/
    ]
    
    return allowedPatterns.some(pattern => pattern.test(origin))
  },
  
  allowedMethods: ["GET", "POST", "PUT", "DELETE", "PATCH"],
  
  allowedHeaders: [
    "content-type",
    "authorization",
    "x-request-id",
    "x-api-key"
  ],
  
  exposedHeaders: [
    "x-total-count",
    "x-page-count",
    "x-rate-limit-remaining"
  ],
  
  maxAge: 86400, // 24 hours
  
  credentials: true
}

// Dynamic CORS middleware
const dynamicCors = HttpMiddleware.make((app) =>
  Effect.gen(function* () {
    const request = yield* HttpServerRequest.HttpServerRequest
    const origin = request.headers["origin"]
    
    // Handle preflight requests
    if (request.method === "OPTIONS") {
      const headers: Record<string, string> = {
        "access-control-allow-methods": corsConfig.allowedMethods.join(", "),
        "access-control-allow-headers": corsConfig.allowedHeaders.join(", "),
        "access-control-max-age": corsConfig.maxAge.toString()
      }
      
      if (origin && corsConfig.allowedOrigins(origin)) {
        headers["access-control-allow-origin"] = origin
        headers["access-control-allow-credentials"] = "true"
      }
      
      return yield* HttpServerResponse.empty({
        status: 204,
        headers
      })
    }
    
    // Handle actual requests
    const response = yield* app
    
    if (origin && corsConfig.allowedOrigins(origin)) {
      return yield* pipe(
        response,
        HttpServerResponse.setHeaders({
          "access-control-allow-origin": origin,
          "access-control-allow-credentials": "true",
          "access-control-expose-headers": corsConfig.exposedHeaders.join(", ")
        })
      )
    }
    
    return response
  })
)

// Content Security Policy builder
const buildCSP = (directives: Record<string, readonly string[]>) =>
  Object.entries(directives)
    .map(([key, values]) => `${key} ${values.join(" ")}`)
    .join("; ")

// API with comprehensive security
const secureRouter = HttpRouter.empty.pipe(
  HttpRouter.use(securityHeaders),
  HttpRouter.use(dynamicCors),
  
  // Health check endpoint
  HttpRouter.get("/health",
    HttpServerResponse.json({ status: "ok" })
  ),
  
  // API routes with specific CSP
  HttpRouter.mount("/api",
    HttpRouter.empty.pipe(
      HttpRouter.use(
        HttpMiddleware.make((app) =>
          app.pipe(
            Effect.map(response =>
              HttpServerResponse.setHeader(
                response,
                "content-security-policy",
                buildCSP({
                  "default-src": ["'self'"],
                  "script-src": ["'self'", "'unsafe-inline'"],
                  "style-src": ["'self'", "'unsafe-inline'"],
                  "img-src": ["'self'", "data:", "https:"],
                  "connect-src": ["'self'", "https://api.example.com"]
                })
              )
            )
          )
        )
      )
    )
  )
)
```

## Practical Patterns & Best Practices

### Pattern 1: Error Handling and Recovery

Creating a comprehensive error handling system with proper error responses:

```typescript
import { HttpRouter, HttpServerResponse, HttpMiddleware } from "@effect/platform"
import { Effect, Match, Data } from "effect"

// Define application errors
class ValidationError extends Data.TaggedError("ValidationError")<{
  readonly field: string
  readonly message: string
}> {}

class BusinessError extends Data.TaggedError("BusinessError")<{
  readonly code: string
  readonly message: string
  readonly details?: unknown
}> {}

class NotFoundError extends Data.TaggedError("NotFoundError")<{
  readonly resource: string
  readonly id: string
}> {}

class ConflictError extends Data.TaggedError("ConflictError")<{
  readonly resource: string
  readonly message: string
}> {}

// Error response builder
const errorResponse = (error: unknown) =>
  Match.value(error).pipe(
    Match.tag("ValidationError", (e) =>
      HttpServerResponse.badRequest({
        headers: { "content-type": "application/json" },
        body: JSON.stringify({
          error: "Validation Error",
          field: e.field,
          message: e.message
        })
      })
    ),
    Match.tag("NotFoundError", (e) =>
      HttpServerResponse.notFound({
        headers: { "content-type": "application/json" },
        body: JSON.stringify({
          error: "Not Found",
          message: `${e.resource} with id ${e.id} not found`
        })
      })
    ),
    Match.tag("ConflictError", (e) =>
      HttpServerResponse.conflict({
        headers: { "content-type": "application/json" },
        body: JSON.stringify({
          error: "Conflict",
          resource: e.resource,
          message: e.message
        })
      })
    ),
    Match.tag("BusinessError", (e) =>
      HttpServerResponse.unprocessableEntity({
        headers: { "content-type": "application/json" },
        body: JSON.stringify({
          error: "Business Rule Violation",
          code: e.code,
          message: e.message,
          details: e.details
        })
      })
    ),
    Match.orElse(() =>
      HttpServerResponse.internalServerError({
        headers: { "content-type": "application/json" },
        body: JSON.stringify({
          error: "Internal Server Error",
          message: "An unexpected error occurred"
        })
      })
    )
  )

// Global error handling middleware
const errorHandler = HttpMiddleware.make((app) =>
  app.pipe(
    Effect.catchAll((error) =>
      Effect.gen(function* () {
        // Log the error
        yield* Effect.logError("Request failed", error)
        
        // Generate appropriate response
        return yield* errorResponse(error)
      })
    )
  )
)

// Helper for resource operations
const withResourceError = <R, E, A>(
  resource: string,
  id: string,
  effect: Effect.Effect<A, E, R>
) =>
  effect.pipe(
    Effect.mapError((error) => {
      if (error instanceof NotFoundError) {
        return error
      }
      return new NotFoundError({ resource, id })
    })
  )

// Usage example
const userRouter = HttpRouter.empty.pipe(
  HttpRouter.use(errorHandler),
  
  HttpRouter.get("/users/:id",
    Effect.gen(function* () {
      const { id } = yield* HttpRouter.params
      const userService = yield* UserService
      
      const user = yield* withResourceError(
        "User",
        id,
        userService.findById(id)
      )
      
      return yield* HttpServerResponse.json(user)
    })
  ),
  
  HttpRouter.post("/users",
    Effect.gen(function* () {
      const body = yield* HttpServerRequest.schemaBodyJson(CreateUserSchema).pipe(
        Effect.mapError((error) => 
          new ValidationError({
            field: "body",
            message: TreeFormatter.formatErrorSync(error)
          })
        )
      )
      
      const userService = yield* UserService
      
      const user = yield* userService.create(body).pipe(
        Effect.catchTag("DuplicateEmail", () =>
          Effect.fail(new ConflictError({
            resource: "User",
            message: "Email already exists"
          }))
        ),
        Effect.catchTag("InvalidAge", () =>
          Effect.fail(new BusinessError({
            code: "INVALID_AGE",
            message: "User must be at least 18 years old"
          }))
        )
      )
      
      return yield* HttpServerResponse.json(user, { status: 201 })
    })
  )
)
```

### Pattern 2: Request Context and Tracing

Implementing request context propagation and distributed tracing:

```typescript
import { HttpMiddleware, HttpRouter, HttpServerRequest } from "@effect/platform"
import { Effect, Context, Layer, FiberRef, Random } from "effect"
import * as Tracer from "@effect/opentelemetry"

// Request context
interface RequestContext {
  readonly requestId: string
  readonly userId?: string
  readonly correlationId?: string
  readonly startTime: number
}

class RequestContextService extends Context.Tag("RequestContext")<
  RequestContext,
  RequestContext
>() {}

// Request ID fiber ref
const requestIdRef = FiberRef.unsafeMake<string>("")

// Request context middleware
const requestContext = HttpMiddleware.make((app) =>
  Effect.gen(function* () {
    const request = yield* HttpServerRequest.HttpServerRequest
    const requestId = request.headers["x-request-id"] ?? 
                     (yield* Random.nextIntBetween(100000, 999999).pipe(
                       Effect.map(n => n.toString(36))
                     ))
    
    const context: RequestContext = {
      requestId,
      userId: request.headers["x-user-id"],
      correlationId: request.headers["x-correlation-id"],
      startTime: Date.now()
    }
    
    return yield* app.pipe(
      Effect.provideService(RequestContextService, context),
      FiberRef.locally(requestIdRef, requestId),
      Effect.tap(() => {
        const duration = Date.now() - context.startTime
        return Effect.log(`Request ${requestId} completed in ${duration}ms`)
      }),
      Effect.map(response =>
        HttpServerResponse.setHeaders(response, {
          "x-request-id": requestId,
          "x-response-time": `${Date.now() - context.startTime}ms`
        })
      )
    )
  })
)

// Structured logging with context
const log = {
  info: (message: string, data?: Record<string, unknown>) =>
    Effect.gen(function* () {
      const requestId = yield* FiberRef.get(requestIdRef)
      const context = yield* Effect.serviceOption(RequestContextService)
      
      yield* Effect.log({
        level: "INFO",
        message,
        requestId,
        userId: context.pipe(
          Option.map(c => c.userId),
          Option.getOrNull
        ),
        ...data
      })
    }),
    
  error: (message: string, error: unknown, data?: Record<string, unknown>) =>
    Effect.gen(function* () {
      const requestId = yield* FiberRef.get(requestIdRef)
      
      yield* Effect.logError({
        message,
        requestId,
        error: error instanceof Error ? {
          name: error.name,
          message: error.message,
          stack: error.stack
        } : error,
        ...data
      })
    })
}

// Distributed tracing setup
const tracingMiddleware = HttpMiddleware.make((app) =>
  Effect.gen(function* () {
    const request = yield* HttpServerRequest.HttpServerRequest
    const method = request.method
    const path = new URL(request.url, `http://${request.headers.host}`).pathname
    
    return yield* app.pipe(
      Effect.withSpan(`${method} ${path}`, {
        attributes: {
          "http.method": method,
          "http.url": request.url,
          "http.target": path,
          "http.host": request.headers.host ?? "",
          "http.scheme": "http",
          "http.user_agent": request.headers["user-agent"] ?? ""
        }
      }),
      Effect.tap((response) =>
        Effect.annotateCurrentSpan({
          "http.status_code": response.status
        })
      )
    )
  })
)

// Example usage with context
const contextualRouter = HttpRouter.empty.pipe(
  HttpRouter.use(requestContext),
  HttpRouter.use(tracingMiddleware),
  
  HttpRouter.post("/orders",
    Effect.gen(function* () {
      yield* log.info("Creating new order")
      
      const body = yield* HttpServerRequest.schemaBodyJson(CreateOrderSchema)
      const context = yield* RequestContextService
      
      yield* log.info("Order data validated", { 
        items: body.items.length,
        totalAmount: body.totalAmount 
      })
      
      const orderService = yield* OrderService
      
      const order = yield* orderService.create({
        ...body,
        userId: context.userId ?? "anonymous"
      }).pipe(
        Effect.tap((order) => 
          log.info("Order created successfully", { orderId: order.id })
        ),
        Effect.tapError((error) =>
          log.error("Failed to create order", error)
        )
      )
      
      return yield* HttpServerResponse.json(order, { status: 201 })
    })
  )
)
```

## Integration Examples

### Integration with Express

Using Effect HttpServer with existing Express applications:

```typescript
import express from "express"
import { HttpRouter, HttpServerResponse, HttpApp } from "@effect/platform"
import { NodeHttpServer } from "@effect/platform-node"
import { Effect, Layer, Runtime } from "effect"

// Effect router
const effectRouter = HttpRouter.empty.pipe(
  HttpRouter.get("/api/users/:id",
    Effect.gen(function* () {
      const { id } = yield* HttpRouter.params
      const userService = yield* UserService
      const user = yield* userService.findById(id)
      return yield* HttpServerResponse.json(user)
    })
  ),
  HttpRouter.post("/api/users",
    Effect.gen(function* () {
      const body = yield* HttpServerRequest.schemaBodyJson(CreateUserSchema)
      const userService = yield* UserService
      const user = yield* userService.create(body)
      return yield* HttpServerResponse.json(user, { status: 201 })
    })
  )
)

// Convert to HttpApp
const effectApp = HttpRouter.toHttpApp(effectRouter)

// Create runtime with services
const runtime = Runtime.defaultRuntime.pipe(
  Runtime.provideLayer(Layer.mergeAll(
    UserService.layer,
    Database.layer
  ))
)

// Express adapter
const effectMiddleware = (app: HttpApp.HttpApp<any, any>) => {
  return async (req: express.Request, res: express.Response, next: express.NextFunction) => {
    try {
      // Convert Express request to Effect request
      const headers: Record<string, string> = {}
      for (const [key, value] of Object.entries(req.headers)) {
        if (typeof value === "string") {
          headers[key] = value
        }
      }
      
      const request = {
        method: req.method,
        url: req.url,
        headers,
        body: req.body ? Stream.fromIterable([Buffer.from(JSON.stringify(req.body))]) : Stream.empty
      }
      
      // Run Effect app
      const response = await Effect.runPromise(
        app(request).pipe(
          Runtime.provideRuntime(runtime)
        )
      )
      
      // Convert Effect response to Express response
      res.status(response.status)
      
      for (const [key, value] of Object.entries(response.headers)) {
        res.setHeader(key, value)
      }
      
      if (response.body._tag === "Stream") {
        await Effect.runPromise(
          Stream.runForEach(response.body.stream, (chunk) =>
            Effect.sync(() => res.write(chunk))
          ).pipe(Runtime.provideRuntime(runtime))
        )
        res.end()
      } else {
        res.end()
      }
    } catch (error) {
      next(error)
    }
  }
}

// Express app with Effect integration
const app = express()
app.use(express.json())

// Regular Express routes
app.get("/", (req, res) => {
  res.send("Express + Effect Server")
})

// Mount Effect routes
app.use("/effect", effectMiddleware(effectApp))

// Start server
app.listen(3000, () => {
  console.log("Server running on http://localhost:3000")
})
```

### Testing Strategies

Comprehensive testing patterns for Effect HTTP servers:

```typescript
import { HttpClient, HttpClientRequest, HttpRouter } from "@effect/platform"
import { NodeTesting } from "@effect/platform-node"
import { Effect, Layer, TestContext, TestClock } from "effect"
import { it, describe, expect } from "@effect/vitest"

// Test utilities
const TestUserService = Layer.succeed(
  UserService,
  {
    findById: (id: string) =>
      id === "123" 
        ? Effect.succeed({ id, name: "Test User", email: "test@example.com" })
        : Effect.fail(new NotFoundError({ resource: "User", id })),
        
    create: (data: CreateUserInput) =>
      Effect.succeed({ 
        id: "generated-id",
        ...data,
        createdAt: new Date()
      }),
      
    listAll: () =>
      Effect.succeed([
        { id: "1", name: "User 1", email: "user1@example.com" },
        { id: "2", name: "User 2", email: "user2@example.com" }
      ])
  }
)

// Test helper for making requests
const testRequest = <E, A>(
  router: HttpRouter.HttpRouter<E, A>,
  request: HttpClientRequest.HttpClientRequest
) =>
  Effect.gen(function* () {
    const app = HttpRouter.toHttpApp(router)
    const response = yield* NodeTesting.makeHandler(app)(request)
    return response
  })

describe("UserRouter", () => {
  const userRouter = makeUserRouter()
  
  it("should get user by id", () =>
    Effect.gen(function* () {
      const response = yield* testRequest(
        userRouter,
        HttpClientRequest.get("/users/123")
      )
      
      expect(response.status).toBe(200)
      
      const body = yield* response.json
      expect(body).toEqual({
        id: "123",
        name: "Test User",
        email: "test@example.com"
      })
    }).pipe(
      Effect.provide(TestUserService),
      Effect.runPromise
    )
  )
  
  it("should return 404 for non-existent user", () =>
    Effect.gen(function* () {
      const response = yield* testRequest(
        userRouter,
        HttpClientRequest.get("/users/999")
      )
      
      expect(response.status).toBe(404)
    }).pipe(
      Effect.provide(TestUserService),
      Effect.runPromise
    )
  )
  
  it("should create new user", () =>
    Effect.gen(function* () {
      const response = yield* testRequest(
        userRouter,
        HttpClientRequest.post("/users").pipe(
          HttpClientRequest.jsonBody({
            name: "New User",
            email: "new@example.com",
            age: 25
          })
        )
      )
      
      expect(response.status).toBe(201)
      
      const body = yield* response.json
      expect(body).toMatchObject({
        id: "generated-id",
        name: "New User",
        email: "new@example.com"
      })
    }).pipe(
      Effect.provide(TestUserService),
      Effect.runPromise
    )
  )
  
  it("should validate request body", () =>
    Effect.gen(function* () {
      const response = yield* testRequest(
        userRouter,
        HttpClientRequest.post("/users").pipe(
          HttpClientRequest.jsonBody({
            name: "",  // Invalid: empty name
            email: "invalid-email",  // Invalid: bad format
            age: -5  // Invalid: negative age
          })
        )
      )
      
      expect(response.status).toBe(400)
      
      const body = yield* response.json
      expect(body).toHaveProperty("error", "Validation Error")
    }).pipe(
      Effect.provide(TestUserService),
      Effect.runPromise
    )
  )
})

// Integration test with full server
describe("Full Server Integration", () => {
  it("should handle middleware and routing", () =>
    Effect.gen(function* () {
      const client = yield* HttpClient.HttpClient
      
      // Test authentication flow
      const loginResponse = yield* client.execute(
        HttpClientRequest.post("/auth/login").pipe(
          HttpClientRequest.jsonBody({
            email: "test@example.com",
            password: "password123"
          })
        )
      )
      
      expect(loginResponse.status).toBe(200)
      const { token } = yield* loginResponse.json
      
      // Test authenticated request
      const profileResponse = yield* client.execute(
        HttpClientRequest.get("/auth/profile").pipe(
          HttpClientRequest.setHeader("Authorization", `Bearer ${token}`)
        )
      )
      
      expect(profileResponse.status).toBe(200)
    }).pipe(
      Effect.provide(
        Layer.mergeAll(
          HttpClient.layer,
          NodeTesting.TestingLive,
          TestUserService,
          TestAuthService
        )
      ),
      Effect.runPromise
    )
  )
})

// Test with time-based features
describe("Rate Limiting", () => {
  it("should enforce rate limits", () =>
    Effect.gen(function* () {
      const client = yield* HttpClient.HttpClient
      
      // Make requests up to limit
      for (let i = 0; i < 10; i++) {
        const response = yield* client.get("/api/data")
        expect(response.status).toBe(200)
      }
      
      // Next request should be rate limited
      const limitedResponse = yield* client.get("/api/data")
      expect(limitedResponse.status).toBe(429)
      expect(limitedResponse.headers["retry-after"]).toBeDefined()
      
      // Advance time and retry
      yield* TestClock.adjust(Duration.seconds(60))
      
      const retryResponse = yield* client.get("/api/data")
      expect(retryResponse.status).toBe(200)
    }).pipe(
      Effect.provide(
        Layer.mergeAll(
          HttpClient.layer,
          NodeTesting.TestingLive,
          RateLimiterService.testLayer
        )
      ),
      Effect.runPromise
    )
  )
})
```

## Conclusion

HttpServer provides a powerful, type-safe foundation for building production-ready HTTP servers with Effect. It offers superior error handling, composability, and testing capabilities compared to traditional Node.js frameworks.

Key benefits:
- **Type Safety**: Full type inference for requests, responses, and middleware
- **Composability**: Build complex servers from simple, reusable pieces
- **Error Handling**: Automatic error propagation with typed errors
- **Testing**: Built-in testing utilities for comprehensive test coverage
- **Performance**: Efficient streaming and minimal overhead
- **Cross-Platform**: Works seamlessly across Node.js, Bun, and browser environments

HttpServer is ideal for building microservices, REST APIs, real-time applications, and any HTTP-based service that requires reliability, maintainability, and type safety.