# BigInt: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem BigInt Solves

JavaScript's native `number` type is limited to safe integers between -(2⁵³ - 1) and 2⁵³ - 1. Beyond this range, precision is lost and calculations become unreliable, making it unsuitable for applications requiring arbitrary precision arithmetic.

```typescript
// Traditional approach - precision loss and unsafe operations
const largeNumber = 9007199254740991 // Number.MAX_SAFE_INTEGER
const unsafeAddition = largeNumber + 1 // 9007199254740992 (correct)
const unsafeLarger = largeNumber + 2 // 9007199254740992 (incorrect - should be 9007199254740993)

// Financial calculations with large amounts fail
const cryptoWei = 1000000000000000000 // 1 ETH in wei
const calculation = cryptoWei * 1.5 // 1500000000000000000 (loses precision)

// No built-in support for arbitrarily large integers
const factorial100 = 1
for (let i = 1; i <= 100; i++) {
  factorial100 *= i // Will overflow and become Infinity
}

// Type-unsafe operations
function isLargeInteger(value: any): boolean {
  return typeof value === 'number' && value > Number.MAX_SAFE_INTEGER
}
```

This approach leads to:
- **Precision Loss** - Numbers beyond safe integer range lose accuracy
- **Type Safety Issues** - No compile-time guarantees about integer operations  
- **Overflow Problems** - Large calculations result in Infinity or incorrect values
- **No Arbitrary Precision** - Cannot handle cryptographic keys, large financial amounts, or scientific calculations

### The BigInt Solution

Effect BigInt provides comprehensive arbitrary precision integer arithmetic with type safety, functional composition, and robust error handling.

```typescript
import { BigInt, Option, Effect } from "effect"

// Type-safe arbitrary precision arithmetic
const largeNumber = BigInt(9007199254740991)
const safeAddition = BigInt.sum(largeNumber, BigInt(2)) // 9007199254740993n (precise)

// Safe division with error handling
const divide = (a: bigint, b: bigint) =>
  BigInt.divide(a, b).pipe(
    Option.getOrElse(() => BigInt(0))
  )

// Composable mathematical operations
const calculateCompoundInterest = (principal: bigint, rate: bigint, periods: bigint) =>
  Effect.gen(function* () {
    const ratePlus1 = BigInt.sum(BigInt(100), rate)
    const power = yield* Effect.succeed(ratePlus1 ** periods)
    const numerator = BigInt.multiply(principal, power)
    return yield* Effect.fromOption(
      BigInt.divide(numerator, BigInt(100) ** periods),
      () => new Error("Division error in compound interest calculation")
    )
  })
```

### Key Concepts

**Type Guards**: `BigInt.isBigInt(value)` provides runtime type checking with TypeScript type narrowing

**Safe Division**: `BigInt.divide(a, b)` returns `Option<bigint>` to handle division by zero gracefully

**Mathematical Operations**: `BigInt.gcd`, `BigInt.lcm`, `BigInt.sqrt` for advanced mathematical computations

**Range Operations**: `BigInt.between`, `BigInt.clamp`, `BigInt.min`, `BigInt.max` for boundary checking

**Conversion Utilities**: `BigInt.fromString`, `BigInt.fromNumber`, `BigInt.toNumber` for safe type conversions

## Basic Usage Patterns

### Pattern 1: Type Validation and Conversion

```typescript
import { BigInt, Option } from "effect"

// Basic type checking with automatic type narrowing
function processBigIntInput(input: unknown): string {
  if (BigInt.isBigInt(input)) {
    // TypeScript now knows 'input' is a bigint
    return `Processing large integer: ${input.toString()}`
  }
  return "Invalid input: not a bigint"
}

// Safe conversion from strings
const parseUserInput = (input: string): Option.Option<bigint> =>
  BigInt.fromString(input)

// Safe conversion from numbers
const convertNumber = (num: number): Option.Option<bigint> =>
  BigInt.fromNumber(num)

// Safe conversion back to number (when within safe range)
const toSafeNumber = (big: bigint): Option.Option<number> =>
  BigInt.toNumber(big)

// Usage examples
console.log(parseUserInput("123456789012345678901234567890"))
// Output: Some(123456789012345678901234567890n)

console.log(convertNumber(Number.MAX_SAFE_INTEGER))
// Output: Some(9007199254740991n)

console.log(toSafeNumber(BigInt(1000)))
// Output: Some(1000)
```

### Pattern 2: Safe Mathematical Operations

```typescript
import { BigInt, Option, Effect } from "effect"

// Basic arithmetic with safe division
const calculate = (a: bigint, b: bigint, c: bigint) =>
  BigInt.divide(a, b).pipe(
    Option.map(result => BigInt.multiply(result, c)),
    Option.getOrElse(() => BigInt(0))
  )

// Chaining operations with error handling
const complexCalculation = (x: bigint, y: bigint, z: bigint) =>
  Effect.gen(function* () {
    const step1 = yield* Effect.fromOption(
      BigInt.divide(x, y),
      () => new Error("Division by zero in step 1")
    )
    const step2 = BigInt.sum(step1, z)
    const step3 = BigInt.multiply(step2, BigInt(2))
    return BigInt.abs(step3)
  }).pipe(
    Effect.catchAll(() => Effect.succeed(BigInt(0)))
  )

// Aggregation operations
const numbers = [BigInt(1), BigInt(2), BigInt(3), BigInt(4), BigInt(5)]
const total = BigInt.sumAll(numbers) // 15n
const product = BigInt.multiplyAll(numbers) // 120n
```

### Pattern 3: Range and Comparison Operations

```typescript
import { BigInt } from "effect"

// Boundary checking for large numbers
const isValidCryptoAmount = BigInt.between({ 
  minimum: BigInt(0), 
  maximum: BigInt("1000000000000000000000000") // 1 million ETH in wei
})

const isSmallAmount = BigInt.lessThan(BigInt(1000))
const isLargeAmount = BigInt.greaterThan(BigInt("1000000000000000000")) // 1 ETH in wei

// Clamping large values to acceptable ranges
const clampToMaxSupply = BigInt.clamp({ 
  minimum: BigInt(0), 
  maximum: BigInt("21000000000000000000000000") // 21M BTC in satoshis
})

// Finding extremes in large datasets
const cryptoBalances = [
  BigInt("1500000000000000000"), // 1.5 ETH
  BigInt("2300000000000000000"), // 2.3 ETH  
  BigInt("750000000000000000"),  // 0.75 ETH
  BigInt("5000000000000000000")  // 5 ETH
]

const maxBalance = cryptoBalances.reduce(BigInt.max)
const minBalance = cryptoBalances.reduce(BigInt.min)
```

## Real-World Examples

### Example 1: Cryptocurrency Wallet Manager

Building a wallet system that handles large integer amounts with precision and safety.

```typescript
import { BigInt, Option, Effect, Schema } from "effect"

interface CryptocurrencyError extends Error {
  readonly _tag: "CryptocurrencyError"
}

const CryptocurrencyError = (message: string): CryptocurrencyError => ({
  _tag: "CryptocurrencyError",
  name: "CryptocurrencyError",
  message
})

interface CryptoBalance {
  readonly symbol: string
  readonly balance: bigint
  readonly decimals: number
}

interface Transaction {
  readonly from: string
  readonly to: string
  readonly amount: bigint
  readonly gasPrice: bigint
  readonly gasLimit: bigint
  readonly symbol: string
}

interface TransactionCost {
  readonly amount: bigint
  readonly gasFee: bigint
  readonly total: bigint
  readonly symbol: string
}

// Helper for converting between different decimal representations
const convertToBaseUnit = (amount: string, decimals: number): Option.Option<bigint> =>
  BigInt.fromString(amount).pipe(
    Option.map(baseAmount => BigInt.multiply(baseAmount, BigInt(10) ** BigInt(decimals)))
  )

const convertFromBaseUnit = (amount: bigint, decimals: number): string => {
  const divisor = BigInt(10) ** BigInt(decimals)
  const quotient = BigInt.divide(amount, divisor)
  const remainder = amount % divisor
  
  return Option.match(quotient, {
    onNone: () => "0",
    onSome: (q) => {
      const remainderStr = remainder.toString().padStart(decimals, '0')
      return `${q.toString()}.${remainderStr}`
    }
  })
}

// Calculate transaction cost including gas fees
const calculateTransactionCost = (transaction: Transaction): Effect.Effect<TransactionCost, CryptocurrencyError> =>
  Effect.gen(function* () {
    // Validate transaction amount is positive
    if (!BigInt.greaterThan(BigInt(0))(transaction.amount)) {
      return yield* Effect.fail(CryptocurrencyError("Transaction amount must be positive"))
    }
    
    // Validate gas parameters
    if (!BigInt.greaterThan(BigInt(0))(transaction.gasPrice) || !BigInt.greaterThan(BigInt(0))(transaction.gasLimit)) {
      return yield* Effect.fail(CryptocurrencyError("Gas price and limit must be positive"))
    }

    // Calculate gas fee: gasPrice * gasLimit
    const gasFee = BigInt.multiply(transaction.gasPrice, transaction.gasLimit)
    
    // Total cost: amount + gas fee
    const total = BigInt.sum(transaction.amount, gasFee)

    return {
      amount: transaction.amount,
      gasFee,
      total,
      symbol: transaction.symbol
    }
  })

// Validate sufficient balance for transaction
const validateSufficientBalance = (
  balance: CryptoBalance,
  cost: TransactionCost
): Effect.Effect<void, CryptocurrencyError> =>
  Effect.gen(function* () {
    if (balance.symbol !== cost.symbol) {
      return yield* Effect.fail(CryptocurrencyError(`Currency mismatch: ${balance.symbol} vs ${cost.symbol}`))
    }
    
    if (BigInt.lessThan(balance.balance, cost.total)) {
      const balanceFormatted = convertFromBaseUnit(balance.balance, balance.decimals)
      const totalFormatted = convertFromBaseUnit(cost.total, balance.decimals)
      
      return yield* Effect.fail(
        CryptocurrencyError(`Insufficient balance: ${balanceFormatted} ${balance.symbol}, required: ${totalFormatted} ${balance.symbol}`)
      )
    }
  })

// Process a cryptocurrency transaction
export const processTransaction = (
  balance: CryptoBalance,
  transaction: Transaction
): Effect.Effect<{ balance: CryptoBalance, cost: TransactionCost }, CryptocurrencyError> =>
  Effect.gen(function* () {
    // Calculate transaction costs
    const cost = yield* calculateTransactionCost(transaction)
    
    // Validate sufficient balance
    yield* validateSufficientBalance(balance, cost)
    
    // Deduct amount from balance
    const newBalance = BigInt.subtract(balance.balance, cost.total)
    
    return {
      balance: { ...balance, balance: newBalance },
      cost
    }
  })

// Portfolio value calculation
export const calculatePortfolioValue = (balances: ReadonlyArray<CryptoBalance>): Effect.Effect<Map<string, string>, never> =>
  Effect.gen(function* () {
    const portfolioMap = new Map<string, string>()
    
    balances.forEach(balance => {
      const formattedBalance = convertFromBaseUnit(balance.balance, balance.decimals)
      portfolioMap.set(balance.symbol, formattedBalance)
    })
    
    return portfolioMap
  })

// Usage example
const ethBalance: CryptoBalance = {
  symbol: "ETH",
  balance: BigInt("5000000000000000000"), // 5 ETH in wei
  decimals: 18
}

const transaction: Transaction = {
  from: "0x123...",
  to: "0x456...",
  amount: BigInt("1000000000000000000"), // 1 ETH in wei
  gasPrice: BigInt("20000000000"), // 20 gwei
  gasLimit: BigInt("21000"), // Standard transfer
  symbol: "ETH"
}

// In practice, you would run this effect:
// const result = yield* processTransaction(ethBalance, transaction)
// Output: {
//   balance: { symbol: "ETH", balance: 3999580000000000000n, decimals: 18 },
//   cost: { amount: 1000000000000000000n, gasFee: 420000000000000n, total: 1000420000000000000n, symbol: "ETH" }
// }
```

