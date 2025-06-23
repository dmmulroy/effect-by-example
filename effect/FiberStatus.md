# FiberStatus: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem FiberStatus Solves

When building concurrent applications, debugging and monitoring fiber execution becomes challenging. Traditional approaches to tracking async operations often lack visibility into what fibers are doing at any given moment:

```typescript
// Traditional async approach - no visibility into execution state
const processData = async (data: unknown[]) => {
  const promises = data.map(async (item) => {
    // Is this still running? Blocked? What's it waiting for?
    return await processItem(item)
  })
  return await Promise.all(promises)
}

// Traditional debugging - limited information
console.log("Tasks started...") // That's all we know
```

This approach leads to:
- **Poor Observability** - No way to inspect what concurrent operations are doing
- **Difficult Debugging** - When things hang, you can't see what's blocking
- **Limited Monitoring** - No insight into fiber lifecycle for performance tuning
- **Maintenance Challenges** - Hard to identify bottlenecks in complex concurrent systems

### The FiberStatus Solution

FiberStatus provides comprehensive visibility into fiber execution states, enabling rich monitoring and debugging capabilities:

```typescript
import { Effect, Fiber, FiberStatus } from "effect"

const monitoredProcess = Effect.gen(function* () {
  const fiber = yield* Effect.fork(longRunningTask)
  
  // Rich status information available at any time
  const status = yield* Fiber.status(fiber)
  
  // Type-safe pattern matching on status
  if (FiberStatus.isRunning(status)) {
    console.log(`Fiber running with flags: ${status.runtimeFlags}`)
  } else if (FiberStatus.isSuspended(status)) {
    console.log(`Fiber suspended, blocking on: ${status.blockingOn}`)
  } else if (FiberStatus.isDone(status)) {
    console.log("Fiber completed")
  }
  
  return yield* Fiber.await(fiber)
})
```

### Key Concepts

**Done**: Fiber has completed execution (either successfully or with failure)

**Running**: Fiber is actively executing with specific runtime flags controlling its behavior

**Suspended**: Fiber is blocked waiting for other fibers, with visibility into what it's waiting for

## Basic Usage Patterns

### Pattern 1: Basic Status Checking

```typescript
import { Effect, Fiber, FiberStatus } from "effect"

const checkFiberStatus = Effect.gen(function* () {
  const fiber = yield* Effect.fork(
    Effect.gen(function* () {
      yield* Effect.sleep("2 seconds")
      return "completed"
    })
  )
  
  // Check status immediately after forking
  const initialStatus = yield* Fiber.status(fiber)
  console.log("Initial status:", FiberStatus.isRunning(initialStatus) ? "Running" : "Other")
  
  // Wait and check again
  yield* Effect.sleep("1 second")
  const midStatus = yield* Fiber.status(fiber)
  console.log("Mid-execution status:", FiberStatus.isRunning(midStatus) ? "Running" : "Other")
  
  const result = yield* Fiber.await(fiber)
  const finalStatus = yield* Fiber.status(fiber)
  console.log("Final status:", FiberStatus.isDone(finalStatus) ? "Done" : "Other")
  
  return result
})
```

### Pattern 2: Status-Based Conditional Logic

```typescript
const smartFiberManager = (fiber: Fiber.RuntimeFiber<string, Error>) =>
  Effect.gen(function* () {
    const status = yield* Fiber.status(fiber)
    
    if (FiberStatus.isDone(status)) {
      return yield* Fiber.await(fiber)
    }
    
    if (FiberStatus.isRunning(status)) {
      // Give it more time to complete
      yield* Effect.sleep("5 seconds")
      return yield* Fiber.await(fiber)
    }
    
    if (FiberStatus.isSuspended(status)) {
      // Fiber is blocked - we might want to interrupt it
      console.log(`Fiber blocked on: ${status.blockingOn}`)
      yield* Fiber.interrupt(fiber)
      return "interrupted due to suspension"
    }
    
    return "unknown status"
  })
```

