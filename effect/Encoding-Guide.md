# Encoding: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem Encoding Solves

Modern applications frequently need to encode and decode data in various formats - base64 for data URLs and authentication tokens, hex for cryptographic operations, and URL encoding for web parameters. Traditional JavaScript approaches lead to scattered utility functions, inconsistent error handling, and unsafe operations.

```typescript
// Traditional approach - error-prone and inconsistent
const processApiToken = (tokenData: string) => {
  try {
    // Manual base64 encoding with no type safety
    const encoded = btoa(tokenData) // Can throw on invalid Unicode
    
    // Manual URL encoding with potential failures
    const urlSafe = encodeURIComponent(encoded)
    
    // Manual hex conversion for checksums
    const checksum = Array.from(new TextEncoder().encode(encoded))
      .map(b => b.toString(16).padStart(2, '0'))
      .join('')
    
    return { token: urlSafe, checksum }
  } catch (error) {
    // Unclear error types and inconsistent handling
    throw new Error('Encoding failed: ' + error.message)
  }
}

// Decoding with manual error handling
const decodeUserData = (base64Data: string) => {
  try {
    const decoded = atob(base64Data) // Can throw on invalid input
    return JSON.parse(decoded) // Can throw on invalid JSON
  } catch (error) {
    return null // Lost error information
  }
}

// Hex operations scattered across utilities
const bytesToHex = (bytes: Uint8Array) => {
  return Array.from(bytes, byte => 
    byte.toString(16).padStart(2, '0')
  ).join('')
}

const hexToBytes = (hex: string) => {
  if (hex.length % 2 !== 0) throw new Error('Invalid hex length')
  const bytes = new Uint8Array(hex.length / 2)
  for (let i = 0; i < hex.length; i += 2) {
    bytes[i / 2] = parseInt(hex.substr(i, 2), 16)
  }
  return bytes
}
```

This approach leads to:
- **Inconsistent Error Handling** - Some operations throw, others return null
- **Type Unsafety** - No compile-time guarantees about encoding validity
- **Scattered Functions** - Encoding utilities spread across different modules
- **Runtime Failures** - Silent failures or unexpected exceptions

### The Encoding Solution

Effect's Encoding module provides a unified, type-safe API for all common encoding operations with consistent error handling using Either types.

```typescript
import { Encoding, Either, Effect } from "effect"

// Clean, composable encoding pipeline
const processApiToken = (tokenData: string) => Effect.gen(function* () {
  const encoded = Encoding.encodeBase64(tokenData)
  
  const urlSafe = yield* Effect.fromEither(
    Encoding.encodeUriComponent(encoded)
  )
  
  const checksum = Encoding.encodeHex(tokenData)
  
  return { token: urlSafe, checksum }
})

// Type-safe decoding with explicit error handling
const decodeUserData = (base64Data: string) => Effect.gen(function* () {
  const decodedBytes = yield* Effect.fromEither(
    Encoding.decodeBase64(base64Data)
  )
  
  const jsonString = yield* Effect.fromEither(
    Encoding.decodeBase64String(base64Data)
  )
  
  const userData = yield* Effect.try({
    try: () => JSON.parse(jsonString),
    catch: (error) => new Error(`Invalid JSON: ${error}`)
  })
  
  return userData
})

// Unified hex operations with consistent error handling
const processHexData = (hexString: string) => Effect.gen(function* () {
  const bytes = yield* Effect.fromEither(
    Encoding.decodeHex(hexString)
  )
  
  const backToHex = Encoding.encodeHex(bytes)
  const asBase64 = Encoding.encodeBase64(bytes)
  
  return { original: hexString, hex: backToHex, base64: asBase64 }
})
```

### Key Concepts

**Unified API**: All encoding operations follow the same patterns - encode functions return strings directly, decode functions return Either types for error handling.

**Type Safety**: Decode operations return Either<Result, DecodeException> making error handling explicit and type-safe.

**Composability**: All functions work seamlessly with Effect's composition operators and can be used in pipelines.

**Format Support**: Comprehensive support for base64 (RFC4648), base64url, hex, and URI component encoding.

