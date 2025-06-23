# Inspectable: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns) 
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem Inspectable Solves

When building complex applications with custom data structures, debugging becomes painful without proper inspection capabilities. Traditional approaches lead to:

```typescript
// Traditional approach - poor debugging experience
class User {
  constructor(
    public id: string,
    public email: string,
    private password: string
  ) {}
}

const user = new User("123", "john@example.com", "secret123")
console.log(user) // User { id: '123', email: 'john@example.com', password: 'secret123' }
console.log(JSON.stringify(user)) // {"id":"123","email":"john@example.com","password":"secret123"}
```

This approach leads to:
- **Security Issues** - Sensitive data exposed in logs and debugging output
- **Poor Development Experience** - Inconsistent string representations across different environments 
- **Debugging Challenges** - No control over how objects appear in Node.js inspector or JSON serialization
- **Maintenance Overhead** - Manual implementation of toString/toJSON methods for each class

### The Inspectable Solution

Inspectable provides a unified interface for custom inspection, JSON serialization, and Node.js debugging integration:

```typescript
import { Inspectable } from "effect"

class User extends Inspectable.Class {  
  constructor(
    public id: string,
    public email: string,
    private password: string
  ) {
    super()
  }

  toJSON() {
    return {
      _id: "User",
      id: this.id,
      email: this.email,
      // password deliberately omitted for security
    }
  }
}

const user = new User("123", "john@example.com", "secret123")
console.log(user.toString()) // Clean, formatted JSON output
console.log(JSON.stringify(user)) // Secure serialization without password
```

### Key Concepts

**Inspectable Interface**: Defines three methods for custom inspection - `toString()`, `toJSON()`, and `[NodeInspectSymbol]()`

**Inspectable.Class**: Abstract base class that implements the Inspectable interface with sensible defaults

**Node.js Integration**: Seamless integration with Node.js `util.inspect()` and REPL debugging

## Basic Usage Patterns

### Pattern 1: Extending Inspectable.Class

```typescript
import { Inspectable } from "effect"

class Product extends Inspectable.Class {
  constructor(
    public id: string,
    public name: string,
    public price: number,
    private internalCost: number
  ) {
    super()
  }

  toJSON() {
    return {
      _id: "Product", 
      id: this.id,
      name: this.name,
      price: this.price,
      // internalCost omitted from public representation
    }
  }
}

const product = new Product("prod-1", "MacBook Pro", 2499, 1800)
console.log(product.toString()) 
// {
//   "_id": "Product",
//   "id": "prod-1", 
//   "name": "MacBook Pro",
//   "price": 2499
// }
```

### Pattern 2: Implementing Inspectable Interface

```typescript
import { Inspectable } from "effect"

class ApiResponse<T> implements Inspectable.Inspectable {
  constructor(
    public data: T,
    public status: number,
    private headers: Record<string, string>
  ) {}

  toJSON() {
    return {
      _id: "ApiResponse",
      data: Inspectable.toJSON(this.data),
      status: this.status,
      // headers filtered for security
      headers: Object.fromEntries(
        Object.entries(this.headers).filter(([key]) => 
          !key.toLowerCase().includes('auth')
        )
      )
    }
  }

  toString() {
    return Inspectable.format(this.toJSON())
  }

  [Inspectable.NodeInspectSymbol]() {
    return this.toJSON()
  }
}
```

### Pattern 3: Using Inspectable Utilities

```typescript
import { Inspectable } from "effect"

const debugLog = (label: string, value: unknown) => {
  console.log(`${label}:`, Inspectable.toStringUnknown(value, 2))
}

const complexData = {
  users: [
    new User("1", "alice@example.com", "pass1"),
    new User("2", "bob@example.com", "pass2")
  ],
  metadata: { version: "1.0.0", timestamp: new Date() }
}

debugLog("Complex Data", complexData)
// Complex Data: {
//   "users": [
//     {
//       "_id": "User",
//       "id": "1", 
//       "email": "alice@example.com"
//     },
//     {
//       "_id": "User", 
//       "id": "2",
//       "email": "bob@example.com" 
//     }
//   ],
//   "metadata": {
//     "version": "1.0.0",
//     "timestamp": "2023-12-01T10:00:00.000Z"
//   }
// }
```

## Real-World Examples

### Example 1: Database Entity with Audit Fields

