# Streamable: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem Streamable Solves

When working with custom data sources, APIs, or complex streaming scenarios, you often need to create reusable stream abstractions that encapsulate specific business logic while maintaining full compatibility with Effect's Stream ecosystem. Traditional approaches lead to scattered stream creation logic and reduced composability.

```typescript
// Traditional approach - scattered stream creation logic
import { Stream, Effect } from "effect"

// Database cursor handling - no reusability
const getUsersPage1 = Stream.fromEffect(
  Effect.succeed([{ id: 1, name: "Alice" }, { id: 2, name: "Bob" }])
)

const getUsersPage2 = Stream.fromEffect(
  Effect.succeed([{ id: 3, name: "Charlie" }, { id: 4, name: "Diana" }])
)

// API pagination - duplicated logic
const fetchProductsPage = (page: number) =>
  Stream.fromEffect(
    Effect.succeed([
      { id: page * 10, name: `Product ${page * 10}` },
      { id: page * 10 + 1, name: `Product ${page * 10 + 1}` }
    ])
  )

// Real-time event streams - inconsistent patterns
const eventStream = Stream.async<Event, never>((emit) => {
  const listener = (event: Event) => emit.single(event)
  addEventListener('custom', listener)
  return Effect.sync(() => removeEventListener('custom', listener))
})
```

This approach leads to:
- **Code Duplication** - Similar stream creation patterns repeated throughout the codebase
- **Poor Composability** - Difficult to combine and extend custom streaming logic
- **Inconsistent APIs** - Different patterns for similar data source abstractions
- **Limited Reusability** - Stream creation logic tightly coupled to specific implementations

### The Streamable Solution

Effect's Streamable module provides an abstract base class that lets you create custom Stream types with encapsulated behavior, unified interfaces, and full Stream ecosystem compatibility.

```typescript
import { Stream, Effect, Streamable } from "effect"

// The Streamable solution - reusable, composable, type-safe
class DatabaseCursor<T> extends Streamable.Class<T> {
  constructor(
    private query: string,
    private batchSize: number = 100
  ) {
    super()
  }

  toStream(): Stream.Stream<T> {
    return Stream.paginateEffect(0, (offset) =>
      Effect.gen(function* () {
        const results = yield* executeQuery<T>(this.query, offset, this.batchSize)
        const hasMore = results.length === this.batchSize
        return [results, hasMore ? offset + this.batchSize : null] as const
      })
    ).pipe(
      Stream.flatMap(Stream.fromIterable)
    )
  }
}

// Usage - works seamlessly with all Stream operations
const userCursor = new DatabaseCursor<User>("SELECT * FROM users", 50)

const processedUsers = userCursor.pipe(
  Stream.filter(user => user.active),
  Stream.map(user => ({ ...user, processed: true })),
  Stream.take(1000)
)
```

### Key Concepts

**Streamable.Class**: An abstract base class that implements the Stream interface and requires a `toStream()` method implementation.

**Stream Interface Compatibility**: Streamable instances are fully compatible with all Stream operations through automatic delegation.

**Encapsulated Logic**: Business logic for stream creation is contained within the class, promoting reusability and maintainability.

## Basic Usage Patterns

### Pattern 1: Simple Custom Stream

```typescript
import { Stream, Effect, Streamable } from "effect"

class NumberSequence extends Streamable.Class<number> {
  constructor(
    private start: number,
    private end: number,
    private step: number = 1
  ) {
    super()
  }

  toStream(): Stream.Stream<number> {
    return Stream.iterate(this.start, n => n + this.step).pipe(
      Stream.takeWhile(n => n <= this.end)
    )
  }
}

// Usage
const sequence = new NumberSequence(1, 10, 2)
const result = yield* Stream.runCollect(sequence)
// Result: [1, 3, 5, 7, 9]
```

### Pattern 2: Effectful Data Source

```typescript
class ApiDataStream<T> extends Streamable.Class<T, Error> {
  constructor(
    private endpoint: string,
    private headers: Record<string, string> = {}
  ) {
    super()
  }

  toStream(): Stream.Stream<T, Error> {
    return Stream.fromEffect(
      Effect.tryPromise({
        try: () => fetch(this.endpoint, { headers: this.headers })
          .then(res => res.json() as Promise<T[]>),
        catch: (error) => new Error(`API request failed: ${error}`)
      })
    ).pipe(
      Stream.flatMap(Stream.fromIterable)
    )
  }
}

// Usage with proper error handling
const apiStream = new ApiDataStream<Product>("/api/products", {
  "Authorization": "Bearer token123"
})

const products = apiStream.pipe(
  Stream.catchAll(error => 
    Stream.fromEffect(Effect.logError(`Failed to fetch products: ${error.message}`))
      .pipe(Stream.as([]))
  ),
  Stream.take(50)
)
```

### Pattern 3: Configurable Stream with Dependencies

```typescript
interface DatabaseConfig {
  readonly connectionString: string
  readonly timeout: number
}

class ConfigurableQuery<T> extends Streamable.Class<T, Error, DatabaseConfig> {
  constructor(
    private sql: string,
    private params: unknown[] = []
  ) {
    super()
  }

  toStream(): Stream.Stream<T, Error, DatabaseConfig> {
    return Stream.contextWithStream((config: DatabaseConfig) =>
      Stream.fromEffect(
        Effect.tryPromise({
          try: () => executeQuery<T>(config.connectionString, this.sql, this.params),
          catch: (error) => new Error(`Query failed: ${error}`)
        })
      ).pipe(
        Stream.flatMap(Stream.fromIterable),
        Stream.timeout(`${config.timeout}ms`)
      )
    )
  }
}
```

