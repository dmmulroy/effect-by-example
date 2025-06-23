# ScopedRef: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem ScopedRef Solves

Managing stateful resources that require proper cleanup is a complex challenge in modern applications. Traditional approaches to resource-backed state often lead to memory leaks, dangling connections, and inconsistent resource management:

```typescript
// Traditional approach - problematic code
class DatabaseConnectionPool {
  private connections = new Map<string, DatabaseConnection>()
  private currentConnection: DatabaseConnection | null = null
  
  async switchToDatabase(dbName: string) {
    // Close old connection - easy to forget!
    if (this.currentConnection) {
      await this.currentConnection.close()
    }
    
    // Create new connection
    const connection = await createConnection(dbName)
    this.currentConnection = connection
    this.connections.set(dbName, connection)
  }
  
  async getCurrentData() {
    if (!this.currentConnection) {
      throw new Error("No active connection")
    }
    return await this.currentConnection.query("SELECT * FROM data")
  }
  
  // Manual cleanup - often forgotten or incomplete
  async cleanup() {
    for (const [_, connection] of this.connections) {
      try {
        await connection.close()
      } catch (error) {
        console.error("Failed to close connection:", error)
      }
    }
    this.connections.clear()
    this.currentConnection = null
  }
}

// Usage that can leak resources
const pool = new DatabaseConnectionPool()
try {
  await pool.switchToDatabase("users")
  await pool.switchToDatabase("orders") // Old connection might not be closed
  const data = await pool.getCurrentData()
  
  if (someCondition) {
    return data // Oops! No cleanup called
  }
  
  throw new Error("Something went wrong") // cleanup() never called
} finally {
  await pool.cleanup() // Only works if we remember
}
```

This approach leads to:
- **Resource Leaks** - Old resources not released when switching to new ones
- **Manual Cleanup Burden** - Developers must remember to clean up in all code paths
- **Race Conditions** - Multiple operations can interfere with resource state
- **Exception Safety Issues** - Resources leak when exceptions occur during transitions
- **Complex State Management** - Tracking which resources need cleanup becomes error-prone

### The ScopedRef Solution

ScopedRef provides automatic resource management for stateful references, ensuring that old resources are always properly cleaned up when new ones are acquired:

```typescript
import { Effect, ScopedRef, Scope, Console } from "effect"

// Define a resource with automatic cleanup
const createDatabaseConnection = (dbName: string) =>
  Effect.acquireRelease(
    // Acquire: Create connection
    Effect.gen(function* () {
      yield* Console.log(`Connecting to database: ${dbName}`)
      const connection = yield* Effect.tryPromise({
        try: () => connectToDatabase(dbName),
        catch: () => new Error(`Failed to connect to ${dbName}`)
      })
      return { dbName, connection, query: connection.query.bind(connection) }
    }),
    
    // Release: Always close connection
    (dbConnection) => 
      Console.log(`Closing connection to: ${dbConnection.dbName}`).pipe(
        Effect.zipRight(
          Effect.tryPromise({
            try: () => dbConnection.connection.close(),
            catch: () => new Error("Failed to close connection")
          })
        )
      )
  )

// Create a scoped reference that manages database connections
const createConnectionPool = Effect.gen(function* () {
  const connectionRef = yield* ScopedRef.fromAcquire(
    createDatabaseConnection("default")
  )
  
  const switchDatabase = (dbName: string) =>
    ScopedRef.set(connectionRef, createDatabaseConnection(dbName))
  
  const getCurrentData = Effect.gen(function* () {
    const connection = yield* ScopedRef.get(connectionRef)
    return yield* Effect.tryPromise({
      try: () => connection.query("SELECT * FROM data"),
      catch: () => new Error("Query failed")
    })
  })
  
  return { switchDatabase, getCurrentData } as const
})

// Usage - cleanup is automatic and guaranteed
const program = Effect.gen(function* () {
  const pool = yield* createConnectionPool
  
  // Switch databases - old connections automatically closed
  yield* pool.switchDatabase("users")
  const userData = yield* pool.getCurrentData()
  
  yield* pool.switchDatabase("orders") // Previous connection auto-closed
  const orderData = yield* pool.getCurrentData()
  
  if (someCondition) {
    return { userData, orderData } // All connections auto-closed
  }
  
  yield* Effect.fail("Something went wrong") // Still auto-closed
})

// Run with automatic scope management
const main = Effect.scoped(program)
```

### Key Concepts

**ScopedRef**: A mutable reference whose value is associated with resources that require proper cleanup. When the value changes, old resources are automatically released and new ones are acquired.

**Resource-Backed Values**: Values that require acquisition and release of system resources (connections, file handles, memory, etc.). ScopedRef ensures these resources are managed properly throughout the value's lifetime.

**Automatic Cleanup**: When a ScopedRef's value is updated, the old value's resources are automatically released before the new value's resources are acquired, preventing resource leaks.

**Scope Integration**: ScopedRef works seamlessly with Effect's Scope system, ensuring all resources are cleaned up when the containing scope closes.

## Basic Usage Patterns

### Pattern 1: Simple Resource Reference

```typescript
import { Effect, ScopedRef, Console } from "effect"

// Create a simple resource-backed value
const createResource = (id: string) =>
  Effect.acquireRelease(
    Effect.gen(function* () {
      yield* Console.log(`Acquiring resource: ${id}`)
      return { id, data: `Resource data for ${id}`, timestamp: Date.now() }
    }),
    (resource) => Console.log(`Releasing resource: ${resource.id}`)
  )

// Create and use a ScopedRef
const program = Effect.gen(function* () {
  // Create ScopedRef with initial resource
  const resourceRef = yield* ScopedRef.fromAcquire(createResource("initial"))
  
  // Get current value
  const current = yield* ScopedRef.get(resourceRef)
  yield* Console.log(`Current resource: ${current.id}`)
  
  // Update to new resource (old one automatically released)
  yield* ScopedRef.set(resourceRef, createResource("updated"))
  
  const updated = yield* ScopedRef.get(resourceRef)
  yield* Console.log(`Updated resource: ${updated.id}`)
  
  return updated
})

Effect.runPromise(Effect.scoped(program))
/*
Output:
Acquiring resource: initial
Current resource: initial
Releasing resource: initial
Acquiring resource: updated
Updated resource: updated
Releasing resource: updated
*/
```

### Pattern 2: Simple Value Reference (No Resources)

```typescript
import { Effect, ScopedRef, Console } from "effect"

// Create ScopedRef for simple values that don't require resources
const program = Effect.gen(function* () {
  // Create ScopedRef with a simple value
  const counterRef = yield* ScopedRef.make(() => 0)
  
  // Get current value
  const initial = yield* ScopedRef.get(counterRef)
  yield* Console.log(`Initial value: ${initial}`)
  
  // Update value - no resource management needed
  yield* ScopedRef.set(counterRef, Effect.succeed(42))
  
  const updated = yield* ScopedRef.get(counterRef)
  yield* Console.log(`Updated value: ${updated}`)
  
  return updated
})

Effect.runPromise(Effect.scoped(program))
```

### Pattern 3: Direct Effect Access

```typescript
import { Effect, ScopedRef, Console } from "effect"

// ScopedRef implements Effect<A>, so you can use it directly
const program = Effect.gen(function* () {
  const valueRef = yield* ScopedRef.make(() => "Hello, World!")
  
  // Direct access - ScopedRef is an Effect that yields its current value
  const value = yield* valueRef
  yield* Console.log(`Value: ${value}`)
  
  // You can also pipe ScopedRef like any other Effect
  const processed = yield* valueRef.pipe(
    Effect.map(str => str.toUpperCase()),
    Effect.map(str => `Processed: ${str}`)
  )
  yield* Console.log(processed)
  
  return processed
})

Effect.runPromise(Effect.scoped(program))
```

## Real-World Examples

### Example 1: HTTP Client Connection Management

Managing HTTP client connections that need to be properly configured and cleaned up based on different service endpoints.

