# TRef: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem TRef Solves

Managing shared mutable state in concurrent applications is notoriously difficult. Traditional approaches often lead to race conditions, deadlocks, and inconsistent state:

```typescript
// Traditional approach - race conditions and inconsistent state
class BankAccount {
  private balance = 0

  transfer(to: BankAccount, amount: number) {
    if (this.balance >= amount) {
      // Race condition: balance could change here!
      this.balance -= amount
      to.balance += amount // What if this fails?
    }
  }

  getBalance(): number {
    return this.balance // Could read inconsistent state
  }
}

// Multiple concurrent transfers can cause:
// - Lost updates
// - Inconsistent state between accounts
// - No atomicity guarantees
```

This approach leads to:
- **Race Conditions** - Multiple fibers modifying state simultaneously
- **Lost Updates** - Changes being overwritten without detection
- **Inconsistent Reads** - Reading state mid-transaction
- **Complex Locking** - Manual synchronization that's error-prone
- **Deadlock Potential** - Incorrect lock ordering causing system freezes

### The TRef Solution

TRef (Transactional Reference) provides lock-free, composable, atomic state management through Software Transactional Memory (STM):

```typescript
import { TRef, STM, Effect } from "effect"

// Create a transactional reference
const makeBankAccount = (initialBalance: number) =>
  TRef.make(initialBalance)

// Atomic, composable transfer operation
const transfer = (from: TRef<number>, to: TRef<number>, amount: number) =>
  STM.gen(function* () {
    const fromBalance = yield* TRef.get(from)
    if (fromBalance >= amount) {
      yield* TRef.update(from, balance => balance - amount)
      yield* TRef.update(to, balance => balance + amount)
      return { success: true, fromBalance: fromBalance - amount }
    }
    return { success: false, fromBalance }
  })

// All operations are atomic - either all succeed or all fail
```

### Key Concepts

**TRef<A>**: A transactional reference holding a value of type `A` that can be safely modified within STM transactions.

**STM Transaction**: A composable, atomic operation that either commits all changes or retries/fails with no side effects.

**Atomicity**: Multiple TRef operations within a single STM transaction are guaranteed to execute atomically - no partial updates.

**Composability**: STM transactions can be combined to create larger atomic operations without additional synchronization.

**Optimistic Concurrency**: TRef uses optimistic locking - transactions proceed assuming no conflicts and retry if conflicts are detected.

## Basic Usage Patterns

### Pattern 1: Creating and Reading TRefs

```typescript
import { TRef, STM, Effect } from "effect"

// Create a TRef within an STM transaction
const program = Effect.gen(function* () {
  const counter = yield* STM.commit(TRef.make(0))
  const value = yield* STM.commit(TRef.get(counter))
  console.log(`Counter value: ${value}`) // Counter value: 0
})
```

### Pattern 2: Basic Updates

```typescript
import { TRef, STM, Effect } from "effect"

const updateCounter = Effect.gen(function* () {
  const counter = yield* STM.commit(TRef.make(10))
  
  // Set to a specific value
  yield* STM.commit(TRef.set(counter, 25))
  
  // Update using a function
  yield* STM.commit(TRef.update(counter, n => n * 2))
  
  // Get and update atomically
  const oldValue = yield* STM.commit(TRef.getAndUpdate(counter, n => n + 5))
  
  const finalValue = yield* STM.commit(TRef.get(counter))
  return { oldValue, finalValue } // { oldValue: 50, finalValue: 55 }
})
```

### Pattern 3: Conditional Updates

```typescript
import { TRef, STM, Effect, Option } from "effect"

const conditionalUpdate = Effect.gen(function* () {
  const score = yield* STM.commit(TRef.make(85))
  
  // Update only if condition is met
  yield* STM.commit(
    TRef.updateSome(score, current => 
      current >= 90 ? Option.some(current + 10) : Option.none()
    )
  )
  
  // Modify with conditional logic
  const result = yield* STM.commit(
    TRef.modify(score, current => [
      current >= 90 ? "A" : "B", // return value
      current + 5 // new state
    ])
  )
  
  return result // "B" (score was 85, less than 90)
})
```

## Real-World Examples

### Example 1: Thread-Safe Cache with Expiration

