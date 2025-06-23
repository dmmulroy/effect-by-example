# FiberId: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem FiberId Solves

When building concurrent applications, debugging and monitoring fiber execution becomes critical. Traditional approaches lack unique identifiers for concurrent operations, making it nearly impossible to track specific operations through complex execution flows:

```typescript
// Traditional approach - no way to identify specific async operations
async function processMultipleOrders(orders: Order[]): Promise<ProcessedOrder[]> {
  const promises = orders.map(async (order, index) => {
    try {
      console.log(`Processing order ${index}`) // Which one failed?
      const result = await processOrder(order)
      console.log(`Completed order ${index}`) // Hard to correlate
      return result
    } catch (error) {
      console.error(`Failed order ${index}:`, error) // Limited context
      throw error
    }
  })
  
  return Promise.all(promises)
}

// When debugging concurrent operations:
// - Which specific operation failed?
// - How do you track parent-child relationships?
// - How do you correlate logs across fiber boundaries?
// - How do you interrupt specific operations by ID?
```

This approach leads to:
- **No Operation Identity** - Cannot distinguish between concurrent operations
- **Poor Debugging Experience** - Logs lack correlation context
- **Difficult Fiber Management** - Cannot target specific fibers for interruption
- **Limited Observability** - No way to trace fiber relationships and lifecycles

### The FiberId Solution

FiberId provides unique identification for every fiber in Effect, enabling precise tracking, debugging, and control of concurrent operations:

```typescript
import { Effect, FiberId, Fiber } from "effect"

// Each fiber gets a unique identifier automatically
const identifiedOperation = Effect.gen(function* () {
  const currentFiberId = yield* Effect.fiberId
  yield* Effect.log(`Operation started`, { fiberId: currentFiberId })
  
  const result = yield* processOrder(order)
  
  yield* Effect.log(`Operation completed`, { fiberId: currentFiberId })
  return result
})

// Fiber relationships are trackable
const parentChildExample = Effect.gen(function* () {
  const parentId = yield* Effect.fiberId
  
  const childFiber = yield* Effect.fork(
    Effect.gen(function* () {
      const childId = yield* Effect.fiberId
      yield* Effect.log(`Child fiber`, { parentId, childId })
      return yield* someOperation()
    })
  )
  
  // Can interrupt specific fibers by ID
  yield* Fiber.interrupt(childFiber)
})
```

### Key Concepts

**FiberId Types**: Three distinct types representing different fiber states:
- `None` - Represents no fiber (id: -1)  
- `Runtime` - Active fiber with unique ID and start time
- `Composite` - Combination of multiple fiber IDs

**Fiber Identification**: Every running fiber automatically receives a unique ID for tracking and debugging

**Parent-Child Relationships**: Composite FiberIds represent relationships between parent and child fibers

## Basic Usage Patterns

### Pattern 1: Getting Current Fiber ID

```typescript
import { Effect, FiberId } from "effect"

// Get the current fiber's ID
const getCurrentFiberId = Effect.gen(function* () {
  const fiberId = yield* Effect.fiberId
  yield* Effect.log(`Current fiber: ${FiberId.threadName(fiberId)}`)
  return fiberId
})

// Using fiber ID in computations
const trackOperation = <A, E, R>(
  operation: Effect.Effect<A, E, R>,
  operationName: string
) => 
  Effect.gen(function* () {
    const fiberId = yield* Effect.fiberId
    const fiberName = FiberId.threadName(fiberId)
    
    yield* Effect.log(`[${fiberName}] Starting ${operationName}`)
    const result = yield* operation
    yield* Effect.log(`[${fiberName}] Completed ${operationName}`)
    
    return result
  })
```

### Pattern 2: Creating and Managing Fiber IDs

```typescript
import { Effect, FiberId } from "effect"

// Create specific fiber IDs
const createFiberIds = Effect.gen(function* () {
  // Create a runtime fiber ID
  const runtimeId = FiberId.make(1, Date.now())
  
  // Create a composite ID from multiple IDs
  const id1 = FiberId.make(1, Date.now())
  const id2 = FiberId.make(2, Date.now())
  const compositeId = FiberId.combine(id1, id2)
  
  // Get thread names for display
  const compositeName = FiberId.threadName(compositeId)
  yield* Effect.log(`Composite fiber: ${compositeName}`)
  
  return compositeId
})

// Check fiber ID types
const analyzeFiberId = (fiberId: FiberId.FiberId) => 
  Effect.gen(function* () {
    if (FiberId.isNone(fiberId)) {
      yield* Effect.log("No active fiber")
    } else if (FiberId.isRuntime(fiberId)) {
      yield* Effect.log(`Runtime fiber: ID ${fiberId.id}, started at ${fiberId.startTimeMillis}`)
    } else if (FiberId.isComposite(fiberId)) {
      yield* Effect.log(`Composite fiber with left: ${FiberId.threadName(fiberId.left)}, right: ${FiberId.threadName(fiberId.right)}`)
    }
  })
```

### Pattern 3: Fiber Interruption with IDs

```typescript
import { Effect, Fiber, FiberId } from "effect"

// Interrupt specific fibers by ID
const interruptExample = Effect.gen(function* () {
  // Fork a long-running operation
  const longRunningFiber = yield* Effect.fork(
    Effect.gen(function* () {
      const fiberId = yield* Effect.fiberId
      yield* Effect.log(`Long operation started: ${FiberId.threadName(fiberId)}`)
      yield* Effect.sleep("10 seconds")
      return "completed"
    })
  )
  
  // Get the fiber's ID
  const fiberId = Fiber.id(longRunningFiber)
  yield* Effect.log(`Forked fiber ID: ${FiberId.threadName(fiberId)}`)
  
  // Interrupt after a delay
  yield* Effect.sleep("2 seconds")
  yield* Effect.log("Interrupting fiber...")
  yield* Fiber.interrupt(longRunningFiber)
  
  return "main completed"
})
```

## Real-World Examples

### Example 1: Order Processing with Fiber Tracking

Building a robust order processing system that tracks each order through its lifecycle:

```typescript
import { Effect, Fiber, FiberId, Ref } from "effect"

interface Order {
  readonly id: string
  readonly customerId: string
  readonly items: OrderItem[]
  readonly total: number
}

interface OrderItem {
  readonly productId: string
  readonly quantity: number
  readonly price: number
}

interface ProcessingContext {
  readonly orderId: string
  readonly fiberId: FiberId.FiberId
  readonly startTime: number
}

// Create a processing context with fiber identification
const createProcessingContext = (orderId: string) =>
  Effect.gen(function* () {
    const fiberId = yield* Effect.fiberId
    const startTime = Date.now()
    
    const context: ProcessingContext = {
      orderId,
      fiberId,
      startTime
    }
    
    yield* Effect.log(`[${FiberId.threadName(fiberId)}] Created processing context for order ${orderId}`)
    return context
  })

// Process order with full fiber tracking
const processOrderWithTracking = (order: Order) =>
  Effect.gen(function* () {
    const context = yield* createProcessingContext(order.id)
    const fiberName = FiberId.threadName(context.fiberId)
    
    try {
      yield* Effect.log(`[${fiberName}] Starting order validation`)
      const validatedOrder = yield* validateOrder(order)
      
      yield* Effect.log(`[${fiberName}] Processing payment`)
      const paymentResult = yield* processPayment(validatedOrder)
      
      yield* Effect.log(`[${fiberName}] Updating inventory`)
      yield* updateInventory(validatedOrder.items)
      
      yield* Effect.log(`[${fiberName}] Scheduling shipping`)
      const shippingInfo = yield* scheduleShipping(validatedOrder)
      
      const processingTime = Date.now() - context.startTime
      yield* Effect.log(`[${fiberName}] Order completed in ${processingTime}ms`)
      
      return {
        order: validatedOrder,
        payment: paymentResult,
        shipping: shippingInfo,
        processingContext: context
      }
    } catch (error) {
      const processingTime = Date.now() - context.startTime
      yield* Effect.log(`[${fiberName}] Order failed after ${processingTime}ms: ${error}`)
      return yield* Effect.fail(error)
    }
  })

// Concurrent order processing with individual fiber control
const processBatchWithFiberControl = (orders: Order[]) =>
  Effect.gen(function* () {
    const processingRegistry = yield* Ref.make(new Map<string, FiberId.FiberId>())
    
    // Process orders concurrently while tracking fiber IDs
    const orderFibers = yield* Effect.forEach(
      orders,
      (order) => Effect.gen(function* () {
        const fiber = yield* Effect.fork(processOrderWithTracking(order))
        const fiberId = Fiber.id(fiber)
        
        // Register fiber for potential cancellation
        yield* Ref.update(processingRegistry, (registry) => 
          registry.set(order.id, fiberId)
        )
        
        return { orderId: order.id, fiber }
      }),
      { concurrency: 5 }
    )
    
    // Wait for all orders or handle individual failures
    const results = yield* Effect.forEach(
      orderFibers,
      ({ orderId, fiber }) => Effect.gen(function* () {
        const result = yield* Fiber.join(fiber)
        
        // Remove from registry when completed
        yield* Ref.update(processingRegistry, (registry) => {
          registry.delete(orderId)
          return registry
        })
        
        return { orderId, result }
      }),
      { concurrency: "unbounded" }
    )
    
    return results
  })

// Helper functions (simplified for example)
const validateOrder = (order: Order) => 
  Effect.succeed(order).pipe(Effect.delay("100 millis"))

const processPayment = (order: Order) =>
  Effect.succeed({ transactionId: `txn_${order.id}`, amount: order.total })
    .pipe(Effect.delay("200 millis"))

const updateInventory = (items: OrderItem[]) =>
  Effect.succeed(undefined).pipe(Effect.delay("150 millis"))

const scheduleShipping = (order: Order) =>
  Effect.succeed({ trackingNumber: `ship_${order.id}`, estimatedDelivery: "3-5 days" })
    .pipe(Effect.delay("100 millis"))
```

### Example 2: Real-time Chat System with Fiber Management

Building a chat system that tracks active connections and manages user sessions:

```typescript
import { Effect, Fiber, FiberId, Hub, Queue, Ref } from "effect"

interface ChatMessage {
  readonly userId: string
  readonly content: string
  readonly timestamp: number
  readonly fiberId: FiberId.FiberId
}

interface UserSession {
  readonly userId: string
  readonly fiberId: FiberId.FiberId
  readonly connectionTime: number
  readonly fiber: Fiber.RuntimeFiber<void, never>
}

// Connection manager tracking active sessions by fiber ID
const createConnectionManager = () =>
  Effect.gen(function* () {
    const activeConnections = yield* Ref.make(new Map<string, UserSession>())
    const messageHub = yield* Hub.unbounded<ChatMessage>()
    
    const addConnection = (userId: string) =>
      Effect.gen(function* () {
        const fiberId = yield* Effect.fiberId
        const connectionTime = Date.now()
        const fiberName = FiberId.threadName(fiberId)
        
        // Create connection handler fiber
        const connectionFiber = yield* Effect.fork(
          Effect.gen(function* () {
            yield* Effect.log(`[${fiberName}] User ${userId} connected`)
            
            // Simulate connection handling
            yield* Effect.forever(
              Effect.gen(function* () {
                yield* Effect.sleep("30 seconds")
                yield* Effect.log(`[${fiberName}] Heartbeat for ${userId}`)
              })
            )
          }).pipe(
            Effect.ensuring(
              Effect.gen(function* () {
                yield* Effect.log(`[${fiberName}] User ${userId} disconnected`)
                yield* Ref.update(activeConnections, (connections) => {
                  connections.delete(userId)
                  return connections
                })
              })
            )
          )
        )
        
        const session: UserSession = {
          userId,
          fiberId: Fiber.id(connectionFiber),
          connectionTime,
          fiber: connectionFiber
        }
        
        yield* Ref.update(activeConnections, (connections) =>
          connections.set(userId, session)
        )
        
        return session
      })
    
    const removeConnection = (userId: string) =>
      Effect.gen(function* () {
        const connections = yield* Ref.get(activeConnections)
        const session = connections.get(userId)
        
        if (session) {
          const fiberName = FiberId.threadName(session.fiberId)
          yield* Effect.log(`[${fiberName}] Disconnecting user ${userId}`)
          yield* Fiber.interrupt(session.fiber)
        }
      })
    
    const broadcastMessage = (fromUserId: string, content: string) =>
      Effect.gen(function* () {
        const fiberId = yield* Effect.fiberId
        const message: ChatMessage = {
          userId: fromUserId,
          content,
          timestamp: Date.now(),
          fiberId
        }
        
        const fiberName = FiberId.threadName(fiberId)
        yield* Effect.log(`[${fiberName}] Broadcasting message from ${fromUserId}`)
        yield* Hub.publish(messageHub, message)
      })
    
    const getActiveConnections = () =>
      Effect.gen(function* () {
        const connections = yield* Ref.get(activeConnections)
        return Array.from(connections.values()).map(session => ({
          userId: session.userId,
          fiberId: session.fiberId,
          fiberName: FiberId.threadName(session.fiberId),
          connectionTime: session.connectionTime,
          uptime: Date.now() - session.connectionTime
        }))
      })
    
    return {
      addConnection,
      removeConnection,
      broadcastMessage,
      getActiveConnections,
      messageHub
    }
  })

// Chat room with fiber-based session management
const createChatRoom = (roomId: string) =>
  Effect.gen(function* () {
    const connectionManager = yield* createConnectionManager()
    const roomFiberId = yield* Effect.fiberId
    const roomFiberName = FiberId.threadName(roomFiberId)
    
    yield* Effect.log(`[${roomFiberName}] Chat room ${roomId} created`)
    
    const handleUserJoin = (userId: string) =>
      Effect.gen(function* () {
        const session = yield* connectionManager.addConnection(userId)
        const sessionFiberName = FiberId.threadName(session.fiberId)
        
        yield* connectionManager.broadcastMessage("system", `User ${userId} joined (${sessionFiberName})`)
        return session
      })
    
    const handleUserLeave = (userId: string) =>
      Effect.gen(function* () {
        yield* connectionManager.removeConnection(userId)
        yield* connectionManager.broadcastMessage("system", `User ${userId} left`)
      })
    
    const sendMessage = (userId: string, content: string) =>
      connectionManager.broadcastMessage(userId, content)
    
    const getRoomStatus = () =>
      Effect.gen(function* () {
        const connections = yield* connectionManager.getActiveConnections()
        return {
          roomId,
          roomFiberId: roomFiberName,
          activeUsers: connections.length,
          connections
        }
      })
    
    return {
      handleUserJoin,
      handleUserLeave,
      sendMessage,
      getRoomStatus,
      messageHub: connectionManager.messageHub
    }
  })
```

