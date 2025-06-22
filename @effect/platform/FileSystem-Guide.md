# FileSystem: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem FileSystem Solves

Working with file systems in Node.js traditionally involves callbacks or promises with inconsistent error handling and no built-in resource management:

```typescript
// Traditional approach - callback hell and manual resource cleanup
import * as fs from 'fs'
import * as path from 'path'

// Reading a file with callbacks
fs.readFile('config.json', 'utf8', (err, data) => {
  if (err) {
    console.error('Error reading file:', err)
    return
  }
  
  try {
    const config = JSON.parse(data)
    // Process config...
  } catch (parseErr) {
    console.error('Error parsing JSON:', parseErr)
  }
})

// Watching files with manual cleanup
const watcher = fs.watch('logs/', (eventType, filename) => {
  console.log(`File ${filename} changed: ${eventType}`)
})

// Manual cleanup required
process.on('SIGINT', () => {
  watcher.close()
  process.exit()
})

// Stream handling with error-prone patterns
const readStream = fs.createReadStream('large-file.csv')
const writeStream = fs.createWriteStream('output.csv')

readStream.pipe(writeStream)
readStream.on('error', (err) => console.error('Read error:', err))
writeStream.on('error', (err) => console.error('Write error:', err))
```

This approach leads to:
- **Callback Hell** - Nested callbacks make code hard to read and maintain
- **Inconsistent Error Handling** - Easy to miss error cases or handle them inconsistently
- **Manual Resource Management** - File handles, watchers, and streams need manual cleanup
- **No Type Safety** - File paths and options are stringly-typed
- **Platform Differences** - Different APIs for Node.js, Bun, and browsers

### The FileSystem Solution

Effect's FileSystem module provides a unified, type-safe, and composable API for file operations:

```typescript
import { FileSystem } from "@effect/platform"
import { NodeContext, NodeRuntime } from "@effect/platform-node"
import { Effect } from "effect"

// Clean, composable file operations with automatic resource management
const program = Effect.gen(function* () {
  const fs = yield* FileSystem.FileSystem
  
  // Type-safe file operations with proper error handling
  const config = yield* fs.readFileString("config.json").pipe(
    Effect.flatMap(content => Effect.try(() => JSON.parse(content))),
    Effect.catchTag("SystemError", () => 
      Effect.succeed({ defaultConfig: true })
    )
  )
  
  return config
})

// Run with proper context
NodeRuntime.runMain(Effect.provide(program, NodeContext.layer))
```

### Key Concepts

**FileSystem Service**: The main service providing all file system operations, accessible via the `FileSystem.FileSystem` tag.

**PlatformError**: Type-safe errors that represent various file system failures (SystemError, BadArgument, etc.)

**Scoped Resources**: File handles and watchers are automatically managed within Effect's scope system

**Cross-Platform**: Same API works across Node.js, Bun, and browsers (with limitations)

## Basic Usage Patterns

### Reading and Writing Files

```typescript
import { FileSystem } from "@effect/platform"
import { NodeContext, NodeRuntime } from "@effect/platform-node"
import { Effect, Console } from "effect"

const readWriteExample = Effect.gen(function* () {
  const fs = yield* FileSystem.FileSystem
  
  // Read a text file
  const content = yield* fs.readFileString("input.txt", "utf8")
  yield* Console.log(`Read ${content.length} characters`)
  
  // Write a text file
  yield* fs.writeFileString("output.txt", content.toUpperCase())
  
  // Read binary data
  const imageData = yield* fs.readFile("image.png")
  yield* Console.log(`Image size: ${imageData.length} bytes`)
  
  // Write binary data
  yield* fs.writeFile("copy.png", imageData)
})

NodeRuntime.runMain(Effect.provide(readWriteExample, NodeContext.layer))
```

### Directory Operations

