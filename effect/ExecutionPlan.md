# ExecutionPlan: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem ExecutionPlan Solves

When building resilient applications, we often need to provide fallback resources when primary services fail. Traditional approaches force developers to manually implement complex retry logic with different resources:

```typescript
// Traditional approach - problematic code
async function processWithFallback(data: any) {
  try {
    // Try primary service
    return await primaryService.process(data)
  } catch (error) {
    try {
      // Manual fallback to secondary service
      return await secondaryService.process(data)
    } catch (secondError) {
      try {
        // Manual fallback to tertiary service  
        return await tertiaryService.process(data)
      } catch (finalError) {
        throw new Error("All services failed")
      }
    }
  }
}
```

This approach leads to:
- **Nested Error Handling** - Deeply nested try-catch blocks that are hard to maintain
- **Resource Duplication** - Manual management of different service configurations
- **Retry Logic Complexity** - Each fallback needs its own retry strategy implementation
- **Type Safety Issues** - Manual error handling loses type information

### The ExecutionPlan Solution

ExecutionPlan provides a declarative way to define fallback strategies with different resources, retry policies, and scheduling:

```typescript
import { Effect, ExecutionPlan, Layer, Schedule } from "effect"

// Declare services as layers
declare const primaryServiceLayer: Layer.Layer<ProcessingService>
declare const secondaryServiceLayer: Layer.Layer<ProcessingService>
declare const tertiaryServiceLayer: Layer.Layer<ProcessingService>

// Define execution plan with fallback strategy
const processingPlan = ExecutionPlan.make(
  {
    provide: primaryServiceLayer,
    attempts: 3,
    schedule: Schedule.exponential("100 millis")
  },
  {
    provide: secondaryServiceLayer,
    attempts: 2,
    schedule: Schedule.spaced("1 second")
  },
  {
    provide: tertiaryServiceLayer,
    attempts: 1
  }
)

// Apply the plan to any effect that needs ProcessingService
const processWithFallback = (data: any) =>
  Effect.gen(function* () {
    const service = yield* ProcessingService
    return yield* service.process(data)
  }).pipe(
    Effect.withExecutionPlan(processingPlan)
  )
```

### Key Concepts

**Step**: A single execution attempt with a specific resource provider, retry configuration, and scheduling strategy.

**Resource Provider**: Either a `Layer` that provides services or a `Context` that contains pre-built services.

**Fallback Strategy**: The sequence of steps that are tried in order until one succeeds or all are exhausted.

## Basic Usage Patterns

### Pattern 1: Simple Fallback Chain

```typescript
import { Effect, ExecutionPlan, Layer } from "effect"

// Define service layers
declare const primaryDatabase: Layer.Layer<DatabaseService>
declare const backupDatabase: Layer.Layer<DatabaseService>

// Create execution plan
const databasePlan = ExecutionPlan.make(
  { provide: primaryDatabase },
  { provide: backupDatabase }
)

// Apply to any database operation
const saveUser = (user: User) =>
  Effect.gen(function* () {
    const db = yield* DatabaseService
    return yield* db.save(user)
  }).pipe(
    Effect.withExecutionPlan(databasePlan)
  )
```

### Pattern 2: Retry with Scheduling

```typescript
import { Effect, ExecutionPlan, Layer, Schedule } from "effect"

// Plan with retry attempts and scheduling
const apiPlan = ExecutionPlan.make(
  {
    provide: fastApiLayer,
    attempts: 3,
    schedule: Schedule.exponential("50 millis", 2.0)
  },
  {
    provide: slowApiLayer,
    attempts: 2,
    schedule: Schedule.spaced("2 seconds")
  }
)

const fetchData = (query: string) =>
  Effect.gen(function* () {
    const api = yield* ApiService
    return yield* api.fetch(query)
  }).pipe(
    Effect.withExecutionPlan(apiPlan)
  )
```

### Pattern 3: Conditional Execution