```typescript
import { Inspectable, Effect, Context } from "effect"

class OrderEntity extends Inspectable.Class {
  constructor(
    public id: string,
    public customerId: string, 
    public items: Array<OrderItem>,
    public total: number,
    public status: OrderStatus,
    public createdAt: Date,
    public updatedAt: Date,
    private auditLog: Array<AuditEntry>
  ) {
    super()
  }

  toJSON() {
    return {
      _id: "Order",
      id: this.id,
      customerId: this.customerId,
      items: this.items.map(item => Inspectable.toJSON(item)),
      total: this.total,
      status: this.status,
      createdAt: this.createdAt.toISOString(),
      updatedAt: this.updatedAt.toISOString(),
      // auditLog excluded from standard serialization for size/security
    }
  }
}

// Business logic using Effect.gen + yield*
const processOrder = (orderId: string) => Effect.gen(function* () {
  const orderRepo = yield* OrderRepository
  const paymentService = yield* PaymentService
  
  const order = yield* orderRepo.findById(orderId)
  
  if (order.status !== "pending") {
    return yield* Effect.fail(new OrderAlreadyProcessedError({ orderId }))
  }
  
  const paymentResult = yield* paymentService.processPayment({
    amount: order.total,
    customerId: order.customerId
  })
  
  const updatedOrder = yield* orderRepo.updateStatus(orderId, "paid")
  
  return updatedOrder
}).pipe(
  Effect.withSpan("order.process", { attributes: { orderId } }),
  Effect.catchTag("PaymentError", (error) => 
    Effect.fail(new OrderProcessingError({ cause: error, orderId }))
  )
)

interface OrderRepository {
  findById: (id: string) => Effect.Effect<OrderEntity, NotFoundError>
  updateStatus: (id: string, status: OrderStatus) => Effect.Effect<OrderEntity, DatabaseError>
}

const OrderRepository = Context.GenericTag<OrderRepository>("OrderRepository")

interface PaymentService {
  processPayment: (params: PaymentParams) => Effect.Effect<PaymentResult, PaymentError>
}

const PaymentService = Context.GenericTag<PaymentService>("PaymentService")
```

### Example 2: HTTP Client with Request/Response Logging

```typescript
import { Inspectable, Effect, Logger } from "effect"

class HttpRequest extends Inspectable.Class {
  constructor(
    public method: string,
    public url: string,
    public headers: Record<string, string>,
    public body?: unknown,
    private authToken?: string
  ) {
    super()
  }

  toJSON() {
    const safeHeaders = Object.fromEntries(
      Object.entries(this.headers).filter(([key]) => 
        !['authorization', 'cookie', 'x-api-key'].includes(key.toLowerCase())
      )
    )

    return {
      _id: "HttpRequest",
      method: this.method,
      url: this.url,
      headers: safeHeaders,
      body: this.body ? Inspectable.toJSON(this.body) : undefined,
      hasAuth: !!this.authToken
    }
  }
}

class HttpResponse<T> extends Inspectable.Class {
  constructor(
    public status: number,
    public headers: Record<string, string>,
    public body: T,
    public request: HttpRequest,
    public duration: number
  ) {
    super()
  }

  toJSON() {
    return {
      _id: "HttpResponse",
      status: this.status,
      headers: this.headers,
      body: Inspectable.toJSON(this.body),
      request: this.request.toJSON(),
      duration: this.duration
    }
  }
}

// HTTP client with automatic logging
const makeHttpClient = Effect.gen(function* () {
  const logger = yield* Logger.Logger

  const execute = <T>(request: HttpRequest) => Effect.gen(function* () {
    const startTime = yield* Effect.clock.then(clock => clock.currentTimeMillis)
    
    yield* Effect.logInfo("HTTP Request").pipe(
      Effect.annotateLogs({
        method: request.method,
        url: request.url,
        request: request.toString()
      })
    )

    const response = yield* performHttpRequest<T>(request)
    const duration = yield* Effect.clock.then(clock => 
      Effect.map(clock.currentTimeMillis, end => end - startTime)
    )

    const httpResponse = new HttpResponse(
      response.status,
      response.headers,
      response.body,
      request,
      duration
    )

    yield* Effect.logInfo("HTTP Response").pipe(
      Effect.annotateLogs({
        status: response.status,
        duration: duration,
        response: httpResponse.toString()
      })
    )

    return httpResponse
  })

  return { execute } as const
}).pipe(
  Effect.withSpan("http-client.make")
)

const performHttpRequest = <T>(request: HttpRequest): Effect.Effect<{ 
  status: number
  headers: Record<string, string>
  body: T 
}, HttpError> => {
  // Implementation would make actual HTTP request
  return Effect.succeed({
    status: 200,
    headers: { "content-type": "application/json" },
    body: { message: "success" } as T
  })
}
```

