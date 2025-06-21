# SortedSet: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem SortedSet Solves

JavaScript's native Set provides uniqueness but no ordering guarantees. When you need to maintain a collection of unique values in sorted order, traditional approaches require complex manual management:

```typescript
// Traditional approach - manually maintaining sorted unique collection
class SortedUniqueList<T> {
  private items: T[] = []
  private compareFn: (a: T, b: T) => number
  
  constructor(compareFn: (a: T, b: T) => number) {
    this.compareFn = compareFn
  }
  
  add(item: T): void {
    // Check if item already exists
    const existingIndex = this.findIndex(item)
    if (existingIndex !== -1) {
      this.items[existingIndex] = item // Update
      return
    }
    
    // Find insertion point
    let insertIndex = 0
    for (let i = 0; i < this.items.length; i++) {
      if (this.compareFn(item, this.items[i]) < 0) {
        insertIndex = i
        break
      }
      insertIndex = i + 1
    }
    
    // Insert at correct position
    this.items.splice(insertIndex, 0, item)
  }
  
  remove(item: T): boolean {
    const index = this.findIndex(item)
    if (index === -1) return false
    
    this.items.splice(index, 1)
    return true
  }
  
  private findIndex(item: T): number {
    // Linear search (could use binary search for better performance)
    for (let i = 0; i < this.items.length; i++) {
      if (this.compareFn(item, this.items[i]) === 0) {
        return i
      }
    }
    return -1
  }
  
  has(item: T): boolean {
    return this.findIndex(item) !== -1
  }
  
  getRange(min: T, max: T): T[] {
    return this.items.filter(item => 
      this.compareFn(item, min) >= 0 && 
      this.compareFn(item, max) <= 0
    )
  }
  
  // Union requires manual merging while maintaining order
  union(other: SortedUniqueList<T>): SortedUniqueList<T> {
    const result = new SortedUniqueList(this.compareFn)
    
    // Merge two sorted arrays
    let i = 0, j = 0
    const thisItems = this.items
    const otherItems = other.items
    
    while (i < thisItems.length && j < otherItems.length) {
      const cmp = this.compareFn(thisItems[i], otherItems[j])
      if (cmp < 0) {
        result.add(thisItems[i++])
      } else if (cmp > 0) {
        result.add(otherItems[j++])
      } else {
        result.add(thisItems[i++])
        j++ // Skip duplicate
      }
    }
    
    // Add remaining items
    while (i < thisItems.length) result.add(thisItems[i++])
    while (j < otherItems.length) result.add(otherItems[j++])
    
    return result
  }
}

// Native Set with manual sorting
class ScoreBoard {
  private scores = new Set<number>()
  
  addScore(score: number): void {
    this.scores.add(score)
  }
  
  getTopScores(n: number): number[] {
    // Convert to array and sort every time
    return Array.from(this.scores)
      .sort((a, b) => b - a) // Descending
      .slice(0, n)
  }
  
  getScoresInRange(min: number, max: number): number[] {
    return Array.from(this.scores)
      .filter(score => score >= min && score <= max)
      .sort((a, b) => a - b)
  }
}

// Priority queue using array
class TaskQueue {
  private tasks: Array<{ priority: number, task: string }> = []
  
  enqueue(priority: number, task: string): void {
    // Manual insertion to maintain priority order
    let inserted = false
    for (let i = 0; i < this.tasks.length; i++) {
      if (priority > this.tasks[i].priority) {
        this.tasks.splice(i, 0, { priority, task })
        inserted = true
        break
      }
    }
    if (!inserted) {
      this.tasks.push({ priority, task })
    }
  }
  
  dequeue(): { priority: number, task: string } | undefined {
    return this.tasks.shift()
  }
}
```

This approach leads to:
- **Performance issues** - O(n) insertions, O(n log n) sorting operations
- **Complex set operations** - Manual implementation of union, intersection, difference
- **Memory inefficiency** - Array copying for immutable operations
- **Error-prone range queries** - Hand-coded filtering and sorting
- **No type safety** - Custom comparators without compile-time guarantees

### The SortedSet Solution

SortedSet provides a persistent, immutable sorted set implementation with efficient set operations and type-safe custom ordering:

```typescript
import { SortedSet, Order, pipe } from "effect"

// Type-safe sorted set with automatic ordering
const scoreBoard = SortedSet.empty<number>(Order.reverse(Order.number)) // Descending

const withScores = pipe(
  SortedSet.add(scoreBoard, 95),
  SortedSet.add(87),
  SortedSet.add(92),
  SortedSet.add(87), // Duplicate - automatically ignored
  SortedSet.add(88)
)

// Automatic ordering maintained
const topScores = Array.from(SortedSet.values(withScores))
// [95, 92, 88, 87] - automatically sorted descending

// Efficient set operations
const teamAScores = pipe(
  SortedSet.add(SortedSet.empty(Order.number), 85),
  SortedSet.add(90),
  SortedSet.add(88)
)

const teamBScores = pipe(
  SortedSet.add(SortedSet.empty(Order.number), 88),
  SortedSet.add(92),
  SortedSet.add(87)
)

// Union of both teams' scores
const allScores = SortedSet.union(teamAScores, teamBScores)
// [85, 87, 88, 90, 92] - automatically sorted and deduplicated

// Complex custom ordering
interface Task {
  priority: number
  dueDate: Date
  name: string
}

const taskOrder = Order.combine(
  Order.mapInput(Order.reverse(Order.number), (t: Task) => t.priority), // Higher priority first
  Order.mapInput(Order.Date, (t: Task) => t.dueDate) // Then by due date
)

const taskQueue = SortedSet.empty<Task>(taskOrder)

const withTasks = pipe(
  SortedSet.add(taskQueue, { priority: 2, dueDate: new Date('2024-01-20'), name: 'Medium task' }),
  SortedSet.add({ priority: 1, dueDate: new Date('2024-01-15'), name: 'Low task' }),
  SortedSet.add({ priority: 3, dueDate: new Date('2024-01-18'), name: 'High task' })
)

// Automatically ordered by priority then date
const nextTasks = Array.from(SortedSet.values(withTasks))
// High priority task comes first, despite being added last
```

### Key Concepts

**Ordered Uniqueness**: SortedSet maintains both uniqueness (like Set) and order (according to a provided Order instance), combining the benefits of both data structures.

**Efficient Set Operations**: Union, intersection, and difference operations are implemented efficiently using the sorted property, avoiding the need for hash table lookups.

**Structural Sharing**: Like other Effect collections, SortedSet uses persistent data structures with structural sharing for memory efficiency.

**Type-Safe Ordering**: Order instances provide compile-time guarantees about how elements will be compared and sorted.

## Basic Usage Patterns

### Pattern 1: Creating and Initializing SortedSets

```typescript
import { SortedSet, Order, pipe } from "effect"

// Empty SortedSet with natural ordering
const numbers = SortedSet.empty<number>(Order.number)

// From iterable with custom ordering
const words = SortedSet.fromIterable(
  Order.string,
  ['banana', 'apple', 'cherry', 'apple'] // Duplicate 'apple' removed
)

// Case-insensitive string ordering
const caseInsensitive = Order.mapInput(
  Order.string,
  (s: string) => s.toLowerCase()
)

const tags = SortedSet.fromIterable(
  caseInsensitive,
  ['React', 'angular', 'Vue', 'REACT', 'svelte']
)
// Results in: ['angular', 'React', 'svelte', 'Vue'] (REACT merged with React)

// Complex object ordering
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

const supportedVersions = pipe(
  SortedSet.add(SortedSet.empty<Version>(versionOrder), { major: 1, minor: 0, patch: 0 }),
  SortedSet.add({ major: 1, minor: 1, patch: 0 }),
  SortedSet.add({ major: 2, minor: 0, patch: 0 }),
  SortedSet.add({ major: 1, minor: 0, patch: 1 })
)

// Reverse ordering for descending sort
const descendingNumbers = SortedSet.empty<number>(Order.reverse(Order.number))
const scores = pipe(
  SortedSet.add(descendingNumbers, 85),
  SortedSet.add(92),
  SortedSet.add(78),
  SortedSet.add(95)
)
// Results in: [95, 92, 85, 78]
```

### Pattern 2: Basic Operations

```typescript
import { SortedSet, Order, pipe } from "effect"

const fruits = pipe(
  SortedSet.add(SortedSet.empty<string>(Order.string), 'apple'),
  SortedSet.add('banana'),
  SortedSet.add('cherry')
)

// Checking membership
const hasApple = SortedSet.has(fruits, 'apple') // true
const hasOrange = SortedSet.has(fruits, 'orange') // false

// Getting size
const fruitCount = SortedSet.size(fruits) // 3

// Adding elements (returns new set)
const moreFruits = pipe(
  SortedSet.add(fruits, 'date'),
  SortedSet.add('elderberry'),
  SortedSet.add('apple') // Duplicate ignored
)

// Removing elements
const fewerFruits = SortedSet.remove(fruits, 'banana')

// Checking if empty
const isEmpty = SortedSet.isNonEmpty(fruits) // true
const emptySet = SortedSet.empty<string>(Order.string)
const isEmptyCheck = SortedSet.isNonEmpty(emptySet) // false

// Getting min and max elements
const min = SortedSet.min(fruits) // Option.some('apple')
const max = SortedSet.max(fruits) // Option.some('cherry')

// Converting to arrays
const fruitArray = Array.from(SortedSet.values(fruits))
// ['apple', 'banana', 'cherry'] - always sorted
```

### Pattern 3: Set Operations

