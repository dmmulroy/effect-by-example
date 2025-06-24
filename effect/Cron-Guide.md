# Cron: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem Cron Solves

Scheduling recurring tasks in applications often leads to complex, error-prone code that's difficult to maintain:

```typescript
// Traditional approach - problematic scheduling code
const scheduleBackup = () => {
  setInterval(async () => {
    const now = new Date()
    // Is it 2 AM?
    if (now.getHours() === 2 && now.getMinutes() === 0) {
      try {
        await performBackup()
      } catch (error) {
        console.error('Backup failed:', error)
      }
    }
  }, 60000) // Check every minute - wasteful
}

// What about weekends? Time zones? Error handling?
// This quickly becomes a maintenance nightmare
```

This approach leads to:
- **Inefficient polling** - Constantly checking time instead of smart scheduling
- **Time zone nightmares** - Hard to handle different zones correctly
- **Maintenance burden** - Complex logic for different schedule patterns
- **Poor error handling** - No built-in retry or failure management

### The Cron Solution

Effect's Cron module provides a powerful, type-safe scheduling system based on cron expressions:

```typescript
import { Cron, Effect, Schedule } from "effect"

// Clean, declarative scheduling
const backupSchedule = Cron.unsafeParse("0 0 2 * * *") // Daily at 2 AM
const backupTask = Effect.gen(function* () {
  yield* performBackup()
  yield* logSuccess("Backup completed successfully")
})

// Compose with Effect's powerful scheduling
const scheduledBackup = backupTask.pipe(
  Effect.repeat(Schedule.cron(backupSchedule)),
  Effect.catchAll(error => logError(`Backup failed: ${error}`))
)
```

### Key Concepts

**Cron Expression**: A string format (e.g., "0 0 2 * * *") that defines when tasks should run - seconds, minutes, hours, days, months, weekdays

**Time Zone Awareness**: Built-in support for different time zones, ensuring tasks run at the correct local time

**Type Safety**: Parse-time validation and compile-time type checking for schedule definitions

## Basic Usage Patterns

### Pattern 1: Creating Simple Schedules

```typescript
import { Cron, Either } from "effect"

// Parse a cron expression safely
const dailyBackup = Cron.parse("0 0 2 * * *") // Daily at 2:00 AM

if (Either.isRight(dailyBackup)) {
  console.log("Schedule created successfully")
} else {
  console.error("Invalid cron expression:", dailyBackup.left.message)
}

// Or parse unsafely if you're confident
const weeklyReport = Cron.unsafeParse("0 0 9 * * MON") // Mondays at 9:00 AM
```

### Pattern 2: Manual Schedule Construction

```typescript
import { Cron, DateTime } from "effect"

// Build complex schedules programmatically
const businessHoursSchedule = Cron.make({
  seconds: [0],
  minutes: [0, 15, 30, 45], // Every 15 minutes
  hours: [9, 10, 11, 12, 13, 14, 15, 16, 17], // 9 AM to 5 PM
  days: [], // Any day of month
  months: [], // Any month
  weekdays: [1, 2, 3, 4, 5], // Monday to Friday
  tz: DateTime.zoneUnsafeMakeNamed("America/New_York")
})
```

### Pattern 3: Checking Schedule Matches

```typescript
import { Cron } from "effect"

const schedule = Cron.unsafeParse("0 30 14 * * *") // Daily at 2:30 PM

// Check if a specific time matches the schedule
const checkTime = new Date("2024-12-24 14:30:00")
const isScheduled = Cron.match(schedule, checkTime)

console.log(`Task scheduled at ${checkTime.toISOString()}:`, isScheduled)
```

## Real-World Examples

### Example 1: Database Backup System

