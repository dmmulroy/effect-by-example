# Worker: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem Worker Solves

Modern applications often need to perform CPU-intensive tasks that can block the main thread, causing poor user experience and performance issues. Traditional approaches suffer from several problems:

```typescript
// Traditional approach - blocking the main thread
function processLargeDataset(data: number[]): number[] {
  // This blocks the main thread for potentially seconds
  const result = data.map(n => {
    // Simulate intensive computation
    let sum = 0;
    for (let i = 0; i < 1000000; i++) {
      sum += Math.sqrt(n * i);
    }
    return sum;
  });
  return result;
}

// This freezes the UI and blocks other operations
const result = processLargeDataset(largeArray);
console.log("Done processing"); // Only runs after blocking operation
```

This approach leads to:
- **Thread Blocking** - CPU-intensive tasks freeze the main thread
- **Poor User Experience** - UI becomes unresponsive during computation
- **Resource Waste** - Can't utilize multiple CPU cores effectively
- **No Parallelization** - Tasks run sequentially, not in parallel
- **Memory Issues** - Large datasets can cause memory pressure on main thread

### The Worker Solution

Effect's Worker module provides a type-safe, composable way to run tasks in separate threads (Web Workers in browsers, worker_threads in Node.js) with proper error handling and resource management.

```typescript
import { Worker } from "@effect/platform"
import { Effect, Stream } from "effect"

// Create a worker that processes data in parallel
const processDataWorker = Worker.makePool({
  size: 4, // Use 4 worker threads
  worker: Worker.make({
    execute: (data: number[]) => Effect.succeed(
      data.map(n => Math.sqrt(n * 1000))
    )
  })
})

// Non-blocking execution with automatic load balancing
const processInParallel = (chunks: number[][]) => Effect.gen(function* () {
  const pool = yield* processDataWorker
  const results = yield* Effect.all(
    chunks.map(chunk => pool.executeEffect(chunk)),
    { concurrency: "unbounded" }
  )
  return results.flat()
})
```

### Key Concepts

**Worker**: A single worker thread that can execute tasks independently from the main thread

**WorkerPool**: A managed pool of workers that automatically distributes tasks and handles load balancing

**SerializedWorker**: A worker that uses Effect's Schema system for type-safe message serialization

**Spawner**: A factory function that creates new worker instances

**WorkerManager**: Manages the lifecycle of workers and provides coordination

## Basic Usage Patterns

### Pattern 1: Simple Worker Creation

```typescript
import { Worker } from "@effect/platform"
import { Effect } from "effect"

// Define the worker's task
interface ComputeTask {
  numbers: number[]
  operation: 'sum' | 'average' | 'max'
}

interface ComputeResult {
  result: number
  processedCount: number
}

// Create a basic worker
const computeWorker = Worker.make<ComputeTask, ComputeResult>({
  execute: (task) => Effect.gen(function* () {
    const { numbers, operation } = task
    
    let result: number
    switch (operation) {
      case 'sum':
        result = numbers.reduce((acc, n) => acc + n, 0)
        break
      case 'average':
        result = numbers.reduce((acc, n) => acc + n, 0) / numbers.length
        break
      case 'max':
        result = Math.max(...numbers)
        break
    }
    
    return {
      result,
      processedCount: numbers.length
    }
  })
})
```

### Pattern 2: Worker Pool for Parallel Processing

```typescript
import { Worker } from "@effect/platform"
import { Effect, Layer } from "effect"

// Create a worker pool for CPU-intensive tasks
const imageProcessingPool = Worker.makePoolLayer({
  size: 4, // Number of workers
  worker: Worker.make<ImageProcessingTask, ProcessedImage>({
    execute: (task) => Effect.gen(function* () {
      // Simulate image processing
      const processed = yield* processImage(task.imageData, task.filters)
      return {
        id: task.id,
        processedData: processed,
        timestamp: Date.now()
      }
    })
  })
})

// Usage with automatic load balancing
const processBatch = (images: ImageProcessingTask[]) => Effect.gen(function* () {
  const pool = yield* Worker.WorkerPool
  const results = yield* Effect.all(
    images.map(image => pool.executeEffect(image)),
    { concurrency: 4 } // Process up to 4 images simultaneously
  )
  return results
}).pipe(
  Effect.provide(imageProcessingPool)
)
```

### Pattern 3: Serialized Workers with Schema

```typescript
import { Worker } from "@effect/platform"
import { Schema } from "@effect/schema"
import { Effect } from "effect"

// Define request/response schemas
class DataAnalysisRequest extends Schema.TaggedClass<DataAnalysisRequest>()(
  "DataAnalysisRequest",
  {
    dataset: Schema.Array(Schema.Number),
    analysisType: Schema.Literal("correlation", "regression", "clustering"),
    options: Schema.Struct({
      precision: Schema.Number,
      iterations: Schema.Number
    })
  }
) {}

class DataAnalysisResponse extends Schema.TaggedClass<DataAnalysisResponse>()(
  "DataAnalysisResponse",
  {
    result: Schema.Unknown,
    metrics: Schema.Struct({
      processingTime: Schema.Number,
      memoryUsed: Schema.Number
    })
  }
) {}

// Create a serialized worker with type safety
const analyticsWorker = Worker.makeSerialized<
  DataAnalysisRequest,
  DataAnalysisResponse
>({
  // Worker will automatically serialize/deserialize messages
  execute: (request) => Effect.gen(function* () {
    const startTime = Date.now()
    const startMemory = process.memoryUsage().heapUsed
    
    const result = yield* performAnalysis(
      request.dataset,
      request.analysisType,
      request.options
    )
    
    return new DataAnalysisResponse({
      result,
      metrics: {
        processingTime: Date.now() - startTime,
        memoryUsed: process.memoryUsage().heapUsed - startMemory
      }
    })
  })
})
```

## Real-World Examples

### Example 1: Image Processing Pipeline

Modern web applications often need to process images on the client side - resizing, filtering, format conversion. This is perfect for workers since image processing is CPU-intensive.

