# PlatformLogger: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem PlatformLogger Solves

Traditional logging approaches often fall short when building production applications:

```typescript
// Traditional approach - scattered console.log statements
function processOrder(order: Order) {
  console.log(`Processing order ${order.id}`)
  try {
    const validated = validateOrder(order)
    console.log(`Order validated: ${JSON.stringify(validated)}`)
    
    const payment = processPayment(validated)
    console.log(`Payment processed: ${payment.id}`)
    
    return { success: true, orderId: order.id }
  } catch (error) {
    console.error(`Order processing failed: ${error.message}`)
    throw error
  }
}
```

This approach leads to:
- **No Centralized Control** - Logs scattered throughout codebase, no unified configuration
- **Poor Production Logging** - Console output isn't suitable for production log aggregation
- **Limited Structured Data** - Difficult to parse and analyze logs programmatically
- **No Log Rotation** - Files grow indefinitely without management
- **Environment Coupling** - Same logging behavior across dev/staging/production

### The PlatformLogger Solution

PlatformLogger provides file-based logging with proper structure, batching, and production-ready features:

```typescript
import { PlatformLogger } from "@effect/platform"
import { NodeFileSystem } from "@effect/platform-node"
import { Effect, Layer, Logger } from "effect"

const fileLogger = Logger.structuredLogger.pipe(
  PlatformLogger.toFile("/var/log/app/application.log")
)

const LoggerLive = Logger.replaceScoped(
  Logger.defaultLogger,
  fileLogger
).pipe(Layer.provide(NodeFileSystem.layer))

const processOrder = (order: Order) => Effect.gen(function* () {
  yield* Effect.log("Processing order", { orderId: order.id, userId: order.userId })
  const validated = yield* validateOrder(order)
  yield* Effect.log("Order validated", { orderId: order.id, amount: validated.total })
  const payment = yield* processPayment(validated)
  yield* Effect.log("Payment processed", { orderId: order.id, paymentId: payment.id })
  return { success: true, orderId: order.id }
}).pipe(
  Effect.catchAll(error => Effect.gen(function* () {
    yield* Effect.logError("Order processing failed", { orderId: order.id, error: error.message })
    return yield* Effect.fail(error)
  })),
  Effect.withLogSpan("order.process"),
  Effect.annotateLogs({ service: "order-service", version: "1.2.0" })
)
```

### Key Concepts

**File-Based Logging**: Write logs directly to files using FileSystem APIs, suitable for production environments with log rotation and archival.

**Structured Output**: Combine with structured loggers (JSON, logfmt) for machine-readable logs that integrate with monitoring systems.

**Batching**: Group multiple log entries together before writing to reduce I/O overhead in high-throughput applications.

## Basic Usage Patterns

### Pattern 1: Simple File Logging

```typescript
import { PlatformLogger } from "@effect/platform"
import { NodeFileSystem } from "@effect/platform-node"
import { Effect, Layer, Logger } from "effect"

// Create a file logger using the default string format
const fileLogger = Logger.stringLogger.pipe(
  PlatformLogger.toFile("/tmp/app.log")
)

// Replace the default logger
const LoggerLive = Logger.replaceScoped(
  Logger.defaultLogger,
  fileLogger
).pipe(Layer.provide(NodeFileSystem.layer))

const program = Effect.gen(function* () {
  yield* Effect.log("Application started")
  yield* Effect.log("Processing user request")
  yield* Effect.log("Application finished")
})

Effect.runFork(program.pipe(Effect.provide(LoggerLive)))
```

### Pattern 2: Structured JSON Logging

```typescript
import { PlatformLogger } from "@effect/platform"
import { NodeFileSystem } from "@effect/platform-node"
import { Effect, Layer, Logger } from "effect"

// Create a JSON logger for structured logging
const jsonFileLogger = Logger.jsonLogger.pipe(
  PlatformLogger.toFile("/var/log/app/app.json")
)

const LoggerLive = Logger.replaceScoped(
  Logger.defaultLogger,
  jsonFileLogger
).pipe(Layer.provide(NodeFileSystem.layer))

const program = Effect.gen(function* () {
  yield* Effect.log("User action", { userId: "123", action: "login" })
  yield* Effect.log("Database query", { table: "users", duration: "45ms" })
}).pipe(
  Effect.annotateLogs({ service: "auth-service", environment: "production" })
)
```

### Pattern 3: Dual Console and File Logging

