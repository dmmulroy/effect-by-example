# Runtime: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem Runtime Solves

Running production applications involves complex orchestration of initialization, error handling, graceful shutdown, and resource cleanup. Traditional Node.js applications often struggle with:

```typescript
// Traditional approach - problematic patterns
process.on('uncaughtException', (error) => {
  console.error('Uncaught Exception:', error)
  process.exit(1) // Abrupt exit, no cleanup
})

process.on('SIGTERM', () => {
  console.log('SIGTERM received')
  // Manual cleanup needed
  database.close()
  server.close()
  process.exit(0)
})

async function main() {
  try {
    await initializeDatabase()
    await startServer()
    await runScheduledTasks()
  } catch (error) {
    console.error('Application failed:', error)
    process.exit(1) // Resources may leak
  }
}

main()
```

This approach leads to:
- **Resource Leaks** - No guaranteed cleanup on failure
- **Inconsistent Error Handling** - Different error paths, different behaviors
- **Complex Signal Management** - Manual wiring of shutdown procedures
- **Poor Observability** - Scattered logging and error reporting

### The Runtime Solution

Runtime provides a unified application lifecycle manager that handles initialization, error reporting, signal management, and graceful shutdown automatically:

```typescript
import { NodeRuntime } from "@effect/platform-node"
import { Effect } from "effect"

const application = Effect.gen(function* () {
  const database = yield* Database
  const server = yield* HttpServer
  const scheduler = yield* TaskScheduler
  
  yield* Effect.log("Application started successfully")
  yield* Effect.never // Keep running until interrupted
})

NodeRuntime.runMain(application)
```

### Key Concepts

**runMain**: The primary entry point that orchestrates your entire application lifecycle with built-in error handling, logging, and signal management.

**Teardown**: Custom cleanup logic that runs when your application terminates, ensuring proper resource disposal regardless of exit reason.

**Signal Handling**: Automatic handling of SIGTERM, SIGINT, and other process signals with proper Effect interruption semantics.

## Basic Usage Patterns

### Pattern 1: Simple Application Bootstrap

```typescript
import { NodeRuntime } from "@effect/platform-node"
import { Effect } from "effect"

// Simplest possible application
const simpleApp = Effect.gen(function* () {
  yield* Effect.log("Hello, Production!")
  return "success"
})

// Runtime handles everything: logging, exit codes, error reporting
NodeRuntime.runMain(simpleApp)
```

### Pattern 2: Application with Configuration

```typescript
import { NodeRuntime } from "@effect/platform-node"
import { Effect, Config, Layer } from "effect"

const AppConfig = Config.all({
  port: Config.withDefault(Config.integer("PORT"), 3000),
  logLevel: Config.withDefault(Config.string("LOG_LEVEL"), "info"),
  dbUrl: Config.string("DATABASE_URL")
})

const configuredApp = Effect.gen(function* () {
  const config = yield* AppConfig
  yield* Effect.logInfo("Starting app", { config })
  
  // Your application logic here
  yield* Effect.sleep("1 second")
  return "configured app started"
})

NodeRuntime.runMain(
  Effect.provide(configuredApp, Layer.setConfigProvider(Config.fromEnv()))
)
```

### Pattern 3: Error Handling and Custom Teardown

```typescript
import { NodeRuntime } from "@effect/platform-node"
import { Effect, Exit } from "effect"

const flakyApp = Effect.gen(function* () {
  yield* Effect.sleep("2 seconds")
  yield* Effect.fail("Something went wrong!")
})

NodeRuntime.runMain(flakyApp, {
  teardown: (exit, onExit) => {
    if (Exit.isFailure(exit)) {
      console.log("Cleaning up after failure...")
      // Custom cleanup logic here
      onExit(1)
    } else {
      console.log("Clean shutdown")
      onExit(0)
    }
  }
})
```

## Real-World Examples

### Example 1: Web Server with Graceful Shutdown

This example demonstrates a complete web server with database connections, background tasks, and proper shutdown handling:

```typescript
import { NodeRuntime } from "@effect/platform-node"
import { HttpServer, HttpRouter, HttpServerResponse } from "@effect/platform"
import { Effect, Layer, Schedule, Ref, Exit } from "effect"

// Simulated services
class Database extends Effect.Service<Database>()("Database", {
  effect: Effect.gen(function* () {
    const connectionCount = yield* Ref.make(0)
    
    return {
      connect: Effect.gen(function* () {
        yield* Ref.update(connectionCount, n => n + 1)
        yield* Effect.logInfo("Database connected")
      }),
      
      disconnect: Effect.gen(function* () {
        const count = yield* Ref.get(connectionCount)
        if (count > 0) {
          yield* Ref.set(connectionCount, 0)
          yield* Effect.logInfo("Database disconnected")
        }
      }),
      
      healthCheck: Effect.gen(function* () {
        const count = yield* Ref.get(connectionCount)
        return count > 0 ? "healthy" : "unhealthy"
      })
    }
  })
}) {}

class BackgroundTasks extends Effect.Service<BackgroundTasks>()("BackgroundTasks", {
  effect: Effect.gen(function* () {
    const isRunning = yield* Ref.make(false)
    
    const startTasks = Effect.gen(function* () {
      yield* Ref.set(isRunning, true)
      yield* Effect.logInfo("Background tasks started")
      
      // Simulate periodic tasks
      const task = Effect.gen(function* () {
        const running = yield* Ref.get(isRunning)
        if (running) {
          yield* Effect.logDebug("Running background task")
          yield* Effect.sleep("30 seconds")
        }
      }).pipe(
        Effect.repeat(Schedule.whileOutput(running => running))
      )
      
      return yield* Effect.fork(task)
    })
    
    const stopTasks = Effect.gen(function* () {
      yield* Ref.set(isRunning, false)
      yield* Effect.logInfo("Background tasks stopped")
    })
    
    return { startTasks, stopTasks }
  })
}) {}

// Health check endpoint
const healthRouter = HttpRouter.empty.pipe(
  HttpRouter.get("/health", Effect.gen(function* () {
    const db = yield* Database
    const dbStatus = yield* db.healthCheck
    
    const health = {
      status: dbStatus === "healthy" ? "ok" : "degraded",
      database: dbStatus,
      timestamp: new Date().toISOString()
    }
    
    return HttpServerResponse.json(health)
  }))
)

// Main application router
const appRouter = HttpRouter.empty.pipe(
  HttpRouter.get("/", Effect.succeed(
    HttpServerResponse.text("Server is running!")
  )),
  HttpRouter.mount("/api", healthRouter)
)

// Application layer with all dependencies
const AppLayer = Layer.mergeAll(
  Database.Default,
  BackgroundTasks.Default
)

// Main server application
const serverApp = Effect.gen(function* () {
  // Initialize database
  const db = yield* Database
  const tasks = yield* BackgroundTasks
  
  yield* db.connect
  const tasksFiber = yield* tasks.startTasks
  
  // Start HTTP server
  const server = yield* HttpServer.serve(appRouter).pipe(
    HttpServer.withLogAddress,
    Effect.provide(HttpServer.layer({ port: 3000 }))
  )
  
  yield* Effect.logInfo("Server started on port 3000")
  
  // Ensure cleanup happens on shutdown
  yield* Effect.addFinalizer(() => Effect.gen(function* () {
    yield* Effect.logInfo("Shutting down gracefully...")
    yield* tasks.stopTasks
    yield* Effect.Fiber.interrupt(tasksFiber)
    yield* db.disconnect
    yield* Effect.logInfo("Shutdown complete")
  }))
  
  // Keep server running until interrupted
  yield* Effect.never
})

// Custom teardown with detailed logging
const customTeardown = (exit: Exit.Exit<unknown, unknown>, onExit: (code: number) => void) => {
  if (Exit.isFailure(exit)) {
    console.error("Server failed:", Exit.unannotate(exit.cause))
    onExit(1)
  } else if (Exit.isInterrupt(exit)) {
    console.log("Server interrupted - shutdown complete")
    onExit(0)
  } else {
    console.log("Server finished successfully")
    onExit(0)
  }
}

// Run the server with proper lifecycle management
NodeRuntime.runMain(
  serverApp.pipe(Effect.provide(AppLayer)),
  { teardown: customTeardown }
)
```

