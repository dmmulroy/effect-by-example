# NonEmptyIterable: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem NonEmptyIterable Solves

Working with collections in JavaScript often leads to runtime errors when assuming collections contain elements. Traditional approaches require manual length checks and defensive programming throughout your codebase:

```typescript
// Traditional approach - runtime errors waiting to happen
function processUserBatch(users: User[]): ProcessedUser {
  // What if users is empty? Runtime error!
  const firstUser = users[0] // Could be undefined
  const restUsers = users.slice(1)
  
  // Manual length checking everywhere
  if (users.length === 0) {
    throw new Error("Cannot process empty user batch")
  }
  
  // More defensive programming
  const leadUser = users[0]
  if (!leadUser) {
    throw new Error("Lead user is undefined")
  }
  
  return {
    leadUser: processUser(leadUser),
    followers: restUsers.map(processUser),
    batchSize: users.length
  }
}

function getRandomChoice<T>(items: T[]): T {
  if (items.length === 0) {
    throw new Error("Cannot choose from empty array")
  }
  return items[Math.floor(Math.random() * items.length)]
}

// Usage requires constant defensive checks
const users = await fetchUsers()
if (users.length > 0) {
  const processed = processUserBatch(users)
  const randomUser = getRandomChoice(users)
} else {
  // Handle empty case
  console.log("No users to process")
}
```

This approach leads to:
- **Runtime Errors** - Accessing first element of empty collections crashes the application
- **Defensive Programming** - Excessive length checking throughout the codebase
- **Type Uncertainty** - No compile-time guarantee that collections contain elements
- **Boilerplate Code** - Manual empty checks scattered across business logic
- **Hidden Assumptions** - Functions assume non-empty input but don't enforce it

### The NonEmptyIterable Solution

NonEmptyIterable provides a type-safe guarantee that a collection contains at least one element, eliminating empty collection errors at compile time:

```typescript
import { NonEmptyIterable, Chunk, Array as Arr } from "effect"

// Type-safe processing - compiler guarantees non-empty input
function processUserBatch(users: NonEmptyIterable.NonEmptyIterable<User>): ProcessedUser {
  // Safe to extract first element - guaranteed to exist
  const [leadUser, restIterator] = NonEmptyIterable.unprepend(users)
  const restUsers = Array.from(restIterator)
  
  return {
    leadUser: processUser(leadUser),
    followers: restUsers.map(processUser),
    batchSize: 1 + restUsers.length
  }
}

// Type-safe random choice with Effect's Random service
const getRandomChoice = <T>(items: NonEmptyIterable.NonEmptyIterable<T>) =>
  Random.choice(items) // No runtime error possible

// Usage is safe and composable
const processUsers = Effect.gen(function* () {
  const users = yield* fetchUsers()
  
  // Convert to NonEmptyIterable only if non-empty
  const nonEmptyUsers = Arr.isNonEmptyArray(users) ? users : null
  
  if (nonEmptyUsers) {
    const processed = processUserBatch(nonEmptyUsers)
    const randomUser = yield* getRandomChoice(nonEmptyUsers)
    return { processed, randomUser }
  }
  
  return { processed: null, randomUser: null }
})
```

### Key Concepts

**NonEmptyIterable Interface**: A type that extends `Iterable<A>` with a compile-time guarantee of containing at least one element

**Type Safety**: The compiler prevents empty collection errors by ensuring only non-empty collections are accepted

**Zero Runtime Overhead**: NonEmptyIterable is a pure TypeScript interface with no runtime cost

## Basic Usage Patterns

### Pattern 1: Type Narrowing with Guards

```typescript
import { Array as Arr, NonEmptyIterable } from "effect"

// Safe narrowing from regular arrays
function processIfNonEmpty<T>(items: T[]): string {
  if (Arr.isNonEmptyArray(items)) {
    // Type narrowed to NonEmptyArray, which extends NonEmptyIterable
    const [first, restIterator] = NonEmptyIterable.unprepend(items)
    const rest = Array.from(restIterator)
    return `First: ${first}, Rest: ${rest.length} items`
  }
  return "Empty collection"
}

// Usage
const users = ["Alice", "Bob", "Charlie"]
const result = processIfNonEmpty(users) // "First: Alice, Rest: 2 items"

const emptyUsers: string[] = []
const emptyResult = processIfNonEmpty(emptyUsers) // "Empty collection"
```

