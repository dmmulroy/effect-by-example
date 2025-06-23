# ScheduleDecision: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem ScheduleDecision Solves

When building custom scheduling logic, you often need fine-grained control over when operations should continue or stop. Traditional approaches lead to complex state management and brittle retry logic:

```typescript
// Traditional approach - complex state management
class CustomRetryLogic {
  private attempts = 0
  private maxAttempts = 3
  private backoffMs = 1000

  async shouldRetry(error: Error): Promise<{ continue: boolean; delay?: number }> {
    this.attempts++
    if (this.attempts >= this.maxAttempts) {
      return { continue: false }
    }
    
    // Complex logic for different error types
    if (error.message.includes('rate limit')) {
      return { continue: true, delay: this.backoffMs * 2 }
    }
    
    return { continue: true, delay: this.backoffMs }
  }
}
```

This approach leads to:
- **Complex State Management** - Manual tracking of attempts, delays, and conditions
- **Error-Prone Logic** - Easy to introduce bugs in retry/scheduling decisions
- **Limited Composability** - Hard to combine different scheduling strategies
- **Type Safety Issues** - No compile-time guarantees about decision consistency

### The ScheduleDecision Solution

ScheduleDecision provides a type-safe, composable way to make scheduling decisions with precise control over timing intervals:

```typescript
import { ScheduleDecision, Schedule, Effect, Duration } from "effect"

// The Effect solution - clean, type-safe, composable
const makeSmartRetryDecision = (
  attempt: number,
  error: unknown
): ScheduleDecision.ScheduleDecision => {
  if (attempt >= 3) {
    return ScheduleDecision.done
  }
  
  const delay = Duration.millis(1000 * Math.pow(2, attempt))
  return ScheduleDecision.continueWith(
    Interval.after(Date.now() + Duration.toMillis(delay))
  )
}
```

### Key Concepts

**ScheduleDecision**: A discriminated union representing whether to continue scheduling with specific timing or stop entirely

**Continue**: Indicates scheduling should continue with precise interval timing information

**Done**: Indicates scheduling should terminate immediately

## Basic Usage Patterns

### Pattern 1: Simple Continue/Done Decisions

```typescript
import { ScheduleDecision, Interval } from "effect"

// Basic decision making
const shouldContinue = (attempt: number): ScheduleDecision.ScheduleDecision => {
  if (attempt < 5) {
    // Continue after 1 second
    return ScheduleDecision.continueWith(
      Interval.after(Date.now() + 1000)
    )
  }
  return ScheduleDecision.done
}

// Check decision type
const decision = shouldContinue(3)

if (ScheduleDecision.isContinue(decision)) {
  console.log("Will continue with intervals:", decision.intervals)
} else {
  console.log("Scheduling is done")
}
```

### Pattern 2: Interval-Based Decisions

```typescript
import { ScheduleDecision, Interval, Duration } from "effect"

// Create decisions with specific timing
const createDelayedDecision = (delayMs: number): ScheduleDecision.ScheduleDecision => {
  const now = Date.now()
  return ScheduleDecision.continueWith(
    Interval.after(now + delayMs)
  )
}

// Create decisions with time windows
const createWindowedDecision = (
  startMs: number,
  endMs: number
): ScheduleDecision.ScheduleDecision => {
  return ScheduleDecision.continueWith(
    Interval.make(startMs, endMs)
  )
}
```

### Pattern 3: Conditional Decision Logic

```typescript
import { ScheduleDecision, Interval, Duration } from "effect"

// Decision based on multiple conditions
const makeConditionalDecision = (
  attempts: number,
  lastError: Error | null,
  resourceAvailable: boolean
): ScheduleDecision.ScheduleDecision => {
  // Stop if too many attempts
  if (attempts >= 10) {
    return ScheduleDecision.done
  }
  
  // Stop if resource unavailable
  if (!resourceAvailable) {
    return ScheduleDecision.done
  }
  
  // Different delays based on error type
  const baseDelay = 1000
  const delay = lastError?.message.includes('rate-limit') 
    ? baseDelay * 5  // Longer delay for rate limits
    : baseDelay * attempts  // Exponential backoff for other errors
  
  return ScheduleDecision.continueWith(
    Interval.after(Date.now() + delay)
  )
}
```

