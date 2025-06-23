# Subscribable: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem Subscribable Solves

Modern applications need reactive programming patterns that can handle both current state and ongoing changes. Traditional approaches using observables, event emitters, or manual subscription management create complex, error-prone systems with resource leaks and inconsistent behavior:

```typescript
// Traditional observable approach - complex subscription management
import { BehaviorSubject, Observable, Subscription } from 'rxjs'

class TemperatureSensor {
  private temperatureSubject = new BehaviorSubject<number>(20)
  private subscriptions: Subscription[] = []
  
  // Manual subscription lifecycle management
  subscribe(callback: (temp: number) => void): () => void {
    const subscription = this.temperatureSubject.subscribe({
      next: callback,
      error: (error) => console.error('Temperature error:', error)
    })
    
    this.subscriptions.push(subscription)
    
    // Manual cleanup - easy to forget
    return () => {
      const index = this.subscriptions.indexOf(subscription)
      if (index >= 0) {
        this.subscriptions.splice(index, 1)
        subscription.unsubscribe()
      }
    }
  }
  
  // Scattered error handling and state management
  updateTemperature(temp: number): void {
    if (temp < -50 || temp > 60) {
      this.temperatureSubject.error(new Error('Temperature out of range'))
      return
    }
    this.temperatureSubject.next(temp)
  }
  
  getCurrentTemperature(): number {
    return this.temperatureSubject.value
  }
  
  // Manual cleanup - often forgotten
  dispose(): void {
    this.subscriptions.forEach(sub => sub.unsubscribe())
    this.subscriptions = []
    this.temperatureSubject.complete()
  }
}

// Usage leads to memory leaks and complex lifecycle management
const sensor = new TemperatureSensor()
const unsubscribe1 = sensor.subscribe(temp => console.log(`Temp: ${temp}°C`))
const unsubscribe2 = sensor.subscribe(temp => updateUI(temp))

// Forgot to call unsubscribe1() - memory leak!
// Forgot to call sensor.dispose() - more leaks!
```

This approach leads to:
- **Memory Leaks** - Forgotten unsubscribe calls leave dangling references
- **Complex Lifecycle** - Manual subscription and cleanup management is error-prone
- **Scattered Error Handling** - Error handling mixed with business logic
- **Resource Management** - No automatic cleanup of resources and subscriptions

### The Subscribable Solution

Subscribable provides type-safe reactive containers that combine current state access (via Readable) with change streams, enabling clean reactive programming with automatic resource management:

```typescript
import { Effect, Subscribable, Ref, Stream, Schedule, Duration } from "effect"

// Type-safe reactive container with automatic resource management
const makeTemperatureSensor = Effect.gen(function* () {
  const tempRef = yield* Ref.make(20)
  
  // Create temperature change stream from sensor readings
  const temperatureChanges = Stream.repeatEffect(
    Effect.gen(function* () {
      // Simulate sensor reading
      const reading = yield* Effect.sync(() => 
        Math.random() * 40 + 10 // 10-50°C range
      )
      yield* Ref.set(tempRef, reading)
      return reading
    })
  ).pipe(
    Stream.schedule(Schedule.fixed(Duration.seconds(1)))
  )
  
  // Subscribable combines current state access + change stream
  const temperatureSensor: Subscribable<number> = Subscribable.make({
    get: tempRef.get,
    changes: temperatureChanges
  })
  
  return temperatureSensor
})

// Clean, composable reactive patterns with automatic cleanup
const program = Effect.gen(function* () {
  const sensor = yield* makeTemperatureSensor
  
  // Get current temperature - no subscription needed
  const currentTemp = yield* sensor.get
  console.log(`Current temperature: ${currentTemp}°C`)
  
  // Transform the stream for different use cases
  const fahrenheitSensor = sensor.pipe(
    Subscribable.map(celsius => (celsius * 9/5) + 32)
  )
  
  const currentTempF = yield* fahrenheitSensor.get
  console.log(`Current temperature: ${currentTempF}°F`)
  
  // Process change stream with built-in resource management
  yield* sensor.changes.pipe(
    Stream.take(5), // Only take 5 readings
    Stream.tap(temp => Effect.sync(() => 
      console.log(`Temperature update: ${temp}°C`)
    )),
    Stream.runDrain
  )
})
```

### Key Concepts

**Reactive Container**: Subscribable extends Readable to provide both current state access and a stream of changes, enabling complete reactive programming patterns.

**Change Stream**: The `changes` property provides a Stream of all value updates, enabling reactive programming with automatic resource management.

**Composable Transformations**: Transform both current state and change streams using familiar operators while maintaining type safety and resource management.

## Basic Usage Patterns

### Creating Subscribable Containers

```typescript
import { Effect, Subscribable, Ref, Stream, Schedule, Duration } from "effect"

// From Ref and Stream - most common pattern
const makeCounterSubscribable = Effect.gen(function* () {
  const countRef = yield* Ref.make(0)
  
  // Stream of increments every second
  const incrementStream = Stream.repeatEffect(
    Ref.updateAndGet(countRef, n => n + 1)
  ).pipe(
    Stream.schedule(Schedule.fixed(Duration.seconds(1)))
  )
  
  return Subscribable.make({
    get: countRef.get,
    changes: incrementStream
  })
})

// Direct construction with Effects and Streams
const makeDataSubscribable = <T>(
  initialValue: T,
  updateEffect: Effect.Effect<T>
) => Effect.gen(function* () {
  const dataRef = yield* Ref.make(initialValue)
  
  const updateStream = Stream.repeatEffect(
    Effect.gen(function* () {
      const newData = yield* updateEffect
      yield* Ref.set(dataRef, newData)
      return newData
    })
  ).pipe(
    Stream.schedule(Schedule.fixed(Duration.seconds(2)))
  )
  
  return Subscribable.make({
    get: dataRef.get,
    changes: updateStream
  })
})

// Usage example
const program = Effect.gen(function* () {
  // Counter that increments automatically
  const counter = yield* makeCounterSubscribable
  
  // Get current value
  const currentCount = yield* counter.get
  console.log('Current count:', currentCount)
  
  // Watch changes for 5 seconds
  yield* counter.changes.pipe(
    Stream.take(5),
    Stream.tap(count => Effect.sync(() => 
      console.log(`Count updated: ${count}`)
    )),
    Stream.runDrain
  )
})
```

### Transforming Subscribable Values

```typescript
import { Effect, Subscribable, Ref, Stream, Schedule, Duration } from "effect"

const makeWeatherStation = Effect.gen(function* () {
  const weatherRef = yield* Ref.make({
    temperature: 22,
    humidity: 45,
    pressure: 1013.25
  })
  
  // Simulate weather updates
  const weatherUpdates = Stream.repeatEffect(
    Effect.gen(function* () {
      const newWeather = {
        temperature: Math.random() * 40 + 10, // 10-50°C
        humidity: Math.random() * 60 + 20,    // 20-80%
        pressure: Math.random() * 50 + 990   // 990-1040 hPa
      }
      yield* Ref.set(weatherRef, newWeather)
      return newWeather
    })
  ).pipe(
    Stream.schedule(Schedule.fixed(Duration.seconds(3)))
  )
  
  const weatherSubscribable = Subscribable.make({
    get: weatherRef.get,
    changes: weatherUpdates
  })
  
  // Transform with pure functions
  const temperatureOnly = weatherSubscribable.pipe(
    Subscribable.map(weather => weather.temperature)
  )
  
  const temperatureInFahrenheit = temperatureOnly.pipe(
    Subscribable.map(celsius => (celsius * 9/5) + 32)
  )
  
  // Transform with Effects
  const weatherAlert = weatherSubscribable.pipe(
    Subscribable.mapEffect(weather => {
      if (weather.temperature > 35) {
        return Effect.succeed(`⚠️ HIGH TEMPERATURE: ${weather.temperature.toFixed(1)}°C`)
      }
      if (weather.humidity > 70) {
        return Effect.succeed(`💧 HIGH HUMIDITY: ${weather.humidity.toFixed(1)}%`)
      }
      if (weather.pressure < 1000) {
        return Effect.succeed(`⛈️ LOW PRESSURE: ${weather.pressure.toFixed(1)} hPa`)
      }
      return Effect.succeed(`✅ Normal conditions`)
    })
  )
  
  return {
    weather: weatherSubscribable,
    temperature: temperatureOnly,
    temperatureF: temperatureInFahrenheit,
    alerts: weatherAlert
  }
})

// Usage with transformations
const weatherExample = Effect.gen(function* () {
  const station = yield* makeWeatherStation
  
  // Get current values
  const currentWeather = yield* station.weather.get
  const currentTempF = yield* station.temperatureF.get
  const currentAlert = yield* station.alerts.get
  
  console.log('Current weather:', currentWeather)
  console.log(`Temperature: ${currentTempF.toFixed(1)}°F`)
  console.log('Alert:', currentAlert)
  
  // Watch temperature changes in Fahrenheit
  yield* station.temperatureF.changes.pipe(
    Stream.take(3),
    Stream.tap(tempF => Effect.sync(() => 
      console.log(`Temperature update: ${tempF.toFixed(1)}°F`)
    )),
    Stream.runDrain
  )
})
```

### Subscribable with Error Handling

```typescript
import { Effect, Subscribable, Ref, Stream, Schedule, Duration, Data } from "effect"

class SensorError extends Data.TaggedError("SensorError")<{
  sensor: string
  reason: string
}> {}

const makeReliableSensor = (sensorId: string) => Effect.gen(function* () {
  const valueRef = yield* Ref.make(0)
  
  // Sensor reading with potential failures
  const sensorReading = Effect.gen(function* () {
    // Simulate sensor failure (10% chance)
    if (Math.random() < 0.1) {
      return yield* Effect.fail(new SensorError({
        sensor: sensorId,
        reason: 'Communication timeout'
      }))
    }
    
    const value = Math.random() * 100
    yield* Ref.set(valueRef, value)
    return value
  })
  
  // Resilient sensor stream with retry logic
  const sensorStream = Stream.repeatEffect(sensorReading).pipe(
    Stream.retry(Schedule.exponential(Duration.millis(100))),
    Stream.schedule(Schedule.fixed(Duration.seconds(2)))
  )
  
  const sensorSubscribable = Subscribable.make({
    get: Effect.gen(function* () {
      const value = yield* valueRef.get
      // Could add validation here
      if (value < 0 || value > 100) {
        return yield* Effect.fail(new SensorError({
          sensor: sensorId,
          reason: 'Value out of range'
        }))
      }
      return value
    }),
    changes: sensorStream
  })
  
  // Validated readings with error recovery
  const validatedSensor = sensorSubscribable.pipe(
    Subscribable.mapEffect(value => {
      if (value > 90) {
        return Effect.succeed(`🔴 CRITICAL: ${value.toFixed(1)}`)
      }
      if (value > 70) {
        return Effect.succeed(`🟡 WARNING: ${value.toFixed(1)}`)
      }
      return Effect.succeed(`🟢 NORMAL: ${value.toFixed(1)}`)
    })
  )
  
  return {
    sensor: sensorSubscribable,
    validated: validatedSensor
  }
})

// Usage with error handling
const sensorExample = Effect.gen(function* () {
  const { sensor, validated } = yield* makeReliableSensor('TEMP-001')
  
  // Handle errors when getting current value
  const currentValue = yield* Effect.either(sensor.get)
  if (currentValue._tag === 'Left') {
    console.log('Sensor error:', currentValue.left.reason)
  } else {
    console.log('Current reading:', currentValue.right)
  }
  
  // Process change stream with error handling
  yield* sensor.changes.pipe(
    Stream.take(5),
    Stream.tap(value => Effect.sync(() => 
      console.log(`Sensor update: ${value.toFixed(1)}`)
    )),
    Stream.catchAll(error => 
      Stream.make(`Error: ${error.reason}`)
    ),
    Stream.runDrain
  )
})
```

## Real-World Examples

### Example 1: Real-Time Stock Price Monitor

Financial applications need reactive stock price monitoring with current prices, change tracking, and alert systems.

