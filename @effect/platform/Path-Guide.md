# Path: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem Path Solves

Working with file paths across different operating systems is fraught with challenges. Traditional approaches often lead to subtle bugs:

```typescript
// Traditional approach - problematic cross-platform code
const configPath = process.cwd() + '/config' + '/app.json'
// Windows: C:\Users\name\project/config/app.json (mixed separators!)
// Unix: /home/user/project/config/app.json

// Manual path manipulation
const getExtension = (filename: string) => {
  const parts = filename.split('.')
  return parts[parts.length - 1] // Fails for files without extensions
}

// Platform-specific code scattered everywhere
const tempDir = process.platform === 'win32' 
  ? process.env.TEMP 
  : '/tmp'
```

This approach leads to:
- **Platform Inconsistency** - Mixed path separators break on different operating systems
- **Security Vulnerabilities** - Path traversal attacks from unchecked relative paths
- **Edge Case Bugs** - Files without extensions, multiple dots, or special characters break naive implementations

### The Path Solution

Effect's Path module provides a unified, type-safe API for path operations that works consistently across all platforms:

```typescript
import { Path } from "@effect/platform"
import { Effect } from "effect"

// Cross-platform path joining
const configPath = Effect.gen(function* () {
  const path = yield* Path.Path
  return path.join(process.cwd(), "config", "app.json")
})
// Windows: C:\Users\name\project\config\app.json
// Unix: /home/user/project/config/app.json
```

### Key Concepts

**Path Service**: A platform-aware service that provides consistent path operations across Windows, Unix, and web environments.

**Path Normalization**: Automatic resolution of `.` and `..` segments, removing redundant separators and handling edge cases.

**Cross-Platform Abstractions**: Operations that automatically handle platform differences (separators, drive letters, UNC paths).

## Basic Usage Patterns

### Getting Started with Path Service

```typescript
import { Path } from "@effect/platform"
import { NodeContext, NodeRuntime } from "@effect/platform-node"
import { Effect } from "effect"

const program = Effect.gen(function* () {
  // Access the Path service
  const path = yield* Path.Path
  
  // Platform-specific separator
  console.log("Path separator:", path.sep)
  // Unix: /
  // Windows: \
})

NodeRuntime.runMain(program.pipe(Effect.provide(NodeContext.layer)))
```

### Joining Path Segments

```typescript
import { Path } from "@effect/platform"
import { Effect } from "effect"

const buildPath = (segments: string[]) => Effect.gen(function* () {
  const path = yield* Path.Path
  
  // Join handles empty strings, trailing slashes, and platform differences
  const joined = path.join(...segments)
  console.log("Joined path:", joined)
  
  return joined
})

// Usage
const projectPath = buildPath(["src", "components", "Button.tsx"])
// Unix: src/components/Button.tsx
// Windows: src\components\Button.tsx
```

### Path Information Extraction

```typescript
import { Path } from "@effect/platform"
import { Effect } from "effect"

const analyzeFile = (filePath: string) => Effect.gen(function* () {
  const path = yield* Path.Path
  
  return {
    directory: path.dirname(filePath),
    filename: path.basename(filePath),
    extension: path.extname(filePath),
    nameWithoutExt: path.basename(filePath, path.extname(filePath))
  }
})

// Usage
const fileInfo = analyzeFile("/home/user/documents/report.pdf")
// {
//   directory: "/home/user/documents",
//   filename: "report.pdf",
//   extension: ".pdf",
//   nameWithoutExt: "report"
// }
```

## Real-World Examples

### Example 1: Cross-Platform Configuration Loader

Building a configuration system that works across development and production environments:

```typescript
import { Path, FileSystem } from "@effect/platform"
import { NodeContext, NodeRuntime } from "@effect/platform-node"
import { Effect, Config, Layer } from "effect"
import { Schema } from "@effect/schema"

// Configuration schema
const AppConfig = Schema.Struct({
  apiUrl: Schema.String,
  port: Schema.Number,
  debug: Schema.Boolean
})

// Cross-platform config loader
const loadConfig = Effect.gen(function* () {
  const path = yield* Path.Path
  const fs = yield* FileSystem.FileSystem
  
  // Build config path based on environment
  const env = yield* Config.string("NODE_ENV").pipe(
    Config.withDefault("development")
  )
  
  // Use platform-appropriate paths
  const configDir = yield* Effect.succeed(process.cwd()).pipe(
    Effect.map(cwd => path.join(cwd, "config"))
  )
  
  const configFile = path.join(configDir, `${env}.json`)
  
  // Check if config exists
  const exists = yield* fs.exists(configFile)
  if (!exists) {
    // Fall back to default config
    const defaultConfig = path.join(configDir, "default.json")
    const content = yield* fs.readFileString(defaultConfig)
    return yield* Schema.decodeUnknown(AppConfig)(JSON.parse(content))
  }
  
  const content = yield* fs.readFileString(configFile)
  return yield* Schema.decodeUnknown(AppConfig)(JSON.parse(content))
}).pipe(
  Effect.catchTag("SystemError", () => 
    Effect.succeed({
      apiUrl: "http://localhost:3000",
      port: 3000,
      debug: true
    })
  )
)

// Usage
NodeRuntime.runMain(
  loadConfig.pipe(
    Effect.tap(config => Effect.log("Loaded config:", config)),
    Effect.provide(NodeContext.layer)
  )
)
```

### Example 2: Safe File Upload Handler

Preventing directory traversal attacks while handling file uploads:

```typescript
import { Path, FileSystem } from "@effect/platform"
import { Effect, Option } from "effect"
import { Schema } from "@effect/schema"

const UploadError = Schema.TaggedStruct("UploadError", {
  reason: Schema.Literal("InvalidPath", "FileExists", "QuotaExceeded")
})

type UploadError = Schema.Schema.Type<typeof UploadError>

// Safe file upload with path validation
const handleFileUpload = (
  filename: string,
  content: Uint8Array,
  userDir: string
) => Effect.gen(function* () {
  const path = yield* Path.Path
  const fs = yield* FileSystem.FileSystem
  
  // Sanitize filename - remove any path components
  const sanitizedName = path.basename(filename)
  
  // Ensure upload directory is absolute and normalized
  const uploadRoot = path.resolve("uploads")
  const userUploadDir = path.join(uploadRoot, userDir)
  
  // Resolve target path and verify it's within upload directory
  const targetPath = path.resolve(userUploadDir, sanitizedName)
  const relativePath = path.relative(uploadRoot, targetPath)
  
  // Prevent directory traversal
  if (relativePath.startsWith("..")) {
    return yield* Effect.fail(new UploadError({
      reason: "InvalidPath"
    }))
  }
  
  // Check if file already exists
  const exists = yield* fs.exists(targetPath)
  if (exists) {
    // Generate unique filename
    const timestamp = Date.now()
    const ext = path.extname(sanitizedName)
    const base = path.basename(sanitizedName, ext)
    const uniqueName = `${base}-${timestamp}${ext}`
    const uniquePath = path.join(userUploadDir, uniqueName)
    
    yield* fs.makeDirectory(userUploadDir, { recursive: true })
    yield* fs.writeFile(uniquePath, content)
    
    return { path: uniquePath, filename: uniqueName }
  }
  
  yield* fs.makeDirectory(userUploadDir, { recursive: true })
  yield* fs.writeFile(targetPath, content)
  
  return { path: targetPath, filename: sanitizedName }
}).pipe(
  Effect.withSpan("file.upload", {
    attributes: { 
      "file.name": filename,
      "user.dir": userDir 
    }
  })
)

// Usage with proper error handling
const uploadProgram = Effect.gen(function* () {
  const result = yield* handleFileUpload(
    "../../../etc/passwd", // Attempted path traversal
    new TextEncoder().encode("malicious content"),
    "user123"
  ).pipe(
    Effect.catchTag("UploadError", (error) => 
      Effect.succeed({
        success: false,
        error: error.reason
      })
    )
  )
  
  console.log("Upload result:", result)
})
```

