# Supervisor: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem Supervisor Solves

In concurrent applications, managing the lifecycle of parallel operations is complex and error-prone. Traditional approaches to monitoring and controlling concurrent tasks suffer from several critical issues:

```typescript
// Traditional approach - limited visibility and control
class TaskManager {
  private activeTasks = new Map<string, Promise<any>>()
  private taskResults = new Map<string, any>()
  
  async runConcurrentTasks(tasks: Array<() => Promise<any>>): Promise<any[]> {
    const promises = tasks.map(async (task, index) => {
      const taskId = `task-${index}`
      try {
        // No way to monitor task progress or state
        const promise = task()
        this.activeTasks.set(taskId, promise)
        
        const result = await promise
        this.taskResults.set(taskId, result)
        this.activeTasks.delete(taskId)
        return result
      } catch (error) {
        // Can't inspect which tasks are still running
        // Can't gracefully shutdown remaining tasks
        // No visibility into failure propagation
        this.activeTasks.delete(taskId)
        throw error
      }
    })
    
    return Promise.all(promises) // All-or-nothing approach
  }
  
  // Limited monitoring capabilities
  getActiveTaskCount(): number {
    return this.activeTasks.size // Only basic counting
  }
  
  // No way to selectively cancel or restart tasks
  // No hierarchical task management
  // No resource cleanup guarantees
}
```

This approach leads to:
- **No Visibility** - Cannot inspect individual task states, progress, or hierarchy
- **Limited Control** - Cannot selectively cancel, restart, or prioritize specific tasks
- **Resource Leaks** - No guarantees about cleanup when tasks fail or are cancelled
- **Poor Observability** - Cannot track task lifecycle events, performance metrics, or failures
- **Rigid Management** - All-or-nothing failure handling with no fine-grained control

### The Supervisor Solution

Effect's Supervisor module provides comprehensive fiber lifecycle management, enabling tracking, monitoring, and controlling the behavior of concurrent operations with full visibility and precise control.

```typescript
import { Effect, Supervisor, Fiber, Schedule } from "effect"

// Declarative fiber supervision with full lifecycle control
const processOrdersConcurrently = (orders: Order[]) => Effect.gen(function* () {
  // Create a supervisor to track all child fibers
  const supervisor = yield* Supervisor.track
  
  // Process orders under supervision
  const processingFiber = yield* orders.map(processOrder).pipe(
    Effect.all({ concurrency: 5 }),
    Effect.supervised(supervisor), // All child fibers are now tracked
    Effect.withSpan("order-processing"),
    Effect.fork
  )
  
  // Monitor progress in real-time
  const monitorFiber = yield* monitorProcessing(supervisor).pipe(
    Effect.repeat(Schedule.spaced("1 second")),
    Effect.fork
  )
  
  // Wait for completion with full control
  const results = yield* Fiber.join(processingFiber)
  yield* Fiber.interrupt(monitorFiber)
  
  return results
})

// Rich monitoring with detailed fiber information
const monitorProcessing = (supervisor: Supervisor.Supervisor<Fiber.RuntimeFiber<any, any>[]>) =>
  Effect.gen(function* () {
    const fibers = yield* supervisor.value
    const activeCount = fibers.length
    const statuses = yield* Effect.all(fibers.map(Fiber.status))
    
    yield* Effect.log(`Active fibers: ${activeCount}`, {
      running: statuses.filter(s => s._tag === "Running").length,
      suspended: statuses.filter(s => s._tag === "Suspended").length,
      done: statuses.filter(s => s._tag === "Done").length
    })
  })
```

### Key Concepts

**Supervisor**: A utility that manages fiber lifecycles, tracking creation, execution, suspension, resumption, and termination events while producing observable values about the supervised fibers.

**Fiber Tracking**: Supervisors maintain collections of active fibers, providing real-time visibility into concurrent operations and their states.

**Lifecycle Events**: Supervisors can hook into key fiber lifecycle events (`onStart`, `onEnd`, `onSuspend`, `onResume`, `onEffect`) to implement custom monitoring and control logic.

## Basic Usage Patterns

### Pattern 1: Basic Fiber Tracking

```typescript
import { Effect, Supervisor, Fiber } from "effect"

// Create a tracking supervisor and monitor fiber count
const basicTracking = Effect.gen(function* () {
  // Create supervisor that tracks child fibers in an array
  const supervisor = yield* Supervisor.track
  
  // Run some concurrent work under supervision
  const work = Effect.gen(function* () {
    yield* Effect.fork(Effect.delay("100 millis")(Effect.succeed("task 1")))
    yield* Effect.fork(Effect.delay("200 millis")(Effect.succeed("task 2")))
    yield* Effect.fork(Effect.delay("150 millis")(Effect.succeed("task 3")))
    yield* Effect.sleep("50 millis") // Give tasks time to start
  }).pipe(Effect.supervised(supervisor))
  
  yield* work
  
  // Check how many fibers are still active
  const activeFibers = yield* supervisor.value
  yield* Effect.log(`Active fibers: ${activeFibers.length}`)
  
  return activeFibers.length
})
```

### Pattern 2: Custom Supervision Logic

```typescript
import { Effect, Supervisor, MutableRef, SortedSet } from "effect"

// Create supervisor with custom fiber tracking
const customSupervision = Effect.gen(function* () {
  // Create a mutable reference to hold fiber set
  const fiberSet = yield* MutableRef.make(SortedSet.empty<Fiber.RuntimeFiber<any, any>>())
  
  // Create supervisor that uses the custom set
  const supervisor = yield* Supervisor.fibersIn(fiberSet)
  
  // Run supervised work
  const supervisedWork = Effect.gen(function* () {
    yield* Effect.fork(Effect.delay("100 millis")(Effect.succeed("A")))
    yield* Effect.fork(Effect.delay("200 millis")(Effect.succeed("B")))
    return "work complete"
  }).pipe(Effect.supervised(supervisor))
  
  yield* supervisedWork
  
  // Access the custom tracked fibers
  const trackedFibers = yield* supervisor.value
  yield* Effect.log(`Tracked ${SortedSet.size(trackedFibers)} fibers`)
  
  return SortedSet.size(trackedFibers)
})
```

### Pattern 3: No-op Supervision

```typescript
import { Effect, Supervisor } from "effect"

// Sometimes you want supervision without tracking
const noOpSupervision = Effect.gen(function* () {
  // Create a supervisor that doesn't track anything
  const supervisor = Supervisor.none
  
  // Apply supervision without overhead
  const work = Effect.gen(function* () {
    yield* Effect.fork(Effect.succeed("task 1"))
    yield* Effect.fork(Effect.succeed("task 2"))
    return "completed"
  }).pipe(Effect.supervised(supervisor))
  
  const result = yield* work
  
  // supervisor.value is void for none supervisor
  const supervisorValue = yield* supervisor.value
  
  return { result, supervisorValue }
})
```

## Real-World Examples

### Example 1: Web Scraping with Progress Monitoring

