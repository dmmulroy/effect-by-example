# Scheduler: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem Scheduler Solves

In traditional JavaScript applications, task scheduling is often ad-hoc and unpredictable:

```typescript
// Traditional approach - uncontrolled scheduling
setTimeout(() => {
  // High priority task mixed with low priority
  updateUI()
  processLargeDataSet() // Blocks UI
  logAnalytics()
}, 0)

// No priority control, no yielding mechanism
setInterval(() => {
  heavyComputation() // Can starve other tasks
}, 100)

// Manual Promise.resolve() scheduling
function scheduleTask(task: () => void) {
  Promise.resolve().then(task) // No priority, no control
}
```

This approach leads to:
- **Poor Performance** - Tasks can starve the main thread
- **No Priority Control** - Critical tasks compete with background work
- **Unpredictable Execution** - No guarantees about when tasks run
- **Resource Starvation** - Heavy tasks can block user interactions

### The Scheduler Solution

Effect's Scheduler provides a sophisticated task scheduling system with priority-based execution and automatic yielding:

```typescript
import { Effect, Scheduler } from "effect"

// Controlled scheduling with priorities
const scheduler = new Scheduler.MixedScheduler(2048)

// Schedule high-priority UI updates
scheduler.scheduleTask(() => updateUI(), 10)

// Schedule low-priority background work
scheduler.scheduleTask(() => processData(), 1)

// Automatic yielding prevents blocking
const computation = Effect.gen(function* () {
  for (let i = 0; i < 10000; i++) {
    yield* Effect.sync(() => doWork(i))
    // Scheduler automatically yields when needed
  }
})
```

### Key Concepts

**Task**: A unit of work represented as `() => void` function that can be scheduled for execution

**Priority**: Numeric value determining execution order (higher numbers = higher priority)

**Yielding**: Automatic mechanism that prevents long-running tasks from blocking the runtime

**Scheduler Types**: Different scheduling strategies for various use cases (sync, async, controlled)

## Basic Usage Patterns

### Pattern 1: Using the Default Scheduler

```typescript
import { Effect, Scheduler } from "effect"

// Default scheduler handles most use cases
const simpleTask = Effect.gen(function* () {
  yield* Effect.log("Starting background task")
  yield* Effect.sleep("100 millis")
  yield* Effect.log("Task completed")
})

// Runs with default scheduler
const result = yield* simpleTask
```

### Pattern 2: Custom Scheduler Configuration

```typescript
import { Effect, Scheduler } from "effect"

// Create a custom scheduler with specific yielding behavior
const customScheduler = new Scheduler.MixedScheduler(1024) // Yield after 1024 operations

const taskWithCustomScheduler = Effect.gen(function* () {
  yield* Effect.log("Using custom scheduler")
  // Heavy computation that will yield appropriately
  for (let i = 0; i < 5000; i++) {
    yield* Effect.sync(() => compute(i))
  }
}).pipe(
  Effect.withScheduler(customScheduler)
)
```

### Pattern 3: Priority-Based Task Scheduling

```typescript
import { Effect, Scheduler } from "effect"

const priorityScheduler = new Scheduler.MixedScheduler(2048)

// High priority user-facing task
const uiTask = Effect.sync(() => {
  console.log("Updating UI - High Priority")
  updateUserInterface()
})

// Low priority background task
const backgroundTask = Effect.sync(() => {
  console.log("Processing data - Low Priority")
  processBackgroundData()
})

// Schedule with different priorities
const scheduledExecution = Effect.gen(function* () {
  // Schedule high priority task (priority 10)
  yield* uiTask.pipe(Effect.withScheduler(priorityScheduler))
  
  // Schedule low priority task (priority 1)  
  yield* backgroundTask.pipe(Effect.withScheduler(priorityScheduler))
})
```

## Real-World Examples

### Example 1: Background Task Processing System

A real-world system that processes user uploads while maintaining UI responsiveness:

```typescript
import { Effect, Scheduler, Queue, Duration } from "effect"

interface UploadTask {
  readonly id: string
  readonly file: File
  readonly priority: number
}

interface ProcessingResult {
  readonly taskId: string
  readonly success: boolean
  readonly processingTime: number
}

// Background task processor with priority scheduling
const makeTaskProcessor = Effect.gen(function* () {
  const taskQueue = yield* Queue.bounded<UploadTask>(1000)
  const scheduler = new Scheduler.MixedScheduler(512) // Yield frequently for UI responsiveness
  
  const processTask = (task: UploadTask): Effect.Effect<ProcessingResult> =>
    Effect.gen(function* () {
      const startTime = yield* Effect.clockWith(clock => clock.currentTimeMillis)
      
      yield* Effect.log(`Processing task ${task.id} with priority ${task.priority}`)
      
      // Simulate file processing work
      yield* Effect.sleep(Duration.millis(100 * task.priority))
      
      const endTime = yield* Effect.clockWith(clock => clock.currentTimeMillis)
      
      return {
        taskId: task.id,
        success: true,
        processingTime: endTime - startTime
      }
    }).pipe(
      Effect.withScheduler(scheduler)
    )
  
  const worker = Effect.gen(function* () {
    while (true) {
      const task = yield* Queue.take(taskQueue)
      const result = yield* processTask(task)
      yield* Effect.log(`Task ${result.taskId} completed in ${result.processingTime}ms`)
    }
  }).pipe(
    Effect.fork
  )
  
  const addTask = (task: UploadTask) =>
    Queue.offer(taskQueue, task).pipe(
      Effect.tap(() => Effect.log(`Queued task ${task.id}`))
    )
  
  return {
    addTask,
    start: worker
  } as const
})

// Usage example
const uploadProcessor = Effect.gen(function* () {
  const processor = yield* makeTaskProcessor
  const workerFiber = yield* processor.start
  
  // Add high priority user avatar upload
  yield* processor.addTask({
    id: "avatar_001",
    file: new File([], "avatar.jpg"),
    priority: 10
  })
  
  // Add low priority batch document uploads
  for (let i = 0; i < 5; i++) {
    yield* processor.addTask({
      id: `doc_${i}`,
      file: new File([], `document${i}.pdf`),
      priority: 2
    })
  }
  
  // Let it run for a while
  yield* Effect.sleep(Duration.seconds(5))
  yield* workerFiber.interrupt
})
```

### Example 2: Real-Time Analytics Processing

An analytics system that processes events with different priorities and batching:

```typescript
import { Effect, Scheduler, Queue, Duration, Ref } from "effect"

interface AnalyticsEvent {
  readonly type: "click" | "pageview" | "error" | "custom"
  readonly data: Record<string, unknown>
  readonly timestamp: number
  readonly priority: number
}

interface EventBatch {
  readonly events: ReadonlyArray<AnalyticsEvent>
  readonly batchId: string
}

const makeAnalyticsProcessor = Effect.gen(function* () {
  const eventQueue = yield* Queue.bounded<AnalyticsEvent>(10000)
  const batchRef = yield* Ref.make<AnalyticsEvent[]>([])
  
  // High-frequency scheduler for real-time events
  const realtimeScheduler = new Scheduler.MixedScheduler(4096)
  
  // Batch scheduler for efficient processing
  const batchScheduler = Scheduler.makeBatched((runBatch) => {
    setTimeout(runBatch, 1000) // Batch every second
  })
  
  const processEvent = (event: AnalyticsEvent): Effect.Effect<void> =>
    Effect.gen(function* () {
      switch (event.type) {
        case "error":
          // Critical errors need immediate processing
          yield* Effect.log(`CRITICAL: ${JSON.stringify(event.data)}`)
          yield* sendToErrorTracking(event)
          break
        
        case "click":
        case "pageview":
          // User interactions are high priority
          yield* Effect.log(`User interaction: ${event.type}`)
          yield* updateUserSessionData(event)
          break
        
        case "custom":
          // Custom events are lower priority
          yield* Effect.log(`Custom event: ${JSON.stringify(event.data)}`)
          yield* addToAnalyticsBatch(event)
          break
      }
    }).pipe(
      Effect.withScheduler(
        event.priority > 5 ? realtimeScheduler : batchScheduler
      )
    )
  
  const batchProcessor = Effect.gen(function* () {
    while (true) {
      yield* Effect.sleep(Duration.millis(1000))
      
      const events = yield* Ref.getAndSet(batchRef, [])
      
      if (events.length > 0) {
        const batch: EventBatch = {
          events,
          batchId: `batch_${Date.now()}`
        }
        
        yield* Effect.log(`Processing batch ${batch.batchId} with ${events.length} events`)
        yield* sendBatchToAnalytics(batch)
      }
    }
  }).pipe(
    Effect.withScheduler(batchScheduler),
    Effect.fork
  )
  
  const eventProcessor = Effect.gen(function* () {
    while (true) {
      const event = yield* Queue.take(eventQueue)
      yield* processEvent(event)
    }
  }).pipe(
    Effect.fork
  )
  
  const trackEvent = (event: Omit<AnalyticsEvent, "timestamp">) =>
    Queue.offer(eventQueue, {
      ...event,
      timestamp: Date.now()
    })
  
  return {
    trackEvent,
    start: Effect.all([eventProcessor, batchProcessor])
  } as const
})

// Helper functions
const sendToErrorTracking = (event: AnalyticsEvent) =>
  Effect.log(`Sending error to tracking service: ${event.type}`)

const updateUserSessionData = (event: AnalyticsEvent) =>
  Effect.log(`Updating user session for: ${event.type}`)

const addToAnalyticsBatch = (event: AnalyticsEvent) =>
  Effect.log(`Adding to analytics batch: ${event.type}`)

const sendBatchToAnalytics = (batch: EventBatch) =>
  Effect.log(`Sending batch ${batch.batchId} to analytics service`)

// Usage example
const analyticsExample = Effect.gen(function* () {
  const processor = yield* makeAnalyticsProcessor
  const processors = yield* processor.start
  
  // Track high-priority error event
  yield* processor.trackEvent({
    type: "error",
    data: { message: "API timeout", endpoint: "/api/users" },
    priority: 10
  })
  
  // Track user interactions
  yield* processor.trackEvent({
    type: "click",
    data: { buttonId: "subscribe", page: "pricing" },
    priority: 7
  })
  
  // Track low-priority custom events
  for (let i = 0; i < 10; i++) {
    yield* processor.trackEvent({
      type: "custom",
      data: { action: "feature_usage", feature: `feature_${i}` },
      priority: 2
    })
  }
  
  yield* Effect.sleep(Duration.seconds(5))
  yield* processors.interrupt
})
```

### Example 3: Distributed Task Scheduling System

A job scheduler that distributes work across multiple workers with different scheduling strategies:

```typescript
import { Effect, Scheduler, Queue, Duration, Ref, Fiber } from "effect"

interface Job {
  readonly id: string
  readonly type: "cpu_intensive" | "io_bound" | "realtime"
  readonly payload: Record<string, unknown>
  readonly priority: number
  readonly maxRetries: number
}

interface Worker {
  readonly id: string
  readonly type: Job["type"]
  readonly scheduler: Scheduler.Scheduler
}

interface JobResult {
  readonly jobId: string
  readonly workerId: string
  readonly success: boolean
  readonly duration: number
  readonly error?: string
}

const makeJobScheduler = Effect.gen(function* () {
  const jobQueue = yield* Queue.bounded<Job>(5000)
  const resultQueue = yield* Queue.bounded<JobResult>(1000)
  const activeJobs = yield* Ref.make<Map<string, Fiber.RuntimeFiber<JobResult, never>>>(new Map())
  
  // Different schedulers for different job types
  const cpuScheduler = new Scheduler.MixedScheduler(1024) // Frequent yielding for CPU work
  const ioScheduler = new Scheduler.MixedScheduler(4096)  // Less frequent yielding for I/O
  const realtimeScheduler = new Scheduler.SyncScheduler() // Synchronous for real-time work
  
  const workers: Worker[] = [
    { id: "cpu_worker_1", type: "cpu_intensive", scheduler: cpuScheduler },
    { id: "cpu_worker_2", type: "cpu_intensive", scheduler: cpuScheduler },
    { id: "io_worker_1", type: "io_bound", scheduler: ioScheduler },
    { id: "io_worker_2", type: "io_bound", scheduler: ioScheduler },
    { id: "realtime_worker_1", type: "realtime", scheduler: realtimeScheduler }
  ]
  
  const executeJob = (job: Job, worker: Worker): Effect.Effect<JobResult> =>
    Effect.gen(function* () {
      const startTime = yield* Effect.clockWith(clock => clock.currentTimeMillis)
      
      yield* Effect.log(`Worker ${worker.id} starting job ${job.id}`)
      
      try {
        switch (job.type) {
          case "cpu_intensive":
            yield* simulateCpuIntensiveWork(job.payload)
            break
          case "io_bound":
            yield* simulateIoBoundWork(job.payload)
            break
          case "realtime":
            yield* simulateRealtimeWork(job.payload)
            break
        }
        
        const endTime = yield* Effect.clockWith(clock => clock.currentTimeMillis)
        
        return {
          jobId: job.id,
          workerId: worker.id,
          success: true,
          duration: endTime - startTime
        }
      } catch (error) {
        const endTime = yield* Effect.clockWith(clock => clock.currentTimeMillis)
        
        return {
          jobId: job.id,
          workerId: worker.id,
          success: false,
          duration: endTime - startTime,
          error: String(error)
        }
      }
    }).pipe(
      Effect.withScheduler(worker.scheduler)
    )
  
  const createWorker = (worker: Worker) =>
    Effect.gen(function* () {
      while (true) {
        const job = yield* Queue.take(jobQueue).pipe(
          Effect.timeout(Duration.seconds(1)),
          Effect.catchTag("TimeoutException", () => Effect.fail("No jobs available"))
        )
        
        if (job._tag === "Some" && job.value.type === worker.type) {
          const jobFiber = yield* executeJob(job.value, worker).pipe(
            Effect.fork
          )
          
          yield* Ref.update(activeJobs, map => map.set(job.value.id, jobFiber))
          
          const result = yield* Fiber.join(jobFiber)
          yield* Queue.offer(resultQueue, result)
          
          yield* Ref.update(activeJobs, map => {
            map.delete(job.value.id)
            return map
          })
        }
      }
    }).pipe(
      Effect.catchAll(() => Effect.void),
      Effect.fork
    )
  
  const startWorkers = Effect.all(workers.map(createWorker))
  
  const submitJob = (job: Job) =>
    Queue.offer(jobQueue, job).pipe(
      Effect.tap(() => Effect.log(`Submitted job ${job.id} of type ${job.type}`))
    )
  
  const getResult = Queue.take(resultQueue)
  
  const getActiveJobCount = Ref.get(activeJobs).pipe(
    Effect.map(map => map.size)
  )
  
  return {
    submitJob,
    getResult,
    getActiveJobCount,
    start: startWorkers
  } as const
})

// Job simulation functions
const simulateCpuIntensiveWork = (payload: Record<string, unknown>) =>
  Effect.gen(function* () {
    // Simulate heavy computation
    for (let i = 0; i < 1000; i++) {
      yield* Effect.sync(() => {
        // CPU-intensive work that yields regularly
        Math.random() * Math.random()
      })
    }
    yield* Effect.log(`Completed CPU-intensive work: ${JSON.stringify(payload)}`)
  })

const simulateIoBoundWork = (payload: Record<string, unknown>) =>
  Effect.gen(function* () {
    // Simulate I/O operations
    yield* Effect.sleep(Duration.millis(100 + Math.random() * 200))
    yield* Effect.log(`Completed I/O work: ${JSON.stringify(payload)}`)
  })

const simulateRealtimeWork = (payload: Record<string, unknown>) =>
  Effect.gen(function* () {
    // Simulate real-time processing
    yield* Effect.log(`Processing real-time data: ${JSON.stringify(payload)}`)
    yield* Effect.sleep(Duration.millis(10))
  })

// Usage example
const jobSchedulerExample = Effect.gen(function* () {
  const scheduler = yield* makeJobScheduler
  const workers = yield* scheduler.start
  
  // Submit different types of jobs
  yield* scheduler.submitJob({
    id: "cpu_job_1",
    type: "cpu_intensive",
    payload: { dataset: "large_numbers", size: 10000 },
    priority: 5,
    maxRetries: 3
  })
  
  yield* scheduler.submitJob({
    id: "io_job_1",
    type: "io_bound",
    payload: { url: "https://api.example.com/data", timeout: 5000 },
    priority: 7,
    maxRetries: 2
  })
  
  yield* scheduler.submitJob({
    id: "realtime_job_1",
    type: "realtime",
    payload: { stream: "user_events", window: "1m" },
    priority: 10,
    maxRetries: 1
  })
  
  // Process results
  const resultProcessor = Effect.gen(function* () {
    for (let i = 0; i < 3; i++) {
      const result = yield* scheduler.getResult
      yield* Effect.log(`Job ${result.jobId} completed by ${result.workerId}: ${result.success ? 'SUCCESS' : 'FAILED'} (${result.duration}ms)`)
    }
  })
  
  yield* resultProcessor
  yield* workers.interrupt
})
```

## Advanced Features Deep Dive

### Feature 1: Custom Scheduler Implementation

Creating specialized schedulers for specific use cases:

#### Basic Custom Scheduler

```typescript
import { Effect, Scheduler } from "effect"

// Time-based priority scheduler
const makeTimeBasedScheduler = (timeSliceMs: number = 16) => {
  let lastYieldTime = Date.now()
  const tasks = new Scheduler.PriorityBuckets<Scheduler.Task>()
  let running = false
  
  const shouldYield = (fiber: any): number | false => {
    const now = Date.now()
    const elapsed = now - lastYieldTime
    
    if (elapsed > timeSliceMs) {
      lastYieldTime = now
      return fiber.getFiberRef(/* currentSchedulingPriority */)
    }
    
    return false
  }
  
  const scheduleTask = (task: Scheduler.Task, priority: number) => {
    tasks.scheduleTask(task, priority)
    
    if (!running) {
      running = true
      requestAnimationFrame(() => {
        const taskBuckets = tasks.buckets
        tasks.buckets = []
        
        for (const [_, toRun] of taskBuckets) {
          for (const task of toRun) {
            task()
          }
        }
        
        running = false
      })
    }
  }
  
  return Scheduler.make(scheduleTask, shouldYield)
}

// Usage with custom scheduler
const animationWorkflow = Effect.gen(function* () {
  const customScheduler = makeTimeBasedScheduler(16) // 60 FPS
  
  // Animation loop that respects frame timing
  for (let frame = 0; frame < 100; frame++) {
    yield* Effect.sync(() => {
      updateAnimation(frame)
      renderFrame(frame)
    })
  }
}).pipe(
  Effect.withScheduler(makeTimeBasedScheduler())
)

const updateAnimation = (frame: number) => {
  console.log(`Updating animation frame ${frame}`)
}

const renderFrame = (frame: number) => {
  console.log(`Rendering frame ${frame}`)
}
```

#### Advanced Custom Scheduler: Adaptive Priority

