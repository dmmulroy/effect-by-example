# Terminal: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem Terminal Solves

Building interactive command-line applications traditionally involves dealing with complex Node.js APIs, handling various input/output operations, managing user interactions, and ensuring cross-platform compatibility. The traditional approach looks like this:

```typescript
import * as readline from 'readline'
import * as process from 'process'

// Traditional approach - verbose and error-prone
const rl = readline.createInterface({
  input: process.stdin,
  output: process.stdout
})

function askQuestion(question: string): Promise<string> {
  return new Promise((resolve) => {
    rl.question(question, (answer) => {
      resolve(answer)
    })
  })
}

// Manual error handling, no type safety
try {
  const userInput = await askQuestion("Enter your name: ")
  console.log(`Hello, ${userInput}!`)
  rl.close()
} catch (error) {
  console.error("Something went wrong:", error)
}
```

This approach leads to:
- **Verbose Boilerplate** - Repetitive setup code for basic terminal interactions
- **Poor Error Handling** - Manual error management with potential for unhandled exceptions
- **Platform Inconsistencies** - Different behavior across operating systems
- **Testing Difficulties** - Hard to mock and test terminal interactions
- **Resource Management** - Manual cleanup of readline interfaces and event listeners

### The Terminal Solution

The `@effect/platform/Terminal` module provides a declarative, type-safe abstraction for terminal interactions that integrates seamlessly with the Effect ecosystem:

```typescript
import { Terminal } from "@effect/platform"
import { Effect } from "effect"

// The Effect solution - clean, type-safe, composable
const greetUser = Effect.gen(function* () {
  const terminal = yield* Terminal.Terminal
  yield* terminal.display("Enter your name: ")
  const name = yield* terminal.readLine
  yield* terminal.display(`Hello, ${name}!\n`)
})
```

### Key Concepts

**Terminal Service**: A dependency-injected service that abstracts terminal input/output operations, providing a consistent interface across platforms.

**Effect Integration**: Terminal operations return Effect values, enabling composition with other Effect-based operations and automatic error handling.

**QuitException**: A special exception type that represents user-initiated termination (typically Ctrl+C), allowing graceful handling of user interruptions.

## Basic Usage Patterns

### Pattern 1: Service Access

```typescript
import { Terminal } from "@effect/platform"
import { Effect } from "effect"

// Access the Terminal service
const getTerminal = Effect.gen(function* () {
  const terminal = yield* Terminal.Terminal
  // Use terminal methods here
})
```

### Pattern 2: Displaying Output

```typescript
import { Terminal } from "@effect/platform"
import { Effect } from "effect"

// Display messages to the user
const displayMessage = (message: string) => Effect.gen(function* () {
  const terminal = yield* Terminal.Terminal
  yield* terminal.display(`${message}\n`)
})

// Display formatted output with colors (using ANSI codes)
const displayColoredMessage = (message: string, color: 'red' | 'green' | 'blue') => {
  const colors = {
    red: '\x1b[31m',
    green: '\x1b[32m',
    blue: '\x1b[34m'
  }
  const reset = '\x1b[0m'
  
  return Effect.gen(function* () {
    const terminal = yield* Terminal.Terminal
    yield* terminal.display(`${colors[color]}${message}${reset}\n`)
  })
}
```

### Pattern 3: Reading User Input

```typescript
import { Terminal } from "@effect/platform"
import { Effect } from "effect"

// Read a line of input from the user
const getUserInput = (prompt: string) => Effect.gen(function* () {
  const terminal = yield* Terminal.Terminal
  yield* terminal.display(prompt)
  return yield* terminal.readLine
})

// Read input with validation
const getValidatedInput = (prompt: string, validator: (input: string) => boolean) => {
  const askForInput = Effect.gen(function* () {
    const terminal = yield* Terminal.Terminal
    yield* terminal.display(prompt)
    const input = yield* terminal.readLine
    
    if (!validator(input)) {
      yield* terminal.display("Invalid input. Please try again.\n")
      return yield* askForInput
    }
    
    return input
  })
  
  return askForInput
}
```

## Real-World Examples

### Example 1: Interactive Configuration Setup

This example demonstrates building a CLI tool that guides users through setting up a project configuration:

```typescript
import { Terminal } from "@effect/platform"
import { NodeRuntime, NodeTerminal } from "@effect/platform-node"
import { Effect, Option } from "effect"

interface ProjectConfig {
  name: string
  framework: 'react' | 'vue' | 'angular'
  typescript: boolean
  packageManager: 'npm' | 'yarn' | 'pnpm'
}

const setupProject = Effect.gen(function* () {
  const terminal = yield* Terminal.Terminal
  
  // Welcome message
  yield* terminal.display("\n=== Project Setup Wizard ===\n\n")
  
  // Get project name
  yield* terminal.display("Enter project name: ")
  const name = yield* terminal.readLine
  
  // Get framework choice
  yield* terminal.display("\nChoose a framework:\n")
  yield* terminal.display("1. React\n")
  yield* terminal.display("2. Vue\n")
  yield* terminal.display("3. Angular\n")
  yield* terminal.display("Enter choice (1-3): ")
  
  const frameworkChoice = yield* terminal.readLine
  const framework = frameworkChoice === '1' ? 'react' : 
                   frameworkChoice === '2' ? 'vue' : 'angular'
  
  // Get TypeScript preference
  yield* terminal.display("\nUse TypeScript? (y/n): ")
  const tsChoice = yield* terminal.readLine
  const typescript = tsChoice.toLowerCase() === 'y' || tsChoice.toLowerCase() === 'yes'
  
  // Get package manager preference
  yield* terminal.display("\nChoose package manager:\n")
  yield* terminal.display("1. npm\n")
  yield* terminal.display("2. yarn\n")
  yield* terminal.display("3. pnpm\n")
  yield* terminal.display("Enter choice (1-3): ")
  
  const pmChoice = yield* terminal.readLine
  const packageManager = pmChoice === '1' ? 'npm' : 
                        pmChoice === '2' ? 'yarn' : 'pnpm'
  
  const config: ProjectConfig = { name, framework, typescript, packageManager }
  
  // Display summary
  yield* terminal.display("\n=== Configuration Summary ===\n")
  yield* terminal.display(`Project Name: ${config.name}\n`)
  yield* terminal.display(`Framework: ${config.framework}\n`)
  yield* terminal.display(`TypeScript: ${config.typescript ? 'Yes' : 'No'}\n`)
  yield* terminal.display(`Package Manager: ${config.packageManager}\n`)
  
  return config
}).pipe(
  Effect.catchTag('QuitException', () => 
    Effect.gen(function* () {
      const terminal = yield* Terminal.Terminal
      yield* terminal.display("\nSetup cancelled by user.\n")
    })
  )
)

// Run the setup
NodeRuntime.runMain(setupProject.pipe(Effect.provide(NodeTerminal.layer)))
```