```typescript
import { Effect, Supervisor, Schedule, Fiber, Duration } from "effect"

interface ScrapingJob {
  id: string
  url: string
  priority: number
}

interface ScrapingResult {
  jobId: string
  url: string
  content: string
  scrapedAt: Date
}

// Web scraping service with real-time monitoring
const webScrapingService = Effect.gen(function* () {
  const supervisor = yield* Supervisor.track
  
  const jobs: ScrapingJob[] = [
    { id: "1", url: "https://example.com/page1", priority: 1 },
    { id: "2", url: "https://example.com/page2", priority: 2 },
    { id: "3", url: "https://example.com/page3", priority: 1 }
  ]
  
  // Start scraping jobs with supervision
  const scrapingFiber = yield* jobs.map(scrapeUrl).pipe(
    Effect.all({ concurrency: 3 }),
    Effect.supervised(supervisor),
    Effect.withSpan("web-scraping-batch"),
    Effect.fork
  )
  
  // Real-time progress monitoring
  const progressMonitor = yield* Effect.gen(function* () {
    const fibers = yield* supervisor.value
    const totalJobs = jobs.length
    const activeJobs = fibers.length
    const completedJobs = totalJobs - activeJobs
    
    const progress = Math.round((completedJobs / totalJobs) * 100)
    
    yield* Effect.log(`Scraping Progress: ${progress}% (${completedJobs}/${totalJobs})`, {
      active: activeJobs,
      completed: completedJobs,
      total: totalJobs
    })
    
    // Check for stuck fibers (running too long)
    const stuckFibers = yield* Effect.all(
      fibers.map(fiber => 
        Fiber.status(fiber).pipe(
          Effect.map(status => ({ fiber, status }))
        )
      )
    ).pipe(
      Effect.map(fiberStatuses => 
        fiberStatuses.filter(({ status }) => 
          status._tag === "Running" // Could add time-based logic here
        )
      )
    )
    
    if (stuckFibers.length > 0) {
      yield* Effect.logWarning(`Found ${stuckFibers.length} potentially stuck fibers`)
    }
  }).pipe(
    Effect.repeat(Schedule.spaced(Duration.seconds(2))),
    Effect.fork
  )
  
  // Wait for completion
  const results = yield* Fiber.join(scrapingFiber)
  yield* Fiber.interrupt(progressMonitor)
  
  return results
})

const scrapeUrl = (job: ScrapingJob): Effect.Effect<ScrapingResult, ScrapingError> =>
  Effect.gen(function* () {
    yield* Effect.log(`Starting scrape: ${job.url}`)
    yield* Effect.sleep(Duration.millis(Math.random() * 2000 + 1000)) // Simulate work
    
    // Simulate occasional failures
    if (Math.random() < 0.1) {
      yield* Effect.fail(new ScrapingError(`Failed to scrape ${job.url}`))
    }
    
    return {
      jobId: job.id,
      url: job.url,
      content: `Content from ${job.url}`,
      scrapedAt: new Date()
    }
  }).pipe(
    Effect.withSpan("scrape-url", { attributes: { jobId: job.id, url: job.url } })
  )

class ScrapingError extends Error {
  readonly _tag = "ScrapingError"
  constructor(message: string) {
    super(message)
  }
}
```

### Example 2: Background Task Processing with Health Monitoring

```typescript
import { Effect, Supervisor, Queue, Fiber, Schedule, Clock } from "effect"

interface Task {
  id: string
  type: string
  payload: unknown
  priority: number
  createdAt: Date
}

interface TaskResult {
  taskId: string
  result: unknown
  processedAt: Date
  processingTime: Duration.Duration
}

// Background task processor with health monitoring
const taskProcessorService = Effect.gen(function* () {
  const supervisor = yield* Supervisor.track
  const taskQueue = yield* Queue.bounded<Task>(100)
  const resultsQueue = yield* Queue.bounded<TaskResult>(100)
  
  // Start multiple worker fibers under supervision
  const workerCount = 5
  const workers = yield* Effect.all(
    Array.from({ length: workerCount }, (_, i) => 
      createWorker(i, taskQueue, resultsQueue).pipe(
        Effect.supervised(supervisor),
        Effect.fork
      )
    )
  )
  
  // Health monitoring fiber
  const healthMonitor = yield* createHealthMonitor(supervisor).pipe(
    Effect.repeat(Schedule.spaced(Duration.seconds(5))),
    Effect.fork
  )
  
  // Performance metrics collector
  const metricsCollector = yield* collectPerformanceMetrics(supervisor, resultsQueue).pipe(
    Effect.repeat(Schedule.spaced(Duration.seconds(10))),
    Effect.fork
  )
  
  // Return service interface
  return {
    addTask: (task: Task) => Queue.offer(taskQueue, task),
    getResults: () => Queue.takeAll(resultsQueue),
    getHealthStatus: () => getSystemHealth(supervisor),
    shutdown: () => Effect.gen(function* () {
      yield* Effect.log("Shutting down task processor...")
      yield* Fiber.interruptAll([...workers, healthMonitor, metricsCollector])
      yield* Queue.shutdown(taskQueue)
      yield* Queue.shutdown(resultsQueue)
    })
  }
})

const createWorker = (
  id: number, 
  taskQueue: Queue.Queue<Task>, 
  resultsQueue: Queue.Queue<TaskResult>
) => Effect.gen(function* () {
  yield* Effect.log(`Worker ${id} started`)
  
  yield* Effect.forever(
    Effect.gen(function* () {
      const task = yield* Queue.take(taskQueue)
      
      const startTime = yield* Clock.currentTimeMillis
      const result = yield* processTask(task)
      const endTime = yield* Clock.currentTimeMillis
      
      const taskResult: TaskResult = {
        taskId: task.id,
        result,
        processedAt: new Date(),
        processingTime: Duration.millis(endTime - startTime)
      }
      
      yield* Queue.offer(resultsQueue, taskResult)
      yield* Effect.log(`Worker ${id} completed task ${task.id}`)
    })
  )
}).pipe(
  Effect.withSpan(`worker-${id}`),
  Effect.catchAll((error) => 
    Effect.log(`Worker ${id} failed: ${error}`).pipe(
      Effect.zipRight(Effect.fail(error))
    )
  )
)

const createHealthMonitor = (supervisor: Supervisor.Supervisor<Fiber.RuntimeFiber<any, any>[]>) =>
  Effect.gen(function* () {
    const fibers = yield* supervisor.value
    const fiberStatuses = yield* Effect.all(fibers.map(Fiber.status))
    
    const health = {
      totalFibers: fibers.length,
      runningFibers: fiberStatuses.filter(s => s._tag === "Running").length,
      suspendedFibers: fiberStatuses.filter(s => s._tag === "Suspended").length,
      timestamp: new Date()
    }
    
    yield* Effect.log("System Health Check", health)
    
    // Alert on unhealthy conditions
    if (health.runningFibers === 0 && health.totalFibers > 0) {
      yield* Effect.logWarning("No running fibers detected - system may be stalled")
    }
    
    if (health.suspendedFibers > health.totalFibers * 0.8) {
      yield* Effect.logWarning("High number of suspended fibers - possible bottleneck")
    }
    
    return health
  })

const collectPerformanceMetrics = (
  supervisor: Supervisor.Supervisor<Fiber.RuntimeFiber<any, any>[]>,
  resultsQueue: Queue.Queue<TaskResult>
) => Effect.gen(function* () {
  const fibers = yield* supervisor.value
  const recentResults = yield* Queue.takeUpTo(resultsQueue, 100)
  
  if (recentResults.length > 0) {
    const avgProcessingTime = recentResults.reduce(
      (sum, result) => sum + Duration.toMillis(result.processingTime), 
      0
    ) / recentResults.length
    
    const metrics = {
      activeFibers: fibers.length,
      tasksProcessed: recentResults.length,
      avgProcessingTimeMs: avgProcessingTime,
      timestamp: new Date()
    }
    
    yield* Effect.log("Performance Metrics", metrics)
  }
})

const processTask = (task: Task): Effect.Effect<unknown, TaskProcessingError> =>
  Effect.gen(function* () {
    // Simulate different task types with varying processing times
    const processingTime = task.type === "heavy" ? 
      Duration.seconds(2) : 
      Duration.millis(500)
    
    yield* Effect.sleep(processingTime)
    
    // Simulate occasional failures
    if (Math.random() < 0.05) {
      yield* Effect.fail(new TaskProcessingError(`Task ${task.id} failed`))
    }
    
    return `Processed ${task.type} task: ${JSON.stringify(task.payload)}`
  }).pipe(
    Effect.withSpan("process-task", { 
      attributes: { taskId: task.id, taskType: task.type } 
    })
  )

const getSystemHealth = (supervisor: Supervisor.Supervisor<Fiber.RuntimeFiber<any, any>[]>) =>
  Effect.gen(function* () {
    const fibers = yield* supervisor.value
    const statuses = yield* Effect.all(fibers.map(Fiber.status))
    
    return {
      isHealthy: fibers.length > 0 && statuses.some(s => s._tag === "Running"),
      fiberCount: fibers.length,
      runningCount: statuses.filter(s => s._tag === "Running").length,
      suspendedCount: statuses.filter(s => s._tag === "Suspended").length
    }
  })

class TaskProcessingError extends Error {
  readonly _tag = "TaskProcessingError"
  constructor(message: string) {
    super(message)
  }
}
```

