# MutableRef: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem MutableRef Solves

When building performance-critical applications, you sometimes need lightweight synchronous mutable references for local state management. Traditional JavaScript approaches and even Effect's Ref can be overkill for simple, local mutations:

```typescript
// Traditional approach - global mutable state
let requestCounter = 0;
let isProcessing = false;
let cacheStats = { hits: 0, misses: 0 };

function processRequest() {
  // Race conditions in concurrent scenarios
  requestCounter++;
  
  if (isProcessing) {
    return; // Check-then-act race condition
  }
  
  isProcessing = true;
  
  // Manual state management is error-prone
  try {
    // Process request...
    cacheStats.hits++;
  } finally {
    isProcessing = false; // Easy to forget cleanup
  }
}

// Using Effect.Ref - overkill for simple local state
import { Effect, Ref } from "effect";

const heavyweightApproach = Effect.gen(function* () {
  const counterRef = yield* Ref.make(0); // Creates Effect context
  const processingRef = yield* Ref.make(false); // More Effect overhead
  
  // Every operation requires Effect execution
  const count = yield* Ref.updateAndGet(counterRef, n => n + 1);
  const wasProcessing = yield* Ref.getAndSet(processingRef, true);
  
  // Complex for simple local mutations
});
```

This approach leads to:
- **Unnecessary Effect overhead** - Simple local state doesn't need Effect's concurrency machinery
- **Performance costs** - Ref operations create Effect computations even for synchronous mutations
- **Complexity overhead** - Effect.gen syntax is verbose for simple counter increments
- **Global state risks** - Plain variables lack encapsulation and type safety

### The MutableRef Solution

MutableRef provides lightweight, type-safe mutable references for synchronous local state without Effect overhead:

```typescript
import { MutableRef } from "effect";

// Lightweight synchronous mutable state
const requestCounter = MutableRef.make(0);
const isProcessing = MutableRef.make(false);
const cacheStats = MutableRef.make({ hits: 0, misses: 0 });

function processRequest() {
  // Atomic operations without Effect overhead
  const count = MutableRef.incrementAndGet(requestCounter);
  
  // Compare-and-set for safe state transitions
  if (!MutableRef.compareAndSet(isProcessing, false, true)) {
    return; // Already processing
  }
  
  try {
    // Process request...
    MutableRef.update(cacheStats, stats => ({ 
      ...stats, 
      hits: stats.hits + 1 
    }));
  } finally {
    MutableRef.set(isProcessing, false);
  }
}
```

### Key Concepts

**Synchronous Mutations**: All operations complete immediately without creating Effect computations

**Type Safety**: Full TypeScript support with compile-time type checking

**Local State Focus**: Designed for encapsulated mutable state within specific scopes

## Basic Usage Patterns

### Creating and Reading MutableRefs

```typescript
import { MutableRef } from "effect";

// Create MutableRefs with initial values
const counter = MutableRef.make(0);
const name = MutableRef.make("Initial");
const config = MutableRef.make({ theme: "dark", debug: false });
const items = MutableRef.make<string[]>([]);

// Read current values directly
const count = MutableRef.get(counter);
const currentName = MutableRef.get(name);
const currentConfig = MutableRef.get(config);

console.log(`Count: ${count}, Name: ${currentName}`);
// Count: 0, Name: Initial
```

### Basic Updates and Sets

```typescript
const ref = MutableRef.make(10);

// Set a new value
MutableRef.set(ref, 20);
console.log(MutableRef.get(ref)); // 20

// Update with a function
MutableRef.update(ref, n => n * 2);
console.log(MutableRef.get(ref)); // 40

// Set and get the new value atomically
const newValue = MutableRef.setAndGet(ref, 100);
console.log(newValue); // 100
```

### Atomic Get-and-Update Operations

```typescript
const ref = MutableRef.make(5);

// Get old value and set new one atomically
const oldValue = MutableRef.getAndSet(ref, 15);
console.log(oldValue); // 5
console.log(MutableRef.get(ref)); // 15

// Get old value and update atomically
const previousValue = MutableRef.getAndUpdate(ref, n => n + 5);
console.log(previousValue); // 15
console.log(MutableRef.get(ref)); // 20

// Update and get new value atomically
const currentValue = MutableRef.updateAndGet(ref, n => n * 2);
console.log(currentValue); // 40
```

## Real-World Examples

