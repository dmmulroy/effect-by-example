# ScheduleInterval: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem ScheduleInterval Solves

When building time-based systems, developers often struggle with representing and manipulating time intervals precisely:

```typescript
// Traditional approach - error-prone time calculations
class TimeInterval {
  constructor(private start: number, private end: number) {}
  
  getDuration(): number {
    return this.end - this.start // What if start > end?
  }
  
  intersects(other: TimeInterval): boolean {
    // Complex logic with edge cases
    return this.start < other.end && this.end > other.start
  }
  
  union(other: TimeInterval): TimeInterval | null {
    // More complex logic, null handling
    if (!this.intersects(other)) return null
    return new TimeInterval(
      Math.min(this.start, other.start),
      Math.max(this.end, other.end)
    )
  }
}
```

This approach leads to:
- **Edge Case Bugs** - Incorrect handling when start > end
- **Complex Logic** - Manual implementation of interval operations
- **Type Unsafety** - No compile-time guarantees about validity
- **Inconsistent Behavior** - Different interpretations of empty intervals

### The ScheduleInterval Solution

ScheduleInterval provides a robust, type-safe foundation for time interval operations:

```typescript
import { ScheduleInterval, Duration } from "effect"

// Create intervals safely - automatically handles invalid ranges
const interval = ScheduleInterval.make(1000, 5000) // 1s to 5s
const emptyInterval = ScheduleInterval.make(5000, 1000) // Becomes empty interval

// Type-safe operations with clear semantics
const duration = ScheduleInterval.size(interval) // Duration.Duration
const intersection = ScheduleInterval.intersect(interval, otherInterval)
const union = ScheduleInterval.union(interval, otherInterval) // Option<Interval>
```

### Key Concepts

**Interval**: Represents a time span with start and end points in milliseconds. Invalid ranges (start > end) automatically become empty intervals.

**Empty Interval**: A zero-width interval representing no time. All invalid intervals collapse to this state.

**Interval Operations**: Type-safe operations like intersection, union, and size calculation with predictable edge case handling.

## Basic Usage Patterns

### Pattern 1: Creating Intervals

```typescript
import { ScheduleInterval } from "effect"

// Create a specific time interval
const interval = ScheduleInterval.make(1000, 5000) // 1-5 seconds

// Create an empty interval
const empty = ScheduleInterval.empty

// Create open-ended intervals
const afterNow = ScheduleInterval.after(Date.now())
const beforeMidnight = ScheduleInterval.before(Date.now() + 86400000)
```

### Pattern 2: Interval Properties

```typescript
import { ScheduleInterval, Duration } from "effect"

const interval = ScheduleInterval.make(1000, 5000)

// Check if interval is empty
const isEmpty = ScheduleInterval.isEmpty(interval) // false
const isNonEmpty = ScheduleInterval.isNonEmpty(interval) // true

// Get interval duration
const duration = ScheduleInterval.size(interval) // Duration of 4 seconds
const milliseconds = Duration.toMillis(duration) // 4000
```

### Pattern 3: Interval Comparisons and Operations

```typescript
import { ScheduleInterval } from "effect"

const interval1 = ScheduleInterval.make(1000, 3000)
const interval2 = ScheduleInterval.make(2000, 4000)

// Compare intervals
const isLess = ScheduleInterval.lessThan(interval1, interval2) // true
const minimum = ScheduleInterval.min(interval1, interval2) // interval1
const maximum = ScheduleInterval.max(interval1, interval2) // interval2

// Combine intervals
const intersection = ScheduleInterval.intersect(interval1, interval2) // [2000, 3000]
const union = ScheduleInterval.union(interval1, interval2) // Some([1000, 4000])
```

## Real-World Examples

### Example 1: Monitoring System Downtime Windows

Track and analyze system maintenance windows and their overlaps:

```typescript
import { Effect, ScheduleInterval, Duration, Array as Arr, Console } from "effect"

interface MaintenanceWindow {
  readonly id: string
  readonly system: string
  readonly interval: ScheduleInterval.Interval
}

const createMaintenanceWindow = (
  id: string,
  system: string,
  startTime: Date,
  durationHours: number
): MaintenanceWindow => {
  const startMillis = startTime.getTime()
  const endMillis = startMillis + (durationHours * 60 * 60 * 1000)
  
  return {
    id,
    system,
    interval: ScheduleInterval.make(startMillis, endMillis)
  }
}

const findOverlappingWindows = (windows: MaintenanceWindow[]) => {
  return Effect.gen(function* () {
    const overlaps: Array<{
      window1: MaintenanceWindow
      window2: MaintenanceWindow
      overlapDuration: Duration.Duration
    }> = []
    
    for (let i = 0; i < windows.length; i++) {
      for (let j = i + 1; j < windows.length; j++) {
        const intersection = ScheduleInterval.intersect(
          windows[i].interval,
          windows[j].interval
        )
        
        if (ScheduleInterval.isNonEmpty(intersection)) {
          const overlapDuration = ScheduleInterval.size(intersection)
          overlaps.push({
            window1: windows[i],
            window2: windows[j],
            overlapDuration
          })
        }
      }
    }
    
    return overlaps
  })
}

const analyzeMaintenanceSchedule = Effect.gen(function* () {
  const windows = [
    createMaintenanceWindow("MW1", "Database", new Date("2024-03-15T02:00:00Z"), 4),
    createMaintenanceWindow("MW2", "API Server", new Date("2024-03-15T04:00:00Z"), 2),
    createMaintenanceWindow("MW3", "Cache", new Date("2024-03-15T03:00:00Z"), 3)
  ]
  
  const overlaps = yield* findOverlappingWindows(windows)
  
  for (const overlap of overlaps) {
    yield* Console.log(
      `Conflict: ${overlap.window1.system} and ${overlap.window2.system} ` +
      `overlap for ${Duration.toMillis(overlap.overlapDuration)}ms`
    )
  }
  
  // Calculate total maintenance time (union of all intervals)
  const totalWindow = windows.reduce((acc, window) => {
    if (!acc) return window.interval
    
    const unionResult = ScheduleInterval.union(acc, window.interval)
    // If intervals don't overlap, we need to handle this differently
    // For simplicity, using the outer bounds
    return ScheduleInterval.make(
      Math.min(acc.startMillis, window.interval.startMillis),
      Math.max(acc.endMillis, window.interval.endMillis)
    )
  }, null as ScheduleInterval.Interval | null)
  
  if (totalWindow) {
    const totalDuration = ScheduleInterval.size(totalWindow)
    yield* Console.log(`Total maintenance window: ${Duration.toHours(totalDuration)} hours`)
  }
})
```

### Example 2: Rate Limiting with Time Windows

Implement a sliding window rate limiter using intervals:

```typescript
import { 
  Effect, 
  ScheduleInterval, 
  Duration, 
  Clock, 
  Ref, 
  Array as Arr 
} from "effect"

interface RateLimitEntry {
  readonly timestamp: number
  readonly interval: ScheduleInterval.Interval
}

interface RateLimiter {
  readonly checkLimit: (requestCount: number) => Effect.Effect<boolean>
  readonly getCurrentUsage: () => Effect.Effect<number>
}

const createSlidingWindowRateLimiter = (
  maxRequests: number,
  windowDurationMs: number
): Effect.Effect<RateLimiter> => {
  return Effect.gen(function* () {
    const requestLog = yield* Ref.make<RateLimitEntry[]>([])
    
    const checkLimit = (requestCount: number) => {
      return Effect.gen(function* () {
        const now = yield* Clock.currentTimeMillis
        const windowStart = now - windowDurationMs
        const requestInterval = ScheduleInterval.make(windowStart, now)
        
        // Clean up old entries and count current usage
        const currentEntries = yield* Ref.get(requestLog)
        const activeEntries = currentEntries.filter(entry => {
          const intersection = ScheduleInterval.intersect(entry.interval, requestInterval)
          return ScheduleInterval.isNonEmpty(intersection)
        })
        
        const currentUsage = activeEntries.length
        const wouldExceedLimit = currentUsage + requestCount > maxRequests
        
        if (!wouldExceedLimit) {
          // Add new request entries
          const newEntries = Array.from({ length: requestCount }, () => ({
            timestamp: now,
            interval: ScheduleInterval.make(now, now + 1) // Point-in-time request
          }))
          
          yield* Ref.update(requestLog, entries => 
            [...activeEntries, ...newEntries]
          )
        }
        
        return !wouldExceedLimit
      })
    }
    
    const getCurrentUsage = () => {
      return Effect.gen(function* () {
        const now = yield* Clock.currentTimeMillis
        const windowStart = now - windowDurationMs
        const windowInterval = ScheduleInterval.make(windowStart, now)
        
        const entries = yield* Ref.get(requestLog)
        return entries.filter(entry => {
          const intersection = ScheduleInterval.intersect(entry.interval, windowInterval)
          return ScheduleInterval.isNonEmpty(intersection)
        }).length
      })
    }
    
    return { checkLimit, getCurrentUsage }
  })
}

// Usage example
const rateLimitingExample = Effect.gen(function* () {
  // Allow 10 requests per 60 seconds
  const rateLimiter = yield* createSlidingWindowRateLimiter(10, 60000)
  
  // Simulate API requests
  const allowed1 = yield* rateLimiter.checkLimit(5) // Should be true
  const usage1 = yield* rateLimiter.getCurrentUsage() // Should be 5
  
  const allowed2 = yield* rateLimiter.checkLimit(6) // Should be false (would exceed limit)
  const usage2 = yield* rateLimiter.getCurrentUsage() // Should still be 5
  
  return { allowed1, usage1, allowed2, usage2 }
})
```

### Example 3: Meeting Scheduler with Conflict Detection

Build a meeting scheduler that detects conflicts and suggests alternatives:

```typescript
import { 
  Effect, 
  ScheduleInterval, 
  Duration, 
  Array as Arr, 
  Option, 
  Console 
} from "effect"

interface Meeting {
  readonly id: string
  readonly title: string
  readonly attendees: string[]
  readonly interval: ScheduleInterval.Interval
}

interface TimeSlot {
  readonly interval: ScheduleInterval.Interval
  readonly isAvailable: boolean
}

const createMeeting = (
  id: string,
  title: string,
  attendees: string[],
  startTime: Date,
  durationMinutes: number
): Meeting => {
  const startMillis = startTime.getTime()
  const endMillis = startMillis + (durationMinutes * 60 * 1000)
  
  return {
    id,
    title,
    attendees,
    interval: ScheduleInterval.make(startMillis, endMillis)
  }
}

const findMeetingConflicts = (
  existingMeetings: Meeting[],
  newMeeting: Meeting
) => {
  return Effect.gen(function* () {
    const conflicts = existingMeetings.filter(meeting => {
      const intersection = ScheduleInterval.intersect(meeting.interval, newMeeting.interval)
      return ScheduleInterval.isNonEmpty(intersection)
    })
    
    return conflicts
  })
}

const findAvailableTimeSlots = (
  existingMeetings: Meeting[],
  searchStart: Date,
  searchEnd: Date,
  meetingDurationMinutes: number
) => {
  return Effect.gen(function* () {
    const searchInterval = ScheduleInterval.make(
      searchStart.getTime(),
      searchEnd.getTime()
    )
    
    const meetingDurationMs = meetingDurationMinutes * 60 * 1000
    const availableSlots: TimeSlot[] = []
    
    // Sort meetings by start time
    const sortedMeetings = existingMeetings
      .filter(meeting => {
        const intersection = ScheduleInterval.intersect(meeting.interval, searchInterval)
        return ScheduleInterval.isNonEmpty(intersection)
      })
      .sort((a, b) => a.interval.startMillis - b.interval.startMillis)
    
    let currentTime = searchStart.getTime()
    
    for (const meeting of sortedMeetings) {
      // Check if there's a gap before this meeting
      if (currentTime + meetingDurationMs <= meeting.interval.startMillis) {
        const slotEnd = Math.min(meeting.interval.startMillis, currentTime + meetingDurationMs)
        availableSlots.push({
          interval: ScheduleInterval.make(currentTime, slotEnd),
          isAvailable: true
        })
      }
      
      // Add the occupied slot
      availableSlots.push({
        interval: meeting.interval,
        isAvailable: false
      })
      
      currentTime = Math.max(currentTime, meeting.interval.endMillis)
    }
    
    // Check for availability after the last meeting
    if (currentTime + meetingDurationMs <= searchEnd.getTime()) {
      availableSlots.push({
        interval: ScheduleInterval.make(currentTime, searchEnd.getTime()),
        isAvailable: true
      })
    }
    
    return availableSlots.filter(slot => slot.isAvailable)
  })
}

// Example usage
const meetingSchedulerExample = Effect.gen(function* () {
  const existingMeetings = [
    createMeeting("M1", "Team Standup", ["alice", "bob"], new Date("2024-03-15T09:00:00Z"), 30),
    createMeeting("M2", "Client Call", ["alice", "charlie"], new Date("2024-03-15T11:00:00Z"), 60),
    createMeeting("M3", "Code Review", ["bob", "dave"], new Date("2024-03-15T14:00:00Z"), 45)
  ]
  
  const newMeeting = createMeeting(
    "M4", 
    "Planning Session", 
    ["alice", "bob", "charlie"], 
    new Date("2024-03-15T10:30:00Z"), 
    90
  )
  
  // Check for conflicts
  const conflicts = yield* findMeetingConflicts(existingMeetings, newMeeting)
  
  if (conflicts.length > 0) {
    yield* Console.log(`Meeting conflicts detected with: ${conflicts.map(m => m.title).join(", ")}`)
    
    // Find alternative time slots
    const availableSlots = yield* findAvailableTimeSlots(
      existingMeetings,
      new Date("2024-03-15T08:00:00Z"),
      new Date("2024-03-15T18:00:00Z"),
      90
    )
    
    yield* Console.log(`Found ${availableSlots.length} available time slots`)
    
    for (const slot of availableSlots.slice(0, 3)) { // Show first 3 options
      const duration = ScheduleInterval.size(slot.interval)
      const durationMinutes = Duration.toMillis(duration) / (60 * 1000)
      yield* Console.log(
        `Available: ${new Date(slot.interval.startMillis).toISOString()} ` +
        `(${durationMinutes} minutes)`
      )
    }
  } else {
    yield* Console.log("No conflicts found - meeting can be scheduled")
  }
})
```