```typescript
import { Effect, Scheduler, Duration, Ref } from "effect"

interface AdaptiveConfig {
  readonly baseTimeSlice: number
  readonly loadThreshold: number
  readonly adaptationFactor: number
}

const makeAdaptiveScheduler = (config: AdaptiveConfig) =>
  Effect.gen(function* () {
    const loadMetrics = yield* Ref.make({
      averageTaskTime: 0,
      taskCount: 0,
      currentLoad: 0
    })
    
    const adaptiveTimeSlice = yield* Ref.make(config.baseTimeSlice)
    
    const updateMetrics = (taskTime: number) =>
      Ref.update(loadMetrics, metrics => {
        const newTaskCount = metrics.taskCount + 1
        const newAverageTime = (metrics.averageTaskTime * metrics.taskCount + taskTime) / newTaskCount
        const currentLoad = taskTime > config.loadThreshold ? metrics.currentLoad + 1 : Math.max(0, metrics.currentLoad - 1)
        
        return {
          averageTaskTime: newAverageTime,
          taskCount: newTaskCount,
          currentLoad
        }
      })
    
    const adaptScheduling = Effect.gen(function* () {
      const metrics = yield* Ref.get(loadMetrics)
      const currentSlice = yield* Ref.get(adaptiveTimeSlice)
      
      if (metrics.currentLoad > 5) {
        // High load: reduce time slice for more frequent yielding
        const newSlice = Math.max(1, currentSlice * config.adaptationFactor)
        yield* Ref.set(adaptiveTimeSlice, newSlice)
      } else if (metrics.currentLoad < 2) {
        // Low load: increase time slice for better performance
        const newSlice = Math.min(config.baseTimeSlice * 2, currentSlice / config.adaptationFactor)
        yield* Ref.set(adaptiveTimeSlice, newSlice)
      }
    })
    
    // Adapt scheduling every second
    const adaptationLoop = Effect.gen(function* () {
      while (true) {
        yield* Effect.sleep(Duration.seconds(1))
        yield* adaptScheduling
      }
    }).pipe(Effect.fork)
    
    const tasks = new Scheduler.PriorityBuckets<() => void>()
    let running = false
    let lastYieldTime = Date.now()
    
    const shouldYield = (fiber: any): number | false => {
      return Effect.runSync(Effect.gen(function* () {
        const timeSlice = yield* Ref.get(adaptiveTimeSlice)
        const now = Date.now()
        const elapsed = now - lastYieldTime
        
        if (elapsed > timeSlice) {
          lastYieldTime = now
          return fiber.getFiberRef(/* currentSchedulingPriority */)
        }
        
        return false
      }))
    }
    
    const scheduleTask = (task: Scheduler.Task, priority: number) => {
      const wrappedTask = () => {
        const startTime = Date.now()
        task()
        const endTime = Date.now()
        
        Effect.runSync(updateMetrics(endTime - startTime))
      }
      
      tasks.scheduleTask(wrappedTask, priority)
      
      if (!running) {
        running = true
        setTimeout(() => {
          const taskBuckets = tasks.buckets
          tasks.buckets = []
          
          for (const [_, toRun] of taskBuckets) {
            for (const task of toRun) {
              task()
            }
          }
          
          running = false
        }, 0)
      }
    }
    
    yield* adaptationLoop
    
    return Scheduler.make(scheduleTask, shouldYield)
  })
```

### Feature 2: Scheduler Composition and Matrix

Creating complex scheduling strategies by combining multiple schedulers:

```typescript
import { Effect, Scheduler } from "effect"

// Multi-tier scheduler with different strategies per priority range
const makeMultiTierScheduler = () => {
  // High priority: immediate execution
  const highPriorityScheduler = new Scheduler.SyncScheduler()
  
  // Medium priority: batched execution
  const mediumPriorityScheduler = Scheduler.makeBatched((runBatch) => {
    setTimeout(runBatch, 10) // 10ms batching
  })
  
  // Low priority: throttled execution
  const lowPriorityScheduler = Scheduler.timer(100) // 100ms delay
  
  // Create matrix scheduler
  return Scheduler.makeMatrix(
    [8, highPriorityScheduler],   // Priority 8+ -> immediate
    [4, mediumPriorityScheduler], // Priority 4-7 -> batched
    [0, lowPriorityScheduler]     // Priority 0-3 -> throttled
  )
}

// Usage example with multi-tier scheduling
const multiTierExample = Effect.gen(function* () {
  const scheduler = makeMultiTierScheduler()
  
  // Critical system health check (priority 10)
  const healthCheck = Effect.gen(function* () {
    yield* Effect.log("Performing critical health check")
    yield* Effect.sleep(Duration.millis(50))
    return "healthy"
  }).pipe(
    Effect.withScheduler(scheduler)
  )
  
  // User interaction handling (priority 6)
  const handleUserClick = Effect.gen(function* () {
    yield* Effect.log("Processing user click")
    yield* Effect.sleep(Duration.millis(25))
    return "click_processed"
  }).pipe(
    Effect.withScheduler(scheduler)
  )
  
  // Background analytics (priority 2)
  const logAnalytics = Effect.gen(function* () {
    yield* Effect.log("Logging analytics data")
    yield* Effect.sleep(Duration.millis(10))
    return "logged"
  }).pipe(
    Effect.withScheduler(scheduler)
  )
  
  // Execute all tasks - observe scheduling differences
  const results = yield* Effect.all([
    healthCheck,
    handleUserClick,
    logAnalytics
  ])
  
  yield* Effect.log(`Results: ${JSON.stringify(results)}`)
})
```

### Feature 3: Scheduler Testing and Debugging

Tools and patterns for testing scheduler behavior:

```typescript
import { Effect, Scheduler, Duration, Ref, TestClock } from "effect"

// Controlled scheduler for testing
const makeTestScheduler = () => {
  const executionLog = [] as Array<{ task: string; priority: number; timestamp: number }>
  
  const scheduleTask = (task: Scheduler.Task, priority: number) => {
    const taskName = task.name || "anonymous"
    const entry = {
      task: taskName,
      priority,
      timestamp: Date.now()
    }
    
    executionLog.push(entry)
    task()
  }
  
  const getExecutionLog = () => [...executionLog]
  const clearLog = () => { executionLog.length = 0 }
  
  return {
    scheduler: Scheduler.make(scheduleTask),
    getExecutionLog,
    clearLog
  }
}

// Testing scheduler behavior
const testSchedulerBehavior = Effect.gen(function* () {
  const testScheduler = makeTestScheduler()
  
  // Create named tasks for testing
  const createNamedTask = (name: string, duration: number) => {
    const task = () => {
      console.log(`Executing ${name}`)
      // Simulate work
      const start = Date.now()
      while (Date.now() - start < duration) {
        // Busy wait
      }
    }
    Object.defineProperty(task, 'name', { value: name })
    return task
  }
  
  // Schedule tasks with different priorities
  testScheduler.scheduler.scheduleTask(createNamedTask("lowPriority", 10), 1)
  testScheduler.scheduler.scheduleTask(createNamedTask("highPriority", 10), 10)
  testScheduler.scheduler.scheduleTask(createNamedTask("mediumPriority", 10), 5)
  
  yield* Effect.sleep(Duration.millis(100))
  
  const log = testScheduler.getExecutionLog()
  yield* Effect.log("Execution order:")
  log.forEach((entry, index) => {
    console.log(`${index + 1}. ${entry.task} (priority: ${entry.priority})`)
  })
  
  // Verify priority ordering
  const priorities = log.map(entry => entry.priority)
  const expectedOrder = [10, 5, 1] // High to low priority
  const isCorrectOrder = priorities.every((priority, index) => priority === expectedOrder[index])
  
  yield* Effect.log(`Priority ordering correct: ${isCorrectOrder}`)
})

// Performance testing scheduler
const benchmarkScheduler = (scheduler: Scheduler.Scheduler, taskCount: number) =>
  Effect.gen(function* () {
    const startTime = yield* Effect.clockWith(clock => clock.currentTimeMillis)
    const completedTasks = yield* Ref.make(0)
    
    // Create benchmark tasks
    const tasks = Array.from({ length: taskCount }, (_, i) =>
      Effect.gen(function* () {
        // Small amount of work
        yield* Effect.sync(() => Math.random() * Math.random())
        yield* Ref.update(completedTasks, n => n + 1)
      }).pipe(
        Effect.withScheduler(scheduler)
      )
    )
    
    // Execute all tasks
    yield* Effect.all(tasks, { concurrency: "unbounded" })
    
    const endTime = yield* Effect.clockWith(clock => clock.currentTimeMillis)
    const completed = yield* Ref.get(completedTasks)
    
    return {
      totalTime: endTime - startTime,
      tasksCompleted: completed,
      tasksPerSecond: (completed / (endTime - startTime)) * 1000
    }
  })

// Compare scheduler performance
const schedulerComparison = Effect.gen(function* () {
  const taskCount = 1000
  
  const defaultBench = yield* benchmarkScheduler(Scheduler.defaultScheduler, taskCount)
  const syncBench = yield* benchmarkScheduler(new Scheduler.SyncScheduler(), taskCount)
  const mixedBench = yield* benchmarkScheduler(new Scheduler.MixedScheduler(1024), taskCount)
  
  yield* Effect.log("Scheduler Performance Comparison:")
  yield* Effect.log(`Default: ${defaultBench.tasksPerSecond.toFixed(2)} tasks/sec`)
  yield* Effect.log(`Sync: ${syncBench.tasksPerSecond.toFixed(2)} tasks/sec`)
  yield* Effect.log(`Mixed: ${mixedBench.tasksPerSecond.toFixed(2)} tasks/sec`)
})
```