```typescript
import { PlatformLogger } from "@effect/platform"
import { NodeFileSystem } from "@effect/platform-node"
import { Effect, Layer, Logger } from "effect"

// Create file logger
const fileLogger = Logger.jsonLogger.pipe(
  PlatformLogger.toFile("/var/log/app/app.json")
)

// Combine console (pretty format) and file (JSON format) logging
const dualLogger = Effect.map(fileLogger, (fileLogger) =>
  Logger.zip(Logger.prettyLoggerDefault, fileLogger)
)

const LoggerLive = Logger.replaceScoped(
  Logger.defaultLogger,
  dualLogger
).pipe(Layer.provide(NodeFileSystem.layer))

const program = Effect.gen(function* () {
  yield* Effect.log("Server starting on port 3000")
  yield* Effect.log("Database connected")
  yield* Effect.logError("Failed to load configuration")
})
```

## Real-World Examples

### Example 1: HTTP Server Request Logging

```typescript
import { PlatformLogger } from "@effect/platform"
import { NodeFileSystem } from "@effect/platform-node"
import { Effect, Layer, Logger, Duration } from "effect"

// Create request logger with batching for performance
const requestLogger = Logger.structuredLogger.pipe(
  Logger.batched(Duration.seconds(5), (messages) => 
    Effect.gen(function* () {
      const fs = yield* FileSystem.FileSystem
      const logEntries = messages.join('\n') + '\n'
      yield* fs.writeFileString("/var/log/app/requests.log", logEntries, { 
        flag: "a" // append mode
      })
    })
  )
)

const RequestLoggerLive = Logger.replaceScoped(
  Logger.defaultLogger,
  requestLogger
).pipe(Layer.provide(NodeFileSystem.layer))

// HTTP middleware for request logging
const logRequest = (request: HttpRequest) => Effect.gen(function* () {
  const startTime = Date.now()
  
  yield* Effect.log("Request started", {
    method: request.method,
    url: request.url,
    userAgent: request.headers["user-agent"],
    ip: request.remoteAddress,
    timestamp: new Date().toISOString()
  })
  
  const response = yield* processRequest(request)
  const duration = Date.now() - startTime
  
  yield* Effect.log("Request completed", {
    method: request.method,
    url: request.url,
    status: response.status,
    duration: `${duration}ms`,
    responseSize: response.headers["content-length"]
  })
  
  return response
}).pipe(
  Effect.catchAll(error => Effect.gen(function* () {
    const duration = Date.now() - startTime
    yield* Effect.logError("Request failed", {
      method: request.method,
      url: request.url,
      error: error.message,
      duration: `${duration}ms`
    })
    return yield* Effect.fail(error)
  })),
  Effect.withLogSpan("http.request"),
  Effect.annotateLogs({ 
    service: "api-server",
    version: process.env.APP_VERSION || "unknown"
  })
)

const server = Effect.gen(function* () {
  yield* Effect.log("Server initializing")
  
  // Start HTTP server with request logging
  const httpServer = yield* createHttpServer({
    port: 3000,
    middleware: [logRequest]
  })
  
  yield* Effect.log("Server started", { port: 3000 })
  return httpServer
}).pipe(Effect.provide(RequestLoggerLive))
```

### Example 2: Background Job Processing with Error Tracking

```typescript
import { PlatformLogger } from "@effect/platform"
import { NodeFileSystem } from "@effect/platform-node"
import { Effect, Layer, Logger, Schedule, Duration } from "effect"

// Separate loggers for different concerns
const jobLogger = Logger.jsonLogger.pipe(
  PlatformLogger.toFile("/var/log/app/jobs.log")
)

const errorLogger = Logger.structuredLogger.pipe(
  PlatformLogger.toFile("/var/log/app/errors.log")
)

const LoggerLive = Logger.replaceScoped(
  Logger.defaultLogger,
  Effect.map(
    Effect.all([jobLogger, errorLogger]),
    ([jobLogger, errorLogger]) => Logger.zip(jobLogger, errorLogger)
  )
).pipe(Layer.provide(NodeFileSystem.layer))

interface Job {
  id: string
  type: string
  payload: unknown
  attempts: number
  maxAttempts: number
}

const processJob = (job: Job) => Effect.gen(function* () {
  yield* Effect.log("Job started", {
    jobId: job.id,
    jobType: job.type,
    attempt: job.attempts + 1,
    maxAttempts: job.maxAttempts
  })
  
  const result = yield* executeJobLogic(job)
  
  yield* Effect.log("Job completed", {
    jobId: job.id,
    jobType: job.type,
    result: result.summary
  })
  
  return result
}).pipe(
  Effect.retry(
    Schedule.exponential(Duration.seconds(1)).pipe(
      Schedule.intersect(Schedule.recurs(job.maxAttempts - 1))
    )
  ),
  Effect.catchAll(error => Effect.gen(function* () {
    yield* Effect.logError("Job failed permanently", {
      jobId: job.id,
      jobType: job.type,
      attempts: job.maxAttempts,
      error: error.message,
      stack: error.stack
    })
    
    // Store failed job for manual review
    yield* saveFailedJob(job, error)
    
    return yield* Effect.fail(error)
  })),
  Effect.withLogSpan("job.process"),
  Effect.annotateLogs({ 
    service: "job-processor",
    worker: process.env.WORKER_ID || "unknown"
  })
)

const jobProcessor = Effect.gen(function* () {
  yield* Effect.log("Job processor starting")
  
  yield* Effect.forever(
    Effect.gen(function* () {
      const jobs = yield* getNextJobs(10)
      
      if (jobs.length === 0) {
        yield* Effect.sleep(Duration.seconds(5))
        return
      }
      
      yield* Effect.log("Processing job batch", { count: jobs.length })
      
      yield* Effect.forEach(jobs, processJob, { 
        concurrency: 3,
        discard: true
      })
    })
  )
}).pipe(Effect.provide(LoggerLive))
```

