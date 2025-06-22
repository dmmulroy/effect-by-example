# Pool: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem Pool Solves

Resource management in modern applications is challenging. Creating and destroying expensive resources like database connections, HTTP clients, or file handles for every operation leads to performance bottlenecks, resource exhaustion, and unpredictable behavior.

```typescript
// Traditional approach - inefficient resource management
class DatabaseService {
  async queryUser(id: string) {
    // Creates new connection for every query - expensive!
    const connection = await createConnection({
      host: 'localhost',
      port: 5432,
      database: 'myapp'
    })
    
    try {
      const result = await connection.query('SELECT * FROM users WHERE id = $1', [id])
      return result.rows[0]
    } finally {
      // Manual cleanup - error-prone
      await connection.end()
    }
  }
  
  async queryOrder(id: string) {
    // Another expensive connection creation
    const connection = await createConnection({
      host: 'localhost', 
      port: 5432,
      database: 'myapp'
    })
    
    try {
      const result = await connection.query('SELECT * FROM orders WHERE id = $1', [id])
      return result.rows[0]
    } finally {
      await connection.end()
    }
  }
}
```

This approach leads to:
- **Resource Waste** - Repeatedly creating and destroying expensive resources
- **Performance Bottlenecks** - Connection establishment overhead for every operation
- **Resource Exhaustion** - No limits on concurrent resource usage
- **Error-Prone Cleanup** - Manual resource lifecycle management scattered throughout code
- **No Health Monitoring** - Failed resources aren't automatically replaced

### The Pool Solution

Effect's Pool module provides controlled, reusable resource management with automatic lifecycle handling, configurable sizing, and built-in health monitoring.

```typescript
import { Duration, Effect, Pool, Scope } from "effect"

// Define resource acquisition with proper cleanup
const acquireConnection = Effect.acquireRelease(
  Effect.promise(() => createConnection({
    host: 'localhost',
    port: 5432,
    database: 'myapp'
  })),
  (connection) => Effect.promise(() => connection.end())
)

// Create a managed pool
const connectionPool = Pool.make({
  acquire: acquireConnection,
  size: 10,
  concurrency: 1
})

// Use the pool - automatic resource management
const queryUser = (id: string) => Effect.gen(function* () {
  const connection = yield* connectionPool.get
  const result = yield* Effect.promise(() => 
    connection.query('SELECT * FROM users WHERE id = $1', [id])
  )
  return result.rows[0]
}).pipe(
  Effect.scoped // Automatic resource return to pool
)
```

### Key Concepts

**Pool**: A managed collection of reusable resources with controlled lifecycle and automatic cleanup

**Scoped Access**: Resources are borrowed from the pool within a scope and automatically returned when the scope closes

**Target Utilization**: Controls when new pool items are created based on current usage percentage (0.0 to 1.0)

**Concurrency Control**: Limits concurrent access per pool item using semaphore-based permits

**TTL Strategies**: Automatic resource expiration based on creation time or usage patterns

## Basic Usage Patterns

### Pattern 1: Fixed-Size Pool

```typescript
import { Effect, Pool } from "effect"

// Simple fixed-size pool for HTTP clients
const httpClientPool = Pool.make({
  acquire: Effect.sync(() => new HttpClient({ timeout: 5000 })),
  size: 5,
  concurrency: 1
})

// Use with automatic resource management
const fetchUser = (id: string) => Effect.gen(function* () {
  const client = yield* httpClientPool.get
  return yield* Effect.promise(() => client.get(`/users/${id}`))
}).pipe(
  Effect.scoped
)
```

### Pattern 2: Dynamic Pool with TTL

```typescript
import { Duration, Effect, Pool } from "effect"

// Database connection pool with automatic expiration
const dbPool = Pool.makeWithTTL({
  acquire: Effect.acquireRelease(
    Effect.promise(() => createConnection(dbConfig)),
    (conn) => Effect.promise(() => conn.close())
  ),
  min: 2,
  max: 10,
  timeToLive: Duration.minutes(5),
  timeToLiveStrategy: "usage" // Reset TTL on each use
})

const executeQuery = (sql: string, params: unknown[]) => Effect.gen(function* () {
  const connection = yield* dbPool.get
  return yield* Effect.promise(() => connection.query(sql, params))
}).pipe(
  Effect.scoped
)
```

### Pattern 3: High-Concurrency Pool

```typescript
// Pool allowing multiple concurrent operations per resource
const workerPool = Pool.make({
  acquire: Effect.sync(() => new WorkerThread()),
  size: 4,
  concurrency: 10, // Each worker can handle 10 concurrent tasks
  targetUtilization: 0.8 // Create new workers when 80% utilized
})

const processTask = (task: Task) => Effect.gen(function* () {
  const worker = yield* workerPool.get
  return yield* Effect.promise(() => worker.process(task))
}).pipe(
  Effect.scoped
)
```

## Real-World Examples

### Example 1: Database Connection Pool

A production-ready database service with connection pooling, health checks, and retry logic:

```typescript
import { Duration, Effect, Pool, Scope, Logger } from "effect"
import type { Connection, QueryResult } from "pg"
import { Client } from "pg"

// Database configuration
interface DbConfig {
  readonly host: string
  readonly port: number
  readonly database: string
  readonly user: string
  readonly password: string
}

// Connection with health check
const acquireDbConnection = (config: DbConfig) => Effect.gen(function* () {
  const logger = yield* Logger.Logger
  
  // Create connection
  yield* logger.info("Creating database connection")
  const client = new Client(config)
  yield* Effect.promise(() => client.connect())
  
  // Health check
  yield* Effect.promise(() => client.query('SELECT 1'))
  yield* logger.info("Database connection healthy")
  
  return client
}).pipe(
  Effect.acquireRelease((client) => Effect.gen(function* () {
    const logger = yield* Logger.Logger
    yield* logger.info("Closing database connection")
    yield* Effect.promise(() => client.end())
  }))
)

// Production database pool
const createDbPool = (config: DbConfig) => Pool.makeWithTTL({
  acquire: acquireDbConnection(config),
  min: 5,
  max: 20,
  timeToLive: Duration.minutes(10),
  timeToLiveStrategy: "usage",
  targetUtilization: 0.7,
  concurrency: 1
})

// Database service with retry and error handling
export const DatabaseService = Effect.gen(function* () {
  const config = yield* Effect.service(DbConfig)
  const pool = yield* createDbPool(config)
  
  const query = (sql: string, params: unknown[] = []) => Effect.gen(function* () {
    const connection = yield* pool.get
    const result = yield* Effect.promise(() => connection.query(sql, params))
    return result.rows
  }).pipe(
    Effect.scoped,
    Effect.retry({ times: 3, delay: Duration.seconds(1) }),
    Effect.timeout(Duration.seconds(30)),
    Effect.catchAll((error) => Effect.gen(function* () {
      const logger = yield* Logger.Logger
      yield* logger.error("Database query failed", { sql, error })
      return yield* Effect.fail(new DatabaseError({ cause: error, sql }))
    }))
  )
  
  const healthCheck = Effect.gen(function* () {
    const connection = yield* pool.get
    yield* Effect.promise(() => connection.query('SELECT version()'))
    return true
  }).pipe(
    Effect.scoped,
    Effect.catchAll(() => Effect.succeed(false))
  )
  
  return { query, healthCheck } as const
})

// Usage with automatic connection management
const getUserById = (id: string) => Effect.gen(function* () {
  const db = yield* DatabaseService
  const users = yield* db.query('SELECT * FROM users WHERE id = $1', [id])
  return users[0]
})

class DatabaseError extends Error {
  readonly _tag = "DatabaseError"
  constructor(readonly details: { cause: unknown; sql: string }) {
    super(`Database query failed: ${details.sql}`)
  }
}
```

### Example 2: HTTP Client Pool with Circuit Breaker

An HTTP service with connection pooling, circuit breaker pattern, and automatic failover:

```typescript
import { Duration, Effect, Pool, Schedule } from "effect"

interface HttpClientConfig {
  readonly baseUrl: string
  readonly timeout: number
  readonly maxRetries: number
}

// HTTP client with circuit breaker state
class HttpClientWithCircuitBreaker {
  private failures = 0
  private lastFailureTime = 0
  private readonly failureThreshold = 5
  private readonly resetTimeoutMs = 30000

  constructor(private readonly config: HttpClientConfig) {}

  async request(path: string, options?: RequestInit): Promise<Response> {
    if (this.isCircuitOpen()) {
      throw new Error("Circuit breaker is open")
    }

    try {
      const response = await fetch(`${this.config.baseUrl}${path}`, {
        ...options,
        timeout: this.config.timeout
      })
      
      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`)
      }
      
      this.onSuccess()
      return response
    } catch (error) {
      this.onFailure()
      throw error
    }
  }

  private isCircuitOpen(): boolean {
    if (this.failures >= this.failureThreshold) {
      const timeSinceLastFailure = Date.now() - this.lastFailureTime
      return timeSinceLastFailure < this.resetTimeoutMs
    }
    return false
  }

  private onSuccess() {
    this.failures = 0
  }

  private onFailure() {
    this.failures++
    this.lastFailureTime = Date.now()
  }
}

// HTTP client pool with health monitoring
const createHttpPool = (config: HttpClientConfig) => {
  const acquireClient = Effect.gen(function* () {
    const logger = yield* Logger.Logger
    yield* logger.info("Creating HTTP client", { baseUrl: config.baseUrl })
    
    const client = new HttpClientWithCircuitBreaker(config)
    
    // Health check
    yield* Effect.promise(() => client.request('/health'))
    yield* logger.info("HTTP client healthy", { baseUrl: config.baseUrl })
    
    return client
  }).pipe(
    Effect.acquireRelease((client) => Effect.gen(function* () {
      const logger = yield* Logger.Logger
      yield* logger.info("Releasing HTTP client", { baseUrl: config.baseUrl })
    }))
  )

  return Pool.makeWithTTL({
    acquire: acquireClient,
    min: 2,
    max: 8,
    timeToLive: Duration.minutes(15),
    concurrency: 5, // Each client can handle 5 concurrent requests
    targetUtilization: 0.8
  })
}

// HTTP service with automatic retry and failover
export const HttpService = Effect.gen(function* () {
  const config = yield* Effect.service(HttpClientConfig)
  const pool = yield* createHttpPool(config)
  
  const request = (path: string, options?: RequestInit) => Effect.gen(function* () {
    const client = yield* pool.get
    const response = yield* Effect.promise(() => client.request(path, options))
    return yield* Effect.promise(() => response.json())
  }).pipe(
    Effect.scoped,
    Effect.retry(Schedule.exponential(Duration.seconds(1)).pipe(
      Schedule.intersect(Schedule.recurs(3))
    )),
    Effect.timeout(Duration.seconds(30)),
    Effect.catchTag("TimeoutException", () => 
      Effect.fail(new HttpTimeoutError({ path, timeout: config.timeout }))
    )
  )
  
  return { request } as const
})

class HttpTimeoutError extends Error {
  readonly _tag = "HttpTimeoutError"
  constructor(readonly details: { path: string; timeout: number }) {
    super(`HTTP request timed out: ${details.path} (${details.timeout}ms)`)
  }
}
```

### Example 3: File Processing Worker Pool

A worker pool for CPU-intensive file processing with load balancing and progress tracking:

```typescript
import { Duration, Effect, Pool, Queue, Ref } from "effect"
import { Worker } from "worker_threads"
import * as path from "path"

interface FileProcessingTask {
  readonly id: string
  readonly inputPath: string
  readonly outputPath: string
  readonly options: ProcessingOptions
}

interface ProcessingOptions {
  readonly format: "jpeg" | "png" | "webp"
  readonly quality: number
  readonly width?: number
  readonly height?: number
}

interface WorkerStats {
  readonly id: string
  readonly tasksProcessed: number
  readonly lastActivity: number
  readonly isProcessing: boolean
}

// Enhanced worker with stats tracking
class FileProcessingWorker {
  private tasksProcessed = 0
  private lastActivity = Date.now()
  private isProcessing = false

