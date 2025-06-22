# Stream: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem Stream Solves

Processing large datasets, real-time data, and asynchronous sequences efficiently is a fundamental challenge in modern applications. Traditional approaches often lead to memory issues, complex error handling, and difficult-to-maintain code.

```typescript
// Traditional Node.js approach - memory intensive
import fs from 'fs'
import readline from 'readline'

async function processLargeFile(filePath: string) {
  const results: string[] = []
  
  const fileStream = fs.createReadStream(filePath)
  const rl = readline.createInterface({
    input: fileStream,
    crlfDelay: Infinity
  })
  
  try {
    for await (const line of rl) {
      // Process each line
      if (line.includes('ERROR')) {
        results.push(line)
      }
    }
  } catch (error) {
    // Error handling is separate from the main logic
    console.error('Processing failed:', error)
    throw error
  } finally {
    // Resource cleanup is manual
    fileStream.close()
  }
  
  return results
}

// RxJS approach - complex for simple cases
import { from, Observable } from 'rxjs'
import { filter, map, catchError } from 'rxjs/operators'

function processDataRxJS(data: string[]): Observable<string> {
  return from(data).pipe(
    filter(line => line.includes('ERROR')),
    map(line => line.toUpperCase()),
    catchError(error => {
      console.error('Stream error:', error)
      return []
    })
  )
}
```

This approach leads to:
- **Memory Issues** - Loading entire datasets into memory for processing
- **Complex Error Handling** - Error handling scattered throughout the code
- **Resource Management** - Manual cleanup of resources prone to leaks
- **Limited Composability** - Difficult to combine and reuse streaming operations

### The Stream Solution

Effect's Stream module provides a powerful, type-safe, and composable approach to handling asynchronous data sequences with built-in resource management and error handling.

```typescript
import { Stream, Effect, Chunk } from "effect"
import * as NodeFS from "@effect/platform-node/NodeFileSystem"

// The Effect solution - memory efficient, composable, type-safe
const processLargeFile = (filePath: string) =>
  Stream.fromReadableStream(
    () => NodeFS.NodeFileSystem.stream(filePath),
    (_) => new Error("Failed to read file")
  ).pipe(
    Stream.decodeText(),
    Stream.splitLines,
    Stream.filter((line) => line.includes('ERROR')),
    Stream.map((line) => line.toUpperCase()),
    Stream.runCollect
  )
```

### Key Concepts

**Stream**: A description of a program that, when evaluated, may emit zero or more values of type A, may fail with errors of type E, and uses a context of type R.

**Chunk**: An immutable, efficient collection optimized for streaming operations that minimizes allocations and copying.

**Pull-based**: Streams are pull-based, meaning values are only produced when requested by consumers, enabling efficient memory usage.

## Basic Usage Patterns

### Creating Streams

```typescript
import { Stream, Effect, Chunk, Schedule, Duration } from "effect"

// From a single value
const single = Stream.succeed(42)

// From multiple values
const multiple = Stream.make(1, 2, 3, 4, 5)

// From an array
const fromArray = Stream.fromIterable([1, 2, 3, 4, 5])

// From an Effect
const fromEffect = Stream.fromEffect(
  Effect.succeed("Hello, Stream!")
)

// Infinite stream with recursion
const naturals = Stream.iterate(0, n => n + 1)

// Stream that emits values over time
const ticks = Stream.repeatEffectWithSchedule(
  Effect.succeed(Date.now()),
  Schedule.spaced(Duration.seconds(1))
)
```

### Transforming Streams

```typescript
// Map over stream values
const doubled = Stream.map(Stream.make(1, 2, 3, 4, 5), n => n * 2)

// Filter stream values  
const evens = Stream.filter(Stream.range(1, 10), n => n % 2 === 0)

// FlatMap for dependent computations
const expanded = Stream.make(1, 2, 3).pipe(
  Stream.flatMap(n => 
    Stream.make(n, n * 10, n * 100)
  )
)

// Scan for running computations
const runningSum = Stream.scan(Stream.make(1, 2, 3, 4, 5), 0, (acc, n) => acc + n)
```

### Consuming Streams

```typescript
// Collect all values into a Chunk
const collected = Stream.runCollect(Stream.make(1, 2, 3))

// Run for side effects
const logged = Stream.make("Hello", "World").pipe(
  Stream.tap(value => Effect.log(value)),
  Stream.runDrain
)

// Fold into a single value
const sum = Stream.runFold(Stream.range(1, 100), 0, (acc, n) => acc + n)

// Take only first n elements
const firstFive = Stream.iterate(1, n => n + 1).pipe(
  Stream.take(5),
  Stream.runCollect
)
```

## Real-World Examples

### Example 1: File Processing - Log Analysis Pipeline

Processing large log files efficiently while extracting insights and generating reports.

```typescript
import { Stream, Effect, Chunk, pipe } from "effect"
import * as NodeFS from "@effect/platform-node/NodeFileSystem"
import * as Path from "@effect/platform-node/NodePath"

// Define log entry structure
interface LogEntry {
  timestamp: Date
  level: "INFO" | "WARN" | "ERROR"
  message: string
  metadata?: Record<string, unknown>
}

// Parse a log line into a structured entry
const parseLogLine = (line: string): Effect.Effect<LogEntry, Error> =>
  Effect.try({
    try: () => {
      const match = line.match(/^\[(\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2})\] \[(\w+)\] (.+)$/)
      if (!match) throw new Error(`Invalid log format: ${line}`)
      
      const [, timestamp, level, rest] = match
      const metadataMatch = rest.match(/^(.+?) (\{.+\})$/)
      
      return {
        timestamp: new Date(timestamp),
        level: level as LogEntry["level"],
        message: metadataMatch ? metadataMatch[1] : rest,
        metadata: metadataMatch ? JSON.parse(metadataMatch[2]) : undefined
      }
    },
    catch: (error) => new Error(`Failed to parse log line: ${error}`)
  })

// Create a log processing pipeline
const processLogs = (filePath: string) => {
  const fileSystem = NodeFS.NodeFileSystem
  
  return Stream.fromReadableStream(
    () => fileSystem.stream(filePath),
    () => new Error(`Failed to read file: ${filePath}`)
  ).pipe(
    // Decode text and split into lines
    Stream.decodeText(),
    Stream.splitLines,
    
    // Parse each line into a LogEntry
    Stream.mapEffect(parseLogLine),
    
    // Filter for errors and warnings only
    Stream.filter(entry => 
      entry.level === "ERROR" || entry.level === "WARN"
    ),
    
    // Group by log level
    Stream.groupByKey(entry => entry.level),
    
    // Process each group
    Stream.flatMap(([level, entries]) =>
      Stream.fromEffect(
        pipe(
          entries,
          Stream.runCollect,
          Effect.map(chunk => ({
            level,
            count: chunk.length,
            entries: Chunk.toReadonlyArray(chunk)
          }))
        )
      )
    )
  )
}

// Generate a summary report
const generateLogReport = (logPath: string) =>
  Effect.gen(function* () {
    const results = yield* 
      processLogs(logPath).pipe(
        Stream.runCollect
      )
    )
    
    const report = {
      totalIssues: results.pipe(
        Chunk.reduce(0, (acc, group) => acc + group.count)
      ),
      byLevel: results.pipe(
        Chunk.toReadonlyArray,
        arr => Object.fromEntries(
          arr.map(group => [group.level, group.count])
        )
      ),
      samples: results.pipe(
        Chunk.flatMap(group => 
          Chunk.fromIterable(group.entries.slice(0, 3))
        ),
        Chunk.toReadonlyArray
      )
    }
    
    yield* Effect.log("Log Analysis Report")
    yield* Effect.log(`Total Issues: ${report.totalIssues}`)
    yield* Effect.log(`By Level: ${JSON.stringify(report.byLevel)}`)
    
    return report
  })
```