## Advanced Features Deep Dive

### Feature 1: Interval Set Operations

ScheduleInterval provides mathematical set operations for complex time manipulation:

#### Basic Set Operations

```typescript
import { ScheduleInterval, Option } from "effect"

const interval1 = ScheduleInterval.make(1000, 5000)
const interval2 = ScheduleInterval.make(3000, 7000)

// Intersection - time period common to both intervals
const intersection = ScheduleInterval.intersect(interval1, interval2)
// Result: [3000, 5000]

// Union - combined time period (if intervals overlap or touch)
const unionResult = ScheduleInterval.union(interval1, interval2)
// Result: Some([1000, 7000])

// Non-overlapping intervals return None
const nonOverlapping1 = ScheduleInterval.make(1000, 2000)
const nonOverlapping2 = ScheduleInterval.make(3000, 4000)
const noUnion = ScheduleInterval.union(nonOverlapping1, nonOverlapping2)
// Result: None
```

#### Real-World Set Operations Example

```typescript
import { Effect, ScheduleInterval, Option, Array as Arr } from "effect"

interface BusinessHours {
  readonly day: string
  readonly intervals: ScheduleInterval.Interval[]
}

const mergeOverlappingIntervals = (intervals: ScheduleInterval.Interval[]) => {
  return Effect.gen(function* () {
    if (intervals.length === 0) return []
    
    // Sort intervals by start time
    const sorted = intervals.sort((a, b) => a.startMillis - b.startMillis)
    const merged: ScheduleInterval.Interval[] = [sorted[0]]
    
    for (let i = 1; i < sorted.length; i++) {
      const current = sorted[i]
      const lastMerged = merged[merged.length - 1]
      
      const unionResult = ScheduleInterval.union(lastMerged, current)
      
      if (Option.isSome(unionResult)) {
        // Intervals can be merged
        merged[merged.length - 1] = unionResult.value
      } else {
        // Intervals don't overlap
        merged.push(current)
      }
    }
    
    return merged
  })
}

const calculateTotalBusinessHours = (businessHours: BusinessHours[]) => {
  return Effect.gen(function* () {
    let totalMilliseconds = 0
    
    for (const day of businessHours) {
      const mergedIntervals = yield* mergeOverlappingIntervals(day.intervals)
      
      for (const interval of mergedIntervals) {
        const duration = ScheduleInterval.size(interval)
        totalMilliseconds += duration.millis
      }
    }
    
    return totalMilliseconds / (1000 * 60 * 60) // Convert to hours
  })
}
```

