---
name: fp-ts Algebraic Data Types and Type Classes
description: Product types, sum types, semigroups, monoids, Eq, Ord, and building custom type class instances for domain modeling in TypeScript
version: 1.0.0
author: kadu
tags:
  - fp-ts
  - functional-programming
  - typescript
  - algebraic-data-types
  - type-classes
  - semigroup
  - monoid
  - eq
  - ord
  - domain-modeling
---

# fp-ts Algebraic Data Types and Type Classes

This skill covers algebraic data types (ADTs) and type classes in fp-ts for robust domain modeling in TypeScript.

## Core Concepts

### What are Algebraic Data Types?

Algebraic Data Types (ADTs) are composite types formed by combining other types. There are two fundamental kinds:

- **Product Types**: Combine types with AND (tuples, records) - "A user HAS a name AND an email"
- **Sum Types**: Combine types with OR (discriminated unions) - "A result IS either success OR failure"

### What are Type Classes?

Type classes define behavior that can be implemented for different types. In fp-ts, they're represented as interfaces with instances for specific types:

- **Eq**: Equality comparison
- **Ord**: Ordering/comparison
- **Semigroup**: Combining two values of the same type
- **Monoid**: Semigroup with an identity element

## Product Types

Product types represent values that contain ALL of their component parts simultaneously.

### Tuples

Tuples are fixed-length arrays with typed positions.

```typescript
import { pipe } from 'fp-ts/function'
import * as T from 'fp-ts/Tuple'

// Basic tuple type
type Point2D = readonly [number, number]
type NamedValue = readonly [string, number]

const point: Point2D = [10, 20]
const item: NamedValue = ['score', 100]

// Accessing tuple elements
const [x, y] = point
const [label, value] = item

// Tuple operations from fp-ts
const first = T.fst(point)   // 10
const second = T.snd(point)  // 20

// Map over the first element
const doubled = T.mapFst((n: number) => n * 2)(point) // [20, 20]

// Map over the second element
const incremented = T.mapSnd((n: number) => n + 1)(point) // [10, 21]

// Bimap: transform both elements
const transformed = T.bimap(
  (n: number) => n.toString(),
  (n: number) => n * 2
)(point) // ['20', 20]
```

### Records (Object Types)

Records are the most common product type in TypeScript.

```typescript
// Domain model as a product type
interface User {
  readonly id: string
  readonly email: string
  readonly name: string
  readonly createdAt: Date
}

// All fields must be present - this is what makes it a "product"
const user: User = {
  id: '123',
  email: 'john@example.com',
  name: 'John Doe',
  createdAt: new Date()
}

// Nested product types
interface Order {
  readonly id: string
  readonly customer: User
  readonly items: ReadonlyArray<OrderItem>
  readonly shipping: ShippingAddress
  readonly billing: BillingInfo
}

interface OrderItem {
  readonly productId: string
  readonly quantity: number
  readonly unitPrice: number
}

interface ShippingAddress {
  readonly street: string
  readonly city: string
  readonly postalCode: string
  readonly country: string
}

interface BillingInfo {
  readonly method: PaymentMethod
  readonly cardLast4?: string
}
```

### When to Use Product Types

Use product types when:

- An entity requires ALL pieces of data to be valid
- The data naturally belongs together
- You're modeling database entities or API payloads
- Components are always present together

```typescript
// Good: All fields are required for a valid config
interface DatabaseConfig {
  readonly host: string
  readonly port: number
  readonly database: string
  readonly username: string
  readonly password: string
}

// Good: Coordinate always needs both x and y
interface Coordinate {
  readonly x: number
  readonly y: number
}

// Good: Money needs both amount and currency
interface Money {
  readonly amount: number
  readonly currency: string
}
```

## Sum Types (Discriminated Unions)

Sum types represent values that can be ONE OF several variants.

### Basic Discriminated Unions

```typescript
// A payment can be ONE OF these types
type PaymentMethod =
  | { readonly _tag: 'CreditCard'; readonly cardNumber: string; readonly expiry: string }
  | { readonly _tag: 'PayPal'; readonly email: string }
  | { readonly _tag: 'BankTransfer'; readonly accountNumber: string; readonly routingNumber: string }
  | { readonly _tag: 'Crypto'; readonly walletAddress: string; readonly network: string }

// Creating values
const creditCard: PaymentMethod = {
  _tag: 'CreditCard',
  cardNumber: '4111111111111111',
  expiry: '12/25'
}

const paypal: PaymentMethod = {
  _tag: 'PayPal',
  email: 'user@example.com'
}
```

### Pattern Matching with fold/match

```typescript
import { pipe } from 'fp-ts/function'

// Define a fold function for exhaustive pattern matching
const foldPaymentMethod = <R>(handlers: {
  CreditCard: (cardNumber: string, expiry: string) => R
  PayPal: (email: string) => R
  BankTransfer: (accountNumber: string, routingNumber: string) => R
  Crypto: (walletAddress: string, network: string) => R
}) => (payment: PaymentMethod): R => {
  switch (payment._tag) {
    case 'CreditCard':
      return handlers.CreditCard(payment.cardNumber, payment.expiry)
    case 'PayPal':
      return handlers.PayPal(payment.email)
    case 'BankTransfer':
      return handlers.BankTransfer(payment.accountNumber, payment.routingNumber)
    case 'Crypto':
      return handlers.Crypto(payment.walletAddress, payment.network)
  }
}

// Usage
const getPaymentLabel = foldPaymentMethod({
  CreditCard: (num, _) => `Card ending in ${num.slice(-4)}`,
  PayPal: (email) => `PayPal: ${email}`,
  BankTransfer: (acc, _) => `Bank: ${acc.slice(-4)}`,
  Crypto: (wallet, network) => `${network}: ${wallet.slice(0, 8)}...`
})

const label = getPaymentLabel(creditCard) // "Card ending in 1111"
```