### Pattern 2: Safe Element Extraction

```typescript
import { NonEmptyIterable, Chunk } from "effect"

// Extract first element safely
function getFirstAndRest<T>(items: NonEmptyIterable.NonEmptyIterable<T>) {
  const [first, restIterator] = NonEmptyIterable.unprepend(items)
  const rest = Array.from(restIterator)
  
  return {
    first,
    rest,
    hasMore: rest.length > 0
  }
}

// Works with any NonEmptyIterable implementation
const chunk = Chunk.make(1, 2, 3, 4, 5)
const result = getFirstAndRest(chunk)
// { first: 1, rest: [2, 3, 4, 5], hasMore: true }

const singleItem = Chunk.make("only")
const singleResult = getFirstAndRest(singleItem)
// { first: "only", rest: [], hasMore: false }
```

### Pattern 3: Creating NonEmptyIterable Types

```typescript
import { Chunk, Array as Arr } from "effect"

// Create guaranteed non-empty collections
const nonEmptyChunk = Chunk.make("a", "b", "c") // NonEmptyChunk<string>
const nonEmptyArray = Arr.make(1, 2, 3) // NonEmptyArray<number>

// Convert existing collections safely
function toNonEmptyChunk<T>(items: T[]): Chunk.Chunk<T> {
  return Chunk.fromIterable(items)
}

// Usage with type checking
const items = ["apple", "banana", "orange"]
const chunk = toNonEmptyChunk(items)

if (Chunk.isNonEmpty(chunk)) {
  // Type narrowed to NonEmptyChunk, which extends NonEmptyIterable
  const [first] = NonEmptyIterable.unprepend(chunk)
  console.log(`First fruit: ${first}`)
}
```

## Real-World Examples

### Example 1: Batch Processing System

A data processing system that handles batches of records, ensuring each batch contains at least one item:

```typescript
import { Effect, NonEmptyIterable, Array as Arr, pipe } from "effect"

interface ProcessingJob {
  id: string
  data: unknown
  priority: number
}

interface BatchResult {
  leadJob: ProcessingJob
  batchSize: number
  processingTime: number
  successCount: number
}

// Type-safe batch processor
const processBatch = (jobs: NonEmptyIterable.NonEmptyIterable<ProcessingJob>) =>
  Effect.gen(function* () {
    const startTime = yield* Effect.clockWith(clock => clock.currentTimeMillis)
    
    // Safe to extract first element
    const [leadJob, restIterator] = NonEmptyIterable.unprepend(jobs)
    const restJobs = Array.from(restIterator)
    const allJobs = [leadJob, ...restJobs]
    
    // Process all jobs
    const results = yield* Effect.all(
      allJobs.map(job => 
        processJob(job).pipe(
          Effect.catchAll(() => Effect.succeed(null))
        )
      )
    )
    
    const endTime = yield* Effect.clockWith(clock => clock.currentTimeMillis)
    const successCount = results.filter(r => r !== null).length
    
    return {
      leadJob,
      batchSize: allJobs.length,
      processingTime: endTime - startTime,
      successCount
    }
  })

const processJob = (job: ProcessingJob): Effect.Effect<unknown, string> =>
  Effect.gen(function* () {
    // Simulate processing
    yield* Effect.sleep(100)
    
    if (job.priority < 1) {
      return yield* Effect.fail("Invalid priority")
    }
    
    return { jobId: job.id, processed: true }
  })

// Batch manager that ensures non-empty batches
const processingQueue = Effect.gen(function* () {
  const allJobs = yield* fetchJobs()
  
  // Group jobs into batches of 5
  const batches = pipe(
    allJobs,
    Arr.chunksOf(5),
    Arr.filter(Arr.isNonEmptyArray) // Only process non-empty batches
  )
  
  const results = yield* Effect.all(
    batches.map(batch => processBatch(batch))
  )
  
  return {
    totalBatches: batches.length,
    totalJobs: allJobs.length,
    results
  }
})

const fetchJobs = (): Effect.Effect<ProcessingJob[]> =>
  Effect.succeed([
    { id: "job-1", data: "data1", priority: 2 },
    { id: "job-2", data: "data2", priority: 1 },
    { id: "job-3", data: "data3", priority: 3 }
  ])
```