### Example 2: WebSocket Handling - Real-time Data Processing

Building a real-time data processing pipeline for WebSocket connections with automatic reconnection and backpressure handling.

```typescript
import { Stream, Effect, Queue, Fiber, Ref, Schedule, Duration } from "effect"

interface WebSocketMessage {
  type: "data" | "control" | "heartbeat"
  payload: unknown
  timestamp: number
}

// WebSocket stream with reconnection logic
const createWebSocketStream = (url: string) => {
  const connect = Effect.gen(function* () {
    const ws = new WebSocket(url)
    const queue = yield* Queue.unbounded<WebSocketMessage>()
    const isConnected = yield* Ref.make(false)
    
    // Setup WebSocket handlers
    ws.onopen = () => {
      Effect.runSync(Ref.set(isConnected, true))
      console.log("WebSocket connected")
    }
    
    ws.onmessage = (event) => {
      try {
        const message = JSON.parse(event.data) as WebSocketMessage
        Effect.runSync(Queue.offer(queue, message))
      } catch (error) {
        console.error("Failed to parse message:", error)
      }
    }
    
    ws.onerror = (error) => {
      console.error("WebSocket error:", error)
    }
    
    ws.onclose = () => {
      Effect.runSync(Ref.set(isConnected, false))
      Effect.runSync(Queue.shutdown(queue))
    }
    
    // Create stream from queue
    const stream = Stream.fromQueue(queue).pipe(
      Stream.takeUntilEffect(
        Effect.delay(
          Ref.get(isConnected).pipe(
            Effect.map(connected => !connected)
          ),
          Duration.seconds(1)
        )
      )
    )
    
    return { stream, ws, isConnected }
  })
  
  // Stream with automatic reconnection
  return Stream.unwrapScoped(
    Effect.gen(function* () {
      const connection = yield* connect)
      
      yield* 
        Effect.addFinalizer(() =>
          Effect.sync(() => {
            connection.ws.close()
            console.log("WebSocket closed")
          })
        )
      )
      
      return connection.stream
    })
  ).pipe(
    // Retry on failure with exponential backoff
    Stream.retry(
      Schedule.exponential(Duration.seconds(1)).pipe(
        Schedule.compose(Schedule.recurs(5))
      )
    )
  )
}

// Process real-time market data
interface MarketData {
  symbol: string
  price: number
  volume: number
  timestamp: number
}

const processMarketData = (wsUrl: string) =>
  createWebSocketStream(wsUrl).pipe(
    // Filter for data messages only
    Stream.filter(msg => msg.type === "data"),
    
    // Parse market data
    Stream.map(msg => msg.payload as MarketData),
    
    // Calculate moving average
    Stream.groupByKey(data => data.symbol),
    Stream.flatMap(([symbol, dataStream]) =>
      dataStream.pipe(
        Stream.scan(
          { prices: [] as number[], average: 0 },
          (acc, data) => {
            const prices = [...acc.prices.slice(-19), data.price]
            const average = prices.reduce((a, b) => a + b, 0) / prices.length
            return { prices, average }
          }
        ),
        Stream.map(({ average }) => ({
          symbol,
          movingAverage: average,
          timestamp: Date.now()
        }))
      )
    ),
    
    // Throttle output to prevent overwhelming downstream
    Stream.throttle({
      cost: 1,
      duration: Duration.millis(100),
      units: 1
    })
  )

// Alert system for price movements
const createPriceAlerts = (wsUrl: string, threshold: number) =>
  Effect.gen(function* () {
    const alertQueue = yield* Queue.unbounded<string>())
    
    const processor = yield* 
      processMarketData(wsUrl).pipe(
        Stream.tap(data => {
          if (Math.random() > 0.95) { // Simulate significant movement
            return Queue.offer(
              alertQueue,
              `ALERT: ${data.symbol} moving average: ${data.movingAverage.toFixed(2)}`
            )
          }
          return Effect.void
        }),
        Stream.runDrain,
        Effect.fork
      )
    )
    
    // Process alerts separately
    yield* 
      Stream.fromQueue(alertQueue).pipe(
        Stream.tap(alert => Effect.log(alert)),
        Stream.runDrain
      )
    )
    
    yield* Fiber.join(processor)
  })
```

### Example 3: ETL Pipeline - Data Transformation and Loading

Building a complete ETL pipeline that extracts data from multiple sources, transforms it, and loads it into a destination.