```typescript
import { Effect, ScopedRef, Console, Schedule, Duration } from "effect"

// Define HTTP client configuration
interface HttpClientConfig {
  baseUrl: string
  timeout: number
  retries: number
}

interface HttpClient {
  config: HttpClientConfig
  get: (path: string) => Effect.Effect<any, Error>
  post: (path: string, data: any) => Effect.Effect<any, Error>
  close: () => Promise<void>
}

// Create HTTP client with proper resource management
const createHttpClient = (config: HttpClientConfig) =>
  Effect.acquireRelease(
    Effect.gen(function* () {
      yield* Console.log(`Creating HTTP client for ${config.baseUrl}`)
      
      // Simulate creating an HTTP client with connection pooling
      const client: HttpClient = {
        config,
        get: (path: string) => 
          Effect.tryPromise({
            try: () => fetch(`${config.baseUrl}${path}`).then(r => r.json()),
            catch: (error) => new Error(`GET ${path} failed: ${error}`)
          }).pipe(
            Effect.retry({
              schedule: Schedule.exponential(Duration.millis(100)),
              times: config.retries
            })
          ),
        
        post: (path: string, data: any) =>
          Effect.tryPromise({
            try: () => fetch(`${config.baseUrl}${path}`, {
              method: 'POST',
              body: JSON.stringify(data),
              headers: { 'Content-Type': 'application/json' }
            }).then(r => r.json()),
            catch: (error) => new Error(`POST ${path} failed: ${error}`)
          }).pipe(
            Effect.retry({
              schedule: Schedule.exponential(Duration.millis(100)),
              times: config.retries
            })
          ),
        
        close: async () => {
          // Simulate closing connection pool
          await new Promise(resolve => setTimeout(resolve, 100))
        }
      }
      
      return client
    }),
    
    (client) => 
      Console.log(`Closing HTTP client for ${client.config.baseUrl}`).pipe(
        Effect.zipRight(
          Effect.tryPromise({
            try: () => client.close(),
            catch: () => new Error("Failed to close HTTP client")
          })
        )
      )
  )

// Service that manages HTTP clients for different environments
const createApiService = Effect.gen(function* () {
  // Create ScopedRef for HTTP client management
  const clientRef = yield* ScopedRef.fromAcquire(
    createHttpClient({
      baseUrl: "https://api.staging.example.com",
      timeout: 5000,
      retries: 3
    })
  )
  
  const switchEnvironment = (env: "staging" | "production" | "development") => {
    const configs = {
      staging: { baseUrl: "https://api.staging.example.com", timeout: 5000, retries: 3 },
      production: { baseUrl: "https://api.example.com", timeout: 3000, retries: 5 },
      development: { baseUrl: "http://localhost:3000", timeout: 10000, retries: 1 }
    }
    
    return ScopedRef.set(clientRef, createHttpClient(configs[env]))
  }
  
  const getUsers = Effect.gen(function* () {
    const client = yield* ScopedRef.get(clientRef)
    return yield* client.get("/users")
  })
  
  const createUser = (userData: any) => Effect.gen(function* () {
    const client = yield* ScopedRef.get(clientRef)
    return yield* client.post("/users", userData)
  })
  
  const getCurrentConfig = Effect.gen(function* () {
    const client = yield* ScopedRef.get(clientRef)
    return client.config
  })
  
  return { switchEnvironment, getUsers, createUser, getCurrentConfig } as const
})

// Usage in an application
const applicationWorkflow = Effect.gen(function* () {
  const apiService = yield* createApiService
  
  // Start with staging
  yield* Console.log("Working with staging environment...")
  const stagingUsers = yield* apiService.getUsers()
  
  // Switch to production - staging client automatically closed
  yield* Console.log("Switching to production...")
  yield* apiService.switchEnvironment("production")
  
  const prodUsers = yield* apiService.getUsers()
  
  // Create a new user in production
  const newUser = yield* apiService.createUser({ 
    name: "John Doe", 
    email: "john@example.com" 
  })
  
  // Switch to development for testing - production client auto-closed
  yield* Console.log("Switching to development for testing...")
  yield* apiService.switchEnvironment("development")
  
  const config = yield* apiService.getCurrentConfig()
  yield* Console.log(`Current config: ${JSON.stringify(config)}`)
  
  return { stagingUsers, prodUsers, newUser, config }
})

// Run the application - all HTTP clients are automatically cleaned up
const main = Effect.scoped(applicationWorkflow)
```

### Example 2: Database Connection Pooling with Cache Management

A database service that manages both connection pools and associated caches, ensuring both are properly cleaned up when switching databases.

```typescript
import { Effect, ScopedRef, Console, Ref, Schedule, Duration } from "effect"

interface DatabaseConnection {
  host: string
  database: string
  query: (sql: string) => Promise<any[]>
  close: () => Promise<void>
}

interface CacheEntry {
  data: any
  timestamp: number
  ttl: number
}

interface DatabaseService {
  connection: DatabaseConnection
  cache: Map<string, CacheEntry>
  stats: { queries: number, cacheHits: number, cacheMisses: number }
}

// Create database service with connection and cache
const createDatabaseService = (host: string, database: string) =>
  Effect.acquireRelease(
    Effect.gen(function* () {
      yield* Console.log(`Connecting to database ${database} on ${host}`)
      
      // Simulate creating database connection
      const connection: DatabaseConnection = {
        host,
        database,
        query: async (sql: string) => {
          await new Promise(resolve => setTimeout(resolve, Math.random() * 100))
          return [{ id: 1, data: `Result from ${database}` }]
        },
        close: async () => {
          await new Promise(resolve => setTimeout(resolve, 50))
        }
      }
      
      // Test connection
      yield* Effect.tryPromise({
        try: () => connection.query("SELECT 1"),
        catch: () => new Error(`Failed to connect to ${database}`)
      })
      
      const service: DatabaseService = {
        connection,
        cache: new Map<string, CacheEntry>(),
        stats: { queries: 0, cacheHits: 0, cacheMisses: 0 }
      }
      
      yield* Console.log(`Database service ready for ${database}`)
      return service
    }),
    
    (service) => 
      Console.log(`Closing database service for ${service.connection.database}`).pipe(
        Effect.zipRight(
          Effect.gen(function* () {
            // Clear cache
            service.cache.clear()
            
            // Close connection
            yield* Effect.tryPromise({
              try: () => service.connection.close(),
              catch: () => new Error("Failed to close database connection")
            })
            
            // Log final stats
            yield* Console.log(
              `Final stats for ${service.connection.database}: ` +
              `${service.stats.queries} queries, ` +
              `${service.stats.cacheHits} cache hits, ` +
              `${service.stats.cacheMisses} cache misses`
            )
          })
        )
      )
  )

// Repository that manages database services
const createUserRepository = Effect.gen(function* () {
  // Start with primary database
  const dbServiceRef = yield* ScopedRef.fromAcquire(
    createDatabaseService("primary.db.example.com", "users")
  )
  
  const switchDatabase = (host: string, database: string) =>
    ScopedRef.set(dbServiceRef, createDatabaseService(host, database))
  
  const queryWithCache = (sql: string, cacheKey?: string) => 
    Effect.gen(function* () {
      const service = yield* ScopedRef.get(dbServiceRef)
      const now = Date.now()
      
      // Check cache first if key provided
      if (cacheKey) {
        const cached = service.cache.get(cacheKey)
        if (cached && (now - cached.timestamp) < cached.ttl) {
          service.stats.cacheHits++
          yield* Console.log(`Cache hit for key: ${cacheKey}`)
          return cached.data
        }
        service.stats.cacheMisses++
      }
      
      // Execute query
      service.stats.queries++
      const result = yield* Effect.tryPromise({
        try: () => service.connection.query(sql),
        catch: (error) => new Error(`Query failed: ${error}`)
      })
      
      // Cache result if key provided
      if (cacheKey) {
        service.cache.set(cacheKey, {
          data: result,
          timestamp: now,
          ttl: 300_000 // 5 minutes
        })
      }
      
      return result
    })
  
  const getUsers = queryWithCache("SELECT * FROM users", "all_users")
  
  const getUserById = (id: number) => 
    queryWithCache(`SELECT * FROM users WHERE id = ${id}`, `user_${id}`)
  
  const getStats = Effect.gen(function* () {
    const service = yield* ScopedRef.get(dbServiceRef)
    return {
      ...service.stats,
      cacheSize: service.cache.size,
      database: service.connection.database,
      host: service.connection.host
    }
  })
  
  return { switchDatabase, getUsers, getUserById, getStats } as const
})

// Application that uses repository with database switching
const databaseApplication = Effect.gen(function* () {
  const userRepo = yield* createUserRepository
  
  yield* Console.log("=== Primary Database Operations ===")
  const primaryUsers = yield* userRepo.getUsers()
  const user1 = yield* userRepo.getUserById(1)
  const user1Again = yield* userRepo.getUserById(1) // Should hit cache
  
  let stats = yield* userRepo.getStats()
  yield* Console.log(`Primary stats: ${JSON.stringify(stats)}`)
  
  yield* Console.log("\n=== Switching to Replica Database ===")
  yield* userRepo.switchDatabase("replica.db.example.com", "users_replica")
  
  // Cache is cleared, connection is new
  const replicaUsers = yield* userRepo.getUsers()
  const user2 = yield* userRepo.getUserById(2)
  
  stats = yield* userRepo.getStats()
  yield* Console.log(`Replica stats: ${JSON.stringify(stats)}`)
  
  yield* Console.log("\n=== Switching to Analytics Database ===")
  yield* userRepo.switchDatabase("analytics.db.example.com", "user_analytics")
  
  const analyticsData = yield* userRepo.getUsers()
  
  stats = yield* userRepo.getStats()
  yield* Console.log(`Analytics stats: ${JSON.stringify(stats)}`)
  
  return { primaryUsers, user1, replicaUsers, user2, analyticsData, finalStats: stats }
})

// Run application - all database connections and caches cleaned up automatically
const main = Effect.scoped(databaseApplication)
```

### Example 3: WebSocket Connection Manager with Automatic Reconnection

Managing WebSocket connections that require reconnection logic and proper cleanup of event listeners and pending requests.

