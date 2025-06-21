# FiberRef: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem FiberRef Solves

In concurrent applications, managing context and state across async operations is challenging. Traditional approaches like global variables or thread-local storage fall short:

```typescript
// Traditional approach - global state is shared across all operations
let currentUserId: string | null = null;
let requestId: string | null = null;
let logContext: Record<string, any> = {};

async function processRequest(userId: string, reqId: string) {
  // Setting global state - race conditions with concurrent requests
  currentUserId = userId;
  requestId = reqId;
  logContext = { userId, requestId: reqId };
  
  // These operations share global state - dangerous!
  await Promise.all([
    processPayment(),
    sendNotification(),
    updateAnalytics()
  ]);
  
  // Cleanup - but what if another request started?
  currentUserId = null;
  requestId = null;
  logContext = {};
}

async function processPayment() {
  // Which user are we processing? Race condition!
  console.log(`Processing payment for user: ${currentUserId}`);
  
  // If another request modifies currentUserId, we're in trouble
  await simulatePaymentProcess();
}
```

This approach leads to:
- **Race Conditions** - Concurrent requests overwrite each other's context
- **Memory Leaks** - Global state isn't properly cleaned up
- **Testing Complexity** - Global state makes tests non-deterministic
- **Debugging Nightmares** - Hard to track which operation modified state

### The FiberRef Solution

FiberRef provides fiber-local storage that is automatically inherited by child fibers and isolated between concurrent operations:

```typescript
import { Effect, FiberRef } from "effect";

// Create fiber-local storage for user context
const makeUserContextRef = Effect.gen(function* () {
  return yield* FiberRef.make<{
    userId: string;
    requestId: string;
    logContext: Record<string, any>;
  } | null>(null);
});

const processRequest = (userId: string, reqId: string) =>
  Effect.gen(function* () {
    const userContextRef = yield* makeUserContextRef;
    
    // Set context for this fiber and all its children
    yield* FiberRef.set(userContextRef, {
      userId,
      requestId: reqId,
      logContext: { userId, requestId: reqId }
    });
    
    // All child operations inherit this context automatically
    yield* Effect.all([
      processPayment(userContextRef),
      sendNotification(userContextRef),
      updateAnalytics(userContextRef)
    ], { concurrency: "unbounded" });
  });

const processPayment = (userContextRef: FiberRef.FiberRef<any>) =>
  Effect.gen(function* () {
    const context = yield* FiberRef.get(userContextRef);
    console.log(`Processing payment for user: ${context?.userId}`);
    
    // Context is guaranteed to be from the correct request
    yield* simulatePaymentProcessEffect();
  });
```

### Key Concepts

**Fiber-Local Storage**: Each fiber maintains its own isolated copy of FiberRef values, preventing interference between concurrent operations.

**Automatic Inheritance**: Child fibers automatically inherit FiberRef values from their parent, creating a hierarchical context structure.

**Fork Semantics**: When a fiber forks, you can control how values are propagated using custom fork and join strategies.

**Scoped Lifecycle**: FiberRef values are automatically cleaned up when fibers complete, preventing memory leaks.

## Basic Usage Patterns

### Pattern 1: Creating and Using FiberRef

```typescript
import { Effect, FiberRef, pipe } from "effect";

// Create a simple FiberRef
const createCounterRef = Effect.gen(function* () {
  return yield* FiberRef.make(0);
});

const incrementCounter = (counterRef: FiberRef.FiberRef<number>) =>
  FiberRef.update(counterRef, (n) => n + 1);

const getCounterValue = (counterRef: FiberRef.FiberRef<number>) =>
  FiberRef.get(counterRef);

// Usage
const program = Effect.gen(function* () {
  const counterRef = yield* createCounterRef;
  
  // Set initial value
  yield* FiberRef.set(counterRef, 10);
  
  // Update the value
  yield* incrementCounter(counterRef);
  yield* incrementCounter(counterRef);
  
  // Get the final value
  const finalValue = yield* getCounterValue(counterRef);
  console.log(`Final counter value: ${finalValue}`); // 12
});
```

### Pattern 2: Request Context Propagation

```typescript
import { Effect, FiberRef, Context } from "effect";

// Define request context structure
interface RequestContext {
  requestId: string;
  userId?: string;
  traceId: string;
  startTime: number;
}

// Create a service for request context
class RequestContextService extends Context.Tag("RequestContextService")<
  RequestContextService,
  {
    readonly ref: FiberRef.FiberRef<RequestContext | null>;
    readonly get: Effect.Effect<RequestContext | null>;
    readonly set: (context: RequestContext) => Effect.Effect<void>;
    readonly update: (f: (ctx: RequestContext | null) => RequestContext | null) => Effect.Effect<void>;
  }
>() {
  static Live = Effect.gen(function* () {
    const ref = yield* FiberRef.make<RequestContext | null>(null);
    
    return {
      ref,
      get: FiberRef.get(ref),
      set: (context: RequestContext) => FiberRef.set(ref, context),
      update: (f: (ctx: RequestContext | null) => RequestContext | null) => 
        FiberRef.update(ref, f)
    };
  }).pipe(Effect.map(RequestContextService.of));
}

// Helper to create request context
const withRequestContext = <A, E, R>(
  context: RequestContext,
  effect: Effect.Effect<A, E, R>
): Effect.Effect<A, E, R | RequestContextService> =>
  Effect.gen(function* () {
    const contextService = yield* RequestContextService;
    yield* contextService.set(context);
    return yield* effect;
  });

// Usage in a request handler
const handleRequest = (requestId: string, userId?: string) =>
  withRequestContext(
    {
      requestId,
      userId,
      traceId: `trace-${requestId}`,
      startTime: Date.now()
    },
    Effect.gen(function* () {
      // All operations in this effect have access to request context
      yield* processBusinessLogic();
      yield* logRequestCompletion();
    })
  );

const processBusinessLogic = () =>
  Effect.gen(function* () {
    const contextService = yield* RequestContextService;
    const context = yield* contextService.get;
    
    console.log(`Processing business logic for request: ${context?.requestId}`);
    
    // Spawn concurrent operations - they inherit the context
    yield* Effect.all([
      validateInput(),
      fetchUserData(),
      updateDatabase()
    ], { concurrency: "unbounded" });
  });
```

### Pattern 3: Logging Context

```typescript
import { Effect, FiberRef, pipe } from "effect";

// Create a structured logging context
interface LogContext {
  correlationId: string;
  service: string;
  operation: string;
  metadata: Record<string, any>;
}

const createLogContextRef = Effect.gen(function* () {
  return yield* FiberRef.make<LogContext>({
    correlationId: "default",
    service: "unknown",
    operation: "unknown",
    metadata: {}
  });
});

// Enhanced logging that includes context
const logWithContext = (level: "info" | "error" | "warn", message: string) =>
  Effect.gen(function* () {
    const logContextRef = yield* createLogContextRef;
    const context = yield* FiberRef.get(logContextRef);
    
    const logEntry = {
      timestamp: new Date().toISOString(),
      level,
      message,
      ...context
    };
    
    console.log(JSON.stringify(logEntry, null, 2));
  });

// Helper to add metadata to log context
const addLogMetadata = (key: string, value: any) =>
  Effect.gen(function* () {
    const logContextRef = yield* createLogContextRef;
    yield* FiberRef.update(logContextRef, (ctx) => ({
      ...ctx,
      metadata: { ...ctx.metadata, [key]: value }
    }));
  });

// Usage example
const processOrder = (orderId: string) =>
  Effect.gen(function* () {
    const logContextRef = yield* createLogContextRef;
    
    // Set operation context
    yield* FiberRef.set(logContextRef, {
      correlationId: `order-${orderId}`,
      service: "order-service",
      operation: "processOrder",
      metadata: { orderId }
    });
    
    yield* logWithContext("info", "Starting order processing");
    
    // Add metadata as processing continues
    yield* addLogMetadata("step", "validation");
    yield* logWithContext("info", "Validating order");
    
    yield* addLogMetadata("step", "payment");
    yield* logWithContext("info", "Processing payment");
    
    // Child operations inherit the logging context
    yield* Effect.fork(
      Effect.gen(function* () {
        yield* addLogMetadata("childOperation", "inventory-check");
        yield* logWithContext("info", "Checking inventory");
      })
    );
  });
```

