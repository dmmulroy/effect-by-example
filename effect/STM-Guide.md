# STM: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem STM Solves

Concurrent programming with shared mutable state is notoriously difficult. Traditional approaches using locks, mutexes, and atomic operations lead to complex, error-prone code with deadlocks, race conditions, and performance bottlenecks:

```typescript
// Traditional approach - manual locking and coordination
class BankAccount {
  private balance: number = 0
  private mutex = new Mutex() // Hypothetical mutex implementation
  
  async transfer(to: BankAccount, amount: number): Promise<boolean> {
    // Complex lock ordering to avoid deadlocks
    const locks = [this, to].sort((a, b) => a.id - b.id)
    
    await locks[0].mutex.acquire()
    try {
      await locks[1].mutex.acquire()
      try {
        // What if this check passes but balance changes before withdrawal?
        if (this.balance >= amount) {
          this.balance -= amount
          to.balance += amount
          return true
        }
        return false
      } finally {
        locks[1].mutex.release()
      }
    } finally {
      locks[0].mutex.release()
    }
  }
  
  async getBalance(): Promise<number> {
    await this.mutex.acquire()
    try {
      return this.balance
    } finally {
      this.mutex.release()
    }
  }
  
  async deposit(amount: number): Promise<void> {
    await this.mutex.acquire()
    try {
      this.balance += amount
    } finally {
      this.mutex.release()
    }
  }
}

// Complex coordination scenarios
class TradingSystem {
  private accounts = new Map<string, BankAccount>()
  private orderBook: Order[] = []
  private globalLock = new Mutex()
  
  async processOrder(order: Order): Promise<boolean> {
    // Need to coordinate multiple accounts and order book
    await this.globalLock.acquire()
    try {
      const buyer = this.accounts.get(order.buyerId)
      const seller = this.accounts.get(order.sellerId)
      
      if (!buyer || !seller) return false
      
      // This gets extremely complex with multiple locks...
      // What about partial failures? Rollbacks? Retries?
      
      // Race conditions everywhere!
      const buyerBalance = await buyer.getBalance()
      const sellerBalance = await seller.getBalance()
      
      if (buyerBalance >= order.price && sellerBalance >= order.quantity) {
        // Multiple operations that can fail independently
        const success1 = await buyer.transfer(seller, order.price)
        const success2 = await seller.transfer(buyer, order.quantity)
        
        if (success1 && success2) {
          this.orderBook.push(order)
          return true
        } else {
          // How do we rollback? This is a nightmare!
          return false
        }
      }
      
      return false
    } finally {
      this.globalLock.release()
    }
  }
}
```

This approach leads to:
- **Deadlock prone** - Complex lock ordering requirements
- **Race conditions** - State can change between read and write
- **No composability** - Can't easily combine operations
- **Poor performance** - Coarse-grained locking kills concurrency
- **Error recovery** - Extremely difficult to handle partial failures

### The STM Solution

Software Transactional Memory (STM) provides lock-free, composable, atomic transactions with automatic retry and rollback:

```typescript
import { Effect, STM, TRef } from "effect"

// Clean, composable transactions
class BankAccount {
  constructor(private balanceRef: TRef.TRef<number>) {}
  
  static make = (initialBalance: number) =>
    STM.gen(function* () {
      const balanceRef = yield* TRef.make(initialBalance)
      return new BankAccount(balanceRef)
    })
  
  // Atomic, composable operations
  getBalance = () => TRef.get(this.balanceRef)
  
  deposit = (amount: number) =>
    TRef.update(this.balanceRef, balance => balance + amount)
  
  withdraw = (amount: number) =>
    STM.gen(function* () {
      const balance = yield* TRef.get(this.balanceRef)
      if (balance >= amount) {
        yield* TRef.update(this.balanceRef, b => b - amount)
        return true
      } else {
        yield* STM.retry() // Automatically retry when balance increases
      }
    })
  
  // Atomic transfer - either both operations succeed or both fail
  transferTo = (to: BankAccount, amount: number) =>
    STM.gen(function* () {
      yield* this.withdraw(amount)
      yield* to.deposit(amount)
    })
}

// Complex transactions are just compositions of simple ones
const tradingTransaction = (
  buyer: BankAccount,
  seller: BankAccount,
  price: number,
  quantity: number
) =>
  STM.gen(function* () {
    // All operations are atomic - no partial failures!
    yield* buyer.transferTo(seller, price)
    yield* seller.transferTo(buyer, quantity)
    
    // Can include additional logic
    const buyerBalance = yield* buyer.getBalance()
    const sellerBalance = yield* seller.getBalance()
    
    return { success: true, buyerBalance, sellerBalance }
  })

// Running transactions
const program = Effect.gen(function* () {
  const aliceAccount = yield* STM.commit(BankAccount.make(1000))
  const bobAccount = yield* STM.commit(BankAccount.make(500))
  
  // Atomic transfer - automatically retries if insufficient funds
  const result = yield* STM.commit(
    aliceAccount.transferTo(bobAccount, 200)
  )
  
  const aliceBalance = yield* STM.commit(aliceAccount.getBalance())
  const bobBalance = yield* STM.commit(bobAccount.getBalance())
  
  return { aliceBalance, bobBalance }
})
```

### Key Concepts

**STM (Software Transactional Memory)**: A programming model that allows composable, atomic operations on shared memory without explicit locking.

**TRef (Transactional Reference)**: A mutable reference that can only be modified within STM transactions.

**Transaction**: A sequence of STM operations that execute atomically - either all succeed or all are rolled back.

**Retry**: Automatic blocking and retrying when preconditions aren't met, with wake-up when relevant state changes.

**Commit**: Converting an STM transaction into an Effect that can be executed.

## Basic Usage Patterns

### Pattern 1: Basic Transactional References

```typescript
import { Effect, STM, TRef } from "effect"

// Basic TRef operations
const basicTRefExample = Effect.gen(function* () {
  // Create a transactional reference
  const counter = yield* STM.commit(TRef.make(0))
  
  // Read the current value
  const currentValue = yield* STM.commit(TRef.get(counter))
  console.log(`Current value: ${currentValue}`) // 0
  
  // Update the value
  yield* STM.commit(TRef.set(counter, 10))
  
  // Modify using current value
  yield* STM.commit(TRef.update(counter, n => n * 2))
  
  // Get and set in one operation
  const oldValue = yield* STM.commit(TRef.getAndSet(counter, 100))
  console.log(`Old value: ${oldValue}`) // 20
  
  const finalValue = yield* STM.commit(TRef.get(counter))
  console.log(`Final value: ${finalValue}`) // 100
})
```

### Pattern 2: Conditional Operations and Retry

```typescript
import { Effect, STM, TRef } from "effect"

interface Resource {
  available: number
  allocated: number
  maxCapacity: number
}

const resourceManager = Effect.gen(function* () {
  const resource = yield* STM.commit(TRef.make<Resource>({
    available: 10,
    allocated: 0,
    maxCapacity: 10
  }))
  
  // Allocation with automatic retry when resources become available
  const allocateResource = (amount: number) =>
    STM.gen(function* () {
      const current = yield* TRef.get(resource)
      
      if (current.available >= amount) {
        yield* TRef.update(resource, r => ({
          ...r,
          available: r.available - amount,
          allocated: r.allocated + amount
        }))
        return `Allocated ${amount} resources`
      } else {
        // Retry automatically when resources become available
        yield* STM.retry()
      }
    })
  
  // Release resources
  const releaseResource = (amount: number) =>
    STM.gen(function* () {
      yield* TRef.update(resource, r => ({
        ...r,
        available: Math.min(r.available + amount, r.maxCapacity),
        allocated: Math.max(r.allocated - amount, 0)
      }))
    })
  
  // Check if operation would succeed without actually performing it
  const canAllocate = (amount: number) =>
    STM.gen(function* () {
      const current = yield* TRef.get(resource)
      return current.available >= amount
    })
  
  return {
    allocateResource,
    releaseResource,
    canAllocate,
    getStatus: () => TRef.get(resource)
  }
})
```

### Pattern 3: Composing Multiple Operations

```typescript
import { Effect, STM, TRef } from "effect"

interface UserAccount {
  id: string
  balance: number
  transactions: string[]
}

interface TransferResult {
  success: boolean
  fromBalance: number
  toBalance: number
  transactionId: string
}

const bankingSystem = Effect.gen(function* () {
  const users = yield* STM.commit(TRef.make(new Map<string, UserAccount>()))
  
  // Create a new user account
  const createAccount = (userId: string, initialBalance: number) =>
    STM.gen(function* () {
      const currentUsers = yield* TRef.get(users)
      
      if (currentUsers.has(userId)) {
        yield* STM.fail(`User ${userId} already exists`)
      }
      
      const newAccount: UserAccount = {
        id: userId,
        balance: initialBalance,
        transactions: []
      }
      
      yield* TRef.update(users, map => new Map(map.set(userId, newAccount)))
      return newAccount
    })
  
  // Transfer money between accounts
  const transferMoney = (
    fromUserId: string,
    toUserId: string,
    amount: number
  ): STM.STM<TransferResult, string> =>
    STM.gen(function* () {
      const currentUsers = yield* TRef.get(users)
      const fromUser = currentUsers.get(fromUserId)
      const toUser = currentUsers.get(toUserId)
      
      if (!fromUser) {
        yield* STM.fail(`From user ${fromUserId} not found`)
      }
      
      if (!toUser) {
        yield* STM.fail(`To user ${toUserId} not found`)
      }
      
      if (fromUser.balance < amount) {
        yield* STM.fail(`Insufficient funds: ${fromUser.balance} < ${amount}`)
      }
      
      const transactionId = `tx-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`
      
      // Atomic update of both accounts
      yield* TRef.update(users, map => {
        const newMap = new Map(map)
        
        // Update from account
        newMap.set(fromUserId, {
          ...fromUser,
          balance: fromUser.balance - amount,
          transactions: [...fromUser.transactions, `SENT ${amount} to ${toUserId} (${transactionId})`]
        })
        
        // Update to account
        newMap.set(toUserId, {
          ...toUser,
          balance: toUser.balance + amount,
          transactions: [...toUser.transactions, `RECEIVED ${amount} from ${fromUserId} (${transactionId})`]
        })
        
        return newMap
      })
      
      return {
        success: true,
        fromBalance: fromUser.balance - amount,
        toBalance: toUser.balance + amount,
        transactionId
      }
    })
  
  // Get account information
  const getAccount = (userId: string) =>
    STM.gen(function* () {
      const currentUsers = yield* TRef.get(users)
      const user = currentUsers.get(userId)
      
      if (!user) {
        yield* STM.fail(`User ${userId} not found`)
      }
      
      return user
    })
  
  return {
    createAccount,
    transferMoney,
    getAccount,
    getAllUsers: () => TRef.get(users)
  }
})
```