```typescript
import { TRef, STM, Effect, Option, Duration, Schedule } from "effect"

interface CacheEntry<T> {
  readonly value: T
  readonly expiresAt: number
}

interface Cache<K, V> {
  readonly store: TRef<Map<K, CacheEntry<V>>>
  readonly maxSize: number
}

const makeCache = <K, V>(maxSize: number): Effect.Effect<Cache<K, V>> =>
  Effect.gen(function* () {
    const store = yield* STM.commit(TRef.make(new Map<K, CacheEntry<V>>()))
    return { store, maxSize }
  })

const set = <K, V>(
  cache: Cache<K, V>,
  key: K,
  value: V,
  ttlMs: number
): STM.STM<void> =>
  STM.gen(function* () {
    const now = Date.now()
    const expiresAt = now + ttlMs
    const newEntry: CacheEntry<V> = { value, expiresAt }
    
    yield* TRef.update(cache.store, map => {
      const newMap = new Map(map)
      
      // Remove expired entries and enforce size limit
      const validEntries = Array.from(newMap.entries())
        .filter(([_, entry]) => entry.expiresAt > now)
        .slice(-(cache.maxSize - 1)) // Leave room for new entry
      
      const finalMap = new Map(validEntries)
      finalMap.set(key, newEntry)
      return finalMap
    })
  })

const get = <K, V>(cache: Cache<K, V>, key: K): STM.STM<Option.Option<V>> =>
  STM.gen(function* () {
    const map = yield* TRef.get(cache.store)
    const entry = map.get(key)
    
    if (!entry) return Option.none()
    
    const now = Date.now()
    if (entry.expiresAt <= now) {
      // Remove expired entry
      yield* TRef.update(cache.store, m => {
        const newMap = new Map(m)
        newMap.delete(key)
        return newMap
      })
      return Option.none()
    }
    
    return Option.some(entry.value)
  })

// Usage example
const cacheExample = Effect.gen(function* () {
  const cache = yield* makeCache<string, string>(100)
  
  // Set values with different TTLs
  yield* STM.commit(set(cache, "user:123", "John Doe", 5000))
  yield* STM.commit(set(cache, "session:456", "active", 1000))
  
  // Get values (all atomic)
  const user = yield* STM.commit(get(cache, "user:123"))
  const session = yield* STM.commit(get(cache, "session:456"))
  
  return { user, session }
})
```

### Example 2: Distributed Rate Limiter

```typescript
import { TRef, STM, Effect, Array as Arr, Duration } from "effect"

interface RateLimitState {
  readonly requests: ReadonlyArray<number>
  readonly windowMs: number
  readonly maxRequests: number
}

interface RateLimiter {
  readonly state: TRef<Map<string, RateLimitState>>
  readonly defaultWindowMs: number
  readonly defaultMaxRequests: number
}

const makeRateLimiter = (
  defaultWindowMs: number,
  defaultMaxRequests: number
): Effect.Effect<RateLimiter> =>
  Effect.gen(function* () {
    const state = yield* STM.commit(TRef.make(new Map<string, RateLimitState>()))
    return { state, defaultWindowMs, defaultMaxRequests }
  })

const checkLimit = (
  limiter: RateLimiter,
  clientId: string,
  customWindowMs?: number,
  customMaxRequests?: number
): STM.STM<{ allowed: boolean; remainingRequests: number; resetTime: number }> =>
  STM.gen(function* () {
    const now = Date.now()
    const windowMs = customWindowMs ?? limiter.defaultWindowMs
    const maxRequests = customMaxRequests ?? limiter.defaultMaxRequests
    const windowStart = now - windowMs
    
    const currentState = yield* TRef.get(limiter.state)
    const clientState = currentState.get(clientId)
    
    // Filter requests within current window
    const validRequests = clientState?.requests
      .filter(timestamp => timestamp > windowStart) ?? []
    
    const allowed = validRequests.length < maxRequests
    const newRequests = allowed 
      ? [...validRequests, now]
      : validRequests
    
    // Update state
    yield* TRef.update(limiter.state, map => {
      const newMap = new Map(map)
      newMap.set(clientId, {
        requests: newRequests,
        windowMs,
        maxRequests
      })
      return newMap
    })
    
    return {
      allowed,
      remainingRequests: Math.max(0, maxRequests - newRequests.length),
      resetTime: Math.min(...validRequests, now) + windowMs
    }
  })

const cleanupExpiredEntries = (limiter: RateLimiter): STM.STM<number> =>
  STM.gen(function* () {
    const now = Date.now()
    let cleanedCount = 0
    
    yield* TRef.update(limiter.state, map => {
      const newMap = new Map<string, RateLimitState>()
      
      for (const [clientId, state] of map.entries()) {
        const windowStart = now - state.windowMs
        const validRequests = state.requests.filter(timestamp => timestamp > windowStart)
        
        if (validRequests.length > 0) {
          newMap.set(clientId, { ...state, requests: validRequests })
        } else {
          cleanedCount++
        }
      }
      
      return newMap
    })
    
    return cleanedCount
  })

// Usage in an HTTP server context
const rateLimitMiddleware = (limiter: RateLimiter) =>
  (clientId: string) =>
    Effect.gen(function* () {
      const result = yield* STM.commit(checkLimit(limiter, clientId))
      
      if (!result.allowed) {
        return Effect.fail({
          status: 429,
          message: "Rate limit exceeded",
          resetTime: result.resetTime
        })
      }
      
      return Effect.succeed({
        remainingRequests: result.remainingRequests,
        resetTime: result.resetTime
      })
    })
```

