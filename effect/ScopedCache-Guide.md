# ScopedCache: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem ScopedCache Solves

In long-running applications, resources cached without proper lifecycle management lead to memory leaks, resource exhaustion, and difficult-to-track resource ownership. Traditional caches lack automatic cleanup tied to application scopes.

```typescript
// Traditional approach - problematic resource management
class ResourceCache {
  private cache = new Map<string, DatabaseConnection>()
  
  async getConnection(dbUrl: string): Promise<DatabaseConnection> {
    if (!this.cache.has(dbUrl)) {
      const connection = await createDatabaseConnection(dbUrl)
      this.cache.set(dbUrl, connection)
      return connection
    }
    
    return this.cache.get(dbUrl)!
  }
  
  // Manual cleanup - easy to forget or miss edge cases
  cleanup() {
    for (const [url, connection] of this.cache) {
      connection.close() // What if this throws?
      this.cache.delete(url)
    }
  }
}
```

This approach leads to:
- **Resource Leaks** - No automatic cleanup when scopes close or applications shut down
- **Manual Lifecycle Management** - Error-prone cleanup logic scattered throughout code
- **No Scope Integration** - Resources don't participate in Effect's structured concurrency model
- **Memory Bloat** - Resources accumulate without automatic eviction tied to usage scopes
- **Difficult Debugging** - Hard to track resource ownership and cleanup order

### The ScopedCache Solution

Effect's ScopedCache provides automatic resource lifecycle management tied to scopes, ensuring resources are cleaned up when scopes close, preventing memory leaks in long-running applications.

```typescript
import { ScopedCache, Effect, Duration, Scope } from "effect"

// Declarative scoped cache with automatic resource cleanup
const databaseCache = ScopedCache.make({
  capacity: 100,
  timeToLive: Duration.minutes(30),
  lookup: (dbUrl: string) => Effect.gen(function* () {
    // Resource is created within a managed scope
    const connection = yield* createDatabaseConnection(dbUrl)
    
    // Automatically cleaned up when cache entry expires or scope closes
    yield* Effect.addFinalizer(() => 
      Effect.promise(() => connection.close())
    )
    
    return connection
  })
})

const program = Effect.gen(function* () {
  const cache = yield* databaseCache
  
  // Resources automatically managed and cleaned up
  const connection = yield* cache.get("postgresql://localhost:5432/mydb")
  
  return yield* executeQuery(connection, "SELECT * FROM users")
}).pipe(
  Effect.scoped // Scope ensures all resources are cleaned up
)
```

### Key Concepts

**ScopedCache<Key, Value, Error>**: A cache that manages resources within scopes, automatically cleaning up cached values when their associated scopes close.

**Scoped Lookup Function**: A function that creates resources within a scope, allowing automatic cleanup through finalizers when the scope closes.

**Automatic Resource Management**: Resources are tied to scopes and automatically released when scopes close, preventing memory leaks.

**Scope Integration**: Seamlessly integrates with Effect's scope system for structured resource management.

**Finalizer-Based Cleanup**: Uses Effect's finalizer system to ensure resources are properly released even in error scenarios.

## Basic Usage Patterns

### Pattern 1: Simple Resource Caching

```typescript
import { ScopedCache, Effect, Duration } from "effect"

interface FileHandle {
  readonly path: string
  readonly descriptor: number
  readonly close: () => Effect.Effect<void>
}

// Cache file handles with automatic cleanup
const fileHandleCache = ScopedCache.make({
  capacity: 50,
  timeToLive: Duration.minutes(10),
  lookup: (filePath: string) => Effect.gen(function* () {
    console.log(`Opening file: ${filePath}`)
    
    // Simulate file opening
    const descriptor = Math.floor(Math.random() * 1000)
    const handle: FileHandle = {
      path: filePath,
      descriptor,
      close: () => Effect.sync(() => {
        console.log(`Closing file: ${filePath} (${descriptor})`)
      })
    }
    
    // Register cleanup finalizer
    yield* Effect.addFinalizer(() => handle.close())
    
    return handle
  })
})

const program = Effect.gen(function* () {
  const cache = yield* fileHandleCache
  
  // First access opens the file
  const handle1 = yield* cache.get("/tmp/data.json")
  console.log(`Using file: ${handle1.path}`)
  
  // Second access uses cached handle
  const handle2 = yield* cache.get("/tmp/data.json")
  console.log(`Reusing file: ${handle2.path}`)
  
  return "File operations completed"
}).pipe(
  Effect.scoped // Ensures all file handles are closed when scope exits
)
```

### Pattern 2: Database Connection Pool

```typescript
import { ScopedCache, Effect, Duration } from "effect"

interface DatabaseConnection {
  readonly url: string
  readonly id: string
  readonly query: <T>(sql: string) => Effect.Effect<T>
  readonly close: () => Effect.Effect<void>
}

const connectionCache = ScopedCache.make({
  capacity: 10, // Limit concurrent connections
  timeToLive: Duration.minutes(30),
  lookup: (connectionString: string) => Effect.gen(function* () {
    const connectionId = crypto.randomUUID()
    console.log(`Creating connection ${connectionId} to ${connectionString}`)
    
    // Simulate connection creation
    yield* Effect.sleep(Duration.milliseconds(100))
    
    const connection: DatabaseConnection = {
      url: connectionString,
      id: connectionId,
      query: <T>(sql: string) => Effect.gen(function* () {
        console.log(`Executing query on ${connectionId}: ${sql}`)
        return {} as T
      }),
      close: () => Effect.gen(function* () {
        console.log(`Closing connection ${connectionId}`)
        yield* Effect.sleep(Duration.milliseconds(50))
      })
    }
    
    // Ensure connection is closed when scope closes
    yield* Effect.addFinalizer(() => connection.close())
    
    return connection
  })
})

const queryUser = (userId: string) => Effect.gen(function* () {
  const cache = yield* connectionCache
  const connection = yield* cache.get("postgresql://localhost:5432/users")
  
  return yield* connection.query(`SELECT * FROM users WHERE id = '${userId}'`)
})
```

### Pattern 3: HTTP Client with Connection Reuse

```typescript
import { ScopedCache, Effect, Duration, HttpClient } from "effect"

interface HttpClientInstance {
  readonly baseUrl: string
  readonly client: HttpClient.HttpClient
  readonly close: () => Effect.Effect<void>
}

const httpClientCache = ScopedCache.make({
  capacity: 20,
  timeToLive: Duration.minutes(15),
  lookup: (baseUrl: string) => Effect.gen(function* () {
    console.log(`Creating HTTP client for: ${baseUrl}`)
    
    const client = HttpClient.client.make({
      // Configuration for connection reuse
      keepAlive: true,
      maxSockets: 10
    })
    
    const instance: HttpClientInstance = {
      baseUrl,
      client,
      close: () => Effect.sync(() => {
        console.log(`Closing HTTP client for: ${baseUrl}`)
        // Cleanup connections
      })
    }
    
    // Register cleanup
    yield* Effect.addFinalizer(() => instance.close())
    
    return instance
  })
})

const makeApiCall = (endpoint: string, data: unknown) => Effect.gen(function* () {
  const cache = yield* httpClientCache
  const clientInstance = yield* cache.get("https://api.example.com")
  
  return yield* HttpClient.request.post(`${endpoint}`).pipe(
    HttpClient.request.setBody(HttpClient.body.json(data)),
    clientInstance.client
  )
})
```

## Real-World Examples

### Example 1: Microservice Resource Manager

