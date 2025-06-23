# SubscriptionRef: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem SubscriptionRef Solves

Managing reactive state that multiple components need to observe creates complex coordination challenges. Traditional approaches often lead to memory leaks, inconsistent state notifications, and tightly coupled observer patterns:

```typescript
// Traditional approach - manual observer pattern
class StateManager<T> {
  private value: T
  private observers: Array<(value: T) => void> = []
  
  constructor(initial: T) {
    this.value = initial
  }
  
  get(): T {
    return this.value
  }
  
  set(newValue: T): void {
    this.value = newValue
    // Manual notification - race conditions possible
    this.observers.forEach(observer => {
      try {
        observer(newValue) // Unhandled errors can break the chain
      } catch (error) {
        console.error('Observer error:', error)
      }
    })
  }
  
  subscribe(observer: (value: T) => void): () => void {
    this.observers.push(observer)
    observer(this.value) // Send current value
    
    // Cleanup function - easy to forget to call
    return () => {
      const index = this.observers.indexOf(observer)
      if (index > -1) {
        this.observers.splice(index, 1)
      }
    }
  }
}

// Usage problems
const stateManager = new StateManager(0)

// Memory leaks - forgetting to unsubscribe
const unsubscribe1 = stateManager.subscribe(value => updateUI(value))
const unsubscribe2 = stateManager.subscribe(value => logChange(value))
// unsubscribe1() and unsubscribe2() often forgotten

// Race conditions in concurrent updates
Promise.all([
  Promise.resolve().then(() => stateManager.set(1)),
  Promise.resolve().then(() => stateManager.set(2)),
  Promise.resolve().then(() => stateManager.set(3))
])
```

This approach leads to:
- **Memory Leaks** - Observers not properly cleaned up, holding references indefinitely
- **Race Conditions** - Multiple concurrent updates can cause inconsistent notifications
- **Error Propagation** - One failing observer can affect others or crash the system
- **Resource Management** - Manual subscription cleanup is error-prone and often forgotten
- **Complex Coordination** - Difficult to compose multiple reactive state sources

### The SubscriptionRef Solution

SubscriptionRef provides a thread-safe, reactive state container with automatic subscription management and streaming change notifications:

```typescript
import { SubscriptionRef, Effect, Stream } from "effect"

// The Effect solution - automatic cleanup, type-safe, composable
const makeReactiveCounter = Effect.gen(function* () {
  const counterRef = yield* SubscriptionRef.make(0)
  
  const increment = () => SubscriptionRef.update(counterRef, n => n + 1)
  const decrement = () => SubscriptionRef.update(counterRef, n => n - 1)
  const reset = () => SubscriptionRef.set(counterRef, 0)
  
  // Stream of all changes - automatic cleanup, no memory leaks
  const changes = counterRef.changes
  
  return { increment, decrement, reset, changes, get: () => SubscriptionRef.get(counterRef) }
})

// Usage - clean, composable, safe
const program = Effect.gen(function* () {
  const counter = yield* makeReactiveCounter
  
  // Multiple observers with automatic cleanup
  const observer1 = counter.changes.pipe(
    Stream.take(5),
    Stream.runForEach(value => Effect.log(`UI: Counter is now ${value}`))
  )
  
  const observer2 = counter.changes.pipe(
    Stream.filter(value => value % 2 === 0),
    Stream.take(3),
    Stream.runForEach(value => Effect.log(`Analytics: Even value ${value}`))
  )
  
  // Run observers concurrently
  yield* Effect.fork(observer1)
  yield* Effect.fork(observer2)
  
  // Update state - all observers automatically notified
  yield* counter.increment()
  yield* counter.increment()
  yield* counter.increment()
  yield* counter.increment()
})
```

### Key Concepts

**SubscriptionRef<A>**: A thread-safe mutable reference that can be subscribed to, combining the functionality of Ref with reactive change notifications.

**Changes Stream**: A Stream<A> that emits the current value immediately upon subscription, then emits all subsequent changes to the state.

**Atomic Updates**: All state modifications are atomic and thread-safe, ensuring consistency across concurrent operations.

**Automatic Cleanup**: Subscriptions are managed through Effect's resource system, preventing memory leaks.

## Basic Usage Patterns

### Pattern 1: Creating and Basic Operations

```typescript
import { SubscriptionRef, Effect } from "effect"

// Create a SubscriptionRef with initial value
const createUserPreferences = Effect.gen(function* () {
  const prefsRef = yield* SubscriptionRef.make({
    theme: 'light',
    language: 'en',
    notifications: true
  })
  
  return prefsRef
})

// Basic operations
const basicOperations = Effect.gen(function* () {
  const prefs = yield* createUserPreferences
  
  // Get current value
  const current = yield* SubscriptionRef.get(prefs)
  yield* Effect.log(`Current preferences: ${JSON.stringify(current)}`)
  
  // Set new value
  yield* SubscriptionRef.set(prefs, {
    theme: 'dark',
    language: 'es', 
    notifications: false
  })
  
  // Update with function
  yield* SubscriptionRef.update(prefs, prev => ({
    ...prev,
    theme: prev.theme === 'dark' ? 'light' : 'dark'
  }))
})
```

### Pattern 2: Subscribing to Changes

```typescript
import { Stream } from "effect"

// Subscribe to all changes
const subscribeToChanges = Effect.gen(function* () {
  const prefs = yield* createUserPreferences
  
  // Stream includes current value + all future changes
  const subscription = prefs.changes.pipe(
    Stream.runForEach(preferences => 
      Effect.log(`Preferences updated: ${JSON.stringify(preferences)}`)
    )
  )
  
  // Run subscription in background
  yield* Effect.fork(subscription)
  
  // Make some changes
  yield* SubscriptionRef.update(prefs, p => ({ ...p, theme: 'dark' }))
  yield* SubscriptionRef.update(prefs, p => ({ ...p, language: 'fr' }))
  
  yield* Effect.sleep("100 millis") // Allow notifications to process
})
```

### Pattern 3: Filtered and Transformed Subscriptions

```typescript
// Subscribe to specific changes only
const selectiveSubscription = Effect.gen(function* () {
  const prefs = yield* createUserPreferences
  
  // Only react to theme changes
  const themeChanges = prefs.changes.pipe(
    Stream.map(p => p.theme),
    Stream.changes, // Only emit when theme actually changes
    Stream.runForEach(theme => 
      Effect.log(`Theme changed to: ${theme}`)
    )
  )
  
  // Only react to notification setting changes
  const notificationChanges = prefs.changes.pipe(
    Stream.map(p => p.notifications),
    Stream.changes,
    Stream.runForEach(enabled => 
      Effect.log(`Notifications ${enabled ? 'enabled' : 'disabled'}`)
    )
  )
  
  yield* Effect.fork(themeChanges)
  yield* Effect.fork(notificationChanges)
  
  yield* SubscriptionRef.update(prefs, p => ({ ...p, theme: 'dark' }))
  yield* SubscriptionRef.update(prefs, p => ({ ...p, language: 'de' })) // Won't trigger theme/notification observers
  yield* SubscriptionRef.update(prefs, p => ({ ...p, notifications: false }))
})
```

## Real-World Examples

### Example 1: Real-Time Shopping Cart

