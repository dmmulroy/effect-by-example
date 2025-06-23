# Mailbox: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem Mailbox Solves

In traditional concurrent programming, managing actor-style communication requires complex coordination between producers and consumers. You often end up with brittle patterns that mix channels, queues, and manual state management:

```typescript
// Traditional approach - problematic actor pattern
class TraditionalActor {
  private queue: Array<any> = []
  private processing = false
  private done = false

  async send(message: any) {
    if (this.done) return false
    this.queue.push(message)
    this.processQueue()
    return true
  }

  private async processQueue() {
    if (this.processing) return
    this.processing = true
    while (this.queue.length > 0 && !this.done) {
      const message = this.queue.shift()
      await this.handleMessage(message)
    }
    this.processing = false
  }

  stop() {
    this.done = true
    this.queue = []
  }
}
```

This approach leads to:
- **Race Conditions** - Multiple producers can corrupt the internal state
- **Memory Leaks** - No proper cleanup when actors fail or finish
- **Error Handling** - No structured way to handle failures in message processing
- **Backpressure** - No built-in mechanism to handle slow consumers
- **Resource Management** - Manual lifecycle management is error-prone

### The Mailbox Solution

Mailbox provides a type-safe, structured approach to actor-style message passing with built-in error handling, backpressure, and lifecycle management:

```typescript
import { Effect, Mailbox, Fiber } from "effect"

// Clean, type-safe actor pattern
const createActor = <A, E>(handler: (message: A) => Effect.Effect<void, E>) =>
  Effect.gen(function* () {
    const mailbox = yield* Mailbox.make<A, E>({ capacity: 100 })
    
    const processor = yield* Effect.gen(function* () {
      while (true) {
        const message = yield* mailbox.take
        yield* handler(message)
      }
    }).pipe(
      Effect.catchAll(() => Effect.void),
      Effect.fork
    )
    
    return {
      send: (message: A) => mailbox.offer(message),
      stop: () => Effect.gen(function* () {
        yield* mailbox.end
        yield* processor.await
      })
    } as const
  })
```

### Key Concepts

**Mailbox<A, E>**: A queue that can be signaled to be done or failed, with type-safe message passing

**ReadonlyMailbox<A, E>**: A read-only view of a mailbox for consumers, preventing accidental mutations

**Backpressure Strategies**: Built-in handling for when capacity is exceeded ("suspend", "dropping", "sliding")

**Lifecycle Management**: Structured completion and error handling with proper resource cleanup

## Basic Usage Patterns

### Pattern 1: Simple Message Passing

```typescript
import { Effect, Mailbox } from "effect"

const basicUsage = Effect.gen(function* () {
  // Create a mailbox with default settings (unlimited capacity)
  const mailbox = yield* Mailbox.make<string>()
  
  // Send messages
  yield* mailbox.offer("Hello")
  yield* mailbox.offer("World")
  
  // Receive messages
  const [messages, done] = yield* mailbox.takeAll
  console.log(messages) // Chunk(["Hello", "World"])
  console.log(done)     // false
  
  // Signal completion
  yield* mailbox.end
  
  // Check final state
  const [final, isDone] = yield* mailbox
  console.log(isDone) // true
})
```

### Pattern 2: Bounded Mailbox with Backpressure

```typescript
const boundedMailbox = Effect.gen(function* () {
  // Create mailbox with capacity of 2 and suspend strategy
  const mailbox = yield* Mailbox.make<number>({ 
    capacity: 2, 
    strategy: "suspend" 
  })
  
  // First two offers succeed immediately
  console.log(yield* mailbox.offer(1)) // true
  console.log(yield* mailbox.offer(2)) // true
  
  // Third offer will suspend until space is available
  const offerFiber = yield* mailbox.offer(3).pipe(Effect.fork)
  
  // Take a message to make space
  const message = yield* mailbox.take
  console.log(message) // 1
  
  // Now the suspended offer completes
  console.log(yield* offerFiber.await) // Exit.succeed(true)
})
```

### Pattern 3: Error Handling and Completion