  constructor(
    private readonly worker: Worker,
    public readonly id: string
  ) {
    this.worker.on('message', () => {
      this.isProcessing = false
      this.tasksProcessed++
      this.lastActivity = Date.now()
    })
  }

  async processFile(task: FileProcessingTask): Promise<ProcessingResult> {
    this.isProcessing = true
    this.lastActivity = Date.now()
    
    return new Promise((resolve, reject) => {
      const timeout = setTimeout(() => {
        reject(new Error(`Processing timeout for task ${task.id}`))
      }, 300000) // 5 minutes timeout

      this.worker.postMessage(task)
      
      this.worker.once('message', (result) => {
        clearTimeout(timeout)
        if (result.error) {
          reject(new Error(result.error))
        } else {
          resolve(result)
        }
      })
    })
  }

  getStats(): WorkerStats {
    return {
      id: this.id,
      tasksProcessed: this.tasksProcessed,
      lastActivity: this.lastActivity,
      isProcessing: this.isProcessing
    }
  }

  terminate(): Promise<number> {
    return this.worker.terminate()
  }
}

interface ProcessingResult {
  readonly taskId: string
  readonly outputPath: string
  readonly processingTime: number
  readonly outputSize: number
}

// Worker pool with intelligent load balancing
const createWorkerPool = () => {
  const acquireWorker = Effect.gen(function* () {
    const logger = yield* Logger.Logger
    const workerId = `worker-${Math.random().toString(36).substr(2, 9)}`
    
    yield* logger.info("Creating file processing worker", { workerId })
    
    const worker = new Worker(path.join(__dirname, 'file-processor-worker.js'))
    const fileWorker = new FileProcessingWorker(worker, workerId)
    
    // Health check - process a simple test image
    const testResult = yield* Effect.promise(() => fileWorker.processFile({
      id: 'health-check',
      inputPath: path.join(__dirname, 'test-image.jpg'),
      outputPath: '/tmp/health-check.jpg',
      options: { format: 'jpeg', quality: 80 }
    }))
    
    yield* logger.info("Worker health check passed", { workerId, testResult })
    
    return fileWorker
  }).pipe(
    Effect.acquireRelease((worker) => Effect.gen(function* () {
      const logger = yield* Logger.Logger
      yield* logger.info("Terminating worker", { workerId: worker.id })
      yield* Effect.promise(() => worker.terminate())
    }))
  )

  return Pool.makeWithTTL({
    acquire: acquireWorker,
    min: 2,
    max: 8, // Based on CPU cores
    timeToLive: Duration.minutes(30),
    concurrency: 1, // One file per worker at a time
    targetUtilization: 0.9 // High utilization for CPU-bound tasks
  })
}

// File processing service with progress tracking
export const FileProcessingService = Effect.gen(function* () {
  const pool = yield* createWorkerPool()
  const progressRef = yield* Ref.make(new Map<string, number>())
  const statsRef = yield* Ref.make<WorkerStats[]>([])
  
  const processFile = (task: FileProcessingTask) => Effect.gen(function* () {
    // Update progress
    yield* Ref.update(progressRef, (progress) => 
      progress.set(task.id, 0)
    )
    
    const worker = yield* pool.get
    
    // Process with progress updates
    const result = yield* Effect.promise(() => worker.processFile(task))
    
    // Complete progress
    yield* Ref.update(progressRef, (progress) => 
      progress.set(task.id, 100)
    )
    
    return result
  }).pipe(
    Effect.scoped,
    Effect.timeout(Duration.minutes(10)),
    Effect.retry(Schedule.exponential(Duration.seconds(2)).pipe(
      Schedule.intersect(Schedule.recurs(2))
    )),
    Effect.catchAll((error) => Effect.gen(function* () {
      // Clean up progress on failure
      yield* Ref.update(progressRef, (progress) => {
        progress.delete(task.id)
        return progress
      })
      return yield* Effect.fail(new FileProcessingError({ 
        taskId: task.id, 
        cause: error 
      }))
    }))
  )
  
  const batchProcess = (tasks: FileProcessingTask[]) => Effect.gen(function* () {
    const results = yield* Effect.forEach(tasks, processFile, {
      concurrency: 4 // Process 4 files concurrently
    })
    return results
  })
  
  const getProgress = (taskId: string) => Effect.gen(function* () {
    const progress = yield* Ref.get(progressRef)
    return progress.get(taskId) ?? null
  })
  
  const getWorkerStats = Effect.gen(function* () {
    // This would ideally poll worker stats in a real implementation
    return yield* Ref.get(statsRef)
  })
  
  return { 
    processFile, 
    batchProcess, 
    getProgress, 
    getWorkerStats 
  } as const
})

class FileProcessingError extends Error {
  readonly _tag = "FileProcessingError"
  constructor(readonly details: { taskId: string; cause: unknown }) {
    super(`File processing failed for task ${details.taskId}`)
  }
}

// Usage example with batch processing
const processImageBatch = (imagePaths: string[]) => Effect.gen(function* () {
  const processingService = yield* FileProcessingService
  
  const tasks = imagePaths.map((inputPath, index) => ({
    id: `batch-task-${index}`,
    inputPath,
    outputPath: inputPath.replace(/\.[^.]+$/, '-processed.webp'),
    options: {
      format: 'webp' as const,
      quality: 85,
      width: 1920
    }
  }))
  
  const results = yield* processingService.batchProcess(tasks)
  
  yield* Effect.forEach(results, (result) => Effect.gen(function* () {
    const logger = yield* Logger.Logger
    yield* logger.info("File processed successfully", {
      taskId: result.taskId,
      outputPath: result.outputPath,
      processingTime: result.processingTime,
      outputSize: result.outputSize
    })
  }))
  
  return results
})
```

## Advanced Features Deep Dive

### Feature 1: Target Utilization Strategy

Target utilization controls when the pool creates new resources based on current usage patterns.

#### Basic Target Utilization Usage

```typescript
// Conservative approach - create resources early
const eagerPool = Pool.make({
  acquire: acquireResource,
  size: 10,
  targetUtilization: 0.5 // Create new resources at 50% utilization
})