```typescript
import { Effect, ScopedRef, Console, Ref, Queue, Fiber, Duration, Schedule } from "effect"

interface WebSocketMessage {
  type: string
  data: any
  timestamp: number
}

interface WebSocketConnection {
  url: string
  socket: WebSocket
  messageQueue: Queue.Queue<WebSocketMessage>
  connectionState: "connecting" | "connected" | "disconnected" | "error"
  lastPing: number
  send: (message: any) => Effect.Effect<void, Error>
  close: () => Promise<void>
}

// Create WebSocket connection with automatic reconnection
const createWebSocketConnection = (url: string, options?: { 
  pingInterval?: number
  reconnectAttempts?: number 
}) =>
  Effect.acquireRelease(
    Effect.gen(function* () {
      yield* Console.log(`Connecting to WebSocket: ${url}`)
      
      const messageQueue = yield* Queue.unbounded<WebSocketMessage>()
      const connectionStateRef = yield* Ref.make<WebSocketConnection["connectionState"]>("connecting")
      const lastPingRef = yield* Ref.make(Date.now())
      
      // Create WebSocket
      const socket = yield* Effect.tryPromise({
        try: () => new Promise<WebSocket>((resolve, reject) => {
          const ws = new WebSocket(url)
          ws.onopen = () => resolve(ws)
          ws.onerror = (error) => reject(new Error(`WebSocket connection failed: ${error}`))
          
          setTimeout(() => reject(new Error("Connection timeout")), 10000)
        }),
        catch: (error) => new Error(`Failed to create WebSocket: ${error}`)
      })
      
      // Set up event handlers
      socket.onmessage = (event) => {
        const message: WebSocketMessage = {
          type: "message",
          data: JSON.parse(event.data),
          timestamp: Date.now()
        }
        Effect.runSync(Queue.offer(messageQueue, message))
      }
      
      socket.onclose = () => {
        Effect.runSync(Ref.set(connectionStateRef, "disconnected"))
      }
      
      socket.onerror = () => {
        Effect.runSync(Ref.set(connectionStateRef, "error"))
      }
      
      yield* Ref.set(connectionStateRef, "connected")
      
      const connection: WebSocketConnection = {
        url,
        socket,
        messageQueue,
        connectionState: "connected",
        lastPing: Date.now(),
        
        send: (message: any) => Effect.gen(function* () {
          const state = yield* Ref.get(connectionStateRef)
          if (state !== "connected") {
            yield* Effect.fail(new Error(`Cannot send message: connection is ${state}`))
          }
          
          yield* Effect.tryPromise({
            try: () => new Promise<void>((resolve, reject) => {
              if (socket.readyState === WebSocket.OPEN) {
                socket.send(JSON.stringify(message))
                resolve()
              } else {
                reject(new Error("WebSocket is not open"))
              }
            }),
            catch: (error) => new Error(`Failed to send message: ${error}`)
          })
        }),
        
        close: async () => {
          if (socket.readyState === WebSocket.OPEN) {
            socket.close()
          }
          // Wait for close event
          await new Promise(resolve => {
            if (socket.readyState === WebSocket.CLOSED) {
              resolve(void 0)
            } else {
              socket.onclose = () => resolve(void 0)
              setTimeout(resolve, 1000) // Force close after 1s
            }
          })
        }
      }
      
      // Start ping/pong to keep connection alive
      const pingFiber = yield* Effect.fork(
        Effect.gen(function* () {
          yield* Effect.repeat(
            Effect.gen(function* () {
              const state = yield* Ref.get(connectionStateRef)
              if (state === "connected") {
                yield* connection.send({ type: "ping", timestamp: Date.now() })
                yield* Ref.set(lastPingRef, Date.now())
              }
            }),
            {
              schedule: Schedule.fixed(Duration.seconds(options?.pingInterval ?? 30)),
              while: () => Effect.gen(function* () {
                const state = yield* Ref.get(connectionStateRef)
                return state === "connected"
              })
            }
          )
        })
      )
      
      yield* Console.log(`WebSocket connected to ${url}`)
      return { connection, pingFiber }
    }),
    
    ({ connection, pingFiber }) =>
      Console.log(`Closing WebSocket connection to ${connection.url}`).pipe(
        Effect.zipRight(
          Effect.gen(function* () {
            // Interrupt ping fiber
            yield* Fiber.interrupt(pingFiber)
            
            // Close WebSocket
            yield* Effect.tryPromise({
              try: () => connection.close(),
              catch: () => new Error("Failed to close WebSocket")
            })
            
            // Clear message queue
            const remainingMessages = yield* Queue.size(connection.messageQueue)
            if (remainingMessages > 0) {
              yield* Console.log(`Discarded ${remainingMessages} pending messages`)
            }
          })
        )
      )
  )

// WebSocket service manager
const createWebSocketService = Effect.gen(function* () {
  // Start with primary WebSocket endpoint
  const wsConnectionRef = yield* ScopedRef.fromAcquire(
    createWebSocketConnection("wss://api.example.com/ws")
  )
  
  const switchEndpoint = (url: string, options?: { 
    pingInterval?: number
    reconnectAttempts?: number 
  }) =>
    ScopedRef.set(wsConnectionRef, createWebSocketConnection(url, options))
  
  const sendMessage = (message: any) => Effect.gen(function* () {
    const { connection } = yield* ScopedRef.get(wsConnectionRef)
    yield* connection.send(message)
  })
  
  const receiveMessages = Effect.gen(function* () {
    const { connection } = yield* ScopedRef.get(wsConnectionRef)
    return yield* Queue.take(connection.messageQueue)
  })
  
  const getConnectionInfo = Effect.gen(function* () {
    const { connection } = yield* ScopedRef.get(wsConnectionRef)
    const queueSize = yield* Queue.size(connection.messageQueue)
    return {
      url: connection.url,
      state: connection.connectionState,
      lastPing: connection.lastPing,
      pendingMessages: queueSize
    }
  })
  
  const subscribeToMessages = (handler: (message: WebSocketMessage) => Effect.Effect<void>) =>
    Effect.gen(function* () {
      yield* Effect.forever(
        Effect.gen(function* () {
          const message = yield* receiveMessages
          yield* handler(message)
        })
      )
    })
  
  return { 
    switchEndpoint, 
    sendMessage, 
    receiveMessages, 
    getConnectionInfo, 
    subscribeToMessages 
  } as const
})

// Application using WebSocket service
const webSocketApplication = Effect.gen(function* () {
  const wsService = yield* createWebSocketService
  
  yield* Console.log("=== Primary WebSocket Connection ===")
  yield* wsService.sendMessage({ type: "subscribe", channel: "updates" })
  
  // Start message subscription in background
  const subscriptionFiber = yield* Effect.fork(
    wsService.subscribeToMessages((message) =>
      Console.log(`Received: ${JSON.stringify(message)}`)
    )
  )
  
  // Simulate some activity
  yield* Effect.sleep(Duration.seconds(2))
  yield* wsService.sendMessage({ type: "get_status" })
  
  let info = yield* wsService.getConnectionInfo()
  yield* Console.log(`Connection info: ${JSON.stringify(info)}`)
  
  yield* Console.log("\n=== Switching to Backup WebSocket ===")
  yield* wsService.switchEndpoint("wss://backup.example.com/ws", { 
    pingInterval: 60 
  })
  
  // Old connection and subscription automatically cleaned up
  yield* wsService.sendMessage({ type: "subscribe", channel: "backup_updates" })
  
  yield* Effect.sleep(Duration.seconds(1))
  info = yield* wsService.getConnectionInfo()
  yield* Console.log(`New connection info: ${JSON.stringify(info)}`)
  
  yield* Console.log("\n=== Switching to Development WebSocket ===")
  yield* wsService.switchEndpoint("ws://localhost:8080/ws", { 
    pingInterval: 10 
  })
  
  yield* wsService.sendMessage({ type: "test_message", data: "Hello dev server!" })
  
  info = yield* wsService.getConnectionInfo()
  yield* Console.log(`Dev connection info: ${JSON.stringify(info)}`)
  
  // Clean up subscription
  yield* Fiber.interrupt(subscriptionFiber)
  
  return info
})

// Run application - all WebSocket connections automatically cleaned up
const main = Effect.scoped(webSocketApplication)
```

## Advanced Features Deep Dive

### Feature 1: Atomic Resource Swapping

ScopedRef ensures that resource transitions are atomic - old resources are fully released before new ones become active, preventing resource conflicts.

#### Basic Atomic Swapping

```typescript
import { Effect, ScopedRef, Console, Ref } from "effect"

// Resource that tracks its lifecycle
const createTrackedResource = (id: string) => {
  const stateRef = Ref.unsafeMake<"created" | "active" | "released">("created")
  
  return Effect.acquireRelease(
    Effect.gen(function* () {
      yield* Console.log(`Creating resource: ${id}`)
      yield* Ref.set(stateRef, "active")
      
      return {
        id,
        getState: () => Ref.get(stateRef),
        process: (data: any) => 
          Effect.gen(function* () {
            const state = yield* Ref.get(stateRef)
            if (state !== "active") {
              yield* Effect.fail(new Error(`Resource ${id} is not active (state: ${state})`))
            }
            yield* Console.log(`Processing with resource ${id}: ${JSON.stringify(data)}`)
            return `Processed by ${id}: ${data}`
          })
      }
    }),
    
    (resource) =>
      Effect.gen(function* () {
        yield* Console.log(`Releasing resource: ${resource.id}`)
        yield* Ref.set(stateRef, "released")
      })
  )
}

// Demonstrate atomic swapping
const atomicSwapDemo = Effect.gen(function* () {
  const resourceRef = yield* ScopedRef.fromAcquire(createTrackedResource("resource-1"))
  
  // Use initial resource
  const resource1 = yield* ScopedRef.get(resourceRef)
  yield* resource1.process("initial data")
  
  // Atomic swap to new resource
  yield* Console.log("\n--- Starting atomic swap ---")
  yield* ScopedRef.set(resourceRef, createTrackedResource("resource-2"))
  yield* Console.log("--- Atomic swap complete ---\n")
  
  // Use new resource
  const resource2 = yield* ScopedRef.get(resourceRef)
  yield* resource2.process("new data")
  
  // Verify old resource is released
  const oldState = yield* resource1.getState()
  const newState = yield* resource2.getState()
  
  yield* Console.log(`Old resource state: ${oldState}`)
  yield* Console.log(`New resource state: ${newState}`)
  
  return { oldState, newState }
})

Effect.runPromise(Effect.scoped(atomicSwapDemo))
/*
Output:
Creating resource: resource-1
Processing with resource resource-1: "initial data"

--- Starting atomic swap ---
Releasing resource: resource-1
Creating resource: resource-2
--- Atomic swap complete ---

Processing with resource resource-2: "new data"
Old resource state: released
New resource state: active
Releasing resource: resource-2
*/
```

#### Real-World Atomic Swapping: Configuration Hot-Reloading