## Basic Usage Patterns

### Pattern 1: Base64 Operations

```typescript
import { Encoding, Effect } from "effect"

// Encoding strings and bytes to base64
const encodeToBase64 = (data: string | Uint8Array) => {
  return Encoding.encodeBase64(data)
}

// Decoding base64 to bytes
const decodeFromBase64 = (base64String: string) => Effect.gen(function* () {
  const bytes = yield* Effect.fromEither(
    Encoding.decodeBase64(base64String)
  )
  return bytes
})

// Decoding base64 directly to string
const decodeBase64ToString = (base64String: string) => Effect.gen(function* () {
  const text = yield* Effect.fromEither(
    Encoding.decodeBase64String(base64String)
  )
  return text
})

// Example usage
const basicBase64Example = Effect.gen(function* () {
  const original = "Hello, World!"
  const encoded = encodeToBase64(original)
  console.log(`Encoded: ${encoded}`) // "SGVsbG8sIFdvcmxkIQ=="
  
  const decoded = yield* decodeBase64ToString(encoded)
  console.log(`Decoded: ${decoded}`) // "Hello, World!"
  
  return { original, encoded, decoded }
})
```

### Pattern 2: URL-Safe Base64

```typescript
// URL-safe base64 for web applications
const urlSafeBase64Operations = Effect.gen(function* () {
  const data = "secure-token-data?param=value"
  
  // URL-safe encoding (no padding, uses - and _ instead of + and /)
  const urlEncoded = Encoding.encodeBase64Url(data)
  console.log(`URL-safe: ${urlEncoded}`)
  
  // Decoding back to string
  const decoded = yield* Effect.fromEither(
    Encoding.decodeBase64UrlString(urlEncoded)
  )
  
  return { original: data, urlEncoded, decoded }
})
```

### Pattern 3: Hex Operations

```typescript
// Hex encoding for cryptographic operations
const hexOperations = Effect.gen(function* () {
  const data = "Hello, Crypto!"
  const bytes = new TextEncoder().encode(data)
  
  // Encode to hex
  const hexString = Encoding.encodeHex(data)
  const hexFromBytes = Encoding.encodeHex(bytes)
  console.log(`Hex: ${hexString}`) // "48656c6c6f2c2043727970746f21"
  
  // Decode from hex
  const decodedBytes = yield* Effect.fromEither(
    Encoding.decodeHex(hexString)
  )
  const decodedString = yield* Effect.fromEither(
    Encoding.decodeHexString(hexString)
  )
  
  return { 
    original: data, 
    hex: hexString, 
    decodedBytes, 
    decodedString 
  }
})
```

### Pattern 4: URI Component Encoding

```typescript
// Safe URL parameter encoding
const uriComponentOperations = Effect.gen(function* () {
  const searchQuery = "user search: special chars & symbols!"
  
  // Encode for URL usage
  const encoded = yield* Effect.fromEither(
    Encoding.encodeUriComponent(searchQuery)
  )
  console.log(`Encoded: ${encoded}`)
  
  // Decode back
  const decoded = yield* Effect.fromEither(
    Encoding.decodeUriComponent(encoded)
  )
  
  return { original: searchQuery, encoded, decoded }
})
```

## Real-World Examples

### Example 1: JWT Token Processing

JWT tokens require base64url encoding for headers and payloads, making this a perfect real-world application.