### Example 3: Multi-Environment Application Logging

```typescript
import { PlatformLogger } from "@effect/platform"
import { NodeFileSystem } from "@effect/platform-node"
import { Effect, Layer, Logger, LogLevel, Config } from "effect"

// Environment-specific logger configuration
const createEnvironmentLogger = (env: string) => Effect.gen(function* () {
  const logDir = `/var/log/app/${env}`
  
  // Ensure log directory exists
  const fs = yield* FileSystem.FileSystem
  yield* fs.makeDirectory(logDir, { recursive: true })
  
  switch (env) {
    case "development":
      // Pretty console + debug file logging
      const devFileLogger = Logger.structuredLogger.pipe(
        PlatformLogger.toFile(`${logDir}/debug.log`)
      )
      return Effect.map(devFileLogger, (fileLogger) =>
        Logger.zip(Logger.prettyLoggerDefault, fileLogger)
      )
      
    case "staging":
      // JSON file logging with info level
      return Logger.jsonLogger.pipe(
        PlatformLogger.toFile(`${logDir}/app.log`),
        Effect.map(logger => logger.pipe(
          Logger.filterLogLevel(LogLevel.Info)
        ))
      )
      
    case "production":
      // Batched JSON logging with error alerts
      const prodLogger = Logger.jsonLogger.pipe(
        Logger.batched(Duration.seconds(10), (messages) =>
          Effect.gen(function* () {
            const fs = yield* FileSystem.FileSystem
            const logContent = messages.join('\n') + '\n'
            yield* fs.writeFileString(`${logDir}/app.log`, logContent, { 
              flag: "a" 
            })
            
            // Check for errors and send alerts
            const errors = messages.filter(msg => 
              msg.includes('"logLevel":"ERROR"') || 
              msg.includes('"logLevel":"FATAL"')
            )
            
            if (errors.length > 0) {
              yield* sendErrorAlert(errors)
            }
          })
        )
      )
      
      return prodLogger.pipe(
        Effect.map(logger => logger.pipe(
          Logger.filterLogLevel(LogLevel.Warning)
        ))
      )
      
    default:
      return Logger.jsonLogger.pipe(
        PlatformLogger.toFile(`${logDir}/app.log`)
      )
  }
})

// Application logger layer
const AppLoggerLive = Layer.unwrapEffect(
  Effect.gen(function* () {
    const environment = yield* Config.string("NODE_ENV").pipe(
      Config.withDefault("development")
    )
    
    const environmentLogger = yield* createEnvironmentLogger(environment)
    
    return Logger.replaceScoped(
      Logger.defaultLogger,
      environmentLogger
    ).pipe(Layer.provide(NodeFileSystem.layer))
  })
)

// Usage in application
const userService = {
  createUser: (userData: UserData) => Effect.gen(function* () {
    yield* Effect.log("Creating user", { 
      email: userData.email,
      role: userData.role 
    })
    
    const user = yield* saveUser(userData)
    
    yield* Effect.log("User created successfully", {
      userId: user.id,
      email: user.email
    })
    
    return user
  }).pipe(
    Effect.catchAll(error => Effect.gen(function* () {
      yield* Effect.logError("User creation failed", {
        email: userData.email,
        error: error.message
      })
      return yield* Effect.fail(error)
    })),
    Effect.withLogSpan("user.create"),
    Effect.annotateLogs({ service: "user-service" })
  )
}

const app = Effect.gen(function* () {
  yield* Effect.log("Application starting")
  
  const server = yield* startServer()
  const jobProcessor = yield* startJobProcessor()
  
  yield* Effect.log("Application ready", {
    server: { port: server.port },
    jobProcessor: { workerId: jobProcessor.id }
  })
  
  yield* Effect.never
}).pipe(Effect.provide(AppLoggerLive))
```

