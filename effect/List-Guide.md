# List: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem List Solves

Arrays in JavaScript are mutable, index-based, and optimized for random access. However, many functional programming patterns work better with immutable, recursive data structures. Traditional array operations often lead to performance issues when dealing with prepending, recursive algorithms, or persistent data structures:

```typescript
// Traditional approach - inefficient recursive processing with arrays
function processNestedStructure(items: Item[], accumulator: Result[] = []): Result[] {
  if (items.length === 0) return accumulator;
  
  const [head, ...tail] = items; // O(n) operation - creates new array
  
  // Process head element
  const processed = transformItem(head);
  
  // Prepending is O(n) with arrays
  const newAccumulator = [processed, ...accumulator]; // Another O(n) operation
  
  // Recursive call with array destructuring is inefficient
  return processNestedStructure(tail, newAccumulator);
}

// Stack-unsafe recursion with arrays
function findPath(graph: Node[], target: string, visited: string[] = []): string[] | null {
  for (const node of graph) {
    if (visited.includes(node.id)) continue; // O(n) lookup
    
    if (node.id === target) {
      return [...visited, node.id]; // O(n) array copy
    }
    
    const newVisited = [...visited, node.id]; // Another O(n) copy
    const path = findPath(node.children, target, newVisited);
    
    if (path) return path;
  }
  
  return null;
}
```

This approach leads to:
- **Performance degradation** - Array destructuring and spreading create O(n) copies
- **Stack overflow risks** - No tail-call optimization for recursive algorithms
- **Memory inefficiency** - Copying arrays for immutability is expensive
- **Poor prepend performance** - Arrays are optimized for append, not prepend operations

### The List Solution

Effect's List module provides an immutable, persistent linked list that excels at functional programming patterns, recursive algorithms, and prepend operations:

```typescript
import { List, Option } from "effect"

// Efficient recursive processing with List
const processNestedStructure = <A, B>(
  items: List.List<A>,
  transform: (a: A) => B
): List.List<B> => {
  return List.reverse(
    List.reduce(items, List.empty<B>(), (acc, item) => 
      List.prepend(acc, transform(item)) // O(1) prepend operation
    )
  )
}

// Stack-safe recursion with List
const findPath = (
  graph: List.List<Node>,
  target: string
): Option.Option<List.List<string>> => {
  const go = (
    nodes: List.List<Node>,
    visited: List.List<string>
  ): Option.Option<List.List<string>> => {
    return List.findFirst(nodes, (node) => {
      if (List.some(visited, (id) => id === node.id)) {
        return Option.none()
      }
      
      if (node.id === target) {
        return Option.some(List.prepend(visited, node.id))
      }
      
      return go(node.children, List.prepend(visited, node.id))
    })
  }
  
  return go(graph, List.empty())
}
```

### Key Concepts

**Persistent Data Structure**: List is immutable - operations return new lists without modifying the original, enabling safe sharing and time-travel debugging.

**Head/Tail Decomposition**: Lists are either empty or consist of a head element and a tail list, perfect for recursive algorithms and pattern matching.

**Structural Sharing**: Lists share common suffixes, making operations like prepend extremely memory-efficient.

**Lazy Evaluation**: Many List operations are lazy, allowing efficient composition without intermediate allocations.

## Basic Usage Patterns

### Creating and Converting Lists

```typescript
import { List, ReadonlyArray } from "effect"

// Create empty list
const empty = List.empty<number>()

// Create list from elements
const numbers = List.make(1, 2, 3, 4, 5)

// Create list from array
const fromArray = List.fromIterable([1, 2, 3, 4, 5])

// Convert list to array
const toArray = List.toArray(numbers) // [1, 2, 3, 4, 5]

// Create list with cons (prepend)
const consed = List.cons(0, numbers) // List(0, 1, 2, 3, 4, 5)

// Create list with range
const range = List.range(1, 10) // List(1, 2, 3, ..., 10)
```

### Basic Operations

```typescript
import { List, Option, pipe } from "effect"

const users = List.make(
  { id: 1, name: "Alice", age: 30 },
  { id: 2, name: "Bob", age: 25 },
  { id: 3, name: "Charlie", age: 35 }
)

// Head and tail access
const first = List.head(users) // Option.some({ id: 1, name: "Alice", age: 30 })
const rest = List.tail(users) // Option.some(List({ id: 2, ... }, { id: 3, ... }))

// Safe indexing
const secondUser = List.get(users, 1) // Option.some({ id: 2, name: "Bob", age: 25 })
const outOfBounds = List.get(users, 10) // Option.none()

// Length and emptiness
const count = List.length(users) // 3
const isEmpty = List.isEmpty(users) // false
const isNonEmpty = List.isNonEmpty(users) // true
```

### Transformation Operations

```typescript
import { List, pipe } from "effect"

interface Product {
  id: number
  name: string
  price: number
  category: string
}

const products = List.make(
  { id: 1, name: "Laptop", price: 999, category: "Electronics" },
  { id: 2, name: "Coffee", price: 5, category: "Food" },
  { id: 3, name: "Book", price: 20, category: "Education" },
  { id: 4, name: "Phone", price: 699, category: "Electronics" }
)

// Map transformation
const productNames = pipe(
  List.map((p) => p.name)(products)
) // List("Laptop", "Coffee", "Book", "Phone")

// Filter operation
const electronics = pipe(
  List.filter((p) => p.category === "Electronics")(products)
) // List(laptop, phone)

// FlatMap for nested transformations
const expandedProducts = pipe(
  List.flatMap((p) => 
    List.make(
      { ...p, variant: "Standard" },
      { ...p, variant: "Premium", price: p.price * 1.5 }
    )
  )(products)
)
```

## Real-World Examples

### Example 1: Task Queue Processing

A common pattern in applications is processing tasks in FIFO order while maintaining history:

```typescript
import { List, Option, Effect, pipe } from "effect"

interface Task {
  id: string
  priority: "low" | "medium" | "high"
  action: () => Effect.Effect<void>
  createdAt: Date
}

class TaskQueue {
  private pending = List.empty<Task>()
  private completed = List.empty<Task>()
  private failed = List.empty<{ task: Task; error: unknown }>()

  enqueue(task: Task): void {
    // O(1) prepend for FIFO via reverse
    this.pending = List.prepend(this.pending, task)
  }

  enqueueBatch(tasks: ReadonlyArray<Task>): void {
    this.pending = List.prependAll(this.pending, List.fromIterable(tasks))
  }

  async processNext(): Promise<Option.Option<Task>> {
    // Get tasks in FIFO order
    const fifoTasks = List.reverse(this.pending)
    const next = List.head(fifoTasks)

    if (Option.isNone(next)) {
      return Option.none()
    }

    const task = next.value
    const remaining = List.tail(fifoTasks)
    this.pending = Option.getOrElse(remaining, () => List.empty<Task>())

    try {
      await Effect.runPromise(task.action())
      this.completed = List.prepend(this.completed, task)
      return Option.some(task)
    } catch (error) {
      this.failed = List.prepend(this.failed, { task, error })
      throw error
    }
  }

  async processAll(): Promise<void> {
    while (List.isNonEmpty(this.pending)) {
      await this.processNext()
    }
  }

  getHistory(): ReadonlyArray<Task> {
    return List.toArray(List.reverse(this.completed))
  }

  getFailures(): ReadonlyArray<{ task: Task; error: unknown }> {
    return List.toArray(List.reverse(this.failed))
  }

  getPendingByPriority(): Record<Task["priority"], ReadonlyArray<Task>> {
    return pipe(
      List.groupBy((task) => task.priority)(List.reverse(this.pending)),
      (groups) => ({
        high: List.toArray(groups.get("high") ?? List.empty()),
        medium: List.toArray(groups.get("medium") ?? List.empty()),
        low: List.toArray(groups.get("low") ?? List.empty())
      })
    )
  }
}

// Usage
const queue = new TaskQueue()

queue.enqueueBatch([
  {
    id: "task-1",
    priority: "high",
    action: () => Effect.log("Processing high priority task"),
    createdAt: new Date()
  },
  {
    id: "task-2",
    priority: "low",
    action: () => Effect.log("Processing low priority task"),
    createdAt: new Date()
  }
])

await queue.processAll()
console.log("Completed tasks:", queue.getHistory())
```

