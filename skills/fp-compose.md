---
name: Function Composition - Building from Small Pieces
description: Practical patterns for composing functions in TypeScript using pipe, flow, and functional design principles
version: 1.0.0
author: Claude
tags:
  - functional-programming
  - typescript
  - composition
  - pipe
  - flow
  - fp-ts
  - practical-patterns
---

# Function Composition - Building from Small Pieces

The core idea is simple:

```typescript
pipe(data, fn1, fn2, fn3) === fn3(fn2(fn1(data)))
```

That's it. No category theory needed. Just chain functions together, left to right.

---

## 1. pipe() - Your New Best Friend

### The Basic Pattern

```typescript
import { pipe } from 'fp-ts/function'

// Instead of nested calls:
const result = formatOutput(calculateTotal(validateInput(parseData(rawInput))))

// Write it as a pipeline:
const result = pipe(
  rawInput,
  parseData,
  validateInput,
  calculateTotal,
  formatOutput
)
```

### Why This Matters

**Before (nested calls):**
```typescript
// Read from inside out, right to left - confusing
const userName = capitalize(trim(getProperty('name')(user)))
```

**After (pipe):**
```typescript
// Read top to bottom, left to right - natural
const userName = pipe(
  user,
  getProperty('name'),
  trim,
  capitalize
)
```

### Real Example: Processing User Input

```typescript
import { pipe } from 'fp-ts/function'

interface UserInput {
  email: string
  name: string
  age: string
}

interface CleanUser {
  email: string
  name: string
  age: number
}

// Small, focused functions
const trimEmail = (input: UserInput): UserInput => ({
  ...input,
  email: input.email.trim().toLowerCase()
})

const trimName = (input: UserInput): UserInput => ({
  ...input,
  name: input.name.trim()
})

const parseAge = (input: UserInput): CleanUser => ({
  ...input,
  age: parseInt(input.age, 10) || 0
})

// Compose them
const cleanUserInput = (raw: UserInput): CleanUser =>
  pipe(raw, trimEmail, trimName, parseAge)

// Use it
cleanUserInput({ email: '  ALICE@EMAIL.COM  ', name: '  Alice  ', age: '30' })
// { email: 'alice@email.com', name: 'Alice', age: 30 }
```

---

## 2. Building Reusable Utilities

The key is making small functions that do one thing well.

### String Utilities

```typescript
import { pipe, flow } from 'fp-ts/function'

// Basic building blocks
const trim = (s: string): string => s.trim()
const lowercase = (s: string): string => s.toLowerCase()
const uppercase = (s: string): string => s.toUpperCase()
const replace = (pattern: RegExp, replacement: string) =>
  (s: string): string => s.replace(pattern, replacement)
const prefix = (pre: string) => (s: string): string => `${pre}${s}`
const suffix = (suf: string) => (s: string): string => `${s}${suf}`

// Combine them into useful utilities
const slugify = flow(
  trim,
  lowercase,
  replace(/\s+/g, '-'),
  replace(/[^a-z0-9-]/g, '')
)

const titleCase = flow(
  trim,
  lowercase,
  replace(/\b\w/g, c => c.toUpperCase())
)

const kebabCase = flow(
  trim,
  replace(/([a-z])([A-Z])/g, '$1-$2'),
  lowercase,
  replace(/\s+/g, '-')
)

// Use them
slugify('  Hello World! ')     // 'hello-world'
titleCase('hello world')       // 'Hello World'
kebabCase('myVariableName')    // 'my-variable-name'
```

### Number Utilities

```typescript
const clamp = (min: number, max: number) =>
  (n: number): number => Math.max(min, Math.min(max, n))

const round = (decimals: number) =>
  (n: number): number => Math.round(n * 10 ** decimals) / 10 ** decimals

const multiply = (factor: number) =>
  (n: number): number => n * factor

const add = (amount: number) =>
  (n: number): number => n + amount

// Combine for specific use cases
const toPercentage = flow(
  multiply(100),
  round(1),
  suffix('%')
)

const formatPrice = flow(
  round(2),
  n => n.toFixed(2),
  prefix('$')
)

const normalizeScore = flow(
  clamp(0, 100),
  round(0)
)

// Use them
toPercentage(0.8567)  // '85.7%'
formatPrice(19.999)   // '$20.00'
normalizeScore(105)   // 100
```

