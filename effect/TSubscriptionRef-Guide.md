# TSubscriptionRef: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem TSubscriptionRef Solves

Building reactive systems that require transactional consistency across multiple shared states is extremely challenging. Traditional approaches using either pure STM or basic reactive patterns fail to provide both atomicity guarantees and reactive subscription capabilities:

```typescript
// Traditional approach - mixing STM with manual subscription management
class TransactionalStateManager<T> {
  private tref: TRef<T>
  private subscribers: Set<(value: T) => void> = new Set()
  
  constructor(initial: T) {
    this.tref = TRef.make(initial)
  }
  
  async updateTransactionally(updates: Array<(state: T) => T>): Promise<void> {
    // Atomic updates work fine
    const transaction = STM.gen(function* () {
      let current = yield* TRef.get(this.tref)
      for (const update of updates) {
        current = update(current)
      }
      yield* TRef.set(this.tref, current)
      return current
    })
    
    const newValue = await Effect.runPromise(STM.commit(transaction))
    
    // But notifications are outside the transaction!
    // Race conditions and consistency issues
    this.subscribers.forEach(callback => {
      try {
        callback(newValue) // Not atomic with the transaction
      } catch (error) {
        console.error('Notification failed:', error)
      }
    })
  }
  
  subscribe(callback: (value: T) => void): () => void {
    this.subscribers.add(callback)
    // Manual cleanup - memory leaks possible
    return () => this.subscribers.delete(callback)
  }
}

// Complex coordination scenarios
class OrderBookManager {
  private orderBook: TransactionalStateManager<OrderBook>
  private accountBalances: Map<string, TransactionalStateManager<number>>
  
  async executeOrder(order: Order): Promise<boolean> {
    // Need to coordinate multiple transactional states
    // AND notify all subscribers consistently
    // This becomes a nightmare with manual coordination
    
    try {
      // Multiple separate transactions - not atomic together!
      await this.updateAccountBalance(order.buyerId, -order.price)
      await this.updateAccountBalance(order.sellerId, order.price)
      await this.orderBook.updateTransactionally([
        book => ({ ...book, orders: [...book.orders, order] })
      ])
      
      // Notifications happen at different times
      // Observers see inconsistent intermediate states!
      return true
    } catch (error) {
      // Rollback is manual and error-prone
      // Partial failures leave inconsistent state
      return false
    }
  }
}
```

This approach leads to:
- **Broken Atomicity** - Transactional updates and notifications happen separately
- **Race Conditions** - Observers can see intermediate inconsistent states
- **Manual Coordination** - Complex synchronization between STM transactions and subscriptions
- **Memory Leaks** - Manual subscription management without proper cleanup
- **Error Handling Complexity** - Failed notifications can leave system in inconsistent state

### The TSubscriptionRef Solution

TSubscriptionRef combines STM's transactional guarantees with reactive subscription capabilities, providing atomic updates with guaranteed consistent notifications:

```typescript
import { TSubscriptionRef, STM, Effect, Stream } from "effect"

// The Effect solution - atomic updates with consistent reactive notifications
const createReactiveOrderBook = STM.gen(function* () {
  const orderBookRef = yield* TSubscriptionRef.make<OrderBook>({
    orders: [],
    lastUpdated: Date.now()
  })
  
  const buyerBalanceRef = yield* TSubscriptionRef.make(1000)
  const sellerBalanceRef = yield* TSubscriptionRef.make(0)
  
  return { orderBookRef, buyerBalanceRef, sellerBalanceRef }
})

// Atomic multi-state updates with consistent notifications
const executeOrderAtomically = (
  refs: { orderBookRef: TSubscriptionRef<OrderBook>, buyerBalanceRef: TSubscriptionRef<number>, sellerBalanceRef: TSubscriptionRef<number> },
  order: Order
) => STM.gen(function* () {
  const buyerBalance = yield* TSubscriptionRef.get(refs.buyerBalanceRef)
  const orderBook = yield* TSubscriptionRef.get(refs.orderBookRef)
  
  // All updates happen atomically - either all succeed or all fail
  if (buyerBalance >= order.price) {
    yield* TSubscriptionRef.update(refs.buyerBalanceRef, balance => balance - order.price)
    yield* TSubscriptionRef.update(refs.sellerBalanceRef, balance => balance + order.price)
    yield* TSubscriptionRef.update(refs.orderBookRef, book => ({
      orders: [...book.orders, order],
      lastUpdated: Date.now()
    }))
    return true
  }
  
  return false
})

// All subscribers get notified consistently after the transaction commits
const program = Effect.gen(function* () {
  const refs = yield* STM.commit(createReactiveOrderBook)
  
  // Multiple reactive observers - all see consistent state
  const orderBookObserver = TSubscriptionRef.changesStream(refs.orderBookRef).pipe(
    Stream.runForEach(book => Effect.log(`Order book updated: ${book.orders.length} orders`))
  )
  
  const balanceObserver = TSubscriptionRef.changesStream(refs.buyerBalanceRef).pipe(
    Stream.runForEach(balance => Effect.log(`Buyer balance: $${balance}`))
  )
  
  // Start observers
  yield* Effect.fork(orderBookObserver)
  yield* Effect.fork(balanceObserver)
  
  // Execute atomic transaction - all observers notified consistently
  const success = yield* STM.commit(executeOrderAtomically(refs, {
    id: '1',
    buyerId: 'buyer1',
    sellerId: 'seller1',
    price: 100
  }))
  
  yield* Effect.log(`Order executed: ${success}`)
})
```

### Key Concepts

**TSubscriptionRef<A>**: A transactional reference that combines TRef's atomic updates with subscription capabilities, ensuring that all change notifications happen consistently after transaction commits.

**STM Integration**: All TSubscriptionRef operations are STM computations, allowing them to participate in larger transactional compositions with other STM data structures.

**Consistent Notifications**: Unlike manual approaches, all subscribers are notified atomically after the transaction commits, ensuring they never see intermediate inconsistent states.

**Automatic Resource Management**: Subscriptions are managed through Effect's resource system, preventing memory leaks and ensuring proper cleanup.

## Basic Usage Patterns

### Pattern 1: Creating and Basic Operations

```typescript
import { TSubscriptionRef, STM, Effect } from "effect"

// Create a TSubscriptionRef with initial value
const createUserSession = STM.gen(function* () {
  const sessionRef = yield* TSubscriptionRef.make({
    userId: 'user123',
    isAuthenticated: false,
    lastActivity: Date.now(),
    permissions: []
  })
  
  return sessionRef
})

// Basic STM operations
const basicOperations = Effect.gen(function* () {
  const sessionRef = yield* STM.commit(createUserSession)
  
  // Get current value (STM operation)
  const currentSession = yield* STM.commit(TSubscriptionRef.get(sessionRef))
  yield* Effect.log(`Current session: ${JSON.stringify(currentSession)}`)
  
  // Set new value (STM operation)
  const loginTransaction = STM.gen(function* () {
    yield* TSubscriptionRef.set(sessionRef, {
      userId: 'user123',
      isAuthenticated: true,
      lastActivity: Date.now(),
      permissions: ['read', 'write']
    })
  })
  
  yield* STM.commit(loginTransaction)
  
  // Update with function (STM operation)
  const updateActivityTransaction = STM.gen(function* () {
    yield* TSubscriptionRef.update(sessionRef, session => ({
      ...session,
      lastActivity: Date.now()
    }))
  })
  
  yield* STM.commit(updateActivityTransaction)
})
```

### Pattern 2: Subscribing to Transactional Changes

```typescript
import { Stream } from "effect"

// Subscribe to all transactional changes
const subscribeToSessionChanges = Effect.gen(function* () {
  const sessionRef = yield* STM.commit(createUserSession)
  
  // Stream includes current value + all future transactional changes
  const sessionStream = TSubscriptionRef.changesStream(sessionRef)
  
  const subscription = sessionStream.pipe(
    Stream.runForEach(session => 
      Effect.log(`Session updated: ${session.isAuthenticated ? 'Authenticated' : 'Anonymous'}`)
    )
  )
  
  // Run subscription in background
  yield* Effect.fork(subscription)
  
  // Make transactional changes
  const authTransaction = STM.gen(function* () {
    yield* TSubscriptionRef.update(sessionRef, s => ({ ...s, isAuthenticated: true }))
  })
  
  yield* STM.commit(authTransaction)
})
```

### Pattern 3: Scoped Subscriptions