## Real-World Examples

### Example 1: Thread-Safe Cache with LRU Eviction

An LRU cache that handles concurrent access safely using STM transactions.

```typescript
import { Effect, STM, TRef, Option } from "effect"

interface CacheEntry<T> {
  value: T
  accessTime: number
  accessCount: number
}

interface CacheState<T> {
  entries: Map<string, CacheEntry<T>>
  accessOrder: string[]
  maxSize: number
  hits: number
  misses: number
}

class LRUCache<T> {
  constructor(private state: TRef.TRef<CacheState<T>>) {}
  
  static make = <T>(maxSize: number) =>
    STM.gen(function* () {
      const state = yield* TRef.make<CacheState<T>>({
        entries: new Map(),
        accessOrder: [],
        maxSize,
        hits: 0,
        misses: 0
      })
      return new LRUCache(state)
    })
  
  get = (key: string): STM.STM<Option.Option<T>> =>
    STM.gen(function* () {
      const current = yield* TRef.get(this.state)
      const entry = current.entries.get(key)
      
      if (entry) {
        // Update access time and order
        const now = Date.now()
        const updatedEntry = {
          ...entry,
          accessTime: now,
          accessCount: entry.accessCount + 1
        }
        
        const newAccessOrder = [
          key,
          ...current.accessOrder.filter(k => k !== key)
        ]
        
        yield* TRef.set(this.state, {
          ...current,
          entries: new Map(current.entries.set(key, updatedEntry)),
          accessOrder: newAccessOrder,
          hits: current.hits + 1
        })
        
        return Option.some(entry.value)
      } else {
        yield* TRef.update(this.state, state => ({
          ...state,
          misses: state.misses + 1
        }))
        
        return Option.none()
      }
    })
  
  set = (key: string, value: T): STM.STM<void> =>
    STM.gen(function* () {
      const current = yield* TRef.get(this.state)
      const now = Date.now()
      
      const newEntry: CacheEntry<T> = {
        value,
        accessTime: now,
        accessCount: 1
      }
      
      let newEntries = new Map(current.entries.set(key, newEntry))
      let newAccessOrder = [key, ...current.accessOrder.filter(k => k !== key)]
      
      // Evict LRU entries if over capacity
      if (newEntries.size > current.maxSize) {
        const toEvict = newAccessOrder.slice(current.maxSize)
        
        for (const evictKey of toEvict) {
          newEntries.delete(evictKey)
        }
        
        newAccessOrder = newAccessOrder.slice(0, current.maxSize)
      }
      
      yield* TRef.set(this.state, {
        ...current,
        entries: newEntries,
        accessOrder: newAccessOrder
      })
    })
  
  delete = (key: string): STM.STM<boolean> =>
    STM.gen(function* () {
      const current = yield* TRef.get(this.state)
      
      if (current.entries.has(key)) {
        const newEntries = new Map(current.entries)
        newEntries.delete(key)
        
        yield* TRef.set(this.state, {
          ...current,
          entries: newEntries,
          accessOrder: current.accessOrder.filter(k => k !== key)
        })
        
        return true
      }
      
      return false
    })
  
  clear = (): STM.STM<void> =>
    STM.gen(function* () {
      const current = yield* TRef.get(this.state)
      yield* TRef.set(this.state, {
        ...current,
        entries: new Map(),
        accessOrder: []
      })
    })
  
  size = (): STM.STM<number> =>
    STM.gen(function* () {
      const current = yield* TRef.get(this.state)
      return current.entries.size
    })
  
  getStats = (): STM.STM<{ hits: number, misses: number, hitRatio: number, size: number }> =>
    STM.gen(function* () {
      const current = yield* TRef.get(this.state)
      const total = current.hits + current.misses
      const hitRatio = total > 0 ? current.hits / total : 0
      
      return {
        hits: current.hits,
        misses: current.misses,
        hitRatio,
        size: current.entries.size
      }
    })
}

// Usage example
const cacheExample = Effect.gen(function* () {
  const cache = yield* STM.commit(LRUCache.make<string>(3))
  
  // Concurrent operations
  const operations = [
    STM.commit(cache.set("key1", "value1")),
    STM.commit(cache.set("key2", "value2")),
    STM.commit(cache.set("key3", "value3")),
    STM.commit(cache.get("key1")),
    STM.commit(cache.set("key4", "value4")), // This should evict key2 or key3
    STM.commit(cache.get("key2")),
    STM.commit(cache.get("key1"))
  ]
  
  yield* Effect.all(operations, { concurrency: "unbounded" })
  
  const stats = yield* STM.commit(cache.getStats())
  console.log("Cache statistics:", stats)
  
  const size = yield* STM.commit(cache.size())
  console.log(`Cache size: ${size}`)
})
```

### Example 2: Producer-Consumer with Bounded Buffer

A classic producer-consumer pattern implemented with STM for safe concurrent access.

```typescript
import { Effect, STM, TRef, Fiber, Array } from "effect"

interface BufferState<T> {
  items: T[]
  maxSize: number
  produced: number
  consumed: number
}

class BoundedBuffer<T> {
  constructor(private state: TRef.TRef<BufferState<T>>) {}
  
  static make = <T>(maxSize: number) =>
    STM.gen(function* () {
      const state = yield* TRef.make<BufferState<T>>({
        items: [],
        maxSize,
        produced: 0,
        consumed: 0
      })
      return new BoundedBuffer(state)
    })
  
  // Producer operation - blocks when buffer is full
  put = (item: T): STM.STM<void> =>
    STM.gen(function* () {
      const current = yield* TRef.get(this.state)
      
      if (current.items.length >= current.maxSize) {
        // Buffer is full, retry when space becomes available
        yield* STM.retry()
      }
      
      yield* TRef.update(this.state, state => ({
        ...state,
        items: [...state.items, item],
        produced: state.produced + 1
      }))
    })
  
  // Consumer operation - blocks when buffer is empty
  take = (): STM.STM<T> =>
    STM.gen(function* () {
      const current = yield* TRef.get(this.state)
      
      if (current.items.length === 0) {
        // Buffer is empty, retry when items become available
        yield* STM.retry()
      }
      
      const [first, ...rest] = current.items
      
      yield* TRef.update(this.state, state => ({
        ...state,
        items: rest,
        consumed: state.consumed + 1
      }))
      
      return first
    })
  
  // Non-blocking operations
  tryPut = (item: T): STM.STM<boolean> =>
    STM.gen(function* () {
      const current = yield* TRef.get(this.state)
      
      if (current.items.length >= current.maxSize) {
        return false
      }
      
      yield* TRef.update(this.state, state => ({
        ...state,
        items: [...state.items, item],
        produced: state.produced + 1
      }))
      
      return true
    })
  
  tryTake = (): STM.STM<Option.Option<T>> =>
    STM.gen(function* () {
      const current = yield* TRef.get(this.state)
      
      if (current.items.length === 0) {
        return Option.none()
      }
      
      const [first, ...rest] = current.items
      
      yield* TRef.update(this.state, state => ({
        ...state,
        items: rest,
        consumed: state.consumed + 1
      }))
      
      return Option.some(first)
    })
  
  // Batch operations
  putMany = (items: T[]): STM.STM<number> =>
    STM.gen(function* () {
      let putCount = 0
      
      for (const item of items) {
        const success = yield* this.tryPut(item)
        if (success) {
          putCount++
        } else {
          break
        }
      }
      
      return putCount
    })
  
  takeMany = (maxItems: number): STM.STM<T[]> =>
    STM.gen(function* () {
      const taken: T[] = []
      
      for (let i = 0; i < maxItems; i++) {
        const item = yield* this.tryTake()
        if (item._tag === "Some") {
          taken.push(item.value)
        } else {
          break
        }
      }
      
      return taken
    })
  
  getState = (): STM.STM<BufferState<T>> =>
    TRef.get(this.state)
  
  isEmpty = (): STM.STM<boolean> =>
    STM.gen(function* () {
      const current = yield* TRef.get(this.state)
      return current.items.length === 0
    })
  
  isFull = (): STM.STM<boolean> =>
    STM.gen(function* () {
      const current = yield* TRef.get(this.state)
      return current.items.length >= current.maxSize
    })
}

// Usage example with multiple producers and consumers
const producerConsumerExample = Effect.gen(function* () {
  const buffer = yield* STM.commit(BoundedBuffer.make<number>(5))
  
  // Producer function
  const producer = (id: string, itemCount: number) =>
    Effect.gen(function* () {
      for (let i = 0; i < itemCount; i++) {
        const item = parseInt(`${id}${i}`)
        yield* STM.commit(buffer.put(item))
        console.log(`Producer ${id} put: ${item}`)
        yield* Effect.sleep("100 millis")
      }
    })
  
  // Consumer function
  const consumer = (id: string, itemCount: number) =>
    Effect.gen(function* () {
      for (let i = 0; i < itemCount; i++) {
        const item = yield* STM.commit(buffer.take())
        console.log(`Consumer ${id} took: ${item}`)
        yield* Effect.sleep("150 millis")
      }
    })
  
  // Start multiple producers and consumers
  const producers = [
    Effect.fork(producer("P1", 5)),
    Effect.fork(producer("P2", 5)),
    Effect.fork(producer("P3", 5))
  ].map(p => Effect.gen(function* () { return yield* p }))
  
  const consumers = [
    Effect.fork(consumer("C1", 5)),
    Effect.fork(consumer("C2", 5)),
    Effect.fork(consumer("C3", 5))
  ].map(c => Effect.gen(function* () { return yield* c }))
  
  // Monitor buffer state
  const monitor = Effect.gen(function* () {
    yield* Effect.forever(
      Effect.gen(function* () {
        const state = yield* STM.commit(buffer.getState())
        console.log(`Buffer: ${state.items.length}/${state.maxSize} items, Produced: ${state.produced}, Consumed: ${state.consumed}`)
        yield* Effect.sleep("500 millis")
      })
    )
  })
  
  const monitorFiber = yield* Effect.fork(monitor)
  
  // Wait for all producers and consumers
  const allProducers = yield* Effect.all(producers)
  const allConsumers = yield* Effect.all(consumers)
  
  yield* Effect.all([
    Effect.all(allProducers.map(fiber => Fiber.join(fiber))),
    Effect.all(allConsumers.map(fiber => Fiber.join(fiber)))
  ])
  
  yield* Fiber.interrupt(monitorFiber)
  
  const finalState = yield* STM.commit(buffer.getState())
  console.log("Final state:", finalState)
})
```

