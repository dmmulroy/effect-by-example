# KeyValueStore: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem KeyValueStore Solves

Building applications often requires storing and retrieving key-value pairs for caching, session management, configuration, and more. Traditional approaches lead to scattered implementations with inconsistent error handling and no type safety:

```typescript
// Traditional approach - error-prone and inconsistent
class CacheManager {
  private cache: Map<string, string> = new Map()
  
  async get(key: string): Promise<string | null> {
    try {
      return this.cache.get(key) || null
    } catch (error) {
      console.error('Cache get error:', error)
      return null
    }
  }
  
  async set(key: string, value: any): Promise<void> {
    try {
      // No validation, any value accepted
      this.cache.set(key, JSON.stringify(value))
    } catch (error) {
      console.error('Cache set error:', error)
      // Silent failure
    }
  }
}

// Session storage with no type safety
const session = {
  user: JSON.parse(localStorage.getItem('user') || '{}'),
  preferences: JSON.parse(localStorage.getItem('prefs') || '{}')
}

// File-based config with sync I/O blocking
const config = JSON.parse(fs.readFileSync('config.json', 'utf8'))
```

This approach leads to:
- **No type safety** - Values can be anything, leading to runtime errors
- **Inconsistent error handling** - Errors silently swallowed or logged without recovery
- **Platform-specific code** - Different implementations for browser vs Node.js
- **No validation** - Corrupted data can crash the application
- **Poor testability** - Hard to mock or swap implementations

### The KeyValueStore Solution

Effect's KeyValueStore provides a unified, type-safe, and effectful interface for all key-value storage needs:

```typescript
import { KeyValueStore } from "@effect/platform"
import { Effect, Schema } from "effect"

// Type-safe storage with validation
export function createUserCache() {
  return Effect.gen(function* () {
    const kv = yield* KeyValueStore.KeyValueStore
    const User = Schema.Struct({
      id: Schema.String,
      name: Schema.String,
      email: Schema.Email,
      lastLogin: Schema.DateTimeUtc
    })
    
    // Create a schema-validated store
    const userStore = kv.forSchema(User)
    
    // Type-safe operations with automatic validation
    yield* userStore.set("user:123", {
      id: "123",
      name: "Alice",
      email: "alice@example.com",
      lastLogin: new Date()
    })
    
    const user = yield* userStore.get("user:123")
    return user
  })
}
```

### Key Concepts

**KeyValueStore**: A service that provides asynchronous key-value storage operations with consistent error handling across different backends.

**SchemaStore**: A type-safe wrapper around KeyValueStore that validates data using Effect Schema, ensuring data integrity.

**Storage Layers**: Pluggable implementations (in-memory, file system, browser storage) that can be swapped without changing application code.

**Effect Integration**: Full integration with Effect's error handling, tracing, and dependency injection systems.

## Basic Usage Patterns

### Pattern 1: Basic Store Setup

```typescript
import { KeyValueStore } from "@effect/platform"
import { Effect } from "effect"

// Access the KeyValueStore service
const program = Effect.gen(function* () {
  const kv = yield* KeyValueStore.KeyValueStore
  
  // Basic string storage
  yield* kv.set("app:version", "1.0.0")
  yield* kv.set("app:name", "My Application")
  
  // Retrieve values
  const version = yield* kv.get("app:version")
  const name = yield* kv.get("app:name")
  
  console.log(`${name} v${version}`)
})

// Run with in-memory implementation
import { layerMemory } from "@effect/platform/KeyValueStore"

Effect.runPromise(
  program.pipe(Effect.provide(layerMemory))
)
```

### Pattern 2: Working with Binary Data

```typescript
import { KeyValueStore } from "@effect/platform"
import { Effect } from "effect"

const storeBinaryData = Effect.gen(function* () {
  const kv = yield* KeyValueStore.KeyValueStore
  
  // Store binary data (e.g., images, files)
  const imageData = new Uint8Array([137, 80, 78, 71]) // PNG header
  yield* kv.set("image:logo", imageData)
  
  // Retrieve as Uint8Array
  const retrieved = yield* kv.getUint8Array("image:logo")
  
  return retrieved
})
```

### Pattern 3: Store Operations

```typescript
import { KeyValueStore } from "@effect/platform"
import { Effect, Option } from "effect"

const storeOperations = Effect.gen(function* () {
  const kv = yield* KeyValueStore.KeyValueStore
  
  // Check if empty
  const wasEmpty = yield* kv.isEmpty
  console.log("Store empty:", wasEmpty)
  
  // Add multiple items
  yield* kv.set("user:1", "Alice")
  yield* kv.set("user:2", "Bob")
  yield* kv.set("user:3", "Charlie")
  
  // Check size
  const size = yield* kv.size
  console.log("Total items:", size)
  
  // Check existence
  const hasUser = yield* kv.has("user:2")
  console.log("Has user:2:", hasUser)
  
  // Modify existing value
  yield* kv.modify("user:2", (value) =>
    Option.map(value, (name) => `${name} Smith`)
  )
  
  // Remove specific key
  yield* kv.remove("user:3")
  
  // Clear all entries
  yield* kv.clear
})
```

## Real-World Examples

### Example 1: Application Cache with TTL

Managing application cache with time-to-live (TTL) functionality for API responses:

```typescript
import { KeyValueStore } from "@effect/platform"
import { Effect, Schema, DateTime, Duration } from "effect"

// Cache entry with timestamp
const CacheEntry = Schema.Struct({
  data: Schema.Unknown,
  timestamp: Schema.DateTimeUtc,
  ttl: Schema.DurationFromMillis
})

// Cache manager with TTL support
export class CacheManager {
  constructor(private readonly prefix: string = "cache") {}

  private getCacheKey(key: string) {
    return `${this.prefix}:${key}`
  }

  set(key: string, data: unknown, ttl: Duration.Duration) {
    return Effect.gen(function* () {
      const kv = yield* KeyValueStore.KeyValueStore
      const cache = kv.forSchema(CacheEntry)
      
      yield* cache.set(getCacheKey(key), {
        data,
        timestamp: DateTime.now,
        ttl
      })
    })
  }

  get(key: string) {
    return Effect.gen(function* () {
      const kv = yield* KeyValueStore.KeyValueStore
      const cache = kv.forSchema(CacheEntry)
      
      const entry = yield* cache.get(getCacheKey(key))
      
      return Option.flatMap(entry, (e) => {
        const age = DateTime.distance(e.timestamp, DateTime.now)
        if (Duration.lessThan(age, e.ttl)) {
          return Option.some(e.data)
        }
        // Expired - remove it
        yield* Effect.ignore(cache.remove(getCacheKey(key)))
        return Option.none()
      })
    })
  }

  invalidate(pattern?: string) {
    return Effect.gen(function* () {
      const kv = yield* KeyValueStore.KeyValueStore
      
      if (pattern) {
        // Invalidate keys matching pattern
        // Note: This is a simplified example. Real implementation
        // would need to iterate over keys
        yield* kv.remove(`${this.prefix}:${pattern}`)
      } else {
        // Clear all cache entries
        yield* kv.clear
      }
    })
  }
}

// Usage example
const fetchUserWithCache = (userId: string) => {
  const cache = new CacheManager("api")
  
  return Effect.gen(function* () {
    // Try cache first
    const cached = yield* cache.get(`user:${userId}`)
    
    if (Option.isSome(cached)) {
      console.log("Cache hit!")
      return cached.value
    }
    
    console.log("Cache miss - fetching from API")
    // Simulate API call
    const userData = yield* fetchUserFromAPI(userId)
    
    // Store in cache with 5 minute TTL
    yield* cache.set(
      `user:${userId}`,
      userData,
      Duration.minutes(5)
    )
    
    return userData
  })
}
```

### Example 2: Session Management

Implementing secure session storage with automatic expiration:

```typescript
import { KeyValueStore } from "@effect/platform"
import { Effect, Schema, DateTime, Duration, Option, Redacted } from "effect"
import * as Crypto from "node:crypto"

// Session schema with security features
const Session = Schema.Struct({
  id: Schema.String,
  userId: Schema.String,
  data: Schema.Record(Schema.String, Schema.Unknown),
  createdAt: Schema.DateTimeUtc,
  expiresAt: Schema.DateTimeUtc,
  lastAccessedAt: Schema.DateTimeUtc
})

type Session = typeof Session.Type

// Session manager service
export class SessionManager {
  private readonly sessionPrefix = "session"
  private readonly defaultTTL = Duration.hours(24)

  generateSessionId(): string {
    return Crypto.randomBytes(32).toString('hex')
  }

  create(userId: string, data: Record<string, unknown> = {}) {
    return Effect.gen(function* () {
      const kv = yield* KeyValueStore.KeyValueStore
      const store = kv.forSchema(Session)
      
      const sessionId = this.generateSessionId()
      const now = DateTime.now
      const expiresAt = DateTime.add(now, this.defaultTTL)
      
      const session: Session = {
        id: sessionId,
        userId,
        data,
        createdAt: now,
        expiresAt,
        lastAccessedAt: now
      }
      
      yield* store.set(
        `${this.sessionPrefix}:${sessionId}`,
        session
      )
      
      return { sessionId, expiresAt }
    })
  }

  get(sessionId: string) {
    return Effect.gen(function* () {
      const kv = yield* KeyValueStore.KeyValueStore
      const store = kv.forSchema(Session)
      
      const session = yield* store.get(
        `${this.sessionPrefix}:${sessionId}`
      )
      
      return Option.flatMap(session, (s) => {
        // Check if expired
        if (DateTime.lessThanOrEqualTo(DateTime.now, s.expiresAt)) {
          return Option.some(s)
        }
        
        // Session expired - remove it
        yield* Effect.ignore(
          store.remove(`${this.sessionPrefix}:${sessionId}`)
        )
        return Option.none()
      })
    })
  }

  touch(sessionId: string) {
    return Effect.gen(function* () {
      const kv = yield* KeyValueStore.KeyValueStore
      const store = kv.forSchema(Session)
      
      yield* store.modify(
        `${this.sessionPrefix}:${sessionId}`,
        (session) => Option.map(session, (s) => ({
          ...s,
          lastAccessedAt: DateTime.now
        }))
      )
    })
  }

  setData(sessionId: string, key: string, value: unknown) {
    return Effect.gen(function* () {
      const kv = yield* KeyValueStore.KeyValueStore
      const store = kv.forSchema(Session)
      
      yield* store.modify(
        `${this.sessionPrefix}:${sessionId}`,
        (session) => Option.map(session, (s) => ({
          ...s,
          data: { ...s.data, [key]: value },
          lastAccessedAt: DateTime.now
        }))
      )
    })
  }

  destroy(sessionId: string) {
    return Effect.gen(function* () {
      const kv = yield* KeyValueStore.KeyValueStore
      const store = kv.forSchema(Session)
      
      yield* store.remove(`${this.sessionPrefix}:${sessionId}`)
    })
  }

  // Clean up expired sessions
  cleanup() {
    return Effect.gen(function* () {
      const kv = yield* KeyValueStore.KeyValueStore
      const store = kv.forSchema(Session)
      
      // Note: In production, you'd iterate over all sessions
      // This is a simplified example
      console.log("Cleaning up expired sessions...")
    })
  }
}

// Usage with HTTP middleware
const sessionMiddleware = (sessionManager: SessionManager) => {
  return <R, E>(
    handler: (session: Session) => Effect.Effect<Response, E, R>
  ) => {
    return (request: Request) => Effect.gen(function* () {
      const sessionId = getSessionIdFromCookie(request)
      
      if (!sessionId) {
        return new Response("Unauthorized", { status: 401 })
      }
      
      const session = yield* sessionManager.get(sessionId)
      
      if (Option.isNone(session)) {
        return new Response("Session expired", { status: 401 })
      }
      
      // Update last accessed time
      yield* sessionManager.touch(sessionId)
      
      return yield* handler(session.value)
    })
  }
}
```