### Smart Constructors for Sum Types

```typescript
// Define the union
type RemoteData<E, A> =
  | { readonly _tag: 'NotAsked' }
  | { readonly _tag: 'Loading' }
  | { readonly _tag: 'Failure'; readonly error: E }
  | { readonly _tag: 'Success'; readonly data: A }

// Smart constructors
const notAsked: RemoteData<never, never> = { _tag: 'NotAsked' }
const loading: RemoteData<never, never> = { _tag: 'Loading' }
const failure = <E>(error: E): RemoteData<E, never> => ({ _tag: 'Failure', error })
const success = <A>(data: A): RemoteData<never, A> => ({ _tag: 'Success', data })

// Type guards
const isNotAsked = <E, A>(rd: RemoteData<E, A>): rd is { _tag: 'NotAsked' } =>
  rd._tag === 'NotAsked'
const isLoading = <E, A>(rd: RemoteData<E, A>): rd is { _tag: 'Loading' } =>
  rd._tag === 'Loading'
const isFailure = <E, A>(rd: RemoteData<E, A>): rd is { _tag: 'Failure'; error: E } =>
  rd._tag === 'Failure'
const isSuccess = <E, A>(rd: RemoteData<E, A>): rd is { _tag: 'Success'; data: A } =>
  rd._tag === 'Success'

// Fold for exhaustive matching
const fold = <E, A, R>(handlers: {
  NotAsked: () => R
  Loading: () => R
  Failure: (error: E) => R
  Success: (data: A) => R
}) => (rd: RemoteData<E, A>): R => {
  switch (rd._tag) {
    case 'NotAsked': return handlers.NotAsked()
    case 'Loading': return handlers.Loading()
    case 'Failure': return handlers.Failure(rd.error)
    case 'Success': return handlers.Success(rd.data)
  }
}

// Usage in React
const UserProfile = ({ data }: { data: RemoteData<Error, User> }) =>
  pipe(
    data,
    fold({
      NotAsked: () => <button>Load Profile</button>,
      Loading: () => <Spinner />,
      Failure: (err) => <ErrorMessage error={err} />,
      Success: (user) => <ProfileCard user={user} />
    })
  )
```

### When to Use Sum Types

Use sum types when:

- A value can be in different states
- Different variants have different data shapes
- You need exhaustive handling of all cases
- Modeling state machines or workflows

```typescript
// Good: Order status is mutually exclusive
type OrderStatus =
  | { readonly _tag: 'Pending' }
  | { readonly _tag: 'Confirmed'; readonly confirmedAt: Date }
  | { readonly _tag: 'Shipped'; readonly trackingNumber: string; readonly shippedAt: Date }
  | { readonly _tag: 'Delivered'; readonly deliveredAt: Date }
  | { readonly _tag: 'Cancelled'; readonly reason: string; readonly cancelledAt: Date }

// Good: API response states
type ApiResponse<T> =
  | { readonly _tag: 'Idle' }
  | { readonly _tag: 'Loading' }
  | { readonly _tag: 'Error'; readonly message: string; readonly code: number }
  | { readonly _tag: 'Success'; readonly data: T }

// Good: Form field validation state
type FieldState<T> =
  | { readonly _tag: 'Pristine'; readonly value: T }
  | { readonly _tag: 'Valid'; readonly value: T }
  | { readonly _tag: 'Invalid'; readonly value: T; readonly errors: ReadonlyArray<string> }
```

## Semigroups

A Semigroup defines how to combine two values of the same type. It must satisfy the associativity law: `combine(combine(a, b), c) === combine(a, combine(b, c))`

### Basic Semigroup Usage

```typescript
import * as S from 'fp-ts/Semigroup'
import * as N from 'fp-ts/number'
import * as Str from 'fp-ts/string'
import { pipe } from 'fp-ts/function'

// Built-in semigroups
const sumResult = S.concatAll(N.SemigroupSum)(0)([1, 2, 3, 4]) // 10
const productResult = S.concatAll(N.SemigroupProduct)(1)([1, 2, 3, 4]) // 24
const stringResult = S.concatAll(Str.Semigroup)('')(['Hello', ' ', 'World']) // 'Hello World'

// Combining two values
const sum = N.SemigroupSum.concat(5, 3) // 8
const product = N.SemigroupProduct.concat(5, 3) // 15
```

### Custom Semigroup Instances

```typescript
import * as S from 'fp-ts/Semigroup'
import { pipe } from 'fp-ts/function'

// Semigroup for taking the first value
const first = <A>(): S.Semigroup<A> => ({
  concat: (x, _) => x
})

// Semigroup for taking the last value
const last = <A>(): S.Semigroup<A> => ({
  concat: (_, y) => y
})

// Semigroup for max value
const max: S.Semigroup<number> = {
  concat: (x, y) => Math.max(x, y)
}

// Semigroup for min value
const min: S.Semigroup<number> = {
  concat: (x, y) => Math.min(x, y)
}
```

### Practical Example: Merging Configs