```typescript
import { Effect, ExecutionPlan, Layer } from "effect"

// Plan with conditional execution based on error type
const conditionalPlan = ExecutionPlan.make(
  {
    provide: primaryServiceLayer,
    attempts: 2,
    while: (error) => error._tag === "RateLimitError"
  },
  {
    provide: fallbackServiceLayer
  }
)

const processRequest = (request: Request) =>
  Effect.gen(function* () {
    const service = yield* ProcessingService
    return yield* service.handle(request)
  }).pipe(
    Effect.withExecutionPlan(conditionalPlan)
  )
```

## Real-World Examples

### Example 1: Multi-Cloud Storage Fallback

```typescript
import { Effect, ExecutionPlan, Layer, Schedule } from "effect"

// Define storage service interface
interface StorageService {
  readonly upload: (file: File) => Effect.Effect<UploadResult, UploadError>
  readonly download: (key: string) => Effect.Effect<File, DownloadError>
}

const StorageService = Effect.Tag<StorageService>("StorageService")

// Define cloud provider layers
const awsStorageLayer = Layer.effect(
  StorageService,
  Effect.gen(function* () {
    const config = yield* Effect.config(Config.nested("aws"))
    return {
      upload: (file: File) =>
        Effect.gen(function* () {
          // AWS S3 upload implementation
          return { key: `aws-${file.name}`, provider: "aws" as const }
        }),
      download: (key: string) =>
        Effect.gen(function* () {
          // AWS S3 download implementation
          return new File([], key)
        })
    }
  })
)

const gcpStorageLayer = Layer.effect(
  StorageService,
  Effect.gen(function* () {
    const config = yield* Effect.config(Config.nested("gcp"))
    return {
      upload: (file: File) =>
        Effect.gen(function* () {
          // GCP Cloud Storage upload implementation
          return { key: `gcp-${file.name}`, provider: "gcp" as const }
        }),
      download: (key: string) =>
        Effect.gen(function* () {
          // GCP Cloud Storage download implementation
          return new File([], key)
        })
    }
  })
)

const azureStorageLayer = Layer.effect(
  StorageService,
  Effect.gen(function* () {
    const config = yield* Effect.config(Config.nested("azure"))
    return {
      upload: (file: File) =>
        Effect.gen(function* () {
          // Azure Blob Storage upload implementation
          return { key: `azure-${file.name}`, provider: "azure" as const }
        }),
      download: (key: string) =>
        Effect.gen(function* () {
          // Azure Blob Storage download implementation
          return new File([], key)
        })
    }
  })
)

// Create multi-cloud execution plan
const multiCloudPlan = ExecutionPlan.make(
  {
    provide: awsStorageLayer,
    attempts: 2,
    schedule: Schedule.exponential("100 millis"),
    while: (error) => error._tag === "NetworkError" || error._tag === "ThrottleError"
  },
  {
    provide: gcpStorageLayer,
    attempts: 2,
    schedule: Schedule.spaced("500 millis")
  },
  {
    provide: azureStorageLayer,
    attempts: 1
  }
)

// Upload with automatic fallback
export const uploadWithFallback = (file: File) =>
  Effect.gen(function* () {
    const storage = yield* StorageService
    const result = yield* storage.upload(file)
    
    yield* Effect.logInfo(`File uploaded successfully`, {
      key: result.key,
      provider: result.provider
    })
    
    return result
  }).pipe(
    Effect.withExecutionPlan(multiCloudPlan),
    Effect.withSpan("storage.upload", {
      attributes: { "file.name": file.name, "file.size": file.size }
    })
  )
```

### Example 2: AI Model Fallback Strategy