### Example 2: Data Processing Pipeline with Error Recovery

This example shows a robust data processing application with retry logic, error recovery, and comprehensive monitoring:

```typescript
import { NodeRuntime } from "@effect/platform-node"
import { Effect, Schedule, Ref, Cause, Exit, Queue, Fiber } from "effect"

// Error types for different failure scenarios
class ProcessingError extends Error {
  readonly _tag = "ProcessingError"
  constructor(readonly item: string, readonly cause: unknown) {
    super(`Failed to process item: ${item}`)
  }
}

class DataSourceError extends Error {
  readonly _tag = "DataSourceError"
  constructor(readonly message: string) {
    super(message)
  }
}

// Metrics tracking service
class Metrics extends Effect.Service<Metrics>()("Metrics", {
  effect: Effect.gen(function* () {
    const processed = yield* Ref.make(0)
    const failed = yield* Ref.make(0)
    const retried = yield* Ref.make(0)
    
    return {
      incrementProcessed: Ref.update(processed, n => n + 1),
      incrementFailed: Ref.update(failed, n => n + 1),
      incrementRetried: Ref.update(retried, n => n + 1),
      
      getStats: Effect.gen(function* () {
        const p = yield* Ref.get(processed)
        const f = yield* Ref.get(failed)
        const r = yield* Ref.get(retried)
        return { processed: p, failed: f, retried: r }
      }),
      
      logStats: Effect.gen(function* () {
        const stats = yield* Ref.get(processed).pipe(
          Effect.zip(Ref.get(failed)),
          Effect.zip(Ref.get(retried))
        )
        const [[[p, f], r]] = stats
        yield* Effect.logInfo("Processing stats", { 
          processed: p, 
          failed: f, 
          retried: r,
          successRate: p > 0 ? ((p / (p + f)) * 100).toFixed(2) + "%" : "0%"
        })
      })
    }
  })
}) {}

// Data source simulation
const fetchData = Effect.gen(function* () {
  // Simulate network call that might fail
  const shouldFail = Math.random() < 0.1
  if (shouldFail) {
    yield* Effect.fail(new DataSourceError("Network timeout"))
  }
  
  // Return batch of items to process
  return Array.from({ length: 10 }, (_, i) => `item-${Date.now()}-${i}`)
})

// Individual item processing with potential failures
const processItem = (item: string) => Effect.gen(function* () {
  // Simulate processing work
  yield* Effect.sleep("100 millis")
  
  // Random failure chance
  const shouldFail = Math.random() < 0.15
  if (shouldFail) {
    yield* Effect.fail(new ProcessingError(item, "Random processing failure"))
  }
  
  return `processed-${item}`
})

// Resilient processing with retries and error handling
const processWithRetry = (item: string) => {
  return processItem(item).pipe(
    Effect.retry(Schedule.exponential("100 millis").pipe(
      Schedule.intersect(Schedule.recurs(3))
    )),
    Effect.tap(() => Effect.gen(function* () {
      const metrics = yield* Metrics
      yield* metrics.incrementProcessed
    })),
    Effect.tapError(() => Effect.gen(function* () {
      const metrics = yield* Metrics
      yield* metrics.incrementFailed
      yield* Effect.logError(`Failed to process item after retries: ${item}`)
    })),
    Effect.catchAll(error => Effect.gen(function* () {
      // Log error but don't fail the entire pipeline
      yield* Effect.logWarning("Item processing failed", { item, error: error.message })
      return `failed-${item}`
    }))
  )
}

// Main data processing pipeline
const dataProcessingPipeline = Effect.gen(function* () {
  const metrics = yield* Metrics
  const processedQueue = yield* Queue.unbounded<string>()
  
  // Stats reporting fiber
  const statsFiber = yield* Effect.gen(function* () {
    yield* metrics.logStats
    yield* Effect.sleep("10 seconds")
  }).pipe(
    Effect.repeat(Schedule.forever),
    Effect.fork
  )
  
  yield* Effect.logInfo("Starting data processing pipeline")
  
  // Main processing loop
  yield* Effect.gen(function* () {
    // Fetch batch of data
    const items = yield* fetchData.pipe(
      Effect.retry(Schedule.exponential("1 second").pipe(
        Schedule.intersect(Schedule.recurs(5))
      )),
      Effect.tapError(error => Effect.logError("Data source failed", { error: error.message }))
    )
    
    yield* Effect.logInfo("Processing batch", { itemCount: items.length })
    
    // Process items concurrently with controlled parallelism
    const results = yield* Effect.forEach(
      items,
      processWithRetry,
      { concurrency: 5 }
    )
    
    // Queue processed results
    yield* Effect.forEach(results, result => Queue.offer(processedQueue, result))
    
    yield* Effect.sleep("5 seconds")
  }).pipe(
    Effect.repeat(Schedule.forever),
    Effect.race(Effect.gen(function* () {
      // Consumer: process queued results
      yield* Effect.gen(function* () {
        const result = yield* Queue.take(processedQueue)
        yield* Effect.logDebug("Consumed result", { result })
      }).pipe(Effect.repeat(Schedule.forever))
    }))
  )
}).pipe(
  Effect.provide(Metrics.Default)
)

// Enhanced teardown with final metrics
const processingTeardown = (exit: Exit.Exit<unknown, unknown>, onExit: (code: number) => void) => {
  console.log("\n=== Processing Pipeline Shutdown ===")
  
  if (Exit.isFailure(exit)) {
    const cause = Exit.unannotate(exit.cause)
    console.error("Pipeline failed:")
    console.error(Cause.pretty(cause))
    onExit(1)
  } else if (Exit.isInterrupt(exit)) {
    console.log("Pipeline interrupted gracefully")
    onExit(0)
  } else {
    console.log("Pipeline completed successfully")
    onExit(0)
  }
}

// Run the data processing pipeline
NodeRuntime.runMain(dataProcessingPipeline, {
  teardown: processingTeardown,
  disablePrettyLogger: false // Keep pretty logging for development
})
```