```typescript
import { Effect, Subscribable, Ref, Stream, Schedule, Duration, Data, Array as Arr } from "effect"

interface StockPrice {
  readonly symbol: string
  readonly price: number
  readonly change: number
  readonly changePercent: number
  readonly volume: number
  readonly timestamp: Date
}

interface Portfolio {
  readonly stocks: Map<string, number> // symbol -> shares
  readonly totalValue: number
  readonly dayChange: number
  readonly changePercent: number
}

class MarketError extends Data.TaggedError("MarketError")<{
  symbol: string
  reason: string
}> {}

const makeStockPriceSubscribable = (symbol: string, initialPrice = 100) =>
  Effect.gen(function* () {
    const priceRef = yield* Ref.make<StockPrice>({
      symbol,
      price: initialPrice,
      change: 0,
      changePercent: 0,
      volume: 0,
      timestamp: new Date()
    })
    
    // Simulate real-time price updates
    const priceUpdates = Stream.repeatEffect(
      Effect.gen(function* () {
        const currentPrice = yield* priceRef.get
        
        // Simulate price movement (-2% to +2%)
        const changePercent = (Math.random() - 0.5) * 4
        const newPrice = currentPrice.price * (1 + changePercent / 100)
        const change = newPrice - initialPrice
        const volume = Math.floor(Math.random() * 10000) + 1000
        
        const updatedPrice: StockPrice = {
          symbol,
          price: Math.max(0.01, newPrice), // Prevent negative prices
          change,
          changePercent: (change / initialPrice) * 100,
          volume,
          timestamp: new Date()
        }
        
        yield* Ref.set(priceRef, updatedPrice)
        return updatedPrice
      })
    ).pipe(
      Stream.schedule(Schedule.fixed(Duration.millis(500)))
    )
    
    return Subscribable.make({
      get: priceRef.get,
      changes: priceUpdates
    })
  })

const makePortfolioTracker = (stocks: Map<string, number>) =>
  Effect.gen(function* () {
    // Create subscribables for each stock
    const stockSubscribables = yield* Effect.all(
      Array.from(stocks.keys()).map(symbol =>
        makeStockPriceSubscribable(symbol).pipe(
          Effect.map(subscribable => [symbol, subscribable] as const)
        )
      )
    )
    
    const stockMap = new Map(stockSubscribables)
    
    // Portfolio value calculation
    const portfolioRef = yield* Ref.make<Portfolio>({
      stocks,
      totalValue: 0,
      dayChange: 0,
      changePercent: 0
    })
    
    // Combined portfolio updates
    const portfolioUpdates = Stream.mergeAll(
      Array.from(stockMap.values()).map(stock => stock.changes),
      { concurrency: 'unbounded' }
    ).pipe(
      Stream.tap(_ =>
        Effect.gen(function* () {
          // Recalculate portfolio value when any stock changes
          let totalValue = 0
          let totalDayChange = 0
          
          for (const [symbol, shares] of stocks) {
            const stockSub = stockMap.get(symbol)
            if (stockSub) {
              const stockPrice = yield* stockSub.get
              totalValue += stockPrice.price * shares
              totalDayChange += stockPrice.change * shares
            }
          }
          
          const changePercent = totalValue > 0 
            ? (totalDayChange / (totalValue - totalDayChange)) * 100 
            : 0
          
          yield* Ref.set(portfolioRef, {
            stocks,
            totalValue,
            dayChange: totalDayChange,
            changePercent
          })
        })
      ),
      Stream.map(_ => portfolioRef.get),
      Stream.flatMap(Stream.fromEffect)
    )
    
    const portfolioSubscribable = Subscribable.make({
      get: portfolioRef.get,
      changes: portfolioUpdates
    })
    
    // Alert system for significant changes
    const portfolioAlerts = portfolioSubscribable.pipe(
      Subscribable.mapEffect(portfolio => {
        if (Math.abs(portfolio.changePercent) > 5) {
          const direction = portfolio.changePercent > 0 ? '📈' : '📉'
          return Effect.succeed(
            `${direction} Portfolio Alert: ${portfolio.changePercent.toFixed(2)}% ` +
            `($${portfolio.dayChange.toFixed(2)})`
          )
        }
        return Effect.succeed(
          `📊 Portfolio: $${portfolio.totalValue.toFixed(2)} ` +
          `(${portfolio.changePercent >= 0 ? '+' : ''}${portfolio.changePercent.toFixed(2)}%)`
        )
      })
    )
    
    return {
      portfolio: portfolioSubscribable,
      alerts: portfolioAlerts,
      stocks: stockMap
    }
  })

// Usage in trading application
const tradingApp = Effect.gen(function* () {
  // Portfolio with FAANG stocks
  const portfolio = new Map([
    ['AAPL', 10],   // 10 shares of Apple
    ['GOOGL', 5],   // 5 shares of Google
    ['AMZN', 3],    // 3 shares of Amazon
    ['MSFT', 8],    // 8 shares of Microsoft
    ['META', 12]    // 12 shares of Meta
  ])
  
  const tracker = yield* makePortfolioTracker(portfolio)
  
  // Display initial portfolio state
  const initialPortfolio = yield* tracker.portfolio.get
  const initialAlert = yield* tracker.alerts.get
  
  console.log('=== Initial Portfolio ===')
  console.log(`Total Value: $${initialPortfolio.totalValue.toFixed(2)}`)
  console.log(`Day Change: $${initialPortfolio.dayChange.toFixed(2)}`)
  console.log(`Change %: ${initialPortfolio.changePercent.toFixed(2)}%`)
  console.log('Alert:', initialAlert)
  
  // Monitor portfolio changes for 10 seconds
  console.log('\n=== Live Portfolio Updates ===')
  yield* tracker.portfolio.changes.pipe(
    Stream.take(20), // Take 20 updates
    Stream.throttle(Duration.seconds(1)), // Limit to 1 update per second
    Stream.tap(portfolio => 
      Effect.gen(function* () {
        const alert = yield* tracker.alerts.get
        console.log(
          `Value: $${portfolio.totalValue.toFixed(2)} | ` +
          `Change: ${portfolio.changePercent >= 0 ? '+' : ''}${portfolio.changePercent.toFixed(2)}% | ` +
          `Alert: ${alert}`
        )
      })
    ),
    Stream.runDrain
  )
  
  // Show individual stock performance
  console.log('\n=== Individual Stock Performance ===')
  for (const [symbol, stockSub] of tracker.stocks) {
    const stock = yield* stockSub.get
    const shares = portfolio.get(symbol) || 0
    const position = stock.price * shares
    
    console.log(
      `${symbol}: $${stock.price.toFixed(2)} ` +
      `(${stock.changePercent >= 0 ? '+' : ''}${stock.changePercent.toFixed(2)}%) ` +
      `Position: $${position.toFixed(2)}`
    )
  }
})
```

### Example 2: IoT Sensor Network with Data Aggregation

IoT systems need to collect data from multiple sensors, aggregate values, and provide real-time monitoring with fault tolerance.

```typescript
import { Effect, Subscribable, Ref, Stream, Schedule, Duration, Data, Array as Arr } from "effect"

interface SensorReading {
  readonly sensorId: string
  readonly type: 'temperature' | 'humidity' | 'pressure' | 'light'
  readonly value: number
  readonly unit: string
  readonly timestamp: Date
  readonly quality: 'good' | 'fair' | 'poor'
}

interface SensorNetwork {
  readonly sensors: Map<string, SensorReading>
  readonly averages: {
    readonly temperature: number
    readonly humidity: number
    readonly pressure: number
    readonly light: number
  }
  readonly activeCount: number
  readonly lastUpdate: Date
}

class SensorFailure extends Data.TaggedError("SensorFailure")<{
  sensorId: string
  reason: string
}> {}

const makeSensorSubscribable = (
  sensorId: string, 
  type: SensorReading['type'],
  baseValue: number,
  unit: string
) => Effect.gen(function* () {
  const readingRef = yield* Ref.make<SensorReading>({
    sensorId,
    type,
    value: baseValue,
    unit,
    timestamp: new Date(),
    quality: 'good'
  })
  
  // Simulate sensor readings with occasional failures
  const sensorStream = Stream.repeatEffect(
    Effect.gen(function* () {
      // Simulate sensor failure (5% chance)
      if (Math.random() < 0.05) {
        return yield* Effect.fail(new SensorFailure({
          sensorId,
          reason: 'Sensor communication timeout'
        }))
      }
      
      // Generate realistic sensor data
      const previousReading = yield* readingRef.get
      let newValue: number
      let quality: SensorReading['quality'] = 'good'
      
      switch (type) {
        case 'temperature':
          // Temperature fluctuates ±2°C
          newValue = previousReading.value + (Math.random() - 0.5) * 4
          newValue = Math.max(-20, Math.min(50, newValue))
          break
        case 'humidity':
          // Humidity fluctuates ±5%
          newValue = previousReading.value + (Math.random() - 0.5) * 10
          newValue = Math.max(0, Math.min(100, newValue))
          break
        case 'pressure':
          // Pressure fluctuates ±5 hPa
          newValue = previousReading.value + (Math.random() - 0.5) * 10
          newValue = Math.max(980, Math.min(1040, newValue))
          break
        case 'light':
          // Light varies more dramatically
          newValue = previousReading.value * (0.8 + Math.random() * 0.4)
          newValue = Math.max(0, Math.min(1000, newValue))
          break
      }
      
      // Determine reading quality based on variation
      const variation = Math.abs(newValue - previousReading.value)
      if (variation > baseValue * 0.1) quality = 'fair'
      if (variation > baseValue * 0.2) quality = 'poor'
      
      const newReading: SensorReading = {
        sensorId,
        type,
        value: newValue,
        unit,
        timestamp: new Date(),
        quality
      }
      
      yield* Ref.set(readingRef, newReading)
      return newReading
    }).pipe(
      Effect.retry(Schedule.exponential(Duration.millis(100)).pipe(
        Schedule.compose(Schedule.recurs(3))
      ))
    )
  ).pipe(
    Stream.schedule(Schedule.fixed(Duration.seconds(2)))
  )
  
  return Subscribable.make({
    get: readingRef.get,
    changes: sensorStream
  })
})

const makeSensorNetwork = (sensorConfigs: Array<{
  id: string
  type: SensorReading['type']
  baseValue: number
  unit: string
}>) => Effect.gen(function* () {
  // Create all sensor subscribables
  const sensors = yield* Effect.all(
    sensorConfigs.map(config =>
      makeSensorSubscribable(config.id, config.type, config.baseValue, config.unit).pipe(
        Effect.map(subscribable => [config.id, subscribable] as const)
      )
    )
  )
  
  const sensorMap = new Map(sensors)
  const networkRef = yield* Ref.make<SensorNetwork>({
    sensors: new Map(),
    averages: { temperature: 0, humidity: 0, pressure: 0, light: 0 },
    activeCount: 0,
    lastUpdate: new Date()
  })
  
  // Network updates when any sensor changes
  const networkUpdates = Stream.mergeAll(
    Array.from(sensorMap.values()).map(sensor => sensor.changes),
    { concurrency: 'unbounded' }
  ).pipe(
    Stream.tap(_ =>
      Effect.gen(function* () {
        // Collect all current sensor readings
        const sensorReadings = new Map<string, SensorReading>()
        let activeCount = 0
        
        for (const [id, sensor] of sensorMap) {
          try {
            const reading = yield* sensor.get
            sensorReadings.set(id, reading)
            if (reading.quality !== 'poor') activeCount++
          } catch {
            // Sensor is failing, skip it
          }
        }
        
        // Calculate averages by sensor type
        const typeGroups = new Map<SensorReading['type'], number[]>()
        
        for (const reading of sensorReadings.values()) {
          if (reading.quality !== 'poor') {
            const values = typeGroups.get(reading.type) || []
            values.push(reading.value)
            typeGroups.set(reading.type, values)
          }
        }
        
        const averages = {
          temperature: calculateAverage(typeGroups.get('temperature') || []),
          humidity: calculateAverage(typeGroups.get('humidity') || []),
          pressure: calculateAverage(typeGroups.get('pressure') || []),
          light: calculateAverage(typeGroups.get('light') || [])
        }
        
        const networkState: SensorNetwork = {
          sensors: sensorReadings,
          averages,
          activeCount,
          lastUpdate: new Date()
        }
        
        yield* Ref.set(networkRef, networkState)
      })
    ),
    Stream.map(_ => networkRef.get),
    Stream.flatMap(Stream.fromEffect)
  )
  
  const networkSubscribable = Subscribable.make({
    get: networkRef.get,
    changes: networkUpdates
  })
  
  // Network health monitoring
  const networkHealth = networkSubscribable.pipe(
    Subscribable.map(network => {
      const totalSensors = sensorMap.size
      const healthPercent = (network.activeCount / totalSensors) * 100
      
      let status: 'healthy' | 'degraded' | 'critical'
      if (healthPercent >= 90) status = 'healthy'
      else if (healthPercent >= 70) status = 'degraded'
      else status = 'critical'
      
      return {
        status,
        healthPercent,
        activeSensors: network.activeCount,
        totalSensors,
        lastUpdate: network.lastUpdate
      }
    })
  )
  
  // Environmental alerts
  const environmentalAlerts = networkSubscribable.pipe(
    Subscribable.mapEffect(network => {
      const alerts: string[] = []
      
      if (network.averages.temperature > 30) {
        alerts.push(`🌡️ High temperature: ${network.averages.temperature.toFixed(1)}°C`)
      }
      if (network.averages.temperature < 10) {
        alerts.push(`❄️ Low temperature: ${network.averages.temperature.toFixed(1)}°C`)
      }
      if (network.averages.humidity > 80) {
        alerts.push(`💧 High humidity: ${network.averages.humidity.toFixed(1)}%`)
      }
      if (network.averages.pressure < 1000) {
        alerts.push(`⛈️ Low pressure: ${network.averages.pressure.toFixed(1)} hPa`)
      }
      
      return Effect.succeed(
        alerts.length > 0 
          ? alerts.join(' | ')
          : '✅ All environmental conditions normal'
      )
    })
  )
  
  return {
    network: networkSubscribable,
    health: networkHealth,
    alerts: environmentalAlerts,
    sensors: sensorMap
  }
})

// Helper function
const calculateAverage = (values: number[]): number =>
  values.length > 0 ? values.reduce((sum, val) => sum + val, 0) / values.length : 0

// IoT monitoring application
const iotMonitoringApp = Effect.gen(function* () {
  // Configure sensor network
  const sensorConfigs = [
    { id: 'TEMP-01', type: 'temperature' as const, baseValue: 22, unit: '°C' },
    { id: 'TEMP-02', type: 'temperature' as const, baseValue: 24, unit: '°C' },
    { id: 'HUM-01', type: 'humidity' as const, baseValue: 45, unit: '%' },
    { id: 'HUM-02', type: 'humidity' as const, baseValue: 50, unit: '%' },
    { id: 'PRES-01', type: 'pressure' as const, baseValue: 1013, unit: 'hPa' },
    { id: 'LIGHT-01', type: 'light' as const, baseValue: 300, unit: 'lux' }
  ]
  
  const sensorNetwork = yield* makeSensorNetwork(sensorConfigs)
  
  // Display initial network state
  const initialNetwork = yield* sensorNetwork.network.get
  const initialHealth = yield* sensorNetwork.health.get
  const initialAlerts = yield* sensorNetwork.alerts.get
  
  console.log('=== IoT Sensor Network Initialized ===')
  console.log(`Network Health: ${initialHealth.status} (${initialHealth.healthPercent.toFixed(1)}%)`)
  console.log(`Active Sensors: ${initialHealth.activeSensors}/${initialHealth.totalSensors}`)
  console.log('Environmental Alerts:', initialAlerts)
  console.log('\nAverages:')
  console.log(`  Temperature: ${initialNetwork.averages.temperature.toFixed(1)}°C`)
  console.log(`  Humidity: ${initialNetwork.averages.humidity.toFixed(1)}%`)
  console.log(`  Pressure: ${initialNetwork.averages.pressure.toFixed(1)} hPa`)
  console.log(`  Light: ${initialNetwork.averages.light.toFixed(1)} lux`)
  
  // Monitor network changes
  console.log('\n=== Live Network Monitoring ===')
  yield* sensorNetwork.network.changes.pipe(
    Stream.take(15),
    Stream.throttle(Duration.seconds(2)),
    Stream.tap(network =>
      Effect.gen(function* () {
        const health = yield* sensorNetwork.health.get
        const alerts = yield* sensorNetwork.alerts.get
        
        console.log(
          `[${new Date().toLocaleTimeString()}] ` +
          `Health: ${health.status} | ` +
          `Active: ${health.activeSensors}/${health.totalSensors} | ` +
          `Temp: ${network.averages.temperature.toFixed(1)}°C | ` +
          `Alerts: ${alerts}`
        )
      })
    ),
    Stream.runDrain
  )
  
  // Show individual sensor status
  console.log('\n=== Final Sensor Status ===')
  const finalNetwork = yield* sensorNetwork.network.get
  
  for (const [sensorId, reading] of finalNetwork.sensors) {
    console.log(
      `${sensorId}: ${reading.value.toFixed(1)}${reading.unit} ` +
      `(${reading.quality}) - ${reading.timestamp.toLocaleTimeString()}`
    )
  }
})
```