### Example 3: Distributed Task Processing with Fiber Coordination

Building a distributed work queue that coordinates tasks across multiple workers:

```typescript
import { Effect, Fiber, FiberId, Queue, Ref, Schedule } from "effect"
import { Array as Arr, HashMap } from "effect"

interface Task {
  readonly id: string
  readonly type: string
  readonly payload: unknown
  readonly priority: number
  readonly retryCount: number
}

interface WorkerInfo {
  readonly workerId: string
  readonly fiberId: FiberId.FiberId
  readonly status: "idle" | "busy" | "stopped"
  readonly currentTask: string | null
  readonly tasksCompleted: number
  readonly startTime: number
}

interface TaskExecution {
  readonly taskId: string
  readonly workerId: string
  readonly workerFiberId: FiberId.FiberId
  readonly startTime: number
  readonly status: "running" | "completed" | "failed"
}

// Distributed task processor with fiber coordination
const createTaskProcessor = (processorId: string) =>
  Effect.gen(function* () {
    const taskQueue = yield* Queue.unbounded<Task>()
    const workerRegistry = yield* Ref.make(HashMap.empty<string, WorkerInfo>())
    const executionTracker = yield* Ref.make(HashMap.empty<string, TaskExecution>())
    
    // Create a worker with unique fiber identification
    const createWorker = (workerId: string) =>
      Effect.gen(function* () {
        const workerFiberId = yield* Effect.fiberId
        const workerFiberName = FiberId.threadName(workerFiberId)
        
        const workerInfo: WorkerInfo = {
          workerId,
          fiberId: workerFiberId,
          status: "idle",
          currentTask: null,
          tasksCompleted: 0,
          startTime: Date.now()
        }
        
        yield* Ref.update(workerRegistry, HashMap.set(workerId, workerInfo))
        yield* Effect.log(`[${workerFiberName}] Worker ${workerId} started`)
        
        // Worker processing loop
        const processNextTask = Effect.gen(function* () {
          const task = yield* Queue.take(taskQueue)
          const executionFiberId = yield* Effect.fiberId
          const executionFiberName = FiberId.threadName(executionFiberId)
          
          // Update worker status
          yield* Ref.update(workerRegistry, 
            HashMap.modify(workerId, (info) => ({ 
              ...info, 
              status: "busy" as const, 
              currentTask: task.id 
            }))
          )
          
          // Track task execution
          const execution: TaskExecution = {
            taskId: task.id,
            workerId,
            workerFiberId: executionFiberId,
            startTime: Date.now(),
            status: "running"
          }
          
          yield* Ref.update(executionTracker, HashMap.set(task.id, execution))
          yield* Effect.log(`[${executionFiberName}] Worker ${workerId} processing task ${task.id}`)
          
          try {
            // Process the task
            yield* processTask(task)
            
            // Mark as completed
            yield* Ref.update(executionTracker, 
              HashMap.modify(task.id, (exec) => ({ ...exec, status: "completed" as const }))
            )
            
            yield* Effect.log(`[${executionFiberName}] Task ${task.id} completed by worker ${workerId}`)
            
            // Update worker stats
            yield* Ref.update(workerRegistry,
              HashMap.modify(workerId, (info) => ({
                ...info,
                status: "idle" as const,
                currentTask: null,
                tasksCompleted: info.tasksCompleted + 1
              }))
            )
            
          } catch (error) {
            yield* Effect.log(`[${executionFiberName}] Task ${task.id} failed: ${error}`)
            
            yield* Ref.update(executionTracker,
              HashMap.modify(task.id, (exec) => ({ ...exec, status: "failed" as const }))
            )
            
            // Retry logic could be added here
            if (task.retryCount < 3) {
              const retryTask = { ...task, retryCount: task.retryCount + 1 }
              yield* Queue.offer(taskQueue, retryTask)
              yield* Effect.log(`[${executionFiberName}] Task ${task.id} queued for retry`)
            }
          }
        })
        
        return Effect.forever(processNextTask).pipe(
          Effect.ensuring(
            Effect.gen(function* () {
              yield* Effect.log(`[${workerFiberName}] Worker ${workerId} shutting down`)
              yield* Ref.update(workerRegistry, HashMap.remove(workerId))
            })
          )
        )
      })
    
    const startWorkers = (count: number) =>
      Effect.gen(function* () {
        const workerFibers = yield* Effect.forEach(
          Arr.range(0, count - 1),
          (index) => Effect.fork(createWorker(`${processorId}-worker-${index}`)),
          { concurrency: "unbounded" }
        )
        
        const processorFiberId = yield* Effect.fiberId
        const processorFiberName = FiberId.threadName(processorFiberId)
        
        yield* Effect.log(`[${processorFiberName}] Started ${count} workers for processor ${processorId}`)
        return workerFibers
      })
    
    const submitTask = (task: Task) =>
      Effect.gen(function* () {
        yield* Queue.offer(taskQueue, task)
        const fiberId = yield* Effect.fiberId
        const fiberName = FiberId.threadName(fiberId)
        yield* Effect.log(`[${fiberName}] Task ${task.id} submitted to queue`)
      })
    
    const getProcessorStatus = () =>
      Effect.gen(function* () {
        const workers = yield* Ref.get(workerRegistry)
        const executions = yield* Ref.get(executionTracker)
        const queueSize = yield* Queue.size(taskQueue)
        
        return {
          processorId,
          queueSize,
          workerCount: HashMap.size(workers),
          workers: HashMap.values(workers),
          runningTasks: Arr.filter(
            HashMap.values(executions),
            (exec) => exec.status === "running"
          ),
          completedTasks: Arr.filter(
            HashMap.values(executions),
            (exec) => exec.status === "completed"
          ).length,
          failedTasks: Arr.filter(
            HashMap.values(executions),
            (exec) => exec.status === "failed"
          ).length
        }
      })
    
    const stopWorker = (workerId: string) =>
      Effect.gen(function* () {
        const workers = yield* Ref.get(workerRegistry)
        const worker = HashMap.get(workers, workerId)
        
        if (worker._tag === "Some") {
          const fiberName = FiberId.threadName(worker.value.fiberId)
          yield* Effect.log(`[${fiberName}] Stopping worker ${workerId}`)
          // In a real implementation, you'd need to track worker fibers
          // to interrupt them individually
        }
      })
    
    return {
      startWorkers,
      submitTask,
      getProcessorStatus,
      stopWorker
    }
  })

// Simple task processor
const processTask = (task: Task) =>
  Effect.gen(function* () {
    // Simulate task processing time
    const processingTime = Math.random() * 2000 + 500
    yield* Effect.sleep(`${processingTime} millis`)
    
    // Simulate occasional failures
    if (Math.random() < 0.1) {
      return yield* Effect.fail(`Task ${task.id} processing failed`)
    }
    
    return `Task ${task.id} completed`
  })
```