### Pattern 3: Polling Status for Monitoring

```typescript
const pollFiberStatus = (fiber: Fiber.RuntimeFiber<unknown, unknown>) =>
  Effect.gen(function* () {
    const checkStatus = Effect.gen(function* () {
      const status = yield* Fiber.status(fiber)
      const timestamp = new Date().toISOString()
      
      if (FiberStatus.isRunning(status)) {
        console.log(`[${timestamp}] Fiber is running`)
        return false // Continue polling
      } else if (FiberStatus.isSuspended(status)) {
        console.log(`[${timestamp}] Fiber suspended, waiting on: ${status.blockingOn}`)
        return false // Continue polling
      } else {
        console.log(`[${timestamp}] Fiber completed`)
        return true // Stop polling
      }
    })
    
    // Poll every 500ms until fiber is done
    yield* Effect.repeatUntil(
      Effect.zipRight(checkStatus, Effect.sleep("500 millis")),
      (isDone) => isDone
    )
  })
```

## Real-World Examples

### Example 1: Background Task Monitoring Dashboard

```typescript
import { Effect, Fiber, FiberStatus, Ref } from "effect"

interface TaskMetrics {
  readonly taskId: string
  readonly status: "running" | "suspended" | "completed"
  readonly startTime: Date
  readonly lastUpdate: Date
  readonly blockingOn?: string
}

class TaskMonitor {
  constructor(private metricsRef: Ref.Ref<Map<string, TaskMetrics>>) {}
  
  startTask = (taskId: string, task: Effect.Effect<unknown, unknown, unknown>) =>
    Effect.gen(function* () {
      const fiber = yield* Effect.fork(task)
      const startTime = new Date()
      
      // Initialize metrics
      yield* Ref.update(this.metricsRef, (metrics) =>
        metrics.set(taskId, {
          taskId,
          status: "running",
          startTime,
          lastUpdate: startTime
        })
      )
      
      // Start monitoring fiber in background
      yield* Effect.fork(this.monitorFiber(taskId, fiber))
      
      return fiber
    })
  
  private monitorFiber = (
    taskId: string, 
    fiber: Fiber.RuntimeFiber<unknown, unknown>
  ) =>
    Effect.gen(function* () {
      const updateMetrics = Effect.gen(function* () {
        const status = yield* Fiber.status(fiber)
        const lastUpdate = new Date()
        
        if (FiberStatus.isDone(status)) {
          yield* Ref.update(this.metricsRef, (metrics) =>
            metrics.set(taskId, {
              ...metrics.get(taskId)!,
              status: "completed",
              lastUpdate
            })
          )
          return true // Stop monitoring
        }
        
        if (FiberStatus.isSuspended(status)) {
          yield* Ref.update(this.metricsRef, (metrics) =>
            metrics.set(taskId, {
              ...metrics.get(taskId)!,
              status: "suspended",
              lastUpdate,
              blockingOn: status.blockingOn.toString()
            })
          )
        } else {
          yield* Ref.update(this.metricsRef, (metrics) =>
            metrics.set(taskId, {
              ...metrics.get(taskId)!,
              status: "running",
              lastUpdate
            })
          )
        }
        
        return false // Continue monitoring
      })
      
      // Monitor every second until task completes
      yield* Effect.repeatUntil(
        Effect.zipRight(updateMetrics, Effect.sleep("1 second")),
        (isComplete) => isComplete
      )
    })
  
  getMetrics = () => Ref.get(this.metricsRef)
  
  getDashboard = () =>
    Effect.gen(function* () {
      const metrics = yield* this.getMetrics()
      const tasks = Array.from(metrics.values())
      
      console.log("\n=== Task Dashboard ===")
      for (const task of tasks) {
        const duration = Date.now() - task.startTime.getTime()
        const durationStr = `${Math.round(duration / 1000)}s`
        
        let statusInfo = task.status
        if (task.blockingOn) {
          statusInfo += ` (blocking on: ${task.blockingOn})`
        }
        
        console.log(`${task.taskId}: ${statusInfo} - running for ${durationStr}`)
      }
      console.log("====================\n")
    })
}

// Usage example
const createMonitoringExample = Effect.gen(function* () {
  const metricsRef = yield* Ref.make(new Map<string, TaskMetrics>())
  const monitor = new TaskMonitor(metricsRef)
  
  // Start some background tasks
  yield* monitor.startTask("data-processing", 
    Effect.gen(function* () {
      yield* Effect.sleep("5 seconds")
      return "processed"
    })
  )
  
  yield* monitor.startTask("api-calls",
    Effect.gen(function* () {
      yield* Effect.sleep("3 seconds")
      return "api complete"
    })
  )
  
  // Show dashboard updates
  yield* Effect.repeatN(
    Effect.zipRight(monitor.getDashboard(), Effect.sleep("1 second")),
    7
  )
})
```