### Example 3: Multi-Account Banking System

```typescript
import { TRef, STM, Effect, Array as Arr, Option } from "effect"

interface Account {
  readonly id: string
  readonly balance: TRef<number>
  readonly transactions: TRef<ReadonlyArray<Transaction>>
}

interface Transaction {
  readonly id: string
  readonly type: "deposit" | "withdrawal" | "transfer"
  readonly amount: number
  readonly timestamp: number
  readonly fromAccount?: string
  readonly toAccount?: string
}

interface Bank {
  readonly accounts: TRef<Map<string, Account>>
  readonly transactionCounter: TRef<number>
}

const makeBank = (): Effect.Effect<Bank> =>
  Effect.gen(function* () {
    const accounts = yield* STM.commit(TRef.make(new Map<string, Account>()))
    const transactionCounter = yield* STM.commit(TRef.make(0))
    return { accounts, transactionCounter }
  })

const createAccount = (bank: Bank, accountId: string, initialBalance: number): STM.STM<Account> =>
  STM.gen(function* () {
    const accountsMap = yield* TRef.get(bank.accounts)
    
    if (accountsMap.has(accountId)) {
      return yield* STM.fail(new Error(`Account ${accountId} already exists`))
    }
    
    const balance = yield* TRef.make(initialBalance)
    const transactions = yield* TRef.make<ReadonlyArray<Transaction>>([])
    const account: Account = { id: accountId, balance, transactions }
    
    yield* TRef.update(bank.accounts, map => new Map(map).set(accountId, account))
    
    // Record initial deposit transaction
    if (initialBalance > 0) {
      const txId = yield* TRef.getAndUpdate(bank.transactionCounter, n => n + 1)
      const transaction: Transaction = {
        id: `tx-${txId}`,
        type: "deposit",
        amount: initialBalance,
        timestamp: Date.now(),
        toAccount: accountId
      }
      yield* TRef.update(transactions, txs => [...txs, transaction])
    }
    
    return account
  })

const transfer = (
  bank: Bank,
  fromAccountId: string,
  toAccountId: string,
  amount: number
): STM.STM<{ success: boolean; transactionId?: string; error?: string }> =>
  STM.gen(function* () {
    if (amount <= 0) {
      return { success: false, error: "Amount must be positive" }
    }
    
    const accountsMap = yield* TRef.get(bank.accounts)
    const fromAccount = accountsMap.get(fromAccountId)
    const toAccount = accountsMap.get(toAccountId)
    
    if (!fromAccount) {
      return { success: false, error: `Account ${fromAccountId} not found` }
    }
    
    if (!toAccount) {
      return { success: false, error: `Account ${toAccountId} not found` }
    }
    
    const fromBalance = yield* TRef.get(fromAccount.balance)
    if (fromBalance < amount) {
      return { success: false, error: "Insufficient funds" }
    }
    
    // Perform atomic transfer
    yield* TRef.update(fromAccount.balance, balance => balance - amount)
    yield* TRef.update(toAccount.balance, balance => balance + amount)
    
    // Create transaction record
    const txId = yield* TRef.getAndUpdate(bank.transactionCounter, n => n + 1)
    const transaction: Transaction = {
      id: `tx-${txId}`,
      type: "transfer",
      amount,
      timestamp: Date.now(),
      fromAccount: fromAccountId,
      toAccount: toAccountId
    }
    
    // Add transaction to both accounts
    yield* TRef.update(fromAccount.transactions, txs => [...txs, transaction])
    yield* TRef.update(toAccount.transactions, txs => [...txs, transaction])
    
    return { success: true, transactionId: transaction.id }
  })

const getAccountBalance = (bank: Bank, accountId: string): STM.STM<Option.Option<number>> =>
  STM.gen(function* () {
    const accountsMap = yield* TRef.get(bank.accounts)
    const account = accountsMap.get(accountId)
    
    if (!account) return Option.none()
    
    const balance = yield* TRef.get(account.balance)
    return Option.some(balance)
  })

const getBankSummary = (bank: Bank): STM.STM<{
  totalAccounts: number
  totalBalance: number
  totalTransactions: number
}> =>
  STM.gen(function* () {
    const accountsMap = yield* TRef.get(bank.accounts)
    const accounts = Array.from(accountsMap.values())
    
    let totalBalance = 0
    let totalTransactions = 0
    
    for (const account of accounts) {
      const balance = yield* TRef.get(account.balance)
      const transactions = yield* TRef.get(account.transactions)
      totalBalance += balance
      totalTransactions += transactions.length
    }
    
    return {
      totalAccounts: accounts.length,
      totalBalance,
      totalTransactions
    }
  })

// Usage example
const bankingExample = Effect.gen(function* () {
  const bank = yield* makeBank()
  
  // Create accounts
  const alice = yield* STM.commit(createAccount(bank, "alice", 1000))
  const bob = yield* STM.commit(createAccount(bank, "bob", 500))
  const charlie = yield* STM.commit(createAccount(bank, "charlie", 0))
  
  // Perform transfers atomically
  const transfer1 = yield* STM.commit(transfer(bank, "alice", "bob", 200))
  const transfer2 = yield* STM.commit(transfer(bank, "bob", "charlie", 150))
  
  // Get final balances
  const aliceBalance = yield* STM.commit(getAccountBalance(bank, "alice"))
  const bobBalance = yield* STM.commit(getAccountBalance(bank, "bob"))
  const charlieBalance = yield* STM.commit(getAccountBalance(bank, "charlie"))
  
  const summary = yield* STM.commit(getBankSummary(bank))
  
  return {
    transfers: [transfer1, transfer2],
    balances: { alice: aliceBalance, bob: bobBalance, charlie: charlieBalance },
    summary
  }
})
```

