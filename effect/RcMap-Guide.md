# RcMap: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem RcMap Solves

Managing shared resources with proper lifecycle and automatic cleanup is challenging. Traditional approaches lead to resource leaks, premature cleanup, and complex manual reference counting that's error-prone and hard to maintain.

```typescript
// Traditional approach - problematic resource management
class DatabaseConnectionPool {
  private connections = new Map<string, Connection>()
  private refCounts = new Map<string, number>()
  
  async getConnection(config: string): Promise<Connection> {
    if (!this.connections.has(config)) {
      // Expensive connection creation - no deduplication
      const conn = await this.createConnection(config)
      this.connections.set(config, conn)
      this.refCounts.set(config, 1)
      return conn
    }
    
    // Manual reference counting - error-prone
    const current = this.refCounts.get(config) || 0
    this.refCounts.set(config, current + 1)
    return this.connections.get(config)!
  }
  
  releaseConnection(config: string) {
    const refCount = this.refCounts.get(config) || 0
    const newCount = refCount - 1
    
    if (newCount <= 0) {
      // Manual cleanup - easy to forget or mess up
      const conn = this.connections.get(config)
      conn?.close() // What if this throws?
      this.connections.delete(config)
      this.refCounts.delete(config)
    } else {
      this.refCounts.set(config, newCount)
    }
  }
  
  // Cleanup logic scattered and fragile
  private async createConnection(config: string) {
    // Complex connection logic...
    return new Connection(config)
  }
}
```

This approach leads to:
- **Manual Reference Counting** - Error-prone increment/decrement operations
- **Resource Leaks** - Forgotten releases or exception handling
- **Race Conditions** - Concurrent access to reference counts
- **Cleanup Complexity** - Manual resource disposal scattered throughout code
- **No Automatic Expiration** - Resources never cleaned up based on time

### The RcMap Solution

Effect's RcMap provides automatic reference-counted resource management with scope-based lifecycle, automatic cleanup, and built-in support for idle timeouts and capacity limits.

```typescript
import { Effect, RcMap, Scope } from "effect"

// Clean, automatic resource management
const connectionPool = RcMap.make({
  lookup: (config: string) =>
    Effect.acquireRelease(
      Effect.promise(() => createConnection(config)),
      (conn) => Effect.promise(() => conn.close())
    ),
  idleTimeToLive: "5 minutes"
})

// Usage is simple and safe - resources automatically managed
const useConnection = (config: string) =>
  Effect.gen(function* () {
    const pool = yield* connectionPool
    const connection = yield* RcMap.get(pool, config)
    // Connection automatically released when scope closes
    return yield* performDatabaseOperation(connection)
  }).pipe(Effect.scoped)
```

### Key Concepts

**Reference Counting**: Automatically tracks how many active references exist to each resource, cleaning up when count reaches zero.

**Scope-Based Lifecycle**: Resources are tied to Effect scopes, ensuring automatic cleanup when scopes close.

**Lazy Acquisition**: Resources are only created when first requested, optimizing memory usage and startup time.

**Idle Time to Live**: Resources are automatically cleaned up after remaining unused for a specified duration.

**Capacity Management**: Optional capacity limits with automatic eviction of least-recently-used items.

## Basic Usage Patterns

### Pattern 1: Simple Resource Map

```typescript
import { Effect, RcMap } from "effect"

// Create a map for managing expensive computations
const expensiveComputationMap = RcMap.make({
  lookup: (input: string) =>
    Effect.gen(function* () {
      yield* Effect.log(`Computing result for: ${input}`)
      // Simulate expensive computation
      yield* Effect.sleep("1 second")
      return `processed-${input}`
    })
})

// Usage - computation only happens once per key
const useComputation = Effect.gen(function* () {
  const map = yield* expensiveComputationMap
  
  // First call triggers computation
  const result1 = yield* RcMap.get(map, "input1").pipe(Effect.scoped)
  
  // Second call reuses the same result (within scope)
  const result2 = yield* RcMap.get(map, "input1").pipe(Effect.scoped)
  
  return [result1, result2]
})
```

### Pattern 2: Resource Map with Cleanup

```typescript
import { Effect, RcMap } from "effect"

// File handle management with automatic cleanup
const fileHandleMap = RcMap.make({
  lookup: (filepath: string) =>
    Effect.acquireRelease(
      Effect.gen(function* () {
        yield* Effect.log(`Opening file: ${filepath}`)
        return { filepath, handle: `handle-${filepath}` }
      }),
      (file) => Effect.log(`Closing file: ${file.filepath}`)
    )
})

// Multiple operations can share the same file handle
const processFile = (filepath: string) =>
  Effect.gen(function* () {
    const map = yield* fileHandleMap
    
    return yield* RcMap.get(map, filepath).pipe(
      Effect.flatMap((file) =>
        Effect.gen(function* () {
          yield* Effect.log(`Reading from ${file.filepath}`)
          yield* Effect.log(`Writing to ${file.filepath}`)
          return `processed-${file.filepath}`
        })
      ),
      Effect.scoped // File automatically closed when scope ends
    )
  })
```

### Pattern 3: Shared Resource with Reference Counting

```typescript
import { Effect, RcMap, Scope } from "effect"

// Database connection sharing
const dbConnectionMap = RcMap.make({
  lookup: (connectionString: string) =>
    Effect.acquireRelease(
      Effect.gen(function* () {
        yield* Effect.log(`Connecting to: ${connectionString}`)
        return {
          connectionString,
          execute: (query: string) => Effect.succeed(`Result: ${query}`)
        }
      }),
      (conn) => Effect.log(`Disconnecting from: ${conn.connectionString}`)
    ),
  idleTimeToLive: "30 seconds"
})

// Multiple concurrent operations share the same connection
const runQueries = (connectionString: string) =>
  Effect.gen(function* () {
    const map = yield* dbConnectionMap
    
    // These operations will share the same connection
    const query1 = RcMap.get(map, connectionString).pipe(
      Effect.flatMap((conn) => conn.execute("SELECT * FROM users")),
      Effect.scoped
    )
    
    const query2 = RcMap.get(map, connectionString).pipe(
      Effect.flatMap((conn) => conn.execute("SELECT * FROM orders")),
      Effect.scoped
    )
    
    // Connection stays alive until both queries complete
    return yield* Effect.all([query1, query2])
  })
```

## Real-World Examples

### Example 1: API Client Connection Pool

Managing HTTP client connections with automatic cleanup and connection reuse.