### Feature 2: Temporal Boundaries and Edge Cases

Handle edge cases and boundary conditions in time-based operations:

#### Infinite Intervals

```typescript
import { ScheduleInterval } from "effect"

// Create intervals extending to infinity
const afterMidnight = ScheduleInterval.after(Date.now())
const beforeDeadline = ScheduleInterval.before(Date.now() + 86400000)

// These intervals have special properties
const isAfterEmpty = ScheduleInterval.isEmpty(afterMidnight) // false
const afterSize = ScheduleInterval.size(afterMidnight) // Infinite duration

// Intersections with infinite intervals
const finite = ScheduleInterval.make(1000, 5000)
const intersection = ScheduleInterval.intersect(afterMidnight, finite)
// Result depends on the relative positions
```

#### Edge Case Handler

```typescript
import { Effect, ScheduleInterval, Duration, Console } from "effect"

const safeIntervalOperation = <T>(
  operation: () => T,
  fallback: T,
  description: string
) => {
  return Effect.gen(function* () {
    try {
      const result = operation()
      yield* Console.log(`${description}: Success`)
      return result
    } catch (error) {
      yield* Console.log(`${description}: Using fallback due to error: ${error}`)
      return fallback
    }
  })
}

const robustIntervalProcessing = (
  intervals: ScheduleInterval.Interval[]
) => {
  return Effect.gen(function* () {
    const results = []
    
    for (const interval of intervals) {
      // Handle potentially problematic size calculations
      const size = yield* safeIntervalOperation(
        () => ScheduleInterval.size(interval),
        Duration.zero,
        "Size calculation"
      )
      
      // Handle empty interval checks
      const isEmpty = yield* safeIntervalOperation(
        () => ScheduleInterval.isEmpty(interval),
        true,
        "Empty check"
      )
      
      results.push({ interval, size, isEmpty })
    }
    
    return results
  })
}
```

### Feature 3: Precision and Performance Considerations

Optimize interval operations for high-frequency usage:

#### High-Performance Interval Cache

```typescript
import { Effect, ScheduleInterval, Duration, Ref, HashMap } from "effect"

interface IntervalCache {
  readonly get: (key: string) => Effect.Effect<Option.Option<ScheduleInterval.Interval>>
  readonly set: (key: string, interval: ScheduleInterval.Interval) => Effect.Effect<void>
  readonly clear: () => Effect.Effect<void>
}

const createIntervalCache = (maxSize: number): Effect.Effect<IntervalCache> => {
  return Effect.gen(function* () {
    const cache = yield* Ref.make(HashMap.empty<string, ScheduleInterval.Interval>())
    const accessOrder = yield* Ref.make<string[]>([])
    
    const evictIfNeeded = Effect.gen(function* () {
      const currentCache = yield* Ref.get(cache)
      const currentOrder = yield* Ref.get(accessOrder)
      
      if (HashMap.size(currentCache) >= maxSize && currentOrder.length > 0) {
        const oldestKey = currentOrder[0]
        yield* Ref.update(cache, HashMap.remove(oldestKey))
        yield* Ref.update(accessOrder, order => order.slice(1))
      }
    })
    
    const get = (key: string) => {
      return Effect.gen(function* () {
        const currentCache = yield* Ref.get(cache)
        const result = HashMap.get(currentCache, key)
        
        if (Option.isSome(result)) {
          // Update access order
          yield* Ref.update(accessOrder, order => 
            [key, ...order.filter(k => k !== key)]
          )
        }
        
        return result
      })
    }
    
    const set = (key: string, interval: ScheduleInterval.Interval) => {
      return Effect.gen(function* () {
        yield* evictIfNeeded
        yield* Ref.update(cache, HashMap.set(key, interval))
        yield* Ref.update(accessOrder, order => 
          [key, ...order.filter(k => k !== key)]
        )
      })
    }
    
    const clear = () => {
      return Effect.gen(function* () {
        yield* Ref.set(cache, HashMap.empty())
        yield* Ref.set(accessOrder, [])
      })
    }
    
    return { get, set, clear }
  })
}
```

