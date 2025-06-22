# VariantSchema: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

VariantSchema is a powerful tool in Effect-TS's experimental package that solves a common problem in modern applications: **different representations of the same data structure for different contexts**.

### The Problem VariantSchema Solves

Consider a typical user model in an application:

```typescript
// Traditional approach - multiple separate schemas
const UserInsert = Schema.Struct({
  name: Schema.String,
  email: Schema.String,
  // No id or timestamps - these are generated
})

const UserSelect = Schema.Struct({
  id: Schema.String,
  name: Schema.String,
  email: Schema.String,
  createdAt: Schema.Date,
  updatedAt: Schema.Date,
})

const UserUpdate = Schema.Struct({
  id: Schema.String,
  name: Schema.optional(Schema.String),
  email: Schema.optional(Schema.String),
  updatedAt: Schema.Date,
  // No createdAt - it doesn't change
})

const UserJson = Schema.Struct({
  id: Schema.String,
  name: Schema.String,
  email: Schema.String,
  createdAt: Schema.DateTimeUtc,
  updatedAt: Schema.DateTimeUtc,
  // Dates formatted for JSON
})
```

This approach leads to:
- **Code duplication** - similar fields defined multiple times
- **Schema drift** - easy to forget updating all variants when adding fields
- **Type complexity** - managing multiple related types manually
- **Maintenance burden** - keeping variants in sync

### The VariantSchema Solution

VariantSchema allows you to define **one schema with multiple variants**, where each field can have different behavior per variant:

```typescript
import { VariantSchema } from "@effect/experimental"

const { Struct, Field, extract } = VariantSchema.make({
  variants: ["insert", "select", "update", "json"] as const,
  defaultVariant: "select"
})

// Define the User schema once with variant-aware fields
const User = Struct({
  id: Field({
    select: Schema.String,
    json: Schema.String,
    // id is auto-generated, not present in insert/update
  }),
  name: Field({
    insert: Schema.String,
    select: Schema.String,
    update: Schema.optional(Schema.String),
    json: Schema.String,
  }),
  email: Field({
    insert: Schema.String,
    select: Schema.String,
    update: Schema.optional(Schema.String),
    json: Schema.String,
  }),
  createdAt: Field({
    select: Schema.Date,
    json: Schema.DateTimeUtc,
    // Not present in insert (auto-generated) or update (immutable)
  }),
  updatedAt: Field({
    select: Schema.Date,
    update: Schema.Date,
    json: Schema.DateTimeUtc,
    // Not present in insert (auto-generated)
  }),
})

// Extract specific variants as needed
const UserInsert = extract(User, "insert")
const UserSelect = extract(User, "select") 
const UserUpdate = extract(User, "update")
const UserJson = extract(User, "json")
```

### Key Concepts

**Variants**: Different "versions" or "contexts" of your data structure (e.g., "insert", "select", "update", "json")

**Fields**: Schema definitions that can vary per variant. If a field isn't defined for a variant, it's excluded from that variant's type.

**Default Variant**: The variant used when accessing the schema directly (without extraction).

**Evolution**: The ability to transform field definitions programmatically across variants.

## Basic Usage Patterns

### Setting Up Variants

```typescript
import { VariantSchema } from "@effect/experimental"
import { Schema } from "@effect/schema"

// Define your variants upfront
const { Struct, Field, FieldOnly, FieldExcept, extract } = VariantSchema.make({
  variants: ["draft", "published", "archived"] as const,
  defaultVariant: "published"
})
```

### Creating Struct Schemas

```typescript
const Article = Struct({
  id: Field({
    published: Schema.String,
    archived: Schema.String,
    // id not present in draft (not yet saved)
  }),
  title: Field({
    draft: Schema.String,
    published: Schema.String,
    archived: Schema.String,
  }),
  content: Field({
    draft: Schema.String,
    published: Schema.String,
    archived: Schema.String,
  }),
  publishedAt: Field({
    published: Schema.Date,
    archived: Schema.Date,
    // publishedAt not present in draft
  }),
  archivedAt: Field({
    archived: Schema.Date,
    // archivedAt only present when archived
  }),
})

// Extract specific variants
const DraftArticle = extract(Article, "draft")
const PublishedArticle = extract(Article, "published")
const ArchivedArticle = extract(Article, "archived")
```

### Using FieldOnly and FieldExcept

These utilities help you create fields that are present only in specific variants or excluded from specific variants:

```typescript
const Product = Struct({
  id: FieldOnly("published", "archived")(Schema.String),
  name: Field({
    draft: Schema.String,
    published: Schema.String,
    archived: Schema.String,
  }),
  price: Field({
    draft: Schema.Number,
    published: Schema.Number,
    archived: Schema.Number,
  }),
  internalNotes: FieldOnly("draft")(Schema.String),
  publicDescription: FieldExcept("draft")(Schema.String),
})
```

