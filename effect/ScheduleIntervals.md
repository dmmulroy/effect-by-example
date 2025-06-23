# ScheduleIntervals: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem ScheduleIntervals Solves

Managing complex scheduling scenarios often requires working with multiple time intervals simultaneously. Traditional approaches lead to:

```typescript
// Traditional approach - managing intervals manually
interface TimeInterval {
  start: number
  end: number
}

function combineIntervals(intervals1: TimeInterval[], intervals2: TimeInterval[]): TimeInterval[] {
  // Complex, error-prone logic for merging intervals
  const combined = [...intervals1, ...intervals2]
  const sorted = combined.sort((a, b) => a.start - b.start)
  
  // Manual merging logic - prone to off-by-one errors
  const merged: TimeInterval[] = []
  for (const interval of sorted) {
    if (merged.length === 0 || merged[merged.length - 1].end < interval.start) {
      merged.push(interval)
    } else {
      merged[merged.length - 1].end = Math.max(merged[merged.length - 1].end, interval.end)
    }
  }
  return merged
}

function findIntersections(intervals1: TimeInterval[], intervals2: TimeInterval[]): TimeInterval[] {
  // Even more complex manual logic
  // Risk of bugs, hard to maintain, not type-safe
}
```

This approach leads to:
- **Complex Manual Logic** - Hand-written interval merging and intersection algorithms
- **Error-Prone Operations** - Off-by-one errors, edge cases, and overlapping interval bugs
- **Poor Composability** - Hard to combine multiple interval sets in complex scheduling scenarios
- **No Type Safety** - Plain objects don't provide guarantees about interval validity

### The ScheduleIntervals Solution

ScheduleIntervals provides a type-safe, composable way to work with collections of time intervals:

```typescript
import { ScheduleIntervals, ScheduleInterval, Chunk } from "effect"

// Create intervals from individual ScheduleInterval instances
const businessHours = ScheduleIntervals.fromIterable([
  ScheduleInterval.make(9 * 60 * 60 * 1000, 12 * 60 * 60 * 1000), // 9-12 AM
  ScheduleInterval.make(13 * 60 * 60 * 1000, 17 * 60 * 60 * 1000)  // 1-5 PM
])

const maintenanceWindows = ScheduleIntervals.fromIterable([
  ScheduleInterval.make(2 * 60 * 60 * 1000, 4 * 60 * 60 * 1000),   // 2-4 AM
  ScheduleInterval.make(22 * 60 * 60 * 1000, 24 * 60 * 60 * 1000)  // 10 PM-12 AM
])

// Combine intervals with type-safe union
const allAvailableTime = businessHours.pipe(
  ScheduleIntervals.union(maintenanceWindows)
)

// Find overlapping periods with intersection
const conflictingTimes = businessHours.pipe(
  ScheduleIntervals.intersect(maintenanceWindows)
)
```

### Key Concepts

**Intervals Collection**: A `ScheduleIntervals` contains a `Chunk` of `ScheduleInterval` instances, representing multiple time periods that can be operated on as a unit.

**Union Operations**: Combines two interval collections, merging overlapping periods automatically and maintaining chronological order.

**Intersection Operations**: Finds the overlapping portions between two interval collections, useful for finding conflicts or common availability.

**Immutable Operations**: All operations return new `ScheduleIntervals` instances, ensuring referential transparency and safe concurrent usage.

## Basic Usage Patterns

### Pattern 1: Creating Interval Collections

```typescript
import { ScheduleIntervals, ScheduleInterval, Chunk } from "effect"

// Empty intervals collection
const noSchedule = ScheduleIntervals.empty

// From a single interval
const singleInterval = ScheduleIntervals.make(
  Chunk.of(ScheduleInterval.make(1000, 5000))
)

// From multiple intervals
const workingHours = ScheduleIntervals.fromIterable([
  ScheduleInterval.make(9 * 3600 * 1000, 12 * 3600 * 1000),  // Morning
  ScheduleInterval.make(13 * 3600 * 1000, 17 * 3600 * 1000)  // Afternoon
])

// From a Chunk of intervals
const intervals = Chunk.fromIterable([
  ScheduleInterval.make(1000, 2000),
  ScheduleInterval.make(3000, 4000),
  ScheduleInterval.make(5000, 6000)
])
const scheduledIntervals = ScheduleIntervals.make(intervals)
```

### Pattern 2: Combining Collections

```typescript
// Union - combines two collections, merging overlapping intervals
const schedule1 = ScheduleIntervals.fromIterable([
  ScheduleInterval.make(1000, 3000),
  ScheduleInterval.make(5000, 7000)
])

const schedule2 = ScheduleIntervals.fromIterable([
  ScheduleInterval.make(2000, 4000),
  ScheduleInterval.make(6000, 8000)
])

// Results in merged intervals: [1000-4000], [5000-8000]
const combined = schedule1.pipe(
  ScheduleIntervals.union(schedule2)
)
```

### Pattern 3: Finding Intersections

```typescript
// Intersection - finds overlapping periods
const availableSlots = ScheduleIntervals.fromIterable([
  ScheduleInterval.make(9 * 3600 * 1000, 17 * 3600 * 1000)  // 9 AM - 5 PM
])

const requestedTimes = ScheduleIntervals.fromIterable([
  ScheduleInterval.make(8 * 3600 * 1000, 10 * 3600 * 1000), // 8-10 AM
  ScheduleInterval.make(14 * 3600 * 1000, 18 * 3600 * 1000) // 2-6 PM
])

// Results in: [9-10 AM], [2-5 PM]
const bookableSlots = availableSlots.pipe(
  ScheduleIntervals.intersect(requestedTimes)
)
```

## Real-World Examples

