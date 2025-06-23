# MutableHashSet: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem MutableHashSet Solves

Building sets incrementally or performing frequent deduplication operations with traditional JavaScript Sets leads to performance issues and memory waste in functional programming contexts. Traditional approaches suffer from expensive copying operations, lack of structural optimizations, and poor integration with Effect's type system:

```typescript
// Traditional approach - manual set building with performance issues
class EventTracker {
  private seenEvents = new Set<string>()
  
  processEventBatch(events: Event[]): ProcessingResult {
    // For immutability, we need to copy the entire set
    const newSeenEvents = new Set(this.seenEvents)
    const processedEvents: Event[] = []
    const duplicates: Event[] = []
    
    for (const event of events) {
      if (newSeenEvents.has(event.id)) {
        duplicates.push(event)
      } else {
        newSeenEvents.add(event.id)
        processedEvents.push(event)
      }
    }
    
    // Create new tracker instance to maintain immutability - expensive!
    return {
      tracker: new EventTracker(Array.from(newSeenEvents)),
      processedEvents,
      duplicates
    }
  }
  
  // Inefficient batch deduplication
  deduplicateUserActions(actions: UserAction[]): UserAction[] {
    const seen = new Set<string>()
    const result: UserAction[] = []
    
    for (const action of actions) {
      const key = `${action.userId}-${action.type}-${action.timestamp}`
      if (!seen.has(key)) {
        seen.add(key)
        result.push(action)
      }
    }
    return result
  }
  
  // Set operations are verbose and error-prone
  findActiveUsers(currentUsers: Set<string>, recentActivity: Set<string>): Set<string> {
    const activeUsers = new Set<string>()
    for (const user of currentUsers) {
      if (recentActivity.has(user)) {
        activeUsers.add(user)
      }
    }
    return activeUsers
  }
  
  private constructor(initialEvents?: string[]) {
    if (initialEvents) {
      this.seenEvents = new Set(initialEvents)
    }
  }
}

// Performance problems with large datasets
const tracker = new EventTracker()

// Each batch creates a full copy of the seen events set - O(n) memory each time
const result1 = tracker.processEventBatch(batch1) // Copy entire set
const result2 = result1.tracker.processEventBatch(batch2) // Copy entire set again
const result3 = result2.tracker.processEventBatch(batch3) // Copy entire set again

// Memory usage grows linearly with each batch even though data might be mostly the same
```

This approach leads to:
- **Performance degradation** - O(n) copying operations for maintaining immutability
- **Memory inefficiency** - Full set duplication with every modification
- **Complex state management** - Manual tracking of mutations and immutability
- **Verbose deduplication** - Repetitive membership testing patterns
- **Poor composability** - Difficult to integrate with functional data pipelines

### The MutableHashSet Solution

MutableHashSet provides efficient, in-place set operations while maintaining Effect's type safety and composability. It's optimized for scenarios where controlled mutability improves performance:

```typescript
import { MutableHashSet, Effect, pipe } from "effect"

// Efficient event tracking with local mutability
const processEventBatch = (events: Event[]) => Effect.gen(function* () {
  const seenEvents = MutableHashSet.empty<string>()
  const processedEvents: Event[] = []
  const duplicates: Event[] = []
  
  for (const event of events) {
    if (MutableHashSet.has(seenEvents, event.id)) {
      duplicates.push(event)
    } else {
      // O(1) in-place addition - no copying
      MutableHashSet.add(seenEvents, event.id)
      processedEvents.push(event)
    }
  }
  
  return { seenEvents, processedEvents, duplicates }
})

// Streamlined deduplication
const deduplicateActions = (actions: UserAction[]) => Effect.gen(function* () {
  const seen = MutableHashSet.empty<string>()
  const result: UserAction[] = []
  
  for (const action of actions) {
    const key = `${action.userId}-${action.type}-${action.timestamp}`
    if (!MutableHashSet.has(seen, key)) {
      MutableHashSet.add(seen, key)
      result.push(action)
    }
  }
  
  return result
})
```

### Key Concepts

**In-Place Mutation**: Operations modify the original set rather than creating new copies, providing O(1) average performance for additions and removals.

**Controlled Mutability**: Mutability is contained within specific scopes, preventing unintended side effects while enabling performance optimizations.

**Structural Sharing**: When combined with immutable operations, provides efficient data sharing without the overhead of full copying.

## Basic Usage Patterns

### Pattern 1: Creating and Populating Sets

```typescript
import { MutableHashSet } from "effect"

// Create empty set
const userIds = MutableHashSet.empty<string>()

// Create from initial values
const adminRoles = MutableHashSet.make("admin", "moderator", "super_admin")

// Create from iterable
const permissions = MutableHashSet.fromIterable([
  "read", "write", "delete", "admin", "read" // duplicates ignored
])

console.log(MutableHashSet.size(permissions)) // 4
```

### Pattern 2: Basic Set Operations

```typescript
import { MutableHashSet } from "effect"

const activeUsers = MutableHashSet.empty<string>()

// Add elements - returns the same set reference
MutableHashSet.add(activeUsers, "user1")
MutableHashSet.add(activeUsers, "user2")
MutableHashSet.add(activeUsers, "user1") // Duplicate ignored

console.log(MutableHashSet.size(activeUsers)) // 2

// Check membership
const hasUser = MutableHashSet.has(activeUsers, "user1") // true

// Remove elements
MutableHashSet.remove(activeUsers, "user1")
console.log(MutableHashSet.has(activeUsers, "user1")) // false
```

### Pattern 3: Pipelining with Mutable Sets

```typescript
import { MutableHashSet, pipe } from "effect"

// Using pipeable interface for fluent operations
const processedSet = pipe(
  MutableHashSet.make("apple", "banana", "cherry"),
  set => {
    MutableHashSet.add(set, "dragon_fruit")
    MutableHashSet.remove(set, "banana")
    return set
  }
)

// Convert to array for further processing
const fruits = Array.from(processedSet)
console.log(fruits) // ["apple", "cherry", "dragon_fruit"]
```

## Real-World Examples

### Example 1: Event Deduplication System

A real-time analytics system needs to deduplicate incoming events efficiently while processing thousands of events per second:

