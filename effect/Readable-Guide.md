# Readable: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem Readable Solves

Modern applications need reactive state management that can track changes, notify observers, and maintain consistency across complex data flows. Traditional approaches using callbacks, event emitters, or mutable state lead to fragile, hard-to-debug systems:

```typescript
// Traditional approach - callback-based reactive state
interface StateManager<T> {
  value: T
  listeners: Array<(value: T) => void>
}

class UserProfileManager {
  private state: StateManager<UserProfile> = {
    value: { name: '', email: '', preferences: {} },
    listeners: []
  }
  
  // Manual listener management - error-prone
  subscribe(callback: (profile: UserProfile) => void): () => void {
    this.state.listeners.push(callback)
    return () => {
      const index = this.state.listeners.indexOf(callback)
      if (index > -1) this.state.listeners.splice(index, 1)
    }
  }
  
  // Manual notification - must remember to call
  updateProfile(update: Partial<UserProfile>): void {
    this.state.value = { ...this.state.value, ...update }
    // Forgot to notify listeners? Silent bugs!
    this.state.listeners.forEach(listener => {
      try {
        listener(this.state.value)
      } catch (error) {
        // Error in one listener breaks others
        console.error('Listener error:', error)
      }
    })
  }
  
  // Race conditions in async updates
  async loadProfile(userId: string): Promise<void> {
    const profile = await fetchUserProfile(userId)
    // What if another update happened while fetching?
    this.state.value = profile
    this.state.listeners.forEach(listener => listener(profile))
  }
}

// Usage leads to memory leaks and inconsistent behavior
const manager = new UserProfileManager()
const unsubscribe1 = manager.subscribe(profile => updateUI(profile))
const unsubscribe2 = manager.subscribe(profile => logActivity(profile))
// Forgot to call unsubscribe1() - memory leak!
```

This approach leads to:
- **Memory Leaks** - Forgotten unsubscribe calls leave dangling references
- **Race Conditions** - Concurrent updates create inconsistent state
- **Error Propagation** - Exceptions in one observer break the entire chain
- **Manual Plumbing** - Boilerplate listener management and notification code

### The Readable Solution

Readable provides type-safe, composable reactive containers that abstract away the complexity of change tracking and notification:

```typescript
import { Effect, Readable, Ref } from "effect"

// Type-safe reactive state container
const makeUserProfileStore = Effect.gen(function* () {
  const profileRef = yield* Ref.make<UserProfile>({ 
    name: '', 
    email: '', 
    preferences: {} 
  })
  
  // Readable interface provides clean access pattern
  const profileReadable: Readable<UserProfile> = profileRef
  
  const updateProfile = (update: Partial<UserProfile>) =>
    Ref.update(profileRef, current => ({ ...current, ...update }))
  
  const loadProfile = (userId: string) => Effect.gen(function* () {
    const profile = yield* fetchUserProfileEffect(userId)
    yield* Ref.set(profileRef, profile)
  })
  
  return { 
    profile: profileReadable, 
    updateProfile, 
    loadProfile 
  } as const
})

// Clean, composable reactive patterns
const program = Effect.gen(function* () {
  const store = yield* makeUserProfileStore
  
  // Type-safe value access - no manual listener management
  const currentProfile = yield* store.profile
  console.log('Current profile:', currentProfile)
  
  // Composable transformations
  const profileName = store.profile.pipe(
    Readable.map(profile => profile.name)
  )
  
  const displayName = yield* profileName
  console.log('Display name:', displayName)
  
  // Safe concurrent updates
  yield* store.updateProfile({ name: 'Alice' })
  yield* store.loadProfile('user-123')
})
```

### Key Concepts

**Reactive Container**: Readable wraps an Effect that produces a value, enabling reactive access patterns without manual subscription management.

**Composable Transformations**: Transform readable values using familiar operators like map and mapEffect while maintaining type safety.

**Integration Pattern**: Readable serves as a bridge between mutable state (Ref) and reactive consumption patterns, commonly used with UI frameworks.

## Basic Usage Patterns

### Creating Readable Containers

```typescript
import { Effect, Readable, Ref } from "effect"

// From Effect - wrap any Effect as a Readable
const currentTime: Readable<Date> = Readable.make(
  Effect.sync(() => new Date())
)

// From Ref - most common pattern for reactive state
const program = Effect.gen(function* () {
  const counterRef = yield* Ref.make(0)
  
  // Ref implements Readable interface
  const counterReadable: Readable<number> = counterRef
  
  // Access current value
  const value = yield* counterReadable
  console.log('Counter value:', value) // 0
  
  // Update and read again
  yield* Ref.update(counterRef, n => n + 1)
  const newValue = yield* counterReadable
  console.log('Updated value:', newValue) // 1
})
```

### Transforming Readable Values

```typescript
import { Effect, Readable, Ref } from "effect"

const makeTransformExample = Effect.gen(function* () {
  const userRef = yield* Ref.make({ 
    name: 'Alice', 
    age: 30, 
    active: true 
  })
  
  // Transform with pure functions
  const userName = userRef.pipe(
    Readable.map(user => user.name.toUpperCase())
  )
  
  const userAge = userRef.pipe(
    Readable.map(user => user.age)
  )
  
  // Transform with Effects
  const userStatus = userRef.pipe(
    Readable.mapEffect(user => 
      user.active 
        ? Effect.succeed('ACTIVE')
        : Effect.succeed('INACTIVE')
    )
  )
  
  // Access transformed values
  const name = yield* userName
  const age = yield* userAge  
  const status = yield* userStatus
  
  console.log(`${name} (${age}) - ${status}`)
  // "ALICE (30) - ACTIVE"
  
  return { userRef, userName, userAge, userStatus }
})
```

### Readable with Error Handling

```typescript
import { Effect, Readable, Ref, Data } from "effect"

class ValidationError extends Data.TaggedError("ValidationError")<{
  field: string
  message: string
}> {}

const makeValidatedInput = Effect.gen(function* () {
  const inputRef = yield* Ref.make('')
  
  // Readable with validation and error handling
  const validatedInput = inputRef.pipe(
    Readable.mapEffect(input => {
      if (input.length < 3) {
        return Effect.fail(new ValidationError({ 
          field: 'input', 
          message: 'Must be at least 3 characters' 
        }))
      }
      return Effect.succeed(input.trim())
    })
  )
  
  const setInput = (value: string) => Ref.set(inputRef, value)
  
  return { setInput, validatedInput }
})

// Usage with error handling
const program = Effect.gen(function* () {
  const { setInput, validatedInput } = yield* makeValidatedInput
  
  // Test with invalid input
  yield* setInput('ab')
  const result1 = yield* Effect.either(validatedInput)
  console.log(result1) // Either.left(ValidationError)
  
  // Test with valid input
  yield* setInput('hello')
  const result2 = yield* Effect.either(validatedInput)
  console.log(result2) // Either.right('hello')
})
```

## Real-World Examples

### Example 1: Shopping Cart State Management

Real-world e-commerce applications need reactive shopping cart state that can be consumed by multiple UI components and updated from various sources.