## Real-World Examples

### Example 1: Database Pagination with Error Recovery

A robust database cursor that handles connection failures and implements automatic retry logic.

```typescript
import { Stream, Effect, Streamable, Schedule, Layer } from "effect"

interface DatabaseError {
  readonly _tag: "DatabaseError"
  readonly message: string
  readonly retryable: boolean
}

const DatabaseError = (message: string, retryable = true): DatabaseError => ({
  _tag: "DatabaseError",
  message,
  retryable
})

interface Database {
  readonly query: <T>(sql: string, offset: number, limit: number) => Effect.Effect<T[], DatabaseError>
  readonly getCount: (table: string) => Effect.Effect<number, DatabaseError>
}

class PaginatedQuery<T> extends Streamable.Class<T, DatabaseError, Database> {
  constructor(
    private tableName: string,
    private whereClause: string = "",
    private pageSize: number = 100
  ) {
    super()
  }

  toStream(): Stream.Stream<T, DatabaseError, Database> {
    return Effect.gen(function* () {
      const db = yield* Database
      const sql = `SELECT * FROM ${this.tableName} ${this.whereClause} LIMIT ? OFFSET ?`
      
      return Stream.paginateEffect(0, (offset) =>
        db.query<T>(sql, offset, this.pageSize).pipe(
          Effect.map(results => [
            results,
            results.length === this.pageSize ? offset + this.pageSize : null
          ] as const),
          Effect.retry(
            Schedule.exponential("100ms").pipe(
              Schedule.whileInput((error: DatabaseError) => error.retryable),
              Schedule.recurs(3)
            )
          )
        )
      ).pipe(
        Stream.flatMap(Stream.fromIterable)
      )
    }).pipe(
      Stream.unwrap
    )
  }

  // Helper method for counting total records
  count(): Effect.Effect<number, DatabaseError, Database> {
    return Effect.gen(function* () {
      const db = yield* Database
      return yield* db.getCount(this.tableName)
    })
  }
}

// Usage example
const userQuery = new PaginatedQuery<User>("users", "WHERE active = 1", 50)

const processUsers = Effect.gen(function* () {
  const totalCount = yield* userQuery.count()
  console.log(`Processing ${totalCount} users`)
  
  const processedCount = yield* userQuery.pipe(
    Stream.tap(user => Effect.log(`Processing user: ${user.name}`)),
    Stream.mapEffect(user => processUserData(user)),
    Stream.runCount
  )
  
  console.log(`Successfully processed ${processedCount} users`)
})

// Layer providing database implementation
const DatabaseLive = Layer.succeed(Database, {
  query: <T>(sql: string, offset: number, limit: number) =>
    Effect.tryPromise({
      try: () => executeQuery<T>(sql, [limit, offset]),
      catch: (error) => DatabaseError(`Query failed: ${error}`, true)
    }),
  getCount: (table: string) =>
    Effect.tryPromise({
      try: () => executeCountQuery(table),
      catch: (error) => DatabaseError(`Count query failed: ${error}`, true)
    })
})

const program = processUsers.pipe(
  Effect.provide(DatabaseLive)
)
```

### Example 2: Real-Time Event Aggregation

A Streamable class that aggregates real-time events from multiple sources with buffering and batch processing.

```typescript
import { Stream, Effect, Streamable, Queue, Ref, Schedule } from "effect"

interface EventSource {
  readonly subscribe: <T>(topic: string, handler: (event: T) => void) => Effect.Effect<() => void>
}

interface AggregatedEvent<T> {
  readonly events: T[]
  readonly timestamp: number
  readonly source: string
}

class EventAggregator<T> extends Streamable.Class<AggregatedEvent<T>, never, EventSource> {
  constructor(
    private topics: string[],
    private bufferSize: number = 100,
    private flushInterval: string = "5 seconds"
  ) {
    super()
  }

  toStream(): Stream.Stream<AggregatedEvent<T>, never, EventSource> {
    return Effect.gen(function* () {
      const eventSource = yield* EventSource
      const queue = yield* Queue.bounded<{ event: T; topic: string; timestamp: number }>(this.bufferSize * 2)
      const buffer = yield* Ref.make<Map<string, T[]>>(new Map())
      
      // Subscribe to all topics
      const unsubscribers = yield* Effect.all(
        this.topics.map(topic =>
          eventSource.subscribe<T>(topic, (event) => {
            Queue.offer(queue, { event, topic, timestamp: Date.now() }).pipe(
              Effect.runFork
            )
          })
        )
      )
      
      // Cleanup function
      const cleanup = Effect.all(unsubscribers).pipe(
        Effect.flatMap(cleanupFns => Effect.all(cleanupFns.map(fn => Effect.sync(fn))))
      )
      
      // Event processing stream
      const eventStream = Stream.fromQueue(queue).pipe(
        Stream.groupedWithin(this.bufferSize, this.flushInterval),
        Stream.mapEffect(chunk => 
          Effect.gen(function* () {
            const currentBuffer = yield* Ref.get(buffer)
            const newBuffer = new Map(currentBuffer)
            
            // Group events by topic
            chunk.forEach(({ event, topic }) => {
              const topicEvents = newBuffer.get(topic) || []
              topicEvents.push(event)
              newBuffer.set(topic, topicEvents)
            })
            
            // Create aggregated events and reset buffer
            const aggregated: AggregatedEvent<T>[] = Array.from(newBuffer.entries()).map(
              ([topic, events]) => ({
                events,
                timestamp: Date.now(),
                source: topic
              })
            )
            
            yield* Ref.set(buffer, new Map())
            return aggregated
          })
        ),
        Stream.flatMap(Stream.fromIterable),
        Stream.ensuring(cleanup)
      )
      
      return eventStream
    }).pipe(
      Stream.unwrapScoped
    )
  }
}

// Usage with multiple event types
interface UserEvent {
  readonly type: "login" | "logout" | "purchase"
  readonly userId: string
  readonly metadata: Record<string, unknown>
}

const eventAggregator = new EventAggregator<UserEvent>(
  ["user.login", "user.logout", "user.purchase"],
  50,
  "3 seconds"
)

const processAggregatedEvents = eventAggregator.pipe(
  Stream.tap(aggregated => 
    Effect.log(`Received ${aggregated.events.length} events from ${aggregated.source}`)
  ),
  Stream.mapEffect(aggregated => processEventBatch(aggregated)),
  Stream.runDrain
)
```