### Example 2: Deadlock Detection System

```typescript
import { Effect, Fiber, FiberStatus, FiberId, HashSet } from "effect"

interface DeadlockDetector {
  checkForDeadlocks: (fibers: Fiber.RuntimeFiber<unknown, unknown>[]) => Effect.Effect<string[]>
}

const createDeadlockDetector = (): DeadlockDetector => ({
  checkForDeadlocks: (fibers) =>
    Effect.gen(function* () {
      const suspendedFibers = new Map<FiberId.Runtime, FiberId.FiberId>()
      
      // Collect all suspended fibers and what they're blocking on
      for (const fiber of fibers) {
        const status = yield* Fiber.status(fiber)
        if (FiberStatus.isSuspended(status)) {
          suspendedFibers.set(fiber.id(), status.blockingOn)
        }
      }
      
      // Detect cycles in the dependency graph
      const deadlocks: string[] = []
      const visited = new Set<FiberId.Runtime>()
      const path = new Set<FiberId.Runtime>()
      
      const detectCycle = (fiberId: FiberId.Runtime): boolean => {
        if (path.has(fiberId)) {
          // Found a cycle
          const cycleNodes = Array.from(path).concat([fiberId])
          deadlocks.push(`Deadlock detected: ${cycleNodes.map(id => id.toString()).join(" -> ")}`)
          return true
        }
        
        if (visited.has(fiberId)) {
          return false
        }
        
        visited.add(fiberId)
        path.add(fiberId)
        
        const blockingOn = suspendedFibers.get(fiberId)
        if (blockingOn) {
          // Check if any fiber in blockingOn set is also suspended
          const blockingIds = FiberId.ids(blockingOn)
          for (const blockingId of blockingIds) {
            if (suspendedFibers.has(blockingId) && detectCycle(blockingId)) {
              return true
            }
          }
        }
        
        path.delete(fiberId)
        return false
      }
      
      // Check each suspended fiber for cycles
      for (const fiberId of suspendedFibers.keys()) {
        if (!visited.has(fiberId)) {
          detectCycle(fiberId)
        }
      }
      
      return deadlocks
    })
})

// Usage example - simulating potential deadlock
const deadlockExample = Effect.gen(function* () {
  const detector = createDeadlockDetector()
  
  // Create two fibers that could potentially deadlock
  const fiber1 = yield* Effect.fork(
    Effect.gen(function* () {
      yield* Effect.sleep("1 second")
      return "fiber1 complete"
    })
  )
  
  const fiber2 = yield* Effect.fork(
    Effect.gen(function* () {
      // This fiber waits for fiber1
      yield* Fiber.await(fiber1)
      return "fiber2 complete"
    })
  )
  
  // Check for deadlocks periodically
  yield* Effect.sleep("500 millis")
  const deadlocks = yield* detector.checkForDeadlocks([fiber1, fiber2])
  
  if (deadlocks.length > 0) {
    console.log("Deadlocks found:")
    deadlocks.forEach(deadlock => console.log(`  ${deadlock}`))
  } else {
    console.log("No deadlocks detected")
  }
  
  // Clean up
  yield* Fiber.await(fiber2)
})
```