## Real-World Examples

### Example 1: Smart API Retry Logic

Building a resilient API client that adapts its retry strategy based on different error conditions:

```typescript
import { ScheduleDecision, Schedule, Effect, Duration, Interval } from "effect"

interface ApiError {
  readonly status: number
  readonly code: string
  readonly retryAfter?: number
}

interface RetryState {
  readonly attempts: number
  readonly lastError: ApiError | null
  readonly consecutiveFailures: number
}

// Smart retry decision based on error analysis
const makeApiRetryDecision = (
  state: RetryState,
  error: ApiError
): ScheduleDecision.ScheduleDecision => {
  const { attempts, consecutiveFailures } = state
  
  // Stop conditions
  if (attempts >= 5) return ScheduleDecision.done
  if (error.status === 401 || error.status === 403) return ScheduleDecision.done
  if (consecutiveFailures >= 3 && error.status >= 500) return ScheduleDecision.done
  
  // Calculate delay based on error type
  const now = Date.now()
  let delayMs: number
  
  if (error.status === 429) {
    // Rate limiting - respect Retry-After header
    delayMs = error.retryAfter ? error.retryAfter * 1000 : 60000
  } else if (error.status >= 500) {
    // Server errors - exponential backoff with jitter
    const baseDelay = 1000 * Math.pow(2, attempts)
    const jitter = Math.random() * 1000
    delayMs = baseDelay + jitter
  } else {
    // Client errors - linear backoff
    delayMs = 2000 * attempts
  }
  
  return ScheduleDecision.continueWith(
    Interval.after(now + delayMs)
  )
}

// Custom schedule using our decision logic
const createApiRetrySchedule = <A>() => {
  return Schedule.makeWithState<RetryState, ApiError, A>(
    { attempts: 0, lastError: null, consecutiveFailures: 0 },
    (now, error, state) => Effect.gen(function* () {
      const newState: RetryState = {
        attempts: state.attempts + 1,
        lastError: error,
        consecutiveFailures: error.status >= 500 
          ? state.consecutiveFailures + 1 
          : 0
      }
      
      const decision = makeApiRetryDecision(newState, error)
      return [newState, state.attempts, decision] as const
    })
  )
}

// Usage in API client
const apiCall = (url: string) => Effect.gen(function* () {
  const response = yield* fetch(url)
  if (!response.ok) {
    const error: ApiError = {
      status: response.status,
      code: response.statusText,
      retryAfter: response.headers.get('retry-after') 
        ? parseInt(response.headers.get('retry-after')!)
        : undefined
    }
    return yield* Effect.fail(error)
  }
  return yield* response.json()
}).pipe(
  Effect.retry(createApiRetrySchedule()),
  Effect.catchAll((error) => 
    Effect.logError(`API call failed after retries: ${error.code}`)
  )
)
```

### Example 2: Database Connection Pool Management

Managing database connections with intelligent retry and backoff strategies:

```typescript
import { ScheduleDecision, Schedule, Effect, Duration, Interval, Ref } from "effect"

interface ConnectionPoolState {
  readonly activeConnections: number
  readonly maxConnections: number
  readonly failedAttempts: number
  readonly lastConnectionTime: number
}

interface ConnectionError {
  readonly type: 'timeout' | 'pool_exhausted' | 'network' | 'auth'
  readonly message: string
}

const makeConnectionRetryDecision = (
  poolState: ConnectionPoolState,
  error: ConnectionError
): ScheduleDecision.ScheduleDecision => {
  const { activeConnections, maxConnections, failedAttempts } = poolState
  
  // Stop if too many failures
  if (failedAttempts >= 10) {
    return ScheduleDecision.done
  }
  
  // Stop on auth errors
  if (error.type === 'auth') {
    return ScheduleDecision.done
  }
  
  const now = Date.now()
  let delayMs: number
  
  if (error.type === 'pool_exhausted') {
    // Pool exhausted - wait longer, with circuit breaker logic
    const utilizationRatio = activeConnections / maxConnections
    if (utilizationRatio > 0.9) {
      delayMs = 5000 + (failedAttempts * 2000) // Linear increase
    } else {
      delayMs = 1000 // Quick retry if pool has availability
    }
  } else if (error.type === 'timeout') {
    // Network timeouts - exponential backoff with cap
    delayMs = Math.min(30000, 1000 * Math.pow(2, failedAttempts))
  } else {
    // Network errors - moderate backoff
    delayMs = 2000 * failedAttempts
  }
  
  return ScheduleDecision.continueWith(
    Interval.after(now + delayMs)
  )
}

// Connection pool service with retry logic
const ConnectionPoolService = Effect.gen(function* () {
  const poolStateRef = yield* Ref.make<ConnectionPoolState>({
    activeConnections: 0,
    maxConnections: 10,
    failedAttempts: 0,
    lastConnectionTime: 0
  })
  
  const acquireConnection = Effect.gen(function* () {
    const poolState = yield* Ref.get(poolStateRef)
    
    // Simulate connection attempt
    if (poolState.activeConnections >= poolState.maxConnections) {
      return yield* Effect.fail({
        type: 'pool_exhausted' as const,
        message: 'Connection pool exhausted'
      })
    }
    
    // Update pool state
    yield* Ref.update(poolStateRef, state => ({
      ...state,
      activeConnections: state.activeConnections + 1,
      lastConnectionTime: Date.now()
    }))
    
    return { id: Math.random().toString(36) }
  }).pipe(
    Effect.retry(
      Schedule.makeWithState<ConnectionPoolState, ConnectionError, number>(
        { activeConnections: 0, maxConnections: 10, failedAttempts: 0, lastConnectionTime: 0 },
        (now, error, state) => Effect.gen(function* () {
          const currentPoolState = yield* Ref.get(poolStateRef)
          const newState = {
            ...currentPoolState,
            failedAttempts: state.failedAttempts + 1
          }
          
          const decision = makeConnectionRetryDecision(newState, error)
          return [newState, state.failedAttempts, decision] as const
        })
      )
    ),
    Effect.tapError((error) => 
      Effect.logError(`Failed to acquire connection: ${error.message}`)
    )
  )
  
  return { acquireConnection } as const
})
```

### Example 3: Distributed Task Processing

Implementing a task processing system with intelligent scheduling based on worker availability and task priority:

```typescript
import { ScheduleDecision, Schedule, Effect, Duration, Interval, Queue } from "effect"

interface Task {
  readonly id: string
  readonly priority: 'low' | 'medium' | 'high' | 'critical'
  readonly payload: unknown
  readonly maxRetries: number
  readonly createdAt: number
}

interface ProcessingState {
  readonly taskId: string
  readonly attempts: number
  readonly workerLoad: number
  readonly lastProcessedAt: number
}

interface ProcessingError {
  readonly type: 'worker_overload' | 'task_timeout' | 'dependency_failure' | 'system_error'
  readonly taskId: string
  readonly workerLoad: number
}

const makeTaskProcessingDecision = (
  task: Task,
  state: ProcessingState,
  error: ProcessingError
): ScheduleDecision.ScheduleDecision => {
  const { attempts, workerLoad } = state
  
  // Stop if max retries exceeded
  if (attempts >= task.maxRetries) {
    return ScheduleDecision.done
  }
  
  // Stop if task is too old (prevent infinite processing)
  const taskAge = Date.now() - task.createdAt
  if (taskAge > 24 * 60 * 60 * 1000) { // 24 hours
    return ScheduleDecision.done
  }
  
  const now = Date.now()
  let delayMs: number
  
  // Calculate delay based on error type and task priority
  if (error.type === 'worker_overload') {
    // Back off more for overloaded workers
    const loadFactor = Math.min(5, workerLoad / 0.8) // Scale with load
    delayMs = 2000 * loadFactor * attempts
    
    // Critical tasks get priority
    if (task.priority === 'critical') {
      delayMs = Math.min(delayMs, 5000)
    }
  } else if (error.type === 'dependency_failure') {
    // Dependency failures need longer waits
    delayMs = 10000 * Math.pow(1.5, attempts)
  } else if (error.type === 'task_timeout') {
    // Timeouts suggest system stress
    delayMs = 5000 + (attempts * 3000)
  } else {
    // System errors - exponential backoff
    delayMs = 1000 * Math.pow(2, attempts)
  }
  
  // Apply priority-based adjustments
  switch (task.priority) {
    case 'critical':
      delayMs = Math.max(1000, delayMs * 0.5)
      break
    case 'high':
      delayMs = Math.max(2000, delayMs * 0.7)
      break
    case 'low':
      delayMs = delayMs * 2
      break
  }
  
  return ScheduleDecision.continueWith(
    Interval.after(now + delayMs)
  )
}

// Task processor with adaptive scheduling
const TaskProcessor = Effect.gen(function* () {
  const processTask = (task: Task) => Effect.gen(function* () {
    // Simulate task processing
    const startTime = Date.now()
    const workerLoadRef = yield* Ref.make(0.5) // Simulate current worker load
    
    return yield* Effect.gen(function* () {
      const workerLoad = yield* Ref.get(workerLoadRef)
      
      // Simulate various failure conditions
      if (workerLoad > 0.9) {
        return yield* Effect.fail({
          type: 'worker_overload' as const,
          taskId: task.id,
          workerLoad
        })
      }
      
      // Simulate processing
      yield* Effect.sleep(Duration.millis(100))
      return { taskId: task.id, processedAt: Date.now() }
    })
  }).pipe(
    Effect.retry(
      Schedule.makeWithState<ProcessingState, ProcessingError, { taskId: string; processedAt: number }>(
        {
          taskId: task.id,
          attempts: 0,
          workerLoad: 0.5,
          lastProcessedAt: 0
        },
        (now, error, state) => Effect.gen(function* () {
          const newState: ProcessingState = {
            ...state,
            attempts: state.attempts + 1,
            workerLoad: error.workerLoad,
            lastProcessedAt: now
          }
          
          const decision = makeTaskProcessingDecision(task, newState, error)
          return [newState, state.attempts, decision] as const
        })
      )
    ),
    Effect.catchAll((error) => 
      Effect.logError(`Task ${task.id} failed after retries: ${error.type}`)
    )
  )
  
  return { processTask } as const
})
```