## Real-World Examples

### Example 1: Database Model with Multiple Operations

This pattern is commonly used for database models where you need different schemas for different operations:

```typescript
import { VariantSchema } from "@effect/experimental"
import { Schema } from "@effect/schema"

// Define variants for common database operations
const { Struct, Field, extract, fieldEvolve } = VariantSchema.make({
  variants: ["insert", "select", "update", "json", "jsonCreate", "jsonUpdate"] as const,
  defaultVariant: "select"
})

// Helper for generated fields (present in select/json but not insert/update)
const Generated = <T extends Schema.Schema.All>(schema: T) => Field({
  select: schema,
  json: schema,
  jsonCreate: schema,
  jsonUpdate: schema,
})

// Helper for timestamp fields that auto-populate on insert
const DateTimeInsert = Field({
  select: Schema.Date,
  update: Schema.Date,
  json: Schema.DateTimeUtc,
  jsonUpdate: Schema.DateTimeUtc,
})

// Helper for timestamp fields that auto-update
const DateTimeUpdate = Field({
  select: Schema.Date,
  update: Schema.Date,
  json: Schema.DateTimeUtc,
  jsonCreate: Schema.DateTimeUtc,
  jsonUpdate: Schema.DateTimeUtc,
})

// Helper for sensitive fields (excluded from JSON variants)
const Sensitive = <T extends Schema.Schema.All>(schema: T) => Field({
  insert: schema,
  select: schema,
  update: Schema.optional(schema),
})

// Define a comprehensive user model
const User = Struct({
  id: Generated(Schema.String),
  email: Field({
    insert: Schema.String,
    select: Schema.String,
    update: Schema.optional(Schema.String),
    json: Schema.String,
    jsonCreate: Schema.String,
    jsonUpdate: Schema.String,
  }),
  passwordHash: Sensitive(Schema.String),
  firstName: Field({
    insert: Schema.String,
    select: Schema.String,
    update: Schema.optional(Schema.String),
    json: Schema.String,
    jsonCreate: Schema.String,
    jsonUpdate: Schema.String,
  }),
  lastName: Field({
    insert: Schema.String,
    select: Schema.String,
    update: Schema.optional(Schema.String),
    json: Schema.String,
    jsonCreate: Schema.String,
    jsonUpdate: Schema.String,
  }),
  role: Field({
    insert: Schema.Literal("user"),
    select: Schema.Union(Schema.Literal("user"), Schema.Literal("admin")),
    update: Schema.optional(Schema.Union(Schema.Literal("user"), Schema.Literal("admin"))),
    json: Schema.Union(Schema.Literal("user"), Schema.Literal("admin")),
    jsonCreate: Schema.Literal("user"),
    jsonUpdate: Schema.Union(Schema.Literal("user"), Schema.Literal("admin")),
  }),
  isEmailVerified: Field({
    insert: Schema.Literal(false),
    select: Schema.Boolean,
    update: Schema.optional(Schema.Boolean),
    json: Schema.Boolean,
    jsonCreate: Schema.Literal(false),
    jsonUpdate: Schema.Boolean,
  }),
  createdAt: DateTimeInsert,
  updatedAt: DateTimeUpdate,
})

// Extract schemas for different use cases
const UserInsert = extract(User, "insert")        // For creating new users
const UserSelect = extract(User, "select")        // For database queries
const UserUpdate = extract(User, "update")        // For updating existing users
const UserJson = extract(User, "json")            // For API responses
const UserJsonCreate = extract(User, "jsonCreate") // For API create responses
const UserJsonUpdate = extract(User, "jsonUpdate") // For API update responses

// Type-safe usage
type InsertUser = Schema.Schema.Type<typeof UserInsert>
// { email: string; passwordHash: string; firstName: string; lastName: string; role: "user"; isEmailVerified: false }

type SelectUser = Schema.Schema.Type<typeof UserSelect>
// { id: string; email: string; passwordHash: string; firstName: string; lastName: string; role: "user" | "admin"; isEmailVerified: boolean; createdAt: Date; updatedAt: Date }

type JsonUser = Schema.Schema.Type<typeof UserJson>
// { id: string; email: string; firstName: string; lastName: string; role: "user" | "admin"; isEmailVerified: boolean; createdAt: DateTimeUtc; updatedAt: DateTimeUtc }
```

### Example 2: API Response Schemas with Different Serialization

