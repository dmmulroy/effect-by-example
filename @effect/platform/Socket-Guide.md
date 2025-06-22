# Socket: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem Socket Solves

Building network applications with traditional approaches leads to complex, error-prone code:

```typescript
import * as net from "net"
import { EventEmitter } from "events"

// Traditional approach - fragile error handling and resource management
class TcpServer extends EventEmitter {
  private server?: net.Server
  private connections = new Set<net.Socket>()

  async start(port: number): Promise<void> {
    return new Promise((resolve, reject) => {
      this.server = net.createServer((socket) => {
        this.connections.add(socket)
        
        socket.on('data', (data) => {
          // Manual protocol parsing
          try {
            const message = JSON.parse(data.toString())
            this.handleMessage(socket, message)
          } catch (error) {
            socket.destroy(error)
          }
        })
        
        socket.on('error', (error) => {
          // Error handling scattered throughout
          console.error('Socket error:', error)
          this.connections.delete(socket)
        })
        
        socket.on('close', () => {
          this.connections.delete(socket)
        })
      })
      
      this.server.listen(port, () => resolve())
      this.server.on('error', reject)
    })
  }
  
  private handleMessage(socket: net.Socket, message: any) {
    // Unstructured message handling
    // No type safety or error boundaries
  }
}
```

This approach leads to:
- **Resource Leaks** - Connections not properly cleaned up
- **Scattered Error Handling** - Error logic mixed with business logic  
- **Poor Composability** - Hard to combine with other async operations
- **Type Unsafe** - No guarantees about message structure
- **Testing Complexity** - Difficult to mock and test network behavior

### The Socket Solution

Effect's Socket module provides type-safe, composable network programming with automatic resource management:

```typescript
import { NodeSocket } from "@effect/platform-node"
import { Socket } from "@effect/platform"
import { Effect, Layer, Schedule } from "effect"

// Effect solution - clean, type-safe, composable
const createTcpServer = (port: number) => 
  Effect.gen(function* () {
    const server = yield* NodeSocketServer.makeNet({ port })
    
    yield* server.run((socket) =>
      Effect.gen(function* () {
        const channel = Socket.toChannel(socket)
        yield* handleConnection(channel)
      }).pipe(
        Effect.catchAll((error) => 
          Effect.logError(`Connection error: ${error}`)
        ),
        Effect.scoped
      )
    )
  }).pipe(
    Effect.provide(NodeSocket.layerNet),
    Effect.withSpan("tcp.server", { attributes: { port } })
  )
```

### Key Concepts

**Socket**: A typed connection abstraction that works across platforms (Node.js, Browser, Bun)

**Channel Integration**: Sockets convert to Effect Channels for stream-based processing

**Resource Management**: Automatic cleanup using Effect's Scope system

**Error Recovery**: Structured error handling with typed error channels

## Basic Usage Patterns

### Pattern 1: WebSocket Client Connection

```typescript
import { Socket } from "@effect/platform"
import { Effect, Schedule } from "effect"

// WebSocket client with automatic reconnection
const createWebSocketClient = (url: string) =>
  Effect.gen(function* () {
    const socket = yield* Socket.makeWebSocket(url, {
      closeCodeIsError: (code) => code !== 1000,
      openTimeout: "10 seconds"
    })
    
    return socket
  }).pipe(
    Effect.retry(Schedule.exponential("1 second")),
    Effect.provide(Socket.layerWebSocketConstructorGlobal)
  )
```

### Pattern 2: TCP Client Connection

```typescript
import { NodeSocket } from "@effect/platform-node"
import { Socket } from "@effect/platform"
import { Effect } from "effect"

// TCP client connection
const createTcpClient = (host: string, port: number) =>
  Effect.gen(function* () {
    const socket = yield* NodeSocket.makeNet({ host, port })
    const channel = Socket.toChannel(socket)
    return { socket, channel }
  }).pipe(
    Effect.provide(NodeSocket.layerNet),
    Effect.withSpan("tcp.client.connect", { 
      attributes: { host, port } 
    })
  )
```

### Pattern 3: Message Handling with Channels

```typescript
import { Channel, Chunk, Effect } from "effect"
import { Socket } from "@effect/platform"

// Type-safe message processing
interface Message {
  readonly type: string
  readonly payload: unknown
}

const handleMessages = <E>(
  channel: Channel.Channel<
    Chunk.Chunk<Uint8Array>,
    Chunk.Chunk<Uint8Array | string | Socket.CloseEvent>,
    Socket.SocketError | E,
    E,
    void,
    unknown
  >
) =>
  Effect.gen(function* () {
    yield* Channel.runForEach(channel, (chunk) =>
      Effect.gen(function* () {
        for (const data of chunk) {
          if (Socket.isCloseEvent(data)) {
            yield* Effect.logInfo(`Connection closed: ${data.code}`)
            break
          }
          
          if (typeof data === "string") {
            const message = yield* parseMessage(data)
            yield* processMessage(message)
          }
        }
      })
    )
  })

const parseMessage = (data: string): Effect.Effect<Message, Error> =>
  Effect.try(() => JSON.parse(data) as Message)

const processMessage = (message: Message): Effect.Effect<void> =>
  Effect.logInfo(`Processing message: ${message.type}`)
```

## Real-World Examples

### Example 1: Real-Time Chat Server

A production-ready chat server handling multiple clients with rooms and message broadcasting:

```typescript
import { NodeSocket, NodeSocketServer } from "@effect/platform-node"
import { Socket, SocketServer } from "@effect/platform"
import { Effect, Layer, Ref, HashMap, Array as Arr, pipe } from "effect"

// Domain models
interface User {
  readonly id: string
  readonly name: string
  readonly socket: Socket.Socket
}

interface ChatRoom {
  readonly id: string
  readonly name: string
  readonly users: HashMap.HashMap<string, User>
}

interface ChatMessage {
  readonly type: "join" | "leave" | "message" | "user_list"
  readonly userId?: string
  readonly roomId?: string
  readonly content?: string
  readonly users?: ReadonlyArray<string>
}

// Chat server service
class ChatService extends Effect.Service<ChatService>()("ChatService", {
  succeed: Effect.gen(function* () {
    const rooms = yield* Ref.make(HashMap.empty<string, ChatRoom>())
    
    const addUserToRoom = (roomId: string, user: User) =>
      Effect.gen(function* () {
        yield* Ref.update(rooms, HashMap.modify(roomId, (room) => ({
          ...room,
          users: HashMap.set(room.users, user.id, user)
        })))
        
        // Broadcast user joined
        yield* broadcastToRoom(roomId, {
          type: "join",
          userId: user.id,
          content: `${user.name} joined the room`
        })
        
        // Send user list to new user
        const currentRoom = yield* Ref.get(rooms).pipe(
          Effect.map(HashMap.get(roomId)),
          Effect.flatten
        )
        
        const userList = pipe(
          currentRoom.users,
          HashMap.values,
          Arr.fromIterable,
          Arr.map(u => u.name)
        )
        
        yield* sendToUser(user, {
          type: "user_list",
          users: userList
        })
      })
    
    const removeUserFromRoom = (roomId: string, userId: string) =>
      Effect.gen(function* () {
        const userName = yield* Ref.get(rooms).pipe(
          Effect.map(HashMap.get(roomId)),
          Effect.flatten,
          Effect.map(room => HashMap.get(room.users, userId)),
          Effect.flatten,
          Effect.map(user => user.name)
        )
        
        yield* Ref.update(rooms, HashMap.modify(roomId, (room) => ({
          ...room,
          users: HashMap.remove(room.users, userId)
        })))
        
        yield* broadcastToRoom(roomId, {
          type: "leave",
          userId,
          content: `${userName} left the room`
        })
      })
    
    const broadcastToRoom = (roomId: string, message: ChatMessage) =>
      Effect.gen(function* () {
        const room = yield* Ref.get(rooms).pipe(
          Effect.map(HashMap.get(roomId)),
          Effect.flatten
        )
        
        const users = pipe(
          room.users,
          HashMap.values,
          Arr.fromIterable
        )
        
        yield* Effect.forEach(users, (user) =>
          sendToUser(user, message).pipe(
            Effect.catchAll((error) =>
              Effect.logError(`Failed to send to user ${user.id}: ${error}`)
            )
          ),
          { concurrency: "unbounded" }
        )
      })
    
    const sendToUser = (user: User, message: ChatMessage) =>
      Effect.gen(function* () {
        const channel = Socket.toChannel(user.socket)
        const data = JSON.stringify(message)
        
        yield* Channel.writeChunk(channel, Chunk.of(data))
      })
    
    const handleMessage = (user: User, message: ChatMessage) =>
      Effect.gen(function* () {
        switch (message.type) {
          case "message":
            if (message.roomId && message.content) {
              yield* broadcastToRoom(message.roomId, {
                type: "message",
                userId: user.id,
                content: `${user.name}: ${message.content}`
              })
            }
            break
          default:
            yield* Effect.logWarning(`Unknown message type: ${message.type}`)
        }
      })
    
    return {
      addUserToRoom,
      removeUserFromRoom,
      handleMessage
    } as const
  })
}) {}

// Connection handler
const handleConnection = (socket: Socket.Socket) =>
  Effect.gen(function* () {
    const chatService = yield* ChatService
    const userId = yield* Effect.sync(() => crypto.randomUUID())
    
    // Simple auth - in production, implement proper authentication
    const userName = `User_${userId.slice(0, 8)}`
    const user: User = { id: userId, name: userName, socket }
    
    // Add to default room
    const defaultRoomId = "general"
    yield* chatService.addUserToRoom(defaultRoomId, user)
    
    // Handle incoming messages
    const channel = Socket.toChannel(socket)
    
    yield* Channel.runForEach(channel, (chunk) =>
      Effect.gen(function* () {
        for (const data of chunk) {
          if (Socket.isCloseEvent(data)) {
            yield* chatService.removeUserFromRoom(defaultRoomId, userId)
            break
          }
          
          if (typeof data === "string") {
            const message = yield* Effect.try(() => 
              JSON.parse(data) as ChatMessage
            ).pipe(
              Effect.catchAll(() => 
                Effect.logWarning(`Invalid message format: ${data}`)
              )
            )
            
            if (message) {
              yield* chatService.handleMessage(user, {
                ...message,
                roomId: defaultRoomId
              })
            }
          }
        }
      })
    )
  }).pipe(
    Effect.scoped,
    Effect.catchAll((error) =>
      Effect.logError(`Connection error: ${error}`)
    )
  )

// Server setup
const createChatServer = (port: number) =>
  Effect.gen(function* () {
    const server = yield* NodeSocketServer.makeNet({ port })
    
    yield* Effect.logInfo(`Chat server starting on port ${port}`)
    
    yield* server.run(handleConnection)
  }).pipe(
    Effect.provide(ChatService.Default),
    Effect.provide(NodeSocket.layerNet),
    Effect.withSpan("chat.server")
  )

// Usage
const program = createChatServer(8080)

Effect.runFork(program)
```

### Example 2: Load Balancer Health Check System

A sophisticated health checking system for monitoring service availability:

```typescript
import { NodeSocket } from "@effect/platform-node"
import { Socket } from "@effect/platform"
import { Effect, Schedule, Ref, HashMap, Duration } from "effect"

// Health check models
interface ServiceEndpoint {
  readonly id: string
  readonly host: string
  readonly port: number
  readonly protocol: "tcp" | "http"
  readonly timeout: Duration.Duration
}

interface HealthStatus {
  readonly endpoint: ServiceEndpoint
  readonly isHealthy: boolean
  readonly lastCheck: Date
  readonly responseTime: Duration.Duration
  readonly consecutiveFailures: number
}

class HealthChecker extends Effect.Service<HealthChecker>()("HealthChecker", {
  succeed: Effect.gen(function* () {
    const statusMap = yield* Ref.make(HashMap.empty<string, HealthStatus>())
    
    const checkTcpHealth = (endpoint: ServiceEndpoint) =>
      Effect.gen(function* () {
        const startTime = yield* Effect.clockWith(clock => clock.currentTimeMillis)
        
        const socket = yield* NodeSocket.makeNet({
          host: endpoint.host,
          port: endpoint.port,
          timeout: Duration.toMillis(endpoint.timeout)
        }).pipe(
          Effect.timeout(endpoint.timeout)
        )
        
        const endTime = yield* Effect.clockWith(clock => clock.currentTimeMillis)
        const responseTime = Duration.millis(endTime - startTime)
        
        // Close connection immediately for health check
        yield* Effect.sync(() => socket)
        
        return responseTime
      })
    
    const performHealthCheck = (endpoint: ServiceEndpoint) =>
      Effect.gen(function* () {
        const currentStatus = yield* Ref.get(statusMap).pipe(
          Effect.map(HashMap.get(endpoint.id))
        )
        
        const responseTime = yield* checkTcpHealth(endpoint).pipe(
          Effect.either
        )
        
        const newStatus: HealthStatus = {
          endpoint,
          isHealthy: responseTime._tag === "Right",
          lastCheck: new Date(),
          responseTime: responseTime._tag === "Right" 
            ? responseTime.right 
            : Duration.seconds(0),
          consecutiveFailures: responseTime._tag === "Left"
            ? (currentStatus?.consecutiveFailures ?? 0) + 1
            : 0
        }
        
        yield* Ref.update(statusMap, HashMap.set(endpoint.id, newStatus))
        
        // Log health status changes
        const wasHealthy = currentStatus?.isHealthy ?? true
        if (wasHealthy !== newStatus.isHealthy) {
          yield* Effect.logInfo(
            `Health status changed for ${endpoint.id}: ${newStatus.isHealthy ? 'HEALTHY' : 'UNHEALTHY'}`
          )
        }
        
        return newStatus
      }).pipe(
        Effect.catchAll((error) =>
          Effect.gen(function* () {
            yield* Effect.logError(`Health check failed for ${endpoint.id}: ${error}`)
            
            const failedStatus: HealthStatus = {
              endpoint,
              isHealthy: false,
              lastCheck: new Date(),
              responseTime: Duration.seconds(0),
              consecutiveFailures: (currentStatus?.consecutiveFailures ?? 0) + 1
            }
            
            yield* Ref.update(statusMap, HashMap.set(endpoint.id, failedStatus))
            return failedStatus
          })
        )
      )
    
    const startMonitoring = (endpoints: ReadonlyArray<ServiceEndpoint>) =>
      Effect.gen(function* () {
        yield* Effect.logInfo(`Starting health monitoring for ${endpoints.length} endpoints`)
        
        yield* Effect.forEach(
          endpoints,
          (endpoint) =>
            performHealthCheck(endpoint).pipe(
              Effect.repeat(Schedule.fixed("30 seconds")),
              Effect.forkDaemon
            ),
          { concurrency: "unbounded" }
        )
      })
    
    const getHealthStatus = (endpointId: string) =>
      Ref.get(statusMap).pipe(
        Effect.map(HashMap.get(endpointId))
      )
    
    const getAllHealthStatuses = () =>
      Ref.get(statusMap).pipe(
        Effect.map(HashMap.values),
        Effect.map(Array.from)
      )
    
    const getHealthyEndpoints = () =>
      getAllHealthStatuses().pipe(
        Effect.map(statuses => statuses.filter(s => s.isHealthy))
      )
    
    return {
      performHealthCheck,
      startMonitoring,
      getHealthStatus,
      getAllHealthStatuses,
      getHealthyEndpoints
    } as const
  })
}) {}

// Health check server for exposing status
const createHealthStatusServer = (port: number) =>
  Effect.gen(function* () {
    const healthChecker = yield* HealthChecker
    const server = yield* NodeSocketServer.makeNet({ port })
    
    yield* server.run((socket) =>
      Effect.gen(function* () {
        const channel = Socket.toChannel(socket)
        
        // Simple HTTP-like protocol for status endpoint
        yield* Channel.runForEach(channel, (chunk) =>
          Effect.gen(function* () {
            for (const data of chunk) {
              if (typeof data === "string" && data.includes("GET /health")) {
                const statuses = yield* healthChecker.getAllHealthStatuses()
                const response = JSON.stringify(statuses, null, 2)
                
                const httpResponse = `HTTP/1.1 200 OK\r\n` +
                  `Content-Type: application/json\r\n` +
                  `Content-Length: ${response.length}\r\n` +
                  `\r\n${response}`
                
                yield* Channel.writeChunk(channel, Chunk.of(httpResponse))
                break
              }
            }
          })
        )
      }).pipe(
        Effect.scoped,
        Effect.catchAll((error) =>
          Effect.logError(`Health status server error: ${error}`)
        )
      )
    )
  }).pipe(
    Effect.provide(HealthChecker.Default),
    Effect.provide(NodeSocket.layerNet)
  )

// Usage example
const program = Effect.gen(function* () {
  const healthChecker = yield* HealthChecker
  
  const endpoints: ReadonlyArray<ServiceEndpoint> = [
    {
      id: "api-server-1",
      host: "localhost",
      port: 3000,
      protocol: "tcp",
      timeout: Duration.seconds(5)
    },
    {
      id: "database",
      host: "localhost", 
      port: 5432,
      protocol: "tcp",
      timeout: Duration.seconds(3)
    },
    {
      id: "redis",
      host: "localhost",
      port: 6379, 
      protocol: "tcp",
      timeout: Duration.seconds(2)
    }
  ]
  
  // Start monitoring
  yield* healthChecker.startMonitoring(endpoints)
  
  // Start status server
  yield* createHealthStatusServer(8090)
}).pipe(
  Effect.provide(HealthChecker.Default),
  Effect.provide(NodeSocket.layerNet)
)

Effect.runFork(program)
```

### Example 3: Microservice Message Bus

A message bus for inter-service communication with guaranteed delivery:

```typescript
import { NodeSocket, NodeSocketServer } from "@effect/platform-node"
import { Socket } from "@effect/platform"
import { Effect, Queue, Ref, HashMap, Duration, Schedule, Chunk } from "effect"

// Message models
interface ServiceMessage {
  readonly id: string
  readonly from: string
  readonly to: string
  readonly type: string
  readonly payload: unknown
  readonly timestamp: number
  readonly retryCount?: number
}

interface ServiceConnection {
  readonly serviceId: string
  readonly socket: Socket.Socket
  readonly lastHeartbeat: number
}

class MessageBus extends Effect.Service<MessageBus>()("MessageBus", {
  succeed: Effect.gen(function* () {
    const connections = yield* Ref.make(HashMap.empty<string, ServiceConnection>())
    const messageQueue = yield* Queue.unbounded<ServiceMessage>()
    const pendingAcks = yield* Ref.make(HashMap.empty<string, ServiceMessage>())
    
    const registerService = (serviceId: string, socket: Socket.Socket) =>
      Effect.gen(function* () {
        const connection: ServiceConnection = {
          serviceId,
          socket,
          lastHeartbeat: Date.now()
        }
        
        yield* Ref.update(connections, HashMap.set(serviceId, connection))
        yield* Effect.logInfo(`Service registered: ${serviceId}`)
        
        // Start heartbeat monitoring
        yield* Effect.forkDaemon(monitorHeartbeat(serviceId))
      })
    
    const unregisterService = (serviceId: string) =>
      Effect.gen(function* () {
        yield* Ref.update(connections, HashMap.remove(serviceId))
        yield* Effect.logInfo(`Service unregistered: ${serviceId}`)
      })
    
    const sendMessage = (message: ServiceMessage) =>
      Effect.gen(function* () {
        const currentConnections = yield* Ref.get(connections)
        const targetConnection = HashMap.get(currentConnections, message.to)
        
        if (targetConnection._tag === "Some") {
          const socket = targetConnection.value.socket
          const channel = Socket.toChannel(socket)
          const messageData = JSON.stringify(message)
          
          yield* Channel.writeChunk(channel, Chunk.of(messageData))
          
          // Store for ack tracking
          yield* Ref.update(pendingAcks, HashMap.set(message.id, message))
          
          // Set timeout for ack
          yield* Effect.sleep("30 seconds").pipe(
            Effect.andThen(checkAckTimeout(message.id)),
            Effect.forkDaemon
          )
          
          yield* Effect.logDebug(`Message sent from ${message.from} to ${message.to}`)
        } else {
          yield* Queue.offer(messageQueue, message)
          yield* Effect.logWarning(`Service ${message.to} not connected, queuing message`)
        }
      })
    
    const acknowledgeMessage = (messageId: string) =>
      Effect.gen(function* () {
        yield* Ref.update(pendingAcks, HashMap.remove(messageId))
        yield* Effect.logDebug(`Message acknowledged: ${messageId}`)
      })
    
    const checkAckTimeout = (messageId: string) =>
      Effect.gen(function* () {
        const pending = yield* Ref.get(pendingAcks)
        const message = HashMap.get(pending, messageId)
        
        if (message._tag === "Some") {
          const retryCount = (message.value.retryCount ?? 0) + 1
          
          if (retryCount < 3) {
            yield* Effect.logWarning(`Retrying message ${messageId}, attempt ${retryCount}`)
            yield* sendMessage({ ...message.value, retryCount })
          } else {
            yield* Effect.logError(`Message ${messageId} failed after 3 retries`)
            yield* Ref.update(pendingAcks, HashMap.remove(messageId))
          }
        }
      })
    
    const monitorHeartbeat = (serviceId: string) =>
      Effect.gen(function* () {
        const currentConnections = yield* Ref.get(connections)
        const connection = HashMap.get(currentConnections, serviceId)
        
        if (connection._tag === "Some") {
          const timeSinceHeartbeat = Date.now() - connection.value.lastHeartbeat
          
          if (timeSinceHeartbeat > 60000) { // 1 minute timeout
            yield* Effect.logWarning(`Service ${serviceId} heartbeat timeout`)
            yield* unregisterService(serviceId)
          }
        }
      }).pipe(
        Effect.repeat(Schedule.fixed("30 seconds"))
      )
    
    const processQueuedMessages = () =>
      Effect.gen(function* () {
        const message = yield* Queue.take(messageQueue)
        
        // Check if target service is now available
        const currentConnections = yield* Ref.get(connections)
        if (HashMap.has(currentConnections, message.to)) {
          yield* sendMessage(message)
        } else {
          // Re-queue if still not available
          yield* Queue.offer(messageQueue, message)
          yield* Effect.sleep("5 seconds")
        }
      }).pipe(
        Effect.repeat(Schedule.forever),
        Effect.forkDaemon
      )
    
    const handleConnection = (socket: Socket.Socket) =>
      Effect.gen(function* () {
        const channel = Socket.toChannel(socket)
        let serviceId: string | null = null
        
        yield* Channel.runForEach(channel, (chunk) =>
          Effect.gen(function* () {
            for (const data of chunk) {
              if (Socket.isCloseEvent(data)) {
                if (serviceId) {
                  yield* unregisterService(serviceId)
                }
                break
              }
              
              if (typeof data === "string") {
                const parsed = yield* Effect.try(() => JSON.parse(data))
                
                if (parsed.type === "register") {
                  serviceId = parsed.serviceId
                  yield* registerService(serviceId, socket)
                } else if (parsed.type === "message") {
                  yield* sendMessage(parsed as ServiceMessage)
                } else if (parsed.type === "ack") {
                  yield* acknowledgeMessage(parsed.messageId)
                } else if (parsed.type === "heartbeat") {
                  if (serviceId) {
                    yield* Ref.update(connections, HashMap.modify(serviceId, conn => ({
                      ...conn,
                      lastHeartbeat: Date.now()
                    })))
                  }
                }
              }
            }
          })
        )
      }).pipe(
        Effect.scoped,
        Effect.catchAll((error) =>
          Effect.logError(`Message bus connection error: ${error}`)
        )
      )
    
    // Start queue processor
    yield* processQueuedMessages()
    
    return {
      registerService,
      unregisterService,
      sendMessage,
      acknowledgeMessage,
      handleConnection
    } as const
  })
}) {}

// Message bus server
const createMessageBusServer = (port: number) =>
  Effect.gen(function* () {
    const messageBus = yield* MessageBus
    const server = yield* NodeSocketServer.makeNet({ port })
    
    yield* Effect.logInfo(`Message bus server starting on port ${port}`)
    
    yield* server.run(messageBus.handleConnection)
  }).pipe(
    Effect.provide(MessageBus.Default),
    Effect.provide(NodeSocket.layerNet)
  )

// Client helper for services
const createServiceClient = (busHost: string, busPort: number, serviceId: string) =>
  Effect.gen(function* () {
    const socket = yield* NodeSocket.makeNet({ 
      host: busHost, 
      port: busPort 
    })
    
    const channel = Socket.toChannel(socket)
    
    // Register with message bus
    const registerMessage = JSON.stringify({
      type: "register",
      serviceId
    })
    
    yield* Channel.writeChunk(channel, Chunk.of(registerMessage))
    
    // Start heartbeat
    yield* Effect.repeat(
      Effect.gen(function* () {
        const heartbeat = JSON.stringify({
          type: "heartbeat",
          serviceId
        })
        yield* Channel.writeChunk(channel, Chunk.of(heartbeat))
      }),
      Schedule.fixed("30 seconds")
    ), Effect.forkDaemon)
    
    const sendMessage = (to: string, type: string, payload: unknown) =>
      Effect.gen(function* () {
        const message: ServiceMessage = {
          id: crypto.randomUUID(),
          from: serviceId,
          to,
          type,
          payload,
          timestamp: Date.now()
        }
        
        const messageData = JSON.stringify({
          type: "message",
          ...message
        })
        
        yield* Channel.writeChunk(channel, Chunk.of(messageData))
      })
    
    return { sendMessage, socket } as const
  }).pipe(
    Effect.provide(NodeSocket.layerNet)
  )

// Usage
const program = Effect.gen(function* () {
  // Start message bus server
  yield* Effect.forkDaemon(createMessageBusServer(9090))
  
  yield* Effect.sleep("1 second")
  
  // Create service clients
  const userService = yield* createServiceClient("localhost", 9090, "user-service")
  const orderService = yield* createServiceClient("localhost", 9090, "order-service") 
  
  yield* Effect.sleep("1 second")
  
  // Send messages between services
  yield* userService.sendMessage("order-service", "user-created", {
    userId: "123",
    email: "user@example.com"
  })
  
  yield* orderService.sendMessage("user-service", "order-placed", {
    orderId: "456",
    userId: "123",
    amount: 99.99
  })
}).pipe(
  Effect.provide(MessageBus.Default),
  Effect.provide(NodeSocket.layerNet)
)

Effect.runFork(program)
```

## Advanced Features Deep Dive

### Feature 1: Channel-Based Stream Processing

Sockets integrate seamlessly with Effect's Channel system for powerful stream processing:

#### Basic Channel Usage

```typescript
import { Socket } from "@effect/platform"
import { Channel, Chunk, Effect, Stream } from "effect"

const processSocketStream = (socket: Socket.Socket) =>
  Effect.gen(function* () {
    const channel = Socket.toChannel(socket)
    
    // Convert to stream for easier processing
    const stream = Stream.fromChannel(channel)
    
    yield* Stream.runForEach(stream, (chunk) =>
      Effect.gen(function* () {
        for (const data of chunk) {
          if (Socket.isCloseEvent(data)) {
            yield* Effect.logInfo(`Socket closed: ${data.code}`)
          } else if (typeof data === "string") {
            yield* processTextMessage(data)
          } else {
            yield* processBinaryMessage(data)
          }
        }
      })
    )
  })

const processTextMessage = (message: string) =>
  Effect.logInfo(`Text: ${message}`)

const processBinaryMessage = (data: Uint8Array) =>
  Effect.logInfo(`Binary: ${data.length} bytes`)
```

#### Real-World Channel Example: Protocol Parser