// Aggressive approach - maximize resource usage
const efficientPool = Pool.make({
  acquire: acquireResource,
  size: 10,
  targetUtilization: 0.9 // Create resources only at 90% utilization
})
```

#### Real-World Target Utilization Example

```typescript
// Dynamic utilization based on application load
const createAdaptivePool = (loadMetrics: LoadMetrics) => {
  const targetUtilization = Effect.gen(function* () {
    const currentLoad = yield* loadMetrics.getCurrentLoad()
    
    // Adjust utilization based on system load
    if (currentLoad > 0.8) {
      return 0.6 // Create resources early under high load
    } else if (currentLoad < 0.3) {
      return 0.9 // Conserve resources under low load
    }
    return 0.75 // Balanced approach
  })
  
  return Effect.gen(function* () {
    const utilization = yield* targetUtilization
    return yield* Pool.make({
      acquire: acquireExpensiveResource,
      size: 20,
      targetUtilization: utilization
    })
  })
}
```

#### Advanced Target Utilization: Load-Based Scaling

```typescript
interface PoolMetrics {
  readonly currentSize: number
  readonly activeConnections: number
  readonly queueLength: number
  readonly averageWaitTime: number
}

const createIntelligentPool = <A, E, R>(
  acquire: Effect.Effect<A, E, R>,
  baseConfig: {
    minSize: number
    maxSize: number
    targetLatency: number
  }
) => Effect.gen(function* () {
  const metricsRef = yield* Ref.make<PoolMetrics>({
    currentSize: baseConfig.minSize,
    activeConnections: 0,
    queueLength: 0,
    averageWaitTime: 0
  })
  
  // Dynamic utilization calculator
  const calculateTargetUtilization = Effect.gen(function* () {
    const metrics = yield* Ref.get(metricsRef)
    
    if (metrics.averageWaitTime > baseConfig.targetLatency) {
      return 0.5 // Aggressive scaling under latency pressure
    } else if (metrics.queueLength > 10) {
      return 0.6 // Scale up when queue builds
    }
    return 0.8 // Conservative scaling
  })
  
  const utilization = yield* calculateTargetUtilization
  
  return yield* Pool.makeWithTTL({
    acquire,
    min: baseConfig.minSize,
    max: baseConfig.maxSize,
    targetUtilization: utilization,
    timeToLive: Duration.minutes(5)
  })
})
```

### Feature 2: TTL Strategies Deep Dive

Time-to-live strategies determine when pool resources are considered stale and need replacement.

#### Creation-Based TTL

```typescript
// Resources expire based on creation time
const creationTTLPool = Pool.makeWithTTL({
  acquire: acquireConnection,
  min: 5,
  max: 15,
  timeToLive: Duration.minutes(10),
  timeToLiveStrategy: "creation" // Expire 10 minutes after creation
})

// Useful for resources that degrade over time regardless of usage
const cachePool = Pool.makeWithTTL({
  acquire: acquireRedisConnection,
  min: 3,
  max: 12,
  timeToLive: Duration.hours(1),
  timeToLiveStrategy: "creation" // Periodic refresh for long-lived caches
})
```

#### Usage-Based TTL

```typescript
// Resources expire based on last usage time
const usageTTLPool = Pool.makeWithTTL({
  acquire: acquireDBConnection,
  min: 2,
  max: 10,
  timeToLive: Duration.minutes(5),
  timeToLiveStrategy: "usage" // Reset TTL on each use
})

// Perfect for databases with idle connection timeouts
const postgresPool = Pool.makeWithTTL({
  acquire: acquirePostgresConnection,
  min: 5,
  max: 20,
  timeToLive: Duration.minutes(30),
  timeToLiveStrategy: "usage" // Keep active connections alive
})
```

#### Advanced TTL: Health-Based Expiration

```typescript
// Custom TTL logic based on resource health
const createHealthAwarePool = <A extends HealthCheckable, E, R>(
  acquire: Effect.Effect<A, E, R>,
  healthCheck: (resource: A) => Effect.Effect<boolean>,
  ttlConfig: {
    readonly healthyTTL: Duration.DurationInput
    readonly unhealthyTTL: Duration.DurationInput
    readonly checkInterval: Duration.DurationInput
  }
) => Effect.gen(function* () {
  const healthStatusRef = yield* Ref.make(new Map<A, boolean>())
  
  // Background health monitoring
  const healthMonitor = Effect.gen(function* () {
    const status = yield* Ref.get(healthStatusRef)
    
    yield* Effect.forEach(status.keys(), (resource) => Effect.gen(function* () {
      const isHealthy = yield* healthCheck(resource)
      yield* Ref.update(healthStatusRef, (map) => map.set(resource, isHealthy))
      
      if (!isHealthy) {
        // Invalidate unhealthy resources immediately
        yield* pool.invalidate(resource)
      }
    }), { concurrency: 5 })
  }).pipe(
    Effect.repeat(Schedule.fixed(ttlConfig.checkInterval)),
    Effect.fork
  )
  
  const pool = yield* Pool.makeWithTTL({
    acquire: Effect.gen(function* () {
      const resource = yield* acquire
      yield* Ref.update(healthStatusRef, (map) => map.set(resource, true))
      return resource
    }),
    min: 2,
    max: 10,
    timeToLive: ttlConfig.healthyTTL,
    timeToLiveStrategy: "usage"
  })
  
  yield* healthMonitor
  
  return pool
})

interface HealthCheckable {
  health(): Promise<boolean>
}
```

### Feature 3: Concurrency Control Patterns

Pool concurrency controls how many operations can use each resource simultaneously.

#### Single-Access Resources

```typescript
// Database connections - one query at a time
const dbPool = Pool.make({
  acquire: acquireConnection,
  size: 10,
  concurrency: 1 // One query per connection
})

// File handles - one operation at a time
const filePool = Pool.make({
  acquire: acquireFileHandle,
  size: 5,
  concurrency: 1 // One file operation per handle
})
```

#### Multi-Access Resources

```typescript
// HTTP clients - multiple concurrent requests
const httpPool = Pool.make({
  acquire: acquireHttpClient,
  size: 3,
  concurrency: 10 // Each client handles 10 concurrent requests
})