```typescript
import { Worker } from "@effect/platform"
import { Effect, Stream, Chunk } from "effect"
import { Schema } from "@effect/schema"

// Define image processing operations
interface ImageTask {
  id: string
  imageData: ArrayBuffer
  operations: Array<{
    type: 'resize' | 'blur' | 'brightness' | 'contrast'
    params: Record<string, number>
  }>
  outputFormat: 'jpeg' | 'png' | 'webp'
  quality: number
}

interface ProcessedImage {
  id: string
  processedData: ArrayBuffer
  metadata: {
    originalSize: number
    processedSize: number
    processingTime: number
  }
}

// Create image processing worker
const imageProcessor = Worker.make<ImageTask, ProcessedImage>({
  execute: (task) => Effect.gen(function* () {
    const startTime = Date.now()
    const originalSize = task.imageData.byteLength
    
    // Apply each operation sequentially
    let currentData = task.imageData
    for (const operation of task.operations) {
      currentData = yield* applyImageOperation(currentData, operation)
    }
    
    // Convert to output format
    const finalData = yield* convertImageFormat(
      currentData,
      task.outputFormat,
      task.quality
    )
    
    return {
      id: task.id,
      processedData: finalData,
      metadata: {
        originalSize,
        processedSize: finalData.byteLength,
        processingTime: Date.now() - startTime
      }
    }
  })
})

// Create a pool for parallel image processing
const imageProcessingPool = Worker.makePoolLayer({
  size: navigator.hardwareConcurrency || 4, // Use available CPU cores
  worker: imageProcessor,
  concurrency: 2 // Process 2 images per worker simultaneously
})

// Process multiple images in parallel
const processBatch = (images: File[]) => Effect.gen(function* () {
  const pool = yield* Worker.WorkerPool
  
  // Convert files to tasks
  const tasks = yield* Effect.all(
    images.map((file, index) => Effect.gen(function* () {
      const arrayBuffer = yield* Effect.promise(() => file.arrayBuffer())
      return {
        id: `image-${index}`,
        imageData: arrayBuffer,
        operations: [
          { type: 'resize' as const, params: { width: 800, height: 600 } },
          { type: 'brightness' as const, params: { level: 1.1 } }
        ],
        outputFormat: 'webp' as const,
        quality: 0.8
      }
    }))
  )
  
  // Process all images with progress tracking
  const results = yield* Effect.all(
    tasks.map(task => 
      pool.executeEffect(task).pipe(
        Effect.tap(result => 
          Effect.log(`Processed ${result.id}: ${result.metadata.processingTime}ms`)
        )
      )
    ),
    { concurrency: "unbounded" }
  )
  
  return results
}).pipe(
  Effect.provide(imageProcessingPool),
  Effect.withSpan("image-batch-processing")
)

// Helper functions (implementation would use canvas/ImageData APIs)
const applyImageOperation = (
  data: ArrayBuffer,
  operation: { type: string; params: Record<string, number> }
): Effect.Effect<ArrayBuffer, never> => {
  // Implementation would use Canvas API or image processing library
  return Effect.succeed(data) // Simplified
}

const convertImageFormat = (
  data: ArrayBuffer,
  format: string,
  quality: number
): Effect.Effect<ArrayBuffer, never> => {
  // Implementation would use Canvas API to convert formats
  return Effect.succeed(data) // Simplified
}
```

### Example 2: Large Dataset Computation

Data analysis applications often need to process large datasets for statistical analysis, machine learning, or reporting.

```typescript
import { Worker } from "@effect/platform"
import { Effect, Stream, Chunk } from "effect"

interface DatasetChunk {
  id: string
  data: number[]
  computationType: 'statistics' | 'correlation' | 'regression'
  parameters: Record<string, any>
}

interface ComputationResult {
  chunkId: string
  result: {
    mean?: number
    median?: number
    stdDev?: number
    correlation?: number[][]
    rSquared?: number
    coefficients?: number[]
  }
  processingTime: number
  memoryPeak: number
}

// Worker for statistical computations
const statisticsWorker = Worker.make<DatasetChunk, ComputationResult>({
  execute: (chunk) => Effect.gen(function* () {
    const startTime = Date.now()
    const startMemory = getMemoryUsage()
    
    let result: ComputationResult['result'] = {}
    
    switch (chunk.computationType) {
      case 'statistics':
        result = yield* computeStatistics(chunk.data)
        break
      case 'correlation':
        result = yield* computeCorrelation(chunk.data, chunk.parameters)
        break
      case 'regression':
        result = yield* computeRegression(chunk.data, chunk.parameters)
        break
    }
    
    const endMemory = getMemoryUsage()
    
    return {
      chunkId: chunk.id,
      result,
      processingTime: Date.now() - startTime,
      memoryPeak: endMemory - startMemory
    }
  })
})

// Create a pool for parallel computation
const computationPool = Worker.makePoolLayer({
  size: 6, // More workers since computations are CPU-intensive
  worker: statisticsWorker,
  concurrency: 1, // One computation per worker to avoid memory issues
  targetUtilization: 0.8 // Keep workers busy but not overwhelmed
})

// Process large dataset in chunks
const processLargeDataset = (
  dataset: number[],
  chunkSize: number = 10000
) => Effect.gen(function* () {
  const pool = yield* Worker.WorkerPool
  
  // Split dataset into chunks
  const chunks = Chunk.fromIterable(dataset)
    .piped(data => Chunk.chunksOf(data, chunkSize))
    .piped(chunks => 
      Chunk.mapWithIndex(chunks, (chunk, index) => ({
        id: `chunk-${index}`,
        data: Chunk.toReadonlyArray(chunk),
        computationType: 'statistics' as const,
        parameters: {}
      }))
    )
  
  // Process chunks in parallel with progress tracking
  const results = yield* Stream.fromIterable(chunks)
    .piped(
      Stream.mapEffect(chunk => 
        pool.executeEffect(chunk).pipe(
          Effect.tap(result => 
            Effect.log(`Completed chunk ${result.chunkId} in ${result.processingTime}ms`)
          )
        )
      )
    )
    .piped(Stream.buffer({ capacity: 16 })) // Buffer results
    .piped(Stream.runCollect)
  
  // Combine results from all chunks
  const combined = yield* combineChunkResults(Chunk.toReadonlyArray(results))
  
  return combined
}).pipe(
  Effect.provide(computationPool),
  Effect.withSpan("large-dataset-processing")
)

// Helper functions for computations
const computeStatistics = (data: number[]): Effect.Effect<ComputationResult['result'], never> =>
  Effect.gen(function* () {
    const sorted = data.slice().sort((a, b) => a - b)
    const mean = data.reduce((sum, val) => sum + val, 0) / data.length
    const median = sorted[Math.floor(sorted.length / 2)]
    const variance = data.reduce((sum, val) => sum + Math.pow(val - mean, 2), 0) / data.length
    const stdDev = Math.sqrt(variance)
    
    return { mean, median, stdDev }
  })

const computeCorrelation = (
  data: number[],
  params: Record<string, any>
): Effect.Effect<ComputationResult['result'], never> =>
  Effect.succeed({ correlation: [[1.0]] }) // Simplified

const computeRegression = (
  data: number[],
  params: Record<string, any>
): Effect.Effect<ComputationResult['result'], never> =>
  Effect.succeed({ rSquared: 0.85, coefficients: [1.2, 0.8] }) // Simplified

const combineChunkResults = (results: ComputationResult[]) =>
  Effect.succeed({
    totalChunks: results.length,
    totalProcessingTime: results.reduce((sum, r) => sum + r.processingTime, 0),
    averageProcessingTime: results.reduce((sum, r) => sum + r.processingTime, 0) / results.length,
    combinedStatistics: results.reduce((acc, r) => ({ ...acc, ...r.result }), {})
  })

const getMemoryUsage = (): number => {
  if (typeof process !== 'undefined') {
    return process.memoryUsage().heapUsed
  }
  return 0 // Browser fallback
}
```