```typescript
import { Encoding, Effect, Either } from "effect"

interface JWTHeader {
  alg: string
  typ: string
}

interface JWTPayload {
  sub: string
  name: string
  iat: number
  exp: number
}

class JWTError extends Error {
  readonly _tag = "JWTError"
  constructor(message: string, public readonly cause?: unknown) {
    super(message)
  }
}

const createJWT = (header: JWTHeader, payload: JWTPayload, secret: string) => 
  Effect.gen(function* () {
    // Encode header
    const headerJson = JSON.stringify(header)
    const encodedHeader = Encoding.encodeBase64Url(headerJson)
    
    // Encode payload
    const payloadJson = JSON.stringify(payload)
    const encodedPayload = Encoding.encodeBase64Url(payloadJson)
    
    // Create signature (simplified - normally use proper crypto)
    const signatureData = `${encodedHeader}.${encodedPayload}.${secret}`
    const signature = Encoding.encodeBase64Url(signatureData)
    
    return `${encodedHeader}.${encodedPayload}.${signature}`
  })

const parseJWT = (token: string) => Effect.gen(function* () {
  const parts = token.split('.')
  
  if (parts.length !== 3) {
    return yield* Effect.fail(new JWTError('Invalid JWT format'))
  }
  
  const [headerPart, payloadPart, signaturePart] = parts
  
  // Decode header
  const headerJson = yield* Effect.fromEither(
    Encoding.decodeBase64UrlString(headerPart)
  ).pipe(
    Effect.mapError(error => new JWTError('Invalid header encoding', error))
  )
  
  const header = yield* Effect.try({
    try: () => JSON.parse(headerJson) as JWTHeader,
    catch: (error) => new JWTError('Invalid header JSON', error)
  })
  
  // Decode payload
  const payloadJson = yield* Effect.fromEither(
    Encoding.decodeBase64UrlString(payloadPart)
  ).pipe(
    Effect.mapError(error => new JWTError('Invalid payload encoding', error))
  )
  
  const payload = yield* Effect.try({
    try: () => JSON.parse(payloadJson) as JWTPayload,
    catch: (error) => new JWTError('Invalid payload JSON', error)
  })
  
  return { header, payload, signature: signaturePart }
})

// Usage example
const jwtExample = Effect.gen(function* () {
  const header: JWTHeader = { alg: "HS256", typ: "JWT" }
  const payload: JWTPayload = {
    sub: "1234567890",
    name: "John Doe",
    iat: Math.floor(Date.now() / 1000),
    exp: Math.floor(Date.now() / 1000) + 3600
  }
  
  const token = yield* createJWT(header, payload, "secret-key")
  console.log(`JWT: ${token}`)
  
  const parsed = yield* parseJWT(token)
  console.log(`Parsed payload:`, parsed.payload)
  
  return { token, parsed }
})
```

### Example 2: Data URL Generation for Images

Data URLs are commonly used for embedding images directly in HTML or CSS.

```typescript
import { Encoding, Effect } from "effect"

interface DataURL {
  mimeType: string
  base64Data: string
  url: string
}

class ImageProcessingError extends Error {
  readonly _tag = "ImageProcessingError"
}

const createDataURL = (imageBytes: Uint8Array, mimeType: string) => Effect.gen(function* () {
  const base64Data = Encoding.encodeBase64(imageBytes)
  const url = `data:${mimeType};base64,${base64Data}`
  
  return { mimeType, base64Data, url }
})

const parseDataURL = (dataUrl: string) => Effect.gen(function* () {
  const dataPrefix = "data:"
  const base64Prefix = ";base64,"
  
  if (!dataUrl.startsWith(dataPrefix)) {
    return yield* Effect.fail(
      new ImageProcessingError('Invalid data URL format')
    )
  }
  
  const mimeEndIndex = dataUrl.indexOf(base64Prefix)
  if (mimeEndIndex === -1) {
    return yield* Effect.fail(
      new ImageProcessingError('Missing base64 prefix')
    )
  }
  
  const mimeType = dataUrl.slice(dataPrefix.length, mimeEndIndex)
  const base64Data = dataUrl.slice(mimeEndIndex + base64Prefix.length)
  
  const imageBytes = yield* Effect.fromEither(
    Encoding.decodeBase64(base64Data)
  ).pipe(
    Effect.mapError(error => new ImageProcessingError(`Invalid base64: ${error.message}`))
  )
  
  return { mimeType, base64Data, imageBytes, originalUrl: dataUrl }
})

const processImageDataURL = (imageBytes: Uint8Array, mimeType: string) => 
  Effect.gen(function* () {
    // Create data URL
    const dataUrl = yield* createDataURL(imageBytes, mimeType)
    console.log(`Created data URL: ${dataUrl.url.slice(0, 50)}...`)
    
    // Parse it back
    const parsed = yield* parseDataURL(dataUrl.url)
    console.log(`Parsed mime type: ${parsed.mimeType}`)
    console.log(`Original size: ${imageBytes.length}, Parsed size: ${parsed.imageBytes.length}`)
    
    // Verify data integrity
    const isValid = imageBytes.every((byte, index) => byte === parsed.imageBytes[index])
    
    return { dataUrl, parsed, isValid }
  })

// Usage with mock image data
const imageExample = Effect.gen(function* () {
  // Mock PNG header bytes
  const mockImageBytes = new Uint8Array([
    0x89, 0x50, 0x4E, 0x47, 0x0D, 0x0A, 0x1A, 0x0A, // PNG signature
    0x00, 0x00, 0x00, 0x0D, 0x49, 0x48, 0x44, 0x52  // IHDR chunk start
  ])
  
  const result = yield* processImageDataURL(mockImageBytes, "image/png")
  return result
})
```