// Worker threads - multiple tasks per worker
const workerPool = Pool.make({
  acquire: acquireWorker,
  size: 4,
  concurrency: 5 // Each worker processes 5 tasks concurrently
})
```

#### Advanced Concurrency: Dynamic Permit Management

```typescript
import { Semaphore } from "effect"

// Pool with dynamic concurrency based on resource performance
const createAdaptiveConcurrencyPool = <A extends PerformanceAware, E, R>(
  acquire: Effect.Effect<A, E, R>,
  config: {
    readonly baseSize: number
    readonly baseConcurrency: number
    readonly maxConcurrency: number
  }
) => Effect.gen(function* () {
  const performanceRef = yield* Ref.make(new Map<A, number>())
  
  // Custom resource with performance tracking
  const acquireWithTracking = Effect.gen(function* () {
    const resource = yield* acquire
    const semaphore = yield* Semaphore.make(config.baseConcurrency)
    
    const trackedResource = {
      ...resource,
      withPermit: <T>(effect: Effect.Effect<T>) => semaphore.withPermits(1)(effect),
      adjustConcurrency: (permits: number) => Effect.gen(function* () {
        const current = semaphore.available
        const adjustment = permits - current
        
        if (adjustment > 0) {
          yield* semaphore.release(adjustment)
        } else if (adjustment < 0) {
          yield* semaphore.withPermits(Math.abs(adjustment))(Effect.unit)
        }
      })
    }
    
    return trackedResource
  })
  
  // Performance monitoring and adjustment
  const performanceMonitor = Effect.gen(function* () {
    const performance = yield* Ref.get(performanceRef)
    
    yield* Effect.forEach(performance.entries(), ([resource, avgLatency]) => Effect.gen(function* () {
      let newConcurrency = config.baseConcurrency
      
      if (avgLatency < 100) {
        newConcurrency = Math.min(config.maxConcurrency, config.baseConcurrency * 2)
      } else if (avgLatency > 1000) {
        newConcurrency = Math.max(1, Math.floor(config.baseConcurrency / 2))
      }
      
      yield* resource.adjustConcurrency(newConcurrency)
    }))
  }).pipe(
    Effect.repeat(Schedule.fixed(Duration.seconds(30))),
    Effect.fork
  )
  
  const pool = yield* Pool.make({
    acquire: acquireWithTracking,
    size: config.baseSize,
    concurrency: 1 // Each pool item manages its own concurrency
  })
  
  yield* performanceMonitor
  
  return pool
})

interface PerformanceAware {
  getAverageLatency(): number
}
```

## Practical Patterns & Best Practices

### Pattern 1: Pool Health Monitoring

```typescript
// Comprehensive pool health monitoring
const createMonitoredPool = <A, E, R>(
  name: string,
  acquire: Effect.Effect<A, E, R>,
  config: PoolConfig
) => Effect.gen(function* () {
  const healthRef = yield* Ref.make<PoolHealth>({
    totalAcquired: 0,
    totalReleased: 0,
    currentActive: 0,
    failedAcquisitions: 0,
    averageAcquisitionTime: 0,
    lastHealthCheck: Date.now()
  })
  
  const instrumentedAcquire = Effect.gen(function* () {
    const startTime = Date.now()
    
    const resource = yield* acquire.pipe(
      Effect.tap(() => Ref.update(healthRef, (health) => ({
        ...health,
        totalAcquired: health.totalAcquired + 1,
        currentActive: health.currentActive + 1,
        averageAcquisitionTime: (health.averageAcquisitionTime + (Date.now() - startTime)) / 2
      }))),
      Effect.tapError(() => Ref.update(healthRef, (health) => ({
        ...health,
        failedAcquisitions: health.failedAcquisitions + 1
      })))
    )
    
    return resource
  }).pipe(
    Effect.acquireRelease((resource) => Ref.update(healthRef, (health) => ({
      ...health,
      totalReleased: health.totalReleased + 1,
      currentActive: Math.max(0, health.currentActive - 1)
    })))
  )
  
  const pool = yield* Pool.makeWithTTL({
    acquire: instrumentedAcquire,
    ...config
  })
  
  const getHealthMetrics = Effect.gen(function* () {
    const health = yield* Ref.get(healthRef)
    return {
      poolName: name,
      ...health,
      successRate: health.totalAcquired > 0 ? 
        (health.totalAcquired - health.failedAcquisitions) / health.totalAcquired : 1,
      resourceLeakage: health.totalAcquired - health.totalReleased - health.currentActive
    }
  })
  
  // Periodic health reporting
  const healthReporter = Effect.gen(function* () {
    const logger = yield* Logger.Logger
    const metrics = yield* getHealthMetrics
    
    yield* logger.info("Pool health report", metrics)
    
    // Alert on issues
    if (metrics.successRate < 0.95) {
      yield* logger.warn("Pool has low success rate", { 
        poolName: name, 
        successRate: metrics.successRate 
      })
    }
    
    if (metrics.resourceLeakage > 0) {
      yield* logger.error("Pool resource leakage detected", {
        poolName: name,
        leakage: metrics.resourceLeakage
      })
    }
  }).pipe(
    Effect.repeat(Schedule.fixed(Duration.minutes(5))),
    Effect.fork
  )
  
  yield* healthReporter
  
  return { pool, getHealthMetrics } as const
})

interface PoolConfig {
  readonly min: number
  readonly max: number
  readonly timeToLive: Duration.DurationInput
  readonly concurrency?: number
  readonly targetUtilization?: number
}