### Example 3: Background Task Queue

Many applications need to process tasks in the background - sending emails, generating reports, processing uploads, etc.

```typescript
import { Worker } from "@effect/platform"
import { Effect, Queue, Stream, Schedule } from "effect"
import { Schema } from "@effect/schema"

// Define task types
abstract class BackgroundTask extends Schema.TaggedClass<BackgroundTask>()(
  "BackgroundTask",
  {
    id: Schema.String,
    priority: Schema.Number,
    createdAt: Schema.DateFromString,
    maxRetries: Schema.Number.pipe(Schema.int(), Schema.positive())
  }
) {}

class EmailTask extends BackgroundTask.extend<EmailTask>()("EmailTask", {
  to: Schema.String,
  subject: Schema.String,
  body: Schema.String,
  attachments: Schema.Array(Schema.String)
}) {}

class ReportGenerationTask extends BackgroundTask.extend<ReportGenerationTask>()(
  "ReportGenerationTask",
  {
    reportType: Schema.Literal("sales", "analytics", "audit"),
    dateRange: Schema.Struct({
      start: Schema.DateFromString,
      end: Schema.DateFromString
    }),
    filters: Schema.Record(Schema.String, Schema.Unknown)
  }
) {}

class FileProcessingTask extends BackgroundTask.extend<FileProcessingTask>()(
  "FileProcessingTask",
  {
    filePath: Schema.String,
    operation: Schema.Literal("compress", "convert", "analyze"),
    outputPath: Schema.String
  }
) {}

type AnyTask = EmailTask | ReportGenerationTask | FileProcessingTask

interface TaskResult {
  taskId: string
  success: boolean
  result?: any
  error?: string
  processingTime: number
  retryCount: number
}

// Create specialized workers for different task types
const emailWorker = Worker.make<EmailTask, TaskResult>({
  execute: (task) => Effect.gen(function* () {
    const startTime = Date.now()
    
    try {
      // Send email (simplified)
      yield* sendEmail(task.to, task.subject, task.body, task.attachments)
      
      return {
        taskId: task.id,
        success: true,
        result: { messageId: `email-${Date.now()}` },
        processingTime: Date.now() - startTime,
        retryCount: 0
      }
    } catch (error) {
      return {
        taskId: task.id,
        success: false,
        error: error instanceof Error ? error.message : 'Unknown error',
        processingTime: Date.now() - startTime,
        retryCount: 0
      }
    }
  })
})

const reportWorker = Worker.make<ReportGenerationTask, TaskResult>({
  execute: (task) => Effect.gen(function* () {
    const startTime = Date.now()
    
    // Generate report (simplified)
    const report = yield* generateReport(
      task.reportType,
      task.dateRange,
      task.filters
    )
    
    return {
      taskId: task.id,
      success: true,
      result: { reportPath: report.path, size: report.size },
      processingTime: Date.now() - startTime,
      retryCount: 0
    }
  })
})

const fileWorker = Worker.make<FileProcessingTask, TaskResult>({
  execute: (task) => Effect.gen(function* () {
    const startTime = Date.now()
    
    const result = yield* processFile(
      task.filePath,
      task.operation,
      task.outputPath
    )
    
    return {
      taskId: task.id,
      success: true,
      result,
      processingTime: Date.now() - startTime,
      retryCount: 0
    }
  })
})

// Create worker pools for different task types
const backgroundTaskLayer = Layer.mergeAll(
  Worker.makePoolLayer({
    size: 2,
    worker: emailWorker
  }).pipe(Layer.provide(Layer.succeed(Worker.Spawner, () => emailWorker))),
  
  Worker.makePoolLayer({
    size: 3,
    worker: reportWorker
  }).pipe(Layer.provide(Layer.succeed(Worker.Spawner, () => reportWorker))),
  
  Worker.makePoolLayer({
    size: 4,
    worker: fileWorker
  }).pipe(Layer.provide(Layer.succeed(Worker.Spawner, () => fileWorker)))
)

// Task queue processor
const processTaskQueue = (taskQueue: Queue.Queue<AnyTask>) => Effect.gen(function* () {
  const emailPool = yield* Effect.provideService(
    Worker.WorkerPool,
    Worker.Spawner,
    () => emailWorker
  )
  const reportPool = yield* Effect.provideService(
    Worker.WorkerPool,
    Worker.Spawner,
    () => reportWorker
  )
  const filePool = yield* Effect.provideService(
    Worker.WorkerPool,
    Worker.Spawner,
    () => fileWorker
  )
  
  // Process tasks from queue
  yield* Stream.fromQueue(taskQueue)
    .piped(
      Stream.mapEffect(task => Effect.gen(function* () {
        let result: TaskResult
        
        switch (task._tag) {
          case 'EmailTask':
            result = yield* emailPool.executeEffect(task)
            break
          case 'ReportGenerationTask':
            result = yield* reportPool.executeEffect(task)
            break
          case 'FileProcessingTask':
            result = yield* filePool.executeEffect(task)
            break
        }
        
        // Handle retries for failed tasks
        if (!result.success && result.retryCount < task.maxRetries) {
          yield* Effect.sleep("5 seconds")
          yield* Queue.offer(taskQueue, task) // Re-queue for retry
        }
        
        yield* Effect.log(`Task ${task.id} completed: ${result.success ? 'SUCCESS' : 'FAILED'}`)
        
        return result
      }))
    )
    .piped(Stream.runDrain)
}).pipe(
  Effect.provide(backgroundTaskLayer),
  Effect.fork // Run in background
)

// Helper functions (simplified implementations)
const sendEmail = (to: string, subject: string, body: string, attachments: string[]) =>
  Effect.succeed(void 0) // Simplified

const generateReport = (
  type: string,
  dateRange: { start: Date; end: Date },
  filters: Record<string, unknown>
) =>
  Effect.succeed({ path: `/reports/${type}-${Date.now()}.pdf`, size: 1024 * 1024 })

const processFile = (filePath: string, operation: string, outputPath: string) =>
  Effect.succeed({ outputPath, size: 1024 * 512 })
```