### Example 3: Feature Flags Store

Implementing a feature flag system with runtime configuration:

```typescript
import { KeyValueStore } from "@effect/platform"
import { Effect, Schema, Option, Match } from "effect"

// Feature flag configuration
const FeatureFlag = Schema.Struct({
  name: Schema.String,
  enabled: Schema.Boolean,
  rolloutPercentage: Schema.optional(Schema.Number.pipe(Schema.between(0, 100))),
  enabledForUsers: Schema.optional(Schema.Array(Schema.String)),
  metadata: Schema.optional(Schema.Record(
    Schema.String,
    Schema.Unknown
  ))
})

type FeatureFlag = typeof FeatureFlag.Type

// Feature flag evaluation context
interface FlagContext {
  userId?: string
  properties?: Record<string, unknown>
}

export class FeatureFlagStore {
  private readonly prefix = "feature"

  set(flag: FeatureFlag) {
    return Effect.gen(function* () {
      const kv = yield* KeyValueStore.KeyValueStore
      const store = kv.forSchema(FeatureFlag)
      
      yield* store.set(`${this.prefix}:${flag.name}`, flag)
    })
  }

  get(name: string) {
    return Effect.gen(function* () {
      const kv = yield* KeyValueStore.KeyValueStore
      const store = kv.forSchema(FeatureFlag)
      
      return yield* store.get(`${this.prefix}:${name}`)
    })
  }

  isEnabled(name: string, context?: FlagContext) {
    return Effect.gen(function* () {
      const flag = yield* this.get(name)
      
      return Option.match(flag, {
        onNone: () => false,
        onSome: (f) => this.evaluateFlag(f, context)
      })
    })
  }

  private evaluateFlag(flag: FeatureFlag, context?: FlagContext): boolean {
    // If globally disabled, return false
    if (!flag.enabled) return false
    
    // Check user-specific enablement
    if (context?.userId && flag.enabledForUsers) {
      if (flag.enabledForUsers.includes(context.userId)) {
        return true
      }
    }
    
    // Check rollout percentage
    if (flag.rolloutPercentage !== undefined && context?.userId) {
      const hash = this.hashUserId(context.userId)
      const bucket = (hash % 100) + 1
      return bucket <= flag.rolloutPercentage
    }
    
    // Default to enabled if no specific rules
    return true
  }

  private hashUserId(userId: string): number {
    // Simple hash function for demo
    let hash = 0
    for (let i = 0; i < userId.length; i++) {
      hash = ((hash << 5) - hash) + userId.charCodeAt(i)
      hash = hash & hash // Convert to 32bit integer
    }
    return Math.abs(hash)
  }

  // Bulk operations
  setMany(flags: FeatureFlag[]) {
    return Effect.forEach(flags, (flag) => this.set(flag), {
      concurrency: "unbounded"
    })
  }

  getAllFlags() {
    return Effect.gen(function* () {
      const kv = yield* KeyValueStore.KeyValueStore
      const store = kv.forSchema(FeatureFlag)
      
      // Note: In real implementation, you'd iterate over keys
      // This is a simplified example
      return []
    })
  }
}

// Usage example with feature flag checks
const processPayment = (userId: string, amount: number) => {
  const flags = new FeatureFlagStore()
  
  return Effect.gen(function* () {
    // Check if new payment system is enabled
    const useNewPaymentSystem = yield* flags.isEnabled(
      "new-payment-system",
      { userId }
    )
    
    if (useNewPaymentSystem) {
      console.log("Using new payment system")
      return yield* processWithNewSystem(userId, amount)
    } else {
      console.log("Using legacy payment system")
      return yield* processWithLegacySystem(userId, amount)
    }
  })
}

// Admin interface for managing flags
const toggleFeature = (name: string, enabled: boolean) => {
  const flags = new FeatureFlagStore()
  
  return Effect.gen(function* () {
    const existing = yield* flags.get(name)
    
    yield* Option.match(existing, {
      onNone: () => flags.set({ name, enabled }),
      onSome: (flag) => flags.set({ ...flag, enabled })
    })
  })
}
```

## Advanced Features Deep Dive

### Feature 1: Schema-Validated Storage

SchemaStore provides type-safe storage with automatic validation and serialization:

#### Basic SchemaStore Usage

```typescript
import { KeyValueStore } from "@effect/platform"
import { Effect, Schema } from "effect"

// Define complex domain models
const Product = Schema.Struct({
  id: Schema.String,
  name: Schema.String,
  price: Schema.Number,
  categories: Schema.Array(Schema.String),
  metadata: Schema.Record(Schema.String, Schema.Unknown),
  createdAt: Schema.DateTimeUtc,
  updatedAt: Schema.DateTimeUtc
})

const createProductStore = Effect.gen(function* () {
  const kv = yield* KeyValueStore.KeyValueStore
  
  // Create a schema-validated store
  const productStore = kv.forSchema(Product)
  
  // All operations are now type-safe
  yield* productStore.set("product:123", {
    id: "123",
    name: "Laptop",
    price: 999.99,
    categories: ["electronics", "computers"],
    metadata: { brand: "TechCorp", warranty: "2 years" },
    createdAt: new Date(),
    updatedAt: new Date()
  })
  
  // Retrieved data is automatically validated
  const product = yield* productStore.get("product:123")
  
  return product
})
```

#### Real-World SchemaStore Example