## Practical Patterns & Best Practices

### Pattern 1: Scheduler Selection Strategy

```typescript
import { Effect, Scheduler } from "effect"

// Helper function to choose appropriate scheduler based on task characteristics
const selectScheduler = (taskType: {
  readonly type: "cpu" | "io" | "ui" | "background"
  readonly urgency: "critical" | "high" | "medium" | "low"
  readonly duration: "short" | "medium" | "long"
}): Scheduler.Scheduler => {
  // Critical tasks get immediate execution
  if (taskType.urgency === "critical") {
    return new Scheduler.SyncScheduler()
  }
  
  // UI tasks need frequent yielding for responsiveness
  if (taskType.type === "ui") {
    return new Scheduler.MixedScheduler(512) // Yield every 512 operations
  }
  
  // Long-running CPU tasks need aggressive yielding
  if (taskType.type === "cpu" && taskType.duration === "long") {
    return new Scheduler.MixedScheduler(256) // Yield every 256 operations
  }
  
  // I/O tasks can yield less frequently
  if (taskType.type === "io") {
    return new Scheduler.MixedScheduler(2048)
  }
  
  // Background tasks can be batched
  if (taskType.type === "background") {
    return Scheduler.makeBatched((runBatch) => {
      setTimeout(runBatch, 100) // Batch every 100ms
    })
  }
  
  // Default scheduler for everything else
  return Scheduler.defaultScheduler
}

// Smart task executor with automatic scheduler selection
const smartExecute = <A, E, R>(
  effect: Effect.Effect<A, E, R>,
  taskCharacteristics: Parameters<typeof selectScheduler>[0]
): Effect.Effect<A, E, R> => {
  const scheduler = selectScheduler(taskCharacteristics)
  return effect.pipe(Effect.withScheduler(scheduler))
}

// Usage examples
const taskExamples = Effect.gen(function* () {
  // Critical system operation
  yield* smartExecute(
    Effect.log("Critical system check"),
    { type: "cpu", urgency: "critical", duration: "short" }
  )
  
  // UI update
  yield* smartExecute(
    Effect.log("Updating user interface"),
    { type: "ui", urgency: "high", duration: "short" }
  )
  
  // Long-running computation
  yield* smartExecute(
    Effect.log("Processing large dataset"),
    { type: "cpu", urgency: "medium", duration: "long" }
  )
  
  // Background analytics
  yield* smartExecute(
    Effect.log("Sending analytics data"),
    { type: "background", urgency: "low", duration: "medium" }
  )
})
```

### Pattern 2: Scheduler Pool Management

```typescript
import { Effect, Scheduler, Pool, Duration } from "effect"

interface SchedulerPool {
  readonly acquire: Effect.Effect<Scheduler.Scheduler>
  readonly release: (scheduler: Scheduler.Scheduler) => Effect.Effect<void>
}

// Pool of schedulers for different workload types
const makeSchedulerPool = (config: {
  readonly maxSchedulers: number
  readonly schedulerType: "mixed" | "sync" | "batched"
}): Effect.Effect<SchedulerPool> =>
  Effect.gen(function* () {
    const createScheduler = (): Scheduler.Scheduler => {
      switch (config.schedulerType) {
        case "mixed":
          return new Scheduler.MixedScheduler(2048)
        case "sync":
          return new Scheduler.SyncScheduler()
        case "batched":
          return Scheduler.makeBatched((runBatch) => {
            setTimeout(runBatch, 50)
          })
      }
    }
    
    const pool = yield* Pool.make({
      acquire: Effect.sync(createScheduler),
      size: config.maxSchedulers
    })
    
    return {
      acquire: Pool.get(pool),
      release: (scheduler) => Pool.invalidate(pool, scheduler)
    }
  })

// Task executor with scheduler pooling
const executeWithPooledScheduler = <A, E, R>(
  effect: Effect.Effect<A, E, R>,
  pool: SchedulerPool
): Effect.Effect<A, E, R> =>
  Effect.gen(function* () {
    const scheduler = yield* pool.acquire
    
    try {
      return yield* effect.pipe(Effect.withScheduler(scheduler))
    } finally {
      yield* pool.release(scheduler)
    }
  })

// Usage example with scheduler pools
const schedulerPoolExample = Effect.gen(function* () {
  const mixedPool = yield* makeSchedulerPool({
    maxSchedulers: 5,
    schedulerType: "mixed"
  })
  
  const batchedPool = yield* makeSchedulerPool({
    maxSchedulers: 3,
    schedulerType: "batched"
  })
  
  // Execute CPU-intensive task with mixed scheduler
  const cpuTask = Effect.gen(function* () {
    for (let i = 0; i < 1000; i++) {
      yield* Effect.sync(() => Math.random() * Math.random())
    }
    return "CPU task completed"
  })
  
  // Execute I/O task with batched scheduler
  const ioTask = Effect.gen(function* () {
    yield* Effect.sleep(Duration.millis(100))
    return "I/O task completed"
  })
  
  const results = yield* Effect.all([
    executeWithPooledScheduler(cpuTask, mixedPool),
    executeWithPooledScheduler(ioTask, batchedPool)
  ])
  
  yield* Effect.log(`Results: ${JSON.stringify(results)}`)
})
```

### Pattern 3: Monitoring and Metrics