### Example 2: High-Precision Scientific Calculator

Processing large scientific calculations with arbitrary precision requirements.

```typescript
import { BigInt, Option, Effect, Array as Arr } from "effect"

interface CalculationError extends Error {
  readonly _tag: "CalculationError"
}

const CalculationError = (message: string): CalculationError => ({
  _tag: "CalculationError",
  name: "CalculationError",
  message
})

interface ScientificResult {
  readonly value: bigint
  readonly operation: string
  readonly inputs: ReadonlyArray<bigint>
  readonly isExact: boolean
}

// Calculate factorial with arbitrary precision
const factorial = (n: bigint): Effect.Effect<ScientificResult, CalculationError> =>
  Effect.gen(function* () {
    if (BigInt.lessThan(n, BigInt(0))) {
      return yield* Effect.fail(CalculationError("Factorial is undefined for negative numbers"))
    }
    
    if (BigInt.lessThanOrEqualTo(n, BigInt(1))) {
      return {
        value: BigInt(1),
        operation: `factorial(${n})`,
        inputs: [n],
        isExact: true
      }
    }
    
    let result = BigInt(1)
    let current = BigInt(2)
    
    while (BigInt.lessThanOrEqualTo(current, n)) {
      result = BigInt.multiply(result, current)
      current = BigInt.increment(current)
    }
    
    return {
      value: result,
      operation: `factorial(${n})`,
      inputs: [n],
      isExact: true
    }
  })

// Calculate power with integer exponents
const power = (base: bigint, exponent: bigint): Effect.Effect<ScientificResult, CalculationError> =>
  Effect.gen(function* () {
    if (BigInt.lessThan(exponent, BigInt(0))) {
      return yield* Effect.fail(CalculationError("Negative exponents not supported for integer arithmetic"))
    }
    
    if (BigInt.equals(exponent, BigInt(0))) {
      return {
        value: BigInt(1),
        operation: `power(${base}, ${exponent})`,
        inputs: [base, exponent],
        isExact: true
      }
    }
    
    let result = BigInt(1)
    let currentBase = base
    let currentExp = exponent
    
    // Fast exponentiation by squaring
    while (BigInt.greaterThan(currentExp, BigInt(0))) {
      if (currentExp % BigInt(2) === BigInt(1)) {
        result = BigInt.multiply(result, currentBase)
      }
      currentBase = BigInt.multiply(currentBase, currentBase)
      currentExp = BigInt.divide(currentExp, BigInt(2)).pipe(
        Option.getOrElse(() => BigInt(0))
      )
    }
    
    return {
      value: result,
      operation: `power(${base}, ${exponent})`,
      inputs: [base, exponent],
      isExact: true
    }
  })

// Calculate Fibonacci numbers with arbitrary precision
const fibonacci = (n: bigint): Effect.Effect<ScientificResult, CalculationError> =>
  Effect.gen(function* () {
    if (BigInt.lessThan(n, BigInt(0))) {
      return yield* Effect.fail(CalculationError("Fibonacci is undefined for negative numbers"))
    }
    
    if (BigInt.lessThanOrEqualTo(n, BigInt(1))) {
      return {
        value: n,
        operation: `fibonacci(${n})`,
        inputs: [n],
        isExact: true
      }
    }
    
    let a = BigInt(0)
    let b = BigInt(1)
    let current = BigInt(2)
    
    while (BigInt.lessThanOrEqualTo(current, n)) {
      const temp = BigInt.sum(a, b)
      a = b
      b = temp
      current = BigInt.increment(current)
    }
    
    return {
      value: b,
      operation: `fibonacci(${n})`,
      inputs: [n],
      isExact: true
    }
  })

// Calculate greatest common divisor for multiple numbers
const gcdMultiple = (numbers: ReadonlyArray<bigint>): Effect.Effect<ScientificResult, CalculationError> =>
  Effect.gen(function* () {
    if (numbers.length === 0) {
      return yield* Effect.fail(CalculationError("Cannot calculate GCD of empty array"))
    }
    
    if (numbers.length === 1) {
      return {
        value: BigInt.abs(numbers[0]),
        operation: `gcd([${numbers.join(", ")}])`,
        inputs: numbers,
        isExact: true
      }
    }
    
    const result = numbers.reduce((acc, num) => BigInt.gcd(acc, num))
    
    return {
      value: BigInt.abs(result),
      operation: `gcd([${numbers.join(", ")}])`,
      inputs: numbers,
      isExact: true
    }
  })

// Calculate least common multiple for multiple numbers
const lcmMultiple = (numbers: ReadonlyArray<bigint>): Effect.Effect<ScientificResult, CalculationError> =>
  Effect.gen(function* () {
    if (numbers.length === 0) {
      return yield* Effect.fail(CalculationError("Cannot calculate LCM of empty array"))
    }
    
    if (numbers.some(n => BigInt.equals(n, BigInt(0)))) {
      return yield* Effect.fail(CalculationError("LCM is undefined when any number is zero"))
    }
    
    const result = numbers.reduce((acc, num) => BigInt.lcm(acc, num))
    
    return {
      value: BigInt.abs(result),
      operation: `lcm([${numbers.join(", ")}])`,
      inputs: numbers,
      isExact: true
    }
  })

// Integer square root (floor)
const integerSqrt = (n: bigint): Effect.Effect<ScientificResult, CalculationError> =>
  Effect.gen(function* () {
    const sqrtOption = BigInt.sqrt(n)
    
    return Option.match(sqrtOption, {
      onNone: () => Effect.fail(CalculationError("Square root is undefined for negative numbers")),
      onSome: (result) => Effect.succeed({
        value: result,
        operation: `sqrt(${n})`,
        inputs: [n],
        isExact: BigInt.multiply(result, result) === n
      })
    })
  }).pipe(Effect.flatten)

// Batch calculation processor
export const processBatchCalculations = (
  operations: ReadonlyArray<{ type: string, params: ReadonlyArray<bigint> }>
): Effect.Effect<ReadonlyArray<ScientificResult>, CalculationError> =>
  Effect.gen(function* () {
    const results = []
    
    for (const op of operations) {
      const result = yield* (function() {
        switch (op.type) {
          case "factorial":
            return factorial(op.params[0])
          case "power":
            return power(op.params[0], op.params[1])
          case "fibonacci":
            return fibonacci(op.params[0])
          case "gcd":
            return gcdMultiple(op.params)
          case "lcm":
            return lcmMultiple(op.params)
          case "sqrt":
            return integerSqrt(op.params[0])
          default:
            return Effect.fail(CalculationError(`Unknown operation: ${op.type}`))
        }
      })()
      
      results.push(result)
    }
    
    return results
  })

// Usage examples
const calculations = [
  { type: "factorial", params: [BigInt(50)] },
  { type: "power", params: [BigInt(2), BigInt(100)] },
  { type: "fibonacci", params: [BigInt(100)] },
  { type: "gcd", params: [BigInt(48), BigInt(18), BigInt(24)] },
  { type: "lcm", params: [BigInt(12), BigInt(18), BigInt(24)] },
  { type: "sqrt", params: [BigInt(1000000)] }
]

// In practice, you would run this effect:
// const results = yield* processBatchCalculations(calculations)
// Example output:
// [
//   { value: 30414093201713378043612608166064768844377641568960512000000000000n, operation: "factorial(50)", ... },
//   { value: 1267650600228229401496703205376n, operation: "power(2, 100)", ... },
//   { value: 354224848179261915075n, operation: "fibonacci(100)", ... },
//   { value: 6n, operation: "gcd([48, 18, 24])", ... },
//   { value: 144n, operation: "lcm([12, 18, 24])", ... },
//   { value: 1000n, operation: "sqrt(1000000)", ... }
// ]
```