### Example 3: Distributed System Coordination with Hierarchical Supervision

```typescript
import { Effect, Supervisor, Fiber, Schedule, Duration, Logger } from "effect"

interface ServiceNode {
  id: string
  name: string
  endpoint: string
  healthCheckInterval: Duration.Duration
}

interface ServiceHealth {
  nodeId: string
  isHealthy: boolean
  lastCheck: Date
  responseTime?: number
  error?: string
}

// Distributed service coordinator with hierarchical supervision
const serviceCoordinator = Effect.gen(function* () {
  // Master supervisor tracks all service groups
  const masterSupervisor = yield* Supervisor.track
  
  const services: ServiceNode[] = [
    { id: "api-1", name: "API Gateway", endpoint: "http://api-1:8080", healthCheckInterval: Duration.seconds(10) },
    { id: "db-1", name: "Database", endpoint: "http://db-1:5432", healthCheckInterval: Duration.seconds(30) },
    { id: "cache-1", name: "Cache", endpoint: "http://cache-1:6379", healthCheckInterval: Duration.seconds(15) }
  ]
  
  // Create service group supervisors
  const serviceGroups = yield* Effect.all(
    services.map(service => createServiceGroup(service, masterSupervisor))
  )
  
  // Global system monitor
  const systemMonitor = yield* createSystemMonitor(masterSupervisor).pipe(
    Effect.repeat(Schedule.spaced(Duration.seconds(30))),
    Effect.fork
  )
  
  // Failure recovery coordinator
  const recoveryCoordinator = yield* createRecoveryCoordinator(masterSupervisor).pipe(
    Effect.repeat(Schedule.spaced(Duration.seconds(60))),
    Effect.fork
  )
  
  return {
    getSystemStatus: () => getDistributedSystemStatus(masterSupervisor),
    triggerHealthCheck: () => triggerAllHealthChecks(masterSupervisor),
    shutdown: () => Effect.gen(function* () {
      yield* Effect.log("Shutting down service coordinator...")
      yield* Fiber.interrupt(systemMonitor)
      yield* Fiber.interrupt(recoveryCoordinator)
      
      // Gracefully shutdown all service groups
      const allFibers = yield* masterSupervisor.value
      yield* Fiber.interruptAll(allFibers)
    })
  }
})

const createServiceGroup = (service: ServiceNode, parentSupervisor: Supervisor.Supervisor<any>) =>
  Effect.gen(function* () {
    // Each service group has its own supervisor
    const groupSupervisor = yield* Supervisor.track
    
    // Health checker fiber for this service
    const healthChecker = yield* createHealthChecker(service).pipe(
      Effect.supervised(groupSupervisor),
      Effect.repeat(Schedule.spaced(service.healthCheckInterval)),
      Effect.fork
    )
    
    // Service-specific monitoring
    const serviceMonitor = yield* createServiceMonitor(service, groupSupervisor).pipe(
      Effect.supervised(groupSupervisor),
      Effect.repeat(Schedule.spaced(Duration.seconds(20))),
      Effect.fork
    )
    
    // Register the group with parent supervisor
    const groupCoordinator = yield* Effect.gen(function* () {
      yield* Effect.log(`Service group ${service.name} initialized`)
      
      // Keep group alive and monitor for failures
      yield* Effect.forever(
        Effect.gen(function* () {
          yield* Effect.sleep(Duration.seconds(5))
          
          const groupFibers = yield* groupSupervisor.value
          if (groupFibers.length === 0) {
            yield* Effect.logWarning(`Service group ${service.name} has no active fibers`)
          }
        })
      )
    }).pipe(
      Effect.supervised(parentSupervisor),
      Effect.fork
    )
    
    return {
      service,
      supervisor: groupSupervisor,
      fibers: [healthChecker, serviceMonitor, groupCoordinator]
    }
  })

const createHealthChecker = (service: ServiceNode) =>
  Effect.gen(function* () {
    const startTime = yield* Clock.currentTimeMillis
    
    // Simulate health check (would be actual HTTP call in real implementation)
    const isHealthy = Math.random() > 0.1 // 90% success rate
    yield* Effect.sleep(Duration.millis(Math.random() * 100 + 50)) // Simulate network delay
    
    const endTime = yield* Clock.currentTimeMillis
    const responseTime = endTime - startTime
    
    const health: ServiceHealth = {
      nodeId: service.id,
      isHealthy,
      lastCheck: new Date(),
      responseTime,
      error: isHealthy ? undefined : "Service unreachable"
    }
    
    if (isHealthy) {
      yield* Effect.log(`Health check passed for ${service.name}`, { responseTime })
    } else {
      yield* Effect.logError(`Health check failed for ${service.name}`, { 
        error: health.error,
        responseTime 
      })
    }
    
    return health
  }).pipe(
    Effect.withSpan("health-check", { 
      attributes: { serviceId: service.id, serviceName: service.name } 
    }),
    Effect.catchAll((error) =>
      Effect.gen(function* () {
        yield* Effect.logError(`Health check error for ${service.name}: ${error}`)
        return {
          nodeId: service.id,
          isHealthy: false,
          lastCheck: new Date(),
          error: String(error)
        } as ServiceHealth
      })
    )
  )

const createServiceMonitor = (service: ServiceNode, supervisor: Supervisor.Supervisor<any>) =>
  Effect.gen(function* () {
    const fibers = yield* supervisor.value
    const fiberStatuses = yield* Effect.all(fibers.map(Fiber.status))
    
    const monitoring = {
      serviceId: service.id,
      serviceName: service.name,
      activeFibers: fibers.length,
      runningFibers: fiberStatuses.filter(s => s._tag === "Running").length,
      suspendedFibers: fiberStatuses.filter(s => s._tag === "Suspended").length,
      timestamp: new Date()
    }
    
    yield* Effect.log(`Service Monitor - ${service.name}`, monitoring)
    
    // Detect service-level issues
    if (monitoring.runningFibers === 0 && monitoring.activeFibers > 0) {
      yield* Effect.logWarning(`Service ${service.name} has stalled fibers`)
    }
    
    return monitoring
  })

const createSystemMonitor = (masterSupervisor: Supervisor.Supervisor<any>) =>
  Effect.gen(function* () {
    const allFibers = yield* masterSupervisor.value
    const systemHealth = yield* getDistributedSystemStatus(masterSupervisor)
    
    yield* Effect.log("System Status", {
      totalFibers: allFibers.length,
      healthyServices: systemHealth.services.filter(s => s.isHealthy).length,
      totalServices: systemHealth.services.length,
      uptime: systemHealth.uptime
    })
    
    // System-wide health checks
    const unhealthyServices = systemHealth.services.filter(s => !s.isHealthy)
    if (unhealthyServices.length > 0) {
      yield* Effect.logError(`Unhealthy services detected: ${unhealthyServices.map(s => s.name).join(", ")}`)
    }
    
    // Check for memory leaks (growing fiber count)
    if (allFibers.length > 100) {
      yield* Effect.logWarning(`High fiber count detected: ${allFibers.length}`)
    }
  })

const createRecoveryCoordinator = (masterSupervisor: Supervisor.Supervisor<any>) =>
  Effect.gen(function* () {
    const allFibers = yield* masterSupervisor.value
    const stuckFibers = yield* Effect.all(
      allFibers.map(fiber =>
        Fiber.status(fiber).pipe(
          Effect.map(status => ({ fiber, status }))
        )
      )
    ).pipe(
      Effect.map(fiberStatuses =>
        fiberStatuses.filter(({ status }) => status._tag === "Suspended")
      )
    )
    
    if (stuckFibers.length > 0) {
      yield* Effect.logWarning(`Recovery coordinator found ${stuckFibers.length} stuck fibers`)
      
      // Could implement recovery strategies here:
      // - Restart stuck fibers
      // - Escalate to service restart
      // - Alert operations team
    }
    
    return {
      totalFibers: allFibers.length,
      stuckFibers: stuckFibers.length,
      recoveryActions: 0 // Would track actual recovery actions
    }
  })

const getDistributedSystemStatus = (supervisor: Supervisor.Supervisor<any>) =>
  Effect.gen(function* () {
    const fibers = yield* supervisor.value
    
    // In a real implementation, this would aggregate actual service health data
    const mockServices = [
      { name: "API Gateway", isHealthy: true },
      { name: "Database", isHealthy: true },
      { name: "Cache", isHealthy: Math.random() > 0.2 }
    ]
    
    return {
      services: mockServices,
      totalFibers: fibers.length,
      uptime: Duration.minutes(42), // Would track actual uptime
      lastUpdate: new Date()
    }
  })

const triggerAllHealthChecks = (supervisor: Supervisor.Supervisor<any>) =>
  Effect.gen(function* () {
    const fibers = yield* supervisor.value
    yield* Effect.log(`Triggering health checks for ${fibers.length} services`)
    // Implementation would send signals to health check fibers
    return { triggered: fibers.length }
  })

// Import Clock for timing operations
import { Clock } from "effect"
```