```typescript
import { Effect, ScopedRef, Console, Ref, Schedule, Duration } from "effect"

interface AppConfig {
  databaseUrl: string
  apiKey: string
  logLevel: "debug" | "info" | "warn" | "error"
  rateLimits: { requests: number, windowMs: number }
}

interface ConfiguredService {
  config: AppConfig
  rateLimiter: {
    requests: Ref.Ref<number>
    windowStart: Ref.Ref<number>
  }
  logger: {
    debug: (msg: string) => Effect.Effect<void>
    info: (msg: string) => Effect.Effect<void>
    warn: (msg: string) => Effect.Effect<void>
    error: (msg: string) => Effect.Effect<void>
  }
  processRequest: (request: any) => Effect.Effect<string, Error>
}

// Create service with configuration
const createConfiguredService = (config: AppConfig) =>
  Effect.acquireRelease(
    Effect.gen(function* () {
      yield* Console.log(`Initializing service with config: ${JSON.stringify(config)}`)
      
      // Initialize rate limiter
      const rateLimiter = {
        requests: yield* Ref.make(0),
        windowStart: yield* Ref.make(Date.now())
      }
      
      // Create logger based on config
      const shouldLog = (level: string) => {
        const levels = ["debug", "info", "warn", "error"]
        return levels.indexOf(level) >= levels.indexOf(config.logLevel)
      }
      
      const logger = {
        debug: (msg: string) => shouldLog("debug") ? Console.log(`[DEBUG] ${msg}`) : Effect.void,
        info: (msg: string) => shouldLog("info") ? Console.log(`[INFO] ${msg}`) : Effect.void,
        warn: (msg: string) => shouldLog("warn") ? Console.log(`[WARN] ${msg}`) : Effect.void,
        error: (msg: string) => shouldLog("error") ? Console.log(`[ERROR] ${msg}`) : Effect.void
      }
      
      // Service implementation
      const service: ConfiguredService = {
        config,
        rateLimiter,
        logger,
        
        processRequest: (request: any) => 
          Effect.gen(function* () {
            const now = Date.now()
            const windowStart = yield* Ref.get(rateLimiter.windowStart)
            const currentRequests = yield* Ref.get(rateLimiter.requests)
            
            // Reset window if needed
            if (now - windowStart > config.rateLimits.windowMs) {
              yield* Ref.set(rateLimiter.windowStart, now)
              yield* Ref.set(rateLimiter.requests, 0)
            }
            
            // Check rate limit
            if (currentRequests >= config.rateLimits.requests) {
              yield* logger.warn(`Rate limit exceeded: ${currentRequests}/${config.rateLimits.requests}`)
              yield* Effect.fail(new Error("Rate limit exceeded"))
            }
            
            // Process request
            yield* Ref.update(rateLimiter.requests, n => n + 1)
            yield* logger.debug(`Processing request: ${JSON.stringify(request)}`)
            
            // Simulate processing
            yield* Effect.sleep(Duration.millis(100))
            
            const response = `Processed: ${request.data} with key ${config.apiKey.substring(0, 4)}***`
            yield* logger.info(`Request processed successfully`)
            
            return response
          })
      }
      
      yield* logger.info("Service initialization complete")
      return service
    }),
    
    (service) =>
      service.logger.info("Shutting down service").pipe(
        Effect.zipRight(Console.log("Service cleanup complete"))
      )
  )

// Configuration manager with hot reload
const createConfigManager = Effect.gen(function* () {
  // Initial configuration
  const initialConfig: AppConfig = {
    databaseUrl: "postgres://localhost:5432/app",
    apiKey: "api-key-12345",
    logLevel: "info",
    rateLimits: { requests: 10, windowMs: 60000 }
  }
  
  const serviceRef = yield* ScopedRef.fromAcquire(createConfiguredService(initialConfig))
  
  const reloadConfig = (newConfig: AppConfig) => 
    ScopedRef.set(serviceRef, createConfiguredService(newConfig))
  
  const processRequest = (request: any) => Effect.gen(function* () {
    const service = yield* ScopedRef.get(serviceRef)
    return yield* service.processRequest(request)
  })
  
  const getCurrentConfig = Effect.gen(function* () {
    const service = yield* ScopedRef.get(serviceRef)
    return service.config
  })
  
  return { reloadConfig, processRequest, getCurrentConfig } as const
})

// Application demonstrating hot config reload
const configReloadDemo = Effect.gen(function* () {
  const configManager = yield* createConfigManager
  
  yield* Console.log("=== Initial Configuration ===")
  const config1 = yield* configManager.getCurrentConfig()
  yield* Console.log(`Config: ${JSON.stringify(config1)}`)
  
  // Process some requests
  yield* configManager.processRequest({ data: "request-1" })
  yield* configManager.processRequest({ data: "request-2" })
  
  yield* Console.log("\n=== Hot Reload: Debug Logging + Higher Rate Limits ===")
  yield* configManager.reloadConfig({
    databaseUrl: "postgres://prod.example.com:5432/app",
    apiKey: "prod-api-key-67890",
    logLevel: "debug", // More verbose logging
    rateLimits: { requests: 100, windowMs: 60000 } // Higher limits
  })
  
  // Old service is atomically replaced - no requests are lost
  yield* configManager.processRequest({ data: "request-3" })
  yield* configManager.processRequest({ data: "request-4" })
  
  yield* Console.log("\n=== Hot Reload: Error-Only Logging + Strict Rate Limits ===")
  yield* configManager.reloadConfig({
    databaseUrl: "postgres://backup.example.com:5432/app",
    apiKey: "backup-api-key-11111",
    logLevel: "error", // Minimal logging
    rateLimits: { requests: 2, windowMs: 10000 } // Very strict limits
  })
  
  // Test new rate limits
  yield* configManager.processRequest({ data: "request-5" })
  yield* configManager.processRequest({ data: "request-6" })
  
  // This should fail due to rate limiting
  const result = yield* Effect.either(
    configManager.processRequest({ data: "request-7" })
  )
  
  yield* Console.log(`Rate limit test result: ${result._tag}`)
  
  const finalConfig = yield* configManager.getCurrentConfig()
  return finalConfig
})

Effect.runPromise(Effect.scoped(configReloadDemo))
```

### Feature 2: Integration with Scope Lifecycle

ScopedRef integrates seamlessly with Effect's Scope system, ensuring all resources are cleaned up when scopes close.

#### Nested Scope Management

```typescript
import { Effect, ScopedRef, Scope, Console, Fiber, Duration } from "effect"

// Resource that requires nested scope management
const createNestedResource = (name: string, children: string[] = []) =>
  Effect.acquireRelease(
    Effect.gen(function* () {
      yield* Console.log(`Creating ${name}`)
      
      // Create child resources in nested scopes
      const childResources = yield* Effect.forEach(children, (childName) =>
        Effect.acquireRelease(
          Effect.gen(function* () {
            yield* Console.log(`  Creating child: ${childName}`)
            return { name: childName, data: `Child data for ${childName}` }
          }),
          (child) => Console.log(`  Releasing child: ${child.name}`)
        )
      )
      
      return {
        name,
        children: childResources,
        getData: () => ({
          parent: name,
          childData: childResources.map(c => c.data)
        })
      }
    }),
    
    (resource) => Console.log(`Releasing ${resource.name} and ${resource.children.length} children`)
  )

// Demonstrate nested scope cleanup
const nestedScopeDemo = Effect.gen(function* () {
  const resourceRef = yield* ScopedRef.fromAcquire(
    createNestedResource("Parent-1", ["Child-A", "Child-B"])
  )
  
  const resource1 = yield* ScopedRef.get(resourceRef)
  yield* Console.log(`Data: ${JSON.stringify(resource1.getData())}`)
  
  yield* Console.log("\n--- Switching to new nested resource ---")
  yield* ScopedRef.set(resourceRef, 
    createNestedResource("Parent-2", ["Child-X", "Child-Y", "Child-Z"])
  )
  
  const resource2 = yield* ScopedRef.get(resourceRef)
  yield* Console.log(`New data: ${JSON.stringify(resource2.getData())}`)
  
  return resource2.getData()
})

Effect.runPromise(Effect.scoped(nestedScopeDemo))
/*
Output:
Creating Parent-1
  Creating child: Child-A
  Creating child: Child-B
Data: {"parent":"Parent-1","childData":["Child data for Child-A","Child data for Child-B"]}

--- Switching to new nested resource ---
Releasing Parent-1 and 2 children
  Releasing child: Child-A
  Releasing child: Child-B
Creating Parent-2
  Creating child: Child-X
  Creating child: Child-Y
  Creating child: Child-Z
New data: {"parent":"Parent-2","childData":["Child data for Child-X","Child data for Child-Y","Child data for Child-Z"]}
Releasing Parent-2 and 3 children
  Releasing child: Child-X
  Releasing child: Child-Y
  Releasing child: Child-Z
*/
```

#### Manual Scope Control