### Array Utilities

```typescript
import * as A from 'fp-ts/Array'
import { pipe, flow } from 'fp-ts/function'

// Property accessors
const prop = <T, K extends keyof T>(key: K) =>
  (obj: T): T[K] => obj[key]

// Predicates
const isNotNull = <T>(value: T | null | undefined): value is T =>
  value != null

const hasLength = (min: number) =>
  (arr: readonly unknown[]): boolean => arr.length >= min

// Combining filters and maps
interface Product {
  id: string
  name: string
  price: number
  inStock: boolean
}

const getInStockProductNames = flow(
  A.filter((p: Product) => p.inStock),
  A.map(prop('name'))
)

const getTotalValue = flow(
  A.map((p: Product) => p.price),
  A.reduce(0, (acc, price) => acc + price)
)

const getProductsSortedByPrice = flow(
  A.sort<Product>((a, b) => a.price - b.price)
)
```

---

## 3. Data-Last for Flexibility

### Why Argument Order Matters

```typescript
// Data-first: Hard to compose
const map1 = <A, B>(arr: A[], fn: (a: A) => B): B[] => arr.map(fn)

// Can't easily create reusable functions:
const doubleAll = (arr: number[]) => map1(arr, n => n * 2)  // Must wrap

// Data-last: Easy to compose
const map2 = <A, B>(fn: (a: A) => B) => (arr: A[]): B[] => arr.map(fn)

// Create reusable functions by partial application:
const doubleAll = map2((n: number) => n * 2)

// Works naturally in pipes:
pipe([1, 2, 3], doubleAll)  // [2, 4, 6]
```

### The Pattern

```typescript
// General rule: configuration first, data last

// Good: Configuration -> Data
const filter = <A>(predicate: (a: A) => boolean) =>
  (arr: A[]): A[] => arr.filter(predicate)

const map = <A, B>(fn: (a: A) => B) =>
  (arr: A[]): B[] => arr.map(fn)

const formatWith = (formatter: Intl.NumberFormat) =>
  (n: number): string => formatter.format(n)

// All work smoothly in pipes:
const processNumbers = flow(
  filter((n: number) => n > 0),
  map(n => n * 2),
  formatWith(new Intl.NumberFormat('en-US'))
)
```

### Converting Data-First APIs

```typescript
// Many built-in methods are data-first (method on object)
// Wrap them to be data-last

// Date formatting
const formatDate = (options: Intl.DateTimeFormatOptions) =>
  (locale: string) =>
    (date: Date): string => date.toLocaleDateString(locale, options)

const formatShortDate = formatDate({ month: 'short', day: 'numeric' })('en-US')

pipe(new Date(), formatShortDate)  // 'Jan 30'

// JSON operations
const parseJSON = <T>() =>
  (str: string): T => JSON.parse(str)

const stringifyJSON = (indent?: number) =>
  <T>(data: T): string => JSON.stringify(data, null, indent)

// Regular expressions
const match = (regex: RegExp) =>
  (str: string): RegExpMatchArray | null => str.match(regex)

const test = (regex: RegExp) =>
  (str: string): boolean => regex.test(str)

const split = (separator: string | RegExp) =>
  (str: string): string[] => str.split(separator)
```

---

## 4. When Composition Helps (And When It Doesn't)

### Good Uses for Composition

**1. Multi-step data transformations:**
```typescript
// Processing API responses
const processApiResponse = flow(
  extractData,
  normalizeFields,
  validateSchema,
  transformForUI
)
```

**2. Building specialized functions:**
```typescript
// Currency formatters from a general formatter
const formatCurrency = (locale: string, currency: string) =>
  (amount: number): string =>
    new Intl.NumberFormat(locale, { style: 'currency', currency }).format(amount)

const formatUSD = formatCurrency('en-US', 'USD')
const formatEUR = formatCurrency('de-DE', 'EUR')
const formatGBP = formatCurrency('en-GB', 'GBP')
```