```typescript
import { Effect, Readable, Ref, Data, Array as Arr } from "effect"

interface CartItem {
  readonly id: string
  readonly name: string
  readonly price: number
  readonly quantity: number
}

interface ShoppingCart {
  readonly items: readonly CartItem[]
  readonly total: number
  readonly itemCount: number
}

class CartError extends Data.TaggedError("CartError")<{
  reason: string
}> {}

const makeShoppingCartStore = Effect.gen(function* () {
  const itemsRef = yield* Ref.make<readonly CartItem[]>([])
  
  // Core reactive cart state
  const cart: Readable<ShoppingCart> = itemsRef.pipe(
    Readable.map(items => ({
      items,
      total: items.reduce((sum, item) => sum + (item.price * item.quantity), 0),
      itemCount: items.reduce((sum, item) => sum + item.quantity, 0)
    }))
  )
  
  // Derived reactive computations
  const cartTotal = cart.pipe(
    Readable.map(cart => cart.total)
  )
  
  const cartItemCount = cart.pipe(
    Readable.map(cart => cart.itemCount)
  )
  
  const cartIsEmpty = cart.pipe(
    Readable.map(cart => cart.items.length === 0)
  )
  
  // Cart operations
  const addItem = (item: Omit<CartItem, 'quantity'>, quantity = 1) =>
    Ref.update(itemsRef, items => {
      const existingIndex = items.findIndex(i => i.id === item.id)
      if (existingIndex >= 0) {
        return items.map((i, index) =>
          index === existingIndex
            ? { ...i, quantity: i.quantity + quantity }
            : i
        )
      }
      return [...items, { ...item, quantity }]
    })
  
  const removeItem = (itemId: string) =>
    Ref.update(itemsRef, items =>
      items.filter(item => item.id !== itemId)
    )
  
  const updateQuantity = (itemId: string, quantity: number) =>
    quantity <= 0
      ? removeItem(itemId)
      : Ref.update(itemsRef, items =>
          items.map(item =>
            item.id === itemId ? { ...item, quantity } : item
          )
        )
  
  const clearCart = () => Ref.set(itemsRef, [])
  
  return {
    cart,
    cartTotal,
    cartItemCount,
    cartIsEmpty,
    addItem,
    removeItem,
    updateQuantity,
    clearCart
  } as const
})

// Usage in e-commerce application
const ecommerceApp = Effect.gen(function* () {
  const cartStore = yield* makeShoppingCartStore
  
  // Add some items
  yield* cartStore.addItem({ 
    id: 'book-1', 
    name: 'Effect Guide', 
    price: 29.99 
  }, 2)
  
  yield* cartStore.addItem({ 
    id: 'mug-1', 
    name: 'Coffee Mug', 
    price: 15.99 
  })
  
  // Reactive UI updates - cart components automatically reflect changes
  const currentCart = yield* cartStore.cart
  const total = yield* cartStore.cartTotal
  const itemCount = yield* cartStore.cartItemCount
  
  console.log('Cart Summary:')
  console.log(`Items: ${itemCount}`)
  console.log(`Total: $${total.toFixed(2)}`)
  console.log('Items:', currentCart.items)
  
  // Update quantity triggers reactive updates
  yield* cartStore.updateQuantity('book-1', 1)
  
  const updatedTotal = yield* cartStore.cartTotal
  console.log(`Updated total: $${updatedTotal.toFixed(2)}`)
})
```

### Example 2: Real-Time Dashboard with Multiple Data Sources

Complex dashboards need to aggregate data from multiple sources and provide reactive updates to UI components.

```typescript
import { Effect, Readable, Ref, Schedule, Duration } from "effect"

interface SystemMetrics {
  readonly cpu: number
  readonly memory: number
  readonly diskUsage: number
  readonly timestamp: Date
}

interface UserActivity {
  readonly activeUsers: number
  readonly newSignups: number
  readonly totalSessions: number
  readonly timestamp: Date
}

interface DashboardState {
  readonly metrics: SystemMetrics
  readonly activity: UserActivity
  readonly isHealthy: boolean
  readonly lastUpdated: Date
}

const makeDashboardStore = Effect.gen(function* () {
  // Individual data source refs
  const metricsRef = yield* Ref.make<SystemMetrics>({
    cpu: 0,
    memory: 0,
    diskUsage: 0,
    timestamp: new Date()
  })
  
  const activityRef = yield* Ref.make<UserActivity>({
    activeUsers: 0,
    newSignups: 0,
    totalSessions: 0,
    timestamp: new Date()
  })
  
  // Combine multiple readable sources into comprehensive dashboard state
  const dashboardState: Readable<DashboardState> = Readable.make(
    Effect.gen(function* () {
      const metrics = yield* metricsRef
      const activity = yield* activityRef
      
      const isHealthy = metrics.cpu < 80 && 
                       metrics.memory < 85 && 
                       metrics.diskUsage < 90
      
      return {
        metrics,
        activity,
        isHealthy,
        lastUpdated: new Date()
      }
    })
  )
  
  // Derived reactive computations for specific UI components
  const systemHealth = dashboardState.pipe(
    Readable.map(state => ({
      status: state.isHealthy ? 'healthy' : 'warning',
      cpu: state.metrics.cpu,
      memory: state.metrics.memory,
      disk: state.metrics.diskUsage
    }))
  )
  
  const userStats = dashboardState.pipe(
    Readable.map(state => ({
      activeUsers: state.activity.activeUsers,
      newSignups: state.activity.newSignups,
      totalSessions: state.activity.totalSessions,
      signupRate: state.activity.newSignups / Math.max(state.activity.activeUsers, 1)
    }))
  )
  
  // Data fetching functions
  const updateMetrics = Effect.gen(function* () {
    const metrics = yield* fetchSystemMetricsEffect()
    yield* Ref.set(metricsRef, {
      ...metrics,
      timestamp: new Date()
    })
  })
  
  const updateActivity = Effect.gen(function* () {
    const activity = yield* fetchUserActivityEffect()
    yield* Ref.set(activityRef, {
      ...activity,
      timestamp: new Date()
    })
  })
  
  // Real-time update schedules
  const startMetricsUpdates = updateMetrics.pipe(
    Effect.repeat(Schedule.fixed(Duration.seconds(30))),
    Effect.fork
  )
  
  const startActivityUpdates = updateActivity.pipe(
    Effect.repeat(Schedule.fixed(Duration.minutes(1))),
    Effect.fork
  )
  
  return {
    dashboardState,
    systemHealth,
    userStats,
    updateMetrics,
    updateActivity,
    startMetricsUpdates,
    startActivityUpdates
  } as const
})

// Mock data fetching functions
const fetchSystemMetricsEffect = (): Effect.Effect<SystemMetrics> =>
  Effect.succeed({
    cpu: Math.random() * 100,
    memory: Math.random() * 100,
    diskUsage: Math.random() * 100,
    timestamp: new Date()
  })

const fetchUserActivityEffect = (): Effect.Effect<UserActivity> =>
  Effect.succeed({
    activeUsers: Math.floor(Math.random() * 1000) + 100,
    newSignups: Math.floor(Math.random() * 50),
    totalSessions: Math.floor(Math.random() * 5000) + 500,
    timestamp: new Date()
  })

// Dashboard application with reactive updates
const dashboardApp = Effect.gen(function* () {
  const dashboard = yield* makeDashboardStore
  
  // Start background data updates
  yield* dashboard.startMetricsUpdates
  yield* dashboard.startActivityUpdates
  
  // Simulate UI components reading reactive state
  const logDashboardState = Effect.gen(function* () {
    const state = yield* dashboard.dashboardState
    const health = yield* dashboard.systemHealth
    const stats = yield* dashboard.userStats
    
    console.log('=== Dashboard Update ===')
    console.log(`System Health: ${health.status}`)
    console.log(`CPU: ${health.cpu.toFixed(1)}%`)
    console.log(`Memory: ${health.memory.toFixed(1)}%`)
    console.log(`Active Users: ${stats.activeUsers}`)
    console.log(`New Signups: ${stats.newSignups}`)
    console.log(`Signup Rate: ${(stats.signupRate * 100).toFixed(2)}%`)
    console.log(`Last Updated: ${state.lastUpdated.toISOString()}`)
  })
  
  // Log initial state
  yield* logDashboardState
  
  // Manually trigger updates to see reactive changes
  yield* Effect.sleep(Duration.seconds(2))
  yield* dashboard.updateMetrics
  yield* dashboard.updateActivity
  yield* logDashboardState
  
  return dashboard
})
```