### Example 1: Performance Counter for Hot Paths

A high-performance counter for tracking operations in performance-critical code paths:

```typescript
import { MutableRef } from "effect";

interface PerformanceMetrics {
  readonly requestCount: number;
  readonly totalLatency: number;
  readonly errorCount: number;
  readonly activeRequests: number;
}

class RequestTracker {
  private metrics = MutableRef.make<PerformanceMetrics>({
    requestCount: 0,
    totalLatency: 0,
    errorCount: 0,
    activeRequests: 0
  });
  
  private processingFlags = new Map<string, MutableRef.MutableRef<boolean>>();
  
  startRequest(requestId: string): void {
    // Create per-request processing flag
    const processingFlag = MutableRef.make(true);
    this.processingFlags.set(requestId, processingFlag);
    
    // Increment active requests atomically
    MutableRef.update(this.metrics, m => ({
      ...m,
      requestCount: m.requestCount + 1,
      activeRequests: m.activeRequests + 1
    }));
  }
  
  completeRequest(requestId: string, latencyMs: number, hasError: boolean): void {
    const processingFlag = this.processingFlags.get(requestId);
    
    // Ensure request was actually started and is still processing
    if (processingFlag && MutableRef.compareAndSet(processingFlag, true, false)) {
      MutableRef.update(this.metrics, m => ({
        ...m,
        totalLatency: m.totalLatency + latencyMs,
        errorCount: hasError ? m.errorCount + 1 : m.errorCount,
        activeRequests: m.activeRequests - 1
      }));
      
      this.processingFlags.delete(requestId);
    }
  }
  
  getMetrics(): PerformanceMetrics {
    return MutableRef.get(this.metrics);
  }
  
  getAverageLatency(): number {
    const metrics = MutableRef.get(this.metrics);
    return metrics.requestCount > 0 
      ? metrics.totalLatency / metrics.requestCount 
      : 0;
  }
  
  isRequestProcessing(requestId: string): boolean {
    const flag = this.processingFlags.get(requestId);
    return flag ? MutableRef.get(flag) : false;
  }
}

// Usage in hot path
const tracker = new RequestTracker();

function handleRequest(requestId: string) {
  const startTime = Date.now();
  tracker.startRequest(requestId);
  
  try {
    // Process request (simulated)
    const result = processBusinessLogic();
    
    const latency = Date.now() - startTime;
    tracker.completeRequest(requestId, latency, false);
    
    return result;
  } catch (error) {
    const latency = Date.now() - startTime;
    tracker.completeRequest(requestId, latency, true);
    throw error;
  }
}

// Monitoring
setInterval(() => {
  const metrics = tracker.getMetrics();
  const avgLatency = tracker.getAverageLatency();
  
  console.log(`Active: ${metrics.activeRequests}, Total: ${metrics.requestCount}`);
  console.log(`Avg Latency: ${avgLatency.toFixed(2)}ms, Errors: ${metrics.errorCount}`);
}, 5000);
```

### Example 2: Local Cache with Statistics

A lightweight local cache that tracks hit rates and access patterns:

```typescript
import { MutableRef } from "effect";

interface CacheEntry<T> {
  readonly value: T;
  readonly timestamp: number;
  readonly accessCount: number;
}

interface CacheStats {
  readonly hits: number;
  readonly misses: number;
  readonly evictions: number;
  readonly totalSize: number;
}

class LocalCache<K, V> {
  private store = new Map<K, CacheEntry<V>>();
  private stats = MutableRef.make<CacheStats>({
    hits: 0,
    misses: 0, 
    evictions: 0,
    totalSize: 0
  });
  private maxSize = MutableRef.make(1000);
  private ttlMs = MutableRef.make(5 * 60 * 1000); // 5 minutes
  
  constructor(maxSize = 1000, ttlMs = 5 * 60 * 1000) {
    MutableRef.set(this.maxSize, maxSize);
    MutableRef.set(this.ttlMs, ttlMs);
  }
  
  get(key: K): V | undefined {
    const entry = this.store.get(key);
    const now = Date.now();
    const ttl = MutableRef.get(this.ttlMs);
    
    if (!entry) {
      // Cache miss
      MutableRef.update(this.stats, s => ({ ...s, misses: s.misses + 1 }));
      return undefined;
    }
    
    if (now - entry.timestamp > ttl) {
      // Expired entry
      this.store.delete(key);
      MutableRef.update(this.stats, s => ({ 
        ...s, 
        misses: s.misses + 1,
        totalSize: s.totalSize - 1
      }));
      return undefined;
    }
    
    // Cache hit - update access count
    const updatedEntry: CacheEntry<V> = {
      ...entry,
      accessCount: entry.accessCount + 1
    };
    this.store.set(key, updatedEntry);
    
    MutableRef.update(this.stats, s => ({ ...s, hits: s.hits + 1 }));
    return entry.value;
  }
  
  set(key: K, value: V): void {
    const now = Date.now();
    const currentMaxSize = MutableRef.get(this.maxSize);
    
    // Check if we need to evict
    if (this.store.size >= currentMaxSize && !this.store.has(key)) {
      this.evictLeastRecentlyUsed();
    }
    
    const isNewKey = !this.store.has(key);
    
    this.store.set(key, {
      value,
      timestamp: now,
      accessCount: 1
    });
    
    if (isNewKey) {
      MutableRef.update(this.stats, s => ({ 
        ...s, 
        totalSize: s.totalSize + 1 
      }));
    }
  }
  
  private evictLeastRecentlyUsed(): void {
    let lruKey: K | undefined;
    let lruTimestamp = Date.now();
    let lruAccessCount = Number.MAX_SAFE_INTEGER;
    
    for (const [key, entry] of this.store.entries()) {
      if (entry.accessCount < lruAccessCount || 
          (entry.accessCount === lruAccessCount && entry.timestamp < lruTimestamp)) {
        lruKey = key;
        lruTimestamp = entry.timestamp;
        lruAccessCount = entry.accessCount;
      }
    }
    
    if (lruKey !== undefined) {
      this.store.delete(lruKey);
      MutableRef.update(this.stats, s => ({ 
        ...s, 
        evictions: s.evictions + 1,
        totalSize: s.totalSize - 1
      }));
    }
  }
  
  getStats(): CacheStats {
    return MutableRef.get(this.stats);
  }
  
  getHitRatio(): number {
    const stats = MutableRef.get(this.stats);
    const total = stats.hits + stats.misses;
    return total > 0 ? stats.hits / total : 0;
  }
  
  resize(newMaxSize: number): void {
    const oldMaxSize = MutableRef.getAndSet(this.maxSize, newMaxSize);
    
    // If shrinking, evict excess entries
    while (this.store.size > newMaxSize) {
      this.evictLeastRecentlyUsed();
    }
  }
  
  clear(): void {
    const size = this.store.size;
    this.store.clear();
    
    MutableRef.update(this.stats, s => ({ 
      hits: 0,
      misses: 0,
      evictions: s.evictions,
      totalSize: 0
    }));
  }
}

// Usage
const userCache = new LocalCache<string, { name: string; email: string }>(500);

// Cache operations are synchronous and fast
userCache.set("user1", { name: "Alice", email: "alice@example.com" });
const user = userCache.get("user1"); // Fast synchronous lookup

// Monitor performance
const stats = userCache.getStats();
const hitRatio = userCache.getHitRatio();
console.log(`Hit ratio: ${(hitRatio * 100).toFixed(1)}%, Size: ${stats.totalSize}`);
```

### Example 3: State Machine with MutableRef

A finite state machine for managing component lifecycle states:

```typescript
import { MutableRef } from "effect";

type ComponentState = 
  | { readonly _tag: "idle" }
  | { readonly _tag: "loading"; readonly startTime: number }
  | { readonly _tag: "ready"; readonly data: unknown }
  | { readonly _tag: "error"; readonly error: string; readonly retryCount: number }
  | { readonly _tag: "disposed" };

interface StateTransition<From extends ComponentState["_tag"], To extends ComponentState["_tag"]> {
  readonly from: From;
  readonly to: To;
  readonly timestamp: number;
}

class ComponentStateMachine {
  private state = MutableRef.make<ComponentState>({ _tag: "idle" });
  private transitions = MutableRef.make<Array<StateTransition<any, any>>>([]);
  private stateChangeListeners = new Set<(state: ComponentState) => void>();
  
  getCurrentState(): ComponentState {
    return MutableRef.get(this.state);
  }
  
  getTransitionHistory(): ReadonlyArray<StateTransition<any, any>> {
    return MutableRef.get(this.transitions);
  }
  
  startLoading(): boolean {
    const currentState = MutableRef.get(this.state);
    
    // Only allow loading from idle or error states
    if (currentState._tag === "idle" || currentState._tag === "error") {
      const newState: ComponentState = { 
        _tag: "loading", 
        startTime: Date.now() 
      };
      
      this.transitionTo(newState);
      return true;
    }
    
    return false; // Invalid transition
  }
  
  setReady(data: unknown): boolean {
    const currentState = MutableRef.get(this.state);
    
    if (currentState._tag === "loading") {
      const newState: ComponentState = { 
        _tag: "ready", 
        data 
      };
      
      this.transitionTo(newState);
      return true;
    }
    
    return false;
  }
  
  setError(error: string): boolean {
    const currentState = MutableRef.get(this.state);
    
    if (currentState._tag === "loading") {
      const retryCount = currentState._tag === "error" ? currentState.retryCount + 1 : 0;
      const newState: ComponentState = { 
        _tag: "error", 
        error, 
        retryCount 
      };
      
      this.transitionTo(newState);
      return true;
    }
    
    return false;
  }
  
  reset(): boolean {
    const currentState = MutableRef.get(this.state);
    
    if (currentState._tag !== "disposed") {
      this.transitionTo({ _tag: "idle" });
      return true;
    }
    
    return false;
  }
  
  dispose(): void {
    const currentState = MutableRef.get(this.state);
    
    if (currentState._tag !== "disposed") {
      this.transitionTo({ _tag: "disposed" });
      this.stateChangeListeners.clear();
    }
  }
  
  private transitionTo(newState: ComponentState): void {
    const oldState = MutableRef.getAndSet(this.state, newState);
    
    // Record transition
    const transition: StateTransition<any, any> = {
      from: oldState._tag,
      to: newState._tag,
      timestamp: Date.now()
    };
    
    MutableRef.update(this.transitions, transitions => [...transitions, transition]);
    
    // Notify listeners
    this.stateChangeListeners.forEach(listener => {
      try {
        listener(newState);
      } catch (error) {
        console.error("State change listener error:", error);
      }
    });
  }
  
  onStateChange(listener: (state: ComponentState) => void): () => void {
    this.stateChangeListeners.add(listener);
    
    // Return cleanup function
    return () => {
      this.stateChangeListeners.delete(listener);
    };
  }
  
  isInState<T extends ComponentState["_tag"]>(tag: T): boolean {
    const currentState = MutableRef.get(this.state);
    return currentState._tag === tag;
  }
  
  getLoadingDuration(): number | null {
    const currentState = MutableRef.get(this.state);
    
    if (currentState._tag === "loading") {
      return Date.now() - currentState.startTime;
    }
    
    return null;
  }
}

// Usage
const stateMachine = new ComponentStateMachine();

// Listen to state changes
const unsubscribe = stateMachine.onStateChange(state => {
  console.log(`State changed to: ${state._tag}`);
  
  if (state._tag === "error") {
    console.log(`Error occurred: ${state.error} (retry #${state.retryCount})`);
  }
});

// State transitions
async function loadData() {
  if (stateMachine.startLoading()) {
    try {
      // Simulate async operation
      await new Promise(resolve => setTimeout(resolve, 1000));
      
      const data = { message: "Hello World" };
      stateMachine.setReady(data);
      
      console.log(`Loading took: ${stateMachine.getLoadingDuration()}ms`);
    } catch (error) {
      stateMachine.setError(error instanceof Error ? error.message : "Unknown error");
    }
  }
}

// Check current state
console.log(`Current state: ${stateMachine.getCurrentState()._tag}`);
console.log(`Is loading: ${stateMachine.isInState("loading")}`);

// Cleanup
// unsubscribe();
// stateMachine.dispose();
```

## Advanced Features Deep Dive

### Compare-and-Set Operations

Compare-and-set provides atomic conditional updates, crucial for lock-free programming:

```typescript
import { MutableRef } from "effect";

const makeCounter = (initialValue = 0) => {
  const ref = MutableRef.make(initialValue);
  
  return {
    // Increment only if current value matches expected
    incrementIfEquals: (expected: number): boolean => {
      return MutableRef.compareAndSet(ref, expected, expected + 1);
    },
    
    // Reset only if above threshold
    resetIfAbove: (threshold: number): boolean => {
      const current = MutableRef.get(ref);
      return current > threshold ? MutableRef.compareAndSet(ref, current, 0) : false;
    },
    
    // Thread-safe increment with retry
    safeIncrement: (): number => {
      let current: number;
      let success: boolean;
      
      do {
        current = MutableRef.get(ref);
        success = MutableRef.compareAndSet(ref, current, current + 1);
      } while (!success);
      
      return current + 1;
    },
    
    get: () => MutableRef.get(ref)
  };
};

