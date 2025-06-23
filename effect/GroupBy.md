# GroupBy: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem GroupBy Solves

When processing streaming data, you often need to partition items by keys and process each group separately. Traditional approaches lead to several problems:

```typescript
// Traditional approach - manual grouping and processing
const processUserEvents = (events: UserEvent[]) => {
  const groups = new Map<string, UserEvent[]>()
  
  // Manual grouping
  for (const event of events) {
    const userId = event.userId
    if (!groups.has(userId)) {
      groups.set(userId, [])
    }
    groups.get(userId)!.push(event)
  }
  
  // Manual processing - all sequential, no backpressure control
  const results = []
  for (const [userId, userEvents] of groups) {
    const processed = processUserSpecificEvents(userEvents)
    results.push(processed)
  }
  
  return results
}
```

This approach leads to:
- **Memory Issues** - All data must be loaded into memory before processing
- **No Concurrency** - Groups are processed sequentially, not in parallel
- **No Backpressure** - No control over processing rate or buffering
- **Resource Leaks** - No automatic cleanup of completed groups

### The GroupBy Solution

GroupBy provides a streaming partitioning mechanism where groups are processed in parallel with automatic resource management:

```typescript
import { Effect, GroupBy, Stream } from "effect"

const processUserEvents = (eventStream: Stream.Stream<UserEvent>) => {
  return eventStream.pipe(
    Stream.groupByKey(event => event.userId),
    GroupBy.evaluate((userId, userEventStream) =>
      userEventStream.pipe(
        Stream.run(processUserSpecificEvents(userId))
      )
    )
  )
}
```

### Key Concepts

**GroupBy<K, V, E, R>**: A representation of a grouped stream where items are partitioned by keys of type `K`, values of type `V`, with possible errors `E` and requirements `R`

**Dynamic Partitioning**: Groups are created dynamically as new keys are encountered in the stream

**Parallel Processing**: Each group is processed concurrently with automatic resource management

## Basic Usage Patterns

### Pattern 1: Simple Key-Based Grouping

```typescript
import { Effect, GroupBy, Stream } from "effect"

// Group words by their first letter
const groupWordsByFirstLetter = Stream.fromIterable([
  "apple", "banana", "cherry", "apricot", "blueberry", "coconut"
]).pipe(
  Stream.groupByKey(word => word[0]),
  GroupBy.evaluate((letter, wordStream) =>
    Effect.gen(function* () {
      const words = yield* Stream.runCollect(wordStream)
      return { letter, words: Array.from(words), count: words.length }
    }).pipe(Stream.fromEffect)
  )
)

// Results: [
//   { letter: 'a', words: ['apple', 'apricot'], count: 2 },
//   { letter: 'b', words: ['banana', 'blueberry'], count: 2 },
//   { letter: 'c', words: ['cherry', 'coconut'], count: 2 }
// ]
```

### Pattern 2: Transform-and-Group Pattern

```typescript
// Group and transform items in one step
const processUserActions = Stream.fromIterable([
  { userId: "1", action: "login", timestamp: Date.now() },
  { userId: "2", action: "view", timestamp: Date.now() },
  { userId: "1", action: "purchase", timestamp: Date.now() }
]).pipe(
  Stream.groupBy(action => 
    Effect.succeed([action.userId, {
      action: action.action,
      processedAt: new Date(),
      originalTime: action.timestamp
    }])
  ),
  GroupBy.evaluate((userId, actionStream) =>
    actionStream.pipe(
      Stream.runCollect,
      Effect.map(actions => ({ userId, actions: Array.from(actions) })),
      Stream.fromEffect
    )
  )
)
```

### Pattern 3: Filtering Groups

```typescript
// Only process specific groups
const processHighPriorityUsers = userEventStream.pipe(
  Stream.groupByKey(event => event.userId),
  GroupBy.filter(userId => isHighPriorityUser(userId)),
  GroupBy.evaluate((userId, events) =>
    events.pipe(
      Stream.run(processHighPriorityEvents(userId))
    )
  )
)

// Only process first N groups
const processFirstThreeGroups = dataStream.pipe(
  Stream.groupByKey(item => item.category),
  GroupBy.first(3),
  GroupBy.evaluate((category, items) =>
    items.pipe(
      Stream.run(processCategoryItems(category))
    )
  )
)
```