### Example 2: Progress Bar Implementation

This example shows how to create an animated progress bar for long-running operations:

```typescript
import { Terminal } from "@effect/platform"
import { NodeRuntime, NodeTerminal } from "@effect/platform-node"
import { Effect, Duration } from "effect"

const createProgressBar = (total: number) => {
  let current = 0
  const barWidth = 40
  
  const updateProgress = (increment: number = 1) => Effect.gen(function* () {
    const terminal = yield* Terminal.Terminal
    current = Math.min(current + increment, total)
    
    const percentage = Math.round((current / total) * 100)
    const filled = Math.round((current / total) * barWidth)
    const empty = barWidth - filled
    
    const bar = '█'.repeat(filled) + '░'.repeat(empty)
    const progressLine = `\r[${bar}] ${percentage}% (${current}/${total})`
    
    yield* terminal.display(progressLine)
    
    if (current >= total) {
      yield* terminal.display('\n')
    }
  })
  
  return { updateProgress }
}

const simulateWork = (description: string, steps: number) => Effect.gen(function* () {
  const terminal = yield* Terminal.Terminal
  const { updateProgress } = createProgressBar(steps)
  
  yield* terminal.display(`\n${description}\n`)
  
  for (let i = 0; i < steps; i++) {
    // Simulate work
    yield* Effect.sleep(Duration.millis(100))
    yield* updateProgress()
  }
  
  yield* terminal.display("✅ Complete!\n")
})

const deploymentProcess = Effect.gen(function* () {
  yield* simulateWork("Building application...", 20)
  yield* simulateWork("Running tests...", 15)
  yield* simulateWork("Deploying to production...", 10)
  
  const terminal = yield* Terminal.Terminal
  yield* terminal.display("\n🎉 Deployment successful!\n")
})

NodeRuntime.runMain(deploymentProcess.pipe(Effect.provide(NodeTerminal.layer)))
```

### Example 3: Interactive File Manager

This example demonstrates a simple file browser with keyboard navigation:

```typescript
import { Terminal } from "@effect/platform"
import { FileSystem } from "@effect/platform"
import { NodeRuntime, NodeTerminal, NodeFileSystem } from "@effect/platform-node"
import { Effect, Array as Arr, Option } from "effect"

interface FileItem {
  name: string
  isDirectory: boolean
  path: string
}

const listFiles = (directory: string) => Effect.gen(function* () {
  const fs = yield* FileSystem.FileSystem
  const entries = yield* fs.readDirectory(directory)
  
  const items: FileItem[] = yield* Effect.all(
    entries.map(name => Effect.gen(function* () {
      const path = `${directory}/${name}`
      const stat = yield* fs.stat(path)
      return {
        name,
        isDirectory: stat.type === 'Directory',
        path
      }
    }))
  )
  
  return items.sort((a, b) => {
    // Directories first, then alphabetical
    if (a.isDirectory && !b.isDirectory) return -1
    if (!a.isDirectory && b.isDirectory) return 1
    return a.name.localeCompare(b.name)
  })
})

const displayFiles = (files: FileItem[], selectedIndex: number) => Effect.gen(function* () {
  const terminal = yield* Terminal.Terminal
  
  // Clear screen
  yield* terminal.display('\x1b[2J\x1b[H')
  
  yield* terminal.display("=== File Browser ===\n")
  yield* terminal.display("Use arrow keys to navigate, Enter to select, 'q' to quit\n\n")
  
  for (let i = 0; i < files.length; i++) {
    const file = files[i]
    const isSelected = i === selectedIndex
    const icon = file.isDirectory ? '📁' : '📄'
    const prefix = isSelected ? '> ' : '  '
    const style = isSelected ? '\x1b[7m' : '' // Reverse video for selection
    const reset = isSelected ? '\x1b[0m' : ''
    
    yield* terminal.display(`${prefix}${style}${icon} ${file.name}${reset}\n`)
  }
})

const fileBrowser = (initialPath: string = process.cwd()) => {
  const browse = (currentPath: string, selectedIndex: number = 0) => Effect.gen(function* () {
    const files = yield* listFiles(currentPath)
    yield* displayFiles(files, selectedIndex)
    
    const terminal = yield* Terminal.Terminal
    const input = yield* terminal.readLine
    
    // Handle user input
    switch (input.toLowerCase()) {
      case 'q':
        yield* terminal.display("Goodbye!\n")
        return
      case 'j': // Down
        const nextIndex = Math.min(selectedIndex + 1, files.length - 1)
        return yield* browse(currentPath, nextIndex)
      case 'k': // Up
        const prevIndex = Math.max(selectedIndex - 1, 0)
        return yield* browse(currentPath, prevIndex)
      case '': // Enter
        const selectedFile = files[selectedIndex]
        if (selectedFile?.isDirectory) {
          return yield* browse(selectedFile.path, 0)
        } else {
          yield* terminal.display(`\nSelected file: ${selectedFile?.path}\n`)
          yield* terminal.display("Press Enter to continue...")
          yield* terminal.readLine
          return yield* browse(currentPath, selectedIndex)
        }
      default:
        return yield* browse(currentPath, selectedIndex)
    }
  })
  
  return browse(initialPath)
}

NodeRuntime.runMain(
  fileBrowser().pipe(
    Effect.provide(NodeTerminal.layer),
    Effect.provide(NodeFileSystem.layer),
    Effect.catchTag('QuitException', () => 
      Effect.gen(function* () {
        const terminal = yield* Terminal.Terminal
        yield* terminal.display("\nBrowser closed.\n")
      })
    )
  )
)
```