### Example 3: Real-Time Chat System with Presence

Modern chat applications need reactive user presence, message streams, and real-time updates across multiple clients.

```typescript
import { Effect, Subscribable, Ref, Stream, Schedule, Duration, Data, Array as Arr } from "effect"

interface User {
  readonly id: string
  readonly name: string
  readonly avatar: string
  readonly status: 'online' | 'away' | 'offline'
  readonly lastSeen: Date
}

interface Message {
  readonly id: string
  readonly senderId: string
  readonly content: string
  readonly timestamp: Date
  readonly type: 'text' | 'system' | 'typing'
}

interface ChatRoom {
  readonly id: string
  readonly name: string
  readonly users: Map<string, User>
  readonly messages: readonly Message[]
  readonly typingUsers: Set<string>
  readonly lastActivity: Date
}

class ChatError extends Data.TaggedError("ChatError")<{
  room: string
  reason: string
}> {}

const makeChatRoomSubscribable = (roomId: string, roomName: string) =>
  Effect.gen(function* () {
    const roomRef = yield* Ref.make<ChatRoom>({
      id: roomId,
      name: roomName,
      users: new Map(),
      messages: [],
      typingUsers: new Set(),
      lastActivity: new Date()
    })
    
    // Simulated events stream (in real app, this would be WebSocket/SSE)
    const chatEvents = Stream.repeatEffect(
      Effect.gen(function* () {
        const currentRoom = yield* roomRef.get
        const eventType = Math.random()
        
        if (eventType < 0.3) {
          // User joins/leaves
          const userId = `user-${Math.floor(Math.random() * 10)}`
          const isJoining = Math.random() > 0.3
          
          if (isJoining && !currentRoom.users.has(userId)) {
            const user: User = {
              id: userId,
              name: `User ${userId.slice(-1)}`,
              avatar: `https://api.dicebear.com/7.x/avataaars/svg?seed=${userId}`,
              status: 'online',
              lastSeen: new Date()
            }
            
            yield* Ref.update(roomRef, room => ({
              ...room,
              users: new Map(room.users).set(userId, user),
              messages: [...room.messages, {
                id: `msg-${Date.now()}`,
                senderId: 'system',
                content: `${user.name} joined the room`,
                timestamp: new Date(),
                type: 'system'
              }],
              lastActivity: new Date()
            }))
          } else if (!isJoining && currentRoom.users.has(userId)) {
            const user = currentRoom.users.get(userId)!
            const newUsers = new Map(currentRoom.users)
            newUsers.delete(userId)
            
            yield* Ref.update(roomRef, room => ({
              ...room,
              users: newUsers,
              messages: [...room.messages, {
                id: `msg-${Date.now()}`,
                senderId: 'system',
                content: `${user.name} left the room`,
                timestamp: new Date(),
                type: 'system'
              }],
              lastActivity: new Date()
            }))
          }
        } else if (eventType < 0.7) {
          // New message
          const userIds = Array.from(currentRoom.users.keys())
          if (userIds.length > 0) {
            const senderId = userIds[Math.floor(Math.random() * userIds.length)]
            const messages = [
              'Hello everyone!',
              'How is everyone doing?',
              'Great weather today!',
              'Anyone working on interesting projects?',
              'Thanks for the help!',
              'See you later!',
              'What do you think about this?',
              'Good morning!',
              'Have a great day!'
            ]
            
            const message: Message = {
              id: `msg-${Date.now()}`,
              senderId,
              content: messages[Math.floor(Math.random() * messages.length)],
              timestamp: new Date(),
              type: 'text'
            }
            
            yield* Ref.update(roomRef, room => ({
              ...room,
              messages: [...room.messages.slice(-49), message], // Keep last 50 messages
              lastActivity: new Date()
            }))
          }
        } else {
          // Typing indicator
          const userIds = Array.from(currentRoom.users.keys())
          if (userIds.length > 0) {
            const userId = userIds[Math.floor(Math.random() * userIds.length)]
            const isTyping = Math.random() > 0.5
            
            yield* Ref.update(roomRef, room => {
              const newTypingUsers = new Set(room.typingUsers)
              if (isTyping) {
                newTypingUsers.add(userId)
              } else {
                newTypingUsers.delete(userId)
              }
              
              return {
                ...room,
                typingUsers: newTypingUsers,
                lastActivity: new Date()
              }
            })
            
            // Clear typing indicator after 3 seconds
            yield* Effect.fork(
              Effect.gen(function* () {
                yield* Effect.sleep(Duration.seconds(3))
                yield* Ref.update(roomRef, room => {
                  const newTypingUsers = new Set(room.typingUsers)
                  newTypingUsers.delete(userId)
                  return { ...room, typingUsers: newTypingUsers }
                })
              })
            )
          }
        }
        
        return yield* roomRef.get
      })
    ).pipe(
      Stream.schedule(Schedule.fixed(Duration.seconds(1)))
    )
    
    const chatSubscribable = Subscribable.make({
      get: roomRef.get,
      changes: chatEvents
    })
    
    // Derived subscribables for different aspects
    const userPresence = chatSubscribable.pipe(
      Subscribable.map(room => ({
        onlineCount: Array.from(room.users.values())
          .filter(user => user.status === 'online').length,
        users: Array.from(room.users.values())
          .sort((a, b) => {
            if (a.status !== b.status) {
              const statusOrder = { online: 0, away: 1, offline: 2 }
              return statusOrder[a.status] - statusOrder[b.status]
            }
            return a.name.localeCompare(b.name)
          }),
        typingUsers: Array.from(room.typingUsers).map(id => 
          room.users.get(id)?.name || 'Unknown'
        )
      }))
    )
    
    const recentMessages = chatSubscribable.pipe(
      Subscribable.map(room => 
        room.messages
          .slice(-10) // Last 10 messages
          .map(msg => ({
            ...msg,
            senderName: msg.senderId === 'system' 
              ? 'System' 
              : room.users.get(msg.senderId)?.name || 'Unknown'
          }))
      )
    )
    
    const roomActivity = chatSubscribable.pipe(
      Subscribable.map(room => ({
        messageCount: room.messages.length,
        userCount: room.users.size,
        lastActivity: room.lastActivity,
        isActive: Date.now() - room.lastActivity.getTime() < 30000 // Active within 30s
      }))
    )
    
    // Helper functions for interaction
    const addUser = (user: User) =>
      Ref.update(roomRef, room => ({
        ...room,
        users: new Map(room.users).set(user.id, user),
        lastActivity: new Date()
      }))
    
    const removeUser = (userId: string) =>
      Ref.update(roomRef, room => {
        const newUsers = new Map(room.users)
        newUsers.delete(userId)
        const newTypingUsers = new Set(room.typingUsers)
        newTypingUsers.delete(userId)
        
        return {
          ...room,
          users: newUsers,
          typingUsers: newTypingUsers,
          lastActivity: new Date()
        }
      })
    
    const sendMessage = (senderId: string, content: string) =>
      Ref.update(roomRef, room => {
        const message: Message = {
          id: `msg-${Date.now()}-${Math.random()}`,
          senderId,
          content,
          timestamp: new Date(),
          type: 'text'
        }
        
        return {
          ...room,
          messages: [...room.messages.slice(-49), message],
          lastActivity: new Date()
        }
      })
    
    return {
      chat: chatSubscribable,
      presence: userPresence,
      messages: recentMessages,
      activity: roomActivity,
      addUser,
      removeUser,
      sendMessage
    }
  })

// Multi-room chat manager
const makeChatManager = (rooms: string[]) => Effect.gen(function* () {
  const chatRooms = yield* Effect.all(
    rooms.map(roomId =>
      makeChatRoomSubscribable(roomId, `Room ${roomId}`).pipe(
        Effect.map(room => [roomId, room] as const)
      )
    )
  )
  
  const roomMap = new Map(chatRooms)
  
  // Global chat statistics
  const globalStatsRef = yield* Ref.make({
    totalUsers: 0,
    totalMessages: 0,
    activeRooms: 0,
    lastUpdate: new Date()
  })
  
  // Update global stats when any room changes
  const globalUpdates = Stream.mergeAll(
    Array.from(roomMap.values()).map(room => room.chat.changes),
    { concurrency: 'unbounded' }
  ).pipe(
    Stream.tap(_ =>
      Effect.gen(function* () {
        let totalUsers = 0
        let totalMessages = 0
        let activeRooms = 0
        
        for (const room of roomMap.values()) {
          const chatRoom = yield* room.chat.get
          const activity = yield* room.activity.get
          
          totalUsers += chatRoom.users.size
          totalMessages += chatRoom.messages.length
          if (activity.isActive) activeRooms++
        }
        
        yield* Ref.set(globalStatsRef, {
          totalUsers,
          totalMessages,
          activeRooms,
          lastUpdate: new Date()
        })
      })
    ),
    Stream.map(_ => globalStatsRef.get),
    Stream.flatMap(Stream.fromEffect)
  )
  
  const globalStats = Subscribable.make({
    get: globalStatsRef.get,
    changes: globalUpdates
  })
  
  return {
    rooms: roomMap,
    globalStats
  }
})