## Real-World Examples

### Example 1: Database Transaction Context

```typescript
import { Effect, FiberRef, Context, pipe } from "effect";

// Database transaction context
interface TransactionContext {
  transactionId: string;
  isolationLevel: "READ_COMMITTED" | "SERIALIZABLE";
  isReadOnly: boolean;
  startedAt: number;
}

// Database connection service
class DatabaseService extends Context.Tag("DatabaseService")<
  DatabaseService,
  {
    readonly beginTransaction: (
      isolationLevel?: TransactionContext["isolationLevel"]
    ) => Effect.Effect<TransactionContext>;
    readonly commit: (ctx: TransactionContext) => Effect.Effect<void>;
    readonly rollback: (ctx: TransactionContext) => Effect.Effect<void>;
    readonly query: <T>(sql: string, params?: any[]) => Effect.Effect<T>;
  }
>() {}

// Transaction context service
class TransactionContextService extends Context.Tag("TransactionContextService")<
  TransactionContextService,
  {
    readonly ref: FiberRef.FiberRef<TransactionContext | null>;
    readonly get: Effect.Effect<TransactionContext | null>;
    readonly set: (ctx: TransactionContext) => Effect.Effect<void>;
    readonly getCurrentOrFail: Effect.Effect<TransactionContext>;
  }
>() {
  static Live = Effect.gen(function* () {
    const ref = yield* FiberRef.make<TransactionContext | null>(null);
    
    return {
      ref,
      get: FiberRef.get(ref),
      set: (ctx: TransactionContext) => FiberRef.set(ref, ctx),
      getCurrentOrFail: Effect.gen(function* () {
        const ctx = yield* FiberRef.get(ref);
        if (!ctx) {
          return yield* Effect.fail(new Error("No active transaction"));
        }
        return ctx;
      })
    };
  }).pipe(Effect.map(TransactionContextService.of));
}

// Higher-order function to run operations in a transaction
const withTransaction = <A, E, R>(
  effect: Effect.Effect<A, E, R>,
  isolationLevel: TransactionContext["isolationLevel"] = "READ_COMMITTED"
): Effect.Effect<A, E | Error, R | DatabaseService | TransactionContextService> =>
  Effect.gen(function* () {
    const db = yield* DatabaseService;
    const txContext = yield* TransactionContextService;
    
    // Begin transaction
    const tx = yield* db.beginTransaction(isolationLevel);
    yield* txContext.set(tx);
    
    try {
      // Execute the effect within transaction context
      const result = yield* effect;
      
      // Commit on success
      yield* db.commit(tx);
      return result;
    } catch (error) {
      // Rollback on error
      yield* db.rollback(tx);
      throw error;
    }
  });

// Database operations that use transaction context
const insertUser = (userData: { name: string; email: string }) =>
  Effect.gen(function* () {
    const db = yield* DatabaseService;
    const txContext = yield* TransactionContextService;
    const tx = yield* txContext.getCurrentOrFail;
    
    console.log(`Inserting user in transaction: ${tx.transactionId}`);
    
    const userId = yield* db.query<{ id: number }>(
      "INSERT INTO users (name, email) VALUES (?, ?) RETURNING id",
      [userData.name, userData.email]
    );
    
    return userId;
  });

const insertUserProfile = (userId: number, profileData: { bio: string }) =>
  Effect.gen(function* () {
    const db = yield* DatabaseService;
    const txContext = yield* TransactionContextService;
    const tx = yield* txContext.getCurrentOrFail;
    
    console.log(`Inserting profile in transaction: ${tx.transactionId}`);
    
    return yield* db.query(
      "INSERT INTO user_profiles (user_id, bio) VALUES (?, ?)",
      [userId, profileData.bio]
    );
  });

// Business logic using transactions
const createUserWithProfile = (
  userData: { name: string; email: string },
  profileData: { bio: string }
) =>
  withTransaction(
    Effect.gen(function* () {
      // Both operations run in the same transaction
      const user = yield* insertUser(userData);
      yield* insertUserProfile(user.id, profileData);
      
      // Additional operations can be spawned and they inherit the transaction
      yield* Effect.fork(
        Effect.gen(function* () {
          const txContext = yield* TransactionContextService;
          const tx = yield* txContext.getCurrentOrFail;
          console.log(`Audit log in transaction: ${tx.transactionId}`);
          
          yield* db.query(
            "INSERT INTO audit_log (action, entity_id) VALUES (?, ?)",
            ["user_created", user.id]
          );
        })
      );
      
      return user;
    }),
    "SERIALIZABLE"
  );
```

### Example 2: Multi-tenant Application Context