```typescript
import { Effect, ExecutionPlan, Layer, Schedule } from "effect"

// AI service interface
interface AiService {
  readonly generateText: (prompt: string) => Effect.Effect<string, AiError>
  readonly analyzeImage: (image: ImageData) => Effect.Effect<Analysis, AiError>
}

const AiService = Effect.Tag<AiService>("AiService")

// Different AI provider layers
const openAiLayer = Layer.effect(
  AiService,
  Effect.gen(function* () {
    const apiKey = yield* Effect.config(Config.string("OPENAI_API_KEY"))
    return {
      generateText: (prompt: string) =>
        Effect.gen(function* () {
          // OpenAI API call
          return `OpenAI response for: ${prompt}`
        }),
      analyzeImage: (image: ImageData) =>
        Effect.gen(function* () {
          // OpenAI Vision API call
          return { description: "OpenAI image analysis", confidence: 0.95 }
        })
    }
  })
)

const anthropicLayer = Layer.effect(
  AiService,
  Effect.gen(function* () {
    const apiKey = yield* Effect.config(Config.string("ANTHROPIC_API_KEY"))
    return {
      generateText: (prompt: string) =>
        Effect.gen(function* () {
          // Anthropic API call
          return `Anthropic response for: ${prompt}`
        }),
      analyzeImage: (image: ImageData) =>
        Effect.gen(function* () {
          // Anthropic Vision API call
          return { description: "Anthropic image analysis", confidence: 0.92 }
        })
    }
  })
)

const localModelLayer = Layer.effect(
  AiService,
  Effect.gen(function* () {
    return {
      generateText: (prompt: string) =>
        Effect.gen(function* () {
          // Local model inference
          return `Local model response for: ${prompt}`
        }),
      analyzeImage: (image: ImageData) =>
        Effect.gen(function* () {
          // Local vision model
          return { description: "Local model analysis", confidence: 0.78 }
        })
    }
  })
)

// AI execution plan with cost and performance tiers
const aiPlan = ExecutionPlan.make(
  {
    provide: openAiLayer,
    attempts: 2,
    schedule: Schedule.exponential("200 millis"),
    while: (error) => error._tag === "RateLimitError"
  },
  {
    provide: anthropicLayer,
    attempts: 2,
    schedule: Schedule.spaced("1 second"),
    while: (error) => error._tag === "RateLimitError" || error._tag === "NetworkError"
  },
  {
    provide: localModelLayer,
    attempts: 1
  }
)

// Generate text with fallback
export const generateTextWithFallback = (prompt: string) =>
  Effect.gen(function* () {
    const ai = yield* AiService
    const startTime = Date.now()
    
    const response = yield* ai.generateText(prompt)
    const duration = Date.now() - startTime
    
    yield* Effect.logInfo(`Text generated successfully`, {
      prompt: prompt.slice(0, 100),
      duration,
      responseLength: response.length
    })
    
    return response
  }).pipe(
    Effect.withExecutionPlan(aiPlan),
    Effect.withSpan("ai.generateText", {
      attributes: { "prompt.length": prompt.length }
    })
  )
```

### Example 3: Database Connection Pool Fallback

