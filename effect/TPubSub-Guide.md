# TPubSub: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem TPubSub Solves

Traditional publish-subscribe patterns in concurrent programming often suffer from race conditions and coordination issues:

```typescript
// Traditional approach - complex event emitter management
class UnsafeEventBus<T> {
  private subscribers: Array<(value: T) => void> = []
  
  subscribe(handler: (value: T) => void) {
    this.subscribers.push(handler) // Not thread-safe!
    
    return () => {
      const index = this.subscribers.indexOf(handler)
      if (index > -1) {
        this.subscribers.splice(index, 1) // Race condition!
      }
    }
  }
  
  publish(value: T) {
    // Snapshot to avoid concurrent modification
    const snapshot = [...this.subscribers]
    snapshot.forEach(handler => {
      try {
        handler(value) // No backpressure handling!
      } catch (error) {
        // Error in one handler affects others
        console.error(error)
      }
    })
  }
}
```

This approach leads to:
- **Race Conditions** - Concurrent subscription/unsubscription operations
- **No Backpressure** - Publishers can overwhelm slow subscribers
- **Error Propagation** - Errors in one subscriber affect others
- **Memory Leaks** - Difficult to properly clean up subscriptions

### The TPubSub Solution

TPubSub provides a transactional publish-subscribe mechanism with built-in backpressure strategies:

```typescript
import { STM, TPubSub, Queue } from "effect"

// Safe publish-subscribe with TPubSub
const program = STM.gen(function* () {
  const pubsub = yield* TPubSub.bounded<string>(10)
  
  // Multiple subscribers get independent message streams
  const subscription1 = yield* TPubSub.subscribe(pubsub)
  const subscription2 = yield* TPubSub.subscribe(pubsub)
  
  // Publishing is atomic and handles backpressure
  yield* TPubSub.publish(pubsub, "Hello, World!")
  yield* TPubSub.publishAll(pubsub, ["Message 1", "Message 2"])
  
  return { subscription1, subscription2 }
})
```

### Key Concepts

**Transactional Publishing**: All publish operations are atomic within STM transactions, ensuring consistency.

**Independent Subscriptions**: Each subscriber gets their own queue with independent backpressure handling.

**Backpressure Strategies**: Built-in strategies (bounded, dropping, sliding) to handle slow subscribers.

## Basic Usage Patterns

### Pattern 1: Creating TPubSubs with Different Strategies

```typescript
import { STM, TPubSub } from "effect"

// Bounded - blocks publishers when full
const createBounded = STM.gen(function* () {
  return yield* TPubSub.bounded<string>(100)
})

// Dropping - drops new messages when full
const createDropping = STM.gen(function* () {
  return yield* TPubSub.dropping<string>(50)
})

// Sliding - drops old messages when full
const createSliding = STM.gen(function* () {
  return yield* TPubSub.sliding<string>(50)
})

// Unbounded - unlimited capacity
const createUnbounded = STM.gen(function* () {
  return yield* TPubSub.unbounded<string>()
})
```

### Pattern 2: Basic Publishing and Subscribing

```typescript
// Publishing operations
const publishOperations = (pubsub: TPubSub.TPubSub<string>) => STM.gen(function* () {
  // Single message
  yield* TPubSub.publish(pubsub, "Single message")
  
  // Multiple messages
  yield* TPubSub.publishAll(pubsub, ["Message 1", "Message 2", "Message 3"])
  
  return yield* TPubSub.size(pubsub)
})

// Subscribing and consuming
const subscribeAndConsume = (pubsub: TPubSub.TPubSub<string>) => STM.gen(function* () {
  // Create subscription
  const subscription = yield* TPubSub.subscribe(pubsub)
  
  // Subscribe with scope management
  const scopedSubscription = yield* TPubSub.subscribeScoped(pubsub)
  
  return { subscription, scopedSubscription }
})
```

### Pattern 3: State Management

```typescript
// State checking
const stateOperations = (pubsub: TPubSub.TPubSub<string>) => STM.gen(function* () {
  const capacity = yield* TPubSub.capacity(pubsub)
  const size = yield* TPubSub.size(pubsub)
  const isEmpty = yield* TPubSub.isEmpty(pubsub)
  const isFull = yield* TPubSub.isFull(pubsub)
  const isShutdown = yield* TPubSub.isShutdown(pubsub)
  
  return { capacity, size, isEmpty, isFull, isShutdown }
})

// Lifecycle management
const lifecycleOperations = (pubsub: TPubSub.TPubSub<string>) => STM.gen(function* () {
  // Shutdown pubsub
  yield* TPubSub.shutdown(pubsub)
  
  // Wait for shutdown completion
  yield* TPubSub.awaitShutdown(pubsub)
})
```

## Real-World Examples

### Example 1: Event-Driven Microservice Communication