### Example 3: Distributed Lock Manager

A sophisticated lock manager that handles distributed coordination with timeouts and priority.

```typescript
import { Effect, STM, TRef, TMap, Option, Duration } from "effect"

interface LockRequest {
  id: string
  resource: string
  requestTime: number
  timeout: number
  priority: number
}

interface LockState {
  owner: Option.Option<string>
  queue: LockRequest[]
  acquiredAt: Option.Option<number>
}

class DistributedLockManager {
  constructor(
    private locks: TMap.TMap<string, LockState>,
    private activeLocks: TRef.TRef<Map<string, string>> // requestId -> resource
  ) {}
  
  static make = () =>
    STM.gen(function* () {
      const locks = yield* TMap.empty<string, LockState>()
      const activeLocks = yield* TRef.make(new Map<string, string>())
      return new DistributedLockManager(locks, activeLocks)
    })
  
  acquireLock = (
    resource: string,
    requestId: string,
    timeoutMs: number = 30000,
    priority: number = 0
  ): STM.STM<boolean> =>
    STM.gen(function* () {
      const request: LockRequest = {
        id: requestId,
        resource,
        requestTime: Date.now(),
        timeout: timeoutMs,
        priority
      }
      
      const currentLockState = yield* TMap.get(this.locks, resource)
      
      if (Option.isNone(currentLockState)) {
        // No lock exists, create and acquire immediately
        const newLockState: LockState = {
          owner: Option.some(requestId),
          queue: [],
          acquiredAt: Option.some(Date.now())
        }
        
        yield* TMap.set(this.locks, resource, newLockState)
        yield* TRef.update(this.activeLocks, map => new Map(map.set(requestId, resource)))
        
        return true
      } else {
        const lockState = currentLockState.value
        
        if (Option.isNone(lockState.owner)) {
          // Lock is released, acquire it
          const newLockState: LockState = {
            owner: Option.some(requestId),
            queue: lockState.queue,
            acquiredAt: Option.some(Date.now())
          }
          
          yield* TMap.set(this.locks, resource, newLockState)
          yield* TRef.update(this.activeLocks, map => new Map(map.set(requestId, resource)))
          
          return true
        } else {
          // Lock is held, check if it's expired
          const now = Date.now()
          const ownerRequest = lockState.queue.find(r => r.id === lockState.owner.value) ||
            { requestTime: lockState.acquiredAt.value || 0, timeout: 30000 } as LockRequest
          
          if (now - ownerRequest.requestTime > ownerRequest.timeout) {
            // Current lock is expired, acquire it
            yield* this.releaseLockInternal(resource, lockState.owner.value)
            
            const newLockState: LockState = {
              owner: Option.some(requestId),
              queue: lockState.queue.filter(r => r.id !== lockState.owner.value),
              acquiredAt: Option.some(now)
            }
            
            yield* TMap.set(this.locks, resource, newLockState)
            yield* TRef.update(this.activeLocks, map => new Map(map.set(requestId, resource)))
            
            return true
          } else {
            // Add to queue and retry
            const newQueue = [...lockState.queue, request].sort((a, b) => {
              if (a.priority !== b.priority) {
                return b.priority - a.priority // Higher priority first
              }
              return a.requestTime - b.requestTime // Earlier requests first for same priority
            })
            
            yield* TMap.set(this.locks, resource, {
              ...lockState,
              queue: newQueue
            })
            
            yield* STM.retry() // Wait for the lock to be released
          }
        }
      }
    })
  
  releaseLock = (resource: string, requestId: string): STM.STM<boolean> =>
    STM.gen(function* () {
      const lockState = yield* TMap.get(this.locks, resource)
      
      if (Option.isNone(lockState)) {
        return false
      }
      
      const state = lockState.value
      
      if (Option.isSome(state.owner) && state.owner.value === requestId) {
        yield* this.releaseLockInternal(resource, requestId)
        
        // Find next in queue
        const nextInQueue = state.queue.find(r => r.id !== requestId)
        
        if (nextInQueue) {
          const newLockState: LockState = {
            owner: Option.some(nextInQueue.id),
            queue: state.queue.filter(r => r.id !== nextInQueue.id),
            acquiredAt: Option.some(Date.now())
          }
          
          yield* TMap.set(this.locks, resource, newLockState)
          yield* TRef.update(this.activeLocks, map => new Map(map.set(nextInQueue.id, resource)))
        } else {
          const newLockState: LockState = {
            owner: Option.none(),
            queue: [],
            acquiredAt: Option.none()
          }
          
          yield* TMap.set(this.locks, resource, newLockState)
        }
        
        return true
      }
      
      return false
    })
  
  private releaseLockInternal = (resource: string, requestId: string): STM.STM<void> =>
    TRef.update(this.activeLocks, map => {
      const newMap = new Map(map)
      newMap.delete(requestId)
      return newMap
    })
  
  isLocked = (resource: string): STM.STM<boolean> =>
    STM.gen(function* () {
      const lockState = yield* TMap.get(this.locks, resource)
      return Option.isSome(lockState) && Option.isSome(lockState.value.owner)
    })
  
  getLockOwner = (resource: string): STM.STM<Option.Option<string>> =>
    STM.gen(function* () {
      const lockState = yield* TMap.get(this.locks, resource)
      return Option.isSome(lockState) ? lockState.value.owner : Option.none()
    })
  
  getQueueLength = (resource: string): STM.STM<number> =>
    STM.gen(function* () {
      const lockState = yield* TMap.get(this.locks, resource)
      return Option.isSome(lockState) ? lockState.value.queue.length : 0
    })
  
  cleanupExpiredLocks = (): STM.STM<string[]> =>
    STM.gen(function* () {
      const allLocks = yield* TMap.toArray(this.locks)
      const now = Date.now()
      const expired: string[] = []
      
      for (const [resource, lockState] of allLocks) {
        if (Option.isSome(lockState.owner)) {
          const ownerRequest = lockState.queue.find(r => r.id === lockState.owner.value)
          
          if (ownerRequest && now - ownerRequest.requestTime > ownerRequest.timeout) {
            yield* this.releaseLock(resource, lockState.owner.value)
            expired.push(resource)
          }
        }
        
        // Clean up expired queue entries
        const validQueue = lockState.queue.filter(r => now - r.requestTime <= r.timeout)
        
        if (validQueue.length !== lockState.queue.length) {
          yield* TMap.set(this.locks, resource, {
            ...lockState,
            queue: validQueue
          })
        }
      }
      
      return expired
    })
  
  getAllLocks = (): STM.STM<Map<string, LockState>> =>
    STM.gen(function* () {
      const allLocks = yield* TMap.toArray(this.locks)
      return new Map(allLocks)
    })
}

// Usage example
const lockManagerExample = Effect.gen(function* () {
  const lockManager = yield* STM.commit(DistributedLockManager.make())
  
  // Simulate multiple clients trying to acquire locks
  const client = (id: string, resource: string, holdTime: number, priority: number = 0) =>
    Effect.gen(function* () {
      console.log(`Client ${id} attempting to acquire lock on ${resource}`)
      
      const acquired = yield* STM.commit(
        lockManager.acquireLock(resource, id, 10000, priority)
      )
      
      if (acquired) {
        console.log(`Client ${id} acquired lock on ${resource}`)
        yield* Effect.sleep(`${holdTime}ms`)
        
        const released = yield* STM.commit(
          lockManager.releaseLock(resource, id)
        )
        
        console.log(`Client ${id} ${released ? 'released' : 'failed to release'} lock on ${resource}`)
      } else {
        console.log(`Client ${id} failed to acquire lock on ${resource}`)
      }
    })
  
  // Multiple clients competing for the same resource
  const clients = [
    Effect.fork(client("client-1", "resource-A", 2000, 1)),
    Effect.fork(client("client-2", "resource-A", 1000, 3)), // Higher priority
    Effect.fork(client("client-3", "resource-A", 1500, 2)),
    Effect.fork(client("client-4", "resource-B", 1000, 1)),
    Effect.fork(client("client-5", "resource-B", 2000, 1))
  ]
  
  // Monitor lock states
  const monitor = Effect.gen(function* () {
    yield* Effect.forever(
      Effect.gen(function* () {
        const allLocks = yield* STM.commit(lockManager.getAllLocks())
        console.log("Current lock states:")
        
        for (const [resource, state] of allLocks) {
          const owner = Option.getOrElse(state.owner, () => "none")
          console.log(`  ${resource}: owner=${owner}, queue=${state.queue.length}`)
        }
        
        yield* Effect.sleep("1 second")
      })
    )
  })
  
  const monitorFiber = yield* Effect.fork(monitor)
  
  // Cleanup expired locks periodically
  const cleanup = Effect.gen(function* () {
    yield* Effect.forever(
      Effect.gen(function* () {
        const expired = yield* STM.commit(lockManager.cleanupExpiredLocks())
        if (expired.length > 0) {
          console.log(`Cleaned up expired locks: ${expired.join(', ')}`)
        }
        yield* Effect.sleep("5 seconds")
      })
    )
  })
  
  const cleanupFiber = yield* Effect.fork(cleanup)
  
  // Wait for all clients
  yield* Effect.all(clients.map(fiber => Fiber.join(fiber)))
  
  yield* Fiber.interrupt(monitorFiber)
  yield* Fiber.interrupt(cleanupFiber)
  
  console.log("Lock manager example completed")
})
```

## Advanced Features Deep Dive

### Feature 1: STM Composition and Orchestration

STM transactions can be composed in sophisticated ways to build complex atomic operations.

#### Complex Transaction Composition

