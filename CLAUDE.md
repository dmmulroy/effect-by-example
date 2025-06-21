# Effect Module Documentation Guide

This guide provides a template and best practices for creating comprehensive, real-world focused documentation for Effect modules, with specific guidelines for Effect-TS syntax patterns.

## When to Use This Template

Use this template when documenting:
- Core Effect modules (@effect/schema, @effect/platform, etc.)
- Experimental Effect modules (@effect/experimental)
- Effect ecosystem libraries
- Complex Effect patterns or integrations

## Effect Module Documentation Template

```markdown
# [ModuleName]: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem [ModuleName] Solves

[Start with a concrete problem developers face. Show the pain with traditional approaches]

```typescript
// Traditional approach - show the problematic code
// Highlight issues: boilerplate, error-prone, type-unsafe, etc.
```

This approach leads to:
- **[Problem 1]** - Brief explanation
- **[Problem 2]** - Brief explanation
- **[Problem 3]** - Brief explanation

### The [ModuleName] Solution

[Introduce how the module solves these problems]

```typescript
import { [ModuleName] } from "@effect/[package]"

// Show the Effect solution - clean, type-safe, composable
```

### Key Concepts

**[Concept 1]**: Brief definition with inline example

**[Concept 2]**: Brief definition with inline example

**[Concept 3]**: Brief definition with inline example

## Basic Usage Patterns

### [Pattern 1: Setup/Initialization]

```typescript
// Show the simplest possible usage
// Include all necessary imports
// Use clear variable names
```

### [Pattern 2: Core Feature]

```typescript
// Demonstrate the main feature
// Build incrementally from Pattern 1
// Add inline comments for clarity
```

### [Pattern 3: Common Operations]

```typescript
// Show frequently used operations
// Include type annotations where helpful
```

## Real-World Examples

### Example 1: [Common Use Case - e.g., Database Operations]

[Brief context of why this example matters]

```typescript
// Complete, runnable example
// Shows real patterns developers use
// Includes error handling
```

### Example 2: [Another Use Case - e.g., API Integration]

```typescript
// Different domain to show versatility
// Demonstrates composability with other Effect modules
```

### Example 3: [Complex Scenario]

```typescript
// More advanced but still practical
// Shows how pieces fit together
```

## Advanced Features Deep Dive

### [Feature 1]: [Descriptive Name]

[Explain what it does and when to use it]

#### Basic [Feature] Usage

```typescript
// Simple example focusing on this feature
```

#### Real-World [Feature] Example

```typescript
// Practical application
// Show the "why" not just the "how"
```

#### Advanced [Feature]: [Specific Technique]

```typescript
// Power-user techniques
// Include helper functions if applicable
```

## Practical Patterns & Best Practices

### Pattern 1: [Common Pattern Name]

```typescript
// Helper functions or utilities
// Reusable abstractions
// Explain the pattern's benefits
```

### Pattern 2: [Another Pattern]

```typescript
// Show composition patterns
// Error handling strategies
// Performance considerations
```

## Integration Examples

### Integration with [Popular Library/Framework]

```typescript
// Show how to use with common tools
// Include necessary adapters or converters
// Demonstrate bidirectional data flow
```

### Testing Strategies

```typescript
// Test utilities and patterns
// Property-based testing if applicable
// Mock/stub strategies
```

## Conclusion

[ModuleName] provides [key benefit 1], [key benefit 2], and [key benefit 3] for [target use case].

Key benefits:
- **[Benefit 1]**: Explanation
- **[Benefit 2]**: Explanation  
- **[Benefit 3]**: Explanation

[Closing statement about when to use this module]
```

## Writing Guidelines

1. **Start with Why**: Always lead with the problem before the solution
2. **Show, Don't Tell**: Use code examples to demonstrate concepts
3. **Progressive Disclosure**: Simple → Intermediate → Advanced
4. **Real-World Focus**: Every example should solve actual developer problems
5. **Type Safety**: Show inferred types to demonstrate type safety benefits
6. **Composability**: Demonstrate how the module works with other Effect modules
7. **Helper Patterns**: Create reusable helpers that readers can copy
8. **Integration Examples**: Show how to use with popular libraries
9. **Comments**: Use inline comments sparingly but effectively
10. **Consistency**: Use consistent naming and formatting throughout

## Code Example Guidelines

1. **Complete Examples**: Include all imports and setup code
2. **Runnable Code**: Examples should work when copy-pasted
3. **Realistic Scenarios**: Use domain models developers recognize (User, Product, Order)
4. **Error Handling**: Show proper error handling patterns
5. **Type Annotations**: Include type annotations for clarity when beneficial
6. **Progressive Enhancement**: Build complexity incrementally
7. **Before/After**: Show traditional approach vs Effect approach
8. **Helper Functions**: Create utilities that encapsulate patterns

## Structure Guidelines