```typescript
import { Effect, STM, TPubSub, Queue, Fiber, Layer, Context } from "effect"

interface DomainEvent {
  readonly id: string
  readonly type: string
  readonly aggregateId: string
  readonly timestamp: number
  readonly payload: unknown
  readonly version: number
}

interface EventHandler<T = unknown> {
  readonly name: string
  readonly eventTypes: readonly string[]
  readonly handle: (event: DomainEvent) => Effect.Effect<T>
  readonly errorHandler?: (event: DomainEvent, error: unknown) => Effect.Effect<void>
}

class EventBus {
  constructor(
    private pubsub: TPubSub.TPubSub<DomainEvent>,
    private handlers: Map<string, EventHandler[]>
  ) {}

  static make = (capacity: number = 1000): Effect.Effect<EventBus> =>
    STM.gen(function* () {
      const pubsub = yield* TPubSub.bounded<DomainEvent>(capacity)
      return new EventBus(pubsub, new Map())
    }).pipe(STM.commit)

  publish = (event: DomainEvent): Effect.Effect<void> =>
    STM.commit(TPubSub.publish(this.pubsub, event))

  publishBatch = (events: readonly DomainEvent[]): Effect.Effect<void> =>
    STM.commit(TPubSub.publishAll(this.pubsub, events))

  subscribe = <T>(handler: EventHandler<T>): Effect.Effect<Fiber.RuntimeFiber<never, void>> =>
    Effect.gen(function* () {
      const subscription = yield* STM.commit(TPubSub.subscribe(this.pubsub))
      
      // Register handler
      for (const eventType of handler.eventTypes) {
        const existingHandlers = this.handlers.get(eventType) || []
        this.handlers.set(eventType, [...existingHandlers, handler])
      }
      
      // Start processing fiber
      const processor = Effect.gen(function* () {
        while (true) {
          const event = yield* Queue.take(subscription)
          
          // Check if this handler should process this event
          if (handler.eventTypes.includes(event.type)) {
            yield* handler.handle(event).pipe(
              Effect.catchAll((error) => {
                if (handler.errorHandler) {
                  return handler.errorHandler(event, error)
                } else {
                  return Effect.logError(`Handler ${handler.name} failed for event ${event.id}`, error)
                }
              })
            )
          }
        }
      })
      
      return yield* Effect.fork(processor)
    })

  getStats = (): Effect.Effect<{
    totalHandlers: number
    queueSize: number
    capacity: number
    isShutdown: boolean
  }> =>
    STM.gen(function* () {
      const queueSize = yield* TPubSub.size(this.pubsub)
      const capacity = yield* TPubSub.capacity(this.pubsub)
      const isShutdown = yield* TPubSub.isShutdown(this.pubsub)
      
      const totalHandlers = Array.from(this.handlers.values())
        .reduce((sum, handlers) => sum + handlers.length, 0)
      
      return { totalHandlers, queueSize, capacity, isShutdown }
    }).pipe(STM.commit)

  shutdown = (): Effect.Effect<void> =>
    STM.commit(TPubSub.shutdown(this.pubsub))
}

// Usage example
const eventDrivenExample = Effect.gen(function* () {
  const eventBus = yield* EventBus.make(500)
  
  // Order service handler
  const orderHandler: EventHandler = {
    name: "OrderService",
    eventTypes: ["order.created", "order.cancelled"],
    handle: (event) => Effect.gen(function* () {
      yield* Effect.sleep("50 millis") // Simulate processing
      yield* Effect.log(`Order service processed: ${event.type} - ${event.id}`)
      return `Processed ${event.type}`
    }),
    errorHandler: (event, error) => Effect.logError(`Order service error for ${event.id}`, error)
  }
  
  // Inventory service handler
  const inventoryHandler: EventHandler = {
    name: "InventoryService",
    eventTypes: ["order.created", "product.updated"],
    handle: (event) => Effect.gen(function* () {
      yield* Effect.sleep("30 millis")
      yield* Effect.log(`Inventory service processed: ${event.type} - ${event.id}`)
      return `Inventory updated for ${event.type}`
    })
  }
  
  // Analytics service handler
  const analyticsHandler: EventHandler = {
    name: "AnalyticsService",
    eventTypes: ["order.created", "order.cancelled", "user.registered"],
    handle: (event) => Effect.gen(function* () {
      yield* Effect.sleep("20 millis")
      yield* Effect.log(`Analytics service processed: ${event.type} - ${event.id}`)
      return `Analytics tracked ${event.type}`
    })
  }
  
  // Subscribe handlers
  const orderFiber = yield* eventBus.subscribe(orderHandler)
  const inventoryFiber = yield* eventBus.subscribe(inventoryHandler)
  const analyticsFiber = yield* eventBus.subscribe(analyticsHandler)
  
  // Publish events
  const events: DomainEvent[] = [
    {
      id: "evt-1",
      type: "order.created",
      aggregateId: "order-123",
      timestamp: Date.now(),
      payload: { orderId: "order-123", amount: 99.99 },
      version: 1
    },
    {
      id: "evt-2",
      type: "user.registered",
      aggregateId: "user-456",
      timestamp: Date.now(),
      payload: { userId: "user-456", email: "user@example.com" },
      version: 1
    },
    {
      id: "evt-3",
      type: "order.cancelled",
      aggregateId: "order-123",
      timestamp: Date.now(),
      payload: { orderId: "order-123", reason: "customer_request" },
      version: 2
    },
    {
      id: "evt-4",
      type: "product.updated",
      aggregateId: "product-789",
      timestamp: Date.now(),
      payload: { productId: "product-789", price: 29.99 },
      version: 1
    }
  ]
  
  yield* eventBus.publishBatch(events)
  
  // Let handlers process events
  yield* Effect.sleep("2 seconds")
  
  const stats = yield* eventBus.getStats()
  
  // Cleanup
  yield* Fiber.interrupt(orderFiber)
  yield* Fiber.interrupt(inventoryFiber)
  yield* Fiber.interrupt(analyticsFiber)
  yield* eventBus.shutdown()
  
  return { eventsPublished: events.length, stats }
})
```

### Example 2: Real-Time Chat System