### Example 3: Performance Profiler with Fiber Analysis

```typescript
import { Effect, Fiber, FiberStatus, Ref } from "effect"

interface FiberProfile {
  readonly fiberId: FiberId.Runtime
  readonly totalRunTime: number
  readonly suspendedTime: number
  readonly suspendedCount: number
  readonly createdAt: Date
  readonly completedAt?: Date
}

class FiberProfiler {
  constructor(
    private profiles: Ref.Ref<Map<FiberId.Runtime, FiberProfile>>,
    private startTimes: Ref.Ref<Map<FiberId.Runtime, Date>>
  ) {}
  
  startProfiling = <A, E>(
    effect: Effect.Effect<A, E, unknown>
  ): Effect.Effect<A, E, unknown> =>
    Effect.gen(function* () {
      const fiber = yield* Effect.fork(effect)
      const fiberId = fiber.id()
      const createdAt = new Date()
      
      // Initialize profile
      yield* Ref.update(this.profiles, (profiles) =>
        profiles.set(fiberId, {
          fiberId,
          totalRunTime: 0,
          suspendedTime: 0,
          suspendedCount: 0,
          createdAt
        })
      )
      
      // Start monitoring in background
      yield* Effect.fork(this.monitorFiberPerformance(fiber))
      
      return yield* Fiber.await(fiber)
    })
  
  private monitorFiberPerformance = (
    fiber: Fiber.RuntimeFiber<unknown, unknown>
  ) =>
    Effect.gen(function* () {
      let lastCheckTime = Date.now()
      let lastStatus: FiberStatus.FiberStatus | null = null
      
      const checkPerformance = Effect.gen(function* () {
        const currentTime = Date.now()
        const status = yield* Fiber.status(fiber)
        const fiberId = fiber.id()
        
        if (FiberStatus.isDone(status)) {
          // Final update
          yield* Ref.update(this.profiles, (profiles) => {
            const profile = profiles.get(fiberId)
            if (profile) {
              return profiles.set(fiberId, {
                ...profile,
                completedAt: new Date(currentTime)
              })
            }
            return profiles
          })
          return true // Stop monitoring
        }
        
        // Update timing based on previous status
        if (lastStatus) {
          const timeDelta = currentTime - lastCheckTime
          
          yield* Ref.update(this.profiles, (profiles) => {
            const profile = profiles.get(fiberId)
            if (profile) {
              if (FiberStatus.isSuspended(lastStatus)) {
                return profiles.set(fiberId, {
                  ...profile,
                  suspendedTime: profile.suspendedTime + timeDelta
                })
              } else if (FiberStatus.isRunning(lastStatus)) {
                return profiles.set(fiberId, {
                  ...profile,
                  totalRunTime: profile.totalRunTime + timeDelta
                })
              }
            }
            return profiles
          })
        }
        
        // Track suspension events
        if (FiberStatus.isSuspended(status) && 
            (!lastStatus || !FiberStatus.isSuspended(lastStatus))) {
          yield* Ref.update(this.profiles, (profiles) => {
            const profile = profiles.get(fiberId)
            if (profile) {
              return profiles.set(fiberId, {
                ...profile,
                suspendedCount: profile.suspendedCount + 1
              })
            }
            return profiles
          })
        }
        
        lastCheckTime = currentTime
        lastStatus = status
        return false // Continue monitoring
      })
      
      // Monitor every 100ms for detailed profiling
      yield* Effect.repeatUntil(
        Effect.zipRight(checkPerformance, Effect.sleep("100 millis")),
        (isComplete) => isComplete
      )
    })
  
  getProfiles = () => Ref.get(this.profiles)
  
  generateReport = () =>
    Effect.gen(function* () {
      const profiles = yield* this.getProfiles()
      
      console.log("\n=== Fiber Performance Report ===")
      for (const profile of profiles.values()) {
        const totalLifetime = profile.completedAt 
          ? profile.completedAt.getTime() - profile.createdAt.getTime()
          : Date.now() - profile.createdAt.getTime()
        
        const efficiency = totalLifetime > 0 
          ? (profile.totalRunTime / totalLifetime * 100).toFixed(1)
          : "0.0"
        
        console.log(`Fiber ${profile.fiberId}:`)
        console.log(`  Total lifetime: ${totalLifetime}ms`)
        console.log(`  Actual run time: ${profile.totalRunTime}ms`)
        console.log(`  Suspended time: ${profile.suspendedTime}ms`)
        console.log(`  Suspended count: ${profile.suspendedCount}`)
        console.log(`  Efficiency: ${efficiency}%`)
        console.log(`  Status: ${profile.completedAt ? 'Completed' : 'Running'}`)
        console.log("")
      }
      console.log("===============================\n")
    })
}

// Usage example
const profilingExample = Effect.gen(function* () {
  const profiles = yield* Ref.make(new Map<FiberId.Runtime, FiberProfile>())
  const startTimes = yield* Ref.make(new Map<FiberId.Runtime, Date>())
  const profiler = new FiberProfiler(profiles, startTimes)
  
  // Profile a complex task with I/O operations
  yield* profiler.startProfiling(
    Effect.gen(function* () {
      // Simulate CPU work
      yield* Effect.sleep("100 millis")
      
      // Simulate I/O wait
      yield* Effect.sleep("500 millis")
      
      // More CPU work
      yield* Effect.sleep("200 millis")
      
      return "complex task complete"
    })
  )
  
  // Generate performance report
  yield* profiler.generateReport()
})
```