```typescript
const errorHandling = Effect.gen(function* () {
  const mailbox = yield* Mailbox.make<string, string>()
  
  // Send some messages
  yield* mailbox.offer("message1")
  yield* mailbox.offer("message2")
  
  // Fail the mailbox with an error
  yield* mailbox.fail("Something went wrong")
  
  // Messages can still be consumed
  const message1 = yield* mailbox.take
  console.log(message1) // "message1"
  
  // But eventually we'll get the error
  const result = yield* mailbox.take.pipe(Effect.either)
  console.log(result) // Either.left("Something went wrong")
})
```

## Real-World Examples

### Example 1: HTTP Request Actor with Rate Limiting

```typescript
import { Effect, Mailbox, Schedule, Duration, HttpClient } from "effect"

interface HttpRequest {
  readonly url: string
  readonly method: "GET" | "POST" | "PUT" | "DELETE"
  readonly body?: unknown
  readonly respond: (response: HttpClient.response.ClientResponse) => Effect.Effect<void>
}

const createHttpActor = (requestsPerSecond: number) =>
  Effect.gen(function* () {
    const httpClient = yield* HttpClient.HttpClient
    const mailbox = yield* Mailbox.make<HttpRequest>({ capacity: 1000 })
    
    // Process requests with rate limiting
    const processor = yield* Effect.gen(function* () {
      while (true) {
        const request = yield* mailbox.take
        
        const response = yield* HttpClient.request(request.url, {
          method: request.method,
          body: request.body ? HttpClient.bodyJson(request.body) : undefined
        }).pipe(
          Effect.retry(Schedule.exponential(Duration.millis(100)).pipe(
            Schedule.compose(Schedule.recurs(3))
          )),
          Effect.orDie
        )
        
        yield* request.respond(response)
        
        // Rate limiting delay
        yield* Effect.sleep(Duration.millis(1000 / requestsPerSecond))
      }
    }).pipe(
      Effect.catchAll(() => Effect.void),
      Effect.fork
    )
    
    return {
      request: (req: Omit<HttpRequest, 'respond'>) =>
        Effect.gen(function* () {
          const deferred = yield* Effect.Deferred.make<HttpClient.response.ClientResponse>()
          
          const success = yield* mailbox.offer({
            ...req,
            respond: (response) => deferred.succeed(response)
          })
          
          if (!success) {
            return yield* Effect.fail(new Error("Actor is shutting down"))
          }
          
          return yield* deferred.await
        }),
      
      shutdown: () => Effect.gen(function* () {
        yield* mailbox.end
        yield* processor.await
      })
    } as const
  })

// Usage
const httpActorExample = Effect.gen(function* () {
  const actor = yield* createHttpActor(10) // 10 requests per second
  
  // Make concurrent requests
  const responses = yield* Effect.all([
    actor.request({ url: "https://api.example.com/users", method: "GET" }),
    actor.request({ url: "https://api.example.com/posts", method: "GET" }),
    actor.request({ 
      url: "https://api.example.com/posts", 
      method: "POST",
      body: { title: "New Post", content: "Hello World" }
    })
  ], { concurrency: "unbounded" })
  
  console.log(`Processed ${responses.length} requests`)
  
  yield* actor.shutdown()
})
```

### Example 2: Event Sourcing with Mailbox