const counter = makeCounter();

// Conditional operations
console.log(counter.incrementIfEquals(0)); // true, increments from 0 to 1
console.log(counter.incrementIfEquals(0)); // false, current is 1, not 0
console.log(counter.get()); // 1
```

### Numeric Operations

MutableRef provides optimized operations for numeric values:

```typescript
const makeStatistics = () => {
  const count = MutableRef.make(0);
  const sum = MutableRef.make(0);
  const min = MutableRef.make(Number.MAX_SAFE_INTEGER);
  const max = MutableRef.make(Number.MIN_SAFE_INTEGER);
  
  return {
    add: (value: number) => {
      // Increment count
      MutableRef.increment(count);
      
      // Add to sum
      MutableRef.update(sum, s => s + value);
      
      // Update min/max using compare-and-set pattern
      let currentMin: number;
      do {
        currentMin = MutableRef.get(min);
      } while (value < currentMin && !MutableRef.compareAndSet(min, currentMin, value));
      
      let currentMax: number;
      do {
        currentMax = MutableRef.get(max);
      } while (value > currentMax && !MutableRef.compareAndSet(max, currentMax, value));
    },
    
    getAverage: (): number => {
      const totalCount = MutableRef.get(count);
      const totalSum = MutableRef.get(sum);
      return totalCount > 0 ? totalSum / totalCount : 0;
    },
    
    getStats: () => ({
      count: MutableRef.get(count),
      sum: MutableRef.get(sum),
      min: MutableRef.get(min),
      max: MutableRef.get(max),
      average: Math.round((MutableRef.get(sum) / MutableRef.get(count)) * 100) / 100
    }),
    
    reset: () => {
      MutableRef.set(count, 0);
      MutableRef.set(sum, 0);
      MutableRef.set(min, Number.MAX_SAFE_INTEGER);
      MutableRef.set(max, Number.MIN_SAFE_INTEGER);
    }
  };
};

const stats = makeStatistics();

// Add some values
[10, 5, 15, 8, 12].forEach(value => stats.add(value));

console.log(stats.getStats());
// { count: 5, sum: 50, min: 5, max: 15, average: 10 }
```

### Boolean Toggle Operations

Convenient operations for boolean flags:

```typescript
const makeFeatureFlags = () => {
  const debugMode = MutableRef.make(false);
  const maintenanceMode = MutableRef.make(false);
  const experimentalFeatures = MutableRef.make(false);
  
  return {
    toggleDebug: () => {
      MutableRef.toggle(debugMode);
      console.log(`Debug mode: ${MutableRef.get(debugMode) ? 'ON' : 'OFF'}`);
    },
    
    enableMaintenance: () => {
      const wasEnabled = MutableRef.getAndSet(maintenanceMode, true);
      if (!wasEnabled) {
        console.log("Maintenance mode enabled");
      }
    },
    
    disableMaintenance: () => {
      const wasEnabled = MutableRef.getAndSet(maintenanceMode, false);
      if (wasEnabled) {
        console.log("Maintenance mode disabled");
      }
    },
    
    toggleExperimental: (): boolean => {
      const newValue = MutableRef.updateAndGet(experimentalFeatures, flag => !flag);
      console.log(`Experimental features: ${newValue ? 'ENABLED' : 'DISABLED'}`);
      return newValue;
    },
    
    getStatus: () => ({
      debug: MutableRef.get(debugMode),
      maintenance: MutableRef.get(maintenanceMode),
      experimental: MutableRef.get(experimentalFeatures)
    })
  };
};

const flags = makeFeatureFlags();

flags.toggleDebug(); // Debug mode: ON
flags.enableMaintenance(); // Maintenance mode enabled
flags.toggleExperimental(); // Experimental features: ENABLED