```typescript
import { Channel, Chunk, Effect, pipe } from "effect"
import { Socket } from "@effect/platform"

// Custom protocol: [length:4][type:1][payload:length-1]
interface ProtocolMessage {
  readonly type: number
  readonly payload: Uint8Array
}

const createProtocolParser = () => {
  let buffer = new Uint8Array(0)
  
  const parseMessages = (chunk: Chunk.Chunk<Uint8Array>): Effect.Effect<ReadonlyArray<ProtocolMessage>> =>
    Effect.gen(function* () {
      // Concatenate with existing buffer
      for (const data of chunk) {
        const newBuffer = new Uint8Array(buffer.length + data.length)
        newBuffer.set(buffer)
        newBuffer.set(data, buffer.length)
        buffer = newBuffer
      }
      
      const messages: ProtocolMessage[] = []
      
      // Parse complete messages
      while (buffer.length >= 5) { // Minimum message size
        const length = new DataView(buffer.buffer).getUint32(0, false)
        
        if (buffer.length >= length + 4) {
          const type = buffer[4]
          const payload = buffer.slice(5, length + 4)
          
          messages.push({ type, payload })
          
          // Remove processed message from buffer
          buffer = buffer.slice(length + 4)
        } else {
          break // Wait for more data
        }
      }
      
      return messages
    })
  
  return { parseMessages }
}

const handleProtocolSocket = (socket: Socket.Socket) =>
  Effect.gen(function* () {
    const parser = createProtocolParser()
    const channel = Socket.toChannel(socket)
    
    yield* Channel.runForEach(channel, (chunk) =>
      Effect.gen(function* () {
        // Filter only Uint8Array chunks
        const binaryChunks = pipe(
          chunk,
          Chunk.filter((data): data is Uint8Array => data instanceof Uint8Array)
        )
        
        if (Chunk.size(binaryChunks) > 0) {
          const messages = yield* parser.parseMessages(binaryChunks)
          
          yield* Effect.forEach(messages, (message) =>
            handleProtocolMessage(message)
          )
        }
      })
    )
  })

const handleProtocolMessage = (message: ProtocolMessage) =>
  Effect.gen(function* () {
    switch (message.type) {
      case 1: // Ping
        yield* Effect.logInfo("Received ping")
        break
      case 2: // Data
        yield* Effect.logInfo(`Received data: ${message.payload.length} bytes`)
        break
      default:
        yield* Effect.logWarning(`Unknown message type: ${message.type}`)
    }
  })
```

### Feature 2: Connection Pooling and Management

#### Socket Pool Implementation

```typescript
import { Effect, Pool, Ref, Duration } from "effect"
import { NodeSocket } from "@effect/platform-node"
import { Socket } from "@effect/platform"

interface SocketPoolConfig {
  readonly host: string
  readonly port: number
  readonly minConnections: number
  readonly maxConnections: number
  readonly connectionTimeout: Duration.Duration
  readonly idleTimeout: Duration.Duration
}

class SocketPool extends Effect.Service<SocketPool>()("SocketPool", {
  succeed: (config: SocketPoolConfig) =>
    Effect.gen(function* () {
      const createSocket = () =>
        NodeSocket.makeNet({
          host: config.host,
          port: config.port,
          timeout: Duration.toMillis(config.connectionTimeout)
        }).pipe(
          Effect.provide(NodeSocket.layerNet)
        )
      
      const closeSocket = (socket: Socket.Socket) =>
        Effect.sync(() => {
          // Socket cleanup logic
        })
      
      const pool = yield* Pool.make({
        acquire: createSocket,
        release: closeSocket,
        size: config.maxConnections,
        concurrency: config.maxConnections
      })
      
      const withSocket = <R, E, A>(
        f: (socket: Socket.Socket) => Effect.Effect<A, E, R>
      ) =>
        Effect.scoped(
          Effect.gen(function* () {
            const socket = yield* Pool.get(pool)
            return yield* f(socket)
          })
        )
      
      const getPoolStats = () =>
        Effect.gen(function* () {
          const metrics = yield* Pool.metrics(pool)
          return {
            size: metrics.size,
            free: metrics.free,
            running: metrics.running
          }
        })
      
      return {
        withSocket,
        getPoolStats
      } as const
    })
}) {}

// Usage with pool
const useSocketPool = Effect.gen(function* () {
  const pool = yield* SocketPool
  
  yield* pool.withSocket((socket) =>
    Effect.gen(function* () {
      const channel = Socket.toChannel(socket)
      const message = "Hello from pool!"
      
      yield* Channel.writeChunk(channel, Chunk.of(message))
      
      // Process response
      yield* Channel.runForEach(channel, (chunk) =>
        Effect.forEach(chunk, (data) =>
          Effect.logInfo(`Pool response: ${data}`)
        )
      )
    })
  )
}).pipe(
  Effect.provide(SocketPool.Default({
    host: "localhost",
    port: 8080,
    minConnections: 2,
    maxConnections: 10,
    connectionTimeout: Duration.seconds(5),
    idleTimeout: Duration.minutes(5)
  }))
)
```

### Feature 3: Advanced Error Recovery

#### Resilient Socket Client

```typescript
import { Effect, Schedule, Ref, pipe } from "effect"
import { NodeSocket } from "@effect/platform-node"
import { Socket } from "@effect/platform"

interface ConnectionState {
  readonly isConnected: boolean
  readonly lastError?: string
  readonly reconnectCount: number
  readonly lastConnectTime?: number
}

class ResilientSocketClient extends Effect.Service<ResilientSocketClient>()("ResilientSocketClient", {
  succeed: (host: string, port: number) =>
    Effect.gen(function* () {
      const state = yield* Ref.make<ConnectionState>({
        isConnected: false,
        reconnectCount: 0
      })
      
      const currentSocket = yield* Ref.make<Socket.Socket | null>(null)
      
      const connect = () =>
        Effect.gen(function* () {
          yield* Effect.logInfo(`Connecting to ${host}:${port}`)
          
          const socket = yield* NodeSocket.makeNet({ host, port }).pipe(
            Effect.timeout("10 seconds"),
            Effect.provide(NodeSocket.layerNet)
          )
          
          yield* Ref.set(currentSocket, socket)
          yield* Ref.update(state, (s) => ({
            ...s,
            isConnected: true,
            lastConnectTime: Date.now(),
            lastError: undefined
          }))
          
          yield* Effect.logInfo("Connected successfully")
          return socket
        }).pipe(
          Effect.catchAll((error) =>
            Effect.gen(function* () {
              yield* Ref.update(state, (s) => ({
                ...s,
                isConnected: false,
                lastError: String(error),
                reconnectCount: s.reconnectCount + 1
              }))
              
              yield* Effect.logError(`Connection failed: ${error}`)
              return yield* Effect.fail(error)
            })
          )
        )
      
      const connectWithRetry = () =>
        connect().pipe(
          Effect.retry(
            pipe(
              Schedule.exponential("1 second"),
              Schedule.compose(Schedule.recurs(5)),
              Schedule.either(Schedule.spaced("30 seconds"))
            )
          )
        )
      
      const sendMessage = (message: string) =>
        Effect.gen(function* () {
          const socket = yield* Ref.get(currentSocket)
          
          if (!socket) {
            return yield* Effect.fail("Not connected")
          }
          
          const channel = Socket.toChannel(socket)
          yield* Channel.writeChunk(channel, Chunk.of(message))
        }).pipe(
          Effect.retry(Schedule.recurs(3)),
          Effect.catchAll((error) =>
            Effect.gen(function* () {
              yield* Effect.logWarning(`Send failed, reconnecting: ${error}`)
              yield* connectWithRetry()
              
              // Retry send after reconnection
              const socket = yield* Ref.get(currentSocket)
              if (socket) {
                const channel = Socket.toChannel(socket)
                yield* Channel.writeChunk(channel, Chunk.of(message))
              }
            })
          )
        )
      
      const startHeartbeat = () =>
        Effect.gen(function* () {
          const socket = yield* Ref.get(currentSocket)
          if (socket) {
            yield* sendMessage("PING")
          }
        }).pipe(
          Effect.repeat(Schedule.fixed("30 seconds")),
          Effect.catchAll((error) =>
            Effect.logWarning(`Heartbeat failed: ${error}`)
          ),
          Effect.forkDaemon
        )
      
      const getConnectionState = () => Ref.get(state)
      
      // Initial connection
      yield* connectWithRetry()
      yield* startHeartbeat()
      
      return {
        sendMessage,
        getConnectionState,
        connect: connectWithRetry
      } as const
    })
}) {}

// Usage
const program = Effect.gen(function* () {
  const client = yield* ResilientSocketClient
  
  // Send messages with automatic retry/reconnection
  yield* client.sendMessage("Hello, resilient world!")
  
  yield* Effect.sleep("5 seconds")
  
  const connectionState = yield* client.getConnectionState()
  yield* Effect.logInfo(`Connection state: ${JSON.stringify(connectionState)}`)
}).pipe(
  Effect.provide(ResilientSocketClient.Default("localhost", 8080))
)

Effect.runFork(program)
```