```typescript
import { Cron, Effect, Schedule, Console, DateTime } from "effect"
import { pipe } from "effect/Function"

// Define backup schedules for different environments
const createBackupSchedule = (environment: "production" | "staging") => {
  const schedules = {
    production: "0 0 2 * * *", // Daily at 2 AM
    staging: "0 0 4 * * SUN"   // Weekly on Sunday at 4 AM
  }
  return Cron.unsafeParse(schedules[environment], "UTC")
}

// Database backup task
const performDatabaseBackup = (database: string) => Effect.gen(function* () {
  yield* Console.log(`Starting backup for database: ${database}`)
  
  // Simulate backup process
  yield* Effect.sleep("5 seconds")
  
  const timestamp = new Date().toISOString()
  const backupFile = `${database}_backup_${timestamp}.sql`
  
  yield* Console.log(`Backup completed: ${backupFile}`)
  return backupFile
})

// Scheduled backup service
const createBackupService = (database: string, environment: "production" | "staging") => {
  const schedule = createBackupSchedule(environment)
  
  return performDatabaseBackup(database).pipe(
    Effect.repeat(Schedule.cron(schedule)),
    Effect.catchAll(error => 
      Console.error(`Backup failed for ${database}: ${error}`)
    ),
    Effect.withSpan("database.backup", {
      attributes: { 
        "db.name": database, 
        "env": environment 
      }
    })
  )
}

// Usage
const productionBackup = createBackupService("users_db", "production")
const stagingBackup = createBackupService("test_db", "staging")
```

### Example 2: Report Generation System

```typescript
import { Cron, Effect, Schedule, Console, DateTime, Array as Arr } from "effect"

// Report configuration
interface ReportConfig {
  readonly name: string
  readonly cronExpression: string
  readonly timeZone: string
  readonly recipients: ReadonlyArray<string>
}

const reportConfigs: ReadonlyArray<ReportConfig> = [
  {
    name: "Daily Sales Report",
    cronExpression: "0 0 8 * * MON-FRI", // Weekdays at 8 AM
    timeZone: "America/New_York",
    recipients: ["sales@company.com", "management@company.com"]
  },
  {
    name: "Weekly Performance Report", 
    cronExpression: "0 0 9 * * MON", // Mondays at 9 AM
    timeZone: "America/New_York",
    recipients: ["executives@company.com"]
  },
  {
    name: "Monthly Financial Report",
    cronExpression: "0 0 10 1 * *", // First day of month at 10 AM
    timeZone: "America/New_York", 
    recipients: ["finance@company.com", "cfo@company.com"]
  }
]

// Report generation logic
const generateReport = (config: ReportConfig) => Effect.gen(function* () {
  yield* Console.log(`Generating ${config.name}...`)
  
  // Simulate report generation
  yield* Effect.sleep("2 seconds")
  
  const reportData = {
    name: config.name,
    generatedAt: new Date().toISOString(),
    recipients: config.recipients
  }
  
  yield* Console.log(`${config.name} generated and sent to ${config.recipients.length} recipients`)
  return reportData
})

// Create scheduled report service
const createScheduledReport = (config: ReportConfig) => {
  const schedule = Cron.unsafeParse(
    config.cronExpression, 
    DateTime.zoneUnsafeMakeNamed(config.timeZone)
  )
  
  return generateReport(config).pipe(
    Effect.repeat(Schedule.cron(schedule)),
    Effect.catchAll(error => 
      Console.error(`Report generation failed for ${config.name}: ${error}`)
    )
  )
}

// Start all scheduled reports
const reportingService = Effect.gen(function* () {
  const reportTasks = reportConfigs.map(createScheduledReport)
  yield* Effect.all(reportTasks, { concurrency: "unbounded" })
})
```

### Example 3: System Maintenance Scheduler