## Advanced Features Deep Dive

### Feature 1: Interval-Based Timing Control

ScheduleDecision provides precise control over when operations should execute through interval specifications.

#### Basic Interval Usage

```typescript
import { ScheduleDecision, Interval, Duration } from "effect"

// Schedule for specific time
const scheduleAt = (timestamp: number): ScheduleDecision.ScheduleDecision => {
  return ScheduleDecision.continueWith(
    Interval.after(timestamp)
  )
}

// Schedule within time window
const scheduleWindow = (start: number, end: number): ScheduleDecision.ScheduleDecision => {
  return ScheduleDecision.continueWith(
    Interval.make(start, end)
  )
}
```

#### Real-World Interval Example

```typescript
import { ScheduleDecision, Interval, Schedule, Effect } from "effect"

// Business hours scheduler
const createBusinessHoursDecision = (
  currentTime: number,
  timezone: string = 'UTC'
): ScheduleDecision.ScheduleDecision => {
  const now = new Date(currentTime)
  const hour = now.getHours()
  
  // If within business hours (9 AM - 5 PM), continue immediately
  if (hour >= 9 && hour < 17) {
    return ScheduleDecision.continueWith(
      Interval.after(currentTime)
    )
  }
  
  // Otherwise, schedule for next business day at 9 AM
  const nextBusinessDay = new Date(now)
  nextBusinessDay.setDate(now.getDate() + 1)
  nextBusinessDay.setHours(9, 0, 0, 0)
  
  // Skip weekends
  const dayOfWeek = nextBusinessDay.getDay()
  if (dayOfWeek === 0) { // Sunday
    nextBusinessDay.setDate(nextBusinessDay.getDate() + 1)
  } else if (dayOfWeek === 6) { // Saturday
    nextBusinessDay.setDate(nextBusinessDay.getDate() + 2)
  }
  
  return ScheduleDecision.continueWith(
    Interval.after(nextBusinessDay.getTime())
  )
}

// Usage in email sending service
const sendBusinessEmail = (email: Email) => Effect.gen(function* () {
  // Email sending logic
  yield* Effect.log(`Sending email to ${email.recipient}`)
  return email.id
}).pipe(
  Effect.retry(
    Schedule.makeWithState(
      { attempts: 0 },
      (now, _input, state) => Effect.gen(function* () {
        const decision = createBusinessHoursDecision(now)
        return [
          { attempts: state.attempts + 1 },
          state.attempts,
          decision
        ] as const
      })
    )
  )
)
```