```typescript
import { ScopedCache, Effect, Duration, Layer, Context } from "effect"

interface ServiceConnection {
  readonly serviceName: string
  readonly endpoint: string
  readonly client: HttpClient.HttpClient
  readonly metrics: {
    readonly requests: number
    readonly errors: number
  }
  readonly close: () => Effect.Effect<void>
}

class ServiceDiscovery extends Context.Tag("ServiceDiscovery")<
  ServiceDiscovery,
  {
    readonly getEndpoint: (serviceName: string) => Effect.Effect<string, ServiceDiscoveryError>
  }
>() {}

class ServiceDiscoveryError extends Error {
  readonly _tag = "ServiceDiscoveryError"
  constructor(readonly serviceName: string) {
    super(`Failed to discover service: ${serviceName}`)
  }
}

class ConnectionError extends Error {
  readonly _tag = "ConnectionError"
  constructor(readonly serviceName: string, readonly cause: unknown) {
    super(`Failed to connect to service: ${serviceName}`)
  }
}

// Service connection cache with health checking
const serviceConnectionCache = Effect.gen(function* () {
  const serviceDiscovery = yield* ServiceDiscovery
  
  return yield* ScopedCache.make({
    capacity: 50,
    timeToLive: Duration.minutes(20),
    lookup: (serviceName: string) => Effect.gen(function* () {
      console.log(`Establishing connection to service: ${serviceName}`)
      
      // Discover service endpoint
      const endpoint = yield* serviceDiscovery.getEndpoint(serviceName).pipe(
        Effect.mapError(() => new ServiceDiscoveryError(serviceName))
      )
      
      // Create HTTP client with circuit breaker
      const client = yield* HttpClient.client.make({
        timeout: Duration.seconds(30),
        retry: {
          times: 3,
          delay: Duration.seconds(1)
        }
      }).pipe(
        Effect.mapError(cause => new ConnectionError(serviceName, cause))
      )
      
      // Health check
      yield* HttpClient.request.get(`${endpoint}/health`).pipe(
        client,
        Effect.timeout(Duration.seconds(5)),
        Effect.mapError(cause => new ConnectionError(serviceName, cause))
      )
      
      const connection: ServiceConnection = {
        serviceName,
        endpoint,
        client,
        metrics: { requests: 0, errors: 0 },
        close: () => Effect.gen(function* () {
          console.log(`Closing connection to service: ${serviceName}`)
          // Graceful shutdown logic
          yield* Effect.sleep(Duration.milliseconds(100))
        })
      }
      
      // Register cleanup finalizer
      yield* Effect.addFinalizer(() => connection.close())
      
      // Start health monitoring
      const healthCheck = Effect.gen(function* () {
        try {
          yield* HttpClient.request.get(`${endpoint}/health`).pipe(client)
          console.log(`Health check passed for ${serviceName}`)
        } catch {
          console.warn(`Health check failed for ${serviceName}`)
        }
      }).pipe(
        Effect.repeat(Schedule.fixed(Duration.seconds(30))),
        Effect.fork
      )
      
      yield* healthCheck
      
      return connection
    })
  })
})

// Business logic using the service cache
const callUserService = (userId: string) => Effect.gen(function* () {
  const cache = yield* serviceConnectionCache
  const connection = yield* cache.get("user-service")
  
  const response = yield* HttpClient.request.get(`/users/${userId}`).pipe(
    connection.client,
    Effect.mapError(error => new Error(`User service call failed: ${error}`))
  )
  
  return yield* HttpClient.response.json(response)
})

const callOrderService = (orderId: string) => Effect.gen(function* () {
  const cache = yield* serviceConnectionCache
  const connection = yield* cache.get("order-service")
  
  return yield* HttpClient.request.get(`/orders/${orderId}`).pipe(
    connection.client,
    HttpClient.response.json
  )
})

// Aggregate data from multiple services
const getUserWithOrders = (userId: string) => Effect.gen(function* () {
  const [user, orders] = yield* Effect.all([
    callUserService(userId),
    Effect.all([
      callOrderService("order-1"),
      callOrderService("order-2")
    ])
  ], { concurrency: "unbounded" })
  
  return { user, orders }
}).pipe(
  Effect.scoped // Ensures all service connections are cleaned up
)
```

### Example 2: Database Connection Pool with Transaction Support

```typescript
import { ScopedCache, Effect, Duration, Queue, Ref } from "effect"

interface DatabaseTransaction {
  readonly id: string
  readonly connection: DatabaseConnection
  readonly commit: () => Effect.Effect<void>
  readonly rollback: () => Effect.Effect<void>
}

interface DatabasePool {
  readonly url: string
  readonly activeConnections: number
  readonly maxConnections: number
  readonly getConnection: () => Effect.Effect<DatabaseConnection, PoolExhaustedException>
  readonly returnConnection: (conn: DatabaseConnection) => Effect.Effect<void>
  readonly close: () => Effect.Effect<void>
}

class PoolExhaustedException extends Error {
  readonly _tag = "PoolExhaustedException"
  constructor(readonly poolUrl: string) {
    super(`Database pool exhausted for: ${poolUrl}`)
  }
}

class TransactionError extends Error {
  readonly _tag = "TransactionError"
  constructor(readonly message: string, readonly cause?: unknown) {
    super(message)
  }
}

// Database pool cache with connection management
const databasePoolCache = ScopedCache.make({
  capacity: 5, // Max 5 different database pools
  timeToLive: Duration.hours(2),
  lookup: (connectionString: string) => Effect.gen(function* () {
    console.log(`Creating database pool for: ${connectionString}`)
    
    const maxConnections = 20
    const connectionQueue = yield* Queue.bounded<DatabaseConnection>(maxConnections)
    const activeCount = yield* Ref.make(0)
    
    // Pre-populate with initial connections
    for (let i = 0; i < 5; i++) {
      const connection = yield* createDatabaseConnection(connectionString)
      yield* Queue.offer(connectionQueue, connection)
    }
    
    const pool: DatabasePool = {
      url: connectionString,
      activeConnections: 0,
      maxConnections,
      getConnection: () => Effect.gen(function* () {
        const currentActive = yield* Ref.get(activeCount)
        
        // Try to get from pool first
        const maybeConnection = yield* Queue.poll(connectionQueue)
        
        if (maybeConnection._tag === "Some") {
          yield* Ref.update(activeCount, n => n + 1)
          return maybeConnection.value
        }
        
        // Create new connection if under limit
        if (currentActive < maxConnections) {
          const connection = yield* createDatabaseConnection(connectionString)
          yield* Ref.update(activeCount, n => n + 1)
          return connection
        }
        
        // Pool exhausted
        return yield* Effect.fail(new PoolExhaustedException(connectionString))
      }),
      
      returnConnection: (connection) => Effect.gen(function* () {
        yield* Ref.update(activeCount, n => Math.max(0, n - 1))
        yield* Queue.offer(connectionQueue, connection)
      }),
      
      close: () => Effect.gen(function* () {
        console.log(`Closing database pool for: ${connectionString}`)
        
        // Close all connections in queue
        let connection = yield* Queue.poll(connectionQueue)
        while (connection._tag === "Some") {
          yield* connection.value.close()
          connection = yield* Queue.poll(connectionQueue)
        }
      })
    }
    
    // Register pool cleanup
    yield* Effect.addFinalizer(() => pool.close())
    
    return pool
  })
})

// Transaction management with scoped cache
const transactionCache = ScopedCache.make({
  capacity: 100,
  timeToLive: Duration.minutes(5), // Transactions should not last long
  lookup: (transactionId: string) => Effect.gen(function* () {
    console.log(`Starting transaction: ${transactionId}`)
    
    const poolCache = yield* databasePoolCache
    const pool = yield* poolCache.get("postgresql://localhost:5432/myapp")
    const connection = yield* pool.getConnection()
    
    // Begin transaction
    yield* connection.query("BEGIN")
    
    const transaction: DatabaseTransaction = {
      id: transactionId,
      connection,
      commit: () => Effect.gen(function* () {
        console.log(`Committing transaction: ${transactionId}`)
        yield* connection.query("COMMIT")
        yield* pool.returnConnection(connection)
      }),
      rollback: () => Effect.gen(function* () {
        console.log(`Rolling back transaction: ${transactionId}`)
        yield* connection.query("ROLLBACK")
        yield* pool.returnConnection(connection)
      })
    }
    
    // Auto-rollback on scope close if not committed
    let committed = false
    yield* Effect.addFinalizer(() => 
      committed 
        ? Effect.void 
        : transaction.rollback()
    )
    
    return transaction
  })
})

// Business operations using transactions
const transferMoney = (fromUserId: string, toUserId: string, amount: number) =>
  Effect.gen(function* () {
    const transactionId = crypto.randomUUID()
    const cache = yield* transactionCache
    const transaction = yield* cache.get(transactionId)
    
    try {
      // Debit from source account
      yield* transaction.connection.query(
        "UPDATE accounts SET balance = balance - $1 WHERE user_id = $2",
        [amount, fromUserId]
      )
      
      // Credit to destination account
      yield* transaction.connection.query(
        "UPDATE accounts SET balance = balance + $1 WHERE user_id = $2",
        [amount, toUserId]
      )
      
      // Verify balances
      const sourceBalance = yield* transaction.connection.query(
        "SELECT balance FROM accounts WHERE user_id = $1",
        [fromUserId]
      )
      
      if (sourceBalance < 0) {
        return yield* Effect.fail(
          new TransactionError("Insufficient funds")
        )
      }
      
      // Commit transaction
      yield* transaction.commit()
      
      return { success: true, transactionId }
    } catch (error) {
      yield* transaction.rollback()
      return yield* Effect.fail(
        new TransactionError("Transfer failed", error)
      )
    }
  }).pipe(
    Effect.scoped // Ensures transaction cleanup
  )
```

