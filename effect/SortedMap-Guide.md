# SortedMap: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem SortedMap Solves

Traditional JavaScript Maps and Objects provide no ordering guarantees for their keys. While insertion order is preserved in modern JavaScript, maintaining a sorted collection requires manual effort, leading to complex and error-prone code:

```typescript
// Traditional approach - manually maintaining sorted order
class SortedPriceIndex {
  private prices: Array<[string, number]> = []
  
  addPrice(product: string, price: number): void {
    // Remove existing entry if present
    this.prices = this.prices.filter(([p]) => p !== product)
    
    // Find correct position to maintain sort order
    let insertIndex = 0
    for (let i = 0; i < this.prices.length; i++) {
      if (price < this.prices[i][1]) {
        insertIndex = i
        break
      }
      insertIndex = i + 1
    }
    
    // Insert at correct position
    this.prices.splice(insertIndex, 0, [product, price])
  }
  
  getPriceRange(minPrice: number, maxPrice: number): Array<[string, number]> {
    // Manual range filtering
    const result: Array<[string, number]> = []
    for (const [product, price] of this.prices) {
      if (price >= minPrice && price <= maxPrice) {
        result.push([product, price])
      }
      if (price > maxPrice) break // Can optimize due to sorting
    }
    return result
  }
  
  getNearest(targetPrice: number): [string, number] | undefined {
    // Manual nearest neighbor search
    if (this.prices.length === 0) return undefined
    
    let closest = this.prices[0]
    let minDiff = Math.abs(targetPrice - closest[1])
    
    for (const entry of this.prices) {
      const diff = Math.abs(targetPrice - entry[1])
      if (diff < minDiff) {
        closest = entry
        minDiff = diff
      }
    }
    
    return closest
  }
}

// Using native Map with manual sorting
function getRankedLeaderboard(scores: Map<string, number>): Array<[string, number]> {
  // Convert to array and sort every time
  return Array.from(scores.entries())
    .sort(([, a], [, b]) => b - a) // Descending order
}

// Database index simulation with manual B-tree-like structure
class IndexedTable<K, V> {
  private data = new Map<K, V>()
  private indexes = new Map<string, Array<K>>()
  
  addRow(key: K, value: V, indexedFields: Record<string, any>): void {
    this.data.set(key, value)
    
    // Manually maintain sorted indexes
    for (const [field, fieldValue] of Object.entries(indexedFields)) {
      const index = this.indexes.get(field) || []
      const sortedIndex = [...index, key].sort((a, b) => {
        // Complex sorting logic for each field type
        const valA = this.getFieldValue(a, field)
        const valB = this.getFieldValue(b, field)
        return valA < valB ? -1 : valA > valB ? 1 : 0
      })
      this.indexes.set(field, sortedIndex)
    }
  }
  
  private getFieldValue(key: K, field: string): any {
    // Complex extraction logic
    return undefined
  }
}
```

This approach leads to:
- **Performance issues** - O(n) insertion and O(n log n) sorting on every operation
- **Memory inefficiency** - Duplicate data structures for maintaining order
- **Complex range queries** - Manual implementation of between, greater than, less than operations
- **Error-prone neighbor searches** - Hand-coded algorithms for finding nearest values
- **No type safety** - Custom comparators without type guarantees

### The SortedMap Solution

SortedMap provides a persistent, immutable sorted map implementation with efficient ordered operations and type-safe custom comparators:

```typescript
import { SortedMap, Order, pipe } from "effect"

// Type-safe sorted collection with custom ordering
const priceIndex = SortedMap.empty<string, number>(Order.number)

const withPrices = pipe(
  SortedMap.set(priceIndex, "laptop", 999.99),
  SortedMap.set("mouse", 29.99),
  SortedMap.set("keyboard", 79.99),
  SortedMap.set("monitor", 299.99)
)

// Efficient range queries - O(log n)
const budgetItems = SortedMap.between(withPrices, 20, 100) // All items between $20-$100

// Type-safe custom ordering
interface TimeEntry {
  timestamp: Date
  userId: string
  action: string
}

const timeBasedOrder = Order.mapInput(
  Order.Date,
  (entry: TimeEntry) => entry.timestamp
)

const activityLog = SortedMap.empty<TimeEntry, string>(timeBasedOrder)

// Automatic ordering maintenance
const withActivities = pipe(
  SortedMap.set(
    activityLog,
    { timestamp: new Date('2024-01-15T10:30:00'), userId: 'user1', action: 'login' },
    'User logged in'
  ),
  SortedMap.set(
    { timestamp: new Date('2024-01-15T09:15:00'), userId: 'user2', action: 'logout' },
    'User logged out'
  )
)

// Entries automatically sorted by timestamp
const chronologicalEvents = SortedMap.values(withActivities)
```

### Key Concepts

**Ordered Keys**: SortedMap maintains keys in sorted order according to a provided Order instance, enabling efficient range queries and ordered iteration.

**Custom Comparators**: Type-safe Order instances define how keys are compared, supporting complex sorting strategies while maintaining type safety.

**Structural Sharing**: Like HashMap, SortedMap uses persistent data structures with structural sharing for memory efficiency.

**Logarithmic Operations**: Most operations (get, set, remove) run in O(log n) time, with efficient implementations for range queries and nearest neighbor searches.

## Basic Usage Patterns

### Pattern 1: Creating and Initializing SortedMaps

```typescript
import { SortedMap, Order, pipe } from "effect"

// Empty SortedMap with number ordering
const emptyNumbers = SortedMap.empty<string, number>(Order.string)

// From entries with custom ordering
const scoreBoard = SortedMap.fromIterable(
  Order.mapInput(Order.number, ([, score]: [string, number]) => -score), // Descending
  [
    ["Alice", 95],
    ["Bob", 87],
    ["Charlie", 92],
    ["David", 88]
  ]
)

// Complex key ordering
interface Version {
  major: number
  minor: number
  patch: number
}

const versionOrder = Order.struct({
  major: Order.number,
  minor: Order.number,
  patch: Order.number
})

const versionHistory = SortedMap.empty<Version, string>(versionOrder)

const withVersions = pipe(
  SortedMap.set(versionHistory, { major: 1, minor: 0, patch: 0 }, "Initial release"),
  SortedMap.set({ major: 1, minor: 1, patch: 0 }, "Feature update"),
  SortedMap.set({ major: 1, minor: 0, patch: 1 }, "Bug fix"),
  SortedMap.set({ major: 2, minor: 0, patch: 0 }, "Major release")
)

// Date-based ordering
const eventLog = SortedMap.empty<Date, string>(Order.Date)

const withEvents = pipe(
  eventLog,
  SortedMap.set(new Date('2024-01-15'), "Project started"),
  SortedMap.set(new Date('2024-02-01'), "First milestone"),
  SortedMap.set(new Date('2024-01-20'), "Team meeting")
)
```

### Pattern 2: Basic Operations

```typescript
import { SortedMap, Order, pipe, Option } from "effect"

const inventory = pipe(
  SortedMap.empty<string, number>(Order.string),
  SortedMap.set("apples", 50),
  SortedMap.set("bananas", 30),
  SortedMap.set("cherries", 20),
  SortedMap.set("dates", 15)
)

// Getting values
const appleCount = SortedMap.get(inventory, "apples")  // Option.some(50)
const grapeCount = SortedMap.get(inventory, "grapes")  // Option.none()

// Checking existence
const hasApples = SortedMap.has(inventory, "apples")   // true
const hasMangos = SortedMap.has(inventory, "mangos")   // false

// Getting size
const itemCount = SortedMap.size(inventory)  // 4

// Updating values
const updatedInventory = pipe(
  inventory,
  SortedMap.set("apples", 45),      // Update existing
  SortedMap.set("elderberries", 10) // Add new
)

// Removing values
const afterSale = pipe(
  inventory,
  SortedMap.remove("dates"),
  SortedMap.modify("apples", count => count - 5)
)

// Getting first and last entries
const firstItem = SortedMap.head(inventory)  // Option.some(["apples", 50])
const lastItem = SortedMap.last(inventory)   // Option.some(["dates", 15])
```