### Example 3: Event Store with Structured Logging

```typescript
import { Inspectable, Effect, Array as Arr } from "effect"

class DomainEvent extends Inspectable.Class {
  constructor(
    public id: string,
    public type: string,
    public aggregateId: string,
    public data: unknown,
    public metadata: EventMetadata,
    public timestamp: Date
  ) {
    super()
  }

  toJSON() {
    return {
      _id: "DomainEvent",
      id: this.id,
      type: this.type,
      aggregateId: this.aggregateId,
      data: Inspectable.toJSON(this.data),
      metadata: {
        ...this.metadata,
        // Redact sensitive metadata fields
        userId: this.metadata.userId ? "***" : undefined
      },
      timestamp: this.timestamp.toISOString()
    }
  }
}

interface EventMetadata {
  userId?: string
  correlationId: string
  causationId?: string
  version: number
}

class EventStream extends Inspectable.Class {
  constructor(
    public aggregateId: string,
    public events: Array<DomainEvent>,
    public version: number
  ) {
    super()
  }

  toJSON() {
    return {
      _id: "EventStream", 
      aggregateId: this.aggregateId,
      eventCount: this.events.length,
      events: this.events.map(event => event.toJSON()),
      version: this.version
    }
  }
}

// Event store operations with structured logging
const appendEvents = (
  aggregateId: string,
  expectedVersion: number,
  events: Array<DomainEvent>
) => Effect.gen(function* () {
  const eventStore = yield* EventStore
  const logger = yield* Logger.Logger

  yield* Effect.logDebug("Appending events").pipe(
    Effect.annotateLogs({
      aggregateId,
      expectedVersion,
      eventCount: events.length,
      events: Arr.map(events, event => event.toString())
    })
  )

  const currentStream = yield* eventStore.getStream(aggregateId)
  
  if (currentStream.version !== expectedVersion) {
    return yield* Effect.fail(new ConcurrencyError({
      aggregateId,
      expectedVersion,
      actualVersion: currentStream.version
    }))
  }

  const newEvents = Arr.map(events, event => 
    new DomainEvent(
      event.id,
      event.type,
      aggregateId,
      event.data,
      { ...event.metadata, version: currentStream.version + 1 },
      event.timestamp
    )
  )

  const updatedStream = new EventStream(
    aggregateId,
    [...currentStream.events, ...newEvents],
    currentStream.version + newEvents.length
  )

  yield* eventStore.saveStream(updatedStream)

  yield* Effect.logInfo("Events appended successfully").pipe(
    Effect.annotateLogs({
      aggregateId,
      newVersion: updatedStream.version,
      stream: updatedStream.toString()
    })
  )

  return updatedStream
}).pipe(
  Effect.withSpan("event-store.append", {
    attributes: { aggregateId, expectedVersion }
  })
)

interface EventStore {
  getStream: (aggregateId: string) => Effect.Effect<EventStream, NotFoundError>
  saveStream: (stream: EventStream) => Effect.Effect<void, DatabaseError>
}

const EventStore = Context.GenericTag<EventStore>("EventStore")
```

## Advanced Features Deep Dive

### Redactable: Secure Inspection with Context

The Redactable interface allows objects to provide different representations based on the current fiber context, enabling context-aware data redaction:

```typescript
import { Inspectable, FiberRefs, Effect, Context } from "effect"

// Configuration for what should be redacted
interface RedactionConfig {
  redactPasswords: boolean
  redactEmails: boolean
  redactCreditCards: boolean
}

const RedactionConfig = Context.GenericTag<RedactionConfig>("RedactionConfig")

class UserProfile extends Inspectable.Class implements Inspectable.Redactable {
  constructor(
    public id: string,
    public email: string,
    public name: string,
    private password: string,
    private creditCard?: string
  ) {
    super()
  }

  // Standard toJSON - always secure
  toJSON() {
    return {
      _id: "UserProfile",
      id: this.id,
      name: this.name,
      // Always redact sensitive data in standard serialization
      email: "***@***.com",
      hasPassword: !!this.password,
      hasCreditCard: !!this.creditCard
    }
  }

  // Context-aware redaction
  [Inspectable.symbolRedactable](fiberRefs: FiberRefs.FiberRefs) {
    const config = FiberRefs.getOrDefault(fiberRefs, RedactionConfig, {
      redactPasswords: true,
      redactEmails: true, 
      redactCreditCards: true
    })

    return {
      _id: "UserProfile",
      id: this.id,
      name: this.name,
      email: config.redactEmails ? "***@***.com" : this.email,
      password: config.redactPasswords ? "***" : this.password,
      creditCard: config.redactCreditCards ? "****-****-****-1234" : this.creditCard
    }
  }
}

// Usage in different contexts
const debugUserInDevelopment = (user: UserProfile) => Effect.gen(function* () {
  // In development, show all data for debugging
  const devConfig: RedactionConfig = {
    redactPasswords: false,
    redactEmails: false,
    redactCreditCards: true // Still redact credit cards
  }

  yield* Effect.logDebug("User Debug Info").pipe(
    Effect.annotateLogs("user", Inspectable.toStringUnknown(user)),
    Effect.provideService(RedactionConfig, devConfig)
  )
})

const logUserInProduction = (user: UserProfile) => Effect.gen(function* () {
  // In production, redact everything sensitive
  const prodConfig: RedactionConfig = {
    redactPasswords: true,
    redactEmails: true,
    redactCreditCards: true  
  }

  yield* Effect.logInfo("User Activity").pipe(
    Effect.annotateLogs("user", Inspectable.toStringUnknown(user)),
    Effect.provideService(RedactionConfig, prodConfig)
  )
})
```

### Circular Reference Handling

Inspectable provides automatic circular reference detection and handling:

```typescript
import { Inspectable } from "effect"

class TreeNode extends Inspectable.Class {
  public children: Array<TreeNode> = []
  public parent?: TreeNode

  constructor(public value: string) {
    super()
  }

  addChild(child: TreeNode) {
    child.parent = this
    this.children.push(child)
    return this
  }

  toJSON() {
    return {
      _id: "TreeNode",
      value: this.value,
      children: this.children.map(child => Inspectable.toJSON(child)),
      // Avoid circular reference by not including parent
      hasParent: !!this.parent
    }
  }
}

const root = new TreeNode("root")
const child1 = new TreeNode("child1")  
const child2 = new TreeNode("child2")

root.addChild(child1).addChild(child2)
child1.addChild(new TreeNode("grandchild"))

// Safe serialization with circular reference handling
console.log(Inspectable.stringifyCircular(root, 2))
// Will properly handle the parent -> child -> parent references
```

### Custom Formatting and Whitespace Control

```typescript
import { Inspectable } from "effect"

class FormattedData extends Inspectable.Class {
  constructor(public data: Record<string, unknown>) {
    super()
  }

  toJSON() {
    return {
      _id: "FormattedData",
      timestamp: new Date().toISOString(),
      data: this.data
    }
  }

  // Custom compact representation
  toCompactString() {
    return Inspectable.stringifyCircular(this.toJSON(), 0) // No whitespace
  }

  // Custom pretty representation
  toPrettyString() {
    return Inspectable.stringifyCircular(this.toJSON(), 4) // 4-space indentation
  }
}

const data = new FormattedData({
  users: [{ id: 1, name: "Alice" }, { id: 2, name: "Bob" }],
  config: { theme: "dark", notifications: true }
})

console.log("Compact:", data.toCompactString())
// Compact: {"_id":"FormattedData","timestamp":"2023-12-01T10:00:00.000Z","data":{"users":[{"id":1,"name":"Alice"},{"id":2,"name":"Bob"}],"config":{"theme":"dark","notifications":true}}}

console.log("Pretty:\n", data.toPrettyString())
// Pretty:
// {
//     "_id": "FormattedData",
//     "timestamp": "2023-12-01T10:00:00.000Z", 
//     "data": {
//         "users": [
//             {
//                 "id": 1,
//                 "name": "Alice"  
//             },
//             {
//                 "id": 2,
//                 "name": "Bob"
//             }
//         ],
//         "config": {
//             "theme": "dark",
//             "notifications": true
//         }
//     }
// }
```

## Practical Patterns & Best Practices

### Pattern 1: Inspectable Factory Functions

