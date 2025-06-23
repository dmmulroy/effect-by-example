# PubSub: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem PubSub Solves

In modern applications, components often need to communicate asynchronously without tight coupling. Traditional approaches lead to fragile, hard-to-maintain architectures:

```typescript
// Traditional approach - tightly coupled event handling
class OrderService {
  async processOrder(order: Order) {
    // Process the order
    const processedOrder = await this.validateAndProcess(order)
    
    // Manually notify each dependent service - brittle!
    await this.emailService.sendConfirmation(processedOrder)
    await this.inventoryService.updateStock(processedOrder)
    await this.analyticsService.trackOrder(processedOrder)
    await this.auditService.logOrder(processedOrder)
    
    return processedOrder
  }
}
```

This approach leads to:
- **Tight Coupling** - OrderService must know about all dependent services
- **Brittleness** - Adding new services requires modifying existing code
- **Error Cascading** - If any notification fails, the entire operation may fail
- **Testing Complexity** - Must mock all dependent services for unit tests

### The PubSub Solution

PubSub enables decoupled, event-driven communication where publishers broadcast messages to all current subscribers without knowing who's listening:

```typescript
import { Effect, PubSub, Layer } from "effect"

// Clean, decoupled approach
export const processOrder = (order: Order) => Effect.gen(function* () {
  const orderPubSub = yield* OrderPubSub
  
  // Process the order
  const processedOrder = yield* validateAndProcess(order)
  
  // Broadcast event - OrderService doesn't know who's listening
  yield* orderPubSub.publish({ type: 'OrderProcessed', data: processedOrder })
  
  return processedOrder
}).pipe(
  Effect.withSpan('order.process')
)
```

### Key Concepts

**PubSub**: An asynchronous message hub where publishers send messages that are broadcast to all current subscribers, unlike queues where each message goes to only one consumer.

**Publishing**: Sending messages to the PubSub that will be delivered to all active subscribers.

**Subscribing**: Creating a scoped subscription that receives all messages published while the subscription is active.

**Backpressure Strategies**: Different approaches for handling when the PubSub reaches capacity (bounded, dropping, sliding, unbounded).

## Basic Usage Patterns

### Pattern 1: Creating and Basic Publishing

```typescript
import { Effect, PubSub, Queue } from "effect"

// Create a bounded PubSub with capacity of 16
const createNotificationHub = () => PubSub.bounded<string>(16)

// Basic publish and subscribe
const basicExample = Effect.gen(function* () {
  const pubsub = yield* createNotificationHub()
  
  // Subscribe to receive messages
  const subscription = yield* pubsub.subscribe
  
  // Publish a message
  const published = yield* pubsub.publish("Hello, subscribers!")
  
  // Receive the message
  const message = yield* subscription.take
  
  console.log(`Published: ${published}, Received: ${message}`)
}).pipe(Effect.scoped)
```

### Pattern 2: Multiple Subscribers

```typescript
import { Effect, PubSub, Queue, Array as Arr } from "effect"

const multiSubscriberExample = Effect.gen(function* () {
  const pubsub = yield* PubSub.bounded<string>(16)
  
  // Create multiple subscribers
  const subscriber1 = yield* pubsub.subscribe
  const subscriber2 = yield* pubsub.subscribe
  const subscriber3 = yield* pubsub.subscribe
  
  // Publish multiple messages
  yield* pubsub.publishAll(["Message 1", "Message 2", "Message 3"])
  
  // Each subscriber receives all messages
  const messages1 = yield* subscriber1.takeAll
  const messages2 = yield* subscriber2.takeAll
  const messages3 = yield* subscriber3.takeAll
  
  console.log("Subscriber 1:", [...messages1])
  console.log("Subscriber 2:", [...messages2])
  console.log("Subscriber 3:", [...messages3])
}).pipe(Effect.scoped)
```

### Pattern 3: Different PubSub Strategies

```typescript
import { Effect, PubSub } from "effect"

// Bounded: Applies backpressure when full
const boundedPubSub = PubSub.bounded<string>(10)

// Dropping: Drops new messages when full
const droppingPubSub = PubSub.dropping<string>(10)

// Sliding: Removes oldest messages when full
const slidingPubSub = PubSub.sliding<string>(10)

// Unbounded: No capacity limit (use carefully!)
const unboundedPubSub = PubSub.unbounded<string>()
```

## Real-World Examples

### Example 1: Event-Driven Order Processing System

Real-world e-commerce applications need to handle order events across multiple services without tight coupling:

```typescript
import { Effect, PubSub, Layer, Context, Fiber } from "effect"

// Domain models
interface OrderEvent {
  readonly type: 'OrderCreated' | 'OrderPaid' | 'OrderShipped' | 'OrderCancelled'
  readonly orderId: string
  readonly timestamp: Date
  readonly data: Record<string, unknown>
}

interface OrderProcessedEvent extends OrderEvent {
  readonly type: 'OrderPaid'
  readonly data: {
    readonly amount: number
    readonly customerId: string
    readonly items: ReadonlyArray<{ productId: string; quantity: number }>
  }
}

// PubSub service
class OrderEventBus extends Context.Tag("OrderEventBus")<
  OrderEventBus,
  PubSub.PubSub<OrderEvent>
>() {}

// Publisher service
export const OrderService = {
  processPayment: (orderId: string, amount: number, customerId: string) => 
    Effect.gen(function* () {
      const eventBus = yield* OrderEventBus
      
      // Process payment logic here...
      const processedAt = new Date()
      
      // Broadcast the event
      const event: OrderProcessedEvent = {
        type: 'OrderPaid',
        orderId,
        timestamp: processedAt,
        data: { amount, customerId, items: [] }
      }
      
      yield* eventBus.publish(event)
      
      return { orderId, processedAt, amount }
    }).pipe(
      Effect.withSpan('order.process-payment')
    )
}

// Subscriber services
export const EmailService = {
  start: () => Effect.gen(function* () {
    const eventBus = yield* OrderEventBus
    const subscription = yield* eventBus.subscribe
    
    yield* Effect.forever(
      Effect.gen(function* () {
        const event = yield* subscription.take
        
        if (event.type === 'OrderPaid') {
          console.log(`📧 Sending confirmation email for order ${event.orderId}`)
          // Send email logic here...
        }
      })
    ).pipe(
      Effect.fork
    )
  }).pipe(Effect.scoped)
}

export const InventoryService = {
  start: () => Effect.gen(function* () {
    const eventBus = yield* OrderEventBus
    const subscription = yield* eventBus.subscribe
    
    yield* Effect.forever(
      Effect.gen(function* () {
        const event = yield* subscription.take
        
        if (event.type === 'OrderPaid') {
          console.log(`📦 Updating inventory for order ${event.orderId}`)
          // Update inventory logic here...
        }
      })
    ).pipe(
      Effect.fork
    )
  }).pipe(Effect.scoped)
}

export const AnalyticsService = {
  start: () => Effect.gen(function* () {
    const eventBus = yield* OrderEventBus
    const subscription = yield* eventBus.subscribe
    
    yield* Effect.forever(
      Effect.gen(function* () {
        const event = yield* subscription.take
        
        console.log(`📊 Recording analytics for ${event.type} event`)
        // Analytics logic here...
      })
    ).pipe(
      Effect.fork
    )
  }).pipe(Effect.scoped)
}

// Application setup
const OrderEventBusLive = Layer.effect(
  OrderEventBus,
  PubSub.bounded<OrderEvent>(100)
)

const program = Effect.gen(function* () {
  // Start all subscriber services
  yield* EmailService.start()
  yield* InventoryService.start()
  yield* AnalyticsService.start()
  
  // Simulate order processing
  yield* OrderService.processPayment("order-123", 99.99, "customer-456")
  yield* OrderService.processPayment("order-124", 149.50, "customer-789")
  
  // Give subscribers time to process
  yield* Effect.sleep("1 second")
}).pipe(
  Effect.provide(OrderEventBusLive),
  Effect.scoped
)
```