### Example 3: Form State Management with Validation

Complex forms need reactive validation, field dependencies, and clean state management that scales with form complexity.

```typescript
import { Effect, Readable, Ref, Data, Option, Either } from "effect"

interface UserRegistrationForm {
  readonly email: string
  readonly password: string
  readonly confirmPassword: string
  readonly firstName: string
  readonly lastName: string
  readonly agreeToTerms: boolean
}

interface FieldValidation {
  readonly isValid: boolean
  readonly errors: readonly string[]
}

interface FormValidation {
  readonly email: FieldValidation
  readonly password: FieldValidation
  readonly confirmPassword: FieldValidation
  readonly firstName: FieldValidation
  readonly lastName: FieldValidation
  readonly agreeToTerms: FieldValidation
  readonly isFormValid: boolean
}

class ValidationError extends Data.TaggedError("ValidationError")<{
  field: string
  errors: readonly string[]
}> {}

const makeRegistrationFormStore = Effect.gen(function* () {
  // Form field refs
  const formRef = yield* Ref.make<UserRegistrationForm>({
    email: '',
    password: '',
    confirmPassword: '',
    firstName: '',
    lastName: '',
    agreeToTerms: false
  })
  
  // Validation functions
  const validateEmail = (email: string): FieldValidation => {
    const errors: string[] = []
    if (!email) errors.push('Email is required')
    else if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
      errors.push('Invalid email format')
    }
    return { isValid: errors.length === 0, errors }
  }
  
  const validatePassword = (password: string): FieldValidation => {
    const errors: string[] = []
    if (!password) errors.push('Password is required')
    else if (password.length < 8) errors.push('Password must be at least 8 characters')
    else if (!/(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/.test(password)) {
      errors.push('Password must contain uppercase, lowercase, and number')
    }
    return { isValid: errors.length === 0, errors }
  }
  
  const validateConfirmPassword = (password: string, confirmPassword: string): FieldValidation => {
    const errors: string[] = []
    if (!confirmPassword) errors.push('Please confirm your password')
    else if (password !== confirmPassword) errors.push('Passwords do not match')
    return { isValid: errors.length === 0, errors }
  }
  
  const validateName = (name: string, fieldName: string): FieldValidation => {
    const errors: string[] = []
    if (!name) errors.push(`${fieldName} is required`)
    else if (name.length < 2) errors.push(`${fieldName} must be at least 2 characters`)
    return { isValid: errors.length === 0, errors }
  }
  
  const validateTerms = (agreeToTerms: boolean): FieldValidation => {
    const errors: string[] = []
    if (!agreeToTerms) errors.push('You must agree to the terms and conditions')
    return { isValid: errors.length === 0, errors }
  }
  
  // Reactive form validation
  const formValidation: Readable<FormValidation> = formRef.pipe(
    Readable.map(form => {
      const email = validateEmail(form.email)
      const password = validatePassword(form.password)
      const confirmPassword = validateConfirmPassword(form.password, form.confirmPassword)
      const firstName = validateName(form.firstName, 'First name')
      const lastName = validateName(form.lastName, 'Last name')
      const agreeToTerms = validateTerms(form.agreeToTerms)
      
      const isFormValid = [
        email, password, confirmPassword, firstName, lastName, agreeToTerms
      ].every(field => field.isValid)
      
      return {
        email,
        password,
        confirmPassword,
        firstName,
        lastName,
        agreeToTerms,
        isFormValid
      }
    })
  )
  
  // Derived readable for specific UI needs
  const canSubmit = formValidation.pipe(
    Readable.map(validation => validation.isFormValid)
  )
  
  const fieldErrors = formValidation.pipe(
    Readable.map(validation => ({
      email: validation.email.errors,
      password: validation.password.errors,
      confirmPassword: validation.confirmPassword.errors,
      firstName: validation.firstName.errors,
      lastName: validation.lastName.errors,
      agreeToTerms: validation.agreeToTerms.errors
    }))
  )
  
  // Form update functions
  const updateField = <K extends keyof UserRegistrationForm>(
    field: K,
    value: UserRegistrationForm[K]
  ) =>
    Ref.update(formRef, form => ({ ...form, [field]: value }))
  
  const updateForm = (updates: Partial<UserRegistrationForm>) =>
    Ref.update(formRef, form => ({ ...form, ...updates }))
  
  const resetForm = () =>
    Ref.set(formRef, {
      email: '',
      password: '',
      confirmPassword: '',
      firstName: '',
      lastName: '',
      agreeToTerms: false
    })
  
  // Form submission with validation
  const submitForm = Effect.gen(function* () {
    const form = yield* formRef
    const validation = yield* formValidation
    
    if (!validation.isFormValid) {
      const allErrors = Object.entries(validation)
        .filter(([key, value]) => key !== 'isFormValid' && !value.isValid)
        .flatMap(([field, fieldValidation]) =>
          fieldValidation.errors.map(error => `${field}: ${error}`)
        )
      
      return yield* Effect.fail(new ValidationError({
        field: 'form',
        errors: allErrors
      }))
    }
    
    // Simulate form submission
    console.log('Submitting form:', form)
    yield* Effect.sleep(1000) // Simulate API call
    yield* resetForm()
    
    return { success: true, message: 'Registration successful!' }
  })
  
  return {
    form: formRef as Readable<UserRegistrationForm>,
    formValidation,
    canSubmit,
    fieldErrors,
    updateField,
    updateForm,
    resetForm,
    submitForm
  } as const
})

// Form usage example
const formExample = Effect.gen(function* () {
  const formStore = yield* makeRegistrationFormStore
  
  // Simulate user input
  yield* formStore.updateField('email', 'alice@example.com')
  yield* formStore.updateField('firstName', 'Alice')
  yield* formStore.updateField('lastName', 'Johnson')
  yield* formStore.updateField('password', 'Password123')
  yield* formStore.updateField('confirmPassword', 'Password123')
  
  // Check validation state
  let validation = yield* formStore.formValidation
  let canSubmit = yield* formStore.canSubmit
  let errors = yield* formStore.fieldErrors
  
  console.log('Form validation state:')
  console.log('Can submit:', canSubmit)
  console.log('Errors:', errors)
  
  // Accept terms to make form valid
  yield* formStore.updateField('agreeToTerms', true)
  
  validation = yield* formStore.formValidation
  canSubmit = yield* formStore.canSubmit
  
  console.log('After accepting terms:')
  console.log('Can submit:', canSubmit)
  console.log('Form is valid:', validation.isFormValid)
  
  // Submit form
  const result = yield* Effect.either(formStore.submitForm)
  console.log('Submission result:', result)
})
```

## Advanced Features Deep Dive