```typescript
import { FileSystem } from "@effect/platform"
import { NodeContext, NodeRuntime } from "@effect/platform-node"
import { Effect, Console } from "effect"

const directoryOps = Effect.gen(function* () {
  const fs = yield* FileSystem.FileSystem
  
  // Create directory (with parents if needed)
  yield* fs.makeDirectory("logs/2024/01", { recursive: true })
  
  // Check if path exists
  const exists = yield* fs.exists("logs/2024/01")
  yield* Console.log(`Directory exists: ${exists}`)
  
  // Read directory contents
  const files = yield* fs.readDirectory("logs")
  yield* Console.log("Directory contents:", files)
  
  // Get file information
  const stat = yield* fs.stat("logs/2024/01")
  yield* Console.log(`Is directory: ${stat.type === "Directory"}`)
})

NodeRuntime.runMain(Effect.provide(directoryOps, NodeContext.layer))
```

### File Watching

```typescript
import { FileSystem } from "@effect/platform"
import { NodeContext, NodeRuntime } from "@effect/platform-node"
import { Effect, Stream, Console, Scope } from "effect"

const watchExample = Effect.gen(function* () {
  const fs = yield* FileSystem.FileSystem
  
  // Watch a directory for changes
  const watcher = yield* fs.watch("./config")
  
  // Process watch events
  yield* Stream.fromAsyncIterable(watcher, identity).pipe(
    Stream.tap(event => 
      Console.log(`File ${event.path} ${event.type}`)
    ),
    Stream.take(10), // Stop after 10 events
    Stream.runDrain
  )
}).pipe(
  Effect.scoped, // Watcher is automatically cleaned up
  Effect.provide(NodeContext.layer)
)

NodeRuntime.runMain(watchExample)
```

## Real-World Examples

### Example 1: JSON Configuration Manager

A robust configuration manager that handles file reading, parsing, validation, and hot-reloading:

```typescript
import { FileSystem } from "@effect/platform"
import { NodeContext, NodeRuntime } from "@effect/platform-node"
import { Effect, Stream, Console, Ref, Schema, Option } from "effect"

// Configuration schema
const ConfigSchema = Schema.Struct({
  port: Schema.Number,
  database: Schema.Struct({
    host: Schema.String,
    port: Schema.Number,
    name: Schema.String
  }),
  features: Schema.Record(Schema.String, Schema.Boolean)
})

type Config = Schema.Schema.Type<typeof ConfigSchema>

// Configuration manager with hot-reload support
const createConfigManager = (configPath: string) => Effect.gen(function* () {
  const fs = yield* FileSystem.FileSystem
  const configRef = yield* Ref.make<Option.Option<Config>>(Option.none())
  
  // Load configuration
  const loadConfig = Effect.gen(function* () {
    const content = yield* fs.readFileString(configPath)
    const json = yield* Effect.try({
      try: () => JSON.parse(content),
      catch: (e) => new Error(`Invalid JSON: ${e}`)
    })
    const config = yield* Schema.decodeUnknown(ConfigSchema)(json)
    yield* Ref.set(configRef, Option.some(config))
    yield* Console.log("Configuration loaded successfully")
    return config
  }).pipe(
    Effect.catchAll(error => 
      Console.error(`Failed to load config: ${error}`).pipe(
        Effect.zipRight(Effect.fail(error))
      )
    )
  )
  
  // Initial load
  yield* loadConfig
  
  // Watch for changes
  const watcher = yield* fs.watch(configPath)
  
  yield* Effect.forkScoped(
    Stream.fromAsyncIterable(watcher, identity).pipe(
      Stream.filter(event => event.type === "update"),
      Stream.debounce("100 millis"),
      Stream.tap(() => Console.log("Configuration file changed, reloading...")),
      Stream.mapEffect(() => loadConfig),
      Stream.catchAll(() => Stream.empty),
      Stream.runDrain
    )
  )
  
  // Return config accessor
  return {
    get: Ref.get(configRef).pipe(
      Effect.flatMap(Option.match({
        onNone: () => Effect.fail(new Error("Configuration not loaded")),
        onSome: Effect.succeed
      }))
    )
  }
})

// Usage
const program = Effect.gen(function* () {
  const configManager = yield* createConfigManager("./app-config.json")
  
  // Get current config
  const config = yield* configManager.get
  yield* Console.log(`Server will run on port ${config.port}`)
  
  // Simulate app running
  yield* Effect.sleep("30 seconds")
}).pipe(
  Effect.scoped,
  Effect.provide(NodeContext.layer)
)

NodeRuntime.runMain(program)
```