### Example 3: Microservice with Health Checks and Observability

This example demonstrates a production-ready microservice with comprehensive health checks, metrics, and monitoring:

```typescript
import { NodeRuntime } from "@effect/platform-node"
import { HttpServer, HttpRouter, HttpServerResponse, HttpServerRequest } from "@effect/platform"
import { Effect, Layer, Ref, Schedule, Exit, Clock, Cause } from "effect"

// Health status types
type HealthStatus = "healthy" | "degraded" | "unhealthy"

interface HealthCheck {
  readonly name: string
  readonly status: HealthStatus
  readonly lastChecked: Date
  readonly details?: Record<string, unknown>
}

// Application metrics service
class AppMetrics extends Effect.Service<AppMetrics>()("AppMetrics", {
  effect: Effect.gen(function* () {
    const requestCount = yield* Ref.make(0)
    const errorCount = yield* Ref.make(0)
    const uptime = yield* Clock.currentTimeMillis
    
    return {
      recordRequest: Ref.update(requestCount, n => n + 1),
      recordError: Ref.update(errorCount, n => n + 1),
      
      getMetrics: Effect.gen(function* () {
        const requests = yield* Ref.get(requestCount)
        const errors = yield* Ref.get(errorCount)
        const now = yield* Clock.currentTimeMillis
        
        return {
          requests,
          errors,
          uptime: now - uptime,
          errorRate: requests > 0 ? (errors / requests) * 100 : 0
        }
      })
    }
  })
}) {}

// External dependency health checker
class HealthChecker extends Effect.Service<HealthChecker>()("HealthChecker", {
  effect: Effect.gen(function* () {
    const lastChecks = yield* Ref.make<Map<string, HealthCheck>>(new Map())
    
    const checkDatabase = Effect.gen(function* () {
      // Simulate database health check
      yield* Effect.sleep("50 millis")
      const isHealthy = Math.random() > 0.05 // 95% healthy
      
      return {
        name: "database",
        status: isHealthy ? "healthy" as const : "degraded" as const,
        lastChecked: new Date(),
        details: {
          connectionPool: isHealthy ? "active" : "limited",
          responseTime: "45ms"
        }
      }
    })
    
    const checkExternalAPI = Effect.gen(function* () {
      // Simulate external API health check
      yield* Effect.sleep("100 millis")
      const isHealthy = Math.random() > 0.1 // 90% healthy
      
      return {
        name: "external-api",
        status: isHealthy ? "healthy" as const : "unhealthy" as const,
        lastChecked: new Date(),
        details: {
          endpoint: "https://api.example.com",
          status: isHealthy ? "responding" : "timeout"
        }
      }
    })
    
    const runHealthChecks = Effect.gen(function* () {
      const [dbHealth, apiHealth] = yield* Effect.all([
        checkDatabase,
        checkExternalAPI
      ], { concurrency: 2 })
      
      const checks = new Map([
        [dbHealth.name, dbHealth],
        [apiHealth.name, apiHealth]
      ])
      
      yield* Ref.set(lastChecks, checks)
      return checks
    })
    
    const getHealthStatus = Effect.gen(function* () {
      const checks = yield* Ref.get(lastChecks)
      const healthChecks = Array.from(checks.values())
      
      const overallStatus: HealthStatus = healthChecks.some(c => c.status === "unhealthy")
        ? "unhealthy"
        : healthChecks.some(c => c.status === "degraded")
        ? "degraded"
        : "healthy"
      
      return {
        status: overallStatus,
        checks: healthChecks,
        timestamp: new Date().toISOString()
      }
    })
    
    return { runHealthChecks, getHealthStatus }
  })
}) {}

// Main application routes
const createAppRouter = Effect.gen(function* () {
  const metrics = yield* AppMetrics
  const health = yield* HealthChecker
  
  // Middleware to track requests
  const trackingMiddleware = HttpRouter.use((app) =>
    Effect.gen(function* () {
      yield* metrics.recordRequest
      return yield* app
    }).pipe(
      Effect.tapError(() => metrics.recordError)
    )
  )
  
  const mainRouter = HttpRouter.empty.pipe(
    HttpRouter.get("/", Effect.succeed(
      HttpServerResponse.json({ 
        message: "Microservice is running",
        version: "1.0.0" 
      })
    )),
    
    HttpRouter.get("/health", Effect.gen(function* () {
      const healthStatus = yield* health.getHealthStatus
      const statusCode = healthStatus.status === "healthy" ? 200 : 503
      
      return HttpServerResponse.json(healthStatus, { status: statusCode })
    })),
    
    HttpRouter.get("/metrics", Effect.gen(function* () {
      const appMetrics = yield* metrics.getMetrics
      return HttpServerResponse.json({
        ...appMetrics,
        timestamp: new Date().toISOString()
      })
    })),
    
    HttpRouter.get("/ready", Effect.gen(function* () {
      // Readiness check - simpler than health check
      const healthStatus = yield* health.getHealthStatus
      const isReady = healthStatus.status !== "unhealthy"
      
      return HttpServerResponse.json(
        { ready: isReady },
        { status: isReady ? 200 : 503 }
      )
    })),
    
    // Simulate business endpoints
    HttpRouter.get("/api/users", Effect.gen(function* () {
      // Simulate occasional errors
      const shouldFail = Math.random() < 0.05
      if (shouldFail) {
        yield* Effect.fail(new Error("Database connection failed"))
      }
      
      return HttpServerResponse.json({
        users: [
          { id: 1, name: "Alice" },
          { id: 2, name: "Bob" }
        ]
      })
    }))
  )
  
  return mainRouter.pipe(trackingMiddleware)
})

// Background health monitoring
const healthMonitoring = Effect.gen(function* () {
  const health = yield* HealthChecker
  
  yield* Effect.gen(function* () {
    yield* health.runHealthChecks
    yield* Effect.logDebug("Health checks completed")
  }).pipe(
    Effect.repeat(Schedule.fixed("30 seconds")),
    Effect.catchAll(error => Effect.logError("Health check failed", { error: error.message }))
  )
})

// Application layer
const AppLayer = Layer.mergeAll(
  AppMetrics.Default,
  HealthChecker.Default
)

// Main microservice application
const microserviceApp = Effect.gen(function* () {
  yield* Effect.logInfo("Starting microservice...")
  
  // Initialize health checks
  const health = yield* HealthChecker
  yield* health.runHealthChecks
  
  // Start background monitoring
  const monitoringFiber = yield* Effect.fork(healthMonitoring)
  
  // Create and start HTTP server
  const router = yield* createAppRouter
  const server = yield* HttpServer.serve(router).pipe(
    HttpServer.withLogAddress,
    Effect.provide(HttpServer.layer({ port: 8080 }))
  )
  
  yield* Effect.logInfo("Microservice ready", { port: 8080 })
  
  // Cleanup on shutdown
  yield* Effect.addFinalizer(() => Effect.gen(function* () {
    yield* Effect.logInfo("Shutting down microservice...")
    yield* Effect.Fiber.interrupt(monitoringFiber)
    yield* Effect.logInfo("Microservice shutdown complete")
  }))
  
  yield* Effect.never
})

// Production teardown with detailed logging
const microserviceTeardown = (exit: Exit.Exit<unknown, unknown>, onExit: (code: number) => void) => {
  const timestamp = new Date().toISOString()
  
  if (Exit.isFailure(exit)) {
    console.error(`[${timestamp}] Microservice failed:`)
    console.error(Cause.pretty(Exit.unannotate(exit.cause)))
    onExit(1)
  } else if (Exit.isInterrupt(exit)) {
    console.log(`[${timestamp}] Microservice interrupted - graceful shutdown`)
    onExit(0)
  } else {
    console.log(`[${timestamp}] Microservice completed successfully`)
    onExit(0)
  }
}

// Run the microservice
NodeRuntime.runMain(
  microserviceApp.pipe(Effect.provide(AppLayer)),
  { 
    teardown: microserviceTeardown,
    disablePrettyLogger: process.env.NODE_ENV === "production"
  }
)
```

