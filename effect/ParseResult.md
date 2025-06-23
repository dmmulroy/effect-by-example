# ParseResult: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem ParseResult Solves

When working with data validation and parsing in traditional approaches, you often face several challenges:

```typescript
// Traditional approach - problematic validation handling
function validateUser(data: unknown): User {
  if (!data || typeof data !== 'object') {
    throw new Error('Invalid input')
  }
  
  const obj = data as Record<string, unknown>
  
  if (typeof obj.name !== 'string') {
    throw new Error('Name must be a string')
  }
  
  if (typeof obj.email !== 'string' || !obj.email.includes('@')) {
    throw new Error('Invalid email')
  }
  
  if (typeof obj.age !== 'number' || obj.age < 0) {
    throw new Error('Age must be a positive number')
  }
  
  return { name: obj.name, email: obj.email, age: obj.age }
}
```

This approach leads to:
- **Poor Error Context** - Errors don't indicate exactly where validation failed
- **All-or-Nothing** - You get no information about partial successes
- **Complex Error Handling** - Managing different error types becomes unwieldy
- **No Composition** - Hard to build complex validators from simple ones

### The ParseResult Solution

ParseResult provides a structured, composable way to represent parsing outcomes with rich error information:

```typescript
import { ParseResult, Schema } from "effect"

const UserSchema = Schema.Struct({
  name: Schema.String,
  email: Schema.String.pipe(Schema.includes("@")),
  age: Schema.Number.pipe(Schema.positive())
})

// Rich error information with exact paths and context
const result = Schema.decodeUnknownEither(UserSchema)({ 
  name: 123, 
  email: "invalid", 
  age: -5 
})
// Returns detailed ParseIssue information
```

### Key Concepts

**ParseIssue**: Represents specific validation failures with context about what went wrong and where

**ParseError**: A tagged error containing a ParseIssue, integrable with Effect's error handling

**Formatters**: Transform ParseIssues into human-readable messages or structured data

## Basic Usage Patterns

### Pattern 1: Creating and Handling Parse Results

```typescript
import { ParseResult, Schema, Either } from "effect"

// Basic validation that returns Either<A, ParseIssue>
const validateNumber = (input: unknown) => {
  return Schema.decodeUnknownEither(Schema.Number)(input)
}

// Handle the result
const result = validateNumber("not a number")
if (Either.isLeft(result)) {
  // Access the ParseIssue
  const issue = result.left.issue
  console.log(ParseResult.TreeFormatter.formatIssueSync(issue))
  // Output: "Expected number, actual "not a number""
}
```

### Pattern 2: Working with ParseError

```typescript
import { ParseResult, Schema, Effect } from "effect"

const processUserData = (data: unknown) => {
  return Effect.gen(function* () {
    // This will throw ParseError on failure
    const user = yield* Schema.decodeUnknown(UserSchema)(data)
    return `Processing user: ${user.name}`
  }).pipe(
    Effect.catchTag("ParseError", (error) => {
      const message = ParseResult.TreeFormatter.formatErrorSync(error)
      return Effect.succeed(`Validation failed: ${message}`)
    })
  )
}
```

### Pattern 3: Custom Issue Creation

```typescript
import { ParseResult, Either } from "effect"

// Create custom parse issues
const validatePassword = (password: string): Either.Either<string, ParseResult.ParseIssue> => {
  if (password.length < 8) {
    return Either.left(new ParseResult.Type(
      Schema.String.ast, 
      password, 
      "Password must be at least 8 characters"
    ))
  }
  return Either.right(password)
}
```

## Real-World Examples

### Example 1: API Request Validation

Processing API requests with comprehensive error reporting:

```typescript
import { ParseResult, Schema, Effect, Either } from "effect"

// Define API request schema
const CreateUserRequest = Schema.Struct({
  name: Schema.String.pipe(Schema.minLength(2)),
  email: Schema.String.pipe(Schema.includes("@")),
  age: Schema.Number.pipe(Schema.between(18, 120)),
  preferences: Schema.optional(Schema.Struct({
    newsletter: Schema.Boolean,
    theme: Schema.Literal("light", "dark")
  }))
})

// API handler with detailed error reporting
const handleCreateUser = (requestBody: unknown) => {
  return Effect.gen(function* () {
    const request = yield* Schema.decodeUnknown(CreateUserRequest)(requestBody)
    
    // Simulate user creation
    const userId = yield* Effect.succeed(`user_${Date.now()}`)
    
    return {
      success: true,
      userId,
      user: request
    }
  }).pipe(
    Effect.catchTag("ParseError", (error) => {
      // Format validation errors for API response
      const issues = ParseResult.ArrayFormatter.formatErrorSync(error)
      
      return Effect.succeed({
        success: false,
        error: "Validation failed",
        details: issues.map(issue => ({
          path: issue.path.join('.'),
          message: issue.message,
          code: issue._tag
        }))
      })
    })
  )
}

// Usage
const invalidRequest = {
  name: "A", // too short
  email: "invalid-email", // no @
  age: 15, // too young
  preferences: {
    newsletter: "yes", // not boolean
    theme: "blue" // invalid literal
  }
}

const response = await Effect.runPromise(handleCreateUser(invalidRequest))
console.log(JSON.stringify(response, null, 2))
```

### Example 2: Configuration File Processing

Validating and processing configuration files with fallbacks:

```typescript
import { ParseResult, Schema, Effect, Option } from "effect"

const DatabaseConfig = Schema.Struct({
  host: Schema.String,
  port: Schema.Number.pipe(Schema.between(1, 65535)),
  database: Schema.String,
  ssl: Schema.optional(Schema.Boolean),
  retries: Schema.optional(Schema.Number.pipe(Schema.nonnegative()))
})

const ServerConfig = Schema.Struct({
  database: DatabaseConfig,
  server: Schema.Struct({
    port: Schema.Number.pipe(Schema.between(1000, 9999)),
    cors: Schema.optional(Schema.Boolean)
  }),
  logging: Schema.optional(Schema.Struct({
    level: Schema.Literal("debug", "info", "warn", "error"),
    file: Schema.optional(Schema.String)
  }))
})

const processConfig = (configData: unknown) => {
  return Effect.gen(function* () {
    // Try to parse the config
    const parseResult = Schema.decodeUnknownEither(ServerConfig)(configData)
    
    if (Either.isRight(parseResult)) {
      return {
        config: parseResult.right,
        warnings: []
      }
    }
    
    // Handle validation errors gracefully
    const error = parseResult.left
    const issues = ParseResult.ArrayFormatter.formatErrorSync(error)
    
    // Check if we can provide reasonable defaults for some issues
    const criticalIssues = issues.filter(issue => 
      issue.path.includes('host') || 
      issue.path.includes('database') ||
      issue.path[0] === 'server'
    )
    
    if (criticalIssues.length > 0) {
      // Critical config missing, can't continue
      return yield* Effect.fail({
        type: "ConfigurationError" as const,
        message: "Critical configuration errors found",
        issues: criticalIssues
      })
    }
    
    // Non-critical issues, apply defaults
    const defaultConfig = {
      database: {
        host: "localhost",
        port: 5432,
        database: "app",
        ssl: false,
        retries: 3
      },
      server: {
        port: 3000,
        cors: true
      },
      logging: {
        level: "info" as const,
        file: undefined
      }
    }
    
    return {
      config: defaultConfig,
      warnings: issues.map(issue => `${issue.path.join('.')}: ${issue.message}`)
    }
  })
}
```

### Example 3: Data Transformation Pipeline

Building a robust data processing pipeline with detailed error tracking:

```typescript
import { ParseResult, Schema, Effect, Array as Arr } from "effect"

const RawDataSchema = Schema.Struct({
  id: Schema.String,
  timestamp: Schema.String, // Will be transformed to Date
  value: Schema.Union(Schema.String, Schema.Number),
  metadata: Schema.optional(Schema.Record({ key: Schema.String, value: Schema.Unknown }))
})

const ProcessedDataSchema = Schema.Struct({
  id: Schema.String,
  timestamp: Schema.Date,
  numericValue: Schema.Number,
  metadata: Schema.Record({ key: Schema.String, value: Schema.String })
})

const transformRawData = Schema.transformOrFail(
  RawDataSchema,
  ProcessedDataSchema,
  {
    decode: (raw) => Effect.gen(function* () {
      // Transform timestamp string to Date
      const timestamp = new Date(raw.timestamp)
      if (isNaN(timestamp.getTime())) {
        return yield* Effect.fail(new ParseResult.Type(
          Schema.Date.ast,
          raw.timestamp,
          "Invalid timestamp format"
        ))
      }
      
      // Transform value to number
      const numericValue = typeof raw.value === 'number' 
        ? raw.value 
        : parseFloat(raw.value)
      
      if (isNaN(numericValue)) {
        return yield* Effect.fail(new ParseResult.Type(
          Schema.Number.ast,
          raw.value,
          "Cannot convert value to number"
        ))
      }
      
      // Ensure metadata values are strings
      const metadata = raw.metadata || {}
      const stringMetadata: Record<string, string> = {}
      
      for (const [key, value] of Object.entries(metadata)) {
        stringMetadata[key] = String(value)
      }
      
      return {
        id: raw.id,
        timestamp,
        numericValue,
        metadata: stringMetadata
      }
    }),
    encode: (processed) => Effect.succeed({
      id: processed.id,
      timestamp: processed.timestamp.toISOString(),
      value: processed.numericValue,
      metadata: processed.metadata
    })
  }
)

const processBatch = (rawData: readonly unknown[]) => {
  return Effect.gen(function* () {
    const results: Array<{
      index: number
      success: boolean
      data?: any
      error?: string
    }> = []
    
    for (let i = 0; i < rawData.length; i++) {
      const item = rawData[i]
      const result = Schema.decodeUnknownEither(transformRawData)(item)
      
      if (Either.isRight(result)) {
        results.push({
          index: i,
          success: true,
          data: result.right
        })
      } else {
        const errorMessage = ParseResult.TreeFormatter.formatErrorSync(result.left)
        results.push({
          index: i,
          success: false,
          error: errorMessage
        })
      }
    }
    
    const successful = results.filter(r => r.success)
    const failed = results.filter(r => !r.success)
    
    return {
      processed: successful.length,
      failed: failed.length,
      results,
      data: successful.map(r => r.data)
    }
  })
}
```

## Advanced Features Deep Dive

### Feature 1: Custom Formatters

Creating specialized formatters for different output needs:

```typescript
import { ParseResult, Effect, Option } from "effect"

// Custom formatter for JSON API responses
const APIFormatter: ParseResult.ParseResultFormatter<{
  code: string
  message: string
  path?: string[]
}> = {
  formatIssue: (issue) => {
    const getAPICode = (issue: ParseResult.ParseIssue): string => {
      switch (issue._tag) {
        case "Type": return "INVALID_TYPE"
        case "Missing": return "REQUIRED_FIELD"
        case "Unexpected": return "UNEXPECTED_FIELD"
        case "Forbidden": return "FORBIDDEN_OPERATION"
        case "Refinement": return "VALIDATION_FAILED"
        case "Transformation": return "TRANSFORMATION_ERROR"
        default: return "VALIDATION_ERROR"
      }
    }
    
    const formatMessage = (issue: ParseResult.ParseIssue): string => {
      // Simplified message formatting logic
      return ParseResult.TreeFormatter.formatIssueSync(issue)
    }
    
    const getPath = (issue: ParseResult.ParseIssue): string[] | undefined => {
      if (issue._tag === "Pointer") {
        const path = Array.isArray(issue.path) ? issue.path : [issue.path]
        return path.map(String)
      }
      return undefined
    }
    
    return Effect.succeed({
      code: getAPICode(issue),
      message: formatMessage(issue),
      path: getPath(issue)
    })
  },
  
  formatIssueSync: (issue) => {
    const result = Effect.runSync(APIFormatter.formatIssue(issue))
    return result
  },
  
  formatError: (error) => APIFormatter.formatIssue(error.issue),
  formatErrorSync: (error) => APIFormatter.formatIssueSync(error.issue)
}

// Usage with custom formatter
const formatValidationError = (error: ParseResult.ParseError) => {
  return APIFormatter.formatErrorSync(error)
}
```

### Feature 2: Advanced Error Composition

Building complex validation scenarios with detailed error tracking:

```typescript
import { ParseResult, Schema, Effect, Either, Array as Arr } from "effect"

// Helper to collect multiple validation errors
const validateMultiple = <T>(
  items: readonly T[],
  validator: (item: T, index: number) => Either.Either<any, ParseResult.ParseIssue>
): Either.Either<any[], ParseResult.ParseIssue> => {
  const results: Array<[number, Either.Either<any, ParseResult.ParseIssue>]> = 
    items.map((item, index) => [index, validator(item, index)])
  
  const failures = results.filter(([, result]) => Either.isLeft(result))
  
  if (failures.length > 0) {
    // Create composite error with all failures
    const issues = failures.map(([index, result]) => 
      new ParseResult.Pointer([index], items, Either.getLeft(result)!)
    )
    
    const compositeIssue = new ParseResult.Composite(
      Schema.Array(Schema.Unknown).ast,
      items,
      issues
    )
    
    return Either.left(compositeIssue)
  }
  
  const successes = results.map(([, result]) => Either.getRight(result)!)
  return Either.right(successes)
}

// Advanced validation with custom business rules
const validateBusinessRules = (data: {
  users: unknown[]
  permissions: unknown[]
}) => {
  return Effect.gen(function* () {
    // Validate individual users
    const userResults = validateMultiple(
      data.users,
      (user, index) => Schema.decodeUnknownEither(UserSchema)(user)
    )
    
    if (Either.isLeft(userResults)) {
      return yield* Effect.fail(ParseResult.parseError(userResults.left))
    }
    
    const users = userResults.right
    
    // Cross-validation: check email uniqueness
    const emails = users.map(u => u.email)
    const duplicateEmails = emails.filter((email, index) => 
      emails.indexOf(email) !== index
    )
    
    if (duplicateEmails.length > 0) {
      const duplicateIssue = new ParseResult.Refinement(
        Schema.Array(UserSchema).ast,
        data.users,
        "Predicate",
        new ParseResult.Type(
          Schema.Array(UserSchema).ast,
          data.users,
          `Duplicate emails found: ${duplicateEmails.join(', ')}`
        )
      )
      
      return yield* Effect.fail(ParseResult.parseError(duplicateIssue))
    }
    
    return { users, permissions: data.permissions }
  })
}
```

### Feature 3: Performance Optimization

Optimizing parse operations for high-throughput scenarios:

```typescript
import { ParseResult, Schema, Effect } from "effect"

// Pre-compiled parsers for better performance
const createOptimizedParser = <A, I>(schema: Schema.Schema<A, I>) => {
  // Pre-compile the parser functions
  const decodeEither = Schema.decodeUnknownEither(schema)
  const validateEither = Schema.validateEither(schema)
  
  return {
    // Fast path for known-good data
    validateKnown: (data: I) => validateEither(data),
    
    // Full parsing for unknown data
    parseUnknown: (data: unknown) => decodeEither(data),
    
    // Batch processing with early termination
    parseBatch: (items: readonly unknown[], { failFast = false } = {}) => {
      const results: Array<Either.Either<A, ParseResult.ParseError>> = []
      
      for (const item of items) {
        const result = Either.mapLeft(decodeEither(item), ParseResult.parseError)
        results.push(result)
        
        if (failFast && Either.isLeft(result)) {
          break
        }
      }
      
      return results
    }
  }
}

// Usage for high-performance scenarios
const userParser = createOptimizedParser(UserSchema)

const processHighVolumeData = (data: readonly unknown[]) => {
  return Effect.gen(function* () {
    const batchSize = 1000
    const results: A[] = []
    const errors: ParseResult.ParseError[] = []
    
    for (let i = 0; i < data.length; i += batchSize) {
      const batch = data.slice(i, i + batchSize)
      const batchResults = userParser.parseBatch(batch)
      
      for (const result of batchResults) {
        if (Either.isRight(result)) {
          results.push(result.right)
        } else {
          errors.push(result.left)
        }
      }
      
      // Yield control periodically for long operations
      if (i % (batchSize * 10) === 0) {
        yield* Effect.yieldNow()
      }
    }
    
    return { results, errors }
  })
}
```

## Practical Patterns & Best Practices

### Pattern 1: Error Recovery and Fallbacks