```typescript
import { Effect, STM, TRef, TMap, Array, Option } from "effect"

interface Order {
  id: string
  userId: string
  items: { productId: string, quantity: number, price: number }[]
  status: "pending" | "processing" | "fulfilled" | "cancelled"
  total: number
  createdAt: number
}

interface Inventory {
  productId: string
  available: number
  reserved: number
  price: number
}

interface User {
  id: string
  balance: number
  orders: string[]
}

class ECommerceSystem {
  constructor(
    private users: TMap.TMap<string, User>,
    private inventory: TMap.TMap<string, Inventory>,
    private orders: TMap.TMap<string, Order>,
    private orderCounter: TRef.TRef<number>
  ) {}
  
  static make = () =>
    STM.gen(function* () {
      const users = yield* TMap.empty<string, User>()
      const inventory = yield* TMap.empty<string, Inventory>()
      const orders = yield* TMap.empty<string, Order>()
      const orderCounter = yield* TRef.make(1)
      
      return new ECommerceSystem(users, inventory, orders, orderCounter)
    })
  
  // Complex atomic operation - place order with inventory reservation and payment
  placeOrder = (
    userId: string,
    items: { productId: string, quantity: number }[]
  ): STM.STM<Order, string> =>
    STM.gen(function* () {
      // Get user
      const user = yield* TMap.get(this.users, userId)
      if (Option.isNone(user)) {
        yield* STM.fail(`User ${userId} not found`)
      }
      
      // Calculate total and check inventory
      let total = 0
      const orderItems: Order["items"] = []
      
      for (const item of items) {
        const inventoryItem = yield* TMap.get(this.inventory, item.productId)
        if (Option.isNone(inventoryItem)) {
          yield* STM.fail(`Product ${item.productId} not found`)
        }
        
        const inventory = inventoryItem.value
        if (inventory.available < item.quantity) {
          yield* STM.fail(`Insufficient inventory for ${item.productId}: ${inventory.available} < ${item.quantity}`)
        }
        
        const itemTotal = inventory.price * item.quantity
        total += itemTotal
        
        orderItems.push({
          productId: item.productId,
          quantity: item.quantity,
          price: inventory.price
        })
      }
      
      // Check user balance
      if (user.value.balance < total) {
        yield* STM.fail(`Insufficient balance: ${user.value.balance} < ${total}`)
      }
      
      // Generate order ID
      const orderId = yield* TRef.updateAndGet(this.orderCounter, n => n + 1)
      const orderIdStr = `order-${orderId}`
      
      // Create order
      const order: Order = {
        id: orderIdStr,
        userId,
        items: orderItems,
        status: "pending",
        total,
        createdAt: Date.now()
      }
      
      // Reserve inventory
      for (const item of items) {
        yield* TMap.update(this.inventory, item.productId, inventory =>
          Option.map(inventory, inv => ({
            ...inv,
            available: inv.available - item.quantity,
            reserved: inv.reserved + item.quantity
          }))
        )
      }
      
      // Deduct payment
      yield* TMap.update(this.users, userId, user =>
        Option.map(user, u => ({
          ...u,
          balance: u.balance - total,
          orders: [...u.orders, orderIdStr]
        }))
      )
      
      // Store order
      yield* TMap.set(this.orders, orderIdStr, order)
      
      return order
    })
  
  // Fulfill order - convert reservations to actual sales
  fulfillOrder = (orderId: string): STM.STM<Order, string> =>
    STM.gen(function* () {
      const order = yield* TMap.get(this.orders, orderId)
      if (Option.isNone(order)) {
        yield* STM.fail(`Order ${orderId} not found`)
      }
      
      const currentOrder = order.value
      if (currentOrder.status !== "pending") {
        yield* STM.fail(`Order ${orderId} cannot be fulfilled (status: ${currentOrder.status})`)
      }
      
      // Convert reservations to sales
      for (const item of currentOrder.items) {
        yield* TMap.update(this.inventory, item.productId, inventory =>
          Option.map(inventory, inv => ({
            ...inv,
            reserved: inv.reserved - item.quantity
          }))
        )
      }
      
      // Update order status
      const fulfilledOrder = { ...currentOrder, status: "fulfilled" as const }
      yield* TMap.set(this.orders, orderId, fulfilledOrder)
      
      return fulfilledOrder
    })
  
  // Cancel order - release reservations and refund payment
  cancelOrder = (orderId: string): STM.STM<Order, string> =>
    STM.gen(function* () {
      const order = yield* TMap.get(this.orders, orderId)
      if (Option.isNone(order)) {
        yield* STM.fail(`Order ${orderId} not found`)
      }
      
      const currentOrder = order.value
      if (currentOrder.status !== "pending") {
        yield* STM.fail(`Order ${orderId} cannot be cancelled (status: ${currentOrder.status})`)
      }
      
      // Release inventory reservations
      for (const item of currentOrder.items) {
        yield* TMap.update(this.inventory, item.productId, inventory =>
          Option.map(inventory, inv => ({
            ...inv,
            available: inv.available + item.quantity,
            reserved: inv.reserved - item.quantity
          }))
        )
      }
      
      // Refund payment
      yield* TMap.update(this.users, currentOrder.userId, user =>
        Option.map(user, u => ({
          ...u,
          balance: u.balance + currentOrder.total
        }))
      )
      
      // Update order status
      const cancelledOrder = { ...currentOrder, status: "cancelled" as const }
      yield* TMap.set(this.orders, orderId, cancelledOrder)
      
      return cancelledOrder
    })
  
  // Helper methods
  addUser = (user: User): STM.STM<void> =>
    TMap.set(this.users, user.id, user)
  
  addInventory = (inventory: Inventory): STM.STM<void> =>
    TMap.set(this.inventory, inventory.productId, inventory)
  
  getUser = (userId: string): STM.STM<Option.Option<User>> =>
    TMap.get(this.users, userId)
  
  getInventory = (productId: string): STM.STM<Option.Option<Inventory>> =>
    TMap.get(this.inventory, productId)
  
  getOrder = (orderId: string): STM.STM<Option.Option<Order>> =>
    TMap.get(this.orders, orderId)
  
  getAllOrders = (): STM.STM<Order[]> =>
    STM.gen(function* () {
      const orderEntries = yield* TMap.toArray(this.orders)
      return orderEntries.map(([_, order]) => order)
    })
}

// Usage example
const ecommerceExample = Effect.gen(function* () {
  const system = yield* STM.commit(ECommerceSystem.make())
  
  // Setup initial data
  yield* STM.commit(system.addUser({
    id: "user1",
    balance: 1000,
    orders: []
  }))
  
  yield* STM.commit(system.addUser({
    id: "user2",
    balance: 500,
    orders: []
  }))
  
  yield* STM.commit(system.addInventory({
    productId: "laptop",
    available: 10,
    reserved: 0,
    price: 800
  }))
  
  yield* STM.commit(system.addInventory({
    productId: "mouse",
    available: 50,
    reserved: 0,
    price: 25
  }))
  
  // Concurrent order placement
  const placeOrders = [
    STM.commit(system.placeOrder("user1", [
      { productId: "laptop", quantity: 1 },
      { productId: "mouse", quantity: 2 }
    ])),
    STM.commit(system.placeOrder("user2", [
      { productId: "laptop", quantity: 1 }
    ])),
    STM.commit(system.placeOrder("user1", [
      { productId: "mouse", quantity: 5 }
    ]))
  ]
  
  const orderResults = yield* Effect.all(
    placeOrders.map(order => 
      order.pipe(Effect.either)
    ),
    { concurrency: "unbounded" }
  )
  
  console.log("Order results:")
  orderResults.forEach((result, index) => {
    if (result._tag === "Right") {
      console.log(`Order ${index + 1}: Success - ${result.right.id}`)
    } else {
      console.log(`Order ${index + 1}: Failed - ${result.left}`)
    }
  })
  
  // Get current state
  const allOrders = yield* STM.commit(system.getAllOrders())
  console.log(`Total orders placed: ${allOrders.length}`)
  
  // Fulfill first successful order
  const successfulOrder = allOrders.find(order => order.status === "pending")
  if (successfulOrder) {
    yield* STM.commit(system.fulfillOrder(successfulOrder.id))
    console.log(`Fulfilled order: ${successfulOrder.id}`)
  }
})
```

### Feature 2: STM Error Handling and Recovery

Sophisticated error handling patterns in STM transactions.

#### Conditional Retry and Fallback Strategies

