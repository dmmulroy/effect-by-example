# TReentrantLock: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem TReentrantLock Solves

In concurrent applications, managing access to shared resources is critical. Traditional approaches often lead to issues:

```typescript
// Traditional approach - race conditions and data corruption
let sharedCounter = 0
let fileHandle: FileHandle | null = null

// Multiple concurrent operations
async function unsafeIncrement() {
  const current = sharedCounter  // Thread 1 reads 0
  // Thread 2 also reads 0 here
  sharedCounter = current + 1    // Both write 1, losing one increment
}

// Resource access without proper coordination
async function unsafeFileOperation() {
  if (!fileHandle) {
    fileHandle = await openFile("data.txt")
  }
  // Another thread might also initialize fileHandle
  return await fileHandle.read()
}
```

This approach leads to:
- **Race Conditions** - Multiple threads accessing shared state simultaneously
- **Data Corruption** - Lost updates and inconsistent state
- **Resource Leaks** - Multiple initializations of the same resource

### The TReentrantLock Solution

TReentrantLock provides a reentrant read/write lock for STM transactions and effectful code:

```typescript
import { TReentrantLock, STM, Effect } from "effect"

// Safe, coordinated access to shared resources
const safeResourceAccess = Effect.gen(function* () {
  const lock = yield* STM.commit(TReentrantLock.make)
  
  return yield* TReentrantLock.withWriteLock(
    lock,
    Effect.gen(function* () {
      // Critical section - only one writer at a time
      const resource = yield* initializeResource()
      return yield* processResource(resource)
    })
  )
})
```

### Key Concepts

**Reentrant**: A fiber that holds a lock can acquire additional locks of the same type

**Read/Write Semantics**: Multiple readers can hold read locks simultaneously, but write locks are exclusive

**STM Integration**: Built on STM for composable, transactional lock operations

## Basic Usage Patterns

### Pattern 1: Simple Exclusive Access

```typescript
import { TReentrantLock, STM, Effect } from "effect"

// Create a lock and use it for exclusive access
const exclusiveOperation = Effect.gen(function* () {
  const lock = yield* STM.commit(TReentrantLock.make)
  
  return yield* TReentrantLock.withLock(
    lock,
    Effect.gen(function* () {
      // Only one fiber can execute this section at a time
      yield* Effect.logInfo("Performing exclusive operation")
      yield* Effect.sleep("100 milli")
      return "Operation complete"
    })
  )
})
```

### Pattern 2: Read/Write Lock Usage

```typescript
// Multiple readers, single writer pattern
const createReadWriteExample = Effect.gen(function* () {
  const lock = yield* STM.commit(TReentrantLock.make)
  
  // Read operation - multiple concurrent readers allowed
  const readOperation = TReentrantLock.withReadLock(
    lock,
    Effect.gen(function* () {
      yield* Effect.logInfo("Reading data")
      yield* Effect.sleep("50 milli")
      return "data-value"
    })
  )
  
  // Write operation - exclusive access required
  const writeOperation = TReentrantLock.withWriteLock(
    lock,
    Effect.gen(function* () {
      yield* Effect.logInfo("Writing data")
      yield* Effect.sleep("100 milli")
      return "write-complete"
    })
  )
  
  return { readOperation, writeOperation }
})
```

### Pattern 3: Scoped Lock Management

```typescript
// Using scoped locks for automatic cleanup
const scopedLockExample = Effect.scoped(
  Effect.gen(function* () {
    const lock = yield* STM.commit(TReentrantLock.make)
    
    // Acquire lock in scoped context
    const writeCount = yield* TReentrantLock.writeLock(lock)
    
    yield* Effect.logInfo(`Acquired write lock (count: ${writeCount})`)
    yield* Effect.sleep("200 milli")
    
    // Lock automatically released when scope exits
    return "Scoped operation complete"
  })
)
```

## Real-World Examples

### Example 1: Database Connection Pool