```typescript
import { MutableHashSet, Effect, Queue, Stream } from "effect"

interface AnalyticsEvent {
  id: string
  userId: string
  eventType: string
  timestamp: number
  metadata: Record<string, unknown>
}

class EventDeduplicator {
  private readonly seenEvents = MutableHashSet.empty<string>()
  private readonly recentUserActions = MutableHashSet.empty<string>()
  
  processEventStream = (events: Stream.Stream<AnalyticsEvent>) => 
    Stream.mapEffect(events, event => this.processEvent(event))
  
  private processEvent = (event: AnalyticsEvent) => Effect.gen(function* () {
    // Generate composite key for deduplication
    const eventKey = `${event.id}-${event.timestamp}`
    const userActionKey = `${event.userId}-${event.eventType}`
    
    // Check for duplicate events
    if (MutableHashSet.has(this.seenEvents, eventKey)) {
      return { type: 'duplicate', event } as const
    }
    
    // Check for rapid repeated actions (within processing window)
    const isRepeatedAction = MutableHashSet.has(this.recentUserActions, userActionKey)
    
    // Add to tracking sets
    MutableHashSet.add(this.seenEvents, eventKey)
    MutableHashSet.add(this.recentUserActions, userActionKey)
    
    // Periodic cleanup to prevent memory growth
    yield* this.periodicCleanup()
    
    return {
      type: 'processed',
      event,
      isRepeatedAction,
      totalEvents: MutableHashSet.size(this.seenEvents)
    } as const
  })
  
  private periodicCleanup = () => Effect.gen(function* () {
    // Clear recent actions periodically to prevent unbounded growth
    if (MutableHashSet.size(this.recentUserActions) > 10000) {
      MutableHashSet.clear(this.recentUserActions)
    }
    
    // In a real system, you might implement time-based cleanup
    // or LRU eviction instead of simple clearing
  })
  
  getStats = () => Effect.sync(() => ({
    totalEventsSeen: MutableHashSet.size(this.seenEvents),
    recentActionsTracked: MutableHashSet.size(this.recentUserActions)
  }))
}

// Usage in analytics pipeline
const runAnalytics = Effect.gen(function* () {
  const deduplicator = new EventDeduplicator()
  const eventQueue = yield* Queue.unbounded<AnalyticsEvent>()
  
  // Create stream from queue
  const eventStream = Stream.fromQueue(eventQueue)
  
  // Process events with deduplication
  const processedStream = deduplicator.processEventStream(eventStream)
  
  // Handle processed events
  yield* Stream.runForEach(processedStream, result => Effect.gen(function* () {
    switch (result.type) {
      case 'duplicate':
        yield* Effect.log(`Duplicate event filtered: ${result.event.id}`)
        break
      case 'processed':
        yield* Effect.log(`Event processed: ${result.event.id} (total: ${result.totalEvents})`)
        // Forward to downstream processing
        break
    }
  }))
})
```

### Example 2: User Permission Cache

A web application caching system that tracks user permissions and roles with efficient membership testing:

```typescript
import { MutableHashSet, Effect, Ref, Schedule } from "effect"

interface UserPermission {
  userId: string
  permission: string
  grantedAt: Date
  expiresAt?: Date
}

class PermissionCache {
  private readonly userPermissions = new Map<string, MutableHashSet<string>>()
  private readonly adminUsers = MutableHashSet.empty<string>()
  private readonly expiredPermissions = MutableHashSet.empty<string>()
  
  grantPermission = (userId: string, permission: string) => Effect.gen(function* () {
    // Get or create user's permission set
    if (!this.userPermissions.has(userId)) {
      this.userPermissions.set(userId, MutableHashSet.empty<string>())
    }
    
    const userPerms = this.userPermissions.get(userId)!
    MutableHashSet.add(userPerms, permission)
    
    // Track admin users separately
    if (permission === 'admin') {
      MutableHashSet.add(this.adminUsers, userId)
    }
    
    yield* Effect.log(`Permission '${permission}' granted to user ${userId}`)
  })
  
  revokePermission = (userId: string, permission: string) => Effect.gen(function* () {
    const userPerms = this.userPermissions.get(userId)
    if (!userPerms) return
    
    MutableHashSet.remove(userPerms, permission)
    
    // Update admin tracking
    if (permission === 'admin') {
      MutableHashSet.remove(this.adminUsers, userId)
    }
    
    // Clean up empty permission sets
    if (MutableHashSet.size(userPerms) === 0) {
      this.userPermissions.delete(userId)
    }
    
    yield* Effect.log(`Permission '${permission}' revoked from user ${userId}`)
  })
  
  hasPermission = (userId: string, permission: string): Effect.Effect<boolean> => 
    Effect.sync(() => {
      const userPerms = this.userPermissions.get(userId)
      if (!userPerms) return false
      
      // Check direct permission or admin override
      return MutableHashSet.has(userPerms, permission) || 
             MutableHashSet.has(this.adminUsers, userId)
    })
  
  batchGrantPermissions = (grants: Array<{ userId: string; permissions: string[] }>) => 
    Effect.gen(function* () {
      for (const grant of grants) {
        // Get or create user's permission set
        if (!this.userPermissions.has(grant.userId)) {
          this.userPermissions.set(grant.userId, MutableHashSet.empty<string>())
        }
        
        const userPerms = this.userPermissions.get(grant.userId)!
        
        // Batch add permissions to the mutable set
        for (const permission of grant.permissions) {
          MutableHashSet.add(userPerms, permission)
          
          if (permission === 'admin') {
            MutableHashSet.add(this.adminUsers, grant.userId)
          }
        }
      }
      
      yield* Effect.log(`Batch granted permissions to ${grants.length} users`)
    })
  
  getUsersWithPermission = (permission: string): Effect.Effect<string[]> =>
    Effect.sync(() => {
      const result: string[] = []
      
      for (const [userId, permissions] of this.userPermissions.entries()) {
        if (MutableHashSet.has(permissions, permission)) {
          result.push(userId)
        }
      }
      
      return result
    })
  
  getCacheStats = (): Effect.Effect<CacheStats> =>
    Effect.sync(() => ({
      totalUsers: this.userPermissions.size,
      adminUsers: MutableHashSet.size(this.adminUsers),
      totalUniquePermissions: this.getAllUniquePermissions().length,
      averagePermissionsPerUser: this.calculateAveragePermissions()
    }))
  
  private getAllUniquePermissions = (): string[] => {
    const allPermissions = MutableHashSet.empty<string>()
    
    for (const permissions of this.userPermissions.values()) {
      for (const permission of permissions) {
        MutableHashSet.add(allPermissions, permission)
      }
    }
    
    return Array.from(allPermissions)
  }
  
  private calculateAveragePermissions = (): number => {
    if (this.userPermissions.size === 0) return 0
    
    let totalPermissions = 0
    for (const permissions of this.userPermissions.values()) {
      totalPermissions += MutableHashSet.size(permissions)
    }
    
    return totalPermissions / this.userPermissions.size
  }
}

interface CacheStats {
  totalUsers: number
  adminUsers: number
  totalUniquePermissions: number
  averagePermissionsPerUser: number
}

// Usage in authentication middleware
const permissionCache = new PermissionCache()

const checkUserAccess = (userId: string, requiredPermission: string) =>
  Effect.gen(function* () {
    const hasAccess = yield* permissionCache.hasPermission(userId, requiredPermission)
    
    if (!hasAccess) {
      return yield* Effect.fail(new UnauthorizedError(
        `User ${userId} lacks permission: ${requiredPermission}`
      ))
    }
    
    return { userId, permission: requiredPermission, granted: true }
  })

class UnauthorizedError extends Error {
  readonly _tag = 'UnauthorizedError'
}
```