## Advanced Features Deep Dive

### Feature 1: Custom Teardown Logic

Teardown functions provide complete control over application shutdown behavior, allowing custom cleanup logic and exit code management.

#### Basic Teardown Usage

```typescript
import { NodeRuntime } from "@effect/platform-node"
import { Effect, Exit } from "effect"

const basicTeardown = (exit: Exit.Exit<unknown, unknown>, onExit: (code: number) => void) => {
  console.log("Application is shutting down...")
  
  if (Exit.isFailure(exit)) {
    console.error("Application failed")
    onExit(1)
  } else {
    console.log("Application completed successfully")
    onExit(0)
  }
}

const app = Effect.succeed("Hello World")
NodeRuntime.runMain(app, { teardown: basicTeardown })
```

#### Real-World Teardown Example

```typescript
import { NodeRuntime } from "@effect/platform-node"
import { Effect, Exit, Cause } from "effect"

interface AppResources {
  connections: number
  activeRequests: number
  tempFiles: string[]
}

const createResourceTracker = Effect.gen(function* () {
  const resources: AppResources = {
    connections: 0,
    activeRequests: 0,
    tempFiles: []
  }
  
  return {
    addConnection: () => resources.connections++,
    removeConnection: () => resources.connections--,
    addRequest: () => resources.activeRequests++,
    removeRequest: () => resources.activeRequests--,
    addTempFile: (file: string) => resources.tempFiles.push(file),
    getResources: () => ({ ...resources })
  }
})

const productionTeardown = (
  resourceTracker: ReturnType<typeof createResourceTracker>
) => (exit: Exit.Exit<unknown, unknown>, onExit: (code: number) => void) => {
  const resources = resourceTracker.getResources()
  
  console.log("=== Application Shutdown Report ===")
  console.log(`Timestamp: ${new Date().toISOString()}`)
  console.log(`Open Connections: ${resources.connections}`)
  console.log(`Active Requests: ${resources.activeRequests}`)
  console.log(`Temp Files: ${resources.tempFiles.length}`)
  
  if (Exit.isFailure(exit)) {
    const cause = Exit.unannotate(exit.cause)
    console.error("Exit Reason: FAILURE")
    console.error("Error Details:")
    console.error(Cause.pretty(cause))
    
    // Cleanup temp files
    resources.tempFiles.forEach(file => {
      try {
        console.log(`Cleaning up temp file: ${file}`)
        // fs.unlinkSync(file) in real application
      } catch (error) {
        console.error(`Failed to cleanup ${file}:`, error)
      }
    })
    
    onExit(1)
  } else if (Exit.isInterrupt(exit)) {
    console.log("Exit Reason: INTERRUPTED (Signal received)")
    console.log("Graceful shutdown completed")
    onExit(0)
  } else {
    console.log("Exit Reason: SUCCESS")
    onExit(0)
  }
  
  console.log("=== Shutdown Complete ===")
}
```

#### Advanced Teardown: Async Cleanup