```typescript
import { Effect, Scheduler, Ref, Duration, Metric } from "effect"

interface SchedulerMetrics {
  readonly tasksScheduled: number
  readonly tasksCompleted: number
  readonly averageExecutionTime: number
  readonly yieldCount: number
  readonly lastYieldTime: number
}

// Instrumented scheduler with metrics collection
const makeInstrumentedScheduler = (baseScheduler: Scheduler.Scheduler) =>
  Effect.gen(function* () {
    const metrics = yield* Ref.make<SchedulerMetrics>({
      tasksScheduled: 0,
      tasksCompleted: 0,
      averageExecutionTime: 0,
      yieldCount: 0,
      lastYieldTime: 0
    })
    
    const scheduleTask = (task: Scheduler.Task, priority: number) => {
      const wrappedTask = () => {
        const startTime = Date.now()
        
        Effect.runSync(Ref.update(metrics, m => ({
          ...m,
          tasksScheduled: m.tasksScheduled + 1
        })))
        
        task()
        
        const endTime = Date.now()
        const executionTime = endTime - startTime
        
        Effect.runSync(Ref.update(metrics, m => ({
          ...m,
          tasksCompleted: m.tasksCompleted + 1,
          averageExecutionTime: (m.averageExecutionTime * (m.tasksCompleted - 1) + executionTime) / m.tasksCompleted
        })))
      }
      
      baseScheduler.scheduleTask(wrappedTask, priority)
    }
    
    const shouldYield = (fiber: any) => {
      const result = baseScheduler.shouldYield(fiber)
      
      if (result !== false) {
        Effect.runSync(Ref.update(metrics, m => ({
          ...m,
          yieldCount: m.yieldCount + 1,
          lastYieldTime: Date.now()
        })))
      }
      
      return result
    }
    
    const getMetrics = Ref.get(metrics)
    const resetMetrics = Ref.set(metrics, {
      tasksScheduled: 0,
      tasksCompleted: 0,
      averageExecutionTime: 0,
      yieldCount: 0,
      lastYieldTime: 0
    })
    
    return {
      scheduler: Scheduler.make(scheduleTask, shouldYield),
      getMetrics,
      resetMetrics
    }
  })

// Monitoring dashboard for scheduler performance
const createSchedulerDashboard = (instrumentedScheduler: {
  scheduler: Scheduler.Scheduler
  getMetrics: Effect.Effect<SchedulerMetrics>
}) =>
  Effect.gen(function* () {
    const reportMetrics = Effect.gen(function* () {
      const metrics = yield* instrumentedScheduler.getMetrics
      
      yield* Effect.log("=== Scheduler Metrics ===")
      yield* Effect.log(`Tasks Scheduled: ${metrics.tasksScheduled}`)
      yield* Effect.log(`Tasks Completed: ${metrics.tasksCompleted}`)
      yield* Effect.log(`Average Execution Time: ${metrics.averageExecutionTime.toFixed(2)}ms`)
      yield* Effect.log(`Yield Count: ${metrics.yieldCount}`)
      yield* Effect.log(`Last Yield: ${metrics.lastYieldTime > 0 ? new Date(metrics.lastYieldTime).toISOString() : 'Never'}`)
      yield* Effect.log("========================")
    })
    
    // Report metrics every 5 seconds
    const metricsLoop = Effect.gen(function* () {
      while (true) {
        yield* Effect.sleep(Duration.seconds(5))
        yield* reportMetrics
      }
    }).pipe(Effect.fork)
    
    return {
      start: metricsLoop,
      reportNow: reportMetrics
    }
  })

// Usage example with monitoring
const monitoringExample = Effect.gen(function* () {
  const baseScheduler = new Scheduler.MixedScheduler(1024)
  const instrumented = yield* makeInstrumentedScheduler(baseScheduler)
  const dashboard = yield* createSchedulerDashboard(instrumented)
  
  const monitoring = yield* dashboard.start
  
  // Generate some workload
  const workload = Effect.gen(function* () {
    for (let i = 0; i < 100; i++) {
      yield* Effect.gen(function* () {
        // Varying workload
        const workAmount = Math.floor(Math.random() * 100) + 10
        for (let j = 0; j < workAmount; j++) {
          yield* Effect.sync(() => Math.random() * Math.random())
        }
        yield* Effect.sleep(Duration.millis(Math.random() * 50))
      }).pipe(
        Effect.withScheduler(instrumented.scheduler)
      )
    }
  })
  
  yield* workload
  yield* dashboard.reportNow
  yield* monitoring.interrupt
})
```

## Integration Examples

### Integration with Effect Runtime

```typescript
import { Effect, Scheduler, Runtime, FiberRef } from "effect"

// Custom runtime with specialized scheduler configuration
const makeCustomRuntime = (schedulerConfig: {
  readonly maxOpsBeforeYield: number
  readonly schedulingPriority: number
  readonly scheduler: Scheduler.Scheduler
}) =>
  Effect.gen(function* () {
    // Create runtime with custom scheduler settings
    const runtime = yield* Effect.runtime<never>()
    
    // Configure scheduler-related FiberRefs
    const configuredRuntime = Runtime.updateFiberRefs(runtime, (fiberRefs) =>
      fiberRefs
        .set(FiberRef.currentMaxOpsBeforeYield, schedulerConfig.maxOpsBeforeYield)
        .set(FiberRef.currentSchedulingPriority, schedulerConfig.schedulingPriority)
    )
    
    return configuredRuntime
  })

// Application-specific runtime configurations
const RuntimeConfigs = {
  // High-performance runtime for CPU-intensive tasks
  highPerformance: {
    maxOpsBeforeYield: 4096,
    schedulingPriority: 10,
    scheduler: new Scheduler.MixedScheduler(4096)
  },
  
  // UI-responsive runtime for user-facing operations
  uiResponsive: {
    maxOpsBeforeYield: 512,
    schedulingPriority: 8,
    scheduler: new Scheduler.MixedScheduler(512)
  },
  
  // Background runtime for non-critical tasks
  background: {
    maxOpsBeforeYield: 8192,
    schedulingPriority: 2,
    scheduler: Scheduler.makeBatched((runBatch) => {
      setTimeout(runBatch, 100)
    })
  }
}

// Task execution with runtime-specific scheduling
const executeWithRuntime = <A, E, R>(
  effect: Effect.Effect<A, E, R>,
  runtimeType: keyof typeof RuntimeConfigs
) =>
  Effect.gen(function* () {
    const config = RuntimeConfigs[runtimeType]
    const runtime = yield* makeCustomRuntime(config)
    
    return yield* effect.pipe(
      Effect.withScheduler(config.scheduler),
      Effect.provide(runtime)
    )
  })

// Example usage with different runtime configurations
const runtimeIntegrationExample = Effect.gen(function* () {
  // CPU-intensive computation with high-performance runtime
  const computationTask = Effect.gen(function* () {
    yield* Effect.log("Starting CPU-intensive computation")
    
    let result = 0
    for (let i = 0; i < 100000; i++) {
      result += Math.sqrt(i)
    }
    
    yield* Effect.log("Computation completed")
    return result
  })
  
  // UI update with responsive runtime
  const uiUpdateTask = Effect.gen(function* () {
    yield* Effect.log("Updating user interface")
    
    // Simulate UI updates
    for (let i = 0; i < 10; i++) {
      yield* Effect.sync(() => {
        // Simulate DOM updates
        console.log(`UI update ${i + 1}`)
      })
      yield* Effect.sleep(Duration.millis(16)) // 60 FPS
    }
    
    yield* Effect.log("UI updates completed")
    return "ui_updated"
  })
  
  // Background analytics with background runtime
  const analyticsTask = Effect.gen(function* () {
    yield* Effect.log("Processing analytics data")
    
    // Simulate analytics processing
    for (let i = 0; i < 50; i++) {
      yield* Effect.sync(() => {
        // Process analytics event
        console.log(`Processing event ${i + 1}`)
      })
    }
    
    yield* Effect.log("Analytics processing completed")
    return "analytics_processed"
  })
  
  // Execute tasks with appropriate runtime configurations
  const results = yield* Effect.all([
    executeWithRuntime(computationTask, "highPerformance"),
    executeWithRuntime(uiUpdateTask, "uiResponsive"),
    executeWithRuntime(analyticsTask, "background")
  ], { concurrency: "unbounded" })
  
  yield* Effect.log(`All tasks completed: ${JSON.stringify(results)}`)
})
```

