# Sink: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem Sink Solves

Processing streaming data efficiently while maintaining control over memory usage, handling backpressure, and managing complex aggregation logic is a fundamental challenge in modern applications. Traditional approaches often lead to memory bloat, unclear error boundaries, and difficult-to-compose consumption logic.

```typescript
// Traditional approach - manual stream processing
import { Readable } from 'stream'

async function processLogStream(stream: Readable) {
  const errorCount = { count: 0 }
  const logLevels = new Map<string, number>()
  const firstHundredErrors: string[] = []
  
  return new Promise((resolve, reject) => {
    let buffer = ''
    
    stream.on('data', (chunk) => {
      buffer += chunk.toString()
      const lines = buffer.split('\n')
      buffer = lines.pop() || '' // Keep incomplete line
      
      for (const line of lines) {
        try {
          const log = JSON.parse(line)
          
          // Multiple manual aggregations
          if (log.level === 'ERROR') {
            errorCount.count++
            if (firstHundredErrors.length < 100) {
              firstHundredErrors.push(log.message)
            }
          }
          
          // Manual grouping logic
          logLevels.set(log.level, (logLevels.get(log.level) || 0) + 1)
          
        } catch (error) {
          // Error handling scattered throughout
          console.error('Parse error:', error)
        }
      }
    })
    
    stream.on('end', () => {
      resolve({ errorCount: errorCount.count, logLevels, firstHundredErrors })
    })
    
    stream.on('error', reject)
  })
}
```

This approach leads to:
- **Memory Accumulation** - No built-in backpressure handling for large datasets
- **Complex State Management** - Manual tracking of multiple aggregations
- **Error Handling Complexity** - Error recovery scattered throughout processing logic
- **Poor Composability** - Difficult to reuse or combine different processing strategies
- **Resource Management** - Manual cleanup and resource management

### The Sink Solution

Effect's Sink module provides a composable, type-safe, and memory-efficient approach to stream consumption with built-in backpressure handling and automatic resource management.

```typescript
import { Sink, Stream, Effect, Chunk, HashMap } from "effect"

// Define composable sinks for different aggregations
const errorCountSink = Sink.foldLeft(0, (count, log: LogEntry) =>
  log.level === 'ERROR' ? count + 1 : count
)

const logLevelsSink = Sink.collectAllToMap(
  (log: LogEntry) => log.level,
  (a: LogEntry, b: LogEntry) => a
).pipe(
  Sink.map(HashMap.size)
)

const firstHundredErrorsSink = Stream.filter((log: LogEntry) => log.level === 'ERROR').pipe(
  Stream.map(log => log.message),
  Stream.run(Sink.take(100))
)

// Compose multiple sinks efficiently
const processLogStream = (stream: Stream.Stream<LogEntry, Error>) =>
  Effect.gen(function* () {
    const errorCount = yield* Stream.run(stream, errorCountSink)
    const logLevelCount = yield* Stream.run(stream, logLevelsSink)
    const firstErrors = yield* firstHundredErrorsSink
    
    return { errorCount, logLevelCount, firstErrors }
  })
```

### Key Concepts

**Sink**: A description of a computation that consumes elements from a stream, potentially fails with errors, and produces a result with optional leftover elements.

**Backpressure**: Automatic flow control that prevents memory overflow by coordinating production and consumption rates.

**Composability**: Sinks can be combined, transformed, and reused to build complex stream processing pipelines.

**Resource Safety**: Automatic resource management ensures proper cleanup even when errors occur.

## Basic Usage Patterns

### Simple Collection Sinks

```typescript
import { Sink, Stream, Effect } from "effect"

// Collect all elements into a Chunk
const collectAllExample = Effect.gen(function* () {
  const stream = Stream.make(1, 2, 3, 4, 5)
  const result = yield* Stream.run(stream, Sink.collectAll())
  console.log(result) // Chunk([1, 2, 3, 4, 5])
})

// Take only first N elements
const takeExample = Effect.gen(function* () {
  const stream = Stream.range(1, 100)
  const result = yield* Stream.run(stream, Sink.take(5))
  console.log(result) // Chunk([1, 2, 3, 4, 5])
})

// Count elements
const countExample = Effect.gen(function* () {
  const stream = Stream.fromIterable(['a', 'b', 'c', 'd'])
  const result = yield* Stream.run(stream, Sink.count)
  console.log(result) // 4
})
```

### Aggregation Sinks

```typescript
import { Sink, Stream, Effect } from "effect"

// Sum numeric values
const sumExample = Effect.gen(function* () {
  const stream = Stream.make(10, 20, 30, 40)
  const result = yield* Stream.run(stream, Sink.sum)
  console.log(result) // 100
})

// Fold with accumulator
const foldExample = Effect.gen(function* () {
  const stream = Stream.make("hello", " ", "world", "!")
  const result = yield* Stream.run(stream, 
    Sink.foldLeft("", (acc, str) => acc + str)
  )
  console.log(result) // "hello world!"
})

// Conditional folding
const conditionalFoldExample = Effect.gen(function* () {
  const stream = Stream.iterate(1, n => n + 1)
  const result = yield* Stream.run(stream,
    Sink.fold(0, sum => sum < 10, (acc, n) => acc + n)
  )
  console.log(result) // 10 (stops when sum reaches 10)
})
```

### Filtering and Transformation Sinks

```typescript
import { Sink, Stream, Effect } from "effect"

// Transform input before processing
const transformInputExample = Effect.gen(function* () {
  const stream = Stream.make("1", "2", "3", "4")
  const result = yield* Stream.run(stream,
    Sink.collectAll<number>().pipe(
      Sink.mapInput((str: string) => parseInt(str, 10))
    )
  )
  console.log(result) // Chunk([1, 2, 3, 4])
})

// Transform result after processing
const transformResultExample = Effect.gen(function* () {
  const stream = Stream.make(1, 2, 3, 4)
  const result = yield* Stream.run(stream,
    Sink.sum.pipe(
      Sink.map(total => `Total: ${total}`)
    )
  )
  console.log(result) // "Total: 10"
})
```

## Real-World Examples

### Example 1: Real-Time Log Analysis System