```typescript
import { Effect, FiberRef, Context, pipe } from "effect";

// Tenant context
interface TenantContext {
  tenantId: string;
  tenantName: string;
  features: string[];
  limits: {
    maxUsers: number;
    maxStorage: number;
  };
}

// Security context
interface SecurityContext {
  userId: string;
  roles: string[];
  permissions: string[];
  sessionId: string;
}

// Combined application context
interface AppContext {
  tenant: TenantContext;
  security: SecurityContext;
  request: {
    ip: string;
    userAgent: string;
    timestamp: number;
  };
}

// Application context service
class AppContextService extends Context.Tag("AppContextService")<
  AppContextService,
  {
    readonly ref: FiberRef.FiberRef<AppContext | null>;
    readonly get: Effect.Effect<AppContext | null>;
    readonly set: (ctx: AppContext) => Effect.Effect<void>;
    readonly getTenant: Effect.Effect<TenantContext>;
    readonly getSecurity: Effect.Effect<SecurityContext>;
    readonly hasPermission: (permission: string) => Effect.Effect<boolean>;
    readonly checkLimit: (resource: keyof TenantContext["limits"]) => Effect.Effect<boolean>;
  }
>() {
  static Live = Effect.gen(function* () {
    const ref = yield* FiberRef.make<AppContext | null>(null);
    
    return {
      ref,
      get: FiberRef.get(ref),
      set: (ctx: AppContext) => FiberRef.set(ref, ctx),
      getTenant: Effect.gen(function* () {
        const ctx = yield* FiberRef.get(ref);
        if (!ctx) return yield* Effect.fail(new Error("No app context"));
        return ctx.tenant;
      }),
      getSecurity: Effect.gen(function* () {
        const ctx = yield* FiberRef.get(ref);
        if (!ctx) return yield* Effect.fail(new Error("No app context"));
        return ctx.security;
      }),
      hasPermission: (permission: string) =>
        Effect.gen(function* () {
          const security = yield* this.getSecurity;
          return security.permissions.includes(permission);
        }),
      checkLimit: (resource: keyof TenantContext["limits"]) =>
        Effect.gen(function* () {
          const tenant = yield* this.getTenant;
          // Implementation would check current usage against limits
          return true; // Simplified
        })
    };
  }).pipe(Effect.map(AppContextService.of));
}

// Middleware to establish app context
const withAppContext = <A, E, R>(
  tenantId: string,
  userId: string,
  requestInfo: { ip: string; userAgent: string },
  effect: Effect.Effect<A, E, R>
): Effect.Effect<A, E | Error, R | AppContextService> =>
  Effect.gen(function* () {
    const appContext = yield* AppContextService;
    
    // In a real app, you'd fetch this from databases
    const tenant: TenantContext = {
      tenantId,
      tenantName: `Tenant ${tenantId}`,
      features: ["feature1", "feature2"],
      limits: { maxUsers: 100, maxStorage: 1000000 }
    };
    
    const security: SecurityContext = {
      userId,
      roles: ["user"],
      permissions: ["read", "write"],
      sessionId: `session-${userId}-${Date.now()}`
    };
    
    const appCtx: AppContext = {
      tenant,
      security,
      request: {
        ...requestInfo,
        timestamp: Date.now()
      }
    };
    
    yield* appContext.set(appCtx);
    return yield* effect;
  });

// Business operations that use app context
const getUsersByTenant = () =>
  Effect.gen(function* () {
    const appContext = yield* AppContextService;
    const hasPermission = yield* appContext.hasPermission("read");
    
    if (!hasPermission) {
      return yield* Effect.fail(new Error("Permission denied"));
    }
    
    const tenant = yield* appContext.getTenant;
    console.log(`Fetching users for tenant: ${tenant.tenantId}`);
    
    // Query would be scoped to tenant
    return [
      { id: 1, name: "User 1", tenantId: tenant.tenantId },
      { id: 2, name: "User 2", tenantId: tenant.tenantId }
    ];
  });

const createUser = (userData: { name: string; email: string }) =>
  Effect.gen(function* () {
    const appContext = yield* AppContextService;
    
    // Check permissions
    const hasPermission = yield* appContext.hasPermission("write");
    if (!hasPermission) {
      return yield* Effect.fail(new Error("Permission denied"));
    }
    
    // Check tenant limits
    const withinLimits = yield* appContext.checkLimit("maxUsers");
    if (!withinLimits) {
      return yield* Effect.fail(new Error("User limit exceeded"));
    }
    
    const tenant = yield* appContext.getTenant;
    const security = yield* appContext.getSecurity;
    
    console.log(
      `Creating user for tenant ${tenant.tenantId} by user ${security.userId}`
    );
    
    // User creation logic with tenant isolation
    return {
      id: Date.now(),
      ...userData,
      tenantId: tenant.tenantId,
      createdBy: security.userId
    };
  });

// API handler that establishes context
const handleCreateUserRequest = (
  tenantId: string,
  userId: string,
  requestInfo: { ip: string; userAgent: string },
  userData: { name: string; email: string }
) =>
  withAppContext(
    tenantId,
    userId,
    requestInfo,
    Effect.gen(function* () {
      // All operations in this scope have access to the app context
      const existingUsers = yield* getUsersByTenant();
      const newUser = yield* createUser(userData);
      
      // Concurrent operations inherit the context
      yield* Effect.fork(
        Effect.gen(function* () {
          const appContext = yield* AppContextService;
          const tenant = yield* appContext.getTenant;
          console.log(`Audit: User created in tenant ${tenant.tenantId}`);
        })
      );
      
      return newUser;
    })
  );
```

### Example 3: Distributed Tracing Context