```typescript
import { SubscriptionRef, Effect, Stream, Array as Arr } from "effect"

interface CartItem {
  readonly id: string
  readonly name: string
  readonly price: number
  readonly quantity: number
}

interface ShoppingCart {
  readonly items: ReadonlyArray<CartItem>
  readonly total: number
  readonly itemCount: number
}

const makeShoppingCart = Effect.gen(function* () {
  const cartRef = yield* SubscriptionRef.make<ShoppingCart>({
    items: [],
    total: 0,
    itemCount: 0
  })
  
  const calculateTotals = (items: ReadonlyArray<CartItem>): ShoppingCart => ({
    items,
    total: items.reduce((sum, item) => sum + (item.price * item.quantity), 0),
    itemCount: items.reduce((sum, item) => sum + item.quantity, 0)
  })
  
  const addItem = (newItem: Omit<CartItem, 'quantity'>, quantity = 1) =>
    SubscriptionRef.update(cartRef, cart => {
      const existingIndex = cart.items.findIndex(item => item.id === newItem.id)
      
      if (existingIndex >= 0) {
        // Update existing item quantity
        const updatedItems = cart.items.map((item, index) =>
          index === existingIndex 
            ? { ...item, quantity: item.quantity + quantity }
            : item
        )
        return calculateTotals(updatedItems)
      } else {
        // Add new item
        const updatedItems = [...cart.items, { ...newItem, quantity }]
        return calculateTotals(updatedItems)
      }
    })
  
  const removeItem = (itemId: string) =>
    SubscriptionRef.update(cartRef, cart => {
      const updatedItems = cart.items.filter(item => item.id !== itemId)
      return calculateTotals(updatedItems)
    })
  
  const updateQuantity = (itemId: string, quantity: number) =>
    SubscriptionRef.update(cartRef, cart => {
      if (quantity <= 0) {
        return calculateTotals(cart.items.filter(item => item.id !== itemId))
      }
      
      const updatedItems = cart.items.map(item =>
        item.id === itemId ? { ...item, quantity } : item
      )
      return calculateTotals(updatedItems)
    })
  
  const clear = () => SubscriptionRef.set(cartRef, {
    items: [],
    total: 0,
    itemCount: 0
  })
  
  // Reactive streams for different aspects
  const totalStream = cartRef.changes.pipe(Stream.map(cart => cart.total))
  const itemCountStream = cartRef.changes.pipe(Stream.map(cart => cart.itemCount))
  const itemsStream = cartRef.changes.pipe(Stream.map(cart => cart.items))
  
  return {
    addItem,
    removeItem,
    updateQuantity,
    clear,
    get: () => SubscriptionRef.get(cartRef),
    changes: cartRef.changes,
    totalStream,
    itemCountStream,
    itemsStream
  }
})

// Usage: Multiple UI components react to cart changes
const ecommerceApp = Effect.gen(function* () {
  const cart = yield* makeShoppingCart
  
  // Header component subscribes to item count
  const headerUpdater = cart.itemCountStream.pipe(
    Stream.runForEach(count => 
      Effect.log(`Header: Cart badge shows ${count} items`)
    )
  )
  
  // Sidebar component subscribes to total
  const sidebarUpdater = cart.totalStream.pipe(
    Stream.runForEach(total => 
      Effect.log(`Sidebar: Cart total $${total.toFixed(2)}`)
    )
  )
  
  // Cart page subscribes to full state
  const cartPageUpdater = cart.changes.pipe(
    Stream.runForEach(cartState => 
      Effect.log(`Cart Page: ${cartState.itemCount} items, $${cartState.total.toFixed(2)} total`)
    )
  )
  
  // Start all UI updaters
  yield* Effect.fork(headerUpdater)
  yield* Effect.fork(sidebarUpdater)
  yield* Effect.fork(cartPageUpdater)
  
  // Simulate user actions
  yield* cart.addItem({ id: '1', name: 'Laptop', price: 999.99 })
  yield* cart.addItem({ id: '2', name: 'Mouse', price: 29.99 }, 2)
  yield* cart.updateQuantity('1', 2)
  yield* cart.removeItem('2')
  
  yield* Effect.sleep("100 millis") // Allow UI updates to process
})
```

### Example 2: Real-Time Application State Management

```typescript
interface User {
  readonly id: string
  readonly name: string
  readonly email: string
  readonly isOnline: boolean
}

interface AppState {
  readonly currentUser: User | null
  readonly users: ReadonlyArray<User>
  readonly notifications: ReadonlyArray<string>
  readonly isLoading: boolean
}

const makeAppStateManager = Effect.gen(function* () {
  const stateRef = yield* SubscriptionRef.make<AppState>({
    currentUser: null,
    users: [],
    notifications: [],
    isLoading: false
  })
  
  // User management
  const setCurrentUser = (user: User) =>
    SubscriptionRef.update(stateRef, state => ({
      ...state,
      currentUser: user
    }))
  
  const updateUserStatus = (userId: string, isOnline: boolean) =>
    SubscriptionRef.update(stateRef, state => ({
      ...state,
      users: state.users.map(user =>
        user.id === userId ? { ...user, isOnline } : user
      ),
      currentUser: state.currentUser?.id === userId 
        ? { ...state.currentUser, isOnline }
        : state.currentUser
    }))
  
  const addUsers = (newUsers: ReadonlyArray<User>) =>
    SubscriptionRef.update(stateRef, state => ({
      ...state,
      users: [...state.users, ...newUsers]
    }))
  
  // Notification management
  const addNotification = (message: string) =>
    SubscriptionRef.update(stateRef, state => ({
      ...state,
      notifications: [...state.notifications, message]
    }))
  
  const clearNotifications = () =>
    SubscriptionRef.update(stateRef, state => ({
      ...state,
      notifications: []
    }))
  
  // Loading state
  const setLoading = (isLoading: boolean) =>
    SubscriptionRef.update(stateRef, state => ({
      ...state,
      isLoading
    }))
  
  // Reactive selectors
  const currentUserStream = stateRef.changes.pipe(
    Stream.map(state => state.currentUser)
  )
  
  const onlineUsersStream = stateRef.changes.pipe(
    Stream.map(state => state.users.filter(user => user.isOnline))
  )
  
  const notificationCountStream = stateRef.changes.pipe(
    Stream.map(state => state.notifications.length)
  )
  
  const loadingStream = stateRef.changes.pipe(
    Stream.map(state => state.isLoading)
  )
  
  return {
    setCurrentUser,
    updateUserStatus,
    addUsers,
    addNotification,
    clearNotifications,
    setLoading,
    get: () => SubscriptionRef.get(stateRef),
    changes: stateRef.changes,
    currentUserStream,
    onlineUsersStream,
    notificationCountStream,
    loadingStream
  }
})

// Multiple components reacting to different parts of state
const appExample = Effect.gen(function* () {
  const appState = yield* makeAppStateManager
  
  // Navigation component watches current user
  const navUpdater = appState.currentUserStream.pipe(
    Stream.runForEach(user => 
      Effect.log(`Nav: ${user ? `Logged in as ${user.name}` : 'Not logged in'}`)
    )
  )
  
  // User list component watches online users
  const userListUpdater = appState.onlineUsersStream.pipe(
    Stream.runForEach(onlineUsers => 
      Effect.log(`User List: ${onlineUsers.length} users online`)
    )
  )
  
  // Notification component watches notification count
  const notificationUpdater = appState.notificationCountStream.pipe(
    Stream.runForEach(count => 
      Effect.log(`Notifications: ${count} unread messages`)
    )
  )
  
  // Loading spinner watches loading state
  const loadingUpdater = appState.loadingStream.pipe(
    Stream.runForEach(isLoading => 
      Effect.log(`Loading: ${isLoading ? 'Showing spinner' : 'Hiding spinner'}`)
    )
  )
  
  // Start all updaters
  yield* Effect.fork(navUpdater)
  yield* Effect.fork(userListUpdater)
  yield* Effect.fork(notificationUpdater)
  yield* Effect.fork(loadingUpdater)
  
  // Simulate app lifecycle
  yield* appState.setLoading(true)
  yield* appState.setCurrentUser({ 
    id: '1', 
    name: 'Alice Smith', 
    email: 'alice@example.com', 
    isOnline: true 
  })
  yield* appState.addUsers([
    { id: '2', name: 'Bob Jones', email: 'bob@example.com', isOnline: true },
    { id: '3', name: 'Carol White', email: 'carol@example.com', isOnline: false }
  ])
  yield* appState.addNotification('Welcome to the application!')
  yield* appState.updateUserStatus('3', true)
  yield* appState.setLoading(false)
  
  yield* Effect.sleep("100 millis")
})
```