```typescript
import { Effect, RcMap, Duration } from "effect"

interface ApiClient {
  readonly baseUrl: string
  readonly get: (path: string) => Effect.Effect<unknown>
  readonly post: (path: string, data: unknown) => Effect.Effect<unknown>
}

// HTTP client pool with connection reuse and cleanup
const apiClientPool = RcMap.make({
  lookup: (baseUrl: string) =>
    Effect.acquireRelease(
      Effect.gen(function* () {
        yield* Effect.log(`Creating API client for: ${baseUrl}`)
        
        // Simulate client initialization
        yield* Effect.sleep("100 millis")
        
        const client: ApiClient = {
          baseUrl,
          get: (path: string) =>
            Effect.gen(function* () {
              yield* Effect.log(`GET ${baseUrl}${path}`)
              return { status: 200, data: `Response from ${path}` }
            }),
          post: (path: string, data: unknown) =>
            Effect.gen(function* () {
              yield* Effect.log(`POST ${baseUrl}${path}`)
              return { status: 201, data: `Created via ${path}` }
            })
        }
        
        return client
      }),
      (client) => Effect.log(`Disposing API client for: ${client.baseUrl}`)
    ),
  idleTimeToLive: Duration.minutes(5),
  capacity: 20
})

// Service layer using the connection pool
const makeApiService = Effect.gen(function* () {
  const pool = yield* apiClientPool
  
  const fetchUser = (baseUrl: string, userId: string) =>
    RcMap.get(pool, baseUrl).pipe(
      Effect.flatMap((client) => client.get(`/users/${userId}`)),
      Effect.scoped
    )
  
  const createUser = (baseUrl: string, userData: unknown) =>
    RcMap.get(pool, baseUrl).pipe(
      Effect.flatMap((client) => client.post("/users", userData)),
      Effect.scoped
    )
  
  const batchOperations = (baseUrl: string, userIds: string[]) =>
    RcMap.get(pool, baseUrl).pipe(
      Effect.flatMap((client) =>
        Effect.all(
          userIds.map((id) => client.get(`/users/${id}`)),
          { concurrency: 5 }
        )
      ),
      Effect.scoped
    )
  
  return { fetchUser, createUser, batchOperations } as const
})

// Usage example
const example = Effect.gen(function* () {
  const apiService = yield* makeApiService
  
  // Multiple operations sharing the same client connection
  const results = yield* Effect.all([
    apiService.fetchUser("https://api.example.com", "123"),
    apiService.fetchUser("https://api.example.com", "456"),
    apiService.createUser("https://api.example.com", { name: "John" })
  ])
  
  return results
})
```

### Example 2: Database Connection Pool with Transaction Support

Advanced database connection management with transaction support and connection pooling.

```typescript
import { Effect, RcMap, Scope, Duration } from "effect"

interface DatabaseConnection {
  readonly id: string
  readonly connectionString: string
  readonly query: <T>(sql: string, params?: unknown[]) => Effect.Effect<T>
  readonly transaction: <T>(
    work: (conn: DatabaseConnection) => Effect.Effect<T>
  ) => Effect.Effect<T>
}

// Database connection pool with sophisticated lifecycle management
const databasePool = RcMap.make({
  lookup: (connectionString: string) =>
    Effect.acquireRelease(
      Effect.gen(function* () {
        const connectionId = `conn-${Math.random().toString(36).substr(2, 9)}`
        yield* Effect.log(`🔌 Opening database connection: ${connectionId}`)
        
        // Simulate connection establishment
        yield* Effect.sleep("200 millis")
        
        const connection: DatabaseConnection = {
          id: connectionId,
          connectionString,
          query: <T>(sql: string, params?: unknown[]) =>
            Effect.gen(function* () {
              yield* Effect.log(`📊 Executing query on ${connectionId}: ${sql}`)
              yield* Effect.sleep("50 millis") // Simulate query time
              return { rows: [`Result for ${sql}`], count: 1 } as T
            }),
          transaction: <T>(work: (conn: DatabaseConnection) => Effect.Effect<T>) =>
            Effect.gen(function* () {
              yield* Effect.log(`🔄 Starting transaction on ${connectionId}`)
              
              // In a real implementation, you'd start a transaction here
              const result = yield* work(connection)
              
              yield* Effect.log(`✅ Committing transaction on ${connectionId}`)
              return result
            })
        }
        
        return connection
      }),
      (connection) =>
        Effect.gen(function* () {
          yield* Effect.log(`🔌 Closing database connection: ${connection.id}`)
          yield* Effect.sleep("100 millis") // Simulate cleanup
        })
    ),
  idleTimeToLive: Duration.minutes(10),
  capacity: 10 // Maximum 10 connections
})

// Repository pattern using the connection pool
const makeUserRepository = Effect.gen(function* () {
  const pool = yield* databasePool
  
  const findById = (connectionString: string, id: string) =>
    RcMap.get(pool, connectionString).pipe(
      Effect.flatMap((conn) =>
        conn.query("SELECT * FROM users WHERE id = ?", [id])
      ),
      Effect.scoped
    )
  
  const findByEmail = (connectionString: string, email: string) =>
    RcMap.get(pool, connectionString).pipe(
      Effect.flatMap((conn) =>
        conn.query("SELECT * FROM users WHERE email = ?", [email])
      ),
      Effect.scoped
    )
  
  const create = (connectionString: string, userData: { name: string; email: string }) =>
    RcMap.get(pool, connectionString).pipe(
      Effect.flatMap((conn) =>
        conn.transaction((txConn) =>
          Effect.gen(function* () {
            // Multiple operations in a transaction
            yield* txConn.query("INSERT INTO users (name, email) VALUES (?, ?)", [
              userData.name,
              userData.email
            ])
            return yield* txConn.query("SELECT LAST_INSERT_ID() as id")
          })
        )
      ),
      Effect.scoped
    )
  
  const findUserWithOrders = (connectionString: string, userId: string) =>
    RcMap.get(pool, connectionString).pipe(
      Effect.flatMap((conn) =>
        // Multiple queries sharing the same connection
        Effect.all([
          conn.query("SELECT * FROM users WHERE id = ?", [userId]),
          conn.query("SELECT * FROM orders WHERE user_id = ?", [userId])
        ])
      ),
      Effect.scoped
    )
  
  return { findById, findByEmail, create, findUserWithOrders } as const
})

// Service layer example
const userService = Effect.gen(function* () {
  const repo = yield* makeUserRepository
  const connectionString = "postgresql://localhost:5432/myapp"
  
  const getUserProfile = (userId: string) =>
    Effect.gen(function* () {
      const [user, orders] = yield* repo.findUserWithOrders(connectionString, userId)
      return { user, orders, totalOrders: orders.count }
    })
  
  const createUserWithValidation = (userData: { name: string; email: string }) =>
    Effect.gen(function* () {
      // Check if user exists (uses connection from pool)
      const existing = yield* repo.findByEmail(connectionString, userData.email)
      
      if (existing.count > 0) {
        return yield* Effect.fail(new Error("User already exists"))
      }
      
      // Create user (may reuse the same connection or get a new one)
      return yield* repo.create(connectionString, userData)
    })
  
  return { getUserProfile, createUserWithValidation } as const
})
```