### Example 2: Real-Time Chat System

Building a real-time chat system where messages are broadcast to all connected users:

```typescript
import { Effect, PubSub, Layer, Context, Ref } from "effect"

interface ChatMessage {
  readonly id: string
  readonly userId: string
  readonly username: string
  readonly content: string
  readonly timestamp: Date
  readonly roomId: string
}

interface UserJoinedEvent {
  readonly type: 'user-joined'
  readonly userId: string
  readonly username: string
  readonly roomId: string
  readonly timestamp: Date
}

interface UserLeftEvent {
  readonly type: 'user-left'
  readonly userId: string
  readonly roomId: string
  readonly timestamp: Date
}

type RoomEvent = ChatMessage | UserJoinedEvent | UserLeftEvent

// Room-specific PubSub services
class RoomEventBus extends Context.Tag("RoomEventBus")<
  RoomEventBus,
  PubSub.PubSub<RoomEvent>
>() {}

class RoomUsers extends Context.Tag("RoomUsers")<
  RoomUsers,
  Ref.Ref<Set<string>>
>() {}

export const ChatRoom = {
  sendMessage: (userId: string, username: string, content: string, roomId: string) =>
    Effect.gen(function* () {
      const eventBus = yield* RoomEventBus
      
      const message: ChatMessage = {
        id: crypto.randomUUID(),
        userId,
        username,
        content,
        timestamp: new Date(),
        roomId
      }
      
      yield* eventBus.publish(message)
      
      return message
    }).pipe(
      Effect.withSpan('chat.send-message')
    ),

  joinRoom: (userId: string, username: string, roomId: string) =>
    Effect.gen(function* () {
      const eventBus = yield* RoomEventBus
      const users = yield* RoomUsers
      
      // Add user to room
      yield* Ref.update(users, (userSet) => userSet.add(userId))
      
      // Broadcast join event
      const joinEvent: UserJoinedEvent = {
        type: 'user-joined',
        userId,
        username,
        roomId,
        timestamp: new Date()
      }
      
      yield* eventBus.publish(joinEvent)
      
      return { userId, roomId }
    }).pipe(
      Effect.withSpan('chat.join-room')
    ),

  leaveRoom: (userId: string, roomId: string) =>
    Effect.gen(function* () {
      const eventBus = yield* RoomEventBus
      const users = yield* RoomUsers
      
      // Remove user from room
      yield* Ref.update(users, (userSet) => {
        const newSet = new Set(userSet)
        newSet.delete(userId)
        return newSet
      })
      
      // Broadcast leave event
      const leaveEvent: UserLeftEvent = {
        type: 'user-left',
        userId,
        roomId,
        timestamp: new Date()
      }
      
      yield* eventBus.publish(leaveEvent)
      
      return { userId, roomId }
    }).pipe(
      Effect.withSpan('chat.leave-room')
    )
}

// User connection handler
export const UserConnection = {
  startListening: (userId: string, onMessage: (event: RoomEvent) => void) =>
    Effect.gen(function* () {
      const eventBus = yield* RoomEventBus
      const subscription = yield* eventBus.subscribe
      
      return yield* Effect.forever(
        Effect.gen(function* () {
          const event = yield* subscription.take
          onMessage(event)
        })
      ).pipe(
        Effect.fork
      )
    }).pipe(Effect.scoped)
}

// Chat room setup
const RoomEventBusLive = Layer.effect(
  RoomEventBus,
  PubSub.bounded<RoomEvent>(1000)
)

const RoomUsersLive = Layer.effect(
  RoomUsers,
  Ref.make(new Set<string>())
)

const chatRoomLayer = Layer.mergeAll(RoomEventBusLive, RoomUsersLive)

// Simulate chat room usage
const simulateChat = Effect.gen(function* () {
  // Simulate users joining
  yield* ChatRoom.joinRoom("user1", "Alice", "room-general")
  yield* ChatRoom.joinRoom("user2", "Bob", "room-general")
  
  // Start listening for one user
  const aliceConnection = yield* UserConnection.startListening("user1", (event) => {
    if (event.type === 'user-joined') {
      console.log(`👋 ${event.username} joined the room`)
    } else if (event.type === 'user-left') {
      console.log(`👋 User ${event.userId} left the room`)
    } else {
      console.log(`💬 ${event.username}: ${event.content}`)
    }
  })
  
  // Send some messages
  yield* ChatRoom.sendMessage("user1", "Alice", "Hello everyone!", "room-general")
  yield* ChatRoom.sendMessage("user2", "Bob", "Hey Alice! How's it going?", "room-general")
  yield* ChatRoom.sendMessage("user1", "Alice", "Great! Thanks for asking 😊", "room-general")
  
  yield* Effect.sleep("2 seconds")
  
  // User leaves
  yield* ChatRoom.leaveRoom("user2", "room-general")
  
  yield* Effect.sleep("1 second")
}).pipe(
  Effect.provide(chatRoomLayer),
  Effect.scoped
)
```

