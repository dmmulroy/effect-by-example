# MutableQueue: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem MutableQueue Solves

In synchronous scenarios, developers often need efficient FIFO (First-In-First-Out) data structures for task processing, buffering, and batch operations. Traditional approaches using JavaScript arrays or custom implementations quickly become inefficient and error-prone:

```typescript
// Traditional approach - inefficient array-based queue
class ArrayQueue<T> {
  private items: T[] = []
  private maxCapacity?: number

  constructor(capacity?: number) {
    this.maxCapacity = capacity
  }

  // O(n) operation - shifts all elements
  dequeue(): T | undefined {
    return this.items.shift() // Expensive for large queues!
  }

  // Capacity checking is manual and error-prone
  enqueue(item: T): boolean {
    if (this.maxCapacity && this.items.length >= this.maxCapacity) {
      return false // Queue full, but no clear signal
    }
    this.items.push(item)
    return true
  }

  get size(): number {
    return this.items.length
  }
}

// Usage problems become apparent
const taskQueue = new ArrayQueue<() => void>(100)

// Performance degrades with size due to shift() operations
for (let i = 0; i < 10000; i++) {
  taskQueue.enqueue(() => console.log(`Task ${i}`))
}

// Processing becomes slow
while (taskQueue.size > 0) {
  const task = taskQueue.dequeue() // O(n) operation each time!
  task?.()
}

// Manual batch processing is complex
const batchProcess = (queue: ArrayQueue<string>, batchSize: number): string[][] => {
  const batches: string[][] = []
  let currentBatch: string[] = []
  
  while (queue.size > 0) {
    const item = queue.dequeue()
    if (item) {
      currentBatch.push(item)
      if (currentBatch.length === batchSize) {
        batches.push([...currentBatch]) // More copying!
        currentBatch = []
      }
    }
  }
  
  if (currentBatch.length > 0) {
    batches.push(currentBatch)
  }
  
  return batches
}
```

This approach leads to:
- **Poor performance** - Array.shift() is O(n), making dequeue operations expensive
- **Memory inefficiency** - Arrays resize and copy elements unnecessarily
- **Manual capacity management** - No built-in backpressure handling
- **Complex batch operations** - Extracting multiple items requires manual iteration
- **Error-prone state management** - Easy to introduce bugs with manual index tracking

### The MutableQueue Solution

MutableQueue provides an efficient, mutable FIFO data structure optimized for synchronous operations with proper capacity management and batch processing capabilities:

```typescript
import { MutableQueue } from "effect"

// Efficient queue operations with proper capacity handling
const createTaskProcessor = (capacity: number) => {
  const queue = MutableQueue.bounded<() => void>(capacity)
  
  // O(1) enqueue with built-in capacity checking
  const addTask = (task: () => void): boolean => 
    queue.pipe(MutableQueue.offer(task))
  
  // O(1) dequeue with safe handling
  const processNext = (): boolean => {
    const task = queue.pipe(MutableQueue.poll(MutableQueue.EmptyMutableQueue))
    if (task === MutableQueue.EmptyMutableQueue) {
      return false // No more tasks
    }
    task()
    return true
  }
  
  // Efficient batch processing
  const processBatch = (batchSize: number): number => {
    const tasks = queue.pipe(MutableQueue.pollUpTo(batchSize))
    tasks.forEach(task => task())
    return tasks.length
  }
  
  return { addTask, processNext, processBatch, queue }
}

// Usage is clean and efficient
const processor = createTaskProcessor(1000)

// Adding tasks is fast and handles capacity automatically
for (let i = 0; i < 10000; i++) {
  const success = processor.addTask(() => console.log(`Task ${i}`))
  if (!success) {
    console.log(`Queue full at task ${i}`) // Clear feedback
  }
}

// Processing is O(1) per operation
while (processor.processNext()) {
  // Process tasks one by one efficiently
}

// Batch processing is built-in and efficient
const processedCount = processor.processBatch(50)
console.log(`Processed ${processedCount} tasks in batch`)
```

### Key Concepts

**FIFO Ordering**: Elements are removed in the same order they were added, ensuring predictable processing order.

**Capacity Management**: Bounded queues automatically handle capacity limits with clear success/failure signals for offer operations.

**Efficient Operations**: All core operations (offer, poll) are O(1), making them suitable for high-throughput scenarios.

**Batch Processing**: Built-in support for processing multiple elements at once with `pollUpTo`.

**Mutable Performance**: Unlike immutable collections, MutableQueue is optimized for scenarios where mutation is acceptable for performance gains.

## Basic Usage Patterns

### Pattern 1: Creating Queues

```typescript
import { MutableQueue } from "effect"

// Bounded queue with capacity limit
const boundedQueue = MutableQueue.bounded<string>(100)

// Unbounded queue (limited only by available memory)
const unboundedQueue = MutableQueue.unbounded<number>()

// Check queue properties
const maxCapacity = boundedQueue.pipe(MutableQueue.capacity) // 100
const unlimitedCapacity = unboundedQueue.pipe(MutableQueue.capacity) // Infinity
```

### Pattern 2: Basic Operations

```typescript
import { MutableQueue } from "effect"

const queue = MutableQueue.bounded<string>(5)

// Adding elements (offer)
const success1 = queue.pipe(MutableQueue.offer("first"))  // true
const success2 = queue.pipe(MutableQueue.offer("second")) // true

// Check queue state
const currentLength = queue.pipe(MutableQueue.length)     // 2
const isEmpty = queue.pipe(MutableQueue.isEmpty)          // false
const isFull = queue.pipe(MutableQueue.isFull)           // false

// Removing elements (poll)
const firstItem = queue.pipe(MutableQueue.poll("default")) // "first"
const secondItem = queue.pipe(MutableQueue.poll("default")) // "second"
const emptyResult = queue.pipe(MutableQueue.poll("nothing")) // "nothing"
```

### Pattern 3: Batch Operations

```typescript
import { MutableQueue, Chunk } from "effect"

const queue = MutableQueue.bounded<number>(10)

// Adding multiple elements
const values = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12]
const rejected = queue.pipe(MutableQueue.offerAll(values))
// Queue takes first 10, rejected contains [11, 12]

console.log("Rejected items:", rejected) // Chunk containing [11, 12]
console.log("Queue is full:", queue.pipe(MutableQueue.isFull)) // true

// Removing multiple elements
const batch = queue.pipe(MutableQueue.pollUpTo(3)) // Chunk containing [1, 2, 3]
const remaining = queue.pipe(MutableQueue.length)   // 7
```

## Real-World Examples

### Example 1: Task Processing System

Task queues are fundamental to many applications - from background job processing to UI event handling. Here's a complete task processing system:

```typescript
import { MutableQueue, Chunk } from "effect"

// Define task types
interface Task {
  readonly id: string
  readonly type: 'email' | 'image' | 'report'
  readonly priority: 'high' | 'normal' | 'low'
  readonly payload: unknown
  readonly createdAt: Date
}

interface TaskResult {
  readonly taskId: string
  readonly success: boolean
  readonly processedAt: Date
  readonly duration: number
  readonly error?: string
}

// Task processor with priority handling
const createTaskProcessor = (capacity: number = 1000) => {
  const highPriorityQueue = MutableQueue.bounded<Task>(Math.floor(capacity * 0.3))
  const normalPriorityQueue = MutableQueue.bounded<Task>(Math.floor(capacity * 0.5))
  const lowPriorityQueue = MutableQueue.bounded<Task>(Math.floor(capacity * 0.2))
  
  const submitTask = (task: Task): boolean => {
    const targetQueue = task.priority === 'high' ? highPriorityQueue :
                       task.priority === 'normal' ? normalPriorityQueue :
                       lowPriorityQueue
    
    return targetQueue.pipe(MutableQueue.offer(task))
  }
  
  const getNextTask = (): Task | null => {
    // Process high priority first
    let task = highPriorityQueue.pipe(MutableQueue.poll(MutableQueue.EmptyMutableQueue))
    if (task !== MutableQueue.EmptyMutableQueue) return task
    
    // Then normal priority
    task = normalPriorityQueue.pipe(MutableQueue.poll(MutableQueue.EmptyMutableQueue))
    if (task !== MutableQueue.EmptyMutableQueue) return task
    
    // Finally low priority
    task = lowPriorityQueue.pipe(MutableQueue.poll(MutableQueue.EmptyMutableQueue))
    return task === MutableQueue.EmptyMutableQueue ? null : task
  }
  
  const processBatch = async (batchSize: number): Promise<TaskResult[]> => {
    const results: TaskResult[] = []
    
    for (let i = 0; i < batchSize; i++) {
      const task = getNextTask()
      if (!task) break
      
      const startTime = Date.now()
      try {
        await processTask(task)
        results.push({
          taskId: task.id,
          success: true,
          processedAt: new Date(),
          duration: Date.now() - startTime
        })
      } catch (error) {
        results.push({
          taskId: task.id,
          success: false,
          processedAt: new Date(),
          duration: Date.now() - startTime,
          error: error instanceof Error ? error.message : 'Unknown error'
        })
      }
    }
    
    return results
  }
  
  const getQueueStats = () => ({
    high: {
      length: highPriorityQueue.pipe(MutableQueue.length),
      capacity: highPriorityQueue.pipe(MutableQueue.capacity),
      isFull: highPriorityQueue.pipe(MutableQueue.isFull)
    },
    normal: {
      length: normalPriorityQueue.pipe(MutableQueue.length),
      capacity: normalPriorityQueue.pipe(MutableQueue.capacity),
      isFull: normalPriorityQueue.pipe(MutableQueue.isFull)
    },
    low: {
      length: lowPriorityQueue.pipe(MutableQueue.length),
      capacity: lowPriorityQueue.pipe(MutableQueue.capacity),
      isFull: lowPriorityQueue.pipe(MutableQueue.isFull)
    }
  })
  
  return { submitTask, getNextTask, processBatch, getQueueStats }
}

// Mock task processor
const processTask = async (task: Task): Promise<void> => {
  const processingTime = task.type === 'image' ? 1000 :
                        task.type === 'report' ? 500 :
                        100
  
  await new Promise(resolve => setTimeout(resolve, processingTime))
  console.log(`Processed ${task.type} task ${task.id}`)
}

// Usage example
const processor = createTaskProcessor(1000)

// Submit various tasks
const tasks: Task[] = [
  {
    id: '1',
    type: 'email',
    priority: 'high',
    payload: { to: 'user@example.com', subject: 'Welcome' },
    createdAt: new Date()
  },
  {
    id: '2',
    type: 'image',
    priority: 'normal',
    payload: { imageUrl: '/path/to/image.jpg', size: 'thumbnail' },
    createdAt: new Date()
  },
  {
    id: '3',
    type: 'report',
    priority: 'low',
    payload: { reportType: 'monthly', userId: '123' },
    createdAt: new Date()
  }
]

// Submit tasks
tasks.forEach(task => {
  const success = processor.submitTask(task)
  console.log(`Task ${task.id} ${success ? 'queued' : 'rejected'}`)
})

// Process tasks in batches
const processInBatches = async () => {
  while (true) {
    const results = await processor.processBatch(5)
    if (results.length === 0) break
    
    console.log(`Processed batch of ${results.length} tasks`)
    console.log('Queue stats:', processor.getQueueStats())
  }
}

processInBatches()
```

### Example 2: Real-Time Data Buffering

Buffering streaming data for batch processing is common in analytics, logging, and data pipeline scenarios:

```typescript
import { MutableQueue, Chunk } from "effect"

interface DataPoint {
  readonly timestamp: number
  readonly value: number
  readonly source: string
  readonly metadata?: Record<string, unknown>
}

interface BatchProcessor<T> {
  readonly add: (item: T) => boolean
  readonly flush: () => T[]
  readonly forceFlush: () => T[]
  readonly getStats: () => BufferStats
}

interface BufferStats {
  readonly currentSize: number
  readonly capacity: number
  readonly totalProcessed: number
  readonly totalDropped: number
  readonly lastFlushTime: Date | null
}

// Generic batch buffer with automatic flushing
const createBatchBuffer = <T>(
  capacity: number,
  flushSize: number,
  flushInterval: number,
  onBatch: (items: T[]) => Promise<void>
): BatchProcessor<T> => {
  const buffer = MutableQueue.bounded<T>(capacity)
  let totalProcessed = 0
  let totalDropped = 0
  let lastFlushTime: Date | null = null
  let flushTimer: NodeJS.Timeout | null = null
  
  const flush = (): T[] => {
    const items = buffer.pipe(MutableQueue.pollUpTo(flushSize))
    const itemsArray = Chunk.toReadonlyArray(items)
    
    if (itemsArray.length > 0) {
      totalProcessed += itemsArray.length
      lastFlushTime = new Date()
      
      // Process batch asynchronously
      onBatch(itemsArray).catch(error => {
        console.error('Batch processing failed:', error)
      })
    }
    
    return itemsArray
  }
  
  const forceFlush = (): T[] => {
    const remainingItems = buffer.pipe(MutableQueue.pollUpTo(buffer.pipe(MutableQueue.length)))
    const itemsArray = Chunk.toReadonlyArray(remainingItems)
    
    if (itemsArray.length > 0) {
      totalProcessed += itemsArray.length
      lastFlushTime = new Date()
      
      onBatch(itemsArray).catch(error => {
        console.error('Force flush processing failed:', error)
      })
    }
    
    return itemsArray
  }
  
  const resetFlushTimer = () => {
    if (flushTimer) {
      clearTimeout(flushTimer)
    }
    
    flushTimer = setTimeout(() => {
      if (!buffer.pipe(MutableQueue.isEmpty)) {
        flush()
      }
      resetFlushTimer()
    }, flushInterval)
  }
  
  const add = (item: T): boolean => {
    const success = buffer.pipe(MutableQueue.offer(item))
    
    if (!success) {
      totalDropped++
      return false
    }
    
    // Auto-flush if buffer reaches flush size
    if (buffer.pipe(MutableQueue.length) >= flushSize) {
      flush()
    }
    
    // Reset timer on first item
    if (buffer.pipe(MutableQueue.length) === 1) {
      resetFlushTimer()
    }
    
    return true
  }
  
  const getStats = (): BufferStats => ({
    currentSize: buffer.pipe(MutableQueue.length),
    capacity: buffer.pipe(MutableQueue.capacity),
    totalProcessed,
    totalDropped,
    lastFlushTime
  })
  
  // Start flush timer
  resetFlushTimer()
  
  return { add, flush, forceFlush, getStats }
}

// Usage for analytics data
const analyticsBuffer = createBatchBuffer<DataPoint>(
  10000,  // capacity
  100,    // flush size
  5000,   // flush interval (5 seconds)
  async (dataPoints: DataPoint[]) => {
    // Simulate sending to analytics service
    console.log(`Sending ${dataPoints.length} data points to analytics service`)
    
    // Group by source for efficient processing
    const grouped = dataPoints.reduce((acc, point) => {
      if (!acc[point.source]) {
        acc[point.source] = []
      }
      acc[point.source].push(point)
      return acc
    }, {} as Record<string, DataPoint[]>)
    
    // Process each source group
    for (const [source, points] of Object.entries(grouped)) {
      await sendToAnalytics(source, points)
    }
  }
)

const sendToAnalytics = async (source: string, points: DataPoint[]): Promise<void> => {
  // Simulate API call
  await new Promise(resolve => setTimeout(resolve, 100))
  console.log(`Sent ${points.length} points from ${source}`)
}

// Simulate real-time data generation
const generateDataPoints = () => {
  const sources = ['web-app', 'mobile-app', 'api-gateway', 'background-jobs']
  
  setInterval(() => {
    const dataPoint: DataPoint = {
      timestamp: Date.now(),
      value: Math.random() * 100,
      source: sources[Math.floor(Math.random() * sources.length)],
      metadata: {
        version: '1.0',
        environment: 'production'
      }
    }
    
    const success = analyticsBuffer.add(dataPoint)
    if (!success) {
      console.warn('Data point dropped - buffer full')
    }
  }, 10) // Generate data every 10ms
}

// Monitor buffer stats
setInterval(() => {
  const stats = analyticsBuffer.getStats()
  console.log('Buffer stats:', stats)
}, 10000) // Log stats every 10 seconds

// Start data generation
generateDataPoints()

// Graceful shutdown
process.on('SIGINT', () => {
  console.log('Shutting down, flushing remaining data...')
  analyticsBuffer.forceFlush()
  setTimeout(() => process.exit(0), 1000)
})
```

