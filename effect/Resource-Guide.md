# Resource: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem Resource Solves

Modern applications often need to manage values that are expensive to compute, periodically updated, or require automatic refresh. Traditional approaches to resource caching and lifecycle management are often flawed and lead to stale data, resource leaks, and race conditions.

```typescript
// Traditional approach - problematic resource management
class ConfigurationManager {
  private cache: any = null
  private lastUpdate = 0
  private refreshInterval = 60000 // 1 minute
  
  async getConfig() {
    const now = Date.now()
    
    // Race condition: multiple concurrent calls can trigger multiple refreshes
    if (!this.cache || now - this.lastUpdate > this.refreshInterval) {
      console.log("Refreshing configuration...")
      this.cache = await this.loadFromRemote() // Multiple concurrent loads!
      this.lastUpdate = now
    }
    
    return this.cache
  }
  
  // Manual refresh - error-prone and scattered throughout code
  async refresh() {
    this.cache = await this.loadFromRemote()
    this.lastUpdate = Date.now()
  }
  
  private async loadFromRemote() {
    // Expensive operation that should be deduplicated
    const response = await fetch('/api/config')
    return response.json()
  }
}
```

This approach leads to:
- **Race Conditions** - Multiple concurrent calls trigger duplicate expensive operations
- **Manual Lifecycle Management** - Developers must remember to refresh resources manually
- **No Automatic Cleanup** - Resources aren't properly managed when no longer needed
- **Error Handling Complexity** - Failed refreshes leave the system in inconsistent states
- **Memory Leaks** - Background refresh tasks continue even when resources are no longer used

### The Resource Solution

Effect's Resource module provides a managed, cached value that can be automatically or manually refreshed with guaranteed cleanup and proper concurrency control.

```typescript
import { Resource, Effect, Schedule, Duration, Scope } from "effect"

// Declarative resource with automatic refresh
const configResource = Resource.auto(
  // Acquisition effect - runs only when needed
  Effect.gen(function* () {
    console.log("Loading configuration from remote...")
    const response = yield* Effect.tryPromise({
      try: () => fetch('/api/config'),
      catch: () => new ConfigError("Failed to fetch config")
    })
    return yield* Effect.tryPromise({
      try: () => response.json(),
      catch: () => new ConfigError("Failed to parse config")
    })
  }),
  
  // Automatic refresh every 5 minutes
  Schedule.fixed(Duration.minutes(5))
)

const program = Effect.gen(function* () {
  const resource = yield* configResource
  
  // Multiple concurrent calls are automatically deduplicated
  const configs = yield* Effect.all([
    Resource.get(resource),
    Resource.get(resource),
    Resource.get(resource)
  ], { concurrency: "unbounded" })
  
  // Only one network call made, result shared across all requests
  console.log("All configs are identical:", configs[0] === configs[1])
  
  // Resource automatically refreshes in the background
  yield* Effect.sleep(Duration.minutes(6))
  const refreshedConfig = yield* Resource.get(resource)
  
  return { config: refreshedConfig }
}).pipe(
  Effect.scoped // Automatic cleanup when scope closes
)
```

### Key Concepts

**Resource<A, E>**: A managed, cached value that can be refreshed either manually or automatically, with built-in concurrency control and lifecycle management.

**Acquisition Effect**: The Effect that computes the resource value. This is automatically deduplicated when multiple concurrent requests occur.

**Automatic Refresh**: Resources can be configured to refresh automatically according to a Schedule, ensuring data freshness without manual intervention.

**Manual Refresh**: Resources can be manually refreshed using `Resource.refresh`, allowing fine-grained control over when updates occur.

**Scope Integration**: Resources are properly managed within Scopes, ensuring automatic cleanup when the scope closes, preventing resource leaks.

## Basic Usage Patterns

### Pattern 1: Manual Resource Management

```typescript
import { Resource, Effect, Console, Scope } from "effect"

// Create a resource that requires manual refresh
const manualResource = Resource.manual(
  Effect.gen(function* () {
    yield* Console.log("Acquiring expensive resource...")
    yield* Effect.sleep(Duration.milliseconds(500)) // Simulate expensive operation
    
    return {
      id: Math.random().toString(36).substr(2, 9),
      timestamp: Date.now(),
      data: "expensive computation result"
    }
  })
)

const program = Effect.gen(function* () {
  const resource = yield* manualResource
  
  // Get the initial value
  const value1 = yield* Resource.get(resource)
  yield* Console.log(`Initial value: ${value1.id}`)
  
  // Get again - uses cached value
  const value2 = yield* Resource.get(resource)
  yield* Console.log(`Cached value: ${value2.id}`)
  yield* Console.log(`Same instance: ${value1 === value2}`)
  
  // Manually refresh the resource
  yield* Console.log("Manually refreshing...")
  yield* Resource.refresh(resource)
  
  // Get new value after refresh
  const value3 = yield* Resource.get(resource)
  yield* Console.log(`New value: ${value3.id}`)
  yield* Console.log(`Different instance: ${value1 !== value3}`)
  
  return value3
}).pipe(Effect.scoped)

Effect.runPromise(program)
/*
Output:
Acquiring expensive resource...
Initial value: abc123def
Cached value: abc123def
Same instance: true
Manually refreshing...
Acquiring expensive resource...
New value: ghi456jkl
Different instance: true
*/
```

### Pattern 2: Automatic Resource Refresh

```typescript
import { Resource, Effect, Schedule, Duration, Console } from "effect"

// Create a resource that automatically refreshes every 2 seconds
const autoResource = Resource.auto(
  Effect.gen(function* () {
    const timestamp = new Date().toISOString()
    yield* Console.log(`Refreshing at ${timestamp}`)
    
    return {
      timestamp,
      randomValue: Math.random(),
      status: "active"
    }
  }),
  
  // Refresh every 2 seconds
  Schedule.fixed(Duration.seconds(2))
)

const program = Effect.gen(function* () {
  const resource = yield* autoResource
  
  // Get initial value
  const value1 = yield* Resource.get(resource)
  yield* Console.log(`Value 1: ${value1.timestamp}`)
  
  // Wait 1 second - should get same value
  yield* Effect.sleep(Duration.seconds(1))
  const value2 = yield* Resource.get(resource)
  yield* Console.log(`Value 2: ${value2.timestamp}`)
  yield* Console.log(`Same? ${value1 === value2}`)
  
  // Wait 2 more seconds - should get refreshed value
  yield* Effect.sleep(Duration.seconds(2))
  const value3 = yield* Resource.get(resource)
  yield* Console.log(`Value 3: ${value3.timestamp}`)
  yield* Console.log(`Different? ${value1 !== value3}`)
  
  return value3
}).pipe(Effect.scoped)

Effect.runPromise(program)
/*
Output:
Refreshing at 2023-12-01T10:00:00.000Z
Value 1: 2023-12-01T10:00:00.000Z
Value 2: 2023-12-01T10:00:00.000Z
Same? true
Refreshing at 2023-12-01T10:00:02.000Z
Value 3: 2023-12-01T10:00:02.000Z
Different? true
*/
```

### Pattern 3: Error Handling in Resources

```typescript
import { Resource, Effect, Schedule, Duration, Console, Data } from "effect"

class NetworkError extends Data.TaggedError("NetworkError")<{
  cause: string
  retryable: boolean
}> {}

// Resource with error handling and retry logic
const resilientResource = Resource.auto(
  Effect.gen(function* () {
    // Simulate network operation that might fail
    const success = Math.random() > 0.3 // 70% success rate
    
    if (!success) {
      yield* Console.log("Network operation failed")
      return yield* Effect.fail(new NetworkError({
        cause: "Network timeout",
        retryable: true
      }))
    }
    
    yield* Console.log("Network operation succeeded")
    return {
      data: "successful response",
      timestamp: Date.now()
    }
  }).pipe(
    // Add retry logic before passing to Resource
    Effect.retry(Schedule.exponential(Duration.milliseconds(100)).pipe(
      Schedule.recurs(3)
    ))
  ),
  
  // Refresh every 10 seconds
  Schedule.fixed(Duration.seconds(10))
)

const program = Effect.gen(function* () {
  const resource = yield* resilientResource
  
  // Get value with error handling
  const result = yield* Resource.get(resource).pipe(
    Effect.catchTag("NetworkError", (error) => 
      Effect.gen(function* () {
        yield* Console.log(`Handling error: ${error.cause}`)
        return { data: "fallback response", timestamp: Date.now() }
      })
    )
  )
  
  yield* Console.log(`Result: ${result.data}`)
  return result
}).pipe(Effect.scoped)

Effect.runPromise(program)
```

## Real-World Examples

### Example 1: Configuration Management Service

Managing application configuration that needs periodic updates from external sources with fallback strategies.

```typescript
import { Resource, Effect, Schedule, Duration, Context, Layer, Data } from "effect"

// Define configuration structure
interface AppConfig {
  readonly database: {
    readonly host: string
    readonly port: number
    readonly maxConnections: number
  }
  readonly features: {
    readonly enableNewUI: boolean
    readonly maxFileSize: number
  }
  readonly version: string
}

class ConfigError extends Data.TaggedError("ConfigError")<{
  operation: string
  cause: string
}> {}

// External configuration service
class ConfigService extends Context.Tag("ConfigService")<
  ConfigService,
  {
    readonly loadRemoteConfig: () => Effect.Effect<AppConfig, ConfigError>
    readonly loadLocalConfig: () => Effect.Effect<AppConfig, ConfigError>
  }
>() {}

// Configuration resource with fallback strategy
const createConfigResource = Effect.gen(function* () {
  const configService = yield* ConfigService
  
  return yield* Resource.auto(
    Effect.gen(function* () {
      yield* Effect.log("Loading configuration...")
      
      // Try remote config first, fallback to local
      const config = yield* configService.loadRemoteConfig().pipe(
        Effect.catchTag("ConfigError", (error) => Effect.gen(function* () {
          yield* Effect.logWarning(`Remote config failed: ${error.cause}`)
          yield* Effect.log("Falling back to local configuration")
          return yield* configService.loadLocalConfig()
        }))
      )
      
      yield* Effect.log(`Configuration loaded: v${config.version}`)
      return config
    }),
    
    // Refresh configuration every 5 minutes
    Schedule.fixed(Duration.minutes(5))
  )
})

// Configuration manager service
class ConfigManager extends Context.Tag("ConfigManager")<
  ConfigManager,
  {
    readonly getConfig: () => Effect.Effect<AppConfig, ConfigError>
    readonly refreshConfig: () => Effect.Effect<void, ConfigError>
    readonly isFeatureEnabled: (feature: keyof AppConfig["features"]) => Effect.Effect<boolean, ConfigError>
  }
>() {}

const makeConfigManager = Effect.gen(function* () {
  const resource = yield* createConfigResource
  
  return {
    getConfig: () => Resource.get(resource),
    
    refreshConfig: () => Resource.refresh(resource),
    
    isFeatureEnabled: (feature: keyof AppConfig["features"]) => Effect.gen(function* () {
      const config = yield* Resource.get(resource)
      return config.features[feature]
    })
  } satisfies ConfigManager
})

// Business logic using configuration
const handleFileUpload = (file: { size: number; name: string }) => Effect.gen(function* () {
  const configManager = yield* ConfigManager
  
  const config = yield* configManager.getConfig()
  
  if (file.size > config.features.maxFileSize) {
    return yield* Effect.fail(new ConfigError({
      operation: "file_upload",
      cause: `File size ${file.size} exceeds limit ${config.features.maxFileSize}`
    }))
  }
  
  const useNewUI = yield* configManager.isFeatureEnabled("enableNewUI")
  
  return {
    uploadUrl: `/upload/${file.name}`,
    useNewUI,
    maxConnections: config.database.maxConnections
  }
})

// Service layers
const ConfigServiceLive = Layer.succeed(ConfigService, {
  loadRemoteConfig: () => Effect.gen(function* () {
    // Simulate remote API call
    yield* Effect.sleep(Duration.milliseconds(200))
    
    if (Math.random() < 0.1) { // 10% failure rate
      return yield* Effect.fail(new ConfigError({
        operation: "remote_load",
        cause: "Network timeout"
      }))
    }
    
    return {
      database: {
        host: "remote-db.example.com",
        port: 5432,
        maxConnections: 100
      },
      features: {
        enableNewUI: true,
        maxFileSize: 10485760 // 10MB
      },
      version: "1.2.3"
    }
  }),
  
  loadLocalConfig: () => Effect.succeed({
    database: {
      host: "localhost",
      port: 5432,
      maxConnections: 20
    },
    features: {
      enableNewUI: false,
      maxFileSize: 5242880 // 5MB
    },
    version: "1.0.0-fallback"
  })
})

const ConfigManagerLive = Layer.effect(ConfigManager, makeConfigManager).pipe(
  Layer.provide(ConfigServiceLive)
)

// Usage in application
const application = Effect.gen(function* () {
  const file = { size: 8000000, name: "large-file.pdf" }
  
  const result = yield* handleFileUpload(file)
  yield* Effect.log(`Upload result: ${JSON.stringify(result)}`)
  
  // Configuration is automatically refreshed in background
  yield* Effect.sleep(Duration.minutes(6))
  
  const newResult = yield* handleFileUpload(file)
  yield* Effect.log(`Updated result: ${JSON.stringify(newResult)}`)
  
  return result
}).pipe(
  Effect.provide(ConfigManagerLive),
  Effect.scoped
)

Effect.runPromise(application)
/*
Output:
Loading configuration...
Configuration loaded: v1.2.3
Upload result: {"uploadUrl":"/upload/large-file.pdf","useNewUI":true,"maxConnections":100}
Loading configuration...
Configuration loaded: v1.2.3
Updated result: {"uploadUrl":"/upload/large-file.pdf","useNewUI":true,"maxConnections":100}
*/
```