Processing server logs to extract insights and detect anomalies in real-time.

```typescript
import { Sink, Stream, Effect, Chunk, HashMap, Option } from "effect"
import { pipe } from "effect/Function"

interface LogEntry {
  timestamp: number
  level: 'INFO' | 'WARN' | 'ERROR' | 'DEBUG'
  service: string
  message: string
  userId?: string
  responseTime?: number
}

// Helper to create reusable log processing sinks
const LogSinks = {
  // Count logs by level
  byLevel: Sink.collectAllToMap(
    (log: LogEntry) => log.level,
    (a: LogEntry, b: LogEntry) => b // Keep latest
  ).pipe(
    Sink.map(HashMap.size)
  ),

  // Average response time for services
  avgResponseTime: Effect.gen(function* () {
    const responseTimes: Array<{ service: string; time: number }> = []
    
    return Sink.forEach((log: LogEntry) => 
      Effect.sync(() => {
        if (log.responseTime !== undefined) {
          responseTimes.push({ service: log.service, time: log.responseTime })
        }
      })
    ).pipe(
      Sink.as(responseTimes),
      Sink.map(times => {
        const serviceAvgs = new Map<string, number>()
        const serviceCounts = new Map<string, number>()
        
        for (const { service, time } of times) {
          serviceAvgs.set(service, (serviceAvgs.get(service) || 0) + time)
          serviceCounts.set(service, (serviceCounts.get(service) || 0) + 1)
        }
        
        const result = new Map<string, number>()
        for (const [service, total] of serviceAvgs) {
          result.set(service, total / (serviceCounts.get(service) || 1))
        }
        return result
      })
    )
  }),

  // Detect error spikes
  errorSpike: Sink.foldWeighted({
    initial: { errorCount: 0, totalCount: 0, windowStart: Date.now() },
    maxCost: 1000, // Process 1000 logs at a time
    cost: () => 1,
    body: (state, log: LogEntry) => ({
      errorCount: state.errorCount + (log.level === 'ERROR' ? 1 : 0),
      totalCount: state.totalCount + 1,
      windowStart: state.windowStart
    })
  }).pipe(
    Sink.map(state => ({
      errorRate: state.errorCount / state.totalCount,
      isSpike: state.errorCount / state.totalCount > 0.05 // 5% error rate threshold
    }))
  )
}

// Real-time log analysis pipeline
const analyzeLogStream = (logStream: Stream.Stream<LogEntry, Error>) =>
  Effect.gen(function* () {
    // Process multiple metrics concurrently
    const [levelCounts, avgTimes, spikeDetection] = yield* Effect.all([
      Stream.run(logStream, LogSinks.byLevel),
      Stream.run(logStream, yield* LogSinks.avgResponseTime),
      Stream.run(logStream, LogSinks.errorSpike)
    ], { concurrency: 3 })

    return {
      levelDistribution: levelCounts,
      servicePerformance: avgTimes,
      alerting: spikeDetection
    }
  })

// Usage with file or network stream
const processLogFile = (filePath: string) =>
  Effect.gen(function* () {
    const logStream = Stream.fromReadableStream(
      () => import('fs').then(fs => fs.createReadStream(filePath)),
      error => new Error(`Failed to read log file: ${error}`)
    ).pipe(
      Stream.decodeText(),
      Stream.splitLines,
      Stream.map(line => JSON.parse(line) as LogEntry)
    )

    return yield* analyzeLogStream(logStream)
  })
```

### Example 2: Financial Data Aggregation

Processing trading data to calculate various financial metrics and detect trading patterns.

```typescript
import { Sink, Stream, Effect, Chunk, Duration } from "effect"

interface Trade {
  symbol: string
  price: number
  volume: number
  timestamp: number
  side: 'BUY' | 'SELL'
}

interface MarketMetrics {
  vwap: number // Volume Weighted Average Price
  totalVolume: number
  priceRange: { high: number; low: number }
  tradeCount: number
}

// Financial calculation sinks
const FinancialSinks = {
  // Volume Weighted Average Price
  vwap: Sink.foldLeft(
    { totalValue: 0, totalVolume: 0 },
    (acc, trade: Trade) => ({
      totalValue: acc.totalValue + (trade.price * trade.volume),
      totalVolume: acc.totalVolume + trade.volume
    })
  ).pipe(
    Sink.map(acc => acc.totalVolume > 0 ? acc.totalValue / acc.totalVolume : 0)
  ),

  // Price range tracking
  priceRange: Sink.foldLeft(
    { high: 0, low: Number.MAX_VALUE },
    (range, trade: Trade) => ({
      high: Math.max(range.high, trade.price),
      low: Math.min(range.low, trade.price)
    })
  ),

  // Trading volume by time windows
  volumeByWindow: (windowSizeMs: number) => 
    Sink.foldWeighted({
      initial: new Map<number, number>(),
      maxCost: 1000,
      cost: () => 1,
      body: (windows, trade: Trade) => {
        const windowKey = Math.floor(trade.timestamp / windowSizeMs) * windowSizeMs
        windows.set(windowKey, (windows.get(windowKey) || 0) + trade.volume)
        return windows
      }
    }),

  // Detect large trades (whales)
  largeTrades: (threshold: number) =>
    Sink.collectAllWhile((trade: Trade) => trade.volume >= threshold).pipe(
      Sink.map(chunk => Chunk.toReadonlyArray(chunk))
    )
}

// Market analysis for a symbol
const analyzeMarketData = (symbol: string, tradeStream: Stream.Stream<Trade, Error>) =>
  Effect.gen(function* () {
    // Filter trades for specific symbol
    const symbolTrades = tradeStream.pipe(
      Stream.filter(trade => trade.symbol === symbol)
    )

    // Run multiple analyses concurrently
    const windowSize = Duration.toMillis(Duration.minutes(5))
    const whaleThreshold = 10000

    const [vwap, range, volumeWindows, whales] = yield* Effect.all([
      Stream.run(symbolTrades, FinancialSinks.vwap),
      Stream.run(symbolTrades, FinancialSinks.priceRange),
      Stream.run(symbolTrades, FinancialSinks.volumeByWindow(windowSize)),
      Stream.run(symbolTrades, FinancialSinks.largeTrades(whaleThreshold))
    ], { concurrency: 4 })

    const metrics: MarketMetrics = {
      vwap,
      totalVolume: Array.from(volumeWindows.values()).reduce((a, b) => a + b, 0),
      priceRange: range,
      tradeCount: whales.length
    }

    return { metrics, whales, volumeWindows }
  })

// Risk monitoring sink
const riskMonitoringSink = Sink.foldWeighted({
  initial: {
    positions: new Map<string, number>(),
    exposure: 0,
    lastUpdate: Date.now()
  },
  maxCost: 500,
  cost: (state, trade: Trade) => trade.volume > 5000 ? 10 : 1, // Higher cost for large trades
  body: (state, trade: Trade) => {
    const currentPosition = state.positions.get(trade.symbol) || 0
    const newPosition = currentPosition + (trade.side === 'BUY' ? trade.volume : -trade.volume)
    
    state.positions.set(trade.symbol, newPosition)
    
    return {
      positions: state.positions,
      exposure: Array.from(state.positions.values()).reduce((sum, pos) => sum + Math.abs(pos), 0),
      lastUpdate: trade.timestamp
    }
  }
}).pipe(
  Sink.map(state => ({
    totalExposure: state.exposure,
    positionCount: state.positions.size,
    riskLevel: state.exposure > 100000 ? 'HIGH' : state.exposure > 50000 ? 'MEDIUM' : 'LOW'
  }))
)
```