### Example 2: Undo/Redo System

Lists excel at implementing undo/redo functionality due to their efficient prepend operations:

```typescript
import { List, Option, pipe } from "effect"

interface Command<T> {
  execute: (state: T) => T
  undo: (state: T) => T
  description: string
}

class UndoRedoStack<T> {
  private undoStack = List.empty<Command<T>>()
  private redoStack = List.empty<Command<T>>()
  
  constructor(private state: T) {}

  execute(command: Command<T>): T {
    // Execute command
    this.state = command.execute(this.state)
    
    // Add to undo stack (O(1))
    this.undoStack = List.prepend(this.undoStack, command)
    
    // Clear redo stack
    this.redoStack = List.empty()
    
    return this.state
  }

  undo(): Option.Option<T> {
    return pipe(
      List.head(this.undoStack),
      Option.map((command) => {
        // Remove from undo stack
        this.undoStack = Option.getOrElse(
          List.tail(this.undoStack),
          () => List.empty<Command<T>>()
        )
        
        // Apply undo
        this.state = command.undo(this.state)
        
        // Add to redo stack
        this.redoStack = List.prepend(this.redoStack, command)
        
        return this.state
      })
    )
  }

  redo(): Option.Option<T> {
    return pipe(
      List.head(this.redoStack),
      Option.map((command) => {
        // Remove from redo stack
        this.redoStack = Option.getOrElse(
          List.tail(this.redoStack),
          () => List.empty<Command<T>>()
        )
        
        // Apply redo
        this.state = command.execute(this.state)
        
        // Add back to undo stack
        this.undoStack = List.prepend(this.undoStack, command)
        
        return this.state
      })
    )
  }

  getState(): T {
    return this.state
  }

  getHistory(): ReadonlyArray<string> {
    return pipe(
      List.reverse(this.undoStack),
      List.map((cmd) => cmd.description),
      List.toArray
    )
  }

  canUndo(): boolean {
    return List.isNonEmpty(this.undoStack)
  }

  canRedo(): boolean {
    return List.isNonEmpty(this.redoStack)
  }
}

// Example: Text editor with undo/redo
interface EditorState {
  content: string
  cursor: number
}

const insertText = (text: string, position: number): Command<EditorState> => ({
  execute: (state) => ({
    content: state.content.slice(0, position) + text + state.content.slice(position),
    cursor: position + text.length
  }),
  undo: (state) => ({
    content: state.content.slice(0, position) + state.content.slice(position + text.length),
    cursor: position
  }),
  description: `Insert "${text}" at position ${position}`
})

const deleteText = (start: number, length: number): Command<EditorState> => {
  let deletedText = ""
  
  return {
    execute: (state) => {
      deletedText = state.content.slice(start, start + length)
      return {
        content: state.content.slice(0, start) + state.content.slice(start + length),
        cursor: start
      }
    },
    undo: (state) => ({
      content: state.content.slice(0, start) + deletedText + state.content.slice(start),
      cursor: start + deletedText.length
    }),
    description: `Delete ${length} characters at position ${start}`
  }
}

// Usage
const editor = new UndoRedoStack<EditorState>({ content: "", cursor: 0 })

editor.execute(insertText("Hello ", 0))
editor.execute(insertText("World", 6))
editor.execute(deleteText(5, 1)) // Delete space

console.log(editor.getState()) // { content: "HelloWorld", cursor: 5 }

Option.match(editor.undo(), {
  onNone: () => console.log("Nothing to undo"),
  onSome: (state) => console.log("After undo:", state) // { content: "Hello World", cursor: 6 }
})

Option.match(editor.redo(), {
  onNone: () => console.log("Nothing to redo"),
  onSome: (state) => console.log("After redo:", state) // { content: "HelloWorld", cursor: 5 }
})
```

### Example 3: Tree Traversal with Path Tracking

Lists are ideal for tracking paths through tree structures:

```typescript
import { List, Option, Either, pipe } from "effect"

interface FileNode {
  name: string
  type: "file" | "directory"
  children: List.List<FileNode>
  size: number
}

class FileSystem {
  constructor(private root: FileNode) {}

  findFile(fileName: string): Option.Option<List.List<string>> {
    const search = (
      node: FileNode,
      path: List.List<string>
    ): Option.Option<List.List<string>> => {
      const currentPath = List.prepend(path, node.name)
      
      if (node.type === "file" && node.name === fileName) {
        return Option.some(List.reverse(currentPath))
      }
      
      if (node.type === "directory") {
        return pipe(
          node.children,
          List.findFirstMap((child) => search(child, currentPath))
        )
      }
      
      return Option.none()
    }
    
    return search(this.root, List.empty())
  }

  getAllPaths(): List.List<string> {
    const collect = (
      node: FileNode,
      path: List.List<string>
    ): List.List<string> => {
      const currentPath = List.prepend(path, node.name)
      const pathString = pipe(
        currentPath,
        List.reverse,
        List.join("/")
      )
      
      if (node.type === "file") {
        return List.of(pathString)
      }
      
      return pipe(
        List.of(pathString),
        List.appendAll(
          pipe(
            node.children,
            List.flatMap((child) => collect(child, currentPath))
          )
        )
      )
    }
    
    return collect(this.root, List.empty())
  }

  calculateSize(path: string): Either.Either<number, string> {
    const pathParts = path.split("/").filter(Boolean)
    
    const navigate = (
      node: FileNode,
      remaining: List.List<string>
    ): Either.Either<number, string> => {
      if (List.isEmpty(remaining)) {
        return Either.right(calculateNodeSize(node))
      }
      
      return pipe(
        List.head(remaining),
        Option.flatMap((nextPart) =>
          List.findFirst(node.children, (child) => child.name === nextPart)
        ),
        Option.match({
          onNone: () => Either.left(`Path not found: ${path}`),
          onSome: (child) => navigate(
            child,
            Option.getOrElse(List.tail(remaining), () => List.empty())
          )
        })
      )
    }
    
    const calculateNodeSize = (node: FileNode): number => {
      if (node.type === "file") return node.size
      
      return pipe(
        node.children,
        List.reduce(0, (acc, child) => acc + calculateNodeSize(child))
      )
    }
    
    return navigate(this.root, List.fromIterable(pathParts))
  }
}

// Example usage
const fileSystem = new FileSystem({
  name: "root",
  type: "directory",
  size: 0,
  children: List.make(
    {
      name: "src",
      type: "directory",
      size: 0,
      children: List.make(
        { name: "index.ts", type: "file", size: 1024, children: List.empty() },
        { name: "utils.ts", type: "file", size: 512, children: List.empty() }
      )
    },
    {
      name: "docs",
      type: "directory", 
      size: 0,
      children: List.make(
        { name: "README.md", type: "file", size: 2048, children: List.empty() }
      )
    }
  )
})

// Find a file
Option.match(fileSystem.findFile("utils.ts"), {
  onNone: () => console.log("File not found"),
  onSome: (path) => console.log("Found at:", List.toArray(path)) // ["root", "src", "utils.ts"]
})

// Get all paths
const allPaths = List.toArray(fileSystem.getAllPaths())
console.log("All paths:", allPaths)

// Calculate directory size
Either.match(fileSystem.calculateSize("root/src"), {
  onLeft: (error) => console.error(error),
  onRight: (size) => console.log("Size of src:", size) // 1536
})
```

### Example 4: Event Sourcing with Lists

Event sourcing patterns benefit from List's immutable append operations and efficient replay:

```typescript
import { List, Option, Effect, pipe, DateTime, Order } from "effect"

interface Event {
  id: string
  type: string
  timestamp: DateTime.DateTime
  payload: unknown
}

interface Snapshot<T> {
  version: number
  state: T
  timestamp: DateTime.DateTime
}

class EventStore<T> {
  private events = List.empty<Event>()
  private snapshots = List.empty<Snapshot<T>>()
  
  constructor(
    private readonly reducer: (state: T, event: Event) => T,
    private readonly initialState: T,
    private readonly snapshotInterval: number = 100
  ) {}

  append(event: Event): T {
    // Append event (O(1) with reverse later)
    this.events = List.prepend(this.events, event)
    
    // Check if we need a snapshot
    if (List.length(this.events) % this.snapshotInterval === 0) {
      this.createSnapshot()
    }
    
    return this.getCurrentState()
  }

  appendBatch(events: ReadonlyArray<Event>): T {
    // Efficient batch append
    this.events = List.prependAll(
      this.events,
      List.reverse(List.fromIterable(events))
    )
    
    const newEventCount = List.length(this.events)
    const lastSnapshotCount = Option.match(List.head(this.snapshots), {
      onNone: () => 0,
      onSome: (snapshot) => snapshot.version
    })
    
    if (newEventCount - lastSnapshotCount >= this.snapshotInterval) {
      this.createSnapshot()
    }
    
    return this.getCurrentState()
  }

  private createSnapshot(): void {
    const state = this.getCurrentState()
    const version = List.length(this.events)
    
    this.snapshots = List.prepend(this.snapshots, {
      version,
      state,
      timestamp: DateTime.now
    })
    
    // Keep only last 5 snapshots
    this.snapshots = List.take(this.snapshots, 5)
  }

  getCurrentState(): T {
    // Find the most recent snapshot
    const snapshot = List.head(this.snapshots)
    
    return Option.match(snapshot, {
      onNone: () => {
        // No snapshot, replay all events
        return pipe(
          List.reduce(this.initialState, this.reducer)(List.reverse(this.events))
        )
      },
      onSome: (snap) => {
        // Replay events after snapshot
        const eventsAfterSnapshot = pipe(
          List.take(List.length(this.events) - snap.version)(List.reverse(this.events))
        )
        
        return pipe(
          List.reduce(snap.state, this.reducer)(eventsAfterSnapshot)
        )
      }
    })
  }

  getEventHistory(
    since?: DateTime.DateTime,
    until?: DateTime.DateTime
  ): List.List<Event> {
    return pipe(
      List.filter((event) => {
        if (since && DateTime.lessThan(event.timestamp, since)) return false
        if (until && DateTime.greaterThan(event.timestamp, until)) return false
        return true
      })(List.reverse(this.events))
    )
  }

  replayFrom(timestamp: DateTime.DateTime): T {
    const relevantEvents = pipe(
      List.filter((event) => DateTime.greaterThanOrEqualTo(event.timestamp, timestamp))(List.reverse(this.events))
    )
    
    return List.reduce(relevantEvents, this.initialState, this.reducer)
  }

  fork(): EventStore<T> {
    const forked = new EventStore(
      this.reducer,
      this.initialState,
      this.snapshotInterval
    )
    
    // Share immutable event history
    forked.events = this.events
    forked.snapshots = this.snapshots
    
    return forked
  }
}

// Example: Order management system
interface OrderState {
  orders: Map<string, Order>
  revenue: number
  itemsSold: number
}

interface Order {
  id: string
  customerId: string
  items: List.List<OrderItem>
  status: "pending" | "confirmed" | "shipped" | "delivered" | "cancelled"
  total: number
}

interface OrderItem {
  productId: string
  quantity: number
  price: number
}

type OrderEvent = 
  | { type: "OrderCreated"; orderId: string; customerId: string; items: OrderItem[] }
  | { type: "OrderConfirmed"; orderId: string }
  | { type: "OrderShipped"; orderId: string; trackingNumber: string }
  | { type: "OrderDelivered"; orderId: string }
  | { type: "OrderCancelled"; orderId: string; reason: string }

const orderReducer = (state: OrderState, event: Event): OrderState => {
  const payload = event.payload as OrderEvent
  
  switch (payload.type) {
    case "OrderCreated": {
      const order: Order = {
        id: payload.orderId,
        customerId: payload.customerId,
        items: List.fromIterable(payload.items),
        status: "pending",
        total: payload.items.reduce((sum, item) => sum + item.price * item.quantity, 0)
      }
      
      return {
        ...state,
        orders: new Map(state.orders).set(order.id, order)
      }
    }
    
    case "OrderConfirmed": {
      const order = state.orders.get(payload.orderId)
      if (!order) return state
      
      const updatedOrder = { ...order, status: "confirmed" as const }
      
      return {
        ...state,
        orders: new Map(state.orders).set(order.id, updatedOrder),
        revenue: state.revenue + order.total,
        itemsSold: state.itemsSold + List.length(order.items)
      }
    }
    
    // ... other event handlers
    
    default:
      return state
  }
}

// Usage
const orderStore = new EventStore<OrderState>(
  orderReducer,
  { orders: new Map(), revenue: 0, itemsSold: 0 },
  50 // Snapshot every 50 events
)

// Process events
orderStore.append({
  id: "evt-1",
  type: "OrderCreated",
  timestamp: DateTime.now,
  payload: {
    type: "OrderCreated",
    orderId: "order-123",
    customerId: "customer-456",
    items: [
      { productId: "prod-1", quantity: 2, price: 29.99 },
      { productId: "prod-2", quantity: 1, price: 49.99 }
    ]
  }
})

// Get analytics for a time period
const lastWeekEvents = orderStore.getEventHistory(
  DateTime.subtract(DateTime.now, { days: 7 }),
  DateTime.now
)

// Fork for what-if analysis
const analysisStore = orderStore.fork()
```

### Example 5: Graph Algorithms with Lists

Lists excel at graph traversal algorithms where paths need to be tracked:

```typescript
import { List, Option, HashSet, HashMap, pipe, Equal } from "effect"

interface Graph<T> {
  vertices: HashSet.HashSet<T>
  edges: HashMap.HashMap<T, List.List<T>>
}

class GraphAlgorithms<T> {
  constructor(
    private readonly graph: Graph<T>,
    private readonly equals: Equal.Equal<T>
  ) {}

  // Depth-first search with path tracking
  dfs(start: T, target: T): Option.Option<List.List<T>> {
    const visited = new Set<T>()
    
    const search = (current: T, path: List.List<T>): Option.Option<List.List<T>> => {
      if (this.equals(current, target)) {
        return Option.some(List.reverse(List.prepend(path, current)))
      }
      
      visited.add(current)
      const currentPath = List.prepend(path, current)
      
      const neighbors = HashMap.get(this.graph.edges, current)
      
      return Option.flatMap(neighbors, (neighborList) =>
        List.findFirstMap(neighborList, (neighbor) => {
          if (visited.has(neighbor)) return Option.none()
          return search(neighbor, currentPath)
        })
      )
    }
    
    return search(start, List.empty())
  }

  // Find all paths between two vertices
  allPaths(start: T, target: T, maxLength?: number): List.List<List.List<T>> {
    const findPaths = (
      current: T,
      path: List.List<T>,
      visited: HashSet.HashSet<T>
    ): List.List<List.List<T>> => {
      if (this.equals(current, target)) {
        return List.of(List.reverse(List.prepend(path, current)))
      }
      
      if (maxLength && List.length(path) >= maxLength) {
        return List.empty()
      }
      
      const newVisited = HashSet.add(visited, current)
      const currentPath = List.prepend(path, current)
      
      return pipe(
        HashMap.get(this.graph.edges, current),
        Option.getOrElse(() => List.empty<T>()),
        List.filter((neighbor) => !HashSet.has(newVisited, neighbor)),
        List.flatMap((neighbor) => findPaths(neighbor, currentPath, newVisited))
      )
    }
    
    return findPaths(start, List.empty(), HashSet.empty())
  }

  // Topological sort using DFS
  topologicalSort(): Option.Option<List.List<T>> {
    const visited = new Set<T>()
    const finished = new Set<T>()
    let sorted = List.empty<T>()
    let hasCycle = false
    
    const visit = (vertex: T): void => {
      if (finished.has(vertex)) return
      if (visited.has(vertex)) {
        hasCycle = true
        return
      }
      
      visited.add(vertex)
      
      const neighbors = HashMap.get(this.graph.edges, vertex)
      Option.match(neighbors, {
        onNone: () => {},
        onSome: (neighborList) => {
          List.forEach(neighborList, visit)
        }
      })
      
      finished.add(vertex)
      sorted = List.prepend(sorted, vertex)
    }
    
    HashSet.forEach(this.graph.vertices, visit)
    
    return hasCycle ? Option.none() : Option.some(sorted)
  }

  // Find strongly connected components
  stronglyConnectedComponents(): List.List<List.List<T>> {
    const index = new Map<T, number>()
    const lowlink = new Map<T, number>()
    const onStack = new Set<T>()
    let stack = List.empty<T>()
    let currentIndex = 0
    let components = List.empty<List.List<T>>()
    
    const strongConnect = (v: T): void => {
      index.set(v, currentIndex)
      lowlink.set(v, currentIndex)
      currentIndex++
      stack = List.prepend(stack, v)
      onStack.add(v)
      
      const neighbors = HashMap.get(this.graph.edges, v)
      Option.match(neighbors, {
        onNone: () => {},
        onSome: (neighborList) => {
          List.forEach(neighborList, (w) => {
            if (!index.has(w)) {
              strongConnect(w)
              lowlink.set(v, Math.min(lowlink.get(v)!, lowlink.get(w)!))
            } else if (onStack.has(w)) {
              lowlink.set(v, Math.min(lowlink.get(v)!, index.get(w)!))
            }
          })
        }
      })
      
      if (lowlink.get(v) === index.get(v)) {
        let component = List.empty<T>()
        let w: T
        
        do {
          const popped = List.head(stack)
          if (Option.isNone(popped)) break
          
          w = popped.value
          stack = Option.getOrElse(List.tail(stack), () => List.empty<T>())
          onStack.delete(w)
          component = List.prepend(component, w)
        } while (!this.equals(w!, v))
        
        if (List.isNonEmpty(component)) {
          components = List.prepend(components, component)
        }
      }
    }
    
    HashSet.forEach(this.graph.vertices, (v) => {
      if (!index.has(v)) {
        strongConnect(v)
      }
    })
    
    return components
  }
}

// Example: Dependency resolution
interface Package {
  name: string
  version: string
}

const packageEquals: Equal.Equal<Package> = Equal.make((a, b) => 
  a.name === b.name && a.version === b.version
)

const buildDependencyGraph = (
  packages: List.List<Package>,
  dependencies: Map<string, List.List<string>>
): Graph<Package> => {
  const vertices = HashSet.fromIterable(packages)
  const edges = HashMap.empty<Package>()
  
  return List.reduce(packages, { vertices, edges }, (graph, pkg) => {
    const deps = dependencies.get(pkg.name) ?? []
    const depPackages = pipe(
      List.fromIterable(deps),
      List.filterMap((depName) =>
        List.findFirst(packages, (p) => p.name === depName)
      )
    )
    
    return {
      ...graph,
      edges: HashMap.set(graph.edges, pkg, depPackages)
    }
  })
}

// Usage
const packages = List.make(
  { name: "react", version: "18.0.0" },
  { name: "react-dom", version: "18.0.0" },
  { name: "webpack", version: "5.0.0" },
  { name: "babel", version: "7.0.0" }
)

const dependencies = new Map([
  ["react-dom", ["react"]],
  ["webpack", ["babel"]],
  ["babel", []]
])

const depGraph = buildDependencyGraph(packages, dependencies)
const algorithms = new GraphAlgorithms(depGraph, packageEquals)

// Find installation order
Option.match(algorithms.topologicalSort(), {
  onNone: () => console.log("Circular dependency detected!"),
  onSome: (order) => {
    console.log("Installation order:", 
      pipe(order, List.map((p) => p.name), List.toArray)
    )
  }
})
```

## Advanced Features Deep Dive

### Recursive Algorithms with Stack Safety

Effect's List provides stack-safe operations for recursive algorithms that would overflow with regular recursion:

```typescript
import { List, pipe } from "effect"

// Stack-safe factorial using List accumulator
const factorial = (n: number): bigint => {
  const numbers = List.range(1, n)
  
  return pipe(
    List.reduce(1n, (acc, num) => acc * BigInt(num))(numbers)
  )
}

// Stack-safe Fibonacci sequence
const fibonacci = (n: number): List.List<number> => {
  const go = (
    count: number,
    current: number,
    next: number,
    acc: List.List<number>
  ): List.List<number> => {
    if (count === 0) return List.reverse(acc)
    
    return go(
      count - 1,
      next,
      current + next,
      List.prepend(acc, current)
    )
  }
  
  return go(n, 0, 1, List.empty())
}

// Stack-safe tree flattening
interface TreeNode<T> {
  value: T
  children: List.List<TreeNode<T>>
}

const flattenTree = <T>(tree: TreeNode<T>): List.List<T> => {
  // Using a work list pattern for stack safety
  const go = (
    workList: List.List<TreeNode<T>>,
    acc: List.List<T>
  ): List.List<T> => {
    if (List.isEmpty(workList)) return acc
    
    return pipe(
      List.head(workList),
      Option.match({
        onNone: () => acc,
        onSome: (node) => {
          const remaining = Option.getOrElse(
            List.tail(workList),
            () => List.empty<TreeNode<T>>()
          )
          const newWorkList = List.prependAll(remaining, node.children)
          return go(newWorkList, List.prepend(acc, node.value))
        }
      })
    )
  }
  
  return List.reverse(go(List.of(tree), List.empty()))
}

// Example: Deep tree traversal
const deepTree: TreeNode<number> = {
  value: 1,
  children: List.make(
    {
      value: 2,
      children: List.make(
        { value: 4, children: List.empty() },
        { value: 5, children: List.empty() }
      )
    },
    {
      value: 3,
      children: List.make(
        { value: 6, children: List.empty() }
      )
    }
  )
}

console.log("Flattened tree:", List.toArray(flattenTree(deepTree))) // [1, 2, 4, 5, 3, 6]
```

### Performance Optimization Patterns

Understanding when to use List vs Array for optimal performance:

```typescript
import { List, Array, pipe, Effect, Duration } from "effect"

// List excels at prepend-heavy operations
const buildListPrepend = (n: number): Effect.Effect<List.List<number>> =>
  Effect.sync(() => {
    let list = List.empty<number>()
    for (let i = 0; i < n; i++) {
      list = List.prepend(list, i) // O(1) per operation
    }
    return list
  })

// Array struggles with prepend
const buildArrayPrepend = (n: number): Effect.Effect<Array<number>> =>
  Effect.sync(() => {
    let arr: number[] = []
    for (let i = 0; i < n; i++) {
      arr = [i, ...arr] // O(n) per operation - quadratic overall!
    }
    return arr
  })

// List with proper batching for append operations
const buildListAppendBatched = (n: number): Effect.Effect<List.List<number>> =>
  Effect.sync(() => {
    const chunks: List.List<number>[] = []
    let current = List.empty<number>()
    
    for (let i = 0; i < n; i++) {
      current = List.prepend(current, i)
      
      // Batch reversal for efficiency
      if (List.length(current) >= 1000) {
        chunks.push(List.reverse(current))
        current = List.empty()
      }
    }
    
    if (List.isNonEmpty(current)) {
      chunks.push(List.reverse(current))
    }
    
    return chunks.reduce(
      (acc, chunk) => List.appendAll(acc, chunk),
      List.empty<number>()
    )
  })

// Benchmark utility
const benchmark = <A>(name: string, effect: Effect.Effect<A>) =>
  pipe(
    Effect.timed(effect),
    Effect.map(([duration, _]) => {
      console.log(`${name}: ${Duration.toMillis(duration)}ms`)
    })
  )

// Compare performance
const runBenchmarks = Effect.gen(function* () {
  const size = 10000
  
  yield* benchmark("List prepend", buildListPrepend(size))
  yield* benchmark("Array prepend", buildArrayPrepend(size))
  yield* benchmark("List append (batched)", buildListAppendBatched(size))
})
```