## Advanced Features Deep Dive

### Feature 1: Log Batching for Performance

Batching reduces I/O operations by collecting multiple log entries before writing:

#### Basic Batching Usage

```typescript
import { PlatformLogger } from "@effect/platform"
import { NodeFileSystem } from "@effect/platform-node"
import { Effect, Layer, Logger, Duration } from "effect"

const batchedLogger = Logger.jsonLogger.pipe(
  Logger.batched(
    Duration.seconds(5), // Batch window
    (messages) => Effect.gen(function* () {
      const fs = yield* FileSystem.FileSystem
      const logContent = messages.join('\n') + '\n'
      yield* fs.writeFileString("/var/log/app/batched.log", logContent, { 
        flag: "a" 
      })
      console.log(`Wrote ${messages.length} log entries`)
    })
  )
)
```

#### Real-World Batching Example

```typescript
// High-throughput logging with intelligent batching
const createHighThroughputLogger = (logPath: string) => Effect.gen(function* () {
  const fs = yield* FileSystem.FileSystem
  
  // Ensure log directory exists
  const logDir = logPath.substring(0, logPath.lastIndexOf('/'))
  yield* fs.makeDirectory(logDir, { recursive: true })
  
  return Logger.structuredLogger.pipe(
    Logger.batched(
      Duration.seconds(2), // Frequent batching for responsiveness
      (messages) => Effect.gen(function* () {
        const timestamp = new Date().toISOString()
        const batchHeader = `=== BATCH START ${timestamp} (${messages.length} entries) ===\n`
        const batchFooter = `=== BATCH END ${timestamp} ===\n\n`
        const content = batchHeader + messages.join('\n') + '\n' + batchFooter
        
        yield* fs.writeFileString(logPath, content, { flag: "a" })
        
        // Rotate log file if it gets too large
        const stats = yield* fs.stat(logPath)
        if (stats.size > 100 * 1024 * 1024) { // 100MB
          yield* rotateLogFile(logPath)
        }
      }).pipe(
        Effect.catchAll(error => Effect.gen(function* () {
          // Fallback to console if file writing fails
          console.error(`Failed to write batch to ${logPath}:`, error)
          messages.forEach(msg => console.log(msg))
        }))
      )
    )
  )
})
```

### Feature 2: Log Rotation and Archival

```typescript
// Log rotation helper
const rotateLogFile = (logPath: string) => Effect.gen(function* () {
  const fs = yield* FileSystem.FileSystem
  const timestamp = new Date().toISOString().replace(/[:.]/g, '-')
  const rotatedPath = `${logPath}.${timestamp}`
  
  yield* Effect.log("Rotating log file", { 
    from: logPath, 
    to: rotatedPath 
  })
  
  // Move current log to rotated name
  yield* fs.rename(logPath, rotatedPath)
  
  // Compress rotated log
  yield* compressLogFile(rotatedPath)
  
  // Clean up old rotated logs (keep last 10)
  yield* cleanupOldLogs(logPath)
})

const compressLogFile = (logPath: string) => Effect.gen(function* () {
  const compressedPath = `${logPath}.gz`
  // Implementation would use compression library
  yield* Effect.log("Log file compressed", { 
    original: logPath,
    compressed: compressedPath 
  })
})

const cleanupOldLogs = (baseLogPath: string) => Effect.gen(function* () {
  const fs = yield* FileSystem.FileSystem
  const logDir = baseLogPath.substring(0, baseLogPath.lastIndexOf('/'))
  const baseName = baseLogPath.substring(baseLogPath.lastIndexOf('/') + 1)
  
  const files = yield* fs.readDirectory(logDir)
  const rotatedLogs = files
    .filter(file => file.startsWith(`${baseName}.`) && file.endsWith('.gz'))
    .sort()
    .reverse()
  
  // Keep only the 10 most recent rotated logs
  const logsToDelete = rotatedLogs.slice(10)
  
  yield* Effect.forEach(logsToDelete, (file) => 
    fs.remove(`${logDir}/${file}`).pipe(
      Effect.tap(() => Effect.log("Deleted old log file", { file }))
    ),
    { discard: true }
  )
})
```

### Feature 3: Custom Log Processors