```typescript
import { NodeRuntime } from "@effect/platform-node"
import { Effect, Exit } from "effect"

// For complex async cleanup scenarios
const asyncTeardown = (exit: Exit.Exit<unknown, unknown>, onExit: (code: number) => void) => {
  const cleanup = async () => {
    try {
      console.log("Starting async cleanup...")
      
      // Simulate async cleanup operations
      await new Promise(resolve => setTimeout(resolve, 1000))
      console.log("Database connections closed")
      
      await new Promise(resolve => setTimeout(resolve, 500))
      console.log("Cache cleared")
      
      await new Promise(resolve => setTimeout(resolve, 200))
      console.log("Temp files removed")
      
      console.log("Async cleanup completed")
      
      if (Exit.isFailure(exit)) {
        onExit(1)
      } else {
        onExit(0)
      }
    } catch (error) {
      console.error("Cleanup failed:", error)
      onExit(1)
    }
  }
  
  cleanup()
}
```

### Feature 2: Runtime Configuration Options

Runtime provides several configuration options to customize application behavior for different environments.

#### Development vs Production Configuration

```typescript
import { NodeRuntime } from "@effect/platform-node"
import { Effect } from "effect"

const app = Effect.gen(function* () {
  yield* Effect.logInfo("Application starting")
  yield* Effect.sleep("1 second")
  return "completed"
})

// Development configuration
const devConfig = {
  disableErrorReporting: false,    // Show all errors
  disablePrettyLogger: false,      // Use pretty formatting
  teardown: (exit, onExit) => {
    console.log("Dev teardown - quick restart")
    onExit(Exit.isFailure(exit) ? 1 : 0)
  }
}

// Production configuration
const prodConfig = {
  disableErrorReporting: false,    // Log errors for monitoring
  disablePrettyLogger: true,       // Use structured JSON logging
  teardown: (exit, onExit) => {
    // Comprehensive production teardown
    const timestamp = new Date().toISOString()
    if (Exit.isFailure(exit)) {
      console.error(`[${timestamp}] PROD-ERROR: Application failed`)
      // Could send to monitoring service here
      onExit(1)
    } else {
      console.log(`[${timestamp}] PROD-INFO: Clean shutdown`)
      onExit(0)
    }
  }
}

// Environment-based configuration
const config = process.env.NODE_ENV === "production" ? prodConfig : devConfig
NodeRuntime.runMain(app, config)
```

#### Error Reporting Configuration

```typescript
import { NodeRuntime } from "@effect/platform-node"
import { Effect } from "effect"

const flakyApp = Effect.gen(function* () {
  const shouldFail = Math.random() < 0.3
  if (shouldFail) {
    yield* Effect.fail(new Error("Random failure for testing"))
  }
  return "success"
})

// Silent error handling (useful for testing)
NodeRuntime.runMain(flakyApp, {
  disableErrorReporting: true,
  teardown: (exit, onExit) => {
    // Custom error handling without default logging
    if (Exit.isFailure(exit)) {
      console.log("Handled failure silently")
    }
    onExit(0) // Always exit successfully
  }
})
```

### Feature 3: Signal Handling and Interruption

Runtime automatically handles process signals and provides clean interruption semantics.

#### Signal Handling Behavior

```typescript
import { NodeRuntime } from "@effect/platform-node"
import { Effect, Ref } from "effect"

const longRunningApp = Effect.gen(function* () {
  const counter = yield* Ref.make(0)
  
  yield* Effect.logInfo("Long-running application started")
  yield* Effect.logInfo("Press Ctrl+C to trigger graceful shutdown")
  
  // Simulate work that responds to interruption
  yield* Effect.gen(function* () {
    const count = yield* Ref.updateAndGet(counter, n => n + 1)
    yield* Effect.logInfo(`Iteration: ${count}`)
    yield* Effect.sleep("2 seconds")
  }).pipe(
    Effect.repeat(Schedule.forever),
    Effect.interruptible // This ensures clean interruption
  )
})

NodeRuntime.runMain(longRunningApp, {
  teardown: (exit, onExit) => {
    if (Exit.isInterrupt(exit)) {
      console.log("Application was interrupted by signal")
      console.log("All resources cleaned up properly")
    }
    onExit(0)
  }
})
```

## Practical Patterns & Best Practices

### Pattern 1: Application Factory with Environment Configuration

```typescript
import { NodeRuntime } from "@effect/platform-node"
import { Effect, Config, Layer, LogLevel } from "effect"

// Application configuration schema
const AppConfig = Config.all({
  environment: Config.withDefault(Config.string("NODE_ENV"), "development"),
  port: Config.withDefault(Config.integer("PORT"), 3000),
  logLevel: Config.withDefault(Config.string("LOG_LEVEL"), "info"),
  shutdownTimeout: Config.withDefault(Config.integer("SHUTDOWN_TIMEOUT_MS"), 30000)
})

// Application factory
const createApplication = (config: Config.Config.Success<typeof AppConfig>) => {
  const isDevelopment = config.environment === "development"
  const isProduction = config.environment === "production"
  
  return Effect.gen(function* () {
    yield* Effect.logInfo("Application starting", { 
      environment: config.environment,
      port: config.port 
    })
    
    // Environment-specific initialization
    if (isDevelopment) {
      yield* Effect.logDebug("Development mode: enabling hot reload")
    }
    
    if (isProduction) {
      yield* Effect.logInfo("Production mode: optimized settings active")
    }
    
    // Your application logic here
    yield* Effect.never
  })
}

// Environment-specific runtime configuration
const createRuntimeConfig = (config: Config.Config.Success<typeof AppConfig>) => {
  const isProduction = config.environment === "production"
  
  return {
    disablePrettyLogger: isProduction,
    teardown: (exit, onExit) => {
      const timestamp = new Date().toISOString()
      const env = config.environment.toUpperCase()
      
      if (Exit.isFailure(exit)) {
        console.error(`[${timestamp}] ${env}-ERROR: Application failed`)
        onExit(1)
      } else if (Exit.isInterrupt(exit)) {
        console.log(`[${timestamp}] ${env}-INFO: Graceful shutdown`)
        onExit(0)
      } else {
        console.log(`[${timestamp}] ${env}-INFO: Clean exit`)
        onExit(0)
      }
    }
  }
}

// Main application bootstrap
const main = Effect.gen(function* () {
  const config = yield* AppConfig
  const app = createApplication(config)
  const runtimeConfig = createRuntimeConfig(config)
  
  return { app, runtimeConfig }
}).pipe(
  Effect.provide(Layer.setConfigProvider(Config.fromEnv()))
)

// Run with configuration
Effect.runPromise(main).then(({ app, runtimeConfig }) => {
  NodeRuntime.runMain(app, runtimeConfig)
})
```

### Pattern 2: Resource Pool Management

