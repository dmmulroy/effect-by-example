# Command: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem Command Solves

When building automation tools, CI/CD pipelines, or system administration scripts, developers often struggle with:

```typescript
// Traditional approach - spawn processes with Node.js
import { spawn } from 'child_process'

// Fragile error handling
const runCommand = (cmd: string, args: string[]): Promise<string> => {
  return new Promise((resolve, reject) => {
    const process = spawn(cmd, args)
    let output = ''
    let error = ''
    
    process.stdout.on('data', (data) => {
      output += data.toString()
    })
    
    process.stderr.on('data', (data) => {
      error += data.toString()
    })
    
    process.on('close', (code) => {
      if (code === 0) {
        resolve(output)
      } else {
        reject(new Error(`Process failed with code ${code}: ${error}`))
      }
    })
    
    process.on('error', reject)
  })
}

// Complex composition and error handling
async function buildProject() {
  try {
    await runCommand('npm', ['install'])
    await runCommand('npm', ['run', 'build'])
    await runCommand('npm', ['test'])
  } catch (error) {
    // What went wrong? Which step failed?
    console.error('Build failed:', error)
    throw error
  }
}
```

This approach leads to:
- **Boilerplate Hell** - Verbose process management and event handling
- **Poor Error Context** - Lost information about which command failed and why
- **Type Unsafe** - No compile-time guarantees about command success or output format
- **Difficult Composition** - Hard to chain commands, handle retries, or implement timeouts
- **Platform Inconsistencies** - Windows vs Unix differences in shell behavior

### The Command Solution

Effect's Command module provides type-safe, composable command execution with comprehensive error handling:

```typescript
import { Command } from "@effect/platform"
import { NodeContext, NodeRuntime } from "@effect/platform-node"
import { Effect } from "effect"

// Type-safe command creation
const buildProject = Effect.gen(function* () {
  yield* Command.string(Command.make("npm", "install"))
  yield* Command.string(Command.make("npm", "run", "build"))
  yield* Command.string(Command.make("npm", "test"))
  console.log("Build completed successfully!")
}).pipe(
  Effect.catchTag("SystemError", (error) => 
    Effect.fail(`Build failed: ${error.reason}`)
  ),
  Effect.withSpan("project.build"),
  Effect.timeout("5 minutes")
)

// Clean execution
NodeRuntime.runMain(buildProject.pipe(Effect.provide(NodeContext.layer)))
```

### Key Concepts

**Command Creation**: Build command objects with `Command.make(command, ...args)` - immutable and reusable

**Execution Methods**: Multiple output formats - `string`, `lines`, `stream`, `streamLines` for different use cases

**Error Handling**: Structured error types (`SystemError`, `BadArgument`) with rich context and automatic retries

**Composability**: Commands compose naturally with Effect operators - timeouts, retries, fallbacks work seamlessly

**Platform Independence**: Consistent behavior across Windows and Unix with automatic shell detection

## Basic Usage Patterns

### Pattern 1: Simple Command Execution

```typescript
import { Command } from "@effect/platform"
import { NodeContext, NodeRuntime } from "@effect/platform-node"
import { Effect } from "effect"

// Create and run a simple command
const listFiles = Effect.gen(function* () {
  const output = yield* Command.string(Command.make("ls", "-la"))
  console.log("Directory contents:")
  console.log(output)
})

NodeRuntime.runMain(listFiles.pipe(Effect.provide(NodeContext.layer)))
```

### Pattern 2: Command with Environment Variables

```typescript
// Set custom environment for command execution
const buildWithEnv = Effect.gen(function* () {
  const command = Command.make("npm", "run", "build").pipe(
    Command.env({
      NODE_ENV: "production",
      BUILD_TARGET: "web",
      API_URL: "https://api.production.com"
    }),
    Command.runInShell(true) // Use shell for environment variable expansion
  )
  
  const output = yield* Command.string(command)
  return output
}).pipe(
  Effect.catchTag("SystemError", (error) =>
    Effect.logError(`Build failed with exit code ${error.exitCode}: ${error.reason}`)
  )
)
```

### Pattern 3: Working Directory and Shell Commands

```typescript
// Execute commands in specific directories
const deployApp = Effect.gen(function* () {
  // Build in source directory
  const buildCommand = Command.make("npm", "run", "build").pipe(
    Command.workingDirectory("/path/to/source")
  )
  yield* Command.string(buildCommand)
  
  // Deploy from build directory
  const deployCommand = Command.make("rsync", "-av", "dist/", "user@server:/var/www/").pipe(
    Command.workingDirectory("/path/to/source")
  )
  yield* Command.string(deployCommand)
  
  console.log("Deployment completed!")
})
```

## Real-World Examples

### Example 1: Git Operations and Version Control

A common automation need is managing git repositories with proper error handling:

```typescript
import { Command } from "@effect/platform"
import { NodeContext, NodeRuntime } from "@effect/platform-node"
import { Effect, pipe } from "effect"

interface GitOperationError {
  readonly _tag: "GitOperationError"
  readonly operation: string
  readonly reason: string
  readonly exitCode: number
}

const GitOperationError = (operation: string, reason: string, exitCode: number): GitOperationError => ({
  _tag: "GitOperationError",
  operation,
  reason,
  exitCode
})

// Helper for git commands with enhanced error context
const gitCommand = (workingDir: string, ...args: string[]) =>
  Command.make("git", ...args).pipe(
    Command.workingDirectory(workingDir),
    Command.string,
    Effect.mapError(error => 
      error._tag === "SystemError" 
        ? GitOperationError(`git ${args.join(" ")}`, error.reason, error.exitCode ?? -1)
        : error
    )
  )

// Comprehensive git workflow
const gitWorkflow = (repoPath: string, branch: string, commitMessage: string) =>
  Effect.gen(function* () {
    // Check if repo is clean
    const status = yield* gitCommand(repoPath, "status", "--porcelain")
    if (status.trim() === "") {
      yield* Effect.logInfo("Repository is clean, no changes to commit")
      return
    }
    
    // Create and switch to feature branch
    yield* gitCommand(repoPath, "checkout", "-b", branch)
    yield* Effect.logInfo(`Created and switched to branch: ${branch}`)
    
    // Stage all changes
    yield* gitCommand(repoPath, "add", ".")
    yield* Effect.logInfo("Staged all changes")
    
    // Commit changes
    yield* gitCommand(repoPath, "commit", "-m", commitMessage)
    yield* Effect.logInfo(`Committed changes: ${commitMessage}`)
    
    // Push to remote
    yield* gitCommand(repoPath, "push", "-u", "origin", branch)
    yield* Effect.logInfo(`Pushed branch ${branch} to remote`)
    
    return { branch, commitMessage }
  }).pipe(
    Effect.catchTag("GitOperationError", (error) =>
      Effect.gen(function* () {
        yield* Effect.logError(`Git operation failed: ${error.operation}`)
        yield* Effect.logError(`Reason: ${error.reason}`)
        yield* Effect.logError(`Exit code: ${error.exitCode}`)
        return yield* Effect.fail(error)
      })
    ),
    Effect.withSpan("git.workflow", {
      attributes: { 
        "git.repo": repoPath, 
        "git.branch": branch,
        "git.commit_message": commitMessage
      }
    })
  )

// Usage example
const program = gitWorkflow(
  "/path/to/my/project", 
  "feature/new-authentication", 
  "Add OAuth2 authentication system"
)

NodeRuntime.runMain(program.pipe(Effect.provide(NodeContext.layer)))
```