## Real-World Examples

### Example 1: Real-Time Log Processing

Processing server logs grouped by service name with parallel processing:

```typescript
import { Effect, GroupBy, Stream, Sink, Schedule } from "effect"

interface LogEntry {
  readonly service: string
  readonly level: "info" | "warn" | "error"
  readonly message: string
  readonly timestamp: number
}

interface ProcessedLogs {
  readonly service: string
  readonly totalLogs: number
  readonly errorCount: number
  readonly warningCount: number
  readonly recentErrors: Array<string>
}

const createLogProcessor = () => {
  return Effect.gen(function* () {
    const processServiceLogs = (service: string, logStream: Stream.Stream<LogEntry>) => {
      return Effect.gen(function* () {
        const logs = yield* Stream.runCollect(logStream)
        const logArray = Array.from(logs)
        
        const errorCount = logArray.filter(log => log.level === "error").length
        const warningCount = logArray.filter(log => log.level === "warn").length
        const recentErrors = logArray
          .filter(log => log.level === "error")
          .slice(-5)
          .map(log => log.message)
        
        const result: ProcessedLogs = {
          service,
          totalLogs: logArray.length,
          errorCount,
          warningCount,
          recentErrors
        }
        
        // Send alerts for high error rates
        if (errorCount > logArray.length * 0.1) {
          yield* Effect.log(`ALERT: High error rate for service ${service}: ${errorCount}/${logArray.length}`)
        }
        
        return result
      }).pipe(Stream.fromEffect)
    }
    
    const processLogStream = (logStream: Stream.Stream<LogEntry>) => {
      return logStream.pipe(
        // Group logs by service
        Stream.groupByKey(log => log.service),
        // Process each service's logs in parallel
        GroupBy.evaluate(processServiceLogs, { bufferSize: 100 })
      )
    }
    
    return { processLogStream }
  })
}

// Usage with simulated log stream
const logProcessingDemo = Effect.gen(function* () {
  const processor = yield* createLogProcessor()
  
  // Simulate log entries
  const logEntries: LogEntry[] = [
    { service: "auth", level: "info", message: "User logged in", timestamp: Date.now() },
    { service: "payment", level: "error", message: "Payment failed", timestamp: Date.now() },
    { service: "auth", level: "warn", message: "Slow response", timestamp: Date.now() },
    { service: "inventory", level: "info", message: "Stock updated", timestamp: Date.now() },
    { service: "payment", level: "error", message: "Timeout error", timestamp: Date.now() },
    { service: "auth", level: "info", message: "User logged out", timestamp: Date.now() }
  ]
  
  const results = yield* processor.processLogStream(Stream.fromIterable(logEntries)).pipe(
    Stream.runCollect
  )
  
  yield* Effect.forEach(results, result =>
    Effect.log(`Service: ${result.service}, Total: ${result.totalLogs}, Errors: ${result.errorCount}`)
  )
  
  return results
})
```

### Example 2: E-commerce Order Processing

Process orders grouped by customer with parallel fulfillment:

```typescript
interface Order {
  readonly orderId: string
  readonly customerId: string
  readonly items: Array<{ productId: string; quantity: number; price: number }>
  readonly priority: "standard" | "express" | "overnight"
  readonly timestamp: number
}

interface CustomerOrderSummary {
  readonly customerId: string
  readonly orderCount: number
  readonly totalValue: number
  readonly averageOrderValue: number
  readonly fulfillmentStatus: "processing" | "shipped" | "delivered"
  readonly estimatedDelivery: Date
}

const createOrderProcessor = () => {
  return Effect.gen(function* () {
    const processCustomerOrders = (
      customerId: string, 
      orderStream: Stream.Stream<Order>
    ) => {
      return Effect.gen(function* () {
        yield* Effect.log(`Processing orders for customer ${customerId}`)
        
        const orders = yield* Stream.runCollect(orderStream)
        const orderArray = Array.from(orders)
        
        // Calculate totals
        const totalValue = orderArray.reduce((sum, order) =>
          sum + order.items.reduce((orderSum, item) => 
            orderSum + (item.price * item.quantity), 0
          ), 0
        )
        
        const averageOrderValue = totalValue / orderArray.length
        
        // Determine fulfillment approach based on order priority
        const hasExpressOrders = orderArray.some(order => 
          order.priority === "express" || order.priority === "overnight"
        )
        
        // Simulate fulfillment processing
        yield* Effect.sleep("100 millis") // Simulate processing time
        
        // Estimate delivery based on priority
        const deliveryDays = hasExpressOrders ? 1 : 3
        const estimatedDelivery = new Date(Date.now() + deliveryDays * 24 * 60 * 60 * 1000)
        
        const summary: CustomerOrderSummary = {
          customerId,
          orderCount: orderArray.length,
          totalValue,
          averageOrderValue,
          fulfillmentStatus: "processing",
          estimatedDelivery
        }
        
        yield* Effect.log(`Processed ${orderArray.length} orders for customer ${customerId} (total: $${totalValue})`)
        
        return summary
      }).pipe(Stream.fromEffect)
    }
    
    const processOrderBatch = (orderStream: Stream.Stream<Order>) => {
      return orderStream.pipe(
        Stream.groupByKey(order => order.customerId),
        // Only process first 10 customers to avoid overwhelming the system
        GroupBy.first(10),
        GroupBy.evaluate(processCustomerOrders, { bufferSize: 50 })
      )
    }
    
    return { processOrderBatch }
  })
}

// Usage
const orderProcessingDemo = Effect.gen(function* () {
  const processor = yield* createOrderProcessor()
  
  // Simulate order stream
  const orders: Order[] = [
    {
      orderId: "1", customerId: "cust1", priority: "standard", timestamp: Date.now(),
      items: [{ productId: "prod1", quantity: 2, price: 50 }]
    },
    {
      orderId: "2", customerId: "cust2", priority: "express", timestamp: Date.now(),
      items: [{ productId: "prod2", quantity: 1, price: 100 }]
    },
    {
      orderId: "3", customerId: "cust1", priority: "overnight", timestamp: Date.now(),
      items: [{ productId: "prod3", quantity: 3, price: 25 }]
    }
  ]
  
  const summaries = yield* processor.processOrderBatch(Stream.fromIterable(orders)).pipe(
    Stream.runCollect
  )
  
  yield* Effect.forEach(summaries, summary =>
    Effect.log(`Customer ${summary.customerId}: ${summary.orderCount} orders, $${summary.totalValue} total`)
  )
  
  return summaries
})
```

### Example 3: IoT Sensor Data Processing

Process sensor readings grouped by device with anomaly detection:

```typescript
interface SensorReading {
  readonly deviceId: string
  readonly sensorType: "temperature" | "humidity" | "pressure"
  readonly value: number
  readonly timestamp: number
  readonly location: string
}

interface DeviceAnalysis {
  readonly deviceId: string
  readonly location: string
  readonly readingCount: number
  readonly averageValues: Record<string, number>
  readonly anomalies: Array<{ sensorType: string; value: number; threshold: number }>
  readonly healthStatus: "healthy" | "warning" | "critical"
}

const createSensorProcessor = () => {
  return Effect.gen(function* () {
    const thresholds = {
      temperature: { min: -10, max: 50 },
      humidity: { min: 0, max: 100 },
      pressure: { min: 900, max: 1100 }
    }
    
    const processDeviceReadings = (
      deviceId: string,
      readingStream: Stream.Stream<SensorReading>
    ) => {
      return Effect.gen(function* () {
        yield* Effect.log(`Analyzing readings for device ${deviceId}`)
        
        const readings = yield* Stream.runCollect(readingStream)
        const readingArray = Array.from(readings)
        
        if (readingArray.length === 0) {
          return null
        }
        
        const location = readingArray[0].location
        
        // Group by sensor type and calculate averages
        const sensorGroups = readingArray.reduce((groups, reading) => {
          if (!groups[reading.sensorType]) {
            groups[reading.sensorType] = []
          }
          groups[reading.sensorType].push(reading.value)
          return groups
        }, {} as Record<string, number[]>)
        
        const averageValues = Object.entries(sensorGroups).reduce((avgs, [type, values]) => {
          avgs[type] = values.reduce((sum, val) => sum + val, 0) / values.length
          return avgs
        }, {} as Record<string, number>)
        
        // Detect anomalies
        const anomalies: Array<{ sensorType: string; value: number; threshold: number }> = []
        
        for (const reading of readingArray) {
          const threshold = thresholds[reading.sensorType]
          if (reading.value < threshold.min || reading.value > threshold.max) {
            anomalies.push({
              sensorType: reading.sensorType,
              value: reading.value,
              threshold: reading.value < threshold.min ? threshold.min : threshold.max
            })
          }
        }
        
        // Determine health status
        const healthStatus = anomalies.length === 0 ? "healthy" :
                           anomalies.length <= 2 ? "warning" : "critical"
        
        const analysis: DeviceAnalysis = {
          deviceId,
          location,
          readingCount: readingArray.length,
          averageValues,
          anomalies,
          healthStatus
        }
        
        if (healthStatus === "critical") {
          yield* Effect.log(`CRITICAL: Device ${deviceId} has ${anomalies.length} anomalies`)
        }
        
        return analysis
      }).pipe(
        Effect.map(analysis => analysis ? [analysis] : []),
        Effect.map(Stream.fromIterable),
        Stream.fromEffect,
        Stream.flatten
      )
    }
    
    const processSensorData = (sensorStream: Stream.Stream<SensorReading>) => {
      return sensorStream.pipe(
        Stream.groupByKey(reading => reading.deviceId),
        GroupBy.evaluate(processDeviceReadings, { bufferSize: 200 })
      )
    }
    
    return { processSensorData }
  })
}

// Usage
const sensorProcessingDemo = Effect.gen(function* () {
  const processor = yield* createSensorProcessor()
  
  // Simulate sensor readings
  const readings: SensorReading[] = [
    { deviceId: "sensor1", sensorType: "temperature", value: 22.5, timestamp: Date.now(), location: "Room A" },
    { deviceId: "sensor1", sensorType: "humidity", value: 45.0, timestamp: Date.now(), location: "Room A" },
    { deviceId: "sensor2", sensorType: "temperature", value: 85.0, timestamp: Date.now(), location: "Room B" }, // Anomaly!
    { deviceId: "sensor1", sensorType: "pressure", value: 1013.25, timestamp: Date.now(), location: "Room A" },
    { deviceId: "sensor2", sensorType: "humidity", value: 30.0, timestamp: Date.now(), location: "Room B" },
    { deviceId: "sensor3", sensorType: "temperature", value: 19.8, timestamp: Date.now(), location: "Room C" }
  ]
  
  const analyses = yield* processor.processSensorData(Stream.fromIterable(readings)).pipe(
    Stream.runCollect
  )
  
  yield* Effect.forEach(analyses, analysis =>
    Effect.log(`Device ${analysis.deviceId} (${analysis.location}): ${analysis.healthStatus} - ${analysis.anomalies.length} anomalies`)
  )
  
  return analyses
})
```

## Advanced Features Deep Dive

### Feature 1: Dynamic Group Creation with Effects

Create groups dynamically using effectful computations:

```typescript
// Group items with async key computation
const processWithAsyncGrouping = (dataStream: Stream.Stream<RawData>) => {
  return dataStream.pipe(
    Stream.groupBy(item =>
      Effect.gen(function* () {
        // Async key computation - e.g., lookup user's organization
        const organization = yield* lookupUserOrganization(item.userId)
        const processedItem = yield* transformItem(item)
        return [organization.id, processedItem] as const
      })
    ),
    GroupBy.evaluate((orgId, itemStream) =>
      itemStream.pipe(
        Stream.run(processOrganizationData(orgId))
      )
    )
  )
}
```

### Feature 2: Buffer Size Optimization

Control memory usage and processing efficiency with buffer size tuning:

```typescript
const optimizedGroupProcessing = (highVolumeStream: Stream.Stream<Event>) => {
  return highVolumeStream.pipe(
    Stream.groupByKey(event => event.category, {
      bufferSize: 1000 // Larger buffer for high-volume streams
    }),
    GroupBy.evaluate((category, eventStream) =>
      eventStream.pipe(
        // Process in batches to optimize throughput
        Stream.grouped(50),
        Stream.mapEffectPar(5, (batch) => processBatch(category, batch))
      ),
      { bufferSize: 100 } // Smaller buffer for processed results
    )
  )
}
```