### Example 3: Event-Driven Game State

```typescript
interface Player {
  readonly id: string
  readonly name: string
  readonly score: number
  readonly position: { x: number; y: number }
  readonly health: number
  readonly isAlive: boolean
}

interface GameState {
  readonly players: ReadonlyArray<Player>
  readonly gameStatus: 'waiting' | 'playing' | 'paused' | 'finished'
  readonly currentRound: number
  readonly timeRemaining: number
}

const makeGameStateManager = Effect.gen(function* () {
  const gameRef = yield* SubscriptionRef.make<GameState>({
    players: [],
    gameStatus: 'waiting',
    currentRound: 1,
    timeRemaining: 0
  })
  
  // Player actions
  const addPlayer = (player: Player) =>
    SubscriptionRef.update(gameRef, state => ({
      ...state,
      players: [...state.players, player]
    }))
  
  const updatePlayerScore = (playerId: string, points: number) =>
    SubscriptionRef.update(gameRef, state => ({
      ...state,
      players: state.players.map(player =>
        player.id === playerId 
          ? { ...player, score: player.score + points }
          : player
      )
    }))
  
  const movePlayer = (playerId: string, position: { x: number; y: number }) =>
    SubscriptionRef.update(gameRef, state => ({
      ...state,
      players: state.players.map(player =>
        player.id === playerId ? { ...player, position } : player
      )
    }))
  
  const damagePlayer = (playerId: string, damage: number) =>
    SubscriptionRef.update(gameRef, state => ({
      ...state,
      players: state.players.map(player => {
        if (player.id !== playerId) return player
        
        const newHealth = Math.max(0, player.health - damage)
        return {
          ...player,
          health: newHealth,
          isAlive: newHealth > 0
        }
      })
    }))
  
  // Game control
  const startGame = () =>
    SubscriptionRef.update(gameRef, state => ({
      ...state,
      gameStatus: 'playing',
      timeRemaining: 300 // 5 minutes
    }))
  
  const pauseGame = () =>
    SubscriptionRef.update(gameRef, state => ({
      ...state,
      gameStatus: 'paused'
    }))
  
  const nextRound = () =>
    SubscriptionRef.update(gameRef, state => ({
      ...state,
      currentRound: state.currentRound + 1,
      timeRemaining: 300,
      players: state.players.map(player => ({
        ...player,
        health: 100,
        isAlive: true
      }))
    }))
  
  const decrementTime = () =>
    SubscriptionRef.update(gameRef, state => {
      const newTime = Math.max(0, state.timeRemaining - 1)
      return {
        ...state,
        timeRemaining: newTime,
        gameStatus: newTime === 0 ? 'finished' : state.gameStatus
      }
    })
  
  // Reactive streams for different game aspects
  const scoresStream = gameRef.changes.pipe(
    Stream.map(state => 
      state.players
        .slice()
        .sort((a, b) => b.score - a.score)
        .slice(0, 3) // Top 3 players
    )
  )
  
  const alivePlayersStream = gameRef.changes.pipe(
    Stream.map(state => state.players.filter(p => p.isAlive))
  )
  
  const gameStatusStream = gameRef.changes.pipe(
    Stream.map(state => ({
      status: state.gameStatus,
      round: state.currentRound,
      timeRemaining: state.timeRemaining
    }))
  )
  
  return {
    addPlayer,
    updatePlayerScore,
    movePlayer,
    damagePlayer,
    startGame,
    pauseGame,
    nextRound,
    decrementTime,
    get: () => SubscriptionRef.get(gameRef),
    changes: gameRef.changes,
    scoresStream,
    alivePlayersStream,
    gameStatusStream
  }
})

// Game systems reacting to state changes
const gameExample = Effect.gen(function* () {
  const game = yield* makeGameStateManager
  
  // Scoreboard system
  const scoreboardUpdater = game.scoresStream.pipe(
    Stream.runForEach(topPlayers => 
      Effect.log(`Scoreboard: Top players - ${topPlayers.map(p => `${p.name}: ${p.score}`).join(', ')}`)
    )
  )
  
  // Game UI system
  const uiUpdater = game.gameStatusStream.pipe(
    Stream.runForEach(status => 
      Effect.log(`UI: ${status.status} - Round ${status.round} - ${status.timeRemaining}s remaining`)
    )
  )
  
  // Win condition checker
  const winChecker = game.alivePlayersStream.pipe(
    Stream.runForEach(alivePlayers => {
      if (alivePlayers.length <= 1 && alivePlayers.length > 0) {
        return Effect.log(`Game Over: ${alivePlayers[0].name} wins!`)
      }
      return Effect.log(`Game: ${alivePlayers.length} players still alive`)
    })
  )
  
  // Start all systems
  yield* Effect.fork(scoreboardUpdater)
  yield* Effect.fork(uiUpdater)
  yield* Effect.fork(winChecker)
  
  // Simulate game events
  yield* game.addPlayer({ 
    id: '1', 
    name: 'Player 1', 
    score: 0, 
    position: { x: 0, y: 0 }, 
    health: 100, 
    isAlive: true 
  })
  yield* game.addPlayer({ 
    id: '2', 
    name: 'Player 2', 
    score: 0, 
    position: { x: 10, y: 10 }, 
    health: 100, 
    isAlive: true 
  })
  
  yield* game.startGame()
  yield* game.updatePlayerScore('1', 100)
  yield* game.updatePlayerScore('2', 150)
  yield* game.damagePlayer('2', 80)
  yield* game.movePlayer('1', { x: 5, y: 5 })
  
  yield* Effect.sleep("100 millis")
})
```

## Advanced Features Deep Dive

### Feature 1: Atomic Modifications with Effects

SubscriptionRef provides atomic operations that can perform effectful computations during state updates.

#### Basic Atomic Modifications

```typescript
import { SubscriptionRef, Effect, Random } from "effect"

interface Counter {
  readonly value: number
  readonly lastModified: Date
  readonly modificationCount: number
}

const makeAdvancedCounter = Effect.gen(function* () {
  const counterRef = yield* SubscriptionRef.make<Counter>({
    value: 0,
    lastModified: new Date(),
    modificationCount: 0
  })
  
  // Simple atomic update
  const increment = () =>
    SubscriptionRef.update(counterRef, counter => ({
      ...counter,
      value: counter.value + 1,
      lastModified: new Date(),
      modificationCount: counter.modificationCount + 1
    }))
  
  // Atomic modification with return value
  const incrementAndGet = () =>
    SubscriptionRef.modify(counterRef, counter => {
      const newCounter = {
        ...counter,
        value: counter.value + 1,
        lastModified: new Date(),
        modificationCount: counter.modificationCount + 1
      }
      return [newCounter.value, newCounter] // [returnValue, newState]
    })
  
  return { counterRef, increment, incrementAndGet }
})
```