```typescript
import { NodeRuntime } from "@effect/platform-node"
import { Effect, Pool, Schedule, Ref } from "effect"

// Resource interface
interface DatabaseConnection {
  readonly id: string
  readonly isHealthy: boolean
  query: (sql: string) => Effect.Effect<unknown[], Error>
  close: () => Effect.Effect<void>
}

// Resource pool service
class ConnectionPool extends Effect.Service<ConnectionPool>()("ConnectionPool", {
  effect: Effect.gen(function* () {
    // Create connection factory
    const createConnection = Effect.gen(function* () {
      const id = `conn-${Math.random().toString(36).substr(2, 9)}`
      yield* Effect.logDebug("Creating database connection", { id })
      
      return {
        id,
        isHealthy: true,
        query: (sql: string) => Effect.gen(function* () {
          yield* Effect.sleep("10 millis") // Simulate query
          return [{ result: `Result for: ${sql}` }]
        }),
        close: () => Effect.gen(function* () {
          yield* Effect.logDebug("Closing database connection", { id })
        })
      } satisfies DatabaseConnection
    })
    
    // Pool configuration
    const pool = yield* Pool.make({
      acquire: createConnection,
      size: 10
    })
    
    const healthyConnections = yield* Ref.make(0)
    
    // Health monitoring
    const monitorHealth = Effect.gen(function* () {
      const stats = yield* Pool.get(pool)
      yield* Ref.set(healthyConnections, stats.size)
      yield* Effect.logInfo("Connection pool health", {
        size: stats.size,
        invalidated: stats.invalidated
      })
    }).pipe(
      Effect.repeat(Schedule.fixed("30 seconds"))
    )
    
    const getConnection = Pool.get(pool)
    const invalidateConnection = (conn: DatabaseConnection) => 
      Pool.invalidate(pool, conn)
    
    const shutdown = Effect.gen(function* () {
      yield* Effect.logInfo("Shutting down connection pool")
      yield* Pool.shutdown(pool)
      yield* Effect.logInfo("Connection pool shutdown complete")
    })
    
    return {
      getConnection,
      invalidateConnection,
      monitorHealth,
      shutdown,
      getHealthyCount: Ref.get(healthyConnections)
    }
  })
}) {}

// Application using connection pool
const pooledApp = Effect.gen(function* () {
  const pool = yield* ConnectionPool
  
  // Start health monitoring
  const monitorFiber = yield* Effect.fork(pool.monitorHealth)
  
  // Simulate application work
  yield* Effect.gen(function* () {
    const connection = yield* pool.getConnection
    yield* Effect.logInfo("Got connection", { id: connection.id })
    
    // Use connection
    const results = yield* connection.query("SELECT * FROM users")
    yield* Effect.logDebug("Query results", { count: results.length })
    
    // Connection automatically returned to pool
  }).pipe(
    Effect.repeat(Schedule.spaced("1 second")),
    Effect.race(Effect.interrupt) // Allow interruption
  )
}).pipe(
  Effect.provide(ConnectionPool.Default),
  Effect.ensuring(Effect.gen(function* () {
    // Cleanup on exit
    const pool = yield* ConnectionPool
    yield* pool.shutdown
  }))
)

NodeRuntime.runMain(pooledApp)
```

### Pattern 3: Multi-Stage Application Lifecycle

```typescript
import { NodeRuntime } from "@effect/platform-node"
import { Effect, Ref, Exit } from "effect"

// Application lifecycle stages
type LifecycleStage = 
  | "initializing"
  | "starting"
  | "ready"
  | "stopping"
  | "stopped"

class ApplicationLifecycle extends Effect.Service<ApplicationLifecycle>()("ApplicationLifecycle", {
  effect: Effect.gen(function* () {
    const stage = yield* Ref.make<LifecycleStage>("initializing")
    const startTime = yield* Effect.sync(() => Date.now())
    
    const setStage = (newStage: LifecycleStage) => Effect.gen(function* () {
      yield* Ref.set(stage, newStage)
      yield* Effect.logInfo("Lifecycle stage changed", { stage: newStage })
    })
    
    const getStage = Ref.get(stage)
    
    const getUptime = Effect.gen(function* () {
      const now = yield* Effect.sync(() => Date.now())
      return now - startTime
    })
    
    const initialize = Effect.gen(function* () {
      yield* setStage("initializing")
      yield* Effect.logInfo("Initializing application components...")
      
      // Simulate initialization tasks
      yield* Effect.sleep("500 millis")
      yield* Effect.logInfo("Configuration loaded")
      
      yield* Effect.sleep("300 millis")
      yield* Effect.logInfo("Database connections established")
      
      yield* Effect.sleep("200 millis")
      yield* Effect.logInfo("Cache warmed up")
      
      yield* setStage("starting")
    })
    
    const start = Effect.gen(function* () {
      yield* setStage("starting")
      yield* Effect.logInfo("Starting application services...")
      
      yield* Effect.sleep("400 millis")
      yield* Effect.logInfo("HTTP server started")
      
      yield* Effect.sleep("200 millis")
      yield* Effect.logInfo("Background workers started")
      
      yield* setStage("ready")
      yield* Effect.logInfo("Application is ready to serve requests")
    })
    
    const stop = Effect.gen(function* () {
      const currentStage = yield* getStage
      if (currentStage === "stopped" || currentStage === "stopping") {
        return
      }
      
      yield* setStage("stopping")
      yield* Effect.logInfo("Stopping application services...")
      
      yield* Effect.sleep("300 millis")
      yield* Effect.logInfo("HTTP server stopped")
      
      yield* Effect.sleep("200 millis")
      yield* Effect.logInfo("Background workers stopped")
      
      yield* Effect.sleep("100 millis")
      yield* Effect.logInfo("Database connections closed")
      
      yield* setStage("stopped")
      const uptime = yield* getUptime
      yield* Effect.logInfo("Application stopped", { uptime })
    })
    
    return { getStage, getUptime, initialize, start, stop }
  })
}) {}

// Multi-stage application
const lifecycleApp = Effect.gen(function* () {
  const lifecycle = yield* ApplicationLifecycle
  
  // Initialize
  yield* lifecycle.initialize
  
  // Start
  yield* lifecycle.start
  
  // Add finalizer for cleanup
  yield* Effect.addFinalizer(() => lifecycle.stop)
  
  // Main application loop
  yield* Effect.gen(function* () {
    const stage = yield* lifecycle.getStage
    const uptime = yield* lifecycle.getUptime
    
    yield* Effect.logDebug("Application heartbeat", { 
      stage, 
      uptime: `${Math.round(uptime / 1000)}s` 
    })
    
    yield* Effect.sleep("10 seconds")
  }).pipe(
    Effect.repeat(Schedule.forever)
  )
})

// Lifecycle-aware teardown
const lifecycleTeardown = (exit: Exit.Exit<unknown, unknown>, onExit: (code: number) => void) => {
  console.log("=== Application Lifecycle Report ===")
  
  if (Exit.isFailure(exit)) {
    console.error("Application failed during lifecycle")
    onExit(1)
  } else if (Exit.isInterrupt(exit)) {
    console.log("Application interrupted - lifecycle cleanup completed")
    onExit(0)
  } else {
    console.log("Application completed full lifecycle")
    onExit(0)
  }
}

NodeRuntime.runMain(
  lifecycleApp.pipe(Effect.provide(ApplicationLifecycle.Default)),
  { teardown: lifecycleTeardown }
)
```

