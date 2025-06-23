# RcRef: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem RcRef Solves

Managing shared ownership of expensive resources is a common challenge in concurrent applications. Traditional approaches using manual reference counting or cleanup callbacks are error-prone and lead to resource leaks, double-cleanup, and race conditions:

```typescript
// Traditional approach - manual reference counting with cleanup callbacks
class DatabaseConnection {
  private refCount = 0
  private connection: any = null
  private cleanupCallbacks: Array<() => void> = []
  
  async acquire(): Promise<any> {
    if (this.refCount === 0) {
      console.log("Creating new database connection...")
      this.connection = await createExpensiveConnection()
    }
    
    this.refCount++
    
    // Return cleanup function - easily forgotten or called multiple times
    return {
      connection: this.connection,
      release: () => {
        this.refCount--
        
        // Race condition: multiple concurrent releases
        if (this.refCount === 0) {
          // Memory leak: cleanup callbacks might not run
          this.cleanupCallbacks.forEach(cb => cb())
          this.connection?.close()
          this.connection = null
        }
        
        // Double-release bug: refCount goes negative
        if (this.refCount < 0) {
          throw new Error("Double release detected!")
        }
      }
    }
  }
  
  // Manual cleanup registration - easily missed
  onCleanup(callback: () => void) {
    this.cleanupCallbacks.push(callback)
  }
}

// Usage is error-prone
const sharedConnection = new DatabaseConnection()

async function processData() {
  const { connection, release } = await sharedConnection.acquire()
  
  try {
    // Process data...
    // What if an exception occurs? release() might not be called!
    return await connection.query("SELECT * FROM users")
  } finally {
    release() // Easy to forget or call twice
  }
}

// Multiple concurrent operations lead to race conditions
Promise.all([
  processData(),
  processData(),
  processData()
])
```

This approach leads to:
- **Resource Leaks** - Forgotten release calls leave resources alive indefinitely
- **Race Conditions** - Multiple concurrent acquire/release operations cause inconsistent reference counts
- **Double Cleanup** - Resources might be cleaned up multiple times, causing crashes
- **Manual Lifecycle Management** - Developers must remember to properly release every acquired resource
- **Exception Safety Issues** - Exceptions can bypass cleanup code, leading to leaks

### The RcRef Solution

RcRef provides reference-counted resources with automatic disposal, ensuring that expensive resources are shared safely and cleaned up automatically when no longer needed:

```typescript
import { RcRef, Effect, Scope } from "effect"

// Create a reference-counted database connection
const connectionRcRef = RcRef.make({
  acquire: Effect.gen(function* () {
    console.log("Creating database connection...")
    
    // Expensive resource creation with automatic cleanup
    return yield* Effect.acquireRelease(
      Effect.gen(function* () {
        const connection = yield* createDatabaseConnection()
        console.log(`Database connection ${connection.id} created`)
        return connection
      }),
      (connection) => Effect.gen(function* () {
        console.log(`Closing database connection ${connection.id}`)
        yield* connection.close()
      })
    )
  })
})

const processData = Effect.gen(function* () {
  const rcRef = yield* connectionRcRef
  
  // Get the connection - automatically managed within scope
  const connection = yield* RcRef.get(rcRef)
  
  // Use the connection - no manual cleanup needed
  const result = yield* connection.query("SELECT * FROM users")
  
  return result
  // Connection reference automatically released when scope ends
}).pipe(Effect.scoped)

// Multiple concurrent operations share the same underlying resource
const program = Effect.gen(function* () {
  // All three operations share the same database connection
  const results = yield* Effect.all([
    processData,
    processData,
    processData
  ], { concurrency: "unbounded" })
  
  console.log("All operations completed")
  return results
  // Database connection automatically closed when last reference is released
}).pipe(Effect.scoped)
```

### Key Concepts

**Reference Counting**: RcRef automatically tracks how many active references exist to a resource. The resource is created on first access and destroyed when the last reference is released.

**Scope Integration**: RcRef works seamlessly with Effect's Scope system, ensuring that references are automatically released when scopes close, preventing resource leaks.

**Lazy Acquisition**: The underlying resource is only created when first accessed via `RcRef.get`, allowing for efficient resource usage patterns.

**Idle Time-to-Live**: Resources can be configured with an idle timeout, automatically disposing of unused resources after a specified duration to free up system resources.

**Exception Safety**: Reference counting and cleanup are handled automatically by the Effect system, ensuring resources are properly managed even when exceptions occur.

## Basic Usage Patterns

### Creating and Using RcRefs

```typescript
import { RcRef, Effect, Console } from "effect"

// Create an RcRef for an expensive resource
const expensiveResourceRcRef = RcRef.make({
  acquire: Effect.gen(function* () {
    yield* Console.log("Creating expensive resource...")
    
    return yield* Effect.acquireRelease(
      Effect.succeed({
        id: Math.random().toString(36).substr(2, 9),
        data: "expensive computation result",
        createdAt: Date.now()
      }),
      (resource) => Console.log(`Cleaning up resource ${resource.id}`)
    )
  })
})

const useResource = Effect.gen(function* () {
  const rcRef = yield* expensiveResourceRcRef
  
  // Get the resource - reference automatically managed
  const resource = yield* RcRef.get(rcRef)
  
  yield* Console.log(`Using resource ${resource.id}`)
  
  return resource.data
}).pipe(Effect.scoped)

// Run the program
Effect.runPromise(useResource)
/*
Output:
Creating expensive resource...
Using resource abc123def
Cleaning up resource abc123def
*/
```