### Example 1: Meeting Room Booking System

Managing meeting room availability across multiple booking requests:

```typescript
import { Effect, ScheduleIntervals, ScheduleInterval, DateTime } from "effect"

interface MeetingRequest {
  readonly id: string
  readonly title: string
  readonly startTime: DateTime.DateTime
  readonly endTime: DateTime.DateTime
  readonly roomId: string
}

interface RoomAvailability {
  readonly roomId: string
  readonly availableSlots: ScheduleIntervals.Intervals
  readonly bookedSlots: ScheduleIntervals.Intervals
}

// Helper to convert DateTime to milliseconds
const dateTimeToMillis = (dt: DateTime.DateTime): number =>
  DateTime.toEpochMillis(dt)

const createMeetingBookingService = () => {
  return Effect.gen(function* () {
    // Define business hours (9 AM - 6 PM, Monday-Friday)
    const createBusinessHours = (date: DateTime.DateTime): ScheduleIntervals.Intervals => {
      const startOfDay = DateTime.startOfDay(date)
      const businessStart = DateTime.addHours(startOfDay, 9)
      const businessEnd = DateTime.addHours(startOfDay, 18)
      
      return ScheduleIntervals.fromIterable([
        ScheduleInterval.make(
          dateTimeToMillis(businessStart),
          dateTimeToMillis(businessEnd)
        )
      ])
    }

    const findAvailableSlots = (
      roomAvailability: RoomAvailability,
      requestedSlot: ScheduleIntervals.Intervals
    ): Effect.Effect<ScheduleIntervals.Intervals> => {
      return Effect.gen(function* () {
        // Remove already booked times from available slots
        const actuallyAvailable = roomAvailability.availableSlots
        
        // Find intersection with requested time
        const potentialSlots = actuallyAvailable.pipe(
          ScheduleIntervals.intersect(requestedSlot)
        )
        
        // Remove any conflicts with existing bookings
        const availableSlots = potentialSlots.pipe(
          ScheduleIntervals.intersect(
            // Invert booked slots by finding gaps (simplified for example)
            roomAvailability.availableSlots
          )
        )
        
        return availableSlots
      })
    }

    const bookMeeting = (
      request: MeetingRequest,
      roomAvailability: RoomAvailability
    ): Effect.Effect<boolean, string> => {
      return Effect.gen(function* () {
        const requestedInterval = ScheduleIntervals.fromIterable([
          ScheduleInterval.make(
            dateTimeToMillis(request.startTime),
            dateTimeToMillis(request.endTime)
          )
        ])

        const availableSlots = yield* findAvailableSlots(
          roomAvailability,
          requestedInterval
        )

        if (!ScheduleIntervals.isNonEmpty(availableSlots)) {
          yield* Effect.fail(`No availability for room ${request.roomId} at requested time`)
        }

        // Check if the requested time fits entirely within available slots
        const requestStart = ScheduleIntervals.start(requestedInterval)
        const requestEnd = ScheduleIntervals.end(requestedInterval)
        const availableStart = ScheduleIntervals.start(availableSlots)
        const availableEnd = ScheduleIntervals.end(availableSlots)

        if (requestStart < availableStart || requestEnd > availableEnd) {
          yield* Effect.fail("Requested time partially conflicts with existing bookings")
        }

        return true
      })
    }

    return { findAvailableSlots, bookMeeting, createBusinessHours }
  })
}
```

### Example 2: Distributed System Maintenance Windows

Coordinating maintenance across multiple services with complex scheduling requirements:

```typescript
import { Effect, ScheduleIntervals, ScheduleInterval, Duration, DateTime } from "effect"

interface ServiceMaintenanceConfig {
  readonly serviceName: string
  readonly maintenanceWindows: ScheduleIntervals.Intervals
  readonly criticalityLevel: "low" | "medium" | "high"
  readonly dependencies: readonly string[]
}

interface MaintenanceScheduler {
  readonly scheduleSystemMaintenance: (
    services: readonly ServiceMaintenanceConfig[]
  ) => Effect.Effect<ScheduleIntervals.Intervals, string>
}

const createMaintenanceScheduler = (): Effect.Effect<MaintenanceScheduler> => {
  return Effect.gen(function* () {
    
    const findGlobalMaintenanceWindows = (
      services: readonly ServiceMaintenanceConfig[]
    ): Effect.Effect<ScheduleIntervals.Intervals, string> => {
      return Effect.gen(function* () {
        if (services.length === 0) {
          return ScheduleIntervals.empty
        }

        // Start with the first service's windows
        let globalWindows = services[0].maintenanceWindows

        // Find intersection of all service maintenance windows
        for (const service of services.slice(1)) {
          globalWindows = globalWindows.pipe(
            ScheduleIntervals.intersect(service.maintenanceWindows)
          )
        }

        if (!ScheduleIntervals.isNonEmpty(globalWindows)) {
          yield* Effect.fail("No common maintenance windows found across all services")
        }

        return globalWindows
      })
    }

    const optimizeForCriticality = (
      baseWindows: ScheduleIntervals.Intervals,
      services: readonly ServiceMaintenanceConfig[]
    ): ScheduleIntervals.Intervals => {
      // High-criticality services get priority - schedule them first
      const highCriticalServices = services.filter(s => s.criticalityLevel === "high")
      const mediumCriticalServices = services.filter(s => s.criticalityLevel === "medium")
      const lowCriticalServices = services.filter(s => s.criticalityLevel === "low")

      // For this example, we prioritize high-critical services
      // In practice, you might want more sophisticated scheduling logic
      return baseWindows
    }

    const scheduleSystemMaintenance = (
      services: readonly ServiceMaintenanceConfig[]
    ): Effect.Effect<ScheduleIntervals.Intervals, string> => {
      return Effect.gen(function* () {
        // Find common windows across all services
        const commonWindows = yield* findGlobalMaintenanceWindows(services)
        
        // Optimize based on service criticality
        const optimizedWindows = optimizeForCriticality(commonWindows, services)
        
        // Ensure we have sufficient maintenance time
        const totalMaintenanceTime = services.length * 30 * 60 * 1000 // 30 min per service
        const availableTime = ScheduleIntervals.end(optimizedWindows) - 
                             ScheduleIntervals.start(optimizedWindows)
        
        if (availableTime < totalMaintenanceTime) {
          yield* Effect.fail(
            `Insufficient maintenance window: need ${totalMaintenanceTime}ms, have ${availableTime}ms`
          )
        }

        return optimizedWindows
      })
    }

    return { scheduleSystemMaintenance }
  })
}

// Usage example
const createMaintenanceExample = Effect.gen(function* () {
  const scheduler = yield* createMaintenanceScheduler()
  
  const services: readonly ServiceMaintenanceConfig[] = [
    {
      serviceName: "database",
      maintenanceWindows: ScheduleIntervals.fromIterable([
        ScheduleInterval.make(2 * 3600 * 1000, 6 * 3600 * 1000) // 2-6 AM
      ]),
      criticalityLevel: "high",
      dependencies: []
    },
    {
      serviceName: "api-gateway",
      maintenanceWindows: ScheduleIntervals.fromIterable([
        ScheduleInterval.make(1 * 3600 * 1000, 5 * 3600 * 1000), // 1-5 AM
        ScheduleInterval.make(23 * 3600 * 1000, 24 * 3600 * 1000) // 11 PM-12 AM
      ]),
      criticalityLevel: "medium",
      dependencies: ["database"]
    },
    {
      serviceName: "cache",
      maintenanceWindows: ScheduleIntervals.fromIterable([
        ScheduleInterval.make(0, 8 * 3600 * 1000) // 12-8 AM
      ]),
      criticalityLevel: "low",
      dependencies: ["database", "api-gateway"]
    }
  ]

  const maintenanceSchedule = yield* scheduler.scheduleSystemMaintenance(services)
  
  // The result will be the intersection: 2-5 AM (common to all services)
  return maintenanceSchedule
})
```

### Example 3: Content Delivery Network (CDN) Cache Optimization

Managing cache refresh schedules across different geographic regions:

```typescript
import { Effect, ScheduleIntervals, ScheduleInterval, DateTime, Array as Arr, HashMap } from "effect"

interface GeographicRegion {
  readonly regionId: string
  readonly timeZoneOffset: number // in milliseconds
  readonly peakTrafficHours: ScheduleIntervals.Intervals
  readonly lowTrafficHours: ScheduleIntervals.Intervals
}

interface CacheRefreshStrategy {
  readonly contentType: "static" | "dynamic" | "user-generated"
  readonly refreshFrequency: Duration.Duration
  readonly maxStaleTime: Duration.Duration
}

interface CDNOptimizer {
  readonly optimizeCacheRefreshSchedule: (
    regions: readonly GeographicRegion[],
    strategy: CacheRefreshStrategy
  ) => Effect.Effect<HashMap.HashMap<string, ScheduleIntervals.Intervals>, string>
}

const createCDNOptimizer = (): Effect.Effect<CDNOptimizer> => {
  return Effect.gen(function* () {
    
    const convertToUTC = (
      intervals: ScheduleIntervals.Intervals,
      timezoneOffset: number
    ): ScheduleIntervals.Intervals => {
      // Convert local time intervals to UTC
      const utcIntervals = intervals.intervals.pipe(
        Chunk.map(interval => 
          ScheduleInterval.make(
            interval.startMillis - timezoneOffset,
            interval.endMillis - timezoneOffset
          )
        )
      )
      return ScheduleIntervals.make(utcIntervals)
    }

    const findOptimalRefreshWindows = (
      regions: readonly GeographicRegion[]
    ): Effect.Effect<ScheduleIntervals.Intervals, string> => {
      return Effect.gen(function* () {
        // Convert all regions' low traffic hours to UTC
        const allLowTrafficWindows = regions.map(region =>
          convertToUTC(region.lowTrafficHours, region.timeZoneOffset)
        )

        if (allLowTrafficWindows.length === 0) {
          yield* Effect.fail("No regions provided")
        }

        // Start with first region's low traffic hours
        let globalLowTraffic = allLowTrafficWindows[0]

        // Find intersection of all regions' low traffic periods
        for (const windowSet of allLowTrafficWindows.slice(1)) {
          globalLowTraffic = globalLowTraffic.pipe(
            ScheduleIntervals.intersect(windowSet)
          )
        }

        // If no common low-traffic time, union all windows for distributed refresh
        if (!ScheduleIntervals.isNonEmpty(globalLowTraffic)) {
          globalLowTraffic = allLowTrafficWindows.reduce(
            (acc, windows) => acc.pipe(ScheduleIntervals.union(windows)),
            ScheduleIntervals.empty
          )
        }

        return globalLowTraffic
      })
    }

    const optimizeCacheRefreshSchedule = (
      regions: readonly GeographicRegion[],
      strategy: CacheRefreshStrategy
    ): Effect.Effect<HashMap.HashMap<string, ScheduleIntervals.Intervals>, string> => {
      return Effect.gen(function* () {
        const optimalWindows = yield* findOptimalRefreshWindows(regions)
        
        // Create region-specific schedules
        const regionSchedules = HashMap.empty<string, ScheduleIntervals.Intervals>()

        for (const region of regions) {
          const regionUTCLowTraffic = convertToUTC(
            region.lowTrafficHours, 
            region.timeZoneOffset
          )

          // Find intersection with global optimal windows
          const regionOptimalWindows = optimalWindows.pipe(
            ScheduleIntervals.intersect(regionUTCLowTraffic)
          )

          // If no intersection, use region's own low traffic hours
          const finalSchedule = ScheduleIntervals.isNonEmpty(regionOptimalWindows)
            ? regionOptimalWindows
            : regionUTCLowTraffic

          regionSchedules.pipe(
            HashMap.set(region.regionId, finalSchedule)
          )
        }

        return regionSchedules
      })
    }

    return { optimizeCacheRefreshSchedule }
  })
}

// Usage example with real-world regions
const createCDNExample = Effect.gen(function* () {
  const optimizer = yield* createCDNOptimizer()
  
  const regions: readonly GeographicRegion[] = [
    {
      regionId: "us-east",
      timeZoneOffset: -5 * 3600 * 1000, // EST
      peakTrafficHours: ScheduleIntervals.fromIterable([
        ScheduleInterval.make(9 * 3600 * 1000, 17 * 3600 * 1000) // 9 AM - 5 PM EST
      ]),
      lowTrafficHours: ScheduleIntervals.fromIterable([
        ScheduleInterval.make(2 * 3600 * 1000, 6 * 3600 * 1000) // 2-6 AM EST
      ])
    },
    {
      regionId: "europe-west",
      timeZoneOffset: 1 * 3600 * 1000, // CET
      peakTrafficHours: ScheduleIntervals.fromIterable([
        ScheduleInterval.make(8 * 3600 * 1000, 18 * 3600 * 1000) // 8 AM - 6 PM CET
      ]),
      lowTrafficHours: ScheduleIntervals.fromIterable([
        ScheduleInterval.make(1 * 3600 * 1000, 5 * 3600 * 1000) // 1-5 AM CET
      ])
    },
    {
      regionId: "asia-pacific",
      timeZoneOffset: 9 * 3600 * 1000, // JST
      peakTrafficHours: ScheduleIntervals.fromIterable([
        ScheduleInterval.make(9 * 3600 * 1000, 19 * 3600 * 1000) // 9 AM - 7 PM JST
      ]),
      lowTrafficHours: ScheduleIntervals.fromIterable([
        ScheduleInterval.make(3 * 3600 * 1000, 7 * 3600 * 1000) // 3-7 AM JST
      ])
    }
  ]

  const strategy: CacheRefreshStrategy = {
    contentType: "static",
    refreshFrequency: Duration.minutes(30),
    maxStaleTime: Duration.hours(2)
  }

  const optimizedSchedules = yield* optimizer.optimizeCacheRefreshSchedule(regions, strategy)
  
  return optimizedSchedules
})
```