```typescript
import { Effect, STM, TPubSub, TRef, TMap } from "effect"

interface ChatMessage {
  readonly id: string
  readonly roomId: string
  readonly userId: string
  readonly username: string
  readonly content: string
  readonly timestamp: number
  readonly type: "message" | "join" | "leave" | "system"
}

interface ChatRoom {
  readonly id: string
  readonly name: string
  readonly participants: Set<string>
  readonly messageHistory: readonly ChatMessage[]
  readonly maxHistory: number
}

interface ChatUser {
  readonly id: string
  readonly username: string
  readonly joinedRooms: Set<string>
  readonly isOnline: boolean
}

class ChatServer {
  constructor(
    private globalPubSub: TPubSub.TPubSub<ChatMessage>,
    private rooms: TMap.TMap<string, ChatRoom>,
    private users: TMap.TMap<string, ChatUser>,
    private roomPubSubs: TMap.TMap<string, TPubSub.TPubSub<ChatMessage>>
  ) {}

  static make = (): Effect.Effect<ChatServer> =>
    STM.gen(function* () {
      const globalPubSub = yield* TPubSub.bounded<ChatMessage>(10000)
      const rooms = yield* TMap.empty<string, ChatRoom>()
      const users = yield* TMap.empty<string, ChatUser>()
      const roomPubSubs = yield* TMap.empty<string, TPubSub.TPubSub<ChatMessage>>()
      
      return new ChatServer(globalPubSub, rooms, users, roomPubSubs)
    }).pipe(STM.commit)

  createRoom = (roomId: string, roomName: string, maxHistory: number = 100): Effect.Effect<void> =>
    STM.gen(function* () {
      const room: ChatRoom = {
        id: roomId,
        name: roomName,
        participants: new Set(),
        messageHistory: [],
        maxHistory
      }
      
      const roomPubSub = yield* TPubSub.bounded<ChatMessage>(1000)
      
      yield* TMap.set(this.rooms, roomId, room)
      yield* TMap.set(this.roomPubSubs, roomId, roomPubSub)
    }).pipe(STM.commit)

  registerUser = (userId: string, username: string): Effect.Effect<void> =>
    STM.gen(function* () {
      const user: ChatUser = {
        id: userId,
        username,
        joinedRooms: new Set(),
        isOnline: true
      }
      
      yield* TMap.set(this.users, userId, user)
    }).pipe(STM.commit)

  joinRoom = (userId: string, roomId: string): Effect.Effect<Queue.Dequeue<ChatMessage> | null> =>
    STM.gen(function* () {
      const user = yield* TMap.get(this.users, userId)
      const room = yield* TMap.get(this.rooms, roomId)
      const roomPubSub = yield* TMap.get(this.roomPubSubs, roomId)
      
      if (!user || !room || !roomPubSub) {
        return null
      }
      
      // Update user's joined rooms
      const updatedUser: ChatUser = {
        ...user,
        joinedRooms: new Set([...user.joinedRooms, roomId])
      }
      yield* TMap.set(this.users, userId, updatedUser)
      
      // Update room participants
      const updatedRoom: ChatRoom = {
        ...room,
        participants: new Set([...room.participants, userId])
      }
      yield* TMap.set(this.rooms, roomId, updatedRoom)
      
      // Create subscription for user
      const subscription = yield* TPubSub.subscribe(roomPubSub)
      
      // Broadcast join message
      const joinMessage: ChatMessage = {
        id: `msg-${Date.now()}-${Math.random()}`,
        roomId,
        userId,
        username: user.username,
        content: `${user.username} joined the room`,
        timestamp: Date.now(),
        type: "join"
      }
      
      yield* this.broadcastToRoom(roomId, joinMessage)
      
      return subscription
    }).pipe(STM.commit)

  leaveRoom = (userId: string, roomId: string): Effect.Effect<void> =>
    STM.gen(function* () {
      const user = yield* TMap.get(this.users, userId)
      const room = yield* TMap.get(this.rooms, roomId)
      
      if (!user || !room) {
        return
      }
      
      // Update user's joined rooms
      const updatedJoinedRooms = new Set(user.joinedRooms)
      updatedJoinedRooms.delete(roomId)
      const updatedUser: ChatUser = {
        ...user,
        joinedRooms: updatedJoinedRooms
      }
      yield* TMap.set(this.users, userId, updatedUser)
      
      // Update room participants
      const updatedParticipants = new Set(room.participants)
      updatedParticipants.delete(userId)
      const updatedRoom: ChatRoom = {
        ...room,
        participants: updatedParticipants
      }
      yield* TMap.set(this.rooms, roomId, updatedRoom)
      
      // Broadcast leave message
      const leaveMessage: ChatMessage = {
        id: `msg-${Date.now()}-${Math.random()}`,
        roomId,
        userId,
        username: user.username,
        content: `${user.username} left the room`,
        timestamp: Date.now(),
        type: "leave"
      }
      
      yield* this.broadcastToRoom(roomId, leaveMessage)
    }).pipe(STM.commit)

  sendMessage = (userId: string, roomId: string, content: string): Effect.Effect<boolean> =>
    STM.gen(function* () {
      const user = yield* TMap.get(this.users, userId)
      const room = yield* TMap.get(this.rooms, roomId)
      
      if (!user || !room || !room.participants.has(userId)) {
        return false
      }
      
      const message: ChatMessage = {
        id: `msg-${Date.now()}-${Math.random()}`,
        roomId,
        userId,
        username: user.username,
        content,
        timestamp: Date.now(),
        type: "message"
      }
      
      yield* this.broadcastToRoom(roomId, message)
      return true
    }).pipe(STM.commit)

  private broadcastToRoom = (roomId: string, message: ChatMessage): STM.STM<void> =>
    STM.gen(function* () {
      const roomPubSub = yield* TMap.get(this.roomPubSubs, roomId)
      const room = yield* TMap.get(this.rooms, roomId)
      
      if (!roomPubSub || !room) {
        return
      }
      
      // Broadcast to room
      yield* TPubSub.publish(roomPubSub, message)
      
      // Broadcast to global feed
      yield* TPubSub.publish(this.globalPubSub, message)
      
      // Update room history
      const updatedHistory = [...room.messageHistory, message]
        .slice(-room.maxHistory) // Keep only recent messages
      
      const updatedRoom: ChatRoom = {
        ...room,
        messageHistory: updatedHistory
      }
      
      yield* TMap.set(this.rooms, roomId, updatedRoom)
    })

  getRoomHistory = (roomId: string): Effect.Effect<readonly ChatMessage[]> =>
    STM.gen(function* () {
      const room = yield* TMap.get(this.rooms, roomId)
      return room ? room.messageHistory : []
    }).pipe(STM.commit)

  getRoomStats = (roomId: string): Effect.Effect<{
    name: string
    participantCount: number
    messageCount: number
    participants: readonly string[]
  } | null> =>
    STM.gen(function* () {
      const room = yield* TMap.get(this.rooms, roomId)
      
      if (!room) {
        return null
      }
      
      return {
        name: room.name,
        participantCount: room.participants.size,
        messageCount: room.messageHistory.length,
        participants: Array.from(room.participants)
      }
    }).pipe(STM.commit)

  getGlobalSubscription = (): Effect.Effect<Queue.Dequeue<ChatMessage>> =>
    STM.commit(TPubSub.subscribe(this.globalPubSub))

  shutdown = (): Effect.Effect<void> =>
    STM.gen(function* () {
      yield* TPubSub.shutdown(this.globalPubSub)
      
      const roomPubSubEntries = yield* TMap.toArray(this.roomPubSubs)
      for (const [_, roomPubSub] of roomPubSubEntries) {
        yield* TPubSub.shutdown(roomPubSub)
      }
    }).pipe(STM.commit)
}

// Usage example
const chatSystemExample = Effect.gen(function* () {
  const chatServer = yield* ChatServer.make()
  
  // Create rooms
  yield* chatServer.createRoom("general", "General Discussion")
  yield* chatServer.createRoom("dev", "Development Talk")
  
  // Register users
  yield* chatServer.registerUser("user1", "Alice")
  yield* chatServer.registerUser("user2", "Bob")
  yield* chatServer.registerUser("user3", "Charlie")
  
  // Users join rooms
  const aliceGeneralSub = yield* chatServer.joinRoom("user1", "general")
  const bobGeneralSub = yield* chatServer.joinRoom("user2", "general")
  const bobDevSub = yield* chatServer.joinRoom("user2", "dev")
  const charlieDevSub = yield* chatServer.joinRoom("user3", "dev")
  
  // Global activity monitor
  const globalSub = yield* chatServer.getGlobalSubscription()
  const activityMonitor = Effect.gen(function* () {
    let messageCount = 0
    while (messageCount < 10) { // Monitor first 10 messages
      const message = yield* Queue.take(globalSub)
      yield* Effect.log(`Global Activity: ${message.username} in ${message.roomId}: ${message.content}`)
      messageCount++
    }
  })
  
  const activityFiber = yield* Effect.fork(activityMonitor)
  
  // Send messages
  yield* chatServer.sendMessage("user1", "general", "Hello everyone!")
  yield* chatServer.sendMessage("user2", "general", "Hi Alice!")
  yield* chatServer.sendMessage("user2", "dev", "Anyone working on the new feature?")
  yield* chatServer.sendMessage("user3", "dev", "Yes, I'm on it!")
  yield* chatServer.sendMessage("user1", "general", "Great to see everyone active")
  
  // Simulate some activity
  yield* Effect.sleep("1 second")
  
  // Charlie leaves dev room
  yield* chatServer.leaveRoom("user3", "dev")
  
  // More messages
  yield* chatServer.sendMessage("user2", "dev", "Charlie left, continuing solo")
  yield* chatServer.sendMessage("user1", "general", "Productive day!")
  
  yield* Effect.sleep("1 second")
  
  // Get room stats
  const generalStats = yield* chatServer.getRoomStats("general")
  const devStats = yield* chatServer.getRoomStats("dev")
  
  // Get message history
  const generalHistory = yield* chatServer.getRoomHistory("general")
  const devHistory = yield* chatServer.getRoomHistory("dev")
  
  // Cleanup
  yield* Fiber.interrupt(activityFiber)
  yield* chatServer.shutdown()
  
  return {
    generalStats,
    devStats,
    messagesSent: {
      general: generalHistory.filter(m => m.type === "message").length,
      dev: devHistory.filter(m => m.type === "message").length
    }
  }
})
```