### Example 2: Configuration Validation System

A system that validates configuration files, ensuring required sections are present:

```typescript
import { Effect, NonEmptyIterable, Array as Arr, pipe } from "effect"

interface ConfigSection {
  name: string
  required: boolean
  settings: Record<string, unknown>
}

interface ValidationResult {
  primarySection: ConfigSection
  additionalSections: ConfigSection[]
  isValid: boolean
  warnings: string[]
}

// Validate configuration with guaranteed non-empty sections
const validateConfig = (sections: NonEmptyIterable.NonEmptyIterable<ConfigSection>) =>
  Effect.gen(function* () {
    const [primarySection, restIterator] = NonEmptyIterable.unprepend(sections)
    const additionalSections = Array.from(restIterator)
    
    const warnings: string[] = []
    
    // Validate primary section
    if (!primarySection.required) {
      warnings.push("Primary section should be required")
    }
    
    // Validate additional sections
    const validationResults = yield* Effect.all(
      additionalSections.map(section =>
        validateSection(section).pipe(
          Effect.catchAll(error => Effect.succeed(`Section ${section.name}: ${error}`))
        )
      )
    )
    
    const sectionWarnings = validationResults.filter(r => typeof r === 'string') as string[]
    warnings.push(...sectionWarnings)
    
    return {
      primarySection,
      additionalSections,
      isValid: warnings.length === 0,
      warnings
    }
  })

const validateSection = (section: ConfigSection): Effect.Effect<ConfigSection, string> =>
  Effect.gen(function* () {
    if (Object.keys(section.settings).length === 0) {
      return yield* Effect.fail("Empty settings")
    }
    
    if (section.required && !section.settings.enabled) {
      return yield* Effect.fail("Required section must be enabled")
    }
    
    return section
  })

// Configuration loader with type safety
const loadAndValidateConfig = Effect.gen(function* () {
  const configData = yield* loadConfigFile()
  
  if (Arr.isNonEmptyArray(configData.sections)) {
    const validationResult = yield* validateConfig(configData.sections)
    
    if (validationResult.isValid) {
      yield* Effect.logInfo("Configuration is valid")
    } else {
      yield* Effect.logWarning(`Configuration warnings: ${validationResult.warnings.join(", ")}`)
    }
    
    return validationResult
  }
  
  return yield* Effect.fail("Configuration must contain at least one section")
})

const loadConfigFile = (): Effect.Effect<{ sections: ConfigSection[] }> =>
  Effect.succeed({
    sections: [
      { name: "database", required: true, settings: { host: "localhost", port: 5432 } },
      { name: "cache", required: false, settings: { ttl: 3600 } }
    ]
  })
```

### Example 3: Tournament Bracket System

A tournament system that requires at least one participant:

```typescript
import { Effect, NonEmptyIterable, Random, Array as Arr, pipe } from "effect"

interface Player {
  id: string
  name: string
  skill: number
}

interface Match {
  player1: Player
  player2: Player
  winner?: Player
}

interface Tournament {
  champion: Player
  matches: Match[]
  rounds: number
}

// Create tournament with guaranteed participants
const createTournament = (players: NonEmptyIterable.NonEmptyIterable<Player>) =>
  Effect.gen(function* () {
    const [firstPlayer, restIterator] = NonEmptyIterable.unprepend(players)
    const restPlayers = Array.from(restIterator)
    const allPlayers = [firstPlayer, ...restPlayers]
    
    if (allPlayers.length === 1) {
      // Single player tournament
      return {
        champion: firstPlayer,
        matches: [],
        rounds: 0
      }
    }
    
    // Create tournament bracket
    const shuffledPlayers = yield* shufflePlayers(allPlayers)
    const tournament = yield* runTournament(shuffledPlayers)
    
    return tournament
  })

const shufflePlayers = (players: Player[]): Effect.Effect<Player[]> =>
  Effect.gen(function* () {
    const shuffled = [...players]
    for (let i = shuffled.length - 1; i > 0; i--) {
      const j = yield* Random.nextIntBetween(0, i + 1)
      ;[shuffled[i], shuffled[j]] = [shuffled[j], shuffled[i]]
    }
    return shuffled
  })

const runTournament = (players: Player[]): Effect.Effect<Tournament> =>
  Effect.gen(function* () {
    let currentRound = players
    const allMatches: Match[] = []
    let rounds = 0
    
    while (currentRound.length > 1) {
      const roundMatches: Match[] = []
      const winners: Player[] = []
      
      // Pair up players for matches
      for (let i = 0; i < currentRound.length - 1; i += 2) {
        const player1 = currentRound[i]
        const player2 = currentRound[i + 1]
        
        const match = yield* simulateMatch(player1, player2)
        roundMatches.push(match)
        
        if (match.winner) {
          winners.push(match.winner)
        }
      }
      
      // Handle odd number of players (bye)
      if (currentRound.length % 2 === 1) {
        winners.push(currentRound[currentRound.length - 1])
      }
      
      allMatches.push(...roundMatches)
      currentRound = winners
      rounds++
    }
    
    return {
      champion: currentRound[0],
      matches: allMatches,
      rounds
    }
  })

const simulateMatch = (player1: Player, player2: Player): Effect.Effect<Match> =>
  Effect.gen(function* () {
    // Simulate match based on skill levels
    const player1Chance = player1.skill / (player1.skill + player2.skill)
    const random = yield* Random.next
    
    const winner = random < player1Chance ? player1 : player2
    
    return {
      player1,
      player2,
      winner
    }
  })

// Tournament organizer with type safety
const organizeTournament = Effect.gen(function* () {
  const registeredPlayers = yield* getRegisteredPlayers()
  
  if (Arr.isNonEmptyArray(registeredPlayers)) {
    yield* Effect.logInfo(`Starting tournament with ${registeredPlayers.length} players`)
    
    const tournament = yield* createTournament(registeredPlayers)
    
    yield* Effect.logInfo(`Tournament complete! Champion: ${tournament.champion.name}`)
    yield* Effect.logInfo(`Total rounds: ${tournament.rounds}, Total matches: ${tournament.matches.length}`)
    
    return tournament
  }
  
  return yield* Effect.fail("Cannot start tournament without players")
})

const getRegisteredPlayers = (): Effect.Effect<Player[]> =>
  Effect.succeed([
    { id: "1", name: "Alice", skill: 80 },
    { id: "2", name: "Bob", skill: 75 },
    { id: "3", name: "Charlie", skill: 85 },
    { id: "4", name: "Diana", skill: 90 }
  ])
```

## Advanced Features Deep Dive

### Feature 1: Type-Level Guarantees

NonEmptyIterable provides compile-time safety without runtime overhead:

#### Understanding the Interface

```typescript
import { NonEmptyIterable } from "effect"

// The interface is minimal but powerful
interface NonEmptyIterable<out A> extends Iterable<A> {
  readonly [nonEmpty]: A // Brand property for type safety
}

// Type-level brand prevents empty collections
function processNonEmpty<T>(items: NonEmptyIterable.NonEmptyIterable<T>): T {
  const [first] = NonEmptyIterable.unprepend(items)
  return first // Guaranteed to exist
}

// This would be a compile error:
// processNonEmpty([]) // Type error: empty array not assignable to NonEmptyIterable
```

#### Advanced Type Narrowing

```typescript
import { Array as Arr, Chunk, List } from "effect"

// Generic function that works with any NonEmptyIterable
function analyzeCollection<T>(
  items: NonEmptyIterable.NonEmptyIterable<T>
): { first: T; type: string; size: number } {
  const [first, restIterator] = NonEmptyIterable.unprepend(items)
  const rest = Array.from(restIterator)
  
  // Determine the actual collection type
  let type: string
  if (Arr.isArray(items)) {
    type = "Array"
  } else if (Chunk.isChunk(items)) {
    type = "Chunk"
  } else if (List.isList(items)) {
    type = "List"
  } else {
    type = "Unknown"
  }
  
  return {
    first,
    type,
    size: 1 + rest.length
  }
}

// Usage with different collection types
const arrayResult = analyzeCollection(Arr.make(1, 2, 3))
// { first: 1, type: "Array", size: 3 }

const chunkResult = analyzeCollection(Chunk.make("a", "b"))
// { first: "a", type: "Chunk", size: 2 }
```

### Feature 2: Integration with Effect Ecosystem

NonEmptyIterable integrates seamlessly with other Effect modules:

#### With Random Module