### Example 3: API Response Transformation Pipeline

Many APIs return data in various encoded formats that need transformation.

```typescript
import { Encoding, Effect, Array as Arr } from "effect"

interface APIResponse {
  id: string
  data: string // base64 encoded
  checksum: string // hex encoded
  metadata: string // URI encoded JSON
}

interface ProcessedData {
  id: string
  content: Uint8Array
  contentText: string
  checksum: Uint8Array
  metadata: Record<string, unknown>
  isValid: boolean
}

class APIProcessingError extends Error {
  readonly _tag = "APIProcessingError"
}

const processAPIResponse = (response: APIResponse) => Effect.gen(function* () {
  // Decode base64 data
  const contentBytes = yield* Effect.fromEither(
    Encoding.decodeBase64(response.data)
  ).pipe(
    Effect.mapError(error => 
      new APIProcessingError(`Failed to decode data: ${error.message}`)
    )
  )
  
  const contentText = yield* Effect.fromEither(
    Encoding.decodeBase64String(response.data)
  ).pipe(
    Effect.mapError(error => 
      new APIProcessingError(`Failed to decode data as text: ${error.message}`)
    )
  )
  
  // Decode hex checksum
  const checksumBytes = yield* Effect.fromEither(
    Encoding.decodeHex(response.checksum)
  ).pipe(
    Effect.mapError(error => 
      new APIProcessingError(`Failed to decode checksum: ${error.message}`)
    )
  )
  
  // Decode URI component metadata
  const metadataJson = yield* Effect.fromEither(
    Encoding.decodeUriComponent(response.metadata)
  ).pipe(
    Effect.mapError(error => 
      new APIProcessingError(`Failed to decode metadata: ${error.message}`)
    )
  )
  
  const metadata = yield* Effect.try({
    try: () => JSON.parse(metadataJson) as Record<string, unknown>,
    catch: (error) => new APIProcessingError(`Invalid metadata JSON: ${error}`)
  })
  
  // Verify checksum (simplified)
  const computedChecksum = Encoding.encodeHex(contentBytes)
  const isValid = computedChecksum === response.checksum
  
  return {
    id: response.id,
    content: contentBytes,
    contentText,
    checksum: checksumBytes,
    metadata,
    isValid
  } satisfies ProcessedData
})

// Batch processing multiple API responses
const processBatchAPIResponses = (responses: APIResponse[]) => 
  Effect.gen(function* () {
    const results = yield* Effect.all(
      Arr.map(responses, processAPIResponse),
      { concurrency: 3 } // Process up to 3 concurrently
    )
    
    const valid = Arr.filter(results, result => result.isValid)
    const invalid = Arr.filter(results, result => !result.isValid)
    
    return { 
      total: results.length, 
      valid: valid.length, 
      invalid: invalid.length,
      results 
    }
  })

// Usage example
const apiProcessingExample = Effect.gen(function* () {
  const mockResponses: APIResponse[] = [
    {
      id: "1",
      data: Encoding.encodeBase64("Hello, World!"),
      checksum: Encoding.encodeHex("Hello, World!"),
      metadata: yield* Effect.fromEither(
        Encoding.encodeUriComponent(JSON.stringify({ type: "text", priority: 1 }))
      )
    },
    {
      id: "2", 
      data: Encoding.encodeBase64("Another message"),
      checksum: Encoding.encodeHex("Another message"),
      metadata: yield* Effect.fromEither(
        Encoding.encodeUriComponent(JSON.stringify({ type: "message", priority: 2 }))
      )
    }
  ]
  
  const batchResult = yield* processBatchAPIResponses(mockResponses)
  console.log(`Processed ${batchResult.total} responses: ${batchResult.valid} valid, ${batchResult.invalid} invalid`)
  
  return batchResult
})
```