### Example 3: Build Tool Path Resolution

Implementing a module resolver for a build tool:

```typescript
import { Path, FileSystem } from "@effect/platform"
import { Effect, Option, Array as Arr } from "effect"

// Module resolution with multiple strategies
const resolveModule = (
  moduleName: string,
  fromFile: string
) => Effect.gen(function* () {
  const path = yield* Path.Path
  const fs = yield* FileSystem.FileSystem
  
  // Check if it's a relative import
  if (moduleName.startsWith(".")) {
    const dir = path.dirname(fromFile)
    const resolved = path.resolve(dir, moduleName)
    
    // Try with extensions
    const extensions = [".ts", ".tsx", ".js", ".jsx", "/index.ts", "/index.js"]
    
    for (const ext of extensions) {
      const candidate = resolved + ext
      const exists = yield* fs.exists(candidate)
      if (exists) {
        return Option.some(candidate)
      }
    }
    
    return Option.none()
  }
  
  // Node modules resolution
  const nodeModulesPath = findNodeModules(fromFile)
  
  return yield* Effect.gen(function* () {
    // Walk up directory tree looking for node_modules
    let currentDir = path.dirname(fromFile)
    
    while (true) {
      const nodeModules = path.join(currentDir, "node_modules", moduleName)
      const packageJson = path.join(nodeModules, "package.json")
      
      const exists = yield* fs.exists(packageJson)
      if (exists) {
        // Read package.json to find entry point
        const content = yield* fs.readFileString(packageJson)
        const pkg = JSON.parse(content)
        const main = pkg.main || "index.js"
        
        return Option.some(path.join(nodeModules, main))
      }
      
      const parent = path.dirname(currentDir)
      if (parent === currentDir) {
        // Reached root
        return Option.none()
      }
      currentDir = parent
    }
  })
})

// Helper to find closest node_modules
const findNodeModules = (fromFile: string) => Effect.gen(function* () {
  const path = yield* Path.Path
  const fs = yield* FileSystem.FileSystem
  
  let current = path.dirname(fromFile)
  const candidates = []
  
  while (true) {
    const nodeModules = path.join(current, "node_modules")
    const exists = yield* fs.exists(nodeModules)
    
    if (exists) {
      candidates.push(nodeModules)
    }
    
    const parent = path.dirname(current)
    if (parent === current) break
    current = parent
  }
  
  return candidates
})
```

## Advanced Features Deep Dive

### Working with URLs and File Paths

Converting between file URLs and paths across platforms:

```typescript
import { Path } from "@effect/platform"
import { Effect } from "effect"

const urlConversions = Effect.gen(function* () {
  const path = yield* Path.Path
  
  // Convert file path to URL
  const filePath = "/home/user/my file.txt"
  const fileUrl = path.toFileUrl(filePath)
  console.log("File URL:", fileUrl)
  // file:///home/user/my%20file.txt
  
  // Convert URL back to path
  const backToPath = path.fromFileUrl("file:///C:/Users/name/file.txt")
  console.log("Path from URL:", backToPath)
  // Windows: C:\Users\name\file.txt
  
  // Handle special characters
  const specialPath = "/path/with spaces/and-unicode-émojis-🎉.txt"
  const encoded = path.toFileUrl(specialPath)
  const decoded = path.fromFileUrl(encoded)
  
  console.log("Round trip:", specialPath === decoded) // true
})
```

### Path Parsing and Formatting

Decomposing and reconstructing paths:

```typescript
import { Path } from "@effect/platform"
import { Effect } from "effect"

const pathManipulation = Effect.gen(function* () {
  const path = yield* Path.Path
  
  // Parse a path into components
  const parsed = path.parse("/home/user/documents/report.pdf")
  console.log("Parsed:", parsed)
  // {
  //   root: "/",
  //   dir: "/home/user/documents",
  //   base: "report.pdf",
  //   ext: ".pdf",
  //   name: "report"
  // }
  
  // Modify and reconstruct
  const modified = path.format({
    ...parsed,
    base: undefined, // Must unset base when changing name/ext
    name: "report-final",
    ext: ".docx"
  })
  console.log("Modified:", modified)
  // /home/user/documents/report-final.docx
  
  // Windows path parsing
  const winParsed = path.parse("C:\\Users\\name\\file.txt")
  // {
  //   root: "C:\\",
  //   dir: "C:\\Users\\name",
  //   base: "file.txt",
  //   ext: ".txt",
  //   name: "file"
  // }
})
```