### Integration with Web Workers

```typescript
import { Effect, Scheduler, Queue, Duration } from "effect"

interface WorkerTask {
  readonly id: string
  readonly type: string
  readonly payload: unknown
  readonly priority: number
}

interface WorkerResult {
  readonly taskId: string
  readonly result: unknown
  readonly error?: string
}

// Web Worker integration with custom scheduling
const makeWorkerScheduler = (workerScript: string) =>
  Effect.gen(function* () {
    const worker = new Worker(workerScript)
    const taskQueue = yield* Queue.bounded<WorkerTask>(1000)
    const resultQueue = yield* Queue.bounded<WorkerResult>(1000)
    
    // Custom scheduler for worker communication
    const workerScheduler = Scheduler.makeBatched((runBatch) => {
      // Batch worker messages for efficiency
      setTimeout(runBatch, 10)
    })
    
    // Handle worker messages
    worker.onmessage = (event) => {
      const result: WorkerResult = event.data
      Effect.runSync(Queue.offer(resultQueue, result))
    }
    
    // Worker task processor
    const processWorkerTasks = Effect.gen(function* () {
      while (true) {
        const task = yield* Queue.take(taskQueue)
        
        // Send task to worker
        worker.postMessage(task)
        
        // Log task scheduling
        yield* Effect.log(`Scheduled task ${task.id} to worker`)
      }
    }).pipe(
      Effect.withScheduler(workerScheduler),
      Effect.fork
    )
    
    const submitTask = (task: WorkerTask) =>
      Queue.offer(taskQueue, task).pipe(
        Effect.tap(() => Effect.log(`Queued task ${task.id}`))
      )
    
    const getResult = Queue.take(resultQueue)
    
    const cleanup = Effect.sync(() => {
      worker.terminate()
    })
    
    return {
      submitTask,
      getResult,
      start: processWorkerTasks,
      cleanup
    }
  })

// Example Web Worker script content (would be in a separate file)
const workerScriptContent = `
self.onmessage = function(event) {
  const task = event.data;
  
  try {
    let result;
    
    switch (task.type) {
      case 'computation':
        result = heavyComputation(task.payload);
        break;
      case 'image_processing':
        result = processImage(task.payload);
        break;
      default:
        throw new Error('Unknown task type: ' + task.type);
    }
    
    self.postMessage({
      taskId: task.id,
      result: result
    });
  } catch (error) {
    self.postMessage({
      taskId: task.id,
      result: null,
      error: error.message
    });
  }
};

function heavyComputation(data) {
  // Simulate heavy computation
  let result = 0;
  for (let i = 0; i < data.iterations; i++) {
    result += Math.sqrt(i);
  }
  return result;
}

function processImage(imageData) {
  // Simulate image processing
  return {
    processed: true,
    size: imageData.length,
    timestamp: Date.now()
  };
}
`

// Usage example with Web Worker scheduler
const webWorkerExample = Effect.gen(function* () {
  // Create worker script blob
  const blob = new Blob([workerScriptContent], { type: 'application/javascript' })
  const workerUrl = URL.createObjectURL(blob)
  
  const workerScheduler = yield* makeWorkerScheduler(workerUrl)
  const processor = yield* workerScheduler.start
  
  // Submit various tasks
  yield* workerScheduler.submitTask({
    id: "comp_1",
    type: "computation",
    payload: { iterations: 100000 },
    priority: 5
  })
  
  yield* workerScheduler.submitTask({
    id: "img_1",
    type: "image_processing",
    payload: new Uint8Array(1024),
    priority: 8
  })
  
  // Process results
  for (let i = 0; i < 2; i++) {
    const result = yield* workerScheduler.getResult
    yield* Effect.log(`Task ${result.taskId} completed: ${result.error ? 'ERROR' : 'SUCCESS'}`)
    
    if (result.error) {
      yield* Effect.log(`Error: ${result.error}`)
    } else {
      yield* Effect.log(`Result: ${JSON.stringify(result.result)}`)
    }
  }
  
  yield* processor.interrupt
  yield* workerScheduler.cleanup
  
  // Clean up blob URL
  URL.revokeObjectURL(workerUrl)
})
```

### Testing Strategies