## Practical Patterns & Best Practices

### Pattern 1: Interval Validation and Normalization

```typescript
import { Effect, ScheduleInterval, Duration } from "effect"

const validateInterval = (
  startTime: Date,
  endTime: Date
): Effect.Effect<ScheduleInterval.Interval, string> => {
  return Effect.gen(function* () {
    const startMillis = startTime.getTime()
    const endMillis = endTime.getTime()
    
    if (startMillis > endMillis) {
      return yield* Effect.fail("Start time must be before end time")
    }
    
    if (startMillis < 0 || endMillis < 0) {
      return yield* Effect.fail("Times must be positive")
    }
    
    const interval = ScheduleInterval.make(startMillis, endMillis)
    
    // Validate minimum duration
    const duration = ScheduleInterval.size(interval)
    if (Duration.lessThan(duration, Duration.seconds(1))) {
      return yield* Effect.fail("Interval must be at least 1 second")
    }
    
    return interval
  })
}

const normalizeIntervalToBoundaries = (
  interval: ScheduleInterval.Interval,
  boundaryDuration: Duration.Duration
) => {
  const boundaryMs = Duration.toMillis(boundaryDuration)
  
  // Align start to boundary
  const alignedStart = Math.floor(interval.startMillis / boundaryMs) * boundaryMs
  
  // Align end to boundary (round up)
  const alignedEnd = Math.ceil(interval.endMillis / boundaryMs) * boundaryMs
  
  return ScheduleInterval.make(alignedStart, alignedEnd)
}
```

### Pattern 2: Interval Arithmetic Helpers

```typescript
import { ScheduleInterval, Duration } from "effect"

const IntervalArithmetic = {
  // Extend interval by duration
  extend: (interval: ScheduleInterval.Interval, duration: Duration.Duration) => {
    const extensionMs = Duration.toMillis(duration)
    return ScheduleInterval.make(
      interval.startMillis - extensionMs,
      interval.endMillis + extensionMs
    )
  },
  
  // Shrink interval by duration
  shrink: (interval: ScheduleInterval.Interval, duration: Duration.Duration) => {
    const shrinkMs = Duration.toMillis(duration)
    return ScheduleInterval.make(
      interval.startMillis + shrinkMs,
      interval.endMillis - shrinkMs
    )
  },
  
  // Shift interval by duration
  shift: (interval: ScheduleInterval.Interval, duration: Duration.Duration) => {
    const shiftMs = Duration.toMillis(duration)
    return ScheduleInterval.make(
      interval.startMillis + shiftMs,
      interval.endMillis + shiftMs
    )
  },
  
  // Split interval into equal parts
  split: (interval: ScheduleInterval.Interval, parts: number) => {
    if (parts <= 0) return []
    
    const totalDuration = ScheduleInterval.size(interval)
    const partDuration = Duration.divide(totalDuration, parts)
    const partMs = Duration.toMillis(partDuration)
    
    const result: ScheduleInterval.Interval[] = []
    
    for (let i = 0; i < parts; i++) {
      const start = interval.startMillis + (i * partMs)
      const end = Math.min(start + partMs, interval.endMillis)
      result.push(ScheduleInterval.make(start, end))
    }
    
    return result
  }
}
```

### Pattern 3: Interval Collection Operations