### Shared Resource Access

```typescript
const sharedResourceExample = Effect.gen(function* () {
  const rcRef = yield* expensiveResourceRcRef
  
  // Multiple concurrent accesses share the same underlying resource
  const worker = (workerId: string) => Effect.gen(function* () {
    const resource = yield* RcRef.get(rcRef)
    
    yield* Console.log(`Worker ${workerId} using resource ${resource.id}`)
    yield* Effect.sleep(Duration.millis(100))
    
    return `Worker ${workerId} completed`
  }).pipe(Effect.scoped)
  
  // All workers share the same resource instance
  const results = yield* Effect.all([
    worker("A"),
    worker("B"),
    worker("C")
  ], { concurrency: "unbounded" })
  
  return results
}).pipe(Effect.scoped)

Effect.runPromise(sharedResourceExample)
/*
Output:
Creating expensive resource...
Worker A using resource abc123def
Worker B using resource abc123def
Worker C using resource abc123def
Cleaning up resource abc123def (only called once, after all workers complete)
*/
```

### RcRef with Idle Time-to-Live

```typescript
// Resource that automatically cleans up after 5 seconds of inactivity
const idleTimeoutRcRef = RcRef.make({
  acquire: Effect.gen(function* () {
    yield* Console.log("Creating resource with idle timeout...")
    
    return yield* Effect.acquireRelease(
      Effect.succeed({
        id: "timeout-resource",
        data: "will cleanup after idle time"
      }),
      (resource) => Console.log(`Idle timeout cleanup: ${resource.id}`)
    )
  }),
  
  // Resource disposed after 5 seconds of no active references
  idleTimeToLive: Duration.seconds(5)
})

const idleTimeoutExample = Effect.gen(function* () {
  const rcRef = yield* idleTimeoutRcRef
  
  // Use resource briefly
  const resource1 = yield* RcRef.get(rcRef).pipe(Effect.scoped)
  yield* Console.log(`First access: ${resource1.id}`)
  
  // Wait 3 seconds (less than idle timeout)
  yield* Effect.sleep(Duration.seconds(3))
  
  // Resource still alive due to idle timeout not reached
  const resource2 = yield* RcRef.get(rcRef).pipe(Effect.scoped)
  yield* Console.log(`Second access: ${resource2.id}`)
  
  // Wait 6 seconds - resource should be cleaned up due to idle timeout
  yield* Effect.sleep(Duration.seconds(6))
  
  // New resource created since previous one was cleaned up
  const resource3 = yield* RcRef.get(rcRef).pipe(Effect.scoped)
  yield* Console.log(`Third access: ${resource3.id}`)
}).pipe(Effect.scoped)
```

## Real-World Examples

### Example 1: Database Connection Pool Manager

A sophisticated database connection pool that shares expensive connections across multiple operations:

```typescript
import { RcRef, Effect, Console, Duration, Scope } from "effect"

interface DatabaseConnection {
  readonly id: string
  readonly host: string
  readonly database: string
  readonly createdAt: number
  query<T>(sql: string): Effect.Effect<T[]>
  close(): Effect.Effect<void>
}

interface PoolConfig {
  readonly host: string
  readonly database: string
  readonly maxConnections: number
  readonly idleTimeout: Duration.Duration
}

const makeDatabaseConnectionPool = (config: PoolConfig) => Effect.gen(function* () {
  // Create RcRef for each potential connection slot
  const connectionSlots = Array.from({ length: config.maxConnections }, (_, index) =>
    RcRef.make({
      acquire: Effect.gen(function* () {
        const connectionId = `${config.host}_${config.database}_${index}`
        
        yield* Console.log(`Creating database connection ${connectionId}`)
        
        return yield* Effect.acquireRelease(
          Effect.gen(function* (): Effect.Effect<DatabaseConnection> {
            // Simulate expensive connection creation
            yield* Effect.sleep(Duration.millis(200))
            
            const connection: DatabaseConnection = {
              id: connectionId,
              host: config.host,
              database: config.database,
              createdAt: Date.now(),
              
              query: <T>(sql: string) => Effect.gen(function* () {
                yield* Console.log(`Executing query on ${connectionId}: ${sql}`)
                yield* Effect.sleep(Duration.millis(50))
                return [] as T[]
              }),
              
              close: () => Effect.gen(function* () {
                yield* Console.log(`Closing connection ${connectionId}`)
                yield* Effect.sleep(Duration.millis(100))
              })
            }
            
            return connection
          }),
          (connection) => connection.close()
        )
      }),
      
      // Connections idle for 30 seconds are automatically closed
      idleTimeToLive: config.idleTimeout
    })
  )
  
  const getConnection = Effect.gen(function* () {
    // Simple round-robin connection selection
    const slotIndex = Math.floor(Math.random() * connectionSlots.length)
    const rcRef = yield* connectionSlots[slotIndex]
    
    return yield* RcRef.get(rcRef)
  })
  
  const executeQuery = <T>(sql: string) => Effect.gen(function* () {
    // Get connection from pool and execute query
    const connection = yield* getConnection
    const result = yield* connection.query<T>(sql)
    
    return {
      result,
      connectionId: connection.id,
      executedAt: Date.now()
    }
  }).pipe(Effect.scoped) // Connection automatically returned to pool
  
  const getPoolStats = Effect.gen(function* () {
    // This would require additional state tracking in a real implementation
    return {
      maxConnections: config.maxConnections,
      idleTimeout: Duration.toMillis(config.idleTimeout),
      host: config.host,
      database: config.database
    }
  })
  
  return { executeQuery, getPoolStats }
})

// Usage example
const databasePoolExample = Effect.gen(function* () {
  const pool = yield* makeDatabaseConnectionPool({
    host: "localhost",
    database: "myapp",
    maxConnections: 3,
    idleTimeout: Duration.seconds(30)
  })
  
  // Simulate multiple concurrent database operations
  const queries = [
    "SELECT * FROM users WHERE active = true",
    "SELECT COUNT(*) FROM orders WHERE status = 'pending'",
    "SELECT * FROM products WHERE category = 'electronics'",
    "UPDATE users SET last_seen = NOW() WHERE id = 123",
    "INSERT INTO audit_log (action, timestamp) VALUES ('login', NOW())"
  ]
  
  // Execute all queries concurrently - connections are shared efficiently
  const results = yield* Effect.all(
    queries.map(sql => pool.executeQuery(sql)),
    { concurrency: "unbounded" }
  )
  
  results.forEach((result, index) => {
    console.log(`Query ${index + 1} executed on ${result.connectionId}`)
  })
  
  const stats = yield* pool.getPoolStats
  yield* Console.log(`Pool stats: ${JSON.stringify(stats, null, 2)}`)
  
  return results
}).pipe(Effect.scoped)

Effect.runPromise(databasePoolExample)
```

### Example 2: HTTP Client with Connection Reuse

An HTTP client that efficiently reuses connections across multiple requests:

```typescript
import { RcRef, Effect, Console, Duration } from "effect"

interface HttpConnection {
  readonly id: string
  readonly baseUrl: string
  readonly keepAlive: boolean
  readonly createdAt: number
  request(path: string, options?: any): Effect.Effect<any>
  close(): Effect.Effect<void>
}

const makeHttpClientPool = (baseUrl: string) => Effect.gen(function* () {
  const connectionRcRef = RcRef.make({
    acquire: Effect.gen(function* () {
      const connectionId = `http_${baseUrl.replace(/[^\w]/g, '_')}_${Date.now()}`
      
      yield* Console.log(`Creating HTTP connection to ${baseUrl}`)
      
      return yield* Effect.acquireRelease(
        Effect.gen(function* (): Effect.Effect<HttpConnection> {
          // Simulate connection establishment
          yield* Effect.sleep(Duration.millis(100))
          
          const connection: HttpConnection = {
            id: connectionId,
            baseUrl,
            keepAlive: true,
            createdAt: Date.now(),
            
            request: (path: string, options = {}) => Effect.gen(function* () {
              yield* Console.log(`${connection.id}: ${options.method || 'GET'} ${path}`)
              
              // Simulate HTTP request
              yield* Effect.sleep(Duration.millis(50))
              
              return {
                status: 200,
                data: { path, timestamp: Date.now() },
                connectionId: connection.id
              }
            }),
            
            close: () => Effect.gen(function* () {
              yield* Console.log(`Closing HTTP connection ${connectionId}`)
              yield* Effect.sleep(Duration.millis(50))
            })
          }
          
          return connection
        }),
        (connection) => connection.close()
      )
    }),
    
    // Close idle connections after 60 seconds
    idleTimeToLive: Duration.seconds(60)
  })
  
  // HTTP client methods that reuse the connection
  const get = (path: string) => Effect.gen(function* () {
    const rcRef = yield* connectionRcRef
    const connection = yield* RcRef.get(rcRef)
    
    return yield* connection.request(path, { method: 'GET' })
  }).pipe(Effect.scoped)
  
  const post = (path: string, data: any) => Effect.gen(function* () {
    const rcRef = yield* connectionRcRef
    const connection = yield* RcRef.get(rcRef)
    
    return yield* connection.request(path, { method: 'POST', data })
  }).pipe(Effect.scoped)
  
  const put = (path: string, data: any) => Effect.gen(function* () {
    const rcRef = yield* connectionRcRef
    const connection = yield* RcRef.get(rcRef)
    
    return yield* connection.request(path, { method: 'PUT', data })
  }).pipe(Effect.scoped)
  
  return { get, post, put }
})

// Usage example with multiple concurrent requests
const httpClientExample = Effect.gen(function* () {
  const client = yield* makeHttpClientPool("https://api.example.com")
  
  // Multiple concurrent requests share the same underlying connection
  const requests = [
    client.get("/users"),
    client.get("/posts"),
    client.post("/comments", { text: "Great article!" }),
    client.put("/users/123", { name: "Updated Name" }),
    client.get("/notifications")
  ]
  
  const responses = yield* Effect.all(requests, { concurrency: "unbounded" })
  
  responses.forEach((response, index) => {
    console.log(`Request ${index + 1}: ${response.status} (${response.connectionId})`)
  })
  
  yield* Console.log("All requests completed with shared connection")
  
  return responses
}).pipe(Effect.scoped)

Effect.runPromise(httpClientExample)
/*
Output:
Creating HTTP connection to https://api.example.com
http_https___api_example_com_1234567890: GET /users
http_https___api_example_com_1234567890: GET /posts
http_https___api_example_com_1234567890: POST /comments
http_https___api_example_com_1234567890: PUT /users/123
http_https___api_example_com_1234567890: GET /notifications
Request 1: 200 (http_https___api_example_com_1234567890)
Request 2: 200 (http_https___api_example_com_1234567890)
Request 3: 200 (http_https___api_example_com_1234567890)
Request 4: 200 (http_https___api_example_com_1234567890)
Request 5: 200 (http_https___api_example_com_1234567890)
All requests completed with shared connection
Closing HTTP connection http_https___api_example_com_1234567890
*/
```