```typescript
import { Effect, ScopedRef, Scope, Console } from "effect"

// Helper to create ScopedRef with manual scope control
const withManualScope = <A, E, R>(
  acquire: Effect.Effect<A, E, R>
) => Effect.gen(function* () {
  const scope = yield* Scope.make()
  const scopedRef = yield* ScopedRef.fromAcquire(acquire).pipe(
    Effect.provideService(Scope.Scope, scope)
  )
  
  const closeScope = () => Scope.close(scope, Effect.exitVoid)
  
  return { scopedRef, closeScope } as const
})

// Demonstrate manual scope control
const manualScopeDemo = Effect.gen(function* () {
  const createResource = (id: string) =>
    Effect.acquireRelease(
      Console.log(`Acquiring ${id}`).pipe(Effect.as({ id, data: `Data for ${id}` })),
      (resource) => Console.log(`Releasing ${resource.id}`)
    )
  
  yield* Console.log("=== Manual Scope Control ===")
  
  // Create ScopedRef with manual scope
  const { scopedRef, closeScope } = yield* withManualScope(createResource("manual-1"))
  
  const resource = yield* ScopedRef.get(scopedRef)
  yield* Console.log(`Got resource: ${resource.id}`)
  
  // Update resource
  yield* ScopedRef.set(scopedRef, createResource("manual-2"))
  const updated = yield* ScopedRef.get(scopedRef)
  yield* Console.log(`Updated resource: ${updated.id}`)
  
  // Manually close scope - all resources cleaned up
  yield* Console.log("Manually closing scope...")
  yield* closeScope()
  
  yield* Console.log("Scope closed - all resources released")
  
  return updated
})

Effect.runPromise(manualScopeDemo)
```

### Feature 3: Error Handling and Recovery

ScopedRef provides robust error handling during resource transitions, ensuring system consistency even when resource acquisition fails.

#### Graceful Error Recovery

```typescript
import { Effect, ScopedRef, Console, Schedule, Duration, Random } from "effect"

// Resource that can fail during creation
const createUnreliableResource = (id: string, failureRate: number = 0.3) =>
  Effect.acquireRelease(
    Effect.gen(function* () {
      yield* Console.log(`Attempting to create resource: ${id}`)
      
      // Simulate random failures
      const shouldFail = yield* Random.nextBoolean.pipe(
        Effect.map(random => random && Math.random() < failureRate)
      )
      
      if (shouldFail) {
        yield* Effect.fail(new Error(`Failed to create resource: ${id}`))
      }
      
      // Simulate creation time
      yield* Effect.sleep(Duration.millis(100))
      
      yield* Console.log(`Successfully created resource: ${id}`)
      return { id, data: `Resource data for ${id}`, createdAt: Date.now() }
    }),
    
    (resource) => Console.log(`Releasing resource: ${resource.id}`)
  )

// Service with error recovery
const createResilientService = Effect.gen(function* () {
  // Start with a reliable initial resource
  const serviceRef = yield* ScopedRef.fromAcquire(
    Effect.acquireRelease(
      Effect.succeed({ id: "initial", data: "Initial safe resource", createdAt: Date.now() }),
      (resource) => Console.log(`Releasing initial resource: ${resource.id}`)
    )
  )
  
  const switchResource = (newId: string, retryOptions?: {
    attempts?: number
    delay?: Duration.Duration
  }) => {
    const attempts = retryOptions?.attempts ?? 3
    const delay = retryOptions?.delay ?? Duration.millis(500)
    
    return ScopedRef.set(serviceRef, createUnreliableResource(newId)).pipe(
      Effect.retry({
        schedule: Schedule.exponential(delay),
        times: attempts - 1
      }),
      Effect.catchAll((error) => 
        Console.log(`Failed to switch to ${newId} after ${attempts} attempts: ${error.message}`).pipe(
          Effect.zipRight(Effect.fail(error))
        )
      )
    )
  }
  
  const switchResourceWithFallback = (
    newId: string, 
    fallbackId: string,
    retryOptions?: { attempts?: number, delay?: Duration.Duration }
  ) => {
    return switchResource(newId, retryOptions).pipe(
      Effect.catchAll((primaryError) =>
        Console.log(`Primary resource ${newId} failed, trying fallback ${fallbackId}`).pipe(
          Effect.zipRight(switchResource(fallbackId, { attempts: 1 })),
          Effect.catchAll((fallbackError) =>
            Console.log(`Both primary and fallback failed!`).pipe(
              Effect.zipRight(Effect.fail(
                new Error(`Primary: ${primaryError.message}, Fallback: ${fallbackError.message}`)
              ))
            )
          )
        )
      )
    )
  }
  
  const getCurrentResource = ScopedRef.get(serviceRef)
  
  const getResourceInfo = Effect.gen(function* () {
    const resource = yield* getCurrentResource
    return {
      id: resource.id,
      data: resource.data.substring(0, 50),
      age: Date.now() - resource.createdAt
    }
  })
  
  return { 
    switchResource, 
    switchResourceWithFallback, 
    getCurrentResource, 
    getResourceInfo 
  } as const
})

// Demonstrate error handling and recovery
const errorRecoveryDemo = Effect.gen(function* () {
  const service = yield* createResilientService
  
  yield* Console.log("=== Initial State ===")
  let info = yield* service.getResourceInfo()
  yield* Console.log(`Current resource: ${JSON.stringify(info)}`)
  
  yield* Console.log("\n=== Attempting Switch (May Fail) ===")
  const switchResult1 = yield* Effect.either(
    service.switchResource("unreliable-1", { attempts: 2, delay: Duration.millis(200) })
  )
  
  if (switchResult1._tag === "Left") {
    yield* Console.log(`Switch failed as expected: ${switchResult1.left.message}`)
  } else {
    yield* Console.log("Switch succeeded!")
  }
  
  info = yield* service.getResourceInfo()
  yield* Console.log(`Resource after switch attempt: ${JSON.stringify(info)}`)
  
  yield* Console.log("\n=== Switch with Fallback ===")
  const switchResult2 = yield* Effect.either(
    service.switchResourceWithFallback(
      "unreliable-2", 
      "reliable-fallback",
      { attempts: 2, delay: Duration.millis(200) }
    )
  )
  
  if (switchResult2._tag === "Left") {
    yield* Console.log(`Both primary and fallback failed: ${switchResult2.left.message}`)
  } else {
    yield* Console.log("Switch with fallback succeeded!")
  }
  
  info = yield* service.getResourceInfo()
  yield* Console.log(`Final resource: ${JSON.stringify(info)}`)
  
  yield* Console.log("\n=== Multiple Attempts ===")
  const attempts = yield* Effect.forEach([1, 2, 3, 4, 5], (attempt) =>
    Effect.gen(function* () {
      yield* Console.log(`\nAttempt ${attempt}:`)
      const result = yield* Effect.either(
        service.switchResource(`attempt-${attempt}`, { attempts: 1 })
      )
      
      const info = yield* service.getResourceInfo()
      return { attempt, success: result._tag === "Right", resourceId: info.id }
    })
  )
  
  const successCount = attempts.filter(a => a.success).length
  yield* Console.log(`\nSuccess rate: ${successCount}/${attempts.length}`)
  
  return { attempts, finalResource: info }
})

Effect.runPromise(Effect.scoped(errorRecoveryDemo))
```

## Practical Patterns & Best Practices

### Pattern 1: Resource Pool Management

Creating a reusable pattern for managing pools of resources with automatic cleanup and rotation.

```typescript
import { Effect, ScopedRef, Console, Ref, Array as Arr, Duration, Random } from "effect"

// Generic resource pool manager
const createResourcePool = <T>(
  createResource: (id: string) => Effect.Effect<T, Error, Scope.Scope>,
  poolSize: number,
  rotationInterval?: Duration.Duration
) => Effect.gen(function* () {
  // Track pool state
  const poolStateRef = yield* Ref.make<{
    resources: Array<{ id: string, resource: T, createdAt: number }>
    currentIndex: number
    totalCreated: number
  }>({
    resources: [],
    currentIndex: 0,
    totalCreated: 0
  })
  
  const activeResourceRef = yield* ScopedRef.fromAcquire(
    createResource("pool-resource-0")
  )
  
  // Initialize pool
  yield* Effect.gen(function* () {
    const resource = yield* ScopedRef.get(activeResourceRef)
    yield* Ref.update(poolStateRef, state => ({
      ...state,
      resources: [{ id: "pool-resource-0", resource, createdAt: Date.now() }],
      totalCreated: 1
    }))
  })
  
  const rotateToNext = Effect.gen(function* () {
    const state = yield* Ref.get(poolStateRef)
    const nextIndex = (state.currentIndex + 1) % poolSize
    const nextId = `pool-resource-${state.totalCreated}`
    
    yield* Console.log(`Rotating to next resource: ${nextId}`)
    yield* ScopedRef.set(activeResourceRef, createResource(nextId))
    
    const newResource = yield* ScopedRef.get(activeResourceRef)
    yield* Ref.update(poolStateRef, state => ({
      currentIndex: nextIndex,
      totalCreated: state.totalCreated + 1,
      resources: state.resources.length < poolSize
        ? [...state.resources, { id: nextId, resource: newResource, createdAt: Date.now() }]
        : state.resources.map((r, i) => 
            i === nextIndex 
              ? { id: nextId, resource: newResource, createdAt: Date.now() }
              : r
          )
    }))
  })
  
  const getCurrentResource = ScopedRef.get(activeResourceRef)
  
  const getPoolStats = Effect.gen(function* () {
    const state = yield* Ref.get(poolStateRef)
    return {
      poolSize: state.resources.length,
      currentIndex: state.currentIndex,
      totalCreated: state.totalCreated,
      resourceAges: state.resources.map(r => ({
        id: r.id,
        age: Date.now() - r.createdAt
      }))
    }
  })
  
  const rotateIfNeeded = (maxAge: Duration.Duration) => Effect.gen(function* () {
    const state = yield* Ref.get(poolStateRef)
    const currentResource = state.resources[state.currentIndex]
    
    if (currentResource && (Date.now() - currentResource.createdAt) > Duration.toMillis(maxAge)) {
      yield* Console.log(`Resource ${currentResource.id} is too old, rotating...`)
      yield* rotateToNext
    }
  })
  
  return { 
    rotateToNext, 
    getCurrentResource, 
    getPoolStats, 
    rotateIfNeeded 
  } as const
})

// Usage example: Database connection pool
const databasePoolExample = Effect.gen(function* () {
  const createDbConnection = (id: string) =>
    Effect.acquireRelease(
      Effect.gen(function* () {
        yield* Console.log(`Creating database connection: ${id}`)
        yield* Effect.sleep(Duration.millis(100)) // Simulate connection time
        
        return {
          id,
          host: "database.example.com",
          connected: true,
          query: (sql: string) => Effect.succeed(`Result from ${id}: ${sql}`),
          createdAt: Date.now()
        }
      }),
      (conn) => Console.log(`Closing database connection: ${conn.id}`)
    )
  
  const dbPool = yield* createResourcePool(createDbConnection, 3)
  
  // Use pool
  yield* Console.log("=== Using Database Pool ===")
  const conn1 = yield* dbPool.getCurrentResource()
  const result1 = yield* conn1.query("SELECT * FROM users")
  yield* Console.log(`Query result: ${result1}`)
  
  let stats = yield* dbPool.getPoolStats()
  yield* Console.log(`Pool stats: ${JSON.stringify(stats, null, 2)}`)
  
  // Rotate connections
  yield* Console.log("\n=== Rotating Connections ===")
  yield* dbPool.rotateToNext()
  
  const conn2 = yield* dbPool.getCurrentResource()
  const result2 = yield* conn2.query("SELECT * FROM orders")
  yield* Console.log(`Query result: ${result2}`)
  
  stats = yield* dbPool.getPoolStats()
  yield* Console.log(`Pool stats after rotation: ${JSON.stringify(stats, null, 2)}`)
  
  // Age-based rotation
  yield* Console.log("\n=== Age-Based Rotation ===")
  yield* Effect.sleep(Duration.millis(200))
  yield* dbPool.rotateIfNeeded(Duration.millis(150))
  
  stats = yield* dbPool.getPoolStats()
  yield* Console.log(`Final pool stats: ${JSON.stringify(stats, null, 2)}`)
  
  return stats
})

Effect.runPromise(Effect.scoped(databasePoolExample))
```