**3. Validation chains:**
```typescript
const validateEmail = flow(
  trim,
  lowercase,
  (email: string) => email.includes('@') ? email : null
)

const validateUsername = flow(
  trim,
  (name: string) => name.length >= 3 ? name : null
)
```

**4. Event handlers:**
```typescript
const handleFormSubmit = flow(
  preventDefault,
  extractFormData,
  validateForm,
  submitToAPI
)
```

### When NOT to Compose

**1. When a simple function is clearer:**
```typescript
// Overengineered:
const isAdult = flow(
  prop<Person, 'age'>('age'),
  gte(18)
)

// Just write it:
const isAdult = (person: Person): boolean => person.age >= 18
```

**2. When you need multiple inputs:**
```typescript
// Awkward with composition:
const calculateDiscount = (price: number, discountPercent: number): number =>
  pipe(
    price,
    multiply(1 - discountPercent / 100)
  )
// The discountPercent is awkwardly captured

// Just use a regular function:
const calculateDiscount = (price: number, discountPercent: number): number =>
  price * (1 - discountPercent / 100)
```

**3. When debugging becomes hard:**
```typescript
// If you can't figure out what's happening:
const mysteryPipeline = flow(fn1, fn2, fn3, fn4, fn5, fn6, fn7)

// Break it up and name the stages:
const parse = flow(fn1, fn2)
const validate = flow(fn3, fn4)
const transform = flow(fn5, fn6, fn7)

// Or just use a regular function with intermediate variables:
const processData = (input: Input): Output => {
  const parsed = parse(input)
  const validated = validate(parsed)
  const transformed = transform(validated)
  return transformed
}
```

**4. When the team doesn't know the pattern:**
```typescript
// If your team isn't familiar with FP:
const result = pipe(
  data,
  A.filter(isActive),
  A.map(getName),
  A.sort(ordString.compare)
)

// Consider the more familiar version:
const result = data
  .filter(isActive)
  .map(getName)
  .sort((a, b) => a.localeCompare(b))
```

### Decision Guide

| Situation | Use Composition? |
|-----------|------------------|
| Multi-step transformation | Yes |
| Building reusable utilities | Yes |
| Single operation | No |
| Multiple unrelated inputs | No |
| Complex branching logic | Maybe not |
| Team unfamiliar with FP | Start simple |

---

## 5. Debugging Pipelines

### The trace Function

```typescript
// Simple debug helper
const trace = <A>(label: string) =>
  (value: A): A => {
    console.log(`${label}:`, value)
    return value
  }

// Use it in pipelines
const result = pipe(
  input,
  trace('input'),
  step1,
  trace('after step1'),
  step2,
  trace('after step2'),
  step3,
  trace('final')
)
```

### Conditional Tracing

```typescript
// Only trace in development
const traceIf = (enabled: boolean) =>
  <A>(label: string) =>
    (value: A): A => {
      if (enabled) console.log(`${label}:`, value)
      return value
    }

const debug = traceIf(process.env.NODE_ENV === 'development')

pipe(
  data,
  debug('input'),
  transform,
  debug('output')
)
```

### Structured Logging

```typescript
// More sophisticated tracing
interface TraceOptions {
  label: string
  transform?: (value: unknown) => unknown
  condition?: (value: unknown) => boolean
}

const traceWith = (options: TraceOptions) =>
  <A>(value: A): A => {
    if (!options.condition || options.condition(value)) {
      const output = options.transform ? options.transform(value) : value
      console.log(`[${options.label}]`, output)
    }
    return value
  }

// Use it
pipe(
  users,
  traceWith({ label: 'users', transform: arr => `count: ${arr.length}` }),
  A.filter(isActive),
  traceWith({ label: 'active', transform: arr => `count: ${arr.length}` })
)
```

### Breakpoint Debugging

```typescript
// Insert a breakpoint
const breakpoint = <A>(value: A): A => {
  debugger  // Execution pauses here
  return value
}

pipe(
  data,
  step1,
  breakpoint,  // Inspect value here
  step2
)
```

### Type Checking Mid-Pipeline