#### Effectful Atomic Operations

```typescript
interface UserSession {
  readonly userId: string
  readonly lastActivity: Date
  readonly activityCount: number
  readonly isValid: boolean
}

const makeSessionManager = Effect.gen(function* () {
  const sessionRef = yield* SubscriptionRef.make<UserSession>({
    userId: '',
    lastActivity: new Date(),
    activityCount: 0,
    isValid: false
  })
  
  // Effectful modification - validation during update
  const recordActivity = (userId: string) =>
    SubscriptionRef.modifyEffect(sessionRef, session => 
      Effect.gen(function* () {
        // Perform effectful validation
        const isValidUser = yield* validateUser(userId)
        const currentTime = yield* Effect.sync(() => new Date())
        
        if (!isValidUser) {
          return [false, { ...session, isValid: false }] as const
        }
        
        const newSession = {
          userId,
          lastActivity: currentTime,
          activityCount: session.activityCount + 1,
          isValid: true
        }
        
        // Log the activity (effectful side operation)
        yield* Effect.log(`Activity recorded for user ${userId}`)
        
        return [true, newSession] as const
      })
    )
  
  // Conditional effectful update
  const refreshIfExpired = () =>
    SubscriptionRef.updateEffect(sessionRef, session =>
      Effect.gen(function* () {
        const now = yield* Effect.sync(() => new Date())
        const timeDiff = now.getTime() - session.lastActivity.getTime()
        const isExpired = timeDiff > 30000 // 30 seconds
        
        if (!isExpired) {
          return session // No change needed
        }
        
        // Session expired, refresh or invalidate
        const canRefresh = yield* checkRefreshToken(session.userId)
        
        if (canRefresh) {
          yield* Effect.log(`Session refreshed for user ${session.userId}`)
          return {
            ...session,
            lastActivity: now,
            isValid: true
          }
        } else {
          yield* Effect.log(`Session invalidated for user ${session.userId}`)
          return {
            ...session,
            isValid: false
          }
        }
      })
    )
  
  return { sessionRef, recordActivity, refreshIfExpired }
})

// Helper functions
const validateUser = (userId: string): Effect.Effect<boolean> =>
  Effect.gen(function* () {
    // Simulate async validation
    yield* Effect.sleep("10 millis")
    return userId.length > 0 && !userId.includes('invalid')
  })

const checkRefreshToken = (userId: string): Effect.Effect<boolean> =>
  Effect.gen(function* () {
    // Simulate token validation
    yield* Effect.sleep("5 millis")
    return Math.random() > 0.5 // 50% chance of successful refresh
  })
```

### Feature 2: Advanced Stream Operations on Changes

The changes stream can be composed with powerful Stream operators for complex reactive patterns.

#### Debouncing and Throttling

```typescript
import { Stream, Schedule } from "effect"

const makeSearchState = Effect.gen(function* () {
  const searchRef = yield* SubscriptionRef.make('')
  
  // Debounced search - only emit after user stops typing for 300ms
  const debouncedSearches = searchRef.changes.pipe(
    Stream.filter(query => query.length > 2), // Only search queries > 2 chars
    Stream.debounce("300 millis")
  )
  
  // Throttled updates - at most one update per second
  const throttledUpdates = searchRef.changes.pipe(
    Stream.throttle({ cost: 1, duration: "1 second", units: 1 })
  )
  
  const setQuery = (query: string) => SubscriptionRef.set(searchRef, query)
  
  return { setQuery, debouncedSearches, throttledUpdates }
})

// Usage
const searchExample = Effect.gen(function* () {
  const search = yield* makeSearchState
  
  const debouncedSearchHandler = search.debouncedSearches.pipe(
    Stream.runForEach(query => 
      Effect.log(`Executing search for: "${query}"`)
    )
  )
  
  yield* Effect.fork(debouncedSearchHandler)
  
  // Simulate rapid typing
  yield* search.setQuery('a')
  yield* search.setQuery('ap')
  yield* search.setQuery('app')
  yield* search.setQuery('appl') 
  yield* search.setQuery('apple') // Only this will trigger search after debounce
  
  yield* Effect.sleep("500 millis")
})
```

#### Stream Composition and Merging

```typescript
interface MetricsState {
  readonly requests: number
  readonly errors: number
  readonly avgResponseTime: number
}

const makeMetricsManager = Effect.gen(function* () {
  const metricsRef = yield* SubscriptionRef.make<MetricsState>({
    requests: 0,
    errors: 0,
    avgResponseTime: 0
  })
  
  // Different aspects of metrics as separate streams
  const errorRateStream = metricsRef.changes.pipe(
    Stream.map(metrics => 
      metrics.requests > 0 ? (metrics.errors / metrics.requests) * 100 : 0
    ),
    Stream.changes // Only emit when error rate actually changes
  )
  
  const performanceStream = metricsRef.changes.pipe(
    Stream.map(metrics => ({
      requests: metrics.requests,
      avgResponseTime: metrics.avgResponseTime
    })),
    Stream.changes
  )
  
  // Combined health stream
  const healthStream = Stream.merge(
    errorRateStream.pipe(Stream.map(rate => ({ type: 'error_rate' as const, value: rate }))),
    performanceStream.pipe(Stream.map(perf => ({ type: 'performance' as const, value: perf })))
  )
  
  // Alert stream - only critical conditions
  const alertStream = metricsRef.changes.pipe(
    Stream.filter(metrics => 
      metrics.errors / metrics.requests > 0.05 || // > 5% error rate
      metrics.avgResponseTime > 1000 // > 1 second response time
    ),
    Stream.map(metrics => ({
      message: `ALERT: ${Math.round((metrics.errors / metrics.requests) * 100)}% error rate, ${metrics.avgResponseTime}ms avg response time`,
      severity: metrics.errors / metrics.requests > 0.1 ? 'critical' : 'warning'
    }))
  )
  
  const recordRequest = (responseTime: number, isError = false) =>
    SubscriptionRef.update(metricsRef, metrics => {
      const newRequests = metrics.requests + 1
      const newErrors = metrics.errors + (isError ? 1 : 0)
      const newAvgResponseTime = 
        (metrics.avgResponseTime * metrics.requests + responseTime) / newRequests
      
      return {
        requests: newRequests,
        errors: newErrors,
        avgResponseTime: Math.round(newAvgResponseTime)
      }
    })
  
  return { 
    recordRequest, 
    errorRateStream, 
    performanceStream, 
    healthStream, 
    alertStream,
    get: () => SubscriptionRef.get(metricsRef)
  }
})

const metricsExample = Effect.gen(function* () {
  const metrics = yield* makeMetricsManager
  
  // Health monitoring system
  const healthMonitor = metrics.healthStream.pipe(
    Stream.runForEach(event => 
      Effect.log(`Health: ${event.type} - ${JSON.stringify(event.value)}`)
    )
  )
  
  // Alert system
  const alertSystem = metrics.alertStream.pipe(
    Stream.runForEach(alert => 
      Effect.log(`🚨 ${alert.severity.toUpperCase()}: ${alert.message}`)
    )
  )
  
  yield* Effect.fork(healthMonitor)
  yield* Effect.fork(alertSystem)
  
  // Simulate traffic with some errors
  yield* metrics.recordRequest(200, false)
  yield* metrics.recordRequest(150, false)
  yield* metrics.recordRequest(1200, true) // Slow + error
  yield* metrics.recordRequest(300, false)
  yield* metrics.recordRequest(2000, true) // Very slow + error
  
  yield* Effect.sleep("100 millis")
})
```