### Pattern 2: Conditional Resource Management

Managing resources that should only exist under certain conditions, with automatic cleanup when conditions change.

```typescript
import { Effect, ScopedRef, Console, Ref, Option, Duration } from "effect"

// Conditional resource manager
const createConditionalResourceManager = <T, C>(
  condition: C,
  shouldCreateResource: (current: C, previous: Option.Option<C>) => boolean,
  createResource: (condition: C) => Effect.Effect<T, Error, Scope.Scope>
) => Effect.gen(function* () {
  const conditionRef = yield* Ref.make<C>(condition)
  const resourceRef = yield* Ref.make<Option.Option<ScopedRef<T>>>(Option.none())
  
  // Initialize if condition is met
  const initialize = Effect.gen(function* () {
    if (shouldCreateResource(condition, Option.none())) {
      const scopedRef = yield* ScopedRef.fromAcquire(createResource(condition))
      yield* Ref.set(resourceRef, Option.some(scopedRef))
    }
  })
  
  const updateCondition = (newCondition: C) => Effect.gen(function* () {
    const previousCondition = yield* Ref.get(conditionRef)
    const currentResource = yield* Ref.get(resourceRef)
    
    yield* Ref.set(conditionRef, newCondition)
    
    const shouldCreate = shouldCreateResource(newCondition, Option.some(previousCondition))
    
    if (shouldCreate && Option.isNone(currentResource)) {
      // Create new resource
      yield* Console.log("Condition met - creating resource")
      const scopedRef = yield* ScopedRef.fromAcquire(createResource(newCondition))
      yield* Ref.set(resourceRef, Option.some(scopedRef))
      
    } else if (!shouldCreate && Option.isSome(currentResource)) {
      // Remove existing resource - it will be cleaned up by scope
      yield* Console.log("Condition no longer met - resource will be cleaned up")
      yield* Ref.set(resourceRef, Option.none())
      
    } else if (shouldCreate && Option.isSome(currentResource)) {
      // Update existing resource
      yield* Console.log("Updating existing resource for new condition")
      yield* ScopedRef.set(currentResource.value, createResource(newCondition))
    }
  })
  
  const getCurrentResource = Effect.gen(function* () {
    const resourceOpt = yield* Ref.get(resourceRef)
    return yield* Option.match(resourceOpt, {
      onNone: () => Effect.fail(new Error("No resource available")),
      onSome: (scopedRef) => ScopedRef.get(scopedRef)
    })
  })
  
  const getCurrentCondition = Ref.get(conditionRef)
  
  const hasResource = Effect.gen(function* () {
    const resourceOpt = yield* Ref.get(resourceRef)
    return Option.isSome(resourceOpt)
  })
  
  // Initialize
  yield* initialize
  
  return { updateCondition, getCurrentResource, getCurrentCondition, hasResource } as const
})

// Example: Feature flag controlled service
interface FeatureFlags {
  enableAdvancedLogging: boolean
  enableMetrics: boolean
  enableCaching: boolean
}

const createLoggingService = (flags: FeatureFlags) =>
  Effect.acquireRelease(
    Effect.gen(function* () {
      yield* Console.log(`Creating logging service with flags: ${JSON.stringify(flags)}`)
      
      return {
        level: flags.enableAdvancedLogging ? "debug" : "info",
        withMetrics: flags.enableMetrics,
        withCaching: flags.enableCaching,
        log: (message: string, level: "debug" | "info" | "warn" | "error" = "info") => 
          Effect.gen(function* () {
            const shouldLog = level !== "debug" || flags.enableAdvancedLogging
            if (shouldLog) {
              const prefix = flags.enableMetrics ? `[${Date.now()}]` : ""
              yield* Console.log(`${prefix} [${level.toUpperCase()}] ${message}`)
            }
          })
      }
    }),
    () => Console.log("Shutting down logging service")
  )

// Feature flag demo
const featureFlagDemo = Effect.gen(function* () {
  const shouldHaveLogging = (current: FeatureFlags, previous: Option.Option<FeatureFlags>) =>
    current.enableAdvancedLogging || current.enableMetrics
  
  const logManager = yield* createConditionalResourceManager(
    { enableAdvancedLogging: false, enableMetrics: false, enableCaching: true },
    shouldHaveLogging,
    createLoggingService
  )
  
  yield* Console.log("=== Initial State (No Logging) ===")
  const hasLogging1 = yield* logManager.hasResource()
  yield* Console.log(`Has logging service: ${hasLogging1}`)
  
  yield* Console.log("\n=== Enable Advanced Logging ===")
  yield* logManager.updateCondition({
    enableAdvancedLogging: true,
    enableMetrics: false,
    enableCaching: true
  })
  
  const hasLogging2 = yield* logManager.hasResource()
  yield* Console.log(`Has logging service: ${hasLogging2}`)
  
  if (hasLogging2) {
    const logger = yield* logManager.getCurrentResource()
    yield* logger.log("This is a debug message", "debug")
    yield* logger.log("This is an info message", "info")
  }
  
  yield* Console.log("\n=== Enable Metrics (Update Logging) ===")
  yield* logManager.updateCondition({
    enableAdvancedLogging: true,
    enableMetrics: true,
    enableCaching: false
  })
  
  if (yield* logManager.hasResource()) {
    const logger = yield* logManager.getCurrentResource()
    yield* logger.log("Now with metrics!", "info")
  }
  
  yield* Console.log("\n=== Disable All Features (Remove Logging) ===")
  yield* logManager.updateCondition({
    enableAdvancedLogging: false,
    enableMetrics: false,
    enableCaching: false
  })
  
  const hasLogging3 = yield* logManager.hasResource()
  yield* Console.log(`Has logging service: ${hasLogging3}`)
  
  return { finalHasLogging: hasLogging3 }
})

Effect.runPromise(Effect.scoped(featureFlagDemo))
```

### Pattern 3: Resource Versioning and Migration

Managing versioned resources with automatic migration and backward compatibility.