## Advanced Features Deep Dive

### Feature 1: Worker Pool Management

Worker pools provide automatic load balancing, resource management, and scaling capabilities.

#### Basic Pool Configuration

```typescript
import { Worker } from "@effect/platform"
import { Effect, Duration } from "effect"

// Fixed-size pool
const fixedPool = Worker.makePoolLayer({
  size: 4, // Always maintain 4 workers
  worker: myWorker,
  concurrency: 2, // Each worker can handle 2 tasks simultaneously
  targetUtilization: 0.7 // Keep workers at 70% utilization
})

// Dynamic pool with TTL
const dynamicPool = Worker.makePoolLayer({
  minSize: 2, // At least 2 workers
  maxSize: 8, // Maximum 8 workers
  timeToLive: Duration.minutes(5), // Workers timeout after 5 minutes of inactivity
  worker: myWorker,
  concurrency: 1,
  targetUtilization: 0.8
})
```

#### Real-World Pool Example: Video Processing

```typescript
interface VideoTask {
  id: string
  inputPath: string
  outputPath: string
  codec: 'h264' | 'h265' | 'vp9'
  resolution: { width: number; height: number }
  bitrate: number
}

const videoProcessor = Worker.make<VideoTask, { success: boolean; outputSize: number }>({
  execute: (task) => Effect.gen(function* () {
    // Video processing is very CPU and memory intensive
    const result = yield* processVideo(task)
    return result
  })
})

// Adaptive pool that scales based on system resources
const videoProcessingPool = Worker.makePoolLayer({
  minSize: 1,
  maxSize: Math.max(2, Math.floor((navigator.hardwareConcurrency || 4) / 2)),
  timeToLive: Duration.minutes(10), // Videos take time, longer TTL
  worker: videoProcessor,
  concurrency: 1, // One video per worker (memory intensive)
  targetUtilization: 0.6, // Conservative utilization
  onCreate: (worker) => Effect.gen(function* () {
    // Pre-warm worker with codec initialization
    yield* Effect.log(`Initializing video worker ${worker.id}`)
    yield* initializeVideoCodecs()
  })
})

const processVideo = (task: VideoTask) => Effect.gen(function* () {
  // Simulate video processing
  yield* Effect.sleep(Duration.seconds(30)) // Video processing takes time
  
  const outputSize = Math.floor(Math.random() * 1000000) + 500000
  return { success: true, outputSize }
})

const initializeVideoCodecs = () => Effect.gen(function* () {
  // Initialize video processing libraries
  yield* Effect.sleep(Duration.seconds(2))
  yield* Effect.log("Video codecs initialized")
})
```

### Feature 2: Serialized Workers with Schema

Serialized workers provide type-safe message passing with automatic serialization/deserialization.

#### Advanced Serialization: Machine Learning Pipeline

```typescript
import { Schema } from "@effect/schema"
import { Worker } from "@effect/platform"
import { Effect } from "effect"

// Define complex data structures
class TrainingData extends Schema.Class<TrainingData>("TrainingData")({
  features: Schema.Array(Schema.Array(Schema.Number)),
  labels: Schema.Array(Schema.Number),
  metadata: Schema.Struct({
    normalization: Schema.Literal("none", "standardize", "normalize"),
    splitRatio: Schema.Number.pipe(Schema.between(0, 1))
  })
}) {}

class ModelConfig extends Schema.Class<ModelConfig>("ModelConfig")({
  algorithm: Schema.Literal("linear", "neural", "random_forest"),
  hyperparameters: Schema.Record(Schema.String, Schema.Unknown),
  epochs: Schema.Number.pipe(Schema.int(), Schema.positive()),
  batchSize: Schema.Number.pipe(Schema.int(), Schema.positive())
}) {}

class TrainModelRequest extends Schema.TaggedClass<TrainModelRequest>()(
  "TrainModelRequest",
  {
    data: TrainingData,
    config: ModelConfig,
    modelId: Schema.String
  }
) {}

class TrainModelResponse extends Schema.TaggedClass<TrainModelResponse>()(
  "TrainModelResponse",
  {
    modelId: Schema.String,
    metrics: Schema.Struct({
      accuracy: Schema.Number,
      loss: Schema.Number,
      trainingTime: Schema.Number
    }),
    modelPath: Schema.String
  }
) {}

// Serialized worker with complex schemas
const mlTrainingWorker = Worker.makeSerialized<
  TrainModelRequest,
  TrainModelResponse
>({
  execute: (request) => Effect.gen(function* () {
    const startTime = Date.now()
    
    // Perform machine learning training
    const model = yield* trainModel(request.data, request.config)
    const metrics = yield* evaluateModel(model, request.data)
    const modelPath = yield* saveModel(model, request.modelId)
    
    return new TrainModelResponse({
      modelId: request.modelId,
      metrics: {
        accuracy: metrics.accuracy,
        loss: metrics.loss,
        trainingTime: Date.now() - startTime
      },
      modelPath
    })
  })
})

// Helper functions for ML operations
const trainModel = (data: TrainingData, config: ModelConfig) => Effect.gen(function* () {
  // Simulate model training
  yield* Effect.sleep(Duration.seconds(config.epochs * 0.1))
  return { weights: new Array(data.features[0].length).fill(0).map(() => Math.random()) }
})

const evaluateModel = (model: any, data: TrainingData) => Effect.gen(function* () {
  // Simulate model evaluation
  return {
    accuracy: 0.85 + Math.random() * 0.1,
    loss: Math.random() * 0.3
  }
})

const saveModel = (model: any, modelId: string) => Effect.gen(function* () {
  // Simulate saving model
  const path = `/models/${modelId}.json`
  return path
})
```