```typescript
import * as S from 'fp-ts/Semigroup'
import { pipe } from 'fp-ts/function'

interface ServerConfig {
  readonly host: string
  readonly port: number
  readonly timeout: number
  readonly retries: number
  readonly features: ReadonlyArray<string>
}

// Semigroup that merges configs (later values override)
const ServerConfigSemigroup: S.Semigroup<ServerConfig> = S.struct({
  host: S.last<string>(),           // Last value wins
  port: S.last<number>(),           // Last value wins
  timeout: S.max(N.Ord),            // Take maximum timeout
  retries: S.max(N.Ord),            // Take maximum retries
  features: {                        // Concatenate arrays
    concat: (x, y) => [...new Set([...x, ...y])]
  }
})

// Default config
const defaultConfig: ServerConfig = {
  host: 'localhost',
  port: 3000,
  timeout: 5000,
  retries: 3,
  features: ['logging']
}

// Environment-specific overrides
const prodOverrides: ServerConfig = {
  host: 'api.example.com',
  port: 443,
  timeout: 10000,
  retries: 5,
  features: ['monitoring', 'caching']
}

// Merge configs
const finalConfig = ServerConfigSemigroup.concat(defaultConfig, prodOverrides)
// {
//   host: 'api.example.com',
//   port: 443,
//   timeout: 10000,
//   retries: 5,
//   features: ['logging', 'monitoring', 'caching']
// }

// Merge multiple configs
const configs = [defaultConfig, devOverrides, envOverrides, cliOverrides]
const mergedConfig = S.concatAll(ServerConfigSemigroup)(defaultConfig)(configs)
```

### Semigroup for Optional Values

```typescript
import * as S from 'fp-ts/Semigroup'
import * as O from 'fp-ts/Option'

// Semigroup for Option that prefers Some values
const getOptionSemigroup = <A>(S: S.Semigroup<A>): S.Semigroup<O.Option<A>> =>
  O.getMonoid(S)

// Or use the built-in
import { getApplySemigroup } from 'fp-ts/Apply'
import { Applicative } from 'fp-ts/Option'

const optionStringSemigroup = getApplySemigroup(Applicative)(Str.Semigroup)

// Example: merging optional configs
interface PartialConfig {
  readonly host?: string
  readonly port?: number
}

const mergePartialConfigs = (a: PartialConfig, b: PartialConfig): PartialConfig => ({
  host: b.host ?? a.host,
  port: b.port ?? a.port
})
```

## Monoids

A Monoid is a Semigroup with an identity element (empty). The empty element satisfies: `concat(empty, a) === a` and `concat(a, empty) === a`

### Basic Monoid Usage

```typescript
import * as M from 'fp-ts/Monoid'
import * as N from 'fp-ts/number'
import * as Str from 'fp-ts/string'
import * as A from 'fp-ts/Array'
import { pipe } from 'fp-ts/function'

// Built-in monoids
const sum = M.concatAll(N.MonoidSum)([1, 2, 3, 4, 5]) // 15
const product = M.concatAll(N.MonoidProduct)([1, 2, 3, 4, 5]) // 120
const concatenated = M.concatAll(Str.Monoid)(['a', 'b', 'c']) // 'abc'

// Empty values
N.MonoidSum.empty // 0
N.MonoidProduct.empty // 1
Str.Monoid.empty // ''
A.getMonoid<number>().empty // []
```

### Custom Monoid Instances

```typescript
import * as M from 'fp-ts/Monoid'
import { pipe } from 'fp-ts/function'

// Monoid for boolean AND
const MonoidAll: M.Monoid<boolean> = {
  concat: (x, y) => x && y,
  empty: true
}

// Monoid for boolean OR
const MonoidAny: M.Monoid<boolean> = {
  concat: (x, y) => x || y,
  empty: false
}

// Usage: check if all validations pass
const allValid = M.concatAll(MonoidAll)([true, true, false, true]) // false
const anyValid = M.concatAll(MonoidAny)([false, false, true, false]) // true
```

### Practical Example: Combining Results

```typescript
import * as M from 'fp-ts/Monoid'
import * as A from 'fp-ts/Array'
import { pipe } from 'fp-ts/function'

// Aggregation result type
interface AggregationResult {
  readonly totalCount: number
  readonly successCount: number
  readonly errorCount: number
  readonly errors: ReadonlyArray<string>
  readonly processingTimeMs: number
}

// Monoid for combining aggregation results
const AggregationResultMonoid: M.Monoid<AggregationResult> = {
  concat: (a, b) => ({
    totalCount: a.totalCount + b.totalCount,
    successCount: a.successCount + b.successCount,
    errorCount: a.errorCount + b.errorCount,
    errors: [...a.errors, ...b.errors],
    processingTimeMs: a.processingTimeMs + b.processingTimeMs
  }),
  empty: {
    totalCount: 0,
    successCount: 0,
    errorCount: 0,
    errors: [],
    processingTimeMs: 0
  }
}

// Process items and combine results
const processItem = (item: Item): AggregationResult => {
  const start = Date.now()
  try {
    doProcessing(item)
    return {
      totalCount: 1,
      successCount: 1,
      errorCount: 0,
      errors: [],
      processingTimeMs: Date.now() - start
    }
  } catch (e) {
    return {
      totalCount: 1,
      successCount: 0,
      errorCount: 1,
      errors: [e instanceof Error ? e.message : 'Unknown error'],
      processingTimeMs: Date.now() - start
    }
  }
}

// Combine all results
const processAll = (items: ReadonlyArray<Item>): AggregationResult =>
  pipe(
    items,
    A.map(processItem),
    M.concatAll(AggregationResultMonoid)
  )
```