```typescript
import { TReentrantLock, STM, Effect, TRef } from "effect"

interface DatabaseConnection {
  readonly id: string
  readonly query: (sql: string) => Effect.Effect<Array<Record<string, unknown>>>
  readonly close: () => Effect.Effect<void>
}

interface ConnectionPool {
  readonly lock: TReentrantLock
  readonly connections: TRef<Array<DatabaseConnection>>
  readonly maxConnections: number
}

const createConnectionPool = (maxConnections: number) =>
  Effect.gen(function* () {
    const lock = yield* STM.commit(TReentrantLock.make)
    const connections = yield* STM.commit(TRef.make<Array<DatabaseConnection>>([]))
    
    return { lock, connections, maxConnections } as ConnectionPool
  })

// Safe connection acquisition with read lock for checking availability
const acquireConnection = (pool: ConnectionPool) =>
  Effect.gen(function* () {
    // First check if connections are available (read lock)
    const available = yield* TReentrantLock.withReadLock(
      pool.lock,
      Effect.gen(function* () {
        const conns = yield* STM.commit(TRef.get(pool.connections))
        return conns.length > 0
      })
    )
    
    if (available) {
      // Get connection with write lock
      return yield* TReentrantLock.withWriteLock(
        pool.lock,
        Effect.gen(function* () {
          const conns = yield* STM.commit(TRef.get(pool.connections))
          if (conns.length === 0) {
            return yield* Effect.fail(new Error("No connections available"))
          }
          
          const [connection, ...remaining] = conns
          yield* STM.commit(TRef.set(pool.connections, remaining))
          return connection
        })
      )
    } else {
      return yield* Effect.fail(new Error("Pool exhausted"))
    }
  })

// Safe connection release
const releaseConnection = (pool: ConnectionPool, connection: DatabaseConnection) =>
  TReentrantLock.withWriteLock(
    pool.lock,
    Effect.gen(function* () {
      const conns = yield* STM.commit(TRef.get(pool.connections))
      
      if (conns.length < pool.maxConnections) {
        yield* STM.commit(TRef.update(pool.connections, (existing) => [...existing, connection]))
      } else {
        // Pool is full, close the connection
        yield* connection.close()
      }
    })
  )
```

### Example 2: Shared Cache Management

```typescript
import { TReentrantLock, STM, Effect, TRef } from "effect"

interface CacheEntry<T> {
  readonly value: T
  readonly expiry: Date
  readonly accessCount: number
}

interface SharedCache<K, V> {
  readonly lock: TReentrantLock
  readonly data: TRef<Map<K, CacheEntry<V>>>
  readonly maxSize: number
}

const createSharedCache = <K, V>(maxSize: number) =>
  Effect.gen(function* () {
    const lock = yield* STM.commit(TReentrantLock.make)
    const data = yield* STM.commit(TRef.make(new Map<K, CacheEntry<V>>()))
    
    return { lock, data, maxSize } as SharedCache<K, V>
  })

// Read operations use read lock for concurrent access
const get = <K, V>(cache: SharedCache<K, V>, key: K) =>
  TReentrantLock.withReadLock(
    cache.lock,
    Effect.gen(function* () {
      const map = yield* STM.commit(TRef.get(cache.data))
      const entry = map.get(key)
      
      if (!entry) {
        return yield* Effect.fail(new Error(`Key not found: ${key}`))
      }
      
      if (entry.expiry < new Date()) {
        return yield* Effect.fail(new Error(`Key expired: ${key}`))
      }
      
      // Update access count (requires write lock)
      return yield* TReentrantLock.withWriteLock(
        cache.lock,
        Effect.gen(function* () {
          yield* STM.commit(TRef.update(cache.data, (currentMap) => {
            const updatedEntry = {
              ...entry,
              accessCount: entry.accessCount + 1
            }
            return new Map(currentMap).set(key, updatedEntry)
          }))
          return entry.value
        })
      )
    })
  )

// Write operations use write lock for exclusive access
const put = <K, V>(
  cache: SharedCache<K, V>,
  key: K,
  value: V,
  ttlMs: number
) =>
  TReentrantLock.withWriteLock(
    cache.lock,
    Effect.gen(function* () {
      const map = yield* STM.commit(TRef.get(cache.data))
      const expiry = new Date(Date.now() + ttlMs)
      const entry: CacheEntry<V> = { value, expiry, accessCount: 0 }
      
      // Evict if at capacity
      if (map.size >= cache.maxSize && !map.has(key)) {
        yield* evictLeastRecentlyUsed(cache)
      }
      
      yield* STM.commit(TRef.update(cache.data, (currentMap) =>
        new Map(currentMap).set(key, entry)
      ))
    })
  )

const evictLeastRecentlyUsed = <K, V>(cache: SharedCache<K, V>) =>
  Effect.gen(function* () {
    const map = yield* STM.commit(TRef.get(cache.data))
    let lruKey: K | undefined
    let minAccessCount = Infinity
    
    for (const [key, entry] of map.entries()) {
      if (entry.accessCount < minAccessCount) {
        minAccessCount = entry.accessCount
        lruKey = key
      }
    }
    
    if (lruKey !== undefined) {
      yield* STM.commit(TRef.update(cache.data, (currentMap) => {
        const newMap = new Map(currentMap)
        newMap.delete(lruKey!)
        return newMap
      }))
    }
  })
```