```typescript
import { Effect, ExecutionPlan, Layer, Schedule } from "effect"

// Database service interface
interface DatabaseService {
  readonly query: <T>(sql: string, params?: any[]) => Effect.Effect<T[], DatabaseError>
  readonly transaction: <T>(fn: () => Effect.Effect<T, DatabaseError>) => Effect.Effect<T, DatabaseError>
}

const DatabaseService = Effect.Tag<DatabaseService>("DatabaseService")

// Different database connection layers
const primaryDbLayer = Layer.effect(
  DatabaseService,
  Effect.gen(function* () {
    const config = yield* Effect.config(Config.nested("database.primary"))
    // Primary database connection pool
    return makeDatabaseService(config)
  })
)

const readReplicaLayer = Layer.effect(
  DatabaseService,
  Effect.gen(function* () {
    const config = yield* Effect.config(Config.nested("database.replica"))
    // Read replica connection pool
    return makeDatabaseService(config)
  })
)

const fallbackDbLayer = Layer.effect(
  DatabaseService,
  Effect.gen(function* () {
    const config = yield* Effect.config(Config.nested("database.fallback"))
    // Fallback database connection pool
    return makeDatabaseService(config)
  })
)

// Database execution plan with different strategies for reads vs writes
const readPlan = ExecutionPlan.make(
  {
    provide: primaryDbLayer,
    attempts: 2,
    schedule: Schedule.exponential("50 millis")
  },
  {
    provide: readReplicaLayer,
    attempts: 3,
    schedule: Schedule.spaced("100 millis")
  },
  {
    provide: fallbackDbLayer,
    attempts: 1
  }
)

const writePlan = ExecutionPlan.make(
  {
    provide: primaryDbLayer,
    attempts: 3,
    schedule: Schedule.exponential("100 millis")
  },
  {
    provide: fallbackDbLayer,
    attempts: 2,
    schedule: Schedule.spaced("1 second")
  }
)

// Helper function to create database service
const makeDatabaseService = (config: DatabaseConfig): DatabaseService => ({
  query: <T>(sql: string, params?: any[]) =>
    Effect.gen(function* () {
      // Database query implementation
      return [] as T[]
    }),
  transaction: <T>(fn: () => Effect.Effect<T, DatabaseError>) =>
    Effect.gen(function* () {
      // Transaction implementation
      return yield* fn()
    })
})

// Read operations with fallback
export const findUsers = (filter: UserFilter) =>
  Effect.gen(function* () {
    const db = yield* DatabaseService
    return yield* db.query<User>(
      "SELECT * FROM users WHERE active = ?",
      [filter.active]
    )
  }).pipe(
    Effect.withExecutionPlan(readPlan),
    Effect.withSpan("db.findUsers", {
      attributes: { "filter.active": filter.active }
    })
  )

// Write operations with fallback
export const saveUser = (user: User) =>
  Effect.gen(function* () {
    const db = yield* DatabaseService
    return yield* db.transaction(() =>
      Effect.gen(function* () {
        const result = yield* db.query<{ id: number }>(
          "INSERT INTO users (name, email) VALUES (?, ?) RETURNING id",
          [user.name, user.email]
        )
        return { ...user, id: result[0].id }
      })
    )
  }).pipe(
    Effect.withExecutionPlan(writePlan),
    Effect.withSpan("db.saveUser", {
      attributes: { "user.email": user.email }
    })
  )
```

## Advanced Features Deep Dive

### Feature 1: Conditional Execution with While Clauses

The `while` clause allows you to conditionally execute a step based on the previous error:

#### Basic While Usage

```typescript
import { Effect, ExecutionPlan } from "effect"

const conditionalPlan = ExecutionPlan.make(
  {
    provide: primaryServiceLayer,
    attempts: 3,
    // Only retry on rate limit errors
    while: (error) => error._tag === "RateLimitError"
  },
  {
    provide: fallbackServiceLayer
  }
)
```

#### Real-World While Example

```typescript
import { Effect, ExecutionPlan, Schedule } from "effect"

// Advanced conditional execution plan
const smartRetryPlan = ExecutionPlan.make(
  {
    provide: fastServiceLayer,
    attempts: 5,
    schedule: Schedule.exponential("100 millis"),
    while: (error) => {
      // Only retry on transient errors
      return error._tag === "NetworkError" || 
             error._tag === "TimeoutError" ||
             (error._tag === "HttpError" && error.status >= 500)
    }
  },
  {
    provide: robustServiceLayer,
    attempts: 3,
    schedule: Schedule.spaced("2 seconds"),
    while: (error) => {
      // Different retry conditions for robust service
      return error._tag === "NetworkError" || error._tag === "TimeoutError"
    }
  },
  {
    provide: fallbackServiceLayer
    // No while clause - always try this as final fallback
  }
)

// Effect-based while condition
const effectBasedPlan = ExecutionPlan.make(
  {
    provide: primaryServiceLayer,
    attempts: 3,
    while: (error) =>
      Effect.gen(function* () {
        const metrics = yield* MetricsService
        const errorRate = yield* metrics.getErrorRate("primary-service")
        
        // Only retry if error rate is below threshold
        return errorRate < 0.1 && error._tag === "NetworkError"
      })
  },
  {
    provide: secondaryServiceLayer
  }
)
```

### Feature 2: Plan Merging and Composition

Combine multiple execution plans for complex fallback scenarios:

#### Plan Merging