### Example 2: Docker Container Management

Modern applications often require container orchestration. Here's a practical Docker management system:

```typescript
import { Command } from "@effect/platform"
import { NodeContext, NodeRuntime } from "@effect/platform-node"
import { Effect, Duration } from "effect"

interface DockerContainer {
  readonly id: string
  readonly name: string
  readonly image: string
  readonly status: string
  readonly ports: string
}

interface DockerError {
  readonly _tag: "DockerError"
  readonly operation: string
  readonly containerName?: string
  readonly reason: string
}

const DockerError = (operation: string, reason: string, containerName?: string): DockerError => ({
  _tag: "DockerError",
  operation,
  reason,
  containerName
})

// Docker command helper with error mapping
const dockerCommand = (...args: string[]) =>
  Command.make("docker", ...args).pipe(
    Command.string,
    Effect.mapError(error =>
      error._tag === "SystemError"
        ? DockerError(`docker ${args.join(" ")}`, error.reason)
        : error
    )
  )

// Parse docker ps output into structured data
const parseDockerPs = (output: string): DockerContainer[] => {
  const lines = output.trim().split('\n').slice(1) // Skip header
  return lines
    .filter(line => line.trim() !== '')
    .map(line => {
      const parts = line.split(/\s{2,}/) // Split on multiple spaces
      return {
        id: parts[0] || '',
        image: parts[1] || '',
        name: parts[6] || '',
        status: parts[4] || '',
        ports: parts[5] || ''
      }
    })
}

// Container management operations
const DockerService = {
  // List all containers
  listContainers: () =>
    Effect.gen(function* () {
      const output = yield* dockerCommand("ps", "-a", "--format", "table {{.ID}}\\t{{.Image}}\\t{{.Command}}\\t{{.CreatedAt}}\\t{{.Status}}\\t{{.Ports}}\\t{{.Names}}")
      return parseDockerPs(output)
    }),

  // Start a container with health checks
  startContainer: (name: string) =>
    Effect.gen(function* () {
      yield* dockerCommand("start", name)
      yield* Effect.logInfo(`Starting container: ${name}`)
      
      // Wait for container to be healthy
      yield* Effect.retry(
        Effect.gen(function* () {
          const output = yield* dockerCommand("inspect", name, "--format", "{{.State.Health.Status}}")
          if (output.trim() === "healthy") {
            return true
          }
          return yield* Effect.fail(new Error("Container not healthy yet"))
        }),
        {
          times: 30,
          delay: Duration.seconds(2)
        }
      )
      
      yield* Effect.logInfo(`Container ${name} is healthy and running`)
      return name
    }).pipe(
      Effect.mapError(error => 
        DockerError("start", error.message, name)
      )
    ),

  // Stop container gracefully
  stopContainer: (name: string, timeout: Duration.Duration = Duration.seconds(30)) =>
    Effect.gen(function* () {
      const timeoutSeconds = Duration.toSeconds(timeout)
      yield* dockerCommand("stop", "-t", timeoutSeconds.toString(), name)
      yield* Effect.logInfo(`Stopped container: ${name}`)
      return name
    }).pipe(
      Effect.mapError(error => 
        DockerError("stop", error.message, name)
      )
    ),

  // Get container logs
  getLogs: (name: string, lines: number = 100) =>
    Effect.gen(function* () {
      const output = yield* dockerCommand("logs", "--tail", lines.toString(), name)
      return output
    }),

  // Execute command in running container
  exec: (name: string, command: string[]) =>
    Effect.gen(function* () {
      const output = yield* dockerCommand("exec", name, ...command)
      return output
    })
}

// Application deployment workflow
const deployApplication = (appName: string, imageTag: string) =>
  Effect.gen(function* () {
    yield* Effect.logInfo(`Starting deployment of ${appName}:${imageTag}`)
    
    // Stop existing container if running
    const containers = yield* DockerService.listContainers()
    const existingContainer = containers.find(c => c.name === appName)
    
    if (existingContainer && existingContainer.status.includes("Up")) {
      yield* DockerService.stopContainer(appName)
    }
    
    // Remove old container
    if (existingContainer) {
      yield* dockerCommand("rm", appName).pipe(
        Effect.orElse(() => Effect.unit) // Ignore errors if container doesn't exist
      )
    }
    
    // Pull latest image
    yield* dockerCommand("pull", `${appName}:${imageTag}`)
    
    // Run new container
    yield* dockerCommand(
      "run", "-d",
      "--name", appName,
      "--restart", "unless-stopped",
      "-p", "3000:3000",
      "--health-cmd", "curl -f http://localhost:3000/health || exit 1",
      "--health-interval", "30s",
      "--health-timeout", "3s",
      "--health-retries", "3",
      `${appName}:${imageTag}`
    )
    
    // Start and wait for health check
    yield* DockerService.startContainer(appName)
    
    // Verify deployment
    const logs = yield* DockerService.getLogs(appName, 50)
    yield* Effect.logInfo("Recent container logs:")
    yield* Effect.logInfo(logs)
    
    return { appName, imageTag, status: "deployed" }
  }).pipe(
    Effect.catchTag("DockerError", (error) =>
      Effect.gen(function* () {
        yield* Effect.logError(`Docker deployment failed: ${error.operation}`)
        yield* Effect.logError(`Container: ${error.containerName || "unknown"}`)
        yield* Effect.logError(`Reason: ${error.reason}`)
        return yield* Effect.fail(error)
      })
    ),
    Effect.timeout(Duration.minutes(10)),
    Effect.withSpan("docker.deploy", {
      attributes: { 
        "docker.app": appName,
        "docker.image_tag": imageTag
      }
    })
  )

// Usage
const program = deployApplication("my-web-app", "v1.2.3")

NodeRuntime.runMain(program.pipe(Effect.provide(NodeContext.layer)))
```