### Example 3: File System Synchronization

```typescript
import { TReentrantLock, STM, Effect, TRef } from "effect"

interface FileSystemManager {
  readonly lock: TReentrantLock
  readonly openFiles: TRef<Set<string>>
}

const createFileSystemManager = () =>
  Effect.gen(function* () {
    const lock = yield* STM.commit(TReentrantLock.make)
    const openFiles = yield* STM.commit(TRef.make(new Set<string>()))
    
    return { lock, openFiles } as FileSystemManager
  })

// Safe file operations with proper locking
const safeFileOperation = (
  manager: FileSystemManager,
  filePath: string,
  operation: (content: string) => Effect.Effect<string>
) =>
  Effect.gen(function* () {
    // Check if file is already open (read lock)
    const isOpen = yield* TReentrantLock.withReadLock(
      manager.lock,
      Effect.gen(function* () {
        const openFiles = yield* STM.commit(TRef.get(manager.openFiles))
        return openFiles.has(filePath)
      })
    )
    
    if (isOpen) {
      return yield* Effect.fail(new Error(`File already open: ${filePath}`))
    }
    
    // Perform file operation with write lock
    return yield* TReentrantLock.withWriteLock(
      manager.lock,
      Effect.gen(function* () {
        // Mark file as open
        yield* STM.commit(TRef.update(manager.openFiles, (files) => new Set(files).add(filePath)))
        
        try {
          // Read file content
          const content = yield* Effect.tryPromise(() => 
            require('fs').promises.readFile(filePath, 'utf8')
          )
          
          // Process content
          const processed = yield* operation(content)
          
          // Write back processed content
          yield* Effect.tryPromise(() =>
            require('fs').promises.writeFile(filePath, processed, 'utf8')
          )
          
          return processed
        } finally {
          // Always remove from open files
          yield* STM.commit(TRef.update(manager.openFiles, (files) => {
            const newFiles = new Set(files)
            newFiles.delete(filePath)
            return newFiles
          }))
        }
      })
    )
  })
```

## Advanced Features Deep Dive

### Feature 1: Lock State Inspection

Lock state can be inspected using STM operations for debugging and monitoring.

