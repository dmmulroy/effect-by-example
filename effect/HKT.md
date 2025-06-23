# HKT (Higher-Kinded Types): A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem HKT Solves

TypeScript's type system is powerful, but it struggles with higher-kinded types - types that take other types as parameters. Without HKT, creating generic abstractions over type constructors like `Array<T>`, `Promise<T>`, or `Effect<A, E, R>` becomes impossible or requires extensive code duplication.

```typescript
// Traditional approach - separate implementations for each container type
interface ArrayOperations {
  map<A, B>(arr: Array<A>, f: (a: A) => B): Array<B>
  filter<A>(arr: Array<A>, predicate: (a: A) => boolean): Array<A>
}

interface PromiseOperations {
  map<A, B>(promise: Promise<A>, f: (a: A) => B): Promise<B>
  // Can't implement filter for Promise in a meaningful way
}

interface OptionOperations {
  map<A, B>(option: Option<A>, f: (a: A) => B): Option<B>
  filter<A>(option: Option<A>, predicate: (a: A) => boolean): Option<A>
}
```

This approach leads to:
- **Code Duplication** - Same patterns implemented multiple times
- **Type Inconsistency** - No unified interface for similar operations
- **Limited Composability** - Can't write generic functions that work across different containers
- **Maintenance Burden** - Changes require updates in multiple places

### The HKT Solution

Effect's HKT module provides a sophisticated encoding that brings higher-kinded types to TypeScript, enabling generic programming over type constructors.

```typescript
import { HKT } from "effect"

// Define a type lambda that describes the structure
interface MyContainerTypeLambda extends HKT.TypeLambda {
  readonly type: MyContainer<this["Target"]>
}

// Now we can write generic operations that work with any container
interface Mappable<F extends HKT.TypeLambda> extends HKT.TypeClass<F> {
  readonly map: <A, B>(
    fa: HKT.Kind<F, unknown, never, never, A>,
    f: (a: A) => B
  ) => HKT.Kind<F, unknown, never, never, B>
}
```

### Key Concepts

**TypeLambda**: A type-level function that describes the structure of a generic type constructor with parameters for contravariant inputs, covariant outputs, and invariant targets.

**Kind**: A type alias that applies a TypeLambda to specific type parameters, creating the actual concrete type.

**TypeClass**: An interface that defines operations over types parameterized by a TypeLambda, enabling generic programming patterns.

## Basic Usage Patterns

### Pattern 1: Defining a TypeLambda

```typescript
import { HKT } from "effect"

// Define a simple container type
interface Container<A> {
  readonly value: A
}

// Create its TypeLambda representation
interface ContainerTypeLambda extends HKT.TypeLambda {
  readonly type: Container<this["Target"]>
}

// Kind<ContainerTypeLambda, never, never, never, string> === Container<string>
```

### Pattern 2: Creating TypeClass Instances

```typescript
import { HKT } from "effect"
import { dual } from "effect/Function"

// Define a Functor type class
interface Functor<F extends HKT.TypeLambda> extends HKT.TypeClass<F> {
  readonly map: {
    <A, B>(f: (a: A) => B): <R, O, E>(self: HKT.Kind<F, R, O, E, A>) => HKT.Kind<F, R, O, E, B>
    <R, O, E, A, B>(self: HKT.Kind<F, R, O, E, A>, f: (a: A) => B): HKT.Kind<F, R, O, E, B>
  }
}

// Implement Functor for Container
const ContainerFunctor: Functor<ContainerTypeLambda> = {
  map: dual(2, <A, B>(self: Container<A>, f: (a: A) => B): Container<B> => ({
    value: f(self.value)
  }))
}
```

### Pattern 3: Using Generic Operations

```typescript
// Generic function that works with any Functor
const mapTwoTimes = <F extends HKT.TypeLambda>(
  F: Functor<F>
) => <R, O, E, A>(
  fa: HKT.Kind<F, R, O, E, A>
): HKT.Kind<F, R, O, E, A> =>
  F.map(fa, (a) => a)

// Works with any type that implements Functor
const doubledContainer = ContainerFunctor.map({ value: 5 }, (x) => x * 2)
// { value: 10 }
```

## Real-World Examples

### Example 1: Database Repository Pattern

Creating a generic repository that works with different Effect-like types:

```typescript
import { Effect, HKT } from "effect"

interface User {
  readonly id: string
  readonly name: string
  readonly email: string
}

// Generic repository interface
interface Repository<F extends HKT.TypeLambda> extends HKT.TypeClass<F> {
  readonly findById: <R, O, E>(
    id: string
  ) => HKT.Kind<F, R, O, E | "NotFound", User>
  
  readonly save: <R, O, E>(
    user: User
  ) => HKT.Kind<F, R, O, E | "SaveError", User>
  
  readonly findAll: <R, O, E>() => HKT.Kind<F, R, O, E, ReadonlyArray<User>>
}

// Implementation for Effect
const EffectUserRepository: Repository<Effect.EffectTypeLambda> = {
  findById: (id: string) => Effect.gen(function* () {
    // Simulate database lookup
    if (id === "user-1") {
      return { id, name: "Alice", email: "alice@example.com" }
    }
    return yield* Effect.fail("NotFound" as const)
  }),
  
  save: (user: User) => Effect.gen(function* () {
    // Simulate database save
    yield* Effect.sleep("100 millis")
    return user
  }),
  
  findAll: () => Effect.succeed([
    { id: "user-1", name: "Alice", email: "alice@example.com" },
    { id: "user-2", name: "Bob", email: "bob@example.com" }
  ])
}

// Generic service layer that works with any Repository
const createUserService = <F extends HKT.TypeLambda>(
  repo: Repository<F>
) => ({
  getUserProfile: <R, O, E>(id: string) =>
    repo.findById(id),
    
  createUser: <R, O, E>(name: string, email: string) => {
    const user: User = { id: crypto.randomUUID(), name, email }
    return repo.save(user)
  },
  
  getAllUsers: <R, O, E>() =>
    repo.findAll()
})

// Usage with Effect
const userService = createUserService(EffectUserRepository)

const program = Effect.gen(function* () {
  const user = yield* userService.getUserProfile("user-1")
  console.log(`Found user: ${user.name}`)
  
  const newUser = yield* userService.createUser("Charlie", "charlie@example.com")
  console.log(`Created user: ${newUser.name}`)
  
  const allUsers = yield* userService.getAllUsers()
  console.log(`Total users: ${allUsers.length}`)
})
```

### Example 2: Validation Library

Building a composable validation system using HKT:

```typescript
import { HKT, Array as Arr, Either } from "effect"

// Validation result type
type ValidationResult<E, A> = Either.Either<ReadonlyArray<E>, A>

interface ValidationTypeLambda extends HKT.TypeLambda {
  readonly type: ValidationResult<this["Out1"], this["Target"]>
}

// Applicative instance for validation (collects all errors)
interface Applicative<F extends HKT.TypeLambda> extends HKT.TypeClass<F> {
  readonly of: <A>(a: A) => HKT.Kind<F, unknown, never, never, A>
  readonly ap: <R1, O1, E1, A, R2, O2, E2, B>(
    fab: HKT.Kind<F, R1, O1, E1, (a: A) => B>,
    fa: HKT.Kind<F, R2, O2, E2, A>
  ) => HKT.Kind<F, R1 & R2, O1 | O2, E1 | E2, B>
}

const ValidationApplicative: Applicative<ValidationTypeLambda> = {
  of: <A>(a: A): ValidationResult<never, A> => Either.right(a),
  
  ap: <E, A, B>(
    fab: ValidationResult<E, (a: A) => B>,
    fa: ValidationResult<E, A>
  ): ValidationResult<E, B> => {
    if (Either.isLeft(fab) && Either.isLeft(fa)) {
      return Either.left([...fab.left, ...fa.left])
    }
    if (Either.isLeft(fab)) return fab
    if (Either.isLeft(fa)) return fa
    return Either.right(fab.right(fa.right))
  }
}

// Generic validation functions
const validateEmail = (email: string): ValidationResult<string, string> =>
  email.includes("@") 
    ? Either.right(email)
    : Either.left(["Invalid email format"])

const validateAge = (age: number): ValidationResult<string, number> =>
  age >= 0 && age <= 120
    ? Either.right(age)
    : Either.left(["Age must be between 0 and 120"])

const validateName = (name: string): ValidationResult<string, string> =>
  name.length >= 2
    ? Either.right(name)
    : Either.left(["Name must be at least 2 characters"])

// Compose validations
interface PersonData {
  readonly name: string
  readonly email: string
  readonly age: number
}

const validatePerson = (data: {
  name: string
  email: string
  age: number
}): ValidationResult<string, PersonData> => {
  const nameValidation = validateName(data.name)
  const emailValidation = validateEmail(data.email)
  const ageValidation = validateAge(data.age)
  
  // Combine all validations, collecting errors
  return ValidationApplicative.ap(
    ValidationApplicative.ap(
      ValidationApplicative.of((name: string) => (email: string) => (age: number): PersonData => ({
        name, email, age
      })),
      nameValidation
    ),
    emailValidation
  )(ageValidation)
}

// Usage
const validPerson = validatePerson({
  name: "Alice",
  email: "alice@example.com", 
  age: 30
})
// Right({ name: "Alice", email: "alice@example.com", age: 30 })

const invalidPerson = validatePerson({
  name: "A",
  email: "invalid-email",
  age: -5
})
// Left(["Name must be at least 2 characters", "Invalid email format", "Age must be between 0 and 120"])
```