## Advanced Features Deep Dive

### Feature 1: Custom Supervisor Implementation

You can create custom supervisors by extending the `AbstractSupervisor` class to implement specialized monitoring and control logic.

#### Basic Custom Supervisor

```typescript
import { Effect, Supervisor, Logger, MutableRef } from "effect"

// Custom supervisor that logs all fiber lifecycle events
class LoggingSupervisor extends Supervisor.AbstractSupervisor<void> {
  constructor(private logger: Logger.Logger) {
    super()
  }
  
  get value(): Effect.Effect<void> {
    return Effect.void
  }
  
  onStart<A, E, R>(
    context: Context.Context<R>,
    effect: Effect.Effect<A, E, R>,
    parent: Option.Option<Fiber.RuntimeFiber<any, any>>,
    fiber: Fiber.RuntimeFiber<A, E>
  ): void {
    Effect.runSync(
      Effect.log(`Fiber started: ${fiber.id()}`, {
        hasParent: Option.isSome(parent),
        parentId: Option.match(parent, {
          onNone: () => undefined,
          onSome: (p) => p.id()
        })
      }).pipe(Effect.provide(this.logger))
    )
  }
  
  onEnd<A, E>(value: Exit.Exit<A, E>, fiber: Fiber.RuntimeFiber<A, E>): void {
    Effect.runSync(
      Effect.log(`Fiber ended: ${fiber.id()}`, {
        success: Exit.isSuccess(value),
        exitType: value._tag
      }).pipe(Effect.provide(this.logger))
    )
  }
  
  onSuspend<A, E>(fiber: Fiber.RuntimeFiber<A, E>): void {
    Effect.runSync(
      Effect.log(`Fiber suspended: ${fiber.id()}`).pipe(Effect.provide(this.logger))
    )
  }
  
  onResume<A, E>(fiber: Fiber.RuntimeFiber<A, E>): void {
    Effect.runSync(
      Effect.log(`Fiber resumed: ${fiber.id()}`).pipe(Effect.provide(this.logger))
    )
  }
}

// Usage example
const createLoggingSupervisor = Effect.gen(function* () {
  const logger = yield* Logger.Logger
  return new LoggingSupervisor(logger)
})
```

#### Advanced Custom Supervisor with Metrics