console.log(flags.getStatus());
// { debug: true, maintenance: true, experimental: true }
```

## Practical Patterns & Best Practices

### Pattern 1: Encapsulated Mutable State

Encapsulate MutableRef within classes or closures to provide controlled access:

```typescript
const createRateLimiter = (maxRequests: number, windowMs: number) => {
  const requests = MutableRef.make<Array<{ timestamp: number; id: string }>>([]);
  const blockedUntil = MutableRef.make(0);
  
  const cleanupOldRequests = () => {
    const now = Date.now();
    const cutoff = now - windowMs;
    
    MutableRef.update(requests, reqs => 
      reqs.filter(req => req.timestamp > cutoff)
    );
  };
  
  return {
    tryRequest: (requestId: string): { allowed: boolean; retryAfter?: number } => {
      const now = Date.now();
      
      // Check if we're in a blocked period
      const blockedTime = MutableRef.get(blockedUntil);
      if (now < blockedTime) {
        return { allowed: false, retryAfter: blockedTime - now };
      }
      
      cleanupOldRequests();
      
      const currentRequests = MutableRef.get(requests);
      
      if (currentRequests.length >= maxRequests) {
        // Rate limit exceeded - block for remaining window
        const oldestRequest = currentRequests[0];
        const retryAfter = (oldestRequest.timestamp + windowMs) - now;
        
        MutableRef.set(blockedUntil, now + retryAfter);
        return { allowed: false, retryAfter };
      }
      
      // Add request to tracking
      MutableRef.update(requests, reqs => [
        ...reqs, 
        { timestamp: now, id: requestId }
      ]);
      
      return { allowed: true };
    },
    
    getStatus: () => {
      cleanupOldRequests();
      const currentRequests = MutableRef.get(requests);
      return {
        currentRequests: currentRequests.length,
        maxRequests,
        windowMs,
        remainingCapacity: maxRequests - currentRequests.length
      };
    }
  };
};

// Usage
const limiter = createRateLimiter(10, 60000); // 10 requests per minute

const result = limiter.tryRequest("req-123");
if (result.allowed) {
  console.log("Request allowed");
} else {
  console.log(`Rate limited. Retry after ${result.retryAfter}ms`);
}
```

### Pattern 2: Performance-Optimized Updates

Use MutableRef for hot paths where Effect overhead is too costly:

```typescript
const createMetricsCollector = () => {
  // High-frequency counters
  const requestCount = MutableRef.make(0);
  const errorCount = MutableRef.make(0);
  const totalLatency = MutableRef.make(0);
  
  // Detailed metrics (updated less frequently)
  const detailedMetrics = MutableRef.make({
    lastHour: { requests: 0, errors: 0, avgLatency: 0 },
    lastDay: { requests: 0, errors: 0, avgLatency: 0 },
    allTime: { requests: 0, errors: 0, avgLatency: 0 }
  });
  
  let lastAggregation = Date.now();
  const aggregationInterval = 60000; // 1 minute
  
  return {
    // Hot path - minimal overhead
    recordRequest: (latencyMs: number, hasError: boolean) => {
      MutableRef.increment(requestCount);
      MutableRef.update(totalLatency, total => total + latencyMs);
      
      if (hasError) {
        MutableRef.increment(errorCount);
      }
    },
    
    // Cold path - detailed aggregation
    getDetailedMetrics: () => {
      const now = Date.now();
      
      // Aggregate if enough time has passed
      if (now - lastAggregation > aggregationInterval) {
        const requests = MutableRef.get(requestCount);
        const errors = MutableRef.get(errorCount);
        const latency = MutableRef.get(totalLatency);
        const avgLatency = requests > 0 ? latency / requests : 0;
        
        MutableRef.update(detailedMetrics, metrics => ({
          lastHour: { requests, errors, avgLatency },
          lastDay: metrics.lastHour, // Shift data
          allTime: {
            requests: metrics.allTime.requests + requests,
            errors: metrics.allTime.errors + errors,
            avgLatency: (metrics.allTime.avgLatency + avgLatency) / 2
          }
        }));
        
        // Reset counters for next interval
        MutableRef.set(requestCount, 0);
        MutableRef.set(errorCount, 0);
        MutableRef.set(totalLatency, 0);
        
        lastAggregation = now;
      }
      
      return MutableRef.get(detailedMetrics);
    },
    
    // Quick stats for monitoring
    getCurrentStats: () => ({
      requests: MutableRef.get(requestCount),
      errors: MutableRef.get(errorCount),
      avgLatency: MutableRef.get(requestCount) > 0 
        ? MutableRef.get(totalLatency) / MutableRef.get(requestCount) 
        : 0
    })
  };
};
```

### Pattern 3: Safe State Transitions

Use compare-and-set for safe state machine transitions:

```typescript
type ConnectionState = "disconnected" | "connecting" | "connected" | "error";