```typescript
import { Effect, Random, NonEmptyIterable, Array as Arr } from "effect"

// Type-safe random selection
const selectRandomItems = <T>(
  items: NonEmptyIterable.NonEmptyIterable<T>,
  count: number
) => Effect.gen(function* () {
  const [first, restIterator] = NonEmptyIterable.unprepend(items)
  const rest = Array.from(restIterator)
  const allItems = [first, ...rest]
  
  if (count >= allItems.length) {
    return allItems
  }
  
  const selected: T[] = []
  const availableItems = [...allItems]
  
  for (let i = 0; i < count; i++) {
    const randomIndex = yield* Random.nextIntBetween(0, availableItems.length)
    const selectedItem = availableItems.splice(randomIndex, 1)[0]
    selected.push(selectedItem)
  }
  
  return selected
})

// Usage
const colors = Arr.make("red", "blue", "green", "yellow", "purple")
const randomColors = selectRandomItems(colors, 3)
```

#### With Stream Module

```typescript
import { Effect, Stream, NonEmptyIterable, Array as Arr } from "effect"

// Create streams from NonEmptyIterable
const processAsStream = <T, R, E>(
  items: NonEmptyIterable.NonEmptyIterable<T>,
  processor: (item: T) => Effect.Effect<R, E>
) => Effect.gen(function* () {
  const [first, restIterator] = NonEmptyIterable.unprepend(items)
  const rest = Array.from(restIterator)
  const allItems = [first, ...rest]
  
  const stream = Stream.fromIterable(allItems).pipe(
    Stream.mapEffect(processor),
    Stream.runCollect
  )
  
  return yield* stream
})

// Usage with guaranteed non-empty processing
const numbers = Arr.make(1, 2, 3, 4, 5)
const doubledNumbers = processAsStream(numbers, (n) => Effect.succeed(n * 2))
```

## Practical Patterns & Best Practices

### Pattern 1: Safe Collection Transformations

```typescript
import { NonEmptyIterable, Array as Arr, Chunk, pipe } from "effect"

// Helper to maintain non-empty guarantee through transformations
const mapNonEmpty = <A, B>(
  items: NonEmptyIterable.NonEmptyIterable<A>,
  f: (item: A) => B
): Chunk.NonEmptyChunk<B> => {
  const [first, restIterator] = NonEmptyIterable.unprepend(items)
  const rest = Array.from(restIterator)
  
  return Chunk.make(f(first), ...rest.map(f))
}

// Helper to filter while preserving non-empty guarantee when possible
const filterNonEmpty = <A>(
  items: NonEmptyIterable.NonEmptyIterable<A>,
  predicate: (item: A) => boolean
): Chunk.Chunk<A> => {
  const [first, restIterator] = NonEmptyIterable.unprepend(items)
  const rest = Array.from(restIterator)
  const allItems = [first, ...rest]
  
  return Chunk.fromIterable(allItems.filter(predicate))
}

// Usage patterns
const numbers = Arr.make(1, 2, 3, 4, 5)
const doubled = mapNonEmpty(numbers, n => n * 2) // Always non-empty
const evens = filterNonEmpty(numbers, n => n % 2 === 0) // May be empty
```

### Pattern 2: Accumulator Pattern

```typescript
import { NonEmptyIterable, Effect } from "effect"

// Safe accumulation with guaranteed starting value
const accumulateNonEmpty = <A, B>(
  items: NonEmptyIterable.NonEmptyIterable<A>,
  initial: B,
  f: (acc: B, item: A) => B
): B => {
  const [first, restIterator] = NonEmptyIterable.unprepend(items)
  const rest = Array.from(restIterator)
  
  let acc = f(initial, first)
  for (const item of rest) {
    acc = f(acc, item)
  }
  
  return acc
}

// Effectful accumulation
const accumulateEffect = <A, B, R, E>(
  items: NonEmptyIterable.NonEmptyIterable<A>,
  initial: B,
  f: (acc: B, item: A) => Effect.Effect<B, E, R>
): Effect.Effect<B, E, R> => Effect.gen(function* () {
  const [first, restIterator] = NonEmptyIterable.unprepend(items)
  const rest = Array.from(restIterator)
  
  let acc = yield* f(initial, first)
  for (const item of rest) {
    acc = yield* f(acc, item)
  }
  
  return acc
})

// Usage examples
const words = Arr.make("hello", "world", "effect")
const concatenated = accumulateNonEmpty(words, "", (acc, word) => acc + " " + word)
// Result: " hello world effect"

const sumWithLogging = accumulateEffect(
  Arr.make(1, 2, 3, 4),
  0,
  (acc, num) => Effect.gen(function* () {
    yield* Effect.logInfo(`Adding ${num} to ${acc}`)
    return acc + num
  })
)
```