```typescript
import { Effect, Mailbox, Ref, Array as Arr } from "effect"

interface Event {
  readonly id: string
  readonly type: string
  readonly payload: unknown
  readonly timestamp: number
}

interface EventStore {
  readonly append: (event: Event) => Effect.Effect<void>
  readonly getEvents: (fromId?: string) => Effect.Effect<Array<Event>>
  readonly subscribe: () => Effect.Effect<Mailbox.ReadonlyMailbox<Event>>
  readonly shutdown: () => Effect.Effect<void>
}

const createEventStore = (): Effect.Effect<EventStore> =>
  Effect.gen(function* () {
    const events = yield* Ref.make<Array<Event>>([])
    const subscribers = yield* Ref.make<Set<Mailbox.Mailbox<Event>>>(new Set())
    
    const eventProcessor = yield* Mailbox.make<Event>({ capacity: 10000 })
    
    // Background processor for handling events
    const processor = yield* Effect.gen(function* () {
      while (true) {
        const event = yield* eventProcessor.take
        
        // Store the event
        yield* Ref.update(events, (prev) => [...prev, event])
        
        // Notify all subscribers
        const currentSubscribers = yield* Ref.get(subscribers)
        yield* Effect.forEach(currentSubscribers, (mailbox) =>
          mailbox.offer(event).pipe(
            Effect.catchAll(() => Effect.void) // Remove failed subscribers
          )
        )
      }
    }).pipe(
      Effect.catchAll(() => Effect.void),
      Effect.fork
    )
    
    return {
      append: (event) => eventProcessor.offer(event).pipe(
        Effect.flatMap((success) => 
          success 
            ? Effect.void 
            : Effect.fail(new Error("Event store is shutting down"))
        )
      ),
      
      getEvents: (fromId) => Effect.gen(function* () {
        const allEvents = yield* Ref.get(events)
        if (!fromId) return allEvents
        
        const startIndex = allEvents.findIndex(e => e.id === fromId)
        return startIndex >= 0 ? allEvents.slice(startIndex) : []
      }),
      
      subscribe: () => Effect.gen(function* () {
        const mailbox = yield* Mailbox.make<Event>({ capacity: 1000 })
        yield* Ref.update(subscribers, (set) => new Set([...set, mailbox]))
        
        // Return readonly view
        return mailbox as Mailbox.ReadonlyMailbox<Event>
      }),
      
      shutdown: () => Effect.gen(function* () {
        yield* eventProcessor.end
        yield* processor.await
        
        // Clean up all subscribers
        const currentSubscribers = yield* Ref.get(subscribers)
        yield* Effect.forEach(currentSubscribers, (mailbox) => mailbox.end)
      })
    } as const
  })

// Usage with event sourcing pattern
const eventSourcingExample = Effect.gen(function* () {
  const eventStore = yield* createEventStore()
  
  // Create a subscriber
  const subscriber = yield* eventStore.subscribe()
  
  // Start consuming events
  const consumer = yield* Effect.gen(function* () {
    yield* Effect.forEach(
      Mailbox.toStream(subscriber),
      (event) => Effect.sync(() => console.log("Received event:", event)),
      { concurrency: 1 }
    )
  }).pipe(Effect.fork)
  
  // Append some events
  yield* eventStore.append({
    id: "1",
    type: "user_created",
    payload: { name: "Alice", email: "alice@example.com" },
    timestamp: Date.now()
  })
  
  yield* eventStore.append({
    id: "2", 
    type: "user_updated",
    payload: { id: "1", name: "Alice Smith" },
    timestamp: Date.now()
  })
  
  // Wait a bit for processing
  yield* Effect.sleep(Duration.millis(100))
  
  // Get all events
  const allEvents = yield* eventStore.getEvents()
  console.log(`Total events: ${allEvents.length}`)
  
  yield* eventStore.shutdown()
  yield* consumer.await
})
```

### Example 3: WebSocket Connection Manager