### Example 3: Distributed Logging and Metrics System

Building a system where different components publish logs and metrics to centralized collectors:

```typescript
import { Effect, PubSub, Layer, Context, LogLevel } from "effect"

interface LogEntry {
  readonly level: LogLevel.LogLevel
  readonly message: string
  readonly service: string
  readonly timestamp: Date
  readonly metadata?: Record<string, unknown>
}

interface MetricEvent {
  readonly type: 'counter' | 'gauge' | 'histogram'
  readonly name: string
  readonly value: number
  readonly tags: Record<string, string>
  readonly timestamp: Date
}

// Observability services
class LogPubSub extends Context.Tag("LogPubSub")<
  LogPubSub,
  PubSub.PubSub<LogEntry>
>() {}

class MetricsPubSub extends Context.Tag("MetricsPubSub")<
  MetricsPubSub,
  PubSub.PubSub<MetricEvent>
>() {}

// Helper for creating structured logs
export const createLogger = (serviceName: string) => ({
  info: (message: string, metadata?: Record<string, unknown>) =>
    Effect.gen(function* () {
      const logPubSub = yield* LogPubSub
      
      const logEntry: LogEntry = {
        level: LogLevel.Info,
        message,
        service: serviceName,
        timestamp: new Date(),
        metadata
      }
      
      yield* logPubSub.publish(logEntry)
    }),

  error: (message: string, metadata?: Record<string, unknown>) =>
    Effect.gen(function* () {
      const logPubSub = yield* LogPubSub
      
      const logEntry: LogEntry = {
        level: LogLevel.Error,
        message,
        service: serviceName,
        timestamp: new Date(),
        metadata
      }
      
      yield* logPubSub.publish(logEntry)
    }),

  warn: (message: string, metadata?: Record<string, unknown>) =>
    Effect.gen(function* () {
      const logPubSub = yield* LogPubSub
      
      const logEntry: LogEntry = {
        level: LogLevel.Warning,
        message,
        service: serviceName,
        timestamp: new Date(),
        metadata
      }
      
      yield* logPubSub.publish(logEntry)
    })
})

// Helper for creating metrics
export const createMetrics = (serviceName: string) => ({
  increment: (name: string, tags: Record<string, string> = {}) =>
    Effect.gen(function* () {
      const metricsPubSub = yield* MetricsPubSub
      
      const metric: MetricEvent = {
        type: 'counter',
        name,
        value: 1,
        tags: { ...tags, service: serviceName },
        timestamp: new Date()
      }
      
      yield* metricsPubSub.publish(metric)
    }),

  gauge: (name: string, value: number, tags: Record<string, string> = {}) =>
    Effect.gen(function* () {
      const metricsPubSub = yield* MetricsPubSub
      
      const metric: MetricEvent = {
        type: 'gauge',
        name,
        value,
        tags: { ...tags, service: serviceName },
        timestamp: new Date()
      }
      
      yield* metricsPubSub.publish(metric)
    })
})

// Log collectors
export const ConsoleLogCollector = {
  start: () => Effect.gen(function* () {
    const logPubSub = yield* LogPubSub
    const subscription = yield* logPubSub.subscribe
    
    yield* Effect.forever(
      Effect.gen(function* () {
        const log = yield* subscription.take
        const timestamp = log.timestamp.toISOString()
        const level = log.level.label.toUpperCase()
        console.log(`[${timestamp}] ${level} [${log.service}] ${log.message}`)
        
        if (log.metadata) {
          console.log('  Metadata:', JSON.stringify(log.metadata, null, 2))
        }
      })
    ).pipe(
      Effect.fork
    )
  }).pipe(Effect.scoped)
}

export const MetricsCollector = {
  start: () => Effect.gen(function* () {
    const metricsPubSub = yield* MetricsPubSub
    const subscription = yield* metricsPubSub.subscribe
    
    yield* Effect.forever(
      Effect.gen(function* () {
        const metric = yield* subscription.take
        
        console.log(
          `📊 ${metric.type.toUpperCase()}: ${metric.name} = ${metric.value}`,
          Object.keys(metric.tags).length > 0 ? metric.tags : ''
        )
      })
    ).pipe(
      Effect.fork
    )
  }).pipe(Effect.scoped)
}

// Business service using observability
export const UserService = {
  createUser: (userData: { name: string; email: string }) =>
    Effect.gen(function* () {
      const logger = createLogger('UserService')
      const metrics = createMetrics('UserService')
      
      yield* logger.info('Creating new user', { email: userData.email })
      yield* metrics.increment('user.create.started')
      
      // Simulate user creation
      const userId = crypto.randomUUID()
      
      if (Math.random() > 0.8) {
        yield* logger.error('Failed to create user', { 
          email: userData.email,
          error: 'Database connection failed'
        })
        yield* metrics.increment('user.create.failed', { reason: 'database_error' })
        return Effect.fail(new Error('User creation failed'))
      }
      
      yield* logger.info('User created successfully', { 
        userId, 
        email: userData.email 
      })
      yield* metrics.increment('user.create.success')
      yield* metrics.gauge('users.total_count', Math.floor(Math.random() * 1000))
      
      return { userId, ...userData }
    }).pipe(
      Effect.withSpan('user.create')
    )
}

// Observability infrastructure
const ObservabilityLive = Layer.mergeAll(
  Layer.effect(LogPubSub, PubSub.unbounded<LogEntry>()),
  Layer.effect(MetricsPubSub, PubSub.unbounded<MetricEvent>())
)

// Application using distributed observability
const observabilityDemo = Effect.gen(function* () {
  // Start collectors
  yield* ConsoleLogCollector.start()
  yield* MetricsCollector.start()
  
  // Simulate some user operations
  yield* UserService.createUser({ name: "Alice", email: "alice@example.com" })
  yield* UserService.createUser({ name: "Bob", email: "bob@example.com" })
  yield* UserService.createUser({ name: "Charlie", email: "charlie@example.com" })
  
  yield* Effect.sleep("3 seconds")
}).pipe(
  Effect.provide(ObservabilityLive),
  Effect.scoped
)
```