## Advanced Features Deep Dive

### RuntimeFlags Analysis

FiberStatus provides access to RuntimeFlags, which control fiber behavior:

```typescript
const analyzeRuntimeFlags = (fiber: Fiber.RuntimeFiber<unknown, unknown>) =>
  Effect.gen(function* () {
    const status = yield* Fiber.status(fiber)
    
    if (FiberStatus.isRunning(status) || FiberStatus.isSuspended(status)) {
      const flags = status.runtimeFlags
      
      // You can analyze various runtime flags
      console.log("Runtime flags analysis:")
      console.log(`  Interruption enabled: ${RuntimeFlags.interruption(flags)}`)
      console.log(`  Cooperative yielding: ${RuntimeFlags.cooperativeYielding(flags)}`)
      console.log(`  Runtime metrics: ${RuntimeFlags.runtimeMetrics(flags)}`)
      console.log(`  Wind down: ${RuntimeFlags.windDown(flags)}`)
    }
  })
```

### BlockingOn Analysis for Dependency Tracking

When fibers are suspended, you can analyze their dependencies:

```typescript
const analyzeDependencies = (fiber: Fiber.RuntimeFiber<unknown, unknown>) =>
  Effect.gen(function* () {
    const status = yield* Fiber.status(fiber)
    
    if (FiberStatus.isSuspended(status)) {
      const blockingOn = status.blockingOn
      const blockingIds = FiberId.ids(blockingOn)
      
      console.log(`Fiber is blocked on ${HashSet.size(blockingIds)} other fiber(s):`)
      for (const id of blockingIds) {
        console.log(`  - Fiber ID: ${id}`)
      }
      
      // You can also check if it's a composite or single fiber dependency
      if (FiberId.isComposite(blockingOn)) {
        console.log("This is a composite dependency (multiple fibers)")
      } else {
        console.log("This is a single fiber dependency")
      }
    }
  })
```

## Practical Patterns & Best Practices

### Pattern 1: Fiber Health Checking