### Example 3: File System Resource Manager

Managing file system resources with automatic cleanup and concurrent access control.

```typescript
import { Effect, RcMap, Duration, Scope } from "effect"

interface FileResource {
  readonly path: string
  readonly descriptor: number
  readonly read: (offset: number, length: number) => Effect.Effect<string>
  readonly write: (data: string, offset?: number) => Effect.Effect<void>
  readonly stat: () => Effect.Effect<{ size: number; modified: Date }>
}

// File resource manager with lock-free concurrent access
const fileResourceMap = RcMap.make({
  lookup: (filePath: string) =>
    Effect.acquireRelease(
      Effect.gen(function* () {
        yield* Effect.log(`📁 Opening file resource: ${filePath}`)
        
        // Simulate file opening
        yield* Effect.sleep("50 millis")
        const descriptor = Math.floor(Math.random() * 1000) + 100
        
        const fileResource: FileResource = {
          path: filePath,
          descriptor,
          read: (offset: number, length: number) =>
            Effect.gen(function* () {
              yield* Effect.log(`📖 Reading ${length} bytes from ${filePath} at offset ${offset}`)
              yield* Effect.sleep("10 millis")
              return `File content from ${filePath} (${length} bytes)`
            }),
          write: (data: string, offset = 0) =>
            Effect.gen(function* () {
              yield* Effect.log(`✏️ Writing ${data.length} bytes to ${filePath} at offset ${offset}`)
              yield* Effect.sleep("20 millis")
            }),
          stat: () =>
            Effect.gen(function* () {
              yield* Effect.log(`📊 Getting stats for ${filePath}`)
              return {
                size: Math.floor(Math.random() * 10000),
                modified: new Date()
              }
            })
        }
        
        return fileResource
      }),
      (resource) =>
        Effect.gen(function* () {
          yield* Effect.log(`📁 Closing file resource: ${resource.path}`)
          yield* Effect.sleep("30 millis")
        })
    ),
  idleTimeToLive: Duration.minutes(2),
  capacity: 50
})

// File operations service
const makeFileService = Effect.gen(function* () {
  const resourceMap = yield* fileResourceMap
  
  const readFile = (path: string) =>
    RcMap.get(resourceMap, path).pipe(
      Effect.flatMap((resource) =>
        Effect.gen(function* () {
          const stats = yield* resource.stat()
          return yield* resource.read(0, stats.size)
        })
      ),
      Effect.scoped
    )
  
  const writeFile = (path: string, content: string) =>
    RcMap.get(resourceMap, path).pipe(
      Effect.flatMap((resource) => resource.write(content)),
      Effect.scoped
    )
  
  const copyFile = (sourcePath: string, destPath: string) =>
    Effect.gen(function* () {
      // Both resources managed automatically
      const content = yield* RcMap.get(resourceMap, sourcePath).pipe(
        Effect.flatMap((source) => source.read(0, 1000000)), // Read up to 1MB
        Effect.scoped
      )
      
      return yield* RcMap.get(resourceMap, destPath).pipe(
        Effect.flatMap((dest) => dest.write(content)),
        Effect.scoped
      )
    })
  
  const processMultipleFiles = (paths: string[]) =>
    Effect.gen(function* () {
      // Concurrent file operations with shared resources
      return yield* Effect.all(
        paths.map((path) =>
          RcMap.get(resourceMap, path).pipe(
            Effect.flatMap((resource) =>
              Effect.gen(function* () {
                const stats = yield* resource.stat()
                const content = yield* resource.read(0, Math.min(stats.size, 1000))
                return { path, size: stats.size, preview: content }
              })
            ),
            Effect.scoped
          )
        ),
        { concurrency: 10 }
      )
    })
  
  const appendToFile = (path: string, data: string) =>
    RcMap.get(resourceMap, path).pipe(
      Effect.flatMap((resource) =>
        Effect.gen(function* () {
          const stats = yield* resource.stat()
          return yield* resource.write(data, stats.size)
        })
      ),
      Effect.scoped
    )
  
  return {
    readFile,
    writeFile,
    copyFile,
    processMultipleFiles,
    appendToFile
  } as const
})

// Batch file processing example
const batchFileProcessor = Effect.gen(function* () {
  const fileService = yield* makeFileService
  
  const processLogFiles = (logPaths: string[]) =>
    Effect.gen(function* () {
      // Process multiple log files concurrently
      // Each file resource is shared across operations and automatically managed
      const results = yield* Effect.all(
        logPaths.map((path) =>
          Effect.gen(function* () {
            const content = yield* fileService.readFile(path)
            const processedPath = path.replace('.log', '.processed.log')
            const processedContent = `PROCESSED: ${content}`
            yield* fileService.writeFile(processedPath, processedContent)
            return { original: path, processed: processedPath }
          })
        ),
        { concurrency: 5 }
      )
      
      return results
    })
  
  return { processLogFiles } as const
})
```

## Advanced Features Deep Dive

### Capacity Management and Eviction

RcMap supports capacity limits with automatic eviction of least-recently-used resources.

```typescript
import { Effect, RcMap, Cause } from "effect"

// Limited capacity map with LRU eviction
const limitedResourceMap = RcMap.make({
  lookup: (key: string) =>
    Effect.gen(function* () {
      yield* Effect.log(`Creating expensive resource for key: ${key}`)
      yield* Effect.sleep("500 millis") // Expensive creation
      return `resource-${key}`
    }),
  capacity: 3, // Only 3 resources max
  idleTimeToLive: Duration.minutes(1)
})

// Understanding capacity exceeded behavior
const capacityExample = Effect.gen(function* () {
  const map = yield* limitedResourceMap
  
  // Fill up the capacity
  const resource1 = yield* RcMap.get(map, "key1").pipe(Effect.scoped)
  const resource2 = yield* RcMap.get(map, "key2").pipe(Effect.scoped)
  const resource3 = yield* RcMap.get(map, "key3").pipe(Effect.scoped)
  
  // This will cause eviction of least recently used
  const resource4 = yield* RcMap.get(map, "key4").pipe(
    Effect.catchTag("ExceededCapacityException", (cause) =>
      Effect.gen(function* () {
        yield* Effect.log("Capacity exceeded, handling gracefully")
        return "fallback-resource"
      })
    ),
    Effect.scoped
  )
  
  return [resource1, resource2, resource3, resource4]
})
```

### Key Management and Inspection