#### Advanced Interval: Combining Multiple Intervals

```typescript
import { ScheduleDecision, Interval, Intervals, Schedule, Effect, Chunk } from "effect"

// Create complex scheduling with multiple intervals
const createMultiIntervalDecision = (
  preferredTimes: number[],
  fallbackDelay: number
): ScheduleDecision.ScheduleDecision => {
  const intervals = preferredTimes.map(time => 
    Interval.make(time, time + 60000) // 1-minute windows
  )
  
  if (intervals.length > 0) {
    return ScheduleDecision.continue(
      Intervals.make(Chunk.fromIterable(intervals))
    )
  }
  
  // Fallback to simple delay
  return ScheduleDecision.continueWith(
    Interval.after(Date.now() + fallbackDelay)
  )
}
```

### Feature 2: Decision Composition and Combination

Advanced patterns for combining multiple decision strategies.

#### Decision Combinators

```typescript
import { ScheduleDecision, Interval, Duration } from "effect"

// Combine decisions with priority logic
const combineDecisions = (
  primary: ScheduleDecision.ScheduleDecision,
  fallback: ScheduleDecision.ScheduleDecision
): ScheduleDecision.ScheduleDecision => {
  if (ScheduleDecision.isContinue(primary)) {
    return primary
  }
  return fallback
}

// Create decision with minimum delay
const withMinimumDelay = (
  decision: ScheduleDecision.ScheduleDecision,
  minDelayMs: number
): ScheduleDecision.ScheduleDecision => {
  if (ScheduleDecision.isDone(decision)) {
    return decision
  }
  
  const now = Date.now()
  const minTime = now + minDelayMs
  
  // Ensure the decision doesn't execute too soon
  return ScheduleDecision.continueWith(
    Interval.after(Math.max(minTime, Intervals.start(decision.intervals)))
  )
}

// Create decision with maximum delay cap
const withMaximumDelay = (
  decision: ScheduleDecision.ScheduleDecision,
  maxDelayMs: number
): ScheduleDecision.ScheduleDecision => {
  if (ScheduleDecision.isDone(decision)) {
    return decision
  }
  
  const now = Date.now()
  const maxTime = now + maxDelayMs
  
  return ScheduleDecision.continueWith(
    Interval.after(Math.min(maxTime, Intervals.start(decision.intervals)))
  )
}
```

#### Real-World Decision Composition

```typescript
import { ScheduleDecision, Schedule, Effect, Interval, Duration } from "effect"

interface CircuitBreakerState {
  readonly failures: number
  readonly lastFailureTime: number
  readonly state: 'closed' | 'open' | 'half-open'
}

// Circuit breaker decision logic
const makeCircuitBreakerDecision = (
  state: CircuitBreakerState,
  error: Error
): ScheduleDecision.ScheduleDecision => {
  const now = Date.now()
  const timeSinceLastFailure = now - state.lastFailureTime
  
  switch (state.state) {
    case 'closed':
      if (state.failures >= 5) {
        // Open circuit - long delay
        return ScheduleDecision.continueWith(
          Interval.after(now + 60000) // 1 minute
        )
      }
      // Continue with exponential backoff
      return ScheduleDecision.continueWith(
        Interval.after(now + 1000 * Math.pow(2, state.failures))
      )
      
    case 'open':
      if (timeSinceLastFailure >= 60000) {
        // Try half-open state
        return ScheduleDecision.continueWith(
          Interval.after(now + 1000)
        )
      }
      // Stay open
      return ScheduleDecision.done
      
    case 'half-open':
      // Single attempt, then decide
      return ScheduleDecision.continueWith(
        Interval.after(now + 5000)
      )
  }
}

// Rate limiting decision logic
const makeRateLimitDecision = (
  requestsInWindow: number,
  windowResetTime: number
): ScheduleDecision.ScheduleDecision => {
  const now = Date.now()
  
  if (requestsInWindow >= 100) { // Rate limit threshold
    const waitTime = windowResetTime - now
    if (waitTime > 0) {
      return ScheduleDecision.continueWith(
        Interval.after(windowResetTime + 1000) // Wait for window reset + buffer
      )
    }
  }
  
  return ScheduleDecision.continueWith(
    Interval.after(now + 100) // Small delay between requests
  )
}

// Composite decision strategy
const makeCompositeDecision = (
  circuitState: CircuitBreakerState,
  rateLimitState: { requests: number; resetTime: number },
  error: Error
): ScheduleDecision.ScheduleDecision => {
  // Get individual decisions
  const circuitDecision = makeCircuitBreakerDecision(circuitState, error)
  const rateLimitDecision = makeRateLimitDecision(
    rateLimitState.requests,
    rateLimitState.resetTime
  )
  
  // If either says stop, stop
  if (ScheduleDecision.isDone(circuitDecision) || ScheduleDecision.isDone(rateLimitDecision)) {
    return ScheduleDecision.done
  }
  
  // Take the later of the two delays
  const circuitDelay = Intervals.start(circuitDecision.intervals)
  const rateLimitDelay = Intervals.start(rateLimitDecision.intervals)
  
  return ScheduleDecision.continueWith(
    Interval.after(Math.max(circuitDelay, rateLimitDelay))
  )
}
```

