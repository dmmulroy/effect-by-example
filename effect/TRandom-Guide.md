# TRandom: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem TRandom Solves

In concurrent applications, using standard random number generators leads to several critical issues. Traditional approaches either suffer from race conditions or require complex synchronization:

```typescript
// Traditional approach - NOT thread-safe
class UnsafeRandom {
  private seed: number = Date.now()
  
  next(): number {
    // Multiple threads can corrupt the seed
    this.seed = (this.seed * 1103515245 + 12345) & 0x7fffffff
    return this.seed / 0x7fffffff
  }
}

// Traditional approach with locks - performance bottleneck
class LockedRandom {
  private seed: number = Date.now()
  private mutex = new Mutex()
  
  async next(): Promise<number> {
    await this.mutex.lock()
    try {
      this.seed = (this.seed * 1103515245 + 12345) & 0x7fffffff
      return this.seed / 0x7fffffff
    } finally {
      this.mutex.unlock()
    }
  }
}
```

This approach leads to:
- **Race conditions** - Concurrent access corrupts the internal state
- **Performance bottlenecks** - Lock contention slows down the system
- **No composability** - Can't combine random operations atomically
- **Testing difficulties** - Hard to create deterministic tests

### The TRandom Solution

TRandom provides a transactional random number generator that works within Software Transactional Memory (STM), offering thread-safe, composable random operations:

```typescript
import { TRandom, STM, Effect } from "effect"

// Thread-safe, composable random operations
const program = Effect.gen(function* () {
  const random = yield* TRandom.Tag
  
  // Generate random values in STM transactions
  const randomValue = yield* STM.commit(random.next)
  const randomBool = yield* STM.commit(random.nextBoolean)
  const randomInt = yield* STM.commit(random.nextIntBetween(1, 100))
  
  return { randomValue, randomBool, randomInt }
}).pipe(
  Effect.provideService(TRandom.Tag, TRandom.live)
)
```

### Key Concepts

**Transactional Random**: All random operations occur within STM transactions, ensuring consistency

**Deterministic Testing**: Can provide custom implementations for predictable test scenarios

**Composable Operations**: Random operations can be combined with other STM operations atomically

## Basic Usage Patterns

### Setting Up TRandom

```typescript
import { TRandom, STM, Effect, Layer } from "effect"

// Using the live random implementation
const withLiveRandom = Effect.gen(function* () {
  const random = yield* TRandom.Tag
  const value = yield* STM.commit(random.next)
  return value
}).pipe(
  Effect.provideService(TRandom.Tag, TRandom.live)
)

// Creating a layer for application-wide usage
const RandomLive = Layer.succeed(TRandom.Tag, TRandom.live)

const program = Effect.gen(function* () {
  const random = yield* TRandom.Tag
  // Use random throughout your application
  return yield* STM.commit(random.next)
}).pipe(
  Effect.provide(RandomLive)
)
```

### Basic Random Operations

```typescript
import { TRandom, STM, Effect } from "effect"

const basicOperations = Effect.gen(function* () {
  const random = yield* TRandom.Tag
  
  // Generate different types of random values
  const operations = yield* STM.commit(
    STM.gen(function* () {
      // Random float between 0 and 1
      const float = yield* random.next
      
      // Random boolean
      const bool = yield* random.nextBoolean
      
      // Random integer (full range)
      const int = yield* random.nextInt
      
      // Random integer in range [min, max]
      const rangedInt = yield* random.nextIntBetween(10, 20)
      
      // Random float in range [min, max)
      const rangedFloat = yield* random.nextRange(0.0, 10.0)
      
      return { float, bool, int, rangedInt, rangedFloat }
    })
  )
  
  console.log("Random values:", operations)
}).pipe(
  Effect.provideService(TRandom.Tag, TRandom.live)
)
```

### Shuffling Collections

```typescript
import { TRandom, STM, Effect, Array as Arr } from "effect"

const shuffleExample = Effect.gen(function* () {
  const random = yield* TRandom.Tag
  
  // Original array
  const items = ["A", "B", "C", "D", "E", "F", "G", "H"]
  
  // Shuffle the array
  const shuffled = yield* STM.commit(random.shuffle(items))
  
  console.log("Original:", items)
  console.log("Shuffled:", shuffled)
  
  // Shuffle multiple times to show different results
  const multipleShuffles = yield* STM.commit(
    STM.forEach(
      Arr.range(1, 5),
      () => random.shuffle(items)
    )
  )
  
  console.log("Multiple shuffles:", multipleShuffles)
}).pipe(
  Effect.provideService(TRandom.Tag, TRandom.live)
)
```