## Advanced Features Deep Dive

### Feature 1: Backpressure Strategies

Understanding when and how to use different PubSub strategies is crucial for building robust systems.

#### Basic Strategy Usage

```typescript
import { Effect, PubSub } from "effect"

// Bounded: Best for guaranteed delivery
const boundedExample = Effect.gen(function* () {
  const pubsub = yield* PubSub.bounded<string>(2)
  
  const subscription = yield* pubsub.subscribe
  
  // This will block when capacity is reached
  yield* pubsub.publish("Message 1")
  yield* pubsub.publish("Message 2")
  // Next publish will wait for subscriber to consume
  yield* pubsub.publish("Message 3")
  
  console.log(yield* subscription.take) // "Message 1"
  console.log(yield* subscription.take) // "Message 2"  
  console.log(yield* subscription.take) // "Message 3"
}).pipe(Effect.scoped)
```

#### Real-World Strategy Example

```typescript
interface SystemLoadMonitor {
  readonly strategy: 'bounded' | 'dropping' | 'sliding'
  readonly capacity: number
}

const createMonitoringPubSub = (config: SystemLoadMonitor) => {
  switch (config.strategy) {
    case 'bounded':
      // Critical alerts - ensure delivery
      return PubSub.bounded<AlertEvent>(config.capacity)
    
    case 'dropping':
      // High-frequency metrics - drop if overwhelmed  
      return PubSub.dropping<MetricEvent>(config.capacity)
    
    case 'sliding':
      // Log events - keep most recent
      return PubSub.sliding<LogEvent>(config.capacity)
  }
}

// Use cases for each strategy
const alertingSystem = Effect.gen(function* () {
  // Critical system alerts - never drop
  const criticalAlerts = yield* PubSub.bounded<AlertEvent>(100)
  
  // High-frequency metrics - dropping is acceptable
  const metrics = yield* PubSub.dropping<MetricEvent>(1000)
  
  // Debug logs - sliding window of recent events
  const debugLogs = yield* PubSub.sliding<LogEvent>(500)
  
  return { criticalAlerts, metrics, debugLogs }
})
```

#### Advanced Strategy: Replay Configuration

```typescript
import { Effect, PubSub } from "effect"

// PubSub with replay - new subscribers get recent messages
const replayExample = Effect.gen(function* () {
  // Keep last 5 messages for new subscribers
  const pubsub = yield* PubSub.bounded<string>({ capacity: 16, replay: 5 })
  
  // Publish some messages before subscribing
  yield* pubsub.publishAll(["Msg 1", "Msg 2", "Msg 3", "Msg 4", "Msg 5", "Msg 6"])
  
  // New subscriber gets last 5 messages
  const newSubscriber = yield* pubsub.subscribe
  const replayedMessages = yield* newSubscriber.takeAll
  
  console.log("Replayed messages:", [...replayedMessages])
  // Output: ["Msg 2", "Msg 3", "Msg 4", "Msg 5", "Msg 6"]
}).pipe(Effect.scoped)
```

### Feature 2: Lifecycle Management

Managing PubSub lifecycle and graceful shutdown is essential for production systems.

#### Basic Lifecycle Operations

```typescript
import { Effect, PubSub, Fiber } from "effect"

const lifecycleExample = Effect.gen(function* () {
  const pubsub = yield* PubSub.bounded<string>(16)
  
  // Check current state
  console.log("Capacity:", PubSub.capacity(pubsub))
  console.log("Size:", yield* PubSub.size(pubsub))
  console.log("Is empty:", yield* PubSub.isEmpty(pubsub))
  console.log("Is full:", yield* PubSub.isFull(pubsub))
  
  // Start subscriber in background
  const subscriber = yield* Effect.gen(function* () {
    const subscription = yield* pubsub.subscribe
    
    while (!(yield* PubSub.isShutdown(pubsub))) {
      const message = yield* subscription.take
      console.log("Received:", message)
    }
  }).pipe(
    Effect.scoped,
    Effect.fork
  )
  
  // Publish some messages
  yield* pubsub.publishAll(["Hello", "World"])
  
  // Graceful shutdown
  yield* PubSub.shutdown(pubsub)
  
  // Wait for subscriber to finish
  yield* Fiber.join(subscriber)
  
  console.log("Is shutdown:", yield* PubSub.isShutdown(pubsub))
})
```

#### Real-World Lifecycle: Service Shutdown

```typescript
import { Effect, PubSub, Fiber, Context, Layer, Ref } from "effect"

interface ServiceManager {
  readonly shutdown: Effect.Effect<void>
  readonly awaitShutdown: Effect.Effect<void>
  readonly isShuttingDown: Effect.Effect<boolean>
}

class ServiceManagerTag extends Context.Tag("ServiceManager")<
  ServiceManagerTag,
  ServiceManager
>() {}

const makeServiceManager = Effect.gen(function* () {
  const shutdownRef = yield* Ref.make(false)
  const pubsub = yield* PubSub.unbounded<'shutdown'>()
  
  return {
    shutdown: Effect.gen(function* () {
      const alreadyShuttingDown = yield* Ref.get(shutdownRef)
      if (alreadyShuttingDown) return
      
      yield* Ref.set(shutdownRef, true)
      yield* pubsub.publish('shutdown')
      yield* PubSub.shutdown(pubsub)
    }),
    
    awaitShutdown: Effect.gen(function* () {
      const subscription = yield* pubsub.subscribe
      yield* subscription.take // Wait for shutdown signal
    }).pipe(Effect.scoped),
    
    isShuttingDown: Ref.get(shutdownRef)
  }
})

const ServiceManagerLive = Layer.effect(ServiceManagerTag, makeServiceManager)

// Example service using graceful shutdown
const LongRunningService = {
  start: () => Effect.gen(function* () {
    const serviceManager = yield* ServiceManagerTag
    
    const worker = yield* Effect.forever(
      Effect.gen(function* () {
        const isShuttingDown = yield* serviceManager.isShuttingDown
        if (isShuttingDown) {
          console.log("Service is shutting down, stopping work...")
          return yield* Effect.interrupt
        }
        
        console.log("Doing work...")
        yield* Effect.sleep("1 second")
      })
    ).pipe(
      Effect.fork
    )
    
    // Wait for shutdown signal
    yield* serviceManager.awaitShutdown
    
    console.log("Shutdown signal received, cleaning up...")
    yield* Fiber.interrupt(worker)
    console.log("Service stopped gracefully")
  })
}
```