## Practical Patterns & Best Practices

### Pattern 1: Request-Response Over TCP

```typescript
import { Effect, Ref, HashMap, Duration, Deferred } from "effect"
import { Socket } from "@effect/platform"

// Request-response pattern for TCP communication
interface PendingRequest {
  readonly id: string
  readonly deferred: Deferred.Deferred<any, Error>
  readonly timeout: number
}

const createRequestResponseClient = (socket: Socket.Socket) =>
  Effect.gen(function* () {
    const pendingRequests = yield* Ref.make(HashMap.empty<string, PendingRequest>())
    const channel = Socket.toChannel(socket)
    
    // Response handler
    yield* Channel.runForEach(channel, (chunk) =>
      Effect.gen(function* () {
        for (const data of chunk) {
          if (typeof data === "string") {
            const response = yield* Effect.try(() => JSON.parse(data))
            
            if (response.id) {
              const pending = yield* Ref.get(pendingRequests).pipe(
                Effect.map(HashMap.get(response.id))
              )
              
              if (pending._tag === "Some") {
                yield* Deferred.succeed(pending.value.deferred, response.data)
                yield* Ref.update(pendingRequests, HashMap.remove(response.id))
              }
            }
          }
        }
      })
    ), Effect.forkDaemon)
    
    const sendRequest = <T>(data: unknown, timeout = Duration.seconds(30)) =>
      Effect.gen(function* () {
        const id = crypto.randomUUID()
        const deferred = yield* Deferred.make<T, Error>()
        
        const request = {
          id,
          deferred,
          timeout: Date.now() + Duration.toMillis(timeout)
        }
        
        yield* Ref.update(pendingRequests, HashMap.set(id, request))
        
        // Send request
        const requestData = JSON.stringify({ id, data })
        yield* Channel.writeChunk(channel, Chunk.of(requestData))
        
        // Wait for response with timeout
        return yield* Deferred.await(deferred).pipe(
          Effect.timeout(timeout),
          Effect.catchTag("TimeoutException", () =>
            Effect.gen(function* () {
              yield* Ref.update(pendingRequests, HashMap.remove(id))
              return yield* Effect.fail(new Error("Request timeout"))
            })
          )
        )
      })
    
    // Cleanup expired requests
    yield* Effect.repeat(
      Effect.gen(function* () {
        const now = Date.now()
        const requests = yield* Ref.get(pendingRequests)
        
        const expired = pipe(
          requests,
          HashMap.filter((req) => req.timeout < now)
        )
        
        yield* Effect.forEach(
          HashMap.keys(expired),
          (id) =>
            Effect.gen(function* () {
              const req = HashMap.get(requests, id)
              if (req._tag === "Some") {
                yield* Deferred.fail(req.value.deferred, new Error("Request timeout"))
              }
              yield* Ref.update(pendingRequests, HashMap.remove(id))
            })
        )
      }),
      Schedule.fixed("10 seconds")
    ), Effect.forkDaemon)
    
    return { sendRequest } as const
  })

// Usage
const program = Effect.gen(function* () {
  const socket = yield* NodeSocket.makeNet({ host: "localhost", port: 8080 })
  const client = yield* createRequestResponseClient(socket)
  
  const response = yield* client.sendRequest({ 
    action: "get_user", 
    userId: "123" 
  })
  
  yield* Effect.logInfo(`Response: ${JSON.stringify(response)}`)
})
```

### Pattern 2: Message Broadcasting with Groups