```typescript
import { Inspectable, Effect } from "effect"

// Factory for creating inspectable domain objects
const makeInspectableEntity = <T extends Record<string, unknown>>(
  entityType: string,
  data: T,
  sensitiveFields: Array<keyof T> = []
) => {
  return new (class extends Inspectable.Class {
    constructor() {
      super()
      Object.assign(this, data)
    }

    toJSON() {
      const safeData = Object.fromEntries(
        Object.entries(data).map(([key, value]) => [
          key,
          sensitiveFields.includes(key) ? "***" : Inspectable.toJSON(value)
        ])
      )

      return {
        _id: entityType,
        ...safeData
      }
    }
  })()
}

// Usage
const createUser = (userData: {
  id: string
  email: string  
  password: string
  profile: UserProfile
}) => Effect.gen(function* () {
  const userRepo = yield* UserRepository

  const user = makeInspectableEntity(
    "User",
    userData,
    ["password"] // Mark password as sensitive
  )

  yield* Effect.logDebug("Creating user").pipe(
    Effect.annotateLogs("user", user.toString())
  )

  return yield* userRepo.save(user)
})
```

### Pattern 2: Debugging Utilities

```typescript
import { Inspectable, Effect, Logger, Duration } from "effect"

// Debug helper that formats and logs any value
const debugValue = <A>(label: string, value: A) => 
  Effect.logDebug(`${label}: ${Inspectable.toStringUnknown(value, 2)}`)

// Performance debugging with automatic formatting
const withPerformanceLogging = <A, E, R>(
  name: string,
  effect: Effect.Effect<A, E, R>
) => Effect.gen(function* () {
  const startTime = yield* Effect.clockWith(clock => clock.currentTimeMillis)
  
  yield* debugValue("Input", { operation: name, startTime })
  
  const result = yield* effect
  
  const endTime = yield* Effect.clockWith(clock => clock.currentTimeMillis)
  const duration = endTime - startTime
  
  yield* debugValue("Output", { 
    operation: name,
    duration: `${duration}ms`,
    result: Inspectable.toStringUnknown(result, 1)
  })
  
  return result
}).pipe(
  Effect.withSpan(`performance.${name}`, {
    attributes: { operation: name }
  })
)

// Usage
const processOrder = (orderId: string) =>
  withPerformanceLogging("processOrder", Effect.gen(function* () {
    const order = yield* getOrder(orderId)
    const payment = yield* processPayment(order.total)
    const notification = yield* sendNotification(order.customerId)
    
    return { order, payment, notification }
  }))
```

### Pattern 3: Testing Helpers

```typescript  
import { Inspectable, Effect, TestContext } from "effect"

// Test utility for comparing inspectable objects
const expectInspectableEquals = <A extends Inspectable.Inspectable>(
  actual: A,
  expected: unknown
) => Effect.gen(function* () {
  const actualJson = actual.toJSON()
  const expectedJson = Inspectable.toJSON(expected)
  
  if (!Equal.equals(actualJson, expectedJson)) {
    return yield* Effect.fail(new Error(
      `Inspectable objects not equal:\n` +
      `Expected: ${Inspectable.format(expectedJson)}\n` +
      `Actual: ${Inspectable.format(actualJson)}`
    ))
  }
})

// Snapshot testing helper
const expectInspectableMatchesSnapshot = <A extends Inspectable.Inspectable>(
  value: A,
  snapshotName: string
) => Effect.gen(function* () {
  const json = value.toJSON()
  const formatted = Inspectable.format(json)
  
  // In a real implementation, this would compare against saved snapshots
  yield* Effect.logInfo(`Snapshot ${snapshotName}:\n${formatted}`)
})

// Usage in tests
const testUserCreation = Effect.gen(function* () {
  const user = new User("123", "test@example.com", "password")
  
  yield* expectInspectableEquals(user, {
    _id: "User",
    id: "123", 
    email: "test@example.com"
    // password should not be in output
  })
  
  yield* expectInspectableMatchesSnapshot(user, "user-creation")
})
```

## Integration Examples

### Integration with Popular Logging Libraries