### Example 3: File Processing with Progress Tracking

A Streamable implementation for processing large files with progress tracking and memory-efficient streaming.

```typescript
import { Stream, Effect, Streamable, Ref } from "effect"
import * as NodeFS from "@effect/platform-node/NodeFileSystem"

interface ProcessingProgress {
  readonly totalBytes: number
  readonly processedBytes: number
  readonly currentFile: string
  readonly percentage: number
}

interface FileProcessor {
  readonly processChunk: (chunk: Uint8Array, fileName: string) => Effect.Effect<ProcessedChunk>
}

interface ProcessedChunk {
  readonly fileName: string
  readonly data: Uint8Array
  readonly metadata: Record<string, unknown>
}

class FileStreamProcessor extends Streamable.Class<ProcessedChunk, Error, FileProcessor | NodeFS.NodeFileSystem> {
  constructor(
    private filePaths: string[],
    private chunkSize: number = 8192,
    private onProgress?: (progress: ProcessingProgress) => Effect.Effect<void>
  ) {
    super()
  }

  toStream(): Stream.Stream<ProcessedChunk, Error, FileProcessor | NodeFS.NodeFileSystem> {
    return Effect.gen(function* () {
      const fs = yield* NodeFS.NodeFileSystem
      const processor = yield* FileProcessor
      
      // Calculate total size for progress tracking
      const fileSizes = yield* Effect.all(
        this.filePaths.map(path =>
          fs.stat(path).pipe(
            Effect.map(stats => ({ path, size: stats.size })),
            Effect.catchAll(() => Effect.succeed({ path, size: 0 }))
          )
        )
      )
      
      const totalBytes = fileSizes.reduce((sum, { size }) => sum + size, 0)
      const processedBytes = yield* Ref.make(0)
      
      const fileStreams = this.filePaths.map(filePath => {
        const fileSize = fileSizes.find(f => f.path === filePath)?.size || 0
        
        return Stream.fromReadableStream(
          () => fs.stream(filePath),
          error => new Error(`Failed to read file ${filePath}: ${error}`)
        ).pipe(
          Stream.rechunk(this.chunkSize),
          Stream.mapEffect(chunk =>
            Effect.gen(function* () {
              // Update progress
              const currentProcessed = yield* Ref.updateAndGet(processedBytes, 
                prev => prev + chunk.length
              )
              
              if (this.onProgress) {
                yield* this.onProgress({
                  totalBytes,
                  processedBytes: currentProcessed,
                  currentFile: filePath,
                  percentage: Math.round((currentProcessed / totalBytes) * 100)
                })
              }
              
              // Process the chunk
              return yield* processor.processChunk(chunk, filePath)
            })
          )
        )
      })
      
      return Stream.mergeAll(fileStreams, { concurrency: 3 })
    }).pipe(
      Stream.unwrap
    )
  }
}

// Usage example with progress tracking
const fileProcessor = new FileStreamProcessor(
  ["./data/file1.bin", "./data/file2.bin", "./data/file3.bin"],
  16384,
  (progress) => Effect.log(
    `Processing: ${progress.percentage}% (${progress.processedBytes}/${progress.totalBytes} bytes) - ${progress.currentFile}`
  )
)

const processFiles = fileProcessor.pipe(
  Stream.tap(chunk => Effect.log(`Processed chunk from ${chunk.fileName}: ${chunk.data.length} bytes`)),
  Stream.mapEffect(chunk => saveProcessedChunk(chunk)),
  Stream.runCollect
)

// Provide implementations
const FileProcessorLive = Layer.succeed(FileProcessor, {
  processChunk: (chunk: Uint8Array, fileName: string) =>
    Effect.succeed({
      fileName,
      data: chunk,
      metadata: {
        processedAt: new Date().toISOString(),
        originalSize: chunk.length,
        checksum: calculateChecksum(chunk)
      }
    })
})

const program = processFiles.pipe(
  Effect.provide(FileProcessorLive),
  Effect.provide(NodeFS.layer)
)
```

## Advanced Features Deep Dive

### Advanced Feature 1: Generic Type Parameters and Constraints