```typescript
import { Effect, Ref, HashMap, HashSet } from "effect"
import { Socket } from "@effect/platform"

// Group-based message broadcasting
interface GroupManager {
  readonly subscribe: (socket: Socket.Socket, groupId: string) => Effect.Effect<void>
  readonly unsubscribe: (socket: Socket.Socket, groupId: string) => Effect.Effect<void>
  readonly broadcast: (groupId: string, message: any) => Effect.Effect<void>
  readonly broadcastToAll: (message: any) => Effect.Effect<void>
}

const createGroupManager = (): Effect.Effect<GroupManager> =>
  Effect.gen(function* () {
    const groups = yield* Ref.make(HashMap.empty<string, HashSet.HashSet<Socket.Socket>>())
    const socketGroups = yield* Ref.make(HashMap.empty<Socket.Socket, HashSet.HashSet<string>>())
    
    const subscribe = (socket: Socket.Socket, groupId: string) =>
      Effect.gen(function* () {
        // Add socket to group
        yield* Ref.update(groups, HashMap.modify(groupId, (sockets) =>
          HashSet.add(sockets ?? HashSet.empty(), socket)
        ))
        
        // Track groups for socket
        yield* Ref.update(socketGroups, HashMap.modify(socket, (groupIds) =>
          HashSet.add(groupIds ?? HashSet.empty(), groupId)
        ))
        
        yield* Effect.logDebug(`Socket subscribed to group: ${groupId}`)
      })
    
    const unsubscribe = (socket: Socket.Socket, groupId: string) =>
      Effect.gen(function* () {
        // Remove socket from group
        yield* Ref.update(groups, HashMap.modify(groupId, (sockets) =>
          sockets ? HashSet.remove(sockets, socket) : HashSet.empty()
        ))
        
        // Remove group from socket tracking
        yield* Ref.update(socketGroups, HashMap.modify(socket, (groupIds) =>
          groupIds ? HashSet.remove(groupIds, groupId) : HashSet.empty()
        ))
        
        yield* Effect.logDebug(`Socket unsubscribed from group: ${groupId}`)
      })
    
    const broadcast = (groupId: string, message: any) =>
      Effect.gen(function* () {
        const currentGroups = yield* Ref.get(groups)
        const sockets = HashMap.get(currentGroups, groupId)
        
        if (sockets._tag === "Some") {
          const messageData = JSON.stringify(message)
          
          yield* Effect.forEach(
            HashSet.values(sockets.value),
            (socket) =>
              Effect.gen(function* () {
                const channel = Socket.toChannel(socket)
                yield* Channel.writeChunk(channel, Chunk.of(messageData))
              }).pipe(
                Effect.catchAll((error) =>
                  Effect.logWarning(`Failed to send to socket: ${error}`)
                )
              ),
            { concurrency: "unbounded" }
          )
          
          yield* Effect.logDebug(`Broadcasted to group ${groupId}: ${HashSet.size(sockets.value)} sockets`)
        }
      })
    
    const broadcastToAll = (message: any) =>
      Effect.gen(function* () {
        const currentGroups = yield* Ref.get(groups)
        const allGroupIds = pipe(
          currentGroups,
          HashMap.keys,
          Array.from
        )
        
        yield* Effect.forEach(
          allGroupIds,
          (groupId) => broadcast(groupId, message),
          { concurrency: "unbounded" }
        )
      })
    
    return {
      subscribe,
      unsubscribe,
      broadcast,
      broadcastToAll
    } as const
  })

// Usage in a server
const createGroupServer = (port: number) =>
  Effect.gen(function* () {
    const groupManager = yield* createGroupManager()
    const server = yield* NodeSocketServer.makeNet({ port })
    
    yield* server.run((socket) =>
      Effect.gen(function* () {
        const channel = Socket.toChannel(socket)
        
        yield* Channel.runForEach(channel, (chunk) =>
          Effect.gen(function* () {
            for (const data of chunk) {
              if (typeof data === "string") {
                const message = yield* Effect.try(() => JSON.parse(data))
                
                switch (message.type) {
                  case "subscribe":
                    yield* groupManager.subscribe(socket, message.groupId)
                    break
                  case "unsubscribe":
                    yield* groupManager.unsubscribe(socket, message.groupId)
                    break
                  case "broadcast":
                    yield* groupManager.broadcast(message.groupId, message.data)
                    break
                  default:
                    yield* Effect.logWarning(`Unknown message type: ${message.type}`)
                }
              }
            }
          })
        )
      }).pipe(
        Effect.scoped,
        Effect.ensuring(
          Effect.gen(function* () {
            // Cleanup socket from all groups on disconnect
            const currentSocketGroups = yield* Ref.get(socketGroups)
            const groups = HashMap.get(currentSocketGroups, socket)
            
            if (groups._tag === "Some") {
              yield* Effect.forEach(
                HashSet.values(groups.value),
                (groupId) => groupManager.unsubscribe(socket, groupId)
              )
            }
          })
        )
      )
    )
  })
```

### Pattern 3: Connection Rate Limiting

```typescript
import { Effect, Ref, HashMap, Duration, Schedule } from "effect"
import { Socket } from "@effect/platform"

// Rate limiting for socket connections
interface RateLimitConfig {
  readonly maxConnectionsPerIp: number
  readonly timeWindow: Duration.Duration
  readonly blockDuration: Duration.Duration
}

interface ConnectionInfo {
  readonly count: number
  readonly firstConnection: number
  readonly blocked: boolean
  readonly blockExpiry?: number
}

const createRateLimiter = (config: RateLimitConfig) =>
  Effect.gen(function* () {
    const connections = yield* Ref.make(HashMap.empty<string, ConnectionInfo>())
    
    const isAllowed = (ip: string) =>
      Effect.gen(function* () {
        const now = Date.now()
        const windowStart = now - Duration.toMillis(config.timeWindow)
        
        const currentConnections = yield* Ref.get(connections)
        const info = HashMap.get(currentConnections, ip)
        
        if (info._tag === "None") {
          // First connection from this IP
          const newInfo: ConnectionInfo = {
            count: 1,
            firstConnection: now,
            blocked: false
          }
          
          yield* Ref.update(connections, HashMap.set(ip, newInfo))
          return true
        }
        
        const connInfo = info.value
        
        // Check if IP is currently blocked
        if (connInfo.blocked && connInfo.blockExpiry && now < connInfo.blockExpiry) {
          return false
        }
        
        // Reset count if window has passed
        if (connInfo.firstConnection < windowStart) {
          const resetInfo: ConnectionInfo = {
            count: 1,
            firstConnection: now,
            blocked: false
          }
          
          yield* Ref.update(connections, HashMap.set(ip, resetInfo))
          return true
        }
        
        // Check if limit exceeded
        if (connInfo.count >= config.maxConnectionsPerIp) {
          const blockedInfo: ConnectionInfo = {
            ...connInfo,
            blocked: true,
            blockExpiry: now + Duration.toMillis(config.blockDuration)
          }
          
          yield* Ref.update(connections, HashMap.set(ip, blockedInfo))
          yield* Effect.logWarning(`IP ${ip} blocked for rate limiting`)
          return false
        }
        
        // Increment count
        const updatedInfo: ConnectionInfo = {
          ...connInfo,
          count: connInfo.count + 1
        }
        
        yield* Ref.update(connections, HashMap.set(ip, updatedInfo))
        return true
      })
    
    const cleanup = () =>
      Effect.gen(function* () {
        const now = Date.now()
        const currentConnections = yield* Ref.get(connections)
        
        const cleaned = pipe(
          currentConnections,
          HashMap.filter((info) => {
            const windowStart = now - Duration.toMillis(config.timeWindow)
            const blockExpired = !info.blocked || 
              !info.blockExpiry || 
              now >= info.blockExpiry
            
            return info.firstConnection >= windowStart || !blockExpired
          })
        )
        
        yield* Ref.set(connections, cleaned)
      }).pipe(
        Effect.repeat(Schedule.fixed("1 minute")),
        Effect.forkDaemon
      )
    
    yield* cleanup()
    
    return { isAllowed } as const
  })

// Usage with server
const createRateLimitedServer = (port: number) =>
  Effect.gen(function* () {
    const rateLimiter = yield* createRateLimiter({
      maxConnectionsPerIp: 10,
      timeWindow: Duration.minutes(1),
      blockDuration: Duration.minutes(5)
    })
    
    const server = yield* NodeSocketServer.makeNet({ port })
    
    yield* server.run((socket) =>
      Effect.gen(function* () {
        // Extract IP address (simplified)
        const ip = "127.0.0.1" // In real implementation, extract from socket
        
        const allowed = yield* rateLimiter.isAllowed(ip)
        
        if (!allowed) {
          yield* Effect.logWarning(`Connection rejected for IP: ${ip}`)
          // Close connection immediately
          return
        }
        
        yield* Effect.logInfo(`Connection accepted for IP: ${ip}`)
        
        // Handle connection normally
        const channel = Socket.toChannel(socket)
        yield* handleConnection(channel)
      }).pipe(
        Effect.scoped,
        Effect.catchAll((error) =>
          Effect.logError(`Connection error: ${error}`)
        )
      )
    )
  })
```