### Pattern 3: Iterating Over Sorted Data

```typescript
import { SortedMap, Order, pipe } from "effect"

const grades = pipe(
  SortedMap.empty<string, number>(Order.string),
  SortedMap.set("Alice", 95),
  SortedMap.set("Bob", 87),
  SortedMap.set("Charlie", 92),
  SortedMap.set("David", 88),
  SortedMap.set("Eve", 91)
)

// Iterate in key order
const studentList = Array.from(SortedMap.entries(grades))
// [["Alice", 95], ["Bob", 87], ["Charlie", 92], ["David", 88], ["Eve", 91]]

// Get sorted keys
const students = Array.from(SortedMap.keys(grades))
// ["Alice", "Bob", "Charlie", "David", "Eve"]

// Get values in key order
const scores = Array.from(SortedMap.values(grades))
// [95, 87, 92, 88, 91]

// Reverse iteration
const reverseOrder = SortedMap.empty<string, number>(Order.reverse(Order.string))
const reverseGrades = pipe(
  reverseOrder,
  SortedMap.set("Alice", 95),
  SortedMap.set("Bob", 87),
  SortedMap.set("Charlie", 92)
)

const reverseStudents = Array.from(SortedMap.keys(reverseGrades))
// ["Charlie", "Bob", "Alice"]

// Custom iteration with reduce
const classAverage = pipe(
  grades,
  SortedMap.reduce(
    { total: 0, count: 0 },
    (acc, value) => ({
      total: acc.total + value,
      count: acc.count + 1
    })
  ),
  ({ total, count }) => total / count
)
```

## Real-World Examples

### Example 1: Time-Series Data Management

Managing time-series data efficiently with automatic chronological ordering:

```typescript
import { SortedMap, Order, pipe, Option, Effect } from "effect"

interface MetricPoint {
  timestamp: Date
  value: number
  metadata?: Record<string, any>
}

class TimeSeriesStore {
  private data: SortedMap.SortedMap<Date, MetricPoint>
  
  constructor() {
    this.data = SortedMap.empty(Order.Date)
  }
  
  // Add new data point
  addPoint(point: MetricPoint): TimeSeriesStore {
    const updated = pipe(
      this.data,
      SortedMap.set(point.timestamp, point)
    )
    return new TimeSeriesStore().withData(updated)
  }
  
  // Get data within time range
  getRange(start: Date, end: Date): ReadonlyArray<MetricPoint> {
    return pipe(
      this.data,
      SortedMap.between(start, end),
      Array.from,
      arr => arr.map(([, point]) => point)
    )
  }
  
  // Get latest N points
  getLatest(n: number): ReadonlyArray<MetricPoint> {
    return pipe(
      this.data,
      SortedMap.entries,
      Array.from,
      arr => arr.slice(-n).map(([, point]) => point)
    )
  }
  
  // Downsample data by time buckets
  downsample(bucketSize: number): ReadonlyArray<MetricPoint> {
    const buckets = new Map<number, MetricPoint[]>()
    
    for (const [, point] of this.data) {
      const bucket = Math.floor(point.timestamp.getTime() / bucketSize) * bucketSize
      const existing = buckets.get(bucket) || []
      buckets.set(bucket, [...existing, point])
    }
    
    return Array.from(buckets.entries())
      .sort(([a], [b]) => a - b)
      .map(([timestamp, points]) => ({
        timestamp: new Date(timestamp),
        value: points.reduce((sum, p) => sum + p.value, 0) / points.length,
        metadata: { sampleCount: points.length }
      }))
  }
  
  // Find anomalies using neighboring values
  findAnomalies(threshold: number): ReadonlyArray<MetricPoint> {
    const anomalies: MetricPoint[] = []
    const entries = Array.from(SortedMap.entries(this.data))
    
    for (let i = 1; i < entries.length - 1; i++) {
      const [, prev] = entries[i - 1]
      const [, current] = entries[i]
      const [, next] = entries[i + 1]
      
      const avgNeighbor = (prev.value + next.value) / 2
      const deviation = Math.abs(current.value - avgNeighbor)
      
      if (deviation > threshold) {
        anomalies.push(current)
      }
    }
    
    return anomalies
  }
  
  private withData(data: SortedMap.SortedMap<Date, MetricPoint>): TimeSeriesStore {
    this.data = data
    return this
  }
}

// Usage example: Server monitoring
const serverMetrics = new TimeSeriesStore()

const populated = [
  { timestamp: new Date('2024-01-15T10:00:00'), value: 45.2 },
  { timestamp: new Date('2024-01-15T10:05:00'), value: 48.1 },
  { timestamp: new Date('2024-01-15T10:10:00'), value: 95.8 }, // Spike
  { timestamp: new Date('2024-01-15T10:15:00'), value: 47.3 },
  { timestamp: new Date('2024-01-15T10:20:00'), value: 46.9 }
].reduce((store, point) => store.addPoint(point), serverMetrics)

const anomalies = populated.findAnomalies(30)
console.log(`Found ${anomalies.length} anomalies`)

const recentData = populated.getLatest(3)
console.log('Recent metrics:', recentData)
```

### Example 2: Order Book Implementation

Financial order book with efficient price level management:

```typescript
import { SortedMap, Order, pipe, Option, Match } from "effect"

interface Order {
  id: string
  price: number
  quantity: number
  timestamp: Date
}

class OrderBook {
  // Bids sorted descending (highest price first)
  private bids: SortedMap.SortedMap<number, Order[]>
  // Asks sorted ascending (lowest price first)
  private asks: SortedMap.SortedMap<number, Order[]>
  
  constructor() {
    this.bids = SortedMap.empty(Order.reverse(Order.number))
    this.asks = SortedMap.empty(Order.number)
  }
  
  // Add limit order
  addOrder(side: 'buy' | 'sell', order: Order): OrderBook {
    const book = side === 'buy' ? this.bids : this.asks
    
    const updated = pipe(
      book,
      SortedMap.modifyOption(
        order.price,
        existing => [...existing, order]
      ),
      Option.getOrElse(() => 
        SortedMap.set(book, order.price, [order])
      )
    )
    
    return side === 'buy' 
      ? new OrderBook().withBids(updated).withAsks(this.asks)
      : new OrderBook().withBids(this.bids).withAsks(updated)
  }
  
  // Get best bid/ask
  getBestBid(): Option.Option<[number, Order[]]> {
    return SortedMap.head(this.bids)
  }
  
  getBestAsk(): Option.Option<[number, Order[]]> {
    return SortedMap.head(this.asks)
  }
  
  // Get spread
  getSpread(): Option.Option<number> {
    return pipe(
      Option.zip(this.getBestBid(), this.getBestAsk()),
      Option.map(([[bidPrice], [askPrice]]) => askPrice - bidPrice)
    )
  }
  
  // Get market depth
  getDepth(levels: number): {
    bids: Array<[number, number]>,
    asks: Array<[number, number]>
  } {
    const aggregateOrders = (orders: Order[]) =>
      orders.reduce((sum, order) => sum + order.quantity, 0)
    
    const bidDepth = pipe(
      this.bids,
      SortedMap.entries,
      Array.from,
      arr => arr.slice(0, levels).map(([price, orders]) => 
        [price, aggregateOrders(orders)] as [number, number]
      )
    )
    
    const askDepth = pipe(
      this.asks,
      SortedMap.entries,
      Array.from,
      arr => arr.slice(0, levels).map(([price, orders]) => 
        [price, aggregateOrders(orders)] as [number, number]
      )
    )
    
    return { bids: bidDepth, asks: askDepth }
  }
  
  // Execute market order
  executeMarketOrder(side: 'buy' | 'sell', quantity: number): {
    executed: Order[],
    remaining: number,
    avgPrice: number
  } {
    const book = side === 'buy' ? this.asks : this.bids
    const executed: Order[] = []
    let remaining = quantity
    let totalCost = 0
    
    for (const [price, orders] of book) {
      if (remaining <= 0) break
      
      for (const order of orders) {
        const fillQuantity = Math.min(remaining, order.quantity)
        executed.push({ ...order, quantity: fillQuantity })
        totalCost += fillQuantity * price
        remaining -= fillQuantity
        
        if (remaining <= 0) break
      }
    }
    
    const avgPrice = executed.length > 0 
      ? totalCost / (quantity - remaining)
      : 0
    
    return { executed, remaining, avgPrice }
  }
  
  // Get orders within price range
  getOrdersInRange(side: 'buy' | 'sell', minPrice: number, maxPrice: number): Order[] {
    const book = side === 'buy' ? this.bids : this.asks
    
    return pipe(
      book,
      SortedMap.between(minPrice, maxPrice),
      Array.from,
      arr => arr.flatMap(([, orders]) => orders)
    )
  }
  
  private withBids(bids: SortedMap.SortedMap<number, Order[]>): OrderBook {
    this.bids = bids
    return this
  }
  
  private withAsks(asks: SortedMap.SortedMap<number, Order[]>): OrderBook {
    this.asks = asks
    return this
  }
}

// Usage example
const orderBook = new OrderBook()

const withOrders = [
  { side: 'buy' as const, order: { id: '1', price: 100.00, quantity: 50, timestamp: new Date() } },
  { side: 'buy' as const, order: { id: '2', price: 99.50, quantity: 100, timestamp: new Date() } },
  { side: 'sell' as const, order: { id: '3', price: 100.50, quantity: 75, timestamp: new Date() } },
  { side: 'sell' as const, order: { id: '4', price: 101.00, quantity: 50, timestamp: new Date() } }
].reduce((book, { side, order }) => book.addOrder(side, order), orderBook)

console.log('Market depth:', withOrders.getDepth(5))
console.log('Spread:', withOrders.getSpread())

const marketBuy = withOrders.executeMarketOrder('buy', 80)
console.log(`Executed at avg price: ${marketBuy.avgPrice}`)
```

### Example 3: Database Index Simulation

Simulating database indexes with compound keys and range scans:

```typescript
import { SortedMap, Order, pipe, Option, Either } from "effect"

// Compound index key
interface IndexKey {
  field1: string
  field2: number
  field3: Date
}

interface Record {
  id: string
  data: any
  indexed: IndexKey
}

class IndexedTable {
  private primaryKey: SortedMap.SortedMap<string, Record>
  private compoundIndex: SortedMap.SortedMap<IndexKey, string>
  
  constructor() {
    this.primaryKey = SortedMap.empty(Order.string)
    
    // Compound key ordering
    this.compoundIndex = SortedMap.empty(
      Order.combine(
        Order.mapInput(Order.string, (k: IndexKey) => k.field1),
        Order.combine(
          Order.mapInput(Order.number, (k: IndexKey) => k.field2),
          Order.mapInput(Order.Date, (k: IndexKey) => k.field3)
        )
      )
    )
  }
  
  // Insert with index maintenance
  insert(record: Record): Either.Either<string, IndexedTable> {
    if (SortedMap.has(this.primaryKey, record.id)) {
      return Either.left(`Record ${record.id} already exists`)
    }
    
    const updatedPrimary = SortedMap.set(this.primaryKey, record.id, record)
    const updatedIndex = SortedMap.set(this.compoundIndex, record.indexed, record.id)
    
    return Either.right(
      new IndexedTable()
        .withPrimaryKey(updatedPrimary)
        .withCompoundIndex(updatedIndex)
    )
  }
  
  // Range scan on compound index
  scanRange(
    field1Range?: [string, string],
    field2Range?: [number, number],
    field3Range?: [Date, Date]
  ): ReadonlyArray<Record> {
    // Build min/max keys for range
    const minKey: IndexKey = {
      field1: field1Range?.[0] ?? '',
      field2: field2Range?.[0] ?? Number.MIN_SAFE_INTEGER,
      field3: field3Range?.[0] ?? new Date(0)
    }
    
    const maxKey: IndexKey = {
      field1: field1Range?.[1] ?? '\uFFFF',
      field2: field2Range?.[1] ?? Number.MAX_SAFE_INTEGER,
      field3: field3Range?.[1] ?? new Date(8640000000000000)
    }
    
    return pipe(
      this.compoundIndex,
      SortedMap.between(minKey, maxKey),
      Array.from,
      arr => arr.map(([, id]) => SortedMap.unsafeGet(this.primaryKey, id)),
      // Additional filtering for partial matches
      records => records.filter(record => {
        const idx = record.indexed
        return (
          (!field1Range || (idx.field1 >= field1Range[0] && idx.field1 <= field1Range[1])) &&
          (!field2Range || (idx.field2 >= field2Range[0] && idx.field2 <= field2Range[1])) &&
          (!field3Range || (idx.field3 >= field3Range[0] && idx.field3 <= field3Range[1]))
        )
      })
    )
  }
  
  // Prefix scan (e.g., all records where field1 starts with prefix)
  prefixScan(field1Prefix: string): ReadonlyArray<Record> {
    const minKey: IndexKey = {
      field1: field1Prefix,
      field2: Number.MIN_SAFE_INTEGER,
      field3: new Date(0)
    }
    
    const maxKey: IndexKey = {
      field1: field1Prefix + '\uFFFF',
      field2: Number.MAX_SAFE_INTEGER,
      field3: new Date(8640000000000000)
    }
    
    return pipe(
      this.compoundIndex,
      SortedMap.between(minKey, maxKey),
      Array.from,
      arr => arr.map(([, id]) => SortedMap.unsafeGet(this.primaryKey, id))
    )
  }
  
  // Update with index maintenance
  update(id: string, updates: Partial<Record>): Either.Either<string, IndexedTable> {
    return pipe(
      Option.match(SortedMap.get(this.primaryKey, id), {
        onNone: () => Either.left(`Record ${id} not found`),
        onSome: (existing) => {
          const updated = { ...existing, ...updates }
          
          // Remove old index entry if key changed
          const indexChanged = updates.indexed && 
            JSON.stringify(existing.indexed) !== JSON.stringify(updated.indexed)
          
          let newIndex = this.compoundIndex
          if (indexChanged) {
            newIndex = SortedMap.remove(newIndex, existing.indexed)
            newIndex = SortedMap.set(newIndex, updated.indexed, id)
          }
          
          const newPrimary = SortedMap.set(this.primaryKey, id, updated)
          
          return Either.right(
            new IndexedTable()
              .withPrimaryKey(newPrimary)
              .withCompoundIndex(newIndex)
          )
        }
      })
    )
  }
  
  // Query planner simulation
  explainQuery(
    field1Range?: [string, string],
    field2Range?: [number, number],
    field3Range?: [Date, Date]
  ): string {
    const ranges = []
    if (field1Range) ranges.push(`field1: [${field1Range[0]}, ${field1Range[1]}]`)
    if (field2Range) ranges.push(`field2: [${field2Range[0]}, ${field2Range[1]}]`)
    if (field3Range) ranges.push(`field3: [${field3Range[0].toISOString()}, ${field3Range[1].toISOString()}]`)
    
    const indexSize = SortedMap.size(this.compoundIndex)
    const estimatedRows = Math.floor(indexSize * 0.1) // Rough estimate
    
    return `