### Example 3: Real-Time Stock Price Broadcasting

```typescript
import { Effect, STM, TPubSub, TRef, Random, Schedule } from "effect"

interface StockPrice {
  readonly symbol: string
  readonly price: number
  readonly timestamp: number
  readonly change: number
  readonly changePercent: number
  readonly volume: number
}

interface PriceAlert {
  readonly id: string
  readonly symbol: string
  readonly condition: "above" | "below"
  readonly threshold: number
  readonly userId: string
}

interface MarketUpdate {
  readonly type: "price" | "alert" | "news"
  readonly symbol?: string
  readonly data: unknown
  readonly timestamp: number
}

class StockMarketBroadcaster {
  constructor(
    private marketPubSub: TPubSub.TPubSub<MarketUpdate>,
    private stockPubSubs: TMap.TMap<string, TPubSub.TPubSub<StockPrice>>,
    private currentPrices: TMap.TMap<string, StockPrice>,
    private alerts: TRef.TRef<readonly PriceAlert[]>,
    private isRunning: TRef.TRef<boolean>
  ) {}

  static make = (): Effect.Effect<StockMarketBroadcaster> =>
    STM.gen(function* () {
      const marketPubSub = yield* TPubSub.bounded<MarketUpdate>(10000)
      const stockPubSubs = yield* TMap.empty<string, TPubSub.TPubSub<StockPrice>>()
      const currentPrices = yield* TMap.empty<string, StockPrice>()
      const alerts = yield* TRef.make<readonly PriceAlert[]>([])
      const isRunning = yield* TRef.make(false)
      
      return new StockMarketBroadcaster(
        marketPubSub,
        stockPubSubs,
        currentPrices,
        alerts,
        isRunning
      )
    }).pipe(STM.commit)

  addStock = (symbol: string, initialPrice: number): Effect.Effect<void> =>
    STM.gen(function* () {
      const stockPubSub = yield* TPubSub.bounded<StockPrice>(1000)
      const initialStockPrice: StockPrice = {
        symbol,
        price: initialPrice,
        timestamp: Date.now(),
        change: 0,
        changePercent: 0,
        volume: 0
      }
      
      yield* TMap.set(this.stockPubSubs, symbol, stockPubSub)
      yield* TMap.set(this.currentPrices, symbol, initialStockPrice)
    }).pipe(STM.commit)

  subscribeToStock = (symbol: string): Effect.Effect<Queue.Dequeue<StockPrice> | null> =>
    STM.gen(function* () {
      const stockPubSub = yield* TMap.get(this.stockPubSubs, symbol)
      if (!stockPubSub) {
        return null
      }
      
      return yield* TPubSub.subscribe(stockPubSub)
    }).pipe(STM.commit)

  subscribeToMarket = (): Effect.Effect<Queue.Dequeue<MarketUpdate>> =>
    STM.commit(TPubSub.subscribe(this.marketPubSub))

  private updateStockPrice = (symbol: string): Effect.Effect<void> =>
    Effect.gen(function* () {
      const randomChange = yield* Random.nextRange(-5, 5) // -5% to +5% change
      
      return yield* STM.gen(function* () {
        const currentPrice = yield* TMap.get(this.currentPrices, symbol)
        const stockPubSub = yield* TMap.get(this.stockPubSubs, symbol)
        
        if (!currentPrice || !stockPubSub) {
          return
        }
        
        const priceChange = currentPrice.price * (randomChange / 100)
        const newPrice = Math.max(0.01, currentPrice.price + priceChange)
        const volume = yield* STM.sync(() => Math.floor(Math.random() * 1000000))
        
        const updatedPrice: StockPrice = {
          symbol,
          price: Math.round(newPrice * 100) / 100, // Round to 2 decimals
          timestamp: Date.now(),
          change: Math.round(priceChange * 100) / 100,
          changePercent: Math.round((priceChange / currentPrice.price) * 10000) / 100,
          volume
        }
        
        // Update current price
        yield* TMap.set(this.currentPrices, symbol, updatedPrice)
        
        // Broadcast to stock subscribers
        yield* TPubSub.publish(stockPubSub, updatedPrice)
        
        // Broadcast to market feed
        const marketUpdate: MarketUpdate = {
          type: "price",
          symbol,
          data: updatedPrice,
          timestamp: Date.now()
        }
        yield* TPubSub.publish(this.marketPubSub, marketUpdate)
        
        // Check alerts
        yield* this.checkAlerts(updatedPrice)
      }).pipe(STM.commit)
    })

  private checkAlerts = (price: StockPrice): STM.STM<void> =>
    STM.gen(function* () {
      const currentAlerts = yield* TRef.get(this.alerts)
      const triggeredAlerts: PriceAlert[] = []
      const remainingAlerts: PriceAlert[] = []
      
      for (const alert of currentAlerts) {
        if (alert.symbol === price.symbol) {
          const isTriggered = 
            (alert.condition === "above" && price.price > alert.threshold) ||
            (alert.condition === "below" && price.price < alert.threshold)
          
          if (isTriggered) {
            triggeredAlerts.push(alert)
            
            // Broadcast alert
            const alertUpdate: MarketUpdate = {
              type: "alert",
              symbol: price.symbol,
              data: { alert, price },
              timestamp: Date.now()
            }
            yield* TPubSub.publish(this.marketPubSub, alertUpdate)
          } else {
            remainingAlerts.push(alert)
          }
        } else {
          remainingAlerts.push(alert)
        }
      }
      
      // Update alerts (remove triggered ones)
      yield* TRef.set(this.alerts, remainingAlerts)
    })

  addPriceAlert = (alert: PriceAlert): Effect.Effect<void> =>
    STM.commit(TRef.update(this.alerts, (alerts) => [...alerts, alert]))

  removeAlert = (alertId: string): Effect.Effect<boolean> =>
    STM.gen(function* () {
      const currentAlerts = yield* TRef.get(this.alerts)
      const filteredAlerts = currentAlerts.filter(alert => alert.id !== alertId)
      
      if (filteredAlerts.length < currentAlerts.length) {
        yield* TRef.set(this.alerts, filteredAlerts)
        return true
      }
      
      return false
    }).pipe(STM.commit)

  startBroadcasting = (): Effect.Effect<Fiber.RuntimeFiber<never, void>> =>
    Effect.gen(function* () {
      yield* STM.commit(TRef.set(this.isRunning, true))
      
      const broadcastLoop = Effect.gen(function* () {
        while (true) {
          const running = yield* STM.commit(TRef.get(this.isRunning))
          if (!running) break
          
          // Update all stock prices
          const symbols = yield* STM.commit(TMap.keys(this.currentPrices))
          
          yield* Effect.all(
            symbols.map(symbol => this.updateStockPrice(symbol)),
            { concurrency: "unbounded" }
          )
          
          // Wait before next update
          yield* Effect.sleep("1 second")
        }
      })
      
      return yield* Effect.fork(broadcastLoop)
    })

  stopBroadcasting = (): Effect.Effect<void> =>
    STM.commit(TRef.set(this.isRunning, false))

  getCurrentPrices = (): Effect.Effect<readonly StockPrice[]> =>
    STM.gen(function* () {
      const priceEntries = yield* TMap.toArray(this.currentPrices)
      return priceEntries.map(([_, price]) => price)
    }).pipe(STM.commit)

  getActiveAlerts = (): Effect.Effect<readonly PriceAlert[]> =>
    STM.commit(TRef.get(this.alerts))

  shutdown = (): Effect.Effect<void> =>
    STM.gen(function* () {
      yield* TRef.set(this.isRunning, false)
      yield* TPubSub.shutdown(this.marketPubSub)
      
      const stockPubSubEntries = yield* TMap.toArray(this.stockPubSubs)
      for (const [_, stockPubSub] of stockPubSubEntries) {
        yield* TPubSub.shutdown(stockPubSub)
      }
    }).pipe(STM.commit)
}

// Usage example
const stockMarketExample = Effect.gen(function* () {
  const broadcaster = yield* StockMarketBroadcaster.make()
  
  // Add stocks
  yield* broadcaster.addStock("AAPL", 150.00)
  yield* broadcaster.addStock("GOOGL", 2800.00)
  yield* broadcaster.addStock("MSFT", 300.00)
  yield* broadcaster.addStock("TSLA", 800.00)
  
  // Set up subscriptions
  const marketSub = yield* broadcaster.subscribeToMarket()
  const aaplSub = yield* broadcaster.subscribeToStock("AAPL")
  const googleSub = yield* broadcaster.subscribeToStock("GOOGL")
  
  // Set up price alerts
  yield* broadcaster.addPriceAlert({
    id: "alert-1",
    symbol: "AAPL",
    condition: "above",
    threshold: 155.00,
    userId: "user1"
  })
  
  yield* broadcaster.addPriceAlert({
    id: "alert-2",
    symbol: "TSLA",
    condition: "below",
    threshold: 750.00,
    userId: "user2"
  })
  
  // Market activity monitor
  const marketMonitor = Effect.gen(function* () {
    let updateCount = 0
    while (updateCount < 20) { // Monitor first 20 updates
      const update = yield* Queue.take(marketSub)
      
      if (update.type === "price") {
        const price = update.data as StockPrice
        yield* Effect.log(`Price Update: ${price.symbol} = $${price.price} (${price.changePercent > 0 ? '+' : ''}${price.changePercent}%)`)
      } else if (update.type === "alert") {
        const alertData = update.data as { alert: PriceAlert; price: StockPrice }
        yield* Effect.log(`ALERT: ${alertData.alert.symbol} ${alertData.alert.condition} $${alertData.alert.threshold} - Current: $${alertData.price.price}`)
      }
      
      updateCount++
    }
  })
  
  // AAPL specific monitor
  const aaplMonitor = Effect.gen(function* () {
    if (!aaplSub) return
    
    let priceCount = 0
    while (priceCount < 5) { // Monitor first 5 AAPL updates
      const price = yield* Queue.take(aaplSub)
      yield* Effect.log(`AAPL Specific: $${price.price} (Volume: ${price.volume})`)
      priceCount++
    }
  })
  
  // Start monitoring
  const marketFiber = yield* Effect.fork(marketMonitor)
  const aaplFiber = yield* Effect.fork(aaplMonitor)
  
  // Start broadcasting
  const broadcastFiber = yield* broadcaster.startBroadcasting()
  
  // Let it run for a while
  yield* Effect.sleep("10 seconds")
  
  // Get final state
  const finalPrices = yield* broadcaster.getCurrentPrices()
  const activeAlerts = yield* broadcaster.getActiveAlerts()
  
  // Stop broadcasting
  yield* broadcaster.stopBroadcasting()
  
  // Cleanup
  yield* Fiber.interrupt(marketFiber)
  yield* Fiber.interrupt(aaplFiber)
  yield* Fiber.interrupt(broadcastFiber)
  yield* broadcaster.shutdown()
  
  return {
    finalPrices: finalPrices.map(p => ({
      symbol: p.symbol,
      price: p.price,
      change: p.changePercent
    })),
    remainingAlerts: activeAlerts.length,
    totalStocks: finalPrices.length
  }
})
```