## Advanced Features Deep Dive

### Feature 1: Union Operations with Complex Merging

Union operations automatically merge overlapping intervals and maintain chronological order:

#### Basic Union Usage

```typescript
import { ScheduleIntervals, ScheduleInterval } from "effect"

const morningSlots = ScheduleIntervals.fromIterable([
  ScheduleInterval.make(9 * 3600 * 1000, 11 * 3600 * 1000),  // 9-11 AM
  ScheduleInterval.make(11 * 3600 * 1000, 12 * 3600 * 1000)  // 11-12 PM
])

const afternoonSlots = ScheduleIntervals.fromIterable([
  ScheduleInterval.make(13 * 3600 * 1000, 15 * 3600 * 1000), // 1-3 PM
  ScheduleInterval.make(14 * 3600 * 1000, 16 * 3600 * 1000)  // 2-4 PM (overlaps)
])

// Union automatically merges overlapping intervals
const allSlots = morningSlots.pipe(
  ScheduleIntervals.union(afternoonSlots)
)
// Results in: [9-12 PM], [13-16 PM] (merged overlapping afternoon slots)
```

#### Real-World Union Example: Resource Availability Aggregation

```typescript
import { Effect, ScheduleIntervals, ScheduleInterval, HashMap, Array as Arr } from "effect"

interface ResourceAvailability {
  readonly resourceId: string
  readonly availableIntervals: ScheduleIntervals.Intervals
}

const aggregateResourceAvailability = (
  resources: readonly ResourceAvailability[]
): Effect.Effect<ScheduleIntervals.Intervals> => {
  return Effect.gen(function* () {
    if (resources.length === 0) {
      return ScheduleIntervals.empty
    }

    // Union all resource availability windows
    const aggregatedAvailability = resources.reduce(
      (acc, resource) => acc.pipe(
        ScheduleIntervals.union(resource.availableIntervals)
      ),
      ScheduleIntervals.empty
    )

    return aggregatedAvailability
  })
}

// Helper to create time-based availability patterns
const createDailyAvailability = (
  startHour: number,
  endHour: number,
  daysCount: number = 7
): ScheduleIntervals.Intervals => {
  const intervals = Arr.range(0, daysCount - 1).map(day => {
    const dayStart = day * 24 * 3600 * 1000
    const intervalStart = dayStart + startHour * 3600 * 1000
    const intervalEnd = dayStart + endHour * 3600 * 1000
    return ScheduleInterval.make(intervalStart, intervalEnd)
  })
  
  return ScheduleIntervals.fromIterable(intervals)
}
```

### Feature 2: Intersection Operations for Conflict Detection

Intersection operations find overlapping time periods between interval collections:

#### Advanced Intersection: Scheduling Conflict Resolution