### Unwrapping Nested Readable Effects

The `unwrap` function handles complex scenarios where you have a Readable inside an Effect, enabling dynamic Readable creation patterns.

```typescript
import { Effect, Readable, Ref, Layer, Context } from "effect"

// Service that provides different readable configurations
interface ConfigService {
  readonly getRefreshInterval: Effect.Effect<number>
  readonly getDataSource: Effect.Effect<string>
}

const ConfigService = Context.GenericTag<ConfigService>('ConfigService')

// Dynamic Readable creation based on service configuration
const makeDynamicDataSource = Effect.gen(function* () {
  const config = yield* ConfigService
  
  // Effect that creates a Readable based on configuration
  const readableEffect: Effect.Effect<Readable<string>> = Effect.gen(function* () {
    const dataSource = yield* config.getDataSource
    const interval = yield* config.getRefreshInterval
    
    const dataRef = yield* Ref.make(`data-from-${dataSource}`)
    
    // Background refresh logic
    const refresh = Effect.gen(function* () {
      const newData = yield* Effect.succeed(`updated-data-${Date.now()}`)
      yield* Ref.set(dataRef, newData)
    })
    
    yield* refresh.pipe(
      Effect.repeat(Schedule.fixed(Duration.millis(interval))),
      Effect.fork
    )
    
    return dataRef as Readable<string>
  })
  
  // Unwrap the Effect<Readable<string>> to get Readable<string>
  const dynamicReadable: Readable<string> = Readable.unwrap(readableEffect)
  
  return dynamicReadable
})

// Configuration implementations
const devConfigService: ConfigService = {
  getRefreshInterval: Effect.succeed(1000),
  getDataSource: Effect.succeed('dev-api')
}

const prodConfigService: ConfigService = {
  getRefreshInterval: Effect.succeed(5000),
  getDataSource: Effect.succeed('prod-api')
}

// Usage with different configurations
const unwrapExample = Effect.gen(function* () {
  const dynamicReadable = yield* makeDynamicDataSource
  
  // Access the unwrapped readable
  const data1 = yield* dynamicReadable
  console.log('Initial data:', data1)
  
  yield* Effect.sleep(Duration.seconds(2))
  
  const data2 = yield* dynamicReadable
  console.log('Updated data:', data2)
}).pipe(
  Effect.provide(Layer.succeed(ConfigService, devConfigService))
)
```

### Complex Readable Composition Patterns

Combining multiple Readable sources with complex transformation logic for advanced reactive patterns.

```typescript
import { Effect, Readable, Ref, Array as Arr, Option, Either } from "effect"

interface ApiResponse<T> {
  readonly data: T
  readonly timestamp: Date
  readonly status: 'success' | 'error' | 'loading'
  readonly error?: string
}

interface CombinedData {
  readonly users: readonly User[]
  readonly products: readonly Product[]
  readonly orders: readonly Order[]
  readonly summary: {
    readonly totalUsers: number
    readonly totalProducts: number
    readonly totalOrders: number
    readonly lastUpdated: Date
  }
}

const makeComposedDataStore = Effect.gen(function* () {
  // Individual data source refs
  const usersRef = yield* Ref.make<ApiResponse<readonly User[]>>({
    data: [],
    timestamp: new Date(),
    status: 'loading'
  })
  
  const productsRef = yield* Ref.make<ApiResponse<readonly Product[]>>({
    data: [],
    timestamp: new Date(),
    status: 'loading'
  })
  
  const ordersRef = yield* Ref.make<ApiResponse<readonly Order[]>>({
    data: [],
    timestamp: new Date(),
    status: 'loading'
  })
  
  // Complex composition of multiple readable sources
  const combinedData: Readable<Either.Either<CombinedData, string>> = Readable.make(
    Effect.gen(function* () {
      const usersResponse = yield* usersRef
      const productsResponse = yield* productsRef
      const ordersResponse = yield* ordersRef
      
      // Check if any source has an error
      const errorSources = [usersResponse, productsResponse, ordersResponse]
        .filter(response => response.status === 'error')
      
      if (errorSources.length > 0) {
        const errors = errorSources.map(r => r.error || 'Unknown error').join(', ')
        return Either.left(`Data fetch errors: ${errors}`)
      }
      
      // Check if all sources are loaded
      const isLoading = [usersResponse, productsResponse, ordersResponse]
        .some(response => response.status === 'loading')
      
      if (isLoading) {
        return Either.left('Loading data...')
      }
      
      // Combine successful data
      const combined: CombinedData = {
        users: usersResponse.data,
        products: productsResponse.data,
        orders: ordersResponse.data,
        summary: {
          totalUsers: usersResponse.data.length,
          totalProducts: productsResponse.data.length,
          totalOrders: ordersResponse.data.length,
          lastUpdated: new Date(Math.max(
            usersResponse.timestamp.getTime(),
            productsResponse.timestamp.getTime(),
            ordersResponse.timestamp.getTime()
          ))
        }
      }
      
      return Either.right(combined)
    })
  )
  
  // Derived readable for successful data only
  const successfulData: Readable<Option.Option<CombinedData>> = combinedData.pipe(
    Readable.map(either => 
      Either.match(either, {
        onLeft: () => Option.none(),
        onRight: (data) => Option.some(data)
      })
    )
  )
  
  // Derived readable for error state
  const errorState: Readable<Option.Option<string>> = combinedData.pipe(
    Readable.map(either =>
      Either.match(either, {
        onLeft: (error) => Option.some(error),
        onRight: () => Option.none()
      })
    )
  )
  
  // Update functions for each data source
  const updateUsers = (users: readonly User[]) =>
    Ref.set(usersRef, {
      data: users,
      timestamp: new Date(),
      status: 'success'
    })
  
  const updateProducts = (products: readonly Product[]) =>
    Ref.set(productsRef, {
      data: products,
      timestamp: new Date(),
      status: 'success'
    })
  
  const updateOrders = (orders: readonly Order[]) =>
    Ref.set(ordersRef, {
      data: orders,
      timestamp: new Date(),
      status: 'success'
    })
  
  const setError = (source: 'users' | 'products' | 'orders', error: string) => {
    const setRefError = (ref: Ref.Ref<ApiResponse<any>>) =>
      Ref.update(ref, current => ({
        ...current,
        status: 'error' as const,
        error,
        timestamp: new Date()
      }))
    
    switch (source) {
      case 'users': return setRefError(usersRef)
      case 'products': return setRefError(productsRef)
      case 'orders': return setRefError(ordersRef)
    }
  }
  
  return {
    combinedData,
    successfulData,
    errorState,
    updateUsers,
    updateProducts,
    updateOrders,
    setError
  } as const
})

// Example types for demonstration
interface User { id: string; name: string; email: string }
interface Product { id: string; name: string; price: number }
interface Order { id: string; userId: string; productIds: string[]; total: number }

// Advanced composition usage
const compositionExample = Effect.gen(function* () {
  const store = yield* makeComposedDataStore
  
  // Initially in loading state
  const initial = yield* store.combinedData
  console.log('Initial state:', initial) // Either.left('Loading data...')
  
  // Update data sources one by one
  yield* store.updateUsers([
    { id: '1', name: 'Alice', email: 'alice@example.com' },
    { id: '2', name: 'Bob', email: 'bob@example.com' }
  ])
  
  const afterUsers = yield* store.combinedData
  console.log('After users update:', afterUsers) // Still loading products and orders
  
  yield* store.updateProducts([
    { id: 'p1', name: 'Laptop', price: 999 },
    { id: 'p2', name: 'Mouse', price: 25 }
  ])
  
  yield* store.updateOrders([
    { id: 'o1', userId: '1', productIds: ['p1'], total: 999 }
  ])
  
  // Now all data is loaded
  const complete = yield* store.combinedData
  console.log('Complete data:', complete) // Either.right(CombinedData)
  
  // Access successful data
  const successData = yield* store.successfulData
  Option.match(successData, {
    onNone: () => console.log('No data available'),
    onSome: (data) => console.log('Summary:', data.summary)
  })
  
  // Simulate an error
  yield* store.setError('products', 'API timeout')
  
  const withError = yield* store.errorState
  Option.match(withError, {
    onNone: () => console.log('No errors'),
    onSome: (error) => console.log('Error:', error)
  })
})
```