### Feature 3: Error Handling and Recovery

Handle errors gracefully within groups:

```typescript
const robustGroupProcessing = (dataStream: Stream.Stream<ProcessableItem>) => {
  return dataStream.pipe(
    Stream.groupByKey(item => item.groupKey),
    GroupBy.evaluate((key, itemStream) =>
      itemStream.pipe(
        Stream.mapEffect(item =>
          processItem(item).pipe(
            Effect.retry(Schedule.exponential("100 millis").pipe(
              Schedule.compose(Schedule.recurs(3))
            )),
            Effect.catchAll(error =>
              Effect.gen(function* () {
                yield* Effect.log(`Failed to process item in group ${key}: ${error}`)
                return { item, error: error.toString(), processed: false }
              })
            )
          )
        )
      )
    )
  )
}
```

## Practical Patterns & Best Practices

### Pattern 1: Streaming Group Aggregation

Create running aggregations for each group:

```typescript
const createStreamingAggregator = <K, V, A>(
  keyExtractor: (item: V) => K,
  initialState: A,
  aggregator: (state: A, item: V) => A
) => {
  return (stream: Stream.Stream<V>) => {
    return stream.pipe(
      Stream.groupByKey(keyExtractor),
      GroupBy.evaluate((key, itemStream) =>
        itemStream.pipe(
          Stream.scan(initialState, aggregator),
          Stream.map(state => ({ key, state }))
        )
      )
    )
  }
}

// Usage: Running sum by category
const runningSumByCategory = createStreamingAggregator(
  (item: { category: string; value: number }) => item.category,
  0,
  (sum, item) => sum + item.value
)

const demo = Stream.fromIterable([
  { category: "A", value: 10 },
  { category: "B", value: 20 },
  { category: "A", value: 15 },
  { category: "B", value: 25 }
]).pipe(
  runningSumByCategory,
  Stream.runCollect
)
// Results: [
//   { key: "A", state: 10 }, { key: "B", state: 20 },
//   { key: "A", state: 25 }, { key: "B", state: 45 }
// ]
```

### Pattern 2: Group Lifecycle Management

Manage resource allocation per group with automatic cleanup:

```typescript
const createManagedGroupProcessor = <K, V, R>(
  setupResource: (key: K) => Effect.Effect<R>,
  cleanupResource: (resource: R) => Effect.Effect<void>,
  processor: (key: K, resource: R, itemStream: Stream.Stream<V>) => Stream.Stream<any>
) => {
  return (keyExtractor: (item: V) => K) => (stream: Stream.Stream<V>) => {
    return stream.pipe(
      Stream.groupByKey(keyExtractor),
      GroupBy.evaluate((key, itemStream) =>
        Effect.gen(function* () {
          const resource = yield* setupResource(key)
          
          return yield* processor(key, resource, itemStream).pipe(
            Stream.ensuring(cleanupResource(resource)),
            Stream.runCollect
          )
        }).pipe(
          Stream.fromEffect,
          Stream.flatten
        )
      )
    )
  }
}

// Usage: Database connection per group
const processWithDatabasePerGroup = createManagedGroupProcessor(
  (groupId: string) => createDatabaseConnection(groupId),
  (connection) => connection.close(),
  (groupId, connection, itemStream) =>
    itemStream.pipe(
      Stream.mapEffect(item => saveToDatabase(connection, item))
    )
)
```

### Pattern 3: Conditional Group Processing

Process groups based on dynamic conditions:

```typescript
const createConditionalProcessor = <K, V>(
  condition: (key: K, items: Array<V>) => Effect.Effect<boolean>
) => {
  return (stream: Stream.Stream<V>, keyExtractor: (item: V) => K) => {
    return stream.pipe(
      Stream.groupByKey(keyExtractor),
      GroupBy.evaluate((key, itemStream) =>
        Effect.gen(function* () {
          const items = yield* Stream.runCollect(itemStream)
          const itemArray = Array.from(items)
          
          const shouldProcess = yield* condition(key, itemArray)
          
          if (shouldProcess) {
            yield* Effect.log(`Processing group ${key} with ${itemArray.length} items`)
            return yield* processGroup(key, itemArray)
          } else {
            yield* Effect.log(`Skipping group ${key}`)
            return []
          }
        }).pipe(
          Effect.map(Stream.fromIterable),
          Stream.fromEffect,
          Stream.flatten
        )
      )
    )
  }
}

// Usage: Only process groups with sufficient data
const processLargeGroups = createConditionalProcessor<string, DataItem>(
  (key, items) => Effect.succeed(items.length >= 10)
)
```