## Practical Patterns & Best Practices

### Pattern 1: Adaptive Backoff Strategy

```typescript
import { ScheduleDecision, Interval, Duration } from "effect"

// Helper for adaptive backoff based on success/failure patterns
const createAdaptiveBackoffHelper = () => {
  interface BackoffState {
    readonly consecutiveFailures: number
    readonly recentSuccesses: number
    readonly lastSuccessTime: number
  }
  
  return (
    state: BackoffState,
    isSuccess: boolean,
    currentTime: number = Date.now()
  ): [BackoffState, ScheduleDecision.ScheduleDecision] => {
    if (isSuccess) {
      const newState: BackoffState = {
        consecutiveFailures: 0,
        recentSuccesses: state.recentSuccesses + 1,
        lastSuccessTime: currentTime
      }
      
      // Reduce delay after success
      const reductionFactor = Math.min(0.5, state.recentSuccesses * 0.1)
      const delay = Math.max(100, 1000 * reductionFactor)
      
      return [
        newState,
        ScheduleDecision.continueWith(Interval.after(currentTime + delay))
      ]
    } else {
      const newState: BackoffState = {
        consecutiveFailures: state.consecutiveFailures + 1,
        recentSuccesses: Math.max(0, state.recentSuccesses - 1),
        lastSuccessTime: state.lastSuccessTime
      }
      
      // Stop after too many consecutive failures
      if (newState.consecutiveFailures >= 10) {
        return [newState, ScheduleDecision.done]
      }
      
      // Exponential backoff with jitter
      const baseDelay = 1000 * Math.pow(2, newState.consecutiveFailures)
      const jitter = Math.random() * 1000
      const delay = Math.min(60000, baseDelay + jitter) // Cap at 1 minute
      
      return [
        newState,
        ScheduleDecision.continueWith(Interval.after(currentTime + delay))
      ]
    }
  }
}

// Usage in service
const httpServiceWithAdaptiveBackoff = Effect.gen(function* () {
  const backoffHelper = createAdaptiveBackoffHelper()
  
  const makeRequest = (url: string) => Effect.gen(function* () {
    // Request implementation
    const response = yield* fetch(url)
    return response
  }).pipe(
    Effect.retry(
      Schedule.makeWithState(
        { consecutiveFailures: 0, recentSuccesses: 0, lastSuccessTime: 0 },
        (now, _input, state) => Effect.gen(function* () {
          const [newState, decision] = backoffHelper(state, false, now)
          return [newState, state.consecutiveFailures, decision] as const
        })
      )
    )
  )
  
  return { makeRequest } as const
})
```

### Pattern 2: Time-Based Scheduling Windows