## Advanced Features Deep Dive

### Feature 1: Composite FiberIds and Relationships

Composite FiberIds represent complex relationships between fibers, enabling sophisticated coordination patterns.

#### Basic Composite Usage

```typescript
import { Effect, FiberId } from "effect"

// Creating composite fiber IDs
const compositeExample = Effect.gen(function* () {
  const fiber1Id = FiberId.make(1, Date.now())
  const fiber2Id = FiberId.make(2, Date.now())
  
  // Combine two fiber IDs
  const composite = FiberId.combine(fiber1Id, fiber2Id)
  
  yield* Effect.log(`Composite fiber: ${FiberId.threadName(composite)}`)
  
  // Get all individual IDs from composite
  const allIds = FiberId.ids(composite)
  yield* Effect.log(`Individual IDs: ${Array.from(allIds).join(", ")}`)
  
  return composite
})

// Working with composite hierarchies
const hierarchicalExample = Effect.gen(function* () {
  const parentId = FiberId.make(1, Date.now())
  const child1Id = FiberId.make(2, Date.now())
  const child2Id = FiberId.make(3, Date.now())
  
  // Create parent-children relationship
  const childrenComposite = FiberId.combine(child1Id, child2Id)
  const familyComposite = FiberId.combine(parentId, childrenComposite)
  
  yield* Effect.log(`Family structure: ${FiberId.threadName(familyComposite)}`)
  
  // Analyze the structure
  if (FiberId.isComposite(familyComposite)) {
    yield* Effect.log(`Left: ${FiberId.threadName(familyComposite.left)}`)
    yield* Effect.log(`Right: ${FiberId.threadName(familyComposite.right)}`)
  }
  
  return familyComposite
})
```

#### Real-World Composite Example: Pipeline Processing

```typescript
import { Effect, Fiber, FiberId, Ref } from "effect"

interface PipelineStage<A, B> {
  readonly name: string
  readonly process: (input: A) => Effect.Effect<B, Error, never>
}

interface PipelineExecution<A> {
  readonly pipelineId: string
  readonly input: A
  readonly stages: PipelineStage<any, any>[]
  readonly executionId: FiberId.FiberId
}

// Pipeline processor that tracks stage relationships
const createPipeline = <A>(pipelineId: string, stages: PipelineStage<any, any>[]) =>
  Effect.gen(function* () {
    const pipelineFiberId = yield* Effect.fiberId
    const pipelineFiberName = FiberId.threadName(pipelineFiberId)
    
    const processPipeline = (input: A) =>
      Effect.gen(function* () {
        const executionFiberId = yield* Effect.fiberId
        const stageExecutions = yield* Ref.make<FiberId.FiberId[]>([])
        
        yield* Effect.log(`[${pipelineFiberName}] Starting pipeline ${pipelineId}`)
        
        let currentData: any = input
        
        // Process each stage and track fiber relationships
        for (const [index, stage] of stages.entries()) {
          const stageFiber = yield* Effect.fork(
            Effect.gen(function* () {
              const stageFiberId = yield* Effect.fiberId
              const stageFiberName = FiberId.threadName(stageFiberId)
              
              yield* Effect.log(`[${stageFiberName}] Processing stage ${index}: ${stage.name}`)
              
              // Track this stage's fiber ID
              yield* Ref.update(stageExecutions, (executions) => [...executions, stageFiberId])
              
              const result = yield* stage.process(currentData)
              yield* Effect.log(`[${stageFiberName}] Completed stage ${index}: ${stage.name}`)
              
              return result
            })
          )
          
          currentData = yield* Fiber.join(stageFiber)
        }
        
        // Create composite ID representing the entire pipeline execution
        const allStageIds = yield* Ref.get(stageExecutions)
        const pipelineCompositeId = allStageIds.reduce(
          (acc, stageId) => FiberId.combine(acc, stageId),
          executionFiberId
        )
        
        yield* Effect.log(`[${pipelineFiberName}] Pipeline completed with composite ID: ${FiberId.threadName(pipelineCompositeId)}`)
        
        return {
          result: currentData,
          pipelineId,
          executionId: pipelineCompositeId,
          stageCount: stages.length
        }
      })
    
    return { processPipeline }
  })

// Example usage with data transformation pipeline
const dataProcessingExample = Effect.gen(function* () {
  const stages: PipelineStage<any, any>[] = [
    {
      name: "validate",
      process: (data: unknown) => Effect.succeed(data).pipe(Effect.delay("100 millis"))
    },
    {
      name: "transform",
      process: (data: unknown) => Effect.succeed({ transformed: data }).pipe(Effect.delay("200 millis"))
    },
    {
      name: "enrich",
      process: (data: any) => Effect.succeed({ ...data, enriched: true }).pipe(Effect.delay("150 millis"))
    },
    {
      name: "save",
      process: (data: any) => Effect.succeed({ ...data, saved: true }).pipe(Effect.delay("100 millis"))
    }
  ]
  
  const pipeline = yield* createPipeline("data-processing", stages)
  
  const result = yield* pipeline.processPipeline({ id: "test-data", value: 42 })
  
  yield* Effect.log(`Final result: ${JSON.stringify(result)}`)
  
  return result
})
```

### Feature 2: Fiber ID Utilities and Analysis

FiberId provides powerful utilities for analyzing and working with fiber identities.

#### FiberId Utility Functions