## Advanced Features Deep Dive

### Feature 1: Conditional Modifications with modifySome

The `modifySome` operation allows you to conditionally modify a TRef while returning a value, providing a fallback when the condition isn't met.

#### Basic modifySome Usage

```typescript
import { TRef, STM, Effect, Option } from "effect"

const conditionalIncrement = Effect.gen(function* () {
  const counter = yield* STM.commit(TRef.make(5))
  
  // Only increment if current value is even, return the operation result
  const result1 = yield* STM.commit(
    TRef.modifySome(counter, "not even", current =>
      current % 2 === 0 
        ? Option.some(["incremented", current + 1] as const)
        : Option.none()
    )
  )
  
  const value1 = yield* STM.commit(TRef.get(counter))
  console.log(result1, value1) // "not even", 5 (not modified)
  
  // Set to even number and try again
  yield* STM.commit(TRef.set(counter, 6))
  
  const result2 = yield* STM.commit(
    TRef.modifySome(counter, "not even", current =>
      current % 2 === 0 
        ? Option.some(["incremented", current + 1] as const)
        : Option.none()
    )
  )
  
  const value2 = yield* STM.commit(TRef.get(counter))
  console.log(result2, value2) // "incremented", 7 (modified)
})
```

#### Real-World modifySome Example

```typescript
interface InventoryItem {
  readonly name: string
  readonly quantity: number
  readonly reserved: number
}

const reserveItems = (
  inventory: TRef<InventoryItem>,
  requestedQuantity: number
): STM.STM<{ success: boolean; reserved: number; available: number }> =>
  TRef.modifySome(
    inventory,
    // Fallback when reservation fails
    { success: false, reserved: 0, available: 0 },
    current => {
      const available = current.quantity - current.reserved
      const canReserve = Math.min(requestedQuantity, available)
      
      if (canReserve > 0) {
        const newItem = {
          ...current,
          reserved: current.reserved + canReserve
        }
        return Option.some([
          { success: true, reserved: canReserve, available: available - canReserve },
          newItem
        ] as const)
      }
      
      return Option.none()
    }
  )
```

### Feature 2: Atomic Read-Modify Patterns

TRef provides several atomic read-modify operations that are essential for lock-free programming.

#### getAndUpdate vs updateAndGet

```typescript
import { TRef, STM, Effect } from "effect"

const atomicOperations = Effect.gen(function* () {
  const counter = yield* STM.commit(TRef.make(10))
  
  // getAndUpdate: returns OLD value, then updates
  const oldValue = yield* STM.commit(
    TRef.getAndUpdate(counter, n => n * 2)
  )
  console.log(`Old value: ${oldValue}`) // Old value: 10
  
  const currentValue = yield* STM.commit(TRef.get(counter))
  console.log(`Current value: ${currentValue}`) // Current value: 20
  
  // updateAndGet: updates first, then returns NEW value
  const newValue = yield* STM.commit(
    TRef.updateAndGet(counter, n => n + 5)
  )
  console.log(`New value: ${newValue}`) // New value: 25
})
```

#### Advanced Read-Modify Pattern: Atomic Counters

```typescript
interface CounterStats {
  readonly value: number
  readonly increments: number
  readonly decrements: number
  readonly lastModified: number
}

const makeStatsCounter = (initialValue: number): Effect.Effect<TRef<CounterStats>> =>
  STM.commit(TRef.make({
    value: initialValue,
    increments: 0,
    decrements: 0,
    lastModified: Date.now()
  }))

const incrementWithStats = (counter: TRef<CounterStats>): STM.STM<CounterStats> =>
  TRef.updateAndGet(counter, stats => ({
    value: stats.value + 1,
    increments: stats.increments + 1,
    decrements: stats.decrements,
    lastModified: Date.now()
  }))

const decrementWithStats = (counter: TRef<CounterStats>): STM.STM<CounterStats> =>
  TRef.updateAndGet(counter, stats => ({
    value: stats.value - 1,
    increments: stats.increments,
    decrements: stats.decrements + 1,
    lastModified: Date.now()
  }))

const getStatsSnapshot = (counter: TRef<CounterStats>): STM.STM<{
  current: CounterStats
  totalOperations: number
  netChange: number
}> =>
  STM.gen(function* () {
    const current = yield* TRef.get(counter)
    return {
      current,
      totalOperations: current.increments + current.decrements,
      netChange: current.increments - current.decrements
    }
  })
```