### Feature 3: Advanced Error Handling

```typescript
import { Worker } from "@effect/platform"
import { Effect, Data } from "effect"

// Custom error types for different failure modes
class WorkerTimeoutError extends Data.TaggedError("WorkerTimeoutError")<{
  workerId: number
  taskId: string
  timeoutMs: number
}> {}

class WorkerMemoryError extends Data.TaggedError("WorkerMemoryError")<{
  workerId: number
  memoryUsage: number
  maxMemory: number
}> {}

class TaskValidationError extends Data.TaggedError("TaskValidationError")<{
  taskId: string
  validationErrors: string[]
}> {}

// Robust worker with comprehensive error handling
const robustWorker = Worker.make<ProcessingTask, ProcessingResult>({
  execute: (task) => Effect.gen(function* () {
    // Validate task before processing
    const validation = yield* validateTask(task)
    if (!validation.valid) {
      return yield* Effect.fail(new TaskValidationError({
        taskId: task.id,
        validationErrors: validation.errors
      }))
    }
    
    // Monitor memory usage
    const memoryMonitor = yield* Effect.fork(
      Effect.gen(function* () {
        while (true) {
          const memory = getMemoryUsage()
          if (memory > MAX_MEMORY_THRESHOLD) {
            return yield* Effect.fail(new WorkerMemoryError({
              workerId: 0, // Would get actual worker ID
              memoryUsage: memory,
              maxMemory: MAX_MEMORY_THRESHOLD
            }))
          }
          yield* Effect.sleep(Duration.seconds(1))
        }
      })
    )
    
    // Process with timeout
    const result = yield* processTask(task).pipe(
      Effect.timeout(Duration.minutes(5)),
      Effect.mapError(error => {
        if (error._tag === "TimeoutException") {
          return new WorkerTimeoutError({
            workerId: 0,
            taskId: task.id,
            timeoutMs: 5 * 60 * 1000
          })
        }
        return error
      }),
      Effect.ensuring(Effect.interrupt(memoryMonitor)) // Clean up monitor
    )
    
    return result
  })
})

// Pool with error handling and retry logic
const resilientPool = Worker.makePoolLayer({
  size: 4,
  worker: robustWorker,
  onCreate: (worker) => Effect.gen(function* () {
    // Health check new workers
    yield* Effect.log(`Health checking worker ${worker.id}`)
    yield* healthCheckWorker(worker)
  })
})

const processWithRetry = (task: ProcessingTask) => Effect.gen(function* () {
  const pool = yield* Worker.WorkerPool
  
  return yield* pool.executeEffect(task).pipe(
    Effect.retry(
      Schedule.exponential(Duration.seconds(1)).pipe(
        Schedule.union(Schedule.recurs(3)), // Retry up to 3 times
        Schedule.whileInput((error: any) => {
          // Only retry on specific errors
          return error._tag === "WorkerTimeoutError" || 
                 error._tag === "WorkerMemoryError"
        })
      )
    ),
    Effect.catchAll(error => Effect.gen(function* () {
      // Log error and return safe fallback
      yield* Effect.logError(`Task ${task.id} failed after retries`, error)
      return createFallbackResult(task)
    }))
  )
}).pipe(
  Effect.provide(resilientPool)
)

// Helper functions
const validateTask = (task: ProcessingTask) => Effect.succeed({
  valid: task.id.length > 0,
  errors: task.id.length === 0 ? ["Task ID is required"] : []
})

const healthCheckWorker = (worker: Worker<any, any, any>) => Effect.gen(function* () {
  // Perform health check
  yield* Effect.sleep(Duration.milliseconds(100))
  yield* Effect.log(`Worker ${worker.id} is healthy`)
})

const createFallbackResult = (task: ProcessingTask): ProcessingResult => ({
  taskId: task.id,
  success: false,
  result: null,
  error: "Failed after retries"
})

const MAX_MEMORY_THRESHOLD = 1024 * 1024 * 500 // 500MB
```

## Practical Patterns & Best Practices

### Pattern 1: Resource-Aware Worker Pools