Query Plan:
  Index Scan using compound_index
  Filter: ${ranges.join(' AND ')}
  Estimated rows: ${estimatedRows}
  Index size: ${indexSize}
    `.trim()
  }
  
  private withPrimaryKey(pk: SortedMap.SortedMap<string, Record>): IndexedTable {
    this.primaryKey = pk
    return this
  }
  
  private withCompoundIndex(idx: SortedMap.SortedMap<IndexKey, string>): IndexedTable {
    this.compoundIndex = idx
    return this
  }
}

// Usage example
const table = new IndexedTable()

const records: Record[] = [
  {
    id: '1',
    data: { name: 'Product A', price: 99.99 },
    indexed: { field1: 'electronics', field2: 100, field3: new Date('2024-01-01') }
  },
  {
    id: '2',
    data: { name: 'Product B', price: 49.99 },
    indexed: { field1: 'electronics', field2: 50, field3: new Date('2024-01-15') }
  },
  {
    id: '3',
    data: { name: 'Product C', price: 199.99 },
    indexed: { field1: 'furniture', field2: 200, field3: new Date('2024-02-01') }
  }
]

const populated = records.reduce(
  (tbl, record) => pipe(
    tbl.insert(record),
    Either.getOrElse(() => tbl)
  ),
  table
)

// Complex range query
const results = populated.scanRange(
  ['electronics', 'electronics'],
  [0, 150],
  undefined
)
console.log('Query results:', results.length)

console.log(populated.explainQuery(['electronics', 'furniture']))
```

## Advanced Features Deep Dive

### Range Operations: Efficient Subset Queries

SortedMap provides powerful range query capabilities for extracting ordered subsets:

#### Basic Range Operations

```typescript
import { SortedMap, Order, pipe } from "effect"

const temperatures = pipe(
  SortedMap.empty<Date, number>(Order.Date),
  SortedMap.set(new Date('2024-01-15T00:00:00'), 5.2),
  SortedMap.set(new Date('2024-01-15T06:00:00'), 8.1),
  SortedMap.set(new Date('2024-01-15T12:00:00'), 15.3),
  SortedMap.set(new Date('2024-01-15T18:00:00'), 12.7),
  SortedMap.set(new Date('2024-01-16T00:00:00'), 7.4)
)

// Get all readings between two times
const afternoon = pipe(
  temperatures,
  SortedMap.between(
    new Date('2024-01-15T12:00:00'),
    new Date('2024-01-15T23:59:59')
  )
)

// Get all readings after a specific time
const evening = pipe(
  temperatures,
  SortedMap.greaterThan(new Date('2024-01-15T18:00:00'))
)

// Get all readings before noon
const morning = pipe(
  temperatures,
  SortedMap.lessThanOrEqual(new Date('2024-01-15T12:00:00'))
)
```

#### Real-World Range Query Example

```typescript
import { SortedMap, Order, pipe, Option } from "effect"

interface PricePoint {
  timestamp: Date
  symbol: string
  price: number
  volume: number
}

class PriceRangeAnalyzer {
  private data: SortedMap.SortedMap<number, PricePoint[]>
  
  constructor() {
    this.data = SortedMap.empty(Order.number)
  }
  
  addPricePoint(point: PricePoint): PriceRangeAnalyzer {
    const updated = pipe(
      this.data,
      SortedMap.modifyOption(
        point.price,
        existing => [...existing, point]
      ),
      Option.getOrElse(() => 
        SortedMap.set(this.data, point.price, [point])
      )
    )
    
    return new PriceRangeAnalyzer().withData(updated)
  }
  
  // Find support and resistance levels
  findKeyLevels(tolerance: number = 0.01): {
    support: number[],
    resistance: number[]
  } {
    const priceFrequency = new Map<number, number>()
    
    // Round prices to tolerance level
    for (const [price, points] of this.data) {
      const rounded = Math.round(price / tolerance) * tolerance
      priceFrequency.set(rounded, (priceFrequency.get(rounded) || 0) + points.length)
    }
    
    // Sort by frequency
    const levels = Array.from(priceFrequency.entries())
      .sort(([, a], [, b]) => b - a)
      .slice(0, 5)
      .map(([price]) => price)
    
    const currentPrice = pipe(
      SortedMap.last(this.data),
      Option.map(([price]) => price),
      Option.getOrElse(() => 0)
    )
    
    return {
      support: levels.filter(level => level < currentPrice),
      resistance: levels.filter(level => level > currentPrice)
    }
  }
  
  // Volume-weighted average price in range
  vwapInRange(minPrice: number, maxPrice: number): number {
    const pointsInRange = pipe(
      this.data,
      SortedMap.between(minPrice, maxPrice),
      Array.from,
      arr => arr.flatMap(([, points]) => points)
    )
    
    if (pointsInRange.length === 0) return 0
    
    const totalValue = pointsInRange.reduce(
      (sum, point) => sum + point.price * point.volume,
      0
    )
    const totalVolume = pointsInRange.reduce(
      (sum, point) => sum + point.volume,
      0
    )
    
    return totalValue / totalVolume
  }
  
  // Price distribution analysis
  getPriceDistribution(bucketSize: number): Array<{
    range: [number, number],
    count: number,
    volume: number
  }> {
    const minPrice = pipe(
      SortedMap.head(this.data),
      Option.map(([price]) => price),
      Option.getOrElse(() => 0)
    )
    
    const maxPrice = pipe(
      SortedMap.last(this.data),
      Option.map(([price]) => price),
      Option.getOrElse(() => 0)
    )
    
    const buckets: Array<{
      range: [number, number],
      count: number,
      volume: number
    }> = []
    
    for (let start = minPrice; start < maxPrice; start += bucketSize) {
      const end = start + bucketSize
      const pointsInBucket = pipe(
        this.data,
        SortedMap.between(start, end),
        Array.from,
        arr => arr.flatMap(([, points]) => points)
      )
      
      buckets.push({
        range: [start, end],
        count: pointsInBucket.length,
        volume: pointsInBucket.reduce((sum, p) => sum + p.volume, 0)
      })
    }
    
    return buckets
  }
  
  private withData(data: SortedMap.SortedMap<number, PricePoint[]>): PriceRangeAnalyzer {
    this.data = data
    return this
  }
}

// Usage
const analyzer = new PriceRangeAnalyzer()

const withData = [
  { timestamp: new Date(), symbol: 'AAPL', price: 180.52, volume: 1000 },
  { timestamp: new Date(), symbol: 'AAPL', price: 180.48, volume: 1500 },
  { timestamp: new Date(), symbol: 'AAPL', price: 181.25, volume: 2000 },
  { timestamp: new Date(), symbol: 'AAPL', price: 180.50, volume: 1200 }
].reduce((a, point) => a.addPricePoint(point), analyzer)

const keyLevels = withData.findKeyLevels()
console.log('Key levels:', keyLevels)

const vwap = withData.vwapInRange(180.00, 181.00)
console.log('VWAP:', vwap)
```

#### Advanced Range: Nearest Neighbor Queries