```typescript
// Custom log processor with filtering and enrichment
const createEnhancedLogger = (logPath: string) => Effect.gen(function* () {
  const fs = yield* FileSystem.FileSystem
  
  return Logger.make(({ logLevel, message, annotations, spans }) => 
    Effect.gen(function* () {
      // Enrich log entry with additional context
      const enrichedEntry = {
        timestamp: new Date().toISOString(),
        level: logLevel.label,
        message: Array.isArray(message) ? message.join(' ') : message,
        annotations: {
          ...annotations,
          hostname: process.env.HOSTNAME || 'unknown',
          pid: process.pid,
          memory: process.memoryUsage(),
          uptime: process.uptime()
        },
        spans,
        environment: process.env.NODE_ENV || 'development'
      }
      
      // Filter sensitive data
      const sanitizedEntry = sanitizeLogEntry(enrichedEntry)
      
      // Format for output
      const logLine = JSON.stringify(sanitizedEntry) + '\n'
      
      yield* fs.writeFileString(logPath, logLine, { flag: "a" })
    }).pipe(
      Effect.catchAll(error => 
        Effect.sync(() => console.error('Log write failed:', error))
      )
    )
  )
})

const sanitizeLogEntry = (entry: any): any => {
  const sensitiveFields = ['password', 'token', 'apiKey', 'secret']
  
  const sanitize = (obj: any): any => {
    if (typeof obj !== 'object' || obj === null) return obj
    
    const sanitized: any = {}
    for (const [key, value] of Object.entries(obj)) {
      if (sensitiveFields.some(field => key.toLowerCase().includes(field))) {
        sanitized[key] = '[REDACTED]'
      } else if (typeof value === 'object') {
        sanitized[key] = sanitize(value)
      } else {
        sanitized[key] = value
      }
    }
    return sanitized
  }
  
  return sanitize(entry)
}
```

## Practical Patterns & Best Practices

### Pattern 1: Centralized Logging Configuration

```typescript
// Centralized logging configuration
const LoggingConfig = Config.all({
  level: Config.logLevel("LOG_LEVEL").pipe(Config.withDefault(LogLevel.Info)),
  directory: Config.string("LOG_DIR").pipe(Config.withDefault("/var/log/app")),
  maxFileSize: Config.number("LOG_MAX_FILE_SIZE").pipe(Config.withDefault(100 * 1024 * 1024)),
  batchWindow: Config.duration("LOG_BATCH_WINDOW").pipe(Config.withDefault(Duration.seconds(5))),
  enableConsole: Config.boolean("LOG_ENABLE_CONSOLE").pipe(Config.withDefault(false)),
  enableFileRotation: Config.boolean("LOG_ENABLE_ROTATION").pipe(Config.withDefault(true))
})

const createConfiguredLogger = Effect.gen(function* () {
  const config = yield* LoggingConfig
  const fs = yield* FileSystem.FileSystem
  
  // Ensure log directory exists
  yield* fs.makeDirectory(config.directory, { recursive: true })
  
  const logPath = `${config.directory}/app.log`
  
  // Create base logger
  let logger = Logger.jsonLogger.pipe(
    Logger.filterLogLevel(config.level)
  )
  
  // Add file output
  const fileLogger = logger.pipe(
    PlatformLogger.toFile(logPath)
  )
  
  // Add console output if enabled
  const finalLogger = config.enableConsole
    ? Effect.map(fileLogger, (fLogger) => 
        Logger.zip(Logger.prettyLoggerDefault, fLogger)
      )
    : fileLogger
  
  // Add batching
  const batchedLogger = Effect.map(finalLogger, (logger) =>
    logger.pipe(
      Logger.batched(config.batchWindow, (messages) =>
        writeLogBatch(logPath, messages, config)
      )
    )
  )
  
  return batchedLogger
})

const writeLogBatch = (
  logPath: string, 
  messages: string[], 
  config: any
) => Effect.gen(function* () {
  const fs = yield* FileSystem.FileSystem
  const content = messages.join('\n') + '\n'
  
  yield* fs.writeFileString(logPath, content, { flag: "a" })
  
  // Check for rotation if enabled
  if (config.enableFileRotation) {
    const stats = yield* fs.stat(logPath)
    if (stats.size > config.maxFileSize) {
      yield* rotateLogFile(logPath)
    }
  }
})
```

### Pattern 2: Service-Specific Logging