### Advanced Combinators and Patterns

List provides powerful combinators for complex transformations:

```typescript
import { List, Option, pipe, Predicate } from "effect"

// Sliding window pattern
const slidingWindow = <A>(list: List.List<A>, size: number): List.List<List.List<A>> => {
  if (size <= 0 || List.isEmpty(list)) return List.empty()
  
  const go = (
    remaining: List.List<A>,
    acc: List.List<List.List<A>>
  ): List.List<List.List<A>> => {
    const window = List.take(remaining, size)
    
    if (List.length(window) < size) {
      return List.reverse(acc)
    }
    
    return pipe(
      List.tail(remaining),
      Option.match({
        onNone: () => List.reverse(List.prepend(acc, window)),
        onSome: (tail) => go(tail, List.prepend(acc, window))
      })
    )
  }
  
  return go(list, List.empty())
}

// Group consecutive elements
const groupConsecutive = <A>(
  list: List.List<A>,
  equals: (a: A, b: A) => boolean
): List.List<List.List<A>> => {
  if (List.isEmpty(list)) return List.empty()
  
  return pipe(
    List.reduce(
      list,
      List.empty<List.List<A>>(),
      (groups, item) => {
        return pipe(
          List.head(groups),
          Option.match({
            onNone: () => List.of(List.of(item)),
            onSome: (currentGroup) =>
              pipe(
                List.head(currentGroup),
                Option.match({
                  onNone: () => List.prepend(groups, List.of(item)),
                  onSome: (lastItem) =>
                    equals(lastItem, item)
                      ? List.prepend(
                          Option.getOrElse(List.tail(groups), () => List.empty()),
                          List.prepend(currentGroup, item)
                        )
                      : List.prepend(groups, List.of(item))
                })
              )
          })
        )
      }
    ),
    List.map(List.reverse),
    List.reverse
  )
}

// Partition with multiple predicates
const multiPartition = <A>(
  list: List.List<A>,
  predicates: ReadonlyArray<Predicate.Predicate<A>>
): ReadonlyArray<List.List<A>> => {
  const initial = predicates.map(() => List.empty<A>())
  
  return pipe(
    list,
    List.reduce(initial, (partitions, item) => {
      const index = predicates.findIndex((pred) => pred(item))
      
      if (index === -1) return partitions
      
      return partitions.map((partition, i) =>
        i === index ? List.prepend(partition, item) : partition
      )
    }),
    (partitions) => partitions.map(List.reverse)
  )
}

// Example usage
const numbers = List.range(1, 20)

// Sliding windows
const windows = slidingWindow(numbers, 3)
console.log("Sliding windows:", 
  pipe(windows, List.map(List.toArray), List.toArray)
) // [[1,2,3], [2,3,4], [3,4,5], ...]

// Group consecutive
const consecutive = List.make(1, 1, 2, 2, 2, 3, 1, 1)
const groups = groupConsecutive(consecutive, (a, b) => a === b)
console.log("Grouped:", 
  pipe(groups, List.map(List.toArray), List.toArray)
) // [[1,1], [2,2,2], [3], [1,1]]

// Multi-partition
const [evens, divBy3, divBy5] = multiPartition(
  numbers,
  [
    (n) => n % 2 === 0,
    (n) => n % 3 === 0,
    (n) => n % 5 === 0
  ]
)

console.log("Evens:", List.toArray(evens)) // [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]
console.log("Divisible by 3:", List.toArray(divBy3)) // [3, 6, 9, 12, 15, 18]
console.log("Divisible by 5:", List.toArray(divBy5)) // [5, 10, 15, 20]
```

### List Fusion and Optimization

Effect's List module provides several optimizations for chaining operations:

```typescript
import { List, pipe, Function as Fn } from "effect"

// Automatic fusion for map operations
const fusedMap = <A, B, C, D>(
  list: List.List<A>,
  f: (a: A) => B,
  g: (b: B) => C,
  h: (c: C) => D
): List.List<D> => {
  // Instead of multiple passes
  // return pipe(list, List.map(f), List.map(g), List.map(h))
  
  // Single pass with composed function
  return pipe(
    List.map(Fn.flow(f, g, h))(list)
  )
}

// Optimized filter + map pattern
const filterMap = <A, B>(
  list: List.List<A>,
  predicate: (a: A) => boolean,
  mapper: (a: A) => B
): List.List<B> => {
  return pipe(
    List.reduce(List.empty<B>(), (acc, item) =>
      predicate(item) 
        ? List.prepend(acc, mapper(item))
        : acc
    )(list),
    List.reverse
  )
}

// Optimized take while pattern
const takeWhileWithIndex = <A>(
  list: List.List<A>,
  predicate: (a: A, index: number) => boolean
): List.List<A> => {
  const go = (
    remaining: List.List<A>,
    index: number,
    acc: List.List<A>
  ): List.List<A> => {
    return pipe(
      List.head(remaining),
      Option.match({
        onNone: () => List.reverse(acc),
        onSome: (item) => {
          if (predicate(item, index)) {
            return go(
              Option.getOrElse(List.tail(remaining), () => List.empty<A>()),
              index + 1,
              List.prepend(acc, item)
            )
          }
          return List.reverse(acc)
        }
      })
    )
  }
  
  return go(list, 0, List.empty())
}

// Chunk processing for large lists
const chunkProcess = <A, B>(
  list: List.List<A>,
  chunkSize: number,
  processor: (chunk: List.List<A>) => List.List<B>
): List.List<B> => {
  const processChunks = (
    remaining: List.List<A>,
    acc: List.List<B>
  ): List.List<B> => {
    if (List.isEmpty(remaining)) {
      return List.reverse(acc)
    }
    
    const chunk = List.take(remaining, chunkSize)
    const processed = processor(chunk)
    const newRemaining = List.drop(remaining, chunkSize)
    
    return processChunks(
      newRemaining,
      List.prependAll(acc, List.reverse(processed))
    )
  }
  
  return processChunks(list, List.empty())
}

// Example usage
const largeNumbers = List.range(1, 10000)

// Optimized filtering and mapping
const result = filterMap(
  largeNumbers,
  (n) => n % 2 === 0,
  (n) => n * n
)

// Chunked processing for memory efficiency
const chunks = chunkProcess(
  largeNumbers,
  1000,
  (chunk) => pipe(
    chunk,
    List.filter((n) => n % 3 === 0),
    List.map((n) => n.toString())
  )
)
```

## Practical Patterns & Best Practices

### List vs Array Decision Matrix

```typescript
import { List, Array as EffectArray, pipe } from "effect"

// Use List when:
// 1. Frequent prepend operations
// 2. Recursive algorithms with deep nesting
// 3. Persistent data structures needed
// 4. Undo/redo or history tracking
// 5. Functional programming patterns (fold, unfold)

// Use Array when:
// 1. Random access is required
// 2. Numeric computations
// 3. Large datasets with index-based operations
// 4. Interop with existing JS APIs
// 5. Memory-critical applications

// Helper to choose the right structure
type CollectionChoice<A> = {
  readonly preprendHeavy: List.List<A>
  readonly indexHeavy: ReadonlyArray<A>
  readonly balanced: ReadonlyArray<A>
}

const chooseCollection = <A>(
  operations: {
    prepends: number
    randomAccess: number
    iterations: number
  }
): keyof CollectionChoice<A> => {
  const { prepends, randomAccess, iterations } = operations
  
  if (prepends > randomAccess * 2) return "preprendHeavy"
  if (randomAccess > prepends * 2) return "indexHeavy"
  return "balanced"
}

// Conversion utilities for gradual migration
const listToArrayWhenNeeded = <A, B>(
  list: List.List<A>,
  operation: (arr: ReadonlyArray<A>) => B
): B => {
  return operation(List.toArray(list))
}

const arrayToListForOperation = <A, B>(
  array: ReadonlyArray<A>,
  operation: (list: List.List<A>) => List.List<B>
): ReadonlyArray<B> => {
  return pipe(
    List.fromIterable(array),
    operation,
    List.toArray
  )
}
```