### Example 3: System Monitoring and Health Checks

Building reliable systems requires comprehensive monitoring. Here's a system health check implementation:

```typescript
import { Command } from "@effect/platform"
import { NodeContext, NodeRuntime } from "@effect/platform-node"
import { Effect, Schedule, Duration } from "effect"

interface SystemMetrics {
  readonly cpu: number
  readonly memory: {
    readonly total: number
    readonly used: number
    readonly free: number
    readonly percentage: number
  }
  readonly disk: {
    readonly total: number
    readonly used: number
    readonly available: number
    readonly percentage: number
  }
  readonly uptime: number
  readonly loadAverage: number[]
}

interface HealthCheckResult {
  readonly service: string
  readonly status: "healthy" | "unhealthy" | "degraded"
  readonly responseTime: number
  readonly message: string
  readonly timestamp: Date
}

// Cross-platform system commands
const SystemCommands = {
  // Get CPU usage (works on both Linux and macOS)
  getCpuUsage: () =>
    Effect.gen(function* () {
      const isLinux = process.platform === "linux"
      
      if (isLinux) {
        // Linux: Use top command
        const output = yield* Command.string(
          Command.make("top", "-bn1").pipe(
            Command.runInShell(true)
          )
        )
        const cpuLine = output.split('\n').find(line => line.includes('Cpu(s)'))
        const match = cpuLine?.match(/(\d+\.\d+)%?\s*us/)
        return match ? parseFloat(match[1]) : 0
      } else {
        // macOS: Use iostat
        const output = yield* Command.string(Command.make("iostat", "-c", "1"))
        const lines = output.trim().split('\n')
        const dataLine = lines[lines.length - 1]
        const values = dataLine.trim().split(/\s+/)
        return values[0] ? parseFloat(values[0]) : 0
      }
    }),

  // Get memory information
  getMemoryInfo: () =>
    Effect.gen(function* () {
      const isLinux = process.platform === "linux"
      
      if (isLinux) {
        const output = yield* Command.string(Command.make("free", "-m"))
        const lines = output.split('\n')
        const memLine = lines.find(line => line.startsWith('Mem:'))
        
        if (memLine) {
          const values = memLine.split(/\s+/)
          const total = parseInt(values[1])
          const used = parseInt(values[2])
          const free = parseInt(values[3])
          
          return {
            total,
            used,
            free,
            percentage: Math.round((used / total) * 100)
          }
        }
      } else {
        // macOS: Use vm_stat and system_profiler
        const vmOutput = yield* Command.string(Command.make("vm_stat"))
        const memOutput = yield* Command.string(
          Command.make("system_profiler", "SPHardwareDataType").pipe(
            Command.runInShell(true)
          )
        )
        
        // Parse macOS memory info (simplified)
        const totalMatch = memOutput.match(/Memory:\s*(\d+)\s*GB/)
        const total = totalMatch ? parseInt(totalMatch[1]) * 1024 : 8192
        
        // Estimate used memory from vm_stat
        const freeMatch = vmOutput.match(/Pages free:\s*(\d+)/)
        const free = freeMatch ? Math.round(parseInt(freeMatch[1]) * 4096 / 1024 / 1024) : 0
        const used = total - free
        
        return {
          total,
          used,
          free,
          percentage: Math.round((used / total) * 100)
        }
      }
      
      // Fallback
      return { total: 0, used: 0, free: 0, percentage: 0 }
    }),

  // Get disk usage
  getDiskUsage: (path: string = "/") =>
    Effect.gen(function* () {
      const output = yield* Command.string(Command.make("df", "-h", path))
      const lines = output.split('\n')
      const dataLine = lines.find(line => !line.startsWith('Filesystem'))
      
      if (dataLine) {
        const values = dataLine.split(/\s+/)
        const total = values[1]
        const used = values[2]
        const available = values[3]
        const percentage = parseInt(values[4].replace('%', ''))
        
        return {
          total: total,
          used: used,
          available: available,
          percentage
        }
      }
      
      return { total: "0", used: "0", available: "0", percentage: 0 }
    }),

  // Get system uptime
  getUptime: () =>
    Effect.gen(function* () {
      const output = yield* Command.string(Command.make("uptime"))
      const match = output.match(/up\s+(\d+)\s+days?,\s+(\d+):(\d+)/)
      
      if (match) {
        const days = parseInt(match[1] || "0")
        const hours = parseInt(match[2] || "0")
        const minutes = parseInt(match[3] || "0")
        return days * 24 * 60 * 60 + hours * 60 * 60 + minutes * 60
      }
      
      return 0
    })
}

// Service health check helpers
const HealthChecks = {
  // Check HTTP service
  checkHttpService: (name: string, url: string, timeout: Duration.Duration = Duration.seconds(5)) =>
    Effect.gen(function* () {
      const start = Date.now()
      
      try {
        const command = Command.make("curl", "-f", "-s", "-m", "5", url)
        yield* Command.string(command).pipe(Effect.timeout(timeout))
        
        const responseTime = Date.now() - start
        return {
          service: name,
          status: "healthy" as const,
          responseTime,
          message: "Service responding normally",
          timestamp: new Date()
        }
      } catch (error) {
        const responseTime = Date.now() - start
        return {
          service: name,
          status: "unhealthy" as const,
          responseTime,
          message: `Service check failed: ${error}`,
          timestamp: new Date()
        }
      }
    }),

  // Check database connection
  checkDatabase: (name: string, connectionString: string) =>
    Effect.gen(function* () {
      const start = Date.now()
      
      try {
        // Example for PostgreSQL - adapt for your database
        const command = Command.make("pg_isready", "-d", connectionString)
        yield* Command.string(command)
        
        const responseTime = Date.now() - start
        return {
          service: name,
          status: "healthy" as const,
          responseTime,
          message: "Database connection successful",
          timestamp: new Date()
        }
      } catch (error) {
        const responseTime = Date.now() - start
        return {
          service: name,
          status: "unhealthy" as const,
          responseTime,
          message: `Database check failed: ${error}`,
          timestamp: new Date()
        }
      }
    }),

  // Check port availability
  checkPort: (name: string, host: string, port: number) =>
    Effect.gen(function* () {
      const start = Date.now()
      
      try {
        const command = Command.make("nc", "-z", "-v", host, port.toString()).pipe(
          Command.runInShell(true)
        )
        yield* Command.string(command)
        
        const responseTime = Date.now() - start
        return {
          service: name,
          status: "healthy" as const,
          responseTime,
          message: `Port ${port} is accessible`,
          timestamp: new Date()
        }
      } catch (error) {
        const responseTime = Date.now() - start
        return {
          service: name,
          status: "unhealthy" as const,
          responseTime,
          message: `Port ${port} is not accessible`,
          timestamp: new Date()
        }
      }
    })
}

// Comprehensive system monitoring
const SystemMonitor = {
  // Collect all system metrics
  collectMetrics: () =>
    Effect.gen(function* () {
      const [cpu, memory, disk, uptime] = yield* Effect.all([
        SystemCommands.getCpuUsage(),
        SystemCommands.getMemoryInfo(),
        SystemCommands.getDiskUsage(),
        SystemCommands.getUptime()
      ], { concurrency: 4 })
      
      return {
        cpu,
        memory,
        disk: {
          total: disk.total,
          used: disk.used,
          available: disk.available,
          percentage: disk.percentage
        },
        uptime,
        loadAverage: [], // Could be implemented with `uptime` command parsing
        timestamp: new Date()
      }
    }),

  // Run health checks for all services
  runHealthChecks: (services: Array<{ name: string; type: string; config: any }>) =>
    Effect.gen(function* () {
      const checks = services.map(service => {
        switch (service.type) {
          case "http":
            return HealthChecks.checkHttpService(service.name, service.config.url)
          case "database":
            return HealthChecks.checkDatabase(service.name, service.config.connectionString)
          case "port":
            return HealthChecks.checkPort(service.name, service.config.host, service.config.port)
          default:
            return Effect.succeed({
              service: service.name,
              status: "degraded" as const,
              responseTime: 0,
              message: "Unknown service type",
              timestamp: new Date()
            })
        }
      })
      
      return yield* Effect.all(checks, { concurrency: 5 })
    }),

  // Generate health report
  generateReport: (metrics: SystemMetrics, healthChecks: HealthCheckResult[]) =>
    Effect.gen(function* () {
      const overallHealth = healthChecks.every(check => check.status === "healthy") ? "healthy" : "degraded"
      const criticalServices = healthChecks.filter(check => check.status === "unhealthy")
      
      const report = {
        timestamp: new Date(),
        overallHealth,
        systemMetrics: metrics,
        services: {
          total: healthChecks.length,
          healthy: healthChecks.filter(c => c.status === "healthy").length,
          degraded: healthChecks.filter(c => c.status === "degraded").length,
          unhealthy: healthChecks.filter(c => c.status === "unhealthy").length
        },
        healthChecks,
        criticalServices,
        alerts: [
          ...(metrics.memory.percentage > 90 ? ["High memory usage"] : []),
          ...(metrics.disk.percentage > 85 ? ["High disk usage"] : []),
          ...(criticalServices.length > 0 ? [`${criticalServices.length} services unhealthy`] : [])
        ]
      }
      
      return report
    })
}

// Monitoring program that runs continuously
const monitoringProgram = Effect.gen(function* () {
  const services = [
    { name: "API Server", type: "http", config: { url: "http://localhost:3000/health" } },
    { name: "Database", type: "port", config: { host: "localhost", port: 5432 } },
    { name: "Redis", type: "port", config: { host: "localhost", port: 6379 } },
    { name: "Frontend", type: "http", config: { url: "http://localhost:8080" } }
  ]
  
  const runCheck = Effect.gen(function* () {
    yield* Effect.logInfo("Running system health check...")
    
    const [metrics, healthChecks] = yield* Effect.all([
      SystemMonitor.collectMetrics(),
      SystemMonitor.runHealthChecks(services)
    ], { concurrency: 2 })
    
    const report = yield* SystemMonitor.generateReport(metrics, healthChecks)
    
    // Log summary
    yield* Effect.logInfo(`System Health: ${report.overallHealth}`)
    yield* Effect.logInfo(`CPU: ${metrics.cpu}%, Memory: ${metrics.memory.percentage}%, Disk: ${metrics.disk.percentage}%`)
    yield* Effect.logInfo(`Services: ${report.services.healthy}/${report.services.total} healthy`)
    
    if (report.alerts.length > 0) {
      yield* Effect.logWarning(`Alerts: ${report.alerts.join(", ")}`)
    }
    
    // Log critical services
    if (report.criticalServices.length > 0) {
      yield* Effect.logError("Critical services down:")
      for (const service of report.criticalServices) {
        yield* Effect.logError(`  - ${service.service}: ${service.message}`)
      }
    }
    
    return report
  })
  
  // Run check every 30 seconds
  yield* runCheck.pipe(
    Effect.repeat(Schedule.fixed(Duration.seconds(30))),
    Effect.catchAll(error => 
      Effect.gen(function* () {
        yield* Effect.logError(`Health check failed: ${error}`)
        yield* Effect.sleep(Duration.seconds(10)) // Wait before retry
      })
    )
  )
})

// Run the monitoring system
NodeRuntime.runMain(
  monitoringProgram.pipe(
    Effect.provide(NodeContext.layer),
    Effect.timeout(Duration.hours(24)) // Run for 24 hours
  )
)
```