```typescript
import { Effect, ScheduleIntervals, ScheduleInterval, Array as Arr } from "effect"

interface SchedulingRequest {
  readonly id: string
  readonly requestedIntervals: ScheduleIntervals.Intervals
  readonly priority: number
  readonly flexible: boolean
}

interface ConflictResolution {
  readonly conflictingRequests: readonly string[]
  readonly availableAlternatives: ScheduleIntervals.Intervals
  readonly suggestedResolution: "reschedule" | "split" | "reject"
}

const detectAndResolveConflicts = (
  existingSchedule: ScheduleIntervals.Intervals,
  newRequests: readonly SchedulingRequest[]
): Effect.Effect<readonly ConflictResolution[]> => {
  return Effect.gen(function* () {
    const resolutions: ConflictResolution[] = []

    for (const request of newRequests) {
      // Find conflicts with existing schedule
      const conflicts = existingSchedule.pipe(
        ScheduleIntervals.intersect(request.requestedIntervals)
      )

      if (ScheduleIntervals.isNonEmpty(conflicts)) {
        // Find alternative time slots
        const alternatives = findAlternativeSlots(
          existingSchedule,
          request.requestedIntervals,
          request.flexible
        )

        const resolution: ConflictResolution = {
          conflictingRequests: [request.id],
          availableAlternatives: alternatives,
          suggestedResolution: request.flexible && ScheduleIntervals.isNonEmpty(alternatives)
            ? "reschedule"
            : "reject"
        }

        resolutions.push(resolution)
      }
    }

    return resolutions
  })
}

const findAlternativeSlots = (
  existingSchedule: ScheduleIntervals.Intervals,
  requestedSlots: ScheduleIntervals.Intervals,
  flexible: boolean
): ScheduleIntervals.Intervals => {
  if (!flexible) {
    return ScheduleIntervals.empty
  }

  // For simplicity, this example shows the concept
  // In practice, you'd implement more sophisticated logic
  const requestDuration = ScheduleIntervals.end(requestedSlots) - 
                         ScheduleIntervals.start(requestedSlots)
  
  // Find gaps in existing schedule that could accommodate the request
  // This is a simplified example - real implementation would be more complex
  return ScheduleIntervals.empty
}
```

### Feature 3: Temporal Operations and Queries

Access start/end times and perform temporal comparisons:

#### Advanced Temporal: Schedule Optimization

```typescript
import { Effect, ScheduleIntervals, ScheduleInterval, Option, Duration } from "effect"

interface ScheduleMetrics {
  readonly totalDuration: Duration.Duration
  readonly utilizationRatio: number
  readonly fragmentationScore: number
  readonly peakLoadTimes: ScheduleIntervals.Intervals
}

const analyzeScheduleEfficiency = (
  schedule: ScheduleIntervals.Intervals,
  totalAvailableTime: Duration.Duration
): Effect.Effect<ScheduleMetrics> => {
  return Effect.gen(function* () {
    if (!ScheduleIntervals.isNonEmpty(schedule)) {
      return {
        totalDuration: Duration.zero,
        utilizationRatio: 0,
        fragmentationScore: 0,
        peakLoadTimes: ScheduleIntervals.empty
      }
    }

    // Calculate total scheduled duration
    const totalScheduled = schedule.intervals.pipe(
      Chunk.reduce(Duration.zero, (acc, interval) => 
        Duration.sum(acc, Duration.millis(interval.endMillis - interval.startMillis))
      )
    )

    // Calculate utilization ratio
    const utilizationRatio = Duration.toMillis(totalScheduled) / 
                           Duration.toMillis(totalAvailableTime)

    // Calculate fragmentation score (more intervals = more fragmented)
    const intervalCount = Chunk.size(schedule.intervals)
    const fragmentationScore = intervalCount / Duration.toHours(totalScheduled)

    // Identify peak load times (simplified heuristic)
    const peakThreshold = Duration.toMillis(totalScheduled) / intervalCount * 1.5
    const peakIntervals = schedule.intervals.pipe(
      Chunk.filter(interval => 
        (interval.endMillis - interval.startMillis) > peakThreshold
      )
    )

    return {
      totalDuration: totalScheduled,
      utilizationRatio,
      fragmentationScore,
      peakLoadTimes: ScheduleIntervals.make(peakIntervals)
    }
  })
}

const optimizeScheduleLayout = (
  schedule: ScheduleIntervals.Intervals,
  maxFragmentation: number = 2.0
): Effect.Effect<ScheduleIntervals.Intervals> => {
  return Effect.gen(function* () {
    const metrics = yield* analyzeScheduleEfficiency(
      schedule, 
      Duration.hours(24) // Assume 24-hour window
    )

    if (metrics.fragmentationScore <= maxFragmentation) {
      return schedule // Already optimized
    }

    // Merge nearby intervals to reduce fragmentation
    const optimizedIntervals = schedule.intervals.pipe(
      Chunk.reduce(Chunk.empty<ScheduleInterval.Interval>(), (acc, current) => {
        const lastOption = Chunk.last(acc)
        
        return Option.match(lastOption, {
          onNone: () => Chunk.append(acc, current),
          onSome: (last) => {
            // Merge if intervals are close (within 30 minutes)
            const gap = current.startMillis - last.endMillis
            if (gap <= 30 * 60 * 1000) {
              const merged = ScheduleInterval.make(last.startMillis, current.endMillis)
              return Chunk.modify(acc, Chunk.size(acc) - 1, () => merged)
            } else {
              return Chunk.append(acc, current)
            }
          }
        })
      })
    )

    return ScheduleIntervals.make(optimizedIntervals)
  })
}
```

## Practical Patterns & Best Practices

### Pattern 1: Interval Collection Builder