```typescript
import { Cron, Effect, Schedule, Console, DateTime } from "effect"

// Maintenance task definitions
const maintenanceTasks = {
  clearTempFiles: Effect.gen(function* () {
    yield* Console.log("Clearing temporary files...")
    yield* Effect.sleep("1 second")
    yield* Console.log("Temporary files cleared")
  }),
  
  optimizeDatabase: Effect.gen(function* () {
    yield* Console.log("Optimizing database indexes...")
    yield* Effect.sleep("3 seconds") 
    yield* Console.log("Database optimization completed")
  }),
  
  generateHealthReport: Effect.gen(function* () {
    yield* Console.log("Generating system health report...")
    yield* Effect.sleep("2 seconds")
    yield* Console.log("Health report generated")
  }),
  
  updateSystemMetrics: Effect.gen(function* () {
    yield* Console.log("Updating system metrics...")
    yield* Effect.sleep("500 milliseconds")
    yield* Console.log("System metrics updated")
  })
}

// Maintenance schedules with different frequencies
const maintenanceScheduler = Effect.gen(function* () {
  // Every 6 hours - clear temp files
  const tempFileCleanup = maintenanceTasks.clearTempFiles.pipe(
    Effect.repeat(Schedule.cron("0 0 */6 * * *")),
    Effect.fork
  )
  
  // Daily at 3 AM - database optimization  
  const dbOptimization = maintenanceTasks.optimizeDatabase.pipe(
    Effect.repeat(Schedule.cron("0 0 3 * * *")),
    Effect.fork
  )
  
  // Every 4 hours - health report
  const healthReporting = maintenanceTasks.generateHealthReport.pipe(
    Effect.repeat(Schedule.cron("0 0 */4 * * *")),
    Effect.fork
  )
  
  // Every 15 minutes - metrics update
  const metricsUpdate = maintenanceTasks.updateSystemMetrics.pipe(
    Effect.repeat(Schedule.cron("0 */15 * * * *")),
    Effect.fork
  )
  
  // Wait for all maintenance tasks
  const fibers = [tempFileCleanup, dbOptimization, healthReporting, metricsUpdate]
  yield* Effect.all(fibers)
})
```

## Advanced Features Deep Dive

### Feature 1: Time Zone Management

Time zones are crucial for distributed applications and global systems.

#### Basic Time Zone Usage

```typescript
import { Cron, DateTime } from "effect"

// Create timezone-aware schedules
const newYorkSchedule = Cron.unsafeParse(
  "0 0 9 * * MON-FRI", 
  DateTime.zoneUnsafeMakeNamed("America/New_York")
)

const londonSchedule = Cron.unsafeParse(
  "0 0 14 * * MON-FRI",
  DateTime.zoneUnsafeMakeNamed("Europe/London")  
)

const tokyoSchedule = Cron.unsafeParse(
  "0 0 22 * * MON-FRI",
  DateTime.zoneUnsafeMakeNamed("Asia/Tokyo")
)
```

#### Real-World Time Zone Example

```typescript
import { Cron, Effect, Schedule, Console, DateTime } from "effect"

// Global notification system
const createGlobalNotificationSchedule = (regions: ReadonlyArray<{
  name: string
  timeZone: string
  localTime: string // e.g., "09:00" for 9 AM local time
}>) => {
  
  return regions.map(region => {
    const [hour, minute] = region.localTime.split(":").map(Number)
    const cronExpression = `0 ${minute} ${hour} * * MON-FRI`
    
    const schedule = Cron.unsafeParse(
      cronExpression,
      DateTime.zoneUnsafeMakeNamed(region.timeZone)
    )
    
    const sendNotification = Effect.gen(function* () {
      yield* Console.log(`Sending morning notification to ${region.name}`)
      yield* Effect.sleep("100 milliseconds")
      yield* Console.log(`Notification sent to ${region.name} at local time ${region.localTime}`)
    })
    
    return sendNotification.pipe(
      Effect.repeat(Schedule.cron(schedule)),
      Effect.withSpan("notification.send", { 
        attributes: { region: region.name, timezone: region.timeZone }
      })
    )
  })
}

// Global regions configuration
const globalRegions = [
  { name: "New York", timeZone: "America/New_York", localTime: "09:00" },
  { name: "London", timeZone: "Europe/London", localTime: "09:00" },
  { name: "Tokyo", timeZone: "Asia/Tokyo", localTime: "09:00" },
  { name: "Sydney", timeZone: "Australia/Sydney", localTime: "09:00" }
]

const globalNotificationService = Effect.gen(function* () {
  const schedules = createGlobalNotificationSchedule(globalRegions)
  yield* Effect.all(schedules, { concurrency: "unbounded" })
})
```

#### Advanced Time Zone: Daylight Saving Time Handling