### Example 3: Request Rate Limiting and Throttling

Rate limiting is crucial for API stability and fair resource usage. Here's a sophisticated rate limiter using MutableQueue:

```typescript
import { MutableQueue } from "effect"

interface RateLimitedRequest {
  readonly id: string
  readonly timestamp: number
  readonly priority: number
  readonly execute: () => Promise<unknown>
  readonly onComplete: (result: unknown) => void
  readonly onError: (error: Error) => void
}

interface RateLimitConfig {
  readonly requestsPerSecond: number
  readonly burstCapacity: number
  readonly queueCapacity: number
  readonly priorityLevels: number
}

interface RateLimiterStats {
  readonly queuedRequests: number
  readonly processedRequests: number
  readonly droppedRequests: number
  readonly currentRatePerSecond: number
  readonly averageProcessingTime: number
}

const createRateLimiter = (config: RateLimitConfig) => {
  // Separate queues for different priority levels
  const queues = Array.from({ length: config.priorityLevels }, () =>
    MutableQueue.bounded<RateLimitedRequest>(Math.floor(config.queueCapacity / config.priorityLevels))
  )
  
  let processedRequests = 0
  let droppedRequests = 0
  let totalProcessingTime = 0
  let recentRequestTimes: number[] = []
  
  const intervalMs = 1000 / config.requestsPerSecond
  let lastProcessTime = 0
  let tokenBucket = config.burstCapacity
  
  const addRequest = (request: RateLimitedRequest): boolean => {
    const priorityIndex = Math.min(request.priority, config.priorityLevels - 1)
    const success = queues[priorityIndex].pipe(MutableQueue.offer(request))
    
    if (!success) {
      droppedRequests++
    }
    
    return success
  }
  
  const getNextRequest = (): RateLimitedRequest | null => {
    // Process higher priority queues first
    for (let i = config.priorityLevels - 1; i >= 0; i--) {
      const request = queues[i].pipe(MutableQueue.poll(MutableQueue.EmptyMutableQueue))
      if (request !== MutableQueue.EmptyMutableQueue) {
        return request
      }
    }
    return null
  }
  
  const canProcess = (): boolean => {
    const now = Date.now()
    
    // Refill token bucket based on time elapsed
    const timeSinceLastProcess = now - lastProcessTime
    if (timeSinceLastProcess > 0) {
      const tokensToAdd = Math.floor(timeSinceLastProcess / intervalMs)
      tokenBucket = Math.min(config.burstCapacity, tokenBucket + tokensToAdd)
      lastProcessTime = now
    }
    
    return tokenBucket > 0
  }
  
  const processRequests = async () => {
    if (!canProcess()) {
      return
    }
    
    const request = getNextRequest()
    if (!request) {
      return
    }
    
    tokenBucket--
    const startTime = Date.now()
    
    try {
      const result = await request.execute()
      const processingTime = Date.now() - startTime
      
      totalProcessingTime += processingTime
      processedRequests++
      
      // Track recent request times for rate calculation
      recentRequestTimes.push(Date.now())
      if (recentRequestTimes.length > 100) {
        recentRequestTimes = recentRequestTimes.slice(-50)
      }
      
      request.onComplete(result)
    } catch (error) {
      const processingTime = Date.now() - startTime
      totalProcessingTime += processingTime
      processedRequests++
      
      request.onError(error instanceof Error ? error : new Error(String(error)))
    }
  }
  
  const getStats = (): RateLimiterStats => {
    const totalQueued = queues.reduce((sum, queue) => 
      sum + queue.pipe(MutableQueue.length), 0)
    
    // Calculate current rate from recent requests
    const now = Date.now()
    const recentRequests = recentRequestTimes.filter(time => now - time < 10000)
    const currentRate = recentRequests.length / 10 // requests per second over last 10 seconds
    
    const averageProcessingTime = processedRequests > 0 ? 
      totalProcessingTime / processedRequests : 0
    
    return {
      queuedRequests: totalQueued,
      processedRequests,
      droppedRequests,
      currentRatePerSecond: currentRate,
      averageProcessingTime
    }
  }
  
  // Start processing loop
  const processingLoop = setInterval(processRequests, Math.min(intervalMs / 2, 10))
  
  const shutdown = () => {
    clearInterval(processingLoop)
  }
  
  return { addRequest, getStats, shutdown }
}

// Usage example
const rateLimiter = createRateLimiter({
  requestsPerSecond: 10,
  burstCapacity: 20,
  queueCapacity: 1000,
  priorityLevels: 3
})

// Helper to create rate-limited requests
const createApiRequest = (
  id: string, 
  priority: number, 
  apiCall: () => Promise<unknown>
): Promise<unknown> => {
  return new Promise((resolve, reject) => {
    const request: RateLimitedRequest = {
      id,
      timestamp: Date.now(),
      priority,
      execute: apiCall,
      onComplete: resolve,
      onError: reject
    }
    
    const success = rateLimiter.addRequest(request)
    if (!success) {
      reject(new Error(`Request ${id} was dropped - queue full`))
    }
  })
}

// Simulate high-priority user requests
const makeUserRequest = async (userId: string) => {
  try {
    const result = await createApiRequest(
      `user-${userId}`,
      2, // High priority
      async () => {
        // Simulate API call
        await new Promise(resolve => setTimeout(resolve, 100))
        return { userId, data: 'user data' }
      }
    )
    console.log('User request completed:', result)
  } catch (error) {
    console.error('User request failed:', error)
  }
}

// Simulate background batch requests
const makeBatchRequest = async (batchId: string) => {
  try {
    const result = await createApiRequest(
      `batch-${batchId}`,
      0, // Low priority
      async () => {
        // Simulate longer API call
        await new Promise(resolve => setTimeout(resolve, 500))
        return { batchId, processed: 100 }
      }
    )
    console.log('Batch request completed:', result)
  } catch (error) {
    console.error('Batch request failed:', error)
  }
}

// Generate mixed traffic
const generateTraffic = () => {
  // High-priority user requests
  setInterval(() => {
    const userId = Math.floor(Math.random() * 1000).toString()
    makeUserRequest(userId)
  }, 50)
  
  // Low-priority batch requests
  setInterval(() => {
    const batchId = Math.floor(Math.random() * 100).toString()
    makeBatchRequest(batchId)
  }, 200)
}

// Monitor rate limiter stats
setInterval(() => {
  const stats = rateLimiter.getStats()
  console.log('Rate Limiter Stats:', {
    ...stats,
    currentRatePerSecond: Math.round(stats.currentRatePerSecond * 100) / 100,
    averageProcessingTime: Math.round(stats.averageProcessingTime)
  })
}, 5000)

// Start traffic generation
generateTraffic()

// Graceful shutdown
process.on('SIGINT', () => {
  console.log('Shutting down rate limiter...')
  rateLimiter.shutdown()
  process.exit(0)
})
```