```typescript
const createHealthChecker = () => {
  const isHealthy = (fiber: Fiber.RuntimeFiber<unknown, unknown>) =>
    Effect.gen(function* () {
      const status = yield* Fiber.status(fiber)
      
      if (FiberStatus.isDone(status)) {
        return { healthy: true, reason: "completed" }
      }
      
      if (FiberStatus.isRunning(status)) {
        return { healthy: true, reason: "actively running" }
      }
      
      if (FiberStatus.isSuspended(status)) {
        // Check if suspended too long (potential deadlock or performance issue)
        return { 
          healthy: false, 
          reason: `suspended, blocking on: ${status.blockingOn}` 
        }
      }
      
      return { healthy: false, reason: "unknown status" }
    })
  
  const checkFiberHealth = (fibers: Fiber.RuntimeFiber<unknown, unknown>[]) =>
    Effect.gen(function* () {
      const healthChecks = yield* Effect.all(
        fibers.map(fiber => 
          Effect.gen(function* () {
            const health = yield* isHealthy(fiber)
            return { fiberId: fiber.id(), ...health }
          })
        )
      )
      
      const unhealthyFibers = healthChecks.filter(check => !check.healthy)
      
      if (unhealthyFibers.length > 0) {
        console.log("Unhealthy fibers detected:")
        unhealthyFibers.forEach(fiber => 
          console.log(`  Fiber ${fiber.fiberId}: ${fiber.reason}`)
        )
      }
      
      return {
        total: fibers.length,
        healthy: healthChecks.length - unhealthyFibers.length,
        unhealthy: unhealthyFibers.length,
        details: healthChecks
      }
    })
  
  return { isHealthy, checkFiberHealth }
}
```

### Pattern 2: Status-Based Resource Management

```typescript
const createResourceManager = () => {
  const manageResourcesBasedOnStatus = (
    fiber: Fiber.RuntimeFiber<unknown, unknown>,
    resources: { cleanup: () => Effect.Effect<void> }[]
  ) =>
    Effect.gen(function* () {
      const status = yield* Fiber.status(fiber)
      
      if (FiberStatus.isDone(status)) {
        // Fiber completed - clean up all resources
        console.log("Fiber completed, cleaning up resources...")
        yield* Effect.all(resources.map(r => r.cleanup()), { concurrency: "unbounded" })
      } else if (FiberStatus.isSuspended(status)) {
        // Fiber suspended - maybe clean up some resources to free memory
        console.log("Fiber suspended, performing partial cleanup...")
        const nonCriticalResources = resources.slice(0, Math.floor(resources.length / 2))
        yield* Effect.all(nonCriticalResources.map(r => r.cleanup()), { concurrency: "unbounded" })
      }
      // If running, keep all resources allocated
    })
  
  return { manageResourcesBasedOnStatus }
}
```

### Pattern 3: Dynamic Timeout Based on Status

```typescript
const createDynamicTimeout = () => {
  const waitWithDynamicTimeout = <A, E>(
    fiber: Fiber.RuntimeFiber<A, E>
  ) =>
    Effect.gen(function* () {
      const checkAndWait = Effect.gen(function* () {
        const status = yield* Fiber.status(fiber)
        
        if (FiberStatus.isDone(status)) {
          return yield* Fiber.await(fiber)
        }
        
        if (FiberStatus.isRunning(status)) {
          // Give running fibers more time
          yield* Effect.sleep("2 seconds")
          return yield* checkAndWait
        }
        
        if (FiberStatus.isSuspended(status)) {
          // Suspended fibers get less time before we give up
          yield* Effect.sleep("500 millis")
          return yield* checkAndWait
        }
        
        return yield* Effect.fail(new Error("Unknown fiber status"))
      })
      
      // Overall timeout to prevent infinite loops
      return yield* Effect.timeout(checkAndWait, "30 seconds")
    })
  
  return { waitWithDynamicTimeout }
}
```

## Integration Examples

### Integration with Metrics and Observability