```typescript
import { Effect, ScopedRef, Console, Ref, Array as Arr } from "effect"

// Versioned resource system
interface ResourceVersion {
  version: number
  data: any
  createdAt: number
  migrations: Array<string>
}

interface MigrationPlan {
  fromVersion: number
  toVersion: number
  migrate: (data: any) => Effect.Effect<any, Error>
  description: string
}

const createVersionedResourceManager = <T>(
  initialData: T,
  initialVersion: number,
  migrations: Array<MigrationPlan>
) => Effect.gen(function* () {
  
  const createVersionedResource = (data: T, version: number, appliedMigrations: Array<string> = []) =>
    Effect.acquireRelease(
      Effect.gen(function* () {
        yield* Console.log(`Creating resource version: ${version}`)
        
        return {
          version,
          data,
          createdAt: Date.now(),
          migrations: appliedMigrations,
          getData: () => data,
          getVersion: () => version,
          getMigrationHistory: () => appliedMigrations
        } as ResourceVersion & { getData: () => T, getVersion: () => number, getMigrationHistory: () => Array<string> }
      }),
      (resource) => Console.log(`Releasing resource version: ${resource.version}`)
    )
  
  const resourceRef = yield* ScopedRef.fromAcquire(
    createVersionedResource(initialData, initialVersion)
  )
  
  const migrateToVersion = (targetVersion: number) => Effect.gen(function* () {
    const currentResource = yield* ScopedRef.get(resourceRef)
    const currentVersion = currentResource.getVersion()
    
    if (currentVersion === targetVersion) {
      yield* Console.log(`Already at version ${targetVersion}`)
      return
    }
    
    // Find migration path
    const migrationPath = findMigrationPath(currentVersion, targetVersion, migrations)
    if (migrationPath.length === 0) {
      yield* Effect.fail(new Error(`No migration path from version ${currentVersion} to ${targetVersion}`))
    }
    
    yield* Console.log(`Migration path: ${migrationPath.map(m => `${m.fromVersion}->${m.toVersion}`).join(", ")}`)
    
    // Apply migrations sequentially
    let currentData = currentResource.getData()
    const appliedMigrations = [...currentResource.getMigrationHistory()]
    
    for (const migration of migrationPath) {
      yield* Console.log(`Applying migration: ${migration.description}`)
      currentData = yield* migration.migrate(currentData)
      appliedMigrations.push(`${migration.fromVersion}->${migration.toVersion}: ${migration.description}`)
    }
    
    // Update resource with migrated data
    yield* ScopedRef.set(resourceRef, 
      createVersionedResource(currentData, targetVersion, appliedMigrations)
    )
    
    yield* Console.log(`Successfully migrated to version ${targetVersion}`)
  })
  
  const getCurrentVersion = Effect.gen(function* () {
    const resource = yield* ScopedRef.get(resourceRef)
    return resource.getVersion()
  })
  
  const getCurrentData = Effect.gen(function* () {
    const resource = yield* ScopedRef.get(resourceRef)
    return resource.getData()
  })
  
  const getMigrationHistory = Effect.gen(function* () {
    const resource = yield* ScopedRef.get(resourceRef)
    return resource.getMigrationHistory()
  })
  
  const getResourceInfo = Effect.gen(function* () {
    const resource = yield* ScopedRef.get(resourceRef)
    return {
      version: resource.getVersion(),
      createdAt: resource.createdAt,
      age: Date.now() - resource.createdAt,
      migrationCount: resource.getMigrationHistory().length
    }
  })
  
  return { 
    migrateToVersion, 
    getCurrentVersion, 
    getCurrentData, 
    getMigrationHistory, 
    getResourceInfo 
  } as const
})

// Helper function to find migration path
const findMigrationPath = (
  fromVersion: number, 
  toVersion: number, 
  migrations: Array<MigrationPlan>
): Array<MigrationPlan> => {
  if (fromVersion === toVersion) return []
  
  // Simple path finding - in practice you might want Dijkstra's algorithm
  const path: Array<MigrationPlan> = []
  let currentVersion = fromVersion
  
  while (currentVersion !== toVersion) {
    const nextMigration = migrations.find(m => m.fromVersion === currentVersion)
    if (!nextMigration) {
      return [] // No path found
    }
    
    path.push(nextMigration)
    currentVersion = nextMigration.toVersion
    
    // Prevent infinite loops
    if (path.length > 10) return []
  }
  
  return path
}

// Example: User profile schema migration
interface UserProfileV1 {
  name: string
  email: string
}

interface UserProfileV2 {
  firstName: string
  lastName: string
  email: string
}

interface UserProfileV3 {
  firstName: string
  lastName: string
  email: string
  preferences: {
    theme: "light" | "dark"
    notifications: boolean
  }
}

const userProfileMigrationDemo = Effect.gen(function* () {
  // Define migrations
  const migrations: Array<MigrationPlan> = [
    {
      fromVersion: 1,
      toVersion: 2,
      description: "Split name into firstName and lastName",
      migrate: (data: UserProfileV1) => Effect.gen(function* () {
        yield* Console.log(`Migrating v1 to v2: splitting "${data.name}"`)
        const [firstName, ...lastNameParts] = data.name.split(" ")
        const lastName = lastNameParts.join(" ") || ""
        
        return {
          firstName,
          lastName,
          email: data.email
        } as UserProfileV2
      })
    },
    {
      fromVersion: 2,
      toVersion: 3,
      description: "Add user preferences with defaults",
      migrate: (data: UserProfileV2) => Effect.gen(function* () {
        yield* Console.log("Migrating v2 to v3: adding preferences")
        
        return {
          ...data,
          preferences: {
            theme: "light" as const,
            notifications: true
          }
        } as UserProfileV3
      })
    }
  ]
  
  // Start with v1 data
  const initialProfile: UserProfileV1 = {
    name: "John Doe",
    email: "john.doe@example.com"
  }
  
  const profileManager = yield* createVersionedResourceManager(initialProfile, 1, migrations)
  
  yield* Console.log("=== Initial Profile (v1) ===")
  let data = yield* profileManager.getCurrentData()
  let version = yield* profileManager.getCurrentVersion()
  yield* Console.log(`Version ${version}: ${JSON.stringify(data)}`)
  
  yield* Console.log("\n=== Migrate to v2 ===")
  yield* profileManager.migrateToVersion(2)
  
  data = yield* profileManager.getCurrentData()
  version = yield* profileManager.getCurrentVersion()
  yield* Console.log(`Version ${version}: ${JSON.stringify(data)}`)
  
  yield* Console.log("\n=== Migrate to v3 ===")
  yield* profileManager.migrateToVersion(3)
  
  data = yield* profileManager.getCurrentData()
  version = yield* profileManager.getCurrentVersion()
  yield* Console.log(`Version ${version}: ${JSON.stringify(data)}`)
  
  yield* Console.log("\n=== Migration History ===")
  const history = yield* profileManager.getMigrationHistory()
  history.forEach((migration, index) => {
    Console.log(`${index + 1}. ${migration}`)
  })
  
  const info = yield* profileManager.getResourceInfo()
  yield* Console.log(`\nResource info: ${JSON.stringify(info)}`)
  
  return { finalData: data, migrationHistory: history }
})

Effect.runPromise(Effect.scoped(userProfileMigrationDemo))
```

## Integration Examples

### Integration with HTTP Server Framework

Integrating ScopedRef with an HTTP server to manage request-scoped resources like database transactions and user sessions.

```typescript
import { Effect, ScopedRef, Console, Ref, Layer, Context, Duration } from "effect"

// HTTP Request context
interface HttpRequest {
  method: string
  url: string
  headers: Record<string, string>
  body?: any
  userId?: string
}

interface HttpResponse {
  status: number
  headers: Record<string, string>
  body: any
}

// Database transaction service
interface DatabaseTransaction {
  id: string
  queries: Array<string>
  commit: () => Effect.Effect<void, Error>
  rollback: () => Effect.Effect<void, Error>
  query: (sql: string) => Effect.Effect<any[], Error>
}

class DatabaseTransactionService extends Context.Tag("DatabaseTransactionService")<
  DatabaseTransactionService,
  {
    createTransaction: (userId?: string) => Effect.Effect<DatabaseTransaction, Error, Scope.Scope>
  }
>() {
  static Live = Layer.succeed(this, {
    createTransaction: (userId?: string) => 
      Effect.acquireRelease(
        Effect.gen(function* () {
          const txId = `tx_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`
          yield* Console.log(`Creating database transaction: ${txId} for user: ${userId ?? "anonymous"}`)
          
          const queriesRef = yield* Ref.make<Array<string>>([])
          
          return {
            id: txId,
            queries: [],
            commit: () => 
              Effect.gen(function* () {
                const queries = yield* Ref.get(queriesRef)
                yield* Console.log(`Committing transaction ${txId} with ${queries.length} queries`)
                // Simulate commit
                yield* Effect.sleep(Duration.millis(50))
              }),
            rollback: () => 
              Effect.gen(function* () {
                const queries = yield* Ref.get(queriesRef)
                yield* Console.log(`Rolling back transaction ${txId} (had ${queries.length} queries)`)
                // Simulate rollback
                yield* Effect.sleep(Duration.millis(30))
              }),
            query: (sql: string) => 
              Effect.gen(function* () {
                yield* Ref.update(queriesRef, queries => [...queries, sql])
                yield* Console.log(`Query in ${txId}: ${sql}`)
                // Simulate query execution
                yield* Effect.sleep(Duration.millis(20))
                return [{ result: `Result for: ${sql}` }]
              })
          } as DatabaseTransaction
        }),
        (tx) => 
          Console.log(`Cleaning up transaction: ${tx.id}`).pipe(
            Effect.zipRight(tx.rollback())
          )
      )
  })
}

// Request handler service
const createRequestHandler = Effect.gen(function* () {
  const dbService = yield* DatabaseTransactionService
  
  // Request-scoped transaction reference
  const createTransactionRef = (request: HttpRequest) =>
    ScopedRef.fromAcquire(dbService.createTransaction(request.userId))
  
  const handleRequest = (request: HttpRequest): Effect.Effect<HttpResponse, Error, Scope.Scope> =>
    Effect.gen(function* () {
      yield* Console.log(`\n=== Handling ${request.method} ${request.url} ===`)
      
      // Create request-scoped transaction
      const txRef = yield* createTransactionRef(request)
      
      try {
        // Route the request
        const response = yield* routeRequest(request, txRef)
        
        // Commit transaction on success
        const tx = yield* ScopedRef.get(txRef)
        yield* tx.commit()
        
        return response
      } catch (error) {
        // Transaction will be rolled back automatically by ScopedRef cleanup
        yield* Console.log(`Request failed: ${error}`)
        throw error
      }
    })
  
  const routeRequest = (
    request: HttpRequest, 
    txRef: ScopedRef<DatabaseTransaction>
  ): Effect.Effect<HttpResponse, Error> =>
    Effect.gen(function* () {
      const tx = yield* ScopedRef.get(txRef)
      
      switch (`${request.method} ${request.url}`) {
        case "GET /users": {
          const users = yield* tx.query("SELECT * FROM users")
          return {
            status: 200,
            headers: { "Content-Type": "application/json" },
            body: users
          }
        }
        
        case "POST /users": {
          const newUser = request.body
          yield* tx.query(`INSERT INTO users (name, email) VALUES ('${newUser.name}', '${newUser.email}')`)
          const result = yield* tx.query(`SELECT * FROM users WHERE email = '${newUser.email}'`)
          
          return {
            status: 201,
            headers: { "Content-Type": "application/json" },
            body: result[0]
          }
        }
        
        case "DELETE /users": {
          // This operation might fail, causing rollback
          if (request.userId !== "admin") {
            yield* Effect.fail(new Error("Unauthorized: Only admin can delete users"))
          }
          
          yield* tx.query("DELETE FROM users WHERE active = false")
          
          return {
            status: 204,
            headers: {},
            body: null
          }
        }
        
        default: {
          return {
            status: 404,
            headers: { "Content-Type": "application/json" },
            body: { error: "Not found" }
          }
        }
      }
    })
  
  return { handleRequest } as const
})

// HTTP server simulation
const httpServerDemo = Effect.gen(function* () {
  const requestHandler = yield* createRequestHandler
  
  // Simulate incoming requests
  const requests: Array<HttpRequest> = [
    {
      method: "GET",
      url: "/users",
      headers: { "Accept": "application/json" },
      userId: "user123"
    },
    {
      method: "POST",
      url: "/users", 
      headers: { "Content-Type": "application/json" },
      body: { name: "Alice Smith", email: "alice@example.com" },
      userId: "user123"
    },
    {
      method: "DELETE",
      url: "/users",
      headers: {},
      userId: "user123" // Not admin - will fail
    },
    {
      method: "DELETE",
      url: "/users",
      headers: {},
      userId: "admin" // Admin - will succeed
    }
  ]
  
  // Handle each request in its own scope
  const responses = yield* Effect.forEach(requests, (request) =>
    Effect.gen(function* () {
      const result = yield* Effect.either(
        Effect.scoped(requestHandler.handleRequest(request))
      )
      
      if (result._tag === "Left") {
        yield* Console.log(`Request failed: ${result.left.message}`)
        return {
          status: 500,
          headers: { "Content-Type": "application/json" },
          body: { error: result.left.message }
        }
      }
      
      yield* Console.log(`Request completed: ${result.right.status}`)
      return result.right
    })
  )
  
  yield* Console.log("\n=== Request Summary ===")
  responses.forEach((response, index) => {
    const request = requests[index]
    Console.log(`${request.method} ${request.url} -> ${response.status}`)
  })
  
  return responses
})

// Run the HTTP server demo
const main = httpServerDemo.pipe(
  Effect.provide(DatabaseTransactionService.Live)
)

Effect.runPromise(main)
```