```typescript
import { KeyValueStore, layerFileSystem } from "@effect/platform/KeyValueStore"
import { Effect, Schema, Array as Arr } from "effect"
import { FileSystem, Path } from "@effect/platform"

// User preferences with nested schemas
const ColorScheme = Schema.Literal("light", "dark", "auto")
const Language = Schema.Literal("en", "es", "fr", "de")

const UserPreferences = Schema.Struct({
  userId: Schema.String,
  theme: ColorScheme,
  language: Language,
  notifications: Schema.Struct({
    email: Schema.Boolean,
    push: Schema.Boolean,
    frequency: Schema.Literal("realtime", "hourly", "daily")
  }),
  privacy: Schema.Struct({
    shareAnalytics: Schema.Boolean,
    publicProfile: Schema.Boolean
  })
})

// Preferences service with migrations
export class PreferencesService {
  private readonly store = Effect.gen(function* () {
    const kv = yield* KeyValueStore.KeyValueStore
    return kv.forSchema(UserPreferences)
  })

  getOrCreateDefaults(userId: string) {
    return Effect.gen(function* () {
      const store = yield* this.store
      const existing = yield* store.get(`prefs:${userId}`)
      
      if (Option.isSome(existing)) {
        return existing.value
      }
      
      // Create default preferences
      const defaults: typeof UserPreferences.Type = {
        userId,
        theme: "auto",
        language: "en",
        notifications: {
          email: true,
          push: false,
          frequency: "daily"
        },
        privacy: {
          shareAnalytics: false,
          publicProfile: true
        }
      }
      
      yield* store.set(`prefs:${userId}`, defaults)
      return defaults
    })
  }

  updatePreferences(
    userId: string,
    updates: Partial<typeof UserPreferences.Type>
  ) {
    return Effect.gen(function* () {
      const store = yield* this.store
      
      yield* store.modify(`prefs:${userId}`, (current) =>
        Option.map(current, (prefs) => ({
          ...prefs,
          ...updates,
          // Ensure nested objects are properly merged
          notifications: updates.notifications 
            ? { ...prefs.notifications, ...updates.notifications }
            : prefs.notifications,
          privacy: updates.privacy
            ? { ...prefs.privacy, ...updates.privacy }
            : prefs.privacy
        }))
      )
    })
  }

  exportAllPreferences() {
    return Effect.gen(function* () {
      const store = yield* this.store
      
      // In production, iterate over all preference keys
      // This is simplified for the example
      const allPrefs: Array<typeof UserPreferences.Type> = []
      
      return {
        version: "1.0",
        exportDate: new Date().toISOString(),
        preferences: allPrefs
      }
    })
  }
}

// Run with file system persistence
const program = Effect.gen(function* () {
  const service = new PreferencesService()
  
  // Create/get user preferences
  const prefs = yield* service.getOrCreateDefaults("user123")
  console.log("Current preferences:", prefs)
  
  // Update theme
  yield* service.updatePreferences("user123", {
    theme: "dark"
  })
}).pipe(
  Effect.provide(layerFileSystem("./data/preferences")),
  Effect.provide(FileSystem.layer),
  Effect.provide(Path.layer)
)
```

#### Advanced SchemaStore: Versioned Schemas

```typescript
import { KeyValueStore } from "@effect/platform"
import { Effect, Schema, Option } from "effect"

// Versioned configuration schema
const ConfigV1 = Schema.Struct({
  version: Schema.Literal(1),
  apiUrl: Schema.String,
  timeout: Schema.Number
})

const ConfigV2 = Schema.Struct({
  version: Schema.Literal(2),
  apiUrl: Schema.String,
  timeout: Schema.Number,
  retryPolicy: Schema.Struct({
    maxRetries: Schema.Number,
    backoffMs: Schema.Number
  })
})

// Union of all versions
const Config = Schema.Union(ConfigV1, ConfigV2)

// Migration logic
const migrateConfig = (config: typeof Config.Type): typeof ConfigV2.Type => {
  if (config.version === 1) {
    return {
      ...config,
      version: 2,
      retryPolicy: {
        maxRetries: 3,
        backoffMs: 1000
      }
    }
  }
  return config
}

// Configuration service with automatic migration
export const ConfigService = Effect.gen(function* () {
  const kv = yield* KeyValueStore.KeyValueStore
  const store = kv.forSchema(Config)
  
  const get = Effect.gen(function* () {
    const config = yield* store.get("app:config")
    
    return Option.map(config, (c) => {
      // Always return latest version
      if (c.version < 2) {
        const migrated = migrateConfig(c)
        // Update stored version
        yield* Effect.ignore(store.set("app:config", migrated))
        return migrated
      }
      return c as typeof ConfigV2.Type
    })
  })
  
  const update = (config: typeof ConfigV2.Type) =>
    store.set("app:config", config)
  
  return { get, update } as const
})
```

### Feature 2: Prefixed Stores

Create isolated namespaces within a KeyValueStore:

#### Basic Prefix Usage

```typescript
import { KeyValueStore, prefix } from "@effect/platform/KeyValueStore"
import { Effect } from "effect"

const createNamespacedStores = Effect.gen(function* () {
  const kv = yield* KeyValueStore.KeyValueStore
  
  // Create isolated stores for different domains
  const userStore = prefix(kv, "users")
  const sessionStore = prefix(kv, "sessions")
  const cacheStore = prefix(kv, "cache")
  
  // Each store operates in its own namespace
  yield* userStore.set("123", "Alice")
  yield* sessionStore.set("123", "session-data")
  yield* cacheStore.set("123", "cached-value")
  
  // Keys don't conflict between stores
  const user = yield* userStore.get("123")      // "Alice"
  const session = yield* sessionStore.get("123") // "session-data"
  const cache = yield* cacheStore.get("123")    // "cached-value"
  
  // Clear only affects the prefixed namespace
  yield* cacheStore.clear // Only clears cache:* keys
})
```

#### Real-World Prefix Example: Multi-Tenant Storage