### Monoid for Statistics

```typescript
import * as M from 'fp-ts/Monoid'

interface Statistics {
  readonly count: number
  readonly sum: number
  readonly min: number
  readonly max: number
}

const StatisticsMonoid: M.Monoid<Statistics> = {
  concat: (a, b) => ({
    count: a.count + b.count,
    sum: a.sum + b.sum,
    min: Math.min(a.min, b.min),
    max: Math.max(a.max, b.max)
  }),
  empty: {
    count: 0,
    sum: 0,
    min: Infinity,
    max: -Infinity
  }
}

// Create statistics from a single value
const fromNumber = (n: number): Statistics => ({
  count: 1,
  sum: n,
  min: n,
  max: n
})

// Calculate statistics for an array
const calculateStats = (numbers: ReadonlyArray<number>): Statistics =>
  pipe(
    numbers,
    A.map(fromNumber),
    M.concatAll(StatisticsMonoid)
  )

const stats = calculateStats([5, 2, 8, 1, 9])
// { count: 5, sum: 25, min: 1, max: 9 }

// Can also compute average
const average = (s: Statistics): number =>
  s.count === 0 ? 0 : s.sum / s.count
```

## Eq Type Class

Eq defines equality comparison for types.

### Basic Eq Usage

```typescript
import * as Eq from 'fp-ts/Eq'
import * as N from 'fp-ts/number'
import * as Str from 'fp-ts/string'
import * as A from 'fp-ts/Array'
import { pipe } from 'fp-ts/function'

// Built-in Eq instances
N.Eq.equals(1, 1) // true
Str.Eq.equals('hello', 'hello') // true

// Eq for arrays
const numberArrayEq = A.getEq(N.Eq)
numberArrayEq.equals([1, 2, 3], [1, 2, 3]) // true
numberArrayEq.equals([1, 2, 3], [1, 2]) // false
```

### Custom Eq Instances

```typescript
import * as Eq from 'fp-ts/Eq'
import { pipe } from 'fp-ts/function'

interface User {
  readonly id: string
  readonly email: string
  readonly name: string
}

// Eq by ID only
const UserEqById: Eq.Eq<User> = Eq.contramap((user: User) => user.id)(Str.Eq)

// Eq by all fields
const UserEqFull: Eq.Eq<User> = Eq.struct({
  id: Str.Eq,
  email: Str.Eq,
  name: Str.Eq
})

// Usage
const user1: User = { id: '1', email: 'a@b.com', name: 'Alice' }
const user2: User = { id: '1', email: 'different@b.com', name: 'Alice Updated' }

UserEqById.equals(user1, user2) // true (same ID)
UserEqFull.equals(user1, user2) // false (different email)
```

### Practical Example: Deduplication

```typescript
import * as Eq from 'fp-ts/Eq'
import * as A from 'fp-ts/Array'
import { pipe } from 'fp-ts/function'

interface Product {
  readonly sku: string
  readonly name: string
  readonly price: number
}

const ProductEqBySku: Eq.Eq<Product> = pipe(
  Str.Eq,
  Eq.contramap((p: Product) => p.sku)
)

// Remove duplicates by SKU
const uniqueProducts = A.uniq(ProductEqBySku)

const products: Product[] = [
  { sku: 'A001', name: 'Widget', price: 10 },
  { sku: 'A002', name: 'Gadget', price: 20 },
  { sku: 'A001', name: 'Widget Updated', price: 15 }, // Duplicate SKU
]

const unique = uniqueProducts(products)
// [{ sku: 'A001', name: 'Widget', price: 10 }, { sku: 'A002', name: 'Gadget', price: 20 }]

// Check if array contains element
const hasProduct = A.elem(ProductEqBySku)
hasProduct({ sku: 'A001', name: '', price: 0 })(products) // true
```

### Case-Insensitive String Eq

```typescript
import * as Eq from 'fp-ts/Eq'

const EqCaseInsensitive: Eq.Eq<string> = {
  equals: (x, y) => x.toLowerCase() === y.toLowerCase()
}

EqCaseInsensitive.equals('Hello', 'hello') // true
EqCaseInsensitive.equals('WORLD', 'world') // true

// Use with user emails
const UserEqByEmail: Eq.Eq<User> = pipe(
  EqCaseInsensitive,
  Eq.contramap((u: User) => u.email)
)
```

## Ord Type Class

Ord extends Eq with ordering/comparison capabilities.

### Basic Ord Usage

```typescript
import * as Ord from 'fp-ts/Ord'
import * as N from 'fp-ts/number'
import * as Str from 'fp-ts/string'
import * as A from 'fp-ts/Array'
import { pipe } from 'fp-ts/function'

// Built-in Ord instances
Ord.lt(N.Ord)(1, 2) // true (1 < 2)
Ord.gt(N.Ord)(5, 3) // true (5 > 3)
Ord.leq(N.Ord)(2, 2) // true (2 <= 2)
Ord.geq(N.Ord)(3, 2) // true (3 >= 2)

// Sorting
const numbers = [3, 1, 4, 1, 5, 9, 2, 6]
const sorted = A.sort(N.Ord)(numbers) // [1, 1, 2, 3, 4, 5, 6, 9]

// Reverse order
const descending = A.sort(Ord.reverse(N.Ord))(numbers) // [9, 6, 5, 4, 3, 2, 1, 1]

// Min and max
const minimum = Ord.min(N.Ord)(5, 3) // 3
const maximum = Ord.max(N.Ord)(5, 3) // 5
```