Streamable classes can leverage TypeScript's advanced type system to create flexible, type-safe streaming abstractions.

```typescript
import { Stream, Effect, Streamable, Schema } from "effect"

// Generic Streamable with schema validation
class ValidatedStream<I, A, E = never, R = never> extends Streamable.Class<A, E | Schema.ParseError, R> {
  constructor(
    private source: Stream.Stream<I, E, R>,
    private schema: Schema.Schema<A, I>
  ) {
    super()
  }

  toStream(): Stream.Stream<A, E | Schema.ParseError, R> {
    return this.source.pipe(
      Stream.mapEffect(input =>
        Schema.decodeUnknown(this.schema)(input)
      )
    )
  }
}

// Usage with schema validation
const UserSchema = Schema.Struct({
  id: Schema.Number,
  name: Schema.String,
  email: Schema.String.pipe(Schema.filter(email => email.includes('@')))
})

const rawDataStream = Stream.make(
  { id: 1, name: "Alice", email: "alice@example.com" },
  { id: "invalid", name: "Bob", email: "invalid-email" },
  { id: 3, name: "Charlie", email: "charlie@example.com" }
)

const validatedUsers = new ValidatedStream(rawDataStream, UserSchema)

const processedUsers = validatedUsers.pipe(
  Stream.catchTag("ParseError", error => 
    Stream.fromEffect(Effect.log(`Validation failed: ${error.message}`))
      .pipe(Stream.drain)
  ),
  Stream.runCollect
)
```

### Advanced Feature 2: Composable Stream Transformations

Create Streamable classes that can be composed and chained for complex data processing pipelines.

```typescript
abstract class StreamTransformer<A, B, E = never, R = never> extends Streamable.Class<B, E, R> {
  constructor(protected source: Stream.Stream<A, E, R>) {
    super()
  }
  
  abstract transform(stream: Stream.Stream<A, E, R>): Stream.Stream<B, E, R>
  
  toStream(): Stream.Stream<B, E, R> {
    return this.transform(this.source)
  }
  
  // Chainable transformation method
  pipe<C>(transformer: (stream: Stream.Stream<B, E, R>) => StreamTransformer<B, C, E, R>): StreamTransformer<A, C, E, R> {
    const self = this
    return new (class extends StreamTransformer<A, C, E, R> {
      transform(stream: Stream.Stream<A, E, R>): Stream.Stream<C, E, R> {
        return transformer(self.transform(stream)).toStream()
      }
    })(this.source)
  }
}

// Specific transformer implementations
class DeduplicationTransformer<A> extends StreamTransformer<A, A> {
  constructor(
    source: Stream.Stream<A>,
    private keyExtractor: (item: A) => string
  ) {
    super(source)
  }
  
  transform(stream: Stream.Stream<A>): Stream.Stream<A> {
    return Effect.gen(function* () {
      const seen = yield* Ref.make(new Set<string>())
      
      return stream.pipe(
        Stream.filterEffect(item => {
          const key = this.keyExtractor(item)
          return Ref.modify(seen, seenSet => [
            !seenSet.has(key),
            new Set(seenSet).add(key)
          ])
        })
      )
    }).pipe(
      Stream.unwrap
    )
  }
}

class BatchingTransformer<A> extends StreamTransformer<A, A[]> {
  constructor(
    source: Stream.Stream<A>,
    private batchSize: number,
    private maxWaitTime: string = "1 second"
  ) {
    super(source)
  }
  
  transform(stream: Stream.Stream<A>): Stream.Stream<A[]> {
    return stream.pipe(
      Stream.groupedWithin(this.batchSize, this.maxWaitTime),
      Stream.map(chunk => chunk.toArray())
    )
  }
}

// Usage - chaining transformations
const dataStream = Stream.make(1, 2, 2, 3, 4, 4, 5, 6, 7, 8, 9, 10)

const processor = new DeduplicationTransformer(dataStream, item => item.toString())
  .pipe(source => new BatchingTransformer(source, 3, "500ms"))

const result = yield* Stream.runCollect(processor)
// Result: [[1, 2, 3], [4, 5, 6], [7, 8, 9], [10]]
```

### Advanced Feature 3: Resource Management and Cleanup

Implement proper resource management in Streamable classes using Effect's resource management capabilities.