```typescript
import { Effect, Supervisor, Metrics, MutableRef, Duration, Clock } from "effect"

interface FiberMetrics {
  totalStarted: number
  totalEnded: number
  currentActive: number
  totalSuspensions: number
  totalResumptions: number
  avgLifetime: number
  createdAt: Date
}

class MetricsSupervisor extends Supervisor.AbstractSupervisor<FiberMetrics> {
  private metrics: MutableRef.MutableRef<FiberMetrics>
  private fiberStartTimes: Map<Fiber.FiberId, number> = new Map()
  private lifetimes: number[] = []
  
  constructor(initialMetrics: FiberMetrics) {
    super()
    this.metrics = MutableRef.make(initialMetrics) as any // Type assertion for demo
  }
  
  get value(): Effect.Effect<FiberMetrics> {
    return Effect.sync(() => MutableRef.get(this.metrics))
  }
  
  onStart<A, E, R>(
    context: Context.Context<R>,
    effect: Effect.Effect<A, E, R>,
    parent: Option.Option<Fiber.RuntimeFiber<any, any>>,
    fiber: Fiber.RuntimeFiber<A, E>
  ): void {
    const now = Date.now()
    this.fiberStartTimes.set(fiber.id(), now)
    
    const current = MutableRef.get(this.metrics)
    MutableRef.set(this.metrics, {
      ...current,
      totalStarted: current.totalStarted + 1,
      currentActive: current.currentActive + 1
    })
  }
  
  onEnd<A, E>(value: Exit.Exit<A, E>, fiber: Fiber.RuntimeFiber<A, E>): void {
    const now = Date.now()
    const startTime = this.fiberStartTimes.get(fiber.id())
    
    if (startTime !== undefined) {
      const lifetime = now - startTime
      this.lifetimes.push(lifetime)
      this.fiberStartTimes.delete(fiber.id())
      
      // Keep only recent lifetimes for rolling average
      if (this.lifetimes.length > 1000) {
        this.lifetimes = this.lifetimes.slice(-500)
      }
    }
    
    const current = MutableRef.get(this.metrics)
    const avgLifetime = this.lifetimes.length > 0 
      ? this.lifetimes.reduce((a, b) => a + b, 0) / this.lifetimes.length 
      : 0
    
    MutableRef.set(this.metrics, {
      ...current,
      totalEnded: current.totalEnded + 1,
      currentActive: Math.max(0, current.currentActive - 1),
      avgLifetime
    })
  }
  
  onSuspend<A, E>(fiber: Fiber.RuntimeFiber<A, E>): void {
    const current = MutableRef.get(this.metrics)
    MutableRef.set(this.metrics, {
      ...current,
      totalSuspensions: current.totalSuspensions + 1
    })
  }
  
  onResume<A, E>(fiber: Fiber.RuntimeFiber<A, E>): void {
    const current = MutableRef.get(this.metrics)
    MutableRef.set(this.metrics, {
      ...current,
      totalResumptions: current.totalResumptions + 1
    })
  }
}

// Helper to create metrics supervisor
const createMetricsSupervisor = (): MetricsSupervisor => {
  const initialMetrics: FiberMetrics = {
    totalStarted: 0,
    totalEnded: 0,
    currentActive: 0,
    totalSuspensions: 0,
    totalResumptions: 0,
    avgLifetime: 0,
    createdAt: new Date()
  }
  
  return new MetricsSupervisor(initialMetrics)
}
```

### Feature 2: Supervisor Composition

Supervisors can be combined and composed to create sophisticated monitoring hierarchies.

#### Combining Multiple Supervisors

```typescript
import { Effect, Supervisor } from "effect"

// Combine tracking with custom logging
const createComposedSupervisor = Effect.gen(function* () {
  const trackingSupervisor = yield* Supervisor.track
  const loggingSupervisor = yield* createLoggingSupervisor
  
  // Zip supervisors to get both tracking and logging
  const composedSupervisor = trackingSupervisor.zip(loggingSupervisor)
  
  return composedSupervisor
})

// Usage with composed supervisor
const useComposedSupervision = Effect.gen(function* () {
  const supervisor = yield* createComposedSupervisor
  
  const work = Effect.gen(function* () {
    yield* Effect.fork(Effect.delay("100 millis")(Effect.succeed("task 1")))
    yield* Effect.fork(Effect.delay("200 millis")(Effect.succeed("task 2")))
    return "work complete"
  }).pipe(Effect.supervised(supervisor))
  
  yield* work
  
  // Access both tracking and logging results
  const [trackedFibers, loggingResult] = yield* supervisor.value
  
  return {
    activeFibers: trackedFibers.length,
    loggingResult
  }
})
```

#### Mapping Supervisor Output

```typescript
import { Effect, Supervisor } from "effect"

// Transform supervisor output for specific use cases
const createMappedSupervisor = Effect.gen(function* () {
  const baseSupervisor = yield* Supervisor.track
  
  // Map the supervisor to return only fiber count
  const countingSupervisor = baseSupervisor.map(fibers => ({
    count: fibers.length,
    timestamp: new Date(),
    fiberIds: fibers.map(f => f.id())
  }))
  
  return countingSupervisor
})

// Create supervisor that categorizes fibers by status
const createStatusSupervisor = Effect.gen(function* () {
  const baseSupervisor = yield* Supervisor.track
  
  const statusSupervisor = baseSupervisor.map(fibers =>
    Effect.gen(function* () {
      const statuses = yield* Effect.all(fibers.map(Fiber.status))
      
      const categorized = statuses.reduce(
        (acc, status, index) => {
          const category = status._tag
          if (!acc[category]) acc[category] = []
          acc[category].push({ fiber: fibers[index], status })
          return acc
        },
        {} as Record<string, Array<{ fiber: Fiber.RuntimeFiber<any, any>, status: any }>>
      )
      
      return {
        total: fibers.length,
        byStatus: categorized,
        summary: Object.entries(categorized).map(([status, items]) => ({
          status,
          count: items.length
        }))
      }
    })
  )
  
  return statusSupervisor
})
```

### Feature 3: Layer-based Supervision

Use supervisors as layers for application-wide fiber management.

#### Global Supervision Layer

```typescript
import { Effect, Layer, Supervisor } from "effect"

// Create a global supervision layer
const SupervisionLayer = Layer.effect(
  Supervisor.Supervisor,
  Effect.gen(function* () {
    const metricsSupervisor = createMetricsSupervisor()
    
    yield* Effect.log("Global supervision layer initialized")
    
    // Start background metrics reporting
    yield* Effect.fork(
      Effect.gen(function* () {
        yield* Effect.forever(
          Effect.gen(function* () {
            const metrics = yield* metricsSupervisor.value
            yield* Effect.log("Global Fiber Metrics", metrics)
            yield* Effect.sleep(Duration.seconds(30))
          })
        )
      })
    )
    
    return metricsSupervisor
  })
)

// Application using global supervision
const applicationWithSupervision = Effect.gen(function* () {
  const supervisor = yield* Supervisor.Supervisor
  
  // All work is automatically supervised
  const results = yield* Effect.all([
    processOrders().pipe(Effect.supervised(supervisor)),
    handleWebhooks().pipe(Effect.supervised(supervisor)),
    backgroundCleanup().pipe(Effect.supervised(supervisor))
  ], { concurrency: 3 })
  
  return results
}).pipe(
  Effect.provide(SupervisionLayer)
)

const processOrders = () => Effect.gen(function* () {
  yield* Effect.fork(Effect.delay("1 second")(Effect.succeed("order 1")))
  yield* Effect.fork(Effect.delay("2 seconds")(Effect.succeed("order 2")))
  return "orders processed"
})

const handleWebhooks = () => Effect.gen(function* () {
  yield* Effect.fork(Effect.delay("500 millis")(Effect.succeed("webhook 1")))
  return "webhooks handled"
})

const backgroundCleanup = () => Effect.gen(function* () {
  yield* Effect.fork(Effect.delay("3 seconds")(Effect.succeed("cleanup done")))
  return "cleanup completed"
})
```