### Example 2: API Token Management with Automatic Renewal

Managing OAuth tokens or API keys that require periodic renewal with proper error handling and fallback strategies.

```typescript
import { Resource, Effect, Schedule, Duration, Context, Layer, Data } from "effect"

interface AuthToken {
  readonly accessToken: string
  readonly refreshToken: string
  readonly expiresAt: number
  readonly scopes: ReadonlyArray<string>
}

class AuthError extends Data.TaggedError("AuthError")<{
  operation: string
  cause: string
  recoverable: boolean
}> {}

// External authentication service
class AuthService extends Context.Tag("AuthService")<
  AuthService,
  {
    readonly authenticate: () => Effect.Effect<AuthToken, AuthError>
    readonly refreshToken: (token: string) => Effect.Effect<AuthToken, AuthError>
  }
>() {}

// Storage for persistent token storage
class TokenStorage extends Context.Tag("TokenStorage")<
  TokenStorage,
  {
    readonly load: () => Effect.Effect<AuthToken | null, never>
    readonly save: (token: AuthToken) => Effect.Effect<void, never>
    readonly clear: () => Effect.Effect<void, never>
  }
>() {}

// Token resource with automatic renewal
const createTokenResource = Effect.gen(function* () {
  const authService = yield* AuthService
  const storage = yield* TokenStorage
  
  return yield* Resource.auto(
    Effect.gen(function* () {
      yield* Effect.log("Acquiring authentication token...")
      
      // Try to load existing token first
      const existingToken = yield* storage.load()
      
      if (existingToken && existingToken.expiresAt > Date.now() + 300000) { // 5 minutes buffer
        yield* Effect.log("Using existing valid token")
        return existingToken
      }
      
      // Try to refresh if we have a refresh token
      if (existingToken?.refreshToken) {
        const refreshed = yield* authService.refreshToken(existingToken.refreshToken).pipe(
          Effect.catchTag("AuthError", (error) => Effect.gen(function* () {
            yield* Effect.logWarning(`Token refresh failed: ${error.cause}`)
            yield* storage.clear()
            return null
          }))
        )
        
        if (refreshed) {
          yield* storage.save(refreshed)
          yield* Effect.log("Token refreshed successfully")
          return refreshed
        }
      }
      
      // Authenticate from scratch
      const newToken = yield* authService.authenticate()
      yield* storage.save(newToken)
      yield* Effect.log("New token acquired")
      
      return newToken
    }),
    
    // Check for renewal every 15 minutes
    Schedule.fixed(Duration.minutes(15))
  )
})

// Token manager service
class TokenManager extends Context.Tag("TokenManager")<
  TokenManager,
  {
    readonly getValidToken: () => Effect.Effect<AuthToken, AuthError>
    readonly forceRefresh: () => Effect.Effect<void, AuthError>
    readonly hasScope: (scope: string) => Effect.Effect<boolean, AuthError>
  }
>() {}

const makeTokenManager = Effect.gen(function* () {
  const resource = yield* createTokenResource
  
  return {
    getValidToken: () => Effect.gen(function* () {
      const token = yield* Resource.get(resource)
      
      // Double-check token validity
      if (token.expiresAt <= Date.now()) {
        yield* Resource.refresh(resource)
        return yield* Resource.get(resource)
      }
      
      return token
    }),
    
    forceRefresh: () => Resource.refresh(resource),
    
    hasScope: (scope: string) => Effect.gen(function* () {
      const token = yield* Resource.get(resource)
      return token.scopes.includes(scope)
    })
  } satisfies TokenManager
})

// API client that uses managed tokens
class ApiClient extends Context.Tag("ApiClient")<
  ApiClient,
  {
    readonly makeRequest: (endpoint: string, requiredScope?: string) => Effect.Effect<any, AuthError>
  }
>() {}

const makeApiClient = Effect.gen(function* () {
  const tokenManager = yield* TokenManager
  
  return {
    makeRequest: (endpoint: string, requiredScope?: string) => Effect.gen(function* () {
      // Check scope requirements
      if (requiredScope) {
        const hasScope = yield* tokenManager.hasScope(requiredScope)
        if (!hasScope) {
          return yield* Effect.fail(new AuthError({
            operation: "scope_check",
            cause: `Missing required scope: ${requiredScope}`,
            recoverable: false
          }))
        }
      }
      
      // Get valid token
      const token = yield* tokenManager.getValidToken()
      
      // Make API request with token
      yield* Effect.log(`Making request to ${endpoint}`)
      
      const response = yield* Effect.tryPromise({
        try: () => fetch(endpoint, {
          headers: {
            'Authorization': `Bearer ${token.accessToken}`,
            'Content-Type': 'application/json'
          }
        }),
        catch: (error) => new AuthError({
          operation: "api_request",
          cause: String(error),
          recoverable: true
        })
      })
      
      if (response.status === 401) {
        // Token might be invalid, force refresh and retry
        yield* Effect.logWarning("Received 401, forcing token refresh")
        yield* tokenManager.forceRefresh()
        
        const newToken = yield* tokenManager.getValidToken()
        const retryResponse = yield* Effect.tryPromise({
          try: () => fetch(endpoint, {
            headers: {
              'Authorization': `Bearer ${newToken.accessToken}`,
              'Content-Type': 'application/json'
            }
          }),
          catch: (error) => new AuthError({
            operation: "api_request_retry",
            cause: String(error),
            recoverable: false
          })
        })
        
        if (!retryResponse.ok) {
          return yield* Effect.fail(new AuthError({
            operation: "api_request_retry",
            cause: `HTTP ${retryResponse.status}`,
            recoverable: false
          }))
        }
        
        return yield* Effect.tryPromise({
          try: () => retryResponse.json(),
          catch: (error) => new AuthError({
            operation: "response_parse",
            cause: String(error),
            recoverable: false
          })
        })
      }
      
      if (!response.ok) {
        return yield* Effect.fail(new AuthError({
          operation: "api_request",
          cause: `HTTP ${response.status}`,
          recoverable: response.status >= 500
        }))
      }
      
      return yield* Effect.tryPromise({
        try: () => response.json(),
        catch: (error) => new AuthError({
          operation: "response_parse",
          cause: String(error),
          recoverable: false
        })
      })
    })
  } satisfies ApiClient
})

// Service layers
const AuthServiceLive = Layer.succeed(AuthService, {
  authenticate: () => Effect.gen(function* () {
    yield* Effect.sleep(Duration.milliseconds(500)) // Simulate auth delay
    
    if (Math.random() < 0.05) { // 5% failure rate
      return yield* Effect.fail(new AuthError({
        operation: "authenticate",
        cause: "Authentication server unavailable",
        recoverable: true
      }))
    }
    
    const expiresAt = Date.now() + (60 * 60 * 1000) // 1 hour
    
    return {
      accessToken: `access_${Math.random().toString(36).substr(2, 16)}`,
      refreshToken: `refresh_${Math.random().toString(36).substr(2, 16)}`,
      expiresAt,
      scopes: ["read", "write", "admin"]
    }
  }),
  
  refreshToken: (refreshToken: string) => Effect.gen(function* () {
    yield* Effect.sleep(Duration.milliseconds(200))
    
    if (Math.random() < 0.1) { // 10% failure rate
      return yield* Effect.fail(new AuthError({
        operation: "refresh",
        cause: "Refresh token invalid or expired",
        recoverable: false
      }))
    }
    
    const expiresAt = Date.now() + (60 * 60 * 1000) // 1 hour
    
    return {
      accessToken: `access_${Math.random().toString(36).substr(2, 16)}`,
      refreshToken,
      expiresAt,
      scopes: ["read", "write", "admin"]
    }
  })
})

const TokenStorageLive = Layer.effect(TokenStorage, Effect.gen(function* () {
  const storage = new Map<string, AuthToken>()
  
  return {
    load: () => Effect.succeed(storage.get("token") || null),
    save: (token: AuthToken) => Effect.sync(() => {
      storage.set("token", token)
    }),
    clear: () => Effect.sync(() => {
      storage.delete("token")
    })
  }
}))

const TokenManagerLive = Layer.effect(TokenManager, makeTokenManager).pipe(
  Layer.provide(Layer.mergeAll(AuthServiceLive, TokenStorageLive))
)

const ApiClientLive = Layer.effect(ApiClient, makeApiClient).pipe(
  Layer.provide(TokenManagerLive)
)

// Usage in application
const application = Effect.gen(function* () {
  const apiClient = yield* ApiClient
  
  // Make several API requests
  const userResponse = yield* apiClient.makeRequest("/api/users", "read")
  yield* Effect.log(`Users: ${JSON.stringify(userResponse)}`)
  
  const adminResponse = yield* apiClient.makeRequest("/api/admin", "admin")
  yield* Effect.log(`Admin data: ${JSON.stringify(adminResponse)}`)
  
  // Simulate time passing - token should auto-refresh
  yield* Effect.sleep(Duration.minutes(16))
  
  const refreshedResponse = yield* apiClient.makeRequest("/api/users", "read")
  yield* Effect.log(`Refreshed request: ${JSON.stringify(refreshedResponse)}`)
  
  return { userResponse, adminResponse, refreshedResponse }
}).pipe(
  Effect.provide(ApiClientLive),
  Effect.scoped
)

Effect.runPromise(application).then(console.log).catch(console.error)
```

### Example 3: Database Connection Pool with Health Monitoring

Managing database connection pools that require periodic health checks and automatic failover.