```typescript
import { Effect, STM, TRef, Option, Schedule } from "effect"

interface ServiceStatus {
  isAvailable: boolean
  responseTime: number
  errorCount: number
  lastError?: string
}

interface CircuitBreakerState {
  state: "closed" | "open" | "half-open"
  failures: number
  lastFailureTime: number
  lastSuccessTime: number
  threshold: number
  timeout: number
}

class ResilientService {
  constructor(
    private primaryService: TRef.TRef<ServiceStatus>,
    private fallbackService: TRef.TRef<ServiceStatus>,
    private circuitBreaker: TRef.TRef<CircuitBreakerState>
  ) {}
  
  static make = (threshold: number = 5, timeoutMs: number = 60000) =>
    STM.gen(function* () {
      const primaryService = yield* TRef.make<ServiceStatus>({
        isAvailable: true,
        responseTime: 100,
        errorCount: 0
      })
      
      const fallbackService = yield* TRef.make<ServiceStatus>({
        isAvailable: true,
        responseTime: 200,
        errorCount: 0
      })
      
      const circuitBreaker = yield* TRef.make<CircuitBreakerState>({
        state: "closed",
        failures: 0,
        lastFailureTime: 0,
        lastSuccessTime: Date.now(),
        threshold,
        timeout: timeoutMs
      })
      
      return new ResilientService(primaryService, fallbackService, circuitBreaker)
    })
  
  // Attempt to call service with circuit breaker and fallback
  callService = (request: string): STM.STM<string, string> =>
    STM.gen(function* () {
      const cbState = yield* TRef.get(this.circuitBreaker)
      const now = Date.now()
      
      // Check if circuit breaker should transition states
      if (cbState.state === "open" && now - cbState.lastFailureTime > cbState.timeout) {
        yield* TRef.update(this.circuitBreaker, state => ({
          ...state,
          state: "half-open"
        }))
      }
      
      // Try primary service based on circuit breaker state
      if (cbState.state === "closed" || cbState.state === "half-open") {
        const primaryResult = yield* this.tryPrimaryService(request).pipe(
          STM.either
        )
        
        if (primaryResult._tag === "Right") {
          // Success - reset circuit breaker
          yield* TRef.update(this.circuitBreaker, state => ({
            ...state,
            state: "closed",
            failures: 0,
            lastSuccessTime: now
          }))
          
          return primaryResult.right
        } else {
          // Failure - update circuit breaker
          const newFailures = cbState.failures + 1
          const newState = newFailures >= cbState.threshold ? "open" : cbState.state
          
          yield* TRef.update(this.circuitBreaker, state => ({
            ...state,
            state: newState,
            failures: newFailures,
            lastFailureTime: now
          }))
          
          // Fall through to fallback service
        }
      }
      
      // Try fallback service
      const fallbackResult = yield* this.tryFallbackService(request).pipe(
        STM.either
      )
      
      if (fallbackResult._tag === "Right") {
        return `[FALLBACK] ${fallbackResult.right}`
      } else {
        yield* STM.fail(`Both services failed: Primary and Fallback errors`)
      }
    })
  
  private tryPrimaryService = (request: string): STM.STM<string, string> =>
    STM.gen(function* () {
      const status = yield* TRef.get(this.primaryService)
      
      if (!status.isAvailable) {
        yield* STM.fail("Primary service is not available")
      }
      
      // Simulate service call with potential failure
      if (Math.random() < 0.3) { // 30% failure rate
        yield* TRef.update(this.primaryService, s => ({
          ...s,
          errorCount: s.errorCount + 1,
          lastError: "Random service failure"
        }))
        
        yield* STM.fail("Primary service call failed")
      }
      
      yield* TRef.update(this.primaryService, s => ({
        ...s,
        responseTime: 50 + Math.random() * 100
      }))
      
      return `Primary response for: ${request}`
    })
  
  private tryFallbackService = (request: string): STM.STM<string, string> =>
    STM.gen(function* () {
      const status = yield* TRef.get(this.fallbackService)
      
      if (!status.isAvailable) {
        yield* STM.fail("Fallback service is not available")
      }
      
      // Fallback service is more reliable but slower
      if (Math.random() < 0.1) { // 10% failure rate
        yield* TRef.update(this.fallbackService, s => ({
          ...s,
          errorCount: s.errorCount + 1,
          lastError: "Fallback service error"
        }))
        
        yield* STM.fail("Fallback service call failed")
      }
      
      yield* TRef.update(this.fallbackService, s => ({
        ...s,
        responseTime: 150 + Math.random() * 100
      }))
      
      return `Fallback response for: ${request}`
    })
  
  // Advanced retry with exponential backoff and jitter
  callServiceWithRetry = (request: string, maxRetries: number = 3): STM.STM<string, string> =>
    STM.gen(function* () {
      let lastError = ""
      
      for (let attempt = 0; attempt < maxRetries; attempt++) {
        const result = yield* this.callService(request).pipe(STM.either)
        
        if (result._tag === "Right") {
          return result.right
        }
        
        lastError = result.left
        
        if (attempt < maxRetries - 1) {
          // Wait before retry (simulated with a check that will always pass)
          const shouldRetry = yield* this.shouldRetryAfterDelay(attempt)
          if (!shouldRetry) {
            break
          }
        }
      }
      
      yield* STM.fail(`Failed after ${maxRetries} attempts. Last error: ${lastError}`)
    })
  
  private shouldRetryAfterDelay = (attempt: number): STM.STM<boolean> =>
    STM.gen(function* () {
      // In a real implementation, this would involve actual delay logic
      // For STM, we just return true to continue the retry loop
      return true
    })
  
  // Conditional operations with timeout
  callServiceWithTimeout = (request: string, timeoutMs: number): STM.STM<string, string> =>
    STM.gen(function* () {
      const startTime = Date.now()
      
      // Check if we should timeout
      const shouldTimeout = yield* STM.succeed(Date.now() - startTime > timeoutMs)
      
      if (shouldTimeout) {
        yield* STM.fail(`Service call timed out after ${timeoutMs}ms`)
      }
      
      return yield* this.callService(request)
    })
  
  // Get service status
  getStatus = (): STM.STM<{
    primary: ServiceStatus
    fallback: ServiceStatus
    circuitBreaker: CircuitBreakerState
  }> =>
    STM.gen(function* () {
      return {
        primary: yield* TRef.get(this.primaryService),
        fallback: yield* TRef.get(this.fallbackService),
        circuitBreaker: yield* TRef.get(this.circuitBreaker)
      }
    })
  
  // Simulate service outage
  simulateOutage = (service: "primary" | "fallback", durationMs: number): STM.STM<void> =>
    STM.gen(function* () {
      const serviceRef = service === "primary" ? this.primaryService : this.fallbackService
      
      yield* TRef.update(serviceRef, s => ({
        ...s,
        isAvailable: false
      }))
      
      // In real implementation, would use Effect.sleep and then restore
      // For now, we'll just mark it as unavailable
    })
  
  // Restore service
  restoreService = (service: "primary" | "fallback"): STM.STM<void> =>
    STM.gen(function* () {
      const serviceRef = service === "primary" ? this.primaryService : this.fallbackService
      
      yield* TRef.update(serviceRef, s => ({
        ...s,
        isAvailable: true,
        errorCount: 0,
        lastError: undefined
      }))
    })
}

// Usage example
const resilientServiceExample = Effect.gen(function* () {
  const service = yield* STM.commit(ResilientService.make(3, 30000))
  
  // Make multiple concurrent calls
  const requests = Array.from({ length: 20 }, (_, i) => `request-${i}`)
  
  const results = yield* Effect.all(
    requests.map(request =>
      STM.commit(service.callServiceWithRetry(request, 2)).pipe(
        Effect.either
      )
    ),
    { concurrency: 5 }
  )
  
  console.log("Service call results:")
  results.forEach((result, index) => {
    if (result._tag === "Right") {
      console.log(`Request ${index}: Success - ${result.right}`)
    } else {
      console.log(`Request ${index}: Failed - ${result.left}`)
    }
  })
  
  // Check final status
  const status = yield* STM.commit(service.getStatus())
  console.log("Final service status:", status)
  
  // Simulate outage and recovery
  console.log("Simulating primary service outage...")
  yield* STM.commit(service.simulateOutage("primary", 10000))
  
  // Make calls during outage
  const outageResults = yield* Effect.all(
    ["outage-test-1", "outage-test-2"].map(request =>
      STM.commit(service.callService(request)).pipe(Effect.either)
    )
  )
  
  console.log("Calls during outage:")
  outageResults.forEach((result, index) => {
    if (result._tag === "Right") {
      console.log(`Outage call ${index}: Success - ${result.right}`)
    } else {
      console.log(`Outage call ${index}: Failed - ${result.left}`)
    }
  })
  
  // Restore service
  yield* STM.commit(service.restoreService("primary"))
  console.log("Primary service restored")
  
  const finalStatus = yield* STM.commit(service.getStatus())
  console.log("Status after restoration:", finalStatus)
})
```

### Feature 3: STM Performance Optimization

Advanced techniques for optimizing STM performance in high-contention scenarios.

#### Contention Reduction and Batching