// Chat application usage
const chatApp = Effect.gen(function* () {
  const chatManager = yield* makeChatManager(['general', 'random', 'dev'])
  
  console.log('=== Chat System Started ===')
  
  // Monitor global chat statistics
  const initialStats = yield* chatManager.globalStats.get
  console.log('Initial Stats:', initialStats)
  
  // Monitor a specific room
  const generalRoom = chatManager.rooms.get('general')!
  
  console.log('\n=== Monitoring General Room ===')
  yield* generalRoom.chat.changes.pipe(
    Stream.take(20),
    Stream.tap(room =>
      Effect.gen(function* () {
        const presence = yield* generalRoom.presence.get
        const messages = yield* generalRoom.messages.get
        const activity = yield* generalRoom.activity.get
        
        console.log(`[${room.lastActivity.toLocaleTimeString()}] General Room:`)
        console.log(`  Users: ${presence.onlineCount} online`)
        
        if (presence.typingUsers.length > 0) {
          console.log(`  Typing: ${presence.typingUsers.join(', ')}`)
        }
        
        const lastMessage = messages[messages.length - 1]
        if (lastMessage) {
          console.log(`  Last: [${lastMessage.senderName}] ${lastMessage.content}`)
        }
        
        console.log(`  Activity: ${activity.messageCount} messages, ${activity.isActive ? 'active' : 'idle'}`)
        console.log('---')
      })
    ),
    Stream.runDrain
  )
  
  // Show final statistics
  const finalStats = yield* chatManager.globalStats.get
  console.log('\n=== Final Statistics ===')
  console.log(`Total Users: ${finalStats.totalUsers}`)
  console.log(`Total Messages: ${finalStats.totalMessages}`)
  console.log(`Active Rooms: ${finalStats.activeRooms}`)
  
  // Show room summaries
  console.log('\n=== Room Summaries ===')
  for (const [roomId, room] of chatManager.rooms) {
    const chatRoom = yield* room.chat.get
    const presence = yield* room.presence.get
    const activity = yield* room.activity.get
    
    console.log(`${roomId}:`)
    console.log(`  Users: ${presence.onlineCount} (${presence.users.map(u => u.name).join(', ')})`)
    console.log(`  Messages: ${activity.messageCount}`)
    console.log(`  Status: ${activity.isActive ? 'active' : 'idle'}`)
  }
})
```

## Advanced Features Deep Dive

### Unwrapping Nested Subscribable Effects

The `unwrap` function handles complex scenarios where you have a Subscribable inside an Effect, enabling dynamic reactive system creation.

```typescript
import { Effect, Subscribable, Ref, Stream, Schedule, Duration, Context, Layer } from "effect"

// Configuration service for dynamic subscribable creation
interface ConfigService {
  readonly getDataSource: Effect.Effect<string>
  readonly getUpdateInterval: Effect.Effect<Duration.Duration>
  readonly getFilterCriteria: Effect.Effect<(value: number) => boolean>
}

const ConfigService = Context.GenericTag<ConfigService>('ConfigService')

// Dynamic data source factory
const makeDataSourceSubscribable = (sourceId: string) => Effect.gen(function* () {
  const dataRef = yield* Ref.make(0)
  
  // Data stream that updates based on source configuration
  const dataStream = Stream.repeatEffect(
    Effect.gen(function* () {
      // Simulate different data sources
      let newValue: number
      switch (sourceId) {
        case 'sensor-a':
          newValue = Math.random() * 100
          break
        case 'sensor-b':
          newValue = Math.sin(Date.now() / 1000) * 50 + 50
          break
        case 'sensor-c':
          newValue = Math.random() * 200 - 100
          break
        default:
          newValue = Math.random() * 10
      }
      
      yield* Ref.set(dataRef, newValue)
      return newValue
    })
  ).pipe(
    Stream.schedule(Schedule.fixed(Duration.seconds(1)))
  )
  
  return Subscribable.make({
    get: dataRef.get,
    changes: dataStream
  })
})

// Dynamic subscribable system based on configuration
const makeDynamicMonitoringSystem = Effect.gen(function* () {
  const config = yield* ConfigService
  
  // Effect that creates a Subscribable based on runtime configuration
  const subscribableEffect: Effect.Effect<Subscribable<number>> = Effect.gen(function* () {
    const dataSource = yield* config.getDataSource
    const updateInterval = yield* config.getUpdateInterval
    const filterCriteria = yield* config.getFilterCriteria
    
    console.log(`Creating subscribable for data source: ${dataSource}`)
    console.log(`Update interval: ${Duration.toMillis(updateInterval)}ms`)
    
    // Create base subscribable
    const baseSubscribable = yield* makeDataSourceSubscribable(dataSource)
    
    // Apply dynamic filtering and interval configuration
    const configuredSubscribable = Subscribable.make({
      get: baseSubscribable.get.pipe(
        Effect.flatMap(value => 
          filterCriteria(value) 
            ? Effect.succeed(value)
            : Effect.succeed(0) // Default value when filtered out
        )
      ),
      changes: baseSubscribable.changes.pipe(
        Stream.schedule(Schedule.fixed(updateInterval)),
        Stream.filter(filterCriteria)
      )
    })
    
    return configuredSubscribable
  })
  
  // Unwrap the Effect<Subscribable<number>> to get Subscribable<number>
  const dynamicSubscribable = Subscribable.unwrap(subscribableEffect)
  
  // Add derived computations
  const smoothedValues = dynamicSubscribable.pipe(
    Subscribable.mapEffect(value => 
      Effect.gen(function* () {
        // Simulate smoothing algorithm
        yield* Effect.sleep(Duration.millis(10))
        return Math.round(value * 100) / 100
      })
    )
  )
  
  const alertSystem = smoothedValues.pipe(
    Subscribable.map(value => {
      if (value > 80) return `🔴 HIGH: ${value.toFixed(2)}`
      if (value > 60) return `🟡 MEDIUM: ${value.toFixed(2)}`
      return `🟢 NORMAL: ${value.toFixed(2)}`
    })
  )
  
  return {
    data: dynamicSubscribable,
    smoothed: smoothedValues,
    alerts: alertSystem
  }
})

// Different configuration implementations
const developmentConfig: ConfigService = {
  getDataSource: Effect.succeed('sensor-a'),
  getUpdateInterval: Effect.succeed(Duration.millis(500)),
  getFilterCriteria: Effect.succeed((value: number) => value > 10)
}

const productionConfig: ConfigService = {
  getDataSource: Effect.succeed('sensor-b'),
  getUpdateInterval: Effect.succeed(Duration.seconds(2)),
  getFilterCriteria: Effect.succeed((value: number) => value > 25)
}

const testingConfig: ConfigService = {
  getDataSource: Effect.succeed('sensor-c'),
  getUpdateInterval: Effect.succeed(Duration.millis(100)),
  getFilterCriteria: Effect.succeed((_: number) => true) // No filtering
}

// Usage with different configurations
const unwrapExample = Effect.gen(function* () {
  const monitoringSystem = yield* makeDynamicMonitoringSystem
  
  console.log('=== Dynamic Monitoring System Started ===')
  
  // Access the unwrapped subscribable
  const initialData = yield* monitoringSystem.data.get
  const initialSmoothed = yield* monitoringSystem.smoothed.get
  const initialAlert = yield* monitoringSystem.alerts.get
  
  console.log(`Initial data: ${initialData}`)
  console.log(`Smoothed: ${initialSmoothed}`)
  console.log(`Alert: ${initialAlert}`)
  
  // Monitor changes for a period
  console.log('\n=== Live Monitoring ===')
  yield* monitoringSystem.data.changes.pipe(
    Stream.take(10),
    Stream.tap(value =>
      Effect.gen(function* () {
        const smoothed = yield* monitoringSystem.smoothed.get
        const alert = yield* monitoringSystem.alerts.get
        
        console.log(
          `Raw: ${value.toFixed(2)} | ` +
          `Smoothed: ${smoothed.toFixed(2)} | ` +
          `Status: ${alert}`
        )
      })
    ),
    Stream.runDrain
  )
})

// Run with different configurations
const runWithDevelopmentConfig = unwrapExample.pipe(
  Effect.provide(Layer.succeed(ConfigService, developmentConfig))
)

const runWithProductionConfig = unwrapExample.pipe(
  Effect.provide(Layer.succeed(ConfigService, productionConfig))
)
```

### Complex Stream Transformations with Subscribable

Subscribable integrates seamlessly with Effect's Stream module for sophisticated data processing pipelines.

```typescript
import { 
  Effect, Subscribable, Ref, Stream, Schedule, Duration, 
  Chunk, Array as Arr, Option, Either 
} from "effect"

interface DataPoint {
  readonly timestamp: Date
  readonly value: number
  readonly source: string
  readonly quality: 'high' | 'medium' | 'low'
}

interface ProcessedData {
  readonly rawData: readonly DataPoint[]
  readonly movingAverage: number
  readonly trend: 'up' | 'down' | 'stable'
  readonly anomalies: readonly DataPoint[]
  readonly summary: {
    readonly count: number
    readonly min: number
    readonly max: number
    readonly avg: number
    readonly stdDev: number
  }
}