```typescript
import { Effect, Mailbox, Ref, HashMap, Schedule, Duration } from "effect"

interface Connection {
  readonly id: string
  readonly send: (message: string) => Effect.Effect<boolean>
  readonly close: () => Effect.Effect<void>
}

interface Message {
  readonly connectionId: string
  readonly data: string
}

const createConnectionManager = () =>
  Effect.gen(function* () {
    const connections = yield* Ref.make<HashMap.HashMap<string, Connection>>(HashMap.empty())
    const incomingMessages = yield* Mailbox.make<Message>({ capacity: 10000 })
    const outgoingMessages = yield* Mailbox.make<Message>({ capacity: 10000 })
    
    // Message router - distributes outgoing messages to connections
    const messageRouter = yield* Effect.gen(function* () {
      while (true) {
        const message = yield* outgoingMessages.take
        const currentConnections = yield* Ref.get(connections)
        
        yield* HashMap.get(currentConnections, message.connectionId).pipe(
          Effect.fromOption,
          Effect.flatMap((connection) => connection.send(message.data)),
          Effect.catchAll(() => Effect.void) // Remove dead connections
        )
      }
    }).pipe(
      Effect.catchAll(() => Effect.void),
      Effect.fork
    )
    
    // Heartbeat system
    const heartbeat = yield* Effect.gen(function* () {
      while (true) {
        yield* Effect.sleep(Duration.seconds(30))
        
        const currentConnections = yield* Ref.get(connections)
        const connectionList = HashMap.values(currentConnections)
        
        yield* Effect.forEach(
          connectionList,
          (connection) => connection.send("ping").pipe(
            Effect.catchAll(() => 
              // Remove dead connection
              Ref.update(connections, HashMap.remove(connection.id))
            )
          ),
          { concurrency: 10 }
        )
      }
    }).pipe(
      Effect.catchAll(() => Effect.void),
      Effect.fork
    )
    
    return {
      addConnection: (connection: Connection) =>
        Ref.update(connections, HashMap.set(connection.id, connection)),
      
      removeConnection: (connectionId: string) =>
        Ref.update(connections, HashMap.remove(connectionId)),
      
      broadcast: (message: string) => Effect.gen(function* () {
        const currentConnections = yield* Ref.get(connections)
        const connectionIds = Array.from(HashMap.keys(currentConnections))
        
        yield* Effect.forEach(
          connectionIds,
          (id) => outgoingMessages.offer({ connectionId: id, data: message }),
          { concurrency: "unbounded" }
        )
      }),
      
      sendToConnection: (connectionId: string, message: string) =>
        outgoingMessages.offer({ connectionId, data: message }),
      
      getIncomingMessages: () => incomingMessages as Mailbox.ReadonlyMailbox<Message>,
      
      receiveMessage: (connectionId: string, data: string) =>
        incomingMessages.offer({ connectionId, data }),
      
      shutdown: () => Effect.gen(function* () {
        yield* incomingMessages.end
        yield* outgoingMessages.end
        yield* messageRouter.await
        yield* heartbeat.await
        
        // Close all connections
        const currentConnections = yield* Ref.get(connections)
        yield* Effect.forEach(
          HashMap.values(currentConnections),
          (connection) => connection.close(),
          { concurrency: 10 }
        )
      })
    } as const
  })

// Usage
const connectionManagerExample = Effect.gen(function* () {
  const manager = yield* createConnectionManager()
  
  // Mock connection
  const mockConnection: Connection = {
    id: "conn-1",
    send: (message) => Effect.succeed(console.log(`Sending: ${message}`) || true),
    close: () => Effect.sync(() => console.log("Connection closed"))
  }
  
  yield* manager.addConnection(mockConnection)
  
  // Process incoming messages
  const processor = yield* Effect.gen(function* () {
    const messages = manager.getIncomingMessages()
    yield* Effect.forEach(
      Mailbox.toStream(messages),
      (message) => Effect.sync(() => 
        console.log(`Received from ${message.connectionId}: ${message.data}`)
      )
    )
  }).pipe(Effect.fork)
  
  // Simulate some activity
  yield* manager.receiveMessage("conn-1", "Hello from client")
  yield* manager.sendToConnection("conn-1", "Hello from server")
  yield* manager.broadcast("Broadcast message")
  
  yield* Effect.sleep(Duration.millis(100))
  yield* manager.shutdown()
  yield* processor.await
})
```

## Advanced Features Deep Dive

### Feature 1: Backpressure Strategies

Mailbox provides three strategies for handling capacity overflow:

#### Basic Strategy Usage

```typescript
const backpressureStrategies = Effect.gen(function* () {
  // Suspend: Block producers when capacity is reached
  const suspendingMailbox = yield* Mailbox.make<number>({ 
    capacity: 2, 
    strategy: "suspend" 
  })
  
  // Dropping: Drop new messages when capacity is reached
  const droppingMailbox = yield* Mailbox.make<number>({ 
    capacity: 2, 
    strategy: "dropping" 
  })
  
  // Sliding: Drop oldest messages when capacity is reached
  const slidingMailbox = yield* Mailbox.make<number>({ 
    capacity: 2, 
    strategy: "sliding" 
  })
})
```