### Feature 3: Resource-Safe Subscriptions

SubscriptionRef integrates with Effect's resource management for automatic cleanup.

#### Scoped Subscriptions

```typescript
import { Scope } from "effect"

const makeManagedSubscription = <A>(
  subscriptionRef: SubscriptionRef.SubscriptionRef<A>,
  handler: (value: A) => Effect.Effect<void>
) => 
  Effect.gen(function* () {
    // Create subscription within a scope for automatic cleanup
    const subscription = subscriptionRef.changes.pipe(
      Stream.runForEach(handler)
    )
    
    // Fork the subscription - it will be cleaned up when scope closes
    const fiber = yield* Effect.fork(subscription)
    
    // Return cleanup function
    return () => Fiber.interrupt(fiber)
  })

// Usage with automatic cleanup
const scopedSubscriptionExample = Effect.gen(function* () {
  const dataRef = yield* SubscriptionRef.make({ count: 0, message: '' })
  
  // Create a scoped subscription
  yield* Effect.scoped(
    Effect.gen(function* () {
      const cleanup = yield* makeManagedSubscription(
        dataRef,
        data => Effect.log(`Scoped handler: ${JSON.stringify(data)}`)
      )
      
      // Make some updates
      yield* SubscriptionRef.update(dataRef, d => ({ ...d, count: d.count + 1 }))
      yield* SubscriptionRef.update(dataRef, d => ({ ...d, message: 'hello' }))
      
      yield* Effect.sleep("50 millis")
      
      // Subscription automatically cleaned up when leaving this scope
    })
  )
  
  // This update won't be handled by the scoped subscription
  yield* SubscriptionRef.update(dataRef, d => ({ ...d, count: d.count + 1 }))
  yield* Effect.log("Update after scope - no handler called")
})
```

#### Multiple Subscription Management

```typescript
interface ConnectionManager {
  readonly connections: ReadonlyArray<{ id: string; status: 'connected' | 'disconnected' }>
  readonly totalConnections: number
  readonly activeConnections: number
}

const makeConnectionManager = Effect.gen(function* () {
  const connectionsRef = yield* SubscriptionRef.make<ConnectionManager>({
    connections: [],
    totalConnections: 0,
    activeConnections: 0
  })
  
  const addConnection = (id: string) =>
    SubscriptionRef.update(connectionsRef, state => {
      const newConnections = [...state.connections, { id, status: 'connected' as const }]
      return {
        connections: newConnections,
        totalConnections: newConnections.length,
        activeConnections: newConnections.filter(c => c.status === 'connected').length
      }
    })
  
  const updateConnectionStatus = (id: string, status: 'connected' | 'disconnected') =>
    SubscriptionRef.update(connectionsRef, state => {
      const newConnections = state.connections.map(conn =>
        conn.id === id ? { ...conn, status } : conn
      )
      return {
        connections: newConnections,
        totalConnections: newConnections.length,
        activeConnections: newConnections.filter(c => c.status === 'connected').length
      }
    })
  
  // Create multiple managed subscriptions
  const createSubscriptions = Effect.gen(function* () {
    // Logging subscription
    const loggingFiber = yield* Effect.fork(
      connectionsRef.changes.pipe(
        Stream.runForEach(state => 
          Effect.log(`Connections: ${state.activeConnections}/${state.totalConnections} active`)
        )
      )
    )
    
    // Monitoring subscription
    const monitoringFiber = yield* Effect.fork(
      connectionsRef.changes.pipe(
        Stream.map(state => state.activeConnections),
        Stream.changes,
        Stream.runForEach(activeCount => 
          Effect.log(`Monitor: Active connections changed to ${activeCount}`)
        )
      )
    )
    
    // Alert subscription for high connection count
    const alertFiber = yield* Effect.fork(
      connectionsRef.changes.pipe(
        Stream.filter(state => state.activeConnections > 100),
        Stream.runForEach(state => 
          Effect.log(`🚨 HIGH CONNECTION COUNT: ${state.activeConnections} active connections`)
        )
      )
    )
    
    // Return cleanup function that stops all subscriptions
    return () => Effect.all([
      Fiber.interrupt(loggingFiber),
      Fiber.interrupt(monitoringFiber),
      Fiber.interrupt(alertFiber)
    ], { concurrency: "unbounded" })
  })
  
  return { addConnection, updateConnectionStatus, createSubscriptions }
})
```

## Practical Patterns & Best Practices

### Pattern 1: State Normalization Helper

```typescript
// Helper for managing normalized state (like Redux pattern)
const makeNormalizedStore = <T extends { id: string }>(initialItems: ReadonlyArray<T> = []) =>
  Effect.gen(function* () {
    interface NormalizedState<T> {
      readonly byId: Record<string, T>
      readonly allIds: ReadonlyArray<string>
    }
    
    const normalize = (items: ReadonlyArray<T>): NormalizedState<T> => ({
      byId: items.reduce((acc, item) => ({ ...acc, [item.id]: item }), {}),
      allIds: items.map(item => item.id)
    })
    
    const stateRef = yield* SubscriptionRef.make(normalize(initialItems))
    
    const add = (item: T) =>
      SubscriptionRef.update(stateRef, state => ({
        byId: { ...state.byId, [item.id]: item },
        allIds: state.allIds.includes(item.id) ? state.allIds : [...state.allIds, item.id]
      }))
    
    const update = (id: string, updater: (item: T) => T) =>
      SubscriptionRef.update(stateRef, state => {
        const item = state.byId[id]
        if (!item) return state
        
        return {
          ...state,
          byId: { ...state.byId, [id]: updater(item) }
        }
      })
    
    const remove = (id: string) =>
      SubscriptionRef.update(stateRef, state => {
        const { [id]: removed, ...remainingById } = state.byId
        return {
          byId: remainingById,
          allIds: state.allIds.filter(existingId => existingId !== id)
        }
      })
    
    const addMany = (items: ReadonlyArray<T>) =>
      SubscriptionRef.update(stateRef, state => {
        const newById = items.reduce((acc, item) => ({ ...acc, [item.id]: item }), state.byId)
        const newIds = items.map(item => item.id).filter(id => !state.allIds.includes(id))
        
        return {
          byId: newById,
          allIds: [...state.allIds, ...newIds]
        }
      })
    
    // Reactive selectors
    const allItemsStream = stateRef.changes.pipe(
      Stream.map(state => state.allIds.map(id => state.byId[id]))
    )
    
    const itemByIdStream = (id: string) => stateRef.changes.pipe(
      Stream.map(state => state.byId[id] || null),
      Stream.changes
    )
    
    const filteredItemsStream = (predicate: (item: T) => boolean) => 
      allItemsStream.pipe(
        Stream.map(items => items.filter(predicate))
      )
    
    return {
      add,
      update,
      remove,
      addMany,
      get: () => SubscriptionRef.get(stateRef),
      allItemsStream,
      itemByIdStream,
      filteredItemsStream
    }
  })

// Usage example with User entities
interface User {
  readonly id: string
  readonly name: string
  readonly email: string
  readonly active: boolean
}

const userStoreExample = Effect.gen(function* () {
  const userStore = yield* makeNormalizedStore<User>([
    { id: '1', name: 'Alice', email: 'alice@example.com', active: true },
    { id: '2', name: 'Bob', email: 'bob@example.com', active: false }
  ])
  
  // Subscribe to active users only
  const activeUsersUpdater = userStore.filteredItemsStream(user => user.active).pipe(
    Stream.runForEach(activeUsers => 
      Effect.log(`Active users: ${activeUsers.map(u => u.name).join(', ')}`)
    )
  )
  
  // Subscribe to specific user changes
  const aliceUpdater = userStore.itemByIdStream('1').pipe(
    Stream.runForEach(alice => 
      Effect.log(`Alice update: ${alice ? JSON.stringify(alice) : 'User removed'}`)
    )
  )
  
  yield* Effect.fork(activeUsersUpdater)
  yield* Effect.fork(aliceUpdater)
  
  // Make changes
  yield* userStore.update('2', user => ({ ...user, active: true }))
  yield* userStore.add({ id: '3', name: 'Carol', email: 'carol@example.com', active: true })
  yield* userStore.update('1', user => ({ ...user, name: 'Alice Smith' }))
  
  yield* Effect.sleep("100 millis")
})
```