```typescript
import { Effect, ScheduleInterval, Array as Arr, Option } from "effect"

const IntervalCollection = {
  // Find gaps between intervals
  findGaps: (intervals: ScheduleInterval.Interval[]) => {
    return Effect.gen(function* () {
      if (intervals.length === 0) return []
      
      const sorted = intervals.sort((a, b) => a.startMillis - b.startMillis)
      const gaps: ScheduleInterval.Interval[] = []
      
      for (let i = 0; i < sorted.length - 1; i++) {
        const current = sorted[i]
        const next = sorted[i + 1]
        
        if (current.endMillis < next.startMillis) {
          gaps.push(ScheduleInterval.make(current.endMillis, next.startMillis))
        }
      }
      
      return gaps
    })
  },
  
  // Merge overlapping intervals
  merge: (intervals: ScheduleInterval.Interval[]) => {
    return Effect.gen(function* () {
      if (intervals.length === 0) return []
      
      const sorted = intervals.sort((a, b) => a.startMillis - b.startMillis)
      const merged: ScheduleInterval.Interval[] = [sorted[0]]
      
      for (let i = 1; i < sorted.length; i++) {
        const current = sorted[i]
        const lastMerged = merged[merged.length - 1]
        
        if (current.startMillis <= lastMerged.endMillis) {
          // Overlapping - merge them
          merged[merged.length - 1] = ScheduleInterval.make(
            lastMerged.startMillis,
            Math.max(lastMerged.endMillis, current.endMillis)
          )
        } else {
          // No overlap - add as new interval
          merged.push(current)
        }
      }
      
      return merged
    })
  },
  
  // Find the interval containing a specific time
  findContaining: (intervals: ScheduleInterval.Interval[], timestamp: number) => {
    return intervals.find(interval => 
      timestamp >= interval.startMillis && timestamp < interval.endMillis
    )
  }
}
```

## Integration Examples

### Integration with Schedule Module

```typescript
import { Effect, Schedule, ScheduleInterval, Duration, Console, Clock } from "effect"

// Create a schedule that runs within specific time intervals
const createIntervalBasedSchedule = (
  allowedInterval: ScheduleInterval.Interval,
  baseSchedule: Schedule.Schedule<number>
) => {
  return Schedule.check(baseSchedule, (_, elapsed) => {
    return Effect.gen(function* () {
      const now = yield* Clock.currentTimeMillis
      const currentTime = now - Duration.toMillis(elapsed)
      
      return currentTime >= allowedInterval.startMillis && 
             currentTime <= allowedInterval.endMillis
    })
  })
}

// Example: Retry only during business hours
const businessHoursRetry = Effect.gen(function* () {
  const now = new Date()
  const startOfDay = new Date(now.getFullYear(), now.getMonth(), now.getDate(), 9, 0, 0)
  const endOfDay = new Date(now.getFullYear(), now.getMonth(), now.getDate(), 17, 0, 0)
  
  const businessHours = ScheduleInterval.make(
    startOfDay.getTime(),
    endOfDay.getTime()
  )
  
  const businessHoursSchedule = createIntervalBasedSchedule(
    businessHours,
    Schedule.exponential("1 seconds").pipe(Schedule.recurs(5))
  )
  
  const failingOperation = Effect.fail("Temporary failure")
  
  return yield* failingOperation.pipe(
    Effect.retry(businessHoursSchedule),
    Effect.catchAll(error => 
      Console.log(`Failed after business hours retries: ${error}`)
    )
  )
})
```

### Integration with Duration and Clock