const makeAdvancedDataProcessor = (sourceId: string) => Effect.gen(function* () {
  const dataHistoryRef = yield* Ref.make<readonly DataPoint[]>([])
  
  // Raw data stream with realistic patterns
  const rawDataStream = Stream.repeatEffect(
    Effect.gen(function* () {
      // Generate realistic time-series data with trends and noise
      const now = new Date()
      const baseValue = 50 + Math.sin(Date.now() / 10000) * 20 // Sine wave base
      const noise = (Math.random() - 0.5) * 10 // Random noise
      const spike = Math.random() < 0.05 ? (Math.random() - 0.5) * 50 : 0 // Occasional spikes
      
      const dataPoint: DataPoint = {
        timestamp: now,
        value: Math.max(0, baseValue + noise + spike),
        source: sourceId,
        quality: Math.random() > 0.1 ? 'high' : Math.random() > 0.5 ? 'medium' : 'low'
      }
      
      // Update history (keep last 100 points)
      yield* Ref.update(dataHistoryRef, history => 
        [...history.slice(-99), dataPoint]
      )
      
      return dataPoint
    })
  ).pipe(
    Stream.schedule(Schedule.fixed(Duration.millis(200)))
  )
  
  // Complex stream processing pipeline
  const processedDataStream = rawDataStream.pipe(
    // Batch processing - collect data in windows
    Stream.groupedWithin(10, Duration.seconds(2)),
    
    // Process each batch
    Stream.mapEffect(chunk =>
      Effect.gen(function* () {
        const dataPoints = Chunk.toReadonlyArray(chunk)
        const history = yield* dataHistoryRef.get
        const recentHistory = history.slice(-50) // Last 50 points for analysis
        
        // Calculate moving average
        const validPoints = recentHistory.filter(p => p.quality !== 'low')
        const movingAverage = validPoints.length > 0
          ? validPoints.reduce((sum, p) => sum + p.value, 0) / validPoints.length
          : 0
        
        // Detect trend
        if (validPoints.length >= 10) {
          const recent = validPoints.slice(-10)
          const older = validPoints.slice(-20, -10)
          const recentAvg = recent.reduce((sum, p) => sum + p.value, 0) / recent.length
          const olderAvg = older.length > 0 
            ? older.reduce((sum, p) => sum + p.value, 0) / older.length
            : recentAvg
          
          const trend = recentAvg > olderAvg * 1.05 ? 'up' 
                      : recentAvg < olderAvg * 0.95 ? 'down' 
                      : 'stable'
          
          // Detect anomalies using simple statistical method
          const values = validPoints.map(p => p.value)
          const mean = values.reduce((sum, v) => sum + v, 0) / values.length
          const variance = values.reduce((sum, v) => sum + Math.pow(v - mean, 2), 0) / values.length
          const stdDev = Math.sqrt(variance)
          const threshold = 2 * stdDev
          
          const anomalies = dataPoints.filter(p => 
            Math.abs(p.value - mean) > threshold
          )
          
          // Calculate summary statistics
          const allValues = recentHistory.map(p => p.value)
          const summary = {
            count: allValues.length,
            min: Math.min(...allValues),
            max: Math.max(...allValues),
            avg: mean,
            stdDev
          }
          
          const processed: ProcessedData = {
            rawData: dataPoints,
            movingAverage,
            trend,
            anomalies,
            summary
          }
          
          return processed
        }
        
        // Fallback for insufficient data
        return {
          rawData: dataPoints,
          movingAverage: 0,
          trend: 'stable' as const,
          anomalies: [],
          summary: {
            count: 0,
            min: 0,
            max: 0,
            avg: 0,
            stdDev: 0
          }
        }
      })
    ),
    
    // Filter out empty results
    Stream.filter(data => data.summary.count > 0)
  )
  
  // Current processed state
  const processedStateRef = yield* Ref.make<Option.Option<ProcessedData>>(Option.none())
  
  // Update state when new processed data arrives
  const stateUpdateStream = processedDataStream.pipe(
    Stream.tap(data => Ref.set(processedStateRef, Option.some(data))),
    Stream.map(_ => processedStateRef.get),
    Stream.flatMap(Stream.fromEffect)
  )
  
  const dataProcessor = Subscribable.make({
    get: processedStateRef.get,
    changes: stateUpdateStream
  })
  
  // Derived subscribables for specific use cases
  const qualityMetrics = dataProcessor.pipe(
    Subscribable.map(dataOption =>
      Option.match(dataOption, {
        onNone: () => ({ quality: 0, reliability: 0, completeness: 0 }),
        onSome: data => {
          const highQuality = data.rawData.filter(p => p.quality === 'high').length
          const total = data.rawData.length
          const quality = total > 0 ? (highQuality / total) * 100 : 0
          
          return {
            quality,
            reliability: data.anomalies.length < 2 ? 100 : Math.max(0, 100 - data.anomalies.length * 10),
            completeness: data.summary.count > 0 ? 100 : 0
          }
        }
      })
    )
  )
  
  const alertStatus = dataProcessor.pipe(
    Subscribable.mapEffect(dataOption =>
      Effect.succeed(
        Option.match(dataOption, {
          onNone: () => '⏳ Waiting for data...',
          onSome: data => {
            const alerts: string[] = []
            
            if (data.anomalies.length > 0) {
              alerts.push(`🚨 ${data.anomalies.length} anomalies detected`)
            }
            
            if (data.trend === 'up' && data.movingAverage > 80) {
              alerts.push('📈 High upward trend')
            }
            
            if (data.trend === 'down' && data.movingAverage < 20) {
              alerts.push('📉 Low downward trend')
            }
            
            if (data.summary.stdDev > 20) {
              alerts.push('⚠️ High volatility')
            }
            
            return alerts.length > 0 
              ? alerts.join(' | ')
              : `✅ Normal operation (Avg: ${data.movingAverage.toFixed(1)})`
          }
        })
      )
    )
  )
  
  return {
    processor: dataProcessor,
    quality: qualityMetrics,
    alerts: alertStatus,
    rawHistory: dataHistoryRef as Ref.Ref<readonly DataPoint[]>
  }
})

// Advanced data processing usage
const advancedProcessingExample = Effect.gen(function* () {
  const processors = yield* Effect.all([
    makeAdvancedDataProcessor('sensor-alpha'),
    makeAdvancedDataProcessor('sensor-beta'),
    makeAdvancedDataProcessor('sensor-gamma')
  ])
  
  console.log('=== Advanced Data Processing System ===')
  console.log('Started 3 data processors with complex stream transformations')
  
  // Monitor all processors simultaneously
  const monitorProcessors = Effect.gen(function* () {
    for (let i = 0; i < 15; i++) {
      console.log(`\n--- Update ${i + 1} ---`)
      
      for (const [index, processor] of processors.entries()) {
        const sensorName = ['Alpha', 'Beta', 'Gamma'][index]
        const data = yield* processor.processor.get
        const quality = yield* processor.quality.get
        const alert = yield* processor.alerts.get
        
        Option.match(data, {
          onNone: () => console.log(`${sensorName}: No data yet`),
          onSome: processedData => {
            console.log(`${sensorName}:`)
            console.log(`  Trend: ${processedData.trend} | Avg: ${processedData.movingAverage.toFixed(1)}`)
            console.log(`  Quality: ${quality.quality.toFixed(1)}% | Reliability: ${quality.reliability.toFixed(1)}%`)
            console.log(`  Anomalies: ${processedData.anomalies.length} | StdDev: ${processedData.summary.stdDev.toFixed(1)}`)
            console.log(`  Alert: ${alert}`)
          }
        })
      }
      
      yield* Effect.sleep(Duration.seconds(3))
    }
  })
  
  yield* monitorProcessors
  
  // Show final statistics
  console.log('\n=== Final Analysis ===')
  for (const [index, processor] of processors.entries()) {
    const sensorName = ['Alpha', 'Beta', 'Gamma'][index]
    const history = yield* processor.rawHistory.get
    const data = yield* processor.processor.get
    
    console.log(`\n${sensorName} Summary:`)
    console.log(`  Total data points: ${history.length}`)
    
    Option.match(data, {
      onNone: () => console.log('  No processed data available'),
      onSome: processedData => {
        console.log(`  Moving average: ${processedData.movingAverage.toFixed(2)}`)
        console.log(`  Trend: ${processedData.trend}`)
        console.log(`  Value range: ${processedData.summary.min.toFixed(1)} - ${processedData.summary.max.toFixed(1)}`)
        console.log(`  Standard deviation: ${processedData.summary.stdDev.toFixed(2)}`)
        console.log(`  Total anomalies detected: ${processedData.anomalies.length}`)
      }
    })
  }
})
```

## Practical Patterns & Best Practices

### Subscribable Service Layer Pattern

Create reusable service layers that provide reactive capabilities across your application architecture.

```typescript
import { Effect, Subscribable, Ref, Context, Layer, Stream, Schedule, Duration } from "effect"

// Core reactive data service interface
interface ReactiveDataService {
  readonly createSubscribable: <T>(
    key: string,
    initialValue: T,
    updateStrategy: 'manual' | 'polling' | 'event-driven'
  ) => Effect.Effect<Subscribable<T>>
  
  readonly getSubscribable: <T>(key: string) => Effect.Effect<Option.Option<Subscribable<T>>>
  readonly updateValue: <T>(key: string, value: T) => Effect.Effect<void>
  readonly removeSubscribable: (key: string) => Effect.Effect<void>
  readonly listKeys: () => Effect.Effect<readonly string[]>
}

const ReactiveDataService = Context.GenericTag<ReactiveDataService>('ReactiveDataService')

// Service implementation with centralized subscribable management
const makeReactiveDataService = Effect.gen(function* () {
  const subscribablesRef = yield* Ref.make<Map<string, {
    subscribable: Subscribable<any>
    valueRef: Ref.Ref<any>
    strategy: 'manual' | 'polling' | 'event-driven'
  }>>(new Map())
  
  const createSubscribable = <T>(
    key: string,
    initialValue: T,
    updateStrategy: 'manual' | 'polling' | 'event-driven'
  ) => Effect.gen(function* () {
    const existing = yield* Ref.get(subscribablesRef)
    
    if (existing.has(key)) {
      return existing.get(key)!.subscribable as Subscribable<T>
    }
    
    const valueRef = yield* Ref.make<T>(initialValue)
    
    // Create different update streams based on strategy
    const updateStream = (() => {
      switch (updateStrategy) {
        case 'manual':
          // Only updates when explicitly triggered
          return Stream.fromEffect(valueRef.get).pipe(
            Stream.concat(Stream.never)
          )
          
        case 'polling':
          // Regular polling updates
          return Stream.repeatEffect(valueRef.get).pipe(
            Stream.schedule(Schedule.fixed(Duration.seconds(5)))
          )
          
        case 'event-driven':
          // Simulated event-driven updates
          return Stream.repeatEffect(
            Effect.gen(function* () {
              // Simulate external events
              if (Math.random() < 0.3) {
                // 30% chance of update each second
                return yield* valueRef.get
              }
              return yield* Effect.never // Wait for actual event
            })
          ).pipe(
            Stream.schedule(Schedule.fixed(Duration.seconds(1)))
          )
      }
    })()
    
    const subscribable = Subscribable.make({
      get: valueRef.get,
      changes: updateStream
    })
    
    // Register in the service
    yield* Ref.update(subscribablesRef, map =>
      new Map(map).set(key, { subscribable, valueRef, strategy: updateStrategy })
    )
    
    return subscribable
  })
  
  const getSubscribable = <T>(key: string) => Effect.gen(function* () {
    const subscribables = yield* Ref.get(subscribablesRef)
    const entry = subscribables.get(key)
    return entry ? Option.some(entry.subscribable as Subscribable<T>) : Option.none()
  })
  
  const updateValue = <T>(key: string, value: T) => Effect.gen(function* () {
    const subscribables = yield* Ref.get(subscribablesRef)
    const entry = subscribables.get(key)
    
    if (entry) {
      yield* Ref.set(entry.valueRef, value)
    } else {
      return yield* Effect.fail(new Error(`Subscribable not found: ${key}`))
    }
  })
  
  const removeSubscribable = (key: string) =>
    Ref.update(subscribablesRef, map => {
      const newMap = new Map(map)
      newMap.delete(key)
      return newMap
    })
  
  const listKeys = () => Effect.gen(function* () {
    const subscribables = yield* Ref.get(subscribablesRef)
    return Array.from(subscribables.keys())
  })
  
  return {
    createSubscribable,
    getSubscribable,
    updateValue,
    removeSubscribable,
    listKeys
  } satisfies ReactiveDataService
})

const ReactiveDataServiceLive = Layer.effect(ReactiveDataService, makeReactiveDataService)

// Application-specific service using the reactive data service
interface UserSessionService {
  readonly currentUser: Subscribable<Option.Option<User>>
  readonly sessionMetrics: Subscribable<SessionMetrics>
  readonly notifications: Subscribable<readonly Notification[]>
  
  readonly login: (user: User) => Effect.Effect<void>
  readonly logout: () => Effect.Effect<void>
  readonly addNotification: (notification: Notification) => Effect.Effect<void>
  readonly clearNotifications: () => Effect.Effect<void>
}

const UserSessionService = Context.GenericTag<UserSessionService>('UserSessionService')

const makeUserSessionService = Effect.gen(function* () {
  const dataService = yield* ReactiveDataService
  
  // Create subscribables for different aspects of user session
  const currentUser = yield* dataService.createSubscribable(
    'current-user',
    Option.none<User>(),
    'manual'
  )
  
  const sessionMetrics = yield* dataService.createSubscribable(
    'session-metrics',
    {
      loginTime: null,
      lastActivity: new Date(),
      pageViews: 0,
      actionsPerformed: 0
    } as SessionMetrics,
    'polling'
  )
  
  const notifications = yield* dataService.createSubscribable(
    'notifications',
    [] as readonly Notification[],
    'manual'
  )
  
  // Service operations
  const login = (user: User) => Effect.gen(function* () {
    yield* dataService.updateValue('current-user', Option.some(user))
    yield* dataService.updateValue('session-metrics', {
      loginTime: new Date(),
      lastActivity: new Date(),
      pageViews: 1,
      actionsPerformed: 0
    })
  })
  
  const logout = () => Effect.gen(function* () {
    yield* dataService.updateValue('current-user', Option.none())
    yield* dataService.updateValue('session-metrics', {
      loginTime: null,
      lastActivity: new Date(),
      pageViews: 0,
      actionsPerformed: 0
    })
    yield* dataService.updateValue('notifications', [])
  })
  
  const addNotification = (notification: Notification) => Effect.gen(function* () {
    const currentNotifications = yield* notifications.get
    yield* dataService.updateValue('notifications', [...currentNotifications, notification])
  })
  
  const clearNotifications = () =>
    dataService.updateValue('notifications', [])
  
  return {
    currentUser,
    sessionMetrics,
    notifications,
    login,
    logout,
    addNotification,
    clearNotifications
  } satisfies UserSessionService
})

const UserSessionServiceLive = Layer.effect(UserSessionService, makeUserSessionService)

// Application layer that combines both services
const AppServiceLayer = Layer.empty.pipe(
  Layer.provide(ReactiveDataServiceLive),
  Layer.provide(UserSessionServiceLive)
)