1. **Logical Flow**: Problem → Solution → Details → Practice
2. **Consistent Sections**: Use the same structure for all modules
3. **Clear Headers**: Descriptive section titles
4. **Table of Contents**: Always include for navigation
5. **Cross-References**: Link to related Effect modules
6. **Code-to-Text Ratio**: Aim for 60-70% code examples
7. **Practical Focus**: Every section should provide actionable knowledge

## Key Patterns from VariantSchema Guide

### Problem/Solution Pattern
```typescript
// Traditional approach - multiple separate schemas
const UserInsert = Schema.Struct({...})
const UserSelect = Schema.Struct({...})
// Shows problems: duplication, drift, complexity

// The Effect Solution
const User = Struct({...})
// Shows benefits: DRY, maintainable, type-safe
```

### Helper Function Pattern
```typescript
// Helper for common use case
const MyHelper = <T extends Schema.Schema.All>(schema: T) => {
  // Encapsulates a pattern
  // Makes it reusable
  // Reduces boilerplate
}
```

### Real-World Integration Pattern
- Database models (CRUD operations)
- API serialization (different views)
- Form validation (state management)
- Third-party libraries (adapters)

### Progressive Enhancement Pattern
Show how requirements evolve and how the module handles increasing complexity.

## Example Modules for Documentation

Modules that would benefit from this documentation style:
- **Schema**: Type-safe schema validation and transformation
- **DateTime**: Date and time handling with timezone support
- **Platform**: HTTP clients, servers, and platform-specific functionality
- **Stream**: Async streaming data processing
- **STM**: Software Transactional Memory
- **Metrics**: Application metrics and monitoring
- **Config**: Configuration management
- **CLI**: Command-line interface building

## Effect-TS Syntax Guidelines

### Core Principle: Hybrid Pattern
Use **Effect.gen + yield*** for business logic, **.pipe** for post-processing and composition.

### Use Effect.gen + yield* for:

#### 1. Primary Business Logic & Sequential Operations
```typescript
// ✅ PREFERRED: Complex business logic with multiple steps
export function createUser(user: User.New): Effect.Effect<User.Entity, CreateUserError, Database> {
  return Effect.gen(function* () {
    const db = yield* Database
    const validatedUser = yield* User.validate(user)
    const hashedPassword = yield* hashPassword(validatedUser.password)
    const userData = { ...validatedUser, password: hashedPassword }
    const result = yield* db.users.create(userData)
    return yield* User.decode(result)
  }).pipe(
    Effect.catchTag('ValidationError', (cause) => new UserCreationError({ cause, entity: 'User' })),
    Effect.withSpan('user.create', { attributes: { 'user.email': user.email } })
  )
}
```

#### 2. Conditional Logic & Branching
```typescript
// ✅ PREFERRED: Control flow with conditionals
export const processPayment = (method: PaymentMethod) => {
  return Effect.gen(function* () {
    if (method === 'credit_card') {
      return yield* processCreditCard()
    }
    if (method === 'paypal') {
      return yield* processPayPal()
    }
    return yield* processBankTransfer()
  }).pipe(
    Effect.withSpan('payment.process'),
    Effect.annotateSpans({ 'payment.method': method })
  )
}
```

#### 3. Resource Management & Service Dependencies
```typescript
// ✅ PREFERRED: Service dependencies and resource management
const makeEmailService = Effect.gen(function* () {
  const config = yield* Config
  const logger = yield* Logger
  const transport = yield* createTransport(config.smtp)
  const send = makeSend(transport, logger)
  const sendBulk = makeSendBulk(transport, logger)
  const validate = makeValidate(config.validation)
  return { send, sendBulk, validate, transport } as const satisfies EmailService
})
```

### Use .pipe for:

#### 1. Post-Processing & Composition
```typescript
// ✅ PREFERRED: Transforming results and adding metadata
export function getUser(id: string, includeProfile: boolean) {
  return Effect.gen(function* () {
    // Core business logic here...
  }).pipe(
    Effect.catchTag('NotFoundError', (cause) => new UserNotFoundError({ cause, userId: id })),
    Effect.withSpan('user.get', {
      attributes: { 'user.id': id, 'user.include_profile': includeProfile }
    })
  )
}
```

#### 2. Layer Composition & Configuration
```typescript
// ✅ PREFERRED: Building dependency layers
export const applicationLayer = Layer.empty.pipe(
  Layer.provide(HttpClient.layer),
  Layer.provide(Logger.consoleLayer),
  Layer.provideMerge(Database.postgresLayer),
  Layer.provideMerge(Cache.redisLayer)
)
```

#### 3. Single-Step Transformations
```typescript
// ✅ PREFERRED: Simple transformations
const activeUserNames = yield* getActiveUsers().pipe(
  Effect.map(users => users.map(user => user.name))
)
```