```typescript
import { Effect, FiberId, HashSet } from "effect"

// Comprehensive fiber ID analysis
const analyzeFiberIds = Effect.gen(function* () {
  // Create various fiber IDs
  const noneId = FiberId.none
  const runtime1 = FiberId.make(1, Date.now())
  const runtime2 = FiberId.make(2, Date.now())
  const composite = FiberId.combine(runtime1, runtime2)
  
  // Convert to sets for analysis
  const compositeSet = FiberId.toSet(composite)
  const compositeIds = FiberId.ids(composite)
  
  yield* Effect.log(`None ID: ${FiberId.threadName(noneId)}`)
  yield* Effect.log(`Runtime 1: ${FiberId.threadName(runtime1)}`)
  yield* Effect.log(`Runtime 2: ${FiberId.threadName(runtime2)}`)
  yield* Effect.log(`Composite: ${FiberId.threadName(composite)}`)
  yield* Effect.log(`Individual IDs in composite: ${Array.from(compositeIds).join(", ")}`)
  
  // Analyze fiber ID properties
  const analysis = {
    noneCheck: FiberId.isNone(noneId),
    runtime1Check: FiberId.isRuntime(runtime1),
    compositeCheck: FiberId.isComposite(composite),
    compositeIdCount: HashSet.size(compositeIds),
    runtimeSetSize: HashSet.size(compositeSet)
  }
  
  yield* Effect.log(`Analysis: ${JSON.stringify(analysis, null, 2)}`)
  
  return analysis
})

// Fiber ID combination strategies
const combinationStrategies = Effect.gen(function* () {
  const baseIds = [
    FiberId.make(1, Date.now()),
    FiberId.make(2, Date.now()),
    FiberId.make(3, Date.now())
  ]
  
  // Combine all IDs into a single composite
  const allCombined = FiberId.combineAll(HashSet.fromIterable(baseIds))
  yield* Effect.log(`All combined: ${FiberId.threadName(allCombined)}`)
  
  // Sequential combination strategy
  const sequentialCombined = baseIds.reduce(
    (acc, id) => FiberId.combine(acc, id),
    FiberId.none
  )
  yield* Effect.log(`Sequential combined: ${FiberId.threadName(sequentialCombined)}`)
  
  // Pair-wise combination strategy
  const pairwiseCombinations = []
  for (let i = 0; i < baseIds.length; i++) {
    for (let j = i + 1; j < baseIds.length; j++) {
      const combined = FiberId.combine(baseIds[i], baseIds[j])
      pairwiseCombinations.push(combined)
      yield* Effect.log(`Pair ${i}-${j}: ${FiberId.threadName(combined)}`)
    }
  }
  
  return {
    allCombined,
    sequentialCombined,
    pairwiseCombinations
  }
})
```

#### Advanced Fiber ID Tracking System

```typescript
import { Effect, FiberId, Ref, HashMap } from "effect"

interface FiberMetrics {
  readonly fiberId: FiberId.FiberId
  readonly creationTime: number
  readonly lastActivity: number
  readonly operationCount: number
  readonly status: "active" | "idle" | "completed"
}

// Advanced fiber tracking system
const createFiberTracker = () =>
  Effect.gen(function* () {
    const fiberMetrics = yield* Ref.make(HashMap.empty<string, FiberMetrics>())
    const trackerFiberId = yield* Effect.fiberId
    const trackerName = FiberId.threadName(trackerFiberId)
    
    yield* Effect.log(`[${trackerName}] Fiber tracker initialized`)
    
    const registerFiber = (fiberId: FiberId.FiberId) =>
      Effect.gen(function* () {
        const fiberKey = FiberId.threadName(fiberId)
        const now = Date.now()
        
        const metrics: FiberMetrics = {
          fiberId,
          creationTime: now,
          lastActivity: now,
          operationCount: 0,
          status: "active"
        }
        
        yield* Ref.update(fiberMetrics, HashMap.set(fiberKey, metrics))
        yield* Effect.log(`[${trackerName}] Registered fiber: ${fiberKey}`)
      })
    
    const recordActivity = (fiberId: FiberId.FiberId, operation: string) =>
      Effect.gen(function* () {
        const fiberKey = FiberId.threadName(fiberId)
        const now = Date.now()
        
        yield* Ref.update(fiberMetrics, 
          HashMap.modify(fiberKey, (metrics) => ({
            ...metrics,
            lastActivity: now,
            operationCount: metrics.operationCount + 1
          }))
        )
        
        yield* Effect.log(`[${trackerName}] Activity recorded for ${fiberKey}: ${operation}`)
      })
    
    const markCompleted = (fiberId: FiberId.FiberId) =>
      Effect.gen(function* () {
        const fiberKey = FiberId.threadName(fiberId)
        
        yield* Ref.update(fiberMetrics,
          HashMap.modify(fiberKey, (metrics) => ({
            ...metrics,
            status: "completed" as const,
            lastActivity: Date.now()
          }))
        )
        
        yield* Effect.log(`[${trackerName}] Fiber completed: ${fiberKey}`)
      })
    
    const getMetrics = (fiberId?: FiberId.FiberId) =>
      Effect.gen(function* () {
        const allMetrics = yield* Ref.get(fiberMetrics)
        
        if (fiberId) {
          const fiberKey = FiberId.threadName(fiberId)
          const metrics = HashMap.get(allMetrics, fiberKey)
          return metrics._tag === "Some" ? [metrics.value] : []
        }
        
        return HashMap.values(allMetrics)
      })
    
    const getActiveCount = () =>
      Effect.gen(function* () {
        const allMetrics = yield* Ref.get(fiberMetrics)
        return HashMap.values(allMetrics).filter(m => m.status === "active").length
      })
    
    const cleanup = (olderThan: number) =>
      Effect.gen(function* () {
        const cutoff = Date.now() - olderThan
        
        yield* Ref.update(fiberMetrics, (metrics) =>
          HashMap.filter(metrics, (m) => 
            m.status === "active" || m.lastActivity > cutoff
          )
        )
        
        yield* Effect.log(`[${trackerName}] Cleaned up old fiber metrics`)
      })
    
    return {
      registerFiber,
      recordActivity,
      markCompleted,
      getMetrics,
      getActiveCount,
      cleanup
    }
  })

// Example usage with automatic tracking
const createTrackedOperation = <A, E, R>(
  name: string,
  operation: Effect.Effect<A, E, R>
) =>
  Effect.gen(function* () {
    const tracker = yield* createFiberTracker()
    const fiberId = yield* Effect.fiberId
    
    yield* tracker.registerFiber(fiberId)
    yield* tracker.recordActivity(fiberId, `Starting ${name}`)
    
    try {
      const result = yield* operation
      yield* tracker.recordActivity(fiberId, `Completed ${name}`)
      yield* tracker.markCompleted(fiberId)
      return result
    } catch (error) {
      yield* tracker.recordActivity(fiberId, `Failed ${name}: ${error}`)
      yield* tracker.markCompleted(fiberId)
      throw error
    }
  })
```