```typescript
// Verify types are what you expect
const assertType = <Expected>() =>
  <Actual extends Expected>(value: Actual): Actual => value

pipe(
  data,
  parseInput,
  assertType<{ name: string; age: number }>(),  // TypeScript error if wrong
  formatOutput
)
```

---

## Practical Patterns

### Pattern 1: Data Processing Pipeline

```typescript
import { pipe, flow } from 'fp-ts/function'
import * as A from 'fp-ts/Array'
import * as O from 'fp-ts/Option'

interface RawRecord {
  id: string
  timestamp: string
  value: string
  status: string
}

interface ProcessedRecord {
  id: string
  date: Date
  value: number
  isActive: boolean
}

// Individual processing steps
const parseTimestamp = (r: RawRecord) => ({
  ...r,
  date: new Date(r.timestamp)
})

const parseValue = (r: { id: string; date: Date; value: string; status: string }) => ({
  id: r.id,
  date: r.date,
  value: parseFloat(r.value) || 0,
  isActive: r.status === 'active'
})

const isValid = (r: ProcessedRecord): boolean =>
  !isNaN(r.date.getTime()) && r.value >= 0

const sortByDate = A.sort<ProcessedRecord>((a, b) =>
  a.date.getTime() - b.date.getTime()
)

// Compose into a pipeline
const processRecords = flow(
  A.map(parseTimestamp),
  A.map(parseValue),
  A.filter(isValid),
  sortByDate
)

// Use it
const result = processRecords(rawData)
```

### Pattern 2: Creating Specialized Functions

```typescript
// General HTTP client
interface RequestConfig {
  baseUrl: string
  headers: Record<string, string>
}

const createFetcher = (config: RequestConfig) =>
  (endpoint: string) =>
    <T>(): Promise<T> =>
      fetch(`${config.baseUrl}${endpoint}`, { headers: config.headers })
        .then(r => r.json())

// Create specialized fetchers
const apiConfig = {
  baseUrl: 'https://api.example.com',
  headers: { 'Authorization': 'Bearer token123' }
}

const apiFetch = createFetcher(apiConfig)

// Even more specialized
const fetchUsers = apiFetch('/users')<User[]>
const fetchProducts = apiFetch('/products')<Product[]>
const fetchOrders = apiFetch('/orders')<Order[]>
```

### Pattern 3: Composing Validators

```typescript
import * as E from 'fp-ts/Either'
import { pipe } from 'fp-ts/function'

type ValidationError = string
type Validator<T> = (value: T) => E.Either<ValidationError, T>

// Basic validators
const nonEmpty: Validator<string> = (s) =>
  s.length > 0
    ? E.right(s)
    : E.left('Value cannot be empty')

const minLength = (min: number): Validator<string> => (s) =>
  s.length >= min
    ? E.right(s)
    : E.left(`Must be at least ${min} characters`)

const maxLength = (max: number): Validator<string> => (s) =>
  s.length <= max
    ? E.right(s)
    : E.left(`Must be at most ${max} characters`)

const matches = (pattern: RegExp, message: string): Validator<string> => (s) =>
  pattern.test(s)
    ? E.right(s)
    : E.left(message)

// Compose validators
const validateUsername = (input: string): E.Either<ValidationError, string> =>
  pipe(
    E.right(input),
    E.flatMap(nonEmpty),
    E.flatMap(minLength(3)),
    E.flatMap(maxLength(20)),
    E.flatMap(matches(/^[a-zA-Z0-9_]+$/, 'Only letters, numbers, and underscores'))
  )

const validateEmail = (input: string): E.Either<ValidationError, string> =>
  pipe(
    E.right(input),
    E.flatMap(nonEmpty),
    E.flatMap(matches(/^[^\s@]+@[^\s@]+\.[^\s@]+$/, 'Invalid email format'))
  )
```

### Pattern 4: Chaining API Transformations