```typescript
// Scoped subscription management for precise resource control
const scopedSessionMonitoring = Effect.gen(function* () {
  const sessionRef = yield* STM.commit(createUserSession)
  
  // Create scoped subscription
  const monitoringLogic = Effect.gen(function* () {
    const changesQueue = yield* TSubscriptionRef.changesScoped(sessionRef)
    
    // Process changes from the queue
    const processChanges = Effect.gen(function* () {
      while (true) {
        const session = yield* STM.commit(TQueue.take(changesQueue))
        yield* Effect.log(`Processing session change: ${session.userId}`)
        
        // Custom processing logic here
        if (!session.isAuthenticated) {
          yield* Effect.log("User logged out - cleaning up resources")
        }
      }
    })
    
    yield* processChanges
  })
  
  // Run with automatic cleanup when scope closes
  yield* Effect.scoped(monitoringLogic)
})
```

## Real-World Examples

### Example 1: Multi-User Gaming Session State

Building a gaming session manager that maintains consistent state across multiple players with real-time updates:

```typescript
import { TSubscriptionRef, TRef, STM, Effect, Stream } from "effect"

interface GameSession {
  readonly id: string
  readonly players: ReadonlyArray<Player>
  readonly gameState: GameState
  readonly currentTurn: string
  readonly lastMove: Move | null
}

interface Player {
  readonly id: string
  readonly name: string
  readonly score: number
  readonly isOnline: boolean
}

interface GameState {
  readonly status: 'waiting' | 'playing' | 'finished'
  readonly board: ReadonlyArray<ReadonlyArray<string>>
  readonly moveCount: number
}

interface Move {
  readonly playerId: string
  readonly position: [number, number]
  readonly timestamp: number
}

// Create a transactional gaming session
const createGameSession = (sessionId: string) => STM.gen(function* () {
  const sessionRef = yield* TSubscriptionRef.make<GameSession>({
    id: sessionId,
    players: [],
    gameState: {
      status: 'waiting',
      board: Array(3).fill(null).map(() => Array(3).fill('')),
      moveCount: 0
    },
    currentTurn: '',
    lastMove: null
  })
  
  // Player connection tracking
  const connectionsRef = yield* TRef.make<ReadonlyArray<string>>([])
  
  return { sessionRef, connectionsRef }
})

// Atomic player operations
const addPlayerToSession = (
  refs: { sessionRef: TSubscriptionRef<GameSession>, connectionsRef: TRef<ReadonlyArray<string>> },
  player: Player
) => STM.gen(function* () {
  const currentSession = yield* TSubscriptionRef.get(refs.sessionRef)
  const connections = yield* TRef.get(refs.connectionsRef)
  
  // Atomic updates - either both succeed or both fail
  if (currentSession.players.length < 2 && !connections.includes(player.id)) {
    const updatedPlayers = [...currentSession.players, player]
    const gameState = updatedPlayers.length === 2 
      ? { ...currentSession.gameState, status: 'playing' as const }
      : currentSession.gameState
    
    yield* TSubscriptionRef.update(refs.sessionRef, session => ({
      ...session,
      players: updatedPlayers,
      gameState,
      currentTurn: updatedPlayers.length === 2 ? updatedPlayers[0].id : session.currentTurn
    }))
    
    yield* TRef.update(refs.connectionsRef, conns => [...conns, player.id])
    return true
  }
  
  return false
})

// Atomic move execution with validation
const makeMove = (
  sessionRef: TSubscriptionRef<GameSession>,
  move: Move
) => STM.gen(function* () {
  const session = yield* TSubscriptionRef.get(sessionRef)
  
  // Validate move atomically
  if (session.gameState.status !== 'playing' || 
      session.currentTurn !== move.playerId ||
      session.gameState.board[move.position[0]][move.position[1]] !== '') {
    return false
  }
  
  // Execute move atomically
  const newBoard = session.gameState.board.map((row, i) =>
    row.map((cell, j) => 
      i === move.position[0] && j === move.position[1] 
        ? move.playerId === session.players[0].id ? 'X' : 'O'
        : cell
    )
  )
  
  const nextPlayer = session.players.find(p => p.id !== move.playerId)
  const moveCount = session.gameState.moveCount + 1
  
  // Check for win condition
  const hasWon = checkWinCondition(newBoard, move.position)
  const gameStatus = hasWon || moveCount >= 9 ? 'finished' : 'playing'
  
  yield* TSubscriptionRef.update(sessionRef, s => ({
    ...s,
    gameState: {
      ...s.gameState,
      board: newBoard,
      moveCount,
      status: gameStatus
    },
    currentTurn: gameStatus === 'playing' ? nextPlayer?.id || '' : '',
    lastMove: move
  }))
  
  return true
})

// Real-time game monitoring
const createGameMonitor = (sessionRef: TSubscriptionRef<GameSession>) => {
  const gameUpdates = TSubscriptionRef.changesStream(sessionRef).pipe(
    Stream.runForEach(session => Effect.gen(function* () {
      yield* Effect.log(`Game ${session.id}: ${session.players.length}/2 players`)
      
      if (session.lastMove) {
        const player = session.players.find(p => p.id === session.lastMove!.playerId)
        yield* Effect.log(`${player?.name} made a move at ${session.lastMove.position}`)
      }
      
      if (session.gameState.status === 'finished') {
        yield* Effect.log(`Game ${session.id} finished after ${session.gameState.moveCount} moves`)
      }
    }))
  )
  
  return gameUpdates
}

// Complete game session program
const gameSessionProgram = Effect.gen(function* () {
  const refs = yield* STM.commit(createGameSession('game-123'))
  
  // Start monitoring
  yield* Effect.fork(createGameMonitor(refs.sessionRef))
  
  // Players join
  const player1: Player = { id: 'p1', name: 'Alice', score: 0, isOnline: true }
  const player2: Player = { id: 'p2', name: 'Bob', score: 0, isOnline: true }
  
  const addPlayer1 = yield* STM.commit(addPlayerToSession(refs, player1))
  const addPlayer2 = yield* STM.commit(addPlayerToSession(refs, player2))
  
  yield* Effect.log(`Players added: ${addPlayer1}, ${addPlayer2}`)
  
  // Simulate game moves
  const moves: Move[] = [
    { playerId: 'p1', position: [0, 0], timestamp: Date.now() },
    { playerId: 'p2', position: [0, 1], timestamp: Date.now() + 1000 },
    { playerId: 'p1', position: [1, 1], timestamp: Date.now() + 2000 },
    { playerId: 'p2', position: [0, 2], timestamp: Date.now() + 3000 },
    { playerId: 'p1', position: [2, 2], timestamp: Date.now() + 4000 }
  ]
  
  for (const move of moves) {
    const success = yield* STM.commit(makeMove(refs.sessionRef, move))
    yield* Effect.log(`Move by ${move.playerId}: ${success}`)
    yield* Effect.sleep(500)
  }
})

// Helper function for win condition
const checkWinCondition = (board: ReadonlyArray<ReadonlyArray<string>>, lastMove: [number, number]): boolean => {
  const [row, col] = lastMove
  const player = board[row][col]
  
  // Check row, column, and diagonals
  const checkLine = (positions: Array<[number, number]>) =>
    positions.every(([r, c]) => board[r][c] === player)
  
  return checkLine([[row, 0], [row, 1], [row, 2]]) || // Row
         checkLine([[0, col], [1, col], [2, col]]) || // Column
         checkLine([[0, 0], [1, 1], [2, 2]]) || // Main diagonal
         checkLine([[0, 2], [1, 1], [2, 0]])   // Anti-diagonal
}
```

### Example 2: Distributed Cache with Invalidation

Building a cache system that maintains consistency across multiple cache levels with atomic invalidation:

```typescript
import { TSubscriptionRef, TMap, STM, Effect, Stream, Schedule } from "effect"

interface CacheEntry<T> {
  readonly value: T
  readonly timestamp: number
  readonly ttl: number
  readonly accessCount: number
}

interface CacheStats {
  readonly hits: number
  readonly misses: number
  readonly evictions: number
  readonly totalSize: number
}

// Multi-level cache with consistent invalidation
const createDistributedCache = <K, V>() => STM.gen(function* () {
  // L1 Cache (hot data)
  const l1Cache = yield* TMap.empty<K, CacheEntry<V>>()
  
  // L2 Cache (warm data)
  const l2Cache = yield* TMap.empty<K, CacheEntry<V>>()
  
  // Cache statistics
  const statsRef = yield* TSubscriptionRef.make<CacheStats>({
    hits: 0,
    misses: 0,
    evictions: 0,
    totalSize: 0
  })
  
  // Invalidation tracking
  const invalidationRef = yield* TSubscriptionRef.make<ReadonlyArray<K>>([])
  
  return { l1Cache, l2Cache, statsRef, invalidationRef }
})

// Atomic cache operations
const cacheGet = <K, V>(
  cache: { l1Cache: TMap<K, CacheEntry<V>>, l2Cache: TMap<K, CacheEntry<V>>, statsRef: TSubscriptionRef<CacheStats> },
  key: K
) => STM.gen(function* () {
  const now = Date.now()
  
  // Try L1 first
  const l1Entry = yield* TMap.get(cache.l1Cache, key)
  if (l1Entry && now - l1Entry.timestamp < l1Entry.ttl) {
    // L1 hit - update access count and stats atomically
    const updatedEntry = { ...l1Entry, accessCount: l1Entry.accessCount + 1 }
    yield* TMap.set(cache.l1Cache, key, updatedEntry)
    yield* TSubscriptionRef.update(cache.statsRef, stats => ({ ...stats, hits: stats.hits + 1 }))
    return l1Entry.value
  }
  
  // Try L2
  const l2Entry = yield* TMap.get(cache.l2Cache, key)
  if (l2Entry && now - l2Entry.timestamp < l2Entry.ttl) {
    // L2 hit - promote to L1 and update stats atomically
    const promotedEntry = { ...l2Entry, accessCount: l2Entry.accessCount + 1 }
    yield* TMap.set(cache.l1Cache, key, promotedEntry)
    yield* TSubscriptionRef.update(cache.statsRef, stats => ({ ...stats, hits: stats.hits + 1 }))
    return l2Entry.value
  }
  
  // Cache miss
  yield* TSubscriptionRef.update(cache.statsRef, stats => ({ ...stats, misses: stats.misses + 1 }))
  return null
})

// Atomic cache set with eviction policy
const cacheSet = <K, V>(
  cache: { l1Cache: TMap<K, CacheEntry<V>>, l2Cache: TMap<K, CacheEntry<V>>, statsRef: TSubscriptionRef<CacheStats> },
  key: K,
  value: V,
  ttl: number = 300000 // 5 minutes default
) => STM.gen(function* () {
  const now = Date.now()
  const entry: CacheEntry<V> = {
    value,
    timestamp: now,
    ttl,
    accessCount: 1
  }
  
  const l1Size = yield* TMap.size(cache.l1Cache)
  
  if (l1Size >= 100) { // L1 cache limit
    // Evict least accessed item from L1 to L2
    const l1Keys = yield* TMap.keys(cache.l1Cache)
    const entries = yield* STM.forEach(l1Keys, key => 
      STM.gen(function* () {
        const entry = yield* TMap.get(cache.l1Cache, key)
        return entry ? [key, entry] as const : null
      })
    )
    
    const validEntries = entries.filter((e): e is [K, CacheEntry<V>] => e !== null)
    if (validEntries.length > 0) {
      // Find least accessed entry
      const leastAccessed = validEntries.reduce((min, current) => 
        current[1].accessCount < min[1].accessCount ? current : min
      )
      
      // Atomic eviction: remove from L1, add to L2
      yield* TMap.remove(cache.l1Cache, leastAccessed[0])
      yield* TMap.set(cache.l2Cache, leastAccessed[0], leastAccessed[1])
      yield* TSubscriptionRef.update(cache.statsRef, stats => ({ 
        ...stats, 
        evictions: stats.evictions + 1 
      }))
    }
  }
  
  // Add new entry to L1
  yield* TMap.set(cache.l1Cache, key, entry)
  yield* TSubscriptionRef.update(cache.statsRef, stats => ({ 
    ...stats, 
    totalSize: stats.totalSize + 1 
  }))
})

// Atomic cache invalidation
const invalidateKeys = <K, V>(
  cache: { l1Cache: TMap<K, CacheEntry<V>>, l2Cache: TMap<K, CacheEntry<V>>, invalidationRef: TSubscriptionRef<ReadonlyArray<K>> },
  keys: ReadonlyArray<K>
) => STM.gen(function* () {
  // Remove from both cache levels atomically
  yield* STM.forEach(keys, key => STM.gen(function* () {
    yield* TMap.remove(cache.l1Cache, key)
    yield* TMap.remove(cache.l2Cache, key)
  }))
  
  // Track invalidation
  yield* TSubscriptionRef.update(cache.invalidationRef, current => [...current, ...keys])
})

// Cache monitoring and metrics
const createCacheMonitor = <K, V>(
  cache: { statsRef: TSubscriptionRef<CacheStats>, invalidationRef: TSubscriptionRef<ReadonlyArray<K>> }
) => {
  const statsMonitor = TSubscriptionRef.changesStream(cache.statsRef).pipe(
    Stream.runForEach(stats => Effect.gen(function* () {
      const hitRate = stats.hits + stats.misses > 0 
        ? (stats.hits / (stats.hits + stats.misses) * 100).toFixed(2)
        : '0.00'
      
      yield* Effect.log(`Cache Stats - Hit Rate: ${hitRate}%, Total Size: ${stats.totalSize}, Evictions: ${stats.evictions}`)
    }))
  )
  
  const invalidationMonitor = TSubscriptionRef.changesStream(cache.invalidationRef).pipe(
    Stream.filter(keys => keys.length > 0),
    Stream.runForEach(keys => Effect.log(`Cache invalidated ${keys.length} keys`))
  )
  
  return Effect.gen(function* () {
    yield* Effect.fork(statsMonitor)
    yield* Effect.fork(invalidationMonitor)
  })
}

// Complete cache system program
const cacheSystemProgram = Effect.gen(function* () {
  const cache = yield* STM.commit(createDistributedCache<string, string>())
  
  // Start monitoring
  yield* createCacheMonitor(cache)
  
  // Simulate cache operations
  yield* STM.commit(cacheSet(cache, 'user:123', 'John Doe'))
  yield* STM.commit(cacheSet(cache, 'product:456', 'Widget'))
  
  const user = yield* STM.commit(cacheGet(cache, 'user:123'))
  yield* Effect.log(`Retrieved user: ${user}`)
  
  const missing = yield* STM.commit(cacheGet(cache, 'missing:key'))
  yield* Effect.log(`Missing key result: ${missing}`)
  
  // Simulate invalidation
  yield* STM.commit(invalidateKeys(cache, ['user:123']))
  
  const invalidatedUser = yield* STM.commit(cacheGet(cache, 'user:123'))
  yield* Effect.log(`After invalidation: ${invalidatedUser}`)
})
```

### Example 3: Real-Time Trading Engine

Building a trading engine that maintains consistent state across orders, positions, and market data:

```typescript
import { TSubscriptionRef, TMap, TQueue, STM, Effect, Stream } from "effect"

interface Order {
  readonly id: string
  readonly symbol: string
  readonly side: 'buy' | 'sell'
  readonly quantity: number
  readonly price: number
  readonly timestamp: number
  readonly status: 'pending' | 'filled' | 'cancelled'
}

interface Position {
  readonly symbol: string
  readonly quantity: number
  readonly averagePrice: number
  readonly unrealizedPnL: number
}

interface MarketData {
  readonly symbol: string
  readonly price: number
  readonly timestamp: number
  readonly volume: number
}

interface TradingAccount {
  readonly accountId: string
  readonly balance: number
  readonly positions: ReadonlyArray<Position>
  readonly orders: ReadonlyArray<Order>
}

// Create trading engine with consistent state management
const createTradingEngine = STM.gen(function* () {
  // Account state
  const accountRef = yield* TSubscriptionRef.make<TradingAccount>({
    accountId: 'acc-001',
    balance: 100000,
    positions: [],
    orders: []
  })
  
  // Market data
  const marketDataRef = yield* TSubscriptionRef.make<Map<string, MarketData>>(new Map())
  
  // Order book
  const orderBookRef = yield* TMap.empty<string, ReadonlyArray<Order>>() // symbol -> orders
  
  // Trade execution queue
  const executionQueueRef = yield* TQueue.unbounded<Order>()
  
  return { accountRef, marketDataRef, orderBookRef, executionQueueRef }
})

// Atomic order placement with validation
const placeOrder = (
  engine: { 
    accountRef: TSubscriptionRef<TradingAccount>, 
    orderBookRef: TMap<string, ReadonlyArray<Order>>,
    executionQueueRef: TQueue<Order>
  },
  order: Order
) => STM.gen(function* () {
  const account = yield* TSubscriptionRef.get(engine.accountRef)
  
  // Validate order atomically
  const requiredMargin = order.side === 'buy' ? order.quantity * order.price : 0
  if (account.balance < requiredMargin) {
    return { success: false, reason: 'Insufficient balance' }
  }
  
  // Add order to account and order book atomically
  const updatedAccount = {
    ...account,
    orders: [...account.orders, order],
    balance: account.balance - requiredMargin
  }
  
  const existingOrders = yield* TMap.get(engine.orderBookRef, order.symbol)
  const updatedOrders = [...(existingOrders || []), order]
  
  yield* TSubscriptionRef.set(engine.accountRef, updatedAccount)
  yield* TMap.set(engine.orderBookRef, order.symbol, updatedOrders)
  yield* TQueue.offer(engine.executionQueueRef, order)
  
  return { success: true, reason: 'Order placed successfully' }
})

// Atomic trade execution
const executeTrade = (
  engine: { 
    accountRef: TSubscriptionRef<TradingAccount>, 
    orderBookRef: TMap<string, ReadonlyArray<Order>>,
    marketDataRef: TSubscriptionRef<Map<string, MarketData>>
  },
  order: Order,
  executionPrice: number
) => STM.gen(function* () {
  const account = yield* TSubscriptionRef.get(engine.accountRef)
  const marketData = yield* TSubscriptionRef.get(engine.marketDataRef)
  
  // Find and update the order
  const updatedOrders = account.orders.map(o => 
    o.id === order.id ? { ...o, status: 'filled' as const } : o
  )
  
  // Update position
  const existingPosition = account.positions.find(p => p.symbol === order.symbol)
  let updatedPositions: ReadonlyArray<Position>
  
  if (existingPosition) {
    // Update existing position
    const newQuantity = order.side === 'buy' 
      ? existingPosition.quantity + order.quantity
      : existingPosition.quantity - order.quantity
    
    const newAveragePrice = newQuantity !== 0
      ? ((existingPosition.averagePrice * existingPosition.quantity) + 
         (executionPrice * order.quantity * (order.side === 'buy' ? 1 : -1))) / newQuantity
      : 0
    
    const currentPrice = marketData.get(order.symbol)?.price || executionPrice
    const unrealizedPnL = (currentPrice - newAveragePrice) * newQuantity
    
    const updatedPosition = {
      symbol: order.symbol,
      quantity: newQuantity,
      averagePrice: newAveragePrice,
      unrealizedPnL
    }
    
    updatedPositions = newQuantity === 0
      ? account.positions.filter(p => p.symbol !== order.symbol)
      : account.positions.map(p => p.symbol === order.symbol ? updatedPosition : p)
  } else {
    // Create new position
    const newPosition: Position = {
      symbol: order.symbol,
      quantity: order.side === 'buy' ? order.quantity : -order.quantity,
      averagePrice: executionPrice,
      unrealizedPnL: 0
    }
    
    updatedPositions = [...account.positions, newPosition]
  }
  
  // Calculate new balance
  const tradePnL = order.side === 'buy' 
    ? (order.price - executionPrice) * order.quantity  // Saved money on better price
    : (executionPrice - order.price) * order.quantity  // Made money on better price
  
  const newBalance = account.balance + tradePnL + (order.side === 'sell' ? executionPrice * order.quantity : 0)
  
  // Atomic update of all state
  const updatedAccount = {
    ...account,
    balance: newBalance,
    positions: updatedPositions,
    orders: updatedOrders
  }
  
  yield* TSubscriptionRef.set(engine.accountRef, updatedAccount)
  
  // Remove from order book
  const orderBookOrders = yield* TMap.get(engine.orderBookRef, order.symbol)
  if (orderBookOrders) {
    const filteredOrders = orderBookOrders.filter(o => o.id !== order.id)
    yield* TMap.set(engine.orderBookRef, order.symbol, filteredOrders)
  }
  
  return { success: true, executionPrice, tradePnL }
})

// Real-time market data updates
const updateMarketData = (
  marketDataRef: TSubscriptionRef<Map<string, MarketData>>,
  data: MarketData
) => STM.gen(function* () {
  const currentData = yield* TSubscriptionRef.get(marketDataRef)
  const updatedData = new Map(currentData)
  updatedData.set(data.symbol, data)
  
  yield* TSubscriptionRef.set(marketDataRef, updatedData)
})

// Trading engine monitoring
const createTradingMonitor = (
  engine: { 
    accountRef: TSubscriptionRef<TradingAccount>, 
    marketDataRef: TSubscriptionRef<Map<string, MarketData>>
  }
) => {
  const accountMonitor = TSubscriptionRef.changesStream(engine.accountRef).pipe(
    Stream.runForEach(account => Effect.gen(function* () {
      const totalPnL = account.positions.reduce((sum, pos) => sum + pos.unrealizedPnL, 0)
      yield* Effect.log(`Account Balance: $${account.balance.toFixed(2)}, Unrealized PnL: $${totalPnL.toFixed(2)}`)
      yield* Effect.log(`Active Orders: ${account.orders.filter(o => o.status === 'pending').length}`)
      yield* Effect.log(`Positions: ${account.positions.length}`)
    }))
  )
  
  const marketDataMonitor = TSubscriptionRef.changesStream(engine.marketDataRef).pipe(
    Stream.runForEach(dataMap => Effect.gen(function* () {
      const symbols = Array.from(dataMap.keys()).slice(0, 3) // Show first 3 symbols
      for (const symbol of symbols) {
        const data = dataMap.get(symbol)
        if (data) {
          yield* Effect.log(`${symbol}: $${data.price.toFixed(2)} (Vol: ${data.volume})`)
        }
      }
    }))
  )
  
  return Effect.gen(function* () {
    yield* Effect.fork(accountMonitor)
    yield* Effect.fork(marketDataMonitor)
  })
}

// Complete trading system program
const tradingSystemProgram = Effect.gen(function* () {
  const engine = yield* STM.commit(createTradingEngine)
  
  // Start monitoring
  yield* createTradingMonitor(engine)
  
  // Simulate market data
  const marketDataUpdates = Effect.gen(function* () {
    const symbols = ['AAPL', 'GOOGL', 'MSFT']
    for (let i = 0; i < 10; i++) {
      for (const symbol of symbols) {
        const data: MarketData = {
          symbol,
          price: 100 + Math.random() * 50,
          timestamp: Date.now(),
          volume: Math.floor(Math.random() * 10000)
        }
        yield* STM.commit(updateMarketData(engine.marketDataRef, data))
      }
      yield* Effect.sleep(1000)
    }
  })
  
  yield* Effect.fork(marketDataUpdates)
  
  // Place some orders
  const orders: Order[] = [
    {
      id: 'order-1',
      symbol: 'AAPL',
      side: 'buy',
      quantity: 100,
      price: 120,
      timestamp: Date.now(),
      status: 'pending'
    },
    {
      id: 'order-2', 
      symbol: 'GOOGL',
      side: 'buy',
      quantity: 50,
      price: 150,
      timestamp: Date.now(),
      status: 'pending'
    }
  ]
  
  for (const order of orders) {
    const result = yield* STM.commit(placeOrder(engine, order))
    yield* Effect.log(`Order ${order.id}: ${result.success ? 'Success' : result.reason}`)
    
    if (result.success) {
      // Simulate execution after a delay
      yield* Effect.sleep(2000)
      const executionResult = yield* STM.commit(executeTrade(engine, order, order.price * 0.98)) // 2% better price
      yield* Effect.log(`Execution ${order.id}: ${executionResult.success ? `Filled at $${executionResult.executionPrice}` : 'Failed'}`)
    }
  }
  
  yield* Effect.sleep(5000) // Let monitoring run
})
```

## Advanced Features Deep Dive

### Feature 1: Transactional Composition

TSubscriptionRef operations can be composed with other STM operations to create complex transactional workflows:

#### Basic Transactional Composition

```typescript
import { TSubscriptionRef, TRef, STM, Effect } from "effect"

// Compose TSubscriptionRef with other STM data structures
const atomicTransfer = (
  fromAccountRef: TSubscriptionRef<number>,
  toAccountRef: TSubscriptionRef<number>, 
  auditLogRef: TRef<ReadonlyArray<string>>,
  amount: number
) => STM.gen(function* () {
  const fromBalance = yield* TSubscriptionRef.get(fromAccountRef)
  const toBalance = yield* TSubscriptionRef.get(toAccountRef)
  
  if (fromBalance >= amount) {
    // All operations are atomic - either all succeed or all fail
    yield* TSubscriptionRef.set(fromAccountRef, fromBalance - amount)
    yield* TSubscriptionRef.set(toAccountRef, toBalance + amount)
    
    // Add to audit log atomically
    const currentLog = yield* TRef.get(auditLogRef)
    yield* TRef.set(auditLogRef, [
      ...currentLog,
      `Transfer: $${amount} from account A to account B at ${new Date().toISOString()}`
    ])
    
    return { success: true, newFromBalance: fromBalance - amount, newToBalance: toBalance + amount }
  }
  
  return { success: false, newFromBalance: fromBalance, newToBalance: toBalance }
})
```