## Advanced Features Deep Dive

### Feature 1: Command Streaming and Real-time Output

When dealing with long-running commands or large outputs, streaming provides better resource management and user experience:

#### Basic Streaming Usage

```typescript
import { Command } from "@effect/platform"
import { NodeContext, NodeRuntime } from "@effect/platform-node"
import { Effect, Stream } from "effect"

// Stream command output line by line
const streamingTail = Effect.gen(function* () {
  const command = Command.make("tail", "-f", "/var/log/system.log")
  
  yield* Command.streamLines(command).pipe(
    Stream.take(100), // Take first 100 lines
    Stream.tap(line => Effect.logInfo(`Log: ${line}`)),
    Stream.runDrain
  )
})
```

#### Real-World Streaming Example: Build Progress

```typescript
// Stream build output with progress tracking
const streamBuildProgress = (projectPath: string) =>
  Effect.gen(function* () {
    const buildCommand = Command.make("npm", "run", "build").pipe(
      Command.workingDirectory(projectPath),
      Command.env({ NODE_ENV: "production" })
    )
    
    let linesProcessed = 0
    const startTime = Date.now()
    
    yield* Command.streamLines(buildCommand).pipe(
      Stream.tap(line => 
        Effect.gen(function* () {
          linesProcessed++
          
          // Track specific build stages
          if (line.includes("Compiling")) {
            yield* Effect.logInfo(`🔄 ${line}`)
          } else if (line.includes("Built at:")) {
            yield* Effect.logInfo(`✅ ${line}`)
          } else if (line.includes("ERROR")) {
            yield* Effect.logError(`❌ ${line}`)
          } else if (line.includes("WARNING")) {
            yield* Effect.logWarning(`⚠️  ${line}`)
          }
          
          // Progress update every 10 lines
          if (linesProcessed % 10 === 0) {
            const elapsed = Date.now() - startTime
            yield* Effect.logInfo(`Progress: ${linesProcessed} lines processed in ${elapsed}ms`)
          }
        })
      ),
      Stream.runDrain
    )
    
    const totalTime = Date.now() - startTime
    yield* Effect.logInfo(`Build completed: ${linesProcessed} lines processed in ${totalTime}ms`)
  }).pipe(
    Effect.catchTag("SystemError", (error) =>
      Effect.logError(`Build failed: ${error.reason} (exit code: ${error.exitCode})`)
    )
  )
```