## Advanced Features Deep Dive

### Feature 1: Backpressure Strategy Comparison

Different TPubSub strategies handle backpressure differently:

```typescript
const backpressureComparison = Effect.gen(function* () {
  // Bounded - blocks publishers when full
  const boundedPubSub = yield* STM.commit(TPubSub.bounded<number>(3))
  
  // Dropping - drops new messages when full
  const droppingPubSub = yield* STM.commit(TPubSub.dropping<number>(3))
  
  // Sliding - drops old messages when full
  const slidingPubSub = yield* STM.commit(TPubSub.sliding<number>(3))
  
  // Test each strategy
  const testStrategy = (pubsub: TPubSub.TPubSub<number>, name: string) =>
    Effect.gen(function* () {
      const subscription = yield* STM.commit(TPubSub.subscribe(pubsub))
      
      // Fill the queue beyond capacity
      const publishResults = yield* Effect.all([
        STM.commit(TPubSub.publish(pubsub, 1)),
        STM.commit(TPubSub.publish(pubsub, 2)),
        STM.commit(TPubSub.publish(pubsub, 3)),
        STM.commit(TPubSub.publish(pubsub, 4)), // This will behave differently per strategy
        STM.commit(TPubSub.publish(pubsub, 5))
      ], { concurrency: "unbounded" }).pipe(Effect.either)
      
      // Read all available messages
      const messages: number[] = []
      while (true) {
        const message = yield* Queue.poll(subscription).pipe(Effect.either)
        if (message._tag === "Left") break
        messages.push(message.right)
      }
      
      return { strategy: name, messages, publishResults }
    })
  
  const results = yield* Effect.all([
    testStrategy(boundedPubSub, "bounded"),
    testStrategy(droppingPubSub, "dropping"),
    testStrategy(slidingPubSub, "sliding")
  ])
  
  return results
})
```