// Usage example
const serviceLayerExample = Effect.gen(function* () {
  const userSession = yield* UserSessionService
  const dataService = yield* ReactiveDataService
  
  console.log('=== Reactive Service Layer Demo ===')
  
  // Show available subscribables
  const keys = yield* dataService.listKeys()
  console.log('Available subscribables:', keys)
  
  // Monitor user session changes
  const sessionMonitor = Effect.gen(function* () {
    yield* userSession.currentUser.changes.pipe(
      Stream.take(5),
      Stream.tap(userOption =>
        Effect.sync(() => {
          Option.match(userOption, {
            onNone: () => console.log('👤 User logged out'),
            onSome: user => console.log(`👤 User logged in: ${user.name}`)
          })
        })
      ),
      Stream.runDrain
    )
  })
  
  // Monitor notifications
  const notificationMonitor = Effect.gen(function* () {
    yield* userSession.notifications.changes.pipe(
      Stream.take(3),
      Stream.tap(notifications =>
        Effect.sync(() => {
          if (notifications.length > 0) {
            console.log(`🔔 ${notifications.length} notifications:`)
            notifications.forEach(n => console.log(`  - ${n.message}`))
          }
        })
      ),
      Stream.runDrain
    )
  })
  
  // Start monitoring
  yield* Effect.fork(sessionMonitor)
  yield* Effect.fork(notificationMonitor)
  
  // Simulate user interactions
  yield* Effect.sleep(Duration.seconds(1))
  
  yield* userSession.login({
    id: 'user-123',
    name: 'Alice Johnson',
    email: 'alice@example.com'
  })
  
  yield* Effect.sleep(Duration.seconds(1))
  
  yield* userSession.addNotification({
    id: 'notif-1',
    message: 'Welcome to the application!',
    type: 'info',
    timestamp: new Date()
  })
  
  yield* Effect.sleep(Duration.seconds(1))
  
  yield* userSession.addNotification({
    id: 'notif-2',
    message: 'Your profile has been updated',
    type: 'success',
    timestamp: new Date()
  })
  
  yield* Effect.sleep(Duration.seconds(2))
  
  yield* userSession.logout()
  
  yield* Effect.sleep(Duration.seconds(1))
  
  // Show final state
  const finalUser = yield* userSession.currentUser.get
  const finalNotifications = yield* userSession.notifications.get
  
  console.log('\n=== Final State ===')
  console.log('User:', Option.isNone(finalUser) ? 'Logged out' : finalUser.value.name)
  console.log('Notifications:', finalNotifications.length)
  
}).pipe(Effect.provide(AppServiceLayer))

// Supporting types
interface User {
  readonly id: string
  readonly name: string
  readonly email: string
}

interface SessionMetrics {
  readonly loginTime: Date | null
  readonly lastActivity: Date
  readonly pageViews: number
  readonly actionsPerformed: number
}

interface Notification {
  readonly id: string
  readonly message: string
  readonly type: 'info' | 'success' | 'warning' | 'error'
  readonly timestamp: Date
}
```

### Resource Management and Cleanup Pattern

Implement automatic resource management for subscribables that handle external resources.

```typescript
import { 
  Effect, Subscribable, Ref, Stream, Schedule, Duration, 
  Scope, Context, Layer, Data 
} from "effect"

class ResourceError extends Data.TaggedError("ResourceError")<{
  resource: string
  reason: string
}> {}

// Resource-aware subscribable that handles cleanup
interface ManagedSubscribable<T> {
  readonly subscribable: Subscribable<T>
  readonly cleanup: Effect.Effect<void>
  readonly status: Subscribable<'active' | 'cleaning' | 'closed'>
}

// File watcher subscribable with automatic cleanup
const makeFileWatcherSubscribable = (filePath: string) =>
  Effect.gen(function* () {
    const contentRef = yield* Ref.make('')
    const statusRef = yield* Ref.make<'active' | 'cleaning' | 'closed'>('active')
    
    // Simulate file watching (in real app, would use fs.watch)
    const fileChangeStream = Stream.repeatEffect(
      Effect.gen(function* () {
        const status = yield* statusRef.get
        if (status !== 'active') {
          return yield* Effect.never // Stop if not active
        }
        
        // Simulate file content changes
        const content = `File content updated at ${new Date().toISOString()}`
        yield* Ref.set(contentRef, content)
        return content
      })
    ).pipe(
      Stream.schedule(Schedule.fixed(Duration.seconds(2))),
      Stream.takeUntil(
        Effect.gen(function* () {
          const status = yield* statusRef.get
          return status !== 'active'
        })
      )
    )
    
    const fileSubscribable = Subscribable.make({
      get: Effect.gen(function* () {
        const status = yield* statusRef.get
        if (status === 'closed') {
          return yield* Effect.fail(new ResourceError({
            resource: filePath,
            reason: 'File watcher has been closed'
          }))
        }
        return yield* contentRef.get
      }),
      changes: fileChangeStream
    })
    
    const statusSubscribable = Subscribable.make({
      get: statusRef.get,
      changes: Stream.fromRef(statusRef)
    })
    
    const cleanup = Effect.gen(function* () {
      yield* Ref.set(statusRef, 'cleaning')
      console.log(`🧹 Cleaning up file watcher for: ${filePath}`)
      
      // Simulate cleanup delay
      yield* Effect.sleep(Duration.millis(100))
      
      yield* Ref.set(statusRef, 'closed')
      console.log(`✨ File watcher cleaned up: ${filePath}`)
    })
    
    return {
      subscribable: fileSubscribable,
      cleanup,
      status: statusSubscribable
    } satisfies ManagedSubscribable<string>
  })

// Database connection pool subscribable
const makeDatabasePoolSubscribable = (connectionString: string) =>
  Effect.gen(function* () {
    const poolStatsRef = yield* Ref.make({
      activeConnections: 0,
      totalQueries: 0,
      avgResponseTime: 0,
      lastActivity: new Date()
    })
    
    const statusRef = yield* Ref.make<'active' | 'cleaning' | 'closed'>('active')
    
    // Simulate database pool monitoring
    const poolMonitoringStream = Stream.repeatEffect(
      Effect.gen(function* () {
        const status = yield* statusRef.get
        if (status !== 'active') {
          return yield* Effect.never
        }
        
        // Simulate changing pool statistics
        const currentStats = yield* poolStatsRef.get
        const newStats = {
          activeConnections: Math.max(0, currentStats.activeConnections + (Math.random() - 0.5) * 4),
          totalQueries: currentStats.totalQueries + Math.floor(Math.random() * 10),
          avgResponseTime: Math.max(1, currentStats.avgResponseTime + (Math.random() - 0.5) * 20),
          lastActivity: new Date()
        }
        
        yield* Ref.set(poolStatsRef, newStats)
        return newStats
      })
    ).pipe(
      Stream.schedule(Schedule.fixed(Duration.seconds(1)))
    )
    
    const poolSubscribable = Subscribable.make({
      get: Effect.gen(function* () {
        const status = yield* statusRef.get
        if (status === 'closed') {
          return yield* Effect.fail(new ResourceError({
            resource: connectionString,
            reason: 'Database pool has been closed'
          }))
        }
        return yield* poolStatsRef.get
      }),
      changes: poolMonitoringStream
    })
    
    const statusSubscribable = Subscribable.make({
      get: statusRef.get,
      changes: Stream.fromRef(statusRef)
    })
    
    const cleanup = Effect.gen(function* () {
      yield* Ref.set(statusRef, 'cleaning')
      console.log(`🗄️ Closing database connections: ${connectionString}`)
      
      // Simulate connection cleanup
      yield* Effect.sleep(Duration.millis(500))
      
      yield* Ref.set(statusRef, 'closed')
      console.log(`✅ Database pool closed: ${connectionString}`)
    })
    
    return {
      subscribable: poolSubscribable,
      cleanup,
      status: statusSubscribable
    } satisfies ManagedSubscribable<typeof poolStatsRef extends Ref.Ref<infer T> ? T : never>
  })

// Resource manager service
interface ResourceManagerService {
  readonly registerResource: <T>(
    name: string,
    resource: ManagedSubscribable<T>
  ) => Effect.Effect<void>
  
  readonly getResource: <T>(name: string) => Effect.Effect<Option.Option<Subscribable<T>>>
  readonly cleanupResource: (name: string) => Effect.Effect<void>
  readonly cleanupAll: () => Effect.Effect<void>
  readonly listResources: () => Effect.Effect<readonly string[]>
}

const ResourceManagerService = Context.GenericTag<ResourceManagerService>('ResourceManagerService')

const makeResourceManagerService = Effect.gen(function* () {
  const resourcesRef = yield* Ref.make<Map<string, ManagedSubscribable<any>>>(new Map())
  
  const registerResource = <T>(name: string, resource: ManagedSubscribable<T>) =>
    Ref.update(resourcesRef, resources => 
      new Map(resources).set(name, resource)
    )
  
  const getResource = <T>(name: string) => Effect.gen(function* () {
    const resources = yield* Ref.get(resourcesRef)
    const resource = resources.get(name)
    return resource ? Option.some(resource.subscribable as Subscribable<T>) : Option.none()
  })
  
  const cleanupResource = (name: string) => Effect.gen(function* () {
    const resources = yield* Ref.get(resourcesRef)
    const resource = resources.get(name)
    
    if (resource) {
      yield* resource.cleanup
      yield* Ref.update(resourcesRef, map => {
        const newMap = new Map(map)
        newMap.delete(name)
        return newMap
      })
    }
  })
  
  const cleanupAll = () => Effect.gen(function* () {
    const resources = yield* Ref.get(resourcesRef)
    
    console.log(`🧹 Cleaning up ${resources.size} resources...`)
    
    // Cleanup all resources concurrently
    yield* Effect.all(
      Array.from(resources.entries()).map(([name, resource]) =>
        Effect.gen(function* () {
          console.log(`Cleaning up resource: ${name}`)
          yield* resource.cleanup
        })
      ),
      { concurrency: 5 }
    )
    
    yield* Ref.set(resourcesRef, new Map())
    console.log('✨ All resources cleaned up')
  })
  
  const listResources = () => Effect.gen(function* () {
    const resources = yield* Ref.get(resourcesRef)
    return Array.from(resources.keys())
  })
  
  return {
    registerResource,
    getResource,
    cleanupResource,
    cleanupAll,
    listResources
  } satisfies ResourceManagerService
})

const ResourceManagerServiceLive = Layer.effect(
  ResourceManagerService,
  makeResourceManagerService
)

// Usage with automatic cleanup
const resourceManagementExample = Effect.gen(function* () {
  const resourceManager = yield* ResourceManagerService
  
  console.log('=== Resource Management Demo ===')
  
  // Create and register managed resources
  const fileWatcher = yield* makeFileWatcherSubscribable('/tmp/config.json')
  const dbPool = yield* makeDatabasePoolSubscribable('postgresql://localhost:5432/mydb')
  
  yield* resourceManager.registerResource('file-watcher', fileWatcher)
  yield* resourceManager.registerResource('db-pool', dbPool)
  
  const resources = yield* resourceManager.listResources()
  console.log('Registered resources:', resources)
  
  // Monitor resources
  const fileSubscribable = yield* resourceManager.getResource<string>('file-watcher')
  const dbSubscribable = yield* resourceManager.getResource<any>('db-pool')
  
  // Start monitoring both resources
  const monitorFile = Option.match(fileSubscribable, {
    onNone: () => Effect.unit,
    onSome: sub => sub.changes.pipe(
      Stream.take(3),
      Stream.tap(content => Effect.sync(() => 
        console.log(`📄 File updated: ${content.slice(0, 50)}...`)
      )),
      Stream.runDrain
    )
  })
  
  const monitorDb = Option.match(dbSubscribable, {
    onNone: () => Effect.unit,
    onSome: sub => sub.changes.pipe(
      Stream.take(3),
      Stream.tap(stats => Effect.sync(() => 
        console.log(
          `🗄️ DB Pool - Active: ${stats.activeConnections.toFixed(0)}, ` +
          `Queries: ${stats.totalQueries}, Avg: ${stats.avgResponseTime.toFixed(1)}ms`
        )
      )),
      Stream.runDrain
    )
  })
  
  // Run monitors concurrently
  yield* Effect.all([monitorFile, monitorDb], { concurrency: 'unbounded' })
  
  console.log('\n=== Resource Status Check ===')
  
  // Check resource status
  const fileStatus = yield* fileWatcher.status.get
  const dbStatus = yield* dbPool.status.get
  
  console.log(`File watcher status: ${fileStatus}`)
  console.log(`Database pool status: ${dbStatus}`)
  
  // Simulate early cleanup of one resource
  console.log('\n=== Cleaning up file watcher ===')
  yield* resourceManager.cleanupResource('file-watcher')
  
  const remainingResources = yield* resourceManager.listResources()
  console.log('Remaining resources:', remainingResources)
  
  // Continue monitoring remaining resource
  yield* Effect.sleep(Duration.seconds(2))
  
  console.log('\n=== Final cleanup ===')
  
}).pipe(
  Effect.provide(ResourceManagerServiceLive),
  Effect.ensuring(
    // Ensure cleanup happens even if the effect fails
    Effect.gen(function* () {
      const resourceManager = yield* ResourceManagerService
      yield* resourceManager.cleanupAll()
    }).pipe(Effect.provide(ResourceManagerServiceLive))
  )
)
```

## Integration Examples

### Integration with WebSocket and Real-Time APIs

Subscribable provides excellent integration with WebSocket connections and real-time API streams.

```typescript
import { Effect, Subscribable, Ref, Stream, Schedule, Duration, Data } from "effect"