```typescript
import { Stream, Effect, Chunk, Schema } from "effect"
import * as NodeFS from "@effect/platform-node/NodeFileSystem"

// Define schemas for data validation
const RawCustomerSchema = Schema.Struct({
  id: Schema.String,
  name: Schema.String,
  email: Schema.String,
  created_at: Schema.String,
  orders: Schema.Array(Schema.Struct({
    order_id: Schema.String,
    amount: Schema.Number,
    date: Schema.String
  }))
})

const TransformedCustomerSchema = Schema.Struct({
  customerId: Schema.String,
  fullName: Schema.String,
  email: Schema.String.pipe(Schema.lowercased()),
  createdAt: Schema.Date,
  totalOrders: Schema.Number,
  totalSpent: Schema.Number,
  averageOrderValue: Schema.Number,
  lastOrderDate: Schema.Date.pipe(Schema.optional)
})

type RawCustomer = Schema.Schema.Type<typeof RawCustomerSchema>
type TransformedCustomer = Schema.Schema.Type<typeof TransformedCustomerSchema>

// Extract data from multiple CSV files
const extractFromCSV = (filePath: string) =>
  Stream.fromReadableStream(
    () => NodeFS.NodeFileSystem.stream(filePath),
    () => new Error(`Failed to read CSV: ${filePath}`)
  ).pipe(
    Stream.decodeText(),
    Stream.splitLines,
    Stream.drop(1), // Skip header
    Stream.map(line => line.split(',')),
    Stream.map(fields => ({
      id: fields[0],
      name: fields[1],
      email: fields[2],
      created_at: fields[3],
      orders: [] // Will be enriched later
    }))
  )

// Enrich with order data
const enrichWithOrders = (customer: RawCustomer) =>
  Effect.gen(function* () {
    // Simulate fetching orders from a database
    const orders = yield* Effect.succeed([
      { order_id: "ORD001", amount: 99.99, date: "2024-01-15" },
      { order_id: "ORD002", amount: 149.99, date: "2024-02-20" }
    ]))
    
    return { ...customer, orders }
  })

// Transform raw data to target schema
const transformCustomer = (raw: RawCustomer): TransformedCustomer => {
  const totalSpent = raw.orders.reduce((sum, order) => sum + order.amount, 0)
  const lastOrder = raw.orders
    .map(o => ({ ...o, date: new Date(o.date) }))
    .sort((a, b) => b.date.getTime() - a.date.getTime())[0]
  
  return {
    customerId: raw.id,
    fullName: raw.name.trim(),
    email: raw.email.toLowerCase(),
    createdAt: new Date(raw.created_at),
    totalOrders: raw.orders.length,
    totalSpent,
    averageOrderValue: raw.orders.length > 0 ? totalSpent / raw.orders.length : 0,
    lastOrderDate: lastOrder?.date
  }
}

// Create ETL pipeline
const createETLPipeline = (
  sourceFiles: string[],
  destinationPath: string
) => {
  // Extract phase - read from multiple sources
  const extract = Stream.fromIterable(sourceFiles).pipe(
    Stream.flatMap(extractFromCSV),
    Stream.mapEffect(raw => 
      Schema.decode(RawCustomerSchema)(raw).pipe(
        Effect.catchTag("ParseError", () => 
          Effect.fail(new Error(`Invalid customer data: ${JSON.stringify(raw)}`))
        )
      )
    )
  )
  
  // Transform phase - enrich and transform data
  const transform = extract.pipe(
    // Enrich with additional data
    Stream.mapEffect(enrichWithOrders),
    
    // Apply transformations
    Stream.map(transformCustomer),
    
    // Validate transformed data
    Stream.mapEffect(data =>
      Schema.decode(TransformedCustomerSchema)(data).pipe(
        Effect.catchTag("ParseError", () =>
          Effect.fail(new Error(`Invalid transformed data: ${JSON.stringify(data)}`))
        )
      )
    ),
    
    // Remove duplicates
    Stream.dedupe((a, b) => a.customerId === b.customerId),
    
    // Sort by total spent
    Stream.groupBy(
      () => true, // Single group
      (_, stream) => 
        stream.pipe(
          Stream.runCollect,
          Effect.map(chunk =>
            Chunk.sort(
              chunk,
              (a, b) => b.totalSpent - a.totalSpent
            )
          ),
          Stream.fromEffect,
          Stream.flattenChunks
        )
    )
  )
  
  // Load phase - write to destination
  const load = transform.pipe(
    // Batch for efficient writing
    Stream.groupedWithin(100, Duration.seconds(5)),
    
    // Convert to JSON lines format
    Stream.map(chunk =>
      Chunk.toReadonlyArray(chunk)
        .map(customer => JSON.stringify(customer))
        .join('\n') + '\n'
    ),
    
    // Write to destination file
    Stream.runForEach(batch =>
      NodeFS.NodeFileSystem.writeFileString(
        destinationPath,
        batch,
        { flag: "a" } // Append mode
      )
    )
  )
  
  return load
}

// Monitor ETL progress
const monitoredETL = (
  sourceFiles: string[],
  destinationPath: string
) => {
  const progress = Ref.make({ processed: 0, errors: 0 })
  
  return Effect.gen(function* () {
    const progressRef = yield* progress)
    
    // Start progress reporting
    const reporter = yield* 
      Effect.repeat(
        Effect.gen(function* () {
          const stats = yield* Ref.get(progressRef))
          yield* Effect.log(
            `ETL Progress: ${stats.processed} processed, ${stats.errors} errors`
          ))
        }),
        Schedule.spaced(Duration.seconds(10))
      ).pipe(Effect.fork)
    )
    
    // Run ETL with monitoring
    yield* 
      createETLPipeline(sourceFiles, destinationPath).pipe(
        Effect.tap(() =>
          Ref.update(progressRef, stats => ({
            ...stats,
            processed: stats.processed + 1
          }))
        ),
        Effect.catchAll(error =>
          Ref.update(progressRef, stats => ({
            ...stats,
            errors: stats.errors + 1
          })).pipe(
            Effect.zipRight(Effect.fail(error))
          )
        )
      )
    )
    
    yield* Fiber.interrupt(reporter))
    
    const finalStats = yield* Ref.get(progressRef))
    yield* Effect.log(`ETL Complete: ${JSON.stringify(finalStats)}`))
    
    return finalStats
  })
}
```

## Advanced Features Deep Dive

### Backpressure and Flow Control

Control the rate of data processing to prevent overwhelming consumers and ensure system stability.

#### Basic Backpressure with Throttling

```typescript
import { Stream, Duration, Effect } from "effect"

// Throttle by number of elements
const throttledByCount = Stream.make(1, 2, 3, 4, 5).pipe(
  Stream.throttle({
    cost: 1,
    duration: Duration.seconds(1),
    units: 2 // Allow 2 elements per second
  })
)

// Throttle with custom cost function
const throttledByCost = Stream.make(
  { size: 100 },
  { size: 50 },
  { size: 200 }
).pipe(
  Stream.throttle({
    cost: (item) => item.size, // Cost based on item size
    duration: Duration.seconds(1),
    units: 150 // Allow 150 units per second
  })
)
```