## Advanced Features Deep Dive

### Feature 1: Capacity Management and Backpressure

MutableQueue provides sophisticated capacity management that's essential for building robust systems that handle varying load conditions.

#### Basic Capacity Usage

```typescript
import { MutableQueue } from "effect"

const queue = MutableQueue.bounded<string>(3)

// Check capacity and state
console.log('Capacity:', queue.pipe(MutableQueue.capacity))     // 3
console.log('Is empty:', queue.pipe(MutableQueue.isEmpty))      // true
console.log('Is full:', queue.pipe(MutableQueue.isFull))        // false

// Fill the queue
queue.pipe(MutableQueue.offer("first"))
queue.pipe(MutableQueue.offer("second"))
queue.pipe(MutableQueue.offer("third"))

console.log('Is full now:', queue.pipe(MutableQueue.isFull))    // true

// Attempt to add beyond capacity
const success = queue.pipe(MutableQueue.offer("fourth"))
console.log('Addition successful:', success)                    // false
```

#### Real-World Backpressure Example

```typescript
import { MutableQueue } from "effect"

interface BackpressureHandler<T> {
  readonly offer: (item: T) => 'accepted' | 'rejected' | 'dropped_oldest'
  readonly getMetrics: () => BackpressureMetrics
}

interface BackpressureMetrics {
  readonly capacity: number
  readonly currentSize: number
  readonly totalOffered: number
  readonly totalAccepted: number
  readonly totalRejected: number
  readonly totalDropped: number
}

const createBackpressureHandler = <T>(
  capacity: number,
  strategy: 'reject' | 'drop_oldest' = 'reject'
): BackpressureHandler<T> => {
  const queue = MutableQueue.bounded<T>(capacity)
  let totalOffered = 0
  let totalAccepted = 0
  let totalRejected = 0
  let totalDropped = 0
  
  const offer = (item: T): 'accepted' | 'rejected' | 'dropped_oldest' => {
    totalOffered++
    
    const success = queue.pipe(MutableQueue.offer(item))
    if (success) {
      totalAccepted++
      return 'accepted'
    }
    
    if (strategy === 'drop_oldest') {
      // Remove oldest item to make space
      const dropped = queue.pipe(MutableQueue.poll(MutableQueue.EmptyMutableQueue))
      if (dropped !== MutableQueue.EmptyMutableQueue) {
        totalDropped++
        // Now add the new item
        const retrySuccess = queue.pipe(MutableQueue.offer(item))
        if (retrySuccess) {
          totalAccepted++
          return 'dropped_oldest'
        }
      }
    }
    
    totalRejected++
    return 'rejected'
  }
  
  const getMetrics = (): BackpressureMetrics => ({
    capacity: queue.pipe(MutableQueue.capacity),
    currentSize: queue.pipe(MutableQueue.length),
    totalOffered,
    totalAccepted,
    totalRejected,
    totalDropped
  })
  
  return { offer, getMetrics }
}

// Example: HTTP request buffer with backpressure
const requestBuffer = createBackpressureHandler<{ url: string, timestamp: number }>(
  100,
  'drop_oldest'
)

// Simulate high request load
const simulateRequests = () => {
  for (let i = 0; i < 150; i++) {
    const result = requestBuffer.offer({
      url: `/api/data/${i}`,
      timestamp: Date.now()
    })
    
    if (result !== 'accepted') {
      console.log(`Request ${i}: ${result}`)
    }
  }
  
  console.log('Final metrics:', requestBuffer.getMetrics())
}

simulateRequests()
```

### Feature 2: Efficient Batch Processing with pollUpTo

The `pollUpTo` function is designed for efficient batch processing, allowing you to extract multiple items in a single operation.

#### Basic Batch Processing

```typescript
import { MutableQueue, Chunk } from "effect"

const queue = MutableQueue.unbounded<number>()

// Add items to queue
for (let i = 1; i <= 20; i++) {
  queue.pipe(MutableQueue.offer(i))
}

// Process in batches of 5
while (!queue.pipe(MutableQueue.isEmpty)) {
  const batch = queue.pipe(MutableQueue.pollUpTo(5))
  const items = Chunk.toReadonlyArray(batch)
  console.log('Processing batch:', items)
  
  // Process each item in the batch
  items.forEach(item => {
    // Simulate processing
    console.log(`  Processing item ${item}`)
  })
}
```

#### Advanced Batch Processing: Adaptive Batch Sizes