```typescript
import { Effect, Fiber, FiberStatus, Metric } from "effect"

const createFiberMetrics = () => {
  const runningFibersGauge = Metric.gauge("fiber_running_count", {
    description: "Number of currently running fibers"
  })
  
  const suspendedFibersGauge = Metric.gauge("fiber_suspended_count", {
    description: "Number of currently suspended fibers"  
  })
  
  const completedFibersCounter = Metric.counter("fiber_completed_total", {
    description: "Total number of completed fibers"
  })
  
  const fiberSuspensionCounter = Metric.counter("fiber_suspensions_total", {
    description: "Total number of fiber suspensions"
  })
  
  const updateMetrics = (fibers: Fiber.RuntimeFiber<unknown, unknown>[]) =>
    Effect.gen(function* () {
      let runningCount = 0
      let suspendedCount = 0
      let completedCount = 0
      
      for (const fiber of fibers) {
        const status = yield* Fiber.status(fiber)
        
        if (FiberStatus.isRunning(status)) {
          runningCount++
        } else if (FiberStatus.isSuspended(status)) {
          suspendedCount++
          yield* Metric.increment(fiberSuspensionCounter)
        } else if (FiberStatus.isDone(status)) {
          completedCount++
          yield* Metric.increment(completedFibersCounter)
        }
      }
      
      yield* Metric.set(runningFibersGauge, runningCount)
      yield* Metric.set(suspendedFibersGauge, suspendedCount)
    })
  
  return { updateMetrics, runningFibersGauge, suspendedFibersGauge, completedFibersCounter }
}

// Usage with periodic metrics collection
const metricsExample = Effect.gen(function* () {
  const metrics = createFiberMetrics()
  const activeFibers: Fiber.RuntimeFiber<unknown, unknown>[] = []
  
  // Start some fibers
  const fiber1 = yield* Effect.fork(Effect.sleep("5 seconds"))
  const fiber2 = yield* Effect.fork(Effect.sleep("3 seconds"))
  activeFibers.push(fiber1, fiber2)
  
  // Collect metrics every second
  yield* Effect.repeatN(
    Effect.zipRight(metrics.updateMetrics(activeFibers), Effect.sleep("1 second")),
    6
  )
})
```

### Integration with Logging and Tracing

```typescript
import { Effect, Fiber, FiberStatus, Logger } from "effect"

const createFiberLogger = () => {
  const logFiberStatusChange = (
    previousStatus: FiberStatus.FiberStatus | null,
    currentStatus: FiberStatus.FiberStatus,
    fiberId: FiberId.Runtime
  ) =>
    Effect.gen(function* () {
      const timestamp = new Date().toISOString()
      
      if (!previousStatus) {
        yield* Logger.info(`[${timestamp}] Fiber ${fiberId} started`)
        return
      }
      
      // Log status transitions
      if (FiberStatus.isDone(currentStatus) && !FiberStatus.isDone(previousStatus)) {
        yield* Logger.info(`[${timestamp}] Fiber ${fiberId} completed`)
      } else if (FiberStatus.isSuspended(currentStatus) && !FiberStatus.isSuspended(previousStatus)) {
        yield* Logger.warn(
          `[${timestamp}] Fiber ${fiberId} suspended, blocking on: ${currentStatus.blockingOn}`
        )
      } else if (FiberStatus.isRunning(currentStatus) && FiberStatus.isSuspended(previousStatus)) {
        yield* Logger.info(`[${timestamp}] Fiber ${fiberId} resumed from suspension`)
      }
    })
  
  const createTrackedFiber = <A, E>(
    name: string,
    effect: Effect.Effect<A, E, unknown>
  ) =>
    Effect.gen(function* () {
      yield* Logger.info(`Starting tracked fiber: ${name}`)
      
      const fiber = yield* Effect.fork(effect)
      const fiberId = fiber.id()
      
      // Monitor status changes in background
      yield* Effect.fork(
        Effect.gen(function* () {
          let previousStatus: FiberStatus.FiberStatus | null = null
          
          const monitorStatus = Effect.gen(function* () {
            const currentStatus = yield* Fiber.status(fiber)
            
            yield* logFiberStatusChange(previousStatus, currentStatus, fiberId)
            previousStatus = currentStatus
            
            return FiberStatus.isDone(currentStatus)
          })
          
          yield* Effect.repeatUntil(
            Effect.zipRight(monitorStatus, Effect.sleep("200 millis")),
            (isDone) => isDone
          )
        })
      )
      
      return fiber
    })
  
  return { logFiberStatusChange, createTrackedFiber }
}

// Usage example with structured logging
const loggingExample = Effect.gen(function* () {
  const logger = createFiberLogger()
  
  const trackedFiber = yield* logger.createTrackedFiber(
    "data-processor",
    Effect.gen(function* () {
      yield* Effect.sleep("1 second")
      yield* Logger.info("Processing data...")
      yield* Effect.sleep("2 seconds")
      return "processing complete"
    })
  )
  
  const result = yield* Fiber.await(trackedFiber)
  yield* Logger.info(`Final result: ${result}`)
}).pipe(
  Effect.provide(Logger.pretty) // Use pretty logging
)
```