```typescript
import { ParseResult, Schema, Effect, Either, Option } from "effect"

// Helper for graceful degradation
const parseWithFallback = <A, I, B>(
  schema: Schema.Schema<A, I>,
  fallbackSchema: Schema.Schema<B, I>,
  input: unknown
): Either.Either<A | B, ParseResult.ParseIssue> => {
  const primaryResult = Schema.decodeUnknownEither(schema)(input)
  
  if (Either.isRight(primaryResult)) {
    return primaryResult
  }
  
  // Try fallback schema
  const fallbackResult = Schema.decodeUnknownEither(fallbackSchema)(input)
  return fallbackResult
}

// Validation with partial success tracking
const validateWithPartialSuccess = <T>(
  items: readonly unknown[],
  schema: Schema.Schema<T>
) => {
  return Effect.gen(function* () {
    const successful: Array<{ index: number; data: T }> = []
    const failed: Array<{ index: number; error: ParseResult.ParseError }> = []
    
    for (let i = 0; i < items.length; i++) {
      const result = Schema.decodeUnknownEither(schema)(items[i])
      
      if (Either.isRight(result)) {
        successful.push({ index: i, data: result.right })
      } else {
        failed.push({ 
          index: i, 
          error: ParseResult.parseError(result.left.issue) 
        })
      }
    }
    
    return {
      successful,
      failed,
      successRate: successful.length / items.length
    }
  })
}
```

### Pattern 2: Context-Aware Error Messages

```typescript
import { ParseResult, Schema, Effect } from "effect"

// Enhanced error context for better debugging
const createContextualError = (
  issue: ParseResult.ParseIssue,
  context: {
    operation: string
    source: string
    timestamp: Date
    userId?: string
  }
) => {
  const baseMessage = ParseResult.TreeFormatter.formatIssueSync(issue)
  
  return {
    message: baseMessage,
    context,
    details: {
      issueType: issue._tag,
      timestamp: context.timestamp.toISOString(),
      operation: context.operation,
      source: context.source,
      userId: context.userId
    }
  }
}

// Validation with rich context
const validateWithContext = <A, I>(
  schema: Schema.Schema<A, I>,
  data: unknown,
  context: Parameters<typeof createContextualError>[1]
) => {
  return Effect.gen(function* () {
    const result = Schema.decodeUnknownEither(schema)(data)
    
    if (Either.isLeft(result)) {
      const contextualError = createContextualError(result.left.issue, context)
      
      // Log structured error
      yield* Effect.log(JSON.stringify(contextualError, null, 2))
      
      return yield* Effect.fail(contextualError)
    }
    
    return result.right
  })
}
```

### Pattern 3: Streaming Validation

```typescript
import { ParseResult, Schema, Effect, Stream } from "effect"

// Stream-based validation for large datasets
const validateStream = <A, I>(
  schema: Schema.Schema<A, I>,
  options: {
    errorStrategy: "fail-fast" | "collect-all"
    batchSize?: number
  } = { errorStrategy: "collect-all" }
) => {
  return (source: Stream.Stream<unknown>) => {
    return source.pipe(
      Stream.mapEffect((item) => {
        const result = Schema.decodeUnknownEither(schema)(item)
        
        if (Either.isRight(result)) {
          return Effect.succeed({ success: true as const, data: result.right })
        } else {
          const error = ParseResult.parseError(result.left.issue)
          
          if (options.errorStrategy === "fail-fast") {
            return Effect.fail(error)
          }
          
          return Effect.succeed({ 
            success: false as const, 
            error: ParseResult.TreeFormatter.formatErrorSync(error)
          })
        }
      }),
      options.batchSize 
        ? Stream.buffer({ capacity: options.batchSize })
        : Stream.identity
    )
  }
}

// Usage with streaming data
const processLargeDataset = (dataStream: Stream.Stream<unknown>) => {
  return dataStream.pipe(
    validateStream(UserSchema, { errorStrategy: "collect-all", batchSize: 100 }),
    Stream.scan(
      { processed: 0, successful: 0, failed: 0, errors: [] as string[] },
      (acc, result) => ({
        processed: acc.processed + 1,
        successful: acc.successful + (result.success ? 1 : 0),
        failed: acc.failed + (result.success ? 0 : 1),
        errors: result.success ? acc.errors : [...acc.errors, result.error]
      })
    ),
    Stream.runLast
  )
}
```