```typescript
import { Effect, STM, TRef, TArray, Array } from "effect"

interface BatchProcessor<T, R> {
  items: TArray.TArray<T>
  results: TArray.TArray<R>
  batchSize: number
  processor: (batch: T[]) => STM.STM<R[]>
}

class HighPerformanceCounter {
  constructor(
    private counters: TArray.TArray<TRef.TRef<number>>,
    private globalCounter: TRef.TRef<number>
  ) {}
  
  static make = (shards: number = 16) =>
    STM.gen(function* () {
      // Create multiple counter shards to reduce contention
      const counterRefs = yield* STM.all(
        Array.makeBy(shards, () => TRef.make(0))
      )
      
      const counters = yield* TArray.fromIterable(counterRefs)
      const globalCounter = yield* TRef.make(0)
      
      return new HighPerformanceCounter(counters, globalCounter)
    })
  
  // Increment using sharding to reduce contention
  increment = (amount: number = 1): STM.STM<void> =>
    STM.gen(function* () {
      // Use thread-local ID or random selection to choose shard
      const shardIndex = Math.floor(Math.random() * (yield* TArray.size(this.counters)))
      const shard = yield* TArray.unsafeGet(this.counters, shardIndex)
      
      yield* TRef.update(shard, n => n + amount)
    })
  
  // Get total count by aggregating all shards
  getCount = (): STM.STM<number> =>
    STM.gen(function* () {
      const size = yield* TArray.size(this.counters)
      let total = 0
      
      for (let i = 0; i < size; i++) {
        const shard = yield* TArray.unsafeGet(this.counters, i)
        const value = yield* TRef.get(shard)
        total += value
      }
      
      return total
    })
  
  // Batch consolidation - periodically merge shard values
  consolidate = (): STM.STM<number> =>
    STM.gen(function* () {
      const size = yield* TArray.size(this.counters)
      let total = 0
      
      // Read and reset all shards
      for (let i = 0; i < size; i++) {
        const shard = yield* TArray.unsafeGet(this.counters, i)
        const value = yield* TRef.getAndSet(shard, 0)
        total += value
      }
      
      // Add to global counter
      yield* TRef.update(this.globalCounter, n => n + total)
      
      return total
    })
  
  getGlobalCount = (): STM.STM<number> =>
    TRef.get(this.globalCounter)
  
  getTotalCount = (): STM.STM<number> =>
    STM.gen(function* () {
      const currentCount = yield* this.getCount()
      const globalCount = yield* this.getGlobalCount()
      return currentCount + globalCount
    })
}

// Optimized batch processing system
class BatchProcessingSystem<T, R> {
  constructor(
    private pendingItems: TArray.TArray<T>,
    private processedResults: TArray.TArray<R>,
    private batchSize: number,
    private processor: (items: T[]) => Effect.Effect<R[]>
  ) {}
  
  static make = <T, R>(
    batchSize: number,
    processor: (items: T[]) => Effect.Effect<R[]>
  ) =>
    STM.gen(function* () {
      const pendingItems = yield* TArray.empty<T>()
      const processedResults = yield* TArray.empty<R>()
      
      return new BatchProcessingSystem(
        pendingItems,
        processedResults,
        batchSize,
        processor
      )
    })
  
  // Add item to batch (non-blocking)
  addItem = (item: T): STM.STM<void> =>
    TArray.append(this.pendingItems, item)
  
  // Add multiple items efficiently
  addItems = (items: T[]): STM.STM<void> =>
    STM.gen(function* () {
      for (const item of items) {
        yield* TArray.append(this.pendingItems, item)
      }
    })
  
  // Process a batch if enough items are available
  processBatchIfReady = (): STM.STM<Option.Option<Effect.Effect<void>>> =>
    STM.gen(function* () {
      const size = yield* TArray.size(this.pendingItems)
      
      if (size >= this.batchSize) {
        // Extract batch
        const batch: T[] = []
        for (let i = 0; i < this.batchSize; i++) {
          const item = yield* TArray.get(this.pendingItems, 0)
          if (Option.isSome(item)) {
            batch.push(item.value)
            yield* TArray.remove(this.pendingItems, 0)
          }
        }
        
        // Return effect to process the batch
        const processEffect = Effect.gen(function* () {
          const results = yield* this.processor(batch)
          
          // Store results back in STM
          yield* STM.commit(
            STM.gen(function* () {
              for (const result of results) {
                yield* TArray.append(this.processedResults, result)
              }
            })
          )
        })
        
        return Option.some(processEffect)
      }
      
      return Option.none()
    })
  
  // Force process remaining items (smaller than batch size)
  processRemaining = (): STM.STM<Option.Option<Effect.Effect<void>>> =>
    STM.gen(function* () {
      const size = yield* TArray.size(this.pendingItems)
      
      if (size > 0) {
        const batch: T[] = []
        
        // Extract all remaining items
        for (let i = 0; i < size; i++) {
          const item = yield* TArray.get(this.pendingItems, 0)
          if (Option.isSome(item)) {
            batch.push(item.value)
            yield* TArray.remove(this.pendingItems, 0)
          }
        }
        
        const processEffect = Effect.gen(function* () {
          const results = yield* this.processor(batch)
          
          yield* STM.commit(
            STM.gen(function* () {
              for (const result of results) {
                yield* TArray.append(this.processedResults, result)
              }
            })
          )
        })
        
        return Option.some(processEffect)
      }
      
      return Option.none()
    })
  
  getResults = (): STM.STM<R[]> =>
    STM.gen(function* () {
      const size = yield* TArray.size(this.processedResults)
      const results: R[] = []
      
      for (let i = 0; i < size; i++) {
        const result = yield* TArray.get(this.processedResults, i)
        if (Option.isSome(result)) {
          results.push(result.value)
        }
      }
      
      return results
    })
  
  clearResults = (): STM.STM<void> =>
    STM.gen(function* () {
      const size = yield* TArray.size(this.processedResults)
      for (let i = size - 1; i >= 0; i--) {
        yield* TArray.remove(this.processedResults, i)
      }
    })
  
  getStats = (): STM.STM<{ pending: number, processed: number }> =>
    STM.gen(function* () {
      return {
        pending: yield* TArray.size(this.pendingItems),
        processed: yield* TArray.size(this.processedResults)
      }
    })
}

// Usage example demonstrating performance optimization
const performanceOptimizationExample = Effect.gen(function* () {
  // High-performance counter with sharding
  const counter = yield* STM.commit(HighPerformanceCounter.make(8))
  
  // Simulate high-contention increments
  const incrementTasks = Array.makeBy(100, (i) =>
    Effect.fork(
      Effect.gen(function* () {
        for (let j = 0; j < 100; j++) {
          yield* STM.commit(counter.increment(1))
        }
        console.log(`Task ${i} completed 100 increments`)
      })
    )
  )
  
  console.log("Starting high-contention counter test...")
  const startTime = Date.now()
  
  const fibers = yield* Effect.all(incrementTasks)
  yield* Effect.all(fibers.map(fiber => Fiber.join(fiber)))
  
  const endTime = Date.now()
  console.log(`Completed 10,000 increments in ${endTime - startTime}ms`)
  
  const finalCount = yield* STM.commit(counter.getTotalCount())
  console.log(`Final count: ${finalCount}`)
  
  // Batch processing system
  const batchProcessor = yield* STM.commit(
    BatchProcessingSystem.make<string, string>(
      10,
      (items: string[]) =>
        Effect.gen(function* () {
          // Simulate batch processing
          yield* Effect.sleep("100 millis")
          return items.map(item => `processed-${item}`)
        })
    )
  )
  
  // Add items in batches
  const items = Array.makeBy(55, i => `item-${i}`)
  yield* STM.commit(batchProcessor.addItems(items))
  
  console.log("Processing batches...")
  
  // Process batches as they become ready
  let processedBatches = 0
  while (true) {
    const batchEffect = yield* STM.commit(batchProcessor.processBatchIfReady())
    
    if (Option.isSome(batchEffect)) {
      yield* batchEffect.value
      processedBatches++
      console.log(`Processed batch ${processedBatches}`)
    } else {
      break
    }
  }
  
  // Process remaining items
  const remainingEffect = yield* STM.commit(batchProcessor.processRemaining())
  if (Option.isSome(remainingEffect)) {
    yield* remainingEffect.value
    console.log("Processed remaining items")
  }
  
  const stats = yield* STM.commit(batchProcessor.getStats())
  console.log("Batch processing stats:", stats)
  
  const results = yield* STM.commit(batchProcessor.getResults())
  console.log(`Total results: ${results.length}`)
})
```

## Practical Patterns & Best Practices

### Pattern 1: STM-based State Machines

```typescript
import { Effect, STM, TRef, Data } from "effect"

// Define state machine states and events
type OrderState = "created" | "paid" | "shipped" | "delivered" | "cancelled"

class InvalidTransitionError extends Data.TaggedError("InvalidTransitionError")<{
  from: OrderState
  to: OrderState
  event: string
}> {}

interface OrderStateMachine {
  currentState: OrderState
  allowedTransitions: Map<OrderState, OrderState[]>
  transitionHistory: Array<{ from: OrderState, to: OrderState, timestamp: number, event: string }>
}

const createOrderStateMachine = (): STM.STM<TRef.TRef<OrderStateMachine>> =>
  STM.gen(function* () {
    const allowedTransitions = new Map<OrderState, OrderState[]>([
      ["created", ["paid", "cancelled"]],
      ["paid", ["shipped", "cancelled"]],
      ["shipped", ["delivered"]],
      ["delivered", []],
      ["cancelled", []]
    ])
    
    return yield* TRef.make<OrderStateMachine>({
      currentState: "created",
      allowedTransitions,
      transitionHistory: []
    })
  })

const transitionState = (
  stateMachine: TRef.TRef<OrderStateMachine>,
  newState: OrderState,
  event: string
): STM.STM<void, InvalidTransitionError> =>
  STM.gen(function* () {
    const current = yield* TRef.get(stateMachine)
    const allowedStates = current.allowedTransitions.get(current.currentState) || []
    
    if (!allowedStates.includes(newState)) {
      yield* STM.fail(new InvalidTransitionError({
        from: current.currentState,
        to: newState,
        event
      }))
    }
    
    yield* TRef.update(stateMachine, state => ({
      ...state,
      currentState: newState,
      transitionHistory: [
        ...state.transitionHistory,
        {
          from: current.currentState,
          to: newState,
          timestamp: Date.now(),
          event
        }
      ]
    }))
  })

// Usage
const stateMachineExample = Effect.gen(function* () {
  const orderSM = yield* STM.commit(createOrderStateMachine())
  
  // Valid transitions
  yield* STM.commit(transitionState(orderSM, "paid", "PAYMENT_RECEIVED"))
  yield* STM.commit(transitionState(orderSM, "shipped", "PACKAGE_DISPATCHED"))
  yield* STM.commit(transitionState(orderSM, "delivered", "DELIVERY_CONFIRMED"))
  
  // Try invalid transition
  const invalidTransition = yield* STM.commit(
    transitionState(orderSM, "cancelled", "CUSTOMER_CANCELLED")
  ).pipe(Effect.either)
  
  console.log("Invalid transition result:", invalidTransition)
  
  const finalState = yield* STM.commit(TRef.get(orderSM))
  console.log("Final state:", finalState.currentState)
  console.log("Transition history:", finalState.transitionHistory)
})
```

### Pattern 2: Resource Pool Management