### Relative Path Computation

Computing relative paths between directories:

```typescript
import { Path, FileSystem } from "@effect/platform"
import { Effect } from "effect"

const createSymlink = (target: string, linkPath: string) => 
  Effect.gen(function* () {
    const path = yield* Path.Path
    const fs = yield* FileSystem.FileSystem
    
    // Compute relative path for symlink
    const linkDir = path.dirname(linkPath)
    const relativeTarget = path.relative(linkDir, target)
    
    console.log(`Creating symlink: ${linkPath} -> ${relativeTarget}`)
    
    // Create directory if needed
    yield* fs.makeDirectory(linkDir, { recursive: true })
    
    // Create relative symlink
    yield* fs.symlink(relativeTarget, linkPath)
    
    // Verify the link resolves correctly
    const resolved = yield* fs.realPath(linkPath)
    const targetReal = yield* fs.realPath(target)
    
    return resolved === targetReal
  })

// Example: Create project structure with symlinks
const setupProject = Effect.gen(function* () {
  const path = yield* Path.Path
  
  // Create shared components symlink
  const sharedComponents = path.resolve("packages/shared/components")
  const appComponents = path.resolve("packages/app/src/components/shared")
  
  yield* createSymlink(sharedComponents, appComponents)
  
  // The symlink will use relative path: ../../../shared/components
})
```

## Practical Patterns & Best Practices

### Pattern 1: Safe Path Builder

Creating a type-safe path builder for consistent path construction:

```typescript
import { Path } from "@effect/platform"
import { Effect, Context, Layer } from "effect"

// Define application paths configuration
class AppPaths extends Context.Tag("AppPaths")<
  AppPaths,
  {
    readonly root: string
    readonly data: string
    readonly config: string
    readonly logs: string
    readonly temp: string
  }
>() {}

// Path builder service
const makePathBuilder = Effect.gen(function* () {
  const path = yield* Path.Path
  const paths = yield* AppPaths
  
  const buildPath = (category: keyof typeof paths, ...segments: string[]) => {
    const base = paths[category]
    return segments.length > 0 
      ? path.join(base, ...segments)
      : base
  }
  
  const ensureDir = (category: keyof typeof paths, ...segments: string[]) =>
    Effect.gen(function* () {
      const fs = yield* FileSystem.FileSystem
      const dirPath = buildPath(category, ...segments)
      yield* fs.makeDirectory(dirPath, { recursive: true })
      return dirPath
    })
  
  return {
    buildPath,
    ensureDir,
    paths: {
      dataFile: (filename: string) => buildPath("data", filename),
      configFile: (filename: string) => buildPath("config", filename),
      logFile: (date: Date) => {
        const dateStr = date.toISOString().split("T")[0]
        return buildPath("logs", dateStr, "app.log")
      },
      tempFile: (prefix: string) => {
        const random = Math.random().toString(36).substring(7)
        return buildPath("temp", `${prefix}-${random}`)
      }
    }
  } as const
})

// Layer with development paths
const DevPathsLayer = Layer.succeed(AppPaths, {
  root: process.cwd(),
  data: "./data",
  config: "./config",
  logs: "./logs",
  temp: "./tmp"
})

// Layer with production paths
const ProdPathsLayer = Effect.gen(function* () {
  const path = yield* Path.Path
  return {
    root: "/app",
    data: "/var/lib/app/data",
    config: "/etc/app",
    logs: "/var/log/app",
    temp: path.join(process.env.TMPDIR || "/tmp", "app")
  }
}).pipe(Layer.effect(AppPaths))
```

### Pattern 2: Path Validation Utilities