### Feature 2: Subscription Management and Cleanup

```typescript
import { Scope } from "effect"

const subscriptionManagement = Effect.gen(function* () {
  const pubsub = yield* STM.commit(TPubSub.bounded<string>(100))
  
  // Manual subscription management
  const manualSubscription = yield* STM.commit(TPubSub.subscribe(pubsub))
  
  // Scoped subscription (automatically cleaned up)
  const result = yield* Effect.scoped(
    Effect.gen(function* () {
      const scopedSubscription = yield* STM.commit(TPubSub.subscribeScoped(pubsub))
      
      // Publish messages
      yield* STM.commit(TPubSub.publishAll(pubsub, ["msg1", "msg2", "msg3"]))
      
      // Read messages from scoped subscription
      const messages: string[] = []
      for (let i = 0; i < 3; i++) {
        const message = yield* Queue.take(scopedSubscription)
        messages.push(message)
      }
      
      return messages
      // Scoped subscription automatically cleaned up here
    })
  )
  
  // Manual subscription still active, needs explicit cleanup
  const manualMessages: string[] = []
  for (let i = 0; i < 3; i++) {
    const message = yield* Queue.take(manualSubscription)
    manualMessages.push(message)
  }
  
  return { scopedResult: result, manualResult: manualMessages }
})
```