### Pattern 2: Composite State Manager

```typescript
// Pattern for managing multiple related SubscriptionRefs
const makeCompositeState = Effect.gen(function* () {
  // Individual state pieces
  const userRef = yield* SubscriptionRef.make<{ id: string; name: string } | null>(null)
  const settingsRef = yield* SubscriptionRef.make({ theme: 'light', language: 'en' })
  const notificationsRef = yield* SubscriptionRef.make<ReadonlyArray<string>>([])
  
  // Composite stream that combines all state
  const compositeStream = Stream.combineLatest(
    userRef.changes,
    settingsRef.changes,
    notificationsRef.changes
  ).pipe(
    Stream.map(([user, settings, notifications]) => ({
      user,
      settings,
      notifications,
      isLoggedIn: user !== null,
      notificationCount: notifications.length
    }))
  )
  
  // Actions that may affect multiple pieces of state
  const login = (user: { id: string; name: string }) =>
    Effect.gen(function* () {
      yield* SubscriptionRef.set(userRef, user)
      yield* SubscriptionRef.update(notificationsRef, n => [...n, `Welcome back, ${user.name}!`])
    })
  
  const logout = () =>
    Effect.gen(function* () {
      yield* SubscriptionRef.set(userRef, null)
      yield* SubscriptionRef.set(notificationsRef, [])
    })
  
  const updateSettings = (newSettings: { theme: string; language: string }) =>
    Effect.gen(function* () {
      yield* SubscriptionRef.set(settingsRef, newSettings)
      yield* SubscriptionRef.update(notificationsRef, n => [...n, 'Settings updated'])
    })
  
  return {
    login,
    logout,
    updateSettings,
    compositeStream,
    userStream: userRef.changes,
    settingsStream: settingsRef.changes,
    notificationsStream: notificationsRef.changes
  }
})

const compositeExample = Effect.gen(function* () {
  const state = yield* makeCompositeState
  
  // Subscribe to composite state
  const compositeUpdater = state.compositeStream.pipe(
    Stream.runForEach(combined => 
      Effect.log(`App State: ${combined.isLoggedIn ? `Logged in as ${combined.user!.name}` : 'Not logged in'}, ${combined.notificationCount} notifications`)
    )
  )
  
  yield* Effect.fork(compositeUpdater)
  
  yield* state.login({ id: '1', name: 'Alice' })
  yield* state.updateSettings({ theme: 'dark', language: 'es' })
  yield* state.logout()
  
  yield* Effect.sleep("100 millis")
})
```

### Pattern 3: Subscription Lifecycle Management

```typescript
// Helper for managing subscription lifecycles with cleanup
const makeSubscriptionManager = <A>(
  subscriptionRef: SubscriptionRef.SubscriptionRef<A>
) => 
  Effect.gen(function* () {
    const activeSubscriptions = yield* Ref.make<Map<string, Fiber.Fiber<void>>>(new Map())
    
    const subscribe = (
      id: string, 
      handler: (value: A) => Effect.Effect<void>,
      options?: {
        filter?: (value: A) => boolean
        transform?: (value: A) => A
      }
    ) =>
      Effect.gen(function* () {
        // Cancel existing subscription with same ID
        yield* unsubscribe(id)
        
        let stream = subscriptionRef.changes
        
        if (options?.filter) {
          stream = stream.pipe(Stream.filter(options.filter))
        }
        
        if (options?.transform) {
          stream = stream.pipe(Stream.map(options.transform))
        }
        
        const fiber = yield* Effect.fork(
          stream.pipe(Stream.runForEach(handler))
        )
        
        yield* Ref.update(activeSubscriptions, subs => 
          new Map(subs).set(id, fiber)
        )
        
        return id
      })
    
    const unsubscribe = (id: string) =>
      Effect.gen(function* () {
        const subs = yield* Ref.get(activeSubscriptions)
        const fiber = subs.get(id)
        
        if (fiber) {
          yield* Fiber.interrupt(fiber)
          yield* Ref.update(activeSubscriptions, subs => {
            const newSubs = new Map(subs)
            newSubs.delete(id)
            return newSubs
          })
        }
      })
    
    const unsubscribeAll = () =>
      Effect.gen(function* () {
        const subs = yield* Ref.get(activeSubscriptions)
        yield* Effect.forEach(
          Array.from(subs.values()),
          fiber => Fiber.interrupt(fiber),
          { concurrency: "unbounded" }
        )
        yield* Ref.set(activeSubscriptions, new Map())
      })
    
    const getActiveSubscriptions = () =>
      Ref.get(activeSubscriptions).pipe(
        Effect.map(subs => Array.from(subs.keys()))
      )
    
    return { subscribe, unsubscribe, unsubscribeAll, getActiveSubscriptions }
  })

const subscriptionManagerExample = Effect.gen(function* () {
  const dataRef = yield* SubscriptionRef.make({ count: 0, message: '' })
  const manager = yield* makeSubscriptionManager(dataRef)
  
  // Subscribe with different filters and transformations
  yield* manager.subscribe(
    'counter-logger',
    data => Effect.log(`Count: ${data.count}`)
  )
  
  yield* manager.subscribe(
    'even-counter',
    data => Effect.log(`Even count: ${data.count}`),
    { filter: data => data.count % 2 === 0 }
  )
  
  yield* manager.subscribe(
    'message-logger',
    data => Effect.log(`Message: ${data.message}`),
    { 
      filter: data => data.message.length > 0,
      transform: data => ({ ...data, message: data.message.toUpperCase() })
    }
  )
  
  // Make updates
  yield* SubscriptionRef.update(dataRef, d => ({ ...d, count: 1 }))
  yield* SubscriptionRef.update(dataRef, d => ({ ...d, count: 2 })) // Triggers even-counter
  yield* SubscriptionRef.update(dataRef, d => ({ ...d, message: 'hello' }))
  yield* SubscriptionRef.update(dataRef, d => ({ ...d, count: 3 }))
  
  yield* Effect.sleep("50 millis")
  
  // Check active subscriptions
  const active = yield* manager.getActiveSubscriptions()
  yield* Effect.log(`Active subscriptions: ${active.join(', ')}`)
  
  // Clean up specific subscription
  yield* manager.unsubscribe('even-counter')
  
  yield* SubscriptionRef.update(dataRef, d => ({ ...d, count: 4 })) // Won't trigger even-counter
  
  yield* Effect.sleep("50 millis")
  
  // Clean up all subscriptions
  yield* manager.unsubscribeAll()
})
```