```typescript
const { Struct, Field, extract } = VariantSchema.make({
  variants: ["internal", "public", "admin", "minimal"] as const,
  defaultVariant: "public"
})

const UserProfile = Struct({
  id: Field({
    internal: Schema.String,
    public: Schema.String,
    admin: Schema.String,
    minimal: Schema.String,
  }),
  username: Field({
    internal: Schema.String,
    public: Schema.String,
    admin: Schema.String,
    minimal: Schema.String,
  }),
  email: Field({
    internal: Schema.String,
    admin: Schema.String,
    // Email not exposed in public/minimal views
  }),
  fullName: Field({
    internal: Schema.String,
    public: Schema.String,
    admin: Schema.String,
    // Full name not in minimal view
  }),
  internalId: Field({
    internal: Schema.Number,
    admin: Schema.Number,
    // Internal ID only for internal/admin use
  }),
  lastLoginAt: Field({
    internal: Schema.DateTimeUtc,
    admin: Schema.DateTimeUtc,
    // Login info not public
  }),
  permissions: Field({
    internal: Schema.Array(Schema.String),
    admin: Schema.Array(Schema.String),
    // Permissions only for internal/admin
  }),
  profilePicture: Field({
    internal: Schema.String,
    public: Schema.String,
    admin: Schema.String,
    // No profile picture in minimal view
  }),
})

// Different API endpoints can use different variants
const PublicProfile = extract(UserProfile, "public")     // /api/users/:id
const AdminProfile = extract(UserProfile, "admin")       // /api/admin/users/:id
const MinimalProfile = extract(UserProfile, "minimal")   // /api/users/search results
const InternalProfile = extract(UserProfile, "internal") // Internal service calls
```

### Example 3: Form Validation with State Management

```typescript
import { pipe } from "effect"

const { Struct, Field, extract, fieldEvolve } = VariantSchema.make({
  variants: ["input", "validation", "submitted", "persisted"] as const,
  defaultVariant: "input"
})

const ContactForm = Struct({
  name: Field({
    input: Schema.String,
    validation: Schema.String.pipe(Schema.minLength(1), Schema.maxLength(100)),
    submitted: Schema.String.pipe(Schema.minLength(1), Schema.maxLength(100)),
    persisted: Schema.String.pipe(Schema.minLength(1), Schema.maxLength(100)),
  }),
  email: Field({
    input: Schema.String,
    validation: Schema.String.pipe(Schema.pattern(/^[^\s@]+@[^\s@]+\.[^\s@]+$/)),
    submitted: Schema.String.pipe(Schema.pattern(/^[^\s@]+@[^\s@]+\.[^\s@]+$/)),
    persisted: Schema.String.pipe(Schema.pattern(/^[^\s@]+@[^\s@]+\.[^\s@]+$/)),
  }),
  message: Field({
    input: Schema.String,
    validation: Schema.String.pipe(Schema.minLength(10), Schema.maxLength(1000)),
    submitted: Schema.String.pipe(Schema.minLength(10), Schema.maxLength(1000)),
    persisted: Schema.String.pipe(Schema.minLength(10), Schema.maxLength(1000)),
  }),
  id: Field({
    persisted: Schema.String,
    // ID only exists after persistence
  }),
  submittedAt: Field({
    submitted: Schema.Date,
    persisted: Schema.Date,
    // Timestamp only after submission
  }),
  status: Field({
    submitted: Schema.Literal("pending"),
    persisted: Schema.Union(Schema.Literal("pending"), Schema.Literal("processed"), Schema.Literal("archived")),
  }),
})

// Different form states
const FormInput = extract(ContactForm, "input")         // Initial form data
const FormValidation = extract(ContactForm, "validation") // With validation rules
const FormSubmitted = extract(ContactForm, "submitted")   // After form submission
const FormPersisted = extract(ContactForm, "persisted")   // After database save
```

## Advanced Features Deep Dive

### fieldEvolve: Transforming Field Definitions

`fieldEvolve` is one of the most powerful features of VariantSchema. It lets you transform field definitions programmatically, applying changes across variants in a type-safe way.

#### Basic fieldEvolve Usage

```typescript
import { pipe } from "effect"

const { Struct, Field, fieldEvolve } = VariantSchema.make({
  variants: ["draft", "published"] as const,
  defaultVariant: "published"
})

// Original field
const titleField = Field({
  draft: Schema.String,
  published: Schema.String,
})

// Transform the field - make published variant required with min length
const evolvedTitleField = fieldEvolve(titleField, {
  published: (schema) => Schema.minLength(schema, 1),
})

// Result: Field({
//   draft: Schema.String,
//   published: Schema.String.pipe(Schema.minLength(1)),
// })
```

#### Real-World fieldEvolve Example: Adding Validation