```typescript
import { KeyValueStore, prefix } from "@effect/platform/KeyValueStore"
import { Effect, Schema, Context } from "effect"

// Tenant context
interface Tenant {
  readonly id: string
  readonly name: string
}

const Tenant = Context.GenericTag<Tenant>("Tenant")

// Multi-tenant data schema
const TenantData = Schema.Struct({
  settings: Schema.Record(Schema.String, Schema.Unknown),
  users: Schema.Array(Schema.String),
  subscription: Schema.Struct({
    plan: Schema.Literal("free", "pro", "enterprise"),
    validUntil: Schema.DateTimeUtc
  })
})

// Multi-tenant storage service
export class MultiTenantStorage {
  private getTenantStore(tenantId: string) {
    return Effect.gen(function* () {
      const kv = yield* KeyValueStore.KeyValueStore
      return prefix(kv, `tenant:${tenantId}`)
    })
  }

  // Store data for current tenant
  set(key: string, value: string) {
    return Effect.gen(function* () {
      const tenant = yield* Tenant
      const store = yield* this.getTenantStore(tenant.id)
      
      yield* store.set(key, value)
    })
  }

  // Get data for current tenant
  get(key: string) {
    return Effect.gen(function* () {
      const tenant = yield* Tenant
      const store = yield* this.getTenantStore(tenant.id)
      
      return yield* store.get(key)
    })
  }

  // Tenant-specific schema store
  getTenantDataStore() {
    return Effect.gen(function* () {
      const tenant = yield* Tenant
      const store = yield* this.getTenantStore(tenant.id)
      
      return store.forSchema(TenantData)
    })
  }

  // Admin operation: get data across all tenants
  getAllTenantsData(key: string) {
    return Effect.gen(function* () {
      const kv = yield* KeyValueStore.KeyValueStore
      
      // In production, iterate over all tenant prefixes
      // This is simplified for example
      const tenantIds = ["tenant1", "tenant2", "tenant3"]
      
      return yield* Effect.forEach(tenantIds, (tenantId) =>
        Effect.gen(function* () {
          const store = prefix(kv, `tenant:${tenantId}`)
          const value = yield* store.get(key)
          return { tenantId, value }
        })
      )
    })
  }

  // Clean up tenant data
  deleteTenant(tenantId: string) {
    return Effect.gen(function* () {
      const store = yield* this.getTenantStore(tenantId)
      yield* store.clear
    })
  }
}

// Usage with tenant context
const tenantProgram = Effect.gen(function* () {
  const storage = new MultiTenantStorage()
  
  // Store tenant-specific data
  yield* storage.set("config", "tenant-specific-config")
  
  // Get tenant data store
  const dataStore = yield* storage.getTenantDataStore()
  
  yield* dataStore.set("info", {
    settings: { theme: "dark", locale: "en" },
    users: ["user1", "user2"],
    subscription: {
      plan: "pro",
      validUntil: new Date("2025-12-31")
    }
  })
}).pipe(
  Effect.provideService(Tenant, {
    id: "acme-corp",
    name: "ACME Corporation"
  })
)
```

#### Advanced Prefix: Hierarchical Storage

```typescript
import { KeyValueStore, prefix } from "@effect/platform/KeyValueStore"
import { Effect, pipe } from "effect"

// Create hierarchical storage structure
export class HierarchicalStorage {
  constructor(
    private readonly root: KeyValueStore.KeyValueStore
  ) {}

  // Create nested namespaces
  createNamespace(...path: string[]) {
    return path.reduce((s, segment) => prefix(s, segment), this.root)
  }

  // Example: Organization > Project > Environment
  getProjectStore(orgId: string, projectId: string) {
    return this.createNamespace("orgs", orgId, "projects", projectId)
  }

  getEnvironmentStore(
    orgId: string,
    projectId: string,
    env: "dev" | "staging" | "prod"
  ) {
    return this.createNamespace(
      "orgs", orgId,
      "projects", projectId,
      "environments", env
    )
  }
}

// Usage
const hierarchicalExample = Effect.gen(function* () {
  const kv = yield* KeyValueStore.KeyValueStore
  const storage = new HierarchicalStorage(kv)
  
  // Store project-level data
  const projectStore = storage.getProjectStore("org123", "proj456")
  yield* projectStore.set("config", "project-config")
  
  // Store environment-specific data
  const devStore = storage.getEnvironmentStore("org123", "proj456", "dev")
  const prodStore = storage.getEnvironmentStore("org123", "proj456", "prod")
  
  yield* devStore.set("api-key", "dev-key-123")
  yield* prodStore.set("api-key", "prod-key-456")
  
  // Keys are isolated by namespace
  const devKey = yield* devStore.get("api-key")   // "dev-key-123"
  const prodKey = yield* prodStore.get("api-key") // "prod-key-456"
})
```

### Feature 3: Storage Backends

KeyValueStore supports multiple storage backends through layers:

#### In-Memory Storage

```typescript
import { KeyValueStore, layerMemory } from "@effect/platform/KeyValueStore"
import { Effect } from "effect"

// Perfect for testing and caching
const inMemoryExample = Effect.gen(function* () {
  const kv = yield* KeyValueStore.KeyValueStore
  
  // Ultra-fast operations
  yield* kv.set("key1", "value1")
  yield* kv.set("key2", "value2")
  
  const value = yield* kv.get("key1")
  console.log("Retrieved:", value)
}).pipe(
  Effect.provide(layerMemory)
)
```

#### File System Storage

```typescript
import { KeyValueStore, layerFileSystem } from "@effect/platform/KeyValueStore"
import { FileSystem, Path } from "@effect/platform"
import { NodeContext } from "@effect/platform-node"
import { Effect } from "effect"

// Persistent file-based storage
const fileSystemExample = Effect.gen(function* () {
  const kv = yield* KeyValueStore.KeyValueStore
  
  // Data persisted to disk
  yield* kv.set("app:state", JSON.stringify({
    version: "1.0.0",
    lastRun: new Date().toISOString()
  }))
  
  // Survives application restarts
  const state = yield* kv.get("app:state")
  console.log("Restored state:", state)
}).pipe(
  Effect.provide(layerFileSystem("./data/storage")),
  Effect.provide(NodeContext.layer)
)
```

#### Browser Storage (localStorage/sessionStorage)

