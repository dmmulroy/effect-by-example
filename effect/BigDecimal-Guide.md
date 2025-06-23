# BigDecimal: A Real-World Guide

## Table of Contents
1. [Introduction & Core Concepts](#introduction--core-concepts)
2. [Basic Usage Patterns](#basic-usage-patterns)
3. [Real-World Examples](#real-world-examples)
4. [Advanced Features Deep Dive](#advanced-features-deep-dive)
5. [Practical Patterns & Best Practices](#practical-patterns--best-practices)
6. [Integration Examples](#integration-examples)

## Introduction & Core Concepts

### The Problem BigDecimal Solves

JavaScript's standard number type uses 64-bit floating-point representation, which leads to precision errors that can be catastrophic in financial, scientific, and monetary calculations where accuracy is paramount.

```typescript
// Traditional approach - floating-point precision problems
function calculateTotalPrice(price: number, taxRate: number, quantity: number): number {
  const subtotal = price * quantity
  const tax = subtotal * taxRate
  const total = subtotal + tax
  
  // Rounds to 2 decimal places - but precision is already lost
  return Math.round(total * 100) / 100
}

// The floating-point precision nightmare
const price = 0.1
const taxRate = 0.2
const quantity = 3

console.log(price * quantity) // 0.30000000000000004 instead of 0.3
console.log(calculateTotalPrice(price, taxRate, quantity)) // 0.36 but is it actually correct?

// Compound interest calculations become unreliable
function compoundInterest(principal: number, rate: number, periods: number): number {
  return principal * Math.pow(1 + rate, periods)
}

const investment = compoundInterest(1000, 0.05, 10)
console.log(investment) // 1628.8946267774416 - but financial precision requires exact values

// Currency calculations with rounding errors
const prices = [19.99, 29.99, 39.99]
const total = prices.reduce((sum, price) => sum + price, 0)
console.log(total) // 89.97000000000001 instead of 89.97

// Division operations lose precision
const result = 1 / 3
console.log(result * 3) // 0.9999999999999999 instead of 1
```

This approach leads to:
- **Financial Errors** - Rounding errors accumulate in monetary calculations
- **Scientific Inaccuracy** - Precision loss in mathematical computations
- **Compliance Issues** - Regulatory requirements for exact decimal arithmetic
- **Data Corruption** - Accumulated errors in long-running calculations
- **User Trust Loss** - Incorrect totals and balances in applications

### The BigDecimal Solution

Effect's BigDecimal provides arbitrary-precision decimal arithmetic, ensuring exact calculations for financial and scientific applications.

```typescript
import { BigDecimal, Effect, Option, pipe } from "effect"

// Type-safe, exact decimal arithmetic
const calculateTotalPrice = (
  price: BigDecimal.BigDecimal, 
  taxRate: BigDecimal.BigDecimal, 
  quantity: number
): Effect.Effect<BigDecimal.BigDecimal> =>
  Effect.gen(function* () {
    const quantityDecimal = BigDecimal.fromBigInt(BigInt(quantity))
    const subtotal = BigDecimal.multiply(price, quantityDecimal)
    const tax = BigDecimal.multiply(subtotal, taxRate)
    return BigDecimal.sum(subtotal, tax)
  })

// Exact precision without rounding errors
const exactPrice = BigDecimal.unsafeFromString("0.1")
const exactTaxRate = BigDecimal.unsafeFromString("0.2")

// Compound interest with perfect precision
const compoundInterest = (
  principal: BigDecimal.BigDecimal,
  rate: BigDecimal.BigDecimal,
  periods: number
): Effect.Effect<BigDecimal.BigDecimal> =>
  Effect.gen(function* () {
    const onePlusRate = BigDecimal.sum(BigDecimal.fromBigInt(1n), rate)
    let result = principal
    
    for (let i = 0; i < periods; i++) {
      result = BigDecimal.multiply(result, onePlusRate)
    }
    
    return result
  })

// Safe division operations
const safeDivision = (dividend: BigDecimal.BigDecimal, divisor: BigDecimal.BigDecimal) =>
  BigDecimal.divide(dividend, divisor).pipe(
    Option.getOrElse(() => BigDecimal.fromBigInt(0n))
  )
```

### Key Concepts

**Arbitrary Precision**: BigDecimal stores numbers as a `BigInt` value with a scale, allowing for exact decimal representation without floating-point limitations.

**Immutability**: All operations return new BigDecimal instances, preventing mutation bugs common in financial calculations.

**Safe Operations**: Division and other potentially error-prone operations return `Option` types, handling edge cases like division by zero gracefully.

**Scale Management**: The scale determines decimal places, allowing precise control over precision and rounding behavior.

## Basic Usage Patterns

### Pattern 1: Creating and Converting BigDecimal Values

```typescript
import { BigDecimal, Option, Effect } from "effect"

// Creating BigDecimal values safely
const fromString = BigDecimal.fromString("123.456") // Option<BigDecimal>
const unsafeFromString = BigDecimal.unsafeFromString("123.456") // BigDecimal (throws on invalid)
const fromBigInt = BigDecimal.fromBigInt(123n) // BigDecimal with scale 0
const fromMake = BigDecimal.make(123456n, 3) // 123.456 (value: 123456n, scale: 3)

// Safe conversion from numbers (avoiding floating-point issues)
const safeFromNumber = BigDecimal.safeFromNumber(123.456) // Option<BigDecimal>
const unsafeFromNumber = BigDecimal.unsafeFromNumber(123.456) // BigDecimal (throws on non-finite)

// Converting back to other types
const toString = BigDecimal.format(unsafeFromString) // "123.456"
const toNumber = BigDecimal.unsafeToNumber(unsafeFromString) // 123.456 (use carefully!)
const toExponential = BigDecimal.toExponential(BigDecimal.make(123456n, -2)) // "1.23456e+7"

// Working with Options for safe parsing
const parseUserInput = (input: string): Effect.Effect<BigDecimal.BigDecimal, string> =>
  Effect.gen(function* () {
    const parsed = BigDecimal.fromString(input)
    if (Option.isSome(parsed)) {
      return parsed.value
    }
    return yield* Effect.fail(`Invalid decimal: ${input}`)
  })
```

### Pattern 2: Basic Arithmetic Operations

```typescript
import { BigDecimal, Option, pipe } from "effect"

// Safe arithmetic with exact precision
const price = BigDecimal.unsafeFromString("19.99")
const taxRate = BigDecimal.unsafeFromString("0.08")
const quantity = BigDecimal.fromBigInt(3n)

// Addition and subtraction
const subtotal = BigDecimal.multiply(price, quantity) // 59.97
const discount = BigDecimal.unsafeFromString("5.00")
const afterDiscount = BigDecimal.subtract(subtotal, discount) // 54.97

// Multiplication and division
const tax = BigDecimal.multiply(afterDiscount, taxRate) // 4.3976
const total = BigDecimal.sum(afterDiscount, tax) // 59.3676

// Safe division returning Option
const averagePrice = pipe(
  BigDecimal.divide(total, quantity),
  Option.getOrElse(() => BigDecimal.fromBigInt(0n))
) // 19.789200...

// Chaining operations functionally
const calculateFinalPrice = (basePrice: BigDecimal.BigDecimal) =>
  basePrice.pipe(
    BigDecimal.multiply(quantity),
    BigDecimal.subtract(discount),
    base => BigDecimal.sum(base, BigDecimal.multiply(base, taxRate))
  )
```

### Pattern 3: Precision Control and Rounding

```typescript
import { BigDecimal } from "effect"

// Working with different scales
const precise = BigDecimal.make(123456789n, 6) // 123.456789
const scaled = BigDecimal.scale(precise, 2) // Scale to 2 decimal places: 123.45

// Rounding operations with different modes
const value = BigDecimal.unsafeFromString("123.456")

const rounded = BigDecimal.round(value, { scale: 2, mode: "half-even" }) // 123.46
const ceiled = BigDecimal.ceil(value, 2) // 123.46
const floored = BigDecimal.floor(value, 2) // 123.45
const truncated = BigDecimal.truncate(value, 2) // 123.45

// Normalization removes trailing zeros
const withTrailingZeros = BigDecimal.make(123000n, 3) // 123.000
const normalized = BigDecimal.normalize(withTrailingZeros) // 123

// Creating rounded currency values
const roundToCurrency = (amount: BigDecimal.BigDecimal): BigDecimal.BigDecimal =>
  BigDecimal.round(amount, { scale: 2, mode: "half-even" })

const currencyAmount = roundToCurrency(BigDecimal.unsafeFromString("123.456789"))
console.log(BigDecimal.format(currencyAmount)) // "123.46"
```

## Real-World Examples

### Example 1: Financial Portfolio Calculator

A comprehensive portfolio management system that handles multiple currencies, tax calculations, and performance metrics with perfect precision.

```typescript
import { BigDecimal, Effect, Array as Arr, pipe } from "effect"

interface Position {
  readonly symbol: string
  readonly quantity: BigDecimal.BigDecimal
  readonly costBasis: BigDecimal.BigDecimal
  readonly currentPrice: BigDecimal.BigDecimal
  readonly currency: string
}

interface TaxBracket {
  readonly rate: BigDecimal.BigDecimal
  readonly threshold: BigDecimal.BigDecimal
}

const calculatePortfolioValue = (positions: readonly Position[]) =>
  Effect.gen(function* () {
    const values = yield* Effect.all(
      positions.map(position => Effect.gen(function* () {
        const marketValue = BigDecimal.multiply(position.quantity, position.currentPrice)
        const costValue = BigDecimal.multiply(position.quantity, position.costBasis)
        const gainLoss = BigDecimal.subtract(marketValue, costValue)
        
        return {
          symbol: position.symbol,
          marketValue,
          costValue,
          gainLoss,
          gainLossPercent: pipe(
            BigDecimal.divide(gainLoss, costValue),
            option => option.pipe(
              Option.map(ratio => BigDecimal.multiply(ratio, BigDecimal.fromBigInt(100n))),
              Option.getOrElse(() => BigDecimal.fromBigInt(0n))
            )
          )
        }
      }))
    )
    
    const totalMarketValue = BigDecimal.sumAll(values.map(v => v.marketValue))
    const totalCostBasis = BigDecimal.sumAll(values.map(v => v.costValue))
    const totalGainLoss = BigDecimal.subtract(totalMarketValue, totalCostBasis)
    
    return {
      positions: values,
      totalMarketValue,
      totalCostBasis,
      totalGainLoss,
      totalReturn: pipe(
        BigDecimal.divide(totalGainLoss, totalCostBasis),
        option => option.pipe(
          Option.map(ratio => BigDecimal.multiply(ratio, BigDecimal.fromBigInt(100n))),
          Option.getOrElse(() => BigDecimal.fromBigInt(0n))
        )
      )
    }
  })

const calculateCapitalGainsTax = (
  gainLoss: BigDecimal.BigDecimal,
  taxBrackets: readonly TaxBracket[]
) =>
  Effect.gen(function* () {
    if (!BigDecimal.isPositive(gainLoss)) {
      return BigDecimal.fromBigInt(0n)
    }
    
    let remainingGain = gainLoss
    let totalTax = BigDecimal.fromBigInt(0n)
    
    for (const bracket of taxBrackets) {
      if (BigDecimal.lessThanOrEqualTo(remainingGain, BigDecimal.fromBigInt(0n))) {
        break
      }
      
      const taxableAtThisBracket = BigDecimal.min(remainingGain, bracket.threshold)
      const taxAtThisBracket = BigDecimal.multiply(taxableAtThisBracket, bracket.rate)
      
      totalTax = BigDecimal.sum(totalTax, taxAtThisBracket)
      remainingGain = BigDecimal.subtract(remainingGain, taxableAtThisBracket)
    }
    
    return totalTax
  })

// Usage example
const portfolioPositions: readonly Position[] = [
  {
    symbol: "AAPL",
    quantity: BigDecimal.unsafeFromString("100"),
    costBasis: BigDecimal.unsafeFromString("150.00"),
    currentPrice: BigDecimal.unsafeFromString("175.50"),
    currency: "USD"
  },
  {
    symbol: "GOOGL", 
    quantity: BigDecimal.unsafeFromString("50"),
    costBasis: BigDecimal.unsafeFromString("2800.00"),
    currentPrice: BigDecimal.unsafeFromString("2950.75"),
    currency: "USD"
  }
]

const taxBrackets: readonly TaxBracket[] = [
  { rate: BigDecimal.unsafeFromString("0.15"), threshold: BigDecimal.unsafeFromString("10000") },
  { rate: BigDecimal.unsafeFromString("0.20"), threshold: BigDecimal.unsafeFromString("50000") }
]
```

### Example 2: E-commerce Pricing Engine

A sophisticated pricing system handling discounts, taxes, shipping, and multi-currency support with precise decimal calculations.

```typescript
import { BigDecimal, Effect, Option, Array as Arr, pipe } from "effect"

interface Product {
  readonly id: string
  readonly basePrice: BigDecimal.BigDecimal
  readonly category: string
  readonly taxable: boolean
}

interface Discount {
  readonly type: 'percentage' | 'fixed'
  readonly value: BigDecimal.BigDecimal
  readonly minimumOrder?: BigDecimal.BigDecimal
  readonly applicableCategories?: readonly string[]
}

interface TaxRule {
  readonly region: string
  readonly rate: BigDecimal.BigDecimal
  readonly exemptCategories?: readonly string[]
}

interface ShippingRate {
  readonly region: string
  readonly baseRate: BigDecimal.BigDecimal
  readonly perItemRate: BigDecimal.BigDecimal
  readonly freeShippingThreshold?: BigDecimal.BigDecimal
}

const calculateLineItemPrice = (
  product: Product,
  quantity: number,
  discounts: readonly Discount[]
) =>
  Effect.gen(function* () {
    const quantityDecimal = BigDecimal.fromBigInt(BigInt(quantity))
    let itemPrice = product.basePrice
    
    // Apply applicable discounts
    for (const discount of discounts) {
      if (discount.applicableCategories && 
          !discount.applicableCategories.includes(product.category)) {
        continue
      }
      
      if (discount.type === 'percentage') {
        const discountAmount = BigDecimal.multiply(
          itemPrice,
          BigDecimal.divide(discount.value, BigDecimal.fromBigInt(100n)).pipe(
            Option.getOrElse(() => BigDecimal.fromBigInt(0n))
          )
        )
        itemPrice = BigDecimal.subtract(itemPrice, discountAmount)
      } else {
        itemPrice = BigDecimal.subtract(itemPrice, discount.value)
      }
    }
    
    // Ensure price doesn't go negative
    if (BigDecimal.isNegative(itemPrice)) {
      itemPrice = BigDecimal.fromBigInt(0n)
    }
    
    const lineTotal = BigDecimal.multiply(itemPrice, quantityDecimal)
    
    return {
      productId: product.id,
      unitPrice: product.basePrice,
      discountedPrice: itemPrice,
      quantity: quantityDecimal,
      lineTotal
    }
  })

const calculateTaxes = (
  lineItems: readonly { lineTotal: BigDecimal.BigDecimal; productId: string }[],
  products: readonly Product[],
  taxRules: readonly TaxRule[],
  region: string
) =>
  Effect.gen(function* () {
    const applicableTaxRule = taxRules.find(rule => rule.region === region)
    if (!applicableTaxRule) {
      return BigDecimal.fromBigInt(0n)
    }
    
    let taxableAmount = BigDecimal.fromBigInt(0n)
    
    for (const lineItem of lineItems) {
      const product = products.find(p => p.id === lineItem.productId)
      if (product?.taxable && 
          !applicableTaxRule.exemptCategories?.includes(product.category)) {
        taxableAmount = BigDecimal.sum(taxableAmount, lineItem.lineTotal)
      }
    }
    
    return BigDecimal.multiply(taxableAmount, applicableTaxRule.rate)
  })

const calculateShipping = (
  lineItems: readonly { lineTotal: BigDecimal.BigDecimal }[],
  itemCount: number,
  shippingRates: readonly ShippingRate[],
  region: string
) =>
  Effect.gen(function* () {
    const shippingRate = shippingRates.find(rate => rate.region === region)
    if (!shippingRate) {
      return BigDecimal.fromBigInt(0n)
    }
    
    const subtotal = BigDecimal.sumAll(lineItems.map(item => item.lineTotal))
    
    // Check for free shipping threshold
    if (shippingRate.freeShippingThreshold && 
        BigDecimal.greaterThanOrEqualTo(subtotal, shippingRate.freeShippingThreshold)) {
      return BigDecimal.fromBigInt(0n)
    }
    
    const itemCountDecimal = BigDecimal.fromBigInt(BigInt(itemCount))
    const perItemCharge = BigDecimal.multiply(shippingRate.perItemRate, itemCountDecimal)
    
    return BigDecimal.sum(shippingRate.baseRate, perItemCharge)
  })

// Comprehensive order total calculation
const calculateOrderTotal = (
  cart: readonly { product: Product; quantity: number }[],
  discounts: readonly Discount[],
  taxRules: readonly TaxRule[],
  shippingRates: readonly ShippingRate[],
  region: string
) =>
  Effect.gen(function* () {
    // Calculate line items with discounts
    const lineItems = yield* Effect.all(
      cart.map(item => calculateLineItemPrice(item.product, item.quantity, discounts))
    )
    
    const subtotal = BigDecimal.sumAll(lineItems.map(item => item.lineTotal))
    
    // Calculate taxes
    const taxes = yield* calculateTaxes(
      lineItems,
      cart.map(item => item.product),
      taxRules,
      region
    )
    
    // Calculate shipping
    const shipping = yield* calculateShipping(
      lineItems,
      cart.reduce((sum, item) => sum + item.quantity, 0),
      shippingRates,
      region
    )
    
    const total = pipe(
      subtotal,
      total => BigDecimal.sum(total, taxes),
      total => BigDecimal.sum(total, shipping)
    )
    
    return {
      lineItems,
      subtotal,
      taxes,
      shipping,
      total: BigDecimal.round(total, { scale: 2, mode: "half-even" })
    }
  })
```

### Example 3: Scientific Calculation Engine

A high-precision mathematical computation system for scientific applications requiring exact decimal arithmetic.

```typescript
import { BigDecimal, Effect, Array as Arr, pipe } from "effect"

// Statistical calculations with perfect precision
const calculateMean = (values: readonly BigDecimal.BigDecimal[]) =>
  Effect.gen(function* () {
    if (values.length === 0) {
      return yield* Effect.fail("Cannot calculate mean of empty dataset")
    }
    
    const sum = BigDecimal.sumAll(values)
    const count = BigDecimal.fromBigInt(BigInt(values.length))
    
    return yield* pipe(
      BigDecimal.divide(sum, count),
      Effect.fromOption(() => "Division by zero in mean calculation")
    )
  })

const calculateVariance = (values: readonly BigDecimal.BigDecimal[]) =>
  Effect.gen(function* () {
    const mean = yield* calculateMean(values)
    
    const squaredDifferences = values.map(value => {
      const diff = BigDecimal.subtract(value, mean)
      return BigDecimal.multiply(diff, diff)
    })
    
    return yield* calculateMean(squaredDifferences)
  })

const calculateStandardDeviation = (values: readonly BigDecimal.BigDecimal[]) =>
  Effect.gen(function* () {
    const variance = yield* calculateVariance(values)
    
    // For exact square root, we'd need a custom implementation
    // Here we'll use a high-precision approximation using Newton's method
    return yield* calculateSquareRoot(variance, 50) // 50 iterations for high precision
  })

// Newton's method for square root with arbitrary precision
const calculateSquareRoot = (
  value: BigDecimal.BigDecimal,
  iterations: number
) =>
  Effect.gen(function* () {
    if (BigDecimal.isNegative(value)) {
      return yield* Effect.fail("Cannot calculate square root of negative number")
    }
    
    if (BigDecimal.isZero(value)) {
      return BigDecimal.fromBigInt(0n)
    }
    
    // Initial guess - use the value itself
    let x = value
    const two = BigDecimal.fromBigInt(2n)
    
    for (let i = 0; i < iterations; i++) {
      // x_new = (x + value/x) / 2
      const division = yield* pipe(
        BigDecimal.divide(value, x),
        Effect.fromOption(() => "Division error in square root calculation")
      )
      
      const sum = BigDecimal.sum(x, division)
      x = yield* pipe(
        BigDecimal.divide(sum, two),
        Effect.fromOption(() => "Division error in square root calculation")
      )
    }
    
    return x
  })

// Compound interest with exact precision
const calculateCompoundInterest = (
  principal: BigDecimal.BigDecimal,
  annualRate: BigDecimal.BigDecimal,
  periodsPerYear: number,
  years: number
) =>
  Effect.gen(function* () {
    const periodsDecimal = BigDecimal.fromBigInt(BigInt(periodsPerYear))
    const totalPeriods = periodsPerYear * years
    
    // Rate per period = annual rate / periods per year
    const ratePerPeriod = yield* pipe(
      BigDecimal.divide(annualRate, periodsDecimal),
      Effect.fromOption(() => "Division error in interest calculation")
    )
    
    // (1 + rate/n)^(n*t)
    const onePlusRate = BigDecimal.sum(BigDecimal.fromBigInt(1n), ratePerPeriod)
    
    let result = principal
    for (let i = 0; i < totalPeriods; i++) {
      result = BigDecimal.multiply(result, onePlusRate)
    }
    
    return result
  })

// Precise geometric series calculation
const calculateGeometricSeries = (
  firstTerm: BigDecimal.BigDecimal,
  commonRatio: BigDecimal.BigDecimal,
  numberOfTerms: number
) =>
  Effect.gen(function* () {
    if (numberOfTerms <= 0) {
      return BigDecimal.fromBigInt(0n)
    }
    
    if (BigDecimal.equals(commonRatio, BigDecimal.fromBigInt(1n))) {
      // Special case: r = 1, sum = a * n
      const nDecimal = BigDecimal.fromBigInt(BigInt(numberOfTerms))
      return BigDecimal.multiply(firstTerm, nDecimal)
    }
    
    // General case: sum = a * (1 - r^n) / (1 - r)
    let rPowerN = BigDecimal.fromBigInt(1n)
    for (let i = 0; i < numberOfTerms; i++) {
      rPowerN = BigDecimal.multiply(rPowerN, commonRatio)
    }
    
    const numerator = BigDecimal.multiply(
      firstTerm,
      BigDecimal.subtract(BigDecimal.fromBigInt(1n), rPowerN)
    )
    
    const denominator = BigDecimal.subtract(BigDecimal.fromBigInt(1n), commonRatio)
    
    return yield* pipe(
      BigDecimal.divide(numerator, denominator),
      Effect.fromOption(() => "Division error in geometric series calculation")
    )
  })

// Usage example
const scientificCalculation = Effect.gen(function* () {
  const dataPoints = [
    BigDecimal.unsafeFromString("1.23456789"),
    BigDecimal.unsafeFromString("2.34567891"),
    BigDecimal.unsafeFromString("3.45678912"),
    BigDecimal.unsafeFromString("4.56789123"),
    BigDecimal.unsafeFromString("5.67891234")
  ]
  
  const mean = yield* calculateMean(dataPoints)
  const variance = yield* calculateVariance(dataPoints)
  const stdDev = yield* calculateStandardDeviation(dataPoints)
  
  const investment = yield* calculateCompoundInterest(
    BigDecimal.unsafeFromString("1000"),
    BigDecimal.unsafeFromString("0.05"),
    12, // monthly compounding
    10  // 10 years
  )
  
  return {
    statistics: {
      mean: BigDecimal.format(mean),
      variance: BigDecimal.format(variance),
      standardDeviation: BigDecimal.format(stdDev)
    },
    investment: BigDecimal.format(investment)
  }
})
```

## Advanced Features Deep Dive

### Scale Management and Precision Control

Understanding how BigDecimal manages scale is crucial for maintaining precision across calculations.

```typescript
import { BigDecimal, Effect, pipe } from "effect"

// Scale represents the number of digits after the decimal point
const exploreScale = Effect.gen(function* () {
  // Same value, different internal representations
  const value1 = BigDecimal.make(12345n, 2) // 123.45
  const value2 = BigDecimal.make(123450n, 3) // 123.450 
  const value3 = BigDecimal.make(1234500n, 4) // 123.4500
  
  console.log(BigDecimal.format(value1)) // "123.45"
  console.log(BigDecimal.format(value2)) // "123.450"
  console.log(BigDecimal.format(value3)) // "123.4500"
  
  // Normalization removes trailing zeros
  const normalized1 = BigDecimal.normalize(value1) // No change
  const normalized2 = BigDecimal.normalize(value2) // 123.45 (scale 2)
  const normalized3 = BigDecimal.normalize(value3) // 123.45 (scale 2)
  
  // Scale operations for precision control
  const increased = BigDecimal.scale(value1, 4) // 123.4500 (scale 4)
  const decreased = BigDecimal.scale(value1, 1) // 123.4 (scale 1, rounds down)
  
  return { value1, normalized2, increased, decreased }
})

// Dynamic precision management
const adaptivePrecision = (
  calculations: readonly (() => BigDecimal.BigDecimal)[],
  targetScale: number
) =>
  Effect.gen(function* () {
    const results = calculations.map(calc => calc())
    
    // Find the maximum scale in results
    const maxScale = results.reduce((max, result) => 
      Math.max(max, result.scale), 0)
    
    // Normalize all results to the same scale
    const normalizedResults = results.map(result => 
      BigDecimal.scale(result, Math.max(maxScale, targetScale)))
    
    return normalizedResults
  })
```

### Advanced Rounding Strategies

BigDecimal provides comprehensive rounding modes for different use cases.

```typescript
import { BigDecimal } from "effect"

// Understanding all rounding modes
const exploreRoundingModes = (value: string) => {
  const decimal = BigDecimal.unsafeFromString(value)
  const scale = 1
  
  return {
    original: BigDecimal.format(decimal),
    ceil: BigDecimal.format(BigDecimal.round(decimal, { scale, mode: "ceil" })),
    floor: BigDecimal.format(BigDecimal.round(decimal, { scale, mode: "floor" })),
    toZero: BigDecimal.format(BigDecimal.round(decimal, { scale, mode: "to-zero" })),
    fromZero: BigDecimal.format(BigDecimal.round(decimal, { scale, mode: "from-zero" })),
    halfCeil: BigDecimal.format(BigDecimal.round(decimal, { scale, mode: "half-ceil" })),
    halfFloor: BigDecimal.format(BigDecimal.round(decimal, { scale, mode: "half-floor" })),
    halfToZero: BigDecimal.format(BigDecimal.round(decimal, { scale, mode: "half-to-zero" })),
    halfFromZero: BigDecimal.format(BigDecimal.round(decimal, { scale, mode: "half-from-zero" })),
    halfEven: BigDecimal.format(BigDecimal.round(decimal, { scale, mode: "half-even" })),
    halfOdd: BigDecimal.format(BigDecimal.round(decimal, { scale, mode: "half-odd" }))
  }
}

// Financial rounding helper
const financialRound = (amount: BigDecimal.BigDecimal): BigDecimal.BigDecimal =>
  BigDecimal.round(amount, { scale: 2, mode: "half-even" })

// Statistical rounding for data analysis  
const statisticalRound = (value: BigDecimal.BigDecimal, precision: number): BigDecimal.BigDecimal =>
  BigDecimal.round(value, { scale: precision, mode: "half-from-zero" })

// Tax calculation rounding (always round up for safety)
const taxRound = (taxAmount: BigDecimal.BigDecimal): BigDecimal.BigDecimal =>
  BigDecimal.ceil(taxAmount, 2)

// Example usage
const roundingExamples = [
  exploreRoundingModes("12.35"), // Exactly halfway case
  exploreRoundingModes("12.34"), // Below halfway
  exploreRoundingModes("12.36"), // Above halfway
  exploreRoundingModes("-12.35"), // Negative halfway case
]
```

### High-Performance Batch Operations

Optimizing BigDecimal operations for large datasets.

```typescript
import { BigDecimal, Effect, Array as Arr, pipe } from "effect"

// Batch processing for large financial datasets
const processBatchTransactions = (
  transactions: readonly { amount: string; type: 'credit' | 'debit' }[]
) =>
  Effect.gen(function* () {
    // Convert all amounts to BigDecimal in parallel
    const convertedTransactions = yield* Effect.all(
      transactions.map(tx => Effect.gen(function* () {
        const amount = BigDecimal.fromString(tx.amount)
        if (Option.isNone(amount)) {
          return yield* Effect.fail(`Invalid amount: ${tx.amount}`)
        }
        return { amount: amount.value, type: tx.type }
      })),
      { concurrency: "unbounded" }
    )
    
    // Separate credits and debits
    const credits = convertedTransactions
      .filter(tx => tx.type === 'credit')
      .map(tx => tx.amount)
    
    const debits = convertedTransactions
      .filter(tx => tx.type === 'debit')
      .map(tx => tx.amount)
    
    // Calculate totals using sumAll for efficiency
    const totalCredits = BigDecimal.sumAll(credits)
    const totalDebits = BigDecimal.sumAll(debits)
    const netAmount = BigDecimal.subtract(totalCredits, totalDebits)
    
    return {
      transactionCount: convertedTransactions.length,
      totalCredits,
      totalDebits,
      netAmount,
      isPositive: BigDecimal.isPositive(netAmount)
    }
  })

// Streaming calculations for memory efficiency
const processLargeDataset = <T>(
  data: readonly T[],
  transform: (item: T) => BigDecimal.BigDecimal,
  chunkSize: number = 1000
) =>
  Effect.gen(function* () {
    let runningTotal = BigDecimal.fromBigInt(0n)
    let processedCount = 0
    
    // Process in chunks to avoid memory issues
    for (let i = 0; i < data.length; i += chunkSize) {
      const chunk = data.slice(i, i + chunkSize)
      const chunkValues = chunk.map(transform)
      const chunkSum = BigDecimal.sumAll(chunkValues)
      
      runningTotal = BigDecimal.sum(runningTotal, chunkSum)
      processedCount += chunk.length
      
      // Yield control periodically for non-blocking processing
      if (processedCount % 10000 === 0) {
        yield* Effect.sleep("1 millis")
      }
    }
    
    return {
      total: runningTotal,
      processedCount,
      average: pipe(
        BigDecimal.divide(runningTotal, BigDecimal.fromBigInt(BigInt(processedCount))),
        Option.getOrElse(() => BigDecimal.fromBigInt(0n))
      )
    }
  })
```

## Practical Patterns & Best Practices

### Pattern 1: Safe Parsing and Validation Pipeline

```typescript
import { BigDecimal, Effect, pipe, Schema } from "effect"

// Comprehensive input validation pipeline
const validateAndParseDecimal = (
  input: unknown,
  constraints?: {
    min?: BigDecimal.BigDecimal
    max?: BigDecimal.BigDecimal
    scale?: number
  }
) =>
  Effect.gen(function* () {
    // Type validation
    if (typeof input !== 'string') {
      return yield* Effect.fail(`Expected string, got ${typeof input}`)
    }
    
    // Parse to BigDecimal
    const parsed = BigDecimal.fromString(input)
    if (Option.isNone(parsed)) {
      return yield* Effect.fail(`Invalid decimal format: ${input}`)
    }
    
    const decimal = parsed.value
    
    // Range validation
    if (constraints?.min && BigDecimal.lessThan(decimal, constraints.min)) {
      return yield* Effect.fail(`Value below minimum: ${input} < ${BigDecimal.format(constraints.min)}`)
    }
    
    if (constraints?.max && BigDecimal.greaterThan(decimal, constraints.max)) {
      return yield* Effect.fail(`Value above maximum: ${input} > ${BigDecimal.format(constraints.max)}`)
    }
    
    // Scale validation
    if (constraints?.scale !== undefined) {
      const scaledDecimal = BigDecimal.scale(decimal, constraints.scale)
      return scaledDecimal
    }
    
    return decimal
  })

// Monetary amount parser with currency constraints
const parseMonetaryAmount = (input: string) =>
  validateAndParseDecimal(input, {
    min: BigDecimal.fromBigInt(0n),
    max: BigDecimal.unsafeFromString("999999999.99"),
    scale: 2
  })

// Percentage parser (0-100)
const parsePercentage = (input: string) =>
  validateAndParseDecimal(input, {
    min: BigDecimal.fromBigInt(0n),
    max: BigDecimal.fromBigInt(100n),
    scale: 4
  }).pipe(
    Effect.map(pct => BigDecimal.divide(pct, BigDecimal.fromBigInt(100n)).pipe(
      Option.getOrElse(() => BigDecimal.fromBigInt(0n))
    ))
  )
```

### Pattern 2: Error-Safe Calculation Chains

```typescript
import { BigDecimal, Effect, Option, pipe } from "effect"

// Comprehensive calculation with error recovery
const safeCalculationChain = (
  operands: readonly string[],
  operations: readonly ('add' | 'subtract' | 'multiply' | 'divide')[]
) =>
  Effect.gen(function* () {
    // Parse all operands
    const numbers = yield* Effect.all(
      operands.map(op => pipe(
        BigDecimal.fromString(op),
        Effect.fromOption(() => `Invalid number: ${op}`)
      ))
    )
    
    if (numbers.length === 0) {
      return yield* Effect.fail("No operands provided")
    }
    
    // Start with first number
    let result = numbers[0]
    
    // Apply operations sequentially
    for (let i = 0; i < operations.length && i + 1 < numbers.length; i++) {
      const operation = operations[i]
      const operand = numbers[i + 1]
      
      switch (operation) {
        case 'add':
          result = BigDecimal.sum(result, operand)
          break
          
        case 'subtract':
          result = BigDecimal.subtract(result, operand)
          break
          
        case 'multiply':
          result = BigDecimal.multiply(result, operand)
          break
          
        case 'divide':
          const division = BigDecimal.divide(result, operand)
          if (Option.isNone(division)) {
            return yield* Effect.fail(`Division by zero at operation ${i + 1}`)
          }
          result = division.value
          break
      }
    }
    
    return result
  }).pipe(
    Effect.catchAll(error => Effect.succeed({
      error: error,
      result: BigDecimal.fromBigInt(0n)
    }))
  )

// Conditional calculation with fallbacks
const calculateWithFallbacks = (
  primary: () => Effect.Effect<BigDecimal.BigDecimal, string>,
  fallbacks: readonly (() => Effect.Effect<BigDecimal.BigDecimal, string>)[]
) => {
  let calculation = primary()
  
  for (const fallback of fallbacks) {
    calculation = calculation.pipe(
      Effect.catchAll(() => fallback())
    )
  }
  
  return calculation.pipe(
    Effect.catchAll(() => Effect.succeed(BigDecimal.fromBigInt(0n)))
  )
}
```

### Pattern 3: Performance-Optimized Aggregations

```typescript
import { BigDecimal, Effect, Array as Arr, HashMap, pipe } from "effect"

// Optimized grouping and aggregation
const groupAndAggregate = <K extends string | number>(
  data: readonly { key: K; value: BigDecimal.BigDecimal }[],
  aggregation: 'sum' | 'average' | 'min' | 'max' = 'sum'
) =>
  Effect.gen(function* () {
    // Group by key efficiently
    const groups = data.reduce((acc, item) => {
      const existing = acc.get(item.key) || []
      acc.set(item.key, [...existing, item.value])
      return acc
    }, new Map<K, BigDecimal.BigDecimal[]>())
    
    // Apply aggregation function
    const results = new Map<K, BigDecimal.BigDecimal>()
    
    for (const [key, values] of groups.entries()) {
      let result: BigDecimal.BigDecimal
      
      switch (aggregation) {
        case 'sum':
          result = BigDecimal.sumAll(values)
          break
          
        case 'average':
          const sum = BigDecimal.sumAll(values)
          const count = BigDecimal.fromBigInt(BigInt(values.length))
          result = pipe(
            BigDecimal.divide(sum, count),
            Option.getOrElse(() => BigDecimal.fromBigInt(0n))
          )
          break
          
        case 'min':
          result = values.reduce((min, val) => BigDecimal.min(min, val), values[0])
          break
          
        case 'max':
          result = values.reduce((max, val) => BigDecimal.max(max, val), values[0])
          break
      }
      
      results.set(key, result)
    }
    
    return results
  })

// Moving averages for time series data
const calculateMovingAverage = (
  data: readonly BigDecimal.BigDecimal[],
  windowSize: number
) =>
  Effect.gen(function* () {
    if (windowSize <= 0 || windowSize > data.length) {
      return yield* Effect.fail("Invalid window size")
    }
    
    const movingAverages: BigDecimal.BigDecimal[] = []
    
    for (let i = windowSize - 1; i < data.length; i++) {
      const window = data.slice(i - windowSize + 1, i + 1)
      const sum = BigDecimal.sumAll(window)
      const average = pipe(
        BigDecimal.divide(sum, BigDecimal.fromBigInt(BigInt(windowSize))),
        Option.getOrElse(() => BigDecimal.fromBigInt(0n))
      )
      movingAverages.push(average)
    }
    
    return movingAverages
  })
```

## Integration Examples

### Integration with Schema for Type-Safe APIs

```typescript
import { BigDecimal, Schema, Effect, pipe } from "effect"

// Custom Schema for BigDecimal with validation
const MonetaryAmountSchema = Schema.transformOrFail(
  Schema.String,
  Schema.instanceOf(BigDecimal.BigDecimal),
  {
    decode: (str, _, ast) => 
      pipe(
        BigDecimal.fromString(str),
        Effect.fromOption(() => 
          new ParseResult.Type(ast, str, "Invalid decimal format")
        ),
        Effect.flatMap(decimal => {
          if (BigDecimal.isNegative(decimal)) {
            return Effect.fail(
              new ParseResult.Type(ast, str, "Amount cannot be negative")
            )
          }
          const rounded = BigDecimal.round(decimal, { scale: 2, mode: "half-even" })
          return Effect.succeed(rounded)
        })
      ),
    encode: (decimal) => Effect.succeed(BigDecimal.format(decimal))
  }
)

// API request/response schemas
const CreateTransactionRequest = Schema.Struct({
  amount: MonetaryAmountSchema,
  description: Schema.String,
  category: Schema.String,
  date: Schema.DateFromString
})

const TransactionResponse = Schema.Struct({
  id: Schema.String,
  amount: MonetaryAmountSchema,
  description: Schema.String,
  category: Schema.String,
  date: Schema.DateFromString,
  balance: MonetaryAmountSchema
})

// Type-safe API handlers
const handleCreateTransaction = (
  request: unknown,
  currentBalance: BigDecimal.BigDecimal
) =>
  Effect.gen(function* () {
    const validated = yield* Schema.decodeUnknown(CreateTransactionRequest)(request)
    
    const newBalance = BigDecimal.sum(currentBalance, validated.amount)
    
    const response = {
      id: crypto.randomUUID(),
      amount: validated.amount,
      description: validated.description,
      category: validated.category,
      date: validated.date,
      balance: newBalance
    }
    
    return yield* Schema.encode(TransactionResponse)(response)
  })
```

### Integration with Database Libraries

```typescript
import { BigDecimal, Effect, pipe } from "effect"

// Database entity with BigDecimal fields
interface AccountEntity {
  readonly id: string
  readonly balance: string // Stored as string in database
  readonly creditLimit: string
  readonly interestRate: string
}

// Convert database entity to domain model
const fromDatabaseEntity = (entity: AccountEntity) =>
  Effect.gen(function* () {
    const balance = yield* pipe(
      BigDecimal.fromString(entity.balance),
      Effect.fromOption(() => `Invalid balance: ${entity.balance}`)
    )
    
    const creditLimit = yield* pipe(
      BigDecimal.fromString(entity.creditLimit),
      Effect.fromOption(() => `Invalid credit limit: ${entity.creditLimit}`)
    )
    
    const interestRate = yield* pipe(
      BigDecimal.fromString(entity.interestRate),
      Effect.fromOption(() => `Invalid interest rate: ${entity.interestRate}`)
    )
    
    return {
      id: entity.id,
      balance,
      creditLimit,
      interestRate
    }
  })

// Convert domain model to database entity
const toDatabaseEntity = (account: {
  id: string
  balance: BigDecimal.BigDecimal
  creditLimit: BigDecimal.BigDecimal
  interestRate: BigDecimal.BigDecimal
}): AccountEntity => ({
  id: account.id,
  balance: BigDecimal.format(account.balance),
  creditLimit: BigDecimal.format(account.creditLimit),
  interestRate: BigDecimal.format(account.interestRate)
})

// Repository pattern with BigDecimal conversion
const createAccountRepository = (db: Database) => ({
  findById: (id: string) =>
    Effect.gen(function* () {
      const entity = yield* db.findById(id)
      return yield* fromDatabaseEntity(entity)
    }),
    
  save: (account: Account) =>
    Effect.gen(function* () {
      const entity = toDatabaseEntity(account)
      yield* db.save(entity)
      return account
    }),
    
  updateBalance: (id: string, newBalance: BigDecimal.BigDecimal) =>
    Effect.gen(function* () {
      const account = yield* this.findById(id)
      const updated = { ...account, balance: newBalance }
      return yield* this.save(updated)
    })
})
```

### Integration with JSON APIs

```typescript
import { BigDecimal, Effect, Array as Arr, pipe } from "effect"

// JSON serialization helpers
const BigDecimalJson = {
  stringify: (value: BigDecimal.BigDecimal): string =>
    BigDecimal.format(value),
    
  parse: (json: string): Effect.Effect<BigDecimal.BigDecimal, string> =>
    pipe(
      BigDecimal.fromString(json),
      Effect.fromOption(() => `Invalid BigDecimal JSON: ${json}`)
    )
}

// API response handling
const parseFinancialApiResponse = (response: unknown) =>
  Effect.gen(function* () {
    if (typeof response !== 'object' || !response) {
      return yield* Effect.fail("Invalid response format")
    }
    
    const data = response as Record<string, unknown>
    
    // Parse financial fields
    const price = data.price ? yield* BigDecimalJson.parse(String(data.price)) : BigDecimal.fromBigInt(0n)
    const volume = data.volume ? yield* BigDecimalJson.parse(String(data.volume)) : BigDecimal.fromBigInt(0n)
    const marketCap = data.marketCap ? yield* BigDecimalJson.parse(String(data.marketCap)) : BigDecimal.fromBigInt(0n)
    
    return {
      symbol: String(data.symbol || ''),
      price,
      volume,
      marketCap,
      lastUpdated: new Date(String(data.lastUpdated || ''))
    }
  })

// Batch processing API responses
const processBatchApiResponse = (responses: readonly unknown[]) =>
  Effect.gen(function* () {
    const parsed = yield* Effect.all(
      responses.map(parseFinancialApiResponse),
      { concurrency: 10 }
    )
    
    const totalVolume = BigDecimal.sumAll(parsed.map(p => p.volume))
    const averagePrice = pipe(
      BigDecimal.sumAll(parsed.map(p => p.price)),
      sum => BigDecimal.divide(sum, BigDecimal.fromBigInt(BigInt(parsed.length))),
      Option.getOrElse(() => BigDecimal.fromBigInt(0n))
    )
    
    return {
      securities: parsed,
      aggregates: {
        totalVolume,
        averagePrice,
        count: parsed.length
      }
    }
  })
```

## Conclusion

BigDecimal provides precise decimal arithmetic essential for financial applications, scientific calculations, and any domain where floating-point errors are unacceptable. Unlike JavaScript's native number type, BigDecimal ensures mathematical correctness through arbitrary-precision arithmetic.

Key benefits:
- **Precision Guarantee**: Eliminates floating-point rounding errors common in financial calculations
- **Type Safety**: Integration with Effect's type system prevents runtime errors and ensures safe operations
- **Composability**: Functional operations chain together seamlessly with other Effect modules
- **Scale Control**: Explicit precision management for different calculation contexts
- **Safe Operations**: Division and other error-prone operations return Option types for graceful error handling

BigDecimal is essential when mathematical accuracy directly impacts business logic, regulatory compliance, or scientific validity. Use it whenever precision matters more than raw computational speed.