## Practical Patterns & Best Practices

### Readable Service Pattern

Create reusable services that provide reactive state management capabilities across your application.

```typescript
import { Effect, Readable, Ref, Context, Layer } from "effect"

// Service interface for reactive application state
interface AppStateService {
  readonly currentUser: Readable<Option.Option<User>>
  readonly notifications: Readable<readonly Notification[]>
  readonly theme: Readable<'light' | 'dark'>
  readonly isOnline: Readable<boolean>
  
  // State update operations
  readonly setUser: (user: Option.Option<User>) => Effect.Effect<void>
  readonly addNotification: (notification: Notification) => Effect.Effect<void>
  readonly clearNotifications: () => Effect.Effect<void>
  readonly toggleTheme: () => Effect.Effect<void>
  readonly setOnlineStatus: (online: boolean) => Effect.Effect<void>
}

const AppStateService = Context.GenericTag<AppStateService>('AppStateService')

// Service implementation
const makeAppStateService = Effect.gen(function* () {
  // Internal state refs
  const currentUserRef = yield* Ref.make<Option.Option<User>>(Option.none())
  const notificationsRef = yield* Ref.make<readonly Notification[]>([])
  const themeRef = yield* Ref.make<'light' | 'dark'>('light')
  const isOnlineRef = yield* Ref.make(true)
  
  // Expose refs as readonly Readables
  const currentUser: Readable<Option.Option<User>> = currentUserRef
  const notifications: Readable<readonly Notification[]> = notificationsRef
  const theme: Readable<'light' | 'dark'> = themeRef
  const isOnline: Readable<boolean> = isOnlineRef
  
  // State update operations
  const setUser = (user: Option.Option<User>) => Ref.set(currentUserRef, user)
  
  const addNotification = (notification: Notification) =>
    Ref.update(notificationsRef, notifications => 
      [...notifications, notification]
    )
  
  const clearNotifications = () => Ref.set(notificationsRef, [])
  
  const toggleTheme = () =>
    Ref.update(themeRef, current => current === 'light' ? 'dark' : 'light')
  
  const setOnlineStatus = (online: boolean) => Ref.set(isOnlineRef, online)
  
  return {
    currentUser,
    notifications,
    theme,
    isOnline,
    setUser,
    addNotification,
    clearNotifications,
    toggleTheme,
    setOnlineStatus
  } satisfies AppStateService
})

// Service layer
const AppStateServiceLive = Layer.effect(AppStateService, makeAppStateService)

// Usage pattern - consuming services through readable interface
const makeUserDashboard = Effect.gen(function* () {
  const appState = yield* AppStateService
  
  // Derived readable computations
  const userDisplayName = appState.currentUser.pipe(
    Readable.map(userOption =>
      Option.match(userOption, {
        onNone: () => 'Guest',
        onSome: (user) => user.name
      })
    )
  )
  
  const unreadNotificationCount = appState.notifications.pipe(
    Readable.map(notifications => 
      notifications.filter(n => !n.read).length
    )
  )
  
  const dashboardTheme = appState.theme.pipe(
    Readable.map(theme => ({
      theme,
      backgroundColor: theme === 'dark' ? '#1a1a1a' : '#ffffff',
      textColor: theme === 'dark' ? '#ffffff' : '#000000'
    }))
  )
  
  return {
    userDisplayName,
    unreadNotificationCount,
    dashboardTheme,
    appState // Expose service for updates
  }
})

// Example usage
const appExample = Effect.gen(function* () {
  const dashboard = yield* makeUserDashboard
  
  // Read initial state
  const displayName = yield* dashboard.userDisplayName
  const unreadCount = yield* dashboard.unreadNotificationCount
  const theme = yield* dashboard.dashboardTheme
  
  console.log(`Welcome, ${displayName}!`)
  console.log(`You have ${unreadCount} unread notifications`)
  console.log(`Theme:`, theme)
  
  // Update state through service
  yield* dashboard.appState.setUser(Option.some({
    id: '1',
    name: 'Alice Johnson',
    email: 'alice@example.com'
  }))
  
  yield* dashboard.appState.addNotification({
    id: 'n1',
    message: 'Welcome to the app!',
    type: 'info',
    read: false,
    timestamp: new Date()
  })
  
  // State automatically updates through reactive system
  const updatedName = yield* dashboard.userDisplayName
  const updatedCount = yield* dashboard.unreadNotificationCount
  
  console.log(`Updated: Welcome, ${updatedName}!`)
  console.log(`You have ${updatedCount} unread notifications`)
  
}).pipe(Effect.provide(AppStateServiceLive))

interface User {
  readonly id: string
  readonly name: string
  readonly email: string
}

interface Notification {
  readonly id: string
  readonly message: string
  readonly type: 'info' | 'warning' | 'error'
  readonly read: boolean
  readonly timestamp: Date
}
```

### Readable Caching Pattern

Implement intelligent caching for expensive computations with automatic cache invalidation.

```typescript
import { Effect, Readable, Ref, Duration, Schedule, Hash } from "effect"

interface CacheEntry<T> {
  readonly value: T
  readonly timestamp: Date
  readonly expiresAt: Date
}

interface CacheConfig {
  readonly ttl: Duration.Duration
  readonly maxSize: number
}

const makeCachedReadable = <A, E, R>(
  computation: Effect.Effect<A, E, R>,
  config: CacheConfig,
  keyGenerator: () => string = () => 'default'
) => Effect.gen(function* () {
  const cacheRef = yield* Ref.make<Map<string, CacheEntry<A>>>(new Map())
  
  const getCachedValue = Effect.gen(function* () {
    const cache = yield* Ref.get(cacheRef)
    const key = keyGenerator()
    const entry = cache.get(key)
    const now = new Date()
    
    // Check if cached value exists and is not expired
    if (entry && entry.expiresAt > now) {
      return entry.value
    }
    
    // Compute new value
    const newValue = yield* computation
    const newEntry: CacheEntry<A> = {
      value: newValue,
      timestamp: now,
      expiresAt: new Date(now.getTime() + Duration.toMillis(config.ttl))
    }
    
    // Update cache with size limit
    yield* Ref.update(cacheRef, currentCache => {
      const newCache = new Map(currentCache)
      newCache.set(key, newEntry)
      
      // Implement LRU eviction if cache is too large
      if (newCache.size > config.maxSize) {
        const oldestKey = Array.from(newCache.entries())
          .sort(([, a], [, b]) => a.timestamp.getTime() - b.timestamp.getTime())[0][0]
        newCache.delete(oldestKey)
      }
      
      return newCache
    })
    
    return newValue
  })
  
  // Return readable that uses cached computation
  return Readable.make(getCachedValue)
})

// Expensive computation example
const expensiveDataFetch = (userId: string) =>
  Effect.gen(function* () {
    console.log(`Fetching expensive data for user ${userId}...`)
    yield* Effect.sleep(Duration.seconds(2)) // Simulate slow operation
    return {
      userId,
      complexData: `processed-data-${userId}-${Date.now()}`,
      computationTime: new Date()
    }
  })

// Cached readable usage
const cacheExample = Effect.gen(function* () {
  const userId = 'user-123'
  
  // Create cached readable for expensive operation
  const cachedData = yield* makeCachedReadable(
    expensiveDataFetch(userId),
    { ttl: Duration.minutes(5), maxSize: 100 },
    () => `user-data-${userId}`
  )
  
  console.log('First access (cache miss):')
  const data1 = yield* cachedData
  console.log('Data:', data1)
  
  console.log('Second access (cache hit):')
  const data2 = yield* cachedData  
  console.log('Data:', data2)
  
  // Same data returned from cache (timestamps match)
  console.log('Cache working:', data1.computationTime === data2.computationTime)
  
  // Wait for cache to expire
  yield* Effect.sleep(Duration.minutes(6))
  
  console.log('Third access (cache expired):')
  const data3 = yield* cachedData
  console.log('Data:', data3)
  console.log('New computation:', data3.computationTime !== data1.computationTime)
})
```