### Example 3: WebSocket Connection Manager

```typescript
import { ScopedCache, Effect, Duration, Queue, Ref, Schedule } from "effect"

interface WebSocketConnection {
  readonly url: string
  readonly connectionId: string
  readonly send: (message: string) => Effect.Effect<void>
  readonly receive: () => Effect.Effect<string>
  readonly close: () => Effect.Effect<void>
  readonly isConnected: () => boolean
}

interface WebSocketManager {
  readonly url: string
  readonly connection: WebSocketConnection
  readonly messageQueue: Queue.Queue<string>
  readonly reconnectionCount: number
  readonly lastHeartbeat: number
}

class WebSocketError extends Error {
  readonly _tag = "WebSocketError"
  constructor(readonly url: string, readonly cause: unknown) {
    super(`WebSocket error for ${url}: ${cause}`)
  }
}

// WebSocket connection cache with automatic reconnection
const webSocketCache = ScopedCache.make({
  capacity: 10,
  timeToLive: Duration.hours(1),
  lookup: (wsUrl: string) => Effect.gen(function* () {
    console.log(`Creating WebSocket connection to: ${wsUrl}`)
    
    const connectionId = crypto.randomUUID()
    const messageQueue = yield* Queue.unbounded<string>()
    const reconnectionCount = yield* Ref.make(0)
    const lastHeartbeat = yield* Ref.make(Date.now())
    let websocket: WebSocket | null = null
    
    const createConnection = () => Effect.gen(function* () {
      websocket = new WebSocket(wsUrl)
      
      return yield* Effect.async<void, WebSocketError>((resume) => {
        websocket!.onopen = () => {
          console.log(`WebSocket connected: ${wsUrl}`)
          resume(Effect.void)
        }
        
        websocket!.onerror = (error) => {
          console.error(`WebSocket error: ${wsUrl}`, error)
          resume(Effect.fail(new WebSocketError(wsUrl, error)))
        }
        
        websocket!.onmessage = (event) => {
          Effect.runSync(Queue.offer(messageQueue, event.data))
          Effect.runSync(Ref.set(lastHeartbeat, Date.now()))
        }
        
        websocket!.onclose = () => {
          console.log(`WebSocket closed: ${wsUrl}`)
        }
      })
    })
    
    // Initial connection
    yield* createConnection().pipe(
      Effect.retry(Schedule.exponential(Duration.seconds(1)).pipe(
        Schedule.recurs(5)
      ))
    )
    
    const connection: WebSocketConnection = {
      url: wsUrl,
      connectionId,
      send: (message: string) => Effect.gen(function* () {
        if (!websocket || websocket.readyState !== WebSocket.OPEN) {
          return yield* Effect.fail(
            new WebSocketError(wsUrl, "Connection not open")
          )
        }
        
        websocket.send(message)
      }),
      receive: () => Queue.take(messageQueue),
      close: () => Effect.sync(() => {
        if (websocket) {
          websocket.close()
          websocket = null
        }
      }),
      isConnected: () => websocket?.readyState === WebSocket.OPEN
    }
    
    const manager: WebSocketManager = {
      url: wsUrl,
      connection,
      messageQueue,
      reconnectionCount: yield* Ref.get(reconnectionCount),
      lastHeartbeat: yield* Ref.get(lastHeartbeat)
    }
    
    // Heartbeat monitoring and reconnection
    const heartbeatMonitor = Effect.gen(function* () {
      const lastBeat = yield* Ref.get(lastHeartbeat)
      const now = Date.now()
      
      if (now - lastBeat > 30000 && connection.isConnected()) { // 30 seconds
        console.log(`Heartbeat timeout for ${wsUrl}, reconnecting...`)
        
        yield* connection.close()
        yield* createConnection()
        yield* Ref.update(reconnectionCount, n => n + 1)
      }
    }).pipe(
      Effect.repeat(Schedule.fixed(Duration.seconds(10))),
      Effect.fork
    )
    
    yield* heartbeatMonitor
    
    // Register cleanup
    yield* Effect.addFinalizer(() => connection.close())
    
    return manager
  })
})

// Real-time chat service using WebSocket cache
const chatService = Effect.gen(function* () {
  const cache = yield* webSocketCache
  
  const sendMessage = (roomId: string, message: string) => Effect.gen(function* () {
    const manager = yield* cache.get(`wss://chat.example.com/rooms/${roomId}`)
    
    const chatMessage = JSON.stringify({
      type: "message",
      content: message,
      timestamp: Date.now()
    })
    
    yield* manager.connection.send(chatMessage)
  })
  
  const listenForMessages = (roomId: string) => Effect.gen(function* () {
    const manager = yield* cache.get(`wss://chat.example.com/rooms/${roomId}`)
    
    const message = yield* manager.connection.receive()
    const parsed = JSON.parse(message)
    
    console.log(`Received message in room ${roomId}:`, parsed)
    return parsed
  })
  
  return { sendMessage, listenForMessages }
})

// Usage in application
const chatApplication = Effect.gen(function* () {
  const chat = yield* chatService
  
  // Send a message
  yield* chat.sendMessage("room-123", "Hello, world!")
  
  // Listen for incoming messages
  const message = yield* chat.listenForMessages("room-123")
  
  return message
}).pipe(
  Effect.scoped // Ensures WebSocket connections are properly closed
)
```

## Advanced Features Deep Dive

### Feature 1: Dynamic TTL and Resource-Aware Expiration

ScopedCache supports dynamic TTL based on the result of lookup operations, enabling sophisticated resource management strategies.

#### TTL Based on Resource Usage

```typescript
import { ScopedCache, Effect, Duration, Exit, Ref } from "effect"

interface ResourceUsageStats {
  readonly accessCount: number
  readonly lastAccessed: number
  readonly resourceSize: number
}

// Cache with usage-based TTL
const adaptiveTTLCache = ScopedCache.makeWith({
  capacity: 100,
  lookup: (resourceId: string) => Effect.gen(function* () {
    const stats = yield* Ref.make<ResourceUsageStats>({
      accessCount: 0,
      lastAccessed: Date.now(),
      resourceSize: Math.floor(Math.random() * 1000000) // Simulate resource size
    })
    
    const resource = {
      id: resourceId,
      data: `Resource data for ${resourceId}`,
      stats,
      access: () => Effect.gen(function* () {
        yield* Ref.update(stats, current => ({
          ...current,
          accessCount: current.accessCount + 1,
          lastAccessed: Date.now()
        }))
      })
    }
    
    // Register cleanup
    yield* Effect.addFinalizer(() => Effect.gen(function* () {
      const finalStats = yield* Ref.get(stats)
      console.log(`Cleaning up resource ${resourceId}:`, finalStats)
    }))
    
    return resource
  }),
  timeToLive: (exit) => {
    if (Exit.isFailure(exit)) {
      return Duration.seconds(30) // Quick retry on failures
    }
    
    const resource = exit.value
    
    // Access resource stats for TTL calculation
    return Effect.gen(function* () {
      const stats = yield* Ref.get(resource.stats)
      
      // High-usage resources get longer TTL
      if (stats.accessCount > 10) {
        return Duration.hours(2)
      }
      
      // Large resources get shorter TTL to manage memory
      if (stats.resourceSize > 500000) {
        return Duration.minutes(10)
      }
      
      // Default TTL
      return Duration.minutes(30)
    }).pipe(Effect.runSync)
  }
})
```

#### Resource Health-Based Expiration

```typescript
import { ScopedCache, Effect, Duration, Exit, Schedule } from "effect"

interface HealthyResource {
  readonly id: string
  readonly endpoint: string
  readonly isHealthy: () => Effect.Effect<boolean>
  readonly getLatency: () => Effect.Effect<number>
  readonly close: () => Effect.Effect<void>
}