### Example 3: High-Frequency Trading System

Building a trading system that handles large volume calculations with microsecond precision.

```typescript
import { BigInt, Option, Effect, Array as Arr } from "effect"

interface TradingError extends Error {
  readonly _tag: "TradingError"
}

const TradingError = (message: string): TradingError => ({
  _tag: "TradingError",
  name: "TradingError",
  message
})

interface MarketOrder {
  readonly id: string
  readonly symbol: string
  readonly side: "BUY" | "SELL"
  readonly quantity: bigint // in smallest units (e.g., satoshis, wei)
  readonly price: bigint // in smallest units  
  readonly timestamp: bigint // microseconds since epoch
}

interface ExecutedTrade {
  readonly buyOrderId: string
  readonly sellOrderId: string
  readonly symbol: string
  readonly quantity: bigint
  readonly price: bigint
  readonly timestamp: bigint
  readonly buyerFee: bigint
  readonly sellerFee: bigint
  readonly netAmount: bigint
}

interface OrderBook {
  readonly symbol: string
  readonly bids: ReadonlyArray<MarketOrder> // Buy orders, highest price first
  readonly asks: ReadonlyArray<MarketOrder> // Sell orders, lowest price first
  readonly lastPrice: Option.Option<bigint>
}

interface PortfolioPosition {
  readonly symbol: string
  readonly quantity: bigint
  readonly averagePrice: bigint
  readonly totalInvested: bigint
  readonly unrealizedPnL: bigint
}

// Fee calculation (in basis points, e.g., 25 = 0.25%)
const calculateTradingFee = (amount: bigint, feeBasisPoints: bigint): bigint => {
  const feeAmount = BigInt.multiply(amount, feeBasisPoints)
  return BigInt.divide(feeAmount, BigInt(10000)).pipe(
    Option.getOrElse(() => BigInt(0))
  )
}

// Match orders and execute trades
const matchOrders = (orderBook: OrderBook, feeRate: bigint): Effect.Effect<ReadonlyArray<ExecutedTrade>, TradingError> =>
  Effect.gen(function* () {
    const trades: ExecutedTrade[] = []
    let remainingBids = [...orderBook.bids]
    let remainingAsks = [...orderBook.asks]
    
    // Sort bids by price (descending) and asks by price (ascending)
    remainingBids.sort((a, b) => BigInt.greaterThan(a.price, b.price) ? -1 : 1)
    remainingAsks.sort((a, b) => BigInt.lessThan(a.price, b.price) ? -1 : 1)
    
    while (remainingBids.length > 0 && remainingAsks.length > 0) {
      const highestBid = remainingBids[0]
      const lowestAsk = remainingAsks[0]
      
      // Check if orders can match
      if (BigInt.lessThan(highestBid.price, lowestAsk.price)) {
        break // No more matches possible
      }
      
      // Determine trade quantity (minimum of both orders)
      const tradeQuantity = BigInt.min(highestBid.quantity, lowestAsk.quantity)
      const tradePrice = lowestAsk.price // Take the ask price
      const tradeValue = BigInt.multiply(tradeQuantity, tradePrice)
      
      // Calculate fees
      const buyerFee = calculateTradingFee(tradeValue, feeRate)
      const sellerFee = calculateTradingFee(tradeValue, feeRate)
      const netAmount = BigInt.subtract(tradeValue, BigInt.sum(buyerFee, sellerFee))
      
      // Create executed trade
      const trade: ExecutedTrade = {
        buyOrderId: highestBid.id,
        sellOrderId: lowestAsk.id,
        symbol: orderBook.symbol,
        quantity: tradeQuantity,
        price: tradePrice,
        timestamp: BigInt(Date.now() * 1000), // Convert to microseconds
        buyerFee,
        sellerFee,
        netAmount
      }
      
      trades.push(trade)
      
      // Update remaining quantities
      const newBidQuantity = BigInt.subtract(highestBid.quantity, tradeQuantity)
      const newAskQuantity = BigInt.subtract(lowestAsk.quantity, tradeQuantity)
      
      // Remove fully filled orders or update quantities
      if (BigInt.equals(newBidQuantity, BigInt(0))) {
        remainingBids.shift()
      } else {
        remainingBids[0] = { ...highestBid, quantity: newBidQuantity }
      }
      
      if (BigInt.equals(newAskQuantity, BigInt(0))) {
        remainingAsks.shift()
      } else {
        remainingAsks[0] = { ...lowestAsk, quantity: newAskQuantity }
      }
    }
    
    return trades
  })

// Calculate portfolio position with average price
const calculatePosition = (
  currentPosition: Option.Option<PortfolioPosition>,
  trade: ExecutedTrade,
  isAddingToPosition: boolean
): Effect.Effect<PortfolioPosition, TradingError> =>
  Effect.gen(function* () {
    return Option.match(currentPosition, {
      onNone: () => ({
        symbol: trade.symbol,
        quantity: trade.quantity,
        averagePrice: trade.price,
        totalInvested: BigInt.multiply(trade.quantity, trade.price),
        unrealizedPnL: BigInt(0)
      }),
      onSome: (position) => {
        if (isAddingToPosition) {
          // Adding to position: calculate new average price
          const additionalInvestment = BigInt.multiply(trade.quantity, trade.price)
          const newTotalInvested = BigInt.sum(position.totalInvested, additionalInvestment)
          const newQuantity = BigInt.sum(position.quantity, trade.quantity)
          
          const newAveragePrice = BigInt.divide(newTotalInvested, newQuantity).pipe(
            Option.getOrElse(() => position.averagePrice)
          )
          
          return {
            symbol: trade.symbol,
            quantity: newQuantity,
            averagePrice: newAveragePrice,
            totalInvested: newTotalInvested,
            unrealizedPnL: BigInt(0) // Reset PnL calculation
          }
        } else {
          // Reducing position: maintain average price
          const newQuantity = BigInt.subtract(position.quantity, trade.quantity)
          const reducedInvestment = BigInt.multiply(trade.quantity, position.averagePrice)
          const newTotalInvested = BigInt.subtract(position.totalInvested, reducedInvestment)
          
          return {
            symbol: trade.symbol,
            quantity: newQuantity,
            averagePrice: position.averagePrice,
            totalInvested: newTotalInvested,
            unrealizedPnL: BigInt(0)
          }
        }
      }
    })
  })

// Calculate unrealized PnL for a position
const calculateUnrealizedPnL = (
  position: PortfolioPosition,
  currentPrice: bigint
): bigint => {
  const currentValue = BigInt.multiply(position.quantity, currentPrice)
  const costBasis = BigInt.multiply(position.quantity, position.averagePrice)
  return BigInt.subtract(currentValue, costBasis)
}

// Process high-frequency trading batch
export const processHFTBatch = (
  orderBooks: ReadonlyArray<OrderBook>,
  currentPrices: Map<string, bigint>,
  feeRate: bigint
): Effect.Effect<{
  trades: ReadonlyArray<ExecutedTrade>
  totalVolume: Map<string, bigint>
  totalFees: bigint
}, TradingError> =>
  Effect.gen(function* () {
    const allTrades: ExecutedTrade[] = []
    const volumeMap = new Map<string, bigint>()
    let totalFees = BigInt(0)
    
    // Process each order book
    for (const orderBook of orderBooks) {
      const trades = yield* matchOrders(orderBook, feeRate)
      allTrades.push(...trades)
      
      // Aggregate volume and fees
      const symbolVolume = trades.reduce(
        (acc, trade) => BigInt.sum(acc, BigInt.multiply(trade.quantity, trade.price)),
        BigInt(0)
      )
      
      volumeMap.set(orderBook.symbol, symbolVolume)
      
      const symbolFees = trades.reduce(
        (acc, trade) => BigInt.sum(acc, BigInt.sum(trade.buyerFee, trade.sellerFee)),
        BigInt(0)
      )
      
      totalFees = BigInt.sum(totalFees, symbolFees)
    }
    
    return {
      trades: allTrades,
      totalVolume: volumeMap,
      totalFees
    }
  })

// Market making spread calculation
export const calculateOptimalSpread = (
  orderBook: OrderBook,
  volatilityFactor: bigint, // in basis points
  minSpreadBps: bigint // minimum spread in basis points
): Effect.Effect<{ bidPrice: bigint, askPrice: bigint }, TradingError> =>
  Effect.gen(function* () {
    const midPrice = yield* Option.match(orderBook.lastPrice, {
      onNone: () => Effect.fail(TradingError("No last price available for spread calculation")),
      onSome: Effect.succeed
    })
    
    // Calculate spread based on volatility (minimum of minSpreadBps)
    const volatilitySpread = BigInt.multiply(midPrice, volatilityFactor)
    const minSpread = BigInt.multiply(midPrice, minSpreadBps)
    const spread = BigInt.max(
      BigInt.divide(volatilitySpread, BigInt(10000)).pipe(Option.getOrElse(() => minSpread)),
      BigInt.divide(minSpread, BigInt(10000)).pipe(Option.getOrElse(() => BigInt(1)))
    )
    
    const halfSpread = BigInt.divide(spread, BigInt(2)).pipe(
      Option.getOrElse(() => BigInt(1))
    )
    
    return {
      bidPrice: BigInt.subtract(midPrice, halfSpread),
      askPrice: BigInt.sum(midPrice, halfSpread)
    }
  })

// Usage example
const btcOrderBook: OrderBook = {
  symbol: "BTC",
  bids: [
    { id: "bid1", symbol: "BTC", side: "BUY", quantity: BigInt(100000000), price: BigInt(4350000), timestamp: BigInt(Date.now() * 1000) }, // 1 BTC at $43,500
    { id: "bid2", symbol: "BTC", side: "BUY", quantity: BigInt(50000000), price: BigInt(4349500), timestamp: BigInt(Date.now() * 1000) }  // 0.5 BTC at $43,495
  ],
  asks: [
    { id: "ask1", symbol: "BTC", side: "SELL", quantity: BigInt(75000000), price: BigInt(4350500), timestamp: BigInt(Date.now() * 1000) }, // 0.75 BTC at $43,505
    { id: "ask2", symbol: "BTC", side: "SELL", quantity: BigInt(200000000), price: BigInt(4351000), timestamp: BigInt(Date.now() * 1000) } // 2 BTC at $43,510
  ],
  lastPrice: Option.some(BigInt(4350250))
}

const feeRate = BigInt(25) // 0.25% fee

// In practice, you would run this effect:
// const trades = yield* matchOrders(btcOrderBook, feeRate)
// const spread = yield* calculateOptimalSpread(btcOrderBook, BigInt(50), BigInt(5))
```