```typescript
import { Effect, FiberRef, Context, Random, pipe } from "effect";

// Tracing context
interface TraceContext {
  traceId: string;
  spanId: string;
  parentSpanId?: string;
  baggage: Record<string, string>;
  samplingRate: number;
}

// Span information
interface Span {
  traceId: string;
  spanId: string;
  parentSpanId?: string;
  operationName: string;
  startTime: number;
  endTime?: number;
  tags: Record<string, any>;
  logs: Array<{ timestamp: number; message: string; level: string }>;
}

// Tracing service
class TracingService extends Context.Tag("TracingService")<
  TracingService,
  {
    readonly ref: FiberRef.FiberRef<TraceContext | null>;
    readonly getCurrentTrace: Effect.Effect<TraceContext | null>;
    readonly startSpan: (operationName: string, tags?: Record<string, any>) => Effect.Effect<TraceContext>;
    readonly finishSpan: (span: Span) => Effect.Effect<void>;
    readonly addTag: (key: string, value: any) => Effect.Effect<void>;
    readonly addLog: (message: string, level?: string) => Effect.Effect<void>;
    readonly setBaggage: (key: string, value: string) => Effect.Effect<void>;
    readonly getBaggage: (key: string) => Effect.Effect<string | undefined>;
  }
>() {
  static Live = Effect.gen(function* () {
    const ref = yield* FiberRef.make<TraceContext | null>(null);
    const activeSpans = new Map<string, Span>();
    
    return {
      ref,
      getCurrentTrace: FiberRef.get(ref),
      
      startSpan: (operationName: string, tags: Record<string, any> = {}) =>
        Effect.gen(function* () {
          const currentTrace = yield* FiberRef.get(ref);
          const traceId = currentTrace?.traceId ?? yield* generateTraceId();
          const spanId = yield* generateSpanId();
          
          const newTrace: TraceContext = {
            traceId,
            spanId,
            parentSpanId: currentTrace?.spanId,
            baggage: currentTrace?.baggage ?? {},
            samplingRate: currentTrace?.samplingRate ?? 1.0
          };
          
          const span: Span = {
            traceId,
            spanId,
            parentSpanId: currentTrace?.spanId,
            operationName,
            startTime: Date.now(),
            tags,
            logs: []
          };
          
          activeSpans.set(spanId, span);
          yield* FiberRef.set(ref, newTrace);
          
          return newTrace;
        }),
      
      finishSpan: (span: Span) =>
        Effect.gen(function* () {
          span.endTime = Date.now();
          console.log(`Span finished: ${JSON.stringify(span, null, 2)}`);
          activeSpans.delete(span.spanId);
        }),
      
      addTag: (key: string, value: any) =>
        Effect.gen(function* () {
          const trace = yield* FiberRef.get(ref);
          if (trace) {
            const span = activeSpans.get(trace.spanId);
            if (span) {
              span.tags[key] = value;
            }
          }
        }),
      
      addLog: (message: string, level: string = "info") =>
        Effect.gen(function* () {
          const trace = yield* FiberRef.get(ref);
          if (trace) {
            const span = activeSpans.get(trace.spanId);
            if (span) {
              span.logs.push({
                timestamp: Date.now(),
                message,
                level
              });
            }
          }
        }),
      
      setBaggage: (key: string, value: string) =>
        Effect.gen(function* () {
          yield* FiberRef.update(ref, (trace) =>
            trace ? { ...trace, baggage: { ...trace.baggage, [key]: value } } : trace
          );
        }),
      
      getBaggage: (key: string) =>
        Effect.gen(function* () {
          const trace = yield* FiberRef.get(ref);
          return trace?.baggage[key];
        })
    };
  }).pipe(Effect.map(TracingService.of));
}

// Helper functions
const generateTraceId = () =>
  Random.nextIntBetween(0, Number.MAX_SAFE_INTEGER).pipe(
    Effect.map(n => n.toString(16).padStart(16, '0'))
  );

const generateSpanId = () =>
  Random.nextIntBetween(0, Number.MAX_SAFE_INTEGER).pipe(
    Effect.map(n => n.toString(16).padStart(8, '0'))
  );

// Traced operation wrapper
const traced = <A, E, R>(
  operationName: string,
  effect: Effect.Effect<A, E, R>,
  tags: Record<string, any> = {}
): Effect.Effect<A, E, R | TracingService> =>
  Effect.gen(function* () {
    const tracing = yield* TracingService;
    
    // Start new span
    const trace = yield* tracing.startSpan(operationName, tags);
    
    try {
      // Execute the effect
      const result = yield* effect;
      
      // Add success tag
      yield* tracing.addTag("success", true);
      
      return result;
    } catch (error) {
      // Add error information
      yield* tracing.addTag("error", true);
      yield* tracing.addTag("error.message", error instanceof Error ? error.message : String(error));
      yield* tracing.addLog(`Error: ${error}`, "error");
      
      throw error;
    } finally {
      // Finish span
      const span = {
        traceId: trace.traceId,
        spanId: trace.spanId,
        parentSpanId: trace.parentSpanId,
        operationName,
        startTime: Date.now() - 1000, // Mock start time
        tags,
        logs: []
      };
      yield* tracing.finishSpan(span);
    }
  });

// Business operations with tracing
const fetchUser = (userId: string) =>
  traced(
    "fetch-user",
    Effect.gen(function* () {
      const tracing = yield* TracingService;
      
      yield* tracing.addTag("user.id", userId);
      yield* tracing.addLog(`Fetching user ${userId}`);
      
      // Simulate database call
      yield* Effect.sleep("100 millis");
      
      const user = { id: userId, name: `User ${userId}` };
      yield* tracing.addTag("user.name", user.name);
      
      return user;
    }),
    { "db.operation": "select", "db.table": "users" }
  );

const processUserData = (userId: string) =>
  traced(
    "process-user-data",
    Effect.gen(function* () {
      const tracing = yield* TracingService;
      
      // Set baggage that child spans can access
      yield* tracing.setBaggage("user.id", userId);
      
      // Fetch user (creates child span)
      const user = yield* fetchUser(userId);
      
      // Process concurrently (each creates child spans)
      const results = yield* Effect.all([
        traced("validate-user", validateUser(user)),
        traced("enrich-user", enrichUserData(user)),
        traced("audit-user", auditUserAccess(user))
      ], { concurrency: "unbounded" });
      
      return results;
    }),
    { "operation.type": "user-processing" }
  );

const validateUser = (user: { id: string; name: string }) =>
  Effect.gen(function* () {
    const tracing = yield* TracingService;
    
    // Access baggage from parent
    const userId = yield* tracing.getBaggage("user.id");
    yield* tracing.addLog(`Validating user ${userId} from baggage`);
    
    yield* Effect.sleep("50 millis");
    return { valid: true, user };
  });

const enrichUserData = (user: { id: string; name: string }) =>
  Effect.gen(function* () {
    const tracing = yield* TracingService;
    
    yield* tracing.addTag("enrichment.source", "external-api");
    yield* tracing.addLog("Enriching user data from external API");
    
    yield* Effect.sleep("200 millis");
    return { ...user, enriched: true };
  });

const auditUserAccess = (user: { id: string; name: string }) =>
  Effect.gen(function* () {
    const tracing = yield* TracingService;
    
    yield* tracing.addTag("audit.action", "user-access");
    yield* tracing.addLog("Recording user access for audit");
    
    yield* Effect.sleep("30 millis");
    return { audited: true, timestamp: Date.now() };
  });
```

## Advanced Features Deep Dive

### Fork and Join Semantics

FiberRef provides powerful customization of how values are handled when fibers fork and join:

```typescript
import { Effect, FiberRef, Fiber, pipe } from "effect";

// Example 1: Counter with additive join semantics
const createAdditiveCounter = Effect.gen(function* () {
  return yield* FiberRef.make(0, {
    // When forking, child starts with parent's value
    fork: (parentValue) => parentValue,
    // When joining, add child's value to parent's value
    join: (parentValue, childValue) => parentValue + childValue
  });
});

const demonstrateAdditiveCounter = Effect.gen(function* () {
  const counterRef = yield* createAdditiveCounter;
  
  // Set initial value
  yield* FiberRef.set(counterRef, 10);
  
  // Fork multiple fibers that increment the counter
  const fibers = yield* Effect.all([
    Effect.fork(
      Effect.gen(function* () {
        yield* FiberRef.update(counterRef, n => n + 5);
        console.log(`Fiber 1 counter: ${yield* FiberRef.get(counterRef)}`); // 15
      })
    ),
    Effect.fork(
      Effect.gen(function* () {
        yield* FiberRef.update(counterRef, n => n + 3);
        console.log(`Fiber 2 counter: ${yield* FiberRef.get(counterRef)}`); // 13
      })
    ),
    Effect.fork(
      Effect.gen(function* () {
        yield* FiberRef.update(counterRef, n => n + 7);
        console.log(`Fiber 3 counter: ${yield* FiberRef.get(counterRef)}`); // 17
      })
    )
  ]);
  
  // Wait for all fibers to complete
  yield* Effect.all(fibers.map(Fiber.join));
  
  // Parent counter includes all child increments
  const finalValue = yield* FiberRef.get(counterRef);
  console.log(`Final counter: ${finalValue}`); // 10 + 5 + 3 + 7 = 25
});

// Example 2: Set with union join semantics
const createSetRef = <T>(initial: Set<T> = new Set()) =>
  FiberRef.make(initial, {
    fork: (parentSet) => new Set(parentSet),
    join: (parentSet, childSet) => new Set([...parentSet, ...childSet])
  });

const demonstrateSetUnion = Effect.gen(function* () {
  const setRef = yield* createSetRef<string>();
  
  // Initialize with some values
  yield* FiberRef.set(setRef, new Set(["a", "b"]));
  
  // Fork fibers that add different values
  const fibers = yield* Effect.all([
    Effect.fork(
      Effect.gen(function* () {
        yield* FiberRef.update(setRef, set => new Set([...set, "c", "d"]));
      })
    ),
    Effect.fork(
      Effect.gen(function* () {
        yield* FiberRef.update(setRef, set => new Set([...set, "e", "f"]));
      })
    )
  ]);
  
  yield* Effect.all(fibers.map(Fiber.join));
  
  const finalSet = yield* FiberRef.get(setRef);
  console.log(`Final set:`, Array.from(finalSet)); // ["a", "b", "c", "d", "e", "f"]
});

// Example 3: Map with merge join semantics
const createMapRef = <K, V>(initial: Map<K, V> = new Map()) =>
  FiberRef.make(initial, {
    fork: (parentMap) => new Map(parentMap),
    join: (parentMap, childMap) => {
      const result = new Map(parentMap);
      for (const [key, value] of childMap) {
        result.set(key, value);
      }
      return result;
    }
  });

const demonstrateMapMerge = Effect.gen(function* () {
  const mapRef = yield* createMapRef<string, number>();
  
  // Initialize with some values
  yield* FiberRef.set(mapRef, new Map([["initial", 1]]));
  
  // Fork fibers that add different key-value pairs
  const fibers = yield* Effect.all([
    Effect.fork(
      Effect.gen(function* () {
        yield* FiberRef.update(mapRef, map => 
          new Map([...map, ["fiber1", 100], ["shared", 200]])
        );
      })
    ),
    Effect.fork(
      Effect.gen(function* () {
        yield* FiberRef.update(mapRef, map => 
          new Map([...map, ["fiber2", 300], ["shared", 400]]) // shared will be overwritten
        );
      })
    )
  ]);
  
  yield* Effect.all(fibers.map(Fiber.join));
  
  const finalMap = yield* FiberRef.get(mapRef);
  console.log(`Final map:`, Object.fromEntries(finalMap));
  // { initial: 1, fiber1: 100, fiber2: 300, shared: 400 }
});
```