### Feature 3: Composition with Multiple TRefs

TRef operations compose naturally within STM transactions, enabling complex atomic operations across multiple references.

#### Multi-TRef Atomic Operations

```typescript
interface GameState {
  readonly player1Score: TRef<number>
  readonly player2Score: TRef<number>
  readonly gameStatus: TRef<"playing" | "finished">
  readonly winner: TRef<Option.Option<"player1" | "player2">>
}

const makeGame = (): Effect.Effect<GameState> =>
  Effect.gen(function* () {
    const player1Score = yield* STM.commit(TRef.make(0))
    const player2Score = yield* STM.commit(TRef.make(0))
    const gameStatus = yield* STM.commit(TRef.make<"playing" | "finished">("playing"))
    const winner = yield* STM.commit(TRef.make<Option.Option<"player1" | "player2">>(Option.none()))
    
    return { player1Score, player2Score, gameStatus, winner }
  })

const scorePoint = (
  game: GameState,
  player: "player1" | "player2",
  winningScore: number = 21
): STM.STM<{ newScore: number; gameFinished: boolean; winner: Option.Option<"player1" | "player2"> }> =>
  STM.gen(function* () {
    const status = yield* TRef.get(game.gameStatus)
    if (status === "finished") {
      return yield* STM.fail(new Error("Game already finished"))
    }
    
    const scoreRef = player === "player1" ? game.player1Score : game.player2Score
    const newScore = yield* TRef.updateAndGet(scoreRef, score => score + 1)
    
    if (newScore >= winningScore) {
      yield* TRef.set(game.gameStatus, "finished")
      yield* TRef.set(game.winner, Option.some(player))
      return { newScore, gameFinished: true, winner: Option.some(player) }
    }
    
    return { newScore, gameFinished: false, winner: Option.none() }
  })

const resetGame = (game: GameState): STM.STM<void> =>
  STM.gen(function* () {
    yield* TRef.set(game.player1Score, 0)
    yield* TRef.set(game.player2Score, 0)
    yield* TRef.set(game.gameStatus, "playing")
    yield* TRef.set(game.winner, Option.none())
  })

const getGameSnapshot = (game: GameState): STM.STM<{
  player1Score: number
  player2Score: number
  status: "playing" | "finished"
  winner: Option.Option<"player1" | "player2">
}> =>
  STM.gen(function* () {
    const player1Score = yield* TRef.get(game.player1Score)
    const player2Score = yield* TRef.get(game.player2Score)
    const status = yield* TRef.get(game.gameStatus)
    const winner = yield* TRef.get(game.winner)
    
    return { player1Score, player2Score, status, winner }
  })
```

## Practical Patterns & Best Practices

### Pattern 1: TRef Factory Pattern

Create reusable factories for complex TRef-based data structures:

```typescript
import { TRef, STM, Effect, Array as Arr } from "effect"

// Generic event sourcing pattern
interface EventStore<T> {
  readonly events: TRef<ReadonlyArray<T>>
  readonly version: TRef<number>
}

const makeEventStore = <T>(): Effect.Effect<EventStore<T>> =>
  Effect.gen(function* () {
    const events = yield* STM.commit(TRef.make<ReadonlyArray<T>>([]))
    const version = yield* STM.commit(TRef.make(0))
    return { events, version }
  })

const appendEvent = <T>(store: EventStore<T>, event: T): STM.STM<number> =>
  STM.gen(function* () {
    yield* TRef.update(store.events, events => [...events, event])
    return yield* TRef.updateAndGet(store.version, v => v + 1)
  })

const getEventsFrom = <T>(store: EventStore<T>, fromVersion: number): STM.STM<{
  events: ReadonlyArray<T>
  currentVersion: number
}> =>
  STM.gen(function* () {
    const events = yield* TRef.get(store.events)
    const currentVersion = yield* TRef.get(store.version)
    
    return {
      events: events.slice(fromVersion),
      currentVersion
    }
  })

// Usage for specific domain
interface UserEvent {
  readonly type: "created" | "updated" | "deleted"
  readonly userId: string
  readonly data: unknown
  readonly timestamp: number
}

const userEventStore = makeEventStore<UserEvent>()
const logUserCreated = (userId: string, userData: unknown) =>
  STM.commit(appendEvent(yield* userEventStore, {
    type: "created",
    userId,
    data: userData,
    timestamp: Date.now()
  }))
```

### Pattern 2: Resource Pool Management