#### Real-World Backpressure: API Rate Limiting

```typescript
interface APIRequest {
  endpoint: string
  method: "GET" | "POST"
  body?: unknown
}

const createRateLimitedAPIStream = (
  requests: Stream.Stream<APIRequest>,
  apiKey: string,
  maxRequestsPerMinute: number
) => {
  // Track API quota usage
  const quotaRef = Ref.make({ used: 0, resetAt: Date.now() + 60000 })
  
  return Effect.gen(function* () {
    const quota = yield* quotaRef)
    
    return requests.pipe(
      // Apply rate limiting
      Stream.throttle({
        cost: 1,
        duration: Duration.minutes(1),
        units: maxRequestsPerMinute
      }),
      
      // Make API calls with quota tracking
      Stream.mapEffect(request =>
        Effect.gen(function* () {
          const currentQuota = yield* Ref.get(quota))
          
          // Check if quota reset is needed
          if (Date.now() > currentQuota.resetAt) {
            yield* Ref.set(quota, {
              used: 0,
              resetAt: Date.now() + 60000
            }))
          }
          
          // Make the API call
          const response = yield* 
            Effect.tryPromise({
              try: () => fetch(request.endpoint, {
                method: request.method,
                headers: {
                  'Authorization': `Bearer ${apiKey}`,
                  'Content-Type': 'application/json'
                },
                body: request.body ? JSON.stringify(request.body) : undefined
              }),
              catch: (error) => new Error(`API call failed: ${error}`)
            })
          )
          
          // Update quota usage
          yield* Ref.update(quota, q => ({
            ...q,
            used: q.used + 1
          })))
          
          return response
        })
      ),
      
      // Handle rate limit errors with retry
      Stream.retry(
        Schedule.exponential(Duration.seconds(1)).pipe(
          Schedule.whileOutput(Duration.lessThanOrEqualTo(Duration.minutes(1)))
        )
      )
    )
  }).pipe(Effect.flatten)
}
```

#### Advanced: Dynamic Backpressure with Queue

```typescript
const createAdaptiveStream = <A>(
  producer: Stream.Stream<A>,
  slowConsumerSimulation: boolean = false
) =>
  Effect.gen(function* () {
    // Create a bounded queue for backpressure
    const queue = yield* Queue.bounded<A>(10))
    
    // Producer fiber
    const producerFiber = yield* 
      producer.pipe(
        Stream.runForEach(item =>
          Queue.offer(queue).pipe(
            Effect.zipRight(Effect.log(`Produced: ${JSON.stringify(item)}`))
          )
        ),
        Effect.ensuring(Queue.shutdown(queue)),
        Effect.fork
      )
    )
    
    // Consumer stream with adaptive processing
    const consumer = Stream.fromQueue(queue).pipe(
      Stream.mapEffect(item =>
        Effect.gen(function* () {
          // Simulate variable processing time
          const processingTime = slowConsumerSimulation
            ? Duration.seconds(1)
            : Duration.millis(100)
            
          yield* Effect.sleep(processingTime))
          yield* Effect.log(`Consumed: ${JSON.stringify(item)}`))
          
          return item
        })
      )
    )
    
    return consumer
  }).pipe(Stream.unwrap)
```

### Concurrent Stream Processing

Process stream elements concurrently for improved performance while maintaining order or allowing unordered processing.

#### Basic Concurrent Processing

```typescript
// Process elements concurrently, maintaining order
const concurrentOrdered = Stream.make(1, 2, 3, 4, 5).pipe(
  Stream.mapEffectPar(2)(n =>
    Effect.gen(function* () {
      yield* Effect.sleep(Duration.seconds(Math.random())))
      yield* Effect.log(`Processing ${n}`))
      return n * 2
    })
  )
)

// Process concurrently without preserving order
const concurrentUnordered = Stream.make(1, 2, 3, 4, 5).pipe(
  Stream.flatMapPar(2)(n =>
    Stream.fromEffect(
      Effect.gen(function* () {
        yield* Effect.sleep(Duration.seconds(Math.random())))
        return n * 2
      })
    )
  )
)
```

#### Real-World Concurrent Processing: Parallel Data Enrichment

```typescript
interface UserProfile {
  id: string
  name: string
  email: string
}

interface EnrichedUser {
  profile: UserProfile
  preferences: Record<string, unknown>
  activityScore: number
  recommendations: string[]
}

// Simulate external service calls
const fetchUserPreferences = (userId: string) =>
  Effect.gen(function* () {
    yield* Effect.sleep(Duration.millis(100)))
    return {
      theme: "dark",
      notifications: true,
      language: "en"
    }
  })

const calculateActivityScore = (userId: string) =>
  Effect.gen(function* () {
    yield* Effect.sleep(Duration.millis(150)))
    return Math.floor(Math.random() * 100)
  })

const getRecommendations = (userId: string) =>
  Effect.gen(function* () {
    yield* Effect.sleep(Duration.millis(200)))
    return ["Product A", "Product B", "Product C"]
  })

// Enrich user profiles concurrently
const enrichUserProfiles = (users: Stream.Stream<UserProfile>) =>
  users.pipe(
    Stream.mapEffectPar(5)(user =>
      Effect.gen(function* () {
        // Fetch all enrichment data in parallel
        const [preferences, activityScore, recommendations] = yield* 
          Effect.all([
            fetchUserPreferences(user.id),
            calculateActivityScore(user.id),
            getRecommendations(user.id)
          ], { concurrency: "unbounded" })
        )
        
        return {
          profile: user,
          preferences,
          activityScore,
          recommendations
        } satisfies EnrichedUser
      })
    )
  )

// Advanced: Concurrent processing with different strategies
const createProcessingPipeline = (
  users: Stream.Stream<UserProfile>,
  config: {
    enrichmentConcurrency: number
    batchSize: number
    timeout: Duration.Duration
  }
) =>
  users.pipe(
    // Batch users for efficient processing
    Stream.groupedWithin(config.batchSize, config.timeout),
    
    // Process each batch concurrently
    Stream.flatMapPar(2)(batch =>
      Stream.fromIterable(batch).pipe(
        Stream.mapEffectPar(config.enrichmentConcurrency)(user =>
          Effect.gen(function* () {
            const enriched = yield* 
              Effect.all({
                profile: Effect.succeed(user),
                preferences: fetchUserPreferences(user.id),
                activityScore: calculateActivityScore(user.id),
                recommendations: getRecommendations(user.id)
              }).pipe(
                Effect.timeout(config.timeout),
                Effect.catchTag("TimeoutException", () =>
                  Effect.succeed({
                    profile: user,
                    preferences: {},
                    activityScore: 0,
                    recommendations: []
                  })
                )
              )
            )
            
            return enriched
          })
        )
      )
    )
  )
```