```typescript
const inspectLockState = (lock: TReentrantLock) =>
  STM.gen(function* () {
    const isLocked = yield* TReentrantLock.locked(lock)
    const isReadLocked = yield* TReentrantLock.readLocked(lock)
    const isWriteLocked = yield* TReentrantLock.writeLocked(lock)
    const totalReadLocks = yield* TReentrantLock.readLocks(lock)
    const totalWriteLocks = yield* TReentrantLock.writeLocks(lock)
    const fiberReadLocks = yield* TReentrantLock.fiberReadLocks(lock)
    const fiberWriteLocks = yield* TReentrantLock.fiberWriteLocks(lock)
    
    return {
      isLocked,
      isReadLocked,
      isWriteLocked,
      totalReadLocks,
      totalWriteLocks,
      fiberReadLocks,
      fiberWriteLocks
    }
  })

// Usage in monitoring
const monitorLockUsage = (lock: TReentrantLock) =>
  Effect.gen(function* () {
    const state = yield* STM.commit(inspectLockState(lock))
    
    yield* Effect.logInfo(`Lock State: ${JSON.stringify(state, null, 2)}`)
    
    if (state.totalWriteLocks > 5) {
      yield* Effect.logWarning("High write lock contention detected")
    }
  })
```

### Feature 2: Reentrant Lock Patterns

```typescript
// Reentrant lock allows the same fiber to acquire multiple locks
const reentrantExample = (lock: TReentrantLock) =>
  TReentrantLock.withWriteLock(
    lock,
    Effect.gen(function* () {
      yield* Effect.logInfo("First write lock acquired")
      
      // Same fiber can acquire additional write locks
      return yield* TReentrantLock.withWriteLock(
        lock,
        Effect.gen(function* () {
          yield* Effect.logInfo("Second write lock acquired (reentrant)")
          
          // Can also acquire read locks while holding write lock
          return yield* TReentrantLock.withReadLock(
            lock,
            Effect.gen(function* () {
              yield* Effect.logInfo("Read lock acquired while holding write lock")
              return "nested operation complete"
            })
          )
        })
      )
    })
  )
```

### Feature 3: Lock Upgrade Patterns

```typescript
// Reading first, then upgrading to write if needed
const upgradeableOperation = (
  lock: TReentrantLock,
  data: TRef<Array<string>>,
  item: string
) =>
  Effect.gen(function* () {
    // First, check if item exists with read lock
    const exists = yield* TReentrantLock.withReadLock(
      lock,
      Effect.gen(function* () {
        const current = yield* STM.commit(TRef.get(data))
        return current.includes(item)
      })
    )
    
    if (!exists) {
      // Upgrade to write lock to add item
      return yield* TReentrantLock.withWriteLock(
        lock,
        Effect.gen(function* () {
          // Double-check after acquiring write lock
          const current = yield* STM.commit(TRef.get(data))
          if (!current.includes(item)) {
            yield* STM.commit(TRef.update(data, (arr) => [...arr, item]))
            return `Added: ${item}`
          } else {
            return `Already exists: ${item}`
          }
        })
      )
    } else {
      return `Item already exists: ${item}`
    }
  })
```

## Practical Patterns & Best Practices

### Pattern 1: Lock Manager Service

```typescript
import { Context, Layer, TReentrantLock, STM, Effect, TRef } from "effect"

interface LockManager {
  readonly locks: TRef<Map<string, TReentrantLock>>
}

const LockManager = Context.GenericTag<LockManager>("LockManager")

const makeLockManager = Effect.gen(function* () {
  const locks = yield* STM.commit(TRef.make(new Map<string, TReentrantLock>()))
  return { locks } as LockManager
})

const lockManagerLayer = Layer.effect(LockManager, makeLockManager)

// Get or create named lock
const getNamedLock = (name: string) =>
  Effect.gen(function* () {
    const manager = yield* LockManager
    const locks = yield* STM.commit(TRef.get(manager.locks))
    
    if (locks.has(name)) {
      return locks.get(name)!
    }
    
    // Create new lock
    const newLock = yield* STM.commit(TReentrantLock.make)
    yield* STM.commit(TRef.update(manager.locks, (current) => 
      new Map(current).set(name, newLock)
    ))
    
    return newLock
  })

// Helper for safe named operations
const withNamedLock = <A, E, R>(
  name: string,
  operation: Effect.Effect<A, E, R>
) =>
  Effect.gen(function* () {
    const lock = yield* getNamedLock(name)
    return yield* TReentrantLock.withLock(lock, operation)
  })
```