```typescript
import { Cron, Effect, Schedule, Console, DateTime } from "effect"

// Schedule that handles DST transitions gracefully
const createDSTAwareSchedule = (timeZone: string) => {
  const schedule = Cron.unsafeParse("0 0 2 * * *", timeZone) // 2 AM daily
  
  const dstAwareTask = Effect.gen(function* () {
    const currentTime = DateTime.now()
    const zonedTime = DateTime.makeZoned(currentTime, {
      timeZone: DateTime.zoneUnsafeMakeNamed(timeZone)
    })
    
    yield* Console.log(`Task running at ${DateTime.formatIsoString(zonedTime)} ${timeZone}`)
    
    // Your actual task logic here
    yield* Effect.sleep("1 second")
    
    yield* Console.log("DST-aware task completed")
  })
  
  return dstAwareTask.pipe(
    Effect.repeat(Schedule.cron(schedule)),
    Effect.catchAll(error => Console.error(`DST-aware task failed: ${error}`))
  )
}
```

### Feature 2: Schedule Parsing and Validation

Effect Cron provides robust parsing with detailed error information.

#### Safe Parsing with Error Handling

```typescript
import { Cron, Either, Effect, Console } from "effect"

const parseScheduleSafely = (expression: string, timeZone?: string) => 
  Effect.gen(function* () {
    const result = Cron.parse(expression, timeZone)
    
    if (Either.isLeft(result)) {
      yield* Console.error(`Failed to parse cron expression "${expression}": ${result.left.message}`)
      return yield* Effect.fail(result.left)
    }
    
    yield* Console.log(`Successfully parsed cron expression: "${expression}"`)
    return result.right
  })

// Validate multiple schedules
const validateSchedules = (schedules: ReadonlyArray<{
  name: string
  expression: string
  timeZone?: string
}>) => Effect.gen(function* () {
  const results = yield* Effect.all(
    schedules.map(schedule => 
      parseScheduleSafely(schedule.expression, schedule.timeZone).pipe(
        Effect.map(cron => ({ name: schedule.name, cron, valid: true })),
        Effect.catchAll(error => Effect.succeed({ 
          name: schedule.name, 
          error: error.message, 
          valid: false 
        }))
      )
    )
  )
  
  const valid = results.filter(r => r.valid)
  const invalid = results.filter(r => !r.valid)
  
  yield* Console.log(`Validated ${results.length} schedules: ${valid.length} valid, ${invalid.length} invalid`)
  
  if (invalid.length > 0) {
    yield* Console.error("Invalid schedules:")
    yield* Effect.forEach(invalid, (schedule) =>
      Console.error(`  ${schedule.name}: ${schedule.error}`)
    )
  }
  
  return { valid, invalid }
})

// Example usage
const scheduleConfigs = [
  { name: "Daily Backup", expression: "0 0 2 * * *" },
  { name: "Hourly Sync", expression: "0 0 * * * *" },
  { name: "Invalid Schedule", expression: "invalid cron" },
  { name: "Weekly Report", expression: "0 0 9 * * MON", timeZone: "UTC" }
]

const validationResult = validateSchedules(scheduleConfigs)
```

### Feature 3: Next Execution Calculation

Find when schedules will next execute, useful for monitoring and debugging.

#### Basic Next Execution

```typescript
import { Cron, Effect, Console } from "effect"

const analyzeSchedule = (expression: string, timeZone?: string) => 
  Effect.gen(function* () {
    const schedule = Cron.unsafeParse(expression, timeZone)
    const now = new Date()
    
    // Get next 5 execution times
    const upcoming = []
    let current = now
    
    for (let i = 0; i < 5; i++) {
      const next = Cron.next(schedule, current)
      upcoming.push(next)
      current = next
    }
    
    yield* Console.log(`Schedule: ${expression}`)
    yield* Console.log(`Current time: ${now.toISOString()}`)
    yield* Console.log("Upcoming executions:")
    
    yield* Effect.forEach(upcoming, (date, index) =>
      Console.log(`  ${index + 1}. ${date.toISOString()}`)
    )
    
    return upcoming
  })

// Schedule analysis helper
const createScheduleAnalyzer = () => ({
  analyze: analyzeSchedule,
  
  // Check if schedule will run in next N hours
  willRunInNextHours: (expression: string, hours: number, timeZone?: string) => 
    Effect.gen(function* () {
      const schedule = Cron.unsafeParse(expression, timeZone)
      const now = new Date()
      const futureTime = new Date(now.getTime() + hours * 60 * 60 * 1000)
      
      let current = now
      const executions = []
      
      while (current < futureTime) {
        const next = Cron.next(schedule, current)
        if (next <= futureTime) {
          executions.push(next)
          current = next
        } else {
          break
        }
      }
      
      yield* Console.log(`Schedule "${expression}" will run ${executions.length} times in the next ${hours} hours`)
      return executions
    })
})
```