### Example 3: Data Pipeline Deduplication

A data processing pipeline that efficiently removes duplicates from large datasets using composite keys:

```typescript
import { MutableHashSet, Effect, Stream, Chunk } from "effect"

interface DataRecord {
  id: string
  customerId: string
  timestamp: Date
  recordType: 'order' | 'payment' | 'refund'
  amount: number
  metadata: Record<string, unknown>
}

interface DeduplicationConfig {
  keyStrategy: 'id' | 'composite' | 'content'
  timeWindow?: number // milliseconds
  maxRecords?: number
}

class DataDeduplicator {
  private readonly seenRecords = MutableHashSet.empty<string>()
  private readonly timeWindowRecords = MutableHashSet.empty<string>()
  private readonly contentHashes = MutableHashSet.empty<string>()
  
  constructor(private readonly config: DeduplicationConfig) {}
  
  processDataStream = <E, R>(
    stream: Stream.Stream<DataRecord, E, R>
  ): Stream.Stream<DeduplicationResult, E, R> =>
    stream.pipe(
      Stream.mapChunks(chunk => 
        Chunk.map(chunk, record => this.processRecord(record))
      )
    )
  
  private processRecord = (record: DataRecord): DeduplicationResult => {
    const key = this.generateKey(record)
    
    // Check if we've seen this record before
    if (MutableHashSet.has(this.seenRecords, key)) {
      return {
        type: 'duplicate',
        record,
        key,
        totalProcessed: MutableHashSet.size(this.seenRecords)
      }
    }
    
    // Add to seen records
    MutableHashSet.add(this.seenRecords, key)
    
    // Handle time window deduplication
    if (this.config.timeWindow) {
      this.handleTimeWindowDeduplication(record, key)
    }
    
    // Handle max records limit
    if (this.config.maxRecords && MutableHashSet.size(this.seenRecords) > this.config.maxRecords) {
      this.performCleanup()
    }
    
    return {
      type: 'unique',
      record,
      key,
      totalProcessed: MutableHashSet.size(this.seenRecords)
    }
  }
  
  private generateKey = (record: DataRecord): string => {
    switch (this.config.keyStrategy) {
      case 'id':
        return record.id
        
      case 'composite':
        return `${record.customerId}-${record.recordType}-${record.amount}`
        
      case 'content':
        // Generate hash of the entire record content
        const contentString = JSON.stringify({
          customerId: record.customerId,
          recordType: record.recordType,
          amount: record.amount,
          timestamp: record.timestamp.toISOString(),
          metadata: record.metadata
        })
        return this.simpleHash(contentString)
        
      default:
        return record.id
    }
  }
  
  private handleTimeWindowDeduplication = (record: DataRecord, key: string) => {
    const timeKey = `${key}-${Math.floor(record.timestamp.getTime() / this.config.timeWindow!)}`
    
    if (MutableHashSet.has(this.timeWindowRecords, timeKey)) {
      // This is a duplicate within the time window
      return
    }
    
    MutableHashSet.add(this.timeWindowRecords, timeKey)
  }
  
  private performCleanup = () => {
    // Simple cleanup: remove half the records
    // In production, you'd implement LRU or time-based eviction
    const currentRecords = Array.from(this.seenRecords)
    const recordsToKeep = currentRecords.slice(currentRecords.length / 2)
    
    MutableHashSet.clear(this.seenRecords)
    
    for (const record of recordsToKeep) {
      MutableHashSet.add(this.seenRecords, record)
    }
  }
  
  private simpleHash = (str: string): string => {
    let hash = 0
    for (let i = 0; i < str.length; i++) {
      const char = str.charCodeAt(i)
      hash = ((hash << 5) - hash) + char
      hash = hash & hash // Convert to 32-bit integer
    }
    return hash.toString(36)
  }
  
  getDeduplicationStats = (): Effect.Effect<DeduplicationStats> =>
    Effect.sync(() => ({
      totalRecordsSeen: MutableHashSet.size(this.seenRecords),
      timeWindowRecords: MutableHashSet.size(this.timeWindowRecords),
      keyStrategy: this.config.keyStrategy,
      config: this.config
    }))
  
  reset = (): Effect.Effect<void> =>
    Effect.sync(() => {
      MutableHashSet.clear(this.seenRecords)
      MutableHashSet.clear(this.timeWindowRecords)
      MutableHashSet.clear(this.contentHashes)
    })
}

interface DeduplicationResult {
  type: 'unique' | 'duplicate'
  record: DataRecord
  key: string
  totalProcessed: number
}

interface DeduplicationStats {
  totalRecordsSeen: number
  timeWindowRecords: number
  keyStrategy: DeduplicationConfig['keyStrategy']
  config: DeduplicationConfig
}

// Usage in data processing pipeline
const runDeduplicationPipeline = Effect.gen(function* () {
  const deduplicator = new DataDeduplicator({
    keyStrategy: 'composite',
    timeWindow: 5000, // 5 seconds
    maxRecords: 100000
  })
  
  // Simulate data stream
  const dataStream = Stream.make(
    { id: '1', customerId: 'cust1', timestamp: new Date(), recordType: 'order' as const, amount: 100, metadata: {} },
    { id: '2', customerId: 'cust1', timestamp: new Date(), recordType: 'order' as const, amount: 100, metadata: {} }, // Duplicate
    { id: '3', customerId: 'cust2', timestamp: new Date(), recordType: 'payment' as const, amount: 200, metadata: {} }
  )
  
  // Process stream with deduplication
  const processedStream = deduplicator.processDataStream(dataStream)
  
  // Handle results
  yield* Stream.runForEach(processedStream, result => Effect.gen(function* () {
    switch (result.type) {
      case 'unique':
        yield* Effect.log(`Processing unique record: ${result.record.id}`)
        // Forward to downstream processing
        break
        
      case 'duplicate':
        yield* Effect.log(`Filtered duplicate record: ${result.record.id} (key: ${result.key})`)
        // Log or store duplicate info
        break
    }
  }))
  
  // Get final stats
  const stats = yield* deduplicator.getDeduplicationStats()
  yield* Effect.log(`Deduplication complete. Records processed: ${stats.totalRecordsSeen}`)
})
```