// Cache with health monitoring and adaptive TTL
const healthAwareCache = ScopedCache.makeWith({
  capacity: 50,
  lookup: (serviceEndpoint: string) => Effect.gen(function* () {
    console.log(`Creating monitored connection to: ${serviceEndpoint}`)
    
    let healthy = true
    let latency = 0
    
    const resource: HealthyResource = {
      id: crypto.randomUUID(),
      endpoint: serviceEndpoint,
      isHealthy: () => Effect.succeed(healthy),
      getLatency: () => Effect.succeed(latency),
      close: () => Effect.sync(() => {
        console.log(`Closing connection to ${serviceEndpoint}`)
      })
    }
    
    // Start health monitoring
    const healthMonitor = Effect.gen(function* () {
      const start = Date.now()
      
      try {
        // Simulate health check
        yield* Effect.sleep(Duration.milliseconds(Math.random() * 100))
        healthy = Math.random() > 0.1 // 90% healthy
        latency = Date.now() - start
      } catch {
        healthy = false
        latency = 5000 // High latency indicates problems
      }
    }).pipe(
      Effect.repeat(Schedule.fixed(Duration.seconds(30))),
      Effect.fork
    )
    
    yield* healthMonitor
    yield* Effect.addFinalizer(() => resource.close())
    
    return resource
  }),
  timeToLive: (exit) => {
    if (Exit.isFailure(exit)) {
      return Duration.minutes(1) // Quick retry for failed resources
    }
    
    const resource = exit.value
    
    // Check health and latency for TTL decision
    return Effect.gen(function* () {
      const isHealthy = yield* resource.isHealthy()
      const currentLatency = yield* resource.getLatency()
      
      if (!isHealthy) {
        return Duration.minutes(2) // Short TTL for unhealthy resources
      }
      
      if (currentLatency > 1000) {
        return Duration.minutes(5) // Medium TTL for slow resources
      }
      
      return Duration.minutes(20) // Long TTL for healthy, fast resources
    }).pipe(Effect.runSync)
  }
})
```

### Feature 2: Scoped Cache Statistics and Monitoring

Advanced statistics and monitoring capabilities help optimize cache performance and resource usage.

#### Comprehensive Cache Monitoring

```typescript
import { ScopedCache, Effect, Duration, Ref, Schedule } from "effect"

interface CacheMetrics {
  readonly hits: number
  readonly misses: number
  readonly evictions: number
  readonly resourcesCreated: number
  readonly resourcesCleaned: number
  readonly averageResourceLifetime: number
  readonly memoryUsage: number
}

const createMonitoredScopedCache = <K, V, E>(
  name: string,
  options: {
    capacity: number
    timeToLive: Duration.DurationInput
    lookup: (key: K) => Effect.Effect<V, E, Scope.Scope>
    onResourceCreated?: (key: K, value: V) => Effect.Effect<void>
    onResourceCleaned?: (key: K, value: V) => Effect.Effect<void>
  }
) => Effect.gen(function* () {
  const metrics = yield* Ref.make<CacheMetrics>({
    hits: 0,
    misses: 0,
    evictions: 0,
    resourcesCreated: 0,
    resourcesCleaned: 0,
    averageResourceLifetime: 0,
    memoryUsage: 0
  })
  
  const resourceLifetimes = yield* Ref.make<Array<number>>([])
  
  const cache = yield* ScopedCache.make({
    capacity: options.capacity,
    timeToLive: options.timeToLive,
    lookup: (key: K) => Effect.gen(function* () {
      const startTime = Date.now()
      
      // Call original lookup
      const resource = yield* options.lookup(key)
      
      // Track resource creation
      yield* Ref.update(metrics, m => ({
        ...m,
        resourcesCreated: m.resourcesCreated + 1
      }))
      
      yield* options.onResourceCreated?.(key, resource) ?? Effect.void
      
      // Add finalizer to track cleanup
      yield* Effect.addFinalizer(() => Effect.gen(function* () {
        const lifetime = Date.now() - startTime
        
        yield* Ref.update(resourceLifetimes, lifetimes => [...lifetimes, lifetime])
        yield* Ref.update(metrics, m => {
          const newLifetimes = [...resourceLifetimes.current, lifetime]
          const avgLifetime = newLifetimes.reduce((a, b) => a + b, 0) / newLifetimes.length
          
          return {
            ...m,
            resourcesCleaned: m.resourcesCleaned + 1,
            averageResourceLifetime: avgLifetime
          }
        })
        
        yield* options.onResourceCleaned?.(key, resource) ?? Effect.void
      }))
      
      return resource
    })
  })
  
  const monitoredGet = (key: K) => Effect.gen(function* () {
    const exists = yield* cache.contains(key)
    
    if (exists) {
      yield* Ref.update(metrics, m => ({ ...m, hits: m.hits + 1 }))
    } else {
      yield* Ref.update(metrics, m => ({ ...m, misses: m.misses + 1 }))
    }
    
    return yield* cache.get(key)
  })
  
  const getMetrics = () => Ref.get(metrics)
  
  const logMetrics = Effect.gen(function* () {
    const currentMetrics = yield* getMetrics()
    const cacheStats = yield* cache.cacheStats
    
    const hitRate = currentMetrics.hits / (currentMetrics.hits + currentMetrics.misses) || 0
    
    console.log(`Cache ${name} metrics:`, {
      hitRate: `${(hitRate * 100).toFixed(2)}%`,
      size: cacheStats.size,
      resourcesActive: currentMetrics.resourcesCreated - currentMetrics.resourcesCleaned,
      avgLifetime: `${currentMetrics.averageResourceLifetime.toFixed(0)}ms`,
      ...currentMetrics
    })
  })
  
  // Start periodic metrics logging
  const metricsLogger = yield* logMetrics.pipe(
    Effect.repeat(Schedule.fixed(Duration.minutes(5))),
    Effect.fork
  )
  
  return {
    get: monitoredGet,
    cache,
    getMetrics,
    logMetrics,
    metricsLogger
  }
})

// Usage with monitoring
const monitoredFileCache = createMonitoredScopedCache("file-cache", {
  capacity: 100,
  timeToLive: Duration.minutes(15),
  lookup: (filePath: string) => Effect.gen(function* () {
    const handle = yield* openFile(filePath)
    yield* Effect.addFinalizer(() => handle.close())
    return handle
  }),
  onResourceCreated: (path, handle) => 
    Effect.sync(() => console.log(`📁 Opened file: ${path}`)),
  onResourceCleaned: (path, handle) => 
    Effect.sync(() => console.log(`🗑️ Closed file: ${path}`))
})
```

### Feature 3: Advanced Refresh Strategies

ScopedCache refresh functionality with sophisticated strategies for resource management.

#### Proactive Background Refresh

```typescript
import { ScopedCache, Effect, Duration, Schedule, Queue, Fiber } from "effect"

const createAutoRefreshScopedCache = <K, V, E>(
  name: string,
  options: {
    capacity: number
    timeToLive: Duration.DurationInput
    refreshInterval: Duration.DurationInput
    lookup: (key: K) => Effect.Effect<V, E, Scope.Scope>
    shouldRefresh?: (key: K, value: V) => Effect.Effect<boolean>
  }
) => Effect.gen(function* () {
  const cache = yield* ScopedCache.make({
    capacity: options.capacity,
    timeToLive: options.timeToLive,
    lookup: options.lookup
  })
  
  const activeKeys = yield* Ref.make<Set<K>>(new Set())
  const refreshQueue = yield* Queue.unbounded<K>()
  
  // Track accessed keys
  const get = (key: K) => Effect.gen(function* () {
    yield* Ref.update(activeKeys, keys => new Set([...keys, key]))
    return yield* cache.get(key)
  })
  
  // Background refresh worker
  const refreshWorker = Effect.gen(function* () {
    const key = yield* Queue.take(refreshQueue)
    
    try {
      // Check if we should refresh this key
      const shouldRefresh = options.shouldRefresh
      if (shouldRefresh) {
        const exists = yield* cache.contains(key)
        if (exists) {
          const value = yield* cache.getOptionComplete(key)
          if (value._tag === "Some") {
            const shouldRefreshKey = yield* shouldRefresh(key, value.value)
            if (!shouldRefreshKey) {
              return
            }
          }
        }
      }
      
      console.log(`🔄 Background refreshing key: ${key}`)
      yield* cache.refresh(key).pipe(
        Effect.catchAll(error => {
          console.warn(`Failed to refresh ${name} cache for key:`, key, error)
          return Effect.unit
        })
      )
    } catch (error) {
      console.warn(`Refresh worker error for ${name}:`, error)
    }
  }).pipe(
    Effect.forever,
    Effect.fork
  )
  
  // Start refresh workers
  const workers = yield* Effect.all(
    Array.from({ length: 3 }, () => refreshWorker)
  )
  
  // Periodic refresh scheduler
  const refreshScheduler = Effect.gen(function* () {
    const keys = yield* Ref.get(activeKeys)
    
    console.log(`📋 Scheduling refresh for ${keys.size} active keys in ${name}`)
    
    // Queue all active keys for refresh
    yield* Effect.all(
      Array.from(keys).map(key => Queue.offer(refreshQueue, key))
    )
  }).pipe(
    Effect.repeat(Schedule.fixed(options.refreshInterval)),
    Effect.fork
  )
  
  yield* refreshScheduler
  
  return { get, cache, workers, refreshScheduler }
})