## Advanced Features Deep Dive

### Error Handling with DecodeException

The Encoding module uses specific exception types that carry detailed error information.

```typescript
import { Encoding, Effect, Either } from "effect"

const handleEncodingErrors = (input: string) => Effect.gen(function* () {
  const base64Result = Encoding.decodeBase64(input)
  
  if (Either.isLeft(base64Result)) {
    const error = base64Result.left
    console.log(`Decode failed for input: "${error.input}"`)
    if (error.message) {
      console.log(`Error message: ${error.message}`)
    }
    console.log(`Error type: ${error._tag}`)
    console.log(`Is DecodeException: ${Encoding.isDecodeException(error)}`)
  }
  
  return base64Result
})

// Custom error recovery strategies
const decodeWithFallback = (input: string, fallback: string) => 
  Effect.gen(function* () {
    const result = yield* Effect.fromEither(
      Encoding.decodeBase64String(input)
    ).pipe(
      Effect.catchAll(() => Effect.succeed(fallback))
    )
    
    return result
  })

// Error categorization
const categorizeEncodingError = (error: Encoding.DecodeException) => {
  if (error.input.length === 0) return "empty_input"
  if (error.input.length % 4 !== 0) return "invalid_length"
  if (!/^[A-Za-z0-9+/]*={0,2}$/.test(error.input)) return "invalid_characters"
  return "unknown_error"
}
```

### Advanced Base64 Operations

```typescript
// Working with different base64 variants
const base64Variants = Effect.gen(function* () {
  const data = "Test data with special chars: +/="
  
  // Standard base64 (RFC4648)
  const standard = Encoding.encodeBase64(data)
  console.log(`Standard base64: ${standard}`)
  
  // URL-safe base64 (no padding, different chars)
  const urlSafe = Encoding.encodeBase64Url(data)
  console.log(`URL-safe base64: ${urlSafe}`)
  
  // Demonstrate the differences
  const hasStandardChars = /[+/=]/.test(standard)
  const hasUrlSafeChars = /[-_]/.test(urlSafe)
  const hasPadding = standard.endsWith('=')
  const hasNoPadding = !urlSafe.endsWith('=')
  
  return {
    data,
    standard,
    urlSafe,
    differences: {
      hasStandardChars,
      hasUrlSafeChars,
      hasPadding,
      hasNoPadding
    }
  }
})

// Large data handling
const processLargeData = (largeData: Uint8Array) => Effect.gen(function* () {
  console.log(`Processing ${largeData.length} bytes`)
  
  // Chunk processing for memory efficiency
  const chunkSize = 1024 * 1024 // 1MB chunks
  const chunks: string[] = []
  
  for (let i = 0; i < largeData.length; i += chunkSize) {
    const chunk = largeData.slice(i, i + chunkSize)
    const encoded = Encoding.encodeBase64(chunk)
    chunks.push(encoded)
  }
  
  const combined = chunks.join('')
  console.log(`Encoded to ${combined.length} base64 characters`)
  
  return combined
})
```

### Hex Operations for Cryptographic Use Cases