```typescript
import { KeyValueStore, layerStorage } from "@effect/platform/KeyValueStore"
import { Effect } from "effect"

// Browser storage layer
const createBrowserLayer = (storage: Storage) =>
  layerStorage(() => storage)

// localStorage - persists across sessions
const localStorageExample = Effect.gen(function* () {
  const kv = yield* KeyValueStore.KeyValueStore
  
  // Store user preferences
  yield* kv.set("user:theme", "dark")
  yield* kv.set("user:language", "en")
  
  // Survives page reloads
  const theme = yield* kv.get("user:theme")
  console.log("Theme preference:", theme)
}).pipe(
  Effect.provide(createBrowserLayer(localStorage))
)

// sessionStorage - cleared when tab closes
const sessionStorageExample = Effect.gen(function* () {
  const kv = yield* KeyValueStore.KeyValueStore
  
  // Store temporary session data
  yield* kv.set("form:draft", JSON.stringify({
    title: "My Draft",
    content: "Work in progress..."
  }))
  
  // Available during session only
  const draft = yield* kv.get("form:draft")
  console.log("Draft data:", draft)
}).pipe(
  Effect.provide(createBrowserLayer(sessionStorage))
)
```

#### Custom Storage Backend

```typescript
import { KeyValueStore } from "@effect/platform/KeyValueStore"
import { Effect, Option, HashMap } from "effect"
import Redis from "ioredis"

// Redis-backed KeyValueStore
const createRedisLayer = (redis: Redis) => {
  return Layer.succeed(
    KeyValueStore.KeyValueStore,
    KeyValueStore.make({
      get: (key: string) =>
        Effect.tryPromise({
          try: async () => {
            const value = await redis.get(key)
            return value ? Option.some(value) : Option.none()
          },
          catch: (error) => new Error(`Redis get error: ${error}`)
        }),
      
      getUint8Array: (key: string) =>
        Effect.tryPromise({
          try: async () => {
            const value = await redis.getBuffer(key)
            return value ? Option.some(new Uint8Array(value)) : Option.none()
          },
          catch: (error) => new Error(`Redis get error: ${error}`)
        }),
      
      set: (key: string, value: string | Uint8Array) =>
        Effect.tryPromise({
          try: () => redis.set(key, value as any),
          catch: (error) => new Error(`Redis set error: ${error}`)
        }).pipe(Effect.asVoid),
      
      remove: (key: string) =>
        Effect.tryPromise({
          try: () => redis.del(key),
          catch: (error) => new Error(`Redis delete error: ${error}`)
        }).pipe(Effect.asVoid),
      
      clear: Effect.tryPromise({
        try: () => redis.flushdb(),
        catch: (error) => new Error(`Redis clear error: ${error}`)
      }).pipe(Effect.asVoid),
      
      size: Effect.tryPromise({
        try: () => redis.dbsize(),
        catch: (error) => new Error(`Redis size error: ${error}`)
      })
    })
  )
}

// Usage with Redis backend
const redisExample = Effect.gen(function* () {
  const kv = yield* KeyValueStore.KeyValueStore
  
  // Use Redis as storage backend
  yield* kv.set("distributed:cache", "shared-value")
  
  // Available across all application instances
  const value = yield* kv.get("distributed:cache")
  console.log("Distributed value:", value)
}).pipe(
  Effect.provide(createRedisLayer(new Redis()))
)
```

## Practical Patterns & Best Practices

### Pattern 1: Atomic Operations Helper

```typescript
import { KeyValueStore } from "@effect/platform/KeyValueStore"
import { Effect, Option, Duration, pipe } from "effect"

// Atomic increment/decrement operations
export const createCounter = (key: string) => {
  const increment = (amount: number = 1) =>
    Effect.gen(function* () {
      const kv = yield* KeyValueStore.KeyValueStore
      
      return yield* kv.modify(key, (current) =>
        pipe(
          current,
          Option.getOrElse(() => "0"),
          (v) => parseInt(v, 10),
          (n) => Option.some(String(n + amount))
        )
      )
    })
  
  const decrement = (amount: number = 1) =>
    increment(-amount)
  
  const get = () =>
    Effect.gen(function* () {
      const kv = yield* KeyValueStore.KeyValueStore
      const value = yield* kv.get(key)
      
      return pipe(
        Option.map((v: string) => parseInt(v, 10)),
        Option.getOrElse(() => 0)
      )(value)
    })
  
  const reset = () =>
    Effect.gen(function* () {
      const kv = yield* KeyValueStore.KeyValueStore
      yield* kv.set(key, "0")
    })
  
  return { increment, decrement, get, reset }
}

// Usage: Rate limiting
const createRateLimiter = (
  key: string,
  limit: number,
  window: Duration.Duration
) => {
  const counter = createCounter(key)
  
  const checkLimit = () =>
    Effect.gen(function* () {
      const current = yield* counter.get()
      
      if (current >= limit) {
        return false
      }
      
      yield* counter.increment()
      
      // Set expiry on first increment
      if (current === 0) {
        // Note: Real implementation would need TTL support
        yield* Effect.sleep(window).pipe(
          Effect.zipRight(counter.reset()),
          Effect.fork
        )
      }
      
      return true
    })
  
  return { checkLimit, getCount: counter.get }
}
```

### Pattern 2: Transactional Updates

```typescript
import { KeyValueStore } from "@effect/platform/KeyValueStore"
import { Effect, Option, Exit, Cause } from "effect"

// Optimistic concurrency control
export class OptimisticLockStore<A> {
  constructor(
    private readonly schema: Schema.Schema<A, any>,
    private readonly getVersion: (value: A) => number,
    private readonly setVersion: (value: A, version: number) => A
  ) {}

  update(key: string, updater: (value: A) => A) {
    return Effect.gen(function* () {
      const kv = yield* KeyValueStore.KeyValueStore
      const store = kv.forSchema(this.schema)
      
      // Retry loop for optimistic locking
      let retries = 0
      const maxRetries = 3
      
      while (retries < maxRetries) {
        // Get current value and version
        const current = yield* store.get(key)
        
        if (Option.isNone(current)) {
          return yield* Effect.fail(new Error("Key not found"))
        }
        
        const currentValue = current.value
        const currentVersion = this.getVersion(currentValue)
        
        // Apply update
        const updated = updater(currentValue)
        const newValue = this.setVersion(updated, currentVersion + 1)
        
        // Try to save with version check
        const success = yield* store.modify(key, (stored) =>
          Option.flatMap(stored, (s) => {
            if (this.getVersion(s) === currentVersion) {
              return Option.some(newValue)
            }
            return Option.none()
          })
        )
        
        if (Option.isSome(success)) {
          return newValue
        }
        
        // Version conflict - retry
        retries++
        yield* Effect.sleep(Duration.millis(50 * retries))
      }
      
      return yield* Effect.fail(new Error("Update failed after retries"))
    })
  }
}

// Usage: Bank account with balance
const BankAccount = Schema.Struct({
  id: Schema.String,
  balance: Schema.Number,
  version: Schema.Number
})

const accountStore = new OptimisticLockStore(
  BankAccount,
  (account) => account.version,
  (account, version) => ({ ...account, version })
)

const transfer = (fromId: string, toId: string, amount: number) =>
  Effect.gen(function* () {
    // Debit from account
    yield* accountStore.update(fromId, (account) => ({
      ...account,
      balance: account.balance - amount
    }))
    
    // Credit to account
    yield* accountStore.update(toId, (account) => ({
      ...account,
      balance: account.balance + amount
    }))
  })
```