### Example 3: Generic State Management

Creating a state management system that works with different effect types:

```typescript
import { Effect, HKT, Ref } from "effect"

// State operations interface
interface StateOps<F extends HKT.TypeLambda> extends HKT.TypeClass<F> {
  readonly get: <R, O, E, S>() => HKT.Kind<F, R, O, E, S>
  readonly set: <R, O, E, S>(newState: S) => HKT.Kind<F, R, O, E, void>
  readonly modify: <R, O, E, S>(f: (s: S) => S) => HKT.Kind<F, R, O, E, S>
}

// Implementation using Effect + Ref
const createEffectStateOps = <S>(
  ref: Ref.Ref<S>
): StateOps<Effect.EffectTypeLambda> => ({
  get: () => Ref.get(ref),
  set: (newState: S) => Ref.set(ref, newState),
  modify: (f: (s: S) => S) => Ref.updateAndGet(ref, f)
})

// Generic counter using state operations
const createCounter = <F extends HKT.TypeLambda>(
  ops: StateOps<F>
) => ({
  increment: <R, O, E>() => ops.modify<R, O, E, number>((n) => n + 1),
  decrement: <R, O, E>() => ops.modify<R, O, E, number>((n) => n - 1),
  reset: <R, O, E>() => ops.set<R, O, E, number>(0),
  getValue: <R, O, E>() => ops.get<R, O, E, number>()
})

// Usage with Effect
const program = Effect.gen(function* () {
  const countRef = yield* Ref.make(0)
  const stateOps = createEffectStateOps(countRef)
  const counter = createCounter(stateOps)
  
  yield* counter.increment()
  yield* counter.increment()
  const value1 = yield* counter.getValue()
  console.log(`After increment: ${value1}`) // 2
  
  yield* counter.decrement()
  const value2 = yield* counter.getValue()
  console.log(`After decrement: ${value2}`) // 1
  
  yield* counter.reset()
  const value3 = yield* counter.getValue()
  console.log(`After reset: ${value3}`) // 0
})
```

## Advanced Features Deep Dive

### Feature 1: Variance Annotations

Effect's HKT system supports variance annotations to ensure type safety with contravariant and covariant positions.

#### Understanding Variance in TypeLambda

```typescript
import { HKT, Types } from "effect"

// The TypeLambda interface defines variance for each parameter
interface ComplexTypeLambda extends HKT.TypeLambda {
  readonly In: unknown      // Contravariant input
  readonly Out2: unknown    // Covariant output (often errors)
  readonly Out1: unknown    // Covariant output (often values)
  readonly Target: unknown  // Invariant target type
}

// Kind applies variance correctly
type ExampleKind = HKT.Kind<ComplexTypeLambda, string, number, boolean, User>
// Results in a type where:
// - In (string) is contravariant
// - Out2 (number) is covariant  
// - Out1 (boolean) is covariant
// - Target (User) is invariant
```

#### Real-World Variance Example