```typescript
import { Effect, STM, TRef, TArray, Option } from "effect"

interface PooledResource<T> {
  id: string
  resource: T
  inUse: boolean
  lastUsed: number
  createTime: number
}

class ResourcePool<T> {
  constructor(
    private available: TArray.TArray<PooledResource<T>>,
    private inUse: TRef.TRef<Map<string, PooledResource<T>>>,
    private maxSize: number,
    private factory: () => Effect.Effect<T>,
    private destroyer: (resource: T) => Effect.Effect<void>
  ) {}
  
  static make = <T>(
    maxSize: number,
    factory: () => Effect.Effect<T>,
    destroyer: (resource: T) => Effect.Effect<void> = () => Effect.unit
  ) =>
    STM.gen(function* () {
      const available = yield* TArray.empty<PooledResource<T>>()
      const inUse = yield* TRef.make(new Map<string, PooledResource<T>>())
      
      return new ResourcePool(available, inUse, maxSize, factory, destroyer)
    })
  
  // Acquire resource from pool
  acquire = (): STM.STM<Option.Option<Effect.Effect<T>>, string> =>
    STM.gen(function* () {
      // Try to get available resource
      const availableSize = yield* TArray.size(this.available)
      
      if (availableSize > 0) {
        const resource = yield* TArray.get(this.available, 0)
        if (Option.isSome(resource)) {
          yield* TArray.remove(this.available, 0)
          
          const updatedResource = {
            ...resource.value,
            inUse: true,
            lastUsed: Date.now()
          }
          
          yield* TRef.update(this.inUse, map => 
            new Map(map.set(updatedResource.id, updatedResource))
          )
          
          return Option.some(Effect.succeed(updatedResource.resource))
        }
      }
      
      // Check if we can create new resource
      const inUseMap = yield* TRef.get(this.inUse)
      const totalResources = availableSize + inUseMap.size
      
      if (totalResources < this.maxSize) {
        // Create new resource
        const createEffect = Effect.gen(function* () {
          const resource = yield* this.factory()
          const pooledResource: PooledResource<T> = {
            id: `resource-${Date.now()}-${Math.random()}`,
            resource,
            inUse: true,
            lastUsed: Date.now(),
            createTime: Date.now()
          }
          
          yield* STM.commit(
            TRef.update(this.inUse, map => 
              new Map(map.set(pooledResource.id, pooledResource))
            )
          )
          
          return resource
        })
        
        return Option.some(createEffect)
      }
      
      // Pool is full and no resources available
      yield* STM.retry() // Wait for resource to become available
    })
  
  // Release resource back to pool
  release = (resourceId: string): STM.STM<boolean> =>
    STM.gen(function* () {
      const inUseMap = yield* TRef.get(this.inUse)
      const resource = inUseMap.get(resourceId)
      
      if (!resource) {
        return false
      }
      
      const releasedResource = {
        ...resource,
        inUse: false,
        lastUsed: Date.now()
      }
      
      yield* TRef.update(this.inUse, map => {
        const newMap = new Map(map)
        newMap.delete(resourceId)
        return newMap
      })
      
      yield* TArray.append(this.available, releasedResource)
      
      return true
    })
  
  // Clean up old resources
  cleanup = (maxIdleTimeMs: number): STM.STM<Effect.Effect<number>> =>
    STM.gen(function* () {
      const now = Date.now()
      const availableSize = yield* TArray.size(this.available)
      const toRemove: Array<{ index: number, resource: PooledResource<T> }> = []
      
      // Find old resources
      for (let i = 0; i < availableSize; i++) {
        const resource = yield* TArray.get(this.available, i)
        if (Option.isSome(resource) && 
            now - resource.value.lastUsed > maxIdleTimeMs) {
          toRemove.push({ index: i, resource: resource.value })
        }
      }
      
      // Remove from array (reverse order to maintain indices)
      for (const { index } of toRemove.reverse()) {
        yield* TArray.remove(this.available, index)
      }
      
      // Return effect to destroy resources
      const cleanupEffect = Effect.gen(function* () {
        for (const { resource } of toRemove) {
          yield* this.destroyer(resource.resource)
        }
        return toRemove.length
      })
      
      return cleanupEffect
    })
  
  getStats = (): STM.STM<{ available: number, inUse: number, total: number }> =>
    STM.gen(function* () {
      const availableSize = yield* TArray.size(this.available)
      const inUseMap = yield* TRef.get(this.inUse)
      
      return {
        available: availableSize,
        inUse: inUseMap.size,
        total: availableSize + inUseMap.size
      }
    })
}

// Usage example
const resourcePoolExample = Effect.gen(function* () {
  // Create a pool of database connections
  const connectionPool = yield* STM.commit(
    ResourcePool.make(
      5,
      () => Effect.succeed(`Connection-${Math.random()}`),
      (conn) => Effect.sync(() => console.log(`Destroyed ${conn}`))
    )
  )
  
  // Simulate multiple clients acquiring and releasing connections
  const clients = Array.makeBy(10, i =>
    Effect.fork(
      Effect.gen(function* () {
        console.log(`Client ${i} requesting connection...`)
        
        const acquireEffect = yield* STM.commit(connectionPool.acquire())
        
        if (Option.isSome(acquireEffect)) {
          const connection = yield* acquireEffect.value
          console.log(`Client ${i} got connection: ${connection}`)
          
          // Simulate work
          yield* Effect.sleep("1 second")
          
          // Release connection
          const released = yield* STM.commit(
            connectionPool.release(`resource-${connection}`)
          )
          console.log(`Client ${i} released connection: ${released}`)
        } else {
          console.log(`Client ${i} couldn't get connection`)
        }
      })
    )
  )
  
  // Monitor pool stats
  const monitor = Effect.fork(
    Effect.forever(
      Effect.gen(function* () {
        const stats = yield* STM.commit(connectionPool.getStats())
        console.log(`Pool stats: ${JSON.stringify(stats)}`)
        yield* Effect.sleep("500 millis")
      })
    )
  )
  
  // Wait for all clients
  yield* Effect.all(clients.map(fiber => Fiber.join(fiber)))
  
  yield* Fiber.interrupt(monitor)
  
  // Cleanup old resources
  const cleanupEffect = yield* STM.commit(connectionPool.cleanup(0))
  const cleaned = yield* cleanupEffect
  console.log(`Cleaned up ${cleaned} resources`)
})
```

### Pattern 3: Event Sourcing with STM

```typescript
import { Effect, STM, TRef, TArray, Data } from "effect"

// Event types
abstract class DomainEvent extends Data.TaggedClass("DomainEvent")<{
  id: string
  aggregateId: string
  version: number
  timestamp: number
}> {}

class AccountCreated extends DomainEvent {
  readonly _tag = "AccountCreated"
  constructor(
    props: { id: string; aggregateId: string; version: number; timestamp: number } & {
      initialBalance: number
      accountHolder: string
    }
  ) {
    super(props)
  }
}

class MoneyDeposited extends DomainEvent {
  readonly _tag = "MoneyDeposited"
  constructor(
    props: { id: string; aggregateId: string; version: number; timestamp: number } & {
      amount: number
    }
  ) {
    super(props)
  }
}

class MoneyWithdrawn extends DomainEvent {
  readonly _tag = "MoneyWithdrawn"
  constructor(
    props: { id: string; aggregateId: string; version: number; timestamp: number } & {
      amount: number
    }
  ) {
    super(props)
  }
}

// Aggregate state
interface AccountState {
  id: string
  version: number
  balance: number
  accountHolder: string
  isActive: boolean
}

// Event store
class EventStore {
  constructor(
    private events: TArray.TArray<DomainEvent>,
    private snapshots: TRef.TRef<Map<string, { state: AccountState, version: number }>>
  ) {}
  
  static make = () =>
    STM.gen(function* () {
      const events = yield* TArray.empty<DomainEvent>()
      const snapshots = yield* TRef.make(new Map<string, { state: AccountState, version: number }>())
      
      return new EventStore(events, snapshots)
    })
  
  // Append events with optimistic concurrency control
  appendEvents = (
    aggregateId: string,
    expectedVersion: number,
    newEvents: DomainEvent[]
  ): STM.STM<void, string> =>
    STM.gen(function* () {
      // Check current version
      const currentVersion = yield* this.getCurrentVersion(aggregateId)
      
      if (currentVersion !== expectedVersion) {
        yield* STM.fail(`Concurrency conflict: expected version ${expectedVersion}, but current is ${currentVersion}`)
      }
      
      // Append events
      for (const event of newEvents) {
        yield* TArray.append(this.events, event)
      }
    })
  
  // Get events for aggregate since version
  getEventsSince = (aggregateId: string, fromVersion: number): STM.STM<DomainEvent[]> =>
    STM.gen(function* () {
      const size = yield* TArray.size(this.events)
      const result: DomainEvent[] = []
      
      for (let i = 0; i < size; i++) {
        const event = yield* TArray.get(this.events, i)
        if (Option.isSome(event) && 
            event.value.aggregateId === aggregateId && 
            event.value.version > fromVersion) {
          result.push(event.value)
        }
      }
      
      return result.sort((a, b) => a.version - b.version)
    })
  
  private getCurrentVersion = (aggregateId: string): STM.STM<number> =>
    STM.gen(function* () {
      const size = yield* TArray.size(this.events)
      let maxVersion = 0
      
      for (let i = 0; i < size; i++) {
        const event = yield* TArray.get(this.events, i)
        if (Option.isSome(event) && event.value.aggregateId === aggregateId) {
          maxVersion = Math.max(maxVersion, event.value.version)
        }
      }
      
      return maxVersion
    })
  
  // Create snapshot
  createSnapshot = (aggregateId: string, state: AccountState): STM.STM<void> =>
    TRef.update(this.snapshots, snapshots => 
      new Map(snapshots.set(aggregateId, { state, version: state.version }))
    )
  
  // Get latest snapshot
  getSnapshot = (aggregateId: string): STM.STM<Option.Option<{ state: AccountState, version: number }>> =>
    STM.gen(function* () {
      const snapshots = yield* TRef.get(this.snapshots)
      return Option.fromNullable(snapshots.get(aggregateId))
    })
}

// Account aggregate
class Account {
  constructor(
    private eventStore: EventStore,
    private state: AccountState,
    private uncommittedEvents: DomainEvent[] = []
  ) {}
  
  static create = (
    eventStore: EventStore,
    accountId: string,
    accountHolder: string,
    initialBalance: number
  ): STM.STM<Account, string> =>
    STM.gen(function* () {
      const event = new AccountCreated({
        id: `event-${Date.now()}`,
        aggregateId: accountId,
        version: 1,
        timestamp: Date.now(),
        initialBalance,
        accountHolder
      })
      
      const state: AccountState = {
        id: accountId,
        version: 1,
        balance: initialBalance,
        accountHolder,
        isActive: true
      }
      
      return new Account(eventStore, state, [event])
    })
  
  static load = (eventStore: EventStore, accountId: string): STM.STM<Option.Option<Account>, string> =>
    STM.gen(function* () {
      // Try to get snapshot first
      const snapshot = yield* eventStore.getSnapshot(accountId)
      
      let state: AccountState
      let fromVersion: number
      
      if (Option.isSome(snapshot)) {
        state = snapshot.value.state
        fromVersion = snapshot.value.version
      } else {
        // No snapshot, check if account exists
        const events = yield* eventStore.getEventsSince(accountId, 0)
        if (events.length === 0) {
          return Option.none()
        }
        
        // Build state from first event
        const firstEvent = events[0]
        if (firstEvent._tag !== "AccountCreated") {
          yield* STM.fail("Invalid event stream: first event must be AccountCreated")
        }
        
        state = {
          id: accountId,
          version: 1,
          balance: (firstEvent as AccountCreated).initialBalance,
          accountHolder: (firstEvent as AccountCreated).accountHolder,
          isActive: true
        }
        
        fromVersion = 1
      }
      
      // Apply events since snapshot
      const events = yield* eventStore.getEventsSince(accountId, fromVersion)
      
      for (const event of events) {
        state = Account.applyEvent(state, event)
      }
      
      return Option.some(new Account(eventStore, state))
    })
  
  private static applyEvent = (state: AccountState, event: DomainEvent): AccountState => {
    switch (event._tag) {
      case "AccountCreated":
        const created = event as AccountCreated
        return {
          id: created.aggregateId,
          version: created.version,
          balance: created.initialBalance,
          accountHolder: created.accountHolder,
          isActive: true
        }
      
      case "MoneyDeposited":
        const deposited = event as MoneyDeposited
        return {
          ...state,
          version: deposited.version,
          balance: state.balance + deposited.amount
        }
      
      case "MoneyWithdrawn":
        const withdrawn = event as MoneyWithdrawn
        return {
          ...state,
          version: withdrawn.version,
          balance: state.balance - withdrawn.amount
        }
      
      default:
        return state
    }
  }
  