## Practical Patterns & Best Practices

### Pattern 1: Composable Schedule Builder

```typescript
import { Cron, Effect, Schedule, Console } from "effect"

// Helper functions for common schedule patterns
const ScheduleHelpers = {
  // Business hours (9 AM - 5 PM, weekdays)
  businessHours: (timeZone = "UTC") => 
    Cron.unsafeParse("0 0 9-17 * * MON-FRI", timeZone),
  
  // Every N minutes during business hours
  everyNMinutesDuringBusinessHours: (minutes: number, timeZone = "UTC") => 
    Cron.unsafeParse(`0 */${minutes} 9-17 * * MON-FRI`, timeZone),
  
  // Daily at specific time
  dailyAt: (hour: number, minute = 0, timeZone = "UTC") =>
    Cron.unsafeParse(`0 ${minute} ${hour} * * *`, timeZone),
  
  // Weekly on specific day and time
  weeklyOn: (day: "MON" | "TUE" | "WED" | "THU" | "FRI" | "SAT" | "SUN", hour: number, minute = 0, timeZone = "UTC") =>
    Cron.unsafeParse(`0 ${minute} ${hour} * * ${day}`, timeZone),
  
  // Monthly on first/last day
  monthlyOnFirst: (hour: number, minute = 0, timeZone = "UTC") =>
    Cron.unsafeParse(`0 ${minute} ${hour} 1 * *`, timeZone),
  
  monthlyOnLast: (hour: number, minute = 0, timeZone = "UTC") =>
    Cron.unsafeParse(`0 ${minute} ${hour} 28-31 * *`, timeZone)
}

// Composable task scheduler
const createTaskScheduler = <R, E>(tasks: {
  [K in string]: {
    task: Effect.Effect<void, E, R>
    schedule: Cron.Cron
    description: string
  }
}) => {
  
  const startTask = <K extends keyof typeof tasks>(name: K) => {
    const { task, schedule, description } = tasks[name]
    
    return task.pipe(
      Effect.repeat(Schedule.cron(schedule)),
      Effect.withSpan(`scheduled-task.${String(name)}`, {
        attributes: { description }
      }),
      Effect.catchAll(error => 
        Console.error(`Task ${String(name)} failed: ${error}`)
      ),
      Effect.fork
    )
  }
  
  const startAllTasks = () => Effect.gen(function* () {
    const taskNames = Object.keys(tasks) as Array<keyof typeof tasks>
    const fibers = yield* Effect.all(
      taskNames.map(startTask),
      { concurrency: "unbounded" }
    )
    
    yield* Console.log(`Started ${taskNames.length} scheduled tasks`)
    return fibers
  })
  
  return { startTask, startAllTasks }
}

// Example usage
const myTasks = {
  backup: {
    task: Effect.gen(function* () {
      yield* Console.log("Running backup...")
      yield* Effect.sleep("2 seconds")
      yield* Console.log("Backup completed")
    }),
    schedule: ScheduleHelpers.dailyAt(2), // 2 AM daily
    description: "Daily database backup"
  },
  
  healthCheck: {
    task: Effect.gen(function* () {
      yield* Console.log("Health check...")
      yield* Effect.sleep("100 milliseconds")
      yield* Console.log("System healthy")
    }),
    schedule: ScheduleHelpers.everyNMinutesDuringBusinessHours(15), // Every 15 minutes during business hours
    description: "System health monitoring"
  },
  
  weeklyReport: {
    task: Effect.gen(function* () {
      yield* Console.log("Generating weekly report...")
      yield* Effect.sleep("5 seconds")
      yield* Console.log("Weekly report sent")
    }),
    schedule: ScheduleHelpers.weeklyOn("MON", 9), // Monday at 9 AM
    description: "Weekly performance report"
  }
}

const scheduler = createTaskScheduler(myTasks)
const runAllTasks = scheduler.startAllTasks()
```