```typescript
import { SortedMap, Order, pipe, Option } from "effect"

// Helper to find nearest neighbors
function findNearest<K, V>(
  map: SortedMap.SortedMap<K, V>,
  target: K,
  order: Order.Order<K>
): {
  exact: Option.Option<V>,
  lower: Option.Option<[K, V]>,
  higher: Option.Option<[K, V]>
} {
  const exact = SortedMap.get(map, target)
  
  // Find closest lower value
  const lower = pipe(
    map,
    SortedMap.lessThan(target),
    sm => SortedMap.last(sm)
  )
  
  // Find closest higher value
  const higher = pipe(
    map,
    SortedMap.greaterThan(target),
    sm => SortedMap.head(sm)
  )
  
  return { exact, lower, higher }
}

// Interpolation helper
function interpolateValue(
  map: SortedMap.SortedMap<number, number>,
  x: number
): Option.Option<number> {
  const { exact, lower, higher } = findNearest(map, x, Order.number)
  
  return pipe(
    exact,
    Option.orElse(() => 
      pipe(
        Option.zip(lower, higher),
        Option.map(([[x1, y1], [x2, y2]]) => {
          // Linear interpolation
          const slope = (y2 - y1) / (x2 - x1)
          return y1 + slope * (x - x1)
        })
      )
    )
  )
}

// Usage example: Temperature interpolation
const tempReadings = pipe(
  SortedMap.empty<number, number>(Order.number), // hour -> temperature
  SortedMap.set(0, 15.0),
  SortedMap.set(6, 18.5),
  SortedMap.set(12, 25.0),
  SortedMap.set(18, 22.0),
  SortedMap.set(24, 16.0)
)

const temp9am = interpolateValue(tempReadings, 9) // Interpolated between 6am and 12pm
console.log('Temperature at 9am:', temp9am)

// Geographic nearest neighbor
interface Location {
  latitude: number
  longitude: number
  name: string
}

const distanceOrder = (center: { lat: number, lon: number }) =>
  Order.mapInput(
    Order.number,
    (loc: Location) => {
      const dx = loc.latitude - center.lat
      const dy = loc.longitude - center.lon
      return Math.sqrt(dx * dx + dy * dy)
    }
  )

const stores = pipe(
  SortedMap.empty<Location, string>(
    distanceOrder({ lat: 40.7128, lon: -74.0060 }) // NYC center
  ),
  SortedMap.set(
    { latitude: 40.7580, longitude: -73.9855, name: 'Times Square' },
    'Store #1'
  ),
  SortedMap.set(
    { latitude: 40.7484, longitude: -73.9857, name: 'Empire State' },
    'Store #2'
  ),
  SortedMap.set(
    { latitude: 40.7115, longitude: -74.0125, name: 'Financial District' },
    'Store #3'
  )
)

const nearest = SortedMap.head(stores) // Closest to NYC center
console.log('Nearest store:', nearest)
```

### Custom Comparators: Complex Ordering Strategies

SortedMap's power comes from its flexible ordering system that maintains type safety:

#### Basic Custom Ordering

```typescript
import { SortedMap, Order, pipe } from "effect"

// Case-insensitive string ordering
const caseInsensitiveOrder = Order.mapInput(
  Order.string,
  (s: string) => s.toLowerCase()
)

const usernames = pipe(
  SortedMap.empty<string, number>(caseInsensitiveOrder),
  SortedMap.set('Alice', 1),
  SortedMap.set('BOB', 2),
  SortedMap.set('alice', 3), // Updates existing 'Alice' entry
  SortedMap.set('Charlie', 4)
)

// Multi-field ordering with priorities
interface Task {
  priority: 'high' | 'medium' | 'low'
  dueDate: Date
  title: string
}

const priorityValues = { high: 0, medium: 1, low: 2 }

const taskOrder = Order.combine(
  Order.mapInput(Order.number, (t: Task) => priorityValues[t.priority]),
  Order.combine(
    Order.mapInput(Order.Date, (t: Task) => t.dueDate),
    Order.mapInput(Order.string, (t: Task) => t.title)
  )
)

const taskQueue = SortedMap.empty<Task, string>(taskOrder)

// Custom ordering for version numbers
interface Version {
  major: number
  minor: number
  patch: number
  prerelease?: string
}

const versionOrder: Order.Order<Version> = pipe(
  Order.tuple(Order.number, Order.number, Order.number),
  Order.mapInput((v: Version) => [v.major, v.minor, v.patch] as const),
  Order.combine(
    Order.mapInput(
      Order.nullable(Order.string),
      (v: Version) => v.prerelease ?? null
    )
  )
)
```

#### Real-World Custom Order Example

```typescript
import { SortedMap, Order, pipe, Option } from "effect"

// Complex scoring system for search results
interface SearchResult {
  id: string
  title: string
  relevanceScore: number
  popularity: number
  recency: Date
  isSponsored: boolean
}

class SearchRanker {
  private weights = {
    relevance: 0.4,
    popularity: 0.3,
    recency: 0.2,
    sponsored: 0.1
  }
  
  // Composite scoring order
  private resultOrder = Order.mapInput(
    Order.reverse(Order.number), // Higher scores first
    (result: SearchResult) => this.calculateScore(result)
  )
  
  private results: SortedMap.SortedMap<SearchResult, string>
  
  constructor() {
    this.results = SortedMap.empty(this.resultOrder)
  }
  
  private calculateScore(result: SearchResult): number {
    const daysSincePublish = 
      (Date.now() - result.recency.getTime()) / (1000 * 60 * 60 * 24)
    const recencyScore = Math.max(0, 1 - daysSincePublish / 365)
    
    return (
      result.relevanceScore * this.weights.relevance +
      result.popularity * this.weights.popularity +
      recencyScore * this.weights.recency +
      (result.isSponsored ? 1 : 0) * this.weights.sponsored
    )
  }
  
  addResult(result: SearchResult): SearchRanker {
    const updated = SortedMap.set(this.results, result, result.id)
    return new SearchRanker().withResults(updated)
  }
  
  // Re-rank with different weights
  rerank(newWeights: Partial<typeof this.weights>): SearchRanker {
    const ranker = new SearchRanker()
    ranker.weights = { ...this.weights, ...newWeights }
    
    // Rebuild with new ordering
    const entries = Array.from(SortedMap.entries(this.results))
    ranker.results = entries.reduce(
      (acc, [result]) => SortedMap.set(acc, result, result.id),
      SortedMap.empty(ranker.resultOrder)
    )
    
    return ranker
  }
  
  getTopResults(n: number): SearchResult[] {
    return pipe(
      this.results,
      SortedMap.entries,
      Array.from,
      arr => arr.slice(0, n).map(([result]) => result)
    )
  }
  
  // A/B testing different ranking strategies
  compareRankings(
    alternativeWeights: typeof this.weights
  ): {
    original: string[],
    alternative: string[],
    changes: number
  } {
    const original = this.getTopResults(10).map(r => r.id)
    const alternative = this.rerank(alternativeWeights).getTopResults(10).map(r => r.id)
    
    let changes = 0
    for (let i = 0; i < original.length; i++) {
      if (original[i] !== alternative[i]) changes++
    }
    
    return { original, alternative, changes }
  }
  
  private withResults(results: SortedMap.SortedMap<SearchResult, string>): SearchRanker {
    this.results = results
    return this
  }
}

// Geographic ordering with multiple factors
interface Store {
  id: string
  location: { lat: number, lon: number }
  rating: number
  isOpen: boolean
}

function createProximityOrder(userLocation: { lat: number, lon: number }) {
  // Combine distance with other factors
  return Order.combine(
    // First priority: open stores
    Order.mapInput(Order.boolean, (s: Store) => !s.isOpen),
    Order.combine(
      // Second: distance (in 5km buckets)
      Order.mapInput(Order.number, (s: Store) => {
        const distance = calculateDistance(userLocation, s.location)
        return Math.floor(distance / 5) // 5km buckets
      }),
      // Third: rating (descending)
      Order.mapInput(Order.reverse(Order.number), (s: Store) => s.rating)
    )
  )
}

function calculateDistance(
  p1: { lat: number, lon: number },
  p2: { lat: number, lon: number }
): number {
  const R = 6371 // Earth radius in km
  const dLat = (p2.lat - p1.lat) * Math.PI / 180
  const dLon = (p2.lon - p1.lon) * Math.PI / 180
  const a = 
    Math.sin(dLat/2) * Math.sin(dLat/2) +
    Math.cos(p1.lat * Math.PI / 180) * Math.cos(p2.lat * Math.PI / 180) *
    Math.sin(dLon/2) * Math.sin(dLon/2)
  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a))
  return R * c
}
```