### Scoped FiberRef Management

FiberRef can be scoped to automatically manage lifecycle:

```typescript
import { Effect, FiberRef, Scope, pipe } from "effect";

// Create a scoped FiberRef that automatically cleans up
const createScopedConfig = <T>(initialConfig: T) =>
  Effect.gen(function* () {
    const configRef = yield* FiberRef.make(initialConfig);
    
    // Add cleanup logic
    yield* Effect.addFinalizer(() =>
      Effect.gen(function* () {
        const currentConfig = yield* FiberRef.get(configRef);
        console.log(`Cleaning up config:`, currentConfig);
        // Could save to persistent storage, send metrics, etc.
      })
    );
    
    return configRef;
  });

// Configuration management with automatic cleanup
interface AppConfig {
  debugMode: boolean;
  apiUrl: string;
  timeout: number;
  features: string[];
}

const withConfiguration = <A, E, R>(
  config: AppConfig,
  effect: Effect.Effect<A, E, R>
): Effect.Effect<A, E, R | Scope.Scope> =>
  Effect.gen(function* () {
    const configRef = yield* createScopedConfig(config);
    
    // Make configuration available to child effects
    return yield* Effect.locally(
      configRef,
      config,
      effect
    );
  });

// Business logic that uses scoped configuration
const performBusinessOperation = () =>
  Effect.gen(function* () {
    // Configuration is automatically available
    // Implementation would access configuration through service/context
    console.log("Performing business operation with scoped config");
    
    // Simulate some work
    yield* Effect.sleep("100 millis");
    
    return "Operation completed";
  });

// Usage with automatic cleanup
const runWithScopedConfig = withConfiguration(
  {
    debugMode: true,
    apiUrl: "https://api.example.com",
    timeout: 5000,
    features: ["feature1", "feature2"]
  },
  performBusinessOperation()
).pipe(
  Effect.scoped
);
```

### Custom FiberRef Implementations

You can create specialized FiberRef implementations for complex scenarios:

```typescript
import { Effect, FiberRef, pipe } from "effect";

// Stack-based FiberRef for nested contexts
class StackFiberRef<T> {
  constructor(private ref: FiberRef.FiberRef<T[]>) {}
  
  static create = <T>(initial: T) =>
    Effect.gen(function* () {
      const ref = yield* FiberRef.make([initial], {
        fork: (stack) => [...stack], // Copy the stack
        join: (parentStack, childStack) => parentStack // Keep parent's stack
      });
      return new StackFiberRef(ref);
    });
  
  push = (value: T) =>
    FiberRef.update(this.ref, stack => [...stack, value]);
  
  pop = () =>
    Effect.gen(function* () {
      const stack = yield* FiberRef.get(this.ref);
      if (stack.length <= 1) {
        return yield* Effect.fail(new Error("Cannot pop from empty stack"));
      }
      const newStack = stack.slice(0, -1);
      yield* FiberRef.set(this.ref, newStack);
      return stack[stack.length - 1];
    });
  
  peek = () =>
    Effect.gen(function* () {
      const stack = yield* FiberRef.get(this.ref);
      return stack[stack.length - 1];
    });
  
  getAll = () => FiberRef.get(this.ref);
}

// Usage example: nested execution contexts
interface ExecutionContext {
  name: string;
  metadata: Record<string, any>;
}

const useStackContext = Effect.gen(function* () {
  const contextStack = yield* StackFiberRef.create<ExecutionContext>({
    name: "root",
    metadata: { level: 0 }
  });
  
  // Push nested contexts
  yield* contextStack.push({
    name: "middleware",
    metadata: { level: 1, type: "auth" }
  });
  
  yield* contextStack.push({
    name: "handler",
    metadata: { level: 2, operation: "create-user" }
  });
  
  // Current context
  const current = yield* contextStack.peek();
  console.log(`Current context: ${current.name}`);
  
  // All contexts
  const allContexts = yield* contextStack.getAll();
  console.log(`Context stack:`, allContexts.map(c => c.name)); // ["root", "middleware", "handler"]
  
  // Pop context
  const popped = yield* contextStack.pop();
  console.log(`Popped context: ${popped.name}`); // "handler"
  
  const afterPop = yield* contextStack.peek();
  console.log(`After pop: ${afterPop.name}`); // "middleware"
});

// Histogram FiberRef for metrics collection
class HistogramFiberRef {
  constructor(private ref: FiberRef.FiberRef<number[]>) {}
  
  static create = () =>
    Effect.gen(function* () {
      const ref = yield* FiberRef.make<number[]>([], {
        fork: (values) => [...values], // Copy values to child
        join: (parentValues, childValues) => [...parentValues, ...childValues] // Merge all values
      });
      return new HistogramFiberRef(ref);
    });
  
  record = (value: number) =>
    FiberRef.update(this.ref, values => [...values, value]);
  
  getStatistics = () =>
    Effect.gen(function* () {
      const values = yield* FiberRef.get(this.ref);
      if (values.length === 0) {
        return { count: 0, min: 0, max: 0, avg: 0, sum: 0 };
      }
      
      const sum = values.reduce((a, b) => a + b, 0);
      const avg = sum / values.length;
      const min = Math.min(...values);
      const max = Math.max(...values);
      
      return { count: values.length, min, max, avg, sum };
    });
  
  reset = () => FiberRef.set(this.ref, []);
}

// Usage: collecting metrics across fibers
const collectMetrics = Effect.gen(function* () {
  const histogram = yield* HistogramFiberRef.create();
  
  // Simulate work that records metrics
  const tasks = Array.from({ length: 5 }, (_, i) =>
    Effect.fork(
      Effect.gen(function* () {
        // Simulate some processing time
        const processingTime = Math.random() * 1000;
        yield* Effect.sleep(`${processingTime} millis`);
        
        // Record the processing time
        yield* histogram.record(processingTime);
        
        console.log(`Task ${i} completed in ${processingTime.toFixed(2)}ms`);
      })
    )
  );
  
  // Wait for all tasks
  yield* Effect.all(tasks.map(Fiber.join));
  
  // Get combined statistics
  const stats = yield* histogram.getStatistics();
  console.log(`Performance statistics:`, stats);
});
```