### Pattern 2: Read-Preferring vs Write-Preferring

```typescript
// Read-preferring lock (default TReentrantLock behavior)
const readPreferringOperation = <A>(
  lock: TReentrantLock,
  readOp: Effect.Effect<A>,
  writeOp: Effect.Effect<A>
) =>
  Effect.gen(function* () {
    // Try read operation first
    const canRead = yield* STM.commit(
      STM.gen(function* () {
        const isWriteLocked = yield* TReentrantLock.writeLocked(lock)
        return !isWriteLocked
      })
    )
    
    if (canRead) {
      return yield* TReentrantLock.withReadLock(lock, readOp)
    } else {
      return yield* TReentrantLock.withWriteLock(lock, writeOp)
    }
  })

// Write-preferring pattern using custom coordination
const writePreferringOperation = <A>(
  lock: TReentrantLock,
  pendingWrites: TRef<number>,
  readOp: Effect.Effect<A>,
  writeOp: Effect.Effect<A>
) =>
  Effect.gen(function* () {
    const pendingWriteCount = yield* STM.commit(TRef.get(pendingWrites))
    
    if (pendingWriteCount > 0) {
      // Defer to pending writes
      yield* Effect.sleep("10 milli")
      return yield* writePreferringOperation(lock, pendingWrites, readOp, writeOp)
    }
    
    return yield* TReentrantLock.withReadLock(lock, readOp)
  })
```

### Pattern 3: Timeout and Interruption Handling

```typescript
// Lock operations with timeout
const withTimeout = <A, E, R>(
  lock: TReentrantLock,
  operation: Effect.Effect<A, E, R>,
  timeoutMs: number
) =>
  TReentrantLock.withLock(lock, operation).pipe(
    Effect.timeout(`${timeoutMs} milli`),
    Effect.catchTag("TimeoutException", () =>
      Effect.fail(new Error(`Operation timed out after ${timeoutMs}ms`))
    )
  )

// Interruptible lock operations
const interruptibleLockOperation = <A, E, R>(
  lock: TReentrantLock,
  operation: Effect.Effect<A, E, R>
) =>
  Effect.gen(function* () {
    // Check for interruption before acquiring lock
    yield* Effect.yieldNow()
    
    return yield* TReentrantLock.withLock(
      lock,
      Effect.gen(function* () {
        // Periodic interruption checks during long operations
        yield* Effect.yieldNow()
        const result = yield* operation
        yield* Effect.yieldNow()
        return result
      })
    )
  })
```

## Integration Examples

### Integration with Effect Layers