## Practical Patterns & Best Practices

### Pattern 1: Resource-Aware Supervision

```typescript
import { Effect, Supervisor, Semaphore, Queue } from "effect"

// Supervisor that enforces resource limits
const createResourceAwareSupervisor = (maxConcurrency: number) =>
  Effect.gen(function* () {
    const semaphore = yield* Semaphore.make(maxConcurrency)
    const baseSupervisor = yield* Supervisor.track
    
    const resourceSupervisor = baseSupervisor.map(fibers =>
      Effect.gen(function* () {
        const permitCount = yield* Semaphore.available(semaphore)
        
        return {
          activeFibers: fibers.length,
          availablePermits: permitCount,
          resourceUtilization: ((maxConcurrency - permitCount) / maxConcurrency) * 100,
          isAtCapacity: permitCount === 0
        }
      })
    )
    
    const supervisedExecution = <A, E, R>(effect: Effect.Effect<A, E, R>) =>
      Semaphore.withPermit(semaphore, effect).pipe(
        Effect.supervised(baseSupervisor)
      )
    
    return {
      supervisor: resourceSupervisor,
      execute: supervisedExecution,
      getResourceStatus: () => resourceSupervisor.value
    }
  })

// Usage pattern for resource-constrained operations
const processWithResourceControl = Effect.gen(function* () {
  const { supervisor, execute, getResourceStatus } = yield* createResourceAwareSupervisor(3)
  
  // Launch multiple tasks that will be resource-controlled
  const tasks = Array.from({ length: 10 }, (_, i) =>
    execute(
      Effect.gen(function* () {
        yield* Effect.log(`Task ${i} starting`)
        yield* Effect.sleep(Duration.seconds(1))
        yield* Effect.log(`Task ${i} completed`)
        return `Result ${i}`
      })
    ).pipe(Effect.fork)
  )
  
  // Monitor resource usage
  const monitor = yield* Effect.gen(function* () {
    const status = yield* getResourceStatus()
    yield* Effect.log("Resource Status", status)
  }).pipe(
    Effect.repeat(Schedule.spaced(Duration.seconds(500))),
    Effect.fork
  )
  
  const results = yield* Effect.all(tasks.map(Fiber.join))
  yield* Fiber.interrupt(monitor)
  
  return results
})
```

### Pattern 2: Failure-Aware Supervision

```typescript
import { Effect, Supervisor, Schedule, Duration } from "effect"

interface FailureTrackingSupervisor {
  supervisor: Supervisor.Supervisor<{
    totalFailures: number
    recentFailures: Array<{ fiberId: string, error: any, timestamp: Date }>
    failureRate: number
  }>
  reportFailure: (fiberId: string, error: any) => Effect.Effect<void>
  getFailureStats: () => Effect.Effect<{
    totalFailures: number
    recentFailures: number
    failureRate: number
  }>
}

const createFailureTrackingSupervisor = (): Effect.Effect<FailureTrackingSupervisor> =>
  Effect.gen(function* () {
    const failures = yield* MutableRef.make<Array<{ fiberId: string, error: any, timestamp: Date }>>([])
    const baseSupervisor = yield* Supervisor.track
    
    const failureSupervisor = baseSupervisor.map(fibers =>
      Effect.gen(function* () {
        const allFailures = MutableRef.get(failures)
        
        // Keep only failures from last 5 minutes
        const fiveMinutesAgo = new Date(Date.now() - 5 * 60 * 1000)
        const recentFailures = allFailures.filter(f => f.timestamp > fiveMinutesAgo)
        
        // Update the ref to remove old failures
        MutableRef.set(failures, recentFailures)
        
        const failureRate = recentFailures.length / Math.max(1, fibers.length + recentFailures.length)
        
        return {
          totalFailures: allFailures.length,
          recentFailures,
          failureRate
        }
      })
    )
    
    const reportFailure = (fiberId: string, error: any) =>
      Effect.sync(() => {
        const currentFailures = MutableRef.get(failures)
        MutableRef.set(failures, [
          ...currentFailures,
          { fiberId, error, timestamp: new Date() }
        ])
      })
    
    const getFailureStats = () =>
      Effect.gen(function* () {
        const stats = yield* failureSupervisor.value
        return {
          totalFailures: stats.totalFailures,
          recentFailures: stats.recentFailures.length,
          failureRate: stats.failureRate
        }
      })
    
    return {
      supervisor: failureSupervisor,
      reportFailure,
      getFailureStats
    }
  })

// Resilient task execution with failure tracking
const resilientTaskExecution = Effect.gen(function* () {
  const failureTracker = yield* createFailureTrackingSupervisor()
  
  const executeTaskWithTracking = <A>(
    taskId: string,
    task: Effect.Effect<A, any>
  ): Effect.Effect<A> =>
    task.pipe(
      Effect.supervised(failureTracker.supervisor),
      Effect.tapError(error => failureTracker.reportFailure(taskId, error)),
      Effect.retry(Schedule.exponential("100 millis").pipe(
        Schedule.intersect(Schedule.recurs(3))
      ))
    )
  
  // Execute multiple tasks with failure tracking
  const tasks = [
    executeTaskWithTracking("task-1", simulateWork("Task 1", 0.1)),
    executeTaskWithTracking("task-2", simulateWork("Task 2", 0.2)),
    executeTaskWithTracking("task-3", simulateWork("Task 3", 0.15))
  ]
  
  const results = yield* Effect.all(tasks, { concurrency: 2 })
  const failureStats = yield* failureTracker.getFailureStats()
  
  yield* Effect.log("Execution completed", {
    results: results.length,
    failures: failureStats
  })
  
  return { results, failureStats }
})

const simulateWork = (name: string, failureRate: number) =>
  Effect.gen(function* () {
    yield* Effect.sleep(Duration.millis(Math.random() * 1000 + 500))
    
    if (Math.random() < failureRate) {
      yield* Effect.fail(new Error(`${name} failed randomly`))
    }
    
    return `${name} completed successfully`
  })
```

### Pattern 3: Hierarchical Circuit Breaker