#### Real-World Transactional Composition Example

```typescript
// Complex inventory management system
interface InventoryItem {
  readonly id: string
  readonly name: string
  readonly quantity: number
  readonly reservedQuantity: number
  readonly price: number
}

interface Order {
  readonly id: string
  readonly items: ReadonlyArray<{ itemId: string, quantity: number }>
  readonly status: 'pending' | 'confirmed' | 'cancelled'
  readonly totalAmount: number
}

const processOrderWithInventory = (
  inventoryRef: TSubscriptionRef<Map<string, InventoryItem>>,
  ordersRef: TSubscriptionRef<ReadonlyArray<Order>>,
  accountBalanceRef: TSubscriptionRef<number>,
  order: Order
) => STM.gen(function* () {
  const inventory = yield* TSubscriptionRef.get(inventoryRef)
  const currentBalance = yield* TSubscriptionRef.get(accountBalanceRef)
  
  // Check inventory availability
  const availabilityCheck = order.items.every(orderItem => {
    const item = inventory.get(orderItem.itemId)
    return item && (item.quantity - item.reservedQuantity) >= orderItem.quantity
  })
  
  if (!availabilityCheck || currentBalance < order.totalAmount) {
    return { success: false, reason: 'Insufficient inventory or balance' }
  }
  
  // Reserve inventory items atomically
  const updatedInventory = new Map(inventory)
  for (const orderItem of order.items) {
    const item = updatedInventory.get(orderItem.itemId)!
    updatedInventory.set(orderItem.itemId, {
      ...item,
      reservedQuantity: item.reservedQuantity + orderItem.quantity
    })
  }
  
  // Update all state atomically
  yield* TSubscriptionRef.set(inventoryRef, updatedInventory)
  yield* TSubscriptionRef.update(accountBalanceRef, balance => balance - order.totalAmount)
  yield* TSubscriptionRef.update(ordersRef, orders => [...orders, { ...order, status: 'confirmed' }])
  
  return { success: true, reason: 'Order confirmed' }
})
```

### Feature 2: Advanced Subscription Patterns

#### Conditional Subscriptions

```typescript
// Subscribe only to specific state changes
const createConditionalSubscription = <A>(
  ref: TSubscriptionRef<A>,
  predicate: (value: A) => boolean,
  action: (value: A) => Effect.Effect<void>
) => {
  return TSubscriptionRef.changesStream(ref).pipe(
    Stream.filter(predicate),
    Stream.runForEach(action)
  )
}

// Example: Alert when account balance drops below threshold
const createLowBalanceAlert = (accountRef: TSubscriptionRef<number>, threshold: number) => {
  return createConditionalSubscription(
    accountRef,
    balance => balance < threshold,
    balance => Effect.log(`ALERT: Account balance is low: $${balance}`)
  )
}
```

#### Multiple Subscription Coordination

```typescript
// Coordinate multiple TSubscriptionRef streams
const createCoordinatedMonitoring = (
  userRef: TSubscriptionRef<User>,
  sessionRef: TSubscriptionRef<Session>,
  activityRef: TSubscriptionRef<Activity[]>
) => {
  const combinedStream = Stream.combineLatest([
    TSubscriptionRef.changesStream(userRef),
    TSubscriptionRef.changesStream(sessionRef),
    TSubscriptionRef.changesStream(activityRef)
  ])
  
  return combinedStream.pipe(
    Stream.runForEach(([user, session, activities]) => Effect.gen(function* () {
      // Coordinated processing of all state changes
      yield* Effect.log(`User ${user.id} in session ${session.id} with ${activities.length} activities`)
      
      // Complex business logic based on combined state
      if (!session.isValid && activities.length > 0) {
        yield* Effect.log("Suspicious activity detected - invalid session with activities")
      }
    }))
  )
}
```

#### Batched Subscription Processing

```typescript
// Batch multiple changes for efficient processing
const createBatchedSubscription = <A>(
  ref: TSubscriptionRef<A>,
  batchSize: number,
  processBatch: (values: ReadonlyArray<A>) => Effect.Effect<void>
) => {
  return TSubscriptionRef.changesStream(ref).pipe(
    Stream.buffer({ capacity: batchSize }),
    Stream.runForEach(processBatch)
  )
}

// Example: Batch database writes
const createBatchedDatabaseSync = (dataRef: TSubscriptionRef<DatabaseRecord[]>) => {
  return createBatchedSubscription(
    dataRef,
    10,
    records => Effect.gen(function* () {
      yield* Effect.log(`Syncing batch of ${records.length} records to database`)
      // Batch database operation here
    })
  )
}
```

### Feature 3: Error Handling and Recovery

#### Transactional Error Handling

```typescript
// Robust error handling in STM transactions
const safeTransactionalUpdate = <A, E>(
  ref: TSubscriptionRef<A>,
  update: (current: A) => Effect.Effect<A, E>,
  onError: (error: E, current: A) => A
) => Effect.gen(function* () {
  const transaction = STM.gen(function* () {
    const current = yield* TSubscriptionRef.get(ref)
    
    // Attempt update outside STM, then apply result atomically
    const result = yield* STM.suspend(() => 
      update(current).pipe(
        Effect.map(newValue => STM.gen(function* () {
          yield* TSubscriptionRef.set(ref, newValue)
          return newValue
        })),
        Effect.catchAll(error => 
          Effect.succeed(STM.gen(function* () {
            const recoveryValue = onError(error, current)
            yield* TSubscriptionRef.set(ref, recoveryValue)
            return recoveryValue
          }))
        )
      )
    )
    
    return yield* result
  })
  
  return yield* STM.commit(transaction)
})
```

#### Subscription Recovery

```typescript
// Resilient subscription with automatic recovery
const createResilientSubscription = <A>(
  ref: TSubscriptionRef<A>,
  processor: (value: A) => Effect.Effect<void, string>
) => {
  const processWithRetry = (value: A) => 
    processor(value).pipe(
      Effect.retry(Schedule.exponential(1000).pipe(Schedule.compose(Schedule.recurs(3)))),
      Effect.catchAll(error => Effect.log(`Failed to process value after retries: ${error}`))
    )
  
  return TSubscriptionRef.changesStream(ref).pipe(
    Stream.runForEach(processWithRetry),
    Effect.catchAll(error => Effect.gen(function* () {
      yield* Effect.log(`Subscription failed, restarting: ${error}`)
      yield* Effect.sleep(5000)
      // Restart subscription
      return yield* createResilientSubscription(ref, processor)
    }))
  )
}
```

## Practical Patterns & Best Practices

### Pattern 1: State Machine with Reactive Notifications

```typescript
// Implement a state machine using TSubscriptionRef
interface StateMachine<S, E> {
  readonly currentState: S
  readonly history: ReadonlyArray<{ from: S, to: S, event: E, timestamp: number }>
}

const createStateMachine = <S, E>(
  initialState: S,
  transitions: Map<S, Map<E, S>>
) => STM.gen(function* () {
  const machineRef = yield* TSubscriptionRef.make<StateMachine<S, E>>({
    currentState: initialState,
    history: []
  })
  
  const transition = (event: E) => STM.gen(function* () {
    const machine = yield* TSubscriptionRef.get(machineRef)
    const stateTransitions = transitions.get(machine.currentState)
    
    if (!stateTransitions) {
      return { success: false, reason: `No transitions defined for state ${machine.currentState}` }
    }
    
    const nextState = stateTransitions.get(event)
    if (!nextState) {
      return { success: false, reason: `Invalid transition from ${machine.currentState} with event ${event}` }
    }
    
    const historyEntry = {
      from: machine.currentState,
      to: nextState,
      event,
      timestamp: Date.now()
    }
    
    yield* TSubscriptionRef.set(machineRef, {
      currentState: nextState,
      history: [...machine.history, historyEntry]
    })
    
    return { success: true, from: machine.currentState, to: nextState }
  })
  
  return { machineRef, transition }
})

// Usage example: Order processing state machine
type OrderState = 'pending' | 'confirmed' | 'shipped' | 'delivered' | 'cancelled'
type OrderEvent = 'confirm' | 'ship' | 'deliver' | 'cancel'

const createOrderStateMachine = () => {
  const transitions = new Map<OrderState, Map<OrderEvent, OrderState>>([
    ['pending', new Map([
      ['confirm', 'confirmed'],
      ['cancel', 'cancelled']
    ])],
    ['confirmed', new Map([
      ['ship', 'shipped'],
      ['cancel', 'cancelled']
    ])],
    ['shipped', new Map([
      ['deliver', 'delivered']
    ])]
  ])
  
  return createStateMachine<OrderState, OrderEvent>('pending', transitions)
}
```