#### Real-World Strategy Example

```typescript
// Smart buffer for handling bursty data streams
const createSmartBuffer = <A>(
  strategy: "suspend" | "dropping" | "sliding",
  capacity: number
) => Effect.gen(function* () {
  const mailbox = yield* Mailbox.make<A>({ capacity, strategy })
  
  const processor = yield* Effect.gen(function* () {
    while (true) {
      // Process messages in batches for efficiency
      const [messages, done] = yield* mailbox.takeN(Math.min(capacity, 100))
      
      if (messages.length > 0) {
        yield* Effect.sync(() => console.log(`Processing batch of ${messages.length} items`))
        yield* Effect.sleep(Duration.millis(50)) // Simulate processing time
      }
      
      if (done) break
    }
  }).pipe(Effect.fork)
  
  return {
    add: (item: A) => mailbox.offer(item),
    addAll: (items: Iterable<A>) => mailbox.offerAll(items),
    size: () => mailbox.size,
    close: () => Effect.gen(function* () {
      yield* mailbox.end
      yield* processor.await
    })
  } as const
})
```

### Feature 2: Stream Integration

#### Converting Mailbox to Stream

```typescript
import { Stream, Mailbox } from "effect"

const mailboxToStreamExample = Effect.gen(function* () {
  const mailbox = yield* Mailbox.make<number>()
  
  // Convert mailbox to stream for powerful stream processing
  const stream = Mailbox.toStream(mailbox)
  
  // Process with stream operators
  const processedStream = stream.pipe(
    Stream.take(10),
    Stream.map(x => x * 2),
    Stream.filter(x => x > 5),
    Stream.grouped(3),
    Stream.tap(batch => Effect.sync(() => console.log("Batch:", batch)))
  )
  
  // Start processing
  const processor = yield* Stream.runDrain(processedStream).pipe(Effect.fork)
  
  // Send data
  yield* Effect.forEach(
    Array.range(1, 20),
    (n) => mailbox.offer(n),
    { concurrency: "unbounded" }
  )
  
  yield* mailbox.end
  yield* processor.await
})
```

#### Converting Stream to Mailbox

```typescript
const streamToMailboxExample = Effect.gen(function* () {
  // Create a stream of data
  const dataStream = Stream.range(1, 100).pipe(
    Stream.tap(n => Effect.sync(() => console.log(`Producing: ${n}`))),
    Stream.schedule(Schedule.fixed(Duration.millis(10)))
  )
  
  // Convert to mailbox with backpressure
  const mailbox = yield* Mailbox.fromStream(dataStream, {
    capacity: 10,
    strategy: "suspend"
  })
  
  // Consumer that processes slowly
  const consumer = yield* Effect.gen(function* () {
    while (true) {
      const item = yield* mailbox.take
      yield* Effect.sync(() => console.log(`Consuming: ${item}`))
      yield* Effect.sleep(Duration.millis(100)) // Slow consumer
    }
  }).pipe(
    Effect.catchAll(() => Effect.void),
    Effect.fork
  )
  
  yield* consumer.await
})
```

### Feature 3: Advanced Lifecycle Management

#### Graceful Shutdown with Cleanup