const createConnection = () => {
  const state = MutableRef.make<ConnectionState>("disconnected");
  const retryCount = MutableRef.make(0);
  const lastError = MutableRef.make<string | null>(null);
  
  return {
    connect: async (): Promise<boolean> => {
      // Only start connecting if currently disconnected
      if (!MutableRef.compareAndSet(state, "disconnected", "connecting")) {
        return false; // Already connecting or connected
      }
      
      try {
        // Simulate connection process
        await new Promise(resolve => setTimeout(resolve, 1000));
        
        // Transition to connected only if still in connecting state
        if (MutableRef.compareAndSet(state, "connecting", "connected")) {
          MutableRef.set(retryCount, 0);
          MutableRef.set(lastError, null);
          return true;
        }
        
        return false; // State changed during connection
      } catch (error) {
        const errorMessage = error instanceof Error ? error.message : "Connection failed";
        
        // Only transition to error if still connecting
        if (MutableRef.compareAndSet(state, "connecting", "error")) {
          MutableRef.increment(retryCount);
          MutableRef.set(lastError, errorMessage);
        }
        
        return false;
      }
    },
    
    disconnect: (): boolean => {
      const currentState = MutableRef.get(state);
      
      // Can disconnect from connected or error states
      if (currentState === "connected" || currentState === "error") {
        return MutableRef.compareAndSet(state, currentState, "disconnected");
      }
      
      return false;
    },
    
    getStatus: () => ({
      state: MutableRef.get(state),
      retryCount: MutableRef.get(retryCount),
      lastError: MutableRef.get(lastError)
    }),
    
    isConnected: (): boolean => MutableRef.get(state) === "connected",
    canRetry: (): boolean => MutableRef.get(retryCount) < 3
  };
};
```

## Integration Examples

### Integration with Effect

While MutableRef is designed for synchronous operations, it can be integrated with Effect for hybrid approaches:

```typescript
import { Effect, MutableRef } from "effect";

// Local state within Effect context
const createEffectfulService = Effect.gen(function* () {
  // Local performance counters (synchronous)
  const requestCount = MutableRef.make(0);
  const cacheHits = MutableRef.make(0);
  
  const processRequest = (data: unknown) => Effect.gen(function* () {
    // Increment counter immediately (no Effect overhead)
    MutableRef.increment(requestCount);
    
    // Check cache (synchronous)
    const cacheKey = generateCacheKey(data);
    const cached = checkCache(cacheKey);
    
    if (cached) {
      MutableRef.increment(cacheHits);
      return cached;
    }
    
    // Process with Effect (asynchronous)
    const result = yield* processWithEffect(data);
    
    // Update cache (synchronous)
    updateCache(cacheKey, result);
    
    return result;
  });
  
  const getMetrics = () => ({
    requests: MutableRef.get(requestCount),
    cacheHits: MutableRef.get(cacheHits),
    hitRatio: MutableRef.get(requestCount) > 0 
      ? MutableRef.get(cacheHits) / MutableRef.get(requestCount)
      : 0
  });
  
  return { processRequest, getMetrics } as const;
});

// Helper functions (implementation details)
function generateCacheKey(data: unknown): string {
  return JSON.stringify(data);
}

function checkCache(key: string): unknown | null {
  // Simple cache check
  return null;
}

function updateCache(key: string, value: unknown): void {
  // Simple cache update
}

function processWithEffect(data: unknown): Effect.Effect<unknown> {
  return Effect.succeed(data);
}
```

### Integration with React

MutableRef can be used for performance-critical state in React components:

```typescript
import React, { useEffect, useRef, useState } from "react";
import { MutableRef } from "effect";

interface PerformanceTracker {
  renderCount: MutableRef.MutableRef<number>;
  updateCount: MutableRef.MutableRef<number>;
  lastRenderTime: MutableRef.MutableRef<number>;
}