### Feature 3: Advanced Publishing Patterns

#### Conditional Publishing

```typescript
import { Effect, PubSub, Option } from "effect"

// Publish only if conditions are met
const conditionalPublish = <A>(
  pubsub: PubSub.PubSub<A>,
  value: A,
  condition: (value: A) => boolean
) => 
  condition(value) 
    ? pubsub.publish(value)
    : Effect.succeed(false)

// Rate-limited publishing
const rateLimitedPublish = <A>(
  pubsub: PubSub.PubSub<A>,
  values: ReadonlyArray<A>,
  delayBetween: string
) => Effect.gen(function* () {
  const results: boolean[] = []
  
  for (const value of values) {
    const result = yield* pubsub.publish(value)
    results.push(result)
    
    if (value !== values[values.length - 1]) {
      yield* Effect.sleep(delayBetween)
    }
  }
  
  return results
})

// Example usage
const advancedPublishingExample = Effect.gen(function* () {
  const pubsub = yield* PubSub.bounded<number>(10)
  const subscription = yield* pubsub.subscribe
  
  // Only publish even numbers
  yield* conditionalPublish(pubsub, 1, (n) => n % 2 === 0) // false
  yield* conditionalPublish(pubsub, 2, (n) => n % 2 === 0) // true
  yield* conditionalPublish(pubsub, 3, (n) => n % 2 === 0) // false
  yield* conditionalPublish(pubsub, 4, (n) => n % 2 === 0) // true
  
  // Rate-limited batch publishing
  yield* rateLimitedPublish(pubsub, [10, 20, 30], "100 millis")
  
  const messages = yield* subscription.takeAll
  console.log("Received messages:", [...messages])
}).pipe(Effect.scoped)
```

## Practical Patterns & Best Practices

### Pattern 1: Event Sourcing with PubSub

```typescript
import { Effect, PubSub, Context, Layer, Ref, Array as Arr } from "effect"

// Event store pattern
interface DomainEvent {
  readonly id: string
  readonly aggregateId: string
  readonly type: string
  readonly payload: Record<string, unknown>
  readonly timestamp: Date
  readonly version: number
}

class EventStore extends Context.Tag("EventStore")<
  EventStore,
  {
    readonly append: (events: ReadonlyArray<DomainEvent>) => Effect.Effect<void>
    readonly subscribe: Effect.Effect<PubSub.PubSub<DomainEvent>>
  }
>() {}

const makeEventStore = Effect.gen(function* () {
  const eventLog = yield* Ref.make<ReadonlyArray<DomainEvent>>([])
  const eventBus = yield* PubSub.unbounded<DomainEvent>()
  
  return {
    append: (events: ReadonlyArray<DomainEvent>) => Effect.gen(function* () {
      // Store events
      yield* Ref.update(eventLog, (log) => [...log, ...events])
      
      // Publish to subscribers
      yield* eventBus.publishAll(events)
    }),
    
    subscribe: Effect.succeed(eventBus)
  }
})

const EventStoreLive = Layer.effect(EventStore, makeEventStore)

// Domain aggregate using event sourcing
export const UserAggregate = {
  create: (userId: string, name: string, email: string) => Effect.gen(function* () {
    const eventStore = yield* EventStore
    
    const event: DomainEvent = {
      id: crypto.randomUUID(),
      aggregateId: userId,
      type: 'UserCreated',
      payload: { name, email },
      timestamp: new Date(),
      version: 1
    }
    
    yield* eventStore.append([event])
    
    return { userId, name, email, version: 1 }
  }),
  
  updateEmail: (userId: string, newEmail: string, currentVersion: number) => 
    Effect.gen(function* () {
      const eventStore = yield* EventStore
      
      const event: DomainEvent = {
        id: crypto.randomUUID(),
        aggregateId: userId,
        type: 'UserEmailUpdated',
        payload: { newEmail },
        timestamp: new Date(),
        version: currentVersion + 1
      }
      
      yield* eventStore.append([event])
      
      return { userId, newEmail, version: currentVersion + 1 }
    })
}

// Event projections
export const UserProjection = {
  start: () => Effect.gen(function* () {
    const eventStore = yield* EventStore
    const eventBus = yield* eventStore.subscribe
    const subscription = yield* eventBus.subscribe
    
    const userViews = yield* Ref.make(new Map<string, {
      id: string
      name: string
      email: string
      version: number
    }>())
    
    yield* Effect.forever(
      Effect.gen(function* () {
        const event = yield* subscription.take
        
        yield* Ref.update(userViews, (views) => {
          const updatedViews = new Map(views)
          
          if (event.type === 'UserCreated') {
            updatedViews.set(event.aggregateId, {
              id: event.aggregateId,
              name: event.payload.name as string,
              email: event.payload.email as string,
              version: event.version
            })
          } else if (event.type === 'UserEmailUpdated') {
            const existing = updatedViews.get(event.aggregateId)
            if (existing) {
              updatedViews.set(event.aggregateId, {
                ...existing,
                email: event.payload.newEmail as string,
                version: event.version
              })
            }
          }
          
          return updatedViews
        })
        
        console.log(`📊 Projection updated for ${event.type}`)
      })
    ).pipe(
      Effect.fork
    )
  }).pipe(Effect.scoped)
}
```

### Pattern 2: Circuit Breaker with PubSub Notifications