### Stream Composition and Merging

Combine multiple streams in various ways to create complex data processing pipelines.

#### Basic Stream Composition

```typescript
// Merge two streams
const merged = Stream.merge(
  Stream.make(1, 3, 5),
  Stream.make(2, 4, 6)
)

// Zip streams together
const zipped = Stream.zip(
  Stream.make("a", "b", "c"),
  Stream.make(1, 2, 3)
)

// Combine latest values
const combined = Stream.combineLatest(
  Stream.make(1, 2, 3),
  Stream.make("a", "b", "c")
)
```

#### Real-World Composition: Multi-Source Data Aggregation

```typescript
interface SensorReading {
  sensorId: string
  value: number
  timestamp: number
  type: "temperature" | "humidity" | "pressure"
}

interface AggregatedReading {
  timestamp: number
  temperature?: number
  humidity?: number
  pressure?: number
  anomalyDetected: boolean
}

// Create sensor streams
const createSensorStream = (
  sensorId: string,
  type: SensorReading["type"],
  interval: Duration.Duration
) =>
  Stream.repeatEffectWithSchedule(
    Effect.gen(function* () {
      const value = type === "temperature" 
        ? 20 + Math.random() * 10
        : type === "humidity"
        ? 40 + Math.random() * 20
        : 1000 + Math.random() * 50
        
      return {
        sensorId,
        value,
        timestamp: Date.now(),
        type
      } satisfies SensorReading
    }),
    Schedule.spaced(interval)
  )

// Aggregate multiple sensor streams
const createSensorAggregator = () => {
  const temperatureStream = createSensorStream(
    "temp-001",
    "temperature",
    Duration.seconds(1)
  )
  
  const humidityStream = createSensorStream(
    "humid-001",
    "humidity",
    Duration.seconds(1.5)
  )
  
  const pressureStream = createSensorStream(
    "press-001",
    "pressure",
    Duration.seconds(2)
  )
  
  // Merge all sensor streams
  return Stream.mergeAll([
    temperatureStream,
    humidityStream,
    pressureStream
  ]).pipe(
    // Group readings by time window
    Stream.groupedWithin(10, Duration.seconds(5)),
    
    // Aggregate readings in each window
    Stream.map(readings => {
      const byType = Chunk.toReadonlyArray(readings).reduce(
        (acc, reading) => {
          acc[reading.type] = reading.value
          return acc
        },
        {} as Record<string, number>
      )
      
      const aggregated: AggregatedReading = {
        timestamp: Date.now(),
        temperature: byType.temperature,
        humidity: byType.humidity,
        pressure: byType.pressure,
        anomalyDetected: false
      }
      
      // Simple anomaly detection
      if (
        (aggregated.temperature && aggregated.temperature > 28) ||
        (aggregated.humidity && aggregated.humidity > 55) ||
        (aggregated.pressure && aggregated.pressure < 1005)
      ) {
        aggregated.anomalyDetected = true
      }
      
      return aggregated
    }),
    
    // Filter for anomalies
    Stream.tap(reading => {
      if (reading.anomalyDetected) {
        return Effect.log(
          `ANOMALY DETECTED at ${new Date(reading.timestamp).toISOString()}: ` +
          `Temp=${reading.temperature?.toFixed(1)}°C, ` +
          `Humidity=${reading.humidity?.toFixed(1)}%, ` +
          `Pressure=${reading.pressure?.toFixed(1)}hPa`
        )
      }
      return Effect.void
    })
  )
}

// Advanced: Dynamic stream composition
const createDynamicPipeline = <A>(
  primaryStream: Stream.Stream<A>,
  enrichmentStreams: Map<string, Stream.Stream<unknown>>
) =>
  Effect.gen(function* () {
    // Create a switchable stream
    const switchSignal = yield* Ref.make<string>("default"))
    
    return primaryStream.pipe(
      Stream.flatMap(item =>
        Stream.fromEffect(
          Effect.gen(function* () {
            const mode = yield* Ref.get(switchSignal)
            const enrichmentStream = enrichmentStreams.get(mode)
            
            if (enrichmentStream) {
              const chunk = yield* enrichmentStream.pipe(
                Stream.take(1),
                Stream.map(enrichment => ({
                  item,
                  enrichment,
                  mode
                })),
                Stream.runCollect
              )
              return Chunk.toReadonlyArray(chunk)[0]
            }
            
            return { item, enrichment: null, mode }
          })
        )
      )
    )
  }).pipe(Stream.unwrap)
```

## Practical Patterns & Best Practices

### Pattern 1: Resource-Safe Stream Processing

Create streams that properly manage resources and handle cleanup automatically.

```typescript
import { Stream, Effect, Scope, Resource } from "effect"
import * as NodeFS from "@effect/platform-node/NodeFileSystem"
import { createReadStream, createWriteStream } from "fs"

// Helper for creating resource-safe file streams
const fileStreamResource = (path: string, options?: { encoding?: string }) =>
  Resource.make(
    Effect.sync(() => {
      const stream = createReadStream(path, options)
      return stream
    }),
    (stream) => Effect.sync(() => stream.destroy())
  )

// Helper for resource-safe stream processing
const withFileStream = <R, E, A>(
  filePath: string,
  process: (stream: NodeJS.ReadableStream) => Stream.Stream<A, E, R>
) =>
  Stream.unwrapScoped(
    Effect.gen(function* () {
      const resource = yield* fileStreamResource(filePath))
      const stream = yield* Resource.get(resource))
      return process(stream)
    })
  )

// Example: Process large CSV with automatic resource cleanup
const processLargeCSV = (csvPath: string) =>
  withFileStream(csvPath, (nodeStream) =>
    Stream.fromReadableStream(
      () => nodeStream,
      () => new Error("Stream error")
    ).pipe(
      Stream.decodeText(),
      Stream.splitLines,
      Stream.drop(1), // Skip header
      Stream.map(line => line.split(',')),
      Stream.map(fields => ({
        id: fields[0],
        name: fields[1],
        value: parseFloat(fields[2])
      })),
      Stream.filter(record => record.value > 100)
    )
  )

// Helper for creating write streams with buffering
const createBufferedWriter = (
  outputPath: string,
  bufferSize: number = 1000
) => {
  const writeToFile = (items: Chunk.Chunk<string>) =>
    Effect.gen(function* () {
      const content = Chunk.toReadonlyArray(items).join('\n') + '\n'
      yield* 
        NodeFS.NodeFileSystem.writeFileString(
          outputPath,
          content,
          { flag: 'a' }
        )
      )
    })
  
  return <A>(transform: (a: A) => string) =>
    (stream: Stream.Stream<A>) =>
      stream.pipe(
        Stream.map(transform),
        Stream.groupedWithin(bufferSize, Duration.seconds(5)),
        Stream.mapEffect(writeToFile),
        Stream.drain
      )
}
```