Creating reusable path validation helpers:

```typescript
import { Path } from "@effect/platform"
import { Effect, Predicate } from "effect"
import { Schema } from "@effect/schema"

// Path validation schemas
const PathValidation = {
  // Ensure path is within a directory
  isWithin: (root: string) => (testPath: string) =>
    Effect.gen(function* () {
      const path = yield* Path.Path
      const resolvedRoot = path.resolve(root)
      const resolvedTest = path.resolve(testPath)
      const relative = path.relative(resolvedRoot, resolvedTest)
      
      return !relative.startsWith("..")
    }),
  
  // Check for dangerous patterns
  isSafe: (filename: string) =>
    Effect.gen(function* () {
      const path = yield* Path.Path
      const base = path.basename(filename)
      
      // Check for null bytes
      if (base.includes("\0")) return false
      
      // Check for reserved names (Windows)
      const reserved = [
        "CON", "PRN", "AUX", "NUL",
        "COM1", "COM2", "COM3", "COM4",
        "LPT1", "LPT2", "LPT3", "LPT4"
      ]
      
      const nameWithoutExt = path.basename(base, path.extname(base))
      if (reserved.includes(nameWithoutExt.toUpperCase())) {
        return false
      }
      
      // Check for problematic characters
      const problematic = /[<>:"|?*]/
      return !problematic.test(base)
    }),
  
  // Allowed extensions filter
  hasExtension: (allowed: string[]) => (filename: string) =>
    Effect.gen(function* () {
      const path = yield* Path.Path
      const ext = path.extname(filename).toLowerCase()
      return allowed.includes(ext)
    })
}

// Safe path schema
const SafePath = Schema.String.pipe(
  Schema.filter((path) => 
    Effect.gen(function* () {
      const isValid = yield* PathValidation.isSafe(path)
      return isValid
    }),
    {
      message: () => "Invalid or unsafe path"
    }
  )
)
```

## Integration Examples

### Integration with FileSystem Module

Building a file manager with Path and FileSystem:

```typescript
import { Path, FileSystem } from "@effect/platform"
import { Effect, Stream, Option } from "effect"
import { NodeContext, NodeRuntime } from "@effect/platform-node"

// File manager service
const FileManager = Effect.gen(function* () {
  const path = yield* Path.Path
  const fs = yield* FileSystem.FileSystem
  
  // Copy directory with progress
  const copyDirectory = (
    source: string,
    destination: string,
    onProgress?: (file: string) => Effect.Effect<void>
  ) => Effect.gen(function* () {
    // Ensure source exists and is directory
    const stat = yield* fs.stat(source)
    if (stat.type !== "Directory") {
      return yield* Effect.fail(new Error("Source is not a directory"))
    }
    
    // Create destination
    yield* fs.makeDirectory(destination, { recursive: true })
    
    // Walk source directory
    const files = yield* fs.readDirectory(source, { recursive: true })
    
    for (const file of files) {
      const srcPath = path.join(source, file.path)
      const destPath = path.join(destination, file.path)
      
      if (file.type === "Directory") {
        yield* fs.makeDirectory(destPath, { recursive: true })
      } else {
        // Ensure parent directory exists
        yield* fs.makeDirectory(path.dirname(destPath), { recursive: true })
        
        // Copy file
        yield* fs.copyFile(srcPath, destPath)
        
        // Report progress
        if (onProgress) {
          yield* onProgress(file.path)
        }
      }
    }
    
    return files.length
  })
  
  // Find files by pattern
  const findFiles = (
    root: string,
    pattern: RegExp
  ) => fs.readDirectory(root, { recursive: true }).pipe(
    Effect.map(files => 
      files
        .filter(f => f.type === "File")
        .filter(f => pattern.test(f.path))
        .map(f => path.join(root, f.path))
    )
  )
  
  // Clean old files
  const cleanOldFiles = (
    directory: string,
    maxAgeDays: number
  ) => Effect.gen(function* () {
    const now = Date.now()
    const maxAge = maxAgeDays * 24 * 60 * 60 * 1000
    
    const files = yield* fs.readDirectory(directory, { recursive: true })
    let cleaned = 0
    
    for (const file of files) {
      if (file.type === "File") {
        const filePath = path.join(directory, file.path)
        const stat = yield* fs.stat(filePath)
        
        if (now - stat.mtime.getTime() > maxAge) {
          yield* fs.remove(filePath)
          cleaned++
        }
      }
    }
    
    return cleaned
  })
  
  return { copyDirectory, findFiles, cleanOldFiles }
})

// Usage
const fileOpsProgram = Effect.gen(function* () {
  const manager = yield* FileManager
  
  // Copy with progress tracking
  const copied = yield* manager.copyDirectory(
    "./src",
    "./backup/src",
    (file) => Effect.log(`Copying: ${file}`)
  )
  
  yield* Effect.log(`Copied ${copied} files`)
  
  // Find TypeScript files
  const tsFiles = yield* manager.findFiles("./src", /\.tsx?$/)
  yield* Effect.log(`Found ${tsFiles.length} TypeScript files`)
  
  // Clean old log files
  const cleaned = yield* manager.cleanOldFiles("./logs", 30)
  yield* Effect.log(`Cleaned ${cleaned} old log files`)
})

NodeRuntime.runMain(
  fileOpsProgram.pipe(Effect.provide(NodeContext.layer))
)
```