### Error Recovery and Resilience Patterns

```typescript
import { List, Effect, Option, Either, pipe } from "effect"

// Error recovery with partial results
const processWithRecovery = <A, B>(
  list: List.List<A>,
  processor: (a: A) => Effect.Effect<B, Error>
): Effect.Effect<{ successes: List.List<B>; failures: List.List<{ item: A; error: Error }> }> => {
  return Effect.gen(function* () {
    let successes = List.empty<B>()
    let failures = List.empty<{ item: A; error: Error }>()
    
    for (const item of List.toArray(list)) {
      const result = yield* Effect.either(processor(item))
      
      if (Either.isRight(result)) {
        successes = List.prepend(successes, result.right)
      } else {
        failures = List.prepend(failures, { item, error: result.left })
      }
    }
    
    return {
      successes: List.reverse(successes),
      failures: List.reverse(failures)
    }
  })
}

// Circuit breaker pattern with List
class CircuitBreaker<A, B> {
  private failures = List.empty<Date>()
  private readonly failureThreshold: number
  private readonly recoveryTimeout: number
  
  constructor(failureThreshold: number = 5, recoveryTimeout: number = 60000) {
    this.failureThreshold = failureThreshold
    this.recoveryTimeout = recoveryTimeout
  }
  
  process(
    items: List.List<A>,
    processor: (a: A) => Effect.Effect<B, Error>
  ): Effect.Effect<List.List<B>, Error> {
    return Effect.gen(function* () {
      // Clean old failures
      const now = new Date()
      this.failures = List.filter(
        this.failures,
        (failureTime) => now.getTime() - failureTime.getTime() < this.recoveryTimeout
      )
      
      // Check if circuit is open
      if (List.length(this.failures) >= this.failureThreshold) {
        yield* Effect.fail(new Error("Circuit breaker is open"))
      }
      
      let results = List.empty<B>()
      
      for (const item of List.toArray(items)) {
        try {
          const result = yield* processor(item)
          results = List.prepend(results, result)
        } catch (error) {
          this.failures = List.prepend(this.failures, now)
          
          if (List.length(this.failures) >= this.failureThreshold) {
            yield* Effect.fail(new Error("Circuit breaker opened due to failures"))
          }
          
          throw error
        }
      }
      
      return List.reverse(results)
    })
  }
}

// Retry pattern with exponential backoff
const retryWithBackoff = <A, B>(
  items: List.List<A>,
  processor: (a: A) => Effect.Effect<B, Error>,
  maxRetries: number = 3
): Effect.Effect<List.List<B>, Error> => {
  const processWithRetry = (item: A): Effect.Effect<B, Error> => {
    const attempt = (retries: number): Effect.Effect<B, Error> => {
      return Effect.catchAll(processor(item), (error) => {
        if (retries <= 0) {
          return Effect.fail(error)
        }
        
        const delay = Math.pow(2, maxRetries - retries) * 1000
        return pipe(
          Effect.sleep(`${delay} millis`),
          Effect.flatMap(() => attempt(retries - 1))
        )
      })
    }
    
    return attempt(maxRetries)
  }
  
  return pipe(
    items,
    List.map(processWithRetry),
    (effects) => Effect.all(List.toArray(effects)),
    Effect.map(List.fromIterable)
  )
}
```

### Memory-Efficient Patterns

```typescript
import { List, Effect, Option, pipe } from "effect"

// Lazy list generation to avoid memory spikes
const lazyRange = function* (start: number, end: number) {
  for (let i = start; i <= end; i++) {
    yield i
  }
}

const processLargeDataset = <A, B>(
  generator: Generator<A>,
  batchSize: number,
  process: (batch: List.List<A>) => Effect.Effect<List.List<B>>
): Effect.Effect<List.List<B>> => {
  return Effect.gen(function* () {
    let results = List.empty<B>()
    let batch = List.empty<A>()
    let count = 0
    
    for (const item of generator) {
      batch = List.prepend(batch, item)
      count++
      
      if (count >= batchSize) {
        const processed = yield* process(List.reverse(batch))
        results = List.appendAll(results, processed)
        batch = List.empty()
        count = 0
      }
    }
    
    // Process remaining items
    if (List.isNonEmpty(batch)) {
      const processed = yield* process(List.reverse(batch))
      results = List.appendAll(results, processed)
    }
    
    return results
  })
}

// Memory-efficient operations using generators
const streamingMap = <A, B>(
  list: List.List<A>,
  f: (a: A) => B,
  chunkSize: number = 1000
): Generator<B> => {
  function* go(remaining: List.List<A>): Generator<B> {
    const chunk = List.take(remaining, chunkSize)
    
    if (List.isEmpty(chunk)) return
    
    for (const item of List.toArray(chunk)) {
      yield f(item)
    }
    
    const rest = List.drop(remaining, chunkSize)
    if (List.isNonEmpty(rest)) {
      yield* go(rest)
    }
  }
  
  return go(list)
}

// Memory pool for frequent List operations
class ListPool<T> {
  private pool = List.empty<List.List<T>>()
  private readonly maxSize: number
  
  constructor(maxSize: number = 10) {
    this.maxSize = maxSize
  }
  
  acquire(): List.List<T> {
    return pipe(
      List.head(this.pool),
      Option.match({
        onNone: () => List.empty<T>(),
        onSome: (reusable) => {
          this.pool = Option.getOrElse(
            List.tail(this.pool),
            () => List.empty<List.List<T>>()
          )
          return reusable
        }
      })
    )
  }
  
  release(list: List.List<T>): void {
    if (List.length(this.pool) < this.maxSize) {
      // Clear the list and return to pool
      this.pool = List.prepend(this.pool, List.empty<T>())
    }
  }
  
  withPooled<R>(operation: (list: List.List<T>) => R): R {
    const list = this.acquire()
    try {
      return operation(list)
    } finally {
      this.release(list)
    }
  }
}

// Garbage collection friendly operations
const gcFriendlyReduce = <A, B>(
  list: List.List<A>,
  initial: B,
  reducer: (acc: B, item: A) => B,
  gcInterval: number = 1000
): B => {
  let acc = initial
  let count = 0
  
  const processChunk = (chunk: List.List<A>): B => {
    return List.reduce(chunk, acc, reducer)
  }
  
  let current = list
  while (List.isNonEmpty(current)) {
    const chunk = List.take(current, gcInterval)
    acc = processChunk(chunk)
    current = List.drop(current, gcInterval)
    count += List.length(chunk)
    
    // Allow GC to run
    if (count % (gcInterval * 10) === 0) {
      // Force micro-task to allow GC
      setTimeout(() => {}, 0)
    }
  }
  
  return acc
}
```

### Testing Utilities