```typescript
import { Worker } from "@effect/platform"
import { Effect, Layer, Ref } from "effect"

// System resource monitor
const createResourceMonitor = Effect.gen(function* () {
  const cpuUsage = yield* Ref.make(0)
  const memoryUsage = yield* Ref.make(0)
  
  // Monitor system resources
  const monitor = Effect.gen(function* () {
    while (true) {
      const cpu = yield* getCurrentCPUUsage()
      const memory = yield* getCurrentMemoryUsage()
      
      yield* Ref.set(cpuUsage, cpu)
      yield* Ref.set(memoryUsage, memory)
      
      yield* Effect.sleep(Duration.seconds(1))
    }
  }).pipe(Effect.fork)
  
  return {
    cpuUsage,
    memoryUsage,
    monitor
  }
})

// Adaptive pool that adjusts based on system resources
const adaptiveWorkerPool = <I, O, E>(
  baseWorker: Worker.Worker<I, O, E>,
  maxWorkers: number = 8
) => Layer.effect(
  Worker.WorkerPool,
  Effect.gen(function* () {
    const resourceMonitor = yield* createResourceMonitor
    const currentPoolSize = yield* Ref.make(2) // Start with 2 workers
    
    // Adaptive scaling logic
    const scalePool = Effect.gen(function* () {
      while (true) {
        const cpu = yield* Ref.get(resourceMonitor.cpuUsage)
        const memory = yield* Ref.get(resourceMonitor.memoryUsage)
        const currentSize = yield* Ref.get(currentPoolSize)
        
        let newSize = currentSize
        
        // Scale up if resources are available
        if (cpu < 0.7 && memory < 0.8 && currentSize < maxWorkers) {
          newSize = Math.min(currentSize + 1, maxWorkers)
        }
        // Scale down if resources are constrained
        else if ((cpu > 0.9 || memory > 0.9) && currentSize > 1) {
          newSize = Math.max(currentSize - 1, 1)
        }
        
        if (newSize !== currentSize) {
          yield* Ref.set(currentPoolSize, newSize)
          yield* Effect.log(`Scaling worker pool to ${newSize} workers (CPU: ${cpu.toFixed(2)}, Memory: ${memory.toFixed(2)})`)
        }
        
        yield* Effect.sleep(Duration.seconds(5))
      }
    }).pipe(Effect.fork)
    
    return Worker.makePool({
      size: 2, // Will be dynamically adjusted
      worker: baseWorker
    })
  })
)

// Usage with resource-aware scaling
const resourceAwareProcessing = (tasks: ProcessingTask[]) => Effect.gen(function* () {
  const pool = yield* Worker.WorkerPool
  
  // Process tasks with automatic resource management
  const results = yield* Effect.all(
    tasks.map(task => pool.executeEffect(task)),
    { concurrency: "inherit" } // Let the pool manage concurrency
  )
  
  return results
}).pipe(
  Effect.provide(adaptiveWorkerPool(myWorker, 8))
)

// System resource helpers (simplified)
const getCurrentCPUUsage = (): Effect.Effect<number, never> =>
  Effect.succeed(Math.random() * 0.8) // Simplified

const getCurrentMemoryUsage = (): Effect.Effect<number, never> =>
  Effect.succeed(Math.random() * 0.6) // Simplified
```

### Pattern 2: Worker Pipeline Orchestration

```typescript
import { Worker } from "@effect/platform"
import { Effect, Stream, Chunk } from "effect"

// Multi-stage processing pipeline
interface PipelineStage<I, O> {
  name: string
  worker: Worker.Worker<I, O, any>
  batchSize: number
  concurrency: number
}

const createProcessingPipeline = <A, B, C, D>(
  stage1: PipelineStage<A, B>,
  stage2: PipelineStage<B, C>,
  stage3: PipelineStage<C, D>
) => {
  const processStage = <I, O>(
    stage: PipelineStage<I, O>,
    input: Stream.Stream<I>
  ): Stream.Stream<O> =>
    input.pipe(
      Stream.chunks,
      Stream.map(chunk => Chunk.chunksOf(chunk, stage.batchSize)),
      Stream.flatMap(chunks => 
        Stream.fromIterable(chunks).pipe(
          Stream.mapEffect(chunk =>
            Effect.all(
              Chunk.toReadonlyArray(chunk).map(item => 
                stage.worker.executeEffect(item)
              ),
              { concurrency: stage.concurrency }
            )
          ),
          Stream.map(results => Chunk.fromIterable(results))
        )
      ),
      Stream.flatMap(Stream.fromChunk)
    )
  
  return (input: Stream.Stream<A>): Stream.Stream<D> =>
    input.pipe(
      data => processStage(stage1, data),
      data => processStage(stage2, data),
      data => processStage(stage3, data)
    )
}

// Example: Document processing pipeline
interface RawDocument {
  id: string
  content: string
  metadata: Record<string, any>
}

interface ParsedDocument {
  id: string
  sections: Array<{ title: string; content: string }>
  metadata: Record<string, any>
}

interface AnalyzedDocument {
  id: string
  sections: Array<{ title: string; content: string; sentiment: number }>
  summary: string
  keywords: string[]
  metadata: Record<string, any>
}

interface ProcessedDocument {
  id: string
  summary: string
  keywords: string[]
  sentiment: number
  categories: string[]
  metadata: Record<string, any>
}

// Pipeline workers
const documentParser = Worker.make<RawDocument, ParsedDocument>({
  execute: (doc) => Effect.gen(function* () {
    // Parse document structure
    const sections = yield* parseDocumentSections(doc.content)
    return {
      id: doc.id,
      sections,
      metadata: doc.metadata
    }
  })
})

const documentAnalyzer = Worker.make<ParsedDocument, AnalyzedDocument>({
  execute: (doc) => Effect.gen(function* () {
    // Analyze content
    const analyzed = yield* analyzeDocumentContent(doc)
    return analyzed
  })
})

const documentClassifier = Worker.make<AnalyzedDocument, ProcessedDocument>({
  execute: (doc) => Effect.gen(function* () {
    // Classify and categorize
    const categories = yield* classifyDocument(doc)
    return {
      id: doc.id,
      summary: doc.summary,
      keywords: doc.keywords,
      sentiment: doc.sections.reduce((sum, s) => sum + s.sentiment, 0) / doc.sections.length,
      categories,
      metadata: doc.metadata
    }
  })
})

// Create the pipeline
const documentPipeline = createProcessingPipeline(
  { name: "parser", worker: documentParser, batchSize: 10, concurrency: 4 },
  { name: "analyzer", worker: documentAnalyzer, batchSize: 5, concurrency: 2 },
  { name: "classifier", worker: documentClassifier, batchSize: 8, concurrency: 3 }
)

// Process documents through pipeline
const processDocuments = (documents: RawDocument[]) => Effect.gen(function* () {
  const results = yield* Stream.fromIterable(documents)
    .pipe(documentPipeline)
    .pipe(Stream.runCollect)
  
  return Chunk.toReadonlyArray(results)
})

// Helper functions (simplified)
const parseDocumentSections = (content: string) => Effect.succeed([
  { title: "Introduction", content: content.slice(0, 100) }
])

const analyzeDocumentContent = (doc: ParsedDocument) => Effect.succeed({
  ...doc,
  sections: doc.sections.map(s => ({ ...s, sentiment: Math.random() * 2 - 1 })),
  summary: "Document summary",
  keywords: ["keyword1", "keyword2"]
})

const classifyDocument = (doc: AnalyzedDocument) => Effect.succeed(["category1", "category2"])
```