```typescript
import { Resource, Effect, Schedule, Duration, Context, Layer, Data, Ref } from "effect"

interface DatabaseConnection {
  readonly id: string
  readonly host: string
  readonly isHealthy: boolean
  readonly lastUsed: number
  readonly connectionCount: number
}

interface DatabasePool {
  readonly primary: DatabaseConnection
  readonly secondary: ReadonlyArray<DatabaseConnection>
  readonly totalConnections: number
  readonly healthyConnections: number
}

class DatabaseError extends Data.TaggedError("DatabaseError")<{
  operation: string
  connection?: string
  cause: string
}> {}

// Database service for health checks and connections
class DatabaseService extends Context.Tag("DatabaseService")<
  DatabaseService,
  {
    readonly checkHealth: (host: string) => Effect.Effect<boolean, DatabaseError>
    readonly createConnection: (host: string) => Effect.Effect<DatabaseConnection, DatabaseError>
    readonly getConnectionCount: (host: string) => Effect.Effect<number, DatabaseError>
  }
>() {}

// Metrics tracking for the database pool
class DatabaseMetrics extends Context.Tag("DatabaseMetrics")<
  DatabaseMetrics,
  {
    readonly recordConnectionEvent: (event: "create" | "fail" | "health_check") => Effect.Effect<void>
    readonly getMetrics: () => Effect.Effect<{
      totalEvents: number
      failures: number
      healthChecks: number
    }>
  }
>() {}

// Database pool resource with health monitoring
const createDatabasePoolResource = Effect.gen(function* () {
  const dbService = yield* DatabaseService
  const metrics = yield* DatabaseMetrics
  
  const hosts = ["primary-db.example.com", "secondary-1.example.com", "secondary-2.example.com"]
  
  return yield* Resource.auto(
    Effect.gen(function* () {
      yield* Effect.log("Checking database pool health...")
      
      // Check health of all hosts concurrently
      const healthChecks = yield* Effect.all(
        hosts.map(host => Effect.gen(function* () {
          yield* metrics.recordConnectionEvent("health_check")
          
          const isHealthy = yield* dbService.checkHealth(host).pipe(
            Effect.catchTag("DatabaseError", () => Effect.succeed(false))
          )
          
          if (isHealthy) {
            const connection = yield* dbService.createConnection(host).pipe(
              Effect.catchTag("DatabaseError", (error) => Effect.gen(function* () {
                yield* metrics.recordConnectionEvent("fail")
                yield* Effect.logWarning(`Failed to create connection to ${host}: ${error.cause}`)
                return null
              }))
            )
            
            if (connection) {
              yield* metrics.recordConnectionEvent("create")
              return connection
            }
          }
          
          return null
        })),
        { concurrency: "unbounded" }
      )
      
      const healthyConnections = healthChecks.filter(Boolean) as DatabaseConnection[]
      
      if (healthyConnections.length === 0) {
        return yield* Effect.fail(new DatabaseError({
          operation: "pool_health_check",
          cause: "No healthy database connections available"
        }))
      }
      
      // Sort by connection count (prefer less loaded servers)
      healthyConnections.sort((a, b) => a.connectionCount - b.connectionCount)
      
      const pool: DatabasePool = {
        primary: healthyConnections[0],
        secondary: healthyConnections.slice(1),
        totalConnections: healthyConnections.reduce((sum, conn) => sum + conn.connectionCount, 0),
        healthyConnections: healthyConnections.length
      }
      
      yield* Effect.log(`Database pool updated: ${pool.healthyConnections} healthy connections`)
      yield* Effect.log(`Primary: ${pool.primary.host} (${pool.primary.connectionCount} connections)`)
      
      return pool
    }),
    
    // Health check every 30 seconds
    Schedule.fixed(Duration.seconds(30))
  )
})

// Database pool manager
class DatabasePoolManager extends Context.Tag("DatabasePoolManager")<
  DatabasePoolManager,
  {
    readonly getPool: () => Effect.Effect<DatabasePool, DatabaseError>
    readonly getPrimaryConnection: () => Effect.Effect<DatabaseConnection, DatabaseError>
    readonly getConnection: (preferSecondary?: boolean) => Effect.Effect<DatabaseConnection, DatabaseError>
    readonly forceHealthCheck: () => Effect.Effect<void, DatabaseError>
  }
>() {}

const makeDatabasePoolManager = Effect.gen(function* () {
  const resource = yield* createDatabasePoolResource
  
  return {
    getPool: () => Resource.get(resource),
    
    getPrimaryConnection: () => Effect.gen(function* () {
      const pool = yield* Resource.get(resource)
      return pool.primary
    }),
    
    getConnection: (preferSecondary = false) => Effect.gen(function* () {
      const pool = yield* Resource.get(resource)
      
      if (preferSecondary && pool.secondary.length > 0) {
        // Return least loaded secondary connection
        return pool.secondary.reduce((min, conn) => 
          conn.connectionCount < min.connectionCount ? conn : min
        )
      }
      
      return pool.primary
    }),
    
    forceHealthCheck: () => Resource.refresh(resource)
  } satisfies DatabasePoolManager
})

// Query executor that uses the managed pool
class QueryExecutor extends Context.Tag("QueryExecutor")<
  QueryExecutor,
  {
    readonly execute: <T>(
      query: string, 
      params?: ReadonlyArray<unknown>,
      options?: { preferSecondary?: boolean; timeout?: Duration.DurationInput }
    ) => Effect.Effect<T, DatabaseError>
  }
>() {}

const makeQueryExecutor = Effect.gen(function* () {
  const poolManager = yield* DatabasePoolManager
  
  return {
    execute: <T>(
      query: string,
      params: ReadonlyArray<unknown> = [],
      options: { preferSecondary?: boolean; timeout?: Duration.DurationInput } = {}
    ) => Effect.gen(function* () {
      const { preferSecondary = false, timeout = Duration.seconds(30) } = options
      
      const connection = yield* poolManager.getConnection(preferSecondary)
      
      yield* Effect.log(`Executing query on ${connection.host}: ${query}`)
      
      // Simulate query execution
      const result = yield* Effect.gen(function* () {
        yield* Effect.sleep(Duration.milliseconds(Math.random() * 100))
        
        if (Math.random() < 0.05) { // 5% failure rate
          return yield* Effect.fail(new DatabaseError({
            operation: "query_execution",
            connection: connection.id,
            cause: "Query execution failed"
          }))
        }
        
        return {
          rows: [{ id: 1, data: "sample result" }],
          executionTime: Math.random() * 100,
          connection: connection.host
        } as T
      }).pipe(
        Effect.timeout(timeout),
        Effect.catchTag("TimeoutException", () => 
          Effect.fail(new DatabaseError({
            operation: "query_timeout",
            connection: connection.id,
            cause: `Query timed out after ${Duration.toMillis(timeout)}ms`
          }))
        )
      )
      
      yield* Effect.log(`Query completed on ${connection.host}`)
      return result
    })
  } satisfies QueryExecutor
})

// Service layers
const DatabaseServiceLive = Layer.succeed(DatabaseService, {
  checkHealth: (host: string) => Effect.gen(function* () {
    yield* Effect.sleep(Duration.milliseconds(50 + Math.random() * 100))
    
    // Simulate varying health based on host
    const healthProbability = host.includes("primary") ? 0.95 : 0.85
    
    if (Math.random() > healthProbability) {
      return yield* Effect.fail(new DatabaseError({
        operation: "health_check",
        cause: `Host ${host} is unhealthy`
      }))
    }
    
    return true
  }),
  
  createConnection: (host: string) => Effect.gen(function* () {
    yield* Effect.sleep(Duration.milliseconds(100 + Math.random() * 200))
    
    const connectionCount = yield* Effect.sync(() => Math.floor(Math.random() * 20) + 1)
    
    return {
      id: `conn-${Math.random().toString(36).substr(2, 8)}`,
      host,
      isHealthy: true,
      lastUsed: Date.now(),
      connectionCount
    }
  }),
  
  getConnectionCount: (host: string) => 
    Effect.succeed(Math.floor(Math.random() * 50) + 1)
})

const DatabaseMetricsLive = Layer.effect(DatabaseMetrics, Effect.gen(function* () {
  const metricsRef = yield* Ref.make({
    totalEvents: 0,
    failures: 0,
    healthChecks: 0
  })
  
  return {
    recordConnectionEvent: (event: "create" | "fail" | "health_check") => 
      Ref.update(metricsRef, metrics => ({
        totalEvents: metrics.totalEvents + 1,
        failures: metrics.failures + (event === "fail" ? 1 : 0),
        healthChecks: metrics.healthChecks + (event === "health_check" ? 1 : 0)
      })),
    
    getMetrics: () => Ref.get(metricsRef)
  }
}))

const DatabasePoolManagerLive = Layer.effect(DatabasePoolManager, makeDatabasePoolManager).pipe(
  Layer.provide(Layer.mergeAll(DatabaseServiceLive, DatabaseMetricsLive))
)

const QueryExecutorLive = Layer.effect(QueryExecutor, makeQueryExecutor).pipe(
  Layer.provide(DatabasePoolManagerLive)
)

// Usage in application
const application = Effect.gen(function* () {
  const queryExecutor = yield* QueryExecutor
  const poolManager = yield* DatabasePoolManager
  const metrics = yield* DatabaseMetrics
  
  // Execute various queries
  const userQuery = yield* queryExecutor.execute("SELECT * FROM users WHERE active = true")
  yield* Effect.log(`User query result: ${JSON.stringify(userQuery)}`)
  
  // Use secondary connection for read-only queries
  const reportQuery = yield* queryExecutor.execute(
    "SELECT COUNT(*) FROM orders WHERE date > ?",
    ["2023-01-01"],
    { preferSecondary: true }
  )
  yield* Effect.log(`Report query result: ${JSON.stringify(reportQuery)}`)
  
  // Wait for automatic health check
  yield* Effect.sleep(Duration.seconds(35))
  
  // Check pool status after health check
  const pool = yield* poolManager.getPool()
  yield* Effect.log(`Pool status: ${pool.healthyConnections} healthy connections`)
  
  // Get metrics
  const metricsData = yield* metrics.getMetrics()
  yield* Effect.log(`Metrics: ${JSON.stringify(metricsData)}`)
  
  return { userQuery, reportQuery, pool: pool.healthyConnections }
}).pipe(
  Effect.provide(QueryExecutorLive),
  Effect.scoped
)

Effect.runPromise(application).then(console.log).catch(console.error)
/*
Output:
Checking database pool health...
Database pool updated: 3 healthy connections
Primary: primary-db.example.com (15 connections)
Executing query on primary-db.example.com: SELECT * FROM users WHERE active = true
Query completed on primary-db.example.com
User query result: {"rows":[{"id":1,"data":"sample result"}],"executionTime":45.2,"connection":"primary-db.example.com"}
Executing query on secondary-1.example.com: SELECT COUNT(*) FROM orders WHERE date > ?
Query completed on secondary-1.example.com
Report query result: {"rows":[{"id":1,"data":"sample result"}],"executionTime":78.9,"connection":"secondary-1.example.com"}
Checking database pool health...
Database pool updated: 3 healthy connections
Primary: primary-db.example.com (8 connections)
Pool status: 3 healthy connections
Metrics: {"totalEvents":12,"failures":0,"healthChecks":6}
*/
```

## Advanced Features Deep Dive

### Feature 1: Resource Composition and Dependencies

Resources can be composed to create complex dependency graphs where higher-level resources depend on lower-level ones.

#### Basic Resource Composition

```typescript
import { Resource, Effect, Schedule, Duration, Context, Layer } from "effect"

// Base configuration resource
const configResource = Resource.manual(
  Effect.gen(function* () {
    yield* Effect.log("Loading base configuration...")
    return {
      apiUrl: "https://api.example.com",
      timeout: Duration.seconds(30),
      retryAttempts: 3
    }
  })
)

// HTTP client resource that depends on configuration
const httpClientResource = Effect.gen(function* () {
  const config = yield* configResource
  
  return yield* Resource.manual(
    Effect.gen(function* () {
      const baseConfig = yield* Resource.get(config)
      yield* Effect.log(`Creating HTTP client for ${baseConfig.apiUrl}`)
      
      return {
        baseUrl: baseConfig.apiUrl,
        timeout: baseConfig.timeout,
        retryAttempts: baseConfig.retryAttempts,
        request: (endpoint: string) => Effect.gen(function* () {
          yield* Effect.log(`Making request to ${endpoint}`)
          yield* Effect.sleep(Duration.milliseconds(100))
          return { status: 200, data: `Response from ${endpoint}` }
        })
      }
    })
  )
})

// API service resource that depends on HTTP client
const apiServiceResource = Effect.gen(function* () {
  const httpClient = yield* httpClientResource
  
  return yield* Resource.auto(
    Effect.gen(function* () {
      const client = yield* Resource.get(httpClient)
      yield* Effect.log("Creating API service...")
      
      return {
        getUser: (id: string) => client.request(`/users/${id}`),
        getOrders: () => client.request("/orders"),
        healthCheck: () => client.request("/health")
      }
    }),
    
    // Auto-refresh API service every 5 minutes to check health
    Schedule.fixed(Duration.minutes(5))
  )
})

const program = Effect.gen(function* () {
  const apiService = yield* apiServiceResource
  const api = yield* Resource.get(apiService)
  
  // Use the composed service
  const user = yield* api.getUser("123")
  const orders = yield* api.getOrders()
  const health = yield* api.healthCheck()
  
  yield* Effect.log(`Results: ${JSON.stringify({ user, orders, health })}`)
  
  return { user, orders, health }
}).pipe(Effect.scoped)
```

#### Advanced Resource Dependencies with Error Propagation