## Advanced Features Deep Dive

### Feature 1: Memory-Efficient Set Building

MutableHashSet excels at scenarios where you build sets incrementally without the overhead of structural copying.

#### Basic Set Building

```typescript
import { MutableHashSet, Effect } from "effect"

// Efficient incremental building
const buildUserSet = (userIds: string[]) => Effect.gen(function* () {
  const activeUsers = MutableHashSet.empty<string>()
  
  for (const userId of userIds) {
    // O(1) addition without copying
    MutableHashSet.add(activeUsers, userId)
  }
  
  return activeUsers
})
```

#### Real-World Set Building Example

```typescript
import { MutableHashSet, Effect, Stream } from "effect"

// Building feature flags set from configuration
const buildFeatureFlagsSet = (
  configs: Stream.Stream<FeatureConfig>
) => Effect.gen(function* () {
  const enabledFeatures = MutableHashSet.empty<string>()
  const userSpecificFeatures = new Map<string, MutableHashSet<string>>()
  
  yield* Stream.runForEach(configs, config => Effect.gen(function* () {
    if (config.enabled) {
      MutableHashSet.add(enabledFeatures, config.featureName)
    }
    
    // Handle user-specific features
    for (const userId of config.enabledForUsers || []) {
      if (!userSpecificFeatures.has(userId)) {
        userSpecificFeatures.set(userId, MutableHashSet.empty<string>())
      }
      
      const userFeatures = userSpecificFeatures.get(userId)!
      MutableHashSet.add(userFeatures, config.featureName)
    }
  }))
  
  return { enabledFeatures, userSpecificFeatures }
})

interface FeatureConfig {
  featureName: string
  enabled: boolean
  enabledForUsers?: string[]
  enabledForRoles?: string[]
}
```

#### Advanced Building: Conditional Set Construction

```typescript
import { MutableHashSet, Effect, Option } from "effect"

// Building sets with complex business logic
const buildAccessControlSet = (
  users: User[],
  roles: Role[],
  permissions: Permission[]
) => Effect.gen(function* () {
  const adminUsers = MutableHashSet.empty<string>()
  const restrictedResources = MutableHashSet.empty<string>()
  const highValueOperations = MutableHashSet.empty<string>()
  
  // Build admin users set
  for (const user of users) {
    const userRoles = roles.filter(role => role.userId === user.id)
    const hasAdminRole = userRoles.some(role => role.name === 'admin')
    
    if (hasAdminRole) {
      MutableHashSet.add(adminUsers, user.id)
    }
  }
  
  // Build restricted resources set
  for (const permission of permissions) {
    if (permission.restrictionLevel === 'high') {
      MutableHashSet.add(restrictedResources, permission.resourceId)
    }
    
    if (permission.valueLevel === 'high') {
      MutableHashSet.add(highValueOperations, permission.operationType)
    }
  }
  
  return {
    adminUsers,
    restrictedResources,
    highValueOperations,
    stats: {
      totalAdmins: MutableHashSet.size(adminUsers),
      totalRestrictedResources: MutableHashSet.size(restrictedResources),
      totalHighValueOps: MutableHashSet.size(highValueOperations)
    }
  }
})

interface User {
  id: string
  name: string
  email: string
}

interface Role {
  userId: string
  name: string
  grantedAt: Date
}

interface Permission {
  resourceId: string
  operationType: string
  restrictionLevel: 'low' | 'medium' | 'high'
  valueLevel: 'low' | 'medium' | 'high'
}
```

### Feature 2: Performance-Critical Membership Testing

MutableHashSet provides O(1) average lookup performance, making it ideal for high-frequency membership testing scenarios.

#### Basic Membership Testing

```typescript
import { MutableHashSet } from "effect"

// Efficient blacklist checking
const createBlacklistChecker = (blacklistedIds: string[]) => {
  const blacklist = MutableHashSet.fromIterable(blacklistedIds)
  
  return {
    isBlacklisted: (id: string): boolean => 
      MutableHashSet.has(blacklist, id),
    
    addToBlacklist: (id: string): void => {
      MutableHashSet.add(blacklist, id)
    },
    
    removeFromBlacklist: (id: string): void => {
      MutableHashSet.remove(blacklist, id)
    },
    
    getBlacklistSize: (): number => 
      MutableHashSet.size(blacklist)
  }
}
```

#### Real-World Membership Testing: Rate Limiting

```typescript
import { MutableHashSet, Effect, Schedule, Ref } from "effect"

class RateLimiter {
  private readonly currentRequests = MutableHashSet.empty<string>()
  private readonly recentViolators = MutableHashSet.empty<string>()
  
  constructor(
    private readonly maxRequestsPerWindow: number,
    private readonly windowMs: number,
    private readonly violatorCooldownMs: number
  ) {}
  
  checkRateLimit = (userId: string, requestId: string) => Effect.gen(function* () {
    const userKey = `${userId}-${Math.floor(Date.now() / this.windowMs)}`
    
    // Check if user is in violator cooldown
    if (MutableHashSet.has(this.recentViolators, userId)) {
      return yield* Effect.fail(new RateLimitViolation(
        `User ${userId} is in cooldown period`
      ))
    }
    
    // Count current requests for this user in this window
    const currentUserRequests = this.countUserRequests(userId)
    
    if (currentUserRequests >= this.maxRequestsPerWindow) {
      // Add to violators
      MutableHashSet.add(this.recentViolators, userId)
      
      // Schedule removal from violators
      yield* Effect.fork(
        Effect.delay(
          Effect.sync(() => MutableHashSet.remove(this.recentViolators, userId)),
          this.violatorCooldownMs
        )
      )
      
      return yield* Effect.fail(new RateLimitExceeded(
        `Rate limit exceeded for user ${userId}`
      ))
    }
    
    // Add request to tracking
    MutableHashSet.add(this.currentRequests, `${userKey}-${requestId}`)
    
    // Schedule cleanup of this request record
    yield* Effect.fork(
      Effect.delay(
        Effect.sync(() => MutableHashSet.remove(this.currentRequests, `${userKey}-${requestId}`)),
        this.windowMs
      )
    )
    
    return {
      allowed: true,
      remainingRequests: this.maxRequestsPerWindow - currentUserRequests - 1,
      windowResetTime: new Date(Math.ceil(Date.now() / this.windowMs) * this.windowMs)
    }
  })
  
  private countUserRequests = (userId: string): number => {
    const currentWindow = Math.floor(Date.now() / this.windowMs)
    const userPrefix = `${userId}-${currentWindow}-`
    let count = 0
    
    for (const requestKey of this.currentRequests) {
      if (requestKey.startsWith(userPrefix)) {
        count++
      }
    }
    
    return count
  }
  
  getRateLimitStats = (): Effect.Effect<RateLimitStats> =>
    Effect.sync(() => ({
      totalActiveRequests: MutableHashSet.size(this.currentRequests),
      currentViolators: MutableHashSet.size(this.recentViolators),
      maxRequestsPerWindow: this.maxRequestsPerWindow,
      windowMs: this.windowMs
    }))
  
  clearViolators = (): Effect.Effect<void> =>
    Effect.sync(() => MutableHashSet.clear(this.recentViolators))
}

class RateLimitExceeded extends Error {
  readonly _tag = 'RateLimitExceeded'
}

class RateLimitViolation extends Error {
  readonly _tag = 'RateLimitViolation'
}

interface RateLimitStats {
  totalActiveRequests: number
  currentViolators: number
  maxRequestsPerWindow: number
  windowMs: number
}

// Usage
const rateLimiter = new RateLimiter(100, 60000, 300000) // 100 requests per minute, 5 min cooldown

const handleRequest = (userId: string, requestId: string) =>
  rateLimiter.checkRateLimit(userId, requestId).pipe(
    Effect.map(result => ({ success: true, ...result })),
    Effect.catchAll(error => Effect.succeed({ 
      success: false, 
      error: error.message 
    }))
  )
```