```typescript
// Service that requires coordinated access
interface DatabaseService {
  readonly executeQuery: (sql: string) => Effect.Effect<Array<Record<string, unknown>>>
  readonly executeTransaction: <A>(
    operations: Effect.Effect<A>
  ) => Effect.Effect<A>
}

const DatabaseService = Context.GenericTag<DatabaseService>("DatabaseService")

const makeDatabaseService = Effect.gen(function* () {
  const transactionLock = yield* STM.commit(TReentrantLock.make)
  const queryLock = yield* STM.commit(TReentrantLock.make)
  
  const executeQuery = (sql: string) =>
    TReentrantLock.withReadLock(
      queryLock,
      Effect.gen(function* () {
        yield* Effect.logInfo(`Executing query: ${sql}`)
        // Simulate query execution
        yield* Effect.sleep("50 milli")
        return [{ result: "query-result" }]
      })
    )
  
  const executeTransaction = <A>(operations: Effect.Effect<A>) =>
    TReentrantLock.withWriteLock(
      transactionLock,
      Effect.gen(function* () {
        yield* Effect.logInfo("Starting transaction")
        const result = yield* operations
        yield* Effect.logInfo("Transaction complete")
        return result
      })
    )
  
  return { executeQuery, executeTransaction } as DatabaseService
})

const databaseServiceLayer = Layer.effect(DatabaseService, makeDatabaseService)

// Usage with the service
const databaseOperationExample = Effect.gen(function* () {
  const db = yield* DatabaseService
  
  // Concurrent read operations
  const reads = yield* Effect.all([
    db.executeQuery("SELECT * FROM users"),
    db.executeQuery("SELECT * FROM products"),
    db.executeQuery("SELECT * FROM orders")
  ], { concurrency: "unbounded" })
  
  // Exclusive transaction
  const transactionResult = yield* db.executeTransaction(
    Effect.gen(function* () {
      yield* db.executeQuery("UPDATE users SET last_login = NOW()")
      yield* db.executeQuery("INSERT INTO audit_log ...")
      return "transaction-complete"
    })
  )
  
  return { reads, transactionResult }
}).pipe(Effect.provide(databaseServiceLayer))
```

### Testing Strategies

```typescript
import { Array as Arr } from "effect"

// Test helpers for lock behavior
const createTestLock = () => STM.commit(TReentrantLock.make)

const testConcurrentReads = Effect.gen(function* () {
  const lock = yield* createTestLock()
  const results = yield* Effect.all([
    TReentrantLock.withReadLock(lock, Effect.succeed("read1")),
    TReentrantLock.withReadLock(lock, Effect.succeed("read2")),
    TReentrantLock.withReadLock(lock, Effect.succeed("read3"))
  ], { concurrency: "unbounded" })
  
  return results // Should all succeed concurrently
})

const testExclusiveWrite = Effect.gen(function* () {
  const lock = yield* createTestLock()
  const order = yield* STM.commit(TRef.make<Array<string>>([]))
  
  const operations = [
    TReentrantLock.withWriteLock(lock, Effect.gen(function* () {
      yield* STM.commit(TRef.update(order, (arr) => [...arr, "write1"]))
      yield* Effect.sleep("50 milli")
      return "write1"
    })),
    TReentrantLock.withWriteLock(lock, Effect.gen(function* () {
      yield* STM.commit(TRef.update(order, (arr) => [...arr, "write2"]))
      yield* Effect.sleep("50 milli")
      return "write2"
    }))
  ]
  
  yield* Effect.all(operations, { concurrency: "unbounded" })
  const finalOrder = yield* STM.commit(TRef.get(order))
  
  // Writes should be serialized
  return finalOrder.length === 2
})

// Performance testing
const performanceTest = Effect.gen(function* () {
  const lock = yield* createTestLock()
  const iterations = 1000
  
  const start = Date.now()
  
  yield* Effect.all(
    Arr.range(0, iterations - 1).map((i) =>
      TReentrantLock.withReadLock(lock, Effect.succeed(i))
    ),
    { concurrency: 10 }
  )
  
  const duration = Date.now() - start
  yield* Effect.logInfo(`${iterations} read operations completed in ${duration}ms`)
  
  return duration
})
```

## Conclusion

TReentrantLock provides thread-safe, reentrant read/write lock semantics for STM transactions and effectful code, enabling safe concurrent access to shared resources while maintaining composability.

Key benefits:
- **Concurrent Reads**: Multiple fibers can safely read shared state simultaneously
- **Exclusive Writes**: Write operations are serialized to prevent data races
- **Reentrant**: Same fiber can acquire multiple locks without deadlocking
- **STM Integration**: Composable with other STM primitives for complex coordination
- **Resource Safety**: Automatic cleanup through scoped operations

TReentrantLock is ideal for scenarios requiring controlled access to shared resources, such as database connection pools, caches, configuration management, and file system operations where read/write coordination is essential.