## Practical Patterns & Best Practices

### Pattern 1: Fiber ID Correlation in Logging

```typescript
import { Effect, FiberId, Logger, LogLevel } from "effect"

// Enhanced logging with fiber correlation
const createCorrelatedLogger = () => {
  const logWithFiber = (level: LogLevel.LogLevel, message: string, data?: unknown) =>
    Effect.gen(function* () {
      const fiberId = yield* Effect.fiberId
      const fiberName = FiberId.threadName(fiberId)
      const timestamp = new Date().toISOString()
      
      const logEntry = {
        timestamp,
        level: level.label,
        fiberId: fiberName,
        message,
        ...(data && { data })
      }
      
      yield* Effect.log(JSON.stringify(logEntry))
    })
  
  return {
    info: (message: string, data?: unknown) => logWithFiber(LogLevel.Info, message, data),
    warn: (message: string, data?: unknown) => logWithFiber(LogLevel.Warning, message, data),
    error: (message: string, data?: unknown) => logWithFiber(LogLevel.Error, message, data),
    debug: (message: string, data?: unknown) => logWithFiber(LogLevel.Debug, message, data)
  }
}

// Usage in business operations
const businessOperationWithLogging = (orderId: string) =>
  Effect.gen(function* () {
    const logger = createCorrelatedLogger()
    
    yield* logger.info("Starting order processing", { orderId })
    
    try {
      yield* logger.debug("Validating order", { orderId })
      const order = yield* validateOrder(orderId)
      
      yield* logger.debug("Processing payment", { orderId, amount: order.total })
      const payment = yield* processPayment(order)
      
      yield* logger.info("Order processed successfully", { 
        orderId, 
        paymentId: payment.id 
      })
      
      return { order, payment }
    } catch (error) {
      yield* logger.error("Order processing failed", { orderId, error })
      return yield* Effect.fail(error)
    }
  })

// Cross-fiber correlation helper
const withCorrelationId = <A, E, R>(
  correlationId: string,
  operation: Effect.Effect<A, E, R>
) =>
  Effect.gen(function* () {
    const fiberId = yield* Effect.fiberId
    const fiberName = FiberId.threadName(fiberId)
    
    yield* Effect.log(`[${correlationId}:${fiberName}] Operation started`)
    
    const result = yield* operation.pipe(
      Effect.tapError((error) => 
        Effect.log(`[${correlationId}:${fiberName}] Operation failed: ${error}`)
      ),
      Effect.tap(() => 
        Effect.log(`[${correlationId}:${fiberName}] Operation completed`)
      )
    )
    
    return result
  })
```

### Pattern 2: Hierarchical Fiber Management

```typescript
import { Effect, Fiber, FiberId, Ref } from "effect"

interface FiberHierarchy {
  readonly fiberId: FiberId.FiberId
  readonly parentId: FiberId.FiberId | null
  readonly children: Set<FiberId.FiberId>
  readonly level: number
  readonly name: string
}

// Hierarchical fiber manager
const createFiberHierarchyManager = () =>
  Effect.gen(function* () {
    const hierarchy = yield* Ref.make(new Map<string, FiberHierarchy>())
    const rootFiberId = yield* Effect.fiberId
    
    // Register root fiber
    const rootKey = FiberId.threadName(rootFiberId)
    const rootHierarchy: FiberHierarchy = {
      fiberId: rootFiberId,
      parentId: null,
      children: new Set(),
      level: 0,
      name: "root"
    }
    
    yield* Ref.update(hierarchy, (h) => h.set(rootKey, rootHierarchy))
    
    const registerChild = (parentFiberId: FiberId.FiberId, childName: string) =>
      Effect.gen(function* () {
        const childFiberId = yield* Effect.fiberId
        const parentKey = FiberId.threadName(parentFiberId)
        const childKey = FiberId.threadName(childFiberId)
        
        const currentHierarchy = yield* Ref.get(hierarchy)
        const parent = currentHierarchy.get(parentKey)
        
        if (parent) {
          // Update parent to include child
          const updatedParent = {
            ...parent,
            children: new Set([...parent.children, childFiberId])
          }
          
          // Create child entry
          const childHierarchy: FiberHierarchy = {
            fiberId: childFiberId,
            parentId: parentFiberId,
            children: new Set(),
            level: parent.level + 1,
            name: childName
          }
          
          yield* Ref.update(hierarchy, (h) => 
            h.set(parentKey, updatedParent).set(childKey, childHierarchy)
          )
          
          yield* Effect.log(`Registered child ${childName} (${childKey}) under parent ${parent.name} (${parentKey})`)
        }
      })
    
    const getHierarchyTree = () =>
      Effect.gen(function* () {
        const currentHierarchy = yield* Ref.get(hierarchy)
        
        const buildTree = (fiberId: FiberId.FiberId, visited = new Set<string>()): any => {
          const key = FiberId.threadName(fiberId)
          
          if (visited.has(key)) return null
          visited.add(key)
          
          const node = currentHierarchy.get(key)
          if (!node) return null
          
          return {
            fiberId: key,
            name: node.name,
            level: node.level,
            children: Array.from(node.children).map(childId => buildTree(childId, visited)).filter(Boolean)
          }
        }
        
        return buildTree(rootFiberId)
      })
    
    const getAncestors = (fiberId: FiberId.FiberId) =>
      Effect.gen(function* () {
        const currentHierarchy = yield* Ref.get(hierarchy)
        const ancestors: FiberHierarchy[] = []
        
        let currentKey = FiberId.threadName(fiberId)
        let current = currentHierarchy.get(currentKey)
        
        while (current && current.parentId) {
          const parentKey = FiberId.threadName(current.parentId)
          const parent = currentHierarchy.get(parentKey)
          if (parent) {
            ancestors.push(parent)
            current = parent
          } else {
            break
          }
        }
        
        return ancestors.reverse()
      })
    
    return {
      registerChild,
      getHierarchyTree,
      getAncestors,
      rootFiberId
    }
  })

// Usage with nested operations
const hierarchicalOperationExample = Effect.gen(function* () {
  const manager = yield* createFiberHierarchyManager()
  
  const parentOperation = Effect.gen(function* () {
    const parentFiberId = yield* Effect.fiberId
    yield* manager.registerChild(manager.rootFiberId, "parent-operation")
    
    // Spawn child operations
    const childFibers = yield* Effect.forEach(
      ["child-1", "child-2", "child-3"],
      (childName) =>
        Effect.fork(
          Effect.gen(function* () {
            yield* manager.registerChild(parentFiberId, childName)
            yield* Effect.sleep("1 second")
            yield* Effect.log(`${childName} completed`)
          })
        ),
      { concurrency: "unbounded" }
    )
    
    yield* Effect.forEach(childFibers, Fiber.join, { concurrency: "unbounded" })
    
    const tree = yield* manager.getHierarchyTree()
    yield* Effect.log(`Hierarchy tree: ${JSON.stringify(tree, null, 2)}`)
  })
  
  yield* Effect.fork(parentOperation).pipe(Effect.flatMap(Fiber.join))
})
```