```typescript
import { Stream, Effect, Streamable, Scope } from "effect"

interface ConnectionPool {
  readonly acquire: Effect.Effect<Connection>
  readonly release: (connection: Connection) => Effect.Effect<void>
}

interface Connection {
  readonly execute: <T>(query: string) => Effect.Effect<T[]>
  readonly close: Effect.Effect<void>
}

class ManagedQueryStream<T> extends Streamable.Class<T, Error, ConnectionPool> {
  constructor(
    private queries: string[],
    private concurrency: number = 3
  ) {
    super()
  }

  toStream(): Stream.Stream<T, Error, ConnectionPool> {
    return Stream.contextWithStream((pool: ConnectionPool) =>
      Stream.fromIterable(this.queries).pipe(
        Stream.mapEffect(query =>
          Effect.acquireUseRelease(
            pool.acquire,
            connection => connection.execute<T>(query),
            connection => pool.release(connection)
          )
        ),
        Stream.flatMap(Stream.fromIterable)
      )
    )
  }

  // Scoped version that manages the entire stream lifecycle
  toScopedStream(): Effect.Effect<Stream.Stream<T, Error>, Error, ConnectionPool | Scope.Scope> {
    return Effect.gen(function* () {
      const pool = yield* ConnectionPool
      const connections = yield* Effect.all(
        Array.from({ length: this.concurrency }, () =>
          Effect.acquireRelease(
            pool.acquire,
            connection => pool.release(connection)
          )
        )
      )

      const connectionQueue = yield* Queue.bounded<Connection>(this.concurrency)
      yield* Effect.all(connections.map(conn => Queue.offer(connectionQueue, conn)))

      return Stream.fromIterable(this.queries).pipe(
        Stream.mapEffect(query =>
          Effect.gen(function* () {
            const connection = yield* Queue.take(connectionQueue)
            const results = yield* connection.execute<T>(query)
            yield* Queue.offer(connectionQueue, connection)
            return results
          })
        ),
        Stream.flatMap(Stream.fromIterable)
      )
    })
  }
}

// Usage with proper resource management
const queryStream = new ManagedQueryStream<User>([
  "SELECT * FROM users WHERE department = 'engineering'",
  "SELECT * FROM users WHERE department = 'marketing'",
  "SELECT * FROM users WHERE department = 'sales'"
], 2)

const processQueries = queryStream.pipe(
  Stream.tap(user => Effect.log(`Processing user: ${user.name}`)),
  Stream.runCollect
)

// Alternative: using scoped version
const processScopedQueries = Effect.gen(function* () {
  const scopedStream = yield* queryStream.toScopedStream()
  return yield* Stream.runCollect(scopedStream)
}).pipe(
  Effect.scoped
)
```

## Practical Patterns & Best Practices

### Pattern 1: Configuration-Driven Streams

Create flexible Streamable classes that can be configured through dependency injection and configuration objects.

```typescript
interface StreamConfig {
  readonly batchSize: number
  readonly retryAttempts: number
  readonly timeout: string
  readonly concurrency: number
}

const StreamConfig = Context.GenericTag<StreamConfig>("StreamConfig")

class ConfigurableDataProcessor<T, R> extends Streamable.Class<ProcessedItem<T>, Error, StreamConfig | R> {
  constructor(
    private dataSource: Stream.Stream<T, Error, R>,
    private processor: (item: T) => Effect.Effect<ProcessedItem<T>, Error>
  ) {
    super()
  }

  toStream(): Stream.Stream<ProcessedItem<T>, Error, StreamConfig | R> {
    return Stream.contextWithStream((config: StreamConfig) =>
      this.dataSource.pipe(
        Stream.grouped(config.batchSize),
        Stream.mapEffect(batch =>
          Effect.all(
            batch.map(item =>
              this.processor(item).pipe(
                Effect.retry(Schedule.recurs(config.retryAttempts)),
                Effect.timeout(config.timeout)
              )
            ),
            { concurrency: config.concurrency }
          )
        ),
        Stream.flatMap(Stream.fromIterable)
      )
    )
  }
}

// Configuration layers
const DevConfig = Layer.succeed(StreamConfig, {
  batchSize: 10,
  retryAttempts: 2,
  timeout: "5 seconds",
  concurrency: 2
})

const ProdConfig = Layer.succeed(StreamConfig, {
  batchSize: 100,
  retryAttempts: 5,
  timeout: "30 seconds",
  concurrency: 10
})

// Usage
const processor = new ConfigurableDataProcessor(
  dataStream,
  item => processItem(item)
)

const devProgram = processor.pipe(
  Stream.runCollect,
  Effect.provide(DevConfig)
)

const prodProgram = processor.pipe(
  Stream.runCollect,
  Effect.provide(ProdConfig)
)
```

### Pattern 2: Monitoring and Observability

Implement comprehensive monitoring and observability features in your Streamable classes.

```typescript
interface Metrics {
  readonly incrementCounter: (name: string, tags?: Record<string, string>) => Effect.Effect<void>
  readonly recordGauge: (name: string, value: number, tags?: Record<string, string>) => Effect.Effect<void>
  readonly recordHistogram: (name: string, value: number, tags?: Record<string, string>) => Effect.Effect<void>
}

const Metrics = Context.GenericTag<Metrics>("Metrics")

class ObservableStream<T> extends Streamable.Class<T, Error, Metrics> {
  constructor(
    private source: Stream.Stream<T, Error>,
    private name: string
  ) {
    super()
  }

  toStream(): Stream.Stream<T, Error, Metrics> {
    return Effect.gen(function* () {
      const metrics = yield* Metrics
      const startTime = yield* Clock.currentTimeMillis
      
      yield* metrics.incrementCounter("stream.started", { name: this.name })
      
      return this.source.pipe(
        Stream.tap(() => metrics.incrementCounter("stream.item.processed", { name: this.name })),
        Stream.tapError(error => 
          metrics.incrementCounter("stream.error", { 
            name: this.name, 
            error: error.constructor.name 
          })
        ),
        Stream.onDone(() =>
          Effect.gen(function* () {
            const endTime = yield* Clock.currentTimeMillis
            const duration = endTime - startTime
            yield* metrics.recordHistogram("stream.duration", duration, { name: this.name })
            yield* metrics.incrementCounter("stream.completed", { name: this.name })
          })
        )
      )
    }).pipe(
      Stream.unwrap
    )
  }

  // Helper method for creating monitored transformations
  withMonitoring<B>(
    name: string,
    transform: (stream: Stream.Stream<T, Error>) => Stream.Stream<B, Error>
  ): ObservableStream<B> {
    return new ObservableStream(transform(this.toStream()), `${this.name}.${name}`)
  }
}

// Usage with metrics
const monitoredStream = new ObservableStream(dataStream, "user-processing")
  .withMonitoring("filtered", stream => stream.pipe(Stream.filter(isValid)))
  .withMonitoring("transformed", stream => stream.pipe(Stream.map(transform)))

const program = monitoredStream.pipe(
  Stream.runDrain,
  Effect.provide(MetricsLive)
)
```