```typescript
// Function type with proper variance
interface FunctionTypeLambda extends HKT.TypeLambda {
  readonly type: (a: this["In"]) => this["Target"]
}

// Effect-like type with variance
interface ResultTypeLambda extends HKT.TypeLambda {
  readonly type: Result<this["Target"], this["Out1"], this["Out2"]>
}

interface Result<A, E, R> {
  readonly run: (context: R) => Either.Either<E, A>
}

// Contravariant in context (R), covariant in error (E) and value (A)
const ResultFunctor: Functor<ResultTypeLambda> = {
  map: dual(2, <A, B, E, R>(
    self: Result<A, E, R>,
    f: (a: A) => B
  ): Result<B, E, R> => ({
    run: (context) => self.run(context).pipe(Either.map(f))
  }))
}
```

### Feature 2: TypeClass Composition

Building complex abstractions by composing simpler type classes.

#### Composing Type Classes

```typescript
import { HKT } from "effect"

// Base type classes
interface Functor<F extends HKT.TypeLambda> extends HKT.TypeClass<F> {
  readonly map: <A, B>(
    fa: HKT.Kind<F, unknown, never, never, A>,
    f: (a: A) => B
  ) => HKT.Kind<F, unknown, never, never, B>
}

interface Apply<F extends HKT.TypeLambda> extends Functor<F> {
  readonly ap: <A, B>(
    fab: HKT.Kind<F, unknown, never, never, (a: A) => B>,
    fa: HKT.Kind<F, unknown, never, never, A>
  ) => HKT.Kind<F, unknown, never, never, B>
}

interface Applicative<F extends HKT.TypeLambda> extends Apply<F> {
  readonly of: <A>(a: A) => HKT.Kind<F, unknown, never, never, A>
}

interface Chain<F extends HKT.TypeLambda> extends Apply<F> {
  readonly chain: <A, B>(
    fa: HKT.Kind<F, unknown, never, never, A>,
    f: (a: A) => HKT.Kind<F, unknown, never, never, B>
  ) => HKT.Kind<F, unknown, never, never, B>
}

interface Monad<F extends HKT.TypeLambda> extends Applicative<F>, Chain<F> {}

// Automatic derivation of operations from type class composition
const liftA2 = <F extends HKT.TypeLambda>(
  F: Apply<F>
) => <A, B, C>(
  f: (a: A, b: B) => C,
  fa: HKT.Kind<F, unknown, never, never, A>,
  fb: HKT.Kind<F, unknown, never, never, B>
): HKT.Kind<F, unknown, never, never, C> =>
  F.ap(F.map(fa, (a) => (b: B) => f(a, b)), fb)

const sequence = <F extends HKT.TypeLambda>(
  F: Applicative<F>
) => <A>(
  fas: ReadonlyArray<HKT.Kind<F, unknown, never, never, A>>
): HKT.Kind<F, unknown, never, never, ReadonlyArray<A>> => {
  return fas.reduce(
    (acc, fa) => liftA2(F)((as, a) => [...as, a], acc, fa),
    F.of([] as ReadonlyArray<A>)
  )
}
```

### Feature 3: Advanced Type-Level Programming

Using HKT for sophisticated type-level computations.

#### Conditional Type Classes

```typescript
// Type class that only exists for certain types
interface Serializable<F extends HKT.TypeLambda> extends HKT.TypeClass<F> {
  readonly serialize: <A>(
    fa: HKT.Kind<F, unknown, never, never, A>
  ) => string
  readonly deserialize: <A>(
    json: string
  ) => Option.Option<HKT.Kind<F, unknown, never, never, A>>
}

// Higher-order type class - operates on type classes themselves
interface Transformable<F extends HKT.TypeLambda, G extends HKT.TypeLambda> {
  readonly transform: <A>(
    fa: HKT.Kind<F, unknown, never, never, A>
  ) => HKT.Kind<G, unknown, never, never, A>
}

// Natural transformation between functors
const optionToArray: Transformable<Option.OptionTypeLambda, Array.ReadonlyArrayTypeLambda> = {
  transform: <A>(option: Option.Option<A>): ReadonlyArray<A> =>
    Option.isNone(option) ? [] : [option.value]
}
```

## Practical Patterns & Best Practices

### Pattern 1: Generic Effect Composition