```typescript
import { Effect, Scheduler, Duration, TestClock, TestServices } from "effect"

// Test helpers for scheduler behavior verification
const SchedulerTestUtils = {
  // Create a deterministic test scheduler
  createTestScheduler: () => {
    const executionLog: Array<{ name: string; priority: number; timestamp: number }> = []
    
    const scheduleTask = (task: Scheduler.Task, priority: number) => {
      const taskName = task.name || 'anonymous'
      executionLog.push({
        name: taskName,
        priority,
        timestamp: Date.now()
      })
      task()
    }
    
    return {
      scheduler: Scheduler.make(scheduleTask),
      getExecutionLog: () => [...executionLog],
      clearLog: () => { executionLog.length = 0 }
    }
  },
  
  // Verify task execution order
  verifyExecutionOrder: (
    log: Array<{ name: string; priority: number }>,
    expectedOrder: string[]
  ) => {
    const actualOrder = log.map(entry => entry.name)
    return JSON.stringify(actualOrder) === JSON.stringify(expectedOrder)
  },
  
  // Verify priority ordering
  verifyPriorityOrder: (log: Array<{ priority: number }>) => {
    for (let i = 1; i < log.length; i++) {
      if (log[i].priority > log[i - 1].priority) {
        return false
      }
    }
    return true
  }
}

// Comprehensive scheduler test suite
const schedulerTestSuite = Effect.gen(function* () {
  yield* Effect.log("Running Scheduler Test Suite")
  
  // Test 1: Basic priority ordering
  yield* Effect.log("Test 1: Priority Ordering")
  const testScheduler1 = SchedulerTestUtils.createTestScheduler()
  
  const createNamedTask = (name: string) => {
    const task = () => console.log(`Executing ${name}`)
    Object.defineProperty(task, 'name', { value: name })
    return task
  }
  
  // Schedule tasks in reverse priority order
  testScheduler1.scheduler.scheduleTask(createNamedTask('low'), 1)
  testScheduler1.scheduler.scheduleTask(createNamedTask('high'), 10)
  testScheduler1.scheduler.scheduleTask(createNamedTask('medium'), 5)
  
  yield* Effect.sleep(Duration.millis(100))
  
  const log1 = testScheduler1.getExecutionLog()
  const priorityOrderCorrect = SchedulerTestUtils.verifyPriorityOrder(log1)
  yield* Effect.log(`Priority ordering test: ${priorityOrderCorrect ? 'PASSED' : 'FAILED'}`)
  
  // Test 2: Scheduler yielding behavior
  yield* Effect.log("Test 2: Yielding Behavior")
  
  let yieldCount = 0
  const testScheduler2 = Scheduler.make(
    (task, priority) => {
      task()
    },
    (fiber: any) => {
      yieldCount++
      return yieldCount > 5 ? 1 : false // Yield after 5 checks
    }
  )
  
  const yieldingTask = Effect.gen(function* () {
    for (let i = 0; i < 100; i++) {
      yield* Effect.sync(() => {
        // Simulate work
        Math.random()
      })
    }
  }).pipe(
    Effect.withScheduler(testScheduler2)
  )
  
  yield* yieldingTask
  yield* Effect.log(`Yielding test: ${yieldCount > 5 ? 'PASSED' : 'FAILED'} (${yieldCount} yields)`)
  
  // Test 3: Scheduler performance comparison
  yield* Effect.log("Test 3: Performance Comparison")
  
  const performanceTest = (scheduler: Scheduler.Scheduler, name: string) =>
    Effect.gen(function* () {
      const startTime = yield* Effect.clockWith(clock => clock.currentTimeMillis)
      
      const tasks = Array.from({ length: 1000 }, (_, i) =>
        Effect.sync(() => Math.random()).pipe(
          Effect.withScheduler(scheduler)
        )
      )
      
      yield* Effect.all(tasks, { concurrency: "unbounded" })
      
      const endTime = yield* Effect.clockWith(clock => clock.currentTimeMillis)
      const duration = endTime - startTime
      
      yield* Effect.log(`${name}: ${duration}ms`)
      return duration
    })
  
  const defaultTime = yield* performanceTest(Scheduler.defaultScheduler, "Default Scheduler")
  const syncTime = yield* performanceTest(new Scheduler.SyncScheduler(), "Sync Scheduler")
  const mixedTime = yield* performanceTest(new Scheduler.MixedScheduler(2048), "Mixed Scheduler")
  
  yield* Effect.log(`Performance test completed. Fastest: ${Math.min(defaultTime, syncTime, mixedTime)}ms`)
  
  yield* Effect.log("Scheduler Test Suite Completed")
})

// Integration test with TestClock
const testSchedulerWithClock = Effect.gen(function* () {
  yield* Effect.log("Testing scheduler with TestClock")
  
  const timerScheduler = Scheduler.timer(1000) // 1 second delay
  
  const delayedTask = Effect.gen(function* () {
    yield* Effect.log("Task started")
    yield* Effect.sleep(Duration.seconds(2))
    yield* Effect.log("Task completed")
  }).pipe(
    Effect.withScheduler(timerScheduler)
  )
  
  // Run with test clock to control time
  yield* delayedTask.pipe(
    Effect.provide(TestServices.TestServices),
    Effect.tap(() => TestClock.adjust(Duration.seconds(3)))
  )
  
  yield* Effect.log("TestClock integration test completed")
})

// Property-based testing for scheduler behavior
const propertyBasedSchedulerTest = Effect.gen(function* () {
  yield* Effect.log("Running property-based scheduler tests")
  
  // Property: Higher priority tasks should never execute after lower priority tasks
  const testPriorityProperty = (priorities: number[]) =>
    Effect.gen(function* () {
      const testScheduler = SchedulerTestUtils.createTestScheduler()
      const tasks = priorities.map((priority, index) => {
        const task = () => {}
        Object.defineProperty(task, 'name', { value: `task_${index}` })
        return { task, priority }
      })
      
      // Schedule tasks in random order
      const shuffled = [...tasks].sort(() => Math.random() - 0.5)
      shuffled.forEach(({ task, priority }) => {
        testScheduler.scheduler.scheduleTask(task, priority)
      })
      
      yield* Effect.sleep(Duration.millis(50))
      
      const log = testScheduler.getExecutionLog()
      const priorityOrderCorrect = SchedulerTestUtils.verifyPriorityOrder(log)
      
      if (!priorityOrderCorrect) {
        yield* Effect.log(`Priority property violation with priorities: ${JSON.stringify(priorities)}`)
        yield* Effect.log(`Execution order: ${JSON.stringify(log.map(e => e.priority))}`)
      }
      
      return priorityOrderCorrect
    })
  
  // Test with various priority combinations
  const testCases = [
    [1, 2, 3, 4, 5],
    [5, 4, 3, 2, 1],
    [1, 5, 2, 4, 3],
    [10, 1, 10, 1, 10],
    [1, 1, 1, 1, 1]
  ]
  
  const results = yield* Effect.all(
    testCases.map(testPriorityProperty)
  )
  
  const allPassed = results.every(Boolean)
  yield* Effect.log(`Property-based tests: ${allPassed ? 'ALL PASSED' : 'SOME FAILED'}`)
})

// Main test runner
const runAllSchedulerTests = Effect.gen(function* () {
  yield* schedulerTestSuite
  yield* testSchedulerWithClock
  yield* propertyBasedSchedulerTest
})
```

## Conclusion

Scheduler provides **fine-grained task execution control**, **priority-based scheduling**, and **automatic yielding** for Effect applications.

Key benefits:
- **Performance Control**: Prevent blocking operations from starving the runtime
- **Priority Management**: Ensure critical tasks execute before background work
- **Resource Optimization**: Automatic yielding maintains system responsiveness
- **Flexibility**: Multiple scheduler types for different use cases
- **Integration**: Seamless integration with Effect runtime and ecosystem

The Scheduler module is essential for building responsive applications that handle both user-facing operations and background processing efficiently. Use it when you need precise control over task execution timing and priority, especially in applications with mixed workloads of varying importance and computational intensity.