```typescript
import { Effect, Supervisor, Schedule, Duration, MutableRef } from "effect"

interface CircuitBreakerState {
  isOpen: boolean
  failureCount: number
  lastFailureTime?: Date
  successCount: number
}

const createCircuitBreakerSupervisor = (
  failureThreshold: number,
  timeoutWindow: Duration.Duration
) => Effect.gen(function* () {
  const state = yield* MutableRef.make<CircuitBreakerState>({
    isOpen: false,
    failureCount: 0,
    successCount: 0
  })
  
  const baseSupervisor = yield* Supervisor.track
  
  const circuitSupervisor = baseSupervisor.map(fibers =>
    Effect.gen(function* () {
      const currentState = MutableRef.get(state)
      
      // Check if circuit should be closed after timeout
      if (currentState.isOpen && currentState.lastFailureTime) {
        const timeSinceFailure = Date.now() - currentState.lastFailureTime.getTime()
        if (timeSinceFailure > Duration.toMillis(timeoutWindow)) {
          MutableRef.set(state, {
            ...currentState,
            isOpen: false,
            failureCount: 0
          })
        }
      }
      
      return {
        ...currentState,
        activeFibers: fibers.length,
        healthStatus: currentState.isOpen ? "CIRCUIT_OPEN" : "HEALTHY"
      }
    })
  )
  
  const executeWithCircuitBreaker = <A, E>(effect: Effect.Effect<A, E>) =>
    Effect.gen(function* () {
      const currentState = MutableRef.get(state)
      
      if (currentState.isOpen) {
        yield* Effect.fail(new Error("Circuit breaker is open"))
      }
      
      const result = yield* effect.pipe(
        Effect.supervised(baseSupervisor),
        Effect.tapError(() =>
          Effect.sync(() => {
            const current = MutableRef.get(state)
            const newFailureCount = current.failureCount + 1
            
            MutableRef.set(state, {
              ...current,
              failureCount: newFailureCount,
              lastFailureTime: new Date(),
              isOpen: newFailureCount >= failureThreshold
            })
          })
        ),
        Effect.tap(() =>
          Effect.sync(() => {
            const current = MutableRef.get(state)
            MutableRef.set(state, {
              ...current,
              successCount: current.successCount + 1,
              failureCount: Math.max(0, current.failureCount - 1) // Gradual recovery
            })
          })
        )
      )
      
      return result
    })
  
  return {
    supervisor: circuitSupervisor,
    execute: executeWithCircuitBreaker,
    getState: () => Effect.sync(() => MutableRef.get(state)),
    forceOpen: () => Effect.sync(() => {
      const current = MutableRef.get(state)
      MutableRef.set(state, { ...current, isOpen: true })
    }),
    forceClose: () => Effect.sync(() => {
      MutableRef.set(state, {
        isOpen: false,
        failureCount: 0,
        successCount: 0
      })
    })
  }
})
```

## Integration Examples

### Integration with Effect Testing

```typescript
import { Effect, Supervisor, TestClock, TestContext, Duration } from "effect"

// Testing fiber supervision behavior
describe("Supervisor Integration Tests", () => {
  it("should track fiber lifecycle correctly", async () => {
    const testProgram = Effect.gen(function* () {
      const supervisor = yield* Supervisor.track
      
      // Start some test fibers
      const fiber1 = yield* Effect.fork(
        Effect.delay("100 millis")(Effect.succeed("result1"))
      ).pipe(Effect.supervised(supervisor))
      
      const fiber2 = yield* Effect.fork(
        Effect.delay("200 millis")(Effect.succeed("result2"))
      ).pipe(Effect.supervised(supervisor))
      
      // Check initial state
      const initialFibers = yield* supervisor.value
      expect(initialFibers.length).toBe(2)
      
      // Advance time and check intermediate state
      yield* TestClock.adjust("150 millis")
      const midFibers = yield* supervisor.value
      expect(midFibers.length).toBe(1) // One should be complete
      
      // Complete all fibers
      yield* TestClock.adjust("100 millis")
      const finalFibers = yield* supervisor.value
      expect(finalFibers.length).toBe(0)
      
      return "test complete"
    }).pipe(
      Effect.provide(TestClock.layer)
    )
    
    const result = await Effect.runPromise(testProgram)
    expect(result).toBe("test complete")
  })
  
  it("should handle supervised fiber failures", async () => {
    const testProgram = Effect.gen(function* () {
      const supervisor = yield* Supervisor.track
      
      const failingFiber = yield* Effect.fork(
        Effect.gen(function* () {
          yield* Effect.sleep("50 millis")
          yield* Effect.fail(new Error("Test failure"))
        })
      ).pipe(Effect.supervised(supervisor))
      
      const successFiber = yield* Effect.fork(
        Effect.delay("100 millis")(Effect.succeed("success"))
      ).pipe(Effect.supervised(supervisor))
      
      // Advance time to trigger failure
      yield* TestClock.adjust("75 millis")
      
      const fibers = yield* supervisor.value
      const statuses = yield* Effect.all(fibers.map(Fiber.status))
      
      // One should be done (failed), one still running
      const doneCount = statuses.filter(s => s._tag === "Done").length
      const runningCount = statuses.filter(s => s._tag === "Running").length
      
      expect(doneCount).toBe(1)
      expect(runningCount).toBe(1)
      
      return "failure test complete"
    }).pipe(
      Effect.provide(TestClock.layer)
    )
    
    const result = await Effect.runPromise(testProgram)
    expect(result).toBe("failure test complete")
  })
})
```

### Integration with Effect Metrics

```typescript
import { Effect, Supervisor, Metrics, Layer } from "effect"

// Create metrics-integrated supervisor
const createMetricsIntegratedSupervisor = Effect.gen(function* () {
  const baseSupervisor = yield* Supervisor.track
  
  // Define metrics for fiber tracking
  const activeFibersGauge = Metrics.gauge("supervisor_active_fibers", {
    description: "Number of currently active fibers"
  })
  
  const fiberStartsCounter = Metrics.counter("supervisor_fiber_starts", {
    description: "Total number of fibers started"
  })
  
  const fiberEndsCounter = Metrics.counter("supervisor_fiber_ends", {
    description: "Total number of fibers ended"
  })
  
  // Create enhanced supervisor that updates metrics
  const metricsSupervisor = baseSupervisor.map(fibers =>
    Effect.gen(function* () {
      // Update active fibers gauge
      yield* Metrics.update(activeFibersGauge, fibers.length)
      
      return {
        activeFibers: fibers.length,
        fiberIds: fibers.map(f => f.id()),
        timestamp: new Date()
      }
    })
  )
  
  // Custom supervisor that tracks starts/ends
  class MetricsTrackingSupervisor extends Supervisor.AbstractSupervisor<{
    activeFibers: number
    totalStarts: number
    totalEnds: number
  }> {
    private totalStarts = 0
    private totalEnds = 0
    private activeFibers = 0
    
    get value() {
      return Effect.succeed({
        activeFibers: this.activeFibers,
        totalStarts: this.totalStarts,
        totalEnds: this.totalEnds
      })
    }
    
    onStart<A, E, R>(
      context: Context.Context<R>,
      effect: Effect.Effect<A, E, R>,
      parent: Option.Option<Fiber.RuntimeFiber<any, any>>,
      fiber: Fiber.RuntimeFiber<A, E>
    ): void {
      this.totalStarts++
      this.activeFibers++
      
      // Update metrics asynchronously
      Effect.runFork(Metrics.increment(fiberStartsCounter))
      Effect.runFork(Metrics.update(activeFibersGauge, this.activeFibers))
    }
    
    onEnd<A, E>(value: Exit.Exit<A, E>, fiber: Fiber.RuntimeFiber<A, E>): void {
      this.totalEnds++
      this.activeFibers = Math.max(0, this.activeFibers - 1)
      
      // Update metrics asynchronously
      Effect.runFork(Metrics.increment(fiberEndsCounter))
      Effect.runFork(Metrics.update(activeFibersGauge, this.activeFibers))
    }
  }
  
  return new MetricsTrackingSupervisor()
})

// Application with metrics integration
const applicationWithMetrics = Effect.gen(function* () {
  const supervisor = yield* createMetricsIntegratedSupervisor
  
  // Simulate some work that creates fibers
  const work = Effect.gen(function* () {
    const fibers = yield* Effect.all(
      Array.from({ length: 10 }, (_, i) =>
        Effect.fork(
          Effect.delay(`${i * 100}millis`)(Effect.succeed(`Task ${i}`))
        )
      )
    ).pipe(Effect.supervised(supervisor))
    
    yield* Effect.all(fibers.map(Fiber.join))
    return "All tasks completed"
  })
  
  const result = yield* work
  const finalMetrics = yield* supervisor.value
  
  yield* Effect.log("Final metrics", finalMetrics)
  
  return { result, metrics: finalMetrics }
}).pipe(
  Effect.provide(Metrics.layer) // Provide metrics layer
)
```