### Pattern 2: Stream Error Recovery and Resilience

Build resilient stream processing with sophisticated error handling and recovery strategies.

```typescript
// Helper for creating resilient streams
const createResilientStream = <A, E = never>(
  source: () => Stream.Stream<A, E>,
  options: {
    maxRetries?: number
    retryDelay?: Duration.Duration
    fallbackValue?: A
    onError?: (error: E) => Effect.Effect<void>
  } = {}
) => {
  const {
    maxRetries = 3,
    retryDelay = Duration.seconds(1),
    fallbackValue,
    onError = () => Effect.void
  } = options
  
  return source().pipe(
    // Add retry logic
    Stream.retry(
      Schedule.recurs(maxRetries).pipe(
        Schedule.addDelay(() => retryDelay)
      )
    ),
    
    // Handle errors with fallback
    Stream.catchAll((error) =>
      Stream.fromEffect(
        onError(error).pipe(
          Effect.zipRight(
            fallbackValue !== undefined
              ? Effect.succeed(fallbackValue)
              : Effect.fail(error)
          )
        )
      )
    )
  )
}

// Helper for circuit breaker pattern
const createCircuitBreakerStream = <A>(
  source: Stream.Stream<A>,
  options: {
    failureThreshold: number
    resetTimeout: Duration.Duration
    halfOpenLimit: number
  }
) =>
  Effect.gen(function* () {
    const state = yield* 
      Ref.make<
        | { type: "closed"; failures: number }
        | { type: "open"; openedAt: number }
        | { type: "halfOpen"; successes: number }
      >({ type: "closed", failures: 0 })
    )
    
    const checkCircuit = Effect.gen(function* () {
      const current = yield* Ref.get(state))
      
      switch (current.type) {
        case "open": {
          const elapsed = Date.now() - current.openedAt
          if (elapsed > Duration.toMillis(options.resetTimeout)) {
            yield* Ref.set(state, { type: "halfOpen", successes: 0 }))
            return true
          }
          return false
        }
        case "halfOpen":
          return current.successes < options.halfOpenLimit
        case "closed":
          return true
      }
    })
    
    const recordSuccess = Ref.update(state, (current) => {
      switch (current.type) {
        case "halfOpen":
          return current.successes + 1 >= options.halfOpenLimit
            ? { type: "closed", failures: 0 }
            : { type: "halfOpen", successes: current.successes + 1 }
        default:
          return { type: "closed", failures: 0 }
      }
    })
    
    const recordFailure = Ref.update(state, (current) => {
      switch (current.type) {
        case "closed":
          return current.failures + 1 >= options.failureThreshold
            ? { type: "open", openedAt: Date.now() }
            : { type: "closed", failures: current.failures + 1 }
        case "halfOpen":
          return { type: "open", openedAt: Date.now() }
        default:
          return current
      }
    })
    
    return source.pipe(
      Stream.filterEffect(() => checkCircuit),
      Stream.tap(() => recordSuccess),
      Stream.catchAll((error) =>
        Stream.fromEffect(
          recordFailure.pipe(
            Effect.zipRight(Effect.fail(error))
          )
        )
      )
    )
  }).pipe(Stream.unwrap)

// Example: Resilient API stream
const createResilientAPIStream = (apiEndpoints: string[]) => {
  const primaryEndpoint = apiEndpoints[0]
  const fallbackEndpoints = apiEndpoints.slice(1)
  
  const createAPIStream = (endpoint: string) =>
    Stream.repeatEffectWithSchedule(
      Effect.tryPromise({
        try: () => fetch(endpoint).then(r => r.json()),
        catch: () => new Error(`Failed to fetch from ${endpoint}`)
      }),
      Schedule.spaced(Duration.seconds(5))
    )
  
  return createResilientStream(
    () => createAPIStream(primaryEndpoint),
    {
      maxRetries: 2,
      retryDelay: Duration.seconds(2),
      onError: (error) => Effect.log(`Primary endpoint failed: ${error}`)
    }
  ).pipe(
    Stream.catchAll(() => {
      // Try fallback endpoints
      return Stream.fromIterable(fallbackEndpoints).pipe(
        Stream.flatMap(endpoint =>
          createResilientStream(
            () => createAPIStream(endpoint),
            { maxRetries: 1 }
          )
        ),
        Stream.take(1)
      )
    })
  )
}
```

### Pattern 3: Stream Analytics and Aggregation

Build reusable patterns for real-time analytics and aggregation over streams.