### Feature 2: Command Composition and Pipelines

Compose multiple commands into complex workflows with proper error handling and resource management:

#### Advanced Pipeline: Image Processing

```typescript
interface ImageProcessingOptions {
  readonly inputPath: string
  readonly outputPath: string
  readonly width: number
  readonly height: number
  readonly quality: number
  readonly format: "jpg" | "png" | "webp"
}

const imageProcessingPipeline = (options: ImageProcessingOptions) =>
  Effect.gen(function* () {
    const { inputPath, outputPath, width, height, quality, format } = options
    
    // Validate input file exists
    yield* Command.string(Command.make("test", "-f", inputPath)).pipe(
      Effect.mapError(() => new Error(`Input file not found: ${inputPath}`))
    )
    
    // Create temporary working directory
    const tempDir = yield* Command.string(Command.make("mktemp", "-d"))
    const cleanTempDir = tempDir.trim()
    
    // Ensure cleanup
    yield* Effect.addFinalizer(() => 
      Command.string(Command.make("rm", "-rf", cleanTempDir)).pipe(
        Effect.orElse(() => Effect.unit)
      )
    )
    
    try {
      // Step 1: Convert and resize image
      const resizedPath = `${cleanTempDir}/resized.${format}`
      yield* Command.string(
        Command.make("convert", inputPath, 
          "-resize", `${width}x${height}`,
          "-quality", quality.toString(),
          resizedPath
        )
      )
      yield* Effect.logInfo(`Resized image to ${width}x${height}`)
      
      // Step 2: Optimize the image
      const optimizedPath = `${cleanTempDir}/optimized.${format}`
      
      if (format === "jpg") {
        yield* Command.string(
          Command.make("jpegoptim", "--max=" + quality, "--strip-all", 
            "--dest=" + cleanTempDir, resizedPath
          )
        )
        yield* Command.string(Command.make("mv", resizedPath, optimizedPath))
      } else if (format === "png") {
        yield* Command.string(
          Command.make("optipng", "-o2", "-out", optimizedPath, resizedPath)
        )
      } else if (format === "webp") {
        yield* Command.string(
          Command.make("cwebp", "-q", quality.toString(), resizedPath, "-o", optimizedPath)
        )
      }
      
      yield* Effect.logInfo("Optimized image")
      
      // Step 3: Move to final location
      yield* Command.string(Command.make("cp", optimizedPath, outputPath))
      yield* Effect.logInfo(`Image processing complete: ${outputPath}`)
      
      // Return metadata
      const sizeOutput = yield* Command.string(Command.make("wc", "-c", outputPath))
      const fileSize = parseInt(sizeOutput.trim().split(' ')[0])
      
      return {
        inputPath,
        outputPath,
        dimensions: { width, height },
        quality,
        format,
        fileSize,
        processingTime: Date.now()
      }
      
    } catch (error) {
      yield* Effect.logError(`Image processing failed: ${error}`)
      return yield* Effect.fail(error)
    }
  }).pipe(
    Effect.scoped, // Ensure cleanup finalizers run
    Effect.withSpan("image.process", {
      attributes: {
        "image.input": options.inputPath,
        "image.output": options.outputPath,
        "image.dimensions": `${options.width}x${options.height}`,
        "image.format": options.format
      }
    })
  )
```

### Feature 3: Cross-Platform Command Execution

Handle platform differences gracefully while maintaining consistent behavior:

#### Advanced Cross-Platform: Package Management