// Usage for critical resources
const criticalResourceCache = createAutoRefreshScopedCache("critical-resources", {
  capacity: 50,
  timeToLive: Duration.minutes(30),
  refreshInterval: Duration.minutes(5),
  lookup: (resourceId: string) => Effect.gen(function* () {
    console.log(`🔧 Creating critical resource: ${resourceId}`)
    
    const resource = {
      id: resourceId,
      data: `Critical data for ${resourceId}`,
      lastUpdated: Date.now(),
      close: () => Effect.sync(() => {
        console.log(`🧹 Cleaning up critical resource: ${resourceId}`)
      })
    }
    
    yield* Effect.addFinalizer(() => resource.close())
    
    return resource
  }),
  shouldRefresh: (key, value) => Effect.gen(function* () {
    // Refresh if resource is older than 3 minutes
    const age = Date.now() - value.lastUpdated
    return age > 3 * 60 * 1000
  })
})
```

## Practical Patterns & Best Practices

### Pattern 1: Resource Pool Management

```typescript
import { ScopedCache, Effect, Duration, Queue, Ref, Array as Arr } from "effect"

// Generic resource pool using ScopedCache
const createResourcePool = <T>(
  poolName: string,
  options: {
    maxResources: number
    resourceTTL: Duration.DurationInput
    createResource: () => Effect.Effect<T, never, Scope.Scope>
    healthCheck: (resource: T) => Effect.Effect<boolean>
    onResourceRemoved?: (resource: T) => Effect.Effect<void>
  }
) => Effect.gen(function* () {
  const availableResources = yield* Queue.unbounded<T>()
  const resourceCount = yield* Ref.make(0)
  
  const cache = yield* ScopedCache.make({
    capacity: 1, // Single pool entry
    timeToLive: Duration.hours(24), // Pool lives for a day
    lookup: (_poolKey: string) => Effect.gen(function* () {
      console.log(`🏊 Creating resource pool: ${poolName}`)
      
      const pool = {
        acquire: () => Effect.gen(function* () {
          // Try to get from available resources first
          const maybeResource = yield* Queue.poll(availableResources)
          
          if (maybeResource._tag === "Some") {
            const resource = maybeResource.value
            
            // Health check
            const isHealthy = yield* options.healthCheck(resource)
            if (isHealthy) {
              return resource
            } else {
              // Resource is unhealthy, remove it
              yield* options.onResourceRemoved?.(resource) ?? Effect.void
              yield* Ref.update(resourceCount, n => n - 1)
            }
          }
          
          // Create new resource if under limit
          const currentCount = yield* Ref.get(resourceCount)
          if (currentCount < options.maxResources) {
            const resource = yield* options.createResource()
            yield* Ref.update(resourceCount, n => n + 1)
            return resource
          }
          
          // Wait for available resource
          return yield* Queue.take(availableResources)
        }),
        
        release: (resource: T) => Effect.gen(function* () {
          const isHealthy = yield* options.healthCheck(resource)
          
          if (isHealthy) {
            yield* Queue.offer(availableResources, resource)
          } else {
            yield* options.onResourceRemoved?.(resource) ?? Effect.void
            yield* Ref.update(resourceCount, n => n - 1)
          }
        }),
        
        size: () => Ref.get(resourceCount),
        
        close: () => Effect.gen(function* () {
          console.log(`🏊‍♂️ Closing resource pool: ${poolName}`)
          
          // Close all available resources
          let resource = yield* Queue.poll(availableResources)
          while (resource._tag === "Some") {
            yield* options.onResourceRemoved?.(resource.value) ?? Effect.void
            resource = yield* Queue.poll(availableResources)
          }
          
          yield* Ref.set(resourceCount, 0)
        })
      }
      
      yield* Effect.addFinalizer(() => pool.close())
      
      return pool
    })
  })
  
  const getPool = () => cache.get(poolName)
  
  return { getPool, cache }
})

// Database connection pool example
const databaseConnectionPool = createResourcePool("db-connections", {
  maxResources: 20,
  resourceTTL: Duration.minutes(30),
  createResource: () => Effect.gen(function* () {
    const connection = yield* createDatabaseConnection("postgresql://localhost/app")
    
    yield* Effect.addFinalizer(() => connection.close())
    
    return connection
  }),
  healthCheck: (connection) => Effect.gen(function* () {
    try {
      yield* connection.query("SELECT 1")
      return true
    } catch {
      return false
    }
  }),
  onResourceRemoved: (connection) => connection.close()
})

// Usage pattern with automatic resource management
const withDatabaseConnection = <T, E>(
  operation: (connection: DatabaseConnection) => Effect.Effect<T, E>
) => Effect.gen(function* () {
  const { getPool } = yield* databaseConnectionPool
  const pool = yield* getPool()
  
  const connection = yield* pool.acquire()
  
  try {
    const result = yield* operation(connection)
    yield* pool.release(connection)
    return result
  } catch (error) {
    yield* pool.release(connection)
    return yield* Effect.fail(error)
  }
}).pipe(Effect.scoped)
```

### Pattern 2: Hierarchical Cache Invalidation

```typescript
import { ScopedCache, Effect, Duration, Ref, Array as Arr } from "effect"

// Cache with dependency tracking and cascading invalidation
const createDependentScopedCache = <K, V, E>(options: {
  name: string
  capacity: number
  timeToLive: Duration.DurationInput
  lookup: (key: K) => Effect.Effect<V, E, Scope.Scope>
  getDependencies: (key: K, value: V) => ReadonlyArray<string>
}) => Effect.gen(function* () {
  const dependencyMap = yield* Ref.make<Map<string, Set<K>>>(new Map())
  const keyDependencies = yield* Ref.make<Map<K, Set<string>>>(new Map())
  
  const cache = yield* ScopedCache.make({
    capacity: options.capacity,
    timeToLive: options.timeToLive,
    lookup: (key: K) => Effect.gen(function* () {
      const resource = yield* options.lookup(key)
      const dependencies = options.getDependencies(key, resource)
      
      // Track dependencies
      yield* Ref.update(keyDependencies, map => {
        const newMap = new Map(map)
        newMap.set(key, new Set(dependencies))
        return newMap
      })
      
      yield* Ref.update(dependencyMap, map => {
        const newMap = new Map(map)
        dependencies.forEach(dep => {
          const keySet = newMap.get(dep) || new Set()
          keySet.add(key)
          newMap.set(dep, keySet)
        })
        return newMap
      })
      
      // Clean up dependencies on resource cleanup
      yield* Effect.addFinalizer(() => Effect.gen(function* () {
        const keyDeps = yield* Ref.get(keyDependencies)
        const deps = keyDeps.get(key)
        
        if (deps) {
          yield* Ref.update(dependencyMap, map => {
            const newMap = new Map(map)
            deps.forEach(dep => {
              const keySet = newMap.get(dep)
              if (keySet) {
                keySet.delete(key)
                if (keySet.size === 0) {
                  newMap.delete(dep)
                }
              }
            })
            return newMap
          })
          
          yield* Ref.update(keyDependencies, map => {
            const newMap = new Map(map)
            newMap.delete(key)
            return newMap
          })
        }
      }))
      
      return resource
    })
  })
  
  const get = (key: K) => cache.get(key)
  
  const invalidateByDependency = (dependency: string) => Effect.gen(function* () {
    const depMap = yield* Ref.get(dependencyMap)
    const keysToInvalidate = depMap.get(dependency)
    
    if (keysToInvalidate) {
      console.log(`🗑️ Invalidating ${keysToInvalidate.size} keys dependent on: ${dependency}`)
      
      yield* Effect.all(
        Arr.fromIterable(keysToInvalidate).map(key => cache.invalidate(key))
      )
    }
  })
  
  const invalidateAll = () => cache.invalidateAll
  
  return {
    get,
    invalidateByDependency,
    invalidateAll,
    cache
  }
})