// WebSocket-like interface for demonstration
interface WebSocketConnection {
  readonly send: (message: string) => Effect.Effect<void>
  readonly close: () => Effect.Effect<void>
  readonly onMessage: Stream.Stream<string>
  readonly onError: Stream.Stream<Error>
  readonly status: 'connecting' | 'connected' | 'disconnected' | 'error'
}

class WebSocketError extends Data.TaggedError("WebSocketError")<{
  reason: string
  url: string
}> {}

// Mock WebSocket implementation
const createMockWebSocket = (url: string): Effect.Effect<WebSocketConnection> =>
  Effect.gen(function* () {
    const statusRef = yield* Ref.make<WebSocketConnection['status']>('connecting')
    const messageBuffer = yield* Ref.make<string[]>([])
    
    // Simulate connection establishment
    yield* Effect.sleep(Duration.millis(500))
    yield* Ref.set(statusRef, 'connected')
    
    // Simulate incoming messages
    const messageStream = Stream.repeatEffect(
      Effect.gen(function* () {
        const status = yield* statusRef.get
        if (status !== 'connected') {
          return yield* Effect.never
        }
        
        // Simulate different types of messages
        const messageTypes = [
          '{"type":"heartbeat","timestamp":' + Date.now() + '}',
          '{"type":"data","payload":{"value":' + Math.random() * 100 + '}}',
          '{"type":"notification","message":"System update available"}',
          '{"type":"user_joined","userId":"user-' + Math.floor(Math.random() * 1000) + '"}'
        ]
        
        return messageTypes[Math.floor(Math.random() * messageTypes.length)]
      })
    ).pipe(
      Stream.schedule(Schedule.fixed(Duration.seconds(1)))
    )
    
    const errorStream = Stream.repeatEffect(
      Effect.gen(function* () {
        // Simulate occasional errors (5% chance)
        if (Math.random() < 0.05) {
          return new Error('Connection interrupted')
        }
        return yield* Effect.never
      })
    )
    
    const send = (message: string) =>
      Effect.gen(function* () {
        const status = yield* statusRef.get
        if (status !== 'connected') {
          return yield* Effect.fail(new WebSocketError({
            reason: 'WebSocket not connected',
            url
          }))
        }
        
        console.log(`📤 Sending: ${message}`)
        // Simulate send delay
        yield* Effect.sleep(Duration.millis(10))
      })
    
    const close = () =>
      Effect.gen(function* () {
        yield* Ref.set(statusRef, 'disconnected')
        console.log(`🔌 WebSocket closed: ${url}`)
      })
    
    return {
      send,
      close,
      onMessage: messageStream,
      onError: errorStream,
      get status() { return Ref.get(statusRef) as any }
    }
  })

// Real-time data subscribable using WebSocket
const makeRealTimeDataSubscribable = (apiUrl: string) =>
  Effect.gen(function* () {
    const connectionRef = yield* Ref.make<Option.Option<WebSocketConnection>>(Option.none())
    const dataRef = yield* Ref.make<{
      lastMessage: Option.Option<any>
      messageCount: number
      connected: boolean
      errors: readonly string[]
    }>({
      lastMessage: Option.none(),
      messageCount: 0,
      connected: false,
      errors: []
    })
    
    // Establish WebSocket connection
    const connection = yield* createMockWebSocket(apiUrl)
    yield* Ref.set(connectionRef, Option.some(connection))
    
    // Handle incoming messages
    const messageHandler = connection.onMessage.pipe(
      Stream.tap(message =>
        Effect.gen(function* () {
          try {
            const parsed = JSON.parse(message)
            
            yield* Ref.update(dataRef, current => ({
              ...current,
              lastMessage: Option.some(parsed),
              messageCount: current.messageCount + 1,
              connected: true
            }))
            
            console.log(`📥 Received: ${parsed.type}`)
          } catch (error) {
            yield* Ref.update(dataRef, current => ({
              ...current,
              errors: [...current.errors.slice(-9), `Parse error: ${message}`]
            }))
          }
        })
      ),
      Stream.runDrain,
      Effect.fork
    )
    
    // Handle errors
    const errorHandler = connection.onError.pipe(
      Stream.tap(error =>
        Ref.update(dataRef, current => ({
          ...current,
          connected: false,
          errors: [...current.errors.slice(-9), error.message]
        }))
      ),
      Stream.runDrain,
      Effect.fork
    )
    
    yield* messageHandler
    yield* errorHandler
    
    // Create data stream from state changes
    const dataStream = Stream.repeatEffect(dataRef.get).pipe(
      Stream.schedule(Schedule.fixed(Duration.millis(100))),
      Stream.changes // Only emit when data actually changes
    )
    
    const realTimeSubscribable = Subscribable.make({
      get: dataRef.get,
      changes: dataStream
    })
    
    // Message sending capability
    const sendMessage = (message: any) =>
      Effect.gen(function* () {
        const conn = yield* connectionRef.get
        
        yield* Option.match(conn, {
          onNone: () => Effect.fail(new WebSocketError({
            reason: 'No active connection',
            url: apiUrl
          })),
          onSome: connection => connection.send(JSON.stringify(message))
        })
      })
    
    // Connection management
    const disconnect = () =>
      Effect.gen(function* () {
        const conn = yield* connectionRef.get
        
        yield* Option.match(conn, {
          onNone: () => Effect.unit,
          onSome: connection => connection.close()
        })
        
        yield* Ref.set(connectionRef, Option.none())
        yield* Ref.update(dataRef, current => ({
          ...current,
          connected: false
        }))
      })
    
    return {
      data: realTimeSubscribable,
      sendMessage,
      disconnect
    }
  })

// Multi-channel real-time system
const makeMultiChannelSystem = (channels: readonly string[]) =>
  Effect.gen(function* () {
    // Create subscribables for each channel
    const channelSubscribables = yield* Effect.all(
      channels.map(channel =>
        makeRealTimeDataSubscribable(`wss://api.example.com/${channel}`).pipe(
          Effect.map(sub => [channel, sub] as const)
        )
      )
    )
    
    const channelMap = new Map(channelSubscribables)
    
    // Aggregated system status
    const systemStatusRef = yield* Ref.make({
      connectedChannels: 0,
      totalMessages: 0,
      totalErrors: 0,
      lastActivity: new Date()
    })
    
    // Update system status when any channel changes
    const statusUpdates = Stream.mergeAll(
      Array.from(channelMap.values()).map(sub => sub.data.changes),
      { concurrency: 'unbounded' }
    ).pipe(
      Stream.tap(_ =>
        Effect.gen(function* () {
          let connectedChannels = 0
          let totalMessages = 0
          let totalErrors = 0
          
          for (const sub of channelMap.values()) {
            const data = yield* sub.data.get
            if (data.connected) connectedChannels++
            totalMessages += data.messageCount
            totalErrors += data.errors.length
          }
          
          yield* Ref.set(systemStatusRef, {
            connectedChannels,
            totalMessages,
            totalErrors,
            lastActivity: new Date()
          })
        })
      ),
      Stream.map(_ => systemStatusRef.get),
      Stream.flatMap(Stream.fromEffect)
    )
    
    const systemStatus = Subscribable.make({
      get: systemStatusRef.get,
      changes: statusUpdates
    })
    
    // Broadcast message to all channels
    const broadcast = (message: any) =>
      Effect.all(
        Array.from(channelMap.values()).map(sub => sub.sendMessage(message)),
        { concurrency: 'unbounded' }
      )
    
    // Disconnect all channels
    const disconnectAll = () =>
      Effect.all(
        Array.from(channelMap.values()).map(sub => sub.disconnect()),
        { concurrency: 'unbounded' }
      )
    
    return {
      channels: channelMap,
      systemStatus,
      broadcast,
      disconnectAll
    }
  })

// Real-time application usage
const realTimeApp = Effect.gen(function* () {
  console.log('=== Real-Time WebSocket Integration ===')
  
  const system = yield* makeMultiChannelSystem(['trades', 'news', 'notifications'])
  
  // Monitor system status
  const statusMonitor = system.systemStatus.changes.pipe(
    Stream.take(10),
    Stream.tap(status =>
      Effect.sync(() => {
        console.log(
          `🌐 System Status - Connected: ${status.connectedChannels}/3, ` +
          `Messages: ${status.totalMessages}, Errors: ${status.totalErrors}`
        )
      })
    ),
    Stream.runDrain
  )
  
  // Monitor individual channels
  const channelMonitors = Array.from(system.channels.entries()).map(([name, sub]) =>
    sub.data.changes.pipe(
      Stream.take(5),
      Stream.tap(data =>
        Effect.gen(function* () {
          Option.match(data.lastMessage, {
            onNone: () => console.log(`📡 ${name}: No messages yet`),
            onSome: msg => console.log(`📡 ${name}: ${msg.type} (${data.messageCount} total)`)
          })
          
          if (data.errors.length > 0) {
            console.log(`⚠️ ${name} errors: ${data.errors.slice(-1)[0]}`)
          }
        })
      ),
      Stream.runDrain
    )
  )
  
  // Start all monitors
  yield* Effect.fork(statusMonitor)
  yield* Effect.all(channelMonitors.map(Effect.fork), { concurrency: 'unbounded' })
  
  // Wait for connections to establish
  yield* Effect.sleep(Duration.seconds(2))
  
  // Send test messages
  console.log('\n=== Broadcasting Messages ===')
  
  yield* system.broadcast({
    type: 'test',
    message: 'Hello from the application!',
    timestamp: Date.now()
  })
  
  yield* Effect.sleep(Duration.seconds(1))
  
  yield* system.broadcast({
    type: 'status_check',
    requestId: 'req-123'
  })
  
  // Wait for responses
  yield* Effect.sleep(Duration.seconds(5))
  
  // Show final statistics
  const finalStatus = yield* system.systemStatus.get
  console.log('\n=== Final Statistics ===')
  console.log(`Connected channels: ${finalStatus.connectedChannels}`)
  console.log(`Total messages processed: ${finalStatus.totalMessages}`)
  console.log(`Total errors: ${finalStatus.totalErrors}`)
  
  // Show channel details
  for (const [name, sub] of system.channels) {
    const data = yield* sub.data.get
    console.log(`${name}: ${data.messageCount} messages, ${data.connected ? 'connected' : 'disconnected'}`)
  }
  
  // Cleanup
  console.log('\n=== Cleaning up ===')
  yield* system.disconnectAll()
  
}).pipe(
  Effect.ensuring(
    Effect.sync(() => console.log('🧹 Real-time system cleaned up'))
  )
)
```

### Integration with HTTP Polling and REST APIs

Subscribable can elegantly handle HTTP polling scenarios and REST API integration.

```typescript
import { Effect, Subscribable, Ref, Stream, Schedule, Duration, Data, Option } from "effect"

// HTTP client interface (simplified)
interface HttpClient {
  readonly get: <T>(url: string) => Effect.Effect<T, HttpError>
  readonly post: <T, B>(url: string, body: B) => Effect.Effect<T, HttpError>
}

class HttpError extends Data.TaggedError("HttpError")<{
  status: number
  message: string
  url: string
}> {}