## Integration Examples

### Integration with Effect HTTP Server

```typescript
import { NodeHttpServer } from "@effect/platform-node"
import { HttpApi, HttpApiBuilder, HttpMiddleware } from "@effect/platform"
import { NodeSocket } from "@effect/platform-node"
import { Socket } from "@effect/platform"
import { Effect, Layer } from "effect"

// WebSocket upgrade integration with HTTP server
const createWebSocketApi = HttpApi.make("WebSocketApi").pipe(
  HttpApi.addEndpoint(
    HttpApi.get("websocket", "/ws").pipe(
      HttpApi.setHeaders({ "Connection": "Upgrade", "Upgrade": "websocket" })
    )
  )
)

const WebSocketApiLive = HttpApiBuilder.api(createWebSocketApi).pipe(
  HttpApiBuilder.handle("websocket", (req) =>
    Effect.gen(function* () {
      // Upgrade to WebSocket
      const socket = yield* Socket.fromWebSocket(
        Effect.succeed(new WebSocket("ws://localhost:8080/ws"))
      )
      
      const channel = Socket.toChannel(socket)
      
      // Handle WebSocket messages
      yield* Channel.runForEach(channel, (chunk) =>
        Effect.forEach(chunk, (data) =>
          Effect.logInfo(`WebSocket message: ${data}`)
        )
      )
      
      return yield* HttpApi.response.text("WebSocket connection established")
    })
  ),
  HttpApiBuilder.catchAllCause(Effect.succeed(HttpApi.response.text("Internal Server Error", { status: 500 })))
)

// Combined HTTP + WebSocket server
const program = Effect.gen(function* () {
  yield* NodeHttpServer.listen({ port: 8080 })
}).pipe(
  Effect.provide(WebSocketApiLive),
  Effect.provide(NodeHttpServer.layer),
  Effect.provide(NodeSocket.layerWebSocketConstructor)
)
```

### Integration with Database Connections

```typescript
import { SqlClient } from "@effect/sql"
import { NodeSocket } from "@effect/platform-node"
import { Socket } from "@effect/platform"
import { Effect, Layer } from "effect"

// Socket server with database integration
interface UserMessage {
  readonly userId: string
  readonly content: string
  readonly timestamp: number
}

const createDatabaseSocketServer = (port: number) =>
  Effect.gen(function* () {
    const sql = yield* SqlClient.SqlClient
    const server = yield* NodeSocketServer.makeNet({ port })
    
    yield* server.run((socket) =>
      Effect.gen(function* () {
        const channel = Socket.toChannel(socket)
        
        yield* Channel.runForEach(channel, (chunk) =>
          Effect.gen(function* () {
            for (const data of chunk) {
              if (typeof data === "string") {
                const message = yield* Effect.try(() => JSON.parse(data) as UserMessage)
                
                // Store message in database
                yield* sql`
                  INSERT INTO messages (user_id, content, timestamp)
                  VALUES (${message.userId}, ${message.content}, ${message.timestamp})
                `
                
                // Broadcast to other clients
                yield* broadcastMessage(message)
                
                yield* Effect.logInfo(`Stored message from user ${message.userId}`)
              }
            }
          })
        )
      }).pipe(
        Effect.scoped,
        Effect.catchAll((error) =>
          Effect.logError(`Socket error: ${error}`)
        )
      )
    )
  }).pipe(
    Effect.provide(SqlClient.layer),
    Effect.provide(NodeSocket.layerNet)
  )

const broadcastMessage = (message: UserMessage) =>
  Effect.logInfo(`Broadcasting: ${message.content}`)
```

### Testing Strategies

```typescript
import { Effect, TestContext, Layer, Ref } from "effect"
import { Socket } from "@effect/platform"

// Mock Socket implementation for testing
interface MockSocket extends Socket.Socket {
  readonly sent: ReadonlyArray<string>
  readonly received: ReadonlyArray<string>
}

const createMockSocket = (received: ReadonlyArray<string> = []) =>
  Effect.gen(function* () {
    const sentRef = yield* Ref.make<ReadonlyArray<string>>([])
    const receivedRef = yield* Ref.make(received)
    
    const mockSocket: MockSocket = {
      sent: [],
      received: [],
      [Socket.TypeId]: Socket.TypeId,
      // Implement other Socket methods as needed for testing
    }
    
    return mockSocket
  })

// Test example
const testSocketCommunication = Effect.gen(function* () {
  const mockSocket = yield* createMockSocket(["test message"])
  
  // Test your socket handling logic
  const result = yield* handleSocketMessage(mockSocket, "test input")
  
  // Assertions
  yield* Effect.sync(() => {
    expect(result).toBe("expected output")
  })
}).pipe(
  Effect.provide(TestContext.TestContext)
)

// Integration test with real sockets
const testRealSocketConnection = Effect.gen(function* () {
  const testPort = 9999
  
  // Start test server
  const server = yield* Effect.forkDaemon(createTestServer(testPort))
  
  yield* Effect.sleep("100 millis") // Let server start
  
  // Connect client
  const client = yield* NodeSocket.makeNet({ 
    host: "localhost", 
    port: testPort 
  })
  
  // Send test message
  const channel = Socket.toChannel(client)
  yield* Channel.writeChunk(channel, Chunk.of("test message"))
  
  // Verify response
  yield* Channel.runForEach(channel, (chunk) =>
    Effect.forEach(chunk, (data) =>
      Effect.sync(() => {
        expect(data).toContain("expected response")
      })
    )
  )
  
  // Cleanup
  yield* Effect.interrupt(server)
}).pipe(
  Effect.provide(NodeSocket.layerNet),
  Effect.scoped
)

const createTestServer = (port: number) =>
  Effect.gen(function* () {
    const server = yield* NodeSocketServer.makeNet({ port })
    
    yield* server.run((socket) =>
      Effect.gen(function* () {
        const channel = Socket.toChannel(socket)
        
        yield* Channel.runForEach(channel, (chunk) =>
          Effect.forEach(chunk, (data) =>
            Effect.gen(function* () {
              if (typeof data === "string") {
                const response = `Echo: ${data}`
                yield* Channel.writeChunk(channel, Chunk.of(response))
              }
            })
          )
        )
      }), Effect.scoped)
    )
  })
```

## Conclusion

Socket provides powerful, type-safe networking capabilities for building robust real-time applications, chat systems, and microservice communication in the Effect ecosystem.

Key benefits:
- **Type Safety**: Full TypeScript support with proper error handling
- **Resource Management**: Automatic cleanup using Effect's Scope system  
- **Composability**: Seamless integration with Channels, Streams, and other Effect modules
- **Cross-Platform**: Works across Node.js, Browser, Bun, and other platforms
- **Error Recovery**: Built-in retry mechanisms and structured error handling

Use Socket when you need real-time bidirectional communication, custom protocols, or high-performance networking with the safety and composability of the Effect ecosystem.