### Example 2: CSV File Processor with Streaming

Processing large CSV files efficiently using streams:

```typescript
import { FileSystem } from "@effect/platform"
import { NodeContext, NodeRuntime } from "@effect/platform-node"
import { Effect, Stream, Console, Chunk } from "effect"

// CSV row parser
const parseCSVLine = (line: string): string[] => {
  const result: string[] = []
  let current = ""
  let inQuotes = false
  
  for (let i = 0; i < line.length; i++) {
    const char = line[i]
    
    if (char === '"') {
      inQuotes = !inQuotes
    } else if (char === ',' && !inQuotes) {
      result.push(current.trim())
      current = ""
    } else {
      current += char
    }
  }
  
  result.push(current.trim())
  return result
}

// Process large CSV file
const processLargeCSV = (inputPath: string, outputPath: string) => Effect.gen(function* () {
  const fs = yield* FileSystem.FileSystem
  
  // Track processing stats
  let totalRows = 0
  let validRows = 0
  
  // Create read stream
  const inputStream = yield* fs.stream(inputPath)
  
  // Process CSV
  const processedStream = inputStream.pipe(
    Stream.decodeText("utf8"),
    Stream.splitLines,
    Stream.drop(1), // Skip header
    Stream.map(line => {
      totalRows++
      const columns = parseCSVLine(line)
      
      // Example: Filter and transform data
      if (columns.length >= 3 && parseFloat(columns[2]) > 100) {
        validRows++
        return Option.some({
          id: columns[0],
          name: columns[1],
          value: parseFloat(columns[2]) * 1.1 // Add 10% markup
        })
      }
      return Option.none()
    }),
    Stream.filterMap(identity),
    Stream.map(record => 
      `${record.id},${record.name},${record.value.toFixed(2)}\n`
    ),
    Stream.prepend(Stream.make("id,name,adjusted_value\n")),
    Stream.encodeText("utf8")
  )
  
  // Write to output file
  const sink = yield* fs.sink(outputPath)
  yield* Stream.run(processedStream, sink)
  
  yield* Console.log(`Processed ${totalRows} rows, wrote ${validRows} valid rows`)
}).pipe(
  Effect.scoped,
  Effect.withSpan("csv.process", {
    attributes: { 
      "csv.input": inputPath,
      "csv.output": outputPath
    }
  })
)

// Usage
const program = Effect.provide(
  processLargeCSV("./sales-data.csv", "./processed-sales.csv"),
  NodeContext.layer
)

NodeRuntime.runMain(program)
```

### Example 3: Log Rotation System

Implementing a production-ready log rotation system:

```typescript
import { FileSystem } from "@effect/platform"
import { NodeContext, NodeRuntime } from "@effect/platform-node"
import { Effect, Stream, Console, Queue, Scope, Schedule, Duration } from "effect"
import * as path from "path"

interface LogRotationConfig {
  directory: string
  baseFileName: string
  maxFileSize: number // in bytes
  maxFiles: number
  rotationInterval: Duration.Duration
}

const createLogRotationSystem = (config: LogRotationConfig) => Effect.gen(function* () {
  const fs = yield* FileSystem.FileSystem
  const scope = yield* Scope.Scope
  
  // Ensure log directory exists
  yield* fs.makeDirectory(config.directory, { recursive: true })
  
  // Current log file state
  let currentFile: string | null = null
  let currentSize = 0
  let fileHandle: FileSystem.File.Descriptor | null = null
  
  // Generate log file name with timestamp
  const generateFileName = () => {
    const timestamp = new Date().toISOString().replace(/[:.]/g, '-')
    return path.join(config.directory, `${config.baseFileName}-${timestamp}.log`)
  }
  
  // Rotate log file
  const rotate = Effect.gen(function* () {
    // Close current file if open
    if (fileHandle) {
      yield* Effect.promise(() => fileHandle!.close())
      fileHandle = null
    }
    
    // Clean up old files
    const files = yield* fs.readDirectory(config.directory)
    const logFiles = files
      .filter(f => f.name.startsWith(config.baseFileName))
      .sort((a, b) => b.name.localeCompare(a.name)) // newest first
    
    // Remove excess files
    if (logFiles.length >= config.maxFiles) {
      const toDelete = logFiles.slice(config.maxFiles - 1)
      yield* Effect.forEach(toDelete, file => 
        fs.remove(path.join(config.directory, file.name))
      )
    }
    
    // Create new file
    currentFile = generateFileName()
    currentSize = 0
    fileHandle = yield* fs.open(currentFile, { flag: "w" })
    
    yield* Console.log(`Rotated to new log file: ${currentFile}`)
  })
  
  // Write log entry
  const write = (message: string) => Effect.gen(function* () {
    const entry = `[${new Date().toISOString()}] ${message}\n`
    const buffer = new TextEncoder().encode(entry)
    
    // Check if rotation needed
    if (!fileHandle || currentSize + buffer.length > config.maxFileSize) {
      yield* rotate
    }
    
    // Write to file
    yield* Effect.promise(() => fileHandle!.write(buffer))
    currentSize += buffer.length
  })
  
  // Initial rotation
  yield* rotate
  
  // Schedule periodic rotation
  yield* Schedule.fixed(config.rotationInterval).pipe(
    Schedule.forever,
    Effect.schedule(rotate),
    Effect.forkIn(scope)
  )
  
  // Create log queue for async writes
  const logQueue = yield* Queue.unbounded<string>()
  
  // Process log queue
  yield* Stream.fromQueue(logQueue).pipe(
    Stream.tap(write),
    Stream.catchAll(error => {
      console.error("Log rotation error:", error)
      return Stream.empty
    }),
    Stream.runDrain,
    Effect.forkIn(scope)
  )
  
  return {
    log: (message: string) => Queue.offer(logQueue, message),
    rotate: rotate,
    close: Effect.sync(() => {
      if (fileHandle) {
        fileHandle.close()
      }
    })
  }
})

// Usage
const program = Effect.gen(function* () {
  const logger = yield* createLogRotationSystem({
    directory: "./logs",
    baseFileName: "app",
    maxFileSize: 1024 * 1024, // 1MB
    maxFiles: 5,
    rotationInterval: Duration.hours(1)
  })
  
  // Simulate application logging
  yield* Effect.forEach(
    Array.from({ length: 1000 }, (_, i) => i),
    (i) => Effect.gen(function* () {
      yield* logger.log(`Processing request ${i}`)
      yield* Effect.sleep("10 millis")
      
      if (Math.random() < 0.1) {
        yield* logger.log(`ERROR: Random error in request ${i}`)
      }
    }),
    { concurrency: 10 }
  )
  
  yield* Console.log("Logging simulation complete")
}).pipe(
  Effect.scoped,
  Effect.provide(NodeContext.layer)
)

NodeRuntime.runMain(program)
```

## Advanced Features Deep Dive

### Streaming Large Files

Efficiently process files that don't fit in memory:

#### Basic Streaming

```typescript
import { FileSystem } from "@effect/platform"
import { NodeContext, NodeRuntime } from "@effect/platform-node"
import { Effect, Stream, Console } from "effect"

const streamBasics = Effect.gen(function* () {
  const fs = yield* FileSystem.FileSystem
  
  // Read file as stream
  const stream = yield* fs.stream("large-file.txt")
  
  // Process stream
  yield* stream.pipe(
    Stream.decodeText("utf8"),
    Stream.splitLines,
    Stream.map(line => line.toUpperCase()),
    Stream.intersperse("\n"),
    Stream.encodeText("utf8"),
    Stream.run(yield* fs.sink("output.txt"))
  )
})
```