## Practical Patterns & Best Practices

### Pattern 1: Request-Scoped Services

```typescript
import { Effect, FiberRef, Context, pipe } from "effect";

// Create a pattern for request-scoped services that automatically clean up
interface RequestScopedService<T> {
  readonly get: Effect.Effect<T>;
  readonly set: (value: T) => Effect.Effect<void>;
  readonly update: (f: (current: T) => T) => Effect.Effect<void>;
  readonly cleanup: Effect.Effect<void>;
}

const createRequestScopedService = <T>(
  initialValue: T,
  cleanup?: (value: T) => Effect.Effect<void>
): Effect.Effect<RequestScopedService<T>> =>
  Effect.gen(function* () {
    const ref = yield* FiberRef.make(initialValue);
    
    return {
      get: FiberRef.get(ref),
      set: (value: T) => FiberRef.set(ref, value),
      update: (f: (current: T) => T) => FiberRef.update(ref, f),
      cleanup: cleanup ? 
        Effect.gen(function* () {
          const value = yield* FiberRef.get(ref);
          yield* cleanup(value);
        }) : 
        Effect.unit
    };
  });

// Database connection pool per request
interface DatabaseConnection {
  id: string;
  isActive: boolean;
  queries: number;
}

const createDatabaseService = () =>
  createRequestScopedService<DatabaseConnection>(
    { id: "", isActive: false, queries: 0 },
    (connection) =>
      Effect.gen(function* () {
        if (connection.isActive) {
          console.log(`Closing database connection: ${connection.id}`);
          // Close connection logic
        }
      })
  );

// Cache per request
interface RequestCache {
  data: Map<string, any>;
  hits: number;
  misses: number;
}

const createCacheService = () =>
  createRequestScopedService<RequestCache>(
    { data: new Map(), hits: 0, misses: 0 },
    (cache) =>
      Effect.gen(function* () {
        console.log(`Cache stats - Hits: ${cache.hits}, Misses: ${cache.misses}`);
        cache.data.clear();
      })
  );

// Request coordinator that manages multiple services
const withRequestServices = <A, E, R>(
  effect: Effect.Effect<A, E, R>
): Effect.Effect<A, E, R> =>
  Effect.gen(function* () {
    const dbService = yield* createDatabaseService();
    const cacheService = yield* createCacheService();
    
    // Initialize services
    yield* dbService.set({
      id: `conn-${Date.now()}`,
      isActive: true,
      queries: 0
    });
    
    try {
      return yield* effect;
    } finally {
      // Cleanup services
      yield* dbService.cleanup();
      yield* cacheService.cleanup();
    }
  });
```

### Pattern 2: Hierarchical Configuration

```typescript
import { Effect, FiberRef, pipe } from "effect";

// Configuration hierarchy pattern
interface ConfigLayer {
  name: string;
  values: Record<string, any>;
  overrides: Record<string, any>;
}

class HierarchicalConfig {
  constructor(private ref: FiberRef.FiberRef<ConfigLayer[]>) {}
  
  static create = (baseConfig: Record<string, any>) =>
    Effect.gen(function* () {
      const ref = yield* FiberRef.make<ConfigLayer[]>([
        { name: "base", values: baseConfig, overrides: {} }
      ], {
        fork: (layers) => layers.map(layer => ({ ...layer })), // Deep copy
        join: (parentLayers, childLayers) => parentLayers // Keep parent config
      });
      return new HierarchicalConfig(ref);
    });
  
  pushLayer = (name: string, config: Record<string, any>) =>
    FiberRef.update(this.ref, layers => [
      ...layers,
      { name, values: config, overrides: {} }
    ]);
  
  popLayer = () =>
    Effect.gen(function* () {
      const layers = yield* FiberRef.get(this.ref);
      if (layers.length <= 1) {
        return yield* Effect.fail(new Error("Cannot pop base layer"));
      }
      yield* FiberRef.set(this.ref, layers.slice(0, -1));
      return layers[layers.length - 1];
    });
  
  override = (key: string, value: any) =>
    FiberRef.update(this.ref, layers => {
      const newLayers = [...layers];
      const topLayer = { ...newLayers[newLayers.length - 1] };
      topLayer.overrides = { ...topLayer.overrides, [key]: value };
      newLayers[newLayers.length - 1] = topLayer;
      return newLayers;
    });
  
  get = <T>(key: string, defaultValue?: T): Effect.Effect<T> =>
    Effect.gen(function* () {
      const layers = yield* FiberRef.get(this.ref);
      
      // Check overrides first (from top to bottom)
      for (let i = layers.length - 1; i >= 0; i--) {
        if (key in layers[i].overrides) {
          return layers[i].overrides[key];
        }
      }
      
      // Then check values (from top to bottom)
      for (let i = layers.length - 1; i >= 0; i--) {
        if (key in layers[i].values) {
          return layers[i].values[key];
        }
      }
      
      if (defaultValue !== undefined) {
        return defaultValue;
      }
      
      return yield* Effect.fail(new Error(`Configuration key not found: ${key}`));
    });
  
  getAllKeys = () =>
    Effect.gen(function* () {
      const layers = yield* FiberRef.get(this.ref);
      const keys = new Set<string>();
      
      layers.forEach(layer => {
        Object.keys(layer.values).forEach(key => keys.add(key));
        Object.keys(layer.overrides).forEach(key => keys.add(key));
      });
      
      return Array.from(keys);
    });
}

// Usage example
const useHierarchicalConfig = Effect.gen(function* () {
  const config = yield* HierarchicalConfig.create({
    database: { host: "localhost", port: 5432 },
    api: { timeout: 5000, retries: 3 },
    logging: { level: "info" }
  });
  
  // Add environment-specific config
  yield* config.pushLayer("environment", {
    database: { host: "prod-db.example.com" },
    api: { timeout: 10000 }
  });
  
  // Add user-specific overrides
  yield* config.override("logging.level", "debug");
  yield* config.override("api.retries", 5);
  
  // Access configuration
  const dbHost = yield* config.get("database.host");
  const apiTimeout = yield* config.get("api.timeout");
  const logLevel = yield* config.get("logging.level");
  
  console.log(`Database: ${dbHost}`); // "prod-db.example.com"
  console.log(`API Timeout: ${apiTimeout}`); // 10000
  console.log(`Log Level: ${logLevel}`); // "debug"
  
  // Fork a child process with additional config
  yield* Effect.fork(
    Effect.gen(function* () {
      yield* config.pushLayer("child", {
        api: { baseUrl: "https://child-api.example.com" }
      });
      
      const childApiUrl = yield* config.get("api.baseUrl");
      console.log(`Child API URL: ${childApiUrl}`);
      
      // Child has all parent config plus its own
      const childDbHost = yield* config.get("database.host");
      console.log(`Child DB Host: ${childDbHost}`); // "prod-db.example.com"
    })
  );
});
```

### Pattern 3: Event Aggregation