## Integration Examples

### Integration with Express.js Server

```typescript
import { NodeRuntime } from "@effect/platform-node"
import { Effect, Layer, Ref } from "effect"
import express from "express"
import { createServer, Server } from "http"

// Express server service
class ExpressServer extends Effect.Service<ExpressServer>()("ExpressServer", {
  effect: Effect.gen(function* () {
    const app = express()
    const serverRef = yield* Ref.make<Server | null>(null)
    
    // Configure Express
    app.use(express.json())
    
    // Health endpoint
    app.get('/health', (req, res) => {
      res.json({ status: 'healthy', timestamp: new Date().toISOString() })
    })
    
    // Main API endpoint
    app.get('/api/data', (req, res) => {
      res.json({ message: 'Hello from Express + Effect!' })
    })
    
    const start = (port: number) => Effect.gen(function* () {
      const server = createServer(app)
      
      yield* Effect.async<void, Error>((resume) => {
        server.listen(port, (error?: Error) => {
          if (error) {
            resume(Effect.fail(error))
          } else {
            resume(Effect.void)
          }
        })
      })
      
      yield* Ref.set(serverRef, server)
      yield* Effect.logInfo("Express server started", { port })
    })
    
    const stop = Effect.gen(function* () {
      const server = yield* Ref.get(serverRef)
      
      if (server) {
        yield* Effect.async<void>((resume) => {
          server.close(() => {
            resume(Effect.void)
          })
        })
        
        yield* Ref.set(serverRef, null)
        yield* Effect.logInfo("Express server stopped")
      }
    })
    
    return { start, stop, app }
  })
}) {}

// Express + Effect application
const expressApp = Effect.gen(function* () {
  const server = yield* ExpressServer
  
  // Start Express server
  yield* server.start(3000)
  
  // Add shutdown finalizer
  yield* Effect.addFinalizer(() => server.stop)
  
  // Keep running
  yield* Effect.never
})

NodeRuntime.runMain(
  expressApp.pipe(Effect.provide(ExpressServer.Default))
)
```

### Integration with WebSocket Server

```typescript
import { NodeRuntime } from "@effect/platform-node"
import { Effect, Queue, Ref, Schedule } from "effect"
import { WebSocketServer, WebSocket } from "ws"

// WebSocket service
class WSService extends Effect.Service<WSService>()("WSService", {
  effect: Effect.gen(function* () {
    const connections = yield* Ref.make<Set<WebSocket>>(new Set())
    const messageQueue = yield* Queue.unbounded<{ message: string; timestamp: Date }>()
    
    const wss = new WebSocketServer({ port: 8080 })
    
    const addConnection = (ws: WebSocket) => 
      Ref.update(connections, set => new Set([...set, ws]))
    
    const removeConnection = (ws: WebSocket) =>
      Ref.update(connections, set => {
        const newSet = new Set(set)
        newSet.delete(ws)
        return newSet
      })
    
    const broadcast = (message: string) => Effect.gen(function* () {
      const conns = yield* Ref.get(connections)
      const activeConnections = Array.from(conns).filter(ws => 
        ws.readyState === WebSocket.OPEN
      )
      
      yield* Effect.forEach(
        activeConnections,
        (ws) => Effect.sync(() => ws.send(message)),
        { concurrency: "unbounded", discard: true }
      )
      
      yield* Effect.logDebug("Broadcast message", { 
        message, 
        recipients: activeConnections.length 
      })
    })
    
    const setupWebSocket = Effect.gen(function* () {
      yield* Effect.async<void>((resume) => {
        wss.on('connection', (ws) => {
          Effect.runSync(addConnection(ws))
          
          ws.on('message', (data) => {
            const message = data.toString()
            Effect.runSync(
              Queue.offer(messageQueue, { message, timestamp: new Date() })
            )
          })
          
          ws.on('close', () => {
            Effect.runSync(removeConnection(ws))
          })
          
          ws.send(JSON.stringify({ 
            type: 'welcome', 
            message: 'Connected to WebSocket server' 
          }))
        })
        
        resume(Effect.void)
      })
      
      yield* Effect.logInfo("WebSocket server listening", { port: 8080 })
    })
    
    const processMessages = Effect.gen(function* () {
      const { message, timestamp } = yield* Queue.take(messageQueue)
      
      // Process incoming message
      const response = {
        type: 'echo',
        original: message,
        timestamp: timestamp.toISOString(),
        processed: new Date().toISOString()
      }
      
      yield* broadcast(JSON.stringify(response))
    }).pipe(
      Effect.repeat(Schedule.forever)
    )
    
    const heartbeat = Effect.gen(function* () {
      const connectionCount = yield* Ref.get(connections).pipe(
        Effect.map(set => set.size)
      )
      
      const heartbeatMessage = {
        type: 'heartbeat',
        timestamp: new Date().toISOString(),
        connections: connectionCount
      }
      
      yield* broadcast(JSON.stringify(heartbeatMessage))
    }).pipe(
      Effect.repeat(Schedule.fixed("30 seconds"))
    )
    
    const shutdown = Effect.gen(function* () {
      yield* Effect.logInfo("Shutting down WebSocket server")
      
      // Close all connections
      const conns = yield* Ref.get(connections)
      Array.from(conns).forEach(ws => {
        if (ws.readyState === WebSocket.OPEN) {
          ws.close(1000, 'Server shutting down')
        }
      })
      
      // Close server
      yield* Effect.async<void>((resume) => {
        wss.close(() => {
          resume(Effect.void)
        })
      })
      
      yield* Effect.logInfo("WebSocket server shutdown complete")
    })
    
    return { setupWebSocket, processMessages, heartbeat, shutdown }
  })
}) {}

// WebSocket application
const wsApp = Effect.gen(function* () {
  const ws = yield* WSService
  
  // Setup WebSocket server
  yield* ws.setupWebSocket
  
  // Start message processing and heartbeat
  const processingFiber = yield* Effect.fork(ws.processMessages)
  const heartbeatFiber = yield* Effect.fork(ws.heartbeat)
  
  // Cleanup on shutdown
  yield* Effect.addFinalizer(() => Effect.gen(function* () {
    yield* Effect.Fiber.interrupt(processingFiber)
    yield* Effect.Fiber.interrupt(heartbeatFiber)
    yield* ws.shutdown
  }))
  
  yield* Effect.never
})

NodeRuntime.runMain(
  wsApp.pipe(Effect.provide(WSService.Default))
)
```