```typescript
import { Effect, PubSub, Context, Layer, Ref } from "effect"

interface CircuitBreakerEvent {
  readonly type: 'opened' | 'closed' | 'half-open' | 'failure' | 'success'
  readonly circuitName: string
  readonly timestamp: Date
  readonly metadata?: Record<string, unknown>
}

interface CircuitBreakerState {
  readonly status: 'closed' | 'open' | 'half-open'
  readonly failureCount: number
  readonly lastFailureTime: Date | null
  readonly lastSuccessTime: Date | null
}

class CircuitBreakerEvents extends Context.Tag("CircuitBreakerEvents")<
  CircuitBreakerEvents,
  PubSub.PubSub<CircuitBreakerEvent>
>() {}

const makeCircuitBreaker = (
  name: string,
  options: {
    failureThreshold: number
    timeoutDuration: number
    resetTimeoutDuration: number
  }
) => Effect.gen(function* () {
  const eventBus = yield* CircuitBreakerEvents
  const state = yield* Ref.make<CircuitBreakerState>({
    status: 'closed',
    failureCount: 0,
    lastFailureTime: null,
    lastSuccessTime: null
  })
  
  const publishEvent = (type: CircuitBreakerEvent['type'], metadata?: Record<string, unknown>) =>
    eventBus.publish({
      type,
      circuitName: name,
      timestamp: new Date(),
      metadata
    })
  
  const execute = <A, E>(effect: Effect.Effect<A, E>) => Effect.gen(function* () {
    const currentState = yield* Ref.get(state)
    
    // Check if circuit is open
    if (currentState.status === 'open') {
      const now = new Date()
      const timeSinceLastFailure = currentState.lastFailureTime 
        ? now.getTime() - currentState.lastFailureTime.getTime()
        : 0
      
      if (timeSinceLastFailure < options.resetTimeoutDuration) {
        return yield* Effect.fail(new Error(`Circuit breaker ${name} is open`))
      } else {
        // Move to half-open
        yield* Ref.update(state, (s) => ({ ...s, status: 'half-open' }))
        yield* publishEvent('half-open')
      }
    }
    
    // Execute the operation
    return yield* effect.pipe(
      Effect.timeout(options.timeoutDuration),
      Effect.tapBoth({
        onFailure: () => Effect.gen(function* () {
          const now = new Date()
          
          yield* Ref.update(state, (s) => ({
            ...s,
            failureCount: s.failureCount + 1,
            lastFailureTime: now
          }))
          
          const updatedState = yield* Ref.get(state)
          
          if (updatedState.failureCount >= options.failureThreshold) {
            yield* Ref.update(state, (s) => ({ ...s, status: 'open' }))
            yield* publishEvent('opened', { failureCount: updatedState.failureCount })
          } else {
            yield* publishEvent('failure', { failureCount: updatedState.failureCount })
          }
        }),
        onSuccess: () => Effect.gen(function* () {
          const now = new Date()
          
          yield* Ref.update(state, (s) => ({
            status: 'closed',
            failureCount: 0,
            lastFailureTime: null,
            lastSuccessTime: now
          }))
          
          yield* publishEvent('success')
          
          const currentState = yield* Ref.get(state)
          if (currentState.status !== 'closed') {
            yield* publishEvent('closed')
          }
        })
      })
    )
  })
  
  return { execute, getState: () => Ref.get(state) }
})

// Circuit breaker monitoring
export const CircuitBreakerMonitor = {
  start: () => Effect.gen(function* () {
    const eventBus = yield* CircuitBreakerEvents
    const subscription = yield* eventBus.subscribe
    
    yield* Effect.forever(
      Effect.gen(function* () {
        const event = yield* subscription.take
        
        const status = event.type === 'opened' ? '🔴' :
                      event.type === 'closed' ? '🟢' :
                      event.type === 'half-open' ? '🟡' : '⚪'
        
        console.log(
          `${status} Circuit Breaker [${event.circuitName}]: ${event.type.toUpperCase()}`,
          event.metadata ? event.metadata : ''
        )
      })
    ).pipe(
      Effect.fork
    )
  }).pipe(Effect.scoped)
}

const CircuitBreakerEventsLive = Layer.effect(
  CircuitBreakerEvents,
  PubSub.unbounded<CircuitBreakerEvent>()
)
```

### Pattern 3: Message Deduplication

```typescript
import { Effect, PubSub, Context, Layer, Ref, Hash } from "effect"

interface DeduplicatedMessage<A> {
  readonly id: string
  readonly payload: A
  readonly timestamp: Date
}

const makeDedupePubSub = <A>(
  capacity: number,
  dedupWindowMs: number
) => Effect.gen(function* () {
  const pubsub = yield* PubSub.bounded<DeduplicatedMessage<A>>(capacity)
  const seenMessages = yield* Ref.make(new Map<string, Date>())
  
  const publish = (id: string, payload: A) => Effect.gen(function* () {
    const now = new Date()
    const currentSeen = yield* Ref.get(seenMessages)
    
    // Check if we've seen this message recently
    const lastSeen = currentSeen.get(id)
    if (lastSeen && (now.getTime() - lastSeen.getTime()) < dedupWindowMs) {
      return false // Duplicate message, not published
    }
    
    // Clean up old entries
    const cutoff = new Date(now.getTime() - dedupWindowMs)
    yield* Ref.update(seenMessages, (seen) => {
      const cleaned = new Map<string, Date>()
      for (const [messageId, timestamp] of seen) {
        if (timestamp.getTime() > cutoff.getTime()) {
          cleaned.set(messageId, timestamp)
        }
      }
      cleaned.set(id, now)
      return cleaned
    })
    
    // Publish the message
    const message: DeduplicatedMessage<A> = { id, payload, timestamp: now }
    return yield* pubsub.publish(message)
  })
  
  return {
    publish,
    subscribe: pubsub.subscribe,
    capacity: () => PubSub.capacity(pubsub),
    size: () => PubSub.size(pubsub)
  }
})

// Example usage
const deduplicationExample = Effect.gen(function* () {
  const dedupePubSub = yield* makeDedupePubSub<string>(10, 5000) // 5 second dedup window
  const subscription = yield* dedupePubSub.subscribe
  
  // Publish the same message multiple times
  console.log("First publish:", yield* dedupePubSub.publish("msg-1", "Hello World"))
  console.log("Duplicate publish:", yield* dedupePubSub.publish("msg-1", "Hello World"))
  console.log("Different message:", yield* dedupePubSub.publish("msg-2", "Different message"))
  console.log("Another duplicate:", yield* dedupePubSub.publish("msg-1", "Hello World"))
  
  const messages = yield* subscription.takeAll
  console.log("Received messages:", [...messages])
}).pipe(Effect.scoped)
```

## Integration Examples

### Integration with Stream Processing

PubSub integrates seamlessly with Effect's Stream API for building reactive data pipelines:

```typescript
import { Effect, PubSub, Stream, Layer, Context } from "effect"

// Stream-based data processing pipeline
export const StreamProcessor = {
  fromPubSub: <A>(pubsub: PubSub.PubSub<A>) =>
    Stream.fromQueue(pubsub.subscribe).pipe(
      Stream.scoped
    ),
  
  // Transform and republish to another PubSub
  pipeline: <A, B>(
    input: PubSub.PubSub<A>,
    output: PubSub.PubSub<B>,
    transform: (a: A) => B
  ) => Effect.gen(function* () {
    yield* StreamProcessor.fromPubSub(input).pipe(
      Stream.map(transform),
      Stream.tap((item) => output.publish(item)),
      Stream.runDrain,
      Effect.fork
    )
  })
}

// Real-time analytics pipeline
interface RawEvent {
  readonly userId: string
  readonly action: string
  readonly timestamp: Date
  readonly metadata: Record<string, unknown>
}

interface ProcessedEvent {
  readonly userId: string
  readonly action: string
  readonly timestamp: Date
  readonly sessionId: string
  readonly enrichedData: Record<string, unknown>
}

interface Analytics {
  readonly userCount: number
  readonly actionCounts: Record<string, number>
  readonly activeSessionsCount: number
}

const analyticsExample = Effect.gen(function* () {
  const rawEvents = yield* PubSub.bounded<RawEvent>(1000)
  const processedEvents = yield* PubSub.bounded<ProcessedEvent>(1000)
  const analytics = yield* PubSub.bounded<Analytics>(100)
  
  // Processing pipeline
  yield* StreamProcessor.fromPubSub(rawEvents).pipe(
    Stream.map((raw): ProcessedEvent => ({
      ...raw,
      sessionId: `session-${raw.userId}-${Date.now()}`,
      enrichedData: { processed: true, ...raw.metadata }
    })),
    Stream.tap((processed) => processedEvents.publish(processed)),
    Stream.runDrain,
    Effect.fork
  )
  
  // Analytics aggregation
  yield* StreamProcessor.fromPubSub(processedEvents).pipe(
    Stream.groupedWithin(10, "1 second"),
    Stream.map((batch) => {
      const uniqueUsers = new Set(batch.map(e => e.userId)).size
      const actionCounts: Record<string, number> = {}
      const activeSessions = new Set(batch.map(e => e.sessionId)).size
      
      for (const event of batch) {
        actionCounts[event.action] = (actionCounts[event.action] || 0) + 1
      }
      
      return {
        userCount: uniqueUsers,
        actionCounts,
        activeSessionsCount: activeSessions
      }
    }),
    Stream.tap((analyticsData) => analytics.publish(analyticsData)),
    Stream.runDrain,
    Effect.fork
  )
  
  // Subscribe to analytics
  const analyticsSubscription = yield* analytics.subscribe
  
  // Simulate events
  yield* rawEvents.publishAll([
    { userId: "user1", action: "click", timestamp: new Date(), metadata: {} },
    { userId: "user2", action: "view", timestamp: new Date(), metadata: {} },
    { userId: "user1", action: "purchase", timestamp: new Date(), metadata: {} }
  ])
  
  yield* Effect.sleep("2 seconds")
  
  const analyticsResult = yield* analyticsSubscription.takeAll
  console.log("Analytics:", [...analyticsResult])
}).pipe(Effect.scoped)
```

### Integration with HTTP Server

Building real-time web applications with Server-Sent Events:

```typescript
import { Effect, PubSub, Layer, Context } from "effect"

// Simulate a simplified HTTP context
interface HttpContext {
  readonly request: { readonly url: string }
  readonly response: {
    readonly writeHead: (status: number, headers: Record<string, string>) => void
    readonly write: (data: string) => void
    readonly end: () => void
  }
}

interface NotificationEvent {
  readonly type: 'user_message' | 'system_alert' | 'status_update'
  readonly userId?: string
  readonly message: string
  readonly timestamp: Date
}

class NotificationPubSub extends Context.Tag("NotificationPubSub")<
  NotificationPubSub,
  PubSub.PubSub<NotificationEvent>
>() {}

// SSE endpoint handler
const handleSSEConnection = (ctx: HttpContext, userId: string) => 
  Effect.gen(function* () {
    const notifications = yield* NotificationPubSub
    
    // Set up SSE headers
    ctx.response.writeHead(200, {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
      'Access-Control-Allow-Origin': '*'
    })
    
    // Subscribe to notifications
    const subscription = yield* notifications.subscribe
    
    // Send initial connection event
    ctx.response.write(`data: ${JSON.stringify({
      type: 'connected',
      message: 'SSE connection established'
    })}\n\n`)
    
    // Stream notifications to client
    yield* Effect.forever(
      Effect.gen(function* () {
        const event = yield* subscription.take
        
        // Filter events for this user
        if (!event.userId || event.userId === userId) {
          const sseData = `data: ${JSON.stringify(event)}\n\n`
          ctx.response.write(sseData)
        }
      })
    ).pipe(
      Effect.ensuring(Effect.sync(() => ctx.response.end()))
    )
  }).pipe(Effect.scoped)

// Notification service
export const NotificationService = {
  broadcast: (message: string) => Effect.gen(function* () {
    const notifications = yield* NotificationPubSub
    
    const event: NotificationEvent = {
      type: 'system_alert',
      message,
      timestamp: new Date()
    }
    
    yield* notifications.publish(event)
  }),
  
  sendToUser: (userId: string, message: string) => Effect.gen(function* () {
    const notifications = yield* NotificationPubSub
    
    const event: NotificationEvent = {
      type: 'user_message',
      userId,
      message,
      timestamp: new Date()
    }
    
    yield* notifications.publish(event)
  })
}

// WebSocket-style real-time updates
export const WebSocketHandler = {
  handleConnection: (userId: string) => Effect.gen(function* () {
    const notifications = yield* NotificationPubSub
    const subscription = yield* notifications.subscribe
    
    console.log(`📡 User ${userId} connected to WebSocket`)
    
    const messageHandler = yield* Effect.forever(
      Effect.gen(function* () {
        const event = yield* subscription.take
        
        if (!event.userId || event.userId === userId) {
          console.log(`📨 Sending to ${userId}:`, event.message)
          // In real implementation, send via WebSocket
        }
      })
    ).pipe(
      Effect.fork
    )
    
    return {
      close: Effect.interrupt(messageHandler),
      send: (message: string) => NotificationService.sendToUser(userId, message)
    }
  }).pipe(Effect.scoped)
}

const NotificationPubSubLive = Layer.effect(
  NotificationPubSub,
  PubSub.bounded<NotificationEvent>(1000)
)

// Example real-time application
const realTimeApp = Effect.gen(function* () {
  // Simulate WebSocket connections
  const connection1 = yield* WebSocketHandler.handleConnection("user1")
  const connection2 = yield* WebSocketHandler.handleConnection("user2")
  
  // Send some notifications
  yield* NotificationService.sendToUser("user1", "Welcome to the app!")
  yield* NotificationService.broadcast("System maintenance in 10 minutes")
  yield* NotificationService.sendToUser("user2", "You have a new message")
  
  yield* Effect.sleep("2 seconds")
  
  // Clean up connections
  yield* connection1.close
  yield* connection2.close
}).pipe(
  Effect.provide(NotificationPubSubLive),
  Effect.scoped
)
```