```typescript
// Cryptographic hex operations helper
const cryptoHexOperations = Effect.gen(function* () {
  const data = "Sensitive data"
  const key = "secret-key"
  
  // Simulate hash (normally use proper crypto library)
  const hashInput = `${data}:${key}`
  const hashHex = Encoding.encodeHex(hashInput)
  
  // Convert between hex and other formats
  const hashBytes = yield* Effect.fromEither(
    Encoding.decodeHex(hashHex)
  )
  
  const hashBase64 = Encoding.encodeBase64(hashBytes)
  
  return {
    originalData: data,
    hashHex,
    hashBase64,
    hashBytes
  }
})

// Hex validation and normalization
const normalizeHexString = (hex: string) => Effect.gen(function* () {
  // Remove common prefixes and whitespace
  const cleaned = hex
    .replace(/^0x/i, '') // Remove 0x prefix
    .replace(/\s+/g, '') // Remove whitespace
    .toLowerCase() // Normalize case
  
  // Validate and decode
  const bytes = yield* Effect.fromEither(
    Encoding.decodeHex(cleaned)
  )
  
  // Re-encode to get canonical form
  const canonical = Encoding.encodeHex(bytes)
  
  return {
    original: hex,
    cleaned,
    canonical,
    bytes
  }
})
```

## Practical Patterns & Best Practices

### Pattern 1: Safe Encoding Pipeline

Create reusable pipelines that handle encoding transformations safely.

```typescript
import { Encoding, Effect, Either, Array as Arr } from "effect"

const createEncodingPipeline = <T extends string | Uint8Array>(
  operations: Array<(input: T) => Either.Either<unknown, unknown>>
) => (input: T) => Effect.gen(function* () {
  let current: unknown = input
  const results: unknown[] = []
  
  for (const operation of operations) {
    const result = operation(current as T)
    if (Either.isLeft(result)) {
      return yield* Effect.fail(result.left)
    }
    current = result.right
    results.push(current)
  }
  
  return { final: current, intermediate: results }
})

// Usage example
const dataTransformPipeline = createEncodingPipeline([
  (input: string) => Either.right(Encoding.encodeBase64(input)),
  (input: string) => Encoding.encodeUriComponent(input),
  (input: string) => Either.right(input.toUpperCase())
])

const pipelineExample = Effect.gen(function* () {
  const result = yield* dataTransformPipeline("Hello, World!")
  return result
})
```

### Pattern 2: Format Detection and Auto-Decoding

Automatically detect and decode different encoding formats.

```typescript
interface EncodingDetectionResult {
  format: 'base64' | 'base64url' | 'hex' | 'uri' | 'unknown'
  decoded: string | Uint8Array
  confidence: number
}

const detectAndDecode = (input: string): Effect.Effect<EncodingDetectionResult, Error> =>
  Effect.gen(function* () {
    // Try base64 first (most common)
    const base64Result = Encoding.decodeBase64String(input)
    if (Either.isRight(base64Result)) {
      return {
        format: 'base64' as const,
        decoded: base64Result.right,
        confidence: 0.9
      }
    }
    
    // Try base64url
    const base64UrlResult = Encoding.decodeBase64UrlString(input)
    if (Either.isRight(base64UrlResult)) {
      return {
        format: 'base64url' as const,
        decoded: base64UrlResult.right,
        confidence: 0.85
      }
    }
    
    // Try hex
    const hexResult = Encoding.decodeHexString(input)
    if (Either.isRight(hexResult)) {
      return {
        format: 'hex' as const,
        decoded: hexResult.right,
        confidence: 0.8
      }
    }
    
    // Try URI component
    const uriResult = Encoding.decodeUriComponent(input)
    if (Either.isRight(uriResult)) {
      return {
        format: 'uri' as const,
        decoded: uriResult.right,
        confidence: 0.7
      }
    }
    
    return {
      format: 'unknown' as const,
      decoded: input,
      confidence: 0.1
    }
  })

// Batch detection
const detectEncodingFormats = (inputs: string[]) => Effect.gen(function* () {
  const results = yield* Effect.all(
    Arr.map(inputs, detectAndDecode)
  )
  
  const byFormat = Arr.groupBy(results, result => result.format)
  
  return { results, byFormat }
})
```

### Pattern 3: Streaming Encoding Operations

Handle large data streams efficiently.