RcMap provides utilities for inspecting and managing keys in the map.

```typescript
import { Effect, RcMap } from "effect"

// Resource map with key inspection capabilities
const inspectableMap = RcMap.make({
  lookup: (key: string) =>
    Effect.acquireRelease(
      Effect.succeed(`resource-${key}`),
      (resource) => Effect.log(`Cleaning up: ${resource}`)
    ),
  idleTimeToLive: Duration.seconds(30)
})

// Key management utilities
const keyManagementExample = Effect.gen(function* () {
  const map = yield* inspectableMap
  
  // Create some resources
  yield* RcMap.get(map, "active-1").pipe(Effect.scoped)
  yield* RcMap.get(map, "active-2").pipe(Effect.scoped)
  
  // Inspect current keys
  const currentKeys = yield* RcMap.keys(map)
  yield* Effect.log(`Current keys: ${currentKeys.join(", ")}`)
  
  // Touch a key to refresh its idle timer
  yield* RcMap.touch(map, "active-1")
  
  // Manually invalidate a resource
  yield* RcMap.invalidate(map, "active-2")
  
  // Check keys again
  const keysAfterInvalidation = yield* RcMap.keys(map)
  yield* Effect.log(`Keys after invalidation: ${keysAfterInvalidation.join(", ")}`)
  
  return keysAfterInvalidation
})
```

### Complex Key Types with Equal and Hash

Using complex objects as keys with proper equality and hashing.

```typescript
import { Effect, RcMap, Equal, Hash, Data } from "effect"

// Complex key type with custom equality
class DatabaseConfig extends Data.Class<{
  readonly host: string
  readonly port: number
  readonly database: string
  readonly username: string
}> {
  // Custom connection string for display
  get connectionString() {
    return `${this.username}@${this.host}:${this.port}/${this.database}`
  }
}

// Connection pool using complex keys
const complexKeyMap = RcMap.make({
  lookup: (config: DatabaseConfig) =>
    Effect.acquireRelease(
      Effect.gen(function* () {
        yield* Effect.log(`Connecting to: ${config.connectionString}`)
        yield* Effect.sleep("300 millis")
        return {
          config,
          id: `conn-${Math.random().toString(36).substr(2, 9)}`,
          query: (sql: string) => Effect.succeed(`Result for: ${sql}`)
        }
      }),
      (connection) =>
        Effect.log(`Disconnecting: ${connection.id}`)
    ),
  idleTimeToLive: Duration.minutes(5)
})

// Usage with complex keys
const complexKeyExample = Effect.gen(function* () {
  const map = yield* complexKeyMap
  
  const config1 = new DatabaseConfig({
    host: "localhost",
    port: 5432,
    database: "app_db",
    username: "admin"
  })
  
  const config2 = new DatabaseConfig({
    host: "localhost",
    port: 5432,
    database: "app_db",
    username: "admin"
  })
  
  // These will be treated as the same key due to structural equality
  const conn1 = yield* RcMap.get(map, config1).pipe(Effect.scoped)
  const conn2 = yield* RcMap.get(map, config2).pipe(Effect.scoped) // Reuses same connection
  
  yield* Effect.log(`Connection 1 ID: ${conn1.id}`)
  yield* Effect.log(`Connection 2 ID: ${conn2.id}`) // Same ID
  
  return [conn1, conn2]
})
```

### Error Handling and Recovery

Comprehensive error handling patterns with RcMap.

```typescript
import { Effect, RcMap, Duration } from "effect"

// Error types for resource creation
class ResourceCreationError extends Data.TaggedError("ResourceCreationError")<{
  readonly key: string
  readonly cause: string
}> {}

class ResourceValidationError extends Data.TaggedError("ResourceValidationError")<{
  readonly key: string
  readonly details: string
}> {}

// Resource map with error handling
const errorHandlingMap = RcMap.make({
  lookup: (key: string) =>
    Effect.gen(function* () {
      // Simulate potential failures
      if (key.includes("invalid")) {
        return yield* Effect.fail(
          new ResourceValidationError({
            key,
            details: "Key contains invalid characters"
          })
        )
      }
      
      if (key.includes("timeout")) {
        return yield* Effect.fail(
          new ResourceCreationError({
            key,
            details: "Resource creation timed out"
          })
        )
      }
      
      // Simulate random failures
      if (Math.random() < 0.1) {
        return yield* Effect.fail(
          new ResourceCreationError({
            key,
            details: "Random resource creation failure"
          })
        )
      }
      
      yield* Effect.sleep("100 millis")
      return `healthy-resource-${key}`
    }),
  idleTimeToLive: Duration.minutes(1)
})

// Error handling patterns
const errorHandlingExample = Effect.gen(function* () {
  const map = yield* errorHandlingMap
  
  // Successful resource access
  const healthyResource = yield* RcMap.get(map, "healthy-key").pipe(
    Effect.scoped
  )
  
  // Error handling with retry
  const retryableResource = yield* RcMap.get(map, "timeout-key").pipe(
    Effect.retry({
      times: 3,
      delay: Duration.seconds(1)
    }),
    Effect.catchTag("ResourceCreationError", (error) =>
      Effect.gen(function* () {
        yield* Effect.log(`Failed to create resource: ${error.details}`)
        return "fallback-resource"
      })
    ),
    Effect.scoped
  )
  
  // Graceful degradation
  const withFallback = yield* RcMap.get(map, "invalid-key").pipe(
    Effect.catchAll((error) =>
      Effect.gen(function* () {
        yield* Effect.log(`Using fallback due to error: ${error}`)
        return "fallback-resource"
      })
    ),
    Effect.scoped
  )
  
  return {
    healthy: healthyResource,
    retried: retryableResource,
    fallback: withFallback
  }
})
```

## Practical Patterns & Best Practices

### Pattern 1: Resource Factory with Configuration

Create reusable resource factories that can be configured for different environments.