#### Advanced Custom Order: Dynamic Ordering

```typescript
import { SortedMap, Order, pipe } from "effect"

// Dynamic column sorting for tables
type SortDirection = 'asc' | 'desc'
type SortableColumns = 'name' | 'age' | 'salary' | 'department'

interface Employee {
  id: string
  name: string
  age: number
  salary: number
  department: string
}

class SortableTable {
  private data: Map<string, Employee> = new Map()
  private sortedViews: Map<string, SortedMap.SortedMap<Employee, string>> = new Map()
  
  createSortedView(
    column: SortableColumns,
    direction: SortDirection = 'asc'
  ): SortedMap.SortedMap<Employee, string> {
    const cacheKey = `${column}-${direction}`
    
    if (this.sortedViews.has(cacheKey)) {
      return this.sortedViews.get(cacheKey)!
    }
    
    const baseOrder = this.getOrderForColumn(column)
    const finalOrder = direction === 'desc' 
      ? Order.reverse(baseOrder) 
      : baseOrder
    
    const sorted = Array.from(this.data.values()).reduce(
      (acc, emp) => SortedMap.set(acc, emp, emp.id),
      SortedMap.empty<Employee, string>(finalOrder)
    )
    
    this.sortedViews.set(cacheKey, sorted)
    return sorted
  }
  
  private getOrderForColumn(column: SortableColumns): Order.Order<Employee> {
    switch (column) {
      case 'name':
        return Order.mapInput(Order.string, (e: Employee) => e.name)
      case 'age':
        return Order.mapInput(Order.number, (e: Employee) => e.age)
      case 'salary':
        return Order.mapInput(Order.number, (e: Employee) => e.salary)
      case 'department':
        return Order.combine(
          Order.mapInput(Order.string, (e: Employee) => e.department),
          Order.mapInput(Order.string, (e: Employee) => e.name)
        )
    }
  }
  
  // Multi-column sorting
  createMultiColumnView(
    sorts: Array<{ column: SortableColumns, direction: SortDirection }>
  ): SortedMap.SortedMap<Employee, string> {
    if (sorts.length === 0) {
      return SortedMap.empty(Order.mapInput(Order.string, (e: Employee) => e.id))
    }
    
    const orders = sorts.map(({ column, direction }) => {
      const base = this.getOrderForColumn(column)
      return direction === 'desc' ? Order.reverse(base) : base
    })
    
    const combinedOrder = orders.reduce((acc, order) => Order.combine(acc, order))
    
    return Array.from(this.data.values()).reduce(
      (acc, emp) => SortedMap.set(acc, emp, emp.id),
      SortedMap.empty<Employee, string>(combinedOrder)
    )
  }
  
  addEmployee(employee: Employee): void {
    this.data.set(employee.id, employee)
    this.sortedViews.clear() // Invalidate cache
  }
}

// Usage
const table = new SortableTable()

// Add employees
[
  { id: '1', name: 'Alice', age: 30, salary: 75000, department: 'Engineering' },
  { id: '2', name: 'Bob', age: 25, salary: 65000, department: 'Engineering' },
  { id: '3', name: 'Charlie', age: 35, salary: 85000, department: 'Sales' }
].forEach(emp => table.addEmployee(emp))

// Different sorted views
const byAge = table.createSortedView('age', 'asc')
const bySalaryDesc = table.createSortedView('salary', 'desc')

// Multi-column sort: department ascending, then salary descending
const complex = table.createMultiColumnView([
  { column: 'department', direction: 'asc' },
  { column: 'salary', direction: 'desc' }
])
```

## Practical Patterns & Best Practices

### Pattern 1: Efficient Bulk Operations

```typescript
import { SortedMap, Order, pipe, Chunk } from "effect"

// Batch insertion helper
function bulkInsert<K, V>(
  map: SortedMap.SortedMap<K, V>,
  entries: Iterable<[K, V]>
): SortedMap.SortedMap<K, V> {
  return Array.from(entries).reduce(
    (acc, [key, value]) => SortedMap.set(acc, key, value),
    map
  )
}

// Batch update with conflict resolution
function bulkUpdate<K, V>(
  map: SortedMap.SortedMap<K, V>,
  updates: Iterable<[K, V]>,
  onConflict: (existing: V, update: V) => V
): SortedMap.SortedMap<K, V> {
  return Array.from(updates).reduce(
    (acc, [key, value]) => pipe(
      acc,
      SortedMap.modifyOption(
        key,
        existing => onConflict(existing, value)
      ),
      Option.getOrElse(() => SortedMap.set(acc, key, value))
    ),
    map
  )
}

// Efficient merge of multiple sorted maps
function mergeSortedMaps<K, V>(
  order: Order.Order<K>,
  maps: Array<SortedMap.SortedMap<K, V>>,
  onConflict: (values: V[]) => V
): SortedMap.SortedMap<K, V> {
  // Collect all unique keys
  const keyMap = new Map<K, V[]>()
  
  for (const map of maps) {
    for (const [key, value] of map) {
      const existing = keyMap.get(key) || []
      keyMap.set(key, [...existing, value])
    }
  }
  
  // Build result with conflict resolution
  return Array.from(keyMap.entries()).reduce(
    (acc, [key, values]) => SortedMap.set(
      acc,
      key,
      values.length === 1 ? values[0] : onConflict(values)
    ),
    SortedMap.empty<K, V>(order)
  )
}

// Sliding window over sorted data
function* slidingWindow<K, V>(
  map: SortedMap.SortedMap<K, V>,
  windowSize: number
): Generator<Array<[K, V]>, void, unknown> {
  const entries = Array.from(map)
  
  for (let i = 0; i <= entries.length - windowSize; i++) {
    yield entries.slice(i, i + windowSize)
  }
}

// Usage example: Time-based aggregation
interface DataPoint {
  timestamp: Date
  value: number
}

const timeSeries = pipe(
  SortedMap.empty<Date, DataPoint>(Order.Date),
  map => bulkInsert(map, [
    [new Date('2024-01-01T10:00'), { timestamp: new Date('2024-01-01T10:00'), value: 100 }],
    [new Date('2024-01-01T11:00'), { timestamp: new Date('2024-01-01T11:00'), value: 110 }],
    [new Date('2024-01-01T12:00'), { timestamp: new Date('2024-01-01T12:00'), value: 105 }]
  ])
)

// Moving average calculation
const movingAverages = Array.from(slidingWindow(timeSeries, 2))
  .map(window => {
    const avg = window.reduce((sum, [, point]) => sum + point.value, 0) / window.length
    const midTime = window[Math.floor(window.length / 2)][0]
    return [midTime, avg] as const
  })
```

### Pattern 2: Memory-Efficient Large Collections