### Pattern 3: Error Recovery and Circuit Breaking

Implement sophisticated error handling and circuit breaking patterns in Streamable classes.

```typescript
interface CircuitBreakerState {
  readonly failures: number
  readonly lastFailure: number
  readonly state: "closed" | "open" | "half-open"
}

class ResilientStream<T, E> extends Streamable.Class<T, E> {
  private circuitBreaker: Ref.Ref<CircuitBreakerState>

  constructor(
    private source: Stream.Stream<T, E>,
    private failureThreshold: number = 5,
    private recoveryTimeout: number = 60000,
    private fallback?: Stream.Stream<T, E>
  ) {
    super()
    this.circuitBreaker = Ref.unsafeMake({
      failures: 0,
      lastFailure: 0,
      state: "closed" as const
    })
  }

  toStream(): Stream.Stream<T, E> {
    return Stream.repeatEffectOption(
      this.executeWithCircuitBreaker()
    ).pipe(
      Stream.flatMap(Stream.fromIterable)
    )
  }

  private executeWithCircuitBreaker(): Effect.Effect<T[], Option.Option<never>, E> {
    return Effect.gen(() => {
      const state = yield* Ref.get(this.circuitBreaker)
      const now = Date.now()

      // Check if circuit should transition from open to half-open
      if (state.state === "open" && now - state.lastFailure > this.recoveryTimeout) {
        yield* Ref.update(this.circuitBreaker, s => ({ ...s, state: "half-open" }))
      }

      // If circuit is open, use fallback or fail
      if (state.state === "open") {
        if (this.fallback) {
          return yield* Stream.take(this.fallback, 10).pipe(
            Stream.runCollect,
            Effect.map(chunk => chunk.toArray())
          )
        } else {
          return yield* Effect.fail(Option.none())
        }
      }

      // Execute the stream
      return yield* this.source.pipe(
        Stream.take(10),
        Stream.runCollect,
        Effect.map(chunk => chunk.toArray()),
        Effect.tapDefect(() => this.recordFailure()),
        Effect.tapError(() => this.recordFailure()),
        Effect.tap(() => this.recordSuccess())
      )
    }).pipe(
      Effect.catchAllDefect(() => Effect.fail(Option.none())),
      Effect.catchAll(() => Effect.fail(Option.none()))
    )
  }

  private recordFailure(): Effect.Effect<void> {
    return Ref.update(this.circuitBreaker, state => {
      const newFailures = state.failures + 1
      return {
        failures: newFailures,
        lastFailure: Date.now(),
        state: newFailures >= this.failureThreshold ? "open" : state.state
      }
    })
  }

  private recordSuccess(): Effect.Effect<void> {
    return Ref.update(this.circuitBreaker, state => ({
      failures: 0,
      lastFailure: state.lastFailure,
      state: "closed"
    }))
  }
}

// Usage with circuit breaking
const fallbackData = Stream.make(
  { id: -1, name: "Fallback User 1" },
  { id: -2, name: "Fallback User 2" }
)

const resilientUserStream = new ResilientStream(
  unreliableUserStream,
  3, // failure threshold
  30000, // 30 second recovery timeout
  fallbackData
)

const processUsers = resilientUserStream.pipe(
  Stream.tap(user => Effect.log(`Processing: ${user.name}`)),
  Stream.runDrain
)
```

## Integration Examples

### Integration with HTTP APIs and Real-time Data

Integrate Streamable with HTTP clients and real-time data sources for comprehensive data processing pipelines.