## Advanced Features Deep Dive

### Feature 1: Mathematical Operations and Number Theory

Effect BigInt provides advanced mathematical functions for number theory and cryptographic applications.

#### Basic Mathematical Functions

```typescript
import { BigInt, Option } from "effect"

// Greatest Common Divisor and Least Common Multiple
const calculateGcdLcm = (a: bigint, b: bigint) => ({
  gcd: BigInt.gcd(a, b),
  lcm: BigInt.lcm(a, b)
})

// Integer square root with remainder
const integerSqrtWithRemainder = (n: bigint): Option.Option<{ root: bigint, remainder: bigint }> =>
  BigInt.sqrt(n).pipe(
    Option.map(root => ({
      root,
      remainder: BigInt.subtract(n, BigInt.multiply(root, root))
    }))
  )

// Modular arithmetic operations
const modularArithmetic = (a: bigint, b: bigint, modulus: bigint) => ({
  addition: (BigInt.sum(a, b)) % modulus,
  subtraction: (BigInt.subtract(a, b) + modulus) % modulus,
  multiplication: (BigInt.multiply(a, b)) % modulus
})

console.log(calculateGcdLcm(BigInt(48), BigInt(18)))
// Output: { gcd: 6n, lcm: 144n }

console.log(integerSqrtWithRemainder(BigInt(50)))
// Output: Some({ root: 7n, remainder: 1n })
```

#### Real-World Mathematical Example

```typescript
// RSA Key Generation Helper (simplified)
interface RSAKeyParams {
  readonly p: bigint // First prime
  readonly q: bigint // Second prime
  readonly n: bigint // Modulus (p * q)
  readonly phi: bigint // Euler's totient (p-1)(q-1)
  readonly e: bigint // Public exponent
  readonly d: bigint // Private exponent
}

const generateRSAParams = (p: bigint, q: bigint, e: bigint): Effect.Effect<RSAKeyParams, Error> =>
  Effect.gen(function* () {
    // Validate primes are different
    if (BigInt.equals(p, q)) {
      return yield* Effect.fail(new Error("Primes p and q must be different"))
    }
    
    // Calculate modulus n = p * q
    const n = BigInt.multiply(p, q)
    
    // Calculate Euler's totient φ(n) = (p-1)(q-1)
    const phi = BigInt.multiply(
      BigInt.subtract(p, BigInt(1)),
      BigInt.subtract(q, BigInt(1))
    )
    
    // Verify gcd(e, φ(n)) = 1
    const gcdResult = BigInt.gcd(e, phi)
    if (!BigInt.equals(gcdResult, BigInt(1))) {
      return yield* Effect.fail(new Error("e and φ(n) must be coprime"))
    }
    
    // Calculate private exponent d using extended Euclidean algorithm
    // This is a simplified version - real implementation would use modular inverse
    const d = yield* calculateModularInverse(e, phi)
    
    return { p, q, n, phi, e, d }
  })

// Simplified modular inverse calculation
const calculateModularInverse = (a: bigint, m: bigint): Effect.Effect<bigint, Error> =>
  Effect.gen(function* () {
    // Extended Euclidean Algorithm implementation
    let oldR = a
    let r = m
    let oldS = BigInt(1)
    let s = BigInt(0)
    
    while (!BigInt.equals(r, BigInt(0))) {
      const quotient = BigInt.divide(oldR, r).pipe(
        Option.getOrElse(() => BigInt(0))
      )
      
      const newR = BigInt.subtract(oldR, BigInt.multiply(quotient, r))
      oldR = r
      r = newR
      
      const newS = BigInt.subtract(oldS, BigInt.multiply(quotient, s))
      oldS = s
      s = newS
    }
    
    if (!BigInt.equals(oldR, BigInt(1))) {
      return yield* Effect.fail(new Error("Modular inverse does not exist"))
    }
    
    // Ensure positive result
    const result = oldS < BigInt(0) 
      ? BigInt.sum(oldS, m) 
      : oldS
    
    return result
  })
```

#### Advanced Mathematical: Prime Number Operations

```typescript
// Miller-Rabin primality test (simplified)
const isProbablyPrime = (n: bigint, rounds: number = 10): Effect.Effect<boolean, never> =>
  Effect.gen(function* () {
    if (BigInt.lessThanOrEqualTo(n, BigInt(1))) return false
    if (BigInt.equals(n, BigInt(2)) || BigInt.equals(n, BigInt(3))) return true
    if (n % BigInt(2) === BigInt(0)) return false
    
    // Write n-1 as 2^r * d
    let r = BigInt(0)
    let d = BigInt.subtract(n, BigInt(1))
    
    while (d % BigInt(2) === BigInt(0)) {
      r = BigInt.increment(r)
      d = BigInt.divide(d, BigInt(2)).pipe(
        Option.getOrElse(() => d)
      )
    }
    
    // Perform rounds of testing
    for (let i = 0; i < rounds; i++) {
      // Generate random witness (simplified - would use crypto random)
      const a = BigInt(2) + BigInt(Math.floor(Math.random() * 100)) % BigInt.subtract(n, BigInt(3))
      
      // Perform Miller-Rabin test
      let x = modularExponentiation(a, d, n)
      
      if (BigInt.equals(x, BigInt(1)) || BigInt.equals(x, BigInt.subtract(n, BigInt(1)))) {
        continue
      }
      
      let continueTest = false
      for (let j = 0; j < Number(r) - 1; j++) {
        x = modularExponentiation(x, BigInt(2), n)
        if (BigInt.equals(x, BigInt.subtract(n, BigInt(1)))) {
          continueTest = true
          break
        }
      }
      
      if (!continueTest) return false
    }
    
    return true
  })

// Modular exponentiation: (base^exp) mod modulus
const modularExponentiation = (base: bigint, exponent: bigint, modulus: bigint): bigint => {
  let result = BigInt(1)
  base = base % modulus
  
  while (BigInt.greaterThan(exponent, BigInt(0))) {
    if (exponent % BigInt(2) === BigInt(1)) {
      result = (BigInt.multiply(result, base)) % modulus
    }
    exponent = BigInt.divide(exponent, BigInt(2)).pipe(
      Option.getOrElse(() => BigInt(0))
    )
    base = (BigInt.multiply(base, base)) % modulus
  }
  
  return result
}
```

### Feature 2: Safe Conversion and Interoperability

Handle conversions between BigInt and other numeric types safely.

#### Basic Conversion Operations

```typescript
import { BigInt, Option, Effect } from "effect"

// Safe conversions with validation
const ConversionHelpers = {
  // Convert string to BigInt with validation
  fromStringWithValidation: (str: string, radix: number = 10): Option.Option<bigint> => {
    if (radix !== 10) {
      // Handle other radixes manually since BigInt() only supports base 10 from strings
      return Option.none()
    }
    return BigInt.fromString(str.trim())
  },
  
  // Convert number to BigInt with range checking
  fromNumberSafe: (num: number): Option.Option<bigint> => {
    if (!Number.isInteger(num)) {
      return Option.none()
    }
    return BigInt.fromNumber(num)
  },
  
  // Convert BigInt to number with overflow checking
  toNumberSafe: (big: bigint): Option.Option<number> =>
    BigInt.toNumber(big),
  
  // Convert BigInt to string with formatting options
  toStringFormatted: (big: bigint, options?: { 
    thousands?: string
    prefix?: string 
  }): string => {
    const str = big.toString()
    const { thousands = ',', prefix = '' } = options || {}
    
    if (thousands) {
      const parts = []
      let remaining = str.replace(/^-/, '')
      const isNegative = str.startsWith('-')
      
      while (remaining.length > 3) {
        parts.unshift(remaining.slice(-3))
        remaining = remaining.slice(0, -3)
      }
      
      if (remaining) {
        parts.unshift(remaining)
      }
      
      return `${isNegative ? '-' : ''}${prefix}${parts.join(thousands)}`
    }
    
    return `${prefix}${str}`
  }
}

// Usage examples
console.log(ConversionHelpers.fromStringWithValidation("123456789012345678901234567890"))
// Output: Some(123456789012345678901234567890n)

console.log(ConversionHelpers.toStringFormatted(BigInt("1234567890123456789"), { thousands: ',', prefix: '$' }))
// Output: $1,234,567,890,123,456,789
```

#### Real-World Conversion Example