```typescript
// Service-specific logger factory
const createServiceLogger = (serviceName: string, version: string) => 
  Effect.gen(function* () {
    const baseLogger = yield* createConfiguredLogger
    
    return Logger.replaceScoped(
      Logger.defaultLogger,
      Effect.map(baseLogger, (logger) =>
        logger.pipe(
          Logger.mapInput(({ logLevel, message, annotations, spans }) => ({
            logLevel,
            message,
            annotations: {
              ...annotations,
              service: serviceName,
              version,
              timestamp: new Date().toISOString()
            },
            spans
          }))
        )
      )
    )
  })

// Usage in different services
const UserServiceLogger = createServiceLogger("user-service", "1.0.0")
const OrderServiceLogger = createServiceLogger("order-service", "2.1.0")
const PaymentServiceLogger = createServiceLogger("payment-service", "1.5.0")

// Service implementations with their specific loggers
const userService = {
  getUser: (id: string) => Effect.gen(function* () {
    yield* Effect.log("Fetching user", { userId: id })
    const user = yield* fetchUserFromDatabase(id)
    yield* Effect.log("User fetched successfully", { 
      userId: id, 
      email: user.email 
    })
    return user
  }).pipe(
    Effect.withLogSpan("user.get"),
    Effect.provide(UserServiceLogger)
  )
}
```

### Pattern 3: Error Aggregation and Alerting

```typescript
// Error tracking and alerting system
const createErrorTrackingLogger = Effect.gen(function* () {
  const baseLogger = yield* createConfiguredLogger
  const errorCounts = new Map<string, number>()
  
  return Effect.map(baseLogger, (logger) =>
    Logger.make(({ logLevel, message, annotations }) =>
      Effect.gen(function* () {
        // Log normally
        yield* logger.log({ logLevel, message, annotations })
        
        // Track errors for alerting
        if (logLevel._tag === "Error" || logLevel._tag === "Fatal") {
          const errorKey = `${annotations.service || 'unknown'}:${message}`
          const currentCount = errorCounts.get(errorKey) || 0
          errorCounts.set(errorKey, currentCount + 1)
          
          // Send alert if error threshold reached
          if (currentCount + 1 >= 5) { // 5 errors in window
            yield* sendErrorAlert({
              service: annotations.service,
              error: message,
              count: currentCount + 1,
              timestamp: new Date().toISOString()
            })
            errorCounts.delete(errorKey) // Reset counter
          }
        }
      })
    )
  )
})

const sendErrorAlert = (errorInfo: {
  service: string
  error: string
  count: number
  timestamp: string
}) => Effect.gen(function* () {
  yield* Effect.log("Sending error alert", errorInfo)
  
  // Integration with alerting system (Slack, PagerDuty, etc.)
  const alertPayload = {
    title: `High Error Rate: ${errorInfo.service}`,
    message: `Error "${errorInfo.error}" occurred ${errorInfo.count} times`,
    timestamp: errorInfo.timestamp,
    severity: "high"
  }
  
  yield* sendSlackAlert(alertPayload)
  yield* sendPagerDutyAlert(alertPayload)
})
```

## Integration Examples

### Integration with Express.js Server

```typescript
import express from "express"
import { PlatformLogger } from "@effect/platform"
import { NodeFileSystem } from "@effect/platform-node"
import { Effect, Layer, Logger, Runtime } from "effect"

// Express middleware for Effect logging
const createLoggingMiddleware = (runtime: Runtime.Runtime<never>) => {
  return (req: express.Request, res: express.Response, next: express.NextFunction) => {
    const startTime = Date.now()
    
    const logRequest = Effect.gen(function* () {
      yield* Effect.log("HTTP Request", {
        method: req.method,
        url: req.url,
        userAgent: req.get('User-Agent'),
        ip: req.ip,
        timestamp: new Date().toISOString()
      })
    }).pipe(
      Effect.withLogSpan("http.request.start"),
      Effect.annotateLogs({ 
        requestId: req.get('X-Request-ID') || 'unknown'
      })
    )
    
    Runtime.runFork(runtime)(logRequest)
    
    // Log response
    res.on('finish', () => {
      const duration = Date.now() - startTime
      
      const logResponse = Effect.gen(function* () {
        yield* Effect.log("HTTP Response", {
          method: req.method,
          url: req.url,
          status: res.statusCode,
          duration: `${duration}ms`,
          contentLength: res.get('Content-Length') || '0'
        })
      }).pipe(
        Effect.withLogSpan("http.request.complete"),
        Effect.annotateLogs({ 
          requestId: req.get('X-Request-ID') || 'unknown'
        })
      )
      
      Runtime.runFork(runtime)(logResponse)
    })
    
    next()
  }
}

// Express server setup
const setupExpressServer = Effect.gen(function* () {
  const app = express()
  
  // Create logger
  const fileLogger = Logger.jsonLogger.pipe(
    PlatformLogger.toFile("/var/log/app/express.log")
  )
  
  const LoggerLive = Logger.replaceScoped(
    Logger.defaultLogger,
    fileLogger
  ).pipe(Layer.provide(NodeFileSystem.layer))
  
  const runtime = yield* Effect.runtime<never>().pipe(
    Effect.provide(LoggerLive)
  )
  
  // Add logging middleware
  app.use(createLoggingMiddleware(runtime))
  
  // Routes
  app.get('/users/:id', (req, res) => {
    const getUserEffect = Effect.gen(function* () {
      const userId = req.params.id
      yield* Effect.log("Fetching user", { userId })
      
      const user = yield* fetchUser(userId)
      
      yield* Effect.log("User fetched", { userId, email: user.email })
      res.json(user)
    }).pipe(
      Effect.catchAll(error => Effect.gen(function* () {
        yield* Effect.logError("Failed to fetch user", {
          userId: req.params.id,
          error: error.message
        })
        res.status(500).json({ error: "Internal server error" })
      }))
    )
    
    Runtime.runFork(runtime)(getUserEffect)
  })
  
  const server = app.listen(3000, () => {
    Runtime.runFork(runtime)(
      Effect.log("Express server started", { port: 3000 })
    )
  })
  
  return server
})
```