// User profile cache with organization dependencies
const userProfileCache = createDependentScopedCache({
  name: "user-profiles",
  capacity: 1000,
  timeToLive: Duration.minutes(20),
  lookup: (userId: string) => Effect.gen(function* () {
    const database = yield* Database
    const profile = yield* database.users.findById(userId)
    
    if (!profile) {
      return yield* Effect.fail(new UserNotFoundError(userId))
    }
    
    // Add cleanup finalizer
    yield* Effect.addFinalizer(() => Effect.gen(function* () {
      console.log(`🧹 Cleaning up profile cache for user: ${userId}`)
    }))
    
    return profile
  }),
  getDependencies: (userId, profile) => [
    `org:${profile.organizationId}`,
    `role:${profile.role}`,
    `department:${profile.departmentId}`
  ]
})

// Usage with dependency-based invalidation
const handleOrganizationUpdate = (orgId: string) => Effect.gen(function* () {
  const cache = yield* userProfileCache
  
  // This will invalidate all user profiles in the organization
  yield* cache.invalidateByDependency(`org:${orgId}`)
  
  console.log(`✅ Updated organization ${orgId} and invalidated dependent caches`)
}).pipe(Effect.scoped)

const handleRoleChange = (roleId: string) => Effect.gen(function* () {
  const cache = yield* userProfileCache
  
  // This will invalidate all user profiles with this role
  yield* cache.invalidateByDependency(`role:${roleId}`)
  
  console.log(`✅ Updated role ${roleId} and invalidated dependent caches`)
}).pipe(Effect.scoped)
```

### Pattern 3: Multi-Tier Scoped Caching

```typescript
import { ScopedCache, Effect, Duration, Option } from "effect"

// Multi-tier cache with automatic fallback and promotion
const createMultiTierScopedCache = <K, V, E>(
  name: string,
  options: {
    l1: { capacity: number; timeToLive: Duration.DurationInput }
    l2: { capacity: number; timeToLive: Duration.DurationInput }
    lookup: (key: K) => Effect.Effect<V, E, Scope.Scope>
    serialize?: (value: V) => string
    deserialize?: (data: string) => V
  }
) => Effect.gen(function* () {
  // L1 Cache: Fast in-memory cache
  const l1Cache = yield* ScopedCache.make({
    capacity: options.l1.capacity,
    timeToLive: options.l1.timeToLive,
    lookup: (key: K) => Effect.gen(function* () {
      console.log(`🔥 L1 miss for ${name}: ${key}`)
      
      // Try L2 cache first
      const l2Result = yield* l2Cache.getOption(key)
      
      if (l2Result._tag === "Some") {
        console.log(`⚡ L2 hit for ${name}: ${key}`)
        return l2Result.value
      }
      
      // L2 miss, go to source
      console.log(`💾 L2 miss for ${name}: ${key}, fetching from source`)
      const value = yield* options.lookup(key)
      
      // Warm L2 cache with the new value
      yield* l2Cache.get(key).pipe(Effect.fork)
      
      return value
    })
  })
  
  // L2 Cache: Larger, longer-lived cache
  const l2Cache = yield* ScopedCache.make({
    capacity: options.l2.capacity,
    timeToLive: options.l2.timeToLive,
    lookup: options.lookup
  })
  
  const get = (key: K) => Effect.gen(function* () {
    // Try L1 first
    const l1Result = yield* l1Cache.getOption(key)
    
    if (l1Result._tag === "Some") {
      console.log(`⚡ L1 hit for ${name}: ${key}`)
      return l1Result.value
    }
    
    // L1 miss, get from L1 (which will check L2 and source)
    return yield* l1Cache.get(key)
  })
  
  const invalidate = (key: K) => Effect.gen(function* () {
    console.log(`🗑️ Invalidating ${name} caches for key: ${key}`)
    yield* Effect.all([
      l1Cache.invalidate(key),
      l2Cache.invalidate(key)
    ])
  })
  
  const invalidateAll = () => Effect.gen(function* () {
    console.log(`🗑️ Invalidating all ${name} caches`)
    yield* Effect.all([
      l1Cache.invalidateAll,
      l2Cache.invalidateAll
    ])
  })
  
  const getStats = () => Effect.gen(function* () {
    const [l1Stats, l2Stats] = yield* Effect.all([
      l1Cache.cacheStats,
      l2Cache.cacheStats
    ])
    
    return {
      l1: l1Stats,
      l2: l2Stats,
      totalHits: l1Stats.hits + l2Stats.hits,
      totalMisses: l1Stats.misses + l2Stats.misses
    }
  })
  
  return {
    get,
    invalidate,
    invalidateAll,
    getStats,
    l1Cache,
    l2Cache
  }
})

// Product catalog with multi-tier caching
const productCatalogCache = createMultiTierScopedCache("product-catalog", {
  l1: { 
    capacity: 100,     // Hot products
    timeToLive: Duration.minutes(5) 
  },
  l2: { 
    capacity: 5000,    // All products
    timeToLive: Duration.minutes(30) 
  },
  lookup: (productId: string) => Effect.gen(function* () {
    console.log(`🛒 Loading product from database: ${productId}`)
    
    const database = yield* Database
    const product = yield* database.products.findById(productId)
    
    if (!product) {
      return yield* Effect.fail(new ProductNotFoundError(productId))
    }
    
    // Add resource cleanup
    yield* Effect.addFinalizer(() => Effect.gen(function* () {
      console.log(`🧹 Cleaning up product cache entry: ${productId}`)
    }))
    
    return product
  })
})

// Usage with intelligent caching
const getProductWithRecommendations = (productId: string) => Effect.gen(function* () {
  const cache = yield* productCatalogCache
  
  // Get main product (uses multi-tier cache)
  const product = yield* cache.get(productId)
  
  // Get related products concurrently (may hit different cache tiers)
  const relatedProducts = yield* Effect.all(
    product.relatedProductIds.map(id => cache.get(id)),
    { concurrency: 5 }
  )
  
  return {
    product,
    relatedProducts
  }
}).pipe(Effect.scoped)
```

## Integration Examples

### Integration with HTTP Server Middleware

```typescript
import { ScopedCache, Effect, Duration, HttpServer, Context } from "effect"

// Request-scoped cache middleware
interface RequestCache {
  readonly get: <T>(key: string, lookup: () => Effect.Effect<T>) => Effect.Effect<T>
  readonly set: <T>(key: string, value: T) => Effect.Effect<void>
  readonly clear: () => Effect.Effect<void>
}

const RequestCache = Context.GenericTag<RequestCache>("RequestCache")

const requestCacheMiddleware = HttpServer.middleware.make(
  (app) => Effect.gen(function* () {
    // Create request-scoped cache
    const requestCache = yield* ScopedCache.make({
      capacity: 100,
      timeToLive: Duration.minutes(5), // Cache lives for request duration
      lookup: (cacheKey: string) => Effect.gen(function* () {
        // This will be populated by the actual cache operations
        return yield* Effect.fail(new Error(`No lookup defined for key: ${cacheKey}`))
      })
    })
    
    const cache: RequestCache = {
      get: <T>(key: string, lookup: () => Effect.Effect<T>) => Effect.gen(function* () {
        const existing = yield* requestCache.getOption(key)
        
        if (existing._tag === "Some") {
          return existing.value as T
        }
        
        // Execute lookup and cache result
        const value = yield* lookup()
        yield* requestCache.invalidate(key) // Clear any failed lookup
        
        // Update the cache with a new lookup that returns the value
        const newCache = yield* ScopedCache.make({
          capacity: 100,
          timeToLive: Duration.minutes(5),
          lookup: (k: string) => k === key ? Effect.succeed(value) : Effect.fail(new Error("Not found"))
        })
        
        return value
      }),
      
      set: <T>(key: string, value: T) => Effect.gen(function* () {
        // Implementation would update the cache
        return yield* Effect.void
      }),
      
      clear: () => requestCache.invalidateAll
    }
    
    return yield* app.pipe(
      Effect.provideService(RequestCache, cache),
      Effect.scoped // Ensures cache is cleaned up after request
    )
  })
)

// HTTP route using request cache
const userRoutes = HttpServer.router.empty.pipe(
  HttpServer.router.get("/users/:id", Effect.gen(function* () {
    const request = yield* HttpServer.request.HttpServerRequest
    const cache = yield* RequestCache
    const userId = request.params.id
    
    // Cache user data for the request duration
    const user = yield* cache.get(`user:${userId}`, () => Effect.gen(function* () {
      const database = yield* Database
      return yield* database.users.findById(userId)
    }))
    
    // Cache user permissions (may reuse user data)
    const permissions = yield* cache.get(`permissions:${userId}`, () => Effect.gen(function* () {
      const authService = yield* AuthService
      return yield* authService.getUserPermissions(user)
    }))
    
    return HttpServer.response.json({ user, permissions })
  }))
)