### Pattern 3: Worker Health Monitoring

```typescript
import { Worker } from "@effect/platform"
import { Effect, Stream, Ref, Schedule } from "effect"

interface WorkerHealth {
  workerId: number
  isHealthy: boolean
  lastHealthCheck: Date
  metrics: {
    tasksCompleted: number
    averageResponseTime: number
    errorRate: number
    memoryUsage: number
  }
}

const createHealthMonitor = <I, O, E>(
  pool: Worker.WorkerPool<I, O, E>
) => Effect.gen(function* () {
  const healthData = yield* Ref.make<Map<number, WorkerHealth>>(new Map())
  
  // Health check function
  const performHealthCheck = (workerId: number) => Effect.gen(function* () {
    const startTime = Date.now()
    
    try {
      // Send a health check task
      const testTask = createHealthCheckTask() as I
      yield* pool.executeEffect(testTask).pipe(
        Effect.timeout(Duration.seconds(5))
      )
      
      const responseTime = Date.now() - startTime
      
      // Update health data
      yield* Ref.update(healthData, map => {
        const current = map.get(workerId) || createDefaultHealth(workerId)
        map.set(workerId, {
          ...current,
          isHealthy: true,
          lastHealthCheck: new Date(),
          metrics: {
            ...current.metrics,
            averageResponseTime: (current.metrics.averageResponseTime + responseTime) / 2
          }
        })
        return map
      })
      
    } catch (error) {
      // Mark as unhealthy
      yield* Ref.update(healthData, map => {
        const current = map.get(workerId) || createDefaultHealth(workerId)
        map.set(workerId, {
          ...current,
          isHealthy: false,
          lastHealthCheck: new Date(),
          metrics: {
            ...current.metrics,
            errorRate: current.metrics.errorRate + 0.1
          }
        })
        return map
      })
    }
  })
  
  // Periodic health checks
  const healthCheckSchedule = Effect.gen(function* () {
    // Get all worker IDs (simplified - would need actual worker tracking)
    const workerIds = [1, 2, 3, 4] // Example worker IDs
    
    yield* Effect.all(
      workerIds.map(id => performHealthCheck(id)),
      { concurrency: "unbounded" }
    )
  }).pipe(
    Effect.repeat(Schedule.fixed(Duration.seconds(30)))
  )
  
  return {
    healthData,
    startMonitoring: Effect.fork(healthCheckSchedule),
    getHealth: Ref.get(healthData)
  }
})

// Usage with health monitoring
const monitoredWorkerPool = <I, O, E>(
  worker: Worker.Worker<I, O, E>,
  poolSize: number
) => Layer.effect(
  Worker.WorkerPool,
  Effect.gen(function* () {
    const pool = yield* Worker.makePool({
      size: poolSize,
      worker
    })
    
    const healthMonitor = yield* createHealthMonitor(pool)
    yield* healthMonitor.startMonitoring
    
    // Periodic health reporting
    const reporter = Effect.gen(function* () {
      while (true) {
        const health = yield* healthMonitor.getHealth
        const healthReport = Array.from(health.values())
        
        yield* Effect.log(`Worker Health Report:`)
        for (const worker of healthReport) {
          yield* Effect.log(
            `Worker ${worker.workerId}: ${worker.isHealthy ? 'HEALTHY' : 'UNHEALTHY'} ` +
            `(Tasks: ${worker.metrics.tasksCompleted}, Avg Response: ${worker.metrics.averageResponseTime}ms)`
          )
        }
        
        yield* Effect.sleep(Duration.minutes(1))
      }
    }).pipe(Effect.fork)
    
    return pool
  })
)

const createHealthCheckTask = () => ({
  type: 'health_check',
  timestamp: Date.now()
})

const createDefaultHealth = (workerId: number): WorkerHealth => ({
  workerId,
  isHealthy: true,
  lastHealthCheck: new Date(),
  metrics: {
    tasksCompleted: 0,
    averageResponseTime: 0,
    errorRate: 0,
    memoryUsage: 0
  }
})
```

## Integration Examples

### Integration with Express.js Server

```typescript
import { Worker } from "@effect/platform"
import { Effect, Layer } from "effect"
import express from "express"
import multer from "multer"

// Express app with Effect Worker integration
const app = express()
const upload = multer({ dest: 'uploads/' })

// Worker for file processing
const fileProcessor = Worker.make<FileProcessingTask, ProcessingResult>({
  execute: (task) => Effect.gen(function* () {
    // Process uploaded file
    const result = yield* processUploadedFile(task.filePath, task.options)
    return result
  })
})

const fileProcessingLayer = Worker.makePoolLayer({
  size: 4,
  worker: fileProcessor
})

// Express route with worker integration
app.post('/process-file', upload.single('file'), async (req, res) => {
  if (!req.file) {
    return res.status(400).json({ error: 'No file uploaded' })
  }
  
  const task: FileProcessingTask = {
    id: `file-${Date.now()}`,
    filePath: req.file.path,
    options: req.body.options ? JSON.parse(req.body.options) : {}
  }
  
  // Run Effect with worker pool
  const program = Effect.gen(function* () {
    const pool = yield* Worker.WorkerPool
    const result = yield* pool.executeEffect(task)
    return result
  }).pipe(
    Effect.provide(fileProcessingLayer)
  )
  
  try {
    const result = await Effect.runPromise(program)
    res.json(result)
  } catch (error) {
    res.status(500).json({ error: 'Processing failed' })
  }
})

const processUploadedFile = (filePath: string, options: any) => Effect.gen(function* () {
  // Simulate file processing
  yield* Effect.sleep(Duration.seconds(2))
  return {
    success: true,
    processedPath: filePath + '.processed',
    metadata: { size: 1024, format: 'processed' }
  }
})
```

### Integration with React (Browser)