```typescript
import * as TE from 'fp-ts/TaskEither'
import { pipe } from 'fp-ts/function'

interface ApiUser { id: number; name: string; email: string }
interface ApiPosts { userId: number; title: string; body: string }[]
interface UserWithPosts { user: ApiUser; posts: ApiPosts; postCount: number }

// API calls as TaskEither
const fetchUser = (id: number): TE.TaskEither<Error, ApiUser> =>
  TE.tryCatch(
    () => fetch(`/api/users/${id}`).then(r => r.json()),
    (e) => new Error(String(e))
  )

const fetchUserPosts = (userId: number): TE.TaskEither<Error, ApiPosts> =>
  TE.tryCatch(
    () => fetch(`/api/users/${userId}/posts`).then(r => r.json()),
    (e) => new Error(String(e))
  )

// Combine into a pipeline
const getUserWithPosts = (userId: number): TE.TaskEither<Error, UserWithPosts> =>
  pipe(
    fetchUser(userId),
    TE.flatMap(user =>
      pipe(
        fetchUserPosts(user.id),
        TE.map(posts => ({
          user,
          posts,
          postCount: posts.length
        }))
      )
    )
  )

// Usage
const program = pipe(
  getUserWithPosts(123),
  TE.map(data => console.log(`${data.user.name} has ${data.postCount} posts`)),
  TE.mapLeft(error => console.error('Failed:', error.message))
)

program()  // Execute the async operation
```

### Pattern 5: Configurable Utilities

```typescript
// Configuration-driven formatting
interface FormatConfig {
  locale: string
  currency: string
  dateFormat: Intl.DateTimeFormatOptions
  numberFormat: Intl.NumberFormatOptions
}

const createFormatter = (config: FormatConfig) => ({
  currency: (amount: number): string =>
    new Intl.NumberFormat(config.locale, {
      style: 'currency',
      currency: config.currency
    }).format(amount),

  date: (date: Date): string =>
    date.toLocaleDateString(config.locale, config.dateFormat),

  number: (n: number): string =>
    new Intl.NumberFormat(config.locale, config.numberFormat).format(n),

  percent: (n: number): string =>
    new Intl.NumberFormat(config.locale, { style: 'percent' }).format(n)
})

// Create region-specific formatters
const usFormatter = createFormatter({
  locale: 'en-US',
  currency: 'USD',
  dateFormat: { month: 'short', day: 'numeric', year: 'numeric' },
  numberFormat: { maximumFractionDigits: 2 }
})

const euFormatter = createFormatter({
  locale: 'de-DE',
  currency: 'EUR',
  dateFormat: { day: '2-digit', month: '2-digit', year: 'numeric' },
  numberFormat: { maximumFractionDigits: 2 }
})

// Use them
usFormatter.currency(1234.56)  // '$1,234.56'
euFormatter.currency(1234.56)  // '1.234,56 EUR'
```

---

## Quick Reference

### pipe vs flow

```typescript
// pipe: Start with a value, transform immediately
const result = pipe(value, fn1, fn2, fn3)

// flow: Create a reusable function
const transform = flow(fn1, fn2, fn3)
const result = transform(value)
```

### Creating Composable Functions

```typescript
// Data-last for pipes
const filter = <A>(pred: (a: A) => boolean) => (arr: A[]): A[] => arr.filter(pred)
const map = <A, B>(fn: (a: A) => B) => (arr: A[]): B[] => arr.map(fn)

// Configuration first, data last
const format = (options: Options) => (value: Value): string => ...
```

### Debug Helpers

```typescript
const trace = <A>(label: string) => (a: A): A => { console.log(label, a); return a }
const breakpoint = <A>(a: A): A => { debugger; return a }
```

### When to Use

| Scenario | Approach |
|----------|----------|
| Transform data through steps | `pipe(data, step1, step2, ...)` |
| Create reusable transform | `flow(step1, step2, ...)` |
| Simple single operation | Regular function |
| Multiple unrelated inputs | Regular function |
| Team learning FP | Start with `pipe`, add `flow` later |

---

## Summary

Function composition is about building complex behavior from simple pieces:

1. **Start with pipe** - Chain operations on data, read top to bottom
2. **Extract reusable utilities** - Small functions that do one thing well
3. **Use data-last** - Configuration first, data last enables composition
4. **Know when to stop** - Not everything needs to be composed
5. **Debug with trace** - Insert logging without breaking the pipeline

The goal isn't to compose everything. The goal is clearer, more maintainable code. Use composition when it helps, skip it when it doesn't.