#### Advanced Membership Testing: Multi-Level Access Control

```typescript
import { MutableHashSet, Effect, pipe } from "effect"

class AccessControlManager {
  private readonly adminUsers = MutableHashSet.empty<string>()
  private readonly moderatorUsers = MutableHashSet.empty<string>()
  private readonly suspendedUsers = MutableHashSet.empty<string>()
  private readonly restrictedResources = MutableHashSet.empty<string>()
  private readonly publicResources = MutableHashSet.empty<string>()
  
  initializeFromConfig = (config: AccessConfig) => Effect.gen(function* () {
    // Initialize sets from configuration
    config.adminUsers.forEach(userId => MutableHashSet.add(this.adminUsers, userId))
    config.moderatorUsers.forEach(userId => MutableHashSet.add(this.moderatorUsers, userId))
    config.suspendedUsers.forEach(userId => MutableHashSet.add(this.suspendedUsers, userId))
    config.restrictedResources.forEach(resource => MutableHashSet.add(this.restrictedResources, resource))
    config.publicResources.forEach(resource => MutableHashSet.add(this.publicResources, resource))
    
    yield* Effect.log('Access control initialized')
  })
  
  checkAccess = (userId: string, resourceId: string, operation: string) => Effect.gen(function* () {
    // First check: user suspension
    if (MutableHashSet.has(this.suspendedUsers, userId)) {
      return yield* Effect.fail(new AccessDenied('User is suspended'))
    }
    
    // Second check: public resources (always accessible)
    if (MutableHashSet.has(this.publicResources, resourceId)) {
      return { granted: true, level: 'public' as const }
    }
    
    // Third check: admin access (full access)
    if (MutableHashSet.has(this.adminUsers, userId)) {
      return { granted: true, level: 'admin' as const }
    }
    
    // Fourth check: restricted resources
    if (MutableHashSet.has(this.restrictedResources, resourceId)) {
      // Only admins can access restricted resources
      return yield* Effect.fail(new AccessDenied('Resource requires admin access'))
    }
    
    // Fifth check: moderator access
    if (MutableHashSet.has(this.moderatorUsers, userId)) {
      if (operation === 'write' || operation === 'delete') {
        return { granted: true, level: 'moderator' as const }
      }
    }
    
    // Default: basic read access for regular users
    if (operation === 'read') {
      return { granted: true, level: 'user' as const }
    }
    
    return yield* Effect.fail(new AccessDenied(`Operation '${operation}' not permitted`))
  })
  
  promoteUser = (userId: string, role: 'moderator' | 'admin') => Effect.gen(function* () {
    switch (role) {
      case 'moderator':
        MutableHashSet.add(this.moderatorUsers, userId)
        break
      case 'admin':
        MutableHashSet.add(this.adminUsers, userId)
        // Admins are also moderators
        MutableHashSet.add(this.moderatorUsers, userId)
        break
    }
    
    // Remove from suspended if present
    MutableHashSet.remove(this.suspendedUsers, userId)
    
    yield* Effect.log(`User ${userId} promoted to ${role}`)
  })
  
  suspendUser = (userId: string) => Effect.gen(function* () {
    MutableHashSet.add(this.suspendedUsers, userId)
    yield* Effect.log(`User ${userId} suspended`)
  })
  
  getAccessStats = (): Effect.Effect<AccessStats> =>
    Effect.sync(() => ({
      totalAdmins: MutableHashSet.size(this.adminUsers),
      totalModerators: MutableHashSet.size(this.moderatorUsers),
      totalSuspended: MutableHashSet.size(this.suspendedUsers),
      totalRestrictedResources: MutableHashSet.size(this.restrictedResources),
      totalPublicResources: MutableHashSet.size(this.publicResources)
    }))
}

interface AccessConfig {
  adminUsers: string[]
  moderatorUsers: string[]
  suspendedUsers: string[]
  restrictedResources: string[]
  publicResources: string[]
}

interface AccessStats {
  totalAdmins: number
  totalModerators: number
  totalSuspended: number
  totalRestrictedResources: number
  totalPublicResources: number
}

class AccessDenied extends Error {
  readonly _tag = 'AccessDenied'
}
```

### Feature 3: Set Operations and Comparisons

MutableHashSet supports efficient operations for combining and comparing sets.

#### Basic Set Operations with Helper Functions