interface PoolHealth {
  readonly totalAcquired: number
  readonly totalReleased: number
  readonly currentActive: number
  readonly failedAcquisitions: number
  readonly averageAcquisitionTime: number
  readonly lastHealthCheck: number
}
```

### Pattern 2: Pool Warm-up Strategy

```typescript
// Pool warm-up for predictable performance
const createWarmPool = <A, E, R>(
  acquire: Effect.Effect<A, E, R>,
  config: {
    readonly size: number
    readonly warmupPercentage: number
    readonly warmupTimeout: Duration.DurationInput
  }
) => Effect.gen(function* () {
  const pool = yield* Pool.make({
    acquire,
    size: config.size,
    targetUtilization: 0.1 // Aggressive pre-allocation
  })
  
  // Pre-warm the pool
  const warmupCount = Math.ceil(config.size * config.warmupPercentage)
  const logger = yield* Logger.Logger
  
  yield* logger.info("Starting pool warm-up", { 
    targetSize: config.size, 
    warmupCount 
  })
  
  const warmupResources = yield* Effect.gen(function* () {
    return yield* Effect.forEach(
      Array.from({ length: warmupCount }, (_, i) => i),
      () => pool.get,
      { concurrency: Math.min(warmupCount, 5) }
    )
  }).pipe(
    Effect.scoped,
    Effect.timeout(config.warmupTimeout),
    Effect.catchAll((error) => Effect.gen(function* () {
      yield* logger.warn("Pool warm-up failed", { error })
      return []
    }))
  )
  
  yield* logger.info("Pool warm-up completed", { 
    warmedResources: warmupResources.length 
  })
  
  return pool
})

// Usage with automatic warm-up
const warmDatabasePool = Effect.gen(function* () {
  const config = yield* Effect.service(DatabaseConfig)
  
  return yield* createWarmPool(
    acquireDbConnection(config),
    {
      size: 20,
      warmupPercentage: 0.5, // Warm-up 50% of pool
      warmupTimeout: Duration.seconds(30)
    }
  )
})
```

### Pattern 3: Pool Resize and Circuit Breaker

```typescript
// Dynamic pool resizing with circuit breaker
const createAdaptivePool = <A, E, R>(
  acquire: Effect.Effect<A, E, R>,
  initialConfig: {
    readonly minSize: number
    readonly maxSize: number
    readonly initialSize: number
  }
) => Effect.gen(function* () {
  const configRef = yield* Ref.make(initialConfig)
  const circuitStateRef = yield* Ref.make<CircuitState>({ 
    state: "closed", 
    failures: 0, 
    lastFailure: 0 
  })
  
  // Circuit breaker logic
  const withCircuitBreaker = <T>(effect: Effect.Effect<T, E>) => Effect.gen(function* () {
    const circuitState = yield* Ref.get(circuitStateRef)
    
    if (circuitState.state === "open") {
      const timeSinceLastFailure = Date.now() - circuitState.lastFailure
      if (timeSinceLastFailure < 30000) { // 30 second timeout
        return yield* Effect.fail(new Error("Circuit breaker is open") as any)
      }
      // Try to half-open
      yield* Ref.set(circuitStateRef, { ...circuitState, state: "half-open" })
    }
    
    return yield* effect.pipe(
      Effect.tap(() => Effect.gen(function* () {
        // Success - close circuit
        yield* Ref.set(circuitStateRef, { state: "closed", failures: 0, lastFailure: 0 })
      })),
      Effect.tapError(() => Effect.gen(function* () {
        // Failure - increment failure count
        const current = yield* Ref.get(circuitStateRef)
        const newFailures = current.failures + 1
        
        yield* Ref.set(circuitStateRef, {
          state: newFailures >= 5 ? "open" : "closed",
          failures: newFailures,
          lastFailure: Date.now()
        })
      }))
    )
  })
  
  // Protected acquire function
  const protectedAcquire = withCircuitBreaker(acquire)
  
  // Dynamic pool creation
  const createPool = (config: typeof initialConfig) => Pool.makeWithTTL({
    acquire: protectedAcquire,
    min: config.minSize,
    max: config.maxSize,
    timeToLive: Duration.minutes(10),
    targetUtilization: 0.8
  })
  
  let currentPool = yield* createPool(initialConfig)
  
  // Pool resizing logic
  const resizePool = (newConfig: typeof initialConfig) => Effect.gen(function* () {
    const logger = yield* Logger.Logger
    yield* logger.info("Resizing pool", { oldConfig: yield* Ref.get(configRef), newConfig })
    
    // Create new pool
    const newPool = yield* createPool(newConfig)
    
    // Update references
    yield* Ref.set(configRef, newConfig)
    currentPool = newPool
    
    yield* logger.info("Pool resized successfully")
  })
  
  // Auto-scaling based on metrics
  const autoScale = Effect.gen(function* () {
    const circuitState = yield* Ref.get(circuitStateRef)
    const currentConfig = yield* Ref.get(configRef)
    
    if (circuitState.failures > 3) {
      // Scale down on failures
      const newSize = Math.max(
        currentConfig.minSize,
        Math.floor(currentConfig.initialSize * 0.7)
      )
      yield* resizePool({ ...currentConfig, initialSize: newSize })
    }
    // Add scale-up logic based on usage metrics
  }).pipe(
    Effect.repeat(Schedule.fixed(Duration.minutes(2))),
    Effect.fork
  )
  
  yield* autoScale
  
  return {
    get: currentPool.get,
    invalidate: currentPool.invalidate,
    resize: resizePool,
    getCircuitState: Ref.get(circuitStateRef)
  } as const
})

interface CircuitState {
  readonly state: "open" | "closed" | "half-open"
  readonly failures: number
  readonly lastFailure: number
}
```

## Integration Examples

### Integration with Express.js HTTP Server

```typescript
import express from "express"
import { Effect, Layer, Pool } from "effect"

// Database pool for Express middleware
const createExpressDbPool = (dbConfig: DatabaseConfig) => Pool.makeWithTTL({
  acquire: Effect.acquireRelease(
    Effect.promise(() => createConnection(dbConfig)),
    (conn) => Effect.promise(() => conn.end())
  ),
  min: 5,
  max: 25,
  timeToLive: Duration.minutes(15),
  concurrency: 1
})

// Effect-based request handler
const createEffectHandler = <T>(
  effectHandler: (req: express.Request) => Effect.Effect<T, Error, DatabaseService>
) => {
  return (req: express.Request, res: express.Response, next: express.NextFunction) => {
    const program = Effect.gen(function* () {
      const result = yield* effectHandler(req)
      res.json({ success: true, data: result })
    }).pipe(
      Effect.catchAll((error) => Effect.sync(() => {
        res.status(500).json({ success: false, error: error.message })
      }))
    )
    
    Effect.runPromise(program.pipe(
      Effect.provide(DatabaseService.Live)
    )).catch(next)
  }
}