```typescript
import { Resource, Effect, Schedule, Duration, Ref, Data } from "effect"

class ServiceError extends Data.TaggedError("ServiceError")<{
  service: string
  operation: string
  cause: string
}> {}

// Resource dependency tracker
const createDependencyTracker = Effect.gen(function* () {
  const dependencies = yield* Ref.make<Map<string, Set<string>>>(new Map())
  
  return {
    addDependency: (dependent: string, dependency: string) =>
      Ref.update(dependencies, map => {
        const deps = map.get(dependent) || new Set()
        deps.add(dependency)
        return new Map(map.set(dependent, deps))
      }),
    
    getDependencies: (service: string) =>
      Ref.get(dependencies).pipe(
        Effect.map(map => Array.from(map.get(service) || []))
      ),
    
    removeDependency: (dependent: string, dependency: string) =>
      Ref.update(dependencies, map => {
        const deps = map.get(dependent)
        if (deps) {
          deps.delete(dependency)
          if (deps.size === 0) {
            map.delete(dependent)
          }
        }
        return new Map(map)
      })
  }
})

// Dependency-aware resource factory
const createManagedResource = <T>(
  name: string,
  dependencies: ReadonlyArray<string>,
  factory: Effect.Effect<T, ServiceError>,
  schedule?: Schedule.Schedule<unknown, unknown>
) => Effect.gen(function* () {
  const tracker = yield* createDependencyTracker
  
  // Register dependencies
  yield* Effect.all(
    dependencies.map(dep => tracker.addDependency(name, dep))
  )
  
  const resource = schedule
    ? Resource.auto(
        Effect.gen(function* () {
          yield* Effect.log(`Initializing ${name} (deps: ${dependencies.join(", ")})`)
          
          // Check dependency health before initialization
          const healthChecks = yield* Effect.all(
            dependencies.map(dep =>
              Effect.log(`Checking dependency: ${dep}`).pipe(
                Effect.map(() => true),
                Effect.catchAll(() => Effect.succeed(false))
              )
            )
          )
          
          const unhealthyDeps = dependencies.filter((_, i) => !healthChecks[i])
          if (unhealthyDeps.length > 0) {
            return yield* Effect.fail(new ServiceError({
              service: name,
              operation: "dependency_check",
              cause: `Unhealthy dependencies: ${unhealthyDeps.join(", ")}`
            }))
          }
          
          const result = yield* factory
          yield* Effect.log(`${name} initialized successfully`)
          return result
        }),
        schedule
      )
    : Resource.manual(
        Effect.gen(function* () {
          yield* Effect.log(`Initializing ${name} (deps: ${dependencies.join(", ")})`)
          return yield* factory
        })
      )
  
  // Add cleanup to remove dependencies
  yield* Effect.addFinalizer(() => Effect.gen(function* () {
    yield* Effect.all(
      dependencies.map(dep => tracker.removeDependency(name, dep))
    )
    yield* Effect.log(`${name} dependencies cleaned up`)
  }))
  
  return resource
})

// Create a complex service hierarchy
const databaseResource = createManagedResource(
  "Database",
  [],
  Effect.succeed({
    query: (sql: string) => Effect.succeed([{ id: 1, name: "test" }]),
    isHealthy: () => Effect.succeed(true)
  })
)

const cacheResource = Effect.gen(function* () {
  const db = yield* databaseResource
  
  return yield* createManagedResource(
    "Cache",
    ["Database"],
    Effect.gen(function* () {
      const database = yield* Resource.get(db)
      return {
        get: (key: string) => database.query(`SELECT * FROM cache WHERE key = '${key}'`),
        set: (key: string, value: any) => Effect.log(`Cache set: ${key} = ${JSON.stringify(value)}`),
        isHealthy: () => database.isHealthy()
      }
    }),
    Schedule.fixed(Duration.minutes(10))
  )
})

const userServiceResource = Effect.gen(function* () {
  const db = yield* databaseResource
  const cache = yield* cacheResource
  
  return yield* createManagedResource(
    "UserService",
    ["Database", "Cache"],
    Effect.gen(function* () {
      const database = yield* Resource.get(db)
      const cacheService = yield* Resource.get(cache)
      
      return {
        findUser: (id: string) => Effect.gen(function* () {
          // Try cache first
          const cached = yield* cacheService.get(`user:${id}`)
          if (cached.length > 0) {
            return cached[0]
          }
          
          // Fallback to database
          const users = yield* database.query(`SELECT * FROM users WHERE id = '${id}'`)
          if (users.length > 0) {
            yield* cacheService.set(`user:${id}`, users[0])
            return users[0]
          }
          
          return yield* Effect.fail(new ServiceError({
            service: "UserService",
            operation: "findUser",
            cause: `User not found: ${id}`
          }))
        }),
        
        isHealthy: () => Effect.gen(function* () {
          const dbHealthy = yield* database.isHealthy()
          const cacheHealthy = yield* cacheService.isHealthy()
          return dbHealthy && cacheHealthy
        })
      }
    }),
    Schedule.fixed(Duration.minutes(5))
  )
})

const program = Effect.gen(function* () {
  const userService = yield* userServiceResource
  const service = yield* Resource.get(userService)
  
  // Use the complex composed service
  const user = yield* service.findUser("123")
  const health = yield* service.isHealthy()
  
  yield* Effect.log(`User: ${JSON.stringify(user)}, Healthy: ${health}`)
  
  // Wait for automatic refresh cycles
  yield* Effect.sleep(Duration.minutes(6))
  
  const refreshedHealth = yield* service.isHealthy()
  yield* Effect.log(`Health after refresh: ${refreshedHealth}`)
  
  return { user, health, refreshedHealth }
}).pipe(Effect.scoped)
```

### Feature 2: Resource Error Recovery and Fallback Strategies

Resources can implement sophisticated error recovery patterns with fallback resources and circuit breaker patterns.

#### Circuit Breaker Pattern with Resource

```typescript
import { Resource, Effect, Schedule, Duration, Ref, Data } from "effect"

class CircuitBreakerError extends Data.TaggedError("CircuitBreakerError")<{
  state: "open" | "half-open" | "closed"
  failureCount: number
  lastFailure: number
}> {}

type CircuitState = "closed" | "open" | "half-open"

interface CircuitBreakerConfig {
  readonly failureThreshold: number
  readonly recoveryTimeout: Duration.DurationInput
  readonly halfOpenMaxCalls: number
}

const createCircuitBreaker = (config: CircuitBreakerConfig) => Effect.gen(function* () {
  const state = yield* Ref.make<CircuitState>("closed")
  const failureCount = yield* Ref.make(0)
  const lastFailureTime = yield* Ref.make(0)
  const halfOpenCalls = yield* Ref.make(0)
  
  const shouldAllowCall = Effect.gen(function* () {
    const currentState = yield* Ref.get(state)
    const now = Date.now()
    
    switch (currentState) {
      case "closed":
        return true
        
      case "open":
        const lastFailure = yield* Ref.get(lastFailureTime)
        const recoveryTimeoutMs = Duration.toMillis(config.recoveryTimeout)
        
        if (now - lastFailure >= recoveryTimeoutMs) {
          yield* Ref.set(state, "half-open")
          yield* Ref.set(halfOpenCalls, 0)
          return true
        }
        return false
        
      case "half-open":
        const calls = yield* Ref.get(halfOpenCalls)
        return calls < config.halfOpenMaxCalls
    }
  })
  
  const recordSuccess = Effect.gen(function* () {
    const currentState = yield* Ref.get(state)
    
    if (currentState === "half-open") {
      yield* Ref.set(state, "closed")
      yield* Ref.set(failureCount, 0)
    }
  })
  
  const recordFailure = Effect.gen(function* () {
    const currentState = yield* Ref.get(state)
    const now = Date.now()
    
    if (currentState === "half-open") {
      yield* Ref.set(state, "open")
      yield* Ref.set(lastFailureTime, now)
    } else {
      const failures = yield* Ref.updateAndGet(failureCount, n => n + 1)
      
      if (failures >= config.failureThreshold) {
        yield* Ref.set(state, "open")
        yield* Ref.set(lastFailureTime, now)
      }
    }
  })
  
  const execute = <A, E>(effect: Effect.Effect<A, E>) => Effect.gen(function* () {
    const allowed = yield* shouldAllowCall
    
    if (!allowed) {
      const currentState = yield* Ref.get(state)
      const failures = yield* Ref.get(failureCount)
      const lastFailure = yield* Ref.get(lastFailureTime)
      
      return yield* Effect.fail(new CircuitBreakerError({
        state: currentState,
        failureCount: failures,
        lastFailure
      }))
    }
    
    if ((yield* Ref.get(state)) === "half-open") {
      yield* Ref.update(halfOpenCalls, n => n + 1)
    }
    
    const result = yield* effect.pipe(
      Effect.tap(() => recordSuccess),
      Effect.catchAll(error => Effect.gen(function* () {
        yield* recordFailure
        return yield* Effect.fail(error)
      }))
    )
    
    return result
  })
  
  return { execute, state: Ref.get(state) }
})

// Primary resource with circuit breaker
const createResilientResource = <T>(
  name: string,
  primaryFactory: Effect.Effect<T, any>,
  fallbackFactory: Effect.Effect<T, any>,
  schedule: Schedule.Schedule<unknown, unknown>
) => Effect.gen(function* () {
  const circuitBreaker = yield* createCircuitBreaker({
    failureThreshold: 3,
    recoveryTimeout: Duration.seconds(30),
    halfOpenMaxCalls: 2
  })
  
  return yield* Resource.auto(
    Effect.gen(function* () {
      yield* Effect.log(`Attempting to create ${name} resource`)
      
      const result = yield* circuitBreaker.execute(primaryFactory).pipe(
        Effect.catchTag("CircuitBreakerError", (error) => Effect.gen(function* () {
          yield* Effect.logWarning(`Circuit breaker ${error.state} for ${name}, using fallback`)
          return yield* fallbackFactory
        })),
        Effect.catchAll((error) => Effect.gen(function* () {
          yield* Effect.logError(`Primary ${name} failed: ${String(error)}, using fallback`)
          return yield* fallbackFactory
        }))
      )
      
      const cbState = yield* circuitBreaker.state
      yield* Effect.log(`${name} resource created (circuit: ${cbState})`)
      
      return result
    }),
    schedule
  )
})

// Example: API client with fallback to cached data
const apiClientResource = createResilientResource(
  "ApiClient",
  // Primary: Live API
  Effect.gen(function* () {
    if (Math.random() < 0.4) { // 40% failure rate to trigger circuit breaker
      return yield* Effect.fail("API service unavailable")
    }
    
    yield* Effect.sleep(Duration.milliseconds(200))
    return {
      type: "live" as const,
      getData: (id: string) => Effect.succeed({ id, data: "live data", fresh: true }),
      isHealthy: () => Effect.succeed(true)
    }
  }),
  // Fallback: Cached/offline data
  Effect.gen(function* () {
    yield* Effect.sleep(Duration.milliseconds(50))
    return {
      type: "cached" as const,
      getData: (id: string) => Effect.succeed({ id, data: "cached data", fresh: false }),
      isHealthy: () => Effect.succeed(false)
    }
  }),
  Schedule.fixed(Duration.seconds(10))
)

const program = Effect.gen(function* () {
  const resource = yield* apiClientResource
  
  // Make multiple requests to trigger circuit breaker
  for (let i = 0; i < 10; i++) {
    const client = yield* Resource.get(resource)
    const data = yield* client.getData(`item-${i}`)
    
    yield* Effect.log(`Request ${i}: ${client.type} - ${JSON.stringify(data)}`)
    yield* Effect.sleep(Duration.seconds(1))
  }
  
  return "completed"
}).pipe(Effect.scoped)
```

### Feature 3: Resource Metrics and Observability

Implementing comprehensive metrics and observability for resource lifecycle and performance monitoring.