```typescript
import { MutableHashSet } from "effect"

// Helper functions for set operations
const unionSets = <T>(set1: MutableHashSet<T>, set2: Iterable<T>): MutableHashSet<T> => {
  const result = MutableHashSet.fromIterable(set1)
  for (const item of set2) {
    MutableHashSet.add(result, item)
  }
  return result
}

const intersectionSets = <T>(set1: MutableHashSet<T>, set2: Iterable<T>): MutableHashSet<T> => {
  const result = MutableHashSet.empty<T>()
  for (const item of set2) {
    if (MutableHashSet.has(set1, item)) {
      MutableHashSet.add(result, item)
    }
  }
  return result
}

const differenceSets = <T>(set1: MutableHashSet<T>, set2: Iterable<T>): MutableHashSet<T> => {
  const result = MutableHashSet.fromIterable(set1)
  for (const item of set2) {
    MutableHashSet.remove(result, item)
  }
  return result
}

// Usage examples
const tags1 = MutableHashSet.make('javascript', 'typescript', 'react')
const tags2 = ['typescript', 'node', 'express']

const allTags = unionSets(tags1, tags2)
const commonTags = intersectionSets(tags1, tags2)
const uniqueToFirst = differenceSets(tags1, tags2)

console.log(Array.from(allTags)) // ['javascript', 'typescript', 'react', 'node', 'express']
console.log(Array.from(commonTags)) // ['typescript']
console.log(Array.from(uniqueToFirst)) // ['javascript', 'react']
```

## Practical Patterns & Best Practices

### Pattern 1: Local Mutation Scope

Contain mutations within limited scopes to maintain functional programming benefits while gaining performance:

```typescript
import { MutableHashSet, Effect, pipe } from "effect"

// Safe local mutation pattern
const processUserBatch = (users: User[]) => Effect.gen(function* () {
  // Create mutable set in local scope
  const processedIds = MutableHashSet.empty<string>()
  const errors: ProcessingError[] = []
  const results: ProcessedUser[] = []
  
  for (const user of users) {
    try {
      // Check for duplicates within batch
      if (MutableHashSet.has(processedIds, user.id)) {
        errors.push(new DuplicateUserError(user.id))
        continue
      }
      
      // Process user
      const processed = yield* processUser(user)
      
      // Track processed ID
      MutableHashSet.add(processedIds, user.id)
      results.push(processed)
      
    } catch (error) {
      errors.push(new UserProcessingError(user.id, error))
    }
  }
  
  // Return immutable results - mutable set doesn't escape scope
  return {
    results,
    errors,
    processedCount: MutableHashSet.size(processedIds)
  }
})

const processUser = (user: User): Effect.Effect<ProcessedUser> =>
  Effect.succeed({ ...user, processedAt: new Date() })

interface User {
  id: string
  name: string
  email: string
}

interface ProcessedUser extends User {
  processedAt: Date
}

class DuplicateUserError extends Error {
  readonly _tag = 'DuplicateUserError'
  constructor(readonly userId: string) {
    super(`Duplicate user ID: ${userId}`)
  }
}

class UserProcessingError extends Error {
  readonly _tag = 'UserProcessingError'
  constructor(readonly userId: string, readonly cause: unknown) {
    super(`Failed to process user: ${userId}`)
  }
}

type ProcessingError = DuplicateUserError | UserProcessingError
```

### Pattern 2: Efficient Batch Operations

Use MutableHashSet for efficient batch processing where multiple operations need to be performed:

```typescript
import { MutableHashSet, Effect, Array as Arr } from "effect"

// Batch deduplication helper
const createBatchDeduplicator = <T>(keyExtractor: (item: T) => string) => {
  return {
    deduplicate: (items: T[]): Effect.Effect<{ unique: T[]; duplicates: T[] }> =>
      Effect.gen(function* () {
        const seen = MutableHashSet.empty<string>()
        const unique: T[] = []
        const duplicates: T[] = []
        
        for (const item of items) {
          const key = keyExtractor(item)
          
          if (MutableHashSet.has(seen, key)) {
            duplicates.push(item)
          } else {
            MutableHashSet.add(seen, key)
            unique.push(item)
          }
        }
        
        return { unique, duplicates }
      }),
    
    deduplicateStream: <E, R>(
      stream: Stream.Stream<T, E, R>
    ): Stream.Stream<T, E, R> => {
      const seen = MutableHashSet.empty<string>()
      
      return stream.pipe(
        Stream.filter(item => {
          const key = keyExtractor(item)
          if (MutableHashSet.has(seen, key)) {
            return false
          }
          MutableHashSet.add(seen, key)
          return true
        })
      )
    }
  }
}

// Usage for different data types
const userDeduplicator = createBatchDeduplicator<User>(user => user.id)
const orderDeduplicator = createBatchDeduplicator<Order>(order => 
  `${order.customerId}-${order.timestamp}-${order.amount}`
)

interface Order {
  id: string
  customerId: string
  timestamp: number
  amount: number
}

// Process large batches efficiently
const processBulkOrders = (orders: Order[]) => Effect.gen(function* () {
  // Deduplicate orders
  const { unique, duplicates } = yield* orderDeduplicator.deduplicate(orders)
  
  yield* Effect.log(`Processing ${unique.length} unique orders, found ${duplicates.length} duplicates`)
  
  // Process unique orders
  const results = yield* Effect.all(
    unique.map(order => processOrder(order)),
    { concurrency: 10 }
  )
  
  return { results, duplicates }
})

const processOrder = (order: Order): Effect.Effect<ProcessedOrder> =>
  Effect.succeed({ ...order, processedAt: new Date() })

interface ProcessedOrder extends Order {
  processedAt: Date
}
```

### Pattern 3: Memory Management

Implement cleanup strategies to prevent unbounded memory growth:

```typescript
import { MutableHashSet, Effect, Schedule, Ref } from "effect"

// Memory-managed cache with automatic cleanup
class ManagedMutableHashSet<T> {
  constructor(
    private readonly maxSize: number,
    private readonly cleanupThreshold: number = 0.8,
    private readonly set = MutableHashSet.empty<T>(),
    private readonly insertionOrder: T[] = []
  ) {}
  
  add = (item: T): Effect.Effect<boolean> => Effect.gen(function* () {
    // Check if item already exists
    if (MutableHashSet.has(this.set, item)) {
      return false
    }
    
    // Add item
    MutableHashSet.add(this.set, item)
    this.insertionOrder.push(item)
    
    // Check if cleanup is needed
    if (MutableHashSet.size(this.set) >= this.maxSize * this.cleanupThreshold) {
      yield* this.performCleanup()
    }
    
    return true
  })
  
  has = (item: T): boolean => MutableHashSet.has(this.set, item)
  
  remove = (item: T): boolean => {
    if (MutableHashSet.has(this.set, item)) {
      MutableHashSet.remove(this.set, item)
      const index = this.insertionOrder.indexOf(item)
      if (index > -1) {
        this.insertionOrder.splice(index, 1)
      }
      return true
    }
    return false
  }
  
  size = (): number => MutableHashSet.size(this.set)
  
  clear = (): Effect.Effect<void> => Effect.sync(() => {
    MutableHashSet.clear(this.set)
    this.insertionOrder.length = 0
  })
  
  private performCleanup = (): Effect.Effect<void> => Effect.gen(function* () {
    const targetSize = Math.floor(this.maxSize * 0.6) // Remove 40% when cleaning up
    const itemsToRemove = this.insertionOrder.length - targetSize
    
    if (itemsToRemove > 0) {
      // Remove oldest items (FIFO)
      const removedItems = this.insertionOrder.splice(0, itemsToRemove)
      
      for (const item of removedItems) {
        MutableHashSet.remove(this.set, item)
      }
      
      yield* Effect.log(`Cleaned up ${itemsToRemove} items from managed set`)
    }
  })
  
  getStats = (): Effect.Effect<ManagedSetStats> => Effect.sync(() => ({
    currentSize: MutableHashSet.size(this.set),
    maxSize: this.maxSize,
    utilizationPercentage: (MutableHashSet.size(this.set) / this.maxSize) * 100,
    cleanupThreshold: this.cleanupThreshold
  }))
}

interface ManagedSetStats {
  currentSize: number
  maxSize: number
  utilizationPercentage: number
  cleanupThreshold: number
}

// Usage in session management
const sessionTracker = new ManagedMutableHashSet<string>(1000, 0.9)

const trackUserSession = (sessionId: string) => Effect.gen(function* () {
  const wasAdded = yield* sessionTracker.add(sessionId)
  
  if (wasAdded) {
    yield* Effect.log(`New session tracked: ${sessionId}`)
  } else {
    yield* Effect.log(`Session already exists: ${sessionId}`)
  }
  
  const stats = yield* sessionTracker.getStats()
  
  if (stats.utilizationPercentage > 95) {
    yield* Effect.log(`Warning: Session tracker at ${stats.utilizationPercentage.toFixed(1)}% capacity`)
  }
})
```

## Integration Examples

### Integration with Effect Streams

MutableHashSet integrates well with Effect's streaming capabilities for efficient data processing:

```typescript
import { MutableHashSet, Effect, Stream, Chunk } from "effect"

// Stream-based deduplication
const createDeduplicatingStream = <T>(
  keyExtractor: (item: T) => string
) => {
  return <E, R>(
    source: Stream.Stream<T, E, R>
  ): Stream.Stream<T, E, R> => {
    const seen = MutableHashSet.empty<string>()
    
    return source.pipe(
      Stream.filter(item => {
        const key = keyExtractor(item)
        if (MutableHashSet.has(seen, key)) {
          return false
        }
        MutableHashSet.add(seen, key)
        return true
      })
    )
  }
}

// Real-time analytics with deduplication
const processAnalyticsEvents = Effect.gen(function* () {
  // Create a stream of analytics events
  const eventStream: Stream.Stream<AnalyticsEvent> = Stream.fromIterable([
    { id: '1', type: 'page_view', userId: 'user1', timestamp: Date.now() },
    { id: '2', type: 'click', userId: 'user1', timestamp: Date.now() },
    { id: '1', type: 'page_view', userId: 'user1', timestamp: Date.now() }, // duplicate
    { id: '3', type: 'page_view', userId: 'user2', timestamp: Date.now() }
  ])
  
  // Apply deduplication
  const deduplicatedStream = createDeduplicatingStream<AnalyticsEvent>(
    event => event.id
  )(eventStream)
  
  // Process deduplicated events
  yield* Stream.runForEach(deduplicatedStream, event => 
    Effect.log(`Processing unique event: ${event.id} - ${event.type}`)
  )
})

interface AnalyticsEvent {
  id: string
  type: string
  userId: string
  timestamp: number
}
```

### Integration with Effect Caching

Combine MutableHashSet with Effect's caching for efficient cache invalidation:

```typescript
import { MutableHashSet, Effect, Cache, Duration } from "effect"

// Cache with tag-based invalidation
class TaggedCache<K, V> {
  private readonly taggedKeys = new Map<string, MutableHashSet<K>>()
  
  constructor(
    private readonly cache: Cache.Cache<K, V>,
    private readonly defaultTags: string[] = []
  ) {}
  
  static make = <K, V>(
    lookup: (key: K) => Effect.Effect<V>,
    capacity: number = 1000,
    timeToLive: Duration.Duration = Duration.minutes(30)
  ): Effect.Effect<TaggedCache<K, V>> => Effect.gen(function* () {
    const cache = yield* Cache.make({
      capacity,
      timeToLive,
      lookup
    })
    
    return new TaggedCache(cache, [])
  })
  
  get = (key: K, tags: string[] = []): Effect.Effect<V> => Effect.gen(function* () {
    // Track key for each tag
    const allTags = [...this.defaultTags, ...tags]
    for (const tag of allTags) {
      if (!this.taggedKeys.has(tag)) {
        this.taggedKeys.set(tag, MutableHashSet.empty<K>())
      }
      
      const taggedSet = this.taggedKeys.get(tag)!
      MutableHashSet.add(taggedSet, key)
    }
    
    return yield* this.cache.get(key)
  })
  
  invalidateByTag = (tag: string): Effect.Effect<number> => Effect.gen(function* () {
    const taggedSet = this.taggedKeys.get(tag)
    if (!taggedSet) {
      return 0
    }
    
    let invalidatedCount = 0
    
    // Invalidate all keys with this tag
    for (const key of taggedSet) {
      yield* this.cache.invalidate(key)
      invalidatedCount++
    }
    
    // Clear the tag set
    MutableHashSet.clear(taggedSet)
    
    return invalidatedCount
  })
  
  invalidateByTags = (tags: string[]): Effect.Effect<number> => Effect.gen(function* () {
    let totalInvalidated = 0
    
    for (const tag of tags) {
      const count = yield* this.invalidateByTag(tag)
      totalInvalidated += count
    }
    
    return totalInvalidated
  })
  
  getCacheStats = (): Effect.Effect<TaggedCacheStats> => Effect.gen(function* () {
    const cacheStats = yield* this.cache.cacheStats
    
    const tagStats = new Map<string, number>()
    for (const [tag, keys] of this.taggedKeys) {
      tagStats.set(tag, MutableHashSet.size(keys))
    }
    
    return {
      ...cacheStats,
      totalTags: this.taggedKeys.size,
      tagStats: Object.fromEntries(tagStats)
    }
  })
}

interface TaggedCacheStats extends Cache.CacheStats {
  totalTags: number
  tagStats: Record<string, number>
}

// Usage in API caching
const createUserCache = () => Effect.gen(function* () {
  const userCache = yield* TaggedCache.make<string, User>(
    userId => fetchUserFromDatabase(userId),
    500,
    Duration.minutes(15)
  )
  
  return {
    getUser: (userId: string, tags: string[] = []) =>
      userCache.get(userId, ['users', `user:${userId}`, ...tags]),
    
    invalidateUserCache: (userId: string) =>
      userCache.invalidateByTag(`user:${userId}`),
    
    invalidateAllUsers: () =>
      userCache.invalidateByTag('users'),
    
    getCacheStats: () =>
      userCache.getCacheStats()
  }
})

const fetchUserFromDatabase = (userId: string): Effect.Effect<User> =>
  Effect.gen(function* () {
    yield* Effect.log(`Fetching user ${userId} from database`)
    // Simulate database fetch
    yield* Effect.sleep(Duration.millis(100))
    return {
      id: userId,
      name: `User ${userId}`,
      email: `user${userId}@example.com`
    }
  })
```