// Express app with pooled database connections
const createApp = () => Effect.gen(function* () {
  const dbConfig = yield* Effect.service(DatabaseConfig)
  const pool = yield* createExpressDbPool(dbConfig)
  
  const DatabaseService = Effect.gen(function* () {
    const query = (sql: string, params: unknown[]) => Effect.gen(function* () {
      const connection = yield* pool.get
      return yield* Effect.promise(() => connection.query(sql, params))
    }).pipe(Effect.scoped)
    
    return { query } as const
  })
  
  const DatabaseServiceLive = Layer.effect(
    "DatabaseService", 
    DatabaseService
  )
  
  const app = express()
  app.use(express.json())
  
  // Effect-powered routes
  app.get('/users/:id', createEffectHandler((req) => Effect.gen(function* () {
    const db = yield* Effect.service(DatabaseService)
    const users = yield* db.query('SELECT * FROM users WHERE id = $1', [req.params.id])
    return users[0]
  })))
  
  app.get('/health', createEffectHandler(() => Effect.gen(function* () {
    const db = yield* Effect.service(DatabaseService)
    yield* db.query('SELECT 1')
    return { status: 'healthy', timestamp: new Date().toISOString() }
  })))
  
  return { app, layer: DatabaseServiceLive } as const
})
```

### Integration with Redis Cache

```typescript
import { Redis } from "ioredis"
import { Cache, Duration, Effect, Pool } from "effect"

// Redis connection pool with automatic retry
const createRedisPool = (config: RedisConfig) => Pool.makeWithTTL({
  acquire: Effect.acquireRelease(
    Effect.gen(function* () {
      const logger = yield* Logger.Logger
      yield* logger.info("Creating Redis connection")
      
      const redis = new Redis({
        host: config.host,
        port: config.port,
        retryDelayOnFailover: 100,
        maxRetriesPerRequest: 3
      })
      
      // Test connection
      yield* Effect.promise(() => redis.ping())
      
      return redis
    }),
    (redis) => Effect.gen(function* () {
      const logger = yield* Logger.Logger
      yield* logger.info("Closing Redis connection")
      yield* Effect.promise(() => redis.quit())
    })
  ),
  min: 2,
  max: 10,
  timeToLive: Duration.minutes(30),
  concurrency: 5 // Redis can handle multiple concurrent operations
})

// Effect Cache with Redis backend
const createRedisCacheLayer = (redisConfig: RedisConfig) => Effect.gen(function* () {
  const pool = yield* createRedisPool(redisConfig)
  
  // Redis-backed cache implementation
  const RedisCacheService = Effect.gen(function* () {
    const get = (key: string) => Effect.gen(function* () {
      const redis = yield* pool.get
      const value = yield* Effect.promise(() => redis.get(key))
      return value ? JSON.parse(value) : null
    }).pipe(Effect.scoped)
    
    const set = (key: string, value: unknown, ttl?: Duration.DurationInput) => Effect.gen(function* () {
      const redis = yield* pool.get
      const serialized = JSON.stringify(value)
      
      if (ttl) {
        const seconds = Math.floor(Duration.toMillis(ttl) / 1000)
        yield* Effect.promise(() => redis.setex(key, seconds, serialized))
      } else {
        yield* Effect.promise(() => redis.set(key, serialized))
      }
    }).pipe(Effect.scoped)
    
    const del = (key: string) => Effect.gen(function* () {
      const redis = yield* pool.get
      yield* Effect.promise(() => redis.del(key))
    }).pipe(Effect.scoped)
    
    const clear = Effect.gen(function* () {
      const redis = yield* pool.get
      yield* Effect.promise(() => redis.flushdb())
    }).pipe(Effect.scoped)
    
    return { get, set, del, clear } as const
  })
  
  return Layer.effect("CacheService", RedisCacheService)
})

// Application layer with Redis cache
const createCachedUserService = Effect.gen(function* () {
  const cache = yield* Effect.service(CacheService)
  const db = yield* Effect.service(DatabaseService)
  
  const getUser = (id: string) => Effect.gen(function* () {
    // Try cache first
    const cached = yield* cache.get(`user:${id}`)
    if (cached) {
      return cached as User
    }
    
    // Fallback to database
    const user = yield* db.query('SELECT * FROM users WHERE id = $1', [id])
      .pipe(Effect.map(rows => rows[0]))
    
    // Cache for 10 minutes
    yield* cache.set(`user:${id}`, user, Duration.minutes(10))
    
    return user
  })
  
  const updateUser = (id: string, updates: Partial<User>) => Effect.gen(function* () {
    // Update database
    const updatedUser = yield* db.query(
      'UPDATE users SET name = $1, email = $2 WHERE id = $3 RETURNING *',
      [updates.name, updates.email, id]
    ).pipe(Effect.map(rows => rows[0]))
    
    // Invalidate cache
    yield* cache.del(`user:${id}`)
    
    return updatedUser
  })
  
  return { getUser, updateUser } as const
})

interface CacheService {
  readonly get: (key: string) => Effect.Effect<unknown | null>
  readonly set: (key: string, value: unknown, ttl?: Duration.DurationInput) => Effect.Effect<void>
  readonly del: (key: string) => Effect.Effect<void>
  readonly clear: Effect.Effect<void>
}

interface User {
  readonly id: string
  readonly name: string
  readonly email: string
}

interface RedisConfig {
  readonly host: string
  readonly port: number
}
```

### Testing Strategies

```typescript
import { Effect, Layer, Pool, TestServices } from "effect"
import { describe, it, expect } from "@effect/vitest"

// Mock resource for testing
class MockDatabase {
  private queries: string[] = []
  private shouldFail = false
  
  constructor(public readonly id: string) {}
  
  async query(sql: string): Promise<unknown[]> {
    this.queries.push(sql)
    
    if (this.shouldFail) {
      throw new Error("Database connection failed")
    }
    
    return [{ id: 1, name: "Test User" }]
  }
  