### Readable Observer Pattern

Implement the observer pattern using Readable for type-safe, composable reactive programming.

```typescript
import { Effect, Readable, Ref, Array as Arr, Schedule } from "effect"

// Observer interface for type-safe notifications
interface Observer<T> {
  readonly id: string
  readonly onNext: (value: T) => Effect.Effect<void>
  readonly onError?: (error: unknown) => Effect.Effect<void>
}

interface Observable<T> {
  readonly subscribe: (observer: Observer<T>) => Effect.Effect<() => Effect.Effect<void>>
  readonly current: Readable<T>
}

const makeObservable = <T>(initialValue: T) => Effect.gen(function* () {
  const valueRef = yield* Ref.make(initialValue)
  const observersRef = yield* Ref.make<readonly Observer<T>[]>([])
  
  // Notify all observers of value changes
  const notifyObservers = (value: T) => Effect.gen(function* () {
    const observers = yield* Ref.get(observersRef)
    
    // Run all observer notifications concurrently
    yield* Effect.all(
      observers.map(observer =>
        Effect.catchAll(
          observer.onNext(value),
          error => observer.onError?.(error) ?? Effect.unit
        )
      ),
      { concurrency: 'unbounded' }
    )
  })
  
  // Update value and notify observers
  const setValue = (newValue: T) => Effect.gen(function* () {
    yield* Ref.set(valueRef, newValue)
    yield* notifyObservers(newValue)
  })
  
  // Subscribe to changes
  const subscribe = (observer: Observer<T>) => Effect.gen(function* () {
    yield* Ref.update(observersRef, observers => [...observers, observer])
    
    // Send current value to new observer
    const currentValue = yield* Ref.get(valueRef)
    yield* observer.onNext(currentValue)
    
    // Return unsubscribe function
    const unsubscribe = () =>
      Ref.update(observersRef, observers =>
        observers.filter(obs => obs.id !== observer.id)
      )
    
    return unsubscribe
  })
  
  // Readable interface for current value
  const current: Readable<T> = valueRef
  
  return {
    subscribe,
    current,
    setValue
  } satisfies Observable<T> & { setValue: (value: T) => Effect.Effect<void> }
})

// Usage example with multiple observers
const observerExample = Effect.gen(function* () {
  const stockPrice = yield* makeObservable(100.0)
  
  // Create different types of observers
  const priceLogger: Observer<number> = {
    id: 'price-logger',
    onNext: (price) => Effect.sync(() => 
      console.log(`📊 Price update: $${price.toFixed(2)}`)
    )
  }
  
  const alertSystem: Observer<number> = {
    id: 'alert-system',
    onNext: (price) => Effect.gen(function* () {
      if (price > 150) {
        console.log(`🚨 HIGH PRICE ALERT: $${price.toFixed(2)}`)
      } else if (price < 50) {
        console.log(`⚠️ LOW PRICE ALERT: $${price.toFixed(2)}`)
      }
    })
  }
  
  const portfolioTracker: Observer<number> = {
    id: 'portfolio-tracker',
    onNext: (price) => Effect.sync(() => {
      const shares = 10
      const portfolioValue = price * shares
      console.log(`💰 Portfolio value: $${portfolioValue.toFixed(2)}`)
    }),
    onError: (error) => Effect.sync(() => 
      console.error('Portfolio tracking error:', error)
    )
  }
  
  // Subscribe observers
  const unsubscribeLogger = yield* stockPrice.subscribe(priceLogger)
  const unsubscribeAlerts = yield* stockPrice.subscribe(alertSystem)
  const unsubscribePortfolio = yield* stockPrice.subscribe(portfolioTracker)
  
  // Current value access through Readable
  const currentPrice = yield* stockPrice.current
  console.log(`Initial price: $${currentPrice.toFixed(2)}`)
  
  // Simulate price changes
  yield* stockPrice.setValue(120.5)
  yield* Effect.sleep(Duration.seconds(1))
  
  yield* stockPrice.setValue(45.0) // Triggers low price alert
  yield* Effect.sleep(Duration.seconds(1))
  
  yield* stockPrice.setValue(175.0) // Triggers high price alert
  yield* Effect.sleep(Duration.seconds(1))
  
  // Unsubscribe some observers
  yield* unsubscribeAlerts()
  console.log('Alert system unsubscribed')
  
  yield* stockPrice.setValue(110.0) // Only logger and portfolio will receive this
  
  // Clean up remaining subscriptions
  yield* unsubscribeLogger()
  yield* unsubscribePortfolio()
})
```

## Integration Examples

### Integration with React and UI Frameworks

Readable provides a clean bridge between Effect's reactive state management and UI frameworks like React.