```typescript
import { Effect, HKT } from "effect"

// Helper for composing effects with different type lambdas
const composeEffects = <F extends HKT.TypeLambda, G extends HKT.TypeLambda>(
  F: Monad<F>,
  G: Monad<G>
) => ({
  // Sequence two effects and combine their results
  both: <A, B>(
    fa: HKT.Kind<F, unknown, never, never, A>,
    gb: HKT.Kind<G, unknown, never, never, B>
  ) => ({
    inF: F.map(fa, (a) => ({ a, gb })),
    inG: G.map(gb, (b) => ({ b, fa }))
  }),
  
  // Transform from F to G
  liftF: <A>(
    transform: Transformable<F, G>
  ) => (fa: HKT.Kind<F, unknown, never, never, A>) =>
    transform.transform(fa),
    
  // Generic error handling
  handleError: <E, A>(
    fa: HKT.Kind<F, unknown, never, E, A>,
    handler: (e: E) => HKT.Kind<F, unknown, never, never, A>
  ): HKT.Kind<F, unknown, never, never, A> => {
    // Implementation depends on specific F
    // This is a conceptual example
    return fa as any
  }
})

// Usage with concrete types
const effectUtils = composeEffects(
  Effect.Monad,
  { map: Option.map, ap: /* ... */, of: Option.some, chain: Option.flatMap }
)
```

### Pattern 2: Generic Resource Management

```typescript
// Resource management that works with any effect type
interface Resource<F extends HKT.TypeLambda, A> {
  readonly acquire: HKT.Kind<F, unknown, never, never, A>
  readonly release: (a: A) => HKT.Kind<F, unknown, never, never, void>
}

const bracket = <F extends HKT.TypeLambda>(
  F: Monad<F>
) => <A, B>(
  resource: Resource<F, A>,
  use: (a: A) => HKT.Kind<F, unknown, never, never, B>
): HKT.Kind<F, unknown, never, never, B> => {
  return F.chain(resource.acquire, (a) => 
    F.chain(use(a), (b) =>
      F.map(resource.release(a), () => b)
    )
  )
}

// File resource example
const fileResource = (path: string): Resource<Effect.EffectTypeLambda, FileHandle> => ({
  acquire: Effect.gen(function* () {
    console.log(`Opening file: ${path}`)
    return { path, handle: "file-handle" } as FileHandle
  }),
  release: (handle) => Effect.gen(function* () {
    console.log(`Closing file: ${handle.path}`)
  })
})

interface FileHandle {
  readonly path: string
  readonly handle: string
}

// Usage
const useFile = bracket(Effect.Monad)

const program = useFile(
  fileResource("data.txt"),
  (handle) => Effect.gen(function* () {
    console.log(`Processing file: ${handle.path}`)
    return "file content"
  })
)
```

### Pattern 3: Type-Safe Configuration

```typescript
// Configuration system using HKT for type safety
interface Config<F extends HKT.TypeLambda> extends HKT.TypeClass<F> {
  readonly get: <A>(
    key: string,
    decoder: (value: unknown) => HKT.Kind<F, unknown, never, string, A>
  ) => HKT.Kind<F, unknown, never, string, A>
  
  readonly getOptional: <A>(
    key: string,
    decoder: (value: unknown) => HKT.Kind<F, unknown, never, string, A>
  ) => HKT.Kind<F, unknown, never, never, Option.Option<A>>
}

// Environment variable configuration
const EnvConfig: Config<Effect.EffectTypeLambda> = {
  get: <A>(key: string, decoder: (value: unknown) => Effect.Effect<A, string>) =>
    Effect.gen(function* () {
      const value = process.env[key]
      if (value === undefined) {
        return yield* Effect.fail(`Missing environment variable: ${key}`)
      }
      return yield* decoder(value)
    }),
    
  getOptional: <A>(key: string, decoder: (value: unknown) => Effect.Effect<A, string>) =>
    Effect.gen(function* () {
      const value = process.env[key]
      if (value === undefined) {
        return Option.none()
      }
      const decoded = yield* decoder(value)
      return Option.some(decoded)
    })
}

// Decoders
const stringDecoder = (value: unknown): Effect.Effect<string, string> =>
  typeof value === "string" 
    ? Effect.succeed(value)
    : Effect.fail("Expected string")

const numberDecoder = (value: unknown): Effect.Effect<number, string> =>
  Effect.gen(function* () {
    const str = yield* stringDecoder(value)
    const num = Number(str)
    return isNaN(num) 
      ? yield* Effect.fail("Expected number")
      : num
  })

// Generic configuration loader
const loadAppConfig = <F extends HKT.TypeLambda>(
  config: Config<F>
) => Effect.gen(function* () {
  const port = yield* config.get("PORT", numberDecoder)
  const host = yield* config.get("HOST", stringDecoder)
  const debug = yield* config.getOptional("DEBUG", stringDecoder)
  
  return {
    port,
    host,
    debug: Option.getOrElse(debug, () => "false") === "true"
  }
})
```