  getQueries(): string[] {
    return [...this.queries]
  }
  
  setShouldFail(fail: boolean): void {
    this.shouldFail = fail
  }
  
  async close(): Promise<void> {
    // Mock cleanup
  }
}

// Test utilities
const createTestPool = (config: { size?: number; shouldFailAcquisition?: boolean } = {}) => {
  let createdConnections = 0
  
  const acquire = Effect.gen(function* () {
    if (config.shouldFailAcquisition) {
      return yield* Effect.fail(new Error("Failed to acquire resource"))
    }
    
    const id = `test-db-${++createdConnections}`
    return new MockDatabase(id)
  }).pipe(
    Effect.acquireRelease((db) => Effect.promise(() => db.close()))
  )
  
  return Pool.make({
    acquire,
    size: config.size ?? 3,
    concurrency: 1
  })
}

describe("Pool", () => {
  it("should create and reuse resources", () => Effect.gen(function* () {
    const pool = yield* createTestPool({ size: 2 })
    
    // Get first resource
    const db1 = yield* pool.get
    const query1Result = yield* Effect.promise(() => db1.query("SELECT 1"))
    
    expect(query1Result).toEqual([{ id: 1, name: "Test User" }])
    expect(db1.id).toMatch(/test-db-\d+/)
    
    // Get second resource (should reuse from pool)
    const db2 = yield* pool.get
    const query2Result = yield* Effect.promise(() => db2.query("SELECT 2"))
    
    expect(query2Result).toEqual([{ id: 1, name: "Test User" }])
    expect(db2.id).toMatch(/test-db-\d+/)
  }).pipe(
    Effect.scoped,
    Effect.provide(TestServices.layer)
  ))
  
  it("should handle resource acquisition failures", () => Effect.gen(function* () {
    const pool = yield* createTestPool({ shouldFailAcquisition: true })
    
    const result = yield* pool.get.pipe(
      Effect.either
    )
    
    expect(result._tag).toBe("Left")
    if (result._tag === "Left") {
      expect(result.left.message).toBe("Failed to acquire resource")
    }
  }).pipe(
    Effect.scoped,
    Effect.provide(TestServices.layer)
  ))
  
  it("should handle concurrent access", () => Effect.gen(function* () {
    const pool = yield* createTestPool({ size: 2 })
    
    // Simulate 5 concurrent operations with 2 pool resources
    const operations = Array.from({ length: 5 }, (_, i) => 
      Effect.gen(function* () {
        const db = yield* pool.get
        const result = yield* Effect.promise(() => db.query(`SELECT ${i}`))
        return { connectionId: db.id, result }
      }).pipe(Effect.scoped)
    )
    
    const results = yield* Effect.forEach(operations, (op) => op, {
      concurrency: 3
    })
    
    expect(results).toHaveLength(5)
    
    // Should reuse the 2 pool connections
    const uniqueConnections = new Set(results.map(r => r.connectionId))
    expect(uniqueConnections.size).toBeLessThanOrEqual(2)
  }).pipe(
    Effect.provide(TestServices.layer)
  ))
  
  it("should invalidate resources", () => Effect.gen(function* () {
    const pool = yield* createTestPool({ size: 1 })
    
    // Get resource and mark it as failed
    const db1 = yield* pool.get.pipe(Effect.scoped)
    db1.setShouldFail(true)
    
    // Invalidate the resource
    yield* pool.invalidate(db1)
    
    // Get a new resource (should be different)
    const db2 = yield* pool.get
    expect(db2.id).not.toBe(db1.id)
    
    // New resource should work
    const result = yield* Effect.promise(() => db2.query("SELECT 1"))
    expect(result).toEqual([{ id: 1, name: "Test User" }])
  }).pipe(
    Effect.scoped,
    Effect.provide(TestServices.layer)
  ))
})

// Property-based testing with fast-check
import { fc } from "@fast-check/effect"

describe("Pool Property Tests", () => {
  it("should never exceed maximum pool size", () => 
    fc.assert(fc.asyncProperty(
      fc.integer({ min: 1, max: 10 }),
      fc.integer({ min: 1, max: 20 }),
      (poolSize, operationCount) => Effect.gen(function* () {
        const acquiredResources = new Set<string>()
        let maxConcurrentResources = 0
        
        const trackingAcquire = Effect.gen(function* () {
          const id = `resource-${Math.random()}`
          acquiredResources.add(id)
          maxConcurrentResources = Math.max(maxConcurrentResources, acquiredResources.size)
          
          return {
            id,
            release: () => {
              acquiredResources.delete(id)
            }
          }
        }).pipe(
          Effect.acquireRelease((resource) => Effect.sync(() => resource.release()))
        )
        
        const pool = yield* Pool.make({
          acquire: trackingAcquire,
          size: poolSize,
          concurrency: 1
        })
        
        const operations = Array.from({ length: operationCount }, () =>
          pool.get.pipe(
            Effect.scoped,
            Effect.delay(Duration.millis(Math.random() * 10))
          )
        )
        
        yield* Effect.forEach(operations, (op) => op, {
          concurrency: operationCount
        })
        
        // Pool should never exceed specified size
        expect(maxConcurrentResources).toBeLessThanOrEqual(poolSize)
      }).pipe(
        Effect.scoped,
        Effect.provide(TestServices.layer),
        Effect.runPromise
      )
    ))
  )
})
```

## Conclusion

Pool provides efficient, controlled resource management for scalable applications. It handles the complexity of resource lifecycle, concurrency control, and health monitoring while providing a simple, composable API.

Key benefits:
- **Resource Efficiency**: Reuse expensive resources instead of recreating them
- **Automatic Lifecycle**: Handles acquisition, release, and cleanup automatically
- **Concurrency Control**: Manages concurrent access with configurable permits
- **Health Monitoring**: Automatic invalidation and replacement of failed resources
- **Dynamic Scaling**: TTL strategies and utilization controls for adaptive sizing

Pool is essential when working with expensive resources like database connections, HTTP clients, file handles, or worker threads. It transforms manual resource management into declarative, composable patterns that scale with your application's needs.