```typescript
import { pipe } from "effect"

const BaseUser = Struct({
  email: Field({
    input: Schema.String,
    validated: Schema.String,
    persisted: Schema.String,
  }),
  age: Field({
    input: Schema.Number,
    validated: Schema.Number,
    persisted: Schema.Number,
  }),
})

// Add validation rules to all "validated" and "persisted" variants
const ValidatedUser = Struct({
  email: fieldEvolve(BaseUser.schemas.email, {
    validated: (schema) => Schema.pattern(schema, /^[^\s@]+@[^\s@]+\.[^\s@]+$/),
    persisted: (schema) => Schema.pattern(schema, /^[^\s@]+@[^\s@]+\.[^\s@]+$/),
  }),
  age: fieldEvolve(BaseUser.schemas.age, {
    validated: (schema) => schema.pipe(Schema.int(), Schema.between(0, 150)),
    persisted: (schema) => schema.pipe(Schema.int(), Schema.between(0, 150)),
  }),
})
```

#### Advanced fieldEvolve: Conditional Transformations

```typescript
import { pipe } from "effect"

// Helper function to add validation only to specific variants
const addValidation = <T extends Field<any>>(
  field: T,
  validationVariants: string[],
  validator: (schema: any) => any
) => {
  const transformations = Object.fromEntries(
    validationVariants.map(variant => [variant, validator])
  )
  return fieldEvolve(field, transformations)
}

const User = Struct({
  email: addValidation(
    Field({
      input: Schema.String,
      draft: Schema.String,
      published: Schema.String,
    }),
    ["published"],
    (schema) => Schema.pattern(schema, /^[^\s@]+@[^\s@]+\.[^\s@]+$/)
  ),
})
```

### fieldFromKey: Field Renaming with fromKey.Rename

`fieldFromKey` allows you to rename fields differently across variants using Schema's `fromKey` functionality:

```typescript
import { Schema } from "@effect/schema"

const { Struct, Field, fieldFromKey } = VariantSchema.make({
  variants: ["api", "database", "internal"] as const,
  defaultVariant: "internal"
})

const User = Struct({
  id: fieldFromKey(
    Field({
      api: Schema.String,
      database: Schema.String,
      internal: Schema.String,
    }),
    {
      api: "userId",        // Renamed to "userId" in API variant
      database: "user_id",  // Renamed to "user_id" in database variant
      // "id" remains "id" in internal variant
    }
  ),
  firstName: fieldFromKey(
    Field({
      api: Schema.String,
      database: Schema.String,
      internal: Schema.String,
    }),
    {
      api: "firstName",
      database: "first_name",
      internal: "firstName",
    }
  ),
})

// When using these schemas:
const ApiUser = extract(User, "api")
// Type: { userId: string; firstName: string }

const DatabaseUser = extract(User, "database")
// Type: { user_id: string; first_name: string }

const InternalUser = extract(User, "internal")
// Type: { id: string; firstName: string }
```

### extract: Runtime Variant Selection

The `extract` function provides both compile-time and runtime access to specific variants:

```typescript
const { Struct, Field, extract } = VariantSchema.make({
  variants: ["v1", "v2", "v3"] as const,
  defaultVariant: "v2"
})

const ApiResponse = Struct({
  data: Field({
    v1: Schema.String,
    v2: Schema.Struct({ value: Schema.String }),
    v3: Schema.Struct({ value: Schema.String, metadata: Schema.Record({ key: Schema.String, value: Schema.Unknown }) }),
  }),
  version: Field({
    v1: Schema.Literal("1.0"),
    v2: Schema.Literal("2.0"),
    v3: Schema.Literal("3.0"),
  }),
})

// Static extraction (compile-time)
const V1Response = extract(ApiResponse, "v1")
const V2Response = extract(ApiResponse, "v2")
const V3Response = extract(ApiResponse, "v3")

// Dynamic extraction (runtime)
const getResponseSchema = (version: "v1" | "v2" | "v3") => {
  return extract(ApiResponse, version)
}

// Usage in API versioning
const handleRequest = (version: string, data: unknown) => {
  switch (version) {
    case "v1":
      return Schema.decodeUnknown(extract(ApiResponse, "v1"))(data)
    case "v2":
      return Schema.decodeUnknown(extract(ApiResponse, "v2"))(data)
    case "v3":
      return Schema.decodeUnknown(extract(ApiResponse, "v3"))(data)
    default:
      return Schema.decodeUnknown(ApiResponse)(data) // Uses default variant
  }
}
```

### Union: Combining Multiple Variant Structs

The `Union` utility allows you to create discriminated unions from multiple variant-aware structs:

```typescript
const { Struct, Field, Union, extract } = VariantSchema.make({
  variants: ["create", "read", "update"] as const,
  defaultVariant: "read"
})

const UserEvent = Struct({
  type: Field({
    create: Schema.Literal("user_created"),
    read: Schema.Literal("user_created"),
    update: Schema.Literal("user_created"),
  }),
  userId: Field({
    create: Schema.String,
    read: Schema.String,
    update: Schema.String,
  }),
  userData: Field({
    create: Schema.Struct({
      email: Schema.String,
      name: Schema.String,
    }),
    read: Schema.Struct({
      email: Schema.String,
      name: Schema.String,
    }),
  }),
})

const ProductEvent = Struct({
  type: Field({
    create: Schema.Literal("product_created"),
    read: Schema.Literal("product_created"),
    update: Schema.Literal("product_created"),
  }),
  productId: Field({
    create: Schema.String,
    read: Schema.String,
    update: Schema.String,
  }),
  productData: Field({
    create: Schema.Struct({
      name: Schema.String,
      price: Schema.Number,
    }),
    read: Schema.Struct({
      name: Schema.String,
      price: Schema.Number,
    }),
  }),
})

// Create a union of events
const Event = Union(UserEvent, ProductEvent)

// Extract variants
const CreateEvent = extract(Event, "create")
const ReadEvent = extract(Event, "read")
const UpdateEvent = extract(Event, "update")
```

### Class: Creating Variant-Aware Classes

The `Class` utility creates Effect-TS classes that are aware of your variants:

```typescript
const { Class, Field } = VariantSchema.make({
  variants: ["create", "persisted", "json"] as const,
  defaultVariant: "persisted"
})

// Create a variant-aware User class
export class User extends Class<User>("User")({
  id: Field({
    persisted: Schema.String,
    json: Schema.String,
    // No id in create variant
  }),
  email: Field({
    create: Schema.String,
    persisted: Schema.String,
    json: Schema.String,
  }),
  name: Field({
    create: Schema.String,
    persisted: Schema.String,
    json: Schema.String,
  }),
  createdAt: Field({
    persisted: Schema.Date,
    json: Schema.DateTimeUtc,
    // No createdAt in create variant
  }),
}) {
  // Add methods that work with the default variant (persisted)
  get displayName() {
    return this.name
  }
  
  // Method to convert to JSON variant
  toJson() {
    return extract(this, "json")
  }
}

// Usage
const createUserData = { email: "test@example.com", name: "John Doe" }
const user = new User(createUserData) // Uses default variant (persisted)
```

## Practical Patterns & Best Practices

### Pattern 1: Handling Sensitive Data

```typescript
const { Struct, Field, extract } = VariantSchema.make({
  variants: ["internal", "api", "audit"] as const,
  defaultVariant: "internal"
})

// Helper for sensitive fields that should be excluded from API responses
const Sensitive = <T extends Schema.Schema.All>(schema: T) => Field({
  internal: schema,
  audit: Schema.redacted(schema), // Redacted in audit logs
  // Not present in API variant
})

// Helper for API-safe fields
const Public = <T extends Schema.Schema.All>(schema: T) => Field({
  internal: schema,
  api: schema,
  audit: schema,
})

const User = Struct({
  id: Public(Schema.String),
  email: Public(Schema.String),
  passwordHash: Sensitive(Schema.String),
  socialSecurityNumber: Sensitive(Schema.String),
  name: Public(Schema.String),
  apiKey: Sensitive(Schema.String),
})

const ApiUser = extract(User, "api")        // Safe for API responses
const InternalUser = extract(User, "internal") // Full data for internal use
const AuditUser = extract(User, "audit")    // Redacted sensitive data for logs
```

### Pattern 2: Generated Fields vs User Inputs

```typescript
const { Struct, Field, extract } = VariantSchema.make({
  variants: ["input", "persisted", "api"] as const,
  defaultVariant: "persisted"
})

// Helper for auto-generated fields
const Generated = <T extends Schema.Schema.All>(schema: T) => Field({
  persisted: schema,
  api: schema,
  // Not present in input - generated by the system
})

// Helper for user-provided fields
const UserProvided = <T extends Schema.Schema.All>(inputSchema: T, persistedSchema?: Schema.Schema.All) => Field({
  input: inputSchema,
  persisted: persistedSchema ?? inputSchema,
  api: persistedSchema ?? inputSchema,
})

const BlogPost = Struct({
  id: Generated(Schema.String),
  slug: Generated(Schema.String), // Generated from title
  title: UserProvided(Schema.String),
  content: UserProvided(Schema.String),
  excerpt: Generated(Schema.String), // Auto-generated from content
  publishedAt: Generated(Schema.Date),
  createdAt: Generated(Schema.Date),
  updatedAt: Generated(Schema.Date),
})

const BlogPostInput = extract(BlogPost, "input")      // { title: string; content: string }
const BlogPostPersisted = extract(BlogPost, "persisted") // Full persisted data
const BlogPostApi = extract(BlogPost, "api")          // API response data
```

### Pattern 3: DateTime Handling for Create/Update Operations