### Integration with Next.js API Routes

```typescript
// Next.js API route with Effect logging
import { NextApiRequest, NextApiResponse } from 'next'
import { PlatformLogger } from "@effect/platform"
import { NodeFileSystem } from "@effect/platform-node"
import { Effect, Layer, Logger, Runtime } from "effect"

// Create logger runtime
const createLoggerRuntime = () => {
  const fileLogger = Logger.jsonLogger.pipe(
    PlatformLogger.toFile("/var/log/nextjs/api.log")
  )
  
  const LoggerLive = Logger.replaceScoped(
    Logger.defaultLogger,
    fileLogger
  ).pipe(Layer.provide(NodeFileSystem.layer))
  
  return Runtime.make(LoggerLive)
}

const loggerRuntime = createLoggerRuntime()

// API route handler with logging
export default function handler(req: NextApiRequest, res: NextApiResponse) {
  const apiHandler = Effect.gen(function* () {
    yield* Effect.log("API Request", {
      method: req.method,
      url: req.url,
      userAgent: req.headers['user-agent'],
      timestamp: new Date().toISOString()
    })
    
    switch (req.method) {
      case 'GET':
        return yield* handleGetRequest(req, res)
      case 'POST':
        return yield* handlePostRequest(req, res)
      default:
        yield* Effect.logWarning("Method not allowed", { method: req.method })
        res.status(405).json({ error: 'Method not allowed' })
    }
  }).pipe(
    Effect.catchAll(error => Effect.gen(function* () {
      yield* Effect.logError("API Error", {
        method: req.method,
        url: req.url,
        error: error.message,
        stack: error.stack
      })
      res.status(500).json({ error: 'Internal server error' })
    })),
    Effect.withLogSpan("api.request"),
    Effect.annotateLogs({
      service: "nextjs-api",
      route: req.url || 'unknown'
    })
  )
  
  Runtime.runPromise(loggerRuntime)(apiHandler)
}

const handleGetRequest = (req: NextApiRequest, res: NextApiResponse) => 
  Effect.gen(function* () {
    const data = yield* fetchData()
    yield* Effect.log("Data fetched successfully", { count: data.length })
    res.json(data)
  })

const handlePostRequest = (req: NextApiRequest, res: NextApiResponse) => 
  Effect.gen(function* () {
    yield* Effect.log("Processing POST request", { bodySize: JSON.stringify(req.body).length })
    const result = yield* processData(req.body)
    yield* Effect.log("POST request processed", { resultId: result.id })
    res.json(result)
  })
```

### Testing Strategies