function usePerformanceTracking(): [PerformanceTracker, () => void] {
  const tracker = useRef<PerformanceTracker>({
    renderCount: MutableRef.make(0),
    updateCount: MutableRef.make(0),
    lastRenderTime: MutableRef.make(Date.now())
  });
  
  const [, forceUpdate] = useState({});
  
  // Track render on every render
  useEffect(() => {
    MutableRef.increment(tracker.current.renderCount);
    MutableRef.set(tracker.current.lastRenderTime, Date.now());
  });
  
  const triggerUpdate = () => {
    MutableRef.increment(tracker.current.updateCount);
    forceUpdate({});
  };
  
  return [tracker.current, triggerUpdate];
}

function PerformanceDemo() {
  const [perfTracker, triggerUpdate] = usePerformanceTracking();
  
  // Get performance stats (synchronous, no re-render)
  const getStats = () => ({
    renders: MutableRef.get(perfTracker.renderCount),
    updates: MutableRef.get(perfTracker.updateCount),
    lastRender: new Date(MutableRef.get(perfTracker.lastRenderTime)).toLocaleTimeString()
  });
  
  return (
    <div>
      <button onClick={triggerUpdate}>
        Trigger Update
      </button>
      
      <button onClick={() => console.log(getStats())}>
        Log Performance Stats
      </button>
      
      <div>
        Component rendered {MutableRef.get(perfTracker.renderCount)} times
      </div>
    </div>
  );
}
```

### Testing Strategies

Testing MutableRef-based code focuses on state transitions and synchronization:

```typescript
import { describe, it, expect } from "vitest";
import { MutableRef } from "effect";

describe("MutableRef State Machine", () => {
  it("should handle concurrent state transitions correctly", () => {
    const state = MutableRef.make<"idle" | "processing" | "done">("idle");
    const processCount = MutableRef.make(0);
    
    // Simulate concurrent processing attempts
    const results = [];
    
    for (let i = 0; i < 10; i++) {
      const canProcess = MutableRef.compareAndSet(state, "idle", "processing");
      
      if (canProcess) {
        MutableRef.increment(processCount);
        // Simulate processing
        MutableRef.set(state, "done");
        results.push("processed");
      } else {
        results.push("rejected");
      }
      
      // Reset for next iteration
      if (MutableRef.get(state) === "done") {
        MutableRef.set(state, "idle");
      }
    }
    
    // Only one should have processed each time
    expect(MutableRef.get(processCount)).toBe(results.filter(r => r === "processed").length);
  });
  
  it("should maintain consistency under rapid updates", () => {
    const counter = MutableRef.make(0);
    const iterations = 1000;
    
    // Rapid increments
    for (let i = 0; i < iterations; i++) {
      MutableRef.increment(counter);
    }
    
    expect(MutableRef.get(counter)).toBe(iterations);
    
    // Rapid decrements
    for (let i = 0; i < iterations; i++) {
      MutableRef.decrement(counter);
    }
    
    expect(MutableRef.get(counter)).toBe(0);
  });
  
  it("should handle complex state updates atomically", () => {
    interface ComplexState {
      counters: Record<string, number>;
      flags: Record<string, boolean>;
      metadata: { lastUpdate: number; version: number };
    }
    
    const state = MutableRef.make<ComplexState>({
      counters: {},
      flags: {},
      metadata: { lastUpdate: 0, version: 0 }
    });
    
    // Complex atomic update
    MutableRef.update(state, current => ({
      counters: { ...current.counters, requests: (current.counters.requests || 0) + 1 },
      flags: { ...current.flags, active: true },
      metadata: { 
        lastUpdate: Date.now(), 
        version: current.metadata.version + 1 
      }
    }));
    
    const result = MutableRef.get(state);
    expect(result.counters.requests).toBe(1);
    expect(result.flags.active).toBe(true);
    expect(result.metadata.version).toBe(1);
  });
});
```

## Conclusion

MutableRef provides lightweight, type-safe synchronous mutable references for performance-critical local state management. It excels when Effect's Ref is overkill and plain JavaScript variables lack type safety and encapsulation.

Key benefits:
- **Zero Effect Overhead**: Synchronous operations without Effect computation costs
- **Type Safety**: Full TypeScript support with compile-time guarantees  
- **Atomic Operations**: Compare-and-set and other atomic primitives for safe concurrent access
- **Performance Focus**: Optimized for hot paths and high-frequency updates

Use MutableRef when you need fast, local mutable state within specific scopes, performance counters, lightweight caches, or simple state machines where Effect's concurrency features aren't required.