```typescript
// Helper for time-based windowing
const windowByTime = <A>(
  window: Duration.Duration,
  maxSize: number = 1000
) => (stream: Stream.Stream<A>) =>
  stream.pipe(
    Stream.groupedWithin(maxSize, window),
    Stream.map(chunk => ({
      window: {
        start: Date.now() - Duration.toMillis(window),
        end: Date.now()
      },
      items: chunk
    }))
  )

// Helper for creating rolling aggregations
const rollingAggregate = <A, B>(
  windowSize: number,
  aggregate: (items: ReadonlyArray<A>) => B
) => (stream: Stream.Stream<A>) =>
  stream.pipe(
    Stream.scan(
      [] as A[],
      (buffer, item) => [...buffer.slice(-(windowSize - 1)), item]
    ),
    Stream.drop(windowSize - 1), // Wait for full window
    Stream.map(buffer => aggregate(buffer))
  )

// Helper for creating histograms
interface Histogram {
  buckets: Map<string, number>
  total: number
  mean: number
  p50: number
  p95: number
  p99: number
}

const createHistogram = (
  bucketSize: number,
  maxBuckets: number = 10
) => <A>(getValue: (a: A) => number) =>
  (stream: Stream.Stream<A>) =>
    stream.pipe(
      Stream.scan(
        {
          values: [] as number[],
          buckets: new Map<string, number>()
        },
        (state, item) => {
          const value = getValue(item)
          const bucket = Math.floor(value / bucketSize) * bucketSize
          const bucketKey = `${bucket}-${bucket + bucketSize}`
          
          const newBuckets = new Map(state.buckets)
          newBuckets.set(bucketKey, (newBuckets.get(bucketKey) || 0) + 1)
          
          // Keep only top buckets
          if (newBuckets.size > maxBuckets) {
            const sorted = Array.from(newBuckets.entries())
              .sort((a, b) => b[1] - a[1])
              .slice(0, maxBuckets)
            newBuckets.clear()
            sorted.forEach(([k, v]) => newBuckets.set(k, v))
          }
          
          return {
            values: [...state.values, value].slice(-1000), // Keep last 1000
            buckets: newBuckets
          }
        }
      ),
      Stream.map(state => {
        const sorted = [...state.values].sort((a, b) => a - b)
        const total = sorted.length
        
        return {
          buckets: state.buckets,
          total,
          mean: sorted.reduce((a, b) => a + b, 0) / total,
          p50: sorted[Math.floor(total * 0.5)],
          p95: sorted[Math.floor(total * 0.95)],
          p99: sorted[Math.floor(total * 0.99)]
        } satisfies Histogram
      })
    )

// Example: Real-time performance monitoring
interface PerformanceMetric {
  operation: string
  duration: number
  timestamp: number
  success: boolean
}

const createPerformanceMonitor = (
  metrics: Stream.Stream<PerformanceMetric>
) => {
  const byOperation = metrics.pipe(
    Stream.groupByKey(m => m.operation),
    Stream.flatMap(([operation, opStream]) =>
      opStream.pipe(
        windowByTime(Duration.minutes(1)),
        Stream.map(window => {
          const items = Chunk.toReadonlyArray(window.items)
          const durations = items.map(m => m.duration)
          const successRate = items.filter(m => m.success).length / items.length
          
          return {
            operation,
            window: window.window,
            avgDuration: durations.reduce((a, b) => a + b, 0) / durations.length,
            maxDuration: Math.max(...durations),
            minDuration: Math.min(...durations),
            successRate,
            count: items.length
          }
        })
      )
    )
  )
  
  const overall = metrics.pipe(
    createHistogram(10, 20)(m => m.duration),
    Stream.sample(Duration.seconds(10))
  )
  
  return { byOperation, overall }
}
```

## Integration Examples

### Integration with Effect for Error Handling and Resource Management

```typescript
import { Stream, Effect, Layer, Context, Resource } from "effect"
import * as NodeFS from "@effect/platform-node/NodeFileSystem"
import { Pool } from "pg"

// Define service interfaces
interface DatabaseService {
  readonly query: <T>(sql: string) => Effect.Effect<T[]>
  readonly stream: <T>(sql: string) => Stream.Stream<T>
}

const DatabaseService = Context.GenericTag<DatabaseService>("DatabaseService")

// Create database layer
const DatabaseLive = Layer.scoped(
  DatabaseService,
  Effect.gen(function* () {
    const pool = yield* 
      Resource.make(
        Effect.sync(() => new Pool({
          connectionString: process.env.DATABASE_URL
        })),
        (pool) => Effect.promise(() => pool.end())
      )
    )
    
    const poolResource = yield* Resource.get(pool))
    
    return DatabaseService.of({
      query: <T>(sql: string) =>
        Effect.tryPromise({
          try: () => poolResource.query(sql).then(r => r.rows as T[]),
          catch: (error) => new Error(`Query failed: ${error}`)
        }),
      
      stream: <T>(sql: string) =>
        Stream.unwrapEffect(
          Effect.gen(function* () {
            const client = yield* 
              Effect.tryPromise({
                try: () => poolResource.connect(),
                catch: () => new Error("Failed to get client")
              })
            )
            
            yield* 
              Effect.addFinalizer(() =>
                Effect.sync(() => client.release())
              )
            )
            
            return Stream.async<T>((emit) => {
              const query = client.query(sql)
              
              query.on('row', (row) => emit.single(row))
              query.on('end', () => emit.end)
              query.on('error', (error) => emit.fail(error))
            })
          })
        )
    })
  })
)

// Example: ETL pipeline with Effect integration
const runETLPipeline = Effect.gen(function* () {
  const db = yield* DatabaseService)
  const fs = yield* NodeFS.NodeFileSystem)
  
  // Extract from database
  const extract = db.stream<{
    id: number
    data: unknown
    created_at: Date
  }>(`
    SELECT id, data, created_at 
    FROM raw_events 
    WHERE created_at > NOW() - INTERVAL '1 day'
  `)
  
  // Transform with validation
  const transform = extract.pipe(
    Stream.mapEffect(row =>
      Schema.decode(EventSchema)(row.data).pipe(
        Effect.map(validated => ({
          ...row,
          validated
        })),
        Effect.catchTag("ParseError", (error) =>
          Effect.gen(function* () {
            yield* Effect.log(`Validation failed for row ${row.id}: ${error}`))
            return Effect.fail(error)
          }).pipe(Effect.flatten)
        )
      )
    ),
    
    // Enrich with external data
    Stream.mapEffectPar(5)(row =>
      Effect.gen(function* () {
        const enrichment = yield* 
          Effect.tryPromise({
            try: () => fetch(`/api/enrich/${row.id}`).then(r => r.json()),
            catch: () => new Error("Enrichment failed")
          }).pipe(
            Effect.retry(Schedule.exponential(Duration.seconds(1), 2).pipe(
              Schedule.compose(Schedule.recurs(3))
            ))
          )
        )
        
        return { ...row, enrichment }
      })
    )
  )
  
  // Load to file system
  const outputPath = "/data/processed/events.jsonl"
  const load = transform.pipe(
    Stream.groupedWithin(100, Duration.seconds(10)),
    Stream.mapEffect(batch =>
      fs.writeFileString(
        outputPath,
        Chunk.toReadonlyArray(batch)
          .map(item => JSON.stringify(item))
          .join('\n') + '\n',
        { flag: 'a' }
      )
    ),
    Stream.runCount
  )
  
  const processedCount = yield* load)
  yield* Effect.log(`ETL complete: ${processedCount} batches processed`))
  
  return processedCount
}).pipe(
  Effect.provide(DatabaseLive),
  Effect.catchAll(error =>
    Effect.gen(function* () {
      yield* Effect.log(`ETL failed: ${error}`))
      // Send alert
      yield* 
        Effect.tryPromise({
          try: () => fetch('/api/alerts', {
            method: 'POST',
            body: JSON.stringify({
              type: 'etl_failure',
              error: String(error),
              timestamp: new Date()
            })
          }),
          catch: () => new Error("Failed to send alert")
        })
      )
      return 0
    })
  )
)
```