### Pattern 2: Event Sourcing with TSubscriptionRef

```typescript
// Event sourcing pattern with reactive projections
interface Event {
  readonly id: string
  readonly type: string
  readonly data: any
  readonly timestamp: number
  readonly version: number
}

interface EventStore {
  readonly events: ReadonlyArray<Event>
  readonly version: number
}

interface Projection<T> {
  readonly data: T
  readonly lastEventVersion: number
}

const createEventSourcedAggregate = <T>(
  initialState: T,
  reducer: (state: T, event: Event) => T
) => STM.gen(function* () {
  const eventStoreRef = yield* TSubscriptionRef.make<EventStore>({
    events: [],
    version: 0
  })
  
  const projectionRef = yield* TSubscriptionRef.make<Projection<T>>({
    data: initialState,
    lastEventVersion: 0
  })
  
  const appendEvent = (type: string, data: any) => STM.gen(function* () {
    const store = yield* TSubscriptionRef.get(eventStoreRef)
    const event: Event = {
      id: `event-${Date.now()}-${Math.random()}`,
      type,
      data,
      timestamp: Date.now(),
      version: store.version + 1
    }
    
    const newStore = {
      events: [...store.events, event],
      version: event.version
    }
    
    // Update both event store and projection atomically
    yield* TSubscriptionRef.set(eventStoreRef, newStore)
    
    const projection = yield* TSubscriptionRef.get(projectionRef)
    const newData = reducer(projection.data, event)
    yield* TSubscriptionRef.set(projectionRef, {
      data: newData,
      lastEventVersion: event.version
    })
    
    return event
  })
  
  return { eventStoreRef, projectionRef, appendEvent }
})

// Usage: User aggregate with event sourcing
interface User {
  readonly id: string
  readonly name: string
  readonly email: string
  readonly isActive: boolean
}

const userReducer = (user: User, event: Event): User => {
  switch (event.type) {
    case 'UserCreated':
      return { ...event.data }
    case 'EmailUpdated':
      return { ...user, email: event.data.email }
    case 'UserDeactivated':
      return { ...user, isActive: false }
    case 'UserActivated':
      return { ...user, isActive: true }
    default:
      return user
  }
}

const createUserAggregate = (userId: string) => {
  const initialUser: User = {
    id: userId,
    name: '',
    email: '',
    isActive: false
  }
  
  return createEventSourcedAggregate(initialUser, userReducer)
}
```

### Pattern 3: Reactive Validation Pipeline

```typescript
// Create a validation pipeline that reacts to state changes
interface ValidationResult {
  readonly isValid: boolean
  readonly errors: ReadonlyArray<string>
  readonly warnings: ReadonlyArray<string>
}

interface ValidatedData<T> {
  readonly data: T
  readonly validation: ValidationResult
  readonly lastValidated: number
}

const createValidatedRef = <T>(
  initialValue: T,
  validators: ReadonlyArray<(value: T) => ValidationResult>
) => STM.gen(function* () {
  const validate = (value: T): ValidationResult => {
    const results = validators.map(validator => validator(value))
    
    return {
      isValid: results.every(r => r.isValid),
      errors: results.flatMap(r => r.errors),
      warnings: results.flatMap(r => r.warnings)
    }
  }
  
  const validationResult = validate(initialValue)
  const validatedRef = yield* TSubscriptionRef.make<ValidatedData<T>>({
    data: initialValue,
    validation: validationResult,
    lastValidated: Date.now()
  })
  
  const updateWithValidation = (newValue: T) => STM.gen(function* () {
    const validation = validate(newValue)
    yield* TSubscriptionRef.set(validatedRef, {
      data: newValue,
      validation,
      lastValidated: Date.now()
    })
    return validation
  })
  
  return { validatedRef, updateWithValidation }
})

// Usage: Form validation with reactive feedback
interface FormData {
  readonly email: string
  readonly password: string
  readonly confirmPassword: string
}

const emailValidator = (form: FormData): ValidationResult => ({
  isValid: /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(form.email),
  errors: /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(form.email) ? [] : ['Invalid email format'],
  warnings: []
})

const passwordValidator = (form: FormData): ValidationResult => {
  const isValid = form.password.length >= 8 && form.password === form.confirmPassword
  const errors = []
  
  if (form.password.length < 8) errors.push('Password must be at least 8 characters')
  if (form.password !== form.confirmPassword) errors.push('Passwords do not match')
  
  return { isValid, errors, warnings: [] }
}

const createFormValidator = () => {
  const initialForm: FormData = { email: '', password: '', confirmPassword: '' }
  return createValidatedRef(initialForm, [emailValidator, passwordValidator])
}
```

### Pattern 4: Resource Pool with Reactive Monitoring

```typescript
// Resource pool with reactive capacity monitoring
interface PoolResource<T> {
  readonly id: string
  readonly resource: T
  readonly isInUse: boolean
  readonly checkedOutAt?: number
  readonly usageCount: number
}

interface ResourcePool<T> {
  readonly resources: ReadonlyArray<PoolResource<T>>
  readonly capacity: number
  readonly inUse: number
  readonly available: number
}

const createResourcePool = <T>(
  resources: ReadonlyArray<T>,
  maxCapacity: number
) => STM.gen(function* () {
  const poolResources: ReadonlyArray<PoolResource<T>> = resources.map((resource, index) => ({
    id: `resource-${index}`,
    resource,
    isInUse: false,
    usageCount: 0
  }))
  
  const poolRef = yield* TSubscriptionRef.make<ResourcePool<T>>({
    resources: poolResources,
    capacity: maxCapacity,
    inUse: 0,
    available: poolResources.length
  })
  
  const acquireResource = () => STM.gen(function* () {
    const pool = yield* TSubscriptionRef.get(poolRef)
    const availableResource = pool.resources.find(r => !r.isInUse)
    
    if (!availableResource) {
      return null
    }
    
    const updatedResources = pool.resources.map(r =>
      r.id === availableResource.id
        ? { ...r, isInUse: true, checkedOutAt: Date.now(), usageCount: r.usageCount + 1 }
        : r
    )
    
    yield* TSubscriptionRef.set(poolRef, {
      ...pool,
      resources: updatedResources,
      inUse: pool.inUse + 1,
      available: pool.available - 1
    })
    
    return availableResource.resource
  })
  
  const releaseResource = (resourceId: string) => STM.gen(function* () {
    const pool = yield* TSubscriptionRef.get(poolRef)
    const updatedResources = pool.resources.map(r =>
      r.id === resourceId
        ? { ...r, isInUse: false, checkedOutAt: undefined }
        : r
    )
    
    yield* TSubscriptionRef.set(poolRef, {
      ...pool,
      resources: updatedResources,
      inUse: pool.inUse - 1,
      available: pool.available + 1
    })
  })
  
  return { poolRef, acquireResource, releaseResource }
})

// Pool monitoring
const createPoolMonitor = <T>(poolRef: TSubscriptionRef<ResourcePool<T>>) => {
  return TSubscriptionRef.changesStream(poolRef).pipe(
    Stream.runForEach(pool => Effect.gen(function* () {
      const utilizationRate = (pool.inUse / pool.capacity) * 100
      yield* Effect.log(`Pool Utilization: ${utilizationRate.toFixed(1)}% (${pool.inUse}/${pool.capacity})`)
      
      if (utilizationRate > 90) {
        yield* Effect.log("WARNING: Pool utilization is very high!")
      }
      
      const longRunningResources = pool.resources.filter(r => 
        r.isInUse && r.checkedOutAt && (Date.now() - r.checkedOutAt > 30000)
      )
      
      if (longRunningResources.length > 0) {
        yield* Effect.log(`WARNING: ${longRunningResources.length} resources have been checked out for >30s`)
      }
    }))
  )
}
```

## Integration Examples

### Integration with Effect HTTP Server