```typescript
import { Effect, ScheduleInterval, Duration, Clock, Console } from "effect"

const createTimerWithInterval = (interval: ScheduleInterval.Interval) => {
  return Effect.gen(function* () {
    const startTime = yield* Clock.currentTimeMillis
    const targetStart = interval.startMillis
    const targetEnd = interval.endMillis
    
    // Wait until interval starts
    if (startTime < targetStart) {
      const waitDuration = Duration.millis(targetStart - startTime)
      yield* Console.log(`Waiting ${Duration.toMillis(waitDuration)}ms for interval to start`)
      yield* Effect.sleep(waitDuration)
    }
    
    // Check if we're still within the interval
    const currentTime = yield* Clock.currentTimeMillis
    if (currentTime >= targetEnd) {
      return yield* Effect.fail("Interval has already passed")
    }
    
    // Calculate remaining time in interval
    const remainingTime = Duration.millis(targetEnd - currentTime)
    yield* Console.log(`Operating within interval for ${Duration.toMillis(remainingTime)}ms`)
    
    return {
      interval,
      startTime: currentTime,
      remainingTime
    }
  })
}

// Real-world example: Batch processing within maintenance windows
const batchProcessWithinWindow = (
  processingInterval: ScheduleInterval.Interval,
  batchSize: number
) => {
  return Effect.gen(function* () {
    const timer = yield* createTimerWithInterval(processingInterval)
    const processedItems: string[] = []
    
    // Simulate processing items
    while (processedItems.length < batchSize) {
      const currentTime = yield* Clock.currentTimeMillis
      
      if (currentTime >= processingInterval.endMillis) {
        yield* Console.log("Processing window expired")
        break
      }
      
      // Simulate processing one item
      yield* Effect.sleep(Duration.millis(100))
      processedItems.push(`item-${processedItems.length + 1}`)
      
      if (processedItems.length % 10 === 0) {
        yield* Console.log(`Processed ${processedItems.length} items`)
      }
    }
    
    return {
      processedCount: processedItems.length,
      completedWithinWindow: processedItems.length === batchSize
    }
  })
}
```

### Testing Strategies

```typescript
import { Effect, ScheduleInterval, Duration, TestClock, TestContext } from "effect"

const testIntervalOperations = Effect.gen(function* () {
  // Test interval creation and validation
  const testInterval = ScheduleInterval.make(1000, 5000)
  const size = ScheduleInterval.size(testInterval)
  
  console.assert(Duration.toMillis(size) === 4000, "Interval size should be 4000ms")
  console.assert(!ScheduleInterval.isEmpty(testInterval), "Interval should not be empty")
  
  // Test invalid interval handling
  const invalidInterval = ScheduleInterval.make(5000, 1000)
  console.assert(ScheduleInterval.isEmpty(invalidInterval), "Invalid interval should be empty")
  
  // Test interval operations
  const interval1 = ScheduleInterval.make(1000, 3000)
  const interval2 = ScheduleInterval.make(2000, 4000)
  
  const intersection = ScheduleInterval.intersect(interval1, interval2)
  const intersectionSize = ScheduleInterval.size(intersection)
  console.assert(Duration.toMillis(intersectionSize) === 1000, "Intersection should be 1000ms")
  
  yield* Effect.succeed("All interval tests passed")
})

// Property-based testing helper
const generateRandomInterval = (minStart: number, maxEnd: number) => {
  return Effect.gen(function* () {
    const start = Math.floor(Math.random() * (maxEnd - minStart)) + minStart
    const end = Math.floor(Math.random() * (maxEnd - start)) + start
    return ScheduleInterval.make(start, end)
  })
}

const propertyBasedIntervalTest = Effect.gen(function* () {
  // Test that interval intersection is commutative
  for (let i = 0; i < 100; i++) {
    const interval1 = yield* generateRandomInterval(0, 10000)
    const interval2 = yield* generateRandomInterval(0, 10000)
    
    const intersection1 = ScheduleInterval.intersect(interval1, interval2)
    const intersection2 = ScheduleInterval.intersect(interval2, interval1)
    
    console.assert(
      intersection1.startMillis === intersection2.startMillis &&
      intersection1.endMillis === intersection2.endMillis,
      "Intersection should be commutative"
    )
  }
  
  yield* Effect.succeed("Property-based tests passed")
})
```

## Conclusion

ScheduleInterval provides a robust foundation for time interval operations in Effect applications. It eliminates common time-handling bugs through type-safe operations and predictable edge case handling.

Key benefits:
- **Type Safety**: Compile-time guarantees about interval validity and operations
- **Edge Case Handling**: Automatic handling of invalid ranges and boundary conditions  
- **Composability**: Clean integration with Duration, Schedule, and other Effect modules
- **Performance**: Efficient operations suitable for high-frequency usage

ScheduleInterval is essential when building scheduling systems, rate limiters, monitoring tools, or any application requiring precise time interval management. Its mathematical approach to set operations makes complex temporal logic both readable and reliable.