```typescript
import { HttpClient, HttpClientRequest } from "@effect/platform"

interface ApiConfig {
  readonly baseUrl: string
  readonly apiKey: string
  readonly rateLimit: number
}

const ApiConfig = Context.GenericTag<ApiConfig>("ApiConfig")

class PaginatedApiStream<T> extends Streamable.Class<T, HttpClientError.HttpClientError, HttpClient.HttpClient | ApiConfig> {
  constructor(
    private endpoint: string,
    private pageSize: number = 100,
    private maxPages?: number
  ) {
    super()
  }

  toStream(): Stream.Stream<T, HttpClientError.HttpClientError, HttpClient.HttpClient | ApiConfig> {
    return Stream.contextWithStream(({ httpClient, config }: { httpClient: HttpClient.HttpClient, config: ApiConfig }) =>
      Stream.paginateEffect(1, (page) =>
        Effect.gen(function* () {
          if (this.maxPages && page > this.maxPages) {
            return [[], null] as const
          }

          const request = HttpClientRequest.get(`${config.baseUrl}${this.endpoint}`).pipe(
            HttpClientRequest.setUrlParams({
              page: page.toString(),
              limit: this.pageSize.toString()
            }),
            HttpClientRequest.setHeader("Authorization", `Bearer ${config.apiKey}`)
          )

          const response = yield* httpClient.execute(request).pipe(
            Effect.flatMap(response => response.json),
            Effect.retry(Schedule.exponential("1 second").pipe(Schedule.recurs(3)))
          )

          const data = response.data as T[]
          const hasNextPage = response.hasMore || data.length === this.pageSize

          // Rate limiting
          yield* Effect.sleep(`${1000 / config.rateLimit}ms`)

          return [data, hasNextPage ? page + 1 : null] as const
        })
      ).pipe(
        Stream.flatMap(Stream.fromIterable)
      )
    )
  }
}

// WebSocket integration
class WebSocketEventStream<T> extends Streamable.Class<T, Error> {
  constructor(
    private url: string,
    private messageHandler: (data: unknown) => T | null
  ) {
    super()
  }

  toStream(): Stream.Stream<T, Error> {
    return Stream.async<T, Error>((emit) =>
      Effect.gen(function* () {
        const ws = new WebSocket(this.url)
        
        ws.onopen = () => {
          console.log(`Connected to ${this.url}`)
        }
        
        ws.onmessage = (event) => {
          try {
            const data = JSON.parse(event.data)
            const processed = this.messageHandler(data)
            if (processed !== null) {
              emit.single(processed)
            }
          } catch (error) {
            emit.fail(new Error(`Failed to process message: ${error}`))
          }
        }
        
        ws.onerror = (error) => {
          emit.fail(new Error(`WebSocket error: ${error}`))
        }
        
        ws.onclose = () => {
          emit.end()
        }
        
        return Effect.sync(() => {
          ws.close()
        })
      })
    )
  }
}

// Combined usage - API + WebSocket
const apiStream = new PaginatedApiStream<User>("/users", 50, 10)
const wsStream = new WebSocketEventStream<UserEvent>(
  "wss://api.example.com/events", 
  (data: any) => data.type === "user_event" ? data : null
)

const combinedStream = Stream.merge(
  apiStream.pipe(
    Stream.map(user => ({ type: "user_data" as const, user }))
  ),
  wsStream.pipe(
    Stream.map(event => ({ type: "user_event" as const, event }))
  )
)

const program = combinedStream.pipe(
  Stream.tap(data => 
    data.type === "user_data" 
      ? Effect.log(`User data: ${data.user.name}`)
      : Effect.log(`User event: ${data.event.type}`)
  ),
  Stream.runDrain,
  Effect.provide(Layer.mergeAll(HttpClient.layer, ApiConfigLive))
)
```

### Integration with Database and Queue Systems

Demonstrate integration with databases and message queue systems for robust data processing.

```typescript
import { SqlClient } from "@effect/sql"
import { Queue } from "effect"

// Database cursor with connection pooling
class SqlCursorStream<T> extends Streamable.Class<T, SqlError.SqlError, SqlClient.SqlClient> {
  constructor(
    private query: string,
    private params: ReadonlyArray<unknown> = [],
    private batchSize: number = 1000
  ) {
    super()
  }

  toStream(): Stream.Stream<T, SqlError.SqlError, SqlClient.SqlClient> {
    return Stream.contextWithStream((sql: SqlClient.SqlClient) =>
      Stream.paginateEffect(0, (offset) =>
        Effect.gen(function* () {
          const paginatedQuery = `${this.query} LIMIT ${this.batchSize} OFFSET ${offset}`
          const rows = yield* sql.unsafe.execute(paginatedQuery, this.params)
          
          const results = rows as T[]
          const hasMore = results.length === this.batchSize
          
          return [results, hasMore ? offset + this.batchSize : null] as const
        })
      ).pipe(
        Stream.flatMap(Stream.fromIterable)
      )
    )
  }
}

// Message queue integration
interface MessageQueue {
  readonly subscribe: <T>(topic: string) => Effect.Effect<Queue.Queue<T>>
  readonly publish: <T>(topic: string, message: T) => Effect.Effect<void>
}

const MessageQueue = Context.GenericTag<MessageQueue>("MessageQueue")

class QueueConsumerStream<T> extends Streamable.Class<T, never, MessageQueue> {
  constructor(
    private topic: string,
    private maxConcurrency: number = 10
  ) {
    super()
  }

  toStream(): Stream.Stream<T, never, MessageQueue> {
    return Stream.contextWithStream((messageQueue: MessageQueue) =>
      Stream.fromEffect(messageQueue.subscribe<T>(this.topic)).pipe(
        Stream.flatMap(queue => Stream.fromQueue(queue))
      )
    )
  }

  // Process messages with acknowledgment
  withProcessing<B>(
    processor: (message: T) => Effect.Effect<B>,
    onSuccess?: (message: T, result: B) => Effect.Effect<void>,
    onError?: (message: T, error: unknown) => Effect.Effect<void>
  ): Stream.Stream<B, never, MessageQueue> {
    return this.toStream().pipe(
      Stream.mapEffect(message =>
        processor(message).pipe(
          Effect.tap(result => onSuccess?.(message, result) ?? Effect.unit),
          Effect.catchAll(error => 
            Effect.gen(function* () {
              yield* onError?.(message, error) ?? Effect.unit
              return yield* Effect.fail(error)
            })
          )
        )
      )
    )
  }
}

// Combined database + queue processing
const processUserUpdates = Effect.gen(function* () {
  const userCursor = new SqlCursorStream<User>(
    "SELECT * FROM users WHERE updated_at > ?",
    [new Date(Date.now() - 24 * 60 * 60 * 1000)], // Last 24 hours
    500
  )

  const queueConsumer = new QueueConsumerStream<UserUpdateMessage>("user.updates")

  // Process database updates
  const dbUpdates = userCursor.pipe(
    Stream.tap(user => Effect.log(`Processing DB user: ${user.id}`)),
    Stream.runDrain
  )

  // Process queue messages
  const queueUpdates = queueConsumer.withProcessing(
    message => processUserUpdateMessage(message),
    (message, result) => Effect.log(`Processed message ${message.id}: ${result}`),
    (message, error) => Effect.logError(`Failed to process message ${message.id}: ${error}`)
  ).pipe(
    Stream.runDrain
  )

  // Run both concurrently
  yield* Effect.all([dbUpdates, queueUpdates], { concurrency: "unbounded" })
})

// Layer implementations
const SqlClientLive = Layer.succeed(SqlClient.SqlClient, /* implementation */)
const MessageQueueLive = Layer.succeed(MessageQueue, /* implementation */)

const program = processUserUpdates.pipe(
  Effect.provide(Layer.mergeAll(SqlClientLive, MessageQueueLive))
)
```