// Server with request-scoped caching
const server = HttpServer.router.empty.pipe(
  HttpServer.router.mount("/api", userRoutes),
  HttpServer.middleware.use(requestCacheMiddleware),
  HttpServer.listen({ port: 3000 })
)
```

### Integration with Task Queue System

```typescript
import { ScopedCache, Effect, Duration, Queue, Schedule } from "effect"

interface TaskProcessor<T> {
  readonly process: (task: T) => Effect.Effect<void>
  readonly close: () => Effect.Effect<void>
}

interface TaskQueue<T> {
  readonly enqueue: (task: T) => Effect.Effect<void>
  readonly size: () => Effect.Effect<number>
  readonly close: () => Effect.Effect<void>
}

// Task processor cache for handling different task types
const taskProcessorCache = ScopedCache.make({
  capacity: 10,
  timeToLive: Duration.minutes(30),
  lookup: (taskType: string) => Effect.gen(function* () {
    console.log(`🔧 Creating task processor for: ${taskType}`)
    
    const taskQueue = yield* Queue.unbounded<any>()
    let isRunning = true
    
    const processor: TaskProcessor<any> = {
      process: (task) => Effect.gen(function* () {
        yield* Queue.offer(taskQueue, task)
      }),
      close: () => Effect.gen(function* () {
        console.log(`🛑 Stopping task processor: ${taskType}`)
        isRunning = false
      })
    }
    
    // Start task processing worker
    const worker = Effect.gen(function* () {
      while (isRunning) {
        const task = yield* Queue.take(taskQueue)
        
        try {
          yield* processTaskByType(taskType, task)
          console.log(`✅ Processed ${taskType} task:`, task.id)
        } catch (error) {
          console.error(`❌ Failed to process ${taskType} task:`, task.id, error)
        }
      }
    }).pipe(
      Effect.fork
    )
    
    yield* worker
    
    // Register cleanup
    yield* Effect.addFinalizer(() => processor.close())
    
    return processor
  })
})

// Task queue factory using processor cache
const createTaskQueue = <T>(taskType: string): Effect.Effect<TaskQueue<T>, never, Scope.Scope> =>
  Effect.gen(function* () {
    const cache = yield* taskProcessorCache
    const processor = yield* cache.get(taskType)
    
    const queue: TaskQueue<T> = {
      enqueue: (task: T) => processor.process(task),
      size: () => Effect.succeed(0), // Would implement actual size tracking
      close: () => processor.close()
    }
    
    return queue
  })

// Usage in application
const emailTaskQueue = createTaskQueue<EmailTask>("email")
const imageProcessingQueue = createTaskQueue<ImageTask>("image-processing")
const reportGenerationQueue = createTaskQueue<ReportTask>("report-generation")

const enqueueEmail = (email: EmailTask) => Effect.gen(function* () {
  const queue = yield* emailTaskQueue
  yield* queue.enqueue(email)
}).pipe(Effect.scoped)

const processTaskByType = (taskType: string, task: any) => Effect.gen(function* () {
  switch (taskType) {
    case "email":
      yield* sendEmail(task)
      break
    case "image-processing":
      yield* processImage(task)
      break
    case "report-generation":
      yield* generateReport(task)
      break
    default:
      console.warn(`Unknown task type: ${taskType}`)
  }
})
```

### Integration with WebSocket Connection Manager

```typescript
import { ScopedCache, Effect, Duration, Ref, Schedule } from "effect"

interface WebSocketRoom {
  readonly roomId: string
  readonly connections: Set<WebSocketConnection>
  readonly addConnection: (conn: WebSocketConnection) => Effect.Effect<void>
  readonly removeConnection: (conn: WebSocketConnection) => Effect.Effect<void>
  readonly broadcast: (message: string) => Effect.Effect<void>
  readonly close: () => Effect.Effect<void>
}

// WebSocket room cache with automatic cleanup
const webSocketRoomCache = ScopedCache.make({
  capacity: 1000, // Max 1000 concurrent rooms
  timeToLive: Duration.minutes(60), // Rooms expire after 1 hour of inactivity
  lookup: (roomId: string) => Effect.gen(function* () {
    console.log(`🏠 Creating WebSocket room: ${roomId}`)
    
    const connections = yield* Ref.make<Set<WebSocketConnection>>(new Set())
    const lastActivity = yield* Ref.make(Date.now())
    
    const room: WebSocketRoom = {
      roomId,
      connections: new Set(),
      
      addConnection: (conn) => Effect.gen(function* () {
        yield* Ref.update(connections, conns => new Set([...conns, conn]))
        yield* Ref.set(lastActivity, Date.now())
        
        console.log(`➕ Added connection to room ${roomId}`)
        
        // Notify other connections
        const currentConns = yield* Ref.get(connections)
        yield* Effect.all(
          Array.from(currentConns)
            .filter(c => c !== conn)
            .map(c => c.send(JSON.stringify({
              type: "user_joined",
              roomId,
              timestamp: Date.now()
            })).pipe(Effect.catchAll(() => Effect.void)))
        )
      }),
      
      removeConnection: (conn) => Effect.gen(function* () {
        yield* Ref.update(connections, conns => {
          const newConns = new Set(conns)
          newConns.delete(conn)
          return newConns
        })
        
        console.log(`➖ Removed connection from room ${roomId}`)
        
        // Notify other connections
        const currentConns = yield* Ref.get(connections)
        yield* Effect.all(
          Array.from(currentConns).map(c => 
            c.send(JSON.stringify({
              type: "user_left",
              roomId,
              timestamp: Date.now()
            })).pipe(Effect.catchAll(() => Effect.void))
          )
        )
      }),
      
      broadcast: (message) => Effect.gen(function* () {
        const currentConns = yield* Ref.get(connections)
        yield* Ref.set(lastActivity, Date.now())
        
        console.log(`📢 Broadcasting to ${currentConns.size} connections in room ${roomId}`)
        
        yield* Effect.all(
          Array.from(currentConns).map(conn =>
            conn.send(message).pipe(
              Effect.catchAll(error => {
                console.warn(`Failed to send message to connection:`, error)
                return room.removeConnection(conn)
              })
            )
          ),
          { concurrency: "unbounded" }
        )
      }),
      
      close: () => Effect.gen(function* () {
        console.log(`🚪 Closing WebSocket room: ${roomId}`)
        
        const currentConns = yield* Ref.get(connections)
        
        // Notify all connections that room is closing
        yield* Effect.all(
          Array.from(currentConns).map(conn =>
            conn.send(JSON.stringify({
              type: "room_closing",
              roomId,
              timestamp: Date.now()
            })).pipe(Effect.catchAll(() => Effect.void))
          )
        )
        
        // Close all connections
        yield* Effect.all(
          Array.from(currentConns).map(conn => conn.close())
        )
        
        yield* Ref.set(connections, new Set())
      })
    }
    
    // Periodic cleanup of inactive connections
    const cleanupTask = Effect.gen(function* () {
      const currentConns = yield* Ref.get(connections)
      const activeConns = new Set<WebSocketConnection>()
      
      for (const conn of currentConns) {
        if (conn.isConnected()) {
          activeConns.add(conn)
        } else {
          console.log(`🧹 Removing inactive connection from room ${roomId}`)
        }
      }
      
      yield* Ref.set(connections, activeConns)
    }).pipe(
      Effect.repeat(Schedule.fixed(Duration.minutes(5))),
      Effect.fork
    )
    
    yield* cleanupTask
    
    // Register room cleanup
    yield* Effect.addFinalizer(() => room.close())
    
    return room
  })
})

// WebSocket message handler
const handleWebSocketMessage = (
  roomId: string,
  connection: WebSocketConnection,
  message: string
) => Effect.gen(function* () {
  const cache = yield* webSocketRoomCache
  const room = yield* cache.get(roomId)
  
  const parsedMessage = JSON.parse(message)
  
  switch (parsedMessage.type) {
    case "join_room":
      yield* room.addConnection(connection)
      break
      
    case "leave_room":
      yield* room.removeConnection(connection)
      break
      
    case "chat_message":
      const chatMessage = JSON.stringify({
        type: "chat_message",
        roomId,
        message: parsedMessage.content,
        timestamp: Date.now(),
        sender: parsedMessage.sender
      })
      yield* room.broadcast(chatMessage)
      break
      
    default:
      console.warn(`Unknown message type: ${parsedMessage.type}`)
  }
}).pipe(Effect.scoped)