#### Real-World Streaming: Log Analysis

```typescript
interface LogEntry {
  timestamp: Date
  level: "INFO" | "WARN" | "ERROR"
  message: string
  metadata?: Record<string, unknown>
}

const analyzeServerLogs = (logPath: string) => Effect.gen(function* () {
  const fs = yield* FileSystem.FileSystem
  
  // Parse log line
  const parseLogLine = (line: string): Option.Option<LogEntry> => {
    const match = line.match(/^\[(\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2})\] \[(\w+)\] (.+)$/)
    if (!match) return Option.none()
    
    const [, timestamp, level, message] = match
    
    // Try to extract JSON metadata
    let metadata: Record<string, unknown> | undefined
    const jsonMatch = message.match(/\{.+\}$/)
    if (jsonMatch) {
      try {
        metadata = JSON.parse(jsonMatch[0])
      } catch {}
    }
    
    return Option.some({
      timestamp: new Date(timestamp),
      level: level as LogEntry["level"],
      message: message.replace(/\s*\{.+\}$/, ''),
      metadata
    })
  }
  
  // Analysis state
  const stats = {
    total: 0,
    errors: 0,
    warnings: 0,
    info: 0,
    errorPatterns: new Map<string, number>()
  }
  
  // Process log stream
  const logStream = yield* fs.stream(logPath)
  
  yield* logStream.pipe(
    Stream.decodeText("utf8"),
    Stream.splitLines,
    Stream.map(parseLogLine),
    Stream.filterMap(identity),
    Stream.tap(entry => Effect.sync(() => {
      stats.total++
      stats[entry.level.toLowerCase() as keyof typeof stats]++
      
      // Track error patterns
      if (entry.level === "ERROR") {
        const pattern = entry.message.split(':')[0].trim()
        stats.errorPatterns.set(
          pattern,
          (stats.errorPatterns.get(pattern) || 0) + 1
        )
      }
    })),
    // Optional: write errors to separate file
    Stream.filter(entry => entry.level === "ERROR"),
    Stream.map(entry => JSON.stringify(entry) + "\n"),
    Stream.encodeText("utf8"),
    Stream.run(yield* fs.sink("errors.log"))
  )
  
  // Generate report
  yield* Console.log("=== Log Analysis Report ===")
  yield* Console.log(`Total entries: ${stats.total}`)
  yield* Console.log(`Errors: ${stats.errors}`)
  yield* Console.log(`Warnings: ${stats.warnings}`)
  yield* Console.log(`Info: ${stats.info}`)
  yield* Console.log("\nTop Error Patterns:")
  
  const topErrors = Array.from(stats.errorPatterns.entries())
    .sort((a, b) => b[1] - a[1])
    .slice(0, 5)
  
  yield* Effect.forEach(topErrors, ([pattern, count]) =>
    Console.log(`  ${pattern}: ${count} occurrences`)
  )
}).pipe(Effect.scoped)
```

### File System Watching

Advanced file watching patterns for real-world applications:

#### Hot Module Reloading

```typescript
const createHotReloader = (watchDir: string) => Effect.gen(function* () {
  const fs = yield* FileSystem.FileSystem
  const moduleCache = new Map<string, unknown>()
  
  // Watch for file changes
  const watcher = yield* fs.watch(watchDir, { recursive: true })
  
  // Module loader
  const loadModule = (modulePath: string) => Effect.gen(function* () {
    const content = yield* fs.readFileString(modulePath)
    
    // Simple module evaluation (in practice, use proper module system)
    const moduleExports: any = {}
    const moduleFunc = new Function('exports', 'module', content)
    const module = { exports: moduleExports }
    moduleFunc(moduleExports, module)
    
    moduleCache.set(modulePath, module.exports)
    return module.exports
  }).pipe(
    Effect.catchAll(error => {
      Console.error(`Failed to load module ${modulePath}: ${error}`)
      return Effect.succeed(null)
    })
  )
  
  // Process file changes
  yield* Stream.fromAsyncIterable(watcher, identity).pipe(
    Stream.filter(event => 
      event.path.endsWith('.js') && 
      (event.type === "create" || event.type === "update")
    ),
    Stream.debounce("50 millis"),
    Stream.tap(event => Console.log(`Reloading module: ${event.path}`)),
    Stream.mapEffect(event => loadModule(event.path)),
    Stream.runDrain,
    Effect.forkScoped
  )
  
  return {
    getModule: (path: string) => Effect.sync(() => moduleCache.get(path)),
    loadModule
  }
})
```