```typescript
import { TSubscriptionRef, STM, Effect, Stream } from "effect"
import { HttpServer, HttpRouter, HttpServerRequest, HttpServerResponse } from "@effect/platform"

// Real-time API with TSubscriptionRef
interface ApiMetrics {
  readonly requestCount: number
  readonly errorCount: number
  readonly averageResponseTime: number
  readonly activeConnections: number
}

const createMetricsAPI = STM.gen(function* () {
  const metricsRef = yield* TSubscriptionRef.make<ApiMetrics>({
    requestCount: 0,
    errorCount: 0,
    averageResponseTime: 0,
    activeConnections: 0
  })
  
  const connectionsRef = yield* TSubscriptionRef.make<Set<string>>(new Set())
  
  return { metricsRef, connectionsRef }
})

// HTTP routes with reactive metrics
const createMetricsRouter = (
  refs: { metricsRef: TSubscriptionRef<ApiMetrics>, connectionsRef: TSubscriptionRef<Set<string>> }
) => {
  const metricsMiddleware = (handler: (req: HttpServerRequest.HttpServerRequest) => Effect.Effect<HttpServerResponse.HttpServerResponse>) => 
    (req: HttpServerRequest.HttpServerRequest) => Effect.gen(function* () {
      const startTime = Date.now()
      
      // Increment active connections
      const connectionId = Math.random().toString(36)
      yield* STM.commit(STM.gen(function* () {
        const currentConnections = yield* TSubscriptionRef.get(refs.connectionsRef)
        const newConnections = new Set(currentConnections)
        newConnections.add(connectionId)
        yield* TSubscriptionRef.set(refs.connectionsRef, newConnections)
        
        yield* TSubscriptionRef.update(refs.metricsRef, m => ({
          ...m,
          activeConnections: newConnections.size
        }))
      }))
      
      try {
        const response = yield* handler(req)
        const duration = Date.now() - startTime
        
        // Update success metrics
        yield* STM.commit(STM.gen(function* () {
          const metrics = yield* TSubscriptionRef.get(refs.metricsRef)
          const newAverage = (metrics.averageResponseTime * metrics.requestCount + duration) / (metrics.requestCount + 1)
          
          yield* TSubscriptionRef.update(refs.metricsRef, m => ({
            ...m,
            requestCount: m.requestCount + 1,
            averageResponseTime: newAverage
          }))
        }))
        
        return response
      } catch (error) {
        // Update error metrics
        yield* STM.commit(STM.gen(function* () {
          yield* TSubscriptionRef.update(refs.metricsRef, m => ({
            ...m,
            requestCount: m.requestCount + 1,
            errorCount: m.errorCount + 1
          }))
        }))
        
        throw error
      } finally {
        // Decrement active connections
        yield* STM.commit(STM.gen(function* () {
          const currentConnections = yield* TSubscriptionRef.get(refs.connectionsRef)
          const newConnections = new Set(currentConnections)
          newConnections.delete(connectionId)
          yield* TSubscriptionRef.set(refs.connectionsRef, newConnections)
          
          yield* TSubscriptionRef.update(refs.metricsRef, m => ({
            ...m,
            activeConnections: newConnections.size
          }))
        }))
      }
    })
  
  const router = HttpRouter.empty.pipe(
    HttpRouter.get('/metrics', metricsMiddleware(() => Effect.gen(function* () {
      const metrics = yield* STM.commit(TSubscriptionRef.get(refs.metricsRef))
      return HttpServerResponse.json(metrics)
    }))),
    
    HttpRouter.get('/metrics/stream', metricsMiddleware(() => Effect.gen(function* () {
      const stream = TSubscriptionRef.changesStream(refs.metricsRef).pipe(
        Stream.map(metrics => JSON.stringify(metrics) + '\n'),
        Stream.encodeText
      )
      
      return HttpServerResponse.stream(stream, {
        headers: { 'Content-Type': 'application/x-ndjson' }
      })
    })))
  )
  
  return router
}

// Complete HTTP server with reactive metrics
const metricsServerProgram = Effect.gen(function* () {
  const refs = yield* STM.commit(createMetricsAPI)
  const router = createMetricsRouter(refs)
  
  // Start metrics monitoring
  const metricsLogger = TSubscriptionRef.changesStream(refs.metricsRef).pipe(
    Stream.runForEach(metrics => 
      Effect.log(`Requests: ${metrics.requestCount}, Errors: ${metrics.errorCount}, Avg Response: ${metrics.averageResponseTime.toFixed(2)}ms, Active: ${metrics.activeConnections}`)
    )
  )
  
  yield* Effect.fork(metricsLogger)
  
  // Start HTTP server
  const server = HttpServer.serve(router, { port: 3000 })
  yield* server
})
```

### Integration with Database Transactions

```typescript
import { TSubscriptionRef, STM, Effect, Layer } from "effect"
import { SqlClient } from "@effect/sql"

// Database integration with transactional consistency
interface DatabaseEntity {
  readonly id: string
  readonly version: number
  readonly data: any
  readonly updatedAt: Date
}

interface EntityCache {
  readonly entities: Map<string, DatabaseEntity>
  readonly pendingWrites: Set<string>
  readonly lastSync: number
}

const createDatabaseSync = STM.gen(function* () {
  const cacheRef = yield* TSubscriptionRef.make<EntityCache>({
    entities: new Map(),
    pendingWrites: new Set(),
    lastSync: Date.now()
  })
  
  return cacheRef
})

// Synchronize cache changes with database
const createDatabaseSyncService = (cacheRef: TSubscriptionRef<EntityCache>) => {
  const syncToDatabase = (entity: DatabaseEntity) => Effect.gen(function* () {
    const sql = yield* SqlClient.SqlClient
    
    yield* sql.withTransaction(
      sql`
        INSERT INTO entities (id, version, data, updated_at)
        VALUES (${entity.id}, ${entity.version}, ${entity.data}, ${entity.updatedAt})
        ON CONFLICT (id) DO UPDATE SET
          version = EXCLUDED.version,
          data = EXCLUDED.data,
          updated_at = EXCLUDED.updated_at
        WHERE entities.version < EXCLUDED.version
      `
    )
  })
  
  const processPendingWrites = Effect.gen(function* () {
    const cache = yield* STM.commit(TSubscriptionRef.get(cacheRef))
    
    if (cache.pendingWrites.size === 0) return
    
    const entities = Array.from(cache.pendingWrites).map(id => cache.entities.get(id)!).filter(Boolean)
    
    // Batch database writes
    yield* Effect.forEach(entities, syncToDatabase, { batching: true })
    
    // Clear pending writes atomically
    yield* STM.commit(STM.gen(function* () {
      yield* TSubscriptionRef.update(cacheRef, c => ({
        ...c,
        pendingWrites: new Set(),
        lastSync: Date.now()
      }))
    }))
    
    yield* Effect.log(`Synced ${entities.length} entities to database`)
  })
  
  // Reactive sync processor
  const syncProcessor = TSubscriptionRef.changesStream(cacheRef).pipe(
    Stream.filter(cache => cache.pendingWrites.size > 0),
    Stream.debounce(1000), // Batch writes every 1 second
    Stream.runForEach(() => processPendingWrites)
  )
  
  return { syncProcessor, processPendingWrites }
}

// Entity operations with cache-database consistency
const createEntityService = (cacheRef: TSubscriptionRef<EntityCache>) => {
  const upsertEntity = (id: string, data: any) => STM.gen(function* () {
    const cache = yield* TSubscriptionRef.get(cacheRef)
    const existingEntity = cache.entities.get(id)
    
    const entity: DatabaseEntity = {
      id,
      version: existingEntity ? existingEntity.version + 1 : 1,
      data,
      updatedAt: new Date()
    }
    
    const updatedEntities = new Map(cache.entities)
    updatedEntities.set(id, entity)
    
    const updatedPending = new Set(cache.pendingWrites)
    updatedPending.add(id)
    
    yield* TSubscriptionRef.set(cacheRef, {
      ...cache,
      entities: updatedEntities,
      pendingWrites: updatedPending
    })
    
    return entity
  })
  
  const getEntity = (id: string) => STM.gen(function* () {
    const cache = yield* TSubscriptionRef.get(cacheRef)
    return cache.entities.get(id) || null
  })
  
  const deleteEntity = (id: string) => STM.gen(function* () {
    const cache = yield* TSubscriptionRef.get(cacheRef)
    const updatedEntities = new Map(cache.entities)
    updatedEntities.delete(id)
    
    const updatedPending = new Set(cache.pendingWrites)
    updatedPending.add(id) // Mark for deletion sync
    
    yield* TSubscriptionRef.set(cacheRef, {
      ...cache,
      entities: updatedEntities,
      pendingWrites: updatedPending
    })
    
    return true
  })
  
  return { upsertEntity, getEntity, deleteEntity }
}

// Complete database integration program
const databaseIntegrationProgram = Effect.gen(function* () {
  const cacheRef = yield* STM.commit(createDatabaseSync)
  const syncService = createDatabaseSyncService(cacheRef)
  const entityService = createEntityService(cacheRef)
  
  // Start background sync
  yield* Effect.fork(syncService.syncProcessor)
  
  // Example usage
  const entity1 = yield* STM.commit(entityService.upsertEntity('user-1', { name: 'John', email: 'john@example.com' }))
  const entity2 = yield* STM.commit(entityService.upsertEntity('user-2', { name: 'Jane', email: 'jane@example.com' }))
  
  yield* Effect.log(`Created entities: ${entity1.id}, ${entity2.id}`)
  
  // Entities will be automatically synced to database
  yield* Effect.sleep(2000)
  
  const retrieved = yield* STM.commit(entityService.getEntity('user-1'))
  yield* Effect.log(`Retrieved entity: ${retrieved?.data.name}`)
})
```