### Example 3: Expensive Computation Cache with Memory Management

A sophisticated caching system that manages expensive computations with automatic memory cleanup:

```typescript
import { RcRef, Effect, Console, Duration, Hash } from "effect"

interface ComputationResult<T> {
  readonly key: string
  readonly value: T
  readonly computedAt: number
  readonly cost: number // Computation cost metric
}

interface CacheEntry<T> {
  readonly result: ComputationResult<T>
  readonly accessCount: number
  readonly lastAccessed: number
}

const makeExpensiveComputationCache = <T>() => Effect.gen(function* () {
  // Map of computation key to RcRef
  const cacheMap = new Map<string, RcRef.RcRef<CacheEntry<T>>>()
  
  const createCacheEntry = (key: string, computation: Effect.Effect<T>) =>
    RcRef.make({
      acquire: Effect.gen(function* () {
        yield* Console.log(`Computing expensive result for key: ${key}`)
        
        return yield* Effect.acquireRelease(
          Effect.gen(function* () {
            const startTime = Date.now()
            
            // Execute the expensive computation
            const value = yield* computation
            
            const endTime = Date.now()
            const cost = endTime - startTime
            
            const result: ComputationResult<T> = {
              key,
              value,
              computedAt: endTime,
              cost
            }
            
            const entry: CacheEntry<T> = {
              result,
              accessCount: 0,
              lastAccessed: endTime
            }
            
            yield* Console.log(`Computation completed for ${key} (${cost}ms)`)
            return entry
          }),
          (entry) => Console.log(`Cleaning up cached result for ${entry.result.key}`)
        )
      }),
      
      // Clean up unused cache entries after 5 minutes
      idleTimeToLive: Duration.minutes(5)
    })
  
  const get = (key: string, computation: Effect.Effect<T>) => Effect.gen(function* () {
    // Get or create RcRef for this computation
    let rcRef = cacheMap.get(key)
    
    if (!rcRef) {
      rcRef = yield* createCacheEntry(key, computation)
      cacheMap.set(key, rcRef)
    }
    
    // Get the cached entry and update access statistics
    const entry = yield* RcRef.get(rcRef)
    
    // Update access statistics (in a real implementation, this would be atomic)
    const updatedEntry: CacheEntry<T> = {
      ...entry,
      accessCount: entry.accessCount + 1,
      lastAccessed: Date.now()
    }
    
    yield* Console.log(`Cache hit for ${key} (accessed ${updatedEntry.accessCount} times)`)
    
    return updatedEntry.result.value
  }).pipe(Effect.scoped)
  
  const getStats = Effect.gen(function* () {
    return {
      cacheSize: cacheMap.size,
      keys: Array.from(cacheMap.keys())
    }
  })
  
  const clearCache = Effect.gen(function* () {
    const keys = Array.from(cacheMap.keys())
    cacheMap.clear()
    
    yield* Console.log(`Cleared cache (${keys.length} entries removed)`)
    return keys
  })
  
  return { get, getStats, clearCache }
})

// Example expensive computations
const fibonacciComputation = (n: number): Effect.Effect<number> => Effect.gen(function* () {
  yield* Effect.sleep(Duration.millis(100 * n)) // Simulate expensive work
  
  if (n <= 1) return n
  
  // This would typically use the cache recursively, but simplified for example
  return n * n // Simplified computation
})

const primeCheckComputation = (n: number): Effect.Effect<boolean> => Effect.gen(function* () {
  yield* Effect.sleep(Duration.millis(50)) // Simulate expensive work
  
  if (n < 2) return false
  
  for (let i = 2; i <= Math.sqrt(n); i++) {
    if (n % i === 0) return false
  }
  
  return true
})

// Usage example
const computationCacheExample = Effect.gen(function* () {
  const cache = yield* makeExpensiveComputationCache<number | boolean>()
  
  // Compute some fibonacci numbers - results are cached
  yield* Console.log("=== Computing Fibonacci Numbers ===")
  const fib10a = yield* cache.get("fib_10", fibonacciComputation(10))
  const fib10b = yield* cache.get("fib_10", fibonacciComputation(10)) // Cache hit
  const fib15 = yield* cache.get("fib_15", fibonacciComputation(15))
  
  yield* Console.log("\n=== Computing Prime Checks ===")
  const prime97a = yield* cache.get("prime_97", primeCheckComputation(97))
  const prime97b = yield* cache.get("prime_97", primeCheckComputation(97)) // Cache hit
  const prime101 = yield* cache.get("prime_101", primeCheckComputation(101))
  
  const stats = yield* cache.getStats
  yield* Console.log(`\nCache stats: ${JSON.stringify(stats, null, 2)}`)
  
  // Wait for cache to potentially clean up idle entries
  yield* Console.log("\nWaiting 6 minutes for idle cleanup...")
  yield* Effect.sleep(Duration.minutes(6))
  
  // Try to access cached values after idle timeout
  yield* Console.log("Attempting to access cached values after idle timeout...")
  const fib10c = yield* cache.get("fib_10", fibonacciComputation(10))
  
  return { fib10a, fib10b, fib15, prime97a, prime97b, prime101, fib10c }
}).pipe(Effect.scoped)

Effect.runPromise(computationCacheExample)
```