### Temporary File Handling

Safe temporary file operations with automatic cleanup:

```typescript
const processWithTempFiles = Effect.gen(function* () {
  const fs = yield* FileSystem.FileSystem
  
  // Create temp directory with automatic cleanup
  const tempDir = yield* fs.makeTempDirectoryScoped({ prefix: "data-process-" })
  yield* Console.log(`Working in temp directory: ${tempDir}`)
  
  // Create temp files
  const inputTemp = path.join(tempDir, "input.json")
  const outputTemp = path.join(tempDir, "output.json")
  
  // Process data
  yield* fs.writeFileString(inputTemp, JSON.stringify({ data: "test" }))
  
  // Simulate processing
  const data = yield* fs.readFileString(inputTemp).pipe(
    Effect.map(JSON.parse),
    Effect.map(obj => ({ ...obj, processed: true }))
  )
  
  yield* fs.writeFileString(outputTemp, JSON.stringify(data, null, 2))
  
  // Read result
  const result = yield* fs.readFileString(outputTemp)
  
  return result
  // Temp directory and files are automatically cleaned up here
}).pipe(Effect.scoped)
```

## Practical Patterns & Best Practices

### Pattern 1: Safe File Operations with Retry

```typescript
const safeFileOperation = <A>(
  operation: Effect.Effect<A, PlatformError, FileSystem.FileSystem>
) => {
  return operation.pipe(
    Effect.retry(
      Schedule.exponential("100 millis").pipe(
        Schedule.intersect(Schedule.recurs(3))
      )
    ),
    Effect.catchTag("SystemError", (error) => {
      if (error.reason === "NotFound") {
        return Effect.fail(new Error("File not found"))
      }
      return Effect.fail(error)
    })
  )
}

// Usage
const readConfigSafely = (path: string) => Effect.gen(function* () {
  const fs = yield* FileSystem.FileSystem
  
  return yield* safeFileOperation(
    fs.readFileString(path).pipe(
      Effect.flatMap(content => 
        Effect.try(() => JSON.parse(content))
      )
    )
  )
})
```

### Pattern 2: Atomic File Writing

```typescript
const writeFileAtomic = (
  path: string,
  content: string | Uint8Array
) => Effect.gen(function* () {
  const fs = yield* FileSystem.FileSystem
  
  // Write to temporary file first
  const tempPath = `${path}.tmp.${Date.now()}`
  
  try {
    // Write content
    if (typeof content === "string") {
      yield* fs.writeFileString(tempPath, content)
    } else {
      yield* fs.writeFile(tempPath, content)
    }
    
    // Atomic rename
    yield* fs.rename(tempPath, path)
  } catch (error) {
    // Clean up temp file on error
    yield* fs.remove(tempPath).pipe(Effect.orElse(() => Effect.void))
    throw error
  }
})
```

### Pattern 3: Directory Tree Walker