```typescript
const createManagedActor = <A, E>(
  handler: (message: A) => Effect.Effect<void, E>,
  capacity = 1000
) => Effect.gen(function* () {
  const mailbox = yield* Mailbox.make<A, E>({ capacity })
  const isShuttingDown = yield* Ref.make(false)
  const activeMessages = yield* Ref.make(0)
  
  const processor = yield* Effect.gen(function* () {
    while (true) {
      const message = yield* mailbox.take
      
      yield* Ref.update(activeMessages, n => n + 1)
      
      yield* handler(message).pipe(
        Effect.ensuring(Ref.update(activeMessages, n => n - 1)),
        Effect.catchAll(() => Effect.void)
      )
    }
  }).pipe(
    Effect.catchAll(() => Effect.void),
    Effect.fork
  )
  
  return {
    send: (message: A) => Effect.gen(function* () {
      const shuttingDown = yield* Ref.get(isShuttingDown)
      if (shuttingDown) {
        return yield* Effect.fail(new Error("Actor is shutting down"))
      }
      
      const offered = yield* mailbox.offer(message)
      if (!offered) {
        return yield* Effect.fail(new Error("Mailbox is full or closed"))
      }
    }),
    
    gracefulShutdown: (timeout: Duration.Duration) => Effect.gen(function* () {
      yield* Ref.set(isShuttingDown, true)
      yield* mailbox.end
      
      // Wait for active messages to complete with timeout
      yield* Effect.gen(function* () {
        while (true) {
          const active = yield* Ref.get(activeMessages)
          if (active === 0) break
          yield* Effect.sleep(Duration.millis(10))
        }
      }).pipe(
        Effect.timeoutTo({
          duration: timeout,
          onTimeout: () => Effect.sync(() => console.log("Timeout reached, forcing shutdown"))
        })
      )
      
      yield* processor.await
    }),
    
    forceShutdown: () => Effect.gen(function* () {
      yield* Ref.set(isShuttingDown, true)
      yield* mailbox.shutdown
      yield* processor.await
    })
  } as const
})
```

## Practical Patterns & Best Practices

### Pattern 1: Request-Response Pattern

```typescript
interface Request<T> {
  readonly id: string
  readonly data: T
  readonly respond: (response: string) => Effect.Effect<void>
}

const createRequestResponseActor = <T>() => Effect.gen(function* () {
  const mailbox = yield* Mailbox.make<Request<T>>({ capacity: 1000 })
  
  const processor = yield* Effect.gen(function* () {
    while (true) {
      const request = yield* mailbox.take
      
      // Simulate processing
      const response = `Processed: ${JSON.stringify(request.data)}`
      yield* request.respond(response)
    }
  }).pipe(
    Effect.catchAll(() => Effect.void),
    Effect.fork
  )
  
  return {
    request: <R>(data: R) => Effect.gen(function* () {
      const deferred = yield* Effect.Deferred.make<string>()
      const id = crypto.randomUUID()
      
      yield* mailbox.offer({
        id,
        data,
        respond: (response) => deferred.succeed(response)
      })
      
      return yield* deferred.await
    }),
    
    shutdown: () => Effect.gen(function* () {
      yield* mailbox.end
      yield* processor.await
    })
  } as const
})
```

### Pattern 2: Priority Message Handling

```typescript
interface PriorityMessage<A> {
  readonly priority: number
  readonly message: A
}

const createPriorityMailbox = <A>() => Effect.gen(function* () {
  const highPriority = yield* Mailbox.make<A>({ capacity: 100 })
  const normalPriority = yield* Mailbox.make<A>({ capacity: 1000 })
  const lowPriority = yield* Mailbox.make<A>({ capacity: 1000 })
  
  const getMailboxForPriority = (priority: number) => {
    if (priority >= 8) return highPriority
    if (priority >= 5) return normalPriority
    return lowPriority
  }
  
  return {
    offer: (message: PriorityMessage<A>) => {
      const mailbox = getMailboxForPriority(message.priority)
      return mailbox.offer(message.message)
    },
    
    // Take from highest priority mailbox first
    take: Effect.gen(function* () {
      // Try high priority first
      const highSize = yield* highPriority.size
      if (highSize.pipe(Option.getOrElse(() => 0)) > 0) {
        return yield* highPriority.take
      }
      
      // Then normal priority
      const normalSize = yield* normalPriority.size
      if (normalSize.pipe(Option.getOrElse(() => 0)) > 0) {
        return yield* normalPriority.take
      }
      
      // Finally low priority
      return yield* lowPriority.take
    }),
    
    shutdown: () => Effect.gen(function* () {
      yield* Effect.all([
        highPriority.end,
        normalPriority.end,
        lowPriority.end
      ])
    })
  } as const
})
```

### Pattern 3: Circuit Breaker with Mailbox