```typescript
import { Effect, FiberRef, pipe } from "effect";

// Event aggregation across fiber boundaries
interface DomainEvent {
  id: string;
  type: string;
  timestamp: number;
  data: any;
  metadata?: Record<string, any>;
}

class EventAggregator {
  constructor(private ref: FiberRef.FiberRef<DomainEvent[]>) {}
  
  static create = () =>
    Effect.gen(function* () {
      const ref = yield* FiberRef.make<DomainEvent[]>([], {
        fork: (events) => [...events], // Child inherits parent events
        join: (parentEvents, childEvents) => [
          ...parentEvents,
          ...childEvents.filter(e => !parentEvents.some(pe => pe.id === e.id))
        ] // Merge unique events
      });
      return new EventAggregator(ref);
    });
  
  emit = (type: string, data: any, metadata?: Record<string, any>) =>
    Effect.gen(function* () {
      const event: DomainEvent = {
        id: `${type}-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`,
        type,
        timestamp: Date.now(),
        data,
        metadata
      };
      
      yield* FiberRef.update(this.ref, events => [...events, event]);
      return event;
    });
  
  getEvents = (filter?: (event: DomainEvent) => boolean) =>
    Effect.gen(function* () {
      const events = yield* FiberRef.get(this.ref);
      return filter ? events.filter(filter) : events;
    });
  
  getEventsByType = (type: string) =>
    this.getEvents(event => event.type === type);
  
  getEventsAfter = (timestamp: number) =>
    this.getEvents(event => event.timestamp > timestamp);
  
  clear = () => FiberRef.set(this.ref, []);
  
  replay = (handler: (event: DomainEvent) => Effect.Effect<void>) =>
    Effect.gen(function* () {
      const events = yield* FiberRef.get(this.ref);
      yield* Effect.forEach(events, handler, { concurrency: 1 });
    });
}

// Usage in a business process
const processOrderWithEvents = (orderId: string) =>
  Effect.gen(function* () {
    const eventAggregator = yield* EventAggregator.create();
    
    // Emit order started event
    yield* eventAggregator.emit("order.started", { orderId });
    
    // Process in parallel - events from all branches are collected
    yield* Effect.all([
      // Validation branch
      Effect.fork(
        Effect.gen(function* () {
          yield* eventAggregator.emit("order.validation.started", { orderId });
          
          // Simulate validation
          yield* Effect.sleep("100 millis");
          
          yield* eventAggregator.emit("order.validation.completed", { 
            orderId,
            result: "valid" 
          });
        })
      ),
      
      // Payment branch
      Effect.fork(
        Effect.gen(function* () {
          yield* eventAggregator.emit("payment.started", { orderId });
          
          // Simulate payment processing
          yield* Effect.sleep("200 millis");
          
          yield* eventAggregator.emit("payment.completed", { 
            orderId,
            amount: 99.99,
            transactionId: "tx-123" 
          });
        })
      ),
      
      // Inventory branch
      Effect.fork(
        Effect.gen(function* () {
          yield* eventAggregator.emit("inventory.reserved", { 
            orderId,
            items: ["item1", "item2"] 
          });
        })
      )
    ], { concurrency: "unbounded" });
    
    // Emit completion event
    yield* eventAggregator.emit("order.completed", { orderId });
    
    // Get all events for this order
    const allEvents = yield* eventAggregator.getEvents();
    console.log(`Total events: ${allEvents.length}`);
    
    // Get specific event types
    const paymentEvents = yield* eventAggregator.getEventsByType("payment.completed");
    console.log(`Payment events:`, paymentEvents);
    
    // Replay events (e.g., for audit or rebuilding state)
    yield* eventAggregator.replay(event =>
      Effect.gen(function* () {
        console.log(`Replaying: ${event.type} at ${new Date(event.timestamp).toISOString()}`);
      })
    );
    
    return allEvents;
  });
```

## Integration Examples

### Integration with React Context

```typescript
import { Effect, FiberRef, Context, Runtime } from "effect";
import React, { createContext, useContext, useEffect, useState } from "react";

// React-Effect bridge for FiberRef
interface ReactFiberRefContext<T> {
  get: () => Promise<T>;
  set: (value: T) => Promise<void>;
  subscribe: (callback: (value: T) => void) => () => void;
}

function createReactFiberRef<T>(
  initialValue: T,
  runtime: Runtime.Runtime<never>
): ReactFiberRefContext<T> {
  let currentValue = initialValue;
  const subscribers = new Set<(value: T) => void>();
  
  const fiberRefEffect = Effect.gen(function* () {
    return yield* FiberRef.make(initialValue);
  });
  
  const fiberRefPromise = Runtime.runPromise(runtime)(fiberRefEffect);
  
  return {
    get: async () => {
      const fiberRef = await fiberRefPromise;
      return Runtime.runPromise(runtime)(FiberRef.get(fiberRef));
    },
    
    set: async (value: T) => {
      const fiberRef = await fiberRefPromise;
      await Runtime.runPromise(runtime)(FiberRef.set(fiberRef, value));
      currentValue = value;
      subscribers.forEach(callback => callback(value));
    },
    
    subscribe: (callback: (value: T) => void) => {
      subscribers.add(callback);
      return () => subscribers.delete(callback);
    }
  };
}

// React context for user preferences
interface UserPreferences {
  theme: "light" | "dark";
  language: string;
  notifications: boolean;
}

const UserPreferencesContext = createContext<ReactFiberRefContext<UserPreferences> | null>(null);

// React hook for using FiberRef
function useFiberRef<T>(context: React.Context<ReactFiberRefContext<T> | null>) {
  const fiberRefContext = useContext(context);
  if (!fiberRefContext) {
    throw new Error("FiberRef context not found");
  }
  
  const [value, setValue] = useState<T | null>(null);
  
  useEffect(() => {
    // Get initial value
    fiberRefContext.get().then(setValue);
    
    // Subscribe to changes
    const unsubscribe = fiberRefContext.subscribe(setValue);
    return unsubscribe;
  }, [fiberRefContext]);
  
  const updateValue = (newValue: T) => {
    fiberRefContext.set(newValue);
  };
  
  return [value, updateValue] as const;
}

// React component using FiberRef
function UserPreferencesProvider({ children }: { children: React.ReactNode }) {
  const [runtime] = useState(() => Runtime.defaultRuntime);
  const [preferencesContext] = useState(() =>
    createReactFiberRef<UserPreferences>(
      { theme: "light", language: "en", notifications: true },
      runtime
    )
  );
  
  return (
    <UserPreferencesContext.Provider value={preferencesContext}>
      {children}
    </UserPreferencesContext.Provider>
  );
}

function UserPreferencesComponent() {
  const [preferences, setPreferences] = useFiberRef(UserPreferencesContext);
  
  if (!preferences) return <div>Loading...</div>;
  
  return (
    <div>
      <h3>User Preferences</h3>
      <label>
        Theme:
        <select
          value={preferences.theme}
          onChange={(e) =>
            setPreferences({
              ...preferences,
              theme: e.target.value as "light" | "dark"
            })
          }
        >
          <option value="light">Light</option>
          <option value="dark">Dark</option>
        </select>
      </label>
      
      <label>
        Language:
        <select
          value={preferences.language}
          onChange={(e) =>
            setPreferences({
              ...preferences,
              language: e.target.value
            })
          }
        >
          <option value="en">English</option>
          <option value="es">Spanish</option>
          <option value="fr">French</option>
        </select>
      </label>
      
      <label>
        <input
          type="checkbox"
          checked={preferences.notifications}
          onChange={(e) =>
            setPreferences({
              ...preferences,
              notifications: e.target.checked
            })
          }
        />
        Enable Notifications
      </label>
    </div>
  );
}

// Usage
function App() {
  return (
    <UserPreferencesProvider>
      <UserPreferencesComponent />
    </UserPreferencesProvider>
  );
}
```