```typescript
import { Effect, ExecutionPlan } from "effect"

// Define individual plans
const databasePlan = ExecutionPlan.make(
  { provide: primaryDbLayer },
  { provide: secondaryDbLayer }
)

const cachePlan = ExecutionPlan.make(
  { provide: redisLayer },
  { provide: memcachedLayer }
)

// Merge plans for services that need both
const compositePlan = ExecutionPlan.merge(databasePlan, cachePlan)

const complexOperation = Effect.gen(function* () {
  const db = yield* DatabaseService
  const cache = yield* CacheService
  
  // Use both services with their respective fallback strategies
  const cached = yield* cache.get("key")
  if (cached) return cached
  
  const result = yield* db.query("SELECT * FROM data")
  yield* cache.set("key", result)
  return result
}).pipe(
  Effect.withExecutionPlan(compositePlan)
)
```

### Feature 3: Requirements Management

Handle complex dependency requirements across execution steps:

```typescript
import { Effect, ExecutionPlan, Layer } from "effect"

// Plan with varying requirements
const complexPlan = ExecutionPlan.make(
  {
    provide: Layer.effect(
      Service,
      Effect.gen(function* () {
        const logger = yield* Logger
        const metrics = yield* Metrics
        // Service implementation using logger and metrics
        return makeService(logger, metrics)
      })
    )
  },
  {
    provide: Layer.effect(
      Service,
      Effect.gen(function* () {
        const logger = yield* Logger
        // Simpler service implementation with just logger
        return makeSimpleService(logger)
      })
    )
  }
)

// Satisfy requirements before using the plan
const executeWithPlan = Effect.gen(function* () {
  const planWithRequirements = yield* complexPlan.withRequirements
  
  return yield* someEffect.pipe(
    Effect.withExecutionPlan(planWithRequirements)
  )
}).pipe(
  Effect.provide(Layer.merge(Logger.layer, Metrics.layer))
)
```

## Practical Patterns & Best Practices

### Pattern 1: Tiered Service Architecture

```typescript
import { Effect, ExecutionPlan, Schedule } from "effect"

// Helper to create tiered execution plans
const createTieredPlan = <T>(
  primaryLayer: Layer.Layer<T>,
  secondaryLayer: Layer.Layer<T>,
  fallbackLayer: Layer.Layer<T>
) => ExecutionPlan.make(
  {
    provide: primaryLayer,
    attempts: 3,
    schedule: Schedule.exponential("50 millis", 2.0),
    while: (error) => error._tag === "NetworkError" || error._tag === "TimeoutError"
  },
  {
    provide: secondaryLayer,
    attempts: 2,
    schedule: Schedule.spaced("1 second"),
    while: (error) => error._tag === "NetworkError"
  },
  {
    provide: fallbackLayer,
    attempts: 1
  }
)

// Usage across different services
const paymentPlan = createTieredPlan(
  stripeLayer,
  paypalLayer,
  bankTransferLayer
)

const emailPlan = createTieredPlan(
  sendgridLayer,
  sesLayer,
  smtpLayer
)

const storagePlan = createTieredPlan(
  s3Layer,
  gcsLayer,
  localStorageLayer
)
```

### Pattern 2: Environment-Specific Plans

```typescript
import { Effect, ExecutionPlan } from "effect"

// Environment-aware execution plans
const createEnvironmentPlan = (env: "development" | "staging" | "production") => {
  const basePlan = ExecutionPlan.make(
    { provide: primaryServiceLayer },
    { provide: fallbackServiceLayer }
  )
  
  switch (env) {
    case "development":
      return ExecutionPlan.make(
        { provide: mockServiceLayer },
        { provide: primaryServiceLayer }
      )
    
    case "staging":
      return ExecutionPlan.make(
        { provide: primaryServiceLayer, attempts: 2 },
        { provide: fallbackServiceLayer }
      )
    
    case "production":
      return ExecutionPlan.make(
        { provide: primaryServiceLayer, attempts: 5, schedule: Schedule.exponential("100 millis") },
        { provide: secondaryServiceLayer, attempts: 3, schedule: Schedule.spaced("1 second") },
        { provide: fallbackServiceLayer }
      )
    
    default:
      return basePlan
  }
}

// Use environment-specific plan
const processData = (data: any) =>
  Effect.gen(function* () {
    const env = yield* Effect.config(Config.string("NODE_ENV"))
    const plan = createEnvironmentPlan(env as any)
    
    return yield* Effect.gen(function* () {
      const service = yield* DataService
      return yield* service.process(data)
    }).pipe(
      Effect.withExecutionPlan(plan)
    )
  })
```