```typescript
import { MutableQueue, Chunk } from "effect"

interface AdaptiveBatcher<T> {
  readonly addItem: (item: T) => void
  readonly processBatch: () => Promise<number>
  readonly getStats: () => BatchStats
}

interface BatchStats {
  readonly queueSize: number
  readonly avgBatchSize: number
  readonly totalBatches: number
  readonly totalProcessed: number
  readonly processingRate: number
}

const createAdaptiveBatcher = <T>(
  maxCapacity: number,
  minBatchSize: number,
  maxBatchSize: number,
  processor: (items: T[]) => Promise<void>
): AdaptiveBatcher<T> => {
  const queue = MutableQueue.bounded<T>(maxCapacity)
  let totalBatches = 0
  let totalProcessed = 0
  let totalBatchSize = 0
  let lastProcessTime = Date.now()
  let recentProcessingTimes: number[] = []
  
  const addItem = (item: T): void => {
    const success = queue.pipe(MutableQueue.offer(item))
    if (!success) {
      console.warn('Item dropped - queue at capacity')
    }
  }
  
  const calculateOptimalBatchSize = (): number => {
    const queueSize = queue.pipe(MutableQueue.length)
    
    // Start with queue-based sizing
    let batchSize = Math.min(maxBatchSize, Math.max(minBatchSize, Math.floor(queueSize / 2)))
    
    // Adjust based on recent processing performance
    if (recentProcessingTimes.length > 0) {
      const avgProcessingTime = recentProcessingTimes.reduce((a, b) => a + b) / recentProcessingTimes.length
      
      // If processing is slow, reduce batch size
      if (avgProcessingTime > 1000) { // > 1 second
        batchSize = Math.max(minBatchSize, Math.floor(batchSize * 0.7))
      }
      // If processing is fast, increase batch size
      else if (avgProcessingTime < 100) { // < 100ms
        batchSize = Math.min(maxBatchSize, Math.floor(batchSize * 1.3))
      }
    }
    
    return Math.min(batchSize, queueSize)
  }
  
  const processBatch = async (): Promise<number> => {
    const batchSize = calculateOptimalBatchSize()
    if (batchSize === 0) return 0
    
    const batch = queue.pipe(MutableQueue.pollUpTo(batchSize))
    const items = Chunk.toReadonlyArray(batch)
    
    if (items.length === 0) return 0
    
    const startTime = Date.now()
    
    try {
      await processor(items)
      
      const processingTime = Date.now() - startTime
      recentProcessingTimes.push(processingTime)
      
      // Keep only recent times for calculation
      if (recentProcessingTimes.length > 10) {
        recentProcessingTimes = recentProcessingTimes.slice(-5)
      }
      
      totalBatches++
      totalProcessed += items.length
      totalBatchSize += items.length
      
      console.log(`Processed batch of ${items.length} items in ${processingTime}ms`)
      
    } catch (error) {
      console.error('Batch processing failed:', error)
      // Re-queue items on failure (optional)
      items.forEach(item => queue.pipe(MutableQueue.offer(item)))
      throw error
    }
    
    return items.length
  }
  
  const getStats = (): BatchStats => {
    const now = Date.now()
    const timeSinceLastProcess = now - lastProcessTime
    const processingRate = timeSinceLastProcess > 0 ? 
      (totalProcessed * 1000) / timeSinceLastProcess : 0
    
    return {
      queueSize: queue.pipe(MutableQueue.length),
      avgBatchSize: totalBatches > 0 ? totalBatchSize / totalBatches : 0,
      totalBatches,
      totalProcessed,
      processingRate
    }
  }
  
  return { addItem, processBatch, getStats }
}

// Example: Log processing with adaptive batching
const logProcessor = createAdaptiveBatcher<string>(
  10000,  // max capacity
  10,     // min batch size
  100,    // max batch size
  async (logs: string[]) => {
    // Simulate variable processing time based on batch size
    const processingTime = logs.length * 10 + Math.random() * 200
    await new Promise(resolve => setTimeout(resolve, processingTime))
    
    console.log(`Sent ${logs.length} logs to external service`)
  }
)

// Simulate variable log generation
const generateLogs = () => {
  let logCount = 0
  
  const generateBurst = () => {
    const burstSize = Math.floor(Math.random() * 50) + 10
    for (let i = 0; i < burstSize; i++) {
      logProcessor.addItem(`Log entry ${++logCount}: ${new Date().toISOString()}`)
    }
  }
  
  // Random bursts of logs
  setInterval(generateBurst, Math.random() * 2000 + 500)
  
  // Continuous processing
  setInterval(async () => {
    await logProcessor.processBatch()
  }, 100)
  
  // Periodic stats
  setInterval(() => {
    console.log('Adaptive Batcher Stats:', logProcessor.getStats())
  }, 5000)
}

generateLogs()
```

### Feature 3: Performance Characteristics and Optimization

Understanding MutableQueue's performance characteristics helps you choose the right tool for your use case.

#### Performance Comparison

```typescript
import { MutableQueue } from "effect"

// Performance test utilities
const measureTime = <T>(operation: () => T, iterations: number = 1): [T, number] => {
  const start = process.hrtime.bigint()
  let result: T
  
  for (let i = 0; i < iterations; i++) {
    result = operation()
  }
  
  const end = process.hrtime.bigint()
  const timeMs = Number(end - start) / 1_000_000 / iterations
  
  return [result!, timeMs]
}

const performanceComparison = () => {
  const sizes = [100, 1000, 10000, 100000]
  
  console.log('Performance Comparison: MutableQueue vs Array')
  console.log('Size\t\tMutableQueue (ms)\tArray (ms)\t\tRatio')
  console.log('----\t\t-----------------\t----------\t\t-----')
  
  sizes.forEach(size => {
    // MutableQueue performance
    const mutableQueue = MutableQueue.unbounded<number>()
    
    const [, mqOfferTime] = measureTime(() => {
      for (let i = 0; i < size; i++) {
        mutableQueue.pipe(MutableQueue.offer(i))
      }
    })
    
    const [, mqPollTime] = measureTime(() => {
      for (let i = 0; i < size; i++) {
        mutableQueue.pipe(MutableQueue.poll(-1))
      }
    })
    
    const mqTotalTime = mqOfferTime + mqPollTime
    
    // Array performance (using shift which is O(n))
    let array: number[] = []
    
    const [, arrayPushTime] = measureTime(() => {
      for (let i = 0; i < size; i++) {
        array.push(i)
      }
    })
    
    const [, arrayShiftTime] = measureTime(() => {
      for (let i = 0; i < size; i++) {
        array.shift()
      }
    })
    
    const arrayTotalTime = arrayPushTime + arrayShiftTime
    const ratio = arrayTotalTime / mqTotalTime
    
    console.log(`${size}\t\t${mqTotalTime.toFixed(3)}\t\t\t${arrayTotalTime.toFixed(3)}\t\t\t${ratio.toFixed(2)}x`)
  })
}

// Memory usage comparison
const memoryComparison = () => {
  const size = 100000
  
  console.log('\nMemory Usage Comparison:')
  
  // Measure initial memory
  const initialMemory = process.memoryUsage().heapUsed
  
  // MutableQueue memory usage
  const mutableQueue = MutableQueue.unbounded<{ id: number, data: string }>()
  
  for (let i = 0; i < size; i++) {
    mutableQueue.pipe(MutableQueue.offer({
      id: i,
      data: `Item ${i} with some data content`
    }))
  }
  
  const mqMemory = process.memoryUsage().heapUsed
  
  // Array memory usage
  const array: { id: number, data: string }[] = []
  
  for (let i = 0; i < size; i++) {
    array.push({
      id: i,
      data: `Item ${i} with some data content`
    })
  }
  
  const arrayMemory = process.memoryUsage().heapUsed
  
  console.log(`MutableQueue: ${((mqMemory - initialMemory) / 1024 / 1024).toFixed(2)} MB`)
  console.log(`Array: ${((arrayMemory - mqMemory) / 1024 / 1024).toFixed(2)} MB`)
}

// Run performance tests
performanceComparison()
memoryComparison()
```

## Practical Patterns & Best Practices

### Pattern 1: Queue Monitoring and Health Checks