### Testing Strategies

```typescript
import { Effect, Fiber, FiberStatus, TestClock } from "effect"

const createFiberStatusTester = () => {
  const testFiberLifecycle = (
    effect: Effect.Effect<unknown, unknown, unknown>
  ) =>
    Effect.gen(function* () {
      const fiber = yield* Effect.fork(effect)
      
      // Test initial status
      const initialStatus = yield* Fiber.status(fiber)
      console.log("Initial status should be Running:", FiberStatus.isRunning(initialStatus))
      
      // Advance test clock to trigger suspensions/completions
      yield* TestClock.adjust("1 second")
      
      const midStatus = yield* Fiber.status(fiber)
      console.log("Mid-execution status:", {
        running: FiberStatus.isRunning(midStatus),
        suspended: FiberStatus.isSuspended(midStatus),
        done: FiberStatus.isDone(midStatus)
      })
      
      // Complete the effect
      yield* TestClock.adjust("10 seconds")
      
      const finalStatus = yield* Fiber.status(fiber)
      console.log("Final status should be Done:", FiberStatus.isDone(finalStatus))
      
      return yield* Fiber.await(fiber)
    })
  
  const testSuspendedFiberDetails = Effect.gen(function* () {
    const blocker = yield* Effect.fork(Effect.never)
    const waiter = yield* Effect.fork(Fiber.await(blocker))
    
    // Wait for waiter to suspend
    yield* Effect.sleep("100 millis")
    
    const waiterStatus = yield* Fiber.status(waiter)
    
    if (FiberStatus.isSuspended(waiterStatus)) {
      const blockingOnIds = FiberId.ids(waiterStatus.blockingOn)
      const blockerIdSet = FiberId.ids(FiberId.runtime(blocker.id().id, blocker.id().startTimeMillis))
      
      console.log("Blocking relationship verified:", 
        HashSet.size(HashSet.intersection(blockingOnIds, blockerIdSet)) > 0
      )
    }
    
    // Clean up
    yield* Fiber.interrupt(blocker)
    yield* Fiber.interrupt(waiter)
  })
  
  return { testFiberLifecycle, testSuspendedFiberDetails }
}

// Test runner
const runFiberStatusTests = Effect.gen(function* () {
  const tester = createFiberStatusTester()
  
  console.log("=== Testing Fiber Status Lifecycle ===")
  yield* tester.testFiberLifecycle(
    Effect.gen(function* () {
      yield* Effect.sleep("2 seconds")
      return "test complete"
    })
  )
  
  console.log("\n=== Testing Suspended Fiber Details ===")
  yield* tester.testSuspendedFiberDetails
}).pipe(
  Effect.provide(TestClock.make()) // Provide test clock for deterministic testing
)
```

## Conclusion

FiberStatus provides comprehensive visibility into fiber execution states, enabling sophisticated monitoring, debugging, and optimization of concurrent Effect applications.

Key benefits:
- **Rich Observability**: Deep insight into fiber lifecycle and execution state
- **Debugging Power**: Ability to identify deadlocks, performance bottlenecks, and blocking relationships
- **Monitoring Integration**: Easy integration with metrics, logging, and APM systems

FiberStatus is essential when building production Effect applications that require observability, debugging capabilities, or performance optimization of concurrent operations.