```typescript
interface ResourcePool<T> {
  readonly available: TRef<ReadonlyArray<T>>
  readonly inUse: TRef<ReadonlyArray<T>>
  readonly maxSize: number
}

const makeResourcePool = <T>(resources: ReadonlyArray<T>): Effect.Effect<ResourcePool<T>> =>
  Effect.gen(function* () {
    const available = yield* STM.commit(TRef.make(resources))
    const inUse = yield* STM.commit(TRef.make<ReadonlyArray<T>>([]))
    return { available, inUse, maxSize: resources.length }
  })

const acquireResource = <T>(pool: ResourcePool<T>): STM.STM<Option.Option<T>> =>
  STM.gen(function* () {
    const availableResources = yield* TRef.get(pool.available)
    
    if (availableResources.length === 0) {
      return Option.none()
    }
    
    const [resource, ...remaining] = availableResources
    yield* TRef.set(pool.available, remaining)
    yield* TRef.update(pool.inUse, inUse => [...inUse, resource])
    
    return Option.some(resource)
  })

const releaseResource = <T>(pool: ResourcePool<T>, resource: T): STM.STM<boolean> =>
  STM.gen(function* () {
    const inUse = yield* TRef.get(pool.inUse)
    const resourceIndex = inUse.findIndex(r => r === resource)
    
    if (resourceIndex === -1) {
      return false // Resource not in use
    }
    
    const updatedInUse = inUse.filter((_, i) => i !== resourceIndex)
    yield* TRef.set(pool.inUse, updatedInUse)
    yield* TRef.update(pool.available, available => [...available, resource])
    
    return true
  })

const getPoolStats = <T>(pool: ResourcePool<T>): STM.STM<{
  available: number
  inUse: number
  total: number
  utilizationPercent: number
}> =>
  STM.gen(function* () {
    const available = yield* TRef.get(pool.available)
    const inUse = yield* TRef.get(pool.inUse)
    const availableCount = available.length
    const inUseCount = inUse.length
    const total = availableCount + inUseCount
    
    return {
      available: availableCount,
      inUse: inUseCount,
      total,
      utilizationPercent: total > 0 ? (inUseCount / total) * 100 : 0
    }
  })

// Database connection pool example
interface DatabaseConnection {
  readonly id: string
  readonly host: string
  readonly isHealthy: boolean
}

const createConnectionPool = (connections: ReadonlyArray<DatabaseConnection>) =>
  Effect.gen(function* () {
    const pool = yield* makeResourcePool(connections)
    
    const getConnection = (): Effect.Effect<DatabaseConnection, Error> =>
      Effect.gen(function* () {
        const maybeConnection = yield* STM.commit(acquireResource(pool))
        return yield* Option.match(maybeConnection, {
          onNone: () => Effect.fail(new Error("No connections available")),
          onSome: Effect.succeed
        })
      })
    
    const returnConnection = (connection: DatabaseConnection): Effect.Effect<void> =>
      Effect.gen(function* () {
        const released = yield* STM.commit(releaseResource(pool, connection))
        if (!released) {
          yield* Effect.logWarning(`Failed to release connection ${connection.id}`)
        }
      })
    
    return { pool, getConnection, returnConnection }
  })
```

### Pattern 3: State Machine Implementation

```typescript
import { TRef, STM, Effect, Option } from "effect"

// Generic state machine using TRef
interface StateMachine<S, E> {
  readonly currentState: TRef<S>
  readonly transitions: Map<S, Map<E, S>>
}

const makeStateMachine = <S, E>(
  initialState: S,
  transitions: ReadonlyArray<{ from: S; event: E; to: S }>
): Effect.Effect<StateMachine<S, E>> =>
  Effect.gen(function* () {
    const currentState = yield* STM.commit(TRef.make(initialState))
    
    // Build transition map
    const transitionMap = new Map<S, Map<E, S>>()
    for (const { from, event, to } of transitions) {
      if (!transitionMap.has(from)) {
        transitionMap.set(from, new Map())
      }
      transitionMap.get(from)!.set(event, to)
    }
    
    return { currentState, transitions: transitionMap }
  })

const sendEvent = <S, E>(
  machine: StateMachine<S, E>,
  event: E
): STM.STM<{ success: boolean; oldState: S; newState: S }> =>
  STM.gen(function* () {
    const oldState = yield* TRef.get(machine.currentState)
    const stateTransitions = machine.transitions.get(oldState)
    
    if (!stateTransitions) {
      return { success: false, oldState, newState: oldState }
    }
    
    const newState = stateTransitions.get(event)
    if (!newState) {
      return { success: false, oldState, newState: oldState }
    }
    
    yield* TRef.set(machine.currentState, newState)
    return { success: true, oldState, newState }
  })

// Order state machine example
type OrderState = "pending" | "confirmed" | "shipped" | "delivered" | "cancelled"
type OrderEvent = "confirm" | "ship" | "deliver" | "cancel"

const createOrderStateMachine = () =>
  makeStateMachine<OrderState, OrderEvent>("pending", [
    { from: "pending", event: "confirm", to: "confirmed" },
    { from: "pending", event: "cancel", to: "cancelled" },
    { from: "confirmed", event: "ship", to: "shipped" },
    { from: "confirmed", event: "cancel", to: "cancelled" },
    { from: "shipped", event: "deliver", to: "delivered" },
    { from: "shipped", event: "cancel", to: "cancelled" }
  ])

const orderWorkflow = Effect.gen(function* () {
  const orderMachine = yield* createOrderStateMachine()
  
  // Process order events
  const confirmResult = yield* STM.commit(sendEvent(orderMachine, "confirm"))
  console.log("Confirm:", confirmResult)
  
  const shipResult = yield* STM.commit(sendEvent(orderMachine, "ship"))
  console.log("Ship:", shipResult)
  
  const deliverResult = yield* STM.commit(sendEvent(orderMachine, "deliver"))
  console.log("Deliver:", deliverResult)
  
  const finalState = yield* STM.commit(TRef.get(orderMachine.currentState))
  return finalState // "delivered"
})
```