### Pattern 3: Batch Operations

```typescript
import { KeyValueStore } from "@effect/platform/KeyValueStore"
import { Effect, Chunk, pipe } from "effect"

// Batch operations for performance
export class BatchStore {
  constructor(
    private readonly batchSize: number = 100,
    private readonly concurrency: number = 5
  ) {}

  setMany(entries: Array<[string, string]>) {
    return Effect.gen(function* () {
      const kv = yield* KeyValueStore.KeyValueStore
      
      // Process in batches
      const batches = Chunk.fromIterable(entries).pipe(
        Chunk.chunksOf(this.batchSize)
      )
      
      yield* Effect.forEach(
        batches,
        (batch) => Effect.forEach(
          batch,
          ([key, value]) => kv.set(key, value),
          { concurrency: "unbounded" }
        ),
        { concurrency: this.concurrency }
      )
    })
  }

  getMany(keys: string[]) {
    return Effect.gen(function* () {
      const kv = yield* KeyValueStore.KeyValueStore
      
      return yield* Effect.forEach(
        keys,
        (key) => kv.get(key).pipe(
          Effect.map((value) => [key, value] as const)
        ),
        { concurrency: this.concurrency }
      )
    })
  }

  removeMany(keys: string[]) {
    return Effect.gen(function* () {
      const kv = yield* KeyValueStore.KeyValueStore
      
      yield* Effect.forEach(
        keys,
        (key) => kv.remove(key),
        { concurrency: this.concurrency }
      )
    })
  }
}

// Usage: Bulk data import
const importUsers = (users: Array<{ id: string; data: any }>) => {
  const batch = new BatchStore()
  
  return Effect.gen(function* () {
    // Prepare entries
    const entries = users.map(({ id, data }) => [
      `user:${id}`,
      JSON.stringify(data)
    ] as [string, string])
    
    // Batch insert
    console.log(`Importing ${users.length} users...`)
    yield* batch.setMany(entries)
    
    console.log("Import complete!")
  })
}
```

## Integration Examples

### Integration with HTTP Server

```typescript
import { HttpServer, HttpServerRequest, HttpRouter } from "@effect/platform"
import { KeyValueStore } from "@effect/platform/KeyValueStore"
import { Effect, Schema, Option } from "effect"

// API key management
const ApiKey = Schema.Struct({
  key: Schema.String,
  name: Schema.String,
  permissions: Schema.Array(Schema.String),
  createdAt: Schema.DateTimeUtc,
  lastUsed: Schema.Option(Schema.DateTimeUtc),
  requestCount: Schema.Number
})

// Authentication middleware using KeyValueStore
const authMiddleware = HttpServer.middleware.make((app) =>
  Effect.gen(function* () {
    const request = yield* HttpServerRequest.ServerRequest
    const apiKey = request.headers["x-api-key"]
    
    if (!apiKey) {
      return yield* HttpServer.response.unauthorized()
    }
    
    const kv = yield* KeyValueStore.KeyValueStore
    const store = kv.forSchema(ApiKey)
    
    const keyData = yield* store.get(`apikey:${apiKey}`)
    
    if (Option.isNone(keyData)) {
      return yield* HttpServer.response.unauthorized()
    }
    
    // Update last used and request count
    yield* store.modify(`apikey:${apiKey}`, (current) =>
      Option.map(current, (key) => ({
        ...key,
        lastUsed: Option.some(new Date()),
        requestCount: key.requestCount + 1
      }))
    )
    
    // Add key data to request context
    return yield* app.pipe(
      Effect.provideService(ApiKeyContext, keyData.value)
    )
  })
)

// Routes using stored data
const apiRouter = HttpRouter.empty.pipe(
  HttpRouter.get("/user/:id", 
    Effect.gen(function* () {
      const { id } = yield* HttpRouter.params
      const kv = yield* KeyValueStore.KeyValueStore
      
      const userData = yield* kv.get(`user:${id}`)
      
      return Option.match(userData, {
        onNone: () => HttpServer.response.notFound(),
        onSome: (data) => HttpServer.response.json(JSON.parse(data))
      })
    })
  ),
  
  HttpRouter.post("/cache/invalidate",
    Effect.gen(function* () {
      const kv = yield* KeyValueStore.KeyValueStore
      const cacheStore = prefix(kv, "cache")
      
      yield* cacheStore.clear
      
      return HttpServer.response.json({ 
        message: "Cache invalidated successfully" 
      })
    })
  )
)

// Server setup
const server = apiRouter.pipe(
  authMiddleware,
  HttpServer.serve(HttpServer.listen({ port: 3000 })),
  Effect.provide(layerFileSystem("./data/api"))
)
```

### Testing Strategies