### Integration with Effect Logger

```typescript
import { Effect, Supervisor, Logger, LogLevel } from "effect"

// Create supervisor with structured logging
const createLoggingIntegratedSupervisor = Effect.gen(function* () {
  const baseSupervisor = yield* Supervisor.track
  
  // Enhanced supervisor that logs fiber lifecycle with context
  class StructuredLoggingSupervisor extends Supervisor.AbstractSupervisor<{
  activeFibers: number
  events: Array<{ type: string, fiberId: string, timestamp: Date }>
}> {
    private events: Array<{ type: string, fiberId: string, timestamp: Date }> = []
    private maxEvents = 1000
    
    get value() {
      return Effect.gen(function* () {
        const baseFibers = yield* baseSupervisor.value
        return {
          activeFibers: baseFibers.length,
          events: this.events.slice(-100) // Return recent events
        }
      })
    }
    
    private addEvent(type: string, fiberId: string) {
      this.events.push({ type, fiberId, timestamp: new Date() })
      if (this.events.length > this.maxEvents) {
        this.events = this.events.slice(-this.maxEvents / 2)
      }
    }
    
    onStart<A, E, R>(
      context: Context.Context<R>,
      effect: Effect.Effect<A, E, R>,
      parent: Option.Option<Fiber.RuntimeFiber<any, any>>,
      fiber: Fiber.RuntimeFiber<A, E>
    ): void {
      this.addEvent("start", fiber.id().toString())
      
      Effect.runFork(
        Effect.logDebug("Fiber started", {
          fiberId: fiber.id(),
          hasParent: Option.isSome(parent),
          parentId: Option.match(parent, {
            onNone: () => null,
            onSome: (p) => p.id()
          })
        })
      )
    }
    
    onEnd<A, E>(value: Exit.Exit<A, E>, fiber: Fiber.RuntimeFiber<A, E>): void {
      this.addEvent("end", fiber.id().toString())
      
      const logLevel = Exit.isSuccess(value) ? LogLevel.Debug : LogLevel.Warning
      
      Effect.runFork(
        Effect.log("Fiber ended", {
          fiberId: fiber.id(),
          success: Exit.isSuccess(value),
          exitType: value._tag
        }).pipe(Effect.withLogSpan("fiber-lifecycle"))
      )
    }
    
    onSuspend<A, E>(fiber: Fiber.RuntimeFiber<A, E>): void {
      this.addEvent("suspend", fiber.id().toString())
      
      Effect.runFork(
        Effect.logTrace("Fiber suspended", {
          fiberId: fiber.id()
        })
      )
    }
    
    onResume<A, E>(fiber: Fiber.RuntimeFiber<A, E>): void {
      this.addEvent("resume", fiber.id().toString())
      
      Effect.runFork(
        Effect.logTrace("Fiber resumed", {
          fiberId: fiber.id()
        })
      )
    }
  }
  
  return new StructuredLoggingSupervisor()
})

// Usage with custom logger configuration
const applicationWithStructuredLogging = Effect.gen(function* () {
  const supervisor = yield* createLoggingIntegratedSupervisor
  
  const work = Effect.gen(function* () {
    yield* Effect.log("Starting supervised work")
    
    const results = yield* Effect.all([
      Effect.fork(Effect.delay("100 millis")(Effect.succeed("A"))),
      Effect.fork(Effect.delay("200 millis")(Effect.succeed("B"))),
      Effect.fork(Effect.delay("150 millis")(Effect.fail(new Error("C failed"))))
    ]).pipe(
      Effect.supervised(supervisor),
      Effect.map(fibers => Effect.all(fibers.map(Fiber.await))),
      Effect.flatten,
      Effect.catchAll(errors => 
        Effect.log("Some fibers failed", { errors }).pipe(
          Effect.as([])
        )
      )
    )
    
    const supervisorState = yield* supervisor.value
    yield* Effect.log("Work completed", {
      results: results.length,
      events: supervisorState.events.length,
      activeFibers: supervisorState.activeFibers
    })
    
    return results
  })
  
  return yield* work
}).pipe(
  Effect.provide(
    Logger.replace(
      Logger.defaultLogger,
      Logger.make(({ message, logLevel, annotations, spans }) => {
        const timestamp = new Date().toISOString()
        const level = logLevel.label
        const spanInfo = spans.length > 0 ? 
          ` [${spans.map(s => s.label).join(" > ")}]` : ""
        
        const logData = {
          timestamp,
          level,
          message,
          ...annotations,
          spans: spanInfo
        }
        
        console.log(JSON.stringify(logData, null, 2))
      })
    )
  )
)
```

## Conclusion

Supervisor provides comprehensive fiber lifecycle management, monitoring, and control for Effect applications. It enables visibility into concurrent operations, resource management, failure tracking, and system observability that are essential for production applications.

Key benefits:
- **Complete Visibility** - Track fiber creation, execution, suspension, resumption, and termination
- **Fine-grained Control** - Selectively monitor, cancel, or restart specific fibers and operations
- **Resource Management** - Implement resource-aware supervision with limits and cleanup guarantees
- **Observability Integration** - Seamlessly integrate with metrics, logging, and monitoring systems
- **Hierarchical Organization** - Create supervision trees that match your application architecture

Use Supervisor when you need insight into or control over concurrent operations, especially in production systems where monitoring, resource management, and failure recovery are critical requirements.