## Integration Examples

### Integration with Schema Validation

Combining HKT with Effect's Schema for type-safe validation:

```typescript
import { Effect, HKT, Schema } from "effect"

// Generic schema validator using HKT
interface SchemaValidator<F extends HKT.TypeLambda> extends HKT.TypeClass<F> {
  readonly validate: <A, I>(
    schema: Schema.Schema<A, I>,
    input: I
  ) => HKT.Kind<F, unknown, never, Schema.ParseError, A>
}

const EffectSchemaValidator: SchemaValidator<Effect.EffectTypeLambda> = {
  validate: <A, I>(schema: Schema.Schema<A, I>, input: I) =>
    Schema.decodeUnknown(schema)(input)
}

// Generic form validation
const createFormValidator = <F extends HKT.TypeLambda>(
  validator: SchemaValidator<F>
) => {
  const UserSchema = Schema.Struct({
    name: Schema.String.pipe(Schema.minLength(2)),
    email: Schema.String.pipe(Schema.pattern(/^[^\s@]+@[^\s@]+\.[^\s@]+$/)),
    age: Schema.Number.pipe(Schema.between(0, 120))
  })
  
  return {
    validateUser: (input: unknown) => validator.validate(UserSchema, input),
    
    validateUserForm: (formData: FormData) => Effect.gen(function* () {
      const rawData = {
        name: formData.get("name"),
        email: formData.get("email"),
        age: Number(formData.get("age"))
      }
      return yield* validator.validate(UserSchema, rawData)
    })
  }
}

// Usage
const formValidator = createFormValidator(EffectSchemaValidator)

const processForm = (formData: FormData) => Effect.gen(function* () {
  const user = yield* formValidator.validateUserForm(formData)
  console.log("Valid user:", user)
  return user
}).pipe(
  Effect.catchTag("ParseError", (error) => 
    Effect.gen(function* () {
      console.error("Validation failed:", error.message)
      return yield* Effect.fail("Invalid form data")
    })
  )
)
```

### Integration with Stream Processing

Using HKT to create generic stream processing operations:

```typescript
import { Effect, HKT, Stream } from "effect"

// Generic stream operations using HKT
interface StreamOps<F extends HKT.TypeLambda> extends HKT.TypeClass<F> {
  readonly fromIterable: <A>(
    iterable: Iterable<A>
  ) => HKT.Kind<F, unknown, never, never, A>
  
  readonly map: <A, B>(
    stream: HKT.Kind<F, unknown, never, never, A>,
    f: (a: A) => B
  ) => HKT.Kind<F, unknown, never, never, B>
  
  readonly filter: <A>(
    stream: HKT.Kind<F, unknown, never, never, A>,
    predicate: (a: A) => boolean
  ) => HKT.Kind<F, unknown, never, never, A>
  
  readonly take: <A>(
    stream: HKT.Kind<F, unknown, never, never, A>,
    n: number
  ) => HKT.Kind<F, unknown, never, never, A>
  
  readonly toArray: <A>(
    stream: HKT.Kind<F, unknown, never, never, A>
  ) => Effect.Effect<ReadonlyArray<A>>
}

const EffectStreamOps: StreamOps<Stream.StreamTypeLambda> = {
  fromIterable: Stream.fromIterable,
  map: (stream, f) => stream.pipe(Stream.map(f)),
  filter: (stream, predicate) => stream.pipe(Stream.filter(predicate)),
  take: (stream, n) => stream.pipe(Stream.take(n)),
  toArray: Stream.runCollect
}

// Generic data processing pipeline
const createDataProcessor = <F extends HKT.TypeLambda>(
  ops: StreamOps<F>
) => {
  const processNumbers = (numbers: ReadonlyArray<number>) => Effect.gen(function* () {
    const stream = ops.fromIterable(numbers)
    const processed = ops.take(
      ops.filter(
        ops.map(stream, (n) => n * 2),
        (n) => n > 10
      ),
      5
    )
    return yield* ops.toArray(processed)
  })
  
  const processUsers = (users: ReadonlyArray<{ name: string; age: number }>) => 
    Effect.gen(function* () {
      const stream = ops.fromIterable(users)
      const processed = ops.map(
        ops.filter(stream, (user) => user.age >= 18),
        (user) => user.name.toUpperCase()
      )
      return yield* ops.toArray(processed)
    })
    
  return { processNumbers, processUsers }
}

// Usage
const processor = createDataProcessor(EffectStreamOps)

const program = Effect.gen(function* () {
  const numbers = yield* processor.processNumbers([1, 2, 3, 4, 5, 6, 7, 8, 9, 10])
  console.log("Processed numbers:", numbers) // [12, 14, 16, 18, 20]
  
  const users = yield* processor.processUsers([
    { name: "alice", age: 25 },
    { name: "bob", age: 17 },
    { name: "charlie", age: 30 }
  ])
  console.log("Processed users:", users) // ["ALICE", "CHARLIE"]
})
```