### Pattern 2: Conditional Schedule Execution

```typescript
import { Cron, Effect, Schedule, Console, Option } from "effect"

// Conditional execution based on external factors
const createConditionalSchedule = <R, E>(
  schedule: Cron.Cron,
  task: Effect.Effect<void, E, R>,
  condition: Effect.Effect<boolean, never, never>
) => {
  const conditionalTask = Effect.gen(function* () {
    const shouldRun = yield* condition
    
    if (shouldRun) {
      yield* Console.log("Condition met, executing task...")
      yield* task
    } else {
      yield* Console.log("Condition not met, skipping task execution")
    }
  })
  
  return conditionalTask.pipe(
    Effect.repeat(Schedule.cron(schedule))
  )
}

// Example: Only run backup during maintenance window
const isMaintenanceWindow = Effect.gen(function* () {
  const now = new Date()
  const hour = now.getHours()
  
  // Maintenance window: 2 AM - 4 AM
  const inWindow = hour >= 2 && hour < 4
  yield* Console.log(`Current hour: ${hour}, in maintenance window: ${inWindow}`)
  
  return inWindow
})

const backupTask = Effect.gen(function* () {
  yield* Console.log("Performing backup during maintenance window...")
  yield* Effect.sleep("1 second")
  yield* Console.log("Backup completed")
})

const conditionalBackup = createConditionalSchedule(
  Cron.unsafeParse("0 */30 * * * *"), // Every 30 minutes
  backupTask,
  isMaintenanceWindow
)
```

### Pattern 3: Schedule Monitoring and Alerting

```typescript
import { Cron, Effect, Schedule, Console, DateTime, Ref } from "effect"

// Schedule monitoring system
const createScheduleMonitor = () => {
  
  const createMonitoredSchedule = <R, E>(
    name: string,
    schedule: Cron.Cron,
    task: Effect.Effect<void, E, R>,
    options?: {
      maxFailures?: number
      alertThreshold?: number
    }
  ) => {
    const maxFailures = options?.maxFailures ?? 3
    const alertThreshold = options?.alertThreshold ?? 2
    
    return Effect.gen(function* () {
      const failureCount = yield* Ref.make(0)
      const lastSuccess = yield* Ref.make(Option.none<Date>())
      const lastFailure = yield* Ref.make(Option.none<Date>())
      
      const monitoredTask = Effect.gen(function* () {
        const startTime = new Date()
        
        yield* task.pipe(
          Effect.tapBoth({
            onFailure: (error) => Effect.gen(function* () {
              yield* Ref.update(failureCount, n => n + 1)
              yield* Ref.set(lastFailure, Option.some(startTime))
              
              const failures = yield* Ref.get(failureCount)
              
              if (failures >= alertThreshold) {
                yield* Console.error(`ALERT: Task "${name}" has failed ${failures} times`)
              }
              
              if (failures >= maxFailures) {
                yield* Console.error(`CRITICAL: Task "${name}" exceeded max failures (${maxFailures})`)
                return yield* Effect.fail(new Error(`Task "${name}" exceeded failure threshold`))
              }
              
              yield* Console.error(`Task "${name}" failed: ${error}`)
            }),
            onSuccess: () => Effect.gen(function* () {
              yield* Ref.set(failureCount, 0) // Reset failure count on success
              yield* Ref.set(lastSuccess, Option.some(startTime))
              yield* Console.log(`Task "${name}" completed successfully`)
            })
          })
        )
      })
      
      return monitoredTask.pipe(
        Effect.repeat(Schedule.cron(schedule)),
        Effect.withSpan(`monitored-task.${name}`)
      )
    })
  }
  
  return { createMonitoredSchedule }
}

// Example usage
const monitor = createScheduleMonitor()

const monitoredBackup = monitor.createMonitoredSchedule(
  "database-backup",
  Cron.unsafeParse("0 0 2 * * *"), // Daily at 2 AM
  Effect.gen(function* () {
    // Simulate backup that might fail
    const shouldFail = Math.random() < 0.3 // 30% chance of failure
    
    if (shouldFail) {
      return yield* Effect.fail(new Error("Backup operation failed"))
    }
    
    yield* Effect.sleep("2 seconds")
    yield* Console.log("Database backup completed successfully")
  }),
  {
    maxFailures: 3,
    alertThreshold: 2
  }
)
```