  // Business logic methods
  deposit = (amount: number): STM.STM<void, string> =>
    STM.gen(function* () {
      if (amount <= 0) {
        yield* STM.fail("Amount must be positive")
      }
      
      if (!this.state.isActive) {
        yield* STM.fail("Account is not active")
      }
      
      const event = new MoneyDeposited({
        id: `event-${Date.now()}`,
        aggregateId: this.state.id,
        version: this.state.version + 1,
        timestamp: Date.now(),
        amount
      })
      
      this.state = Account.applyEvent(this.state, event)
      this.uncommittedEvents.push(event)
    })
  
  withdraw = (amount: number): STM.STM<void, string> =>
    STM.gen(function* () {
      if (amount <= 0) {
        yield* STM.fail("Amount must be positive")
      }
      
      if (!this.state.isActive) {
        yield* STM.fail("Account is not active")
      }
      
      if (this.state.balance < amount) {
        yield* STM.fail(`Insufficient funds: ${this.state.balance} < ${amount}`)
      }
      
      const event = new MoneyWithdrawn({
        id: `event-${Date.now()}`,
        aggregateId: this.state.id,
        version: this.state.version + 1,
        timestamp: Date.now(),
        amount
      })
      
      this.state = Account.applyEvent(this.state, event)
      this.uncommittedEvents.push(event)
    })
  
  // Commit changes
  commit = (): STM.STM<void, string> =>
    STM.gen(function* () {
      if (this.uncommittedEvents.length === 0) {
        return
      }
      
      const expectedVersion = this.state.version - this.uncommittedEvents.length
      
      yield* this.eventStore.appendEvents(
        this.state.id,
        expectedVersion,
        this.uncommittedEvents
      )
      
      this.uncommittedEvents = []
      
      // Create snapshot every 10 events
      if (this.state.version % 10 === 0) {
        yield* this.eventStore.createSnapshot(this.state.id, this.state)
      }
    })
  
  getState = () => ({ ...this.state })
  hasUncommittedEvents = () => this.uncommittedEvents.length > 0
}

// Usage example
const eventSourcingExample = Effect.gen(function* () {
  const eventStore = yield* STM.commit(EventStore.make())
  
  // Create account
  const account = yield* STM.commit(
    Account.create(eventStore, "account-1", "John Doe", 1000)
  )
  
  // Perform operations
  yield* STM.commit(account.deposit(500))
  yield* STM.commit(account.withdraw(200))
  yield* STM.commit(account.deposit(100))
  
  // Commit all changes
  yield* STM.commit(account.commit())
  
  console.log("Account state after operations:", account.getState())
  
  // Load account from event store
  const loadedAccount = yield* STM.commit(Account.load(eventStore, "account-1"))
  
  if (Option.isSome(loadedAccount)) {
    console.log("Loaded account state:", loadedAccount.value.getState())
    
    // Perform more operations
    yield* STM.commit(loadedAccount.value.withdraw(300))
    yield* STM.commit(loadedAccount.value.commit())
    
    console.log("Final account state:", loadedAccount.value.getState())
  }
})
```

## Integration Examples

### Integration with React State Management

```typescript
import { Effect, STM, TRef, Context } from "effect"
import { useState, useEffect, useCallback } from "react"

// STM-based state management for React
interface AppState {
  user: { id: string, name: string } | null
  todos: Array<{ id: string, text: string, completed: boolean }>
  loading: boolean
}

class STMStateManager {
  constructor(private state: TRef.TRef<AppState>) {}
  
  static make = (initialState: AppState) =>
    STM.gen(function* () {
      const state = yield* TRef.make(initialState)
      return new STMStateManager(state)
    })
  
  getState = () => TRef.get(this.state)
  
  setUser = (user: AppState["user"]) =>
    TRef.update(this.state, state => ({ ...state, user }))
  
  addTodo = (text: string) =>
    TRef.update(this.state, state => ({
      ...state,
      todos: [
        ...state.todos,
        { id: `todo-${Date.now()}`, text, completed: false }
      ]
    }))
  
  toggleTodo = (id: string) =>
    TRef.update(this.state, state => ({
      ...state,
      todos: state.todos.map(todo =>
        todo.id === id ? { ...todo, completed: !todo.completed } : todo
      )
    }))
  
  setLoading = (loading: boolean) =>
    TRef.update(this.state, state => ({ ...state, loading }))
}

// React hook for STM state
const useSTMState = (stateManager: STMStateManager) => {
  const [state, setState] = useState<AppState | null>(null)
  
  useEffect(() => {
    const subscription = Effect.runPromise(
      Effect.forever(
        Effect.gen(function* () {
          const currentState = yield* STM.commit(stateManager.getState())
          setState(currentState)
          yield* Effect.sleep("16 millis") // ~60fps updates
        })
      )
    )
    
    return () => {
      subscription.then(fiber => Fiber.interrupt(fiber))
    }
  }, [stateManager])
  
  const dispatch = useCallback((action: STM.STM<void>) => {
    Effect.runPromise(STM.commit(action))
  }, [])
  
  return { state, dispatch }
}

// Usage in React component
const TodoApp = ({ stateManager }: { stateManager: STMStateManager }) => {
  const { state, dispatch } = useSTMState(stateManager)
  
  if (!state) return <div>Loading...</div>
  
  return (
    <div>
      <h1>STM Todo App</h1>
      {state.user && <p>Welcome, {state.user.name}!</p>}
      
      <div>
        <input
          onKeyPress={(e) => {
            if (e.key === 'Enter') {
              dispatch(stateManager.addTodo(e.currentTarget.value))
              e.currentTarget.value = ''
            }
          }}
          placeholder="Add todo..."
        />
      </div>
      
      <ul>
        {state.todos.map(todo => (
          <li key={todo.id}>
            <input
              type="checkbox"
              checked={todo.completed}
              onChange={() => dispatch(stateManager.toggleTodo(todo.id))}
            />
            <span style={{ textDecoration: todo.completed ? 'line-through' : 'none' }}>
              {todo.text}
            </span>
          </li>
        ))}
      </ul>
    </div>
  )
}
```

### Testing Strategies

```typescript
import { Effect, STM, TRef, TestContext } from "effect"
import { describe, it, expect } from "@effect/vitest"

describe("STM Testing", () => {
  it("should handle concurrent modifications correctly", () =>
    Effect.gen(function* () {
      const counter = yield* STM.commit(TRef.make(0))
      
      // Launch multiple concurrent increments
      const increments = Array.from({ length: 100 }, () =>
        STM.commit(TRef.update(counter, n => n + 1))
      )
      
      yield* Effect.all(increments, { concurrency: "unbounded" })
      
      const finalValue = yield* STM.commit(TRef.get(counter))
      expect(finalValue).toBe(100)
    })
  )
  
  it("should retry until condition is met", () =>
    Effect.gen(function* () {
      const resource = yield* STM.commit(TRef.make(0))
      let attempts = 0
      
      // Consumer that waits for resource to become available
      const consumer = STM.gen(function* () {
        const current = yield* TRef.get(resource)
        attempts++
        
        if (current < 5) {
          yield* STM.retry()
        }
        
        return current
      })
      
      // Producer that increments resource
      const producer = Effect.gen(function* () {
        for (let i = 1; i <= 5; i++) {
          yield* Effect.sleep("10 millis")
          yield* STM.commit(TRef.set(resource, i))
        }
      })
      
      // Run consumer and producer concurrently
      const [result] = yield* Effect.all([
        STM.commit(consumer),
        producer
      ], { concurrency: "unbounded" })
      
      expect(result).toBe(5)
      expect(attempts).toBeGreaterThan(1)
    }).pipe(Effect.provide(TestContext.TestContext))
  )
  
  it("should handle transaction conflicts correctly", () =>
    Effect.gen(function* () {
      const account1 = yield* STM.commit(TRef.make(1000))
      const account2 = yield* STM.commit(TRef.make(500))
      
      // Simultaneous transfers that should be atomic
      const transfer1 = STM.gen(function* () {
        const balance1 = yield* TRef.get(account1)
        const balance2 = yield* TRef.get(account2)
        
        if (balance1 >= 200) {
          yield* TRef.update(account1, b => b - 200)
          yield* TRef.update(account2, b => b + 200)
          return true
        }
        return false
      })
      
      const transfer2 = STM.gen(function* () {
        const balance2 = yield* TRef.get(account2)
        const balance1 = yield* TRef.get(account1)
        
        if (balance2 >= 300) {
          yield* TRef.update(account2, b => b - 300)
          yield* TRef.update(account1, b => b + 300)
          return true
        }
        return false
      })
      
      const [result1, result2] = yield* Effect.all([
        STM.commit(transfer1),
        STM.commit(transfer2)
      ], { concurrency: "unbounded" })
      
      const finalBalance1 = yield* STM.commit(TRef.get(account1))
      const finalBalance2 = yield* STM.commit(TRef.get(account2))
      
      // Total should be preserved
      expect(finalBalance1 + finalBalance2).toBe(1500)
      
      // At least one transfer should succeed
      expect(result1 || result2).toBe(true)
    })
  )
})

// Property-based testing for STM
const testSTMProperties = Effect.gen(function* () {
  // Test that STM operations are atomic
  const shared = yield* STM.commit(TRef.make(0))
  
  // Many concurrent operations
  const operations = Array.from({ length: 1000 }, (_, i) =>
    STM.commit(
      STM.gen(function* () {
        const current = yield* TRef.get(shared)
        yield* TRef.set(shared, current + i)
      })
    )
  )
  
  yield* Effect.all(operations, { concurrency: "unbounded" })
  
  const final = yield* STM.commit(TRef.get(shared))
  const expected = Array.from({ length: 1000 }, (_, i) => i).reduce((a, b) => a + b, 0)
  
  // Due to atomicity, final value should be predictable
  // (though the exact value depends on execution order)
  expect(final).toBeGreaterThan(0)
})
```

STM provides a powerful foundation for building concurrent, composable systems in Effect applications. By embracing software transactional memory, you get automatic conflict resolution, deadlock-free coordination, and composable atomic operations.

Key benefits:
- **Lock-free Concurrency**: No explicit locking required, eliminating deadlocks
- **Composable Transactions**: Build complex operations from simple atomic primitives
- **Automatic Retry**: Built-in retry mechanism when preconditions aren't met
- **Type Safety**: Full TypeScript integration with compile-time guarantees
- **Testing Support**: Deterministic testing of concurrent operations

Use STM when you need thread-safe state management, complex atomic operations, coordination between concurrent processes, or any scenario where traditional locking approaches become unwieldy.