```typescript
import { Effect, RcMap, Layer, Context, Duration } from "effect"

// Configuration service
class ResourceConfig extends Context.Tag("ResourceConfig")<
  ResourceConfig,
  {
    readonly maxConnections: number
    readonly idleTimeout: Duration.Duration
    readonly connectionTimeout: Duration.Duration
  }
>() {}

// Factory function for creating configured resource maps
const makeResourceMap = <K, A, E, R>(
  options: {
    readonly name: string
    readonly lookup: (key: K) => Effect.Effect<A, E, R>
  }
) =>
  Effect.gen(function* () {
    const config = yield* ResourceConfig
    
    return yield* RcMap.make({
      lookup: options.lookup,
      capacity: config.maxConnections,
      idleTimeToLive: config.idleTimeout
    })
  })

// Usage in different environments
const developmentLayer = Layer.succeed(ResourceConfig, {
  maxConnections: 5,
  idleTimeout: Duration.seconds(30),
  connectionTimeout: Duration.seconds(5)
})

const productionLayer = Layer.succeed(ResourceConfig, {
  maxConnections: 100,
  idleTimeout: Duration.minutes(10),
  connectionTimeout: Duration.seconds(30)
})

// Service using the factory
const databaseService = Effect.gen(function* () {
  const connectionMap = yield* makeResourceMap({
    name: "database-connections",
    lookup: (connectionString: string) =>
      Effect.acquireRelease(
        Effect.gen(function* () {
          const config = yield* ResourceConfig
          yield* Effect.log(`Opening DB connection (timeout: ${config.connectionTimeout})`)
          yield* Effect.sleep(config.connectionTimeout)
          return { connectionString, id: `db-${Date.now()}` }
        }),
        (conn) => Effect.log(`Closing DB connection: ${conn.id}`)
      )
  })
  
  return {
    query: (connectionString: string, sql: string) =>
      RcMap.get(connectionMap, connectionString).pipe(
        Effect.flatMap((conn) =>
          Effect.gen(function* () {
            yield* Effect.log(`Executing query on ${conn.id}: ${sql}`)
            return `Query result for: ${sql}`
          })
        ),
        Effect.scoped
      )
  }
})
```

### Pattern 2: Hierarchical Resource Management

Managing resources with dependencies and hierarchical cleanup.

```typescript
import { Effect, RcMap, Scope } from "effect"

// Hierarchical resource structure
interface DatabaseCluster {
  readonly clusterId: string
  readonly nodes: DatabaseNode[]
  readonly selectNode: () => DatabaseNode
}

interface DatabaseNode {
  readonly nodeId: string
  readonly clusterId: string
  readonly connection: DatabaseConnection
}

interface DatabaseConnection {
  readonly connectionId: string
  readonly execute: (query: string) => Effect.Effect<unknown>
}

// Cluster-level resource map
const clusterMap = RcMap.make({
  lookup: (clusterId: string) =>
    Effect.acquireRelease(
      Effect.gen(function* () {
        yield* Effect.log(`🏗️ Setting up database cluster: ${clusterId}`)
        
        // Create multiple nodes in the cluster
        const nodes: DatabaseNode[] = []
        for (let i = 0; i < 3; i++) {
          const nodeId = `${clusterId}-node-${i}`
          const connectionId = `${nodeId}-conn`
          
          const connection: DatabaseConnection = {
            connectionId,
            execute: (query: string) =>
              Effect.gen(function* () {
                yield* Effect.log(`Executing on ${connectionId}: ${query}`)
                return `Result from ${connectionId}`
              })
          }
          
          nodes.push({
            nodeId,
            clusterId,
            connection
          })
        }
        
        const cluster: DatabaseCluster = {
          clusterId,
          nodes,
          selectNode: () => nodes[Math.floor(Math.random() * nodes.length)]
        }
        
        return cluster
      }),
      (cluster) =>
        Effect.gen(function* () {
          yield* Effect.log(`🏗️ Tearing down database cluster: ${cluster.clusterId}`)
          // Cleanup would happen here for each node
        })
    ),
  idleTimeToLive: Duration.minutes(15)
})

// Node-level resource map for individual connections
const nodeConnectionMap = RcMap.make({
  lookup: (nodeId: string) =>
    Effect.acquireRelease(
      Effect.gen(function* () {
        yield* Effect.log(`🔌 Opening dedicated connection to node: ${nodeId}`)
        yield* Effect.sleep("200 millis")
        return {
          nodeId,
          connectionId: `dedicated-${nodeId}-${Date.now()}`,
          execute: (query: string) =>
            Effect.gen(function* () {
              yield* Effect.log(`Dedicated execution on ${nodeId}: ${query}`)
              return `Dedicated result from ${nodeId}`
            })
        }
      }),
      (conn) =>
        Effect.log(`🔌 Closing dedicated connection: ${conn.connectionId}`)
    ),
  idleTimeToLive: Duration.minutes(5)
})

// Service using hierarchical resources
const clusterService = Effect.gen(function* () {
  const clusters = yield* clusterMap
  const nodeConnections = yield* nodeConnectionMap
  
  // High-level cluster operations
  const executeOnCluster = (clusterId: string, query: string) =>
    RcMap.get(clusters, clusterId).pipe(
      Effect.flatMap((cluster) => {
        const node = cluster.selectNode()
        return node.connection.execute(query)
      }),
      Effect.scoped
    )
  
  // Direct node operations with dedicated connections
  const executeOnNode = (nodeId: string, query: string) =>
    RcMap.get(nodeConnections, nodeId).pipe(
      Effect.flatMap((conn) => conn.execute(query)),
      Effect.scoped
    )
  
  // Batch operations across cluster
  const batchExecute = (clusterId: string, queries: string[]) =>
    RcMap.get(clusters, clusterId).pipe(
      Effect.flatMap((cluster) =>
        Effect.all(
          queries.map((query) => {
            const node = cluster.selectNode()
            return node.connection.execute(query)
          }),
          { concurrency: 5 }
        )
      ),
      Effect.scoped
    )
  
  return {
    executeOnCluster,
    executeOnNode,
    batchExecute
  } as const
})
```

### Pattern 3: Resource Monitoring and Health Checks

Implementing monitoring and health checking for managed resources.