## Integration Examples

### Integration with Effect Layers and Services

```typescript
import { TRef, STM, Effect, Context, Layer } from "effect"

// Service definition using TRef for state
interface CounterService {
  readonly increment: () => Effect.Effect<number>
  readonly decrement: () => Effect.Effect<number>
  readonly get: () => Effect.Effect<number>
  readonly reset: () => Effect.Effect<void>
}

const CounterService = Context.GenericTag<CounterService>("CounterService")

// Implementation using TRef
const makeCounterService = Effect.gen(function* () {
  const counter = yield* STM.commit(TRef.make(0))
  
  const increment = () => STM.commit(TRef.updateAndGet(counter, n => n + 1))
  const decrement = () => STM.commit(TRef.updateAndGet(counter, n => n - 1))
  const get = () => STM.commit(TRef.get(counter))
  const reset = () => STM.commit(TRef.set(counter, 0))
  
  return CounterService.of({ increment, decrement, get, reset })
})

// Layer for dependency injection
const CounterServiceLive = Layer.effect(CounterService, makeCounterService)

// Usage in application
const counterApp = Effect.gen(function* () {
  const counter = yield* CounterService
  
  yield* counter.increment()
  yield* counter.increment()
  const value1 = yield* counter.get()
  
  yield* counter.decrement()
  const value2 = yield* counter.get()
  
  yield* counter.reset()
  const value3 = yield* counter.get()
  
  return { value1, value2, value3 } // { value1: 2, value2: 1, value3: 0 }
}).pipe(Effect.provide(CounterServiceLive))
```

### Integration with HTTP Server State

```typescript
import { TRef, STM, Effect, Context, Layer, HttpApi, HttpServer } from "@effect/platform"

// Application state using TRef
interface AppState {
  readonly users: TRef<Map<string, User>>
  readonly sessions: TRef<Map<string, Session>>
  readonly metrics: TRef<ServerMetrics>
}

interface User {
  readonly id: string
  readonly name: string
  readonly email: string
  readonly createdAt: number
}

interface Session {
  readonly id: string
  readonly userId: string
  readonly expiresAt: number
}

interface ServerMetrics {
  readonly requestCount: number
  readonly errorCount: number
  readonly activeUsers: number
}

const AppState = Context.GenericTag<AppState>("AppState")

const makeAppState = Effect.gen(function* () {
  const users = yield* STM.commit(TRef.make(new Map<string, User>()))
  const sessions = yield* STM.commit(TRef.make(new Map<string, Session>()))
  const metrics = yield* STM.commit(TRef.make<ServerMetrics>({
    requestCount: 0,
    errorCount: 0,
    activeUsers: 0
  }))
  
  return AppState.of({ users, sessions, metrics })
})

const AppStateLive = Layer.effect(AppState, makeAppState)

// Service methods using STM transactions
const createUser = (userData: Omit<User, "id" | "createdAt">) =>
  Effect.gen(function* () {
    const state = yield* AppState
    const userId = `user-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`
    
    const user: User = {
      id: userId,
      ...userData,
      createdAt: Date.now()
    }
    
    yield* STM.commit(
      STM.gen(function* () {
        yield* TRef.update(state.users, users => new Map(users).set(userId, user))
        yield* TRef.update(state.metrics, metrics => ({
          ...metrics,
          activeUsers: metrics.activeUsers + 1
        }))
      })
    )
    
    return user
  })

const incrementRequestMetrics = (isError: boolean = false) =>
  Effect.gen(function* () {
    const state = yield* AppState
    yield* STM.commit(
      TRef.update(state.metrics, metrics => ({
        ...metrics,
        requestCount: metrics.requestCount + 1,
        errorCount: metrics.errorCount + (isError ? 1 : 0)
      }))
    )
  })

// HTTP routes using the state
const routes = HttpApi.make("api").pipe(
  HttpApi.post("users", "/users", {
    response: User,
    request: {
      body: Schema.struct({
        name: Schema.string,
        email: Schema.string
      })
    }
  }),
  
  HttpApi.get("metrics", "/metrics", {
    response: ServerMetrics
  })
)

const handlers = routes.pipe(
  HttpApi.handle("users", ({ body }) =>
    Effect.gen(function* () {
      yield* incrementRequestMetrics()
      return yield* createUser(body)
    }).pipe(
      Effect.catchAll(error => 
        Effect.gen(function* () {
          yield* incrementRequestMetrics(true)
          return yield* Effect.fail(error)
        })
      )
    )
  ),
  
  HttpApi.handle("metrics", () =>
    Effect.gen(function* () {
      const state = yield* AppState
      return yield* STM.commit(TRef.get(state.metrics))
    })
  )
)

// Server setup
const serverLayer = HttpServer.layerWithDefaults({ port: 3000 })

const app = handlers.pipe(
  Effect.provide(AppStateLive),
  Effect.provide(serverLayer)
)
```