```typescript
interface PackageManager {
  readonly name: string
  readonly installCommand: (packages: string[]) => string[]
  readonly updateCommand: string[]
  readonly listCommand: string[]
  readonly cleanCommand: string[]
}

const detectPackageManager = (): Effect.Effect<PackageManager, Error> =>
  Effect.gen(function* () {
    // Try to detect available package managers
    const managers: Array<[string, PackageManager]> = [
      ["yarn", {
        name: "yarn",
        installCommand: (packages) => ["add", ...packages],
        updateCommand: ["upgrade"],
        listCommand: ["list"],
        cleanCommand: ["cache", "clean"]
      }],
      ["pnpm", {
        name: "pnpm",
        installCommand: (packages) => ["add", ...packages],
        updateCommand: ["update"],
        listCommand: ["list"],
        cleanCommand: ["store", "prune"]
      }],
      ["npm", {
        name: "npm",
        installCommand: (packages) => ["install", ...packages],
        updateCommand: ["update"],
        listCommand: ["list"],
        cleanCommand: ["cache", "clean", "--force"]
      }]
    ]
    
    for (const [cmd, manager] of managers) {
      try {
        yield* Command.string(Command.make("which", cmd))
        return manager
      } catch {
        // Continue to next manager
      }
    }
    
    return yield* Effect.fail(new Error("No supported package manager found"))
  })

const crossPlatformShell = (command: string): Effect.Effect<string> =>
  Effect.gen(function* () {
    const isWindows = process.platform === "win32"
    
    if (isWindows) {
      return yield* Command.string(
        Command.make("cmd", "/C", command).pipe(
          Command.runInShell(true)
        )
      )
    } else {
      return yield* Command.string(
        Command.make("sh", "-c", command)
      )
    }
  })

// Advanced project setup that works across platforms
const setupProject = (projectPath: string, dependencies: string[]) =>
  Effect.gen(function* () {
    yield* Effect.logInfo(`Setting up project at: ${projectPath}`)
    
    // Detect platform and package manager
    const packageManager = yield* detectPackageManager()
    yield* Effect.logInfo(`Using package manager: ${packageManager.name}`)
    
    // Create project directory if it doesn't exist
    yield* crossPlatformShell(`mkdir -p "${projectPath}"`)
    
    // Initialize package.json if it doesn't exist
    const packageJsonPath = `${projectPath}/package.json`
    const packageJsonExists = yield* Command.exitCode(
      Command.make("test", "-f", packageJsonPath)
    ).pipe(
      Effect.map(code => code === 0),
      Effect.orElse(() => Effect.succeed(false))
    )
    
    if (!packageJsonExists) {
      yield* Command.string(
        Command.make(packageManager.name, ...["init", "-y"]).pipe(
          Command.workingDirectory(projectPath)
        )
      )
      yield* Effect.logInfo("Initialized package.json")
    }
    
    // Install dependencies
    if (dependencies.length > 0) {
      yield* Command.string(
        Command.make(packageManager.name, ...packageManager.installCommand(dependencies)).pipe(
          Command.workingDirectory(projectPath)
        )
      )
      yield* Effect.logInfo(`Installed dependencies: ${dependencies.join(", ")}`)
    }
    
    // Create basic project structure
    const directories = ["src", "test", "dist", "docs"]
    for (const dir of directories) {
      yield* crossPlatformShell(`mkdir -p "${projectPath}/${dir}"`)
    }
    
    // Create basic files
    const files = [
      { path: "src/index.ts", content: "// Entry point\nexport {};\n" },
      { path: ".gitignore", content: "node_modules/\ndist/\n*.log\n" },
      { path: "README.md", content: `# ${projectPath.split('/').pop()}\n\nProject created with Effect Command\n` }
    ]
    
    for (const file of files) {
      const fullPath = `${projectPath}/${file.path}`
      yield* crossPlatformShell(`echo '${file.content}' > "${fullPath}"`)
    }
    
    yield* Effect.logInfo("Created project structure")
    
    return {
      projectPath,
      packageManager: packageManager.name,
      dependencies,
      structure: directories,
      files: files.map(f => f.path)
    }
  }).pipe(
    Effect.timeout(Duration.minutes(5)),
    Effect.withSpan("project.setup", {
      attributes: {
        "project.path": projectPath,
        "project.dependencies": dependencies.join(",")
      }
    })
  )
```

## Practical Patterns & Best Practices

### Pattern 1: Command Factory with Configuration

Create reusable command builders with sensible defaults and configuration:

```typescript
interface CommandConfig {
  readonly workingDirectory?: string
  readonly environment?: Record<string, string>
  readonly timeout?: Duration.Duration
  readonly retries?: number
  readonly shell?: boolean
}

// Command factory for consistent command creation
const createCommand = (
  command: string, 
  args: string[], 
  config: CommandConfig = {}
) => {
  let cmd = Command.make(command, ...args)
  
  if (config.workingDirectory) {
    cmd = cmd.pipe(Command.workingDirectory(config.workingDirectory))
  }
  
  if (config.environment) {
    cmd = cmd.pipe(Command.env(config.environment))
  }
  
  if (config.shell) {
    cmd = cmd.pipe(Command.runInShell(true))
  }
  
  let effect = Command.string(cmd)
  
  if (config.timeout) {
    effect = effect.pipe(Effect.timeout(config.timeout))
  }
  
  if (config.retries && config.retries > 0) {
    effect = effect.pipe(
      Effect.retry(Schedule.recurs(config.retries).pipe(
        Schedule.intersect(Schedule.exponential(Duration.seconds(1)))
      ))
    )
  }
  
  return effect
}

// Usage examples
const gitCommands = {
  status: (repoPath: string) => 
    createCommand("git", ["status", "--porcelain"], {
      workingDirectory: repoPath,
      timeout: Duration.seconds(10)
    }),
    
  commit: (repoPath: string, message: string) =>
    createCommand("git", ["commit", "-m", message], {
      workingDirectory: repoPath,
      timeout: Duration.seconds(30),
      retries: 1
    }),
    
  push: (repoPath: string, branch: string = "main") =>
    createCommand("git", ["push", "origin", branch], {
      workingDirectory: repoPath,
      timeout: Duration.minutes(5),
      retries: 2
    })
}
```

### Pattern 2: Command Result Processing and Validation

Create processors for common command output patterns:

```typescript
interface ProcessResult<T> {
  readonly success: boolean
  readonly data?: T
  readonly error?: string
  readonly exitCode: number
  readonly duration: number
}

// Generic command processor
const processCommand = <T>(
  command: Effect.Effect<string, any, any>,
  parser: (output: string) => T,
  validator?: (data: T) => boolean
): Effect.Effect<ProcessResult<T>> =>
  Effect.gen(function* () {
    const startTime = Date.now()
    
    try {
      const output = yield* command
      const data = parser(output)
      const isValid = validator ? validator(data) : true
      
      return {
        success: isValid,
        data: isValid ? data : undefined,
        error: isValid ? undefined : "Validation failed",
        exitCode: 0,
        duration: Date.now() - startTime
      }
    } catch (error: any) {
      return {
        success: false,
        error: error.message || String(error),
        exitCode: error.exitCode || -1,
        duration: Date.now() - startTime
      }
    }
  })

// Specific parsers for common commands
const parsers = {
  // Parse `ls -la` output into file information
  parseFileList: (output: string) => {
    return output
      .split('\n')
      .filter(line => line.trim() !== '' && !line.startsWith('total'))
      .map(line => {
        const parts = line.split(/\s+/)
        return {
          permissions: parts[0],
          size: parseInt(parts[4]) || 0,
          name: parts.slice(8).join(' '),
          isDirectory: parts[0].startsWith('d')
        }
      })
  },
  
  // Parse `ps aux` output
  parseProcessList: (output: string) => {
    const lines = output.split('\n')
    const header = lines[0]
    return lines.slice(1)
      .filter(line => line.trim() !== '')
      .map(line => {
        const parts = line.split(/\s+/)
        return {
          user: parts[0],
          pid: parseInt(parts[1]),
          cpu: parseFloat(parts[2]),
          memory: parseFloat(parts[3]),
          command: parts.slice(10).join(' ')
        }
      })
  },
  
  // Parse JSON output
  parseJson: <T>(output: string): T => JSON.parse(output),
  
  // Parse key=value pairs
  parseKeyValue: (output: string) => {
    const result: Record<string, string> = {}
    output.split('\n')
      .filter(line => line.includes('='))
      .forEach(line => {
        const [key, ...valueParts] = line.split('=')
        result[key.trim()] = valueParts.join('=').trim()
      })
    return result
  }
}