```typescript
import { KeyValueStore, layerMemory } from "@effect/platform/KeyValueStore"
import { Effect, TestContext, TestClock, Option } from "effect"
import { describe, it, expect } from "@effect/vitest"

// Test utilities for KeyValueStore
export const TestKeyValueStore = {
  // Create a test layer with initial data
  layer: (initialData: Record<string, string> = {}) =>
    Layer.effect(
      KeyValueStore.KeyValueStore,
      Effect.gen(function* () {
        const base = yield* Layer.build(layerMemory)
        const kv = Context.get(base, KeyValueStore.KeyValueStore)
        
        // Populate initial data
        yield* Effect.forEach(
          Object.entries(initialData),
          ([key, value]) => kv.set(key, value),
          { discard: true }
        )
        
        return kv
      })
    ),
  
  // Assertion helpers
  assertStored: (key: string, expected: string) =>
    Effect.gen(function* () {
      const kv = yield* KeyValueStore.KeyValueStore
      const actual = yield* kv.get(key)
      
      expect(actual).toEqual(Option.some(expected))
    }),
  
  assertNotStored: (key: string) =>
    Effect.gen(function* () {
      const kv = yield* KeyValueStore.KeyValueStore
      const actual = yield* kv.get(key)
      
      expect(actual).toEqual(Option.none())
    }),
  
  assertSize: (expected: number) =>
    Effect.gen(function* () {
      const kv = yield* KeyValueStore.KeyValueStore
      const size = yield* kv.size
      
      expect(size).toBe(expected)
    })
}

// Testing cache with TTL
describe("CacheManager", () => {
  it("should expire entries after TTL", () =>
    Effect.gen(function* () {
      const cache = new CacheManager("test")
      
      // Set cache with 1 hour TTL
      yield* cache.set("key1", "value1", Duration.hours(1))
      
      // Should exist initially
      const fresh = yield* cache.get("key1")
      expect(Option.isSome(fresh)).toBe(true)
      
      // Advance time by 2 hours
      yield* TestClock.adjust(Duration.hours(2))
      
      // Should be expired
      const expired = yield* cache.get("key1")
      expect(Option.isNone(expired)).toBe(true)
      
      // Should have been removed
      yield* TestKeyValueStore.assertNotStored("test:key1")
    }).pipe(
      Effect.provide(TestKeyValueStore.layer()),
      Effect.provide(TestContext.TestContext)
    ))
  
  it("should handle concurrent operations", () =>
    Effect.gen(function* () {
      const cache = new CacheManager("test")
      
      // Concurrent sets
      yield* Effect.all([
        cache.set("k1", "v1", Duration.minutes(5)),
        cache.set("k2", "v2", Duration.minutes(5)),
        cache.set("k3", "v3", Duration.minutes(5))
      ], { concurrency: "unbounded" })
      
      // Verify all stored
      yield* TestKeyValueStore.assertSize(3)
      
      // Concurrent gets
      const values = yield* Effect.all([
        cache.get("k1"),
        cache.get("k2"),
        cache.get("k3")
      ], { concurrency: "unbounded" })
      
      expect(values.map(Option.isSome)).toEqual([true, true, true])
    }).pipe(
      Effect.provide(TestKeyValueStore.layer())
    ))
})

// Testing schema validation
describe("SchemaStore", () => {
  const User = Schema.Struct({
    id: Schema.String,
    email: Schema.Email,
    age: Schema.Number.pipe(Schema.positive())
  })
  
  it("should validate on set", () =>
    Effect.gen(function* () {
      const kv = yield* KeyValueStore.KeyValueStore
      const store = kv.forSchema(User)
      
      // Valid data succeeds
      yield* store.set("user:1", {
        id: "1",
        email: "test@example.com",
        age: 25
      })
      
      // Invalid data fails
      const invalid = store.set("user:2", {
        id: "2",
        email: "not-an-email",
        age: -5
      })
      
      yield* Effect.flip(invalid)
    }).pipe(
      Effect.provide(layerMemory)
    ))
  
  it("should validate on get", () =>
    Effect.gen(function* () {
      const kv = yield* KeyValueStore.KeyValueStore
      
      // Store invalid data directly
      yield* kv.set("user:bad", JSON.stringify({
        id: "bad",
        email: "invalid"
        // missing age
      }))
      
      // SchemaStore should fail to parse
      const store = kv.forSchema(User)
      const result = yield* Effect.flip(store.get("user:bad"))
      
      expect(result).toBeDefined()
    }).pipe(
      Effect.provide(layerMemory)
    ))
})

// Testing prefixed stores
describe("Prefixed Stores", () => {
  it("should isolate namespaces", () =>
    Effect.gen(function* () {
      const kv = yield* KeyValueStore.KeyValueStore
      
      const store1 = prefix(kv, "app1")
      const store2 = prefix(kv, "app2")
      
      // Set same key in different namespaces
      yield* store1.set("config", "app1-config")
      yield* store2.set("config", "app2-config")
      
      // Values should be different
      const v1 = yield* store1.get("config")
      const v2 = yield* store2.get("config")
      
      expect(v1).toEqual(Option.some("app1-config"))
      expect(v2).toEqual(Option.some("app2-config"))
      
      // Clear should only affect one namespace
      yield* store1.clear
      
      const after1 = yield* store1.get("config")
      const after2 = yield* store2.get("config")
      
      expect(after1).toEqual(Option.none())
      expect(after2).toEqual(Option.some("app2-config"))
    }).pipe(
      Effect.provide(layerMemory)
    ))
})
```

## Conclusion

KeyValueStore provides a robust, type-safe, and effectful solution for managing key-value storage across different platforms and backends. It solves the common problems of inconsistent storage implementations, lack of type safety, and poor error handling in traditional approaches.

Key benefits:
- **Unified Interface**: Same API works across in-memory, file system, browser storage, and custom backends
- **Type Safety**: SchemaStore ensures data integrity with automatic validation and serialization
- **Effect Integration**: Full support for error handling, tracing, dependency injection, and testing
- **Composability**: Prefix support enables namespace isolation and multi-tenant architectures
- **Production Ready**: Built-in implementations for common storage needs with proper error handling

KeyValueStore is ideal for caching, session management, configuration storage, feature flags, and any scenario requiring reliable key-value storage with type safety and consistent error handling.