### Example 3: IoT Sensor Data Processing

Processing time-series data from IoT sensors with anomaly detection and aggregation.

```typescript
import { Sink, Stream, Effect, Chunk, Option } from "effect"

interface SensorReading {
  sensorId: string
  value: number
  timestamp: number
  location: { lat: number; lng: number }
  type: 'temperature' | 'humidity' | 'pressure'
}

interface AnomalyAlert {
  sensorId: string
  value: number
  threshold: number
  timestamp: number
  severity: 'LOW' | 'MEDIUM' | 'HIGH'
}

// IoT data processing sinks
const IoTSinks = {
  // Moving average for anomaly detection
  movingAverage: (windowSize: number) =>
    Sink.foldLeft(
      { values: [] as number[], sum: 0 },
      (state, reading: SensorReading) => {
        const newValues = [...state.values, reading.value]
        if (newValues.length > windowSize) {
          newValues.shift()
        }
        return {
          values: newValues,
          sum: newValues.reduce((a, b) => a + b, 0)
        }
      }
    ).pipe(
      Sink.map(state => state.values.length > 0 ? state.sum / state.values.length : 0)
    ),

  // Anomaly detection sink
  anomalyDetector: (thresholdMultiplier: number = 2) =>
    Effect.gen(function* () {
      const readings: SensorReading[] = []
      
      return Sink.forEach((reading: SensorReading) =>
        Effect.sync(() => {
          readings.push(reading)
          
          if (readings.length >= 10) { // Need baseline
            const recent = readings.slice(-10)
            const avg = recent.reduce((sum, r) => sum + r.value, 0) / recent.length
            const variance = recent.reduce((sum, r) => sum + Math.pow(r.value - avg, 2), 0) / recent.length
            const stdDev = Math.sqrt(variance)
            const threshold = avg + (stdDev * thresholdMultiplier)
            
            if (reading.value > threshold) {
              const severity = reading.value > threshold * 1.5 ? 'HIGH' : 
                             reading.value > threshold * 1.2 ? 'MEDIUM' : 'LOW'
              
              const alert: AnomalyAlert = {
                sensorId: reading.sensorId,
                value: reading.value,
                threshold,
                timestamp: reading.timestamp,
                severity
              }
              
              console.log('ANOMALY DETECTED:', alert)
            }
          }
        })
      ).pipe(
        Sink.as(readings.length)
      )
    }),

  // Geographical aggregation
  spatialAggregation: (gridSize: number) =>
    Sink.collectAllToMap(
      (reading: SensorReading) => {
        // Grid-based spatial binning
        const gridX = Math.floor(reading.location.lat / gridSize)
        const gridY = Math.floor(reading.location.lng / gridSize)
        return `${gridX},${gridY}`
      },
      (a: SensorReading, b: SensorReading) => ({
        ...b,
        value: (a.value + b.value) / 2 // Average values in same grid cell
      })
    ),

  // Time-series downsampling
  downsample: (intervalMs: number) =>
    Sink.foldLeft(
      new Map<number, { sum: number; count: number; sensorId: string }>(),
      (acc, reading: SensorReading) => {
        const bucket = Math.floor(reading.timestamp / intervalMs) * intervalMs
        const existing = acc.get(bucket) || { sum: 0, count: 0, sensorId: reading.sensorId }
        acc.set(bucket, {
          sum: existing.sum + reading.value,
          count: existing.count + 1,
          sensorId: reading.sensorId
        })
        return acc
      }
    ).pipe(
      Sink.map(buckets => 
        Array.from(buckets.entries()).map(([timestamp, data]) => ({
          timestamp,
          sensorId: data.sensorId,
          averageValue: data.sum / data.count,
          sampleCount: data.count
        }))
      )
    )
}

// Complete IoT data processing pipeline
const processIoTData = (sensorStream: Stream.Stream<SensorReading, Error>) =>
  Effect.gen(function* () {
    // Process different metrics concurrently
    const gridSize = 0.01 // ~1km grid cells
    const downsampleInterval = 60000 // 1 minute intervals

    const [anomalyCount, spatialData, timeSeriesData] = yield* Effect.all([
      Stream.run(sensorStream, yield* IoTSinks.anomalyDetector(2.5)),
      Stream.run(sensorStream, IoTSinks.spatialAggregation(gridSize)),
      Stream.run(sensorStream, IoTSinks.downsample(downsampleInterval))
    ], { concurrency: 3 })

    return {
      processedReadings: anomalyCount,
      spatialAggregates: spatialData,
      downsampledSeries: timeSeriesData
    }
  })

// Real-time monitoring with backpressure handling
const monitorSensors = (sensorStream: Stream.Stream<SensorReading, Error>) =>
  Effect.gen(function* () {
    // Use buffering to handle bursty data
    const bufferedStream = sensorStream.pipe(
      Stream.buffer({ capacity: 1000 }),
      Stream.mapChunks(chunk => 
        Chunk.filter(chunk, reading => 
          reading.timestamp > Date.now() - 300000 // Only last 5 minutes
        )
      )
    )

    return yield* processIoTData(bufferedStream)
  })
```