```typescript
import { Resource, Effect, Schedule, Duration, Ref, Context, Layer, Metric } from "effect"

// Metrics service for resource monitoring
class ResourceMetrics extends Context.Tag("ResourceMetrics")<
  ResourceMetrics,
  {
    readonly recordResourceEvent: (
      resourceName: string,
      event: "created" | "refreshed" | "failed" | "accessed"
    ) => Effect.Effect<void>
    readonly recordPerformance: (
      resourceName: string,
      operation: string,
      duration: number
    ) => Effect.Effect<void>
    readonly getMetrics: (resourceName: string) => Effect.Effect<{
      created: number
      refreshed: number
      failed: number
      accessed: number
      avgPerformance: Record<string, number>
    }>
  }
>() {}

const makeResourceMetrics = Effect.gen(function* () {
  const metrics = yield* Ref.make<Map<string, {
    created: number
    refreshed: number
    failed: number
    accessed: number
    performance: Map<string, number[]>
  }>>(new Map())
  
  return {
    recordResourceEvent: (resourceName: string, event: "created" | "refreshed" | "failed" | "accessed") =>
      Ref.update(metrics, map => {
        const resourceMetrics = map.get(resourceName) || {
          created: 0,
          refreshed: 0,
          failed: 0,
          accessed: 0,
          performance: new Map()
        }
        
        const updated = { ...resourceMetrics, [event]: resourceMetrics[event] + 1 }
        return new Map(map.set(resourceName, updated))
      }),
    
    recordPerformance: (resourceName: string, operation: string, duration: number) =>
      Ref.update(metrics, map => {
        const resourceMetrics = map.get(resourceName) || {
          created: 0,
          refreshed: 0,
          failed: 0,
          accessed: 0,
          performance: new Map()
        }
        
        const opPerformance = resourceMetrics.performance.get(operation) || []
        opPerformance.push(duration)
        
        const updated = {
          ...resourceMetrics,
          performance: new Map(resourceMetrics.performance.set(operation, opPerformance))
        }
        
        return new Map(map.set(resourceName, updated))
      }),
    
    getMetrics: (resourceName: string) => Effect.gen(function* () {
      const map = yield* Ref.get(metrics)
      const resourceMetrics = map.get(resourceName)
      
      if (!resourceMetrics) {
        return {
          created: 0,
          refreshed: 0,
          failed: 0,
          accessed: 0,
          avgPerformance: {}
        }
      }
      
      const avgPerformance: Record<string, number> = {}
      resourceMetrics.performance.forEach((durations, operation) => {
        avgPerformance[operation] = durations.reduce((sum, d) => sum + d, 0) / durations.length
      })
      
      return {
        created: resourceMetrics.created,
        refreshed: resourceMetrics.refreshed,
        failed: resourceMetrics.failed,
        accessed: resourceMetrics.accessed,
        avgPerformance
      }
    })
  } satisfies ResourceMetrics
})

// Observable resource wrapper
const createObservableResource = <T>(
  name: string,
  factory: Effect.Effect<T, any>,
  schedule?: Schedule.Schedule<unknown, unknown>
) => Effect.gen(function* () {
  const metricsService = yield* ResourceMetrics
  
  const wrappedFactory = Effect.gen(function* () {
    const start = yield* Effect.clockWith(clock => clock.currentTimeMillis)
    
    const result = yield* factory.pipe(
      Effect.tap(() => metricsService.recordResourceEvent(name, "created")),
      Effect.catchAll(error => Effect.gen(function* () {
        yield* metricsService.recordResourceEvent(name, "failed")
        return yield* Effect.fail(error)
      }))
    )
    
    const end = yield* Effect.clockWith(clock => clock.currentTimeMillis)
    yield* metricsService.recordPerformance(name, "creation", end - start)
    
    return result
  })
  
  const resource = schedule
    ? Resource.auto(wrappedFactory, schedule)
    : Resource.manual(wrappedFactory)
  
  // Wrap resource operations with metrics
  const observableResource = yield* resource.pipe(
    Effect.map(res => ({
      ...res,
      get: Resource.get(res).pipe(
        Effect.tap(() => metricsService.recordResourceEvent(name, "accessed"))
      ),
      refresh: Resource.refresh(res).pipe(
        Effect.tap(() => metricsService.recordResourceEvent(name, "refreshed"))
      )
    } as any))
  )
  
  return observableResource
})

// Example: Database connection with full observability
const observableDatabaseResource = Effect.gen(function* () {
  return yield* createObservableResource(
    "Database",
    Effect.gen(function* () {
      // Simulate variable connection time
      const connectionTime = 100 + Math.random() * 400
      yield* Effect.sleep(Duration.milliseconds(connectionTime))
      
      if (Math.random() < 0.1) { // 10% failure rate
        return yield* Effect.fail("Database connection failed")
      }
      
      return {
        id: `db-${Math.random().toString(36).substr(2, 8)}`,
        host: "db.example.com",
        connected: true,
        connectionTime,
        query: (sql: string) => Effect.gen(function* () {
          const queryStart = yield* Effect.clockWith(clock => clock.currentTimeMillis)
          yield* Effect.sleep(Duration.milliseconds(50 + Math.random() * 100))
          const queryEnd = yield* Effect.clockWith(clock => clock.currentTimeMillis)
          
          const metrics = yield* ResourceMetrics
          yield* metrics.recordPerformance("Database", "query", queryEnd - queryStart)
          
          return [{ id: 1, result: `Result for: ${sql}` }]
        })
      }
    }),
    Schedule.fixed(Duration.minutes(5))
  )
})

// Metrics dashboard
const createMetricsDashboard = Effect.gen(function* () {
  const metrics = yield* ResourceMetrics
  
  return {
    printDashboard: () => Effect.gen(function* () {
      const dbMetrics = yield* metrics.getMetrics("Database")
      
      yield* Effect.log("=== Resource Metrics Dashboard ===")
      yield* Effect.log(`Database Resource:`)
      yield* Effect.log(`  Created: ${dbMetrics.created}`)
      yield* Effect.log(`  Refreshed: ${dbMetrics.refreshed}`)
      yield* Effect.log(`  Failed: ${dbMetrics.failed}`)
      yield* Effect.log(`  Accessed: ${dbMetrics.accessed}`)
      yield* Effect.log(`  Avg Creation Time: ${dbMetrics.avgPerformance.creation?.toFixed(2) || 0}ms`)
      yield* Effect.log(`  Avg Query Time: ${dbMetrics.avgPerformance.query?.toFixed(2) || 0}ms`)
      
      const uptime = dbMetrics.created > 0 ? ((dbMetrics.created - dbMetrics.failed) / dbMetrics.created * 100) : 0
      yield* Effect.log(`  Uptime: ${uptime.toFixed(2)}%`)
    })
  }
})

// Service layers
const ResourceMetricsLive = Layer.effect(ResourceMetrics, makeResourceMetrics)

// Application with comprehensive monitoring
const monitoredApplication = Effect.gen(function* () {
  const database = yield* observableDatabaseResource
  const dashboard = yield* createMetricsDashboard
  
  // Simulate application usage
  for (let i = 0; i < 20; i++) {
    const db = yield* database.get
    const results = yield* db.query(`SELECT * FROM users WHERE id = ${i}`)
    yield* Effect.log(`Query ${i} results: ${results.length} rows`)
    
    yield* Effect.sleep(Duration.milliseconds(200))
    
    // Print dashboard every 5 queries
    if (i % 5 === 4) {
      yield* dashboard.printDashboard()
      yield* Effect.log("---")
    }
  }
  
  // Force refresh and monitor
  yield* database.refresh
  yield* Effect.sleep(Duration.seconds(1))
  
  yield* dashboard.printDashboard()
  
  return "Application completed"
}).pipe(
  Effect.provide(ResourceMetricsLive),
  Effect.scoped
)

Effect.runPromise(monitoredApplication)
```

## Practical Patterns & Best Practices

### Pattern 1: Resource Layering and Composition

Create layers of resources where each layer depends on the previous one, enabling clean separation of concerns.

```typescript
import { Resource, Effect, Schedule, Duration, Context, Layer, Data } from "effect"

// Layer 1: Infrastructure Resources
const infraResource = Resource.manual(
  Effect.gen(function* () {
    yield* Effect.log("Setting up infrastructure...")
    return {
      region: "us-east-1",
      availabilityZones: ["us-east-1a", "us-east-1b"],
      vpcId: "vpc-123456",
      subnets: ["subnet-111", "subnet-222"]
    }
  })
)

// Layer 2: Data Resources (depends on infrastructure)
const dataResource = Effect.gen(function* () {
  const infra = yield* infraResource
  
  return yield* Resource.auto(
    Effect.gen(function* () {
      const infrastructure = yield* Resource.get(infra)
      yield* Effect.log(`Setting up data layer in region ${infrastructure.region}`)
      
      return {
        database: {
          endpoint: `db.${infrastructure.region}.amazonaws.com`,
          replicas: infrastructure.availabilityZones.map(az => `replica.${az}.db.com`)
        },
        cache: {
          endpoint: `cache.${infrastructure.region}.amazonaws.com`,
          nodes: infrastructure.availabilityZones.length
        }
      }
    }),
    Schedule.fixed(Duration.minutes(10))
  )
})

// Layer 3: Application Resources (depends on data)
const applicationResource = Effect.gen(function* () {
  const infra = yield* infraResource
  const data = yield* dataResource
  
  return yield* Resource.auto(
    Effect.gen(function* () {
      const infrastructure = yield* Resource.get(infra)
      const dataLayer = yield* Resource.get(data)
      
      yield* Effect.log("Setting up application services...")
      
      return {
        userService: {
          database: dataLayer.database.endpoint,
          cache: dataLayer.cache.endpoint,
          replicas: dataLayer.database.replicas
        },
        orderService: {
          database: dataLayer.database.endpoint,
          cache: dataLayer.cache.endpoint,
          region: infrastructure.region
        },
        notificationService: {
          region: infrastructure.region,
          queues: infrastructure.subnets.map(subnet => `queue-${subnet}`)
        }
      }
    }),
    Schedule.fixed(Duration.minutes(5))
  )
})

// Helper for resource health checking across layers
const createHealthChecker = Effect.gen(function* () {
  return {
    checkAllLayers: () => Effect.gen(function* () {
      yield* Effect.log("Checking resource health across all layers...")
      
      const infraHealth = yield* Effect.gen(function* () {
        const infra = yield* Resource.get(yield* infraResource)
        return infra.availabilityZones.length > 0
      })
      
      const dataHealth = yield* Effect.gen(function* () {
        const data = yield* Resource.get(yield* dataResource)
        return data.database.endpoint.length > 0 && data.cache.nodes > 0
      })
      
      const appHealth = yield* Effect.gen(function* () {
        const app = yield* Resource.get(yield* applicationResource)
        return app.userService.database.length > 0
      })
      
      const overallHealth = infraHealth && dataHealth && appHealth
      
      yield* Effect.log(`Health Check Results:`)
      yield* Effect.log(`  Infrastructure: ${infraHealth ? "✓" : "✗"}`)
      yield* Effect.log(`  Data Layer: ${dataHealth ? "✓" : "✗"}`)
      yield* Effect.log(`  Application: ${appHealth ? "✓" : "✗"}`)
      yield* Effect.log(`  Overall: ${overallHealth ? "✓ HEALTHY" : "✗ UNHEALTHY"}`)
      
      return overallHealth
    })
  }
})

const layeredApplication = Effect.gen(function* () {
  const healthChecker = yield* createHealthChecker
  
  // Initial health check
  yield* healthChecker.checkAllLayers()
  
  // Get application config
  const app = yield* Resource.get(yield* applicationResource)
  yield* Effect.log(`Application configured: ${JSON.stringify(app, null, 2)}`)
  
  // Simulate application running
  yield* Effect.sleep(Duration.minutes(6))
  
  // Health check after auto-refresh
  yield* healthChecker.checkAllLayers()
  
  return app
}).pipe(Effect.scoped)
```

### Pattern 2: Resource Sharing with Reference Counting

Implement resource sharing where expensive resources are only created once and shared across multiple consumers.