## Real-World Examples

### Example 1: Distributed Load Balancer

A load balancer that randomly distributes requests across servers with weighted selection:

```typescript
import { TRandom, TRef, STM, Effect, Array as Arr, pipe } from "effect"

interface Server {
  id: string
  weight: number
  activeConnections: number
  maxConnections: number
}

const createLoadBalancer = Effect.gen(function* () {
  const random = yield* TRandom.Tag
  
  // Server pool with weights
  const servers = yield* STM.commit(
    TRef.make<Array<Server>>([
      { id: "server-1", weight: 3, activeConnections: 0, maxConnections: 100 },
      { id: "server-2", weight: 2, activeConnections: 0, maxConnections: 100 },
      { id: "server-3", weight: 1, activeConnections: 0, maxConnections: 50 }
    ])
  )
  
  // Weighted random selection
  const selectServer = STM.gen(function* () {
    const serverList = yield* TRef.get(servers)
    const availableServers = serverList.filter(
      s => s.activeConnections < s.maxConnections
    )
    
    if (availableServers.length === 0) {
      return STM.fail("No available servers")
    }
    
    // Calculate total weight
    const totalWeight = availableServers.reduce((sum, s) => sum + s.weight, 0)
    
    // Random selection based on weight
    const randomWeight = yield* random.nextRange(0, totalWeight)
    
    let accumulatedWeight = 0
    for (const server of availableServers) {
      accumulatedWeight += server.weight
      if (randomWeight < accumulatedWeight) {
        // Update connection count
        yield* TRef.update(servers, list =>
          list.map(s =>
            s.id === server.id
              ? { ...s, activeConnections: s.activeConnections + 1 }
              : s
          )
        )
        return server
      }
    }
    
    // Fallback (should not reach here)
    return availableServers[0]
  })
  
  // Release connection
  const releaseConnection = (serverId: string) =>
    STM.gen(function* () {
      yield* TRef.update(servers, list =>
        list.map(s =>
          s.id === serverId
            ? { ...s, activeConnections: Math.max(0, s.activeConnections - 1) }
            : s
        )
      )
    })
  
  // Simulate request handling
  const handleRequest = (requestId: string) =>
    Effect.gen(function* () {
      const server = yield* STM.commit(selectServer)
      yield* Effect.log(`Request ${requestId} -> ${server.id}`)
      
      // Simulate processing
      yield* Effect.sleep(`${Math.random() * 1000} millis`)
      
      // Release connection
      yield* STM.commit(releaseConnection(server.id))
      yield* Effect.log(`Request ${requestId} completed on ${server.id}`)
    })
  
  // Generate load
  yield* Effect.all(
    Arr.range(1, 100).pipe(
      Arr.map(i => handleRequest(`req-${i}`))
    ),
    { concurrency: 20 }
  )
  
  // Show final stats
  const finalStats = yield* STM.commit(TRef.get(servers))
  console.log("Final server stats:", finalStats)
}).pipe(
  Effect.provideService(TRandom.Tag, TRandom.live)
)
```

### Example 2: Game Mechanics System

A game system with various random mechanics like loot drops, critical hits, and procedural generation:

```typescript
import { TRandom, TRef, STM, Effect, Option, Array as Arr } from "effect"

interface LootTable {
  common: Array<string>
  uncommon: Array<string>
  rare: Array<string>
  legendary: Array<string>
}

interface Player {
  id: string
  level: number
  critChance: number
  luckModifier: number
}

const gameSystem = Effect.gen(function* () {
  const random = yield* TRandom.Tag
  
  const lootTable: LootTable = {
    common: ["Potion", "Bread", "Arrow"],
    uncommon: ["Steel Sword", "Chain Mail", "Magic Scroll"],
    rare: ["Enchanted Bow", "Dragon Scale", "Phoenix Feather"],
    legendary: ["Excalibur", "Crown of Kings", "Time Crystal"]
  }
  
  // Determine rarity based on player luck
  const determineRarity = (player: Player) =>
    STM.gen(function* () {
      const roll = yield* random.nextRange(0, 100 + player.luckModifier)
      
      if (roll >= 98) return "legendary"
      if (roll >= 90) return "rare"
      if (roll >= 70) return "uncommon"
      return "common"
    })
  
  // Generate loot drop
  const generateLoot = (player: Player, quantity: number) =>
    STM.gen(function* () {
      const drops: Array<{ rarity: string; item: string }> = []
      
      for (let i = 0; i < quantity; i++) {
        const rarity = yield* determineRarity(player)
        const items = lootTable[rarity as keyof LootTable]
        const shuffled = yield* random.shuffle(items)
        
        drops.push({
          rarity,
          item: shuffled[0]
        })
      }
      
      return drops
    })
  
  // Combat system with critical hits
  const calculateDamage = (baseDamage: number, player: Player) =>
    STM.gen(function* () {
      const critRoll = yield* random.nextRange(0, 100)
      const isCritical = critRoll < player.critChance
      
      const variance = yield* random.nextRange(0.8, 1.2)
      const damage = baseDamage * variance * (isCritical ? 2.5 : 1.0)
      
      return {
        damage: Math.floor(damage),
        isCritical
      }
    })
  
  // Procedural dungeon room generation
  const generateDungeonRoom = STM.gen(function* () {
    const width = yield* random.nextIntBetween(5, 15)
    const height = yield* random.nextIntBetween(5, 15)
    const hasMonster = yield* random.nextBoolean
    const hasTreasure = yield* random.nextBoolean
    
    const monsterCount = hasMonster
      ? yield* random.nextIntBetween(1, 5)
      : 0
    
    const treasureType = hasTreasure
      ? yield* determineRarity({ luckModifier: 0 } as Player)
      : null
    
    return {
      dimensions: { width, height },
      hasMonster,
      monsterCount,
      hasTreasure,
      treasureType
    }
  })
  
  // Example gameplay
  const player: Player = {
    id: "player-1",
    level: 10,
    critChance: 15,
    luckModifier: 5
  }
  
  // Generate some game events
  yield* Effect.log("=== Game Events ===")
  
  // Combat
  const attacks = yield* STM.commit(
    STM.forEach(
      Arr.range(1, 5),
      () => calculateDamage(50, player)
    )
  )
  
  attacks.forEach((attack, i) => {
    console.log(`Attack ${i + 1}: ${attack.damage} damage${attack.isCritical ? " (CRITICAL!)" : ""}`)
  })
  
  // Loot drops
  const loot = yield* STM.commit(generateLoot(player, 10))
  console.log("\nLoot drops:")
  loot.forEach(drop => {
    console.log(`- [${drop.rarity.toUpperCase()}] ${drop.item}`)
  })
  
  // Dungeon generation
  const rooms = yield* STM.commit(
    STM.forEach(
      Arr.range(1, 5),
      generateDungeonRoom
    )
  )
  
  console.log("\nDungeon rooms:")
  rooms.forEach((room, i) => {
    console.log(`Room ${i + 1}: ${room.dimensions.width}x${room.dimensions.height}`)
    if (room.hasMonster) console.log(`  - ${room.monsterCount} monsters`)
    if (room.hasTreasure) console.log(`  - ${room.treasureType} treasure`)
  })
}).pipe(
  Effect.provideService(TRandom.Tag, TRandom.live)
)
```

### Example 3: A/B Testing Framework

An A/B testing system with random variant assignment and statistical tracking:

```typescript
import { TRandom, TRef, TMap, STM, Effect, Option, HashMap } from "effect"

interface Experiment {
  id: string
  variants: Array<{
    id: string
    weight: number
    configuration: unknown
  }>
  active: boolean
}

interface Assignment {
  userId: string
  experimentId: string
  variantId: string
  timestamp: number
}

const abTestingFramework = Effect.gen(function* () {
  const random = yield* TRandom.Tag
  
  // Storage for experiments and assignments
  const experiments = yield* STM.commit(TMap.empty<string, Experiment>())
  const assignments = yield* STM.commit(TMap.empty<string, Assignment>())
  
  // Create an experiment
  const createExperiment = (experiment: Experiment) =>
    STM.gen(function* () {
      yield* TMap.set(experiments, experiment.id, experiment)
    })
  
  // Get variant assignment for user
  const getVariantAssignment = (userId: string, experimentId: string) =>
    STM.gen(function* () {
      // Check existing assignment
      const assignmentKey = `${userId}:${experimentId}`
      const existing = yield* TMap.get(assignments, assignmentKey)
      
      if (Option.isSome(existing)) {
        return existing.value.variantId
      }
      
      // Get experiment
      const experiment = yield* TMap.get(experiments, experimentId)
      
      if (Option.isNone(experiment) || !experiment.value.active) {
        return null
      }
      
      // Calculate weighted random assignment
      const exp = experiment.value
      const totalWeight = exp.variants.reduce((sum, v) => sum + v.weight, 0)
      const roll = yield* random.nextRange(0, totalWeight)
      
      let accumulatedWeight = 0
      let selectedVariant = exp.variants[0]
      
      for (const variant of exp.variants) {
        accumulatedWeight += variant.weight
        if (roll < accumulatedWeight) {
          selectedVariant = variant
          break
        }
      }
      
      // Store assignment
      const assignment: Assignment = {
        userId,
        experimentId,
        variantId: selectedVariant.id,
        timestamp: Date.now()
      }
      
      yield* TMap.set(assignments, assignmentKey, assignment)
      
      return selectedVariant.id
    })
  
  // Get experiment statistics
  const getExperimentStats = (experimentId: string) =>
    STM.gen(function* () {
      const allAssignments = yield* TMap.toArray(assignments)
      const experimentAssignments = allAssignments
        .map(([_, assignment]) => assignment)
        .filter(a => a.experimentId === experimentId)
      
      const stats = experimentAssignments.reduce((acc, assignment) => {
        acc[assignment.variantId] = (acc[assignment.variantId] || 0) + 1
        return acc
      }, {} as Record<string, number>)
      
      return {
        total: experimentAssignments.length,
        variants: stats
      }
    })
  
  // Set up experiments
  yield* STM.commit(
    STM.all([
      createExperiment({
        id: "homepage-cta",
        active: true,
        variants: [
          { id: "control", weight: 50, configuration: { text: "Sign Up Now" } },
          { id: "variant-a", weight: 25, configuration: { text: "Get Started Free" } },
          { id: "variant-b", weight: 25, configuration: { text: "Try It Today" } }
        ]
      }),
      createExperiment({
        id: "pricing-layout",
        active: true,
        variants: [
          { id: "grid", weight: 33, configuration: { layout: "grid" } },
          { id: "table", weight: 33, configuration: { layout: "table" } },
          { id: "cards", weight: 34, configuration: { layout: "cards" } }
        ]
      })
    ])
  )
  
  // Simulate user visits
  const simulateUsers = yield* Effect.all(
    Arr.range(1, 1000).pipe(
      Arr.map(i => {
        const userId = `user-${i}`
        return STM.commit(
          STM.gen(function* () {
            const homepageVariant = yield* getVariantAssignment(userId, "homepage-cta")
            const pricingVariant = yield* getVariantAssignment(userId, "pricing-layout")
            return { userId, homepageVariant, pricingVariant }
          })
        )
      })
    ),
    { concurrency: 50 }
  )
  
  // Get statistics
  const stats = yield* STM.commit(
    STM.gen(function* () {
      const homepageStats = yield* getExperimentStats("homepage-cta")
      const pricingStats = yield* getExperimentStats("pricing-layout")
      return { homepageStats, pricingStats }
    })
  )
  
  console.log("Experiment Statistics:")
  console.log("Homepage CTA:", stats.homepageStats)
  console.log("Pricing Layout:", stats.pricingStats)
  
  // Verify distribution is roughly as expected
  const homepageTotal = stats.homepageStats.total
  console.log("\nHomepage distribution:")
  console.log(`- Control: ${((stats.homepageStats.variants.control / homepageTotal) * 100).toFixed(1)}% (expected: 50%)`)
  console.log(`- Variant A: ${((stats.homepageStats.variants["variant-a"] / homepageTotal) * 100).toFixed(1)}% (expected: 25%)`)
  console.log(`- Variant B: ${((stats.homepageStats.variants["variant-b"] / homepageTotal) * 100).toFixed(1)}% (expected: 25%)`)
}).pipe(
  Effect.provideService(TRandom.Tag, TRandom.live)
)
```

## Advanced Features Deep Dive

### Custom Random Implementations

Create deterministic random implementations for testing:

```typescript
import { TRandom, STM, Effect } from "effect"

// Deterministic random for testing
class DeterministicRandom implements TRandom.TRandom {
  readonly [TRandom.TRandomTypeId] = TRandom.TRandomTypeId
  
  private values: number[]
  private index = 0
  
  constructor(values: number[]) {
    this.values = values
  }
  
  readonly next = STM.gen(function* (this: DeterministicRandom) {
    const value = this.values[this.index % this.values.length]
    this.index++
    return value
  }).bind(this)
  
  readonly nextBoolean = STM.gen(function* (this: DeterministicRandom) {
    const value = yield* this.next
    return value > 0.5
  }).bind(this)
  
  readonly nextInt = STM.gen(function* (this: DeterministicRandom) {
    const value = yield* this.next
    return Math.floor(value * Number.MAX_SAFE_INTEGER)
  }).bind(this)
  
  nextRange = (min: number, max: number) =>
    STM.gen(function* (this: DeterministicRandom) {
      const value = yield* this.next
      return min + value * (max - min)
    }).bind(this)
  
  nextIntBetween = (min: number, max: number) =>
    STM.gen(function* (this: DeterministicRandom) {
      const value = yield* this.nextRange(min, max + 1)
      return Math.floor(value)
    }).bind(this)
  
  shuffle = <A>(elements: Iterable<A>) =>
    STM.succeed(Array.from(elements))
}

// Usage in tests
const testProgram = Effect.gen(function* () {
  const random = yield* TRandom.Tag
  
  const results = yield* STM.commit(
    STM.gen(function* () {
      const nums = []
      for (let i = 0; i < 5; i++) {
        nums.push(yield* random.next)
      }
      return nums
    })
  )
  
  // In tests, this will always produce [0.1, 0.2, 0.3, 0.4, 0.5]
  console.log("Random values:", results)
}).pipe(
  Effect.provideService(
    TRandom.Tag,
    new DeterministicRandom([0.1, 0.2, 0.3, 0.4, 0.5])
  )
)
```

### Seeded Random Generation

Create reproducible random sequences:

```typescript
import { TRandom, TRef, STM, Effect } from "effect"

// Seeded random implementation
class SeededRandom implements TRandom.TRandom {
  readonly [TRandom.TRandomTypeId] = TRandom.TRandomTypeId
  
  private seedRef: TRef.TRef<number>
  
  constructor(seed: number) {
    this.seedRef = STM.unsafeCommit(TRef.make(seed))
  }
  
  private nextSeed = STM.gen(function* (this: SeededRandom) {
    const seed = yield* TRef.get(this.seedRef)
    const newSeed = (seed * 1103515245 + 12345) & 0x7fffffff
    yield* TRef.set(this.seedRef, newSeed)
    return newSeed / 0x7fffffff
  }).bind(this)
  
  readonly next = this.nextSeed
  
  readonly nextBoolean = STM.gen(function* (this: SeededRandom) {
    return (yield* this.next) > 0.5
  }).bind(this)
  
  readonly nextInt = STM.gen(function* (this: SeededRandom) {
    return Math.floor((yield* this.next) * Number.MAX_SAFE_INTEGER)
  }).bind(this)
  
  nextRange = (min: number, max: number) =>
    STM.gen(function* (this: SeededRandom) {
      return min + (yield* this.next) * (max - min)
    }).bind(this)
  
  nextIntBetween = (min: number, max: number) =>
    STM.gen(function* (this: SeededRandom) {
      return Math.floor(yield* this.nextRange(min, max + 1))
    }).bind(this)
  
  shuffle = <A>(elements: Iterable<A>) =>
    STM.gen(function* (this: SeededRandom) {
      const array = Array.from(elements)
      const result = [...array]
      
      for (let i = result.length - 1; i > 0; i--) {
        const j = yield* this.nextIntBetween(0, i)
        ;[result[i], result[j]] = [result[j], result[i]]
      }
      
      return result
    }).bind(this)
}

// Reproducible random sequences
const reproducibleExample = Effect.gen(function* () {
  const seed = 12345
  
  // First run
  const firstRun = yield* Effect.gen(function* () {
    const random = yield* TRandom.Tag
    return yield* STM.commit(
      STM.forEach(
        Arr.range(1, 5),
        () => random.nextIntBetween(1, 100)
      )
    )
  }).pipe(
    Effect.provideService(TRandom.Tag, new SeededRandom(seed))
  )
  
  // Second run with same seed
  const secondRun = yield* Effect.gen(function* () {
    const random = yield* TRandom.Tag
    return yield* STM.commit(
      STM.forEach(
        Arr.range(1, 5),
        () => random.nextIntBetween(1, 100)
      )
    )
  }).pipe(
    Effect.provideService(TRandom.Tag, new SeededRandom(seed))
  )
  
  console.log("First run:", firstRun)
  console.log("Second run:", secondRun)
  console.log("Are equal:", JSON.stringify(firstRun) === JSON.stringify(secondRun))
})
```