```typescript
import { Inspectable, Effect, Logger } from "effect"

// Winston integration
const createWinstonInspectableLogger = (winston: any) => {
  return Logger.make<unknown>(({ message, level, spans, annotations }) => {
    const logData = {
      message: Inspectable.toStringUnknown(message),
      level,
      spans: spans.map(span => ({
        name: span.name,
        attributes: Object.fromEntries(
          Object.entries(span.attributes).map(([key, value]) => [
            key,
            Inspectable.toStringUnknown(value)
          ])
        )
      })),
      annotations: Object.fromEntries(
        Object.entries(annotations).map(([key, value]) => [
          key, 
          Inspectable.toStringUnknown(value)
        ])
      )
    }

    winston[level](logData)
  })
}

// Pino integration  
const createPinoInspectableLogger = (pino: any) => {
  return Logger.make<unknown>(({ message, level, spans, annotations }) => {
    pino[level]({
      msg: Inspectable.toStringUnknown(message),
      spans: spans.map(span => span.name),
      ...Object.fromEntries(
        Object.entries(annotations).map(([key, value]) => [
          key,
          Inspectable.toStringUnknown(value, 0) // Compact for structured logging
        ])
      )
    })
  })
}
```

### Node.js REPL and Debugging Integration

```typescript
import { Inspectable } from "effect"
import * as util from "node:util"

// Enhanced REPL experience
class EnhancedUser extends Inspectable.Class {
  constructor(
    public id: string,
    public email: string,
    private password: string
  ) {
    super()
  }

  toJSON() {
    return {
      _id: "User",
      id: this.id,
      email: this.email
      // password excluded
    }
  }

  // Custom Node.js inspector integration
  [util.inspect.custom](depth: number, options: util.InspectOptions) {
    if (depth < 0) return "[User]"

    const obj = this.toJSON()
    return util.inspect(obj, {
      ...options,
      colors: true,
      depth: depth - 1
    })
  }
}

// REPL helper functions
const inspect = (value: unknown, depth = 2) => {
  console.log(util.inspect(value, { 
    colors: true, 
    depth,
    showHidden: false,
    customInspect: true
  }))
}

const inspectJson = (value: unknown) => {
  console.log(Inspectable.toStringUnknown(value, 2))
}

// Usage in Node.js REPL:
// > const user = new EnhancedUser("123", "john@example.com", "secret")
// > inspect(user)
// User {
//   _id: 'User',
//   id: '123', 
//   email: 'john@example.com'
// }
// > inspectJson(user)  
// {
//   "_id": "User",
//   "id": "123",
//   "email": "john@example.com"
// }
```

### Testing Integration with Vitest/Jest

```typescript
import { Inspectable, Effect } from "effect"
import { expect, test } from "vitest"

// Custom matcher for inspectable objects
expect.extend({
  toHaveInspectableShape(received: Inspectable.Inspectable, expected: unknown) {
    const actualJson = received.toJSON()
    const expectedJson = Inspectable.toJSON(expected)
    
    const pass = JSON.stringify(actualJson) === JSON.stringify(expectedJson)
    
    if (pass) {
      return {
        message: () => 
          `Expected inspectable not to match shape:\n${Inspectable.format(expectedJson)}`,
        pass: true
      }
    } else {
      return {
        message: () => 
          `Expected inspectable to match shape:\n` +
          `Expected: ${Inspectable.format(expectedJson)}\n` +
          `Received: ${Inspectable.format(actualJson)}`,
        pass: false
      }
    }
  }
})

// Test helpers
const runTestEffect = <A, E>(effect: Effect.Effect<A, E>) =>
  Effect.runPromise(Effect.provide(effect, TestContext.TestContext))

// Usage in tests
test("user serialization excludes password", async () => {
  const user = new User("123", "john@example.com", "secret123")
  
  expect(user).toHaveInspectableShape({
    _id: "User",
    id: "123",
    email: "john@example.com"
  })
})

test("order processing creates proper audit trail", async () => {
  await runTestEffect(Effect.gen(function* () {
    const order = yield* processOrder("order-123")
    
    expect(order).toHaveInspectableShape({
      _id: "Order",
      id: "order-123",
      status: "processed",
      // ... other expected fields
    })
  }))
})
```

## Conclusion

Inspectable provides **secure debugging**, **consistent serialization**, and **seamless Node.js integration** for Effect applications.

Key benefits:
- **Security First**: Control exactly what data appears in logs and debugging output
- **Developer Experience**: Rich, formatted output in REPL and debugging tools
- **Consistency**: Unified interface across all your domain objects for inspection

Inspectable is essential when building production Effect applications that need secure, debuggable, and maintainable object representations across different environments.