## Integration Examples

### Integration with Web Framework (Express-like)

```typescript
import { SubscriptionRef, Effect, Layer, Context } from "effect"

// Application state service
interface AppStateService {
  readonly state: SubscriptionRef.SubscriptionRef<{
    readonly requestCount: number
    readonly activeUsers: number
    readonly systemHealth: 'healthy' | 'degraded' | 'down'
  }>
  readonly incrementRequests: () => Effect.Effect<void>
  readonly updateActiveUsers: (count: number) => Effect.Effect<void>
  readonly updateHealth: (health: 'healthy' | 'degraded' | 'down') => Effect.Effect<void>
}

const AppStateService = Context.GenericTag<AppStateService>('AppStateService')

const makeAppStateService = Effect.gen(function* () {
  const state = yield* SubscriptionRef.make({
    requestCount: 0,
    activeUsers: 0,
    systemHealth: 'healthy' as const
  })
  
  const incrementRequests = () =>
    SubscriptionRef.update(state, s => ({ ...s, requestCount: s.requestCount + 1 }))
  
  const updateActiveUsers = (count: number) =>
    SubscriptionRef.update(state, s => ({ ...s, activeUsers: count }))
  
  const updateHealth = (health: 'healthy' | 'degraded' | 'down') =>
    SubscriptionRef.update(state, s => ({ ...s, systemHealth: health }))
  
  return { state, incrementRequests, updateActiveUsers, updateHealth }
})

const AppStateServiceLive = Layer.effect(AppStateService, makeAppStateService)

// WebSocket service that streams state to clients
interface WebSocketService {
  readonly broadcastState: () => Effect.Effect<void>
  readonly addClient: (clientId: string) => Effect.Effect<void>
  readonly removeClient: (clientId: string) => Effect.Effect<void>
}

const WebSocketService = Context.GenericTag<WebSocketService>('WebSocketService')

const makeWebSocketService = Effect.gen(function* () {
  const appState = yield* AppStateService
  const clients = yield* Ref.make<Set<string>>(new Set())
  
  // Stream state changes to all connected clients
  const stateStreamer = appState.state.changes.pipe(
    Stream.runForEach(state => 
      Effect.gen(function* () {
        const clientSet = yield* Ref.get(clients)
        yield* Effect.log(`Broadcasting to ${clientSet.size} clients: ${JSON.stringify(state)}`)
        // In real implementation, this would send via WebSocket
      })
    )
  )
  
  // Start the state streaming in background
  yield* Effect.fork(stateStreamer)
  
  const broadcastState = () => Effect.unit
  
  const addClient = (clientId: string) =>
    Effect.gen(function* () {
      yield* Ref.update(clients, set => new Set(set).add(clientId))
      const clientCount = yield* Ref.get(clients).pipe(Effect.map(set => set.size))
      yield* appState.updateActiveUsers(clientCount)
      yield* Effect.log(`Client ${clientId} connected. Total: ${clientCount}`)
    })
  
  const removeClient = (clientId: string) =>
    Effect.gen(function* () {
      yield* Ref.update(clients, set => {
        const newSet = new Set(set)
        newSet.delete(clientId)
        return newSet
      })
      const clientCount = yield* Ref.get(clients).pipe(Effect.map(set => set.size))
      yield* appState.updateActiveUsers(clientCount)
      yield* Effect.log(`Client ${clientId} disconnected. Total: ${clientCount}`)
    })
  
  return { broadcastState, addClient, removeClient }
})

const WebSocketServiceLive = Layer.effect(WebSocketService, makeWebSocketService)

// Express-like middleware
const requestCounterMiddleware = Effect.gen(function* () {
  const appState = yield* AppStateService
  yield* appState.incrementRequests()
})

// Health check endpoint  
const healthCheckEndpoint = Effect.gen(function* () {
  const appState = yield* AppStateService
  const currentState = yield* SubscriptionRef.get(appState.state)
  
  return {
    status: currentState.systemHealth,
    requestCount: currentState.requestCount,
    activeUsers: currentState.activeUsers,
    timestamp: new Date().toISOString()
  }
})

// Application setup
const webAppExample = Effect.gen(function* () {
  const appState = yield* AppStateService
  const webSocket = yield* WebSocketService
  
  // Simulate requests and connections
  yield* requestCounterMiddleware
  yield* requestCounterMiddleware
  yield* requestCounterMiddleware
  
  yield* webSocket.addClient('client-1')
  yield* webSocket.addClient('client-2')
  
  const healthCheck = yield* healthCheckEndpoint
  yield* Effect.log(`Health check: ${JSON.stringify(healthCheck)}`)
  
  yield* webSocket.removeClient('client-1')
  
  yield* Effect.sleep("100 millis")
}).pipe(
  Effect.provide(AppStateServiceLive),
  Effect.provide(WebSocketServiceLive)
)
```

### Integration with React-like UI Library

```typescript
// Simulated React-like hooks using SubscriptionRef
interface UIState {
  readonly theme: 'light' | 'dark'
  readonly user: { name: string; avatar: string } | null
  readonly notifications: ReadonlyArray<{ id: string; message: string; read: boolean }>
}

interface UIStore {
  readonly state: SubscriptionRef.SubscriptionRef<UIState>
  readonly setTheme: (theme: 'light' | 'dark') => Effect.Effect<void>
  readonly setUser: (user: { name: string; avatar: string } | null) => Effect.Effect<void>
  readonly addNotification: (message: string) => Effect.Effect<void>
  readonly markNotificationRead: (id: string) => Effect.Effect<void>
}

const makeUIStore = Effect.gen(function* () {
  const state = yield* SubscriptionRef.make<UIState>({
    theme: 'light',
    user: null,
    notifications: []
  })
  
  const setTheme = (theme: 'light' | 'dark') =>
    SubscriptionRef.update(state, s => ({ ...s, theme }))
  
  const setUser = (user: { name: string; avatar: string } | null) =>
    SubscriptionRef.update(state, s => ({ ...s, user }))
  
  const addNotification = (message: string) =>
    SubscriptionRef.update(state, s => ({
      ...s,
      notifications: [
        ...s.notifications,
        { id: Math.random().toString(36), message, read: false }
      ]
    }))
  
  const markNotificationRead = (id: string) =>
    SubscriptionRef.update(state, s => ({
      ...s,
      notifications: s.notifications.map(n =>
        n.id === id ? { ...n, read: true } : n
      )
    }))
  
  return { state, setTheme, setUser, addNotification, markNotificationRead }
})

// Simulated React-like components using SubscriptionRef
const makeHeaderComponent = (store: UIStore) =>
  Effect.gen(function* () {
    const userStream = store.state.changes.pipe(
      Stream.map(state => state.user),
      Stream.changes
    )
    
    const unreadCountStream = store.state.changes.pipe(
      Stream.map(state => state.notifications.filter(n => !n.read).length),
      Stream.changes
    )
    
    // Component lifecycle: subscribe to changes
    const userUpdater = userStream.pipe(
      Stream.runForEach(user => 
        Effect.log(`Header: User ${user ? `${user.name} logged in` : 'logged out'}`)
      )
    )
    
    const notificationUpdater = unreadCountStream.pipe(
      Stream.runForEach(count => 
        Effect.log(`Header: ${count} unread notifications`)
      )
    )
    
    yield* Effect.fork(userUpdater)
    yield* Effect.fork(notificationUpdater)
    
    return 'HeaderComponent'
  })

const makeThemeComponent = (store: UIStore) =>
  Effect.gen(function* () {
    const themeStream = store.state.changes.pipe(
      Stream.map(state => state.theme),
      Stream.changes
    )
    
    const themeUpdater = themeStream.pipe(
      Stream.runForEach(theme => 
        Effect.log(`Theme: Applied ${theme} theme to all components`)
      )
    )
    
    yield* Effect.fork(themeUpdater)
    
    return 'ThemeComponent'
  })

const makeNotificationCenter = (store: UIStore) =>
  Effect.gen(function* () {
    const notificationsStream = store.state.changes.pipe(
      Stream.map(state => state.notifications),
      Stream.changes
    )
    
    const notificationUpdater = notificationsStream.pipe(
      Stream.runForEach(notifications => 
        Effect.log(`Notifications: ${notifications.length} total, ${notifications.filter(n => !n.read).length} unread`)
      )
    )
    
    yield* Effect.fork(notificationUpdater)
    
    return 'NotificationCenter'
  })

// Simulated React app
const reactLikeApp = Effect.gen(function* () {
  const store = yield* makeUIStore
  
  // Initialize components
  const header = yield* makeHeaderComponent(store)
  const themeProvider = yield* makeThemeComponent(store)
  const notificationCenter = yield* makeNotificationCenter(store)
  
  yield* Effect.log(`App initialized with components: ${header}, ${themeProvider}, ${notificationCenter}`)
  
  // Simulate user interactions
  yield* store.setUser({ name: 'Alice Smith', avatar: 'avatar1.png' })
  yield* store.addNotification('Welcome to the application!')
  yield* store.setTheme('dark')
  yield* store.addNotification('Theme changed successfully')
  yield* store.markNotificationRead(
    (yield* SubscriptionRef.get(store.state)).notifications[0].id
  )
  
  yield* Effect.sleep("100 millis")
})
```