#### 4. Error Handling & Middleware
```typescript
// ✅ PREFERRED: Error transformation and middleware
return AuthService.validateToken(token).pipe(
  Effect.provideService(JwtService, jwtService),
  Effect.mapError(() => new UnauthorizedError('Invalid token')),
  Effect.withSpan('auth.validate_token')
)
```

### Anti-Patterns to Avoid

#### ❌ Don't Use .pipe for Complex Sequential Logic
```typescript
// ❌ BAD: Hard to read and maintain
export const processOrder = (order: Order) => Effect.succeed(order)
  .pipe(
    Effect.flatMap((order) => validateOrder(order)),
    Effect.flatMap((validated) => calculateTotals(validated)),
    Effect.flatMap((calculated) => applyDiscounts(calculated)),
    Effect.flatMap((discounted) => processPayment(discounted)),
    Effect.flatMap((paid) => updateInventory(paid)),
    Effect.flatMap((updated) => sendConfirmation(updated))
  )

// ✅ GOOD: Clear and maintainable
export const processOrder = (order: Order) => Effect.gen(function* () {
  const validated = yield* validateOrder(order)
  const calculated = yield* calculateTotals(validated)
  const discounted = yield* applyDiscounts(calculated)
  const paid = yield* processPayment(discounted)
  const updated = yield* updateInventory(paid)
  return yield* sendConfirmation(updated)
})
```

#### ❌ Don't Use Effect.gen for Simple Transformations
```typescript
// ❌ BAD: Unnecessary complexity
const formatUserNames = Effect.gen(function* () {
  const users = yield* getUsers()
  return users.map(user => user.name.toUpperCase())
})

// ✅ GOOD: Simple and direct
const formatUserNames = getUsers().pipe(
  Effect.map(users => users.map(user => user.name.toUpperCase()))
)
```

### Decision Matrix

| Context             | Syntax Choice       | Reasoning                                       |
|---------------------|---------------------|-------------------------------------------------|
| Business Logic      | Effect.gen + yield* | Multiple steps, dependencies, conditional logic |
| Error Handling      | .pipe               | Composable error transformations                |
| Layer Building      | .pipe               | Dependency composition patterns                 |
| Single Transforms   | .pipe               | Simple, functional transformations              |
| Resource Management | Effect.gen + yield* | Managing dependencies and cleanup               |
| Post-Processing     | .pipe               | Adding tracing, caching, metadata               |
| Control Flow        | Effect.gen + yield* | Conditionals, loops, complex branching          |
| Function Factories  | Effect.gen + yield* | Building functions with dependencies            |

### Best Practices for Documentation

1. **Start with Business Logic**: Use Effect.gen + yield* for the core domain logic
2. **Compose with .pipe**: Add cross-cutting concerns (tracing, error handling) using .pipe
3. **Layer Composition**: Always use .pipe for building dependency layers
4. **Error Boundaries**: Use .pipe for error transformation and recovery
5. **Keep Consistency**: Follow the hybrid pattern established in examples
6. **Readability First**: Choose the syntax that makes the intent clearest

## Documentation Checklist

Before publishing, ensure your guide includes:
- [ ] Clear problem statement in the introduction
- [ ] At least 3 real-world examples
- [ ] Helper functions for common patterns
- [ ] Integration with at least one popular library
- [ ] Testing strategies
- [ ] Type inference demonstrations
- [ ] Error handling examples
- [ ] Progressive complexity in examples
- [ ] Consistent code style and naming
- [ ] Runnable code examples
- [ ] **Effect.gen + yield* used for business logic**
- [ ] **.pipe used for post-processing and composition**
- [ ] **Hybrid pattern consistently applied throughout**

# Code Refactoring Standards

## Pipe Function Refactoring Rules

When refactoring Effect guides, apply these pipe transformation rules consistently:

### Rule 1: Pipe Value-First to Function-First
Convert `pipe(value, fn1, fn2)` patterns to `pipe(fn1(value), fn2)`:

```typescript
// BEFORE (value-first):
const result = pipe(
  numbers,
  Array.filter(n => n % 2 === 0),
  Array.map(n => n * n),
  Array.take(3)
)

// AFTER (function-first):
const result = pipe(
  Array.filter(numbers, n => n % 2 === 0),
  Array.map(n => n * n),
  Array.take(3)
)
```

### Rule 2: Import Alias Standardization
Update single-letter module aliases to descriptive names:

```typescript
// BEFORE:
import { Array as A, Function as F } from "effect"

// AFTER:
import { Array as Arr, Function as Fn } from "effect"
```

**Standard Aliases:**
- `Array as A` → `Array as Arr`
- `Function as F` → `Function as Fn`

### Rule 3: Update All References
When changing import aliases, update ALL usage throughout the file:
- `A.filter` → `Arr.filter`
- `F.pipe` → `Fn.pipe`

### Rule 4: Maintain Functionality
- Ensure pipe transformations preserve exact same behavior
- Keep all TypeScript types intact
- Preserve all comments and explanations
- Maintain code block syntax highlighting