// Example usage with validation
const getSystemInfo = Effect.gen(function* () {
  // Get memory info with validation
  const memoryResult = yield* processCommand(
    createCommand("free", ["-m"]),
    (output) => {
      const lines = output.split('\n')
      const memLine = lines.find(line => line.startsWith('Mem:'))
      if (!memLine) throw new Error("Memory info not found")
      
      const values = memLine.split(/\s+/)
      return {
        total: parseInt(values[1]),
        used: parseInt(values[2]),
        free: parseInt(values[3])
      }
    },
    (data) => data.total > 0 && data.used >= 0 && data.free >= 0
  )
  
  // Get disk info with validation
  const diskResult = yield* processCommand(
    createCommand("df", ["-h", "/"]),
    parsers.parseKeyValue,
    (data) => Object.keys(data).length > 0
  )
  
  return {
    memory: memoryResult,
    disk: diskResult,
    timestamp: new Date()
  }
})
```

### Pattern 3: Command Orchestration with Dependencies

Manage complex workflows with proper dependency management and parallel execution:

```typescript
interface TaskDefinition {
  readonly id: string
  readonly name: string
  readonly command: Effect.Effect<string, any, any>
  readonly dependencies: string[]
  readonly parallel?: boolean
  readonly timeout?: Duration.Duration
  readonly retries?: number
}

// Task orchestrator that handles dependencies
const TaskOrchestrator = {
  // Execute tasks respecting dependencies
  executeTasks: (tasks: TaskDefinition[]) =>
    Effect.gen(function* () {
      const results = new Map<string, any>()
      const completed = new Set<string>()
      const running = new Set<string>()
      
      // Validate dependencies
      for (const task of tasks) {
        for (const dep of task.dependencies) {
          if (!tasks.find(t => t.id === dep)) {
            return yield* Effect.fail(new Error(`Task ${task.id} depends on unknown task ${dep}`))
          }
        }
      }
      
      const canRun = (task: TaskDefinition): boolean => {
        return task.dependencies.every(dep => completed.has(dep)) && !running.has(task.id)
      }
      
      const runTask = (task: TaskDefinition) =>
        Effect.gen(function* () {
          running.add(task.id)
          yield* Effect.logInfo(`Starting task: ${task.name}`)
          
          let taskEffect = task.command
          
          if (task.timeout) {
            taskEffect = taskEffect.pipe(Effect.timeout(task.timeout))
          }
          
          if (task.retries && task.retries > 0) {
            taskEffect = taskEffect.pipe(
              Effect.retry(Schedule.recurs(task.retries))
            )
          }
          
          try {
            const result = yield* taskEffect
            results.set(task.id, result)
            completed.add(task.id)
            running.delete(task.id)
            yield* Effect.logInfo(`Completed task: ${task.name}`)
            return result
          } catch (error) {
            running.delete(task.id)
            yield* Effect.logError(`Failed task: ${task.name} - ${error}`)
            return yield* Effect.fail(error)
          }
        })
      
      // Execute tasks in waves based on dependencies
      while (completed.size < tasks.length) {
        const runnableTasks = tasks.filter(task => 
          !completed.has(task.id) && !running.has(task.id) && canRun(task)
        )
        
        if (runnableTasks.length === 0) {
          if (running.size === 0) {
            return yield* Effect.fail(new Error("Circular dependency detected or no runnable tasks"))
          }
          // Wait for running tasks to complete
          yield* Effect.sleep(Duration.millis(100))
          continue
        }
        
        // Group parallel tasks
        const parallelTasks = runnableTasks.filter(task => task.parallel)
        const sequentialTasks = runnableTasks.filter(task => !task.parallel)
        
        // Run parallel tasks concurrently
        if (parallelTasks.length > 0) {
          yield* Effect.all(
            parallelTasks.map(runTask),
            { concurrency: parallelTasks.length }
          )
        }
        
        // Run sequential tasks one by one
        for (const task of sequentialTasks) {
          yield* runTask(task)
        }
      }
      
      return Object.fromEntries(results)
    })
}

// Example: CI/CD Pipeline
const cicdTasks: TaskDefinition[] = [
  {
    id: "install",
    name: "Install Dependencies",
    command: createCommand("npm", ["ci"]),
    dependencies: [],
    timeout: Duration.minutes(5)
  },
  {
    id: "lint",
    name: "Lint Code",
    command: createCommand("npm", ["run", "lint"]),
    dependencies: ["install"],
    parallel: true,
    timeout: Duration.minutes(2)
  },
  {
    id: "test-unit",
    name: "Unit Tests",
    command: createCommand("npm", ["run", "test:unit"]),
    dependencies: ["install"],
    parallel: true,
    timeout: Duration.minutes(10)
  },
  {
    id: "test-integration",
    name: "Integration Tests",
    command: createCommand("npm", ["run", "test:integration"]),
    dependencies: ["install"],
    parallel: true,
    timeout: Duration.minutes(15)
  },
  {
    id: "build",
    name: "Build Application",
    command: createCommand("npm", ["run", "build"]),
    dependencies: ["lint", "test-unit", "test-integration"],
    timeout: Duration.minutes(5)
  },
  {
    id: "deploy-staging",
    name: "Deploy to Staging",
    command: createCommand("npm", ["run", "deploy:staging"]),
    dependencies: ["build"],
    timeout: Duration.minutes(10)
  },
  {
    id: "test-e2e",
    name: "E2E Tests",
    command: createCommand("npm", ["run", "test:e2e"]),
    dependencies: ["deploy-staging"],
    timeout: Duration.minutes(20)
  },
  {
    id: "deploy-production",
    name: "Deploy to Production",
    command: createCommand("npm", ["run", "deploy:production"]),
    dependencies: ["test-e2e"],
    timeout: Duration.minutes(15)
  }
]