```typescript
const { Struct, Field, extract, fieldEvolve } = VariantSchema.make({
  variants: ["create", "update", "select", "json"] as const,
  defaultVariant: "select"
})

// Timestamp field that's auto-set on creation, never updated
const CreatedTimestamp = Field({
  select: Schema.Date,
  json: Schema.DateTimeUtc,
  // Not present in create/update - set by database
})

// Timestamp field that's auto-set on creation and auto-updated
const UpdatedTimestamp = Field({
  select: Schema.Date,
  update: Schema.Date, // Updated by database trigger
  json: Schema.DateTimeUtc,
  // Not present in create - set by database
})

// User-controlled timestamp (e.g., scheduled publish date)
const UserTimestamp = <Required extends boolean = false>(required?: Required) => Field({
  create: required ? Schema.Date : Schema.optional(Schema.Date),
  update: Schema.optional(Schema.Date),
  select: Schema.Date,
  json: Schema.DateTimeUtc,
})

const Article = Struct({
  id: Field({
    select: Schema.String,
    json: Schema.String,
  }),
  title: Field({
    create: Schema.String,
    update: Schema.optional(Schema.String),
    select: Schema.String,
    json: Schema.String,
  }),
  publishedAt: UserTimestamp(false), // Optional publish date
  createdAt: CreatedTimestamp,
  updatedAt: UpdatedTimestamp,
})
```

### Pattern 4: Optional vs Required Fields Per Variant

```typescript
const { Struct, Field, extract } = VariantSchema.make({
  variants: ["draft", "review", "published"] as const,
  defaultVariant: "published"
})

// Helper for progressive validation - more fields required as status advances
const Progressive = <T extends Schema.Schema.All>(
  schema: T, 
  stages: { draft?: boolean; review?: boolean; published?: boolean } = {}
) => {
  const draftSchema = stages.draft ? schema : Schema.optional(schema)
  const reviewSchema = stages.review ? schema : Schema.optional(schema)
  const publishedSchema = stages.published !== false ? schema : Schema.optional(schema)
  
  return Field({
    draft: draftSchema,
    review: reviewSchema,
    published: publishedSchema,
  })
}

const Article = Struct({
  id: Field({
    review: Schema.String,
    published: Schema.String,
    // No ID in draft
  }),
  title: Progressive(Schema.String, { draft: true, review: true, published: true }),
  content: Progressive(Schema.String, { draft: false, review: true, published: true }),
  excerpt: Progressive(Schema.String, { draft: false, review: false, published: true }),
  tags: Progressive(Schema.Array(Schema.String), { draft: false, review: false, published: true }),
  featuredImage: Progressive(Schema.String, { draft: false, review: false, published: true }),
  seoTitle: Progressive(Schema.String, { draft: false, review: false, published: true }),
  seoDescription: Progressive(Schema.String, { draft: false, review: false, published: true }),
})

// Draft: Only title required
// Review: Title and content required
// Published: All fields required
```

### Pattern 5: Error Handling and Validation

```typescript
import { ParseResult } from "@effect/schema"
import { Effect } from "effect"

const { Struct, Field, extract } = VariantSchema.make({
  variants: ["lenient", "strict", "api"] as const,
  defaultVariant: "strict"
})

const UserInput = Struct({
  email: Field({
    lenient: Schema.String, // Accept any string
    strict: Schema.String.pipe(Schema.pattern(/^[^\s@]+@[^\s@]+\.[^\s@]+$/)),
    api: Schema.String.pipe(Schema.pattern(/^[^\s@]+@[^\s@]+\.[^\s@]+$/)),
  }),
  age: Field({
    lenient: Schema.Union(Schema.String, Schema.Number), // Accept string or number
    strict: Schema.Number.pipe(Schema.int(), Schema.between(0, 150)),
    api: Schema.Number.pipe(Schema.int(), Schema.between(0, 150)),
  }),
  name: Field({
    lenient: Schema.String,
    strict: Schema.String.pipe(Schema.minLength(1), Schema.maxLength(100)),
    api: Schema.String.pipe(Schema.minLength(1), Schema.maxLength(100)),
  }),
})

// Progressive validation approach
const validateUser = (data: unknown) => {
  // First, try lenient validation to see what data we have
  return Effect.gen(function* () {
    const lenientResult = yield* Schema.decodeUnknown(extract(UserInput, "lenient"))(data)
    
    // Then try strict validation
    const strictResult = yield* Schema.decodeUnknown(extract(UserInput, "strict"))(data).pipe(
      Effect.catchAll((error) => Effect.succeed({ error, data: lenientResult }))
    )
    
    return strictResult
  })
}
```

## Integration Examples

### Integration with ElectroDB