```typescript
interface CircuitBreakerState {
  readonly status: "closed" | "open" | "half-open"
  readonly failures: number
  readonly lastFailureTime: number
}

const createCircuitBreakerActor = <A, E>(
  handler: (message: A) => Effect.Effect<void, E>,
  options: {
    readonly failureThreshold: number
    readonly resetTimeout: Duration.Duration
  }
) => Effect.gen(function* () {
  const mailbox = yield* Mailbox.make<A>({ capacity: 1000 })
  const state = yield* Ref.make<CircuitBreakerState>({
    status: "closed",
    failures: 0,
    lastFailureTime: 0
  })
  
  const processor = yield* Effect.gen(function* () {
    while (true) {
      const message = yield* mailbox.take
      const currentState = yield* Ref.get(state)
      
      // Check if circuit breaker should reset
      if (currentState.status === "open") {
        const now = Date.now()
        const timeSinceFailure = now - currentState.lastFailureTime
        
        if (timeSinceFailure > Duration.toMillis(options.resetTimeout)) {
          yield* Ref.update(state, s => ({ ...s, status: "half-open" }))
        } else {
          // Skip message when circuit is open
          continue
        }
      }
      
      // Process message
      const result = yield* handler(message).pipe(Effect.either)
      
      yield* Effect.match(result, {
        onFailure: () => Ref.update(state, s => ({
          status: s.failures + 1 >= options.failureThreshold ? "open" : "closed",
          failures: s.failures + 1,
          lastFailureTime: Date.now()
        })),
        onSuccess: () => Ref.update(state, s => ({
          ...s,
          status: "closed",
          failures: 0
        }))
      })
    }
  }).pipe(
    Effect.catchAll(() => Effect.void),
    Effect.fork
  )
  
  return {
    send: (message: A) => mailbox.offer(message),
    getState: () => Ref.get(state),
    shutdown: () => Effect.gen(function* () {
      yield* mailbox.end
      yield* processor.await
    })
  } as const
})
```

## Integration Examples

### Integration with Fiber Supervision

```typescript
import { Effect, Mailbox, Supervisor, Schedule, Duration } from "effect"

const createSupervisedActorSystem = () => Effect.gen(function* () {
  const supervisor = yield* Supervisor.track
  const actors = yield* Ref.make<Array<{ id: string; fiber: Fiber.Fiber<void> }>>(Array.empty())
  
  const createActor = (id: string, handler: (msg: string) => Effect.Effect<void>) =>
    Effect.gen(function* () {
      const mailbox = yield* Mailbox.make<string>({ capacity: 100 })
      
      const fiber = yield* Effect.gen(function* () {
        while (true) {
          const message = yield* mailbox.take
          yield* handler(message).pipe(
            Effect.retry(Schedule.exponential(Duration.millis(100)).pipe(
              Schedule.compose(Schedule.recurs(3))
            ))
          )
        }
      }).pipe(
        Effect.supervised(supervisor),
        Effect.fork
      )
      
      yield* Ref.update(actors, arr => [...arr, { id, fiber }])
      
      return {
        send: (message: string) => mailbox.offer(message),
        mailbox: mailbox as Mailbox.ReadonlyMailbox<string>
      } as const
    })
  
  return {
    createActor,
    
    shutdown: () => Effect.gen(function* () {
      const currentActors = yield* Ref.get(actors)
      yield* Effect.forEach(
        currentActors,
        ({ fiber }) => Fiber.interrupt(fiber),
        { concurrency: "unbounded" }
      )
    }),
    
    getFailures: () => supervisor.value
  } as const
})
```

### Integration with STM for Coordinated State