### Feature 3: Multi-Level Publishing Hierarchies

```typescript
const hierarchicalPublishing = STM.gen(function* () {
  // Global -> Regional -> Local hierarchy
  const globalPubSub = yield* TPubSub.bounded<{ level: string; message: string }>(1000)
  const regionalPubSubs = yield* TMap.empty<string, TPubSub.TPubSub<{ level: string; message: string }>>()
  const localPubSubs = yield* TMap.empty<string, TPubSub.TPubSub<{ level: string; message: string }>>()
  
  // Create regional pubsubs
  const regions = ["north", "south", "east", "west"]
  for (const region of regions) {
    const regionalPubSub = yield* TPubSub.bounded<{ level: string; message: string }>(500)
    yield* TMap.set(regionalPubSubs, region, regionalPubSub)
    
    // Create local pubsubs for each region
    for (let i = 1; i <= 3; i++) {
      const localKey = `${region}-${i}`
      const localPubSub = yield* TPubSub.bounded<{ level: string; message: string }>(100)
      yield* TMap.set(localPubSubs, localKey, localPubSub)
    }
  }
  
  // Publish to global (cascades down)
  const publishGlobal = (message: string) => STM.gen(function* () {
    const globalMessage = { level: "global", message }
    yield* TPubSub.publish(globalPubSub, globalMessage)
    
    // Cascade to regional
    const regionalEntries = yield* TMap.toArray(regionalPubSubs)
    for (const [region, regionalPubSub] of regionalEntries) {
      const regionalMessage = { level: `regional-${region}`, message }
      yield* TPubSub.publish(regionalPubSub, regionalMessage)
    }
    
    // Cascade to local
    const localEntries = yield* TMap.toArray(localPubSubs)
    for (const [local, localPubSub] of localEntries) {
      const localMessage = { level: `local-${local}`, message }
      yield* TPubSub.publish(localPubSub, localMessage)
    }
  })
  
  // Test hierarchical publishing
  yield* publishGlobal("System maintenance in 1 hour")
  
  return { 
    globalSize: yield* TPubSub.size(globalPubSub),
    regionalCount: yield* TMap.size(regionalPubSubs),
    localCount: yield* TMap.size(localPubSubs)
  }
})
```

## Practical Patterns & Best Practices

### Pattern 1: Message Filtering and Routing

```typescript
// Helper for filtered subscriptions
const createFilteredSubscription = <T>(
  pubsub: TPubSub.TPubSub<T>,
  filter: (message: T) => boolean
): Effect.Effect<Queue.Dequeue<T>> =>
  Effect.gen(function* () {
    const subscription = yield* STM.commit(TPubSub.subscribe(pubsub))
    const filteredQueue = yield* Queue.unbounded<T>()
    
    // Background fiber that filters messages
    const filterFiber = Effect.gen(function* () {
      while (true) {
        const message = yield* Queue.take(subscription)
        if (filter(message)) {
          yield* Queue.offer(filteredQueue, message)
        }
      }
    })
    
    yield* Effect.fork(filterFiber)
    return filteredQueue
  })

// Usage
const filteringExample = Effect.gen(function* () {
  const pubsub = yield* STM.commit(TPubSub.bounded<{ priority: number; content: string }>(100))
  
  // High priority subscription
  const highPriorityQueue = yield* createFilteredSubscription(
    pubsub,
    (msg) => msg.priority >= 8
  )
  
  // Low priority subscription
  const lowPriorityQueue = yield* createFilteredSubscription(
    pubsub,
    (msg) => msg.priority < 5
  )
  
  // Publish mixed priority messages
  yield* STM.commit(TPubSub.publishAll(pubsub, [
    { priority: 10, content: "Critical alert" },
    { priority: 3, content: "Debug info" },
    { priority: 8, content: "Important update" },
    { priority: 1, content: "Trace log" }
  ]))
  
  return { highPriorityQueue, lowPriorityQueue }
})
```

### Pattern 2: Message Replay and History

```typescript
// Helper for replay capability
const createReplayablePubSub = <T>(
  capacity: number,
  historySize: number
): STM.STM<{
  pubsub: TPubSub.TPubSub<T>
  publish: (message: T) => STM.STM<void>
  subscribeWithHistory: () => STM.STM<Queue.Dequeue<T>>
  getHistory: () => STM.STM<readonly T[]>
}> =>
  STM.gen(function* () {
    const pubsub = yield* TPubSub.bounded<T>(capacity)
    const history = yield* TRef.make<readonly T[]>([])
    
    const publish = (message: T) => STM.gen(function* () {
      yield* TPubSub.publish(pubsub, message)
      yield* TRef.update(history, (h) => [...h, message].slice(-historySize))
    })
    
    const subscribeWithHistory = () => STM.gen(function* () {
      const subscription = yield* TPubSub.subscribe(pubsub)
      const currentHistory = yield* TRef.get(history)
      
      // Send history first
      for (const message of currentHistory) {
        yield* Queue.offer(subscription, message)
      }
      
      return subscription
    })
    
    const getHistory = () => TRef.get(history)
    
    return { pubsub, publish, subscribeWithHistory, getHistory }
  })
```

### Pattern 3: Fan-out with Load Balancing