### Integration with Testing Framework

Using ScopedRef in testing scenarios to manage test fixtures and ensure proper cleanup between tests.

```typescript
import { Effect, ScopedRef, Console, Ref, Array as Arr, Duration } from "effect"

// Test fixture types
interface TestDatabase {
  name: string
  tables: Map<string, Array<any>>
  seed: (data: Record<string, Array<any>>) => Effect.Effect<void>
  query: (table: string) => Effect.Effect<Array<any>>
  cleanup: () => Effect.Effect<void>
}

interface TestEnvironment {
  database: TestDatabase
  tempFiles: Array<string>
  httpServer: { port: number, running: boolean }
}

// Test fixture manager
const createTestFixtureManager = Effect.gen(function* () {
  
  const createTestDatabase = (name: string) =>
    Effect.acquireRelease(
      Effect.gen(function* () {
        yield* Console.log(`Setting up test database: ${name}`)
        const tables = new Map<string, Array<any>>()
        
        return {
          name,
          tables,
          seed: (data: Record<string, Array<any>>) => 
            Effect.gen(function* () {
              yield* Console.log(`Seeding database ${name} with data`)
              Object.entries(data).forEach(([table, rows]) => {
                tables.set(table, rows)
              })
            }),
          query: (table: string) =>
            Effect.succeed(tables.get(table) || []),
          cleanup: () => 
            Effect.gen(function* () {
              yield* Console.log(`Cleaning database ${name}`)
              tables.clear()
            })
        } as TestDatabase
      }),
      (db) => 
        Console.log(`Tearing down test database: ${db.name}`).pipe(
          Effect.zipRight(db.cleanup())
        )
    )
  
  const createTestEnvironment = (testName: string) =>
    Effect.acquireRelease(
      Effect.gen(function* () {
        yield* Console.log(`Creating test environment for: ${testName}`)
        
        const database = yield* createTestDatabase(`test_db_${testName.replace(/\s+/g, '_')}`)
        const tempFiles: Array<string> = []
        
        // Simulate HTTP server setup
        const port = 3000 + Math.floor(Math.random() * 1000)
        yield* Console.log(`Starting test HTTP server on port ${port}`)
        yield* Effect.sleep(Duration.millis(100)) // Simulate startup time
        
        return {
          database,
          tempFiles,
          httpServer: { port, running: true }
        } as TestEnvironment
      }),
      (env) =>
        Console.log("Tearing down test environment").pipe(
          Effect.zipRight(
            Effect.gen(function* () {
              // Clean up temp files
              if (env.tempFiles.length > 0) {
                yield* Console.log(`Cleaning up ${env.tempFiles.length} temp files`)
              }
              
              // Stop HTTP server
              if (env.httpServer.running) {
                yield* Console.log(`Stopping HTTP server on port ${env.httpServer.port}`)
              }
            })
          )
        )
    )
  
  // Test environment reference
  const envRef = yield* ScopedRef.fromAcquire(createTestEnvironment("initial"))
  
  const setupTestEnvironment = (testName: string, seedData?: Record<string, Array<any>>) =>
    Effect.gen(function* () {
      yield* ScopedRef.set(envRef, createTestEnvironment(testName))
      
      if (seedData) {
        const env = yield* ScopedRef.get(envRef)
        yield* env.database.seed(seedData)
      }
    })
  
  const getCurrentEnvironment = ScopedRef.get(envRef)
  
  const addTempFile = (filename: string) => Effect.gen(function* () {
    const env = yield* getCurrentEnvironment
    env.tempFiles.push(filename)
    yield* Console.log(`Added temp file: ${filename}`)
  })
  
  const queryTestData = (table: string) => Effect.gen(function* () {
    const env = yield* getCurrentEnvironment
    return yield* env.database.query(table)
  })
  
  return { 
    setupTestEnvironment, 
    getCurrentEnvironment, 
    addTempFile, 
    queryTestData 
  } as const
})

// Test suite simulation
const testSuiteDemo = Effect.gen(function* () {
  const fixtureManager = yield* createTestFixtureManager
  
  // Test 1: User CRUD operations
  yield* Console.log("\n=== Test 1: User CRUD Operations ===")
  yield* fixtureManager.setupTestEnvironment("user_crud_test", {
    users: [
      { id: 1, name: "John Doe", email: "john@example.com" },
      { id: 2, name: "Jane Smith", email: "jane@example.com" }
    ]
  })
  
  let users = yield* fixtureManager.queryTestData("users")
  yield* Console.log(`Initial users: ${users.length}`)
  
  yield* fixtureManager.addTempFile("user_export.json")
  yield* fixtureManager.addTempFile("user_backup.sql")
  
  // Test 2: Order processing (different environment)
  yield* Console.log("\n=== Test 2: Order Processing ===")
  yield* fixtureManager.setupTestEnvironment("order_processing_test", {
    orders: [
      { id: 1, userId: 1, total: 99.99, status: "pending" },
      { id: 2, userId: 2, total: 149.50, status: "completed" }
    ],
    products: [
      { id: 1, name: "Widget", price: 49.99 },
      { id: 2, name: "Gadget", price: 99.99 }
    ]
  })
  
  const orders = yield* fixtureManager.queryTestData("orders")
  const products = yield* fixtureManager.queryTestData("products")
  yield* Console.log(`Orders: ${orders.length}, Products: ${products.length}`)
  
  yield* fixtureManager.addTempFile("order_report.pdf")
  
  // Test 3: Integration test (new environment again)
  yield* Console.log("\n=== Test 3: Integration Test ===")
  yield* fixtureManager.setupTestEnvironment("integration_test", {
    users: [{ id: 1, name: "Test User", email: "test@example.com" }],
    orders: [{ id: 1, userId: 1, total: 25.00, status: "pending" }],
    audit_log: []
  })
  
  const env = yield* fixtureManager.getCurrentEnvironment()
  yield* Console.log(`Test environment database: ${env.database.name}`)
  yield* Console.log(`Test HTTP server port: ${env.httpServer.port}`)
  
  const auditLog = yield* fixtureManager.queryTestData("audit_log")
  yield* Console.log(`Audit log entries: ${auditLog.length}`)
  
  yield* fixtureManager.addTempFile("integration_test_results.xml")
  yield* fixtureManager.addTempFile("coverage_report.html")
  
  return { 
    testsCompleted: 3,
    finalEnvironment: env.database.name
  }
})

// Run test suite - each test gets clean environment
const main = Effect.scoped(testSuiteDemo)

Effect.runPromise(main)
```

## Conclusion

ScopedRef provides automatic resource lifecycle management for stateful references, ensuring proper cleanup and preventing memory leaks in complex applications.

Key benefits:
- **Automatic Cleanup**: Resources are automatically released when ScopedRef values change or scopes close
- **Atomic Transitions**: Resource swapping is atomic, preventing race conditions and resource conflicts  
- **Memory Safety**: Eliminates manual cleanup burden and prevents resource leaks
- **Scope Integration**: Works seamlessly with Effect's Scope system for structured resource management
- **Type Safety**: Provides compile-time guarantees about resource lifecycle management

ScopedRef is ideal for managing database connections, HTTP clients, file handles, WebSocket connections, and any other resources that require proper acquisition and release. It shines in scenarios where resources need to be dynamically swapped based on configuration changes, feature flags, or runtime conditions while ensuring no resources are leaked in the process.