### Testing Strategies for Streams

```typescript
import { Stream, Effect, TestClock, TestContext, Fiber, Duration } from "effect"
import { describe, it, expect } from "@effect/vitest"

// Helper for testing stream outputs
const collectStream = <A>(stream: Stream.Stream<A>) =>
  stream.pipe(
    Stream.runCollect,
    Effect.map(chunk => Chunk.toReadonlyArray(chunk))
  )

// Helper for testing time-based streams
const testTimeBasedStream = <A>(
  stream: Stream.Stream<A>,
  advances: Duration.Duration[],
  expectedCounts: number[]
) =>
  Effect.gen(function* () {
    const fiber = yield* Stream.runCollect(stream).pipe(Effect.fork))
    
    for (let i = 0; i < advances.length; i++) {
      yield* TestClock.adjust(advances[i]))
      
      const result = yield* Fiber.poll(fiber))
      if (result._tag === "Some") {
        const items = Chunk.toReadonlyArray(result.value)
        expect(items.length).toBe(expectedCounts[i])
      }
    }
    
    return yield* Fiber.join(fiber))
  })

// Test data generators
const createTestStream = <A>(items: A[], delay?: Duration.Duration) =>
  delay
    ? Stream.fromIterable(items).pipe(
        Stream.schedule(Schedule.spaced(delay))
      )
    : Stream.fromIterable(items)

describe("Stream Processing", () => {
  it("should handle backpressure correctly", () =>
    Effect.gen(function* () {
      const source = Stream.iterate(1, n => n + 1).pipe(
        Stream.take(100)
      )
      
      const processed = source.pipe(
        Stream.throttle({
          cost: 1,
          duration: Duration.millis(10),
          units: 5
        }),
        Stream.scan(0, (acc, _) => acc + 1)
      )
      
      const startTime = yield* Effect.sync(() => Date.now()))
      const result = yield* Stream.runLast(processed))
      const endTime = yield* Effect.sync(() => Date.now()))
      
      expect(result).toBe(100)
      expect(endTime - startTime).toBeGreaterThanOrEqual(190) // ~200ms for 100 items at 5/10ms
    })
  )
  
  it("should recover from errors", () =>
    Effect.gen(function* () {
      let attempts = 0
      
      const failingStream = Stream.fromEffect(
        Effect.gen(function* () {
          attempts++
          if (attempts < 3) {
            return yield* Effect.fail(new Error("Temporary failure")))
          }
          return "Success"
        })
      )
      
      const resilient = failingStream.pipe(
        Stream.retry(Schedule.recurs(5))
      )
      
      const result = yield* collectStream(resilient))
      
      expect(result).toEqual(["Success"])
      expect(attempts).toBe(3)
    })
  )
  
  it("should merge streams correctly", () =>
    Effect.gen(function* () {
      const stream1 = createTestStream([1, 3, 5], Duration.millis(10))
      const stream2 = createTestStream([2, 4, 6], Duration.millis(15))
      
      const merged = Stream.merge(stream1, stream2)
      const result = yield* collectStream(merged))
      
      expect(result.length).toBe(6)
      expect(new Set(result)).toEqual(new Set([1, 2, 3, 4, 5, 6]))
    }).pipe(Effect.provide(TestContext.TestContext))
  )
  
  it("should window by time correctly", () =>
    Effect.gen(function* () {
      const source = Stream.repeatEffectWithSchedule(
        Effect.succeed(1),
        Schedule.spaced(Duration.millis(10))
      ).pipe(Stream.take(20))
      
      const windowed = source.pipe(
        Stream.groupedWithin(100, Duration.millis(50)),
        Stream.map(chunk => chunk.length)
      )
      
      const result = yield* 
        testTimeBasedStream(
          windowed,
          [
            Duration.millis(55),
            Duration.millis(55),
            Duration.millis(55),
            Duration.millis(55)
          ],
          [5, 10, 15, 20]
        )
      )
      
      expect(Chunk.toReadonlyArray(result)).toEqual([5, 5, 5, 5])
    }).pipe(Effect.provide(TestContext.TestContext))
  )
})

// Property-based testing for streams
import * as fc from "fast-check"

const streamArbitrary = <A>(arb: fc.Arbitrary<A>) =>
  fc.array(arb).map(arr => Stream.fromIterable(arr))

describe("Stream Properties", () => {
  it("map preserves length", () =>
    fc.assert(
      fc.property(
        streamArbitrary(fc.integer()),
        fc.func(fc.integer()),
        (stream, f) =>
          Effect.gen(function* () {
            const original = yield* collectStream(stream))
            const mapped = yield* collectStream(stream.pipe(Stream.map(f))))
            
            expect(mapped.length).toBe(original.length)
          }).pipe(Effect.runPromise)
      )
    )
  )
  
  it("filter reduces or preserves length", () =>
    fc.assert(
      fc.property(
        streamArbitrary(fc.integer()),
        fc.func(fc.boolean()),
        (stream, predicate) =>
          Effect.gen(function* () {
            const original = yield* collectStream(stream))
            const filtered = yield* 
              collectStream(stream.pipe(Stream.filter(predicate)))
            )
            
            expect(filtered.length).toBeLessThanOrEqual(original.length)
          }).pipe(Effect.runPromise)
      )
    )
  )
})
```

## Conclusion

Stream provides a powerful, composable, and type-safe approach to processing asynchronous data sequences in Effect applications.

Key benefits:
- **Memory Efficiency**: Pull-based processing prevents memory overflow with large datasets
- **Composability**: Rich set of operators for building complex processing pipelines
- **Resource Safety**: Automatic resource management with proper cleanup
- **Error Handling**: Integrated error handling with retry and recovery strategies
- **Concurrency**: Built-in support for concurrent processing with backpressure
- **Type Safety**: Full type inference throughout the pipeline

Stream excels in scenarios involving file processing, real-time data handling, ETL pipelines, and any situation requiring efficient processing of potentially infinite data sequences.