## Advanced Features Deep Dive

### Weighted Folding: Handling Variable Cost Elements

When processing elements with different computational or memory costs, `foldWeighted` provides sophisticated control over resource usage.

```typescript
import { Sink, Stream, Effect, Chunk } from "effect"

interface ProcessingTask {
  id: string
  priority: 'LOW' | 'MEDIUM' | 'HIGH'
  data: string
  estimatedCost: number
}

// Basic weighted folding
const basicWeightedExample = Effect.gen(function* () {
  const tasks = Stream.make(
    { id: '1', priority: 'HIGH', data: 'complex_task', estimatedCost: 10 },
    { id: '2', priority: 'LOW', data: 'simple_task', estimatedCost: 1 },
    { id: '3', priority: 'MEDIUM', data: 'medium_task', estimatedCost: 5 }
  )

  const result = yield* Stream.run(tasks,
    Sink.foldWeighted({
      initial: [] as ProcessingTask[],
      maxCost: 15, // Process up to 15 cost units
      cost: (_, task) => task.estimatedCost,
      body: (batch, task) => [...batch, task]
    })
  )

  console.log(`Processed batch with ${result.length} tasks`)
  return result
})

// Advanced weighted folding with decomposition
const advancedWeightedExample = Effect.gen(function* () {
  const largeTasks = Stream.make(
    { id: '1', data: 'x'.repeat(100), estimatedCost: 50 },
    { id: '2', data: 'y'.repeat(80), estimatedCost: 40 },
    { id: '3', data: 'z'.repeat(30), estimatedCost: 15 }
  )

  const result = yield* Stream.run(largeTasks,
    Sink.foldWeightedDecompose({
      initial: '',
      maxCost: 30, // Small batch size
      cost: (_, task) => task.estimatedCost,
      decompose: (task) => {
        // Split large tasks into smaller chunks
        if (task.estimatedCost > 25) {
          const chunkSize = Math.ceil(task.data.length / 2)
          return Chunk.make(
            { ...task, id: `${task.id}_a`, data: task.data.slice(0, chunkSize), estimatedCost: task.estimatedCost / 2 },
            { ...task, id: `${task.id}_b`, data: task.data.slice(chunkSize), estimatedCost: task.estimatedCost / 2 }
          )
        }
        return Chunk.make(task)
      },
      body: (acc, task) => acc + task.data
    })
  )

  console.log(`Processed data length: ${result.length}`)
  return result
})
```

### Racing and Concurrent Sinks

Combine multiple sinks to handle different processing strategies or implement failover mechanisms.

```typescript
import { Sink, Stream, Effect, Either, Duration } from "effect"

interface DataPacket {
  id: string
  data: string
  priority: number
}

// Racing sinks for fastest completion
const racingExample = Effect.gen(function* () {
  const dataStream = Stream.make(
    { id: '1', data: 'fast_data', priority: 1 },
    { id: '2', data: 'slow_data', priority: 2 }
  )

  const fastSink = Sink.take(1).pipe(
    Sink.mapEffect(chunk => 
      Effect.delay(
        Effect.succeed(`Fast: ${Chunk.size(chunk)} items`),
        Duration.millis(100)
      )
    )
  )

  const thoroughSink = Sink.collectAll<DataPacket>().pipe(
    Sink.mapEffect(chunk =>
      Effect.delay(
        Effect.succeed(`Thorough: ${Chunk.size(chunk)} items`),
        Duration.millis(500)
      )
    )
  )

  // First sink to complete wins
  const winner = yield* Stream.run(dataStream, 
    fastSink.pipe(Sink.race(thoroughSink))
  )

  console.log(`Winner: ${winner}`)
  return winner
})

// Concurrent processing with result tagging
const concurrentExample = Effect.gen(function* () {
  const dataStream = Stream.range(1, 100)

  const sumSink = Sink.sum.pipe(
    Sink.map(sum => ({ type: 'sum' as const, value: sum }))
  )

  const countSink = Sink.count.pipe(
    Sink.map(count => ({ type: 'count' as const, value: count }))
  )

  // Both sinks process concurrently, get both results
  const results = yield* Stream.run(dataStream,
    sumSink.pipe(Sink.raceBoth(countSink))
  )

  if (Either.isLeft(results)) {
    console.log('Count completed first:', results.left)
  } else {
    console.log('Sum completed first:', results.right)
  }

  return results
})

// Advanced racing with custom merge strategies
const customRacingExample = Effect.gen(function* () {
  const dataStream = Stream.fromIterable(Array.from({ length: 1000 }, (_, i) => i))

  const quickSink = Sink.take(10).pipe(
    Sink.map(chunk => ({ strategy: 'quick', sample: chunk }))
  )

  const completeSink = Sink.collectAll<number>().pipe(
    Sink.map(chunk => ({ strategy: 'complete', all: chunk }))
  )

  const result = yield* Stream.run(dataStream,
    quickSink.pipe(
      Sink.raceWith({
        other: completeSink,
        onSelfDone: (exit) => 
          Effect.succeed('Quick strategy won').pipe(Effect.as(exit)),
        onOtherDone: (exit) =>
          Effect.succeed('Complete strategy won').pipe(Effect.as(exit))
      })
    )
  )

  console.log('Racing result:', result)
  return result
})
```

### Resource Management and Finalization

Ensure proper cleanup and resource management even when processing fails.