// Mock HTTP client
const mockHttpClient: HttpClient = {
  get: <T>(url: string) =>
    Effect.gen(function* () {
      // Simulate network delay
      yield* Effect.sleep(Duration.millis(100 + Math.random() * 200))
      
      // Simulate occasional failures
      if (Math.random() < 0.05) {
        return yield* Effect.fail(new HttpError({
          status: 500,
          message: 'Internal server error',
          url
        }))
      }
      
      // Mock different API responses based on URL
      if (url.includes('/api/weather')) {
        return {
          temperature: Math.random() * 30 + 10,
          humidity: Math.random() * 40 + 30,
          condition: ['sunny', 'cloudy', 'rainy'][Math.floor(Math.random() * 3)],
          timestamp: new Date().toISOString()
        } as T
      }
      
      if (url.includes('/api/stock')) {
        const symbol = url.split('/').pop()
        return {
          symbol,
          price: Math.random() * 100 + 50,
          change: (Math.random() - 0.5) * 10,
          volume: Math.floor(Math.random() * 10000) + 1000,
          timestamp: new Date().toISOString()
        } as T
      }
      
      if (url.includes('/api/system/health')) {
        return {
          status: Math.random() > 0.1 ? 'healthy' : 'degraded',
          uptime: Math.floor(Date.now() / 1000) - 86400,
          memory: Math.random() * 100,
          cpu: Math.random() * 100,
          timestamp: new Date().toISOString()
        } as T
      }
      
      return {} as T
    }),
  
  post: <T, B>(url: string, body: B) =>
    Effect.gen(function* () {
      yield* Effect.sleep(Duration.millis(150 + Math.random() * 300))
      
      if (Math.random() < 0.03) {
        return yield* Effect.fail(new HttpError({
          status: 400,
          message: 'Bad request',
          url
        }))
      }
      
      return { success: true, id: Math.random().toString(36) } as T
    })
}

// HTTP polling subscribable factory
const makePollingSubscribable = <T>(
  url: string,
  interval: Duration.Duration,
  httpClient: HttpClient
) => Effect.gen(function* () {
  const dataRef = yield* Ref.make<Option.Option<T>>(Option.none())
  const errorRef = yield* Ref.make<Option.Option<HttpError>>(Option.none())
  const lastFetchRef = yield* Ref.make<Option.Option<Date>>(Option.none())
  
  // Polling stream with error handling
  const pollingStream = Stream.repeatEffect(
    Effect.gen(function* () {
      try {
        const data = yield* httpClient.get<T>(url)
        yield* Ref.set(dataRef, Option.some(data))
        yield* Ref.set(errorRef, Option.none())
        yield* Ref.set(lastFetchRef, Option.some(new Date()))
        return data
      } catch (error) {
        const httpError = error instanceof HttpError ? error : new HttpError({
          status: 0,
          message: 'Unknown error',
          url
        })
        
        yield* Ref.set(errorRef, Option.some(httpError))
        
        // Return the last known good data if available
        const lastData = yield* dataRef.get
        return Option.match(lastData, {
          onNone: () => { throw httpError },
          onSome: data => data
        })
      }
    }).pipe(
      Effect.retry(
        Schedule.exponential(Duration.millis(100)).pipe(
          Schedule.compose(Schedule.recurs(3))
        )
      )
    )
  ).pipe(
    Stream.schedule(Schedule.fixed(interval))
  )
  
  const pollingSubscribable = Subscribable.make({
    get: Effect.gen(function* () {
      const data = yield* dataRef.get
      const error = yield* errorRef.get
      
      return Option.match(data, {
        onNone: () => Option.match(error, {
          onNone: () => { throw new HttpError({ status: 0, message: 'No data available', url }) },
          onSome: err => { throw err }
        }),
        onSome: d => d
      })
    }),
    
    changes: pollingStream
  })
  
  // Metadata subscribable
  const metadataRef = yield* Ref.make({
    url,
    interval: Duration.toMillis(interval),
    isPolling: true,
    errorCount: 0,
    successCount: 0
  })
  
  const metadataStream = pollingStream.pipe(
    Stream.tap(_ => Ref.update(metadataRef, m => ({ ...m, successCount: m.successCount + 1 }))),
    Stream.catchAll(_ => 
      Stream.fromEffect(
        Ref.update(metadataRef, m => ({ ...m, errorCount: m.errorCount + 1 }))
      )
    ),
    Stream.map(_ => metadataRef.get),
    Stream.flatMap(Stream.fromEffect)
  )
  
  const metadata = Subscribable.make({
    get: metadataRef.get,
    changes: metadataStream
  })
  
  return {
    data: pollingSubscribable,
    metadata,
    lastFetch: lastFetchRef as Ref.Ref<Option.Option<Date>>,
    lastError: errorRef as Ref.Ref<Option.Option<HttpError>>
  }
})

// Multi-endpoint API monitoring system
const makeApiMonitoringSystem = (endpoints: Array<{
  name: string
  url: string
  interval: Duration.Duration
}>) => Effect.gen(function* () {
  // Create polling subscribables for each endpoint
  const pollingServices = yield* Effect.all(
    endpoints.map(endpoint =>
      makePollingSubscribable(endpoint.url, endpoint.interval, mockHttpClient).pipe(
        Effect.map(service => [endpoint.name, service] as const)
      )
    )
  )
  
  const serviceMap = new Map(pollingServices)
  
  // Aggregated system health
  const systemHealthRef = yield* Ref.make({
    totalEndpoints: endpoints.length,
    healthyEndpoints: 0,
    totalRequests: 0,
    totalErrors: 0,
    overallStatus: 'unknown' as 'healthy' | 'degraded' | 'critical' | 'unknown',
    lastUpdate: new Date()
  })
  
  // Update system health when any service changes
  const healthUpdates = Stream.mergeAll(
    Array.from(serviceMap.values()).map(service => service.metadata.changes),
    { concurrency: 'unbounded' }
  ).pipe(
    Stream.tap(_ =>
      Effect.gen(function* () {
        let healthyEndpoints = 0
        let totalRequests = 0
        let totalErrors = 0
        
        for (const service of serviceMap.values()) {
          const metadata = yield* service.metadata.get
          const lastError = yield* service.lastError.get
          
          totalRequests += metadata.successCount
          totalErrors += metadata.errorCount
          
          if (Option.isNone(lastError) && metadata.successCount > 0) {
            healthyEndpoints++
          }
        }
        
        const healthPercent = (healthyEndpoints / endpoints.length) * 100
        const overallStatus = healthPercent >= 90 ? 'healthy'
                            : healthPercent >= 70 ? 'degraded'  
                            : 'critical'
        
        yield* Ref.set(systemHealthRef, {
          totalEndpoints: endpoints.length,
          healthyEndpoints,
          totalRequests,
          totalErrors,
          overallStatus,
          lastUpdate: new Date()
        })
      })
    ),
    Stream.map(_ => systemHealthRef.get),
    Stream.flatMap(Stream.fromEffect)
  )
  
  const systemHealth = Subscribable.make({
    get: systemHealthRef.get,
    changes: healthUpdates
  })
  
  // Create combined data subscribable
  const combinedDataRef = yield* Ref.make<Map<string, any>>(new Map())
  
  const dataUpdates = Stream.mergeAll(
    Array.from(serviceMap.entries()).map(([name, service]) =>
      service.data.changes.pipe(
        Stream.tap(data =>
          Ref.update(combinedDataRef, map => new Map(map).set(name, data))
        )
      )
    ),
    { concurrency: 'unbounded' }
  ).pipe(
    Stream.map(_ => combinedDataRef.get),
    Stream.flatMap(Stream.fromEffect)
  )
  
  const combinedData = Subscribable.make({
    get: combinedDataRef.get,
    changes: dataUpdates
  })
  
  return {
    services: serviceMap,
    systemHealth,
    combinedData
  }
})

// API monitoring application
const apiMonitoringApp = Effect.gen(function* () {
  console.log('=== HTTP Polling API Monitoring ===')
  
  // Configure multiple endpoints with different polling intervals
  const endpoints = [
    { name: 'weather', url: 'https://api.example.com/api/weather/current', interval: Duration.seconds(30) },
    { name: 'stock-AAPL', url: 'https://api.example.com/api/stock/AAPL', interval: Duration.seconds(5) },
    { name: 'stock-GOOGL', url: 'https://api.example.com/api/stock/GOOGL', interval: Duration.seconds(5) },
    { name: 'system-health', url: 'https://api.example.com/api/system/health', interval: Duration.seconds(10) }
  ]
  
  const monitor = yield* makeApiMonitoringSystem(endpoints)
  
  // Monitor system health
  const healthMonitor = monitor.systemHealth.changes.pipe(
    Stream.take(15),
    Stream.tap(health =>
      Effect.sync(() => {
        const statusIcon = health.overallStatus === 'healthy' ? '🟢'
                         : health.overallStatus === 'degraded' ? '🟡'  
                         : '🔴'
        
        console.log(
          `${statusIcon} System Health: ${health.overallStatus} | ` +
          `Healthy: ${health.healthyEndpoints}/${health.totalEndpoints} | ` +
          `Requests: ${health.totalRequests} | Errors: ${health.totalErrors}`
        )
      })
    ),
    Stream.runDrain
  )
  
  // Monitor individual services
  const serviceMonitors = Array.from(monitor.services.entries()).map(([name, service]) =>
    service.data.changes.pipe(
      Stream.take(8),
      Stream.tap(data =>
        Effect.gen(function* () {
          const metadata = yield* service.metadata.get
          const lastError = yield* service.lastError.get
          
          const status = Option.match(lastError, {
            onNone: () => '✅',
            onSome: _ => '❌'
          })
          
          console.log(`${status} ${name}: Updated (${metadata.successCount} success, ${metadata.errorCount} errors)`)
          
          // Show sample data for some services
          if (name === 'weather' && 'temperature' in data) {
            console.log(`  🌡️ ${(data as any).temperature.toFixed(1)}°C, ${(data as any).condition}`)
          } else if (name.startsWith('stock-') && 'price' in data) {
            const stock = data as any
            const changeIcon = stock.change >= 0 ? '📈' : '📉'
            console.log(`  ${changeIcon} $${stock.price.toFixed(2)} (${stock.change >= 0 ? '+' : ''}${stock.change.toFixed(2)})`)
          } else if (name === 'system-health' && 'status' in data) {
            const health = data as any
            console.log(`  💻 ${health.status} - CPU: ${health.cpu.toFixed(1)}%, Memory: ${health.memory.toFixed(1)}%`)
          }
        })
      ),
      Stream.runDrain
    )
  )
  
  // Start all monitors
  yield* Effect.fork(healthMonitor)
  yield* Effect.all(serviceMonitors.map(Effect.fork), { concurrency: 'unbounded' })
  
  // Let the system run for a while
  yield* Effect.sleep(Duration.seconds(45))
  
  // Show final statistics
  console.log('\n=== Final API Monitoring Report ===')
  
  const finalHealth = yield* monitor.systemHealth.get
  console.log(`Overall Status: ${finalHealth.overallStatus}`)
  console.log(`Healthy Endpoints: ${finalHealth.healthyEndpoints}/${finalHealth.totalEndpoints}`)
  console.log(`Total Requests: ${finalHealth.totalRequests}`)
  console.log(`Total Errors: ${finalHealth.totalErrors}`)
  
  console.log('\nEndpoint Details:')
  for (const [name, service] of monitor.services) {
    const metadata = yield* service.metadata.get
    const lastFetch = yield* service.lastFetch.get
    const lastError = yield* service.lastError.get
    
    const errorRate = metadata.successCount + metadata.errorCount > 0
      ? (metadata.errorCount / (metadata.successCount + metadata.errorCount) * 100)
      : 0
    
    console.log(`  ${name}:`)
    console.log(`    Success: ${metadata.successCount}, Errors: ${metadata.errorCount}`)
    console.log(`    Error Rate: ${errorRate.toFixed(1)}%`)
    console.log(`    Last Fetch: ${Option.match(lastFetch, {
      onNone: () => 'Never',
      onSome: date => date.toLocaleTimeString()
    })}`)
    
    Option.match(lastError, {
      onNone: () => {},
      onSome: error => console.log(`    Last Error: ${error.message}`)
    })
  }
})
```

## Conclusion

Subscribable provides a powerful foundation for reactive programming in Effect applications, combining the simplicity of current state access with the flexibility of change streams. It bridges the gap between imperative state management and reactive programming patterns while maintaining Effect's core principles of type safety, composability, and resource management.

Key benefits:
- **Reactive Programming Made Simple**: Combines current state access and change streams in a single, coherent interface
- **Automatic Resource Management**: Built-in cleanup and lifecycle management prevents memory leaks and resource exhaustion  
- **Type-Safe Composition**: Full TypeScript support with precise error handling and dependency tracking
- **Stream Integration**: Seamless integration with Effect's Stream module for sophisticated data processing pipelines

Subscribable excels in scenarios requiring real-time data processing, reactive UI systems, event-driven architectures, and complex state coordination where traditional callback-based approaches become unwieldy. Its integration with Effect's ecosystem makes it particularly powerful for building robust, scalable reactive applications with predictable resource management and error handling.