```typescript
import { MutableQueue } from "effect"

interface QueueHealth {
  readonly healthy: boolean
  readonly utilizationPercent: number
  readonly throughputPerSecond: number
  readonly avgProcessingTime: number
  readonly warnings: string[]
}

const createQueueMonitor = <T>(
  queue: MutableQueue<T>,
  name: string,
  thresholds: {
    readonly utilizationWarning: number  // 0.0 - 1.0
    readonly utilizationCritical: number // 0.0 - 1.0
    readonly minThroughput: number       // items/second
  }
) => {
  let totalProcessed = 0
  let totalProcessingTime = 0
  let recentProcessTimes: number[] = []
  const startTime = Date.now()
  
  const recordProcessing = (processingTimeMs: number) => {
    totalProcessed++
    totalProcessingTime += processingTimeMs
    recentProcessTimes.push(Date.now())
    
    // Keep only last 100 processing times
    if (recentProcessTimes.length > 100) {
      recentProcessTimes = recentProcessTimes.slice(-50)
    }
  }
  
  const getHealth = (): QueueHealth => {
    const capacity = queue.pipe(MutableQueue.capacity)
    const currentSize = queue.pipe(MutableQueue.length)
    const utilization = capacity === Infinity ? 0 : currentSize / capacity
    
    // Calculate throughput from recent processing times
    const now = Date.now()
    const recentItems = recentProcessTimes.filter(time => now - time < 60000) // last minute
    const throughput = recentItems.length / 60 // per second
    
    const avgProcessingTime = totalProcessed > 0 ? 
      totalProcessingTime / totalProcessed : 0
    
    const warnings: string[] = []
    let healthy = true
    
    // Check utilization thresholds
    if (utilization >= thresholds.utilizationCritical) {
      warnings.push(`CRITICAL: Queue utilization at ${(utilization * 100).toFixed(1)}%`)
      healthy = false
    } else if (utilization >= thresholds.utilizationWarning) {
      warnings.push(`WARNING: Queue utilization at ${(utilization * 100).toFixed(1)}%`)
    }
    
    // Check throughput
    if (throughput < thresholds.minThroughput && totalProcessed > 10) {
      warnings.push(`WARNING: Low throughput (${throughput.toFixed(2)}/sec, expected ${thresholds.minThroughput}/sec)`)
    }
    
    // Check if queue is full
    if (queue.pipe(MutableQueue.isFull)) {
      warnings.push('CRITICAL: Queue is full - items may be dropped')
      healthy = false
    }
    
    return {
      healthy,
      utilizationPercent: utilization * 100,
      throughputPerSecond: throughput,
      avgProcessingTime,
      warnings
    }
  }
  
  const startHealthChecks = (intervalMs: number = 30000) => {
    setInterval(() => {
      const health = getHealth()
      
      console.log(`[${name}] Health Check:`, {
        healthy: health.healthy,
        utilization: `${health.utilizationPercent.toFixed(1)}%`,
        throughput: `${health.throughputPerSecond.toFixed(2)}/sec`,
        avgProcessingTime: `${health.avgProcessingTime.toFixed(2)}ms`,
        queueSize: queue.pipe(MutableQueue.length),
        capacity: queue.pipe(MutableQueue.capacity)
      })
      
      if (health.warnings.length > 0) {
        health.warnings.forEach(warning => console.warn(`[${name}] ${warning}`))
      }
      
      if (!health.healthy) {
        console.error(`[${name}] Queue is unhealthy!`)
      }
    }, intervalMs)
  }
  
  return { recordProcessing, getHealth, startHealthChecks }
}

// Usage example
const taskQueue = MutableQueue.bounded<() => Promise<void>>(1000)
const monitor = createQueueMonitor(taskQueue, 'TaskProcessor', {
  utilizationWarning: 0.75,
  utilizationCritical: 0.90,
  minThroughput: 10
})

// Start health monitoring
monitor.startHealthChecks(10000) // Check every 10 seconds

const processTask = async (task: () => Promise<void>) => {
  const startTime = Date.now()
  try {
    await task()
  } finally {
    monitor.recordProcessing(Date.now() - startTime)
  }
}
```

### Pattern 2: Graceful Shutdown and Resource Cleanup

```typescript
import { MutableQueue } from "effect"

interface GracefulQueue<T> {
  readonly offer: (item: T) => boolean
  readonly processAll: () => Promise<void>
  readonly shutdown: (timeoutMs?: number) => Promise<void>
  readonly isShuttingDown: () => boolean
}

const createGracefulQueue = <T>(
  capacity: number,
  processor: (item: T) => Promise<void>,
  onError?: (error: Error, item: T) => void
): GracefulQueue<T> => {
  const queue = MutableQueue.bounded<T>(capacity)
  let shuttingDown = false
  let processingPromise: Promise<void> | null = null
  
  const offer = (item: T): boolean => {
    if (shuttingDown) {
      console.warn('Cannot add items during shutdown')
      return false
    }
    
    return queue.pipe(MutableQueue.offer(item))
  }
  
  const processAll = async (): Promise<void> => {
    if (processingPromise) {
      return processingPromise
    }
    
    processingPromise = (async () => {
      while (!queue.pipe(MutableQueue.isEmpty)) {
        const item = queue.pipe(MutableQueue.poll(MutableQueue.EmptyMutableQueue))
        if (item === MutableQueue.EmptyMutableQueue) break
        
        try {
          await processor(item)
        } catch (error) {
          const err = error instanceof Error ? error : new Error(String(error))
          if (onError) {
            onError(err, item)
          } else {
            console.error('Failed to process item:', err)
          }
        }
      }
    })()
    
    await processingPromise
    processingPromise = null
  }
  
  const shutdown = async (timeoutMs: number = 30000): Promise<void> => {
    console.log('Starting graceful shutdown...')
    shuttingDown = true
    
    const remainingItems = queue.pipe(MutableQueue.length)
    console.log(`Processing ${remainingItems} remaining items...`)
    
    // Race between processing all items and timeout
    const timeoutPromise = new Promise<void>((_, reject) => {
      setTimeout(() => reject(new Error('Shutdown timeout exceeded')), timeoutMs)
    })
    
    try {
      await Promise.race([processAll(), timeoutPromise])
      console.log('Graceful shutdown completed successfully')
    } catch (error) {
      const remaining = queue.pipe(MutableQueue.length)
      console.warn(`Shutdown timeout - ${remaining} items left unprocessed`)
      throw error
    }
  }
  
  const isShuttingDown = (): boolean => shuttingDown
  
  return { offer, processAll, shutdown, isShuttingDown }
}

// Usage example with signal handling
const emailQueue = createGracefulQueue<{ to: string, subject: string, body: string }>(
  500,
  async (email) => {
    // Simulate email sending
    console.log(`Sending email to ${email.to}: ${email.subject}`)
    await new Promise(resolve => setTimeout(resolve, 100))
  },
  (error, email) => {
    console.error(`Failed to send email to ${email.to}:`, error.message)
  }
)

// Add graceful shutdown handling
const gracefulShutdown = async (signal: string) => {
  console.log(`Received ${signal}, shutting down gracefully...`)
  
  try {
    await emailQueue.shutdown(30000) // 30 second timeout
    console.log('Shutdown completed successfully')
    process.exit(0)
  } catch (error) {
    console.error('Forced shutdown due to timeout')
    process.exit(1)
  }
}

process.on('SIGTERM', () => gracefulShutdown('SIGTERM'))
process.on('SIGINT', () => gracefulShutdown('SIGINT'))

// Simulate email generation
setInterval(() => {
  if (!emailQueue.isShuttingDown()) {
    emailQueue.offer({
      to: `user${Math.floor(Math.random() * 1000)}@example.com`,
      subject: 'Daily Newsletter',
      body: 'Your daily content...'
    })
  }
}, 50)

// Process emails continuously
const processEmails = async () => {
  while (!emailQueue.isShuttingDown()) {
    await emailQueue.processAll()
    await new Promise(resolve => setTimeout(resolve, 100))
  }
}

processEmails()
```

### Pattern 3: Queue Federation and Load Distribution