```typescript
import { Sink, Stream, Effect, Scope, Ref } from "effect"

interface DatabaseConnection {
  query: (sql: string) => Effect.Effect<any[], Error>
  close: () => Effect.Effect<void>
}

// Resource-safe database sink
const databaseSink = (tableName: string) =>
  Effect.gen(function* () {
    const connectionCount = yield* Ref.make(0)
    
    return Sink.unwrapScoped(
      Effect.gen(function* () {
        // Acquire database connection
        yield* Ref.update(connectionCount, n => n + 1)
        console.log('Acquiring database connection')
        
        const connection: DatabaseConnection = {
          query: (sql) => Effect.succeed([{ result: `Executed: ${sql}` }]),
          close: () => Effect.sync(() => console.log('Closing database connection'))
        }
        
        // Ensure connection is closed when scope ends
        yield* Effect.addFinalizer(() => connection.close())
        
        return Sink.forEach((data: any) =>
          connection.query(`INSERT INTO ${tableName} VALUES (?)`)
        ).pipe(
          Sink.ensuring(
            Effect.sync(() => console.log('Sink processing completed'))
          )
        )
      })
    )
  })

// File processing with cleanup
const fileProcessingSink = (outputPath: string) =>
  Effect.gen(function* () {
    const fileHandle = yield* Ref.make<any>(null)
    
    return Sink.unwrapScoped(
      Effect.gen(function* () {
        // Open file for writing
        const handle = { write: (data: string) => Effect.succeed(undefined) }
        yield* Ref.set(fileHandle, handle)
        
        // Ensure file is closed when scope ends
        yield* Effect.addFinalizer(() =>
          Effect.sync(() => {
            console.log(`Closing file: ${outputPath}`)
            // handle.close() in real implementation
          })
        )
        
        return Sink.forEach((line: string) =>
          Effect.gen(function* () {
            const h = yield* Ref.get(fileHandle)
            if (h) {
              yield* h.write(line + '\n')
            }
          })
        ).pipe(
          Sink.ensuringWith((exit) =>
            Effect.sync(() => {
              console.log('File processing finished with:', exit._tag)
            })
          )
        )
      })
    )
  })

// Usage with automatic cleanup
const resourceExampleUsage = Effect.gen(function* () {
  const dataStream = Stream.make('line1', 'line2', 'line3')
  
  // Even if processing fails, resources are cleaned up
  const result = yield* Stream.run(
    dataStream,
    yield* fileProcessingSink('/tmp/output.txt')
  ).pipe(
    Effect.scoped // Ensures proper resource cleanup
  )
  
  return result
})
```

## Practical Patterns & Best Practices

### Pattern 1: Composable Analytics Pipeline

Create reusable sink components that can be combined for complex analytics.

```typescript
import { Sink, Stream, Effect, HashMap, Chunk } from "effect"
import { pipe } from "effect/Function"

// Reusable analytics components
const AnalyticsSinks = {
  // Statistical measures
  stats: <T>(getValue: (item: T) => number) =>
    Sink.foldLeft(
      { count: 0, sum: 0, min: Infinity, max: -Infinity, values: [] as number[] },
      (acc, item: T) => {
        const value = getValue(item)
        return {
          count: acc.count + 1,
          sum: acc.sum + value,
          min: Math.min(acc.min, value),
          max: Math.max(acc.max, value),
          values: [...acc.values, value]
        }
      }
    ).pipe(
      Sink.map(acc => ({
        count: acc.count,
        mean: acc.count > 0 ? acc.sum / acc.count : 0,
        min: acc.min === Infinity ? 0 : acc.min,
        max: acc.max === -Infinity ? 0 : acc.max,
        stdDev: acc.count > 1 ? Math.sqrt(
          acc.values.reduce((sum, v) => sum + Math.pow(v - acc.sum / acc.count, 2), 0) / (acc.count - 1)
        ) : 0
      }))
    ),

  // Frequency distribution
  frequency: <T>(getKey: (item: T) => string) =>
    Sink.collectAllToMap(getKey, (a: T, b: T) => b).pipe(
      Sink.map(map => HashMap.size(map))
    ),

  // Percentiles
  percentiles: <T>(getValue: (item: T) => number, percentiles: number[] = [50, 90, 95, 99]) =>
    Sink.collectAll<T>().pipe(
      Sink.map(chunk => {
        const values = Chunk.toReadonlyArray(chunk)
          .map(getValue)
          .sort((a, b) => a - b)
        
        const result: Record<string, number> = {}
        for (const p of percentiles) {
          const index = Math.ceil((p / 100) * values.length) - 1
          result[`p${p}`] = values[Math.max(0, index)] || 0
        }
        return result
      })
    ),

  // Time-based windowing
  timeWindow: <T>(
    getTimestamp: (item: T) => number,
    windowSizeMs: number,
    aggregator: Sink.Sink<any, T>
  ) =>
    Sink.foldLeft(
      new Map<number, T[]>(),
      (windows, item: T) => {
        const windowKey = Math.floor(getTimestamp(item) / windowSizeMs) * windowSizeMs
        const windowItems = windows.get(windowKey) || []
        windows.set(windowKey, [...windowItems, item])
        return windows
      }
    ).pipe(
      Sink.flatMap(windows =>
        Effect.gen(function* () {
          const results = new Map<number, any>()
          for (const [windowKey, items] of windows) {
            const result = yield* Stream.run(Stream.fromIterable(items), aggregator)
            results.set(windowKey, result)
          }
          return results
        })
      )
    )
}

// Usage example: Comprehensive data analysis
interface SalesRecord {
  timestamp: number
  amount: number
  region: string
  productCategory: string
}

const analyzeSalesData = (salesStream: Stream.Stream<SalesRecord, Error>) =>
  Effect.gen(function* () {
    // Run multiple analytics concurrently
    const [amountStats, regionFreq, percentiles, hourlyTrends] = yield* Effect.all([
      Stream.run(salesStream, AnalyticsSinks.stats((sale: SalesRecord) => sale.amount)),
      Stream.run(salesStream, AnalyticsSinks.frequency((sale: SalesRecord) => sale.region)),
      Stream.run(salesStream, AnalyticsSinks.percentiles((sale: SalesRecord) => sale.amount)),
      Stream.run(salesStream, AnalyticsSinks.timeWindow(
        (sale: SalesRecord) => sale.timestamp,
        3600000, // 1 hour windows
        Sink.sum.pipe(Sink.mapInput((sale: SalesRecord) => sale.amount))
      ))
    ], { concurrency: 4 })

    return {
      amountStatistics: amountStats,
      regionDistribution: regionFreq,
      amountPercentiles: percentiles,
      hourlySales: hourlyTrends
    }
  })
```