```typescript
import { List, Option, Equal, pipe, Array as EffectArray } from "effect"
import * as fc from "fast-check"

// Property-based testing for List operations
const listArbitrary = <A>(arb: fc.Arbitrary<A>): fc.Arbitrary<List.List<A>> =>
  fc.array(arb).map(List.fromIterable)

// Test properties
describe("List Laws", () => {
  test("prepend/head/tail consistency", () => {
    fc.assert(
      fc.property(
        fc.integer(),
        listArbitrary(fc.integer()),
        (head, tail) => {
          const list = List.prepend(tail, head)
          
          return (
            Option.getOrNull(List.head(list)) === head &&
            Equal.equals(
              Option.getOrElse(List.tail(list), () => List.empty<number>()),
              tail
            )
          )
        }
      )
    )
  })
  
  test("reverse involution", () => {
    fc.assert(
      fc.property(
        listArbitrary(fc.integer()),
        (list) => Equal.equals(List.reverse(List.reverse(list)), list)
      )
    )
  })
  
  test("map fusion", () => {
    fc.assert(
      fc.property(
        listArbitrary(fc.integer()),
        (list) => {
          const f = (x: number) => x * 2
          const g = (x: number) => x + 1
          
          return Equal.equals(
            pipe(list, List.map(f), List.map(g)),
            pipe(list, List.map((x) => g(f(x))))
          )
        }
      )
    )
  })
})

// Test helpers for async operations
const testListEffect = async <A>(
  list: List.List<Effect.Effect<A>>,
  expected: List.List<A>
): Promise<void> => {
  const results = await Effect.runPromise(
    Effect.all(List.toArray(list), { concurrency: "unbounded" })
  )
  
  expect(List.fromIterable(results)).toEqual(expected)
}

// Snapshot testing for complex transformations
const snapshotList = <A>(
  list: List.List<A>,
  name: string
): void => {
  const snapshot = {
    length: List.length(list),
    head: Option.getOrNull(List.head(list)),
    last: Option.getOrNull(List.last(list)),
    items: List.toArray(List.take(list, 10)), // First 10 for readability
    isEmpty: List.isEmpty(list)
  }
  
  expect(snapshot).toMatchSnapshot(name)
}
```

## Integration Examples

### Integration with Stream

```typescript
import { List, Stream, Effect, pipe, Chunk } from "effect"

// Convert List to Stream for async processing
const listToStream = <A>(list: List.List<A>): Stream.Stream<A> => {
  return Stream.fromIterable(List.toArray(list))
}

// Collect Stream into List
const streamToList = <A>(stream: Stream.Stream<A>): Effect.Effect<List.List<A>> => {
  return pipe(
    stream,
    Stream.runCollect,
    Effect.map((chunk) => List.fromIterable(Chunk.toArray(chunk)))
  )
}

// Process List items as Stream with backpressure
const processListAsync = <A, B>(
  list: List.List<A>,
  process: (a: A) => Effect.Effect<B>,
  concurrency: number = 5
): Effect.Effect<List.List<B>> => {
  return pipe(
    Stream.mapEffect(process, { concurrency })(listToStream(list)),
    streamToList
  )
}

// Example: Parallel API calls with rate limiting
interface User {
  id: number
  name: string
}

interface EnrichedUser extends User {
  profile: UserProfile
  posts: Post[]
}

const enrichUsers = (users: List.List<User>): Effect.Effect<List.List<EnrichedUser>> => {
  const fetchProfile = (userId: number): Effect.Effect<UserProfile> =>
    Effect.succeed({ userId, avatar: "avatar.jpg", bio: "Bio text" })
  
  const fetchPosts = (userId: number): Effect.Effect<Post[]> =>
    Effect.succeed([{ id: 1, userId, title: "Post 1", content: "Content" }])
  
  return pipe(
    Stream.mapEffect(
      (user) =>
        Effect.all({
          profile: fetchProfile(user.id),
          posts: fetchPosts(user.id)
        }).pipe(
          Effect.map(({ profile, posts }) => ({
            ...user,
            profile,
            posts
          }))
        ),
      { concurrency: 3 } // Process 3 users at a time
    )(listToStream(users)),
    Stream.throttle({
      cost: 1,
      duration: "100 millis",
      units: 10
    }), // Rate limit to 10 requests per 100ms
    streamToList
  )
}
```

### Integration with Schema

```typescript
import { List, Schema, pipe, ParseResult } from "effect"

// Define schemas for List-based data structures
const TaskSchema = Schema.Struct({
  id: Schema.String,
  title: Schema.String,
  completed: Schema.Boolean,
  priority: Schema.Literal("low", "medium", "high"),
  tags: Schema.Array(Schema.String),
  createdAt: Schema.DateFromString,
  dueDate: Schema.optional(Schema.DateFromString)
})

const TaskListSchema = Schema.Array(TaskSchema)

// Custom schema for List with validation
const ValidatedTaskListSchema = pipe(
  TaskListSchema,
  Schema.transform(
    Schema.Array(TaskSchema),
    Schema.unknown,
    {
      decode: (array) => {
        const list = List.fromIterable(array)
        
        // Custom validation: ensure no duplicate IDs
        const ids = List.map(list, (task) => task.id)
        const uniqueIds = new Set(List.toArray(ids))
        
        if (uniqueIds.size !== List.length(list)) {
          return ParseResult.fail(ParseResult.type("Tasks must have unique IDs"))
        }
        
        // Custom validation: high priority tasks must have due dates
        const invalidHighPriority = List.findFirst(list, (task) => 
          task.priority === "high" && !task.dueDate
        )
        
        if (Option.isSome(invalidHighPriority)) {
          return ParseResult.fail(
            ParseResult.type("High priority tasks must have due dates")
          )
        }
        
        return ParseResult.succeed(list)
      },
      encode: (list) => List.toArray(list as List.List<any>)
    }
  )
)

// Advanced schema transformations with List
const ProjectSchema = Schema.Struct({
  id: Schema.String,
  name: Schema.String,
  tasks: ValidatedTaskListSchema,
  teamMembers: Schema.Array(Schema.String)
})

// Schema for serialization (List -> Array)
const ProjectSerializationSchema = pipe(
  ProjectSchema,
  Schema.transform(
    ProjectSchema,
    Schema.Struct({
      id: Schema.String,
      name: Schema.String,
      tasks: Schema.Array(TaskSchema),
      teamMembers: Schema.Array(Schema.String),
      metadata: Schema.Struct({
        totalTasks: Schema.Number,
        completedTasks: Schema.Number,
        highPriorityTasks: Schema.Number
      })
    }),
    {
      decode: (serialized) => ({
        id: serialized.id,
        name: serialized.name,
        tasks: List.fromIterable(serialized.tasks),
        teamMembers: serialized.teamMembers
      }),
      encode: (project) => {
        const tasks = project.tasks as List.List<typeof TaskSchema.Type>
        const completed = List.filter(tasks, (task) => task.completed)
        const highPriority = List.filter(tasks, (task) => task.priority === "high")
        
        return {
          id: project.id,
          name: project.name,
          tasks: List.toArray(tasks),
          teamMembers: project.teamMembers,
          metadata: {
            totalTasks: List.length(tasks),
            completedTasks: List.length(completed),
            highPriorityTasks: List.length(highPriority)
          }
        }
      }
    }
  )
)

// List-specific schema helpers
const OrderedListSchema = <A>(itemSchema: Schema.Schema<A, A>) =>
  pipe(
    Schema.Array(itemSchema),
    Schema.transform(
      Schema.Array(itemSchema),
      Schema.unknown,
      {
        decode: (array) => {
          const list = List.fromIterable(array)
          // Ensure list maintains insertion order
          return ParseResult.succeed(list)
        },
        encode: (list) => List.toArray(list as List.List<A>)
      }
    )
  )

const SortedListSchema = <A>(
  itemSchema: Schema.Schema<A, A>,
  compare: (a: A, b: A) => number
) =>
  pipe(
    Schema.Array(itemSchema),
    Schema.transform(
      Schema.Array(itemSchema),
      Schema.unknown,
      {
        decode: (array) => {
          const sorted = [...array].sort(compare)
          return ParseResult.succeed(List.fromIterable(sorted))
        },
        encode: (list) => List.toArray(list as List.List<A>)
      }
    )
  )

// Usage examples
const validateAndParseProject = (data: unknown) =>
  Schema.decodeUnknown(ProjectSchema)(data)

const serializeProject = (project: typeof ProjectSchema.Type) =>
  Schema.encode(ProjectSerializationSchema)(project)

// Example with sorted list
const PriorityTaskListSchema = SortedListSchema(
  TaskSchema,
  (a, b) => {
    const priorityOrder = { high: 3, medium: 2, low: 1 }
    return priorityOrder[b.priority] - priorityOrder[a.priority]
  }
)
```