```typescript
import { MutableQueue, Chunk } from "effect"

interface QueueNode<T> {
  readonly id: string
  readonly queue: MutableQueue<T>
  readonly capacity: number
  weight: number
  totalProcessed: number
}

interface FederatedQueue<T> {
  readonly addNode: (id: string, capacity: number, weight: number) => void
  readonly removeNode: (id: string) => void
  readonly offer: (item: T) => boolean
  readonly poll: () => T | null
  readonly getStats: () => FederationStats
  readonly rebalance: () => void
}

interface FederationStats {
  readonly totalNodes: number
  readonly totalCapacity: number
  readonly totalItems: number
  readonly nodeUtilization: Record<string, number>
  readonly loadDistribution: Record<string, number>
}

const createFederatedQueue = <T>(): FederatedQueue<T> => {
  const nodes = new Map<string, QueueNode<T>>()
  
  const addNode = (id: string, capacity: number, weight: number): void => {
    if (nodes.has(id)) {
      throw new Error(`Node ${id} already exists`)
    }
    
    nodes.set(id, {
      id,
      queue: MutableQueue.bounded<T>(capacity),
      capacity,
      weight,
      totalProcessed: 0
    })
  }
  
  const removeNode = (id: string): void => {
    const node = nodes.get(id)
    if (!node) return
    
    // Redistribute items from removed node
    const remainingItems: T[] = []
    while (!node.queue.pipe(MutableQueue.isEmpty)) {
      const item = node.queue.pipe(MutableQueue.poll(MutableQueue.EmptyMutableQueue))
      if (item !== MutableQueue.EmptyMutableQueue) {
        remainingItems.push(item)
      }
    }
    
    nodes.delete(id)
    
    // Redistribute items to remaining nodes
    remainingItems.forEach(item => offer(item))
  }
  
  const selectNodeForOffer = (): QueueNode<T> | null => {
    const availableNodes = Array.from(nodes.values()).filter(node => 
      !node.queue.pipe(MutableQueue.isFull)
    )
    
    if (availableNodes.length === 0) return null
    
    // Weighted selection based on available capacity and weight
    const weightedNodes = availableNodes.map(node => {
      const utilization = node.queue.pipe(MutableQueue.length) / node.capacity
      const availableCapacity = 1 - utilization
      const effectiveWeight = node.weight * availableCapacity
      
      return { node, effectiveWeight }
    })
    
    const totalWeight = weightedNodes.reduce((sum, { effectiveWeight }) => sum + effectiveWeight, 0)
    
    if (totalWeight === 0) return availableNodes[0]
    
    const random = Math.random() * totalWeight
    let accumulator = 0
    
    for (const { node, effectiveWeight } of weightedNodes) {
      accumulator += effectiveWeight
      if (random <= accumulator) {
        return node
      }
    }
    
    return availableNodes[0]
  }
  
  const selectNodeForPoll = (): QueueNode<T> | null => {
    const nodesWithItems = Array.from(nodes.values()).filter(node => 
      !node.queue.pipe(MutableQueue.isEmpty)
    )
    
    if (nodesWithItems.length === 0) return null
    
    // Round-robin selection, but prefer nodes with more items
    return nodesWithItems.reduce((maxNode, currentNode) => {
      const maxItems = maxNode.queue.pipe(MutableQueue.length)
      const currentItems = currentNode.queue.pipe(MutableQueue.length)
      return currentItems > maxItems ? currentNode : maxNode
    })
  }
  
  const offer = (item: T): boolean => {
    const node = selectNodeForOffer()
    if (!node) return false
    
    return node.queue.pipe(MutableQueue.offer(item))
  }
  
  const poll = (): T | null => {
    const node = selectNodeForPoll()
    if (!node) return null
    
    const item = node.queue.pipe(MutableQueue.poll(MutableQueue.EmptyMutableQueue))
    if (item !== MutableQueue.EmptyMutableQueue) {
      node.totalProcessed++
      return item
    }
    
    return null
  }
  
  const getStats = (): FederationStats => {
    const totalCapacity = Array.from(nodes.values()).reduce((sum, node) => sum + node.capacity, 0)
    const totalItems = Array.from(nodes.values()).reduce((sum, node) => 
      sum + node.queue.pipe(MutableQueue.length), 0)
    
    const nodeUtilization: Record<string, number> = {}
    const loadDistribution: Record<string, number> = {}
    const totalProcessed = Array.from(nodes.values()).reduce((sum, node) => sum + node.totalProcessed, 0)
    
    nodes.forEach((node, id) => {
      const items = node.queue.pipe(MutableQueue.length)
      nodeUtilization[id] = items / node.capacity
      loadDistribution[id] = totalProcessed > 0 ? node.totalProcessed / totalProcessed : 0
    })
    
    return {
      totalNodes: nodes.size,
      totalCapacity,
      totalItems,
      nodeUtilization,
      loadDistribution
    }
  }
  
  const rebalance = (): void => {
    const allItems: T[] = []
    
    // Collect all items
    nodes.forEach(node => {
      while (!node.queue.pipe(MutableQueue.isEmpty)) {
        const item = node.queue.pipe(MutableQueue.poll(MutableQueue.EmptyMutableQueue))
        if (item !== MutableQueue.EmptyMutableQueue) {
          allItems.push(item)
        }
      }
    })
    
    // Redistribute items
    allItems.forEach(item => offer(item))
  }
  
  return { addNode, removeNode, offer, poll, getStats, rebalance }
}

// Usage example: Multi-region task processing
const taskFederation = createFederatedQueue<{ id: string, type: string, payload: unknown }>()

// Add nodes for different regions
taskFederation.addNode('us-east', 1000, 1.0)     // Primary region
taskFederation.addNode('us-west', 800, 0.8)      // Secondary region
taskFederation.addNode('eu-west', 600, 0.6)      // Tertiary region

// Simulate task generation
const generateTasks = () => {
  setInterval(() => {
    const task = {
      id: `task-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`,
      type: ['email', 'image', 'report'][Math.floor(Math.random() * 3)],
      payload: { data: 'task data' }
    }
    
    const success = taskFederation.offer(task)
    if (!success) {
      console.warn(`Task ${task.id} rejected - all queues full`)
    }
  }, 10)
}

// Process tasks
const processTasks = () => {
  setInterval(() => {
    const task = taskFederation.poll()
    if (task) {
      console.log(`Processing task ${task.id} of type ${task.type}`)
    }
  }, 50)
}

// Monitor federation stats
const monitorStats = () => {
  setInterval(() => {
    const stats = taskFederation.getStats()
    console.log('Federation Stats:', {
      nodes: stats.totalNodes,
      totalItems: stats.totalItems,
      capacity: stats.totalCapacity,
      utilization: Object.entries(stats.nodeUtilization).map(([id, util]) => 
        `${id}: ${(util * 100).toFixed(1)}%`).join(', '),
      loadDistribution: Object.entries(stats.loadDistribution).map(([id, load]) => 
        `${id}: ${(load * 100).toFixed(1)}%`).join(', ')
    })
  }, 5000)
}

// Start the federation
generateTasks()
processTasks()
monitorStats()

// Simulate node management
setTimeout(() => {
  console.log('Adding emergency node...')
  taskFederation.addNode('emergency', 500, 1.2)
  taskFederation.rebalance()
}, 30000)
```

## Integration Examples

### Integration with Effect's Concurrent Processing