### Testing Strategies

Testing path operations with mocked filesystem:

```typescript
import { Path, FileSystem } from "@effect/platform"
import { Effect, Layer, TestContext, TestClock } from "effect"
import { describe, it, expect } from "vitest"

// Path operations to test
const createBackup = (filePath: string) => Effect.gen(function* () {
  const path = yield* Path.Path
  const fs = yield* FileSystem.FileSystem
  
  const dir = path.dirname(filePath)
  const ext = path.extname(filePath)
  const base = path.basename(filePath, ext)
  
  const timestamp = yield* TestClock.currentTimeMillis
  const backupName = `${base}_backup_${timestamp}${ext}`
  const backupPath = path.join(dir, backupName)
  
  yield* fs.copyFile(filePath, backupPath)
  return backupPath
})

describe("Path Operations", () => {
  it("creates backup with timestamp", () =>
    Effect.gen(function* () {
      const testLayer = Layer.merge(
        FileSystem.layerNoop({
          copyFile: (from, to) => {
            expect(from).toBe("/data/important.db")
            expect(to).toMatch(/important_backup_\d+\.db$/)
            return Effect.succeed(undefined)
          },
          exists: () => Effect.succeed(true)
        }),
        TestContext.TestContext
      )
      
      const backup = yield* createBackup("/data/important.db").pipe(
        Effect.provide(testLayer),
        TestClock.setTime(1234567890)
      )
      
      expect(backup).toBe("/data/important_backup_1234567890.db")
    })
  )
  
  it("handles cross-platform paths", () =>
    Effect.gen(function* () {
      const path = yield* Path.Path
      
      // Test path joining
      const joined = path.join("user", "documents", "file.txt")
      expect(joined).toMatch(/user[\/\\]documents[\/\\]file\.txt/)
      
      // Test normalization
      const normalized = path.normalize("./user/../documents/./file.txt")
      expect(normalized).toBe(path.join("documents", "file.txt"))
      
      // Test resolution
      const resolved = path.resolve("documents", "..", "downloads")
      expect(resolved).toContain("downloads")
      expect(path.isAbsolute(resolved)).toBe(true)
    }).pipe(
      Effect.provide(TestContext.TestContext)
    )
  )
})
```

## Conclusion

The Path module provides **cross-platform compatibility**, **security through validation**, and **type-safe operations** for file path manipulation in Effect applications.

Key benefits:
- **Platform Abstraction**: Write once, run anywhere without platform-specific code
- **Security by Default**: Built-in protection against path traversal and injection attacks
- **Composable API**: Seamlessly integrates with FileSystem and other Effect modules

Use the Path module whenever you need to work with file paths, especially in applications that must run across different operating systems or handle user-provided paths.