```typescript
import { Effect, RcMap, Schedule, Ref, Duration } from "effect"

interface MonitoredResource {
  readonly id: string
  readonly createdAt: Date
  readonly lastAccessed: Date
  readonly accessCount: number
  readonly isHealthy: boolean
  readonly execute: (operation: string) => Effect.Effect<string>
  readonly healthCheck: () => Effect.Effect<boolean>
}

// Monitored resource map with health tracking
const monitoredResourceMap = RcMap.make({
  lookup: (key: string) =>
    Effect.acquireRelease(
      Effect.gen(function* () {
        const id = `resource-${key}-${Date.now()}`
        const accessCountRef = yield* Ref.make(0)
        const lastAccessedRef = yield* Ref.make(new Date())
        const healthyRef = yield* Ref.make(true)
        
        yield* Effect.log(`🏗️ Creating monitored resource: ${id}`)
        
        const resource: MonitoredResource = {
          id,
          createdAt: new Date(),
          lastAccessed: new Date(),
          accessCount: 0,
          isHealthy: true,
          execute: (operation: string) =>
            Effect.gen(function* () {
              // Update access tracking
              yield* Ref.update(accessCountRef, (count) => count + 1)
              yield* Ref.set(lastAccessedRef, new Date())
              
              // Simulate potential health issues
              if (Math.random() < 0.05) {
                yield* Ref.set(healthyRef, false)
                return yield* Effect.fail(new Error(`Resource ${id} is unhealthy`))
              }
              
              yield* Effect.log(`Executing ${operation} on ${id}`)
              return `Result from ${id}: ${operation}`
            }),
          healthCheck: () =>
            Effect.gen(function* () {
              const isHealthy = yield* Ref.get(healthyRef)
              
              // Simulate health check
              yield* Effect.sleep("10 millis")
              
              if (!isHealthy && Math.random() < 0.3) {
                // Resource recovered
                yield* Ref.set(healthyRef, true)
                yield* Effect.log(`Resource ${id} recovered`)
                return true
              }
              
              return isHealthy
            })
        }
        
        // Start background health monitoring
        const healthMonitor = Effect.gen(function* () {
          while (true) {
            yield* resource.healthCheck()
            yield* Effect.sleep("30 seconds")
          }
        }).pipe(
          Effect.forkDaemon
        )
        
        yield* healthMonitor
        return resource
      }),
      (resource) =>
        Effect.gen(function* () {
          yield* Effect.log(`🏗️ Disposing monitored resource: ${resource.id}`)
          // Cleanup monitoring, close connections, etc.
        })
    ),
  idleTimeToLive: Duration.minutes(10)
})

// Health monitoring service
const healthMonitoringService = Effect.gen(function* () {
  const resourceMap = yield* monitoredResourceMap
  
  const executeWithHealthCheck = (key: string, operation: string) =>
    RcMap.get(resourceMap, key).pipe(
      Effect.flatMap((resource) =>
        Effect.gen(function* () {
          const isHealthy = yield* resource.healthCheck()
          
          if (!isHealthy) {
            yield* Effect.log(`Resource ${resource.id} failed health check, retrying...`)
            // Could implement retry logic or circuit breaker here
            return yield* Effect.fail(new Error(`Resource ${key} is unhealthy`))
          }
          
          return yield* resource.execute(operation)
        })
      ),
      Effect.retry({
        times: 3,
        delay: Duration.seconds(1)
      }),
      Effect.scoped
    )
  
  const getResourceStats = (key: string) =>
    RcMap.get(resourceMap, key).pipe(
      Effect.map((resource) => ({
        id: resource.id,
        createdAt: resource.createdAt,
        lastAccessed: resource.lastAccessed,
        accessCount: resource.accessCount,
        isHealthy: resource.isHealthy
      })),
      Effect.scoped
    )
  
  const performHealthSweep = Effect.gen(function* () {
    const keys = yield* RcMap.keys(resourceMap)
    yield* Effect.log(`Performing health sweep on ${keys.length} resources`)
    
    const results = yield* Effect.all(
      keys.map((key) =>
        RcMap.get(resourceMap, key).pipe(
          Effect.flatMap((resource) => resource.healthCheck()),
          Effect.map((healthy) => ({ key, healthy })),
          Effect.scoped
        )
      ),
      { concurrency: 10 }
    )
    
    const unhealthyKeys = results.filter((r) => !r.healthy).map((r) => r.key)
    
    if (unhealthyKeys.length > 0) {
      yield* Effect.log(`Found ${unhealthyKeys.length} unhealthy resources: ${unhealthyKeys.join(", ")}`)
      
      // Optionally invalidate unhealthy resources
      yield* Effect.all(
        unhealthyKeys.map((key) => RcMap.invalidate(resourceMap, key))
      )
    }
    
    return results
  })
  
  return {
    executeWithHealthCheck,
    getResourceStats,
    performHealthSweep
  } as const
})

// Scheduled health monitoring
const scheduledHealthMonitoring = Effect.gen(function* () {
  const healthService = yield* healthMonitoringService
  
  // Run health sweep every 5 minutes
  return yield* healthService.performHealthSweep.pipe(
    Effect.repeat(Schedule.fixed(Duration.minutes(5))),
    Effect.forkDaemon
  )
})
```

## Integration Examples

### Integration with Layer and Dependency Injection

Using RcMap with Effect's Layer system for proper dependency management.

```typescript
import { Effect, RcMap, Layer, Context, Duration } from "effect"

// Service definitions
class DatabaseService extends Context.Tag("DatabaseService")<
  DatabaseService,
  {
    readonly query: (sql: string) => Effect.Effect<unknown>
    readonly transaction: <T>(work: Effect.Effect<T>) => Effect.Effect<T>
  }
>() {}

class CacheService extends Context.Tag("CacheService")<
  CacheService,
  {
    readonly get: (key: string) => Effect.Effect<Option.Option<string>>
    readonly set: (key: string, value: string) => Effect.Effect<void>
  }
>() {}

class ResourceMapService extends Context.Tag("ResourceMapService")<
  ResourceMapService,
  RcMap.RcMap<string, { id: string; type: string }>
>() {}

// Layer implementations using RcMap
const resourceMapLayer = Layer.effect(
  ResourceMapService,
  RcMap.make({
    lookup: (key: string) =>
      Effect.acquireRelease(
        Effect.gen(function* () {
          yield* Effect.log(`Creating resource for key: ${key}`)
          return {
            id: `resource-${key}-${Date.now()}`,
            type: key.startsWith("db") ? "database" : "cache"
          }
        }),
        (resource) =>
          Effect.log(`Cleaning up resource: ${resource.id}`)
      ),
    idleTimeToLive: Duration.minutes(5)
  })
)

const databaseServiceLayer = Layer.effect(
  DatabaseService,
  Effect.gen(function* () {
    const resourceMap = yield* ResourceMapService
    
    return {
      query: (sql: string) =>
        RcMap.get(resourceMap, "db-primary").pipe(
          Effect.flatMap((resource) =>
            Effect.gen(function* () {
              yield* Effect.log(`Executing query with ${resource.id}: ${sql}`)
              return { rows: [`Result for ${sql}`] }
            })
          ),
          Effect.scoped
        ),
      transaction: <T>(work: Effect.Effect<T>) =>
        RcMap.get(resourceMap, "db-transaction").pipe(
          Effect.flatMap(() => work), // Simplified transaction logic
          Effect.scoped
        )
    }
  })
)

const cacheServiceLayer = Layer.effect(
  CacheService,
  Effect.gen(function* () {
    const resourceMap = yield* ResourceMapService
    const cache = new Map<string, string>()
    
    return {
      get: (key: string) =>
        RcMap.get(resourceMap, "cache-primary").pipe(
          Effect.flatMap((resource) =>
            Effect.gen(function* () {
              yield* Effect.log(`Cache GET with ${resource.id}: ${key}`)
              return Option.fromNullable(cache.get(key))
            })
          ),
          Effect.scoped
        ),
      set: (key: string, value: string) =>
        RcMap.get(resourceMap, "cache-primary").pipe(
          Effect.flatMap((resource) =>
            Effect.gen(function* () {
              yield* Effect.log(`Cache SET with ${resource.id}: ${key}`)
              cache.set(key, value)
            })
          ),
          Effect.scoped
        )
    }
  })
)

// Application layer combining all services
const applicationLayer = Layer.empty.pipe(
  Layer.provide(resourceMapLayer),
  Layer.provide(databaseServiceLayer),
  Layer.provide(cacheServiceLayer)
)

// Application using the layered services
const application = Effect.gen(function* () {
  const db = yield* DatabaseService
  const cache = yield* CacheService
  
  // Use database
  const queryResult = yield* db.query("SELECT * FROM users")
  
  // Use cache
  yield* cache.set("user:123", JSON.stringify(queryResult))
  const cached = yield* cache.get("user:123")
  
  return { queryResult, cached }
}).pipe(Effect.provide(applicationLayer))
```