```typescript
import { SortedMap, Order, pipe, Option, Effect, Ref } from "effect"

// Paginated sorted map for large datasets
class PaginatedSortedMap<K, V> {
  private pages = new Map<number, SortedMap.SortedMap<K, V>>()
  private pageSize: number
  private order: Order.Order<K>
  private keyIndex = new Map<K, number>()
  
  constructor(pageSize: number, order: Order.Order<K>) {
    this.pageSize = pageSize
    this.order = order
  }
  
  set(key: K, value: V): PaginatedSortedMap<K, V> {
    const existingPage = this.keyIndex.get(key)
    
    if (existingPage !== undefined) {
      // Update existing entry
      const page = this.pages.get(existingPage) || SortedMap.empty(this.order)
      const updated = SortedMap.set(page, key, value)
      this.pages.set(existingPage, updated)
      return this
    }
    
    // Find appropriate page for new entry
    const targetPage = this.findTargetPage(key)
    const page = this.pages.get(targetPage) || SortedMap.empty(this.order)
    
    if (SortedMap.size(page) < this.pageSize) {
      // Add to existing page
      const updated = SortedMap.set(page, key, value)
      this.pages.set(targetPage, updated)
      this.keyIndex.set(key, targetPage)
    } else {
      // Split page
      this.splitPage(targetPage, key, value)
    }
    
    return this
  }
  
  get(key: K): Option.Option<V> {
    const pageNum = this.keyIndex.get(key)
    if (pageNum === undefined) return Option.none()
    
    const page = this.pages.get(pageNum)
    if (!page) return Option.none()
    
    return SortedMap.get(page, key)
  }
  
  private findTargetPage(key: K): number {
    // Binary search through pages
    const pageNums = Array.from(this.pages.keys()).sort((a, b) => a - b)
    
    for (const pageNum of pageNums) {
      const page = this.pages.get(pageNum)!
      const lastKey = pipe(
        SortedMap.last(page),
        Option.map(([k]) => k)
      )
      
      if (Option.isNone(lastKey) || Order.lessThanOrEqualTo(this.order)(key, lastKey.value)) {
        return pageNum
      }
    }
    
    return pageNums.length
  }
  
  private splitPage(pageNum: number, newKey: K, newValue: V): void {
    const page = this.pages.get(pageNum) || SortedMap.empty(this.order)
    const allEntries = [...Array.from(page), [newKey, newValue] as [K, V]]
      .sort(([a], [b]) => Order.compare(this.order)(a, b))
    
    const mid = Math.floor(allEntries.length / 2)
    const leftEntries = allEntries.slice(0, mid)
    const rightEntries = allEntries.slice(mid)
    
    // Update pages
    const leftPage = leftEntries.reduce(
      (acc, [k, v]) => SortedMap.set(acc, k, v),
      SortedMap.empty<K, V>(this.order)
    )
    const rightPage = rightEntries.reduce(
      (acc, [k, v]) => SortedMap.set(acc, k, v),
      SortedMap.empty<K, V>(this.order)
    )
    
    this.pages.set(pageNum, leftPage)
    this.pages.set(pageNum + 1, rightPage)
    
    // Update key index
    leftEntries.forEach(([k]) => this.keyIndex.set(k, pageNum))
    rightEntries.forEach(([k]) => this.keyIndex.set(k, pageNum + 1))
  }
  
  *iterate(): Generator<[K, V], void, unknown> {
    const pageNums = Array.from(this.pages.keys()).sort((a, b) => a - b)
    
    for (const pageNum of pageNums) {
      const page = this.pages.get(pageNum)!
      yield* page
    }
  }
}

// LRU cache backed by SortedMap
class LRUCache<K, V> {
  private data: SortedMap.SortedMap<{ key: K, timestamp: number }, V>
  private keyMap = new Map<K, number>()
  private counter = 0
  private maxSize: number
  
  constructor(maxSize: number) {
    this.maxSize = maxSize
    this.data = SortedMap.empty(
      Order.mapInput(
        Order.number,
        (entry: { key: K, timestamp: number }) => entry.timestamp
      )
    )
  }
  
  get(key: K): Option.Option<V> {
    const timestamp = this.keyMap.get(key)
    if (timestamp === undefined) return Option.none()
    
    // Update access time
    const entry = { key, timestamp }
    const value = pipe(
      Array.from(this.data),
      arr => arr.find(([k]) => k.key === key),
      opt => opt?.[1]
    )
    
    if (value !== undefined) {
      // Remove old entry and add with new timestamp
      this.data = pipe(
        this.data,
        map => {
          // Remove old
          const filtered = Array.from(map).filter(([k]) => k.key !== key)
          const newMap = filtered.reduce(
            (acc, [k, v]) => SortedMap.set(acc, k, v),
            SortedMap.empty(Order.mapInput(
              Order.number,
              (e: { key: K, timestamp: number }) => e.timestamp
            ))
          )
          // Add new
          return SortedMap.set(newMap, { key, timestamp: ++this.counter }, value)
        }
      )
      this.keyMap.set(key, this.counter)
    }
    
    return Option.fromNullable(value)
  }
  
  set(key: K, value: V): void {
    const newEntry = { key, timestamp: ++this.counter }
    this.keyMap.set(key, this.counter)
    
    // Remove existing entry if present
    this.data = pipe(
      Array.from(this.data).filter(([k]) => k.key !== key),
      entries => entries.reduce(
        (acc, [k, v]) => SortedMap.set(acc, k, v),
        SortedMap.empty(Order.mapInput(
          Order.number,
          (e: { key: K, timestamp: number }) => e.timestamp
        ))
      )
    )
    
    this.data = SortedMap.set(this.data, newEntry, value)
    
    // Evict oldest if over capacity
    if (SortedMap.size(this.data) > this.maxSize) {
      const oldest = SortedMap.head(this.data)
      if (Option.isSome(oldest)) {
        const [oldEntry] = oldest.value
        this.data = SortedMap.remove(this.data, oldEntry)
        this.keyMap.delete(oldEntry.key)
      }
    }
  }
}
```

## Integration Examples

### Integration with Effect Ecosystem

```typescript
import { SortedMap, Order, pipe, Effect, Option, Either, Schema } from "effect"

// Schema integration for validated sorted maps
const PriceSchema = Schema.number.pipe(
  Schema.positive(),
  Schema.lessThanOrEqual(1000000)
)

const ProductSchema = Schema.struct({
  id: Schema.string,
  name: Schema.string,
  price: PriceSchema,
  category: Schema.literal('electronics', 'clothing', 'food')
})

type Product = Schema.Schema.Type<typeof ProductSchema>

class ProductCatalog {
  private products: SortedMap.SortedMap<string, Product>
  
  constructor() {
    this.products = SortedMap.empty(Order.string)
  }
  
  addProduct(data: unknown): Effect.Effect<ProductCatalog, Schema.ParseError> {
    return pipe(
      Schema.decodeUnknown(ProductSchema)(data),
      Effect.map(product => {
        const updated = SortedMap.set(this.products, product.id, product)
        return new ProductCatalog().withProducts(updated)
      })
    )
  }
  
  updatePrice(
    id: string,
    newPrice: unknown
  ): Effect.Effect<ProductCatalog, Schema.ParseError | string> {
    return Effect.gen(function* () {
      const price = yield* Schema.decodeUnknown(PriceSchema)(newPrice)
      
      const productOption = SortedMap.get(this.products, id)
      const product = yield* Option.match(productOption, {
        onNone: () => Effect.fail(`Product ${id} not found`),
        onSome: product => Effect.succeed(product)
      })
      
      const updated = { ...product, price }
      const newProducts = SortedMap.set(this.products, id, updated)
      
      return new ProductCatalog().withProducts(newProducts)
    })
  }
  
  getByPriceRange(
    min: number,
    max: number
  ): Effect.Effect<ReadonlyArray<Product>, never> {
    // Create a temporary price-sorted map
    const byPrice = Array.from(this.products.values()).reduce(
      (acc, product) => SortedMap.set(acc, product.price, product),
      SortedMap.empty<number, Product>(Order.number)
    )
    
    return Effect.succeed(
      pipe(
        byPrice,
        SortedMap.between(min, max),
        Array.from,
        arr => arr.map(([, product]) => product)
      )
    )
  }
  
  private withProducts(products: SortedMap.SortedMap<string, Product>): ProductCatalog {
    this.products = products
    return this
  }
}

// Effect-based batch processing
const processPriceUpdates = (
  catalog: ProductCatalog,
  updates: Array<{ id: string, price: number }>
): Effect.Effect<ProductCatalog, Schema.ParseError | string> =>
  Effect.reduce(
    updates,
    catalog,
    (cat, update) => cat.updatePrice(update.id, update.price)
  )

// Stream integration for real-time sorted data
import { Stream } from "effect"

const createSortedStream = <T>(
  items: ReadonlyArray<T>,
  order: Order.Order<T>
): Stream.Stream<T, never, never> => {
  const sorted = [...items].sort(Order.compare(order))
  return Stream.fromIterable(sorted)
}

// Usage with metrics
interface Metric {
  name: string
  value: number
  timestamp: Date
}

const metricOrder = Order.struct({
  timestamp: Order.Date,
  name: Order.string
})

const processMetrics = (
  metrics: ReadonlyArray<Metric>
): Effect.Effect<void, never, never> =>
  pipe(
    createSortedStream(metrics, metricOrder),
    Stream.grouped(10),
    Stream.map(chunk =>
      chunk.reduce(
        (acc, metric) => SortedMap.set(acc, metric.timestamp, metric),
        SortedMap.empty<Date, Metric>(Order.Date)
      )
    ),
    Stream.runForEach(batch =>
      Effect.log(`Processed ${SortedMap.size(batch)} metrics`)
    )
  )
```