```typescript
import { SortedSet, Order, pipe } from "effect"

const setA = SortedSet.fromIterable(Order.number, [1, 2, 3, 4, 5])
const setB = SortedSet.fromIterable(Order.number, [3, 4, 5, 6, 7])
const setC = SortedSet.fromIterable(Order.number, [5, 6, 7, 8, 9])

// Union - all elements from both sets
const union = SortedSet.union(setA, setB)
// [1, 2, 3, 4, 5, 6, 7]

// Intersection - elements in both sets
const intersection = SortedSet.intersection(setA, setB)
// [3, 4, 5]

// Difference - elements in A but not in B
const difference = SortedSet.difference(setA, setB)
// [1, 2]

// Symmetric difference - elements in either set but not both
const symmetricDiff = SortedSet.difference(
  SortedSet.union(setA, setB),
  SortedSet.intersection(setA, setB)
)
// [1, 2, 6, 7]

// Multiple set operations
const result = SortedSet.difference(
  SortedSet.union(
    SortedSet.union(setA, setB),
    setC
  ),
  SortedSet.fromIterable(Order.number, [1, 9])
)
// Union of all three sets, minus 1 and 9

// Subset checking
const isSubset = SortedSet.isSubset(
  SortedSet.fromIterable(Order.number, [2, 3]),
  setA
) // true - [2, 3] is subset of [1, 2, 3, 4, 5]

const isSuperset = SortedSet.isSubset(setA, setB) // false

// Disjoint checking
const areDisjoint = SortedSet.isDisjoint(
  SortedSet.fromIterable(Order.number, [1, 2]),
  SortedSet.fromIterable(Order.number, [6, 7])
) // true - no common elements
```

## Real-World Examples

### Example 1: Tag Management System

A comprehensive tagging system for blog posts with hierarchical categories and automatic organization:

```typescript
import { SortedSet, Order, pipe, Option } from "effect"

interface Tag {
  name: string
  category: 'technology' | 'business' | 'lifestyle' | 'education'
  popularity: number
  created: Date
}

class TagManager {
  // Different sorted views of the same data
  private byName: SortedSet.SortedSet<Tag>
  private byPopularity: SortedSet.SortedSet<Tag>
  private byCategory: Map<string, SortedSet.SortedSet<Tag>>
  
  constructor() {
    // Primary index by name
    this.byName = SortedSet.empty(
      Order.mapInput(Order.string, (t: Tag) => t.name.toLowerCase())
    )
    
    // Secondary index by popularity (descending)
    this.byPopularity = SortedSet.empty(
      Order.combine(
        Order.mapInput(Order.reverse(Order.number), (t: Tag) => t.popularity),
        Order.mapInput(Order.string, (t: Tag) => t.name) // Tie-breaker
      )
    )
    
    // Category-specific indexes
    this.byCategory = new Map()
    const categories = ['technology', 'business', 'lifestyle', 'education']
    categories.forEach(cat => {
      this.byCategory.set(cat, SortedSet.empty(
        Order.mapInput(Order.string, (t: Tag) => t.name)
      ))
    })
  }
  
  // Add tag with automatic indexing
  addTag(tag: Tag): TagManager {
    const updated = new TagManager()
    
    // Update primary indexes
    updated.byName = SortedSet.add(this.byName, tag)
    updated.byPopularity = SortedSet.add(this.byPopularity, tag)
    
    // Update category index
    updated.byCategory = new Map(this.byCategory)
    const categorySet = this.byCategory.get(tag.category) || SortedSet.empty(
      Order.mapInput(Order.string, (t: Tag) => t.name)
    )
    updated.byCategory.set(tag.category, SortedSet.add(categorySet, tag))
    
    return updated
  }
  
  // Get popular tags across categories
  getPopularTags(limit: number = 10): ReadonlyArray<Tag> {
    return Array.from(SortedSet.values(this.byPopularity)).slice(0, limit)
  }
  
  // Get tags by category sorted by name
  getTagsByCategory(category: string): ReadonlyArray<Tag> {
    const categorySet = this.byCategory.get(category)
    return categorySet ? Array.from(SortedSet.values(categorySet)) : []
  }
  
  // Find related tags using popularity overlap
  getRelatedTags(tag: Tag, limit: number = 5): ReadonlyArray<Tag> {
    const categoryTags = this.getTagsByCategory(tag.category)
    
    // Sort by popularity difference
    return categoryTags
      .filter(t => t.name !== tag.name)
      .sort((a, b) => {
        const aDiff = Math.abs(a.popularity - tag.popularity)
        const bDiff = Math.abs(b.popularity - tag.popularity)
        return aDiff - bDiff
      })
      .slice(0, limit)
  }
  
  // Tag suggestions for auto-complete
  getSuggestions(prefix: string): ReadonlyArray<Tag> {
    const lowerPrefix = prefix.toLowerCase()
    
    return Array.from(SortedSet.values(this.byName))
      .filter(tag => tag.name.toLowerCase().startsWith(lowerPrefix))
      .slice(0, 10)
  }
  
  // Trending tags (recent + popular)
  getTrendingTags(daysCutoff: number = 30): ReadonlyArray<Tag> {
    const cutoffDate = new Date(Date.now() - daysCutoff * 24 * 60 * 60 * 1000)
    
    const recentTags = Array.from(SortedSet.values(this.byPopularity))
      .filter(tag => tag.created >= cutoffDate)
    
    // Custom trending score: popularity * recency factor
    return recentTags
      .map(tag => {
        const daysSince = (Date.now() - tag.created.getTime()) / (1000 * 60 * 60 * 24)
        const recencyFactor = Math.max(0.1, 1 - daysSince / daysCutoff)
        const trendingScore = tag.popularity * recencyFactor
        return { tag, trendingScore }
      })
      .sort((a, b) => b.trendingScore - a.trendingScore)
      .slice(0, 10)
      .map(({ tag }) => tag)
  }
  
  // Merge tag sets from different sources
  static mergeTags(...managers: TagManager[]): TagManager {
    if (managers.length === 0) return new TagManager()
    if (managers.length === 1) return managers[0]
    
    const merged = new TagManager()
    const allTags = new Set<Tag>()
    
    // Collect all unique tags
    for (const manager of managers) {
      for (const tag of SortedSet.values(manager.byName)) {
        allTags.add(tag)
      }
    }
    
    // Add all tags to merged manager
    return Array.from(allTags).reduce(
      (acc, tag) => acc.addTag(tag),
      merged
    )
  }
}

// Usage example
const tagManager = new TagManager()

const sampleTags: Tag[] = [
  { name: 'React', category: 'technology', popularity: 95, created: new Date('2024-01-15') },
  { name: 'JavaScript', category: 'technology', popularity: 98, created: new Date('2024-01-10') },
  { name: 'Leadership', category: 'business', popularity: 87, created: new Date('2024-01-20') },
  { name: 'Productivity', category: 'lifestyle', popularity: 89, created: new Date('2024-01-18') },
  { name: 'TypeScript', category: 'technology', popularity: 91, created: new Date('2024-01-25') }
]

const populated = sampleTags.reduce(
  (manager, tag) => manager.addTag(tag),
  tagManager
)

console.log('Popular tags:', populated.getPopularTags(3))
console.log('Technology tags:', populated.getTagsByCategory('technology'))
console.log('Trending:', populated.getTrendingTags())

// Auto-complete
console.log('Suggestions for "re":', populated.getSuggestions('re'))
```

### Example 2: Multi-Criteria Priority Queue

A sophisticated task scheduling system with multiple priority dimensions:

```typescript
import { SortedSet, Order, pipe, Option } from "effect"

interface ScheduledTask {
  id: string
  priority: number
  estimatedDuration: number // minutes
  deadline: Date
  dependencies: string[]
  category: 'urgent' | 'important' | 'routine'
  assignee?: string
}

class TaskScheduler {
  private allTasks: Map<string, ScheduledTask> = new Map()
  
  // Multiple sorted views for different scheduling strategies
  private byDeadline: SortedSet.SortedSet<ScheduledTask>
  private byPriority: SortedSet.SortedSet<ScheduledTask>
  private byDuration: SortedSet.SortedSet<ScheduledTask>
  private byComposite: SortedSet.SortedSet<ScheduledTask>
  
  constructor() {
    // Sort by deadline (earliest first)
    this.byDeadline = SortedSet.empty(
      Order.combine(
        Order.mapInput(Order.Date, (t: ScheduledTask) => t.deadline),
        Order.mapInput(Order.string, (t: ScheduledTask) => t.id) // Tie-breaker
      )
    )
    
    // Sort by priority (highest first)
    this.byPriority = SortedSet.empty(
      Order.combine(
        Order.mapInput(Order.reverse(Order.number), (t: ScheduledTask) => t.priority),
        Order.mapInput(Order.Date, (t: ScheduledTask) => t.deadline)
      )
    )
    
    // Sort by duration (shortest first)
    this.byDuration = SortedSet.empty(
      Order.mapInput(Order.number, (t: ScheduledTask) => t.estimatedDuration)
    )
    
    // Composite scoring
    this.byComposite = SortedSet.empty(
      Order.mapInput(
        Order.reverse(Order.number),
        (t: ScheduledTask) => this.calculateCompositeScore(t)
      )
    )
  }
  
  private calculateCompositeScore(task: ScheduledTask): number {
    const now = Date.now()
    const timeToDeadline = (task.deadline.getTime() - now) / (1000 * 60 * 60) // hours
    
    // Urgency factor (higher as deadline approaches)
    const urgencyFactor = timeToDeadline <= 0 ? 100 :
                          timeToDeadline <= 24 ? 50 :
                          timeToDeadline <= 72 ? 25 : 10
    
    // Category multiplier
    const categoryMultiplier = {
      urgent: 3,
      important: 2,
      routine: 1
    }[task.category]
    
    // Duration penalty (longer tasks get lower priority)
    const durationPenalty = Math.max(1, task.estimatedDuration / 60) // hours
    
    return (task.priority * categoryMultiplier * urgencyFactor) / durationPenalty
  }
  
  addTask(task: ScheduledTask): TaskScheduler {
    const scheduler = new TaskScheduler()
    
    // Copy existing tasks
    scheduler.allTasks = new Map(this.allTasks)
    scheduler.allTasks.set(task.id, task)
    
    // Rebuild sorted indexes
    const allTaskList = Array.from(scheduler.allTasks.values())
    
    scheduler.byDeadline = allTaskList.reduce(
      (acc, t) => SortedSet.add(acc, t),
      SortedSet.empty(Order.combine(
        Order.mapInput(Order.Date, (t: ScheduledTask) => t.deadline),
        Order.mapInput(Order.string, (t: ScheduledTask) => t.id)
      ))
    )
    
    scheduler.byPriority = allTaskList.reduce(
      (acc, t) => SortedSet.add(acc, t),
      SortedSet.empty(Order.combine(
        Order.mapInput(Order.reverse(Order.number), (t: ScheduledTask) => t.priority),
        Order.mapInput(Order.Date, (t: ScheduledTask) => t.deadline)
      ))
    )
    
    scheduler.byDuration = allTaskList.reduce(
      (acc, t) => SortedSet.add(acc, t),
      SortedSet.empty(Order.mapInput(Order.number, (t: ScheduledTask) => t.estimatedDuration))
    )
    
    scheduler.byComposite = allTaskList.reduce(
      (acc, t) => SortedSet.add(acc, t),
      SortedSet.empty(Order.mapInput(
        Order.reverse(Order.number),
        (t: ScheduledTask) => scheduler.calculateCompositeScore(t)
      ))
    )
    
    return scheduler
  }
  
  // Get next task by different strategies
  getNextByDeadline(): Option.Option<ScheduledTask> {
    return SortedSet.min(this.byDeadline)
  }
  
  getNextByPriority(): Option.Option<ScheduledTask> {
    return SortedSet.min(this.byPriority)
  }
  
  getNextByDuration(): Option.Option<ScheduledTask> {
    return SortedSet.min(this.byDuration)
  }
  
  getNextByComposite(): Option.Option<ScheduledTask> {
    return SortedSet.min(this.byComposite)
  }
  
  // Schedule tasks for a time window
  scheduleWindow(
    availableMinutes: number,
    strategy: 'deadline' | 'priority' | 'duration' | 'composite' = 'composite'
  ): ScheduledTask[] {
    const sortedTasks = this.getTasksByStrategy(strategy)
    const scheduled: ScheduledTask[] = []
    let remainingTime = availableMinutes
    
    for (const task of sortedTasks) {
      if (task.estimatedDuration <= remainingTime && this.canSchedule(task, scheduled)) {
        scheduled.push(task)
        remainingTime -= task.estimatedDuration
      }
    }
    
    return scheduled
  }
  
  private getTasksByStrategy(strategy: string): ScheduledTask[] {
    switch (strategy) {
      case 'deadline': return Array.from(SortedSet.values(this.byDeadline))
      case 'priority': return Array.from(SortedSet.values(this.byPriority))
      case 'duration': return Array.from(SortedSet.values(this.byDuration))
      case 'composite': return Array.from(SortedSet.values(this.byComposite))
      default: return Array.from(SortedSet.values(this.byComposite))
    }
  }
  
  private canSchedule(task: ScheduledTask, scheduled: ScheduledTask[]): boolean {
    // Check if all dependencies are already scheduled
    const scheduledIds = new Set(scheduled.map(t => t.id))
    return task.dependencies.every(dep => scheduledIds.has(dep))
  }
  
  // Get overdue tasks (past deadline)
  getOverdueTasks(): ScheduledTask[] {
    const now = new Date()
    return Array.from(SortedSet.values(this.byDeadline))
      .filter(task => task.deadline < now)
  }
  
  // Workload analysis by assignee
  getWorkloadByAssignee(): Map<string, { tasks: number, totalTime: number }> {
    const workload = new Map<string, { tasks: number, totalTime: number }>()
    
    for (const task of this.allTasks.values()) {
      const assignee = task.assignee || 'unassigned'
      const current = workload.get(assignee) || { tasks: 0, totalTime: 0 }
      workload.set(assignee, {
        tasks: current.tasks + 1,
        totalTime: current.totalTime + task.estimatedDuration
      })
    }
    
    return workload
  }
  
  // Bottleneck analysis - tasks with most dependencies
  getBottleneckTasks(threshold: number = 2): ScheduledTask[] {
    return Array.from(this.allTasks.values())
      .filter(task => {
        // Count how many other tasks depend on this one
        const dependentCount = Array.from(this.allTasks.values())
          .filter(other => other.dependencies.includes(task.id))
          .length
        return dependentCount >= threshold
      })
      .sort((a, b) => {
        const aDeps = Array.from(this.allTasks.values())
          .filter(other => other.dependencies.includes(a.id)).length
        const bDeps = Array.from(this.allTasks.values())
          .filter(other => other.dependencies.includes(b.id)).length
        return bDeps - aDeps
      })
  }
}

// Usage example
const scheduler = new TaskScheduler()

const tasks: ScheduledTask[] = [
  {
    id: 'task1',
    priority: 8,
    estimatedDuration: 120,
    deadline: new Date(Date.now() + 2 * 24 * 60 * 60 * 1000), // 2 days
    dependencies: [],
    category: 'important',
    assignee: 'alice'
  },
  {
    id: 'task2',
    priority: 9,
    estimatedDuration: 60,
    deadline: new Date(Date.now() + 1 * 24 * 60 * 60 * 1000), // 1 day
    dependencies: ['task1'],
    category: 'urgent',
    assignee: 'bob'
  },
  {
    id: 'task3',
    priority: 6,
    estimatedDuration: 30,
    deadline: new Date(Date.now() + 3 * 24 * 60 * 60 * 1000), // 3 days
    dependencies: [],
    category: 'routine',
    assignee: 'alice'
  }
]

const populatedScheduler = tasks.reduce(
  (sched, task) => sched.addTask(task),
  scheduler
)

console.log('Next by composite:', populatedScheduler.getNextByComposite())
console.log('4-hour schedule:', populatedScheduler.scheduleWindow(240))
console.log('Workload:', populatedScheduler.getWorkloadByAssignee())
```

### Example 3: Geographic Spatial Index

A location-based service with efficient spatial queries using sorted sets:

```typescript
import { SortedSet, Order, pipe, Option } from "effect"

interface Location {
  id: string
  name: string
  latitude: number
  longitude: number
  category: 'restaurant' | 'hotel' | 'gas_station' | 'hospital' | 'school'
  rating: number
}

interface BoundingBox {
  minLat: number
  maxLat: number
  minLon: number
  maxLon: number
}

class SpatialIndex {
  // Multiple sorted indexes for different query patterns
  private byLatitude: SortedSet.SortedSet<Location>
  private byLongitude: SortedSet.SortedSet<Location>
  private byDistance: Map<string, SortedSet.SortedSet<Location>>
  private byCategory: Map<string, SortedSet.SortedSet<Location>>
  
  constructor(centerPoint?: { lat: number, lon: number }) {
    // Index by latitude
    this.byLatitude = SortedSet.empty(
      Order.combine(
        Order.mapInput(Order.number, (loc: Location) => loc.latitude),
        Order.mapInput(Order.string, (loc: Location) => loc.id)
      )
    )
    
    // Index by longitude
    this.byLongitude = SortedSet.empty(
      Order.combine(
        Order.mapInput(Order.number, (loc: Location) => loc.longitude),
        Order.mapInput(Order.string, (loc: Location) => loc.id)
      )
    )
    
    // Distance-based indexes for common query points
    this.byDistance = new Map()
    
    // Category indexes with rating as secondary sort
    this.byCategory = new Map()
    const categories = ['restaurant', 'hotel', 'gas_station', 'hospital', 'school']
    categories.forEach(cat => {
      this.byCategory.set(cat, SortedSet.empty(
        Order.combine(
          Order.mapInput(Order.reverse(Order.number), (loc: Location) => loc.rating),
          Order.mapInput(Order.string, (loc: Location) => loc.name)
        )
      ))
    })
  }
  
  addLocation(location: Location): SpatialIndex {
    const index = new SpatialIndex()
    
    // Update latitude index
    index.byLatitude = SortedSet.add(this.byLatitude, location)
    
    // Update longitude index
    index.byLongitude = SortedSet.add(this.byLongitude, location)
    
    // Update category index
    index.byCategory = new Map(this.byCategory)
    const categorySet = this.byCategory.get(location.category) || 
      SortedSet.empty(Order.combine(
        Order.mapInput(Order.reverse(Order.number), (loc: Location) => loc.rating),
        Order.mapInput(Order.string, (loc: Location) => loc.name)
      ))
    index.byCategory.set(location.category, SortedSet.add(categorySet, location))
    
    // Update distance indexes
    index.byDistance = new Map(this.byDistance)
    for (const [centerKey, distanceSet] of this.byDistance) {
      const [lat, lon] = centerKey.split(',').map(Number)
      const distanceOrder = this.createDistanceOrder({ lat, lon })
      index.byDistance.set(centerKey, SortedSet.add(distanceSet, location))
    }
    
    return index
  }
  
  private createDistanceOrder(center: { lat: number, lon: number }) {
    return Order.combine(
      Order.mapInput(
        Order.number,
        (loc: Location) => this.calculateDistance(center, loc)
      ),
      Order.mapInput(Order.reverse(Order.number), (loc: Location) => loc.rating)
    )
  }
  
  private calculateDistance(
    p1: { lat: number, lon: number },
    p2: Location
  ): number {
    const R = 6371 // Earth radius in km
    const dLat = (p2.latitude - p1.lat) * Math.PI / 180
    const dLon = (p2.longitude - p1.lon) * Math.PI / 180
    const a = 
      Math.sin(dLat/2) * Math.sin(dLat/2) +
      Math.cos(p1.lat * Math.PI / 180) * Math.cos(p2.latitude * Math.PI / 180) *
      Math.sin(dLon/2) * Math.sin(dLon/2)
    const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a))
    return R * c
  }
  
  // Find locations within bounding box
  findInBoundingBox(box: BoundingBox): Location[] {
    // Use latitude index to find candidates
    const latCandidates = Array.from(SortedSet.values(this.byLatitude))
      .filter(loc => loc.latitude >= box.minLat && loc.latitude <= box.maxLat)
    
    // Filter by longitude
    return latCandidates
      .filter(loc => loc.longitude >= box.minLon && loc.longitude <= box.maxLon)
  }
  
  // Find nearest locations to a point
  findNearest(
    center: { lat: number, lon: number },
    limit: number = 10,
    category?: string
  ): Location[] {
    const centerKey = `${center.lat},${center.lon}`
    
    // Create distance-sorted set if not exists
    if (!this.byDistance.has(centerKey)) {
      const distanceOrder = this.createDistanceOrder(center)
      const allLocations = Array.from(SortedSet.values(this.byLatitude))
      const distanceSet = allLocations.reduce(
        (acc, loc) => SortedSet.add(acc, loc),
        SortedSet.empty<Location>(distanceOrder)
      )
      this.byDistance.set(centerKey, distanceSet)
    }
    
    let candidates = Array.from(SortedSet.values(this.byDistance.get(centerKey)!))
    
    // Filter by category if specified
    if (category) {
      candidates = candidates.filter(loc => loc.category === category)
    }
    
    return candidates.slice(0, limit)
  }
  
  // Find best rated locations in category
  findBestInCategory(category: string, limit: number = 10): Location[] {
    const categorySet = this.byCategory.get(category)
    if (!categorySet) return []
    
    return Array.from(SortedSet.values(categorySet)).slice(0, limit)
  }
  
  // Find locations within radius
  findWithinRadius(
    center: { lat: number, lon: number },
    radiusKm: number,
    category?: string
  ): Location[] {
    // Use bounding box as initial filter
    const latRange = radiusKm / 111 // Rough km to degree conversion
    const lonRange = radiusKm / (111 * Math.cos(center.lat * Math.PI / 180))
    
    const box: BoundingBox = {
      minLat: center.lat - latRange,
      maxLat: center.lat + latRange,
      minLon: center.lon - lonRange,
      maxLon: center.lon + lonRange
    }
    
    const candidates = this.findInBoundingBox(box)
    
    // Filter by actual distance
    let results = candidates
      .map(loc => ({
        location: loc,
        distance: this.calculateDistance(center, loc)
      }))
      .filter(({ distance }) => distance <= radiusKm)
      .sort((a, b) => a.distance - b.distance)
      .map(({ location }) => location)
    
    // Filter by category if specified
    if (category) {
      results = results.filter(loc => loc.category === category)
    }
    
    return results
  }
  
  // Clustering for heatmap generation
  generateClusters(
    gridSize: number = 0.01 // degrees
  ): Array<{
    centerLat: number,
    centerLon: number,
    count: number,
    categories: Map<string, number>
  }> {
    const clusters = new Map<string, {
      centerLat: number,
      centerLon: number,
      count: number,
      categories: Map<string, number>
    }>()
    
    for (const location of SortedSet.values(this.byLatitude)) {
      // Determine grid cell
      const gridLat = Math.floor(location.latitude / gridSize) * gridSize
      const gridLon = Math.floor(location.longitude / gridSize) * gridSize
      const key = `${gridLat},${gridLon}`
      
      const existing = clusters.get(key) || {
        centerLat: gridLat + gridSize / 2,
        centerLon: gridLon + gridSize / 2,
        count: 0,
        categories: new Map<string, number>()
      }
      
      existing.count++
      const catCount = existing.categories.get(location.category) || 0
      existing.categories.set(location.category, catCount + 1)
      
      clusters.set(key, existing)
    }
    
    return Array.from(clusters.values())
      .sort((a, b) => b.count - a.count)
  }
}

// Usage example
const spatialIndex = new SpatialIndex()

const locations: Location[] = [
  {
    id: '1',
    name: 'Downtown Diner',
    latitude: 40.7580,
    longitude: -73.9855,
    category: 'restaurant',
    rating: 4.2
  },
  {
    id: '2',
    name: 'City Hotel',
    latitude: 40.7505,
    longitude: -73.9934,
    category: 'hotel',
    rating: 4.5
  },
  {
    id: '3',
    name: 'Corner Cafe',
    latitude: 40.7614,
    longitude: -73.9776,
    category: 'restaurant',
    rating: 4.7
  },
  {
    id: '4',
    name: 'Metro Hospital',
    latitude: 40.7691,
    longitude: -73.9810,
    category: 'hospital',
    rating: 4.0
  }
]

const populated = locations.reduce(
  (index, location) => index.addLocation(location),
  spatialIndex
)

// Find nearest restaurants to Times Square
const nearbyRestaurants = populated.findNearest(
  { lat: 40.7580, lon: -73.9855 },
  5,
  'restaurant'
)
console.log('Nearby restaurants:', nearbyRestaurants)

// Find all locations within 1km
const within1km = populated.findWithinRadius(
  { lat: 40.7580, lon: -73.9855 },
  1.0
)
console.log('Within 1km:', within1km)

// Generate clusters for heatmap
const clusters = populated.generateClusters(0.01)
console.log('Clusters:', clusters)
```

## Advanced Features Deep Dive

### Range Operations: Efficient Subset Queries

SortedSet provides powerful range query capabilities for extracting ordered subsets based on the sorting criteria:

#### Basic Range Operations

```typescript
import { SortedSet, Order, pipe } from "effect"

const scores = SortedSet.fromIterable(
  Order.number,
  [85, 92, 78, 95, 88, 91, 76, 89, 94, 87]
)

// Get scores within a range
const midRangeScores = Array.from(SortedSet.values(scores))
  .filter(score => score >= 85 && score <= 92)
// [85, 87, 88, 89, 91, 92] - automatically sorted

// Custom range operations using split operations
function betweenInclusive<T>(
  set: SortedSet.SortedSet<T>,
  min: T,
  max: T,
  order: Order.Order<T>
): SortedSet.SortedSet<T> {
  return Array.from(SortedSet.values(set))
    .filter(item => 
      Order.greaterThanOrEqualTo(order)(item, min) &&
      Order.lessThanOrEqualTo(order)(item, max)
    )
    .reduce(
      (acc, item) => SortedSet.add(acc, item),
      SortedSet.empty<T>(order)
    )
}

// Time range queries
const events = SortedSet.fromIterable(
  Order.Date,
  [
    new Date('2024-01-15T10:00:00'),
    new Date('2024-01-15T14:30:00'),
    new Date('2024-01-15T09:15:00'),
    new Date('2024-01-15T16:45:00'),
    new Date('2024-01-15T12:20:00')
  ]
)

const afternoonEvents = betweenInclusive(
  events,
  new Date('2024-01-15T12:00:00'),
  new Date('2024-01-15T18:00:00'),
  Order.Date
)
```

#### Real-World Range Query Example

```typescript
import { SortedSet, Order, pipe } from "effect"

// Student grade analysis system
interface Student {
  id: string
  name: string
  grade: number
  subject: string
}

class GradeAnalyzer {
  private students: SortedSet.SortedSet<Student>
  
  constructor() {
    // Sort by grade descending, then by name
    this.students = SortedSet.empty(
      Order.combine(
        Order.mapInput(Order.reverse(Order.number), (s: Student) => s.grade),
        Order.mapInput(Order.string, (s: Student) => s.name)
      )
    )
  }
  
  addStudent(student: Student): GradeAnalyzer {
    const analyzer = new GradeAnalyzer()
    analyzer.students = SortedSet.add(this.students, student)
    return analyzer
  }
  
  // Grade distribution analysis
  getGradeDistribution(): {
    excellent: Student[], // 90-100
    good: Student[],      // 80-89
    average: Student[],   // 70-79
    needsImprovement: Student[] // Below 70
  } {
    const all = Array.from(SortedSet.values(this.students))
    
    return {
      excellent: all.filter(s => s.grade >= 90),
      good: all.filter(s => s.grade >= 80 && s.grade < 90),
      average: all.filter(s => s.grade >= 70 && s.grade < 80),
      needsImprovement: all.filter(s => s.grade < 70)
    }
  }
  
  // Percentile calculations
  getPercentile(percentile: number): number {
    const grades = Array.from(SortedSet.values(this.students))
      .map(s => s.grade)
      .sort((a, b) => a - b) // Sort ascending for percentile
    
    const index = Math.ceil((percentile / 100) * grades.length) - 1
    return grades[Math.max(0, index)]
  }
  
  // Find students within grade range
  findStudentsInRange(minGrade: number, maxGrade: number): Student[] {
    return Array.from(SortedSet.values(this.students))
      .filter(s => s.grade >= minGrade && s.grade <= maxGrade)
  }
  
  // Statistical analysis
  getStatistics(): {
    count: number,
    mean: number,
    median: number,
    q1: number,
    q3: number,
    iqr: number
  } {
    const grades = Array.from(SortedSet.values(this.students))
      .map(s => s.grade)
      .sort((a, b) => a - b)
    
    const count = grades.length
    const mean = grades.reduce((sum, g) => sum + g, 0) / count
    
    const median = count % 2 === 0
      ? (grades[count / 2 - 1] + grades[count / 2]) / 2
      : grades[Math.floor(count / 2)]
    
    const q1 = grades[Math.floor(count * 0.25)]
    const q3 = grades[Math.floor(count * 0.75)]
    const iqr = q3 - q1
    
    return { count, mean, median, q1, q3, iqr }
  }
}

// Usage
const analyzer = new GradeAnalyzer()

const students: Student[] = [
  { id: '1', name: 'Alice', grade: 95, subject: 'Math' },
  { id: '2', name: 'Bob', grade: 87, subject: 'Math' },
  { id: '3', name: 'Charlie', grade: 92, subject: 'Math' },
  { id: '4', name: 'David', grade: 78, subject: 'Math' },
  { id: '5', name: 'Eve', grade: 88, subject: 'Math' }
]

const populated = students.reduce(
  (analyzer, student) => analyzer.addStudent(student),
  analyzer
)

console.log('Grade distribution:', populated.getGradeDistribution())
console.log('Statistics:', populated.getStatistics())
console.log('75th percentile:', populated.getPercentile(75))
```

#### Advanced Range: Dynamic Partitioning