### Pattern 2: Error Recovery and Fallback Strategies

Handle different types of errors gracefully with fallback processing strategies.

```typescript
import { Sink, Stream, Effect, Either, Option } from "effect"

interface ProcessingItem {
  id: string
  data: string
  retryCount?: number
}

// Error-resilient sink with multiple fallback strategies
const resilientProcessingSink = <T>(
  primaryProcessor: (item: T) => Effect.Effect<string, Error>,
  fallbackProcessor: (item: T) => Effect.Effect<string, never>
) =>
  Effect.gen(function* () {
    const successCount = yield* Ref.make(0)
    const failureCount = yield* Ref.make(0)
    const retryQueue = yield* Ref.make<T[]>([])

    return Sink.forEach((item: T) =>
      primaryProcessor(item).pipe(
        Effect.tapBoth({
          onFailure: () => Effect.flatMap(
            Ref.update(failureCount, n => n + 1),
            () => Ref.update(retryQueue, queue => [...queue, item])
          ),
          onSuccess: () => Ref.update(successCount, n => n + 1)
        }),
        Effect.orElse(() => fallbackProcessor(item))
      )
    ).pipe(
      Sink.ensuring(
        Effect.gen(function* () {
          const successes = yield* Ref.get(successCount)
          const failures = yield* Ref.get(failureCount)
          const retries = yield* Ref.get(retryQueue)
          
          console.log(`Processing complete: ${successes} successes, ${failures} failures, ${retries.length} retries needed`)
        })
      )
    )
  })

// Graduated fallback strategy
const graduatedFallbackSink = <T>(
  item: T,
  strategies: Array<(item: T) => Effect.Effect<string, Error>>
) =>
  Effect.gen(function* () {
    for (let i = 0; i < strategies.length; i++) {
      const result = yield* strategies[i](item).pipe(Effect.either)
      
      if (Either.isRight(result)) {
        return result.right
      }
      
      if (i === strategies.length - 1) {
        // All strategies failed
        yield* Effect.logWarning(`All processing strategies failed for item: ${JSON.stringify(item)}`)
        return `FAILED_PROCESSING_${JSON.stringify(item)}`
      }
    }
    
    return "UNEXPECTED_STATE"
  })

// Circuit breaker pattern for sink processing
const circuitBreakerSink = <T>(
  processor: (item: T) => Effect.Effect<string, Error>,
  failureThreshold: number = 5,
  timeoutMs: number = 60000
) =>
  Effect.gen(function* () {
    const state = yield* Ref.make({
      failures: 0,
      lastFailureTime: 0,
      isOpen: false
    })

    return Sink.forEach((item: T) =>
      Effect.gen(function* () {
        const currentState = yield* Ref.get(state)
        
        // Check if circuit breaker should close
        if (currentState.isOpen && 
            Date.now() - currentState.lastFailureTime > timeoutMs) {
          yield* Ref.update(state, s => ({ ...s, isOpen: false, failures: 0 }))
        }
        
        // If circuit is open, use fallback
        if (currentState.isOpen) {
          return `CIRCUIT_OPEN_FALLBACK_${item}`
        }
        
        // Try main processor
        return yield* processor(item).pipe(
          Effect.tapError(() =>
            Ref.update(state, s => {
              const newFailures = s.failures + 1
              return {
                failures: newFailures,
                lastFailureTime: Date.now(),
                isOpen: newFailures >= failureThreshold
              }
            })
          ),
          Effect.tap(() => 
            Ref.update(state, s => ({ ...s, failures: 0 }))
          ),
          Effect.orElse(() => Effect.succeed(`FALLBACK_${item}`))
        )
      })
    )
  })
```

### Pattern 3: Performance Optimization and Batching

Optimize sink performance through intelligent batching and resource management.

```typescript
import { Sink, Stream, Effect, Chunk, Queue, Duration } from "effect"

// Adaptive batching sink that adjusts batch size based on throughput
const adaptiveBatchingSink = <T>(
  processor: (batch: Chunk.Chunk<T>) => Effect.Effect<void, Error>,
  initialBatchSize: number = 100,
  maxBatchSize: number = 1000
) =>
  Effect.gen(function* () {
    const metrics = yield* Ref.make({
      batchSize: initialBatchSize,
      lastProcessingTime: 0,
      totalProcessed: 0,
      avgProcessingTime: 0
    })

    return Sink.foldLeftChunks(
      [] as T[],
      (acc, chunk) => [...acc, ...Chunk.toReadonlyArray(chunk)]
    ).pipe(
      Sink.flatMap(allItems =>
        Effect.gen(function* () {
          const currentMetrics = yield* Ref.get(metrics)
          const chunks = []
          
          // Create batches of current optimal size
          for (let i = 0; i < allItems.length; i += currentMetrics.batchSize) {
            chunks.push(Chunk.fromIterable(allItems.slice(i, i + currentMetrics.batchSize)))
          }
          
          // Process batches and measure performance
          for (const chunk of chunks) {
            const startTime = Date.now()
            yield* processor(chunk)
            const processingTime = Date.now() - startTime
            
            yield* Ref.update(metrics, m => {
              const newAvg = (m.avgProcessingTime * m.totalProcessed + processingTime) / (m.totalProcessed + 1)
              let newBatchSize = m.batchSize
              
              // Adjust batch size based on processing time
              if (processingTime < 100 && m.batchSize < maxBatchSize) {
                newBatchSize = Math.min(maxBatchSize, Math.floor(m.batchSize * 1.2))
              } else if (processingTime > 500 && m.batchSize > 10) {
                newBatchSize = Math.max(10, Math.floor(m.batchSize * 0.8))
              }
              
              return {
                batchSize: newBatchSize,
                lastProcessingTime: processingTime,
                totalProcessed: m.totalProcessed + 1,
                avgProcessingTime: newAvg
              }
            })
          }
          
          return allItems.length
        })
      )
    )
  })

// Memory-efficient streaming sink for large datasets
const memoryEfficientSink = <T, R>(
  processor: (item: T) => Effect.Effect<R, Error>,
  bufferSize: number = 1000,
  flushInterval: Duration.Duration = Duration.seconds(5)
) =>
  Effect.gen(function* () {
    const buffer = yield* Queue.bounded<T>(bufferSize)
    const results = yield* Ref.make<R[]>([])
    
    // Background flush process
    const flushProcess = Effect.gen(function* () {
      while (true) {
        yield* Effect.sleep(flushInterval)
        const items = yield* Queue.takeAll(buffer)
        
        if (Chunk.size(items) > 0) {
          const processed = yield* Effect.all(
            Chunk.map(items, processor),
            { concurrency: 10 }
          )
          yield* Ref.update(results, current => [...current, ...processed])
        }
      }
    }).pipe(
      Effect.forkDaemon
    )
    
    yield* flushProcess
    
    return Sink.forEach((item: T) =>
      Queue.offer(buffer, item).pipe(
        Effect.timeout(Duration.seconds(1)),
        Effect.orElse(() => 
          Effect.logWarning("Buffer full, dropping item").pipe(
            Effect.as(undefined)
          )
        )
      )
    ).pipe(
      Sink.ensuring(
        Effect.gen(function* () {
          // Final flush
          const remaining = yield* Queue.takeAll(buffer)
          if (Chunk.size(remaining) > 0) {
            const processed = yield* Effect.all(
              Chunk.map(remaining, processor),
              { concurrency: 10 }
            )
            yield* Ref.update(results, current => [...current, ...processed])
          }
        })
      )
    )
  })

// Parallel processing sink with work stealing
const parallelProcessingSink = <T, R>(
  processor: (item: T) => Effect.Effect<R, Error>,
  concurrency: number = 4
) =>
  Sink.collectAll<T>().pipe(
    Sink.flatMap(items =>
      Effect.gen(function* () {
        const chunks = Chunk.chunksOf(items, Math.ceil(Chunk.size(items) / concurrency))
        
        const results = yield* Effect.all(
          Chunk.map(chunks, chunk =>
            Effect.all(
              Chunk.map(chunk, processor),
              { concurrency: 'unbounded' }
            )
          ),
          { concurrency }
        )
        
        return Chunk.flatten(results)
      })
    )
  )
```