## Integration Examples

### Integration with Stream Processing Pipeline

Build a complete streaming pipeline with multiple grouping stages:

```typescript
import { Effect, GroupBy, Stream, Sink } from "effect"

interface RawEvent {
  readonly id: string
  readonly userId: string
  readonly eventType: string
  readonly payload: unknown
  readonly timestamp: number
}

interface ProcessedEvent {
  readonly id: string
  readonly userId: string
  readonly eventType: string
  readonly processedPayload: unknown
  readonly processingTime: number
}

interface UserEventSummary {
  readonly userId: string
  readonly eventCount: number
  readonly eventTypes: Array<string>
  readonly totalProcessingTime: number
  readonly firstEvent: number
  readonly lastEvent: number
}

const createEventProcessingPipeline = () => {
  return Effect.gen(function* () {
    const processEventBatch = (events: Array<RawEvent>): Effect.Effect<Array<ProcessedEvent>> => {
      return Effect.gen(function* () {
        const startTime = yield* Effect.clockWith(_ => _.currentTimeMillis)
        
        const processed = yield* Effect.forEach(events, event =>
          Effect.gen(function* () {
            // Simulate event processing
            yield* Effect.sleep("10 millis")
            const processedPayload = yield* transformPayload(event.payload)
            
            return {
              id: event.id,
              userId: event.userId,
              eventType: event.eventType,
              processedPayload,
              processingTime: Date.now() - startTime
            }
          })
        )
        
        return processed
      })
    }
    
    const createUserSummary = (userId: string, events: Array<ProcessedEvent>): UserEventSummary => {
      const eventTypes = [...new Set(events.map(e => e.eventType))]
      const totalProcessingTime = events.reduce((sum, e) => sum + e.processingTime, 0)
      const timestamps = events.map(e => e.timestamp || Date.now())
      
      return {
        userId,
        eventCount: events.length,
        eventTypes,
        totalProcessingTime,
        firstEvent: Math.min(...timestamps),
        lastEvent: Math.max(...timestamps)
      }
    }
    
    const pipeline = (eventStream: Stream.Stream<RawEvent>) => {
      return eventStream.pipe(
        // Stage 1: Group by event type for batch processing
        Stream.groupByKey(event => event.eventType),
        GroupBy.evaluate((eventType, typeEventStream) =>
          typeEventStream.pipe(
            // Process in batches of 10
            Stream.grouped(10),
            Stream.mapEffect(eventChunk => processEventBatch(Array.from(eventChunk))),
            Stream.flattenChunks
          )
        ),
        // Stage 2: Regroup by user for summary creation
        Stream.groupByKey(event => event.userId),
        GroupBy.evaluate((userId, userEventStream) =>
          Effect.gen(function* () {
            const events = yield* Stream.runCollect(userEventStream)
            const eventArray = Array.from(events)
            const summary = createUserSummary(userId, eventArray)
            
            yield* Effect.log(`Generated summary for user ${userId}: ${eventArray.length} events`)
            
            return summary
          }).pipe(Stream.fromEffect)
        )
      )
    }
    
    return { pipeline }
  })
}

// Usage
const eventPipelineDemo = Effect.gen(function* () {
  const processor = yield* createEventProcessingPipeline()
  
  // Simulate event stream
  const events: RawEvent[] = Array.from({ length: 50 }, (_, i) => ({
    id: `event-${i}`,
    userId: `user-${i % 5}`, // 5 different users
    eventType: `type-${i % 3}`, // 3 different event types
    payload: { data: `payload-${i}` },
    timestamp: Date.now() + i * 1000
  }))
  
  const summaries = yield* processor.pipeline(Stream.fromIterable(events)).pipe(
    Stream.runCollect
  )
  
  yield* Effect.log(`Processed ${summaries.length} user summaries`)
  
  return summaries
})
```