### Integration with Testing and Mocking

Testing strategies for RcMap-based services with proper resource isolation.

```typescript
import { Effect, RcMap, Layer, TestContext, Duration } from "effect"

// Mock resource for testing
interface MockResource {
  readonly id: string
  readonly calls: string[]
  readonly addCall: (operation: string) => Effect.Effect<void>
  readonly getCalls: () => Effect.Effect<string[]>
}

// Test utilities
const makeTestResourceMap = (resources: Record<string, MockResource>) =>
  RcMap.make({
    lookup: (key: string) => {
      const resource = resources[key]
      if (!resource) {
        return Effect.fail(new Error(`Test resource not found: ${key}`))
      }
      return Effect.succeed(resource)
    },
    idleTimeToLive: Duration.seconds(1) // Short TTL for testing
  })

// Service under test
const makeServiceUnderTest = (resourceMap: RcMap.RcMap<string, MockResource>) =>
  Effect.gen(function* () {
    return {
      performOperation: (resourceKey: string, operation: string) =>
        RcMap.get(resourceMap, resourceKey).pipe(
          Effect.flatMap((resource) =>
            Effect.gen(function* () {
              yield* resource.addCall(operation)
              return `Operation ${operation} completed with ${resource.id}`
            })
          ),
          Effect.scoped
        ),
      
      batchOperations: (resourceKey: string, operations: string[]) =>
        RcMap.get(resourceMap, resourceKey).pipe(
          Effect.flatMap((resource) =>
            Effect.all(
              operations.map((op) => resource.addCall(op)),
              { concurrency: 3 }
            )
          ),
          Effect.scoped
        )
    } as const
  })

// Test suite
const testSuite = Effect.gen(function* () {
  // Create mock resources
  const mockResources: Record<string, MockResource> = {
    "test-resource-1": {
      id: "mock-1",
      calls: [],
      addCall: (operation: string) =>
        Effect.sync(() => {
          mockResources["test-resource-1"].calls.push(operation)
        }),
      getCalls: () => Effect.succeed([...mockResources["test-resource-1"].calls])
    },
    "test-resource-2": {
      id: "mock-2", 
      calls: [],
      addCall: (operation: string) =>
        Effect.sync(() => {
          mockResources["test-resource-2"].calls.push(operation)
        }),
      getCalls: () => Effect.succeed([...mockResources["test-resource-2"].calls])
    }
  }
  
  const testResourceMap = yield* makeTestResourceMap(mockResources)
  const service = yield* makeServiceUnderTest(testResourceMap)
  
  // Test 1: Single operation
  yield* Effect.log("Test 1: Single operation")
  const result1 = yield* service.performOperation("test-resource-1", "read")
  const calls1 = yield* mockResources["test-resource-1"].getCalls()
  yield* Effect.log(`Result: ${result1}`)
  yield* Effect.log(`Calls: ${calls1.join(", ")}`)
  
  // Test 2: Multiple operations on same resource (should reuse)
  yield* Effect.log("Test 2: Multiple operations on same resource")
  yield* Effect.all([
    service.performOperation("test-resource-1", "write"),
    service.performOperation("test-resource-1", "update")
  ])
  const calls2 = yield* mockResources["test-resource-1"].getCalls()
  yield* Effect.log(`Total calls after reuse: ${calls2.join(", ")}`)
  
  // Test 3: Batch operations
  yield* Effect.log("Test 3: Batch operations")
  yield* service.batchOperations("test-resource-2", ["op1", "op2", "op3"])
  const calls3 = yield* mockResources["test-resource-2"].getCalls()
  yield* Effect.log(`Batch operation calls: ${calls3.join(", ")}`)
  
  // Test 4: Resource isolation
  yield* Effect.log("Test 4: Resource isolation")
  const keys = yield* RcMap.keys(testResourceMap)
  yield* Effect.log(`Active resources: ${keys.join(", ")}`)
  
  return {
    test1: { result: result1, calls: calls1 },
    test2: { calls: calls2 },
    test3: { calls: calls3 },
    activeKeys: keys
  }
}).pipe(
  Effect.provide(TestContext.TestContext),
  Effect.scoped
)

// Property-based testing with Effect
const propertyBasedTest = Effect.gen(function* () {
  const testData = Array.from({ length: 10 }, (_, i) => ({
    key: `resource-${i % 3}`, // Will cause resource reuse
    operation: `operation-${i}`
  }))
  
  const mockResource: MockResource = {
    id: "property-test-resource",
    calls: [],
    addCall: (operation: string) =>
      Effect.sync(() => {
        mockResource.calls.push(operation)
      }),
    getCalls: () => Effect.succeed([...mockResource.calls])
  }
  
  const resourceMap = yield* RcMap.make({
    lookup: (key: string) => Effect.succeed(mockResource),
    idleTimeToLive: Duration.seconds(1)
  })
  
  const service = yield* makeServiceUnderTest(resourceMap)
  
  // Execute all operations
  yield* Effect.all(
    testData.map(({ key, operation }) =>
      service.performOperation(key, operation)
    ),
    { concurrency: 5 }
  )
  
  const finalCalls = yield* mockResource.getCalls()
  
  // Verify all operations were called
  const expectedOperations = testData.map(d => d.operation)
  const allOperationsExecuted = expectedOperations.every(op => 
    finalCalls.includes(op)
  )
  
  yield* Effect.log(`Property test - All operations executed: ${allOperationsExecuted}`)
  yield* Effect.log(`Expected: ${expectedOperations.length}, Actual: ${finalCalls.length}`)
  
  return { allOperationsExecuted, expectedCount: expectedOperations.length, actualCount: finalCalls.length }
}).pipe(Effect.scoped)
```

### Integration with External Libraries