### Testing Strategies

```typescript
import { NodeRuntime } from "@effect/platform-node"
import { Effect, TestServices, TestClock, Ref } from "effect"
import { describe, it, expect } from "vitest"

// Testable application service
class TestableApp extends Effect.Service<TestableApp>()("TestableApp", {
  effect: Effect.gen(function* () {
    const counter = yield* Ref.make(0)
    const errors = yield* Ref.make<Error[]>([])
    
    const increment = Ref.update(counter, n => n + 1)
    const getCount = Ref.get(counter)
    
    const simulateWork = Effect.gen(function* () {
      yield* increment
      yield* Effect.sleep("1 second")
      
      // Simulate occasional errors
      const shouldFail = Math.random() < 0.1
      if (shouldFail) {
        const error = new Error("Simulated failure")
        yield* Ref.update(errors, errs => [...errs, error])
        yield* Effect.fail(error)
      }
      
      return "work completed"
    })
    
    const getErrors = Ref.get(errors)
    const reset = Effect.all([
      Ref.set(counter, 0),
      Ref.set(errors, [])
    ])
    
    return { increment, getCount, simulateWork, getErrors, reset }
  })
}) {}

// Test utilities
const createTestRuntime = () => {
  const testLayer = TestServices.layer.pipe(
    Layer.provide(TestableApp.Default)
  )
  
  return {
    runTest: <A, E>(effect: Effect.Effect<A, E, TestableApp | TestServices.TestServices>) =>
      Effect.provide(effect, testLayer),
    
    runMainTest: <A, E>(
      effect: Effect.Effect<A, E, TestableApp>,
      config?: Parameters<typeof NodeRuntime.runMain>[1]
    ) => {
      const testConfig = {
        disableErrorReporting: true,
        ...config
      }
      
      return new Promise<{ result: A | null; exit: Exit.Exit<E, A> }>((resolve) => {
        let result: A | null = null
        let capturedExit: Exit.Exit<E, A>
        
        const wrappedEffect = effect.pipe(
          Effect.tap(value => Effect.sync(() => { result = value })),
          Effect.provide(TestableApp.Default)
        )
        
        NodeRuntime.runMain(wrappedEffect, {
          ...testConfig,
          teardown: (exit, onExit) => {
            capturedExit = exit as Exit.Exit<E, A>
            resolve({ result, exit: capturedExit })
            onExit(0)
          }
        })
      })
    }
  }
}

// Test examples
describe("Runtime Testing", () => {
  const testRuntime = createTestRuntime()
  
  it("should handle successful application", async () => {
    const app = Effect.gen(function* () {
      const service = yield* TestableApp
      yield* service.increment
      const count = yield* service.getCount
      return count
    })
    
    const result = await Effect.runPromise(testRuntime.runTest(app))
    expect(result).toBe(1)
  })
  
  it("should handle application with timing", async () => {
    const app = Effect.gen(function* () {
      const service = yield* TestableApp
      yield* service.simulateWork
      return "completed"
    })
    
    const timedTest = testRuntime.runTest(
      app.pipe(Effect.provide(TestClock.default))
    )
    
    const result = await Effect.runPromise(timedTest)
    expect(result).toBe("completed")
  })
  
  it("should test main runtime behavior", async () => {
    const app = Effect.gen(function* () {
      const service = yield* TestableApp
      yield* service.increment
      yield* service.increment
      return yield* service.getCount
    })
    
    const { result, exit } = await testRuntime.runMainTest(app)
    
    expect(Exit.isSuccess(exit)).toBe(true)
    expect(result).toBe(2)
  })
  
  it("should test error handling", async () => {
    const failingApp = Effect.gen(function* () {
      yield* Effect.fail(new Error("Test error"))
    })
    
    const { result, exit } = await testRuntime.runMainTest(failingApp)
    
    expect(Exit.isFailure(exit)).toBe(true)
    expect(result).toBe(null)
  })
})

// Integration test example
const integrationTestApp = Effect.gen(function* () {
  const service = yield* TestableApp
  
  // Test full application lifecycle
  yield* service.reset
  
  // Simulate some work
  yield* service.simulateWork.pipe(
    Effect.retry(Schedule.recurs(3)),
    Effect.catchAll(() => Effect.succeed("handled failure"))
  )
  
  const finalCount = yield* service.getCount
  const errors = yield* service.getErrors
  
  return { finalCount, errorCount: errors.length }
})

// Run integration test
NodeRuntime.runMain(
  integrationTestApp.pipe(Effect.provide(TestableApp.Default)),
  {
    disableErrorReporting: true,
    teardown: (exit, onExit) => {
      if (Exit.isSuccess(exit)) {
        console.log("Integration test passed:", exit.value)
      } else {
        console.error("Integration test failed")
      }
      onExit(0)
    }
  }
)
```

## Conclusion

Runtime provides comprehensive application lifecycle management, error handling, and graceful shutdown capabilities for production Effect applications.

Key benefits:
- **Unified Lifecycle Management**: Single entry point handles initialization, execution, and cleanup
- **Automatic Signal Handling**: Built-in SIGTERM/SIGINT handling with proper Effect interruption
- **Flexible Error Reporting**: Configurable logging and error handling for different environments
- **Resource Safety**: Guaranteed cleanup through finalizers and custom teardown logic
- **Production Ready**: Optimized for both development and production deployment scenarios

Runtime is essential for any Effect application that needs reliable startup, error handling, and shutdown behavior in production environments.