```typescript
// Financial amount conversion system
interface CurrencyAmount {
  readonly amount: bigint
  readonly currency: string
  readonly decimals: number
}

interface ConversionRate {
  readonly fromCurrency: string
  readonly toCurrency: string
  readonly rate: bigint // Rate in smallest units (e.g., rate * 10^18 for 18 decimals)
  readonly rateDecimals: number
}

const CurrencyConverter = {
  // Convert human-readable amount to internal representation
  fromDecimal: (
    amount: string, 
    currency: string, 
    decimals: number
  ): Effect.Effect<CurrencyAmount, Error> =>
    Effect.gen(function* () {
      const parts = amount.split('.')
      const wholePart = parts[0] || '0'
      const fractionalPart = (parts[1] || '').padEnd(decimals, '0').slice(0, decimals)
      
      const wholeAmount = yield* Effect.fromOption(
        BigInt.fromString(wholePart),
        () => new Error(`Invalid whole part: ${wholePart}`)
      )
      
      const fractionalAmount = yield* Effect.fromOption(
        BigInt.fromString(fractionalPart),
        () => new Error(`Invalid fractional part: ${fractionalPart}`)
      )
      
      const multiplier = BigInt(10) ** BigInt(decimals)
      const totalAmount = BigInt.sum(
        BigInt.multiply(wholeAmount, multiplier),
        fractionalAmount
      )
      
      return { amount: totalAmount, currency, decimals }
    }),
  
  // Convert internal representation to human-readable
  toDecimal: (currencyAmount: CurrencyAmount): string => {
    const { amount, decimals } = currencyAmount
    const divisor = BigInt(10) ** BigInt(decimals)
    const wholePart = BigInt.divide(amount, divisor).pipe(
      Option.getOrElse(() => BigInt(0))
    )
    const fractionalPart = amount % divisor
    
    const fractionalStr = fractionalPart.toString().padStart(decimals, '0')
    return `${wholePart.toString()}.${fractionalStr}`
  },
  
  // Convert between currencies using exchange rate
  convert: (
    amount: CurrencyAmount,
    rate: ConversionRate
  ): Effect.Effect<CurrencyAmount, Error> =>
    Effect.gen(function* () {
      if (amount.currency !== rate.fromCurrency) {
        return yield* Effect.fail(new Error(`Currency mismatch: ${amount.currency} vs ${rate.fromCurrency}`))
      }
      
      // Normalize to same decimal places for calculation
      const rateMultiplier = BigInt(10) ** BigInt(rate.rateDecimals)
      const targetDecimals = Math.max(amount.decimals, rate.rateDecimals)
      const targetMultiplier = BigInt(10) ** BigInt(targetDecimals)
      
      // Scale amounts to target decimals
      const scaledAmount = amount.decimals < targetDecimals 
        ? BigInt.multiply(amount.amount, BigInt(10) ** BigInt(targetDecimals - amount.decimals))
        : amount.amount
      
      const scaledRate = rate.rateDecimals < targetDecimals
        ? BigInt.multiply(rate.rate, BigInt(10) ** BigInt(targetDecimals - rate.rateDecimals))
        : rate.rate
      
      // Perform conversion: (amount * rate) / rateMultiplier
      const convertedAmount = BigInt.multiply(scaledAmount, scaledRate)
      const finalAmount = BigInt.divide(convertedAmount, rateMultiplier).pipe(
        Option.getOrElse(() => BigInt(0))
      )
      
      return {
        amount: finalAmount,
        currency: rate.toCurrency,
        decimals: targetDecimals
      }
    }),
  
  // Batch conversion for multiple amounts
  convertBatch: (
    amounts: ReadonlyArray<CurrencyAmount>,
    rates: Map<string, ConversionRate>,
    targetCurrency: string
  ): Effect.Effect<ReadonlyArray<CurrencyAmount>, Error> =>
    Effect.gen(function* () {
      const results = []
      
      for (const amount of amounts) {
        if (amount.currency === targetCurrency) {
          results.push(amount)
          continue
        }
        
        const rateKey = `${amount.currency}_${targetCurrency}`
        const rate = rates.get(rateKey)
        
        if (!rate) {
          return yield* Effect.fail(new Error(`No exchange rate found for ${amount.currency} to ${targetCurrency}`))
        }
        
        const converted = yield* CurrencyConverter.convert(amount, rate)
        results.push(converted)
      }
      
      return results
    })
}

// Usage example
const usdAmount = Effect.runSync(CurrencyConverter.fromDecimal("1234.56", "USD", 6))
// Output: { amount: 1234560000n, currency: "USD", decimals: 6 }

const readable = CurrencyConverter.toDecimal(usdAmount)
// Output: "1234.560000"
```

### Feature 3: Aggregation and Collection Operations

Efficiently process large collections of BigInt values.

#### Basic Aggregation Operations

```typescript
import { BigInt, Array as Arr, Option } from "effect"

// Statistical aggregation functions
const BigIntStats = {
  // Sum with overflow detection
  sumSafe: (values: ReadonlyArray<bigint>): bigint =>
    BigInt.sumAll(values),
  
  // Product with zero short-circuit
  productSafe: (values: ReadonlyArray<bigint>): bigint =>
    BigInt.multiplyAll(values),
  
  // Average (returns Option due to potential division issues)
  average: (values: ReadonlyArray<bigint>): Option.Option<bigint> =>
    values.length === 0
      ? Option.none()
      : BigInt.divide(BigInt.sumAll(values), BigInt(values.length)),
  
  // Median calculation
  median: (values: ReadonlyArray<bigint>): Option.Option<bigint> => {
    if (values.length === 0) return Option.none()
    
    const sorted = Arr.sort(values, BigInt.Order)
    const middle = Math.floor(sorted.length / 2)
    
    if (sorted.length % 2 === 0) {
      const sum = BigInt.sum(sorted[middle - 1], sorted[middle])
      return BigInt.divide(sum, BigInt(2))
    }
    
    return Option.some(sorted[middle])
  },
  
  // Mode (most frequent values)
  mode: (values: ReadonlyArray<bigint>): ReadonlyArray<bigint> => {
    if (values.length === 0) return []
    
    const frequency = new Map<string, { value: bigint, count: number }>()
    
    values.forEach(value => {
      const key = value.toString()
      const existing = frequency.get(key)
      frequency.set(key, {
        value,
        count: existing ? existing.count + 1 : 1
      })
    })
    
    const maxCount = Math.max(...Array.from(frequency.values()).map(f => f.count))
    return Array.from(frequency.values())
      .filter(f => f.count === maxCount)
      .map(f => f.value)
  },
  
  // Range calculation
  range: (values: ReadonlyArray<bigint>): Option.Option<bigint> => {
    if (values.length === 0) return Option.none()
    
    const min = values.reduce(BigInt.min)
    const max = values.reduce(BigInt.max)
    return Option.some(BigInt.subtract(max, min))
  },
  
  // Variance calculation (simplified integer version)
  variance: (values: ReadonlyArray<bigint>): Option.Option<bigint> =>
    BigIntStats.average(values).pipe(
      Option.flatMap(mean => {
        const squaredDifferences = values.map(value => {
          const diff = BigInt.subtract(value, mean)
          return BigInt.multiply(diff, diff)
        })
        return BigIntStats.average(squaredDifferences)
      })
    ),
  
  // Percentile calculation
  percentile: (values: ReadonlyArray<bigint>, p: number): Option.Option<bigint> => {
    if (values.length === 0 || p < 0 || p > 100) return Option.none()
    
    const sorted = Arr.sort(values, BigInt.Order)
    const index = Math.floor((p / 100) * (sorted.length - 1))
    return Option.some(sorted[Math.max(0, Math.min(index, sorted.length - 1))])
  }
}

// Usage example
const largeNumbers = [
  BigInt("123456789012345678901234567890"),
  BigInt("987654321098765432109876543210"),
  BigInt("555555555555555555555555555555"),
  BigInt("111111111111111111111111111111"),
  BigInt("999999999999999999999999999999")
]

const stats = {
  sum: BigIntStats.sumSafe(largeNumbers),
  average: BigIntStats.average(largeNumbers),
  median: BigIntStats.median(largeNumbers),
  range: BigIntStats.range(largeNumbers)
}

console.log("Sum:", stats.sum.toString())
console.log("Average:", Option.getOrElse(stats.average, () => BigInt(0)).toString())
```

## Practical Patterns & Best Practices

### Pattern 1: Type-Safe BigInt Operations Pipeline

Create reusable and composable BigInt operation chains.