## Integration Examples

### Integration with Effect Platform HTTP

Processing HTTP request/response streams with Sink for server applications.

```typescript
import { Sink, Stream, Effect, Chunk } from "effect"
import { HttpServer, HttpServerRequest } from "@effect/platform/HttpServer"

interface RequestMetrics {
  path: string
  method: string
  responseTime: number
  statusCode: number
  userAgent?: string
}

// HTTP request metrics collection
const httpMetricsSink = Sink.collectAllToMap(
  (metrics: RequestMetrics) => `${metrics.method}:${metrics.path}`,
  (a: RequestMetrics, b: RequestMetrics) => ({
    ...b,
    responseTime: (a.responseTime + b.responseTime) / 2 // Average response time
  })
).pipe(
  Sink.map(metricsMap => ({
    endpointCount: metricsMap.size,
    metrics: Array.from(metricsMap.values())
  }))
)

// Request rate limiting sink
const rateLimitingSink = (requestsPerSecond: number) =>
  Sink.foldWeighted({
    initial: { count: 0, windowStart: Date.now() },
    maxCost: requestsPerSecond,
    cost: () => 1,
    body: (state, _: RequestMetrics) => {
      const now = Date.now()
      if (now - state.windowStart >= 1000) {
        return { count: 1, windowStart: now }
      }
      return { count: state.count + 1, windowStart: state.windowStart }
    }
  })

// HTTP middleware using Sink for request processing
const createRequestProcessor = (metricsStream: Stream.Stream<RequestMetrics, Error>) =>
  Effect.gen(function* () {
    // Process metrics in background
    const metricsProcessor = Stream.run(metricsStream, httpMetricsSink).pipe(
      Effect.tap(metrics => 
        Effect.sync(() => console.log('Processed metrics:', metrics))
      ),
      Effect.forkDaemon
    )
    
    yield* metricsProcessor
    
    return (request: HttpServerRequest.HttpServerRequest) =>
      Effect.gen(function* () {
        const startTime = Date.now()
        
        // Process request
        const response = yield* Effect.succeed({
          status: 200,
          body: JSON.stringify({ message: 'Hello World' })
        })
        
        // Emit metrics
        const metrics: RequestMetrics = {
          path: request.url,
          method: request.method,
          responseTime: Date.now() - startTime,
          statusCode: response.status,
          userAgent: request.headers['user-agent']
        }
        
        // Would emit to metrics stream in real implementation
        yield* Effect.sync(() => console.log('Request processed:', metrics))
        
        return response
      })
  })
```

### Integration with Database Streams

Using Sink for efficient database operations and result processing.