### Combining with Other STM Operations

Use TRandom within complex STM transactions:

```typescript
import { TRandom, TRef, TQueue, STM, Effect } from "effect"

const complexTransaction = Effect.gen(function* () {
  const random = yield* TRandom.Tag
  
  // Shared state
  const inventory = yield* STM.commit(
    TRef.make<Array<{ id: string; quantity: number }>>([
      { id: "sword", quantity: 10 },
      { id: "shield", quantity: 5 },
      { id: "potion", quantity: 20 }
    ])
  )
  
  const orderQueue = yield* STM.commit(TQueue.unbounded<string>())
  
  // Complex transaction: randomly select items and update inventory
  const placeRandomOrder = STM.gen(function* () {
    const items = yield* TRef.get(inventory)
    
    // Randomly select an item
    const shuffled = yield* random.shuffle(items)
    const selectedItem = shuffled.find(item => item.quantity > 0)
    
    if (!selectedItem) {
      return STM.fail("No items in stock")
    }
    
    // Random quantity (1 to available)
    const quantity = yield* random.nextIntBetween(
      1,
      Math.min(selectedItem.quantity, 5)
    )
    
    // Update inventory atomically
    yield* TRef.update(inventory, items =>
      items.map(item =>
        item.id === selectedItem.id
          ? { ...item, quantity: item.quantity - quantity }
          : item
      )
    )
    
    // Create order
    const orderId = `order-${Date.now()}-${yield* random.nextInt}`
    const order = `${orderId}: ${quantity}x ${selectedItem.id}`
    
    // Queue order for processing
    yield* TQueue.offer(orderQueue, order)
    
    return order
  })
  
  // Place multiple orders concurrently
  const orders = yield* Effect.all(
    Arr.range(1, 10).pipe(
      Arr.map(() =>
        STM.commit(placeRandomOrder).pipe(
          Effect.either
        )
      )
    ),
    { concurrency: 5 }
  )
  
  // Check results
  const successfulOrders = orders.filter(r => r._tag === "Right")
  const finalInventory = yield* STM.commit(TRef.get(inventory))
  
  console.log(`Successful orders: ${successfulOrders.length}`)
  console.log("Final inventory:", finalInventory)
}).pipe(
  Effect.provideService(TRandom.Tag, TRandom.live)
)
```

## Practical Patterns & Best Practices

### Pattern 1: Random with Retry Logic

Implement random backoff for retry operations:

```typescript
import { TRandom, STM, Effect, Schedule, Duration } from "effect"

const makeRandomBackoffSchedule = Effect.gen(function* () {
  const random = yield* TRandom.Tag
  
  return Schedule.exponential("100 millis").pipe(
    Schedule.modifyDelay(delay =>
      STM.commit(
        STM.gen(function* () {
          // Add random jitter: ±20% of the delay
          const jitter = yield* random.nextRange(0.8, 1.2)
          return Duration.scale(delay, jitter)
        })
      )
    )
  )
})

// Usage
const retryWithRandomBackoff = Effect.gen(function* () {
  let attempts = 0
  
  const failingOperation = Effect.gen(function* () {
    attempts++
    yield* Effect.log(`Attempt ${attempts}`)
    
    if (attempts < 4) {
      return yield* Effect.fail("Operation failed")
    }
    
    return "Success!"
  })
  
  const schedule = yield* makeRandomBackoffSchedule
  
  const result = yield* failingOperation.pipe(
    Effect.retry(schedule)
  )
  
  console.log("Final result:", result)
}).pipe(
  Effect.provideService(TRandom.Tag, TRandom.live)
)
```

### Pattern 2: Probabilistic Data Structures

Build probabilistic data structures using TRandom:

```typescript
import { TRandom, TRef, STM, Effect, Array as Arr, pipe } from "effect"

// Bloom filter implementation
interface BloomFilter {
  add: (item: string) => STM.STM<void>
  mightContain: (item: string) => STM.STM<boolean>
  falsePositiveRate: () => STM.STM<number>
}

const makeBloomFilter = (size: number, hashCount: number): Effect.Effect<BloomFilter> =>
  Effect.gen(function* () {
    const random = yield* TRandom.Tag
    const bits = yield* STM.commit(TRef.make(new Array(size).fill(false)))
    const seeds = yield* STM.commit(
      STM.forEach(
        Arr.range(0, hashCount - 1),
        () => random.nextInt
      )
    )
    
    const hash = (item: string, seed: number): number => {
      let hash = seed
      for (let i = 0; i < item.length; i++) {
        hash = ((hash << 5) - hash + item.charCodeAt(i)) | 0
      }
      return Math.abs(hash) % size
    }
    
    const add = (item: string) =>
      STM.gen(function* () {
        const indices = seeds.map(seed => hash(item, seed))
        yield* TRef.update(bits, array => {
          const newArray = [...array]
          indices.forEach(i => { newArray[i] = true })
          return newArray
        })
      })
    
    const mightContain = (item: string) =>
      STM.gen(function* () {
        const indices = seeds.map(seed => hash(item, seed))
        const bitArray = yield* TRef.get(bits)
        return indices.every(i => bitArray[i])
      })
    
    const falsePositiveRate = () =>
      STM.gen(function* () {
        const bitArray = yield* TRef.get(bits)
        const setBits = bitArray.filter(b => b).length
        const k = hashCount
        const m = size
        const n = setBits / k // Estimated number of elements
        
        // Calculate theoretical false positive rate
        return Math.pow(1 - Math.exp(-k * n / m), k)
      })
    
    return { add, mightContain, falsePositiveRate }
  }).pipe(
    Effect.provideService(TRandom.Tag, TRandom.live)
  )

// Usage example
const bloomFilterExample = Effect.gen(function* () {
  const filter = yield* makeBloomFilter(1000, 3)
  
  // Add items
  const items = ["apple", "banana", "cherry", "date", "elderberry"]
  yield* STM.commit(
    STM.forEach(items, item => filter.add(item))
  )
  
  // Test membership
  const tests = [
    ...items,
    "grape", "kiwi", "mango", "orange", "peach"
  ]
  
  const results = yield* STM.commit(
    STM.forEach(tests, item =>
      STM.gen(function* () {
        const mightContain = yield* filter.mightContain(item)
        return { item, mightContain }
      })
    )
  )
  
  console.log("Bloom filter results:")
  results.forEach(({ item, mightContain }) => {
    const actual = items.includes(item)
    console.log(`${item}: ${mightContain} (actual: ${actual})`)
  })
  
  const fpr = yield* STM.commit(filter.falsePositiveRate())
  console.log(`\nEstimated false positive rate: ${(fpr * 100).toFixed(2)}%`)
})
```

## Integration Examples

### Integration with Effect Streams

Use TRandom to generate random data streams:

```typescript
import { TRandom, STM, Effect, Stream, Schedule } from "effect"

const randomDataStream = Effect.gen(function* () {
  const random = yield* TRandom.Tag
  
  // Generate random events
  interface Event {
    type: "click" | "view" | "purchase"
    value: number
    timestamp: number
  }
  
  const eventTypes = ["click", "view", "purchase"] as const
  
  const randomEventStream = Stream.repeatEffect(
    STM.commit(
      STM.gen(function* () {
        const type = yield* random.shuffle(eventTypes).pipe(
          STM.map(arr => arr[0])
        )
        const value = yield* random.nextIntBetween(1, 1000)
        
        return {
          type,
          value,
          timestamp: Date.now()
        } satisfies Event
      })
    )
  ).pipe(
    Stream.schedule(
      Schedule.exponential("10 millis").pipe(
        Schedule.compose(
          Schedule.recurs(100)
        )
      )
    )
  )
  
  // Process stream with random sampling
  const sampledStream = randomEventStream.pipe(
    Stream.filterEffect(() =>
      STM.commit(
        STM.gen(function* () {
          // Random sampling - keep 30% of events
          const keep = yield* random.nextRange(0, 1)
          return keep < 0.3
        })
      )
    )
  )
  
  // Aggregate sampled data
  const aggregated = yield* sampledStream.pipe(
    Stream.groupByKey(
      event => event.type,
      { bufferSize: 100 }
    ),
    Stream.mergeAll(3),
    Stream.take(50),
    Stream.runCollect
  )
  
  // Summary statistics
  const summary = Chunk.toArray(aggregated).reduce((acc, event) => {
    acc[event.type] = (acc[event.type] || 0) + 1
    return acc
  }, {} as Record<string, number>)
  
  console.log("Sampled event summary:", summary)
}).pipe(
  Effect.provideService(TRandom.Tag, TRandom.live)
)
```