### Testing Strategies

Testing type-level constructs and generic abstractions:

```typescript
import { Effect, HKT, Option } from "effect"

// Test utilities for HKT-based code
const testTypeClass = <F extends HKT.TypeLambda>(
  name: string,
  F: Functor<F>,
  examples: ReadonlyArray<HKT.Kind<F, unknown, never, never, number>>
) => {
  describe(name, () => {
    test("functor laws - identity", () => {
      examples.forEach((fa) => {
        const identity = <A>(a: A): A => a
        const mapped = F.map(fa, identity)
        // In real tests, you'd compare the results
        expect(mapped).toEqual(fa)
      })
    })
    
    test("functor laws - composition", () => {
      const f = (x: number) => x * 2
      const g = (x: number) => x + 1
      
      examples.forEach((fa) => {
        const composed = F.map(fa, (x) => g(f(x)))
        const separate = F.map(F.map(fa, f), g)
        expect(composed).toEqual(separate)
      })
    })
  })
}

// Property-based testing with HKT
const testMonadLaws = <F extends HKT.TypeLambda>(
  name: string,
  M: Monad<F>,
  generator: () => HKT.Kind<F, unknown, never, never, number>
) => {
  describe(`${name} monad laws`, () => {
    test("left identity", () => {
      const a = 42
      const f = (x: number) => M.of(x * 2)
      
      const leftSide = M.chain(M.of(a), f)
      const rightSide = f(a)
      
      expect(leftSide).toEqual(rightSide)
    })
    
    test("right identity", () => {
      const ma = generator()
      
      const leftSide = M.chain(ma, M.of)
      const rightSide = ma
      
      expect(leftSide).toEqual(rightSide)
    })
    
    test("associativity", () => {
      const ma = generator()
      const f = (x: number) => M.of(x * 2)
      const g = (x: number) => M.of(x + 1)
      
      const leftSide = M.chain(M.chain(ma, f), g)
      const rightSide = M.chain(ma, (x) => M.chain(f(x), g))
      
      expect(leftSide).toEqual(rightSide)
    })
  })
}

// Mock implementations for testing
const TestOptionFunctor: Functor<Option.OptionTypeLambda> = {
  map: dual(2, <A, B>(self: Option.Option<A>, f: (a: A) => B): Option.Option<B> => 
    Option.map(self, f)
  )
}

// Run tests
testTypeClass("Option Functor", TestOptionFunctor, [
  Option.some(1),
  Option.some(42),
  Option.none()
])
```

## Conclusion

HKT provides the foundation for type-safe generic programming in Effect, enabling powerful abstractions that work across different type constructors.

Key benefits:
- **Type Safety**: Compile-time guarantees about generic operations
- **Code Reuse**: Write once, use with any compatible type
- **Composability**: Build complex abstractions from simple building blocks

Effect's HKT system brings the power of functional programming abstractions to TypeScript, allowing you to write more maintainable and reusable code while maintaining full type safety. Use HKT when you need to create generic libraries, implement mathematical abstractions like functors and monads, or build type-safe APIs that work across different effect types.