### Pattern 3: Fiber-Based Circuit Breaker

```typescript
import { Effect, Fiber, FiberId, Ref, Schedule } from "effect"

interface CircuitBreakerState {
  readonly status: "closed" | "open" | "half-open"
  readonly failureCount: number
  readonly lastFailureTime: number
  readonly successCount: number
  readonly activeFibers: Set<string>
}

// Circuit breaker with fiber tracking
const createCircuitBreaker = (
  failureThreshold: number,
  recoveryTimeout: number,
  name: string
) =>
  Effect.gen(function* () {
    const state = yield* Ref.make<CircuitBreakerState>({
      status: "closed",
      failureCount: 0,
      lastFailureTime: 0,
      successCount: 0,
      activeFibers: new Set()
    })
    
    const breakerFiberId = yield* Effect.fiberId
    const breakerName = FiberId.threadName(breakerFiberId)
    
    yield* Effect.log(`[${breakerName}] Circuit breaker ${name} initialized`)
    
    const execute = <A, E, R>(operation: Effect.Effect<A, E, R>) =>
      Effect.gen(function* () {
        const executionFiberId = yield* Effect.fiberId
        const executionName = FiberId.threadName(executionFiberId)
        const currentState = yield* Ref.get(state)
        
        // Check circuit breaker state
        if (currentState.status === "open") {
          const timeSinceFailure = Date.now() - currentState.lastFailureTime
          
          if (timeSinceFailure < recoveryTimeout) {
            yield* Effect.log(`[${executionName}] Circuit breaker ${name} is OPEN - rejecting request`)
            return yield* Effect.fail(new Error(`Circuit breaker ${name} is open`))
          } else {
            // Transition to half-open
            yield* Ref.update(state, (s) => ({ ...s, status: "half-open" as const }))
            yield* Effect.log(`[${breakerName}] Circuit breaker ${name} transitioning to HALF-OPEN`)
          }
        }
        
        // Register active fiber
        yield* Ref.update(state, (s) => ({
          ...s,
          activeFibers: new Set([...s.activeFibers, executionName])
        }))
        
        try {
          yield* Effect.log(`[${executionName}] Executing operation through circuit breaker ${name}`)
          
          const result = yield* operation
          
          // Success - update state
          const updatedState = yield* Ref.modify(state, (s) => {
            const newState: CircuitBreakerState = {
              ...s,
              status: s.status === "half-open" ? "closed" : s.status,
              successCount: s.successCount + 1,
              failureCount: s.status === "half-open" ? 0 : s.failureCount,
              activeFibers: new Set([...s.activeFibers].filter(f => f !== executionName))
            }
            return [newState, newState]
          })
          
          if (updatedState.status === "closed" && currentState.status === "half-open") {
            yield* Effect.log(`[${breakerName}] Circuit breaker ${name} recovered to CLOSED`)
          }
          
          yield* Effect.log(`[${executionName}] Operation succeeded through circuit breaker ${name}`)
          
          return result
          
        } catch (error) {
          // Failure - update state
          const updatedState = yield* Ref.modify(state, (s) => {
            const newFailureCount = s.failureCount + 1
            const shouldOpen = newFailureCount >= failureThreshold
            
            const newState: CircuitBreakerState = {
              ...s,
              status: shouldOpen ? "open" : s.status,
              failureCount: newFailureCount,
              lastFailureTime: shouldOpen ? Date.now() : s.lastFailureTime,
              activeFibers: new Set([...s.activeFibers].filter(f => f !== executionName))
            }
            return [newState, newState]
          })
          
          if (updatedState.status === "open" && currentState.status !== "open") {
            yield* Effect.log(`[${breakerName}] Circuit breaker ${name} OPENED after ${updatedState.failureCount} failures`)
            
            // Cancel remaining active operations
            yield* Effect.gen(function* () {
              const activeCount = updatedState.activeFibers.size
              if (activeCount > 0) {
                yield* Effect.log(`[${breakerName}] Cancelling ${activeCount} active operations`)
                // In a real implementation, you'd track and cancel actual fibers
              }
            })
          }
          
          yield* Effect.log(`[${executionName}] Operation failed through circuit breaker ${name}: ${error}`)
          
          return yield* Effect.fail(error)
        }
      })
    
    const getStatus = () =>
      Effect.gen(function* () {
        const currentState = yield* Ref.get(state)
        return {
          name,
          status: currentState.status,
          failureCount: currentState.failureCount,
          successCount: currentState.successCount,
          activeFiberCount: currentState.activeFibers.size,
          activeFibers: Array.from(currentState.activeFibers)
        }
      })
    
    const reset = () =>
      Effect.gen(function* () {
        yield* Ref.set(state, {
          status: "closed",
          failureCount: 0,
          lastFailureTime: 0,
          successCount: 0,
          activeFibers: new Set()
        })
        
        yield* Effect.log(`[${breakerName}] Circuit breaker ${name} reset`)
      })
    
    return { execute, getStatus, reset }
  })

// Example usage with service calls
const createResilientService = (serviceName: string) =>
  Effect.gen(function* () {
    const circuitBreaker = yield* createCircuitBreaker(5, 30000, serviceName)
    
    const callExternalService = (endpoint: string, data: unknown) =>
      circuitBreaker.execute(
        Effect.gen(function* () {
          // Simulate service call
          yield* Effect.sleep("500 millis")
          
          // Simulate failures (20% chance)
          if (Math.random() < 0.2) {
            return yield* Effect.fail(new Error(`Service ${serviceName} unavailable`))
          }
          
          return { endpoint, data, timestamp: Date.now() }
        })
      )
    
    const healthCheck = () => circuitBreaker.getStatus()
    
    const forceReset = () => circuitBreaker.reset()
    
    return {
      callExternalService,
      healthCheck,
      forceReset
    }
  })
```

## Integration Examples

### Integration with Tracing and Observability