```typescript
import { Effect, ScheduleIntervals, ScheduleInterval, Array as Arr, Duration } from "effect"

class IntervalCollectionBuilder {
  private intervals: ScheduleInterval.Interval[] = []

  addInterval(start: number, end: number): this {
    this.intervals.push(ScheduleInterval.make(start, end))
    return this
  }

  addDailyRecurrence(
    startHour: number,
    endHour: number,
    days: number
  ): this {
    const dailyIntervals = Arr.range(0, days - 1).map(day => {
      const dayOffset = day * 24 * 60 * 60 * 1000
      return ScheduleInterval.make(
        dayOffset + startHour * 60 * 60 * 1000,
        dayOffset + endHour * 60 * 60 * 1000
      )
    })
    
    this.intervals.push(...dailyIntervals)
    return this
  }

  addBusinessHours(days: number = 5): this {
    return this.addDailyRecurrence(9, 17, days) // 9 AM - 5 PM
  }

  addMaintenanceWindows(days: number = 7): this {
    return this.addDailyRecurrence(2, 4, days) // 2-4 AM daily
  }

  excludeWeekends(): this {
    // Filter out weekend intervals (simplified logic)
    this.intervals = this.intervals.filter(interval => {
      const day = Math.floor(interval.startMillis / (24 * 60 * 60 * 1000)) % 7
      return day >= 1 && day <= 5 // Monday=1, Friday=5
    })
    return this
  }

  build(): ScheduleIntervals.Intervals {
    return ScheduleIntervals.fromIterable(this.intervals)
  }
}

// Usage example
const createWorkSchedule = () => {
  return new IntervalCollectionBuilder()
    .addBusinessHours(30) // 30 days of business hours
    .excludeWeekends()
    .build()
}

const createMaintenanceSchedule = () => {
  return new IntervalCollectionBuilder()
    .addMaintenanceWindows(30)
    .addInterval(
      Date.now() + Duration.toMillis(Duration.days(7)), // Next week
      Date.now() + Duration.toMillis(Duration.days(7)) + Duration.toMillis(Duration.hours(4))
    )
    .build()
}
```

### Pattern 2: Schedule Validation and Constraints

```typescript
import { Effect, ScheduleIntervals, ScheduleInterval, Duration, Either } from "effect"

interface ScheduleConstraints {
  readonly minIntervalDuration: Duration.Duration
  readonly maxIntervalDuration: Duration.Duration
  readonly maxDailyDuration: Duration.Duration
  readonly requiredBreakBetweenIntervals: Duration.Duration
}

interface ValidationError {
  readonly _tag: "ValidationError"
  readonly message: string
  readonly intervalIndex?: number
}

const validateScheduleConstraints = (
  schedule: ScheduleIntervals.Intervals,
  constraints: ScheduleConstraints
): Effect.Effect<boolean, ValidationError> => {
  return Effect.gen(function* () {
    if (!ScheduleIntervals.isNonEmpty(schedule)) {
      return true // Empty schedule is valid
    }

    const intervals = schedule.intervals
    
    // Validate each interval duration
    for (let i = 0; i < Chunk.size(intervals); i++) {
      const interval = Chunk.unsafeGet(intervals, i)
      const duration = Duration.millis(interval.endMillis - interval.startMillis)
      
      if (Duration.lessThan(duration, constraints.minIntervalDuration)) {
        yield* Effect.fail({
          _tag: "ValidationError" as const,
          message: `Interval ${i} duration too short: ${Duration.toMillis(duration)}ms`,
          intervalIndex: i
        })
      }
      
      if (Duration.greaterThan(duration, constraints.maxIntervalDuration)) {
        yield* Effect.fail({
          _tag: "ValidationError" as const,
          message: `Interval ${i} duration too long: ${Duration.toMillis(duration)}ms`,
          intervalIndex: i
        })
      }
    }

    // Validate breaks between intervals
    for (let i = 0; i < Chunk.size(intervals) - 1; i++) {
      const current = Chunk.unsafeGet(intervals, i)
      const next = Chunk.unsafeGet(intervals, i + 1)
      const breakDuration = Duration.millis(next.startMillis - current.endMillis)
      
      if (Duration.lessThan(breakDuration, constraints.requiredBreakBetweenIntervals)) {
        yield* Effect.fail({
          _tag: "ValidationError" as const,
          message: `Insufficient break between intervals ${i} and ${i + 1}`,
          intervalIndex: i
        })
      }
    }

    return true
  })
}

const enforceScheduleConstraints = (
  schedule: ScheduleIntervals.Intervals,
  constraints: ScheduleConstraints
): Effect.Effect<ScheduleIntervals.Intervals, ValidationError> => {
  return Effect.gen(function* () {
    // First validate
    yield* validateScheduleConstraints(schedule, constraints)

    // If validation passes, apply automatic adjustments
    const adjustedIntervals = schedule.intervals.pipe(
      Chunk.map(interval => {
        const duration = interval.endMillis - interval.startMillis
        const minDuration = Duration.toMillis(constraints.minIntervalDuration)
        const maxDuration = Duration.toMillis(constraints.maxIntervalDuration)
        
        // Adjust duration if needed
        if (duration < minDuration) {
          return ScheduleInterval.make(interval.startMillis, interval.startMillis + minDuration)
        } else if (duration > maxDuration) {
          return ScheduleInterval.make(interval.startMillis, interval.startMillis + maxDuration)
        }
        
        return interval
      })
    )

    return ScheduleIntervals.make(adjustedIntervals)
  })
}
```

### Pattern 3: Schedule Template System