### Testing Strategies

Test random behavior deterministically:

```typescript
import { TRandom, STM, Effect, TestClock } from "effect"
import { describe, it, expect } from "vitest"

describe("TRandom", () => {
  it("should produce deterministic results with fixed values", () =>
    Effect.gen(function* () {
      const values = [0.1, 0.5, 0.9, 0.3, 0.7]
      const random = new DeterministicRandom(values)
      
      const results = yield* STM.commit(
        STM.gen(function* () {
          const bools = []
          for (let i = 0; i < 5; i++) {
            bools.push(yield* random.nextBoolean)
          }
          return bools
        })
      )
      
      expect(results).toEqual([false, false, true, false, true])
    }).pipe(Effect.runPromise))
  
  it("should distribute values according to weights", () =>
    Effect.gen(function* () {
      // Use many iterations with deterministic random
      const iterations = 10000
      const values: number[] = []
      
      // Generate uniform distribution
      for (let i = 0; i < iterations; i++) {
        values.push(i / iterations)
      }
      
      const random = new DeterministicRandom(values)
      
      // Test weighted selection
      const weights = { A: 50, B: 30, C: 20 }
      const totalWeight = 100
      
      const results = yield* STM.commit(
        STM.forEach(
          values,
          () =>
            STM.gen(function* () {
              const roll = yield* random.nextRange(0, totalWeight)
              if (roll < 50) return "A"
              if (roll < 80) return "B"
              return "C"
            })
        )
      )
      
      const counts = results.reduce((acc, result) => {
        acc[result] = (acc[result] || 0) + 1
        return acc
      }, {} as Record<string, number>)
      
      const percentages = {
        A: (counts.A / iterations) * 100,
        B: (counts.B / iterations) * 100,
        C: (counts.C / iterations) * 100
      }
      
      // Check distribution is within 1% of expected
      expect(Math.abs(percentages.A - 50)).toBeLessThan(1)
      expect(Math.abs(percentages.B - 30)).toBeLessThan(1)
      expect(Math.abs(percentages.C - 20)).toBeLessThan(1)
    }).pipe(Effect.runPromise))
  
  it("should maintain shuffle properties", () =>
    Effect.gen(function* () {
      const random = yield* TRandom.Tag
      const original = [1, 2, 3, 4, 5]
      
      const shuffled = yield* STM.commit(random.shuffle(original))
      
      // Same elements, different order (usually)
      expect(shuffled.sort()).toEqual(original.sort())
      expect(shuffled.length).toBe(original.length)
      
      // Test that shuffle produces different results
      const multipleShuffles = yield* STM.commit(
        STM.forEach(
          Arr.range(1, 10),
          () => random.shuffle(original)
        )
      )
      
      const uniqueOrders = new Set(
        multipleShuffles.map(arr => arr.join(","))
      )
      
      // Should produce multiple different orders
      expect(uniqueOrders.size).toBeGreaterThan(1)
    }).pipe(
      Effect.provideService(TRandom.Tag, TRandom.live),
      Effect.runPromise
    ))
})
```

## Conclusion

TRandom provides a powerful, thread-safe solution for random number generation within STM transactions. Its composability with other STM operations makes it ideal for building complex concurrent systems that require randomness.

Key benefits:
- **Thread Safety**: All operations are transactional and safe for concurrent use
- **Composability**: Seamlessly integrates with other STM operations
- **Testability**: Easy to provide deterministic implementations for testing
- **Type Safety**: Full type inference and compile-time guarantees

Use TRandom when you need random number generation in concurrent contexts, game mechanics, simulations, testing frameworks, or any scenario requiring thread-safe randomness.