```typescript
// Stream-based encoding for large files
const createEncodingStream = (chunkSize = 8192) => ({
  encodeStream: function* (data: Uint8Array) {
    for (let i = 0; i < data.length; i += chunkSize) {
      const chunk = data.slice(i, i + chunkSize)
      yield Encoding.encodeBase64(chunk)
    }
  },
  
  decodeStream: function* (base64Chunks: string[]) {
    for (const chunk of base64Chunks) {
      const result = Encoding.decodeBase64(chunk)
      if (Either.isRight(result)) {
        yield result.right
      }
    }
  }
})

const streamingExample = Effect.gen(function* () {
  const largeData = new Uint8Array(100000).fill(65) // Large array of 'A's
  const encoder = createEncodingStream(1024)
  
  // Encode in chunks
  const encodedChunks = Array.from(encoder.encodeStream(largeData))
  console.log(`Encoded ${largeData.length} bytes into ${encodedChunks.length} chunks`)
  
  // Decode back
  const decodedChunks = Array.from(encoder.decodeStream(encodedChunks))
  const totalDecoded = decodedChunks.reduce((acc, chunk) => acc + chunk.length, 0)
  
  return { 
    originalSize: largeData.length, 
    encodedChunks: encodedChunks.length,
    decodedSize: totalDecoded 
  }
})
```

### Pattern 4: Encoding Validation and Sanitization

```typescript
// Comprehensive validation and sanitization
const validateAndSanitizeEncoding = (input: string, expectedFormat: 'base64' | 'hex' | 'uri') =>
  Effect.gen(function* () {
    // Sanitize input
    const sanitized = input.trim()
    
    // Format-specific validation
    switch (expectedFormat) {
      case 'base64': {
        if (!/^[A-Za-z0-9+/]*={0,2}$/.test(sanitized)) {
          return yield* Effect.fail(new Error('Invalid base64 format'))
        }
        const decoded = yield* Effect.fromEither(
          Encoding.decodeBase64String(sanitized)
        )
        return { format: 'base64' as const, sanitized, decoded }
      }
      
      case 'hex': {
        if (!/^[0-9a-fA-F]*$/.test(sanitized)) {
          return yield* Effect.fail(new Error('Invalid hex format'))
        }
        if (sanitized.length % 2 !== 0) {
          return yield* Effect.fail(new Error('Hex string must have even length'))
        }
        const decoded = yield* Effect.fromEither(
          Encoding.decodeHexString(sanitized)
        )
        return { format: 'hex' as const, sanitized, decoded }
      }
      
      case 'uri': {
        const decoded = yield* Effect.fromEither(
          Encoding.decodeUriComponent(sanitized)
        )
        return { format: 'uri' as const, sanitized, decoded }
      }
      
      default:
        return yield* Effect.fail(new Error(`Unsupported format: ${expectedFormat}`))
    }
  })
```

## Integration Examples

### Integration with HTTP Clients

Show how Encoding integrates with HTTP operations for API communication.

```typescript
import { Encoding, Effect, HttpClient } from "effect"

interface APICredentials {
  username: string
  password: string
}

// Basic auth header creation
const createBasicAuthHeader = (credentials: APICredentials) => Effect.gen(function* () {
  const authString = `${credentials.username}:${credentials.password}`
  const encodedAuth = Encoding.encodeBase64(authString)
  
  return {
    Authorization: `Basic ${encodedAuth}`
  }
})

// File upload with base64 encoding
const uploadBase64File = (fileData: Uint8Array, filename: string, mimeType: string) =>
  Effect.gen(function* () {
    const base64Data = Encoding.encodeBase64(fileData)
    
    const payload = {
      filename,
      mimeType,
      data: base64Data,
      size: fileData.length
    }
    
    // Simulate HTTP client usage
    return yield* Effect.succeed({
      status: 'uploaded',
      payload
    })
  })

// API response processing
const processEncodedAPIResponse = (response: {
  data: string // base64 encoded
  signature: string // hex encoded
}) => Effect.gen(function* () {
  const data = yield* Effect.fromEither(
    Encoding.decodeBase64String(response.data)
  )
  
  const signatureBytes = yield* Effect.fromEither(
    Encoding.decodeHex(response.signature)
  )
  
  return { data, signatureBytes }
})
```

### Integration with File System Operations