// WebSocket server integration
const webSocketServer = Effect.gen(function* () {
  // Create WebSocket server
  const server = new WebSocketServer({ port: 8080 })
  
  server.on("connection", (ws, req) => {
    const url = new URL(req.url!, `http://${req.headers.host}`)
    const roomId = url.searchParams.get("room")
    
    if (!roomId) {
      ws.close(1008, "Room ID required")
      return
    }
    
    const connection: WebSocketConnection = {
      send: (message) => Effect.promise(() => {
        return new Promise((resolve, reject) => {
          ws.send(message, (error) => {
            if (error) reject(error)
            else resolve()
          })
        })
      }),
      close: () => Effect.sync(() => ws.close()),
      isConnected: () => ws.readyState === WebSocket.OPEN
    }
    
    ws.on("message", (data) => {
      Effect.runFork(
        handleWebSocketMessage(roomId, connection, data.toString())
      )
    })
    
    ws.on("close", () => {
      Effect.runFork(Effect.gen(function* () {
        const cache = yield* webSocketRoomCache
        const room = yield* cache.get(roomId)
        yield* room.removeConnection(connection)
      }).pipe(Effect.scoped))
    })
  })
  
  console.log("🚀 WebSocket server listening on port 8080")
  
  return server
})
```

### Testing Strategies

```typescript
import { ScopedCache, Effect, Duration, TestContext, TestClock, TestScope } from "effect"

// Test utilities for ScopedCache
const createTestScopedCache = <K, V>(
  lookup: (key: K) => V,
  options: {
    capacity?: number
    timeToLive?: Duration.DurationInput
    onResourceCreated?: (key: K, value: V) => void
    onResourceCleaned?: (key: K, value: V) => void
  } = {}
) => ScopedCache.make({
  capacity: options.capacity ?? 100,
  timeToLive: options.timeToLive ?? Duration.minutes(5),
  lookup: (key: K) => Effect.gen(function* () {
    const value = lookup(key)
    
    options.onResourceCreated?.(key, value)
    
    yield* Effect.addFinalizer(() => Effect.sync(() => {
      options.onResourceCleaned?.(key, value)
    }))
    
    return value
  })
})

// Test resource cleanup behavior
const testResourceCleanup = Effect.gen(function* () {
  let resourcesCreated = 0
  let resourcesCleaned = 0
  
  const cache = yield* createTestScopedCache(
    (key: string) => ({ id: key, data: `data-${key}` }),
    {
      capacity: 2,
      timeToLive: Duration.seconds(1),
      onResourceCreated: () => { resourcesCreated++ },
      onResourceCleaned: () => { resourcesCleaned++ }
    }
  )
  
  // Create resources
  const resource1 = yield* cache.get("key1")
  const resource2 = yield* cache.get("key2")
  const resource3 = yield* cache.get("key3") // Should evict key1
  
  console.assert(resourcesCreated === 3, "Should create 3 resources")
  console.assert(resourcesCleaned >= 1, "Should clean up at least 1 resource due to capacity")
  
  // Wait for TTL expiration
  yield* TestClock.adjust(Duration.seconds(2))
  
  // Force cache cleanup by trying to access
  yield* cache.get("key4")
  
  console.assert(resourcesCleaned >= 3, "Should clean up expired resources")
  
  console.log("✅ Resource cleanup test passed")
}).pipe(
  Effect.provide(TestContext.TestContext),
  Effect.scoped
)

// Test scope integration
const testScopeIntegration = Effect.gen(function* () {
  let cleanupCalled = false
  
  // Inner scope with cache
  yield* Effect.scoped(Effect.gen(function* () {
    const cache = yield* createTestScopedCache(
      (key: string) => ({ id: key }),
      {
        onResourceCleaned: () => { cleanupCalled = true }
      }
    )
    
    yield* cache.get("test-key")
    
    // Resource should still be alive within scope
    console.assert(!cleanupCalled, "Resource should not be cleaned up yet")
  }))
  
  // After scope closes, resource should be cleaned up
  console.assert(cleanupCalled, "Resource should be cleaned up after scope closes")
  
  console.log("✅ Scope integration test passed")
})

// Test concurrent access
const testConcurrentAccess = Effect.gen(function* () {
  let lookupCount = 0
  
  const cache = yield* createTestScopedCache(
    (key: string) => {
      lookupCount++
      return { id: key, lookupCount }
    }
  )
  
  // Concurrent requests for same key
  const results = yield* Effect.all(
    Array.from({ length: 10 }, () => cache.get("concurrent-key")),
    { concurrency: "unbounded" }
  )
  
  // Should only lookup once despite concurrent requests
  console.assert(lookupCount === 1, "Should only lookup once for concurrent requests")
  console.assert(results.every(r => r.id === "concurrent-key"), "All results should be the same")
  
  console.log("✅ Concurrent access test passed")
}).pipe(Effect.scoped)

// Test cache invalidation
const testCacheInvalidation = Effect.gen(function* () {
  let lookupCount = 0
  
  const cache = yield* createTestScopedCache(
    (key: string) => {
      lookupCount++
      return { id: key, lookupCount }
    }
  )
  
  // First access
  const result1 = yield* cache.get("invalidation-key")
  console.assert(result1.lookupCount === 1, "First lookup should increment counter")
  
  // Second access (should use cache)
  const result2 = yield* cache.get("invalidation-key")
  console.assert(result2.lookupCount === 1, "Second access should use cached value")
  console.assert(lookupCount === 1, "Lookup should only be called once")
  
  // Invalidate cache
  yield* cache.invalidate("invalidation-key")
  
  // Third access (should lookup again)
  const result3 = yield* cache.get("invalidation-key")
  console.assert(result3.lookupCount === 2, "After invalidation should lookup again")
  console.assert(lookupCount === 2, "Lookup should be called twice")
  
  console.log("✅ Cache invalidation test passed")
}).pipe(Effect.scoped)

// Mock ScopedCache for testing
const createMockScopedCache = <K, V, E = never>(
  mockData: Map<K, V>
) => Effect.succeed({
  get: (key: K) => mockData.has(key) 
    ? Effect.succeed(mockData.get(key)!)
    : Effect.fail("Not found" as E),
  getOption: (key: K) => Effect.succeed(
    mockData.has(key) ? Option.some(mockData.get(key)!) : Option.none()
  ),
  contains: (key: K) => Effect.succeed(mockData.has(key)),
  invalidate: (key: K) => Effect.sync(() => { mockData.delete(key) }),
  invalidateAll: Effect.sync(() => { mockData.clear() }),
  refresh: (key: K) => Effect.void,
  size: Effect.succeed(mockData.size),
  cacheStats: Effect.succeed({
    hits: 0,
    misses: 0,
    size: mockData.size
  }),
  entryStats: (key: K) => Effect.succeed(Option.none())
} satisfies ScopedCache.ScopedCache<K, V, E>)

// Integration test with mock
const testWithMock = Effect.gen(function* () {
  const mockData = new Map([
    ["user-1", { id: "user-1", name: "Alice" }],
    ["user-2", { id: "user-2", name: "Bob" }]
  ])
  
  const mockCache = yield* createMockScopedCache(mockData)
  
  // Test with mock data
  const user1 = yield* mockCache.get("user-1")
  console.assert(user1.name === "Alice", "Should return mocked user")
  
  const user3 = yield* Effect.either(mockCache.get("user-3"))
  console.assert(user3._tag === "Left", "Should fail for non-existent user")
  
  console.log("✅ Mock integration test passed")
})

// Run all tests
const runAllTests = Effect.gen(function* () {
  console.log("🧪 Running ScopedCache tests...")
  
  yield* testResourceCleanup
  yield* testScopeIntegration
  yield* testConcurrentAccess
  yield* testCacheInvalidation
  yield* testWithMock
  
  console.log("✅ All ScopedCache tests passed!")
})
```

## Conclusion

ScopedCache provides automatic resource lifecycle management tied to scopes, ensuring proper cleanup and preventing memory leaks in long-running Effect applications.

Key benefits:
- **Automatic Resource Management** - Resources are tied to scopes and automatically cleaned up when scopes close
- **Memory Leak Prevention** - Prevents resource accumulation through scope-based lifecycle management
- **Structured Concurrency Integration** - Seamlessly integrates with Effect's scope system for predictable resource management
- **Finalizer-Based Cleanup** - Uses Effect's finalizer system to ensure resources are properly released even in error scenarios
- **Scoped Lookup Functions** - Lookup functions execute within scopes, enabling automatic resource cleanup through finalizers

Use ScopedCache when you need cached resources with automatic cleanup tied to application scopes, especially in long-running applications where resource lifecycle management is critical for preventing memory leaks and resource exhaustion.