### Integration with Testing Framework

Use GroupBy for parallel test execution:

```typescript
interface TestCase {
  readonly id: string
  readonly suite: string
  readonly name: string
  readonly test: () => Effect.Effect<TestResult>
}

interface TestResult {
  readonly passed: boolean
  readonly duration: number
  readonly error?: string
}

interface SuiteResult {
  readonly suite: string
  readonly totalTests: number
  readonly passed: number
  readonly failed: number
  readonly totalDuration: number
  readonly results: Array<TestResult & { testName: string }>
}

const createParallelTestRunner = () => {
  return Effect.gen(function* () {
    const runTestSuite = (suite: string, testStream: Stream.Stream<TestCase>) => {
      return Effect.gen(function* () {
        yield* Effect.log(`Running test suite: ${suite}`)
        
        const results = yield* testStream.pipe(
          Stream.mapEffectPar(5, testCase => // Run up to 5 tests in parallel
            Effect.gen(function* () {
              const startTime = yield* Effect.clockWith(_ => _.currentTimeMillis)
              
              const result = yield* testCase.test().pipe(
                Effect.map(result => ({ ...result, testName: testCase.name })),
                Effect.catchAll(error => Effect.succeed({
                  passed: false,
                  duration: 0,
                  error: error.toString(),
                  testName: testCase.name
                }))
              )
              
              const endTime = yield* Effect.clockWith(_ => _.currentTimeMillis)
              const duration = endTime - startTime
              
              return { ...result, duration }
            })
          ),
          Stream.runCollect
        )
        
        const resultArray = Array.from(results)
        const passed = resultArray.filter(r => r.passed).length
        const failed = resultArray.length - passed
        const totalDuration = resultArray.reduce((sum, r) => sum + r.duration, 0)
        
        const suiteResult: SuiteResult = {
          suite,
          totalTests: resultArray.length,
          passed,
          failed,
          totalDuration,
          results: resultArray
        }
        
        yield* Effect.log(`Suite ${suite} completed: ${passed}/${resultArray.length} passed`)
        
        return suiteResult
      }).pipe(Stream.fromEffect)
    }
    
    const runAllTests = (testStream: Stream.Stream<TestCase>) => {
      return testStream.pipe(
        Stream.groupByKey(testCase => testCase.suite),
        GroupBy.evaluate(runTestSuite)
      )
    }
    
    return { runAllTests }
  })
}

// Usage
const testRunnerDemo = Effect.gen(function* () {
  const runner = yield* createParallelTestRunner()
  
  // Define test cases
  const testCases: TestCase[] = [
    {
      id: "1", suite: "auth", name: "login test",
      test: () => Effect.succeed({ passed: true, duration: 100 })
    },
    {
      id: "2", suite: "auth", name: "logout test",
      test: () => Effect.succeed({ passed: true, duration: 50 })
    },
    {
      id: "3", suite: "payment", name: "process payment",
      test: () => Effect.succeed({ passed: false, duration: 200, error: "Invalid card" })
    },
    {
      id: "4", suite: "payment", name: "refund payment",
      test: () => Effect.succeed({ passed: true, duration: 150 })
    }
  ]
  
  const suiteResults = yield* runner.runAllTests(Stream.fromIterable(testCases)).pipe(
    Stream.runCollect
  )
  
  const totalTests = suiteResults.reduce((sum, suite) => sum + suite.totalTests, 0)
  const totalPassed = suiteResults.reduce((sum, suite) => sum + suite.passed, 0)
  
  yield* Effect.log(`All tests completed: ${totalPassed}/${totalTests} passed`)
  
  return suiteResults
})
```

## Conclusion

GroupBy provides powerful stream partitioning and parallel processing capabilities for building scalable data processing pipelines in Effect applications.

Key benefits:
- **Dynamic Partitioning** - Groups are created automatically as new keys are encountered
- **Parallel Processing** - Each group is processed concurrently with automatic resource management  
- **Memory Efficiency** - Streaming approach avoids loading all data into memory
- **Error Isolation** - Failures in one group don't affect others

GroupBy is ideal for log processing, event stream processing, batch job orchestration, real-time analytics, and any scenario requiring key-based stream partitioning with parallel processing and automatic resource management.