```typescript
import { Effect, ScheduleIntervals, ScheduleInterval, Duration, HashMap } from "effect"

interface ScheduleTemplate {
  readonly name: string
  readonly description: string
  readonly generate: (params: Record<string, unknown>) => Effect.Effect<ScheduleIntervals.Intervals, string>
}

const createScheduleTemplateLibrary = (): Effect.Effect<HashMap.HashMap<string, ScheduleTemplate>> => {
  return Effect.gen(function* () {
    const templates = HashMap.empty<string, ScheduleTemplate>()

    // Business hours template
    const businessHoursTemplate: ScheduleTemplate = {
      name: "business-hours",
      description: "Standard business hours schedule",
      generate: (params) => Effect.gen(function* () {
        const startHour = (params.startHour as number) ?? 9
        const endHour = (params.endHour as number) ?? 17
        const days = (params.days as number) ?? 5
        
        const intervals = Array.from({ length: days }, (_, day) => {
          const dayOffset = day * 24 * 60 * 60 * 1000
          return ScheduleInterval.make(
            dayOffset + startHour * 60 * 60 * 1000,
            dayOffset + endHour * 60 * 60 * 1000
          )
        })
        
        return ScheduleIntervals.fromIterable(intervals)
      })
    }

    // Shift work template
    const shiftWorkTemplate: ScheduleTemplate = {
      name: "shift-work",
      description: "Rotating shift schedule",
      generate: (params) => Effect.gen(function* () {
        const shiftDuration = (params.shiftDuration as number) ?? 8
        const shiftsPerDay = (params.shiftsPerDay as number) ?? 3
        const days = (params.days as number) ?? 7
        
        const intervals: ScheduleInterval.Interval[] = []
        
        for (let day = 0; day < days; day++) {
          for (let shift = 0; shift < shiftsPerDay; shift++) {
            const dayOffset = day * 24 * 60 * 60 * 1000
            const shiftStart = dayOffset + shift * shiftDuration * 60 * 60 * 1000
            const shiftEnd = shiftStart + shiftDuration * 60 * 60 * 1000
            
            intervals.push(ScheduleInterval.make(shiftStart, shiftEnd))
          }
        }
        
        return ScheduleIntervals.fromIterable(intervals)
      })
    }

    // On-call rotation template
    const onCallTemplate: ScheduleTemplate = {
      name: "on-call-rotation",
      description: "On-call duty rotation schedule",
      generate: (params) => Effect.gen(function* () {
        const rotationDays = (params.rotationDays as number) ?? 7
        const engineersCount = (params.engineersCount as number) ?? 4
        const totalWeeks = (params.totalWeeks as number) ?? 4
        
        const intervals: ScheduleInterval.Interval[] = []
        
        for (let week = 0; week < totalWeeks; week++) {
          const engineerIndex = week % engineersCount
          const weekStart = week * 7 * 24 * 60 * 60 * 1000
          const weekEnd = weekStart + rotationDays * 24 * 60 * 60 * 1000
          
          intervals.push(ScheduleInterval.make(weekStart, weekEnd))
        }
        
        return ScheduleIntervals.fromIterable(intervals)
      })
    }

    return templates.pipe(
      HashMap.set("business-hours", businessHoursTemplate),
      HashMap.set("shift-work", shiftWorkTemplate),
      HashMap.set("on-call-rotation", onCallTemplate)
    )
  })
}

// Template usage service
const createScheduleFromTemplate = (
  templateName: string,
  params: Record<string, unknown>
): Effect.Effect<ScheduleIntervals.Intervals, string> => {
  return Effect.gen(function* () {
    const templates = yield* createScheduleTemplateLibrary()
    const template = HashMap.get(templates, templateName)
    
    if (Option.isNone(template)) {
      yield* Effect.fail(`Template "${templateName}" not found`)
    }
    
    return yield* template.value.generate(params)
  })
}
```

## Integration Examples

### Integration with Schedule Module

ScheduleIntervals works seamlessly with the Schedule module for advanced retry and repeat patterns:

```typescript
import { Effect, Schedule, ScheduleIntervals, ScheduleInterval, Duration } from "effect"

// Create a custom schedule that respects business hours
const createBusinessHoursSchedule = <A>(
  businessHours: ScheduleIntervals.Intervals
): Schedule.Schedule<number, A> => {
  return Schedule.makeWithState(0, (now, _input, attempt) => {
    const currentTime = now
    
    // Check if current time falls within business hours
    const isInBusinessHours = businessHours.intervals.pipe(
      Chunk.some(interval =>
        currentTime >= interval.startMillis && currentTime <= interval.endMillis
      )
    )
    
    if (!isInBusinessHours) {
      // Find next business hour window
      const nextWindow = businessHours.intervals.pipe(
        Chunk.findFirst(interval => interval.startMillis > currentTime)
      )
      
      return Option.match(nextWindow, {
        onNone: () => Effect.succeed([attempt + 1, attempt, ScheduleDecision.done]),
        onSome: (interval) => {
          const delay = Duration.millis(interval.startMillis - currentTime)
          return Effect.succeed([
            attempt + 1,
            attempt,
            ScheduleDecision.continue(ScheduleIntervals.fromIterable([interval]))
          ])
        }
      })
    }
    
    // Continue with exponential backoff during business hours
    return Effect.succeed([
      attempt + 1,
      attempt,
      ScheduleDecision.continue(
        ScheduleIntervals.fromIterable([
          ScheduleInterval.make(now, now + Math.pow(2, attempt) * 1000)
        ])
      )
    ])
  })
}

// Usage with retry
const retryDuringBusinessHours = <A, E, R>(
  effect: Effect.Effect<A, E, R>,
  businessHours: ScheduleIntervals.Intervals
): Effect.Effect<A, E, R> => {
  return effect.pipe(
    Effect.retry(createBusinessHoursSchedule(businessHours))
  )
}
```

### Integration with Testing Frameworks

Create comprehensive test utilities for schedule validation:

```typescript
import { Effect, ScheduleIntervals, ScheduleInterval, TestClock, Chunk } from "effect"

interface ScheduleTestHarness {
  readonly validateScheduleExecution: (
    schedule: ScheduleIntervals.Intervals,
    expectedBehavior: readonly { time: number; shouldExecute: boolean }[]
  ) => Effect.Effect<boolean, string>
}

const createScheduleTestHarness = (): Effect.Effect<ScheduleTestHarness> => {
  return Effect.gen(function* () {
    
    const validateScheduleExecution = (
      schedule: ScheduleIntervals.Intervals,
      expectedBehavior: readonly { time: number; shouldExecute: boolean }[]
    ): Effect.Effect<boolean, string> => {
      return Effect.gen(function* () {
        for (const expectation of expectedBehavior) {
          const shouldExecute = schedule.intervals.pipe(
            Chunk.some(interval =>
              expectation.time >= interval.startMillis &&
              expectation.time <= interval.endMillis
            )
          )
          
          if (shouldExecute !== expectation.shouldExecute) {
            yield* Effect.fail(
              `Schedule validation failed at time ${expectation.time}: ` +
              `expected ${expectation.shouldExecute}, got ${shouldExecute}`
            )
          }
        }
        
        return true
      })
    }

    return { validateScheduleExecution }
  })
}

// Test helper functions
const createTestSchedule = (
  intervals: readonly [number, number][]
): ScheduleIntervals.Intervals => {
  return ScheduleIntervals.fromIterable(
    intervals.map(([start, end]) => ScheduleInterval.make(start, end))
  )
}

const runScheduleTests = Effect.gen(function* () {
  const harness = yield* createScheduleTestHarness()
  
  // Test business hours schedule
  const businessHours = createTestSchedule([
    [9 * 3600 * 1000, 12 * 3600 * 1000], // 9 AM - 12 PM
    [13 * 3600 * 1000, 17 * 3600 * 1000] // 1 PM - 5 PM
  ])
  
  const testCases = [
    { time: 8 * 3600 * 1000, shouldExecute: false }, // 8 AM - before hours
    { time: 10 * 3600 * 1000, shouldExecute: true }, // 10 AM - during hours
    { time: 12.5 * 3600 * 1000, shouldExecute: false }, // 12:30 PM - lunch break
    { time: 15 * 3600 * 1000, shouldExecute: true }, // 3 PM - during hours
    { time: 18 * 3600 * 1000, shouldExecute: false } // 6 PM - after hours
  ]
  
  const isValid = yield* harness.validateScheduleExecution(businessHours, testCases)
  return isValid
})
```

### Integration with Monitoring and Observability

Track schedule performance and execution patterns:

```typescript
import { Effect, ScheduleIntervals, ScheduleInterval, Metrics, Duration } from "effect"

interface ScheduleMetrics {
  readonly executionCount: Metrics.Counter.Counter
  readonly averageExecutionTime: Metrics.Histogram.Histogram
  readonly scheduleUtilization: Metrics.Gauge.Gauge
  readonly intervalOverruns: Metrics.Counter.Counter
}

const createScheduleMetrics = (): Effect.Effect<ScheduleMetrics> => {
  return Effect.gen(function* () {
    const executionCount = yield* Metrics.counter("schedule_executions_total")
    const averageExecutionTime = yield* Metrics.histogram("schedule_execution_duration_ms")
    const scheduleUtilization = yield* Metrics.gauge("schedule_utilization_ratio")
    const intervalOverruns = yield* Metrics.counter("schedule_interval_overruns_total")
    
    return {
      executionCount,
      averageExecutionTime,
      scheduleUtilization,
      intervalOverruns
    }
  })
}

const withScheduleMetrics = <A, E, R>(
  schedule: ScheduleIntervals.Intervals,
  operation: Effect.Effect<A, E, R>
): Effect.Effect<A, E, R> => {
  return Effect.gen(function* () {
    const metrics = yield* createScheduleMetrics()
    const startTime = yield* Effect.sync(() => Date.now())
    
    // Check if current time is within schedule
    const currentTime = startTime
    const isInSchedule = schedule.intervals.pipe(
      Chunk.some(interval =>
        currentTime >= interval.startMillis && currentTime <= interval.endMillis
      )
    )
    
    if (!isInSchedule) {
      yield* Effect.fail(new Error("Operation attempted outside of scheduled intervals"))
    }
    
    // Execute operation with metrics
    const result = yield* operation.pipe(
      Effect.tap(() => Metrics.counter.increment(metrics.executionCount)),
      Effect.timed
    )
    
    const [duration, value] = result
    
    // Record metrics
    yield* Metrics.histogram.update(metrics.averageExecutionTime, Duration.toMillis(duration))
    
    // Calculate and record utilization
    const totalScheduledTime = schedule.intervals.pipe(
      Chunk.reduce(0, (acc, interval) => acc + (interval.endMillis - interval.startMillis))
    )
    const executionTime = Duration.toMillis(duration)
    const utilization = executionTime / totalScheduledTime
    
    yield* Metrics.gauge.set(metrics.scheduleUtilization, utilization)
    
    return value
  })
}
```

## Conclusion

ScheduleIntervals provides a powerful foundation for managing complex interval collections in scheduling systems. It offers union operations for combining availability windows, intersection operations for conflict detection, and temporal queries for schedule optimization.

Key benefits:
- **Type Safety**: Compile-time guarantees about interval validity and operations
- **Composability**: Easy combination of multiple interval collections with automatic merging
- **Performance**: Efficient algorithms for union and intersection operations on large interval sets

ScheduleIntervals is essential when building sophisticated scheduling systems, resource management platforms, or any application requiring complex time-based coordination across multiple entities or constraints.