## Advanced Features Deep Dive

### Understanding Reference Counting Mechanics

RcRef uses a sophisticated reference counting system that tracks active references across scopes:

```typescript
const rcRefMechanicsExample = Effect.gen(function* () {
  const rcRef = yield* RcRef.make({
    acquire: Effect.gen(function* () {
      yield* Console.log("Resource acquired (ref count: 1)")
      
      return yield* Effect.acquireRelease(
        Effect.succeed("shared-resource"),
        () => Console.log("Resource disposed (ref count: 0)")
      )
    })
  })
  
  // Demonstrate how references are counted across scopes
  const useResourceInNestedScopes = Effect.gen(function* () {
    // First scope - reference count becomes 1
    const resource1 = yield* RcRef.get(rcRef).pipe(
      Effect.tap(resource => Console.log(`Scope 1: Using ${resource}`)),
      Effect.scoped
    )
    
    yield* Console.log("Exited scope 1 - reference count decreases")
    
    // Concurrent scopes sharing the resource
    yield* Effect.all([
      RcRef.get(rcRef).pipe(
        Effect.tap(resource => Console.log(`Scope 2A: Using ${resource}`)),
        Effect.scoped
      ),
      RcRef.get(rcRef).pipe(
        Effect.tap(resource => Console.log(`Scope 2B: Using ${resource}`)),
        Effect.scoped
      )
    ], { concurrency: "unbounded" })
    
    yield* Console.log("Exited concurrent scopes")
    
    return "completed"
  })
  
  const result = yield* useResourceInNestedScopes
  yield* Console.log(`Final result: ${result}`)
  
  return result
}).pipe(Effect.scoped)
```

### Custom Resource Lifecycle Management

RcRef allows fine-grained control over resource lifecycle with custom acquisition and cleanup logic:

```typescript
interface ManagedFileHandle {
  readonly path: string
  readonly handle: string
  readonly openedAt: number
  read(): Effect.Effect<string>
  write(data: string): Effect.Effect<void>
  flush(): Effect.Effect<void>
}

const makeManagedFileSystem = Effect.gen(function* () {
  const openFiles = new Map<string, RcRef.RcRef<ManagedFileHandle>>()
  
  const openFile = (path: string) => Effect.gen(function* () {
    // Check if file is already managed
    let fileRcRef = openFiles.get(path)
    
    if (!fileRcRef) {
      // Create new managed file handle
      fileRcRef = yield* RcRef.make({
        acquire: Effect.gen(function* () {
          yield* Console.log(`Opening file: ${path}`)
          
          return yield* Effect.acquireRelease(
            Effect.gen(function* (): Effect.Effect<ManagedFileHandle> {
              // Simulate file opening
              yield* Effect.sleep(Duration.millis(100))
              
              const handle = Math.random().toString(36).substr(2, 9)
              
              const fileHandle: ManagedFileHandle = {
                path,
                handle,
                openedAt: Date.now(),
                
                read: () => Effect.gen(function* () {
                  yield* Console.log(`Reading from ${path} (${handle})`)
                  yield* Effect.sleep(Duration.millis(50))
                  return `Content of ${path}`
                }),
                
                write: (data: string) => Effect.gen(function* () {
                  yield* Console.log(`Writing to ${path} (${handle}): ${data}`)
                  yield* Effect.sleep(Duration.millis(75))
                }),
                
                flush: () => Effect.gen(function* () {
                  yield* Console.log(`Flushing ${path} (${handle})`)
                  yield* Effect.sleep(Duration.millis(25))
                })
              }
              
              return fileHandle
            }),
            (fileHandle) => Effect.gen(function* () {
              // Custom cleanup: flush before closing
              yield* fileHandle.flush()
              yield* Console.log(`Closing file: ${fileHandle.path} (${fileHandle.handle})`)
              
              // Remove from tracking map
              openFiles.delete(path)
            })
          )
        }),
        
        // Files are closed after 10 seconds of inactivity
        idleTimeToLive: Duration.seconds(10)
      })
      
      openFiles.set(path, fileRcRef)
    }
    
    return yield* RcRef.get(fileRcRef)
  }).pipe(Effect.scoped)
  
  const getOpenFileStats = Effect.gen(function* () {
    return {
      openFiles: openFiles.size,
      filePaths: Array.from(openFiles.keys())
    }
  })
  
  return { openFile, getOpenFileStats }
})

// Complex file operations with shared handles
const fileSystemExample = Effect.gen(function* () {
  const fs = yield* makeManagedFileSystem
  
  // Multiple operations on the same file share the handle
  yield* Effect.all([
    Effect.gen(function* () {
      const file = yield* fs.openFile("/tmp/shared.txt")
      yield* file.write("Data from operation 1")
      const content = yield* file.read()
      return content
    }),
    
    Effect.gen(function* () {
      const file = yield* fs.openFile("/tmp/shared.txt")
      yield* file.write("Data from operation 2")
      const content = yield* file.read()
      return content
    }),
    
    Effect.gen(function* () {
      const file = yield* fs.openFile("/tmp/different.txt")
      yield* file.write("Different file data")
      return "different-file-result"
    })
  ], { concurrency: "unbounded" })
  
  const stats = yield* fs.getOpenFileStats
  yield* Console.log(`File system stats: ${JSON.stringify(stats, null, 2)}`)
  
  // Wait for idle timeout to trigger cleanup
  yield* Console.log("Waiting for idle cleanup...")
  yield* Effect.sleep(Duration.seconds(11))
  
  const finalStats = yield* fs.getOpenFileStats
  yield* Console.log(`Final stats: ${JSON.stringify(finalStats, null, 2)}`)
}).pipe(Effect.scoped)
```

### Composing RcRefs for Complex Resource Dependencies

RcRefs can be composed to manage complex resource dependency graphs:

```typescript
interface DatabaseConnection {
  readonly id: string
  query(sql: string): Effect.Effect<any[]>
  close(): Effect.Effect<void>
}

interface CacheConnection {
  readonly id: string
  get(key: string): Effect.Effect<any>
  set(key: string, value: any): Effect.Effect<void>
  close(): Effect.Effect<void>
}

interface ApplicationServices {
  readonly database: DatabaseConnection
  readonly cache: CacheConnection
  readonly startedAt: number
}

const makeApplicationServices = Effect.gen(function* () {
  // Database connection RcRef
  const databaseRcRef = RcRef.make({
    acquire: Effect.gen(function* () {
      yield* Console.log("Establishing database connection...")
      
      return yield* Effect.acquireRelease(
        Effect.gen(function* (): Effect.Effect<DatabaseConnection> {
          yield* Effect.sleep(Duration.millis(200))
          
          const dbId = `db_${Date.now()}`
          
          return {
            id: dbId,
            query: (sql: string) => Effect.gen(function* () {
              yield* Console.log(`DB ${dbId}: ${sql}`)
              yield* Effect.sleep(Duration.millis(50))
              return []
            }),
            close: () => Console.log(`Closing database connection ${dbId}`)
          }
        }),
        (db) => db.close()
      )
    }),
    idleTimeToLive: Duration.minutes(5)
  })
  
  // Cache connection RcRef
  const cacheRcRef = RcRef.make({
    acquire: Effect.gen(function* () {
      yield* Console.log("Establishing cache connection...")
      
      return yield* Effect.acquireRelease(
        Effect.gen(function* (): Effect.Effect<CacheConnection> {
          yield* Effect.sleep(Duration.millis(100))
          
          const cacheId = `cache_${Date.now()}`
          
          return {
            id: cacheId,
            get: (key: string) => Effect.gen(function* () {
              yield* Console.log(`Cache ${cacheId}: GET ${key}`)
              return null
            }),
            set: (key: string, value: any) => Effect.gen(function* () {
              yield* Console.log(`Cache ${cacheId}: SET ${key} = ${JSON.stringify(value)}`)
            }),
            close: () => Console.log(`Closing cache connection ${cacheId}`)
          }
        }),
        (cache) => cache.close()
      )
    }),
    idleTimeToLive: Duration.minutes(3)
  })
  
  // Composite application services RcRef
  const servicesRcRef = RcRef.make({
    acquire: Effect.gen(function* () {
      yield* Console.log("Initializing application services...")
      
      return yield* Effect.acquireRelease(
        Effect.gen(function* (): Effect.Effect<ApplicationServices> {
          // Get both database and cache connections
          const [dbRcRef, cacheRcRef_] = yield* Effect.all([databaseRcRef, cacheRcRef])
          const [database, cache] = yield* Effect.all([
            RcRef.get(dbRcRef),
            RcRef.get(cacheRcRef_)
          ])
          
          const services: ApplicationServices = {
            database,
            cache,
            startedAt: Date.now()
          }
          
          yield* Console.log("Application services initialized successfully")
          return services
        }),
        (services) => Console.log(`Shutting down application services (uptime: ${Date.now() - services.startedAt}ms)`)
      )
    }),
    idleTimeToLive: Duration.minutes(10)
  })
  
  return servicesRcRef
})

// Business logic using composed services
const makeUserService = Effect.gen(function* () {
  const servicesRcRef = yield* makeApplicationServices
  
  const getUser = (userId: string) => Effect.gen(function* () {
    const services = yield* RcRef.get(servicesRcRef)
    
    // Check cache first
    const cachedUser = yield* services.cache.get(`user:${userId}`)
    if (cachedUser) {
      return cachedUser
    }
    
    // Fetch from database
    const users = yield* services.database.query(`SELECT * FROM users WHERE id = '${userId}'`)
    const user = users[0] || null
    
    // Cache the result
    if (user) {
      yield* services.cache.set(`user:${userId}`, user)
    }
    
    return user
  }).pipe(Effect.scoped)
  
  const createUser = (userData: any) => Effect.gen(function* () {
    const services = yield* RcRef.get(servicesRcRef)
    
    // Create in database
    yield* services.database.query(`INSERT INTO users (...) VALUES (...)`)
    
    // Invalidate related caches
    yield* services.cache.set(`user:${userData.id}`, userData)
    
    return userData
  }).pipe(Effect.scoped)
  
  return { getUser, createUser }
})

// Application usage
const composedServicesExample = Effect.gen(function* () {
  const userService = yield* makeUserService
  
  // Multiple concurrent operations share all underlying resources
  const operations = yield* Effect.all([
    userService.getUser("user1"),
    userService.getUser("user2"),
    userService.createUser({ id: "user3", name: "Alice" }),
    userService.getUser("user3") // Should hit cache
  ], { concurrency: "unbounded" })
  
  yield* Console.log("All operations completed successfully")
  return operations
}).pipe(Effect.scoped)
```