### Pattern 3: Circuit Breaker Integration

```typescript
import { Effect, ExecutionPlan, Schedule } from "effect"

// Circuit breaker-aware execution plan
const createCircuitBreakerPlan = <T>(
  services: Array<Layer.Layer<T>>,
  circuitBreaker: CircuitBreaker
) => {
  const steps = services.map(service => ({
    provide: service,
    attempts: 3,
    schedule: Schedule.exponential("100 millis"),
    while: (error: any) =>
      Effect.gen(function* () {
        const isOpen = yield* circuitBreaker.isOpen
        // Only retry if circuit breaker is closed and error is retryable
        return !isOpen && (error._tag === "NetworkError" || error._tag === "TimeoutError")
      })
  }))
  
  return ExecutionPlan.make(...steps)
}

// Usage with circuit breaker
const processWithCircuitBreaker = (data: any) =>
  Effect.gen(function* () {
    const circuitBreaker = yield* CircuitBreakerService
    const plan = createCircuitBreakerPlan(
      [primaryServiceLayer, secondaryServiceLayer, fallbackServiceLayer],
      circuitBreaker
    )
    
    return yield* Effect.gen(function* () {
      const service = yield* DataService
      return yield* service.process(data)
    }).pipe(
      Effect.withExecutionPlan(plan),
      Effect.tap(() => circuitBreaker.recordSuccess),
      Effect.tapError(() => circuitBreaker.recordFailure)
    )
  })
```

## Integration Examples

### Integration with Effect Runtime and Metrics

```typescript
import { Effect, ExecutionPlan, Metrics, Runtime } from "effect"

// Metrics-aware execution plan
const createMetricsPlan = <T>(
  name: string,
  layers: Array<Layer.Layer<T>>
) => {
  const counter = Metrics.counter(`${name}_attempts`)
  const histogram = Metrics.histogram(`${name}_duration`, {
    unit: "milliseconds",
    buckets: [10, 50, 100, 500, 1000, 5000]
  })
  
  const steps = layers.map((layer, index) => ({
    provide: layer,
    attempts: 3,
    schedule: Schedule.exponential("100 millis"),
    while: (error: any) =>
      Effect.gen(function* () {
        yield* counter.increment({
          step: index.toString(),
          error_type: error._tag
        })
        return error._tag === "NetworkError" || error._tag === "TimeoutError"
      })
  }))
  
  return ExecutionPlan.make(...steps)
}

// Runtime configuration with execution plan
const createRuntimeWithPlan = () => {
  const plan = createMetricsPlan("api_calls", [
    primaryApiLayer,
    secondaryApiLayer,
    fallbackApiLayer
  ])
  
  return Runtime.make({
    layer: Layer.merge(
      Logger.consoleLayer,
      Metrics.layer,
      plan
    )
  })
}

// Usage with runtime
const processWithRuntime = (data: any) => {
  const runtime = createRuntimeWithPlan()
  
  return Runtime.runPromise(runtime)(
    Effect.gen(function* () {
      const service = yield* ApiService
      return yield* service.process(data)
    }).pipe(
      Effect.withExecutionPlan(createMetricsPlan("api_calls", [
        primaryApiLayer,
        secondaryApiLayer,
        fallbackApiLayer
      ]))
    )
  )
}
```

### Integration with Stream Processing