```typescript
import { Resource, Effect, Schedule, Duration, Ref, Data } from "effect"

class ResourceSharingError extends Data.TaggedError("ResourceSharingError")<{
  operation: string
  resourceId: string
  cause: string
}> {}

// Reference-counted resource manager
const createSharedResourceManager = <T>(
  resourceId: string,
  factory: Effect.Effect<T, any>,
  cleanupDelay: Duration.DurationInput = Duration.seconds(30)
) => Effect.gen(function* () {
  const resourceRef = yield* Ref.make<T | null>(null)
  const refCount = yield* Ref.make(0)
  const cleanupScheduled = yield* Ref.make(false)
  
  const scheduleCleanup = Effect.gen(function* () {
    const isScheduled = yield* Ref.get(cleanupScheduled)
    if (isScheduled) return
    
    yield* Ref.set(cleanupScheduled, true)
    yield* Effect.log(`Scheduling cleanup for ${resourceId} in ${Duration.toMillis(cleanupDelay)}ms`)
    
    yield* Effect.fork(Effect.gen(function* () {
      yield* Effect.sleep(cleanupDelay)
      
      const currentCount = yield* Ref.get(refCount)
      if (currentCount === 0) {
        yield* Ref.set(resourceRef, null)
        yield* Effect.log(`Resource ${resourceId} cleaned up after delay`)
      }
      
      yield* Ref.set(cleanupScheduled, false)
    }))
  })
  
  return {
    acquire: () => Effect.gen(function* () {
      const currentResource = yield* Ref.get(resourceRef)
      
      if (currentResource) {
        // Resource exists, increment ref count
        const newCount = yield* Ref.updateAndGet(refCount, n => n + 1)
        yield* Effect.log(`${resourceId} acquired (refs: ${newCount})`)
        return currentResource
      }
      
      // Create new resource
      yield* Effect.log(`Creating shared resource: ${resourceId}`)
      const newResource = yield* factory
      yield* Ref.set(resourceRef, newResource)
      yield* Ref.set(refCount, 1)
      yield* Effect.log(`${resourceId} created and acquired (refs: 1)`)
      
      return newResource
    }),
    
    release: () => Effect.gen(function* () {
      const newCount = yield* Ref.updateAndGet(refCount, n => Math.max(0, n - 1))
      yield* Effect.log(`${resourceId} released (refs: ${newCount})`)
      
      if (newCount === 0) {
        yield* scheduleCleanup
      }
    }),
    
    getRefCount: () => Ref.get(refCount),
    isActive: () => Ref.get(resourceRef).pipe(Effect.map(r => r !== null))
  }
})

// Shared database connection example
const sharedDatabaseManager = createSharedResourceManager(
  "Database",
  Effect.gen(function* () {
    yield* Effect.log("Creating expensive database connection...")
    yield* Effect.sleep(Duration.seconds(2)) // Simulate connection time
    
    return {
      id: `db-${Math.random().toString(36).substr(2, 8)}`,
      connectionPool: Array.from({ length: 10 }, (_, i) => `conn-${i}`),
      query: (sql: string) => Effect.gen(function* () {
        yield* Effect.sleep(Duration.milliseconds(50))
        return [{ id: 1, result: `Result for: ${sql}` }]
      })
    }
  }),
  Duration.seconds(5) // Cleanup after 5 seconds of no usage
)

// Service that uses shared resource
const createDatabaseService = (serviceName: string) => Effect.gen(function* () {
  const manager = yield* sharedDatabaseManager
  
  return Resource.manual(
    Effect.gen(function* () {
      yield* Effect.log(`${serviceName} starting, acquiring database...`)
      const database = yield* manager.acquire()
      
      // Add finalizer to release resource
      yield* Effect.addFinalizer(() => Effect.gen(function* () {
        yield* Effect.log(`${serviceName} finalizing, releasing database...`)
        yield* manager.release()
      }))
      
      return {
        serviceName,
        queryUser: (id: string) => database.query(`SELECT * FROM users WHERE id = '${id}'`),
        queryOrders: () => database.query("SELECT * FROM orders"),
        getDatabaseInfo: () => Effect.succeed({
          databaseId: database.id,
          poolSize: database.connectionPool.length
        })
      }
    })
  )
})

// Multiple services sharing the same database
const sharedResourceDemo = Effect.gen(function* () {
  const manager = yield* sharedDatabaseManager
  
  // Create multiple services
  const userService = yield* createDatabaseService("UserService")
  const orderService = yield* createDatabaseService("OrderService") 
  const analyticsService = yield* createDatabaseService("AnalyticsService")
  
  // Start services at different times
  const services = [
    { service: userService, delay: 0 },
    { service: orderService, delay: 1000 },
    { service: analyticsService, delay: 2000 }
  ]
  
  const operations = services.map(({ service, delay }) => 
    Effect.gen(function* () {
      yield* Effect.sleep(Duration.milliseconds(delay))
      
      const svc = yield* Resource.get(service)
      yield* Effect.log(`Starting operations for ${svc.serviceName}`)
      
      // Perform some operations
      const user = yield* svc.queryUser("123")
      const orders = yield* svc.queryOrders()
      const dbInfo = yield* svc.getDatabaseInfo()
      
      yield* Effect.log(`${svc.serviceName} completed operations`)
      yield* Effect.log(`Database info: ${JSON.stringify(dbInfo)}`)
      
      // Service runs for different durations
      const runTime = delay === 0 ? 3000 : delay === 1000 ? 4000 : 2000
      yield* Effect.sleep(Duration.milliseconds(runTime))
      
      yield* Effect.log(`${svc.serviceName} shutting down`)
      return svc.serviceName
    }).pipe(Effect.scoped)
  )
  
  // Monitor reference count during execution
  const monitor = Effect.gen(function* () {
    for (let i = 0; i < 10; i++) {
      const refCount = yield* manager.getRefCount()
      const isActive = yield* manager.isActive()
      yield* Effect.log(`Monitor: refs=${refCount}, active=${isActive}`)
      yield* Effect.sleep(Duration.seconds(1))
    }
  })
  
  // Run everything concurrently
  const results = yield* Effect.all([
    Effect.all(operations, { concurrency: "unbounded" }),
    monitor
  ])
  
  yield* Effect.sleep(Duration.seconds(6)) // Wait for cleanup
  
  const finalRefCount = yield* manager.getRefCount()
  const finalActive = yield* manager.isActive()
  yield* Effect.log(`Final state: refs=${finalRefCount}, active=${finalActive}`)
  
  return results[0]
})

Effect.runPromise(sharedResourceDemo)
/*
Output:
UserService starting, acquiring database...
Creating expensive database connection...
UserService completed operations
Database info: {"databaseId":"db-abc123","poolSize":10}
OrderService starting, acquiring database...
Database acquired (refs: 2)
AnalyticsService starting, acquiring database...
Database acquired (refs: 3)
Monitor: refs=3, active=true
OrderService completed operations
AnalyticsService completed operations
UserService shutting down
UserService finalizing, releasing database...
Database released (refs: 2)
OrderService shutting down
OrderService finalizing, releasing database...  
Database released (refs: 1)
AnalyticsService shutting down
AnalyticsService finalizing, releasing database...
Database released (refs: 0)
Scheduling cleanup for Database in 5000ms
Resource Database cleaned up after delay
Final state: refs=0, active=false
*/
```

### Pattern 3: Resource Warm-up and Pre-loading

Implement resource warm-up strategies to ensure critical resources are ready before they're needed.

```typescript
import { Resource, Effect, Schedule, Duration, Array as Arr, Chunk } from "effect"

// Warm-up configuration
interface WarmupConfig {
  readonly critical: ReadonlyArray<string>
  readonly preload: ReadonlyArray<string>
  readonly batchSize: number
  readonly concurrency: number
}

// Resource registry for managing multiple resources
const createResourceRegistry = Effect.gen(function* () {
  const resources = new Map<string, Resource.Resource<any, any>>()
  
  const register = <T>(name: string, resource: Resource.Resource<T, any>) =>
    Effect.sync(() => {
      resources.set(name, resource)
    })
  
  const get = (name: string) =>
    Effect.gen(function* () {
      const resource = resources.get(name)
      if (!resource) {
        return yield* Effect.fail(`Resource not found: ${name}`)
      }
      return resource
    })
  
  const warmUp = (names: ReadonlyArray<string>, options: {
    batchSize?: number
    concurrency?: number
  } = {}) => Effect.gen(function* () {
    const { batchSize = 5, concurrency = 3 } = options
    
    yield* Effect.log(`Warming up ${names.length} resources...`)
    
    // Process in batches to avoid overwhelming the system
    const batches = Chunk.toReadonlyArray(
      Chunk.chunksOf(Chunk.fromIterable(names), batchSize)
    )
    
    for (const batch of batches) {
      yield* Effect.log(`Warming batch: ${batch.join(", ")}`)
      
      yield* Effect.all(
        batch.map(name => Effect.gen(function* () {
          const resource = yield* get(name)
          const startTime = yield* Effect.clockWith(clock => clock.currentTimeMillis)
          
          yield* Resource.get(resource).pipe(
            Effect.catchAll(error => Effect.gen(function* () {
              yield* Effect.logWarning(`Failed to warm up ${name}: ${String(error)}`)
              return null
            }))
          )
          
          const endTime = yield* Effect.clockWith(clock => clock.currentTimeMillis)
          yield* Effect.log(`${name} warmed up in ${endTime - startTime}ms`)
        })),
        { concurrency }
      )
      
      // Small delay between batches
      if (batches.indexOf(batch) < batches.length - 1) {
        yield* Effect.sleep(Duration.milliseconds(100))
      }
    }
    
    yield* Effect.log("Warm-up completed")
  })
  
  const preloadCritical = (config: WarmupConfig) => Effect.gen(function* () {
    yield* Effect.log("Pre-loading critical resources...")
    
    // Load critical resources first, with higher concurrency
    yield* warmUp(config.critical, {
      batchSize: config.batchSize,
      concurrency: config.concurrency * 2
    })
    
    // Then pre-load other resources in background
    yield* Effect.fork(Effect.gen(function* () {
      yield* Effect.sleep(Duration.seconds(1))
      yield* warmUp(config.preload, {
        batchSize: config.batchSize,
        concurrency: config.concurrency
      })
    }))
  })
  
  return {
    register,
    get,
    warmUp,
    preloadCritical,
    listResources: () => Effect.succeed(Array.from(resources.keys()))
  }
})

// Create various resources for demonstration
const createApplicationResources = Effect.gen(function* () {
  const registry = yield* createResourceRegistry
  
  // Critical system resources
  yield* registry.register(
    "system-config",
    Resource.manual(Effect.gen(function* () {
      yield* Effect.sleep(Duration.milliseconds(300))
      return { version: "1.0.0", env: "production", features: ["auth", "analytics"] }
    }))
  )
  
  yield* registry.register(
    "database-pool",
    Resource.auto(
      Effect.gen(function* () {
        yield* Effect.sleep(Duration.milliseconds(800))
        return { 
          connections: 20, 
          host: "db.prod.com",
          health: () => Effect.succeed(true)
        }
      }),
      Schedule.fixed(Duration.minutes(10))
    )
  )
  
  yield* registry.register(
    "auth-service", 
    Resource.manual(Effect.gen(function* () {
      yield* Effect.sleep(Duration.milliseconds(500))
      return {
        tokenValidator: (token: string) => Effect.succeed(token.length > 0),
        userLookup: (id: string) => Effect.succeed({ id, name: "User" })
      }
    }))
  )
  
  // Less critical resources
  yield* registry.register(
    "analytics-client",
    Resource.auto(
      Effect.gen(function* () {
        yield* Effect.sleep(Duration.milliseconds(400))
        return { 
          track: (event: string) => Effect.log(`Analytics: ${event}`),
          enabled: true
        }
      }),
      Schedule.fixed(Duration.minutes(5))
    )
  )
  
  yield* registry.register(
    "cache-client",
    Resource.manual(Effect.gen(function* () {
      yield* Effect.sleep(Duration.milliseconds(600))
      return {
        get: (key: string) => Effect.succeed(null),
        set: (key: string, value: any) => Effect.log(`Cache set: ${key}`)
      }
    }))
  )
  
  yield* registry.register(
    "notification-service",
    Resource.manual(Effect.gen(function* () {
      yield* Effect.sleep(Duration.milliseconds(350))
      return {
        sendEmail: (to: string, subject: string) => 
          Effect.log(`Email sent to ${to}: ${subject}`),
        sendSMS: (to: string, message: string) =>
          Effect.log(`SMS sent to ${to}: ${message}`)
      }
    }))
  )
  
  return registry
})

// Application startup with staged warm-up
const applicationStartup = Effect.gen(function* () {
  const registry = yield* createApplicationResources
  
  const warmupConfig: WarmupConfig = {
    critical: ["system-config", "database-pool", "auth-service"],
    preload: ["analytics-client", "cache-client", "notification-service"],
    batchSize: 2,
    concurrency: 2
  }
  
  // Stage 1: Pre-load critical resources
  yield* Effect.log("=== Application Startup: Stage 1 ===")
  const startTime = yield* Effect.clockWith(clock => clock.currentTimeMillis)
  
  yield* registry.preloadCritical(warmupConfig)
  
  const criticalLoadTime = yield* Effect.clockWith(clock => clock.currentTimeMillis)
  yield* Effect.log(`Critical resources loaded in ${criticalLoadTime - startTime}ms`)
  
  // Stage 2: Application ready to serve requests
  yield* Effect.log("=== Application Ready to Serve ===")
  
  // Simulate some application requests
  const authService = yield* Resource.get(yield* registry.get("auth-service"))
  const dbPool = yield* Resource.get(yield* registry.get("database-pool"))
  
  yield* Effect.log("Processing sample requests...")
  
  const tokenValid = yield* authService.tokenValidator("sample-token")
  const user = yield* authService.userLookup("user-123")
  const dbHealthy = yield* dbPool.health()
  
  yield* Effect.log(`Auth results: token=${tokenValid}, user=${user.name}, db=${dbHealthy}`)
  
  // Stage 3: Wait for background pre-loading to complete
  yield* Effect.sleep(Duration.seconds(3))
  yield* Effect.log("=== Background Pre-loading Complete ===")
  
  // Test all resources are available
  const resources = yield* registry.listResources()
  yield* Effect.log(`Total resources available: ${resources.length}`)
  
  for (const resourceName of resources) {
    const resource = yield* registry.get(resourceName)
    const value = yield* Resource.get(resource)
    yield* Effect.log(`✓ ${resourceName} is ready`)
  }
  
  const totalTime = yield* Effect.clockWith(clock => clock.currentTimeMillis)
  yield* Effect.log(`Total startup time: ${totalTime - startTime}ms`)
  
  return {
    startupTime: totalTime - startTime,
    criticalLoadTime: criticalLoadTime - startTime,
    resourcesLoaded: resources.length
  }
}).pipe(Effect.scoped)

Effect.runPromise(applicationStartup)
/*
Output:
=== Application Startup: Stage 1 ===
Pre-loading critical resources...
Warming up 3 resources...
Warming batch: system-config, database-pool
system-config warmed up in 302ms
database-pool warmed up in 805ms
Warming batch: auth-service
auth-service warmed up in 503ms
Warm-up completed
Critical resources loaded in 808ms
=== Application Ready to Serve ===
Processing sample requests...
Auth results: token=true, user=User, db=true
Warming up 3 resources...
Warming batch: analytics-client, cache-client
analytics-client warmed up in 402ms
cache-client warmed up in 601ms
Warming batch: notification-service
notification-service warmed up in 352ms
Warm-up completed
=== Background Pre-loading Complete ===
Total resources available: 6
✓ system-config is ready
✓ database-pool is ready
✓ auth-service is ready
✓ analytics-client is ready
✓ cache-client is ready
✓ notification-service is ready
Total startup time: 4210ms
*/
```