Adapting third-party libraries to work with RcMap resource management.

```typescript
import { Effect, RcMap, Duration } from "effect"

// Simulated third-party library interfaces
interface ThirdPartyConnection {
  connect(): Promise<void>
  disconnect(): Promise<void>
  send(data: string): Promise<string>
  isConnected(): boolean
}

interface RedisClient {
  connect(): Promise<void>
  disconnect(): Promise<void>
  get(key: string): Promise<string | null>
  set(key: string, value: string): Promise<void>
  ping(): Promise<string>
}

// Adapter functions for third-party libraries
const adaptThirdPartyConnection = (config: { host: string; port: number }) =>
  Effect.acquireRelease(
    Effect.gen(function* () {
      yield* Effect.log(`Connecting to ${config.host}:${config.port}`)
      
      // Simulate third-party connection
      const connection: ThirdPartyConnection = {
        connect: async () => {
          // Third-party connection logic
          await new Promise(resolve => setTimeout(resolve, 100))
        },
        disconnect: async () => {
          await new Promise(resolve => setTimeout(resolve, 50))
        },
        send: async (data: string) => {
          return `Response to: ${data}`
        },
        isConnected: () => true
      }
      
      // Convert Promise to Effect
      yield* Effect.promise(() => connection.connect())
      
      return connection
    }),
    (connection) =>
      Effect.gen(function* () {
        yield* Effect.log(`Disconnecting from third-party service`)
        yield* Effect.promise(() => connection.disconnect())
      }).pipe(
        Effect.catchAll((error) =>
          Effect.log(`Error during disconnect: ${error}`)
        )
      )
  )

const adaptRedisClient = (connectionString: string) =>
  Effect.acquireRelease(
    Effect.gen(function* () {
      yield* Effect.log(`Connecting to Redis: ${connectionString}`)
      
      // Simulate Redis client
      const client: RedisClient = {
        connect: async () => {
          await new Promise(resolve => setTimeout(resolve, 200))
        },
        disconnect: async () => {
          await new Promise(resolve => setTimeout(resolve, 100))
        },
        get: async (key: string) => {
          return Math.random() > 0.5 ? `value-${key}` : null
        },
        set: async (key: string, value: string) => {
          // Set operation
        },
        ping: async () => "PONG"
      }
      
      yield* Effect.promise(() => client.connect())
      
      // Test connection
      const pingResult = yield* Effect.promise(() => client.ping())
      yield* Effect.log(`Redis connection test: ${pingResult}`)
      
      return client
    }),
    (client) =>
      Effect.gen(function* () {
        yield* Effect.log(`Disconnecting from Redis`)
        yield* Effect.promise(() => client.disconnect())
      }).pipe(
        Effect.catchAll((error) =>
          Effect.log(`Redis disconnect error: ${error}`)
        )
      )
  )

// RcMap for managing third-party connections
const thirdPartyConnectionMap = RcMap.make({
  lookup: (config: { host: string; port: number }) =>
    adaptThirdPartyConnection(config),
  idleTimeToLive: Duration.minutes(10),
  capacity: 20
})

const redisConnectionMap = RcMap.make({
  lookup: (connectionString: string) =>
    adaptRedisClient(connectionString),
  idleTimeToLive: Duration.minutes(15),
  capacity: 10
})

// Service layer using adapted third-party libraries
const externalServiceManager = Effect.gen(function* () {
  const thirdPartyMap = yield* thirdPartyConnectionMap
  const redisMap = yield* redisConnectionMap
  
  const sendToThirdParty = (host: string, port: number, data: string) =>
    RcMap.get(thirdPartyMap, { host, port }).pipe(
      Effect.flatMap((connection) =>
        Effect.promise(() => connection.send(data))
      ),
      Effect.scoped
    )
  
  const cacheOperation = (redisUrl: string, key: string, value?: string) =>
    RcMap.get(redisMap, redisUrl).pipe(
      Effect.flatMap((client) =>
        value
          ? Effect.promise(() => client.set(key, value)).pipe(
              Effect.map(() => `Set ${key} = ${value}`)
            )
          : Effect.promise(() => client.get(key)).pipe(
              Effect.map((result) => result ?? "null")
            )
      ),
      Effect.scoped
    )
  
  const healthCheckThirdParty = (host: string, port: number) =>
    RcMap.get(thirdPartyMap, { host, port }).pipe(
      Effect.flatMap((connection) =>
        Effect.succeed(connection.isConnected())
      ),
      Effect.scoped
    )
  
  const healthCheckRedis = (redisUrl: string) =>
    RcMap.get(redisMap, redisUrl).pipe(
      Effect.flatMap((client) =>
        Effect.promise(() => client.ping()).pipe(
          Effect.map((result) => result === "PONG")
        )
      ),
      Effect.catchAll(() => Effect.succeed(false)),
      Effect.scoped
    )
  
  return {
    sendToThirdParty,
    cacheOperation,
    healthCheckThirdParty,
    healthCheckRedis
  } as const
})

// Usage example integrating multiple external services
const integratedExample = Effect.gen(function* () {
  const serviceManager = yield* externalServiceManager
  
  // Use third-party service
  const thirdPartyResult = yield* serviceManager.sendToThirdParty(
    "api.example.com",
    443,
    "Hello from Effect!"
  )
  
  // Cache the result
  yield* serviceManager.cacheOperation(
    "redis://localhost:6379",
    "third-party-result",
    thirdPartyResult
  )
  
  // Retrieve from cache
  const cachedResult = yield* serviceManager.cacheOperation(
    "redis://localhost:6379",
    "third-party-result"
  )
  
  // Health checks
  const thirdPartyHealthy = yield* serviceManager.healthCheckThirdParty(
    "api.example.com",
    443
  )
  const redisHealthy = yield* serviceManager.healthCheckRedis(
    "redis://localhost:6379"
  )
  
  return {
    thirdPartyResult,
    cachedResult,
    health: {
      thirdParty: thirdPartyHealthy,
      redis: redisHealthy
    }
  }
})
```

## Conclusion

RcMap provides automatic reference-counted resource management, scope-based lifecycle control, and built-in support for capacity limits and idle timeouts for modern TypeScript applications.

Key benefits:
- **Automatic Resource Management** - No manual reference counting or cleanup logic required
- **Scope-Based Lifecycle** - Resources automatically cleaned up when scopes close
- **Concurrent Safety** - Thread-safe resource sharing and deduplication
- **Memory Efficiency** - Automatic eviction based on capacity and idle time
- **Error Resilience** - Proper error handling and resource cleanup even during failures

RcMap is ideal for managing shared resources like database connections, HTTP clients, file handles, and expensive computations where automatic cleanup and resource reuse are critical for application performance and reliability.