### Testing Strategies

```typescript
import { TRef, STM, Effect, TestClock, TestContext } from "effect"

// Test utilities for TRef-based code
const testTRefOperations = Effect.gen(function* () {
  // Test concurrent modifications
  const counter = yield* STM.commit(TRef.make(0))
  
  // Simulate concurrent increments
  const increments = Array.from({ length: 100 }, () =>
    STM.commit(TRef.update(counter, n => n + 1))
  )
  
  yield* Effect.all(increments, { concurrency: 10 })
  
  const finalValue = yield* STM.commit(TRef.get(counter))
  
  // With STM, all 100 increments should be applied atomically
  expect(finalValue).toBe(100)
})

// Property-based testing with TRef
const testTRefProperties = Effect.gen(function* () {
  const ref = yield* STM.commit(TRef.make(0))
  
  // Property: get after set should return the set value
  const testValue = 42
  yield* STM.commit(TRef.set(ref, testValue))
  const getValue = yield* STM.commit(TRef.get(ref))
  expect(getValue).toBe(testValue)
  
  // Property: update should apply function correctly
  const updateFn = (n: number) => n * 2
  const beforeUpdate = yield* STM.commit(TRef.get(ref))
  yield* STM.commit(TRef.update(ref, updateFn))
  const afterUpdate = yield* STM.commit(TRef.get(ref))
  expect(afterUpdate).toBe(updateFn(beforeUpdate))
  
  // Property: getAndUpdate returns old value
  const oldValueFromGetAndUpdate = yield* STM.commit(
    TRef.getAndUpdate(ref, n => n + 10)
  )
  expect(oldValueFromGetAndUpdate).toBe(afterUpdate)
})

// Mock TRef for testing
const makeMockTRef = <A>(initialValue: A) => {
  let value = initialValue
  const history: Array<A> = [initialValue]
  
  return {
    get: Effect.succeed(value),
    set: (newValue: A) => Effect.sync(() => {
      value = newValue
      history.push(newValue)
    }),
    update: (f: (a: A) => A) => Effect.sync(() => {
      value = f(value)
      history.push(value)
    }),
    getHistory: () => Effect.succeed([...history])
  }
}

// Test with mocked time-dependent operations
const testWithMockedTime = Effect.gen(function* () {
  const timeBasedRef = yield* STM.commit(TRef.make({ value: 0, lastUpdated: 0 }))
  
  const updateWithTimestamp = STM.gen(function* () {
    const now = Date.now()
    yield* TRef.update(timeBasedRef, state => ({
      value: state.value + 1,
      lastUpdated: now
    }))
  })
  
  // Test with controlled time
  yield* TestClock.adjust(Duration.seconds(1))
  yield* STM.commit(updateWithTimestamp)
  
  const state = yield* STM.commit(TRef.get(timeBasedRef))
  expect(state.value).toBe(1)
  expect(state.lastUpdated).toBeGreaterThan(0)
}).pipe(Effect.provide(TestContext.TestContext))
```

## Conclusion

TRef provides lock-free, composable, atomic state management for Effect applications through Software Transactional Memory (STM).

Key benefits:
- **Composable Atomicity**: Multiple TRef operations in a single STM transaction are guaranteed atomic
- **Lock-Free Concurrency**: No manual locking required - STM handles synchronization automatically  
- **Retry Semantics**: Failed transactions automatically retry without side effects
- **Type Safety**: Full TypeScript support with compile-time guarantees
- **Integration**: Seamless integration with Effect's ecosystem of modules and patterns

TRef is ideal for applications requiring coordinated state changes across multiple references, high-concurrency scenarios, and complex state management where traditional locking would be error-prone or lead to deadlocks.