## Advanced Features Deep Dive

### Feature 1: ANSI Escape Sequences for Rich Terminal Output

The Terminal module supports ANSI escape sequences for creating rich, interactive terminal experiences with colors, cursor control, and screen manipulation.

#### Basic ANSI Color Usage

```typescript
import { Terminal } from "@effect/platform"
import { Effect } from "effect"

const AnsiColors = {
  reset: '\x1b[0m',
  bright: '\x1b[1m',
  dim: '\x1b[2m',
  underscore: '\x1b[4m',
  blink: '\x1b[5m',
  reverse: '\x1b[7m',
  hidden: '\x1b[8m',
  
  // Foreground colors
  fgBlack: '\x1b[30m',
  fgRed: '\x1b[31m',
  fgGreen: '\x1b[32m',
  fgYellow: '\x1b[33m',
  fgBlue: '\x1b[34m',
  fgMagenta: '\x1b[35m',
  fgCyan: '\x1b[36m',
  fgWhite: '\x1b[37m',
  
  // Background colors
  bgBlack: '\x1b[40m',
  bgRed: '\x1b[41m',
  bgGreen: '\x1b[42m',
  bgYellow: '\x1b[43m',
  bgBlue: '\x1b[44m',
  bgMagenta: '\x1b[45m',
  bgCyan: '\x1b[46m',
  bgWhite: '\x1b[47m'
} as const

const coloredOutput = Effect.gen(function* () {
  const terminal = yield* Terminal.Terminal
  
  yield* terminal.display(`${AnsiColors.fgRed}Error: ${AnsiColors.reset}Something went wrong\n`)
  yield* terminal.display(`${AnsiColors.fgGreen}Success: ${AnsiColors.reset}Operation completed\n`)
  yield* terminal.display(`${AnsiColors.fgYellow}Warning: ${AnsiColors.reset}Check your configuration\n`)
  yield* terminal.display(`${AnsiColors.bright}${AnsiColors.fgBlue}Info: ${AnsiColors.reset}Process started\n`)
})
```

#### Real-World ANSI Example: Status Dashboard

```typescript
import { Terminal } from "@effect/platform"
import { Effect, Duration, Array as Arr } from "effect"

interface ServiceStatus {
  name: string
  status: 'running' | 'stopped' | 'error'
  uptime: string
  memory: string
}

const createStatusDisplay = (services: ServiceStatus[]) => Effect.gen(function* () {
  const terminal = yield* Terminal.Terminal
  
  // Clear screen and move cursor to top
  yield* terminal.display('\x1b[2J\x1b[H')
  
  // Title
  yield* terminal.display(`${AnsiColors.bright}${AnsiColors.fgCyan}`)
  yield* terminal.display('='.repeat(60) + '\n')
  yield* terminal.display('           SERVICE STATUS DASHBOARD\n')
  yield* terminal.display('='.repeat(60) + '\n')
  yield* terminal.display(AnsiColors.reset)
  
  // Headers
  yield* terminal.display(`${AnsiColors.bright}`)
  yield* terminal.display('SERVICE'.padEnd(20))
  yield* terminal.display('STATUS'.padEnd(12))
  yield* terminal.display('UPTIME'.padEnd(15))
  yield* terminal.display('MEMORY\n')
  yield* terminal.display('-'.repeat(60) + '\n')
  yield* terminal.display(AnsiColors.reset)
  
  // Service rows
  for (const service of services) {
    const statusColor = service.status === 'running' ? AnsiColors.fgGreen :
                       service.status === 'error' ? AnsiColors.fgRed : AnsiColors.fgYellow
    const statusIcon = service.status === 'running' ? '●' :
                      service.status === 'error' ? '●' : '●'
    
    yield* terminal.display(service.name.padEnd(20))
    yield* terminal.display(`${statusColor}${statusIcon} ${service.status.toUpperCase()}${AnsiColors.reset}`.padEnd(20))
    yield* terminal.display(service.uptime.padEnd(15))
    yield* terminal.display(service.memory + '\n')
  }
  
  yield* terminal.display('\n')
  yield* terminal.display(`${AnsiColors.dim}Last updated: ${new Date().toLocaleTimeString()}${AnsiColors.reset}\n`)
  yield* terminal.display(`${AnsiColors.dim}Press Ctrl+C to exit${AnsiColors.reset}\n`)
})

const monitorServices = Effect.gen(function* () {
  const services: ServiceStatus[] = [
    { name: 'web-server', status: 'running', uptime: '2d 4h 32m', memory: '245MB' },
    { name: 'database', status: 'running', uptime: '5d 12h 18m', memory: '512MB' },
    { name: 'cache-server', status: 'error', uptime: '0m', memory: '0MB' },
    { name: 'worker-queue', status: 'running', uptime: '1d 8h 45m', memory: '128MB' }
  ]
  
  // Update display every 2 seconds
  yield* createStatusDisplay(services)
  yield* Effect.sleep(Duration.seconds(2))
  yield* monitorServices // Recursively update
})
```

#### Advanced ANSI: Cursor Control and Interactive Menus