### Testing Strategies

```typescript
import { SortedMap, Order, pipe, Option } from "effect"
import { it, expect, describe } from "@effect/vitest"
import * as fc from "fast-check"

describe("SortedMap Testing", () => {
  // Property-based testing for ordering invariants
  it("maintains sort order after insertions", () => {
    fc.assert(
      fc.property(
        fc.array(fc.tuple(fc.string(), fc.integer())),
        (entries) => {
          const map = entries.reduce(
            (acc, [k, v]) => SortedMap.set(acc, k, v),
            SortedMap.empty<string, number>(Order.string)
          )
          
          const keys = Array.from(SortedMap.keys(map))
          const sorted = [...keys].sort()
          
          expect(keys).toEqual(sorted)
        }
      )
    )
  })
  
  // Test custom orderings
  it("respects custom ordering", () => {
    const reverseOrder = Order.reverse(Order.number)
    const map = pipe(
      SortedMap.empty<number, string>(reverseOrder),
      SortedMap.set(1, "one"),
      SortedMap.set(3, "three"),
      SortedMap.set(2, "two")
    )
    
    const keys = Array.from(SortedMap.keys(map))
    expect(keys).toEqual([3, 2, 1])
  })
  
  // Range query testing
  it("between returns correct range", () => {
    fc.assert(
      fc.property(
        fc.array(fc.integer({ min: 0, max: 100 })),
        fc.integer({ min: 0, max: 100 }),
        fc.integer({ min: 0, max: 100 }),
        (values, min, max) => {
          const [lower, upper] = min <= max ? [min, max] : [max, min]
          
          const map = values.reduce(
            (acc, v) => SortedMap.set(acc, v, v.toString()),
            SortedMap.empty<number, string>(Order.number)
          )
          
          const inRange = pipe(
            map,
            SortedMap.between(lower, upper),
            Array.from,
            arr => arr.map(([k]) => k)
          )
          
          // All values should be within range
          expect(inRange.every(k => k >= lower && k <= upper)).toBe(true)
          
          // Should include all values in range
          const expected = [...new Set(values)]
            .filter(v => v >= lower && v <= upper)
            .sort((a, b) => a - b)
          
          expect(inRange).toEqual(expected)
        }
      )
    )
  })
  
  // Structural sharing test
  it("shares structure between versions", () => {
    const map1 = pipe(
      SortedMap.empty<string, number>(Order.string),
      SortedMap.set("a", 1),
      SortedMap.set("b", 2),
      SortedMap.set("c", 3)
    )
    
    const map2 = SortedMap.set(map1, "d", 4)
    
    // Maps are different instances
    expect(map1).not.toBe(map2)
    
    // But share common structure (implementation detail)
    // This is more of a performance characteristic
    expect(SortedMap.size(map1)).toBe(3)
    expect(SortedMap.size(map2)).toBe(4)
  })
  
  // Benchmarking helper
  const benchmark = (name: string, fn: () => void, iterations = 1000) => {
    const start = performance.now()
    for (let i = 0; i < iterations; i++) {
      fn()
    }
    const end = performance.now()
    console.log(`${name}: ${(end - start) / iterations}ms per operation`)
  }
  
  // Performance comparison
  it.skip("performance characteristics", () => {
    const size = 10000
    const entries = Array.from({ length: size }, (_, i) => [i, i] as [number, number])
    
    // SortedMap construction
    benchmark("SortedMap construction", () => {
      entries.reduce(
        (acc, [k, v]) => SortedMap.set(acc, k, v),
        SortedMap.empty<number, number>(Order.number)
      )
    }, 10)
    
    // Native Map construction
    benchmark("Native Map construction", () => {
      new Map(entries)
    }, 10)
    
    const sortedMap = entries.reduce(
      (acc, [k, v]) => SortedMap.set(acc, k, v),
      SortedMap.empty<number, number>(Order.number)
    )
    
    // Range query
    benchmark("SortedMap range query", () => {
      SortedMap.between(sortedMap, size / 4, size * 3 / 4)
    }, 100)
    
    // Native Map range query (manual)
    const nativeMap = new Map(entries)
    benchmark("Native Map range query", () => {
      const result = []
      for (const [k, v] of nativeMap) {
        if (k >= size / 4 && k <= size * 3 / 4) {
          result.push([k, v])
        }
      }
    }, 100)
  })
})

// Test utilities
const generateSortedMap = <K, V>(
  keyArb: fc.Arbitrary<K>,
  valueArb: fc.Arbitrary<V>,
  order: Order.Order<K>
): fc.Arbitrary<SortedMap.SortedMap<K, V>> =>
  fc.array(fc.tuple(keyArb, valueArb)).map(entries =>
    entries.reduce(
      (acc, [k, v]) => SortedMap.set(acc, k, v),
      SortedMap.empty<K, V>(order)
    )
  )

// Testing custom data structures
interface CustomKey {
  primary: string
  secondary: number
}

const customKeyOrder = Order.struct({
  primary: Order.string,
  secondary: Order.number
})

describe("Custom key ordering", () => {
  it("orders by primary then secondary", () => {
    const map = pipe(
      SortedMap.empty<CustomKey, string>(customKeyOrder),
      SortedMap.set({ primary: "b", secondary: 1 }, "b1"),
      SortedMap.set({ primary: "a", secondary: 2 }, "a2"),
      SortedMap.set({ primary: "a", secondary: 1 }, "a1"),
      SortedMap.set({ primary: "b", secondary: 2 }, "b2")
    )
    
    const keys = Array.from(SortedMap.keys(map))
    expect(keys).toEqual([
      { primary: "a", secondary: 1 },
      { primary: "a", secondary: 2 },
      { primary: "b", secondary: 1 },
      { primary: "b", secondary: 2 }
    ])
  })
})
```

## Conclusion

SortedMap provides efficient ordered key-value storage, automatic sorting maintenance, and powerful range query capabilities for building high-performance applications with complex ordering requirements.

Key benefits:
- **Automatic Ordering**: Keys stay sorted according to your custom comparator without manual intervention
- **Efficient Operations**: O(log n) performance for most operations with structural sharing for memory efficiency
- **Type-Safe Comparators**: Order instances ensure type safety while enabling complex sorting strategies
- **Range Query Power**: Built-in support for between, greater than, and less than queries essential for real-world applications

SortedMap is the ideal choice when you need persistent ordered collections with efficient range queries, custom sorting strategies, or index-like data structures in your Effect applications.