```typescript
import { Effect, FiberId, Tracer } from "effect"

// Custom tracer with fiber correlation
const createFiberAwareTracer = () =>
  Effect.gen(function* () {
    const withFiberSpan = <A, E, R>(
      spanName: string,
      operation: Effect.Effect<A, E, R>
    ) =>
      Effect.gen(function* () {
        const fiberId = yield* Effect.fiberId
        const fiberName = FiberId.threadName(fiberId)
        const spanWithFiber = `${spanName}[${fiberName}]`
        
        return yield* operation.pipe(
          Effect.withSpan(spanWithFiber, {
            attributes: {
              "fiber.id": fiberName,
              "fiber.name": spanName
            }
          })
        )
      })
    
    const traceWithFiberContext = <A, E, R>(
      traceName: string,
      operation: Effect.Effect<A, E, R>
    ) =>
      Effect.gen(function* () {
        const fiberId = yield* Effect.fiberId
        const fiberName = FiberId.threadName(fiberId)
        
        return yield* operation.pipe(
          Effect.withSpan(traceName, {
            attributes: {
              "trace.fiber.id": fiberName,
              "trace.name": traceName
            }
          }),
          Effect.tap(() => Effect.log(`Trace ${traceName} completed in fiber ${fiberName}`))
        )
      })
    
    return { withFiberSpan, traceWithFiberContext }
  })

// Business operation with full tracing
const tracedBusinessOperation = (userId: string, action: string) =>
  Effect.gen(function* () {
    const tracer = yield* createFiberAwareTracer()
    
    return yield* tracer.traceWithFiberContext(
      `user-action-${action}`,
      Effect.gen(function* () {
        yield* tracer.withFiberSpan(
          "validate-user",
          Effect.gen(function* () {
            yield* Effect.sleep("100 millis")
            yield* Effect.log(`Validated user ${userId}`)
          })
        )
        
        yield* tracer.withFiberSpan(
          "execute-action",
          Effect.gen(function* () {
            yield* Effect.sleep("200 millis")
            yield* Effect.log(`Executed action ${action} for user ${userId}`)
          })
        )
        
        return { userId, action, completed: true }
      })
    )
  })
```

### Testing Strategies with FiberId

```typescript
import { Effect, Fiber, FiberId, TestClock, TestContext } from "effect"

// Test utilities for fiber-based operations
const createFiberTestUtils = () => {
  const trackFiberExecution = <A, E, R>(
    name: string,
    operation: Effect.Effect<A, E, R>
  ) =>
    Effect.gen(function* () {
      const fiberId = yield* Effect.fiberId
      const fiberName = FiberId.threadName(fiberId)
      const startTime = Date.now()
      
      console.log(`[TEST] ${name} started in fiber ${fiberName}`)
      
      try {
        const result = yield* operation
        const duration = Date.now() - startTime
        console.log(`[TEST] ${name} completed in ${duration}ms (fiber ${fiberName})`)
        return result
      } catch (error) {
        const duration = Date.now() - startTime
        console.log(`[TEST] ${name} failed in ${duration}ms (fiber ${fiberName}): ${error}`)
        throw error
      }
    })
  
  const createFiberMock = (id: number, behavior: "success" | "failure" | "timeout") =>
    Effect.gen(function* () {
      const fiberId = FiberId.make(id, Date.now())
      
      switch (behavior) {
        case "success":
          return yield* Effect.succeed(`Mock result from fiber ${id}`)
            .pipe(Effect.delay("100 millis"))
            
        case "failure":
          return yield* Effect.fail(`Mock failure from fiber ${id}`)
            .pipe(Effect.delay("50 millis"))
            
        case "timeout":
          return yield* Effect.never
      }
    })
  
  return {
    trackFiberExecution,
    createFiberMock
  }
}

// Example test suite
const fiberTestSuite = Effect.gen(function* () {
  const testUtils = createFiberTestUtils()
  
  // Test 1: Concurrent fiber execution
  yield* testUtils.trackFiberExecution(
    "concurrent-execution-test",
    Effect.gen(function* () {
      const fibers = yield* Effect.forEach(
        [1, 2, 3],
        (id) => Effect.fork(testUtils.createFiberMock(id, "success")),
        { concurrency: "unbounded" }
      )
      
      const results = yield* Effect.forEach(
        fibers,
        (fiber) => Effect.gen(function* () {
          const fiberId = Fiber.id(fiber)
          const result = yield* Fiber.join(fiber)
          return { fiberId: FiberId.threadName(fiberId), result }
        }),
        { concurrency: "unbounded" }
      )
      
      console.log("Concurrent test results:", results)
      return results
    })
  )
  
  // Test 2: Fiber interruption
  yield* testUtils.trackFiberExecution(
    "interruption-test",
    Effect.gen(function* () {
      const fiber = yield* Effect.fork(testUtils.createFiberMock(999, "timeout"))
      const fiberId = Fiber.id(fiber)
      
      yield* Effect.sleep("50 millis")
      yield* Fiber.interrupt(fiber)
      
      console.log(`Interrupted fiber: ${FiberId.threadName(fiberId)}`)
      return "interruption-successful"
    })
  )
  
  return "All tests completed"
})

// Property-based testing with fibers
const fiberPropertyTests = Effect.gen(function* () {
  const testUtils = createFiberTestUtils()
  
  // Property: All forked fibers should have unique IDs
  const uniqueIdProperty = Effect.gen(function* () {
    const fiberCount = 10
    const fibers = yield* Effect.forEach(
      Array.from({ length: fiberCount }, (_, i) => i),
      (id) => Effect.fork(Effect.succeed(id)),
      { concurrency: "unbounded" }
    )
    
    const fiberIds = fibers.map(fiber => FiberId.threadName(Fiber.id(fiber)))
    const uniqueIds = new Set(fiberIds)
    
    console.log(`Created ${fiberCount} fibers, ${uniqueIds.size} unique IDs`)
    console.log("Fiber IDs:", Array.from(uniqueIds))
    
    if (uniqueIds.size !== fiberCount) {
      return yield* Effect.fail("Property violated: Fiber IDs are not unique")
    }
    
    // Clean up
    yield* Effect.forEach(fibers, Fiber.join, { concurrency: "unbounded" })
    
    return "Property satisfied: All fiber IDs are unique"
  })
  
  yield* testUtils.trackFiberExecution("unique-id-property", uniqueIdProperty)
  
  return "Property tests completed"
})
```

## Conclusion

FiberId provides essential fiber identification and tracking capabilities for Effect applications, enabling precise debugging, monitoring, and control of concurrent operations.

Key benefits:
- **Precise Identification**: Every fiber gets a unique identifier for tracking and debugging
- **Relationship Tracking**: Composite FiberIds capture parent-child and sibling relationships
- **Enhanced Observability**: Thread names and fiber correlation improve logging and tracing
- **Targeted Control**: Ability to interrupt, monitor, and manage specific fibers by ID

FiberId is fundamental when building sophisticated concurrent applications that require fine-grained control, comprehensive monitoring, or complex fiber coordination patterns. Use it whenever you need to track individual operations through complex execution flows or build robust concurrent systems with proper observability.