### Custom Ord Instances

```typescript
import * as Ord from 'fp-ts/Ord'
import * as A from 'fp-ts/Array'
import { pipe } from 'fp-ts/function'

interface Product {
  readonly name: string
  readonly price: number
  readonly rating: number
  readonly createdAt: Date
}

// Ord by price
const OrdByPrice: Ord.Ord<Product> = pipe(
  N.Ord,
  Ord.contramap((p: Product) => p.price)
)

// Ord by rating (descending - highest first)
const OrdByRatingDesc: Ord.Ord<Product> = pipe(
  N.Ord,
  Ord.contramap((p: Product) => p.rating),
  Ord.reverse
)

// Ord by date (newest first)
const OrdByDateDesc: Ord.Ord<Product> = pipe(
  N.Ord,
  Ord.contramap((p: Product) => p.createdAt.getTime()),
  Ord.reverse
)

// Sort products by price
const sortByPrice = A.sort(OrdByPrice)

// Sort by rating (highest first)
const sortByRating = A.sort(OrdByRatingDesc)
```

### Compound Ordering

```typescript
import * as Ord from 'fp-ts/Ord'
import * as A from 'fp-ts/Array'
import { pipe } from 'fp-ts/function'
import * as M from 'fp-ts/Monoid'

interface Employee {
  readonly department: string
  readonly name: string
  readonly salary: number
  readonly startDate: Date
}

// Primary: department (alphabetical)
// Secondary: salary (descending)
// Tertiary: name (alphabetical)

const OrdByDepartment: Ord.Ord<Employee> = pipe(
  Str.Ord,
  Ord.contramap((e: Employee) => e.department)
)

const OrdBySalaryDesc: Ord.Ord<Employee> = pipe(
  N.Ord,
  Ord.contramap((e: Employee) => e.salary),
  Ord.reverse
)

const OrdByName: Ord.Ord<Employee> = pipe(
  Str.Ord,
  Ord.contramap((e: Employee) => e.name)
)

// Combine orderings with monoid
const EmployeeOrd: Ord.Ord<Employee> = M.concatAll(Ord.getMonoid<Employee>())([
  OrdByDepartment,
  OrdBySalaryDesc,
  OrdByName
])

const employees: Employee[] = [
  { department: 'Engineering', name: 'Alice', salary: 100000, startDate: new Date() },
  { department: 'Engineering', name: 'Bob', salary: 120000, startDate: new Date() },
  { department: 'Sales', name: 'Charlie', salary: 90000, startDate: new Date() },
  { department: 'Engineering', name: 'Diana', salary: 120000, startDate: new Date() },
]

const sorted = A.sort(EmployeeOrd)(employees)
// Engineering: Bob (120k), Diana (120k - alphabetical), Alice (100k)
// Sales: Charlie (90k)
```

### Practical Example: Sorting and Filtering

```typescript
import * as Ord from 'fp-ts/Ord'
import * as A from 'fp-ts/Array'
import * as O from 'fp-ts/Option'
import { pipe } from 'fp-ts/function'

interface Task {
  readonly id: string
  readonly title: string
  readonly priority: 'low' | 'medium' | 'high' | 'critical'
  readonly dueDate: O.Option<Date>
  readonly completed: boolean
}

// Priority ordering (critical > high > medium > low)
const priorityOrder: Record<Task['priority'], number> = {
  critical: 4,
  high: 3,
  medium: 2,
  low: 1
}

const OrdByPriority: Ord.Ord<Task> = pipe(
  N.Ord,
  Ord.contramap((t: Task) => priorityOrder[t.priority]),
  Ord.reverse // Highest priority first
)

// Due date ordering (soonest first, no date last)
const OrdByDueDate: Ord.Ord<Task> = pipe(
  O.getOrd(N.Ord),
  Ord.contramap((t: Task) => pipe(t.dueDate, O.map(d => d.getTime())))
)

// Combined: incomplete first, then by priority, then by due date
const TaskOrd: Ord.Ord<Task> = M.concatAll(Ord.getMonoid<Task>())([
  pipe(
    Ord.trivial, // Boolean doesn't have natural ordering
    Ord.contramap((t: Task) => t.completed ? 1 : 0) // Incomplete (0) before complete (1)
  ),
  OrdByPriority,
  OrdByDueDate
])

// Get top N tasks
const getTopTasks = (n: number) => (tasks: Task[]): Task[] =>
  pipe(
    tasks,
    A.filter(t => !t.completed),
    A.sort(TaskOrd),
    A.takeLeft(n)
  )
```

## Using fp-ts Type Class Instances

### Getting Instances from Modules

```typescript
import * as O from 'fp-ts/Option'
import * as E from 'fp-ts/Either'
import * as A from 'fp-ts/Array'
import * as R from 'fp-ts/Record'
import * as NEA from 'fp-ts/NonEmptyArray'

// Each module exports type class instances

// Option instances
O.Eq        // Eq<Option<A>> given Eq<A>
O.Ord       // Ord<Option<A>> given Ord<A>
O.Semigroup // Semigroup using Apply
O.Monoid    // Monoid with None as empty

// Array instances
A.getEq     // (Eq<A>) => Eq<Array<A>>
A.getOrd    // (Ord<A>) => Ord<Array<A>>
A.getMonoid // <A>() => Monoid<Array<A>>
A.getSemigroup // <A>() => Semigroup<NonEmptyArray<A>>

// Record instances
R.getEq     // Eq for records
R.getMonoid // Monoid for records
```