### Integration with WebSocket Real-Time Updates

```typescript
import { TSubscriptionRef, STM, Effect, Stream, Queue } from "effect"

// WebSocket integration for real-time updates
interface WebSocketConnection {
  readonly id: string
  readonly userId: string
  readonly connectedAt: number
  readonly subscriptions: Set<string>
}

interface WebSocketMessage {
  readonly type: string
  readonly payload: any
  readonly timestamp: number
}

interface WebSocketState {
  readonly connections: Map<string, WebSocketConnection>
  readonly messageQueue: ReadonlyArray<{ connectionId: string, message: WebSocketMessage }>
  readonly totalConnections: number
}

const createWebSocketManager = STM.gen(function* () {
  const stateRef = yield* TSubscriptionRef.make<WebSocketState>({
    connections: new Map(),
    messageQueue: [],
    totalConnections: 0
  })
  
  const messageQueueRef = yield* Queue.unbounded<{ connectionId: string, message: WebSocketMessage }>()
  
  return { stateRef, messageQueueRef }
})

// WebSocket connection management
const createWebSocketService = (
  refs: { stateRef: TSubscriptionRef<WebSocketState>, messageQueueRef: Queue.Queue<{ connectionId: string, message: WebSocketMessage }> }
) => {
  const addConnection = (connection: WebSocketConnection) => STM.gen(function* () {
    const state = yield* TSubscriptionRef.get(refs.stateRef)
    const updatedConnections = new Map(state.connections)
    updatedConnections.set(connection.id, connection)
    
    yield* TSubscriptionRef.set(refs.stateRef, {
      ...state,
      connections: updatedConnections,
      totalConnections: updatedConnections.size
    })
  })
  
  const removeConnection = (connectionId: string) => STM.gen(function* () {
    const state = yield* TSubscriptionRef.get(refs.stateRef)
    const updatedConnections = new Map(state.connections)
    updatedConnections.delete(connectionId)
    
    yield* TSubscriptionRef.set(refs.stateRef, {
      ...state,
      connections: updatedConnections,
      totalConnections: updatedConnections.size
    })
  })
  
  const broadcastMessage = (message: WebSocketMessage, filter?: (conn: WebSocketConnection) => boolean) => STM.gen(function* () {
    const state = yield* TSubscriptionRef.get(refs.stateRef)
    const targetConnections = Array.from(state.connections.values()).filter(filter || (() => true))
    
    // Queue messages for all target connections
    yield* STM.forEach(targetConnections, conn => 
      Queue.offer(refs.messageQueueRef, { connectionId: conn.id, message }).pipe(STM.fromEffect)
    )
  })
  
  const subscribeToChannel = (connectionId: string, channel: string) => STM.gen(function* () {
    const state = yield* TSubscriptionRef.get(refs.stateRef)
    const connection = state.connections.get(connectionId)
    
    if (!connection) return false
    
    const updatedConnection = {
      ...connection,
      subscriptions: new Set([...connection.subscriptions, channel])
    }
    
    const updatedConnections = new Map(state.connections)
    updatedConnections.set(connectionId, updatedConnection)
    
    yield* TSubscriptionRef.set(refs.stateRef, {
      ...state,
      connections: updatedConnections
    })
    
    return true
  })
  
  return { addConnection, removeConnection, broadcastMessage, subscribeToChannel }
}

// Real-time data publisher using TSubscriptionRef
const createRealtimePublisher = <T>(
  dataRef: TSubscriptionRef<T>,
  wsService: ReturnType<typeof createWebSocketService>,
  channel: string
) => {
  return TSubscriptionRef.changesStream(dataRef).pipe(
    Stream.runForEach(data => STM.commit(
      wsService.broadcastMessage(
        {
          type: 'data-update',
          payload: { channel, data },
          timestamp: Date.now()
        },
        conn => conn.subscriptions.has(channel)
      )
    ))
  )
}

// Message processing pipeline
const createMessageProcessor = (
  messageQueueRef: Queue.Queue<{ connectionId: string, message: WebSocketMessage }>
) => {
  return Stream.fromQueue(messageQueueRef).pipe(
    Stream.runForEach(({ connectionId, message }) => Effect.gen(function* () {
      // Simulate sending message to WebSocket connection
      yield* Effect.log(`Sending to ${connectionId}: ${message.type}`)
      
      // In real implementation, this would send to actual WebSocket
      // await webSocketConnection.send(JSON.stringify(message))
    }))
  )
}

// Complete WebSocket program
const webSocketProgram = Effect.gen(function* () {
  const refs = yield* STM.commit(createWebSocketManager)
  const wsService = createWebSocketService(refs)
  
  // Start message processor
  yield* Effect.fork(createMessageProcessor(refs.messageQueueRef))
  
  // Create some test data sources
  const userCountRef = yield* STM.commit(TSubscriptionRef.make(0))
  const systemStatusRef = yield* STM.commit(TSubscriptionRef.make({ status: 'healthy', uptime: 0 }))
  
  // Set up real-time publishers
  yield* Effect.fork(createRealtimePublisher(userCountRef, wsService, 'user-count'))
  yield* Effect.fork(createRealtimePublisher(systemStatusRef, wsService, 'system-status'))
  
  // Monitor connection state
  const connectionMonitor = TSubscriptionRef.changesStream(refs.stateRef).pipe(
    Stream.runForEach(state => 
      Effect.log(`WebSocket connections: ${state.totalConnections}`)
    )
  )
  
  yield* Effect.fork(connectionMonitor)
  
  // Simulate connections and data updates
  const connection1: WebSocketConnection = {
    id: 'conn-1',
    userId: 'user-123',
    connectedAt: Date.now(),
    subscriptions: new Set()
  }
  
  const connection2: WebSocketConnection = {
    id: 'conn-2',
    userId: 'user-456',
    connectedAt: Date.now(),
    subscriptions: new Set()
  }
  
  yield* STM.commit(wsService.addConnection(connection1))
  yield* STM.commit(wsService.addConnection(connection2))
  
  // Subscribe to channels
  yield* STM.commit(wsService.subscribeToChannel('conn-1', 'user-count'))
  yield* STM.commit(wsService.subscribeToChannel('conn-2', 'system-status'))
  yield* STM.commit(wsService.subscribeToChannel('conn-2', 'user-count'))
  
  // Simulate data changes
  for (let i = 1; i <= 5; i++) {
    yield* STM.commit(TSubscriptionRef.set(userCountRef, i * 100))
    yield* STM.commit(TSubscriptionRef.update(systemStatusRef, s => ({ ...s, uptime: i * 1000 })))
    yield* Effect.sleep(1000)
  }
  
  // Cleanup
  yield* STM.commit(wsService.removeConnection('conn-1'))
  yield* STM.commit(wsService.removeConnection('conn-2'))
})
```

## Conclusion

TSubscriptionRef provides powerful transactional reactive state management for Effect applications, combining STM's atomicity guarantees with reactive subscription capabilities.

Key benefits:
- **Transactional Consistency**: All state updates and notifications happen atomically
- **Reactive Programming**: Stream-based subscriptions with automatic resource management  
- **Composability**: Seamless integration with other STM data structures and Effect modules
- **Type Safety**: Full TypeScript support with compile-time guarantees

TSubscriptionRef is ideal for building reactive systems that require strong consistency guarantees across multiple shared states, such as real-time applications, trading systems, game engines, and distributed caches where atomic updates with consistent notifications are critical.