```typescript
// react-effect-bridge.tsx
import React, { useEffect, useState } from 'react'
import { Effect, Readable, Runtime, FiberRef } from "effect"

// Custom hook to bridge Readable to React state
function useReadable<A, E, R>(
  readable: Readable<A, E, R>,
  runtime: Runtime.Runtime<R>,
  fallback: A
): [A, boolean, E | null] {
  const [value, setValue] = useState<A>(fallback)
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState<E | null>(null)
  
  useEffect(() => {
    // Subscribe to readable changes
    const fiber = Runtime.runFork(runtime)(
      Effect.gen(function* () {
        while (true) {
          try {
            const newValue = yield* readable
            setValue(newValue)
            setError(null)
            setLoading(false)
          } catch (err) {
            setError(err as E)
            setLoading(false)
          }
          
          // Poll for changes - in real implementation, you'd use 
          // a more sophisticated change detection mechanism
          yield* Effect.sleep(Duration.millis(100))
        }
      })
    )
    
    return () => {
      Runtime.runSync(runtime)(Fiber.interrupt(fiber))
    }
  }, [readable, runtime])
  
  return [value, loading, error]
}

// React component using Readable state
interface ShoppingCartProps {
  cartStore: {
    cart: Readable<ShoppingCart>
    cartTotal: Readable<number>
    cartItemCount: Readable<number>
    addItem: (item: CartItem) => Effect.Effect<void>
    removeItem: (itemId: string) => Effect.Effect<void>
  }
  runtime: Runtime.Runtime<never>
}

const ShoppingCartComponent: React.FC<ShoppingCartProps> = ({ 
  cartStore, 
  runtime 
}) => {
  // Use readable state in React
  const [cart, cartLoading] = useReadable(
    cartStore.cart, 
    runtime, 
    { items: [], total: 0, itemCount: 0 }
  )
  
  const [total] = useReadable(cartStore.cartTotal, runtime, 0)
  const [itemCount] = useReadable(cartStore.cartItemCount, runtime, 0)
  
  const handleAddItem = (item: Omit<CartItem, 'quantity'>) => {
    Runtime.runPromise(runtime)(
      cartStore.addItem({ ...item, quantity: 1 })
    )
  }
  
  const handleRemoveItem = (itemId: string) => {
    Runtime.runPromise(runtime)(
      cartStore.removeItem(itemId)
    )
  }
  
  if (cartLoading) {
    return <div>Loading cart...</div>
  }
  
  return (
    <div className="shopping-cart">
      <h2>Shopping Cart ({itemCount} items)</h2>
      
      <div className="cart-items">
        {cart.items.map(item => (
          <div key={item.id} className="cart-item">
            <span>{item.name}</span>
            <span>${item.price}</span>
            <span>Qty: {item.quantity}</span>
            <button onClick={() => handleRemoveItem(item.id)}>
              Remove
            </button>
          </div>
        ))}
      </div>
      
      <div className="cart-total">
        <strong>Total: ${total.toFixed(2)}</strong>
      </div>
      
      <button 
        onClick={() => handleAddItem({
          id: 'sample',
          name: 'Sample Item',
          price: 9.99
        })}
      >
        Add Sample Item
      </button>
    </div>
  )
}

// App setup with Effect runtime
const App: React.FC = () => {
  const [runtime] = useState(() => Runtime.defaultRuntime)
  const [cartStore, setCartStore] = useState<any>(null)
  
  useEffect(() => {
    // Initialize cart store
    Runtime.runPromise(runtime)(
      makeShoppingCartStore
    ).then(setCartStore)
  }, [runtime])
  
  if (!cartStore) {
    return <div>Initializing...</div>
  }
  
  return (
    <div className="app">
      <ShoppingCartComponent 
        cartStore={cartStore} 
        runtime={runtime} 
      />
    </div>
  )
}

export default App
```

### Integration with Streaming and Real-Time Data

Readable integrates seamlessly with Effect's Stream module for real-time data processing.

```typescript
import { Effect, Readable, Ref, Stream, Schedule, Duration } from "effect"

// WebSocket-like real-time data source
interface MarketData {
  readonly symbol: string
  readonly price: number
  readonly volume: number
  readonly timestamp: Date
}

const makeRealtimeMarketData = (symbol: string) => Effect.gen(function* () {
  const currentDataRef = yield* Ref.make<MarketData>({
    symbol,
    price: 100,
    volume: 0,
    timestamp: new Date()
  })
  
  // Simulate real-time market data stream
  const marketDataStream = Stream.repeatEffect(
    Effect.gen(function* () {
      // Simulate price fluctuation
      const currentData = yield* Ref.get(currentDataRef)
      const priceChange = (Math.random() - 0.5) * 2 // -1 to +1
      const newPrice = Math.max(0.01, currentData.price + priceChange)
      const newVolume = Math.floor(Math.random() * 1000)
      
      const newData: MarketData = {
        symbol,
        price: newPrice,
        volume: newVolume,
        timestamp: new Date()
      }
      
      yield* Ref.set(currentDataRef, newData)
      return newData
    })
  ).pipe(
    Stream.schedule(Schedule.fixed(Duration.millis(500)))
  )
  
  // Process stream and update reactive state
  const processMarketData = marketDataStream.pipe(
    Stream.tap(data => 
      Effect.sync(() => console.log(`📈 ${data.symbol}: $${data.price.toFixed(2)}`))
    ),
    Stream.runDrain,
    Effect.fork
  )
  
  yield* processMarketData
  
  // Readable interface for current market data
  const currentData: Readable<MarketData> = currentDataRef
  
  // Derived readable computations
  const currentPrice = currentData.pipe(
    Readable.map(data => data.price)
  )
  
  const priceHistory = yield* Ref.make<readonly number[]>([])
  
  // Track price history
  const updatePriceHistory = currentPrice.pipe(
    Readable.mapEffect(price =>
      Ref.update(priceHistory, history => {
        const newHistory = [...history, price]
        return newHistory.length > 20 
          ? newHistory.slice(-20) // Keep last 20 prices
          : newHistory
      })
    )
  )
  
  const priceMovingAverage = priceHistory.pipe(
    Readable.map(history => {
      if (history.length === 0) return 0
      return history.reduce((sum, price) => sum + price, 0) / history.length
    })
  )
  
  return {
    currentData,
    currentPrice,
    priceHistory: priceHistory as Readable<readonly number[]>,
    priceMovingAverage,
    updatePriceHistory
  }
})

// Trading dashboard with multiple symbols
const makeTradingDashboard = Effect.gen(function* () {
  const symbols = ['AAPL', 'GOOGL', 'TSLA', 'MSFT']
  
  // Create market data streams for multiple symbols
  const marketDataSources = yield* Effect.all(
    symbols.map(symbol => 
      makeRealtimeMarketData(symbol).pipe(
        Effect.map(source => [symbol, source] as const)
      )
    )
  )
  
  const marketDataMap = new Map(marketDataSources)
  
  // Aggregate readable for all market data
  const allMarketData: Readable<Map<string, MarketData>> = Readable.make(
    Effect.gen(function* () {
      const dataMap = new Map<string, MarketData>()
      
      for (const [symbol, source] of marketDataMap) {
        const data = yield* source.currentData
        dataMap.set(symbol, data)
      }
      
      return dataMap
    })
  )
  
  // Portfolio value calculation
  const portfolioRef = yield* Ref.make<Map<string, number>>(
    new Map([
      ['AAPL', 10],   // 10 shares
      ['GOOGL', 5],   // 5 shares  
      ['TSLA', 8],    // 8 shares
      ['MSFT', 15]    // 15 shares
    ])
  )
  
  const portfolioValue: Readable<number> = Readable.make(
    Effect.gen(function* () {
      const portfolio = yield* portfolioRef
      const marketData = yield* allMarketData
      
      let totalValue = 0
      for (const [symbol, shares] of portfolio) {
        const data = marketData.get(symbol)
        if (data) {
          totalValue += data.price * shares
        }
      }
      
      return totalValue
    })
  )
  
  return {
    allMarketData,
    portfolioValue,
    marketDataSources: marketDataMap,
    portfolio: portfolioRef
  }
})

// Real-time dashboard usage
const tradingDashboardExample = Effect.gen(function* () {
  const dashboard = yield* makeTradingDashboard
  
  // Monitor portfolio value changes
  const monitorPortfolio = Effect.gen(function* () {
    while (true) {
      const value = yield* dashboard.portfolioValue
      const marketData = yield* dashboard.allMarketData
      
      console.log('=== Portfolio Update ===')
      console.log(`Total Value: $${value.toFixed(2)}`)
      
      for (const [symbol, data] of marketData) {
        console.log(`${symbol}: $${data.price.toFixed(2)} (Vol: ${data.volume})`)
      }
      
      yield* Effect.sleep(Duration.seconds(2))
    }
  })
  
  // Run portfolio monitoring
  yield* monitorPortfolio.pipe(
    Effect.raceFirst(Effect.sleep(Duration.seconds(30))) // Run for 30 seconds
  )
})
```