```typescript
import { Effect, ExecutionPlan, Stream } from "effect"

// Stream processing with execution plan
const processStreamWithFallback = <A, E, R>(
  stream: Stream.Stream<A, E, R>,
  plan: ExecutionPlan<{
    provides: any
    input: E
    error: any
    requirements: any
  }>
) =>
  stream.pipe(
    Stream.withExecutionPlan(plan),
    Stream.catchAll((error) =>
      Stream.make(error).pipe(
        Stream.mapEffect((e) =>
          Effect.logError(`Stream processing failed`, { error: e })
        ),
        Stream.drain
      )
    )
  )

// Real-world stream processing example
const processDataStream = (inputStream: Stream.Stream<RawData, ProcessingError>) => {
  const processingPlan = ExecutionPlan.make(
    { provide: fastProcessorLayer, attempts: 2 },
    { provide: robustProcessorLayer, attempts: 1 }
  )
  
  return inputStream.pipe(
    Stream.mapEffect((data) =>
      Effect.gen(function* () {
        const processor = yield* DataProcessor
        return yield* processor.transform(data)
      })
    ),
    Stream.withExecutionPlan(processingPlan),
    Stream.buffer(1000),
    Stream.grouped(100),
    Stream.mapEffect((batch) =>
      Effect.gen(function* () {
        const storage = yield* StorageService
        return yield* storage.saveBatch(batch)
      })
    )
  )
}
```

### Testing Strategies

```typescript
import { Effect, ExecutionPlan, Layer, TestContext } from "effect"

// Test utilities for execution plans
const createTestPlan = <T>(
  mockService: T,
  shouldFail: boolean = false
) => {
  const testLayer = Layer.succeed(Service.Tag, mockService)
  
  if (shouldFail) {
    const failingLayer = Layer.effect(
      Service.Tag,
      Effect.fail(new Error("Test failure"))
    )
    
    return ExecutionPlan.make(
      { provide: failingLayer },
      { provide: testLayer }
    )
  }
  
  return ExecutionPlan.make({ provide: testLayer })
}

// Test execution plan behavior
describe("ExecutionPlan", () => {
  it("should fallback to secondary service on primary failure", () =>
    Effect.gen(function* () {
      const mockPrimary = { process: () => Effect.fail(new Error("Primary failed")) }
      const mockSecondary = { process: () => Effect.succeed("Secondary success") }
      
      const plan = ExecutionPlan.make(
        { provide: Layer.succeed(Service.Tag, mockPrimary) },
        { provide: Layer.succeed(Service.Tag, mockSecondary) }
      )
      
      const result = yield* Effect.gen(function* () {
        const service = yield* Service.Tag
        return yield* service.process()
      }).pipe(
        Effect.withExecutionPlan(plan)
      )
      
      expect(result).toBe("Secondary success")
    }).pipe(
      Effect.provide(TestContext.TestContext)
    )
  )
  
  it("should respect retry attempts", () =>
    Effect.gen(function* () {
      let attempts = 0
      const mockService = {
        process: () =>
          Effect.gen(function* () {
            attempts++
            if (attempts < 3) {
              return yield* Effect.fail(new Error(`Attempt ${attempts} failed`))
            }
            return `Success after ${attempts} attempts`
          })
      }
      
      const plan = ExecutionPlan.make({
        provide: Layer.succeed(Service.Tag, mockService),
        attempts: 3,
        schedule: Schedule.exponential("10 millis")
      })
      
      const result = yield* Effect.gen(function* () {
        const service = yield* Service.Tag
        return yield* service.process()
      }).pipe(
        Effect.withExecutionPlan(plan)
      )
      
      expect(result).toBe("Success after 3 attempts")
      expect(attempts).toBe(3)
    }).pipe(
      Effect.provide(TestContext.TestContext)
    )
  )
})
```

## Conclusion

ExecutionPlan provides **declarative fallback strategies**, **resource management**, and **resilient error handling** for Effect applications.

Key benefits:
- **Declarative Configuration**: Define complex fallback strategies without nested error handling
- **Resource Management**: Automatically manage different service layers and contexts
- **Type Safety**: Maintain full type safety across all fallback steps and error scenarios
- **Composability**: Combine and merge execution plans for complex architectural patterns
- **Observability**: Integrate seamlessly with Effect's tracing, metrics, and logging systems

ExecutionPlan is ideal for building resilient systems that need to gracefully handle service failures, implement multi-tier architectures, or provide fallback strategies for critical operations.