## Integration Examples

### Integration with HTTP APIs

```typescript
import { ParseResult, Schema, Effect, Layer } from "effect"

// HTTP request/response validation
class ValidationService extends Effect.Service<ValidationService>()("ValidationService", {
  effect: Effect.gen(function* () {
    const formatError = (error: ParseResult.ParseError) => ({
      type: "ValidationError",
      message: "Request validation failed",
      details: ParseResult.ArrayFormatter.formatErrorSync(error)
        .map(issue => ({
          field: issue.path.join('.'),
          message: issue.message,
          code: issue._tag
        }))
    })
    
    return {
      validateRequest: <A, I>(schema: Schema.Schema<A, I>) => (data: unknown) =>
        Schema.decodeUnknown(schema)(data).pipe(
          Effect.mapError(formatError)
        ),
      
      validateResponse: <A, I>(schema: Schema.Schema<A, I>) => (data: A) =>
        Schema.encode(schema)(data).pipe(
          Effect.mapError(formatError)
        )
    }
  })
}) {}

// HTTP handler with validation
const createUserEndpoint = (requestBody: unknown) => {
  return Effect.gen(function* () {
    const validation = yield* ValidationService
    const request = yield* validation.validateRequest(CreateUserRequest)(requestBody)
    
    // Process the validated request
    const user = yield* createUser(request)
    
    return { success: true, data: user }
  }).pipe(
    Effect.catchAll((error) => 
      Effect.succeed({ 
        success: false, 
        error: error.type === "ValidationError" ? error : "Internal server error"
      })
    )
  )
}
```

### Testing Strategies

```typescript
import { ParseResult, Schema, Effect } from "effect"

// Test utilities for ParseResult validation
const expectValidationSuccess = <A, I>(
  schema: Schema.Schema<A, I>,
  input: unknown,
  expected: A
) => {
  const result = Schema.decodeUnknownEither(schema)(input)
  
  if (Either.isLeft(result)) {
    const error = ParseResult.TreeFormatter.formatErrorSync(result.left)
    throw new Error(`Expected validation to succeed, but got error: ${error}`)
  }
  
  expect(result.right).toEqual(expected)
}

const expectValidationFailure = <A, I>(
  schema: Schema.Schema<A, I>,
  input: unknown,
  expectedErrorPattern?: string | RegExp
) => {
  const result = Schema.decodeUnknownEither(schema)(input)
  
  if (Either.isRight(result)) {
    throw new Error(`Expected validation to fail, but got success: ${JSON.stringify(result.right)}`)
  }
  
  if (expectedErrorPattern) {
    const error = ParseResult.TreeFormatter.formatErrorSync(result.left)
    const pattern = typeof expectedErrorPattern === 'string' 
      ? new RegExp(expectedErrorPattern)
      : expectedErrorPattern
    
    if (!pattern.test(error)) {
      throw new Error(`Error message "${error}" did not match pattern ${pattern}`)
    }
  }
}

// Property-based testing with ParseResult
const testSchemaProperties = <A, I>(
  schema: Schema.Schema<A, I>,
  generator: () => A
) => {
  return Effect.gen(function* () {
    for (let i = 0; i < 100; i++) {
      const testValue = generator()
      
      // Test encode -> decode roundtrip
      const encoded = yield* Schema.encode(schema)(testValue)
      const decoded = yield* Schema.decode(schema)(encoded)
      
      // Values should be equal after roundtrip
      expect(decoded).toEqual(testValue)
    }
  })
}
```

## Conclusion

ParseResult provides comprehensive error handling, rich context information, and flexible formatting options for data validation and parsing operations.

Key benefits:
- **Rich Error Context**: Detailed information about what went wrong and where
- **Composable Validation**: Build complex validators from simple components
- **Flexible Formatting**: Multiple formatters for different output needs
- **Effect Integration**: Seamless integration with Effect's error handling system

ParseResult is essential when you need robust data validation with detailed error reporting, making it perfect for API validation, configuration processing, and data transformation pipelines.