```typescript
import { BigInt, Option, Effect, pipe } from "effect"

// BigInt operation result type
interface BigIntResult<T> {
  readonly value: T
  readonly isValid: boolean
  readonly errors: ReadonlyArray<string>
}

// Pipeline builder for BigInt operations
const createBigIntPipeline = () => {
  const operations: Array<(value: bigint) => Effect.Effect<bigint, Error>> = []
  
  const addOperation = (op: (value: bigint) => Effect.Effect<bigint, Error>) => {
    operations.push(op)
    return api
  }
  
  const validate = (predicate: (value: bigint) => boolean, message: string) =>
    addOperation((value) =>
      predicate(value)
        ? Effect.succeed(value)
        : Effect.fail(new Error(message))
    )
  
  const transform = (fn: (value: bigint) => bigint) =>
    addOperation((value) => Effect.succeed(fn(value)))
  
  const divideBy = (divisor: bigint) =>
    addOperation((value) =>
      Effect.fromOption(
        BigInt.divide(value, divisor),
        () => new Error(`Division by zero: ${value} / ${divisor}`)
      )
    )
  
  const multiplyBy = (multiplier: bigint) =>
    addOperation((value) => Effect.succeed(BigInt.multiply(value, multiplier)))
  
  const addValue = (addend: bigint) =>
    addOperation((value) => Effect.succeed(BigInt.sum(value, addend)))
  
  const subtractValue = (subtrahend: bigint) =>
    addOperation((value) => Effect.succeed(BigInt.subtract(value, subtrahend)))
  
  const clampToRange = (min: bigint, max: bigint) =>
    addOperation((value) => 
      Effect.succeed(BigInt.clamp({ minimum: min, maximum: max })(value))
    )
  
  const abs = () =>
    addOperation((value) => Effect.succeed(BigInt.abs(value)))
  
  const pow = (exponent: bigint) =>
    addOperation((value) => {
      if (BigInt.lessThan(exponent, BigInt(0))) {
        return Effect.fail(new Error("Negative exponents not supported"))
      }
      
      // Simple exponentiation (could be optimized)
      let result = BigInt(1)
      let exp = exponent
      let base = value
      
      while (BigInt.greaterThan(exp, BigInt(0))) {
        if (exp % BigInt(2) === BigInt(1)) {
          result = BigInt.multiply(result, base)
        }
        base = BigInt.multiply(base, base)
        exp = BigInt.divide(exp, BigInt(2)).pipe(
          Option.getOrElse(() => BigInt(0))
        )
      }
      
      return Effect.succeed(result)
    })
  
  const execute = (initialValue: bigint): Effect.Effect<bigint, Error> =>
    operations.reduce(
      (acc, operation) => Effect.flatMap(acc, operation),
      Effect.succeed(initialValue)
    )
  
  const executeWithValidation = (initialValue: bigint): BigIntResult<bigint> =>
    Effect.match(execute(initialValue), {
      onFailure: (error) => ({
        value: BigInt(0),
        isValid: false,
        errors: [error.message]
      }),
      onSuccess: (value) => ({
        value,
        isValid: true,
        errors: []
      })
    }).pipe(Effect.runSync)
  
  const api = {
    validate,
    transform,
    divideBy,
    multiplyBy,
    addValue,
    subtractValue,
    clampToRange,
    abs,
    pow,
    execute,
    executeWithValidation
  }
  
  return api
}

// Usage examples
const pipeline1 = createBigIntPipeline()
  .validate(
    (v) => BigInt.greaterThan(v, BigInt(0)),
    "Value must be positive"
  )
  .multiplyBy(BigInt(2))
  .divideBy(BigInt(3))
  .clampToRange(BigInt(0), BigInt(1000))

const result1 = pipeline1.executeWithValidation(BigInt(150))
// Result: { value: 100n, isValid: true, errors: [] }

const pipeline2 = createBigIntPipeline()
  .validate(
    (v) => BigInt.between({ minimum: BigInt(1), maximum: BigInt(10) })(v),
    "Value must be between 1 and 10"
  )
  .pow(BigInt(3))
  .subtractValue(BigInt(100))
  .abs()

const result2 = pipeline2.executeWithValidation(BigInt(5))
// Result: { value: 25n, isValid: true, errors: [] } (5^3 - 100 = 125 - 100 = 25, then abs)
```

### Pattern 2: BigInt-Based Fixed-Point Arithmetic

Implement precise decimal arithmetic using BigInt for financial calculations.

```typescript
import { BigInt, Option, pipe } from "effect"

// Fixed-point decimal representation using BigInt
class FixedPointDecimal {
  constructor(
    private readonly value: bigint,
    private readonly scale: number = 18 // Default 18 decimal places
  ) {}
  
  static from(value: string | number | bigint, scale: number = 18): FixedPointDecimal {
    if (typeof value === 'string') {
      const parts = value.split('.')
      const wholePart = parts[0] || '0'
      const fractionalPart = (parts[1] || '').padEnd(scale, '0').slice(0, scale)
      
      const wholeValue = BigInt(wholePart)
      const fractionalValue = BigInt(fractionalPart)
      const multiplier = BigInt(10) ** BigInt(scale)
      
      const totalValue = BigInt.sum(
        BigInt.multiply(wholeValue, multiplier),
        fractionalValue
      )
      
      return new FixedPointDecimal(totalValue, scale)
    }
    
    if (typeof value === 'number') {
      const stringValue = value.toFixed(scale)
      return FixedPointDecimal.from(stringValue, scale)
    }
    
    return new FixedPointDecimal(value, scale)
  }
  
  add(other: FixedPointDecimal): FixedPointDecimal {
    const [scaledThis, scaledOther] = this.normalizeScale(other)
    const result = BigInt.sum(scaledThis.value, scaledOther.value)
    return new FixedPointDecimal(result, Math.max(this.scale, other.scale))
  }
  
  subtract(other: FixedPointDecimal): FixedPointDecimal {
    const [scaledThis, scaledOther] = this.normalizeScale(other)
    const result = BigInt.subtract(scaledThis.value, scaledOther.value)
    return new FixedPointDecimal(result, Math.max(this.scale, other.scale))
  }
  
  multiply(other: FixedPointDecimal): FixedPointDecimal {
    const result = BigInt.multiply(this.value, other.value)
    const newScale = this.scale + other.scale
    
    // Adjust scale back to reasonable level
    const targetScale = Math.max(this.scale, other.scale)
    const scaleDiff = newScale - targetScale
    
    if (scaleDiff > 0) {
      const divisor = BigInt(10) ** BigInt(scaleDiff)
      const adjustedResult = BigInt.divide(result, divisor).pipe(
        Option.getOrElse(() => BigInt(0))
      )
      return new FixedPointDecimal(adjustedResult, targetScale)
    }
    
    return new FixedPointDecimal(result, newScale)
  }
  
  divide(other: FixedPointDecimal): Option.Option<FixedPointDecimal> {
    if (BigInt.equals(other.value, BigInt(0))) {
      return Option.none()
    }
    
    // Scale up dividend to maintain precision
    const targetScale = Math.max(this.scale, other.scale)
    const scaleUpFactor = BigInt(10) ** BigInt(targetScale)
    const scaledDividend = BigInt.multiply(this.value, scaleUpFactor)
    
    return BigInt.divide(scaledDividend, other.value).pipe(
      Option.map(result => new FixedPointDecimal(result, targetScale))
    )
  }
  
  percentage(percent: FixedPointDecimal): FixedPointDecimal {
    const hundred = FixedPointDecimal.from("100", this.scale)
    return Option.getOrElse(
      percent.divide(hundred).pipe(
        Option.map(rate => this.multiply(rate))
      ),
      () => FixedPointDecimal.from("0", this.scale)
    )
  }
  
  round(decimals: number = 2): FixedPointDecimal {
    if (decimals >= this.scale) {
      return this
    }
    
    const factor = BigInt(10) ** BigInt(this.scale - decimals)
    const rounded = BigInt.divide(BigInt.sum(this.value, BigInt.divide(factor, BigInt(2)).pipe(Option.getOrElse(() => BigInt(0)))), factor).pipe(
      Option.getOrElse(() => BigInt(0))
    )
    
    return new FixedPointDecimal(BigInt.multiply(rounded, factor), this.scale)
  }
  
  compare(other: FixedPointDecimal): number {
    const [scaledThis, scaledOther] = this.normalizeScale(other)
    return BigInt.Order(scaledThis.value, scaledOther.value)
  }
  
  toString(): string {
    const divisor = BigInt(10) ** BigInt(this.scale)
    const wholePart = BigInt.divide(this.value, divisor).pipe(
      Option.getOrElse(() => BigInt(0))
    )
    const fractionalPart = this.value % divisor
    
    const fractionalStr = BigInt.abs(fractionalPart).toString().padStart(this.scale, '0')
    const sign = BigInt.lessThan(this.value, BigInt(0)) ? '-' : ''
    
    return `${sign}${BigInt.abs(wholePart).toString()}.${fractionalStr}`
  }
  
  toBigInt(): bigint {
    return this.value
  }
  
  toNumber(): Option.Option<number> {
    return BigInt.toNumber(this.value).pipe(
      Option.map(n => n / Math.pow(10, this.scale))
    )
  }
  
  private normalizeScale(other: FixedPointDecimal): [FixedPointDecimal, FixedPointDecimal] {
    const maxScale = Math.max(this.scale, other.scale)
    
    const scaledThis = this.scale < maxScale
      ? new FixedPointDecimal(
          BigInt.multiply(this.value, BigInt(10) ** BigInt(maxScale - this.scale)),
          maxScale
        )
      : this
    
    const scaledOther = other.scale < maxScale
      ? new FixedPointDecimal(
          BigInt.multiply(other.value, BigInt(10) ** BigInt(maxScale - other.scale)),
          maxScale
        )
      : other
    
    return [scaledThis, scaledOther]
  }
}

// Usage examples for financial calculations
const price = FixedPointDecimal.from("99.99")
const taxRate = FixedPointDecimal.from("8.25")
const discount = FixedPointDecimal.from("15.0")

// Calculate discounted price
const discountAmount = price.percentage(discount)
const discountedPrice = price.subtract(discountAmount)

// Calculate tax
const taxAmount = discountedPrice.percentage(taxRate)

// Calculate total
const total = discountedPrice.add(taxAmount)

console.log(`Original Price: $${price.toString()}`)
console.log(`Discount (${discount.toString()}%): $${discountAmount.toString()}`)
console.log(`Discounted Price: $${discountedPrice.toString()}`)
console.log(`Tax (${taxRate.toString()}%): $${taxAmount.toString()}`)
console.log(`Total: $${total.toString()}`)

// Output:
// Original Price: $99.990000000000000000
// Discount (15.000000000000000000%): $14.998500000000000000
// Discounted Price: $84.991500000000000000
// Tax (8.250000000000000000%): $7.011798750000000000
// Total: $92.003298750000000000
```

### Pattern 3: Memory-Efficient BigInt Operations

Optimize BigInt operations for large-scale computations.