```typescript
import { Entity } from "electrodb"
import { VariantSchema } from "@effect/experimental"
import { Schema } from "@effect/schema"

const { Struct, Field, extract } = VariantSchema.make({
  variants: ["electrodb", "api", "internal"] as const,
  defaultVariant: "internal"
})

// Define schema with ElectroDB-specific transformations
const User = Struct({
  pk: Field({
    electrodb: Schema.String, // Partition key for ElectroDB
    // Not exposed in API or internal variants
  }),
  sk: Field({
    electrodb: Schema.String, // Sort key for ElectroDB
    // Not exposed in API or internal variants
  }),
  id: Field({
    internal: Schema.String,
    api: Schema.String,
    // Maps to pk/sk in ElectroDB
  }),
  email: Field({
    electrodb: Schema.String,
    internal: Schema.String,
    api: Schema.String,
  }),
  name: Field({
    electrodb: Schema.String,
    internal: Schema.String,
    api: Schema.String,
  }),
  createdAt: Field({
    electrodb: Schema.Number, // Unix timestamp for ElectroDB
    internal: Schema.Date,
    api: Schema.DateTimeUtc,
  }),
})

// ElectroDB entity definition
const UserEntity = new Entity({
  model: {
    entity: "user",
    version: "1",
    service: "app",
  },
  attributes: {
    pk: { type: "string" },
    sk: { type: "string" },
    email: { type: "string" },
    name: { type: "string" },
    createdAt: { type: "number" },
  },
  indexes: {
    primary: {
      pk: { field: "pk" },
      sk: { field: "sk" },
    },
  },
})

// Helper to convert between variants
const toElectroDBFormat = (user: Schema.Schema.Type<typeof extract(User, "internal")>) => {
  return Schema.encode(extract(User, "electrodb"))({
    pk: `USER#${user.id}`,
    sk: `USER#${user.id}`,
    email: user.email,
    name: user.name,
    createdAt: Math.floor(user.createdAt.getTime() / 1000),
  })
}

const fromElectroDBFormat = (dbUser: any) => {
  return Schema.decode(extract(User, "internal"))({
    id: dbUser.pk.replace("USER#", ""),
    email: dbUser.email,
    name: dbUser.name,
    createdAt: new Date(dbUser.createdAt * 1000),
  })
}
```

### Integration with API Serialization

```typescript
import { FastifyInstance } from "fastify"
import { VariantSchema } from "@effect/experimental"
import { Schema } from "@effect/schema"

const { Struct, Field, extract } = VariantSchema.make({
  variants: ["input", "output", "error"] as const,
  defaultVariant: "output"
})

const CreateUserRequest = Struct({
  email: Field({
    input: Schema.String.pipe(Schema.pattern(/^[^\s@]+@[^\s@]+\.[^\s@]+$/)),
  }),
  name: Field({
    input: Schema.String.pipe(Schema.minLength(1), Schema.maxLength(100)),
  }),
  password: Field({
    input: Schema.String.pipe(Schema.minLength(8)),
  }),
})

const CreateUserResponse = Struct({
  id: Field({
    output: Schema.String,
  }),
  email: Field({
    output: Schema.String,
  }),
  name: Field({
    output: Schema.String,
  }),
  createdAt: Field({
    output: Schema.DateTimeUtc,
  }),
  errors: Field({
    error: Schema.Array(Schema.Struct({
      field: Schema.String,
      message: Schema.String,
    })),
  }),
})

// Fastify route with automatic validation
const registerRoutes = (app: FastifyInstance) => {
  app.post('/users', {
    schema: {
      body: Schema.to(extract(CreateUserRequest, "input")),
      response: {
        200: Schema.to(extract(CreateUserResponse, "output")),
        400: Schema.to(extract(CreateUserResponse, "error")),
      },
    },
  }, async (request, reply) => {
    try {
      // Input is automatically validated by Fastify schema
      const userData = request.body
      
      // Process user creation...
      const user = await createUser(userData)
      
      // Return success response
      return Schema.encode(extract(CreateUserResponse, "output"))(user)
    } catch (error) {
      reply.code(400)
      return Schema.encode(extract(CreateUserResponse, "error"))({
        errors: [{ field: "general", message: "Failed to create user" }]
      })
    }
  })
}
```

### Integration with Form Libraries (React Hook Form)

```typescript
import { useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import { VariantSchema } from "@effect/experimental"
import { Schema } from "@effect/schema"

const { Struct, Field, extract } = VariantSchema.make({
  variants: ["form", "api", "display"] as const,
  defaultVariant: "form"
})

const ContactForm = Struct({
  name: Field({
    form: Schema.String,
    api: Schema.String.pipe(Schema.minLength(1)),
    display: Schema.String,
  }),
  email: Field({
    form: Schema.String,
    api: Schema.String.pipe(Schema.pattern(/^[^\s@]+@[^\s@]+\.[^\s@]+$/)),
    display: Schema.String,
  }),
  message: Field({
    form: Schema.String,
    api: Schema.String.pipe(Schema.minLength(10)),
    display: Schema.String,
  }),
  newsletter: Field({
    form: Schema.Boolean,
    api: Schema.Boolean,
    display: Schema.Boolean,
  }),
})

// React component
const ContactFormComponent = () => {
  const form = useForm({
    resolver: zodResolver(Schema.to(extract(ContactForm, "form"))),
    defaultValues: {
      name: "",
      email: "",
      message: "",
      newsletter: false,
    },
  })

  const onSubmit = async (data: Schema.Schema.Type<typeof extract(ContactForm, "form")>) => {
    try {
      // Validate for API submission
      const apiData = await Schema.decode(extract(ContactForm, "api"))(data)
      
      // Submit to API
      const response = await fetch("/api/contact", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(apiData),
      })
      
      if (response.ok) {
        // Success handling
        console.log("Form submitted successfully")
      }
    } catch (error) {
      // Handle validation errors
      console.error("Validation failed:", error)
    }
  }

  return (
    <form onSubmit={form.handleSubmit(onSubmit)}>
      {/* Form fields */}
    </form>
  )
}
```

### Testing Strategies for Variant Schemas

```typescript
import { describe, it, expect } from "vitest"
import { VariantSchema } from "@effect/experimental"
import { Schema } from "@effect/schema"
import { Effect } from "effect"