```typescript
import { Effect, MutableQueue, Layer, Context } from "effect"

interface TaskProcessor {
  readonly submitTask: <E, A>(task: Effect.Effect<A, E>) => Effect.Effect<boolean>
  readonly processNext: Effect.Effect<void>
  readonly shutdown: Effect.Effect<void>
}

const TaskProcessor = Context.GenericTag<TaskProcessor>("TaskProcessor")

interface TaskItem<E, A> {
  readonly id: string
  readonly effect: Effect.Effect<A, E>
  readonly onComplete: (result: A) => void
  readonly onError: (error: E) => void
}

const makeTaskProcessor = (capacity: number): Effect.Effect<TaskProcessor> =>
  Effect.gen(function* () {
    const queue = MutableQueue.bounded<TaskItem<any, any>>(capacity)
    let shuttingDown = false
    
    const submitTask = <E, A>(task: Effect.Effect<A, E>): Effect.Effect<boolean> =>
      Effect.gen(function* () {
        if (shuttingDown) {
          return false
        }
        
        return yield* Effect.async<boolean>((resume) => {
          const taskItem: TaskItem<E, A> = {
            id: Math.random().toString(36),
            effect: task,
            onComplete: (result: A) => {
              // Task completed successfully
            },
            onError: (error: E) => {
              // Task failed
            }
          }
          
          const success = queue.pipe(MutableQueue.offer(taskItem))
          resume(Effect.succeed(success))
        })
      })
    
    const processNext = Effect.gen(function* () {
      const taskItem = queue.pipe(MutableQueue.poll(MutableQueue.EmptyMutableQueue))
      
      if (taskItem === MutableQueue.EmptyMutableQueue) {
        return
      }
      
      yield* Effect.fork(
        taskItem.effect.pipe(
          Effect.tap((result) => Effect.sync(() => taskItem.onComplete(result))),
          Effect.catchAll((error) => Effect.sync(() => taskItem.onError(error)))
        )
      )
    })
    
    const shutdown = Effect.gen(function* () {
      shuttingDown = true
      
      // Process remaining tasks
      while (!queue.pipe(MutableQueue.isEmpty)) {
        yield* processNext
      }
    })
    
    return TaskProcessor.of({ submitTask, processNext, shutdown })
  })

const TaskProcessorLayer = Layer.effect(TaskProcessor, makeTaskProcessor(1000))

// Usage with Effect
const exampleUsage = Effect.gen(function* () {
  const processor = yield* TaskProcessor
  
  // Submit a task
  const task = Effect.gen(function* () {
    yield* Effect.sleep("100 millis")
    return { result: "Task completed" }
  })
  
  const submitted = yield* processor.submitTask(task)
  console.log("Task submitted:", submitted)
  
  // Process tasks
  yield* processor.processNext
}).pipe(Effect.provide(TaskProcessorLayer))

Effect.runSync(exampleUsage)
```

### Testing Strategies

```typescript
import { MutableQueue, Chunk } from "effect"

// Test utilities for MutableQueue
const createTestQueue = <T>(items?: T[], capacity?: number): MutableQueue<T> => {
  const queue = capacity ? MutableQueue.bounded<T>(capacity) : MutableQueue.unbounded<T>()
  
  if (items) {
    items.forEach(item => queue.pipe(MutableQueue.offer(item)))
  }
  
  return queue
}

const drainQueue = <T>(queue: MutableQueue<T>): T[] => {
  const items: T[] = []
  
  while (!queue.pipe(MutableQueue.isEmpty)) {
    const item = queue.pipe(MutableQueue.poll(MutableQueue.EmptyMutableQueue))
    if (item !== MutableQueue.EmptyMutableQueue) {
      items.push(item)
    }
  }
  
  return items
}

// Property-based testing helpers
const testQueueProperty = <T>(
  name: string,
  generator: () => T[],
  property: (items: T[], queue: MutableQueue<T>) => boolean,
  iterations: number = 100
): void => {
  console.log(`Testing property: ${name}`)
  
  for (let i = 0; i < iterations; i++) {
    const items = generator()
    const queue = createTestQueue(items)
    
    if (!property(items, queue)) {
      throw new Error(`Property ${name} failed on iteration ${i} with items: ${JSON.stringify(items)}`)
    }
  }
  
  console.log(`✓ Property ${name} passed ${iterations} iterations`)
}

// Example tests
const runTests = () => {
  // Test FIFO ordering
  testQueueProperty(
    "FIFO ordering",
    () => Array.from({ length: Math.floor(Math.random() * 100) + 1 }, (_, i) => i),
    (items, queue) => {
      const drained = drainQueue(queue)
      return JSON.stringify(drained) === JSON.stringify(items)
    }
  )
  
  // Test capacity limits
  testQueueProperty(
    "Capacity limits",
    () => Array.from({ length: Math.floor(Math.random() * 50) + 10 }, (_, i) => i),
    (items) => {
      const capacity = Math.floor(items.length / 2)
      const queue = MutableQueue.bounded<number>(capacity)
      
      let accepted = 0
      items.forEach(item => {
        if (queue.pipe(MutableQueue.offer(item))) {
          accepted++
        }
      })
      
      return accepted === capacity && queue.pipe(MutableQueue.isFull)
    }
  )
  
  // Test batch operations
  testQueueProperty(
    "Batch operations consistency",
    () => Array.from({ length: Math.floor(Math.random() * 100) + 10 }, (_, i) => i),
    (items, queue) => {
      const batchSize = Math.floor(items.length / 3) + 1
      const batches: number[][] = []
      
      while (!queue.pipe(MutableQueue.isEmpty)) {
        const batch = queue.pipe(MutableQueue.pollUpTo(batchSize))
        batches.push(Chunk.toReadonlyArray(batch))
      }
      
      const flattened = batches.flat()
      return JSON.stringify(flattened) === JSON.stringify(items)
    }
  )
  
  console.log("All tests passed! ✓")
}

// Mock implementations for testing
const createMockQueue = <T>(): {
  queue: MutableQueue<T>
  mockOffer: jest.Mock
  mockPoll: jest.Mock
} => {
  const realQueue = MutableQueue.unbounded<T>()
  
  const mockOffer = jest.fn((item: T) => realQueue.pipe(MutableQueue.offer(item)))
  const mockPoll = jest.fn((def: any) => realQueue.pipe(MutableQueue.poll(def)))
  
  return {
    queue: {
      ...realQueue,
      pipe: (fn: any) => {
        if (fn === MutableQueue.offer) return mockOffer
        if (fn === MutableQueue.poll) return mockPoll
        return realQueue.pipe(fn)
      }
    } as any,
    mockOffer,
    mockPoll
  }
}

// Integration test example
const integrationTest = async () => {
  console.log("Running integration test...")
  
  const { queue, mockOffer, mockPoll } = createMockQueue<string>()
  
  // Test offer calls
  queue.pipe(MutableQueue.offer("test1"))
  queue.pipe(MutableQueue.offer("test2"))
  
  expect(mockOffer).toHaveBeenCalledTimes(2)
  expect(mockOffer).toHaveBeenCalledWith("test1")
  expect(mockOffer).toHaveBeenCalledWith("test2")
  
  // Test poll calls
  queue.pipe(MutableQueue.poll("default"))
  queue.pipe(MutableQueue.poll("default"))
  
  expect(mockPoll).toHaveBeenCalledTimes(2)
  expect(mockPoll).toHaveBeenCalledWith("default")
  
  console.log("Integration test passed! ✓")
}

// Run all tests
runTests()
```

## Conclusion

MutableQueue provides **high-performance FIFO operations**, **efficient capacity management**, and **built-in batch processing** for synchronous scenarios requiring optimal throughput and minimal memory overhead.

Key benefits:
- **O(1) Performance**: All core operations (offer, poll) are constant time, making MutableQueue suitable for high-throughput scenarios
- **Capacity Management**: Built-in support for bounded queues with clear backpressure signals and automatic capacity checking
- **Batch Processing**: Efficient `pollUpTo` operations enable optimal batch processing without manual iteration
- **Memory Efficiency**: Optimized internal structure minimizes memory allocation and provides better performance than array-based alternatives
- **Synchronous Design**: Perfect for scenarios where mutation is acceptable in exchange for maximum performance

MutableQueue is ideal when you need efficient FIFO operations in synchronous contexts, such as task processing systems, data buffering, rate limiting, and any scenario where traditional array-based queues become performance bottlenecks. For asynchronous producer-consumer scenarios with backpressure and coordination, consider using Effect's Queue instead.