### Testing Strategies

```typescript
import { Effect, FiberRef, TestContext, TestClock, pipe } from "effect";
import { describe, it, expect } from "vitest";

// Test utilities for FiberRef
const createTestFiberRef = <T>(initialValue: T) =>
  Effect.gen(function* () {
    const ref = yield* FiberRef.make(initialValue);
    const testAPI = {
      ref,
      get: () => FiberRef.get(ref),
      set: (value: T) => FiberRef.set(ref, value),
      update: (f: (current: T) => T) => FiberRef.update(ref, f),
      
      // Test helper: assert current value
      assertValue: (expected: T) =>
        Effect.gen(function* () {
          const current = yield* FiberRef.get(ref);
          expect(current).toEqual(expected);
        }),
      
      // Test helper: fork and assert isolation
      forkAndAssert: (childEffect: Effect.Effect<void>, expectedParentValue: T) =>
        Effect.gen(function* () {
          const fiber = yield* Effect.fork(childEffect);
          yield* Fiber.join(fiber);
          yield* testAPI.assertValue(expectedParentValue);
        })
    };
    
    return testAPI;
  });

describe("FiberRef", () => {
  it("should maintain isolation between fibers", () =>
    Effect.gen(function* () {
      const counter = yield* createTestFiberRef(0);
      
      // Parent fiber modifies value
      yield* counter.set(10);
      yield* counter.assertValue(10);
      
      // Child fiber has its own copy
      yield* counter.forkAndAssert(
        Effect.gen(function* () {
          yield* counter.assertValue(10); // Inherits parent value
          yield* counter.set(20); // Modifies child copy
          yield* counter.assertValue(20);
        }),
        10 // Parent value unchanged
      );
    }).pipe(Effect.provide(TestContext.TestContext))
  );
  
  it("should support custom fork/join semantics", () =>
    Effect.gen(function* () {
      // Create FiberRef with additive join
      const additiveRef = yield* FiberRef.make(0, {
        fork: (parent) => parent,
        join: (parent, child) => parent + child
      });
      
      yield* FiberRef.set(additiveRef, 10);
      
      // Fork multiple fibers that increment
      const fibers = yield* Effect.all([
        Effect.fork(FiberRef.update(additiveRef, n => n + 5)),
        Effect.fork(FiberRef.update(additiveRef, n => n + 3)),
        Effect.fork(FiberRef.update(additiveRef, n => n + 2))
      ]);
      
      yield* Effect.all(fibers.map(Fiber.join));
      
      // Parent should have sum of all increments
      const finalValue = yield* FiberRef.get(additiveRef);
      expect(finalValue).toBe(20); // 10 + 5 + 3 + 2
    }).pipe(Effect.provide(TestContext.TestContext))
  );
  
  it("should handle concurrent access correctly", () =>
    Effect.gen(function* () {
      const sharedRef = yield* createTestFiberRef<number[]>([]);
      
      // Launch concurrent fibers that append values
      const fibers = Array.from({ length: 10 }, (_, i) =>
        Effect.fork(
          sharedRef.update(arr => [...arr, i])
        )
      );
      
      yield* Effect.all(fibers.map(Fiber.join));
      
      const finalArray = yield* sharedRef.get();
      
      // Should have all values (order may vary due to concurrency)
      expect(finalArray).toHaveLength(10);
      expect(finalArray.sort()).toEqual([0, 1, 2, 3, 4, 5, 6, 7, 8, 9]);
    }).pipe(Effect.provide(TestContext.TestContext))
  );
  
  it("should work with time-based operations", () =>
    Effect.gen(function* () {
      const timestampRef = yield* createTestFiberRef<number[]>([]);
      
      // Record timestamps at intervals
      yield* Effect.fork(
        Effect.gen(function* () {
          for (let i = 0; i < 5; i++) {
            yield* Effect.sleep("100 millis");
            const now = yield* TestClock.currentTimeMillis;
            yield* timestampRef.update(arr => [...arr, now]);
          }
        })
      );
      
      // Advance test clock
      yield* TestClock.adjust("500 millis");
      
      const timestamps = yield* timestampRef.get();
      expect(timestamps).toHaveLength(5);
      
      // Verify timestamps are spaced correctly
      for (let i = 1; i < timestamps.length; i++) {
        expect(timestamps[i] - timestamps[i - 1]).toBe(100);
      }
    }).pipe(
      Effect.provide(TestContext.TestContext)
    )
  );
});

// Property-based testing with FiberRef
const testFiberRefProperties = () => {
  const gen = fc.record({
    initialValue: fc.integer(),
    operations: fc.array(
      fc.oneof(
        fc.record({ type: fc.constant("set"), value: fc.integer() }),
        fc.record({ type: fc.constant("update"), delta: fc.integer() })
      ),
      { maxLength: 20 }
    )
  });
  
  return fc.asyncProperty(gen, async ({ initialValue, operations }) => {
    const effect = Effect.gen(function* () {
      const ref = yield* FiberRef.make(initialValue);
      let expectedValue = initialValue;
      
      for (const op of operations) {
        if (op.type === "set") {
          yield* FiberRef.set(ref, op.value);
          expectedValue = op.value;
        } else {
          yield* FiberRef.update(ref, n => n + op.delta);
          expectedValue += op.delta;
        }
      }
      
      const finalValue = yield* FiberRef.get(ref);
      expect(finalValue).toBe(expectedValue);
    });
    
    await Effect.runPromise(effect.pipe(Effect.provide(TestContext.TestContext)));
  });
};

// Mock implementations for testing
const createMockFiberRefService = <T>(initialValue: T) => {
  let currentValue = initialValue;
  const history: Array<{ operation: string; value: T; timestamp: number }> = [];
  
  return {
    get: Effect.succeed(currentValue),
    set: (value: T) =>
      Effect.gen(function* () {
        currentValue = value;
        history.push({ operation: "set", value, timestamp: Date.now() });
      }),
    update: (f: (current: T) => T) =>
      Effect.gen(function* () {
        const newValue = f(currentValue);
        currentValue = newValue;
        history.push({ operation: "update", value: newValue, timestamp: Date.now() });
      }),
    getHistory: () => Effect.succeed([...history]),
    reset: () =>
      Effect.gen(function* () {
        currentValue = initialValue;
        history.length = 0;
      })
  };
};
```

## Conclusion

FiberRef provides fiber-local storage that automatically manages context propagation, inheritance, and cleanup in concurrent Effect applications.

Key benefits:
- **Isolation**: Each fiber maintains its own isolated copy of values, preventing race conditions
- **Automatic Inheritance**: Child fibers automatically inherit parent context without explicit passing
- **Custom Semantics**: Fork and join behaviors can be customized for complex aggregation patterns
- **Memory Safety**: Values are automatically cleaned up when fibers complete, preventing leaks

FiberRef is essential for managing request context, logging metadata, configuration, metrics collection, and any state that needs to be scoped to a particular execution context while remaining accessible to child operations.

Use FiberRef when you need thread-local-like behavior in your concurrent Effect applications, especially for cross-cutting concerns like tracing, logging, authentication context, or request-scoped services.