## Practical Patterns & Best Practices

### Pattern 1: Resource Pool Pattern

Create pools of expensive resources that are shared across operations:

```typescript
const makeResourcePool = <T>(
  createResource: Effect.Effect<T>,
  poolSize: number,
  idleTimeout: Duration.Duration
) => Effect.gen(function* () {
  // Create multiple RcRefs for the pool
  const pool = Array.from({ length: poolSize }, () =>
    RcRef.make({
      acquire: createResource.pipe(
        Effect.tap(() => Console.log("Creating pooled resource")),
        Effect.map(resource => Effect.acquireRelease(
          Effect.succeed(resource),
          () => Console.log("Disposing pooled resource")
        )),
        Effect.flatten
      ),
      idleTimeToLive: idleTimeout
    })
  )
  
  let currentIndex = 0
  
  const acquire = Effect.gen(function* () {
    // Simple round-robin selection
    const rcRef = yield* pool[currentIndex % poolSize]
    currentIndex++
    
    return yield* RcRef.get(rcRef)
  }).pipe(Effect.scoped)
  
  return { acquire }
})
```

### Pattern 2: Hierarchical Resource Dependencies

Model complex resource hierarchies where child resources depend on parent resources:

```typescript
const makeHierarchicalResources = Effect.gen(function* () {
  // Parent resource
  const parentRcRef = RcRef.make({
    acquire: Effect.acquireRelease(
      Effect.gen(function* () {
        yield* Console.log("Creating parent resource")
        return { id: "parent", children: new Map() }
      }),
      () => Console.log("Disposing parent resource")
    )
  })
  
  // Child resource factory
  const createChild = (childId: string) => RcRef.make({
    acquire: Effect.gen(function* () {
      // Child requires parent to exist
      const parentRef = yield* parentRcRef
      const parent = yield* RcRef.get(parentRef)
      
      return yield* Effect.acquireRelease(
        Effect.gen(function* () {
          yield* Console.log(`Creating child ${childId}`)
          const child = { id: childId, parentId: parent.id }
          parent.children.set(childId, child)
          return child
        }),
        (child) => Effect.gen(function* () {
          yield* Console.log(`Disposing child ${child.id}`)
          const parent = yield* RcRef.get(parentRef)
          parent.children.delete(childId)
        })
      )
    })
  })
  
  return { parentRcRef, createChild }
})
```

### Pattern 3: Resource State Validation

Ensure resources are in valid states before use:

```typescript
interface ValidatedResource {
  readonly id: string
  readonly isValid: boolean
  validate(): Effect.Effect<boolean>
  use(): Effect.Effect<string>
}

const makeValidatedResourceRcRef = RcRef.make({
  acquire: Effect.gen(function* () {
    return yield* Effect.acquireRelease(
      Effect.gen(function* (): Effect.Effect<ValidatedResource> {
        const resource: ValidatedResource = {
          id: `resource_${Date.now()}`,
          isValid: true,
          
          validate: () => Effect.gen(function* () {
            // Simulate validation logic
            yield* Effect.sleep(Duration.millis(10))
            return Math.random() > 0.1 // 90% success rate
          }),
          
          use: () => Effect.gen(function* () {
            yield* Effect.sleep(Duration.millis(50))
            return `Result from ${resource.id}`
          })
        }
        
        return resource
      }),
      (resource) => Console.log(`Cleaning up ${resource.id}`)
    )
  })
})

const useValidatedResource = Effect.gen(function* () {
  const rcRef = yield* makeValidatedResourceRcRef
  const resource = yield* RcRef.get(rcRef)
  
  // Validate before use
  const isValid = yield* resource.validate()
  
  if (!isValid) {
    return yield* Effect.fail(new Error("Resource validation failed"))
  }
  
  return yield* resource.use()
}).pipe(Effect.scoped)
```

## Integration Examples

### Integration with Layer System

RcRef integrates seamlessly with Effect's Layer system for dependency injection:

```typescript
import { Layer, Context } from "effect"

// Service interface
interface DatabaseService {
  readonly query: (sql: string) => Effect.Effect<any[]>
}

const DatabaseService = Context.GenericTag<DatabaseService>("DatabaseService")

// Layer that provides DatabaseService using RcRef
const DatabaseServiceLive = Layer.effect(
  DatabaseService,
  Effect.gen(function* () {
    const connectionRcRef = yield* RcRef.make({
      acquire: Effect.acquireRelease(
        Effect.gen(function* () {
          yield* Console.log("Creating database connection for service")
          return {
            id: `db_service_${Date.now()}`,
            query: (sql: string) => Effect.gen(function* () {
              yield* Console.log(`Service query: ${sql}`)
              return []
            })
          }
        }),
        (conn) => Console.log(`Closing service connection ${conn.id}`)
      )
    })
    
    // Return service implementation
    return DatabaseService.of({
      query: (sql: string) => Effect.gen(function* () {
        const connection = yield* RcRef.get(connectionRcRef)
        return yield* connection.query(sql)
      }).pipe(Effect.scoped)
    })
  })
)

// Usage with dependency injection
const businessLogic = Effect.gen(function* () {
  const db = yield* DatabaseService
  
  const users = yield* db.query("SELECT * FROM users")
  const orders = yield* db.query("SELECT * FROM orders")
  
  return { users, orders }
})

const layerExample = businessLogic.pipe(
  Effect.provide(DatabaseServiceLive),
  Effect.scoped
)
```

### Testing Strategies with RcRef

Effective testing patterns for RcRef-based systems:

```typescript
// Mock resource for testing
const makeMockResource = (behavior: "success" | "failure" | "slow") => Effect.gen(function* () {
  return yield* Effect.acquireRelease(
    Effect.gen(function* () {
      switch (behavior) {
        case "success":
          return { id: "mock-success", data: "test-data" }
        case "failure":
          return yield* Effect.fail(new Error("Mock failure"))
        case "slow":
          yield* Effect.sleep(Duration.seconds(5))
          return { id: "mock-slow", data: "slow-data" }
      }
    }),
    (resource) => Console.log(`Cleaning up mock resource ${resource.id}`)
  )
})

// Test utility for RcRef behavior
const testRcRefBehavior = Effect.gen(function* () {
  const mockRcRef = yield* RcRef.make({
    acquire: makeMockResource("success")
  })
  
  // Test 1: Resource sharing
  const [resource1, resource2] = yield* Effect.all([
    RcRef.get(mockRcRef).pipe(Effect.scoped),
    RcRef.get(mockRcRef).pipe(Effect.scoped)
  ])
  
  console.assert(resource1.id === resource2.id, "Resources should be shared")
  
  // Test 2: Cleanup after scope
  let cleanupCalled = false
  const testCleanupRcRef = yield* RcRef.make({
    acquire: Effect.acquireRelease(
      Effect.succeed({ id: "cleanup-test" }),
      () => Effect.sync(() => { cleanupCalled = true })
    )
  })
  
  yield* RcRef.get(testCleanupRcRef).pipe(Effect.scoped)
  yield* Effect.sleep(Duration.millis(100))
  
  console.assert(cleanupCalled, "Cleanup should be called after scope ends")
  
  return "tests passed"
}).pipe(Effect.scoped)
```

## Conclusion

RcRef provides **reference-counted resource management**, **shared ownership with automatic disposal**, and **scope-integrated lifecycle management** for Effect applications.

Key benefits:
- **Automatic Resource Sharing** - Multiple concurrent operations can safely share expensive resources without duplication
- **Memory Leak Prevention** - Resources are automatically disposed when no longer referenced, preventing resource leaks
- **Exception Safety** - Resource cleanup is guaranteed even when exceptions occur, thanks to Effect's scope system
- **Configurable Idle Timeout** - Resources can be automatically cleaned up after periods of inactivity to free system resources

RcRef is essential when you need to share expensive resources (database connections, file handles, HTTP clients, computational caches) across multiple concurrent operations while ensuring proper cleanup and preventing resource leaks in long-running applications.