```typescript
import { Terminal } from "@effect/platform"
import { Effect } from "effect"

const CursorControl = {
  hide: '\x1b[?25l',
  show: '\x1b[?25h',
  savePosition: '\x1b[s',
  restorePosition: '\x1b[u',
  moveUp: (lines: number) => `\x1b[${lines}A`,
  moveDown: (lines: number) => `\x1b[${lines}B`,
  moveRight: (cols: number) => `\x1b[${cols}C`,
  moveLeft: (cols: number) => `\x1b[${cols}D`,
  moveTo: (row: number, col: number) => `\x1b[${row};${col}H`,
  clearLine: '\x1b[2K',
  clearScreen: '\x1b[2J\x1b[H'
} as const

const createInteractiveMenu = (options: string[], title: string) => {
  const displayMenu = (selectedIndex: number) => Effect.gen(function* () {
    const terminal = yield* Terminal.Terminal
    
    yield* terminal.display(CursorControl.clearScreen)
    yield* terminal.display(CursorControl.hide)
    
    yield* terminal.display(`${AnsiColors.bright}${AnsiColors.fgCyan}${title}${AnsiColors.reset}\n\n`)
    
    for (let i = 0; i < options.length; i++) {
      const isSelected = i === selectedIndex
      const pointer = isSelected ? '▶ ' : '  '
      const style = isSelected ? `${AnsiColors.reverse}` : ''
      const reset = isSelected ? AnsiColors.reset : ''
      
      yield* terminal.display(`${pointer}${style}${options[i]}${reset}\n`)
    }
    
    yield* terminal.display(`\n${AnsiColors.dim}Use j/k or arrow keys to navigate, Enter to select${AnsiColors.reset}\n`)
  })
  
  const handleInput = (selectedIndex: number) => Effect.gen(function* () {
    const terminal = yield* Terminal.Terminal
    const input = yield* terminal.readLine
    
    switch (input.toLowerCase()) {
      case 'j':
        const nextIndex = (selectedIndex + 1) % options.length
        yield* displayMenu(nextIndex)
        return yield* handleInput(nextIndex)
      case 'k':
        const prevIndex = selectedIndex === 0 ? options.length - 1 : selectedIndex - 1
        yield* displayMenu(prevIndex)
        return yield* handleInput(prevIndex)
      case '':
        yield* terminal.display(CursorControl.show)
        return selectedIndex
      default:
        return yield* handleInput(selectedIndex)
    }
  })
  
  return Effect.gen(function* () {
    yield* displayMenu(0)
    const selection = yield* handleInput(0)
    yield* terminal.display(CursorControl.show)
    return { selectedIndex: selection, selectedOption: options[selection] }
  })
}
```

### Feature 2: Error Handling and User Interruption

The Terminal module provides robust error handling, particularly for user interruptions via QuitException.

#### Graceful Interruption Handling

```typescript
import { Terminal } from "@effect/platform"
import { Effect, Exit, Duration } from "effect"

const longRunningTask = Effect.gen(function* () {
  const terminal = yield* Terminal.Terminal
  
  yield* terminal.display("Starting long task...\n")
  yield* terminal.display("Press Ctrl+C to cancel\n\n")
  
  for (let i = 1; i <= 10; i++) {
    yield* terminal.display(`Step ${i}/10 processing...\n`)
    yield* Effect.sleep(Duration.seconds(1))
  }
  
  yield* terminal.display("Task completed successfully!\n")
}).pipe(
  Effect.catchTag('QuitException', () => 
    Effect.gen(function* () {
      const terminal = yield* Terminal.Terminal
      yield* terminal.display("\n\n⚠️  Task interrupted by user\n")
      yield* terminal.display("Cleaning up resources...\n")
      
      // Simulate cleanup
      yield* Effect.sleep(Duration.millis(500))
      yield* terminal.display("✅ Cleanup complete\n")
      yield* terminal.display("Goodbye!\n")
    })
  ),
  Effect.onExit((exit) => 
    Exit.match(exit, {
      onFailure: (cause) => Effect.gen(function* () {
        const terminal = yield* Terminal.Terminal
        yield* terminal.display(`\n❌ Task failed: ${cause}\n`)
      }),
      onSuccess: () => Effect.unit
    })
  )
)
```

#### Advanced Error Recovery Patterns

```typescript
import { Terminal } from "@effect/platform"
import { Effect, Option, Duration } from "effect"

const retryableOperation = (operation: string) => Effect.gen(function* () {
  const terminal = yield* Terminal.Terminal
  
  // Simulate potential failure
  const shouldFail = Math.random() < 0.6
  if (shouldFail) {
    return yield* Effect.fail(new Error(`${operation} failed`))
  }
  
  yield* terminal.display(`✅ ${operation} succeeded\n`)
})

const operationWithRetry = (operation: string, maxRetries: number = 3) => {
  const attemptOperation = (attempt: number) => Effect.gen(function* () {
    const terminal = yield* Terminal.Terminal
    
    yield* terminal.display(`Attempting ${operation} (${attempt}/${maxRetries})...\n`)
    
    return yield* retryableOperation(operation).pipe(
      Effect.catchAll((error) => 
        Effect.gen(function* () {
          yield* terminal.display(`❌ Attempt ${attempt} failed: ${error.message}\n`)
          
          if (attempt >= maxRetries) {
            yield* terminal.display(`🚫 Max retries exceeded for ${operation}\n`)
            yield* terminal.display("Would you like to try again? (y/n): ")
            
            const input = yield* terminal.readLine
            if (input.toLowerCase() === 'y' || input.toLowerCase() === 'yes') {
              yield* terminal.display("Resetting retry counter...\n")
              return yield* attemptOperation(1)
            } else {
              return yield* Effect.fail(new Error(`User cancelled ${operation} after ${maxRetries} retries`))
            }
          } else {
            yield* terminal.display(`Waiting 2 seconds before retry...\n`)
            yield* Effect.sleep(Duration.seconds(2))
            return yield* attemptOperation(attempt + 1)
          }
        })
      )
    )
  })
  
  return attemptOperation(1)
}
```

### Feature 3: Cross-Platform Terminal Capabilities

The Terminal module handles cross-platform differences automatically through platform-specific implementations.

#### Platform Detection and Feature Availability

```typescript
import { Terminal } from "@effect/platform"
import { Effect } from "effect"

const detectTerminalCapabilities = Effect.gen(function* () {
  const terminal = yield* Terminal.Terminal
  
  // Check if we're in a TTY (interactive terminal)
  const isTTY = process.stdout.isTTY
  
  // Check for color support
  const supportsColor = !!(
    process.env.COLORTERM ||
    process.env.TERM?.includes('color') ||
    process.env.TERM?.includes('xterm') ||
    process.env.TERM?.includes('screen')
  )
  
  // Check terminal width/height
  const width = process.stdout.columns || 80
  const height = process.stdout.rows || 24
  
  yield* terminal.display("=== Terminal Capabilities ===\n")
  yield* terminal.display(`Platform: ${process.platform}\n`)
  yield* terminal.display(`TTY: ${isTTY ? 'Yes' : 'No'}\n`)
  yield* terminal.display(`Color Support: ${supportsColor ? 'Yes' : 'No'}\n`)
  yield* terminal.display(`Dimensions: ${width}x${height}\n`)
  yield* terminal.display(`Terminal: ${process.env.TERM || 'Unknown'}\n`)
  
  return { isTTY, supportsColor, width, height }
})

const adaptiveOutput = (message: string, type: 'info' | 'success' | 'error' | 'warning') => 
  Effect.gen(function* () {
    const terminal = yield* Terminal.Terminal
    const capabilities = yield* detectTerminalCapabilities
    
    let formattedMessage = message
    
    if (capabilities.supportsColor && capabilities.isTTY) {
      const colors = {
        info: '\x1b[34m',
        success: '\x1b[32m', 
        error: '\x1b[31m',
        warning: '\x1b[33m'
      }
      const icons = {
        info: 'ℹ️',
        success: '✅',
        error: '❌', 
        warning: '⚠️'
      }
      
      formattedMessage = `${colors[type]}${icons[type]} ${message}\x1b[0m`
    } else {
      const prefixes = {
        info: '[INFO]',
        success: '[SUCCESS]',
        error: '[ERROR]',
        warning: '[WARNING]'
      }
      
      formattedMessage = `${prefixes[type]} ${message}`
    }
    
    yield* terminal.display(formattedMessage + '\n')
  })
```

## Practical Patterns & Best Practices

### Pattern 1: Terminal UI Components

Create reusable terminal UI components for consistent user interfaces:

```typescript
import { Terminal } from "@effect/platform"
import { Effect, Array as Arr } from "effect"

// Reusable spinner component
const createSpinner = () => {
  const frames = ['⠋', '⠙', '⠹', '⠸', '⠼', '⠴', '⠦', '⠧', '⠇', '⠏']
  let frameIndex = 0
  
  const show = (message: string) => Effect.gen(function* () {
    const terminal = yield* Terminal.Terminal
    const frame = frames[frameIndex % frames.length]
    frameIndex++
    
    yield* terminal.display(`\r${frame} ${message}`)
  })
  
  const hide = Effect.gen(function* () {
    const terminal = yield* Terminal.Terminal
    yield* terminal.display('\r\x1b[K') // Clear line
  })
  
  return { show, hide }
}

// Reusable table component
const createTable = <T>(data: T[], columns: Array<{ key: keyof T; title: string; width: number }>) => 
  Effect.gen(function* () {
    const terminal = yield* Terminal.Terminal
    
    // Header
    const headerRow = columns
      .map(col => col.title.padEnd(col.width))
      .join(' | ')
    
    yield* terminal.display(headerRow + '\n')
    yield* terminal.display('-'.repeat(headerRow.length) + '\n')
    
    // Rows
    for (const row of data) {
      const dataRow = columns
        .map(col => String(row[col.key]).padEnd(col.width))
        .join(' | ')
      
      yield* terminal.display(dataRow + '\n')
    }
  })

// Reusable confirmation dialog
const confirm = (message: string, defaultValue: boolean = false) => Effect.gen(function* () {
  const terminal = yield* Terminal.Terminal
  const defaultText = defaultValue ? '[Y/n]' : '[y/N]'
  
  yield* terminal.display(`${message} ${defaultText}: `)
  const input = yield* terminal.readLine
  
  if (input === '') return defaultValue
  
  const normalized = input.toLowerCase()
  return normalized === 'y' || normalized === 'yes'
})

// Usage example
const demonstrateComponents = Effect.gen(function* () {
  const terminal = yield* Terminal.Terminal
  const spinner = createSpinner()
  
  // Show loading with spinner
  yield* spinner.show("Loading data...")
  yield* Effect.sleep(Duration.seconds(2))
  yield* spinner.hide()
  
  // Display data in table
  const userData = [
    { name: 'John Doe', email: 'john@example.com', role: 'Admin' },
    { name: 'Jane Smith', email: 'jane@example.com', role: 'User' },
    { name: 'Bob Wilson', email: 'bob@example.com', role: 'Editor' }
  ]
  
  yield* terminal.display("User Data:\n")
  yield* createTable(userData, [
    { key: 'name', title: 'Name', width: 15 },
    { key: 'email', title: 'Email', width: 25 },
    { key: 'role', title: 'Role', width: 10 }
  ])
  
  // Confirm action
  const shouldContinue = yield* confirm("Continue with operation?", true)
  if (shouldContinue) {
    yield* terminal.display("Operation confirmed!\n")
  } else {
    yield* terminal.display("Operation cancelled.\n")
  }
})
```

### Pattern 2: Input Validation and Sanitization

Implement robust input validation patterns:

```typescript
import { Terminal } from "@effect/platform"
import { Effect, Option, Either } from "effect"

// Generic validation function type
type Validator<T> = (input: string) => Either.Either<string, T>

// Common validators
const Validators = {
  email: (input: string): Either.Either<string, string> => {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
    return emailRegex.test(input) 
      ? Either.right(input)
      : Either.left("Please enter a valid email address")
  },
  
  number: (min?: number, max?: number) => (input: string): Either.Either<string, number> => {
    const num = parseFloat(input)
    if (isNaN(num)) return Either.left("Please enter a valid number")
    if (min !== undefined && num < min) return Either.left(`Number must be at least ${min}`)
    if (max !== undefined && num > max) return Either.left(`Number must be at most ${max}`)
    return Either.right(num)
  },
  
  required: (input: string): Either.Either<string, string> => {
    return input.trim().length > 0 
      ? Either.right(input.trim())
      : Either.left("This field is required")
  },
  
  minLength: (min: number) => (input: string): Either.Either<string, string> => {
    return input.length >= min
      ? Either.right(input)
      : Either.left(`Must be at least ${min} characters`)
  },
  
  choice: <T extends string>(options: readonly T[]) => (input: string): Either.Either<string, T> => {
    const option = options.find(opt => opt.toLowerCase() === input.toLowerCase()) as T | undefined
    return option
      ? Either.right(option)
      : Either.left(`Please choose one of: ${options.join(', ')}`)
  }
}

// Generic input function with validation
const getValidatedInput = <T>(
  prompt: string,
  validator: Validator<T>,
  helpText?: string
) => {
  const askForInput = Effect.gen(function* () {
    const terminal = yield* Terminal.Terminal
    
    if (helpText) {
      yield* terminal.display(`${AnsiColors.dim}${helpText}${AnsiColors.reset}\n`)
    }
    
    yield* terminal.display(prompt)
    const input = yield* terminal.readLine
    
    const result = validator(input)
    
    return Either.match(result, {
      onLeft: (error) => Effect.gen(function* () {
        yield* terminal.display(`${AnsiColors.fgRed}Error: ${error}${AnsiColors.reset}\n`)
        return yield* askForInput
      }),
      onRight: (value) => Effect.succeed(value)
    })
  })
  
  return askForInput
}

// Usage example: User registration form
const userRegistrationForm = Effect.gen(function* () {
  const terminal = yield* Terminal.Terminal
  
  yield* terminal.display("=== User Registration ===\n\n")
  
  const name = yield* getValidatedInput(
    "Full name: ",
    Validators.required,
    "Enter your full name"
  )
  
  const email = yield* getValidatedInput(
    "Email address: ",
    Validators.email,
    "Enter a valid email address (e.g., user@example.com)"
  )
  
  const age = yield* getValidatedInput(
    "Age: ",
    Validators.number(18, 120),
    "Enter your age (must be 18 or older)"
  )
  
  const role = yield* getValidatedInput(
    "Role (admin/user/guest): ",
    Validators.choice(['admin', 'user', 'guest'] as const),
    "Choose your role: admin, user, or guest"
  )
  
  // Display summary
  yield* terminal.display("\n=== Registration Summary ===\n")
  yield* terminal.display(`Name: ${name}\n`)
  yield* terminal.display(`Email: ${email}\n`)
  yield* terminal.display(`Age: ${age}\n`)
  yield* terminal.display(`Role: ${role}\n`)
  
  const confirmed = yield* getValidatedInput(
    "\nConfirm registration (yes/no): ",
    Validators.choice(['yes', 'no'] as const)
  )
  
  if (confirmed === 'yes') {
    yield* terminal.display(`\n${AnsiColors.fgGreen}✅ Registration successful!${AnsiColors.reset}\n`)
    return { name, email, age, role }
  } else {
    yield* terminal.display(`\n${AnsiColors.fgYellow}Registration cancelled.${AnsiColors.reset}\n`)
    return Option.none()
  }
})
```

### Pattern 3: State Management in Terminal Applications

Manage application state effectively in interactive terminal applications:

```typescript
import { Terminal } from "@effect/platform"
import { Effect, Ref, Array as Arr } from "effect"

interface AppState {
  currentView: 'menu' | 'list' | 'add' | 'edit'
  items: Array<{ id: number; name: string; completed: boolean }>
  selectedIndex: number
  editingId?: number
}

const initialState: AppState = {
  currentView: 'menu',
  items: [],
  selectedIndex: 0
}

// State management helpers
const createStateManager = (initialState: AppState) => Effect.gen(function* () {
  const stateRef = yield* Ref.make(initialState)
  
  const getState = Ref.get(stateRef)
  const updateState = (updater: (state: AppState) => AppState) => Ref.update(stateRef, updater)
  const setState = (newState: AppState) => Ref.set(stateRef, newState)
  
  return { getState, updateState, setState }
})

// View components
const renderMenu = (state: AppState) => Effect.gen(function* () {
  const terminal = yield* Terminal.Terminal
  
  yield* terminal.display('\x1b[2J\x1b[H') // Clear screen
  yield* terminal.display("=== TODO APP ===\n\n")
  yield* terminal.display("1. View Items\n")
  yield* terminal.display("2. Add Item\n")
  yield* terminal.display("3. Exit\n\n")
  yield* terminal.display("Choose an option: ")
})

const renderItemList = (state: AppState) => Effect.gen(function* () {
  const terminal = yield* Terminal.Terminal
  
  yield* terminal.display('\x1b[2J\x1b[H')
  yield* terminal.display("=== TODO ITEMS ===\n\n")
  
  if (state.items.length === 0) {
    yield* terminal.display("No items found.\n")
  } else {
    for (let i = 0; i < state.items.length; i++) {
      const item = state.items[i]
      const isSelected = i === state.selectedIndex
      const status = item.completed ? '✅' : '⭕'
      const prefix = isSelected ? '> ' : '  '
      const style = isSelected ? '\x1b[7m' : ''
      const reset = isSelected ? '\x1b[0m' : ''
      
      yield* terminal.display(`${prefix}${style}${status} ${item.name}${reset}\n`)
    }
  }
  
  yield* terminal.display("\nControls: j/k (navigate), space (toggle), d (delete), b (back)\n")
})

// Application logic
const todoApp = Effect.gen(function* () {
  const { getState, updateState } = yield* createStateManager(initialState)
  const terminal = yield* Terminal.Terminal
  
  const handleMenuInput = (input: string) => Effect.gen(function* () {
    switch (input) {
      case '1':
        yield* updateState(state => ({ ...state, currentView: 'list', selectedIndex: 0 }))
        break
      case '2':
        yield* updateState(state => ({ ...state, currentView: 'add' }))
        break
      case '3':
        yield* terminal.display("Goodbye!\n")
        return false
      default:
        yield* terminal.display("Invalid option. Press Enter to continue...")
        yield* terminal.readLine
    }
    return true
  })
  
  const handleListInput = (input: string) => Effect.gen(function* () {
    const state = yield* getState
    
    switch (input.toLowerCase()) {
      case 'j': // Down
        const nextIndex = Math.min(state.selectedIndex + 1, state.items.length - 1)
        yield* updateState(s => ({ ...s, selectedIndex: nextIndex }))
        break
      case 'k': // Up
        const prevIndex = Math.max(state.selectedIndex - 1, 0)
        yield* updateState(s => ({ ...s, selectedIndex: prevIndex }))
        break
      case ' ': // Toggle completion
        yield* updateState(s => ({
          ...s,
          items: s.items.map((item, i) => 
            i === s.selectedIndex ? { ...item, completed: !item.completed } : item
          )
        }))
        break
      case 'd': // Delete
        yield* updateState(s => ({
          ...s,
          items: s.items.filter((_, i) => i !== s.selectedIndex),
          selectedIndex: Math.min(s.selectedIndex, s.items.length - 2)
        }))
        break
      case 'b': // Back
        yield* updateState(s => ({ ...s, currentView: 'menu' }))
        break
    }
    return true
  })
  
  const handleAddInput = Effect.gen(function* () {
    yield* terminal.display('\x1b[2J\x1b[H')
    yield* terminal.display("=== ADD ITEM ===\n\n")
    yield* terminal.display("Item name: ")
    
    const name = yield* terminal.readLine
    
    if (name.trim()) {
      yield* updateState(state => ({
        ...state,
        items: [...state.items, { 
          id: Date.now(), 
          name: name.trim(), 
          completed: false 
        }],
        currentView: 'list'
      }))
    } else {
      yield* updateState(state => ({ ...state, currentView: 'menu' }))
    }
    
    return true
  })
  
  // Main application loop
  const mainLoop = Effect.gen(function* () {
    const state = yield* getState
    
    // Render current view
    switch (state.currentView) {
      case 'menu':
        yield* renderMenu(state)
        const menuInput = yield* terminal.readLine
        const continueMenu = yield* handleMenuInput(menuInput)
        if (!continueMenu) return
        break
        
      case 'list':
        yield* renderItemList(state)
        const listInput = yield* terminal.readLine
        yield* handleListInput(listInput)
        break
        
      case 'add':
        const continueAdd = yield* handleAddInput()
        if (!continueAdd) return
        break
    }
    
    // Continue the loop
    yield* mainLoop
  })
  
  return mainLoop
}).pipe(
  Effect.catchTag('QuitException', () => 
    Effect.gen(function* () {
      const terminal = yield* Terminal.Terminal
      yield* terminal.display("\nApplication closed.\n")
    })
  )
)
```

## Integration Examples

### Integration with File System Operations

Combine Terminal with FileSystem for file management applications:

```typescript
import { Terminal } from "@effect/platform"
import { FileSystem } from "@effect/platform"
import { NodeRuntime, NodeTerminal, NodeFileSystem } from "@effect/platform-node"
import { Effect, Array as Arr, Option } from "effect"

const fileManager = Effect.gen(function* () {
  const terminal = yield* Terminal.Terminal
  const fs = yield* FileSystem.FileSystem
  
  yield* terminal.display("Enter directory path to analyze: ")
  const dirPath = yield* terminal.readLine
  
  // Check if directory exists
  const exists = yield* fs.exists(dirPath)
  if (!exists) {
    yield* terminal.display(`Directory ${dirPath} does not exist.\n`)
    return
  }
  
  yield* terminal.display("Analyzing directory...\n")
  
  // Read directory contents
  const entries = yield* fs.readDirectory(dirPath)
  
  // Get file details
  const fileDetails = yield* Effect.all(
    entries.map(name => Effect.gen(function* () {
      const fullPath = `${dirPath}/${name}`
      const stat = yield* fs.stat(fullPath)
      return {
        name,
        isDirectory: stat.type === 'Directory',
        size: stat.size,
        modified: new Date(stat.mtime)
      }
    }))
  )
  
  // Display results
  yield* terminal.display("\n=== Directory Analysis ===\n")
  
  const directories = fileDetails.filter(f => f.isDirectory)
  const files = fileDetails.filter(f => !f.isDirectory)
  
  yield* terminal.display(`Total items: ${fileDetails.length}\n`)
  yield* terminal.display(`Directories: ${directories.length}\n`)
  yield* terminal.display(`Files: ${files.length}\n`)
  
  if (files.length > 0) {
    const totalSize = files.reduce((sum, f) => sum + f.size, 0)
    yield* terminal.display(`Total size: ${(totalSize / 1024).toFixed(2)} KB\n`)
  }
  
  yield* terminal.display("\nLargest files:\n")
  const largestFiles = files
    .sort((a, b) => b.size - a.size)
    .slice(0, 5)
  
  for (const file of largestFiles) {
    const sizeKB = (file.size / 1024).toFixed(2)
    yield* terminal.display(`  ${file.name.padEnd(30)} ${sizeKB.padStart(10)} KB\n`)
  }
})

NodeRuntime.runMain(
  fileManager.pipe(
    Effect.provide(NodeTerminal.layer),
    Effect.provide(NodeFileSystem.layer)
  )
)
```

### Integration with HTTP Client for API Tools

Create terminal-based API testing tools:

```typescript
import { Terminal } from "@effect/platform"
import { HttpClient, HttpClientRequest } from "@effect/platform"
import { NodeRuntime, NodeTerminal, NodeHttpClient } from "@effect/platform-node"
import { Effect, Schema } from "effect"

// Define API response schemas
const UserSchema = Schema.Struct({
  id: Schema.Number,
  name: Schema.String,
  email: Schema.String,
  username: Schema.String
})

const apiClient = Effect.gen(function* () {
  const terminal = yield* Terminal.Terminal
  const httpClient = yield* HttpClient.HttpClient
  
  yield* terminal.display("=== API Testing Tool ===\n\n")
  
  // Get API endpoint
  yield* terminal.display("Enter API endpoint (default: https://jsonplaceholder.typicode.com/users): ")
  const endpoint = yield* terminal.readLine
  const url = endpoint.trim() || "https://jsonplaceholder.typicode.com/users"
  
  yield* terminal.display("Making request...\n")
  
  try {
    // Make HTTP request
    const request = HttpClientRequest.get(url)
    const response = yield* httpClient.execute(request)
    
    // Parse response
    const json = yield* response.json
    
    // Display response info
    yield* terminal.display(`Status: ${response.status}\n`)
    yield* terminal.display(`Content-Type: ${response.headers['content-type'] || 'unknown'}\n\n`)
    
    // Try to parse as user array
    const parseResult = Schema.decodeUnknownEither(Schema.Array(UserSchema))(json)
    
    if (parseResult._tag === 'Right') {
      const users = parseResult.right
      yield* terminal.display("Users found:\n")
      
      for (const user of users.slice(0, 5)) { // Show first 5 users
        yield* terminal.display(`  ${user.id}. ${user.name} (${user.email})\n`)
      }
      
      if (users.length > 5) {
        yield* terminal.display(`  ... and ${users.length - 5} more\n`)
      }
    } else {
      // Fallback to pretty JSON display
      yield* terminal.display("Response data:\n")
      yield* terminal.display(JSON.stringify(json, null, 2))
    }
    
  } catch (error) {
    yield* terminal.display(`Error: ${error}\n`)
  }
})

NodeRuntime.runMain(
  apiClient.pipe(
    Effect.provide(NodeTerminal.layer),
    Effect.provide(NodeHttpClient.layer)
  )
)
```

### Testing Strategies

Effective testing approaches for Terminal-based applications:

```typescript
import { Terminal } from "@effect/platform"
import { Effect, TestContext, Layer, Ref, Array as Arr } from "effect"
import { describe, it, expect } from "vitest"

// Mock Terminal implementation for testing
interface MockTerminalState {
  displayed: string[]
  inputs: string[]
  inputIndex: number
}

const makeMockTerminal = (inputs: string[]) => Effect.gen(function* () {
  const stateRef = yield* Ref.make<MockTerminalState>({
    displayed: [],
    inputs,
    inputIndex: 0
  })
  
  const mockTerminal: Terminal.Terminal = {
    display: (message: string) => Effect.gen(function* () {
      yield* Ref.update(stateRef, state => ({
        ...state,
        displayed: [...state.displayed, message]
      }))
    }),
    
    readLine: Effect.gen(function* () {
      const state = yield* Ref.get(stateRef)
      
      if (state.inputIndex >= state.inputs.length) {
        return yield* Effect.fail(new Error("No more mock inputs available"))
      }
      
      const input = state.inputs[state.inputIndex]
      yield* Ref.update(stateRef, s => ({ ...s, inputIndex: s.inputIndex + 1 }))
      
      return input
    })
  }
  
  const getDisplayed = Effect.map(Ref.get(stateRef), state => state.displayed)
  const getInputsUsed = Effect.map(Ref.get(stateRef), state => state.inputIndex)
  
  return { mockTerminal, getDisplayed, getInputsUsed }
})

const MockTerminalLayer = (inputs: string[]) => 
  Layer.effect(
    Terminal.Terminal,
    Effect.map(makeMockTerminal(inputs), ({ mockTerminal }) => mockTerminal)
  )

// Example application to test
const greetingApp = (name: string) => Effect.gen(function* () {
  const terminal = yield* Terminal.Terminal
  
  yield* terminal.display("What's your name? ")
  const userName = yield* terminal.readLine
  yield* terminal.display(`Hello, ${userName}! Nice to meet you.\n`)
  
  return userName
})

// Test suite
describe("Terminal Applications", () => {
  it("should handle user input correctly", async () => {
    const inputs = ["Alice"]
    const { mockTerminal, getDisplayed } = yield* makeMockTerminal(inputs)
    
    const program = greetingApp("test").pipe(
      Effect.provide(Layer.succeed(Terminal.Terminal, mockTerminal))
    )
    
    const result = yield* program
    const displayed = yield* getDisplayed
    
    expect(result).toBe("Alice")
    expect(displayed).toEqual([
      "What's your name? ",
      "Hello, Alice! Nice to meet you.\n"
    ])
  })
  
  it("should handle multiple interactions", async () => {
    const questionnaire = Effect.gen(function* () {
      const terminal = yield* Terminal.Terminal
      
      yield* terminal.display("Enter your age: ")
      const age = yield* terminal.readLine
      
      yield* terminal.display("Enter your city: ")
      const city = yield* terminal.readLine
      
      yield* terminal.display(`You are ${age} years old and live in ${city}.\n`)
      
      return { age: parseInt(age), city }
    })
    
    const inputs = ["25", "New York"]
    const { mockTerminal, getDisplayed } = yield* makeMockTerminal(inputs)
    
    const program = questionnaire.pipe(
      Effect.provide(Layer.succeed(Terminal.Terminal, mockTerminal))
    )
    
    const result = yield* program
    const displayed = yield* getDisplayed
    
    expect(result).toEqual({ age: 25, city: "New York" })
    expect(displayed).toEqual([
      "Enter your age: ",
      "Enter your city: ",
      "You are 25 years old and live in New York.\n"
    ])
  })
})

// Integration test with real Terminal
describe("Terminal Integration", () => {
  it("should work with real terminal layer", async () => {
    // This test would run against the actual terminal in CI/CD
    const simpleApp = Effect.gen(function* () {
      const terminal = yield* Terminal.Terminal
      yield* terminal.display("Integration test\n")
      return "success"
    })
    
    // Only run this test if we're in a TTY environment
    if (process.stdout.isTTY) {
      const result = yield* simpleApp.pipe(
        Effect.provide(NodeTerminal.layer)
      )
      expect(result).toBe("success")
    }
  })
})

// Property-based testing helper
const generateRandomInputs = (count: number) => 
  Array.from({ length: count }, () => 
    Math.random().toString(36).substring(2, 8)
  )

describe("Terminal Property Tests", () => {
  it("should handle arbitrary input sequences", async () => {
    const inputs = generateRandomInputs(5)
    
    const echoApp = Effect.gen(function* () {
      const terminal = yield* Terminal.Terminal
      const results: string[] = []
      
      for (let i = 0; i < inputs.length; i++) {
        yield* terminal.display(`Input ${i + 1}: `)
        const input = yield* terminal.readLine
        results.push(input)
      }
      
      return results
    })
    
    const { mockTerminal } = yield* makeMockTerminal(inputs)
    
    const program = echoApp.pipe(
      Effect.provide(Layer.succeed(Terminal.Terminal, mockTerminal))
    )
    
    const result = yield* program
    expect(result).toEqual(inputs)
  })
})
```

## Conclusion

The Terminal module provides type-safe, composable terminal interactions for Effect applications, enabling the creation of sophisticated command-line tools and interactive applications.

Key benefits:
- **Type Safety**: All terminal operations are type-safe and integrate seamlessly with Effect's type system
- **Error Handling**: Built-in support for graceful error handling and user interruption via QuitException
- **Cross-Platform**: Consistent behavior across different operating systems and terminal environments
- **Composability**: Terminal operations compose naturally with other Effect operations and services
- **Testing**: Easy to mock and test terminal interactions in isolated environments

The Terminal module is ideal for building CLI tools, interactive configuration wizards, development utilities, monitoring dashboards, and any application requiring rich terminal-based user interaction. Its integration with the Effect ecosystem ensures that terminal applications benefit from Effect's powerful error handling, resource management, and composability features.