```typescript
// Testing with mock file system
import { PlatformLogger } from "@effect/platform"
import { FileSystem } from "@effect/platform"
import { Effect, Layer, Logger } from "effect"
import { describe, it, expect } from "vitest"

describe("PlatformLogger", () => {
  it("should write logs to file", async () => {
    const mockFiles = new Map<string, string>()
    
    const mockFileSystem = FileSystem.layerNoop({
      writeFileString: (path: string, content: string, options?: any) => 
        Effect.sync(() => {
          const existing = mockFiles.get(path) || ""
          const newContent = options?.flag === "a" ? existing + content : content
          mockFiles.set(path, newContent)
        }),
      makeDirectory: () => Effect.void,
      stat: () => Effect.succeed({ size: 1024 } as any)
    })
    
    const testLogger = Logger.stringLogger.pipe(
      PlatformLogger.toFile("/test/app.log")
    )
    
    const LoggerLive = Logger.replaceScoped(
      Logger.defaultLogger,
      testLogger
    ).pipe(Layer.provide(mockFileSystem))
    
    const program = Effect.gen(function* () {
      yield* Effect.log("Test message 1")
      yield* Effect.log("Test message 2")
    })
    
    await Effect.runPromise(program.pipe(Effect.provide(LoggerLive)))
    
    const logContent = mockFiles.get("/test/app.log")
    expect(logContent).toContain("Test message 1")
    expect(logContent).toContain("Test message 2")
  })
  
  it("should handle file write errors gracefully", async () => {
    const mockFileSystem = FileSystem.layerNoop({
      writeFileString: () => Effect.fail(new Error("Disk full")),
      makeDirectory: () => Effect.void
    })
    
    const testLogger = Logger.stringLogger.pipe(
      PlatformLogger.toFile("/test/app.log")
    )
    
    const LoggerLive = Logger.replaceScoped(
      Logger.defaultLogger,
      testLogger
    ).pipe(Layer.provide(mockFileSystem))
    
    const program = Effect.log("Test message")
    
    // Should not throw, but log the error
    await expect(
      Effect.runPromise(program.pipe(Effect.provide(LoggerLive)))
    ).rejects.toThrow("Disk full")
  })
  
  it("should batch logs correctly", async () => {
    const writtenBatches: string[][] = []
    
    const mockFileSystem = FileSystem.layerNoop({
      writeFileString: (path: string, content: string) => 
        Effect.sync(() => {
          const messages = content.trim().split('\n')
          writtenBatches.push(messages)
        }),
      makeDirectory: () => Effect.void
    })
    
    const batchedLogger = Logger.stringLogger.pipe(
      Logger.batched(Duration.millis(100), (messages) => 
        Effect.gen(function* () {
          const fs = yield* FileSystem.FileSystem
          yield* fs.writeFileString("/test/batched.log", messages.join('\n'))
        })
      )
    )
    
    const LoggerLive = Logger.replaceScoped(
      Logger.defaultLogger,
      batchedLogger
    ).pipe(Layer.provide(mockFileSystem))
    
    const program = Effect.gen(function* () {
      yield* Effect.log("Message 1")
      yield* Effect.log("Message 2")
      yield* Effect.log("Message 3")
      yield* Effect.sleep(Duration.millis(150)) // Wait for batch to flush
    })
    
    await Effect.runPromise(program.pipe(Effect.provide(LoggerLive)))
    
    expect(writtenBatches).toHaveLength(1)
    expect(writtenBatches[0]).toHaveLength(3)
  })
})

// Integration testing with temporary files
const createTempLoggerTest = (testName: string) => Effect.gen(function* () {
  const fs = yield* FileSystem.FileSystem
  const tempDir = yield* fs.makeTempDirectoryScoped()
  const logPath = `${tempDir}/test.log`
  
  const fileLogger = Logger.jsonLogger.pipe(
    PlatformLogger.toFile(logPath)
  )
  
  const LoggerLive = Logger.replaceScoped(
    Logger.defaultLogger,
    fileLogger
  )
  
  yield* Effect.log("Test started", { testName })
  yield* Effect.log("Test data", { value: 42, success: true })
  yield* Effect.log("Test completed", { testName })
  
  // Verify file contents
  const content = yield* fs.readFileString(logPath)
  const lines = content.trim().split('\n')
  
  expect(lines).toHaveLength(3)
  lines.forEach(line => {
    const parsed = JSON.parse(line)
    expect(parsed).toHaveProperty('timestamp')
    expect(parsed).toHaveProperty('logLevel')
    expect(parsed).toHaveProperty('message')
  })
  
  return { logPath, content }
}).pipe(Effect.scoped)
```

## Conclusion

PlatformLogger provides **production-ready file logging**, **structured data support**, and **performance optimization** for Effect applications.

Key benefits:
- **File-Based Logging**: Write logs directly to files with proper rotation and management
- **Structured Data**: JSON and logfmt formats for machine-readable logs
- **Performance Optimization**: Batching reduces I/O overhead in high-throughput scenarios
- **Production Ready**: Integrates with monitoring systems and supports multi-environment configurations
- **Type Safety**: Full TypeScript support with Effect's error handling
- **Composability**: Works seamlessly with other Effect modules and existing logging infrastructure

PlatformLogger is ideal when you need centralized logging, structured data for analysis, log persistence, or integration with external monitoring systems in production applications.