### Testing Strategies

Comprehensive testing patterns for PubSub-based systems:

```typescript
import { Effect, PubSub, TestClock, TestContext, Queue, Ref } from "effect"

// Test utilities for PubSub
export const PubSubTestUtils = {
  // Collect all published messages for testing
  collectMessages: <A>(pubsub: PubSub.PubSub<A>) => Effect.gen(function* () {
    const messages = yield* Ref.make<A[]>([])
    const subscription = yield* pubsub.subscribe
    
    const collector = yield* Effect.forever(
      Effect.gen(function* () {
        const message = yield* subscription.take
        yield* Ref.update(messages, (msgs) => [...msgs, message])
      })
    ).pipe(
      Effect.fork
    )
    
    return {
      getMessages: () => Ref.get(messages),
      stop: () => Effect.interrupt(collector)
    }
  }).pipe(Effect.scoped),
  
  // Wait for a specific number of messages
  waitForMessages: <A>(pubsub: PubSub.PubSub<A>, count: number) => 
    Effect.gen(function* () {
      const subscription = yield* pubsub.subscribe
      const messages: A[] = []
      
      while (messages.length < count) {
        const message = yield* subscription.take
        messages.push(message)
      }
      
      return messages
    }).pipe(Effect.scoped),
    
  // Assert message ordering
  assertMessageOrder: <A>(
    actual: ReadonlyArray<A>,
    expected: ReadonlyArray<A>,
    equals?: (a: A, b: A) => boolean
  ) => Effect.gen(function* () {
    const eq = equals ?? ((a, b) => a === b)
    
    if (actual.length !== expected.length) {
      return yield* Effect.fail(
        new Error(`Expected ${expected.length} messages, got ${actual.length}`)
      )
    }
    
    for (let i = 0; i < actual.length; i++) {
      if (!eq(actual[i], expected[i])) {
        return yield* Effect.fail(
          new Error(`Message at index ${i} does not match. Expected: ${expected[i]}, Actual: ${actual[i]}`)
        )
      }
    }
  })
}

// Example test cases
const testOrderProcessing = Effect.gen(function* () {
  const orderEvents = yield* PubSub.bounded<OrderEvent>(100)
  const messageCollector = yield* PubSubTestUtils.collectMessages(orderEvents)
  
  // Test event publishing
  yield* orderEvents.publishAll([
    { type: 'OrderCreated', orderId: 'order-1', timestamp: new Date(), data: {} },
    { type: 'OrderPaid', orderId: 'order-1', timestamp: new Date(), data: {} }
  ])
  
  // Verify messages were received
  const messages = yield* messageCollector.getMessages
  
  yield* PubSubTestUtils.assertMessageOrder(
    messages,
    [
      { type: 'OrderCreated', orderId: 'order-1', timestamp: new Date(), data: {} },
      { type: 'OrderPaid', orderId: 'order-1', timestamp: new Date(), data: {} }
    ],
    (a, b) => a.type === b.type && a.orderId === b.orderId
  )
  
  console.log("✅ Order processing test passed")
}).pipe(Effect.scoped)

// Test with controlled timing
const testTimingDependentBehavior = Effect.gen(function* () {
  const events = yield* PubSub.bounded<{ timestamp: Date; data: string }>(10)
  
  // Use TestClock for controlled time
  const testClock = yield* TestClock.TestClock
  
  // Schedule events at specific times
  yield* events.publish({ timestamp: new Date(), data: "event-1" })
  
  yield* TestClock.adjust("1 second")
  yield* events.publish({ timestamp: new Date(), data: "event-2" })
  
  yield* TestClock.adjust("2 seconds")
  yield* events.publish({ timestamp: new Date(), data: "event-3" })
  
  // Collect and verify
  const messages = yield* PubSubTestUtils.waitForMessages(events, 3)
  
  console.log("✅ Timing test passed, received:", messages.length, "messages")
}).pipe(
  Effect.provide(TestContext.TestContext),
  Effect.scoped
)

// Property-based testing
const testBackpressureProperty = Effect.gen(function* () {
  const smallPubSub = yield* PubSub.bounded<number>(2)
  
  // Test that bounded PubSub applies backpressure
  const publisher = Effect.gen(function* () {
    // Try to publish more than capacity
    const results = []
    for (let i = 0; i < 5; i++) {
      const result = yield* smallPubSub.publish(i)
      results.push(result)
    }
    return results
  })
  
  // Without subscribers, should block after capacity
  const publishResults = yield* publisher.pipe(
    Effect.timeout("100 millis"),
    Effect.either
  )
  
  if (publishResults._tag === "Left") {
    console.log("✅ Backpressure test passed - publishing blocked as expected")
  } else {
    console.log("❌ Backpressure test failed - publishing should have blocked")
  }
}).pipe(Effect.scoped)
```

## Conclusion

PubSub provides **event-driven architecture**, **decoupled communication**, and **scalable message broadcasting** for building reactive Effect applications.

Key benefits:
- **Decoupling**: Publishers and subscribers don't need to know about each other, enabling flexible architectures
- **Scalability**: Multiple subscribers can receive the same messages without impacting publishers
- **Backpressure Management**: Different strategies (bounded, dropping, sliding) handle various load scenarios
- **Integration**: Seamless integration with Effect's ecosystem including Stream, Layer, and testing utilities

Use PubSub when you need to broadcast messages to multiple consumers, implement event-driven architectures, or build real-time reactive systems where components need to communicate asynchronously without tight coupling.