// Run the CI/CD pipeline
const runPipeline = TaskOrchestrator.executeTasks(cicdTasks).pipe(
  Effect.withSpan("cicd.pipeline"),
  Effect.catchAll(error => 
    Effect.gen(function* () {
      yield* Effect.logError(`Pipeline failed: ${error}`)
      return yield* Effect.fail(error)
    })
  )
)
```

## Integration Examples

### Integration with Popular Build Tools

Integrate Command with popular build systems and task runners:

```typescript
// Webpack build integration
const WebpackIntegration = {
  build: (configPath: string, mode: "development" | "production") =>
    Effect.gen(function* () {
      const command = Command.make("npx", "webpack", "--config", configPath, "--mode", mode)
      
      // Stream webpack output for progress tracking
      yield* Command.streamLines(command).pipe(
        Stream.tap(line => {
          if (line.includes("%")) {
            // Extract progress percentage
            const match = line.match(/(\d+)%/)
            if (match) {
              return Effect.logInfo(`Build progress: ${match[1]}%`)
            }
          }
          return Effect.unit
        }),
        Stream.runDrain
      )
      
      yield* Effect.logInfo(`Webpack build completed in ${mode} mode`)
    }),
    
  // Watch mode with hot reloading
  watch: (configPath: string) =>
    Effect.gen(function* () {
      const command = Command.make("npx", "webpack", "--config", configPath, "--watch")
      
      yield* Command.streamLines(command).pipe(
        Stream.tap(line => {
          if (line.includes("compiled successfully")) {
            return Effect.logInfo("✅ Compilation successful")
          } else if (line.includes("Failed to compile")) {
            return Effect.logError("❌ Compilation failed")
          }
          return Effect.unit
        }),
        Stream.take(1000), // Limit for example
        Stream.runDrain
      )
    })
}

// Vite integration
const ViteIntegration = {
  dev: (port: number = 3000) =>
    Effect.gen(function* () {
      const command = Command.make("npx", "vite", "--port", port.toString())
      
      let serverReady = false
      yield* Command.streamLines(command).pipe(
        Stream.tap(line => {
          if (line.includes("Local:") && !serverReady) {
            serverReady = true
            return Effect.logInfo(`🚀 Development server ready on port ${port}`)
          }
          return Effect.unit
        }),
        Stream.take(100), // Just for startup
        Stream.runDrain
      )
    }),
    
  build: (outDir: string = "dist") =>
    createCommand("npx", ["vite", "build", "--outDir", outDir], {
      timeout: Duration.minutes(10)
    }),
    
  preview: (port: number = 4173) =>
    createCommand("npx", ["vite", "preview", "--port", port.toString()])
}
```

### Testing Strategies

Comprehensive testing approaches for command-based applications:

```typescript
import { Command } from "@effect/platform"
import { Effect, TestContext, Layer } from "effect"
import { describe, test, expect } from "@effect/vitest"

// Mock command executor for testing
interface MockCommandExecutor {
  readonly responses: Map<string, string>
  readonly setResponse: (command: string, response: string) => void
  readonly setError: (command: string, error: Error) => void
}

const makeMockCommandExecutor = (): MockCommandExecutor => {
  const responses = new Map<string, string>()
  const errors = new Map<string, Error>()
  
  return {
    responses,
    setResponse: (command: string, response: string) => {
      responses.set(command, response)
    },
    setError: (command: string, error: Error) => {
      errors.set(command, error)
    }
  }
}

// Test suite for command operations
describe("Command Integration Tests", () => {
  test("should execute git status successfully", () =>
    Effect.gen(function* () {
      const mockOutput = "On branch main\nnothing to commit, working tree clean"
      
      // Mock the git status command
      const gitStatus = Effect.succeed(mockOutput)
      
      const result = yield* gitStatus
      expect(result).toContain("working tree clean")
    })
  )
  
  test("should handle command failures gracefully", () =>
    Effect.gen(function* () {
      const failingCommand = Effect.fail(new Error("Command not found"))
      
      const result = yield* failingCommand.pipe(
        Effect.catchAll(error => Effect.succeed(`Error: ${error.message}`))
      )
      
      expect(result).toBe("Error: Command not found")
    })
  )
  
  test("should timeout long-running commands", () =>
    Effect.gen(function* () {
      const longRunningCommand = Effect.sleep(Duration.seconds(10)).pipe(
        Effect.as("This should not complete")
      )
      
      const result = yield* longRunningCommand.pipe(
        Effect.timeout(Duration.seconds(1)),
        Effect.catchAll(() => Effect.succeed("Timed out as expected"))
      )
      
      expect(result).toBe("Timed out as expected")
    })
  )
})

// Property-based testing for command validation
const commandValidationTests = Effect.gen(function* () {
  // Test command argument validation
  const validCommands = ["ls", "pwd", "echo", "cat"]
  const invalidCommands = ["", " ", "../../bin/bash", "rm -rf /"]
  
  for (const cmd of validCommands) {
    const command = Command.make(cmd)
    expect(command).toBeDefined()
  }
  
  for (const cmd of invalidCommands) {
    try {
      const command = Command.make(cmd)
      // Should validate command safety
    } catch (error) {
      expect(error).toBeDefined()
    }
  }
})

// Integration test with real commands (use sparingly)
const realCommandTests = Effect.gen(function* () {
  // Only run safe, read-only commands in tests
  const echoTest = yield* Command.string(Command.make("echo", "test"))
  expect(echoTest.trim()).toBe("test")
  
  const pwdTest = yield* Command.string(Command.make("pwd"))
  expect(pwdTest.trim()).toMatch(/^\//)
  
  // Test command chaining
  const pipeline = Effect.gen(function* () {
    const date = yield* Command.string(Command.make("date", "+%Y"))
    const currentYear = parseInt(date.trim())
    expect(currentYear).toBeGreaterThan(2020)
    
    return currentYear
  })
  
  yield* pipeline
})
```

## Conclusion

Command provides **type-safe command execution**, **comprehensive error handling**, and **powerful composition** for building reliable automation tools and system integrations.

Key benefits:
- **Type Safety**: Compile-time guarantees about command success and output formats
- **Error Context**: Rich error information with exit codes, stderr, and operation context  
- **Composability**: Natural integration with Effect's ecosystem for timeouts, retries, and resource management
- **Cross-Platform**: Consistent behavior across Windows and Unix systems with automatic platform detection
- **Streaming Support**: Efficient handling of large outputs and long-running processes
- **Resource Management**: Automatic cleanup and proper process lifecycle management

Command excels when building CI/CD pipelines, system administration tools, build automation, container orchestration, and any application requiring reliable process execution with robust error handling and composition capabilities.

The module's integration with Effect's broader ecosystem makes it particularly powerful for complex workflows requiring coordination between multiple system operations, external services, and resource management.