### Combining Type Classes

```typescript
import * as Eq from 'fp-ts/Eq'
import * as Ord from 'fp-ts/Ord'
import * as S from 'fp-ts/Semigroup'
import * as M from 'fp-ts/Monoid'
import * as O from 'fp-ts/Option'
import { pipe } from 'fp-ts/function'

interface CartItem {
  readonly productId: string
  readonly quantity: number
  readonly price: number
}

// Eq for cart items (by product ID)
const CartItemEq: Eq.Eq<CartItem> = pipe(
  Str.Eq,
  Eq.contramap((item: CartItem) => item.productId)
)

// Semigroup that merges quantities for same product
const CartItemSemigroup: S.Semigroup<CartItem> = {
  concat: (a, b) => ({
    productId: a.productId,
    quantity: a.quantity + b.quantity,
    price: a.price // Keep original price
  })
}

// Merge cart items, combining quantities for duplicates
const mergeCartItems = (items: CartItem[]): CartItem[] => {
  const grouped = items.reduce((acc, item) => {
    const existing = acc.get(item.productId)
    if (existing) {
      acc.set(item.productId, CartItemSemigroup.concat(existing, item))
    } else {
      acc.set(item.productId, item)
    }
    return acc
  }, new Map<string, CartItem>())

  return Array.from(grouped.values())
}

// Calculate cart total using Monoid
interface CartTotal {
  readonly itemCount: number
  readonly subtotal: number
}

const CartTotalMonoid: M.Monoid<CartTotal> = {
  concat: (a, b) => ({
    itemCount: a.itemCount + b.itemCount,
    subtotal: a.subtotal + b.subtotal
  }),
  empty: { itemCount: 0, subtotal: 0 }
}

const calculateTotal = (items: CartItem[]): CartTotal =>
  pipe(
    items,
    A.map(item => ({
      itemCount: item.quantity,
      subtotal: item.quantity * item.price
    })),
    M.concatAll(CartTotalMonoid)
  )
```

### Building Domain-Specific Type Classes

```typescript
import * as Eq from 'fp-ts/Eq'
import * as Ord from 'fp-ts/Ord'
import * as S from 'fp-ts/Semigroup'
import * as M from 'fp-ts/Monoid'
import { pipe } from 'fp-ts/function'

// Domain: Money handling

interface Money {
  readonly amount: number
  readonly currency: string
}

// Only allow operations on same currency
const MoneyEq: Eq.Eq<Money> = Eq.struct({
  amount: N.Eq,
  currency: Str.Eq
})

// Compare money amounts (same currency only)
const MoneyOrd: Ord.Ord<Money> = pipe(
  N.Ord,
  Ord.contramap((m: Money) => m.amount)
)

// Add money (throws if different currencies)
const MoneySemigroup = (currency: string): S.Semigroup<Money> => ({
  concat: (a, b) => {
    if (a.currency !== currency || b.currency !== currency) {
      throw new Error('Cannot combine different currencies')
    }
    return { amount: a.amount + b.amount, currency }
  }
})

// Safe money addition using Option
const addMoney = (a: Money, b: Money): O.Option<Money> =>
  a.currency === b.currency
    ? O.some({ amount: a.amount + b.amount, currency: a.currency })
    : O.none

// Monoid for specific currency
const MoneyMonoid = (currency: string): M.Monoid<Money> => ({
  ...MoneySemigroup(currency),
  empty: { amount: 0, currency }
})

// Sum all USD amounts
const totalUSD = (amounts: Money[]): Money =>
  pipe(
    amounts,
    A.filter(m => m.currency === 'USD'),
    M.concatAll(MoneyMonoid('USD'))
  )
```

## Practical Domain Modeling Examples

### Example 1: E-commerce Order System