const { Struct, Field, extract } = VariantSchema.make({
  variants: ["input", "output", "test"] as const,
  defaultVariant: "output"
})

const User = Struct({
  id: Field({
    output: Schema.String,
    test: Schema.String,
  }),
  email: Field({
    input: Schema.String,
    output: Schema.String,
    test: Schema.Literal("test@example.com"),
  }),
  name: Field({
    input: Schema.String,
    output: Schema.String,
    test: Schema.Literal("Test User"),
  }),
})

describe("User Schema Variants", () => {
  it("should validate input variant", async () => {
    const inputData = { email: "test@example.com", name: "Test User" }
    const result = await Effect.runPromise(
      Schema.decode(extract(User, "input"))(inputData)
    )
    expect(result).toEqual(inputData)
  })

  it("should validate output variant", async () => {
    const outputData = { id: "123", email: "test@example.com", name: "Test User" }
    const result = await Effect.runPromise(
      Schema.decode(extract(User, "output"))(outputData)
    )
    expect(result).toEqual(outputData)
  })

  it("should create test fixtures using test variant", async () => {
    const testFixture = { id: "test-id", email: "test@example.com", name: "Test User" }
    
    // Test variant provides default values
    const result = await Effect.runPromise(
      Schema.decode(extract(User, "test"))(testFixture)
    )
    
    expect(result.email).toBe("test@example.com")
    expect(result.name).toBe("Test User")
  })

  it("should handle variant transformation", async () => {
    const inputData = { email: "test@example.com", name: "Test User" }
    
    // Simulate processing: input -> output
    const processedData = { ...inputData, id: "generated-id" }
    
    const result = await Effect.runPromise(
      Schema.decode(extract(User, "output"))(processedData)
    )
    
    expect(result).toHaveProperty("id")
    expect(result.id).toBe("generated-id")
  })
})

// Test helpers for variant schemas
const createTestSuite = <T extends Struct<any>>(
  schema: T,
  variants: Record<string, unknown>
) => {
  return Object.entries(variants).map(([variantName, testData]) => ({
    variant: variantName,
    data: testData,
    test: () => Schema.decode(extract(schema, variantName as any))(testData),
  }))
}

// Usage
const userTestSuite = createTestSuite(User, {
  input: { email: "test@example.com", name: "Test User" },
  output: { id: "123", email: "test@example.com", name: "Test User" },
  test: { id: "test-id", email: "test@example.com", name: "Test User" },
})

describe("User Schema Test Suite", () => {
  userTestSuite.forEach(({ variant, data, test }) => {
    it(`should validate ${variant} variant`, async () => {
      const result = await Effect.runPromise(test())
      expect(result).toEqual(data)
    })
  })
})
```

## Conclusion

VariantSchema is a powerful tool for managing complex data schemas with multiple representations. It shines in real-world applications where the same logical entity needs different shapes for different contexts - whether that's database operations, API serialization, form validation, or internal processing.

Key benefits:
- **DRY (Don't Repeat Yourself)**: Define fields once, use across variants
- **Type Safety**: Full TypeScript support with proper inference
- **Maintainability**: Changes propagate across all variants automatically
- **Flexibility**: Rich transformation and evolution capabilities
- **Performance**: Compile-time optimizations and runtime efficiency

The patterns and examples in this guide should provide a solid foundation for using VariantSchema in your Effect-TS applications. Remember to start simple and gradually add complexity as your needs evolve.