```typescript
interface FileTreeNode {
  path: string
  name: string
  type: "file" | "directory"
  size?: number
  children?: FileTreeNode[]
}

const walkDirectoryTree = (
  rootPath: string,
  options?: {
    maxDepth?: number
    filter?: (path: string) => boolean
  }
): Effect.Effect<FileTreeNode, PlatformError, FileSystem.FileSystem> => {
  const { maxDepth = Infinity, filter = () => true } = options || {}
  
  const walk = (path: string, depth: number): Effect.Effect<FileTreeNode, PlatformError, FileSystem.FileSystem> => 
    Effect.gen(function* () {
      const fs = yield* FileSystem.FileSystem
      const stat = yield* fs.stat(path)
      const name = path.split('/').pop() || path
      
      if (stat.type === "File") {
        return {
          path,
          name,
          type: "file",
          size: stat.size
        }
      }
      
      const node: FileTreeNode = {
        path,
        name,
        type: "directory",
        children: []
      }
      
      if (depth < maxDepth) {
        const entries = yield* fs.readDirectory(path)
        
        node.children = yield* Effect.forEach(
          entries.filter(entry => filter(path + '/' + entry.name)),
          entry => walk(path + '/' + entry.name, depth + 1),
          { concurrency: 5 }
        )
      }
      
      return node
    })
  
  return walk(rootPath, 0)
}

// Usage
const analyzeProjectStructure = Effect.gen(function* () {
  const tree = yield* walkDirectoryTree("./src", {
    maxDepth: 3,
    filter: (path) => !path.includes('node_modules') && !path.includes('.git')
  })
  
  // Calculate total size
  const calculateSize = (node: FileTreeNode): number => {
    if (node.type === "file") return node.size || 0
    return node.children?.reduce((sum, child) => sum + calculateSize(child), 0) || 0
  }
  
  const totalSize = calculateSize(tree)
  yield* Console.log(`Total project size: ${(totalSize / 1024 / 1024).toFixed(2)} MB`)
  
  return tree
})
```

## Integration Examples

### Integration with Express.js for File Uploads

```typescript
import express from "express"
import multer from "multer"
import { FileSystem } from "@effect/platform"
import { NodeContext, NodeRuntime } from "@effect/platform-node"
import { Effect, Layer, ManagedRuntime } from "effect"

// Create Effect runtime for Express integration
const runtime = ManagedRuntime.make(NodeContext.layer)

// File upload handler
const handleFileUpload = (file: Express.Multer.File) => Effect.gen(function* () {
  const fs = yield* FileSystem.FileSystem
  
  // Validate file
  if (!file.mimetype.startsWith('image/')) {
    return yield* Effect.fail(new Error('Only images are allowed'))
  }
  
  // Create upload directory
  const uploadDir = './uploads/' + new Date().toISOString().split('T')[0]
  yield* fs.makeDirectory(uploadDir, { recursive: true })
  
  // Generate unique filename
  const ext = file.originalname.split('.').pop()
  const filename = `${Date.now()}-${Math.random().toString(36).substring(7)}.${ext}`
  const filepath = `${uploadDir}/${filename}`
  
  // Move file from temp to permanent location
  yield* fs.rename(file.path, filepath)
  
  // Generate thumbnail
  const thumbnailPath = filepath.replace(`.${ext}`, `-thumb.${ext}`)
  // In real app, use image processing library
  yield* fs.copyFile(filepath, thumbnailPath)
  
  return {
    original: filepath,
    thumbnail: thumbnailPath,
    size: file.size,
    mimetype: file.mimetype
  }
}).pipe(
  Effect.withSpan("file.upload", {
    attributes: {
      "file.name": file.originalname,
      "file.size": file.size
    }
  })
)

// Express setup
const app = express()
const upload = multer({ dest: 'temp/' })

app.post('/upload', upload.single('file'), async (req, res) => {
  if (!req.file) {
    return res.status(400).json({ error: 'No file uploaded' })
  }
  
  const result = await runtime.runPromise(
    handleFileUpload(req.file).pipe(
      Effect.catchAll(error => 
        Effect.succeed({
          error: error.message
        })
      )
    )
  )
  
  if ('error' in result) {
    res.status(400).json(result)
  } else {
    res.json(result)
  }
})

// Cleanup on shutdown
process.on('SIGTERM', () => {
  runtime.dispose()
})
```

### Testing Strategies

Testing file system operations with mocked implementations:

```typescript
import { FileSystem } from "@effect/platform"
import { Effect, Layer, HashMap, Ref } from "effect"
import { describe, it, expect } from "vitest"

// Create in-memory file system for testing
const createMockFileSystem = () => Effect.gen(function* () {
  const files = yield* Ref.make(HashMap.empty<string, string | Uint8Array>())
  const watchers = yield* Ref.make(HashMap.empty<string, Array<(event: any) => void>>())
  
  const mockFs: Partial<FileSystem.FileSystem> = {
    readFileString: (path: string) => 
      Ref.get(files).pipe(
        Effect.flatMap(map => 
          HashMap.get(map, path).pipe(
            Option.match({
              onNone: () => Effect.fail(new SystemError({ 
                reason: "NotFound",
                module: "FileSystem",
                method: "readFileString",
                pathOrDescriptor: path
              })),
              onSome: (content) => 
                typeof content === "string" 
                  ? Effect.succeed(content)
                  : Effect.fail(new Error("Not a text file"))
            })
          )
        )
      ),
    
    writeFileString: (path: string, content: string) =>
      Ref.update(files, HashMap.set(path, content)).pipe(
        Effect.tap(() => 
          // Trigger watchers
          Ref.get(watchers).pipe(
            Effect.flatMap(map => 
              Effect.forEach(
                HashMap.values(map),
                listeners => Effect.sync(() => 
                  listeners.forEach(fn => fn({ 
                    type: "update", 
                    path 
                  }))
                )
              )
            )
          )
        )
      ),
    
    exists: (path: string) =>
      Ref.get(files).pipe(
        Effect.map(map => HashMap.has(map, path))
      ),
    
    makeDirectory: () => Effect.void,
    
    watch: (path: string) => Effect.gen(function* () {
      const listener = (fn: (event: any) => void) => {
        Ref.update(watchers, map => {
          const current = HashMap.get(map, path).pipe(
            Option.getOrElse(() => [])
          )
          return HashMap.set(map, path, [...current, fn])
        })
      }
      
      return {
        [Symbol.asyncIterator]: () => ({
          async next() {
            return new Promise((resolve) => {
              listener((event) => resolve({ value: event, done: false }))
            })
          }
        })
      }
    })
  }
  
  return mockFs as FileSystem.FileSystem
})

// Test example
describe("ConfigManager", () => {
  it("should load and reload configuration", async () => {
    const program = Effect.gen(function* () {
      const fs = yield* FileSystem.FileSystem
      
      // Write initial config
      yield* fs.writeFileString("config.json", JSON.stringify({
        port: 3000,
        host: "localhost"
      }))
      
      // Create config manager
      const configManager = yield* createConfigManager("config.json")
      
      // Check initial config
      const config1 = yield* configManager.get
      expect(config1.port).toBe(3000)
      
      // Update config file
      yield* fs.writeFileString("config.json", JSON.stringify({
        port: 4000,
        host: "0.0.0.0"
      }))
      
      // Wait for reload
      yield* Effect.sleep("200 millis")
      
      // Check updated config
      const config2 = yield* configManager.get
      expect(config2.port).toBe(4000)
    })
    
    const mockLayer = Layer.effect(
      FileSystem.FileSystem,
      createMockFileSystem()
    )
    
    await Effect.runPromise(
      Effect.scoped(
        Effect.provide(program, mockLayer)
      )
    )
  })
})
```

## Conclusion

FileSystem provides a robust, type-safe, and composable foundation for file system operations in Effect applications.

Key benefits:
- **Type Safety**: Catch file system errors at compile time with proper error types
- **Resource Safety**: Automatic cleanup of file handles and watchers with scoped operations
- **Cross-Platform**: Same API across Node.js, Bun, and browsers (with limitations)
- **Composability**: Seamlessly integrates with streams, queues, and other Effect modules
- **Testability**: Easy to mock for testing without touching the real file system

Use FileSystem when you need reliable file operations with proper error handling, automatic resource management, and the ability to compose complex file processing pipelines.