```typescript
// Helper for load-balanced fan-out
const createLoadBalancedFanOut = <T>(
  sources: readonly TPubSub.TPubSub<T>[],
  workerCount: number
): Effect.Effect<{
  workers: readonly Queue.Dequeue<T>[]
  distributor: Fiber.RuntimeFiber<never, void>
}> =>
  Effect.gen(function* () {
    // Create worker queues
    const workers = yield* Effect.all(
      Array.from({ length: workerCount }, () => Queue.bounded<T>(100))
    )
    
    // Subscribe to all sources
    const subscriptions = yield* Effect.all(
      sources.map(source => STM.commit(TPubSub.subscribe(source)))
    )
    
    // Round-robin distributor
    const distributorFiber = Effect.gen(function* () {
      let currentWorker = 0
      
      while (true) {
        // Read from any source
        for (const subscription of subscriptions) {
          const message = yield* Queue.poll(subscription).pipe(Effect.option)
          
          if (message._tag === "Some") {
            // Distribute to next worker
            yield* Queue.offer(workers[currentWorker], message.value)
            currentWorker = (currentWorker + 1) % workerCount
          }
        }
        
        yield* Effect.sleep("1 millis")
      }
    })
    
    const distributor = yield* Effect.fork(distributorFiber)
    
    return { workers, distributor }
  })
```

## Integration Examples

### Integration with Effect Resource Management

```typescript
import { Resource } from "effect"

const resourceManagedPubSub = Resource.make(
  // Acquire: Create and setup pubsub
  Effect.gen(function* () {
    const pubsub = yield* STM.commit(TPubSub.bounded<string>(1000))
    
    // Setup any background processes
    const healthCheck = Effect.gen(function* () {
      while (true) {
        const size = yield* STM.commit(TPubSub.size(pubsub))
        if (size > 800) {
          yield* Effect.logWarning(`PubSub near capacity: ${size}/1000`)
        }
        yield* Effect.sleep("10 seconds")
      }
    })
    
    const healthFiber = yield* Effect.fork(healthCheck)
    
    return { pubsub, healthFiber }
  }),
  
  // Release: Cleanup resources
  ({ pubsub, healthFiber }) => Effect.gen(function* () {
    yield* Fiber.interrupt(healthFiber)
    yield* STM.commit(TPubSub.shutdown(pubsub))
  })
)

const resourceExample = Effect.gen(function* () {
  return yield* Resource.use(resourceManagedPubSub, ({ pubsub }) =>
    Effect.gen(function* () {
      // Use pubsub safely
      yield* STM.commit(TPubSub.publish(pubsub, "Hello"))
      return yield* STM.commit(TPubSub.size(pubsub))
    })
  )
  // Resource automatically cleaned up here
})
```

### Testing Strategies

```typescript
import { TestClock, TestContext } from "effect"

// Test utility for pubsub behavior
const testPubSubBehavior = <T>(
  createPubSub: () => Effect.Effect<TPubSub.TPubSub<T>>,
  messages: readonly T[],
  subscriberCount: number
) =>
  Effect.gen(function* () {
    const pubsub = yield* createPubSub()
    
    // Create multiple subscribers
    const subscriptions = yield* Effect.all(
      Array.from({ length: subscriberCount }, () =>
        STM.commit(TPubSub.subscribe(pubsub))
      )
    )
    
    // Publish messages
    yield* STM.commit(TPubSub.publishAll(pubsub, messages))
    
    // Collect messages from all subscribers
    const results = yield* Effect.all(
      subscriptions.map(sub =>
        Effect.gen(function* () {
          const received: T[] = []
          for (let i = 0; i < messages.length; i++) {
            const message = yield* Queue.take(sub)
            received.push(message)
          }
          return received
        })
      )
    )
    
    return {
      publishedCount: messages.length,
      subscriberResults: results,
      allReceived: results.every(r => r.length === messages.length)
    }
  })

// Test concurrent publishing and subscribing
const testConcurrentOperations = Effect.gen(function* () {
  const pubsub = yield* STM.commit(TPubSub.bounded<number>(100))
  
  // Concurrent publishers
  const publishers = Array.from({ length: 5 }, (_, i) =>
    Effect.gen(function* () {
      const messages = Array.from({ length: 10 }, (_, j) => i * 10 + j)
      yield* STM.commit(TPubSub.publishAll(pubsub, messages))
      return messages.length
    })
  )
  
  // Concurrent subscribers
  const subscribers = Array.from({ length: 3 }, () =>
    Effect.gen(function* () {
      const subscription = yield* STM.commit(TPubSub.subscribe(pubsub))
      const received: number[] = []
      
      // Try to read 50 messages (total published)
      for (let i = 0; i < 50; i++) {
        const message = yield* Queue.take(subscription).pipe(
          Effect.timeout("1 second"),
          Effect.option
        )
        if (message._tag === "Some") {
          received.push(message.value)
        } else {
          break
        }
      }
      
      return received.length
    })
  )
  
  // Run publishers and subscribers concurrently
  const [publishResults, subscribeResults] = yield* Effect.all([
    Effect.all(publishers, { concurrency: "unbounded" }),
    Effect.all(subscribers, { concurrency: "unbounded" })
  ])
  
  return {
    totalPublished: publishResults.reduce((sum, count) => sum + count, 0),
    subscriberCounts: subscribeResults
  }
}).pipe(
  Effect.provide(TestContext.TestContext)
)
```

## Conclusion

TPubSub provides thread-safe, transactional publish-subscribe messaging with built-in backpressure strategies for event-driven architectures, real-time systems, and distributed communication.

Key benefits:
- **Atomic Publishing**: All publish operations are atomic within STM transactions
- **Independent Subscriptions**: Each subscriber gets their own queue with independent backpressure
- **Multiple Strategies**: Built-in backpressure strategies (bounded, dropping, sliding, unbounded)
- **Composability**: Seamlessly integrates with other STM data structures and Effect constructs

TPubSub is ideal for scenarios requiring publish-subscribe patterns, event-driven architectures, real-time broadcasting, and distributed system communication where consistency and backpressure handling are critical.