### Pattern 3: Safe Parallel Processing

```typescript
import { Effect, NonEmptyIterable, Array as Arr } from "effect"

// Process items in parallel with guaranteed non-empty input
const processParallel = <A, B, R, E>(
  items: NonEmptyIterable.NonEmptyIterable<A>,
  processor: (item: A) => Effect.Effect<B, E, R>,
  options?: { concurrency?: number | "unbounded" }
) => Effect.gen(function* () {
  const [first, restIterator] = NonEmptyIterable.unprepend(items)
  const rest = Array.from(restIterator)
  const allItems = [first, ...rest]
  
  const results = yield* Effect.all(
    allItems.map(processor),
    { concurrency: options?.concurrency ?? "unbounded" }
  )
  
  return results
})

// Batch processing with size limits
const processBatches = <A, B, R, E>(
  items: NonEmptyIterable.NonEmptyIterable<A>,
  processor: (batch: NonEmptyIterable.NonEmptyIterable<A>) => Effect.Effect<B, E, R>,
  batchSize: number = 10
) => Effect.gen(function* () {
  const [first, restIterator] = NonEmptyIterable.unprepend(items)
  const rest = Array.from(restIterator)
  const allItems = [first, ...rest]
  
  const batches = pipe(
    allItems,
    Arr.chunksOf(batchSize),
    Arr.filter(Arr.isNonEmptyArray)
  )
  
  const results = yield* Effect.all(
    batches.map(processor),
    { concurrency: 3 }
  )
  
  return results
})

// Usage
const userIds = Arr.make("user1", "user2", "user3", "user4", "user5")

const processedUsers = processParallel(
  userIds,
  (userId) => Effect.gen(function* () {
    yield* Effect.sleep(100) // Simulate API call
    return { id: userId, processed: true }
  }),
  { concurrency: 2 }
)
```

## Integration Examples

### Integration with Schema Validation

```typescript
import { Effect, NonEmptyIterable, Array as Arr, Schema } from "effect"

// Schema for validating non-empty collections
const NonEmptyStringArraySchema = Schema.Array(Schema.String).pipe(
  Schema.filter((arr): arr is Arr.NonEmptyArray<string> => 
    Arr.isNonEmptyArray(arr), 
    {
      message: () => "Array must not be empty"
    }
  )
)

// Validate and process non-empty input
const validateAndProcess = (input: unknown) => Effect.gen(function* () {
  const validatedArray = yield* Schema.decodeUnknown(NonEmptyStringArraySchema)(input)
  
  // Now we have a guaranteed non-empty array
  const [first, restIterator] = NonEmptyIterable.unprepend(validatedArray)
  const rest = Array.from(restIterator)
  
  return {
    primary: first.toUpperCase(),
    secondary: rest.map(s => s.toLowerCase()),
    total: 1 + rest.length
  }
})

// Usage
const validInput = ["Apple", "Banana", "Cherry"]
const result = validateAndProcess(validInput)

const invalidInput: string[] = []
const errorResult = validateAndProcess(invalidInput) // Will fail with validation error
```

### Integration with Database Operations