## Integration Examples

### Integration with HTTP Servers

```typescript
import { Cron, Effect, Schedule, Console, HttpServer, HttpRouter } from "effect"

// HTTP server with scheduled tasks
const createScheduledWebServer = () => {
  // Scheduled task to clean up old sessions
  const cleanupTask = Effect.gen(function* () {
    yield* Console.log("Cleaning up expired sessions...")
    yield* Effect.sleep("500 milliseconds")
    yield* Console.log("Session cleanup completed")
  })
  
  // Schedule cleanup every hour
  const cleanupSchedule = cleanupTask.pipe(
    Effect.repeat(Schedule.cron("0 0 * * * *")),
    Effect.fork
  )
  
  // HTTP endpoints
  const routes = HttpRouter.empty.pipe(
    HttpRouter.get("/health", Effect.succeed({ status: "healthy", timestamp: new Date().toISOString() })),
    HttpRouter.get("/schedule/status", Effect.gen(function* () {
      const schedule = Cron.unsafeParse("0 0 * * * *")
      const nextExecution = Cron.next(schedule)
      
      return {
        nextCleanup: nextExecution.toISOString(),
        scheduleExpression: "0 0 * * * *"
      }
    }))
  )
  
  const server = HttpServer.make(routes, { port: 3000 })
  
  return Effect.gen(function* () {
    // Start cleanup schedule
    yield* cleanupSchedule
    
    // Start HTTP server
    yield* server
  })
}
```

### Integration with Database Operations

```typescript
import { Cron, Effect, Schedule, Console, DateTime } from "effect"

// Database service interface
interface DatabaseService {
  readonly backup: () => Effect.Effect<string, Error, never>
  readonly optimize: () => Effect.Effect<void, Error, never>
  readonly cleanup: (olderThan: Date) => Effect.Effect<number, Error, never>
}

// Mock database service
const DatabaseServiceLive: DatabaseService = {
  backup: () => Effect.succeed(`backup_${Date.now()}.sql`),
  optimize: () => Effect.gen(function* () {
    yield* Effect.sleep("2 seconds")
    yield* Console.log("Database optimization completed")
  }),
  cleanup: (olderThan: Date) => Effect.gen(function* () {
    yield* Effect.sleep("1 second")
    const deletedCount = Math.floor(Math.random() * 100)
    yield* Console.log(`Cleaned up ${deletedCount} records older than ${olderThan.toISOString()}`)
    return deletedCount
  })
}

// Database maintenance scheduler
const createDatabaseMaintenanceScheduler = (db: DatabaseService) => {
  
  // Daily backup at 2 AM
  const backupSchedule = Effect.gen(function* () {
    yield* Console.log("Starting database backup...")
    const backupFile = yield* db.backup()
    yield* Console.log(`Backup completed: ${backupFile}`)
  }).pipe(
    Effect.repeat(Schedule.cron("0 0 2 * * *")),
    Effect.catchAll(error => Console.error(`Backup failed: ${error}`)),
    Effect.withSpan("db.backup"),
    Effect.fork
  )
  
  // Weekly optimization on Sunday at 3 AM
  const optimizeSchedule = Effect.gen(function* () {
    yield* Console.log("Starting database optimization...")
    yield* db.optimize()
  }).pipe(
    Effect.repeat(Schedule.cron("0 0 3 * * SUN")),
    Effect.catchAll(error => Console.error(`Optimization failed: ${error}`)),
    Effect.withSpan("db.optimize"),
    Effect.fork
  )
  
  // Daily cleanup of old records at 1 AM
  const cleanupSchedule = Effect.gen(function* () {
    const thirtyDaysAgo = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000)
    yield* Console.log(`Starting cleanup of records older than ${thirtyDaysAgo.toISOString()}`)
    const deletedCount = yield* db.cleanup(thirtyDaysAgo)
    yield* Console.log(`Cleanup completed: ${deletedCount} records deleted`)
  }).pipe(
    Effect.repeat(Schedule.cron("0 0 1 * * *")),
    Effect.catchAll(error => Console.error(`Cleanup failed: ${error}`)),
    Effect.withSpan("db.cleanup"),
    Effect.fork
  )
  
  return Effect.gen(function* () {
    yield* Console.log("Starting database maintenance scheduler...")
    yield* Effect.all([backupSchedule, optimizeSchedule, cleanupSchedule])
  })
}

// Usage
const maintenanceService = createDatabaseMaintenanceScheduler(DatabaseServiceLive)
```