```typescript
import { Worker } from "@effect/platform"
import { Effect, Stream } from "effect"
import React, { useState, useEffect } from "react"

// React component with Web Worker integration
const ImageProcessor: React.FC = () => {
  const [images, setImages] = useState<File[]>([])
  const [processing, setProcessing] = useState(false)
  const [results, setResults] = useState<ProcessedImage[]>([])
  const [progress, setProgress] = useState(0)
  
  // Web Worker for image processing
  const imageWorker = Worker.make<ImageProcessingTask, ProcessedImage>({
    execute: (task) => Effect.gen(function* () {
      // Process image in Web Worker
      const processed = yield* processImageInWorker(task)
      return processed
    })
  })
  
  const processImages = async () => {
    setProcessing(true)
    setProgress(0)
    
    const tasks = images.map((file, index) => ({
      id: `image-${index}`,
      file,
      operations: ['resize', 'filter'] as const
    }))
    
    const program = Effect.gen(function* () {
      const pool = yield* Worker.makePool({
        size: 2, // Limit workers in browser
        worker: imageWorker
      })
      
      // Process with progress tracking
      const results = yield* Stream.fromIterable(tasks)
        .pipe(
          Stream.mapEffect((task, index) => 
            pool.executeEffect(task).pipe(
              Effect.tap(() => {
                setProgress(prev => prev + (100 / tasks.length))
                return Effect.unit
              })
            )
          )
        )
        .pipe(Stream.runCollect)
      
      return results
    })
    
    try {
      const processedImages = await Effect.runPromise(program)
      setResults(processedImages)
    } catch (error) {
      console.error('Processing failed:', error)
    } finally {
      setProcessing(false)
    }
  }
  
  return (
    <div>
      <input 
        type="file" 
        multiple 
        accept="image/*"
        onChange={(e) => setImages(Array.from(e.target.files || []))}
      />
      <button onClick={processImages} disabled={processing || images.length === 0}>
        {processing ? 'Processing...' : 'Process Images'}
      </button>
      {processing && (
        <div>
          <progress value={progress} max={100} />
          <span>{Math.round(progress)}% complete</span>
        </div>
      )}
      <div>
        {results.map(result => (
          <div key={result.id}>
            <img src={result.processedDataUrl} alt={`Processed ${result.id}`} />
          </div>
        ))}
      </div>
    </div>
  )
}

const processImageInWorker = (task: ImageProcessingTask) => Effect.gen(function* () {
  // Simulate image processing
  yield* Effect.sleep(Duration.seconds(1))
  
  // In real implementation, this would use Canvas API or WebGL
  const processedDataUrl = `data:image/jpeg;base64,${btoa('processed-image-data')}`
  
  return {
    id: task.id,
    processedDataUrl,
    metadata: { originalSize: 1024, processedSize: 800 }
  }
})
```

### Testing Strategies

```typescript
import { Worker } from "@effect/platform"
import { Effect, TestServices } from "effect"
import { describe, it, expect } from "vitest"

describe("Worker Tests", () => {
  it("should process tasks correctly", async () => {
    const testWorker = Worker.make<number, number>({
      execute: (input) => Effect.succeed(input * 2)
    })
    
    const result = await Effect.runPromise(
      testWorker.executeEffect(5)
    )
    
    expect(result).toBe(10)
  })
  
  it("should handle worker pool correctly", async () => {
    const testWorker = Worker.make<number, number>({
      execute: (input) => Effect.succeed(input * input)
    })
    
    const poolLayer = Worker.makePoolLayer({
      size: 2,
      worker: testWorker
    })
    
    const program = Effect.gen(function* () {
      const pool = yield* Worker.WorkerPool
      const results = yield* Effect.all([
        pool.executeEffect(2),
        pool.executeEffect(3),
        pool.executeEffect(4)
      ])
      return results
    }).pipe(
      Effect.provide(poolLayer)
    )
    
    const results = await Effect.runPromise(program)
    expect(results).toEqual([4, 9, 16])
  })
  
  it("should handle worker errors", async () => {
    const errorWorker = Worker.make<string, never, Error>({
      execute: (input) => 
        input === "error" 
          ? Effect.fail(new Error("Test error"))
          : Effect.succeed("success" as never)
    })
    
    const result = await Effect.runPromise(
      errorWorker.executeEffect("error").pipe(
        Effect.either
      )
    )
    
    expect(result._tag).toBe("Left")
    if (result._tag === "Left") {
      expect(result.left.message).toBe("Test error")
    }
  })
  
  it("should test worker with mock services", async () => {
    const mockDatabase = {
      save: (data: any) => Effect.succeed({ id: "test-id", ...data })
    }
    
    const databaseWorker = Worker.make<any, any>({
      execute: (data) => Effect.gen(function* () {
        const db = yield* Effect.service(Database)
        return yield* db.save(data)
      })
    })
    
    const program = databaseWorker.executeEffect({ name: "test" }).pipe(
      Effect.provideService(Database, mockDatabase)
    )
    
    const result = await Effect.runPromise(program)
    expect(result).toEqual({ id: "test-id", name: "test" })
  })
})

// Test utilities for worker pools
const createTestWorkerPool = <I, O>(
  worker: Worker.Worker<I, O, any>,
  size: number = 2
) => Layer.succeed(
  Worker.WorkerPool,
  Worker.makePool({ size, worker })
)

// Mock services for testing
const Database = Context.Tag<{
  save: (data: any) => Effect.Effect<any, never>
}>()
```

## Conclusion

Worker provides powerful abstractions for parallel processing, thread management, and resource optimization in Effect applications.

Key benefits:
- **Type Safety**: Schema-based serialization ensures type-safe communication between main thread and workers
- **Resource Management**: Automatic pooling, scaling, and cleanup of worker threads
- **Error Handling**: Comprehensive error handling with retry policies and fallback mechanisms
- **Cross-Platform**: Works with both Web Workers (browser) and worker_threads (Node.js)
- **Composability**: Integrates seamlessly with other Effect modules like Stream, Layer, and Schedule

Use Worker when you need to:
- Perform CPU-intensive computations without blocking the main thread
- Process large datasets in parallel
- Handle background tasks like file processing, image manipulation, or data analysis
- Build scalable applications that can utilize multiple CPU cores effectively
- Maintain responsive user interfaces while performing heavy computations

The Worker module transforms complex thread management into composable, type-safe operations that scale with your application's needs.