```typescript
import { SortedSet, Order, pipe } from "effect"

// Dynamic data partitioning for large datasets
class PartitionedSortedSet<T> {
  private partitions: Map<string, SortedSet.SortedSet<T>> = new Map()
  private order: Order.Order<T>
  private partitionFn: (item: T) => string
  private maxPartitionSize: number
  
  constructor(
    order: Order.Order<T>,
    partitionFn: (item: T) => string,
    maxPartitionSize: number = 1000
  ) {
    this.order = order
    this.partitionFn = partitionFn
    this.maxPartitionSize = maxPartitionSize
  }
  
  add(item: T): PartitionedSortedSet<T> {
    const partitioned = new PartitionedSortedSet(
      this.order,
      this.partitionFn,
      this.maxPartitionSize
    )
    partitioned.partitions = new Map(this.partitions)
    
    const partitionKey = this.partitionFn(item)
    const partition = this.partitions.get(partitionKey) || 
      SortedSet.empty<T>(this.order)
    
    const updated = SortedSet.add(partition, item)
    
    // Check if partition needs splitting
    if (SortedSet.size(updated) > this.maxPartitionSize) {
      const split = this.splitPartition(updated, partitionKey)
      split.forEach((set, key) => partitioned.partitions.set(key, set))
    } else {
      partitioned.partitions.set(partitionKey, updated)
    }
    
    return partitioned
  }
  
  private splitPartition(
    partition: SortedSet.SortedSet<T>,
    baseKey: string
  ): Map<string, SortedSet.SortedSet<T>> {
    const items = Array.from(SortedSet.values(partition))
    const mid = Math.floor(items.length / 2)
    
    const left = items.slice(0, mid).reduce(
      (acc, item) => SortedSet.add(acc, item),
      SortedSet.empty<T>(this.order)
    )
    const right = items.slice(mid).reduce(
      (acc, item) => SortedSet.add(acc, item),
      SortedSet.empty<T>(this.order)
    )
    
    return new Map([
      [`${baseKey}_0`, left],
      [`${baseKey}_1`, right]
    ])
  }
  
  // Query across partitions
  findInRange(
    predicate: (item: T) => boolean
  ): T[] {
    const results: T[] = []
    
    for (const partition of this.partitions.values()) {
      for (const item of SortedSet.values(partition)) {
        if (predicate(item)) {
          results.push(item)
        }
      }
    }
    
    // Sort final results
    return results.sort(Order.compare(this.order))
  }
  
  getPartitionStats(): Array<{
    key: string,
    size: number,
    min: T | undefined,
    max: T | undefined
  }> {
    return Array.from(this.partitions.entries()).map(([key, partition]) => ({
      key,
      size: SortedSet.size(partition),
      min: SortedSet.min(partition)?._tag === 'Some' ? SortedSet.min(partition).value : undefined,
      max: SortedSet.max(partition)?._tag === 'Some' ? SortedSet.max(partition).value : undefined
    }))
  }
}

// Usage: Partitioned time series data
interface TimePoint {
  timestamp: Date
  value: number
  sensor: string
}

const timePartitioner = (point: TimePoint): string => {
  // Partition by day
  return point.timestamp.toISOString().split('T')[0]
}

const timeOrder = Order.mapInput(
  Order.Date,
  (p: TimePoint) => p.timestamp
)

const timeSeries = new PartitionedSortedSet(
  timeOrder,
  timePartitioner,
  100
)

// Add data points
const dataPoints: TimePoint[] = Array.from({ length: 500 }, (_, i) => ({
  timestamp: new Date(Date.now() + i * 60000), // Every minute
  value: Math.random() * 100,
  sensor: `sensor${i % 10}`
}))

const populated = dataPoints.reduce(
  (ts, point) => ts.add(point),
  timeSeries
)

console.log('Partition stats:', populated.getPartitionStats())

// Range query across partitions
const highValues = populated.findInRange(point => point.value > 80)
console.log('High values:', highValues.length)
```

### Custom Comparators: Complex Sorting Strategies

SortedSet's flexibility comes from its support for sophisticated ordering strategies:

#### Multi-Level Sorting

```typescript
import { SortedSet, Order, pipe } from "effect"

// Complex employee ranking system
interface Employee {
  id: string
  name: string
  department: string
  level: number
  performance: number
  yearsExperience: number
  lastReviewDate: Date
}

class EmployeeRanking {
  // Different ranking strategies
  private byPerformance: SortedSet.SortedSet<Employee>
  private byExperience: SortedSet.SortedSet<Employee>
  private byPromotionReadiness: SortedSet.SortedSet<Employee>
  
  constructor() {
    // Rank by performance, then experience
    this.byPerformance = SortedSet.empty(
      Order.combine(
        Order.mapInput(Order.reverse(Order.number), (e: Employee) => e.performance),
        Order.mapInput(Order.reverse(Order.number), (e: Employee) => e.yearsExperience)
      )
    )
    
    // Rank by experience, then performance
    this.byExperience = SortedSet.empty(
      Order.combine(
        Order.mapInput(Order.reverse(Order.number), (e: Employee) => e.yearsExperience),
        Order.mapInput(Order.reverse(Order.number), (e: Employee) => e.performance)
      )
    )
    
    // Complex promotion readiness score
    this.byPromotionReadiness = SortedSet.empty(
      Order.mapInput(
        Order.reverse(Order.number),
        (e: Employee) => this.calculatePromotionScore(e)
      )
    )
  }
  
  private calculatePromotionScore(employee: Employee): number {
    const performanceWeight = 0.4
    const experienceWeight = 0.3
    const levelWeight = 0.2
    const recencyWeight = 0.1
    
    // Experience factor (diminishing returns after 10 years)
    const expFactor = Math.min(1, employee.yearsExperience / 10)
    
    // Level factor (higher level = ready for next level)
    const levelFactor = employee.level / 10
    
    // Review recency factor
    const daysSinceReview = (Date.now() - employee.lastReviewDate.getTime()) / (1000 * 60 * 60 * 24)
    const recencyFactor = Math.max(0, 1 - daysSinceReview / 365) // Decay over a year
    
    return (
      employee.performance * performanceWeight +
      expFactor * 100 * experienceWeight +
      levelFactor * 100 * levelWeight +
      recencyFactor * 100 * recencyWeight
    )
  }
  
  addEmployee(employee: Employee): EmployeeRanking {
    const ranking = new EmployeeRanking()
    
    ranking.byPerformance = SortedSet.add(this.byPerformance, employee)
    ranking.byExperience = SortedSet.add(this.byExperience, employee)
    ranking.byPromotionReadiness = SortedSet.add(this.byPromotionReadiness, employee)
    
    return ranking
  }
  
  getTopPerformers(n: number): Employee[] {
    return Array.from(SortedSet.values(this.byPerformance)).slice(0, n)
  }
  
  getPromotionCandidates(n: number): Employee[] {
    return Array.from(SortedSet.values(this.byPromotionReadiness)).slice(0, n)
  }
  
  getMostExperienced(n: number): Employee[] {
    return Array.from(SortedSet.values(this.byExperience)).slice(0, n)
  }
  
  // Comparative analysis
  compareRankings(): {
    performanceTop10: string[],
    promotionTop10: string[],
    experienceTop10: string[],
    overlap: {
      performanceVsPromotion: number,
      performanceVsExperience: number,
      promotionVsExperience: number
    }
  } {
    const perfTop10 = this.getTopPerformers(10).map(e => e.id)
    const promTop10 = this.getPromotionCandidates(10).map(e => e.id)
    const expTop10 = this.getMostExperienced(10).map(e => e.id)
    
    const overlap = {
      performanceVsPromotion: perfTop10.filter(id => promTop10.includes(id)).length,
      performanceVsExperience: perfTop10.filter(id => expTop10.includes(id)).length,
      promotionVsExperience: promTop10.filter(id => expTop10.includes(id)).length
    }
    
    return {
      performanceTop10: perfTop10,
      promotionTop10: promTop10,
      experienceTop10: expTop10,
      overlap
    }
  }
}

// Dynamic ordering based on context
function createContextualOrder(
  context: 'performance' | 'promotion' | 'layoffs' | 'training'
): Order.Order<Employee> {
  switch (context) {
    case 'performance':
      return Order.combine(
        Order.mapInput(Order.reverse(Order.number), (e: Employee) => e.performance),
        Order.mapInput(Order.string, (e: Employee) => e.name)
      )
    
    case 'promotion':
      return Order.combine(
        Order.mapInput(Order.reverse(Order.number), (e: Employee) => e.performance),
        Order.mapInput(Order.reverse(Order.number), (e: Employee) => e.yearsExperience),
        Order.mapInput(Order.number, (e: Employee) => e.level)
      )
    
    case 'layoffs':
      // Controversial but realistic: performance ascending, experience ascending
      return Order.combine(
        Order.mapInput(Order.number, (e: Employee) => e.performance),
        Order.mapInput(Order.number, (e: Employee) => e.yearsExperience)
      )
    
    case 'training':
      // Junior employees with potential
      return Order.combine(
        Order.mapInput(Order.number, (e: Employee) => e.yearsExperience),
        Order.mapInput(Order.reverse(Order.number), (e: Employee) => e.performance)
      )
    
    default:
      return Order.mapInput(Order.string, (e: Employee) => e.name)
  }
}

// Usage
const ranking = new EmployeeRanking()

const employees: Employee[] = [
  {
    id: '1',
    name: 'Alice Johnson',
    department: 'Engineering',
    level: 5,
    performance: 4.8,
    yearsExperience: 8,
    lastReviewDate: new Date('2024-01-15')
  },
  {
    id: '2',
    name: 'Bob Smith',
    department: 'Engineering',
    level: 3,
    performance: 4.2,
    yearsExperience: 3,
    lastReviewDate: new Date('2024-02-01')
  },
  {
    id: '3',
    name: 'Carol Davis',
    department: 'Marketing',
    level: 6,
    performance: 4.5,
    yearsExperience: 12,
    lastReviewDate: new Date('2023-12-20')
  }
]

const populated = employees.reduce(
  (ranking, employee) => ranking.addEmployee(employee),
  ranking
)

console.log('Top performers:', populated.getTopPerformers(2))
console.log('Promotion candidates:', populated.getPromotionCandidates(2))
console.log('Ranking comparison:', populated.compareRankings())

// Contextual ordering
const promotionContext = SortedSet.fromIterable(
  createContextualOrder('promotion'),
  employees
)

console.log('Promotion order:', Array.from(SortedSet.values(promotionContext)))
```