```typescript
import { Sink, Stream, Effect, Chunk } from "effect"

interface DatabaseRecord {
  id: string
  data: Record<string, any>
  timestamp: number
}

interface BatchInsertResult {
  insertedCount: number
  errors: Error[]
  lastInsertedId?: string
}

// Database batch insert sink
const batchInsertSink = (
  tableName: string,
  batchSize: number = 1000
) =>
  Sink.foldWeighted({
    initial: [] as DatabaseRecord[],
    maxCost: batchSize,
    cost: () => 1,
    body: (batch, record: DatabaseRecord) => [...batch, record]
  }).pipe(
    Sink.flatMap(batch =>
      Effect.gen(function* () {
        if (batch.length === 0) return { insertedCount: 0, errors: [] }
        
        // Simulate database batch insert
        const insertResult = yield* Effect.try(() => ({
          insertedCount: batch.length,
          errors: [],
          lastInsertedId: batch[batch.length - 1]?.id
        })).pipe(
          Effect.tapError(error =>
            Effect.sync(() => console.error(`Batch insert failed:`, error))
          ),
          Effect.orElse(() => Effect.succeed({
            insertedCount: 0,
            errors: [new Error('Batch insert failed')],
            lastInsertedId: undefined
          }))
        )
        
        return insertResult
      })
    )
  )

// Database query result streaming
const queryResultSink = <T>(
  query: string,
  transformer: (row: any) => T
) =>
  Effect.gen(function* () {
    return Sink.forEach((row: any) =>
      Effect.gen(function* () {
        const transformed = transformer(row)
        yield* Effect.sync(() => console.log('Processed row:', transformed))
        return transformed
      })
    ).pipe(
      Sink.ensuring(
        Effect.sync(() => console.log(`Query completed: ${query}`))
      )
    )
  })

// Database migration sink with progress tracking
const migrationSink = (
  migrationName: string,
  totalRecords: number
) =>
  Effect.gen(function* () {
    const processedCount = yield* Ref.make(0)
    
    return Sink.forEach((record: DatabaseRecord) =>
      Effect.gen(function* () {
        // Simulate record migration
        yield* Effect.sleep(Duration.millis(1))
        
        const current = yield* Ref.updateAndGet(processedCount, n => n + 1)
        
        if (current % 1000 === 0) {
          const progress = (current / totalRecords) * 100
          yield* Effect.sync(() => 
            console.log(`Migration ${migrationName}: ${progress.toFixed(1)}% complete`)
          )
        }
        
        return record
      })
    ).pipe(
      Sink.ensuring(
        Effect.gen(function* () {
          const final = yield* Ref.get(processedCount)
          yield* Effect.sync(() => 
            console.log(`Migration ${migrationName} completed: ${final} records processed`)
          )
        })
      )
    )
  })

// Usage with database streams
const processDatabaseData = (
  recordStream: Stream.Stream<DatabaseRecord, Error>
) =>
  Effect.gen(function* () {
    // Concurrent processing for different operations
    const [insertResults, migrationResults] = yield* Effect.all([
      Stream.run(recordStream, batchInsertSink('processed_records', 500)),
      Stream.run(recordStream, yield* migrationSink('data_cleanup', 10000))
    ], { concurrency: 2 })
    
    return { insertResults, migrationResults }
  })
```

### Integration with Message Queues

Processing message queue streams with Sink for reliable message handling.

```typescript
import { Sink, Stream, Effect, Queue, Duration } from "effect"

interface Message {
  id: string
  topic: string
  payload: any
  timestamp: number
  retryCount: number
}

interface MessageProcessingResult {
  processed: number
  failed: number
  requeued: number
}

// Dead letter queue sink for failed messages
const deadLetterSink = (maxRetries: number = 3) =>
  Effect.gen(function* () {
    const dlq = yield* Queue.unbounded<Message>()
    const stats = yield* Ref.make({ processed: 0, failed: 0, requeued: 0 })
    
    return Sink.forEach((message: Message) =>
      Effect.gen(function* () {
        if (message.retryCount >= maxRetries) {
          yield* Queue.offer(dlq, message)
          yield* Ref.update(stats, s => ({ ...s, failed: s.failed + 1 }))
          yield* Effect.logError(`Message ${message.id} sent to DLQ after ${message.retryCount} retries`)
        } else {
          // Requeue for retry
          const retryMessage = { ...message, retryCount: message.retryCount + 1 }
          yield* Effect.sync(() => console.log(`Requeuing message ${message.id} (attempt ${retryMessage.retryCount})`))
          yield* Ref.update(stats, s => ({ ...s, requeued: s.requeued + 1 }))
        }
      })
    ).pipe(
      Sink.ensuring(
        Effect.gen(function* () {
          const finalStats = yield* Ref.get(stats)
          const dlqSize = yield* Queue.size(dlq)
          yield* Effect.sync(() => 
            console.log(`DLQ processing complete:`, { ...finalStats, dlqSize })
          )
        })
      )
    )
  })

// Message acknowledgment sink
const ackSink = (ackTimeout: Duration.Duration = Duration.seconds(30)) =>
  Sink.forEach((message: Message) =>
    Effect.gen(function* () {
      // Simulate message processing
      yield* Effect.sleep(Duration.millis(Math.random() * 100))
      
      // Acknowledge message with timeout
      yield* Effect.succeed(undefined).pipe(
        Effect.timeout(ackTimeout),
        Effect.tapError(() => 
          Effect.logWarning(`Ack timeout for message ${message.id}`)
        ),
        Effect.orElse(() => Effect.succeed(undefined))
      )
      
      yield* Effect.sync(() => console.log(`Acknowledged message ${message.id}`))
    })
  )

// Message routing sink based on topic
const routingSink = (
  routes: Record<string, Sink.Sink<any, Message>>
) =>
  Sink.forEach((message: Message) =>
    Effect.gen(function* () {
      const routeSink = routes[message.topic]
      
      if (routeSink) {
        yield* Stream.run(Stream.make(message), routeSink)
      } else {
        yield* Effect.logWarning(`No route found for topic: ${message.topic}`)
      }
    })
  )

// Message queue processing pipeline
const processMessageQueue = (
  messageStream: Stream.Stream<Message, Error>
) =>
  Effect.gen(function* () {
    // Create topic-specific sinks
    const orderSink = Sink.forEach((msg: Message) => 
      Effect.sync(() => console.log('Processing order:', msg.payload))
    )
    
    const userSink = Sink.forEach((msg: Message) => 
      Effect.sync(() => console.log('Processing user event:', msg.payload))
    )
    
    const routes = {
      'orders': orderSink,
      'users': userSink
    }
    
    // Process with routing and error handling
    const result = yield* Stream.run(
      messageStream,
      routingSink(routes).pipe(
        Sink.orElse(() => yield* deadLetterSink(3))
      )
    )
    
    return result
  })
```

## Conclusion

Sink provides **type-safe stream consumption**, **automatic backpressure handling**, and **composable aggregation patterns** for **efficient data processing**.

Key benefits:
- **Memory Efficiency**: Built-in backpressure prevents memory overflow in stream processing
- **Composability**: Sinks can be combined, transformed, and reused across different use cases
- **Resource Safety**: Automatic resource management ensures proper cleanup even during failures
- **Type Safety**: Full TypeScript integration with compile-time error detection
- **Performance**: Optimized for high-throughput scenarios with configurable batching and concurrency

Use Sink when you need reliable, memory-efficient stream consumption with complex aggregation logic, error recovery, and resource management requirements.