```typescript
// Mock FileSystem operations for demonstration
const mockFileSystem = {
  readFile: (path: string) => Effect.succeed(new TextEncoder().encode(`Content of ${path}`)),
  writeFile: (path: string, content: Uint8Array) => Effect.succeed(void 0)
}

const encodeFileToBase64 = (filePath: string) => Effect.gen(function* () {
  const fileContent = yield* mockFileSystem.readFile(filePath)
  const base64Content = Encoding.encodeBase64(fileContent)
  
  return {
    filePath,
    size: fileContent.length,
    base64: base64Content,
    base64Size: base64Content.length
  }
})

const decodeAndSaveFile = (base64Content: string, outputPath: string) => 
  Effect.gen(function* () {
    const fileContent = yield* Effect.fromEither(
      Encoding.decodeBase64(base64Content)
    )
    
    yield* mockFileSystem.writeFile(outputPath, fileContent)
    
    return {
      outputPath,
      size: fileContent.length
    }
  })

// Batch file processing
const processFilesBatch = (files: Array<{ path: string, base64?: string }>) =>
  Effect.gen(function* () {
    const results = yield* Effect.all(
      files.map(file => 
        file.base64 
          ? decodeAndSaveFile(file.base64, file.path + '.decoded')
          : encodeFileToBase64(file.path)
      )
    )
    
    return results
  })
```

### Integration with Schema Validation

```typescript
// Mock Schema operations for demonstration
const mockSchema = {
  decode: <T>(value: unknown): Effect.Effect<T, Error> => 
    Effect.succeed(value as T),
  encode: <T>(value: T): Effect.Effect<unknown, Error> => 
    Effect.succeed(value)
}

interface EncodedDocument {
  id: string
  title: string
  content: string // base64 encoded
  checksum: string // hex encoded
  metadata: string // URI encoded JSON
}

const decodeDocument = (doc: EncodedDocument) => Effect.gen(function* () {
  // Decode content
  const content = yield* Effect.fromEither(
    Encoding.decodeBase64String(doc.content)
  )
  
  // Decode checksum
  const checksumBytes = yield* Effect.fromEither(
    Encoding.decodeHex(doc.checksum)
  )
  
  // Decode metadata
  const metadataJson = yield* Effect.fromEither(
    Encoding.decodeUriComponent(doc.metadata)
  )
  
  const metadata = yield* Effect.try({
    try: () => JSON.parse(metadataJson),
    catch: (error) => new Error(`Invalid metadata: ${error}`)
  })
  
  // Validate schema
  const validatedDoc = yield* mockSchema.decode({
    id: doc.id,
    title: doc.title,
    content,
    checksum: checksumBytes,
    metadata
  })
  
  return validatedDoc
})

const encodeDocument = (doc: {
  id: string
  title: string
  content: string
  metadata: Record<string, unknown>
}) => Effect.gen(function* () {
  // Encode content
  const encodedContent = Encoding.encodeBase64(doc.content)
  
  // Generate checksum
  const checksum = Encoding.encodeHex(doc.content)
  
  // Encode metadata
  const metadataJson = JSON.stringify(doc.metadata)
  const encodedMetadata = yield* Effect.fromEither(
    Encoding.encodeUriComponent(metadataJson)
  )
  
  const encodedDoc: EncodedDocument = {
    id: doc.id,
    title: doc.title,
    content: encodedContent,
    checksum,
    metadata: encodedMetadata
  }
  
  // Validate encoded document
  return yield* mockSchema.encode(encodedDoc)
})
```

## Conclusion

The Encoding module provides comprehensive, type-safe encoding and decoding operations for base64, hex, and URI components. It eliminates the common pitfalls of traditional JavaScript encoding with consistent error handling, composable operations, and excellent TypeScript integration.

Key benefits:
- **Type Safety**: All decode operations return Either types, making error handling explicit
- **Consistency**: Unified API across all encoding formats with predictable patterns  
- **Composability**: Seamless integration with Effect's composition operators and pipelines
- **Error Handling**: Detailed exception types with meaningful error messages
- **Performance**: Efficient implementations suitable for both small data and streaming operations

Use the Encoding module when you need reliable, type-safe encoding operations in your Effect applications, especially for API integrations, data serialization, cryptographic operations, and web protocols.