#### Domain-Specific Comparators

```typescript
import { SortedSet, Order, pipe } from "effect"

// Financial instrument ordering
interface Bond {
  id: string
  issuer: string
  maturityDate: Date
  couponRate: number
  creditRating: 'AAA' | 'AA' | 'A' | 'BBB' | 'BB' | 'B' | 'CCC'
  yieldToMaturity: number
  duration: number
}

const creditRatingOrder = Order.mapInput(
  Order.number,
  (rating: Bond['creditRating']) => {
    const ratings = { 'AAA': 1, 'AA': 2, 'A': 3, 'BBB': 4, 'BB': 5, 'B': 6, 'CCC': 7 }
    return ratings[rating]
  }
)

const bondOrder = Order.combine(
  Order.mapInput(creditRatingOrder, (b: Bond) => b.creditRating),
  Order.mapInput(Order.Date, (b: Bond) => b.maturityDate),
  Order.mapInput(Order.reverse(Order.number), (b: Bond) => b.yieldToMaturity)
)

const bondPortfolio = SortedSet.empty<Bond>(bondOrder)

// IP address ordering
interface NetworkDevice {
  ip: string
  hostname: string
  status: 'online' | 'offline' | 'maintenance'
}

const ipOrder = Order.mapInput(
  Order.tuple(Order.number, Order.number, Order.number, Order.number),
  (device: NetworkDevice) => {
    const parts = device.ip.split('.').map(Number) as [number, number, number, number]
    return parts
  }
)

const networkOrder = Order.combine(
  Order.mapInput(Order.string, (d: NetworkDevice) => d.status), // Group by status
  ipOrder
)

const network = SortedSet.empty<NetworkDevice>(networkOrder)

// URL path ordering for routing
interface Route {
  path: string
  method: 'GET' | 'POST' | 'PUT' | 'DELETE'
  priority: number
  wildcards: number
}

const routeOrder = Order.combine(
  // Exact matches first (fewer wildcards)
  Order.mapInput(Order.number, (r: Route) => r.wildcards),
  // Then by priority (higher first)
  Order.mapInput(Order.reverse(Order.number), (r: Route) => r.priority),
  // Then by path length (longer/more specific first)
  Order.mapInput(Order.reverse(Order.number), (r: Route) => r.path.length),
  // Finally alphabetical
  Order.mapInput(Order.string, (r: Route) => r.path)
)

const routingTable = SortedSet.empty<Route>(routeOrder)

// Game leaderboard with tie-breaking
interface GameScore {
  playerId: string
  playerName: string
  score: number
  level: number
  completionTime: number // milliseconds
  achievements: number
}

const leaderboardOrder = Order.combine(
  // Primary: score (highest first)
  Order.mapInput(Order.reverse(Order.number), (g: GameScore) => g.score),
  // Tie-breaker 1: level (highest first)
  Order.mapInput(Order.reverse(Order.number), (g: GameScore) => g.level),
  // Tie-breaker 2: completion time (fastest first)
  Order.mapInput(Order.number, (g: GameScore) => g.completionTime),
  // Tie-breaker 3: achievements (most first)
  Order.mapInput(Order.reverse(Order.number), (g: GameScore) => g.achievements),
  // Final tie-breaker: player name (alphabetical)
  Order.mapInput(Order.string, (g: GameScore) => g.playerName)
)

const leaderboard = SortedSet.empty<GameScore>(leaderboardOrder)
```

## Practical Patterns & Best Practices

### Pattern 1: Efficient Bulk Operations and Transformations

```typescript
import { SortedSet, Order, pipe, Option } from "effect"

// Batch operations helper
function bulkAdd<T>(
  set: SortedSet.SortedSet<T>,
  items: Iterable<T>
): SortedSet.SortedSet<T> {
  return Array.from(items).reduce(
    (acc, item) => SortedSet.add(acc, item),
    set
  )
}

// Efficient set transformations
function mapSortedSet<A, B>(
  set: SortedSet.SortedSet<A>,
  f: (a: A) => B,
  newOrder: Order.Order<B>
): SortedSet.SortedSet<B> {
  return Array.from(SortedSet.values(set))
    .map(f)
    .reduce(
      (acc, item) => SortedSet.add(acc, item),
      SortedSet.empty<B>(newOrder)
    )
}

// Filter with early termination
function takeWhile<T>(
  set: SortedSet.SortedSet<T>,
  predicate: (item: T) => boolean
): SortedSet.SortedSet<T> {
  const order = set[Symbol.for("SortedSet/Order")] // Internal access pattern
  const result = SortedSet.empty<T>(order)
  
  for (const item of SortedSet.values(set)) {
    if (!predicate(item)) break
    SortedSet.add(result, item)
  }
  
  return result
}

// Efficient set merging with custom conflict resolution
function mergeSortedSets<T>(
  sets: Array<SortedSet.SortedSet<T>>,
  order: Order.Order<T>,
  onDuplicate?: (existing: T, incoming: T) => T
): SortedSet.SortedSet<T> {
  if (sets.length === 0) return SortedSet.empty(order)
  if (sets.length === 1) return sets[0]
  
  // Use merge sort approach for efficiency
  let result = sets[0]
  
  for (let i = 1; i < sets.length; i++) {
    result = SortedSet.union(result, sets[i])
  }
  
  return result
}

// Sliding window operations
function* slidingWindow<T>(
  set: SortedSet.SortedSet<T>,
  windowSize: number
): Generator<Array<T>, void, unknown> {
  const items = Array.from(SortedSet.values(set))
  
  for (let i = 0; i <= items.length - windowSize; i++) {
    yield items.slice(i, i + windowSize)
  }
}

// Moving statistics over sorted data
function movingStatistics<T>(
  set: SortedSet.SortedSet<T>,
  extractor: (item: T) => number,
  windowSize: number
): Array<{
  window: Array<T>,
  mean: number,
  min: number,
  max: number,
  stddev: number
}> {
  const windows = Array.from(slidingWindow(set, windowSize))
  
  return windows.map(window => {
    const values = window.map(extractor)
    const mean = values.reduce((sum, v) => sum + v, 0) / values.length
    const min = Math.min(...values)
    const max = Math.max(...values)
    
    const variance = values.reduce((sum, v) => sum + Math.pow(v - mean, 2), 0) / values.length
    const stddev = Math.sqrt(variance)
    
    return { window, mean, min, max, stddev }
  })
}

// Usage example: Time series analysis
interface DataPoint {
  timestamp: Date
  value: number
  sensor: string
}

const timeSeries = SortedSet.fromIterable(
  Order.mapInput(Order.Date, (p: DataPoint) => p.timestamp),
  [
    { timestamp: new Date('2024-01-01T10:00'), value: 23.5, sensor: 'temp1' },
    { timestamp: new Date('2024-01-01T10:05'), value: 24.1, sensor: 'temp1' },
    { timestamp: new Date('2024-01-01T10:10'), value: 23.8, sensor: 'temp1' },
    { timestamp: new Date('2024-01-01T10:15'), value: 25.2, sensor: 'temp1' },
    { timestamp: new Date('2024-01-01T10:20'), value: 24.7, sensor: 'temp1' }
  ]
)

// Moving average with window size of 3
const movingStats = movingStatistics(
  timeSeries,
  point => point.value,
  3
)

console.log('Moving statistics:', movingStats)

// Transform to normalized values
const normalizedSeries = mapSortedSet(
  timeSeries,
  point => ({
    ...point,
    normalizedValue: (point.value - 23) / 2 // Simple normalization
  }),
  Order.mapInput(Order.Date, (p) => p.timestamp)
)
```

### Pattern 2: Memory-Efficient Large Collections