### Testing Strategies

```typescript
import { Cron, Effect, Schedule, TestClock, TestContext, Fiber, Console } from "effect"

// Test helper for cron schedules
const testCronSchedule = <A, E, R>(
  schedule: Cron.Cron,
  task: Effect.Effect<A, E, R>,
  duration: string = "1 day"
) => Effect.gen(function* () {
  
  const results: Array<A> = []
  
  const scheduledTask = Effect.gen(function* () {
    const result = yield* task
    results.push(result)
    return result
  }).pipe(
    Effect.repeat(Schedule.cron(schedule))
  )
  
  const fiber = yield* Effect.fork(scheduledTask)
  
  // Advance test clock
  yield* TestClock.adjust(duration)
  
  // Interrupt the scheduled task
  yield* Fiber.interrupt(fiber)
  
  return results
})

// Example test
const testDailySchedule = Effect.gen(function* () {
  const schedule = Cron.unsafeParse("0 0 12 * * *") // Daily at noon
  
  const task = Effect.gen(function* () {
    const timestamp = new Date().toISOString()
    yield* Console.log(`Task executed at: ${timestamp}`)
    return timestamp
  })
  
  const results = yield* testCronSchedule(schedule, task, "3 days")
  
  yield* Console.log(`Task executed ${results.length} times over 3 days`)
  yield* Effect.forEach(results, result => 
    Console.log(`Execution: ${result}`)
  )
  
  return results
}).pipe(
  Effect.provide(TestContext.TestContext)
)

// Property-based testing for cron expressions
const testCronExpression = (expression: string) => Effect.gen(function* () {
  // Test parsing
  const parseResult = Cron.parse(expression)
  
  if (parseResult._tag === "Left") {
    yield* Console.error(`Failed to parse: ${expression}`)
    return false
  }
  
  const cron = parseResult.right
  
  // Test next execution calculation
  const now = new Date()
  const next = Cron.next(cron, now)
  
  // Verify next execution is in the future
  const isInFuture = next.getTime() > now.getTime()
  
  // Test matching
  const matches = Cron.match(cron, next)
  
  yield* Console.log(`Expression: ${expression}`)
  yield* Console.log(`Next execution: ${next.toISOString()}`)
  yield* Console.log(`Is in future: ${isInFuture}`)
  yield* Console.log(`Matches schedule: ${matches}`)
  
  return isInFuture && matches
})

// Test suite
const cronTestSuite = Effect.gen(function* () {
  const expressions = [
    "0 0 0 * * *",     // Daily at midnight
    "0 0 12 * * *",    // Daily at noon
    "0 0 9 * * MON",   // Monday at 9 AM
    "0 */15 * * * *",  // Every 15 minutes
    "0 0 0 1 * *"      // First day of month
  ]
  
  yield* Console.log("Testing cron expressions...")
  
  const results = yield* Effect.all(
    expressions.map(testCronExpression)
  )
  
  const passed = results.filter(Boolean).length
  yield* Console.log(`Test results: ${passed}/${results.length} passed`)
  
  return results
})
```

## Conclusion

The Cron module provides powerful, type-safe scheduling capabilities for Effect applications, offering robust error handling, time zone awareness, and seamless integration with Effect's ecosystem.

Key benefits:
- **Type Safety**: Compile-time validation and runtime parsing with detailed error messages
- **Time Zone Support**: First-class support for global applications with DST handling  
- **Composability**: Natural integration with Effect's scheduling, error handling, and concurrency primitives

Use Cron when you need reliable, maintainable scheduling for background tasks, data processing pipelines, maintenance operations, or any time-based automation in your Effect applications.