```typescript
import { ScheduleDecision, Interval, Schedule, Effect } from "effect"

// Helper for creating time-based scheduling windows
const createTimeWindowHelper = () => {
  interface TimeWindow {
    readonly startHour: number
    readonly endHour: number
    readonly timezone: string
    readonly weekdays?: number[] // 0 = Sunday, 1 = Monday, etc.
  }
  
  const isWithinWindow = (
    timestamp: number,
    window: TimeWindow
  ): boolean => {
    const date = new Date(timestamp)
    const hour = date.getHours()
    const dayOfWeek = date.getDay()
    
    // Check hour window
    if (hour < window.startHour || hour >= window.endHour) {
      return false
    }
    
    // Check weekday restriction
    if (window.weekdays && !window.weekdays.includes(dayOfWeek)) {
      return false
    }
    
    return true
  }
  
  const getNextWindowStart = (
    timestamp: number,
    window: TimeWindow
  ): number => {
    const date = new Date(timestamp)
    const nextDate = new Date(date)
    
    // Start with next day
    nextDate.setDate(date.getDate() + 1)
    nextDate.setHours(window.startHour, 0, 0, 0)
    
    // Find next valid weekday
    if (window.weekdays) {
      while (!window.weekdays.includes(nextDate.getDay())) {
        nextDate.setDate(nextDate.getDate() + 1)
      }
    }
    
    return nextDate.getTime()
  }
  
  return {
    createWindowDecision: (
      currentTime: number,
      window: TimeWindow
    ): ScheduleDecision.ScheduleDecision => {
      if (isWithinWindow(currentTime, window)) {
        return ScheduleDecision.continueWith(
          Interval.after(currentTime)
        )
      }
      
      const nextWindowStart = getNextWindowStart(currentTime, window)
      return ScheduleDecision.continueWith(
        Interval.after(nextWindowStart)
      )
    }
  }
}

// Usage for business hours scheduling
const businessHoursScheduler = Effect.gen(function* () {
  const timeWindowHelper = createTimeWindowHelper()
  
  const businessWindow: TimeWindow = {
    startHour: 9,
    endHour: 17,
    timezone: 'UTC',
    weekdays: [1, 2, 3, 4, 5] // Monday through Friday
  }
  
  const scheduleBusinessHoursTask = <A>(task: Effect.Effect<A>) => {
    return task.pipe(
      Effect.retry(
        Schedule.makeWithState(
          { attempts: 0 },
          (now, _input, state) => Effect.gen(function* () {
            const decision = timeWindowHelper.createWindowDecision(now, businessWindow)
            return [
              { attempts: state.attempts + 1 },
              state.attempts,
              decision
            ] as const
          })
        )
      )
    )
  }
  
  return { scheduleBusinessHoursTask } as const
})
```

## Integration Examples

### Integration with Custom Schedule Systems

```typescript
import { ScheduleDecision, Schedule, Effect, Interval, Duration, Ref } from "effect"

// Integration with external scheduling system
interface ExternalSchedulerConfig {
  readonly endpoint: string
  readonly apiKey: string
  readonly maxRetries: number
}

const createExternalSchedulerIntegration = (config: ExternalSchedulerConfig) => {
  return Effect.gen(function* () {
    const stateRef = yield* Ref.make({ 
      lastSync: 0, 
      syncInterval: 30000, // 30 seconds
      failures: 0 
    })
    
    // Create decision based on external scheduler state
    const createSchedulerDecision = (
      taskId: string,
      currentTime: number
    ): Effect.Effect<ScheduleDecision.ScheduleDecision> => Effect.gen(function* () {
      const state = yield* Ref.get(stateRef)
      
      // Check if we need to sync with external scheduler
      if (currentTime - state.lastSync > state.syncInterval) {
        try {
          // Simulate external API call
          const externalDecision = yield* Effect.tryPromise({
            try: () => fetch(`${config.endpoint}/schedule/${taskId}`, {
              headers: { 'Authorization': `Bearer ${config.apiKey}` }
            }),
            catch: (error) => new Error(`External scheduler error: ${error}`)
          })
          
          const data = yield* Effect.tryPromise({
            try: () => externalDecision.json(),
            catch: () => new Error('Failed to parse external scheduler response')
          })
          
          // Update sync state
          yield* Ref.update(stateRef, s => ({ 
            ...s, 
            lastSync: currentTime, 
            failures: 0 
          }))
          
          // Return decision based on external response
          if (data.shouldExecute) {
            return ScheduleDecision.continueWith(
              Interval.after(currentTime + data.delayMs)
            )
          } else {
            return ScheduleDecision.done
          }
        } catch (error) {
          // Handle external scheduler failure
          yield* Ref.update(stateRef, s => ({ 
            ...s, 
            failures: s.failures + 1 
          }))
          
          // Fall back to local decision
          if (state.failures >= config.maxRetries) {
            return ScheduleDecision.done
          }
          
          return ScheduleDecision.continueWith(
            Interval.after(currentTime + 5000 * Math.pow(2, state.failures))
          )
        }
      }
      
      // Use cached decision
      return ScheduleDecision.continueWith(
        Interval.after(currentTime + 1000)
      )
    })
    
    const scheduleWithExternalSystem = <A>(
      taskId: string,
      task: Effect.Effect<A>
    ) => {
      return task.pipe(
        Effect.retry(
          Schedule.makeWithState(
            { attempts: 0, taskId },
            (now, _input, state) => Effect.gen(function* () {
              const decision = yield* createSchedulerDecision(taskId, now)
              return [
                { ...state, attempts: state.attempts + 1 },
                state.attempts,
                decision
              ] as const
            })
          )
        )
      )
    }
    
    return { scheduleWithExternalSystem } as const
  })
}
```