### Testing Strategies

Comprehensive testing patterns for MutableHashSet-based components:

```typescript
import { MutableHashSet, Effect, TestServices, TestClock, Duration } from "effect"
import { describe, it, expect } from "vitest"

// Helper for testing mutable set operations
const testMutableSetOperations = () => {
  describe("MutableHashSet Operations", () => {
    it("should handle basic add/remove operations", () => 
      Effect.gen(function* () {
        const set = MutableHashSet.empty<string>()
        
        // Test additions
        MutableHashSet.add(set, "item1")
        MutableHashSet.add(set, "item2")
        MutableHashSet.add(set, "item1") // duplicate
        
        expect(MutableHashSet.size(set)).toBe(2)
        expect(MutableHashSet.has(set, "item1")).toBe(true)
        expect(MutableHashSet.has(set, "item2")).toBe(true)
        expect(MutableHashSet.has(set, "item3")).toBe(false)
        
        // Test removal
        MutableHashSet.remove(set, "item1")
        expect(MutableHashSet.size(set)).toBe(1)
        expect(MutableHashSet.has(set, "item1")).toBe(false)
        
        // Test clear
        MutableHashSet.clear(set)
        expect(MutableHashSet.size(set)).toBe(0)
      }).pipe(Effect.runSync)
    )
    
    it("should handle concurrent access patterns", () => 
      Effect.gen(function* () {
        const set = MutableHashSet.empty<number>()
        
        // Simulate concurrent additions
        const effects = Array.from({ length: 100 }, (_, i) =>
          Effect.sync(() => MutableHashSet.add(set, i % 50)) // Create some duplicates
        )
        
        yield* Effect.all(effects, { concurrency: 10 })
        
        // Should have 50 unique items (0-49)
        expect(MutableHashSet.size(set)).toBe(50)
        
        // Verify all expected items are present
        for (let i = 0; i < 50; i++) {
          expect(MutableHashSet.has(set, i)).toBe(true)
        }
      }).pipe(Effect.runPromise)
    )
  })
}

// Test deduplication performance
const testDeduplicationPerformance = () => {
  describe("Deduplication Performance", () => {
    it("should efficiently handle large datasets", () => 
      Effect.gen(function* () {
        const startTime = Date.now()
        const set = MutableHashSet.empty<string>()
        
        // Add 10,000 items with 50% duplicates
        for (let i = 0; i < 10000; i++) {
          MutableHashSet.add(set, `item${i % 5000}`)
        }
        
        const endTime = Date.now()
        const duration = endTime - startTime
        
        expect(MutableHashSet.size(set)).toBe(5000)
        expect(duration).toBeLessThan(100) // Should complete in under 100ms
        
        yield* Effect.log(`Processed 10,000 items with deduplication in ${duration}ms`)
      }).pipe(
        Effect.provide(TestServices.TestServices),
        Effect.runPromise
      )
    )
  })
}

// Test memory management patterns
const testMemoryManagement = () => {
  describe("Memory Management", () => {
    it("should properly clean up managed sets", () => 
      Effect.gen(function* () {
        const managedSet = new ManagedMutableHashSet<string>(100, 0.8)
        
        // Fill set to trigger cleanup
        for (let i = 0; i < 150; i++) {
          yield* managedSet.add(`item${i}`)
        }
        
        const stats = yield* managedSet.getStats()
        
        // Should have triggered cleanup
        expect(stats.currentSize).toBeLessThan(100)
        expect(stats.utilizationPercentage).toBeLessThan(80)
        
        yield* Effect.log(`Set cleaned up to ${stats.currentSize} items`)
      }).pipe(Effect.runPromise)
    )
  })
}

// Property-based testing helper
const testSetProperties = () => {
  describe("Set Properties", () => {
    it("should maintain set properties under all operations", () => 
      Effect.gen(function* () {
        const set = MutableHashSet.empty<number>()
        const operations: Array<{ type: 'add' | 'remove', value: number }> = []
        
        // Generate random operations
        for (let i = 0; i < 1000; i++) {
          const type = Math.random() > 0.5 ? 'add' : 'remove'
          const value = Math.floor(Math.random() * 100)
          operations.push({ type, value })
        }
        
        // Track expected unique values
        const expectedSet = new Set<number>()
        
        // Apply operations
        for (const op of operations) {
          if (op.type === 'add') {
            MutableHashSet.add(set, op.value)
            expectedSet.add(op.value)
          } else {
            MutableHashSet.remove(set, op.value)
            expectedSet.delete(op.value)
          }
        }
        
        // Verify size matches expected
        expect(MutableHashSet.size(set)).toBe(expectedSet.size)
        
        // Verify all expected items are present
        for (const value of expectedSet) {
          expect(MutableHashSet.has(set, value)).toBe(true)
        }
        
        // Verify no unexpected items
        const actualValues = Array.from(set)
        expect(actualValues.length).toBe(expectedSet.size)
        
        for (const value of actualValues) {
          expect(expectedSet.has(value)).toBe(true)
        }
      }).pipe(Effect.runPromise)
    )
  })
}

// Run all tests
export const runMutableHashSetTests = () => {
  testMutableSetOperations()
  testDeduplicationPerformance()
  testMemoryManagement()
  testSetProperties()
}
```

## Conclusion

MutableHashSet provides efficient, in-place set operations for performance-critical scenarios while maintaining Effect's type safety and composability benefits.

Key benefits:
- **Performance Optimization**: O(1) average operations with no copying overhead for incremental modifications
- **Memory Efficiency**: In-place mutations reduce memory allocation and garbage collection pressure
- **Controlled Mutability**: Localized mutations within functional contexts preserve overall immutability guarantees

Use MutableHashSet when you need high-performance set operations in controlled scopes, efficient deduplication for large datasets, or memory-conscious caching and tracking systems. For general-purpose immutable set operations, prefer HashSet with its structural sharing and immutability guarantees.