### Testing Strategies

Implement comprehensive testing strategies for Streamable classes with mocking and property-based testing.

```typescript
import { Effect, TestContext, TestClock, Ref } from "effect"

// Mock data source for testing
class MockDataSource<T> extends Streamable.Class<T, Error> {
  constructor(
    private data: T[],
    private delay: string = "100ms",
    private errorRate: number = 0
  ) {
    super()
  }

  toStream(): Stream.Stream<T, Error> {
    return Stream.fromIterable(this.data).pipe(
      Stream.mapEffect(item =>
        Effect.gen(function* () {
          yield* TestClock.adjust(this.delay)
          
          if (Math.random() < this.errorRate) {
            return yield* Effect.fail(new Error(`Simulated error for item: ${JSON.stringify(item)}`))
          }
          
          return item
        })
      )
    )
  }
}

// Test utilities
const createTestStream = <T>(data: T[], options: { delay?: string; errorRate?: number } = {}) =>
  new MockDataSource(data, options.delay || "100ms", options.errorRate || 0)

// Property-based testing helper
const testStreamProperty = <T, B>(
  streamFactory: (data: T[]) => Streamable.Class<B>,
  property: (input: T[], output: B[]) => boolean,
  generator: Effect.Effect<T[]>
) =>
  Effect.gen(function* () {
    const testData = yield* generator
    const stream = streamFactory(testData)
    const output = yield* Stream.runCollect(stream)
    
    const result = property(testData, output.toArray())
    if (!result) {
      return yield* Effect.fail(new Error(`Property violated for input: ${JSON.stringify(testData)}`))
    }
    
    return result
  })

// Example tests
const testSuite = Effect.gen(function* () {
  // Test 1: Basic functionality
  yield* Effect.log("Testing basic stream functionality...")
  const basicStream = createTestStream([1, 2, 3, 4, 5])
  const basicResult = yield* Stream.runCollect(basicStream)
  
  if (basicResult.toArray().length !== 5) {
    return yield* Effect.fail(new Error("Basic stream test failed"))
  }

  // Test 2: Error handling
  yield* Effect.log("Testing error handling...")
  const errorStream = createTestStream([1, 2, 3], { errorRate: 0.5 })
  const errorResult = yield* Stream.runCollect(errorStream).pipe(
    Effect.catchAll(() => Effect.succeed(Chunk.empty<number>()))
  )
  
  yield* Effect.log(`Error stream produced ${errorResult.length} items`)

  // Test 3: Property-based testing
  yield* Effect.log("Running property-based tests...")
  yield* testStreamProperty(
    (data: number[]) => createTestStream(data),
    (input, output) => output.length <= input.length, // Stream should not produce more items than input
    Effect.succeed([1, 2, 3, 4, 5, 6, 7, 8, 9, 10])
  )

  // Test 4: Performance testing
  yield* Effect.log("Performance testing...")
  const largeDataset = Array.from({ length: 10000 }, (_, i) => i)
  const performanceStream = createTestStream(largeDataset, { delay: "1ms" })
  
  const startTime = yield* TestClock.currentTimeMillis
  yield* Stream.runDrain(performanceStream)
  const endTime = yield* TestClock.currentTimeMillis
  
  yield* Effect.log(`Processed ${largeDataset.length} items in ${endTime - startTime}ms`)

  yield* Effect.log("All tests passed!")
}).pipe(
  Effect.provide(TestContext.TestContext)
)
```

## Conclusion

Streamable provides powerful abstractions for creating custom, reusable stream types that integrate seamlessly with Effect's ecosystem. By encapsulating streaming logic in classes, you achieve better code organization, improved testability, and enhanced composability.

Key benefits:
- **Type Safety**: Full TypeScript integration with proper error and dependency typing
- **Composability**: Seamless integration with all Stream operations and Effect ecosystem
- **Reusability**: Encapsulated streaming logic that can be shared across applications
- **Resource Management**: Built-in support for proper resource cleanup and lifecycle management
- **Observability**: Easy integration with monitoring, logging, and metrics collection

Use Streamable when you need to create custom streaming abstractions that go beyond simple data transformation, especially for complex data sources like databases, APIs, message queues, or real-time event systems.