```typescript
import * as E from 'fp-ts/Either'
import * as O from 'fp-ts/Option'
import * as A from 'fp-ts/Array'
import * as M from 'fp-ts/Monoid'
import { pipe } from 'fp-ts/function'

// Sum type for order status
type OrderStatus =
  | { readonly _tag: 'Draft' }
  | { readonly _tag: 'Pending'; readonly submittedAt: Date }
  | { readonly _tag: 'Paid'; readonly paidAt: Date; readonly transactionId: string }
  | { readonly _tag: 'Shipped'; readonly shippedAt: Date; readonly trackingNumber: string }
  | { readonly _tag: 'Delivered'; readonly deliveredAt: Date }
  | { readonly _tag: 'Cancelled'; readonly cancelledAt: Date; readonly reason: string }
  | { readonly _tag: 'Refunded'; readonly refundedAt: Date; readonly refundAmount: number }

// Smart constructors
const OrderStatus = {
  draft: (): OrderStatus => ({ _tag: 'Draft' }),
  pending: (submittedAt: Date): OrderStatus => ({ _tag: 'Pending', submittedAt }),
  paid: (paidAt: Date, transactionId: string): OrderStatus => ({ _tag: 'Paid', paidAt, transactionId }),
  shipped: (shippedAt: Date, trackingNumber: string): OrderStatus => ({ _tag: 'Shipped', shippedAt, trackingNumber }),
  delivered: (deliveredAt: Date): OrderStatus => ({ _tag: 'Delivered', deliveredAt }),
  cancelled: (cancelledAt: Date, reason: string): OrderStatus => ({ _tag: 'Cancelled', cancelledAt, reason }),
  refunded: (refundedAt: Date, refundAmount: number): OrderStatus => ({ _tag: 'Refunded', refundedAt, refundAmount })
}

// Pattern matching
const foldOrderStatus = <R>(handlers: {
  Draft: () => R
  Pending: (submittedAt: Date) => R
  Paid: (paidAt: Date, transactionId: string) => R
  Shipped: (shippedAt: Date, trackingNumber: string) => R
  Delivered: (deliveredAt: Date) => R
  Cancelled: (cancelledAt: Date, reason: string) => R
  Refunded: (refundedAt: Date, refundAmount: number) => R
}) => (status: OrderStatus): R => {
  switch (status._tag) {
    case 'Draft': return handlers.Draft()
    case 'Pending': return handlers.Pending(status.submittedAt)
    case 'Paid': return handlers.Paid(status.paidAt, status.transactionId)
    case 'Shipped': return handlers.Shipped(status.shippedAt, status.trackingNumber)
    case 'Delivered': return handlers.Delivered(status.deliveredAt)
    case 'Cancelled': return handlers.Cancelled(status.cancelledAt, status.reason)
    case 'Refunded': return handlers.Refunded(status.refundedAt, status.refundAmount)
  }
}

// Product type for order
interface Order {
  readonly id: string
  readonly customerId: string
  readonly items: ReadonlyArray<OrderItem>
  readonly status: OrderStatus
  readonly shippingAddress: Address
  readonly billingAddress: Address
  readonly createdAt: Date
  readonly updatedAt: Date
}

interface OrderItem {
  readonly productId: string
  readonly productName: string
  readonly quantity: number
  readonly unitPrice: number
  readonly discount: number
}

interface Address {
  readonly line1: string
  readonly line2: O.Option<string>
  readonly city: string
  readonly state: string
  readonly postalCode: string
  readonly country: string
}

// Monoid for order totals
interface OrderTotals {
  readonly subtotal: number
  readonly discount: number
  readonly tax: number
  readonly shipping: number
  readonly total: number
}

const OrderTotalsMonoid: M.Monoid<OrderTotals> = M.struct({
  subtotal: N.MonoidSum,
  discount: N.MonoidSum,
  tax: N.MonoidSum,
  shipping: N.MonoidSum,
  total: N.MonoidSum
})

const calculateItemTotal = (item: OrderItem): OrderTotals => {
  const subtotal = item.quantity * item.unitPrice
  const discount = item.discount
  return {
    subtotal,
    discount,
    tax: 0, // Calculated separately
    shipping: 0,
    total: subtotal - discount
  }
}

const calculateOrderTotals = (order: Order, taxRate: number, shippingCost: number): OrderTotals => {
  const itemTotals = pipe(
    order.items,
    A.map(calculateItemTotal),
    M.concatAll(OrderTotalsMonoid)
  )

  const tax = (itemTotals.subtotal - itemTotals.discount) * taxRate
  const total = itemTotals.subtotal - itemTotals.discount + tax + shippingCost

  return {
    ...itemTotals,
    tax,
    shipping: shippingCost,
    total
  }
}

// State transitions with validation
const canTransition = (from: OrderStatus, to: OrderStatus['_tag']): boolean =>
  foldOrderStatus({
    Draft: () => to === 'Pending' || to === 'Cancelled',
    Pending: () => to === 'Paid' || to === 'Cancelled',
    Paid: () => to === 'Shipped' || to === 'Refunded',
    Shipped: () => to === 'Delivered' || to === 'Refunded',
    Delivered: () => to === 'Refunded',
    Cancelled: () => false,
    Refunded: () => false
  })(from)
```

### Example 2: Configuration Management