```typescript
import { Effect, Mailbox, STM, TRef } from "effect"

interface BankAccount {
  readonly id: string
  readonly balance: TRef.TRef<number>
}

interface Transaction {
  readonly from: string
  readonly to: string
  readonly amount: number
  readonly resolve: (success: boolean) => Effect.Effect<void>
}

const createBankingSystem = () => Effect.gen(function* () {
  const accounts = yield* TRef.make<Map<string, BankAccount>>(new Map())
  const transactions = yield* Mailbox.make<Transaction>({ capacity: 10000 })
  
  const processor = yield* Effect.gen(function* () {
    while (true) {
      const transaction = yield* transactions.take
      
      const result = yield* STM.gen(function* () {
        const accountMap = yield* TRef.get(accounts)
        const fromAccount = accountMap.get(transaction.from)
        const toAccount = accountMap.get(transaction.to)
        
        if (!fromAccount || !toAccount) {
          return false
        }
        
        const fromBalance = yield* TRef.get(fromAccount.balance)
        if (fromBalance < transaction.amount) {
          return false
        }
        
        yield* TRef.update(fromAccount.balance, b => b - transaction.amount)
        yield* TRef.update(toAccount.balance, b => b + transaction.amount)
        
        return true
      }).pipe(STM.commit)
      
      yield* transaction.resolve(result)
    }
  }).pipe(
    Effect.catchAll(() => Effect.void),
    Effect.fork
  )
  
  return {
    createAccount: (id: string, initialBalance: number) =>
      STM.gen(function* () {
        const balance = yield* TRef.make(initialBalance)
        const account: BankAccount = { id, balance }
        yield* TRef.update(accounts, map => new Map(map).set(id, account))
        return account
      }).pipe(STM.commit),
    
    transfer: (from: string, to: string, amount: number) =>
      Effect.gen(function* () {
        const deferred = yield* Effect.Deferred.make<boolean>()
        
        yield* transactions.offer({
          from,
          to,
          amount,
          resolve: (success) => deferred.succeed(success)
        })
        
        return yield* deferred.await
      }),
    
    getBalance: (accountId: string) =>
      STM.gen(function* () {
        const accountMap = yield* TRef.get(accounts)
        const account = accountMap.get(accountId)
        if (!account) return Option.none()
        
        const balance = yield* TRef.get(account.balance)
        return Option.some(balance)
      }).pipe(STM.commit),
    
    shutdown: () => Effect.gen(function* () {
      yield* transactions.end
      yield* processor.await
    })
  } as const
})
```

### Testing Mailbox-Based Systems

```typescript
import { Effect, Testable, TestClock, Duration } from "effect"

const testMailboxSystem = Effect.gen(function* () {
  const testClock = yield* TestClock.TestClock
  
  // Create a mailbox-based timer system
  const createTimerSystem = () => Effect.gen(function* () {
    const timers = yield* Mailbox.make<{ id: string; delay: Duration.Duration }>()
    
    const processor = yield* Effect.gen(function* () {
      while (true) {
        const timer = yield* timers.take
        yield* Effect.sleep(timer.delay)
        yield* Effect.sync(() => console.log(`Timer ${timer.id} fired`))
      }
    }).pipe(Effect.fork)
    
    return {
      setTimer: (id: string, delay: Duration.Duration) =>
        timers.offer({ id, delay }),
      shutdown: () => Effect.gen(function* () {
        yield* timers.end
        yield* processor.await
      })
    } as const
  })
  
  const system = yield* createTimerSystem()
  
  // Set some timers
  yield* system.setTimer("timer1", Duration.seconds(1))
  yield* system.setTimer("timer2", Duration.seconds(2))
  
  // Advance test clock
  yield* TestClock.adjust(Duration.seconds(1))
  yield* TestClock.adjust(Duration.seconds(1))
  
  yield* system.shutdown()
})
```

## Conclusion

Mailbox provides type-safe, structured message passing for actor-style systems in Effect, eliminating the complexity and error-proneness of traditional concurrent programming patterns.

Key benefits:
- **Type Safety**: Compile-time guarantees for message types and error handling
- **Backpressure Management**: Built-in strategies for handling capacity overflow
- **Lifecycle Management**: Structured completion and error handling with proper cleanup
- **Integration**: Seamless integration with Effect's ecosystem (Fiber, STM, Stream, etc.)
- **Testing**: Testable patterns with Effect's testing utilities

Mailbox is ideal for building robust concurrent systems where you need reliable message passing, proper error handling, and coordinated resource management. It's particularly valuable for implementing actor systems, event sourcing, request-response patterns, and any scenario requiring structured communication between concurrent processes.