```typescript
import { Effect, NonEmptyIterable, Array as Arr, pipe } from "effect"

interface User {
  id: string
  email: string
  name: string
}

interface DatabaseService {
  readonly insertUsers: (users: NonEmptyIterable.NonEmptyIterable<User>) => Effect.Effect<User[]>
  readonly findUsersByIds: (ids: NonEmptyIterable.NonEmptyIterable<string>) => Effect.Effect<User[]>
}

// Safe database operations with guaranteed non-empty input
const makeDatabaseService = (): DatabaseService => ({
  insertUsers: (users) => Effect.gen(function* () {
    const [firstUser, restIterator] = NonEmptyIterable.unprepend(users)
    const restUsers = Array.from(restIterator)
    const allUsers = [firstUser, ...restUsers]
    
    yield* Effect.logInfo(`Inserting batch of ${allUsers.length} users`)
    
    // Simulate database insertion
    yield* Effect.sleep(100 * allUsers.length)
    
    return allUsers
  }),
  
  findUsersByIds: (ids) => Effect.gen(function* () {
    const [firstId, restIterator] = NonEmptyIterable.unprepend(ids)
    const restIds = Array.from(restIterator)
    const allIds = [firstId, ...restIds]
    
    yield* Effect.logInfo(`Finding users with IDs: ${allIds.join(", ")}`)
    
    // Simulate database query
    yield* Effect.sleep(50 * allIds.length)
    
    return allIds.map(id => ({
      id,
      email: `user${id}@example.com`,
      name: `User ${id}`
    }))
  })
})

// Bulk user operations
const bulkUserOperations = Effect.gen(function* () {
  const database = makeDatabaseService()
  
  const newUsers = Arr.make(
    { id: "1", email: "alice@example.com", name: "Alice" },
    { id: "2", email: "bob@example.com", name: "Bob" },
    { id: "3", email: "charlie@example.com", name: "Charlie" }
  )
  
  // Insert users (guaranteed non-empty)
  const insertedUsers = yield* database.insertUsers(newUsers)
  
  // Find users by IDs
  const userIds = pipe(
    insertedUsers,
    Arr.map(user => user.id),
    // Safe conversion since insertedUsers came from non-empty input
    ids => ids as Arr.NonEmptyArray<string>
  )
  
  const foundUsers = yield* database.findUsersByIds(userIds)
  
  return {
    inserted: insertedUsers.length,
    found: foundUsers.length
  }
})
```

### Testing Strategies

```typescript
import { Effect, NonEmptyIterable, Array as Arr, Chunk } from "effect"
import { describe, it, expect } from "vitest"

// Test utilities for NonEmptyIterable
const createTestData = <T>(...items: [T, ...T[]]): Arr.NonEmptyArray<T> => 
  Arr.make(...items)

const expectNonEmpty = <T>(items: Iterable<T>): NonEmptyIterable.NonEmptyIterable<T> => {
  const array = Array.from(items)
  if (array.length === 0) {
    throw new Error("Expected non-empty collection")
  }
  return array as Arr.NonEmptyArray<T>
}

// Property-based testing helper
const testWithNonEmptyData = <T>(
  generator: () => [T, ...T[]],
  testFn: (data: NonEmptyIterable.NonEmptyIterable<T>) => void
) => {
  for (let i = 0; i < 100; i++) {
    const data = Arr.make(...generator())
    testFn(data)
  }
}

// Test examples
describe("NonEmptyIterable operations", () => {
  it("should safely extract first element", () => {
    const data = createTestData(1, 2, 3, 4, 5)
    const [first, restIterator] = NonEmptyIterable.unprepend(data)
    const rest = Array.from(restIterator)
    
    expect(first).toBe(1)
    expect(rest).toEqual([2, 3, 4, 5])
  })
  
  it("should work with single element", () => {
    const data = createTestData("only")
    const [first, restIterator] = NonEmptyIterable.unprepend(data)
    const rest = Array.from(restIterator)
    
    expect(first).toBe("only")
    expect(rest).toEqual([])
  })
  
  it("should maintain non-empty guarantee through transformations", () => {
    testWithNonEmptyData(
      () => [Math.random(), Math.random(), Math.random()],
      (data) => {
        const [first] = NonEmptyIterable.unprepend(data)
        expect(typeof first).toBe("number")
      }
    )
  })
})
```

## Conclusion

NonEmptyIterable provides **compile-time safety**, **type-level guarantees**, and **runtime reliability** for collections that must never be empty.

Key benefits:
- **Eliminates Empty Collection Errors**: Compile-time guarantee prevents runtime crashes
- **Improves Code Reliability**: Functions can safely assume at least one element exists
- **Enables Safe Transformations**: Extract first elements without defensive programming
- **Integrates with Effect Ecosystem**: Works seamlessly with Random, Stream, and other modules
- **Zero Runtime Cost**: Pure TypeScript interface with no performance overhead

Use NonEmptyIterable when your functions require guaranteed non-empty input, need to safely extract first elements, or want to eliminate defensive empty-checking throughout your codebase.