```typescript
import { SortedSet, Order, pipe, Option } from "effect"

// Paginated sorted set for large datasets
class PaginatedSortedSet<T> {
  private pages: Map<number, SortedSet.SortedSet<T>> = new Map()
  private pageSize: number
  private order: Order.Order<T>
  private totalSize: number = 0
  
  constructor(pageSize: number, order: Order.Order<T>) {
    this.pageSize = pageSize
    this.order = order
  }
  
  add(item: T): PaginatedSortedSet<T> {
    const paginated = new PaginatedSortedSet(this.pageSize, this.order)
    paginated.pages = new Map(this.pages)
    paginated.totalSize = this.totalSize
    
    // Find appropriate page
    const targetPage = this.findTargetPage(item)
    const page = this.pages.get(targetPage) || SortedSet.empty(this.order)
    
    if (!SortedSet.has(page, item)) {
      paginated.totalSize++
    }
    
    const updatedPage = SortedSet.add(page, item)
    
    // Check if page needs splitting
    if (SortedSet.size(updatedPage) > this.pageSize) {
      const split = this.splitPage(updatedPage, targetPage)
      split.forEach((set, pageNum) => paginated.pages.set(pageNum, set))
    } else {
      paginated.pages.set(targetPage, updatedPage)
    }
    
    return paginated
  }
  
  private findTargetPage(item: T): number {
    for (const [pageNum, page] of this.pages) {
      const maxItem = SortedSet.max(page)
      if (Option.isNone(maxItem) || Order.lessThanOrEqualTo(this.order)(item, maxItem.value)) {
        return pageNum
      }
    }
    return this.pages.size
  }
  
  private splitPage(
    page: SortedSet.SortedSet<T>,
    pageNum: number
  ): Map<number, SortedSet.SortedSet<T>> {
    const items = Array.from(SortedSet.values(page))
    const mid = Math.floor(items.length / 2)
    
    const leftItems = items.slice(0, mid)
    const rightItems = items.slice(mid)
    
    const leftPage = leftItems.reduce(
      (acc, item) => SortedSet.add(acc, item),
      SortedSet.empty<T>(this.order)
    )
    const rightPage = rightItems.reduce(
      (acc, item) => SortedSet.add(acc, item),
      SortedSet.empty<T>(this.order)
    )
    
    // Shift existing pages
    const result = new Map<number, SortedSet.SortedSet<T>>()
    for (const [pNum, pSet] of this.pages) {
      if (pNum < pageNum) {
        result.set(pNum, pSet)
      } else if (pNum > pageNum) {
        result.set(pNum + 1, pSet)
      }
    }
    
    result.set(pageNum, leftPage)
    result.set(pageNum + 1, rightPage)
    
    return result
  }
  
  *iterate(): Generator<T, void, unknown> {
    const pageNums = Array.from(this.pages.keys()).sort((a, b) => a - b)
    
    for (const pageNum of pageNums) {
      const page = this.pages.get(pageNum)!
      yield* SortedSet.values(page)
    }
  }
  
  getPage(pageNumber: number, pageSize: number = this.pageSize): T[] {
    const allItems = Array.from(this.iterate())
    const start = pageNumber * pageSize
    return allItems.slice(start, start + pageSize)
  }
  
  size(): number {
    return this.totalSize
  }
}

// Lazy sorted set with on-demand loading
class LazySortedSet<T> {
  private loadedChunks: Map<string, SortedSet.SortedSet<T>> = new Map()
  private order: Order.Order<T>
  private loader: (chunkKey: string) => Promise<T[]>
  private chunkSize: number
  
  constructor(
    order: Order.Order<T>,
    loader: (chunkKey: string) => Promise<T[]>,
    chunkSize: number = 1000
  ) {
    this.order = order
    this.loader = loader
    this.chunkSize = chunkSize
  }
  
  async getRange(
    chunkKeys: string[]
  ): Promise<SortedSet.SortedSet<T>> {
    let result = SortedSet.empty<T>(this.order)
    
    for (const chunkKey of chunkKeys) {
      if (!this.loadedChunks.has(chunkKey)) {
        const items = await this.loader(chunkKey)
        const chunk = items.reduce(
          (acc, item) => SortedSet.add(acc, item),
          SortedSet.empty<T>(this.order)
        )
        this.loadedChunks.set(chunkKey, chunk)
      }
      
      const chunk = this.loadedChunks.get(chunkKey)!
      result = SortedSet.union(result, chunk)
    }
    
    return result
  }
  
  async preload(chunkKeys: string[]): Promise<void> {
    const promises = chunkKeys
      .filter(key => !this.loadedChunks.has(key))
      .map(key => this.getRange([key]))
    
    await Promise.all(promises)
  }
  
  getLoadedChunks(): string[] {
    return Array.from(this.loadedChunks.keys())
  }
}

// Usage: Large dataset handling
const hugeSortedSet = new PaginatedSortedSet(100, Order.number)

// Add many items
const populated = Array.from({ length: 10000 }, (_, i) => i)
  .reduce((set, num) => set.add(num), hugeSortedSet)

console.log('Total size:', populated.size())
console.log('Page 5:', populated.getPage(5, 20))

// Lazy loading example
const lazySet = new LazySortedSet(
  Order.string,
  async (chunkKey: string) => {
    // Simulate database query
    console.log(`Loading chunk: ${chunkKey}`)
    return [`item_${chunkKey}_1`, `item_${chunkKey}_2`, `item_${chunkKey}_3`]
  }
)

// Usage would be:
// const results = await lazySet.getRange(['chunk1', 'chunk2'])
```

### Pattern 3: Set Algebra and Composite Operations

```typescript
import { SortedSet, Order, pipe } from "effect"

// Advanced set operations
class SetAlgebra {
  static symmetricDifference<T>(
    set1: SortedSet.SortedSet<T>,
    set2: SortedSet.SortedSet<T>
  ): SortedSet.SortedSet<T> {
    const union = SortedSet.union(set1, set2)
    const intersection = SortedSet.intersection(set1, set2)
    return SortedSet.difference(union, intersection)
  }
  
  static cartesianProduct<A, B>(
    setA: SortedSet.SortedSet<A>,
    setB: SortedSet.SortedSet<B>,
    pairOrder: Order.Order<[A, B]>
  ): SortedSet.SortedSet<[A, B]> {
    let result = SortedSet.empty<[A, B]>(pairOrder)
    
    for (const a of SortedSet.values(setA)) {
      for (const b of SortedSet.values(setB)) {
        result = SortedSet.add(result, [a, b])
      }
    }
    
    return result
  }
  
  static powerSet<T>(
    set: SortedSet.SortedSet<T>,
    subsetOrder: Order.Order<SortedSet.SortedSet<T>>
  ): SortedSet.SortedSet<SortedSet.SortedSet<T>> {
    const items = Array.from(SortedSet.values(set))
    const originalOrder = set[Symbol.for("SortedSet/Order")] // Access pattern
    
    let result = SortedSet.empty<SortedSet.SortedSet<T>>(subsetOrder)
    
    // Generate all 2^n subsets
    for (let i = 0; i < Math.pow(2, items.length); i++) {
      let subset = SortedSet.empty<T>(originalOrder)
      
      for (let j = 0; j < items.length; j++) {
        if (i & (1 << j)) {
          subset = SortedSet.add(subset, items[j])
        }
      }
      
      result = SortedSet.add(result, subset)
    }
    
    return result
  }
  
  // Multi-way operations
  static unionAll<T>(
    sets: Array<SortedSet.SortedSet<T>>
  ): SortedSet.SortedSet<T> {
    if (sets.length === 0) {
      throw new Error("Cannot compute union of empty array")
    }
    
    return sets.reduce((acc, set) => SortedSet.union(acc, set))
  }
  
  static intersectionAll<T>(
    sets: Array<SortedSet.SortedSet<T>>
  ): SortedSet.SortedSet<T> {
    if (sets.length === 0) {
      throw new Error("Cannot compute intersection of empty array")
    }
    
    return sets.reduce((acc, set) => SortedSet.intersection(acc, set))
  }
  
  // Jaccard similarity
  static jaccardSimilarity<T>(
    set1: SortedSet.SortedSet<T>,
    set2: SortedSet.SortedSet<T>
  ): number {
    const intersection = SortedSet.intersection(set1, set2)
    const union = SortedSet.union(set1, set2)
    
    if (SortedSet.size(union) === 0) return 1 // Both empty
    
    return SortedSet.size(intersection) / SortedSet.size(union)
  }
}

// Usage: Document similarity analysis
interface Document {
  id: string
  title: string
  tags: string[]
}

class DocumentSimilarity {
  private documents: Map<string, SortedSet.SortedSet<string>> = new Map()
  
  addDocument(doc: Document): void {
    const tagSet = SortedSet.fromIterable(Order.string, doc.tags)
    this.documents.set(doc.id, tagSet)
  }
  
  findSimilarDocuments(
    docId: string,
    threshold: number = 0.3
  ): Array<{ id: string, similarity: number }> {
    const targetDoc = this.documents.get(docId)
    if (!targetDoc) return []
    
    const similarities: Array<{ id: string, similarity: number }> = []
    
    for (const [otherId, otherDoc] of this.documents) {
      if (otherId === docId) continue
      
      const similarity = SetAlgebra.jaccardSimilarity(targetDoc, otherDoc)
      if (similarity >= threshold) {
        similarities.push({ id: otherId, similarity })
      }
    }
    
    return similarities.sort((a, b) => b.similarity - a.similarity)
  }
  
  getDocumentClusters(
    threshold: number = 0.5
  ): Array<Array<string>> {
    const visited = new Set<string>()
    const clusters: Array<Array<string>> = []
    
    for (const docId of this.documents.keys()) {
      if (visited.has(docId)) continue
      
      const cluster = [docId]
      visited.add(docId)
      
      const similar = this.findSimilarDocuments(docId, threshold)
      for (const { id } of similar) {
        if (!visited.has(id)) {
          cluster.push(id)
          visited.add(id)
        }
      }
      
      clusters.push(cluster)
    }
    
    return clusters.sort((a, b) => b.length - a.length)
  }
}

// Usage
const docSimilarity = new DocumentSimilarity()

const documents: Document[] = [
  { id: '1', title: 'React Guide', tags: ['react', 'javascript', 'frontend', 'web'] },
  { id: '2', title: 'Vue Tutorial', tags: ['vue', 'javascript', 'frontend', 'web'] },
  { id: '3', title: 'Node.js Basics', tags: ['nodejs', 'javascript', 'backend', 'server'] },
  { id: '4', title: 'Angular Forms', tags: ['angular', 'typescript', 'frontend', 'web'] }
]

documents.forEach(doc => docSimilarity.addDocument(doc))

console.log('Similar to React:', docSimilarity.findSimilarDocuments('1'))
console.log('Document clusters:', docSimilarity.getDocumentClusters(0.3))
```

## Integration Examples

### Integration with Effect Ecosystem