### Integration with Platform HTTP

```typescript
import { List, Effect, pipe, HttpApi, HttpApiBuilder, HttpApiGroup } from "@effect/platform"

// HTTP API with List-based responses
const TasksApi = HttpApiGroup.make("tasks")
  .add(
    HttpApi.get("getAllTasks", "/tasks")
      .setSuccess(Schema.Array(TaskSchema))
  )
  .add(
    HttpApi.post("createTasks", "/tasks/batch")
      .setPayload(Schema.Array(TaskSchema))
      .setSuccess(Schema.Struct({
        created: Schema.Array(TaskSchema),
        errors: Schema.Array(Schema.Struct({
          task: TaskSchema,
          error: Schema.String
        }))
      }))
  )
  .add(
    HttpApi.get("getTasksByPriority", "/tasks/priority/:priority")
      .setParams(Schema.Struct({ priority: Schema.Literal("low", "medium", "high") }))
      .setSuccess(Schema.Array(TaskSchema))
  )

// Service implementation using Lists
interface TaskService {
  getAllTasks(): Effect.Effect<List.List<Task>>
  createTasks(tasks: List.List<Task>): Effect.Effect<{
    created: List.List<Task>
    errors: List.List<{ task: Task; error: string }>
  }>
  getTasksByPriority(priority: "low" | "medium" | "high"): Effect.Effect<List.List<Task>>
}

const TaskServiceLive = Effect.gen(function* () {
  let tasks = List.empty<Task>()
  
  const getAllTasks = () => Effect.succeed(tasks)
  
  const createTasks = (newTasks: List.List<Task>) =>
    Effect.gen(function* () {
      let created = List.empty<Task>()
      let errors = List.empty<{ task: Task; error: string }>()
      
      for (const task of List.toArray(newTasks)) {
        // Validate task
        if (!task.title.trim()) {
          errors = List.prepend(errors, {
            task,
            error: "Task title cannot be empty"
          })
          continue
        }
        
        // Check for duplicate ID
        const exists = List.some(tasks, (existing) => existing.id === task.id)
        if (exists) {
          errors = List.prepend(errors, {
            task,
            error: "Task with this ID already exists"
          })
          continue
        }
        
        // Add task
        tasks = List.prepend(tasks, task)
        created = List.prepend(created, task)
      }
      
      return {
        created: List.reverse(created),
        errors: List.reverse(errors)
      }
    })
  
  const getTasksByPriority = (priority: "low" | "medium" | "high") =>
    Effect.succeed(
      List.filter(tasks, (task) => task.priority === priority)
    )
  
  return {
    getAllTasks,
    createTasks,
    getTasksByPriority
  } satisfies TaskService
})

// HTTP handlers
const TasksApiLive = HttpApiBuilder.group(TasksApi, "/api", (handlers) =>
  handlers
    .handle("getAllTasks", () =>
      Effect.gen(function* () {
        const taskService = yield* TaskService
        const tasks = yield* taskService.getAllTasks()
        return List.toArray(tasks) // Convert to Array for HTTP response
      })
    )
    .handle("createTasks", ({ payload }) =>
      Effect.gen(function* () {
        const taskService = yield* TaskService
        const taskList = List.fromIterable(payload)
        const result = yield* taskService.createTasks(taskList)
        
        return {
          created: List.toArray(result.created),
          errors: List.toArray(result.errors)
        }
      })
    )
    .handle("getTasksByPriority", ({ params }) =>
      Effect.gen(function* () {
        const taskService = yield* TaskService
        const tasks = yield* taskService.getTasksByPriority(params.priority)
        return List.toArray(tasks)
      })
    )
)

// Client usage
const taskClient = HttpApi.client(TasksApi)

const createMultipleTasks = (tasks: List.List<Task>) =>
  Effect.gen(function* () {
    const result = yield* taskClient.createTasks({
      payload: List.toArray(tasks)
    })
    
    console.log(`Created ${result.created.length} tasks`)
    console.log(`${result.errors.length} errors occurred`)
    
    return {
      created: List.fromIterable(result.created),
      errors: List.fromIterable(result.errors)
    }
  })
```

### Testing Strategies

```typescript
import { List, Effect, TestContext, TestClock, pipe } from "effect"
import { describe, test, expect } from "vitest"

// Test utility for List-based operations
const testListOperation = <A, B>(
  name: string,
  input: List.List<A>,
  operation: (list: List.List<A>) => List.List<B>,
  expected: List.List<B>
) => {
  test(name, () => {
    const result = operation(input)
    expect(List.toArray(result)).toEqual(List.toArray(expected))
  })
}

// Property-based testing utilities
const genList = <A>(genItem: () => A, maxLength: number = 100): List.List<A> => {
  const length = Math.floor(Math.random() * maxLength)
  const items = Array.from({ length }, genItem)
  return List.fromIterable(items)
}

// Test async List operations
describe("Async List Operations", () => {
  test("concurrent processing maintains order", async () => {
    const items = List.range(1, 10)
    
    const processWithDelay = (n: number) =>
      Effect.delay(
        Effect.succeed(n * 2),
        `${Math.random() * 100} millis`
      )
    
    const result = await Effect.runPromise(
      pipe(
        List.map(processWithDelay)(items),
        (effects) => Effect.all(List.toArray(effects), {
          concurrency: "unbounded"
        }),
        Effect.map(List.fromIterable)
      )
    )
    
    expect(List.toArray(result)).toEqual([2, 4, 6, 8, 10, 12, 14, 16, 18, 20])
  })
  
  test("error handling in List operations", async () => {
    const items = List.make(1, 2, 3, 4, 5)
    
    const processWithError = (n: number) =>
      n === 3
        ? Effect.fail("Error at 3")
        : Effect.succeed(n * 2)
    
    const result = await Effect.runPromise(
      pipe(
        List.map(processWithError)(items),
        (effects) => Effect.all(List.toArray(effects), {
          concurrency: 1,
          mode: "either"
        }),
        Effect.map((results) =>
          results
            .filter(Either.isRight)
            .map((e) => e.right)
        ),
        Effect.map(List.fromIterable)
      )
    )
    
    expect(List.toArray(result)).toEqual([2, 4, 8, 10])
  })
})

// Mock data generators for List
const mockUserList = (count: number): List.List<User> =>
  List.fromIterable(
    Array.from({ length: count }, (_, i) => ({
      id: i + 1,
      name: `User ${i + 1}`,
      email: `user${i + 1}@example.com`,
      active: Math.random() > 0.3
    }))
  )

// Performance testing
const benchmarkListOperation = async <A>(
  name: string,
  list: List.List<A>,
  operation: (list: List.List<A>) => unknown
) => {
  const start = performance.now()
  operation(list)
  const end = performance.now()
  
  console.log(`${name}: ${(end - start).toFixed(2)}ms`)
}
```

## Conclusion

List provides an immutable, persistent linked list that excels at functional programming patterns, recursive algorithms, and prepend-heavy operations in the Effect ecosystem.

Key benefits:
- **O(1) Prepend Operations**: Ideal for building collections from head to tail
- **Stack-Safe Recursion**: Process deeply nested structures without stack overflow
- **Persistent Data Structure**: Efficient undo/redo and time-travel debugging
- **Functional Programming**: First-class support for map, filter, fold, and other FP patterns
- **Memory Efficiency**: Structural sharing reduces memory overhead for immutable operations

List shines when you need efficient prepend operations, recursive algorithms, or persistent data structures, while Array remains the better choice for index-based access and numeric computations.