### Testing Schedule Decision Systems

```typescript
import { ScheduleDecision, Schedule, Effect, TestClock, Duration, Interval } from "effect"

// Test utilities for schedule decisions
const createScheduleDecisionTestUtils = () => {
  // Mock time for testing
  const createMockTimeDecision = (
    currentTime: number,
    shouldContinue: boolean,
    delayMs?: number
  ): ScheduleDecision.ScheduleDecision => {
    if (!shouldContinue) {
      return ScheduleDecision.done
    }
    
    return ScheduleDecision.continueWith(
      Interval.after(currentTime + (delayMs ?? 1000))
    )
  }
  
  // Test schedule decision sequence
  const testDecisionSequence = (
    decisions: ScheduleDecision.ScheduleDecision[],
    inputs: unknown[]
  ) => Effect.gen(function* () {
    let decisionIndex = 0
    
    const testSchedule = Schedule.makeWithState(
      { index: 0 },
      (now, input, state) => Effect.gen(function* () {
        if (decisionIndex >= decisions.length) {
          return [state, input, ScheduleDecision.done] as const
        }
        
        const decision = decisions[decisionIndex]
        decisionIndex++
        
        return [
          { index: state.index + 1 },
          input,
          decision
        ] as const
      })
    )
    
    const results: unknown[] = []
    for (const input of inputs) {
      try {
        const result = yield* Effect.succeed(input).pipe(
          Effect.retry(testSchedule)
        )
        results.push(result)
      } catch (error) {
        results.push(error)
      }
    }
    
    return results
  })
  
  return {
    createMockTimeDecision,
    testDecisionSequence
  }
}

// Example test
const testScheduleDecisionBehavior = Effect.gen(function* () {
  const testUtils = createScheduleDecisionTestUtils()
  
  // Test exponential backoff behavior
  const backoffDecisions = [
    testUtils.createMockTimeDecision(1000, true, 1000),   // 1s delay
    testUtils.createMockTimeDecision(2000, true, 2000),   // 2s delay
    testUtils.createMockTimeDecision(4000, true, 4000),   // 4s delay
    testUtils.createMockTimeDecision(8000, false)         // Stop
  ]
  
  const results = yield* testUtils.testDecisionSequence(
    backoffDecisions,
    ['input1', 'input2', 'input3', 'input4']
  )
  
  yield* Effect.log(`Test results: ${JSON.stringify(results)}`)
  return results
}).pipe(
  Effect.provide(TestClock.layer)
)
```

## Conclusion

ScheduleDecision provides **precise scheduling control**, **composable decision logic**, and **type-safe timing management** for complex scheduling scenarios.

Key benefits:
- **Precise Timing Control**: Define exact intervals and timing windows for operations
- **Composable Logic**: Combine multiple decision strategies for complex scheduling requirements
- **Type Safety**: Compile-time guarantees about scheduling decisions and their consistency
- **Integration Friendly**: Works seamlessly with Effect's scheduling system and external services

ScheduleDecision is ideal when you need fine-grained control over scheduling behavior, want to implement complex retry strategies, or need to integrate with external scheduling systems while maintaining type safety and composability.