```typescript
import { SortedSet, Order, pipe, Effect, Option, Either, Schema, Stream } from "effect"

// Schema integration for validated sorted sets
const ScoreSchema = Schema.between(
  Schema.int(Schema.number),
  0,
  100
)

const PlayerSchema = Schema.struct({
  id: Schema.string,
  name: Schema.string,
  score: ScoreSchema,
  level: Schema.positive(Schema.number)
})

type Player = Schema.Schema.Type<typeof PlayerSchema>

class ValidatedLeaderboard {
  private players: SortedSet.SortedSet<Player>
  
  constructor() {
    this.players = SortedSet.empty(
      Order.combine(
        Order.mapInput(Order.reverse(Order.number), (p: Player) => p.score),
        Order.mapInput(Order.string, (p: Player) => p.name)
      )
    )
  }
  
  addPlayer(data: unknown): Effect.Effect<ValidatedLeaderboard, Schema.ParseError> {
    return Effect.map(
      Schema.decodeUnknown(PlayerSchema)(data),
      player => {
        const updated = SortedSet.add(this.players, player)
        return new ValidatedLeaderboard().withPlayers(updated)
      }
    )
  }
  
  updateScore(
    playerId: string,
    newScore: unknown
  ): Effect.Effect<ValidatedLeaderboard, Schema.ParseError | string> {
    return Effect.gen(function* () {
      const score = yield* Schema.decodeUnknown(ScoreSchema)(newScore)
      
      const existing = Array.from(SortedSet.values(this.players))
        .find(p => p.id === playerId)
      
      if (!existing) {
        return yield* Effect.fail(`Player ${playerId} not found`)
      }
      
      const updated = { ...existing, score }
      const newPlayers = SortedSet.add(
        SortedSet.remove(this.players, existing),
        updated
      )
      
      return new ValidatedLeaderboard().withPlayers(newPlayers)
    })
  }
  
  getTopPlayers(n: number): Effect.Effect<ReadonlyArray<Player>, never> {
    return Effect.succeed(
      Array.from(SortedSet.values(this.players)).slice(0, n)
    )
  }
  
  private withPlayers(players: SortedSet.SortedSet<Player>): ValidatedLeaderboard {
    this.players = players
    return this
  }
}

// Stream integration for real-time sorted data
const createSortedStream = <T>(
  items: ReadonlyArray<T>,
  order: Order.Order<T>
): Stream.Stream<SortedSet.SortedSet<T>, never, never> => {
  return Stream.scan(
    Stream.fromIterable(items),
    SortedSet.empty<T>(order),
    (acc, item) => SortedSet.add(acc, item)
  )
}

// Real-time leaderboard updates
const processGameEvents = (
  events: Stream.Stream<{ playerId: string, scoreChange: number }, never, never>
): Stream.Stream<SortedSet.SortedSet<Player>, never, never> => {
  const initialBoard = SortedSet.empty<Player>(
    Order.mapInput(Order.reverse(Order.number), (p: Player) => p.score)
  )
  
  return Stream.scan(
    events,
    initialBoard,
    (leaderboard, event) => {
        // Find and update player (simplified)
        const players = Array.from(SortedSet.values(leaderboard))
        const playerIndex = players.findIndex(p => p.id === event.playerId)
        
        if (playerIndex === -1) return leaderboard
        
        const player = players[playerIndex]
        const updated = {
          ...player,
          score: Math.max(0, Math.min(100, player.score + event.scoreChange))
        }
        
        return SortedSet.add(
          SortedSet.remove(leaderboard, player),
          updated
        )
      }
    )
  )
}

// Effect-based batch operations
const bulkUpdateLeaderboard = (
  leaderboard: ValidatedLeaderboard,
  updates: Array<{ playerId: string, score: number }>
): Effect.Effect<ValidatedLeaderboard, Schema.ParseError | string> =>
  Effect.reduce(
    updates,
    leaderboard,
    (board, update) => board.updateScore(update.playerId, update.score)
  )

// Usage example
const leaderboard = new ValidatedLeaderboard()

const program = Effect.gen(function* () {
  let board = yield* leaderboard.addPlayer({
    id: '1',
    name: 'Alice',
    score: 95,
    level: 5
  })
  
  board = yield* board.addPlayer({
    id: '2',
    name: 'Bob',
    score: 87,
    level: 3
  })
  
  const players = yield* board.getTopPlayers(10)
  console.log('Top players:', players)
  
  return players
}

Effect.catchAll(program, error => Effect.log(`Error: ${error}`))

// Effect.runSync(program) // Uncomment to run
```

### Testing Strategies

```typescript
import { SortedSet, Order, pipe } from "effect"
import { it, expect, describe } from "@effect/vitest"
import * as fc from "fast-check"

describe("SortedSet Testing", () => {
  // Property-based testing for ordering invariants
  it("maintains sort order after operations", () => {
    fc.assert(
      fc.property(
        fc.array(fc.integer()),
        (numbers) => {
          const set = SortedSet.fromIterable(Order.number, numbers)
          const values = Array.from(SortedSet.values(set))
          
          // Check if sorted
          for (let i = 1; i < values.length; i++) {
            expect(values[i]).toBeGreaterThanOrEqual(values[i - 1])
          }
        }
      )
    )
  })
  
  // Test uniqueness property
  it("maintains uniqueness", () => {
    fc.assert(
      fc.property(
        fc.array(fc.string()),
        (strings) => {
          const set = SortedSet.fromIterable(Order.string, strings)
          const values = Array.from(SortedSet.values(set))
          const uniqueValues = [...new Set(values)]
          
          expect(values.length).toBe(uniqueValues.length)
        }
      )
    )
  })
  
  // Test set operations properties
  it("union is commutative", () => {
    fc.assert(
      fc.property(
        fc.array(fc.integer()),
        fc.array(fc.integer()),
        (arr1, arr2) => {
          const set1 = SortedSet.fromIterable(Order.number, arr1)
          const set2 = SortedSet.fromIterable(Order.number, arr2)
          
          const union1 = SortedSet.union(set1, set2)
          const union2 = SortedSet.union(set2, set1)
          
          const values1 = Array.from(SortedSet.values(union1))
          const values2 = Array.from(SortedSet.values(union2))
          
          expect(values1).toEqual(values2)
        }
      )
    )
  })
  
  // Test intersection properties
  it("intersection with self returns self", () => {
    fc.assert(
      fc.property(
        fc.array(fc.string()),
        (strings) => {
          const set = SortedSet.fromIterable(Order.string, strings)
          const intersection = SortedSet.intersection(set, set)
          
          const originalValues = Array.from(SortedSet.values(set))
          const intersectionValues = Array.from(SortedSet.values(intersection))
          
          expect(intersectionValues).toEqual(originalValues)
        }
      )
    )
  })
  
  // Custom comparator testing
  it("respects custom ordering", () => {
    const reverseOrder = Order.reverse(Order.number)
    const set = pipe(
      SortedSet.add(SortedSet.empty<number>(reverseOrder), 1),
      SortedSet.add(3),
      SortedSet.add(2)
    )
    
    const values = Array.from(SortedSet.values(set))
    expect(values).toEqual([3, 2, 1])
  })
  
  // Performance benchmarks
  it.skip("performance characteristics", () => {
    const size = 10000
    const numbers = Array.from({ length: size }, (_, i) => i)
    
    // SortedSet construction
    const start1 = performance.now()
    const sortedSet = SortedSet.fromIterable(Order.number, numbers)
    const end1 = performance.now()
    console.log(`SortedSet construction: ${end1 - start1}ms`)
    
    // Native Set construction + sorting
    const start2 = performance.now()
    const nativeSet = new Set(numbers)
    const sorted = Array.from(nativeSet).sort((a, b) => a - b)
    const end2 = performance.now()
    console.log(`Native Set + sort: ${end2 - start2}ms`)
    
    // Membership testing
    const start3 = performance.now()
    for (let i = 0; i < 1000; i++) {
      SortedSet.has(sortedSet, Math.floor(Math.random() * size))
    }
    const end3 = performance.now()
    console.log(`SortedSet membership (1000): ${end3 - start3}ms`)
  })
  
  // Complex scenario testing
  it("handles complex object ordering", () => {
    interface Task {
      id: string
      priority: number
      name: string
    }
    
    const taskOrder = Order.combine(
      Order.mapInput(Order.reverse(Order.number), (t: Task) => t.priority),
      Order.mapInput(Order.string, (t: Task) => t.name)
    )
    
    const tasks: Task[] = [
      { id: '1', priority: 2, name: 'Task B' },
      { id: '2', priority: 3, name: 'Task A' },
      { id: '3', priority: 2, name: 'Task A' },
      { id: '4', priority: 3, name: 'Task B' }
    ]
    
    const taskSet = SortedSet.fromIterable(taskOrder, tasks)
    const sorted = Array.from(SortedSet.values(taskSet))
    
    expect(sorted).toEqual([
      { id: '2', priority: 3, name: 'Task A' },
      { id: '4', priority: 3, name: 'Task B' },
      { id: '3', priority: 2, name: 'Task A' },
      { id: '1', priority: 2, name: 'Task B' }
    ])
  })
})

// Test utilities for sorted set operations
const generateSortedSet = <T>(
  itemArb: fc.Arbitrary<T>,
  order: Order.Order<T>
): fc.Arbitrary<SortedSet.SortedSet<T>> =>
  fc.array(itemArb).map(items =>
    SortedSet.fromIterable(order, items)
  )

// Mock data generators
const mockPlayer = fc.record({
  id: fc.string(),
  name: fc.string(),
  score: fc.integer({ min: 0, max: 100 }),
  level: fc.integer({ min: 1, max: 10 })
})

const mockLeaderboard = generateSortedSet(
  mockPlayer,
  Order.mapInput(Order.reverse(Order.number), (p) => p.score)
)

// Regression tests for specific scenarios
describe("SortedSet Edge Cases", () => {
  it("handles empty set operations", () => {
    const empty1 = SortedSet.empty<number>(Order.number)
    const empty2 = SortedSet.empty<number>(Order.number)
    const nonEmpty = SortedSet.fromIterable(Order.number, [1, 2, 3])
    
    expect(SortedSet.size(SortedSet.union(empty1, empty2))).toBe(0)
    expect(SortedSet.size(SortedSet.intersection(empty1, nonEmpty))).toBe(0)
    expect(Array.from(SortedSet.values(SortedSet.union(empty1, nonEmpty)))).toEqual([1, 2, 3])
  })
  
  it("handles single element sets", () => {
    const single = SortedSet.fromIterable(Order.string, ['only'])
    
    expect(SortedSet.size(single)).toBe(1)
    expect(SortedSet.has(single, 'only')).toBe(true)
    expect(SortedSet.has(single, 'other')).toBe(false)
  })
})
```

## Conclusion

SortedSet provides automatic ordering, efficient set operations, and unique constraint enforcement for building sophisticated applications that require both uniqueness and ordering guarantees.

Key benefits:
- **Automatic Ordering**: Elements stay sorted according to your custom comparator without manual maintenance
- **Efficient Set Operations**: Union, intersection, and difference operations optimized for sorted data structures
- **Unique Constraint Enforcement**: Combines the benefits of Set (uniqueness) with automatic sorting
- **Type-Safe Comparators**: Order instances provide compile-time guarantees about element comparison and sorting behavior

SortedSet is the perfect choice when you need persistent ordered collections with uniqueness constraints, priority queues, or any scenario where maintaining both order and uniqueness is critical to your application's functionality.