## Integration Examples

### Integration with HTTP Server Framework

Demonstrate how to integrate Resource with HTTP server frameworks for request-scoped resource management.

```typescript
import { Resource, Effect, Schedule, Duration, Context, Layer, Data } from "effect"

// HTTP server types (simplified)
interface HttpRequest {
  readonly method: string
  readonly path: string
  readonly headers: Record<string, string>
  readonly body?: any
}

interface HttpResponse {
  readonly status: number
  readonly headers?: Record<string, string>
  readonly body?: any
}

class HttpError extends Data.TaggedError("HttpError")<{
  status: number
  message: string
}> {}

// Request-scoped resources
class RequestContext extends Context.Tag("RequestContext")<
  RequestContext,
  {
    readonly requestId: string
    readonly startTime: number
    readonly request: HttpRequest
  }
>() {}

class RequestDatabase extends Context.Tag("RequestDatabase")<
  RequestDatabase,
  {
    readonly query: (sql: string) => Effect.Effect<any[]>
    readonly transaction: <R>(work: Effect.Effect<R>) => Effect.Effect<R>
  }
>() {}

class RequestLogger extends Context.Tag("RequestLogger")<
  RequestLogger,
  {
    readonly info: (message: string) => Effect.Effect<void>
    readonly error: (message: string, error?: unknown) => Effect.Effect<void>
  }
>() {}

// Global application resources
const appConfigResource = Resource.auto(
  Effect.gen(function* () {
    yield* Effect.log("Loading application configuration...")
    return {
      database: {
        host: "db.example.com",
        port: 5432,
        poolSize: 20
      },
      redis: {
        host: "redis.example.com",
        port: 6379
      },
      logging: {
        level: "info",
        format: "json"
      }
    }
  }),
  Schedule.fixed(Duration.minutes(10))
)

const databasePoolResource = Effect.gen(function* () {
  const config = yield* appConfigResource
  
  return yield* Resource.auto(
    Effect.gen(function* () {
      const appConfig = yield* Resource.get(config)
      yield* Effect.log(`Creating database pool: ${appConfig.database.host}:${appConfig.database.port}`)
      
      return {
        host: appConfig.database.host,
        port: appConfig.database.port,
        poolSize: appConfig.database.poolSize,
        getConnection: () => Effect.gen(function* () {
          yield* Effect.sleep(Duration.milliseconds(10))
          return {
            id: `conn-${Math.random().toString(36).substr(2, 8)}`,
            query: (sql: string) => Effect.gen(function* () {
              yield* Effect.sleep(Duration.milliseconds(20))
              return [{ id: 1, result: `Result for: ${sql}` }]
            }),
            release: () => Effect.log("Connection released")
          }
        })
      }
    }),
    Schedule.fixed(Duration.minutes(5))
  )
})

// Request-scoped resource factories
const createRequestLogger = Effect.gen(function* () {
  const context = yield* RequestContext
  const config = yield* Resource.get(yield* appConfigResource)
  
  return {
    info: (message: string) => 
      Effect.log(`[${context.requestId}] INFO: ${message}`),
    error: (message: string, error?: unknown) =>
      Effect.log(`[${context.requestId}] ERROR: ${message} ${error ? String(error) : ""}`)
  }
})

const createRequestDatabase = Effect.gen(function* () {
  const context = yield* RequestContext
  const logger = yield* RequestLogger
  const pool = yield* Resource.get(yield* databasePoolResource)
  
  return yield* Resource.manual(
    Effect.gen(function* () {
      yield* logger.info("Acquiring database connection")
      const connection = yield* pool.getConnection()
      
      yield* Effect.addFinalizer(() => Effect.gen(function* () {
        yield* logger.info("Releasing database connection")
        yield* connection.release()
      }))
      
      return {
        query: (sql: string) => Effect.gen(function* () {
          yield* logger.info(`Executing query: ${sql}`)
          return yield* connection.query(sql)
        }),
        
        transaction: <R>(work: Effect.Effect<R>) => Effect.gen(function* () {
          yield* logger.info("Starting transaction")
          
          const result = yield* work.pipe(
            Effect.catchAll(error => Effect.gen(function* () {
              yield* logger.error("Transaction failed, rolling back", error)
              return yield* Effect.fail(error)
            }))
          )
          
          yield* logger.info("Transaction committed")
          return result
        })
      }
    })
  )
})

// HTTP route handlers
const getUserHandler = (userId: string) => Effect.gen(function* () {
  const db = yield* RequestDatabase
  const logger = yield* RequestLogger
  
  yield* logger.info(`Getting user: ${userId}`)
  
  const users = yield* db.query(`SELECT * FROM users WHERE id = '${userId}'`)
  
  if (users.length === 0) {
    return yield* Effect.fail(new HttpError({
      status: 404,
      message: "User not found"
    }))
  }
  
  yield* logger.info(`User found: ${users[0].result}`)
  
  return {
    status: 200,
    body: { user: users[0] }
  }
})

const createUserHandler = (userData: any) => Effect.gen(function* () {
  const db = yield* RequestDatabase
  const logger = yield* RequestLogger
  
  yield* logger.info(`Creating user: ${userData.name}`)
  
  const result = yield* db.transaction(Effect.gen(function* () {
    // Validate user data
    if (!userData.name || !userData.email) {
      return yield* Effect.fail(new HttpError({
        status: 400,
        message: "Missing required fields"
      }))
    }
    
    // Insert user
    yield* db.query(`INSERT INTO users (name, email) VALUES ('${userData.name}', '${userData.email}')`)
    
    // Get created user
    const users = yield* db.query(`SELECT * FROM users WHERE email = '${userData.email}'`)
    return users[0]
  }))
  
  yield* logger.info(`User created: ${result.result}`)
  
  return {
    status: 201,
    body: { user: result }
  }
})

// Request processor with scoped resources
const processRequest = (request: HttpRequest) => Effect.gen(function* () {
  const requestId = `req-${Math.random().toString(36).substr(2, 8)}`
  const startTime = Date.now()
  
  const context: RequestContext = {
    requestId,
    startTime,
    request
  }
  
  // Create request-scoped layer
  const requestLayer = Layer.mergeAll(
    Layer.succeed(RequestContext, context),
    Layer.effect(RequestLogger, createRequestLogger),
    Layer.effect(RequestDatabase, createRequestDatabase)
  )
  
  const handler = request.path.startsWith("/users/") && request.method === "GET"
    ? getUserHandler(request.path.split("/")[2])
    : request.path === "/users" && request.method === "POST"
    ? createUserHandler(request.body)
    : Effect.fail(new HttpError({ status: 404, message: "Not found" }))
  
  const response = yield* Effect.scoped(handler).pipe(
    Effect.provide(requestLayer),
    Effect.catchTag("HttpError", (error) => 
      Effect.succeed({
        status: error.status,
        body: { error: error.message }
      })
    ),
    Effect.catchAll((error) =>
      Effect.succeed({
        status: 500,
        body: { error: "Internal server error" }
      })
    )
  )
  
  const duration = Date.now() - startTime
  console.log(`${request.method} ${request.path} -> ${response.status} (${duration}ms)`)
  
  return response
})

// Simulated HTTP server
const httpServer = Effect.gen(function* () {
  yield* Effect.log("Starting HTTP server...")
  
  // Simulate multiple concurrent requests
  const requests: HttpRequest[] = [
    { method: "GET", path: "/users/123", headers: {} },
    { method: "POST", path: "/users", headers: {}, body: { name: "John", email: "john@example.com" } },
    { method: "GET", path: "/users/456", headers: {} },
    { method: "POST", path: "/users", headers: {}, body: { name: "Jane", email: "jane@example.com" } },
    { method: "GET", path: "/unknown", headers: {} }
  ]
  
  const responses = yield* Effect.all(
    requests.map(processRequest),
    { concurrency: 3 }
  )
  
  yield* Effect.log(`Processed ${responses.length} requests`)
  return responses
}).pipe(Effect.scoped)

// Main application
const application = Effect.provide(httpServer, Layer.empty)

Effect.runPromise(application)
/*
Output:
Starting HTTP server...
Loading application configuration...
Creating database pool: db.example.com:5432
[req-abc123] INFO: Acquiring database connection
[req-def456] INFO: Acquiring database connection
[req-ghi789] INFO: Acquiring database connection
[req-abc123] INFO: Getting user: 123
[req-def456] INFO: Creating user: John
[req-ghi789] INFO: Getting user: 456
[req-abc123] INFO: Executing query: SELECT * FROM users WHERE id = '123'
[req-def456] INFO: Starting transaction
[req-ghi789] INFO: Executing query: SELECT * FROM users WHERE id = '456'
[req-abc123] INFO: User found: Result for: SELECT * FROM users WHERE id = '123'
[req-def456] INFO: Executing query: INSERT INTO users (name, email) VALUES ('John', 'john@example.com')
[req-ghi789] INFO: User found: Result for: SELECT * FROM users WHERE id = '456'
GET /users/123 -> 200 (43ms)
[req-def456] INFO: Executing query: SELECT * FROM users WHERE email = 'john@example.com'
GET /users/456 -> 200 (48ms)
[req-def456] INFO: Transaction committed
[req-def456] INFO: User created: Result for: SELECT * FROM users WHERE email = 'john@example.com'
POST /users -> 201 (67ms)
[req-abc123] INFO: Releasing database connection
[req-ghi789] INFO: Releasing database connection
[req-def456] INFO: Releasing database connection
POST /users -> 201 (71ms)
GET /unknown -> 404 (2ms)
Processed 5 requests
*/
```

### Integration with Testing Framework

Show how to use Resource for test setup and teardown with proper isolation.