### Testing Strategies

```typescript
import { TestClock } from "effect/TestClock"

// Test utilities for SubscriptionRef
const makeTestableSubscriptionRef = <A>(initial: A) =>
  Effect.gen(function* () {
    const ref = yield* SubscriptionRef.make(initial)
    const events: Array<{ timestamp: number; value: A }> = []
    
    // Collect all changes for testing
    const collector = ref.changes.pipe(
      Stream.runForEach(value => 
        Effect.gen(function* () {
          const now = yield* Effect.sync(() => Date.now())
          events.push({ timestamp: now, value })
        })
      )
    )
    
    yield* Effect.fork(collector)
    
    const getEvents = () => Effect.succeed([...events])
    const clearEvents = () => Effect.sync(() => { events.length = 0 })
    
    return { ref, getEvents, clearEvents }
  })

// Test: Debounced updates
const testDebouncedUpdates = Effect.gen(function* () {
  const testRef = yield* makeTestableSubscriptionRef('')
  
  const debouncedStream = testRef.ref.changes.pipe(
    Stream.debounce("100 millis")
  )
  
  const debouncedEvents: Array<string> = []
  const debouncedCollector = debouncedStream.pipe(
    Stream.runForEach(value => 
      Effect.sync(() => { debouncedEvents.push(value) })
    )
  )
  
  yield* Effect.fork(debouncedCollector)
  
  // Rapid updates
  yield* SubscriptionRef.set(testRef.ref, 'a')
  yield* SubscriptionRef.set(testRef.ref, 'ab')
  yield* SubscriptionRef.set(testRef.ref, 'abc')
  
  // Wait for debounce
  yield* TestClock.adjust("150 millis")
  
  const events = yield* testRef.getEvents()
  
  // All changes recorded
  yield* Effect.log(`All events: ${events.length}`)
  // Only last debounced value
  yield* Effect.log(`Debounced events: ${debouncedEvents.length} - last value: ${debouncedEvents[debouncedEvents.length - 1]}`)
  
  // Assert: should have multiple raw events but only one debounced
  return {
    totalEvents: events.length,
    debouncedEvents: debouncedEvents.length,
    lastDebouncedValue: debouncedEvents[debouncedEvents.length - 1]
  }
})

// Test: State consistency under concurrent updates
const testConcurrentUpdates = Effect.gen(function* () {
  const counterRef = yield* SubscriptionRef.make(0)
  
  // Perform 100 concurrent increments
  const increments = Array.from({ length: 100 }, (_, i) =>
    SubscriptionRef.update(counterRef, n => n + 1)
  )
  
  yield* Effect.all(increments, { concurrency: "unbounded" })
  
  const finalValue = yield* SubscriptionRef.get(counterRef)
  
  // Should be exactly 100 despite concurrent access
  yield* Effect.log(`Final counter value: ${finalValue}`)
  
  return finalValue === 100
})

// Test: Memory leak prevention with subscriptions
const testSubscriptionCleanup = Effect.gen(function* () {
  const dataRef = yield* SubscriptionRef.make({ count: 0 })
  let subscriptionEvents = 0
  
  // Create a scoped subscription
  const result = yield* Effect.scoped(
    Effect.gen(function* () {
      const subscription = dataRef.changes.pipe(
        Stream.runForEach(_ => 
          Effect.sync(() => { subscriptionEvents++ })
        )
      )
      
      yield* Effect.fork(subscription)
      
      // Make updates within scope
      yield* SubscriptionRef.update(dataRef, d => ({ count: d.count + 1 }))
      yield* SubscriptionRef.update(dataRef, d => ({ count: d.count + 1 }))
      
      yield* Effect.sleep("10 millis")
      
      return subscriptionEvents
    })
  )
  
  // Make updates after scope
  yield* SubscriptionRef.update(dataRef, d => ({ count: d.count + 1 }))
  yield* Effect.sleep("10 millis")
  
  // Should have events from within scope but not after
  yield* Effect.log(`Events during scope: ${result}`)
  yield* Effect.log(`Total events after scope: ${subscriptionEvents}`)
  
  return { duringScope: result, afterScope: subscriptionEvents }
})

const testSuite = Effect.gen(function* () {
  yield* Effect.log("Running SubscriptionRef tests...")
  
  const debouncedTest = yield* testDebouncedUpdates
  yield* Effect.log(`Debounced test: ${JSON.stringify(debouncedTest)}`)
  
  const concurrentTest = yield* testConcurrentUpdates
  yield* Effect.log(`Concurrent test passed: ${concurrentTest}`)
  
  const cleanupTest = yield* testSubscriptionCleanup
  yield* Effect.log(`Cleanup test: ${JSON.stringify(cleanupTest)}`)
})
```

## Conclusion

SubscriptionRef provides reactive state management, automatic change notifications, and thread-safe concurrent access for building robust event-driven applications.

Key benefits:
- **Reactive State Management**: Automatic notifications to multiple observers when state changes
- **Thread Safety**: All operations are atomic and safe for concurrent access  
- **Resource Management**: Automatic cleanup prevents memory leaks and resource issues
- **Stream Integration**: Powerful composition with Effect's Stream ecosystem
- **Type Safety**: Full type inference and compile-time safety

SubscriptionRef is ideal when you need reactive state containers that multiple components can observe, such as application state management, real-time UI updates, event-driven architectures, and systems requiring automatic change propagation across loosely coupled components.