```typescript
import { BigInt, Effect, Stream, Array as Arr } from "effect"

// Memory-efficient BigInt processing utilities
const BigIntProcessor = {
  // Process large arrays in chunks to avoid memory issues
  processInChunks: <T>(
    items: ReadonlyArray<bigint>,
    chunkSize: number,
    processor: (chunk: ReadonlyArray<bigint>) => Effect.Effect<T, Error>
  ): Effect.Effect<ReadonlyArray<T>, Error> =>
    Effect.gen(function* () {
      const results: T[] = []
      
      for (let i = 0; i < items.length; i += chunkSize) {
        const chunk = items.slice(i, i + chunkSize)
        const result = yield* processor(chunk)
        results.push(result)
      }
      
      return results
    }),
  
  // Streaming aggregation for very large datasets
  streamingSum: (stream: Stream.Stream<bigint>): Effect.Effect<bigint, never> =>
    stream.pipe(
      Stream.fold(BigInt(0), (acc, value) => BigInt.sum(acc, value))
    ),
  
  // Lazy evaluation for expensive computations
  lazyFactorialSequence: function* (limit: bigint): Generator<bigint> {
    let current = BigInt(1)
    let factorial = BigInt(1)
    
    while (BigInt.lessThanOrEqualTo(current, limit)) {
      if (BigInt.greaterThan(current, BigInt(1))) {
        factorial = BigInt.multiply(factorial, current)
      }
      yield factorial
      current = BigInt.increment(current)
    }
  },
  
  // Parallel processing for independent operations
  parallelCalculations: (
    operations: ReadonlyArray<() => Effect.Effect<bigint, Error>>
  ): Effect.Effect<ReadonlyArray<bigint>, Error> =>
    Effect.all(operations.map(op => op()), { concurrency: "unbounded" }),
  
  // Memoization for recursive BigInt calculations
  createMemoizedCalculator: <T extends ReadonlyArray<bigint>, R>(
    calculator: (...args: T) => Effect.Effect<R, Error>
  ): (...args: T) => Effect.Effect<R, Error> => {
    const cache = new Map<string, R>()
    
    return (...args: T) => {
      const key = args.map(arg => arg.toString()).join(',')
      const cached = cache.get(key)
      
      if (cached !== undefined) {
        return Effect.succeed(cached)
      }
      
      return calculator(...args).pipe(
        Effect.tap(result => Effect.sync(() => cache.set(key, result)))
      )
    }
  },
  
  // Optimized power calculation using binary exponentiation
  fastPower: (base: bigint, exponent: bigint): Effect.Effect<bigint, Error> =>
    Effect.gen(function* () {
      if (BigInt.lessThan(exponent, BigInt(0))) {
        return yield* Effect.fail(new Error("Negative exponents not supported"))
      }
      
      if (BigInt.equals(exponent, BigInt(0))) {
        return BigInt(1)
      }
      
      let result = BigInt(1)
      let currentBase = base
      let currentExp = exponent
      
      while (BigInt.greaterThan(currentExp, BigInt(0))) {
        if (currentExp % BigInt(2) === BigInt(1)) {
          result = BigInt.multiply(result, currentBase)
        }
        
        currentBase = BigInt.multiply(currentBase, currentBase)
        currentExp = BigInt.divide(currentExp, BigInt(2)).pipe(
          Option.getOrElse(() => BigInt(0))
        )
      }
      
      return result
    })
}

// Usage examples
const largeNumbers = Array.from(
  { length: 10000 }, 
  (_, i) => BigInt(i + 1)
)

// Process in chunks to avoid memory issues
const chunkProcessor = (chunk: ReadonlyArray<bigint>) =>
  Effect.succeed(BigInt.sumAll(chunk))

const chunkedResults = Effect.runSync(
  BigIntProcessor.processInChunks(largeNumbers, 1000, chunkProcessor)
)

console.log("Chunk sums:", chunkedResults.map(r => r.toString()))

// Memoized Fibonacci calculation
const memoizedFib = BigIntProcessor.createMemoizedCalculator(
  (n: bigint): Effect.Effect<bigint, Error> =>
    Effect.gen(function* () {
      if (BigInt.lessThanOrEqualTo(n, BigInt(1))) {
        return n
      }
      
      const fib1 = yield* memoizedFib(BigInt.subtract(n, BigInt(1)))
      const fib2 = yield* memoizedFib(BigInt.subtract(n, BigInt(2)))
      
      return BigInt.sum(fib1, fib2)
    })
)

// Calculate Fibonacci numbers efficiently
const fibResults = Effect.runSync(
  Effect.all([
    memoizedFib(BigInt(10)),
    memoizedFib(BigInt(15)),
    memoizedFib(BigInt(20))
  ])
)

console.log("Fibonacci results:", fibResults.map(r => r.toString()))
```

## Integration Examples

### Integration with Effect Schema

Combine BigInt operations with Schema validation for robust data processing.

```typescript
import { Schema, BigInt, Effect, Option } from "effect"

// Define schemas with BigInt validation
const PositiveBigInt = Schema.BigIntFromSelf.pipe(
  Schema.filter(BigInt.greaterThan(BigInt(0)), {
    message: () => "BigInt must be positive"
  })
)

const BigIntRange = (min: bigint, max: bigint) =>
  Schema.BigIntFromSelf.pipe(
    Schema.filter(BigInt.between({ minimum: min, maximum: max }), {
      message: () => `BigInt must be between ${min} and ${max}`
    })
  )

const CryptoCurrency = Schema.BigIntFromSelf.pipe(
  Schema.filter(BigInt.greaterThanOrEqualTo(BigInt(0)), {
    message: () => "Cryptocurrency amount cannot be negative"
  }),
  Schema.transform(
    Schema.BigIntFromSelf,
    {
      decode: (value) => BigInt.clamp({ minimum: BigInt(0), maximum: BigInt("21000000000000000000000000") })(value),
      encode: (value) => value
    }
  )
)

// Blockchain transaction schema
const BlockchainTransactionSchema = Schema.Struct({
  id: Schema.String,
  from: Schema.String,
  to: Schema.String,
  amount: CryptoCurrency,
  gasPrice: PositiveBigInt,
  gasLimit: BigIntRange(BigInt(21000), BigInt(8000000)),
  nonce: Schema.BigIntFromSelf,
  blockNumber: PositiveBigInt,
  timestamp: Schema.BigIntFromSelf
})

// Large number calculation schema
const ScientificCalculationSchema = Schema.Struct({
  operation: Schema.Literal("factorial", "power", "fibonacci"),
  operands: Schema.Array(Schema.BigIntFromSelf).pipe(
    Schema.filter(
      (operands) => operands.length >= 1 && operands.length <= 2,
      { message: () => "Must have 1-2 operands" }
    )
  ),
  result: Schema.optional(Schema.BigIntFromSelf)
})

// Process validated blockchain transaction
const processBlockchainTransaction = (rawData: unknown) =>
  Effect.gen(function* () {
    const transaction = yield* Schema.decodeUnknown(BlockchainTransactionSchema)(rawData)
    
    // Calculate transaction cost
    const gasCost = BigInt.multiply(transaction.gasPrice, transaction.gasLimit)
    const totalCost = BigInt.sum(transaction.amount, gasCost)
    
    // Validate sufficient funds (mock check)
    const sufficientFunds = BigInt.greaterThan(totalCost, BigInt("1000000000000000000")) // 1 ETH
    
    return {
      transaction,
      gasCost,
      totalCost,
      isValid: sufficientFunds
    }
  })

// Usage example
const transactionData = {
  id: "0x123...",
  from: "0xabc...",
  to: "0xdef...",
  amount: "500000000000000000", // 0.5 ETH in wei
  gasPrice: "20000000000", // 20 gwei
  gasLimit: "21000",
  nonce: "42",
  blockNumber: "12345678",
  timestamp: String(Date.now() * 1000) // microseconds
}

// In practice, you would run this effect:
// const result = yield* processBlockchainTransaction(transactionData)
```

### Integration with Effect Stream

Process large BigInt datasets using streaming for memory efficiency.

```typescript
import { Stream, BigInt, Effect, Chunk } from "effect"

// Create streams of large numbers for processing
const createBigIntStream = (start: bigint, count: number): Stream.Stream<bigint> =>
  Stream.iterate(start, (n) => BigInt.increment(n)).pipe(
    Stream.take(count)
  )

// Stream processing for factorials
const factorialStream = (limit: bigint): Stream.Stream<{ n: bigint, factorial: bigint }> =>
  createBigIntStream(BigInt(1), Number(limit)).pipe(
    Stream.scan({ n: BigInt(0), factorial: BigInt(1) }, (acc, n) => ({
      n,
      factorial: BigInt.multiply(acc.factorial, n)
    })),
    Stream.drop(1) // Skip the initial accumulator value
  )

// Prime number generation stream
const primeStream = (limit: number): Stream.Stream<bigint> =>
  createBigIntStream(BigInt(2), limit).pipe(
    Stream.filter(isProbablyPrime),
    Stream.take(limit)
  )

// Simple primality test for demonstration
const isProbablyPrime = (n: bigint): boolean => {
  if (BigInt.lessThanOrEqualTo(n, BigInt(1))) return false
  if (BigInt.equals(n, BigInt(2))) return true
  if (n % BigInt(2) === BigInt(0)) return false
  
  let i = BigInt(3)
  while (BigInt.multiply(i, i) <= n) {
    if (n % i === BigInt(0)) return false
    i = BigInt.sum(i, BigInt(2))
  }
  return true
}

// Fibonacci stream
const fibonacciStream = (count: number): Stream.Stream<bigint> =>
  Stream.iterate(
    { prev: BigInt(0), curr: BigInt(1) },
    ({ prev, curr }) => ({ prev: curr, curr: BigInt.sum(prev, curr) })
  ).pipe(
    Stream.map(({ curr }) => curr),
    Stream.take(count)
  )

// Real-time cryptocurrency price calculation stream
const cryptoPriceStream = (
  baseAmounts: ReadonlyArray<bigint>,
  priceUpdates: Stream.Stream<bigint>
): Stream.Stream<{ amount: bigint, value: bigint, timestamp: number }> =>
  priceUpdates.pipe(
    Stream.map(price => 
      baseAmounts.map(amount => ({
        amount,
        value: BigInt.multiply(amount, price),
        timestamp: Date.now()
      }))
    ),
    Stream.flatMap(Stream.fromIterable)
  )

// Batch processing of large BigInt calculations
const batchCalculationStream = (
  operations: ReadonlyArray<{ type: string, operands: ReadonlyArray<bigint> }>
): Stream.Stream<{ operation: string, result: bigint, duration: number }> =>
  Stream.fromIterable(operations).pipe(
    Stream.mapEffect(op => 
      Effect.gen(function* () {
        const start = Date.now()
        
        const result = yield* (function() {
          switch (op.type) {
            case "sum":
              return Effect.succeed(BigInt.sumAll(op.operands))
            case "product":
              return Effect.succeed(BigInt.multiplyAll(op.operands))
            case "gcd":
              return Effect.succeed(op.operands.reduce(BigInt.gcd))
            case "lcm":
              return Effect.succeed(op.operands.reduce(BigInt.lcm))
            default:
              return Effect.fail(new Error(`Unknown operation: ${op.type}`))
          }
        })()
        
        const duration = Date.now() - start
        
        return {
          operation: op.type,
          result,
          duration
        }
      })
    )
  )

// Usage examples
const processFactorialStream = Effect.runPromise(
  factorialStream(BigInt(20)).pipe(
    Stream.take(10),
    Stream.runCollect
  )
).then(chunk => {
  console.log("First 10 factorials:")
  Chunk.toReadonlyArray(chunk).forEach(item => 
    console.log(`${item.n}! = ${item.factorial}`)
  )
})

const processPrimeStream = Effect.runPromise(
  primeStream(50).pipe(
    Stream.runCollect
  )
).then(chunk => {
  console.log("First 50 primes:")
  console.log(Chunk.toReadonlyArray(chunk).map(p => p.toString()).join(", "))
})
```