### Integration with External APIs and Data Sources

Readable can elegantly handle complex data synchronization with external APIs and caching strategies.

```typescript
import { Effect, Readable, Ref, Schedule, Duration, Layer, Context } from "effect"

// External API service interface
interface ApiService {
  readonly fetchUser: (id: string) => Effect.Effect<User, ApiError>
  readonly fetchUserPosts: (userId: string) => Effect.Effect<readonly Post[], ApiError>
  readonly fetchPostComments: (postId: string) => Effect.Effect<readonly Comment[], ApiError>
}

const ApiService = Context.GenericTag<ApiService>('ApiService')

class ApiError extends Data.TaggedError("ApiError")<{
  status: number
  message: string
}> {}

// Synchronized data store with external API
const makeSynchronizedUserStore = (userId: string) => Effect.gen(function* () {
  const api = yield* ApiService
  
  // State refs
  const userRef = yield* Ref.make<Option.Option<User>>(Option.none())
  const postsRef = yield* Ref.make<readonly Post[]>([])
  const commentsRef = yield* Ref.make<Map<string, readonly Comment[]>>(new Map())
  const lastSyncRef = yield* Ref.make<Option.Option<Date>>(Option.none())
  const isSyncingRef = yield* Ref.make(false)
  
  // Sync operations
  const syncUser = Effect.gen(function* () {
    yield* Ref.set(isSyncingRef, true)
    try {
      const user = yield* api.fetchUser(userId)
      yield* Ref.set(userRef, Option.some(user))
    } finally {
      yield* Ref.set(isSyncingRef, false)
      yield* Ref.set(lastSyncRef, Option.some(new Date()))
    }
  })
  
  const syncPosts = Effect.gen(function* () {
    const posts = yield* api.fetchUserPosts(userId)
    yield* Ref.set(postsRef, posts)
    
    // Sync comments for each post
    yield* Effect.all(
      posts.map(post =>
        api.fetchPostComments(post.id).pipe(
          Effect.flatMap(comments =>
            Ref.update(commentsRef, map => 
              new Map(map).set(post.id, comments)
            )
          )
        )
      ),
      { concurrency: 5 }
    )
  })
  
  const fullSync = Effect.gen(function* () {
    yield* syncUser
    yield* syncPosts
  }).pipe(
    Effect.catchAll(error => 
      Effect.sync(() => console.error('Sync failed:', error))
    )
  )
  
  // Auto-sync schedule
  const startAutoSync = fullSync.pipe(
    Effect.repeat(Schedule.fixed(Duration.minutes(5))),
    Effect.fork
  )
  
  // Readable interfaces
  const user: Readable<Option.Option<User>> = userRef
  const posts: Readable<readonly Post[]> = postsRef
  const isSyncing: Readable<boolean> = isSyncingRef
  const lastSync: Readable<Option.Option<Date>> = lastSyncRef
  
  // Combined readable for complete user data
  const userData: Readable<Option.Option<UserData>> = Readable.make(
    Effect.gen(function* () {
      const userOption = yield* userRef
      const userPosts = yield* postsRef
      const allComments = yield* commentsRef
      
      return Option.map(userOption, user => ({
        user,
        posts: userPosts,
        comments: allComments,
        totalComments: Array.from(allComments.values())
          .reduce((sum, comments) => sum + comments.length, 0)
      }))
    })
  )
  
  // Initialize with first sync
  yield* fullSync
  yield* startAutoSync
  
  return {
    user,
    posts,
    userData,
    isSyncing,
    lastSync,
    syncUser,
    syncPosts,
    fullSync
  }
})

// Mock API implementation
const mockApiService: ApiService = {
  fetchUser: (id) => Effect.succeed({
    id,
    name: `User ${id}`,
    email: `user${id}@example.com`,
    avatar: `https://api.dicebear.com/7.x/avataaars/svg?seed=${id}`
  }),
  
  fetchUserPosts: (userId) => Effect.succeed([
    {
      id: `${userId}-post-1`,
      userId,
      title: 'My First Post',
      content: 'This is my first post content',
      createdAt: new Date()
    },
    {
      id: `${userId}-post-2`, 
      userId,
      title: 'Another Post',
      content: 'More interesting content here',
      createdAt: new Date()
    }
  ]),
  
  fetchPostComments: (postId) => Effect.succeed([
    {
      id: `${postId}-comment-1`,
      postId,
      author: 'Commenter 1',
      content: 'Great post!',
      createdAt: new Date()
    },
    {
      id: `${postId}-comment-2`,
      postId, 
      author: 'Commenter 2',
      content: 'Very insightful, thanks for sharing',
      createdAt: new Date()
    }
  ])
}

const ApiServiceLive = Layer.succeed(ApiService, mockApiService)

// Usage example
const apiIntegrationExample = Effect.gen(function* () {
  const userStore = yield* makeSynchronizedUserStore('user-123')
  
  // Monitor user data changes
  console.log('Initial sync started...')
  
  const userData = yield* userStore.userData
  Option.match(userData, {
    onNone: () => console.log('No user data available'),
    onSome: (data) => {
      console.log('User Data:')
      console.log(`Name: ${data.user.name}`)
      console.log(`Posts: ${data.posts.length}`)
      console.log(`Total Comments: ${data.totalComments}`)
    }
  })
  
  // Check sync status
  const isSyncing = yield* userStore.isSyncing
  const lastSync = yield* userStore.lastSync
  
  console.log(`Syncing: ${isSyncing}`)
  Option.match(lastSync, {
    onNone: () => console.log('Never synced'),
    onSome: (date) => console.log(`Last sync: ${date.toISOString()}`)
  })
  
  // Manual sync
  console.log('Triggering manual sync...')
  yield* userStore.fullSync
  
  // Updated data
  const updatedData = yield* userStore.userData
  Option.match(updatedData, {
    onNone: () => console.log('Still no data'),
    onSome: (data) => console.log(`Updated - Posts: ${data.posts.length}`)
  })
  
}).pipe(Effect.provide(ApiServiceLive))

// Supporting types
interface User {
  readonly id: string
  readonly name: string
  readonly email: string
  readonly avatar: string
}

interface Post {
  readonly id: string
  readonly userId: string
  readonly title: string
  readonly content: string
  readonly createdAt: Date
}

interface Comment {
  readonly id: string
  readonly postId: string
  readonly author: string
  readonly content: string
  readonly createdAt: Date
}

interface UserData {
  readonly user: User
  readonly posts: readonly Post[]
  readonly comments: Map<string, readonly Comment[]>
  readonly totalComments: number
}
```

## Conclusion

Readable provides type-safe reactive containers for Effect applications, bridging the gap between mutable state management and reactive consumption patterns. It enables clean separation of concerns while maintaining the full power of Effect's composable, type-safe programming model.

Key benefits:
- **Type Safety**: Full TypeScript support with precise error and requirement tracking
- **Composability**: Seamless integration with Effect's ecosystem and functional programming patterns  
- **Performance**: Lazy evaluation and efficient state access without unnecessary computations

Readable shines in scenarios requiring reactive state management, UI framework integration, real-time data processing, and complex application state coordination where traditional callback-based approaches become unwieldy.