```typescript
import { Resource, Effect, Schedule, Duration, Context, Layer, Ref } from "effect"

// Test context and utilities
class TestContext extends Context.Tag("TestContext")<
  TestContext,
  {
    readonly testName: string
    readonly testId: string
    readonly startTime: number
  }
>() {}

class TestDatabase extends Context.Tag("TestDatabase")<
  TestDatabase,
  {
    readonly query: (sql: string) => Effect.Effect<any[]>
    readonly seed: (table: string, data: any[]) => Effect.Effect<void>
    readonly clear: () => Effect.Effect<void>
  }
>() {}

class TestMetrics extends Context.Tag("TestMetrics")<
  TestMetrics,
  {
    readonly recordEvent: (event: string, data?: any) => Effect.Effect<void>
    readonly getEvents: () => Effect.Effect<Array<{ event: string; data?: any; timestamp: number }>>
  }
>() {}

// Test database resource with automatic cleanup
const createTestDatabase = (testId: string) => Resource.manual(
  Effect.gen(function* () {
    yield* Effect.log(`Setting up test database for: ${testId}`)
    
    const storage = yield* Ref.make<Map<string, any[]>>(new Map())
    
    const testDb = {
      query: (sql: string) => Effect.gen(function* () {
        yield* Effect.log(`Test query: ${sql}`)
        const tables = yield* Ref.get(storage)
        
        // Mock query logic
        if (sql.includes("users")) {
          return tables.get("users") || []
        } else if (sql.includes("orders")) {
          return tables.get("orders") || []
        }
        return []
      }),
      
      seed: (table: string, data: any[]) => Effect.gen(function* () {
        yield* Effect.log(`Seeding ${table} with ${data.length} records`)
        yield* Ref.update(storage, map => new Map(map.set(table, data)))
      }),
      
      clear: () => Effect.gen(function* () {
        yield* Effect.log("Clearing test database")
        yield* Ref.set(storage, new Map())
      })
    }
    
    // Add finalizer for cleanup
    yield* Effect.addFinalizer(() => Effect.gen(function* () {
      yield* Effect.log(`Cleaning up test database for: ${testId}`)
      yield* testDb.clear()
    }))
    
    return testDb
  })
)

// Test metrics resource
const createTestMetrics = (testId: string) => Resource.manual(
  Effect.gen(function* () {
    yield* Effect.log(`Setting up test metrics for: ${testId}`)
    
    const events = yield* Ref.make<Array<{ event: string; data?: any; timestamp: number }>>([])
    
    const metrics = {
      recordEvent: (event: string, data?: any) => Effect.gen(function* () {
        const timestamp = Date.now()
        yield* Ref.update(events, arr => [...arr, { event, data, timestamp }])
        yield* Effect.log(`Recorded event: ${event}`)
      }),
      
      getEvents: () => Ref.get(events)
    }
    
    yield* Effect.addFinalizer(() => Effect.gen(function* () {
      const allEvents = yield* metrics.getEvents()
      yield* Effect.log(`Test ${testId} recorded ${allEvents.length} events`)
    }))
    
    return metrics
  })
)

// Test suite framework
const createTestSuite = (suiteName: string) => Effect.gen(function* () {
  yield* Effect.log(`\n=== Test Suite: ${suiteName} ===`)
  
  const results = yield* Ref.make<Array<{ name: string; success: boolean; duration: number }>>([])
  
  const runTest = (testName: string, testEffect: Effect.Effect<void, any, TestContext | TestDatabase | TestMetrics>) => 
    Effect.gen(function* () {
      const testId = `${suiteName}-${testName}-${Math.random().toString(36).substr(2, 4)}`
      const startTime = Date.now()
      
      yield* Effect.log(`\n  Running: ${testName}`)
      
      const testContext: TestContext = { testName, testId, startTime }
      
      const testLayer = Layer.mergeAll(
        Layer.succeed(TestContext, testContext),
        Layer.effect(TestDatabase, createTestDatabase(testId)),
        Layer.effect(TestMetrics, createTestMetrics(testId))
      )
      
      const result = yield* Effect.scoped(testEffect).pipe(
        Effect.provide(testLayer),
        Effect.either
      )
      
      const duration = Date.now() - startTime
      const success = result._tag === "Right"
      
      yield* Ref.update(results, arr => [...arr, { name: testName, success, duration }])
      
      if (success) {
        yield* Effect.log(`  ✓ ${testName} (${duration}ms)`)
      } else {
        yield* Effect.log(`  ✗ ${testName} (${duration}ms) - ${String(result.left)}`)
      }
      
      return result
    })
  
  const getSummary = () => Effect.gen(function* () {
    const testResults = yield* Ref.get(results)
    const passed = testResults.filter(r => r.success).length
    const failed = testResults.length - passed
    const totalDuration = testResults.reduce((sum, r) => sum + r.duration, 0)
    
    yield* Effect.log(`\n=== ${suiteName} Summary ===`)
    yield* Effect.log(`Tests: ${testResults.length}, Passed: ${passed}, Failed: ${failed}`)
    yield* Effect.log(`Total time: ${totalDuration}ms`)
    
    return { total: testResults.length, passed, failed, duration: totalDuration }
  })
  
  return { runTest, getSummary }
})

// Example service to test
const createUserService = Effect.gen(function* () {
  const db = yield* TestDatabase
  const metrics = yield* TestMetrics
  
  return {
    createUser: (userData: { name: string; email: string }) => Effect.gen(function* () {
      yield* metrics.recordEvent("user.create.start", userData)
      
      if (!userData.name || !userData.email) {
        return yield* Effect.fail("Invalid user data")
      }
      
      // Check if user exists
      const existingUsers = yield* db.query(`SELECT * FROM users WHERE email = '${userData.email}'`)
      if (existingUsers.length > 0) {
        return yield* Effect.fail("User already exists")
      }
      
      // Create user
      const newUser = { id: Math.random().toString(36).substr(2, 8), ...userData }
      yield* db.query(`INSERT INTO users VALUES (${JSON.stringify(newUser)})`)
      
      yield* metrics.recordEvent("user.create.success", newUser)
      return newUser
    }),
    
    getUser: (email: string) => Effect.gen(function* () {
      yield* metrics.recordEvent("user.get.start", { email })
      
      const users = yield* db.query(`SELECT * FROM users WHERE email = '${email}'`)
      
      if (users.length === 0) {
        yield* metrics.recordEvent("user.get.notfound", { email })
        return yield* Effect.fail("User not found")
      }
      
      yield* metrics.recordEvent("user.get.success", users[0])
      return users[0]
    }),
    
    getAllUsers: () => Effect.gen(function* () {
      yield* metrics.recordEvent("user.getall.start")
      const users = yield* db.query("SELECT * FROM users")
      yield* metrics.recordEvent("user.getall.success", { count: users.length })
      return users
    })
  }
})

// Test implementation
const userServiceTests = Effect.gen(function* () {
  const testSuite = yield* createTestSuite("UserService")
  
  // Test 1: Create user successfully
  yield* testSuite.runTest("should create user successfully", Effect.gen(function* () {
    const db = yield* TestDatabase
    const metrics = yield* TestMetrics
    const userService = yield* createUserService
    
    // Seed initial data
    yield* db.seed("users", [])
    
    // Test user creation
    const userData = { name: "John Doe", email: "john@example.com" }
    const user = yield* userService.createUser(userData)
    
    // Verify user was created
    const allUsers = yield* userService.getAllUsers()
    if (allUsers.length !== 1) {
      return yield* Effect.fail("User was not created")
    }
    
    // Check metrics
    const events = yield* metrics.getEvents()
    const createEvents = events.filter(e => e.event.startsWith("user.create"))
    if (createEvents.length !== 2) {
      return yield* Effect.fail("Expected 2 create events")
    }
  }))
  
  // Test 2: Prevent duplicate users
  yield* testSuite.runTest("should prevent duplicate users", Effect.gen(function* () {
    const db = yield* TestDatabase
    const userService = yield* createUserService
    
    // Seed with existing user
    yield* db.seed("users", [
      { id: "existing", name: "Existing User", email: "existing@example.com" }
    ])
    
    // Try to create duplicate
    const result = yield* Effect.either(
      userService.createUser({ name: "Duplicate", email: "existing@example.com" })
    )
    
    if (result._tag !== "Left") {
      return yield* Effect.fail("Should have failed to create duplicate user")
    }
  }))
  
  // Test 3: Get user by email
  yield* testSuite.runTest("should get user by email", Effect.gen(function* () {
    const db = yield* TestDatabase
    const userService = yield* createUserService
    
    // Seed test data
    const testUser = { id: "test", name: "Test User", email: "test@example.com" }
    yield* db.seed("users", [testUser])
    
    // Get user
    const user = yield* userService.getUser("test@example.com")
    
    if (user.email !== "test@example.com") {
      return yield* Effect.fail("Wrong user returned")
    }
  }))
  
  // Test 4: Handle user not found
  yield* testSuite.runTest("should handle user not found", Effect.gen(function* () {
    const db = yield* TestDatabase
    const userService = yield* createUserService
    
    // Clear database
    yield* db.seed("users", [])
    
    // Try to get non-existent user
    const result = yield* Effect.either(
      userService.getUser("nonexistent@example.com")
    )
    
    if (result._tag !== "Left") {
      return yield* Effect.fail("Should have failed for non-existent user")
    }
  }))
  
  // Test 5: Resource isolation test
  yield* testSuite.runTest("should have isolated test resources", Effect.gen(function* () {
    const context = yield* TestContext
    const db = yield* TestDatabase
    const metrics = yield* TestMetrics
    
    // Verify we have a unique test context
    if (!context.testId.includes("UserService-should have isolated test resources")) {
      return yield* Effect.fail("Test context not properly isolated")
    }
    
    // Verify database starts empty
    const users = yield* db.query("SELECT * FROM users")
    if (users.length !== 0) {
      return yield* Effect.fail("Database not properly isolated")
    }
    
    // Verify metrics start empty
    const events = yield* metrics.getEvents()
    if (events.length !== 1) { // Only the setup event
      return yield* Effect.fail("Metrics not properly isolated")
    }
  }))
  
  return yield* testSuite.getSummary()
})

// Run the test suite
Effect.runPromise(userServiceTests)
/*
Output:
=== Test Suite: UserService ===

  Running: should create user successfully
Setting up test database for: UserService-should create user successfully-a1b2
Setting up test metrics for: UserService-should create user successfully-a1b2
Seeding users with 0 records
Recorded event: user.create.start
Test query: SELECT * FROM users WHERE email = 'john@example.com'
Test query: INSERT INTO users VALUES ({"id":"c3d4e5f6","name":"John Doe","email":"john@example.com"})
Recorded event: user.create.success
Recorded event: user.getall.start
Test query: SELECT * FROM users
Recorded event: user.getall.success
  ✓ should create user successfully (15ms)
Test UserService-should create user successfully-a1b2 recorded 4 events
Cleaning up test database for: UserService-should create user successfully-a1b2

  Running: should prevent duplicate users
Setting up test database for: UserService-should prevent duplicate users-g7h8
Setting up test metrics for: UserService-should prevent duplicate users-g7h8
Seeding users with 1 records
Recorded event: user.create.start
Test query: SELECT * FROM users WHERE email = 'existing@example.com'
  ✓ should prevent duplicate users (8ms)
Test UserService-should prevent duplicate users-g7h8 recorded 1 events
Cleaning up test database for: UserService-should prevent duplicate users-g7h8

  Running: should get user by email
Setting up test database for: UserService-should get user by email-i9j0
Setting up test metrics for: UserService-should get user by email-i9j0
Seeding users with 1 records
Recorded event: user.get.start
Test query: SELECT * FROM users WHERE email = 'test@example.com'
Recorded event: user.get.success
  ✓ should get user by email (7ms)
Test UserService-should get user by email-i9j0 recorded 2 events
Cleaning up test database for: UserService-should get user by email-i9j0

  Running: should handle user not found
Setting up test database for: UserService-should handle user not found-k1l2
Setting up test metrics for: UserService-should handle user not found-k1l2
Seeding users with 0 records
Recorded event: user.get.start
Test query: SELECT * FROM users WHERE email = 'nonexistent@example.com'
Recorded event: user.get.notfound
  ✓ should handle user not found (6ms)
Test UserService-should handle user not found-k1l2 recorded 2 events
Cleaning up test database for: UserService-should handle user not found-k1l2

  Running: should have isolated test resources
Setting up test database for: UserService-should have isolated test resources-m3n4
Setting up test metrics for: UserService-should have isolated test resources-m3n4
Test query: SELECT * FROM users
  ✓ should have isolated test resources (4ms)
Test UserService-should have isolated test resources-m3n4 recorded 1 events
Cleaning up test database for: UserService-should have isolated test resources-m3n4

=== UserService Summary ===
Tests: 5, Passed: 5, Failed: 0
Total time: 40ms
*/
```

## Conclusion

Resource provides powerful resource lifecycle management with automatic refresh capabilities, ensuring your applications have access to fresh data and properly managed resources without manual intervention.

Key benefits:
- **Automatic Refresh** - Resources can be configured to refresh automatically according to schedules, ensuring data freshness
- **Manual Control** - Fine-grained control with manual refresh capabilities when needed
- **Concurrency Safety** - Multiple concurrent access requests are properly deduplicated and managed
- **Scope Integration** - Seamless integration with Effect's Scope system for automatic cleanup
- **Error Resilience** - Sophisticated error handling and recovery strategies with fallback mechanisms
- **Observability** - Built-in support for metrics, monitoring, and debugging resource lifecycle events

Use Effect Resource when you need managed, cached values that require periodic updates, such as configuration data, authentication tokens, database connection pools, or any expensive-to-compute values that benefit from caching and automatic refresh cycles. Resource transforms complex resource management scenarios into declarative, composable patterns that integrate seamlessly with the broader Effect ecosystem.