### Testing Strategies

Comprehensive testing approaches for BigInt-based calculations.

```typescript
import { BigInt, Effect, Gen } from "effect"
import { describe, it, expect, beforeEach } from "vitest"

// Property-based testing generators for BigInt
const PositiveBigIntGen = Gen.bigint({ min: 1n, max: 1000000n })
const SmallBigIntGen = Gen.bigint({ min: 1n, max: 100n })
const LargeBigIntGen = Gen.bigint({ min: 1000000000000000000n, max: 9999999999999999999n })

describe("BigInt Operations", () => {
  it("should maintain mathematical properties for addition", () => {
    Effect.gen(function* () {
      const a = yield* PositiveBigIntGen
      const b = yield* PositiveBigIntGen
      const c = yield* PositiveBigIntGen
      
      // Commutative property: a + b = b + a
      expect(BigInt.sum(a, b)).toBe(BigInt.sum(b, a))
      
      // Associative property: (a + b) + c = a + (b + c)
      const left = BigInt.sum(BigInt.sum(a, b), c)
      const right = BigInt.sum(a, BigInt.sum(b, c))
      expect(left).toBe(right)
      
      // Identity property: a + 0 = a
      expect(BigInt.sum(a, BigInt(0))).toBe(a)
    }).pipe(Effect.runSync)
  })
  
  it("should handle division by zero safely", () => {
    const result = BigInt.divide(BigInt(10), BigInt(0))
    expect(Option.isNone(result)).toBe(true)
  })
  
  it("should calculate GCD correctly", () => {
    Effect.gen(function* () {
      const a = yield* SmallBigIntGen
      const b = yield* SmallBigIntGen
      
      const gcd = BigInt.gcd(a, b)
      
      // GCD should divide both numbers
      expect(a % gcd).toBe(BigInt(0))
      expect(b % gcd).toBe(BigInt(0))
      
      // GCD should be positive
      expect(BigInt.greaterThan(gcd, BigInt(0))).toBe(true)
    }).pipe(Effect.runSync)
  })
  
  it("should handle large number arithmetic correctly", () => {
    Effect.gen(function* () {
      const large1 = yield* LargeBigIntGen
      const large2 = yield* LargeBigIntGen
      
      // Addition should not overflow
      const sum = BigInt.sum(large1, large2)
      expect(BigInt.greaterThan(sum, large1)).toBe(true)
      expect(BigInt.greaterThan(sum, large2)).toBe(true)
      
      // Multiplication should work for large numbers
      const product = BigInt.multiply(large1, large2)
      expect(BigInt.greaterThan(product, large1)).toBe(true)
      expect(BigInt.greaterThan(product, large2)).toBe(true)
    }).pipe(Effect.runSync)
  })
  
  it("should maintain precision in complex calculations", () => {
    // Test cryptocurrency calculation precision
    const weiPerEth = BigInt("1000000000000000000") // 10^18
    const ethAmount = BigInt(5) // 5 ETH
    const totalWei = BigInt.multiply(ethAmount, weiPerEth)
    
    // Should equal exactly 5 * 10^18
    expect(totalWei).toBe(BigInt("5000000000000000000"))
    
    // Converting back should preserve precision
    const convertedBack = BigInt.divide(totalWei, weiPerEth)
    expect(Option.getOrElse(convertedBack, () => BigInt(0))).toBe(ethAmount)
  })
})

// Mock testing for external BigInt operations
const createMockBigIntCalculator = () => ({
  factorial: vi.fn((n: bigint) => {
    let result = BigInt(1)
    for (let i = BigInt(2); i <= n; i = BigInt.increment(i)) {
      result = BigInt.multiply(result, i)
    }
    return result
  }),
  
  fibonacci: vi.fn((n: bigint) => {
    if (BigInt.lessThanOrEqualTo(n, BigInt(1))) return n
    
    let a = BigInt(0)
    let b = BigInt(1)
    let current = BigInt(2)
    
    while (BigInt.lessThanOrEqualTo(current, n)) {
      const temp = BigInt.sum(a, b)
      a = b
      b = temp
      current = BigInt.increment(current)
    }
    
    return b
  }),
  
  isPrime: vi.fn((n: bigint) => {
    if (BigInt.lessThanOrEqualTo(n, BigInt(1))) return false
    if (BigInt.equals(n, BigInt(2))) return true
    if (n % BigInt(2) === BigInt(0)) return false
    
    let i = BigInt(3)
    while (BigInt.multiply(i, i) <= n) {
      if (n % i === BigInt(0)) return false
      i = BigInt.sum(i, BigInt(2))
    }
    return true
  })
})

// Integration testing with realistic scenarios
describe("Real-World BigInt Scenarios", () => {
  let calculator: ReturnType<typeof createMockBigIntCalculator>
  
  beforeEach(() => {
    calculator = createMockBigIntCalculator()
  })
  
  it("should handle cryptocurrency wallet operations", () => {
    const balanceWei = BigInt("5000000000000000000") // 5 ETH
    const transferAmountWei = BigInt("1500000000000000000") // 1.5 ETH
    const gasPriceWei = BigInt("20000000000") // 20 gwei
    const gasLimit = BigInt(21000)
    
    const gasCost = BigInt.multiply(gasPriceWei, gasLimit)
    const totalCost = BigInt.sum(transferAmountWei, gasCost)
    const remainingBalance = BigInt.subtract(balanceWei, totalCost)
    
    expect(BigInt.greaterThan(remainingBalance, BigInt(0))).toBe(true)
    expect(calculator.factorial).not.toHaveBeenCalled()
  })
  
  it("should calculate large factorials efficiently", () => {
    const result = calculator.factorial(BigInt(50))
    
    expect(calculator.factorial).toHaveBeenCalledWith(BigInt(50))
    expect(BigInt.greaterThan(result, BigInt(0))).toBe(true)
    
    // 50! should be a very large number
    const resultStr = result.toString()
    expect(resultStr.length).toBeGreaterThan(60)
  })
  
  it("should generate Fibonacci sequence accurately", () => {
    const fib10 = calculator.fibonacci(BigInt(10))
    const fib20 = calculator.fibonacci(BigInt(20))
    
    expect(fib10).toBe(BigInt(55))
    expect(fib20).toBe(BigInt(6765))
    expect(calculator.fibonacci).toHaveBeenCalledTimes(2)
  })
})

// Performance testing for large BigInt operations
describe("BigInt Performance", () => {
  it("should handle large number operations within reasonable time", () => {
    const start = performance.now()
    
    const large1 = BigInt("12345678901234567890123456789012345678901234567890")
    const large2 = BigInt("98765432109876543210987654321098765432109876543210")
    
    const sum = BigInt.sum(large1, large2)
    const product = BigInt.multiply(large1, large2)
    const gcd = BigInt.gcd(large1, large2)
    
    const duration = performance.now() - start
    
    expect(duration).toBeLessThan(100) // Should complete within 100ms
    expect(BigInt.greaterThan(sum, large1)).toBe(true)
    expect(BigInt.greaterThan(product, sum)).toBe(true)
    expect(BigInt.greaterThanOrEqualTo(gcd, BigInt(1))).toBe(true)
  })
})
```

## Conclusion

BigInt provides comprehensive arbitrary precision integer arithmetic for applications requiring calculations beyond JavaScript's safe integer range. It enables reliable handling of large numbers in cryptographic, financial, and scientific computing contexts.

Key benefits:
- **Arbitrary Precision** - Handle integers of unlimited size without precision loss
- **Type Safety** - Compile-time guarantees and runtime type guards prevent numeric errors
- **Functional Composition** - All operations are pipeable and work seamlessly with other Effect modules
- **Safe Operations** - Division returns Option types and all operations handle edge cases gracefully
- **Mathematical Rigor** - Built-in support for advanced number theory operations like GCD, LCM, and modular arithmetic
- **Performance Optimized** - Efficient algorithms for common operations and memory-conscious processing patterns

Use BigInt when working with cryptocurrency amounts, large scientific calculations, cryptographic operations, high-precision financial computations, or any application where JavaScript's native number type limitations would cause precision loss or calculation errors.