```typescript
import * as S from 'fp-ts/Semigroup'
import * as O from 'fp-ts/Option'
import * as A from 'fp-ts/Array'
import { pipe } from 'fp-ts/function'

// Sum type for log level
type LogLevel = 'debug' | 'info' | 'warn' | 'error'

const LogLevelOrd: Ord.Ord<LogLevel> = pipe(
  N.Ord,
  Ord.contramap((level: LogLevel) => {
    const order: Record<LogLevel, number> = { debug: 0, info: 1, warn: 2, error: 3 }
    return order[level]
  })
)

// Product type for app config
interface AppConfig {
  readonly server: ServerConfig
  readonly database: DatabaseConfig
  readonly logging: LoggingConfig
  readonly features: FeatureFlags
}

interface ServerConfig {
  readonly host: string
  readonly port: number
  readonly timeout: number
  readonly maxConnections: number
}

interface DatabaseConfig {
  readonly host: string
  readonly port: number
  readonly database: string
  readonly poolSize: number
  readonly ssl: boolean
}

interface LoggingConfig {
  readonly level: LogLevel
  readonly format: 'json' | 'text'
  readonly outputs: ReadonlyArray<string>
}

interface FeatureFlags {
  readonly [key: string]: boolean
}

// Semigroups for merging configs (later values override)
const ServerConfigSemigroup: S.Semigroup<ServerConfig> = S.struct({
  host: S.last<string>(),
  port: S.last<number>(),
  timeout: S.max(N.Ord), // Take higher timeout
  maxConnections: S.max(N.Ord) // Take higher max connections
})

const DatabaseConfigSemigroup: S.Semigroup<DatabaseConfig> = S.struct({
  host: S.last<string>(),
  port: S.last<number>(),
  database: S.last<string>(),
  poolSize: S.max(N.Ord),
  ssl: { concat: (a, b) => a || b } // SSL if either enables it
})

const LoggingConfigSemigroup: S.Semigroup<LoggingConfig> = {
  concat: (a, b) => ({
    level: Ord.min(LogLevelOrd)(a.level, b.level), // Most verbose level
    format: b.format, // Last format wins
    outputs: [...new Set([...a.outputs, ...b.outputs])] // Union of outputs
  })
}

const FeatureFlagsSemigroup: S.Semigroup<FeatureFlags> = {
  concat: (a, b) => ({ ...a, ...b }) // Later flags override
}

const AppConfigSemigroup: S.Semigroup<AppConfig> = S.struct({
  server: ServerConfigSemigroup,
  database: DatabaseConfigSemigroup,
  logging: LoggingConfigSemigroup,
  features: FeatureFlagsSemigroup
})

// Load and merge configs from multiple sources
const loadConfig = (sources: AppConfig[]): AppConfig =>
  pipe(
    sources,
    A.reduce(defaultConfig, AppConfigSemigroup.concat)
  )

// Example usage
const defaultConfig: AppConfig = {
  server: { host: 'localhost', port: 3000, timeout: 5000, maxConnections: 100 },
  database: { host: 'localhost', port: 5432, database: 'app', poolSize: 10, ssl: false },
  logging: { level: 'info', format: 'text', outputs: ['console'] },
  features: {}
}

const envConfig: AppConfig = {
  server: { host: 'api.prod.com', port: 443, timeout: 10000, maxConnections: 1000 },
  database: { host: 'db.prod.com', port: 5432, database: 'app_prod', poolSize: 50, ssl: true },
  logging: { level: 'warn', format: 'json', outputs: ['console', 'file', 'cloudwatch'] },
  features: { newCheckout: true, betaFeature: false }
}

const finalConfig = loadConfig([defaultConfig, envConfig])
```

## Best Practices

### 1. Use Discriminant Property Consistently

```typescript
// Good: consistent _tag for all variants
type Result<E, A> =
  | { readonly _tag: 'Failure'; readonly error: E }
  | { readonly _tag: 'Success'; readonly value: A }

// Bad: inconsistent discriminant
type BadResult<E, A> =
  | { readonly type: 'error'; readonly error: E }
  | { readonly kind: 'success'; readonly value: A }
```

### 2. Make Illegal States Unrepresentable

```typescript
// Bad: allows invalid combinations
interface BadOrder {
  readonly status: 'shipped' | 'pending'
  readonly trackingNumber: string | null // Only valid for shipped
}

// Good: sum type ensures valid combinations
type GoodOrder =
  | { readonly _tag: 'Pending' }
  | { readonly _tag: 'Shipped'; readonly trackingNumber: string }
```

### 3. Use Smart Constructors

```typescript
// Expose constructors, not raw objects
const Order = {
  pending: (): Order => ({ _tag: 'Pending' }),
  shipped: (trackingNumber: string): Order => ({ _tag: 'Shipped', trackingNumber })
}

// Don't export the type definition directly for construction
// Export only through smart constructors
```

### 4. Prefer Structural Type Classes

```typescript
import * as Eq from 'fp-ts/Eq'
import * as S from 'fp-ts/Semigroup'

// Good: use struct to build from smaller pieces
const UserEq: Eq.Eq<User> = Eq.struct({
  id: Str.Eq,
  email: Str.Eq,
  name: Str.Eq
})

// Good: compose type class instances
const ConfigSemigroup: S.Semigroup<Config> = S.struct({
  server: ServerConfigSemigroup,
  database: DatabaseConfigSemigroup
})
```

### 5. Keep Fold/Match Exhaustive

```typescript
// TypeScript will error if you miss a case
const handleStatus = (status: OrderStatus): string =>
  pipe(
    status,
    foldOrderStatus({
      Draft: () => 'Draft',
      Pending: () => 'Pending',
      Paid: () => 'Paid',
      Shipped: () => 'Shipped',
      Delivered: () => 'Delivered',
      Cancelled: () => 'Cancelled',
      Refunded: () => 'Refunded'
      // Missing a case would cause compile error
    })
  )
```

## Anti-Patterns to Avoid

### Don't Use Enums for Sum Types

```typescript
// Bad: enums don't carry associated data
enum OrderStatus {
  Pending,
  Shipped,
  Delivered
}
// Where does tracking number go for Shipped?

// Good: discriminated unions carry data
type OrderStatus =
  | { readonly _tag: 'Pending' }
  | { readonly _tag: 'Shipped'; readonly trackingNumber: string }
  | { readonly _tag: 'Delivered'; readonly deliveredAt: Date }
```

### Don't Mix Type Classes Incorrectly

```typescript
// Bad: using Ord when you only need Eq
const findUser = (users: User[], target: User) =>
  users.find(u => Ord.equals(UserOrd)(u, target)) // Ord.equals doesn't exist

// Good: use the right type class
const findUser = (users: User[], target: User) =>
  users.find(u => UserEq.equals(u, target))
```

### Don't Forget Identity Laws for Monoids

```typescript
// Bad: this isn't a valid Monoid
const BadMonoid: M.Monoid<number> = {
  concat: (a, b) => a + b + 1, // Adding 1 breaks identity law
  empty: 0
}
// concat(0, 5) = 6, not 5!

// Good: respects identity laws
const GoodMonoid: M.Monoid<number> = {
  concat: (a, b) => a + b,
  empty: 0
}
// concat(0, 5) = 5 ✓
```
